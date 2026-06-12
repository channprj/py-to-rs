# Chapter 13. 에러 전파와 커스텀 에러 설계

파이썬에서는 콜스택(Call Stack) 깊은 곳에서 예외를 던지면(`raise`), 그 예외는 처리될 때까지(except) 상위 호출자를 향해 자동으로 거슬러 올라갑니다. 이것이 예외(Exception)의 본질적인 전파 방식입니다.

Rust는 `Result`를 값으로 반환하기 때문에, 에러가 발생했을 때 이를 상위 호출자에게 전달하려면 코드가 매우 장황해집니다.

* **초기 러스트 프로그래머의 고통스러운 에러 전파**
  ```rust
  use std::fs::File;
  use std::io::{self, Read};

  fn read_username() -> Result<String, io::Error> {
      let f_result = File::open("hello.txt");

      let mut f = match f_result {
          Ok(file) => file,
          Err(e) => return Err(e), // 에러면 여기서 함수를 종료하고 Err 반환
      };

      let mut s = String::new();
      match f.read_to_string(&mut s) {
          Ok(_) => Ok(s), // 성공하면 결과 반환
          Err(e) => Err(e), // 에러면 Err 반환
      }
  }
  ```

이러한 `match`와 `return Err(e)`의 지루한 반복을 완벽하게 제거하기 위해 도입된 것이 바로 **`?` 연산자**입니다.

---

## 13.1 `?` 연산자: 파이썬 예외 전파의 완벽한 대체재

`?` 기호를 `Result` 값 뒤에 붙이면, 파이썬의 예외 전파와 동일한 효과를 냅니다.
- 만약 값이 `Ok`라면 내부의 데이터를 꺼내어 넘겨줍니다.
- 만약 값이 `Err`라면 **현재 실행 중인 함수를 즉시 종료하고 에러를 상위 호출자에게 반환**합니다.

* **The Rust Way (`?` 연산자 사용)**
  ```rust
  use std::fs::File;
  use std::io::{self, Read};

  // 파이썬만큼 짧고 읽기 쉬워졌습니다!
  fn read_username_short() -> Result<String, io::Error> {
      let mut s = String::new();
      // File::open이 실패하면 즉시 함수가 종료되고 io::Error가 반환됩니다.
      // 성공하면 파일 핸들이 f에 할당됩니다.
      let mut f = File::open("hello.txt")?; 
      
      // read_to_string이 실패하면 즉시 종료되고 에러가 반환됩니다.
      f.read_to_string(&mut s)?; 
      
      Ok(s) // 모든 것이 성공했을 때
  }

  // 더 줄이면 이렇게도 가능합니다 (Chaining).
  fn read_username_shorter() -> Result<String, io::Error> {
      let mut s = String::new();
      File::open("hello.txt")?.read_to_string(&mut s)?;
      Ok(s)
  }
  ```

---

## 13.2 에러 타입의 통합과 커스텀 에러

파이썬에서는 여러 종류의 에러(네트워크 에러, 파싱 에러, DB 에러)를 `Exception`이라는 하나의 부모 클래스로 뭉뚱그려 잡을 수 있습니다.

```python
try:
    data = fetch_data()
    parse_data(data)
except Exception as e: # 모든 에러를 캐치
    print(f"Something went wrong: {e}")
```

하지만 Rust의 `read_username` 함수는 `io::Error`만 반환하도록 타입이 고정되어 있습니다. 만약 파일도 읽고, 숫자로 파싱도 해야 한다면 `io::Error`와 `ParseIntError` 두 가지가 동시에 발생할 수 있습니다. 

어떻게 두 개의 다른 에러를 하나의 `Result<T, E>`로 반환할 수 있을까요?

### 방법 1. `Box<dyn std::error::Error>` (Trait Object 사용)
가장 쉽고 파이썬의 `Exception`과 유사한 방법입니다. "에러 트레이트를 구현한 아무 에러 객체나 힙에 넣어서 포인터로 반환하겠다"는 뜻입니다.

```rust
use std::fs::File;
use std::io::Read;
use std::error::Error;

// 어떤 에러든 반환할 수 있는 만능 시그니처
fn read_and_parse() -> Result<i32, Box<dyn Error>> {
    let mut s = String::new();
    File::open("number.txt")?.read_to_string(&mut s)?; // io::Error 발생 가능
    
    let num: i32 = s.trim().parse()?; // ParseIntError 발생 가능
    
    Ok(num)
}
```
* **장점**: 작성이 매우 빠르고 파이썬 개발자에게 익숙합니다. (스크립트 짤 때 유용)
* **단점**: 나중에 호출자가 에러 종류별로 정교한 처리를 하려고 할 때, 내부 에러 타입을 다시 다운캐스팅해야 하므로 귀찮습니다.

### 방법 2. Custom Error Enum 정의
엔터프라이즈급 백엔드를 구축할 때는 에러의 원인을 정확히 분류하기 위해 Enum을 사용합니다. 파이썬에서 커스텀 Exception 클래스들을 계층적으로 만드는 것과 같습니다.

```rust
use std::num::ParseIntError;
use std::io;

// 나만의 커스텀 에러 타입 (대수적 데이터 타입)
#[derive(Debug)]
enum MyAppError {
    Io(io::Error),
    Parse(ParseIntError),
    InvalidFormat(String), // 내가 직접 만든 에러
}

// 표준 라이브러리의 Error 트레이트 등을 구현해야 하지만
// 코드가 너무 길어지므로 생략... (이를 해결해 주는 것이 thiserror 크레이트입니다)
```

이 커스텀 에러 작성의 엄청난 보일러플레이트를 줄여주기 위해 러스트 생태계는 두 개의 표준적인 크레이트를 사용합니다.
1. **`anyhow`**: 애플리케이션 개발용. (어떤 에러든 쉽게 `?`로 넘기고 콜스택을 추적하고 싶을 때)
2. **`thiserror`**: 라이브러리 개발용. (사용자에게 명확한 Enum 에러 타입을 제공하고 싶을 때 매크로로 코드 생성)

이제 우리는 파이썬의 `try-except`가 런타임에 어떤 성능 저하와 타입 모호성을 주는지 알게 되었고, Rust의 `Result`와 `?` 연산자가 이를 어떻게 우아하고 안전하게 해결하는지 배웠습니다. 

다음 파트(Part 5)에서는 파이썬의 `List`, `Dict`와 러스트의 컬렉션, 그리고 `Generator(yield)`를 뛰어넘는 강력한 러스트의 `Iterator`에 대해 다루겠습니다.
