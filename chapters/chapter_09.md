# Chapter 9. 매직 메서드(Dunder)에서 Trait으로

파이썬 클래스의 강력함은 속칭 **'던더(Dunder)'**라고 불리는 매직 메서드(Magic Method)에서 나옵니다. 어떤 객체든 `__str__`을 구현하면 `print()`로 출력할 수 있고, `__len__`을 구현하면 `len()` 함수를 쓸 수 있으며, `__add__`를 구현하면 `+` 기호로 더할 수 있습니다. 이것이 파이썬의 핵심 철학인 **덕 타이핑(Duck Typing)**의 근간입니다.

"오리처럼 걷고 오리처럼 꽥꽥거리면, 그것은 오리다." 파이썬은 객체의 '실제 타입'이 무엇인지 묻지 않습니다. 그저 런타임에 해당 매직 메서드가 존재하는지만 검사합니다. 이 자유로움은 `numpy`의 `ndarray`가 `+`, `*`, `@` 같은 연산자를 자연스럽게 받아들이고, `pandas`의 `DataFrame`이 `df['col']`로 인덱싱되며, `pathlib.Path`가 `/` 연산자로 경로를 합치는 풍부한 API를 만들어냈습니다.

하지만 10년 차 파이썬 엔지니어라면 그 대가를 뼈저리게 알고 있을 겁니다. `obj + other`라고 쓴 순간, 파이썬은 `type(obj).__add__`를 찾고, 없으면 `type(other).__radd__`를 찾고, 그것도 실패하면 `TypeError: unsupported operand type(s)`를 **런타임에** 던집니다. `__hash__`를 깜빡 잊고 `__eq__`만 정의하면 객체가 조용히 unhashable이 되어 `dict` 키로 못 쓰게 됩니다. `__lt__`만 정의하고 `__le__`를 빠뜨리면 정렬은 되는데 `<=` 비교에서 터집니다. 이 모든 함정이 **테스트가 그 코드 경로를 밟기 전까지는 드러나지 않습니다.**

Rust는 이 덕 타이핑의 표현력을 거의 그대로 유지하면서, 그 모든 검사를 **컴파일 타임**으로 끌어옵니다. 무기는 **트레이트(Trait)**입니다. 파이썬이 "메서드가 런타임에 존재하는가?"를 묻는다면, Rust는 "이 타입이 이 트레이트를 구현했음을 컴파일러가 증명할 수 있는가?"를 묻습니다. 그리고 그 증명은 **0의 런타임 비용**으로 끝납니다.

이번 챕터에서는 Chapter 8에서 구현한 `EmailAddress` 뉴타입과 컴포지션 사고방식을 이어받아, 파이썬의 던더 메서드를 Rust 표준 트레이트로 1:1 매핑합니다. 기존 `Vector2D` 예제를 `Display`, `Add`, `Sub`, `Neg`, `Mul<f64>`, `Index`, `PartialEq`, `From<(f64, f64)>`까지 단계적으로 완성하고, 그 과정에서 정적 디스패치(단일형화)와 동적 디스패치(vtable)의 메모리·성능 트레이드오프를 바이트 단위와 벤치마크 수치로 해부하겠습니다.

---

## 9.1 The Python Way — 던더 메서드의 자유와 그 청구서

수학 벡터를 모델링하며 시작합시다. 파이썬에서는 매직 메서드를 클래스 본문에 욱여넣습니다.

```python
from __future__ import annotations
from typing import Union

class Vector2D:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __repr__(self) -> str:            # 디버깅용
        return f"Vector2D(x={self.x}, y={self.y})"

    def __str__(self) -> str:             # 사람용
        return f"({self.x}, {self.y})"

    def __add__(self, other: Vector2D) -> Vector2D:
        if not isinstance(other, Vector2D):
            return NotImplemented         # 런타임 협상
        return Vector2D(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar: float) -> Vector2D:
        return Vector2D(self.x * scalar, self.y * scalar)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector2D):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __getitem__(self, idx: int) -> float:
        return (self.x, self.y)[idx]

v1 = Vector2D(1.0, 2.0)
v2 = Vector2D(3.0, 4.0)
print(v1 + v2)        # (4.0, 6.0)
print(v1 * 2.0)       # (2.0, 4.0)
print(v1[0])          # 1.0
```

겉보기엔 깔끔합니다. 하지만 여기엔 시니어 엔지니어가 알아야 할 세 가지 청구서가 숨어 있습니다.

**첫째, 런타임 디스패치 비용.** `v1 + v2`를 평가할 때 CPython은 `type(v1)`의 타입 객체에서 `tp_as_number->nb_add` 슬롯을 조회하고, 그 슬롯이 채워져 있으면 `__add__`를 호출합니다. 결과가 `NotImplemented`이면 `v2`의 `__radd__`로 폴백합니다. 이 일련의 슬롯 조회·디스패치는 **매 연산마다** 일어납니다. 1억 개 벡터를 더하는 시뮬레이션이라면 1억 번의 딕셔너리성 조회가 누적됩니다.

**둘째, 비대칭 함정.** `__eq__`를 재정의하면 파이썬은 `__hash__`를 자동으로 `None`으로 설정합니다. 위 클래스의 인스턴스를 `set`에 넣거나 `dict` 키로 쓰려는 순간 `TypeError: unhashable type: 'Vector2D'`가 터집니다. `__lt__`는 정의했는데 `__le__`, `__gt__`, `__ge__`를 빠뜨리면 일부 비교만 동작합니다. (`functools.total_ordering` 데코레이터가 이 문제를 완화하지만, 그것을 붙이는 것 자체를 잊을 수 있습니다.)

**셋째, 타입 안전성의 부재.** `v1 + "hello"`는 `__add__`가 `NotImplemented`를 반환하고, `str`에도 적절한 핸들러가 없어 결국 `TypeError`를 던집니다. 하지만 이 에러는 **그 코드가 실행될 때** 발생합니다. CI에서 그 분기를 안 밟으면 프로덕션에서 터집니다.

Rust는 이 세 청구서를 모두 컴파일 타임으로 옮깁니다. 어떻게 사고방식을 바꾸는지 봅시다.

---

## 9.2 The Rust Mental Model — 데이터와 행동의 분리, 그리고 명시적 약속

파이썬에서 "행동"은 클래스 본문에 정의된 메서드입니다. 데이터(`self.x`)와 행동(`__add__`)이 한 덩어리(class block) 안에 섞여 있습니다. Rust는 이 둘을 물리적으로 떼어놓습니다.

```rust
// 데이터: 무엇을 담는가
pub struct Vector2D {
    pub x: f64,
    pub y: f64,
}

// 행동: 무엇을 할 수 있는가 (별도의 impl 블록)
impl std::ops::Add for Vector2D {
    // ...
}
```

이 분리가 핵심 멘탈 모델입니다. **트레이트는 "타입이 따라야 할 행동의 계약"**이고, `impl Trait for Type`은 "이 타입이 그 계약을 이행한다는 서명"입니다. 파이썬의 던더 메서드가 "관습적 이름의 일반 메서드"라면, Rust의 트레이트 메서드는 **표준 라이브러리가 정의한 인터페이스에 대한 명시적 구현**입니다.

