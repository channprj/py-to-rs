# Chapter 18. 비동기 프로그래밍 (Async/Await)

FastAPI의 대성공 이후, 파이썬 백엔드 생태계는 빠르게 `asyncio` 기반으로 재편되었습니다. I/O 대기 시간(DB 쿼리, 외부 API 호출, 파일 읽기 등) 동안 스레드를 놀리지 않고 다른 요청을 처리하는 비동기(Asynchronous) 프로그래밍은 이제 현대 웹 서버의 선택이 아니라 기본값입니다. 수천 개의 동시 커넥션을 단일 프로세스로 감당하려면 OS 스레드를 커넥션마다 하나씩 띄우는 방식(스레드당 8MB 스택)은 메모리만으로도 파산하니까요.

하지만 10년 차 파이썬 엔지니어라면 `asyncio`에 데어본 경험이 한두 번은 있을 것입니다. 가장 흔한 사고는 이렇습니다. FastAPI 핸들러 안에서 무심코 동기 라이브러리(예: 옛날 `requests`, `psycopg2`, 혹은 `time.sleep`)를 호출했더니, 그 한 줄이 전체 이벤트 루프(Event Loop)를 통째로 얼려버립니다. 한 요청의 블로킹이 무관한 다른 수백 개의 요청을 동시에 멈춰 세우는 것입니다. p99 레이턴시가 갑자기 치솟고, 헬스체크가 타임아웃되고, 오토스케일러가 미쳐 날뜁니다. 범인을 찾는 데 며칠이 걸리기도 합니다.

이 사고의 근본 원인은 두 가지입니다. 첫째, **`asyncio`의 이벤트 루프는 단일 스레드 위에서만 돈다**는 구조적 한계(GIL의 또 다른 얼굴)입니다. 둘째, **파이썬은 어떤 호출이 블로킹인지 컴파일 타임에 알려주지 않는다**는 점입니다. `await`을 빠뜨려도, 블로킹 함수를 섞어 써도, 런타임에 실제로 터지기 전까지는 아무도 모릅니다. `RuntimeWarning: coroutine '...' was never awaited`라는 경고를 프로덕션 로그에서 처음 발견하는 순간의 그 서늘함을 기억하실 겁니다.

Rust의 `async/await`은 파이썬과 키워드까지 똑같이 생겼습니다. 그래서 더 위험합니다. 겉모습이 같으니 똑같이 동작하리라 가정하기 쉽지만, 내부 구현과 생태계 구조는 근본부터 다릅니다. Rust의 `async`는 **컴파일 타임에 상태 기계(State Machine)로 변환되는 무비용 추상화(Zero-cost Abstraction)**이고, 런타임은 언어가 아니라 라이브러리(`tokio`)가 제공하며, 그 런타임은 기본적으로 **멀티스레드 work-stealing 스케줄러**입니다. 그리고 가장 중요하게도, "락을 든 채 `await` 하지 마라" 같은 규칙을 Rust는 **컴파일러가 강제**합니다.

이번 챕터에서는 Chapter 16~17에서 다진 멀티스레딩·`Arc`/`Mutex`·`Send`/`Sync` 지식 위에, 비동기라는 마지막 동시성 도구를 얹습니다. 단순한 문법 소개를 넘어 "`async fn`이 컴파일되면 메모리에 무엇이 만들어지는가", "왜 Rust는 런타임을 표준에 넣지 않았는가", "왜 컴파일러가 `future is not Send` 에러를 내뱉는가"까지 내부로 파고듭니다.

미리 한 가지 마음의 준비를 해두면 좋겠습니다. 파이썬에서 비동기는 "성능 최적화 기법"의 색채가 강했습니다. 동기 코드로도 일단 돌아가긴 하니까요. 반면 Rust에서 비동기는 **별도의 실행 모델이자 타입 세계**입니다. `async fn`이 반환하는 값은 일반 함수의 반환값과 타입이 다르고(`impl Future<Output = T>`), 그 세계 안에서만 통하는 규칙(`Send` 경계, `Pin`, 게으름)이 적용됩니다. 동기 함수에서 `async fn`을 그냥 호출하면 아무 일도 일어나지 않고, 비동기 함수 안에서 블로킹 동기 함수를 호출하면 런타임이 마비됩니다. 두 세계의 경계를 의식적으로 다루는 것이 이 챕터의 또 다른 학습 목표입니다.

이 책의 다른 챕터와 마찬가지로, 우리는 "어떻게 쓰는가"보다 "왜 그렇게 동작하는가"에 무게를 둡니다. `tokio::spawn`을 호출하는 법은 다섯 줄이면 배우지만, 그 future가 왜 `Send`여야 하는지, 왜 `MutexGuard`를 들고 `await`하면 컴파일이 안 되는지를 이해하려면 상태 기계의 메모리 레이아웃까지 내려가야 합니다. 일단 그 바닥을 보고 나면, 컴파일러가 내뱉는 난해해 보이는 에러들이 전부 "당신의 동시성 버그를 미리 잡아준 것"임을 알게 됩니다.

---

## 18.1 The Python Way — `asyncio` 이벤트 루프와 그 약점

먼저 우리가 익숙한 방식을 정확히 짚고 넘어갑시다. 다음은 전형적인 `asyncio` 동시 처리 코드입니다.

```python
import asyncio

async def fetch_data(id: int) -> str:
    print(f"Fetching {id}...")
    await asyncio.sleep(1)  # I/O 대기를 시뮬레이션
    return f"Data {id}"

async def main() -> None:
    # 두 작업을 동시에 실행하고 모두 끝나길 기다림
    results = await asyncio.gather(fetch_data(1), fetch_data(2))
    print(results)

asyncio.run(main())
```

`fetch_data(1)`을 호출하면 즉시 **코루틴 객체(coroutine object)**가 만들어집니다. 이 코루틴은 `await` 되거나 `gather`/`create_task`에 넘겨지기 전까지는 실행되지 않습니다. 이 "게으름(Laziness)"은 사실 Rust와 공유하는 특성입니다. 그런데 파이썬은 코루틴을 만들어 놓고 `await`을 빠뜨리면 어떻게 될까요?

```python
async def main() -> None:
    fetch_data(1)  # await 누락! 코루틴은 만들어졌지만 절대 실행되지 않음
    await fetch_data(2)

# 런타임 경고:
# RuntimeWarning: coroutine 'fetch_data' was never awaited
```

이것이 첫 번째 약점입니다. **`await` 누락이 런타임 경고로만 잡힙니다.** 컴파일러가 없으니 당연합니다. 운이 좋으면 경고를 보고, 운이 나쁘면 조용히 사라진 작업 때문에 데이터 정합성이 깨집니다.

두 번째 약점이 더 치명적입니다. 이벤트 루프가 단일 스레드라는 점입니다.

```python
import asyncio, time

async def good_io() -> None:
    await asyncio.sleep(1)        # 협력적 양보: 루프가 다른 일을 함

async def bad_blocking() -> None:
    time.sleep(1)                 # 🚨 동기 블로킹! 루프 전체가 1초간 정지
    # 혹은 무거운 CPU 연산:
    # sum(i * i for i in range(10**8))

async def main() -> None:
    # bad_blocking이 도는 1초 동안 다른 모든 태스크가 굶는다
    await asyncio.gather(bad_blocking(), good_io(), good_io())
```

`asyncio`의 이벤트 루프는 협력적 멀티태스킹(Cooperative Multitasking)입니다. 각 코루틴이 `await` 지점에서 **자발적으로** 제어권을 루프에 반납해야만 다른 코루틴이 실행됩니다. `time.sleep(1)`이나 무거운 `for` 루프는 `await`이 없으므로 절대 양보하지 않습니다. 그 1초 동안 단일 스레드는 그 함수에 인질로 잡히고, 같은 루프의 수백 개 태스크가 전부 굶습니다.

