# Chapter 5. 참조와 대여 (Borrowing & References)

파이썬에서 이런 디버깅을 해본 적 있나요? 어떤 리스트가 분명히 `[1, 2, 3]`이어야 하는데, 한참 떨어진 다른 함수가 같은 리스트 객체를 손에 쥐고 있다가 슬그머니 원소를 하나 더 밀어넣어(`append`) 버려서, 정작 버그가 터진 지점에서는 "대체 누가 이걸 건드렸지?"라며 `grep`으로 코드 전체를 뒤지던 그 밤 말입니다. 파이썬의 객체는 한 번 만들어지면 여러 변수, 여러 함수, 여러 자료구조가 동시에 같은 메모리를 가리킬 수 있고(aliasing), 그중 누구든 그 객체를 변형(mutation)할 수 있습니다. 이 **별칭(Aliasing)과 변형(Mutation)의 자유로운 공존**이야말로 파이썬의 편리함인 동시에, 추적 불가능한 버그의 근원입니다.

[Chapter 4](chapter_04.md)에서 우리는 소유권(Ownership)을 배웠습니다. 값을 다른 변수에 대입하거나 함수에 넘기면 소유권이 이동(Move)하고, 원래 변수는 무효화됩니다. 그런데 이 규칙을 곧이곧대로 따른다면 정상적인 프로그래밍이 불가능합니다. 리스트의 길이를 재는 함수에 리스트를 넘겼더니 함수가 끝나는 순간 리스트가 통째로 해제되어 버린다면, 매번 `clone()`으로 힙 메모리를 복사해서 넘겨야 할 테니까요. 그것은 GC의 비용을 피하려다 더 비싼 복사 비용을 떠안는 어리석은 거래입니다.

파이썬은 이 문제를 **객체 참조 전달(Pass-by-object-reference)**로 해결합니다. 함수에 객체를 넘기면 객체의 메모리 주소(포인터)만 전달되고, 함수는 그 주소를 통해 원본을 읽고 쓸 수 있습니다. Rust에도 똑같이 "주소만 넘기는" 메커니즘이 있습니다. 이를 **참조(Reference)**라 부르고, 참조를 만드는 행위를 **대여(Borrowing)**라 부릅니다. 소유권은 그대로 둔 채 "잠시 빌려주는" 것입니다.

하지만 여기서 두 언어의 길은 완전히 갈라집니다. 파이썬의 참조 전달에는 아무런 규칙이 없습니다. 누구나, 언제든, 같은 객체를 빌려서 읽거나 쓸 수 있습니다. 반면 Rust의 대여에는 **단 하나의 절대 규칙**이 있고, 이 규칙은 컴파일러(Borrow Checker)에 의해 무자비하게 강제됩니다. 그 규칙은 한 문장으로 요약됩니다.

> **불변 참조는 여럿이거나(XOR) 가변 참조는 하나거나, 둘 중 하나만 가능하다.** (Many readers XOR one writer.)

이 한 줄짜리 규칙이 어떻게 데이터 경쟁(Data Race)을 *컴파일 타임에* 물리적으로 박멸하는지, 그리고 그 대가로 우리가 무엇을 배워야 하는지 — 이 장 전체가 그 이야기입니다.

---

## 5.1 The Python Way — 별칭과 변형의 자유, 그리고 그 대가

먼저 우리가 지금까지 살아온 방식을 정직하게 들여다봅시다. 파이썬에서 객체를 함수에 넘기는 것은 "참조의 사본"을 넘기는 것입니다. 객체 자체가 복사되지 않습니다.

```python
from __future__ import annotations

def calculate_length(s: list[int]) -> int:
    # 리스트의 주소(참조)를 넘겨받아 길이만 읽음. 복사는 일어나지 않음.
    return len(s)

my_list = [1, 2, 3]
length = calculate_length(my_list)
print(my_list)  # [1, 2, 3] — my_list는 여전히 유효
```

여기까지는 평화롭습니다. `calculate_length`는 리스트를 읽기만 하니까요. 문제는 함수가 원본을 *변형*할 수 있다는 데서 시작됩니다.

```python
def append_audit_log(events: list[str], event: str) -> None:
    events.append(event)  # 인자로 받은 원본 리스트를 직접 수정

shared_events: list[str] = []
append_audit_log(shared_events, "user_login")
append_audit_log(shared_events, "file_upload")
print(shared_events)  # ['user_login', 'file_upload'] — 원본이 바뀜
```

이것이 의도된 동작이라면 좋습니다. 그런데 현실의 코드베이스에서는 같은 리스트 객체가 여러 곳에 *별칭(alias)*으로 흩어져 있다가, 그중 한 곳에서의 변형이 다른 곳을 망가뜨립니다. 전형적인 시나리오를 봅시다.

```python
class Order:
    def __init__(self, items: list[str]) -> None:
        self.items = items  # 🚨 외부 리스트를 그대로 보관 (방어적 복사를 안 함)

default_items = ["bread", "milk"]

order_a = Order(default_items)
order_b = Order(default_items)  # 같은 리스트 객체를 두 주문이 공유!

order_a.items.append("eggs")

print(order_b.items)  # ['bread', 'milk', 'eggs'] — B는 건드린 적도 없는데 바뀜
print(default_items)  # ['bread', 'milk', 'eggs'] — 원본 상수마저 오염됨
```

`order_a`의 장바구니에 계란을 담았더니 `order_b`의 장바구니와 심지어 "기본값" 상수까지 오염됩니다. 세 변수가 모두 **같은 힙 객체를 가리키는 별칭**이고, 그중 하나를 통한 변형이 전부에게 보이기 때문입니다. 파이썬 개발자라면 이 버그를 막기 위해 `self.items = list(items)`처럼 방어적 복사(defensive copy)를 습관적으로 작성하거나, `copy.deepcopy()`를 남발하거나, 불변 자료구조(`tuple`, `frozenset`)를 쓰는 등의 규율(discipline)에 의존합니다. 하지만 규율은 사람이 지키는 것이고, 사람은 새벽 3시에 실수합니다.

> **💡 핵심:** 파이썬의 버그는 "별칭(여러 참조)"과 "변형(쓰기)"이 *아무 제약 없이 공존*하기 때문에 생깁니다. Aliasing + Mutation = 위험. 이 등식을 기억하세요. Rust의 대여 규칙은 정확히 이 등식의 좌변을 컴파일 타임에 깨뜨리도록 설계되었습니다.

### 더 무서운 사례: 순회 중 변형 (Iterator Invalidation)

별칭-변형 문제가 가장 악랄하게 드러나는 곳은 컬렉션을 순회하면서 동시에 그 컬렉션을 수정할 때입니다.

```python
numbers = [1, 2, 3, 4, 5]
for n in numbers:
    if n % 2 == 0:
        numbers.append(n * 10)  # 순회 중인 리스트를 수정
# 리스트는 운이 좋으면 끝나고, 운이 나쁘면 무한 루프에 빠지거나 원소를 건너뜀
```

리스트의 경우 파이썬은 예외조차 던지지 않습니다. 그저 조용히 잘못된 결과를 내거나 무한 루프에 빠집니다. 딕셔너리라면 그나마 `RuntimeError: dictionary changed size during iteration`을 던져주지만, 이건 **런타임에, 그것도 운이 좋아야** 잡히는 버그입니다. 프로덕션의 특정 입력에서만 터지는 종류의 버그죠. 이 장의 끝에서 우리는 Rust 컴파일러가 정확히 이 코드를 *컴파일조차 거부*하는 것을 보게 됩니다.

---

## 5.2 The Rust Mental Model — 빌려주되, 규칙을 강제한다

Rust도 "주소만 넘기는" 행위를 지원합니다. 앰퍼샌드(`&`) 기호 하나로요. 하지만 Rust는 파이썬과 달리 대여에 **권한(permission)의 개념**을 부여합니다. 참조에는 두 종류가 있습니다.

- **불변 참조 `&T`**: 읽기 권한만. 도서관에서 책을 빌려 읽는 것과 같습니다. 여러 명이 동시에 같은 책을 읽을(borrow) 수 있습니다.
- **가변 참조 `&mut T`**: 읽기 + 쓰기 권한. 책에 빨간 펜으로 교정을 보는 편집자입니다. 한 권을 동시에 두 명이 교정할 수는 없습니다 — 한 명이 3페이지를 지우는 동안 다른 명이 그 페이지를 읽으면 무엇을 보겠습니까?

