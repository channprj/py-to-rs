# Chapter 17. 상태 공유와 동기화 (Shared State)

지난 16장에서 우리는 채널(Channel)을 통한 메시지 패싱(Message Passing)이 스레드 간 통신의 가장 우아한 방법이라고 못 박았습니다. *"공유 메모리로 통신하지 말고, 통신함으로써 메모리를 공유하라."* 이 격언은 대부분의 경우 옳습니다. 하지만 현실의 백엔드는 그렇게 깔끔하지 않습니다.

조회수 카운터를 생각해 봅시다. 초당 5만 건의 요청이 들어오는 API 게이트웨이에서, 각 워커 스레드가 "내가 처리한 요청 수"를 메인 스레드로 채널을 통해 일일이 보내고, 메인 스레드가 그것을 합산한다면? 채널의 직렬화·역직렬화 비용과 단일 합산 스레드가 곧 병목이 됩니다. 인메모리 캐시(Cache)도 마찬가지입니다. 수십 개의 스레드가 하나의 거대한 `HashMap`을 읽고, 가끔 갱신해야 한다면, 매번 캐시 전체를 채널로 복사해 보내는 것은 미친 짓입니다. 이럴 때는 어쩔 수 없이 **여러 스레드가 하나의 데이터를 직접 공유(Shared State)**해야 합니다.

파이썬에서 이 문제는 익숙합니다. 전역 딕셔너리나 카운터를 두고, 건드리기 전에 `threading.Lock()`을 거는 것이죠. 문제는, 파이썬의 락은 **신호등**일 뿐이라는 데 있습니다. 신호등이 빨간불이어도 차를 몰고 지나갈 수 있듯이, 개발자가 락을 거는 것을 깜빡하고 변수를 건드려도 인터프리터는 아무 말 하지 않습니다. 그 결과 데이터는 조용히 꼬이고, 버그는 운영 환경에서, 그것도 부하가 몰리는 새벽 3시에야 모습을 드러냅니다.

Rust는 이 문제를 정면으로 다릅니다. "락을 거는 규율"을 개발자의 양심에 맡기지 않고, **타입 시스템에 박아 넣어 물리적으로 강제**합니다. 락을 걸지 않으면 데이터에 접근할 방법 자체가 없도록 설계한 것이죠. 더 나아가, 어떤 타입이 스레드 간에 안전하게 이동·공유될 수 있는지를 `Send`와 `Sync`라는 두 개의 마커 트레이트로 컴파일 타임에 검증합니다. 이 장이 끝날 무렵이면, 여러분은 왜 Rust 커뮤니티가 "두려움 없는 동시성(Fearless Concurrency)"이라는 말을 자신 있게 쓰는지 이해하게 될 것입니다.

이 장에서는 `Mutex<T>`로 데이터를 가두는 법, `Arc<T>`로 소유권을 여러 스레드에 나눠 주는 법, 읽기가 압도적으로 많을 때의 `RwLock<T>`, 락조차 필요 없는 원자 타입(Atomic Types), 그리고 이 모든 것을 떠받치는 `Send`/`Sync`의 내부 원리를 다룹니다. 마지막에는 파이썬의 `multiprocessing.Manager` 기반 공유 메모리를 `Arc<Mutex<HashMap>>`으로 리팩토링하는 실전 예제로 마무리합니다.

---

## 17.1 The Python Way — 신호등에 불과한 Lock

파이썬에서 공유 카운터를 안전하게 다루는 정석 코드는 다음과 같습니다.

```python
import threading

counter = 0
lock = threading.Lock()

def increment() -> None:
    global counter
    with lock:
        counter += 1

threads = [threading.Thread(target=increment) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # 10 — 운이 좋으면
```

이 코드는 `with lock:` 블록 덕분에 잘 동작합니다. 문제는 **`lock`과 `counter` 사이에 아무런 구조적 연결이 없다**는 것입니다. 둘은 그냥 같은 스코프에 나란히 놓인 별개의 변수일 뿐입니다. 누군가 다음과 같이 작성해도 인터프리터는 전혀 막지 않습니다.

```python
def increment_buggy() -> None:
    global counter
    # 🚨 lock을 깜빡함. 문법적으로 완벽히 유효하다.
    counter += 1
```

`counter += 1`은 파이썬 바이트코드로 보면 `LOAD → INC → STORE` 세 단계로 쪼개집니다. GIL이 이 세 바이트코드 사이에서 스레드 전환을 일으키면, 두 스레드가 같은 값을 읽고 같은 값을 쓰는 갱신 분실(Lost Update)이 발생합니다. CPython의 GIL은 *바이트코드 하나*의 원자성만 보장할 뿐, *연산 하나*의 원자성은 보장하지 않습니다. 그래서 `counter += 1`조차 멀티스레드에서는 위험합니다.

여기에 더해 파이썬 락은 두 가지 함정을 더 안고 있습니다.

* **해제 누락.** `with` 문 없이 `lock.acquire()` / `lock.release()`를 직접 쓰다가 중간에 예외가 터지면 `release()`가 호출되지 않아 영구 데드락에 빠집니다. `try/finally`로 감싸야 하지만, 이 역시 개발자의 규율에 달려 있습니다.
* **무엇을 보호하는지가 코드에 없다.** `lock`이 `counter`를 보호하는지, `cache`를 보호하는지, 둘 다인지는 오직 주석과 개발자의 기억 속에만 존재합니다. 6개월 뒤 합류한 동료는 알 길이 없습니다.

여기서 한 가지 반론이 나올 수 있습니다. "GIL이 있으니 어차피 한 번에 한 스레드만 실행되는데, 왜 락이 필요한가?" 이것이 파이썬 동시성에서 가장 흔한 오해입니다. GIL은 **한 번에 하나의 바이트코드만** 실행됨을 보장할 뿐, 여러 바이트코드로 이루어진 **논리적 연산 하나**의 원자성은 보장하지 않습니다. `counter += 1`은 앞서 본 대로 `LOAD_FAST → LOAD_CONST → INPLACE_ADD → STORE_FAST`로 쪼개지고, GIL은 이 네 명령 사이 어디서든(기본적으로 일정 바이트코드 수마다, 혹은 `sys.setswitchinterval`이 정한 시간마다) 다른 스레드에게 실행권을 넘겨줄 수 있습니다. 그 틈에 두 스레드가 같은 `counter` 값을 읽으면 갱신 하나가 증발합니다. 그래서 GIL이 있어도 락은 여전히 필요합니다.

게다가 이 안전 가정마저 흔들리고 있습니다. Python 3.13부터 실험적으로 도입된 **free-threaded(no-GIL) 빌드**(PEP 703)에서는 GIL이라는 거대한 안전망이 사라집니다. GIL에 암묵적으로 기대 "운 좋게" 동작하던 코드들이 진짜 병렬 환경에서 일제히 깨질 것입니다. 즉, 파이썬은 이제껏 GIL이라는 전역 락 하나로 데이터 레이스의 상당 부분을 *우연히* 가려 왔을 뿐, 언어 차원에서 막아 준 적이 없습니다.

요약하면, 파이썬의 동기화는 **컨벤션(convention)**입니다. 잘 지키면 동작하고, 한 곳이라도 어기면 무너지는데, 어긴 지점을 컴파일러가 알려주지 않습니다. 락을 깜빡한 코드, 보호 대상이 모호한 락, GIL에 기댄 위태로운 가정 — 이 모든 위험이 런타임까지 살아남았다가 운영 환경의 부하 속에서 비결정적으로 터집니다. Rust는 이 컨벤션을 통째로 **타입**으로 바꿉니다. 다음 절부터 그 방식을 봅니다.

