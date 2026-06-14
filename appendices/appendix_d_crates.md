# 부록 D. 실전 크레이트 생태계 지도

10년 동안 파이썬으로 일하면서 머릿속에 쌓인 라이브러리 지도가 있을 겁니다. "HTTP 요청은 `requests`, 비동기는 `aiohttp`, 데이터는 `pandas`, 직렬화는 `pydantic`, 웹은 `FastAPI`" — 이 반사 신경 덕분에 새 프로젝트를 시작할 때 무엇을 `pip install` 할지 고민하지 않습니다. Rust로 넘어오면 이 지도가 통째로 백지가 됩니다. 표준 라이브러리(std)는 의도적으로 작게 유지되어 있어서 HTTP 클라이언트도, JSON 파서도, 날짜 처리도 들어 있지 않습니다. 처음에는 불친절해 보이지만, 이는 crates.io 생태계가 그 빈자리를 채우되 표준이 특정 구현에 묶이지 않게 하려는 설계 결정입니다.

이 부록은 그 백지 위에 새 지도를 그립니다. 도메인별로 "파이썬에서 쓰던 것 → 대응하는 Rust 크레이트 → 무엇을 언제 쓰는가"를 정리했습니다. 본문의 여러 챕터에서 이미 깊이 다룬 크레이트(serde는 7장, thiserror/anyhow는 13장, tokio/reqwest는 18장, rayon은 15·16장, pyo3는 22장)는 여기서 다시 원리를 설명하지 않습니다. 대신 **선택의 기준**에 집중합니다. 같은 일을 하는 크레이트가 둘 이상일 때 무엇이 갈리는지, 어느 쪽이 사실상의 표준(de facto standard)인지, 어떤 함정이 있는지를 짚습니다.

한 가지 미리 강조할 점이 있습니다. 파이썬에서는 "표준 라이브러리에 있으니 안전하다"는 기준이 통했습니다(`json`, `datetime`, `logging`). Rust에서는 그 기준 대신 **"커뮤니티가 수렴한 크레이트인가"**가 신뢰의 척도입니다. serde가 직렬화의 표준인 것은 std에 있어서가 아니라 생태계 전체가 serde의 `Serialize`/`Deserialize` 트레이트를 중심으로 모였기 때문입니다. 이 부록의 표에서 "사실상 표준"이라고 표기한 크레이트들은, 파이썬으로 치면 표준 라이브러리만큼의 신뢰를 가져도 좋습니다.

> 💡 버전 표기: 아래 표의 크레이트는 모두 활발히 유지보수되고 있으며, 메이저 버전은 시기에 따라 달라질 수 있습니다. `cargo add <크레이트명>`이 항상 최신 호환 버전을 넣어주므로 버전 번호는 외우지 마세요. 대신 "이 도메인엔 이 크레이트"라는 매핑을 기억하는 것이 이 부록의 목적입니다.

---

## D.1 직렬화와 데이터 표현 (Serialization & Data)

