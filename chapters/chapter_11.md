# Chapter 11. 믹스인(Mixin) 패턴과 생성(Creational) 패턴

![Chapter 11. 믹스인(Mixin) 패턴과 생성(Creational) 패턴](../images/chapter-11-patterns.jpg)

객체에 기능을 "끼워 넣어 본" 적이 있나요? Django로 뷰(View)를 짜다 보면 어느 순간 클래스 선언부가 이렇게 변합니다. `class OrderUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateModelMixin, GenericAPIView)`. 네 개의 부모 클래스가 한 줄에 늘어서 있고, 각각은 인증·권한·부분 업데이트·기본 라우팅이라는 **횡단 관심사(Cross-cutting Concern)**를 조각조각 들고 있습니다. 우리는 이 조각들을 다중 상속(Multiple Inheritance)으로 조립해 거대한 클래스를 만들어 냅니다. 편리합니다. 그리고 위험합니다.

위험한 이유는 두 가지입니다. 첫째, 저 네 클래스 중 두 개가 같은 이름의 메서드(`get_queryset` 같은)를 가지면, 실제로 어느 것이 호출될지는 파이썬의 **MRO(Method Resolution Order)**라는 C3 선형화 알고리즘이 결정합니다. 상속 순서를 한 칸만 바꿔도 동작이 달라지는데, 그걸 알아내려면 클래스 계층 전체를 머릿속에서 위상 정렬해야 합니다. 둘째, 이 모든 디스패치는 런타임에 `__mro__` 튜플을 따라 사전(`__dict__`)을 순회하며 이뤄집니다. 비싸고, 불투명하고, 정적 분석 도구(mypy)가 완벽히 추적하지 못합니다.

객체를 **생성하는** 쪽도 사정은 비슷합니다. 옵션이 많은 설정 객체를 만들 때 우리는 `def __init__(self, host, port=8080, *args, **kwargs)` 같은 시그니처에 의존합니다. 호출하는 쪽은 `ServerConfig("localhost", tls=True, retry=5)`처럼 키워드 인자를 골라 넘기죠. 우아해 보이지만, `**kwargs`는 타입 체커에게는 블랙박스입니다. 오타(`tls`를 `tsl`로)를 내도 조용히 무시되고, 어떤 옵션이 유효한지는 생성자 본문을 읽어야만 알 수 있습니다.

Rust에는 상속이 없습니다. 기본 인자도, `*args`도, `**kwargs`도 없습니다. 파이썬 개발자에게는 무기를 다 빼앗긴 느낌일 겁니다. 하지만 이 "결핍"이야말로 Rust 설계의 핵심입니다. Rust는 **데이터(Struct)의 상속**과 **행동(Trait)의 재사용**을 철저히 분리하고, 객체 생성을 **타입 시스템으로 검증되는 명시적 절차**로 끌어올립니다. 이 장에서는 파이썬의 Mixin이 Rust에서 트레이트 기본 메서드(Default Method)와 블랭킷 구현(Blanket impl)으로 어떻게 더 안전하게 환생하는지, 그리고 복잡한 `__init__`이 빌더 패턴(Builder Pattern) — 특히 필수 필드 누락을 **컴파일 타임에 차단하는** 타입 상태(Typestate) 빌더 — 으로 어떻게 우아하게 풀리는지를 해부합니다.

---

## 11.1 The Python Way — 다중 상속 Mixin과 MRO의 늪

파이썬의 Mixin은 "상태는 거의 없고 행동만 제공하는, 단독으로는 인스턴스화하지 않는 클래스"를 다중 상속으로 끼워 넣는 패턴입니다. 가장 흔한 예가 로깅, 직렬화, 타임스탬프 관리 같은 횡단 관심사입니다.

* **The Python Way (Mixin 조립)**
  ```python
  from __future__ import annotations
  import json
  from datetime import datetime

  class TimestampMixin:
      created_at: datetime

      def age_seconds(self) -> float:
          return (datetime.now() - self.created_at).total_seconds()

  class LoggableMixin:
      def log_state(self) -> None:
          print(f"[{self.__class__.__name__}] state logged")

  class SerializableMixin:
      def to_json(self) -> str:
          # __dict__ 를 통째로 직렬화 — 어떤 속성이 있는지 런타임에야 안다
          return json.dumps(self.__dict__, default=str)

  class User(TimestampMixin, LoggableMixin, SerializableMixin):
      def __init__(self, username: str) -> None:
          self.username = username
          self.created_at = datetime.now()

  u = User("pythonista")
  u.log_state()
  print(u.to_json())
  ```

  이 코드는 잘 동작합니다. `User`는 자기 코드를 한 줄도 안 쓰고 `age_seconds`, `log_state`, `to_json` 세 행동을 얻었습니다. 문제는 이 편의성이 숨기고 있는 비용입니다.

**비용 1 — MRO의 비결정성(체감상).** Mixin 두 개가 우연히 같은 메서드 이름을 쓰면, 호출되는 것은 MRO 순서상 먼저 오는 쪽입니다. C3 선형화가 수학적으로는 결정적이지만, 사람이 머릿속에서 추적하기는 어렵습니다.

```python
class A:
    def process(self) -> str:
        return "A"

class B:
    def process(self) -> str:
        return "B"

class C(A, B):  # A가 B보다 먼저 → A.process 가 이긴다
    pass

print(C().process())     # "A"
print(C.__mro__)
# (<class 'C'>, <class 'A'>, <class 'B'>, <class 'object'>)
```

상속 순서를 `class C(B, A)`로 바꾸면 결과가 `"B"`로 뒤집힙니다. Mixin이 5~6개 쌓이고 그중 일부가 `super().__init__()`을 호출하는 협력적(cooperative) 상속을 하면, 어느 `__init__`이 어떤 순서로 불리는지 추적하는 것은 디버깅 지옥이 됩니다.

그리고 `super()`는 흔한 오해와 달리 "부모 클래스"를 가리키지 않습니다. **MRO 상에서 현재 클래스의 다음 클래스**를 가리킵니다. 즉 `super().__init__()`이 실제로 어디로 가는지는 `self`의 실제 타입에 달려 있어서, 같은 Mixin이라도 어떤 클래스 조합에 끼워졌느냐에 따라 다른 곳으로 점프합니다. 협력적 다중 상속에서 모든 `__init__`이 `**kwargs`를 받아 `super().__init__(**kwargs)`로 떠넘기는 관용구가 생긴 것도 이 비결정성 때문입니다. 한 단계라도 `super()` 호출을 빠뜨리면 체인이 끊겨 일부 Mixin의 초기화가 조용히 건너뛰어집니다. 이것은 "어떤 클래스를 어떤 순서로 상속하느냐"라는 *호출부의 결정*이 *각 Mixin의 동작*을 바꾼다는 뜻이며, 캡슐화의 정면 위반입니다.

**비용 2 — 묵시적 의존성.** `TimestampMixin.age_seconds`는 `self.created_at`이 존재한다고 가정합니다. 하지만 그 속성은 `User.__init__`에서 설정됩니다. Mixin 자신은 그 필드를 보장하지 않습니다. `created_at`을 설정하지 않은 클래스에 이 Mixin을 붙이면, 호출하는 순간까지 멀쩡하다가 런타임에 `AttributeError`로 터집니다. 즉 **"이 Mixin을 쓰려면 무엇이 전제되어야 하는가"**가 코드 어디에도 명시되어 있지 않습니다.

**비용 3 — 런타임 디스패치 오버헤드.** `u.log_state()`를 호출하면 인터프리터는 `type(u).__mro__`를 순회하며 각 클래스의 `__dict__`에서 `log_state`를 찾습니다. 모든 메서드 호출이 이 해시 탐색을 거칩니다. Chapter 4에서 본 `PyObject` 헤더와 마찬가지로, 이것은 동적 언어가 지불하는 보이지 않는 세금입니다. CPython 3.11+가 도입한 인라인 캐시(adaptive specializing interpreter)가 반복 호출에서 이 탐색을 일부 단축하긴 하지만, MRO가 깊을수록(상속 6단계라면 최악의 경우 6번의 사전 조회) 첫 조회 비용과 캐시 무효화 비용은 그대로 남습니다. 반면 Rust의 트레이트 기본 메서드 호출은 다음 절에서 보듯 컴파일 타임에 한 함수로 확정되어, 탐색이라는 개념 자체가 없습니다.

이제 같은 문제를 Rust가 어떻게 바라보는지 봅시다.

---

## 11.2 The Rust Mental Model — 상속이 아니라 합성된 능력(Capability)

Rust의 사고방식은 이렇게 요약됩니다. **"객체는 데이터를 상속하지 않는다. 대신 능력(트레이트)을 구현(implement)한다."**

파이썬에서 `class User(LoggableMixin)`은 두 가지를 동시에 의미합니다. (1) `User`는 `LoggableMixin`의 메서드를 물려받고, (2) `User`는 `LoggableMixin`의 *서브타입*이 되어 `isinstance(u, LoggableMixin)`이 `True`가 됩니다. 즉 데이터 재사용과 타입 관계가 한 덩어리로 엉겨 있습니다.

Rust는 이 둘을 분리합니다. Chapter 8에서 본 것처럼 Rust에 데이터 상속은 없습니다. 대신 Chapter 9에서 배운 트레이트(Trait)가 "행동의 인터페이스"를 정의하고, 트레이트는 **기본 메서드(Default Method)**를 가질 수 있습니다. 어떤 타입이 트레이트를 구현하면, 그 기본 메서드를 공짜로 얻습니다. 이것이 파이썬 Mixin의 "행동 재사용" 측면을 그대로 대체합니다.

차이는 디스패치 방식에 있습니다. 11.1의 비용 3에서 본 런타임 MRO 순회 대신, Rust는 **단일형화(Monomorphization)**를 통해 컴파일 타임에 어느 함수를 호출할지 못 박습니다. 호출 비용은 일반 함수 호출과 동일합니다 — 즉, 무비용 추상화(Zero-cost Abstraction)입니다.