---

## 17.2 The Rust Mental Model — 데이터를 락 안에 가둔다

Rust의 핵심 전환은 한 문장으로 요약됩니다. **락과 데이터를 분리하지 않는다.** 보호해야 할 데이터를 `Mutex<T>`라는 껍질 안에 물리적으로 집어넣고, 그 데이터에 접근하는 유일한 통로를 "락을 획득하는 행위"로 만들어 버립니다.

```rust
use std::sync::Mutex;

fn main() {
    // 0이라는 정수를 Mutex 안에 가둡니다. 이제 이 0은 m.lock() 없이는 만질 수 없습니다.
    let m = Mutex::new(0);

    {
        // lock()은 MutexGuard를 돌려줍니다. 이 가드를 통해서만 내부 데이터에 접근합니다.
        let mut num = m.lock().unwrap();
        *num = 6;
    } // 스코프가 끝나면 num(MutexGuard)이 Drop되면서 락이 '자동으로' 해제됩니다 (RAII).

    println!("m = {:?}", m); // m = Mutex { data: 6, poisoned: false, .. }
}
```

이 코드에서 일어나는 일을 파이썬 개발자의 눈높이로 분해해 봅시다.

1. `Mutex::new(0)`은 정수 `0`을 뮤텍스 내부에 **소유권째로** 가져갑니다. 이제 바깥에서 이 `0`을 직접 가리키는 경로는 존재하지 않습니다.
2. `m.lock()`은 락을 획득하고 `LockResult<MutexGuard<i32>>`를 반환합니다. `unwrap()`으로 `MutexGuard<i32>`를 꺼냅니다.
3. `MutexGuard`는 `Deref`/`DerefMut`를 구현한 스마트 포인터입니다. `*num`으로 내부 `i32`에 접근합니다.
4. 핵심은 **3번에서 데이터에 접근하려면 반드시 2번을 거쳐야 한다**는 점입니다. 락을 안 걸고 내부 데이터에 닿을 타입 시스템상의 경로가 없습니다.

파이썬의 `counter += 1` 같은 "락 없는 직접 접근"을 작성하고 싶어도, Rust에서는 그런 코드를 **표현할 수단 자체가 없습니다.** 신호등을 무시할 수 있느냐 없느냐의 차이가 아니라, 애초에 신호등 없이는 도로에 진입조차 못 하도록 도로 구조를 바꾼 것입니다.

이 설계가 주는 부수 효과 하나를 짚고 넘어가야 합니다. 파이썬에서 `lock`과 `counter`는 별개 변수라 "이 락이 무엇을 보호하는가"가 코드에 드러나지 않았습니다. Rust에서는 `Mutex<HashMap<String, Session>>` 같은 타입 자체가 "이 락은 이 세션 맵을 보호한다"를 선언합니다. 보호 관계가 **타입 시그니처에 명시**되므로, 6개월 뒤 합류한 동료도, 코드 리뷰어도, 심지어 IDE의 자동완성도 그 관계를 알고 있습니다. 동기화의 의도가 주석이 아니라 타입으로 문서화되는 것입니다.

파이썬과 Rust의 동기화 개념을 한눈에 매핑하면 다음과 같습니다.

| Python | Rust | 핵심 차이 |
|--------|------|-----------|
| `threading.Lock` (데이터와 분리) | `Mutex<T>` (데이터를 내장) | Rust는 락 없이는 데이터 접근 불가 |
| `with lock:` 블록 | `MutexGuard` + 스코프 (RAII) | Rust는 패닉 시에도 자동 해제 보장 |
| 직접 `acquire()`/`release()` | (불가능 — 가드를 통해서만) | Rust는 해제 누락이 구조적으로 불가능 |
| `threading.RLock` (재진입 가능) | `Mutex` (재진입 불가) / 별도 설계 | Rust 기본 Mutex는 재진입 시 데드락 |
| `queue.Queue` (스레드 안전 큐) | `mpsc::channel` (16장) | Rust는 송신 시 소유권 이동 |
| `multiprocessing.Manager().dict()` | `Arc<Mutex<HashMap>>` | Rust는 IPC·직렬화 없이 메모리 직접 공유 |
| (표준 RWLock 없음) | `RwLock<T>` (표준 제공) | Rust는 다중 읽기/단독 쓰기 내장 |
| `multiprocessing.Value` + lock | `AtomicUsize` 등 원자 타입 | Rust는 락 없는 원자 연산 |
| GIL (전역 단일 락) | (없음 — 타입별 세분화 동기화) | Rust는 진정한 병렬 + 컴파일 타임 검증 |

이 표의 마지막 줄이 핵심입니다. 파이썬은 GIL이라는 단 하나의 거대한 락으로 인터프리터 전체를 직렬화해 안전을 *흉내* 냈습니다. Rust는 GIL이 없는 대신, 데이터마다 적절한 동기화 도구를 골라 붙이고 그 정확성을 컴파일러가 검증합니다. 자유와 책임을 동시에 주되, 책임의 이행을 타입 시스템이 강제하는 구조입니다.

> **Under the Hood: MutexGuard와 RAII가 해제를 보장하는 원리**
>
> 파이썬의 `with lock:`은 `__enter__`/`__exit__` 프로토콜로 동작합니다. Rust의 `MutexGuard`는 같은 효과를 `Drop` 트레이트로 냅니다(4장에서 본 RAII). `lock()`이 반환하는 `MutexGuard<T>`는 내부적으로 대략 이런 모양입니다.
>
> ```rust
> // 표준 라이브러리의 개념적 형태 (단순화)
> pub struct MutexGuard<'a, T> {
>     lock: &'a Mutex<T>,
> }
>
> impl<T> Drop for MutexGuard<'_, T> {
>     fn drop(&mut self) {
>         // 가드가 스코프를 벗어나는 '바로 그 지점'에서 락을 해제한다.
>         unsafe { self.lock.raw_unlock() }
>     }
> }
> ```
>
> 함수가 정상 반환하든, `?`로 조기 반환하든, `panic!`으로 스택을 되감든(unwinding), 가드는 스코프를 벗어나는 순간 반드시 `drop`됩니다. 따라서 파이썬에서 `try/finally`를 깜빡해 발생하던 "예외로 인한 락 누수"가 Rust에서는 **구조적으로 불가능**합니다. `MutexGuard`에는 `'a` 수명이 박혀 있어서(6장의 수명), 가드가 원본 `Mutex`보다 오래 살 수도 없습니다.

---

## 17.3 여러 스레드로 확장하기 — `Arc<T>`

`Mutex<T>` 하나만으로는 멀티스레드에서 쓸 수 없습니다. 소유권 규칙(4장) 때문입니다. 하나의 `Mutex`를 10개의 스레드에 넘기려면 각 스레드가 그것을 소유해야 하는데, 소유권은 단 하나뿐입니다.

