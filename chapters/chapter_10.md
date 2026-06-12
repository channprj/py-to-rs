# Chapter 10. 열거형(Enums)을 활용한 상태 기계와 다형성

웹 서버를 만들거나 비즈니스 로직을 다룰 때, 우리는 데이터가 가질 수 있는 "상태(State)"나 "종류(Type)"를 모델링해야 합니다. 

파이썬 개발자들은 주로 다음과 같은 방식을 사용합니다.
1. `Literal`이나 `Enum`을 사용하고 `if-elif` 문으로 분기하기
2. `Union[TypeA, TypeB]`을 선언하고 `isinstance()`로 타입 검사하기
3. 부모 클래스를 상속받는 자식 클래스들을 여러 개 만들기 (다형성)

하지만 이런 방식들은 컴파일 타임(또는 Mypy 린팅 타임)에 **"모든 케이스를 빠짐없이 처리했는가(Exhaustiveness)"**를 완벽하게 보장하기 어렵습니다. 

Rust의 **열거형(Enum)**과 **패턴 매칭(Pattern Matching)**은 단순한 상수 집합이 아닙니다. 파이썬의 `Union` 타입과 상속 다형성을 완벽하게 런타임 오버헤드 없이 대체할 수 있는 가장 강력한 무기인 대수적 데이터 타입(Algebraic Data Types, ADT)입니다.

---

## 10.1 파이썬의 `Union`과 `isinstance()`의 한계

어떤 메시지 처리 시스템이 있다고 가정해 봅시다. 메시지에는 단순히 종료(Quit)를 알리는 것도 있고, 텍스트가 포함된 것도 있고, x/y 좌표가 포함된 것도 있습니다.

* **The Python Way (Dataclass + Union)**
  ```python
  from dataclasses import dataclass
  from typing import Union

  @dataclass
  class QuitMessage:
      pass

  @dataclass
  class WriteMessage:
      text: str

  @dataclass
  class MoveMessage:
      x: int
      y: int

  # 세 가지 타입을 하나로 묶음
  Message = Union[QuitMessage, WriteMessage, MoveMessage]

  def process_message(msg: Message):
      # 런타임에 타입을 하나하나 검사해야 함 (속도 저하, 누락 위험)
      if isinstance(msg, QuitMessage):
          print("Quit")
      elif isinstance(msg, WriteMessage):
          print(f"Write: {msg.text}")
      elif isinstance(msg, MoveMessage):
          print(f"Move to {msg.x}, {msg.y}")
      else:
          # 누군가 나중에 ChangeColorMessage를 추가하면 여기서 터집니다.
          raise ValueError("Unknown message type")
  ```

---

## 10.2 Rust의 Enum: 모든 것을 품는 대수적 타입

Rust의 Enum은 각 Variant(변형)마다 **서로 다른 타입과 크기의 데이터**를 담을 수 있습니다.

* **The Rust Implementation**
  ```rust
  // 세 가지 각기 다른 데이터 구조를 하나의 Enum으로 묶음
  enum Message {
      Quit,                       // 데이터가 없는 단위(Unit) 구조체와 동일
      Write(String),              // 튜플 구조체처럼 하나의 String을 가짐
      Move { x: i32, y: i32 },    // 일반 구조체처럼 익명 필드들을 가짐
  }

  // 이제 Message라는 단일 타입으로 모든 것을 처리할 수 있습니다.
  fn process_message(msg: Message) {
      // match 표현식은 if-elif-else의 완벽한 상위 호환입니다.
      match msg {
          Message::Quit => println!("Quit"),
          
          // Enum 내부의 데이터를 즉시 추출(Destructuring)하여 변수에 바인딩
          Message::Write(text) => println!("Write: {}", text),
          
          Message::Move { x, y } => println!("Move to {}, {}", x, y),
      }
  }
  ```

### `match` 구문의 강력함: 철저한 검사 (Exhaustiveness Checking)
파이썬 코드에서 `ChangeColorMessage`를 추가하고 `process_message` 함수를 수정하는 것을 깜빡했다면 런타임 에러가 발생합니다.

