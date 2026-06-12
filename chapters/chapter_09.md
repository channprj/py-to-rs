# Chapter 9. 매직 메서드(Dunder)에서 Trait으로

파이썬 클래스의 강력함은 속칭 **'던더(Dunder)'**라고 불리는 매직 메서드(Magic Method)에서 나옵니다. 어떤 객체든 `__str__`을 구현하면 `print()`로 출력할 수 있고, `__len__`을 구현하면 `len()` 함수를 쓸 수 있으며, `__add__`를 구현하면 `+` 기호로 더할 수 있습니다. 이것이 파이썬의 핵심 철학인 **덕 타이핑(Duck Typing)**의 근간입니다.

"오리처럼 걷고 오리처럼 꽥꽥거리면, 그것은 오리다."
파이썬은 객체의 '실제 타입'이 무엇인지 묻지 않습니다. 그저 런타임에 해당 매직 메서드가 존재하는지만 검사합니다. 편리하지만, 오타 하나로 런타임 오류가 발생하기 십상입니다.

Rust는 이 덕 타이핑의 유연함을 유지하면서도, 컴파일 타임에 100% 안전성을 보장하기 위해 **트레이트(Trait)**라는 강력한 무기를 사용합니다.

---

## 9.1 Trait이란 무엇인가?

Trait은 객체가 반드시 구현해야 할 **"행동(메서드)의 약속(Interface)"**입니다. 파이썬의 매직 메서드들을 Rust는 모두 표준 라이브러리의 Trait으로 분리해 두었습니다.

| 역할 | 파이썬 매직 메서드 | Rust 표준 Trait |
| :--- | :--- | :--- |
| 문자열 출력 | `__str__` | `std::fmt::Display` |
| 디버깅 출력 | `__repr__` | `std::fmt::Debug` |
| 덧셈 연산 | `__add__` | `std::ops::Add` |
| 동등성 비교 | `__eq__` | `std::cmp::PartialEq` |
| 대소 비교 | `__lt__`, `__gt__` | `std::cmp::PartialOrd` |

---

## 9.2 파이썬의 `__str__`을 Rust의 `Display`로

수학의 벡터를 표현하는 데이터 모델을 만들어보며 두 언어의 차이를 비교해 봅시다.

* **The Python Way (Magic Methods)**
  ```python
  class Vector2D:
      def __init__(self, x: float, y: float):
          self.x = x
          self.y = y

      # print() 호출 시 사용됨
      def __str__(self) -> str:
          return f"({self.x}, {self.y})"
          
  v = Vector2D(3.0, 4.0)
  print(v) # (3.0, 4.0)
  ```

* **The Rust Implementation (Trait)**
  Rust에서는 기본적으로 커스텀 구조체를 바로 `println!("{ }", v)`로 출력할 수 없습니다. 문자열로 표현하는 행동(`Display` 트레이트)을 명시적으로 구현해야 합니다.

  ```rust
  use std::fmt;

  // 데이터 구조 정의
  pub struct Vector2D {
      pub x: f64,
      pub y: f64,
  }

  // 행동(Trait)의 구현: 파이썬의 __str__ 역할
  impl fmt::Display for Vector2D {
      fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
          write!(f, "({}, {})", self.x, self.y)
      }
  }

  fn main() {
      let v = Vector2D { x: 3.0, y: 4.0 };
      println!("{}", v); // Display 트레이트가 구현되어 있으므로 정상 출력됨!
  }
  ```
  파이썬처럼 클래스 안에 메서드를 욱여넣지 않고, `impl [Trait 이름] for [타입]` 문법을 사용해 데이터와 행동의 구현을 완전히 분리합니다.

---

## 9.3 연산자 오버로딩: `__add__` vs `std::ops::Add`

두 벡터를 더하는 로직을 추가해 봅시다.

* **The Python Way**
  ```python
  class Vector2D:
      # ... (생략) ...
      def __add__(self, other: 'Vector2D') -> 'Vector2D':
          if not isinstance(other, Vector2D):
              return NotImplemented
          return Vector2D(self.x + other.x, self.y + other.y)
          
  v1 = Vector2D(1.0, 2.0)
  v2 = Vector2D(3.0, 4.0)
  v3 = v1 + v2
  ```

