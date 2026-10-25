# Chapter 13. 에러 전파와 커스텀 에러 설계

![Chapter 13. 에러 전파와 커스텀 에러 설계](../images/chapter-13-errors.jpg)

10년 차 백엔드 엔지니어라면 이런 새벽 알림을 받아본 적이 있을 겁니다. 결제 처리 API가 `500 Internal Server Error`를 토해내고 있고, Sentry에는 `KeyError: 'amount'`가 콜스택 12단계 깊은 곳에서 올라와 있습니다. 정작 그 키를 읽은 함수는 에러를 던질 수 있다는 사실을 시그니처 어디에도 표시하지 않았습니다. 그 함수를 호출한 함수도, 또 그 위의 함수도 마찬가지였습니다. 예외(Exception)는 12개 함수를 *조용히* 관통해 마침내 누구도 `except`로 받지 못한 최상단에서 프로세스를 죽였습니다.

파이썬에서 예외 전파는 공짜이자 자동입니다. 콜스택 깊은 곳에서 `raise`를 하면, 그 예외는 누군가 `except`로 잡을 때까지 호출자를 향해 알아서 거슬러 올라갑니다(스택 언와인딩, Stack Unwinding). 편리합니다. 하지만 그 편리함의 대가가 바로 위의 새벽 알림입니다. *어떤 함수가 무엇을 던지는지가 타입 시스템에 기록되지 않기* 때문에, 에러 처리는 영원히 "문서를 잘 읽었기를 바라는" 신앙의 영역에 머뭅니다.

[Chapter 12](./chapter_12.md)에서 우리는 Rust가 예외라는 개념 자체를 언어에서 삭제하고, 실패를 `Result<T, E>`라는 **반환 값의 타입**으로 명시한다는 것을 배웠습니다. 함수 시그니처만 봐도 "이 함수는 실패할 수 있고, 실패하면 `E` 타입의 에러를 준다"는 계약이 컴파일러에 의해 강제됩니다. 이것은 강력하지만, 한 가지 명백한 문제를 남깁니다. 콜스택 12단계를 거슬러 올라가는 그 우아한 자동 전파를, 매번 `match`로 풀어서 `return Err(e)`를 손으로 써야 할까요?

이번 장의 전반부는 이 질문에 답합니다. Rust의 **`?` 연산자**는 파이썬의 자동 전파와 동일한 *개발 경험*을 주면서도, 전파되는 에러 타입을 컴파일 타임에 한 줄도 빠짐없이 추적합니다. `?`가 단순한 문법 설탕이 아니라 `From` 트레이트를 통한 **에러 타입 자동 변환** 위에서 동작한다는 것, 그리고 이 한 가지 사실이 어떻게 대규모 코드베이스의 에러 아키텍처 전체를 좌우하는지를 바이트 단위로 해부합니다.

후반부에서는 실무로 넘어갑니다. 파이썬에서 `AppError`를 최상위로 두고 `DBError`, `ValidationError`, `NetworkError`를 상속시키던 그 커스텀 예외 계층을, Rust에서는 어떤 모양으로 옮겨야 하는가. 그리고 러스트 생태계의 두 표준 무기 — 라이브러리를 위한 **`thiserror`**와 애플리케이션을 위한 **`anyhow`** — 를 언제 무엇으로 골라야 하는지, 실제로 컴파일되고 실행되는 코드와 컴파일러 에러 메시지로 증명합니다.

---

## 13.1 The Python Way — 자동 전파의 청구서

먼저 우리가 지금까지 어떻게 살아왔는지 정직하게 들여다봅시다. 파이썬의 예외 전파는 두 가지 성질을 동시에 가집니다. 하나는 **암묵성(implicit)**이고 다른 하나는 **계층성(hierarchy)**입니다.

```python
from __future__ import annotations


class AppError(Exception):
    """애플리케이션 최상위 에러."""


class DBError(AppError):
    """DB 계층 에러."""


class ValidationError(AppError):
    """입력 검증 에러."""


class NetworkError(AppError):
    """외부 통신 에러."""


def fetch_user_row(user_id: int) -> dict[str, str]:
    # ... DB 호출 중 실패하면 ...
    raise DBError(f"connection refused while fetching {user_id}")


def validate(row: dict[str, str]) -> dict[str, str]:
    if "email" not in row:
        raise ValidationError("email is required")
    return row


def load_user(user_id: int) -> dict[str, str]:
    # 여기서는 try-except가 한 줄도 없습니다.
    # fetch_user_row가 던지는 DBError는 이 함수를 그냥 관통해 위로 올라갑니다.
    row = fetch_user_row(user_id)
    return validate(row)


def main() -> None:
    try:
        user = load_user(42)
        print(user)
    except AppError as e:  # 부모 하나로 모든 자식을 잡는다
        print(f"[{type(e).__name__}] {e}")
```

이 코드의 우아함은 `load_user`에 있습니다. 이 함수는 에러를 *처리하지 않습니다*. 다만 가만히 있을 뿐인데, `fetch_user_row`가 던진 `DBError`가 알아서 `main`까지 올라갑니다. 그리고 `main`은 `except AppError` 하나로 `DBError`, `ValidationError`, `NetworkError`를 전부 잡습니다. 계층(상속)이 다형성(polymorphism)을 만들어주기 때문입니다.

여기에 두 가지 청구서가 숨어 있습니다.

**청구서 1: 시그니처는 거짓말을 한다.** `def load_user(user_id: int) -> dict[str, str]`라는 시그니처는 이 함수가 성공하면 dict를 준다고만 말합니다. `DBError`나 `ValidationError`가 튀어나올 수 있다는 사실은 시그니처 어디에도 없습니다. Java의 `throws`처럼 검사 예외(checked exception)를 강제하는 장치가 파이썬에는 없습니다. 결국 호출자는 *소스를 끝까지 따라 읽거나*, 운영 중에 터지는 걸 보고 나서야 어떤 예외가 가능한지 알게 됩니다.

**청구서 2: 잡지 않으면 프로세스가 죽는다.** 자동 전파는 양날의 검입니다. 누군가 잡아주리라는 가정이 깨지면, 예외는 최상단까지 올라가 인터프리터를 멈춥니다. `except`를 한 군데라도 빠뜨리면 그 경로 전체가 잠재적 크래시입니다. 그리고 이 누락은 컴파일 타임이 아니라 **런타임에**, 그것도 보통 가장 트래픽이 몰리는 시간에 발견됩니다.

**청구서 3: 예외는 제어 흐름으로 오용된다.** 파이썬 커뮤니티에는 "허락보다 용서가 쉽다(EAFP, Easier to Ask Forgiveness than Permission)"는 관용구가 있습니다. 값이 있는지 미리 검사(`if key in d`)하지 말고 일단 시도한 뒤 예외로 받으라는 것입니다. 이 스타일은 정상 경로에서는 빠르지만, 예외가 *실제로 자주 발생하는* 경로에서는 비싸집니다. 파이썬에서 예외 객체를 생성하고 던지는 순간 트레이스백(traceback) 객체가 콜스택 프레임마다 만들어지고, 스택을 언와인딩하며 각 프레임의 `finally`와 컨텍스트 매니저(`__exit__`)를 실행합니다. 그래서 루프 안에서 예외를 제어 흐름으로 쓰면 — 가령 수백만 건의 레코드를 파싱하며 절반이 `ValueError`를 던진다면 — 예외 처리 자체가 핫스팟(hotspot)이 됩니다. 예외를 "정말 예외적인 상황"에만 쓰라는 조언은 이 비용 때문입니다. Rust의 `Result`는 정상이든 에러든 똑같이 *값을 반환할 뿐*이라 이런 비대칭 비용이 없습니다. `Err`을 만드는 것은 `Ok`를 만드는 것과 동일한 비용입니다(스택 언와인딩도, 트레이스백 캡처도 없습니다).

Rust는 이 세 청구서를 모두 거부합니다. 에러는 반드시 타입에 적히고(`Result<T, E>`의 `E`), 그 에러를 무시하면 컴파일이 되지 않습니다. 문제는 그러면서도 청구서 없는 자동 전파의 편리함을 어떻게 되찾느냐입니다.

---

## 13.2 The Rust Mental Model — 전파는 명시적이되, 보이지 않는다

파이썬의 사고방식은 "에러는 던지면 알아서 올라간다. 처리하고 싶은 곳에서만 잡아라"입니다. Rust의 사고방식은 정반대 출발점에서 시작해 같은 편의에 도달합니다. "에러는 값이다. 값은 저절로 올라가지 않는다. 하지만 *올려보내는 행위*를 한 글자로 압축해 주겠다."

그 한 글자가 `?`입니다. [Chapter 12](./chapter_12.md)에서 본 장황한 `match`를 떠올려 봅시다.

