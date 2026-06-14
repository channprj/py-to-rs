# Chapter 7. Struct와 데이터 모델링 (`dataclass`와 `Pydantic`의 대체)

`@dataclass` 데코레이터 한 줄로 모델을 선언하고, `BaseModel`을 상속받아 FastAPI 엔드포인트에 그대로 꽂아 넣어본 적이 있을 겁니다. 파이썬에서 "데이터를 담는 그릇"을 만드는 일은 지난 10년간 점점 더 우아해졌습니다. 손으로 쓰던 `__init__`은 `dataclasses`가 생성해 주고, JSON 검증과 타입 강제는 `Pydantic`이 떠맡았습니다. 표면적으로는 거의 정적 타입 언어처럼 보일 정도입니다.

하지만 그 우아함은 런타임이라는 무대 뒤에서 비싼 대가를 치르고 있습니다. 파이썬에서 `user.email`에 접근하는 단순한 연산조차도, 실제로는 인스턴스에 매달린 `__dict__`라는 해시 테이블에서 `"email"`이라는 문자열 키를 해싱하여 버킷을 찾고, 그 안에 든 `PyObject` 포인터를 역참조하는 과정입니다. 필드 세 개짜리 객체 하나가 차지하는 메모리를 측정해 보면, 데이터 자체보다 자료구조의 부대 비용이 압도적으로 큽니다. 게다가 `user.is_admin = True`처럼 정의된 적도 없는 속성을 런타임에 슬쩍 끼워 넣어도 파이썬은 아무 불평 없이 받아들입니다. 이 "유연함"이야말로 대규모 코드베이스에서 오타 하나가 프로덕션까지 살아남는 통로입니다.

`Pydantic`은 이 문제를 런타임 검증으로 막아냅니다. 똑똑한 접근이지만, 본질적으로는 "동적 타입 언어 위에 검증 레이어를 한 겹 더 쌓는" 방식입니다. 모든 객체 생성마다 검증기가 동작하고, CPU 사이클과 메모리를 추가로 소모합니다. 흥미롭게도 Pydantic v2 팀은 이 비용을 줄이기 위해 핵심 엔진(`pydantic-core`)을 **Rust로 재작성**했습니다. 즉, 파이썬에서 가장 사랑받는 데이터 모델링 라이브러리의 심장은 이미 Rust입니다. 이번 장의 질문은 자연스럽습니다 — 그 Rust를 우리가 직접 쓰면 어떻게 될까요?

Rust에는 **`class` 키워드가 존재하지 않습니다.** 대신 데이터의 *형태*를 정의하는 `struct`(구조체), 데이터의 *행동*을 정의하는 `impl` 블록, 그리고 *다형성*을 담당하는 `trait`(트레이트, Chapter 9)으로 관심사가 철저히 분리되어 있습니다. 파이썬 클래스가 "데이터 + 메서드 + 상속 + 동적 속성"을 한 덩어리에 욱여넣은 만능 그릇이었다면, Rust는 이 책임들을 칼같이 갈라놓습니다. 그리고 그 결과물은 객체 헤더도, `__dict__`도, 런타임 타입 검사도 없이 메모리에 연속으로 포장된 데이터입니다.

이번 장에서는 Chapter 4~6에서 다진 소유권·대여·수명의 기반 위에서, 파이썬의 클래스 데이터 모델이 Rust의 `struct`로 어떻게 매핑되는지 — 그리고 `Pydantic`이 하던 일을 `serde`가 어떻게 컴파일 타임에 끝내버리는지 — 메모리 바이트 단위까지 파고들어 살펴봅니다.

---

## 7.1 The Python Way — `__dict__`라는 이름의 해시 테이블

파이썬의 인스턴스가 내부적으로 무엇인지부터 정확히 짚고 가겠습니다. 다음은 가장 단순한 데이터 클래스입니다.

```python
from dataclasses import dataclass, field
from typing import Optional
import uuid

@dataclass
class User:
    username: str
    email: str
    age: int
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    is_active: bool = True

# 인스턴스 생성
user = User(username="pythonista", email="py@example.com", age=30)

# 🚨 파이썬은 정의된 적 없는 속성도 런타임에 받아들입니다.
user.is_admin = False  # 오타든 의도든, 통과됩니다 (__slots__를 쓰지 않는 이상)
```

`@dataclass`는 `__init__`, `__repr__`, `__eq__`를 자동 생성해 줄 뿐, **객체의 실체는 바뀌지 않습니다.** `user`는 여전히 `__dict__`라는 딕셔너리를 들고 다니는 평범한 파이썬 객체입니다. 필드 접근 `user.email`은 다음과 같이 동작합니다.

1. `user`의 타입 객체에서 `email`이 데이터 디스크립터인지 확인 (MRO 탐색).
2. 아니라면 `user.__dict__`에서 `"email"` 키를 해싱하여 조회.
3. 찾은 `PyObject*` 포인터를 역참조하여 실제 `str` 객체에 도달.

이 모든 과정이 **속성 접근 한 번마다** 일어납니다. 이제 메모리를 실측해 봅시다. 필드 세 개짜리 객체입니다.

```python
import sys

class Plain:
    def __init__(self, a, b, c):
        self.a, self.b, self.c = a, b, c

p = Plain(1, 2, 3)
print(sys.getsizeof(p))            # 48 (인스턴스 헤더)
print(sys.getsizeof(p.__dict__))   # 296 (속성 해시 테이블)
# 합계: 344 bytes — 게다가 a/b/c는 각각 별도 int 객체를 가리키는 포인터
print(sys.getsizeof(1))            # 28 (int 객체 하나의 크기)
```

> 💡 **Under the Hood — 파이썬 객체의 진짜 크기**
>
> `Plain(1, 2, 3)` 인스턴스 하나의 총 메모리 비용을 계산해 봅시다.
>
> - 인스턴스 헤더(`PyObject`: 타입 포인터 + 참조 카운트 + `__dict__` 슬롯 등): **48 bytes**
> - `__dict__` 해시 테이블: **296 bytes** (CPython 3.13, 키 공유 최적화 적용 후에도)
> - 필드 `a`, `b`, `c`가 가리키는 `int` 객체 3개: **각 28 bytes × 3 = 84 bytes**
>
> 합계는 **400 bytes를 가뿐히 넘깁니다.** 우리가 저장하려던 데이터는 정수 세 개 — 본질적으로 24 bytes면 충분합니다. 나머지 90% 이상은 동적 타입 시스템을 떠받치기 위한 부대 비용입니다. 그리고 이 객체들은 힙 여기저기에 흩어져 있어, CPU 캐시 입장에서는 최악의 접근 패턴입니다. 필드 하나를 읽을 때마다 포인터를 따라 캐시 미스가 날 메모리 주소로 점프해야 합니다.

### `__slots__`로 발버둥치기

