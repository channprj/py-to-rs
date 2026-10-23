# Chapter 25. 웹 서비스: FastAPI에서 Axum으로

![Chapter 25. 웹 서비스: FastAPI에서 Axum으로](../images/chapter-25-web.jpg)

10년 차 Python 백엔드 엔지니어에게 FastAPI는 거의 종착지처럼 느껴집니다. 타입 힌트만 달면 Pydantic이 요청 본문을 검증해 주고, 의존성 주입(`Depends`)으로 인증·DB 세션을 깔끔하게 끼워 넣고, `uvicorn`으로 ASGI 워커를 띄우면 끝입니다. 생산성이라는 잣대에서 FastAPI는 흠잡을 데가 거의 없습니다. 그런데 그 서비스가 진짜로 트래픽을 받기 시작하면, 익숙한 고통의 목록이 하나씩 깨어납니다.

가장 먼저 부딪히는 벽은 **단일 프로세스가 한 코어밖에 못 쓴다**는 사실입니다(Chapter 18에서 본 GIL과 단일 스레드 이벤트 루프의 그 한계입니다). 그래서 우리는 `gunicorn -k uvicorn.workers.UvicornWorker -w 8`처럼 워커 프로세스를 코어 수만큼 띄웁니다. 그런데 워커마다 인터프리터·임포트된 모듈·ML 모델 가중치가 통째로 복제되어 메모리를 8배 먹습니다. 32GB 인스턴스가 워커 8개에 모델을 4GB씩 물고 있으면 그 순간 OOM이 새벽에 호출기를 울립니다. 워커 수를 줄이면 처리량이 떨어지고, 늘리면 메모리가 터지는, 출구 없는 줄다리기입니다.

두 번째 고통은 **검증과 직렬화가 공짜가 아니라는 것**입니다. Pydantic v2가 검증 엔진을 Rust(`pydantic-core`)로 다시 써서 v1보다 훨씬 빨라졌지만(Chapter 1, Chapter 7에서 본 그 이야기입니다), 검증을 통과한 데이터를 다시 Python 객체로 빚어내는 경계 비용, 응답을 JSON으로 직렬화하는 비용은 여전히 핸들러마다 청구됩니다. p99 레이턴시를 프로파일링하면 비즈니스 로직이 아니라 직렬화·역직렬화에 시간이 깔려 있는 경우가 흔합니다.

세 번째는 더 은밀합니다. **어떤 호출이 이벤트 루프를 얼리는지 컴파일러가 알려주지 않습니다.** FastAPI 핸들러 안에서 무심코 동기 DB 드라이버나 `time.sleep`, 무거운 CPU 루프를 호출하면, 그 한 줄이 워커 하나의 이벤트 루프를 통째로 멈춰 세웁니다. Chapter 18에서 해부한 바로 그 사고가 웹 핸들러 안에서 매일 재현됩니다. 헬스체크가 타임아웃되고, 오토스케일러가 미쳐 날뛰고, 범인을 찾는 데 며칠이 걸립니다.

이번 챕터에서는 같은 웹 서비스를 Rust의 **Axum**으로 옮깁니다. Axum은 `tokio`(Chapter 18) 위에서 도는 웹 프레임워크로, FastAPI의 핵심 발상 — 타입 기반 의존성 주입, 선언적 라우팅, 자동 검증 — 을 거의 그대로 가져오되, 그 전부를 **런타임 인트로스펙션이 아니라 컴파일 타임 트레이트 디스패치**로 구현합니다. 핸들러는 `async fn`이고, 익스트랙터(Extractor)는 `FromRequest`/`FromRequestParts` 트레이트이며, 공유 상태는 `State(Arc<...>)`(Chapter 17)이고, 에러를 HTTP 응답으로 바꾸는 것은 `IntoResponse`(Chapter 13)입니다. Chapter 7에서 만든 serde 모델을 한 글자도 고치지 않고 요청/응답 타입으로 재사용하며, Chapter 22의 집계 서비스를 REST 엔드포인트로 노출합니다. 그리고 늘 그렇듯, "어떻게 쓰는가"보다 "왜 익스트랙터가 런타임 비용 없이 동작하는가", "왜 핸들러 반환 타입이 `IntoResponse`여야 하는가"를 트레이트와 메모리 수준에서 파고듭니다.

---

## 25.1 The Python Way — FastAPI, Pydantic, 그리고 워커 프로세스

먼저 우리가 익숙한 방식을 정확히 짚고 넘어갑시다. 센서 측정값을 받아 누적 집계하는 작은 서비스를 FastAPI로 작성하면 이렇습니다. Chapter 22의 집계 로직(`"id,value"`를 합산하던 그 코드)을 웹 API로 감싼 모습입니다.

```python
from __future__ import annotations
from collections import defaultdict
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

# 인메모리 공유 상태 (id -> [합계, 개수])
_totals: dict[int, list[float]] = defaultdict(lambda: [0.0, 0])

class Reading(BaseModel):
    sensor_id: int
    value: float

class Aggregate(BaseModel):
    sensor_id: int
    total: float
    count: int

@app.post("/readings")
async def ingest(reading: Reading) -> Aggregate:
    if reading.value != reading.value or reading.value in (float("inf"), float("-inf")):
        raise HTTPException(status_code=400, detail="value는 유한수여야 합니다")
    slot = _totals[reading.sensor_id]
    slot[0] += reading.value
    slot[1] += 1
    return Aggregate(sensor_id=reading.sensor_id, total=slot[0], count=slot[1])

@app.get("/aggregate/{sensor_id}")
async def get_aggregate(sensor_id: int) -> Aggregate:
    if sensor_id not in _totals:
        raise HTTPException(status_code=404, detail=f"센서 {sensor_id} 없음")
    total, count = _totals[sensor_id]
    return Aggregate(sensor_id=sensor_id, total=total, count=int(count))
```

이 코드는 명료하고, 타입 힌트가 곧 검증 규칙이며, 5분 안에 작성됩니다. 이것이 FastAPI의 위대함입니다. 그런데 이 우아함을 떠받치는 런타임 메커니즘을 들여다보면, 비용이 어디 숨어 있는지 보입니다.

`async def ingest(reading: Reading)` — FastAPI는 서버가 뜰 때 이 함수의 **시그니처를 런타임에 들여다봅니다(introspection).** `inspect.signature()`로 파라미터 목록을 읽고, 각 파라미터의 타입 어노테이션(`Reading`)을 보고 "이건 요청 본문(body)이구나", `sensor_id: int` 같은 경로 변수는 "이건 path parameter구나" 하고 **실행 중에** 분류합니다. 요청이 들어올 때마다 이 분류 결과에 따라 본문을 읽고, Pydantic으로 검증하고, 키워드 인자로 핸들러를 호출합니다. 이 시그니처 파싱은 시작 시 한 번이지만, 그 결과를 들고 매 요청마다 동적으로 디스패치하는 비용은 요청마다 발생합니다.

`Depends(get_current_user)` 같은 의존성 주입도 같은 원리입니다. FastAPI는 의존성 그래프를 런타임에 구성하고, 요청마다 그 그래프를 순회하며 각 의존성 함수를 호출합니다. 강력하지만, 전부 **인터프리터가 실행 중에** 하는 일입니다. 타입 힌트는 런타임까지 살아남아 디스패치에 쓰이지만, 그 디스패치는 동적입니다.

그리고 25장 도입부에서 말한 그 한계들이 여기 그대로 적용됩니다. `_totals`라는 전역 딕셔너리는 단일 워커 프로세스 안에서만 공유됩니다. 워커를 8개 띄우면 각 워커가 자기만의 `_totals`를 갖게 되어, 같은 센서의 측정값이 워커마다 흩어져 집계가 틀립니다. 진짜 공유 상태를 원하면 Redis 같은 외부 저장소로 빼야 합니다 — 인메모리로는 멀티코어를 못 쓰기 때문입니다. Rust는 바로 이 지점, **단일 프로세스 안에서 여러 코어가 하나의 인메모리 상태를 안전하게 공유하는 것**을 가능하게 합니다.

