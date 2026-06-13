# Chapter 12. `try-except`는 없다: `Option`과 `Result`

새벽에 PagerDuty 알람을 받고 일어나, 스택 트레이스(Stack Trace) 맨 아래에서 `KeyError: 'user_id'` 한 줄을 발견해 본 적 있나요? 그 키가 없을 수 있다는 사실은 어디에도 적혀 있지 않았습니다. 함수 시그니처는 `def enrich(payload: dict) -> dict:`라고 떳떳하게 선언되어 있었고, 호출하는 쪽은 그 약속을 믿었을 뿐입니다. 예외(Exception)는 타입 시스템 바깥에서 조용히 흐르다가, 가장 처리하기 곤란한 운영 환경에서 터집니다.

파이썬 개발자는 에러가 날 만한 곳에 본능적으로 `try-except`를 두릅니다. 딕셔너리 접근(`KeyError`), 파일 열기(`FileNotFoundError`), 네트워크 호출(`ConnectionError`), 정수 변환(`ValueError`) — 모두 예외를 던지고(`raise`) 받습니다(`except`). 이 모델의 매력은 분명합니다. 정상 경로(Happy Path)에 에러 처리 코드가 끼어들지 않아 본문이 깨끗하고, 깊은 콜스택에서 던진 예외가 처리자를 만날 때까지 자동으로 거슬러 올라갑니다. 파이썬다운 EAFP(Easier to Ask Forgiveness than Permission) 스타일은 이 자동 전파 위에 세워진 문화입니다.

문제는 그 자동성이 **숨겨진 제어 흐름(Hidden Control Flow)**이라는 점입니다. 어떤 함수가 어떤 예외를 던지는지는 시그니처에 나타나지 않습니다. 자바의 검사 예외(Checked Exception)처럼 `throws`를 강제하지도 않습니다. 그래서 처리해야 할 예외를 누락하기가 너무 쉽고, 누락은 컴파일러도 타입 체커도 잡지 못합니다. 결국 누락을 막는 것은 사람의 주의력과 테스트 커버리지뿐이며, Chapter 10에서 보았듯 둘 다 새벽 3시의 핫픽스 앞에서는 무력합니다.

Rust는 예외라는 개념 자체를 언어에서 삭제했습니다. `throw`도 `try`도 `catch`도 없습니다. 대신 "값이 없을 수 있다"와 "연산이 실패할 수 있다"는 사실을 **반환 값의 타입으로 인코딩**합니다. 부재(Absence)는 `Option<T>`로, 실패(Failure)는 `Result<T, E>`로. 이 두 열거형(Enum)은 Chapter 10에서 본 대수적 데이터 타입(Algebraic Data Type)의 가장 중요한 응용이며, Chapter 10의 망라성 검사(Exhaustiveness Check)가 그대로 에러 처리에 적용됩니다. 함수 시그니처가 `-> Result<Config, ParseError>`라고 말하는 순간, 호출자는 실패 가능성을 **타입으로** 통보받고, 컴파일러는 그것을 처리하지 않으면 진행을 거부합니다.

이 장에서는 파이썬의 `None`과 예외가 Rust에서 어떻게 타입 시스템 안으로 끌려 들어오는지, 그 두 열거형이 메모리에서 어떻게 배치되어 **0바이트 추가 비용**으로 부재를 표현하는지(니치 최적화, Niche Optimization), 그리고 `match`의 장황함을 어떻게 모나드(Monad)적 조합자(Combinator)로 우아하게 녹이는지를 해부합니다. `?` 연산자는 맛만 보고, 본격적인 에러 전파는 Chapter 13으로 넘깁니다.

---

## 12.1 The Python Way — 예외라는 보이지 않는 제어 흐름

파이썬에서 "실패할 수 있는 연산"의 관용구는 예외입니다. 다음은 흔한 사용자 조회 + 포트 파싱 코드입니다.

* **The Python Way (Exceptions + EAFP)**
  ```python
  from __future__ import annotations

  def find_user(users: dict[int, str], user_id: int) -> str:
      return users[user_id]  # 키 없으면 KeyError (시그니처엔 안 보임)

  def parse_port(raw: str) -> int:
      port = int(raw)  # 숫자 아니면 ValueError (역시 안 보임)
      if port < 1024:
          raise PermissionError(f"reserved port: {port}")
      return port

  def boot(config: dict[str, str], users: dict[int, str]) -> None:
      # EAFP: 일단 시도하고, 터지면 받는다
      try:
          name = find_user(users, int(config["owner_id"]))
          port = parse_port(config["port"])
          print(f"{name} 서버를 {port} 포트로 기동")
      except KeyError as e:
          print(f"설정 키 누락: {e}")
      except ValueError as e:
          print(f"숫자 변환 실패: {e}")
      # 🚨 PermissionError를 빠뜨렸다. 컴파일러도 mypy도 침묵한다.
  ```

이 코드에는 세 가지 함정이 동시에 숨어 있습니다.

**첫째, 시그니처가 거짓말을 합니다.** `find_user(...) -> str`는 "나는 항상 문자열을 준다"고 선언하지만, 실제로는 `KeyError`를 던질 수 있습니다. `parse_port(...) -> int`도 `ValueError`와 `PermissionError`를 던질 수 있지만 어디에도 적혀 있지 않습니다. 어떤 예외가 날지 알려면 **본문을 읽거나, 본문이 호출하는 함수들의 본문까지 재귀적으로 읽어야** 합니다.

**둘째, 누락이 조용합니다.** 위 `boot`은 `PermissionError`를 처리하지 않았습니다. 평소엔 멀쩡히 돌다가, 누군가 `port=80`을 설정하는 날 운영 서버가 콜스택을 끝까지 거슬러 올라가며(Call Stack Unwinding) 죽습니다. 이 버그는 코드 리뷰에서도, 단위 테스트에서도, mypy에서도 잡히지 않습니다.

**셋째, 비용이 보이지 않습니다.** 예외 처리는 "정상 경로는 공짜, 예외 경로는 비싸다"는 비대칭 모델입니다. CPython은 예외가 던져지면 예외 객체를 생성하고, 트레이스백(Traceback) 프레임을 한 단계씩 쌓으며 처리자를 찾을 때까지 스택을 되감습니다. 정상 흐름에서 이 비용은 0에 가깝지만, EAFP 패턴으로 루프 안에서 예외를 던지고 받는 코드(예: 매 반복마다 `try: d[k] except KeyError`)는 측정 가능한 오버헤드를 냅니다.

