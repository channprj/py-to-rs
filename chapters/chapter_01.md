# Chapter 1. 왜 Python 개발자가 Rust를 배워야 하는가?

10년 이상 Python으로 밥을 먹고살아온 백엔드 개발자, 데이터 엔지니어, 인프라 개발자라면 누구나 한 번쯤 "성능의 벽"에 부딪혀 본 경험이 있을 것입니다. API 서버의 응답 속도 개선을 위해 `asyncio`나 `FastAPI`를 도입해 보고, CPU 집약적인 작업을 처리하기 위해 `multiprocessing`을 띄워 보며, 메모리 릭(Memory Leak)을 잡기 위해 `gc` 모듈과 `objgraph`를 파헤쳐 보았을 것입니다.

Python은 '개발자의 생산성(Developer Velocity)'을 최우선으로 설계된 언어입니다. 인간의 뇌가 코드를 짜는 속도는 놀랍도록 단축시켜 주지만, 작성된 코드가 실리콘(CPU) 위에서 돌아갈 때는 상당한 오버헤드를 동반합니다. 

이 챕터에서는 10년 차 Python 개발자의 시각에서 Python의 근본적인 한계(GIL, GC Pause, Duck Typing)를 내부 구조 관점에서 되짚어보고, 왜 수많은 언어 중 하필 **Rust**가 그 완벽한 해독제로 떠오르고 있는지 비교해 봅니다.

---

## 1.1 Python의 구조적 한계와 런타임 오버헤드

### 1.1.1 모든 것은 `PyObject`이다 (메모리 오버헤드)
Python에서 단순한 정수 `1`을 선언할 때 메모리에서는 어떤 일이 일어날까요? C나 Rust에서는 4바이트(또는 8바이트)의 연속된 메모리 공간만 있으면 충분합니다. 하지만 CPython 구현체에서 모든 데이터는 `PyObject`라는 C 구조체의 포인터로 관리됩니다.

```c
// CPython 소스 코드 내부의 정수형 구조체 (간소화)
struct _longobject {
    PyObject_VAR_HEAD // 참조 횟수(Reference Count), 타입 정보 등을 담은 헤더
    digit ob_digit[1]; // 실제 데이터
};
```

* **The Python Way**: 
  ```python
  # Python에서는 단순 배열을 만들어도 메모리 오버헤드가 큽니다.
  # 이 리스트는 실제 정수들이 연속으로 저장된 것이 아니라, 
  # 각기 다른 Heap 위치에 흩어져 있는 PyObject들에 대한 '포인터' 배열입니다.
  numbers = [1, 2, 3, 4, 5]
  ```
Python의 `List`는 연속된 객체 배열이 아닙니다. 포인터들의 배열입니다. 따라서 리스트를 순회하며 계산을 수행할 때, CPU 캐시 히트율(Cache Hit Rate)이 처참하게 떨어집니다. CPU가 메모리를 읽을 때 주변 데이터도 한꺼번에 가져오는데(Spatial Locality), 포인터가 가리키는 실제 데이터는 메모리 여기저기 파편화되어 있기 때문입니다.

* **The Rust Mental Model**:
  Rust에서 `Vec<i32>`를 선언하면 Heap 메모리상에 **정확히 4바이트짜리 정수들이 틈 없이 연속적으로(contiguous) 배치**됩니다. 포인터 추적(Dereferencing)이 필요 없으므로 CPU L1/L2 캐시를 극한으로 활용합니다.

  ```rust
  // Rust Implementation
  fn main() {
      // 힙(Heap)에 4바이트(i32) * 5개 = 총 20바이트가 연속적으로 할당됨.
      // 포인터 배열 오버헤드, 객체 헤더(PyObject_HEAD) 오버헤드 제로.
      let numbers: Vec<i32> = vec![1, 2, 3, 4, 5];
      
      // CPU 캐시를 완벽히 활용하는 초고속 순회
      let sum: i32 = numbers.iter().sum();
      println!("Sum: {}", sum);
  }
  ```

