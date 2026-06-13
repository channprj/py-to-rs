# Python to Rust Complete Guide — 서문과 시작하기

이 책의 첫 페이지를 펼친 여러분은 아마도 막연한 호기심이 아니라 구체적인 통증을 가지고 있을 것입니다. 새벽에 울린 OOM 알림, p99 레이턴시 그래프의 톱니, `multiprocessing` 워커가 `pickle` 직렬화에 잡아먹은 시간, 그리고 "이 핫 패스만 C로 다시 쓰면…"이라고 중얼거리다 만 수많은 순간들. 이 앞부분(Front Matter)은 그 통증의 정체를 짚고, 이 책이 그것을 어떻게 다룰지, 그리고 첫 줄의 Rust 코드를 컴파일하기까지 무엇을 준비해야 하는지를 안내합니다.

---

## 서문 (Preface)

### 왜 또 하나의 Rust 책인가

Rust 입문서는 이미 차고 넘칩니다. 그런데 그 대부분은 암묵적으로 **C/C++ 출신**이거나 **프로그래밍 입문자**를 독자로 가정합니다. 전자를 위한 책은 "포인터가 무엇인지"를 설명하지 않고 곧장 소유권(Ownership)으로 들어가고, 후자를 위한 책은 변수와 반복문에 한 챕터를 씁니다. 둘 다 10년 차 Python 엔지니어에게는 어긋난 옷입니다. 여러분은 스택과 힙, 포인터, 동시성, 다형성, 시간복잡도를 이미 압니다. 동시에 — 그리고 이것이 핵심인데 — 여러분의 손가락과 뇌는 **GC가 있는 동적 언어의 멘탈 모델**에 10년간 길들여져 있습니다. `a = b`가 "참조를 복사"한다고 믿고, 객체의 수명은 신경 쓸 필요가 없으며, "일단 돌려보고 안 되면 고친다"가 몸에 배어 있습니다.

바로 이 지점에서 대부분의 시니어 Python 개발자가 Rust에 입문했다가 좌절합니다. 문법이 어려워서가 아닙니다. `match`나 클로저는 여러분이 이미 아는 패턴 매칭, 람다와 다르지 않습니다. 무너지는 지점은 언제나 **소유권과 대여의 벽(The Borrow Checker Wall)**입니다. 컴파일러가 `error[E0382]: borrow of moved value`를 내뱉는 순간, 여러분이 지난 10년간 한 번도 의식한 적 없던 질문 — "이 데이터의 주인은 누구이며, 언제 해제되는가" — 을 컴파일러가 강제로 묻기 시작합니다. Python 런타임이 GC와 참조 카운팅(Reference Counting)으로 조용히 대신 처리해 주던 일을, Rust는 컴파일 타임에 여러분의 입으로 직접 답하게 만듭니다. 이 인지 부조화를 넘지 못하면, 사람들은 `.clone()`을 사방에 박아 컴파일러를 입막음하고 "Rust는 까다롭기만 하고 안 빠르다"는 잘못된 결론에 도달합니다.

### 이 책의 접근: 1:1 대응 + 내부 구조

이 책은 다른 길을 택합니다. 모든 핵심 개념을 **여러분이 매일 쓰는 Python 코드와 1:1로 대응**시켜 출발합니다. `dataclass`는 `struct`로, `@property`는 게터/세터로, 매직 메서드(Dunder)는 트레이트(Trait)로, `Union[A, B]`와 `isinstance`는 열거형(Enum)과 패턴 매칭으로, `try/except`는 `Result`로, `asyncio`는 `tokio`로 — 익숙한 개념을 발판 삼아 낯선 개념으로 건너갑니다. 여러분의 머릿속에는 이미 풍부한 Python 개념 그래프가 있습니다. 새 개념을 그 그래프에 매달면 학습 속도가 몇 배로 빨라집니다.