> **Under the Hood: CPython의 예외 비용은 어디서 오는가**
>
> CPython 3.11부터 정상 경로의 `try` 진입 비용은 사실상 0입니다. 바이트코드에 예외 처리 정보를 별도 테이블(이른바 "zero-cost exception table")로 빼두고, 정상 흐름에서는 그 테이블을 건드리지 않기 때문입니다. 하지만 예외가 **실제로 발생하면** 이야기가 달라집니다. 인터프리터는 (1) 예외 객체를 힙에 할당하고, (2) 현재 프레임부터 처리자를 가진 프레임까지 트레이스백 링크를 연결하며, (3) 각 프레임의 `finally`/컨텍스트 매니저 정리 코드를 실행합니다. 이 되감기(Unwinding)는 콜스택 깊이에 비례합니다. 즉 "흐름 제어 수단으로서의 예외"는 깊은 스택에서 호출될수록 비싸집니다. Rust의 `Result`는 정반대 모델입니다 — 실패 경로도 그냥 **값을 반환하는 함수 리턴**일 뿐이라, 비용이 스택 깊이와 무관하게 일정합니다. 이 비교는 12.7에서 어셈블리 수준으로 다시 다룹니다.

파이썬에서 "값의 부재"는 예외와는 또 다른 함정입니다. 흔히 `None`을 반환하지요.

```python
def first_admin(users: list[dict]) -> dict | None:
    for u in users:
        if u["role"] == "admin":
            return u
    return None  # 못 찾으면 None

admin = first_admin(load_users())
print(admin["email"])  # 🚨 None이면 TypeError: 'NoneType' object is not subscriptable
```

Tony Hoare가 "10억 달러의 실수(The Billion Dollar Mistake)"라고 부른 Null 참조의 파이썬 버전입니다. `admin["email"]` 앞에 `if admin is not None:` 검사를 빼먹는 순간, 그 `None`은 한참 떨어진 곳에서 폭발합니다. `dict | None` 타입 힌트는 mypy에게 힌트를 주지만, mypy를 CI에 강제하지 않는 한(그리고 많은 레거시 코드가 그렇지 않은 한) 이 검사는 선택사항입니다.

---

## 12.2 The Rust Mental Model — 부재와 실패를 타입으로 끌어올리다

Rust의 철학은 한 문장으로 요약됩니다: **"틀린 상태를 표현 불가능하게 만든다(Make Invalid States Unrepresentable)."** 파이썬에서 `str`은 암묵적으로 "문자열 또는 어쩌면 `None` 또는 어쩌면 예외"를 의미합니다. Rust에서 `String`은 오직 문자열입니다. 부재의 가능성이 있다면 그것을 `Option<String>`이라는 **다른 타입**으로 명시해야 하고, 실패의 가능성이 있다면 `Result<String, E>`라는 또 다른 타입으로 명시해야 합니다.

두 타입 모두 표준 라이브러리의 평범한 열거형입니다. 마법이 아니라 Chapter 10에서 본 합 타입(Sum Type)일 뿐입니다.

```rust
// 표준 라이브러리 정의 (개념적으로)
enum Option<T> {
    Some(T), // 값이 있다
    None,    // 값이 없다
}

enum Result<T, E> {
    Ok(T),  // 성공: 결과 데이터 T
    Err(E), // 실패: 에러 정보 E
}
```

핵심은 이 두 타입이 받는 **언어 차원의 대우**입니다.

1. **시그니처가 진실을 말한다.** `fn find_user(id: u32) -> Option<String>`은 "이 함수는 못 찾을 수 있다"고 타입으로 선언합니다. `fn parse_port(s: &str) -> Result<u16, ParseError>`는 "이 함수는 실패할 수 있고, 실패 정보는 `ParseError` 타입이다"라고 선언합니다. 호출자는 본문을 읽을 필요가 없습니다.

2. **처리를 강제한다.** `Option<String>`을 `String`처럼 쓸 수 없습니다. 안의 값을 꺼내려면 `None` 경우를 명시적으로 다뤄야 합니다. 이 강제는 Chapter 10의 망라성 검사와 동일한 메커니즘입니다.

3. **무시하면 경고한다.** `Result`에는 `#[must_use]` 속성이 붙어 있어, 반환된 `Result`를 그냥 버리면 컴파일러가 경고합니다. 파이썬에서 예외를 빠뜨리면 침묵하지만, Rust에서 실패를 빠뜨리면 컴파일러가 손가락을 흔듭니다.

여기서 EAFP와 LBYL(Look Before You Leap)의 대립 구도가 무너집니다. 파이썬에서는 둘 중 하나를 골라야 했습니다. Rust의 모델은 **"검사하되, 검사를 잊으면 컴파일이 안 된다(Look Before You Leap, enforced by the compiler)"**에 가깝습니다. LBYL의 안전성과 EAFP의 간결함을 조합자(Combinator)로 동시에 얻습니다 — 이것이 12.4의 주제입니다.

다음은 파이썬 예외 모델과 Rust 타입 모델의 정면 비교입니다.

| 측면 | Python (예외) | Rust (`Option`/`Result`) |
|------|---------------|--------------------------|
| 부재 표현 | `None` 반환 (`X | None`) | `Option<T>` (`Some`/`None`) |
| 실패 표현 | `raise SomeException` | `Result<T, E>` (`Ok`/`Err`) |
| 시그니처 가시성 | 안 보임 (본문 읽어야 함) | 반환 타입에 명시 |
| 처리 강제 | 없음 (누락해도 침묵) | 컴파일러가 강제 |
| 무시 시 동작 | 조용히 통과 → 런타임 폭발 | `#[must_use]` 경고 |
| 전파 방식 | 자동 스택 되감기 | 명시적 (`?` 연산자, Ch13) |
| 정상 경로 비용 | ~0 | ~0 (단순 값) |
| 실패 경로 비용 | 스택 깊이에 비례 | 분기 1회 (일정) |
| 망라성 검사 | 없음 | `match`로 컴파일 타임 강제 |
| 복구 불가 오류 | 같은 예외 메커니즘 | `panic!` (별도 채널) |

표의 마지막 행이 중요합니다. Rust는 "복구 가능한 오류(Recoverable Error)"와 "프로그래밍 버그(Bug)"를 다른 채널로 분리합니다. 전자는 `Result`로, 후자는 `panic!`으로. 파이썬은 둘 다 예외로 뭉뚱그렸기에 `try-except`가 `KeyError`(복구 가능)와 `ZeroDivisionError`(대개 버그)를 같은 문법으로 다뤘습니다. 이 분리는 12.6에서 깊이 파고듭니다.

---

## 12.3 `Option<T>` — `None`을 타입으로 길들이기

`Option<T>`은 "값이 있거나 없거나"입니다. 파이썬의 `X | None`을 컴파일러가 강제하는 버전이라고 보면 정확합니다. 12.1의 `first_admin`을 Rust로 옮겨 봅시다.

```rust
#[derive(Debug)]
struct User {
    email: String,
    role: String,
}

fn first_admin(users: &[User]) -> Option<&User> {
    for u in users {
        if u.role == "admin" {
            return Some(u);
        }
    }
    None
}

fn main() {
    let users = vec![
        User { email: "a@x.com".into(), role: "member".into() },
        User { email: "b@x.com".into(), role: "admin".into() },
    ];

    // 반드시 None 경우를 다뤄야 컴파일된다
    match first_admin(&users) {
        Some(admin) => println!("관리자: {}", admin.email),
        None => println!("관리자 없음"),
    }
}
```