> 💡 파이썬의 처방은 `loop.run_in_executor()`로 블로킹 작업을 별도 스레드풀(`ThreadPoolExecutor`)이나 프로세스풀로 빼는 것입니다. 하지만 이것은 개발자가 "이 호출이 블로킹임"을 스스로 인지해야만 적용할 수 있는 수동 방어책입니다. 어떤 라이브러리가 내부에서 동기 I/O를 하는지 일일이 알 수 없다는 게 진짜 문제입니다.

세 번째 약점은 더 미묘하고, 시니어 엔지니어조차 자주 헷갈립니다. **`asyncio`는 CPU 코어를 하나밖에 못 씁니다.** `asyncio.gather`로 수백 개를 "동시에" 돌려도, 실제로 같은 순간에 실행되는 코드는 단 하나입니다. GIL이 보장하는 단일 스레드 모델 그대로입니다. 비동기는 I/O 대기 시간을 겹쳐 처리량을 높일 뿐, CPU 작업을 병렬화하지 못합니다. 그래서 파이썬에서 진짜 멀티코어를 쓰려면 `multiprocessing`으로 프로세스를 쪼개고, 프로세스 간 데이터 전달은 피클링(pickling)·공유메모리·메시지큐로 우회해야 합니다. 비동기와 병렬을 한 프로세스에서 동시에 누리는 길이 막혀 있는 것입니다.

정리하면 `asyncio`의 구조적 약점은 세 가지입니다. (1) `await` 누락이 런타임에야 드러난다, (2) 블로킹/CPU 호출이 단일 스레드 루프를 얼린다, (3) 코어를 하나밖에 못 써서 비동기와 병렬이 분리되어 있다. Python 3.12+에서 `asyncio`가 멀티 이벤트 루프나 서브인터프리터(PEP 734) 방향으로 개선되고 있지만, "이벤트 루프 = 단일 스레드 협력 스케줄러"라는 핵심 모델과 위 세 함정은 여전히 유효합니다. Rust의 `tokio`는 이 세 가지를 각각 컴파일 경고, `spawn_blocking`, 멀티스레드 work-stealing으로 정면 돌파합니다.

---

## 18.2 The Rust Mental Model — `Future`는 게으른 상태 기계다

Rust는 이 문제를 정반대 방향에서 접근합니다. 파이썬이 "런타임에 협력적으로 양보하는 코루틴"이라면, Rust는 "컴파일 타임에 상태 기계로 변환되는, 폴링(polling)당하는 데이터 구조"입니다.

핵심 개념은 표준 라이브러리의 `Future` 트레이트 단 하나입니다.

```rust
// std::future::Future (단순화)
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // 결과가 나왔다
    Pending,     // 아직. 나중에 다시 폴링해달라
}
```

`async fn`을 호출하면 즉시 실행되는 게 아니라, **이 `Future` 트레이트를 구현하는 익명 구조체**가 하나 반환됩니다. 파이썬의 코루틴 객체와 정확히 같은 위치에 있습니다. 그리고 이 구조체는 누군가 `poll`을 호출해주기 전까지 아무 일도 하지 않습니다.

`poll`이 호출되면 future는 가능한 만큼 일을 진행하다가, I/O가 아직 안 끝났으면 `Poll::Pending`을 반환하고 멈춥니다. 일이 다 끝나면 `Poll::Ready(결과)`를 반환합니다. 누가 이 `poll`을 반복해서 호출해줄까요? 바로 **런타임(executor)**입니다. 그리고 Rust 표준 라이브러리에는 이 런타임이 **없습니다.** 트레이트라는 "약속"만 정의하고, 그것을 실행하는 엔진은 생태계(`tokio`, `async-std`, `smol`)에 맡겼습니다.

여기서 파이썬과의 가장 중요한 인식 차이가 드러납니다. 파이썬의 코루틴은 `await`을 만나면 **스스로** `yield`로 값을 이벤트 루프에 반납하는, 능동적인 주체에 가깝습니다(실제로 코루틴은 강화된 제너레이터 위에 구현됩니다). 반면 Rust의 future는 **수동적인 데이터**입니다. 자기가 능동적으로 양보하는 게 아니라, 외부의 executor가 "이제 진행할 수 있니?" 하고 `poll`을 호출해주면 그때 가능한 만큼만 전진합니다. 양보의 주도권이 코루틴(파이썬)에 있느냐, 폴링하는 executor(Rust)에 있느냐의 차이입니다. 이 "폴링 기반(poll-based)" 모델이 Rust 비동기의 모든 성질 — 게으름, 무비용 추상화, `Pin`의 필요성 — 의 뿌리입니다.

또 하나, "게으름"의 결과가 파이썬과 미묘하게 다릅니다. 파이썬에서 `await`을 빠뜨린 코루틴은 만들어진 채 GC를 기다리다 `RuntimeWarning`을 남깁니다. Rust에서 `.await`하지 않은 future는 그냥 즉시 드롭(drop)되어 사라집니다 — 부작용도, 경고도(컴파일러가 잡지 않는 한) 없습니다. future는 폴링되기 전까지 **단 한 줄의 코드도 실행하지 않으므로**, 만들었다 버려도 아무 일도 안 일어난 것과 같습니다. 이 철저한 게으름이 "수만 개의 future를 미리 만들어두고 `select!`로 골라 폴링한다" 같은 패턴을 공짜로 가능하게 합니다.

이 설계 철학을 파이썬과 한 표로 정리하면 다음과 같습니다.

| 개념 | Python `asyncio` | Rust + `tokio` |
|------|------------------|----------------|
| 비동기 함수 정의 | `async def` | `async fn` |
| 호출 시 반환물 | 코루틴 객체(Coroutine) | `impl Future` 구현 구조체 |
| 게으름(Laziness) | 게으름 (await/gather 전엔 안 돎) | 게으름 (poll 전엔 안 돎) |
| 실행 단위 | Task (`create_task`) | Task (`tokio::spawn` → `JoinHandle`) |
| 동시 대기 | `asyncio.gather(...)` | `join!` / `join_all` |
| 먼저 끝나는 것 선택 | `asyncio.wait(FIRST_COMPLETED)` | `select!` |
| **런타임 위치** | **언어 표준 내장** | **외부 크레이트(tokio 등)** |
| **스케줄러 모델** | **단일 스레드 이벤트 루프** | **멀티스레드 work-stealing(기본)** |
| 진입점 | `asyncio.run(main())` | `#[tokio::main] async fn main()` |
| 블로킹 작업 격리 | `loop.run_in_executor()` | `tokio::task::spawn_blocking()` |
| 비동기 락 | `asyncio.Lock` | `tokio::sync::Mutex` |
| await 누락 검출 | 런타임 경고(`RuntimeWarning`) | 컴파일 경고(`unused_must_use`) |
| 메모리 모델 | 힙 할당 코루틴 프레임(스택리스) | 스택리스 상태 기계 구조체(인라인) |

표의 마지막 다섯 줄이 이 챕터의 핵심입니다. **런타임이 외부에 있고, 스케줄러가 멀티스레드이며, await 누락과 락 오용을 컴파일러가 잡는다.** 이제 각 항목을 내부까지 파고듭니다.

---

## 18.3 Under the Hood — `async fn`은 어떻게 상태 기계가 되는가

이 챕터에서 단 하나만 기억해야 한다면 이것입니다. **Rust의 `async fn`은 컴파일 타임에 `enum` 기반 상태 기계로 변환됩니다.** 파이썬의 코루틴이 힙에 프레임 객체를 만들고 인터프리터가 바이트코드를 한 단계씩 돌리는 것과 달리, Rust는 함수 전체를 하나의 자료 구조로 압축합니다.

다음 비동기 함수를 봅시다.

```rust
async fn do_work() -> i32 {
    let x = 1;                    // (A) 첫 await 이전
    step_one().await;             // (B) 첫 일시정지 지점
    let y = x + 1;                // (C) 두 await 사이 (x를 여기서도 쓴다!)
    step_two(y).await;            // (D) 두 번째 일시정지 지점
    y * 10                        // (E) 최종 결과
}
```

컴파일러는 `.await` 지점을 경계로 함수를 쪼개고, 각 구간을 상태로 갖는 `enum`을 자동 생성합니다. 개념적으로 이런 모습입니다.