```rust
use std::fs::File;
use std::io::{self, Read};

// 손으로 푼 에러 전파: match가 두 번
fn read_username() -> Result<String, io::Error> {
    let mut f = match File::open("hello.txt") {
        Ok(file) => file,
        Err(e) => return Err(e), // 에러면 여기서 함수를 종료하고 Err 반환
    };

    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```

핵심 통찰은 이것입니다. `match` 블록이 하는 일은 늘 똑같습니다. *"`Ok`면 안의 값을 꺼내 계속 진행하고, `Err`면 즉시 이 함수를 종료하며 그 에러를 반환한다."* 늘 같은 동작이라면, 언어가 키워드 하나로 대신해줄 수 있습니다.

```rust
use std::fs::File;
use std::io::{self, Read};

// ?로 푼 에러 전파: 파이썬만큼 짧다
fn read_username_short() -> Result<String, io::Error> {
    let mut s = String::new();
    // File::open이 Err이면 즉시 io::Error를 반환하고 함수 종료.
    // Ok이면 안의 File을 꺼내 f에 바인딩.
    let mut f = File::open("hello.txt")?;
    f.read_to_string(&mut s)?;
    Ok(s)
}

// 메서드 체이닝과도 자연스럽게 결합된다
fn read_username_shorter() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```

겉보기에 `read_username_short`는 파이썬의 `load_user`와 거의 같은 줄 수입니다. 에러를 처리하지 않고 그냥 위로 흘려보냅니다. 하지만 결정적 차이가 있습니다. **파이썬은 `load_user`의 시그니처에 에러 정보가 0바이트 적혀 있지만, Rust는 `-> Result<String, io::Error>`라고 정확히 적혀 있습니다.** 호출자는 소스를 따라 읽을 필요가 없습니다. 타입이 곧 문서이고, 그 문서는 컴파일러가 영원히 최신으로 유지합니다.

> **💡 핵심: `?`는 "처리"가 아니라 "전파"다**
>
> `?`는 에러를 *처리하지 않습니다*. 에러를 호출자에게 떠넘길 뿐입니다. 파이썬에서 `try-except`를 쓰지 않고 예외가 그냥 올라가게 두는 것과 정확히 대응됩니다. 진짜 처리(로깅, 폴백, 사용자 응답 변환)는 콜스택 어딘가 — 보통 `main`이나 요청 핸들러 — 에서 `match`나 `if let`으로 한 번 합니다. 즉 Rust의 에러 흐름은 "깊은 곳에서는 `?`로 전파, 얕은 곳에서 한 번 처리"라는 파이썬과 동일한 형태를 갖되, 전파 경로의 모든 타입이 컴파일 타임에 검증됩니다.

### Under the Hood: `?`의 디슈가링(Desugaring)

`?`가 마법이 아니라는 것을 증명해 봅시다. 컴파일러는 `expr?`를 다음과 같은 코드로 **기계적으로 펼칩니다(desugar)**. 개념적으로 `?`는 `std::ops::Try` 트레이트를 통해 동작하지만, `Result`에 대해서는 다음 `match`와 의미가 동일합니다.

```rust
// 우리가 쓰는 것
let mut f = File::open("hello.txt")?;

// 컴파일러가 펼치는 것 (개념적으로)
let mut f = match File::open("hello.txt") {
    Ok(value) => value,
    Err(err) => return Err(From::from(err)),
    //                     ^^^^^^^^^^^^^^^ 여기가 핵심!
};
```

`Ok` 가지는 안의 값을 꺼내 표현식의 결과로 만들고, `Err` 가지는 즉시 `return` 합니다. 여기까지는 우리가 손으로 쓰던 `match`와 같습니다. **그런데 한 가지가 다릅니다.** 손으로 쓸 때는 `return Err(e)`였지만, `?`는 `return Err(From::from(err))`입니다. 즉 `?`는 에러를 반환하기 직전에 `From::from`(= `.into()`)을 호출해 **에러 타입을 자동으로 변환**합니다.

이 한 줄이 13.4에서 다룰 커스텀 에러 설계의 전부를 떠받치는 기둥입니다. 지금은 이것만 기억하면 됩니다. `?`로 전파되는 에러의 원래 타입 `E`가 함수의 반환 에러 타입 `F`와 다르더라도, `From<E>`가 `F`에 구현되어 있기만 하면 `?`가 알아서 `E → F` 변환을 끼워 넣어 줍니다. `From<E>`가 없으면? 컴파일이 거부됩니다. 13.3에서 그 거부 장면을 직접 보겠습니다.

`Option`에도 `?`를 쓸 수 있습니다. 이때는 `None`이 `Err`의 자리를 대신합니다.

```rust
// None이면 즉시 None을 반환하고 함수 종료
fn first_char_upper(s: &str) -> Option<char> {
    let c = s.chars().next()?; // 빈 문자열이면 여기서 None 반환
    Some(c.to_ascii_uppercase())
}
```

다만 `Result`를 반환하는 함수 안에서 `Option`에 `?`를 쓰거나 그 반대를 하면 타입이 맞지 않아 컴파일되지 않습니다. 둘을 섞어야 할 때는 `Option`을 `ok_or`/`ok_or_else`로 `Result`로 바꾸거나, `Result`를 `ok()`로 `Option`으로 바꿔 맞춰줍니다(이 어댑터들은 Chapter 12에서 다뤘습니다).

`?`가 `match`로 펼쳐진다는 사실은 한 가지 중요한 결과를 낳습니다. **`?`는 함수의 반환 타입을 본다는 것입니다.** `expr?`가 펼쳐낸 `return Err(From::from(err))`의 `Err`은 *현재 함수가 반환하기로 선언한 에러 타입*이어야 합니다. 그래서 같은 `expr?`라도 어떤 함수 안에 있느냐에 따라 `From` 변환의 목적지가 달라집니다. `-> Result<T, io::Error>` 안에서는 변환이 필요 없고(이미 `io::Error`), `-> Result<T, AppError>` 안에서는 `From<io::Error> for AppError`를 통해 변환되며, `-> Result<T, Box<dyn Error>>` 안에서는 `Box<dyn Error>`로 박싱됩니다. 똑같은 `?` 한 글자가 문맥에 따라 무비용 통과부터 힙 박싱까지 다른 일을 하는 것입니다. 이 "반환 타입 주도" 동작이 13.4에서 커스텀 에러를 설계할 때의 핵심 지렛대가 됩니다.

> **💡 무비용 추상화로서의 `?`**
>
> `?`는 [Chapter 12](./chapter_12.md)에서 본 무비용 추상화(Zero-cost Abstraction)의 또 다른 예입니다. 우리가 손으로 쓰던 `match`와 `?`는 디슈가링 후 *동일한* 기계어로 컴파일됩니다. `?`를 썼다고 런타임 오버헤드가 한 사이클도 추가되지 않습니다. 가독성은 파이썬의 자동 전파만큼 좋아지면서, 비용은 손으로 짠 `match`와 같습니다. `--release` 빌드에서 옵티마이저는 대개 이 분기조차 인라이닝하고 정리합니다.

### Under the Hood: `?`와 `match`는 정말 같은 기계어인가 — 어셈블리로 증명

"동일한 기계어로 컴파일된다"는 말은 검증 없이 믿을 주장이 아닙니다. 직접 디스어셈블해 봅시다. 다음 두 함수는 의미가 같습니다. 하나는 `?`, 하나는 손으로 푼 `match`입니다.

```rust
#[inline(never)]
#[no_mangle]
pub fn with_question(s: &str) -> Result<i64, std::num::ParseIntError> {
    let n: i64 = s.parse()?; // ? 버전
    Ok(n * 2)
}

#[inline(never)]
#[no_mangle]
pub fn with_match(s: &str) -> Result<i64, std::num::ParseIntError> {
    let n: i64 = match s.parse() { // 손으로 푼 match 버전
        Ok(v) => v,
        Err(e) => return Err(e),
    };
    Ok(n * 2)
}
```

`rustc -O --emit asm`(rustc 1.90.0, aarch64/Apple Silicon)로 두 함수의 어셈블리를 뽑아 비교하면, 두 함수가 내뱉는 **명령어의 멀티셋(multiset)이 한 개도 빠짐없이 동일**합니다. 차이는 단 둘뿐입니다. (1) 레지스터 할당 선택(`w9` vs `w10` 같은 임의 배정), (2) 의미가 같은 인접한 두 저장 명령(`strb`)의 순서. 둘 다 *실행 동작에 영향이 없는* 표기 차이입니다.