베테랑이라면 `__slots__`로 `__dict__`를 없앨 수 있다는 걸 알 겁니다.

```python
class Slotted:
    __slots__ = ("a", "b", "c")
    def __init__(self, a, b, c):
        self.a, self.b, self.c = a, b, c

s = Slotted(1, 2, 3)
print(sys.getsizeof(s))            # 56 (dict 296 bytes 제거!)
print(hasattr(s, "__dict__"))      # False — 동적 속성 추가 차단
# s.is_admin = True                # 🚨 AttributeError!
```

`__slots__`는 인스턴스마다 딕셔너리를 만드는 대신, 클래스 레벨에 고정된 디스크립터 슬롯을 둡니다. 메모리는 344 → 56 bytes로 줄고, 동적 속성 추가도 막힙니다. **이것이 바로 우리가 Rust `struct`에서 *공짜로, 기본값으로* 얻는 것입니다.** 파이썬에서는 `__slots__`를 의식적으로 선언하고, 상속과의 미묘한 상호작용을 감수하고, 그래도 여전히 각 필드는 힙의 `int` 객체를 가리키는 포인터로 남습니다. Rust에서는 이 모든 게 언어의 기본 동작입니다.

| 측정 항목 | 파이썬 `Plain` (dict) | 파이썬 `Slotted` (slots) | Rust `struct` (3 × i64) |
|---|---|---|---|
| 인스턴스 헤더 | 48 bytes | 56 bytes (슬롯 포함) | **0 bytes** |
| 필드 저장 방식 | `__dict__` 해시 테이블 (296 bytes) | 클래스 디스크립터 슬롯 | **인라인 연속 배치** |
| 필드 값 | 힙의 `int` 객체 포인터 (각 28 bytes) | 동일 | **값 그 자체 (8 bytes씩)** |
| 동적 속성 추가 | 허용 (위험) | 차단 (`AttributeError`) | 컴파일 에러 (E0560/E0609) |
| 3개 정수 총 메모리 | ~428 bytes | ~140 bytes | **24 bytes** |
| 필드 접근 비용 | 해시 + 포인터 역참조 | 슬롯 오프셋 + 포인터 역참조 | 컴파일 타임 오프셋 (명령어 1개) |

이 표의 마지막 줄이 핵심입니다. 파이썬에서 `p.a`는 런타임 조회지만, Rust에서 `p.a`는 컴파일 타임에 "베이스 주소 + 오프셋 N"으로 확정되어 단일 `mov` 명령어로 끝납니다.

---

## 7.2 The Rust Mental Model — 데이터는 메모리 위의 고정된 형태다

Rust의 `struct`를 이해하는 가장 정확한 방법은, 파이썬 객체가 아니라 **C의 구조체**를 떠올리는 것입니다. `struct`는 "이 데이터는 정확히 이런 필드들을, 이런 순서로, 이만큼의 바이트를 차지하며 가진다"는 컴파일 타임 청사진입니다. 런타임에 필드를 추가하거나 제거하는 일은 존재하지 않습니다 — 그건 메모리 레이아웃 자체를 바꾸겠다는 뜻이고, 컴파일된 바이너리에서는 불가능합니다.

Rust 데이터 모델링의 철학은 세 가지로 압축됩니다.

1. **데이터와 행동의 분리.** `struct`는 형태만 정의합니다. 행동(메서드)은 `impl` 블록에, 다형성은 `trait`에 둡니다. 파이썬 클래스의 "한 덩어리" 모델을 세 조각으로 나눕니다.
2. **명시적 생성.** 매직 메서드 `__init__`은 없습니다. 구조체를 만드는 일은 모든 필드를 채우는 일반 함수(관례상 `new`)일 뿐이며, 이건 특별한 문법이 아니라 그냥 함수입니다.
3. **기본 비공개(Private by default).** 파이썬의 `_balance` 언더스코어는 신사협정에 불과하지만, Rust의 비공개 필드는 컴파일러가 강제합니다. `pub`을 붙인 것만 모듈 밖에서 보입니다.

이 멘탈 모델을 코드로 옮기면 다음과 같습니다. Chapter 4에서 본 소유권 개념이 여기서 다시 등장합니다 — 구조체가 `String`을 필드로 가지면, 구조체가 그 `String`을 **소유**합니다.

```rust
use uuid::Uuid;

// 1. 데이터의 '형태'만 정의합니다.
pub struct User {
    pub id: String,
    pub username: String,
    pub email: String,
    pub age: u8, // 0~255면 충분한 나이를 1 byte로 (파이썬 int는 최소 28 bytes)
    pub is_active: bool,
}

// 2. 데이터의 '행동'은 impl 블록에 분리합니다.
impl User {
    // 파이썬의 __init__ 역할. self를 받지 않으므로 @classmethod에 가깝습니다.
    // 반환 타입 Self는 impl 대상 타입(User)의 별칭입니다.
    pub fn new(username: String, email: String, age: u8) -> Self {
        Self {
            id: Uuid::new_v4().to_string(), // 기본값 직접 채움
            username, // 필드 초기화 단축: username: username 과 동일
            email,
            age,
            is_active: true,
        }
    }
}

fn main() {
    // __init__이 아니라 User::new(...) — 연관 함수 호출
    let user = User::new(
        String::from("rustacean"),
        String::from("rs@example.com"),
        30,
    );
    println!("{} / active={}", user.username, user.is_active);

    // user.is_admin = true; // 🚨 컴파일 에러! 정의되지 않은 필드
}
```

### named / tuple / unit struct

파이썬에는 클래스 하나뿐이지만, Rust는 데이터의 성격에 따라 세 종류의 구조체를 제공합니다. 이 구분은 Chapter 8의 뉴타입 패턴(Newtype Pattern)에서 본격적으로 활약합니다.

```rust
// 1. Named struct (명명 필드) — 가장 흔함. dataclass에 대응.
struct Point { x: f64, y: f64 }

// 2. Tuple struct (튜플 구조체) — 필드에 이름 대신 위치 인덱스.
//    "의미 있는 래퍼"를 만들 때 유용 (Chapter 8 뉴타입).
struct Meters(f64);          // .0 으로 접근
struct Rgb(u8, u8, u8);      // .0, .1, .2

// 3. Unit struct (유닛 구조체) — 필드가 아예 없음. 크기 0 bytes.
//    타입 레벨의 마커나 trait 구현 대상으로 사용.
struct Marker;

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    let dist = Meters(42.0);
    let color = Rgb(255, 128, 0);
    println!("{} {} {:?}", p.x, dist.0, (color.0, color.1, color.2));
}
```

