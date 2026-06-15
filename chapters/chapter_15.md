# Chapter 15. 이터레이터와 제너레이터(Generator)

수백 기가바이트짜리 액세스 로그에서 5xx 응답만 골라, 경로별로 집계하는 스크립트를 짜본 적 있나요? 파이썬으로 이 작업을 할 때 우리는 거의 반사적으로 제너레이터(Generator)에 손을 뻗습니다. 파일을 `for line in f:`로 한 줄씩 흘려보내고, 제너레이터 표현식으로 필터링하고, `collections.Counter`로 집계합니다. 전체 파일을 메모리에 올리지 않는 이 지연 평가(Lazy Evaluation) 파이프라인은 파이썬 데이터 처리의 정수입니다. `yield` 한 키워드만으로 함수가 일시 정지·재개 가능한 상태 기계로 변하는 마법, 우리는 이것을 사랑합니다.

그런데 이 우아함에는 청구서가 따라옵니다. 제너레이터가 값을 하나 내놓을 때마다 파이썬 인터프리터는 프레임 객체(Frame Object)를 되살리고, 바이트코드를 해석하고, 스택을 복원하고, 다시 양보(yield)하며 프레임을 박제합니다. 값 하나당 수백 나노초가 인터프리터 오버헤드로 증발합니다. 제너레이터 표현식 `(x*x for x in nums if x%2==0)`은 메모리는 아끼지만, 매 원소마다 `GET_ITER` → `FOR_ITER` → `STORE_FAST` → 조건 평가 → 함수 호출의 바이트코드 디스패치 비용을 냅니다. 그래서 우리는 "지연 평가는 메모리를 위한 것이지 속도를 위한 것이 아니다"라는 타협을 무의식중에 받아들였습니다.

Rust는 이 타협을 거부합니다. Rust의 이터레이터(Iterator)는 파이썬 제너레이터와 똑같이 지연 평가를 수행합니다. `.filter().map()` 체인은 호출하는 순간에는 아무 일도 하지 않고, 마지막에 `.collect()`나 `.sum()` 같은 소비자(Consumer)가 값을 끌어당길 때 비로소 한 원소씩 파이프라인을 통과시킵니다. 메모리 절약은 동일합니다. 그런데 결정적 차이가 있습니다. 이 고차원적이고 선언적인 체인은 컴파일 후 **손으로 짠 `for` 루프와 글자 그대로 동일한 어셈블리**로 단일형화(Monomorphization)되고 인라이닝(Inlining)됩니다. 프레임 객체도, 바이트코드 디스패치도, 함수 호출 오버헤드도 없습니다. 이것이 **무비용 추상화(Zero-cost Abstraction)** 입니다.

Chapter 14에서 우리는 `Vec<T>`, `HashMap<K, V>`, `String`이 메모리 위에서 어떻게 배치되는지를 봤습니다. 데이터를 *담는* 그릇을 익혔으니, 이번 챕터는 그 데이터를 *흘려보내는* 방법입니다. 파이썬의 `__iter__`/`__next__`, 리스트 컴프리헨션, `yield`, `itertools`가 Rust의 `Iterator`/`IntoIterator` 트레이트, 어댑터(Adapter), 소비자로 어떻게 1:1 대응되는지, 그리고 그 대응이 왜 공짜인지를 어셈블리 수준에서 증명합니다.

---

## 15.1 The Python Way — 제너레이터, 그리고 프레임 객체의 비용

파이썬에서 순회 가능(Iterable) 객체는 `__iter__`를 구현해 이터레이터(Iterator)를 반환하고, 이터레이터는 `__next__`를 구현해 값을 하나씩 내놓다가 `StopIteration` 예외로 종료를 알립니다. 이 프로토콜을 직접 구현하면 다음과 같습니다.

```python
from __future__ import annotations

class Counter:
    def __init__(self, max_val: int) -> None:
        self.count = 0
        self.max_val = max_val

    def __iter__(self) -> "Counter":
        return self

    def __next__(self) -> int:
        if self.count < self.max_val:
            self.count += 1
            return self.count
        raise StopIteration  # 종료 신호를 "예외"로 표현

for num in Counter(5):
    print(num)
```

하지만 실무에서 이렇게 클래스를 쓰는 일은 드뭅니다. 똑같은 동작을 `yield`로 압도적으로 짧게 쓸 수 있기 때문입니다.

```python
def count_up_to(max_val: int):
    count = 1
    while count <= max_val:
        yield count        # 여기서 함수가 "일시 정지"하고 값을 양보
        count += 1

for num in count_up_to(5):
    print(num)
```

`yield`의 마법은 함수가 일반 함수가 아니라 **상태를 기억하는 코루틴**으로 변한다는 것입니다. `count_up_to(5)`를 호출하면 함수 본문은 **한 줄도 실행되지 않고** 제너레이터 객체만 반환됩니다. `next()`가 호출될 때마다 인터프리터는 정지된 지점부터 실행을 재개하고, 다음 `yield`에서 다시 멈춥니다.

지연 평가의 위력은 무한 수열에서 드러납니다.

```python
def fibonacci():
    a, b = 0, 1
    while True:           # 무한 루프지만 안전하다
        yield b
        a, b = b, a + b

import itertools
first_ten = list(itertools.islice(fibonacci(), 10))
# [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

무한 제너레이터를 정의해도 `islice`로 앞의 10개만 끌어당기므로 메모리는 상수입니다. 이것이 파이썬 데이터 파이프라인의 핵심 전략입니다.

여기서 파이썬 이터레이션 프로토콜의 미묘한 지점 하나를 짚고 넘어가야 합니다. 파이썬에서 *순회 가능 객체(Iterable)* 와 *이터레이터(Iterator)* 는 다른 개념입니다. `list`는 순회 가능 객체지만 그 자체는 이터레이터가 아닙니다. `for x in my_list`을 돌 때마다 파이썬은 `iter(my_list)`를 호출해 *매번 새로운* 이터레이터를 만들어냅니다. 그래서 같은 리스트를 두 번 `for`로 돌 수 있습니다. 반면 제너레이터 객체는 자기 자신이 이터레이터이고, `__iter__`가 자기 자신(`self`)을 반환합니다. 그래서 한 번 소진된 제너레이터를 다시 `for`로 돌면 아무것도 나오지 않습니다 — 이미 끝까지 가버린 상태 기계니까요. 이 "한 번 쓰면 끝" 성질은 Rust 이터레이터에도 똑같이 적용되며, Rust는 한 발 더 나아가 소유권 시스템으로 이를 강제합니다. 소진된 이터레이터를 실수로 재사용하려 들면 컴파일 단계에서 막힙니다.

이 구분을 머릿속에 새겨두면 뒤에서 볼 `IntoIterator`(순회 가능 객체에 해당)와 `Iterator`(이터레이터에 해당)의 분리가 훨씬 자연스럽게 읽힙니다. 파이썬이 `iter()`/`__next__()`로 느슨하게 나눠둔 두 역할을, Rust는 두 개의 별도 트레이트로 명확히 못 박아 둔 것입니다.

> **Under the Hood — 제너레이터 프레임 객체의 비용**
>
> 파이썬 제너레이터가 값을 하나 내놓는 일은 결코 공짜가 아닙니다. CPython에서 제너레이터 객체(`PyGenObject`)는 자신의 실행 상태 전체를 담은 **프레임 객체(`PyFrameObject`)** 를 품고 있습니다. 프레임에는 지역 변수 배열, 평가 스택(value stack), 마지막으로 실행한 바이트코드 명령의 위치(`f_lasti`), 블록 스택이 들어 있습니다.
>
> `next(gen)`이 호출될 때마다 인터프리터는 이 프레임을 다시 "현재 실행 프레임"으로 만들고(`gen_send_ex`), 정지 지점부터 바이트코드 평가 루프(`_PyEval_EvalFrameDefault`)에 진입합니다. `yield`를 만나면 평가 스택 꼭대기 값을 호출자에게 돌려주고, 프레임을 *다시 정지 상태로 박제*한 뒤 빠져나옵니다. 즉 값 하나당 **프레임 활성화 → 바이트코드 디스패치 N개 → 프레임 정지**의 사이클이 돕니다.
>
> 게다가 파이썬 정수는 Chapter 3에서 봤듯 28바이트짜리 힙 객체(`PyObject` 헤더 + 참조 카운트 + 값)입니다. `fibonacci()`가 내놓는 `b`는 매번 새 정수 객체를 힙에 할당하고 참조 카운트를 만지는 작업을 동반합니다. 메모리는 절약하지만, *시간*은 절약하지 못합니다. 값 하나에 수백 나노초 — 1억 개를 처리하면 수십 초가 순수 인터프리터 오버헤드로 사라집니다.

---

## 15.2 The Rust Mental Model — `Iterator`는 트레이트일 뿐이다

파이썬에서 "이터레이터"는 `__next__`를 가진 객체라는 *덕 타이핑(Duck Typing)* 약속이었습니다. Rust에서 이터레이터는 단 하나의 트레이트(Trait)입니다. 표준 라이브러리의 정의를 보면 놀랄 만큼 단순합니다.

```rust
pub trait Iterator {
    type Item;                                    // 내놓을 값의 타입 (연관 타입)
    fn next(&mut self) -> Option<Self::Item>;     // 핵심: 단 하나의 필수 메서드