```text
# with_question 의 핵심 instruction multiset (개수까지 with_match와 동일)
$ diff <(asm_of with_question | mnemonics | sort | uniq -c) \
       <(asm_of with_match   | mnemonics | sort | uniq -c)
# (출력 없음 → 명령어 종류·개수 완전히 동일)

# with_question 본문 발췌 (aarch64)
with_question:
    cbz   x1, ...        # 길이 0이면 에러 분기
    subs  x10, x1, #1
    b.ne  ...
    ldrb  w9, [x0]       # 첫 바이트 로드 (부호/숫자 판별)
    cmp   w9, #43        # '+' ?
    b.eq  ...
    cmp   w9, #45        # '-' ?
    ...
    ret
```

`?`가 디슈가링한 `match { Ok(v)=>v, Err(e)=>return Err(From::from(e)) }`와, 우리가 손으로 쓴 `match`가 동일한 제어 흐름 그래프를 만들기 때문에, LLVM 옵티마이저는 둘을 같은 기계어로 수렴시킵니다. 여기서 에러 타입이 동일(`ParseIntError → ParseIntError`)하므로 `From::from`은 항등 변환(`From<T> for T`)이 되어 한 사이클도 추가하지 않습니다. **즉 `?`의 가독성은 100% 공짜입니다.** 파이썬에서 `try-except`로 감싸면 인터프리터가 예외 핸들러 블록을 설치하고 프레임마다 정리 루틴을 등록하는 것과는 차원이 다릅니다 — Rust에서는 추가되는 명령어가 글자 그대로 0개입니다.

---

## 13.3 컴파일러 에러로 배우는 `?`의 진실

말로 설명하는 대신, 컴파일러가 직접 가르치게 합시다. 다음은 두 종류의 에러를 한 함수에서 `?`로 전파하려는 자연스러운 시도입니다.

```rust
use std::fs::File;
use std::io::Read;

#[derive(Debug)]
struct MyError; // 내가 만든 에러. From 구현은 아직 없음.

fn read_config() -> Result<String, MyError> {
    let mut s = String::new();
    // 🚨 컴파일 에러! File::open과 read_to_string은 io::Error를 주는데,
    //    함수는 MyError를 반환하라고 되어 있다.
    File::open("config.toml")?.read_to_string(&mut s)?;
    Ok(s)
}
```

`rustc`는 이렇게 응답합니다.

```text
error[E0277]: `?` couldn't convert the error to `MyError`
 --> src/main.rs:9:30
  |
7 | fn read_config() -> Result<String, MyError> {
  |                     ----------------------- expected `MyError` because of this
8 |     let mut s = String::new();
9 |     File::open("config.toml")?.read_to_string(&mut s)?;
  |     -------------------------^ the trait `From<std::io::Error>` is not implemented for `MyError`
  |     |
  |     this can't be annotated with `?` because it has type `Result<_, std::io::Error>`
  |
note: `MyError` needs to implement `From<std::io::Error>`
 --> src/main.rs:5:1
  |
5 | struct MyError;
  | ^^^^^^^^^^^^^^
  = note: the question mark operation (`?`) implicitly performs a conversion
          on the error value using the `From` trait
```

이 메시지를 한 줄씩 해독해 봅시다. 마지막 `note`가 우리가 13.2에서 디슈가링으로 알아낸 사실을 그대로 확인해 줍니다. *"물음표 연산(`?`)은 `From` 트레이트를 사용해 에러 값에 암묵적 변환을 수행한다."* 그리고 `the trait From<std::io::Error> is not implemented for MyError`. 즉 컴파일러는 `?`를 디슈가링하면서 `From::<io::Error>::from(err)`을 호출하려 했는데, 그 변환이 정의되어 있지 않으니 거부한 것입니다.

`error[E0277]`은 "특정 트레이트 바운드(trait bound)가 충족되지 않았다"는 뜻의 에러 코드입니다. 파이썬이라면 이 코드가 아무 문제 없이 배포되어, 실제로 파일이 없는 그 순간에 `io::Error`에 대응하는 예외가 잡히지 않은 채 런타임에 터졌을 것입니다. Rust는 이것을 *빌드 단계에서* 잡습니다. 해결책은 명백합니다. 컴파일러가 요구하는 대로 `From<io::Error>`를 `MyError`에 구현해 주면 됩니다. 다음 절에서 바로 그 작업을 합니다.

---

## 13.4 커스텀 에러 enum: 파이썬 예외 계층의 이주

이제 본론입니다. 파이썬의 `AppError → {DBError, ValidationError, NetworkError}` 계층을 Rust로 옮겨봅시다. Rust에는 상속이 없습니다([Chapter 8](./chapter_08.md)에서 본 대로 상속은 컴포지션과 트레이트로 대체됩니다). 에러의 "여러 종류 중 하나"라는 본질은 클래스 계층이 아니라 **열거형(Enum)**으로 표현하는 것이 자연스럽습니다. 파이썬의 `isinstance(e, DBError)` 분기는 Rust의 `match e { AppError::Db(..) => ... }` 패턴 매칭으로 대응됩니다.

먼저 표준 라이브러리만으로, 즉 매크로 없이 손으로 전부 구현해 봅시다. 보일러플레이트의 실체를 봐야 `thiserror`가 무엇을 자동화하는지 체감할 수 있습니다.

```rust
use std::fmt;
use std::io;
use std::num::ParseIntError;

// 파이썬의 AppError 계층에 대응하는 단일 enum
#[derive(Debug)]
enum AppError {
    Db(io::Error),                // DBError 대응 (예시로 io::Error 래핑)
    Validation { field: String }, // ValidationError 대응
    Network(io::Error),           // NetworkError 대응
}

// 요구사항 1: Display — 사람이 읽을 메시지
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Db(e) => write!(f, "DB 오류: {e}"),
            AppError::Validation { field } => write!(f, "검증 실패: '{field}' 필드"),
            AppError::Network(e) => write!(f, "네트워크 오류: {e}"),
        }
    }
}

// 요구사항 2: Error 트레이트 — source()로 원인 체인 노출
impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::Db(e) | AppError::Network(e) => Some(e),
            AppError::Validation { .. } => None, // 하위 원인 없음
        }
    }
}

// 요구사항 3: From — ?가 io::Error를 AppError로 자동 변환하게
impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Db(e) // 기본적으로 DB 오류로 흡수 (예시)
    }
}
```

세 가지 요구사항이 보입니다. **`Display`**(사용자용 메시지), **`std::error::Error`**(원인 체이닝), **`From`**(`?`를 위한 자동 변환). 13.3에서 컴파일러가 요구했던 바로 그 `From` 구현이 여기 들어갑니다. 이제 함수는 `?` 하나로 모든 에러를 전파할 수 있습니다.

```rust
// io::Error가 자동으로 AppError::Db로 변환되어 전파된다
fn read_raw(path: &str) -> Result<String, AppError> {
    let mut s = String::new();
    std::fs::File::open(path)?.read_to_string(&mut s)?; // io::Error -> AppError
    Ok(s)
}
```

`std::error::Error` 트레이트의 핵심은 `source()` 메서드입니다. 이것이 파이썬의 예외 체이닝(`raise ... from ...` 또는 `__cause__`)에 대응합니다. 에러가 다른 에러를 감쌀 때, `source()`는 그 안쪽 원인을 가리키는 참조를 돌려줍니다. 호출자는 `source()`를 반복 호출해 "포트 파싱 실패 ← 잘못된 숫자 ← ..." 같은 원인 사슬 전체를 거슬러 올라갈 수 있습니다.

> **흥미로운 사실**: `Error` 트레이트는 `Debug + Display`를 슈퍼트레이트(supertrait)로 요구합니다. 즉 `impl std::error::Error for T`를 쓰려면 `T`는 반드시 `Debug`(개발자용, `{:?}`)와 `Display`(사용자용, `{}`)를 둘 다 구현해야 합니다. 파이썬의 `Exception`은 `__str__`(Display)만 사실상 강제하지만, Rust는 디버깅용 표현과 사용자용 표현을 타입 수준에서 분리하도록 요구합니다.

### `thiserror`: 라이브러리를 위한 보일러플레이트 제거기

위 코드의 `Display`, `Error::source`, `From` 구현은 전부 *기계적*입니다. 패턴이 뻔합니다. `thiserror` 크레이트는 이 셋을 파생 매크로(derive macro, [Chapter 19](./chapter_19.md)에서 매크로의 내부를 다룹니다)로 자동 생성합니다. 위 30여 줄이 다음으로 줄어듭니다.

```toml
# Cargo.toml
[dependencies]
thiserror = "2"
```

