# Chapter 16. 멀티스레딩: `threading` 모듈의 한계 돌파

10년 차 파이썬 개발자라면 'GIL(Global Interpreter Lock)'이라는 단어만 들어도 진저리가 날 것입니다. 멀티 코어 CPU 시대가 열린 지 십수 년이 지났지만, 파이썬은 여전히 한 번에 단 하나의 스레드만 파이썬 바이트코드를 실행할 수 있도록 인터프리터 전체에 자물쇠(Lock)를 걸어둡니다. 10코어 머신에서 `ThreadPoolExecutor(max_workers=10)`로 CPU 바운드 작업을 돌려놓고, `htop`을 켰을 때 코어 하나만 100%이고 나머지 아홉 개가 놀고 있는 광경을 본 적이 있다면 — 이 챕터는 여러분을 위한 것입니다.

파이썬에서 CPU 바운드 작업을 진짜로 병렬 처리하려면 `multiprocessing`을 써서 메모리를 통째로 복제한 무거운 프로세스들을 띄워야만 했습니다. 프로세스 하나를 `fork`하는 비용, 인자와 결과를 `pickle`로 직렬화해 파이프로 주고받는 IPC 오버헤드, 그리고 "왜 내 객체는 pickle이 안 되지?"라는 30분짜리 디버깅 — 이 모든 것이 GIL을 우회하기 위해 우리가 치러온 대가입니다. (I/O 바운드 작업은 예외입니다. 네트워크나 디스크를 기다리는 동안에는 GIL이 풀리므로 `threading`이 잘 동작합니다.)

Rust는 GIL이 존재하지 않습니다. 모든 스레드는 OS 레벨의 진짜 커널 스레드이며, CPU 코어 개수만큼 동시에 기계어를 실행합니다. 더 중요한 것은, 이 스레드들이 **컴파일 타임에 검증된** 데이터 레이스(Data Race) 안전성을 보장받는다는 점입니다. 파이썬에서 락을 깜빡 잊어 데이터가 조용히 꼬이는 일이, Rust에서는 `cargo build`가 거부하는 컴파일 에러로 바뀝니다. Rust 커뮤니티는 이를 **두려움 없는 동시성(Fearless Concurrency)** 이라 부릅니다.

Chapter 15에서 우리는 이터레이터 체이닝이 어셈블리에서 0의 비용으로 증발하는 것을 보았고, `rayon`의 `par_iter()`가 "이터레이터를 병렬로 흘려보내기"를 한 줄로 해낸다는 것을 잠깐 맛보았습니다. 이번 챕터는 그 약속을 회수합니다. `std::thread`로 진짜 OS 스레드를 띄우고, 채널(Channel)로 메시지를 주고받고, `rayon`으로 데이터 병렬성을 한 줄로 얻어내며, 그 모든 안전성의 뿌리인 `Send`/`Sync` 트레이트를 들여다봅니다. 그리고 마지막에는 소수 세기라는 동일한 CPU 바운드 작업을 Python `threading` / `multiprocessing` / Rust `thread` / `rayon`으로 측정해, GIL이 만든 직렬화의 벽과 진짜 하드웨어 병렬성의 차이를 숫자로 마주합니다.

---

## 16.1 GIL의 해부학 — 왜 파이썬은 스레드로 빨라지지 않는가

GIL을 그냥 "파이썬의 한계"라고 욕하기 전에, **왜 존재하는가**를 정확히 알아야 합니다. GIL은 게으름의 산물이 아니라 CPython의 메모리 관리 모델이 내린 합리적(그러나 비싼) 선택입니다.

### 참조 카운팅과 스레드 안전성

CPython의 모든 객체는 헤더에 참조 카운트(`ob_refcnt`)를 들고 있습니다. Chapter 4에서 본 그 카운터입니다. 변수에 대입할 때마다 `Py_INCREF`로 카운트가 1 늘고, 스코프를 벗어나거나 `del` 하면 `Py_DECREF`로 1 줄며, 0이 되는 순간 객체가 해제됩니다.

```python
import sys

a = [1, 2, 3]
b = a              # refcount += 1
print(sys.getrefcount(a))  # 보통 3 (a, b, getrefcount의 임시 참조)
```

문제는 이 `ob_refcnt += 1`이 원자적(atomic)이지 않다는 것입니다. 두 스레드가 동시에 같은 객체의 카운트를 증가시키면 — 읽고(read), 더하고(modify), 쓰는(write) 세 단계 사이에 다른 스레드가 끼어들면 — 카운트가 어긋납니다. 카운트가 실제보다 작게 기록되면 객체가 아직 쓰이고 있는데도 해제되어 use-after-free가, 크게 기록되면 영원히 해제되지 않는 메모리 누수가 발생합니다.

가장 단순한 해법은 "인터프리터 전체에 락 하나를 걸어, 한 번에 한 스레드만 바이트코드를 실행하게 하는 것"입니다. 이것이 GIL입니다. 모든 객체마다 락을 거는 세밀한(fine-grained) 방식은 락 획득/해제 오버헤드가 너무 커서 단일 스레드 성능을 심하게 떨어뜨렸기에, CPython은 인터프리터당 락 하나라는 거친(coarse-grained) 선택을 했습니다. 덤으로, GIL은 스레드 안전하지 않은 수많은 C 확장(C extension)을 보호하는 방패 역할도 합니다.

### I/O 바운드는 괜찮고 CPU 바운드는 막힌다

GIL은 "바이트코드를 실행하는 동안"만 잡혀 있습니다. 스레드가 `socket.recv()`나 `time.sleep()`처럼 블로킹 시스템 콜에 진입하면, CPython은 GIL을 놓아줍니다(release). 그래서 수백 개의 소켓을 동시에 기다리는 I/O 바운드 워크로드는 `threading`으로 잘 처리됩니다 — 어차피 대부분의 시간을 커널에서 대기하니까요.