Rust의 핵심 통찰은 이것입니다. **공유(여러 별칭)와 변형(쓰기)을 동시에 허용하지 않으면, 데이터 경쟁과 메모리 오염은 정의상 불가능해진다.** 그래서 Rust는 다음 불변식(invariant)을 컴파일 타임에 강제합니다.

> 어느 한 시점에, 특정 데이터에 대해:
> - 불변 참조 `&T`를 **원하는 만큼 여러 개** 가질 수 있다. **단, 가변 참조는 0개여야 한다.**
> - 또는 가변 참조 `&mut T`를 **정확히 하나** 가질 수 있다. **단, 다른 어떤 참조(불변이든 가변이든)도 0개여야 한다.**

이것이 **"불변 참조 여럿 XOR 가변 참조 하나"** 규칙입니다. XOR(배타적 논리합)라는 표현이 정확합니다. 두 상태는 결코 공존할 수 없습니다. 파이썬 개발자에게 이것은 처음엔 족쇄처럼 느껴집니다. "내 객체인데 내 마음대로 못 쓴다고?" 하지만 곧 깨닫게 됩니다. 이 족쇄는 당신이 파이썬에서 방어적 복사와 코드 리뷰와 새벽 디버깅으로 *수동으로* 지켜오던 바로 그 규율을, 컴파일러가 *대신* 지켜주는 것입니다.

### Python ↔ Rust 참조 모델 매핑

| 개념 | Python | Rust | 핵심 차이 |
|------|--------|------|-----------|
| 객체 전달 | Pass-by-object-reference (항상) | `&T` / `&mut T` 명시 | Rust는 읽기/쓰기 권한을 타입으로 구분 |
| 별칭(aliasing) | 무제한 허용 | 불변 참조만 무제한, 가변은 단 하나 | Rust는 공유 가변성을 금지 |
| 변형 가능 여부 | 모든 참조가 변형 가능 | `&mut`만 변형 가능, `&`는 읽기 전용 | 변형 권한이 타입에 박혀 있음 |
| 방어적 복사 | `list(x)`, `copy.deepcopy(x)` 수동 | 보통 불필요 (참조로 충분) | 복사를 강제할 일이 거의 없음 |
| 순회 중 수정 | 런타임 오류 또는 조용한 버그 | **컴파일 에러** (E0502) | 위험을 빌드 타임으로 끌어올림 |
| 데이터 경쟁 | GIL로 부분 방어, 멀티프로세싱 시 위험 | 컴파일 타임에 원천 차단 | `Send`/`Sync`와 결합 (Chapter 16~17) |
| 참조 수명 관리 | GC가 런타임 추적 | Borrow Checker가 컴파일 타임 검증 | 0 런타임 오버헤드 (Chapter 6) |

이 표의 마지막 두 행은 이 책 후반부의 복선입니다. 대여 규칙은 단일 스레드의 메모리 안전을 넘어, [Chapter 16](chapter_16.md)·[Chapter 17](chapter_17.md)에서 다룰 "두려움 없는 동시성(Fearless Concurrency)"의 기초가 됩니다. 그리고 참조가 "원본보다 오래 살아남지 못하도록" 보장하는 메커니즘이 [Chapter 6](chapter_06.md)의 수명(Lifetime)입니다.

---

## 5.3 불변 참조 (`&T`): 소유권 없이 읽기

가장 기본적인 대여입니다. `&` 기호로 값의 소유권은 넘기지 않고 "읽기 권한"만 빌려줍니다. [Chapter 4](chapter_04.md)에서 함수에 `Vec`을 넘기면 소유권을 빼앗겨 다시 못 쓰던 답답함을, 참조가 해소합니다.

* **The Python Way**
  ```python
  def calculate_length(s: list[int]) -> int:
      return len(s)  # 리스트의 주소를 넘겨받아 길이만 읽음

  my_list = [1, 2, 3]
  length = calculate_length(my_list)
  print(my_list)  # my_list는 여전히 유효함
  ```

* **The Rust Implementation**
  ```rust
  // s는 Vec<i32>의 '참조(Reference)' 타입인 &Vec<i32>를 받습니다.
  fn calculate_length(s: &Vec<i32>) -> usize {
      s.len()
  } // 함수가 끝나도 s가 가리키는 원본은 버려지지(Drop) 않습니다. 소유권이 없기 때문입니다.

  fn main() {
      let my_vec = vec![1, 2, 3];

      // & 기호를 붙여서 값을 빌려(Borrow) 줍니다.
      let length = calculate_length(&my_vec);

      // my_vec는 소유권을 잃지 않았으므로 계속 사용할 수 있습니다.
      println!("Length of {:?} is {}", my_vec, length);
  }
  ```

불변 참조(`&`)는 책을 여러 명이 동시에 읽는 것과 같습니다. **동시에 여러 개의 불변 참조를 만드는 것이 가능합니다.** 단, 값을 읽을 수만 있고 수정할 수는 없습니다.

```rust
fn main() {
    let data = vec![1, 2, 3];

    let r1 = &data;
    let r2 = &data;
    let r3 = &data; // 불변 참조는 몇 개든 OK

    println!("{:?} {:?} {:?}", r1, r2, r3); // [1, 2, 3] x 3
    // r1.push(4); // 🚨 불변 참조로는 수정 불가
}
```

함수 시그니처에서 `&Vec<i32>`보다 `&[i32]`(슬라이스)를 받는 것이 관용적입니다(5.6절에서 다룹니다). 하지만 개념을 먼저 잡기 위해 여기서는 `&Vec<i32>`로 둡니다.

> **Under the Hood: 참조는 그냥 포인터입니다 (8바이트)**
>
> 파이썬의 변수는 `PyObject*`를 가리키고, 그 `PyObject`는 참조 카운트(`ob_refcnt`), 타입 포인터(`ob_type`) 등 최소 16바이트의 헤더를 짊어집니다. 참조를 하나 더 만들 때마다 `Py_INCREF`로 카운트를 원자적으로 증가시켜야 하고, 이는 캐시 라인을 더럽히고(cache pollution) GIL과 얽힙니다.
>
> Rust의 불변 참조 `&T`는 그런 것이 없습니다. 그저 **메모리 주소를 담은 64비트 정수**, 즉 8바이트짜리 포인터입니다. 참조를 만들어도 카운터 증가가 없고, 참조를 버려도 카운터 감소가 없습니다. 런타임에는 사실상 아무 일도 일어나지 않습니다 — 대여의 적법성은 전부 컴파일 타임에 이미 검증됐기 때문입니다.
>
> ```rust
> use std::mem;
> fn main() {
>     println!("&i32:      {} bytes", mem::size_of::<&i32>());      // 8
>     println!("&Vec<i32>: {} bytes", mem::size_of::<&Vec<i32>>()); // 8
>     println!("&mut i32:  {} bytes", mem::size_of::<&mut i32>());  // 8
>     println!("Vec<i32>:  {} bytes", mem::size_of::<Vec<i32>>());  // 24 (ptr+len+cap)
> }
> ```
>
> `&Vec<i32>`가 8바이트라는 점에 주목하세요. 24바이트짜리 `Vec` 구조체 전체가 아니라, 그 구조체가 놓인 스택 슬롯의 주소 하나만 넘어갑니다. 이것이 "참조는 무비용(zero-cost)"이라는 말의 의미입니다. 100만 원소짜리 벡터를 함수에 넘기든 빈 벡터를 넘기든, 넘어가는 것은 똑같이 8바이트 주소 하나입니다. 반면 `copy.deepcopy`나 Rust의 `.clone()`은 힙 버퍼 전체를 OS로부터 재할당받아 복사합니다 — 100만 원소면 그만큼의 비용입니다.

---

## 5.4 가변 참조 (`&mut T`): 빌려서 수정하기

빌려온 값을 수정하려면 가변 참조가 필요합니다. 파이썬에서는 모든 참조가 묵시적으로 변형 가능하지만, Rust는 쓰기 권한을 `&mut` 키워드로 *명시적으로 요청*하게 합니다.

```rust
// 매개변수 s를 가변 참조(&mut)로 받습니다.
fn add_element(s: &mut Vec<i32>) {
    s.push(4); // 원본 데이터 수정 가능!
}

fn main() {
    // 1. 변수 자체가 가변(mut)이어야 합니다.
    let mut my_vec = vec![1, 2, 3];

    // 2. 가변 참조(&mut)로 빌려줍니다.
    add_element(&mut my_vec);

    println!("{:?}", my_vec); // [1, 2, 3, 4]
}
```

