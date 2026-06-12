# Chapter 7. Struct와 데이터 모델링 (`dataclass`와 `Pydantic`의 대체)

파이썬 개발자들이 OOP(객체 지향 프로그래밍) 패러다임에서 가장 많이, 그리고 편하게 사용하는 것은 바로 "데이터를 담는 그릇"으로서의 클래스입니다. 과거에는 길고 지루한 `__init__` 메서드를 손수 작성해야 했지만, 최근에는 `dataclasses` 모듈이나 FastAPI와 찰떡궁합인 `Pydantic` 덕분에 데이터 모델링이 매우 간결해졌습니다.

하지만 파이썬의 클래스는 내부적으로 거대한 `__dict__` (해시 테이블)에 불과합니다. 런타임에 동적으로 속성(Attribute)을 추가하거나 삭제할 수 있는 유연성을 제공하지만, 이는 극심한 메모리 오버헤드와 타입 불확실성을 초래합니다.

Rust에는 **`class` 키워드가 존재하지 않습니다.** 대신 데이터를 정의하는 `Struct`(구조체)와 행동을 정의하는 `impl` 블록, 그리고 다형성을 담당하는 `Trait`으로 철저히 분리되어 있습니다. 이번 장에서는 파이썬의 클래스 데이터 모델이 Rust의 Struct로 어떻게 우아하게 매핑되는지 심층적으로 알아봅니다.

---

## 7.1 `__init__`의 종말: Struct 선언과 초기화

파이썬에서 단순한 회원(User) 데이터 모델을 만든다고 가정해 보겠습니다.

* **The Python Way (Dataclass)**
  ```python
  from dataclasses import dataclass
  from typing import Optional
  import uuid

  @dataclass
  class User:
      username: str
      email: str
      age: int
      id: str = dataclass.field(default_factory=lambda: str(uuid.uuid4()))
      is_active: bool = True
  
  # 인스턴스 생성
  user = User(username="pythonista", email="py@example.com", age=30)
  # 파이썬은 언제든 속성을 동적으로 추가할 수 있는 위험이 있습니다.
  user.is_admin = False # 런타임에 통과됨! (slots를 쓰지 않는 이상)
  ```

* **The Rust Mental Model**
  Rust의 `Struct`는 C 언어의 구조체와 유사하게 **메모리 상의 고정된 크기**를 갖는 데이터 블록입니다. 런타임에 속성을 동적으로 추가하거나 삭제하는 것은 컴파일러가 원천 차단합니다. 생성자(`__init__`)라는 특별한 매직 메서드는 없으며, 단순히 구조체의 모든 필드 값을 채워서 반환하는 일반 함수(보통 `new`라고 명명)를 만듭니다.

  ```rust
  use uuid::Uuid;

  // 데이터의 '형태'만 정의합니다.
  pub struct User {
      pub id: String,
      pub username: String,
      pub email: String,
      pub age: u8, // 0~255 사이의 정수 (메모리 최적화)
      pub is_active: bool,
  }

  // 데이터의 '행동(메서드)'은 impl 블록에 분리해서 작성합니다.
  impl User {
      // 파이썬의 __init__ 역할을 하는 연관 함수 (Associated Function)
      // self를 파라미터로 받지 않으므로 클래스 메서드(Class Method)와 유사합니다.
      pub fn new(username: String, email: String, age: u8) -> Self {
          Self {
              id: Uuid::new_v4().to_string(), // 기본값 처리
              username,
              email,
              age,
              is_active: true, // 기본값 처리
          }
      }
  }

  fn main() {
      // User::new(...) 형태로 인스턴스화
      let user = User::new(
          String::from("rustacean"),
          String::from("rs@example.com"),
          30
      );
      // user.is_admin = true; // 컴파일 에러! 정의되지 않은 필드
  }
  ```

---

## 7.2 캡슐화: `@property` 데코레이터를 Getter/Setter로 구현하기

파이썬에서는 변수에 직접 접근하는 것을 막고 로직을 태우기 위해 `@property` 데코레이터를 사용합니다.

* **The Python Way**
  ```python
  class Account:
      def __init__(self, balance: int):
          self._balance = balance # 언더스코어로 '내부 변수'임을 암시 (강제성 없음)

      @property
      def balance(self) -> int:
          return self._balance

      @balance.setter
      def balance(self, amount: int):
          if amount < 0:
              raise ValueError("Balance cannot be negative")
          self._balance = amount
  ```