`Meters(f64)`가 왜 의미 있을까요? `fn fly(distance: f64)`라는 함수에 실수로 초 단위 시간을 넘겨도 컴파일러는 막지 못합니다. 하지만 `fn fly(distance: Meters)`라면 `Seconds`를 넘기는 순간 타입 에러가 납니다. 파이썬의 `NewType("Meters", float)`이 런타임에 사라지는 힌트인 것과 달리, Rust의 튜플 구조체는 진짜 별개의 타입입니다.

---

## 7.3 `__init__`의 종말: 연관 함수와 메서드 수신자

파이썬에서 메서드의 첫 인자는 늘 `self`였습니다. `@classmethod`는 `cls`를, `@staticmethod`는 아무것도 받지 않았죠. Rust는 이 구분을 훨씬 더 정밀하게, 그리고 **소유권의 언어로** 표현합니다. 메서드가 `self`를 어떻게 받느냐는 단순한 관례가 아니라, 그 메서드가 인스턴스를 **빌릴 것인지, 빌려서 수정할 것인지, 통째로 가져가 소멸시킬 것인지**를 선언하는 일입니다.

```rust
pub struct Document {
    title: String,
    content: String,
}

impl Document {
    // 연관 함수 (Associated Function): self 없음 → @classmethod / @staticmethod
    // Document::new(...) 로 호출. 생성자 패턴.
    pub fn new(title: String) -> Self {
        Self { title, content: String::new() }
    }

    // &self: 불변 대여 → 읽기 전용 메서드. 파이썬의 일반 메서드 + @property getter.
    //        호출해도 인스턴스는 그대로 살아있고, 동시에 여러 번 빌릴 수 있음.
    pub fn word_count(&self) -> usize {
        self.content.split_whitespace().count()
    }

    // &mut self: 가변 대여 → 인스턴스를 수정하는 메서드.
    //            이 메서드가 도는 동안 다른 어떤 참조도 허용되지 않음 (Chapter 5).
    pub fn append(&mut self, text: &str) {
        self.content.push_str(text);
    }

    // self: 소유권 이동 → 인스턴스를 '소비'. 호출 후 원본은 더 이상 사용 불가.
    //       빌더 패턴(Chapter 11)이나 타입 변환에서 자주 등장.
    pub fn into_published(self) -> PublishedDoc {
        PublishedDoc { title: self.title, body: self.content }
    }
}

pub struct PublishedDoc { title: String, body: String }
```

세 수신자의 선택 기준은 명확합니다.

- **`&self`** — 데이터를 읽기만 한다. 기본 선택지. 파이썬 메서드의 90%가 여기 해당합니다.
- **`&mut self`** — 데이터를 수정한다. 파이썬에서 `self.x = ...`를 하던 모든 메서드.
- **`self`** — 인스턴스를 끝내고 다른 무언가로 변환한다. 파이썬에는 직접적 대응이 없습니다.

> 💡 **Under the Hood — 메서드는 그냥 첫 인자가 self인 함수다**
>
> `doc.word_count()`는 컴파일 후 `Document::word_count(&doc)`와 *완전히 동일한* 기계어가 됩니다. Rust 메서드 호출에는 파이썬과 같은 동적 디스패치가 없습니다 — 어떤 타입의 어떤 메서드인지 컴파일 타임에 100% 확정되므로, V-table 조회 없이 직접 주소로 점프하는 정적 호출(static dispatch)입니다. 이것이 무비용 추상화(Zero-cost Abstraction)의 한 단면입니다. `self`를 쓰는 문법적 편의가 런타임 비용 0으로 제공됩니다. (동적 디스패치가 필요한 trait object는 Chapter 9에서 다룹니다.)

### 컴파일러 에러 시연 ①: `self`가 소유권을 가져갈 때

`self`(소유)와 `&self`(대여)의 차이는 추상적인 개념이 아니라 컴파일러가 실제로 강제하는 규칙입니다. `into_published`처럼 `self`를 받는 메서드를 호출하면, 그 인스턴스는 **소비(consume)** 되어 더는 쓸 수 없습니다.

```rust
struct Account { balance: i32 }
impl Account {
    fn consume(self) -> i32 { self.balance } // self를 소유로 받음 → 소비
    fn peek(&self) -> i32 { self.balance }   // &self → 대여
}
fn main() {
    let acc = Account { balance: 100 };
    let _ = acc.consume();
    println!("{}", acc.peek()); // 🚨 consume이 acc를 가져간 뒤 사용 시도
}
```

`cargo run`이 내뱉는 에러는 다음과 같습니다.

```text
error[E0382]: borrow of moved value: `acc`
 --> src/main.rs:9:20
  |
7 |     let acc = Account { balance: 100 };
  |         --- move occurs because `acc` has type `Account`, which does not implement the `Copy` trait
8 |     let _ = acc.consume();
  |                 --------- `acc` moved due to this method call
9 |     println!("{}", acc.peek());
  |                    ^^^ value borrowed here after move
  |
note: `Account::consume` takes ownership of the receiver `self`, which moves `acc`
 --> src/main.rs:3:16
  |
3 |     fn consume(self) -> i32 { self.balance }
  |                ^^^^
```

이 에러를 해독해 봅시다. `E0382: borrow of moved value`는 Chapter 4의 이동 의미론(Move Semantics)이 메서드 수신자에도 그대로 적용됨을 보여줍니다. 컴파일러는 친절하게 "`consume`이 수신자 `self`의 소유권을 가져가므로 `acc`가 이동되었다"는 `note`까지 달아줍니다. 파이썬이라면 `peek()`이 멀쩡히 동작했겠지만, Rust는 "이미 소비된 객체를 또 쓰려 한다"는 논리적 오류를 컴파일 타임에 잡아냅니다. 해법은 간단합니다 — `consume`이 인스턴스를 끝낼 의도가 아니라면 `&self`로 받으면 됩니다.

---

## 7.4 캡슐화: `@property`를 Getter/Setter와 모듈 경계로

파이썬에서 직접 필드 접근을 막고 로직을 태우는 표준 도구는 `@property`입니다.

```python
class Account:
    def __init__(self, balance: int):
        self._balance = balance  # 언더스코어: '내부용'이라는 신사협정 (강제 없음)

    @property
    def balance(self) -> int:
        return self._balance

    @balance.setter
    def balance(self, amount: int):
        if amount < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = amount

acc = Account(100)
acc._balance = -999   # 🚨 아무도 못 막습니다. 언더스코어는 약속일 뿐.
```

`_balance`의 언더스코어는 "건드리지 말아 주세요"라는 부탁이지, 강제력이 없습니다. 누구든 `acc._balance = -999`로 검증을 우회할 수 있습니다. Rust는 이 캡슐화를 **컴파일러가 강제**합니다. 필드에 `pub`을 붙이지 않으면 모듈 외부에서는 존재조차 보이지 않습니다.