하지만 CPU 바운드 작업은 다릅니다. 소수를 세거나 해시를 계산하는 동안에는 계속 바이트코드를 실행해야 하므로 GIL을 놓지 않습니다. CPython은 일정 간격(기본 5ms, `sys.setswitchinterval()`로 조회·변경)마다 "이제 GIL을 놓아달라"는 플래그를 세워 현재 스레드가 다음 바이트코드 경계에서 GIL을 양보하게 만들고, 대기하던 다른 스레드가 그것을 잡아 실행을 이어갑니다. 즉 스레드들이 GIL이라는 단 하나의 바통을 5ms마다 주고받으며 돌아가는 구조입니다. 이것은 **병렬(parallel)이 아니라 동시(concurrent)** 실행일 뿐입니다. 코어가 10개여도 매 순간 단 하나의 스레드만 계산합니다. 오히려 바통을 뺏고 빼앗기는 컨텍스트 스위칭과, 깨어난 스레드가 다시 GIL을 획득하려 경합하는 비용 때문에, CPU 바운드 작업에 스레드를 늘리면 단일 스레드보다 **느려지는** 경우도 흔합니다(이 챕터의 벤치마크에서 0.99x로 직접 봅니다). 멀티코어 머신에서는 여러 코어의 스레드가 동시에 GIL을 노리며 깨었다 자기를 반복하는 "GIL 경합(GIL battle)" 현상으로, 코어가 많을수록 오히려 손해를 보기도 합니다.

### free-threaded(nogil) 동향

이 책을 쓰는 시점(2026년)에 CPython 3.13은 PEP 703에 따라 GIL을 제거한 **free-threaded** 빌드를 실험적으로 제공합니다(`python3.13t`). GIL을 없애는 대신 참조 카운팅을 편향 참조 카운팅(biased reference counting)과 원자적 연산으로 재설계했습니다. 방향은 옳지만, 단일 스레드 성능이 다소 떨어지고, 수많은 C 확장이 아직 호환되지 않으며, 기본 빌드도 아닙니다. 즉 "곧 GIL이 사라진다"는 기대로 실무 결정을 미룰 단계는 아닙니다. 게다가 GIL을 없앤다고 해서 파이썬이 데이터 레이스로부터 안전해지는 것도 아닙니다 — 오히려 그동안 GIL이 우연히 막아주던 레이스들이 드러납니다. 바로 그 지점에서 Rust의 접근이 빛납니다. Rust는 "락을 없애자"가 아니라 "레이스를 컴파일 타임에 타입으로 금지하자"를 택했습니다.

### multiprocessing의 청구서

GIL을 우회하는 정통 해법은 `multiprocessing`입니다. 하지만 프로세스는 스레드보다 훨씬 비쌉니다.

```python
from multiprocessing import Pool

def count_primes(args: tuple[int, int]) -> int:
    start, end = args
    return sum(1 for n in range(start, end) if is_prime(n))

if __name__ == "__main__":
    ranges = [(2, 500_000), (500_000, 1_000_000),
              (1_000_000, 1_500_000), (1_500_000, 2_000_000)]
    with Pool(4) as p:
        total = sum(p.map(count_primes, ranges))
```

겉보기에는 깔끔하지만 청구서가 뒤따릅니다.

1. **프로세스 생성/fork 비용**: 각 워커는 독립된 OS 프로세스입니다. `fork`(Linux)는 copy-on-write로 다소 가볍지만, `spawn`(macOS 기본, Windows 유일)은 인터프리터를 통째로 새로 띄우고 모듈을 다시 import합니다.
2. **메모리 복제**: 프로세스는 메모리를 공유하지 않습니다. 부모의 10GB 데이터셋을 워커들이 봐야 한다면, copy-on-write가 깨지는 순간(파이썬은 객체를 읽기만 해도 refcount를 건드려 페이지를 더럽힙니다) 메모리가 폭증합니다.
3. **pickle 직렬화 IPC**: `p.map(fn, args)`에서 `fn`과 `args`, 그리고 반환값이 전부 `pickle`로 직렬화되어 파이프를 통해 오갑니다. 람다, 로컬 함수, 열린 파일 핸들, 락은 pickle이 안 됩니다. 큰 numpy 배열을 주고받으면 직렬화/역직렬화 비용이 계산 자체를 압도하기도 합니다.

요약하면, 파이썬에서 진짜 병렬성을 얻으려면 **메모리 공유를 포기하고 프로세스 경계 너머로 데이터를 복사**해야 합니다. Rust는 정반대로 갑니다 — 스레드끼리 메모리를 공유하되, 그 공유를 컴파일러가 검증합니다.

---

## 16.2 The Rust Mental Model — 스레드는 OS 자원이고, 데이터는 소유된다

Rust의 동시성 설계를 한 문장으로 요약하면 이렇습니다. **"스레드 간에 무언가를 주고받으려면, 그 무언가의 소유권(Ownership) 또는 안전한 참조 권한을 타입 시스템에 증명해야 한다."**

파이썬의 멘탈 모델은 "모든 스레드는 같은 전역 객체 그래프를 본다. 충돌은 개발자가 락으로 알아서 피하라"입니다. Rust의 멘탈 모델은 "스레드로 넘어가는 모든 값은 소유권 규칙을 따른다. 컴파일러가 데이터 레이스 가능성을 발견하면 빌드를 거부한다"입니다.

이 차이를 떠받치는 두 개의 마커 트레이트(marker trait)가 있습니다. 자세한 것은 다음 절과 Chapter 17로 미루되, 직관만 먼저 잡겠습니다.

- **`Send`**: 이 타입의 값을 다른 스레드로 **소유권을 넘겨** 보낼 수 있는가? (대부분의 타입이 `Send`입니다. `Rc<T>`는 아닙니다.)
- **`Sync`**: 이 타입의 **참조(`&T`)를 여러 스레드가 동시에** 공유해도 안전한가?

`std::thread::spawn`은 자신에게 넘겨지는 클로저가 `Send + 'static`일 것을 요구합니다. 즉 "스레드로 보내도 안전하고(`Send`), 메인 함수가 끝난 뒤에도 살아있을 수 있는(`'static`) 데이터만 캡처하라"는 계약입니다. 이 계약 하나가 댕글링 포인터와 데이터 레이스를 동시에 차단합니다.