그러나 1:1 대응만으로는 절반입니다. 시니어 독자를 진짜로 설득하는 것은 "어떻게"가 아니라 **"왜, 그리고 내부에서 무슨 일이 일어나는가"**입니다. 그래서 이 책은 모든 중요한 비교에 **Under the Hood** 분석을 붙입니다. Python의 정수 `1`이 왜 28바이트인지(`PyObject` 헤더의 해부), `list`가 왜 정수 배열이 아니라 포인터 배열이라 캐시 미스(Cache Miss)를 유발하는지, 트레이트 객체(`dyn Trait`)의 v-table이 메모리에 어떻게 놓이는지, `async fn`이 어떻게 상태 기계(State Machine)로 컴파일되는지를 **바이트 단위 메모리 레이아웃, 정적/동적 디스패치, 그리고 가끔은 디스어셈블리** 수준에서 들여다봅니다. 추측이 아니라 측정과 컴파일러 출력으로 이야기합니다. 과장된 "100배" 마케팅 대신, 입력 크기와 빌드 모드(`--release`)를 명시한 정직한 수치로 설득합니다.

### 누가 읽어야 하는가

이 책은 다음 독자를 위해 쓰였습니다.

- **10년 차 이상의 Python 백엔드/데이터/인프라 엔지니어.** `FastAPI`로 API를 짜고, `pandas`/`numpy`로 데이터를 다루고, `asyncio`/`multiprocessing`으로 동시성을 처리해 본 사람.
- 성능의 벽에 부딪혀 본 사람. CPython 인터프리터 루프가 병목이라는 것을 프로파일러로 확인했지만, C 확장을 짜는 것은 부담스러웠던 사람.
- 자신이 매일 쓰는 도구(`ruff`, `uv`, `pydantic` v2, `polars`)가 Rust로 쓰였다는 사실을 알고, "그렇다면 나도"라고 생각해 본 사람.

전제하는 지식은 다음과 같습니다. **CS 기초** — 스택과 힙의 차이, 포인터와 참조, 시간/공간 복잡도, 동시성과 경쟁 상태(Race Condition), 다형성. 그리고 **Python 실무 경험** — 타입 힌트, 데코레이터, 컨텍스트 매니저, 제너레이터, `asyncio`를 코드로 읽고 쓸 수 있는 수준. 이 책은 "변수란 무엇인가"를 설명하지 않습니다. `for` 루프에 한 페이지를 쓰지 않습니다. 그것은 여러분의 경력에 대한 모욕일 것입니다. 대신 여러분이 **이미 아는 것을 지렛대로** 모르는 것을 빠르게 정복하도록 돕습니다.

마지막으로 솔직하게 밝힙니다. 이 책은 "Python을 버리고 Rust로 갈아타라"고 설득하지 않습니다. Python은 여전히 세계 최고의 접착제(Glue) 언어이자 프로토타이핑 도구입니다. 이 책의 목표는 **Python이 끝나는 지점 — CPU 바운드 핫 패스, GIL이 가로막는 진짜 병렬성, 결정론적 메모리 제어가 필요한 경계 — 에서 꺼내 들 무기를 여러분의 손에 쥐여 주는 것**입니다. 22장에서 다루는 PyO3는 그 두 세계를 잇는 다리이며, 이 책 전체가 그 다리를 건너기 위한 준비입니다.

---

## 이 책을 읽는 법

### 8부 22장의 흐름

이 책은 8개의 부(Part), 22개의 장(Chapter)으로 구성됩니다. 단순히 주제를 나열한 것이 아니라, **소유권이라는 핵심 개념을 중심에 두고 동심원처럼 확장**하는 구조입니다.