| FastAPI 개념 | 런타임에 일어나는 일 |
|--------------|---------------------|
| `def ingest(reading: Reading)` | 시작 시 시그니처 인트로스펙션 → 요청마다 동적 분류·호출 |
| `Depends(dep)` | 런타임 의존성 그래프 순회 |
| Pydantic 검증 | `pydantic-core`(Rust) 호출 + Python 객체 빌드 경계 비용 |
| `-w 8` 워커 | 프로세스 8개 = 메모리 8배, 상태 공유 불가(외부 저장소 필요) |
| 블로킹 호출 | 컴파일러 경고 없음 → 런타임에 이벤트 루프 정지 |

---

## 25.2 The Rust Mental Model — 라우터는 타입의 합, 핸들러는 함수

Axum은 FastAPI와 같은 그림을 그리되, 모든 것을 타입으로 옮깁니다. 핵심 발상은 한 문장입니다: **"요청에서 무엇을 뽑아낼지를 핸들러의 인자 타입으로 선언하고, 응답으로 무엇을 돌려줄지를 반환 타입으로 선언한다. 나머지는 컴파일러가 채운다."**

```rust
use axum::{Json, extract::{Path, State}};

// 핸들러는 그냥 async fn이다. 인자 타입이 "무엇을 추출할지"를 선언한다.
async fn get_aggregate(
    State(state): State<SharedState>,   // 공유 상태에서 (의존성 주입)
    Path(id): Path<i64>,                 // URL 경로에서 {id}
) -> Result<Json<Aggregate>, ApiError> { // 성공이면 JSON, 실패면 에러 응답
    // ...
}
```

FastAPI에서 `reading: Reading`이라고 쓰면 "본문에서 뽑아 검증해 달라"는 뜻이듯, Axum에서 `Json(reading): Json<Reading>`이라고 쓰면 같은 뜻입니다. `Path(id): Path<i64>`는 경로 변수, `Query(f): Query<Filter>`는 쿼리 스트링, `State(s): State<T>`는 공유 상태입니다. 차이는 **이 분류가 런타임 인트로스펙션이 아니라 컴파일 타임 트레이트 구현으로 결정된다**는 점입니다. `Path`, `Query`, `Json`, `State`는 모두 `FromRequest`(또는 `FromRequestParts`) 트레이트를 구현하는 타입이고, 컴파일러는 핸들러의 각 인자 타입을 보고 "이 타입의 추출 코드를 호출하는" 디스패치를 **컴파일 시점에 생성**합니다. 요청이 들어왔을 때 "이 인자가 path인지 body인지"를 다시 알아내는 일은 없습니다 — 이미 기계어에 박혀 있습니다.

라우터는 경로와 핸들러를 엮는 선언입니다. FastAPI의 `@app.post("/readings")` 데코레이터에 대응하는 것이 `Router::new().route("/readings", post(ingest))`입니다.

```rust
use axum::{Router, routing::{get, post}};

let app = Router::new()
    .route("/health", get(health))
    .route("/readings", post(ingest))
    .route("/aggregate/{id}", get(get_aggregate))  // {id}가 경로 변수
    .with_state(state);                              // 모든 핸들러가 공유할 상태 주입
```

데코레이터가 함수를 전역 레지스트리에 런타임 등록하는 대신, Axum은 `Router`라는 **값(value)**을 빌드합니다. `.route(...)`는 라우터를 받아 새 라우터를 돌려주는 평범한 메서드 체이닝이고, `.with_state(state)`는 그 라우터에 상태 타입을 못 박습니다. 이 모든 조립이 끝나면 하나의 `Router` 값이 만들어지고, `axum::serve`가 그것으로 요청을 처리합니다.

여기서 FastAPI 사용자가 즉시 체감할 차이는 **검증 결과의 타입**입니다. FastAPI에서 검증 실패는 `RequestValidationError` 예외로 던져지고 미들웨어가 422 응답으로 바꿉니다. Axum에서 `Json<Reading>` 추출이 실패하면(JSON이 깨졌거나 필드 타입이 안 맞으면), 그 익스트랙터가 **자기 자신을 응답으로 변환할 줄 압니다** — `FromRequest`의 `Rejection` 타입이 `IntoResponse`라서, 추출 실패가 곧 적절한 HTTP 응답이 됩니다. 예외 던지기와 잡기가 아니라, "추출 결과는 `Result`이고, 그 `Err`은 응답이 될 수 있는 타입"이라는 Chapter 12~13의 `Result` 철학이 웹 계층까지 일관되게 흐릅니다.

| FastAPI | Axum | 결정 시점 |
|---------|------|-----------|
| `@app.post("/x")` 데코레이터 | `.route("/x", post(handler))` | 컴파일 타임(값 조립) |
| `async def handler(...)` | `async fn handler(...)` | — |
| `reading: Reading` (본문) | `Json(reading): Json<Reading>` | 컴파일 타임 트레이트 |
| `sensor_id: int` (경로) | `Path(id): Path<i64>` | 컴파일 타임 트레이트 |
| `q: str = Query(...)` | `Query(q): Query<Filter>` | 컴파일 타임 트레이트 |
| `Depends(get_state)` | `State(s): State<T>` | 컴파일 타임 트레이트 |
| `Depends(get_api_key)` | 커스텀 `FromRequestParts` 익스트랙터 | 컴파일 타임 트레이트 |
| `raise HTTPException(404)` | `Err(ApiError::NotFound(..))` + `IntoResponse` | 컴파일 타임 트레이트 |
| `BaseModel` 검증 | `serde` `Deserialize` + `Json` 익스트랙터 | 컴파일 타임 코드 생성 |
| `-w 8` 워커 프로세스 | 단일 프로세스 멀티스레드 `tokio` | 런타임(스케줄러) |

> 이 챕터의 코드는 **axum 0.8** 기준입니다. 25.8의 서버와 통합 테스트는 2026-10-22에 rustc 1.99.1, axum 0.8.9, tokio 1.54.0, edition 2021에서 `cargo test`가 통과했습니다. 처음 검증 기록은 rustc 1.90.0입니다. axum 0.7과 0.8 사이에 경로 변수 문법이 `/:id`에서 `/{id}`로 바뀌었고, 커스텀 익스트랙터에서 `#[async_trait]`가 더는 필요 없어졌습니다(트레이트 안의 `async fn`을 언어가 직접 지원). 0.7 코드를 보고 있다면 이 두 가지를 먼저 의심하세요.

---

## 25.3 핸들러와 라우터 — `async fn`이 어떻게 핸들러가 되는가

핸들러는 그냥 `async fn`입니다. 인자가 없어도 되고, 익스트랙터를 여러 개 받아도 됩니다. 가장 단순한 핸들러부터 봅시다.

```python
# FastAPI
@app.get("/health")
async def health() -> str:
    return "ok"
```

```rust
// Axum
async fn health() -> &'static str {
    "ok"
}
```

반환한 `&'static str`이 그대로 `200 OK` 응답의 본문이 됩니다. 어떻게 문자열이 HTTP 응답이 될까요? `&str`이 `IntoResponse` 트레이트를 구현하기 때문입니다(25.5에서 깊이 다룹니다). Axum은 핸들러의 반환값에 `.into_response()`를 호출해 `Response`로 바꿉니다. 그래서 핸들러는 "응답으로 변환 가능한 무엇이든" 반환할 수 있습니다 — `String`, `Json<T>`, `(StatusCode, String)`, `Result<T, E>` 모두 가능합니다.

`route`의 두 번째 인자에 주목하세요. `get(health)`, `post(ingest)`처럼 HTTP 메서드 함수로 핸들러를 감쌉니다. FastAPI의 `@app.get`/`@app.post`와 일대일 대응이며, 같은 경로에 메서드별로 다른 핸들러를 붙일 수도 있습니다.