| Python 개념 | Rust 개념 | 핵심 차이 |
|---|---|---|
| `threading.Thread` | `std::thread::spawn` | GIL로 직렬화됨 vs 진짜 병렬 OS 스레드 |
| `multiprocessing.Process` | (불필요) | 프로세스 복제·pickle vs 스레드 메모리 공유 |
| `queue.Queue` | `std::sync::mpsc::channel` | 런타임 안전 vs 소유권 이동으로 컴파일 타임 안전 |
| `threading.Lock` (Ch17) | `Mutex<T>` (Ch17) | 깜빡 잊으면 레이스 vs 데이터를 락이 감싸 강제 |
| `concurrent.futures` 풀 | `rayon` 스레드 풀 | 수동 분할 vs work-stealing 자동 부하 분산 |
| (GIL이 막아줌) | `Send` / `Sync` 트레이트 | 런타임 우연 vs 컴파일 타임 보장 |
| 데몬 스레드/`join()` | `JoinHandle::join()` | 반환값 못 받음 vs 타입 있는 반환값 회수 |

이 표가 챕터 전체의 지도입니다. 이제 한 줄씩 코드로 내려가 봅니다.

---

## 16.3 진짜 하드웨어 스레드 띄우기 (`thread::spawn`)

* **The Python Way (Multithreading)**

  ```python
  import threading
  import time

  def worker(num: int) -> None:
      print(f"Worker {num} is running")
      time.sleep(1)  # I/O 대기처럼 GIL을 놓는 구간

  threads: list[threading.Thread] = []
  for i in range(5):
      t = threading.Thread(target=worker, args=(i,))
      threads.append(t)
      t.start()

  for t in threads:
      t.join()
  # sleep(I/O 대기) 중에는 GIL이 풀려 동시에 잘 돕니다.
  # 그러나 sleep을 무거운 계산으로 바꾸면 전혀 빨라지지 않습니다.
  ```

* **The Rust Implementation (`std::thread`)**

  Rust에서는 `thread::spawn`에 **클로저(Closure)** 를 넘겨 새 OS 스레드를 실행합니다.

  ```rust
  use std::thread;
  use std::time::Duration;

  fn main() {
      let mut handles = vec![];

      for i in 0..5 {
          // move: 환경의 변수(i)의 소유권을 스레드 안으로 이동시킴
          let handle = thread::spawn(move || {
              println!("Worker {i} is running");
              thread::sleep(Duration::from_secs(1));
          });
          handles.push(handle);
      }

      // 메인 스레드가 끝나기 전에 모든 자식 스레드를 기다립니다.
      for handle in handles {
          handle.join().unwrap();
      }
  }
  ```

`thread::spawn`은 `JoinHandle<T>`를 반환합니다. 여기서 `T`는 클로저의 반환 타입입니다. 파이썬의 `Thread.join()`은 `None`을 반환할 뿐이라(결과를 받으려면 `queue`나 공유 변수를 써야 했습니다), Rust의 `join()`은 스레드의 결과를 **타입 그대로 회수**합니다.

```rust
use std::thread;

fn main() {
    // 각 스레드가 부분합을 계산해 u64로 반환
    let handles: Vec<_> = (0..4)
        .map(|k| thread::spawn(move || {
            let (start, end) = (k * 250_000u64, (k + 1) * 250_000);
            (start..end).sum::<u64>()
        }))
        .collect();

    // join()의 반환 타입이 thread::Result<u64> 이므로 결과를 그대로 합산
    let total: u64 = handles.into_iter().map(|h| h.join().unwrap()).sum();
    println!("total = {total}"); // total = 499999500000
}
```

`join()`이 `Result`를 반환하는 이유는, 스레드가 패닉(panic)으로 죽었을 수 있기 때문입니다. 자식 스레드의 패닉은 메인 스레드를 죽이지 않습니다 — `join()`에서 `Err`로 회수됩니다. 이것은 파이썬에서 스레드 안의 예외가 메인에 전파되지 않고 조용히 사라지던 것과 대조됩니다. Rust는 그 실패를 `Result`로 명시적으로 손에 쥐여 줍니다.

### `move` 키워드와 컴파일러의 거부

위 코드에서 `move`가 왜 들어갔을까요? `move`를 빼면 클로저는 `i`(또는 `data`)를 **참조로 캡처**하려 듭니다. 그런데 `thread::spawn`은 스레드가 부모 함수보다 오래 살 수 있다고 가정하므로(`'static` 요구), 부모 스택의 변수를 참조하는 것을 허용할 수 없습니다. 메인 스레드가 먼저 끝나 변수가 해제(Drop)되면 자식 스레드가 죽은 메모리를 가리키게 되니까요(댕글링 포인터). 직접 위반해 봅시다.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let handle = thread::spawn(|| {       // 🚨 move 없음
        println!("{:?}", data);           // 부모 스택의 data를 빌리려 함
    });
    handle.join().unwrap();
}
```

`rustc`가 내뱉는 에러는 정확히 그 위험을 짚습니다.

```text
error[E0373]: closure may outlive the current function, but it borrows `data`,
              which is owned by the current function
 --> a_borrow.rs:5:32
  |
5 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `data`
6 |         println!("{:?}", data);
  |                          ---- `data` is borrowed here
  |
note: function requires argument type to outlive `'static`
help: to force the closure to take ownership of `data` (and any other referenced
      variables), use the `move` keyword
  |
