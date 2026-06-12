# Chapter 19. 매크로 (Macros): 데코레이터와 메타클래스의 대체재

파이썬 개발자들은 코드의 중복을 없애고 로직을 주입하기 위해 **데코레이터(Decorator)**를 사랑합니다. 함수 위쪽에 `@timer`나 `@login_required`를 붙이면, 런타임에 원본 함수를 감싸는(Wrapping) 새로운 함수가 동적으로 생성됩니다. 더 나아가, 클래스가 생성되는 방식 자체를 뜯어고치기 위해 **메타클래스(Metaclass)**라는 흑마술을 부리기도 합니다.

하지만 이 모든 것들은 "런타임(Runtime)"에 수행됩니다. 실행 속도를 갉아먹고, 타입 힌팅을 깨트리며, 디버깅 콜스택을 더럽히는 주범입니다.

Rust는 런타임 오버헤드 없이 코드 생성을 자동화하기 위해 **매크로(Macro)**를 사용합니다. Rust의 매크로는 C 언어의 단순한 텍스트 치환(`#define`)을 넘어선, **"코드를 짜는 코드(Metaprogramming)"** 그 자체입니다.

---

## 19.1 선언적 매크로 (Declarative Macros)

가장 흔히 보게 될 매크로는 이름 뒤에 느낌표(`!`)가 붙은 녀석들입니다. `println!()`이나 `vec!()`이 대표적입니다. 왜 이들은 함수가 아니라 매크로일까요?

파이썬의 `print()` 함수는 매개변수를 몇 개든 받을 수 있습니다 (`*args`). 하지만 Rust의 함수는 가변 인자(Variadic Arguments)를 허용하지 않습니다. 매개변수 개수가 고정되어야만 컴파일러가 메모리 크기를 정확히 예측할 수 있기 때문입니다.

따라서 임의의 개수를 받는 함수 대신, 컴파일 타임에 **입력된 개수만큼의 실제 코드로 팽창(Expand)**하는 매크로를 사용합니다.

* **The Rust Implementation (`macro_rules!`)**
  ```rust
  // 우리가 자주 쓰던 vec! 매크로의 단순화된 내부 구현 모습입니다.
  macro_rules! my_vec {
      // ( $(패턴),* ) => ( 치환할 코드 )
      ( $( $x:expr ),* ) => {
          {
              let mut temp_vec = Vec::new();
              $(
                  temp_vec.push($x); // 입력된 개수만큼 이 줄이 반복 생성됨
              )*
              temp_vec
          }
      };
  }

  fn main() {
      // 개발자가 이렇게 적으면
      let v = my_vec![1, 2, 3]; 

      /*
      // 컴파일러는 빌드하기 전에 코드를 아래와 같이 통째로 바꿔버립니다.
      let v = {
          let mut temp_vec = Vec::new();
          temp_vec.push(1);
          temp_vec.push(2);
          temp_vec.push(3);
          temp_vec
      };
      */
  }
  ```
런타임에는 그저 평범한 변수 할당과 `push` 연산 3번이 일어날 뿐입니다. 데코레이터처럼 런타임에 함수 객체를 동적으로 생성하는 오버헤드가 단 1도 없습니다.

---

## 19.2 절차적 매크로 (Procedural Macros): 데코레이터의 진화

파이썬의 `@decorator`와 시각적으로 가장 비슷한 것이 바로 절차적 매크로입니다. 앞서 우리가 `#[derive(Debug, Serialize)]` 라고 적었을 때 뒤에서 일어난 마법이 바로 이것입니다.

절차적 매크로는 **"컴파일 타임에 실행되는 또 다른 Rust 프로그램"**입니다. 이 프로그램은 소스 코드의 추상 구문 트리(AST, Abstract Syntax Tree)를 입력으로 받아, 분석하고 조작한 뒤, 새로운 Rust 코드를 출력합니다.

* **The Python Way (Decorator)**
  ```python
  import time
  from functools import wraps

  def timer(func):
      @wraps(func)
      def wrapper(*args, **kwargs):
          start = time.time()
          result = func(*args, **kwargs)
          print(f"{func.__name__} took {time.time() - start}s")
          return result
      return wrapper

  @timer
  def heavy_computation():
      sum(i * i for i in range(1000000))
  ```

* **The Rust Way (Attribute-like Procedural Macro)**
  Rust에서는 매크로를 작성하는 전용 라이브러리(Crate)를 따로 분리해야 하는 등 초기 설정이 복잡합니다. 하지만 한 번 만들어두면 사용법은 파이썬과 똑같습니다.

  ```rust
  // 외부 라이브러리(예: my_macros)에서 만든 매크로를 가져옵니다.
  use my_macros::timer;

  // 파이썬의 @timer 와 동일하게 붙입니다.
  #[timer]
  fn heavy_computation() {
      let _sum: i64 = (0..1000000).map(|i| i * i).sum();
  }
  ```

  이 `#[timer]` 매크로는 컴파일러가 코드를 빌드할 때 다음과 같이 코드를 **재작성(Rewrite)**해 줍니다.

  ```rust
  // 컴파일러가 팽창시킨 코드
  fn heavy_computation() {
      let start = std::time::Instant::now(); // 매크로가 삽입한 코드
      
      let _sum: i64 = (0..1000000).map(|i| i * i).sum(); // 원본 로직
      
      println!("heavy_computation took {:?}", start.elapsed()); // 매크로가 삽입한 코드
  }
  ```

### 왜 매크로가 메타클래스보다 나은가?
파이썬의 메타클래스나 몽키 패칭(Monkey Patching)을 사용하면, IDE의 자동 완성이 깨지고 `mypy`가 타입을 제대로 추론하지 못하는 문제가 흔히 발생합니다.

반면 Rust의 절차적 매크로는 `rust-analyzer` (IDE Language Server)가 매크로 팽창 결과를 실시간으로 분석합니다. 따라서 우리가 매크로를 통해 자동으로 생성한 메서드(`derive(Builder)` 등)들도 VSCode에서 완벽하게 자동 완성과 타입 체킹이 지원됩니다.

이로써 파이썬 개발자들이 코드의 유연성을 확보하기 위해 런타임에 부리던 마법들이, Rust에서는 컴파일 타임 매크로라는 견고하고 빠른 요새 안으로 어떻게 편입되는지 알아보았습니다. 다음 챕터에서는 Rust의 또 다른 마법, 참조 그래프를 속이는 **스마트 포인터**에 대해 살펴봅니다.