이 설계가 주는 직접적 이득은 다음과 같습니다.

- `+` 연산자를 쓰려면 `Add` 트레이트가 구현되어 있어야 하고, **컴파일러가 그것을 강제**합니다. 안 했으면 `error[E0369]`가 즉시 뜹니다.
- 한 타입이 어떤 트레이트를 구현했는지는 정적으로 결정되므로, 런타임 슬롯 조회가 사라집니다. 호출은 직접 함수 호출로 컴파일됩니다.
- 트레이트 간 의존 관계(예: `Eq`는 `PartialEq`를 요구)가 타입 시스템에 박혀 있어, 비대칭 함정이 컴파일 에러로 드러납니다.

아래는 파이썬 던더 메서드와 Rust 표준 트레이트의 **확장된 매핑 비교표**입니다. 이 표는 챕터 전체의 지도이니, 앞으로 새 던더를 만날 때마다 여기로 돌아오세요.

| 역할 | 파이썬 매직 메서드 | Rust 표준 Trait | 핵심 차이 |
| :--- | :--- | :--- | :--- |
| 사람용 출력 | `__str__` | `std::fmt::Display` | Rust는 수동 구현만 가능(derive 불가) |
| 디버깅 출력 | `__repr__` | `std::fmt::Debug` | `#[derive(Debug)]`로 자동 생성 |
| 동등성 | `__eq__`, `__ne__` | `std::cmp::PartialEq` | `__ne__`는 `eq`에서 자동 유도 |
| 완전 동등성 | (개념 없음) | `std::cmp::Eq` | 반사성 보장 표식, 메서드 없음 |
| 대소 비교 | `__lt__`,`__le__`,`__gt__`,`__ge__` | `std::cmp::PartialOrd` | 단 하나 `partial_cmp`로 4개 연산 유도 |
| 전순서 | (개념 없음) | `std::cmp::Ord` | `cmp`로 `sort()` 가능 |
| 해시 | `__hash__` | `std::hash::Hash` | `Eq`와 정합성을 타입이 강제 |
| 덧셈/곱셈/부정 | `__add__`,`__mul__`,`__neg__` | `std::ops::{Add, Mul, Neg}` | `Output` 연관 타입으로 결과 타입 명시 |
| 인덱싱 | `__getitem__` | `std::ops::Index` | 참조(`&T`) 반환, 음수 인덱스 없음 |
| 인덱스 대입 | `__setitem__` | `std::ops::IndexMut` | `&mut T` 반환 |
| 호출 가능 | `__call__` | `Fn`/`FnMut`/`FnOnce` | 캡처 방식에 따라 3종으로 분화 |
| 반복 | `__iter__`,`__next__` | `std::iter::Iterator` | Chapter 15에서 심화 |
| 길이 | `__len__` | (관습) `fn len(&self)` | 표준 트레이트 아님, 관용 메서드 |
| 컨텍스트 매니저 | `__enter__`,`__exit__` | `Drop` (RAII) | 스코프 종료 시 자동 호출, Chapter 4 연계 |
| 타입 변환 | `__int__`,`__float__`,`__str__` | `From`/`Into`/`TryFrom` | 실패 가능 변환은 `TryFrom`으로 분리 |
| 기본값 | (개념 없음, `__init__` 기본 인자) | `std::default::Default` | `#[derive(Default)]` 또는 수동 |

이 표에서 즉시 눈에 띄는 패턴이 있습니다. 파이썬에서 "메서드 하나"였던 것이 Rust에서는 **여러 트레이트로 쪼개져 있고**(`__eq__` → `PartialEq` + `Eq`, 비교 → `PartialOrd` + `Ord`), 실패 가능성에 따라 갈라집니다(`From` vs `TryFrom`). 이는 더 번거로워 보이지만, 정확히 그 분리가 부동소수점 같은 까다로운 타입을 안전하게 다루게 해줍니다. 그 이유는 9.5에서 `f64`로 직접 확인하겠습니다.

이제 표를 따라 `Vector2D`를 출판 품질로 완성해 봅시다.

---

## 9.3 `__str__`/`__repr__`을 `Display`와 `Debug`로

파이썬은 한 객체에 대해 두 종류의 문자열 표현을 둡니다. `__repr__`은 개발자용(이상적으로는 `eval()`로 객체를 복원할 수 있는 형태)이고, `__str__`은 사용자용입니다. Rust도 정확히 이 이분법을 가집니다. `Debug`(`{:?}`)는 개발자용, `Display`(`{}`)는 사용자용입니다.

결정적 차이: **`Debug`는 자동 생성할 수 있지만 `Display`는 못 합니다.** Rust 표준 라이브러리 설계자들은 "사람에게 보여줄 문자열"은 도메인 지식이 필요한 결정이라 기계가 추측해선 안 된다고 봤습니다. 반면 디버그 출력은 필드를 그대로 찍으면 되므로 매크로가 대신합니다.

```rust
use std::fmt;

#[derive(Debug)] // __repr__ 자동 구현
pub struct Vector2D {
    pub x: f64,
    pub y: f64,
}

// __str__ 역할: 반드시 수동 구현
impl fmt::Display for Vector2D {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let v = Vector2D { x: 3.0, y: 4.0 };
    println!("{}", v);   // Display:  (3, 4)
    println!("{:?}", v); // Debug:    Vector2D { x: 3.0, y: 4.0 }
    println!("{:#?}", v); // Pretty-Debug: 여러 줄로 들여쓰기
}
```

`#[derive(Debug)]`는 Chapter 8에서 본 컴포지션처럼, 필드를 재귀적으로 순회하며 각 필드의 `Debug` 구현을 호출합니다. 즉 **모든 필드가 `Debug`를 구현해야** `Vector2D`도 `Debug`를 derive할 수 있습니다. `f64`는 표준 라이브러리가 이미 구현해 두었으니 문제없습니다.

> **🚨 컴파일러 에러 시연: `Display` 미구현 (E0277)**
>
> `Display`를 구현하지 않은 채 `{}`로 출력하려 하면 어떻게 될까요? 파이썬이라면 객체의 기본 `__str__`(`<__main__.Point object at 0x...>`)이 조용히 호출됩니다. Rust는 거부합니다.
>
> ```rust
> struct Point { x: i32, y: i32 }
> fn main() {
>     let p = Point { x: 1, y: 2 };
>     println!("{}", p); // 🚨 Display 미구현
> }
> ```
>
> `rustc`가 내뱉는 실제 메시지:
>
> ```text
> error[E0277]: `Point` doesn't implement `std::fmt::Display`
>  --> src/main.rs:4:20
>   |
> 4 |     println!("{}", p);
>   |               --   ^ `Point` cannot be formatted with the default formatter
>   |
> help: the trait `std::fmt::Display` is not implemented for `Point`
>   = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
> ```
>
> 컴파일러는 단순히 거부만 하지 않습니다. "사용자용 표현(`Display`)이 없다면, 디버그용 표현(`{:?}`)을 써보라"고 **대안까지 제시**합니다. 파이썬에서 `repr(obj)`가 무의미한 `<object at 0x...>`를 토해내 디버깅을 방해했던 경험과 정반대입니다. 여기서 "포맷 가능 여부"는 타입 시스템의 일급 속성이고, 빠뜨림은 버그가 아니라 컴파일 에러입니다.