여기에는 두 단계의 명시성이 있습니다. 첫째, 변수 `my_vec` 자체가 `let mut`로 가변이어야 합니다. 둘째, 호출 지점에서 `&mut`로 가변 대여임을 드러내야 합니다. 파이썬에서는 `add_element(my_vec)`만 보고는 이 함수가 내 리스트를 변형하는지 알 수 없습니다 — 함수 본문을 열어봐야 합니다. Rust에서는 **호출 지점의 `&mut`만 보고도** "아, 이 함수가 내 데이터를 수정하겠구나"를 압니다. 변형 가능성이 타입과 호출 표기에 박혀 있어, 코드를 읽는 것만으로 부수 효과(side effect)의 지도가 그려집니다.

> **💡 핵심:** `*` 연산자에 주목하세요. `&mut i32`를 받은 함수에서 그 값을 바꾸려면 `*counter += 1`처럼 역참조(dereference)해야 합니다. `Vec`의 `push` 같은 메서드는 Rust가 자동으로 `(*s).push(...)`로 역참조해주지만(자동 역참조, auto-deref), 원시 타입의 직접 대입에는 `*`가 필요합니다.

```rust
fn bump(counter: &mut i32) {
    *counter += 1; // 역참조 후 수정
}

fn main() {
    let mut n = 10;
    bump(&mut n);
    println!("{}", n); // 11
}
```

---

## 5.5 파이썬엔 없는 규칙: Data Race의 완벽한 차단

Rust의 참조 메커니즘에서 파이썬 개발자들이 가장 고통스러워하는 부분이자, Rust의 가장 위대한 혁신이 바로 여기에 있습니다. 5.2절에서 선언한 XOR 규칙을 이제 코드로 마주합니다.

> **특정 데이터에 대해 '가변 참조(`&mut`)'를 만들면, 그 참조가 살아있는 동안에는 다른 어떠한 참조(불변/가변 모두)도 동시에 존재할 수 없습니다.**

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let r1 = &mut data; // 첫 번째 가변 참조 (쓰기 권한 획득)
    // let r2 = &mut data; // 🚨 컴파일 에러! 동시에 두 개의 쓰기 권한은 불가능 (E0499)
    // let r3 = &data;     // 🚨 컴파일 에러! 누군가 쓰고 있는데 읽으려 하면 안 됨 (E0502)

    r1.push(4); // r1을 다 쓰고 나면
    let r4 = &data; // 여기서부터는 다시 읽기(불변 참조)가 가능해집니다.
    println!("{:?}", r4);
}
```

### 컴파일러 에러 시연 ①: E0502 — 불변 대여 중 가변 대여 시도

규칙을 어기면 컴파일러가 정확히 무엇을 막는지 보여줍니다. 다음 코드를 실제로 `cargo build` 하면:

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let r1 = &data;      // 불변 참조
    let r2 = &mut data;  // 가변 참조 시도

    println!("{:?} {:?}", r1, r2);
}
```

`rustc`(1.96.0)는 다음과 같이 출력합니다.

```text
error[E0502]: cannot borrow `data` as mutable because it is also borrowed as immutable
 --> src/main.rs:5:14
  |
4 |     let r1 = &data;      // 불변 참조
  |              ----- immutable borrow occurs here
5 |     let r2 = &mut data;  // 가변 참조 시도
  |              ^^^^^^^^^ mutable borrow occurs here
6 |
7 |     println!("{:?} {:?}", r1, r2);
  |                           -- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
```

에러 메시지를 해독해 봅시다. 컴파일러는 세 곳을 정확히 짚습니다. (1) 4행에서 불변 대여가 시작됐고("immutable borrow occurs here"), (2) 5행에서 가변 대여를 시도했으며("mutable borrow occurs here"), (3) 7행에서 그 불변 참조 `r1`이 *여전히 사용된다*("immutable borrow later used here")는 것입니다. 즉 `r1`의 생애가 `r2`의 생성을 가로질러 살아있기 때문에, XOR 규칙이 깨진 것입니다. 이것이 파이썬이라면 아무 불평 없이 실행됐을 코드입니다.

### 컴파일러 에러 시연 ②: E0499 — 두 개의 가변 대여

가변 참조를 두 개 만들려 하면 다른 번호의 에러가 나옵니다.

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let r1 = &mut data;
    let r2 = &mut data; // 두 번째 가변 참조

    r1.push(4);
    r2.push(5);
}
```

```text
error[E0499]: cannot borrow `data` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let r1 = &mut data;
  |              --------- first mutable borrow occurs here
5 |     let r2 = &mut data;
  |              ^^^^^^^^^ second mutable borrow occurs here
6 |
7 |     r1.push(4);
  |     -- first borrow later used here
```

"cannot borrow `data` as mutable more than once at a time" — 한 번에 두 번 가변 대여할 수 없다는 명확한 선언입니다. `E0502`(불변+가변 충돌)와 `E0499`(가변+가변 충돌)는 같은 XOR 규칙의 두 얼굴입니다.

### Under the Hood: 왜 이토록 엄격할까요?

파이썬의 `threading` 모듈로 개발해 본 경험이 있다면, 여러 스레드가 동시에 같은 리스트를 수정하려 할 때 데이터가 깨지는 것을 막기 위해 `Lock`을 걸어야 함을 알 것입니다. 그런데 `Lock`을 거는 것은 개발자의 *규율*입니다. 깜빡하고 락 없이 공유 자료를 만지면, 파이썬은(GIL이 보호하지 못하는 복합 연산에서) 조용히 데이터를 망가뜨립니다.

데이터 경쟁의 본질은 이것입니다. **(1) 두 개 이상의 접근 경로가 같은 메모리를 가리키고(aliasing), (2) 그중 적어도 하나가 쓰기이며(mutation), (3) 동기화가 없으면** 데이터 경쟁입니다. Rust의 XOR 규칙은 이 세 조건 중 (1)과 (2)의 공존 자체를 금지합니다. 쓰기(`&mut`)가 있으면 별칭이 0개고, 별칭이 여럿(`&`)이면 쓰기가 0개입니다. 따라서 데이터 경쟁의 전제 조건이 *문법적으로* 성립할 수 없습니다.

구체적인 메모리 재난을 그려봅시다. `Vec`이 용량(capacity)을 초과해 `push`되면, 더 큰 힙 버퍼를 새로 할당받고 기존 원소를 복사한 뒤 *옛 버퍼를 해제*합니다(reallocation). 만약 스레드 A가 `&mut data`로 `push`하며 재할당을 일으키는 동안, 스레드 B가 같은 `data`의 `&` 참조로 옛 버퍼의 3번 원소를 읽고 있었다면? B의 포인터는 이미 해제된 쓰레기 메모리(dangling pointer)를 가리킵니다. Use-after-free, 즉 수많은 C/C++ 보안 취약점의 원흉입니다.

Rust 컴파일러(Borrow Checker)는 애초에 "재할당을 일으킬 수 있는 `&mut`가 살아있는 동안에는 그 어떤 `&`도 공존 못 한다"를 컴파일 타임에 강제합니다. 그래서 위 시나리오는 *작성조차 불가능*합니다. 그리고 결정적으로, 이 모든 검증은 **런타임 비용이 0**입니다 — 락도, 카운터도, 런타임 검사도 없습니다. 검증된 후의 기계어는 그냥 포인터를 다루는 C 코드와 동일한 속도로 달립니다. 이것이 [Chapter 1](chapter_01.md)에서 약속한 무비용 추상화(Zero-cost Abstraction)의 실체입니다.

### NLL: 참조의 스코프는 "마지막 사용"까지

규칙이 엄격하긴 해도, 생각만큼 답답하지는 않습니다. 옛날(2018 이전) Rust는 참조가 *렉시컬 스코프*(중괄호 블록) 끝까지 살아있다고 보았지만, 현재의 컴파일러는 **비렉시컬 수명(Non-Lexical Lifetimes, NLL)**을 사용합니다. 즉 참조의 대여는 **그 참조가 마지막으로 사용된 지점**에서 끝난 것으로 간주됩니다. 다음 코드가 그 증거입니다.

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let r1 = &data;
    println!("{:?}", r1); // r1의 마지막 사용 지점. NLL에 의해 여기서 r1의 대여가 끝남.

    let r2 = &mut data;   // NLL 덕분에 OK! r1은 이미 죽었으므로 충돌하지 않음
    r2.push(4);
    println!("{:?}", r2); // [1, 2, 3, 4]
}
```