먼저, **`Arc` 없이** 그냥 `Mutex`를 스레드로 넘기려고 하면 무슨 일이 벌어지는지 봅시다.

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn(move || {     // 🚨 컴파일 에러!
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

`rustc`는 정확히 무엇이 문제인지 짚어 줍니다.

```text
error[E0382]: use of moved value: `counter`
 --> src/main.rs:8:36
  |
5 |     let counter = Mutex::new(0);
  |         ------- move occurs because `counter` has type `std::sync::Mutex<i32>`,
  |                 which does not implement the `Copy` trait
...
8 |         let handle = thread::spawn(move || {
  |                                    ^^^^^^^ value moved into closure here,
  |                                            in previous iteration of loop
9 |             let mut num = counter.lock().unwrap();
  |                           ------- use occurs due to use in closure
```

해독하면 이렇습니다. `move` 클로저가 첫 번째 루프 반복에서 `counter`의 소유권을 통째로 가져갔습니다(`Mutex`는 `Copy`가 아니므로 이동입니다). 두 번째 반복에서는 더 이상 넘겨줄 `counter`가 없습니다. 파이썬이라면 참조 카운트를 올려가며 같은 객체를 무한정 넘길 수 있지만, Rust의 기본 규칙은 "소유권은 하나"입니다.

해법은 **공유 소유권(Shared Ownership)**입니다. 파이썬의 모든 객체는 내부에 참조 카운트(`ob_refcnt`)를 들고 있어, 변수를 여기저기 넘겨도 카운트만 오르내릴 뿐 객체는 하나로 유지됩니다. Rust는 이 메커니즘을 **옵트인(opt-in)** 방식으로 제공합니다. 단일 스레드용은 `Rc<T>`(Reference Counted), 멀티스레드용은 `Arc<T>`(**A**tomic **R**eference **C**ounted)입니다.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // 1. 데이터를 Mutex로 가두고, 그 Mutex를 다시 Arc로 감쌉니다.
    //    Arc는 소유권 공유를, Mutex는 동시 수정 차단을 담당합니다.
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        // 2. Arc::clone은 데이터를 복사하지 않습니다. 포인터를 복제하고
        //    원자적 카운트를 1 올릴 뿐입니다 (비용: 정수 증가 한 번).
        let counter_clone = Arc::clone(&counter);

        let handle = thread::spawn(move || {
            // 3. 각 스레드가 자기 몫의 Arc를 들고 들어가 락을 획득하고 수정합니다.
            let mut num = counter_clone.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final Result: {}", *counter.lock().unwrap()); // Final Result: 10
}
```

`Arc::clone(&counter)`는 파이썬 개발자가 흔히 오해하는 지점입니다. 이것은 내부 `Mutex<0>`을 복제하는 것이 **아닙니다.** 힙에 있는 데이터 하나를 가리키는 **포인터를 하나 더 만들고, 그 데이터에 붙은 참조 카운터를 원자적으로 +1** 하는 것이 전부입니다. 그래서 관용적으로 `arc.clone()`보다 `Arc::clone(&arc)`로 적어, "이건 깊은 복사가 아니라 참조 카운트 증가다"라는 의도를 코드에 드러냅니다.

### `Rc` vs `Arc` — 왜 두 개인가

여기서 자연스러운 질문이 나옵니다. 그냥 `Arc` 하나만 쓰면 안 되나? 왜 단일 스레드용 `Rc`를 따로 두었나?

| 항목 | `Rc<T>` | `Arc<T>` |
|------|---------|----------|
| 카운터 증감 방식 | 일반 정수 연산 (`count += 1`) | 원자적 연산 (`fetch_add`, 메모리 펜스 동반) |
| 스레드 안전성 | `!Send`, `!Sync` (단일 스레드 전용) | `Send + Sync` (T가 `Send + Sync`일 때) |
| 카운트 증가 비용 | 1~2 사이클 | 수~수십 사이클 (CPU·경합도에 따라) |
| 용도 | 단일 스레드 그래프/트리(20장) | 멀티스레드 공유 상태 |

핵심은 **원자적 연산은 공짜가 아니다**라는 것입니다. 원자적 증감은 CPU의 캐시 일관성 프로토콜(MESI 등)을 건드리고 메모리 순서를 강제하기 때문에, 단순 정수 증가보다 훨씬 비쌉니다. 단일 스레드에서만 쓸 거라면 이 비용을 낼 이유가 없습니다. Rust는 "무비용 추상화(Zero-cost Abstraction)" 철학에 따라, **필요 없는 비용은 강요하지 않습니다.** 그래서 값싼 `Rc`와 안전한 `Arc`를 분리해 두고, 멀티스레드가 필요할 때만 비싼 `Arc`를 고르게 합니다. 파이썬은 모든 객체가 (GIL 보호 하의) 참조 카운트를 항상 들고 다니므로 이런 선택지가 없습니다.

타입을 바꾸는 일도 간단합니다. `Rc<T>`로 짠 단일 스레드 코드를 멀티스레드로 확장해야 한다면, 대부분 `Rc`를 `Arc`로 치환하는 것만으로 됩니다. 컴파일러가 `Send`/`Sync` 경계에서 빠진 곳을 정확히 짚어 주므로, "어디를 고쳐야 하는지 몰라서" 헤매는 일이 없습니다. 이것이 컴파일러를 동시성 리팩토링의 길잡이로 쓰는 전형적인 방식입니다.

한 가지 더. `Arc`도 파이썬 참조 카운팅과 똑같은 약점을 공유합니다. 바로 **순환 참조(reference cycle)**입니다. A가 B를 `Arc`로 들고 B가 A를 `Arc`로 들면, 둘의 강한 카운트가 영원히 0이 되지 않아 메모리가 누수됩니다. 파이썬은 별도의 순환 탐지 GC(`gc` 모듈)로 이를 청소하지만, Rust에는 그런 백그라운드 수집기가 없습니다. 대신 한쪽을 **`Weak<T>`(약한 참조)**로 바꿔 카운트에 영향을 주지 않게 하여 순환을 끊습니다. 부모-자식 트리에서 자식→부모 링크를 `Weak`로 두는 패턴이 대표적이며, 이 주제는 20장의 스마트 포인터에서 본격적으로 다룹니다.

> **Under the Hood: Arc의 메모리 레이아웃과 원자적 카운터**
>
> `Arc<T>`는 스택에 단 한 개의 포인터(64비트에서 8바이트)만 차지합니다. 그 포인터가 가리키는 힙에는 `ArcInner<T>`라는 내부 구조체가 놓입니다. 표준 라이브러리의 실제 형태를 단순화하면 이렇습니다.
>
> ```rust
> // std::sync::Arc 내부 (개념적 단순화)
> struct ArcInner<T> {
>     strong: AtomicUsize,  // 강한 참조 카운트
>     weak: AtomicUsize,    // 약한 참조(Weak) 카운트
>     data: T,              // 실제 데이터
> }
> ```
>
> 힙 메모리 배치(64비트, `T = Mutex<i32>` 가정):
>
> ```text
> 스택                힙 (ArcInner)
> ┌──────────┐       ┌────────────────────────────┐
> │ Arc(ptr) │──────▶│ strong: AtomicUsize (8B)    │
> └──────────┘       │ weak:   AtomicUsize (8B)    │
> (Arc 복제본들이     │ data:   Mutex<i32> { ... }  │
>  모두 같은 곳을     └────────────────────────────┘
>  가리킨다)
> ```
>
> `Arc::clone`은 `strong.fetch_add(1, Relaxed)` 한 번입니다. 드롭 시에는 `strong.fetch_sub(1, Release)`로 카운트를 내리고, 그 결과가 0이면 메모리 펜스(`Acquire`) 후 `data`를 드롭하고 힙을 해제합니다. 이 `Release`/`Acquire` 조합은 "마지막으로 카운트를 내리는 스레드가 다른 모든 스레드의 쓰기를 빠짐없이 관측한 뒤에야 해제하도록" 보장하는 동기화 장치입니다. 파이썬의 `Py_INCREF`/`Py_DECREF`가 GIL이라는 거대한 자물쇠에 의존해 단일 스레드 가정 하에서 카운트를 만지는 것과 달리, `Arc`는 **개별 변수 단위의 원자적 연산**으로 GIL 없이 정확성을 달성합니다. 이것이 Rust가 진정한 병렬 스레드를 가질 수 있는 이유입니다(16장에서 본 "GIL 없음").

---

## 17.4 읽기가 많을 때 — `RwLock<T>`

`Mutex`는 읽기든 쓰기든 한 번에 한 스레드만 들여보냅니다. 그런데 캐시나 설정(config)처럼 **읽기는 초당 수만 번, 쓰기는 분당 한 번** 같은 패턴이라면 어떨까요? 읽기끼리는 서로 충돌하지 않는데도 `Mutex`는 읽는 스레드들을 줄 세워 직렬화합니다. 명백한 낭비입니다.

`RwLock<T>`(Reader-Writer Lock)는 이 비대칭을 활용합니다. **읽기 락(`read()`)은 여러 스레드가 동시에** 보유할 수 있지만, **쓰기 락(`write()`)은 단 하나만**, 그것도 읽는 스레드가 하나도 없을 때만 획득할 수 있습니다. 파이썬에는 표준 `RWLock`이 없어 보통 직접 구현하거나 외부 라이브러리에 의존하지만, Rust는 표준 라이브러리에 포함합니다.

```rust
use std::sync::{Arc, RwLock};
use std::thread;
use std::collections::HashMap;

fn main() {
    let config: Arc<RwLock<HashMap<String, String>>> =
        Arc::new(RwLock::new(HashMap::new()));

    // 초기 설정 주입 (쓰기 락)
    {
        let mut w = config.write().unwrap();
        w.insert("region".to_string(), "ap-northeast-2".to_string());
    } // 여기서 쓰기 락 해제

    let mut handles = vec![];
    for id in 0..4 {
        let cfg = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            // 읽기 락: 4개 스레드가 '동시에' 진입 가능
            let r = cfg.read().unwrap();
            let region = r.get("region").cloned().unwrap_or_default();
            format!("reader {id} sees region={region}")
        }));
    }

    for h in handles {
        println!("{}", h.join().unwrap());
    }
}
```

### `Mutex` vs `RwLock` — 트레이드오프

`RwLock`이 항상 더 빠를 것 같지만 그렇지 않습니다. 읽기 락도 내부적으로 리더 카운트를 원자적으로 증감해야 하므로, `Mutex`보다 락 자체의 오버헤드가 큽니다. 쓰기가 잦거나 임계 구역이 아주 짧으면 `RwLock`이 오히려 느립니다.

| 항목 | `Mutex<T>` | `RwLock<T>` |
|------|-----------|-------------|
| 동시 접근 | 항상 1개 (읽기/쓰기 무관) | 읽기 N개 동시 / 쓰기 1개 단독 |
| 락 자체 오버헤드 | 작음 | 큼 (리더/라이터 상태 관리) |
| 최적 시나리오 | 읽기·쓰기 비율이 비슷하거나 임계구역이 짧음 | 읽기가 쓰기보다 압도적으로 많음 |
| 기아(starvation) 위험 | 낮음 | 쓰기 기아 가능 (리더가 계속 몰리면 라이터가 못 들어감) |
| 가드 타입 | `MutexGuard<T>` | `RwLockReadGuard<T>` / `RwLockWriteGuard<T>` |

실무 기준은 단순합니다. **기본값은 `Mutex`로 시작하고, 프로파일링으로 읽기 경합이 병목임을 확인했을 때만 `RwLock`으로 바꿉니다.** "읽기가 많으니 무조건 `RwLock`"이라는 직감은 종종 틀립니다. 측정하십시오.

> **Under the Hood: RwLock의 상태와 쓰기 기아**
>
> `std::sync::RwLock`의 내부 상태는 하나의 원자적 워드로 표현됩니다(플랫폼마다 다르지만 개념적으로). 최하위 비트들은 "라이터가 대기/보유 중"을, 나머지 비트는 "현재 활성 리더 수"를 인코딩합니다. `read()`는 라이터 비트가 0일 때만 리더 카운트를 +1 하고, `write()`는 리더 카운트와 라이터 비트가 모두 0일 때만 진입합니다.
>
> 여기서 **쓰기 기아(Writer Starvation)**가 나옵니다. 리더가 끊임없이 들어오면 활성 리더 수가 0으로 떨어지는 순간이 오지 않아, 라이터가 영원히 대기할 수 있습니다. 표준 라이브러리의 `RwLock`은 플랫폼 OS 프리미티브에 위임하므로 공정성(fairness) 보장이 플랫폼마다 다릅니다. 공정성이 중요하면 `parking_lot::RwLock`(쓰기 우선 옵션 제공)을 쓰는 것이 실무 정석입니다.

---

## 17.5 락조차 필요 없을 때 — 원자 타입과 메모리 순서

조회수 카운터처럼 **하나의 정수를 여러 스레드가 증감**하기만 한다면, `Mutex<usize>`는 과합니다. 락을 잡고 푸는 비용이 정수 하나 더하는 비용을 압도하기 때문입니다. 이럴 때는 **원자 타입(Atomic Types)**을 직접 씁니다. CPU가 하드웨어 수준에서 원자성을 보장하는 명령(x86의 `lock xadd` 등)에 대응하는 타입들입니다.

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;

fn main() {
    let hits = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..8 {
        let hits = Arc::clone(&hits);
        handles.push(thread::spawn(move || {
            for _ in 0..100_000 {
                // 락 없이, CPU 명령 한 방으로 원자적 증가.
                hits.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("total hits = {}", hits.load(Ordering::Relaxed)); // total hits = 800000
}
```

