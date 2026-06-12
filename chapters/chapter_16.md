# Chapter 16. 멀티스레딩: `threading` 모듈의 한계 돌파

10년 차 파이썬 개발자라면 'GIL(Global Interpreter Lock)'이라는 단어만 들어도 진저리가 날 것입니다. 멀티 코어 CPU 시대가 열린 지 십수 년이 지났지만, 파이썬은 여전히 한 번에 단 하나의 스레드만 파이썬 바이트코드를 실행할 수 있도록 인터프리터 전체에 자물쇠(Lock)를 걸어둡니다.

파이썬에서 CPU 바운드 작업을 병렬 처리하려면 `multiprocessing`을 써서 메모리를 통째로 복제한 무거운 프로세스들을 띄워야만 했습니다. (I/O 바운드 작업은 예외입니다.)

Rust는 GIL이 존재하지 않습니다. 모든 스레드는 OS 레벨의 진짜 스레드이며, CPU 코어 개수만큼 동시에 기계어를 실행합니다. 게다가 이 스레드들은 컴파일 타임에 검증된 완벽한 안전성을 보장받습니다.

---

## 16.1 진짜 하드웨어 스레드 띄우기

* **The Python Way (Multithreading)**
  ```python
  import threading
  import time

  def worker(num):
      print(f"Worker {num} is running")
      time.sleep(1)

  threads = []
  for i in range(5):
      t = threading.Thread(target=worker, args=(i,))
      threads.push(t)
      t.start()

  for t in threads:
      t.join()
  # I/O 대기(sleep) 중에는 GIL이 풀리지만, 계산 작업이었다면 전혀 빨라지지 않음.
  ```

* **The Rust Implementation (`std::thread`)**
  Rust에서는 `thread::spawn`에 **클로저(Closure)**를 넘겨 새로운 스레드를 실행합니다.

  ```rust
  use std::thread;
  use std::time::Duration;

  fn main() {
      let mut handles = vec![];

      for i in 0..5 {
          // move 키워드: 환경에 있는 변수(i)의 소유권을 스레드 안으로 이동시킴
          let handle = thread::spawn(move || {
              println!("Worker {} is running", i);
              thread::sleep(Duration::from_secs(1));
          });
          handles.push(handle);
      }

      // 메인 스레드가 종료되기 전에 모든 자식 스레드를 기다립니다.
      for handle in handles {
          handle.join().unwrap();
      }
  }
  ```

### `move` 키워드의 마법
위 Rust 코드에서 `move`라는 키워드가 왜 들어갔을까요? 
만약 메인 스레드의 변수를 자식 스레드가 참조(`&`)하게 두었다면, 메인 스레드가 먼저 종료되어 변수가 해제(Drop)되었을 때 자식 스레드는 죽은 메모리를 가리키게 됩니다(Dangling Pointer). Rust 컴파일러는 이를 막기 위해 "스레드로 데이터를 넘길 때는 참조하지 말고 아예 소유권을 넘겨라(`move`)"라고 강제합니다.

---

## 16.2 공유 메모리 대신 메시지 패싱 (Channels)

멀티스레드 환경에서 가장 안전하게 데이터를 주고받는 방식은 한 스레드에서 다른 스레드로 값을 "보내는(Send)" 것입니다. 파이썬에서는 주로 `queue.Queue`를 씁니다. Rust에서는 이를 **채널(Channel)**이라고 부릅니다. 

러스트 표준 라이브러리는 `mpsc` (Multiple Producer, Single Consumer) 채널을 제공합니다. 송신자(Producer)는 여러 개일 수 있지만, 수신자(Consumer)는 하나입니다.

* **The Python Way (`queue.Queue`)**
  ```python
  import threading
  import queue

  q = queue.Queue()

  def producer(q):
      q.put("Data from worker")

  t = threading.Thread(target=producer, args=(q,))
  t.start()

  # 큐에서 데이터를 꺼냄 (안전함)
  print(q.get()) 
  ```

* **The Rust Implementation (`std::sync::mpsc`)**
  ```rust
  use std::sync::mpsc;
  use std::thread;

  fn main() {
      // tx: 송신자(Transmitter), rx: 수신자(Receiver)
      let (tx, rx) = mpsc::channel();

      thread::spawn(move || {
          let val = String::from("Data from worker");
          
          // tx를 통해 값을 보냅니다. 
          // 이 순간 val의 "소유권"이 수신자 쪽으로 완전히 넘어갑니다(Move).
          tx.send(val).unwrap();

          // println!("Sent: {}", val); // 🚨 컴파일 에러! 소유권 잃은 변수 사용 불가
      });

      // 메인 스레드에서 값을 받습니다. (블로킹 대기)
      let received = rx.recv().unwrap();
      println!("Got: {}", received);
  }
  ```

### 소유권이 만드는 완벽한 스레드 통신
파이썬의 `Queue`에 리스트를 넣은 뒤, 넣은 쪽에서 계속 그 리스트를 조작하면 받는 쪽에서는 엉망이 된 데이터를 받게 됩니다 (Race Condition). 

하지만 Rust에서는 채널을 통해 `tx.send(val)`을 호출하는 순간, `val`의 소유권 자체가 다른 스레드로 날아갑니다. 송신자 스레드에서 다시 `val`을 읽거나 수정하려 들면 **컴파일 자체가 거부**됩니다. 

> *"공유 메모리로 통신하지 마라. 통신함으로써 메모리를 공유하라."* (Go 언어의 유명한 격언이자 Rust 채널의 철학)

그럼에도 불구하고 어떤 데이터는 반드시 여러 스레드가 "동시에" 읽고 써야 할 때가 있습니다. 글로벌 캐시(Cache)나 조회수 카운터 같은 것들이죠. 다음 챕터에서는 Rust가 이 위험천만한 공유 상태(Shared State)를 어떻게 컴파일 타임에 안전하게 통제하는지 알아보겠습니다.