```rust
use axum::routing::{get, post};

Router::new()
    // 같은 경로 "/readings"에 GET과 POST를 각각
    .route("/readings", get(list_readings).post(ingest))
    // 경로 변수는 {} 안에 (axum 0.8). 핸들러의 Path<i64>로 추출됨
    .route("/aggregate/{id}", get(get_aggregate));
```

> **💡 Under the Hood — `Handler` 트레이트와 정적 디스패치**
>
> `get(health)`가 컴파일되려면 `health`가 `Handler` 트레이트를 구현해야 합니다. 그런데 우리는 `health`에 그 트레이트를 직접 구현한 적이 없습니다. 비결은 Axum이 **함수와 클로저에 대해 `Handler`를 블랭킷 구현(blanket impl)**해 두었다는 데 있습니다. 그것도 인자 개수별로(0개, 1개, 2개, …) 매크로로 생성된 여러 impl이 있습니다. 핵심은 그 impl의 트레이트 바운드입니다(개념적으로 단순화하면):
>
> ```rust
> // 개념적 형태: 마지막 인자가 FromRequest, 나머지가 FromRequestParts,
> // 반환 타입이 IntoResponse인 모든 async fn에 대해 Handler를 구현한다.
> impl<F, Fut, T1, T2, Res> Handler<(T1, T2), S> for F
> where
>     F: FnOnce(T1, T2) -> Fut,
>     Fut: Future<Output = Res>,
>     T1: FromRequestParts<S>,
>     T2: FromRequest<S>,
>     Res: IntoResponse,
> { /* ... */ }
> ```
>
> 그래서 핸들러를 작성할 때 우리가 만족시켜야 하는 계약은 셋입니다. **(1) 모든 인자가 익스트랙터 트레이트를 구현할 것, (2) 마지막 인자만 본문을 소비하는 `FromRequest`이고 나머지는 헤더/경로만 보는 `FromRequestParts`일 것, (3) 반환 타입이 `IntoResponse`일 것.** 셋 중 하나라도 어기면 컴파일이 거부됩니다. FastAPI라면 잘못된 시그니처가 서버 시작 시점(혹은 첫 요청 시점)에 런타임 에러로 드러나지만, Axum은 빌드 단계에서 막습니다. 그리고 이 모든 디스패치 — "이 인자를 어떻게 추출하고, 반환값을 어떻게 응답으로 바꿀지" — 는 단일형화(Monomorphization)되어 핸들러마다 전용 기계어로 박힙니다. 가상 테이블 조회도, 시그니처 재파싱도 런타임에 없습니다.

이 "마지막 인자만 `FromRequest`" 규칙에는 명확한 이유가 있습니다. 요청 본문(body)은 스트림이라 **한 번만 소비**할 수 있습니다. `Json<T>`, `Bytes`, `String` 같은 익스트랙터는 본문을 통째로 읽어 들이므로(`FromRequest`), 핸들러당 하나만 올 수 있고 맨 마지막에 와야 합니다. 반면 경로·쿼리·헤더·상태는 본문을 건드리지 않으므로(`FromRequestParts`), 몇 개든 앞쪽에 자유롭게 둘 수 있습니다. 이 구분이 다음 절의 주제입니다.

---

## 25.4 익스트랙터 — 타입 기반 의존성 주입

익스트랙터는 Axum의 심장이자, FastAPI의 `Depends`와 파라미터 선언을 한데 합친 개념입니다. "요청의 어떤 부분을, 어떤 타입으로 꺼낼 것인가"를 인자 타입으로 선언하면 끝입니다.

### 기본 익스트랙터

```rust
use axum::{
    Json,
    extract::{Path, Query, State},
};
use serde::Deserialize;

#[derive(Deserialize)]
struct Reading { sensor_id: i64, value: f64 }

#[derive(Deserialize)]
struct Filter { min: Option<f64> }

// 경로 변수 {id} → Path<i64>
async fn by_path(Path(id): Path<i64>) -> String { format!("id={id}") }

// 쿼리 스트링 ?min=1.0 → Query<Filter> (serde로 역직렬화)
async fn by_query(Query(f): Query<Filter>) -> String { format!("min={:?}", f.min) }

// JSON 본문 → Json<Reading> (serde Deserialize, FromRequest이므로 마지막 인자)
async fn by_body(Json(r): Json<Reading>) -> String { format!("{}", r.sensor_id) }
```

각 익스트랙터를 FastAPI와 짝지어 보면 직관적입니다. `Path<T>`는 경로 변수, `Query<T>`는 쿼리 파라미터(`serde`가 `?a=1&b=2`를 `T`로 역직렬화), `Json<T>`는 본문 JSON입니다. `Filter { min: Option<f64> }`에서 `Option`은 FastAPI의 기본값 있는 쿼리 파라미터(`min: float | None = None`)에 대응합니다. 검증 규칙을 타입으로 표현한다는 발상이 Chapter 7(serde)에서 본 그대로 웹 계층에 적용됩니다.

### 공유 상태: `State(Arc<...>)`

FastAPI에서 DB 풀이나 설정을 `Depends`로 주입하던 것을, Axum은 `State` 익스트랙터로 합니다. 상태는 보통 `Arc`로 감싸 모든 핸들러가 무비용으로 공유합니다(Chapter 17).

```rust
use std::sync::Arc;
use std::collections::HashMap;
use tokio::sync::RwLock;

// 공유 상태. 여러 스레드의 핸들러가 동시에 읽고 쓰므로 RwLock으로 보호.
#[derive(Default)]
struct AppState {
    totals: RwLock<HashMap<i64, (f64, u64)>>,
}
type SharedState = Arc<AppState>;

async fn use_state(State(state): State<SharedState>) -> String {
    let totals = state.totals.read().await; // 비동기 읽기 락 (Ch18)
    format!("{} sensors", totals.len())
}

// 라우터에 .with_state(Arc::new(AppState::default()))로 주입
```

여기서 Chapter 17과 Chapter 18의 지식이 합쳐집니다. `Arc<AppState>`는 스레드 간 공유 소유권이고(work-stealing 스케줄러가 핸들러를 어느 워커 스레드에서든 돌릴 수 있으므로 `Send + Sync`가 필요합니다), `RwLock`은 그 안의 가변 데이터를 보호합니다. 핸들러 안에서 `.await`을 넘어 락을 들고 있어야 하므로(읽고 나서 비동기 I/O를 더 한다면), 여기서는 `std::sync::RwLock`이 아니라 `tokio::sync::RwLock`을 씁니다 — Chapter 18에서 본 "락 가드를 `await` 경계 너머로 들고 갈 때"의 정확한 그 상황입니다. 단순히 읽고 즉시 끝난다면 `std` 락 + 짧은 스코프가 더 빠릅니다.

> **💡 Under the Hood — 익스트랙터는 `FromRequestParts` 트레이트다**
>
> `State`, `Path`, `Query`가 핸들러 인자로 동작하는 비결은 이들이 전부 `FromRequestParts` 트레이트를 구현한다는 데 있습니다. 트레이트의 형태는 이렇습니다(axum 0.8 기준, async fn in trait).
>
> ```rust
> pub trait FromRequestParts<S>: Sized {
>     type Rejection: IntoResponse;        // 추출 실패 시 응답이 될 타입
>     async fn from_request_parts(parts: &mut Parts, state: &S)
>         -> Result<Self, Self::Rejection>;
> }
> ```
>
> 핸들러가 `Path<i64>`를 인자로 받으면, 컴파일러는 `Path::<i64>::from_request_parts`를 호출하는 코드를 생성합니다. 이 호출은 **정적 디스패치**입니다 — 어떤 함수를 부를지 컴파일 타임에 확정되고, 인라이닝 대상이 됩니다. FastAPI가 요청마다 시그니처 분류 결과를 들고 동적으로 "이 인자는 path니까 경로에서 뽑고…" 하는 인터프리터 작업과 정반대입니다. 더 결정적인 것은 `Rejection` 타입입니다. 추출이 실패하면(경로 변수가 `i64`로 파싱 안 되면) `from_request_parts`가 `Err(rejection)`을 반환하고, Axum은 그 `rejection`을 `.into_response()`해 곧바로 적절한 4xx 응답을 만듭니다. "검증 실패 = 던져진 예외"가 아니라 "검증 실패 = 응답으로 변환 가능한 `Err` 값"이라는 것이 핵심입니다. 본문을 소비하는 `Json<T>`만 `FromRequest`(`parts`가 아니라 요청 전체 `Request`를 받음)이고, 나머지는 본문을 안 건드리는 `FromRequestParts`라서 25.3에서 본 "마지막 인자" 규칙이 타입 수준에서 강제됩니다.