8개 스레드가 각자 10만 번씩, 총 80만 번을 락 없이 증가시켰는데 결과는 정확히 `800000`입니다. `AtomicUsize`는 `Mutex<usize>`보다 훨씬 가볍습니다. 단, 만능은 아닙니다. 원자 타입은 정수·불리언·포인터 같은 **단일 워드 값**에만 쓸 수 있습니다. `HashMap`이나 여러 필드를 한꺼번에 일관되게 갱신해야 하는 경우에는 여전히 `Mutex`/`RwLock`이 필요합니다.

### 메모리 순서(Ordering)의 기초

`fetch_add`의 두 번째 인자 `Ordering`은 파이썬 개발자에게 가장 낯선 개념일 것입니다. 현대 CPU와 컴파일러는 성능을 위해 메모리 읽기·쓰기 순서를 **재배치**합니다. 단일 스레드에서는 결과가 같아 보이지만, 멀티스레드에서는 "스레드 A가 X를 쓴 뒤 Y를 썼는데, 스레드 B가 Y의 변화는 봤지만 X는 아직 옛값으로 보이는" 일이 벌어집니다. `Ordering`은 이 재배치를 어디까지 허용할지 컴파일러와 CPU에 지시하는 손잡이입니다.

* **`Relaxed`** — 원자성만 보장하고 순서 제약은 없습니다. 다른 변수와의 관계가 중요하지 않은 **단순 카운터**에 적합합니다. 위 예제처럼 "총 몇 번 증가했나"만 알면 될 때 가장 빠릅니다.
* **`Acquire` / `Release`** — 짝으로 씁니다. `Release` 쓰기 이전의 모든 메모리 작업은, 그 값을 `Acquire`로 읽은 스레드에게 빠짐없이 보이도록 보장합니다. "플래그를 세우기 전에 준비한 데이터를, 플래그를 본 스레드가 반드시 완성된 상태로 보게 한다"는 인계(handoff) 패턴의 핵심입니다.
* **`SeqCst`** (Sequentially Consistent) — 가장 강한 보장. 모든 스레드가 동일한 전역 순서로 연산을 관측합니다. 추론하기 가장 쉽지만 가장 비쌉니다.

