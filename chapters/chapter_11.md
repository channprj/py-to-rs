# Chapter 11. 믹스인(Mixin) 패턴과 생성(Creational) 패턴

파이썬 개발자들은 클래스의 기능을 조립식 블록처럼 끼워 넣기 위해 **다중 상속(Multiple Inheritance)** 기반의 **믹스인(Mixin)** 패턴을 즐겨 씁니다. Django 프레임워크의 `LoginRequiredMixin`이나 `UpdateModelMixin`이 대표적입니다.

또한 객체를 생성할 때 수많은 인자들을 유연하게 받기 위해 `*args`와 `**kwargs`를 남발하거나, 생성자(`__init__`) 내부를 복잡한 로직으로 도배하곤 합니다.

하지만 Rust는 상속 자체를 지원하지 않으며, 함수에서 가변 인자(`*args`)나 키워드 인자(`**kwargs`)도 지원하지 않습니다. 그렇다면 파이썬 특유의 이 유연성을 Rust에서는 어떻게 아키텍처로 풀어낼까요?

---

## 11.1 다중 상속과 Mixin의 한계

먼저 파이썬의 전형적인 Mixin 코드를 살펴보겠습니다.

* **The Python Way (Mixin)**
  ```python
  class TimestampMixin:
      # 자체적인 데이터와 로직을 가짐
      def get_created_at(self):
          return self.created_at

  class LoggableMixin:
      def log_state(self):
          print(f"[{self.__class__.__name__}] Current state logged.")

  # 다중 상속을 통해 조립
  class User(TimestampMixin, LoggableMixin):
      def __init__(self, username: str, created_at: str):
          self.username = username
          self.created_at = created_at

  u = User("pythonista", "2023-01-01")
  u.log_state()
  ```
  파이썬의 Mixin은 편리하지만, **MRO(Method Resolution Order)**의 늪에 빠지기 십상입니다. 여러 Mixin 클래스들이 동일한 이름의 속성이나 메서드를 가질 경우, 어떤 순서로 덮어씌워질지 파악하기 위해 클래스 계층도를 전부 뒤져야 합니다.

---

## 11.2 Rust의 해법: Trait Default Methods

Rust는 데이터(Struct)의 상속은 불가능하지만, **행동(Trait)의 기본 구현(Default Methods)**을 제공함으로써 Mixin과 동일한 코드 재사용 효과를 안전하게 얻어냅니다.

* **The Rust Implementation**
  ```rust
  // 1. 공통 행동과 기본 구현을 담은 Trait 정의
  pub trait Loggable {
      // 구현을 강제하는 메서드 (반드시 구현해야 함)
      fn get_name(&self) -> &str;

      // 기본 구현(Default Method)이 제공되는 메서드 (Mixin 역할)
      // 구현체에서 오버라이딩(Overriding)하지 않아도 바로 사용할 수 있음
      fn log_state(&self) {
          println!("[{}] Current state logged.", self.get_name());
      }
  }

  // 2. Struct 정의 (상속 없이 순수 데이터만 가짐)
  pub struct User {
      pub username: String,
      pub created_at: String,
  }

  // 3. Trait(행동) 결합
  impl Loggable for User {
      fn get_name(&self) -> &str {
          &self.username // 요구된 메서드만 구현하면 됨
      }
      // log_state()는 기본 구현을 그대로 재사용함!
  }

  fn main() {
      let u = User {
          username: String::from("rustacean"),
          created_at: String::from("2023-01-01"),
      };
      
      u.log_state(); // [rustacean] Current state logged.
  }
  ```

Rust의 Trait은 파이썬 클래스의 `__dict__`를 오염시키지도 않고, 이름 충돌(Name Collision)이 발생하면 컴파일러가 명확하게 에러를 뿜어주어 안전한 확장을 보장합니다.

---

## 11.3 `**kwargs`의 종말과 Builder Pattern

파이썬 클래스의 또 다른 특징은 유연한 `__init__`입니다.

* **The Python Way (Kwargs & Optional)**
  ```python
  class ServerConfig:
      def __init__(self, host: str, port: int = 8080, **kwargs):
          self.host = host
          self.port = port
          self.timeout = kwargs.get("timeout", 30)
          self.tls = kwargs.get("tls", False)

  # 옵션 인자들을 맘대로 골라서 넘길 수 있음
  config = ServerConfig("localhost", tls=True)
  ```

* **The Rust Mental Model**
  Rust는 기본 인자(Default Arguments)나 키워드 인자(`**kwargs`) 문법이 없습니다. 수많은 선택적 매개변수를 처리하기 위해 Rust는 **빌더 패턴(Builder Pattern)**을 관례적으로(Idiomatic) 사용합니다.

  ```rust
  pub struct ServerConfig {
      host: String,
      port: u16,
      timeout: u32,
      tls: bool,
  }

  impl ServerConfig {
      // 필수 파라미터만 받는 builder() 시작점을 제공
      pub fn builder(host: &str) -> ServerConfigBuilder {
          ServerConfigBuilder {
              host: host.to_string(),
              port: 8080,      // 기본값
              timeout: 30,     // 기본값
              tls: false,      // 기본값
          }
      }
  }

  // 옵션 설정만을 담당하는 보조 구조체
  pub struct ServerConfigBuilder {
      host: String,
      port: u16,
      timeout: u32,
      tls: bool,
  }

  impl ServerConfigBuilder {
      // 메서드 체이닝(Chaining)을 위해 자기 자신(self)을 반환
      pub fn port(mut self, port: u16) -> Self {
          self.port = port;
          self
      }

      pub fn tls(mut self, tls: bool) -> Self {
          self.tls = tls;
          self
      }

      // 최종적으로 원본 객체를 조립해서 반환
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
      // 파이썬의 키워드 인자처럼 명시적이고 우아하게 생성 가능!
      let config = ServerConfig::builder("localhost")
          .port(443)
          .tls(true)
          .build();
  }
  ```

이 패턴은 너무 많이 쓰이기 때문에, Rust 생태계에서는 `derive_builder`라는 매크로 크레이트를 활용하여 이 엄청난 보일러플레이트 코드를 **컴파일 타임에 자동으로 생성**하게 만듭니다. 덕분에 개발자는 파이썬만큼 짧은 코드로, 파이썬보다 훨씬 타입 안전한 생성 로직을 짤 수 있습니다.

지금까지 3개의 파트에 걸쳐 파이썬 객체 지향 모델이 Rust에서 어떻게 '데이터와 행동의 분리'로 치환되는지 완전히 해부했습니다. 
다음 장(Part 4)에서는 `try-except`에 의존하던 파이썬의 예외 처리(Exception)가 Rust에서 어떻게 **우아한 값의 전파(Option, Result)**로 바뀌는지, 그 패러다임 전환을 다뤄보겠습니다.