| Part | 장 | 학습 목표 |
|------|------|-----------|
| **1. 기반 다지기** | 1–3 | Python의 런타임 비용(GIL, GC, 동적 디스패치)을 바이트 수준에서 해부하고, Cargo로 패키징 지옥을 끝내고, 표현식 기반 타입 시스템에 적응합니다. |
| **2. 패러다임 전환** | 4–6 | 이 책의 심장. 소유권(Ownership), 대여(Borrowing), 수명(Lifetime)을 정복합니다. 여기를 넘으면 나머지는 응용입니다. |
| **3. 객체지향의 해체** | 7–11 | `dataclass`/`Pydantic`/상속/믹스인/매직 메서드를 struct·트레이트·열거형·빌더 패턴으로 재구성합니다. |
| **4. 우아한 실패** | 12–13 | `try/except`를 버리고 `Option`/`Result`와 `?` 연산자로 에러를 타입 시스템 안에 들입니다. |
| **5. 컬렉션과 이터레이터** | 14–15 | `list`/`dict`/`str`을 `Vec`/`HashMap`/`String`으로, 제너레이터를 무비용(Zero-cost) 이터레이터로 옮깁니다. |
| **6. 동시성과 비동기** | 16–18 | GIL 없는 진짜 병렬성, `Arc`/`Mutex`/`RwLock`, 그리고 `asyncio` 대 `tokio`를 다룹니다. |
| **7. 고급 Rust** | 19–20 | 매크로(데코레이터/메타클래스의 대체)와 스마트 포인터(`Box`/`Rc`/`RefCell`)로 메타프로그래밍과 내부 가변성(Interior Mutability)에 진입합니다. |
| **8. 궁극의 브릿지** | 21–22 | C FFI와 `unsafe`를 거쳐, PyO3로 Python과 Rust를 결합해 실전 병목을 최적화합니다. |

### 두 가지 독서 경로

**정독파(처음부터 끝까지).** 권장 경로입니다. 특히 **Part 2(4–6장)는 절대 건너뛰지 마십시오.** 소유권을 어설프게 이해한 채로 7장 이후를 읽으면, 모든 코드 예제에서 컴파일러와 싸우게 됩니다. 4–6장은 천천히, 컴파일러 에러를 직접 재현해 가며 읽으십시오.

**속독파(필요한 곳부터).** 이미 Rust 기초를 어느 정도 아는 독자라면, 1장(동기 부여)과 4–6장(소유권 3부작)만 확인한 뒤 관심 주제로 점프해도 됩니다. 다만 각 장은 이전 장의 예제를 점진적으로 발전시키므로(예: `Vector2D` 수학 벡터, 결제 상태 기계는 여러 장에 걸쳐 진화합니다), 본문의 상호 참조 링크(예: "Chapter 4에서 본 소유권")를 만나면 해당 장을 가볍게 훑는 것을 권합니다.

급한 실무 목적이 명확하다면 다음 지름길도 가능합니다.

- **"파이썬 함수를 Rust로 가속하고 싶다"** → 1 → 4 → 5 → 14 → 22
- **"동시성 병목을 풀고 싶다"** → 1 → 4 → 16 → 17 → 18
- **"데이터 모델/직렬화를 옮기고 싶다"** → 7 → 9 → 10 → 12 → 13

### 각 장의 3단 구조: Python Way → Rust Mental Model → Under the Hood

모든 장은 일관된 사고 흐름을 따릅니다. 이 리듬에 익숙해지면 책 전체가 한결 빠르게 읽힙니다.

1. **The Python Way — 우리가 지금까지 해온 방식.** 익숙한 Python 코드와, 그 방식이 런타임에 치르는 숨은 비용을 구체적 시나리오로 제시합니다.
2. **The Rust Mental Model — 사고방식의 전환.** 같은 문제를 Rust가 어떤 철학으로 바라보는가. "왜 이렇게 설계했는가"에 답합니다.
3. **Rust Implementation.** 컴파일 가능한 완전한 코드(대부분 `fn main()` 포함)로 구현합니다. 발췌일 때는 `// ...`로 명시합니다.
4. **Under the Hood.** 메모리 레이아웃, 디스패치 방식, 디스어셈블리 관점의 심화 분석. `> 💡` 인용 박스로 표시됩니다.