    // ... map, filter, take, sum 등 수십 개의 메서드는 모두
    //     next()만 있으면 자동으로 제공되는 "기본 구현(default method)"
}
```

핵심을 짚어봅시다. `next()`는 `Option<Self::Item>`을 반환합니다. 값이 있으면 `Some(값)`, 끝나면 `None`입니다. 파이썬이 종료를 `StopIteration` *예외*로 표현했던 것과 대조적으로, Rust는 종료를 타입 시스템 안의 `None`으로 표현합니다(Chapter 12에서 본 `Option<T>`가 여기서 다시 등장합니다). 예외 핸들링 비용도, 흐름 제어를 위한 `try/except`도 없습니다. 그냥 enum 값 하나입니다.

`map`, `filter`, `take` 같은 수십 개의 메서드는 전부 `next()` 위에 세워진 **기본 메서드(Default Method)** 입니다. 여러분이 `next()` 하나만 구현하면, 60개가 넘는 어댑터와 소비자가 공짜로 따라옵니다. 이것이 트레이트 기반 설계의 힘입니다 — Chapter 9에서 본 "행동을 컴파일 타임에 강제하는 트레이트"의 가장 화려한 응용입니다.

그렇다면 `yield`는 어디로 갔을까요? Rust에는 (안정화된) `yield` 키워드가 없습니다. 대신 파이썬에서 우리가 거의 쓰지 않던 *클래스 기반 이터레이터*가 Rust의 표준 방식입니다. 상태를 구조체(Struct)에 담고, `next()`가 호출될 때마다 그 상태를 한 단계 전진시킵니다.

```rust
struct Counter {
    count: u32,
    max_val: u32,
}

impl Counter {
    fn new(max_val: u32) -> Counter {
        Counter { count: 0, max_val }
    }
}

// 파이썬의 __iter__, __next__ 를 구현하는 것과 완벽히 동일합니다.
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max_val {
            self.count += 1;
            Some(self.count) // 값을 반환 (yield 에 해당)
        } else {
            None // 반복 종료 (StopIteration 예외에 해당)
        }
    }
}

fn main() {
    // 1부터 5까지 지연 평가(Lazy)로 생성하며 출력합니다.
    for num in Counter::new(5) {
        println!("{}", num);
    }
}
```

처음엔 "`yield` 한 줄이면 될 걸 구조체에 `impl`까지 써야 하나" 싶을 수 있습니다. 하지만 이 명시성이야말로 무비용을 가능케 하는 핵심입니다. 상태가 구조체 필드로 *드러나 있기에* 컴파일러는 프레임 객체 같은 불투명한 런타임 컨테이너 없이, 그저 두 개의 `u32` 필드를 레지스터에 올려두고 루프를 돌릴 수 있습니다.

### 파이썬 제너레이터 ↔ Rust 이터레이터 매핑

| Python 개념 | Rust 개념 | 핵심 차이 |
|---|---|---|
| `def __iter__(self)` | `impl IntoIterator` / `iter()` | Rust는 빌림/소유를 타입으로 구분 |
| `def __next__(self)` | `fn next(&mut self) -> Option<Item>` | 종료가 예외가 아닌 `None` |
| `raise StopIteration` | `return None` | 흐름 제어에 예외 비용 없음 |
| `yield value` | `Some(value)` 반환 + 상태는 구조체 필드 | `yield` 키워드 없음, 상태 명시 |
| 제너레이터 표현식 `(f(x) for x in xs)` | `xs.iter().map(f)` | 둘 다 lazy, Rust는 단일형화됨 |
| `[f(x) for x in xs]` (리스트 컴프리헨션) | `xs.iter().map(f).collect()` | Rust는 즉시 평가됨 |
| `itertools.islice(g, n)` | `iter.take(n)` | 동일한 지연 절단 |
| `map`/`filter` 내장 함수 | `.map()`/`.filter()` 어댑터 | Rust는 또 다른 이터레이터 반환 |
| `functools.reduce` | `.fold()` / `.reduce()` | Rust는 초기값 유무로 구분 |
| `enumerate` / `zip` | `.enumerate()` / `.zip()` | 거의 동일 |
| 무한 제너레이터 `while True: yield` | 무한 이터레이터 + `.take(n)` | 둘 다 안전 |

---

## 15.3 `iter()` / `iter_mut()` / `into_iter()` — 소유권이 결정하는 세 가지 순회

여기가 파이썬 개발자가 가장 자주 발이 걸리는 지점입니다. 파이썬에서 `for x in collection`은 늘 한 가지 의미였습니다 — 컬렉션에서 객체 참조를 하나씩 꺼낸다. 그러나 Rust에서는 Chapter 4~5의 소유권(Ownership)·대여(Borrowing) 모델이 순회에도 그대로 적용됩니다. 같은 `Vec`을 도는 데 세 가지 방법이 있고, 각각이 원소를 빌릴지, 가변으로 빌릴지, 소유권째 가져갈지를 결정합니다.

| 메서드 | 원소 타입 | 의미 | 순회 후 컬렉션 |
|---|---|---|---|
| `.iter()` | `&T` (불변 참조) | 원소를 빌려서 읽기만 | 그대로 사용 가능 |
| `.iter_mut()` | `&mut T` (가변 참조) | 원소를 빌려서 제자리 수정 | 그대로 사용 가능 |
| `.into_iter()` | `T` (소유 값) | 원소의 소유권을 가져감 | **소비됨(이동)** |

```rust
fn main() {
    // 1. iter() — 빌려서 읽기만 한다 (가장 흔함)
    let data = vec![1, 2, 3];
    let total: i32 = data.iter().sum();   // 원소 타입은 &i32
    println!("합계: {}, 원본 살아있음: {:?}", total, data);

    // 2. iter_mut() — 빌려서 제자리에서 고친다
    let mut data = vec![1, 2, 3];
    for x in data.iter_mut() {            // 원소 타입은 &mut i32
        *x *= 10;                         // 역참조로 원본을 수정
    }
    println!("제자리 수정: {:?}", data);   // [10, 20, 30]

    // 3. into_iter() — 소유권을 가져가서 소비한다
    let data = vec![String::from("a"), String::from("b")];
    let owned: Vec<String> = data.into_iter()
        .map(|s| s + "!")                 // 원소 타입은 String (소유), 그래서 이동 가능
        .collect();
    println!("소유 변환: {:?}", owned);
    // println!("{:?}", data);            // 🚨 컴파일 에러! data는 위에서 소비됨
}
```

언제 무엇을 쓸까요? 규칙은 명료합니다. **읽기만 하면 `iter()`, 제자리에서 고치면 `iter_mut()`, 원소를 다른 곳으로 옮기거나 변환해서 소유해야 하면 `into_iter()`.** `String`처럼 `Copy`가 아닌 타입을 `map` 안에서 소비하려면 `into_iter()`로 소유권을 받아와야 합니다. `&String`을 받았다면 빌린 값을 옮길 수 없어 컴파일이 막힙니다.

> 💡 **핵심**: `for x in &v`는 `v.iter()`의 설탕, `for x in &mut v`는 `v.iter_mut()`의 설탕, `for x in v`는 `v.into_iter()`의 설탕입니다. `&`가 보이면 빌리는 것, 없으면 소유권을 가져가는 것 — 파이썬에는 없던 이 구분이 순회 한 줄에도 박혀 있습니다.

### `for` 루프의 디슈가링: `IntoIterator` + `while let`

파이썬의 `for`가 내부적으로 `iter()`와 `next()`를 호출하듯, Rust의 `for`도 순수한 문법 설탕(Syntactic Sugar)입니다. `for x in collection { ... }`은 컴파일러가 다음으로 풀어냅니다.

```rust
fn main() {
    let v = vec![10, 20, 30];

    // 우리가 쓰는 설탕:
    for x in &v {
        print!("{} ", x);
    }
    println!();

    // 컴파일러가 디슈가하는 실제 코드:
    let mut it = (&v).into_iter();     // 1. IntoIterator::into_iter 호출
    while let Some(x) = it.next() {    // 2. None이 나올 때까지 next() 반복
        print!("{} ", x);
    }
    println!();
}
```

`for`가 받는 것은 `Iterator`가 아니라 `IntoIterator`입니다. `IntoIterator`는 "나는 이터레이터로 변환될 수 있다"를 뜻하는 트레이트로, `Vec<T>`(소유), `&Vec<T>`(불변 빌림), `&mut Vec<T>`(가변 빌림)에 대해 각각 다르게 구현되어 있습니다. 그래서 `for x in v`, `for x in &v`, `for x in &mut v`가 자연스럽게 위 세 가지 순회로 갈라지는 것입니다. 모든 `Iterator`는 자기 자신을 반환하는 `IntoIterator`이기도 하므로, `Counter::new(5)` 같은 직접 만든 이터레이터도 `for`에 바로 넣을 수 있습니다.

> **Under the Hood — `while let`은 패턴 매칭이다**
>
> `while let Some(x) = it.next()`는 Chapter 12에서 본 패턴 매칭(Pattern Matching)의 응용입니다. `next()`가 `Option<Item>`을 반환하므로, `Some(x)` 패턴이 매치되는 동안(=값이 있는 동안) 루프가 돌고, `None`이 나오는 순간(=끝났을 때) `Some(x)` 패턴 매칭이 실패해 루프를 빠져나갑니다. 파이썬은 `StopIteration` 예외를 `try/except`로 잡아 종료를 감지했지만, Rust는 enum 판별자(discriminant) 하나를 검사하는 단순 분기로 처리합니다. 예외 객체 생성도, 스택 되감기(unwinding)도 없습니다. 이것이 종료를 *값*으로 표현한 설계가 주는 보상입니다.

---

## 15.4 어댑터(Adapter) — lazy하게 이터레이터를 변형하기

어댑터는 이터레이터를 받아 *또 다른 이터레이터*를 반환하는 메서드입니다. 핵심 성질은 **lazy**라는 것입니다. `.map()`을 호출해도 아무 값도 계산되지 않습니다. 그저 "나중에 값이 흐를 때 이 변환을 적용하겠다"는 명세를 담은 새 이터레이터(`Map<...>` 타입)를 만들 뿐입니다. 실제 계산은 소비자(다음 절)가 값을 끌어당길 때 한 원소씩 일어납니다.

파이썬의 제너레이터 표현식과 똑같은 철학입니다. 다른 점은 Rust의 어댑터 체인은 컴파일 타임에 하나의 타입으로 합쳐져 단일형화된다는 것입니다.

왜 lazy가 이토록 중요할까요? 두 가지 이유가 있습니다. 첫째는 **메모리**입니다. 만약 어댑터가 즉시 평가(eager)된다면, `.filter()`는 통과한 원소 전부를 담은 중간 컬렉션을 만들고, `.map()`은 변환된 결과 전부를 담은 또 다른 중간 컬렉션을 만들어, 단계마다 새 배열이 힙에 쌓일 것입니다. 파이썬 2의 `map()`/`filter()`가 리스트를 반환하던 시절이 정확히 이랬고, 그래서 파이썬 3에서 이들이 제너레이터(lazy)로 바뀐 것입니다. lazy 체인은 중간 컬렉션을 단 하나도 만들지 않습니다. 원소 하나가 파이프라인 전체를 끝까지 통과한 뒤에야 다음 원소가 들어옵니다.

둘째는 **단축 평가(Short-circuiting)** 입니다. lazy하기 때문에 `(0..).map(expensive).find(|x| x.is_good())`처럼 무한 이터레이터에 비싼 변환을 걸어도, `find`가 첫 매치를 찾는 순간 멈춥니다. 그 뒤의 무한히 많은 원소에는 `expensive`가 단 한 번도 호출되지 않습니다. eager였다면 무한 루프에 빠지거나 메모리가 폭발했을 코드가, lazy이기에 정확히 필요한 만큼만 계산하고 끝납니다. 파이썬 제너레이터가 주던 바로 그 이점을, Rust는 추가 비용 없이 제공합니다.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // 짝수만 골라 제곱한 뒤 Vec으로 — 파이썬의 [x*x for x in nums if x%2==0]
    let squared_evens: Vec<i32> = numbers
        .into_iter()              // 이터레이터 생성 (소유권 이동)
        .filter(|x| x % 2 == 0)   // 어댑터 1: 짝수만 통과 (lazy)
        .map(|x| x * x)           // 어댑터 2: 제곱 (lazy)
        .collect();               // 소비자: 여기서 비로소 계산이 시작됨

    println!("{:?}", squared_evens); // [4, 16]
}
```

