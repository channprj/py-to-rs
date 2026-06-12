# Chapter 15. 이터레이터와 제너레이터(Generator)

파이썬 개발자들이 데이터를 처리할 때 가장 사랑하는 문법은 리스트 컴프리헨션(List Comprehension)과 제너레이터(Generator)입니다. 무거운 배열을 메모리에 한 번에 다 올리지 않고, `yield` 키워드를 사용해 필요할 때마다 값을 하나씩 뽑아 쓰는(Lazy Evaluation) 방식은 메모리 최적화의 핵심입니다.

Rust는 파이썬의 이터레이션(Iteration) 철학을 극단적으로 최적화하여 구현했습니다. 파이썬의 제너레이터처럼 지연 평가(Lazy Evaluation)를 수행할 뿐만 아니라, 어셈블리 레벨에서 명시적인 `for` 루프(Loop)보다도 이터레이터 체이닝이 더 빠르거나 같은 속도를 내도록 설계했습니다. 이를 **Zero-cost Abstraction**이라고 부릅니다.

---

## 15.1 파이썬 `__iter__`, `__next__`의 대체: `Iterator` Trait

파이썬에서는 `__iter__`와 `__next__` 매직 메서드를 구현하면 어떤 객체든 순회할 수 있습니다. Rust는 이를 `Iterator` 트레이트로 구현합니다.

* **The Python Way (Generator)**
  ```python
  def count_up_to(max_val):
      count = 1
      while count <= max_val:
          yield count
          count += 1

  for num in count_up_to(5):
      print(num)
  ```

* **The Rust Implementation (`Iterator` Trait)**
  Rust에는 `yield` 키워드가 내장되어 있지 않습니다 (비동기 코루틴 제외). 대신 상태를 저장하는 구조체를 만들고 `Iterator` 트레이트의 `next()` 메서드를 구현하여 제너레이터처럼 동작하게 만듭니다.

  ```rust
  struct Counter {
      count: u32,
      max_val: u32,
  }

  impl Counter {
      fn new(max_val: u32) -> Counter {
          Counter { count: 0, max_val }
      }
  }

  // 파이썬의 __iter__, __next__ 를 구현하는 것과 완벽히 동일합니다.
  impl Iterator for Counter {
      type Item = u32;

      fn next(&mut self) -> Option<Self::Item> {
          self.count += 1;
          if self.count <= self.max_val {
              Some(self.count) // 값을 반환 (yield)
          } else {
              None // 반복 종료 (StopIteration 예외 발생에 해당)
          }
      }
  }

  fn main() {
      // 1부터 5까지 지연 평가(Lazy)로 생성하며 출력합니다.
      for num in Counter::new(5) {
          println!("{}", num);
      }
  }
  ```

---

## 15.2 이터레이터 체이닝 (Map, Filter, Fold)

파이썬의 리스트 컴프리헨션이나 내장 `map()`, `filter()` 함수는 코드를 간결하게 만들어 줍니다. Rust의 이터레이터도 수십 가지의 강력한 어댑터(Adapter) 메서드를 제공합니다.

* **The Python Way**
  ```python
  numbers = [1, 2, 3, 4, 5]
  
  # 짝수만 골라서 제곱한 뒤 리스트로 만들기
  squared_evens = [x * x for x in numbers if x % 2 == 0]
  # 또는
  # squared_evens = list(map(lambda x: x*x, filter(lambda x: x%2 == 0, numbers)))
  ```

* **The Rust Implementation (Iterator Chaining & Closure)**
  Rust에서는 클로저(Closure, 파이썬의 `lambda` 함수)를 사용하여 이터레이터를 체이닝합니다.

  ```rust
  fn main() {
      let numbers = vec![1, 2, 3, 4, 5];

      let squared_evens: Vec<i32> = numbers
          .into_iter()                  // 1. 이터레이터 생성 (소유권 이동)
          .filter(|x| x % 2 == 0)       // 2. 짝수만 필터링 (클로저 사용)
          .map(|x| x * x)               // 3. 제곱 연산
          .collect();                   // 4. 결과를 다시 Vec 컬렉션으로 수집

      println!("{:?}", squared_evens);
  }
  ```

### 클로저(Closure) 문법
러스트의 클로저는 파이썬의 `lambda`와 거의 비슷합니다. 파이프(`|`) 기호 사이에 매개변수를 넣습니다.
- 파이썬: `lambda x: x * x`
- 러스트: `|x| x * x`

---

## 15.3 왜 Rust의 이터레이터는 빠른가? (Under the Hood)

파이썬에서 리스트 컴프리헨션을 쓰는 이유는 단순한 `for` 루프보다 인터프리터 수준에서 최적화가 잘 되어 있기 때문입니다. 그러나 여전히 런타임 오버헤드가 발생합니다.

Rust에서 `for` 루프를 쓰지 않고 `map`, `filter`, `fold`를 체이닝하는 것은 성능에 어떤 영향을 미칠까요? 놀랍게도 **직접 작성한 while 문보다 더 빠를 때가 많습니다.**

1. **루프 언롤링(Loop Unrolling)**: Rust 컴파일러(LLVM)는 이터레이터 체이닝 코드를 분석하여, 반복문을 돌지 않고도 순차적으로 실행되도록 어셈블리 코드를 복제(Unroll)해 버립니다.
2. **경계 검사(Bounds Checking) 생략**: 일반적인 `for i in 0..len` 루프를 돌며 `numbers[i]`로 접근하면, Rust는 배열 범위를 벗어나지 않는지 매번 검사(Bounds Check)합니다. 하지만 `Iterator`를 사용하면 컴파일러가 배열의 끝을 알고 있으므로 런타임 경계 검사 코드를 어셈블리에서 아예 삭제해 버립니다.

이것이 Rust 개발자들이 C++ 개발자들조차 부러워하는 **Zero-cost Abstraction**의 진수입니다. 코드는 함수형 프로그래밍처럼 고차원적이고 선언적(Declarative)으로 작성하지만, 컴파일된 기계어는 가장 저열하고 원시적인 C 언어의 메모리 접근 속도와 동일하거나 더 빠릅니다.

지금까지 Rust의 데이터 다루기(에러, 컬렉션, 이터레이터)를 완벽히 마스터했습니다. 
이제 이 시리즈의 하이라이트이자 파이썬의 가장 큰 한계인 GIL(Global Interpreter Lock)을 박살 내고 진정한 멀티 코어의 힘을 깨우는 **Part 6: 동시성(Concurrency & Async)**으로 넘어갑니다.
