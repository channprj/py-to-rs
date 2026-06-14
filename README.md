# Python to Rust Complete Guide

> 파이썬 개발자를 위한 러스트 마스터클래스 — 문법 나열이 아니라, **Python 코드와의 1:1 대응 + 내부 메모리 구조·디스패치 관점의 비교**로 Rust를 정복합니다.

10년 차 이상의 Python 백엔드/데이터/인프라 엔지니어를 위해 쓴 책입니다. `asyncio`, `multiprocessing`, `Pydantic`, `numpy`로 성능의 벽에 부딪혀 본 적이 있다면, 이 책은 그 벽 너머의 도구를 손에 쥐여 줍니다. Rust는 단순히 "빠른 언어"가 아니라 **프로그래밍에 대한 멘탈 모델의 전환**을 요구하는 언어이며, 이 책은 그 전환을 Python 개발자의 언어로 안내합니다.

전 챕터는 **The Python Way → The Rust Mental Model → Rust Implementation → Under the Hood**의 흐름을 따르며, 실전 예제·벤치마크·컴파일러 에러 시연·Best Practices를 담았습니다. 책에 등장하는 Rust 코드 예제와 컴파일러 에러 메시지는 모두 실제 `rustc` / `cargo`로 컴파일·실행해 검증했습니다.

---

## 대상 독자와 전제

- **대상**: CS 기초(메모리 구조, 포인터, 동시성, 다형성, 시간복잡도)를 갖춘 시니어 Python 개발자.
- **전제**: Python 실무 경험, 타입 힌트·데코레이터·`asyncio` 사용 경험. Rust 사전 지식은 필요 없습니다.
- **분량**: 8부 22장 + 부록 5종, A4 약 600페이지 분량.

처음 시작한다면 **[서문과 시작하기 (frontmatter.md)](./frontmatter.md)** 부터 읽으세요. 집필 의도, 읽는 법, 개발 환경 설정 가이드가 담겨 있습니다.

---

## 목차 (Table of Contents)

상세 목차와 집필 방향은 **[BOOK_TOC.md](./BOOK_TOC.md)** 를 참고하세요.

### 시작하기

- [서문 · 이 책을 읽는 법 · 개발 환경 설정](./frontmatter.md)

### Part 1. 기반 다지기 (Foundations &amp; Tooling)

- [Chapter 1. 왜 Python 개발자가 Rust를 배워야 하는가?](./chapters/chapter_01.md)
- [Chapter 2. 도구 생태계: 패키징 지옥에서 벗어나기](./chapters/chapter_02.md)
- [Chapter 3. 문법의 이면: 표현식과 타입 시스템](./chapters/chapter_03.md)

### Part 2. 핵심 패러다임 전환 (The Paradigm Shift)

- [Chapter 4. 소유권(Ownership): GC 없이 살아남기](./chapters/chapter_04.md)
- [Chapter 5. 참조와 대여(Borrowing &amp; References)](./chapters/chapter_05.md)
- [Chapter 6. 수명(Lifetimes): 컴파일러에게 의도 설명하기](./chapters/chapter_06.md)

### Part 3. 데이터 모델링 (Python OOP의 해체)

- [Chapter 7. Struct와 데이터 모델링 (`dataclass`/`Pydantic` 대체)](./chapters/chapter_07.md)
- [Chapter 8. 상속의 종말과 컴포지션](./chapters/chapter_08.md)
- [Chapter 9. 매직 메서드(Dunder)에서 Trait으로](./chapters/chapter_09.md)
- [Chapter 10. 열거형(Enum) 상태 기계와 다형성](./chapters/chapter_10.md)
- [Chapter 11. 믹스인 패턴과 생성 패턴(Builder)](./chapters/chapter_11.md)

### Part 4. 우아한 실패 (Error Handling)

- [Chapter 12. `try-except`는 없다: `Option`과 `Result`](./chapters/chapter_12.md)
- [Chapter 13. 에러 전파와 커스텀 에러 설계 (`thiserror`/`anyhow`)](./chapters/chapter_13.md)

### Part 5. 컬렉션과 이터레이터 (Collections &amp; Iteration)