```rust
mod bank {
    pub struct Account {
        balance: i32, // pub 없음 → 모듈 밖에서 접근 불가 (진짜 private)
    }

    impl Account {
        pub fn new(balance: i32) -> Result<Self, String> {
            if balance < 0 {
                return Err("Initial balance cannot be negative".to_string());
            }
            Ok(Self { balance })
        }

        // Getter: &self로 읽기만. @property getter에 대응.
        pub fn balance(&self) -> i32 {
            self.balance
        }

        // Setter: &mut self로 수정 + 검증. @balance.setter에 대응.
        // 검증 실패를 예외 대신 Result로 표현 (Chapter 12에서 심화).
        pub fn deposit(&mut self, amount: i32) -> Result<(), String> {
            if amount < 0 {
                return Err("Cannot deposit a negative amount".to_string());
            }
            self.balance += amount;
            Ok(())
        }
    }
}

fn main() {
    let mut acc = bank::Account::new(100).unwrap();
    acc.deposit(50).unwrap();
    println!("balance = {}", acc.balance()); // 150

    // acc.balance = -999; // 🚨 컴파일 에러! field `balance` is private
}
```

여기서 중요한 설계 결정이 드러납니다. 파이썬의 setter는 검증 실패 시 `ValueError`를 **던집니다**(throw). Rust에는 예외가 없으므로(Chapter 12), 실패 가능성을 반환 타입 `Result<(), String>`에 **명시**합니다. 호출자는 이 메서드가 실패할 수 있다는 사실을 타입 시그니처만 보고 알 수 있고, 무시할 수도 없습니다 — `unwrap()`이든 `?`든 `match`든, 어떻게든 결과를 처리해야 컴파일됩니다. 이것이 "검증을 타입으로 표현한다"는 Rust 철학의 출발점입니다.

> 💡 **핵심 — 과도한 getter/setter는 안티패턴**
>
> 파이썬에서 `@property`로 모든 필드를 감싸는 습관이 있다면 잠시 멈추세요. Rust에서 단순히 값을 그대로 반환하는 getter는 **불필요한 보일러플레이트**인 경우가 많습니다. 검증 로직이나 불변식(invariant)이 *없다면*, 그냥 필드를 `pub`으로 공개하는 편이 더 정직하고 관용적입니다. getter/setter는 "이 필드는 항상 음수가 아니다" 같은 **지켜야 할 규칙이 있을 때만** 만드세요. 7.9의 안티패턴 섹션에서 다시 다룹니다.

---

## 7.5 구조체 갱신 문법, `Default`, 그리고 불변 기본값

파이썬 `dataclass`의 기본값은 클래스 정의 시점에 평가됩니다. 이 점이 그 악명 높은 **가변 기본 인자(mutable default argument)** 함정의 근원입니다.

```python
@dataclass
class Config:
    name: str = "default"
    tags: list = []   # 🚨 모든 인스턴스가 같은 리스트를 공유! (field(default_factory=list) 필요)
```

Rust는 이 문제를 `Default` 트레이트와 구조체 갱신 문법(struct update syntax)으로, 그것도 가변 공유 위험 없이 해결합니다.

### `#[derive(Default)]`

```rust
#[derive(Debug, Default)]
struct Config {
    name: String,        // Default → "" (빈 문자열)
    retries: u32,        // Default → 0
    verbose: bool,       // Default → false
    tags: Vec<String>,   // Default → [] (매번 새 빈 Vec, 공유 위험 0)
}

fn main() {
    let c = Config::default();
    println!("{:?}", c);
    // Config { name: "", retries: 0, verbose: false, tags: [] }
}
```

`#[derive(Default)]`는 각 필드 타입의 `Default::default()`를 호출해 구조체를 채우는 코드를 컴파일 타임에 생성합니다. 핵심은 **`tags`가 매 호출마다 새로운 빈 `Vec`을 만든다**는 점입니다 — 파이썬의 가변 기본값처럼 인스턴스끼리 같은 리스트를 공유할 일이 원천적으로 없습니다. 소유권 규칙상 두 구조체가 같은 `Vec`을 동시에 소유할 수 없기 때문입니다.

기본값을 직접 지정하고 싶다면 `Default`를 수동 구현합니다.

```rust
struct Config {
    name: String,
    retries: u32,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            name: String::from("app"),
            retries: 3, // 0이 아닌 기본값
        }
    }
}
```

### 구조체 갱신 문법 `..`

파이썬에서 `dataclasses.replace(obj, field=new_value)`로 하던, "기존 객체를 베이스로 일부 필드만 바꾼 새 객체" 만들기를 Rust는 `..` 문법으로 우아하게 표현합니다.

```rust
#[derive(Debug, Default)]
struct Config {
    name: String,
    retries: u32,
    verbose: bool,
    tags: Vec<String>,
}

fn main() {
    let base = Config::default();

    // name만 바꾸고 나머지는 base에서 가져옴
    let custom = Config {
        name: String::from("production"),
        ..base // "나머지 필드는 base의 값으로 채워라"
    };
    println!("{:?}", custom);
    // Config { name: "production", retries: 0, verbose: false, tags: [] }
}
```

`..base`는 "명시하지 않은 모든 필드를 `base`에서 가져오라"는 뜻입니다. 단, 주의할 점이 있습니다 — `base`의 `String`이나 `Vec` 같은 비-Copy 필드는 새 구조체로 **이동(move)** 됩니다. 즉 위 코드 이후 `base.tags`에 접근하면 컴파일 에러가 납니다. Chapter 4의 소유권 규칙이 여기서도 일관되게 적용됩니다. `..Default::default()`와 결합하면 "몇 개 필드만 지정하고 나머진 전부 기본값"이라는 흔한 패턴이 됩니다.

```rust
let c = Config {
    verbose: true,
    ..Default::default() // 임시 default 객체에서 나머지를 끌어옴
};
```

---

## 7.6 Pydantic의 대체: `serde`로 검증을 타입에 새기기

이제 이번 장의 본론, FastAPI 백엔드의 심장 `Pydantic`을 Rust로 옮길 차례입니다. Rust 생태계의 사실상 표준은 **`serde`** (SERialize/DEserialize)와 그 JSON 백엔드 **`serde_json`**입니다.

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    id: int
    name: str
    price: float = Field(gt=0)
    tags: list[str] = []

item = Item.model_validate_json('{"id": 1, "name": "Apple", "price": 10.5}')
```

`Pydantic`은 런타임에 JSON을 파싱하고, 각 필드를 검증하고, 객체를 만듭니다. v2부터는 이 작업의 핵심을 Rust(`pydantic-core`)가 처리하지만, 파이썬 객체로 변환하는 경계 비용은 여전히 남습니다. `serde`는 다른 차원의 접근을 합니다 — **컴파일 타임에 직렬화/역직렬화 코드를 생성**합니다.

```rust
use serde::{Deserialize, Serialize};