파이썬의 `admin["email"]`에서 발생했던 `TypeError`가 Rust에서는 **구조적으로 불가능**합니다. `first_admin`이 `Option<&User>`를 반환하므로, `admin.email`에 곧장 접근하려 하면 컴파일러가 막습니다. 다음 코드를 보세요.

```rust
fn greet(name: &str) {
    println!("안녕하세요, {}님", name);
}

fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("rustacean".to_string()) } else { None }
}

fn main() {
    let user = find_user(1);
    // 🚨 컴파일 에러: Option<String>을 &str로 넘기려 함
    greet(&user);
}
```

`rustc`는 다음과 같이 거부합니다.

```text
error[E0308]: mismatched types
  --> src/main.rs:12:11
   |
12 |     greet(&user);
   |     ----- ^^^^^ expected `&str`, found `&Option<String>`
   |     |
   |     arguments to this function are incorrect
   |
   = note: expected reference `&str`
              found reference `&Option<String>`
note: function defined here
  --> src/main.rs:5:4
   |
 5 | fn greet(name: &str) {
   |    ^^^^^ ----------
```

`error[E0308]: mismatched types`는 Rust에서 가장 자주 만나는 에러입니다. 메시지의 핵심은 `expected &str, found &Option<String>` — "너는 문자열을 줘야 하는데, 부재 가능성이 포장된 것을 줬다"는 뜻입니다. 파이썬이라면 이 코드는 멀쩡히 컴파일되어 운영에서 `None`을 만났을 때 폭발했겠지만, Rust는 그 순간을 **개발자의 화면**으로 끌어옵니다. 컴파일러가 가장 친절한 코드 리뷰어인 셈입니다.

값을 꺼내려면 부재를 다뤄야 합니다. 가장 기본은 `match`이고, 단일 variant에만 관심 있다면 `if let`이 간결합니다.

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("rustacean".to_string()) } else { None }
}

fn main() {
    // if let: Some일 때만 처리, None은 무시
    if let Some(name) = find_user(1) {
        println!("찾음: {}", name);
    }

    // let-else: 정상 경로를 평탄하게(early return)
    let Some(name) = find_user(2) else {
        println!("사용자 없음, 종료");
        return;
    };
    // 여기서부터 name은 String (Option이 벗겨짐)
    println!("이름 길이: {}", name.len());
}
```

`let-else`(Rust 1.65+)는 파이썬 개발자에게 특히 유용합니다. 파이썬의 `if x is None: return` 가드 패턴을 그대로 옮기되, **`else` 블록은 반드시 발산(diverge)**해야 합니다 — `return`, `break`, `continue`, `panic!` 중 하나로 함수/루프를 떠나야 합니다. 그 덕에 `else` 아래의 코드는 `name`이 확실히 존재한다는 사실을 타입으로 보장받습니다. "가드를 통과했으면 값은 있다"는 약속을 컴파일러가 강제하는 것이죠.

> **Under the Hood: `Option<T>`의 메모리와 니치 최적화(Niche Optimization)**
>
> 순진하게 생각하면 `Option<T>`은 "어느 variant인지 알려주는 태그(tag) 1바이트 + 값 T"라서 `T`보다 항상 커야 할 것 같습니다. 실제로 `Option<i32>`는 8바이트입니다 (`i32` 4바이트 + 태그 + 정렬 패딩). 그런데 흥미로운 경우가 있습니다.
>
> ```rust
> use std::mem::size_of;
> fn main() {
>     println!("{}", size_of::<Box<i32>>());          // 8
>     println!("{}", size_of::<Option<Box<i32>>>());   // 8 (똑같다!)
>     println!("{}", size_of::<&i32>());               // 8
>     println!("{}", size_of::<Option<&i32>>());       // 8 (똑같다!)
>     println!("{}", size_of::<bool>());               // 1
>     println!("{}", size_of::<Option<bool>>());       // 1 (똑같다!)
> }
> ```
>
> `Option<Box<i32>>`이 `Box<i32>`와 **같은 8바이트**입니다. 어떻게? `Box`(힙 포인터)와 `&`(참조)는 절대 널(0x0)이 될 수 없다는 불변식이 있습니다. 사용하지 않는 이 비트 패턴(0x0)을 "니치(niche, 틈새)"라고 부르고, Rust는 `None`을 바로 이 0x0으로 인코딩합니다. 별도 태그 바이트가 필요 없습니다. `bool`도 마찬가지로 0과 1만 쓰므로, 2 이후의 값(예: 2)을 `None`으로 씁니다. 그래서 파이썬에서 `Optional[SomeObject]`이 객체 참조에 추가 오버헤드를 전혀 주지 않는 것처럼, Rust의 `Option<&T>`, `Option<Box<T>>`, `Option<NonZeroU32>` 등도 **0바이트 추가 비용**입니다. "널 가능성을 타입으로 표현하는데 메모리를 더 쓰지 않는다" — 이것이 무비용 추상화(Zero-cost Abstraction)의 교과서적 사례입니다. 반면 `Option<i32>`처럼 모든 비트 패턴이 유효한 타입은 니치가 없어 태그를 위한 공간을 따로 씁니다.

`Option`은 풍부한 메서드를 제공해 굳이 `match`를 쓰지 않아도 됩니다. 특히 소유권(Chapter 4)과 얽힌 두 메서드가 실무에서 자주 쓰입니다.

```rust
fn main() {
    // unwrap_or: None이면 기본값
    let port: Option<u16> = None;
    let p = port.unwrap_or(8080);
    println!("port = {}", p); // 8080

    // filter: 조건을 통과한 Some만 유지
    let even = Some(7).filter(|&n| n % 2 == 0);
    println!("{:?}", even); // None

    // take / replace: 소유권을 옮기며 Option을 비우거나 교체
    let mut slot: Option<u32> = Some(10);
    let taken = slot.take();           // slot은 이제 None, taken은 Some(10)
    println!("{:?} {:?}", taken, slot); // Some(10) None
    let prev = slot.replace(99);       // slot은 Some(99), prev는 None
    println!("{:?} {:?}", prev, slot);  // None Some(99)
}
```

`take()`는 특히 중요합니다. Rust는 `&mut self` 메서드 안에서 필드를 통째로 꺼내(move) 가는 것을 소유권 규칙상 막는데, `self.field.take()`는 그 자리에 `None`을 남기고 값만 꺼내오므로 빌림 검사기(Borrow Checker)를 만족시킵니다. 연결 리스트 노드 교체나 상태 전이 구현에서 단골로 등장하는 관용구입니다 (Chapter 20에서 다시 만납니다).

---

## 12.4 `Result<T, E>` — 실패를 시그니처에 적다

실패할 수 있는 연산은 `Result<T, E>`를 반환합니다. 파일 I/O, 파싱, 네트워크, 검증 — 12.1의 `parse_port`를 옮겨 봅시다.

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    if s.trim().is_empty() {
        return Err("빈 문자열은 포트가 될 수 없습니다".to_string());
    }
    match s.trim().parse::<u16>() {
        Ok(p) if p >= 1024 => Ok(p),
        Ok(p) => Err(format!("포트 {}는 예약 영역입니다", p)),
        Err(e) => Err(format!("'{}'는 유효한 포트가 아닙니다: {}", s, e)),
    }
}

fn main() {
    for input in ["8080", "80", "abc", ""] {
        match parse_port(input) {
            Ok(p) => println!("'{}' -> 포트 {}", input, p),
            Err(e) => println!("'{}' -> 에러: {}", input, e),
        }
    }
}
```