### 1.1.2 GIL (Global Interpreter Lock)의 저주
"Python은 멀티스레딩이 안 된다"는 말은 반만 맞고 반은 틀린 말입니다. I/O 바운드 작업(`requests.get()` 등)에서는 멀티스레딩이 유효하지만, CPU 바운드 연산(for 루프로 큰 행렬을 계산하는 등)에서는 GIL 때문에 여러 스레드를 띄워도 한 번에 단 하나의 스레드만 Python 바이트코드를 실행할 수 있습니다.

**왜 GIL을 만들었을까요?** 앞서 말한 `PyObject`의 '참조 횟수(Reference Count)'를 스레드 안전(Thread-safe)하게 관리하기 위해서입니다. 수만 개의 객체마다 일일이 락(Lock)을 걸면 너무 느려지니, 아예 "인터프리터 전체에 커다란 자물쇠 하나"를 걸어버린 것입니다.

* **The Python Way**: CPU 바운드 작업을 병렬화하려면 울며 겨자 먹기로 `multiprocessing`을 써야 합니다.
  ```python
  import multiprocessing

  def heavy_computation(data_chunk):
      # 복잡한 계산 수행
      return sum(x * x for x in data_chunk)

  if __name__ == '__main__':
      data = [list(range(1000000)) for _ in range(4)]
      # 스레드 대신 아예 별도의 Python 프로세스(OS 레벨)를 4개 복제해서 띄움 (메모리 4배 낭비)
      with multiprocessing.Pool(4) as p:
          results = p.map(heavy_computation, data)
  ```
  별도의 프로세스를 띄우면 객체 직렬화/역직렬화(Pickle) 비용과 막대한 메모리 복제 비용(IPC Overhead)이 발생합니다. 

### 1.1.3 Duck Typing과 런타임 검사의 늪
Python의 강력함은 어디서 올까요? 바로 객체의 타입을 런타임에 결정짓는 Duck Typing입니다.

```python
def add(a, b):
    return a + b  # a와 b가 int인지, str인지, list인지 실행해보기 전엔 모름
```
파이썬 인터프리터는 이 코드를 실행할 때마다 `a`의 타입이 무엇인지, `a` 객체 안에 `__add__` 매직 메서드가 정의되어 있는지 딕셔너리 조회를 통해 검색(Dynamic Dispatch)해야 합니다. 이 런타임 타입 검사와 메서드 룩업 과정이 프로그램 실행 시간의 상당 부분을 갉아먹습니다. 아무리 Type Hint(`a: int, b: int`)를 달아도, 이는 린터(Mypy)용일 뿐 런타임 최적화에는 전혀 기여하지 못합니다.

---

## 1.2 Rust의 해독제: Zero-cost Abstraction과 Fearless Concurrency

Rust는 위의 세 가지 문제를 언어의 설계 철학 단계에서 완벽하게 해결합니다.

### 1.2.1 컴파일러에게 책임을 떠넘기다 (Static Typing & Monomorphization)
Rust는 동적 디스패치의 비용을 **단일형화(Monomorphization)**라는 컴파일 타임 기법으로 없애버립니다. 

* **The Rust Implementation**:
  ```rust
  // 제네릭(Generic)을 사용해 다형성을 구현한 Rust 함수
  // T는 Add 트레이트를 구현해야 하고(std::ops::Add), Output 타입이 T와 같아야 한다는 제약(Bound)
  fn add<T>(a: T, b: T) -> T 
  where 
      T: std::ops::Add<Output = T> 
  {
      a + b
  }

  fn main() {
      let int_result = add(5, 10);
      let float_result = add(3.14, 2.71);
  }
  ```

**Under the Hood**: 위 Rust 코드가 컴파일될 때, 컴파일러는 `add`라는 단 하나의 함수를 만들지 않습니다. `add`가 정수(`i32`)로 호출된 곳과 실수(`f64`)로 호출된 곳을 분석하여 **어셈블리 레벨에서 두 개의 서로 다른 함수(`add_i32`, `add_f64`)를 하드코딩해서 복제 생성**합니다. 
런타임에는 타입을 검사하거나 메서드를 조회하는 시간이 단 1나노초도 들지 않습니다. 그저 컴파일러가 만들어둔 최적화된 기계어를 실행할 뿐입니다. 이것이 바로 C++와 Rust가 자랑하는 **Zero-cost Abstraction(추상화 비용 제로)**입니다.