`r1`과 `r2`는 같은 블록에 있지만, `r1`이 4행에서 마지막으로 쓰이고 다시 등장하지 않으므로 5행 시점에는 이미 "죽은" 참조입니다. 따라서 가변 참조 `r2`를 만들어도 충돌하지 않습니다. NLL은 "참조는 만들어진 곳부터 마지막으로 쓰인 곳까지만 산다"는 직관을 컴파일러에 심어, 우리가 불필요하게 블록을 쪼개거나 변수를 일찍 버릴 필요를 없애줍니다.

> 💡 이 "마지막 사용까지"라는 시간 구간이 바로 [Chapter 6](chapter_06.md)에서 본격적으로 다룰 **수명(Lifetime)**의 비공식적 정의입니다. 지금은 컴파일러가 알아서 추론해 주지만, 함수가 참조를 받아 참조를 반환하는 순간 우리가 직접 `'a` 같은 수명 표기를 달아 컴파일러에게 의도를 설명해야 합니다.

### 한 줄 요약 규칙

1. **아무리 많은 불변 참조(`&T`)라도 동시에 존재할 수 있다.** (Readers-Many)
2. **가변 참조(`&mut T`)는 특정 시점에 오직 단 하나만 존재할 수 있다.** (Writer-One)
3. **가변 참조와 불변 참조는 동시에 섞어 쓸 수 없다.**
4. **참조의 수명은 마지막 사용까지다(NLL).** — 블록 끝까지가 아니다.

### Under the Hood: `&mut T`의 `noalias`와 LLVM 최적화 (C의 `restrict`와 비교)

XOR 규칙은 메모리 안전을 위한 안전벨트로만 보이기 쉽습니다. 하지만 이 규칙에는 안전과 무관한, *순수하게 성능을 위한* 두 번째 얼굴이 있습니다. 바로 컴파일러에게 **별칭이 없음(no aliasing)을 약속**한다는 점입니다. C 프로그래머라면 이 약속의 가치를 압니다 — C99의 `restrict` 키워드가 바로 그것이니까요.

C에서 `void f(int *dst, int *src)`를 받으면, 컴파일러는 `dst`와 `src`가 같은 메모리를 가리킬 수도 있다고 가정해야 합니다(보수적 가정). 그래서 `*dst`를 두 번 더하는 코드라면 `*src`를 *매번 다시 읽어야* 합니다 — 첫 번째 `*dst += *src`가 `*src`를 바꿨을 가능성을 배제할 수 없으니까요. 개발자가 "이 둘은 절대 겹치지 않는다"고 `restrict`로 명시해야만 컴파일러가 `*src`를 한 번만 읽고 재사용하는 최적화를 합니다. 문제는 `restrict`가 *개발자의 약속*일 뿐, 컴파일러가 검증하지 못한다는 것입니다. 약속을 어기면 정의되지 않은 동작(UB)입니다.

Rust에서는 이 약속이 공짜로, 그리고 컴파일러가 *검증한 채로* 주어집니다. `&mut T`는 정의상 그 데이터에 대한 유일한 접근 경로이므로(XOR 규칙), Rust는 LLVM에게 모든 `&mut` 인자에 `noalias` 속성을, 모든 `&` 인자에 `noalias readonly` 속성을 붙여 넘깁니다. 직접 확인해 봅시다.

```rust
#[no_mangle]
pub fn add_assign(dst: &mut i64, src: &i64) {
    *dst += *src;
    *dst += *src;
}
```

이것을 `rustc -O --emit=llvm-ir`로 컴파일하면 함수 시그니처에 `noalias`가 박힙니다.

```text
define void @add_assign(ptr noalias nocapture noundef align 8 ... %dst,
                        ptr noalias nocapture noundef readonly align 8 ... %src)
start:
  %_3 = load i64, ptr %src       ; *src를 단 한 번만 읽음
  %0  = load i64, ptr %dst
  %reass.add = shl i64 %_3, 1    ; src << 1  =  src * 2
  %1  = add i64 %0, %reass.add   ; dst + 2*src
  store i64 %1, ptr %dst         ; 단 한 번만 씀
  ret void
```

LLVM은 `*src`를 **단 한 번만 로드**하고, 두 번의 `+= *src`를 `dst + 2*src`로 접어(fold) 버렸습니다. `dst`와 `src`가 겹치지 않음을 `noalias`로 알기에 가능한 최적화입니다. 같은 로직을 `noalias`가 없는 원시 포인터(`*mut i64`, `*const i64`)로 쓰면 결과가 달라집니다.

```text
; unsafe fn add_assign_raw(dst: *mut i64, src: *const i64)
  %_3 = load i64, ptr %src       ; 첫 번째 로드
  ...
  store i64 %1, ptr %dst
  %_4 = load i64, ptr %src       ; src를 다시 읽음! (dst가 src를 바꿨을 수 있으니)
  ...
  store i64 %2, ptr %dst         ; 두 번째 store도 따로
```

`noalias`가 없으면 LLVM은 `*src`를 **두 번 로드하고 두 번 store**합니다 — C의 `restrict` 없는 코드와 정확히 같은 보수적 코드입니다. 즉 Rust의 XOR 규칙은 단지 버그를 막는 것을 넘어, *Fortran 수준의 별칭 정보*를 컴파일러에 무상으로 제공합니다. C가 `restrict`로 손수, 검증 없이, UB의 위험을 안고 얻어내던 최적화를, Rust는 `&mut`라는 평범한 참조를 쓰는 것만으로 안전하게 가져갑니다. (역사적 각주: rustc는 한때 LLVM의 `noalias` 버그로 이 속성을 끄기도 했지만, 현재는 안정적으로 방출합니다.)

> 🚨 이 때문에 `unsafe`로 `&mut`의 별칭 규칙을 몰래 어기는 것은 단순한 "규칙 위반"이 아니라 **즉각적인 UB**입니다. 컴파일러는 이미 `noalias`를 믿고 코드를 최적화해 버렸기 때문에, 약속을 어긴 코드는 당신이 작성한 것과 전혀 다르게 동작할 수 있습니다. [Chapter 21](chapter_21.md)에서 unsafe를 다룰 때 이 지점을 다시 만납니다.

---

## 5.6 슬라이스(`&[T]`, `&str`): 일부만 빌리는 Fat Pointer

지금까지의 참조는 "값 하나 전체"를 가리켰습니다. 그런데 벡터의 *일부 구간*만, 또는 문자열의 *부분 문자열*만 소유 없이 바라보고 싶다면? 이것이 **슬라이스(Slice)**입니다. 슬라이스는 Rust에서 가장 자주 쓰이면서도 파이썬 개발자가 가장 자주 헷갈리는 개념이므로 깊이 파고들 가치가 있습니다.

* **The Python Way**
  ```python
  s = "hello rust world"
  print(s[0:5])    # 'hello' — 새 문자열 객체를 할당 (복사)

  v = [10, 20, 30, 40, 50]
  print(v[1:4])    # [20, 30, 40] — 새 리스트 객체를 할당 (복사)
  ```
  파이썬의 슬라이싱 `[a:b]`은 **항상 새 객체를 힙에 할당하고 데이터를 복사**합니다. 1GB 문자열에서 앞 100바이트만 떼어내도 100바이트짜리 새 `str` 객체가 생깁니다. 작아 보이지만, 대용량 데이터 파이프라인에서 이 복사는 누적되어 심각한 비용이 됩니다.

* **The Rust Implementation**
  ```rust
  fn sum_slice(slice: &[i32]) -> i32 {
      slice.iter().sum()
  }

  fn main() {
      let v = vec![10, 20, 30, 40, 50];

      println!("{}", sum_slice(&v[1..4])); // 90  — 복사 없이 구간만 빌림
      println!("{}", sum_slice(&v));       // 150 — 전체를 슬라이스로
      let arr = [1, 2, 3];
      println!("{}", sum_slice(&arr));     // 6   — 배열도 슬라이스로 빌림
  }
  ```
  `&v[1..4]`는 **복사하지 않습니다.** 원본 벡터의 버퍼 안, 인덱스 1번 원소의 주소와 "원소 3개"라는 길이 정보만 묶어 가리킵니다. 1GB 데이터의 한 구간을 슬라이스로 빌려도 비용은 상수(고정 16바이트)입니다.