출력은 다음과 같습니다.

```text
'8080' -> 포트 8080
'80' -> 에러: 포트 80는 예약 영역입니다
'abc' -> 에러: 'abc'는 유효한 포트가 아닙니다: invalid digit found in string
'' -> 에러: 빈 문자열은 포트가 될 수 없습니다
```

12.1의 파이썬 버전이 `PermissionError`를 누락해도 침묵했던 것을 기억하나요? Rust에서 `parse_port`의 호출자는 `Result`를 받으므로, 성공과 모든 실패 경우를 다루지 않으면 컴파일러가 진행을 막거나 최소한 경고합니다. 그리고 `Result`를 **아예 무시하면** — 이것이 파이썬과 가장 다른 지점입니다 — 다음과 같은 일이 벌어집니다.

```rust
use std::fs::File;

fn main() {
    File::open("config.toml"); // 반환된 Result를 그냥 버림
    println!("계속 진행");
}
```

`rustc`의 반응:

```text
warning: unused `Result` that must be used
 --> src/main.rs:4:5
  |
4 |     File::open("config.toml");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
help: use `let _ = ...` to ignore the resulting value
  |
4 |     let _ = File::open("config.toml");
  |     +++++++
```

`Result`에는 `#[must_use]` 속성이 붙어 있어, 결과를 사용하지 않으면 "이 `Result`는 `Err`일 수 있으니 처리해야 한다"고 경고합니다. 정말로 무시하고 싶다면 `let _ = ...`로 **의도를 명시**해야 합니다. 파이썬에서 예외를 처리하지 않으면 침묵하지만, Rust에서 실패를 처리하지 않으면 컴파일러가 먼저 말을 겁니다. clippy를 `-D warnings`로 CI에 걸어두면 이 경고는 곧 빌드 실패가 되어, "실패 처리 누락"이라는 버그 카테고리가 머지(merge) 단계에서 차단됩니다.

> **Under the Hood: `Result<T, E>`의 메모리 레이아웃**
>
> `Result<T, E>`는 합 타입이라, 크기는 대략 `max(size_of::<T>(), size_of::<E>())` + 판별자(discriminant) + 정렬 패딩입니다. 몇 가지 실측값(64비트):
>
> ```rust
> use std::mem::size_of;
> fn main() {
>     println!("{}", size_of::<Result<u32, u32>>()); // 8  (4바이트 페이로드 + 태그 + 패딩)
>     println!("{}", size_of::<Result<(), ()>>());    // 1  (페이로드 0, 태그만)
>     println!("{}", size_of::<Result<u64, ()>>());   // 16 (8바이트 정렬 때문에 태그가 8 차지)
>     println!("{}", size_of::<Result<String, std::io::Error>>()); // 24
> }
> ```
>
> `Option`처럼 `Result`도 니치가 있으면 태그를 생략합니다. 예컨대 `Result<NonZeroU32, ()>`나 `T`/`E` 중 하나가 니치를 가진 경우 최적화가 적용됩니다. 실무적 함의: 에러 타입 `E`를 작게 유지하는 것이 좋습니다. `E`가 거대한 구조체(예: 12개 필드)면 모든 `Ok` 값도 그 크기만큼의 공간을 차지하게 되어 정상 경로의 캐시 효율이 떨어집니다. 그래서 Chapter 13에서 보겠지만, 큰 에러 타입은 `Box<dyn Error>`나 `Box<MyError>`로 박싱(boxing)해 `Result`를 8바이트 포인터 크기로 유지하는 패턴이 흔합니다. 핵심은 — `Result`는 마법의 객체가 아니라 **스택에 그대로 놓이는 평범한 값**이고, 그 크기를 우리가 통제할 수 있다는 것입니다.

`Result`도 `Option`처럼 풍부한 메서드를 갖습니다. 자주 쓰는 것들:

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    s.trim().parse::<u16>().map_err(|e| e.to_string())
}

fn main() {
    // unwrap_or_else: Err면 클로저로 기본값 계산 (지연 평가)
    let p = parse_port("xyz").unwrap_or_else(|_| 8080);
    println!("{}", p); // 8080

    // ok(): Result<T, E> -> Option<T> (에러를 버리고 부재로)
    let maybe: Option<u16> = parse_port("3000").ok();
    println!("{:?}", maybe); // Some(3000)

    // is_ok / is_err: 불리언 검사
    println!("{}", parse_port("80").is_ok()); // true (80도 u16으로 파싱은 됨)

    // unwrap_or_default: Err면 타입의 기본값 (u16 -> 0)
    let d = parse_port("nope").unwrap_or_default();
    println!("{}", d); // 0
}
```

`unwrap_or`와 `unwrap_or_else`의 차이는 실무에서 중요합니다. `unwrap_or(compute_expensive_default())`는 성공/실패와 무관하게 **항상** 인자를 평가하지만, `unwrap_or_else(|| compute_expensive_default())`는 `Err`일 때만 클로저를 실행합니다. 기본값 계산이 비싸다면(DB 조회, 파일 읽기) 반드시 `_else` 변형을 쓰세요. 파이썬의 `dict.get(k, default)` vs `dict.get(k) or compute()`의 차이와 같은 맥락입니다.

---

## 12.5 모나드적 조합자 — `match`의 장황함을 녹이다

매번 `match`를 쓰는 것은 파이썬의 `try-except`보다 길어 보입니다. 하지만 `Option`과 `Result`는 함수형 프로그래밍의 모나드(Monad) 패턴처럼 안전하게 **체이닝**할 수 있는 조합자를 제공합니다. 파이썬에서 "성공하면 변형하고, 실패하면 그대로 흘려보낸다"를 `try-except`로 짜면 들여쓰기가 깊어지지만, Rust 조합자는 그것을 한 줄의 파이프라인으로 만듭니다.

핵심 조합자를 두 그룹으로 나눠 봅시다.

**값을 변형하는 조합자 (`map` 계열)**

```rust
fn main() {
    // map: Some/Ok 안의 값을 변형, None/Err은 그대로 통과
    let upper = Some("pdf").map(|ext| ext.to_uppercase());
    println!("{:?}", upper); // Some("PDF")

    let nope: Option<&str> = None;
    println!("{:?}", nope.map(|s| s.to_uppercase())); // None (클로저 미실행)

    // map_err: Err 안의 에러만 변형 (에러 타입 변환에 필수, Ch13에서 핵심)
    let r: Result<i32, String> = "x".parse::<i32>().map_err(|e| format!("파싱 실패: {e}"));
    println!("{:?}", r); // Err("파싱 실패: invalid digit found in string")
}
```

**연산을 연결하는 조합자 (`and_then` 계열)**

`map`은 "값을 값으로" 바꾸지만, `and_then`은 "값을 또 다른 `Option`/`Result`로" 바꿉니다 (모나드의 bind, `>>=`). 즉, 또 실패할 수 있는 연산을 이어 붙입니다.

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    s.trim()
        .parse::<u16>()
        .map_err(|e| format!("'{}'는 숫자가 아닙니다: {}", s, e)) // 에러 타입 통일
        .and_then(|p| {                                          // 또 실패할 수 있는 검증 연결
            if p >= 1024 {
                Ok(p)
            } else {
                Err(format!("포트 {}는 예약 영역입니다", p))
            }
        })
}

fn main() {
    println!("{:?}", parse_port("8080")); // Ok(8080)
    println!("{:?}", parse_port("80"));   // Err("포트 80는 예약 영역입니다")
    println!("{:?}", parse_port("abc"));  // Err("'abc'는 숫자가 아닙니다: invalid digit found in string")
}
```

