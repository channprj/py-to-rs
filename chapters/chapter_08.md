# Chapter 8. 상속(Inheritance)의 종말과 컴포지션(Composition)

![Chapter 8. 상속(Inheritance)의 종말과 컴포지션(Composition)](../images/chapter-08-composition.jpg)

객체 지향 프로그래밍(OOP)을 10년 이상 해온 파이썬 개발자에게 **상속(Inheritance)**은 코드 재사용을 위한 가장 자연스러운 도구입니다. `BaseModel`, `AbstractUser`, `Mixin` 등 수많은 부모 클래스를 조합하여(다중 상속, MRO) 자식 클래스를 만들어 내는 것은 파이썬 프레임워크(특히 Django)에서 흔히 볼 수 있는 패턴입니다.

Django 모델 하나를 떠올려 봅시다. `class Post(TimestampedModel, SoftDeleteModel, models.Model)`처럼 세 개의 부모를 한 줄에 늘어놓는 순간, 여러분은 사실 거대한 빙산의 일각 위에 서 있는 것입니다. `models.Model`만 해도 메타클래스(`ModelBase`), 디스크립터, `__init_subclass__` 훅, 그리고 수백 개의 상속된 메서드를 끌고 옵니다. `Post` 인스턴스의 `save()`를 호출하면, 파이썬은 런타임에 MRO(Method Resolution Order)라는 선형화된 클래스 목록을 따라 어느 부모의 `save()`를 실행할지 탐색합니다. 잘 동작할 때는 마법 같지만, 부모 클래스 하나의 시그니처가 바뀌는 순간 — 혹은 두 부모가 같은 이름의 메서드를 가지고 있을 때 — 여러분은 "왜 내 코드가 깨졌지?"를 디버깅하며 밤을 새우게 됩니다.

유명한 격언이 있습니다.
> "나는 바나나를 원했는데, 바나나를 들고 있는 고릴라와 정글 전체를 얻게 되었다." — 조 암스트롱 (Erlang 창시자)

상속은 코드를 단단하게 결합(Coupling)시키며, 부모 클래스의 변경이 의도치 않게 모든 자식 클래스에 영향을 미치는 부작용 — **취약한 기반 클래스 문제(Fragile Base Class Problem)** — 를 낳습니다. 구현 상속(Implementation Inheritance)은 본질적으로 캡슐화를 깨뜨립니다. 자식이 부모의 *구현 세부사항*에 의존하기 때문입니다.

**Rust에는 상속이 없습니다.** `extends` 키워드도, `super()` 함수도, 클래스 계층도 존재하지 않습니다. 이것은 Rust가 OOP를 "못 만든" 것이 아니라, 50년 OOP 역사에서 축적된 교훈을 언어 차원에서 받아들인 의도적 설계입니다. Rust는 상속을 배제하고, 그 빈자리를 **컴포지션(Composition, 합성)**과 **트레이트(Trait)**로 채웠습니다. 이번 챕터에서는 파이썬의 상속 계층을 Rust의 컴포지션으로 옮기는 사고방식을 익히고, 그 과정에서 왜 이 전환이 "타이핑이 늘어나는 불편함"이 아니라 "버그가 줄어드는 안전함"인지 메모리와 디스패치 수준에서 증명하겠습니다.

---

## 8.1 The Python Way — 상속 트리, MRO, 그리고 그 함정들

파이썬의 상속은 단순합니다. 하지만 그 단순함은 **런타임 동적 탐색**이라는 비용 위에 세워져 있습니다. 시니어 개발자라면 이미 겪어 봤을 네 가지 고통을 차례로 짚어 봅시다.

### 깊은 상속 트리와 다중 상속

```python
from __future__ import annotations
import time

class TimestampedModel:
    def __init__(self) -> None:
        self.created_at: float = time.time()
        self.updated_at: float = self.created_at

    def touch(self) -> None:
        self.updated_at = time.time()

class SoftDeleteModel:
    def __init__(self) -> None:
        self.deleted_at: float | None = None

    def soft_delete(self) -> None:
        self.deleted_at = time.time()

# 다중 상속: 두 부모의 행동을 한꺼번에 물려받는다
class Post(TimestampedModel, SoftDeleteModel):
    def __init__(self, title: str) -> None:
        super().__init__()   # 🚨 여기서 무슨 일이 일어날까?
        self.title = title
```

`Post("hi")`를 만들면 `TimestampedModel.__init__`만 호출되고 `SoftDeleteModel.__init__`은 **호출되지 않습니다.** `super().__init__()`는 MRO상 *바로 다음* 클래스 하나만 호출하기 때문입니다. `self.deleted_at`은 존재하지 않고, 나중에 `post.soft_delete()`를 호출하는 순간 `AttributeError`가 터집니다. 이 버그는 **런타임에야** 발견됩니다.

### MRO와 C3 선형화

파이썬은 다중 상속의 모호함을 해결하기 위해 **C3 선형화(C3 Linearization)** 알고리즘으로 클래스들을 하나의 선형 목록(MRO)으로 정렬합니다.

```python
>>> Post.__mro__
(<class 'Post'>, <class 'TimestampedModel'>,
 <class 'SoftDeleteModel'>, <class 'object'>)
```

`post.touch()`를 호출하면 파이썬은 이 목록을 **왼쪽에서 오른쪽으로** 순회하며 `touch` 속성을 처음 발견하는 클래스에서 멈춥니다. 메서드를 호출할 때마다 이 탐색이 일어납니다(물론 캐싱은 있습니다). 문제는 이 순서가 사람의 직관과 어긋날 때입니다.

### 다이아몬드 문제 (Diamond Problem)

```python
class Base:
    def render(self) -> str:
        return "base"

class HtmlMixin(Base):
    def render(self) -> str:
        return "<html>" + super().render() + "</html>"

class JsonMixin(Base):
    def render(self) -> str:
        return '{"r": "' + super().render() + '"}'

class Widget(HtmlMixin, JsonMixin):  # 다이아몬드: 두 경로가 Base로 수렴
    pass

>>> Widget().render()
'<html>{"r": "base"}</html>'
```

`HtmlMixin.render` 안의 `super().render()`는 직관적으로는 `Base.render`를 가리킬 것 같지만, **실제로는 MRO상 다음 클래스인 `JsonMixin.render`**를 호출합니다. `super()`는 "부모"가 아니라 "MRO상 다음 노드"를 의미합니다. 믹스인을 작성한 사람과 조합한 사람이 다르면, 이 협력적 `super()` 체인은 추론하기 거의 불가능한 블랙박스가 됩니다.

### 취약한 기반 클래스 문제

```python
class Account:
    def deposit(self, amount: int) -> None:
        self._balance += amount

    def deposit_all(self, amounts: list[int]) -> None:
        for a in amounts:
            self.deposit(a)        # 내부적으로 deposit을 재사용

class AuditedAccount(Account):
    def deposit(self, amount: int) -> None:
        super().deposit(amount)
        print(f"감사 로그: +{amount}")
```

`AuditedAccount`는 `deposit`만 오버라이드했지만, 부모의 `deposit_all`이 내부에서 `self.deposit`을 호출하므로 감사 로그가 **자동으로** 따라옵니다. 편리해 보입니다. 그러나 어느 날 라이브러리 저자가 성능을 위해 `deposit_all`을 `self._balance += sum(amounts)`로 바꾸면? `AuditedAccount`의 감사 로그는 **소리 없이 사라집니다.** 자식이 부모의 *구현 세부사항*(deposit_all이 deposit을 호출한다는 사실)에 의존했기 때문입니다. 이것이 취약한 기반 클래스 문제의 본질입니다. 이 문제는 외부 라이브러리(SQLAlchemy, Django, DRF의 제네릭 뷰 등)를 상속할 때 특히 치명적입니다. 라이브러리 마이너 버전 업데이트가 여러분의 오버라이드를 조용히 무력화할 수 있고, 그것을 막을 컴파일러도, 타입 체커도 없습니다.

> 💡 **정리:** 파이썬의 상속은 (1) 다중 상속 초기화 누락, (2) 직관과 어긋나는 MRO, (3) 협력적 `super()`의 불투명함, (4) 구현 세부사항에 대한 암묵적 의존 — 이 네 가지 함정을 항상 동반합니다. 그리고 이 모든 것은 **컴파일 타임이 아니라 런타임에** 드러납니다.