이 분리가 주는 더 깊은 이점은 **선형성**입니다. 파이썬에서 `class User(A, B, C, D)`는 네 부모를 하나의 선형 MRO로 녹여 넣고, 그 안에서 누가 누구를 덮어쓰는지가 순서에 따라 결정됩니다. Rust에서 `User`에 `impl A`, `impl B`, `impl C`, `impl D`를 다는 것은 네 개의 **독립적이고 직교하는(orthogonal)** 능력 부여이며, 서로 순서가 없습니다. 어느 트레이트의 메서드를 부를지는 호출 지점의 타입 문맥이 결정하지, 선언 순서가 결정하지 않습니다. 충돌이 있으면(11.3.1) 컴파일러가 "당신이 직접 골라라"라고 요구할 뿐, 임의의 순서로 묵시적 승자를 정하지 않습니다. "조립의 결과가 조립 순서에 의존하지 않는다"는 이 성질이, 파이썬 Mixin이 커질수록 무너지던 추론 가능성을 지켜 줍니다.

| Python (다중 상속 Mixin) | Rust (Trait + 기본 메서드) | 핵심 차이 |
|---|---|---|
| `class C(A, B)` | `impl A for C {}` `impl B for C {}` | 데이터 상속 없음, 능력만 구현 |
| MRO(C3 선형화)로 메서드 결정 | 호출 지점에서 트레이트 명시 가능 | 이름 충돌 시 컴파일 에러 |
| 이름 충돌 → 조용히 덮어씀 | 이름 충돌 → `error[E0034]` | 모호함을 컴파일러가 잡음 |
| 묵시적 전제(`self.created_at`) | `supertrait`/`required method`로 명시 | 의존성이 시그니처에 드러남 |
| 런타임 `__dict__` 순회 디스패치 | 정적 디스패치(단일형화) | 무비용, 인라이닝 가능 |
| `*args/**kwargs`로 유연한 생성 | 빌더 패턴 / `Default` / `From` | 타입으로 검증되는 명시적 생성 |
| 필수 인자 누락 → 런타임 `TypeError` | 타입 상태 빌더 → 컴파일 에러 | 누락이 빌드를 막음 |

이 표의 마지막 두 줄, 즉 객체 생성 부분은 11.5부터 다룹니다. 먼저 Mixin의 대체재인 기본 메서드부터 봅시다.

---

## 11.3 Mixin = Trait + 기본 메서드

11.1의 `LoggableMixin`을 Rust로 옮겨 봅시다. 핵심은 트레이트가 **반드시 구현해야 하는 메서드(required method)**와 **기본 구현이 딸린 메서드(default method)**를 모두 가질 수 있다는 점입니다.

* **The Rust Implementation**
  ```rust
  pub trait Loggable {
      // required method: 구현체가 반드시 채워야 한다 (Mixin의 "전제")
      fn name(&self) -> &str;

      // default method: 구현체가 오버라이드하지 않으면 이 구현이 쓰인다 (Mixin 본체)
      fn log_state(&self) {
          println!("[{}] state logged", self.name());
      }
  }

  pub struct User {
      pub username: String,
  }

  impl Loggable for User {
      fn name(&self) -> &str {
          &self.username    // required method만 채우면 된다
      }
      // log_state()는 기본 구현을 그대로 재사용
  }

  fn main() {
      let u = User { username: "rustacean".into() };
      u.log_state();        // [rustacean] state logged
  }
  ```

여기서 파이썬 Mixin의 "비용 2"(묵시적 의존성)가 어떻게 해소되는지 보세요. `Loggable`은 `log_state`가 동작하려면 `name()`이 필요하다는 사실을 `fn name(&self) -> &str;`라는 **시그니처로 명시**합니다. `User`가 이 트레이트를 구현하려면 컴파일러가 `name()` 구현을 *강제*합니다. 빠뜨리면 컴파일이 안 됩니다. 파이썬처럼 "호출하는 순간 `AttributeError`"가 날 일이 없습니다. 전제 조건이 코드에 박제되는 것입니다.

### 11.3.1 이름 충돌 — 파이썬은 덮어쓰고, Rust는 멈춘다

11.1 비용 1의 MRO 충돌을 Rust에서 재현해 봅시다. 두 트레이트가 같은 이름의 메서드를 제공하고, 한 타입이 둘 다 구현하면 어떻게 될까요?

```rust
trait Pilot {
    fn fly(&self) { println!("Pilot 비행"); }
}
trait Wizard {
    fn fly(&self) { println!("Wizard 비행"); }
}

struct Human;
impl Pilot for Human {}
impl Wizard for Human {}

fn main() {
    let h = Human;
    h.fly();   // 🚨 컴파일 에러! 어느 trait의 fly인가?
}
```

파이썬이라면 MRO 순서상 먼저 오는 쪽이 조용히 이깁니다. Rust는 거부합니다.

```text
error[E0034]: multiple applicable items in scope
  --> src/main.rs:10:7
   |
10 |     h.fly();   // 🚨 컴파일 에러! 어느 trait의 fly인가?
   |       ^^^ multiple `fly` found
   |
note: candidate #1 is defined in an impl of the trait `Pilot` for the type `Human`
note: candidate #2 is defined in an impl of the trait `Wizard` for the type `Human`
help: disambiguate the method for candidate #1
   |
10 -     h.fly();
10 +     Pilot::fly(&h);
   |
```

컴파일러는 모호함을 발견하고 **멈춥니다.** 그리고 친절하게 해결책까지 제시합니다 — `Pilot::fly(&h)` 또는 `Wizard::fly(&h)`처럼 **완전 정규화 구문(Fully Qualified Syntax)**으로 어느 쪽인지 명시하라는 것입니다. 이것이 파이썬 MRO의 "비결정성 체감"과의 결정적 차이입니다. Rust에서는 충돌이 절대 조용히 묻히지 않습니다.

```rust
fn main() {
    let h = Human;
    Pilot::fly(&h);          // Pilot 비행
    Wizard::fly(&h);         // Wizard 비행
    <Human as Pilot>::fly(&h); // 더 명시적인 형태
}
```

### 11.3.2 트레이트 합성: Supertrait

11.1의 `TimestampMixin`은 `created_at` 속성을 전제로 했습니다. 더 나아가 "이 능력을 가지려면 먼저 저 능력을 가져야 한다"는 의존 관계 자체를 표현하고 싶을 때가 있습니다. Rust는 **상위 트레이트(Supertrait)**로 이를 표현합니다. `trait B: A`는 "B를 구현하려면 A도 구현해야 한다"는 뜻입니다.

* **The Rust Implementation (Supertrait 합성)**
  ```rust
  pub trait Identifiable {
      fn id(&self) -> u64;
  }

  // Auditable을 구현하려면 Identifiable과 Debug를 먼저 갖춰야 한다.
  // 즉 audit()의 기본 구현이 id()와 {:?}를 안전하게 쓸 수 있다.
  pub trait Auditable: Identifiable + std::fmt::Debug {
      fn audit(&self) {
          println!("[AUDIT id={}] {:?}", self.id(), self);
      }
  }

  #[derive(Debug)]
  struct Order {
      order_id: u64,
      total: u64,
  }

  impl Identifiable for Order {
      fn id(&self) -> u64 { self.order_id }
  }
  impl Auditable for Order {}   // audit()는 기본 구현 그대로

  fn main() {
      let o = Order { order_id: 1001, total: 50_000 };
      o.audit();   // [AUDIT id=1001] Order { order_id: 1001, total: 50000 }
  }
  ```

`Auditable: Identifiable + std::fmt::Debug`라는 한 줄이 파이썬 Mixin의 묵시적 전제를 **타입 제약으로 박제**합니다. `Identifiable`을 구현하지 않은 타입에 `impl Auditable`을 달려고 하면 `error[E0277]`로 거부당합니다. 파이썬에서 `created_at` 없는 객체에 `TimestampMixin`을 붙였다가 런타임에 터지던 것과 정반대입니다.

### 11.3.3 블랭킷 구현(Blanket impl): "모든 T에 자동 적용"

여기서 Rust만의 진짜 강력한 무기가 나옵니다. 파이썬 Mixin은 *상속받은 클래스에만* 적용됩니다. 그런데 Rust에서는 "특정 조건을 만족하는 **모든 타입**에 어떤 능력을 한 번에 부여"할 수 있습니다. 이것이 **블랭킷 구현(Blanket Implementation)**입니다.

* **The Rust Implementation (Blanket impl)**
  ```rust
  use std::fmt::Debug;

  pub trait Loggable {
      fn log(&self);
  }

  // 핵심: Debug를 구현한 "모든 T"에 대해 Loggable을 자동으로 구현한다.
  impl<T: Debug> Loggable for T {
      fn log(&self) {
          println!("[LOG] {:?}", self);
      }
  }

  #[derive(Debug)]
  struct User {
      name: String,
      age: u32,
  }

  fn main() {
      let u = User { name: "rustacean".into(), age: 30 };
      u.log();              // User에 impl을 단 적이 없는데 호출됨
      42i32.log();          // i32에도 자동 적용
      "hello".log();        // &str에도
      vec![1, 2, 3].log();  // Vec<i32>에도
  }
  ```

  출력:
  ```text
  [LOG] User { name: "rustacean", age: 30 }
  [LOG] 42
  [LOG] "hello"
  [LOG] [1, 2, 3]
  ```

`impl<T: Debug> Loggable for T`는 "`Debug`를 만족하는 모든 `T`는 자동으로 `Loggable`이다"라고 선언합니다. `User`, `i32`, `&str`, `Vec<i32>` 어느 곳에도 `Loggable` 구현을 직접 쓰지 않았는데, 전부 `.log()`를 얻습니다. 파이썬으로 치면 "`__repr__`을 가진 모든 클래스에 `log()` 메서드를 자동 주입"하는 것과 같지만, 파이썬에서는 메타클래스나 몽키패칭으로나 흉내 낼 수 있고 타입 안전하지도 않습니다.

표준 라이브러리도 이 기법을 정확히 이렇게 씁니다. `From`을 구현하면 `Into`가 **공짜로** 따라오는데, 그 정체가 바로 블랭킷 구현입니다.

```rust
// std 내부 (개념적 형태)
impl<T, U: From<T>> Into<U> for T {
    fn into(self) -> U { U::from(self) }
}
```

`From<T> for U`만 작성하면, `Into<U> for T`는 위 블랭킷 impl이 자동으로 채워 줍니다. 그래서 11.7에서 보게 될 `let f: Fahrenheit = c.into();`가 `From` 하나만으로 동작하는 것입니다.