### 커스텀 익스트랙터 — FastAPI `Depends`의 완전한 대응물

FastAPI에서 인증을 `api_key: str = Depends(get_api_key)`로 주입하듯, Axum에서는 `FromRequestParts`를 직접 구현해 나만의 익스트랙터를 만듭니다. 다음은 `Authorization` 헤더에서 API 키를 뽑아 검증하는 익스트랙터로, **컴파일·실행이 검증된 코드**입니다.

```rust
use axum::{
    extract::FromRequestParts,
    http::{header, request::Parts, StatusCode},
    response::{IntoResponse, Response},
    Json,
};

// 커스텀 익스트랙터: 요청 헤더에서 API 키를 뽑는다.
struct ApiKey(String);

impl<S> FromRequestParts<S> for ApiKey
where
    S: Send + Sync,
{
    type Rejection = Response; // 실패하면 이 응답을 그대로 반환

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        let key = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .map(|s| s.to_owned())
            .ok_or_else(|| {
                (StatusCode::UNAUTHORIZED,
                 Json(serde_json::json!({ "error": "API 키 없음" })))
                    .into_response()
            })?;
        Ok(ApiKey(key))
    }
}

// 핸들러는 그냥 ApiKey를 받는다. 인증은 익스트랙터가 끝냈다.
async fn protected(ApiKey(key): ApiKey) -> String {
    format!("hello, {key}")
}
```

이것이 FastAPI `Depends`와의 진짜 대응입니다. `protected` 핸들러는 인증 로직을 한 줄도 갖지 않습니다 — 단지 `ApiKey`를 인자로 선언했을 뿐인데, 키가 없으면 핸들러는 **호출조차 되지 않고** 401이 나갑니다. 의존성 주입의 본질("핸들러가 필요로 하는 것을 타입으로 선언하면 프레임워크가 채워준다")이 트레이트 한 개로 구현됩니다. FastAPI가 런타임 의존성 그래프를 순회하는 대신, Axum은 `ApiKey::from_request_parts`를 정적으로 호출합니다.

---

## 25.5 에러를 HTTP 응답으로 — `IntoResponse`

Chapter 13에서 우리는 에러 전파(`?`)와 커스텀 에러 설계(`thiserror`)를 배웠습니다. 그 에러가 웹 계층에서 HTTP 상태 코드로 바뀌는 다리가 바로 `IntoResponse` 트레이트입니다.

핸들러의 반환 타입은 `IntoResponse`를 구현해야 합니다. 표준 라이브러리와 Axum이 수많은 타입에 이미 구현해 두었습니다.

```rust
use axum::{Json, http::StatusCode};

// 모두 IntoResponse를 구현 → 핸들러 반환 타입으로 OK
async fn a() -> &'static str { "text" }                       // 200 + 텍스트
async fn b() -> String { "owned".into() }                     // 200 + 텍스트
async fn c() -> Json<Vec<i64>> { Json(vec![1, 2, 3]) }        // 200 + JSON
async fn d() -> StatusCode { StatusCode::NO_CONTENT }         // 204, 본문 없음
async fn e() -> (StatusCode, Json<&'static str>) {            // 상태 + 본문 튜플
    (StatusCode::CREATED, Json("done"))
}
```

`(StatusCode, T)` 튜플이 `IntoResponse`라는 점이 특히 유용합니다 — "상태 코드와 본문을 함께"라는 가장 흔한 패턴을 추가 타입 없이 표현합니다. FastAPI에서 `JSONResponse(status_code=201, content=...)`를 쓰던 것이 그냥 튜플 반환이 됩니다.

진짜 힘은 **커스텀 에러 타입에 `IntoResponse`를 구현**할 때 나옵니다. 이렇게 하면 핸들러가 `Result<T, MyError>`를 반환하고, `?`로 에러를 전파하면 그대로 적절한 HTTP 응답이 나갑니다.

```rust
use axum::{
    Json,
    http::StatusCode,
    response::{IntoResponse, Response},
};

// 도메인 에러. Ch13의 커스텀 에러 enum을 웹용으로.
#[derive(Debug)]
enum ApiError {
    NotFound(i64),
    Invalid(String),
}

// 에러 → HTTP 응답 변환. 여기서 상태 코드와 본문을 결정한다.
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            ApiError::NotFound(id) => (StatusCode::NOT_FOUND, format!("센서 {id} 없음")),
            ApiError::Invalid(m) => (StatusCode::BAD_REQUEST, m),
        };
        (status, Json(serde_json::json!({ "error": msg }))).into_response()
    }
}
```

`ApiError`가 `IntoResponse`이므로, 다음과 같은 핸들러가 곧바로 컴파일됩니다. `Result<Json<T>, ApiError>`도 `IntoResponse`라서(성공이면 `Ok`의 응답, 실패면 `Err`의 응답), 핸들러 반환 타입으로 직접 쓸 수 있습니다.

```rust
async fn get_aggregate(
    State(state): State<SharedState>,
    Path(id): Path<i64>,
) -> Result<Json<Aggregate>, ApiError> {
    let totals = state.totals.read().await;
    // ok_or로 Option을 Result로, ?로 전파 → ApiError::NotFound가 404 응답이 됨
    let &(total, count) = totals.get(&id).ok_or(ApiError::NotFound(id))?;
    Ok(Json(Aggregate { sensor_id: id, total, count }))
}
```

`raise HTTPException(status_code=404)` 대신 `Err(ApiError::NotFound(id))`를 `?`로 전파하면, 그 에러가 `IntoResponse`를 통해 404 JSON 응답으로 변환됩니다. Chapter 12~13에서 다진 `Option`→`Result`→`?` 사슬이 그대로 HTTP 계층까지 이어집니다.

> **💡 Under the Hood — `Result<T, E>: IntoResponse`의 정체**
>
> 어떻게 `Result<Json<Aggregate>, ApiError>`가 응답이 될까요? Axum의 블랭킷 구현 때문입니다.
>
> ```rust
> // 개념적 형태: T와 E가 둘 다 IntoResponse면 Result<T, E>도 IntoResponse
> impl<T, E> IntoResponse for Result<T, E>
> where
>     T: IntoResponse,
>     E: IntoResponse,
> {
>     fn into_response(self) -> Response {
>         match self {
>             Ok(value) => value.into_response(),
>             Err(err) => err.into_response(),
>         }
>     }
> }
> ```
>
> 그래서 `Json<Aggregate>`가 `IntoResponse`이고 `ApiError`가 `IntoResponse`이기만 하면, `Result<Json<Aggregate>, ApiError>`는 자동으로 `IntoResponse`가 됩니다 — 우리가 따로 구현할 필요가 없습니다. 이것이 무비용 추상화(Zero-cost Abstraction)의 한 모습입니다. 컴파일러는 이 `match`를 핸들러마다 단일형화해 인라이닝하므로, 런타임에 "이게 성공인가 에러인가"를 분기하는 비용은 평범한 `match` 한 번, 즉 분기 명령어 하나입니다. FastAPI가 예외를 던지고 미들웨어 스택을 거슬러 올라가며 핸들러를 찾는 것과 달리, 여기엔 예외 풀기(unwinding)도, 핸들러 탐색도 없습니다. 핸들러의 반환 타입 자체가 "이 함수는 이런 응답들을 낼 수 있다"는 명세이고, 컴파일러가 그것을 강제합니다.

