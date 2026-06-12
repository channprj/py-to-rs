# Chapter 8. 상속(Inheritance)의 종말과 컴포지션(Composition)

객체 지향 프로그래밍(OOP)을 10년 이상 해온 파이썬 개발자에게 **상속(Inheritance)**은 코드 재사용을 위한 가장 자연스러운 도구입니다. `BaseModel`, `AbstractUser`, `Mixin` 등 수많은 부모 클래스를 조합하여(다중 상속, MRO) 자식 클래스를 만들어 내는 것은 파이썬 프레임워크(특히 Django)에서 흔히 볼 수 있는 패턴입니다.

하지만 유명한 격언이 있습니다.
> "나는 바나나를 원했는데, 바나나를 들고 있는 고릴라와 정글 전체를 얻게 되었다." — 조 암스트롱 (Erlang 창시자)

상속은 코드를 단단하게 결합(Coupling)시키며, 부모 클래스의 변경이 의도치 않게 모든 자식 클래스에 영향을 미치는 부작용(Fragile Base Class Problem)을 낳습니다.

**Rust에는 상속이 없습니다.** `extends` 키워드도, `super()` 함수도 존재하지 않습니다. Rust는 의도적으로 상속을 배제하고, 그 빈자리를 **컴포지션(Composition, 합성)**과 **트레이트(Trait)**로 채웠습니다.

---

## 8.1 Base Class를 버리고 컴포지션(Composition)으로

파이썬에서 흔히 작성하는 "기본 모델(Base Model)" 패턴을 살펴봅시다.

* **The Python Way (상속)**
  ```python
  class BaseDocument:
      def __init__(self, id: int, created_at: str):
          self.id = id
          self.created_at = created_at

      def get_metadata(self):
          return f"Doc {self.id} created at {self.created_at}"

  class Article(BaseDocument):
      def __init__(self, id: int, created_at: str, title: str, content: str):
          super().__init__(id, created_at) # 부모 초기화 호출
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

이 방식은 코드를 더 많이 타이핑해야 하는 것처럼 보이지만(특히 위임 메서드 작성 시), 아키텍처 관점에서는 **부모와 자식 간의 결합도를 완벽하게 분리**해 냅니다. 부모 구조체의 내부 구현이 바뀌어도, 자식 구조체는 명시적으로 위임한 인터페이스만 유지된다면 전혀 타격을 받지 않습니다.

---

## 8.2 내장 타입 상속의 대체: 뉴타입 패턴 (Newtype Pattern)

파이썬 개발자들이 타입 힌팅이나 검증 로직을 추가하기 위해 흔히 내장 타입을 상속받습니다.

* **The Python Way (내장 타입 상속)**
  ```python
  class EmailAddress(str):
      def __init__(self, value: str):
          if "@" not in value:
              raise ValueError("Invalid email format")
          # 내부적으로 str 초기화
          super().__init__()
          
  my_email = EmailAddress("user@example.com")
  print(my_email.upper()) # str의 모든 메서드를 사용할 수 있음
  ```

* **The Rust Implementation (Newtype Pattern)**
  Rust에서는 `String`이나 `i32` 같은 표준 라이브러리 타입을 직접 확장(상속)할 수 없습니다. 대신 **필드가 하나뿐인 튜플 구조체(Tuple Struct)**를 사용하여 기존 타입을 감싸는 '뉴타입 패턴'을 씁니다.

  ```rust
  // String을 감싸는 새로운 독립적인 타입 'EmailAddress' 선언
  #[derive(Debug, Clone, PartialEq)]
  pub struct EmailAddress(String);

  impl EmailAddress {
      // 생성 시점에 검증 로직 통과를 강제함
      pub fn new(value: &str) -> Result<Self, String> {
          if !value.contains('@') {
              return Err("Invalid email format".to_string());
          }
          Ok(EmailAddress(value.to_string()))
      }

      // 내부 String 값에 접근하는 Getter
      pub fn as_str(&self) -> &str {
          &self.0 // 튜플의 첫 번째 필드는 .0 으로 접근
      }
  }

  fn main() {
      // 1. 타입 안정성 확보
      // let email: EmailAddress = String::from("hello"); // 컴파일 에러! 타입 불일치
      
      let email = EmailAddress::new("user@example.com").unwrap();
      
      // 2. 메서드 제한을 통한 캡슐화
      // 파이썬처럼 기존 str의 upper() 같은 메서드를 멋대로 호출할 수 없음.
      // email.to_uppercase(); // 컴파일 에러! EmailAddress에는 이런 메서드가 없음.
      
      println!("Email is: {}", email.as_str());
  }
  ```

### 뉴타입 패턴의 강력함
뉴타입 패턴은 **"도메인 주도 설계(DDD, Domain Driven Design)"**의 핵심입니다. 
파이썬에서는 `User_ID`와 `Product_ID`가 모두 `int`라면, 실수로 상품 ID가 들어갈 자리에 유저 ID를 넣어도 코드가 정상적으로 실행됩니다(논리적 버그). 

Rust에서는 `struct UserId(i32);`와 `struct ProductId(i32);`로 뉴타입을 정의하면, 두 타입은 완전히 분리됩니다. `UserId`를 받는 함수에 `ProductId`를 넣으면 **컴파일 자체가 거부**됩니다. 런타임 오버헤드는 0입니다. 메모리상에서 `UserId`는 정확히 `i32`와 똑같은 크기와 구조를 갖기 때문입니다.

파이썬의 덕 타이핑(Duck Typing)과 다형성이 그립다면 어떻게 해야 할까요? 바로 다음 챕터에서 그 해답인 **매직 메서드와 Trait**에 대해 다루겠습니다.