5 |     let handle = thread::spawn(move || {
  |                                ++++
```

`closure may outlive the current function` — 클로저가 부모 함수보다 오래 살 수 있다. `to force the closure to take ownership ... use the move keyword` — 해결책까지 친절하게 제시합니다. `move`를 붙이면 `data`의 소유권이 클로저 안으로 이동하여, 부모가 끝나도 데이터는 스레드가 소유하므로 안전합니다. 파이썬이라면 이 코드는 컴파일은 물론 실행도 되고, 운이 나쁘면 프로덕션에서 간헐적으로 터졌을 것입니다.

> **💡 Under the Hood: OS 스레드의 진짜 비용**
>
> `thread::spawn`은 `pthread_create`(POSIX) 또는 `CreateThread`(Windows)를 호출하는 얇은 래퍼입니다. 새 스레드는 기본 **2MiB의 스택**(Linux 기준, `RUST_MIN_STACK`로 조정 가능)을 가상 메모리에 예약하고, 커널 스케줄러에 등록됩니다. 이 스택은 가상 주소 공간만 예약하고 실제 물리 페이지는 접근할 때 할당되므로(lazy), 1,000개를 띄워도 즉시 2GiB 물리 메모리를 먹지는 않습니다.
>
> 그래도 스레드는 공짜가 아닙니다. 생성/소멸에 수 마이크로초, 컨텍스트 스위칭에 수백 나노초~마이크로초가 들고, 커널 스케줄러의 런큐(run queue)에 부담을 줍니다. "요청마다 스레드 하나"는 코어가 10개인데 스레드가 10,000개면 스케줄러가 스레드를 바꿔 끼우느라 정작 일을 못 하는 스래싱(thrashing)에 빠집니다. 그래서 실무에서는 코어 수에 맞춘 **스레드 풀**(다음 절의 `rayon`이 정확히 이것)을 씁니다. 파이썬의 `multiprocessing.Process`는 여기에 가상 메모리 공간 전체와 인터프리터 상태 복제까지 얹은, 훨씬 더 비싼 단위입니다.

---

## 16.4 공유 메모리 대신 메시지 패싱 (Channels, `mpsc`)

멀티스레드에서 가장 안전하게 데이터를 주고받는 방식은, 한 스레드에서 다른 스레드로 값을 **"보내는(Send)"** 것입니다. 파이썬에서는 `queue.Queue`를 씁니다. Rust에서는 이를 **채널(Channel)** 이라 부릅니다.

표준 라이브러리의 `std::sync::mpsc`는 **MPSC(Multiple Producer, Single Consumer)** 채널을 제공합니다. 송신자(Sender)는 여러 개로 복제할 수 있지만, 수신자(Receiver)는 하나입니다.

* **The Python Way (`queue.Queue`)**

  ```python
  import threading
  import queue

  q: queue.Queue[str] = queue.Queue()

  def producer(q: queue.Queue[str]) -> None:
      q.put("Data from worker")

  t = threading.Thread(target=producer, args=(q,))
  t.start()
  print(q.get())  # 블로킹 대기 후 데이터 수신
  t.join()
  ```

* **The Rust Implementation (`std::sync::mpsc`)**

  ```rust
  use std::sync::mpsc;
  use std::thread;

  fn main() {
      // tx: 송신자(Sender), rx: 수신자(Receiver)
      let (tx, rx) = mpsc::channel();

      thread::spawn(move || {
          let val = String::from("Data from worker");

          // send: val의 소유권이 채널을 통해 수신자 쪽으로 완전히 이동
          tx.send(val).unwrap();

          // println!("{val}"); // 🚨 컴파일 에러! 소유권을 잃은 변수 사용 불가
      });

      let received = rx.recv().unwrap(); // 블로킹 대기
      println!("Got: {received}");
  }
  ```

### 소유권이 만드는 완벽한 통신

파이썬의 `Queue`에 가변 리스트를 넣은 뒤, 넣은 쪽에서 그 리스트를 계속 조작하면 받는 쪽은 도중에 바뀐 데이터를 보게 됩니다(공유 가변 상태로 인한 레이스). 객체를 넣는 것은 참조를 넣는 것이기 때문입니다.

Rust에서는 `tx.send(val)`을 호출하는 순간 `val`의 소유권 자체가 채널 너머로 날아갑니다. 송신 스레드가 다시 `val`을 읽거나 수정하려 들면 **컴파일이 거부**됩니다(Chapter 4의 Move Semantics가 그대로 작동합니다). 보낸 쪽과 받는 쪽이 같은 데이터를 동시에 건드릴 가능성 자체가 타입 시스템에서 제거됩니다.

> *"공유 메모리로 통신하지 마라. 통신함으로써 메모리를 공유하라."* — Go의 격언이자 Rust 채널 철학.

### 다중 생산자: `Sender`를 복제하기

`Sender`는 `Clone`이 되므로 여러 스레드에 나눠줄 수 있습니다. 수신자는 모든 송신자가 살아있는 동안 메시지를 받고, **모든 `Sender`가 Drop되면 채널이 닫혀** 수신 측 이터레이션이 자연스럽게 끝납니다.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for id in 0..3 {
        let tx = tx.clone(); // 각 스레드에 줄 송신자 핸들 복제
        thread::spawn(move || {
            // 무거운 계산을 했다고 치고, 결과를 보냄
            tx.send(format!("worker {id} done")).unwrap();
        });
    }
    drop(tx); // 🔑 원본 송신자를 명시적으로 닫아야 rx 이터레이션이 끝난다

    // 채널이 닫힐 때까지 도착하는 순서대로 수신 (순서는 비결정적)
    for msg in rx {
        println!("got: {msg}");
    }
}
```

```text
got: worker 0 done
got: worker 2 done
got: worker 1 done
```

`drop(tx)`를 빠뜨리면 어떻게 될까요? 원본 `tx`가 `main` 끝까지 살아있으므로 채널이 영원히 열린 것으로 간주되어, `for msg in rx`가 마지막 메시지 뒤에도 다음 메시지를 무한정 기다리며 데드락에 빠집니다. 파이썬의 `queue.Queue`에서 센티넬(`None`)을 넣어 "이제 끝"을 알리던 관용구를, Rust에서는 **송신자의 생명(소유권)** 이 대신합니다.

블로킹이 싫다면 `try_recv()`(즉시 반환, 비었으면 `Err(TryRecvError::Empty)`)를, 송신자가 다 죽었는지 알고 싶다면 `recv()`의 `Err(RecvError)`를 확인합니다.

```rust
use std::sync::mpsc::TryRecvError;
// ...
match rx.try_recv() {
    Ok(msg) => println!("받음: {msg}"),
    Err(TryRecvError::Empty) => { /* 아직 없음, 다른 일 하러 감 */ }
    Err(TryRecvError::Disconnected) => { /* 모든 송신자 종료, 루프 탈출 */ }
}
```

### 백프레셔(Backpressure): `sync_channel`로 버퍼 한도 걸기

`mpsc::channel()`은 **무한 버퍼**입니다 — 생산자가 소비자보다 빠르면 메시지가 메모리에 무한정 쌓여 OOM 위험이 생깁니다. 파이썬에서 `queue.Queue(maxsize=N)`로 한도를 걸어 생산자를 잠시 멈춰 세우던(`put`이 블로킹) 그 백프레셔를, Rust에서는 `mpsc::sync_channel(N)`로 얻습니다.

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // 버퍼 용량 2: 버퍼가 차면 send가 소비자를 기다리며 블로킹
    let (tx, rx) = mpsc::sync_channel::<i32>(2);

    let producer = thread::spawn(move || {
        for i in 0..5 {
            tx.send(i).unwrap(); // 버퍼가 꽉 차면 여기서 대기
            println!("sent {i}");
        }
    });

    thread::sleep(Duration::from_millis(50)); // 일부러 소비를 늦춰 버퍼를 채움
    for v in rx {
        println!("recv {v}");
        thread::sleep(Duration::from_millis(20));
    }
    producer.join().unwrap();
}
```

```text
sent 0
sent 1          <- 버퍼(용량 2)가 가득 참. 생산자는 여기서 멈춤
recv 0          <- 소비자가 하나 꺼내자 한 칸 비어서
sent 2          <- 생산자가 다시 진행
recv 1
sent 3
...
```

생산자가 소비 속도에 맞춰 자연스럽게 제동이 걸립니다. 데이터 파이프라인을 짤 때, 무한 버퍼 채널은 "느린 소비자 + 빠른 생산자"라는 흔한 시나리오에서 조용히 메모리를 잡아먹는 폭탄입니다. 용량을 0으로 주면(`sync_channel(0)`) 송신과 수신이 반드시 만나는 랑데부(rendezvous) 채널이 됩니다.

> **💡 Under the Hood: 채널 내부에는 무엇이 있는가**
>
> 채널은 마법이 아니라 **공유 큐 + 동기화 프리미티브**입니다. `mpsc::channel()`은 힙에 내부 버퍼(잠금 보호 또는 락-프리 자료구조)를 만들고, `Sender`와 `Receiver`는 이를 가리키는 핸들입니다. `send`는 값을 큐 뒤에 밀어 넣고 대기 중인 수신자를 깨우며(condition variable / futex), `recv`는 큐가 비어있으면 스레드를 재워(park) CPU를 양보합니다 — 바쁜 대기(busy-wait)가 아닙니다.
>
> 핵심은, `send(val)`에서 넘어가는 것이 값의 **소유권**이라는 점입니다. 데이터가 한 스레드의 손을 완전히 떠나 다른 스레드로 옮겨가므로, 양쪽이 동시에 같은 메모리를 건드릴 일이 없습니다. 그래서 채널로 보낼 수 있는 타입에는 `T: Send` 제약이 붙습니다. 더 높은 성능이 필요하면 `crossbeam` 크레이트의 MPMC(다중 생산자·다중 소비자) 채널이나 락-프리 구현을 씁니다. 비동기 컨텍스트에서는 `tokio::sync::mpsc`를 쓰며, 이는 Chapter 18에서 다룹니다.

---

## 16.5 스코프 스레드 (`thread::scope`)로 빌림을 안전하게 공유

`move`는 강력하지만 때로 과합니다. 큰 `Vec`를 두 스레드가 절반씩 **읽기만** 하고 싶은데, `move`로 소유권을 넘기면 메인은 그 데이터를 다시 못 씁니다. `clone`은 낭비고요. 핵심 통찰은 이렇습니다 — **스레드들이 메인 함수보다 먼저 끝난다고 컴파일러가 확신할 수 있다면, 부모 스택의 데이터를 안전하게 빌려도 된다.** Rust 1.63부터 안정화된 `std::thread::scope`가 정확히 이것을 보장합니다.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let mid = data.len() / 2;

    // scope 블록이 끝나기 전에 내부의 모든 스레드가 자동으로 join됨
    let (left, right) = thread::scope(|s| {
        let h_left = s.spawn(|| data[..mid].iter().sum::<i32>());  // move 불필요!
        let h_right = s.spawn(|| data[mid..].iter().sum::<i32>()); // &data를 빌림
        (h_left.join().unwrap(), h_right.join().unwrap())
    });

    println!("left={left}, right={right}, total={}", left + right);
    // scope가 끝났으므로 data는 다시 온전히 사용 가능
    println!("original still usable: {data:?}");
}
```

```text
left=15, right=40, total=55
original still usable: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

`s.spawn`에 넘긴 클로저에 `move`가 없는데도 컴파일됩니다. `scope`는 클로저들이 캡처한 빌림이 `scope` 블록의 수명을 넘지 않음을 보장하고, 블록을 벗어나기 전에 모든 자식 스레드를 자동으로 `join`하기 때문입니다(명시적 `join`을 안 해도 됩니다). 부모 데이터가 자식보다 오래 산다는 것이 컴파일 타임에 증명되므로, 댕글링 위험이 없습니다.

이것이 "데이터를 통째로 넘기지 않고도 안전하게 공유 참조"하는 첫걸음입니다. 단, 여기서는 모두 **불변(&) 빌림**입니다. 여러 스레드가 동시에 **쓰려면** 락이 필요하고, 그것이 Chapter 17의 `Arc<Mutex<T>>`입니다.

---

## 16.6 데이터 병렬성을 한 줄로: `rayon`

지금까지는 스레드를 손으로 띄웠습니다. 하지만 "이 거대한 컬렉션의 모든 원소에 같은 연산을 적용한다"는 데이터 병렬(data-parallel) 패턴에서는, 스레드 분할·부하 분산·결과 취합을 직접 짜는 것이 번거롭고 실수투성이입니다. 실무 표준 크레이트 **`rayon`** 은 이를 한 글자 수정으로 해결합니다 — `iter()`를 `par_iter()`로, `into_iter()`를 `into_par_iter()`로 바꾸기만 하면 됩니다.

Chapter 15에서 익힌 이터레이터 어댑터(`map`, `filter`, `fold`, `sum`...)가 그대로 병렬 버전으로 존재합니다.

```rust
use rayon::prelude::*;

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    let mut i = 2;
    while i * i <= n {
        if n % i == 0 { return false; }
        i += 1;
    }
    true
}