문자열 슬라이스 `&str`도 같은 원리입니다.

```rust
// 첫 단어를 '빌려서' 반환. 새 String을 할당하지 않고 원본의 일부를 가리킨다.
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &b) in bytes.iter().enumerate() {
        if b == b' ' {
            return &s[..i]; // 0..i 구간 슬라이스
        }
    }
    s // 공백이 없으면 전체가 첫 단어
}

fn main() {
    let sentence = String::from("hello rust world");
    let word = first_word(&sentence); // &String이 &str로 자동 변환(Deref Coercion)
    println!("first word: {}", word); // hello
}
```

여기서 함수가 `&str`를 받는다는 점이 중요합니다. `&String`을 인자로 주면 컴파일러가 자동으로 `&str`로 변환해 줍니다(역참조 강제, Deref Coercion). 그래서 `&str`를 받도록 작성하면 `String`, 문자열 리터럴(`&'static str`), 다른 `&str` 슬라이스를 모두 받을 수 있어 가장 유연합니다. 같은 이유로 함수 인자는 `&Vec<i32>`보다 `&[i32]`를, `&String`보다 `&str`를 받는 것이 관용입니다.

> **Under the Hood: 슬라이스는 Fat Pointer입니다 (16바이트)**
>
> 평범한 참조 `&i32`는 주소만 담은 8바이트 "얇은 포인터(thin pointer)"였습니다. 하지만 슬라이스 참조 `&[i32]`나 `&str`는 다릅니다. 길이를 컴파일 타임에 알 수 없는 동적 크기 타입(DST, Dynamically Sized Type)을 가리키므로, 포인터에 **추가 메타데이터(길이)**를 함께 담아야 합니다. 이를 **뚱뚱한 포인터(Fat Pointer)**라 부르며, 다음 두 워드로 구성됩니다.
>
> ```text
> &[T] 또는 &str 의 메모리 레이아웃 (64비트):
> ┌─────────────────────┬─────────────────────┐
> │  data ptr (8 bytes) │  len (8 bytes)      │   = 16 bytes
> └─────────────────────┴─────────────────────┘
>        │
>        └──> 원본 버퍼의 시작 원소를 가리킴 (소유하지 않음)
> ```
>
> ```rust
> use std::mem;
> fn main() {
>     println!("&i32:  {} bytes", mem::size_of::<&i32>());  // 8  (thin)
>     println!("&[i32]:{} bytes", mem::size_of::<&[i32]>()); // 16 (fat: ptr+len)
>     println!("&str:  {} bytes", mem::size_of::<&str>());   // 16 (fat: ptr+len)
> }
> ```
>
> `&v[1..4]`를 만들면 컴파일러는 `(원본_버퍼_주소 + 1*4바이트, 3)`이라는 16바이트 쌍을 만듭니다 — 시작 주소와 길이뿐, 데이터 복사는 없습니다. `&str`의 `len`은 바이트 길이(UTF-8 인코딩 기준)입니다. 그래서 `&s[0..1]`이 멀티바이트 문자의 경계를 자르면 런타임 패닉이 납니다 — Rust 문자열은 코드포인트가 아니라 바이트로 인덱싱되며, 이 UTF-8 처리의 미묘함은 [Chapter 14](chapter_14.md)에서 깊이 다룹니다.
>
> 같은 16바이트 fat pointer가 트레이트 객체 `&dyn Trait`에서도 등장하는데, 그때는 `(데이터 주소, vtable 주소)` 쌍입니다. 동적 디스패치의 vtable 이야기는 [Chapter 9](chapter_09.md)·[Chapter 10](chapter_10.md)에서 만납니다.

---

## 5.7 재대여(Reborrow)와 깊은/얕은 복사의 재정의

### 재대여: 가변 참조를 함수에 넘겨도 잃지 않는다

가변 참조 `&mut T`는 `Vec`처럼 소유 값이 아니지만, 그 자체로는 `Copy`가 아닙니다. 그렇다면 가변 참조를 함수에 넘기면 그 참조의 소유권을 잃어버려서 두 번 못 쓰는 걸까요? 다음 코드는 멀쩡히 컴파일됩니다.

```rust
fn bump(counter: &mut i32) {
    *counter += 1;
}

fn bump_twice(counter: &mut i32) {
    bump(counter); // counter를 move하지 않고 '재대여(reborrow)'
    bump(counter); // 재대여가 끝났으므로 다시 사용 가능
}

fn main() {
    let mut n = 0;
    let r = &mut n;
    bump_twice(r); // r 자체도 재대여됨
    bump_twice(r);
    println!("n = {}", *r); // 4
}
```

`bump(counter)`를 호출할 때 컴파일러는 암묵적으로 `bump(&mut *counter)`로 해석합니다 — 기존 가변 참조에서 *임시로 새 가변 참조를 떠내는* 것입니다. 이것이 **재대여(Reborrow)**입니다. 호출이 끝나면 떠낸 참조가 사라지고 원래 `counter`가 다시 유일한 가변 참조로 복귀합니다. 그래서 두 번 연속 호출이 가능합니다. 재대여 덕분에 XOR 규칙("가변 참조는 하나뿐")을 어기지 않으면서도 가변 참조를 자연스럽게 전달할 수 있습니다. 파이썬 개발자가 무심코 가변 참조를 넘기고 다시 쓸 때, 이 보이지 않는 재대여가 묵묵히 일을 처리하고 있는 것입니다.

### 얕은 복사와 깊은 복사의 재정의

[Chapter 4](chapter_04.md)에서 보았듯, 파이썬의 대입 `=`은 얕은 복사(참조 공유)이고 `copy.deepcopy()`는 깊은 복사(전체 재할당)입니다. Rust에서 이 스펙트럼은 세 가지로 명확히 분리됩니다.

| 의도 | Python | Rust | 비용 |
|------|--------|------|------|
| 같은 데이터를 함께 보기 (읽기) | `b = a` (별칭) | `let b = &a;` (불변 참조) | 0 (8/16바이트 포인터) |
| 같은 데이터를 함께 수정 | `b = a` (별칭) | `let b = &mut a;` (한 번에 하나) | 0 + XOR 규칙 강제 |
| 독립적인 복제본 만들기 | `copy.deepcopy(a)` | `let b = a.clone();` | 힙 버퍼 전체 복사 |
| 소유권 넘기기 | (개념 없음) | `let b = a;` (Move) | 0 (포인터 이동, 원본 무효화) |

파이썬은 `b = a` 한 줄에 "별칭"과 "이동"의 의미가 뭉뚱그려져 있고, 깊은 복사를 원하면 별도 함수를 호출해야 합니다. Rust는 정반대입니다. **무비용 동작(참조, 이동)이 기본이고, 비싼 힙 복사(`.clone()`)만 코드에 눈에 띄는 꼬리표로 명시**됩니다. 그래서 Rust 코드를 읽으며 `.clone()`을 찾으면 그것이 곧 잠재적 성능 병목 지점의 지도가 됩니다. "참조로 충분한가, 정말 복제가 필요한가?"를 묻는 습관 — 이것이 파이썬에서 Rust로 넘어올 때 길러야 할 핵심 직관입니다.

---

## 5.8 실전 예제 (Hands-on): 로그 분석 파이프라인

이제 배운 것을 한데 모아, 현실적인 미니 프로젝트를 만들어 봅시다. 시나리오는 이렇습니다. 서버 로그 한 덩어리(`&str`)를 받아 (1) 단어 빈도를 집계하고, (2) 점수 배열을 제자리(in-place)에서 정규화합니다. 핵심은 **데이터를 복사하지 않고 빌려서** 처리하는 것입니다.