---

## 9.4 연산자 오버로딩: `__add__`·`__sub__`·`__neg__`·`__mul__`을 `std::ops`로

이제 벡터 산술을 완성합니다. 파이썬에서는 `__add__`, `__sub__`, `__neg__`, `__mul__`을 클래스에 추가하면 끝이지만, Rust에서는 각각 대응하는 `std::ops` 트레이트를 구현합니다. 핵심은 **`Output` 연관 타입**입니다. 파이썬의 던더는 무엇이든 반환할 수 있지만(런타임에 결정), Rust는 "이 연산의 결과 타입은 무엇인가"를 컴파일 타임에 못 박습니다.

먼저 derive를 정리합시다. 산술 연산은 값을 소비하므로 `Vector2D`를 매번 복사할 수 있으면 편합니다. `f64` 두 개짜리 구조체는 16바이트로 작고 포인터를 담지 않으므로 `Copy`를 derive해도 안전합니다.

```rust
use std::fmt;
use std::ops::{Add, Sub, Neg, Mul};

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Vector2D {
    pub x: f64,
    pub y: f64,
}

impl Vector2D {
    pub fn new(x: f64, y: f64) -> Self {
        Vector2D { x, y }
    }
    pub fn magnitude(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}

impl fmt::Display for Vector2D {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

// __add__
impl Add for Vector2D {
    type Output = Vector2D; // 결과 타입 명시
    fn add(self, rhs: Vector2D) -> Self::Output {
        Vector2D::new(self.x + rhs.x, self.y + rhs.y)
    }
}

// __sub__
impl Sub for Vector2D {
    type Output = Vector2D;
    fn sub(self, rhs: Vector2D) -> Self::Output {
        Vector2D::new(self.x - rhs.x, self.y - rhs.y)
    }
}

// __neg__ (단항 연산자 -)
impl Neg for Vector2D {
    type Output = Vector2D;
    fn neg(self) -> Self::Output {
        Vector2D::new(-self.x, -self.y)
    }
}

// __mul__: 벡터 * 스칼라(f64)
impl Mul<f64> for Vector2D {
    type Output = Vector2D;
    fn mul(self, scalar: f64) -> Self::Output {
        Vector2D::new(self.x * scalar, self.y * scalar)
    }
}

fn main() {
    let v1 = Vector2D::new(1.0, 2.0);
    let v2 = Vector2D::new(3.0, 4.0);

    println!("v1 + v2 = {}", v1 + v2); // (4, 6)
    println!("v2 - v1 = {}", v2 - v1); // (2, 2)
    println!("-v1     = {}", -v1);     // (-1, -2)
    println!("v1 * 2  = {}", v1 * 2.0);// (2, 4)
}
```

여기서 파이썬 개발자가 가장 자주 발을 헛디디는 지점이 `Mul<f64>`의 **제네릭 파라미터**입니다. 파이썬에서 `__mul__(self, other)`의 `other`는 무엇이든 될 수 있어, `isinstance` 분기로 "스칼라면 이렇게, 벡터면 저렇게"를 한 메서드에 몰아넣었습니다. Rust에서는 `Mul<f64>`(스칼라곱)와 `Mul<Vector2D>`(내적 또는 성분곱)를 **서로 다른 트레이트 구현**으로 분리합니다. 우변 타입마다 별도 `impl`이 존재하고, 컴파일러가 우변 타입을 보고 정확한 구현을 정적으로 고릅니다.

그런데 `2.0 * v1`(스칼라가 왼쪽)은 어떨까요? 이건 `f64`의 `Mul`이 우변에 `Vector2D`를 받아야 한다는 뜻입니다. 파이썬이라면 `__rmul__`로 해결했지만, Rust에는 `__rmul__`이 없습니다. 대신 **`f64`에 대해 `Mul<Vector2D>`를 구현**합니다.

```rust
// 스칼라 * 벡터: __rmul__ 역할
impl Mul<Vector2D> for f64 {
    type Output = Vector2D;
    fn mul(self, v: Vector2D) -> Self::Output {
        Vector2D::new(self * v.x, self * v.y)
    }
}
// 이제 2.0 * v1 도 동작
```

이게 가능한 이유는 **고아 규칙(Orphan Rule)** 때문입니다. `Mul`은 표준 라이브러리 트레이트이고 `f64`도 표준 타입이지만, `impl Mul<Vector2D> for f64`는 합법입니다. 왜냐하면 `Mul<Vector2D>`의 타입 파라미터 자리에 **우리 크레이트의 로컬 타입 `Vector2D`**가 등장하기 때문입니다. 고아 규칙의 정확한 작동은 9.7에서 다룹니다.

### `__getitem__`을 `Index`로

파이썬에서 `v[0]`은 `__getitem__(0)`을 호출합니다. Rust는 `std::ops::Index` 트레이트입니다. 결정적 차이는 **`Index::index`가 참조(`&Self::Output`)를 반환한다**는 점입니다. 이는 `v[0]`이 값의 복사가 아니라 내부 데이터에 대한 차용(borrow)이며, `&mut v[0] = ...` 같은 대입을 `IndexMut`로 분리할 수 있게 해줍니다.

```rust
use std::ops::Index;

impl Index<usize> for Vector2D {
    type Output = f64;
    fn index(&self, idx: usize) -> &Self::Output {
        match idx {
            0 => &self.x,
            1 => &self.y,
            n => panic!("Vector2D 인덱스 범위 초과: {n}"),
        }
    }
}
// v2[0] -> &self.x 를 역참조한 값
```

여기서 파이썬과의 두 가지 큰 차이가 드러납니다. 첫째, **음수 인덱스가 없습니다.** 파이썬의 `v[-1]`은 `__getitem__(-1)`을 호출하고 시퀀스가 알아서 끝에서부터 셉니다. Rust의 인덱스 타입은 보통 `usize`(부호 없음)라 `-1`은 애초에 표현 불가입니다. 둘째, **범위를 벗어나면 패닉**입니다. `Index`는 `Option`을 반환하지 않으므로(반환 타입이 `&Self::Output`으로 고정), 잘못된 인덱스는 패닉으로 처리합니다. 실패를 값으로 다루고 싶다면 `Index`가 아니라 `fn get(&self, i: usize) -> Option<&f64>` 같은 명시적 메서드를 따로 두는 것이 관용적입니다(`Vec::get`이 정확히 이 패턴입니다).

### `From<(f64, f64)>`로 생성을 간결하게

마지막 조각은 변환입니다. 파이썬에서 `Vector2D(*tuple)`처럼 튜플을 풀어 생성하던 패턴을 Rust는 `From` 트레이트로 표현합니다.