```rust
// 컴파일러가 내부적으로 만들어내는 코드의 "개념적" 모습
enum DoWorkStateMachine {
    // 아직 시작 안 함
    Start,
    // step_one().await에서 멈춤. 재개하려면 x를 보존해야 한다.
    WaitingOnStepOne { x: i32, fut1: StepOneFuture },
    // step_two().await에서 멈춤. y를 보존.
    WaitingOnStepTwo { y: i32, fut2: StepTwoFuture },
    // 완료됨 (다시 poll하면 panic)
    Done,
}
```

여기서 결정적인 통찰이 나옵니다. **변수 `x`와 `y`는 스택이 아니라 이 `enum` 안에 저장됩니다.** 왜냐하면 `step_one().await`에서 함수가 일시정지하면 스택 프레임이 사라지기 때문입니다. `x`가 스택에만 있었다면 재개할 때 사라져버립니다. 그래서 컴파일러는 "`await` 경계를 넘어 살아남아야 하는 모든 지역 변수"를 골라내 상태 기계 구조체의 필드로 끌어올립니다.

반대로 말하면, **`await` 경계를 넘지 않는 지역 변수는 상태 기계에 들어가지 않습니다.** 예를 들어 두 `await` 사이에서만 잠깐 쓰이고 소멸하는 임시 변수는 그 구간의 스택에만 존재합니다. 그래서 18.7에서 다룰 "락 가드를 `await` 경계 안에 두느냐 밖에 두느냐"가 결정적인 차이를 만듭니다. 같은 변수라도 `await`을 사이에 끼고 살아남으면 상태 기계의 필드가 되어 그 future 전체의 타입 성질(`Send` 여부)을 좌우하고, 끼지 않으면 그냥 스쳐 지나가는 스택 값일 뿐입니다. 컴파일러의 "어떤 변수가 await를 가로지르는가" 분석은 단순한 최적화가 아니라, future의 타입 자체를 결정하는 의미론입니다.

`poll`이 호출될 때마다 상태 기계는 현재 상태를 보고 다음으로 진행합니다.

```rust
// poll 구현의 개념적 모습 (실제로는 컴파일러가 생성)
fn poll(&mut self, cx: &mut Context) -> Poll<i32> {
    loop {
        match self {
            Start => {
                let x = 1;
                let fut1 = step_one();
                *self = WaitingOnStepOne { x, fut1 };
            }
            WaitingOnStepOne { x, fut1 } => {
                match fut1.poll(cx) {
                    Poll::Pending => return Poll::Pending,  // 아직 → 제어권 반납
                    Poll::Ready(()) => {
                        let y = *x + 1;
                        let fut2 = step_two(y);
                        *self = WaitingOnStepTwo { y, fut2 };
                    }
                }
            }
            WaitingOnStepTwo { y, fut2 } => {
                match fut2.poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => {
                        let result = *y * 10;
                        *self = Done;
                        return Poll::Ready(result);
                    }
                }
            }
            Done => panic!("polled after completion"),
        }
    }
}
```

> **Under the Hood — 스택리스 코루틴과 메모리 레이아웃**
>
> 이 상태 기계가 바로 **스택리스 코루틴(Stackless Coroutine)**입니다. 코루틴이 자기만의 별도 스택을 갖는 게 아니라(그것이 스택풀 코루틴, 예: Go의 고루틴), 일시정지 시점의 상태를 **하나의 값(value)**에 인라인으로 보존합니다.
>
> 결과적으로 future 하나의 메모리 크기는 **컴파일 타임에 완전히 고정**됩니다. 그 크기는 대략 "동시에 살아있어야 하는 모든 await-경계 변수 + 가장 큰 하위 future"의 합입니다. `enum`이므로 모든 변형(variant) 중 가장 큰 것의 크기로 정해집니다. 위 예에서 `x`와 `y`는 동시에 살아있지 않으므로 메모리가 겹쳐 재사용됩니다.
>
> 이 고정 크기 덕분에 future는 **힙 할당 없이** 호출자의 상태 기계 안에 직접 박혀 들어갑니다. 즉 `async fn`이 다른 `async fn`을 `await`하면, 자식 future가 부모 future 구조체의 필드가 되어 하나의 평평한(flat) 메모리 블록을 이룹니다. 파이썬이 코루틴 프레임마다 힙 객체(PyObject 헤더 + 딕셔너리)를 할당하고 GC가 관리하는 것과 정반대입니다. 컨텍스트 스위칭은 그저 `enum` 태그를 바꾸고 함수 포인터를 점프하는 것이라, 수십 나노초 수준입니다.

`async`/`await`을 한 줄도 쓰지 않고 이 상태 기계를 손으로 구현해보면 정체가 명확해집니다. 다음은 `n`번 폴링되어야 완료되는 future를 직접 만든, **컴파일·실행되는 완전한 코드**입니다.

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// "3번 poll되면 완료"되는 카운트다운 future를 손으로 구현한다.
struct CountdownFuture {
    remaining: u32,
}

impl Future for CountdownFuture {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.remaining == 0 {
            Poll::Ready("done")
        } else {
            self.remaining -= 1;
            // "다시 깨워달라"고 런타임에 신호. 데모용으로 즉시 재폴링 요청.
            cx.waker().wake_by_ref();
            Poll::Pending     // 아직 안 끝남 → 제어권 반납
        }
    }
}