```rust
use std::io;
use std::num::ParseIntError;
use thiserror::Error;

#[derive(Debug, Error)] // Error 파생이 Display/Error/From까지 생성
pub enum ConfigError {
    #[error("설정 파일을 읽지 못했습니다")]
    Io(#[from] io::Error), // #[from]이 From<io::Error> 자동 생성

    #[error("포트 번호 '{0}'를 정수로 파싱할 수 없습니다")]
    InvalidPort(#[from] ParseIntError),

    #[error("필수 키 '{key}'가 누락되었습니다")]
    MissingKey { key: String },
}

// 에러 타입을 한 번만 적어두는 관례적 타입 별칭
type Result<T> = std::result::Result<T, ConfigError>;

fn parse_port(raw: &str) -> Result<u16> {
    let port: u16 = raw.trim().parse()?; // ParseIntError -> ConfigError 자동 변환
    if port < 1024 {
        return Err(ConfigError::MissingKey { key: "port>=1024".into() });
    }
    Ok(port)
}

fn main() {
    if let Err(e) = parse_port("abc") {
        eprintln!("에러: {e}"); // 에러: 포트 번호 'invalid digit found in string'를 ...
    }
}
```

각 매크로 속성의 역할은 명확합니다.

- `#[derive(Error)]` — `Display`(아래 `#[error(...)]`로부터)와 `std::error::Error::source`를 자동 구현.
- `#[error("...")]` — 해당 variant의 `Display` 메시지. `{0}`은 튜플 필드, `{key}`는 이름 있는 필드를 보간(interpolate)합니다.
- `#[from]` — 해당 inner 타입에 대한 `From` 구현을 생성. 동시에 그 필드를 `source()`로 노출.

> **🚨 함정: `#[from]`과 `#[error("{0}")]`의 미묘한 조합**
>
> 위 코드에서 `InvalidPort(#[from] ParseIntError)`의 메시지를 `'{0}'`로 적었습니다. 직관적으로는 파싱하려던 원본 문자열 `"abc"`가 들어갈 것 같지만, 실제 출력은 `포트 번호 'invalid digit found in string'를 ...`입니다. `{0}`은 0번 필드, 즉 `ParseIntError` 그 자체의 `Display`를 보간하기 때문입니다. 원본 입력값을 메시지에 남기고 싶다면 `#[from]` 대신 명시적 필드(`InvalidPort { raw: String, source: ParseIntError }`)를 두고 `#[source]` 속성으로 원인을 표시해야 합니다. 시니어 개발자가 `thiserror`에서 가장 흔히 헷갈리는 지점입니다.

`type Result<T> = std::result::Result<T, ConfigError>;`라는 타입 별칭도 주목하세요. 한 모듈/크레이트 안에서 거의 모든 함수가 같은 에러 타입을 반환한다면, 매번 `Result<u16, ConfigError>`를 적는 대신 `Result<u16>`으로 줄일 수 있습니다. 표준 라이브러리의 `std::io::Result<T>`가 정확히 이 패턴입니다.

---

## 13.5 `anyhow`: 애플리케이션을 위한 만능 에러

`thiserror`가 "내 라이브러리 사용자에게 *정확히 어떤* 에러가 가능한지 enum으로 알려주는" 도구라면, `anyhow`는 정반대 철학입니다. "이건 최종 애플리케이션(바이너리)이고, 나는 에러가 *무엇이든* 상관없이 그냥 위로 흘려보내다가 맨 위에서 로그만 잘 찍으면 된다."

[Chapter 12](./chapter_12.md)에서 잠깐 본 `Box<dyn std::error::Error>`를 떠올려 봅시다. 그것이 "`Error` 트레이트를 구현한 무엇이든 힙에 박싱해 포인터로 들고 다니겠다"는 만능 타입이었습니다. `anyhow::Error`는 이 아이디어를 다듬어, 백트레이스(backtrace)와 컨텍스트(context) 체이닝 기능까지 얹은 애플리케이션용 에러 타입입니다.

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{bail, Context, Result}; // anyhow::Result<T> = Result<T, anyhow::Error>
use std::collections::HashMap;

fn load_setting(cfg: &HashMap<String, String>, key: &str) -> Result<String> {
    cfg.get(key)
        .cloned()
        // Option을 anyhow::Error로 바꾸면서 "왜 실패했는지" 맥락을 붙인다
        .with_context(|| format!("설정 키 '{key}'를 찾을 수 없습니다"))
}

fn parse_timeout(raw: &str) -> Result<u32> {
    let v: u32 = raw
        .trim()
        .parse()
        // Result에 .context()를 붙이면 원래 에러를 source로 보존하며 맥락 추가
        .context("timeout 값을 정수로 변환하는 데 실패했습니다")?;
    if v == 0 {
        bail!("timeout은 0일 수 없습니다"); // 즉시 Err(anyhow!(...))를 반환하는 매크로
    }
    Ok(v)
}

fn run(cfg: &HashMap<String, String>) -> Result<()> {
    let raw = load_setting(cfg, "timeout")?; // 어떤 에러든 ?로 전파
    let timeout = parse_timeout(&raw)?;
    println!("timeout = {timeout}s");
    Ok(())
}

fn main() {
    let mut cfg = HashMap::new();
    cfg.insert("timeout".to_string(), "xx".to_string());

    if let Err(e) = run(&cfg) {
        eprintln!("{e:#}"); // alternate 포맷: context 체인을 한 줄에 합쳐 출력
        eprintln!("---");
        eprintln!("{e:?}"); // Debug 포맷: 전체 원인 체인 + (활성화 시) 백트레이스
    }
}
```

실제 실행 결과는 다음과 같습니다(검증된 출력).

```text
timeout 값을 정수로 변환하는 데 실패했습니다: invalid digit found in string
---
timeout 값을 정수로 변환하는 데 실패했습니다

Caused by:
    invalid digit found in string
```

`anyhow`의 결정적 장점은 **`.context()`**입니다. 파이썬에서 우리는 종종 이렇게 합니다.

```python
try:
    timeout = int(raw)
except ValueError as e:
    raise AppError("timeout 값을 정수로 변환하는 데 실패했습니다") from e
    #                                                         ^^^^^^ 원인 보존
```

`raise ... from e`는 새 메시지를 얹으면서 원래 예외를 `__cause__`로 보존합니다. `anyhow`의 `.context()`가 정확히 이 일을 합니다. 원래 에러를 `source()` 체인의 안쪽으로 밀어 넣으면서 바깥에 사람이 읽을 맥락을 한 겹 씌웁니다. `{e:?}`로 출력하면 `Caused by:` 블록에 그 체인이 그대로 펼쳐집니다. 또한 `RUST_BACKTRACE=1` 환경 변수를 설정하면 에러가 처음 만들어진 지점의 백트레이스가 함께 출력되어, 파이썬 트레이스백에 익숙한 우리에게 친숙한 디버깅 경험을 줍니다.

### `main` 자체를 `Result`로

파이썬 스크립트의 최상단에서 잡지 못한 예외가 트레이스백과 함께 종료 코드 1을 내듯, Rust의 `main`도 `Result`를 반환할 수 있습니다.

```rust
use std::error::Error;

// main이 Err을 반환하면 Debug 출력과 함께 종료 코드 1로 끝난다
fn main() -> Result<(), Box<dyn Error>> {
    let n: i32 = "42".parse()?; // ParseIntError -> Box<dyn Error>로 자동 변환
    println!("{n}");
    Ok(())
}
```

`main`이 `Err`을 반환하면 런타임은 그 에러를 `Debug` 포맷으로 stderr에 찍고 프로세스를 종료 코드 1로 끝냅니다. `anyhow`를 쓴다면 `fn main() -> anyhow::Result<()>`로 적으면 context 체인과 백트레이스까지 깔끔하게 출력됩니다. 프로토타입이나 CLI 도구에서는 이 패턴 하나로 에러 처리 인프라가 완성됩니다.

### Under the Hood: `Box<dyn Error>`의 팻 포인터와 에러 enum의 크기

`anyhow::Error`와 `Box<dyn Error>`가 어떻게 "무엇이든" 담을 수 있는지는 메모리 레이아웃을 봐야 이해됩니다. 핵심은 **동적 디스패치(dynamic dispatch)**와 **팻 포인터(fat pointer)**입니다.

`Box<dyn Error>`는 단순한 포인터 하나가 아닙니다. 두 개의 워드(word)로 이루어진 **팻 포인터**입니다.

```text
Box<dyn Error>  (64비트 기준 16바이트)
┌──────────────────┬──────────────────┐
│  data pointer    │  vtable pointer   │
│  (힙의 실제 에러)  │  (메서드 테이블)   │
└──────────────────┴──────────────────┘
       8바이트              8바이트