fn main() {
    // 순차 버전 (Chapter 15)
    let seq = (2..2_000_000u64).filter(|&n| is_prime(n)).count();

    // 병렬 버전: iter -> par_iter 한 곳만 바뀜
    let par = (2..2_000_000u64)
        .into_par_iter()
        .filter(|&n| is_prime(n))
        .count();

    assert_eq!(seq, par);
    println!("found {par} primes");
}
```

`Cargo.toml`에 `rayon = "1"` 한 줄만 추가하면 됩니다. 락도, 채널도, 스레드 핸들도, `join`도 직접 쓸 일이 없습니다. 그런데도 `rayon`은 머신의 모든 코어를 자동으로 활용합니다.

### Work-Stealing: 부하를 스스로 분산한다

`rayon`이 단순한 "범위 N등분"보다 빠른 비결은 **작업 훔치기(Work-Stealing)** 스케줄러입니다. `rayon`은 시작 시 코어 수만큼 워커 스레드 풀을 만듭니다. 작업(예: 큰 범위)을 재귀적으로 절반씩 쪼개(split) 각 워커의 로컬 덱(deque)에 넣고, 자기 일을 끝낸 한가한 워커는 바쁜 워커의 덱 **반대편 끝에서 작업을 훔쳐** 옵니다.

이것이 왜 중요한지는 소수 세기에서 극명합니다. 큰 수일수록 소수 판정에 시간이 더 걸리므로(`while i*i <= n`), 범위를 단순히 10등분하면 **마지막 청크가 첫 청크보다 훨씬 무겁습니다(부하 불균형)**. 수동 분할은 가벼운 청크를 맡은 스레드가 일찍 놀게 되지만, work-stealing은 그 놀던 스레드가 무거운 작업을 훔쳐와 코어를 끝까지 채웁니다. 이 챕터의 벤치마크에서 수동 `std::thread` 분할보다 `rayon`이 더 빠른 이유가 바로 이것입니다.

> **💡 Under the Hood: 진짜 병렬성 vs GIL 직렬화의 타임라인**
>
> 코어가 4개인 머신에서 같은 양의 CPU 작업을 4개로 나눠 돌릴 때, 두 세계의 시간 축은 이렇게 다릅니다.
>
> ```text
> Python threading (GIL):   하드웨어 코어는 4개지만, 매 순간 1개만 실행
>   Core0: [T0][T1][T2][T3][T0][T1]...   <- GIL을 5ms마다 돌려가며 잡음
>   Core1: ........(idle).............    <- 놀고 있음
>   Core2: ........(idle).............    <- 놀고 있음
>   Core3: ........(idle).............    <- 놀고 있음
>   => 총 시간 ≈ 단일 스레드 + 컨텍스트 스위칭 오버헤드 (오히려 느려질 수 있음)
>
> Rust threads / rayon:     4개 코어가 진짜로 동시에 기계어 실행
>   Core0: [====== T0 ======]
>   Core1: [====== T1 ======]
>   Core2: [====== T2 ======]
>   Core3: [====== T3 ======]
>   => 총 시간 ≈ 단일 스레드 / 4 (부하 분산이 완벽하다면)
> ```
>
> 파이썬에서 `multiprocessing`은 코어 4개를 진짜로 채우지만(아래 그림처럼), 각 칸이 **별도 프로세스**라 시작 시 인터프리터 복제 비용과, 칸 사이로 데이터를 옮길 때 pickle 직렬화 비용을 추가로 냅니다. Rust 스레드는 같은 주소 공간을 공유하므로 그 두 비용이 0입니다.

---

## 16.7 `Send`와 `Sync` 맛보기 — 안전성의 뿌리

지금까지 본 모든 안전성("스레드로 못 보내는 것은 컴파일이 막는다")은 두 마커 트레이트에서 나옵니다. 깊은 논의는 Chapter 17로 미루지만, 정체는 짚고 갑니다.

- **`Send`**: 값의 소유권을 다른 스레드로 옮겨도 안전한 타입. `i32`, `String`, `Vec<T>`(T가 Send면) 등 거의 모두 해당. **`Rc<T>`는 `Send`가 아닙니다** — 참조 카운트를 원자적이지 않게 증감하므로 두 스레드가 동시에 건드리면 카운트가 깨집니다(바로 CPython의 refcount 문제와 똑같습니다!).
- **`Sync`**: `&T`를 여러 스레드가 동시에 공유해도 안전한 타입. `T: Sync` ⟺ `&T: Send` 입니다.

이 트레이트들은 컴파일러가 **자동으로 유도**(auto trait)합니다. 구조체의 모든 필드가 `Send`면 그 구조체도 `Send`입니다. 직접 구현할 일은 거의 없고(`unsafe`가 필요), 보통은 "내 타입이 왜 Send가 아니지?"를 추적하게 됩니다.

### `Send` 위반 시연

`Rc`(단일 스레드용 참조 카운트 포인터, Chapter 20)를 스레드로 보내려 해 봅시다.

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let counter = Rc::new(5);
    let c = Rc::clone(&counter);
    thread::spawn(move || {
        println!("{c}"); // 🚨 Rc는 Send가 아니다
    }).join().unwrap();
}
```