#[tokio::main]
async fn main() {
    let fut = CountdownFuture { remaining: 3 };
    // .await은 결국 이 poll을 Ready가 나올 때까지 반복 호출하는 것이다.
    let result = fut.await;
    println!("{}", result); // done
}
```

여기서 `cx.waker()`가 등장합니다. `Poll::Pending`을 반환하기 전에, future는 "내 I/O가 준비되면 이 **Waker**를 호출해서 나를 다시 깨워달라"고 등록합니다. 실제 네트워크 future라면 OS의 `epoll`/`kqueue`에 소켓을 등록하고, 데이터가 도착했을 때 OS가 알려주면 Waker를 통해 런타임이 그 future를 다시 폴링합니다. 즉 런타임은 무의미하게 바쁜 폴링(busy-poll)을 하지 않습니다. Waker로 깨워진 future만 골라서 폴링합니다.

이것이 **실행기(executor)와 리액터(reactor)**의 분업입니다.

- **Executor(실행기)**: 준비된(ready) future들을 골라 `poll`을 호출하고, 완료된 작업을 회수하는 스케줄러. 여러 워커 스레드에 작업을 분배합니다.
- **Reactor(리액터)**: OS의 이벤트 통지 메커니즘(`epoll`, `kqueue`, `IOCP`)을 감싸, "어떤 소켓/타이머가 준비됐는지"를 감지해 해당 Waker를 호출하는 부분.

조금 더 구체적으로 그려보면 이렇습니다. 어떤 비동기 함수가 TCP 소켓에서 데이터를 읽으려고 `socket.read().await`를 호출합니다. 첫 `poll`에서 데이터가 아직 안 왔으면, 그 read future는 (1) 자신을 깨워줄 `Waker`를 `Context`에서 복제해 보관하고, (2) 그 소켓의 파일 디스크립터를 리액터의 `epoll` 인스턴스에 "읽기 준비되면 알려달라"고 등록한 뒤, (3) `Poll::Pending`을 반환합니다. 이제 executor는 이 태스크를 대기 목록으로 보내고 다른 준비된 태스크를 폴링합니다 — 이 스레드는 노는 시간 없이 다른 일을 합니다. 나중에 OS가 소켓에 데이터를 채우면, 리액터(보통 별도의 I/O 드라이버 스레드 또는 워커가 틈틈이 `epoll_wait`로 확인)가 이를 감지해 보관해둔 `Waker`를 호출하고, executor는 그 태스크를 다시 실행 큐에 넣어 폴링합니다. 이번에는 데이터가 있으니 read future가 `Poll::Ready(바이트)`를 반환하며 전진합니다.

이 흐름의 아름다움은 **바쁜 대기(busy-wait)가 전혀 없다**는 점입니다. 파이썬의 `asyncio`도 내부적으로 `selectors`(epoll/kqueue 래퍼)를 써서 같은 원리로 동작합니다. 차이는 파이썬은 이 selector 루프가 단일 스레드에 묶여 있고, Rust의 `tokio`는 여러 워커 스레드가 각자 폴링하며 리액터를 공유한다는 점입니다.

`tokio`는 executor와 reactor를 한 패키지로 묶어 제공합니다. 그리고 Rust 표준이 이를 내장하지 않은 이유가 바로 여기 있습니다. 임베디드(스레드 없는 단일 코어, `embassy`), 고성능 서버(멀티스레드 work-stealing, `tokio`), WASM(브라우저 이벤트 루프 위, `wasm-bindgen-futures`)은 각각 완전히 다른 executor/reactor를 필요로 합니다. 표준에 하나를 박아 넣으면 나머지 영역을 다 죽입니다. 게다가 비동기 런타임은 빠르게 진화하는 영역이라, 6주마다 릴리스되는 표준 라이브러리에 묶어두면 발전 속도가 발목 잡힙니다. 그래서 Rust는 `Future`·`Poll`·`Context`·`Waker`라는 **공통 인터페이스만 표준화**하고, 엔진은 시장 경쟁에 맡겼습니다. `async-std`, `smol` 등과의 경쟁에서 사실상의 표준(de facto standard) 자리를 차지한 승자가 `tokio`이며, 오늘날 대부분의 비동기 크레이트(`reqwest`, `axum`, `sqlx`, `tonic`)가 `tokio` 위에서 돕니다.

> **Under the Hood — 표준에 런타임이 없다는 설계 결정의 무게**
>
> 이 결정은 양날의 검입니다. 장점은 위에서 본 유연성입니다. 단점은 생태계 분열 가능성과, 라이브러리가 특정 런타임에 암묵적으로 종속되는 문제입니다. 어떤 크레이트가 내부에서 `tokio::spawn`을 호출하면, 그것을 쓰는 앱은 반드시 `tokio` 런타임 위에서 돌아야 합니다. 그래서 잘 만든 비동기 라이브러리는 런타임 비종속적으로 설계하거나, `tokio`/`async-std` 양쪽을 feature flag로 지원합니다. 파이썬은 `asyncio`가 표준이라 이런 고민이 없지만, 대신 "표준 `asyncio` vs `uvloop` vs `trio` vs `anyio`"라는 또 다른 분열을 겪고 있으니, 결국 비동기 런타임 다양성은 어느 언어든 피하기 어려운 숙제인 셈입니다.

---

## 18.4 런타임 비교 — 단일 스레드 이벤트 루프 vs work-stealing 스케줄러

`asyncio`는 단일 스레드를 시분할합니다. `tokio`는 기본적으로 CPU 코어 수만큼 워커 스레드를 띄우고, 수천 개의 태스크를 그 스레드들에 분배하는 **멀티스레드 work-stealing 스케줄러**입니다.

`#[tokio::main]` 매크로는 사실 런타임을 만들고 `main`을 그 위에서 돌리는 보일러플레이트를 숨긴 것입니다. 펼치면 이렇습니다.

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()          // 타이머·I/O 드라이버(리액터) 활성화
        .build()
        .unwrap()
        .block_on(async {
            // 여기가 원래의 async fn main() 본문
        });
}
```

런타임은 직접 구성할 수 있고, 이때 **워커 스레드 수**나 **단일 스레드 모드(`asyncio`와 가장 유사한 형태)**를 고를 수 있습니다. 다음은 컴파일·실행되는 완전한 예제입니다.

```rust
use futures::future::join_all;

async fn work(id: usize) -> usize {
    id * 2
}

fn main() {
    // 1) 멀티스레드 런타임: 워커 4개로 명시적 구성
    let rt = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(4)
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        let futs: Vec<_> = (0..5).map(work).collect();
        let results = join_all(futs).await; // 5개를 동시에 폴링
        println!("{:?}", results); // [0, 2, 4, 6, 8]
    });

    // 2) 단일 스레드 런타임: asyncio 이벤트 루프와 거의 동일한 모델
    let rt2 = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();

    rt2.block_on(async {
        println!("current_thread runtime OK");
    });
}
```

> **Under the Hood — Work-stealing의 작동 방식**
>
> work-stealing 스케줄러의 핵심은 각 워커 스레드가 자기만의 로컬 실행 큐(deque, 양방향 큐)를 갖는다는 점입니다. 새 태스크는 현재 워커의 로컬 큐 맨 위에 쌓이고, 워커는 자기 큐를 LIFO로 소비합니다(캐시 지역성이 좋습니다 — 방금 만든 태스크가 캐시에 따뜻하게 남아 있습니다).
>
> 어떤 워커의 로컬 큐가 비면, 그 워커는 놀지 않고 **다른 워커의 큐 맨 아래(FIFO 쪽)에서 태스크를 훔쳐옵니다.** 이것이 "work-stealing"입니다. 결과적으로 부하가 자동으로 평준화됩니다. 한 핸들러가 폭주해도 다른 코어가 일을 가져가니, 단일 스레드 `asyncio`처럼 한 곳의 정체가 전체를 마비시키지 않습니다.
>
> 파이썬으로 비유하면 `asyncio`(I/O 동시성)와 `multiprocessing`(코어 병렬성)을 동시에, 그것도 GIL 없이 메모리를 공유하며 쓰는 것과 같습니다. 단, 이 자유에는 대가가 있습니다. 태스크가 스레드를 넘나들 수 있으므로, `tokio::spawn`에 넘기는 future는 **`Send`여야 합니다**(Chapter 17의 그 `Send`입니다). 이 제약이 18.7에서 만날 컴파일 에러의 정체입니다.

이 멀티스레드 아키텍처 덕분에 Rust 웹 프레임워크(Axum, Actix-web)가 TechEmpower 같은 벤치마크 상위권을 놓치지 않습니다. 단, 여기서 **동시성(Concurrency)과 병렬성(Parallelism)**을 구분해야 합니다. `join!`/`select!`로 한 태스크 안에서 여러 future를 번갈아 폴링하는 것은 동시성입니다(한 스레드에서 일어날 수 있음). `tokio::spawn`으로 태스크를 쪼개 여러 워커 스레드에 분산시키는 것이 병렬성을 끌어냅니다. 이 둘의 도구가 다음 절의 주제입니다.

---

## 18.5 태스크와 동시성 도구 — spawn, join!, select!

파이썬에서 `asyncio.create_task()`로 코루틴을 백그라운드 태스크로 던지듯, `tokio`는 `tokio::spawn`으로 future를 런타임에 등록합니다. 차이는 `tokio::spawn`이 즉시 그 future를 **다른 워커 스레드로 보낼 수 있다**는 점, 그리고 결과 회수용 핸들(`JoinHandle`)을 돌려준다는 점입니다.

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn fetch_data(id: i32) -> String {
    println!("Fetching {}...", id);
    sleep(Duration::from_secs(1)).await; // tokio가 제공하는 비동기 sleep
    format!("Data {}", id)
}

#[tokio::main]
async fn main() {
    // (1) join!: 여러 future를 같은 태스크에서 동시에 폴링하고 모두 기다림
    //     asyncio.gather와 가장 유사. 1초 대기가 합쳐져 총 ~1초.
    let (res1, res2) = tokio::join!(fetch_data(1), fetch_data(2));
    println!("Results: {}, {}", res1, res2);

    // (2) spawn: 작업을 런타임에 "던지고" JoinHandle을 회수
    //     spawn된 태스크는 다른 워커 스레드에서 병렬로 돌 수 있다.
    let handle = tokio::spawn(async { 40 + 2 });
    let val = handle.await.unwrap(); // JoinHandle도 await 가능
    println!("spawned result: {}", val);

    // (3) select!: 여러 future 중 먼저 끝나는 하나만 취하고 나머지는 취소
    //     타임아웃·취소 패턴의 기본 도구.
    tokio::select! {
        _ = sleep(Duration::from_millis(10)) => println!("fast branch won"),
        _ = sleep(Duration::from_secs(10))   => println!("slow branch won"),
    }
}
```