```

첫 워드는 힙에 박싱된 실제 에러 데이터를 가리킵니다. 둘째 워드는 **vtable**을 가리킵니다. vtable에는 그 구체 타입의 `Display::fmt`, `Debug::fmt`, `Error::source`의 함수 포인터가 들어 있습니다. 그래서 `Box<dyn Error>`를 들고 있으면, 그 안의 실제 타입이 `io::Error`든 `ParseIntError`든 우리가 만든 `AppError`든 상관없이, vtable을 통해 올바른 `Display`를 호출할 수 있습니다. 이것이 파이썬에서 `except Exception as e: print(e)`가 어떤 예외든 받아 출력하는 것과 정확히 같은 다형성을, *런타임 타입 태그가 아니라 컴파일 타임에 만들어진 vtable*로 구현한 것입니다. 실제로 측정해 보면(`std::mem::size_of`):

```text
size_of::<Box<dyn Error>>()         = 16   // 팻 포인터: data + vtable
size_of::<Result<(), Box<dyn Error>>>() = 16   // 니치 최적화로 추가 비용 0
```

> **💡 정정: `anyhow::Error`는 16바이트가 아니라 8바이트다**
>
> 흔한 오해 하나를 바로잡고 갑시다. `anyhow::Error`가 내부적으로 `Box<dyn Error>`를 쓴다고 들으면, 그 크기도 팻 포인터인 16바이트일 것이라 짐작하기 쉽습니다. **틀렸습니다.** 직접 측정하면(rustc 1.90.0, `anyhow` 1.x):
>
> ```text
> size_of::<Box<dyn Error>>()       = 16   // 팻 포인터 (data + vtable)
> size_of::<anyhow::Error>()        = 8    // 씬 포인터 (thin pointer)!
> size_of::<anyhow::Result<()>>()   = 8    // 니치 최적화로 추가 비용 0
> ```
>
> `anyhow::Error`는 8바이트, 즉 포인터 *하나*입니다. 비결은 `anyhow`가 vtable을 팻 포인터의 둘째 워드로 들고 다니지 않고, **힙 할당 블록의 맨 앞에 vtable 포인터를 함께 박아 넣기** 때문입니다. 즉 `Box<dyn Error>`가 `(data, vtable)` 두 워드를 스택/레지스터에 얹고 다니는 대신, `anyhow::Error`는 `[vtable | 실제 에러 데이터]` 레이아웃의 힙 블록을 가리키는 포인터 하나만 들고 다닙니다. 이 설계 덕분에 `Result<T, anyhow::Error>`가 함수 사이를 오갈 때 옮겨야 할 에러 채널이 8바이트로 끝납니다 — `Box<dyn Error>`(16바이트)의 절반입니다. 에러 경로가 드물수록(거의 항상 그렇습니다) 성공 경로의 반환 비용을 작게 유지하는 이 8바이트 설계가 빛을 발합니다. "라이브러리 에러는 enum으로 슬림하게"라는 원칙을 `anyhow` 자신이 자기 타입에 적용해 둔 셈입니다.

반면 커스텀 enum은 **정적 디스패치**입니다. enum 자체가 모든 가능한 variant를 인라인으로 품기 때문에 vtable도 힙 박싱도 없습니다. 대신 크기는 가장 큰 variant에 의해 결정됩니다. 여기서 실무적으로 중요한 함정이 나옵니다.

```rust
use std::io;
use std::mem::size_of;

// 비대한 variant를 인라인으로 품은 에러
enum FatError {
    Io(io::Error),
    Validation { field: String, message: String, payload: [u8; 256] },
}

// 큰 데이터를 Box로 빼내 enum 본체를 슬림하게
enum SlimError {
    Io(io::Error),
    Validation(Box<ValidationDetail>),
}
struct ValidationDetail { field: String, message: String, payload: [u8; 256] }

fn main() {
    println!("FatError  = {}", size_of::<FatError>());  // 304
    println!("SlimError = {}", size_of::<SlimError>()); // 16
}
```

실제 측정 결과 `FatError`는 **304바이트**, `SlimError`는 **16바이트**입니다. enum의 크기는 "가장 큰 variant + 판별자(discriminant)"로 결정되므로, 거대한 `[u8; 256]`을 품은 `Validation` 하나가 *모든* `FatError` 값을 304바이트로 부풀립니다. 문제는 이 에러가 `Result<T, FatError>`로 함수에서 반환될 때마다 304바이트가 통째로 복사·이동된다는 점입니다. 에러는 "거의 발생하지 않는 경로"인데, 그 드문 에러를 위해 성공 경로의 모든 반환이 무거워지는 것은 본말이 전도된 것입니다. `SlimError`처럼 큰 variant를 `Box`로 빼면 enum은 16바이트로 슬림해지고, 비용은 *실제로 에러가 났을 때만* 힙 할당으로 지불됩니다. 이것이 `clippy::result_large_err` 린트가 잡아내는 안티패턴입니다.

---

## 13.6 thiserror vs anyhow — 언제 무엇을

두 크레이트는 경쟁자가 아니라 *역할 분담*입니다. 한 워크스페이스 안에서 라이브러리 크레이트는 `thiserror`로 에러를 정의하고, 그것을 소비하는 바이너리 크레이트는 `anyhow`로 받는 조합이 가장 흔합니다. 핵심 판단 기준은 **"내 에러를 보는 사람이 그 종류별로 다르게 분기 처리해야 하는가?"**입니다.

| 기준 | `thiserror` | `anyhow` |
|------|-------------|----------|
| 주 사용처 | **라이브러리/크레이트** (공개 API) | **애플리케이션/바이너리** (최종 실행물) |
| 생성하는 타입 | 구체적 `enum`/`struct` (이름 있는 variant) | 단일 불투명(opaque) `anyhow::Error` |
| 디스패치 | 정적 — `match`로 variant 분기 가능 | 동적 — 힙에 박싱. 스택 포인터는 8바이트(13.5) |
| 호출자의 처리 | variant별 정밀 분기 (`match e { Db => .., Validation => .. }`) | 보통 로깅 후 전파; 종류별 분기는 `downcast`로 |
| 메시지 | `#[error("...")]`로 variant마다 명시 | `.context()`로 호출 지점마다 누적 |
| `?` 친화성 | `#[from]`으로 특정 타입만 자동 변환 | 거의 모든 `Error`를 자동 흡수 |
| 의존성 무게 | 컴파일 타임 매크로, 런타임 비용 0 | 힙 할당 + 백트레이스 캡처 비용 |
| 파이썬 대응 | 정교한 커스텀 `Exception` 계층 | `except Exception as e` + `raise from` |
| 안정성 보장 | variant가 공개 API → 신중히 설계 | 내부 구현 → 자유롭게 바꿔도 됨 |

판단을 한 문장으로 압축하면 이렇습니다.

- **내 코드를 import 할 *다른 개발자*가 에러를 본다 → `thiserror`.** 그들은 `match`로 `NetworkError`만 재시도하고 `ValidationError`는 사용자에게 400을 돌려주는 식의 분기가 필요합니다. 불투명한 `anyhow::Error`로는 이게 불가능합니다.
- **에러를 보는 게 결국 *나 자신*(또는 로그 수집기)뿐이다 → `anyhow`.** CLI 도구, 배치 잡, 마이그레이션 스크립트, 서버의 최상위 핸들러처럼 "실패하면 맥락과 함께 로그 찍고 끝"인 경우, enum을 설계하는 노동은 순수한 낭비입니다.

> **🚨 가장 흔한 함정: 라이브러리에서 `anyhow`를 노출하지 마라**
>
> 신입이 흔히 저지르는 — 그리고 시니어도 마감에 쫓기면 저지르는 — 실수는 공개 라이브러리 함수의 시그니처에 `-> anyhow::Result<T>`를 적는 것입니다. 이러면 그 라이브러리를 쓰는 사람은 모든 에러를 단일한 불투명 덩어리로만 받게 됩니다. `ValidationError`인지 `NetworkError`인지 구분하려면 `downcast_ref::<ConcreteType>()`으로 일일이 시도해야 하는데, 그러려면 라이브러리 내부의 구체 타입을 알아야 하니 불투명성의 의미가 사라집니다. **라이브러리의 공개 경계에서는 항상 `thiserror`로 만든 구체 타입을 노출하세요.** `anyhow`는 그 라이브러리를 *사용하는* 애플리케이션 쪽 코드의 사적인 편의 도구로 남겨야 합니다.

`anyhow`에서 종류별 분기가 정말 필요할 때를 위한 탈출구도 있습니다. `downcast_ref`로 안에 든 구체 타입을 되찾을 수 있습니다.

```rust
use anyhow::Result;

fn handle(e: &anyhow::Error) {
    // anyhow::Error 안에 든 구체 타입을 시도해서 꺼낸다 (파이썬의 isinstance 분기와 유사)
    if let Some(io_err) = e.downcast_ref::<std::io::Error>() {
        eprintln!("I/O 계층 오류, 재시도 가능: {io_err}");
    } else {
        eprintln!("처리 불가, 그대로 전파: {e:#}");
    }
}
```