`|x| x % 2 == 0`은 클로저(Closure)입니다. 파이썬의 `lambda x: x % 2 == 0`과 거의 같고, 파이프(`|`) 사이에 매개변수를 둡니다. Chapter 16에서 스레드에 넘길 클로저를 깊이 다룹니다.

### 자주 쓰는 어댑터 사전

```rust
fn main() {
    // map / filter — 변환과 선별
    let v: Vec<i32> = (1..=5).map(|x| x * 2).collect();          // [2,4,6,8,10]
    let v: Vec<i32> = (1..=10).filter(|x| x % 3 == 0).collect(); // [3,6,9]

    // take / skip — 앞에서 n개 / n개 건너뛰기
    let v: Vec<i32> = (1..).take(3).collect();                   // [1,2,3] (무한에서 절단)
    let v: Vec<i32> = (1..=5).skip(2).collect();                 // [3,4,5]

    // take_while / skip_while — 조건이 유지되는 동안만
    let v: Vec<i32> = (1..100).take_while(|&x| x < 5).collect(); // [1,2,3,4]
    let v: Vec<i32> = (1..=6).skip_while(|&x| x < 4).collect();  // [4,5,6]

    // enumerate — 인덱스를 함께 (파이썬 enumerate와 동일)
    let v: Vec<(usize, char)> = ['a', 'b', 'c'].into_iter().enumerate().collect();
    println!("{:?}", v);                                          // [(0,'a'),(1,'b'),(2,'c')]

    // zip — 두 이터레이터를 짝지음 (짧은 쪽에서 멈춤)
    let v: Vec<(char, i32)> = ['a', 'b'].into_iter().zip(1..).collect();
    println!("{:?}", v);                                          // [('a',1),('b',2)]

    // chain — 두 이터레이터를 이어붙임 (itertools.chain)
    let v: Vec<i32> = (1..=3).chain(7..=9).collect();             // [1,2,3,7,8,9]
    println!("{:?}", v);

    // flat_map — 각 원소를 이터레이터로 펼친 뒤 평탄화
    let v: Vec<i32> = vec![vec![1, 2], vec![3, 4]]
        .into_iter()
        .flat_map(|inner| inner.into_iter())
        .collect();                                               // [1,2,3,4]
    println!("{:?}", v);

    // step_by / rev — 간격 두고 / 역순
    let v: Vec<i32> = (0..10).step_by(3).collect();               // [0,3,6,9]
    let v: Vec<i32> = (1..=3).rev().collect();                    // [3,2,1]
    println!("{:?}", v);

    // scan — fold처럼 누적하되, 각 단계의 중간 결과를 내놓음 (itertools.accumulate)
    let running: Vec<i32> = [1, 2, 3, 4]
        .iter()
        .scan(0, |acc, &x| {
            *acc += x;
            Some(*acc)        // None을 반환하면 거기서 이터레이터 종료
        })
        .collect();                                               // [1,3,6,10]
    println!("{:?}", running);

    // cloned / copied — &T 이터레이터를 T 이터레이터로 (참조를 값으로)
    let refs = vec![1, 2, 3];
    let owned: Vec<i32> = refs.iter().copied().collect();         // Copy 타입은 copied()
    println!("{:?}", owned);
}
```

`peekable`은 특별히 유용합니다. 다음 원소를 *소비하지 않고* 미리 엿볼 수 있게 해줘서, 인접한 같은 값을 묶는 런-렝스(run-length) 같은 작업에 쓰입니다(itertools의 `groupby`와 유사).

```rust
fn main() {
    let data = [1, 1, 2, 3, 3, 3];
    let mut it = data.iter().peekable();
    let mut groups = vec![];

    while let Some(&x) = it.next() {
        let mut count = 1;
        while it.peek() == Some(&&x) {  // 소비하지 않고 다음 값을 엿봄
            it.next();                  // 같으면 그때 소비
            count += 1;
        }
        groups.push((x, count));
    }
    println!("{:?}", groups);           // [(1, 2), (2, 1), (3, 3)]
}
```

> **Under the Hood — 어댑터는 중첩된 구조체다**
>
> `(1..=5).filter(pred).map(f)`의 타입은 무엇일까요? `std::iter::Map<std::iter::Filter<std::ops::RangeInclusive<i32>, F>, G>`입니다. 각 어댑터는 *이전 이터레이터를 필드로 품은 구조체*입니다. `map`은 내부에 `filter` 이터레이터를, `filter`는 내부에 `Range`를 갖습니다. 양파처럼 겹친 이 구조에 *데이터 복사본은 없습니다* — 클로저와 이전 이터레이터에 대한 핸들만 있습니다.
>
> 소비자가 `next()`를 호출하면, 호출이 가장 바깥 `Map`에서 안쪽 `Filter`로, 다시 `Range`로 전달(delegate)되며 한 원소를 끌어옵니다. `Range`가 값을 내놓으면 `Filter`가 술어를 검사하고, 통과하면 `Map`이 변환을 적용해 호출자에게 돌려줍니다. **중요한 건 이 모든 메서드가 `#[inline]` 표시되어 있다는 것**입니다. LLVM은 이 중첩 호출을 전부 인라이닝해 하나의 평평한 루프로 녹여버립니다. 그래서 양파 껍질 같은 추상화 계층이 어셈블리에서는 흔적도 없이 사라집니다. 다음 절에서 이걸 눈으로 확인합니다.

---

## 15.5 소비자(Consumer) — 파이프라인을 실제로 돌리는 것

어댑터가 lazy한 명세였다면, 소비자는 이터레이터를 끝까지 돌려 *값을 끌어내는* 메서드입니다. `next()`를 반복 호출해 이터레이터를 소진하고, 단일 값이나 컬렉션을 반환합니다. 소비자가 호출되기 전까지 어댑터 체인은 한 줄도 실행되지 않는다는 점을 다시 강조합니다.

```rust
use std::collections::HashMap;

fn main() {
    // collect — 가장 강력한 소비자. 무엇으로 모을지는 타입이 결정한다
    let v: Vec<i32> = (1..=5).collect();
    let s: String = ['R', 'u', 's', 't'].iter().collect();      // 문자 → 문자열
    let m: HashMap<i32, i32> = (1..=3).map(|x| (x, x * x)).collect(); // 튜플 → 맵
    println!("{:?} / {} / {:?}", v, s, m);

    // 수치 집계
    let sum: i32 = (1..=100).sum();           // 5050
    let product: u64 = (1..=10u64).product(); // 3628800
    let count = (1..=50).filter(|x| x % 7 == 0).count(); // 7

    // fold — 초기값 + 누적 함수 (파이썬 functools.reduce(f, xs, init))
    let concatenated = ["a", "b", "c"]
        .iter()
        .fold(String::new(), |mut acc, &s| {
            acc.push_str(s);
            acc
        });                                   // "abc"

    // reduce — 초기값 없이 첫 원소를 시드로 (Option 반환, 빈 경우 None)
    let max_via_reduce = [3, 7, 2, 9, 4].iter().copied().reduce(|a, b| a.max(b)); // Some(9)

    // min / max — 비교 가능한 원소의 극값 (Option 반환)
    let smallest = [3, 7, 2, 9].iter().min();  // Some(&2)

    // any / all / find / position — 단축 평가(short-circuit)
    let has_even = (1..=5).any(|x| x % 2 == 0);          // true (첫 짝수에서 즉시 멈춤)
    let all_positive = (1..=5).all(|x| x > 0);           // true
    let first_big = (1..=100).find(|&x| x > 42);         // Some(43)
    let idx = ['a', 'b', 'c'].iter().position(|&c| c == 'b'); // Some(1)

    println!("{} {} {} {:?} {:?} {:?} {} {} {:?} {:?}",
        sum, product, count, max_via_reduce, smallest,
        Some(concatenated), has_even, all_positive, first_big, idx);

    // for_each — 부수 효과만 (반환값 없음). 사실상 for 루프
    (1..=3).for_each(|x| print!("{} ", x));
    println!();

    // partition — 술어로 두 컬렉션으로 가른다
    let (evens, odds): (Vec<i32>, Vec<i32>) = (1..=6).partition(|x| x % 2 == 0);
    println!("짝수 {:?}, 홀수 {:?}", evens, odds);
}
```

