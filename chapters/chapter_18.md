# Chapter 18. 비동기 프로그래밍 (Async/Await)

FastAPI의 대성공 이후, 파이썬 백엔드 생태계는 빠르게 `asyncio` 기반으로 재편되었습니다. I/O 대기 시간(DB 쿼리, 외부 API 호출 등) 동안 스레드를 놀리지 않고 다른 작업을 처리할 수 있는 비동기(Asynchronous) 프로그래밍은 현대 웹 서버의 필수 요건입니다.

하지만 파이썬의 `asyncio`에는 치명적인 약점이 있습니다. 바로 **이벤트 루프(Event Loop)가 단일 스레드 위에서만 동작한다**는 것입니다. (최근 멀티스레드 지원을 늘려가고 있지만 근본적인 한계는 여전합니다.) 만약 비동기 함수 안에서 실수로 무거운 연산(CPU Bound)이나 블로킹(Blocking) I/O 함수를 호출하면, 전체 이벤트 루프가 멈춰버리고 서버의 모든 요청 처리가 중단됩니다.

Rust의 `async/await`은 파이썬과 겉모습은 매우 비슷하지만, 내부 구현과 생태계 구조는 완전히 다릅니다.

---

## 18.1 파이썬 `asyncio` vs 러스트 `Tokio`

파이썬은 언어 내장 표준 라이브러리로 `asyncio` 런타임을 제공합니다. 하지만 Rust 표준 라이브러리에는 비동기 코드를 정의하는 뼈대(`Future` 트레이트)만 존재할 뿐, 그것을 실제로 실행하는 엔진(런타임)은 없습니다.

대신 Rust 커뮤니티는 **`Tokio`**라는 압도적인 사실상의 표준(De facto standard) 비동기 런타임을 만들어 냈습니다.

* **The Python Way (`asyncio`)**
  ```python
  import asyncio
  import time

  async def fetch_data(id: int):
      print(f"Fetching {id}...")
      await asyncio.sleep(1) # I/O 시뮬레이션
      return f"Data {id}"

  async def main():
      # 여러 작업을 동시에 실행
      results = await asyncio.gather(fetch_data(1), fetch_data(2))
      print(results)

  # 파이썬 내장 이벤트 루프 실행
  asyncio.run(main())
  ```

* **The Rust Implementation (`Tokio`)**
  ```rust
  // tokio 매크로를 사용하여 메인 함수를 비동기 런타임 진입점으로 만듭니다.
  #[tokio::main]
  async fn main() {
      // 여러 작업을 동시에 실행(join)
      let (res1, res2) = tokio::join!(
          fetch_data(1),
          fetch_data(2)
      );
      println!("Results: {}, {}", res1, res2);
  }

  async fn fetch_data(id: i32) -> String {
      println!("Fetching {}...", id);
      // Tokio가 제공하는 비동기 sleep 사용
      tokio::time::sleep(std::time::Duration::from_secs(1)).await;
      format!("Data {}", id)
  }
  ```

---

## 18.2 런타임의 차이: Multi-thread Event Loop

파이썬의 `asyncio`는 단일 스레드를 쪼개 씁니다. 하지만 Rust의 `Tokio` 런타임은 기본적으로 **멀티스레드 스케줄러(Multi-thread Work-Stealing Scheduler)**를 작동시킵니다. 

서버에 CPU 코어가 8개 있다면, Tokio는 백그라운드에 8개의 워커 스레드를 띄웁니다. 수천 개의 `async` 작업(Task)들이 이 8개의 스레드에 나뉘어 배정됩니다. 만약 한 스레드에 일이 너무 몰리면 다른 여유로운 스레드가 그 일을 훔쳐 와서(Work-stealing) 대신 처리합니다.

이는 파이썬으로 치면 `asyncio`와 `multiprocessing`을 동시에, 그것도 완벽하게 안전하고 오버헤드 없이 사용하는 것과 같습니다. 이 강력한 아키텍처 덕분에 Rust 웹 프레임워크(Axum, Actix-web 등)가 벤치마크 테스트에서 전 세계 최상위권을 놓치지 않는 것입니다.

---

## 18.3 상태 머신 변환 (State Machine Translation)

파이썬의 `await`은 내부적으로 제너레이터(Generator)를 통해 구현됩니다. 그렇다면 Rust 컴파일러는 `async` 함수를 어떻게 처리할까요?

```rust
async fn do_something() {
    let x = 1;
    await_step_one().await;
    let y = x + 1;
    await_step_two(y).await;
}
```

Rust 컴파일러는 이 코드를 보면 뒤에서 놀라운 마법을 부립니다. 함수 코드를 쪼개어, `await`을 만날 때마다 멈추고 재개할 수 있는 거대한 **상태 머신(State Machine) 구조체**를 자동으로 생성합니다.

```rust
// (컴파일러가 내부적으로 만들어내는 코드의 개념적 모습)
enum DoSomethingStateMachine {
    Start,
    WaitForStepOne { x: i32 },
    WaitForStepTwo { y: i32 },
    Done,
}
```
변수 `x`와 `y`는 스택(Stack) 메모리가 아니라 이 상태 머신 구조체 내부에 안전하게 저장됩니다. 이 구조체는 메모리 크기가 정확히 고정되어 있으므로, 힙(Heap) 할당(Garbage) 없이 초고속으로 컨텍스트 스위칭을 수행할 수 있습니다.

---

## 18.4 게으른 비동기 (Lazy by default)

파이썬 개발자가 가장 흔히 하는 실수 중 하나를 살펴봅시다.

* **The Python Way (Eager Evaluation)**
  ```python
  async def my_task():
      print("Task started!")

  task = my_task() # 함수를 호출하는 순간 이벤트 루프에 등록(Eager)되거나 경고가 뜸
  # await task
  ```

파이썬에서는 `async` 함수를 호출하기만 해도 코드가 돌기 시작하거나 상태가 변경됩니다. 하지만 Rust에서 비동기 함수는 철저하게 **게으릅니다(Lazy)**.

* **The Rust Implementation (Lazy Evaluation)**
  ```rust
  async fn my_task() {
      println!("Task started!");
  }

  #[tokio::main]
  async fn main() {
      let task = my_task(); // 아무 일도 일어나지 않음. (Future 객체만 생성됨)
      
      // ... 1시간 대기 ...
      
      task.await; // .await을 호출하는 이 순간! 비로소 코드가 실행되기 시작함.
  }
  ```

Rust의 `Future`(파이썬의 `Coroutine` 객체에 해당)는 `.await`으로 실행을 요구받기 전까지는 그저 데이터를 들고 있는 무해한 구조체 덩어리에 불과합니다. 이 "게으른 특성" 덕분에 수만 개의 작업을 미리 만들어두고 타이밍을 완벽히 통제할 수 있습니다.

지금까지 Rust의 동시성과 비동기 프로그래밍 모델에 대해 파악했습니다. GIL의 사슬을 끊고 진정한 멀티 코어 성능을 만끽할 준비가 되셨나요? 
다음 파트(Part 7)에서는 파이썬의 데코레이터를 완벽하게 대체하고 코드를 생성해 내는 **매크로(Macro)**의 마법과 흑마술(Unsafe)에 대해 깊이 파헤쳐 보겠습니다.