`parse()`가 성공하면(`Ok(p)`) `and_then`의 클로저가 실행되어 범위 검증을 수행하고, 실패하면(`Err`) 클로저는 건너뛰고 에러가 그대로 흘러갑니다. 이것이 정확히 파이썬에서 `try` 블록 안에 두 연산을 나열했을 때 첫 연산이 실패하면 둘째가 실행되지 않는 동작과 같습니다 — 다만 Rust는 들여쓰기 없이 평탄한 체인으로 표현하고, 에러 타입이 컴파일 타임에 추적됩니다.

**`Option`과 `Result` 사이를 오가는 다리**

실무에서 가장 자주 필요한 변환은 "부재를 실패로" 바꾸는 `ok_or`/`ok_or_else`입니다. `HashMap::get`은 `Option`을 반환하지만, 그 키가 **필수**라면 부재는 곧 에러이기 때문입니다.

```rust
use std::collections::HashMap;

fn get_required<'a>(map: &'a HashMap<String, String>, key: &str) -> Result<&'a String, String> {
    map.get(key)
        .ok_or_else(|| format!("필수 키 '{}'가 없습니다", key)) // Option<&V> -> Result<&V, String>
}

fn main() {
    let mut cfg = HashMap::new();
    cfg.insert("host".to_string(), "0.0.0.0".to_string());
    println!("{:?}", get_required(&cfg, "host")); // Ok("0.0.0.0")
    println!("{:?}", get_required(&cfg, "port")); // Err("필수 키 'port'가 없습니다")
}
```

**대체 경로를 잇는 조합자 (`or` / `or_else` 계열)**

`and_then`이 "성공하면 다음을 시도"라면, `or_else`는 "실패하면 다음을 시도"입니다. 설정값을 환경 변수 → 설정 파일 → 기본값 순으로 찾는 폴백(fallback) 체인이 대표적입니다. 파이썬의 `os.environ.get("X") or read_file() or "default"`를 타입 안전하게 옮긴 모습입니다.

```rust
fn from_env() -> Option<String> { std::env::var("APP_PORT").ok() }
fn from_file() -> Option<String> { None } // 예시상 파일엔 없다고 가정

fn main() {
    // or / or_else: 앞이 None이면 대체 Option을 시도 (지연 평가 버전은 _else)
    let port = from_env()
        .or_else(from_file)        // None이면 파일에서
        .or(Some("8080".into()));  // 그래도 None이면 상수 기본값
    println!("{:?}", port);        // Some("8080")

    // Result::or_else: Err면 복구를 시도 (다른 소스로 재파싱 등)
    let parsed: Result<i32, _> = "abc".parse::<i32>().or_else(|_| "0".parse::<i32>());
    println!("{:?}", parsed);      // Ok(0)
}
```

`or`는 인자를 항상 평가하고 `or_else`는 앞이 `None`/`Err`일 때만 클로저를 실행한다는 점은 `unwrap_or` 계열과 동일한 지연 평가 규칙입니다. 폴백 소스가 파일 읽기처럼 비싸면 반드시 `or_else`를 쓰세요.

**컬렉션 단위의 실패 처리 — `collect`의 마법**

실무에서 가장 강력한 패턴 하나를 짚고 넘어갑시다. 파이썬에서 `[int(s) for s in raw]`는 한 항목이라도 변환에 실패하면 `ValueError`로 전체가 중단됩니다. Rust는 이 "전부 성공 또는 첫 실패" 의미론을 `collect`의 타입 추론만으로 표현합니다.

```rust
fn main() {
    // Vec<Result<T, E>>를 Result<Vec<T>, E>로 뒤집어 수집
    // 하나라도 Err면 전체가 그 첫 Err, 모두 Ok면 Vec<T>
    let all: Result<Vec<i32>, _> =
        ["1", "2", "3"].iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", all); // Ok([1, 2, 3])

    let bad: Result<Vec<i32>, _> =
        ["1", "x", "3"].iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", bad); // Err(ParseIntError { kind: InvalidDigit })

    // 반대로, 실패한 항목은 버리고 성공만 모으려면 filter_map + ok()
    let valid: Vec<i32> =
        ["1", "x", "3"].iter().filter_map(|s| s.parse::<i32>().ok()).collect();
    println!("{:?}", valid); // [1, 3]
}
```

같은 `collect`가 반환 타입 주석(`Result<Vec<i32>, _>` vs `Vec<i32>`)만 바꿔도 완전히 다른 정책을 구현합니다 — 이것이 Chapter 15에서 본격적으로 다룰 이터레이터(Iterator)와 `Result`/`Option`의 결합입니다. 더 정교한 분할(성공/실패를 동시에 두 컬렉션으로 나누기)이 필요하면 `itertools` 크레이트의 `partition_result` 같은 어댑터가 유용하고, 에러 누적(모든 실패를 모아 보고)이 필요하면 Chapter 13에서 만날 검증 패턴을 씁니다.

조합자 선택 기준을 정리하면 — **단일 변형이면 `map`, 또 실패할 수 있는 연산을 잇는다면 `and_then`, 대체 경로면 `or_else`, 부재를 실패로 승격하려면 `ok_or_else`, 실패를 부재로 강등하려면 `ok()`.** 그리고 `match`와 조합자, `if let`, `let-else` 중 무엇을 쓸지는 가독성으로 결정합니다.

- **`match`**: 모든 variant에서 서로 다른, 비자명한 처리를 할 때. 망라성이 필요할 때.
- **`if let`**: 한 variant만 관심 있고 나머지는 무시하거나 단순할 때.
- **`let-else`**: 부재/실패 시 early-return하고, 이후 정상 경로를 평탄하게 이어갈 때(가드 패턴).
- **조합자 체인**: 값을 변형·연결하는 파이프라인이고, 중간 처리가 없을 때.

세 줄짜리 `match`를 한 줄짜리 `unwrap_or`로 바꿀 수 있다면 바꾸되, 다섯 개의 `and_then`을 억지로 한 줄에 욱여넣어 읽을 수 없게 만들지는 마세요. clippy는 `match x { Some(v) => v, None => default }`를 `x.unwrap_or(default)`로 바꾸라고 친절히 제안합니다(`clippy::manual_unwrap_or`). 가독성이 곧 정답입니다.

