# Chapter 4. 소유권 (Ownership): GC 없이 살아남기

![Part 2. 핵심 패러다임 전환](../images/part-02-ownership.jpg)


파이썬 개발자로 살아오면서 메모리 해제 시점을 명확하게 통제해 본 적이 있나요? 메모리릭이 터져서 `gc.collect()`를 수동으로 호출해 본 적은 있겠지만, 변수를 선언할 때 "이 변수의 바이트 배열은 정확히 코드 몇 번째 줄에서 운영체제(OS)로 반환될까?"를 예측하며 코딩하는 파이썬 개발자는 없습니다.

10년 차 백엔드 엔지니어라면 이런 새벽을 한 번쯤 겪었을 겁니다. 트래픽이 몰리는 시간대에 P99 레이턴시(latency) 그래프가 톱니처럼 주기적으로 치솟습니다. 코드에는 문제가 없습니다. APM 트레이스를 까보면 특정 요청만 평소보다 80ms씩 느립니다. 범인은 가비지 컬렉터(Garbage Collector, GC)입니다. 누적된 객체를 청소하느라 인터프리터가 잠시 세상을 멈춘 것입니다. 여러분의 코드는 죄가 없는데, 런타임이 여러분을 배신합니다.

파이썬은 **참조 횟수 계산(Reference Counting)**과 순환 참조(Circular Reference)를 감지하는 **세대별 가비지 컬렉터(Generational Garbage Collector)**를 결합하여 백그라운드에서 주기적으로 메모리를 청소합니다. 편리하지만, 이 청소 프로세스가 언제 깨어나 CPU를 점유하고 애플리케이션의 응답을 멈출지(GC Pause) 예측하기 어렵습니다. 그리고 이 모든 편의는 공짜가 아닙니다. 모든 객체가 참조 카운트를 위한 추가 바이트를 짊어지고, 모든 대입과 함수 호출이 카운터를 증감시키는 보이지 않는 명령어를 실행합니다.

Rust는 이 런타임 GC를 완전히 없애버렸습니다. 그렇다고 C나 C++처럼 개발자가 실수하기 쉬운 `malloc`, `free`를 직접 타이핑하게 두지도 않습니다. 대신 **소유권(Ownership)**이라는 3가지 핵심 규칙을 컴파일러가 강제하여, 스코프(Scope)를 벗어나는 순간 메모리가 *정확히 0초의 런타임 추적 오버헤드 없이* 해제되도록 설계했습니다. 메모리 해제 시점이 소스 코드를 읽는 것만으로 결정되는 것, 이것이 Rust가 자랑하는 "예측 가능성(determinism)"과 "성능"의 근간입니다.

이번 장에서는 파이썬 개발자가 Rust에서 마주하는 가장 거대한 첫 번째 벽인 소유권을 다룹니다. 변수 대입 한 줄에서 힙(Heap) 메모리 안에 무슨 일이 벌어지는지를 바이트 단위로 해부하고, 컴파일러가 어떻게 use-after-free와 double-free를 *컴파일 타임에* 차단하는지 그 수학적 원리를 추적합니다.

---

## 4.1 The Python Way — 참조 카운팅의 청구서

먼저 우리가 지금까지 어떻게 살아왔는지 정직하게 들여다봅시다. 파이썬의 메모리 관리는 두 개의 엔진이 동시에 돌아갑니다.

**엔진 1: 참조 카운팅(Reference Counting).** 모든 파이썬 객체는 헤더에 `ob_refcnt`라는 정수 필드를 가집니다. 누군가 이 객체를 가리키면 +1, 가리키던 변수가 사라지면 -1. 카운트가 0이 되는 순간 즉시 해제됩니다.

```python
import sys

data = [1, 2, 3]            # 리스트 객체 refcnt = 1
print(sys.getrefcount(data))  # 2 출력 (getrefcount 인자로 넘기며 일시적 +1)

alias = data               # refcnt = 2
del alias                  # refcnt = 1
del data                   # refcnt = 0 → 즉시 해제
```

이 방식은 결정적(deterministic)이라는 장점이 있습니다. 마지막 참조가 사라지면 *그 자리에서* 메모리가 회수됩니다. 그런데 두 가지 비용이 따라붙습니다.

첫째, **모든 객체가 카운터 비용을 짊어집니다.** CPython에서 가장 작은 객체조차 무겁습니다. `sys.getsizeof(1)`은 28바이트입니다. 정수 `1` 하나가 28바이트입니다. 이 안에는 타입 포인터(`ob_type`), 참조 카운트(`ob_refcnt`), 그리고 실제 값이 들어 있습니다. Rust의 `i64`가 정확히 8바이트인 것과 비교하면 3.5배입니다.

> 이 책 전체에서 파이썬 정수 크기는 다음 측정값(Python 3.13.6, `sys.getsizeof`)으로 통일합니다 — 머신 워드 하나에 들어가는 작은 정수는 **28바이트**(`getsizeof(1) == 28`, `getsizeof(256) == 28`), 워드 하나를 넘기는 정수(`2**40`)는 **32바이트**, 더 큰 정수(`10**30`)는 **40바이트**입니다. CPython은 정수를 30비트 "digit" 단위로 저장하므로 값이 커질수록 8바이트씩 계단식으로 늘어납니다. 따라서 "정수 하나 = 28바이트"는 작은 값에만 정확하고, 일반적인 데이터에서는 32바이트 이상으로 보는 것이 맞습니다. 이후 모든 메모리 추정은 이 기준을 따릅니다.

둘째, **모든 대입이 카운터를 건드립니다.** `alias = data` 같은 단순 대입조차 내부적으로는 `Py_INCREF` 매크로를 실행합니다. 멀티스레드 환경이라면 이 증감이 원자적(atomic)이어야 하므로 더 비쌉니다. 바로 이것이 GIL(Global Interpreter Lock)이 존재하는 이유 중 하나입니다. 참조 카운트를 락 없이 안전하게 만들려면 인터프리터 전체를 직렬화하는 것이 가장 간단했던 것입니다.

**엔진 2: 세대별 GC(Generational Garbage Collector).** 참조 카운팅만으로는 못 잡는 것이 있습니다. 바로 순환 참조입니다.

```python
class Node:
    def __init__(self) -> None:
        self.partner: "Node | None" = None

a = Node()
b = Node()
a.partner = b   # b의 refcnt = 2
b.partner = a   # a의 refcnt = 2

del a           # a 객체의 refcnt: 2 → 1 (b.partner가 아직 가리킴)
del b           # b 객체의 refcnt: 2 → 1 (a.partner가 아직 가리킴)
# 두 객체 모두 외부에서 접근 불가능한데도 refcnt가 0이 아니라 메모리에 남음 → 누수!
```

`a`와 `b`는 이제 프로그램 어디서도 접근할 수 없습니다. 하지만 서로를 가리키고 있어 참조 카운트가 절대 0으로 떨어지지 않습니다. 이것이 메모리 누수입니다. 이걸 잡기 위해 파이썬은 별도의 추적형 GC를 주기적으로 돌립니다. 객체들을 0, 1, 2 세대로 나눠 관리하고, 오래 살아남은 객체일수록 덜 자주 검사합니다(generational hypothesis). 그런데 이 GC가 깨어나 객체 그래프를 순회하는 동안에는 다른 작업이 멈춥니다(stop-the-world).

```python
import gc

# 실무에서 흔히 보이는 처방들 — 모두 "GC를 통제 못 한다"는 증거입니다
gc.disable()          # 레이턴시 민감 구간에서 GC 멈춤을 막으려 아예 끔
gc.collect()          # 수동으로 청소 강제 (언제 돌릴지 우리가 모르므로)
gc.set_threshold(...) # 세대별 임계치 튜닝
```

레이턴시에 민감한 서비스에서 `gc.disable()`을 호출해 본 경험이 있다면, 여러분은 이미 GC가 통제 불가능한 변수임을 몸으로 알고 있는 것입니다.

> **Under the Hood: CPython 객체 헤더의 실제 무게**
>
> 64비트 CPython에서 모든 객체는 `PyObject` 헤더로 시작합니다.
>
> ```text
> ┌──────────────────────┬──────────────────────┐
> │ ob_refcnt (8 bytes)  │ ob_type   (8 bytes)  │   ← 모든 객체 공통 헤더 = 16 bytes
> ├──────────────────────┴──────────────────────┤
> │ ... 실제 데이터 (int 값, 리스트 포인터 등) ...   │
> └──────────────────────────────────────────────┘
> ```
>
> 정수 객체 하나가 28바이트, 빈 리스트가 56바이트, 작은 딕셔너리가 수백 바이트입니다. 100만 개의 작은 정수를 리스트에 담으면 데이터 자체보다 헤더와 포인터가 차지하는 메모리가 더 큽니다. Rust의 `Vec<i64>`가 8바이트 × 100만 = 8MB의 연속된 버퍼인 반면, 파이썬 `list`는 각 정수를 가리키는 포인터 배열(8MB) *더하기* 흩어진 정수 객체들(28MB)입니다. 메모리 사용량이 4~5배, 그리고 캐시 지역성(cache locality) 차이로 인한 순회 속도는 그 이상으로 벌어집니다.

---

## 4.2 The Rust Mental Model — 컴파일러가 청소부다

Rust의 사고방식은 단순하지만 급진적입니다. **"메모리를 누가, 언제 해제할지를 런타임이 아니라 컴파일 타임에 결정하자."**

런타임에 참조 카운트를 추적하지도, GC가 그래프를 순회하지도 않습니다. 대신 컴파일러가 소스 코드를 정적으로 분석해서, 각 값이 정확히 어느 코드 줄에서 마지막으로 쓰이고 어느 중괄호 `}`에서 해제되어야 하는지를 *컴파일 시점에* 못 박아 버립니다. 그리고 그 해제 코드(`free` 호출)를 컴파일러가 자동으로 삽입합니다.

이것이 왜 "무비용 추상화(Zero-cost Abstraction)"인지 이해하는 것이 핵심입니다. 컴파일된 바이너리에는 가비지 컬렉터도, 참조 카운터도 없습니다. 그냥 적절한 위치에 `free`가 박혀 있는 C 코드와 동일합니다. 다만 그 `free`를 사람이 손으로 쓰다가 실수하는 대신, 컴파일러가 규칙에 따라 기계적으로 검증하고 삽입한다는 점만 다릅니다.