실무 조언: **확신이 없으면 `SeqCst`를 쓰십시오.** 정확성이 성능보다 우선이고, 단순 카운터에는 `Relaxed`로 충분합니다. `Acquire`/`Release`로 직접 동기화 프로토콜을 짜는 것은 락프리(lock-free) 자료구조를 만드는 전문가의 영역이며, 99%의 백엔드 코드는 `Mutex`나 `Relaxed` 카운터로 충분합니다. 잘못 고른 `Ordering`은 컴파일도 통과하고 평소엔 멀쩡하다가, 특정 CPU·특정 부하에서만 데이터가 어긋나는 최악의 버그를 만듭니다.

### CAS — 락 없는 갱신의 진짜 모습

`fetch_add` 같은 단순 증감을 넘어, "현재 값을 읽고 → 계산하고 → 그 사이 아무도 안 바꿨을 때만 쓰기"라는 조건부 갱신이 필요할 때가 있습니다. 이것이 **CAS(Compare-And-Swap, 비교 후 교환)**이고, 모든 락프리 알고리즘의 심장입니다. Rust에서는 `compare_exchange`로 표현합니다.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

// 최댓값을 락 없이 갱신하는 패턴 (여러 스레드가 동시 호출해도 안전)
fn update_max(current_max: &AtomicUsize, candidate: usize) {
    let mut observed = current_max.load(Ordering::Relaxed);
    loop {
        if candidate <= observed {
            return; // 이미 더 큰 값이 들어 있음
        }
        // observed가 여전히 현재 값일 때만 candidate로 교체.
        // 그 사이 다른 스레드가 바꿨으면 Err(실제 값)를 받아 재시도.
        match current_max.compare_exchange_weak(
            observed,
            candidate,
            Ordering::Relaxed,
            Ordering::Relaxed,
        ) {
            Ok(_) => return,            // 교체 성공
            Err(actual) => observed = actual, // 경합 발생 -> 최신 값으로 재시도
        }
    }
}
```

이 `loop { compare_exchange }` 패턴이 락프리 자료구조의 기본 골격입니다. 락을 잡지 않으므로 한 스레드가 멈춰도 다른 스레드가 진행할 수 있다는 강력한 장점이 있지만, 경합이 심하면 재시도가 폭증해 오히려 락보다 느려질 수 있습니다. 그래서 실무에서 락프리 자료구조를 직접 짜기보다는, `crossbeam` 같은 검증된 크레이트의 락프리 큐·스택을 가져다 쓰는 것이 정석입니다. 직접 CAS 루프를 작성하는 것은 정말로 핫 패스의 마지막 한 방울까지 짜내야 할 때의 선택지입니다.

---

## 17.6 컴파일러는 어떻게 Data Race를 막는가 — `Send`와 `Sync`

지금까지 본 모든 안전성의 뿌리에는 두 개의 마커 트레이트가 있습니다.

* **`Send`** — 이 타입의 값을 **다른 스레드로 이동(move)**해도 안전한가? (소유권 이전)
* **`Sync`** — 이 타입에 대한 참조 `&T`를 **여러 스레드가 동시에 가져도** 안전한가? (공유 접근)

두 트레이트의 관계는 정확히 이렇게 정의됩니다. **`T`가 `Sync`인 것은 `&T`가 `Send`인 것과 동치입니다.** 즉, 어떤 타입의 불변 참조를 다른 스레드로 보내도 안전하다면, 그 타입은 `Sync`입니다.

이 두 트레이트의 특별한 점은 **자동 트레이트(auto trait)**라는 것입니다. 개발자가 `impl Send for MyType`이라고 쓰지 않습니다. 컴파일러가 구조체의 **모든 필드**를 검사해서, 모든 필드가 `Send`면 그 구조체도 자동으로 `Send`가 됩니다(`Sync`도 동일). 마치 전염되는 속성처럼, 안전한 부품으로만 조립한 타입은 자동으로 안전하다고 판정됩니다.

`thread::spawn`의 시그니처가 이 모든 것을 강제하는 관문입니다.

```rust
// std::thread::spawn (단순화)
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,  // 클로저가 Send여야 한다!
    T: Send + 'static,
```

`move` 클로저가 캡처한 모든 값이 `Send`여야만 클로저 자신이 `Send`가 됩니다. 따라서 `Send`가 아닌 무언가를 스레드로 보내려는 시도는 이 경계에서 컴파일 에러로 차단됩니다. 여기에 더해 `'static` 경계도 붙어 있는데, 이는 "스레드가 빌린 참조보다 오래 살 수 있으므로, 다른 스레드로 보내는 값은 빌린 참조를 품고 있으면 안 된다"는 수명(6장) 규칙입니다. `Send`가 데이터 레이스를, `'static`이 댕글링 참조를 각각 막는 이중 방어선입니다.

`Send`와 `Sync`가 **자동 트레이트**라는 점이 왜 중요한지 다시 강조할 가치가 있습니다. 파이썬에서 "이 클래스는 스레드 안전한가?"는 문서를 읽거나 소스를 직접 뒤져야 알 수 있는, 본질적으로 *런타임에도 보장되지 않는* 질문입니다. Rust에서는 컴파일러가 타입의 필드를 재귀적으로 검사해 자동으로 답을 냅니다. 여러분이 `struct AppState { cache: Arc<Mutex<HashMap<...>>>, counter: AtomicUsize }`를 정의하면, 모든 필드가 `Send + Sync`이므로 `AppState`도 자동으로 `Send + Sync`가 되어, 별도 선언 없이 스레드 간에 안전하게 공유할 수 있습니다. 반대로 필드 하나라도 `Rc`를 끼워 넣는 순간 전체가 `!Send`로 전염되어, 스레드로 보내려는 시도가 컴파일 에러로 막힙니다. 안전성이 부품에서 조립품으로 자동 전파되는 것입니다. (드물게 `unsafe impl Send`로 이 판정을 수동 재정의할 수 있지만, 이는 FFI 등 컴파일러가 알 수 없는 불변식을 개발자가 책임지고 보증할 때의 예외적 수단이며 21장에서 다룹니다.)

### 컴파일러 에러 시연: `Rc`를 스레드로 보내기

`Rc`는 단일 스레드 전용이라 `!Send`(Send 아님)입니다. 이것을 `thread::spawn`에 넣으면 어떻게 될까요?

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(42);
    let data_clone = Rc::clone(&data);

    thread::spawn(move || {            // 🚨 컴파일 에러!
        println!("{}", data_clone);
    });
}
```

`rustc`의 실제 출력:

```text
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs:7:19
    |
7   |       thread::spawn(move || {
    |       ------------- ^------
    |       |             |
    |  _____|_____________within this `{closure@src/main.rs:7:19: 7:26}`
    | |     |
    | |     required by a bound introduced by this call
8   | |         println!("{}", data_clone);
9   | |     });
    | |_____^ `Rc<i32>` cannot be sent between threads safely
    |
    = help: within `{closure@...}`, the trait `Send` is not implemented for `Rc<i32>`
note: required because it's used within this closure
note: required by a bound in `spawn`
```

해독: 클로저가 `Rc<i32>`를 캡처했는데 `Rc<i32>`는 `Send`를 구현하지 않으므로, 클로저도 `Send`가 아니고, 따라서 `spawn`의 `F: Send` 경계를 위반합니다. **데이터 레이스의 가능성이 단 한 줄의 잘못된 코드도 실행되기 전에, 타입 검사 단계에서 박멸됩니다.** 해법은 `Rc`를 `Arc`로 바꾸는 것, 그뿐입니다.