```rust
impl From<(f64, f64)> for Vector2D {
    fn from(t: (f64, f64)) -> Self {
        Vector2D::new(t.0, t.1)
    }
}

fn main() {
    let v3: Vector2D = (5.0, 6.0).into();   // Into 자동 제공
    let v4 = Vector2D::from((7.0, 8.0));    // From 직접 호출
    let _ = (v3, v4);
}
```

`From`을 구현하면 `Into`는 **표준 라이브러리가 공짜로 제공**합니다(`impl<T, U: From<T>> Into<U> for T`라는 포괄 구현, blanket impl 덕분). 그래서 우리는 `From`만 구현하고 `into()`도 함께 얻습니다. `From`/`Into`의 실무 가치는 9.6에서 깊이 파헤칩니다.

> **Under the Hood: `Vector2D`의 메모리 레이아웃과 무비용 연산자**
>
> `#[derive(Copy)]`를 붙인 `Vector2D`는 메모리에서 정확히 `[f64; 2]`와 동일합니다. 16바이트(8바이트 × 2)가 연속으로 놓이고, 정렬(alignment)은 8바이트입니다. 포인터도, 길이 필드도, 가상 메서드 테이블도 없습니다. 파이썬의 `Vector2D` 인스턴스가 PyObject 헤더(참조 카운트 + 타입 포인터, 64비트에서 최소 16바이트) + `__dict__`(속성 딕셔너리, 보통 수십~수백 바이트) + 박싱된 `float` 두 개(각각 별도 힙 할당된 PyObject)로 흩어져 있는 것과 대조적입니다.
>
> 더 중요한 것은 연산의 컴파일 결과입니다. `v1 + v2`는 `--release` 빌드에서 `Add::add` 함수 호출조차 남지 않습니다. 컴파일러가 함수를 인라인하고 두 번의 `addsd`(SSE2 배정밀도 덧셈) 명령으로 줄여버립니다. 즉 `v1 + v2`의 어셈블리는 손으로 쓴 `self.x + rhs.x; self.y + rhs.y`와 **바이트 단위로 동일**합니다. 이것이 무비용 추상화(Zero-cost Abstraction)의 실체입니다. 연산자 오버로딩이라는 고수준 문법을 쓰지만, 그 추상화의 대가가 런타임에 0입니다. CPython이 `v1 + v2`마다 `nb_add` 슬롯을 조회하고 새 PyObject를 힙에 할당하는 것과 비교하면, 같은 표기법이지만 완전히 다른 기계가 돌아갑니다.

---

## 9.5 `#[derive]`의 힘과 한계: `PartialEq`/`Eq`/`Hash`/`Ord`

파이썬의 `dataclasses`는 `@dataclass(eq=True, order=True, frozen=True)`로 `__eq__`, `__lt__`, `__hash__`를 자동 생성합니다. Rust의 `#[derive]`가 정확히 대응합니다. 차이는 Rust가 **트레이트 간 정합성을 타입 시스템으로 강제**한다는 점입니다.

가장 흔히 derive하는 트레이트와 파이썬 대응:

```rust
#[derive(Debug, Clone, Default, PartialEq, Eq, Hash, PartialOrd, Ord)]
struct UserId {
    id: u64,
    region: String,
}
```

| derive | 파이썬 대응 | 생성되는 것 |
| :--- | :--- | :--- |
| `Debug` | `__repr__` | `{:?}` 포맷 |
| `Clone` | `copy.deepcopy` 유사 | `.clone()` 깊은 복사 |
| `Copy` | (값 시맨틱) | 비트 복사 허용(이동 대신) |
| `Default` | 기본 인자 | `T::default()` |
| `PartialEq` | `__eq__` | `==`, `!=` |
| `Eq` | (없음) | 반사성 보장 표식 |
| `Hash` | `__hash__` | `HashMap`/`HashSet` 키 |
| `PartialOrd` | `__lt__` 등 | `<`, `<=`, `>`, `>=` |
| `Ord` | (없음) | `sort()`, `BTreeMap` 키 |

여기서 `Eq`와 `Ord`는 메서드가 없는 **표식 트레이트(marker trait)**입니다. `PartialEq`는 "부분 동등성"으로 NaN처럼 자기 자신과도 같지 않은 값을 허용하지만, `Eq`는 "완전 동등성(반사성: 모든 `x`에 대해 `x == x`)"을 약속합니다. `HashMap`의 키는 반드시 `Eq`여야 합니다. 자기 자신과도 다를 수 있는 키로는 해시 테이블이 망가지니까요.

이 설계가 빛나는 순간이 바로 우리의 `Vector2D`입니다.

> **🚨 컴파일러 에러 시연: `f64`는 `Eq`가 아니다 (E0277)**
>
> `Vector2D`에 `Eq`와 `Hash`를 derive해서 `HashMap` 키로 쓰고 싶다고 합시다.
>
> ```rust
> #[derive(PartialEq, Eq)] // 🚨
> struct Vector2D { x: f64, y: f64 }
> fn main() {}
> ```
>
> 컴파일러는 단호히 거부합니다:
>
> ```text
> error[E0277]: the trait bound `f64: Eq` is not satisfied
>  --> src/main.rs:2:18
>   |
> 1 | #[derive(PartialEq, Eq)]
>   |                     -- in this derive macro expansion
> 2 | struct Vector2D { x: f64, y: f64 }
>   |                   ^^^^^^ the trait `Eq` is not implemented for `f64`
> ```
>
> 이유는 IEEE 754 부동소수점의 `NaN`입니다. `NaN == NaN`은 `false`이므로 `f64`는 반사성을 만족하지 못합니다. 따라서 표준 라이브러리는 `f64`에 `PartialEq`만 구현하고 `Eq`는 구현하지 않았습니다. 그 결과 `f64` 필드를 가진 구조체도 `Eq`/`Hash`/`Ord`를 derive할 수 없습니다. 그래서 이번 챕터 내내 `Vector2D`에는 **`PartialEq`만** derive했습니다.
>
> 파이썬에서 이건 어땠나요? `float('nan') == float('nan')`은 파이썬에서도 `False`입니다. 하지만 파이썬은 `Vector2D`를 아무 검사 없이 `dict` 키로 넣게 해줍니다. 그러다 `NaN`이 끼면 키 조회가 조용히 실패합니다. Rust는 이 부류의 버그를 **타입 정의 단계에서 차단**합니다. NaN을 신경 써서 키로 쓰고 싶다면, `f64`를 정수 비트 패턴으로 변환하는 뉴타입(`struct OrderedF64(u64)`)을 만들거나 `ordered-float` 크레이트를 씁니다. "안전하지 않은 것은 애초에 컴파일되지 않는다"가 Rust의 일관된 태도입니다.

반대로, **derive가 항상 옳은 것은 아닙니다.** derive는 "모든 필드를 동등하게 취급"하는 구조적 구현을 만듭니다. 도메인 의미가 그와 다르면 수동 구현이 필요합니다. 예를 들어 캐시 항목에서 `value`는 비교에서 제외하고 `key`로만 동등성을 판단해야 한다면, `PartialEq`를 손으로 구현해야 합니다.