여기에 각 장은 **실전 예제(Hands-on)**, **Best Practices & 안티패턴 체크리스트(clippy 경고 연계)**, 그리고 **흔한 함정(Python 습관이 컴파일 에러로 이어지는 사례)** 섹션으로 마무리됩니다. 의도적으로 틀린 코드를 보여주고 실제 `rustc`가 내뱉는 에러 메시지(`error[E0382]` 형태)를 해독하는 시연이 곳곳에 등장합니다 — 컴파일러는 이 책에서 가장 엄격하면서도 친절한 교사입니다.

### 표기 박스 읽는 법

본문에는 세 종류의 강조 박스가 반복됩니다.

- **💡 Under the Hood** — 메모리 바이트, v-table, 캐시 동작 등 내부 구조 심화. 시간이 없다면 건너뛰어도 코드는 돌아가지만, 시니어 독자라면 여기서 가장 큰 재미를 느낄 것입니다.
- **🚨 경고(No Free Lunch)** — 트레이드오프, 흔한 오해, 함정. 실무에서 발등을 찍기 전에 읽으십시오.
- **비교표** — Python 개념 ↔ Rust 개념 매핑. 각 장 최소 하나씩, 빠른 참조용으로 활용하십시오.

### 부록 활용법

본문이 개념의 흐름을 위해 깊이 파고들지 않은 주제는 부록(Appendices)에 정리됩니다. 부록은 처음부터 읽는 것이 아니라 **사전(Reference)처럼** 필요할 때 펼치는 용도입니다. 본문에서 "부록 X 참고"를 만나면 그때 들춰보십시오. 마이그레이션 중 막혔을 때, 또는 특정 크레이트(serde, thiserror, anyhow, tokio, rayon, pyo3)의 빠른 레시피가 필요할 때 가장 유용합니다.

---

## 개발 환경 설정 가이드

이론으로 들어가기 전에, 첫 줄의 코드를 컴파일할 수 있는 환경을 갖춥시다. Python의 `pyenv` + `venv` + `pip` + `black` + `mypy` 조합과 달리, Rust는 **단 두 개의 도구(`rustup`과 `cargo`)**만 설치하면 빌드·의존성·테스트·포맷·린팅·문서화가 전부 따라옵니다. 이 절은 그 차이를 손으로 체감하는 데 목적이 있습니다. (도구 생태계의 철학과 `Cargo.toml`의 한 줄 한 줄에 대한 깊은 해부는 [Chapter 2](./chapters/chapter_02.md)에서 다룹니다. 여기서는 일단 작동하는 환경부터 만듭니다.)

### 1단계: rustup 설치

`rustup`은 Rust의 **툴체인 관리자**입니다. Python으로 치면 `pyenv`(버전 관리) + 그 이상의 역할을 합니다. 컴파일러(`rustc`), 패키지 매니저(`cargo`), 포매터(`rustfmt`), 린터(`clippy`)를 한 번에 설치하고 채널(stable/beta/nightly)을 전환합니다.

**macOS / Linux:**

```bash
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

설치 마법사가 뜨면 대개 기본값(`1) Proceed with standard installation`)을 선택하면 됩니다. 끝나면 현재 셸에 PATH를 적용합니다.

```bash
$ source "$HOME/.cargo/env"   # 또는 새 터미널을 엽니다
```

**Windows:** [https://rustup.rs](https://rustup.rs) 에서 `rustup-init.exe`를 내려받아 실행합니다. Visual Studio C++ 빌드 도구(MSVC 링커)가 함께 필요하다는 안내가 나오면 동의해 설치하십시오. (WSL2를 쓴다면 위의 Linux 방식이 더 매끄럽습니다.)

설치를 검증합니다.

```bash
$ rustc --version
rustc 1.96.0 (ac68faa20 2026-05-25)

$ cargo --version
cargo 1.96.0 (30a34c682 2026-05-25)