Rust의 강력하고 무자비한 컴파일러(이 검증 로직을 **빌림 검사기(Borrow Checker)**라 부릅니다)는 코드를 컴파일할 때 다음 세 가지 규칙을 수학적으로 검증합니다.

1. **Rust의 모든 값은 그 값의 '소유자(Owner)'라 불리는 변수를 가진다.**
2. **특정 시점에 값의 소유자는 단 하나뿐이다.**
3. **소유자가 스코프(Scope)를 벗어나면, 값은 버려진다(Dropped).**

이 세 줄이 전부입니다. 하지만 이 세 줄에서 파이썬 개발자가 평생 의존해 온 GC, 그리고 C 개발자가 평생 두려워해 온 메모리 버그가 동시에 사라집니다.

### 스택(Stack)과 힙(Heap)의 차이

이 규칙들이 왜 중요한지 이해하려면 데이터가 메모리 어디에 저장되는지 구분해야 합니다. 3장에서 본 메모리 레이아웃 논의의 연장선입니다.

* `i32`, `bool`, `f64` 등 컴파일 타임에 크기가 고정되어 있는 단순한 데이터는 빠르고 복사 오버헤드가 적은 **스택(Stack)** 메모리에 저장됩니다. 이 타입들은 `Copy` 트레이트를 구현하고 있어, 할당이나 파라미터 전달 시 자동으로 값이 비트 단위 복사됩니다. 파이썬과 거의 동일한 감각으로 쓸 수 있습니다.
* `String`, `Vec<T>` 등 런타임에 크기가 동적으로 변할 수 있는 데이터는 OS에 메모리를 요청하여 **힙(Heap)** 메모리에 할당받습니다. 소유권 규칙의 진가는 바로 이 힙 데이터를 다룰 때 발휘됩니다.

이 구분이 중요한 이유는, 스택 데이터는 복사가 싸지만 힙 데이터는 복사가 비싸기 때문입니다. Rust는 비싼 복사를 절대 묵시적으로(implicitly) 일으키지 않습니다. 비싼 일이 일어날 때는 반드시 코드에 흔적이 남습니다. 이 철학을 머릿속에 새겨 두면 앞으로 나올 모든 규칙이 자연스럽게 따라옵니다.

| 파이썬 개념 | Rust 개념 | 핵심 차이 |
|---|---|---|
| 참조 카운팅 (`ob_refcnt`) | 소유권 (Ownership) | 런타임 카운터 vs 컴파일 타임 추적 (런타임 비용 0) |
| 세대별 GC (stop-the-world) | `Drop` 트레이트 / RAII | 예측 불가능한 일시정지 vs 결정적·즉시 해제 |
| `obj2 = obj1` (참조 복사, refcnt+1) | `let b = a;` (Move, a 무효화) | 공유 vs 소유권 이전 |
| `copy.deepcopy(obj)` | `.clone()` (명시적) | 묵시적/명시적 모두 가능 vs 항상 명시적 |
| `del obj` (refcnt -1, 0이면 해제) | `drop(obj)` (즉시 해제) | 카운트 의존 vs 강제 즉시 해제 |
| `with open() as f:` (`__enter__`/`__exit__`) | 스코프 종료 시 자동 `Drop` | 명시적 컨텍스트 매니저 vs 모든 자원 자동 |
| `gc.collect()` 수동 호출 | (해당 없음 — 컴파일러가 위치 결정) | 런타임 개입 vs 컴파일 타임 확정 |
| 순환 참조 → GC가 청소 | 순환 참조 → 컴파일러가 애초에 차단(또는 `Weak`) | 사후 청소 vs 사전 방지 |

---

## 4.3 Move Semantics — 얕은 복사도 깊은 복사도 아니다

파이썬에서 가변 객체인 리스트를 다른 변수에 대입하면 어떤 일이 일어날까요?

**The Python Way (Reference Assignment)**

```python
list1 = [1, 2, 3]  # Heap에 리스트 객체가 생성되고, list1은 포인터를 가짐
list2 = list1      # list2도 같은 Heap 주소를 가리킴 (Reference Count 1 → 2로 증가)

list2.append(4)
print(list1)  # [1, 2, 3, 4] 출력 (list1과 list2는 운명 공동체)
```

파이썬은 단순히 메모리 주소(포인터)만 복사합니다. 이를 얕은 복사, 또는 객체 참조(Object Reference)라고 부릅니다. 두 변수가 같은 데이터를 수정할 수 있으므로 예측하지 못한 버그의 온상이 되곤 합니다. 함수에 리스트를 넘긴 뒤 그 함수가 몰래 `.append()`를 호출해서 호출자의 데이터가 바뀌어 버린 디버깅 경험, 한 번쯤 있을 겁니다.

그렇다면 Rust에서 같은 코드를 작성해 봅시다. 동적 배열인 `Vec<i32>`를 사용합니다.

**The Rust Implementation (Move)**

```rust
fn main() {
    // vec1은 Heap에 할당된 데이터 버퍼 [1, 2, 3]의 유일하고 절대적인 "소유자"가 됩니다.
    let vec1 = vec![1, 2, 3];

    // vec2에 vec1을 대입합니다. 파이썬처럼 참조 카운트가 올라갈까요?
    let vec2 = vec1;

    // println!("{:?}", vec1);
    // 🚨 컴파일 에러 발생! "borrow of moved value: `vec1`"
    // (이동된 값 vec1을 사용할 수 없습니다)

    println!("{:?}", vec2); // vec2만 유효
}
```

이것이 파이썬 개발자가 Rust에서 마주하는 첫 번째 거대한 벽입니다. `let vec2 = vec1;`은 파이썬에서라면 그냥 별칭(alias)을 하나 더 만드는 무해한 코드입니다. 그런데 Rust에서는 `vec1`을 *죽여 버립니다.*

주석 처리된 `println!`의 주석을 풀고 컴파일하면 컴파일러는 이렇게 말합니다.

```text
error[E0382]: borrow of moved value: `vec1`
 --> src/main.rs:9:22
  |
3 |     let vec1 = vec![1, 2, 3];
  |         ---- move occurs because `vec1` has type `Vec<i32>`,
  |              which does not implement the `Copy` trait
4 |     let vec2 = vec1;
  |                ---- value moved here
...
9 |     println!("{:?}", vec1);
  |                      ^^^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
4 |     let vec2 = vec1.clone();
  |                    ++++++++
```

이 에러 메시지는 Rust 컴파일러의 진가를 보여줍니다. 단순히 "에러"라고 외치는 것이 아니라, (1) `vec1`이 `Copy`를 구현하지 않은 타입이라서 (2) 4번 줄에서 값이 이동했고 (3) 9번 줄에서 이동 후에 빌리려 해서 문제라는 인과관계를 짚어 줍니다. 심지어 성능 비용을 감수할 수 있다면 `.clone()`을 붙이라는 구체적 해법까지 제시합니다. 파이썬의 `NameError`나 `AttributeError`가 런타임에 터지는 것과 달리, 이 모든 진단은 프로그램이 단 한 번도 실행되기 전에 끝납니다.

에러 메시지는 **"값이 vec2로 이동(Move)했기 때문에, 더 이상 구 소유자인 vec1을 사용할 수 없다"**고 단호하게 말합니다.

> **Under the Hood: Move 시 메모리에서 실제로 일어나는 일**
>
> `String`을 예로 들어 봅시다. `String`은 스택에 정확히 3개의 워드(word), 64비트 시스템에서 24바이트를 차지합니다(`std::mem::size_of::<String>()`을 찍어 보면 정확히 `24`가 나옵니다). 이 3워드는 다음과 같습니다.
>
> ```text
> 변수 s = String::from("hello")
>
>   스택 (Stack)                          힙 (Heap)
> ┌──────────────────────┐              ┌───┬───┬───┬───┬───┐
> │ ptr      ─────────────┼─────────────▶│ h │ e │ l │ l │ o │
> ├──────────────────────┤              └───┴───┴───┴───┴───┘
> │ len      = 5          │               0x1018ee160 (예시 주소)
> ├──────────────────────┤
> │ capacity = 5          │
> └──────────────────────┘
>   24 bytes (3 × 8)
> ```
>
> 이제 `let s2 = s;`를 실행하면 어떤 일이 벌어질까요?
>
> ```text
>   s (무효화됨)                          힙 (Heap) — 그대로!
> ┌──────────────────────┐              ┌───┬───┬───┬───┬───┐
> │ ✗ 접근 불가 (moved)    │       ┌─────▶│ h │ e │ l │ l │ o │
> └──────────────────────┘       │      └───┴───┴───┴───┴───┘
>                                 │       0x1018ee160 (동일 주소!)
>   s2 (새 소유자)                 │
> ┌──────────────────────┐       │
> │ ptr      ─────────────┼───────┘
> ├──────────────────────┤
> │ len      = 5          │
> ├──────────────────────┤
> │ capacity = 5          │
> └──────────────────────┘
> ```
>
> **스택의 24바이트(ptr/len/capacity)만 비트 단위로 복사됩니다.** 힙에 있는 `"hello"` 버퍼는 단 1바이트도 건드리지 않습니다. `s.as_ptr()`과 `s2.as_ptr()`을 찍어 보면 둘 다 똑같은 힙 주소(`0x1018ee160`)를 가리킵니다. 즉, Move는 물리적으로는 *얕은 복사(24바이트 memcpy)*입니다.
>
> 그런데 파이썬의 얕은 복사와 결정적 차이가 있습니다. 파이썬은 복사 후 두 변수 모두 살아 있어서 둘 다 힙을 가리킵니다(공유). Rust는 복사 후 **원본 `s`를 컴파일러가 무효화(invalidate)**합니다. 이제 `s`는 그저 스택에 남은 죽은 비트일 뿐, 컴파일러가 접근을 거부합니다. 결과적으로 힙 버퍼의 소유자는 여전히 정확히 한 명(`s2`)입니다. 규칙 2번("소유자는 단 하나")이 지켜집니다.

