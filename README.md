# Python to Rust: 파이썬 개발자를 위한 러스트 입문 가이드

환영합니다. 이 가이드는 CS 기초(메모리 구조, 포인터, 동시성, 다형성 등)에 대한 이해가 충분한 Python 개발자를 대상으로 합니다. 

Rust는 단순한 "빠른 언어"가 아닙니다. **프로그래밍에 대한 멘탈 모델의 전환**을 요구하는 언어입니다. Python이 "개발자의 생산성"을 위해 많은 것을 런타임과 GC(Garbage Collector)에 위임했다면, Rust는 "런타임 비용 제로와 메모리 안정성"을 위해 책임을 컴파일러와 개발자(타입 시스템을 통해)에게 명시적으로 요구합니다.

---

## 목차

1. [패러다임 전환: 런타임에서 컴파일 타임으로](#chapter-1-패러다임-전환-런타임에서-컴파일-타임으로)
2. [생태계 맵핑: Python Tooling vs Rust Tooling](#chapter-2-생태계-맵핑-python-tooling-vs-rust-tooling)
3. [변수, 제어문, 함수: 표현식 기반 언어](#chapter-3-변수-제어문-함수-표현식-기반-언어)
4. [핵심 혁신 1: 소유권(Ownership)과 대여(Borrowing)](#chapter-4-핵심-혁신-1-소유권ownership과-대여borrowing)
5. [데이터 모델: Struct, Enum, Pattern Matching](#chapter-5-데이터-모델-struct-enum-pattern-matching)
6. [핵심 혁신 2: 에러 처리 (Exceptions vs Monads)](#chapter-6-핵심-혁신-2-에러-처리-exceptions-vs-monads)
7. [다형성과 제네릭: Duck Typing에서 Trait으로](#chapter-7-다형성과-제네릭-duck-typing에서-trait으로)
8. [동시성: GIL의 해방과 Fearless Concurrency](#chapter-8-동시성-gil의-해방과-fearless-concurrency)
9. [통합: PyO3 (Python과 Rust의 결합)](#chapter-9-통합-pyo3-python과-rust의-결합)

---

## Chapter 1: 패러다임 전환 (런타임에서 컴파일 타임으로)

- **Garbage Collection (X)**: Rust는 GC가 없습니다. 대신 스코프 기반의 리소스 관리(RAII)와 컴파일러의 'Borrow Checker'를 통해 메모리를 해제합니다.
- **Dynamic Typing -&gt; Static Typing**: Python의 Type Hint(`mypy`)를 넘어섭니다. Rust의 타입 시스템은 튜링 완전(Turing Complete)하며, 컴파일러는 타입이 맞지 않으면 바이트코드 자체를 생성하지 않습니다.
- **Exceptions (X)**: `try-except`가 없습니다. 대신 함수의 반환값으로 `Result<T, E>` (성공 또는 실패)를 명시적으로 반환합니다.
- **Null (X)**: 10억 달러의 실수라 불리는 Null 포인터 에러를 방지하기 위해 `None` 대신 `Option<T>` 타입을 사용합니다.

---

## Chapter 2: 생태계 맵핑 (Python Tooling vs Rust Tooling)

Python에서 여러 도구를 조합해서 쓰던 것들이 Rust에서는 `cargo` 하나로 통합되어 있습니다.


| Python 생태계                           | Rust 생태계            | 설명                                            |
| :------------------------------------ | :------------------- | :--------------------------------------------- |
| `pip`, `poetry`, `pipenv`            | `**cargo**`         | 패키지 관리자 및 빌드 시스템. 모든 것을 관장합니다.                |
| `requirements.txt`, `pyproject.toml` | `**Cargo.toml**`    | 의존성 및 프로젝트 메타데이터 정의 파일                        |
| `venv`, `virtualenv`                 | **(필요 없음)**         | Cargo는 기본적으로 글로벌하게 캐싱하고 프로젝트별로 격리된 빌드를 수행합니다. |
| `pytest`, `unittest`                 | `**cargo test**`    | 내장 테스트 러너. 코드 바로 옆에 테스트를 작성할 수 있습니다.          |
| `black`, `yapf`                      | `**cargo fmt**`     | 공식 코드 포매터                                     |
| `flake8`, `pylint`                   | `**cargo clippy**`  | 러스트의 공식 린터 (매우 강력하고 교육적입니다)                   |
| `pylance`, `mypy`                    | `**rust-analyzer**` | IDE 통합을 위한 Language Server (VSCode의 필수 확장)    |


---

## Chapter 3: 변수, 제어문, 함수 (표현식 기반 언어)

Rust는 구문(Statement)보다 **표현식(Expression)** 중심의 언어입니다.

```rust
// 변수는 기본적으로 불변(Immutable)입니다. 
// 가변성(Mutating)이 필요하면 `mut` 키워드를 명시해야 합니다.
let x = 5;
let mut y = 10;
y += 1;

// 타입 추론이 강력하지만, 명시할 수도 있습니다.
let name: &str = "Pythonista";
```

### 표현식으로서의 if/else

Python의 `x = "A" if cond else "B"` (Ternary operator)와 비슷하지만, Rust에서는 블록 자체가 값을 반환합니다.

```rust
let condition = true;
let number = if condition { 
    5 // 세미콜론(;)이 없으면 반환(return)을 의미합니다.
} else { 
    10 
};
```

### 함수

Python과 달리 파라미터와 반환 타입은 반드시 명시해야 합니다.

```rust
// Python: def add(a: int, b: int) -> int: return a + b
fn add(a: i32, b: i32) -> i32 {
    a + b // return 키워드 생략 가능 (세미콜론 없음)
}
```

---

## Chapter 4: 핵심 혁신 1: 소유권(Ownership)과 대여(Borrowing)

Python 개발자가 Rust에서 가장 크게 부딪히는 벽입니다. Python은 객체 참조(Reference)를 복사해서 전달하지만, Rust는 **소유권(Ownership)**을 이동(Move)시킵니다.

### 4.1 소유권 (Ownership)

메모리(Heap)를 가리키는 포인터의 '주인'은 오직 하나뿐입니다.

```rust
let s1 = String::from("hello"); // Heap에 할당됨
let s2 = s1; // 소유권이 s1에서 s2로 이동(Move)됨 (Shallow copy가 아님!)

// println!("{}", s1); // 컴파일 에러! s1은 더 이상 유효하지 않음.
println!("{}", s2); // 정상 동작
```

### 4.2 대여 (Borrowing)

매번 소유권을 넘겨주면 프로그래밍이 불가능합니다. 그래서 **참조(Reference, `&`)**를 통해 값을 "빌려" 줍니다. (Python에서 객체를 함수 인자로 넘기는 것과 논리적으로 비슷합니다.)

- **불변 참조 (`&T`)**: 여러 명이 동시에 읽을 수 있습니다. (Read-only)
- **가변 참조 (`&mut T`)**: 한 번에 오직 한 명만 쓸 수 있습니다. (Exclusive Read-Write)

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
} // 여기서 s가 스코프를 벗어나도, 소유권이 없으므로 Heap 메모리가 해제되지 않음.

let s1 = String::from("hello");
let len = calculate_length(&s1); // &를 통해 '대여'해줌
println!("{}'s length is {}", s1, len); // s1은 여전히 소유권을 가지고 있으므로 사용 가능
```

> **CS Note**: Rust 컴파일러의 Borrow Checker는 컴파일 타임에 Data Race 조건(여러 쓰레드가 동시에 가변 참조를 가지는 상황)을 수학적으로 완벽히 차단합니다.

---

## Chapter 5: 데이터 모델 (Struct, Enum, Pattern Matching)

Python의 Class 기반 객체 지향에서 벗어나 데이터(Struct)와 동작(Trait)을 분리합니다.

### Struct (Python `dataclasses` 매핑)

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

// 메서드는 impl 블록에 분리해서 작성합니다.
impl User {
    fn new(username: String, email: String) -> Self { // Class Method (Constructor)
        Self { username, email, sign_in_count: 1, active: true }
    }

    fn deactivate(&mut self) { // Instance Method (requires mutable reference to self)
        self.active = false;
    }
}
```

### Enum과 Pattern Matching (Rust의 꽃)

Rust의 Enum은 단순한 상수가 아니라, Algebraic Data Types(대수적 데이터 타입)입니다. Enum의 각 Variants가 다른 타입의 데이터를 품을 수 있습니다.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 }, // 익명 구조체 포함
    Write(String),           // 튜플 포함
    ChangeColor(i32, i32, i32),
}

fn process_message(msg: Message) {
    // Python 3.10+의 match 구문보다 훨씬 강력하며, 
    // 컴파일러가 모든 케이스(Exhaustiveness)를 처리했는지 검사합니다.
    match msg {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to {}, {}", x, y),
        Message::Write(text) => println!("Text: {}", text),
        Message::ChangeColor(r, g, b) => println!("Color: {}, {}, {}", r, g, b),
    }
}
```

---

## Chapter 6: 핵심 혁신 2: 에러 처리 (Exceptions vs Monads)

Python에서는 예기치 않은 상황에 `raise Exception`을 사용하지만, 이는 Control Flow를 숨기고 예외 처리를 누락할 위험을 만듭니다. Rust는 이를 값으로 명시합니다.

### 6.1 `Option<T>` (Null 대신)

값이 없을 수 있는 상황은 `Option`으로 감쌉니다.

```rust
enum Option<T> {
    Some(T),
    None,
}
```

### 6.2 `Result<T, E>` (try-except 대신)

작업이 실패할 수 있는 경우 `Result`를 반환합니다.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}

use std::fs::File;

fn open_config() -> Result<File, std::io::Error> {
    let f = File::open("config.json");
    match f {
        Ok(file) => Ok(file),
        Err(e) => Err(e),
    }
}
```

### 6.3 `?` 연산자 (에러 전파의 마법)

파이썬에서 예외가 위로 전파되듯, Rust에서는 `?` 연산자를 사용해 우아하게 에러를 전파합니다.

```rust
use std::fs::File;
use std::io::Read;

// ? 연산자를 쓰면 코드가 파이썬처럼 짧아집니다.
fn read_username() -> Result<String, std::io::Error> {
    let mut f = File::open("hello.txt")?; // 실패하면 여기서 바로 Err 반환 후 종료
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

---

## Chapter 7: 다형성과 제네릭 (Duck Typing에서 Trait으로)

Python의 Duck Typing(`hasattr` 또는 매직 메서드)과 `abc.ABC`를 대체하는 것이 **Trait**입니다. Interface와 유사합니다.

```rust
// 행동(Behavior) 정의
trait Summary {
    fn summarize(&self) -> String;
}

struct Article { title: String, author: String }
struct Tweet { username: String, content: String }

// 다형성 구현
impl Summary for Article {
    fn summarize(&self) -> String { format!("{}: {}", self.title, self.author) }
}

impl Summary for Tweet {
    fn summarize(&self) -> String { format!("@{} - {}", self.username, self.content) }
}

// 다형성 활용 (제네릭 + Trait Bound)
// "Summary 트레이트를 구현한 어떤 타입(T)이든 받을 수 있다"
fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

---

## Chapter 8: 동시성 (GIL의 해방과 Fearless Concurrency)

Python에서 멀티쓰레딩을 작성할 때 GIL(Global Interpreter Lock) 때문에 병목을 겪거나 `multiprocessing`을 억지로 써야 했던 경험이 있으실 겁니다. 

Rust는 GIL이 없습니다. 진정한 하드웨어 쓰레드를 가벼운 비용으로 사용하며, 앞서 말한 **소유권과 타입 시스템(Send, Sync trait)이 Data Race를 컴파일 타임에 완벽하게 차단**합니다. 코드가 컴파일된다면, 데드락이 있을지언정 메모리 충돌(Data Race)은 발생하지 않습니다. ("Fearless Concurrency")

### 쓰레드와 채널 (Message Passing)

Python의 `queue.Queue` 구조를 Rust에서는 `mpsc` (Multi-Producer, Single-Consumer) 채널로 구현합니다.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || { // tx의 소유권을 쓰레드로 이동(move)
        let val = String::from("hi from thread");
        tx.send(val).unwrap();
        // println!("{}", val); // 컴파일 에러! val은 이미 채널로 이동됨
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

### 공유 상태 (Shared State: Arc &amp; Mutex)

Python의 `threading.Lock`은 사용자가 실수로 Lock을 걸지 않고 변수에 접근하는 것을 막지 못합니다. Rust의 `Mutex<T>`는 데이터를 Mutex **안에** 보관하므로, Lock을 획득(`lock().unwrap()`)하지 않고는 데이터에 물리적으로 접근할 수 없습니다.

- `Arc`: Atomic Reference Counted (Python 객체의 레퍼런스 카운팅을 멀티쓰레드 환경에서 안전하게 수행)

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap(); // Lock 획득 (스코프 종료 시 자동 해제됨 - RAII)
        *num += 1;
    });
    handles.push(handle);
}
```

*비동기 처리(Async/Await) 역시 `tokio`라는 강력한 생태계를 통해 Python `asyncio`보다 월등한 성능으로 처리됩니다.*

---

## Chapter 9: 통합: PyO3 (Python과 Rust의 결합)

Rust를 배웠다고 Python을 버려야 하는 것은 아닙니다. `pydantic-core`, `polars`, `cryptography` 등 최근 Python 생태계의 고성능 라이브러리들은 모두 Rust로 작성되고 있습니다.

**PyO3**와 **Maturin**을 사용하면 Rust 코드를 Python 모듈로 매우 쉽게 빌드할 수 있습니다. 계산 집약적인 로직, GIL 우회가 필요한 병렬 처리 로직만 Rust로 작성하고, 비즈니스 로직과 접착 코드는 생산성이 높은 Python으로 작성하는 아키텍처가 개발자에게 가장 강력한 무기가 될 것입니다.

---

## 다음 단계 (추천 학습 경로)

1. [Rust 공식 문서 (The Book)](https://doc.rust-lang.org/book/): 챕터 4(Ownership)를 3번 정독하세요.
2. [Rustlings](https://github.com/rust-lang/rustlings): 작은 예제 코드를 컴파일되도록 고치는 실습 도구입니다.
3. [Rust by Example](https://doc.rust-lang.org/rust-by-example/): 문법 사전처럼 활용하세요.
4. 기존에 작성했던 Python 백그라운드 워커/크롤러 스크립트 중 하나를 골라 Rust로 포팅해보세요!