> **Under the Hood: 블랭킷 impl과 기본 메서드의 단일형화**
>
> `42i32.log()`와 `vec![1,2,3].log()`는 같은 `impl<T: Debug> Loggable for T` 블록에서 나오지만, 컴파일러는 이를 **하나의 제네릭 함수로 남겨 두지 않습니다.** 호출되는 구체 타입마다 별도의 함수를 찍어냅니다. 이것이 **단일형화(Monomorphization)**입니다.
>
> ```text
> 소스:   impl<T: Debug> Loggable for T { fn log(&self) {...} }
>
> 컴파일 후 (개념):
>   Loggable_log_for_i32(&self: &i32)        { println!("[LOG] {:?}", self); }
>   Loggable_log_for_User(&self: &User)      { println!("[LOG] {:?}", self); }
>   Loggable_log_for_VecI32(&self: &Vec<i32>){ println!("[LOG] {:?}", self); }
> ```
>
> 각 호출 지점은 정확히 자기 타입에 맞는 함수로 **정적으로 연결**됩니다. 런타임에 "이 객체의 타입이 뭐지?"를 묻지 않습니다 — 파이썬의 `__mro__` 순회가 사라진 자리입니다. 게다가 함수 본문이 짧으면 LLVM이 호출 자체를 **인라이닝(inlining)**해 버려, `u.log()`는 결국 `println!` 한 줄로 펼쳐집니다. 트레이트라는 추상화 계층이 어셈블리에서는 흔적도 없이 증발합니다.
>
> 대가는 코드 크기입니다. 100가지 타입이 `log()`를 호출하면 100개의 기계어 함수가 생깁니다(코드 비대화, code bloat). Chapter 10에서 본 정적 디스패치 vs 동적 디스패치의 트레이드오프가 여기서도 동일하게 적용됩니다. 코드 크기가 정말 문제라면 `&dyn Loggable` 트레이트 객체로 동적 디스패치를 택해 함수 하나만 남길 수 있습니다.
>
> 수치로 감을 잡아 봅시다. "필드 네 개짜리 객체를 만들고 그 필드를 읽는" 작업을 1,000만 번 반복하는 마이크로벤치마크를 직접 측정했습니다(2026-06, Apple Silicon, Python 3.13.6 / rustc 1.90.0 `-O`).
>
> ```text
> CPython 3.13.6  (class + __slots__)  : 1,460 ms   (146 ns/객체)
> CPython 3.13.6  (plain class, __dict__): 1,690 ms   (169 ns/객체)
> Rust 1.90.0     (필드 직접 대입)       :   7.8 ms   (0.78 ns/객체)
> Rust 1.90.0     (타입 상태 빌더 경유)   :   7.9 ms   (0.79 ns/객체)
> ```
>
> CPython은 객체당 힙 할당 + 참조 카운트 갱신 + 인터프리터 디스패치를 매번 치르므로 객체 하나에 146~169나노초가 듭니다. 같은 의미의 Rust 코드는 스택 구성 + 인라이닝으로 펼쳐져 객체당 0.8나노초 미만입니다 — 이 CPU 핫 루프에서는 **약 187~217배** 차이입니다(`__slots__` 대비 1460/7.8≈187, plain class 대비 1690/7.8≈217). 일반적인 객체 생성에서 "수십 배", 이처럼 할당·디스패치가 지배하는 핫 루프에서는 "100배 이상"으로 벌어집니다. 측정 환경·객체 크기·할당 패턴에 따라 배수는 달라지므로 절대 수치보다 **자릿수의 격차**로 기억하세요.
>
> 결정적인 대목은 마지막 두 줄입니다. **타입 상태 빌더를 경유한 Rust(7.9 ms)와 필드를 직접 대입한 Rust(7.8 ms)가 측정 오차 안에서 동일합니다.** 빌더/트레이트라는 추상화 계층이 이 격차의 *원인이 아니라*, 컴파일 타임에 제거되어 격차에 *기여조차 하지 않는다*는 것이 핵심입니다. 이 "빌더가 정말로 무비용인가"는 11.6.2에서 어셈블리 수준으로 증명합니다.

---

## 11.4 복잡한 `__init__`의 문제

이제 생성 패턴으로 넘어갑니다. 파이썬의 유연한 생성자를 다시 봅시다.

* **The Python Way (kwargs 지옥)**
  ```python
  class ServerConfig:
      def __init__(
          self,
          host: str,
          port: int = 8080,
          timeout: int = 30,
          tls: bool = False,
          max_connections: int = 100,
          **kwargs,
      ) -> None:
          self.host = host
          self.port = port
          self.timeout = timeout
          self.tls = tls
          self.max_connections = max_connections
          # 알 수 없는 옵션은 조용히 흡수된다
          for k, v in kwargs.items():
              setattr(self, k, v)

  # 옵션을 골라서 키워드로 전달 — 우아해 보인다
  config = ServerConfig("localhost", tls=True, max_connections=500)

  # 그러나 오타는 잡히지 않는다
  oops = ServerConfig("localhost", tsl=True)  # tls의 오타 → 조용히 무시
  print(hasattr(oops, "tsl"))  # True (setattr로 엉뚱한 속성이 생김)
  print(oops.tls)              # False (의도와 정반대!)
  ```

이 코드의 함정을 정리하면:

1. **오타가 침묵한다.** `tsl=True`는 `TypeError`를 내지 않고 `setattr`로 흡수되어 `oops.tsl` 속성이 됩니다. 정작 `oops.tls`는 기본값 `False`로 남습니다. 의도와 정반대로 동작하는데 아무 경고가 없습니다.
2. **유효 옵션이 시그니처에 없다.** `**kwargs` 때문에 "이 설정이 받는 옵션이 무엇인가"를 시그니처만 보고는 알 수 없습니다. 생성자 본문을 읽거나 문서를 뒤져야 합니다.
3. **부분적으로 구성된 객체.** 필수 검증(예: `port`가 1~65535인지)을 생성자에서 빠뜨리면, 잘못된 상태의 객체가 그대로 돌아다닙니다.

Rust에는 기본 인자도 `**kwargs`도 없으니, 같은 `ServerConfig`를 만들려면 다섯 개 인자를 전부 넘기는 `fn new(host, port, timeout, tls, max_connections)` 같은 함수를 써야 할 것 같습니다. 호출부는 `ServerConfig::new("localhost", 8080, 30, false, 100)`처럼 의미 없는 위치 인자의 나열이 됩니다 — 세 번째 `30`이 뭔지, 네 번째 `false`가 뭔지 호출부만 봐서는 알 수 없습니다. 이것이 바로 빌더 패턴이 등장하는 이유입니다.

---

## 11.5 빌더 패턴의 세 가지 스타일

빌더 패턴은 "객체를 한 번에 생성하지 않고, 옵션을 하나씩 설정한 뒤 마지막에 조립(`build`)한다"는 발상입니다. 파이썬의 키워드 인자가 주던 "필요한 것만 명시적으로" 경험을, 타입 안전성을 더해 재현합니다. Rust에는 세 가지 스타일이 있고, 각각 트레이드오프가 다릅니다.

### 11.5.1 소비형(Consuming) 빌더 — `self`를 소비하는 표준 스타일

가장 흔한 형태입니다. 각 설정 메서드가 `self`를 통째로 받아(소유권을 가져와서) 수정한 뒤 다시 `Self`로 돌려줍니다. 그래서 메서드 체이닝(`.port(443).tls(true)`)이 자연스럽게 이어집니다.

```rust
#[derive(Debug)]
pub struct ServerConfig {
    host: String,
    port: u16,
    timeout: u32,
    tls: bool,
}

pub struct ServerConfigBuilder {
    host: String,
    port: u16,
    timeout: u32,
    tls: bool,
}

impl ServerConfig {
    // 필수 파라미터(host)만 받는 빌더 시작점
    pub fn builder(host: &str) -> ServerConfigBuilder {
        ServerConfigBuilder {
            host: host.into(),
            port: 8080,     // 기본값 — 파이썬의 port=8080 자리
            timeout: 30,
            tls: false,
        }
    }
}

impl ServerConfigBuilder {
    // self를 소비하고 Self를 반환 → 체이닝 가능
    pub fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }
    pub fn timeout(mut self, timeout: u32) -> Self {
        self.timeout = timeout;
        self
    }
    pub fn tls(mut self, tls: bool) -> Self {
        self.tls = tls;
        self
    }
    pub fn build(self) -> ServerConfig {
        ServerConfig {
            host: self.host,
            port: self.port,
            timeout: self.timeout,
            tls: self.tls,
        }
    }
}

fn main() {
    // 파이썬의 키워드 인자처럼 명시적이고, 타입은 컴파일러가 검증한다
    let config = ServerConfig::builder("localhost")
        .port(443)
        .tls(true)
        .build();
    println!("{:?}", config);
    // ServerConfig { host: "localhost", port: 443, timeout: 30, tls: true }
}
```

`.port(443)`은 호출부만 봐도 "포트를 443으로"임이 명확합니다. 11.4의 `ServerConfig::new("localhost", 8080, 30, false, 100)`에서 위치 인자 `false`가 뭐였는지 헷갈리던 문제가 사라집니다. 그리고 `tsl`(오타)이라는 메서드는 존재하지 않으므로, 오타를 내면 즉시 `error[E0599]: no method named 'tsl'`로 컴파일이 막힙니다. 파이썬의 침묵하는 오타가 Rust에서는 빌드 실패가 됩니다.

### 11.5.2 가변 참조형(`&mut self`) 빌더 — 조건부 설정에 유리

소비형 빌더는 `self`를 소비하기 때문에, 루프나 조건문 안에서 빌더를 점진적으로 구성하기에 어색할 때가 있습니다. 이럴 때는 `&mut self`를 받아 `&mut Self`를 돌려주는 스타일이 편합니다.

