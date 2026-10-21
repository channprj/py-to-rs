# Chapter 10. 열거형(Enums)을 활용한 상태 기계와 다형성

10년 동안 Python으로 서버를 짜면서, 한 번쯤은 이런 새벽 호출을 받아본 적 있을 겁니다. 결제 시스템에 새로운 상태 `PartiallyRefunded`(부분 환불)를 추가했는데, 그걸 처리하는 `if-elif` 분기 중 단 한 군데를 빼먹어서 운영 환경에서 `UnboundLocalError` 혹은 조용한 잘못된 값이 흘러갔던 경험. 코드 리뷰에서도, 단위 테스트에서도 잡히지 않았던 그 누락 말입니다.

문제의 본질은 이렇습니다. 우리가 "상태(State)"나 "종류(Variant)"를 표현하는 도구 — 문자열 상수, `Enum` 클래스, `Union[A, B]` + `isinstance()` — 가운데 **"가능한 모든 경우를 빠짐없이 처리했는가(Exhaustiveness)"**를 강제로 검증해 주는 도구가 하나도 없다는 것입니다. mypy도 `match` 문에 대해 일부 검사를 해주긴 하지만, `if-elif` 사슬이나 딕셔너리 디스패치에 대해서는 무력합니다. 결국 누락을 막는 것은 사람의 주의력과 테스트 커버리지뿐이고, 둘 다 새벽 3시의 핫픽스 앞에서는 무력합니다.

Chapter 8에서 우리는 상속 트리를 컴포지션으로 해체했고, Chapter 9에서는 던더 메서드를 트레이트(Trait)로 옮기며 "행동(Behavior)을 컴파일 타임에 강제하는" 법을 배웠습니다. 이번 챕터는 그 흐름의 정점입니다. Rust의 **열거형(Enum)**은 단순한 상수 집합이 아니라 **대수적 데이터 타입(Algebraic Data Types, ADT)** — 각 variant가 자기만의 데이터를 품는 합 타입(Sum Type) — 이며, 여기에 **패턴 매칭(Pattern Matching)**의 망라성 검사가 결합되면, "상태 누락"이라는 버그 카테고리 전체가 컴파일 단계에서 물리적으로 불가능해집니다.

이 챕터에서 우리는 두 가지 질문에 답합니다. 첫째, `Union`과 `isinstance()`가 만들어내는 런타임 위험을 Rust가 어떻게 타입 시스템 안으로 흡수하는가. 둘째, 다형성(Polymorphism)을 구현하는 Rust의 두 갈래 길 — 정적인 닫힌 집합(Enum)과 동적인 열린 집합(Trait Object) — 가운데 언제 무엇을 골라야 하는가. 그리고 그 사이사이, enum이 메모리에서 실제로 몇 바이트를 차지하는지, `Option<&T>`가 어떻게 포인터 하나 크기(8바이트)에 압축되는지를 바이트 단위로 뜯어볼 것입니다.

---

## 10.1 The Python Way — 상태를 표현하는 세 가지 방식과 그 한계

Python에서 "이 값은 여러 종류 중 하나다"를 표현하는 관용구는 세 가지로 수렴합니다. 각각을 결제 시스템의 상태 모델링이라는 렌즈로 살펴보겠습니다.

### 방식 1: 문자열/`Literal` 상수

가장 흔하고, 가장 위험합니다.

```python
from typing import Literal

PaymentState = Literal["pending", "authorized", "captured", "failed", "refunded"]

def describe(state: PaymentState) -> str:
    if state == "pending":
        return "결제 대기 중"
    elif state == "authorized":
        return "승인됨"
    elif state == "captured":
        return "매입 완료"
    # 🚨 "failed", "refunded"를 깜빡함 — 함수는 None을 반환한다
```

`Literal`은 mypy에게 힌트를 주긴 하지만, 위 `if-elif` 사슬에서 모든 케이스를 다뤘는지는 **검사하지 않습니다**. `describe("failed")`는 조용히 `None`을 반환하고, 그 `None`은 호출자에게 흘러가 한참 떨어진 곳에서 `TypeError`로 터집니다. 게다가 `state == "capturedd"` 같은 오타는 영원히 거짓이 되지만 아무도 알려주지 않습니다.

### 방식 2: `enum.Enum` 클래스

조금 낫습니다. 오타는 막을 수 있습니다.

```python
from enum import Enum, auto

class PaymentState(Enum):
    PENDING = auto()
    AUTHORIZED = auto()
    CAPTURED = auto()
    FAILED = auto()
    REFUNDED = auto()

def describe(state: PaymentState) -> str:
    if state is PaymentState.PENDING:
        return "결제 대기 중"
    elif state is PaymentState.AUTHORIZED:
        return "승인됨"
    # 나머지 누락 — 여전히 컴파일/린트 단계에서 못 잡는다
    raise ValueError(f"unhandled: {state}")
```

`enum.Enum`은 멤버 오타를 `AttributeError`로 잡아주는 진보입니다. 하지만 두 가지 한계가 남습니다. 첫째, **여전히 망라성 검사가 없습니다.** 새 멤버 `PARTIALLY_REFUNDED`를 추가해도, 위 함수는 컴파일 에러 없이 그저 `ValueError`를 던지는 런타임 폭탄으로 남습니다. 둘째, 그리고 더 본질적으로, **enum 멤버는 데이터를 품을 수 없습니다.** `AUTHORIZED` 상태에는 카드사 승인 코드(`auth_code`)가, `CAPTURED` 상태에는 영수증 ID가, `FAILED` 상태에는 실패 사유가 따라붙어야 합니다. Python의 `Enum`으로는 "상태"와 "그 상태에 딸린 데이터"를 한 타입으로 묶을 수 없어서, 결국 별도의 필드들을 객체에 흩뿌리게 됩니다.

### 방식 3: `Union` + `dataclass` + `isinstance()`

데이터를 품게 하려면 결국 클래스를 여러 개 만들고 `Union`으로 묶게 됩니다. (기존 스켈레톤에 있던 메시지 예제를 보존·확장합니다.)

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

def process_message(msg: Message) -> None:
    # 런타임에 타입을 하나하나 검사 (속도 저하 + 누락 위험)
    if isinstance(msg, QuitMessage):
        print("Quit")
    elif isinstance(msg, WriteMessage):
        print(f"Write: {msg.text}")
    elif isinstance(msg, MoveMessage):
        print(f"Move to {msg.x}, {msg.y}")
    else:
        # 누군가 ChangeColorMessage를 추가하면 여기서 터진다
        raise ValueError("Unknown message type")
```

이 방식은 데이터를 품을 수는 있지만, 세 가지 비용을 동반합니다.

1. **런타임 비용.** `isinstance()`는 객체의 `__class__`를 따라 MRO(Method Resolution Order)를 거슬러 올라가며 타입을 확인하는 런타임 연산입니다. 분기마다 이 검사가 반복됩니다.
2. **누락 위험.** `Message`에 `ChangeColorMessage`를 추가했을 때, `process_message`를 고치지 않으면 컴파일도 린트도 통과합니다. 버그는 운영 환경의 `else` 절에서 처음 모습을 드러냅니다.
3. **타입 좁히기(Narrowing)의 취약성.** mypy의 `exhaustiveness` 패턴(`assert_never`)을 수동으로 끼워 넣지 않는 한, 분기 누락은 정적 분석에서도 새어나갑니다. 그리고 그 `assert_never`조차 개발자가 매번 기억해서 넣어야 합니다.

세 방식 모두의 공통 결함은 명확합니다. **"가능한 모든 경우의 집합"과 "내가 실제로 처리한 경우의 집합"이 일치하는지를 언어가 책임지지 않는다는 것.** 그 책임을 사람에게 떠넘기는 한, 새벽 호출은 계속됩니다.

---

## 10.2 The Rust Mental Model — Enum은 곱이 아니라 합(Sum)이다

여기서 잠깐 대학 시절의 타입 이론을 꺼내봅시다. 시니어 엔지니어라면 직관적으로 알고 있는 내용입니다.

`struct`는 **곱 타입(Product Type)**입니다. `struct Point { x: f64, y: f64 }`가 표현할 수 있는 값의 개수는 `f64`가 가질 수 있는 값의 개수의 **곱**입니다. x와 y를 *동시에* 가지기 때문입니다. Python의 `dataclass`, `Pydantic` 모델, 튜플이 모두 곱 타입입니다.

`enum`은 **합 타입(Sum Type)**입니다. `enum Shape { Circle(f64), Rect(f64, f64) }`가 표현할 수 있는 값의 개수는 각 variant가 가질 수 있는 값의 개수의 **합**입니다. 어느 한순간에는 Circle *이거나* Rect, 둘 중 하나이기 때문입니다.

대수적 데이터 타입(ADT)이라는 이름은 여기서 나옵니다. 타입을 곱(struct)과 합(enum)으로 *대수적으로* 조합한다는 뜻입니다. Python의 `Union`이 하려던 것이 바로 이 합 타입인데, Python은 이것을 "여러 독립적인 클래스를 느슨하게 묶은 표지판" 수준으로밖에 표현하지 못합니다. Rust는 이것을 **하나의, 닫힌(closed), 일급(first-class) 타입**으로 표현합니다.

"닫혀 있다"는 것이 핵심입니다. `Message`라는 enum을 정의하는 순간, 컴파일러는 **세상에 존재할 수 있는 `Message`의 모든 형태를 완전히 알게 됩니다.** Python의 `Union`은 어디서든 새 멤버가 추가될 수 있는 열린 표지판이지만, Rust의 enum은 정의된 그 자리에서 우주가 확정됩니다. 이 "전지(omniscience)"가 바로 망라성 검사를 가능하게 하는 토대입니다.

같은 메시지 예제를 Rust로 옮겨봅시다.

```rust
// 세 가지 각기 다른 데이터 구조를 하나의 Enum 타입으로 묶음
enum Message {
    Quit,                       // 데이터 없는 단위(Unit) variant
    Write(String),              // 튜플 variant: String 하나를 품음
    Move { x: i32, y: i32 },    // 구조체 variant: 이름 붙은 필드를 품음
}