```rust
use std::collections::HashMap;

/// 텍스트를 '빌려서' 단어 빈도를 집계한다.
/// 반환되는 HashMap의 키는 &str 슬라이스 — 원본 text의 메모리를 그대로 가리킨다(zero-copy).
/// 'a 수명 표기는 "반환된 키들이 입력 text보다 오래 살 수 없다"는 의도를 컴파일러에 설명한다.
fn word_count<'a>(text: &'a str) -> HashMap<&'a str, usize> {
    let mut counts: HashMap<&str, usize> = HashMap::new();
    for word in text.split_whitespace() {
        // entry API: 키가 있으면 그 값을, 없으면 0을 삽입한 뒤 가변 참조를 돌려준다.
        *counts.entry(word).or_insert(0) += 1;
    }
    counts
}

/// 점수 슬라이스를 가변 대여하여 제자리에서 [0, 1] 범위로 정규화한다.
/// 새 Vec을 할당하지 않으므로 추가 힙 비용이 0이다.
fn normalize_scores(scores: &mut [f64]) {
    let max = scores.iter().cloned().fold(f64::MIN, f64::max);
    if max <= 0.0 {
        return;
    }
    for s in scores.iter_mut() {
        *s /= max; // 가변 참조 s를 역참조하여 제자리 수정
    }
}

fn main() {
    let log = "ERROR disk full ERROR timeout INFO ok ERROR disk full";

    // log를 불변 대여하여 집계. log의 소유권은 그대로 main에 남는다.
    let counts = word_count(log);

    // 빈도 내림차순, 동률이면 사전순으로 정렬해 출력
    let mut pairs: Vec<(&&str, &usize)> = counts.iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(a.1).then(a.0.cmp(b.0)));
    for (word, n) in pairs {
        println!("{:>8} : {}", word, n);
    }

    // log는 빌려주기만 했으므로 여전히 유효하다.
    println!("(원본 로그 길이: {} bytes)", log.len());

    let mut scores = vec![2.0, 4.0, 8.0, 1.0];
    normalize_scores(&mut scores); // 가변 대여
    println!("normalized: {:?}", scores); // [0.25, 0.5, 1.0, 0.125]
}
```

이 코드를 `cargo run` 하면 다음을 출력합니다.

```text
   ERROR : 3
    disk : 2
    full : 2
    INFO : 1
      ok : 1
 timeout : 1
(원본 로그 길이: 53 bytes)
normalized: [0.25, 0.5, 1.0, 0.125]
```

음미할 점이 세 가지 있습니다. 첫째, `word_count`의 반환 타입은 `HashMap<&'a str, usize>` — 키가 `String`이 아니라 `&str` 슬라이스입니다. 즉 53바이트짜리 로그(`"ERROR disk full ERROR timeout INFO ok ERROR disk full"`는 ASCII 53글자이므로 정확히 53바이트입니다)에서 단어를 집계하면서 단 한 글자도 복사하지 않았습니다. 모든 키는 원본 `log`의 부분 구간을 가리키는 16바이트 fat pointer일 뿐입니다(zero-copy 집계). 둘째, 그렇기 때문에 `'a` 수명 표기가 필요합니다 — "반환된 키들은 입력 `text`가 살아있는 동안만 유효하다"는 계약을 컴파일러에 명시한 것입니다([Chapter 6](chapter_06.md)에서 본격적으로 다룹니다). 셋째, `normalize_scores`는 `&mut [f64]`로 슬라이스를 가변 대여해 새 벡터 할당 없이 제자리에서 수정합니다. 파이썬이라면 `[s / max for s in scores]`로 새 리스트를 할당하기 쉬운 지점입니다.

> **벤치마크 맥락:** 1천만 단어짜리 로그에서 zero-copy `&str` 집계는 단어를 `String`으로 복제해 집계하는 방식 대비 힙 할당이 사실상 0에 수렴합니다(키 복사 제거). `--release` 빌드 기준, 키를 `.to_string()`으로 복제하는 변형은 수백만 번의 작은 힙 할당으로 느려집니다. 정확한 배수는 단어 길이 분포와 할당기(allocator)에 의존하므로, 자신의 데이터로 `cargo bench`(criterion 크레이트)를 돌려 확인하는 것을 권합니다. 핵심은 "복사를 빌림으로 바꾸면 할당이 사라진다"는 구조적 이점입니다.

### 벤치마크: 제로카피 `&str` 키 vs `String` 키 (criterion, 측정)

말로 "할당이 사라진다"고 하는 것과 측정으로 보여주는 것은 다릅니다. 직접 criterion으로 재 봅시다. 두 함수는 키 타입만 다릅니다.

```rust
use std::collections::HashMap;

// 제로카피: 키가 입력 text의 부분 슬라이스(&str)를 그대로 가리킨다. 힙 할당 없음.
pub fn count_borrowed(text: &str) -> HashMap<&str, usize> {
    let mut counts: HashMap<&str, usize> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;
    }
    counts
}

// 소유: 새 키마다 String을 힙에 새로 할당(to_string)한다.
pub fn count_owned(text: &str) -> HashMap<String, usize> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_string()).or_insert(0) += 1;
    }
    counts
}
```

`benches/wc.rs` (criterion harness):

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use word_count_bench::{count_borrowed, count_owned};

fn bench(c: &mut Criterion) {
    let log = make_log(1_000_000); // 단어 100만 개로 구성한 로그
    let mut g = c.benchmark_group("word_count");
    g.bench_function("borrowed_&str", |b| b.iter(|| count_borrowed(black_box(&log))));
    g.bench_function("owned_String", |b| b.iter(|| count_owned(black_box(&log))));
    g.finish();
}

criterion_group!(benches, bench);
criterion_main!(benches);
```

`Cargo.toml`에 criterion을 `dev-dependencies`로 추가하고 `harness = false`로 벤치 타깃을 등록한 뒤 `cargo bench`를 돌린 실측 결과입니다(로컬: rustc 1.90.0, Apple Silicon, `--release`).

```text
word_count/borrowed_&str   time:   [12.40 ms 12.99 ms 13.26 ms]
word_count/owned_String    time:   [25.33 ms 25.62 ms 28.08 ms]
```

같은 100만 단어 입력에서 `String` 키 버전이 약 **2배** 느립니다. 차이의 전부는 키 복제 비용입니다 — `count_owned`는 새 키를 만날 때마다 짧은 문자열 하나를 힙에 할당하고, 맵을 버릴 때 그만큼을 해제합니다. `count_borrowed`는 키가 입력 버퍼의 16바이트 fat pointer일 뿐이라 그 할당 자체가 존재하지 않습니다.

> **💡 정직한 단서:** 이 2배는 "고유 단어가 적은" 로그(반복되는 토큰)에서 나온 수치입니다. 고유 토큰을 5만 종으로 늘려(`tok0`..`tok49999`) 다시 재면 17.8 ms 대 31.0 ms, 약 **1.7배**로 줄어듭니다 — 고유 키가 많아질수록 양쪽 모두 해시 삽입 비용이 커져 복제 비용의 상대적 비중이 옅어지기 때문입니다. 그래서 정직한 표현은 "복제 키는 워크로드에 따라 약 1.7~2배 느리다"입니다. 마케팅용 "10배"가 아니라, **할당을 빌림으로 바꾸면 일정한 구조적 비용이 사라진다**는 것이 핵심입니다. 단, `&str` 키는 입력 버퍼가 살아있는 동안만 유효하므로(수명, [Chapter 6](chapter_06.md)), 맵이 입력보다 오래 살아야 한다면 `String` 키가 불가피합니다 — 그때는 이 2배가 정당한 대가입니다.

---

## 5.9 Best Practices & 안티패턴

실무에서 대여를 다룰 때의 체크리스트입니다. clippy(Rust 린터)가 잡아주는 항목도 함께 표시합니다.

**이렇게 하세요.**

- **함수 인자는 가장 약한 권한으로 받으세요.** 읽기만 하면 `&T`, 수정하면 `&mut T`, 소유권을 정말 가져가야 할 때만 `T`. 권한을 최소화할수록 호출자의 유연성이 커집니다.
- **컬렉션 인자는 슬라이스로 받으세요.** `&Vec<i32>` 대신 `&[i32]`, `&String` 대신 `&str`. clippy의 `clippy::ptr_arg` 린트가 정확히 이것을 권고합니다. 슬라이스를 받으면 벡터·배열·다른 슬라이스를 모두 받을 수 있습니다.
- **`.clone()`을 쓰기 전에 "참조로 충분한가?"를 물으세요.** 빌림으로 해결되는데 복제하는 것은 GC를 피하려 Rust에 온 이유를 배신하는 일입니다.
- **NLL을 믿으세요.** 충돌을 피하려고 인위적으로 블록 `{ }`을 쪼개기 전에, 그냥 참조를 마지막에 쓰고 끝내면 컴파일러가 알아서 대여를 종료시킵니다.
- **하나의 큰 구조체를 여러 가변 슬라이스로 나눠야 하면 `split_at_mut`을 쓰세요.** (5.10절 참고)

**이건 피하세요.**

- **불필요한 `.clone()` 남발** — "빌림 검사기와 싸우기 싫어서" 일단 복제하는 습관. 단기 처방이지만 성능과 코드 품질을 갉아먹습니다.
- **`&mut`를 기본값처럼 쓰기** — 읽기만 하는데 `&mut`로 받으면, 호출자가 그 데이터에 다른 참조를 만들 수 없게 되어 불필요하게 경직됩니다.
- **참조를 반환하면서 수명을 생각하지 않기** — 함수가 참조를 받아 참조를 반환하면 거의 항상 수명 표기가 필요합니다. 컴파일러가 요구할 때 당황하지 마세요([Chapter 6](chapter_06.md)).
- **`&Box<T>`, `&Rc<T>` 같은 이중 간접 참조** — clippy의 `clippy::borrowed_box`가 경고합니다. 대개 `&T`로 충분합니다.

---

## 5.10 흔한 함정 (Python 개발자가 자주 하는 실수)

파이썬의 자유로운 별칭-변형 습관이 Rust에서 어떻게 컴파일 에러로 번역되는지, 가장 흔한 세 가지 패턴과 그 탈출법입니다. 이것이 입문자가 "빌림 검사기와 싸운다(fighting the borrow checker)"고 부르는 바로 그 전투입니다.

### 함정 ① 순회 중 컬렉션 수정 (Iterator Invalidation)

5.1절에서 파이썬이 조용히 망가뜨리던 그 코드입니다. Rust로 옮기면:

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];

    for n in &numbers {           // numbers를 불변 대여하여 순회
        if *n % 2 == 0 {
            numbers.push(*n * 10); // 🚨 순회 중 가변 대여 시도
        }
    }
}
```