---

## 12.6 `unwrap`, `expect`, 그리고 `panic!` — 복구 가능 오류와 버그의 경계

파이썬을 떠나 Rust에 온 개발자의 첫 번째 나쁜 습관은 `unwrap()` 남발입니다. `unwrap()`은 "이건 무조건 성공하니까, 만에 하나 `None`/`Err`이면 프로그램을 죽여라(panic!)"라는 뜻입니다.

```rust
fn main() {
    let maybe: Option<i32> = None;
    let value = maybe.unwrap(); // 🚨 panic!
    println!("{}", value);
}
```

실행하면:

```text
thread 'main' panicked at src/main.rs:3:23:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`expect("...")`는 같은 패닉을 일으키되 **이유를 메시지로** 남깁니다. `unwrap()`보다 항상 낫습니다. 디버깅할 때 "왜 여기서 죽었는가"가 메시지에 적혀 있기 때문입니다.

```rust
fn main() {
    let config = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL 환경 변수가 설정되어야 합니다"); // 이유 명시
    println!("{}", config);
}
```

그렇다면 `unwrap`/`expect`/`panic!`은 언제 정당한가요? 여기서 Rust의 핵심 철학이 드러납니다: **복구 가능한 오류(Result로)와 프로그래밍 버그(panic으로)를 분리하라.**

- **`Result`/`Option`을 쓸 곳 (복구 가능):** 사용자 입력 검증, 파일 부재, 네트워크 타임아웃, 설정 파싱 실패. 이것들은 "예상 가능한 실패"이며, 프로그램은 이를 우아하게 처리하고 계속 살아남아야 합니다.
- **`panic!`을 쓸 곳 (프로그래밍 버그/불변식 위반):**
  - **불변식이 깨진 경우**: 코드 로직상 절대 `None`일 수 없는데 `None`이면, 그것은 데이터 오류가 아니라 **코드의 버그**입니다. `expect("이미 검증했으므로 Some이 보장됨")`이 정당합니다.
  - **테스트 코드와 프로토타입**: 빠른 실패가 미덕입니다.
  - **`main`이나 초기화 단계**: 환경 변수 누락처럼 "이게 없으면 애초에 실행될 수 없는" 설정은 `expect`로 일찍 죽는 게 낫습니다.
  - **회복 불가능한 상태**: 메모리 깨짐, 도달 불가능한 코드(`unreachable!()`).

핵심 직관: **`Result`는 "이 일은 실패할 수 있다, 호출자가 결정하라"고 말하고, `panic!`은 "이건 일어나서는 안 되는 일이다, 누군가의 코드가 틀렸다"고 말합니다.** 라이브러리를 만든다면 거의 항상 `Result`를 반환해 호출자에게 선택권을 주고, 애플리케이션의 최상위(예: `main`)에서만 "여기까지 올라온 에러는 회복 불가능하니 죽자"고 `panic!` 또는 프로세스 종료를 결정합니다.

> **Under the Hood: `panic!`의 정체 — 되감기(Unwind) vs 중단(Abort)**
>
> `panic!`이 발생하면 기본적으로 Rust는 **스택 되감기(Stack Unwinding)**를 수행합니다. 현재 스레드의 스택을 거슬러 올라가며 각 스택 프레임의 지역 변수들에 대해 `Drop`(소멸자, Chapter 4의 RAII)을 호출해 자원을 정리한 뒤, 스레드를 종료합니다. 이것은 C++의 예외 되감기와 동일한 메커니즘이고, 파이썬 예외의 트레이스백 되감기와도 개념적으로 같습니다. 이 되감기를 지원하려면 컴파일러가 **되감기 테이블(landing pad)**을 바이너리에 심어야 하므로 약간의 바이너리 크기 비용이 듭니다.
>
> `Cargo.toml`에서 `panic = "abort"`로 설정하면 되감기를 생략하고 즉시 프로세스를 죽입니다. 소멸자도 호출되지 않습니다. 이 모드는 (1) 되감기 테이블이 사라져 바이너리가 작아지고, (2) 임베디드/커널처럼 되감기를 감당할 수 없는 환경에 적합합니다.
>
> ```toml
> [profile.release]
> panic = "abort"
> ```
>
> 결정적으로 중요한 점: **`panic!`은 `Result`와 완전히 다른 비용 모델을 가집니다.** `Result`를 반환하고 `match`로 분기하는 것은 단순한 정수 비교와 조건 점프 — CPU가 가장 잘하는 일 — 이고, 정상 경로에서는 분기 예측(Branch Prediction)이 거의 항상 맞아 사실상 공짜입니다. 반면 `panic!`의 되감기는 비싸고, 그래서 절대 일상적 흐름 제어에 쓰면 안 됩니다. 이것이 파이썬과 갈리는 지점입니다: 파이썬은 `StopIteration`처럼 예외를 정상 흐름 제어에 쓰지만, Rust에서 `panic!`은 오직 "버그가 일어났다"는 비상 탈출구입니다.

**비용을 수치로 — EAFP가 비싼 이유.** 이 비대칭은 파이썬 안에서도 측정됩니다. 키의 절반이 없는 딕셔너리에서, "일단 접근하고 `KeyError`로 받는" EAFP 방식과 "`in`으로 미리 검사하는" LBYL 방식을 비교해 봅시다 (CPython 3.13, 1000키 × 1000회 반복).

```python
# EAFP: 미스마다 예외 객체 생성 + 되감기
try:
    hits += d[k]
except KeyError:
    pass
# vs LBYL: 멤버십 검사 한 번
if k in d:
    hits += d[k]
```

```text
EAFP (예외로 미스 처리): 0.054s
LBYL (멤버십 검사):      0.027s
→ 예외 경로가 약 2배 느림 (미스율 50% 기준)
```

미스율이 높을수록 격차는 벌어집니다 — 예외 객체 할당과 프레임 되감기가 매 미스마다 반복되기 때문입니다. Rust의 `match opt { Some(v) => ..., None => ... }`이나 `if let`은 이런 할당·되감기가 전혀 없는 순수 분기라, 미스율과 무관하게 비용이 일정합니다. "에러 처리를 값으로 한다"는 설계의 성능적 함의가 바로 이것입니다: **예외는 정상 경로를 공짜로 만드는 대신 실패 경로를 비싸게 만들지만, `Result`는 양쪽 모두를 평탄한 분기로 만듭니다.**

---

## 12.7 실전 Hands-on — 설정 파서를 `Result`로 재설계하기

이제 배운 것을 모아 현실적인 미니 프로젝트를 만들어 봅시다. 환경 변수나 설정 파일에서 온 `HashMap<String, String>`을 검증된 `ServerConfig`로 변환하는 파서입니다. 12.1의 파이썬 `boot()`이 가졌던 모든 함정 — 누락된 키, 숫자 변환 실패, 예약 포트 — 을 타입으로 다룹니다.

```rust
use std::collections::HashMap;
use std::num::ParseIntError;