```text
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> d_send.rs:7:19
    |
  7 |       thread::spawn(move || {
    |       ------------- ^------
    |       |             |
    |  _____|_____________within this `{closure@d_send.rs:7:19: 7:26}`
    | |     |
    | |     required by a bound introduced by this call
  8 | |         println!("{c}");
  9 | |     }).join().unwrap();
    | |_____^ `Rc<i32>` cannot be sent between threads safely
    |
    = help: within `{closure...}`, the trait `Send` is not implemented for `Rc<i32>`
note: required by a bound in `spawn`
    |
128 |     F: Send + 'static,
    |        ^^^^ required by this bound in `spawn`
```

`Rc<i32> cannot be sent between threads safely` — 닫는 메시지가 곧 진단입니다. 클로저가 `Rc`를 캡처했고, `Rc`가 `Send`가 아니므로 클로저도 `Send`가 아니며, `spawn`은 `F: Send`를 요구하므로 거부됩니다. 해결책은 스레드 안전한 원자적 참조 카운트 `Arc`(Atomic Rc)로 바꾸는 것 — 정확히 Chapter 17의 주제입니다. 파이썬이라면 이런 보호가 없습니다. free-threaded 빌드에서 같은 객체를 여러 스레드가 잡으면 런타임에야 문제가 드러나거나, 더 나쁘게는 조용히 데이터가 꼬입니다.