**왜 이렇게 가혹하게 설계했을까?**

만약 파이썬처럼 `vec1`과 `vec2`가 같은 Heap 메모리를 가리키게 둔다면, `main` 함수 블록이 끝날 때 (소유권 규칙 3번에 의해) `vec2`가 먼저 Heap 메모리를 해제하고, 이어서 `vec1`도 동일한 메모리를 다시 해제하려 시도할 것입니다. 이를 **이중 해제(Double Free) 취약점**이라 부르며, 수많은 C/C++ 프로그램에서 메모리 오염과 보안 사고를 일으킨 주범입니다.

파이썬은 이 문제를 참조 카운팅으로 회피합니다. 두 변수가 같은 객체를 가리키면 카운트가 2이므로, 한쪽이 `del`되어도 카운트가 1로 줄 뿐 해제되지 않습니다. 둘 다 사라져 카운트가 0이 되어야 비로소 한 번 해제됩니다. 안전하지만, 그 대가로 모든 객체에 카운터를 달고 모든 대입에서 카운터를 증감하는 런타임 비용을 영원히 지불합니다.

Rust는 아예 **"소유자는 단 하나뿐이다"**라는 2번 규칙을 강제하여, `vec1`에서 `vec2`로 포인터 정보를 복사하는 순간 **`vec1` 변수 자체의 생명(유효성)을 강제로 끊어버립니다(Invalidation)**. 소유자가 항상 정확히 하나이므로 해제도 정확히 한 번 일어납니다. 카운터가 필요 없습니다. 컴파일 타임에 `vec1`의 접근을 원천 차단하므로 런타임 이중 해제 버그가 수학적으로 발생할 수 없습니다. 이것을 얕은 복사가 아닌 **이동(Move)**이라고 부릅니다.

---

## 4.4 Copy vs Move vs Clone — 세 가지 복제 비용의 스펙트럼

여기서 한 가지 의문이 생깁니다. 4.3절에서 `Vec`과 `String`은 대입하면 Move가 일어나 원본이 죽었습니다. 그런데 정수는 어떨까요?

```rust
fn main() {
    let x = 5;
    let y = x;
    println!("x={}, y={}", x, y); // x=5, y=5 — 둘 다 멀쩡!
}
```

`i32`는 Move 후에도 `x`가 살아 있습니다. 왜 차이가 날까요? 답은 **`Copy` 트레이트**입니다.

`i32`, `f64`, `bool`, `char`, 그리고 이들로만 구성된 튜플 같은 타입은 `Copy` 트레이트를 구현합니다. 이들의 공통점은 **전체 데이터가 스택에 있고 크기가 작으며 힙 자원을 소유하지 않는다**는 것입니다. 이런 값은 비트 단위로 복사해도 두 복사본이 각자 독립적이고, 둘 다 살아 있어도 이중 해제 같은 문제가 없습니다(애초에 해제할 힙이 없으니까요). 그래서 Rust는 이들에 대해 "원본 무효화" 대신 "그냥 복사하고 둘 다 살림"을 선택합니다. 파이썬에서 `int`를 다루는 것과 똑같은 감각입니다.

반대로 `String`, `Vec`처럼 힙 버퍼를 소유한 타입은 `Copy`를 구현할 수 없습니다(컴파일러가 막습니다). 비트 복사를 허용하면 두 소유자가 같은 힙을 가리키게 되어 이중 해제가 발생하기 때문입니다. 그래서 이들은 Move됩니다.

그렇다면 파이썬의 `copy.deepcopy()`처럼 힙 버퍼 자체를 독립적으로 복제하고 싶다면? 명시적으로 **`.clone()`**을 호출해야 합니다.

```rust
fn main() {
    let vec1 = vec![1, 2, 3];
    let vec2 = vec1.clone(); // Heap 데이터를 OS로부터 재할당받아 통째로 복사 (비용이 큼)

    println!("vec1: {:?}", vec1); // clone을 했으므로 vec1의 소유권은 이동하지 않고 유지됨
    println!("vec2: {:?}", vec2); // 각각 독립적인 Heap 주소를 가짐
}
```

`.clone()`은 OS에 새 힙 버퍼를 요청(`malloc`)하고, 원본 버퍼의 내용을 새 버퍼로 `memcpy`합니다. 이제 두 `Vec`은 각자 독립적인 힙을 소유하므로 둘 다 살아남고, 각자 자기 힙을 한 번씩 해제합니다.

세 가지를 비용 관점에서 정리하면 다음과 같습니다.

| 동작 | 무엇을 복사하나 | 비용 | 원본 상태 | 파이썬 대응 |
|---|---|---|---|---|
| **Copy** | 스택 비트 (작은 고정 크기) | 거의 0 (레지스터 이동 수준) | 살아 있음 | `int`, `float` 대입 |
| **Move** | 스택 헤더만 (예: 24바이트) | 거의 0 (memcpy 24B) | **무효화됨** | (직접 대응 없음) |
| **Clone** | 힙 버퍼 전체 (`malloc` + `memcpy`) | **O(n)** + 할당 | 살아 있음 | `copy.deepcopy()` |

핵심은 **Move와 Copy는 둘 다 거의 공짜**라는 점입니다. 차이는 비용이 아니라 "원본이 살아남느냐"입니다. 비싼 것은 오직 `Clone`뿐이고, Rust는 비싼 일에 반드시 `.clone()`이라는 눈에 띄는 꼬리표를 강제합니다.

**현실적인 수치 감각.** `Vec<u8>` 1MB를 `.clone()`하는 비용은 대략 새 1MB 할당 + 1MB memcpy입니다. 최신 x86 머신에서 memcpy 대역폭이 대략 10~20GB/s이므로 1MB 복사는 약 50~100마이크로초입니다. 별것 아닌 것 같지만, 초당 10만 요청을 처리하는 핫 패스(hot path)에서 매 요청마다 1MB를 무심코 `.clone()`한다면 초당 100GB의 메모리 대역폭을 복사에만 쏟아붓는 셈입니다. 반면 Move나 참조(다음 장)로 넘기면 24바이트 헤더만 옮기면 됩니다. 4만 배 차이입니다. Rust에서 `.clone()`이 코드에 명시적으로 보이는 것은 바로 이런 병목을 코드 리뷰에서 즉시 발견하기 위한 장치입니다.

파이썬은 대입 연산(`=`)이 상황에 따라 얕은 복사인지 깊은 복사인지 헷갈릴 때가 많습니다(리스트는 참조, `int`는 값처럼 보이지만 사실 불변 객체 공유...). 하지만 Rust는 OS 레벨의 값비싼 Heap 복사가 일어날 때는 코드에 무조건 `.clone()`이라는 꼬리표가 눈에 확 띄게 명시되도록 언어가 설계되어 있습니다. 코드를 읽기만 해도 성능 병목 지점을 찾을 수 있습니다.

---

## 4.5 Drop 트레이트와 RAII — 결정적 해제의 가치

규칙 3번("소유자가 스코프를 벗어나면 값은 버려진다")이 실제로 어떻게 동작하는지 봅시다. 값이 "버려진다(dropped)"는 것은 `Drop` 트레이트의 `drop` 메서드가 호출된다는 뜻입니다.

파이썬에서 자원 정리는 두 가지 방법이 있었습니다. 명시적 컨텍스트 매니저(`with`)와 `__del__` 소멸자입니다.

```python
class Connection:
    def __init__(self, host: str) -> None:
        self.host = host
        print(f"[연결] {host}")

    def __enter__(self) -> "Connection":
        return self

    def __exit__(self, *args) -> None:
        print(f"[해제] {self.host}")

    def __del__(self) -> None:
        # GC가 언제 호출할지 보장되지 않음! 위험하다.
        print(f"[__del__] {self.host}")

# 컨텍스트 매니저: 명시적이지만 매번 with를 써야 한다
with Connection("db-1") as conn:
    pass  # 블록을 벗어나면 __exit__ 호출

# __del__에만 의존하면? GC 시점이 불확실 → 자원이 언제 풀릴지 모름
c = Connection("db-2")
del c  # refcnt가 0이면 __del__ 호출되지만, 순환 참조면 영원히 안 불릴 수도
```

`with` 블록은 안전하지만, 자원을 다룰 때마다 명시적으로 `with`를 써야 하고 깜빡하면 누수입니다. `__del__`은 GC 시점에 의존하므로 *언제* 호출될지 보장되지 않습니다. 순환 참조에 걸리면 영영 호출되지 않을 수도 있습니다. 그래서 파이썬 베스트 프랙티스는 "`__del__`에 의존하지 말라"입니다.

Rust에서는 모든 값이 스코프를 벗어나는 순간 `Drop`이 자동으로, 결정적으로 호출됩니다. 별도의 `with`도, GC 시점에 대한 불안도 없습니다.

```rust
struct Connection {
    host: String,
}

impl Connection {
    fn new(host: &str) -> Self {
        println!("[연결] {}", host);
        Connection { host: host.to_string() }
    }
}

// Drop 트레이트 구현: 스코프 종료 시 컴파일러가 자동 호출
impl Drop for Connection {
    fn drop(&mut self) {
        println!("[해제] {}", self.host);
    }
}

fn main() {
    let _conn = Connection::new("db-1");
    println!("작업 수행 중...");
    // main 끝 — _conn.drop()이 자동 호출됨. with도, del도 필요 없음.
}
```

출력은 다음과 같습니다.

```text
[연결] db-1
작업 수행 중...
[해제] db-1
```

여기서 C++ 진영의 용어를 빌리면, 이 기법을 **RAII (Resource Acquisition Is Initialization)**라고 부릅니다. 자원의 획득을 객체 초기화에 묶고, 객체 소멸 시 자원을 자동 해제하는 패턴입니다. 파이썬에서는 파일이나 소켓을 닫기 위해 `with open(...) as f:`를 명시적으로 작성하거나 GC의 `__del__` 호출에 의존해야 하지만, Rust는 메모리, 파일 핸들, DB 커넥션, 스레드 락(Lock), 뮤텍스 가드 등 모든 자원에 이 RAII(`Drop` 트레이트)가 숨 쉬듯 기본 탑재되어 있어 자원 누수(Resource Leak)가 원천 차단됩니다. 17장에서 다룰 `Mutex`의 락 해제도 정확히 이 메커니즘으로 동작합니다. 락 가드가 스코프를 벗어나면 자동으로 언락됩니다.