---

## 25.6 미들웨어 — tower 레이어

FastAPI의 미들웨어(`@app.middleware("http")`)와 의존성 기반 횡단 관심사(CORS, 로깅, 인증)에 대응하는 것이 Axum의 **레이어(Layer)**입니다. Axum은 `tower`라는 범용 미들웨어 생태계 위에 서 있어서, `tower`/`tower-http`의 레이어를 그대로 꽂을 수 있습니다.

```rust
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/readings", post(ingest))
    .route("/aggregate/{id}", get(get_aggregate))
    .layer(TraceLayer::new_for_http())  // 모든 요청에 구조화된 로깅
    .with_state(state);
```

`.layer(...)`로 추가한 레이어는 모든 요청을 감쌉니다. `tower-http`는 실무에서 바로 쓰는 레이어들을 제공합니다: `TraceLayer`(요청/응답 로깅), `CorsLayer`(CORS), `CompressionLayer`(gzip/brotli 압축), `TimeoutLayer`(타임아웃), `ServeDir`(정적 파일). FastAPI에서 `CORSMiddleware`, `GZipMiddleware`를 등록하던 것과 일대일입니다.

> **💡 Under the Hood — 레이어는 서비스를 감싸는 함수다**
>
> `tower`의 핵심 추상은 `Service` 트레이트로, 개념적으로 "요청을 받아 응답 future를 돌려주는 비동기 함수"입니다(`async fn(Request) -> Response`). `Layer`는 그 `Service`를 받아 새 `Service`를 돌려주는 변환기입니다 — 데코레이터 패턴의 타입 버전입니다. `.layer(A).layer(B)`로 쌓으면 `A(B(핸들러))` 꼴로 중첩되어, 요청은 바깥 레이어부터 안쪽으로, 응답은 그 역순으로 흐릅니다. 결정적으로 이 모든 중첩이 **컴파일 타임에 타입으로 합성**됩니다. 미들웨어 체인이 런타임 리스트가 아니라 중첩된 제네릭 타입(`Trace<Cors<Router>>` 같은)이라, 디스패치가 단일형화되고 인라이닝됩니다. FastAPI/Starlette의 미들웨어가 런타임에 ASGI 콜러블을 양파처럼 감싸 매 요청마다 그 스택을 파이썬 호출로 거슬러 오르는 것과 달리, tower 레이어는 합성된 단일 기계어 경로에 가깝습니다. 같은 "양파 모델"이되, 하나는 런타임 객체 스택이고 하나는 컴파일 타임 타입 합성입니다.

레이어 적용 순서에는 함정이 하나 있습니다. `.layer()`는 **그 시점까지 정의된 라우트에만** 적용되며, 호출 순서가 곧 바깥/안쪽 순서를 결정합니다. 인증처럼 일부 라우트에만 적용해야 하는 미들웨어는 `Router`를 분리해 따로 `.layer()`한 뒤 `.merge()`하거나 `route_layer`를 씁니다. 전역 미들웨어(로깅, CORS)는 라우트를 다 정의한 뒤 맨 마지막에 `.layer()`하는 것이 보통입니다.

---

## 25.7 무공유 비동기 — 워커 프로세스 없이 멀티코어

25.1에서 본 FastAPI의 가장 아픈 한계 — "워커 프로세스 8개 = 메모리 8배, 상태 공유 불가" — 를 Axum은 어떻게 풀까요? 답은 Chapter 18의 `tokio` 멀티스레드 work-stealing 스케줄러입니다.

`#[tokio::main]`으로 띄운 Axum 서버는 **단일 프로세스**입니다. 그 안에서 `tokio` 런타임이 CPU 코어 수만큼 워커 스레드를 띄우고, 들어오는 각 요청을 하나의 비동기 태스크로 만들어 그 스레드들에 분산합니다. GIL이 없으므로 8개 워커 스레드가 진짜로 8개 코어에서 동시에 핸들러를 실행합니다. 그리고 이 모든 스레드가 **하나의 `Arc<AppState>`를 공유**합니다 — FastAPI 워커들이 각자 격리된 메모리에 갇혀 인메모리 상태를 못 나누던 것과 정반대입니다.

```rust
#[tokio::main]  // 멀티스레드 런타임을 띄우고 main을 그 위에서 실행
async fn main() {
    let state: SharedState = Arc::new(AppState::default());
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    // 단일 프로세스. 내부적으로 코어 수만큼 워커 스레드가 요청을 분산 처리.
    axum::serve(listener, app(state)).await.unwrap();
}
```

요청당 비용을 두 모델로 비교하면 격차가 분명합니다.

| 항목 | FastAPI (`gunicorn -w 8`) | Axum (단일 프로세스 `tokio`) |
|------|---------------------------|------------------------------|
| 멀티코어 활용 | 프로세스 8개로 우회 | 워커 스레드 8개, 단일 프로세스 |
| 인메모리 상태 공유 | 불가(프로세스 격리) → 외부 저장소 필요 | `Arc<State>`로 직접 공유 |
| 기본 메모리(8코어) | 인터프리터·모델 8배 복제 | 1배(상태 1벌) |
| 요청당 태스크 메모리 | 코루틴 프레임(힙, 수 KB급) | 상태 기계(수십~수백 바이트급) |
| 동시 커넥션 한계 | 워커별 단일 루프 합 | work-stealing으로 자동 평준화 |

> **💡 Under the Hood — 요청 하나의 메모리 발자국**
>
> Axum에서 요청 하나가 만드는 것을 메모리로 따라가 봅시다. 도메인 모델 `Reading { sensor_id: i64, value: f64 }`은 `std::mem::size_of`로 측정하면 정확히 **16바이트**입니다(`i64` 8 + `f64` 8, 헤더 0). 집계 슬롯 `(f64, u64)`도 **16바이트**입니다. 비교하면 Python에서 같은 데이터는 `int` 객체 28바이트 + `float` 객체 24바이트 + 이들을 담는 `dict`(빈 dict만 64바이트)로, 헤더와 포인터가 실제 데이터를 압도합니다(Chapter 1의 그 해부입니다). 요청을 처리하는 핸들러 future 자체도 Chapter 18에서 본 스택리스 상태 기계라, 힙 할당 없이 태스크 구조체에 인라인됩니다. 공유 상태로 가는 `Arc<AppState>`는 **8바이트 포인터**일 뿐이고(`size_of::<Arc<_>>()` = 8), `.clone()`은 원자적 카운터 증가 하나입니다 — 상태 데이터를 복사하지 않습니다. 그래서 동시 커넥션이 수만으로 올라가도 태스크당 발자국이 작고, 무엇보다 그 모든 태스크가 코어들에 흩어져 진짜로 병렬 실행됩니다. 이것이 "단일 프로세스로 멀티코어 + 공유 상태"라는, FastAPI가 구조적으로 못 가졌던 조합의 실체입니다.

물론 공짜는 없습니다. 25.10에서 보겠지만, 핸들러 안에서 무거운 CPU 작업이나 블로킹 호출을 하면 그 워커 스레드가 막혀 다른 요청을 못 받습니다. FastAPI의 이벤트 루프 정지와 같은 함정이 Rust에도 있고, 처방도 같습니다 — `spawn_blocking`(Chapter 18).

---

## 25.8 실전 Hands-on — 집계 서비스를 REST API로

이제 모든 조각을 묶어, Chapter 22의 센서 집계 서비스를 완전한 REST API로 만듭니다. `POST /readings`로 측정값을 받아 검증·누적하고, `GET /aggregate/{id}`로 집계를 조회하며, `reqwest`로 통합 테스트까지 작성합니다. 아래는 `cargo test`로 검증을 통과한 **완전한 코드**입니다.