하지만 이 `downcast`가 자주 필요해지는 순간, 그것은 "사실 이 에러는 enum이었어야 한다"는 신호입니다. 13.4의 `thiserror`로 돌아가야 할 때입니다.

---

## 13.7 실전 Hands-on: 설정 로더의 에러 아키텍처

지금까지의 모든 조각 — `?`, 커스텀 enum, `thiserror`의 `#[from]`, 타입 별칭, `source()` 체이닝 — 을 하나의 현실적인 예제로 묶어봅시다. 시나리오는 "TOML 비슷한 설정 문자열을 파싱해 서버 설정을 만드는 라이브러리"입니다. 이 라이브러리는 다른 팀이 쓸 것이므로 `thiserror`로 구체적 에러를 노출합니다. 아래 코드는 `thiserror = "2"` 의존성만 있으면 그대로 `cargo run` 됩니다.

```rust
use std::num::ParseIntError;
use thiserror::Error;

// --- 라이브러리의 공개 에러 타입 (thiserror) ---
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("'{0}' 키를 찾을 수 없습니다")]
    MissingKey(String),

    #[error("'{key}' 값 '{value}'를 정수로 해석할 수 없습니다")]
    InvalidInt {
        key: String,
        value: String,
        source: ParseIntError, // #[from] 대신 명시적 source로 원본 보존
    },

    #[error("포트 {0}는 1024 미만의 예약 포트입니다")]
    ReservedPort(u16),
}

// 라이브러리 전역 Result 별칭
pub type Result<T> = std::result::Result<T, ConfigError>;

#[derive(Debug)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

// 한 줄 "key=value"에서 키를 찾아 값을 꺼낸다
fn get<'a>(lines: &'a [(&'a str, &'a str)], key: &str) -> Result<&'a str> {
    lines
        .iter()
        .find(|(k, _)| *k == key)
        .map(|(_, v)| *v)
        // Option -> Result 변환: 없으면 MissingKey
        .ok_or_else(|| ConfigError::MissingKey(key.to_string()))
}

// 정수 파싱: 실패 시 원본 값과 원인을 함께 보존
fn get_int(lines: &[(&str, &str)], key: &str) -> Result<u16> {
    let raw = get(lines, key)?; // MissingKey가 ?로 전파
    raw.trim().parse::<u16>().map_err(|source| ConfigError::InvalidInt {
        key: key.to_string(),
        value: raw.to_string(),
        source,
    })
}

pub fn parse_config(lines: &[(&str, &str)]) -> Result<ServerConfig> {
    let host = get(lines, "host")?.to_string(); // ? 전파
    let port = get_int(lines, "port")?; // ? 전파
    if port < 1024 {
        return Err(ConfigError::ReservedPort(port));
    }
    Ok(ServerConfig { host, port })
}

// --- 애플리케이션 쪽: 에러를 종류별로 분기 처리 ---
fn main() {
    let good = [("host", "0.0.0.0"), ("port", "8080")];
    let bad_port = [("host", "0.0.0.0"), ("port", "80")];
    let bad_int = [("host", "0.0.0.0"), ("port", "eighty")];
    let missing = [("host", "0.0.0.0")];

    for case in [&good[..], &bad_port[..], &bad_int[..], &missing[..]] {
        match parse_config(case) {
            Ok(cfg) => println!("✅ {cfg:?}"),
            // 구체적 enum이므로 종류별 정밀 분기가 가능하다 (anyhow였다면 불가능)
            Err(e @ ConfigError::ReservedPort(_)) => {
                println!("⚠️  설정 거부(보안): {e}");
            }
            Err(e @ ConfigError::InvalidInt { .. }) => {
                use std::error::Error; // source() 사용
                println!("❌ 파싱 오류: {e}");
                if let Some(cause) = e.source() {
                    println!("    └ 원인: {cause}"); // source 체이닝
                }
            }
            Err(e) => println!("❌ {e}"),
        }
    }
}
```

이 한 예제 안에 이번 장의 모든 결정이 녹아 있습니다.

- **공개 경계는 `thiserror`**: `ConfigError`는 이름 있는 variant를 가지므로, 호출자는 `ReservedPort`만 보안 경고로, `InvalidInt`만 파싱 오류로 정밀하게 분기합니다.
- **`#[from]` 대신 명시적 `source`**: `InvalidInt`는 원본 입력값(`value`)을 메시지에 남겨야 하므로, 13.4의 함정을 피해 `#[from]`을 쓰지 않고 `map_err`로 직접 만들었습니다. 그래도 `source` 필드 덕분에 `e.source()` 체이닝은 그대로 동작합니다.
- **`Option`과 `Result`의 다리**: `get`에서 `ok_or_else`로 `Option`을 `Result`로 변환해 `?` 전파 흐름에 합류시켰습니다.
- **타입 별칭**: 모든 함수가 `Result<T>`(= `Result<T, ConfigError>`)를 반환해 시그니처가 간결합니다.

만약 이 코드가 라이브러리가 아니라 한 번 쓰고 버릴 마이그레이션 스크립트였다면? `ConfigError` 정의를 통째로 지우고 모든 시그니처를 `anyhow::Result<...>`로 바꾼 뒤, `ok_or_else(...)` 대신 `.context("...")` 한 줄로 끝냈을 것입니다. 같은 로직, 다른 에러 전략 — 그 선택의 축이 바로 13.6의 표입니다.

---

## 13.8 Best Practices & 흔한 함정

### Best Practices 체크리스트

- **라이브러리는 `thiserror`, 애플리케이션은 `anyhow`.** 이 한 줄이 90%의 결정을 끝냅니다. 공개 API 경계에서 절대 `anyhow::Error`를 노출하지 마세요.
- **`?`로 전파, 한 곳에서 처리.** 깊은 함수는 에러를 `?`로 흘려보내기만 하고, `main`이나 요청 핸들러 같은 얕은 경계에서 단 한 번 `match`로 처리합니다. 파이썬의 "깊은 곳에서 raise, 얕은 곳에서 except"와 정확히 같은 구조입니다.
- **`Result` 타입 별칭을 모듈마다.** `pub type Result<T> = std::result::Result<T, MyError>;`로 시그니처 노이즈를 줄입니다. 표준 라이브러리도 그렇게 합니다.
- **에러 enum을 슬림하게 유지.** 큰 데이터(긴 `String` 여러 개, 배열)를 품는 variant는 `Box`로 빼서 enum 본체를 작게 유지하세요. `cargo clippy`가 `result_large_err` 린트로 잡아줍니다.
- **`.context()`로 맥락을 쌓되, 메시지에 입력값을 담아라.** "파일을 열 수 없음"보다 "설정 파일 '/etc/app.toml'을 열 수 없음"이 새벽 3시의 당신을 구합니다. `anyhow`는 `.with_context(|| ...)`, `thiserror`는 `#[error("... {field} ...")]`로.
- **`source()`를 끊지 마라.** `#[from]`이나 `#[source]`로 원인 체인을 보존하세요. `anyhow`의 `{e:#}`는 그 사슬을 출력합니다. 일반 `Error`의 `{e}`와 `{e:#}`는 맨 위 메시지만 내고, 사슬은 `source()`로 따라갑니다. `{e:?}`는 Debug입니다. rustc 1.99.1에서 일반 `Error`의 `{e:#}`는 맨 위 메시지였습니다.
- **`unwrap()`/`expect()`는 경계가 분명할 때만.** [Chapter 12](./chapter_12.md)에서 강조한 원칙은 여전히 유효합니다. 라이브러리 코드에서는 `?`로 전파하고, "여기는 절대 실패할 수 없다"가 증명되는 곳에서만 `expect("이유")`를 쓰되 그 이유를 메시지에 남기세요.

### 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: 모든 에러를 `Box<dyn Error>`로 뭉뚱그리기.** 파이썬의 `except Exception`이 익숙해서, Rust 어디서나 `Box<dyn Error>`를 반환하고 싶어집니다. 프로토타이핑에서는 괜찮습니다. 하지만 호출자가 에러 종류별로 분기하려는 순간, `downcast_ref::<T>()`로 일일이 타입을 추측해야 하는 지옥이 열립니다. 분기가 필요하면 enum으로 가세요.

**함정 2: 라이브러리 시그니처에 `anyhow::Result` 노출.** 13.6에서 강조한 그 함정입니다. 다시 한번: 당신의 함수를 `import` 할 사람이 있다면 `anyhow`는 시그니처에서 추방하세요.