세 도구의 역할을 분명히 합시다.

- **`join!`** / **`futures::join_all`**: "전부 다 끝날 때까지 동시에." 파이썬 `asyncio.gather`. 한 태스크 내부에서 future들을 인터리빙(interleaving)하므로, 기본적으로는 동시성이지 병렬성이 아닙니다.
- **`tokio::spawn`**: "이걸 별도 태스크로 떼어내서 워커 스레드에 분산." 병렬성을 얻는 길. 반환된 `JoinHandle`을 `await`하면 결과를, `abort()`하면 취소를 할 수 있습니다.
- **`select!`**: "먼저 끝나는 쪽을 취하고 나머지는 그 자리에서 드롭." 타임아웃(`tokio::time::timeout`은 내부적으로 이걸 씁니다), graceful shutdown 신호 대기, 첫 응답 우선 등에 쓰입니다.

`gather`와 `join!`의 미묘한 차이 하나: 파이썬 `gather`는 내부적으로 각 코루틴을 태스크로 만들어 등록하지만, Rust `join!`은 그냥 한 future 안에서 여러 하위 future를 폴링할 뿐 spawn하지 않습니다. 그래서 `join!`은 추가 할당이 전혀 없고, 진짜 병렬 분산을 원하면 `spawn`을 명시적으로 써야 합니다. Rust는 "공짜로 스레드에 흩뿌리는 마법" 대신 의도를 코드로 드러내게 합니다.

`spawn`과 `JoinHandle`의 의미론도 짚어둘 가치가 있습니다. `tokio::spawn`은 호출 즉시 future를 런타임 스케줄러에 등록하므로, `JoinHandle`을 `await`하기 전부터 그 태스크는 백그라운드에서 돌기 시작합니다(파이썬 `create_task`와 같습니다). 이는 `join!`에 그냥 넘긴 future가 폴링되기 전까지 잠자코 있는 것과 대비됩니다. 또한 `JoinHandle`을 `await`하면 `Result<T, JoinError>`가 나옵니다 — 태스크가 패닉했거나 `abort()`로 취소됐을 수 있으니, 그 실패가 `JoinError`로 회수되는 것입니다. 파이썬에서 태스크 내부 예외가 조용히 삼켜져 `Task exception was never retrieved`로 뒤늦게 나타나던 것과 달리, Rust는 `JoinHandle`의 `Result`를 타입으로 강제해 처리를 잊지 못하게 합니다.

`select!`에는 한 가지 함정이 따라옵니다. 선택받지 못한 가지의 future는 그 자리에서 **드롭**됩니다. 만약 그 future가 진행 중이던 작업(예: 절반쯤 보낸 요청)을 들고 있었다면, 그 작업은 취소됩니다. 이것을 "취소 안전성(Cancellation Safety)"이라 부르며, 루프 안에서 `select!`를 반복할 때 특히 주의해야 합니다. 어떤 future는 중간에 드롭돼도 안전하지만(`tokio::time::sleep`), 어떤 것은 데이터 일부를 잃을 수 있습니다(버퍼에서 읽다 만 스트림). 파이썬 `asyncio.wait`에서도 취소된 태스크 처리가 까다로웠던 그 문제가, Rust에서는 `select!`의 의미론으로 명시화된 것입니다.

---

## 18.6 블로킹 코드와 CPU 작업 격리 — spawn_blocking

18.1에서 본 `asyncio`의 가장 큰 함정, "블로킹 호출이 루프를 얼린다"는 Rust에서도 **똑같이** 발생합니다. `tokio` 워커 스레드 위에서 `std::thread::sleep`이나 무거운 CPU 루프를 돌리면, 그 워커 스레드 하나가 통째로 막힙니다. 워커가 8개라면 8분의 1의 처리량이 사라지고, 여러 핸들러가 동시에 블로킹하면 데드락에 가까운 정체가 옵니다.

차이는 처방의 명확함입니다. `tokio`는 **블로킹 전용 스레드풀**을 따로 두고, `spawn_blocking`으로 거기에 작업을 격리합니다. 이는 파이썬의 `loop.run_in_executor(ThreadPoolExecutor, ...)`에 직접 대응합니다.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

// (A) CPU-바운드 작업은 spawn_blocking으로 격리한다.
//     이 클로저는 별도의 블로킹 스레드풀에서 돌아 워커 스레드를 막지 않는다.
async fn heavy_compute() -> u64 {
    tokio::task::spawn_blocking(|| {
        // 무거운 동기 연산 (CPU 바운드). 여기서 막혀도 async 워커는 멀쩡하다.
        (0..1_000_000u64).sum()
    })
    .await
    .unwrap()
}

#[tokio::main]
async fn main() {
    println!("sum = {}", heavy_compute().await);
    let _ = Arc::new(Mutex::new(0)); // 다음 절 예고
}
```

판단 기준은 명확합니다.

- **비동기 I/O**(네트워크, 비동기 DB 드라이버, 타이머): 그냥 `.await`. 워커를 막지 않습니다.
- **블로킹 I/O**(동기 파일 시스템 호출, 동기 DB 드라이버, 레거시 C 라이브러리): `spawn_blocking`.
- **순수 CPU 작업**(이미지 처리, 압축, 대규모 계산): `spawn_blocking`, 혹은 진짜 데이터 병렬이면 Chapter 16에서 본 `rayon`을 별도로 띄우는 것이 더 낫습니다. `tokio`의 비동기 워커는 I/O 동시성을 위한 것이지 CPU 병렬 계산을 위한 게 아니기 때문입니다.

핵심 원칙: **`async` 함수 본문에서 `.await` 없이 수 밀리초 이상 CPU를 잡거나 블로킹하면, 그 자리에서 멈춰야 한다.** 파이썬에서는 이걸 어겨도 "느려질" 뿐 컴파일은 됩니다. Rust에서도 컴파일은 되지만(블로킹은 타입으로 표현되지 않으므로), 적어도 `tokio`는 디버그 빌드에서 워커가 너무 오래 막히면 경고를 출력하는 옵션(`tokio-console`, 런타임 메트릭)을 제공합니다.

`spawn_blocking`의 내부도 들여다볼 가치가 있습니다. `tokio`는 비동기 워커 스레드풀과는 **완전히 별개의 블로킹 스레드풀**을 운영합니다. 비동기 워커는 코어 수만큼(보통 8~16개) 고정되지만, 블로킹 풀은 기본적으로 최대 512개까지 동적으로 늘어납니다. 왜 이렇게 많을까요? 블로킹 작업은 본질적으로 스레드 하나를 통째로 점유하므로, 동시에 N개의 블로킹 작업을 처리하려면 N개의 스레드가 필요하기 때문입니다. 반대로 비동기 워커는 적은 수로 수만 개의 태스크를 인터리빙하니 코어 수면 충분합니다. 이 분리가 핵심입니다 — 블로킹 작업이 아무리 많아도 비동기 워커 풀은 오염되지 않고, 비동기 I/O는 계속 진행됩니다. 파이썬의 `ThreadPoolExecutor`(블로킹용)와 이벤트 루프(비동기용)의 분리와 정확히 같은 발상이되, GIL이 없으니 블로킹 스레드가 진짜로 다른 코어에서 병렬 실행된다는 점이 다릅니다.

다만 `spawn_blocking`에 넘긴 클로저 안에서는 `.await`를 쓸 수 없습니다(그곳은 비동기 컨텍스트가 아닙니다). 비동기 작업과 무거운 동기 작업이 뒤섞여 있다면, 동기 부분만 `spawn_blocking`으로 떼어내 결과를 받아온 뒤 비동기 부분을 이어가는 식으로 경계를 명확히 나눠야 합니다.

---

## 18.7 공유 상태와 Pin — 컴파일러가 막아주는 함정

이제 Chapter 17의 `Arc<Mutex<T>>`를 비동기 세계로 가져올 차례입니다. 그런데 여기서 파이썬 개발자가 가장 자주 밟는 지뢰가 있습니다. **락을 든 채 `.await` 하기.**

파이썬에서는 이렇게 써도 그냥 동작합니다(다만 그동안 락을 잡고 있어 동시성이 죽을 뿐입니다).

```python
async def increment(lock: asyncio.Lock, state: dict) -> None:
    async with lock:
        state["count"] += 1
        await some_io()        # 락을 든 채 await — 파이썬은 허용 (데드락 위험만 있음)