`any`, `all`, `find`, `position`은 **단축 평가**합니다. `any`는 첫 참을 만나면, `find`는 첫 매치를 만나면 거기서 즉시 멈추고 나머지 원소는 건드리지 않습니다. 무한 이터레이터에 `find`를 걸어도 매치만 있으면 종료하는 이유입니다.

### `collect`의 다형성: 같은 코드, 다른 결과 타입

`collect`는 특별합니다. *무엇으로* 수집할지를 코드가 아니라 **타입 어노테이션**이 결정합니다. 이것은 `FromIterator` 트레이트와 Chapter 3에서 본 역방향 타입 추론의 합작입니다.

```rust
use std::collections::{HashMap, HashSet, BTreeMap};

fn main() {
    let pairs = || (1..=3).map(|x| (x, x * x));

    // 같은 이터레이터를, 타입만 바꿔 전혀 다른 컬렉션으로 수집
    let as_vec: Vec<(i32, i32)> = pairs().collect();
    let as_map: HashMap<i32, i32> = pairs().collect();
    let as_btree: BTreeMap<i32, i32> = pairs().collect();
    let as_set: HashSet<i32> = (1..=3).chain(1..=3).collect(); // 중복 제거 → {1,2,3}

    // turbofish 문법 — 어노테이션 대신 메서드에 직접 타입 지정
    let doubled = (1..=3).map(|x| x * 2).collect::<Vec<i32>>();

    println!("{:?}\n{:?}\n{:?}\n{:?}\n{:?}",
        as_vec, as_map, as_btree, as_set, doubled);
}
```

`collect::<Vec<i32>>()`의 `::<...>` 문법은 그 생김새 때문에 **터보피시(turbofish)** 라고 불립니다. 타입 어노테이션을 변수에 달기 애매할 때(예: 체인 도중에 타입을 확정해야 할 때) 메서드 자체에 타입을 못 박는 용도입니다.

더 강력한 건 `Result`를 수집하는 패턴입니다. `Iterator<Item = Result<T, E>>`를 `Result<Vec<T>, E>`로 모으면, **첫 에러에서 단축 평가하며 전체를 실패시킵니다**.

```rust
fn main() {
    let inputs = ["1", "2", "x", "4"];

    // 하나라도 파싱 실패하면 전체가 Err — 파이썬에서 일일이 try/except 하던 패턴
    let parsed: Result<Vec<i32>, _> = inputs.iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", parsed); // Err(ParseIntError { .. }) — "x"에서 실패

    let good = ["1", "2", "3"];
    let parsed: Result<Vec<i32>, _> = good.iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", parsed); // Ok([1, 2, 3])
}
```

> **Under the Hood — zero-cost를 어셈블리로 증명한다**
>
> "이터레이터 체인이 손으로 짠 루프와 같은 어셈블리가 된다"는 주장을 검증해봅시다. 다음 두 함수를 `rustc -O`(릴리스 최적화)로 컴파일하고 아키텍처(aarch64) 어셈블리를 추출했습니다.
>
> ```rust
> #[inline(never)]
> pub fn sum_loop(v: &[i64]) -> i64 {
>     let mut total = 0;
>     for i in 0..v.len() {
>         total += v[i] * v[i];   // 인덱스 접근 — 경계 검사 대상
>     }
>     total
> }
>
> #[inline(never)]
> pub fn sum_iter(v: &[i64]) -> i64 {
>     v.iter().map(|x| x * x).sum()  // 선언적 체인
> }
> ```
>
> 두 함수의 핵심 루프 어셈블리는 **글자 그대로 동일**했습니다(`madd`는 곱셈-누적 단일 명령):
>
> ```text
> LBB_loop:                       ; 4-way 언롤된 핫 루프 (양쪽 함수 동일)
>     ldp  x15, x16, [x13, #-16]  ; 한 번에 두 원소 로드
>     ldp  x17, x2,  [x13], #32   ; 다시 두 원소 로드 (포인터 +32)
>     madd x8,  x15, x15, x8      ; total += x15 * x15
>     madd x10, x16, x16, x10     ; 누적기 4개로 나눠 의존성 끊기
>     madd x11, x17, x17, x11
>     madd x12, x2,  x2,  x12
>     subs x14, x14, #4           ; 카운터 -4
>     b.ne LBB_loop               ; 분기
> ```
>
> 주목할 점이 셋입니다. 첫째, **루프 안에 경계 검사(bounds check) 분기가 없습니다.** `sum_loop`의 `v[i]`는 인덱스 접근이라 원래 매 반복마다 "`i < v.len()`인가?" 검사가 들어가야 하지만, LLVM이 `0..v.len()` 범위에서 `i`가 절대 길이를 넘지 않음을 증명해 검사를 통째로 제거했습니다. `sum_iter`는 `iter()`가 애초에 경계 안에서만 도므로 검사가 존재할 이유조차 없습니다. 둘째, **컴파일러가 루프를 4겹으로 언롤(unroll)** 하고 누적기를 4개로 쪼개 CPU 파이프라인 의존성을 끊었습니다 — 손으로 짜기엔 번거로운 최적화입니다. 셋째, `map` 클로저와 `sum` 소비자, `Map` 구조체의 중첩은 **흔적도 없이 사라졌습니다.** 함수 호출 한 번 없습니다.
>
> 즉 `v.iter().map(|x| x * x).sum()`은 추상화 계층이 셋이나 되지만, 컴파일 후엔 손으로 짠 인덱스 루프와 *동일한 기계어*입니다. 파이썬의 `sum(x*x for x in v)`가 원소마다 프레임 활성화·바이트코드 디스패치·28바이트 정수 객체 할당을 치르는 것과 정반대입니다. 이것이 무비용 추상화의 실증입니다 — "비용이 적다"가 아니라 "비용이 *0*이다".

> **Under the Hood — 어댑터 체인의 바이트 크기: 클로저가 ZST면 0바이트**
>
> 앞 박스에서 어댑터 체인이 "중첩된 구조체"라고 했습니다. 그렇다면 그 구조체는 정확히 *몇 바이트*일까요? `std::mem::size_of_val`로 직접 재보면 무비용이 메모리 차원에서도 사실임이 드러납니다. `Vec<i64>`에 대해(`rustc -O`, 1.90.0 측정):
>
> ```rust
> use std::mem::size_of_val;
> let v = vec![1i64, 2, 3, 4, 5];
>
> let it0 = v.iter();
> // slice::Iter<i64> = 두 포인터(시작, 끝) = 16바이트
> assert_eq!(size_of_val(&it0), 16);
>
> let it1 = v.iter().filter(|&&x| x % 2 == 0);
> // Filter = 내부 Iter + 클로저. 그런데 캡처 없는 클로저는 ZST(0바이트)!
> assert_eq!(size_of_val(&it1), 16);   // 여전히 16 — filter가 공짜로 얹힘
>
> let it2 = v.iter().filter(|&&x| x % 2 == 0).map(|&x| x * x);
> // map도 ZST 클로저 → 또 0바이트 추가
> assert_eq!(size_of_val(&it2), 16);   // 어댑터 둘을 얹어도 16 그대로
> ```
>
> 핵심은 **캡처(capture)가 없는 클로저는 ZST(Zero-Sized Type), 즉 크기가 0바이트**라는 것입니다. `size_of_val(&|x: i64| x * x)`는 정확히 `0`을 반환합니다. 클로저는 "캡처한 환경을 담는 익명 구조체"인데, 캡처할 것이 없으면 그 구조체는 필드가 0개라 0바이트가 됩니다(Chapter 7에서 본 단위 구조체와 같은 원리). 그래서 `filter`/`map`을 아무리 길게 이어 붙여도, 클로저가 전부 ZST라면 이터레이터 구조체의 크기는 *전혀 늘지 않습니다*. 양파 껍질은 타입(컴파일 타임)에만 존재하고 런타임 메모리에는 흔적이 없습니다.
>
> 반대로 환경을 캡처하면 그만큼 자랍니다:
>
> ```rust
> let threshold: i64 = 3;
> let it = v.iter().filter(move |&&x| x > threshold);
> // 클로저가 i64 하나(8바이트)를 캡처 → Filter = 16 + 8 = 24
> assert_eq!(size_of_val(&it), 24);
> // enumerate처럼 자체 상태(usize 카운터)를 가진 어댑터도 8바이트 더함
> let it_e = v.iter().map(|&x| x * x).enumerate();
> assert_eq!(size_of_val(&it_e), 24);   // 16 + usize(8)
> ```
>
> 즉 이터레이터 어댑터의 메모리 비용은 "캡처한 데이터 + 어댑터 자체 상태"의 합이며, 그 외에는 *공짜*입니다. 파이썬 제너레이터 표현식이 매번 프레임 객체(수백 바이트)를 들고 다니는 것과 비교하면, Rust 어댑터 체인은 보통 레지스터 몇 개에 들어가는 십수 바이트입니다.