* **The Rust Implementation**
  Rust에서는 `+` 연산자를 사용하기 위해 `std::ops::Add` 트레이트를 구현합니다. 이때, 반환되는 타입(Output)이 무엇인지도 명확히 지정합니다.

  ```rust
  use std::ops::Add;

  // Add 트레이트 구현: 파이썬의 __add__ 역할
  impl Add for Vector2D {
      type Output = Vector2D; // 연산 결과의 타입을 명시

      // add 함수의 매개변수로 self(왼쪽 값)와 rhs(오른쪽 값)를 받습니다.
      fn add(self, rhs: Vector2D) -> Self::Output {
          Vector2D {
              x: self.x + rhs.x,
              y: self.y + rhs.y,
          }
      }
  }

  fn main() {
      let v1 = Vector2D { x: 1.0, y: 2.0 };
      let v2 = Vector2D { x: 3.0, y: 4.0 };
      
      let v3 = v1 + v2; // Add 트레이트 덕분에 '+' 기호 사용 가능!
      println!("v3: {}", v3); // (4.0, 6.0)
  }
  ```

### 컴파일러가 만들어주는 마법: `#[derive]`
파이썬의 `dataclasses`를 쓰면 `__eq__`나 `__repr__`이 자동으로 생성되듯, Rust도 **매크로**를 통해 똑같은 편의성을 제공합니다. 이를 `#[derive]`라고 합니다.

```rust
// Debug: __repr__ (디버그용 출력 "{:?}") 자동 구현
// PartialEq: __eq__ (== 연산자 비교) 자동 구현
// Clone: 깊은 복사(.clone()) 지원 자동 구현
#[derive(Debug, PartialEq, Clone)]
pub struct Vector2D {
    pub x: f64,
    pub y: f64,
}

fn main() {
    let v1 = Vector2D { x: 1.0, y: 2.0 };
    let v2 = Vector2D { x: 1.0, y: 2.0 };
    
    // PartialEq 덕분에 바로 비교 가능
    if v1 == v2 {
        // Debug 덕분에 {:?} 로 내부 상태 자동 출력
        println!("They are equal: {:?}", v1); 
    }
}
```

---

## 9.4 덕 타이핑의 정적 디스패치 (Trait Bounds)

이제 가장 중요한 다형성(Polymorphism)을 살펴봅시다. 파이썬에서는 어떤 객체든 `summarize()` 메서드만 있으면 실행되는 코드를 짤 수 있습니다. 

```python
def print_summary(item):
    print(item.summarize()) # item이 뭔지 모르지만 일단 호출해봄 (런타임 덕 타이핑)
```

Rust에서는 **Trait Bounds(트레이트 경계)**를 사용하여 "이 함수는 `Summary` 트레이트를 구현한 객체만 받을 수 있다"고 컴파일러에게 못을 박습니다.

```rust
// 행동의 약속 정의
pub trait Summary {
    fn summarize(&self) -> String;
}

// 1. Tweet 구조체에 Summary 트레이트 구현
pub struct Tweet { pub content: String }
impl Summary for Tweet {
    fn summarize(&self) -> String { format!("Tweet: {}", self.content) }
}

// 2. Article 구조체에도 Summary 트레이트 구현
pub struct Article { pub title: String }
impl Summary for Article {
    fn summarize(&self) -> String { format!("Article: {}", self.title) }
}

// 3. 다형성 활용 함수 (Trait Bound)
// "Summary 트레이트를 구현한 어떤 타입(T)이든 다 받아들이겠다"
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}
```

**Zero-cost Abstraction (단일형화)**
앞선 Chapter 1에서 설명했듯, `print_summary` 함수를 `Tweet`으로 호출할 때와 `Article`로 호출할 때, Rust 컴파일러는 이 함수를 **두 개의 서로 다른 기계어 함수로 복제**하여 최적화합니다. 파이썬처럼 런타임에 메서드 테이블(`__dict__`)을 뒤져서 `summarize` 함수를 찾는 오버헤드가 단 1나노초도 들지 않습니다.

파이썬의 마법은 런타임 오버헤드를 담보로 하지만, Rust의 트레이트는 **컴파일 타임에 검증되는 가장 빠르고 안전한 마법**입니다. 

다음 챕터에서는 파이썬 개발자들이 `isinstance()`와 `Union` 타입을 쓰며 겪는 한계를, Rust의 **Enum과 패턴 매칭**이 얼마나 우아하게 해결해 주는지 확인해 보겠습니다.