먼저 의존성입니다.

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }  # 2026-10-22에 1.54.0으로 풀림
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.6", features = ["trace"] }

[dev-dependencies]
reqwest = { version = "0.12", features = ["json"] }
```

본체입니다. Chapter 7의 serde 모델, Chapter 13의 `IntoResponse` 에러, Chapter 17의 `Arc` 공유 상태, Chapter 18의 비동기 락이 한 파일에 모입니다.

```rust
use std::collections::HashMap;
use std::sync::Arc;

use axum::{
    Json, Router,
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::{get, post},
};
use serde::{Deserialize, Serialize};
use tokio::sync::RwLock;

// --- 도메인 모델 (Ch7 serde 모델 재사용) ---
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Reading {
    sensor_id: i64,
    value: f64,
}

#[derive(Debug, Serialize, Deserialize)]
struct Aggregate {
    sensor_id: i64,
    total: f64,
    count: u64,
}

// --- 공유 상태 (Ch17 Arc + Ch18 비동기 락) ---
#[derive(Default)]
struct AppState {
    totals: RwLock<HashMap<i64, (f64, u64)>>, // id -> (합계, 개수)
}
type SharedState = Arc<AppState>;

// --- 에러 → 응답 (Ch13 IntoResponse) ---
#[derive(Debug)]
enum ApiError {
    NotFound(i64),
    Invalid(String),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            ApiError::NotFound(id) => (StatusCode::NOT_FOUND, format!("센서 {id} 없음")),
            ApiError::Invalid(m) => (StatusCode::BAD_REQUEST, m),
        };
        (status, Json(serde_json::json!({ "error": msg }))).into_response()
    }
}

// --- 핸들러 = async fn ---
async fn health() -> &'static str {
    "ok"
}

// POST /readings : JSON 본문 검증 후 집계에 누적
async fn ingest(
    State(state): State<SharedState>,
    Json(reading): Json<Reading>,
) -> Result<Json<Aggregate>, ApiError> {
    if !reading.value.is_finite() {
        return Err(ApiError::Invalid("value는 유한수여야 합니다".into()));
    }
    let mut totals = state.totals.write().await;
    let entry = totals.entry(reading.sensor_id).or_insert((0.0, 0));
    entry.0 += reading.value;
    entry.1 += 1;
    Ok(Json(Aggregate {
        sensor_id: reading.sensor_id,
        total: entry.0,
        count: entry.1,
    }))
}

// GET /aggregate/{id} : 집계 조회
async fn get_aggregate(
    State(state): State<SharedState>,
    Path(id): Path<i64>,
) -> Result<Json<Aggregate>, ApiError> {
    let totals = state.totals.read().await;
    let &(total, count) = totals.get(&id).ok_or(ApiError::NotFound(id))?;
    Ok(Json(Aggregate { sensor_id: id, total, count }))
}

fn app(state: SharedState) -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/readings", post(ingest))
        .route("/aggregate/{id}", get(get_aggregate))
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let state: SharedState = Arc::new(AppState::default());
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app(state)).await.unwrap();
}
```

핸들러를 다시 보면 FastAPI 버전(25.1)과 줄 수가 거의 같습니다. `Json(reading): Json<Reading>`은 `reading: Reading`에, `Path(id): Path<i64>`는 `sensor_id: int`에, `Err(ApiError::NotFound(id))?`는 `raise HTTPException(404)`에 대응합니다. 다른 점은 이 모든 대응이 컴파일러에 의해 타입으로 검증되고, 단일 프로세스에서 `Arc<AppState>`를 진짜로 공유하며 멀티코어로 돈다는 것입니다.

### 통합 테스트 — `reqwest`로

핵심 가치는 테스트가 실제 HTTP를 탄다는 것입니다. 임의 포트(`127.0.0.1:0`)로 서버를 spawn하고, `reqwest` 클라이언트로 진짜 요청을 보냅니다. FastAPI의 `TestClient`에 대응하지만, 실제 TCP 위에서 검증합니다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn ingest_then_query() {
        // 서버를 임의 포트로 spawn (Ch18의 tokio::spawn)
        let state: SharedState = Arc::new(AppState::default());
        let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(async move {
            axum::serve(listener, app(state)).await.unwrap();
        });

        let client = reqwest::Client::new();
        let base = format!("http://{addr}");

        // POST 두 번 누적 (1.5 + 2.5 = 4.0)
        for v in [1.5, 2.5] {
            let resp = client
                .post(format!("{base}/readings"))
                .json(&serde_json::json!({ "sensor_id": 7, "value": v }))
                .send()
                .await
                .unwrap();
            assert_eq!(resp.status(), 200);
        }

        // GET 조회 → 합계 4.0, 개수 2
        let agg: Aggregate = client
            .get(format!("{base}/aggregate/7"))
            .send()
            .await
            .unwrap()
            .json()
            .await
            .unwrap();
        assert_eq!(agg.count, 2);
        assert_eq!(agg.total, 4.0);

        // 없는 센서 → 404 (ApiError::NotFound가 IntoResponse로 변환됨)
        let resp = client
            .get(format!("{base}/aggregate/999"))
            .send()
            .await
            .unwrap();
        assert_eq!(resp.status(), 404);
    }
}
```

이 테스트를 `cargo test`로 돌리면 통과합니다. 2026-10-22에 rustc 1.99.1, axum 0.8.9, tokio 1.54.0에서 다시 통과했습니다. 처음 검증은 rustc 1.90.0이었습니다. `Aggregate`에 `Deserialize`를 추가한 이유가 여기 있습니다 — 클라이언트가 응답 JSON을 다시 `Aggregate`로 역직렬화하기 때문입니다. 같은 serde 모델이 서버에서는 직렬화에, 테스트 클라이언트에서는 역직렬화에 쓰이는, Chapter 7 serde 설계의 양방향성이 그대로 살아납니다.

> **🚨 No Free Lunch — 인메모리 상태의 수명**
>
> 이 예제의 `Arc<AppState>`는 **프로세스 메모리**에 삽니다. 단일 프로세스 멀티스레드라 FastAPI와 달리 워커 간 공유 문제는 없지만, 프로세스가 죽으면 집계도 사라지고, 수평 확장(인스턴스 여러 대)을 하면 인스턴스마다 상태가 갈라집니다. 즉 "단일 프로세스 멀티코어 공유 상태"는 한 노드 안에서의 이야기입니다. 여러 노드로 확장하거나 영속성이 필요하면, 이 자리에 `sqlx`(비동기 SQL), `redis` 같은 외부 저장소가 들어옵니다 — 그때도 풀(pool)을 `Arc`로 감싸 `State`로 주입하는 구조는 똑같습니다. Rust가 없애주는 것은 "멀티코어를 위한 프로세스 복제"이지, "분산 시스템의 상태 공유" 자체가 아닙니다.

---

## 25.9 Best Practices & 안티패턴

실무 체크리스트입니다.

**이렇게 하라**

- **공유 상태는 `Arc<AppState>` 하나로 묶어 `State`로 주입.** 풀·설정·캐시를 각각 익스트랙터로 흩뿌리지 말고, `AppState` 구조체에 모은 뒤 부분만 필요하면 `#[derive(FromRef)]`로 서브스테이트를 뽑으세요.
- **커넥션 풀·HTTP 클라이언트는 시작 시 한 번 만들어 공유.** `reqwest::Client`, `sqlx::Pool`은 내부가 `Arc`라 `clone`이 싸고 커넥션 풀을 재사용합니다. 요청마다 새로 만들면 TLS 핸드셰이크가 낭비됩니다(Chapter 18의 그 원칙).
- **에러는 도메인 enum + `IntoResponse`로.** `thiserror`(Chapter 13)로 에러 enum을 정의하고 `IntoResponse`를 한 곳에 구현하면, 모든 핸들러가 `?`로 깔끔하게 에러를 전파합니다. 상태 코드 매핑이 한 군데 모여 일관성이 유지됩니다.
- **검증은 타입으로.** `Json<Reading>`의 `Reading`을 뉴타입(Chapter 8)·serde 어트리뷰트로 조이면, 잘못된 입력이 핸들러에 도달하기 전에 거부됩니다. 추가 검증이 필요하면 `serde` 역직렬화 후 핸들러 첫 줄에서 명시적으로(예제의 `is_finite()`처럼).
- **블로킹/CPU는 `spawn_blocking`으로 격리.** 동기 DB 드라이버, 이미지 처리, 무거운 계산은 비동기 워커에서 직접 돌리지 마세요(25.10, Chapter 18).
- **타임아웃·트레이싱 레이어를 깔아라.** `tower-http`의 `TimeoutLayer`, `TraceLayer`를 기본으로 두면 무한 대기와 관측 불가 문제를 예방합니다.
- **`#[tokio::main]`은 진입점에 한 번만.** 라이브러리 코드는 런타임에 의존하지 않게 `async fn`만 노출하세요.