> **Under the Hood: 왜 `Rc`는 `!Send`이고 `Mutex`는 `Sync`인가**
>
> **`Rc`가 `!Send`인 이유.** `Rc`의 참조 카운트는 일반 정수(`usize`)입니다. 두 스레드가 같은 `Rc`의 복제본을 동시에 드롭하면, 둘 다 `count -= 1`을 `읽기 → 빼기 → 쓰기`로 수행하다가 갱신 분실이 일어납니다. 카운트가 실제보다 높게 남으면 메모리 누수, 낮게 남으면 **이중 해제(double free)** 후 use-after-free로 직행합니다. Rust는 이 위험을 원천 차단하기 위해 `Rc`를 `!Send`로 못 박았습니다. 정확히 이 카운트를 `AtomicUsize`로 바꾼 것이 `Arc`이고, 그래서 `Arc<T>`는 (`T: Send + Sync`일 때) `Send + Sync`입니다.
>
> **`Mutex<T>`가 `Sync`인 이유.** `Mutex<T>`는 `T: Send`이기만 하면 `Sync`입니다. 즉 `&Mutex<T>`를 여러 스레드가 동시에 가져도 안전합니다. 왜냐하면 `Mutex`의 존재 의의가 바로 "동시에 데이터에 접근하려는 스레드들을 락으로 한 줄로 세우는 것"이기 때문입니다. 여러 스레드가 같은 `Mutex`를 가리켜도, 내부 데이터에는 한 번에 하나씩만 닿습니다. 흥미로운 점은 **`Mutex`가 `T: Sync`를 요구하지 않는다**는 것입니다. `T`가 `Sync`가 아니어도(예: `Cell`), `Mutex`로 감싸면 락이 동시 접근을 막아 주므로 `Mutex<Cell<i32>>`는 안전하게 `Sync`가 됩니다. 락이 동기화를 책임지므로 내부 타입은 단지 다른 스레드로 "보낼 수만(Send)" 있으면 됩니다.
>
> 다음은 컴파일러로 직접 검증한 결과입니다(보조 함수로 트레이트 구현 여부를 강제 확인).
>
> ```rust
> fn assert_send<T: Send>() {}
> fn assert_sync<T: Sync>() {}
>
> fn main() {
>     assert_send::<std::sync::Arc<std::sync::Mutex<i32>>>(); // OK
>     assert_sync::<std::sync::Arc<std::sync::Mutex<i32>>>(); // OK
>     assert_sync::<std::sync::Mutex<i32>>();                 // OK
>     // assert_send::<std::rc::Rc<i32>>();        // 🚨 Rc: !Send
>     // assert_sync::<std::cell::RefCell<i32>>(); // 🚨 RefCell: !Sync
> }
> ```
>
> 주석 처리한 두 줄을 살리면 각각 `E0277`로 컴파일이 거부됩니다. 이 `Send`/`Sync` 검사가 바로 16장에서 약속한 "두려움 없는 동시성"의 정체입니다. 멀티스레드 안전성이 개발자의 머릿속이 아니라 **타입 시스템**에 들어 있습니다.

---

## 17.7 데드락 — 컴파일러도 막지 못하는 영역

여기서 중요한 한계를 짚어야 합니다. Rust는 데이터 레이스를 컴파일 타임에 막지만, **데드락(Deadlock)은 막지 못합니다.** 이것은 버그가 아니라 의도된 설계입니다. 데드락은 메모리 안전성 위반이 아니기 때문입니다. 프로그램이 멈춰 서서 영원히 진행되지 않을 뿐, 잘못된 메모리에 접근하거나 데이터를 오염시키지는 않습니다. Rust의 안전성 보장 범위 밖입니다.

전형적인 데드락은 **두 스레드가 두 개의 락을 반대 순서로 잡을 때** 발생합니다. 스레드 A가 락1을 잡고 락2를 기다리는데, 스레드 B가 락2를 잡고 락1을 기다리면, 둘 다 영원히 멈춥니다.

해법은 단순하고 강력합니다. **모든 스레드가 락을 항상 동일한 순서로 획득**하면 됩니다. 아래는 두 계좌 사이의 이체에서, 항상 ID가 작은 락을 먼저 잡아 데드락을 원천 차단하는 패턴입니다.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

/// 항상 id가 작은 계좌의 락을 먼저 획득해 락 순서를 통일합니다.
fn transfer(a: &Arc<(usize, Mutex<i64>)>, b: &Arc<(usize, Mutex<i64>)>, amount: i64) {
    // 두 락을 잡기 전에, id 기준으로 잠그는 순서를 고정한다.
    let (first, second) = if a.0 < b.0 { (a, b) } else { (b, a) };
    let mut g1 = first.1.lock().unwrap();
    let mut g2 = second.1.lock().unwrap();

    // 실제 이체 방향(a -> b)에 맞춰 부호를 맞춘다.
    if std::ptr::eq(first.as_ref(), a.as_ref()) {
        *g1 -= amount;
        *g2 += amount;
    } else {
        *g2 -= amount;
        *g1 += amount;
    }
}

fn main() {
    let acc1 = Arc::new((0usize, Mutex::new(1000)));
    let acc2 = Arc::new((1usize, Mutex::new(1000)));

    let (a1, a2) = (Arc::clone(&acc1), Arc::clone(&acc2));
    let t1 = thread::spawn(move || {
        for _ in 0..100 {
            transfer(&a1, &a2, 1); // acc1 -> acc2
        }
    });

    let (b1, b2) = (Arc::clone(&acc1), Arc::clone(&acc2));
    let t2 = thread::spawn(move || {
        for _ in 0..100 {
            transfer(&b2, &b1, 1); // acc2 -> acc1, 락 순서는 동일하게 통일됨
        }
    });

    t1.join().unwrap();
    t2.join().unwrap();

    let sum = *acc1.1.lock().unwrap() + *acc2.1.lock().unwrap();
    println!("총합 보존: {sum}"); // 총합 보존: 2000
}
```

`t1`은 `acc1→acc2` 방향으로, `t2`는 `acc2→acc1` 방향으로 이체하지만, `transfer` 내부에서 항상 ID가 작은 락(acc1)을 먼저 잡으므로 락 획득 순서는 두 스레드 모두 동일합니다. 데드락은 발생하지 않고, 총합 2000은 완벽히 보존됩니다.

### `parking_lot` — 표준 라이브러리를 넘어서

실무에서는 `std::sync::Mutex` 대신 **`parking_lot` 크레이트**의 `Mutex`/`RwLock`을 즐겨 씁니다. 차이는 다음과 같습니다.

* **`unwrap()`이 필요 없습니다.** `parking_lot::Mutex`는 중독(poisoning) 개념이 없어 `lock()`이 가드를 바로 반환합니다(`m.lock()`만으로 끝). 코드가 깔끔해집니다.
* **더 작고 빠릅니다.** 경합이 없을 때는 단 1바이트 크기에, 스핀락 후 OS 대기로 넘어가는 적응형(adaptive) 전략을 씁니다.
* **공정성·타임아웃 옵션**과 `try_lock_for` 같은 편의 기능을 제공합니다.

다만 표준 라이브러리만으로도 정확성에는 부족함이 없으니, `parking_lot`은 성능·편의가 필요할 때 도입하는 선택지로 기억하면 됩니다.

---

## 17.8 실전 예제 (Hands-on): `multiprocessing.Manager` → `Arc<Mutex<HashMap>>`

이제 가장 현실적인 리팩토링을 해봅시다. 파이썬에서 여러 워커가 공유 캐시를 채우는 코드는 보통 `multiprocessing.Manager`를 씁니다.

```python
from __future__ import annotations
import time
from concurrent.futures import ThreadPoolExecutor
from multiprocessing import Manager