```

Rust에서 `std::sync::Mutex`로 똑같이 하면, **컴파일러가 거부합니다.** 다음은 의도적으로 틀린 코드입니다.

```rust
use std::sync::{Arc, Mutex};

// 🚨 컴파일 에러를 유발하는 코드
async fn bad(data: Arc<Mutex<i32>>) {
    let mut guard = data.lock().unwrap();  // std MutexGuard 획득
    *guard += 1;
    tokio::time::sleep(std::time::Duration::from_millis(1)).await; // 가드를 든 채 await!
    println!("{}", *guard);
}

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0));
    tokio::spawn(bad(data)); // spawn은 future가 Send이길 요구
}
```

`rustc`가 내뱉는 실제 에러 메시지입니다.

```text
error: future cannot be sent between threads safely
   --> src/main.rs:14:18
    |
 14 |     tokio::spawn(bad(data)); // spawn은 Future가 Send이길 요구
    |                  ^^^^^^^^^ future returned by `bad` is not `Send`
    |
    = help: within `impl Future<Output = ()>`, the trait `Send` is not
            implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:7:61
    |
  5 |     let mut guard = data.lock().unwrap();
    |         --------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
  6 |     *guard += 1;
  7 |     tokio::time::sleep(...).await; // 가드를 든 채 await
    |                            ^^^^^ await occurs here, with `mut guard` maybe used later
note: required by a bound in `tokio::spawn`
```

**이 에러를 해독해봅시다.** 핵심은 18.3에서 배운 상태 기계입니다. `guard`가 `.await` 경계를 넘어 살아남아야 하므로, 컴파일러는 `guard`를 상태 기계 구조체의 **필드로 끌어올립니다.** 그런데 `std::sync::MutexGuard`는 `Send`가 아닙니다(OS 뮤텍스는 락을 건 스레드가 풀어야 하는 경우가 많아, 스레드 간 이동이 안전하지 않기 때문입니다). 따라서 그 가드를 품은 상태 기계 전체가 `Send`가 아니게 되고, work-stealing 스케줄러에 태스크를 던지는 `tokio::spawn`(`F: Send` 요구)이 이를 거부합니다.

파이썬이라면 런타임에 데드락이 나거나 동시성이 조용히 죽었을 버그를, Rust는 **컴파일 타임에 타입으로 잡아낸 것입니다.** Chapter 17에서 본 `Send` 마커 트레이트가 비동기 상태 기계에까지 전파되어 작동하는 모습입니다.

해결책은 두 가지입니다.

```rust
use std::sync::Arc;
use tokio::sync::Mutex; // 비동기 전용 Mutex

// 해결 1: tokio::sync::Mutex 사용. 가드가 Send이고, lock()이 .await이다.
async fn ok_async_mutex(data: Arc<Mutex<i32>>) {
    let mut guard = data.lock().await;   // 비동기적으로 락 대기
    *guard += 1;
    tokio::time::sleep(std::time::Duration::from_millis(1)).await; // OK
    println!("tokio mutex: {}", *guard);
}

// 해결 2: std Mutex라도 락 스코프를 await '이전'에 닫기 (대개 이쪽이 더 빠르다)
async fn ok_scoped_std(data: Arc<std::sync::Mutex<i32>>) {
    {
        let mut guard = data.lock().unwrap();
        *guard += 1;
    } // 여기서 guard가 drop → 락 해제. await 경계를 넘지 않는다.
    tokio::time::sleep(std::time::Duration::from_millis(1)).await; // OK
}
```

> 💡 실무 지침: **기본은 `std::sync::Mutex` + 짧은 스코프**입니다. 락 안에서 `.await`할 일이 없다면(대부분 그렇습니다 — 카운터 증가, 캐시 업데이트 등) `std` 뮤텍스가 더 가볍고 빠릅니다. `tokio::sync::Mutex`는 "락을 든 채로 비동기 I/O를 해야만 하는" 드문 경우에만 씁니다. 비동기 뮤텍스의 가드는 더 무겁고(상태 기계에 박히고 await 가능한 락 대기를 지원해야 하므로), 남용하면 직렬화 병목이 됩니다.

### Pin과 Unpin — 왜 `Pin<&mut Self>`인가

`Future::poll`의 시그니처에 등장하는 `Pin<&mut Self>`을 기억하실 겁니다. 왜 평범한 `&mut Self`가 아닐까요? 18.3에서 본 상태 기계 때문입니다.

`async` 블록 안에서 지역 변수에 대한 참조를 만든 뒤 `.await`을 넘기면, 그 참조와 참조 대상이 **같은 상태 기계 구조체 안에** 함께 들어갑니다. 즉 구조체가 자기 자신의 다른 필드를 가리키는 **자기참조 구조체(Self-referential Struct)**가 됩니다. 이런 구조체를 메모리상에서 그냥 이동(move)시키면, 내부 포인터가 옛 주소를 가리키게 되어 댕글링(dangling)이 됩니다.

`Pin<&mut T>`은 "이 값은 이 메모리 위치에 고정되었으며, 이동시키지 않겠다"는 타입 수준의 약속입니다. 그래서 future를 폴링하려면 먼저 한 위치에 고정(pin)해야 하고, `poll`은 `Pin<&mut Self>`를 받습니다. 대부분의 타입은 자기참조가 없어 자유롭게 이동해도 되는데, 이런 타입은 `Unpin` 마커 트레이트를 자동 구현합니다. 실무에서는 `tokio::pin!` 매크로나 `Box::pin`이 이 작업을 대신해주므로, `Pin`을 직접 다룰 일은 드뭅니다. 다만 "왜 future는 함부로 옮기면 안 되는가"의 답이 자기참조 상태 기계라는 것만 기억하면 됩니다.

조금 더 풀어보면, 문제의 핵심은 "이동(move)"의 의미에 있습니다. Chapter 4에서 본 것처럼 Rust에서 값의 이동은 단순히 바이트를 새 메모리 위치로 복사하고 옛 위치를 무효화하는 것입니다. 일반 구조체라면 안에 자기참조가 없으니 이 복사가 안전합니다. 그런데 상태 기계 안에 `let s = String::from("hi"); let r = &s; some_io(r).await;` 같은 코드가 있으면, `r`(s를 가리키는 포인터)과 `s`(실제 데이터)가 같은 구조체에 들어갑니다. 이 구조체를 다른 주소로 이동시키면 `s`는 새 주소로 따라가지만 `r`은 옛 주소를 그대로 가리켜 댕글링이 됩니다. 메모리 안전을 컴파일 타임에 보장하는 Rust가 이런 구멍을 허용할 리 없습니다. 그래서 `Pin`이라는 타입으로 "폴링을 시작한 future는 절대 이동시키지 않는다"는 불변식을 강제하고, 그 위에서 자기참조를 안전하게 허용합니다. 파이썬에는 객체가 항상 힙에 고정되어 절대 이동하지 않으므로(변수는 그저 객체를 가리키는 이름표일 뿐) 이런 개념이 아예 필요 없습니다. `Pin`은 Rust의 "값이 이동할 수 있다"는 근본 성질이 비동기와 만나면서 생긴 고유한 장치입니다.

좋은 소식은, `async`/`await`을 쓰는 99%의 코드에서 `Pin`을 직접 타이핑할 일이 없다는 것입니다. `.await` 한 줄이 컴파일러가 생성한 핀 처리 코드를 전부 숨겨주고, 직접 future를 다뤄야 할 때도 `tokio::pin!`이나 `Box::pin`이 보일러플레이트를 대신합니다. `Pin`은 "이 추상화가 어떻게 안전한가"를 설명하는 토대일 뿐, 일상적으로 씨름할 대상은 아닙니다.

---

## 18.8 실전 Hands-on — `aiohttp` vs `reqwest`+`tokio` 동시 크롤러 벤치마크

이제 모든 개념을 묶어 현실적인 시나리오를 만듭니다. **여러 URL에 동시에 HTTP 요청을 보내 상태 코드를 수집하는 크롤러**를 양쪽 언어로 작성하고 처리량을 비교합니다.

### The Python Way — aiohttp

```python
import asyncio, time
import aiohttp

