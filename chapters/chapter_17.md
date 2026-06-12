# Chapter 17. 상태 공유와 동기화 (Shared State)

앞서 메시지 패싱(Channel)이 스레드 간 통신의 가장 훌륭한 방법이라고 소개했습니다. 하지만 여러 스레드가 동시에 하나의 거대한 딕셔너리(캐시)나 카운터를 업데이트해야 할 때는 어쩔 수 없이 **공유 상태(Shared State)**를 가져야 합니다.

파이썬에서는 변수 접근 전에 `threading.Lock()`을 걸어서 Data Race를 피합니다.

* **The Python Way (`threading.Lock`)**
  ```python
  import threading

  counter = 0
  lock = threading.Lock()

  def increment():
      global counter
      # 개발자가 '실수로' 락을 걸지 않고 counter += 1을 해버리면 
      # 런타임에 데이터가 조용히 꼬여버립니다.
      with lock: 
          counter += 1

  threads = [threading.Thread(target=increment) for _ in range(10)]
  for t in threads: t.start()
  for t in threads: t.join()
  ```

이 패턴의 가장 큰 문제는 **"개발자의 실수(Lock 누락)"를 아무도 막아주지 못한다**는 것입니다. 파이썬의 `Lock`은 그저 신호등일 뿐, 신호등을 무시하고 데이터를 건드리는 코드를 막을 강제력이 없습니다.

---

## 17.1 강제된 동기화: Rust의 `Mutex<T>`

Rust의 Mutex(Mutual Exclusion, 상호 배제)는 파이썬처럼 '따로 노는' 락이 아닙니다. **보호해야 할 데이터를 `Mutex`라는 껍질 안에 물리적으로 가둬버립니다.**

```rust
use std::sync::Mutex;

fn main() {
    // 0이라는 정수를 Mutex 안에 가둡니다.
    let m = Mutex::new(0);

    {
        // lock()을 호출하지 않으면 내부의 0에 접근조차 할 수 없습니다!
        let mut num = m.lock().unwrap();
        *num = 6;
    } // 스코프가 끝나면 num이 Drop되면서 락이 '자동으로' 풀립니다. (RAII 패턴)

    println!("m = {:?}", m);
}
```

이 구조 덕분에 Rust에서는 개발자가 "락을 거는 것을 깜빡해서" Data Race가 발생하는 일이 **컴파일 타임에 100% 방지**됩니다.

---

## 17.2 멀티스레드로의 확장: `Arc<T>`

위의 `Mutex`를 여러 스레드에 나누어 주려면 어떻게 해야 할까요? 앞서 배운 소유권 규칙에 따르면 하나의 변수는 동시에 여러 소유자를 가질 수 없습니다. 

파이썬 객체는 내부적으로 Reference Count를 가지므로 변수를 여기저기 넘겨도 문제가 없습니다. Rust에서도 동일하게 **참조 카운터(Reference Counted, Rc)** 스마트 포인터를 제공합니다. 멀티스레드 환경에서 안전하게 카운트를 올리고 내리기 위해 **Atomic(원자적)** 연산을 사용하는 것이 바로 **`Arc` (Atomic Reference Counted)**입니다.

* **The Rust Implementation (`Arc` + `Mutex`)**
  ```rust
  use std::sync::{Arc, Mutex};
  use std::thread;

  fn main() {
      // 1. 데이터를 Mutex로 가두고, 다시 그것을 Arc로 감쌉니다.
      // Arc는 소유권을 공유하게 해주고, Mutex는 동시 수정을 막아줍니다.
      let counter = Arc::new(Mutex::new(0));
      let mut handles = vec![];

      for _ in 0..10 {
          // 2. Arc의 복제본(포인터 복사, 카운트 1 증가)을 만듭니다.
          let counter_clone = Arc::clone(&counter);

          let handle = thread::spawn(move || {
              // 3. 락을 획득하고 데이터를 수정합니다.
              let mut num = counter_clone.lock().unwrap();
              *num += 1;
          });
          handles.push(handle);
      }

      for handle in handles {
          handle.join().unwrap();
      }

      println!("Final Result: {}", *counter.lock().unwrap()); // 10
  }
  ```

이 구조는 파이썬의 `multiprocessing.Manager().dict()`나 Redis를 이용한 IPC(프로세스 간 통신)와 비교할 수 없을 정도로 가볍고 빠릅니다. 진정한 OS 스레드들이 하나의 힙(Heap) 메모리 공간을 공유하면서, 완벽하게 안전한 동기화를 달성한 것입니다.

### `Send`와 `Sync` 마커 트레이트
Rust 컴파일러는 이 모든 걸 어떻게 검증할까요? 바로 마커(Marker) 트레이트 덕분입니다.
*   **`Send`**: 이 타입의 데이터를 다른 스레드로 전송(이동)해도 안전한가?
*   **`Sync`**: 이 타입의 데이터에 여러 스레드가 동시에 접근(`&T`)해도 안전한가?

우리가 일반적인 `Rc`(스레드 안전하지 않은 참조 카운터)를 생성해서 스레드로 넘기려 하면, 컴파일러는 `Rc`가 `Send` 트레이트를 구현하지 않았음을 확인하고 빌드를 막아버립니다. 개발자의 머리가 아닌, 타입 시스템이 멀티스레딩의 안전(Fearless Concurrency)을 담보하는 순간입니다.

다음 챕터에서는 멀티스레딩을 넘어 수만 개의 커넥션을 동시에 처리하는 현대 백엔드의 꽃, **비동기 프로그래밍(Async/Await)**이 파이썬 `asyncio`와 어떻게 다른지 파헤쳐 봅니다.