// #[derive] 매크로 하나가 컴파일 타임에 변환 코드를 주입합니다.
// 리플렉션도, 런타임 타입 검사도 없습니다.
#[derive(Debug, Serialize, Deserialize)]
pub struct Item {
    pub id: i32,
    pub name: String,
    pub price: f64,
    #[serde(default)] // 필드가 JSON에 없으면 Default 값(빈 Vec) 사용
    pub tags: Vec<String>,
}

fn main() {
    let json_data = r#"{"id": 1, "name": "Apple", "price": 10.5}"#;

    // JSON → Rust 구조체 (Pydantic의 model_validate_json 대체)
    // 타입 불일치나 필수 필드 누락 시 Err 반환 — unwrap 대신 ? 가 실무 패턴
    let item: Item = serde_json::from_str(json_data).unwrap();
    println!("{:?}", item);
    // Item { id: 1, name: "Apple", price: 10.5, tags: [] }

    // Rust 구조체 → JSON (model_dump_json 대체)
    let back = serde_json::to_string(&item).unwrap();
    println!("{}", back);
    // {"id":1,"name":"Apple","price":10.5,"tags":[]}
}
```

`Cargo.toml`에는 이 두 줄만 추가하면 됩니다.

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### `#[serde(...)]` 어트리뷰트: Pydantic `Field`의 대응표

Pydantic의 `Field(...)`, `alias`, `Config`로 하던 세밀한 제어는 대부분 `#[serde(...)]` 어트리뷰트로 1:1 대응됩니다.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct ApiConfig {
    pub host: String,

    // 누락 시 default_port() 호출 (Pydantic의 default_factory)
    #[serde(default = "default_port")]
    pub port: u16,

    // None이면 직렬화 시 아예 출력 안 함 (Pydantic의 exclude_none)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub timeout: Option<u32>,

    // JSON 키는 "type"이지만 Rust 필드명은 kind (type은 예약어)
    // Pydantic의 alias="type"에 대응
    #[serde(rename = "type")]
    pub kind: String,
}

fn default_port() -> u16 { 8080 }

fn main() {
    let json = r#"{"host": "localhost", "type": "primary"}"#;
    let cfg: ApiConfig = serde_json::from_str(json).unwrap();
    println!("{:?}", cfg);
    // ApiConfig { host: "localhost", port: 8080, timeout: None, kind: "primary" }

    println!("{}", serde_json::to_string(&cfg).unwrap());
    // {"host":"localhost","port":8080,"type":"primary"}  ← timeout(None)은 생략됨
}
```

| Pydantic | serde 어트리뷰트 | 의미 |
|---|---|---|
| `Field(default=...)` / `= []` | `#[serde(default)]` | 누락 필드를 `Default` 값으로 |
| `Field(default_factory=f)` | `#[serde(default = "f")]` | 누락 필드를 함수 `f()`로 채움 |
| `Field(alias="type")` | `#[serde(rename = "type")]` | JSON 키와 Rust 필드명 매핑 |
| `Config(populate_by_name)` 등 | `#[serde(rename_all = "camelCase")]` | 전체 필드 네이밍 일괄 변환 |
| `exclude=True` / `exclude_none` | `#[serde(skip)]` / `skip_serializing_if` | 직렬화에서 제외 |
| `model_validate_json(s)` | `serde_json::from_str(s)` | 역직렬화 |
| `model_dump_json()` | `serde_json::to_string(&v)` | 직렬화 |

### 검증은 어디로? — 타입과 생성자로

Pydantic의 `Field(gt=0)` 같은 값 검증은 `serde`에 직접 대응되는 어트리뷰트가 없습니다. 이것은 `serde`의 한계가 아니라 **설계 철학의 차이**입니다. Rust에서 "가격은 0보다 커야 한다" 같은 불변식은 두 가지 방식으로 표현합니다.

**방법 1 — 타입으로 표현(가장 강력).** "0보다 큰 정수"가 필요하면 `u32`(음수 불가능)나 `NonZeroU32`(0도 불가능)를 쓰면 됩니다. 타입 자체가 불변식을 보장하므로 검증 코드가 아예 필요 없습니다.

**방법 2 — 검증하는 생성자(TryFrom).** 더 복잡한 규칙은 "검증을 통과해야만 만들 수 있는 타입"으로 캡슐화합니다. 이를 흔히 *parse, don't validate* 패턴이라 부릅니다.

```rust
use serde::Deserialize;

// 외부에서 직접 만들 수 없게 필드를 private으로 (뉴타입 패턴, Chapter 8)
#[derive(Debug)]
pub struct Price(f64);

impl Price {
    // 검증을 통과한 값만 Price가 될 수 있음. 이후로는 항상 양수임이 보장됨.
    pub fn new(value: f64) -> Result<Self, String> {
        if value <= 0.0 {
            return Err(format!("price must be > 0, got {value}"));
        }
        Ok(Price(value))
    }
    pub fn get(&self) -> f64 { self.0 }
}

// 역직렬화 시점에 검증을 끼워 넣기: f64로 읽은 뒤 Price::new로 통과
impl<'de> Deserialize<'de> for Price {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where D: serde::Deserializer<'de> {
        let value = f64::deserialize(deserializer)?;
        Price::new(value).map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Deserialize)]
struct Item {
    name: String,
    price: Price, // 역직렬화가 성공하면 price는 반드시 양수
}

fn main() {
    let ok = serde_json::from_str::<Item>(r#"{"name":"A","price":10.5}"#);
    println!("{:?}", ok.map(|i| i.price.get())); // Ok(10.5)

    let bad = serde_json::from_str::<Item>(r#"{"name":"B","price":-3.0}"#);
    println!("{:?}", bad.is_err()); // true — 검증 실패
}
```

핵심 통찰은 이것입니다. Pydantic은 객체가 만들어진 *후에* "이 객체가 유효한가?"를 매번 물어볼 여지를 남기지만, Rust 뉴타입은 **유효하지 않은 객체는 애초에 만들어질 수 없게** 만듭니다. `Price` 타입의 값이 손에 있다는 사실 자체가 "이미 검증을 통과했다"는 증명입니다. 검증을 한 번만 하고, 이후 코드 전체에서 다시 검증할 필요가 없어집니다. (실무에서는 이런 검증을 어트리뷰트로 간편하게 다는 `validator` 크레이트도 있습니다.)

### 제로카피 역직렬화 — Chapter 6의 수명이 다시 등장하다

Chapter 6 말미에서 "구조체 수명이 `serde`의 제로카피 역직렬화로 다시 등장한다"고 예고했습니다. 그 약속을 지킬 차례입니다. 지금까지 우리의 구조체는 모든 필드를 `String`으로 **소유**했습니다. 즉 JSON을 파싱할 때 입력 버퍼에서 문자열을 **복사**해 새 힙 할당을 만들었습니다. 대량 파싱에서 이 복사는 무시할 수 없는 비용입니다.

