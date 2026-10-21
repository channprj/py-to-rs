# TODO — Python to Rust 마스터클래스 집필

## 목표
파이썬 시니어 개발자 대상, 실전·best practice 중심 Rust 학습서. 초고 목표는 A4 600페이지였고, 26장과 부록 보강 이후 독자용 본문은 A4 약 800페이지다.
기존 22개 스켈레톤 챕터(전체 ~15,800단어)를 각 8,000~11,000단어로 심화 확장.

## 전략
- 단일 집필 스펙(`tasks/chapter_spec.md`)으로 톤·구조·깊이 통일.
- 챕터는 독립적 → 병렬 서브에이전트로 확장(파일 충돌 없음).
- 웨이브마다 `/git-commit-push`로 진행 저장.

## 진행 체크리스트

### Phase 0 — 기반
- [x] 전체 마크다운 파일 분석
- [x] 집필 스펙(`chapter_spec.md`) 작성
- [x] 계획(`todo.md`) 작성

### Phase 1 — 챕터 심화 확장 (병렬 웨이브)
- [x] Wave 1: Ch 1–6 (기반 + 소유권/대여/수명)
- [x] Wave 2: Ch 7–11 (데이터 모델링/Trait/Enum/패턴)
- [x] Wave 3: Ch 12–15 (에러 처리/컬렉션/이터레이터)
- [x] Wave 4: Ch 16–18 (동시성/비동기)
- [x] Wave 5: Ch 19–22 (매크로/스마트포인터/FFI/PyO3)

### Phase 2 — 부속 자료
- [x] 서문(Preface) + 이 책을 읽는 법 + 환경 설정 가이드 (frontmatter)
- [x] 부록 A: Cargo 명령어 레퍼런스
- [x] 부록 B: 컴파일러 에러 해독집 (E-codes)
- [x] 부록 C: Python→Rust 표준 라이브러리/관용구 치트시트
- [x] 부록 D: 실전 크레이트 생태계 지도
- [x] 부록 E: Python→Rust 마이그레이션 체크리스트

### Phase 3 — 마무리
- [x] README.md 갱신 (책 소개/내비게이션)
- [x] BOOK_TOC.md를 최종 구조와 동기화
- [x] 분량 검증 (단어 수 집계 → 페이지 환산)
- [x] 최종 커밋/푸시

## Review (작업 후 기록)

### 결과 요약
- **최종 분량**: 서문 1 + 본문 22장 + 부록 5종. 총 약 242,000 한글 음절 + 8,674 코드 줄 + 19,500여 줄.
  A4 추정 ~578페이지(산문 ~372 + 코드 ~206), 표/헤더 포함 시 ~600페이지 목표 도달.
  이 600페이지 추정은 22장 시점의 기록이다. 이후 26장과 부록 보강으로 독자용 본문은 A4 약 800페이지다.
- **확장 배수**: 시작 시 22개 스켈레톤 챕터 합계 ~15,800 단어 → 각 챕터 8,500~12,000 음절로 약 10~13배 심화.
- **품질**: 모든 챕터가 The Python Way → Rust Mental Model → Under the Hood 흐름 준수.
  Python↔Rust 비교표, 메모리 레이아웃 심화, 검증된 벤치마크, 실제 rustc 에러(E0382/E0502/E0277/E0004/E0072/E0133 등) 시연 포함.
  서브에이전트가 모든 Rust 예제를 rustc 1.96/cargo로 컴파일·실행 검증.
- **방법**: 공유 집필 스펙(`chapter_spec.md`)으로 톤·구조 통일 후 병렬 서브에이전트 5웨이브 + 부록 6병렬로 집필.
- **커밋**: 웨이브마다 git-commit-push로 분할 커밋(총 7개 docs/chore 커밋), 모두 origin/main에 푸시 완료.

### 남은 개선 여지 (향후)
- 일부 부록(D/E)이 목표 음절을 다소 초과 — 내용 가치는 충분하나 추후 압축 가능.
- 실제 PDF/제본용 조판(LaTeX/Typst 등) 파이프라인은 범위 외.

---

# TODO (2차) — 완성도·컨텐츠 볼륨 심화 + 신규 추가 (2026-06)

## 목표
이미 출판 수준인 본문을 **희석 없이 더 깊게**(각 챕터 ~30~50% 심화), 그리고
시니어 Python 개발자에게 비어 있는 주제를 **신규 챕터/부록으로 추가**해 완성도를 끌어올린다.
- 전략: 기존 깊게 + 신규 추가 (사용자 확정).
- 보강 요소(연습문제/벤치마크/미니프로젝트/용어집)는 챕터별 가치에 따라 재량 적용.
- 신규 챕터는 **뒤에 덧붙여(23~)** 기존 번호·상호참조를 깨지 않는다.

## Phase A — 감사(Audit)
- [x] 28개 파일 병렬 감사 → DoD 점수 + 파일별 고가치 추가안 블루프린트
- [x] 종합본 확보(워크플로 ws2gn5cnr) → 측정 기반 ground-truth 확정