> **Under the Hood: MRO 탐색의 런타임 비용**
>
> 상속 트리가 깊을수록, 메서드 호출은 더 많은 클래스를 거쳐야 합니다. CPython은 `type(obj).__mro__` 튜플을 순회하며 메서드를 찾고, 그 결과를 타입별 메서드 캐시(`tp_version_tag` 기반)에 저장합니다. 캐시가 있어도 "어트리뷰트 이름 → C 레벨 슬롯/딕셔너리 조회 → 바운드 메서드 객체 생성"이라는 동적 디스패치의 본질은 사라지지 않습니다. 5단계 깊이 상속에서 최상위에 정의된 메서드를 500만 번 호출해 측정해 봅시다.
>
> ```python
> class A:
>     def m(self) -> int: return 1
> class B(A): pass
> class C(B): pass
> class D(C): pass
> class E(D): pass   # E().m()은 MRO를 따라 A까지 거슬러 올라간다
>
> # 측정 결과(CPython 3.x, 단일 코어): 약 23 ns/call
> ```
>
> 호출당 약 **23 나노초**. 작아 보이지만, 핫 루프에서 수억 번 호출되면 누적됩니다. 더 중요한 것은 이 비용이 *언어 구조상 제거 불가능*하다는 점입니다. 같은 일을 Rust 트레이트의 정적 디스패치로 하면(8.5에서 측정) 컴파일러가 호출을 **인라인하고 심지어 결과 상수를 접어버려** 호출 비용이 사실상 0이 됩니다. MRO 순회라는 개념 자체가 컴파일 후에는 존재하지 않습니다.

---

## 8.2 The Rust Mental Model — 왜 상속을 버렸는가

Rust 설계자들은 위의 함정들을 알고 있었습니다. 그래서 구현 상속을 언어에서 통째로 제거하고, 두 가지 직교하는(orthogonal) 메커니즘으로 분리했습니다.

- **데이터의 재사용** → **컴포지션(Composition)**: 구조체가 다른 구조체를 *필드로 소유*한다. "has-a" 관계.
- **행동의 공유** → **트레이트(Trait)**: 인터페이스와 기본 메서드를 정의하고, 타입이 *명시적으로 구현*한다. "can-do" 관계.

파이썬의 상속은 이 둘을 하나로 뭉뚱그립니다. `class Post(TimestampedModel)`은 데이터(`created_at`)와 행동(`touch`)을 동시에, 그리고 암묵적으로 끌어옵니다. Rust는 이 둘을 강제로 떼어 놓습니다. 처음에는 번거롭지만, 결과적으로 "이 타입이 정확히 어떤 데이터를 가지고 어떤 행동을 할 수 있는가"가 코드에 **명시적으로** 드러납니다.

이것이 단순한 취향 문제가 아님을 강조하고 싶습니다. 사실 OOP 커뮤니티 자체가 수십 년에 걸쳐 같은 결론에 도달했습니다. GoF의 『디자인 패턴』(1994)은 첫 장에서 이미 **"클래스 상속보다 객체 컴포지션을 선호하라(Favor object composition over class inheritance)"**고 못 박았습니다. Effective Java, 그리고 거의 모든 현대 아키텍처 가이드가 같은 격언을 반복합니다. 즉, "상속 대신 컴포지션"은 Rust의 별난 발명이 아니라 *업계의 정설*입니다. 다른 점은, 파이썬·자바·C++은 상속을 *제공하되 자제를 권고*하는 반면, **Rust는 아예 구현 상속을 언어에서 제거하여 그 정설을 강제**한다는 것뿐입니다. 자제력에 의존하지 않고 구조로 보장하는 것 — 이것이 Rust의 일관된 철학입니다(소유권으로 메모리 안전을 강제했듯이).

여기서 한 가지 짚어둘 것이 있습니다. Rust에는 "인터페이스 상속(`extends interface`)"에 해당하는 **슈퍼트레이트(supertrait)** 문법(`trait Admin: User`)이 존재합니다. 이것은 "구현 상속"이 아니라 "이 트레이트를 구현하려면 저 트레이트도 구현해야 한다"는 *제약(bound)*일 뿐, 데이터나 코드를 물려받는 메커니즘이 아닙니다. 우리가 이번 챕터에서 "상속이 없다"고 말할 때 가리키는 것은 정확히 데이터·구현을 끌어오는 **구현 상속(implementation inheritance)**입니다.

| 파이썬 (상속 기반 OOP) | Rust (컴포지션 + 트레이트) | 차이의 핵심 |
| :--- | :--- | :--- |
| `class B(A)` (is-a) | `struct B { a: A }` (has-a) | 결합도 분리 |
| 내장 타입 서브클래싱 `class X(str)` | 뉴타입 `struct X(String)` | 타입 안전성 |
| 다중 상속 + MRO | 여러 트레이트 구현 + 컴포지션 | 모호함 제거 |
| `super().method()` | `self.field.method()` (명시적 위임) | 탐색 비용 0 |
| Mixin 클래스 | 트레이트 + 기본 메서드 (Ch11) | 충돌 시 컴파일 에러 |
| 런타임 메서드 탐색(MRO) | 컴파일 타임 정적 해소 | 무비용 추상화 |
| `isinstance()` 다형성 | 제네릭(static) / `dyn Trait`(dynamic) | Ch10에서 심화 |

이 표의 마지막 줄들 — 트레이트, 다형성 — 은 Chapter 9, 10, 11에서 본격적으로 다룹니다. 이번 챕터는 의도적으로 **"상속 → 컴포지션 리팩토링"** 관점에 집중합니다. 즉, "파이썬에서 상속으로 풀던 문제를 Rust에서 어떻게 데이터 소유와 위임으로 재구성하는가"가 핵심입니다.

---

## 8.3 Base Class를 버리고 컴포지션(Composition)으로

파이썬에서 흔히 작성하는 "기본 모델(Base Model)" 패턴을 다시 봅시다.

* **The Python Way (상속)**
  ```python
  class BaseDocument:
      def __init__(self, id: int, created_at: str):
          self.id = id
          self.created_at = created_at

      def get_metadata(self) -> str:
          return f"Doc {self.id} created at {self.created_at}"

  class Article(BaseDocument):
      def __init__(self, id: int, created_at: str, title: str, content: str):
          super().__init__(id, created_at)  # 부모 초기화 호출
          self.title = title
          self.content = content
  ```
  `Article`은 `BaseDocument`의 모든 데이터(`id`, `created_at`)와 행동(`get_metadata`)을 무조건적으로 물려받습니다.

* **The Rust Implementation (컴포지션)**
  Rust에서는 "is-a(~은 ~이다)" 관계인 상속 대신, **"has-a(~을 가지고 있다)"** 관계인 컴포지션을 사용합니다. 구조체 안에 다른 구조체를 필드로 포함시키는 방식입니다.

  ```rust
  // 공통 데이터를 담는 구조체
  pub struct BaseDocument {
      pub id: i32,
      pub created_at: String,
  }

  impl BaseDocument {
      pub fn get_metadata(&self) -> String {
          format!("Doc {} created at {}", self.id, self.created_at)
      }
  }

  // Article은 BaseDocument를 '상속'받지 않고, 필드로 '소유'합니다.
  pub struct Article {
      pub base: BaseDocument, // 합성(Composition)
      pub title: String,
      pub content: String,
  }

  impl Article {
      pub fn new(id: i32, created_at: String, title: String, content: String) -> Self {
          Self {
              base: BaseDocument { id, created_at },
              title,
              content,
          }
      }

      // 위임(Delegation): 부모의 메서드가 필요하면 명시적으로 호출합니다.
      pub fn get_metadata(&self) -> String {
          self.base.get_metadata()
      }
  }
  ```

이 방식은 코드를 더 많이 타이핑해야 하는 것처럼 보이지만(특히 위임 메서드 작성 시), 아키텍처 관점에서는 **부모와 자식 간의 결합도를 완벽하게 분리**해 냅니다. `BaseDocument`의 내부 구현이 바뀌어도, `Article`은 명시적으로 위임한 인터페이스(`get_metadata`)만 유지된다면 전혀 타격을 받지 않습니다. 8.1에서 본 취약한 기반 클래스 문제가 **구조적으로 불가능**해집니다. `Article`은 `BaseDocument`가 내부적으로 무엇을 어떻게 하는지 알 길이 없고, 오직 공개된 메서드만 호출하기 때문입니다.

### 위임은 명시적이다 — 그것이 장점이다

파이썬의 상속에서 `Article`은 `get_metadata`를 *공짜로* 얻습니다. Rust에서는 한 줄을 직접 써야 합니다. "보일러플레이트 아닌가?"라는 의문이 들 수 있습니다. 하지만 이 한 줄이 바로 "Article이 이 기능을 외부에 노출하기로 *결정했다*"는 의도의 표현입니다. 노출하고 싶지 않은 부모 메서드는 위임하지 않으면 됩니다 — 캡슐화가 기본값입니다. 파이썬에서는 부모의 모든 public 메서드가 자동으로 새어 나오지만, Rust에서는 새어 나올 통로 자체가 없습니다.

위임 메서드가 정말로 수십 개에 달해 부담스럽다면, 생태계에는 이를 자동화하는 크레이트들이 있습니다. `delegate`, `ambassador` 같은 크레이트는 매크로로 "이 필드에게 이 메서드들을 위임하라"를 선언적으로 표현하게 해 줍니다(매크로의 작동 원리는 Chapter 19에서 다룹니다). 다만 실무 권고는 보수적입니다 — 위임 대상이 서너 개를 넘지 않는 한, 명시적으로 손으로 쓴 위임 메서드가 가독성과 IDE 탐색 측면에서 더 낫습니다. "공짜 상속"의 편리함에 길든 손이 가장 먼저 찾는 도구가 자동 위임 매크로인데, 그것이 오히려 8.1의 암묵적 결합을 되살릴 수 있음을 기억하세요.