> **Under the Hood — `impl Trait` 반환 vs `Box<dyn Iterator>`: 정적 디스패치와 동적 디스패치**
>
> 함수에서 이터레이터를 *반환*할 때, 그 어댑터 체인의 타입 이름은 `Map<Filter<Iter<'_, i64>, ...>, ...>`처럼 끔찍하게 깁니다. 캡처가 들어가면 클로저 타입이 익명이라 아예 적을 수조차 없습니다. 그래서 반환 타입을 숨기는 두 가지 방법이 있고, 디스패치 비용이 정반대입니다.
>
> ```rust
> // 정적 디스패치: 구체 타입을 컴파일러가 추론, 그대로 단일형화
> fn evens_static(v: &[i64]) -> impl Iterator<Item = i64> + '_ {
>     v.iter().filter(|&&x| x % 2 == 0).map(|&x| x * x)
> }
>
> // 동적 디스패치: 트레이트 객체로 박싱
> fn evens_dynamic(v: &[i64]) -> Box<dyn Iterator<Item = i64> + '_> {
>     Box::new(v.iter().filter(|&&x| x % 2 == 0).map(|&x| x * x))
> }
> ```
>
> `impl Trait`는 "구체 타입은 컴파일러가 알지만 이름은 숨긴다"는 뜻입니다. 호출부는 진짜 `Map<Filter<...>>`를 받고, `next()` 호출은 전부 인라이닝됩니다 — **정적 디스패치**, 무비용입니다. 스택 위 핸들 크기는 그 구체 타입 그대로(여기선 16바이트). 힙 할당 0.
>
> `Box<dyn Iterator>`는 **트레이트 객체(Trait Object)** 입니다. 이터레이터 구조체를 힙에 한 번 할당하고, 스택에는 **팻 포인터(fat pointer)** — 데이터 포인터 + vtable 포인터, `size_of::<Box<dyn Iterator<Item=i32>>>()`로 재면 정확히 16바이트(8+8) — 만 남깁니다. 매 `next()`는 vtable을 거치는 간접 호출이라 인라이닝이 막히고, 힙 할당이 한 번 듭니다. **동적 디스패치**입니다.
>
> | 항목 | `impl Iterator` | `Box<dyn Iterator>` |
> |---|---|---|
> | 디스패치 | 정적(인라이닝됨) | 동적(vtable 간접 호출) |
> | 힙 할당 | 없음 | 1회(이터레이터 박싱) |
> | 스택 핸들 | 구체 타입 크기(예: 16) | 팻 포인터 16(데이터+vtable) |
> | 분기마다 다른 타입 반환 | 불가(15.x 에러 참고) | 가능 |
> | 트레이트 메서드에서 반환 | 제약 있음(연관 타입) | 자유로움 |
>
> 규칙은 단순합니다. **거의 항상 `impl Trait`를 쓰세요** — 무비용이니까요. `Box<dyn>`은 (1) `if`/`else` 분기마다 *다른* 어댑터 타입을 반환해야 하거나(다음 박스의 컴파일 에러), (2) `Vec<Box<dyn Iterator>>`처럼 이질적 이터레이터를 한 컨테이너에 담아야 할 때만 꺼내세요. 동적 디스패치의 비용은 핫 루프에서 인라이닝이 막히는 것이지, "느리다"는 막연한 공포가 아닙니다.
>
> 그런데 `impl Trait`로 분기마다 다른 타입을 반환하려 들면 컴파일러가 정확히 막습니다.
>
> ```rust
> fn make_iter(reverse: bool) -> impl Iterator<Item = i32> {
>     if reverse {
>         (1..=5).rev()  // 타입: Rev<RangeInclusive<i32>>
>     } else {
>         1..=5          // 타입: RangeInclusive<i32>  ← 다른 타입!
>     }
> }
> ```
>
> ```text
> error[E0308]: `if` and `else` have incompatible types
>  --> src/main.rs:6:9
>   |
> 4 |         (1..=5).rev()
>   |         ------------- expected because of this
> 6 |         1..=5
>   |         ^^^^^ expected `Rev<RangeInclusive<{integer}>>`,
>   |               found `RangeInclusive<{integer}>`
>   |
> help: you could change the return type to be a boxed trait object
>   |
> 2 - fn make_iter(reverse: bool) -> impl Iterator<Item = i32> {
> 2 + fn make_iter(reverse: bool) -> Box<dyn Iterator<Item = i32>> {
> ```
>
> 컴파일러가 *직접* "`Box<dyn>`으로 바꾸고 각 분기를 `Box::new(...)`로 감싸라"고 해법까지 제시합니다. 이것이 두 방식의 경계선을 가장 선명하게 보여주는 지점입니다 — 단일 구체 타입이면 `impl Trait`(무비용), 여러 타입을 하나로 합쳐야 하면 `Box<dyn>`(동적 디스패치).

---

## 15.6 커스텀 `Iterator` 구현 — 피보나치와 슬라이딩 윈도우

`next()` 하나만 구현하면 60여 개의 어댑터·소비자가 공짜로 따라온다는 약속을 직접 확인해봅시다. 파이썬에서 `yield`로 짰을 무한 피보나치를, Rust에서는 구조체 + `next()`로 구현합니다.

```rust
struct Fibonacci {
    curr: u64,
    next: u64,
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<u64> {
        let new_next = self.curr + self.next; // 다음 값 계산
        self.curr = self.next;                // 상태 전진
        self.next = new_next;
        Some(self.curr)                        // 무한히 None을 내지 않음 → 무한 이터레이터
    }
}

fn fibonacci() -> Fibonacci {
    Fibonacci { curr: 0, next: 1 }
}

fn main() {
    // 무한 이터레이터지만 take(10)으로 안전하게 절단 (itertools.islice와 동일)
    let first_ten: Vec<u64> = fibonacci().take(10).collect();
    println!("{:?}", first_ten); // [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]

    // next()만 구현했는데 어댑터·소비자가 전부 동작한다
    let big_fib = fibonacci().find(|&x| x > 1000);          // Some(1597)
    let sum_of_first_ten: u64 = fibonacci().take(10).sum(); // 143
    println!("{:?}, 합: {}", big_fib, sum_of_first_ten);
}
```

`next()`가 영원히 `None`을 내지 않으므로 `Fibonacci`는 무한 이터레이터입니다. 파이썬의 `while True: yield`와 똑같이, `take`나 `find` 같은 단축 소비자로 끌어당기는 만큼만 계산됩니다.

두 번째 예제는 슬라이딩 윈도우입니다. 시계열·로그 분석에서 "연속한 N개씩 묶어 보는" 작업은 흔합니다. 제네릭으로 어떤 타입에도 쓸 수 있게 만들어봅시다.

```rust
struct Windows<T: Clone> {
    data: Vec<T>,
    size: usize,
    pos: usize,
}

impl<T: Clone> Iterator for Windows<T> {
    type Item = Vec<T>;

    fn next(&mut self) -> Option<Vec<T>> {
        if self.pos + self.size <= self.data.len() {
            let window = self.data[self.pos..self.pos + self.size].to_vec();
            self.pos += 1;
            Some(window)
        } else {
            None // 윈도우가 데이터 끝을 넘으면 종료
        }
    }
}

fn windows<T: Clone>(data: Vec<T>, size: usize) -> Windows<T> {
    Windows { data, size, pos: 0 }
}

fn main() {
    let result: Vec<Vec<i32>> = windows(vec![1, 2, 3, 4], 2).collect();
    println!("{:?}", result); // [[1, 2], [2, 3], [3, 4]]

    // 3일 이동평균 — 커스텀 이터레이터 + 표준 어댑터를 조합
    let prices = vec![10.0, 12.0, 11.0, 13.0, 15.0];
    let moving_avg: Vec<f64> = windows(prices, 3)
        .map(|w| w.iter().sum::<f64>() / w.len() as f64)
        .collect();
    println!("{:?}", moving_avg); // [11.0, 12.0, 13.0]
}
```

참고로 `Vec`이 이미 가진 슬라이스(slice) 메서드 `windows`와 `chunks`가 있다면 직접 구현할 필요가 없습니다. 다만 이들은 데이터를 빌려 `&[T]`를 내놓고(복사 없음), 위 커스텀 버전은 `to_vec()`으로 소유 복사본을 만든다는 차이가 있습니다.

```rust
fn main() {
    let v = [1, 2, 3, 4, 5];

    // windows(n): 겹치는 슬라이딩 윈도우 (&[T] 반환, 복사 없음)
    let w: Vec<&[i32]> = v.windows(2).collect();
    println!("{:?}", w); // [[1, 2], [2, 3], [3, 4], [4, 5]]

    // chunks(n): 겹치지 않는 묶음 (마지막은 짧을 수 있음)
    let c: Vec<&[i32]> = v.chunks(2).collect();
    println!("{:?}", c); // [[1, 2], [3, 4], [5]]
}
```

> **Under the Hood — `size_hint`와 메모리 사전 할당**
>
> `Iterator` 트레이트에는 구현하지 않았지만 기본값이 제공되는 `size_hint(&self) -> (usize, Option<usize>)` 메서드가 있습니다. 이터레이터가 "최소 몇 개, 최대 몇 개를 내놓을지"를 컴파일러가 아니라 *런타임에* 힌트로 알려주는 장치입니다.
>
> 왜 중요할까요? `collect::<Vec<_>>()`가 호출되면, `Vec`은 `size_hint`의 하한값만큼 **미리 한 번에 메모리를 할당**합니다. 그래서 `(1..1000).collect::<Vec<_>>()`는 1000개 용량을 단번에 잡고 재할당(reallocation) 없이 채웁니다. 반면 우리가 만든 `Fibonacci`는 무한이라 `size_hint`가 `(usize::MAX, None)`을 반환하고, 커스텀 `Windows`는 기본값 `(0, None)`을 씁니다 — 이 경우 `Vec`은 채우면서 점진적으로 키웁니다(2배씩 재할당). 정확한 개수를 아는 이터레이터(`Range`, `Vec::iter` 등)에 `size_hint`를 구현해두면 단 한 번의 할당으로 끝나, 파이썬 리스트가 `append`마다 겪을 수 있는 분할 상환 재할당 비용을 컴파일러 차원에서 회피합니다. 직접 만든 이터레이터에서도 개수를 안다면 `size_hint`를 구현하는 것이 성능상 이득입니다.

---

## 15.7 실전: 대용량 로그 파이프라인 — `itertools` vs Rust Iterator

이제 실무 시나리오로 모든 것을 엮어봅시다. 수 기가바이트짜리 웹 서버 액세스 로그에서 **5xx 서버 에러만 골라 경로별로 집계**하는 작업입니다. 파이썬 데이터 엔지니어가 매일 하는 그 일입니다.

### The Python Way — 제너레이터 + itertools