$ rustup --version
rustup 1.28.2 (e4f3ad6f8 2025-04-28)
```

> 💡 **Under the Hood: 채널(Channel)과 툴체인(Toolchain)**
>
> Rust는 6주마다 새 **stable** 버전을 릴리스합니다. `rustup`은 이를 **채널**로 추상화합니다 — `stable`(실무 기본값), `beta`(다음 릴리스 미리보기), `nightly`(실험적 기능). 프로젝트별로 버전을 고정하고 싶다면 루트에 `rust-toolchain.toml`을 두십시오. Python의 `.python-version`과 같은 역할을 하지만, 팀원이 클론만 하면 `cargo`가 알아서 해당 툴체인을 내려받아 사용한다는 점이 다릅니다.
>
> ```toml
> # rust-toolchain.toml
> [toolchain]
> channel = "1.96.0"
> components = ["rustfmt", "clippy"]
> ```
>
> 현재 설치 상태는 `rustup show`로 확인합니다. 새 stable로 올리려면 `rustup update` 한 줄이면 됩니다.

### 2단계: 에디터 설정 (rust-analyzer + VS Code)

Rust 개발 경험의 절반은 **rust-analyzer**(공식 Language Server)가 책임집니다. 타입 추론 결과를 인레이 힌트(Inlay Hint)로 보여 주고, 소유권 위반을 타이핑하는 즉시 빨간 줄로 표시하며, 컴파일러 에러를 에디터 안에서 해독해 줍니다. Python의 `pyright`/`pylance`에 해당하지만, 타입 정보가 런타임 주석이 아니라 **컴파일러가 실제로 강제하는 진실**이라는 점에서 훨씬 강력합니다.

**VS Code:** 확장 마켓플레이스에서 **`rust-analyzer`** (publisher: The Rust Programming Language)를 설치합니다. 반드시 이것이어야 합니다 — 구형 `Rust`(rls 기반) 확장은 더 이상 권장되지 않습니다.

권장 `settings.json` 최소 설정:

```json
{
  "editor.formatOnSave": true,
  "rust-analyzer.check.command": "clippy",
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer"
  }
}
```

`"check.command": "clippy"`로 설정하면 저장할 때마다 단순 컴파일 체크가 아니라 **clippy 린트**까지 돌아, "이건 더 관용적으로 쓸 수 있다"는 제안을 실시간으로 받습니다.

**JetBrains 사용자:** RustRover(전용 IDE) 또는 IntelliJ의 Rust 플러그인을 쓰면 됩니다. Vim/Neovim/Helix 사용자는 LSP 클라이언트에 `rust-analyzer` 바이너리(`rustup component add rust-analyzer`로 설치)를 연결하십시오.

### 3단계: 첫 프로젝트 — cargo new

이제 `cargo`로 첫 프로젝트를 만듭니다. Python에서 디렉터리를 만들고 `venv`를 켜고 `requirements.txt`를 비워 두는 의식 전체가, 여기서는 한 줄입니다.

```bash
$ cargo new hello_rust
    Creating binary (application) `hello_rust` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

$ cd hello_rust
```

생성된 구조는 다음과 같습니다.

```text
hello_rust/
├── Cargo.toml      # 의존성·메타데이터 (pyproject.toml에 해당)
├── .gitignore      # target/ 등을 자동 제외
└── src/
    └── main.rs     # 진입점
```

`cargo new`는 이미 `git init`까지 마쳐 둡니다. 기본 `src/main.rs`는 다음과 같습니다.

```rust
fn main() {
    println!("Hello, world!");
}
```

실행합니다.

```bash
$ cargo run
   Compiling hello_rust v0.1.0 (/path/to/hello_rust)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.41s
     Running `target/debug/hello_rust`
Hello, world!
```

`cargo run`은 컴파일과 실행을 한 번에 합니다. `Finished 'dev' profile`이라는 문구에 주목하십시오 — 기본은 **디버그 빌드(최적화 꺼짐)**입니다. 성능을 측정할 때는 반드시 `--release`를 붙여야 합니다(이것이 입문자가 "Rust가 Python보다 느리다"고 착각하는 가장 흔한 함정입니다, [Chapter 1](./chapters/chapter_01.md) 참고).

```bash
$ cargo run --release
   Compiling hello_rust v0.1.0 (/path/to/hello_rust)
    Finished `release` profile [optimized] target(s) in 0.52s
     Running `target/release/hello_rust`