## Phase B — 기존 심화 확장 (병렬 웨이브, 웨이브마다 commit+push)
- [x] Wave 1: Ch 1–6 (커밋 264fea6)
- [x] Wave 2: Ch 7–11 (커밋 f0fc1e7)
- [x] Wave 3: Ch 12–15 (커밋 409ecca)
- [x] Ch 16 보강: bounded worker queue + backpressure/graceful shutdown 예제
- [x] Ch 17–18 보강: lock sharding, timeout/semaphore/cancellation 예제
- [x] Ch 19–22 보강: trybuild, arena index, FFI ownership, PyO3 wheel/typing/CI 예제
- [x] frontmatter + 부록 A~E 감사/보강
  - frontmatter: Part 9·Ch23–26·부록 F 내비게이션 반영, Python 3.13.6 기준 통일
  - 부록 A: Cargo 품질 게이트/CI 레시피 추가
  - 부록 D: 실전 크레이트 조합 코드 예제 추가
  - 부록 E: PyO3 미니 마이그레이션 프로젝트 예제 추가
  - 부록 B/C: 기존 분량·구조 감사 후 현행 유지

## Phase C — 신규 추가
- [x] Part 9. 프로덕션으로 가는 길 (커밋 7777001)
  - [x] Ch 23. 테스트와 문서화 / Ch 24. CLI / Ch 25. 웹(Axum) / Ch 26. 성능
- [x] 부록 F. 용어집(Glossary) (커밋 0c7bd38)
- [x] README.md · BOOK_TOC.md 내비게이션 동기화 (커밋 de1067f)
- [x] frontmatter 내비게이션(Part 9·Ch23-26·부록 F 링크) 반영

## Phase D — 검증·마무리
- [x] 신규/추가 Rust 코드 컴파일 재검증(rustc/cargo 1.90)
- [x] 상호참조·링크(특히 신규 챕터→기존 챕터, 부록 F→챕터)·용어 일관성 점검
- [x] 분량 재집계(음절/코드/페이지 환산) → README/BOOK_TOC 수치 확정
- [x] 최종 commit/push + Review 기록

## ⚠️ 중단 사유 및 재개 메모 (2026-06-15)
- **세션 사용 한도 도달**(reset: 07:00 Asia/Seoul). 2차 심화 워크플로(wkw74b4z5)와 신규콘텐츠(ws0sz23yd)가 진행 중 한도로 다수 에이전트 실패.
- **현재 커밋된 상태(안전)**: Ch1–15 심화 완료, Part 9(Ch23–26)+부록 F 신규 완료, README/BOOK_TOC 갱신 완료. 모두 origin/main 푸시됨.
- **주의**: 2차 워크플로가 args 필터를 무시하고 Ch1–11을 재처리해 일부 중복 섹션(ch02 크로스컴파일, ch03 추론 정지, ch08 다중상속)이 생겨 **Ch1–11 미커밋 변경은 전량 revert**함(1차 심화본이 정본). Ch16 부분편집·Ch9 재심화도 revert.
- **해소(2026-06-17)**: 위 Phase B 미완료 범위는 수동 감사 후 직접 보강 완료. 중복 재실행 방지를 위해 Ch1–15는 건드리지 않았고, Ch16–22/frontmatter/부록 A/D/E/부록 F 표기/README/BOOK_TOC/tasks만 갱신.

## Review (2차 작업 후 기록, 2026-06-17)

### 결과 요약
- **보강 범위**: Ch16–22, frontmatter, 부록 A/D/E, 부록 F 표기, README/BOOK_TOC, chapter_spec, todo.
- **핵심 추가 예제**:
  - Ch16: bounded worker queue + backpressure + graceful shutdown
  - Ch17: lock splitting/sharding + `DashMap` 판단 기준
  - Ch18: timeout + `Semaphore` + cancellation safety
  - Ch19: `trybuild` compile-fail 테스트와 매크로 에러 UX
  - Ch20: `Rc<RefCell>` 대안인 arena/index 기반 그래프
  - Ch21: FFI `into_raw`/`from_raw`, buffer ABI, panic 경계
  - Ch22: `abi3` wheel, `.pyi`/`py.typed`, maturin CI
  - 부록 D/E: 바로 실행 가능한 크레이트 조합·PyO3 미니 마이그레이션 레시피
- **재집계**: 서문 1 + 본문 26장 + 부록 6종 = 33개 문서, 총 29,027줄.
  한글 음절 324,393자 + 코드 12,939줄 → A4 추정 약 807페이지(산문 499 + 코드 308).
- **검증**:
  - `/tmp/py-to-rs-verify` 임시 크레이트에서 신규 외부 크레이트 예제 `cargo check --bins` 통과(rustc/cargo 1.90).
  - Ch21 FFI 소유권 예제 `rustc --edition 2021 --crate-type lib` 통과.
  - 로컬 마크다운 링크 37개 파일 검사 통과.
  - 코드 펜스 균형 검사 통과.
  - `git diff --check` 통과.