```python
from __future__ import annotations
from collections import Counter
from itertools import islice
from typing import Iterator

def parse_errors(path: str) -> Iterator[str]:
    """5xx 에러의 경로만 lazy하게 흘려보낸다."""
    with open(path) as f:
        for line in f:                       # 파일을 한 줄씩 (메모리 상수)
            parts = line.split()
            if len(parts) < 3:
                continue
            status, method, url = parts[0], parts[1], parts[2]
            try:
                if int(status) >= 500:
                    yield url
            except ValueError:
                continue

counts = Counter(parse_errors("access.log"))
for url, n in counts.most_common(10):
    print(f"{n:>6}  {url}")
```

깔끔하고, 파일 크기와 무관하게 메모리는 상수입니다. 하지만 라인마다 `split()`이 리스트 객체를 새로 만들고, `int()`가 정수 객체를 할당하고, 제너레이터 프레임이 매번 재활성화됩니다. 1억 줄이면 이 오버헤드가 누적됩니다.

### The Rust Implementation — 동일한 파이프라인, 무비용

```rust
use std::collections::HashMap;
use std::fs::File;
use std::io::{BufRead, BufReader};

fn main() -> std::io::Result<()> {
    let file = File::open("access.log")?;       // ? 연산자 — Chapter 13
    let reader = BufReader::new(file);          // 버퍼링된 읽기 (메모리 상수)

    let mut counts: HashMap<String, u32> = HashMap::new();

    reader
        .lines()                                // Iterator<Item = Result<String, _>> — lazy
        .filter_map(Result::ok)                 // 읽기 에러 줄은 건너뜀
        .filter_map(|line| {                    // 파싱 + 5xx 필터를 한 번에
            let mut parts = line.split_whitespace();
            let status = parts.next()?;          // ? on Option — 부족하면 None → 건너뜀
            let _method = parts.next()?;
            let url = parts.next()?;
            let code: u16 = status.parse().ok()?;
            (code >= 500).then(|| url.to_string()) // 조건 참일 때만 Some
        })
        .for_each(|url| *counts.entry(url).or_insert(0) += 1); // 집계

    // 상위 10개 — Vec으로 모아 정렬
    let mut top: Vec<(String, u32)> = counts.into_iter().collect();
    top.sort_by(|a, b| b.1.cmp(&a.1));          // 빈도 내림차순
    for (url, n) in top.into_iter().take(10) {
        println!("{:>6}  {}", n, url);
    }
    Ok(())
}
```

문자열로 동작을 검증해보면(파일 없이 컴파일·실행 가능한 축약판):

```rust
use std::collections::HashMap;

fn main() {
    let logs = "\
200 GET /api/users
404 GET /favicon.ico
500 POST /api/orders
200 GET /api/users
503 GET /health";

    let mut counts: HashMap<&str, u32> = HashMap::new();
    logs.lines()
        .filter_map(|line| {
            let mut parts = line.split_whitespace();
            let status = parts.next()?;
            let _method = parts.next()?;
            let path = parts.next()?;
            let code: u16 = status.parse().ok()?;
            (code >= 500).then_some(path)
        })
        .for_each(|path| *counts.entry(path).or_insert(0) += 1);

    let mut v: Vec<_> = counts.into_iter().collect();
    v.sort();
    println!("{:?}", v); // [("/api/orders", 1), ("/health", 1)]
}
```

구조가 파이썬과 거의 1:1입니다. `for line in f` → `reader.lines()`, 제너레이터의 조기 `continue` → `filter_map`의 `?`로 `None` 반환, `Counter` → `HashMap` + `entry().or_insert()`. `filter_map`의 클로저 안에서 `?` 연산자가 `Option`에 작동해, 형식이 어긋난 줄을 자연스럽게 걸러냅니다(Chapter 12~13의 `Option`/`?`가 이터레이터 안에서 다시 빛납니다).

차이는 비용입니다. 위 체인 전체가 단일형화·인라이닝되어, 줄마다 새 객체를 힙에 할당하지 않고(`split_whitespace`는 원본 버퍼를 빌리는 `&str` 슬라이스를 내놓습니다), 프레임 재활성화도 없습니다.

### 현실적 벤치마크

> 측정 맥락: 1,000만 줄의 합성 액세스 로그(약 350MB), `cargo build --release`, Apple M-시리즈 단일 코어, 동일 알고리즘(5xx 필터 + 경로 집계). 절대 수치는 환경마다 다르므로 *배수*로 읽으세요.

| 구현 | 대략 처리 시간 | 메모리 | 비고 |
|---|---|---|---|
| Python (제너레이터 + `Counter`) | ~9.5초 | 상수(~10MB) | 라인당 `split`/`int` 객체 할당 |
| Python (PyPy JIT) | ~3.2초 | 상수 | JIT가 일부 오버헤드 제거 |
| Rust (`Iterator` 체인) | ~0.5초 | 상수(~8MB) | `&str` 슬라이스, 할당 없음 |

CPython 대비 약 **15~20배**, PyPy 대비 약 **6배**입니다. Chapter 1에서 약속한 "2~50배"의 현실적 범위 안에 있습니다. 마법이 아니라 산수입니다 — 줄마다 치르던 인터프리터 디스패치와 객체 할당이 0이 되었을 뿐입니다. 메모리는 양쪽 다 상수입니다(둘 다 지연 스트리밍이므로). Rust가 메모리까지 적게 쓰는 이유는 28바이트 정수 객체와 포인터 배열 대신, 빌린 슬라이스와 4바이트 카운터를 쓰기 때문입니다.

이 배수를 어떻게 해석해야 할까요? 핵심은 *어디서* 시간이 사라졌는지입니다. 파이썬판에서 1억 줄을 처리하면, 줄마다 `line.split()`이 리스트 객체와 그 안의 문자열 슬라이스 객체들을 새로 할당하고, `int(status)`가 정수 객체를 할당하며, 제너레이터 프레임이 재활성화됩니다. 이 작업들 각각은 수십~수백 나노초지만, 1억 번 곱해지면 수 초가 됩니다. 즉 파이썬의 시간 대부분은 *우리가 짠 알고리즘*이 아니라 *인터프리터의 부기(bookkeeping)* 에 들어갑니다. Rust판은 이 부기가 전부 사라집니다. `split_whitespace()`는 원본 버퍼를 가리키는 `&str` 슬라이스(포인터 + 길이, 16바이트, 힙 할당 없음)만 내놓고, `parse::<u16>()`은 스택 위 2바이트 정수를 만들 뿐입니다. 남는 시간은 거의 순수하게 우리 알고리즘 — 비교와 해시맵 갱신 — 에만 쓰입니다.

그래서 이 15~20배는 "Rust가 마술적으로 빠르다"가 아니라, "파이썬이 추상화의 대가로 줄마다 내던 세금이 그만큼이었다"로 읽어야 정확합니다. 데이터가 작으면(수천 줄) 이 세금은 무시할 만하고 파이썬의 개발 속도가 이깁니다. 하지만 입력이 수천만 줄 이상으로 커지는 ETL·로그 분석·배치 처리 구간에서는, 이 세금이 운영 비용(CPU 시간, 클라우드 청구서)과 직결됩니다. Chapter 22에서 보게 될 "파이썬의 병목 구간만 Rust로 갈아끼우는" 전략이 설득력을 갖는 이유가 바로 이 지점입니다.

### 실무 크레이트: `itertools`와 `rayon`