```rust
struct CacheEntry { key: String, value: Vec<u8>, hits: u64 }

impl PartialEq for CacheEntry {
    fn eq(&self, other: &Self) -> bool {
        self.key == other.key // value, hits는 동등성 판단에서 제외
    }
}
```

---

## 9.6 트레이트 바운드와 제네릭, 그리고 두 갈래의 디스패치

지금까지는 한 타입에 트레이트를 구현하는 쪽이었습니다. 이제 그 트레이트를 **소비**하는 코드, 즉 다형성을 봅시다. 파이썬 개발자에게 이 부분이 가장 충격적인데, 같은 "다형성"이 Rust에서 두 개의 완전히 다른 기계어로 컴파일되기 때문입니다.

파이썬에서 다형성은 그냥 메서드를 호출하는 것입니다.

```python
def total_area(shapes: list) -> float:
    return sum(s.area() for s in shapes)  # s가 뭐든 area()만 있으면 됨
```

`shapes`에는 `Circle`, `Rectangle`이 섞여 있어도 됩니다. 각 `s.area()` 호출마다 런타임에 `type(s)`를 조회해 메서드를 찾습니다(가상 디스패치). 유연하지만 매 호출이 조회 비용을 냅니다.

Rust는 이 "행동의 약속"을 트레이트로 정의하고, 그것을 **두 방식**으로 소비할 수 있습니다.

```rust
pub trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &'static str;
}

pub struct Circle { pub radius: f64 }
pub struct Rectangle { pub width: f64, pub height: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
    fn name(&self) -> &'static str { "Circle" }
}
impl Shape for Rectangle {
    fn area(&self) -> f64 { self.width * self.height }
    fn name(&self) -> &'static str { "Rectangle" }
}
```

**길 1: 정적 디스패치 (제네릭 + 트레이트 바운드)**

```rust
// "Shape를 구현한 어떤 타입 T이든 받겠다" — 트레이트 바운드
fn describe_static<T: Shape>(shape: &T) {
    println!("[static] {} 면적: {:.2}", shape.name(), shape.area());
}

// impl Trait 문법(동일한 정적 디스패치의 축약형)
fn describe_static2(shape: &impl Shape) {
    println!("[static] {} 면적: {:.2}", shape.name(), shape.area());
}
```

**길 2: 동적 디스패치 (`dyn Trait`)**

```rust
// "Shape를 구현한 무언가에 대한 참조" — 구체 타입은 런타임에 결정
fn describe_dynamic(shape: &dyn Shape) {
    println!("[dynamic] {} 면적: {:.2}", shape.name(), shape.area());
}
```

문법은 거의 비슷합니다(`&T where T: Shape` vs `&dyn Shape`). 하지만 컴파일 결과는 완전히 다릅니다.

복잡한 바운드는 `where` 절로 가독성을 확보합니다. 트레이트 바운드가 여러 개거나 제네릭 파라미터가 많을 때 시그니처가 한 줄에 안 들어오면 `where`로 분리합니다.

```rust
use std::fmt::Display;
use std::hash::Hash;

// 인라인 바운드: 짧을 때
fn print_all<T: Display>(items: &[T]) {
    for it in items { println!("{it}"); }
}

// where 절: 바운드가 많을 때 — 시그니처와 제약을 분리해 가독성 확보
fn index_by_key<K, V>(pairs: Vec<(K, V)>) -> std::collections::HashMap<K, V>
where
    K: Eq + Hash,
    V: Clone,
{
    pairs.into_iter().collect()
}
```

> **Under the Hood: 단일형화(Monomorphization) vs vtable — 코드 복제와 fat pointer**
>
> **정적 디스패치**는 **단일형화**로 컴파일됩니다. `describe_static::<Circle>`와 `describe_static::<Rectangle>`은 컴파일 타임에 **두 개의 독립된 기계어 함수로 복제**됩니다. 마치 우리가 손으로 `describe_circle`, `describe_rectangle`을 따로 쓴 것처럼요. 각 복제본 안에서 `shape.area()`는 해당 타입의 `area`를 **직접 호출**하거나 아예 **인라인**됩니다. 런타임 조회는 0입니다. 대가는 **바이너리 크기**입니다. 타입 종류만큼 함수가 복제되니, 1000개 타입으로 인스턴스화되는 거대한 제네릭 함수는 바이너리를 부풀립니다(C++ 템플릿의 코드 팽창과 동일한 현상).
>
> **동적 디스패치**는 **vtable(가상 메서드 테이블)**로 컴파일됩니다. `&dyn Shape`는 일반 참조와 달리 **fat pointer(뚱뚱한 포인터)**입니다. 두 개의 워드(64비트 시스템에서 16바이트)로 이루어집니다:
>
> ```text
> &dyn Shape  =  [ 데이터 포인터 8B ][ vtable 포인터 8B ]
>                       │                   │
>                       │                   └─> vtable: [ drop_fn ][ size ][ align ][ area ][ name ]
>                       └─> 실제 Circle/Rectangle 데이터
> ```
>
> `shape.area()`를 호출하면, fat pointer의 vtable 포인터를 따라가 `area` 슬롯의 함수 주소를 읽고, 데이터 포인터를 `self`로 넘겨 그 함수를 **간접 호출(indirect call)**합니다. 이는 파이썬의 타입 객체 슬롯 조회와 본질적으로 같은 메커니즘이지만, 딕셔너리 해싱이 아니라 고정 오프셋 배열 접근이라 훨씬 빠릅니다. 대가는 (1) 간접 호출이라 CPU 분기 예측·인라인이 막히고, (2) 포인터가 두 배로 커진다는 점입니다.
>
> 이 크기 차이를 코드로 직접 측정할 수 있습니다:
>
> ```rust
> use std::mem::size_of;
> fn main() {
>     println!("&Circle:        {} 바이트", size_of::<&Circle>());        // 8
>     println!("&dyn Shape:     {} 바이트", size_of::<&dyn Shape>());     // 16
>     println!("Box<Circle>:    {} 바이트", size_of::<Box<Circle>>());    // 8
>     println!("Box<dyn Shape>: {} 바이트", size_of::<Box<dyn Shape>>()); // 16
> }
> ```
>
> 실행 결과: `&Circle`은 8바이트(단순 주소), `&dyn Shape`는 16바이트(주소 + vtable). 이 8바이트의 차이가 fat pointer의 물리적 증거입니다.

동적 디스패치가 필수인 경우가 있습니다. **이종(heterogeneous) 컬렉션**입니다. `Vec<T>`는 모든 원소가 같은 타입 `T`여야 하므로, `Circle`과 `Rectangle`을 한 `Vec`에 담으려면 동적 디스패치가 필요합니다.

```rust
fn main() {
    // 정적 디스패치는 이걸 못 함: 원소 타입이 제각각이므로
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 1.0 }),
        Box::new(Rectangle { width: 2.0, height: 2.0 }),
    ];
    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("총 면적: {:.2}", total);
}
```

