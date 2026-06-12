# Chapter 12. `try-except`는 없다: `Option`과 `Result`

파이썬 개발자들은 에러가 발생할 만한 곳에 본능적으로 `try-except` 블록을 감쌉니다. 키가 없는 딕셔너리에 접근할 때(`KeyError`), 파일을 열 때(`FileNotFoundError`), 네트워크 요청이 실패했을 때(`ConnectionError`) 모두 예외(Exception)를 던지고 받습니다.

* **The Python Way (Exceptions)**
  ```python
  def divide(a: float, b: float) -> float:
      if b == 0:
          raise ValueError("Cannot divide by zero")
      return a / b

  try:
      result = divide(10, 0)
  except ValueError as e:
      print(f"Error: {e}")
  ```

이 방식은 코드를 깔끔하게 만들어 주지만, 치명적인 단점이 있습니다. **"이 함수가 도대체 어떤 예외를 던질지 함수 시그니처만 보고는 절대 알 수 없다"**는 것입니다. 문서를 꼼꼼히 읽거나 소스 코드를 뒤져보지 않으면 예외 처리를 누락하기 십상이고, 결국 배포된 서버가 런타임에 뻗어버립니다 (Call Stack Unwinding).

Rust는 예외(Exception)라는 개념 자체를 언어에서 삭제했습니다. 대신 "값이 없거나", "연산이 실패했다"는 사실을 **반환 값의 타입(Type)으로 명시**합니다.

---

## 12.1 `None` (Null)의 대체: `Option<T>`

파이썬에서 값이 없을 때는 흔히 `None`을 반환합니다. 10억 달러의 실수라고 불리는 Null 포인터 참조 에러의 주범입니다. `None`을 반환하는 함수를 호출하고 실수로 `if result is not None:` 검사를 빼먹으면 `AttributeError: 'NoneType' object has no attribute...` 에러가 발생합니다.

Rust는 값이 있을 수도 있고 없을 수도 있는 상황을 `Option` 열거형(Enum)으로 감싸서 표현합니다.

* **The Rust Implementation (`Option`)**
  ```rust
  // 표준 라이브러리에 정의된 Option Enum
  // enum Option<T> {
  //     Some(T), // 값이 있을 때
  //     None,    // 값이 없을 때
  // }

  fn find_user(id: i32) -> Option<String> {
      if id == 1 {
          Some("rustacean".to_string())
      } else {
          None
      }
  }

  fn main() {
      let user_opt = find_user(2);

      // 컴파일러는 user_opt 안의 String 값을 곧바로 쓰게 내버려두지 않습니다.
      // 반드시 패턴 매칭을 통해 None일 경우를 처리하도록 강제합니다.
      match user_opt {
          Some(name) => println!("Found: {}", name),
          None => println!("User not found"),
      }
  }
  ```
  이처럼 Rust는 컴파일 타임에 Null 검사를 강제하여 런타임 Null 에러를 원천 차단합니다.

---

## 12.2 `try-except`의 대체: `Result<T, E>`

실패할 가능성이 있는 연산(예: 파일 I/O, 네트워크 통신, 파싱)은 `Result` 열거형을 반환합니다.

* **The Rust Implementation (`Result`)**
  ```rust
  // 표준 라이브러리에 정의된 Result Enum
  // enum Result<T, E> {
  //     Ok(T),  // 성공했을 때의 데이터
  //     Err(E), // 실패했을 때의 에러 정보
  // }

  fn divide(a: f64, b: f64) -> Result<f64, String> {
      if b == 0.0 {
          Err("Cannot divide by zero".to_string())
      } else {
          Ok(a / b)
      }
  }

  fn main() {
      let result = divide(10.0, 0.0);

      // 성공/실패 여부를 반드시 컴파일 타임에 처리해야 함
      match result {
          Ok(val) => println!("Result: {}", val),
          Err(err) => println!("Error: {}", err),
      }
  }
  ```

---

## 12.3 모나드(Monad)적 접근: 지루한 `match`를 우아하게 체이닝하기

매번 `match` 구문을 작성하는 것은 파이썬의 `try-except`보다 코드가 훨씬 길어 보입니다. 다행히 `Option`과 `Result`는 내부에 수많은 유틸리티 메서드를 가지고 있어, 함수형 프로그래밍의 모나드 패턴처럼 안전하고 우아하게 에러를 처리할 수 있습니다.

```rust
fn get_file_extension(filename: &str) -> Option<&str> {
    // 1. split('.')으로 이터레이터를 만듭니다.
    // 2. last()는 마지막 요소를 Option으로 반환합니다. (없을 수도 있으니까)
    filename.split('.').last()
}

fn main() {
    let ext = get_file_extension("archive.tar.gz");

    // 패턴 매칭 대신 unwrap_or를 써서, 값이 없으면 기본값을 할당합니다.
    let final_ext = ext.unwrap_or("unknown");
    println!("Extension: {}", final_ext); // "gz"

    // map()을 쓰면 Some 안의 값을 변형할 수 있습니다. (None이면 아무 일도 안 함)
    let uppercase_ext = get_file_extension("document.pdf")
        .map(|ext| ext.to_uppercase()); // Some("PDF")
}
```

파이썬 개발자가 처음 Rust를 배우면 `unwrap()` 메서드를 남발하곤 합니다. `unwrap()`은 "이건 무조건 성공하는 코드니까, 만약 None이나 Err가 들어오면 프로그램을 그냥 죽여버려(Panic)"라는 뜻입니다. 프로토타이핑할 때는 유용하지만, 상용 코드에서는 절대 쓰지 말아야 할 안티 패턴입니다. 

그렇다면 여러 단계를 거쳐 에러가 발생할 때, 파이썬처럼 상위 호출자에게 에러를 훅 던져버리고(전파) 싶다면 매번 `match`로 감싸서 `return Err()`를 해야 할까요? 다음 챕터에서 이 문제를 완벽하게 해결해 주는 마법의 연산자 `?`에 대해 알아보겠습니다.