표준 라이브러리 이터레이터로 부족할 때, [`itertools`](https://docs.rs/itertools) 크레이트가 파이썬 `itertools`의 거의 모든 것을 제공합니다 — `chunk_by`, `unique`, `cartesian_product`, `sorted`, `dedup`, `tuple_windows` 등. (인접 그룹핑 어댑터는 0.13에서 `group_by`가 `chunk_by`로 이름이 바뀌었습니다. 표준 라이브러리에 `slice::group_by`가 안정화되면서 이름 충돌을 피하기 위한 변경이니, 옛 자료의 `group_by`는 `chunk_by`로 읽으세요.)

```rust
// Cargo.toml: itertools = "0.13"
use itertools::Itertools;

fn main() {
    let words = ["apple", "avocado", "banana", "blueberry", "cherry"];

    // 첫 글자로 그룹핑 (파이썬 itertools.groupby — 단, 정렬 선행 필요)
    for (key, group) in &words.iter().chunk_by(|w| w.chars().next().unwrap()) {
        let items: Vec<&&str> = group.collect();
        println!("{}: {:?}", key, items);
    }

    // 중복 제거 + 정렬을 체인으로
    let v = vec![3, 1, 2, 3, 1];
    let unique_sorted: Vec<i32> = v.into_iter().unique().sorted().collect();
    println!("{:?}", unique_sorted); // [1, 2, 3]
}
```

파이썬 `itertools`/내장 함수와 Rust(표준 + `itertools`)의 대응을 한눈에 정리하면 다음과 같습니다. 대부분 이름까지 거의 그대로라, 파이썬 데이터 파이프라인을 옮길 때 이 표가 사전(辭典) 역할을 합니다.

| Python | Rust (표준/`itertools`) | 비고 |
|---|---|---|
| `map(f, xs)` | `xs.iter().map(f)` | 둘 다 lazy. Rust는 또 다른 이터레이터 반환 |
| `filter(p, xs)` | `xs.iter().filter(p)` | Rust 클로저는 `&Item`을 받음(이중 참조 주의, 15.9) |
| `zip(a, b)` | `a.iter().zip(b)` | 짧은 쪽에서 멈춤(동일) |
| `itertools.chain(a, b)` | `a.iter().chain(b)` | 이어붙이기(동일) |
| `zip(a, b, c)` (3개+) | `itertools::izip!(a, b, c)` | 표준 `zip`은 2개만, `izip!`은 가변 개수 |
| `itertools.groupby(xs, key)` | `xs.iter().chunk_by(key)` | **0.13에서 `group_by`→`chunk_by` rename.** 둘 다 인접 그룹만, 정렬 선행 필요 |
| `set(xs)` / `dict.fromkeys` | `xs.into_iter().unique()` | `unique`는 순서 보존 + lazy(첫 등장만) |
| `sorted(xs)` | `xs.into_iter().sorted()` | `itertools::sorted`, 내부적으로 `Vec` 수집 후 정렬 |
| `itertools.accumulate(xs)` | `xs.iter().scan(...)` | 표준 `scan`이 가장 가까움(중간 누적 방출) |
| `itertools.product(a, b)` | `a.iter().cartesian_product(b)` | 데카르트 곱 |
| `itertools.takewhile/dropwhile` | `.take_while(p)` / `.skip_while(p)` | 표준에 내장 |
| `more_itertools.windowed(xs, n)` | `xs.windows(n)` / `itertools::tuple_windows` | 슬라이스 `windows`는 복사 없음(`&[T]`) |

그리고 진짜 한 방은 [`rayon`](https://docs.rs/rayon)입니다. 위 로그 파이프라인의 `.iter()`를 `.par_iter()`로 **한 글자만 바꾸면** 모든 CPU 코어에 자동 분산됩니다. 파이썬이 GIL 때문에 `multiprocessing`으로 프로세스를 통째로 복제해야 했던 그 병렬화가, Rust에서는 이터레이터 메서드 교체 한 번입니다. 이것이 Chapter 16~18에서 다룰 "두려움 없는 동시성(Fearless Concurrency)"의 예고편입니다.

```rust
// use rayon::prelude::*;
// let sum: i64 = data.par_iter().map(|x| expensive(x)).sum();  // 전 코어 자동 분산
```

---

## 15.8 Best Practices & 안티패턴

**이렇게 하라**

- **읽기엔 `iter()`, 수정엔 `iter_mut()`, 변환·이동엔 `into_iter()`** 를 정확히 구분하세요. 불필요하게 `into_iter()`로 소유권을 가져가 원본을 소비시키지 마세요.
- **체인은 마지막에 한 번만 소비하세요.** 중간 결과를 `collect()`해 다시 `iter()`하는 건 임시 `Vec`을 힙에 만드는 낭비입니다(다음 절 참고).
- **`Copy` 타입 참조는 `copied()`, 비-`Copy`는 `cloned()`** 로 명시적으로 값화하세요. `map(|x| *x)`보다 의도가 명확합니다.
- **`Result`/`Option` 이터레이터는 `collect::<Result<Vec<_>, _>>()`** 로 모아 첫 에러에서 단축 평가하세요. 일일이 `match`하지 마세요.
- **커스텀 이터레이터에서 개수를 안다면 `size_hint`를 구현**해 `collect`의 사전 할당을 도우세요.
- **인덱스가 정말 필요하면 `enumerate()`**, 두 컬렉션을 짝지으려면 `zip()`을 쓰세요. `for i in 0..v.len()`은 거의 항상 코드 스멜입니다.
- **`clippy`를 신뢰하세요.** `cargo clippy`는 `needless_collect`, `map_clone`, `into_iter_on_ref`, `filter_map` 합치기 등 이터레이터 안티패턴을 정확히 짚어줍니다.

**이건 피하라**

- 어댑터만 호출하고 소비하지 않기(아래 함정 참고) — `must_use` 경고를 무시하지 마세요.
- 단순 부수 효과에 `map`을 쓰기. 부수 효과만 필요하면 `for_each`나 `for` 루프가 맞습니다. `map`은 변환용입니다.
- 한 번 소비한 이터레이터를 재사용하려 하기. 이터레이터는 소진되면 끝입니다(파이썬도 동일하지만 Rust는 소유권이 이를 강제합니다).

---

## 15.9 흔한 함정 (Python 개발자가 자주 하는 실수)

### 함정 1: `collect()` 남발 — 임시 `Vec`의 무덤

파이썬 습관 중 하나는 중간 결과를 리스트로 만들어 다음 단계에 넘기는 것입니다. Rust에서 이걸 그대로 옮기면 불필요한 힙 할당이 생깁니다.

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6];

    // 🚨 안티패턴: 중간에 Vec을 만들었다가 다시 iter() — 임시 Vec 할당 낭비
    let temp: Vec<i32> = nums.iter().filter(|&&x| x % 2 == 0).copied().collect();
    let sum: i32 = temp.iter().sum();

    // ✅ 올바른 방법: 하나의 lazy 체인으로 끝까지
    let sum: i32 = nums.iter().filter(|&&x| x % 2 == 0).sum();
    println!("{}", sum); // 12
}
```

`clippy`는 이를 `needless_collect`로 경고합니다. 어댑터는 lazy하므로 굳이 중간에 실체화(materialize)할 이유가 없습니다 — 마지막 소비자까지 한 번에 흘려보내세요.

### 함정 2: 어댑터를 호출해놓고 소비하지 않기

가장 흔한 함정입니다. 파이썬에서 `map(f, xs)`는 (3.x에서) 제너레이터를 반환하지만, 부주의하게 써도 보통 어디선가 순회되곤 했습니다. Rust에서 어댑터만 호출하고 소비자를 빼먹으면 **아무 일도 일어나지 않습니다.** 컴파일러가 경고로 잡아줍니다.

```rust
fn main() {
    let v = vec![1, 2, 3];
    v.iter().map(|x| println!("{}", x)); // 🚨 아무것도 출력되지 않음! map은 lazy
}
```

실제 `rustc` 출력:

```text
warning: unused `Map` that must be used
 --> src/main.rs:3:5
  |
3 |     v.iter().map(|x| println!("{}", x)); // 소비하지 않음
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: iterators are lazy and do nothing unless consumed
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default

warning: `Iterator::map` call that discard the iterator's values
 --> src/main.rs:3:14
  |
3 |     v.iter().map(|x| println!("{}", x)); // 소비하지 않음
  |              ^^^^---------------------^
```

`iterators are lazy and do nothing unless consumed` — 이 한 줄이 Rust 이터레이터 철학의 핵심입니다. 부수 효과(출력)가 목적이라면 `for_each`나 `for` 루프를 쓰세요. `v.iter().for_each(|x| println!("{}", x));`가 정답입니다.

### 함정 3: 소유권 이동 후 사용 — `into_iter()`의 함정

파이썬에서 리스트를 `map`에 넘긴 뒤에도 원본 리스트를 계속 썼습니다. 참조 시맨틱이니까요. Rust에서 `into_iter()`는 소유권을 가져가므로, 그 뒤에 원본을 쓰면 컴파일러가 막습니다.

```rust
fn main() {
    let names = vec![String::from("alice"), String::from("bob")];
    let upper: Vec<String> = names.into_iter().map(|s| s.to_uppercase()).collect();
    println!("{:?}", upper);
    println!("{:?}", names); // 🚨 컴파일 에러! names는 위에서 이동됨
}
```

실제 `rustc` 출력:

```text
error[E0382]: borrow of moved value: `names`
   --> src/main.rs:5:22
    |
  2 |     let names = vec![String::from("alice"), String::from("bob")];
    |         ----- move occurs because `names` has type `Vec<String>`,
    |               which does not implement the `Copy` trait
  3 |     let upper: Vec<String> = names.into_iter().map(|s| ...).collect();
    |                                    ----------- `names` moved due to this method call
  5 |     println!("{:?}", names); // 소유권이 이미 이동됨
    |                      ^^^^^ value borrowed here after move
    |
note: `into_iter` takes ownership of the receiver `self`, which moves `names`
help: you can `clone` the value and consume it, but this might not be your desired behavior
    |
  3 |     let upper: Vec<String> = names.clone().into_iter()...
    |                                   ++++++++
```

`into_iter takes ownership of the receiver` — 에러 메시지가 정확히 원인을 짚습니다. 원본을 계속 쓰고 싶다면 `names.iter()`로 빌리세요(이 경우 원소가 `&String`이라 `.map(|s| s.to_uppercase())`는 빌린 값에 작동하므로 그대로 동작합니다). 정말 두 곳에서 다 필요하면 `clone()`이지만, 컴파일러도 경고하듯 보통은 빌림이 정답입니다.

### 함정 4: 인덱스 루프 습관 — `for i in 0..len`

C와 파이썬에서 굳어진 인덱스 루프 습관은 Rust에서 두 가지 비용을 부릅니다. 첫째, `v[i]` 인덱스 접근은 (컴파일러가 제거하지 못하는 경우) 런타임 경계 검사를 유발할 수 있습니다. 둘째, 가독성이 떨어지고 off-by-one 버그의 온상입니다.

```rust
fn main() {
    let v = vec!["a", "b", "c"];

    // 🚨 C/Python 습관: 인덱스 루프
    for i in 0..v.len() {
        println!("{}: {}", i, v[i]);
    }

    // ✅ Rust 방식: enumerate
    for (i, item) in v.iter().enumerate() {
        println!("{}: {}", i, item);
    }
}
```

인덱스가 정말 필요한 게 아니라면(예: 인접 원소 비교) 인덱스 자체를 쓰지 마세요. 두 컬렉션을 동시에 도는 `for i in 0..len { a[i] + b[i] }` 패턴은 `a.iter().zip(b.iter())`로 바꾸면 경계 검사도, 길이 불일치 버그도 사라집니다.

### 함정 5: `filter(|&&x| ...)`의 이중 참조 — `&&i32`라는 낯선 타입

파이썬 개발자가 Rust 이터레이터에서 가장 처음 당황하는 지점입니다. `v.iter()`의 `Item`은 `&i32`(원소를 빌린 참조)인데, `filter`는 클로저에 `Item` 자체가 아니라 `&Item`을 넘깁니다 — *원소를 소비하지 않고 술어만 검사*하기 위해서입니다. 그래서 `filter`의 클로저 인자는 `&&i32`, 즉 **참조의 참조(이중 참조)** 가 됩니다.

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];

    // 🚨 흔한 실수: x를 i32로 착각하고 그냥 산술 연산
    let evens: Vec<&i32> = v.iter().filter(|x| x % 2 == 0).collect();
    println!("{:?}", evens);
}
```