이것이 Chapter 8에서 본 "상속 다형성"의 Rust식 대체재입니다. 파이썬에서 `List[BaseShape]`에 자식 클래스 인스턴스를 섞어 담던 것을, Rust는 `Vec<Box<dyn Shape>>`로 표현합니다. Chapter 10에서 보겠지만, **닫힌 집합(고정된 타입 목록)**이라면 `dyn` 대신 `enum`이 더 빠르고 안전합니다. `dyn`은 "구현 타입을 미리 알 수 없는 열린 집합"(예: 플러그인)에 어울립니다.

### 벤치마크: 정적 vs 동적 디스패치

추상적 설명보다 수치가 설득력 있습니다. 작은 변환 함수를 5천만 원소 슬라이스에 적용하는, 디스패치 비용이 두드러지는 시나리오를 측정했습니다(M-시리즈, `--release`, `opt-level=3`, 5회 중 최소값).

```rust
trait Transform { fn apply(&self, x: f64) -> f64; }
struct Scale(f64);
impl Transform for Scale { fn apply(&self, x: f64) -> f64 { x * self.0 + 1.0 } }

fn sum_static<T: Transform>(op: &T, data: &[f64]) -> f64 {
    let mut acc = 0.0;
    for &x in data { acc += op.apply(x); }  // 인라인 + 벡터화 가능
    acc
}
fn sum_dynamic(op: &dyn Transform, data: &[f64]) -> f64 {
    let mut acc = 0.0;
    for &x in data { acc += op.apply(x); }  // vtable 경유, 벡터화 차단
    acc
}
```

```text
입력: f64 5천만 개, --release, opt-level=3
static (제네릭) :  28.98 ms
dynamic (dyn)   :  37.70 ms
비율            :  1.30x  (동적이 ~30% 느림)
```

정적 디스패치가 약 1.3배 빠릅니다. 차이의 핵심은 **벡터화(SIMD)**입니다. 정적 버전은 `apply`가 인라인되어 컴파일러가 루프 전체를 SSE/AVX 명령으로 묶어 처리합니다. 동적 버전은 매 반복마다 vtable을 경유한 간접 호출이라 인라인이 막히고, 따라서 벡터화도 막힙니다. 단, 이 1.3배는 "함수 본문이 극도로 작은" 최악에 가까운 경우입니다. `apply`가 무거운 작업(예: 정규식 매칭, 파일 I/O)이라면 디스패치 오버헤드는 측정 불가 수준으로 묻힙니다.

> 비교 관점: 같은 작업을 순수 파이썬으로 하면 5천만 번의 메서드 디스패치 + 박싱된 float 산술로 수 초가 걸립니다(대략 1000배 이상 느림). 파이썬에서 `dyn` vs 제네릭 같은 "1.3배" 고민은 사치입니다. 어떤 디스패치를 쓰든 Rust는 파이썬보다 두세 자릿수 빠릅니다. 이 1.3배는 **Rust 내부에서의 미세 최적화** 영역일 뿐, 대부분의 코드에서는 가독성과 유연성을 보고 고르면 됩니다.

### `From`/`Into`의 실무 가치

9.4에서 `From<(f64, f64)>`를 살짝 봤습니다. 이제 그 실무적 위력을 봅시다. `From`/`Into`는 단순한 타입 변환을 넘어 **API를 유연하게 만드는 도구**입니다.

파이썬 함수는 `def log(msg: str | bytes)`처럼 여러 입력 타입을 받고 내부에서 분기합니다. Rust는 `impl Into<String>` 바운드로 같은 유연성을 얻되, 변환 로직을 호출부로 위임합니다.

```rust
// "String으로 변환 가능한 무엇이든" 받는다
fn log_message(msg: impl Into<String>) {
    let s: String = msg.into();
    println!("LOG: {s}");
}

fn main() {
    log_message("문자열 리터럴");          // &str -> String 변환
    log_message(String::from("소유된 값")); // String -> String (무비용)
}
```

호출자는 `&str`이든 `String`이든 신경 쓰지 않고 넘기고, 함수는 통일된 `String`으로 받습니다. 표준 라이브러리 전체가 이 패턴으로 도배되어 있습니다(`Vec::from`, `String::from`, `PathBuf::from` 등).

실패할 수 있는 변환은 `TryFrom`/`TryInto`로 분리합니다. 파이썬의 `int("abc")`가 `ValueError`를 던지는 상황을, Rust는 타입에 박아 넣습니다.

```rust
use std::convert::TryFrom;

#[derive(Debug)]
struct Percentage(u8);

impl TryFrom<i64> for Percentage {
    type Error = String;
    fn try_from(v: i64) -> Result<Self, Self::Error> {
        if (0..=100).contains(&v) {
            Ok(Percentage(v as u8))
        } else {
            Err(format!("0~100 범위를 벗어남: {v}"))
        }
    }
}

fn main() {
    println!("{:?}", Percentage::try_from(80));  // Ok(Percentage(80))
    println!("{:?}", Percentage::try_from(150)); // Err("0~100 범위를 벗어남: 150")
}
```

`From`은 절대 실패하지 않는 변환(`&str` → `String`), `TryFrom`은 실패할 수 있는 변환(`i64` → `Percentage`)을 표현합니다. 파이썬에서 "변환이 실패할 수 있는가?"가 문서에만 적혀 있던 정보가, Rust에서는 시그니처의 `Result`로 드러납니다. `Result`의 전체 활용법은 Chapter 12에서 본격적으로 다룹니다.

---

## 9.7 고아 규칙(Orphan Rule)과 newtype 우회

9.4에서 `impl Mul<Vector2D> for f64`가 합법이라고 했습니다. 그 규칙을 정확히 못 박을 차례입니다. Rust의 **고아 규칙(Orphan Rule)**은 이렇게 말합니다:

> `impl Trait for Type`을 작성하려면, **`Trait`이 내 크레이트의 것이거나, `Type`이 내 크레이트의 것**이어야 한다. 둘 다 외부 것이면 금지한다.

`impl Mul<Vector2D> for f64`에서 `Mul`은 외부(std), `f64`도 외부지만, 트레이트의 타입 파라미터 자리에 로컬 타입 `Vector2D`가 있어 통과합니다. 반면 둘 다 완전히 외부면 막힙니다.

> **🚨 컴파일러 에러 시연: 고아 규칙 위반 (E0117)**
>
> 외부 트레이트(`Display`)를 외부 타입(`Vec<String>`)에 구현하려는 시도입니다.
>
> ```rust
> use std::fmt;
> impl fmt::Display for Vec<String> { // 🚨 둘 다 외부 것
>     fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
>         write!(f, "vec")
>     }
> }
> fn main() {}
> ```
>
> 실제 에러:
>
> ```text
> error[E0117]: only traits defined in the current crate can be
>               implemented for types defined outside of the crate
>  --> src/main.rs:2:1
>   |
> 2 | impl fmt::Display for Vec<String> {
>   | ^^^^^^^^^^^^^^^^^^^^^^-----------
>   |                       |
>   |                       `Vec` is not defined in the current crate
>   |
>   = note: define and implement a trait or new type instead
> ```