#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
}

/// 포트 파싱 + 검증: 두 단계의 실패를 and_then으로 연결
fn parse_port(s: &str) -> Result<u16, String> {
    s.trim()
        .parse::<u16>()
        .map_err(|e| format!("'{}'는 유효한 포트가 아닙니다: {}", s, e))
        .and_then(|p| {
            if p >= 1024 {
                Ok(p)
            } else {
                Err(format!("포트 {}는 예약 영역(< 1024)입니다", p))
            }
        })
}

fn parse_config(raw: &HashMap<String, String>) -> Result<ServerConfig, String> {
    // host: 선택값. 없으면 기본값 (unwrap_or_else로 지연 생성)
    let host = raw
        .get("host")
        .cloned()
        .unwrap_or_else(|| "127.0.0.1".to_string());

    // port: 필수값. 부재를 ok_or_else로 에러 승격 후, ?로 전파(맛보기)
    let port = raw
        .get("port")
        .ok_or_else(|| "필수 키 'port'가 없습니다".to_string())
        .and_then(|s| parse_port(s))?;

    // workers: 선택값이지만, 있으면 반드시 숫자여야 함.
    // Option<Result<_,_>>를 transpose로 Result<Option<_>,_>로 뒤집는 것이 핵심.
    let workers = raw
        .get("workers")
        .map(|s| s.parse::<usize>())          // Option<Result<usize, ParseIntError>>
        .transpose()                          // Result<Option<usize>, ParseIntError>
        .map_err(|e: ParseIntError| format!("workers 파싱 실패: {}", e))?
        .unwrap_or(4);                        // Option<usize> -> usize (기본 4)

    Ok(ServerConfig { host, port, workers })
}

fn main() {
    // 케이스 1: 정상
    let mut raw = HashMap::new();
    raw.insert("port".to_string(), "8080".to_string());
    raw.insert("workers".to_string(), "8".to_string());
    println!("{:?}", parse_config(&raw));
    // Ok(ServerConfig { host: "127.0.0.1", port: 8080, workers: 8 })

    // 케이스 2: 예약 포트
    raw.insert("port".to_string(), "80".to_string());
    println!("{:?}", parse_config(&raw));
    // Err("포트 80는 예약 영역(< 1024)입니다")

    // 케이스 3: 필수 키 누락
    raw.remove("port");
    println!("{:?}", parse_config(&raw));
    // Err("필수 키 'port'가 없습니다")
}
```

이 코드에는 이번 챕터의 거의 모든 도구가 응축되어 있습니다. `unwrap_or_else`(선택값 기본 처리), `ok_or_else`(부재→실패 승격), `and_then`(연쇄 검증), `map_err`(에러 타입 통일), `transpose`(`Option`과 `Result`의 중첩 뒤집기), 그리고 `?`(전파 — Chapter 13의 예고편). 무엇보다 `parse_config`의 **시그니처 `-> Result<ServerConfig, String>` 한 줄이 "이 함수는 실패할 수 있고, 실패하면 사람이 읽을 수 있는 String 이유를 준다"는 계약 전체를 선언**합니다. 12.1의 파이썬 `boot()`이 어떤 예외를 던질지 본문을 다 읽어야 알 수 있었던 것과 대조됩니다.

여기서 `?` 연산자가 두 번 등장합니다. `.and_then(...)?`와 `.map_err(...)?`. `?`는 "`Ok`/`Some`이면 안의 값을 꺼내고, `Err`/`None`이면 그 즉시 함수에서 `return Err(...)`/`return None` 하라"는 축약입니다. 12.1의 파이썬 `try` 블록이 "예외가 나면 즉시 빠져나간다"고 했던 그 동작을 **한 글자로** 표현한 것이죠. 다만 파이썬과 달리 어디서 빠져나가는지가 `?`라는 명시적 기호로 코드에 박혀 있어 보이지 않는 제어 흐름이 아닙니다. 이 한 글자가 어떻게 `From` 트레이트를 통한 에러 타입 자동 변환과 결합해 거대한 코드베이스의 에러 전파를 우아하게 만드는지는 다음 챕터의 주인공입니다.

마지막으로 같은 결제 도메인(Chapter 10)을 확장해, `Option`(조회 실패)과 `Result`(연산 실패)가 한 타입 안에서 협력하는 모습을 봅시다.

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
enum PaymentState {
    Pending,
    Authorized { auth_code: String },
    Captured { receipt_id: u64 },
}

struct PaymentGateway {
    states: HashMap<u64, PaymentState>,
}

impl PaymentGateway {
    // 조회: 거래가 없을 수 있다 -> Option
    fn lookup(&self, tx_id: u64) -> Option<&PaymentState> {
        self.states.get(&tx_id)
    }

    // 캡처: 거래가 없거나(부재) 잘못된 상태일(실패) 수 있다 -> Result
    fn capture(&mut self, tx_id: u64, receipt_id: u64) -> Result<PaymentState, String> {
        // 부재를 실패로 승격: Option -> Result, 그리고 ?로 전파
        let state = self
            .lookup(tx_id)
            .ok_or_else(|| format!("거래 {}를 찾을 수 없습니다", tx_id))?;

        // 상태 검증은 match (망라성, Chapter 10)
        match state {
            PaymentState::Authorized { .. } => {
                let new_state = PaymentState::Captured { receipt_id };
                self.states.insert(tx_id, new_state.clone());
                Ok(new_state)
            }
            other => Err(format!("승인 상태가 아닙니다: {:?}", other)),
        }
    }
}

fn main() {
    let mut gw = PaymentGateway { states: HashMap::new() };
    gw.states.insert(1, PaymentState::Authorized { auth_code: "A123".into() });
    gw.states.insert(2, PaymentState::Pending);

    println!("{:?}", gw.capture(1, 9001)); // Ok(Captured { receipt_id: 9001 })
    println!("{:?}", gw.capture(2, 9002)); // Err("승인 상태가 아닙니다: Pending")
    println!("{:?}", gw.capture(99, 9003)); // Err("거래 99를 찾을 수 없습니다")
}
```

`lookup`은 `Option`(조회는 "없을 수 있다"), `capture`는 `Result`(연산은 "실패할 수 있다")를 반환합니다. 이 구분이 곧 API의 의미론입니다 — `Option`은 "정상적으로 없을 수 있음", `Result`는 "있어야 했는데 실패함". 두 타입이 `ok_or_else`라는 다리로 자연스럽게 연결되는 것을 다시 한번 확인하세요.

---

## 12.8 Best Practices & 안티패턴

실무 체크리스트입니다. 코드 리뷰와 clippy 설정에 그대로 옮겨 쓰세요.

**이렇게 하라**