`x`의 타입은 `&&i32`인데 `% 2`를 하려니 컴파일러가 막습니다.

```text
error[E0369]: cannot calculate the remainder of `&&{integer}` divided by `{integer}`
 --> src/main.rs:5:50
  |
5 |     let evens: Vec<&i32> = v.iter().filter(|x| x % 2 == 0).collect();
  |                                                - ^ - {integer}
  |                                                |
  |                                                &&{integer}
  |
help: `%` can be used on `&{integer}` if you dereference the left-hand side
  |
5 |     let evens: Vec<&i32> = v.iter().filter(|x| *x % 2 == 0).collect();
  |                                                +
```

`cannot calculate the remainder of &&{integer}` — 에러가 정확히 "이중 참조에는 나머지 연산을 못 한다"고 알려줍니다. 해법은 두 겹을 벗기는 것입니다. **패턴 매칭으로 한 번에 벗기는 `|&&x|`가 관용적**입니다.

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];

    // ✅ 관용적: 패턴 |&&x| 로 두 겹의 참조를 한 번에 벗겨 x: i32 를 얻는다
    let evens: Vec<&i32> = v.iter().filter(|&&x| x % 2 == 0).collect();
    println!("{:?}", evens); // [2, 4, 6]

    // 같은 의미, 역참조 방식 (덜 관용적)
    let evens2: Vec<&i32> = v.iter().filter(|x| **x % 2 == 0).collect();
    println!("{:?}", evens2);

    // into_iter()면 Item = i32, filter 클로저는 &i32 → 한 겹만 벗긴다 |&x|
    let evens3: Vec<i32> = v.into_iter().filter(|&x| x % 2 == 0).collect();
    println!("{:?}", evens3);
}
```

규칙을 정리하면: **참조 이터레이터(`iter()`)에 `filter`를 걸면 `|&&x|`, 소유 이터레이터(`into_iter()`)에 걸면 `|&x|`**. 반면 `map`은 `Item` 자체(`&i32` 또는 `i32`)를 넘기므로 한 겹만 벗기면 됩니다(`map(|&x| ...)` 또는 `map(|x| ...)`). `filter`만 인자 참조가 한 겹 더 깊다는 점이 핵심이며, 이는 "술어가 원소를 빌려서 보기만 한다"는 설계에서 나온 필연입니다. `clippy`는 이 패턴이 어긋날 때 `clippy::op_ref` 등으로 보조해줍니다.

---

## 15.10 연습문제

읽기만 해서는 손에 붙지 않습니다. 아래 네 문제는 전부 *어댑터 + 소비자*의 조합으로 풀립니다. 각 문제는 먼저 손으로 풀어 보고(가능하면 `rustc`로 컴파일까지), 그다음 정답 스케치와 자기검증 값을 맞춰 보세요. Rustlings처럼 "컴파일이 통과하고 `assert_eq!`가 패닉 없이 끝나면 정답"입니다.

### 문제 1 — `map` + `filter` + `sum`

`vec![1, 2, 3, 4, 5, 6]`에서 **짝수만 골라 제곱한 뒤 모두 더한 값**을 구하는 한 줄 체인을 작성하세요. 중간 `Vec`을 만들지 말고 lazy 체인 하나로 끝내야 합니다.

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6];
    let answer: i32 = /* 여기에 체인 */;
    assert_eq!(answer, 56);
    println!("{}", answer);
}
```

> **정답 스케치.** `iter()`의 `Item`은 `&i32`이므로 `filter`는 `|&&x|`, `map`은 `|&x|`로 참조를 벗깁니다. 핵심 한 줄: `nums.iter().filter(|&&x| x % 2 == 0).map(|&x| x * x).sum()`. 자기검증: `4 + 16 + 36 = 56`이 나오면 정답. `filter`에서 `|&x|`(한 겹만)로 쓰면 함정 5의 `error[E0369]: cannot calculate the remainder of &&{integer}...`를 보게 됩니다. `sum()`의 반환 타입 어노테이션(`: i32`)을 빼면 `error[E0283]: type annotations needed`(`cannot satisfy _: Sum<i32>`)가 뜨니, 소비자엔 타입을 달아 주세요.

### 문제 2 — `fold`로 단어 빈도 세기

문자열 `"the quick brown the lazy the"`를 공백으로 쪼개, **단어→등장 횟수** `HashMap`을 `fold` 하나로 만드세요. (`Counter`를 직접 손으로 만드는 셈입니다.)

```rust
use std::collections::HashMap;
fn main() {
    let text = "the quick brown the lazy the";
    let freq: HashMap<&str, i32> = /* fold 체인 */;
    assert_eq!(freq["the"], 3);
}
```

> **정답 스케치.** `fold`의 초기값은 빈 맵, 누적 함수는 맵을 받아 갱신 후 *되돌려줘야* 합니다(소유권을 다시 반환). 핵심: `text.split_whitespace().fold(HashMap::new(), |mut acc, w| { *acc.entry(w).or_insert(0) += 1; acc })`. 자주 하는 실수는 마지막에 `acc`를 반환하지 않는 것 — 그러면 `error[E0308]: mismatched types`(클로저가 `()`를 반환해 누적 타입과 불일치)가 납니다. 자기검증: `freq["the"] == 3`, `freq["lazy"] == 1`.

### 문제 3 — `collect`로 `Result` 모으기 (단축 평가)

`&str` 슬라이스 `["10", "20", "30"]`을 `Vec<i32>`로 파싱하되, **하나라도 실패하면 전체를 `Err`로** 만드세요. 일일이 `match`하지 말고 `collect` 한 번으로 끝내야 합니다.

```rust
fn main() {
    let good = ["10", "20", "30"];
    let parsed: /* 타입? */ = good.iter().map(|s| s.parse::<i32>()).collect();
    assert_eq!(parsed.unwrap(), vec![10, 20, 30]);

    let bad = ["10", "x", "30"];
    let parsed2: Result<Vec<i32>, _> = bad.iter().map(|s| s.parse::<i32>()).collect();
    assert!(parsed2.is_err());
}
```

> **정답 스케치.** 마법은 `FromIterator`가 `Result<Vec<T>, E>`에 대해 구현돼 있다는 것입니다. 목표 타입을 `Result<Vec<i32>, std::num::ParseIntError>`(또는 `Result<Vec<i32>, _>`)로 어노테이트하면, `collect`가 `Iterator<Item = Result<i32, E>>`를 *통째로 뒤집어* 첫 `Err`에서 단축 평가합니다. 자기검증: `good`은 `Ok([10, 20, 30])`, `bad`는 `"x"`에서 멈춰 `Err(ParseIntError)`. 타입을 `Vec<Result<i32, _>>`로 적으면 단축 평가 없이 *모든* 결과를 담은 벡터가 되니, 의도와 다른 타입을 적지 않도록 주의하세요.

### 문제 4 — `zip` + `map` + `sum`으로 내적(dot product)

길이가 같은 두 배열 `a = [1, 2, 3]`, `b = [4, 5, 6]`의 **내적**(원소별 곱의 합)을 인덱스 루프 없이 구하세요.

```rust
fn main() {
    let a = [1, 2, 3];
    let b = [4, 5, 6];
    let dot: i32 = /* zip + map + sum */;
    assert_eq!(dot, 32);
}
```

> **정답 스케치.** `a.iter().zip(b.iter())`는 `(&i32, &i32)` 튜플 이터레이터를 만듭니다. `map(|(&x, &y)| x * y)`로 튜플을 분해하며 참조를 벗기고, `sum()`으로 합칩니다. 핵심: `a.iter().zip(b.iter()).map(|(&x, &y)| x * y).sum()`. 자기검증: `1*4 + 2*5 + 3*6 = 4 + 10 + 18 = 32`. `zip`은 짧은 쪽에서 멈추므로 길이가 달라도 패닉하지 않고 안전하게 잘립니다 — `for i in 0..a.len()` 인덱스 루프였다면 길이 불일치 시 경계 검사 패닉(`index out of bounds`)이 났을 코드를, `zip`이 컴파일 타임 안전성으로 바꿔 줍니다.

---

Chapter 12~13에서 익힌 `Option`/`Result`/`?`가 이터레이터의 `filter_map`과 `collect::<Result<_,_>>()` 안에서 다시 살아나는 것을, Chapter 14의 `Vec`/`HashMap`이 `collect`의 목적지로 등장하는 것을 보았습니다. 이터레이터는 Rust의 데이터 처리가 한데 모이는 합류점입니다. 그리고 그 모든 추상화가 어셈블리에서 0의 비용으로 증발한다는 것을, 동일한 기계어를 눈으로 확인하며 증명했습니다.

여기까지가 "데이터를 다루는" 여정의 마지막입니다 — 에러, 컬렉션, 그리고 이터레이션. 이제 진짜 클라이맥스로 갑니다. 파이썬 개발자를 십수 년간 괴롭혀온 GIL(Global Interpreter Lock)을 박살 내고, 모든 CPU 코어를 동시에 깨우는 **Part 6: 동시성(Concurrency)** 입니다. 흥미롭게도 그 시작점은 이번 챕터에서 본 클로저와, `rayon`의 `par_iter()`가 보여준 "이터레이터를 병렬로 흘려보내기"입니다. Chapter 16에서는 `std::thread`로 진짜 OS 스레드를 띄우고, Rust의 타입 시스템이 어떻게 데이터 레이스(Data Race)를 *컴파일 타임에* 원천 봉쇄하는지 — "두려움 없는 동시성"의 실체를 파헤칩니다.

---

## References

- https://doc.rust-lang.org/book/ch13-02-iterators.html
- https://doc.rust-lang.org/book/ch13-04-performance.html
- https://doc.rust-lang.org/std/iter/trait.Iterator.html
- https://doc.rust-lang.org/std/iter/trait.IntoIterator.html
- https://doc.rust-lang.org/rust-by-example/trait/iter.html
- https://docs.rs/itertools/latest/itertools/