### drop 순서는 역순(LIFO)이다

여러 변수가 같은 스코프에 있을 때, 해제 순서는 선언의 **역순**입니다. 스택처럼 마지막에 만든 것이 먼저 해제됩니다(LIFO).

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Dropping: {}", self.name);
    }
}

fn main() {
    let _first = Resource { name: String::from("first") };
    let _second = Resource { name: String::from("second") };
    let _third = Resource { name: String::from("third") };
    println!("--- main end ---");
}
```

실행 결과:

```text
--- main end ---
Dropping: third
Dropping: second
Dropping: first
```

> **Under the Hood: 왜 역순(LIFO)인가**
>
> drop 순서가 역순인 것은 우연이 아니라 의존성 안전을 위한 설계입니다. 보통 나중에 선언된 변수가 먼저 선언된 변수에 의존합니다(예: `parser`가 `buffer`를 참조). 만약 선언 순서대로 `buffer`를 먼저 해제하면, 아직 살아 있는 `parser`가 이미 죽은 `buffer`를 가리키는 dangling 상태가 됩니다. 역순으로 해제하면 의존하는 쪽(`parser`)이 먼저 사라지므로 이런 위험이 없습니다.
>
> 이것은 함수 호출 스택이 풀리는(unwind) 방식과 정확히 일치합니다. 나중에 push된 스택 프레임이 먼저 pop되듯, 나중에 선언된 변수가 먼저 drop됩니다. 구조체 안에서는 필드가 선언 순서대로 drop되고, 컬렉션(`Vec`)은 인덱스 순서대로 원소가 drop됩니다. 이 모든 순서가 컴파일 타임에 결정되어 바이너리에 박힙니다 — 런타임 추적이 전혀 없습니다.

### `std::mem::drop`으로 조기 해제하기

스코프 끝까지 기다리지 않고 *지금 당장* 해제하고 싶다면 `std::mem::drop` 함수(프렐류드에 있어 그냥 `drop`)를 호출합니다. 이 함수의 구현은 놀랍도록 단순합니다 — 단지 값의 소유권을 받아서 아무것도 안 하고 함수가 끝나 버립니다. 그러면 규칙 3번에 의해 함수 끝에서 즉시 drop됩니다.

```rust
fn main() {
    let big = vec![0u8; 10 * 1024 * 1024]; // 10MB 버퍼
    // ... big을 써서 무거운 작업 수행 ...

    drop(big); // 🔻 여기서 즉시 10MB 해제. 스코프 끝까지 안 기다림.
    // println!("{:?}", big); // 🚨 컴파일 에러: drop()이 소유권을 가져갔으므로 무효

    // 이후의 메모리 집약적 작업은 10MB가 이미 반환된 상태에서 실행됨
    heavy_work();
}

fn heavy_work() { /* ... */ }
```

이것이 결정적 해제의 실용적 가치입니다. 파이썬에서 큰 객체를 빨리 비우려고 `del big; gc.collect()`를 호출해 본 적이 있다면, 그것이 얼마나 미덥지 못한 호출이었는지 떠올려 보세요. `gc.collect()`는 "지금 청소해 줘"라는 *요청*일 뿐 보장이 아닙니다. Rust의 `drop(big)`은 컴파일러가 그 위치에 정확히 `free`를 박는 *명령*입니다.

---

## 4.6 함수와 소유권 — 그리고 되돌려주기의 불편함

변수를 다른 변수에 대입할 때뿐만 아니라, **함수에 인자로 넘길 때도 소유권은 이동(Move)**합니다. 함수 파라미터는 본질적으로 또 하나의 대입이기 때문입니다.

```rust
fn take_ownership(v: Vec<i32>) {
    println!("내가 받았다: {:?}", v);
} // 함수 스코프 종료. 파라미터 v는 소유권을 다 썼으므로 여기서 내부 Heap 메모리가 Drop(해제)됨.

fn main() {
    let my_vec = vec![1, 2, 3];

    // my_vec의 소유권이 take_ownership 함수의 파라미터 v로 '이동'합니다.
    take_ownership(my_vec);

    // println!("{:?}", my_vec);
    // 🚨 컴파일 에러! 함수에게 소유권을 넘겨주었으므로 여기서 my_vec은 빈 껍데기입니다.
}
```

`take_ownership`을 호출하는 순간 `my_vec`의 소유권은 함수 안으로 빨려 들어가고, 함수가 끝날 때 그 안에서 힙 메모리가 해제됩니다. 호출 이후 `main`에서 `my_vec`을 다시 쓰려 하면 컴파일 에러입니다.

**하지만 잠깐만요.** 파이썬처럼 객체를 함수에 여러 번 던져서 처리하고 싶은데, 한 번 넘겨줬다고 변수가 폐기된다면 어떻게 해야 할까요? 한 가지 방법은 함수가 소유권을 받은 뒤 처리하고 *다시 돌려주는* 것입니다. 반환값으로 되돌려주면 됩니다.

```rust
fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length) // 소유권(s)을 호출자에게 되돌려줌 + 계산 결과(length)
}

fn main() {
    let s1 = String::from("hello");

    // s1을 넘기고, 튜플로 (소유권 돌려받기 + 결과) 회수
    let (s1, len) = calculate_length(s1);

    println!("'{}' 의 길이는 {}", s1, len); // s1을 되찾았으므로 사용 가능
}
```

동작은 합니다. 하지만 이 코드가 얼마나 어색한지 보세요. 단지 문자열의 길이를 알고 싶었을 뿐인데, 문자열을 함수에 통째로 넘겼다가 튜플로 다시 받아 와야 합니다. 만약 함수가 인자를 세 개 받고 그중 두 개를 계속 써야 한다면? 반환 타입이 `(String, Vec<u8>, HashMap<...>, ActualResult)` 같은 괴물이 됩니다. 소유권을 넘겼다 돌려받는 이 핑퐁(ping-pong)은 명백히 비실용적입니다.

여기서 자연스러운 질문이 떠오릅니다. **"소유권을 넘기지 *않고*, 잠깐 읽기만(혹은 수정만) 하게 빌려줄 수는 없나?"** 정확히 그것이 다음 장의 주제입니다.

---

## 4.7 실전 예제 (Hands-on): 로그 처리 파이프라인

지금까지 배운 Move, Drop, 소유권 이전을 하나로 엮은 작은 예제를 보겠습니다. 로그 파일을 모델링하고, 그것을 처리 함수에 넘겨 ERROR 개수를 세는 파이프라인입니다. 핵심 관전 포인트는 **`LogFile`의 힙 메모리가 정확히 언제 해제되는가**입니다.

```rust
// 그대로 cargo run 가능한 완전한 예제

#[derive(Debug)]
struct LogFile {
    path: String,
    lines: Vec<String>,
}

// 파일 핸들 해제를 시뮬레이션하는 Drop
impl Drop for LogFile {
    fn drop(&mut self) {
        println!("[Drop] '{}' 핸들 해제, {} 줄 메모리 반환", self.path, self.lines.len());
    }
}

// 소유권을 받아서 파싱한다. 입력 LogFile은 이 함수 끝에서 Drop된다.
fn count_errors(file: LogFile) -> usize {
    file.lines.iter().filter(|l| l.contains("ERROR")).count()
} // 여기서 file이 Drop됨 — 호출자로 돌아가기 전에 메모리가 이미 반환된다

