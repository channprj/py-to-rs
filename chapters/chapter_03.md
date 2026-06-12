# Chapter 3. 문법의 이면: 표현식(Expression)과 타입 시스템

파이썬 개발자가 Rust 코드를 처음 마주하면, 세미콜론(`;`)이 여기저기 붙어있는 모습 때문에 C나 Java의 향수를 (혹은 거부감을) 느낄 수 있습니다. 그러나 겉보기에 비슷한 문법 구조 아래 깔린 **철학은 완전히 다릅니다.** 

파이썬은 문(Statement) 중심의 언어이며, 제어 흐름(Control Flow)을 바꾸기 위해 문장을 나열합니다. 반면 Rust는 함수형 언어의 영향을 강하게 받은 **표현식(Expression) 기반 언어**입니다. 거의 모든 것이 값을 반환합니다.

이번 챕터에서는 파이썬의 구문과 Rust의 표현식이 어떻게 다르게 동작하는지, 그리고 동적 타입과 정적 타입의 메모리 레이아웃 차이를 깊이 있게 비교합니다.

---

## 3.1 Statement(문) vs Expression(표현식)

*   **문(Statement)**: 어떤 동작을 수행하며, 값을 반환하지 않습니다. (예: `x = 5`, `def foo():`)
*   **표현식(Expression)**: 평가(Evaluate)되어 결과 **값을 반환**합니다. (예: `5 + 5`, `foo()`)

### 파이썬의 한계: 문(Statement) 위주의 제어 흐름
파이썬의 `if`문이나 `for`문은 값을 반환하지 않는 '문(Statement)'입니다. 따라서 조건에 따라 변수에 값을 할당하려면 밖에서 변수를 미리 선언하거나 삼항 연산자를 억지로 써야 합니다.

* **The Python Way**
  ```python
  condition = True
  
  # 방법 1: 블록을 이용한 부작용(Side-effect) 할당
  number = None
  if condition:
      number = 5
  else:
      number = 10
      
  # 방법 2: 파이썬 특유의 삼항 연산자 (식이 길어지면 가독성이 떨어짐)
  number = 5 if condition else 10
  ```

### 러스트의 접근: 블록 자체를 표현식으로
Rust에서는 중괄호 `{}` 로 묶인 새로운 스코프(블록) 자체가 표현식입니다. 블록의 **마지막 줄에 세미콜론(`;`)을 생략하면**, 그 값이 밖으로 튀어나와(return) 반환됩니다.

* **The Rust Implementation**
  ```rust
  fn main() {
      let condition = true;

      // if 블록 전체가 표현식이므로 변수에 바로 바인딩됩니다.
      let number = if condition {
          5 // 세미콜론 없음 -> 반환
      } else {
          10 // 세미콜론 없음 -> 반환
      };

      // 심지어 일반 블록도 값을 반환할 수 있습니다.
      let x = {
          let y = 3;
          let z = 4;
          y + z // 7을 반환
      };
      
      println!("number: {}, x: {}", number, x);
  }
  ```
이렇게 표현식 기반으로 코드를 작성하면 불필요한 상태 변경(Mutation) 변수를 줄이고, 모든 변수를 불변(`Immutable`)으로 유지하기 쉬워집니다.

---

## 3.2 `return` 키워드의 생략
파이썬에서는 함수가 값을 반환할 때 반드시 `return` 키워드를 써야 합니다. 생략하면 `None`이 반환됩니다.
러스트에서도 명시적인 `return` 키워드가 존재하지만 (주로 조기 종료 Early Return을 위해 사용), 함수의 마지막 결과물은 **표현식의 꼬리(세미콜론 생략)**를 통해 자연스럽게 반환하는 것이 관례(Idiomatic Rust)입니다.

```rust
// 파이썬: 
// def add_and_multiply(a: int, b: int, c: int) -> int:
//     sum_val = a + b
//     return sum_val * c

fn add_and_multiply(a: i32, b: i32, c: i32) -> i32 {
    let sum_val = a + b;
    // return sum_val * c; // 이렇게 써도 되지만
    sum_val * c // 세미콜론을 생략하여 값 자체를 반환 (Idiomatic)
}
```

---

## 3.3 타입 시스템과 메모리 레이아웃의 차이

파이썬 개발자가 가장 충격받는 부분 중 하나가 숫자형 데이터 타입의 세분화입니다. 파이썬은 정수를 쓸 때 그저 `int`, 실수를 쓸 때 `float`라고만 부릅니다. 

하지만 메모리 관점에서 파이썬의 변수는 끔찍하게 뚱뚱합니다. 파이썬의 변수는 무조건 **Heap 공간에 동적 할당**되며, 다음과 같은 헤더를 달고 다닙니다.

* **Python의 `int` (CPython 기준 최소 28 Bytes 오버헤드)**
  * Reference Count (8 bytes)
  * Type Object Pointer (8 bytes)
  * Size 정보 (8 bytes)
  * 실제 정수 값 (4~8 bytes)

* **Rust의 타입 시스템**
  Rust는 정밀한 기계어 통제를 위해 메모리 크기와 부호를 명시합니다. 
  * 부호 있음: `i8`, `i16`, `i32`, `i64`, `i128`, `isize`
  * 부호 없음: `u8`, `u16`, `u32`, `u64`, `u128`, `usize` (메모리 주소나 인덱스용)
  * 실수형: `f32`, `f64`

  ```rust
  let age: u8 = 30; // 1바이트(0~255)만 소모. Stack 메모리에 직접 저장!
  let balance: i64 = -10000; // 8바이트.
  ```

이러한 원시 타입(Primitive Type)들은 힙(Heap) 포인터를 만들지도 않고, 객체 헤더도 없으며, 그저 **스택(Stack) 메모리**에 딱 지정된 바이트만큼만 자리를 차지합니다.

### 튜플(Tuple)의 메모리 배열
파이썬의 튜플은 리스트와 마찬가지로 여러 객체의 '포인터'를 담고 있는 불변 배열입니다.

```python
point = (10, 20.5) 
# 메모리 구조:
# Tuple Pointer -> [ Ptr1, Ptr2 ]
# Ptr1 -> Heap 안의 int 객체(10)
# Ptr2 -> Heap 안의 float 객체(20.5)
```

반면 Rust의 튜플은 스택 메모리에 이웃하게(contiguous) 압축됩니다.

```rust
let point: (i32, f64) = (10, 20.5);
// 메모리 구조 (스택):
// [ 4 Bytes (10) ][ 4 Bytes (패딩) ][ 8 Bytes (20.5) ]
// 총 16 Bytes짜리 단일 덩어리로 캐시 친화적(Cache-friendly)으로 존재합니다.
```

데이터를 포인터로 쫓아다니지 않고(No Dereferencing), 메모리에 옹기종기 모아둔 채 CPU에 밀어 넣기. 이것이 바로 파이썬 코드로는 도달할 수 없는 Rust 속도의 비밀 중 하나입니다.

다음 챕터에서는 스택(Stack)과 힙(Heap)의 차이를 넘어, 러스트의 가장 큰 진입 장벽이자 핵심 혁신인 **소유권(Ownership)** 모델에 대해 파헤쳐 보겠습니다.