```text
error[E0502]: cannot borrow `numbers` as mutable because it is also borrowed as immutable
 --> src/main.rs:7:13
  |
5 |     for n in &numbers {
  |              --------
  |              |
  |              immutable borrow occurs here
  |              immutable borrow later used here
7 |             numbers.push(*n * 10); // 순회 중 컬렉션 수정!
  |             ^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

`for n in &numbers`가 `numbers`를 불변 대여하는 동안, 그 안에서 `numbers.push()`가 가변 대여를 시도하니 XOR 규칙 위반입니다. 파이썬이 런타임에(그것도 운 좋아야) 잡던 버그를 Rust는 **컴파일 타임에 단호히 거부**합니다.

**탈출법:** 수정할 원소를 먼저 모으고 순회가 끝난 뒤에 반영합니다. 순회의 불변 대여와 수정의 가변 대여를 시간적으로 분리하는 것입니다.

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];

    let to_add: Vec<i32> = numbers
        .iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * 10)
        .collect(); // 여기서 numbers의 불변 대여가 끝남

    numbers.extend(to_add); // 이제 가변 대여 OK
    println!("{:?}", numbers); // [1, 2, 3, 4, 5, 20, 40]
}
```

### 함정 ② `&mut self` 메서드 안에서 `&self` 메서드 호출

객체지향 습관에서 가장 자주 부딪히는 벽입니다. 메서드 안에서 다른 메서드를 부르는 것은 파이썬에선 당연하지만, Rust에선 대여가 충돌할 수 있습니다.

```rust
struct Cache {
    data: Vec<i32>,
}

impl Cache {
    fn largest(&self) -> i32 {
        *self.data.iter().max().unwrap()
    }

    fn trim_to_largest(&mut self) {
        for x in self.data.iter_mut() {  // self.data를 가변 대여
            *x = self.largest();         // 🚨 동시에 self를 불변 대여하는 메서드 호출
        }
    }
}

fn main() {
    let mut c = Cache { data: vec![3, 1, 4, 1, 5] };
    c.trim_to_largest();
    println!("{:?}", c.data);
}
```

```text
error[E0502]: cannot borrow `*self` as immutable because it is also borrowed as mutable
  --> src/main.rs:13:18
   |
12 |         for x in self.data.iter_mut() {
   |                  --------------------
   |                  |
   |                  mutable borrow occurs here
   |                  mutable borrow later used here
13 |             *x = self.largest(); // 🚨
   |                  ^^^^ immutable borrow occurs here
```

`self.data.iter_mut()`가 `self`(정확히는 `self.data`)를 가변 대여 중인데, `self.largest()`가 `&self`로 다시 빌리려 하니 충돌입니다.

**탈출법:** 필요한 값을 루프 *전에* 미리 계산해 지역 변수에 담습니다. 그러면 루프 안에서 `self`를 다시 빌릴 일이 없어집니다.

```rust
fn trim_to_largest(&mut self) {
    let largest = self.largest(); // 루프 전에 미리 계산 (불변 대여 종료)
    for x in self.data.iter_mut() {
        *x = largest; // 이제 self를 다시 빌리지 않음
    }
}
```

### 함정 ③ 같은 벡터의 서로 다른 두 원소를 동시에 가변 대여

두 원소를 맞바꾸거나 한 원소를 다른 원소로 갱신하려고 인덱싱으로 두 가변 참조를 만들면 막힙니다.

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4];
    let a = &mut v[0];
    let b = &mut v[3]; // 🚨 컴파일러는 0번과 3번이 다른 원소임을 증명할 수 없음
    *a += *b;
}
```

컴파일러는 `v[0]`과 `v[3]`이 실제로는 겹치지 않는다는 것을 인덱스만 보고 증명할 수 없습니다(인덱스가 런타임 값일 수도 있으니까요). 그래서 "벡터 전체를 두 번 가변 대여"하는 것으로 보고 거부합니다.

**탈출법:** 표준 라이브러리가 이 정확한 상황을 위한 안전한 도구를 제공합니다. `split_at_mut`은 슬라이스를 겹치지 않는 두 가변 슬라이스로 쪼개 줍니다 — 컴파일러가 두 부분의 비중첩(disjoint)을 보장합니다.

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];

    // 인덱스 3을 기준으로 [0..3]과 [3..6] 두 개의 겹치지 않는 가변 슬라이스로 분리
    let (left, right) = v.split_at_mut(3);
    left[0] = 100;
    right[0] = 200;

    println!("{:?}", v); // [100, 2, 3, 200, 5, 6]
}
```

단순 교환이라면 `v.swap(i, j)`, 한 원소만 빼서 바꾸려면 `std::mem::swap`이나 `std::mem::replace`가 있습니다. **빌림 검사기와 싸우기보다, 표준 라이브러리가 이미 안전한 우회로를 마련해 두었는지 먼저 찾는 것**이 숙련된 Rustacean의 자세입니다.

### 함정 ④ `String + &str`에서 좌변이 move된다 (E0382)

대여를 배운 직후 가장 의외의 곳에서 발을 헛디딥니다. 바로 문자열 이어붙이기입니다. 파이썬에서 `s3 = s1 + s2`는 새 문자열을 만들 뿐 `s1`, `s2`를 멀쩡히 남겨둡니다. Rust의 `+`도 새 `String`을 만들지만, 그 과정에서 **좌변의 소유권을 가져가(move) 버립니다.**

```rust
fn main() {
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = s1 + &s2;   // s1이 여기서 move됨 (우변은 &str로 빌림)
    println!("{}", s3);
    println!("{}", s1);  // 🚨 move된 s1을 다시 사용
}
```

```text
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:6:20
  |
2 |     let s1 = String::from("Hello, ");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
4 |     let s3 = s1 + &s2;   // s1이 여기서 move됨
  |              -- value moved here
...
6 |     println!("{}", s1);  // move된 s1을 다시 사용
  |                    ^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
4 |     let s3 = s1.clone() + &s2;
  |                ++++++++
```

왜 좌변만 move되고 우변(`&s2`)은 빌림일까요? `+` 연산자가 호출하는 메서드의 시그니처를 보면 답이 나옵니다. `String`의 덧셈은 `impl Add<&str> for String`으로 정의되어 있고, 그 메서드는 `fn add(self, other: &str) -> String`입니다 — 좌변을 `self`로 **소유권째 받아** 그 내부 버퍼를 재사용해 결과를 만들고, 우변은 `&str`로 빌리기만 합니다. 즉 `s1 + &s2`는 사실상 `Add::add(s1, &s2)`이고, `s1`은 첫 인자 `self`로 이동합니다. 이는 비효율을 피하기 위한 의도된 설계입니다 — 좌변의 힙 버퍼를 새로 할당하지 않고 그대로 키워서 재사용하니까요. 대가로 좌변은 소비됩니다.