**왜 이런 규칙이 있을까요?** 일관성(coherence)을 보장하기 위해서입니다. 만약 고아 규칙이 없다면, 크레이트 A가 `impl Display for Vec<String>`을 한 가지 방식으로 구현하고, 크레이트 B가 또 다른 방식으로 구현할 수 있습니다. 당신의 프로그램이 A와 B를 동시에 의존하면, 컴파일러는 `vec.to_string()`이 어느 구현을 써야 할지 결정할 수 없습니다. 파이썬의 몽키 패칭(`SomeClass.method = my_func`)이 정확히 이 혼돈을 일으킵니다. 라이브러리 두 개가 같은 클래스를 서로 다르게 패치하면 임포트 순서에 따라 동작이 바뀝니다. Rust는 이 부류의 "전역 상태 오염"을 언어 차원에서 원천 봉쇄합니다.

**우회법: newtype 패턴.** Chapter 8에서 만난 `EmailAddress` 뉴타입을 기억하나요? 외부 타입을 내 크레이트의 튜플 구조체로 감싸면, 그 래퍼는 로컬 타입이 되어 고아 규칙을 벗어납니다.

```rust
use std::fmt;

// 외부 타입 Vec<String>을 로컬 타입으로 감싼다
struct Wrapper(Vec<String>);

// 이제 Wrapper는 로컬 타입이므로 Display 구현 합법
impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("a"), String::from("b")]);
    println!("{w}"); // [a, b]
}
```

> **Under the Hood: newtype의 런타임 비용은 0**
>
> `struct Wrapper(Vec<String>)`은 메모리에서 `Vec<String>`과 **완전히 동일**합니다. 추가 필드도, 태그도, 간접 참조도 없습니다. 단일 필드 튜플 구조체는 컴파일 후 그 필드와 같은 레이아웃·같은 크기를 갖습니다(`#[repr(transparent)]`로 이를 명시적으로 보장할 수도 있습니다). 즉 고아 규칙을 우회하기 위한 newtype은 타입 시스템상의 새 이름일 뿐, 기계어에서는 흔적도 없이 사라집니다. Chapter 8의 `UserId(i32)`가 정확히 `i32`와 같은 크기였던 것과 동일한 원리입니다. "타입 안전성을 공짜로 산다"는 Rust의 약속이 여기서도 지켜집니다.

newtype은 단순히 고아 규칙 우회용만이 아닙니다. 외부 타입에 도메인 의미를 부여하고(예: `Meters(f64)` vs `Seconds(f64)`), 의도치 않은 연산을 막는 도구입니다. 다만 래핑한 타입의 메서드를 그대로 쓰려면 `Deref`를 구현하거나 필요한 메서드를 위임(delegate)해야 하는 보일러플레이트가 따릅니다. 모든 것을 newtype으로 감싸지는 마세요.

---

## 9.8 실전 Hands-on: 측정 단위 시스템

배운 것을 모두 엮어, 물리 단위를 타입으로 표현하는 작은 시스템을 만듭니다. 화성 기후 궤도선이 단위 혼동(파운드-초 vs 뉴턴-초)으로 추락한 사건을 기억한다면, 이게 왜 중요한지 알 겁니다. 파이썬에서는 모든 거리·시간이 `float`이라 컴파일러가 막아주지 못하지만, Rust는 단위를 타입으로 분리해 **차원 오류를 컴파일 에러로** 만듭니다.

`Display`, `Add`, `Sub`, `Mul`, `From`, `PartialEq`, `PartialOrd`를 한 예제에 결집했습니다. 아래는 그대로 `cargo run` 되는 완전한 코드입니다.

```rust
use std::fmt;
use std::ops::{Add, Sub, Mul};

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Meters(f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Seconds(f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct MetersPerSecond(f64);

// __str__: 사람용 출력
impl fmt::Display for Meters {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}m", self.0)
    }
}
impl fmt::Display for MetersPerSecond {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}m/s", self.0)
    }
}

// __add__/__sub__: 같은 차원끼리만 가/감산 (Meters + Meters = Meters)
impl Add for Meters {
    type Output = Meters;
    fn add(self, rhs: Meters) -> Meters { Meters(self.0 + rhs.0) }
}
impl Sub for Meters {
    type Output = Meters;
    fn sub(self, rhs: Meters) -> Meters { Meters(self.0 - rhs.0) }
}

// __truediv__ 역할: Meters / Seconds = MetersPerSecond
// (Div를 직접 구현해 차원이 올바르게 바뀌도록)
impl std::ops::Div<Seconds> for Meters {
    type Output = MetersPerSecond;
    fn div(self, t: Seconds) -> MetersPerSecond {
        MetersPerSecond(self.0 / t.0)
    }
}

// MetersPerSecond * Seconds = Meters (역연산)
impl Mul<Seconds> for MetersPerSecond {
    type Output = Meters;
    fn mul(self, t: Seconds) -> Meters {
        Meters(self.0 * t.0)
    }
}

// From: f64 리터럴에서 편하게 생성
impl From<f64> for Meters {
    fn from(v: f64) -> Self { Meters(v) }
}
impl From<f64> for Seconds {
    fn from(v: f64) -> Self { Seconds(v) }
}

fn main() {
    let distance: Meters = 100.0.into();
    let time = Seconds(9.58);

    // 속도 = 거리 / 시간 (차원이 자동으로 m/s가 됨)
    let speed = distance / time;
    println!("{distance} 를 {}s 만에 → {speed}", time.0);

    // 같은 차원끼리는 비교/연산 가능
    let lap1 = Meters(400.0);
    let lap2 = Meters(380.0);
    println!("두 바퀴 합계: {}", lap1 + lap2);
    println!("lap1 > lap2 ? {}", lap1 > lap2); // PartialOrd 덕분

    // 역연산: 속도 × 시간 = 거리
    let predicted = speed * Seconds(20.0);
    println!("20초 후 예상 거리: {predicted}");

    // 🚨 차원 오류는 컴파일이 거부한다:
    // let bad = distance + time;  // error[E0308]: Meters + Seconds 불가
    // let bad2 = distance < time; // 비교 불가, 타입 불일치
}
```

실행 결과:

```text
100m 를 9.58s 만에 → 10.438413361169102m/s
두 바퀴 합계: 780m
lap1 > lap2 ? true
20초 후 예상 거리: 208.76826722338205m
```

주석 처리된 `distance + time`의 주석을 풀면 `error[E0308]: mismatched types`가 즉시 뜹니다. 파이썬이라면 두 `float`을 아무 의심 없이 더해 의미 없는 숫자를 만들었을 코드입니다. Rust는 **거리와 시간은 더할 수 없다**는 물리 법칙을 타입 시스템에 새겨 넣었고, 그 위반을 컴파일 타임에 잡습니다. 게다가 `Meters`, `Seconds`는 모두 단일 필드 뉴타입이라 런타임 표현은 그냥 `f64`입니다 — 9.7에서 본 대로 비용은 0입니다.