> **💡 Under the Hood: `Send`/`Sync`는 런타임 비용이 0이다**
>
> `Send`와 `Sync`는 **메서드가 하나도 없는** 마커 트레이트입니다. 컴파일된 바이너리에 이들을 위한 코드나 데이터는 전혀 없습니다 — 순수하게 컴파일 타임 검사용 "라벨"일 뿐입니다. `thread::spawn`의 시그니처에 박힌 `F: Send` 제약은 타입 검사 단계에서만 평가되고, 통과하면 흔적 없이 사라집니다. 즉 "데이터 레이스 방지"라는 거대한 안전성을 **0의 런타임 비용**으로 얻습니다. 이것이 Chapter 1에서 약속한 무비용 추상화(Zero-cost Abstraction)가 동시성에 적용된 모습입니다. 파이썬의 GIL은 매 바이트코드 실행마다 락을 잡고 푸는 영구적 런타임 세금인 반면, Rust의 안전성은 컴파일이 끝나면 청구서가 사라집니다.

---

## 16.8 실전 벤치마크 — 소수 세기로 본 네 세계

이론을 숫자로 검증할 차례입니다. **2 이상 2,000,000 미만의 소수 개수(148,933개)** 를 세는 동일한 CPU 바운드 작업을, 네 가지 방식으로 측정했습니다.

**측정 환경**: Apple Silicon, 물리 코어 10개. Python 3.13.6(GIL 활성, 표준 빌드), Rust 1.96.0 `--release`(opt-level=3). 같은 알고리즘(시행 나눗셈), 워커 10개.

```python
# Python: threading vs multiprocessing (핵심만)
import time, threading
from multiprocessing import Pool

def is_prime(n: int) -> bool:
    if n < 2: return False
    i = 2
    while i * i <= n:
        if n % i == 0: return False
        i += 1
    return True

def count_range(args: tuple[int, int]) -> int:
    start, end = args
    return sum(1 for n in range(start, end) if is_prime(n))
# threading: Thread 10개로 범위 분할 -> GIL로 직렬화됨
# multiprocessing: Pool(10).map(count_range, ranges) -> 진짜 병렬(+pickle 비용)
```

```rust
// Rust: 단일 / std::thread 수동분할 / rayon (핵심만)
use rayon::prelude::*;
use std::thread;

let single = (2..LIMIT).filter(|&n| is_prime(n)).count();         // 단일 스레드

let handles: Vec<_> = (0..10).map(|k| {                            // std::thread x10
    let (s, e) = chunk_bounds(k);
    thread::spawn(move || (s..e).filter(|&n| is_prime(n)).count())
}).collect();
let manual: usize = handles.into_iter().map(|h| h.join().unwrap()).sum();

let par = (2..LIMIT).into_par_iter().filter(|&n| is_prime(n)).count(); // rayon
```

측정 결과(대표값):

| 방식 | 소요 시간 | 단일 스레드 대비 | 비고 |
|---|---|---|---|
| Python 단일 스레드 | ~8,720 ms | 1.0x (기준) | |
| Python `threading` x10 | ~8,790 ms | **0.99x** | GIL로 직렬화 — 오히려 느림 |
| Python `multiprocessing` x10 | ~2,070 ms | 4.2x | 진짜 병렬, 단 fork+pickle 비용 |
| Rust 단일 스레드 | ~120 ms | (Python의 ~73배) | 인터프리터 오버헤드 제거 |
| Rust `std::thread` x10 | ~30 ms | ~4x (단일 Rust 대비) | 부하 불균형으로 10x 못 미침 |
| Rust `rayon` `par_iter` | ~27 ms | ~4.5x (단일 Rust 대비) | work-stealing이 불균형 흡수 |

이 표에서 읽어야 할 다섯 가지:

1. **GIL은 CPU 바운드 스레드를 막는다.** Python `threading`은 스레드를 10개 띄우고도 단일 스레드와 사실상 동일하거나 약간 느립니다(0.99x). 컨텍스트 스위칭 오버헤드만 추가됐습니다. 코어 10개가 있어도 하나만 일했습니다.
2. **`multiprocessing`은 진짜 병렬이지만 공짜가 아니다.** 4.2x로 가속됐지만, 코어 10개로 10x가 아닌 4.2x에 그친 것은 프로세스 생성·범위 분할의 부하 불균형·결과 IPC 비용 때문입니다.
3. **언어 자체의 격차가 압도적이다.** Rust 단일 스레드가 Python 단일 스레드보다 ~73배 빠릅니다. 병렬화를 논하기 전에, 인터프리터·동적 타이핑·객체 박싱 오버헤드를 걷어낸 것만으로 이미 두 자릿수 배수가 납니다.
4. **수동 분할의 함정.** Rust `std::thread`를 10개로 단순 N등분했는데도 단일 Rust 대비 ~4x에 그쳤습니다. 큰 수 구간이 훨씬 무거워(부하 불균형) 가벼운 청크를 맡은 스레드가 일찍 놀았기 때문입니다.
5. **`rayon`이 불균형을 흡수한다.** 같은 작업을 `rayon`은 work-stealing으로 ~4.5x까지 끌어올렸습니다. 코드는 더 짧은데 더 빠릅니다. (이론적 상한 10x에 못 미치는 것은 소수 판정 알고리즘 자체의 메모리·분기 특성과, 측정 머신의 효율/성능 코어 혼합 때문입니다.)

요점: 파이썬에서 "GIL을 우회하려고 무거운 프로세스를 띄우고 데이터를 직렬화하던" 그 모든 노력의 결과(2,070ms)를, Rust는 단일 스레드(120ms)만으로도 압도하며, 병렬화하면 27ms로 내려갑니다.

---

## 16.9 Best Practices & 안티패턴

**이렇게 하라 (Do)**

- **데이터 병렬이면 먼저 `rayon`을 고려하라.** "컬렉션의 모든 원소에 같은 작업"이라면 `iter()`를 `par_iter()`로 바꾸는 것이 거의 항상 정답입니다. 직접 스레드를 띄우기 전에 한 줄로 끝나는지부터 확인하세요.
- **가능하면 메시지 패싱(채널)으로 설계하라.** 공유 가변 상태(락)보다 채널이 추론하기 쉽고 데드락 위험이 적습니다. "통신으로 메모리를 공유하라."
- **부모 데이터를 빌려 읽기만 한다면 `thread::scope`를 써라.** 불필요한 `clone`이나 `move`로 인한 소유권 이전을 피할 수 있습니다.
- **스레드 수는 코어 수에 맞춰라.** CPU 바운드라면 `std::thread::available_parallelism()`(또는 `num_cpus` 크레이트)로 코어 수를 얻어 그만큼만 띄우세요. `rayon`은 이를 자동으로 합니다.
- **`join()`의 `Result`를 무시하지 마라.** 자식 스레드의 패닉이 `Err`로 돌아옵니다. 실무에서는 `unwrap()` 대신 로깅·재시도·정상 종료를 설계하세요.
- **모든 송신자를 명시적으로 닫아라(`drop`).** `for msg in rx` 패턴은 모든 `Sender`가 Drop되어야 끝납니다.