async def fetch_status(session: aiohttp.ClientSession, url: str) -> int:
    async with session.get(url) as resp:
        return resp.status

async def crawl(urls: list[str]) -> list[int]:
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_status(session, u) for u in urls]
        return await asyncio.gather(*tasks)

if __name__ == "__main__":
    urls = ["https://example.com"] * 500   # 동시 요청 500개
    start = time.perf_counter()
    codes = asyncio.run(crawl(urls))
    print(f"{len(codes)} requests in {time.perf_counter() - start:.2f}s")
```

### The Rust Implementation — reqwest + tokio

다음은 `reqwest`(features = ["json"])와 `tokio`(features = ["full"])를 의존성으로 추가한 뒤 `cargo run --release`로 그대로 도는 완전한 코드입니다.

```rust
use std::time::Instant;

async fn fetch_status(client: &reqwest::Client, url: &str) -> Result<u16, reqwest::Error> {
    let resp = client.get(url).send().await?;
    Ok(resp.status().as_u16())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 커넥션 풀을 가진 클라이언트는 한 번만 만들어 공유(clone은 내부적으로 Arc).
    let client = reqwest::Client::new();
    let urls = vec!["https://example.com"; 500]; // 동시 요청 500개

    let start = Instant::now();

    // 각 요청을 태스크로 spawn → work-stealing 스케줄러가 워커들에 분산(진짜 병렬).
    let mut handles = Vec::with_capacity(urls.len());
    for url in urls {
        let client = client.clone();      // Arc clone (값 복사 아님)
        let url = url.to_string();
        handles.push(tokio::spawn(async move {
            fetch_status(&client, &url).await
        }));
    }

    // JoinHandle들을 회수. ? 연산자(Chapter 13)로 에러를 깔끔히 전파.
    let mut ok = 0usize;
    for h in handles {
        match h.await? {
            Ok(_code) => ok += 1,
            Err(e) => eprintln!("error: {}", e),
        }
    }

    println!("{} requests in {:?}", ok, start.elapsed());
    Ok(())
}
```

이 코드는 Chapter 13의 `?` 연산자, Chapter 17의 `Arc`(여기선 `Client::clone` 내부에 숨어 있음), 그리고 이번 챕터의 `spawn`/`JoinHandle`/`await`을 한 데 엮은 종합 예제입니다.

### 벤치마크 — 무엇을 비교하는가

같은 머신(8코어), 동일 네트워크에서 **500개 동시 요청**을 측정하면, 순수 I/O 대기가 지배적인 워크로드라 두 언어의 벽시계 시간(wall-clock) 차이는 크지 않을 수 있습니다. 둘 다 "500개를 동시에 날리고 기다리는" 작업이고, 병목이 네트워크라면 언어 속도는 가려지기 때문입니다. 정직하게 말해, I/O가 진짜 병목이면 비동기 크롤러의 언어별 처리 시간은 비슷합니다.

진짜 차이는 **요청당 CPU 오버헤드와 메모리, 그리고 동시성 한계**에서 드러납니다. 현실적인 경향치는 다음과 같습니다(`--release` 빌드, 응답 파싱·다수 커넥션 유지 시나리오 기준의 일반적 관찰값).

| 항목 | Python `aiohttp` | Rust `reqwest`+`tokio` |
|------|------------------|------------------------|
| 500 동시 요청 벽시계 시간 (I/O 병목) | 기준 | 거의 동일 (±차이 작음) |
| 요청당 CPU 시간 (파싱·스케줄링 포함) | 기준 | 약 3~10배 적음 |
| 단일 프로세스 최대 동시 커넥션 (코어 활용) | 단일 스레드 한계 | 멀티코어로 수 배 |
| 태스크당 메모리 (수만 태스크 시) | 코루틴 프레임(힙, 수 KB급) | 상태 기계(수십~수백 바이트급) |
| CPU 섞인 워크로드(응답 후 가공) | 루프 정지 위험 | 워커 분산으로 안정 |

왜 이런 결과가 나오는지 메커니즘으로 설명하면 명확합니다. 500개 요청 각각이 평균 100ms의 네트워크 왕복을 기다린다면, 그 100ms 동안 두 언어 모두 CPU는 거의 놀고 있습니다. 벽시계 시간은 "가장 느린 요청이 끝날 때까지"로 수렴하므로, 네트워크가 병목인 한 언어 속도는 가려집니다. 측정 환경에 따라 다르지만, 단일 호스트로 500개를 동시에 칠 때 두 구현 모두 대략 같은 수백 밀리초~수 초 대에 들어옵니다.

차이가 벌어지는 지점은 따로 있습니다. 첫째, **요청당 CPU 오버헤드**입니다. 파이썬은 요청마다 코루틴 객체·태스크·콜백을 힙에 할당하고 GC가 추적하며, 이벤트 루프의 디스패치도 인터프리터 바이트코드로 돕니다. Rust는 상태 기계가 인라인된 값이라 할당이 거의 없고, 폴링 디스패치는 네이티브 코드입니다. 그래서 동시 요청 수가 1만, 10만으로 올라가면 파이썬은 스케줄링 오버헤드와 GC 압력이 누적되어 단일 코어가 포화되는 반면, Rust는 8개 워커에 분산해 한참 더 버팁니다. 둘째, **응답 후 가공**입니다. 받은 JSON을 파싱하고 집계하는 CPU 작업이 끼면, 파이썬은 그 작업이 단일 루프를 점유해 동시성이 무너지지만(이때 `run_in_executor`로 빼야 함), Rust는 work-stealing이 워커 간에 자동 분산하거나 `spawn_blocking`으로 격리합니다. 셋째, **메모리**입니다. 태스크 하나가 파이썬에서는 수 KB 규모의 객체 그래프인데, Rust 상태 기계는 수십~수백 바이트입니다. 10만 동시 태스크를 띄우면 이 차이가 수백 MB 대 수 GB로 벌어집니다.

핵심 결론: **순수 I/O 동시성만 보면 `asyncio`도 충분히 빠릅니다.** Rust로 갈아탈 실익은 (1) 응답을 받은 뒤 CPU 가공(파싱·집계·압축)이 섞일 때, (2) 동시 커넥션이 수만 단위로 올라가 단일 스레드 루프가 한계에 부딪힐 때, (3) `await` 누락·락 오용 같은 동시성 버그를 컴파일 타임에 봉쇄하고 싶을 때입니다. "빠르다"가 아니라 "예측 가능하고 안전하게 확장된다"가 Rust 비동기의 진짜 가치입니다. I/O 워크로드에서 "50배 빠르다" 같은 배수는 거짓말이며, 정직한 수치는 위 표의 "요청당 CPU 3~10배, 메모리 수 배"입니다. 진짜 50배는 PyO3로 CPU 병목을 들어낼 때(Chapter 22) 나오는 숫자입니다.

---

## 18.9 Best Practices & 안티패턴

실무 체크리스트입니다.

**이렇게 하라**

- **런타임은 진입점에서 한 번만.** `#[tokio::main]`이나 명시적 `Runtime::block_on`을 앱 최상단에 한 번 두고, 라이브러리 코드는 런타임에 의존하지 않게 `async fn`만 노출합니다.
- **공유 클라이언트는 만들어 재사용.** `reqwest::Client`, DB 풀은 한 번 생성해 `clone`(내부 `Arc`)으로 공유합니다. 요청마다 새로 만들면 커넥션 풀과 TLS 핸드셰이크가 낭비됩니다.
- **락은 짧게, `await` 이전에 닫아라.** 기본은 `std::sync::Mutex` + 좁은 스코프. 가드를 `await` 경계 너머로 들고 가지 마세요.
- **CPU/블로킹은 `spawn_blocking`(혹은 `rayon`)으로 격리.** 비동기 워커 위에서 무거운 동기 작업을 돌리지 마세요.
- **타임아웃을 걸어라.** 외부 I/O에는 `tokio::time::timeout`을 씌워 무한 대기를 막습니다. `select!`로 graceful shutdown 신호를 함께 기다리세요.
- **`clippy`를 켜라.** `#[warn(clippy::await_holding_lock)]` 린트가 락-보유-중-await를 추가로 잡아줍니다.