또 하나 흔한 컴포지션 형태는 **선택적 소유**입니다. 파이썬에서 "어떤 자식은 캐시를 갖고 어떤 자식은 안 갖는" 상황을 조건부 상속이나 믹스인으로 풀었다면, Rust에서는 그냥 `cache: Option<Cache>` 필드 하나로 표현합니다. 상속 계층을 늘릴 필요 없이, 필드의 유무·타입 조합만으로 무한한 변형을 *데이터로* 표현할 수 있다는 점이 컴포지션의 본질적 자유도입니다.

> **Under the Hood: 평탄한 메모리 레이아웃 (Flat Memory Layout)**
>
> 파이썬에서 `Article` 인스턴스는 `__dict__`라는 해시맵을 들고 다니며, `id`·`created_at`·`title`·`content` 각각은 별도의 PyObject 포인터로 힙 여기저기에 흩어집니다. 속성 하나를 읽으려면 해시맵 조회 → 포인터 역참조의 연쇄가 일어납니다.
>
> Rust의 `Article`은 컴파일 타임에 **하나의 평탄한 메모리 블록**으로 펼쳐집니다. `base: BaseDocument`가 별도의 객체로 힙에 따로 존재하는 것이 아니라, `Article`의 메모리 안에 `id`, `created_at`, `title`, `content`가 **인라인으로** 박힙니다. 다음으로 확인해 봅시다.
>
> ```rust
> use std::mem::size_of;
>
> struct Article {
>     base_id: i32,        // 4 bytes
>     base_created: u64,   // 8 bytes (예시용 타임스탬프)
>     title_len: usize,    // 8 bytes
> }
>
> fn main() {
>     // 출력: 24 (정렬 패딩 포함, 단일 연속 블록)
>     println!("{}", size_of::<Article>());
> }
> ```
>
> 컴포지션은 "포인터로 연결된 객체 그래프"가 아니라 "값이 인라인된 단일 구조체"입니다. CPU 캐시 한 줄(보통 64바이트)에 모든 필드가 함께 올라오므로, 파이썬의 흩어진 포인터 추적 대비 캐시 미스가 극적으로 줄어듭니다. 컴포지션은 추상화 비용이 0일 뿐 아니라, 데이터 지역성(locality) 면에서 오히려 이득입니다.

---

## 8.4 내장 타입 상속의 대체: 뉴타입 패턴 (Newtype Pattern)

파이썬 개발자들은 타입 힌팅이나 검증 로직을 추가하기 위해 흔히 내장 타입을 상속받습니다.

* **The Python Way (내장 타입 상속)**
  ```python
  class EmailAddress(str):
      def __new__(cls, value: str) -> "EmailAddress":
          if "@" not in value:
              raise ValueError("Invalid email format")
          return super().__new__(cls, value)

  my_email = EmailAddress("user@example.com")
  print(my_email.upper())  # str의 모든 메서드를 그대로 사용 가능
  ```
  이 패턴은 편리하지만 위험합니다. `EmailAddress`는 `str`의 **모든** 메서드를 물려받으므로, `my_email + "?reset"` 같은 연산이 가능하고 그 결과는 검증을 거치지 않은 평범한 `str`이 됩니다. 검증 불변식(invariant)이 객체의 수명 내내 보장되지 않습니다.

* **The Rust Implementation (Newtype Pattern)**
  Rust에서는 `String`이나 `i32` 같은 표준 라이브러리 타입을 직접 확장(상속)할 수 없습니다. 대신 **필드가 하나뿐인 튜플 구조체(Tuple Struct)**를 사용하여 기존 타입을 감싸는 '뉴타입 패턴'을 씁니다.

  ```rust
  // String을 감싸는 새로운 독립적인 타입 'EmailAddress' 선언
  #[derive(Debug, Clone, PartialEq)]
  pub struct EmailAddress(String);

  impl EmailAddress {
      // 생성 시점에 검증 로직 통과를 강제함 (한 번 만들어지면 항상 유효)
      pub fn new(value: &str) -> Result<Self, String> {
          if !value.contains('@') {
              return Err(format!("'{value}'는 올바른 이메일이 아닙니다"));
          }
          Ok(EmailAddress(value.to_string()))
      }

      // 내부 String 값에 접근하는 Getter
      pub fn as_str(&self) -> &str {
          &self.0 // 튜플의 첫 번째 필드는 .0 으로 접근
      }
  }

  fn main() {
      let email = EmailAddress::new("user@example.com").unwrap();

      // 파이썬처럼 기존 str의 메서드를 멋대로 호출할 수 없음.
      // email.to_uppercase(); // 🚨 컴파일 에러! EmailAddress에는 이런 메서드가 없음.

      println!("Email is: {}", email.as_str());
  }
  ```

  뉴타입은 **딱 우리가 허용한 행동만** 노출합니다. `str`의 200여 개 메서드가 자동으로 새어 나오지 않습니다. 검증을 거치지 않으면 `EmailAddress`를 만들 길이 없으므로, 타입의 존재 자체가 "이 값은 유효한 이메일이다"라는 증명서가 됩니다. 이것을 **"불법 상태를 표현 불가능하게 만들기(Make illegal states unrepresentable)"**라고 부릅니다.

### 타입 안전성: UserId와 OrderId를 섞을 수 없다

뉴타입 패턴은 **도메인 주도 설계(DDD, Domain Driven Design)**의 핵심입니다. 파이썬에서는 `user_id`와 `order_id`가 모두 `int`라면, 실수로 주문 ID 자리에 유저 ID를 넣어도 코드가 정상 실행됩니다(논리적 버그).

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(u64);

fn fetch_user(id: UserId) -> String {
    format!("user #{}", id.0)
}

fn main() {
    let order = OrderId(1001);
    // 🚨 컴파일 에러! OrderId를 UserId 자리에 넣음
    let who = fetch_user(order);
    println!("{who}");
}
```

이 코드는 컴파일조차 되지 않습니다. `rustc`가 내뱉는 메시지를 그대로 봅시다.

```text
error[E0308]: mismatched types
  --> src/main.rs:13:26
   |