**이건 피하라 (Don't)**

- **요청마다 스레드 하나(thread-per-task) 남발.** 코어 10개에 스레드 10,000개를 띄우면 스케줄러 스래싱으로 오히려 느려집니다. 스레드 풀(`rayon`)이나 비동기(Chapter 18)를 쓰세요.
- **잘게 쪼개 병렬화하기.** 작업 하나가 수 나노초인데 병렬화하면, 스레드 디스패치·동기화 오버헤드가 계산 자체보다 큽니다. `rayon`의 `with_min_len`으로 청크를 키우거나, 그냥 순차로 두세요.
- **I/O 바운드에 무거운 OS 스레드 수천 개.** 수만 개의 동시 연결은 스레드가 아니라 `async`/`tokio`(Chapter 18)의 영역입니다.
- **`clippy`를 끄기.** `cargo clippy`는 `Mutex`를 채널로 대체할 수 있는 경우 등 동시성 안티패턴을 지적합니다. CI에 넣으세요.

---

## 16.10 흔한 함정 (파이썬 개발자가 자주 하는 실수)

### 함정 1: `move`를 잊는다

파이썬은 클로저가 외부 변수를 자유롭게 참조하므로, Rust에서도 그러려다 16.3절의 `error[E0373]`를 만납니다. **스레드에 넘기는 클로저는 거의 항상 `move`** 입니다. 컴파일러가 `use the move keyword`라고 친절히 알려주니, 메시지를 읽고 붙이면 됩니다.

### 함정 2: GIL 직관으로 "스레드 = 동시성 = 안전"이라 믿는다

파이썬에서는 GIL이 (의도치 않게) 많은 레이스를 막아줬기에, "락 없이 전역 변수를 여러 스레드가 건드려도 대충 돌아가던" 경험이 있습니다. Rust에서 같은 짓을 하려면 `Arc<Mutex<T>>`가 필요하고, 안 쓰면 컴파일이 거부합니다(Chapter 17). Rust의 컴파일 에러는 "GIL이 가려주던 버그를 빌드 타임에 드러내는 것"이라 생각하세요.

### 함정 3: 공유 상태에 대한 욕심 — 모든 것을 공유하려 든다

파이썬 습관으로 거대한 전역 딕셔너리를 만들어 모든 스레드가 읽고 쓰게 하려는 충동이 듭니다. Rust에서는 그 순간 `Arc<Mutex<HashMap>>`의 락 경합(contention)이 병렬성을 갉아먹습니다. **먼저 물어보세요: 정말 공유해야 하나, 아니면 각자 계산하고 채널로 결과만 모으면 되나?** 대개 후자가 더 빠르고 단순합니다. 진짜 공유가 필요할 때의 도구가 Chapter 17입니다.

### 함정 4: `multiprocessing` 멘탈로 데이터를 복사한다

`multiprocessing`에서는 데이터를 워커에 보내려면 복사(pickle)가 필수였습니다. 그 습관으로 Rust에서도 스레드마다 `data.clone()`을 남발하면, 진짜 병렬성의 핵심 장점(공유 메모리)을 스스로 버리는 셈입니다. 읽기 공유는 `thread::scope`의 빌림이나 `Arc`(Ch17)로, 소유권 이전은 채널로 — 무지성 `clone`은 마지막 수단입니다.

### 함정 5: 채널을 안 닫고 데드락

16.4절에서 봤듯, `for msg in rx`는 모든 `Sender`가 사라져야 끝납니다. 원본 `tx`를 `drop`하지 않거나, 어딘가에 살아남은 `tx.clone()`이 있으면 수신 루프가 영원히 멈춰 있습니다. "왜 프로그램이 안 끝나지?"의 단골 원인입니다.

---

Rust의 동시성은 파이썬과 근본적으로 다른 약속 위에 서 있습니다. 파이썬은 "한 번에 하나만 실행하게 해서(GIL) refcount를 지키고, 진짜 병렬이 필요하면 프로세스를 복제하라"고 말합니다. Rust는 "모든 코어를 진짜로 동시에 쓰되, 스레드 간에 오가는 데이터의 안전성을 `Send`/`Sync`로 컴파일 타임에 증명하라"고 말합니다. 그 결과가 두려움 없는 동시성 — 데이터 레이스가 런타임의 운이 아니라 빌드의 통과 여부가 되는 세계입니다.

이번 챕터에서 우리는 `move`로 소유권을 넘기고, 채널로 통신하고, `scope`로 안전하게 빌리고, `rayon`으로 데이터를 병렬로 흘려보냈습니다. 하지만 한 가지를 의도적으로 미뤘습니다 — 여러 스레드가 **동시에 하나의 값을 읽고 쓰는** 진짜 공유 상태입니다. 글로벌 캐시, 조회수 카운터, 누적 집계기. 16.10절 함정 3에서 경고했듯, 이것은 위험천만하지만 때로 불가피합니다. 그리고 `Rc`를 스레드로 보내려다 만난 `error[E0277]: Rc cannot be sent`가 가리킨 해답이 바로 그 다음 장에 있습니다.

**Chapter 17: 상태 공유와 동기화**에서는 원자적 참조 카운트 `Arc<T>`와, 데이터를 락으로 감싸 "락 없이는 접근조차 못 하게" 강제하는 `Mutex<T>`·`RwLock<T>`를 파헤칩니다. 파이썬의 `threading.Lock`이 무시할 수 있는 신호등이었다면, Rust의 `Mutex<T>`는 데이터를 물리적으로 가둔 금고입니다. 그리고 그 금고를 잘못 다뤘을 때 발생하는 데드락마저, Rust의 타입 시스템과 RAII가 어떻게 도와주는지 보게 될 것입니다.
