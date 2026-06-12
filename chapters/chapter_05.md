# Chapter 5. 참조와 대여 (Borrowing & References)

앞선 장에서 보았듯, 함수에 객체를 넘길 때마다 소유권을 빼앗긴다면 정상적인 프로그래밍이 불가능합니다. 파이썬에서는 객체의 메모리 주소(Reference)를 함수에 넘기는 방식(Pass by object reference)으로 이 문제를 해결합니다.

Rust에서도 메모리 주소(포인터)를 넘길 수 있습니다. 이를 **참조(Reference)**라고 부르며, 참조를 만드는 행위를 **대여(Borrowing)**라고 합니다. 파이썬의 객체 전달과 비슷해 보이지만, 치명적인 Data Race를 막기 위한 엄격한 규칙이 존재합니다.

---

## 5.1 불변 참조 (Immutable Reference: `&T`)

가장 기본적인 대여 방식입니다. 앰퍼샌드(`&`) 기호를 사용하여 값의 소유권은 넘기지 않고 "읽기 권한"만 빌려줍니다.

* **The Python Way**
  ```python
  def calculate_length(s: list) -> int:
      # 리스트의 주소를 넘겨받아 길이만 읽음
      return len(s)

  my_list = [1, 2, 3]
  length = calculate_length(my_list)
  print(my_list) # my_list는 여전히 유효함
  ```

* **The Rust Implementation**
  ```rust
  // s는 Vec<i32>의 '참조(Reference)' 타입인 &Vec<i32>를 받습니다.
  fn calculate_length(s: &Vec<i32>) -> usize {
      s.len()
  } // 함수가 끝나도 s가 가리키는 원본은 버려지지(Drop) 않습니다. 소유권이 없기 때문입니다.

  fn main() {
      let my_vec = vec![1, 2, 3];

      // & 기호를 붙여서 값을 빌려(Borrow) 줍니다.
      let length = calculate_length(&my_vec); 

      // my_vec는 소유권을 잃지 않았으므로 계속 사용할 수 있습니다.
      println!("Length of {:?} is {}", my_vec, length);
  }
  ```

불변 참조(`&`)는 책을 여러 명이 동시에 읽는 것과 같습니다. **동시에 여러 개의 불변 참조를 만드는 것이 가능합니다.** 단, 값을 읽을 수만 있고 수정할 수는 없습니다.

---

## 5.2 가변 참조 (Mutable Reference: `&mut T`)

빌려온 값을 수정하고 싶다면 어떻게 해야 할까요? 파이썬에서는 리스트를 함수에 넘기면 그 함수 내부에서 리스트 원소를 마음대로 수정(`append()`)할 수 있습니다. 하지만 이는 원본 데이터가 언제 어디서 수정될지 모른다는 런타임 버그의 온상이 되기도 합니다.

Rust에서는 가변 참조를 명시적으로 요청해야 합니다. `&mut` 키워드를 사용합니다.

```rust
// 매개변수 s를 가변 참조(&mut)로 받습니다.
fn add_element(s: &mut Vec<i32>) {
    s.push(4); // 원본 데이터 수정 가능!
}

fn main() {
    // 1. 변수 자체가 가변(mut)이어야 합니다.
    let mut my_vec = vec![1, 2, 3];

    // 2. 가변 참조(&mut)로 빌려줍니다.
    add_element(&mut my_vec); 

    println!("{:?}", my_vec); // [1, 2, 3, 4]
}
```

---

## 5.3 파이썬엔 없는 규칙: Data Race의 완벽한 차단

Rust의 참조 메커니즘에서 파이썬 개발자들이 가장 고통스러워하는 부분이자, Rust의 가장 위대한 혁신이 바로 여기에 있습니다.

> **특정 데이터에 대해 '가변 참조(`&mut`)'를 만들면, 그 스코프 내에서는 다른 어떠한 참조(불변/가변 모두)도 동시에 만들 수 없습니다.**

이 규칙을 코드로 보겠습니다.

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    let r1 = &mut data; // 첫 번째 가변 참조 (쓰기 권한 획득)
    // let r2 = &mut data; // 🚨 컴파일 에러! 동시에 두 개의 쓰기 권한은 불가능!
    // let r3 = &data;     // 🚨 컴파일 에러! 누군가 쓰고 있는데 읽으려 하면 안 됨!

    r1.push(4); // r1을 다 쓰고 나면
    let r4 = &data; // 여기서부터는 다시 읽기(불변 참조)가 가능해집니다.
}
```

### 왜 이토록 엄격할까요? (Under the Hood)
파이썬의 `threading` 모듈로 개발해 본 경험이 있다면, 여러 스레드가 동시에 리스트에 접근해서 수정하려 할 때 데이터가 깨지는 것을 막기 위해 `Lock`을 걸어야 함을 알 것입니다.

만약 두 개의 스레드가 동시에 같은 힙 메모리를 가리키는 가변 참조(`&mut`)를 가질 수 있다면, 한 스레드가 메모리를 재할당(Reallocation)하여 주소가 바뀌는 순간 다른 스레드의 포인터는 엉뚱한 쓰레기 메모리(Dangling Pointer)를 가리키게 됩니다.

Rust 컴파일러(Borrow Checker)는 애초에 컴파일 타임에 **"동시에 두 명 이상 쓰기 접근 금지"**를 강제해 버립니다. 덕분에 Rust 프로그램은 실행 중에 알 수 없는 메모리 충돌이나 Data Race 현상이 물리적으로 발생할 수 없습니다.

### 한 줄 요약 규칙
1. **아무리 많은 불변 참조(`&T`)라도 동시에 존재할 수 있다.** (Readers-Many)
2. **가변 참조(`&mut T`)는 특정 시점에 오직 단 한 개만 존재할 수 있다.** (Writer-One)
3. **가변 참조와 불변 참조는 동시에 섞어 쓸 수 없다.**

지금까지 Rust가 메모리를 다루는 가장 중요한 원리인 '소유권과 대여'를 파이썬의 동적 모델과 비교해 보았습니다. 다음 장(Part 3)에서는 파이썬의 클래스를 대체하는 Rust의 데이터 모델링 기법(Struct, Enum, Trait)에 대해 심도 있게 다뤄보겠습니다.