**이건 피하라(안티패턴)**

- 핸들러 안에서 `std::thread::sleep`·동기 I/O·무거운 CPU 루프 직접 호출 — 워커 스레드를 얼립니다. FastAPI의 이벤트 루프 정지와 같은 사고입니다.
- 모든 것을 `tokio::spawn`으로 흩뿌리기 — 핸들러는 이미 태스크입니다. 핸들러 안에서 또 spawn할 필요는 드뭅니다(백그라운드 작업이 아니라면).
- `State`에 락을 너무 크게 잡고 `.await` 너머로 들고 가기 — 직렬화 병목이 됩니다. 락 스코프는 좁게, 가능하면 `await` 이전에 닫으세요(Chapter 18).
- 에러마다 즉석에서 `(StatusCode, String)`을 반환 — 상태 코드 매핑이 핸들러마다 흩어져 일관성이 깨집니다. 도메인 에러 enum으로 모으세요.
- 거대한 에러 enum을 그대로 반환 — `clippy::result_large_err`가 경고합니다(Chapter 13). 큰 variant는 `Box`로 빼세요.

---

## 25.10 흔한 함정 — Python 습관이 부르는 컴파일 에러와 사고

FastAPI 습관이 Axum에서 어떻게 어긋나는지, 그리고 컴파일러가 어떻게 잡아주는지 정리합니다.

**함정 1 — 핸들러 반환 타입이 `IntoResponse`가 아니다.**

FastAPI는 핸들러가 무엇을 반환하든 런타임에 JSON으로 직렬화하려 시도합니다(실패하면 500). Axum은 반환 타입이 `IntoResponse`가 아니면 **빌드를 거부**합니다. `IntoResponse`를 구현하지 않은 타입을 반환하면 이런 에러가 납니다.

```rust
struct Raw { _x: i32 } // IntoResponse 미구현

// 🚨 컴파일 에러를 유발하는 코드
async fn handler() -> Raw {
    Raw { _x: 1 }
}
// Router::new().route("/", get(handler))
```

`rustc`의 실제 출력입니다.

```text
error[E0277]: the trait bound `fn() -> impl Future<Output = Raw> {handler}: Handler<_, _>` is not satisfied
  --> src/bin/err_response.rs:12:52
   |
12 |     let app: Router = Router::new().route("/", get(handler));
   |                                                --- ^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item `fn() -> impl Future<Output = Raw> {handler}`
   |                                                |
   |                                                required by a bound introduced by this call
   |
   = note: Consider using `#[axum::debug_handler]` to improve the error message
```

에러가 다소 난해해 보이는 것은 25.3에서 본 `Handler` 블랭킷 구현의 바운드(`Res: IntoResponse`)가 안 맞아서입니다. 컴파일러의 `note`가 친절하게 길을 알려줍니다 — `#[axum::debug_handler]`를 핸들러에 붙이면 "반환 타입 `Raw`가 `IntoResponse`를 구현하지 않는다"고 정확히 짚어줍니다. 해결은 `Raw`에 `IntoResponse`를 구현하거나, `Json(Raw)`처럼 이미 `IntoResponse`인 래퍼로 감싸는 것입니다.

**함정 2 — `State` 타입 불일치.**

핸들러가 기대하는 `State<T>`의 `T`와 라우터에 `with_state`로 넣은 상태 타입이 다르면 컴파일 에러입니다. FastAPI라면 `Depends`가 엉뚱한 값을 주입해도 런타임까지 모를 일을, Axum은 빌드에서 막습니다.

```rust
struct AppState { name: String }

// 핸들러는 State<String>을 기대
async fn handler(State(name): State<String>) -> String { name }

// 하지만 라우터에는 Arc<AppState>를 주입 → 불일치
// Router::new().route("/", get(handler)).with_state(Arc::new(AppState{..}))
```

```text
error[E0308]: mismatched types
  --> src/bin/err_state.rs:16:21
   |
16 |         .with_state(state);
   |          ---------- ^^^^^ expected `String`, found `Arc<AppState>`
   |          |
   |          arguments to this method are incorrect
   |
   = note: expected struct `String`
              found struct `Arc<AppState>`
```

`E0308`(타입 불일치)이 정확히 "`String`을 기대했는데 `Arc<AppState>`가 왔다"고 알려줍니다. 핸들러의 `State<T>`와 라우터 상태 타입을 일치시키거나, 큰 상태에서 부분만 쓰려면 `#[derive(FromRef)]`로 서브스테이트를 추출하세요(`FromRef<AppState> for Config`를 구현하면 `State<Config>` 핸들러가 `AppState`에서 `Config`만 뽑아 받습니다).

**함정 3 — 블로킹 작업을 async 핸들러에서 직접 호출.**

Chapter 18에서 본, 그리고 25.7에서 예고한 그 함정입니다. 핸들러 안에서 동기 DB 드라이버, `std::thread::sleep`, 무거운 CPU 루프를 직접 호출하면 워커 스레드가 막혀 다른 요청을 못 받습니다. **이건 타입으로 표현되지 않으므로 컴파일러가 못 잡습니다** — FastAPI와 똑같이 위험한, Axum에서도 유일하게 컴파일러가 봐주지 않는 함정입니다. 처방도 같습니다.

```rust
// 🚨 안티패턴: CPU 작업을 핸들러에서 직접
async fn bad_heavy() -> String {
    let sum: u64 = (0..1_000_000_000).sum(); // 이 워커 스레드가 막힌다
    format!("{sum}")
}

// 해결: spawn_blocking으로 블로킹 전용 스레드풀에 격리 (Ch18)
async fn good_heavy() -> String {
    let sum = tokio::task::spawn_blocking(|| {
        (0..1_000_000_000u64).sum::<u64>()
    })
    .await
    .unwrap();
    format!("{sum}")
}
```

진짜 데이터 병렬 계산이라면 `spawn_blocking`보다 `rayon`(Chapter 16)이 더 나을 수 있습니다 — 비동기 워커는 I/O 동시성을 위한 것이지 CPU 병렬 계산을 위한 게 아니기 때문입니다. 감시 도구로는 `tokio-console`과 런타임 메트릭이 "워커가 너무 오래 막혔다"를 잡아냅니다.

**함정 4 — 본문 익스트랙터를 마지막이 아닌 곳에 둔다.**

`Json<T>`, `String`, `Bytes` 같은 본문 소비 익스트랙터(`FromRequest`)를 핸들러 인자 중간에 두면 컴파일 에러입니다. 25.3에서 본 "본문은 한 번만 소비, 그래서 마지막 인자" 규칙이 타입으로 강제됩니다. 해결은 `Json<T>`를 항상 마지막 인자로 옮기는 것입니다.

**함정 5 — 동기 컨텍스트에서 핸들러를 직접 부르려 한다.**