실무에서 이 패턴이 필요하다면 직접 구현하기 전에 `uom`(units of measurement) 크레이트를 보세요. SI 단위계 전체를 타입 안전하게 제공하며, 차원 분석을 컴파일 타임에 수행합니다. 직렬화가 필요하면 `serde`로 이 뉴타입들에 `Serialize`/`Deserialize`를 derive하면 됩니다.

---

## 9.9 Best Practices & 흔한 함정

### Best Practices 체크리스트

- **derive를 먼저 시도하라.** `#[derive(Debug, Clone, PartialEq)]`는 거의 모든 데이터 타입의 기본 출발점입니다. 수동 구현은 derive가 만든 것이 도메인 의미와 다를 때만.
- **`Display`는 신중하게.** `{}`로 찍히는 출력은 로그·UI에 노출됩니다. 디버깅 정보는 `Debug`로, 사람용 메시지는 `Display`로 명확히 분리하세요.
- **`From`을 구현하고 `Into`는 공짜로 받아라.** 변환은 항상 `From` 쪽에 구현합니다(`Into`는 blanket impl로 자동 제공). 함수 인자는 `impl Into<T>`로 받아 호출부 유연성을 높이세요.
- **실패 가능 변환은 `TryFrom`.** 절대 실패하지 않는 변환만 `From`. 패닉 가능성이 있으면 `TryFrom`으로 `Result`를 반환하세요.
- **연산자 오버로딩은 직관적 의미에만.** `+`는 덧셈처럼 동작해야 합니다. `Path`에 `/`(경로 결합)처럼 강한 관례가 있는 경우를 제외하고, `+`로 전혀 다른 의미를 만들지 마세요.
- **기본은 정적 디스패치.** 제네릭 + 트레이트 바운드를 기본값으로 쓰고, 동적 디스패치는 그것이 *필요할 때만* 도입하세요(다음 항목 참고).
- **clippy를 신뢰하라.** `clippy::needless_lifetimes`, `clippy::should_implement_trait` 등은 트레이트 설계 실수를 잡아줍니다. `cargo clippy`를 CI에 넣으세요.

### 흔한 함정 (파이썬 개발자가 자주 하는 실수)

**함정 1: 불필요한 `dyn` 남용.** 파이썬의 동적 디스패치에 익숙한 나머지, 모든 다형성을 `Box<dyn Trait>`로 처리하려는 충동이 있습니다. 단일 타입만 다루는 함수에 `&dyn Trait`을 쓰면 인라인·벡터화를 스스로 포기하는 셈입니다.

```rust
// 흔한 실수: 호출부가 항상 한 가지 타입인데 dyn 사용
fn process(handler: &dyn Handler) { handler.run(); }

// 권장: 정적 디스패치 (단일형화 + 인라인)
fn process<H: Handler>(handler: &H) { handler.run(); }
```

`dyn`은 (1) 이종 컬렉션(`Vec<Box<dyn T>>`), (2) 반환 타입이 호출부마다 다른데 제네릭으로 표현 불가한 경우, (3) 바이너리 크기가 코드 복제로 문제가 될 때만 쓰세요. 그 외엔 제네릭이 정답입니다.

**함정 2: 트레이트 과설계.** "나중에 다른 구현이 생길지도 모르니" 추상화부터 만드는 습관은 Rust에서 특히 비싼 실수입니다. 구현체가 하나뿐인 트레이트는 보일러플레이트와 `dyn`의 유혹만 늘립니다. **구체 타입으로 시작하고, 두 번째 구현이 실제로 등장할 때 트레이트를 추출**하세요(YAGNI). 트레이트는 "이미 존재하는 공통 행동을 추상화"할 때 진가를 발휘합니다.

**함정 3: `__hash__` 비대칭 버그를 그대로 옮기기.** 파이썬에서 `__eq__`만 정의하고 `__hash__`를 빠뜨린 버그를, Rust로 옮길 때 수동 `PartialEq` 구현과 `#[derive(Hash)]`의 정의가 **불일치**하면 똑같은 버그가 재현됩니다. "같다고 판정된 두 값은 반드시 같은 해시를 가져야 한다"는 계약을 손으로 구현할 때 직접 지켜야 합니다. 가능하면 둘 다 derive하거나 둘 다 수동으로, 일관되게.

**함정 4: `Display` 안에서 `?` 대신 `write!` 결과 무시.** `fmt` 함수는 `fmt::Result`를 반환해야 합니다. `write!(f, ...)`의 반환을 무시하지 말고 그대로 반환하거나 `?`로 전파하세요. 무시하면 버퍼가 가득 찼을 때의 에러를 놓칩니다.

**함정 5: 연산자가 값을 소비하는 것을 잊음.** `impl Add for Vector2D`의 `add(self, rhs)`는 `self`와 `rhs`를 **이동(move)**시킵니다. `Copy`가 아닌 타입(예: `String` 필드를 가진 구조체)이라면 `a + b` 후 `a`를 다시 못 씁니다. 이때는 `impl Add<&Vector2D> for &Vector2D`처럼 참조에 대한 구현을 추가하거나 `Copy`를 derive하세요(우리 `Vector2D`는 `Copy`라 문제없습니다 — 이것이 9.4에서 `Copy`를 붙인 이유입니다).

---

이번 챕터에서 우리는 파이썬의 던더 메서드가 Rust의 표준 트레이트로 어떻게 1:1 대응되는지, 그리고 그 대응이 단순한 문법 치환이 아니라 **"런타임 검사를 컴파일 타임 증명으로 옮기는"** 사고방식의 전환임을 확인했습니다. `Vector2D`는 `Display`부터 `From<(f64, f64)>`까지 갖춘 완전한 값 타입이 되었고, 그 과정에서 단일형화와 vtable, fat pointer, 고아 규칙, newtype 우회를 메모리 바이트와 벤치마크 수치로 해부했습니다. 핵심 교훈은 하나입니다 — Rust의 트레이트는 파이썬 던더의 표현력을 거의 그대로 주면서, 그 비용을 0으로 만들고 함정을 컴파일 에러로 바꿉니다.

`dyn Trait`으로 다형성을 다뤘지만, "구현 타입이 미리 정해진 닫힌 집합"이라면 더 빠르고 안전한 길이 있다고 암시했습니다. 다음 Chapter 10에서는 파이썬 개발자가 `Union` 타입과 `isinstance()`로 씨름하던 문제를, Rust의 **열거형(Enum)과 패턴 매칭**이 런타임 오버헤드 없이, 그리고 "모든 경우를 빠짐없이 처리했는가(Exhaustiveness)"를 컴파일러가 보장하며 어떻게 우아하게 해결하는지 — 결제 상태 기계를 모델링하며 확인하겠습니다.

---

## References

- https://doc.rust-lang.org/book/ch10-02-traits.html
- https://doc.rust-lang.org/std/fmt/trait.Display.html
- https://doc.rust-lang.org/std/fmt/trait.Debug.html
- https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html
- https://doc.rust-lang.org/std/ops/index.html
- https://doc.rust-lang.org/book/ch20-03-advanced-traits.html