Hello, world!
```

### 4단계: 포맷·린트·테스트

세 가지 품질 도구가 별도 설치 없이 따라옵니다. Python에서 `black` + `flake8`/`ruff` + `pytest`를 각각 깔고 설정하던 일이 여기서는 기본 제공됩니다.

```bash
$ cargo fmt           # rustfmt: 코드 포맷 (black에 해당). 무인자로 전체 정리
$ cargo clippy        # clippy: 린트 (ruff/pylint에 해당). 관용적 개선 제안
$ cargo test          # 내장 테스트 러너 (pytest에 해당)
```

`cargo test`를 빈 프로젝트에서 돌리면 다음과 같이 나옵니다(아직 테스트가 없으므로 0개 통과).

```text
$ cargo test
   Compiling hello_rust v0.1.0 (/path/to/hello_rust)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.38s
     Running unittests src/main.rs (target/debug/deps/hello_rust-9c1f...)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Rust는 테스트를 별도 디렉터리가 아니라 **소스 파일 안에** 두는 것을 권장합니다(`#[cfg(test)]` 모듈). 단위 테스트와 구현이 한 파일에 살아 리팩토링 시 함께 움직입니다.

### 5단계: 의존성 추가 — cargo add

외부 크레이트(Crate, Rust의 패키지)는 `cargo add`로 추가합니다. `pip install`에 해당하지만, **가상 환경이 없습니다.** 의존성은 전역이 아니라 프로젝트 단위로 격리되며, `Cargo.lock`이 정확한 버전을 고정합니다(`poetry.lock`에 해당). 가상 환경을 켜고 끄는 의식 자체가 사라집니다.

```bash
$ cargo add serde --features derive
$ cargo add serde_json
```

실행 후 `Cargo.toml`은 이렇게 됩니다. 이것이 실무에서 마주칠 **최소 설정 예시**입니다.

```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"          # 언어 에디션. 2015/2018/2021/2024 중 선택 (이 책은 2021 기준).

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = 3             # 기본값. 0~3 + "s"(크기 최적화). 필요 시 조정.
```

> 💡 **Under the Hood: edition은 버전이 아니다**
>
> `edition = "2021"`을 보고 "이게 Rust 버전인가?"라고 오해하기 쉽습니다. 아닙니다. **에디션(Edition)**은 3년마다 갱신되는 언어 방언으로, 키워드나 기본 동작의 비호환 변경을 모아 적용하는 단위입니다. 핵심은 **에디션이 달라도 모든 크레이트가 함께 컴파일된다**는 것 — 2015 에디션 라이브러리를 2024 에디션 프로젝트가 그대로 의존할 수 있습니다. Python 2→3 마이그레이션이 생태계를 수년간 쪼갰던 비극이 Rust에는 구조적으로 없습니다. 이 책의 모든 예제는 가장 널리 쓰이는 안정 에디션인 2021을 기준으로 하며, 새 프로젝트라면 최신 stable 에디션을 써도 무방합니다.

### 추천 학습 도구

설치 없이, 또는 가볍게 손을 풀 수 있는 도구들입니다.