* **The Rust Implementation**
  Rust는 모듈과 구조체의 접근 제어자(`pub`)를 통해 완벽한 캡슐화를 지원합니다. `pub`을 붙이지 않은 필드는 모듈 외부에서 절대 접근할 수 없습니다 (Private by default).

  ```rust
  pub struct Account {
      balance: i32, // pub이 없으므로 외부에서 접근 불가 (완벽한 private)
  }

  impl Account {
      pub fn new(balance: i32) -> Self {
          if balance < 0 {
              panic!("Initial balance cannot be negative");
          }
          Self { balance }
      }

      // Getter 메서드: 불변 참조(&self)를 받음
      pub fn balance(&self) -> i32 {
          self.balance
      }

      // Setter 메서드: 가변 참조(&mut self)를 받음
      pub fn set_balance(&mut self, amount: i32) -> Result<(), String> {
          if amount < 0 {
              return Err("Balance cannot be negative".to_string());
          }
          self.balance = amount;
          Ok(())
      }
  }
  ```

---

## 7.3 Pydantic의 대체: `Serde` 크레이트의 마법

최근 파이썬 백엔드(FastAPI 등)에서 절대 빠지지 않는 것이 `Pydantic`입니다. 입력된 JSON을 파이썬 객체로 변환하고 타입을 검증하는 강력한 도구입니다. Rust에서는 전 세계 표준이라 할 수 있는 **`serde` (Serialize/Deserialize)** 크레이트가 이 역할을 완벽하게 압도합니다.

* **The Python Way (Pydantic)**
  ```python
  from pydantic import BaseModel, Field

  class Item(BaseModel):
      id: int
      name: str
      price: float = Field(gt=0, description="The price must be greater than zero")
      tags: list[str] = []

  json_data = '{"id": 1, "name": "Apple", "price": 10.5}'
  item = Item.model_validate_json(json_data)
  ```
  Pydantic은 매우 훌륭하지만 런타임에 C/Rust 코드를 호출하여 JSON을 파싱하고 딕셔너리 구조를 검증하는 과정에서 CPU 자원을 소모합니다.

* **The Rust Way (`serde` + `serde_json`)**
  Rust의 `serde`는 매크로(`#[derive(...)]`)를 사용하여 **컴파일 타임에 직렬화/역직렬화 코드를 생성**합니다. 리플렉션(Reflection)이나 런타임 타입 검사가 없으므로 속도가 C++와 맞먹을 만큼 경이롭습니다.

  ```rust
  use serde::{Deserialize, Serialize};

  // #[derive] 매크로 하나면 모든 변환 코드가 컴파일 타임에 주입됩니다.
  #[derive(Debug, Serialize, Deserialize)]
  pub struct Item {
      pub id: i32,
      pub name: String,
      // Pydantic의 Validation처럼 역직렬화 시 기본값 등을 설정할 수 있습니다.
      #[serde(default)] 
      pub tags: Vec<String>,
  }

  fn main() {
      let json_data = r#"{"id": 1, "name": "Apple"}"#;

      // JSON 문자열을 Rust 구조체로 역직렬화 (Pydantic 대체)
      // 타입이 맞지 않거나 필수 필드가 없으면 Err 반환
      let item: Item = serde_json::from_str(json_data).unwrap();
      
      println!("{:?}", item); 
      // 출력: Item { id: 1, name: "Apple", tags: [] }
  }
  ```

### Under the Hood: 메모리 레이아웃
파이썬의 인스턴스는 딕셔너리를 포인터로 들고 다니며 메모리에 흩뿌려집니다. 반면, 위의 `Item` Rust 구조체는 스택과 힙에 규칙적이고 단단하게(Packed) 포장됩니다. CPU는 이런 연속적이고 예측 가능한 데이터를 다룰 때 `Pydantic` 대비 수십 배~수백 배 빠른 속도로 파싱을 완료합니다.

다음 챕터에서는 파이썬 개발자가 가장 버리기 힘들어하는 OOP의 유산, **상속(Inheritance)**이 Rust에서 어떻게 **컴포지션(Composition)**과 **뉴타입 패턴(Newtype)**으로 완벽히 대체되는지 알아보겠습니다.