- [Chapter 14. 소유권을 가진 스마트 포인터 컬렉션 (`Vec`/`HashMap`/`String`)](./chapters/chapter_14.md)
- [Chapter 15. 이터레이터와 제너레이터](./chapters/chapter_15.md)

### Part 6. GIL의 해방과 동시성 (Concurrency &amp; Async)

- [Chapter 16. 멀티스레딩: `threading` 모듈의 한계 돌파](./chapters/chapter_16.md)
- [Chapter 17. 상태 공유와 동기화 (`Arc`/`Mutex`/`RwLock`)](./chapters/chapter_17.md)
- [Chapter 18. 비동기 프로그래밍 (`asyncio` vs `tokio`)](./chapters/chapter_18.md)

### Part 7. 메타프로그래밍과 깊은 곳으로 (Advanced Rust)

- [Chapter 19. 매크로: 데코레이터/메타클래스의 대체재](./chapters/chapter_19.md)
- [Chapter 20. 스마트 포인터와 메모리 트릭 (`Box`/`Rc`/`RefCell`)](./chapters/chapter_20.md)

### Part 8. 궁극의 브릿지 (Python 🤝 Rust)

- [Chapter 21. C FFI와 Unsafe Rust](./chapters/chapter_21.md)
- [Chapter 22. PyO3: Python과 Rust의 완벽한 결합](./chapters/chapter_22.md)

### 부록 (Appendices)

- [부록 A. Cargo &amp; 프로젝트 레퍼런스](./appendices/appendix_a_cargo.md)
- [부록 B. 컴파일러 에러 해독집](./appendices/appendix_b_errors.md)
- [부록 C. Python → Rust 치트시트](./appendices/appendix_c_cheatsheet.md)
- [부록 D. 실전 크레이트 생태계 지도](./appendices/appendix_d_crates.md)
- [부록 E. Python → Rust 마이그레이션 플레이북](./appendices/appendix_e_migration.md)

---

## 어떻게 읽을까

- **정독파**: 1장부터 순서대로. Part 2(소유권·대여·수명)는 Rust의 심장이니 충분히 시간을 들이세요.
- **속독파**: 1 → 4 → 5 → 12 → 14 → 15 → 16 → 22 순으로 핵심 패러다임만 훑은 뒤 필요한 장으로.
- **목적별 지름길**:
  - *성능 최적화가 급하다*: 1 → 14 → 15 → 16 → 17 → 22 → 부록 E
  - *데이터 모델링/API*: 7 → 8 → 9 → 10 → 13
  - *동시성/비동기*: 16 → 17 → 18
  - *Python에 Rust 끼워넣기*: 21 → 22 → 부록 D → 부록 E
- 막히는 컴파일러 에러는 **[부록 B](./appendices/appendix_b_errors.md)**, Python 관용구의 Rust 대응은 **[부록 C](./appendices/appendix_c_cheatsheet.md)** 에서 바로 찾아보세요.

---

## 저장소 구조

```text
py-to-rs/
├── README.md                 # 이 파일 (책 소개 + 전체 내비게이션)
├── BOOK_TOC.md               # 상세 목차와 집필 방향
├── frontmatter.md            # 서문 · 읽는 법 · 환경 설정
├── chapters/                 # 본문 22개 챕터
│   ├── chapter_01.md
│   ├── ...
│   └── chapter_22.md
└── appendices/               # 부록 5종 (레퍼런스)
    ├── appendix_a_cargo.md
    ├── appendix_b_errors.md
    ├── appendix_c_cheatsheet.md
    ├── appendix_d_crates.md
    └── appendix_e_migration.md
```

---

## 다음 단계 (추천 학습 경로)

1. [Rust 공식 문서 — The Book](https://doc.rust-lang.org/book/): 챕터 4(Ownership)를 세 번 정독하세요.
2. [Rustlings](https://github.com/rust-lang/rustlings): 작은 예제를 컴파일되도록 고치는 실습 도구.
3. [Rust by Example](https://doc.rust-lang.org/rust-by-example/): 문법 사전처럼 활용.
4. 기존에 작성했던 Python 백그라운드 워커·크롤러·파싱 스크립트 중 하나를 골라 Rust로 포팅해 보세요. 핫스팟만 PyO3로 옮기는 것부터 시작하면 좋습니다([부록 E](./appendices/appendix_e_migration.md)).