**함정 3: 에러 enum 비대화(God Error).** 반대 극단도 있습니다. 거대한 모듈 하나에 50개 variant짜리 `AppError`를 만들고 모든 함수가 그걸 반환하게 하는 것. 이러면 (1) enum이 비대해져 성능이 나빠지고, (2) 어떤 함수가 실제로 어떤 variant를 낼 수 있는지 시그니처만으로는 알 수 없게 되어 — 파이썬의 청구서 1이 다른 모습으로 부활합니다. 에러 타입은 모듈/계층 단위로 쪼개고, 상위에서 `#[from]`으로 합치세요.

**함정 4: `?`를 쓰려는데 `From`이 없어서 막힘.** 13.3의 `error[E0277]`을 만나면 당황하지 마세요. 컴파일러가 정확히 무엇을 원하는지(`From<X> for MyError`) 알려줍니다. `thiserror`라면 해당 variant에 `#[from]`을 붙이면 끝, 손으로 짠다면 `impl From<X> for MyError`를 추가하면 됩니다.

**함정 5: `panic!`과 `Result`를 혼동하기.** 파이썬에서는 모든 비정상이 예외입니다. Rust는 *복구 가능한 에러*(`Result`)와 *복구 불가능한 버그*(`panic!`)를 의도적으로 분리합니다. "파일이 없다"는 `Result`(호출자가 폴백할 수 있음)지만, "배열 인덱스가 범위를 벗어났다"는 `panic!`(프로그래머의 논리 오류)입니다. 라이브러리 코드에서 입력 검증 실패에 `panic!`을 쓰면, 그것을 `try-except`로 잡을 방법이 사실상 없어 호출자를 곤란하게 만듭니다. 복구 가능성이 조금이라도 있으면 `Result`를 반환하세요.

---

## 13.9 벤치마크: 예외를 제어 흐름으로 쓰면 — 100만 건의 진실

13.1의 "청구서 3"에서 파이썬 예외가 *제어 흐름*으로 오용될 때 비싸진다고 주장했습니다. 주장은 측정으로 증명해야 합니다. 가장 흔한 핫루프 시나리오 — "100만 개의 문자열을 정수로 파싱하는데 절반이 실패한다" — 를 양쪽 언어로 똑같이 짜서 재 봅시다.

파이썬은 EAFP 스타일, 즉 일단 `int()`를 시도하고 `ValueError`를 `except`로 받는 전형적 관용구입니다.

```python
import time

N = 1_000_000
inputs = [str(i) if i % 2 == 0 else f"x{i}" for i in range(N)]

def run() -> tuple[int, int]:
    total, errors = 0, 0
    for s in inputs:
        try:
            total += int(s) * 2      # 절반은 여기서 ValueError
        except ValueError:           # 예외를 제어 흐름으로 소비
            errors += 1
    return total, errors
```

Rust는 같은 로직을 `Result` 반환과 `match`로 표현합니다. `Err`은 던지는 게 아니라 그냥 *값으로 반환*될 뿐입니다.

```rust
fn parse_or_zero(s: &str) -> Result<i64, std::num::ParseIntError> {
    let n: i64 = s.parse()?;
    Ok(n * 2)
}

fn run(inputs: &[String]) -> (i64, u64) {
    let (mut sum, mut errors) = (0i64, 0u64);
    for s in inputs {
        match parse_or_zero(s) {
            Ok(v) => sum = sum.wrapping_add(v),
            Err(_) => errors += 1, // Err을 제어 흐름으로 소비
        }
    }
    (sum, errors)
}
```

측정 환경은 동일합니다(Python 3.13.6, rustc 1.90.0 `--release`, 같은 머신, 각 7~12회 반복의 최솟값). 결과는 다음과 같습니다.

| 시나리오 (100만 건) | 총 시간 | 건당 | Rust 대비 |
|---|---|---|---|
| **Python** EAFP, 50% 예외 | **211 ms** | 211 ns | 약 120배 느림 |
| Python EAFP, 0% 예외 (행복 경로) | 47 ms | 47 ns | — |
| Python LBYL, 50% (`s.isdigit()` 선검사, 예외 없음) | 41 ms | 41 ns | — |
| **Rust** `Result`, 50% Err | **1.76 ms** | 1.76 ns | 1배(기준) |
| Rust `Result`, 0% Err (행복 경로) | 2.24 ms | 2.24 ns | — |

읽어낼 사실이 셋입니다.

**사실 1 — 절대 격차: 약 120배.** 같은 일(절반 실패하는 100만 건 파싱)을 Rust는 1.76ms, Python은 211ms에 합니다. CPython 인터프리터 루프 자체의 오버헤드가 큰 몫이지만, 그 위에 예외 메커니즘이 얹혀 격차를 벌립니다.

**사실 2 — 파이썬의 *예외 자체* 비용: 약 5배.** 같은 EAFP 코드인데 예외가 0%면 47ms, 50%면 211ms입니다. 차이 164ms는 순수하게 *50만 번 예외를 만들고 던지고 잡은* 비용입니다. 예외 한 건당 약 328ns(`(211-47)ms / 500_000`)가 트레이스백 객체 생성·스택 언와인딩·핸들러 매칭에 쓰입니다. `isdigit()`으로 선검사하는 LBYL 버전(41ms)이 EAFP 0% 버전과 비슷하다는 것도 같은 이야기입니다 — 예외를 *아예 발생시키지 않는* 것이 핵심입니다. "예외는 예외적인 상황에만 쓰라"는 조언의 정량적 근거가 바로 이 164ms입니다.

**사실 3 — Rust에는 비대칭이 없다.** Rust는 에러 50%(1.76ms)가 에러 0%(2.24ms)보다 오히려 *빠릅니다*. `Err` 경로가 곱셈·덧셈을 건너뛰기 때문일 뿐, 에러를 만드는 비용 자체는 `Ok`를 만드는 비용과 같습니다. `Result`는 그냥 값이고, `Err(e)` 생성에는 트레이스백도 스택 언와인딩도 없습니다. 그래서 Rust에서는 — 파이썬과 달리 — 에러가 흔한 핫루프에서도 `Result`를 제어 흐름으로 마음껏 써도 됩니다. EAFP의 성능 페널티가 구조적으로 존재하지 않습니다.

> **💡 측정의 정직함**: 위 120배는 "예외를 제어 흐름으로 쓰는 핫루프"라는 특정 시나리오의 수치입니다. CPython의 인터프리터 오버헤드와 예외 비용이 결합된 결과이므로, I/O 바운드 코드나 예외가 드문 코드에서는 격차가 훨씬 작습니다. 핵심 교훈은 "Rust가 무조건 120배 빠르다"가 아니라, **"파이썬에서 비싼 비대칭(예외 = 비싸고 정상 = 싸다)이 Rust에는 없다"**는 구조적 차이입니다.

## 13.10 한 발 앞서: `axum`에서 에러를 HTTP 상태코드로

이번 장의 마지막 조각은 실무 웹 서버로의 다리입니다. 13.6에서 "에러는 깊은 곳에서 `?`로 전파하고 얕은 경계에서 한 번 처리한다"고 했습니다. 백엔드 엔지니어에게 그 "얕은 경계"는 보통 **HTTP 핸들러**입니다. 파이썬 FastAPI라면 예외 핸들러를 등록해 `ValidationError → 400`, `NotFound → 404`로 매핑할 것입니다.

```python
# FastAPI 스타일: 예외를 상태코드로 매핑
@app.exception_handler(ValidationError)
async def on_validation(request, exc):
    return JSONResponse(status_code=400, content={"detail": str(exc)})
```

Rust의 웹 프레임워크 **`axum`**(tokio 기반, [Chapter 18](./chapter_18.md)의 async 위에 세워집니다)에서는 이 매핑을 **`IntoResponse` 트레이트** 한 곳에 모읍니다. 우리가 `thiserror`로 만든 에러 enum에 `IntoResponse`를 구현하면, 핸들러는 그냥 `?`로 에러를 전파만 하고 — 도메인 에러 → HTTP 응답 변환은 단 한 군데에서 일어납니다.

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    #[error("리소스를 찾을 수 없습니다: {0}")]
    NotFound(String),
    #[error("입력 검증 실패: {0}")]
    Validation(String),
    #[error("내부 오류")]
    Internal(#[from] std::io::Error),
}

// 도메인 에러 -> HTTP 상태코드 매핑을 한 곳에 집중
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = match &self {
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,          // 404
            ApiError::Validation(_) => StatusCode::BAD_REQUEST,      // 400
            ApiError::Internal(_) => StatusCode::INTERNAL_SERVER_ERROR, // 500
        };
        (status, self.to_string()).into_response()
    }
}