fn main() {
    let log = LogFile {
        path: String::from("/var/log/app.log"),
        lines: vec![
            String::from("INFO start"),
            String::from("ERROR disk full"),
            String::from("ERROR timeout"),
        ],
    };

    println!("처리 시작");
    let errors = count_errors(log); // log의 소유권이 함수로 이동
    // println!("{:?}", log);       // 🚨 컴파일 에러: borrow of moved value: `log`
    println!("발견된 ERROR: {}개", errors);
    println!("main 종료");
}
```

실행 결과를 보면 소유권과 Drop의 타이밍이 선명하게 드러납니다.

```text
처리 시작
[Drop] '/var/log/app.log' 핸들 해제, 3 줄 메모리 반환
발견된 ERROR: 2개
main 종료
```

`[Drop]` 메시지가 `"발견된 ERROR"`보다 **먼저** 찍힌다는 점에 주목하세요. `count_errors`가 `log`의 소유권을 가져갔으므로, 함수가 반환되는 순간(`main`으로 돌아가기 전에) `log`의 힙 메모리가 이미 해제됩니다. 이것이 결정적 해제입니다. 메모리 반환 시점이 소스 코드 구조만 보고도 정확히 예측됩니다. 파이썬이었다면 이 `LogFile` 객체가 언제 회수될지는 참조 카운트와 GC 사이클에 달려 있어 알 수 없었을 것입니다.

물론 실무에서는 `count_errors`가 `log`를 소비(consume)해 버리는 이 설계가 불편할 수 있습니다 — 처리 후에도 `log`를 더 쓰고 싶을 테니까요. 그 해법(참조로 빌려주기)이 바로 다음 장입니다. 지금은 "소유권을 넘기면 그쪽에서 책임지고 해제한다"는 흐름을 눈에 익히는 것이 목적입니다.

---

## 4.8 소유권이 표준 라이브러리와 생태계에 박혀 있는 방식

소유권은 언어 문법이 아니라 *생태계 전체를 관통하는 계약*입니다. 표준 라이브러리와 주요 크레이트의 API 시그니처를 읽으면, 그 함수가 값을 소비하는지(소유권 take), 빌리는지(다음 장)가 타입에 그대로 드러납니다. 시그니처가 곧 문서입니다.

* **`std::fs::File`** — 파일 핸들이 `Drop`되면 OS 파일 디스크립터가 자동으로 닫힙니다. 파이썬의 `with open(...)`이 강제했던 명시적 `close()`가 필요 없습니다. `File`을 소유한 변수가 스코프를 벗어나면 끝입니다.
* **`std::sync::MutexGuard`** — `mutex.lock()`이 반환하는 가드(guard)는 `Drop` 시점에 락을 해제합니다(17장). C++의 `std::lock_guard`, 파이썬의 `with lock:`과 같은 역할을 소유권이 자동으로 합니다. "락을 깜빡 안 풀어서 데드락" 같은 사고가 구조적으로 줄어듭니다.
* **`Box<T>`** (20장) — 힙에 값 하나를 담는 가장 단순한 스마트 포인터. `Box`가 소유권을 가지며 `Drop` 시 힙을 반환합니다. C의 `malloc`/`free` 한 쌍이 `Box`의 생성과 `Drop`으로 압축됩니다.
* **`Rc<T>` / `Arc<T>`** (20·17장) — 여기서 흥미로운 반전이 있습니다. "소유자는 단 하나"라는 규칙이 너무 엄격해서 정말로 공유 소유권이 필요할 때, Rust는 *옵트인(opt-in)* 방식으로 참조 카운팅을 제공합니다. `Rc<T>`는 파이썬의 참조 카운팅과 똑같이 동작합니다 — 다만 모든 객체에 강제되는 것이 아니라, 명시적으로 `Rc`로 감싼 데이터에만 적용됩니다. 비용을 지불할 곳을 *개발자가 선택*하는 것입니다. 그리고 `Rc`의 순환 참조 문제는 `Weak<T>`로 끊습니다(GC 없이 수동으로).
* **`drop` 기반 자원 정리 크레이트** — `tempfile` 크레이트의 `NamedTempFile`은 `Drop` 시 임시 파일을 디스크에서 삭제합니다. `scopeguard`의 `defer!` 매크로는 Go의 `defer`처럼 스코프 종료 시 임의의 정리 코드를 실행합니다. 모두 RAII 위에 세워진 추상화입니다.

> **Under the Hood: `Rc`는 파이썬 GC의 "옵트인" 버전이다**
>
> 파이썬은 *모든* 객체에 참조 카운트(`ob_refcnt`)를 강제합니다. 정수 하나도, 절대 공유되지 않을 임시 객체도 카운터 비용을 냅니다. Rust의 기본값은 정반대입니다 — 카운터가 *전혀 없습니다.* 그러다 진짜로 그래프 구조(여러 부모가 한 자식을 공유하는 트리, 양방향 연결 리스트 등)가 필요하면, 그 부분만 `Rc<T>`로 감쌉니다.
>
> ```rust
> use std::rc::Rc;
>
> fn main() {
>     let shared = Rc::new(vec![1, 2, 3]); // refcount = 1
>     let a = Rc::clone(&shared);           // refcount = 2 (힙 복사 아님! 카운터만 +1)
>     let b = Rc::clone(&shared);           // refcount = 3
>     println!("count = {}", Rc::strong_count(&shared)); // 3
> } // b, a, shared 순으로 Drop되며 refcount 3→2→1→0, 0에서 힙 해제
> ```
>
> `Rc::clone`은 이름이 `clone`이지만 힙 데이터를 복사하지 않습니다. 포인터를 복사하고 카운터를 +1 할 뿐입니다(파이썬 `Py_INCREF`와 동일). 비용은 정수 증가 한 번입니다. 단일 스레드용이라 원자적 연산조차 쓰지 않아 파이썬보다 가볍습니다. 멀티스레드라면 `Arc`(Atomic Rc)를 쓰며, 이때 비로소 원자적 카운터 비용이 듭니다. 요점은 명확합니다 — 파이썬이 *모두에게* 부과하던 GC 세금을, Rust는 *그것이 정말 필요한 자료구조에만* 청구합니다.

### 메모리 풋프린트, 그 수치의 무게

소유권 모델이 만들어 내는 차이를 구체적인 수치로 못 박아 봅시다. `i64` 정수 100만 개를 담는다고 합시다.

* **Rust `Vec<i64>`**: 8바이트 × 100만 = 정확히 8,000,000바이트 ≈ **7.6MB**. 연속된 단일 힙 버퍼 하나. 캐시 친화적이고, 헤더 오버헤드 0.
* **Python `list[int]`** (캐시되지 않는 큰 정수, 예: `2**40 + i`): 포인터 배열 ≈ 8.4MB + 각 정수 객체 32바이트 × 100만 = 32MB → 합계 **약 40MB**(`sys.getsizeof` 실측: 컨테이너 8,448,728B + 정수 객체 32MB ≈ 40.4MB). 정수 객체들이 힙 여기저기에 흩어져 캐시 미스를 유발.

**메모리는 약 5배(40.4MB / 8MB ≈ 5.1배), 순회 속도는 캐시 지역성(cache locality) 차이로 그 이상** 벌어집니다. 이것은 마케팅 수치가 아니라 `sys.getsizeof()`로 직접 확인되는 사실입니다(작은 `int`는 28바이트, 워드를 넘기는 `int`는 32바이트, 더 큰 `int`는 40바이트). 데이터 엔지니어링에서 수십 GB짜리 배열을 다룰 때 이 5배는 "장비를 한 대 더 살 것이냐"를 가르는 차이가 됩니다. 4.13절에서 N=10M으로 합산·메모리를 실측하고, 14장에서 `Vec`과 컬렉션을 본격적으로 다룰 때 이 주제로 돌아옵니다.

> **벤치마크 정직성 노트:** Move와 Clone의 비용 차이를 `std::time::Instant`로 마이크로 벤치마크하려는 시도는 권하지 않습니다. Move/Copy는 수 나노초 단위라 타이머 해상도와 옵티마이저의 재배치(reordering)에 묻혀 신뢰할 수 없는 숫자가 나옵니다(릴리스 빌드에서는 데드 코드로 사라지기도 합니다). 정직한 추정은 4.4절의 대역폭 기반 계산입니다 — 1MB clone ≈ 50~100μs(memcpy 10~20GB/s 기준), Move는 24바이트 복사로 사실상 0. 진짜 벤치마크가 필요하면 `criterion` 크레이트를 쓰세요. 블랙박스 처리와 반복 측정으로 옵티마이저의 장난을 막아 줍니다.

---

## 4.9 Best Practices & 안티패턴

실무에서 소유권을 다룰 때의 체크리스트입니다. 5장(대여)을 배우기 전이라 일부는 "다음 장에서 더 우아해진다"는 단서가 붙습니다 (이하 4.9·4.10절).

**이렇게 하라:**

* **기본은 Move를 신뢰하라.** Move는 거의 공짜(스택 헤더 memcpy)입니다. 소유권을 넘기는 것을 두려워하지 마세요. 함수가 값을 끝까지 소비하고 책임진다면 소유권을 받는 시그니처(`fn f(v: Vec<T>)`)가 자연스럽습니다.
* **`Copy`는 작은 값에만.** 직접 만든 구조체에 `#[derive(Copy, Clone)]`을 붙일 때는 정말 작고 힙을 소유하지 않는 경우(예: 좌표 `Point { x: f64, y: f64 }`)로 한정하세요. 큰 구조체에 `Copy`를 붙이면 묵시적 복사가 성능 함정이 됩니다.
* **결정적 해제를 활용하라.** 거대한 버퍼를 다 썼다면 스코프 끝까지 기다리지 말고 `drop(buf)`로 즉시 반환해 메모리 풋프린트를 낮추세요.
* **`Drop`은 자원 정리에만.** DB 커넥션 반환, 임시 파일 삭제, 락 해제처럼 "스코프를 벗어나면 반드시 해야 할 정리"를 `Drop`에 넣으세요. 파이썬의 `try/finally`나 `with`가 하던 일을 언어가 자동화해 줍니다.

**이건 피하라:**

* **`.clone()` 남발(가장 흔한 함정).** 컴파일 에러를 빨리 없애려고 `borrow of moved value` 에러가 날 때마다 기계적으로 `.clone()`을 붙이는 습관. 컴파일은 통과하지만 핫 패스에서 힙 복사가 폭증합니다. clippy가 일부를 잡아 주지만, 근본 해법은 대부분 5장의 참조(`&T`)입니다.
* **`Drop`에 무거운 로직.** `drop`은 패닉(panic)을 던지면 안 되고, 블로킹 I/O나 긴 작업을 넣으면 스코프 종료마다 예측 못 한 지연이 발생합니다. 가볍게 유지하세요.
* **`Copy` 타입의 `Clone` 호출.** `let y = x.clone();` (단, `x: i32`)처럼 이미 `Copy`인 값에 `.clone()`을 붙이는 것은 무의미합니다. clippy `clone_on_copy` 경고가 뜹니다.

> **clippy 연계:** `cargo clippy`는 `redundant_clone`(불필요한 clone), `clone_on_copy`(Copy 타입의 clone) 등을 자동 지적합니다. CI에 `cargo clippy -- -D warnings`를 넣어 두면 무심코 들어간 clone이 빌드를 깨뜨려 리뷰 전에 걸러집니다.

---

## 4.10 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: "대입은 별칭일 뿐"이라는 파이썬식 직관.**

```python
items = [1, 2, 3]
backup = items      # 파이썬: backup은 같은 리스트를 가리킴, items도 멀쩡
```

파이썬 개발자는 `let backup = items;` 후에도 `items`를 쓸 수 있다고 무의식적으로 가정합니다. Rust에서 `items`가 `Vec`이면 이 대입은 Move이고 `items`는 죽습니다. **해법:** 정말 두 개의 독립 복사본이 필요하면 `.clone()`, 단지 둘 다 읽고 싶을 뿐이면 다음 장의 참조(`&items`).

**함정 2: 컴파일 에러를 `.clone()`으로 도배하기.** 4.8절에서 언급한, 가장 만연한 안티패턴입니다. `borrow of moved value`가 뜰 때마다 반사적으로 `.clone()`을 붙이면 동작은 하지만 Rust를 쓰는 의미가 없어집니다. 에러를 만나면 먼저 "내가 이 값의 소유권이 정말 필요한가, 아니면 잠깐 빌리면 되는가?"를 자문하세요. 대부분은 후자이고, 그 답은 `&`입니다.

**함정 3: 반복문 안에서 의도치 않게 Move.**