`serde`는 입력 버퍼가 충분히 오래 산다면, 복사 없이 입력을 **그대로 빌려보는** 역직렬화를 지원합니다. 필드를 `String`이 아니라 `&str`로 두고, 구조체에 수명 매개변수를 다는 것입니다.

```rust
use serde::Deserialize;

// 'a: 이 구조체의 필드들이 입력 버퍼를 빌린다는 수명 선언 (Chapter 6)
#[derive(Debug, Deserialize)]
struct LogEntry<'a> {
    level: &'a str,   // String이 아니라 입력에서 빌린 슬라이스 → 힙 할당 0
    message: &'a str,
}

fn main() {
    let raw = r#"{"level":"ERROR","message":"disk full"}"#;

    // borrow_from_str: level/message가 raw의 메모리를 직접 가리킴 (복사 없음)
    let entry: LogEntry = serde_json::from_str(raw).unwrap();
    println!("{} - {}", entry.level, entry.message);
    // entry는 raw보다 오래 살 수 없음 — 컴파일러가 수명으로 강제
}
```

여기서 Chapter 6의 수명 규칙이 정확히 작동합니다. `LogEntry<'a>`의 필드들은 `raw`의 메모리를 빌린 것이므로, `entry`는 `raw`보다 오래 살 수 없습니다. 만약 `raw`가 먼저 해제되는데 `entry`를 쓰려 하면 컴파일러가 dangling reference로 거부합니다. 파이썬에서는 상상할 수 없는, "복사하지 않고도 안전한" 파싱입니다. 단, JSON에 이스케이프 문자(`\n`, `\"` 등)가 있으면 변환이 필요해 빌림이 불가능하니, 이럴 땐 `Cow<str>`(빌리거나 소유하거나 둘 중 하나, Chapter 14)를 쓰는 것이 실무 패턴입니다.

이것이 데이터 모델 필드를 무작정 `String`으로 둘지, `&str`로 둘지 결정하는 진짜 기준입니다. **데이터를 입력 버퍼보다 오래 보관해야 하면 `String`(소유), 입력의 수명 안에서만 훑고 버린다면 `&str`(빌림)** — Chapter 6에서 본 그 판단 기준이 데이터 모델링에 그대로 적용됩니다.

### `Debug`, `Clone`, `PartialEq`는 거저 얻는가

파이썬 `dataclass`는 `__repr__`, `__eq__`를 공짜로 만들어줬습니다. Rust의 대응물은 `#[derive(...)]`입니다.

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: f64, y: f64 }

fn main() {
    let a = Point { x: 1.0, y: 2.0 };
    let b = a.clone();          // Clone → 명시적 복제 (파이썬 copy.copy)
    println!("{:?}", a);        // Debug → Point { x: 1.0, y: 2.0 } (__repr__)
    println!("{}", a == b);     // PartialEq → true (__eq__)
}
```

각 derive가 무엇을 만드는지는 Chapter 9에서 트레이트로 깊이 다루지만, 데이터 모델링 단계에서 알아둘 핵심은 이것입니다. `#[derive(X)]`는 "이 구조체가 트레이트 `X`를 구현하는 코드를 컴파일러가 대신 작성하라"는 지시입니다. **공짜가 아니라 컴파일 타임에 생성되는 코드**이며, 런타임 비용은 그 트레이트의 실제 동작 비용(예: `Clone`은 실제 복제)뿐입니다. 그래서 의미 없는 derive를 남발하면 컴파일 시간만 늘 뿐입니다. 데이터 모델에는 보통 `Debug`(디버깅 필수), 필요시 `Clone`/`PartialEq`/`Default`/`Serialize`/`Deserialize` 정도를 선택적으로 답니다.

---

## 7.7 Under the Hood: 메모리 레이아웃과 필드 정렬

7.1에서 파이썬 객체의 메모리를 해부했으니, 이번엔 Rust `struct`가 메모리에 어떻게 놓이는지 바이트 단위로 들여다봅시다. 이 차이가 곧 다음 절의 벤치마크 수치를 설명합니다.

```rust
use std::mem::{size_of, align_of};

struct Unordered { a: u8, b: u64, c: u8, d: u32 } // 정렬 관점에서 나쁜 순서
struct Ordered   { b: u64, d: u32, a: u8, c: u8 } // 큰 필드부터
#[repr(C)]
struct ReprC     { a: u8, b: u64, c: u8, d: u32 } // C 호환 레이아웃 강제

fn main() {
    println!("Unordered: {} bytes (align {})", size_of::<Unordered>(), align_of::<Unordered>());
    println!("Ordered  : {} bytes (align {})", size_of::<Ordered>(),   align_of::<Ordered>());
    println!("ReprC    : {} bytes (align {})", size_of::<ReprC>(),     align_of::<ReprC>());
}
```

실제 출력(rustc 1.96, x86-64/aarch64 공통)입니다.

```text
Unordered: 16 bytes (align 8)
Ordered  : 16 bytes (align 8)
ReprC    : 24 bytes (align 8)
```

> 💡 **Under the Hood — 정렬(Alignment)과 패딩(Padding), 그리고 Rust의 필드 재배치**
>
> CPU는 8바이트 값(`u64`)을 8의 배수 주소에서 읽을 때 가장 빠릅니다. 이 정렬 요구를 맞추려고 컴파일러는 필드 사이에 **패딩(빈 바이트)**을 넣습니다. `Unordered`를 C가 선언된 순서대로 배치하면 `a`(1) + 패딩 7 + `b`(8) + `c`(1) + 패딩 3 + `d`(4) = **24 bytes**가 됩니다.
>
> 그런데 측정값은 **16 bytes**입니다. 차이는 무엇일까요? **Rust는 기본적으로 필드 순서를 보장하지 않으며, 컴파일러가 자유롭게 재배치하여 패딩을 최소화합니다.** `Unordered`든 `Ordered`든 컴파일러가 알아서 큰 필드부터 모아 패딩을 줄이므로 둘 다 16 bytes입니다. 반면 `#[repr(C)]`를 붙이면 **C와 동일하게 선언 순서를 강제**하므로 패딩이 살아나 24 bytes가 됩니다.
>
> 이것이 `repr(C)`의 존재 이유입니다 — Chapter 21의 FFI(외부 함수 인터페이스)에서 C 코드와 메모리 레이아웃을 정확히 맞춰야 할 때 필수입니다. 평소에는 Rust 기본 레이아웃이 더 작고 빠르니, `repr(C)`는 C 호환이 필요할 때만 쓰세요.

`String`이나 `Vec`을 가진 구조체는 어떨까요?

```rust
use std::mem::size_of;

#[derive(serde::Deserialize)]
struct Item { id: i32, name: String, tags: Vec<String> }

fn main() {
    println!("String      = {} bytes", size_of::<String>());      // 24
    println!("Vec<String> = {} bytes", size_of::<Vec<String>>()); // 24
    println!("Item        = {} bytes", size_of::<Item>());        // 56
}
```

