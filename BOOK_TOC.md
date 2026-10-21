# Python to Rust Complete Guide

## 기획 의도 및 집필 방향
* **타겟 독자**: 10년 차 이상의 Python 백엔드/데이터 엔지니어 (CS 전공자)
* **핵심 컨셉**: "기초 문법 나열"이 아닌, **"Python 코드와의 1:1 대응 및 내부 메모리 구조/어셈블리 관점의 예제 중심 비교"**
* **분량**: 총 9부, 26장 + 서문 + 부록 6종으로 구성된 약 800페이지 분량의 심도 있는 기술 서적.
* **집필 현황**: 26개 챕터 본문, 서문, 부록 6종 작성 완료. 주요 Rust 예제·컴파일러 에러는 실제 `rustc`/`cargo`로 검증했으며, Part 6~8과 부록 A/D/E까지 2차 보강 완료. 설치 안내와 `cargo new`의 기본 에디션은 2026-10-22 stable rustc/cargo 1.99.1에서 `edition = "2024"`로 다시 확인했습니다. 본문 예제는 edition 2021입니다. 재측정하지 않은 성능·크기 수치는 각 자리에 적힌 Rust 1.90/1.96, CPython 3.13.6을 유지합니다.

---

## 📚 목차 (Table of Contents)

### 시작하기 (Front Matter)
* **[서문 · 이 책을 읽는 법 · 개발 환경 설정](./frontmatter.md)**

### Part 1: 기반 다지기 (Foundations & Tooling)
* **Chapter 1. 왜 Python 개발자가 Rust를 배워야 하는가?**
  * Python의 한계: GIL, GC Pause, Duck Typing의 런타임 오버헤드
  * Rust의 약속: Zero-cost Abstraction, Fearless Concurrency
* **Chapter 2. 도구 생태계: 패키징 지옥에서 벗어나기**
  * `pip`, `poetry`, `venv` 멘탈 모델을 `Cargo` 하나로 통합하기
  * 빌드 파이프라인 이해하기: Debug vs Release, Target Architectures
* **Chapter 3. 문법의 이면: 표현식(Expression)과 타입 시스템**
  * Statement vs Expression (모든 것은 값을 반환한다)
  * 강력한 타입 추론과 메모리 레이아웃: Python 객체 헤더(PyObject)와 Rust 기본 타입의 메모리 크기 비교

### Part 2: 핵심 패러다임 전환 (The Paradigm Shift)
* **Chapter 4. 소유권 (Ownership): GC 없이 살아남기**
  * Python의 Reference Counting 체계의 한계와 비용
  * Rust의 Ownership 3원칙과 `Drop` 트레이트 (RAII 패턴)
  * [예제] Move Semantics: 변수 할당 시 Heap 메모리에서 일어나는 일
* **Chapter 5. 참조와 대여 (Borrowing & References)**
  * 깊은 복사(`copy.deepcopy`)와 얕은 복사의 재정의
  * 가변 참조(`&mut`)와 불변 참조(`&`)의 공존 규칙
* **Chapter 6. 수명 (Lifetimes): 컴파일러에게 의도 설명하기**
  * Python에는 존재조차 하지 않는 개념 이해하기
  * Dangling Pointer 방지 원리 및 명시적 라이프타임 선언

### Part 3: Python 객체지향의 해체와 Rust 데이터 모델링 (Deep Dive)
**목표**: 파이썬의 풍부한 클래스 문법(상속, 믹스인, 매직 메서드, Pydantic)이 Rust 체계로 어떻게 번환되는지 마스터하기
* **Chapter 7. Struct와 데이터 모델링 (`dataclass`와 `Pydantic`의 대체)**
  * `__init__`의 종말: Struct 선언과 초기화, 연관 함수(Associated Functions)
  * 속성 캡슐화: Python의 `@property` 데코레이터를 Rust의 Getter/Setter로 구현하기
  * [예제] `Pydantic` 모델을 Rust `Struct` + `Serde` 크레이트로 마이그레이션 및 직렬화/역직렬화 속도 비교
* **Chapter 8. 상속(Inheritance)의 종말과 컴포지션(Composition)**
  * Python의 깊은 상속 트리와 다중 상속(MRO), `super()`의 문제점
  * Rust의 "상속 없음" 철학: Composition over Inheritance
  * 뉴타입 패턴(Newtype Pattern): `class MyString(str):` 을 Rust의 Tuple Struct로 안전하게 대체하기
  * [예제] Base Class를 상속받던 Python 코드를 구조체 합성(Composition)으로 리팩토링하기