13 |     let who = fetch_user(order);
   |               ---------- ^^^^^ expected `UserId`, found `OrderId`
   |               |
   |               arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:6:4
   |
 6 | fn fetch_user(id: UserId) -> String {
   |    ^^^^^^^^^^ ----------
```

`error[E0308]: mismatched types`. 컴파일러는 정확히 "여기서는 `UserId`를 기대했는데 `OrderId`가 들어왔다"고 알려주고, 문제의 함수가 어디 정의됐는지까지 짚어 줍니다. 파이썬이라면 `fetch_user(order)`가 멀쩡히 실행되어 엉뚱한 유저 데이터를 반환하고, 그 버그는 프로덕션의 어느 새벽에 고객 클레임으로 발견됐을 것입니다. Rust에서는 **이런 종류의 버그가 컴파일 게이트를 통과하지 못합니다.**

> **Under the Hood: 뉴타입의 런타임 비용은 정확히 0**
>
> "타입을 하나 더 만들면 래퍼 객체 비용이 들지 않나?"라는 의심이 들 수 있습니다. 직접 측정해 봅시다.
>
> ```rust
> use std::mem::size_of;
>
> struct UserId(u64);
>
> fn main() {
>     println!("u64    = {}", size_of::<u64>());     // 8
>     println!("UserId = {}", size_of::<UserId>());  // 8
> }
> ```
>
> `UserId`는 메모리상에서 정확히 `u64`와 똑같은 8바이트입니다. 별도의 태그도, 박싱도, 포인터 간접 참조도 없습니다. 컴파일러는 단일형화(Monomorphization) 과정에서 뉴타입 래퍼를 **완전히 지워버립니다.** 생성된 기계어 수준에서 `fetch_user(UserId(42))`는 그냥 `fetch_user(42)`와 동일한 코드입니다. 타입 안전성은 100% 컴파일 타임의 약속이며, 런타임에는 그 흔적조차 남지 않습니다. 이것이 Rust가 말하는 **무비용 추상화(Zero-cost Abstraction)**의 교과서적 예시입니다. 파이썬에서 같은 안전성을 흉내내려면 `class UserId(int)` 서브클래싱이 필요한데, 그러면 모든 `UserId`가 별도의 PyObject(최소 28바이트 + 박싱)가 되어 *런타임 비용을 지불하고도* `+`나 비교 연산에서 평범한 `int`로 도로 떨어져 안전성이 새어 나갑니다. Rust는 비용 0으로 더 강한 보장을 줍니다.

> 💡 **Under the Hood: `#[repr(transparent)]`와 FFI ABI**
>
> `size_of`가 같다는 것과 *ABI(호출 규약상 표현)가 같다*는 것은 다른 이야기입니다. 그냥 `struct UserId(u64)`는 크기가 8바이트인 건 맞지만, 컴파일러가 그 표현을 내부 `u64`와 동일하게 유지한다는 **보장은 없습니다**(단일 필드라 실질적으로는 같지만, 언어 차원의 약속은 아닙니다). C 함수에 뉴타입을 그대로 넘기는 FFI 경계에서는 이 보장이 결정적입니다(C FFI는 Chapter 21). 바로 이때 `#[repr(transparent)]`를 씁니다.
>
> ```rust
> use std::mem::{align_of, size_of};
>
> // ABI 수준에서 내부 u64와 '완전히 동일한' 표현임을 언어가 보장한다
> #[repr(transparent)]
> #[derive(Debug, Clone, Copy)]
> pub struct Handle(u64);
>
> // C ABI 함수가 u64를 받는다고 가정
> extern "C" fn raw_consume(h: u64) -> u64 { h.wrapping_mul(2) }
>
> fn main() {
>     println!("{} {}", size_of::<u64>(), align_of::<u64>());        // 8 8
>     println!("{} {}", size_of::<Handle>(), align_of::<Handle>());  // 8 8
>
>     let h = Handle(21);
>     println!("{}", raw_consume(h.0)); // 42
>
>     // u64는 니치(niche)가 없으므로(모든 비트패턴이 유효) Option 태그가 별도 공간을 먹는다
>     println!("{}", size_of::<Option<Handle>>()); // 16, 8이 아님
> }
> ```
>
> `#[repr(transparent)]`는 "이 뉴타입은 레지스터·메모리·함수 인자 전달 방식까지 내부 타입과 한 글자도 다르지 않다"를 컴파일러에게 못 박습니다. 따라서 `extern "C"` 함수 시그니처에서 `u64` 자리에 `Handle`을 그대로 써도 ABI가 깨지지 않습니다 — 의미상 타입 안전성은 Rust 쪽에서 챙기면서, C 쪽에서는 그냥 `uint64_t`로 보입니다. 단일 필드(또는 영-크기 필드 + 단일 비-영 필드)에만 붙일 수 있다는 제약이 있습니다.
>
> 한 가지 주의: 위 출력에서 `Option<Handle>`이 8이 아니라 **16**인 이유는, `u64`에는 "사용되지 않는 비트패턴(니치)"이 없어서 `None`을 표현할 여분의 값이 없기 때문입니다. `Option<&T>`나 `Option<Box<T>>`가 8바이트로 압축되는 니치 최적화는 0이라는 무효 포인터값을 `None`에 빌려 쓸 수 있을 때만 성립합니다. `repr(transparent)`는 이 니치 특성까지 그대로 전달하므로, *내부 타입이 니치를 가지면 뉴타입도 그 최적화를 물려받습니다*. 표현을 정직하게 위임한다는 것의 의미가 여기까지 닿습니다.

### 단위(Unit)를 타입으로: 화성 기후 궤도선의 교훈

1999년 NASA의 화성 기후 궤도선은 한 팀이 야드파운드법(파운드힘·초)으로, 다른 팀이 미터법(뉴턴·초)으로 추력 데이터를 주고받다가 단위 변환 누락으로 추락했습니다. 두 값 모두 평범한 `float`였기에 타입 시스템이 막아줄 수 없었습니다. 뉴타입은 바로 이 부류의 버그를 박멸합니다.

```rust
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Celsius(f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Fahrenheit(f64);

impl Celsius {
    pub fn to_fahrenheit(self) -> Fahrenheit {
        Fahrenheit(self.0 * 9.0 / 5.0 + 32.0)
    }
    pub fn value(self) -> f64 {
        self.0
    }
}

fn set_thermostat(temp: Celsius) {
    println!("set to {}°C", temp.value());
}

fn main() {
    let body = Celsius(36.5);
    println!("{:?}", body.to_fahrenheit()); // Fahrenheit(97.7)

    set_thermostat(body);
    // set_thermostat(Fahrenheit(98.0)); // 🚨 E0308: expected `Celsius`, found `Fahrenheit`
}
```

`Celsius`와 `Fahrenheit`는 둘 다 메모리상 `f64`(8바이트) 그 자체이지만, 타입 시스템은 둘을 절대 섞지 못하게 합니다. 변환은 오직 `to_fahrenheit()` 같은 *명시적* 통로로만 일어납니다. 측정 단위, 통화(`Usd`/`Krw`), 좌표계(`ScreenPixels`/`WorldUnits`), 인코딩 상태(`RawHtml`/`SanitizedHtml`) 등 "값은 같은 원시 타입이지만 의미가 다른" 모든 도메인에서 이 패턴은 강력합니다.

### 흔한 함정: f64 뉴타입에 `#[derive(Ord)]` (E0277)

`Celsius` 값들을 온도순으로 정렬하고 싶어진 순간, 파이썬 습관이 있는 손은 반사적으로 `#[derive(Ord)]`를 추가하려 합니다. 파이썬에서는 `sorted([Celsius(20), Celsius(5)])`가 `__lt__`만 있으면 그냥 동작했으니까요. Rust에서는 다음과 같이 막힙니다.

```rust
// 🚨 컴파일 에러! f64에는 Eq도 Ord도 구현되어 있지 않다
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd, Eq, Ord)]
pub struct Celsius(f64);
```

```text
error[E0277]: the trait bound `f64: Eq` is not satisfied
 --> ord_trap.rs:3:20
  |
2 | #[derive(Debug, Clone, Copy, PartialEq, PartialOrd, Eq, Ord)]
  |                                                     -- in this derive macro expansion
3 | pub struct Celsius(f64);
  |                    ^^^ the trait `Eq` is not implemented for `f64`
...
error[E0277]: the trait bound `f64: Ord` is not satisfied
 --> ord_trap.rs:3:20
  |
2 | #[derive(Debug, Clone, Copy, PartialEq, PartialOrd, Eq, Ord)]
  |                                                         --- in this derive macro expansion
3 | pub struct Celsius(f64);
  |                    ^^^ the trait `Ord` is not implemented for `f64`
```

`derive(Ord)`는 모든 필드가 `Ord`여야 하는데 **`f64`는 `Ord`도 `Eq`도 구현하지 않습니다.** 왜일까요? `f64`에는 `NaN`이라는 악동이 있어 `NaN != NaN`이고 `NaN < x`도 `NaN > x`도 모두 `false`입니다. 즉 **전순서(total order)와 동등성(반사성)이 깨집니다.** Rust는 이 수학적 사실을 타입으로 인코딩했습니다 — `f64`는 *부분 순서(partial order)*인 `PartialOrd`/`PartialEq`만 구현하고, *전순서*를 요구하는 `Ord`/`Eq`는 의도적으로 구현하지 않습니다. 파이썬은 이 함정을 숨긴 채 `sorted([float('nan'), 1.0, 2.0])`이 입력 순서에 따라 *조용히 다른 결과*를 내도록 방치하지만, Rust는 컴파일 게이트에서 멈춰 세웁니다.

그래서 `temps.sort()`도 같은 이유로 거부됩니다 — `sort()`가 `Ord`를 요구하기 때문입니다.

```text
error[E0277]: the trait bound `Celsius: Ord` is not satisfied
 --> sort_trap.rs:6:11
  |
6 |     temps.sort();
  |           ^^^^ the trait `Ord` is not implemented for `Celsius`
  |
note: required by a bound in `slice::<impl [T]>::sort`
```

해법은 "어떤 순서를 원하는지 명시"하는 것입니다. `Eq`/`Ord`는 빼고 `PartialOrd`만 둔 채, 정렬은 비교 함수를 직접 건넵니다.

```rust
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Celsius(f64);

fn main() {
    let mut temps = vec![Celsius(20.0), Celsius(5.0), Celsius(36.5), Celsius(f64::NAN)];

    // 해법 1: 부분 순서(partial_cmp)로 정렬하되 NaN 처리를 직접 결정한다
    temps.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap_or(std::cmp::Ordering::Equal));
    println!("{temps:?}"); // [Celsius(5.0), Celsius(20.0), Celsius(36.5), Celsius(NaN)]

    // 해법 2: f64::total_cmp 로 NaN까지 포함한 전순서를 '명시적으로' 부여한다
    let mut t2 = vec![Celsius(20.0), Celsius(5.0), Celsius(36.5), Celsius(f64::NAN)];
    t2.sort_by(|a, b| a.0.total_cmp(&b.0));
    println!("{t2:?}"); // [Celsius(5.0), Celsius(20.0), Celsius(36.5), Celsius(NaN)]
}
```

핵심 교훈: **정수 뉴타입(`UserId(u64)`)에는 `#[derive(Ord)]`가 잘 동작하지만, 부동소수점을 보유한 뉴타입에는 `Ord`가 따라오지 않습니다.** 정렬·`BTreeMap` 키·`max()` 등 전순서가 필요한 곳에서는 `sort_by(... total_cmp ...)`로 *NaN을 어떻게 다룰지 직접 선택*해야 합니다. 부동소수점 정렬을 자주 한다면 `ordered-float` 크레이트의 `OrderedFloat`/`NotNan` 타입이 바로 이 뉴타입을 표준화해 `Ord`까지 제공합니다.

### Deref와 Deref Coercion: 위임을 자동화하기 (주의 필요)

뉴타입의 단점은 "내부 타입의 메서드를 쓰려면 매번 `.0`이나 `.as_str()`을 거쳐야 한다"는 점입니다. 정말로 내부 타입의 인터페이스 대부분을 그대로 노출하고 싶다면, `Deref` 트레이트를 구현하여 **Deref 강제 변환(Deref Coercion)**을 활성화할 수 있습니다.

```rust
use std::ops::Deref;

#[derive(Debug, Clone, PartialEq)]
pub struct EmailAddress(String);

impl EmailAddress {
    pub fn new(value: &str) -> Result<Self, String> {
        if !value.contains('@') {
            return Err(format!("'{value}'는 올바른 이메일이 아닙니다"));
        }
        Ok(EmailAddress(value.to_string()))
    }
}

// EmailAddress를 &str처럼 다룰 수 있게 해준다
impl Deref for EmailAddress {
    type Target = str;
    fn deref(&self) -> &str {
        &self.0
    }
}

fn main() {
    let email = EmailAddress::new("user@example.com").unwrap();

    // Deref coercion 덕분에 &EmailAddress가 자동으로 &str로 변환되어
    // str의 메서드를 직접 호출할 수 있다.
    println!("upper = {}", email.to_uppercase());  // "USER@EXAMPLE.COM"
    println!("len   = {}", email.len());           // 16
    println!("domain has dot? {}", email.contains('.'));
}
```

이것이 표준 라이브러리가 `String`(→ `Deref<Target = str>`)이나 `Box<T>`(→ `Deref<Target = T>`)에서 쓰는 바로 그 메커니즘입니다. `&String`을 `&str` 파라미터에 넘길 수 있는 이유가 정확히 이것입니다.

🚨 **하지만 `Deref`를 "상속처럼" 남용하면 안 됩니다.** `Deref`는 *스마트 포인터*를 위한 것이지, 일반적인 위임 도구가 아닙니다. 다음은 흔한 안티패턴입니다.

```rust
use std::ops::Deref;

struct Meters(f64);
impl Deref for Meters {
    type Target = f64;
    fn deref(&self) -> &f64 { &self.0 }
}

fn main() {
    let d = Meters(100.0);
    // 의도치 않게 f64의 모든 API가 새어 나온다
    println!("{}", d.sqrt());    // 10 — Meters에 sqrt가 의미가 있나?
    println!("{}", d.is_nan());  // false
    // 캡슐화의 목적(특정 메서드만 노출)이 완전히 무너졌다
}
```

`Meters`에 `sqrt()`가 호출 가능한 것은 도메인 의미상 난센스입니다. `Deref`로 모든 것을 노출하면 뉴타입을 만든 이유(불변식 보호, 의미 있는 API만 노출)가 사라집니다.

> 🚨 **사실 확인(정정):** "이 `Deref` 남용을 Clippy가 경고한다"는 통념이 있지만, **거짓입니다.** 위 `Meters` 코드를 `cargo clippy`로 검사하면(기본 린트는 물론 `-W clippy::pedantic`까지 켜도) **경고가 단 하나도 나오지 않습니다**(rustc/clippy 1.90.0에서 직접 확인). 이름이 비슷한 `clippy::should_implement_trait`는 *전혀 다른 상황*을 잡습니다 — "표준 트레이트 메서드와 같은 이름의 *고유(inherent) 메서드*"를 만들었을 때입니다. 예컨대 `impl Foo { pub fn add(self, other: Foo) -> Foo { ... } }`처럼 `std::ops::Add::add`와 충돌하는 이름의 공개 메서드를 두면, clippy가 `method 'add' can be confused for the standard trait method 'std::ops::Add::add'`라고 경고합니다. 즉 `should_implement_trait`는 `Deref` 오용 탐지기가 아닙니다. **`Deref`를 도메인 뉴타입에 거는 것은 Clippy가 막아주지 않는, 오롯이 설계 규율의 영역입니다.** 도구가 잡아주리라 기대하지 말고, 다음 경험칙을 손으로 지키세요.

경험칙은 이렇습니다.

- 뉴타입이 *진짜로* "내부 타입의 얇은 래퍼"이고 거의 모든 인터페이스를 그대로 제공해야 한다면(예: 스마트 포인터) → `Deref`.
- 뉴타입이 *제한된 도메인 개념*(이메일, 미터, 유저 ID)이라면 → `Deref` 대신 필요한 메서드만 명시적으로 `as_str()`, `value()` 등으로 노출하거나, `AsRef<str>`만 구현.

---

## 8.5 공유 행동은 상속이 아니라 트레이트로

컴포지션은 "데이터"의 재사용을 해결합니다. 그러면 "여러 타입이 같은 *행동*을 공유"해야 할 때는 어떻게 할까요? 파이썬이라면 공통 메서드를 부모 클래스에 두고 상속받겠지만, Rust는 **트레이트(Trait)**를 씁니다. 트레이트는 인터페이스 정의이자, **기본 메서드(Default Method)**를 제공하는 행동의 모음입니다.

> 트레이트의 전체 문법과 매직 메서드 매핑(`Display`, `Add` 등)은 Chapter 9에서, 트레이트 객체를 통한 동적 다형성은 Chapter 10에서, 믹스인 스타일 기본 메서드는 Chapter 11에서 깊이 다룹니다. 여기서는 **"상속으로 풀던 행동 공유를 트레이트로 재구성하는"** 관점만 봅니다.

8.1의 `Account` 예제(취약한 기반 클래스 문제)를 트레이트로 재설계해 봅시다. 핵심은 "공유하고 싶은 행동의 *계약(contract)*"과 "각 타입의 *데이터*"를 분리하는 것입니다.

```rust
// 공유할 행동의 계약. 구현체는 balance에 접근하는 방법만 제공하면 된다.
pub trait Balance {
    fn balance(&self) -> i64;
    fn balance_mut(&mut self) -> &mut i64;

    // 기본 메서드: 계약을 구현하면 공짜로 따라온다
    fn deposit(&mut self, amount: i64) {
        *self.balance_mut() += amount;
    }
    fn deposit_all(&mut self, amounts: &[i64]) {
        for &a in amounts {
            self.deposit(a);
        }
    }
}

pub struct Wallet {
    funds: i64,
}

impl Balance for Wallet {
    fn balance(&self) -> i64 {
        self.funds
    }
    fn balance_mut(&mut self) -> &mut i64 {
        &mut self.funds
    }
}

fn main() {
    let mut w = Wallet { funds: 0 };
    w.deposit_all(&[100, 200, 300]);
    println!("balance = {}", w.balance()); // 600
}
```

`Wallet`은 `balance`와 `balance_mut`만 구현했는데, `deposit`과 `deposit_all`을 공짜로 얻었습니다. 파이썬의 상속과 무엇이 다를까요? 결정적 차이는 **명시성과 충돌 처리**입니다. 두 트레이트가 같은 이름의 기본 메서드를 제공하고 한 타입이 둘 다 구현하면, Rust는 **컴파일 에러**를 내어 어느 쪽을 쓸지 명시(`<Wallet as Balance>::deposit(...)`)하라고 강제합니다. 파이썬의 MRO처럼 "조용히 왼쪽 우선"으로 넘어가지 않습니다. 모호함은 런타임 버그가 아니라 컴파일 에러가 됩니다.

### 컴파일러 에러 시연: 메서드 모호성 (E0034)

말로만 하면 와닿지 않으니 실제로 부딪쳐 봅시다. 8.1의 다이아몬드 문제를 트레이트로 옮긴 형태입니다 — `Html`과 `Json` 두 트레이트가 모두 `render`를 정의하고, `Widget`이 둘 다 구현합니다.

```rust
trait Html { fn render(&self) -> String; }
trait Json { fn render(&self) -> String; }

struct Widget;
impl Html for Widget { fn render(&self) -> String { "<html/>".to_string() } }
impl Json for Widget { fn render(&self) -> String { "{}".to_string() } }

fn main() {
    let w = Widget;
    // 🚨 컴파일 에러! Html::render 인가 Json::render 인가?
    println!("{}", w.render());
}
```

`rustc`의 응답은 파이썬의 MRO처럼 *조용히 하나를 고르는* 대신, 두 후보를 나란히 펼쳐 보이며 손을 듭니다.

```text
error[E0034]: multiple applicable items in scope
  --> e0034.rs:11:22
   |
11 |     println!("{}", w.render());
   |                      ^^^^^^ multiple `render` found
   |
note: candidate #1 is defined in an impl of the trait `Html` for the type `Widget`
note: candidate #2 is defined in an impl of the trait `Json` for the type `Widget`
help: disambiguate the method for candidate #1
   |
11 -     println!("{}", w.render());
11 +     println!("{}", Html::render(&w));
   |
help: disambiguate the method for candidate #2
   |
11 -     println!("{}", w.render());
11 +     println!("{}", Json::render(&w));
   |
```

`error[E0034]: multiple applicable items in scope`. 이것이 파이썬 다이아몬드 문제(8.1)에 대한 Rust의 답입니다. 파이썬에서 `Widget().render()`는 MRO상 `HtmlMixin`을 *말없이* 골라 `<html>...`을 반환하고, 그 선택이 왜 그렇게 됐는지는 C3 선형화 알고리즘을 머릿속으로 돌려야 알 수 있었습니다. Rust는 같은 상황에서 멈춰 서서 "후보가 둘인데 너의 의도를 모르겠다"고 말하고, 심지어 **고쳐 쓸 두 가지 정답(`Html::render(&w)` / `Json::render(&w)`)을 직접 제시**합니다. 해소는 **완전 정규화 문법(Fully Qualified Syntax)**으로 합니다.

```rust
fn main() {
    let w = Widget;
    println!("{}", Html::render(&w));            // <html/>
    println!("{}", <Widget as Json>::render(&w)); // {}
}
```

어느 트레이트의 메서드를 호출할지가 코드에 명시되어, 6개월 뒤 이 코드를 읽는 사람도 MRO를 추론할 필요가 없습니다. *모호함을 런타임 동작이 아니라 컴파일 에러로 환원*하는 것 — 이 챕터가 반복해 온 주제가 메서드 디스패치에서도 그대로 작동합니다.

> **Under the Hood: MRO 런타임 탐색 vs 트레이트 정적 해소**
>
> 파이썬에서 `w.deposit(100)`을 호출하면, 인터프리터는 `type(w).__mro__`를 따라가며 `deposit` 속성을 **런타임에 탐색**합니다. MRO 캐시가 있어도 이름 기반 딕셔너리 조회는 피할 수 없고, 각 호출은 본질적으로 동적 디스패치입니다.
>
> Rust에서 `w.deposit(100)`은 컴파일 타임에 `<Wallet as Balance>::deposit`으로 **완전히 정적 해소**됩니다. 제네릭 함수 `fn pay<T: Balance>(t: &mut T)` 안에서 호출하면 단일형화로 `Wallet` 전용 코드가 생성되고, 호출은 직접 함수 호출(혹은 인라인)이 됩니다. 점프 테이블도, vtable도, 이름 조회도 없습니다. MRO 순회 비용은 정확히 0이 됩니다.
>
> 8.1에서 측정한 파이썬 5단계 상속 메서드 호출(약 23 ns/call)과 같은 일을 Rust 트레이트 기본 메서드의 정적 디스패치로 500만 번 호출해 비교했습니다.
>
> ```text
> python deep-inheritance method call : ~23.0 ns/call
> rust  static-dispatch  trait  call  : ~0.25 ns/call (--release)
> ```
>
> 약 90배 차이입니다. 그런데 이 수치마저 과소평가입니다. Rust 쪽 0.25 ns는 사실상 "루프가 거의 통째로 최적화되어 사라진" 결과입니다. 메서드가 상수를 반환하므로 컴파일러가 호출을 인라인하고 결과를 접어버려, *MRO 탐색에 대응하는 작업 자체가 기계어에 존재하지 않습니다.* 파이썬은 호출마다 동적 디스패치를 *반드시* 수행해야 하지만, Rust는 그 작업을 컴파일 타임에 증발시킵니다.
>
> 동적 디스패치가 필요할 때(`&dyn Balance`)는 vtable을 거치지만, 그것도 포인터 한 번 따라가는 *고정 비용*이지 MRO처럼 계층 깊이에 비례해 길어지는 *가변 탐색*이 아닙니다. 이 둘(static `T: Trait` vs dynamic `dyn Trait`)의 선택은 Chapter 10에서 본격적으로 다룹니다.

여기서 트레이트와 상속의 결정적 차이 하나를 더 짚어야 합니다. 파이썬에서는 `str`이나 `dict` 같은 *남이 만든 클래스*에 메서드를 추가하려면 서브클래싱하거나 몽키패칭해야 합니다. Rust에서는 **고아 규칙(Orphan Rule)** 하에서, 여러분이 정의한 트레이트라면 표준 라이브러리의 `i32`나 `String`에도 `impl`을 붙여 행동을 *확장*할 수 있습니다. 데이터를 물려받지 않고도 행동만 깔끔하게 덧붙이는 이 능력은, 상속이 흉내낼 수 없는 트레이트만의 유연함입니다. 자세한 트레이트 구현 규칙은 Chapter 9에서 다룹니다.

---

## 8.6 실전 Hands-on: Django 스타일 BaseModel 계층 리팩토링

이제 모든 조각을 모아, 현실적인 Django 모델 계층을 Rust로 옮겨 봅시다. 출발점은 다중 상속을 쓰는 전형적인 Django 코드입니다.

* **The Python Way (다중 상속)**
  ```python
  from __future__ import annotations
  from django.db import models   # 개념 예시

  class TimestampedModel(models.Model):
      created_at = models.DateTimeField(auto_now_add=True)
      updated_at = models.DateTimeField(auto_now=True)
      class Meta:
          abstract = True

  class SoftDeleteModel(models.Model):
      deleted_at = models.DateTimeField(null=True)
      def soft_delete(self):
          self.deleted_at = timezone.now()
          self.save()
      class Meta:
          abstract = True

  # 다중 상속으로 두 추상 모델의 필드/행동을 한꺼번에 끌어온다
  class Post(TimestampedModel, SoftDeleteModel):
      title = models.CharField(max_length=200)
  ```
  `Post`는 `created_at`, `updated_at`, `deleted_at`을 상속받고, `soft_delete()`도 물려받습니다. 깔끔해 보이지만 — `TimestampedModel`의 필드 이름이 바뀌면, `super()` 체인이 꼬이면, 혹은 세 번째 믹스인을 추가하면 — 우리는 다시 8.1의 함정들로 돌아갑니다.

* **The Rust Implementation (컴포지션 + 트레이트)**
  전략은 명확합니다. **데이터는 재사용 가능한 조각 구조체로** 뽑아내 컴포지션하고, **공유 행동은 트레이트의 기본 메서드로** 제공합니다.

  ```rust
  // === 1. 재사용 가능한 데이터 조각 (Django의 abstract 필드 묶음) ===
  #[derive(Debug, Clone)]
  pub struct Timestamps {
      pub created_at: u64,
      pub updated_at: u64,
  }

  impl Timestamps {
      pub fn new(now: u64) -> Self {
          Self { created_at: now, updated_at: now }
      }
  }

  #[derive(Debug, Clone, Default)]
  pub struct SoftDelete {
      pub deleted_at: Option<u64>,
  }

  // === 2. 공유 행동의 계약 (Django의 abstract 메서드) ===
  pub trait Timestamped {
      fn timestamps(&self) -> &Timestamps;
      fn timestamps_mut(&mut self) -> &mut Timestamps;

      fn touch(&mut self, now: u64) {
          self.timestamps_mut().updated_at = now;
      }
      fn age(&self, now: u64) -> u64 {
          now - self.timestamps().created_at
      }
  }

  pub trait SoftDeletable {
      fn deletion(&self) -> &SoftDelete;
      fn deletion_mut(&mut self) -> &mut SoftDelete;

      fn soft_delete(&mut self, now: u64) {
          self.deletion_mut().deleted_at = Some(now);
      }
      fn is_deleted(&self) -> bool {
          self.deletion().deleted_at.is_some()
      }
  }

  // === 3. 구체 모델: 조각들을 필드로 '소유' (has-a) ===
  #[derive(Debug, Clone)]
  pub struct Post {
      pub title: String,
      pub ts: Timestamps,
      pub sd: SoftDelete,
  }

  impl Post {
      pub fn new(title: &str, now: u64) -> Self {
          Self {
              title: title.to_string(),
              ts: Timestamps::new(now),
              sd: SoftDelete::default(),
          }
      }
  }

  // === 4. 트레이트 구현: 어느 필드가 그 역할을 하는지만 알려준다 ===
  impl Timestamped for Post {
      fn timestamps(&self) -> &Timestamps { &self.ts }
      fn timestamps_mut(&mut self) -> &mut Timestamps { &mut self.ts }
  }
  impl SoftDeletable for Post {
      fn deletion(&self) -> &SoftDelete { &self.sd }
      fn deletion_mut(&mut self) -> &mut SoftDelete { &mut self.sd }
  }

  // === 5. 제네릭 함수: "Timestamped이면서 SoftDeletable인 무엇이든" ===
  fn summarize<T: Timestamped + SoftDeletable + std::fmt::Debug>(item: &T, now: u64) {
      println!(
          "age={} deleted={} :: {item:?}",
          item.age(now),
          item.is_deleted()
      );
  }

  fn main() {
      let mut post = Post::new("Composition over Inheritance", 1000);
      post.touch(1500);
      summarize(&post, 2000);
      // age=1000 deleted=false :: Post { title: "...", ts: Timestamps { ... }, sd: SoftDelete { deleted_at: None } }

      post.soft_delete(2500);
      summarize(&post, 3000);
      // age=2000 deleted=true :: Post { ..., sd: SoftDelete { deleted_at: Some(2500) } }
  }
  ```

  이 코드는 `cargo run`으로 그대로 실행됩니다. 무엇이 달라졌는지 음미해 봅시다.

  - **초기화 누락이 불가능합니다.** `Post::new`는 `ts`와 `sd`를 *반드시* 채워야 컴파일됩니다. 파이썬의 `super().__init__()` 누락 버그(8.1) 같은 것이 원천 차단됩니다. 어느 필드 하나라도 빠뜨리면 `error[E0063]: missing field`로 거부됩니다.
  - **MRO도, 다이아몬드도 없습니다.** `Post`가 두 트레이트를 구현할 뿐이며, 두 트레이트의 메서드 이름이 겹치지 않는 한 모호함이 없습니다. 겹친다면? 컴파일 에러로 즉시 알려 줍니다.
  - **새 모델 추가가 조립입니다.** `Comment`라는 새 타입을 만들고 싶다면, `Timestamps`와 `SoftDelete` 필드를 넣고 두 트레이트를 구현하면 끝입니다. `summarize`는 수정 없이 `Comment`에도 동작합니다 — 진짜 다형성이 상속 없이 달성됩니다.
  - **`Timestamps`의 내부가 바뀌어도** `Post`는 트레이트 계약(`timestamps()`)만 지키면 영향이 없습니다. 취약한 기반 클래스 문제가 사라집니다.

  "새 모델 추가가 조립"이라는 주장을 코드로 확인해 봅시다. `Comment` 타입을 추가하는 데 위 트레이트들은 **단 한 줄도 수정하지 않습니다.**

  ```rust
  #[derive(Debug, Clone)]
  pub struct Comment {
      pub body: String,
      pub author_id: u64,
      pub ts: Timestamps, // 같은 조각을 재사용
      pub sd: SoftDelete, // 같은 조각을 재사용
  }

  impl Comment {
      pub fn new(body: &str, author_id: u64, now: u64) -> Self {
          Self {
              body: body.to_string(),
              author_id,
              ts: Timestamps::new(now),
              sd: SoftDelete::default(),
          }
      }
  }

  // 어느 필드가 그 역할인지만 알려주면 touch/age/soft_delete/is_deleted를 공짜로 얻는다
  impl Timestamped for Comment {
      fn timestamps(&self) -> &Timestamps { &self.ts }
      fn timestamps_mut(&mut self) -> &mut Timestamps { &mut self.ts }
  }
  impl SoftDeletable for Comment {
      fn deletion(&self) -> &SoftDelete { &self.sd }
      fn deletion_mut(&mut self) -> &mut SoftDelete { &mut self.sd }
  }

  fn main() {
      let mut c = Comment::new("nice post!", 7, 1000);
      c.touch(1200);
      // 8.6의 summarize 함수를 수정 없이 그대로 재사용
      summarize(&c, 2000);
  }
  ```

  `summarize<T>`는 `Post`를 위해 작성했지만 `Comment`에도 즉시 동작합니다. 제네릭 경계 `T: Timestamped + SoftDeletable`가 "이 두 행동을 할 수 있는 타입이면 무엇이든"을 의미하기 때문입니다. 상속이라면 `Post`와 `Comment`가 공통 부모 아래 묶여야 했을 다형성이, 여기서는 **두 평평한 트레이트의 교집합**으로 달성됩니다.

  세 번째 관심사(예: `Versioned`)가 필요하면 조각 구조체 + 트레이트를 하나 더 추가해 조립하면 됩니다. 상속 깊이가 깊어지는 것이 아니라, **수평으로 조립되는** 구조입니다. Django에서 믹스인을 네 개, 다섯 개 쌓을수록 MRO가 미궁이 되던 것과 달리, 트레이트 경계는 몇 개를 더하든 `A + B + C + D`로 평평하게 나열될 뿐 깊이가 생기지 않습니다.

---

## 8.7 Best Practices & 안티패턴

실무에서 상속을 컴포지션으로 옮길 때의 체크리스트입니다.

**이렇게 하라 (Do)**
- ✅ **"is-a"가 아니라 "has-a"로 사고하라.** `Article extends BaseDocument` 대신 `Article { base: BaseDocument }`. 거의 모든 상속은 컴포지션으로 더 명확하게 표현됩니다.
- ✅ **도메인 식별자·검증 값은 뉴타입으로 감싸라.** `UserId(u64)`, `EmailAddress(String)`, `NonEmptyName(String)`. 런타임 비용 0으로 타입 안전성과 불변식을 얻습니다.
- ✅ **공유 행동은 트레이트의 기본 메서드로 제공하라.** 데이터(필드)와 행동(메서드)을 분리하면 조합이 자유로워집니다.
- ✅ **위임 메서드는 의도적으로 노출할 것만 작성하라.** 한 줄의 위임 코드가 "이 기능을 공개 API로 약속한다"는 선언입니다.
- ✅ **위임 보일러플레이트가 정말 많다면 `delegate` 같은 크레이트나 매크로를 고려하라**(Chapter 19에서 매크로를 다룹니다). 단, 명시성을 잃지 않는 선에서.

**이건 피하라 (Don't)**
- ❌ **`Deref`로 모든 것을 노출하지 마라.** `Deref`는 스마트 포인터용입니다. 도메인 뉴타입에 `Deref`를 걸면 캡슐화가 무너집니다. Clippy는 이 남용을 경고하지 않습니다(앞의 정정 박스). 필요한 메서드만 명시하거나 `AsRef`를 쓰세요.
- ❌ **트레이트로 "깊은 계층"을 흉내내려 하지 마라.** 트레이트는 평평한 인터페이스이지 클래스 트리가 아닙니다. 슈퍼트레이트(`trait B: A`)는 가능하지만, 상속 트리를 그대로 옮기는 도구로 오용하지 마세요.
- ❌ **거대한 "갓 구조체(God Struct)"를 만들지 마라.** 모든 필드를 하나에 몰아넣지 말고, `Timestamps`·`SoftDelete`처럼 응집도 높은 조각으로 나눠 컴포지션하세요.
- ❌ **뉴타입에 `pub` 필드를 두지 마라.** `pub struct Email(pub String)`은 외부에서 검증 없이 `Email(invalid)`를 만들 수 있게 합니다. 필드는 비공개로 두고 `new() -> Result<...>`로만 생성하세요.

---

## 8.8 흔한 함정 — 상속을 흉내내려는 시도

파이썬 습관이 Rust에서 어떻게 벽에 부딪히는지, 그리고 올바른 길은 무엇인지 봅시다.

**함정 1: "그냥 부모 구조체를 상속하면 안 되나?"**

```rust
struct Base {
    id: i32,
}

// 🚨 이런 문법은 존재하지 않는다
// struct Article : Base {
//     title: String,
// }
```

`struct Article : Base { ... }` 같은 문법은 없습니다. Rust 컴파일러는 애초에 이런 구문을 파싱하지 못합니다(`error: expected ...`). 해결책은 8.3의 컴포지션입니다: `struct Article { base: Base, title: String }`. 처음에는 답답하지만, 이 제약이 바로 취약한 기반 클래스 문제를 막아 줍니다.

**함정 2: 뉴타입을 만들어 놓고 `i32`처럼 산술 연산을 기대하기**

```rust
struct Cents(i64);

fn main() {
    let a = Cents(100);
    let b = Cents(250);
    // 🚨 컴파일 에러! Cents에는 + 연산이 정의되어 있지 않다
    let total = a + b;
    println!("{}", total.0);
}
```

```text
error[E0369]: cannot add `Cents` to `Cents`
 --> src/main.rs:7:19
  |
7 |     let total = a + b;
  |                 - ^ - Cents
  |                 |
  |                 Cents
  |
note: an implementation of `Add` might be missing for `Cents`
```

뉴타입은 내부 타입의 연산자를 자동으로 물려받지 않습니다(이것은 *기능*입니다 — "센트끼리 곱하기"는 도메인상 난센스일 수 있으니까요). 덧셈이 의미 있다면 `std::ops::Add`를 명시적으로 구현해야 합니다(연산자 오버로딩은 Chapter 9에서). 의도한 연산만 노출하는 것이 뉴타입의 목적입니다.

**함정 3: 다중 상속의 "협력적 super()"를 그리워하기**

파이썬의 다이아몬드(8.1)에서 본 `super().render()` 체인은 Rust에 등가물이 없습니다 — 그리고 그것은 다행입니다. 여러 동작을 *순서대로 합성*하고 싶다면, MRO에 의존하는 암묵적 체인 대신 **명시적 조합**을 쓰세요.

```rust
trait Render {
    fn render(&self, inner: String) -> String;
}

struct Html;
struct Json;

impl Render for Html {
    fn render(&self, inner: String) -> String {
        format!("<html>{inner}</html>")
    }
}
impl Render for Json {
    fn render(&self, inner: String) -> String {
        format!("{{\"r\": \"{inner}\"}}")
    }
}

fn main() {
    // 합성 순서를 코드로 명시한다 — MRO가 몰래 정하지 않는다
    let out = Html.render(Json.render("base".to_string()));
    println!("{out}"); // <html>{"r": "base"}</html>
}
```

파이썬에서는 `super()`가 합성 순서를 *암묵적으로* 결정해 추론을 어렵게 만들었습니다. Rust에서는 `Html.render(Json.render(...))`처럼 합성 순서가 코드에 그대로 드러납니다. 읽는 사람이 MRO 알고리즘을 머릿속으로 돌릴 필요가 없습니다.

**함정 4: 위임이 귀찮아서 내부 필드를 `pub`으로 열기**

`post.ts.updated_at = now`처럼 내부 조각의 필드를 외부에서 직접 만지면 불변식이 깨집니다. 8.6처럼 트레이트 메서드(`touch`)나 구조체 메서드를 통해서만 상태를 바꾸도록 캡슐화하세요. 귀찮음을 못 이겨 `pub`을 남발하는 순간, 컴포지션의 장점(통제된 변경 경로)이 사라집니다.

---

## 8.9 연습문제

머리로 이해한 것과 손으로 쓸 수 있는 것은 다릅니다. 아래 세 문제를 *먼저 직접 풀어 본 뒤* 정답 스케치와 대조하세요. 각 문제에는 스스로 검증할 수 있는 컴파일러 에러 코드 또는 측정값을 달아 두었습니다. Rustlings처럼, 컴파일러를 채점관 삼아 빨간 줄이 사라질 때까지 고치면 됩니다.

**문제 1 — 상속을 컴포지션으로 (Composition)**

파이썬의 다음 상속 코드를 Rust의 컴포지션 + 명시적 위임으로 옮기세요. `Car`는 `Engine`을 *상속*하지 않고 *소유*해야 하며, `start()`만 외부에 노출하고 엔진의 다른 메서드는 새어 나오지 않아야 합니다.

```python
class Engine:
    def __init__(self, horsepower: int) -> None:
        self.horsepower = horsepower
    def start(self) -> str:
        return f"engine roaring at {self.horsepower}hp"

class Car(Engine):                      # ← 이 상속을 없애라
    def __init__(self, model: str, hp: int) -> None:
        super().__init__(hp)
        self.model = model
```

> **정답 스케치.** 핵심 아이디어는 `struct Car { engine: Engine, model: String }`(has-a)와, `Car::start`가 `self.engine.start()`를 *명시적으로* 위임하는 것입니다. `Engine`의 필드를 `pub`으로 열지 말고 `Engine::new`로만 만들게 하면 캡슐화가 유지됩니다.
>
> ```rust
> pub struct Car { engine: Engine, model: String }
> impl Car {
>     pub fn new(model: &str, hp: u32) -> Self {
>         Self { engine: Engine::new(hp), model: model.to_string() }
>     }
>     pub fn start(&self) -> String {
>         format!("[{}] {}", self.model, self.engine.start())
>     }
> }
> ```
>
> **자기검증:** `Car`에서 위임 메서드를 *지우고* `car.start()`를 호출해 보세요. 상속이 아니므로 `Engine::start`가 자동으로 따라오지 않아 `error[E0599]: no method named 'start' found for struct 'Car'`가 떠야 정상입니다. 위임을 다시 넣으면 출력은 `[RustMobile] engine roaring at 320hp`.

**문제 2 — 검증 뉴타입 + 의도한 연산만 (Newtype)**

음수가 될 수 없는 금액 타입 `Cents`를 만드세요. (a) 생성 시 음수면 `Err`를 반환하고, (b) `Cents + Cents`는 허용하되 (c) `Cents * Cents`는 *컴파일조차 안 되게* 두세요. 내부 필드는 비공개여야 합니다.

> **정답 스케치.** `pub struct Cents(i64)`의 필드를 비공개로 두고, `Cents::new(v) -> Result<Self, String>`에서 `v < 0`을 거릅니다. `+`는 도메인상 의미가 있으니 `impl std::ops::Add for Cents`만 구현하고, `*`는 의미가 없으니 **구현하지 않습니다.** 연산자를 자동으로 물려받지 않는 것이 뉴타입의 *기능*입니다(8.8 함정 2).
>
> ```rust
> impl std::ops::Add for Cents {
>     type Output = Cents;
>     fn add(self, rhs: Cents) -> Cents { Cents(self.0 + rhs.0) }
> }
> ```
>
> **자기검증:** `Cents::new(150)? + Cents::new(250)?`의 `.value()`는 **400**이어야 합니다. `Cents::new(-1).is_err()`는 `true`. 그리고 `a * b`를 시도하면 `error[E0369]: cannot multiply 'Cents' by 'Cents'`가 떠야 합니다(8.8의 `E0369`와 동일 계열). 만약 `derive`에 `Ord`를 넣고 싶어졌다면 — `i64`는 `Ord`라 통과하지만, 내부 타입을 `f64`로 바꾸는 순간 8.4에서 본 `error[E0277]: the trait bound 'f64: Ord' is not satisfied`로 막힙니다.

**문제 3 — `Deref`를 쓸 곳과 쓰지 말 곳 (Deref)**

두 타입을 설계하세요. (a) `OwnedBuffer(Vec<u8>)`는 바이트 슬라이스의 거의 모든 API를 그대로 노출해야 하는 "얇은 래퍼"입니다. (b) `EmailAddress(String)`은 검증 불변식을 가진 도메인 개념입니다. 각각에 `Deref`를 걸지 말지 *결정*하고, 그 근거를 코드로 보이세요.

> **정답 스케치.** (a)는 스마트 포인터에 가까우므로 `impl Deref for OwnedBuffer { type Target = [u8]; ... }`이 정당합니다 — `buf.len()`, `buf.iter()` 등이 자동으로 동작합니다. (b)는 8.4의 교훈대로 `Deref`를 **걸지 않습니다.** 대신 `impl AsRef<str>`과 명시 메서드 `as_str()`만 둡니다. 그래야 `email.to_uppercase()` 같은 무의미한 호출이 *컴파일되지 않아* 불변식이 보호됩니다.
>
> ```rust
> fn print_len(s: impl AsRef<str>) { println!("{}", s.as_ref().len()); }
> // print_len(&email)은 OK. 하지만 email.to_uppercase()는 컴파일 에러.
> ```
>
> **자기검증:** `EmailAddress`에 `Deref` 없이 `email.to_uppercase()`를 호출하면 `error[E0599]: no method named 'to_uppercase' found for struct 'EmailAddress'`가 떠야 정상입니다. 반대로 `OwnedBuffer`에 `Deref`를 걸면 `buf.len()`이 동작하고 `vec![1,2,3]` 버퍼에 대해 **3**을 반환합니다. 만약 `EmailAddress`에도 `Deref`를 걸어 버리면 컴파일은 통과하지만 — 그 순간 `str`의 200여 메서드가 전부 새어 나와 뉴타입을 만든 의미가 사라집니다. clippy는 이를 잡아주지 *않으니*(8.4 정정 박스), 이 선택은 순전히 여러분의 설계 규율에 달려 있습니다.

---

Chapter 7에서 우리는 `dataclass`와 Pydantic 모델을 `struct`로 옮기며 데이터의 *형태*를 모델링했습니다. 이번 챕터에서는 그 위에 얹혀 있던 OOP의 가장 무거운 유산 — **상속** — 을 내려놓고, 데이터의 재사용은 **컴포지션**으로, 행동의 공유는 **트레이트**로 분리하는 사고방식을 익혔습니다. 핵심은 하나입니다: Rust는 "is-a"의 암묵적 결합을 거부하고, "has-a"와 "can-do"의 명시적 조립을 택했으며, 그 대가로 취약한 기반 클래스·MRO 모호함·런타임 탐색 비용을 모두 컴파일 타임에 제거했습니다.

이번 챕터에서 트레이트는 "행동 공유의 도구"로만 살짝 다뤘습니다. 하지만 트레이트는 그보다 훨씬 강력합니다. 파이썬 클래스의 진짜 마법인 **던더 메서드(`__str__`, `__add__`, `__eq__` ...)**, 즉 덕 타이핑의 근간이 Rust에서 어떻게 표준 트레이트(`Display`, `Add`, `PartialEq`)로 옮겨지며, 그것이 어떻게 컴파일 타임에 강제되는지 — 바로 다음 **Chapter 9. 매직 메서드(Dunder)에서 Trait으로**에서 본격적으로 파헤치겠습니다.

---

## References

- https://doc.rust-lang.org/book/ch18-01-what-is-oo.html
- https://doc.rust-lang.org/book/ch10-02-traits.html
- https://doc.rust-lang.org/book/ch20-03-advanced-traits.html
- https://doc.rust-lang.org/rust-by-example/generics/new_types.html
- https://doc.rust-lang.org/rust-by-example/trait.html
- https://doc.rust-lang.org/nomicon/other-reprs.html#reprtransparent
- https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp
- https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait
- https://docs.rs/ordered-float/latest/ordered_float/