```text
String      = 24 bytes
Vec<String> = 24 bytes
Item        = 56 bytes
```

`String`은 24 bytes입니다 — 내부적으로 **(힙 데이터 포인터 8 + 길이 8 + 용량 8)**의 세 필드 구조체이기 때문입니다(Chapter 4에서 본 그 구조). 실제 문자 데이터는 힙에 있지만, 구조체 본체에는 이 24바이트 "핸들"이 인라인으로 박힙니다. `Item`은 `id`(4 + 패딩 4) + `name`(24) + `tags`(24) = **56 bytes**가 스택에 연속으로 놓입니다.

핵심 대비는 이렇습니다.

- **파이썬:** 인스턴스 → `__dict__` 포인터 → 해시 테이블 → 각 필드별 `PyObject` 포인터 → 힙의 실제 값. 데이터가 힙 곳곳에 흩어져 매 접근마다 캐시 미스 위험.
- **Rust:** 구조체 본체에 모든 필드(혹은 힙 핸들)가 **연속 배치**. 객체 헤더 0바이트, 참조 카운트 0바이트. 필드 접근은 컴파일 타임에 확정된 오프셋으로 즉시 도달.

`Vec<Item>`을 만들면 이 차이가 증폭됩니다. Rust에서는 56바이트 `Item`들이 힙에 빈틈없이 줄지어 놓여 CPU가 순차적으로 쭉 훑을 수 있습니다(cache-friendly). 파이썬 `list[Item]`은 객체 포인터들의 배열일 뿐이고, 각 객체는 힙 어딘가에 흩어져 있습니다. 이것이 대량 데이터 처리에서 Rust가 압도하는 근본 이유입니다.

---

## 7.8 실전 벤치마크: serde_json vs json vs Pydantic v2

이론은 충분합니다. 실제 수치를 봅시다. 동일한 JSON(필드 5개, 태그 배열 포함)을 100만 번 역직렬화/직렬화한 결과입니다.

**측정 환경:** Apple Silicon, 100만 회 반복, Rust는 `cargo run --release`(최적화 빌드), Python 3.13. JSON 페이로드:
`{"id":1,"name":"Mechanical Keyboard","price":129.99,"tags":["electronics","input","mechanical"],"in_stock":true}`

| 작업 | serde_json (`--release`) | Python `json` (stdlib) | Pydantic v2.11 |
|---|---|---|---|
| 역직렬화 (1M회) | **170 ns/op** | 1,254 ns/op | 964 ns/op |
| 직렬화 (1M회) | **87 ns/op** | 2,356 ns/op | 813 ns/op |
| 역직렬화 상대속도 | 1.0x (기준) | 약 7.4배 느림 | 약 5.7배 느림 |
| 직렬화 상대속도 | 1.0x (기준) | 약 27배 느림 | 약 9.3배 느림 |

읽어낼 교훈이 여럿입니다.

**첫째, serde_json은 stdlib `json` 대비 역직렬화 7배, 직렬화 27배 빠릅니다.** 100만 객체 처리 시 Python `json.dumps`는 2.4초, serde_json은 87밀리초입니다. 처리량이 큰 데이터 파이프라인이나 고빈도 API 게이트웨이에서는 이 차이가 곧 서버 대수와 비용입니다.

**둘째, Pydantic v2가 stdlib `json`보다 빠른 것에 주목하세요.** 검증까지 하는 Pydantic이 검증 안 하는 `json`보다 빠른 이유는 단 하나 — 핵심 엔진 `pydantic-core`가 **Rust로 작성**되었기 때문입니다. Pydantic v1(순수 파이썬)에서 v2로 넘어오며 5~50배 빨라졌다는 공식 벤치마크의 정체가 이것입니다. 즉 여러분이 Pydantic v2의 속도에 만족했다면, 여러분은 이미 Rust의 속도에 만족한 것입니다.

**셋째, 그럼에도 순수 serde가 Pydantic보다 5~9배 더 빠릅니다.** Pydantic은 Rust 코어가 파싱한 결과를 다시 파이썬 객체(`PyObject`)로 변환하는 경계 비용을 치릅니다. serde는 그 경계가 없습니다 — 곧장 네이티브 구조체로 채웁니다. 이 "경계 비용 0"이 Chapter 22 PyO3에서 우리가 직접 활용하게 될 레버리지입니다.

> 🚨 **벤치마크를 읽는 정직한 자세**
>
> 위 수치는 *이 특정 머신, 이 특정 페이로드* 기준입니다. 절대값은 하드웨어·페이로드 크기·필드 구성에 따라 달라집니다. 그리고 **반드시 `--release`로 측정해야 합니다** — 디버그 빌드의 serde는 최적화가 꺼져 있어 수 배 느리며, 이 상태로 "Rust가 안 빠르다"고 결론 내리는 것이 초심자의 가장 흔한 실수입니다. 정밀 측정이 필요하면 `criterion` 크레이트를 쓰세요(통계적 워밍업과 이상치 제거를 제공). 여기서 말하려는 건 "Rust가 항상 N배 빠르다"가 아니라 "직렬화처럼 CPU 바운드 작업에서 한 자릿수~수십 배의 일관된 우위가 있다"는 점입니다.

---

## 7.9 Best Practices & 흔한 함정

지금까지의 내용을 실무 체크리스트로 정리합니다.

### Best Practices ✅

- **생성자는 `new`, 실패할 수 있으면 `Result`를 반환하라.** 단순 생성은 `fn new(...) -> Self`, 검증이 필요하면 `fn new(...) -> Result<Self, Error>`. 여러 생성 방식이 있으면 `from_str`, `with_capacity`처럼 의도가 드러나는 이름을 쓰세요.
- **읽기 메서드는 `&self`, 쓰기 메서드는 `&mut self`, 변환은 `self`.** 이 셋을 의식적으로 구분하면 API가 자기 의도를 문서화합니다.
- **불변식이 없는 필드는 그냥 `pub`으로 공개하라.** 값을 그대로 반환하는 getter는 보일러플레이트입니다. 캡슐화는 "지킬 규칙이 있을 때"의 도구입니다.
- **`#[derive(Debug, Clone, Default)]`를 습관적으로 고려하라.** 디버깅용 `Debug`, 복제용 `Clone`, 기본값용 `Default`는 대부분의 데이터 모델에 유용합니다. 단, 의미 없는 derive는 컴파일 시간만 늘리니 필요한 것만.
- **검증은 타입으로 끌어올려라.** "0보다 큰 정수"는 `NonZeroU32`, "유효한 이메일"은 검증 생성자를 가진 뉴타입으로. *parse, don't validate*.
- **`serde`의 `rename_all = "camelCase"`로 API 컨벤션을 한 번에 맞춰라.** Rust는 `snake_case`, JSON API는 보통 `camelCase`인 충돌을 어트리뷰트 하나로 해결합니다.
- **`clippy`를 켜라.** `cargo clippy`는 불필요한 getter, 빠뜨린 `Default`, 비효율적 `clone` 등을 잡아줍니다.