* **Chapter 9. 매직 메서드(Dunder)에서 Trait으로**
  * Python의 Duck Typing(Behavior)을 컴파일 타임에 강제하는 `Trait`
  * `__str__`, `__repr__` $\rightarrow$ `std::fmt::Display`, `std::fmt::Debug`
  * `__eq__`, `__lt__` $\rightarrow$ `PartialEq`, `Eq`, `PartialOrd`, `Ord`
  * `__add__`, `__mul__` $\rightarrow$ `std::ops::Add`, `std::ops::Mul`
  * [예제] 커스텀 Vector(수학 벡터) 클래스를 만들며 Python 매직 메서드와 Rust 연산자 오버로딩 1:1 매핑
* **Chapter 10. 열거형(Enums)을 활용한 상태 기계와 다형성**
  * 다형성을 구현하는 두 가지 길: Trait Object(Dynamic) vs Enum(Static)
  * `Union[TypeA, TypeB]`와 `isinstance()`를 대체하는 Rust의 대수적 데이터 타입(ADT)
  * [예제] 결제 시스템의 상태(Pending, Success, Failed)를 Enum과 Pattern Matching으로 결함 없이 모델링하기
* **Chapter 11. 믹스인(Mixin) 패턴과 생성(Creational) 패턴**
  * Python의 Mixin 클래스를 Trait Default Methods로 구현하기
  * 복잡한 `__init__` (수많은 `*args`, `**kwargs`) $\rightarrow$ Rust의 빌더 패턴(Builder Pattern)으로 우아하게 해결하기

### Part 4: 우아한 실패 (Error Handling)
* **Chapter 12. `try-except`는 없다: `Option`과 `Result`**
  * `None` (Null)과 예외를 타입 시스템 안으로 끌어들이기
  * 모나드(Monad)적 접근: `map`, `and_then`, `unwrap_or` 체이닝
* **Chapter 13. 에러 전파와 커스텀 에러 설계**
  * `?` 연산자의 마법: 에러 전파 보일러플레이트 제거
  * 실무 에러 핸들링: `thiserror` vs `anyhow`
  * [예제] Python의 복잡한 커스텀 Exception 계층도를 Rust Error Enum으로 마이그레이션

### Part 5: 컬렉션과 이터레이터 (Collections & Iteration)
* **Chapter 14. 소유권을 가진 스마트 포인터 컬렉션**
  * `List`, `Dict`, `Str` vs `Vec<T>`, `HashMap<K, V>`, `String`/`&str`
  * Python의 문자열(UTF-8) 처리와 Rust의 `String` (길이, 인덱싱의 차이)
* **Chapter 15. 이터레이터와 제너레이터(Generator)**
  * Python `yield`, `__iter__`, `__next__` $\rightarrow$ Rust `Iterator` 및 `IntoIterator` 트레이트
  * Zero-cost 이터레이터 체이닝 (Map, Filter, Fold)
  * [예제] 대용량 데이터 파이프라인 작성 비교: `itertools` vs Rust Iterator

### Part 6: GIL의 해방과 동시성 (Concurrency & Async)
* **Chapter 16. 멀티스레딩: `threading` 모듈의 한계 돌파**
  * GIL이 없는 세계: 진정한 하드웨어 병렬성
  * 채널(Channel, `mpsc`)을 통한 메시지 패싱 패턴 (`queue.Queue` 대체)
* **Chapter 17. 상태 공유와 동기화 (Shared State)**
  * `Arc` (Atomic Reference Counted)와 `Mutex`, `RwLock`
  * 데드락과 Data Race의 컴파일 타임 방지 (`Send`, `Sync` 트레이트)
  * [예제] Python `multiprocessing.Manager` 기반 공유 메모리를 Rust `Arc<Mutex<T>>`로 리팩토링
* **Chapter 18. 비동기 프로그래밍 (Async/Await)**
  * `asyncio` Event Loop vs `Tokio` 런타임 아키텍처 비교
  * Future 트레이트의 State Machine 변환 원리
  * [예제] Python `aiohttp` vs Rust `reqwest` + `tokio` 웹 크롤러 벤치마크

### Part 7: 메타프로그래밍과 깊은 곳으로 (Advanced Rust)
* **Chapter 19. 매크로 (Macros): 데코레이터와 메타클래스의 대체재**
  * 선언적 매크로 (`macro_rules!`)와 절차적 매크로 (Procedural Macros)
  * 컴파일 타임 코드 생성 (Python 데코레이터의 런타임 오버헤드 극복)
* **Chapter 20. 스마트 포인터와 메모리 트릭**
  * `Box<T>`, `Rc<T>`, `RefCell<T>`: Interior Mutability 패턴
  * Python의 객체 참조 그래프(Reference Graph)를 Rust로 수동 구현해보기