// 핸들러는 ? 로 전파만. 변환은 IntoResponse가 책임진다.
async fn get_user(id: u64) -> Result<String, ApiError> {
    if id == 0 {
        return Err(ApiError::Validation("id는 0일 수 없습니다".into()));
    }
    if id > 1000 {
        return Err(ApiError::NotFound(format!("user {id}")));
    }
    Ok(format!("user {id}"))
}
```

핸들러가 `Result<String, ApiError>`를 반환하면, `axum`은 `Ok`면 200으로, `Err(ApiError)`면 우리가 정의한 `into_response()`를 호출해 상태코드를 입힙니다. 핵심은 **변환 규칙이 타입 한 곳(`impl IntoResponse for ApiError`)에 모인다**는 것입니다. 새 에러 variant를 추가하면 컴파일러가 `match`의 비완전성을 잡아내므로([Chapter 10](./chapter_10.md)에서 본 망라성 검사), "새 에러 타입을 추가했는데 상태코드 매핑을 깜빡한" 버그가 컴파일 단계에서 차단됩니다. 실무에서는 본문을 `(status, Json(ErrorBody { ... }))`로 직렬화([Chapter 7](./chapter_07.md)의 serde)하지만, 골격은 위와 같습니다. PyO3로 Python에 Rust를 노출하는 [Chapter 22](./chapter_22.md)까지 가면, 이 에러 설계가 언어 경계를 넘어 어떻게 파이썬 예외로 다시 번역되는지도 보게 됩니다.

## 연습문제

아래 네 문제는 손으로 풀고 `cargo`로 검증하세요. 각 문제 뒤 '정답 스케치'는 핵심 아이디어와 자기검증용 컴파일러 에러코드/측정값을 담았습니다. Rustlings처럼, 먼저 막힌 뒤 정답을 보는 편이 오래 남습니다.

**문제 1 — `thiserror`의 `#[from]`.** `f64` 가격 문자열을 파싱하는 `parse_price(raw: &str) -> Result<f64, PriceError>`를 작성하세요. `PriceError`는 `thiserror`로 정의하되, (a) `std::num::ParseFloatError`를 `#[from]`으로 흡수하는 `Parse` variant와 (b) 음수 가격을 거부하는 `Negative(f64)` variant를 가져야 합니다. 본문은 `?` 한 번으로 파싱 에러를 전파해야 합니다.

> **정답 스케치.** `#[derive(Debug, Error)]` enum에 `Parse(#[from] ParseFloatError)`와 `Negative(f64)`를 둡니다. `#[from]`이 `From<ParseFloatError> for PriceError`를 자동 생성하므로 `let v: f64 = raw.trim().parse()?;`가 그대로 컴파일됩니다. `#[from]`을 빼면 `?` 줄에서 `error[E0277]: ? couldn't convert the error to PriceError`(13.3에서 본 그 메시지, `From<ParseFloatError>` 미구현)가 납니다. 자기검증: `matches!(parse_price("xx"), Err(PriceError::Parse(_)))`와 `matches!(parse_price("-3"), Err(PriceError::Negative(_)))`가 모두 `true`.

**문제 2 — 손으로 `From` 구현 (매크로 없이).** `thiserror` 없이, `ParseIntError`를 흡수하는 `CliError` enum을 정의하세요. `Display`, `std::error::Error`(`source()` 포함), 그리고 `From<ParseIntError>`를 손으로 구현해 `parse_level(raw) -> Result<i64, CliError>`가 `let n: i64 = raw.parse()?;`로 동작하게 만드세요. 추가로 0..=10 범위를 벗어나면 `OutOfRange(i64)`를 반환합니다.

> **정답 스케치.** 세 구현이 필요합니다 — `impl fmt::Display`(variant별 `write!`), `impl std::error::Error`(`source()`에서 `BadArg`는 `Some(e)`, `OutOfRange`는 `None`), `impl From<ParseIntError> for CliError`. `From` 구현을 빠뜨리면 `?` 줄에서 `error[E0277]`. `Error` 트레이트는 `Display + Debug`를 슈퍼트레이트로 요구하므로 `#[derive(Debug)]`도 빠지면 안 됩니다(빠지면 `error[E0277]: CliError doesn't implement Debug`). 자기검증: `parse_level("xx").unwrap_err().source().is_some()`가 `true`(원인 체인 보존됨), `parse_level("99")`은 `OutOfRange(99)`.

**문제 3 — `?` 전파 + `Option`→`Result` 다리.** `first_price(lines: &[&str]) -> Result<f64, PriceError>`를 작성하세요. 비어 있지 않은 첫 줄을 찾아(`Iterator::find`) 문제 1의 `parse_price`로 넘기되, 모든 줄이 비어 있으면 에러를 반환해야 합니다. `find`가 주는 `Option`을 `?` 흐름에 합류시키는 어댑터를 쓰세요.

> **정답 스케치.** `lines.iter().find(|l| !l.trim().is_empty())`는 `Option<&&str>`입니다. 이를 `.ok_or(...)?` 또는 `.ok_or_else(|| ...)?`로 `Result`화한 뒤 `?`로 전파하고, 꺼낸 줄을 `parse_price`에 넘기면 그 `Result`도 `?`로 이어집니다. 흔한 실수: `Result` 반환 함수 안에서 `Option`에 곧장 `?`를 쓰는 것 — `error[E0277]: the ? operator can only be used on Results, not Options, in a function that returns Result`가 나고, 컴파일러가 친절히 `use .ok_or(...)? to provide an error compatible with Result<...>`라고 해법까지 제시합니다(`Option`의 `?`는 `None` 전파용이라 에러 타입이 없습니다). 반드시 `ok_or`/`ok_or_else`로 변환하세요. 자기검증: `first_price(&["", " ", "9.9"]).unwrap() == 9.9`.

**문제 4 — `anyhow`와 `.context()`.** `anyhow`만 써서 `load(raw: &str) -> anyhow::Result<u32>`를 작성하세요. `raw`를 `u32`로 파싱하되, 실패하면 `.with_context(|| format!("'{raw}'를 u32로 변환 실패"))`로 원본 입력을 담은 맥락을 붙이세요. 그런 다음, 실패 시 `{e:#}` 출력에 그 맥락이, `e.source()`에 원래 `ParseIntError`가 보존되는지 확인하세요.

> **정답 스케치.** `use anyhow::{Context, Result};` 후 `let v: u32 = raw.trim().parse().with_context(|| ...)?;`. `.context()`/`.with_context()`는 `Result`의 `Err`에 한 겹을 씌우면서 원래 에러를 `source()`로 밀어 넣습니다(파이썬 `raise NewError("...") from e`와 동형). 자기검증: `format!("{e:#}").contains("u32로 변환 실패")`가 `true`이고 `e.source().is_some()`이 `true`. `with_context`(클로저, 지연 평가) 대신 `context`(즉시 평가)를 써도 되지만, 포맷 문자열을 매번 만들기 싫다면 실패할 때만 평가되는 `with_context`가 정석입니다. 라이브러리 공개 API라면 이 반환 타입(`anyhow::Result`)을 노출하면 안 된다는 13.6의 규칙도 함께 떠올리세요.

---

우리는 이제 Rust의 에러 처리 전체 그림을 손에 넣었습니다. [Chapter 12](./chapter_12.md)에서 `Option`과 `Result`라는 타입으로 실패를 *표현*하는 법을 배웠고, 이번 장에서는 그 실패를 `?`로 *전파*하고, `From`으로 *변환*하며, `thiserror`/`anyhow`로 *설계*하는 법을 배웠습니다. 파이썬의 암묵적 자동 전파가 주던 편의는 `?` 연산자가 한 글자로 되돌려주되, 전파되는 모든 에러 타입은 컴파일러가 영원히 추적합니다. 시그니처는 더 이상 거짓말하지 않고, 잡지 않은 에러는 배포될 수 없습니다.

지금까지 Part 1부터 Part 4까지, 우리는 Rust의 *사고방식* — 소유권, 트레이트, 열거형, 에러 — 을 익혔습니다. 이제 그 사고방식 위에 실제 데이터를 담을 차례입니다. 다음 [Chapter 14](./chapter_14.md)부터 시작하는 Part 5에서는, 파이썬 개발자가 매일 쓰는 `list`, `dict`, `str`이 Rust에서 `Vec<T>`, `HashMap<K, V>`, `String`/`&str`으로 어떻게 변신하는지를 다룹니다. 특히 파이썬의 문자열 인덱싱이 Rust에서 왜 그대로 컴파일되지 않는지 — UTF-8과 소유권이 만나는 그 지점이 — 시니어 파이썬 개발자를 가장 자주 넘어뜨리는 다음 벽입니다.

---

## References

- https://doc.rust-lang.org/book/ch09-00-error-handling.html
- https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html
- https://doc.rust-lang.org/std/error/trait.Error.html
- https://doc.rust-lang.org/reference/expressions/operator-expr.html
- https://docs.rs/thiserror/latest/thiserror/
- https://docs.rs/anyhow/latest/anyhow/