- **Rustlings** (`cargo install rustlings`) — 컴파일 에러를 하나씩 고쳐 나가며 배우는 공식 연습 문제 모음. 소유권 감각을 손에 익히는 데 특히 좋습니다. 이 책의 4–6장과 병행하기를 강력히 권합니다.
- **Rust Playground** ([https://play.rust-lang.org](https://play.rust-lang.org)) — 설치 없이 브라우저에서 코드를 컴파일·실행하고, **생성된 어셈블리/LLVM IR/MIR까지 볼 수 있는** 온라인 환경. 이 책의 "Under the Hood" 박스를 읽다가 "정말 그렇게 컴파일되나?"가 궁금하면 여기서 직접 확인하십시오.
- **`cargo expand`** (`cargo install cargo-expand`) — 매크로가 전개된 실제 코드를 보여 줍니다. 19장(매크로) 학습 시 필수입니다.
- **The Rust Book / Rust by Example** — 공식 무료 교재. 이 책이 다루지 않는 문법의 구석은 여기서 채우십시오.

---

## 표기 규약 (Conventions)

이 책 전체에서 일관되게 사용하는 표기 규칙입니다.

**용어 병기.** 기술 용어는 처음 등장할 때 **한국어(English)** 형식으로 병기합니다. 예: **소유권(Ownership)**, **대여(Borrowing)**, **수명(Lifetime)**, **단일형화(Monomorphization)**, **무비용 추상화(Zero-cost Abstraction)**. 책 전반에서 번역어를 통일합니다 — Reference는 "참조", Trait는 "트레이트"로 고정합니다.

**코드 블록.** 언어 태그로 출처를 명시합니다.

- ` ```python ` — 비교 대상 Python 코드. 시니어 독자를 위해 타입 힌트를 포함하며 `from __future__ import annotations`를 가정합니다.
- ` ```rust ` — Rust 코드. `rustfmt` 스타일(4-스페이스 들여쓰기, `snake_case`), `edition = 2021` 기준. 핵심 예제는 `fn main()`을 포함해 그대로 `cargo run` 됩니다. 발췌는 `// ...`로 표시합니다.
- ` ```toml ` / ` ```bash ` / ` ```text ` — 각각 `Cargo.toml`, 셸 명령, 출력/다이어그램.

셸 블록에서 `$`로 시작하는 줄은 **여러분이 입력하는 명령**, 그 아래 줄은 **기대 출력**입니다. 코드 주석은 한국어로, "왜"를 짧게 설명합니다. 의도적으로 컴파일에 실패하는 코드는 `// 🚨 컴파일 에러!`로, 런타임 패닉은 `// panic!`으로 명확히 표시합니다.

**강조 박스.** 본문에는 마크다운 인용(`>`)으로 표시되는 두 종류의 박스가 반복됩니다.

> 💡 **Under the Hood** — 메모리 레이아웃(바이트 단위), 정적/동적 디스패치, 또는 디스어셈블리 관점의 내부 동작 분석.

> 🚨 **경고 / No Free Lunch** — 트레이드오프, 흔한 오해, 실무에서의 함정. 이모지는 절제하여, 위 둘 정도로만 씁니다.

**컴파일러 에러 시연.** 틀린 코드를 보여줄 때는 실제 `rustc` 출력을 `error[E0382]`처럼 에러 코드와 함께 인용하고, 그 메시지를 한 줄씩 해독합니다. 컴파일러 에러는 벌이 아니라 가장 정확한 문서입니다.

**벤치마크 측정 환경 표기.** 성능 수치는 반드시 **측정 맥락**과 함께 제시합니다 — 입력 크기, 빌드 모드(거의 항상 `--release`), 그리고 의미가 있을 때는 CPU 아키텍처. 측정 환경의 기준은 다음과 같습니다.

> **벤치마크 기준 환경:** Apple Silicon (aarch64), Rust 1.96.0 stable `--release` 빌드, CPython 3.11. 수치는 절대값보다 **상대 배수**가 중요하며, 여러분의 머신에서는 달라질 수 있습니다. 과장을 피하기 위해 현실적 범위(보통 2~50배)만 제시하고, 측정 코드를 함께 싣습니다.

이제 준비가 끝났습니다. 다음 [Chapter 1](./chapters/chapter_01.md)에서, 여러분이 매일 치는 다섯 줄짜리 Python 함수가 CPU 위에서 실제로 무엇을 하는지 — 그리고 그 비용이 어디에 숨어 있는지 — 메모리 바이트 단위로 해부하며 본격적인 여정을 시작합니다.