def get_or_compute(cache, key: str) -> str:
    if key in cache:                 # 🚨 체크와 갱신 사이에 락이 없으면 레이스
        return cache[key]
    time.sleep(0.01)                 # 비싼 DB 조회 흉내
    value = f"profile-of-{key}"
    cache[key] = value
    return value

def main() -> None:
    with Manager() as manager:
        cache = manager.dict()       # 프로세스 간 공유 dict (내부적으로 프록시 + IPC)
        with ThreadPoolExecutor(max_workers=6) as pool:
            futures = [
                pool.submit(get_or_compute, cache, f"user:{i % 3}")
                for i in range(6)
            ]
            for f in futures:
                print(f.result())
        print(f"최종 캐시 엔트리 수: {len(cache)}")
```

`manager.dict()`는 별도의 매니저 프로세스가 실제 dict를 들고, 워커들은 **프록시 객체를 통해 매 접근마다 IPC(피클 직렬화 + 소켓 통신)**로 데이터를 주고받습니다. 안전하긴 하지만, 모든 읽기/쓰기가 프로세스 경계를 넘는 직렬화 비용을 치릅니다. 게다가 위 코드의 `get_or_compute`는 "있는지 확인 → 없으면 채움" 사이에 락이 없어, 두 워커가 동시에 같은 키를 보고 둘 다 비싼 연산을 수행하는 레이스가 그대로 남아 있습니다.

Rust로 옮기면, 진짜 스레드들이 **하나의 힙 메모리를 직접 공유**하므로 IPC도 직렬화도 없습니다.

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

/// 여러 스레드가 공유하는 인메모리 캐시. Clone하면 같은 내부 데이터를 가리킵니다.
#[derive(Clone)]
struct SharedCache {
    inner: Arc<RwLock<HashMap<String, String>>>,
}

impl SharedCache {
    fn new() -> Self {
        SharedCache {
            inner: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    fn get(&self, key: &str) -> Option<String> {
        // 읽기 락: 여러 스레드가 동시에 캐시를 조회할 수 있다.
        let map = self.inner.read().unwrap();
        map.get(key).cloned()
    }

    /// 캐시 미스일 때만 compute를 수행하고 결과를 채워 넣습니다.
    fn get_or_compute(&self, key: &str, compute: impl FnOnce() -> String) -> String {
        // 1. 낙관적 읽기: 대부분의 호출은 여기서 끝난다 (읽기 락은 동시성 허용).
        if let Some(v) = self.get(key) {
            return v;
        }

        // 2. 미스일 때만 비싼 연산 수행 (락을 잡지 않은 상태에서!)
        let value = compute();

        // 3. 쓰기 락으로 삽입. 그 사이 다른 스레드가 채웠을 수 있으므로
        //    entry().or_insert()로 한 번 더 확인해 중복 연산을 흡수한다.
        let mut map = self.inner.write().unwrap();
        map.entry(key.to_string()).or_insert(value).clone()
    }
}

fn main() {
    let cache = SharedCache::new();
    let mut handles = vec![];

    for id in 0..6 {
        let cache = cache.clone(); // Arc 카운트만 증가, 내부 HashMap은 공유
        handles.push(thread::spawn(move || {
            let key = format!("user:{}", id % 3); // 3개 키를 6개 스레드가 경합
            let value = cache.get_or_compute(&key, || {
                thread::sleep(Duration::from_millis(10)); // DB 조회 흉내
                format!("profile-of-{key}")
            });
            (id, key, value)
        }));
    }

    for h in handles {
        let (id, key, value) = h.join().unwrap();
        println!("thread {id}: {key} -> {value}");
    }

    let size = cache.inner.read().unwrap().len();
    println!("최종 캐시 엔트리 수: {size}"); // 최종 캐시 엔트리 수: 3
}
```

실행 결과는 키가 3개(`user:0`, `user:1`, `user:2`)이므로 캐시 엔트리도 정확히 3개입니다.

이 코드에서 주목할 설계 포인트가 셋 있습니다.

1. **락 보유 시간 최소화.** `compute()`(10ms의 비싼 연산)는 **어떤 락도 잡지 않은 상태**에서 호출됩니다. 만약 쓰기 락을 잡은 채로 `compute()`를 호출하면, 그 10ms 동안 모든 다른 스레드가 캐시 전체에서 차단됩니다. 임계 구역은 "삽입" 한 줄로 줄여야 합니다.
2. **이중 확인(double-check).** 쓰기 락을 얻은 시점에 다른 스레드가 이미 같은 키를 채웠을 수 있으므로, `entry().or_insert()`로 한 번 더 확인합니다. 파이썬 예제에 없던 정확성을 구조적으로 챙겼습니다.
3. **`#[derive(Clone)]` + 내부 `Arc`.** `SharedCache`를 복제해도 내부 `HashMap`은 공유됩니다. 이것이 "공유 핸들" 패턴으로, 깔끔한 API 뒤에 `Arc`를 숨기는 관용구입니다.

**성능 비교의 맥락.** `manager.dict()`의 모든 접근은 피클링과 IPC를 거칩니다. 캐시 적중(hit) 한 번이 마이크로초 단위가 아니라 수십~수백 마이크로초가 들 수 있습니다. 반면 Rust의 `RwLock<HashMap>` 읽기 적중은 락 획득 + 해시 조회로, **수십 나노초** 수준입니다. 측정 맥락(`--release` 빌드, 적중률 높은 워크로드)에서 캐시 적중 경로만 비교하면 보통 수백 배에서 1000배 이상의 차이가 납니다. 물론 이것은 "프로세스 간 공유"라는 파이썬의 제약(GIL 회피를 위해 멀티프로세스를 쓸 수밖에 없었던)과 "스레드 간 메모리 직접 공유"라는 Rust의 자유가 만든 구조적 차이이지, 단순한 언어 속도 차이가 아닙니다.

---

## 17.9 Best Practices & 안티패턴

먼저 "무엇을 언제 쓰는가"를 한 장의 표로 정리합니다. 이 결정 매트릭스를 손에 익히면 동기화 도구 선택의 90%가 끝납니다.

| 상황 | 권장 도구 | 이유 |
|------|-----------|------|
| 스레드 간 단방향 데이터 전달 | `mpsc::channel` (16장) | 소유권 이동으로 공유 자체를 회피 |
| 단일 정수/불리언 카운터 | `AtomicUsize` / `AtomicBool` | 락 오버헤드 없이 원자 연산 |
| 복합 데이터, 읽기·쓰기 비슷 | `Mutex<T>` | 가장 단순하고 예측 가능 |
| 복합 데이터, 읽기 압도적 다수 | `RwLock<T>` | 다중 읽기 동시 허용 |
| 여러 스레드가 같은 락 공유 | `Arc<Mutex<T>>` / `Arc<RwLock<T>>` | 공유 소유권 + 동기화 |
| 데이터 병렬(map/reduce류) | `rayon` | 공유 상태 없이 병렬화 |
| 락 편의·성능·공정성 필요 | `parking_lot` | `unwrap` 불필요, 더 빠름 |

**이렇게 하라:**