```rust
fn main() {
    let names = vec![String::from("alice"), String::from("bob")];

    for name in names {       // 주의: names의 소유권이 for로 이동, 각 name도 소유권을 가짐
        println!("{}", name);
    }

    // println!("{:?}", names);
    // 🚨 컴파일 에러! for가 names를 소비(consume)해 버렸다.
}
```

`for name in names`는 `names`를 통째로 소비합니다(`IntoIterator`). 루프 후 `names`를 또 쓰려면 에러입니다. **해법:** `for name in &names`처럼 참조로 순회하면 `names`는 살아남습니다(15장 이터레이터에서 깊이 다룹니다). 또 한 번, 답은 `&`입니다.

**함정 4: 함수 인자를 무조건 소유권으로 받기.** 파이썬에서 함수에 객체를 넘기는 데 익숙한 개발자는 Rust에서도 `fn process(data: Vec<u8>)`처럼 소유권을 받는 시그니처를 남발하기 쉽습니다. 그러면 호출자는 매번 소유권을 잃거나 `.clone()`해야 합니다. **해법:** 함수가 값을 읽기만 한다면 `fn process(data: &[u8])`처럼 참조를 받아야 합니다. 이것이 관용적(idiomatic) Rust이며, 역시 다음 장의 핵심입니다.

**함정 5: 부분 이동(Partial Move) 후 전체 구조체 사용.** 파이썬에서는 `name = user.name`으로 속성 하나를 꺼내도 `user` 객체는 아무 영향 없이 멀쩡합니다. Rust에서 구조체의 `Copy`가 아닌 필드를 꺼내면 그 필드만 *부분 이동(partially moved)*되고, 그 순간부터 구조체 전체를 다시 쓸 수 없습니다.

```rust
#[derive(Debug)]
struct User {
    name: String,   // String은 Copy가 아니다
    email: String,
}

fn main() {
    let u = User {
        name: String::from("alice"),
        email: String::from("alice@example.com"),
    };

    let name = u.name;       // u.name이 'name'으로 부분 이동(move)됨
    println!("{:?}", u);     // 🚨 컴파일 에러: u가 부분 이동된 상태로 사용됨
}
```

컴파일러는 이 상황을 정확히 짚어 줍니다.

```text
error[E0382]: borrow of partially moved value: `u`
  --> src/main.rs:17:22
   |
14 |     let name = u.name;
   |                ------ value partially moved here
...
17 |     println!("{:?}", u);
   |                      ^ value borrowed here after partial move
   |
   = note: partial move occurs because `u.name` has type `String`,
           which does not implement the `Copy` trait
```

흥미로운 점은, 부분 이동 후에도 *이동되지 않은 다른 필드*(`u.email`)는 여전히 개별적으로 접근할 수 있다는 것입니다. 컴파일러는 구조체를 통째로가 아니라 필드 단위로 이동 상태를 추적합니다(disjoint move tracking). 즉 `println!("{}", u.email)`은 통과하지만, `u` 전체를 빌리거나 `{:?}`로 통째로 찍는 것은 거부됩니다. **해법:** 구조체 전체가 계속 살아 있어야 한다면 필드를 move하지 말고 참조(`let name = &u.name;`)하거나, 정말 독립 복사가 필요하면 `u.name.clone()`을 쓰세요(컴파일러가 두 해법을 모두 `help:`로 제안합니다).

여기에 한 단계 더 미묘한 함정이 숨어 있습니다. **구조체가 `Drop`을 구현하면 필드를 아예 꺼낼 수조차 없습니다.**

```rust
struct Session {
    token: String,
}

impl Drop for Session {
    fn drop(&mut self) {
        println!("[Drop] 세션 정리: {}", self.token);
    }
}

fn main() {
    let s = Session { token: String::from("abc123") };
    let token = s.token;   // 🚨 컴파일 에러!
    println!("{}", token);
}
```

```text
error[E0509]: cannot move out of type `Session`, which implements the `Drop` trait
  --> src/main.rs:14:17
   |
14 |     let token = s.token;
   |                 ^^^^^^^
   |                 |
   |                 cannot move out of here
   |                 move occurs because `s.token` has type `String`,
   |                 which does not implement the `Copy` trait
```

이것은 부분 이동이 막히는 것과는 별개의 더 근본적인 제약(`error[E0509]`)입니다. 왜일까요? `Session`은 스코프 끝에서 `drop(&mut self)`이 호출되며 `self.token`을 읽습니다. 만약 `token` 필드를 미리 꺼내 가도록 허용하면, `drop`이 호출될 때 `self.token`은 이미 이동되어 *빈 껍데기*가 됩니다. Rust는 이 모순을 원천 차단하기 위해 "`Drop`을 구현한 타입에서는 필드를 부분 이동할 수 없다"는 규칙을 강제합니다. **해법:** `std::mem::take`나 `std::mem::replace`로 필드를 꺼내면서 기본값(`String::new()`)을 채워 넣어 구조체의 무결성을 유지하거나, 참조로 빌리세요.

> **Under the Hood: use-after-free가 컴파일 타임에 막히는 원리**
>
> C에서 가장 악명 높은 두 버그는 double-free와 use-after-free입니다.
>
> ```c
> char *p = malloc(100);
> char *q = p;     // 둘 다 같은 메모리를 가리킴
> free(p);         // 해제
> q[0] = 'x';      // 💀 use-after-free — q는 이미 해제된 메모리를 씀 (UB)
> free(q);         // 💀 double-free — 같은 메모리를 두 번 해제 (UB)
> ```
>
> 이 코드는 컴파일도 되고 운이 좋으면 실행도 됩니다. 그러다 어느 날 프로덕션에서 메모리 오염으로 터지거나, 보안 취약점(CVE)이 됩니다. C 컴파일러는 `p`와 `q`가 같은 메모리를 가리키는지, `free` 이후에 접근하는지 추적하지 않습니다.
>
> Rust에서 같은 패턴은 컴파일 자체가 거부됩니다. `let q = p;`가 Move이므로 `p`는 무효화되고, `q[0] = 'x'`는 유일한 소유자인 `q`를 통한 접근이므로 안전합니다. 그리고 `q`는 정확히 한 번, 스코프 끝에서만 자동 해제됩니다. double-free를 일으키려면 같은 값을 두 번 해제해야 하는데, 소유자가 항상 한 명뿐이고 Move 후 원본은 죽으므로 **물리적으로 두 번 해제할 코드를 작성할 수 없습니다.** use-after-free 역시 마찬가지입니다 — Move 후 원본을 쓰면 `error[E0382]`로 거부됩니다.
>
> 빌림 검사기는 이것을 런타임 검사가 아니라 정적 데이터 흐름 분석(static data-flow analysis)으로 증명합니다. 즉, 안전성에 대한 대가가 0입니다. 컴파일된 바이너리는 C와 동일한 raw 포인터 연산을 하지만, 그 포인터가 유효함이 컴파일 타임에 수학적으로 보장됩니다. 이것이 Rust가 말하는 "메모리 안전성을 런타임 비용 없이(memory safety without a garbage collector)"의 진짜 의미입니다.

---

## 4.11 Move의 진짜 모습 — MIR과 copy-elision

4.3절에서 "Move는 스택 헤더 24바이트를 memcpy하는 얕은 복사"라고 설명했습니다. 그것은 *의미론(semantics)* 수준에서 정확한 모델입니다. 하지만 한 단계 더 내려가면, 시니어가 알아두면 illuminating한 사실이 있습니다 — **컴파일러는 그 24바이트 memcpy조차 대부분 0개로 만들어 버립니다.** Move가 "공짜에 가깝다"가 아니라 종종 *진짜로 0개의 명령어*가 되는 것입니다.

이것을 눈으로 확인하려면 Rust 컴파일러의 중간 표현인 **MIR(Mid-level Intermediate Representation)**을 봐야 합니다. MIR은 빌림 검사기가 실제로 분석하는 표현이자, 소유권/Move/Drop이 명시적으로 드러나는 계층입니다. 다음 함수를 봅시다.

```rust
pub fn make() -> String {
    let s = String::from("hello");
    let s2 = s;   // move
    s2            // move out (반환)
}
```

소스만 보면 "`String`을 만들고 → `s`에서 `s2`로 24바이트 move → `s2`를 반환하며 또 한 번 move", 즉 두 번의 24바이트 복사가 일어날 것 같습니다. `rustc --emit=mir`로 *최적화를 전혀 켜지 않은* MIR을 뽑아 보면 진실이 드러납니다.

```text
fn make() -> String {
    let mut _0: std::string::String;   // 반환 슬롯
    scope 1 {
        debug s => _0;                 // s  == _0
        scope 2 {
            debug s2 => _0;            // s2 == _0  (같은 로컬!)
        }
    }

    bb0: {
        _0 = <String as From<&str>>::from(const "hello") -> [return: bb1, ...];
    }
    bb1: {
        return;
    }
}
```

`s`, `s2`, 그리고 반환 슬롯 `_0`이 **전부 동일한 단일 로컬 `_0`에 매핑**됩니다. `From::from`의 결과를 처음부터 반환 슬롯에 직접 써넣었으므로, `let s2 = s;`와 `return s2;`의 두 move는 단 한 줄의 데이터 이동 명령도 만들지 않습니다. 이것이 **copy-elision(복사 생략)**입니다. 컴파일러가 "이 값들은 결국 같은 자리에 있어야 한다"를 증명하고 메모리 슬롯을 합쳐 버린 것입니다(slot coalescing). 주목할 점은 이것이 `-O` 없이도, *디버그 빌드에서도* 일어난다는 사실입니다.