```rust
#[derive(Debug)]
pub struct ServerConfig {
    host: String,
    port: u16,
    tls: bool,
}

pub struct MutBuilder {
    host: String,
    port: u16,
    tls: bool,
}

impl MutBuilder {
    pub fn new(host: &str) -> Self {
        Self { host: host.into(), port: 8080, tls: false }
    }
    // &mut self → &mut Self : 소유권을 가져가지 않는다
    pub fn port(&mut self, p: u16) -> &mut Self {
        self.port = p;
        self
    }
    pub fn tls(&mut self, t: bool) -> &mut Self {
        self.tls = t;
        self
    }
    // build는 &self만 빌려 ServerConfig를 만든다 (빌더를 여러 번 재사용 가능)
    pub fn build(&self) -> ServerConfig {
        ServerConfig {
            host: self.host.clone(),
            port: self.port,
            tls: self.tls,
        }
    }
}

fn main() {
    let mut builder = MutBuilder::new("localhost");
    builder.port(9000);

    // 조건부 설정이 자연스럽다 — 소비형이라면 self를 다시 받아야 해서 번거롭다
    let production = std::env::var("ENV").as_deref() == Ok("prod");
    if production {
        builder.tls(true);
    }

    let config = builder.build();
    println!("{:?}", config);
}
```

소비형 빌더의 `if production { builder = builder.tls(true); }`는 반환값을 다시 변수에 받아야 해서 어색합니다. `&mut self` 스타일은 `builder.tls(true);`만으로 끝납니다. 대신 체이닝의 최종 결과를 곧장 함수 인자로 넘기기엔(`use_config(MutBuilder::new("x").port(1).build())`) 임시 값의 수명 때문에 불편할 수 있습니다. 둘은 우열이 아니라 용도 차이입니다. **체이닝이 주된 사용 패턴이면 소비형**, **조건부·반복적 구성이 주된 패턴이면 `&mut self`**를 택합니다.

두 스타일의 더 본질적인 차이는 **소유권(Ownership)**에 있습니다(Chapter 4·5의 연장입니다). 소비형은 각 setter가 `self`의 소유권을 가져갔다가 돌려주므로, 빌더를 "마지막에 딱 한 번 `build`로 소비하고 끝내는" 일회성 흐름에 맞습니다. 중간에 빌더를 다른 곳에 빌려주거나 두 번 쓰려 하면 "use after move"(Chapter 4)에 걸립니다. 반대로 `&mut self` 스타일은 빌더의 소유권을 호출자가 계속 쥐고 있으므로, 같은 빌더로 설정을 바꿔 가며 여러 개의 `ServerConfig`를 찍어내는 것도 가능합니다(위 예제의 `build(&self)`가 `&self`만 빌리는 이유입니다). 표준 라이브러리의 `std::process::Command`가 바로 이 `&mut self` 스타일로, `Command::new("ls").arg("-l").arg("/tmp").spawn()`처럼 씁니다.

### 11.5.3 `derive_builder` 크레이트 — 보일러플레이트 박멸