- **부재는 `Option`, 실패는 `Result`로 구분하라.** "정상적으로 값이 없을 수 있다"(캐시 미스, 검색 결과 없음)는 `Option`. "성공해야 하는데 실패했다"(I/O 에러, 파싱 실패)는 `Result`. 의미가 API에 드러난다.
- **라이브러리는 `Result`를 반환하고 패닉하지 마라.** 호출자에게 선택권을 줘라. 패닉할지 복구할지는 애플리케이션의 결정이다.
- **`unwrap()` 대신 `expect("이유")`를 써라.** 패닉이 정당한 자리라도, 왜 안전하다고 판단했는지 메시지로 남겨라. "여기는 위에서 검증했으므로 Some"처럼.
- **기본값 계산이 비싸면 `_or_else` 변형을 써라.** `unwrap_or(expensive())`는 항상 평가, `unwrap_or_else(|| expensive())`는 필요할 때만.
- **`#[must_use]` 경고를 빌드 에러로 승격하라.** CI에 `RUSTFLAGS="-D warnings"` 또는 `cargo clippy -- -D warnings`. 실패 무시를 머지 단계에서 차단한다.
- **`match`와 조합자를 가독성으로 선택하라.** 짧은 변형은 조합자, 복잡한 분기는 `match`, 가드는 `let-else`. clippy의 `manual_unwrap_or`, `option_map_unit_fn` 제안을 따르라.
- **에러 타입 `E`를 작게 유지하라.** 큰 에러는 박싱해 `Result`를 작게. (Chapter 13의 `thiserror`/`anyhow`로 이어진다.)

**이건 피하라**

- **`unwrap()` 남발.** 프로토타입을 졸업한 코드에 `unwrap()`이 흩뿌려져 있다면, 그것은 "처리하지 않은 예외"의 Rust 버전이다.
- **`Option`을 `bool`처럼 쓰기.** `if opt.is_some() { opt.unwrap() }`은 안티패턴이다. `if let Some(v) = opt`를 써라 — clippy가 잡아준다.
- **`Result`를 `let _ =`로 무심코 버리기.** 정말 무시해도 되는지 매번 자문하라. 로그 한 줄이라도 남기는 게 보통 맞다.
- **`panic!`을 흐름 제어에 쓰기.** 검증 실패를 `panic!`으로 처리하면 호출자가 복구할 수 없다. 이것은 파이썬에서 `raise`를 너무 쉽게 쓰던 습관의 잔재다.

---

## 12.9 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: EAFP 습관 — "일단 꺼내고 보자"**

파이썬 개발자는 `d[k]`로 일단 접근하고 `KeyError`로 받는 데 익숙합니다. Rust에서 그 습관은 `unwrap()`으로 나타납니다.

```rust
// 🚨 안티패턴: 파이썬 d[k]를 그대로 옮긴 사고
let port = config.get("port").unwrap(); // 키 없으면 panic — 운영에서 죽는다

// ✅ 부재 가능성을 그대로 다룬다
let port = config.get("port").unwrap_or(&default_port);
// 또는 부재가 곧 에러라면
let port = config.get("port").ok_or("port 누락")?;
```

EAFP의 "Ask Forgiveness"는 Rust에서는 "처리를 미루다 패닉으로 죽기"가 됩니다. Rust의 문화는 LBYL이되, 컴파일러가 "Look"을 강제합니다.

**함정 2: `Option`/`Result` 중첩에 당황하기**

`HashMap`에서 꺼낸 값을 파싱하면 `Option<Result<T, E>>`처럼 중첩됩니다. 파이썬엔 이런 구조가 없어 처음엔 당황스럽습니다.

```rust
// raw.get("n")은 Option<&String>, parse는 Result<usize, _>
// .map(|s| s.parse())는 Option<Result<usize, _>>

// 🚨 중첩된 채로 unwrap().unwrap() — 두 번 죽을 수 있다
let n = raw.get("n").unwrap().parse::<usize>().unwrap();

// ✅ transpose로 평탄화하거나 ?로 흘려보낸다
let n: Option<usize> = raw.get("n").map(|s| s.parse()).transpose()?; // Result로 전파
```

`transpose()`는 `Option<Result<T, E>>`와 `Result<Option<T>, E>`를 맞바꿉니다. "값은 선택적이지만, 있다면 반드시 유효해야 한다"는 흔한 요구를 정확히 표현하는 도구입니다.

**함정 3: `None`을 `Result`로, 또는 그 반대로 헷갈리기**

"사용자를 못 찾음"이 `Option<User>`인지 `Result<User, NotFound>`인지는 API 의미론에 달렸습니다. 조회가 일상적으로 비어 있을 수 있다면 `Option`, 그 부재가 곧 처리해야 할 에러라면 `Result`. 둘을 섞어 쓰면 호출 코드가 일관성을 잃습니다. 한 모듈 안에서 규칙을 정하고 `ok_or_else`/`ok()`로 경계에서만 변환하세요.

**함정 4: `&Option<T>`를 받아 패턴 매칭에서 소유권 에러**

```rust
fn name_len(opt: &Option<String>) -> usize {
    // 🚨 컴파일 에러 가능: match opt { Some(s) => ... } 에서 s를 move하려 함
    match opt {
        Some(s) => s.len(), // 참조라서 s는 &String, .len()은 OK
        None => 0,
    }
}
```

`&Option<String>`을 `match`하면 패턴의 바인딩도 참조가 됩니다(`s: &String`). 이를 모르고 `Some(s) => consume(s)`처럼 값을 소비하려 하면 `error[E0507]: cannot move out of ...`을 만납니다. `opt.as_ref()`, `as_deref()`, 또는 `if let Some(s) = opt`(참조 바인딩)로 해결합니다. 소유권(Chapter 4)과 참조(Chapter 5)의 규칙이 `Option`/`Result`에도 그대로 적용된다는 점을 기억하세요.

---

이번 장에서 우리는 파이썬의 보이지 않는 두 함정 — Null(`None`)과 예외(Exception) — 을 Rust가 어떻게 타입 시스템 안으로 끌어들이는지 보았습니다. `Option<T>`은 부재를, `Result<T, E>`은 실패를 시그니처에 새기고, 컴파일러는 그 처리를 강제합니다. 니치 최적화 덕분에 이 안전성은 대부분 0바이트 추가 비용이며, `match`의 장황함은 `map`/`and_then`/`ok_or_else` 같은 모나드적 조합자로 평탄한 파이프라인이 됩니다. 그리고 우리는 복구 가능한 오류(`Result`)와 프로그래밍 버그(`panic!`)를 다른 채널로 분리하는 Rust의 규율을 배웠습니다.

하지만 한 가지 고통이 남았습니다. 콜스택 깊은 곳에서 발생한 에러를 상위로 전파하려면 매번 `match`로 받아 `return Err(e)`를 해야 할까요? 12.7에서 슬쩍 등장한 `?` 연산자가 바로 그 보일러플레이트를 한 글자로 지워버립니다. 다음 Chapter 13에서는 `?`의 마법을 완전히 해부하고, 서로 다른 에러 타입을 `From` 트레이트로 자동 변환하는 메커니즘, 그리고 실무 표준인 `thiserror`(라이브러리용 구조화된 에러)와 `anyhow`(애플리케이션용 동적 에러)로 파이썬의 복잡한 커스텀 Exception 계층도를 어떻게 Rust 에러 설계로 마이그레이션하는지를 다룹니다.