> **Under the Hood: Move가 memcpy 0개로 컴파일되는 것**
>
> 최적화 빌드(`rustc -O --emit=asm`)에서 위 `make` 함수의 어셈블리를 뽑으면, 24바이트를 옮기는 `memcpy` 호출도, `callq`로 표시되는 어떤 함수 호출도 나타나지 않습니다(`From::from`마저 인라인되어 사라집니다). Move 두 번이 *물리적으로 0개의 명령어*로 컴파일된 것입니다.
>
> 그렇다고 Move가 *항상* 0이라는 뜻은 아닙니다. 값이 서로 다른 메모리 슬롯 사이를 진짜로 건너가야 할 때(예: 구조체 필드 하나를 꺼내 다른 변수로 옮길 때)는 MIR에 `move` 오퍼랜드가 남고, 코드젠 단계에서 24바이트 복사로 내려갑니다. 다음 MIR을 보세요.
>
> ```text
> fn take(_1: Pair) -> String {
>     bb0: {
>         _0 = move (_1.0: std::string::String);   // 필드 a를 move
>         drop((_1.1: std::string::String)) -> ...; // 필드 b는 여기서 Drop
>     }
> }
> ```
>
> `_0 = move (_1.0)`처럼 `move` 키워드가 MIR 오퍼랜드로 명시적으로 박힙니다. 그리고 더 중요한 것 — `take`가 `Pair`의 소유권을 받았으므로, 꺼내 가지 *않은* 나머지 필드 `_1.1`(즉 `b`)은 이 함수 끝에서 자동으로 `drop`됩니다. 부분 이동된 구조체의 "남은 필드만 정확히 해제"하는 이 로직(drop flag/drop elaboration)이 전부 컴파일 타임에 MIR에 새겨집니다. 런타임에 "어느 필드가 살아 있나"를 추적하는 코드는 없습니다 — 정적 분석의 결론이 그대로 명령어가 됩니다.
>
> 정리하면 Move의 비용은 세 단계입니다. (1) 슬롯이 합쳐지면 **0개 명령어**(copy-elision), (2) 합쳐지지 않으면 **고정 크기 memcpy**(`String`이면 24바이트, 스택 한두 번), (3) 어떤 경우에도 **힙 버퍼는 절대 건드리지 않음**. C++의 이동 생성자(move constructor)가 사용자 코드를 호출할 수 있는 것과 달리, Rust의 Move는 언어가 보장하는 단순 비트 이동(혹은 그조차 생략)이라 예측 가능합니다.

---

## 4.12 `'static` 경계와 move 클로저 — 스레드로 가는 다리

소유권 모델은 단일 스레드 안에서만 마법을 부리는 것이 아닙니다. Rust가 "두려움 없는 동시성(fearless concurrency)"을 표방하는 근거 중 절반이 바로 이 장에서 배운 Move에 있습니다. 16·17장에서 본격적으로 다루겠지만, 그 다리를 여기서 미리 놓아 둡시다. 새 스레드를 띄우는 표준 함수의 시그니처를 보세요.

```rust
// std::thread
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,   // ← 'static에 주목
    T: Send + 'static,
```

`F: ... + 'static`이라는 경계가 핵심입니다. 새 스레드는 그것을 띄운 함수보다 *더 오래 살 수 있습니다.* `main`이 끝나도 자식 스레드는 돌고 있을 수 있죠. 그렇다면 그 스레드가 실행하는 클로저가 부모 함수의 지역 변수를 *참조*로 들고 있다면? 부모의 스택이 사라진 뒤 죽은 메모리를 가리키는 dangling 참조가 됩니다. C/Pthreads 세계에서 흔한 use-after-free의 정확한 레시피입니다.

Rust는 이를 `'static` 경계로 컴파일 타임에 차단합니다. 다음은 참조를 캡처하려는(즉 `'static`이 아닌) 클로저입니다.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let handle = thread::spawn(|| {     // move 없이 data를 참조 캡처
        println!("{:?}", data);
    });
    handle.join().unwrap();
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `data`,
              which is owned by the current function
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `data`
7 |         println!("{:?}", data);
  |                          ---- `data` is borrowed here
  |
note: function requires argument type to outlive `'static`
help: to force the closure to take ownership of `data` (and any other
      referenced variables), use the `move` keyword
  |