* **`Mutex`로 시작하라.** 기본값은 `Mutex`입니다. 프로파일링으로 읽기 경합이 병목임을 증명했을 때만 `RwLock`으로 바꾸십시오. 단순 정수 카운터라면 `AtomicUsize`가 더 낫습니다.
* **임계 구역을 최소화하라.** 락을 잡는 시간은 짧을수록 좋습니다. 비싼 연산(I/O, 계산)은 락 밖에서 수행하고, 락 안에서는 짧은 메모리 갱신만 하십시오.
* **공유 핸들 패턴으로 `Arc`를 캡슐화하라.** 위 `SharedCache`처럼 `Arc<Mutex<T>>`를 구조체 안에 숨기고 `Clone`을 derive하면, 호출부가 동기화 디테일을 몰라도 됩니다.
* **블록 스코프로 락을 빨리 풀어라.** `{ let g = m.lock().unwrap(); /* 짧게 */ }`처럼 명시적 블록으로 가드의 수명을 좁혀, 필요 이상으로 락을 들고 있지 마십시오. `drop(guard)`로 명시적 해제도 가능합니다.
* **`Arc::clone(&x)`을 선호하라.** `x.clone()`보다 `Arc::clone(&x)`이 "이건 깊은 복사가 아니라 참조 카운트 증가"임을 코드에 드러냅니다. clippy의 `clippy::clone_on_ref_ptr` 린트도 이를 권장합니다.
* **락 순서를 통일하라.** 여러 락을 잡아야 한다면, 프로그램 전체에서 항상 동일한 순서로 획득하는 규칙을 문서화하고 지키십시오.

**이건 피하라:**

* **락을 잡은 채로 또 다른 락을 부르는 함수 호출.** 어떤 락 순서로 잡히는지 추적할 수 없게 되어 데드락의 온상이 됩니다. 락을 든 상태에서 외부 콜백·트레이트 메서드를 호출하지 마십시오.
* **과도한 잠금(over-locking).** 모든 것을 거대한 단일 `Mutex<AppState>` 하나로 보호하면, 사실상 GIL을 재발명하는 꼴입니다. 독립적으로 갱신되는 데이터는 별도의 락으로 쪼개(lock splitting) 경합을 줄이십시오.
* **`Arc<Mutex<T>>`를 반사적으로 쓰기.** 16장의 채널이나 `rayon`(데이터 병렬 처리)으로 풀리는 문제에 습관적으로 공유 상태를 끌어들이지 마십시오. 공유 상태는 정말 공유가 필요할 때의 도구입니다.

---

## 17.10 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: 락을 잡은 채 `.await` 하기 (다음 장의 예고편).**
18장에서 비동기를 다룰 때 가장 많이 터지는 실수입니다. `std::sync::Mutex`의 가드를 들고 `.await`를 호출하면, 그 await 지점에서 태스크가 다른 스레드로 옮겨갈 수 있는데 가드는 스레드 간 이동이 안 되거나(`!Send`), 락을 든 채로 다른 태스크에 양보해 데드락을 유발합니다. 비동기 코드에서는 `tokio::sync::Mutex`를 쓰거나, await 이전에 락을 풀어야 합니다.

```rust
// 🚨 안티패턴 (비동기): 가드를 들고 await
// let guard = std_mutex.lock().unwrap();
// some_async_fn().await;  // 가드가 await를 가로질러 살아 있음 -> 위험
//
// ✅ 올바른 방법 1: await 전에 락 해제
// let value = { let g = std_mutex.lock().unwrap(); g.clone() }; // 여기서 가드 drop
// some_async_fn_with(value).await;
```

**함정 2: 락을 잡은 채 긴 작업 수행.**
파이썬 개발자는 `with lock:` 블록을 넓게 잡는 데 익숙합니다(GIL이 어차피 직렬화하니까). Rust에서 이러면 진짜 병렬성을 죽입니다. 위 캐시 예제처럼 비싼 `compute()`는 반드시 락 밖에서 호출하십시오. "락은 데이터를 만지는 그 순간에만"이 원칙입니다.

**함정 3: 데드락은 컴파일러가 안 막아준다.**
"Rust니까 동시성 버그는 다 막아주겠지"라는 방심이 가장 위험합니다. 데이터 레이스는 막지만 데드락, 라이브락, 락 경합으로 인한 성능 저하는 여러분의 몫입니다. 락 순서 규칙을 세우고, 의심되면 `parking_lot`의 `try_lock_for`로 타임아웃을 걸어 교착을 탐지하십시오.

**함정 4: 같은 스레드에서 같은 `Mutex`를 두 번 잠그기.**
`std::sync::Mutex`는 재진입(reentrant)이 아닙니다. 한 스레드가 이미 잡은 락을 같은 스레드에서 다시 `lock()`하면, 가드가 해제되기 전이라 **자기 자신과 데드락**에 빠집니다. 락을 잡은 메서드가 다시 락을 잡는 다른 메서드를 호출하는 구조를 경계하십시오.

**함정 5: 락 중독(Poisoning)을 모르고 지나치기.**
락을 잡은 스레드가 패닉하면 `std::sync::Mutex`는 "중독"됩니다. 이후 `lock()`은 `Err(PoisonError)`를 반환합니다. 이것은 "보호하던 데이터가 일관성이 깨진 상태일 수 있다"는 경고입니다. 무지성 `unwrap()`은 중독 시 전파되는 패닉이 되므로, 복구가 필요하면 명시적으로 처리하십시오.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let d = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut guard = d.lock().unwrap();
        guard.push(4);
        panic!("작업 중 패닉!"); // 락을 든 채 패닉 -> 뮤텍스가 '중독'된다
    });
    let _ = handle.join(); // 패닉을 흡수

    // 중독된 뮤텍스를 다시 잠그면 PoisonError를 반환한다.
    let result = data.lock();
    match result {
        Ok(g) => println!("정상: {:?}", *g),
        Err(poisoned) => {
            // into_inner()로 데이터 자체는 복구할 수 있다.
            let recovered = poisoned.into_inner();
            println!("중독되었지만 복구한 데이터: {:?}", *recovered); // [1, 2, 3, 4]
        }
    }
}
```

---

## 맺음말

16장에서 우리는 메시지 패싱으로 스레드 간 통신의 우아함을 배웠고, 이번 17장에서는 그것만으로는 부족한 진짜 공유 상태의 세계를 다뤘습니다. 핵심은 하나입니다. Rust는 "락을 거는 규율"을 양심에서 타입 시스템으로 옮겼습니다. 데이터를 `Mutex`/`RwLock` 안에 가두어 락 없이는 접근조차 못 하게 하고, `Arc`로 소유권을 안전하게 나누며, `Send`/`Sync`라는 자동 트레이트로 무엇이 스레드를 건널 수 있는지를 컴파일 타임에 판정합니다. `Rc`를 스레드로 보내려는 단 한 줄의 실수가 `E0277`로 박멸되는 것을 직접 확인했고, 동시에 데드락은 여전히 개발자의 책임으로 남아 있다는 한계도 분명히 했습니다.

지금까지 본 모든 동시성은 **OS 스레드** 위에 서 있었습니다. 스레드 하나는 메가바이트 단위의 스택을 차지하므로, 수만 개의 동시 커넥션을 OS 스레드로 감당하기는 어렵습니다. 그래서 현대 백엔드는 수만 개의 가벼운 "태스크"를 소수의 스레드 위에서 협력적으로 굴리는 비동기(Async) 모델로 갑니다. 다음 18장에서는 파이썬의 `asyncio` 이벤트 루프와 Rust의 `tokio` 런타임을 정면으로 비교하고, `Future`가 어떻게 상태 기계(State Machine)로 변환되는지, 그리고 이번 장에서 예고한 "락을 든 채 `.await` 하는" 함정이 왜 그토록 위험한지를 파헤칩니다.

---

## References

- https://doc.rust-lang.org/book/ch16-03-shared-state.html
- https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html
- https://doc.rust-lang.org/std/sync/struct.Arc.html
- https://doc.rust-lang.org/std/sync/struct.Mutex.html
- https://doc.rust-lang.org/std/sync/struct.RwLock.html
- https://doc.rust-lang.org/std/marker/trait.Send.html