// 이제 Message라는 단일 타입으로 모든 것을 처리한다
fn process_message(msg: Message) {
    // match는 if-elif-else의 완벽한 상위 호환이다
    match msg {
        Message::Quit => println!("Quit"),

        // variant 내부 데이터를 즉시 추출(Destructuring)하여 바인딩
        Message::Write(text) => println!("Write: {text}"),

        Message::Move { x, y } => println!("Move to {x}, {y}"),
    }
}
```

`Union[QuitMessage, WriteMessage, MoveMessage]`가 세 개의 분리된 클래스를 가리키는 손가락이었다면, `enum Message`는 그 세 형태를 **물리적으로 한 메모리 슬롯 안에 담는 단일 타입**입니다. `isinstance()`라는 런타임 검사도 없습니다. `match`는 enum에 숨겨진 정수 태그(discriminant)를 점프 테이블로 분기할 뿐이며, 이건 C의 `switch`만큼 빠릅니다. (이 태그의 정체는 10.6에서 바이트 단위로 해부합니다.)

> 💡 **핵심.** Python의 `Union`이 "이 변수는 이 클래스들 중 하나를 *참조*한다"는 런타임 관찰이라면, Rust의 `enum`은 "이 메모리 영역은 이 형태들 중 하나를 *담는다*"는 컴파일 타임 사실입니다. 전자는 누락을 허용하고, 후자는 누락을 금지합니다.

---

## 10.3 패턴 매칭의 망라성 — 컴파일러가 상태 누락을 물리적으로 차단한다

이제 이 챕터의 심장으로 들어갑니다. Rust의 `match`는 "모든 경우를 다뤘는가"를 컴파일 타임에 **강제**합니다. 말로만 듣지 말고, 실제 컴파일러가 무엇을 뱉는지 직접 봅시다.

### 컴파일러 에러 시연: `error[E0004]`

다음 코드는 결제 상태 5개 중 3개만 처리하고 `Failed`, `Refunded`를 빼먹었습니다.

```rust
enum PaymentStatus {
    Pending,
    Authorized,
    Captured,
    Failed,
    Refunded,
}

fn describe(status: &PaymentStatus) -> &str {
    match status {
        PaymentStatus::Pending => "결제 대기 중",
        PaymentStatus::Authorized => "승인됨",
        PaymentStatus::Captured => "매입 완료",
        // 🚨 Failed와 Refunded를 깜빡함!
    }
}

fn main() {
    let s = PaymentStatus::Pending;
    println!("{}", describe(&s));
}
```

`rustc`를 돌리면 컴파일이 **거부**됩니다.

```text
error[E0004]: non-exhaustive patterns: `&PaymentStatus::Failed` and
              `&PaymentStatus::Refunded` not covered
  --> nonexhaustive.rs:10:11
   |