6 |     let handle = thread::spawn(move || {
  |                                ++++
```

컴파일러의 진단이 정확합니다 — "클로저가 현재 함수보다 오래 살 수 있는데 `data`를 빌리고 있다", 그리고 "인자 타입이 `'static`을 만족해야 한다(`function requires argument type to outlive 'static`)". 해법도 제시합니다: `move` 키워드를 붙이라는 것입니다.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    // move 클로저: data의 소유권을 클로저 안으로, 그리고 새 스레드로 이동시킨다
    let handle = thread::spawn(move || {
        println!("스레드에서: {:?}", data);
    });
    // println!("{:?}", data); // 🚨 컴파일 에러: data는 스레드로 move됨
    handle.join().unwrap();
}
```

`move`를 붙이는 순간 `data`의 소유권이 클로저 안으로 들어가고, 클로저가 그 `Vec`을 *소유*하므로 부모 스택과 무관하게 독립적으로 살아남습니다. 이제 클로저는 `'static`을 만족합니다(외부에서 빌린 참조가 하나도 없으니까요). 그리고 이 장에서 배운 규칙에 따라, move 이후 `main`에서 `data`를 다시 쓰면 컴파일 에러입니다. **소유권이 스레드 경계를 넘어 이동한 것**입니다.

> 💡 여기서 4장의 Move와 16/17장의 동시성이 만납니다. 파이썬에서는 스레드 간 데이터 공유가 그냥 같은 객체를 가리키는 일이라 GIL과 락으로 사후 방어합니다. Rust에서는 "이 데이터를 스레드에 통째로 넘길 것인가(move), 아니면 여러 스레드가 공유할 것인가(`Arc`, 17장)"를 *타입과 `move` 키워드로 먼저 선언*하게 만듭니다. 데이터 경합(data race)을 런타임 락이 아니라 컴파일 타임 소유권으로 푸는 것입니다. 스레드에 `Vec`을 넘기는 순간 호출자가 그것을 잃는 이 불편함이, 사실은 동시성 버그를 통째로 제거하는 안전장치입니다.

---

## 4.13 벤치마크: `list[int]` vs `Vec<i64>` — 합산과 메모리 실측

지금까지의 메모리 논의를 추상에서 끌어내려 실제 숫자로 못 박겠습니다. 정수 1천만 개(N = 10,000,000)를 담고 전부 더하는, 데이터 파이프라인에서 흔한 작업입니다. 측정 환경은 Python 3.13.6, rustc 1.96.0(`-O` 릴리스 빌드), 동일 머신입니다. 양쪽 모두 값 `0..10,000,000`을 더하며, 합계는 정확히 `49,999,995,000,000`으로 일치합니다(결과 검증 완료).

**Python**

```python
import time, sys

N = 10_000_000
data = list(range(N))

t0 = time.perf_counter()
total = 0
for x in data:
    total += x
loop_ms = (time.perf_counter() - t0) * 1000

t0 = time.perf_counter()
total2 = sum(data)              # 내장 sum (C로 구현된 빠른 경로)
builtin_ms = (time.perf_counter() - t0) * 1000

print(total, loop_ms, builtin_ms, sys.getsizeof(data))
```

**Rust** (`rustc -O`, best-of-5)

```rust
use std::time::Instant;

fn main() {
    const N: usize = 10_000_000;
    let data: Vec<i64> = (0..N as i64).collect();

    let mut best = f64::MAX;
    for _ in 0..5 {
        let t0 = Instant::now();
        let mut total = 0i64;
        for &x in &data {
            total += x;
        }
        let ms = t0.elapsed().as_secs_f64() * 1000.0;
        if ms < best { best = ms; }
        std::hint::black_box(total);  // 옵티마이저가 루프를 지우지 못하게
    }
    println!("best-of-5: {:.3} ms", best);
    println!("mem: {} MB", N * std::mem::size_of::<i64>() / 1_000_000);
}
```

측정 결과입니다.

| 항목 | Python `for` 루프 | Python `sum()` 내장 | Rust `for` 루프 (`-O`) |
|---|---|---|---|
| 합산 시간 (N=10M) | ~3,250 ms | ~96 ms | ~1.2 ms (best-of-5) |
| Rust 대비 배수 | 약 2,700배 | 약 80배 | 1배 (기준) |
| 컨테이너 자체 크기 | 80,000,056 B | (동일) | 80,000,000 B (`Vec<i64>`) |

세 가지를 짚어야 정직한 해석입니다.

첫째, **Python `for` 루프의 3,250ms는 다소 극단적인 수치**입니다. 누적 변수 `total`이 머신 워드(64비트)를 넘어서는 순간 CPython이 임의 정밀도 큰 정수(bignum)로 승격시키고, 매 반복마다 정수 객체를 새로 할당·해제하기 때문입니다. 이 경로는 단순 합산이라기보다 "객체 할당 벤치마크"에 가깝습니다. 그래서 약 2,700배라는 숫자는 인상적이지만 일반화하면 안 됩니다.

둘째, **공정한 비교는 내장 `sum()`과의 80배**입니다. `sum()`은 CPython 인터프리터 바이트코드 루프를 우회해 C로 최적화된 경로를 타므로, 파이썬으로 할 수 있는 거의 최선입니다. 그조차 Rust보다 80배 느립니다. 이 80배가 "관용적 파이썬 vs 관용적 Rust"의 현실적인 핫루프 격차입니다.

셋째, **메모리는 컨테이너만 봐도 동률(80MB)이지만 함정이 있습니다.** `sys.getsizeof(data)`가 보고하는 80MB는 *포인터 배열*(8바이트 × 1천만)의 크기일 뿐, 그 포인터들이 가리키는 정수 객체 자체는 포함하지 않습니다. `0..256` 범위의 작은 정수는 CPython이 캐싱하므로 추가 비용이 없지만, 값이 그 범위를 벗어나거나(예: `2**40 + i`) 일반적인 데이터라면 각 정수가 별도 힙 객체(28~40바이트)로 흩어집니다. 실제로 `[2**40 + i for i in range(10_000_000)]`의 총 메모리는 약 **409MB**입니다(포인터 배열 89MB + 정수 객체 32B × 1천만 ≈ 320MB). 같은 1천만 개 정수가 Rust `Vec<i64>`에서는 **정확히 80MB의 연속 버퍼**입니다. **약 5배 차이**, 그리고 흩어진 객체들이 유발하는 캐시 미스 때문에 순회 속도 격차는 그보다 더 벌어집니다.

> **Under the Hood: 왜 합산에서 1,000배 가까운 격차가 나는가**
>
> Rust의 `for &x in &data` 루프는 `data`의 연속된 8바이트 정수들을 캐시 라인 단위로 순차 로드하며, `total += x`는 단일 `add` 명령입니다. CPU의 하드웨어 프리페처가 다음 캐시 라인을 미리 끌어오므로 이 루프는 사실상 메모리 대역폭에 의해서만 제한됩니다(80MB를 한 번 읽는 시간 ≈ 1ms). 컴파일러는 여기에 SIMD 자동 벡터화까지 적용할 수 있습니다.
>
> CPython의 `for` 루프는 매 반복마다 (1) 바이트코드 디스패치(`FOR_ITER`, `BINARY_OP`), (2) 포인터 역참조로 흩어진 정수 객체 접근(캐시 미스 빈발), (3) `int.__add__` 호출과 결과 정수 객체의 새 힙 할당, (4) 참조 카운트 증감을 수행합니다. 데이터 한 개를 더하는 데 수십 개의 머신 명령과 잠재적 힙 할당이 듭니다. `total`이 bignum으로 승격된 뒤에는 매번 다중 워드 산술까지 더해집니다. 80배(내장 sum)에서 2,700배(순수 루프)에 이르는 격차는 이 구조적 차이에서 나옵니다 — 언어가 느린 것이 아니라, 동적 타입과 객체 모델의 비용을 매 연산마다 지불하는 것입니다.
>
> **벤치마크 정직성 노트:** 위 Rust 수치는 `std::time::Instant`로 best-of-5를 측정한 것이며, 80MB 버퍼를 읽는 메모리 바운드 작업이라 실행마다 1~5ms로 흔들립니다. 정밀한 회귀 추적이 필요하면 `criterion` 크레이트를 쓰세요. 통계적 샘플링과 `black_box`로 옵티마이저의 데드코드 제거를 막아 신뢰 구간이 있는 수치를 줍니다. 이 절의 벤치도 `black_box` 없이는 Rust 컴파일러가 합산 루프 전체를 상수로 접어 버려 0ms가 나옵니다.

---

## 4.14 연습문제

Rustlings 스타일로, 손으로 풀고 컴파일러로 채점하세요. 각 문항의 "정답 스케치"는 핵심 아이디어와 함께 자기 검증용 컴파일러 에러코드 또는 측정값을 담고 있습니다. 먼저 풀고, 막히면 펼치세요.

**문제 1 — Move 진단.** 다음 코드가 왜 컴파일되지 않는지 에러코드를 예측하고, 코드를 *세 가지 방법*으로 고치세요: (a) `clone`, (b) 호출 순서 변경 없이 참조 사용은 다음 장이므로 제외하고, (b') 함수가 합계를 반환하도록 시그니처를 바꿔 두 번 호출이 필요 없게 만들기.

```rust
fn summarize(v: Vec<i32>) -> i32 {
    v.iter().sum()
}

fn main() {
    let nums = vec![10, 20, 30];
    let a = summarize(nums);
    let b = summarize(nums);   // ???
    println!("{} {}", a, b);
}
```

> **정답 스케치.** 첫 호출 `summarize(nums)`가 `nums`의 소유권을 가져가므로 두 번째 호출은 `error[E0382]: use of moved value: \`nums\``로 거부됩니다. (a) 두 번째 호출만 `summarize(nums.clone())` — 단 첫 호출도 소비하므로 사실은 첫 호출을 `nums.clone()`으로 바꾸고 두 번째에 `nums`를 넘겨야 합니다. (b') `summarize`가 이미 합계를 반환하므로 `let a = summarize(nums); let b = a;`처럼 결과를 재사용하면 `Vec`을 두 번 넘길 필요 자체가 없습니다 — 이것이 가장 관용적입니다. 자기 검증: 수정 전 에러코드가 `E0382`이면 진단 정확.

**문제 2 — 부분 이동과 Drop.** 아래 구조체에서 `token` 필드를 꺼내 별도 변수로 옮기려 합니다. 컴파일되지 않는 이유의 에러코드를 맞히고, `std::mem`의 함수 하나로 고치세요.

```rust
struct Session {
    token: String,
}
impl Drop for Session {
    fn drop(&mut self) {
        println!("[Drop] {}", self.token);
    }
}
fn main() {
    let mut s = Session { token: String::from("abc123") };
    let token = s.token;   // ???
    println!("{}", token);
}
```

> **정답 스케치.** `Session`이 `Drop`을 구현하므로 필드를 move로 꺼낼 수 없어 `error[E0509]: cannot move out of type \`Session\`, which implements the \`Drop\` trait`가 납니다(부분 이동의 `E0382`가 아니라 더 근본적인 `E0509`라는 점이 포인트). 해법: `let token = std::mem::take(&mut s.token);` — `token` 자리에 빈 `String`을 채워 넣어 구조체 무결성을 지키므로 `drop`이 안전하게 실행됩니다. 실행하면 `[Drop]`이 빈 토큰과 함께 찍힙니다(`mem::take`가 원래 값을 꺼내 갔으니까요).

**문제 3 — `'static`과 스레드.** 다음 코드는 컴파일되지 않습니다. 에러코드를 맞히고, *한 글자도 아닌 한 단어*만 추가해 고치세요. 그리고 그 수정 후 `main`의 마지막 줄 주석을 풀면 어떤 새 에러가 나는지 예측하세요.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let handle = thread::spawn(|| {
        println!("{:?}", data);
    });
    handle.join().unwrap();
    // println!("{:?}", data);
}
```

> **정답 스케치.** 클로저가 `data`를 참조로 캡처하는데 `thread::spawn`은 `F: 'static`을 요구하므로 `error[E0373]: closure may outlive the current function`이 납니다(진단 노트에 `function requires argument type to outlive 'static`). 수정: 클로저 앞에 `move` 한 단어 추가 → `thread::spawn(move || { ... })`. 그러면 `data`의 소유권이 스레드로 이동합니다. 마지막 줄 주석을 풀면 이제 `data`가 이미 move되었으므로 `error[E0382]: borrow of moved value: \`data\``가 납니다 — 소유권이 스레드 경계를 넘었다는 증거입니다.

**문제 4 — Move의 진짜 비용 측정.** "`Vec`을 함수에 넘기는 Move는 힙을 복사하지 않는다"를 직접 증명하세요. 10MB짜리 `Vec<u8>`을 만들고, (a) 함수에 *소유권으로* 넘긴 직후 같은 힙을 가리키는지 포인터로 확인하고, (b) `clone()`한 버전은 다른 힙 주소를 가짐을 확인하세요. 힌트: `Vec::as_ptr()`.

```rust
fn consume(v: Vec<u8>) -> *const u8 {
    v.as_ptr()   // 받은 Vec의 힙 시작 주소
}
fn main() {
    let big = vec![0u8; 10 * 1024 * 1024];
    let before = big.as_ptr();
    // (a) Move로 넘긴 뒤 주소 비교, (b) clone 주소 비교를 작성하세요
}
```

> **정답 스케치.** (a) `let cloned = big.clone(); let cloned_ptr = cloned.as_ptr();`로 clone 주소를 먼저 잡아 둔 뒤, `let after = consume(big);`로 Move. `before == after`가 `true`여야 합니다 — Move는 스택 헤더 24바이트만 옮기고 힙 버퍼(10MB)는 그대로 두므로 시작 주소가 동일합니다. (b) `before != cloned_ptr`가 `true` — `clone`은 `malloc`으로 새 10MB 버퍼를 받아 `memcpy`하므로 다른 주소입니다. 자기 검증: `assert_eq!(before, after); assert_ne!(before, cloned_ptr);`가 패닉 없이 통과하면 정답. 이 한 줄이 4.4절의 비용표("Move=24B, Clone=O(n)+할당")를 실측으로 증명합니다.

---

## 맺음말

이번 장에서 우리는 파이썬의 GC와 참조 카운팅을 Rust가 어떻게 컴파일 타임 소유권으로 대체했는지 보았습니다. 핵심을 다시 정리하면 이렇습니다. 모든 값은 소유자가 하나뿐이고, 대입과 함수 호출은 소유권을 Move하며(스택 헤더만 복사, 원본은 무효화), 소유자가 스코프를 벗어나면 `Drop`이 결정적으로 호출되어 자원이 해제됩니다. 비싼 힙 복사는 오직 `.clone()`에서만 일어나고 코드에 명시적으로 드러납니다. 그 결과 double-free와 use-after-free가 컴파일 타임에 차단되며, 런타임에는 GC도 참조 카운터도 없습니다.

하지만 우리는 동시에 소유권 모델의 불편함도 정직하게 마주했습니다. 함수에 값을 한 번 넘기면 잃어버리고, 되돌려받으려면 튜플 핑퐁을 해야 하며, 반복문 하나가 컬렉션을 통째로 삼켜 버립니다. 4장에서 만난 거의 모든 함정의 해법이 "그냥 `&`를 붙여라"였다는 점을 눈치챘을 것입니다.

소유권을 빼앗기는 이 답답한 상황을 타개하기 위해, Rust는 객체의 주권을 유지한 채 다른 함수에게 값을 **"잠시 빌려주는"** 메커니즘을 고안했습니다. 소유권은 그대로 둔 채 읽기 권한(`&T`) 또는 쓰기 권한(`&mut T`)만 빌려주는 것입니다. 이것이 바로 다음 장에서 다룰 Rust 철학의 정수, **참조(References)와 대여(Borrowing)**입니다. 그리고 그 빌림에도 — 예상하셨겠지만 — Data Race를 컴파일 타임에 막아 내는 엄격하고 우아한 규칙이 기다리고 있습니다.

---

## References

- https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
- https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
- https://doc.rust-lang.org/rust-by-example/scope/move.html
- https://doc.rust-lang.org/rust-by-example/scope/raii.html
- https://doc.rust-lang.org/std/ops/trait.Drop.html
- https://doc.rust-lang.org/error_codes/E0382.html
- https://doc.rust-lang.org/error_codes/E0509.html
- https://doc.rust-lang.org/error_codes/E0373.html
- https://doc.rust-lang.org/error_codes/E0507.html
- https://doc.rust-lang.org/std/thread/fn.spawn.html
- https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/index.html
- https://rustc-dev-guide.rust-lang.org/mir/index.html
- https://doc.rust-lang.org/std/mem/fn.take.html