### 1.2.2 GIL 파괴자: 소유권(Ownership)과 Fearless Concurrency
Rust에는 GIL이 없습니다. 그렇다고 `PyObject`처럼 객체마다 레퍼런스 카운트를 세며 뮤텍스(Mutex)를 남발하지도 않습니다. 

**Garbage Collector 없이 어떻게 메모리를 해제하는가?**
Rust는 **소유권(Ownership)**이라는 독특한 규칙을 도입했습니다. "모든 메모리 할당은 단 하나의 '주인(Owner)' 변수만 갖는다. 주인이 스코프(Scope)를 벗어나면 메모리는 즉각 해제된다."

이 단순한 규칙 덕분에 런타임에 뒤에서 쓰레기를 줍고 다니는 GC 스레드(GC Pause를 유발하는 주범)가 필요 없어집니다.

* **예제: Python C-extension vs Rust 멀티스레딩 비교**
  Python에서 GIL을 피하기 위해 C++로 확장(Extension)을 작성할 때 가장 무서운 것은 바로 Data Race(여러 스레드가 동시에 같은 메모리를 수정하려다 데이터가 꼬이거나 C-Crash가 나는 현상)입니다. 

  하지만 Rust에서는 여러 스레드가 동시에 안전하지 않게 메모리에 접근하는 코드 자체를 **컴파일러가 거부**해버립니다.

  ```rust
  use std::thread;
  use std::sync::{Arc, Mutex};

  fn main() {
      // Arc: Atomic Reference Counted (스레드 간 안전한 참조 카운트)
      // Mutex: 상호 배제 잠금 (데이터 접근을 1개 스레드로 제한)
      let counter = Arc::new(Mutex::new(0));
      let mut handles = vec![];

      for _ in 0..4 {
          let counter_clone = Arc::clone(&counter);
          // Python의 multiprocessing과 달리, 진정한 OS 스레드 4개가 생성됨!
          let handle = thread::spawn(move || {
              // Lock을 걸지 않으면 데이터에 물리적으로 접근할 수 없도록 강제됨
              let mut num = counter_clone.lock().unwrap();
              *num += 1; // 여기서만 값 수정 가능
          }); // Lock은 스코프를 벗어나면 RAII 패턴에 의해 자동 해제됨.
          handles.push(handle);
      }

      for h in handles {
          h.join().unwrap(); // 모든 스레드 종료 대기
      }

      println!("Result: {}", *counter.lock().unwrap());
  }
  ```

이것이 Rust의 **Fearless Concurrency(두려움 없는 동시성)**입니다. 컴파일이 성공했다면, 데드락(Deadlock, 무한 대기)은 발생할지언정, 메모리가 꼬여서 프로그램이 갑자기 코어 덤프(Core Dump)를 뱉으며 죽는 `Segfault`나 `Data Race`는 절대 발생하지 않음을 언어가 수학적으로 증명해 준 것입니다.

---

## 1.3 요약: Python 개발자에게 Rust란?

1. **런타임 에러에서 컴파일 타임 에러로**: 배포 후 새벽 3시에 발생하던 `AttributeError`나 `TypeError`를, 내 PC에서 코드를 저장하는 순간(Compile Time) 발생하도록 앞당깁니다.
2. **"제발 메모리 좀 덜 먹어라"의 해방**: 서버를 띄워두고 점진적으로 램이 치솟는 OOM(Out of Memory) 현상과 싸우느라 시간을 보냈다면, 소유권 기반의 Rust는 마음의 평화를 줄 것입니다.
3. **완벽한 접착제(Glue) 언어**: Python 코드를 전부 Rust로 바꿀 필요는 없습니다. Python의 유연함이 필요한 비즈니스 로직(Django, FastAPI 등)은 그대로 두고, 병목이 발생하는 수치 연산, 텍스트 파싱, 암호화 등의 코어 모듈만 Rust로 포팅(PyO3)하여 Python 모듈처럼 `import`해서 쓰는 하이브리드 전략이야말로 현대 백엔드 아키텍처의 트렌드입니다. (실례: `pydantic-core`, `polars`, `cryptography` 등)

다음 챕터에서는 의존성 관리 지옥(`pip`, `requirements.txt`, `poetry`, `virtualenv`)에서 해방시켜 줄 완벽한 툴링, **Cargo**에 대해 파헤쳐 봅니다.