하지만 Rust의 `match` 구문은 컴파일러가 "모든 가능한 케이스를 처리했는지"를 강제로 검사합니다. 만약 `Message` Enum에 `ChangeColor(i32, i32, i32)`를 추가한 뒤 `match` 구문을 수정하지 않으면, **컴파일 자체가 실패**합니다. 

이 기능 하나만으로도 수백 개의 파일로 이루어진 대규모 프로젝트를 리팩토링할 때 엄청난 안정감을 줍니다. 컴파일러가 에러로 알려주는 곳만 따라가서 수정하면 버그가 없다는 것이 보장되기 때문입니다.

---

## 10.3 다형성의 두 가지 길: Enum vs Trait Object

"부모 클래스 대신 Enum을 써야 할까, 아니면 Trait을 써야 할까?" 파이썬 개발자들이 Rust 아키텍처를 설계할 때 가장 많이 하는 질문입니다.

**시나리오**: 다양한 UI 컴포넌트(버튼, 텍스트 필드)를 순회하며 `draw()`를 호출해야 합니다.

### 접근 1. Enum을 사용한 정적 디스패치 (Closed Set)
* **언제 쓰는가?**: 앞으로 추가될 UI 컴포넌트의 종류가 고정되어 있거나 거의 변하지 않을 때.
* **장점**: 런타임 속도가 가장 빠릅니다. 데이터가 스택이나 연속된 힙에 촘촘하게 저장됩니다.

```rust
enum UIComponent {
    Button { label: String },
    TextField { value: String },
}

impl UIComponent {
    fn draw(&self) {
        match self {
            UIComponent::Button { label } => println!("Draw Button: {}", label),
            UIComponent::TextField { value } => println!("Draw Text: {}", value),
        }
    }
}

fn main() {
    let components = vec![
        UIComponent::Button { label: "Submit".to_string() },
        UIComponent::TextField { value: "ID".to_string() },
    ];
    for c in components { c.draw(); }
}
```

### 접근 2. Trait Object를 사용한 동적 디스패치 (Open Set)
* **언제 쓰는가?**: 파이썬처럼 외부 라이브러리 사용자가 나만의 UI 컴포넌트를 만들어서 플러그인처럼 끼워 넣을 수 있게 허용해야 할 때 (Open-Closed Principle).
* **장점**: 확장이 자유롭습니다.
* **단점**: 파이썬의 덕 타이핑처럼 런타임에 메서드를 찾는 오버헤드(V-Table 참조)가 발생하고, 힙(Heap) 할당인 `Box`를 사용해야 합니다.

```rust
// 행동 정의
trait Draw {
    fn draw(&self);
}

struct Button { label: String }
impl Draw for Button {
    fn draw(&self) { println!("Draw Button: {}", self.label); }
}

struct TextField { value: String }
impl Draw for TextField {
    fn draw(&self) { println!("Draw Text: {}", self.value); }
}

fn main() {
    // Box<dyn Draw>: "Draw 트레이트를 구현한 어떤 타입이든 Heap에 저장하고 그 포인터를 갖겠다"
    let components: Vec<Box<dyn Draw>> = vec![
        Box::new(Button { label: "Submit".to_string() }),
        Box::new(TextField { value: "ID".to_string() }),
    ];

    // 런타임에 어떤 객체의 draw()인지 V-Table을 뒤져서 실행함 (동적 디스패치)
    for c in components {
        c.draw();
    }
}
```

파이썬은 구조적으로 무조건 2번 방식(동적 디스패치)으로만 동작합니다. 하지만 Rust에서는 90% 이상의 상황에서 성능이 훨씬 뛰어난 1번 방식(Enum)이나 제네릭 단일형화(Monomorphization)를 채택합니다. 

다음 챕터에서는 OOP의 마지막 유산, **다중 상속(Mixin)**을 Trait의 Default Method를 통해 어떻게 해결하는지, 그리고 길고 복잡한 생성자를 **Builder Pattern**으로 어떻게 대체하는지 살펴보겠습니다.