### 흔한 함정 (파이썬 개발자가 자주 하는 실수)

**함정 1 — 가변 기본값을 두려워하기 / `..` 후 원본 재사용.** 파이썬의 `tags: list = []` 트라우마 때문에 과도하게 방어하는 경우가 있는데, Rust의 `#[derive(Default)]`는 매번 새 `Vec`을 만들어 이 문제가 애초에 없습니다. 반대로 새로 생기는 함정은 구조체 갱신 문법입니다.

```rust
#[derive(Debug, Default)]
struct Config { name: String, tags: Vec<String> }

fn main() {
    let base = Config::default();
    let a = Config { name: "a".into(), ..base }; // base의 tags가 a로 이동(move)
    // println!("{:?}", base); // 🚨 컴파일 에러! base는 부분 이동되어 사용 불가
    println!("{:?}", a);
}
```

`..base`는 비-Copy 필드를 **이동**시킵니다. 원본을 계속 쓰려면 `..base.clone()`을 쓰거나, 애초에 `base`를 다시 만들지 마세요.

**함정 2 — 모든 필드를 무지성으로 `pub`으로 만들기.** 파이썬은 사실상 모든 게 public이었으니 습관적으로 `pub`을 남발하기 쉽습니다. 하지만 그러면 7.4의 검증 캡슐화가 무너집니다. 불변식을 지켜야 하는 필드는 private으로 두고 생성자/메서드로만 변경 가능하게 하세요.

**함정 3 — 검증을 런타임 if 문으로만 처리하기.** Pydantic 스타일로 "객체를 만든 뒤 매번 `if item.price > 0` 검사"를 반복하는 것은 Rust답지 않습니다. 검증을 뉴타입과 생성자로 한 번만 하고, 이후엔 타입이 보장하게 하세요.

**함정 4 — 디버그 빌드로 벤치마크하기.** 7.8에서 강조했듯, `cargo run`(디버그)으로 측정하면 serde가 느립니다. **반드시 `--release`.** 이 실수 하나로 "Rust 별로네"라는 잘못된 결론에 도달하는 사람이 많습니다.

**함정 5 — `String` vs `&str`을 구조체에서 혼동하기.** 구조체 필드로 `&str`(참조)을 두면 Chapter 6의 수명 매개변수가 따라붙어 복잡해집니다(`struct Item<'a> { name: &'a str }`). 데이터 모델은 보통 데이터를 **소유**해야 하므로, 특별한 이유(제로카피 파싱 등)가 없다면 필드는 `String`/`Vec<T>` 같은 소유 타입으로 두세요. 참조 필드는 "이 구조체가 다른 데이터의 뷰일 때"만 정당합니다.

### 컴파일러 에러 시연 ②: 정의되지 않은 필드

마지막으로, 도입부에서 파이썬이 조용히 허용했던 `user.is_admin = False`를 Rust에서 시도하면 어떻게 되는지 봅시다.

```rust
struct User { username: String, age: u8 }
impl User {
    fn new(username: String, age: u8) -> Self {
        Self { username, age, is_active: true } // 정의 안 된 필드를 초기화
    }
}
fn main() {
    let mut u = User::new("a".to_string(), 30);
    u.is_admin = true; // 정의 안 된 필드에 접근
}
```

```text
error[E0560]: struct `User` has no field named `is_active`
 --> src/main.rs:4:31
  |
4 |         Self { username, age, is_active: true }
  |                               ^^^^^^^^^ `User` does not have this field
  |
  = note: all struct fields are already assigned

error[E0609]: no field `is_admin` on type `User`
 --> src/main.rs:9:7
  |
9 |     u.is_admin = true;
  |       ^^^^^^^^ unknown field
  |
  = note: available fields are: `username`, `age`
```

`E0560`(구조체에 없는 필드 초기화)과 `E0609`(없는 필드 접근) 두 에러가 동시에 잡힙니다. 주목할 부분은 마지막 `note`입니다 — 컴파일러가 친절하게 `available fields are: username, age`라며 실제로 존재하는 필드를 알려줍니다. 파이썬이라면 `user.is_admin = False`가 런타임에 통과되어 프로덕션의 어느 멀리 떨어진 코드에서 `AttributeError`로 터졌을 오타가, Rust에서는 컴파일 단계에서 정확한 위치와 함께 박멸됩니다. 이것이 `__dict__`의 동적 유연성을 포기하고 얻는 가장 큰 대가이자 보상입니다.

---

이번 장에서 우리는 파이썬 클래스의 정체가 비싼 `__dict__` 해시 테이블이었음을 메모리 바이트로 확인했고, 그것을 객체 헤더 없이 연속 배치되는 Rust `struct`로 옮겼습니다. `__init__` 대신 연관 함수 `new`를, `@property` 대신 모듈 경계와 검증 생성자를, `dataclass`의 기본값 대신 `Default` 트레이트와 구조체 갱신 문법을 손에 넣었습니다. 무엇보다 `Pydantic`의 런타임 검증을, 컴파일 타임에 코드를 생성하는 `serde`와 "유효하지 않으면 만들어질 수 없는" 뉴타입 타입으로 대체하며 — 검증을 코드가 아니라 타입에 새기는 사고방식을 배웠습니다. Chapter 4~6의 소유권·대여·수명이 이 모든 곳에서 일관되게 작동하는 것도 다시 확인했습니다.

하지만 우리는 아직 데이터를 *조합*하는 법은 다루지 않았습니다. 파이썬 개발자가 가장 버리기 힘들어하는 OOP의 유산, **상속(Inheritance)**은 어디로 갔을까요? 다음 장(Chapter 8)에서는 Rust가 의도적으로 상속을 배제한 이유와, 그 빈자리를 채우는 **컴포지션(Composition)**과 **뉴타입 패턴(Newtype Pattern)** — 이번 장 7.6에서 맛본 그 `Price(f64)`의 본격적인 활용 — 을 살펴봅니다. "바나나를 원했는데 고릴라와 정글을 얻는" 상속의 함정에서, Rust가 어떻게 우리를 구해내는지 확인할 차례입니다.

---

## References

- https://doc.rust-lang.org/book/ch05-00-structs.html
- https://doc.rust-lang.org/book/ch05-01-defining-structs.html
- https://doc.rust-lang.org/book/ch05-03-method-syntax.html
- https://doc.rust-lang.org/rust-by-example/custom_types/structs.html
- https://serde.rs/
- https://docs.rs/serde/latest/serde/