### Part 8: 궁극의 브릿지 (Python 🤝 Rust)
* **Chapter 21. C FFI와 Unsafe Rust**
  * 컴파일러의 눈을 가리는 `unsafe` 키워드
  * Python C-API의 한계와 메모리 누수 위험성
* **Chapter 22. PyO3: Python과 Rust의 완벽한 결합**
  * `maturin`을 활용한 Rust로 Python Native Extension 작성하기
  * Python 객체를 Rust에서 조작하기 (PyAny, PyDict 등) 및 GIL 릴리즈
  * [예제] Python `pandas`로 느린 병목 구간의 데이터 처리를 Rust + PyO3로 모듈화하여 50배 최적화하기

### Part 9: 프로덕션으로 가는 길 (From Language to Production)
**목표**: 언어를 익혔으니 실제로 출시한다 — 테스트·문서화·CLI·웹 서비스·성능 튜닝을 Python 도구 경험에 1:1로 대응시켜 마스터하기
* **[Chapter 23. 테스트와 문서화: `pytest`를 넘어서](./chapters/chapter_23.md)**
  * `#[test]`/`#[cfg(test)]` 인라인 테스트, `cargo test`가 unit+integration+doctest를 한 번에
  * 문서화 테스트(doctest): `///` 예제가 실제로 컴파일·실행됨 (Python `doctest`의 강화판)
  * 속성 기반 테스트 `proptest`(Hypothesis 대응), `rstest`/`insta`/`cargo-nextest`
* **[Chapter 24. CLI 애플리케이션: `argparse`/`click`에서 `clap`으로](./chapters/chapter_24.md)**
  * `#[derive(Parser)]` — 구조체가 곧 인자 정의, 서브커맨드는 `enum`(망라성 강제)
  * 자동 `--help`/타입 파싱/검증, `anyhow` 연계 에러 출력, 셸 자동완성
* **[Chapter 25. 웹 서비스: `FastAPI`에서 `Axum`으로](./chapters/chapter_25.md)**
  * 핸들러=`async fn`, 익스트랙터(`Json`/`Path`/`State`)=타입 기반 의존성 주입
  * `tower` 미들웨어, `Arc` 공유 상태, `IntoResponse`로 에러→HTTP 상태코드
  * [예제] Ch22 집계 서비스를 REST API로 — GIL 없는 단일 프로세스 멀티코어
* **[Chapter 26. 성능 엔지니어링: 프로파일링·`criterion`·SIMD](./chapters/chapter_26.md)**
  * `cargo flamegraph`/`perf`/`samply` 프로파일링, `criterion`+`black_box` 방법론 정본화
  * 자동 벡터화 확인(`--emit asm`)과 `std::simd`, 캐시 친화 레이아웃(SoA vs AoS), `#[inline]`/LTO

### 부록 (Appendices)
* **[부록 A. Cargo & 프로젝트 레퍼런스](./appendices/appendix_a_cargo.md)** — 명령어/`Cargo.toml`/프로파일/rustup 빠른 참조
* **[부록 B. 컴파일러 에러 해독집](./appendices/appendix_b_errors.md)** — 자주 만나는 18종 에러 코드의 원인과 해결
* **[부록 C. Python → Rust 치트시트](./appendices/appendix_c_cheatsheet.md)** — 관용구 1:1 번역 표
* **[부록 D. 실전 크레이트 생태계 지도](./appendices/appendix_d_crates.md)** — 도메인별 라이브러리 대응표
* **[부록 E. Python → Rust 마이그레이션 플레이북](./appendices/appendix_e_migration.md)** — 핫스팟 이전 전략과 체크리스트
* **[부록 F. 용어집 (Glossary)](./appendices/appendix_f_glossary.md)** — 핵심 용어의 단일 진실 원천, 최초 등장 챕터 딥링크 + 양방향 색인

---
## 📝 집필 방식 (Sample)
모든 챕터는 다음 플로우로 진행됩니다.
1. **The Python Way**: 기존 파이썬에서 우리가 해결하던 방식 (상속, 매직 메서드, Pydantic 등)과 런타임 한계.
2. **The Rust Mental Model**: 해당 문제를 바라보는 러스트의 데이터 지향적/컴파일 타임 철학.
3. **Rust Implementation**: Trait, Enum, Struct를 조합한 견고한 러스트 코드 예제.
4. **Under the Hood**: 메모리 정렬, V-Table 동적 디스패치 등 10년 차 독자를 위한 심화 CS 지식.