FastAPI 핸들러는 그냥 `async def` 함수라 테스트에서 직접 `await`하기 쉽습니다. Axum 핸들러도 `async fn`이라 직접 호출할 수 있지만, `State`나 `Json` 같은 익스트랙터를 인자로 받으므로 실제 요청 없이 호출하려면 익스트랙터 값을 손으로 구성해야 합니다(`Json(Reading{..})`, `State(arc.clone())`). 단위 테스트는 이렇게 익스트랙터를 직접 만들어 핸들러 함수를 호출하고, 통합 테스트는 25.8처럼 실제 HTTP를 태우는 두 층위로 나누는 것이 정석입니다.

이 함정들을 관통하는 교훈은 Chapter 18과 같습니다. FastAPI에서 웹 계층은 "런타임에 시그니처를 보고 동적으로 엮이는" 유연한 세계였지만, Axum에서 웹 계층은 **타입과 트레이트가 지키는 정적인 계약**입니다. 익스트랙터·핸들러·응답이 전부 트레이트 구현이라, 잘못된 조합은 빌드에서 막힙니다. 처음에는 `Handler` 바운드 에러나 `State` 불일치가 성가시게 느껴지지만, 그 하나하나가 FastAPI라면 첫 요청 때(혹은 프로덕션 새벽에) 터졌을 사고를 빌드 단계로 당겨온 것입니다.

---

지금까지 FastAPI의 웹 계층을 Axum으로 옮기며, 핸들러(`async fn`)·라우터(`Router`)·익스트랙터(`FromRequest`/`FromRequestParts`)·에러 응답(`IntoResponse`)·미들웨어(tower 레이어)가 어떻게 컴파일 타임 트레이트로 구현되는지 내부까지 파고들었습니다. Chapter 7의 serde 모델, Chapter 13의 에러 설계, Chapter 17의 `Arc` 공유 상태, Chapter 18의 `tokio`와 비동기 락, Chapter 22의 집계 서비스가 하나의 REST API로 합류했습니다. 핵심은 "더 빠르다"가 아니라, **워커 프로세스 복제 없이 단일 프로세스로 멀티코어와 인메모리 공유 상태를 동시에 누리면서, 웹 계층의 계약 위반을 컴파일러가 막아준다**는 것입니다.

웹 서비스가 갖춰졌으니, 다음은 그 서비스가 시간을 어디서 쓰는지 재는 일입니다. 다음 [Chapter 26](./chapter_26.md)에서는 프로파일링, `criterion`, 그리고 SIMD까지, 빨라졌다는 주장을 측정으로 확인하는 방법을 다룹니다.

---

## 연습문제

1. **`PUT /aggregate/{id}/reset` 추가.** 특정 센서의 집계를 0으로 초기화하는 엔드포인트를 추가하세요. 없는 센서를 리셋하려 하면 `ApiError::NotFound`로 404를 내야 합니다. 어떤 HTTP 메서드 함수와 익스트랙터 조합이 필요할까요?

2. **API 키 인증 적용.** 25.4의 `ApiKey` 커스텀 익스트랙터를 `POST /readings` 핸들러에 추가하세요. 인자 순서를 어떻게 두어야 컴파일될까요(힌트: `FromRequestParts` vs `FromRequest`)? 그리고 키가 없을 때 응답 상태 코드는?

3. **서브스테이트 분리.** `AppState`에 `totals`와 별개로 `config: AppConfig { max_value: f64 }`를 추가하고, `ingest` 핸들러가 `config.max_value`를 초과하는 값을 거부하도록 하세요. 일부 핸들러는 `config`만 필요한데, 전체 `AppState`를 받지 않고 `config`만 받으려면 어떤 트레이트를 구현해야 할까요?

4. **블로킹 함정 재현과 측정.** 핸들러 안에서 `std::thread::sleep(Duration::from_secs(2))`를 직접 호출하는 버전과 `spawn_blocking`으로 감싼 버전을 만들고, 동시에 여러 요청을 보내 처리량 차이를 관찰하세요. 워커 스레드 수를 1개로 제한하면(`new_current_thread` 또는 `worker_threads(1)`) 차이가 어떻게 극단화되나요?

5. **에러 enum 통합.** `thiserror`(Chapter 13)로 `ApiError`를 `#[derive(Error)]` 버전으로 다시 쓰고, `serde_json::Error`로부터 `From`을 자동 생성해 `?`로 전파되게 하세요. `IntoResponse` 구현에서 각 variant를 어떤 상태 코드로 매핑할지 정하세요.

### 정답 스케치

1. `.route("/aggregate/{id}/reset", put(reset))`로 `put` 핸들러를 추가합니다. 핸들러 시그니처는 `async fn reset(State(s): State<SharedState>, Path(id): Path<i64>) -> Result<StatusCode, ApiError>`. 본문에서 `let mut totals = s.totals.write().await;` 후 `totals.get_mut(&id).ok_or(ApiError::NotFound(id))?`로 존재를 확인하고 슬롯을 `(0.0, 0)`으로 덮은 뒤 `Ok(StatusCode::NO_CONTENT)`를 반환합니다. 본문이 없으므로 `Json` 익스트랙터는 불필요하고, `Path`만으로 충분합니다.

2. `async fn ingest(api_key: ApiKey, State(state): State<SharedState>, Json(reading): Json<Reading>)` 순서입니다. `ApiKey`와 `State`는 `FromRequestParts`(본문 안 건드림)라 앞쪽에 올 수 있고, 본문을 소비하는 `Json`은 `FromRequest`라 **반드시 마지막**이어야 합니다(25.3 규칙). 키가 없으면 익스트랙터의 `Rejection`이 `401 Unauthorized`를 반환하고, 핸들러 본문은 실행조차 되지 않습니다.

3. `FromRef<AppState> for AppConfig`를 구현합니다. `fn from_ref(s: &AppState) -> AppConfig { s.config.clone() }`. 그러면 `config`만 필요한 핸들러는 `State(cfg): State<AppConfig>`로 받을 수 있고, Axum이 `AppState`에서 `AppConfig`만 뽑아 줍니다. `AppState`에 `#[derive(Clone)]`이 있고 각 서브스테이트도 `Clone`이어야 합니다. `ingest`에서는 `if reading.value > cfg.max_value { return Err(ApiError::Invalid(..)); }`로 검증합니다.

4. 직접 호출 버전은 `sleep`이 워커 스레드를 통째로 막아, 워커 수가 N개면 동시에 N개 요청만 2초 안에 처리되고 나머지는 줄을 섭니다. `spawn_blocking` 버전은 블로킹 전용 풀(기본 최대 512스레드)로 격리되어 비동기 워커는 계속 다른 요청을 받습니다. `worker_threads(1)`로 제한하면 직접 호출 버전은 한 번에 요청 하나밖에 처리 못 해 완전히 직렬화되는 반면, `spawn_blocking` 버전은 단일 워커가 막히지 않아 동시성을 유지합니다 — 분리의 효과가 극단적으로 드러납니다.

5. ```rust
   use thiserror::Error;
   #[derive(Debug, Error)]
   enum ApiError {
       #[error("센서 {0} 없음")]
       NotFound(i64),
       #[error("잘못된 입력: {0}")]
       Invalid(String),
       #[error("JSON 오류")]
       Json(#[from] serde_json::Error), // ?로 자동 전파
   }
   ```
   `IntoResponse`에서 `NotFound`→404, `Invalid`→400, `Json`→400(또는 422)으로 매핑합니다. `#[from]`이 `From<serde_json::Error>`를 자동 생성하므로 `serde_json` 작업에 `?`를 바로 쓸 수 있습니다(Chapter 13).

---

## References

- https://docs.rs/axum/latest/axum/
- https://docs.rs/axum/latest/axum/extract/index.html
- https://docs.rs/axum/latest/axum/response/trait.IntoResponse.html
- https://github.com/tokio-rs/axum/tree/main/examples
- https://docs.rs/tower-http/latest/tower_http/
- https://docs.rs/tower/latest/tower/trait.Service.html
- https://tokio.rs/tokio/tutorial
- https://docs.rs/reqwest/latest/reqwest/
- https://serde.rs/