**이건 피하라(안티패턴)**

- 모든 것을 `tokio::spawn`으로 흩뿌리기 — 의존 관계가 있는 작업은 `join!`이면 충분합니다. 불필요한 spawn은 `Send` 제약과 스케줄링 오버헤드만 늘립니다.
- 비동기 함수 안에서 `std::thread::sleep` 호출 — 워커 스레드를 통째로 얼립니다. 반드시 `tokio::time::sleep`.
- `block_on`을 비동기 컨텍스트 안에서 호출 — 런타임 안에서 런타임을 또 돌리려다 패닉/데드락. 중첩 금지.
- 무분별한 `tokio::sync::Mutex` — 대부분의 경우 `std` 뮤텍스가 더 빠릅니다.

---

## 18.10 흔한 함정 — 파이썬 습관이 부르는 사고

파이썬 비동기 습관이 Rust에서 어떻게 어긋나는지, 그리고 어떻게 고치는지 정리합니다.

**함정 1 — `.await`을 빠뜨린다.**

파이썬에서 `await`을 빠뜨리면 런타임 경고로 끝나지만, Rust에서는 future가 게을러서 **아무 일도 일어나지 않습니다.** 다만 컴파일러가 경고로 잡아줍니다.

```rust
async fn my_task() -> i32 {
    println!("Task started!");
    42
}

#[tokio::main]
async fn main() {
    my_task(); // .await을 빠뜨림 → future만 만들고 버림
}
```

`rustc` 경고:

```text
warning: unused implementer of `Future` that must be used
 --> src/main.rs:8:5
  |
8 |     my_task(); // .await을 빠뜨림
  |     ^^^^^^^^^
  |
  = note: futures do nothing unless you `.await` or poll them
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
```

`Future`에 붙은 `#[must_use]` 속성 덕분입니다. 파이썬의 사후 `RuntimeWarning`과 달리, 빌드 시점에 "이 future를 `.await` 안 했다"고 알려줍니다. CI에서 `-D warnings`로 경고를 에러 승격시키면 아예 머지를 막을 수 있습니다.

**함정 2 — 락을 든 채 `await`.**

18.7에서 본 `future is not Send` 에러가 바로 이것입니다. 파이썬에서는 동작하던(다만 동시성을 죽이던) 패턴이, Rust에서는 컴파일 에러입니다. 해결: 락 스코프를 `await` 이전에 닫거나, 꼭 필요하면 `tokio::sync::Mutex`.

**함정 3 — 블로킹 호출을 async에 섞는다.**

`time.sleep` → `tokio::time::sleep`, 동기 DB 드라이버 → 비동기 드라이버 또는 `spawn_blocking`. 이건 타입으로 표현되지 않으므로(블로킹은 트레이트가 아닙니다) 컴파일러가 못 잡습니다. **개발자의 규율과 `tokio-console`/런타임 메트릭으로 감시**해야 하는, Rust에서도 여전히 위험한 유일한 함정입니다.

**함정 4 — `join!`이면 될 것을 매번 `spawn`.**

`spawn`은 `'static` + `Send`를 요구하므로 라이프타임이 짧은 지역 참조를 캡처할 수 없습니다. "이 future들을 그냥 동시에 기다리고 싶다"면 `join!`/`join_all`이 정답입니다. 컴파일러가 `` `data` does not live long enough `` 류의 에러로 막아주는데, 이때 답은 무리하게 `Arc`로 감싸는 게 아니라 `spawn`을 `join!`으로 바꾸는 것입니다.

**함정 5 — 동시성을 병렬성으로 착각.**

`join!`으로 묶은 future들은 한 태스크에서 인터리빙됩니다. 그 안에 CPU 작업이 있으면 서로를 막습니다(다른 future가 폴링될 틈이 없으므로). 진짜 코어 분산이 필요하면 `spawn`으로 태스크를 쪼개야 합니다. 파이썬 `asyncio`에는 애초에 병렬성 선택지가 없었으니, 이 구분 자체가 Rust에서 새로 생기는 사고의 도구입니다.

**함정 6 — 동기 컨텍스트에서 비동기를 호출하려 한다.**

파이썬에서는 `asyncio.run(coro())`을 동기 함수 아무 데서나 부를 수 있어, 비동기와 동기의 경계가 느슨합니다. Rust에서 동기 함수가 `async fn`의 결과를 쓰려면 런타임의 `block_on`을 명시적으로 호출하거나, 애초에 그 동기 함수를 `async fn`으로 바꿔야 합니다. 흔한 실수는 라이브러리 내부에서 임의로 `Runtime::new().block_on(...)`을 만드는 것인데, 이미 `tokio` 런타임 위에서 돌고 있는 상황이라면 "런타임 안의 런타임"이 되어 `Cannot start a runtime from within a runtime` 패닉이 납니다. 비동기는 한 번 시작하면 호출 스택을 따라 위로 전파됩니다("async는 색을 칠한다(function coloring)"고들 합니다). 그래서 비동기를 쓸 함수는 진입점부터 `async`로 설계하고, 동기 세계와의 경계는 앱 최상단 한 곳에서만 `#[tokio::main]`으로 넘는 것이 정석입니다.

이 여섯 가지 함정을 관통하는 교훈은 하나입니다. 파이썬에서 비동기는 "느슨하게 섞어 써도 일단 돌아가는" 편의 기능이었지만, Rust에서 비동기는 **타입과 컴파일러가 지키는 별도의 실행 모델**입니다. 처음에는 `Send` 경계나 `block_on` 패닉이 성가시게 느껴지지만, 그 제약 하나하나가 파이썬에서라면 프로덕션 새벽에 호출기를 울렸을 동시성 버그를 빌드 단계에서 봉쇄한 것임을 깨닫게 됩니다. 컴파일러를 동시성 버그를 잡아주는 페어 프로그래머로 받아들이는 순간, 비동기 Rust는 두려움(fear)이 아니라 담대함(fearless)의 영역이 됩니다.

---

지금까지 Part 6 "GIL의 해방과 동시성"의 마지막 조각을 맞췄습니다. Chapter 16의 진짜 멀티스레딩, Chapter 17의 `Arc`/`Mutex`/`Send`/`Sync`, 그리고 이번 챕터의 `async`/`await`까지 — Rust의 동시성 도구함이 완성됐습니다. 파이썬이 GIL과 단일 스레드 이벤트 루프라는 두 겹의 족쇄로 묶어두었던 것을, Rust는 컴파일러가 안전을 보증하는 멀티코어 동시성·병렬성으로 풀어냈습니다.

그리고 이 모든 안전 장치 뒤에는 한 가지 공통된 마법이 있었습니다. `async fn`이 상태 기계로, `?`가 에러 전파 코드로, `#[derive(Debug)]`가 출력 코드로 **컴파일 타임에 변환**되는 능력입니다. 다음 파트(Part 7)에서는 이 변환을 우리 손으로 직접 일으키는 도구, 파이썬의 데코레이터와 메타클래스를 런타임 오버헤드 없이 대체하는 **매크로(Macro)**의 세계로 들어갑니다. Chapter 19에서 `macro_rules!`와 절차적 매크로로 "코드를 짜는 코드"를 작성해봅니다.