11.5.1의 빌더는 동작하지만, `ServerConfigBuilder` 구조체와 메서드들이 본체 구조체의 필드를 그대로 복제한 보일러플레이트입니다. 필드를 하나 추가할 때마다 빌더에도 메서드를 추가해야 합니다. 실무에서는 [`derive_builder`](https://crates.io/crates/derive_builder) 크레이트가 이 코드를 **컴파일 타임에 매크로로 자동 생성**합니다(Chapter 19에서 절차적 매크로의 원리를 다룹니다).

```toml
# Cargo.toml
[dependencies]
derive_builder = "0.20"
```

```rust
use derive_builder::Builder;

#[derive(Debug, Builder)]
#[builder(setter(into))]    // setter가 &str → String 같은 변환을 자동 처리
pub struct ServerConfig {
    host: String,
    #[builder(default = "8080")]
    port: u16,
    #[builder(default = "30")]
    timeout: u32,
    #[builder(default = "false")]
    tls: bool,
}

fn main() {
    // ServerConfigBuilder 가 자동 생성되어 있다
    let config = ServerConfigBuilder::default()
        .host("localhost")
        .port(443u16)
        .tls(true)
        .build()             // 반환 타입은 Result<ServerConfig, ServerConfigBuilderError>
        .expect("필수 필드 누락");

    println!("{:?}", config);
}
```

`#[derive(Builder)]` 한 줄이 `ServerConfigBuilder` 구조체, 모든 setter, 그리고 `build()`까지 찍어냅니다. 다만 기본 `derive_builder`의 `build()`는 **런타임에** 필수 필드 누락을 검사하여 `Result`로 돌려줍니다(필드를 빠뜨리면 `Err`). 즉 누락이 `expect`/`?` 지점까지 통과합니다. "누락을 아예 **컴파일 타임에** 막고 싶다"면 다음 절의 타입 상태 빌더가 답입니다.

---

## 11.6 Typestate 빌더 — 필수 필드 누락을 컴파일 타임에 차단하기

11.5의 빌더들은 모두 한 가지 빈틈이 있습니다. `build()`를 호출했는데 필수 필드를 설정하지 않았다면? 소비형 빌더에서 `host`를 `Option`으로 두면 `build()` 안에서 `.unwrap()`을 해야 하고, 누락 시 런타임 패닉이 납니다. `derive_builder`는 런타임 `Err`를 돌려줍니다. 둘 다 **런타임**에야 문제를 발견합니다.

진짜 Rust다운 해법은 "필수 필드를 채우지 않으면 `build()` 메서드 자체가 **존재하지 않게** 만드는 것"입니다. 타입 시스템에 "지금까지 어떤 필드가 채워졌는가"를 인코딩하는 이 기법을 **타입 상태(Typestate)** 패턴이라 부릅니다.

핵심 아이디어: 빌더에 타입 파라미터를 달아서, 각 필수 필드의 상태(`Missing` 또는 `Set`)를 타입으로 표현합니다. `build()`는 모든 필수 필드가 `Set`인 타입에만 구현합니다.

이 발상의 뿌리는 "프로그램의 상태를 런타임 플래그가 아니라 타입으로 표현하면, 잘못된 상태 전이를 컴파일러가 막아 준다"는 것입니다. 파이썬이라면 빌더 객체 안에 `self._url_set = False` 같은 불리언 플래그를 두고 `build()`에서 `if not self._url_set: raise ...`로 검사했을 것입니다. 그러나 그 검사는 **실행되어야만** 발동합니다 — 테스트가 그 경로를 밟지 않으면 버그가 프로덕션까지 살아남습니다. 타입 상태는 그 플래그를 *타입 파라미터*로 승격시켜, 검사를 "실행 시점의 if"에서 "컴파일 시점의 메서드 존재 여부"로 옮깁니다. 잘못된 상태의 빌더에는 `build`라는 메서드가 애초에 달려 있지 않으므로, 그 코드는 빌드 산출물에 도달조차 못 합니다.

```rust
use std::marker::PhantomData;

// 타입 상태 마커 — 크기 0인 ZST(Zero-Sized Type). 런타임 데이터가 전혀 없다.
pub struct Missing;
pub struct Set;

#[derive(Debug)]
pub struct HttpRequest {
    url: String,
    method: String,
    body: Option<String>,
}

// 타입 파라미터 U, M이 "url/method가 채워졌는지"를 타입에 인코딩한다.
pub struct HttpRequestBuilder<U, M> {
    url: Option<String>,
    method: Option<String>,
    body: Option<String>,
    _marker: PhantomData<(U, M)>,
}

impl HttpRequestBuilder<Missing, Missing> {
    pub fn new() -> Self {
        HttpRequestBuilder {
            url: None,
            method: None,
            body: None,
            _marker: PhantomData,
        }
    }
}

// url()/method()/body()는 어떤 상태에서든 호출 가능하다.
impl<U, M> HttpRequestBuilder<U, M> {
    // url()을 호출하면 첫 번째 상태만 Set으로 전이한다 (U → Set)
    pub fn url(self, url: &str) -> HttpRequestBuilder<Set, M> {
        HttpRequestBuilder {
            url: Some(url.into()),
            method: self.method,
            body: self.body,
            _marker: PhantomData,
        }
    }
    // method()는 두 번째 상태만 Set으로 전이한다 (M → Set)
    pub fn method(self, method: &str) -> HttpRequestBuilder<U, Set> {
        HttpRequestBuilder {
            url: self.url,
            method: Some(method.into()),
            body: self.body,
            _marker: PhantomData,
        }
    }
    // body는 선택 필드라 상태 전이가 없다
    pub fn body(mut self, body: &str) -> Self {
        self.body = Some(body.into());
        self
    }
}

// build()는 오직 <Set, Set> 상태에만 구현한다 → 필수 필드가 다 차야만 존재한다
impl HttpRequestBuilder<Set, Set> {
    pub fn build(self) -> HttpRequest {
        HttpRequest {
            url: self.url.unwrap(),       // 타입이 채워짐을 보장 → unwrap이 절대 안 터진다
            method: self.method.unwrap(),
            body: self.body,
        }
    }
}

fn main() {
    let req = HttpRequestBuilder::new()
        .url("https://api.example.com")
        .method("POST")
        .body("{}")
        .build();
    println!("{:?}", req);

    // 순서를 바꿔도 동작한다 — 각 메서드가 독립적으로 자기 상태만 전이하기 때문
    let req2 = HttpRequestBuilder::new()
        .method("GET")
        .url("https://example.com")
        .build();
    println!("{:?}", req2);
}
```

출력:

```text
HttpRequest { url: "https://api.example.com", method: "POST", body: Some("{}") }
HttpRequest { url: "https://example.com", method: "GET", body: None }
```

`new()`는 `HttpRequestBuilder<Missing, Missing>`을 돌려줍니다. `.url()`을 부르면 타입이 `<Set, Missing>`이 되고, `.method()`까지 부르면 `<Set, Set>`이 됩니다. 그리고 `build()`는 `impl HttpRequestBuilder<Set, Set>`에만 있으므로, **두 필수 필드가 모두 채워진 타입에서만 호출할 수 있습니다.**

### 11.6.1 컴파일러 에러 시연 — 필수 필드 누락

이제 `method()`를 빠뜨리고 `build()`를 불러 보겠습니다.

```rust
fn main() {
    // method()를 호출하지 않았다 → 타입은 HttpRequestBuilder<Set, Missing>
    let req = HttpRequestBuilder::new()
        .url("https://api.example.com")
        .build();   // 🚨 컴파일 에러!
    let _ = req;
}
```

`rustc`의 반응:

```text
error[E0599]: no method named `build` found for struct
              `HttpRequestBuilder<Set, Missing>` in the current scope
  --> src/main.rs:24:10
   |
 5 | pub struct HttpRequestBuilder<U, M> { ... }
   | ----------------------------------- method `build` not found for this struct
...
24 |         .build();   // 🚨 컴파일 에러!
   |          ^^^^^ method not found in `HttpRequestBuilder<Set, Missing>`
   |
   = note: the method was found for `HttpRequestBuilder<Set, Set>`
```

이 에러 메시지를 음미해 봅시다. `method not found in HttpRequestBuilder<Set, Missing>` — 타입 파라미터의 두 번째 자리가 `Missing`임이 그대로 드러납니다. 그리고 `the method was found for HttpRequestBuilder<Set, Set>`이라는 힌트가 "두 번째 필드도 `Set`이어야 한다", 즉 `method()`를 마저 호출하라고 알려 줍니다. **필수 필드를 빠뜨린 코드는 컴파일조차 되지 않습니다.** 11.4에서 파이썬 `oops = ServerConfig("localhost", tsl=True)`가 잘못된 객체를 조용히 만들어 내던 것과 차원이 다릅니다. 잘못된 객체는 애초에 **존재할 수 없습니다.**

> **Under the Hood: 타입 상태 빌더는 런타임에 흔적도 없이 사라진다 (ZST + PhantomData)**
>
> "타입 파라미터 두 개를 끌고 다니고 매 메서드마다 새 빌더를 만드니, 그만큼 메모리와 CPU를 더 쓰는 것 아닌가?" 정답은 **전혀 그렇지 않다**입니다. 핵심은 `Missing`과 `Set`이 **ZST(Zero-Sized Type, 크기 0 타입)**라는 점입니다.
>
> ```rust
> use std::mem::size_of;
> struct Missing;
> struct Set;
> // 필드가 없는 구조체의 크기는 0이다
> assert_eq!(size_of::<Missing>(), 0);
> assert_eq!(size_of::<Set>(), 0);
> assert_eq!(size_of::<PhantomData<(Set, Set)>>(), 0);
> ```
>
> `Missing`/`Set`은 데이터를 한 비트도 갖지 않습니다. 순수하게 **컴파일러에게 줄 정보**일 뿐입니다. 그런데 빌더 구조체가 이 타입을 *값으로* 들고 있지는 않습니다 — `PhantomData<(U, M)>`로 들고 있죠. `PhantomData<T>`는 "나는 논리적으로 `T`와 관련 있지만, `T`를 실제로 저장하지는 않는다"를 뜻하는 표준 라이브러리의 ZST 마커입니다.
>
> 그 결과, 메모리 레이아웃은 타입 파라미터와 무관하게 동일합니다.
>
> ```text
> HttpRequestBuilder<Missing, Missing> 의 메모리 = HttpRequestBuilder<Set, Set> 의 메모리
> ┌───────────────────────┬───────────────────────┬───────────────────────┐
> │ url:    Option<String>│ method: Option<String>│ body:   Option<String>│  ← 실제 데이터
> ├───────────────────────┴───────────────────────┴───────────────────────┤
> │ _marker: PhantomData<(U, M)>                                  (0 bytes) │  ← 사라짐
> └─────────────────────────────────────────────────────────────────────────┘
> ```
>
> 더 나아가, `.url().method().build()`로 빌더가 상태를 갈아타며 매번 새 구조체를 "만드는" 것처럼 보이지만, 모든 필드는 `move`(Chapter 4의 이동 의미론)로 옮겨지고 타입만 갈아끼워질 뿐 비트 수준에서는 같은 데이터입니다. 옵티마이저는 이 연쇄적 `move`를 꿰뚫어 보고 **빌더 전체를 제거(elide)**해 버립니다. `--release` 빌드에서 위 `main`은 결국 `HttpRequest { url, method, body }`를 스택에 한 번 직접 구성하는 코드로 펼쳐집니다. 즉 **타입 상태 빌더는 컴파일 타임 안전성을 100% 제공하면서 런타임 비용은 0**입니다. 이것이 무비용 추상화의 교과서적 사례입니다.

### 11.6.2 무비용임을 증명하기 — 1천만 객체 벤치마크와 동일 어셈블리

"런타임 비용 0"은 듣기 좋은 주장입니다. 그러나 이 책은 근거로 설득하기로 했으니, 직접 측정하고 어셈블리까지 확인해 봅시다. 타입 상태 빌더를 경유한 생성과 구조체 리터럴 직접 대입을 각각 1,000만 번 반복하고(`--release` 동급의 `rustc -O`, Apple Silicon, rustc 1.90.0), 누산값이 같은지로 결과의 동일성도 검증합니다.

```rust
// 발췌 — 두 경로가 같은 Config 1천만 개를 만든다 (전체는 본문 저장소 bench.rs)
fn via_builder() -> u64 {        // 타입 상태 빌더 경유
    let mut acc = 0u64;
    for i in 0..10_000_000u64 {
        let r = Builder::new()
            .url(String::new())
            .method((i & 0xFF) as u8)
            .timeout((i & 0xFFFF) as u32)
            .build();
        acc = acc.wrapping_add(r.method as u64 + r.timeout as u64 + r.retries as u64);
        std::hint::black_box(&r);   // 옵티마이저가 루프를 통째로 날리지 못하게 고정
    }
    acc
}
fn via_direct() -> u64 { /* 같은 필드를 HttpRequest { .. } 리터럴로 직접 대입 */ }
```

측정 결과는 다음과 같습니다.

```text
builder: 7.88 ms  (acc=328459709696)   0.79 ns/객체
direct : 7.79 ms  (acc=328459709696)   0.78 ns/객체
acc equal: true
```

두 경로의 시간이 측정 오차(약 1%) 안에서 같고, 누산값도 비트 단위로 일치합니다. 그런데 "비슷하다"는 약한 증거입니다. 더 강한 증거는 컴파일러가 두 함수에 대해 무엇을 토해내는가입니다. `#[no_mangle]`을 붙인 두 함수를 `rustc -O --emit asm`으로 뽑아 보면, 충격적인 결과가 나옵니다.

```text
$ rustc -O --crate-type=lib --emit asm asm.rs
$ grep -A1 make_via_builder asm.s
	.globl	_make_via_builder
.set _make_via_builder, _make_direct          # ← 같은 심볼로 별칭(alias) 처리!
```

LLVM이 **두 함수의 기계어가 한 바이트도 다르지 않음을 증명**하고는, 빌더 버전(`make_via_builder`)을 직접 대입 버전(`make_direct`)의 **별칭으로 합쳐 버렸습니다**(identical code folding). 즉 함수 본문이 따로 존재하지도 않습니다. 그렇다면 `make_direct`의 본문은 무엇일까요?

```text
_make_direct:                       # fn make_direct(p: u16, t: u32) -> Config
	and	w8, w0, #0xffff         # port (u16) 마스킹
	mov	w9, w1                  # timeout (u32)
	orr	x8, x9, x8, lsl #32     # 두 필드를 레지스터 한 쌍에 패킹
	orr	x0, x8, #0x3000000000000 # retries=3 상수를 비트 OR로 박아 넣음
	ret
```

빌더 구조체도, `PhantomData`도, `Option`의 래핑/언래핑도, `Missing→Set` 전이도 어셈블리에는 **흔적조차 없습니다.** 세 필드를 레지스터에 채워 반환하는 다섯 줄이 전부입니다. 타입 상태 빌더라는 정교한 컴파일 타임 안전 장치가 런타임에는 글자 그대로 **0바이트, 0사이클**로 증발한 것입니다. 11.1의 `__mro__` 순회, 그리고 11.3.3의 객체당 힙 할당과 비교해 보세요. 한쪽은 추상화가 런타임 세금이고, 다른 쪽은 추상화가 컴파일 타임에 청산됩니다.

### 11.6.3 타입 상태 빌더 vs `&dyn` 빌더 — vtable의 16바이트

타입 상태 빌더가 "정적 디스패치 + ZST"의 극단이라면, 그 반대편에는 **트레이트 객체(`&dyn`) 기반 빌더**가 있습니다. 설정 키를 런타임 문자열로 받아 동적으로 적용하는 플러그인형 빌더가 그 예입니다. 둘의 메모리·디스패치 비용을 정면으로 비교하면 무비용 추상화의 경계가 또렷해집니다.

```rust
// 동적 디스패치 빌더 — 키를 런타임에 해석
trait DynBuilder {
    fn set(&mut self, key: &str, val: &str);
}
struct ConcreteBuilder { url: Option<String>, method: Option<String> }
impl DynBuilder for ConcreteBuilder {
    fn set(&mut self, key: &str, val: &str) {
        match key {
            "url" => self.url = Some(val.into()),
            "method" => self.method = Some(val.into()),
            _ => {}
        }
    }
}

fn configure(b: &mut dyn DynBuilder) {   // &dyn → 동적 디스패치
    b.set("url", "https://example.com");
    b.set("method", "GET");
}
```

> **Under the Hood: ZST 마커(0바이트) vs fat pointer(16바이트)**
>
> 타입 상태 빌더의 `_marker: PhantomData<(U, M)>`는 0바이트입니다(11.6.1). 빌더의 상태 정보는 *값*이 아니라 *타입*에 있으므로, 메모리에는 실제 데이터 필드만 남습니다. 반면 `&dyn DynBuilder`는 **fat pointer(뚱뚱한 포인터)**입니다 — 데이터 포인터 8바이트 + vtable 포인터 8바이트, 합 16바이트. 직접 측정했습니다(rustc 1.90.0, 64비트).
>
> ```text
> size_of::<&ConcreteBuilder>()        = 8    ← 구체 타입 참조: thin pointer
> size_of::<&dyn DynBuilder>()         = 16   ← 트레이트 객체: fat pointer (data 8 + vtable 8)
> size_of::<&mut dyn DynBuilder>()     = 16
> size_of::<Box<dyn DynBuilder>>()     = 16
> size_of::<PhantomData<(Set, Set)>>() = 0    ← 타입 상태 마커: 무게 0
> ```
>
> 두 번째 8바이트가 가리키는 vtable은 컴파일러가 타입마다 한 번 만들어 두는 정적 테이블로, 대략 다음을 담습니다.
>
> ```text
> ConcreteBuilder의 vtable (정적, 읽기 전용):
> ┌────────────────────────────────────────────┐
> │ [0] drop_in_place(*mut ConcreteBuilder)     │  ← 소멸자 포인터
> │ [1] size  = 16  (ConcreteBuilder 크기)       │
> │ [2] align = 8                               │
> │ [3] set(&mut self, &str, &str)  함수 포인터  │  ← 트레이트 메서드
> └────────────────────────────────────────────┘
> ```
>
> `b.set(...)`를 호출할 때마다 런타임은 fat pointer의 두 번째 워드로 vtable을 찾고, 그 안의 슬롯 `[3]`에서 함수 주소를 읽어 **간접 점프(indirect call)**합니다. 이 한 단계의 메모리 적재 + 간접 분기가 동적 디스패치의 비용이며, 분기 예측이 빗나가면 더 비싸집니다. 게다가 함수 주소가 런타임에야 정해지므로 **인라이닝이 불가능**합니다 — 11.6.2에서 본 "다섯 줄로 증발"이 여기서는 일어나지 않습니다.
>
> 정리하면, 타입 상태 빌더는 *상태를 타입으로* 옮겨 메모리 0바이트·디스패치 0사이클·인라이닝 가능이라는 삼박자를 얻지만, 가능한 상태 조합마다 별도의 단일형화 코드를 찍어내 코드 크기가 늘 수 있습니다(11.3.3의 code bloat). `&dyn` 빌더는 fat pointer 16바이트와 간접 호출 비용을 내는 대신, 코드를 한 벌만 두고 런타임에 빌더 종류를 바꿔 끼울 수 있습니다(플러그인·설정 파일 기반 구성). 컴파일 타임에 필요한 빌더가 다 정해지고 성능이 중요하면 타입 상태를, 런타임에 빌더 종류를 갈아 끼워야 하면 `&dyn`을 택합니다. Chapter 10에서 본 정적 vs 동적 디스패치 트레이드오프가 생성 패턴에서도 그대로 반복됩니다.

---

## 11.7 그 외의 생성 패턴: `Default`, `From`/`TryFrom`, `new()`, 팩토리

빌더가 항상 정답은 아닙니다. 필드가 2~3개뿐이거나 모두 합리적 기본값이 있다면, 빌더는 과합니다. Rust는 생성을 위한 더 가벼운 관용구들을 제공합니다.

### 11.7.1 `Default` 트레이트 + 구조체 갱신 문법

파이썬의 `port=8080` 같은 기본값은 `Default` 트레이트로 표현합니다. 더 강력한 것은 **구조체 갱신 문법(Struct Update Syntax)** `..`입니다 — "나머지 필드는 다른 인스턴스에서 가져와라"라는 뜻입니다.

```rust
#[derive(Debug)]
pub struct Settings {
    retries: u8,
    verbose: bool,
    name: String,
}

impl Default for Settings {
    fn default() -> Self {
        Self { retries: 3, verbose: false, name: "anon".into() }
    }
}

fn main() {
    // verbose만 바꾸고 나머지는 기본값에서 가져온다 — 파이썬 kwargs의 "필요한 것만" 느낌
    let s = Settings {
        verbose: true,
        ..Default::default()
    };
    println!("{:?}", s);
    // Settings { retries: 3, verbose: true, name: "anon" }
}
```

필드 대부분이 기본값이라면 `..Default::default()` 한 줄이 빌더보다 훨씬 간결합니다. 필드가 모두 자명한 기본값(숫자 0, `false`, 빈 문자열)을 갖는다면 `#[derive(Default)]`로 `default()` 구현조차 자동 생성할 수 있습니다.

### 11.7.2 `From` / `TryFrom` — 변환을 생성자로

파이썬에서 `Celsius(100).to_fahrenheit()`나 `int("8080")` 같은 "다른 타입으로부터 생성"은, Rust에서 `From`(무조건 성공)과 `TryFrom`(실패 가능) 트레이트로 표현하는 것이 관용적입니다.

```rust
// From: 절대 실패하지 않는 변환 → into() 가 블랭킷 impl로 공짜로 따라온다 (11.3.3 참고)
#[derive(Debug)]
struct Celsius(f64);
#[derive(Debug)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

// TryFrom: 검증이 필요한 변환 → 실패하면 Err (생성과 검증을 한 곳에 묶는다)
#[derive(Debug)]
struct Port(u16);

impl TryFrom<i32> for Port {
    type Error = String;
    fn try_from(v: i32) -> Result<Self, Self::Error> {
        if (1..=65535).contains(&v) {
            Ok(Port(v as u16))
        } else {
            Err(format!("{v}는 유효한 포트가 아닙니다"))
        }
    }
}

fn main() {
    let f: Fahrenheit = Celsius(100.0).into();   // From → into() 자동 제공
    println!("{:?}", f);                          // Fahrenheit(212.0)

    println!("{:?}", Port::try_from(8080));        // Ok(Port(8080))
    println!("{:?}", Port::try_from(70000));       // Err("70000는 유효한 포트가 아닙니다")
}
```

`TryFrom`이 중요한 이유: 11.4의 파이썬 생성자는 `port`가 범위를 벗어나도 잘못된 객체를 만들어 버렸습니다. `TryFrom`은 검증을 통과하지 못한 값으로는 **`Port`를 만들 수 없게** 합니다. 일단 `Port`가 손에 있으면 그 안의 값이 유효함이 타입으로 보장됩니다 — Chapter 12에서 깊이 다룰 `Result`와 결합되어 "잘못된 데이터가 시스템 깊숙이 흘러드는 것"을 입구에서 막습니다.

이것은 파이썬 진영에서 Pydantic이 추구하는 "파싱하되 검증하라(parse, don't validate)" 철학과 정확히 같은 목표입니다. Pydantic은 런타임에 `__init__`에서 검증을 수행하고 실패 시 `ValidationError`를 던집니다. Rust의 `TryFrom` + 뉴타입(Newtype, Chapter 8)은 같은 검증을 컴파일 타임 타입 경계로 끌어올립니다. `fn connect(port: Port)`라는 시그니처를 가진 함수는, 호출자가 이미 `TryFrom`을 통과한 유효한 포트만 넘길 수 있으므로 내부에서 다시 범위 검사를 할 필요가 없습니다. "유효성 검사를 한 번만, 시스템 경계에서" 하는 설계가 자연스럽게 강제됩니다. 또한 `From`을 구현하면 `Into`가, `TryFrom`을 구현하면 `TryInto`가 11.3.3의 블랭킷 impl로 공짜로 따라온다는 점도 기억해 두세요 — `let p: Port = 8080i32.try_into()?;` 같은 호출이 추가 코드 없이 가능합니다.

### 11.7.3 `new()` 관례와 팩토리 함수

필수 인자가 적고 검증이 단순하면, 관례적인 연관 함수(Associated Function) `Self::new(...)`가 가장 직관적입니다(Chapter 7 참고). 그리고 같은 타입을 다양한 방식으로 만드는 **팩토리 함수**도 흔합니다.

```rust
pub struct Color { r: u8, g: u8, b: u8 }

impl Color {
    pub fn new(r: u8, g: u8, b: u8) -> Self {
        Self { r, g, b }
    }
    // 팩토리: 의미가 분명한 이름의 생성자들 (파이썬의 @classmethod 대체)
    pub fn black() -> Self { Self::new(0, 0, 0) }
    pub fn white() -> Self { Self::new(255, 255, 255) }
    pub fn from_hex(hex: u32) -> Self {
        Self::new((hex >> 16) as u8, (hex >> 8) as u8, hex as u8)
    }
}
```

`Color::black()`, `Color::from_hex(0xFF8800)`은 파이썬의 `@classmethod def from_hex(cls, ...)` 팩토리 메서드와 정확히 같은 역할입니다. 이름이 곧 문서가 됩니다.

**선택 가이드 — 어떤 생성 패턴을 쓸까:**

| 상황 | 권장 패턴 |
|---|---|
| 필수 인자 1~3개, 검증 단순 | `Self::new(...)` |
| 대부분 기본값, 일부만 변경 | `Default` + `..Default::default()` |
| 다른 타입에서 무조건 변환 | `From` (→ `into()` 공짜) |
| 다른 타입에서 검증 후 변환 | `TryFrom` (→ `Result`) |
| 옵션이 많고 일부만 설정 | 빌더 (소비형 / `derive_builder`) |
| 필수 필드 누락을 컴파일 타임에 막아야 함 | 타입 상태 빌더 |
| 다양한 의미의 생성 경로 | 팩토리 함수 (`black()`, `from_hex()`) |

---

## 11.8 실전 예제 (Hands-on): Mixin과 빌더를 합친 알림 시스템

지금까지 배운 것을 하나의 미니 프로젝트로 합쳐 봅시다. 요구사항은 이렇습니다.

- 이메일(`Email`)을 만들되, **수신자(`to`)와 본문(`body`)은 필수**, 제목·참조(cc)·우선순위는 선택. 필수 필드 누락은 **컴파일 타임에 차단**한다.
- 모든 알림 타입은 `summary()`(요약)와 `describe()`(상세 표현)라는 **횡단 관심사**를 공유한다. `describe()`는 어디에도 직접 구현하지 않고 자동으로 따라오게 한다.

이를 위해 타입 상태 빌더(11.6) + supertrait(11.3.2) + 블랭킷 impl(11.3.3)을 결합합니다.

```rust
use std::marker::PhantomData;

// ── Mixin 파트: 횡단 관심사 ──────────────────────────────
pub trait Summarize {
    fn summary(&self) -> String;
}

// Describable: Summarize를 전제로 하는 합성(supertrait) 믹스인
pub trait Describable: Summarize {
    fn describe(&self) -> String {
        format!("<{}>", self.summary())   // 상위 트레이트의 메서드를 재사용
    }
}

// Summarize를 구현한 "모든 T"에 Describable을 자동 부여 (블랭킷 impl)
impl<T: Summarize> Describable for T {}

// ── 생성 파트: 타입 상태 빌더 ─────────────────────────────
pub struct Missing;
pub struct Set;

#[derive(Debug)]
pub struct Email {
    to: String,
    body: String,
    subject: String,
    cc: Vec<String>,
    priority: u8,
}

impl Summarize for Email {
    fn summary(&self) -> String {
        format!("Email to {} (priority {})", self.to, self.priority)
    }
}

pub struct EmailBuilder<TO, BODY> {
    to: Option<String>,
    body: Option<String>,
    subject: String,
    cc: Vec<String>,
    priority: u8,
    _marker: PhantomData<(TO, BODY)>,
}

impl EmailBuilder<Missing, Missing> {
    pub fn new() -> Self {
        EmailBuilder {
            to: None,
            body: None,
            subject: "(no subject)".into(),
            cc: Vec::new(),
            priority: 3,
            _marker: PhantomData,
        }
    }
}

impl<TO, BODY> EmailBuilder<TO, BODY> {
    pub fn to(self, to: &str) -> EmailBuilder<Set, BODY> {
        EmailBuilder {
            to: Some(to.into()),
            body: self.body,
            subject: self.subject,
            cc: self.cc,
            priority: self.priority,
            _marker: PhantomData,
        }
    }
    pub fn body(self, body: &str) -> EmailBuilder<TO, Set> {
        EmailBuilder {
            to: self.to,
            body: Some(body.into()),
            subject: self.subject,
            cc: self.cc,
            priority: self.priority,
            _marker: PhantomData,
        }
    }
    // 선택 필드 — 상태 전이 없음
    pub fn subject(mut self, s: &str) -> Self { self.subject = s.into(); self }
    pub fn cc(mut self, addr: &str) -> Self { self.cc.push(addr.into()); self }
    pub fn priority(mut self, p: u8) -> Self { self.priority = p; self }
}

// build()는 to/body가 모두 Set일 때만 존재
impl EmailBuilder<Set, Set> {
    pub fn build(self) -> Email {
        Email {
            to: self.to.unwrap(),
            body: self.body.unwrap(),
            subject: self.subject,
            cc: self.cc,
            priority: self.priority,
        }
    }
}

fn main() {
    let mail = EmailBuilder::new()
        .to("ops@example.com")
        .subject("Deploy complete")
        .body("v2.3.1 is live")
        .cc("lead@example.com")
        .priority(1)
        .build();

    // describe()는 어디에도 직접 구현하지 않았지만 블랭킷 impl로 자동 생성됨
    println!("{}", mail.summary());
    println!("{}", mail.describe());
    println!("{:?}", mail);
}
```

출력:

```text
Email to ops@example.com (priority 1)
<Email to ops@example.com (priority 1)>
Email { to: "ops@example.com", body: "v2.3.1 is live", subject: "Deploy complete", cc: ["lead@example.com"], priority: 1 }
```

`mail.describe()`는 `Email`에 직접 구현한 적이 없습니다. `Email`이 `Summarize`를 구현했고, `impl<T: Summarize> Describable for T {}` 블랭킷 impl이 자동으로 `Describable`을 채워 줬기 때문에 따라온 것입니다. 새 알림 타입(`Sms`, `SlackMessage`)을 추가할 때도 `Summarize`만 구현하면 `describe()`가 공짜로 따라옵니다 — 파이썬 Mixin이 "상속받은 클래스에만" 적용되던 것과 달리, **조건을 만족하는 모든 타입에 한 번에** 능력이 부여됩니다.

그리고 `.body(...)`를 빼고 `.build()`를 호출하면, 11.6.1에서 본 것처럼 `error[E0599]: no method named 'build' found for EmailBuilder<Set, Missing>`로 컴파일이 막힙니다. 본문 없는 이메일은 타입 시스템상 만들 수 없습니다.

---

## 11.9 Best Practices & 안티패턴

**이렇게 하라:**

- **Mixin이 떠오르면 먼저 "기본 메서드가 딸린 트레이트"를 생각하라.** required method로 전제 조건을 시그니처에 명시하고, default method로 공통 행동을 제공합니다. 파이썬 Mixin의 묵시적 `self.x` 의존을 타입 제약으로 박제할 수 있습니다.
- **"X를 구현한 모든 타입에 Y를 주고 싶다"면 블랭킷 impl을 써라.** `impl<T: X> Y for T`. 표준 라이브러리의 `Into`/`ToString`이 정확히 이 방식입니다.
- **트레이트 의존 관계는 supertrait로 표현하라.** `trait B: A`는 "B를 쓰려면 A가 먼저 필요하다"를 컴파일러가 강제하게 합니다.
- **생성 패턴은 복잡도에 맞춰 점진적으로 올려라.** 필드 적음 → `new()`. 기본값 많음 → `Default` + `..`. 옵션 많음 → 빌더. 필수 누락을 막아야 함 → 타입 상태 빌더. 처음부터 타입 상태 빌더를 쓰는 건 대부분 과합니다.
- **검증이 필요한 생성은 `TryFrom`/`Result`로 표현하라.** "유효하지 않은 값으로는 타입을 만들 수 없다"가 가장 강력한 불변식(invariant)입니다.
- **`clippy`의 조언을 들어라.** `clippy::new_without_default`(공개 `new()`가 있는데 `Default`가 없을 때), `clippy::wrong_self_convention`(빌더 메서드의 self 관례) 등이 관용구를 안내합니다.

**이건 피하라 (안티패턴):**

- 트레이트 메서드 이름을 일부러 겹치게 설계하지 마라. `error[E0034]`로 막히고 호출부가 `<T as Trait>::method()`로 지저분해집니다.
- 행동을 데이터로 흉내 내려는 시도(예: 함수 포인터를 필드에 넣어 "상속" 흉내)를 하지 마라. 그건 트레이트가 할 일입니다.
- `Default`를 무의미한 기본값(예: 필수 ID를 0으로)에 구현하지 마라. "기본값으로 만들어진, 사실은 유효하지 않은 객체"는 파이썬의 묵시적 `**kwargs` 함정을 그대로 재현합니다.

---

## 11.10 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1 — 모든 필드를 `Option`으로 감싸기.**

파이썬에서 모든 게 기본값 `None`을 가질 수 있다 보니, Rust에서도 무심코 모든 필드를 `Option`으로 만듭니다.

```rust
// 🚨 안티패턴: 진짜로 필수인 필드까지 Option
struct ServerConfig {
    host: Option<String>,    // host는 사실 필수다!
    port: Option<u16>,
    tls: Option<bool>,
}
```

이렇게 하면 `config.host.unwrap()`이 코드 곳곳에 흩뿌려지고, "이 필드가 있을 수도 없을 수도 있다"는 거짓 정보가 타입에 새겨집니다. **`Option`은 "이 값이 논리적으로 없을 수 있다"를 의미할 때만 쓰세요.** 필수 필드는 그냥 `String`, `u16`으로 두고, 생성 시점의 "아직 안 채워짐"은 타입 상태 빌더의 `Missing`/`Set`으로 표현합니다. `Option`(런타임 값)과 타입 상태(컴파일 타임 정보)는 전혀 다른 도구입니다.

**함정 2 — 빌더 과(過)설계.**

필드 두세 개짜리 구조체에 타입 상태 빌더를 동원하는 것은 전형적인 과설계입니다.

```rust
// 🚨 과설계: Point에 타입 상태 빌더?
let p = PointBuilder::new().x(1.0).y(2.0).build();

// ✅ 이걸로 충분하다
let p = Point { x: 1.0, y: 2.0 };
// 또는
let p = Point::new(1.0, 2.0);
```

빌더는 "옵션이 많아 위치 인자로는 가독성이 무너질 때" 또는 "필수 필드 누락을 컴파일 타임에 막아야 할 때" 쓰는 도구입니다. 그 비용(보일러플레이트, 인지 부담)을 정당화할 만큼 복잡하지 않다면 구조체 리터럴이나 `new()`가 정답입니다. "파이썬 라이브러리가 빌더 비슷한 fluent API를 쓰니까 Rust도 그래야 한다"는 모방은 경계하세요.

**함정 3 — `&str`을 필드에 직접 넣으려다 수명 지옥.**

```rust
// 파이썬 습관: 문자열은 그냥 참조처럼 다룬다
struct Config<'a> {
    host: &'a str,   // 🚨 빌더가 수명에 오염되어 사용이 까다로워진다
}
```

설정 객체처럼 오래 살아야 하는 구조체는 보통 데이터를 **소유**해야 합니다. 빌더의 setter는 `&str`을 받되 내부에서 `.to_string()`/`.into()`로 소유 데이터(`String`)로 변환해 저장하세요(11.6 예제가 이 패턴입니다). Chapter 6에서 본 수명 규칙이 빌더 전체로 전파되어 사용이 불편해지는 것을 막아 줍니다.

**함정 4 — 빌더 setter에 검증 로직을 숨기고 패닉시키기.**

```rust
// 🚨 setter에서 panic — 호출부는 이게 실패할 수 있는 줄 모른다
pub fn port(mut self, p: u16) -> Self {
    if p == 0 { panic!("port cannot be 0"); }
    self.port = p;
    self
}
```

검증 실패는 패닉이 아니라 값으로 표현해야 합니다. 검증이 필요하면 `build()`가 `Result<Email, BuildError>`를 돌려주게 하거나(11.5.3의 `derive_builder` 방식), 애초에 `TryFrom`으로 입력을 정제해 빌더에 넣으세요. Chapter 12에서 다룰 "실패를 타입으로 표현하기"의 연장선입니다.

**함정 5 — 블랭킷 impl을 쓰다 고아 규칙(Orphan Rule)과 충돌하기.**

11.3.3의 블랭킷 impl은 강력하지만, 그 강력함을 두 가지 규칙이 단단히 제약합니다. 파이썬의 몽키패칭("외부 클래스에 메서드를 마음대로 주입")에 익숙한 손이 가장 자주 부딪히는 벽입니다.

첫째는 **고아 규칙(Orphan Rule)**입니다. *외부 크레이트의 트레이트*를 *외부 크레이트의 타입*에 구현하는 것은 금지됩니다 — 트레이트와 타입 중 적어도 하나는 내 크레이트 소속이어야 합니다. 예컨대 표준 라이브러리의 `Display`를 표준 라이브러리의 `Vec<T>`에 구현하려 들면:

```rust
use std::fmt;
// 🚨 둘 다 외부(std) 소속 → 고아 규칙 위반
impl<T> fmt::Display for Vec<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "my vec")
    }
}
```

```text
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
 --> src/main.rs:4:1
  |
4 | impl<T> fmt::Display for Vec<T> {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^------
  |                          |
  |                          `Vec` is not defined in the current crate
  |
  = note: define and implement a trait or new type instead
```

`error[E0117]`은 "트레이트도 타입도 다 남의 것"임을 지적합니다. 이 규칙이 없다면 두 크레이트가 같은 `(트레이트, 타입)` 쌍에 서로 다른 구현을 달았을 때, 어느 쪽이 이길지 정의할 수 없습니다(전역 일관성 붕괴). 해법은 에러 메시지가 친절히 알려 주듯 **뉴타입(Newtype, Chapter 8)으로 감싸기**입니다 — `struct MyVec<T>(Vec<T>);`를 만들면 `MyVec`이 내 타입이므로 `impl Display for MyVec`이 합법입니다.

둘째는 **중첩 금지(Overlap)**입니다. 블랭킷 impl이 이미 커버하는 타입에 다시 구체 impl을 달면, 같은 타입에 두 구현이 겹쳐 충돌합니다.

```rust
trait Summary { fn summary(&self) -> String; }

// 블랭킷: Debug를 가진 모든 T
impl<T: std::fmt::Debug> Summary for T {
    fn summary(&self) -> String { format!("{:?}", self) }
}

#[derive(Debug)]
struct Order { id: u64 }

// 🚨 Order는 Debug이므로 위 블랭킷이 이미 커버 중 → 충돌
impl Summary for Order {
    fn summary(&self) -> String { format!("Order #{}", self.id) }
}
```

```text
error[E0119]: conflicting implementations of trait `Summary` for type `Order`
  --> src/main.rs:14:1
   |
4  | impl<T: std::fmt::Debug> Summary for T {
   | -------------------------------------- first implementation here
...
14 | impl Summary for Order {
   | ^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `Order`
```

`error[E0119]`은 블랭킷 impl과 구체 impl이 `Order`에서 겹친다고 알려 줍니다. 파이썬이라면 나중에 정의한 메서드가 조용히 앞의 것을 덮어썼겠지만, Rust는 "두 후보 중 무엇을 쓸지 결정론적으로 정할 수 없다"며 거부합니다. 그래서 **블랭킷 impl은 한 트레이트당 하나만, 그리고 그것과 겹치는 구체 impl을 함께 둘 수 없다**는 점을 설계 단계에서 못 박아야 합니다. 일부 타입만 다르게 동작시키고 싶다면, 블랭킷 대신 각 타입에 개별 impl을 달거나(겹침 없음), 트레이트를 둘로 쪼개세요. (참고: 표준 라이브러리가 `From`/`Into`에서 블랭킷을 안전하게 쓸 수 있는 것도, 그 한 쌍에 대해 중첩이 발생하지 않도록 신중히 설계되어 있기 때문입니다 — 11.3.3.)

---

## 연습문제

Rustlings 스타일로 손을 움직여 풀어 보세요. 각 문항은 `fn main()`까지 직접 채워 `rustc -O x.rs`로 컴파일·실행하고, 정답 스케치의 **자기검증 신호**(컴파일러 에러코드 또는 측정값)로 답을 확인합니다. 머릿속으로만 읽지 말고 컴파일러에게 물어보세요 — 이 장의 핵심은 "타입 시스템이 답을 알려 준다"입니다.

**문제 1 — 소비형 빌더 (11.5.1).**
`DbConfig { url: String, pool_size: u32, timeout_ms: u32 }`를 위한 소비형 빌더를 작성하세요. 요구사항: (a) `DbConfig::builder("postgres://...")`로 시작하며 `url`은 필수, (b) `pool_size`(기본 10), `timeout_ms`(기본 5000)는 체이닝으로 선택 설정, (c) `.build()`로 완성. 그런 다음, 존재하지 않는 setter `.pool_zise(32)`(오타)를 호출해 보고 컴파일러가 무엇이라 하는지 확인하세요.

> **정답 스케치.** 본체 `DbConfig`와 별개로 `DbConfigBuilder` 구조체를 두고, `builder()`에서 기본값을 채워 반환합니다. 각 setter는 `mut self`를 받아 필드를 갱신하고 `self`를 돌려줘 체이닝을 잇습니다(`pub fn pool_size(mut self, n: u32) -> Self { self.pool_size = n; self }`). `build(self)`가 본체를 조립합니다.
> ```rust
> let cfg = DbConfig::builder("postgres://localhost/app").pool_size(32).build();
> assert_eq!(cfg.pool_size, 32);
> assert_eq!(cfg.timeout_ms, 5000); // 설정 안 한 필드는 기본값 유지
> ```
> 자기검증: 오타 `.pool_zise(32)`는 `error[E0599]: no method named 'pool_zise' found`로 막힙니다. 11.4의 파이썬 `tsl` 오타가 *침묵*하던 것과 정반대 — 빌더는 오타를 빌드 실패로 바꿉니다.

**문제 2 — 기본 메서드 트레이트 (11.3, Mixin 대체).**
재시도 정책을 표현하는 트레이트 `Retryable`을 정의하세요. (a) `max_attempts(&self) -> u32`는 **required method**(구현체가 반드시 채움), (b) `should_retry(&self, attempt: u32) -> bool`과 `backoff_ms(&self, attempt: u32) -> u64`는 **default method**로 제공합니다(`should_retry`는 `attempt < max_attempts()`, `backoff_ms`는 지수 백오프 `100 << attempt`). 그리고 `AggressiveClient`가 `backoff_ms`만 0으로 오버라이드하도록 하세요.

> **정답 스케치.** 핵심은 default method 안에서 required method를 호출하는 것입니다 — 이것이 파이썬 Mixin의 묵시적 `self.x` 의존을 *시그니처로* 박제하는 지점입니다.
> ```rust
> pub trait Retryable {
>     fn max_attempts(&self) -> u32;                       // required (전제)
>     fn should_retry(&self, attempt: u32) -> bool {       // default
>         attempt < self.max_attempts()
>     }
>     fn backoff_ms(&self, attempt: u32) -> u64 { 100u64 << attempt.min(6) }
> }
> ```
> 구현체는 `max_attempts`만 채우면 나머지 두 메서드를 공짜로 얻습니다. `AggressiveClient`는 `fn backoff_ms(&self, _: u32) -> u64 { 0 }`만 추가로 정의해 기본 구현을 덮습니다. 자기검증: `max_attempts` 구현을 빠뜨리면 `error[E0046]: not all trait items implemented, missing: 'max_attempts'`로 컴파일이 막힙니다. 측정 신호: `HttpClient{retries:3}`에 대해 `should_retry(2)==true`, `should_retry(3)==false`, `backoff_ms(3)==800`.

**문제 3 — 제네릭(타입 상태) 빌더 (11.6).**
`Job { name: String, command: String, schedule: Option<String> }`를 위한 타입 상태 빌더를 작성하세요. `name`과 `command`는 **둘 다 필수**라서, 둘 중 하나라도 설정하지 않으면 `.build()`가 **컴파일조차 되지 않아야** 합니다. `schedule`은 선택입니다. 완성 후, `.command(...)`만 호출하고 `.build()`를 불러 에러를 확인하세요.

> **정답 스케치.** ZST 마커 `Missing`/`Set` 두 개와, 타입 파라미터 두 개를 가진 `JobBuilder<N, C>`(필드는 `Option<String>` + `PhantomData<(N, C)>`)를 둡니다. `new()`는 `JobBuilder<Missing, Missing>`을 반환하고, `name()`은 `JobBuilder<Set, C>`로, `command()`는 `JobBuilder<N, Set>`로 *각자 자기 자리만* 전이시킵니다(그래서 호출 순서가 자유롭습니다). 결정적 한 줄은 `build`를 `impl JobBuilder<Set, Set>`에만 다는 것입니다.
> ```rust
> impl JobBuilder<Set, Set> {
>     pub fn build(self) -> Job {
>         Job { name: self.name.unwrap(), command: self.command.unwrap(), schedule: self.schedule }
>     }
> }
> ```
> 타입이 `<Set, Set>`임을 보장하므로 `unwrap()`은 절대 패닉하지 않습니다. 자기검증: `name()`을 빠뜨리면 타입이 `JobBuilder<Missing, Set>`이 되고, `.build()`는 `error[E0599]: no method named 'build' found for struct 'JobBuilder<Missing, Set>'` + `note: the method was found for - 'JobBuilder<Set, Set>'`로 막힙니다(11.6.1). 추가 검증: 11.6.2의 벤치마크처럼 이 빌더를 1천만 번 돌려 직접 대입과 시간을 비교하면 측정 오차 안에서 동일합니다 — 타입 파라미터는 0바이트이기 때문입니다.

---

지금까지 세 개 장(Chapter 8 컴포지션, Chapter 9 트레이트, Chapter 10 열거형)에 이어, 파이썬 객체지향의 마지막 두 기둥인 **Mixin(행동 재사용)**과 **복잡한 생성자(객체 조립)**가 Rust에서 어떻게 환생하는지 해부했습니다. 상속은 트레이트 합성과 블랭킷 impl로, `**kwargs`는 타입으로 검증되는 빌더로 치환됩니다. 그리고 그 모든 추상화가 단일형화와 ZST 덕분에 런타임 비용 0으로 사라진다는 것까지 확인했습니다. 이로써 Part 3 "Python 객체지향의 해체와 Rust 데이터 모델링"을 마칩니다.

여기까지 우리는 "올바른 데이터를 어떻게 모델링하고 만드는가"에 집중했습니다. 하지만 현실의 코드는 실패합니다. 파일이 없고, 네트워크가 끊기고, 입력이 망가집니다. 11.7에서 `TryFrom`이 `Result`를 돌려주는 걸 잠깐 봤고, 11.10에서 "검증 실패는 패닉이 아니라 값으로"라고 예고했습니다. 다음 Part 4의 Chapter 12 「`try-except`는 없다: `Option`과 `Result`」에서는, 파이썬 개발자가 본능적으로 `try-except`로 감싸던 모든 실패를 Rust가 어떻게 **타입 시스템 안의 값**으로 끌어들이는지, 그 패러다임 전환을 다룹니다.

---

## References

- https://doc.rust-lang.org/book/ch10-02-traits.html
- https://doc.rust-lang.org/reference/items/traits.html
- https://doc.rust-lang.org/std/default/trait.Default.html
- https://rust-lang.github.io/api-guidelines/type-safety.html
- https://doc.rust-lang.org/rust-by-example/trait.html