파이썬에서 직렬화는 표준 라이브러리 `json`과, 검증까지 얹은 `pydantic`, 그리고 YAML을 위한 `PyYAML`로 나뉘어 있었습니다. Rust는 이 모든 것을 **serde**라는 단일 추상화 아래 통합합니다. serde는 직렬화 *프레임워크*이고, 각 포맷(JSON, YAML, TOML, MessagePack)은 그 프레임워크에 꽂히는 백엔드입니다. 7장에서 `#[derive(Serialize, Deserialize)]`로 Pydantic 모델을 대체하는 과정을 깊이 다뤘으니, 여기서는 포맷별 크레이트의 선택 기준만 정리합니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| (직렬화 프레임워크 없음) | [`serde`](https://crates.io/crates/serde) | **사실상 표준.** 모든 포맷 크레이트가 의존하는 핵심 트레이트(`Serialize`/`Deserialize`) 제공. `features = ["derive"]` 필수. |
| `json` | [`serde_json`](https://crates.io/crates/serde_json) | JSON 직렬화/역직렬화. 동적 JSON은 `serde_json::Value`, 스트리밍은 `from_reader`. 가장 많이 쓰이는 serde 백엔드. |
| `PyYAML` | [`serde_yaml`](https://crates.io/crates/serde_yaml) | YAML 설정 파일. 단, 원작자가 유지보수를 중단(deprecated)했으므로 신규 프로젝트는 [`serde_yaml_ng`](https://crates.io/crates/serde_yaml_ng)나 [`serde_yml`](https://crates.io/crates/serde_yml) 포크를 고려. |
| `toml`(third-party) | [`toml`](https://crates.io/crates/toml) | TOML 파싱. Cargo 자신이 TOML을 쓰므로 생태계 1급 시민. `Cargo.toml`류 설정에 최적. |
| `csv`(stdlib) | [`csv`](https://crates.io/crates/csv) | CSV 읽기/쓰기. serde와 결합해 행을 구조체로 직접 역직렬화. 대용량은 `ReaderBuilder`로 스트리밍. |
| `pickle` | [`rmp-serde`](https://crates.io/crates/rmp-serde) | MessagePack(이진). pickle보다 안전하고 언어 중립적. 네트워크 전송·캐시 직렬화에 JSON보다 빠르고 작음. |
| `pydantic`(검증) | [`serde`](https://crates.io/crates/serde) + [`validator`](https://crates.io/crates/validator) | serde는 역직렬화, validator는 `#[validate(email)]` 같은 필드 검증을 derive로 추가. 둘을 합치면 Pydantic의 검증 역할을 분담. |

**선택 기준 요약**: 사람이 읽고 편집하는 설정은 TOML(Rust 친화적) 또는 YAML, 시스템 간 통신·API는 JSON, 성능과 크기가 중요한 내부 통신·캐시는 MessagePack(`rmp-serde`)입니다. pickle을 쓰던 자리(임의 객체 직렬화)는 보안상 권장하지 않던 패턴이므로, Rust에서는 명시적 스키마 + MessagePack으로 대체하는 것이 정석입니다.

> 🚨 Pydantic 사용자가 가장 자주 놓치는 점: serde는 기본적으로 **검증을 하지 않습니다.** 타입이 맞으면 통과시킵니다. "이메일 형식인가", "0보다 큰가" 같은 의미적 검증은 `validator` 크레이트나 역직렬화 후 별도 로직이 필요합니다. Pydantic이 한 몸에 묶어두었던 "역직렬화 + 검증"이 Rust에서는 두 단계로 분리된다는 점을 기억하세요.

---

## D.2 에러, 로깅, 관측성 (Errors, Logging & Observability)

파이썬은 `logging` 하나로 로깅을, `try/except`로 에러를 처리했습니다. Rust는 에러 *타입* 설계(13장)와 로깅을 명확히 분리하고, 로깅에서도 단순 로그(log)와 구조화된 추적(tracing)을 구분합니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| 커스텀 `Exception` 클래스 | [`thiserror`](https://crates.io/crates/thiserror) | **라이브러리용 에러.** `#[derive(Error)]`로 에러 enum에 `Display`/`From`을 자동 생성. 호출자가 매칭할 수 있는 구체적 에러 타입. (13장) |
| `except Exception` 포괄 처리 | [`anyhow`](https://crates.io/crates/anyhow) | **애플리케이션용 에러.** 타입을 신경 쓰지 않고 `?`로 전파, `.context("...")`로 맥락 추가. 바이너리/main 함수에 적합. (13장) |
| `logging`(기본) | [`log`](https://crates.io/crates/log) | 로깅 **파사드(facade)**. `info!`, `error!` 매크로만 제공하고 실제 출력은 구현체에 위임. 라이브러리는 이것에만 의존. |
| `logging.basicConfig` | [`env_logger`](https://crates.io/crates/env_logger) | `log` 파사드의 가장 단순한 구현. `RUST_LOG=debug` 환경 변수로 레벨 제어. 작은 도구·예제에 충분. |
| (구조화 로깅) | [`tracing`](https://crates.io/crates/tracing) | **사실상 표준(비동기·서비스).** 스팬(span)으로 비동기 작업 경계를 추적, JSON 출력·OpenTelemetry 연동. tokio 생태계 기본. |
| (tracing 출력 설정) | [`tracing-subscriber`](https://crates.io/crates/tracing-subscriber) | tracing 이벤트를 콘솔/JSON/파일로 내보내는 구독자. `EnvFilter`로 모듈별 레벨 제어. |

**선택 기준 요약**: 라이브러리를 만든다면 에러는 `thiserror`, 로깅은 `log` 파사드에만 의존하세요(구현체 강제는 무례합니다). 애플리케이션을 만든다면 에러는 `anyhow`, 로깅은 단순하면 `env_logger`, 비동기 서비스라면 `tracing` + `tracing-subscriber`입니다. 파이썬의 `logging`이 라이브러리와 앱 모두에서 같은 모듈을 쓰던 것과 달리, Rust는 "정책을 정하는 쪽(앱)"과 "로그만 남기는 쪽(라이브러리)"을 코드 수준에서 분리합니다.

> 💡 비동기 서비스에서 `log`만 쓰면 "어느 요청의 로그인지" 알 수 없어 디버깅이 지옥이 됩니다. `tracing`의 스팬은 비동기 태스크가 `.await`로 중단·재개되어도 컨텍스트를 따라다니므로, 파이썬에서 `contextvars`로 요청 ID를 수동으로 끌고 다니던 고통이 사라집니다.

---

## D.3 비동기, 네트워크, 웹 (Async, Network & Web)

18장에서 asyncio와 tokio의 런타임 아키텍처를 비교했습니다. 여기서는 그 위에 올라가는 클라이언트·서버 크레이트의 지형도를 그립니다. 핵심 분기점은 **런타임 선택**입니다: 거의 모든 비동기 크레이트가 tokio를 전제로 하므로, 특별한 이유가 없다면 tokio를 고르면 됩니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `asyncio`(이벤트 루프) | [`tokio`](https://crates.io/crates/tokio) | **사실상 표준 비동기 런타임.** 멀티스레드 스케줄러, 타이머, 채널, I/O. 생태계 대부분이 tokio 가정. (18장) |
| `asyncio`(경량 대안) | [`async-std`](https://crates.io/crates/async-std) | std API를 비동기로 미러링한 런타임. 학습 곡선은 낮지만 생태계 점유율은 tokio에 크게 밀려 신규 채택은 비권장. |
| `requests` | [`reqwest`](https://crates.io/crates/reqwest) | **사실상 표준 HTTP 클라이언트.** 고수준 API, JSON·쿠키·프록시·TLS 내장. 동기(`blocking`)·비동기 모두 지원. (18장) |
| `http.server`/저수준 | [`hyper`](https://crates.io/crates/hyper) | 저수준 HTTP 구현체. reqwest·axum이 내부적으로 사용. 프레임워크를 직접 만들 때만 직접 다룸. |
| `FastAPI` | [`axum`](https://crates.io/crates/axum) | **현대 웹 프레임워크 1순위.** tokio·tower 기반, 추출자(extractor) 패턴이 FastAPI의 의존성 주입과 유사. 타입 안전 라우팅. |
| `Flask`/`Django`(성숙도) | [`actix-web`](https://crates.io/crates/actix-web) | 액터 모델 기반 고성능 프레임워크. 벤치마크 최상위권, 성숙·풍부한 기능. axum과 양대 선택지. |
| `grpcio`/`grpc` | [`tonic`](https://crates.io/crates/tonic) | gRPC 구현체. `.proto`에서 코드 생성(prost), tokio 기반. 파이썬 gRPC 서비스의 Rust 포팅 표준. |
| `websockets`/`websocket-client` | [`tokio-tungstenite`](https://crates.io/crates/tokio-tungstenite) | WebSocket. 동기 코어는 [`tungstenite`](https://crates.io/crates/tungstenite), 비동기 래퍼가 tokio-tungstenite. axum에도 WS 지원 내장. |

**선택 기준 요약**: HTTP 클라이언트는 고민 없이 `reqwest`. 웹 서버는 새 프로젝트라면 `axum`(tokio 생태계와 가장 잘 맞고 학습 자료가 풍부), 극한 성능이나 액터 모델 선호 시 `actix-web`. gRPC는 `tonic`, WebSocket은 axum 내장 또는 `tokio-tungstenite`. `hyper`는 프레임워크를 직접 만들 게 아니라면 직접 만질 일이 없습니다.

> 💡 FastAPI에서 axum으로 넘어올 때 멘탈 모델 매핑: FastAPI의 함수 인자 타입 힌트로 의존성을 주입받던 패턴은 axum의 **추출자(Extractor)**에 대응합니다. `Json<T>`, `Path<T>`, `State<T>`, `Query<T>` 같은 추출자가 핸들러 인자에 선언되면 컴파일 타임에 타입이 검증됩니다. FastAPI가 런타임에 Pydantic으로 검증하던 것을 axum은 컴파일 타임 + serde로 처리합니다.

---

## D.4 동시성과 병렬 처리 (Concurrency & Parallelism)

GIL 때문에 CPU 병렬 처리에 `multiprocessing`을 써야 했고, I/O 병렬엔 `concurrent.futures`를 쓰던 그 영역입니다. 16·17장에서 다룬 "두려움 없는 동시성"의 실전 도구들입니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `multiprocessing.Pool` / `map` | [`rayon`](https://crates.io/crates/rayon) | **데이터 병렬의 결정타.** `.iter()`를 `.par_iter()`로 바꾸면 끝. 작업 훔치기(work-stealing) 스케줄러. GIL 없이 진짜 병렬. (15·16장) |
| (고급 동시성 프리미티브) | [`crossbeam`](https://crates.io/crates/crossbeam) | std `mpsc`보다 빠른 채널(`crossbeam-channel`), 스코프 스레드, 락프리 자료구조. 저수준 동시성 빌딩 블록. |
| `threading.Lock` | [`parking_lot`](https://crates.io/crates/parking_lot) | std `Mutex`/`RwLock`의 더 빠르고 작은 대체. 락 중독(poisoning) 없음. 고경합 환경에서 std보다 우월. |
| 공유 `dict` + 락 | [`dashmap`](https://crates.io/crates/dashmap) | 동시성 HashMap. 샤딩으로 락 경합 최소화. `Arc<Mutex<HashMap>>`를 대체하는 락프리에 가까운 맵. |

**선택 기준 요약**: CPU 바운드 데이터 병렬 처리(이미지 변환, 대량 계산)는 무조건 `rayon`을 먼저 시도하세요 — 한 글자 변경으로 전 코어를 씁니다. 여러 스레드가 같은 맵을 읽고 쓴다면 `Arc<Mutex<HashMap>>` 대신 `dashmap`. 락 성능이 병목이면 std `Mutex`를 `parking_lot::Mutex`로 교체(API 거의 동일). 채널·스코프 스레드 등 정교한 제어가 필요하면 `crossbeam`.

> 💡 `concurrent.futures.ThreadPoolExecutor`로 I/O를 병렬화하던 코드는 Rust에서 갈림길에 섭니다: I/O 바운드면 tokio의 비동기 태스크(`tokio::spawn`)로, CPU 바운드면 rayon으로. 파이썬은 GIL 때문에 둘 다 스레드 풀로 어정쩡하게 처리했지만, Rust는 이 둘을 명확히 다른 도구로 나눕니다.

---

## D.5 CLI와 설정 (CLI & Configuration)

`argparse`/`click`으로 CLI를, `python-dotenv`로 환경 변수를, `tqdm`으로 진행 막대를 그리던 자리입니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `argparse` / `click` | [`clap`](https://crates.io/crates/clap) | **사실상 표준 CLI.** `#[derive(Parser)]`로 구조체에서 인자 파서 자동 생성. 서브커맨드·자동 `--help`·셸 자동완성. |
| `configparser` / 다중 소스 설정 | [`config`](https://crates.io/crates/config) | 파일(TOML/YAML/JSON)·환경 변수·기본값을 계층적으로 병합. 12-factor 앱 설정 패턴. serde로 구조체에 매핑. |
| `python-dotenv` | [`dotenvy`](https://crates.io/crates/dotenvy) | `.env` 파일을 환경 변수로 로드. 원조 `dotenv`의 유지보수 포크. 개발 환경 시크릿 주입. |
| `tqdm` | [`indicatif`](https://crates.io/crates/indicatif) | 진행 막대·스피너. 멀티 막대(`MultiProgress`)로 병렬 작업 시각화. rayon·tokio와 함께 자주 사용. |

**선택 기준 요약**: CLI는 고민 없이 `clap`(derive 매크로 방식이 `click`의 데코레이터만큼 선언적입니다). 설정 소스가 여러 개(파일 + 환경 변수 + CLI 인자)면 `config`로 계층 병합. `.env` 로딩만 필요하면 `dotenvy` 한 줄. 진행 표시는 `indicatif`.

> 💡 `clap`의 derive 방식은 파이썬 `click`보다 한 발 더 나아갑니다. CLI 인자 명세가 곧 Rust 구조체 정의이므로, 파싱 후 받는 값이 이미 타입이 보장된 구조체입니다. `argparse`가 반환하던 `Namespace`(속성이 동적이라 IDE 자동완성이 약했던)와 달리, 모든 필드가 컴파일 타임에 검증됩니다.

---

## D.6 데이터 과학과 수치 계산 (Data Science & Numerics)

`numpy`/`pandas`의 영역입니다. 솔직히 말하면, **이 영역은 Rust가 파이썬을 완전히 대체하기보다 보완하는 쪽이 현실적**입니다. 탐색적 분석과 시각화는 여전히 파이썬이 강하고, Rust는 성능 병목 구간(22장의 PyO3 시나리오)이나 데이터 파이프라인 백엔드에서 빛납니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `numpy`(N차원 배열) | [`ndarray`](https://crates.io/crates/ndarray) | N차원 배열·브로드캐스팅·슬라이싱. numpy의 `ndarray`에 가장 가까운 API. BLAS 연동 가능. |
| `pandas`(데이터프레임) | [`polars`](https://crates.io/crates/polars) | **고성능 데이터프레임.** Apache Arrow 기반, 지연 평가(lazy)·쿼리 최적화. 대용량에서 pandas보다 수 배~수십 배 빠름. 파이썬 바인딩도 인기. |
| `numpy`(선형대수) | [`nalgebra`](https://crates.io/crates/nalgebra) | 선형대수 특화: 행렬·벡터·쿼터니언·변환. 그래픽스·로보틱스·시뮬레이션. 컴파일 타임 차원 검사 가능. |
| numpy ↔ Rust 변환 | [`rust-numpy`](https://crates.io/crates/numpy) | PyO3와 함께 파이썬 numpy 배열을 Rust `ndarray`로 무복사(zero-copy) 접근. (22장) |

**선택 기준 요약**: 범용 N차원 배열 연산은 `ndarray`, 표 형식 데이터·ETL·분석은 `polars`(pandas 사용자가 가장 만족하는 전환), 선형대수·기하 변환은 `nalgebra`. 파이썬에서 이미 numpy로 돌아가는 코드의 병목만 가속하려면 코드를 다 옮기지 말고 `rust-numpy` + PyO3로 핵심 함수만 Rust로 빼는 것이 비용 대비 효과가 가장 큽니다.

> 💡 `polars`는 단순히 "빠른 pandas"가 아닙니다. 지연 평가(`LazyFrame`)로 전체 쿼리를 먼저 받아 최적화한 뒤 실행하므로, pandas에서 중간 결과를 매번 메모리에 만들던 낭비가 사라집니다. SQL의 쿼리 플래너가 데이터프레임에 들어왔다고 생각하면 됩니다. 파이썬에서도 `pip install polars`로 쓸 수 있으니, Rust 전환 전 맛보기로 좋습니다.

---

## D.7 데이터베이스 (Databases)

`SQLAlchemy`/`psycopg2`로 RDB를, `redis-py`로 캐시를 다루던 자리입니다. Rust DB 크레이트의 핵심 분기점은 **"컴파일 타임에 SQL을 검증할 것인가, 런타임 유연성을 택할 것인가"**입니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `SQLAlchemy Core` / 비동기 | [`sqlx`](https://crates.io/crates/sqlx) | **비동기 SQL 툴킷.** ORM 아님. `query!` 매크로가 **컴파일 타임에 실제 DB 스키마로 SQL 검증.** Postgres·MySQL·SQLite. tokio 친화. |
| `SQLAlchemy ORM` | [`diesel`](https://crates.io/crates/diesel) | 컴파일 타임 안전 ORM·쿼리 빌더. 강력한 타입 보장, 마이그레이션 도구 내장. 기본은 동기(비동기 확장 별도). |
| `SQLAlchemy ORM`(비동기·동적) | [`sea-orm`](https://crates.io/crates/sea-orm) | sqlx 위에 올린 비동기 ORM. 동적 쿼리·액티브 레코드 패턴. SQLAlchemy ORM 사용 경험과 가장 유사. |
| `sqlite3`(stdlib) | [`rusqlite`](https://crates.io/crates/rusqlite) | SQLite 직접 바인딩. 동기·경량. 임베디드 DB·로컬 도구·테스트에 적합. |
| `redis-py` | [`redis`](https://crates.io/crates/redis) | Redis 클라이언트. 동기·비동기(tokio) 모두 지원. 커넥션 풀은 `deadpool-redis` 등과 조합. |

**선택 기준 요약**: 비동기 웹 서비스에서 SQL을 직접 쓰는 쪽이 편하면 `sqlx`(컴파일 타임 SQL 검증이 강력합니다 — 오타 난 컬럼명이 빌드를 막습니다). SQLAlchemy ORM의 객체 매핑·관계 추상화가 그립다면 `sea-orm`(비동기) 또는 `diesel`(동기, 최강 타입 안전성). 로컬 임베디드 SQLite는 `rusqlite`. Redis는 `redis` 크레이트.

> 🚨 `sqlx`의 `query!` 매크로는 빌드 시점에 실제 DB에 연결해 스키마를 검증합니다(또는 `cargo sqlx prepare`로 미리 캐시한 메타데이터 사용). CI에서 DB가 없으면 빌드가 깨질 수 있으니, `SQLX_OFFLINE=true` + `.sqlx` 캐시 디렉터리를 커밋하는 오프라인 모드를 처음부터 설정하세요. 파이썬 ORM에는 없던 "DB가 빌드 의존성"이라는 개념에 처음엔 당황할 수 있습니다.

---

## D.8 텍스트와 파싱 (Text & Parsing)

`re` 모듈로 정규식을 쓰고, 복잡한 문법은 `ply`/`lark`로 파싱하던 자리입니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `re` | [`regex`](https://crates.io/crates/regex) | **사실상 표준 정규식.** 선형 시간 보장(파국적 백트래킹 없음). 백레퍼런스·룩어라운드는 의도적으로 미지원(성능 보장 위해). |
| (룩어라운드 필요 시) | [`fancy-regex`](https://crates.io/crates/fancy-regex) | `regex`가 못 하는 백레퍼런스·룩어라운드 지원. 단 최악의 경우 지수 시간 가능. 정말 필요할 때만. |
| `ply` / 손수 짠 파서 | [`nom`](https://crates.io/crates/nom) | 파서 컴비네이터(parser combinator). 작은 파서를 조합해 큰 파서를 구성. 바이너리·텍스트 포맷 파싱. 빠름. |
| `lark` / `pyparsing` | [`pest`](https://crates.io/crates/pest) | PEG 문법을 `.pest` 파일에 선언적으로 작성. 문법과 코드 분리. 가독성 우선일 때 nom보다 직관적. |

**선택 기준 요약**: 일반 정규식은 `regex` — 단, 파이썬 `re`와 달리 **백레퍼런스·룩어라운드가 없습니다**(이것이 선형 시간 보장의 대가입니다). 그 기능이 꼭 필요하면 `fancy-regex`. 정규식으로 표현하기 버거운 구조적 문법(설정 언어, DSL, 바이너리 포맷)은 파서를 작성하는데, 코드로 조합하는 스타일이면 `nom`, 별도 문법 파일을 선호하면 `pest`.

> 💡 `regex`가 백레퍼런스를 빼버린 것은 기능 부족이 아니라 **보안 결정**입니다. 파이썬 `re`는 `(a+)+$` 같은 패턴에 악의적 입력이 들어오면 지수 시간으로 폭발(ReDoS)할 수 있지만, Rust `regex`는 RE2 알고리즘으로 입력 길이에 선형인 시간을 보장합니다. 사용자 입력으로 정규식을 받는 서비스라면 이 차이가 곧 가용성입니다.

---

## D.9 시간, 난수, 암호 (Time, Random & Crypto)

`datetime`, `random`, `hashlib`/`secrets`로 처리하던 자리입니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `datetime` | [`chrono`](https://crates.io/crates/chrono) | 날짜·시간·타임존(`chrono-tz`)·파싱·포매팅. 가장 널리 쓰이는 시간 크레이트. RFC3339·strftime 호환. |
| `datetime`(현대적 대안) | [`time`](https://crates.io/crates/time) | 더 작고 안전한 설계의 시간 크레이트. std·일부 생태계가 채택. 기능은 chrono보다 좁지만 의존성이 가벼움. |
| `random` | [`rand`](https://crates.io/crates/rand) | 난수 생성·분포·셔플. `thread_rng()`가 기본. 암호학적 난수는 `OsRng`. |
| `hashlib`(sha256 등) | [`sha2`](https://crates.io/crates/sha2) | SHA-256/512. `RustCrypto` 계열의 순수 Rust 구현. 무결성 해시·체크섬. |
| `hashlib`/`cryptography` | [`ring`](https://crates.io/crates/ring) | 종합 암호 라이브러리(서명·AEAD·해시). BoringSSL 기반, 검증된 구현. TLS·서명 등 실전 암호. |
| `secrets`(보안 토큰) | [`rand`](https://crates.io/crates/rand) + `OsRng` / [`getrandom`](https://crates.io/crates/getrandom) | OS 엔트로피 기반 보안 난수. 세션 토큰·솔트 생성. 일반 `thread_rng`와 구분해 사용. |

**선택 기준 요약**: 시간 처리는 기능이 풍부한 `chrono`가 무난하고, 의존성을 최소화하거나 std와 맞물리는 환경이면 `time`. 게임·시뮬레이션 등 일반 난수는 `rand`, 토큰·솔트 같은 **보안용 난수는 반드시 `OsRng`/`getrandom`** 계열(`thread_rng`는 보안용이 아닙니다). 단순 해시는 `sha2`, 서명·암호화·TLS 등 본격 암호는 검증된 `ring`을 쓰고 절대 직접 구현하지 마세요.

> 🚨 파이썬에서 `random.random()`으로 토큰을 만들면 안 된다는 걸 알았듯이, Rust에서도 `rand::thread_rng()`는 암호학적으로 안전하지 않을 수 있는 빠른 PRNG입니다. 비밀번호 솔트, 세션 토큰, API 키처럼 예측 불가능성이 보안과 직결되는 곳에는 `OsRng`(파이썬 `secrets` 대응)를 명시적으로 쓰세요.

---

## D.10 테스트와 벤치마크 (Testing & Benchmarking)

`pytest`, `hypothesis`, `unittest.mock`으로 하던 일입니다. 기본 테스트는 Cargo에 내장(`#[test]`, `cargo test`)되어 있어 별도 크레이트가 필요 없다는 점이 파이썬과 큰 차이입니다.

| Python | Rust 크레이트 | 용도 / 비고 |
|--------|--------------|------------|
| `pytest`(기본 러너) | (내장 `cargo test`) | 테스트 러너가 표준 도구에 내장. `#[test]` 어트리뷰트만 붙이면 됨. 별도 설치 불필요. |
| `timeit` / `pytest-benchmark` | [`criterion`](https://crates.io/crates/criterion) | **통계적 벤치마크.** 워밍업·이상치 제거·회귀 탐지·그래프 생성. 본문의 모든 벤치 수치의 신뢰 기반. |
| `hypothesis` | [`proptest`](https://crates.io/crates/proptest) | 속성 기반 테스트(property-based). 무작위 입력 생성 + 실패 케이스 축소(shrinking). hypothesis와 철학 동일. |
| `unittest.mock` | [`mockall`](https://crates.io/crates/mockall) | 트레이트 기반 목(mock) 자동 생성. `#[automock]`으로 인터페이스의 가짜 구현 생성. 의존성 격리 테스트. |
| (스냅샷 테스트) | [`insta`](https://crates.io/crates/insta) | 스냅샷 테스트. 출력 결과를 파일로 저장·비교. `cargo insta review`로 변경 검토. 회귀 방지에 강력. |

**선택 기준 요약**: 단위 테스트는 내장 `cargo test`로 충분합니다(파이썬처럼 pytest를 따로 깔 필요 없음). 성능 회귀를 막으려면 `criterion`(본문 벤치마크가 모두 이것 기반). 엣지 케이스를 자동 탐색하려면 `proptest`(hypothesis 경험자에게 친숙). 트레이트 의존성을 격리하려면 `mockall`. 복잡한 출력(직렬화 결과, 렌더링 텍스트)의 회귀 검증은 `insta`.

> 💡 Rust 테스트 문화의 특징: 작은 단위 테스트는 소스 파일 안 `#[cfg(test)] mod tests`에 함께 두고, 통합 테스트는 `tests/` 디렉터리에 둡니다. 또한 `///` 문서 주석 안의 코드 예제가 `cargo test`로 **실제 실행·검증**됩니다(doctest). 파이썬 `doctest`와 같은 개념이지만 생태계 전체가 적극 활용한다는 점이 다릅니다 — docs.rs의 모든 예제가 컴파일·검증된 코드라는 신뢰가 여기서 나옵니다.

---

## D.11 파이썬 연동 (Python Interop)

기존 파이썬 코드를 다 버릴 필요는 없습니다. 22장에서 깊이 다룬 대로, 병목 구간만 Rust로 빼서 네이티브 확장으로 호출하는 것이 가장 실용적인 도입 경로입니다.

| 목적 | Rust 크레이트 / 도구 | 용도 / 비고 |
|------|---------------------|------------|
| 파이썬에서 Rust 호출 | [`pyo3`](https://crates.io/crates/pyo3) | **사실상 표준.** Rust 함수·구조체를 파이썬 모듈로 노출. GIL 관리·타입 변환·예외 매핑. (22장) |
| 빌드·패키징 | [`maturin`](https://github.com/PyO3/maturin) | PyO3 확장을 빌드해 wheel로 패키징·배포. `pip install`로 설치 가능한 네이티브 모듈 생성. (22장) |
| numpy 무복사 연동 | [`rust-numpy`](https://crates.io/crates/numpy) | 파이썬 numpy 배열을 Rust `ndarray`로 zero-copy 접근. 수치 병목 가속의 핵심. (D.6, 22장) |

**선택 기준 요약**: 점진적 도입의 정석 — 프로파일러로 파이썬 병목을 찾고(예: pandas 그룹 연산, 무거운 루프), 그 함수만 PyO3 + maturin으로 Rust 확장으로 만들어 `import`합니다. 호출부의 파이썬 코드는 그대로 둡니다. numpy 배열을 주고받으면 `rust-numpy`로 복사 없이 메모리를 공유합니다. 22장의 "pandas 50배 최적화" 예제가 바로 이 패턴입니다.

> 💡 반대 방향(Rust에서 파이썬 호출)도 같은 `pyo3`로 가능합니다. 예를 들어 Rust 서비스에서 파이썬으로 작성된 ML 모델만 호출하고 싶을 때 `Python::with_gil`로 인터프리터를 임베드합니다. 다만 GIL이 다시 등장하므로 성능 이점은 제한적입니다 — 가능하면 "파이썬이 Rust를 호출"하는 방향을 우선하세요.

---

## D.12 크레이트 고르는 법 (How to Choose a Crate)

지도에 없는 도메인을 만났을 때, 또는 위 표의 선택지가 마음에 들지 않을 때 스스로 판단하는 기준이 필요합니다. 파이썬에서 PyPI 패키지를 고를 때 GitHub 스타·최근 커밋·다운로드 수를 봤듯이, Rust에도 동일한(그러나 더 표준화된) 판단 도구가 있습니다.

### 어디서 찾는가

| 사이트 | 파이썬 대응 | 용도 |
|--------|------------|------|
| [crates.io](https://crates.io) | PyPI | 공식 레지스트리. 다운로드 수·버전 이력·라이선스 확인. |
| [lib.rs](https://lib.rs) | (없음) | crates.io의 큐레이션 대안. **카테고리별 인기·활성도 순위**가 한눈에 보여 "이 도메인의 표준"을 찾기에 최적. |
| [docs.rs](https://docs.rs) | readthedocs(자동화 버전) | 모든 크레이트의 API 문서가 **자동 생성·게시**. 게시된 모든 버전의 문서가 영구 보존. |
| [blessed.rs](https://blessed.rs) | (없음) | 도메인별 "추천 크레이트" 큐레이션. 입문자가 표준을 빠르게 파악하는 데 유용. |

### 무엇을 확인하는가

크레이트를 채택하기 전 체크리스트입니다(파이썬 의존성 심사와 본질은 같되, Rust에서는 더 정량적으로 볼 수 있습니다):

- **다운로드 추세**: crates.io의 "Recent Downloads"가 꾸준한가, 한물간 크레이트가 아닌가. 절대 수보다 **추세**가 중요합니다.
- **유지보수 활성도**: 마지막 릴리스 시점, 이슈·PR 응답 속도, "deprecated" 표시 여부(`serde_yaml`처럼 표준이었다가 유지보수 중단된 사례가 있습니다).
- **문서 품질**: docs.rs에 예제가 풍부한가, 모듈 수준 설명(`//!`)이 있는가. Rust는 문서화 문화가 강해, 문서가 빈약하면 그 자체로 경고 신호입니다.
- **의존성 트리 크기**: `cargo tree`로 그 크레이트가 끌고 오는 의존성을 확인. 작은 기능에 100개 의존성을 끌어오면 빌드 시간·공급망 위험이 커집니다.
- **다운스트림 의존자(Dependents)**: crates.io의 "Dependents" 탭. 다른 유명 크레이트들이 의존한다면 사실상 검증된 것입니다(reqwest가 의존하면 신뢰해도 좋습니다).
- **`1.0` 도달 여부**: SemVer상 `1.0` 이전(`0.x`)은 호환성 깨짐(breaking change)이 잦을 수 있습니다. 다만 `tokio` 이전의 많은 핵심 크레이트가 오래 `0.x`였듯, 버전 숫자만으로 단정하지 말고 활성도와 함께 보세요.

### 의존성을 최소화하라

파이썬에서 `requirements.txt`가 비대해져도 런타임에 큰 비용이 없었지만, Rust에서는 **모든 의존성이 컴파일 대상**입니다. 의존성 하나가 빌드 시간, 바이너리 크기, 공급망 공격 표면(2장에서 본 `Cargo.lock`의 체크섬이 방어하는 그 표면)에 직접 더해집니다.

- **표준 라이브러리로 충분한가 먼저 묻기**: 작은 유틸리티는 std에 이미 있는 경우가 많습니다. 한 줄짜리 기능에 크레이트를 추가하지 마세요.
- **기능 플래그(feature flags) 활용**: tokio처럼 거대한 크레이트는 `features`로 필요한 부분만 켭니다. `default-features = false`로 시작해 필요한 것만 추가하면 빌드가 가벼워집니다.
- **`cargo tree --duplicates`로 중복 버전 점검**: 같은 크레이트의 여러 메이저 버전이 동시에 빌드되면 바이너리가 비대해집니다.
- **`cargo audit`로 취약점 점검**: `RustSec` 자문 DB와 대조해 알려진 취약점을 가진 의존성을 찾습니다. CI에 넣어두세요(파이썬 `pip-audit` 대응).

> 💡 마지막 조언: 이 부록의 표는 출발점이지, 정답지가 아닙니다. 생태계는 움직입니다 — 어제의 표준이 오늘 deprecated가 되고(`serde_yaml`, `dotenv`), 새 강자가 등장합니다(`polars`, `axum`). 그러나 판단 *방법*은 변하지 않습니다. lib.rs에서 카테고리를 열고, 다운로드 추세와 유지보수 활성도를 보고, `cargo tree`로 의존성을 확인하고, 다운스트림 의존자를 살피세요. 10년간 PyPI에서 좋은 패키지를 골라낸 그 감각은 그대로 이전됩니다 — 다만 Rust에서는 그 감각을 뒷받침할 정량적 도구가 더 잘 갖춰져 있습니다.

이 부록의 지도를 손에 들었다면, 이제 파이썬에서 하던 거의 모든 작업의 Rust 대응물을 알고 있는 셈입니다. 비어 있던 백지가 채워졌습니다. 남은 것은 실제로 `cargo add`를 입력하고, docs.rs를 열고, 첫 줄을 컴파일하는 일뿐입니다.

---

## References

- https://crates.io/
- https://lib.rs/
- https://blessed.rs/crates
- https://docs.rs/
- https://www.arewewebyet.org/
- https://areweasyncyet.rs/
