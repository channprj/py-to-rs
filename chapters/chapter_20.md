# Chapter 20. 스마트 포인터와 메모리 트릭

우리는 Chapter 4에서 소유권 규칙을 배우며, "가변 참조(`&mut`)는 동시에 하나만 존재할 수 있다"고 배웠습니다. 이 엄격한 규칙은 Data Race를 완벽히 막아주지만, 파이썬처럼 **여러 객체가 서로를 복잡하게 참조하는 그래프 구조(Linked List, Tree 등)**를 만들 때는 거대한 벽으로 다가옵니다.

파이썬 개발자들은 노드 간의 양방향 연결을 아주 쉽게 구현합니다.

* **The Python Way (Cyclic Reference)**
  ```python
  class Node:
      def __init__(self, value):
          self.value = value
          self.next = None
          self.prev = None # 양방향 연결 리스트

  node1 = Node(1)
  node2 = Node(2)

  node1.next = node2
  node2.prev = node1 # 파이썬은 이런 순환 참조를 허용하고, GC가 나중에 정리합니다.
  ```

이 구조를 순수 Rust의 `&mut` 참조만으로 구현하려면 컴파일러와 멱살을 잡고 싸우게 됩니다. `node1`이 `node2`를 가리키고, `node2`가 다시 `node1`을 가리킨다면, 누구의 수명이 더 긴지 컴파일 타임에 증명할 수 없기 때문입니다.

이를 해결하기 위해 Rust는 컴파일러의 깐깐한 규칙을 런타임으로 살짝 미뤄주는 **스마트 포인터(Smart Pointers)**를 제공합니다.

---

## 20.1 `Box<T>`: 힙(Heap)에 데이터 박아두기

Rust의 구조체는 기본적으로 스택(Stack)에 할당됩니다. 따라서 구조체의 크기는 컴파일 타임에 정확히 계산되어야 합니다. 그런데 만약 자기 자신을 포함하는 재귀적(Recursive) 구조체라면 어떨까요?

```rust
// 🚨 컴파일 에러! Node의 크기를 무한대까지 계산하게 됩니다.
// struct Node {
//     value: i32,
//     next: Option<Node>, 
// }
```

이때 포인터를 사용하여 "실제 데이터는 Heap에 있고, 나는 그 주소(8바이트)만 가지고 있어"라고 컴파일러를 안심시켜야 합니다. 파이썬의 객체 포인터와 가장 유사한 역할을 하는 것이 바로 `Box<T>`입니다.

* **The Rust Implementation (`Box<T>`)**
  ```rust
  struct Node {
      value: i32,
      // Box는 힙(Heap) 포인터입니다. 따라서 크기가 8바이트로 고정됩니다.
      next: Option<Box<Node>>, 
  }

  fn main() {
      // 데이터가 Heap에 할당되고, b는 그 포인터를 소유합니다.
      let b = Box::new(Node { 
          value: 1, 
          next: None 
      }); 
  } // b가 스코프를 벗어나면 Heap의 데이터도 즉시 해제(Drop)됩니다.
  ```

---

## 20.2 `Rc<T>`: 소유권 공유 (파이썬의 Reference Counting)

`Box`는 소유자가 단 한 명이어야 합니다. 두 개의 노드가 하나의 노드를 동시에 소유하고 싶다면 어떻게 해야 할까요? 앞서 멀티스레드 환경에서 `Arc`를 배웠지만, 단일 스레드에서는 더 가벼운 `Rc` (Reference Counted)를 사용합니다. 이것이 바로 파이썬의 핵심 메모리 관리 기법과 동일합니다.

```rust
use std::rc::Rc;

struct Node {
    value: i32,
    next: Option<Rc<Node>>, // 이제 여러 Node가 같은 Node를 가리킬 수 있습니다.
}

fn main() {
    let node3 = Rc::new(Node { value: 3, next: None });
    
    // node1과 node2가 모두 node3을 "공유"합니다!
    let node1 = Node { value: 1, next: Some(Rc::clone(&node3)) };
    let node2 = Node { value: 2, next: Some(Rc::clone(&node3)) };
    
    // node3의 참조 카운트는 3이 됩니다. (node3 본인, node1 안, node2 안)
}
```

---

## 20.3 `RefCell<T>`: Interior Mutability (내부 가변성)

`Rc`는 여러 곳에서 데이터를 공유할 수 있게 해주지만, 또 다른 장벽이 있습니다. `Rc`로 감싼 데이터는 **무조건 불변(Immutable)**이라는 것입니다. (공유된 상태를 함부로 바꾸면 버그가 나니까요).

파이썬처럼 변수를 공유하면서도 값을 요리조리 바꾸고 싶다면, **불변 껍질 속에 가변 데이터를 숨기는 마법**이 필요합니다. 이를 내부 가변성(Interior Mutability) 패턴이라고 부르며, `RefCell<T>`이 그 역할을 합니다.

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct SharedData {
    value: i32,
}

fn main() {
    // Rc로 공유하고, RefCell로 가변성을 허용합니다. (가장 널리 쓰이는 조합)
    let data = Rc::new(RefCell::new(SharedData { value: 0 }));

    let data_clone1 = Rc::clone(&data);
    let data_clone2 = Rc::clone(&data);

    // data_clone1은 겉보기엔 불변이지만, borrow_mut()을 통해 내부를 수정할 수 있습니다!
    data_clone1.borrow_mut().value += 10;
    data_clone2.borrow_mut().value += 20;

    println!("Total: {}", data.borrow().value); // 30
}
```

### 주의할 점: 런타임 패닉 (Runtime Panic)
`RefCell`은 컴파일 타임의 대여 검사(Borrow Checking)를 **런타임**으로 미룹니다. 즉, 파이썬처럼 코딩하다가 동시에 두 곳에서 `borrow_mut()`(가변 참조)을 호출하려 들면, 컴파일은 통과하지만 런타임에 프로그램이 패닉(Panic)을 일으키며 뻗어버립니다.

Rust의 스마트 포인터(`Box`, `Rc`, `RefCell`)는 결국 **"어디서든 공유하고 수정할 수 있는 파이썬 객체"**를 명시적이고 조립 가능한 부품으로 쪼개놓은 것과 같습니다. 이 조각들을 잘 맞추면 파이썬의 유연함을 잃지 않으면서도 메모리 누수를 원천 차단하는 견고한 아키텍처를 세울 수 있습니다.

이제 길었던 언어 탐험을 마치고, 파이썬과 Rust가 물리적으로 어떻게 손을 잡을 수 있는지, 이 책의 대미를 장식할 **Part 8: PyO3**로 넘어가 보겠습니다.