**탈출법은 두 가지입니다.** 좌변을 정말 살려둬야 한다면 `.clone()`으로 복제본을 넘기거나(컴파일러가 친절히 제안하는 그것), 애초에 양쪽을 모두 빌리는 `format!`을 씁니다.

```rust
fn main() {
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");

    let a = s1.clone() + &s2;   // s1을 복제해 넘김 -> 원본 s1 보존
    let b = format!("{s1}{s2}"); // 둘 다 빌림. move 없음. 다회 결합에 가독성도 좋음
    println!("{a} | {b} | {s1}"); // Hello, world! | Hello, world! | Hello,
}
```

실무에서는 두세 조각을 넘기면 `+` 체인보다 `format!`(또는 `String::with_capacity` + `push_str`)이 더 읽기 쉽고, 좌변 move의 함정도 자연스럽게 피합니다. `+`는 "왼쪽 문자열을 더 이상 안 쓰고 그 버퍼를 재활용하고 싶을 때"만 쓰는 최적화 도구로 생각하세요.

---

## 연습문제

Rustlings 스타일로, 컴파일러를 직접 마주하며 손으로 풀어 보세요. 각 문항은 가설 → 컴파일/측정으로 자기검증하는 흐름입니다. 정답 스케치는 핵심 아이디어와 함께, 스스로 확인할 수 있도록 **컴파일러 에러코드** 또는 **측정값**을 함께 적었습니다.

### 문제 1 — 불변/가변 대여 규칙과 NLL

다음 코드는 컴파일되지 않습니다. **한 줄도 지우지 말고**, 줄 순서만 바꿔서 컴파일되게 만드세요. 그리고 "왜 순서를 바꾸면 통과하는지"를 NLL로 설명해 보세요.

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let m = &mut data;     // 가변 참조
    let r = &data;         // 불변 참조
    m.push(4);
    println!("{:?}", r);
}
```

> **정답 스케치.** 현재 코드는 가변 대여 `m`이 살아있는 동안(마지막 사용은 `m.push(4)`) 불변 대여 `r`을 만들어 XOR 규칙을 어깁니다 — 그대로 컴파일하면 `error[E0502]`가 납니다. 핵심 아이디어는 **각 대여를 "마지막 사용까지"로 끝내고 시간적으로 분리**하는 것입니다(NLL). `m`을 만들고 `m.push(4)`까지 끝낸 *뒤에* `r`을 만들면, NLL이 `m`의 대여를 `push` 지점에서 종료시키므로 충돌하지 않습니다.
>
> ```rust
> fn main() {
>     let mut data = vec![1, 2, 3];
>     let m = &mut data;     // 가변 대여 시작
>     m.push(4);             // m의 마지막 사용 -> NLL이 여기서 가변 대여 종료
>     let r = &data;         // 이제 불변 대여 OK
>     println!("{:?}", r);   // [1, 2, 3, 4]
> }
> ```
>
> 자기검증: 원본은 `error[E0502]`, 재배치 후에는 컴파일·실행되어 `[1, 2, 3, 4]`를 출력합니다.

### 문제 2 — 재대여(Reborrow)

다음 함수는 가변 참조 `v`를 두 함수에 연달아 넘깁니다. 가변 참조는 `Copy`가 아닌데 어떻게 두 번 넘길 수 있을까요? 그리고 `consume(v); consume(v);`처럼 **소유권을 가져가는** 함수였다면 왜 막힐지 예상해 보세요.

```rust
fn touch(v: &mut Vec<i32>) { v.push(0); }

fn apply_twice(v: &mut Vec<i32>) {
    touch(v);   // (1)
    touch(v);   // (2) — 여기서 왜 v가 아직 살아있을까?
}

fn main() {
    let mut v = vec![];
    apply_twice(&mut v);
    println!("{:?}", v);
}
```

> **정답 스케치.** `touch(v)`는 `v`를 move하지 않습니다. 컴파일러가 암묵적으로 `touch(&mut *v)`로 해석해 **기존 가변 참조에서 임시 가변 참조를 떠내는(reborrow)** 것이기 때문입니다. 호출이 끝나면 떠낸 참조가 사라지고 `v`가 다시 유일한 가변 참조로 복귀하므로, (2)에서 또 빌릴 수 있습니다. 그래서 이 코드는 컴파일되어 `[0, 0]`을 출력합니다.
>
> 반면 `fn consume(v: &mut Vec<i32>)`가 아니라 소유권을 받는 함수(예: `Box<Vec<i32>>`를 `self`로 받는 형태)였다면 첫 호출에서 값이 move되어 두 번째 호출은 `error[E0382]: use of moved value`가 났을 것입니다. 재대여는 **참조에 한해** 작동하는 마법입니다.
>
> 자기검증: 위 코드는 정상 컴파일·출력 `[0, 0]`. 재대여를 막고 싶다면 `let r = &mut v; let _ = { r };`처럼 명시적으로 move시켜 본 뒤 `r`을 다시 쓰면 `error[E0382]`를 직접 관찰할 수 있습니다.

### 문제 3 — 같은 벡터의 두 원소 동시 수정

벡터의 첫 원소에 마지막 원소 값을 더하고 싶습니다. 아래는 컴파일되지 않습니다. (a) 에러코드를 예상하고, (b) 인덱싱으로 두 `&mut`를 만들지 않으면서 고치세요.

```rust
fn main() {
    let mut v = vec![10, 20, 30, 40];
    let first = &mut v[0];
    let last = &mut v[3];   // ?
    *first += *last;
    println!("{:?}", v);
}
```

> **정답 스케치.** (a) 컴파일러는 `v[0]`과 `v[3]`이 겹치지 않음을 인덱스만으로 증명할 수 없어 "`v`를 두 번 가변 대여"로 보고 `error[E0499]: cannot borrow `v` as mutable more than once at a time`를 냅니다. (b) 표준 라이브러리의 `split_at_mut`으로 슬라이스를 **겹치지 않는 두 가변 슬라이스**로 쪼개면 컴파일러가 비중첩(disjoint)을 보장합니다.
>
> ```rust
> fn main() {
>     let mut v = vec![10, 20, 30, 40];
>     let (left, right) = v.split_at_mut(3); // [0..3], [3..4]
>     left[0] += right[0];
>     println!("{:?}", v); // [50, 20, 30, 40]
> }
> ```
>
> (단순 교환이라면 `v.swap(0, 3)` 한 줄로 끝납니다.) 자기검증: 원본은 `error[E0499]`, `split_at_mut` 버전은 `[50, 20, 30, 40]`을 출력합니다.

---

지금까지 우리는 [Chapter 4](chapter_04.md)의 소유권에서 한 걸음 더 나아가, 소유권을 빼앗지 않고 데이터를 빌리는 **대여(Borrowing)**를 배웠습니다. 핵심은 단 하나의 XOR 규칙 — "불변 참조 여럿이거나 가변 참조 하나거나" — 이고, 이 규칙이 데이터 경쟁과 메모리 오염을 *런타임 비용 0으로* 컴파일 타임에 박멸한다는 것이었습니다. 참조는 8바이트 얇은 포인터로, 슬라이스는 16바이트 fat pointer로 표현되며, `copy.deepcopy`의 막대한 비용 없이 데이터를 무비용으로 공유합니다.

하지만 한 가지 질문이 남았습니다. 컴파일러는 "이 참조가 원본보다 오래 살아남지 않는다"는 것을 *어떻게* 보장할까요? `word_count` 함수에 등장한 `'a` 표기는 정확히 무엇이며, 언제 우리가 직접 그것을 써야 할까요? 5.5절에서 NLL을 이야기하며 "참조의 수명은 마지막 사용까지"라고 했던 그 **수명(Lifetime)**의 정체를, 다음 [Chapter 6](chapter_06.md)에서 정면으로 마주합니다. 파이썬에는 개념조차 없는, 그러나 Rust의 메모리 안전을 완성하는 마지막 퍼즐 조각입니다.

---

## References

- https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html
- https://doc.rust-lang.org/book/ch04-03-slices.html
- https://doc.rust-lang.org/rust-by-example/scope/borrow.html
- https://doc.rust-lang.org/error_codes/E0502.html
- https://doc.rust-lang.org/error_codes/E0499.html