10 |     match status {
   |           ^^^^^^ patterns `&PaymentStatus::Failed` and
   |                  `&PaymentStatus::Refunded` not covered
   |
note: `PaymentStatus` defined here
  --> nonexhaustive.rs:1:6
   |
 1 | enum PaymentStatus {
   |      ^^^^^^^^^^^^^
...
 5 |     Failed,
   |     ------ not covered
 6 |     Refunded,
   |     -------- not covered
   = note: the matched value is of type `&PaymentStatus`
help: ensure that all possible cases are being handled by adding a match arm
      with a wildcard pattern, a match arm with multiple or-patterns as shown,
      or multiple match arms
   |
13 ~         PaymentStatus::Captured => "매입 완료",
14 ~         &PaymentStatus::Failed | &PaymentStatus::Refunded => todo!(),
   |

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0004`.
```

이 메시지를 해독해 봅시다.

- **`non-exhaustive patterns`**: 패턴이 모든 경우를 덮지 못했다.
- **`Failed and Refunded not covered`**: 정확히 어떤 variant가 빠졌는지 *이름으로* 짚어준다.
- **`note: defined here`**: enum이 어디에 정의되어 있는지, 그리고 빠진 멤버가 어디인지 소스 위치까지 가리킨다.
- **`help:`**: 해결책까지 제안한다 — 와일드카드를 쓰거나, or-패턴으로 묶어 처리하라고.

이것이 10.1에서 본 Python `if-elif`와의 결정적 차이입니다. Python에서 `Failed`를 빼먹은 코드는 **빌드되고, 배포되고, 운영에서 터집니다.** Rust에서 같은 실수는 **빌드되지 않습니다.** 새 variant `PartiallyRefunded`를 enum에 추가하면, 그것을 처리하지 않은 *모든* `match` 문이 일제히 컴파일 에러를 일으킵니다. 컴파일러가 "여기, 여기, 그리고 여기도 고쳐"라며 수정할 곳의 목록을 정확히 건네주는 것입니다. 수백 개 파일짜리 코드베이스에서 상태를 하나 추가할 때, 이보다 든든한 동료는 없습니다.

### `match`의 정밀 도구들

망라성은 시작일 뿐입니다. `match`는 단순 분기가 아니라 **구조 분해(Destructuring) 기반의 표현식**입니다. (Chapter 3에서 본 "Rust의 모든 것은 표현식"이라는 원칙이 여기서도 작동합니다 — `match`는 값을 반환합니다.)

```rust
#[derive(Debug)]
enum Event {
    Click { x: i64, y: i64 },
    KeyPress(char),
    Scroll(i64),
}

fn handle(ev: &Event) -> String {
    match ev {
        // ① match guard: 패턴 + 추가 불리언 조건
        Event::Click { x, y } if *x < 0 || *y < 0 => "화면 밖 클릭".to_string(),
        Event::Click { x, y } => format!("클릭 ({x}, {y})"),

        // ② @ 바인딩: 범위 검사와 값 캡처를 동시에
        Event::Scroll(delta @ -10..=10) => format!("미세 스크롤 {delta}"),
        Event::Scroll(delta) => format!("큰 스크롤 {delta}"),

        // ③ 리터럴 매칭
        Event::KeyPress('q') => "종료".to_string(),
        Event::KeyPress(c) => format!("키 {c}"),
    }
}
```

세 가지 도구를 짚어봅시다.

1. **Match Guard (`if` 조건).** 패턴이 일치한 *후에* 추가 조건을 검사합니다. 주의할 점: **가드가 붙은 arm은 망라성 검사에서 "전체를 덮는다"고 인정받지 못합니다.** 위에서 가드 붙은 `Click` arm 하나만 있고 가드 없는 `Click` arm이 없다면, 컴파일러는 "음수가 아닌 클릭은 어디서 처리하느냐"며 E0004를 냅니다. 그래서 가드 arm 뒤에는 반드시 가드 없는 포괄 arm이 따라와야 합니다.
2. **`@` 바인딩.** `delta @ -10..=10`은 "값이 -10에서 10 사이인지 검사하면서, 동시에 그 값을 `delta`라는 이름으로 캡처"합니다. 범위에 매칭시키면서 그 값 자체도 쓰고 싶을 때 쓰는 외과용 메스입니다.
3. **리터럴/범위 매칭.** `'q'` 같은 구체적 값, `-10..=10` 같은 범위를 직접 패턴으로 쓸 수 있습니다.

실행하면 다음과 같습니다.

```text
클릭 (10, 20)
화면 밖 클릭
미세 스크롤 3
큰 스크롤 500
종료
키 a
```

### `_ =>` 와일드카드: 편리하지만 양날의 검

`match`가 망라성을 강제한다면, 빠른 길은 `_ => ...`로 "나머지 전부"를 잡는 것입니다.

```rust
fn describe(status: &PaymentStatus) -> &str {
    match status {
        PaymentStatus::Pending => "대기",
        PaymentStatus::Authorized => "승인",
        _ => "기타",   // 🚨 나머지 전부를 한 번에 처리 — 컴파일은 통과한다
    }
}
```

이 코드는 컴파일됩니다. 하지만 여기에 함정이 있습니다. **`_`는 망라성 검사를 무력화합니다.** 나중에 enum에 `PartiallyRefunded`를 추가해도, `_`가 조용히 그것을 삼켜버리기 때문에 컴파일러는 아무 경고도 하지 않습니다. enum의 가장 큰 무기 — "변경 시 모든 처리 지점을 알려주는 능력" — 를 `_` 한 줄로 스스로 버린 셈입니다.

규칙은 이렇습니다. **"의미상 모든 미래의 variant를 동일하게 취급해도 되는 곳"에서만 `_`를 쓰십시오.** 그렇지 않은 곳, 즉 상태마다 다른 처리가 필요할 가능성이 있는 곳에서는, 귀찮더라도 모든 variant를 명시적으로 나열하십시오. 컴파일러가 미래의 당신을 지켜줄 수 있도록. 이 함정은 10.8에서 다시 다룹니다.

### Python 3.10 `match`와 Rust `match`는 이름만 같다

Python 3.10이 `match` 문(구조적 패턴 매칭, PEP 634)을 도입하면서, 표면적으로는 Rust의 `match`와 닮아 보입니다. 하지만 둘은 **철학이 정반대**입니다. Python의 `match`는 *편의 문법*이고, Rust의 `match`는 *증명 도구*입니다. 시니어라면 이 차이를 정확히 알아야, Python의 `match`를 Rust의 안전망으로 착각하는 사고를 피할 수 있습니다.

가장 위험한 차이부터 봅시다. Python의 `match`는 망라성을 **검사하지 않습니다.** 누락된 case가 있으면, 어떤 case에도 걸리지 않은 값은 그냥 블록을 빠져나가고 함수는 조용히 `None`을 반환합니다 (직접 측정).

```python
from enum import Enum

class S(Enum):
    A = 1
    B = 2
    C = 3

def describe(s: S) -> str:
    match s:
        case S.A:
            return "a"
        case S.B:
            return "b"
        # C 누락 — 컴파일/런타임 어디서도 막지 않는다

print(repr(describe(S.C)))   # 측정 결과: None
```

두 번째 함정은 더 미묘하고, Python 패턴 매칭의 가장 악명 높은 발등찍기입니다. **`case`에 쓴 *맨이름(bare name)*은 값 비교가 아니라 캡처 패턴(capture pattern)입니다.** 즉 상수와 비교하는 게 아니라, 매칭된 값을 그 이름으로 *덮어쓰며 무조건 매칭*됩니다 (직접 측정).

```python
GREEN = "green"
color = "red"
match color:
    case GREEN:          # "GREEN과 같은가?"가 아니라 "color를 GREEN에 캡처"!
        print("matched, GREEN is now:", GREEN)   # 측정: matched, GREEN is now: red
print("after:", GREEN)   # 측정: after: red  ← 상수가 'red'로 덮였다
```

값과 비교하려면 `case Color.GREEN:`처럼 *점이 찍힌(dotted)* 이름을 써야만 합니다. Rust에는 이런 함정이 없습니다 — 패턴 위치의 식별자가 상수(`const`)나 enum variant 경로로 해석되면 값 비교이고, 새 이름이면 바인딩이며, 컴파일러가 `unreachable_patterns` 등으로 모호함을 잡아줍니다.

| 항목 | Python 3.13.6 `match` 문 | Rust `match` 식 |
| :--- | :--- | :--- |
| **망라성 검사** | 없음. 누락 시 조용히 통과, 함수는 `None` 반환 | 컴파일 에러 `error[E0004]`로 강제 |
| **표현식/문장** | 문장(statement). 값을 반환하지 않음 | 표현식. 모든 arm이 같은 타입의 값을 반환 |
| **맨이름 `case x`** | 캡처 패턴(무조건 매칭 + 변수 덮어쓰기) | 새 이름이면 바인딩, 상수/variant 경로면 값 비교 |
| **상수와 비교** | `case Color.GREEN`처럼 점 찍힌 이름만 가능 | `Color::Green` 또는 `const` 모두 자연스럽게 비교 |
| **가드** | `case p if cond:` 지원 | `pat if cond =>` 지원(가드 arm은 망라 인정 안 됨) |
| **타입 좁히기** | 런타임 `isinstance` 기반. 정적 보장 없음 | 컴파일 타임 타입 + 구조 분해, 런타임 검사 없음 |
| **검사 시점** | 런타임 | 컴파일 타임 |
| **데이터 추출** | `case Point(x, y)` 가능하나 `__match_args__` 의존 | variant 구조 분해가 타입에 내장 |
| **`_` 와일드카드** | 망라를 "달성"시키는 편의 | 망라를 *무력화*하는 양날의 검(10.3) |

요약하면, Python의 `match`는 길게 늘어진 `if-elif`를 읽기 좋게 만들어주는 *문법 설탕*이고, Rust의 `match`는 "내가 모든 경우를 다뤘다"는 사실을 컴파일러에게 *증명*하게 만드는 도구입니다. 같은 키워드를 쓰지만, 전자는 누락을 허용하고 후자는 누락을 금지합니다 — 이 한 줄이 10.1의 새벽 호출을 막느냐 마느냐를 가릅니다.

---

## 10.4 다형성의 두 갈래 길: Enum(정적) vs Trait Object(동적)

"부모 클래스 대신 Enum을 써야 할까, Trait을 써야 할까?" — Python 개발자가 Rust 아키텍처를 설계할 때 가장 자주, 그리고 가장 오래 고민하는 질문입니다. Python에서는 이 고민이 존재하지 않았습니다. 모든 것이 동적 디스패치였으니까요. 하지만 Rust는 두 가지 길을 모두 열어두었고, 선택은 성능과 확장성의 트레이드오프입니다.

**시나리오**: 다양한 UI 컴포넌트(버튼, 텍스트 필드)를 컬렉션에 담고 순회하며 `draw()`를 호출합니다.

### 길 1: Enum을 사용한 정적 디스패치 (Closed Set)

컴포넌트의 종류가 *고정*되어 있다면, enum이 정답입니다.

```rust
enum UIComponent {
    Button { label: String },
    TextField { value: String },
}

impl UIComponent {
    fn draw(&self) {
        match self {
            UIComponent::Button { label } => println!("Draw Button: {label}"),
            UIComponent::TextField { value } => println!("Draw Text: {value}"),
        }
    }
}

fn main() {
    let components = vec![
        UIComponent::Button { label: "Submit".to_string() },
        UIComponent::TextField { value: "ID".to_string() },
    ];
    for c in &components {
        c.draw();
    }
}
```

여기서 `Vec<UIComponent>`는 **모든 요소를 같은 크기로, 힙의 연속된 메모리에 촘촘히** 저장합니다. 캐시 지역성(cache locality)이 좋고, `c.draw()`는 컴파일 타임에 결정된 점프 테이블 분기일 뿐이라 추가 포인터 추적이 없습니다. 새 컴포넌트를 추가하려면 enum 정의와 `match`를 고쳐야 하지만 — 바로 그 "고쳐야 함"을 컴파일러가 강제하는 것이 안전망입니다.

### 길 2: Trait Object를 사용한 동적 디스패치 (Open Set)

외부 라이브러리 사용자가 *나도 모르는* 새 컴포넌트를 플러그인처럼 끼워 넣게 하려면, trait object입니다. 이것이 개방-폐쇄 원칙(Open-Closed Principle)에 가장 가까운 Rust 표현입니다.

```rust
// 행동 정의 (Chapter 9에서 본 Trait의 연장)
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
    // Box<dyn Draw>: "Draw를 구현한 어떤 타입이든 Heap에 두고 포인터로 가리키겠다"
    let components: Vec<Box<dyn Draw>> = vec![
        Box::new(Button { label: "Submit".to_string() }),
        Box::new(TextField { value: "ID".to_string() }),
    ];

    // 런타임에 V-Table을 뒤져 실제 draw()를 찾아 실행 (동적 디스패치)
    for c in &components {
        c.draw();
    }
}
```

`Vec<Box<dyn Draw>>`의 각 요소는 **힙에 흩어진 객체를 가리키는 포인터**입니다. 확장은 자유롭지만(누구든 `impl Draw`만 하면 끼워 넣을 수 있음), 메서드 호출마다 V-Table 참조라는 간접 비용이 듭니다.

Python은 구조적으로 *무조건* 이 2번 방식으로만 동작합니다. 모든 객체 메서드 호출은 `__dict__`와 MRO를 거치는 동적 조회입니다. 하지만 Rust에서는 대부분의 경우 — 종류가 닫혀 있는 90%의 상황 — 에서 성능이 훨씬 뛰어난 1번(Enum)이나 제네릭 단일형화(Monomorphization, Chapter 9 참고)를 선택할 수 있습니다.

### 결정 기준표

| 기준 | Enum (정적 디스패치) | Trait Object (동적 디스패치) |
| :--- | :--- | :--- |
| **집합의 성격** | 닫힘(Closed) — 종류가 고정 | 열림(Open) — 종류가 무한히 확장 |
| **누가 variant를 추가?** | enum 정의 소유자(나)만 | 외부 사용자도 `impl`로 가능 |
| **디스패치 방식** | 컴파일 타임 점프 테이블 | 런타임 V-Table(vtable) 포인터 추적 |
| **메모리 배치** | `Vec<T>`에 연속 저장(값) | `Vec<Box<dyn T>>`에 포인터 저장(힙 산재) |
| **요소 크기** | 모두 동일(최대 variant 기준) | 포인터 2개(fat pointer, 16바이트) |
| **캐시 지역성** | 우수 | 열등(포인터 추적 + 힙 산재) |
| **인라이닝/최적화** | 가능 | 거의 불가(호출 대상이 런타임 결정) |
| **망라성 검사** | `match`로 컴파일 타임 보장 | 없음(open set이므로 본질적으로 불가) |
| **새 종류 추가 비용** | enum + 모든 match 수정(컴파일러가 강제) | 새 struct + impl만(기존 코드 무수정) |
| **Python 대응** | (해당 없음 — Python엔 정적 디스패치가 없음) | 덕 타이핑 / 추상 기반 클래스(ABC) |

**한 문장 규칙**: 도메인이 정의하는 *상태/종류가 닫혀 있고 내가 소유한다면* Enum, 제3자가 *무한히 확장*하게 열어야 한다면 Trait Object. 결제 상태(Pending/Authorized/...)는 명백히 전자이고, 플러그인 시스템·렌더러 백엔드·서드파티 핸들러는 후자입니다.

### 디스패치 비용, 숫자로 보기

추상적인 "빠르다/느리다" 대신 대략의 수치 감각을 잡아둡시다. 정적 디스패치(enum `match` 또는 단일형화된 제네릭)의 메서드 호출은 점프 테이블 인덱싱 + 직접 점프로, 현대 CPU에서 **수 나노초 이하**이며 호출 대상이 컴파일 타임에 확정되므로 **인라이닝·상수 전파 같은 후속 최적화가 모두 열립니다.** 동적 디스패치(`dyn Trait`)는 V-Table 포인터를 읽고(메모리 접근 1회) 그 안의 함수 주소로 점프하는 간접 호출이라, 분기 예측이 빗나가면 수~수십 사이클이 더 들고 **인라이닝이 막힙니다.** 둘의 절대 차이는 보통 호출당 1~3나노초 수준이라 단발 호출에서는 무의미하지만, 수백만 요소를 타이트 루프로 도는 핫패스에서는 **2~10% 이상의 처리량 차이**로 누적되며, 무엇보다 인라이닝 차단이 컴파일러의 다른 최적화 연쇄를 끊는다는 점이 더 큽니다.

이 모든 것을 Python과 비교하면 규모가 달라집니다. Python의 메서드 호출은 *항상* `__dict__`/MRO를 거치는 해시 조회 + `PyObject` 박싱을 수반하므로, CPython에서 단순 디스패치 한 번이 Rust 정적 디스패치 대비 **수십 배** 느립니다(인터프리터 바이트코드 디스패치 오버헤드 포함). 즉, Rust에서 enum과 trait object 중 무엇을 고르든 Python보다는 압도적으로 빠르며, 둘 사이의 선택은 "Python보다 50배냐 48배냐"가 아니라 *설계상 확장성을 여느냐 닫느냐*의 문제로 봐야 합니다. 성급한 `dyn`은 안티패턴이지만, 마이크로 최적화를 위해 멀쩡한 trait object를 enum으로 강제 전환하는 것도 대개 시기상조입니다 — 핫패스라는 측정 근거가 있을 때만 바꾸십시오.

> **Under the Hood: Trait Object의 Fat Pointer (16바이트)**
>
> `Box<dyn Draw>`는 일반 포인터가 아닙니다. 그것은 **fat pointer** — 64비트 시스템에서 16바이트짜리 두 개의 포인터 묶음 — 입니다.
>
> ```text
> Box<dyn Draw> (16 bytes)
> ┌────────────────────┬────────────────────┐
> │  data ptr (8B)     │  vtable ptr (8B)    │
> └────────┬───────────┴──────────┬─────────┘
>          │                      │
>          ▼                      ▼
>   힙의 실제 객체           V-Table (정적 영역, 타입당 1개)
>   (Button { label })     ┌──────────────────────┐
>                          │ drop_in_place 포인터  │
>                          │ size, align           │
>                          │ draw() 함수 포인터     │
>                          └──────────────────────┘
> ```
>
> 첫 번째 포인터는 힙의 실제 데이터를, 두 번째 포인터는 그 타입의 메서드 구현 주소가 적힌 V-Table을 가리킵니다. `c.draw()`는 "vtable 포인터를 따라가서 → draw 슬롯의 함수 주소를 읽고 → 그 주소로 점프"하는 **2단계 간접 호출**입니다. 반면 enum의 `match` + 메서드 호출은 점프 테이블 한 번이며, 호출 대상이 컴파일 타임에 확정되므로 인라이닝까지 가능합니다. Python의 모든 메서드 호출이 사실상 이 V-Table 추적(그것도 해시맵 조회를 동반한)과 같다는 점을 생각하면, Rust가 왜 굳이 두 길을 나눠놨는지 이해될 겁니다 — 비용을 *선택*하게 하려는 것입니다.

---

## 10.5 실전 Hands-on: 타입으로 잘못된 전이를 차단하는 결제 상태 머신

이제 챕터의 간판 예제를 본격적으로 확장합니다. 결제 시스템의 상태 머신을, **각 상태가 자기만의 데이터를 품도록** 설계하고, **잘못된 전이(transition)를 타입과 `match`로 막아내는** 완전한 프로그램을 만듭니다.

상태 전이 규칙은 이렇습니다.

```text
Pending ──authorize──▶ Authorized ──capture──▶ Captured ──refund──▶ Refunded
   │                       │                       
   └──────(fail)───────────┴────────────────▶ Failed
```

- `Pending`: 결제 요청만 들어온 상태. 금액(`amount`)만 안다.
- `Authorized`: 카드사 승인을 받음. 금액 + 승인 코드(`auth_code`).
- `Captured`: 실제 매입 완료. 금액 + 승인 코드 + 영수증 ID.
- `Failed`: 실패. 금액 + 실패 사유.
- `Refunded`: 환불. 원금 + 환불액 + 영수증 ID.

핵심 설계 결정: **"승인 코드는 Authorized 이후에만 존재한다"**는 사실을 타입으로 표현합니다. Python이라면 `auth_code: Optional[str] = None` 필드를 객체에 늘 달아두고, "Pending인데 auth_code가 있네?" 같은 불가능한 상태를 런타임 검사로 막아야 합니다. Rust에서는 그 데이터를 *해당 variant 안에만* 넣어서, 불가능한 상태를 표현조차 못 하게 만듭니다.

```rust
use std::fmt;

// 금액은 최소 화폐 단위(원/센트)의 정수로 — 부동소수점 오차 방지
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Money {
    cents: u64,
    currency: Currency,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Currency {
    Krw,
    Usd,
}

impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // 통화별로 표기를 분기 — 여기서도 match의 망라성이 보장된다
        match self.currency {
            Currency::Krw => write!(f, "{}원", self.cents),
            Currency::Usd => write!(f, "${}.{:02}", self.cents / 100, self.cents % 100),
        }
    }
}

// 실패 사유 — 또 다른 닫힌 집합
#[derive(Debug, Clone, PartialEq, Eq)]
enum FailureReason {
    InsufficientFunds,
    CardDeclined,
    Expired,
    NetworkError,
}

// 핵심: 각 variant가 "그 상태에서만 의미 있는 데이터"를 품는다
#[derive(Debug, Clone)]
enum PaymentStatus {
    Pending {
        amount: Money,
    },
    Authorized {
        amount: Money,
        auth_code: String,
    },
    Captured {
        amount: Money,
        auth_code: String,
        receipt_id: String,
    },
    Failed {
        amount: Money,
        reason: FailureReason,
    },
    Refunded {
        original: Money,
        refunded: Money,
        receipt_id: String,
    },
}

// 전이 실패도 값으로 표현 (Chapter 12의 Result 예고)
#[derive(Debug, PartialEq, Eq)]
enum TransitionError {
    InvalidTransition { from: &'static str, action: &'static str },
    RefundExceedsCapture,
}

impl PaymentStatus {
    fn label(&self) -> &'static str {
        match self {
            // `{ .. }`는 "필드는 무시하고 variant 종류만 본다"는 뜻
            PaymentStatus::Pending { .. } => "Pending",
            PaymentStatus::Authorized { .. } => "Authorized",
            PaymentStatus::Captured { .. } => "Captured",
            PaymentStatus::Failed { .. } => "Failed",
            PaymentStatus::Refunded { .. } => "Refunded",
        }
    }

    // 전이 함수는 self를 "소비(consume)"하고 새 상태를 반환한다.
    // 소유권을 가져가므로(Chapter 4), 이전 상태는 이 시점 이후 사용 불가 →
    // "낡은 상태를 실수로 다시 쓰는" 버그가 원천 차단된다.
    fn authorize(self, auth_code: String) -> Result<Self, TransitionError> {
        match self {
            // Pending에서만 authorize 가능. amount를 새 variant로 이동시킨다.
            PaymentStatus::Pending { amount } => {
                Ok(PaymentStatus::Authorized { amount, auth_code })
            }
            // 그 외 모든 상태에서의 authorize는 잘못된 전이
            other => Err(TransitionError::InvalidTransition {
                from: other.label(),
                action: "authorize",
            }),
        }
    }

    fn capture(self, receipt_id: String) -> Result<Self, TransitionError> {
        match self {
            PaymentStatus::Authorized { amount, auth_code } => {
                Ok(PaymentStatus::Captured { amount, auth_code, receipt_id })
            }
            other => Err(TransitionError::InvalidTransition {
                from: other.label(),
                action: "capture",
            }),
        }
    }

    fn refund(self, refund_amount: Money) -> Result<Self, TransitionError> {
        match self {
            // match guard로 "환불액 <= 매입액" 비즈니스 규칙을 패턴 단계에서 검사
            PaymentStatus::Captured { amount, receipt_id, .. }
                if refund_amount.cents <= amount.cents =>
            {
                Ok(PaymentStatus::Refunded {
                    original: amount,
                    refunded: refund_amount,
                    receipt_id,
                })
            }
            // Captured이지만 환불액이 초과하는 경우 — 가드가 거짓이면 여기로
            PaymentStatus::Captured { .. } => Err(TransitionError::RefundExceedsCapture),
            other => Err(TransitionError::InvalidTransition {
                from: other.label(),
                action: "refund",
            }),
        }
    }
}

fn main() {
    let krw = |n: u64| Money { cents: n, currency: Currency::Krw };

    // 정상 경로: Pending → Authorized → Captured
    let payment = PaymentStatus::Pending { amount: krw(50_000) };
    println!("[1] {}", payment.label());

    let authorized = payment.authorize("AUTH-XYZ".to_string()).unwrap();
    println!("[2] {}", authorized.label());

    let captured = authorized.capture("RCPT-001".to_string()).unwrap();
    println!("[3] {}", captured.label());

    // 잘못된 전이: 이미 Captured인데 다시 authorize → Err
    let bad = captured.clone().authorize("AUTH-2".to_string());
    println!("[4] 잘못된 전이 결과: {bad:?}");

    // 부분 환불 (가드 통과)
    let refunded = captured.refund(krw(20_000)).unwrap();
    if let PaymentStatus::Refunded { original, refunded, .. } = &refunded {
        println!("[5] 환불 완료: 원금 {original} 중 {refunded} 환불");
    }

    // 과다 환불 (가드 실패)
    let captured2 = PaymentStatus::Captured {
        amount: krw(10_000),
        auth_code: "A".to_string(),
        receipt_id: "R".to_string(),
    };
    println!("[6] 과다 환불: {:?}", captured2.refund(krw(99_999)));
}
```

실행 결과는 다음과 같습니다.

```text
[1] Pending
[2] Authorized
[3] Captured
[4] 잘못된 전이 결과: Err(InvalidTransition { from: "Captured", action: "authorize" })
[5] 환불 완료: 원금 50000원 중 20000원 환불
[6] 과다 환불: Err(RefundExceedsCapture)
```

이 설계의 미덕을 정리해 봅시다.

1. **불가능한 상태가 표현 불가능하다(Make Illegal States Unrepresentable).** `Pending` variant에는 `auth_code` 필드 자체가 없습니다. "승인 코드를 가진 대기 상태"라는 모순된 객체를 *애초에 만들 수 없습니다.* Python의 `Optional[str] = None` 필드로는 이 모순을 런타임에야 막을 수 있습니다.
2. **소유권 기반 전이.** `fn authorize(self, ...)`는 `&self`가 아니라 `self`를 받습니다. 즉 이전 상태의 소유권을 통째로 가져갑니다(Chapter 4의 Move 시맨틱). 전이가 끝나면 옛 상태는 *이동되어 사라지므로*, "이미 승인 처리한 객체를 코드 어딘가에서 또 승인하는" 실수가 컴파일러 단에서 막힙니다.
3. **잘못된 전이가 값이 된다.** 전이 실패를 예외로 던지지 않고 `Result<Self, TransitionError>`로 반환합니다. 호출자는 `?`나 `match`로 *반드시* 결과를 처리해야 합니다. 이건 Chapter 12·13의 본격적인 주제입니다.

### Typestate 맛보기: 잘못된 전이를 컴파일 에러로

위 설계는 잘못된 전이를 `Result::Err`라는 *런타임 값*으로 잡았습니다. 한 걸음 더 나아가, 잘못된 전이를 아예 **컴파일 에러**로 만드는 기법이 typestate 패턴입니다. 상태를 데이터가 아니라 *타입 파라미터*로 인코딩하는 것입니다.

```rust
use std::marker::PhantomData;

// 상태를 나타내는 영(zero)-크기 마커 타입들
struct Pending;
struct Authorized;
struct Captured;

// 상태가 타입 파라미터 State에 박혀 있다
struct Payment<State> {
    amount: u64,
    auth_code: Option<String>,
    _state: PhantomData<State>, // 실제 데이터는 없고 타입 정보만 운반
}

impl Payment<Pending> {
    fn new(amount: u64) -> Self {
        Payment { amount, auth_code: None, _state: PhantomData }
    }
    // Pending 상태에서만 authorize가 존재한다
    fn authorize(self, code: String) -> Payment<Authorized> {
        Payment { amount: self.amount, auth_code: Some(code), _state: PhantomData }
    }
}

impl Payment<Authorized> {
    // Authorized 상태에서만 capture가 존재한다
    fn capture(self) -> Payment<Captured> {
        Payment { amount: self.amount, auth_code: self.auth_code, _state: PhantomData }
    }
}

impl Payment<Captured> {
    fn receipt(&self) -> String {
        format!("captured {} won (auth {:?})", self.amount, self.auth_code)
    }
}

fn main() {
    let p = Payment::<Pending>::new(50_000);
    let p = p.authorize("AUTH-1".to_string());  // Payment<Authorized>
    let p = p.capture();                          // Payment<Captured>
    println!("{}", p.receipt());
}
```

여기서 `Payment<Pending>`에 대고 `capture()`를 호출하면 어떻게 될까요? 그 메서드는 `impl Payment<Authorized>`에만 존재합니다. 컴파일러는 단호히 거부합니다.

```text
error[E0599]: no method named `capture` found for struct `Payment<Pending>`
              in the current scope
  --> typestate_bad.rs:13:15
   |
 3 | struct Payment<S> { amount: u64, _s: PhantomData<S> }
   | ----------------- method `capture` not found for this struct
...
13 |     let _ = p.capture();
   |               ^^^^^^^ method not found in `Payment<Pending>`
   |
   = note: the method was found for `Payment<Authorized>`
```

"이 메서드는 `Payment<Authorized>`에는 있지만 `Payment<Pending>`에는 없다"고 친절히 알려줍니다. 잘못된 전이가 **타입 시스템 차원에서 존재하지 않는 메서드**가 된 것입니다. `PhantomData<State>`는 런타임에 0바이트를 차지하므로(Chapter 6의 수명 마커와 동일한 영-크기 타입), 이 안전성은 무비용(Zero-cost)입니다.

언제 무엇을 쓸까요? 상태가 *런타임 데이터에 따라 동적으로 결정*되면(예: DB에서 읽어온 결제의 현재 상태) enum + `Result` 방식이 맞습니다. 상태 전이가 *코드 흐름에서 정적으로 확정*되면(예: 빌더의 단계, 연결의 핸드셰이크) typestate가 더 강력합니다. 실무에서는 enum 방식이 압도적으로 흔하고, typestate는 라이브러리 API의 오용 방지에 핀포인트로 쓰입니다. (Chapter 11의 Builder 패턴에서 typestate의 또 다른 모습을 보게 됩니다.)

---

## 10.6 Under the Hood: Enum의 메모리 레이아웃과 니치 최적화

시니어 엔지니어라면 추상화 뒤의 바이트를 봐야 마음이 놓입니다. Rust enum이 메모리에서 정확히 어떻게 생겼는지 해부합시다. 모든 수치는 64비트 시스템에서 `std::mem::size_of`로 직접 측정한 값입니다.

### 태그 + 최대 variant: Tagged Union

C 개발자라면 이미 알고 있는 구조입니다. Rust enum은 본질적으로 **태그가 붙은 공용체(Tagged Union)**입니다.

```rust
enum Message {
    Quit,                    // 0 bytes payload
    Write(String),           // String = 24 bytes (ptr 8 + len 8 + cap 8)
    Move { x: i32, y: i32 }, // 8 bytes payload
}
```

`size_of::<Message>()`는 **24**입니다. 어떻게 나온 숫자일까요?

```text
Message (size = 24, align = 8)
┌──────────────┬──────────────────────────────────────────────┐
│ discriminant │ payload (가장 큰 variant 크기만큼 확보)       │
│  (tag)       │ ← Write(String)의 24바이트가 최대             │
└──────────────┴──────────────────────────────────────────────┘
```

- **payload 영역**은 *가장 큰 variant*를 담을 수 있어야 합니다. `Write(String)`이 24바이트로 최대이므로, payload는 최소 24바이트입니다.
- **discriminant(태그)**는 "지금 이게 몇 번 variant인가"를 가리키는 정수입니다. variant가 3개뿐이라 1바이트면 충분합니다.
- 그런데 전체가 25바이트(혹은 정렬 때문에 32바이트)가 아니라 정확히 24바이트입니다. 흔히 "태그가 `String` 내부의 정렬 패딩에 들어갔다"고 설명하지만, **이건 틀린 설명입니다.** `String`은 `ptr 8 + cap 8 + len 8 = 24`바이트로 *전부 8바이트 정렬에 딱 맞아 떨어져, 남는 정렬 패딩이 한 바이트도 없습니다.* 진짜 이유는 다음 절에서 다룰 **니치 최적화(Niche Optimization)**입니다 — `String` 내부 포인터는 `NonNull`이라 절대 0번지가 될 수 없고, 컴파일러는 그 "절대 등장하지 않는 비트 패턴(니치)"에 태그를 숨깁니다. `Option<&T>`가 8바이트인 것과 정확히 같은 메커니즘입니다. Rust는 enum 레이아웃을 *명시적으로 지정하지 않는 한* 자유롭게 재배치하여 크기를 최소화합니다. (C 호환이 필요하면 `#[repr(C)]`나 `#[repr(u8)]`로 고정할 수 있습니다.)

이 주장은 측정으로 검증할 수 있습니다. 같은 24바이트 payload를 가지되 *니치가 없는* 타입과 나란히 재보겠습니다.

```rust
use std::mem::size_of;

enum Message {
    Quit,
    Write(String),           // 24바이트, 내부 포인터에 니치 있음
    Move { x: i32, y: i32 },
}

// payload 크기는 동일(24바이트)하지만 [u64; 3]에는 니치가 없다
enum NoNiche {
    Quit,
    Write([u64; 3]),         // 24바이트, 모든 비트 패턴이 유효 → 니치 없음
    Move { x: i32, y: i32 },
}

// f64 변형만 있는 enum: f64는 모든 64비트 패턴이 유효(NaN 포함)하므로 니치 없음
enum Shape {
    Circle(f64),
    Rect(f64, f64),          // payload 16바이트
}

fn main() {
    println!("Message  = {}", size_of::<Message>());  // 24 (니치 사용)
    println!("NoNiche  = {}", size_of::<NoNiche>());   // 32 (태그 8 + payload 24)
    println!("Shape    = {}", size_of::<Shape>());     // 24 (태그 8 + payload 16, 니치 아님)
}
```

```text
Message  = 24
NoNiche  = 32
Shape    = 24
```

`Message`와 `NoNiche`는 payload 크기가 24바이트로 똑같은데도 `Message`는 24, `NoNiche`는 **32**바이트입니다. 차이를 만든 것은 오직 하나 — `String`에는 니치(`NonNull` 포인터의 0번지)가 있어 태그를 그 안에 숨겼고, `[u64; 3]`에는 니치가 없어 태그용 8바이트를 따로 떼어내야 했다는 점입니다. 만약 "정렬 패딩에 끼워 넣었다"가 사실이라면 `NoNiche`도 24바이트여야 하지만, 그렇지 않습니다. 정렬 패딩에 태그를 끼우는 일이 *전혀* 없는 건 아닙니다 — 가장 큰 variant가 실제로 남는 빈 바이트(trailing padding)를 가질 때는 태그가 그곳에 들어갑니다. 예컨대 variant 6개를 가진 enum의 최대 variant가 `{ x: u32, y: u8 }`(크기 8 = 4+1+3 패딩)이면 1바이트 태그가 그 3바이트 패딩에 흡수되어 enum 전체가 12가 아니라 **8바이트**입니다(직접 측정). 다만 `Write(String)`은 그런 패딩 경우가 아니라 — `String`에는 남는 패딩이 한 바이트도 없으므로 — *니치* 경우라는 것이 핵심입니다.

> 🚨 **혼동 주의: 24바이트라고 다 같은 24바이트가 아닙니다.** `Shape`(`Circle(f64)`/`Rect(f64,f64)`)도 24바이트지만, 여기엔 니치가 *없습니다.* `f64`는 모든 64비트 패턴이 유효한 값(시그널링/콰이엇 NaN 포함)이라 빈틈이 없기 때문입니다. `Shape`의 24바이트는 "태그 8(정렬 때문에 1이 아니라 8로 패딩) + 가장 큰 payload 16 = 24"라는, *니치와 무관한* 평범한 태그드 유니언입니다. 즉 똑같이 24바이트여도 `Write(String)`의 24는 "니치로 태그를 흡수해 32→24로 줄인 결과"이고, `Rect(f64, f64)`의 24는 "태그를 따로 둔 태그드 유니언이 원래 24"인 것입니다. 메모리 크기만 보고 메커니즘을 단정하지 마십시오 — `size_of`는 같아도 그 안의 구조는 다릅니다.

핵심 통찰 하나: **enum의 크기는 가장 큰 variant에 의해 결정됩니다.** `Quit`처럼 데이터 없는 variant도 24바이트를 차지합니다. 그래서 한 variant만 거대하면(예: 큰 배열을 품으면), 작은 variant들도 그 크기를 짊어집니다. 이럴 땐 큰 데이터를 `Box`로 감싸 힙으로 보내 enum 본체를 작게 유지하는 것이 관용구입니다(`clippy::large_enum_variant`가 이를 경고합니다).

데이터 없는 단순 enum은 어떨까요?

```rust
enum Status { Pending, Authorized, Captured, Failed, Refunded }
```

`size_of::<Status>()`는 **1**입니다. payload가 없으니 태그(1바이트)만 남습니다. C의 enum과 동일한 효율입니다 — 그러면서도 망라성 검사라는 안전망은 공짜로 따라옵니다.

### 니치 최적화(Niche Optimization): `Option<&T>`가 8바이트인 마법

이제 Rust 레이아웃의 진짜 보석입니다. 다음 수치들을 보십시오. (모두 직접 측정한 값입니다.)

| 타입 | 크기(bytes) | 비고 |
| :--- | :--- | :--- |
| `&i32` | 8 | 일반 참조 = 포인터 1개 |
| `Option<&i32>` | **8** | 🎯 태그가 사라졌다 |
| `Box<i32>` | 8 | |
| `Option<Box<i32>>` | **8** | 🎯 역시 태그가 없다 |
| `u32` | 4 | |
| `Option<u32>` | 8 | 태그(+패딩)가 붙음 |
| `NonZeroU32` | 4 | 0이 될 수 없는 u32 |
| `Option<NonZeroU32>` | **4** | 🎯 0을 None으로 재활용 |
| `bool` | 1 | 유효값은 0, 1뿐 |
| `Option<bool>` | **1** | 🎯 2를 None으로 재활용 |

`Option<T>`는 그 자체로 enum입니다. `enum Option<T> { None, Some(T) }`. 순진하게 구현하면 `Some/None`을 구분할 태그 1바이트 + `T` 크기 + 정렬 패딩이 필요합니다. 실제로 `Option<u32>`는 그렇게 8바이트(태그 4 + u32 4)를 씁니다.

그런데 `Option<&i32>`는 어떻게 참조와 똑같은 8바이트일까요? 비밀은 **참조는 절대 null(0번지)일 수 없다**는 Rust의 불변식에 있습니다. Rust 참조의 비트 패턴 중 "0"이라는 값은 어떤 유효한 참조로도 쓰이지 않는 **빈틈(niche)**입니다. 컴파일러는 이 빈틈을 가로채서 **"비트 패턴이 0이면 `None`, 아니면 `Some(그 포인터)`"**로 약속합니다. 별도 태그가 필요 없어진 것입니다.

> **Under the Hood: 니치(Niche)란 무엇인가**
>
> "니치"는 어떤 타입의 비트 표현 중 **유효하지 않은(절대 등장하지 않는) 값의 영역**을 뜻합니다.
>
> - `&T`, `Box<T>`, `Rc<T>` 같은 포인터류: 0(null)은 유효하지 않다 → 0을 `None`으로.
> - `NonZeroU32`: 정의상 0이 될 수 없다 → 0을 `None`으로. 그래서 `Option<NonZeroU32>`가 u32와 같은 4바이트.
> - `bool`: 메모리상 0(false), 1(true)만 유효하고 2~255는 빈틈 → 2를 `None`으로. `Option<bool>`이 1바이트.
> - `char`: 유효 코드포인트 범위 밖의 비트 패턴이 빈틈.
>
> 이 최적화 덕분에 `Option<&T>`를 함수 인자로 넘겨도 raw 포인터를 넘기는 것과 메모리·성능이 **완전히 동일**합니다. C의 nullable 포인터(`const int*`, null 가능)와 바이트 단위로 같으면서, 사용하는 쪽은 `match`로 null 처리를 *강제*당합니다. "Rust의 안전성은 공짜"라는 말의 가장 구체적인 증거가 바로 이 8바이트입니다.
>
> 더 놀라운 것: `Option<Option<bool>>`도 **1바이트**입니다. bool의 빈틈(2,3,...)에 두 단계의 `None`을 모두 끼워 넣을 만큼 여유가 있기 때문입니다. 중첩 enum이 태그를 계단처럼 쌓지 않습니다.

### C의 union / tagged union과의 비교

C에서 합 타입을 흉내 내려면 직접 태그와 union을 손으로 묶어야 합니다.

```c
// C: tagged union — 태그와 데이터의 정합성은 전적으로 프로그래머 책임
enum Tag { TAG_QUIT, TAG_WRITE, TAG_MOVE };

struct Message {
    enum Tag tag;
    union {
        struct { char *text; }      write;   // TAG_WRITE일 때만 유효
        struct { int x; int y; }    move;    // TAG_MOVE일 때만 유효
    } data;
};

void process(struct Message *msg) {
    switch (msg->tag) {
        case TAG_WRITE:
            // 🚨 만약 tag는 WRITE인데 실수로 data.move를 읽으면? 컴파일러는 막지 않는다.
            printf("%s\n", msg->data.write.text);
            break;
        // case TAG_MOVE를 빼먹어도 C 컴파일러는 (보통) 경고만 한다
    }
}
```

C의 이 패턴에는 두 가지 구조적 위험이 있습니다. 첫째, **태그와 union의 정합성을 컴파일러가 보장하지 않습니다.** `tag`가 `TAG_WRITE`인데 `data.move.x`를 읽는 것을 막을 방법이 없습니다(undefined behavior로 가는 지름길). 둘째, `switch`의 망라성을 강제하지 않습니다(`-Wswitch-enum`을 켜도 경고일 뿐, `default`가 있으면 침묵합니다).

Rust enum은 정확히 이 C tagged union의 메모리 효율(태그 + union)을 그대로 가지면서, **태그와 payload를 분리해 접근할 방법 자체를 언어에서 봉쇄**합니다. `Message::Write(text)`에서 `text`는 오직 `match`가 태그를 `Write`로 확인한 *후에만*, 그 arm 안에서만 손에 쥘 수 있습니다. C가 30년간 메모리로 풀던 문제를, Rust는 같은 메모리 비용에 타입 안전성을 얹어 푼 것입니다.

---

## 10.7 Best Practices & 안티패턴

실무 체크리스트입니다. clippy 경고와 연계해 정리합니다.

**이렇게 하라 (Do)**

- ✅ **상태/종류가 닫혀 있으면 enum을 첫 번째 선택지로.** "이 도메인의 종류를 내가 다 안다"면 enum이다. Trait Object는 *정말 외부 확장이 필요할 때만* 꺼낸다.
- ✅ **불가능한 상태를 표현 불가능하게 만들어라.** "이 필드는 X 상태에서만 의미 있다"면, 그 필드를 X variant 안에 넣어라. `Optional` 필드 + 런타임 검사로 흩뿌리지 마라.
- ✅ **`match`는 명시적으로, 모든 variant를 나열하라.** 상태마다 다른 처리가 필요할 수 있는 곳에서는 `_ =>`를 피하고 variant를 다 쓴다. 미래의 enum 변경 시 컴파일러가 모든 수정 지점을 짚어주게 하라.
- ✅ **전이 함수는 `self`(소유권)를 소비하게 설계하라.** `fn next(self) -> NextState` 형태. 낡은 상태의 재사용을 컴파일러가 막는다.
- ✅ **거대한 variant는 `Box`로 감싸라.** 한 variant만 큰 데이터를 품으면 enum 전체가 비대해진다. clippy의 `clippy::large_enum_variant`(기본 임계값 200바이트)를 신뢰하라.
- ✅ **null 가능성·실패 가능성은 `Option`/`Result`로.** 이들 자체가 enum이며 니치 최적화로 무비용에 가깝다(Chapter 12에서 본격 다룸).
- ✅ **enum 직렬화는 `serde`에 맡겨라.** Python에서 상태 머신을 DB나 JSON으로 영속화할 때 직접 문자열 매핑을 짜던 코드는, Rust에서 `#[derive(Serialize, Deserialize)]` 한 줄로 끝난다. 데이터 품은 variant는 serde가 태그 방식(외부/내부/인접 태깅)까지 골라준다. variant↔문자열 변환만 필요하면 `strum` 크레이트의 `#[derive(Display, EnumString)]`이 Python `enum`의 `.value`/`__str__`을 컴파일 타임 코드 생성으로 대체한다. (직렬화의 본격적인 내용은 Chapter 7의 serde 절을 참고하라.)

**이건 피하라 (Don't)**

- ❌ **습관적인 `_ => unreachable!()` 또는 `_ => {}`.** 망라성 안전망을 스스로 버리는 행위. clippy의 `clippy::wildcard_enum_match_arm`을 켜서 와일드카드 사용을 의식적으로 감시하라.
- ❌ **enum이면 충분한데 Trait Object를 쓰기.** 닫힌 집합에 `Box<dyn>`을 쓰면 불필요한 힙 할당 + V-Table 추적 + 캐시 미스를 자초한다.
- ❌ **`match` 대신 `if let` 사슬을 길게 늘어놓기.** `if let A = x { } else if let B = x { }`는 망라성 검사를 받지 못한다. 두 갈래 이상이면 `match`를 써라.
- ❌ **enum에 `i32` 같은 원시 태그를 직접 붙여 상태 표현.** `#[repr(i32)]`로 직렬화 호환이 *반드시* 필요한 경우가 아니면, variant 이름 자체가 의미를 담게 하라.

> 💡 **clippy 한 줄 팁.** `#![warn(clippy::wildcard_enum_match_arm)]`를 크레이트에 켜두면, enum에 대해 `_`를 쓸 때마다 경고가 뜹니다. "정말 이 와일드카드가 의도된 것인지"를 매번 검토하게 만들어, 10.3에서 본 함정을 구조적으로 예방합니다.

---

## 10.8 흔한 함정 — Python 습관이 만드는 실수

### 함정 1: `_ =>`로 망라성을 우회하는 습관

Python의 `if-elif-else`에서 `else`는 거의 반사적으로 붙이는 안전망이었습니다. 그 습관을 그대로 가져와 `match`에 `_ => ...`를 붙이면, enum의 가장 큰 장점을 무력화합니다.

```rust
// ❌ Python의 else 습관
fn handle(s: &PaymentStatus) -> u8 {
    match s {
        PaymentStatus::Pending { .. } => 1,
        PaymentStatus::Captured { .. } => 2,
        _ => 0,  // 나머지를 다 삼킴 — 나중에 새 variant 추가해도 침묵
    }
}

// ✅ 명시적 나열 — 새 variant 추가 시 컴파일러가 알려준다
fn handle(s: &PaymentStatus) -> u8 {
    match s {
        PaymentStatus::Pending { .. } => 1,
        PaymentStatus::Authorized { .. } => 0,
        PaymentStatus::Captured { .. } => 2,
        PaymentStatus::Failed { .. } => 0,
        PaymentStatus::Refunded { .. } => 0,
    }
}
```

"여러 variant를 같은 값으로 처리하고 싶다"면 `_` 대신 **or-패턴**으로 *이름을 명시*하십시오: `Authorized { .. } | Failed { .. } | Refunded { .. } => 0`. 이러면 새 variant가 추가됐을 때 여전히 컴파일 에러가 나서, "이 새 상태도 0으로 묶을 건지"를 의식적으로 결정하게 됩니다.

### 함정 2: `isinstance()`를 흉내 내려는 시도

Python 개발자는 `match`를 받고서도 "그래서 이 값이 무슨 타입인지 검사하는 함수는 어디 있지?"를 찾곤 합니다. Rust에는 그런 게 없습니다 — *그리고 그게 정답입니다.* "타입을 검사한 뒤 캐스팅"하는 2단계 대신, `match`가 "검사와 추출을 한 번에" 합니다.

```rust
// ❌ Python식 사고: "먼저 종류를 확인하고, 그 다음 데이터를 꺼내자"
//    Rust에는 이런 분리된 검사가 없다.

// ✅ Rust식: match가 검사와 구조 분해를 동시에 한다
match msg {
    Message::Write(text) => { /* text를 즉시 손에 쥔다 */ }
    Message::Move { x, y } => { /* x, y를 즉시 손에 쥔다 */ }
    Message::Quit => { /* 데이터 없음 */ }
}
```

단일 variant만 관심 있다면 `if let`이 `isinstance` 한 줄짜리 대응입니다.

```rust
// Python: if isinstance(msg, WriteMessage): print(msg.text)
if let Message::Write(text) = &msg {
    println!("{text}");
}
```

### 함정 3: 전이 시 옛 상태를 다시 쓰려다 빌림 검사기에 막히기

전이 함수가 `self`를 소비하도록 설계했다면(권장), 전이 후 옛 변수를 또 쓰면 막힙니다. 이건 버그가 아니라 *의도된 방어*입니다.

```rust
let payment = PaymentStatus::Pending { amount: krw(1000) };
let authorized = payment.authorize("A".into()).unwrap();
println!("{}", payment.label());  // 🚨 컴파일 에러!
```

```text
error[E0382]: borrow of moved value: `payment`
  |
  |  let authorized = payment.authorize("A".into()).unwrap();
  |                   ------- value moved here
  |  println!("{}", payment.label());
  |                 ^^^^^^^ value borrowed here after move
```

`authorize`가 `payment`의 소유권을 가져갔으므로(Chapter 4의 Move), `payment`는 더 이상 유효하지 않습니다. 상태 머신에서 이건 정확히 우리가 원하는 바입니다 — "이미 다음 상태로 넘어간 결제를 옛 상태로 또 조작"하는 것을 컴파일러가 봉쇄합니다. Python이라면 같은 객체를 두 번 `authorize`해도 아무도 막지 않아, 중복 승인 같은 미묘한 버그가 생깁니다.

### 함정 4: variant 크기를 무시한 거대 enum

Python에서는 객체 크기를 신경 쓸 일이 거의 없었습니다(모두 힙에 있고 포인터로 다루니까). Rust enum은 가장 큰 variant가 전체 크기를 결정하므로, 한 variant만 크면 작은 variant들까지 그 무게를 짊어집니다.

```rust
enum Packet {
    Heartbeat,                  // 0 bytes
    Data([u8; 4096]),           // 4096 bytes — 이게 전체 크기를 결정한다
}
// size_of::<Packet>() ≈ 4097+ — Heartbeat도 4KB를 차지한다!

// ✅ 큰 payload는 Box로 힙에 보내 enum 본체를 슬림하게
enum Packet {
    Heartbeat,
    Data(Box<[u8; 4096]>),      // 이제 Data variant는 포인터 8바이트
}
// size_of::<Packet>() ≈ 8~16
```

`Vec<Packet>`처럼 enum을 대량으로 담을 때 이 차이는 메모리 사용량과 캐시 효율에 직결됩니다. clippy가 `large_enum_variant`로 경고해 주니, 그 경고를 무시하지 마십시오.

### 함정 5: `#[non_exhaustive]` enum이 강제하는 `_` arm

10.3에서 우리는 "enum에 `_`를 쓰면 망라성 안전망을 스스로 버린다"고 배웠습니다. 그런데 *남의 크레이트*가 제공하는 enum을 매칭할 때는, 정반대로 **컴파일러가 `_`를 강요하는** 상황을 만납니다. 이 모순처럼 보이는 동작의 정체가 `#[non_exhaustive]` 속성입니다.

라이브러리 작성자는 "이 enum에 앞으로 variant를 더 추가할 수 있으니, 소비자 코드는 그 변경을 *호환성 깨짐 없이* 견뎌야 한다"고 선언하고 싶을 때가 있습니다. 대표적으로 에러 enum이 그렇습니다 — 새 에러 종류가 추가되어도 다운스트림이 재컴파일만 하면 깨지지 않게 하고 싶죠. 이때 enum에 `#[non_exhaustive]`를 붙입니다.

```rust
// upstream_crate (라이브러리 측)
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Timeout,
    RateLimited,
}
```

이제 *다른 크레이트*의 소비자가 이 enum을 매칭하면, 현재 존재하는 variant를 **하나도 빠짐없이 나열했더라도** 컴파일 에러가 납니다.

```rust
// consumer_crate (소비자 측)
use upstream_crate::ApiError;

fn describe(e: &ApiError) -> &str {
    match e {
        ApiError::NotFound => "찾을 수 없음",
        ApiError::Timeout => "시간 초과",
        ApiError::RateLimited => "요청 제한",
        // 🚨 모든 variant를 다 적었는데도 컴파일 에러!
    }
}
```

`cargo build`의 실제 출력입니다 (직접 측정).

```text
error[E0004]: non-exhaustive patterns: `&_` not covered
 --> src/main.rs:5:11
  |
5 |     match e {
  |           ^ pattern `&_` not covered
  |
note: `ApiError` defined here
 --> .../upstream_crate/src/lib.rs:2:1
  |
2 | pub enum ApiError {
  | ^^^^^^^^^^^^^^^^^
  = note: the matched value is of type `&ApiError`
  = note: `ApiError` is marked as non-exhaustive, so a wildcard `_` is necessary
          to match exhaustively
help: ensure that all possible cases are being handled by adding a match arm
      with a wildcard pattern or an explicit pattern as shown
  |
8 ~         ApiError::RateLimited => "요청 제한",
9 ~         &_ => todo!(),
  |
```

핵심은 `note: ... is marked as non-exhaustive, so a wildcard _ is necessary` 한 줄입니다. `#[non_exhaustive]`는 "이 enum의 우주는 *외부에서 볼 때* 닫혀 있지 않다"고 선언하는 것이고, 그래서 소비자는 미래에 추가될지 모를 variant를 위해 `_ => ...` arm을 *의무적으로* 둬야 합니다. 해법은 명확합니다.

```rust
fn describe(e: &ApiError) -> &str {
    match e {
        ApiError::NotFound => "찾을 수 없음",
        ApiError::Timeout => "시간 초과",
        ApiError::RateLimited => "요청 제한",
        _ => "알 수 없는 오류",   // ✅ non_exhaustive가 요구하는 catch-all
    }
}
```

두 가지 비대칭을 기억하십시오. 첫째, **이 강제는 크레이트 경계를 넘을 때만 작동합니다.** `#[non_exhaustive]` enum을 *정의한 그 크레이트 안에서는* 평소처럼 모든 variant를 명시해야 하고 `_`는 불필요합니다 — 같은 크레이트는 enum의 전체 우주를 알기 때문입니다. 둘째, 그래서 `_`가 여기서는 "안전망을 버리는 행위"가 아니라 "버전 호환성을 사는 비용"입니다. `std`의 여러 에러 타입(`std::io::ErrorKind` 등)과 잘 설계된 크레이트의 에러 enum이 이 속성을 답니다. 당신이 *라이브러리를 만든다면*, 공개 enum 특히 에러 enum에는 `#[non_exhaustive]`를 붙여 다운스트림을 미래의 깨짐으로부터 보호하는 것을 고려하십시오. 반대로 당신이 *소비자라면*, 외부 enum에 강제된 `_` arm을 보고 "이건 망라성 안전망 포기인가?"라고 당황하지 말고, 의미를 정확히 이해하고 catch-all을 두십시오.

---

## 10.9 벤치마크: 100만 요소 순회에서 정적 vs 동적 디스패치

10.4에서 "동적 디스패치는 정적보다 느리다"고 말했습니다. 시니어는 형용사를 믿지 않습니다 — 숫자를 봐야 합니다. `criterion`으로 직접 재 봅시다. 도형 100만 개를 `Vec`에 담고, 전체를 순회하며 `area()`를 합산하는 핫루프입니다. 한쪽은 `Vec<Shape>`(enum, 정적 디스패치 + 값을 연속 저장), 다른 쪽은 `Vec<Box<dyn Area>>`(트레이트 객체, 동적 디스패치 + 힙 산재 포인터)입니다.

```toml
# Cargo.toml
[dev-dependencies]
criterion = "0.8"

[[bench]]
name = "dispatch"
harness = false

[profile.release]
codegen-units = 1
lto = true
```

```rust
// benches/dispatch.rs
use criterion::{criterion_group, criterion_main, Criterion};
use std::hint::black_box;

// 정적 디스패치: enum + match
enum Shape {
    Circle(f64),
    Rect(f64, f64),
    Triangle(f64, f64),
}
impl Shape {
    #[inline]
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => std::f64::consts::PI * r * r,
            Shape::Rect(w, h) => w * h,
            Shape::Triangle(b, h) => 0.5 * b * h,
        }
    }
}

// 동적 디스패치: trait object
trait Area {
    fn area(&self) -> f64;
}
struct Circle(f64);
struct Rect(f64, f64);
struct Triangle(f64, f64);
impl Area for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.0 * self.0 }
}
impl Area for Rect {
    fn area(&self) -> f64 { self.0 * self.1 }
}
impl Area for Triangle {
    fn area(&self) -> f64 { 0.5 * self.0 * self.1 }
}

const N: usize = 1_000_000;

fn build_enum() -> Vec<Shape> {
    (0..N).map(|i| match i % 3 {
        0 => Shape::Circle(i as f64 * 0.001 + 1.0),
        1 => Shape::Rect(i as f64 * 0.001 + 1.0, 2.0),
        _ => Shape::Triangle(i as f64 * 0.001 + 1.0, 3.0),
    }).collect()
}
fn build_dyn() -> Vec<Box<dyn Area>> {
    (0..N).map(|i| -> Box<dyn Area> { match i % 3 {
        0 => Box::new(Circle(i as f64 * 0.001 + 1.0)),
        1 => Box::new(Rect(i as f64 * 0.001 + 1.0, 2.0)),
        _ => Box::new(Triangle(i as f64 * 0.001 + 1.0, 3.0)),
    }}).collect()
}

fn bench(c: &mut Criterion) {
    let enums = build_enum();
    let dyns = build_dyn();
    let mut g = c.benchmark_group("sum_area_1M");
    g.bench_function("enum_static", |b| b.iter(|| {
        let mut acc = 0.0_f64;
        for s in black_box(&enums) { acc += s.area(); }
        black_box(acc)
    }));
    g.bench_function("dyn_dynamic", |b| b.iter(|| {
        let mut acc = 0.0_f64;
        for s in black_box(&dyns) { acc += s.area(); }
        black_box(acc)
    }));
    g.finish();
}

criterion_group!(benches, bench);
criterion_main!(benches);
```

`cargo bench`의 측정 결과입니다 (Apple Silicon, `--release` + LTO, criterion 100 샘플의 중앙값. 직접 측정). 시간은 예전 `criterion` 0.5 하니스의 기록이고 다시 재지 않았습니다. 위 하니스는 `criterion` 0.8과 `std::hint::black_box`로 바꿨고, 2026-10-22에 rustc 1.99.1로 `cargo check --benches`만 통과했습니다.

| 방식 | 컬렉션 타입 | 100만 요소 순회 시간 | 상대 비용 |
| :--- | :--- | ---: | ---: |
| 정적 디스패치 | `Vec<Shape>` (enum) | **약 703 µs** | 1.00× (기준) |
| 동적 디스패치 | `Vec<Box<dyn Area>>` | **약 811 µs** | **약 1.15×** |

동적 디스패치가 약 **15% 느립니다.** 이 격차의 정체를 분해해 봅시다.

1. **간접 호출 + 인라이닝 차단.** enum 쪽 `area()`는 `#[inline]`이 붙어 있고 호출 대상이 컴파일 타임에 확정되므로, 컴파일러가 `match`를 루프 안으로 인라인하고 산술을 벡터화할 여지가 큽니다. `dyn Area` 쪽은 매 요소마다 vtable 포인터를 읽어 함수 주소로 점프하는 간접 호출이라(10.4의 fat pointer 박스 참고), 인라이닝이 원천 차단됩니다.
2. **캐시 지역성.** `Vec<Shape>`는 24바이트짜리 값들이 힙에 *연속*으로 놓여, 순차 접근이 CPU 프리페처와 궁합이 좋습니다. `Vec<Box<dyn Area>>`는 16바이트 fat 포인터들만 연속이고, 실제 객체는 `Box::new`가 따로 할당한 *힙 여기저기*에 흩어져 있어 포인터를 따라갈 때마다 캐시 미스 위험이 있습니다.

> 🚨 **15%를 과대·과소 해석하지 마십시오.** 이 워크로드는 `area()` 본문이 산술 몇 번뿐이라 호출 오버헤드 비중이 상대적으로 큽니다. 메서드 본문이 무거우면(예: 수백 줄 작업) 디스패치 비용의 *상대* 비중은 줄어 격차가 좁혀지고, 반대로 본문이 더 가볍고 요소 수가 더 많으면 인라이닝·벡터화 차단 효과로 격차가 2배 이상 벌어지기도 합니다. 핵심은 절대 수치가 아니라 방향입니다 — **닫힌 집합인데 습관적으로 `Box<dyn>`을 쓰면 공짜로 성능을 흘린다.** 그리고 이 둘 모두 Python의 메서드 디스패치(매번 `__dict__`/MRO 해시 조회 + `PyObject` 박싱)보다는 수십 배 빠릅니다. enum이냐 dyn이냐는 "Python 대비 50배냐 45배냐"가 아니라 *확장성을 여느냐 닫느냐*의 설계 결정이며(10.4), 측정 가능한 핫패스가 아니라면 가독성·확장성을 우선하십시오.

---

## 연습문제

손으로 풀어 보십시오(Rustlings 스타일). 각 문제 아래 "정답 스케치"에는 핵심 아이디어와 *자기 검증용 컴파일러 에러코드 또는 측정값*이 있습니다. 먼저 직접 작성해 `rustc`/`cargo`로 돌려 보고, 막히면 스케치를 펼치십시오.

### 문제 1 — 재귀적 ADT 모델링 (Sum + Product의 조합)

간이 JSON 값을 표현하는 enum `Json`을 정의하십시오. variant는 `Null`, `Bool(bool)`, `Number(f64)`, `Str(String)`, `Array(Vec<Json>)`, `Object(BTreeMap<String, Json>)`입니다. 그리고 값의 중첩 깊이를 반환하는 `fn depth(&self) -> usize`를 `match`로 구현하십시오(스칼라는 1, 컨테이너는 `1 + 자식들의 최대 깊이`). `_` 와일드카드를 쓰지 말고 모든 variant를 명시하십시오.

**정답 스케치.** 합 타입(enum) 안에 자기 자신을 `Vec`/`BTreeMap`을 통해 재귀적으로 품는 것이 핵심입니다(`Vec`/`BTreeMap`이 힙 간접 참조를 제공하므로 `Box` 없이도 크기가 무한해지지 않습니다). 스칼라 4종은 or-패턴으로 `Json::Null | Json::Bool(_) | Json::Number(_) | Json::Str(_) => 1`로 묶고, 컨테이너는 `1 + items.iter().map(Json::depth).max().unwrap_or(0)`. 자기 검증: 아래 값의 `depth()`는 **4**입니다(`Object → Array → Array → Bool`).

```rust
// {"name":"ada","tags":[1,[true]]}  → depth == 4
let mut obj = BTreeMap::new();
obj.insert("name".to_string(), Json::Str("ada".to_string()));
obj.insert("tags".to_string(),
    Json::Array(vec![Json::Number(1.0), Json::Array(vec![Json::Bool(true)])]));
assert_eq!(Json::Object(obj).depth(), 4);   // ✅ 측정값 4
```

만약 `Array`나 `Object` arm을 빼먹으면 `error[E0004]: non-exhaustive patterns`로 컴파일러가 정확히 짚어 줍니다.

### 문제 2 — `match`의 정밀 도구 (가드 + `@` 바인딩 + or-패턴)

`enum Token { Num(i64), Op(char), Ident(String) }`를 분류하는 `fn classify(t: &Token) -> String`를 작성하십시오. 규칙: ① 0~9 사이의 한 자리 수는 `"small digit N"`(값을 캡처해 출력), ② 음수는 `"negative N"`, ③ 그 외 정수는 `"number N"`, ④ `+ - * /` 중 하나면 `"arithmetic op"`, ⑤ 다른 연산자는 `"other op C"`, ⑥ 식별자는 `"ident S"`.

**정답 스케치.** ①은 `@` 바인딩과 범위를 결합: `Token::Num(n @ 0..=9) => format!("small digit {n}")`. ②는 가드: `Token::Num(n) if *n < 0 => ...`(참조 매칭이라 `*n`). 순서가 중요합니다 — 좁은 패턴(범위, 가드)을 먼저, 포괄 패턴(`Token::Num(n)`)을 나중에 둬야 합니다. ④는 or-패턴: `Token::Op('+' | '-' | '*' | '/') => "arithmetic op".to_string()`. 자기 검증:

```rust
assert_eq!(classify(&Token::Num(7)),  "small digit 7");   // ✅
assert_eq!(classify(&Token::Num(-3)), "negative -3");      // ✅
assert_eq!(classify(&Token::Op('+')), "arithmetic op");    // ✅
assert_eq!(classify(&Token::Op('%')), "other op %");       // ✅
```

좁은 arm을 포괄 arm 뒤에 두면 `warning: unreachable pattern`이 뜹니다 — 그 경고가 순서가 틀렸다는 신호입니다.

### 문제 3 — 소유권 소비형 상태 머신

TCP 비슷한 연결을 `enum Conn { Closed, Listen, Established { peer: String } }`로 모델링하고, 전이 메서드를 `self`를 *소비*하도록(10.5의 결제 머신과 동일한 스타일) 구현하십시오: `listen(self)`은 `Closed`에서만 `Listen`으로, `accept(self, peer: String)`은 `Listen`에서만 `Established`로, 잘못된 전이는 `Result<Self, ConnError>`의 `Err`로 반환합니다. `close(self) -> Self`는 어디서든 `Closed`로.

**정답 스케치.** 정상 전이는 해당 variant를 매칭해 데이터를 새 variant로 *이동*시키고(`Conn::Listen => Ok(Conn::Established { peer })`), 그 외는 `other => Err(ConnError::InvalidTransition { from: other.label(), action: "..." })`. 핵심 안전성: 전이가 `self`를 소비하므로, 전이 후 옛 변수를 다시 쓰면 `error[E0382]: borrow of moved value`가 납니다(10.8 함정 3과 동일). 자기 검증:

```rust
let c = Conn::Closed.listen().unwrap().accept("10.0.0.1".to_string()).unwrap();
assert_eq!(c, Conn::Established { peer: "10.0.0.1".to_string() });   // ✅
let bad = Conn::Established { peer: "x".to_string() }.listen();
assert_eq!(bad, Err(ConnError::InvalidTransition { from: "Established", action: "listen" })); // ✅
```

(`PartialEq, Eq`를 `derive`해야 `assert_eq!`가 됩니다.)

### 문제 4 — 같은 도메인, 두 디스패치 (정적 vs 동적)

원과 정사각형의 넓이 합을 구하는 코드를, **두 방식으로** 작성하고 메모리 크기를 비교하십시오. (a) `enum ShapeE { Circle(f64), Square(f64) }` + `match`로 정적 디스패치, (b) `trait Area` + `Vec<Box<dyn Area>>`로 동적 디스패치. 두 방식의 합계가 같음을 `assert!`로 확인하고, `size_of::<ShapeE>()`와 `size_of::<Box<dyn Area>>()`를 출력하십시오.

**정답 스케치.** (a)는 10.4 길 1, (b)는 길 2를 그대로 따릅니다. 자기 검증 측정값:

```text
size_of ShapeE        = 16   // 태그 8 + 가장 큰 payload 8(f64) = 16, 니치 없음
size_of Box<dyn Area> = 16   // fat pointer: data ptr 8 + vtable ptr 8
```

두 숫자가 우연히 같아 보이지만 의미는 정반대입니다 — `ShapeE`의 16바이트는 *값 자체*를 담고 `Vec`에 연속 저장되며, `Box<dyn Area>`의 16바이트는 *힙 어딘가의 객체를 가리키는 포인터 쌍*일 뿐입니다(10.4 fat pointer 박스). 한 걸음 더: 100만 개로 늘려 10.9의 벤치마크를 직접 돌려, 어느 쪽이 빠른지 *측정*해 보십시오. 정적 쪽이 대략 1.1~1.2배 빠르게 나오면 정상입니다.

---

## 맺음말 — 닫힌 집합에서 행동의 조립으로

이번 챕터에서 우리는 Python의 `Union`·`isinstance()`·`Enum`이 사람에게 떠넘기던 "망라성"의 책임을, Rust가 어떻게 타입 시스템과 컴파일러에게 넘겼는지 보았습니다. enum은 합 타입(Sum Type)으로서 각 상태에 데이터를 묶고, `match`는 모든 경우의 처리를 강제하며, `Option`·`Result`조차 같은 enum 메커니즘 위에서 니치 최적화로 무비용 안전성을 제공합니다. 다형성에서는 닫힌 집합(Enum, 정적)과 열린 집합(Trait Object, 동적)이라는 두 길을 *비용을 의식하며* 선택할 수 있게 되었고, typestate로는 잘못된 전이를 메서드의 부재 — 즉 컴파일 에러 — 로까지 끌어올렸습니다.

이로써 Part 3 "Python 객체지향의 해체"의 데이터 모델링 축이 거의 완성되었습니다. struct(곱 타입, Ch7), 컴포지션(Ch8), 트레이트로서의 행동(Ch9), 그리고 enum(합 타입)과 다형성(Ch10)까지. 남은 한 조각은 *조립*입니다.

다음 Chapter 11에서는 OOP의 마지막 유산 두 가지를 다룹니다. 첫째, Django의 `LoginRequiredMixin` 같은 **다중 상속 기반 믹스인(Mixin)**을 Trait의 기본 메서드(Default Method)로 어떻게 깔끔히 대체하는가. 둘째, 인자 수십 개짜리 `__init__`와 `*args`/`**kwargs`의 혼돈을, 컴파일 타임에 필수 필드 누락까지 잡아내는 **빌더 패턴(Builder Pattern)** — 그리고 이번 챕터에서 맛본 typestate를 빌더에 적용해 "불완전한 객체의 build()"를 컴파일 에러로 막는 기법 — 으로 어떻게 우아하게 푸는가. 닫힌 집합을 모델링하는 법을 익혔으니, 이제 그것들을 *안전하게 조립*할 차례입니다.

---

## References

- https://doc.rust-lang.org/book/ch06-00-enums.html
- https://doc.rust-lang.org/book/ch06-02-match.html
- https://doc.rust-lang.org/book/ch18-02-trait-objects.html
- https://doc.rust-lang.org/rust-by-example/custom_types/enum.html
- https://doc.rust-lang.org/std/option/enum.Option.html
