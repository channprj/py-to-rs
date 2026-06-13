# Chapter 20. 스마트 포인터와 메모리 트릭

지난 19장에서 우리는 매크로(Macro)가 파이썬의 데코레이터와 메타클래스를 어떻게 컴파일 타임의 견고한 요새 안으로 끌어들이는지 살펴봤습니다. 매크로가 "코드를 짜는 코드"였다면, 이번 장은 "메모리를 다루는 메모리"에 관한 이야기입니다.

10년 차 파이썬 개발자라면 객체 그래프(Object Graph)를 머릿속으로 그리는 데 익숙합니다. ORM의 양방향 관계(`user.posts`와 `post.author`), 트리 구조의 DOM, 옵저버 패턴의 subject-observer 양방향 참조, 캐시 무효화를 위한 의존성 그래프 — 파이썬에서 이런 구조는 그냥 속성에 객체 레퍼런스를 꽂아 넣으면 끝납니다. `self.parent = parent`, `parent.children.append(self)`. 두 줄이면 됩니다. 인터프리터가 참조 카운트(Reference Count)를 올리고, 순환이 생기면 세대별 가비지 컬렉터(Generational GC)가 알아서 정리해 줍니다. 우리는 메모리가 언제 해제되는지 단 한 번도 신경 쓰지 않고 10년을 살아왔습니다.

그런데 Chapter 4에서 배운 Rust의 소유권(Ownership) 규칙은 이 모든 것을 정면으로 가로막습니다. "값의 소유자(Owner)는 단 하나뿐이고, 가변 참조(`&mut`)는 동시에 하나만 존재할 수 있다." 이 단일 규칙은 데이터 경합(Data Race)과 Use-After-Free를 컴파일 타임에 박멸하는 강력한 무기지만, 동시에 **여러 노드가 서로를 가리키는 그래프 구조**를 만들려는 순간 거대한 벽이 됩니다. node1이 node2를 소유하고, node2가 다시 node1을 소유한다면 — 도대체 누가 누구의 소유자입니까? 누구의 수명(Lifetime)이 더 깁니까? 컴파일러는 이 질문에 답할 수 없으면 빌드를 거부합니다.

* **The Python Way (Cyclic Reference)**
  ```python
  class Node:
      def __init__(self, value: int) -> None:
          self.value = value
          self.next: "Node | None" = None
          self.prev: "Node | None" = None  # 양방향 연결 리스트

  node1 = Node(1)
  node2 = Node(2)

  node1.next = node2
  node2.prev = node1  # 파이썬은 이런 순환 참조를 허용하고, GC가 나중에 정리합니다.
  ```

이 구조를 순수 Rust의 `&mut` 참조만으로 구현하려면 컴파일러와 멱살을 잡고 싸우게 됩니다. 그래서 Rust는 컴파일러의 깐깐한 규칙을 **선택적으로 런타임으로 옮기거나, 소유권을 여러 곳에 나눠 갖거나, 힙(Heap)으로 데이터를 밀어내는** 도구들을 제공합니다. 이것이 바로 **스마트 포인터(Smart Pointer)**입니다.

스마트 포인터는 마법이 아닙니다. 파이썬이 인터프리터 내부에 숨겨 둔 메커니즘 — 힙 할당, 참조 카운팅, 가변 객체 — 을 명시적이고 조립 가능한 부품으로 쪼개 놓은 것입니다. 파이썬 객체 하나가 사실은 `Box`(힙 할당) + `Rc`(참조 카운팅) + `RefCell`(언제든 가변)의 조합이었던 셈입니다. 이 장에서는 그 부품들을 하나씩 분해하고, 마지막에는 다시 조립해서 파이썬의 양방향 객체 그래프를 누수 없이 재현해 봅니다.

---

## 20.1 The Python Way — 모든 객체가 스마트 포인터다

파이썬에서 우리가 다루는 모든 것은 사실 포인터입니다. `a = [1, 2, 3]`이라고 쓰면 변수 `a`는 리스트 객체 자체가 아니라 힙 어딘가에 있는 `PyListObject`의 **주소**를 담고 있습니다. `b = a`는 그 객체를 복사하지 않습니다. 같은 주소를 `b`에도 복사할 뿐이고, 객체 헤더의 `ob_refcnt` 필드가 1에서 2로 증가합니다.

```python
import sys

a = [1, 2, 3]
print(sys.getrefcnt(a))  # 2 (a 자신 + getrefcnt 인자로 잠깐 들어간 것)

b = a                    # 같은 객체를 가리킴, refcount += 1
print(sys.getrefcnt(a))  # 3

del b                    # refcount -= 1
print(sys.getrefcnt(a))  # 2
```

이것이 CPython의 메모리 관리 핵심입니다. 모든 `PyObject`는 헤더에 `ob_refcnt`(참조 카운트)를 들고 있고, 이름이 객체에 바인딩될 때마다 카운트가 오르고, 이름이 사라지면(`del`, 스코프 종료, 재할당) 내려갑니다. 카운트가 0이 되는 순간 객체는 즉시 해제됩니다.

문제는 두 가지입니다. 첫째, 이 카운팅에는 비용이 있습니다. `Py_INCREF`/`Py_DECREF`는 단순 정수 증감처럼 보이지만, 멀티스레드 안전성을 위해(GIL 보호 하에서도) 캐시 라인을 더럽히고, 객체에 접근할 때마다 발생합니다. 둘째, **순환 참조는 카운트만으로는 절대 0이 되지 않습니다.** `node1.next = node2; node2.prev = node1`을 만들면 둘의 카운트는 영원히 1 이상으로 남습니다. CPython은 이 누수를 막으려고 별도의 순환 탐지 가비지 컬렉터(`gc` 모듈)를 추가로 돌립니다. 즉, 파이썬은 **참조 카운팅 + 순환 탐지 GC**라는 이중 시스템을 항상 백그라운드에서 돌리고 있는 것입니다.

```python
import gc

class Node:
    def __init__(self) -> None:
        self.ref: "Node | None" = None

a = Node()
b = Node()
a.ref = b
b.ref = a       # 순환! 두 객체의 refcount는 각각 2가 됨
del a
del b           # 이름은 사라졌지만 객체는 살아있다 (서로를 붙잡고 있으므로)

# gc가 돌기 전까지 메모리에 떠다님. 명시적으로 수거해야 함.
collected = gc.collect()
print(f"순환 객체 {collected}개 수거됨")  # 일반적으로 2 이상
```

이 이중 시스템에는 미묘한 함정이 하나 더 있습니다. 순환에 얽힌 객체가 `__del__` 소멸자를 정의하고 있으면, CPython의 순환 GC는 **어느 순서로 소멸자를 불러야 할지 결정할 수 없어** 한때 이들을 `gc.garbage`에 방치하기도 했습니다(파이썬 3.4의 PEP 442 이전). 즉, 파이썬에서조차 "순환 + 소멸자"는 메모리 관리의 어두운 구석이었습니다. 10년 차 개발자라면 운영 중인 서비스에서 메모리가 슬금슬금 차오르다 OOM으로 죽고, `objgraph`나 `gc.get_objects()`로 새벽까지 누수의 근원을 추적해 본 경험이 한 번쯤 있을 겁니다. 그 누수의 상당수가 바로 이 순환 참조였습니다.

Rust는 이 두 가지 비용을 정직하게 드러냅니다. "참조 카운팅이 필요하면 `Rc`를 명시적으로 써라. 그 비용을 코드에서 보이게 하라. 그리고 순환을 막는 것은 GC가 아니라 **너의 설계**다." 파이썬이 자동으로 처리하던 것을 Rust는 개발자에게 선택하게 만듭니다. 대신 그 대가로, 백그라운드 GC가 없고(따라서 GC로 인한 예측 불가능한 멈춤(stop-the-world pause)도 없고), 메모리 해제 시점이 결정론적(Deterministic)이며, 컴파일러가 대부분의 실수를 잡아 줍니다. 객체가 마지막 소유자를 잃는 바로 그 줄에서 메모리가 해제된다는 것을 코드만 보고도 알 수 있다는 것 — 이것이 GC 언어에서 온 개발자가 처음엔 불편해하다 결국 사랑하게 되는 Rust의 특성입니다.

---

## 20.2 The Rust Mental Model — 비용을 눈에 보이게

Rust의 기본값은 **스택 할당 + 단일 소유권**입니다. 구조체를 만들면 스택에 놓이고, 크기는 컴파일 타임에 확정되며, 소유자는 하나고, 스코프를 벗어나면 즉시 해제됩니다. 이 기본값으로 풀리지 않는 문제가 생길 때만, 우리는 정확히 필요한 만큼의 능력을 가진 스마트 포인터를 "추가 비용을 지불하고" 꺼내 씁니다.

스마트 포인터를 고를 때의 사고 흐름은 세 가지 질문으로 정리됩니다.

1. **데이터를 힙에 둬야 하는가?** (크기가 컴파일 타임에 불확정이거나, 재귀적이거나, 트레이트 객체이거나) → `Box<T>`
2. **소유자가 여러 명이어야 하는가?** (그래프의 한 노드를 여러 노드가 공유) → 단일 스레드면 `Rc<T>`, 멀티스레드면 `Arc<T>`
3. **공유하면서 동시에 수정해야 하는가?** (불변 공유 참조 뒤에서 값을 바꿔야) → 단일 스레드면 `RefCell<T>` / `Cell<T>`, 멀티스레드면 `Mutex<T>` / `RwLock<T>`

이 질문들은 직교(Orthogonal)합니다. 그래서 조합됩니다. "여러 곳에서 공유하면서 수정도 하고 싶다"면 답은 `Rc<RefCell<T>>`이고, 이것이 바로 **파이썬 객체의 정체**입니다. 멀티스레드 버전이 `Arc<Mutex<T>>`이며, 이것은 Chapter 17에서 이미 다뤘습니다. 이 장은 그 단일 스레드 사촌들을 깊이 파헤칩니다.

이제 각 부품을 분해해 봅시다. 모든 스마트 포인터는 공통적으로 `Deref` 트레이트(`*` 연산자로 내부 값에 접근)와 `Drop` 트레이트(스코프 종료 시 정리 로직)를 구현합니다. 이 두 트레이트가 "스마트"의 본질입니다 — 평범한 포인터처럼 역참조되지만, 해제 시점에 추가 동작(카운트 감소, 힙 해제)을 자동으로 수행하는 것이죠.

| 파이썬 개념 | Rust 도구 | 핵심 능력 | 비용 |
|---|---|---|---|
| 객체의 힙 할당 (`PyObject*`) | `Box<T>` | 단일 소유 힙 포인터, 재귀 타입 가능 | 힙 할당 1회, 포인터 8바이트 |
| 객체 참조 카운트 (`ob_refcnt`) | `Rc<T>` | 단일 스레드 공유 소유권 | clone 시 카운트 증가(원자 아님) |
| 멀티스레드 안전 공유 | `Arc<T>` | 스레드 간 공유 소유권 | 원자적(Atomic) 카운트 증감 |
| 언제든 가변 (모든 객체) | `RefCell<T>` | 런타임 대여 검사, 내부 가변성 | 런타임 플래그 체크, 위반 시 panic |
| 단순 값 교체 (`Copy` 타입) | `Cell<T>` | 대여 없이 값 통째 교체 | 거의 0 (단, 참조 불가) |
| `threading.Lock` 보호 객체 | `Mutex<T>` | 스레드 간 배타적 가변 접근 | OS 락 또는 스핀락 |
| 읽기 다수 / 쓰기 소수 | `RwLock<T>` | 다중 읽기 또는 단일 쓰기 | 읽기/쓰기 카운터 |
| 복사할지 말지 미루기 | `Cow<T>` | 수정 전엔 빌림, 수정 시에만 복사 | 쓰기 시점에만 할당 |
| 순환 깨는 약한 참조 | `Weak<T>` | 소유하지 않는 참조(카운트 비증가) | upgrade 시 살아있는지 확인 |

이 표가 이 장 전체의 지도입니다. 이제 좌상단부터 시작합니다.

### 비용을 숫자로 확인하기

"비용을 눈에 보이게"라는 철학이 추상적으로 들릴 수 있으니, 실제 수치로 확인해 봅시다. 다음은 각 포인터의 핵심 연산을 2억 회 반복하며 `std::hint::black_box`로 최적화 제거를 막고 `--release`로 측정한 결과입니다(Apple Silicon, rustc 1.96 기준, 단일 스레드).

```text
Rc  clone+drop          : ~7.5 ns/op   (비원자 카운트 증감)
Arc clone+drop          : ~7.0 ns/op   (원자 카운트 증감, 경합 없음)
RefCell borrow_mut+write : ~1.7 ns/op   (플래그 체크 + 쓰기)
```

여기서 시니어 개발자가 오해하기 쉬운 함정이 있습니다. **경합이 없는 단일 스레드에서는 `Arc`가 `Rc`보다 별로 느리지 않습니다.** 위 수치에서 둘은 거의 같습니다. 현대 CPU에서 캐시 라인을 독점한 상태의 원자적 증감(atomic fetch-add)은 일반 증감과 큰 차이가 없기 때문입니다. `Arc`가 진짜 비싸지는 순간은 **여러 코어가 같은 카운터를 동시에 두드릴 때**입니다 — 그때 캐시 라인이 코어 사이를 핑퐁처럼 오가며(cache-line bouncing) 수십~수백 사이클로 치솟습니다. 그래서 "단일 스레드면 `Rc`를 쓰라"는 조언의 진짜 이유는 단일 스레드 성능 차이가 아니라, **`Arc`가 멀티스레드를 암시하는 잘못된 신호를 주지 않게** 하고 경합 가능성을 타입에서 배제하는 데 있습니다. `RefCell`의 `borrow_mut`이 ~1.7ns로 가장 싼 것도 주목할 만합니다 — 내부 가변성의 런타임 비용은 정수 비교 한두 번이라, "런타임 검사"라는 말이 주는 부담스러운 인상과 달리 실제로는 무시할 만합니다.

참고로 파이썬의 `Py_INCREF`/`Py_DECREF`는 객체에 접근할 때마다 발생하며, 단순 속성 접근 `obj.attr` 한 번에도 여러 번의 refcount 연산이 끼어듭니다. Rust는 `Rc::clone`을 명시적으로 호출한 그 지점에서만 카운트를 건드리므로, 카운팅 횟수 자체가 압도적으로 적습니다. 이것이 동일한 참조 카운팅 메커니즘인데도 Rust가 메모리 관리에서 더 빠른 근본 이유입니다.

---

## 20.3 `Box<T>`: 힙(Heap)에 데이터 박아두기

Rust의 구조체는 기본적으로 스택(Stack)에 할당됩니다. 따라서 구조체의 크기는 컴파일 타임에 정확히 계산되어야 합니다. 컴파일러는 함수의 스택 프레임(Stack Frame)을 잡을 때 각 변수가 몇 바이트인지 알아야 하기 때문입니다. 그런데 만약 자기 자신을 포함하는 재귀적(Recursive) 구조체라면 어떨까요?

```rust
// 🚨 컴파일 에러! Node의 크기를 무한대까지 계산하게 됩니다.
struct Node {
    value: i32,
    next: Option<Node>,
}
```

컴파일러는 `Node`의 크기를 계산하려고 합니다. `Node` = `i32`(4바이트) + `Option<Node>`. 그런데 `Option<Node>`의 크기는 `Node`의 크기에 의존하고, `Node`의 크기는 다시 `Option<Node>`에 의존합니다. 4 + (4 + (4 + ...)) — 무한 재귀입니다. 실제 `rustc`가 내뱉는 에러를 봅시다.

```text
error[E0072]: recursive type `Node` has infinite size
 --> src/main.rs:1:1
  |
1 | struct Node {
  | ^^^^^^^^^^^
2 |     value: i32,
3 |     next: Option<Node>,
  |                  ---- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
  |
3 |     next: Option<Box<Node>>,
  |                  ++++    +
```

컴파일러는 친절하게도 정답까지 알려 줍니다. "간접 참조(indirection)를 넣어라 — `Box`, `Rc`, 혹은 `&`로." 핵심 단어는 **indirection**입니다. 데이터를 구조체 안에 직접 박는 대신, "실제 데이터는 힙 어딘가에 있고, 나는 그 주소(고정 크기 8바이트)만 들고 있다"라고 컴파일러를 안심시키는 것이죠. 파이썬의 객체 포인터와 가장 유사한 역할을 하는 것이 바로 `Box<T>`입니다.

* **The Rust Implementation (`Box<T>`)**
  ```rust
  struct Node {
      value: i32,
      // Box는 힙(Heap) 포인터입니다. 따라서 크기가 8바이트로 고정됩니다.
      next: Option<Box<Node>>,
  }

  fn main() {
      // 1 -> 2 -> 3 형태의 연결 리스트. 데이터는 모두 힙에 있습니다.
      let list = Node {
          value: 1,
          next: Some(Box::new(Node {
              value: 2,
              next: Some(Box::new(Node { value: 3, next: None })),
          })),
      };

      // 역참조(*)와 점 연산자로 힙의 데이터에 접근합니다.
      print!("{}", list.value);
      let mut cur = &list.next;
      while let Some(boxed) = cur {
          print!(" -> {}", boxed.value);
          cur = &boxed.next;
      }
      println!();
  } // list가 스코프를 벗어나면 연결된 모든 Node가 힙에서 즉시 해제(Drop)됩니다.
  ```

이제 `Node`의 크기는 `i32`(4바이트) + `Option<Box<Node>>`(8바이트, 정렬 패딩 포함하면 16바이트)로 컴파일 타임에 확정됩니다. `Option<Box<T>>`가 추가 바이트 없이 8바이트인 이유는 Chapter 12에서 본 **널 포인터 최적화(Null Pointer Optimization)** 덕분입니다 — `Box`는 절대 널이 될 수 없으므로, 컴파일러는 "주소 0"이라는 비트 패턴을 `None`을 표현하는 데 재활용합니다.

> 💡 **Under the Hood: Box의 메모리 레이아웃**
>
> `Box<T>`는 런타임에 그냥 **포인터 하나**입니다. 다른 어떤 메타데이터도 없습니다.
>
> ```text
> 스택:                힙:
> ┌──────────────┐    ┌─────────────────────┐
> │ Box<Node>    │    │ Node { value: 1,    │
> │  ptr ────────┼───▶│        next: ...   }│
> │  (8 bytes)   │    │  (16 bytes)         │
> └──────────────┘    └─────────────────────┘
> ```
>
> `std::mem::size_of::<Box<i32>>()`는 정확히 **8바이트**(64비트 머신의 포인터 크기)입니다. C의 `malloc`이 반환하는 포인터와 비트 단위로 동일합니다. 차이는 단 하나 — `Box`가 스코프를 벗어날 때 컴파일러가 자동으로 `Drop` 코드를 삽입해 힙을 해제한다는 점입니다. 이것이 "스마트"의 정체입니다. C라면 `free()`를 손으로 불러야 하고, 파이썬이라면 GC가 나중에 처리하지만, Rust는 컴파일 타임에 정확한 해제 지점을 코드에 박아 넣습니다.

### `Box<dyn Trait>`: 동적 디스패치와 팻 포인터(Fat Pointer)

`Box`의 또 다른 핵심 용도는 트레이트 객체(Trait Object)의 박싱입니다. 파이썬에서는 서로 다른 클래스의 객체를 한 리스트에 자유롭게 담습니다 — 덕 타이핑(Duck Typing) 덕분이죠. `widgets = [Button(), Slider(), TextBox()]`. Rust에서 크기가 제각각인 타입들을 한 `Vec`에 담으려면, 그들을 공통 트레이트 뒤에 숨기고 박싱해야 합니다.

```rust
trait Draw {
    fn draw(&self) -> String;
}

struct Button;
struct Slider;

impl Draw for Button {
    fn draw(&self) -> String { "[Button]".into() }
}
impl Draw for Slider {
    fn draw(&self) -> String { "[Slider]".into() }
}

fn main() {
    // 크기가 다른 타입들을 하나의 Vec에 담으려면 박싱이 필요합니다.
    let widgets: Vec<Box<dyn Draw>> = vec![Box::new(Button), Box::new(Slider)];
    for w in &widgets {
        println!("{}", w.draw());  // 런타임에 vtable을 통해 실제 메서드를 찾음
    }

    println!("Box<dyn Draw> 크기: {} 바이트", std::mem::size_of::<Box<dyn Draw>>()); // 16
    println!("Box<i32>      크기: {} 바이트", std::mem::size_of::<Box<i32>>());      // 8
}
```

여기서 `Box<dyn Draw>`의 크기가 **16바이트**라는 점에 주목하세요. `Box<i32>`가 8바이트였던 것과 다릅니다. 이것이 **팻 포인터(Fat Pointer)**입니다.

> 💡 **Under the Hood: 동적 디스패치와 vtable**
>
> `Box<dyn Trait>`는 포인터 두 개로 이루어진 16바이트 팻 포인터입니다.
>
> ```text
> ┌────────────────────────┐
> │ Box<dyn Draw>          │
> │  data_ptr ─────────────┼──▶ 힙의 Button 인스턴스
> │  vtable_ptr ───────────┼──▶ Button의 vtable
> │  (16 bytes)            │     ┌──────────────────┐
> └────────────────────────┘     │ drop_in_place    │
>                                 │ size, align      │
>                                 │ draw() 함수 주소  │
>                                 └──────────────────┘
> ```
>
> `w.draw()`를 호출하면, 런타임에 vtable에서 `draw`의 실제 함수 주소를 찾아 점프합니다. 이것이 **동적 디스패치(Dynamic Dispatch)**입니다 — 파이썬의 메서드 호출(MRO를 따라 `__dict__`를 뒤지는)과 개념적으로 같지만, 파이썬이 매번 해시맵 조회를 하는 반면 Rust는 고정 오프셋의 함수 포인터 한 번 역참조로 끝납니다. Chapter 9의 정적 디스패치(`impl Trait`, 제네릭의 단일형화)와 달리 인라이닝은 불가능하지만, 이종(heterogeneous) 컬렉션을 만들 수 있다는 유연성을 얻습니다. 트레이드오프는 명확합니다.

`Box`는 또한 파이썬의 예외 클래스처럼 크기가 천차만별인 에러 타입을 통일할 때도 쓰입니다(`Box<dyn Error>`, Chapter 13 참고). 거대한 값을 함수 인자로 넘길 때 스택 복사 대신 힙으로 옮겨 이동(move) 비용을 줄이는 데도 사용됩니다. 하지만 `Box`의 결정적 한계는 여전히 **소유자가 단 한 명**이라는 것입니다. 두 노드가 하나를 동시에 소유할 수는 없습니다.

---

## 20.4 `Rc<T>`: 소유권 공유 (파이썬의 Reference Counting)

두 개의 노드가 하나의 노드를 동시에 소유하고 싶다면 어떻게 해야 할까요? 그래프에서 두 정점이 같은 자식을 공유하는 흔한 상황입니다. `Box`로는 불가능합니다 — 소유자는 하나뿐이니까요. 이때 `Rc<T>`(Reference Counted)를 씁니다. 이름 그대로, 이것이 바로 파이썬의 `ob_refcnt`와 **개념적으로 동일한** 메커니즘입니다.

```rust
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    value: i32,
    next: Option<Rc<Node>>,
}

fn main() {
    let shared = Rc::new(Node { value: 3, next: None });
    println!("생성 직후 count = {}", Rc::strong_count(&shared)); // 1

    // node1과 node2가 모두 shared를 "공유"합니다!
    let node1 = Node { value: 1, next: Some(Rc::clone(&shared)) };
    println!("node1이 공유 후 count = {}", Rc::strong_count(&shared)); // 2

    let node2 = Node { value: 2, next: Some(Rc::clone(&shared)) };
    println!("node2가 공유 후 count = {}", Rc::strong_count(&shared)); // 3

    {
        let node3 = Node { value: 4, next: Some(Rc::clone(&shared)) };
        println!("스코프 내부 count = {}", Rc::strong_count(&shared)); // 4
        let _ = node3;
    } // node3이 여기서 Drop → 카운트 자동 감소

    println!("스코프 종료 후 count = {}", Rc::strong_count(&shared)); // 3
    let _ = (node1, node2);
}
```

`Rc::clone(&shared)`은 데이터를 복제하지 않습니다. 이름에 "clone"이 들어가 있지만, 실제로 하는 일은 **힙의 데이터를 가리키는 포인터를 복사하고 강한 참조 카운트(strong count)를 1 증가**시키는 것뿐입니다. 파이썬의 `b = a`와 정확히 같은 일이죠. 그래서 관례적으로 `shared.clone()`이 아니라 `Rc::clone(&shared)`라고 명시적으로 씁니다 — "이건 깊은 복사가 아니라 그냥 카운트 증가야"라고 읽는 사람에게 알려 주기 위해서입니다(clippy도 이 스타일을 권장합니다).

각 `Rc`가 스코프를 벗어나면 `Drop`이 호출되어 카운트가 1 감소하고, 카운트가 0이 되는 순간 힙의 데이터가 즉시 해제됩니다. 위 예제에서 `node3`이 내부 스코프를 벗어날 때 카운트가 4에서 3으로 떨어지는 것을 직접 확인할 수 있습니다. 이것은 파이썬의 `del`이 일으키는 일과 똑같지만, 시점이 **컴파일 타임에 코드로 박혀 있어** 결정론적입니다.

> 💡 **Under the Hood: Rc의 카운터 힙 레이아웃**
>
> `Rc<T>`는 스택에 포인터 하나(8바이트)만 둡니다. 하지만 힙에는 데이터뿐 아니라 **두 개의 카운터**가 함께 들어 있는 `RcBox`라는 구조가 할당됩니다.
>
> ```text
> 스택:              힙 (RcBox<Node>):
> ┌───────────┐     ┌──────────────────────────┐
> │ Rc<Node>  │     │ strong: usize  (= 3)     │  ← 강한 참조 수
> │  ptr ─────┼────▶│ weak:   usize  (= 0)     │  ← 약한 참조 수 (20.7절)
> │  (8 byte) │     │ value:  Node { value:3 } │  ← 실제 데이터
> └───────────┘     └──────────────────────────┘
> ```
>
> `Rc::clone`은 `strong`을 `+1`, `Rc`의 `Drop`은 `-1` 합니다. `strong`이 0이 되면 `value`가 해제되고, `weak`까지 0이 되면 `RcBox` 블록 전체가 해제됩니다. CPython의 `PyObject` 헤더에 `ob_refcnt`가 박혀 있는 것과 구조가 거의 같습니다. 결정적 차이는 단 하나 — **`Rc`의 카운트 증감은 원자적(atomic)이 아닙니다.** 그냥 평범한 `usize += 1`입니다. 락도, 메모리 배리어도 없습니다. 그래서 단일 스레드에서는 `Arc`보다 훨씬 빠르지만, 두 스레드가 동시에 카운트를 건드리면 경합이 일어나므로 `Rc`는 컴파일 타임에 스레드 경계를 넘지 못하도록 차단됩니다(20.9절에서 그 에러를 직접 봅니다).

그런데 `Rc`에는 치명적인 제약이 하나 더 있습니다. **`Rc`로 감싼 데이터는 불변(Immutable)입니다.** `Rc<Node>`를 통해 `node.value`를 수정할 방법이 없습니다. 생각해 보면 당연합니다 — 여러 소유자가 같은 데이터를 가리키는데 그중 하나가 마음대로 값을 바꾸면, 다른 소유자들의 불변 참조 가정이 깨지고 데이터 경합의 단일 스레드 버전이 됩니다. Rust는 이를 허용하지 않습니다. `Rc<T>`는 오직 `&T`(공유 불변 참조)만 내어 줍니다.

파이썬 객체처럼 "여러 곳에서 공유하면서 동시에 수정"하려면, 불변의 껍질 속에 가변성을 몰래 숨기는 기술이 필요합니다.

---

## 20.5 `RefCell<T>`와 `Cell<T>`: 내부 가변성 (Interior Mutability)

`Rc`로 감싼 데이터는 무조건 불변이라는 장벽을 넘는 도구가 **내부 가변성(Interior Mutability)** 패턴이고, 그 대표 주자가 `RefCell<T>`입니다.

내부 가변성의 핵심 아이디어는 이렇습니다. Rust의 대여 규칙("불변 참조는 여럿 OR 가변 참조는 하나")은 두 군데에서 검사할 수 있습니다 — **컴파일 타임** 또는 **런타임**. `&`/`&mut`는 컴파일러가 정적으로 검사합니다. `RefCell<T>`은 이 검사를 **런타임으로 미룹니다.** 즉, `RefCell`은 겉보기에 불변(`&self`)이지만, 내부적으로 대여 상태를 추적하는 플래그를 들고 있다가, 규칙을 위반하는 대여 요청이 오면 컴파일이 아니라 **런타임에 panic**을 일으킵니다.

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(vec![1, 2, 3]);

    // borrow() → Ref<T> (불변 대여, 여러 개 OK)
    {
        let r1 = cell.borrow();
        let r2 = cell.borrow();
        println!("len={}, first={}", r1.len(), r2[0]); // 불변 대여 2개 공존 OK
    } // r1, r2 Drop → 대여 해제

    // borrow_mut() → RefMut<T> (가변 대여, 동시에 하나만)
    {
        let mut w = cell.borrow_mut();
        w.push(4);
    } // w Drop → 가변 대여 해제

    println!("{:?}", cell.borrow()); // [1, 2, 3, 4]
}
```

겉보기엔 `cell`이 불변(`let cell`, `mut` 없음)인데도 `borrow_mut()`으로 내부 `Vec`을 수정했습니다. 이것이 내부 가변성입니다. `borrow()`는 `Ref<T>` 가드를, `borrow_mut()`은 `RefMut<T>` 가드를 반환하고, 이 가드들이 살아 있는 동안 대여가 유지되며, 가드가 `Drop`될 때 대여가 해제됩니다 — `Mutex`의 락 가드(Chapter 17)와 똑같은 RAII 패턴입니다.

### 🚨 런타임 패닉 시연: 이중 가변 대여

`RefCell`의 검사가 런타임이라는 말은, **컴파일은 멀쩡히 통과하지만 실행 중에 터질 수 있다**는 뜻입니다. 파이썬처럼 무심코 코딩하다가 동시에 두 곳에서 가변 대여를 시도하면, 컴파일러는 아무 말도 하지 않지만 프로그램이 패닉으로 뻗습니다.

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(vec![1, 2, 3]);
    let _first = cell.borrow_mut();   // 첫 번째 가변 대여
    let _second = cell.borrow_mut();  // 🚨 panic! 이미 가변 대여 중
    println!("이 줄에는 도달하지 못합니다");
}
```

이 코드는 **컴파일에 성공합니다.** 하지만 실행하면:

```text
thread 'main' panicked at src/main.rs:6:24:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`first` 가드가 아직 살아 있는 상태에서 `second`가 두 번째 가변 대여를 요청하자, `RefCell`이 내부 플래그를 확인하고 "이미 빌려줬다"며 `panic!`을 던진 것입니다. 컴파일러가 잡아 주던 `error[E0499]`(cannot borrow as mutable more than once)가 런타임 패닉으로 자리만 옮긴 것이죠. 이것이 `RefCell`을 쓰는 대가입니다 — 유연성을 얻는 대신 컴파일 타임 안전망의 일부를 포기합니다.

> 💡 **Under the Hood: RefCell의 런타임 대여 플래그**
>
> `RefCell<T>`은 데이터 옆에 `borrow`라는 작은 정수 플래그(`Cell<isize>`)를 하나 더 들고 있습니다. 이 플래그가 현재 대여 상태를 인코딩합니다.
>
> ```text
> ┌──────────────────────────────────┐
> │ RefCell<Vec<i32>>                │
> │  borrow:  isize                  │  ← 0: 미대여
> │                                  │     양수 N: 불변 대여 N개
> │                                  │     -1: 가변 대여 1개
> │  value:   UnsafeCell<Vec<i32>>   │  ← 실제 데이터
> └──────────────────────────────────┘
> ```
>
> `borrow()`를 부르면 플래그가 음수인지(가변 대여 중인지) 확인하고, 아니면 `+1` 합니다. `borrow_mut()`은 플래그가 0인지 확인하고, 0이면 `-1`로 만듭니다. 0이 아니면 panic. `Ref`/`RefMut` 가드가 `Drop`될 때 플래그가 원상 복구됩니다. 이 모든 검사는 정수 비교 한두 번 — 그래서 런타임 비용이 거의 없습니다. 내부 데이터는 `UnsafeCell<T>`로 감싸여 있는데, 이것이 Rust 전체에서 **불변 참조 뒤에서 가변 접근을 합법화하는 유일한 원시 도구**입니다. `Cell`, `RefCell`, `Mutex`, `RwLock` 전부 그 바닥에는 `UnsafeCell`이 있습니다.

패닉이 무서우면 `try_borrow_mut()`을 쓸 수 있습니다. 패닉 대신 `Result`를 돌려주므로 우아하게 처리할 수 있습니다.

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(0);
    let _g = cell.borrow_mut();
    match cell.try_borrow_mut() {
        Ok(_) => println!("두 번째 대여 성공"),
        Err(e) => println!("두 번째 대여 거부됨: {e}"), // RefCell already borrowed
    }
}
```

### `Cell<T>`: 더 가벼운 사촌

`RefCell`이 데이터를 "대여"해 주는(참조를 빌려주는) 반면, `Cell<T>`은 데이터를 통째로 **교체(set)하거나 꺼내(get/replace)** 가는 방식입니다. 내부 값에 대한 참조를 절대 외부로 내보내지 않기 때문에, 대여 플래그조차 필요 없습니다 — 그래서 런타임 비용이 사실상 0이고 패닉도 없습니다. 대신 `get`은 값을 복사해서 돌려주므로 `T: Copy`인 작은 값(카운터, 플래그, 작은 숫자)에 적합합니다.

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>, // &self 만으로 내부 값 교체 가능
}

impl Counter {
    fn tick(&self) {
        // 대여 없이 값을 통째로 꺼내 +1 한 뒤 다시 넣음
        self.count.set(self.count.get() + 1);
    }
}

fn main() {
    let c = Counter { count: Cell::new(0) };
    c.tick();
    c.tick();
    println!("count = {}", c.count.get()); // 2
}
```

선택 기준은 간단합니다. **작은 `Copy` 값이고 참조가 필요 없으면 `Cell`, 그 외 모든 경우는 `RefCell`.** `Cell`은 옵저버 패턴의 더티 플래그나 호출 횟수 카운터처럼 가벼운 상태에 쓰고, 컬렉션이나 큰 구조체는 `RefCell`로 갑니다.

---

## 20.6 `Rc<RefCell<T>>`: 파이썬 객체의 재현

이제 부품을 조립할 차례입니다. `Rc`는 "여러 곳에서 공유"를, `RefCell`은 "언제든 수정"을 줍니다. 둘을 겹치면 `Rc<RefCell<T>>` — **여러 곳에서 공유하면서 어디서든 수정 가능한** 데이터, 즉 평범한 파이썬 객체가 됩니다.

```python
# The Python Way — 공유 + 수정이 당연한 세계
class SharedData:
    def __init__(self, value: int) -> None:
        self.value = value

data = SharedData(0)
clone1 = data   # 같은 객체 참조
clone2 = data

clone1.value += 10
clone2.value += 20
print(data.value)  # 30 — 셋 다 같은 객체
```

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct SharedData {
    value: i32,
}

fn main() {
    // Rc로 공유, RefCell로 가변성 허용 — 가장 널리 쓰이는 조합
    let data = Rc::new(RefCell::new(SharedData { value: 0 }));

    let clone1 = Rc::clone(&data);
    let clone2 = Rc::clone(&data);

    // 겉보기엔 불변이지만 borrow_mut()으로 내부를 수정합니다.
    clone1.borrow_mut().value += 10;
    clone2.borrow_mut().value += 20;

    println!("Total: {}", data.borrow().value);            // 30
    println!("strong count: {}", Rc::strong_count(&data)); // 3
}
```

파이썬 코드와 Rust 코드가 거의 1:1로 대응합니다. `data = SharedData(0)` ↔ `Rc::new(RefCell::new(...))`, `clone1 = data` ↔ `Rc::clone(&data)`, `clone1.value += 10` ↔ `clone1.borrow_mut().value += 10`. 파이썬이 암묵적으로 하던 일을 Rust는 타입으로 명시할 뿐입니다. 그리고 그 명시 덕분에, 이 데이터가 **단일 스레드 공유 가변 상태**라는 사실이 타입 시그니처에 그대로 드러납니다 — 동료 리뷰어가 `Rc<RefCell<T>>`를 보는 순간 "아, 공유되고 수정되는 단일 스레드 데이터구나, 멀티스레드로 넘기면 안 되겠네"라고 즉시 파악합니다.

중첩의 순서에도 의미가 있습니다. `Rc<RefCell<T>>`는 "여러 소유자가 공유하는 하나의 가변 셀"입니다 — 모든 클론이 같은 `RefCell`을 보므로, 한 곳에서 수정하면 모두가 그 변경을 봅니다(위 예제의 30). 반대로 `RefCell<Rc<T>>`는 "교체 가능한 공유 포인터"로, 의미가 완전히 다릅니다 — 셀 안에 든 `Rc`를 다른 `Rc`로 갈아 끼울 수 있지만, 가리키는 데이터 자체는 불변입니다. 파이썬에는 이 구분이 없습니다(모든 게 `Rc<RefCell>`처럼 동작). Rust에서는 "무엇을 공유하고 무엇을 수정하는가"를 타입 중첩 순서로 정밀하게 표현합니다.

이 패턴의 전형적인 실무 용도는 옵저버(Observer) 패턴입니다. 하나의 상태(subject)를 여러 관찰자(observer)가 공유하며 구독하는 구조 — 파이썬에서 콜백 리스트로 흔히 짜던 그 패턴이 Rust에서는 `Rc<RefCell<Subject>>`를 옵저버들이 나눠 갖는 형태가 됩니다. GUI 상태 관리, 이벤트 버스, 캐시 무효화 알림 등이 모두 여기에 해당합니다.

여기까지가 파이썬 객체의 절반입니다. 나머지 절반 — 순환 참조 — 이 진짜 함정입니다.

---

## 20.7 `Weak<T>`: 순환 참조 누수를 끊다

`Rc`는 파이썬의 참조 카운팅을 그대로 가져왔고, 그렇다면 **파이썬의 약점도 그대로 가져왔습니다.** 순환 참조는 카운트가 절대 0이 되지 않아 메모리가 누수됩니다. 파이썬은 백그라운드 GC로 이를 청소하지만, Rust에는 GC가 없습니다. 누수된 메모리는 프로세스가 죽을 때까지 그대로 남습니다.

직접 누수를 만들어 봅시다. `Drop`을 구현해서 해제가 일어나는지 추적합니다.

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Node {
    value: i32,
    next: RefCell<Option<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("Dropping Node {}", self.value);
    }
}

fn main() {
    let a = Rc::new(Node { value: 1, next: RefCell::new(None) });
    let b = Rc::new(Node { value: 2, next: RefCell::new(None) });

    *a.next.borrow_mut() = Some(Rc::clone(&b)); // a -> b
    *b.next.borrow_mut() = Some(Rc::clone(&a)); // b -> a (순환 완성!)

    println!("a strong = {}", Rc::strong_count(&a)); // 2
    println!("b strong = {}", Rc::strong_count(&b)); // 2

    println!("main 종료 직전");
} // a, b가 스코프를 벗어나지만, 서로를 붙잡고 있어 카운트가 1로만 떨어짐 → Drop이 영원히 호출 안 됨
```

실행 결과:

```text
a strong = 2
b strong = 2
main 종료 직전
```

`"Dropping Node 1"`도 `"Dropping Node 2"`도 **출력되지 않습니다.** `main`이 끝나며 변수 `a`, `b`가 사라졌지만, 두 노드는 서로의 `next` 필드에서 상대를 `Rc::clone`으로 붙잡고 있습니다. `a`의 카운트는 2(변수 `a` + `b.next`)였다가 변수 `a`가 사라지면서 1로 떨어지고, 거기서 멈춥니다. `b`도 마찬가지. 둘 다 1로 멈춰 영원히 0이 되지 못하고, 소멸자는 호출되지 않으며, 힙 메모리는 프로세스 종료까지 누수됩니다. 파이썬에서 `gc.collect()` 없이 순환을 만든 것과 정확히 같은 상황입니다 — 다만 Rust는 청소해 줄 GC가 없습니다.

> 💡 **Under the Hood: Rc 순환참조가 누수되는 메커니즘**
>
> ```text
>   변수 a (사라짐)          변수 b (사라짐)
>        │                        │
>        ▼                        ▼
>   ┌─────────────┐          ┌─────────────┐
>   │ Node 1      │          │ Node 2      │
>   │ strong: 1 ◀─┼──────────┤ next ───────┼──┐
>   │ next ───────┼──────────┼─▶ strong:1  │  │
>   └─────────────┘          └─────────────┘◀─┘
> ```
>
> `Rc`의 `Drop`은 "내 카운트를 1 줄이고, 0이 되면 데이터를 해제한다"가 전부입니다. 누구도 다른 노드의 카운트를 들여다보거나 "이 둘은 서로만 붙잡고 있으니 같이 죽여도 된다"고 판단하지 않습니다. 파이썬의 순환 GC는 정확히 이 판단(도달 불가능한 순환 찾기)을 하려고 객체 그래프를 주기적으로 순회합니다. Rust는 그 런타임 비용을 거부하고, 대신 **설계 단계에서 순환을 만들지 않을 책임을 개발자에게** 넘깁니다. 그 책임을 이행하는 도구가 `Weak<T>`입니다.

### `Weak<T>`: 소유하지 않는 참조

`Weak<T>`는 `Rc`와 같은 데이터를 가리키지만, **강한 카운트(strong count)를 증가시키지 않습니다.** 별도의 약한 카운트(weak count)만 올립니다. 핵심은 이것입니다 — **데이터의 생존은 오직 strong count로만 결정됩니다.** weak count는 데이터를 살려 두지 못합니다. 그래서 순환의 한쪽 방향을 `Weak`로 바꾸면, 그 방향은 더 이상 상대를 "붙잡지" 않게 되어 카운트가 정상적으로 0에 도달합니다.

`Weak`는 가리키는 데이터가 이미 죽었을 수도 있으므로, 직접 역참조할 수 없습니다. 대신 `upgrade()`를 호출해 `Option<Rc<T>>`를 받습니다 — 데이터가 살아 있으면 `Some(Rc)`(이때 잠깐 strong count 증가), 이미 해제됐으면 `None`. 파이썬의 `weakref.ref(obj)`와 정확히 같은 개념입니다.

```python
import weakref

class Node:
    pass

obj = Node()
ref = weakref.ref(obj)   # 약한 참조, refcount 증가 안 함
print(ref())             # <Node object> — 살아있음
del obj
print(ref())             # None — 대상이 해제됨
```

파이썬의 `weakref.ref(obj)`가 `Rc::downgrade(&rc)`에, 약한 참조를 다시 호출해 대상을 얻는 `ref()`가 `weak.upgrade()`에 대응합니다. 차이는 단 하나 — 파이썬의 `ref()`는 죽은 대상에 대해 `None`을 "조용히" 돌려주지만, Rust의 `upgrade()`는 `Option<Rc<T>>`라는 타입으로 "이 참조는 죽었을 수도 있다"는 사실을 강제로 인식하게 만듭니다. `if let Some(parent) = weak.upgrade()`로 명시적으로 살아있음을 확인하지 않으면 데이터에 접근조차 못 합니다. 파이썬에서 `ref()`의 결과가 `None`인지 깜빡 확인 안 해 터지던 `AttributeError`가, Rust에서는 컴파일 타임에 차단됩니다.

`Weak`가 `RcBox`의 별도 카운터(`weak`, 20.4절 레이아웃 참고)를 쓴다는 점도 중요합니다. `strong`이 0이 되면 데이터(`value`)는 해제되지만, `weak`가 0이 아니면 `RcBox` **블록 자체는 해제되지 않습니다.** 카운터를 담은 메모리가 남아 있어야, 살아있는 `Weak`들이 `upgrade()` 시 "strong이 이미 0이다"를 안전하게 읽을 수 있기 때문입니다. 그래서 `Weak`는 데이터를 살리지는 못해도 작은 메타데이터 블록은 붙잡아 둡니다 — Use-After-Free를 원천 차단하는 정교한 설계입니다.

규칙은 명확합니다. **소유 방향(부모→자식)은 `Rc`, 역참조 방향(자식→부모)은 `Weak`.** 트리에서 부모가 죽으면 자식도 죽어야 하므로 부모가 자식을 강하게 소유하고, 자식이 부모를 가리키는 것은 단순 참조이므로 약하게 잡습니다. 이렇게 하면 순환이 끊깁니다.

---

## 20.8 실전: 파이썬 객체 그래프를 Rust로 재현하기 (Hands-on)

이제 배운 모든 것을 조합해, 파이썬에서 흔히 만드는 **양방향 트리** — 조직도(Org Chart) — 를 누수 없이 구현해 봅니다. 매니저는 부하 직원을 알고, 직원은 자기 매니저를 거슬러 올라갈 수 있어야 합니다.

* **The Python Way (양방향 트리)**
  ```python
  from __future__ import annotations

  class Employee:
      def __init__(self, name: str) -> None:
          self.name = name
          self.manager: Employee | None = None   # 자식 → 부모 (역참조)
          self.reports: list[Employee] = []       # 부모 → 자식 (소유)

      def hire(self, report: Employee) -> None:
          report.manager = self
          self.reports.append(report)

  ceo = Employee("Alice")
  lead = Employee("Bob")
  dev = Employee("Carol")
  ceo.hire(lead)
  lead.hire(dev)

  print(dev.manager.name)        # Alice의 부하 Bob
  # 주의: manager <-> reports 가 순환 참조 → CPython의 gc가 청소 담당
  ```

이 파이썬 코드는 `manager`와 `reports`가 서로를 가리키는 순환 구조라, `gc`에 의존합니다. Rust로 옮기면서 그 의존을 `Weak`로 명시적으로 끊어 줍니다.

* **The Rust Implementation (`Rc<RefCell>` + `Weak`)**
  ```rust
  use std::cell::RefCell;
  use std::rc::{Rc, Weak};

  #[derive(Debug)]
  struct Employee {
      name: String,
      manager: RefCell<Weak<Employee>>,      // 자식 → 부모: Weak (순환 차단)
      reports: RefCell<Vec<Rc<Employee>>>,   // 부모 → 자식: Rc (소유)
  }

  impl Employee {
      fn new(name: &str) -> Rc<Employee> {
          Rc::new(Employee {
              name: name.to_string(),
              manager: RefCell::new(Weak::new()), // 처음엔 매니저 없음
              reports: RefCell::new(Vec::new()),
          })
      }

      // self가 report를 고용: 양방향 링크를 건다
      fn hire(self: &Rc<Self>, report: &Rc<Employee>) {
          *report.manager.borrow_mut() = Rc::downgrade(self); // 약한 역참조
          self.reports.borrow_mut().push(Rc::clone(report));  // 강한 소유
      }

      // 매니저 이름을 안전하게 조회 (이미 죽었으면 "(없음)")
      fn manager_name(&self) -> String {
          self.manager
              .borrow()
              .upgrade()                          // Weak → Option<Rc>
              .map(|m| m.name.clone())
              .unwrap_or_else(|| "(없음)".to_string())
      }
  }

  fn main() {
      let ceo = Employee::new("Alice");
      let lead = Employee::new("Bob");
      let dev = Employee::new("Carol");

      ceo.hire(&lead);
      lead.hire(&dev);

      // 자식 → 부모로 거슬러 올라가기 (upgrade)
      println!("{}의 매니저: {}", dev.name, dev.manager_name());   // Carol → Bob
      println!("{}의 매니저: {}", lead.name, lead.manager_name()); // Bob → Alice
      println!("{}의 매니저: {}", ceo.name, ceo.manager_name());   // Alice → (없음)

      // strong/weak 카운트로 누수 없음을 증명
      println!("ceo:  strong={}, weak={}", Rc::strong_count(&ceo), Rc::weak_count(&ceo));
      println!("lead: strong={}, weak={}", Rc::strong_count(&lead), Rc::weak_count(&lead));

      // 부모 → 자식으로 내려가며 보고 라인 출력
      for r in ceo.reports.borrow().iter() {
          println!("{} -> {}", ceo.name, r.name);
          for rr in r.reports.borrow().iter() {
              println!("  {} -> {}", r.name, rr.name);
          }
      }
  }
  ```

실행 결과:

```text
Carol의 매니저: Bob
Bob의 매니저: Alice
Alice의 매니저: (없음)
ceo:  strong=1, weak=1
lead: strong=2, weak=1
Alice -> Bob
  Bob -> Carol
```

핵심을 읽어 봅시다. `lead`의 `strong=2`인 이유는 — 변수 `lead` 자신(1) + `ceo.reports`에 들어간 `Rc::clone`(1)입니다. 그리고 `weak=1`은 `dev.manager`가 `lead`를 `Weak`로 가리키기 때문이죠. `ceo`는 누구의 `reports`에도 들어가지 않으므로 `strong=1`이고, `lead.manager`가 약하게 가리켜 `weak=1`입니다.

결정적으로, 이 구조에는 **강한 순환이 없습니다.** 부모→자식은 `Rc`(강함)지만 자식→부모는 `Weak`(약함)이므로, 강한 참조만 따라가면 절대 출발점으로 돌아오지 않습니다. `main`이 끝나면 `dev`(strong 1→0)가 먼저 해제되고, 그러면 `lead.reports`가 비면서 `lead`(strong 2→1→0)가 해제되고, 연쇄적으로 `ceo`까지 깔끔하게 해제됩니다. GC 없이, 누수 없이, 결정론적으로. `self: &Rc<Self>`라는 시그니처(Chapter 9에서 본 `self` 타입 지정)를 써서 메서드 안에서 `Rc::downgrade(self)`로 자기 자신의 약한 참조를 만든 점도 눈여겨보세요.

> 💡 이 패턴은 실무에서 그래프 알고리즘, 인메모리 트리(파일 시스템 표현, GUI 위젯 트리), 옵저버 패턴, 캐시 의존성 그래프 등에 그대로 쓰입니다. 다만 한 가지 — 이렇게 `Rc<RefCell>`로 그래프를 직접 손으로 짜기 전에, **인덱스 기반 접근**(노드를 `Vec`에 넣고 `usize` 인덱스로 참조)을 먼저 고려하세요. `petgraph` 크레이트가 정확히 이 방식으로 그래프를 안전하고 빠르게 다룹니다. `Rc<RefCell>` 그래프는 가장 직관적이지만, 노드마다 힙 할당과 카운트 관리 비용이 들고 캐시 지역성(cache locality)도 나쁘므로, 성능이 중요하면 인덱스 기반(또는 `slotmap`, `generational-arena` 같은 아레나 할당자)이 보통 더 낫습니다.

> 실무 생태계 한눈에: 트리/그래프는 `petgraph`·`slotmap`·`generational-arena`, 멀티스레드 공유 가변성은 `parking_lot`(표준 `Mutex`보다 빠른 락)·`dashmap`(샤딩된 동시성 해시맵), 단일 스레드 핫 패스의 가변 셀은 표준 `Cell`/`RefCell`로 충분합니다. `Cow`는 `serde`·`std::path`·`regex` 등 표준에 가까운 곳곳에서 이미 여러분이 모르는 사이 쓰이고 있습니다.

---

## 20.9 `Cow<T>`: 복사를 마지막 순간까지 미루기

스마트 포인터 가족 중 실무에서 의외로 자주 쓰이는데 덜 알려진 것이 `Cow<T>`(Clone on Write)입니다. 이름 그대로, **수정이 일어나기 전까지는 빌린 데이터를 그대로 쓰다가, 수정하는 그 순간에만 복사(clone)**하는 영리한 타입입니다.

파이썬 개발자에게 익숙한 시나리오로 설명해 봅시다. 문자열을 정규화하는 함수를 생각해 보세요. 입력의 99%는 이미 정규화되어 있어서 바꿀 게 없습니다. 그런데 파이썬에서 `s.replace(' ', '_')`를 호출하면, 바꿀 게 없어도 **항상 새 문자열 객체를 할당**합니다. 입력을 그대로 돌려줘도 되는데 말이죠.

```rust
use std::borrow::Cow;

// 공백이 없으면 빌린 그대로 반환(할당 0), 있으면 그때만 새 String 할당
fn normalize<'a>(input: &'a str) -> Cow<'a, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_")) // 수정 필요 → 소유(할당 1회)
    } else {
        Cow::Borrowed(input)                 // 수정 불필요 → 빌림(할당 0)
    }
}

fn main() {
    let a = normalize("hello");        // 빌림 — 힙 할당 없음
    let b = normalize("hello world");  // 소유 — 새 String 1회 할당

    println!("{a} / {b}");                                  // hello / hello_world
    println!("a borrowed? {}", matches!(a, Cow::Borrowed(_))); // true
    println!("b owned?    {}", matches!(b, Cow::Owned(_)));    // true
}
```

`Cow<str>`는 내부적으로 `Borrowed(&str)` 또는 `Owned(String)` 중 하나인 열거형(enum)입니다. 호출자는 둘 중 무엇인지 신경 쓸 필요 없이 `&str`처럼 읽을 수 있고(둘 다 `Deref<Target = str>`), 정말로 수정이 필요할 때만 `to_mut()`으로 소유권을 확보합니다. 입력의 대부분이 수정 불필요한 핫 패스(hot path)에서 이 패턴은 불필요한 할당을 통째로 제거합니다 — `serde`의 역직렬화, 경로 정규화(`std::path`), 설정 파싱 등 표준 라이브러리와 생태계 곳곳에서 쓰입니다. 파이썬이라면 매번 새 객체를 할당하고 GC에게 청소를 떠넘겼을 자리에서, Rust는 "정말 필요할 때만 할당"이라는 선택지를 줍니다.

---

## 20.10 Best Practices & 안티패턴

스마트 포인터는 강력하지만, 파이썬 습관 그대로 쓰면 오히려 코드를 망칩니다. 실무 체크리스트입니다.

**이렇게 하라:**

- **기본은 스택과 단일 소유권.** `&T`, `&mut T`로 풀 수 있으면 스마트 포인터를 쓰지 마세요. 스마트 포인터는 빌림 검사기로 표현 불가능한 구조에만 꺼내는 도구입니다.
- **선택은 능력 기준으로.** "힙이 필요한가?" → `Box`. "공유가 필요한가?" → `Rc`/`Arc`. "공유 + 수정?" → `Rc<RefCell>`/`Arc<Mutex>`. 필요 없는 능력에 비용을 지불하지 마세요.
- **`Rc::clone(&x)`를 명시적으로 쓰세요.** `x.clone()`이 아니라. 깊은 복사가 아니라 카운트 증가임을 코드에서 드러냅니다. clippy의 `clippy::clone_on_ref_ptr` 린트가 이를 권장합니다.
- **순환 가능성이 있으면 설계 단계에서 방향을 정하세요.** 소유 방향은 `Rc`, 역참조 방향은 `Weak`. 트리/그래프를 만들기 전에 "어느 쪽이 진짜 주인인가"를 먼저 결정합니다.
- **`RefCell`의 대여 가드 수명을 짧게.** `borrow_mut()` 결과를 변수에 오래 들고 있지 말고, 한 줄에서 수정하고 즉시 가드를 떨어뜨리세요. 패닉의 원인 대부분이 의도치 않게 길어진 가드입니다.
- **패닉을 피하려면 `try_borrow`/`try_borrow_mut`.** 대여 실패를 정상 흐름으로 다뤄야 하는 곳에서는 `Result`를 반환하는 변형을 쓰세요.
- **그래프는 인덱스 기반을 먼저 고려.** `Vec<Node>` + `usize` 인덱스(혹은 `petgraph`)가 `Rc<RefCell>`보다 빠르고 단순한 경우가 많습니다.

**이건 피하라:**

- **`Rc<RefCell<T>>`를 만능 망치로 쓰지 마세요.** 이것이 가장 흔한 안티패턴입니다. 파이썬 객체가 그리워서 모든 필드를 `Rc<RefCell>`로 감싸기 시작하면, 코드는 런타임 패닉 지뢰밭이 되고 성능은 파이썬에 가까워집니다. 컴파일 타임 안전성이라는 Rust의 핵심 장점을 스스로 버리는 셈입니다.
- **`RefCell`을 데이터 흐름을 게을리 설계한 변명으로 쓰지 마세요.** 소유권을 제대로 모델링하면 대부분 `RefCell` 없이 풀립니다.
- **`Box`를 단순 값 전달에 남용하지 마세요.** 작은 값은 그냥 스택에 두고 이동시키는 게 빠릅니다. `Box`는 힙 할당 비용이 있습니다.

---

## 20.11 흔한 함정 (Python 개발자가 자주 하는 실수)

### 함정 1: `Rc`를 멀티스레드에 쓰려다 컴파일 실패

파이썬의 객체는 GIL 아래에서 어느 스레드에서나 그냥 공유됩니다. 그래서 `Rc`로 만든 공유 데이터를 무심코 `thread::spawn`에 넘기는 실수가 흔합니다.

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(42);
    let data2 = Rc::clone(&data);
    thread::spawn(move || {           // 🚨 컴파일 에러!
        println!("{}", data2);
    });
}
```

```text
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs:7:19
    |
7   |       thread::spawn(move || {
    |       ------------- ^---
    |       |             |
    |       |             `Rc<i32>` cannot be sent between threads safely
    |  _____|
    | |     required by a bound introduced by this call
... |
    = help: within `{closure...}`, the trait `Send` is not implemented for `Rc<i32>`
```

`Rc`의 카운트 증감은 원자적이 아니므로(20.4절 Under the Hood), 두 스레드가 동시에 `clone`/`drop`하면 카운트가 깨집니다. Rust는 이를 `Send` 트레이트(Chapter 17)로 컴파일 타임에 차단합니다 — `Rc`는 `Send`가 아니라서 스레드 경계를 넘지 못합니다. **해결:** 멀티스레드라면 `Arc`로 교체하세요. `Rc<RefCell<T>>` → `Arc<Mutex<T>>`가 정석 변환입니다. 파이썬과 달리 Rust는 "이 데이터는 단일 스레드 전용"이라는 사실을 타입에 새겨, 사고를 컴파일 단계에서 막아 줍니다.

### 함정 2: `RefCell` 이중 가변 대여로 런타임 패닉

20.5절에서 본 그대로입니다. 파이썬에서는 같은 객체를 여러 변수가 들고 동시에 수정해도 아무 문제가 없습니다. 그 습관으로 `RefCell`을 쓰면, 특히 재귀 호출이나 콜백 안에서 이미 대여 중인 셀을 다시 대여하려다 패닉합니다.

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(vec![1, 2, 3]);
    let guard = cell.borrow_mut();        // 가드가 살아있음
    // ... 긴 로직 ...
    cell.borrow_mut().push(4);            // 🚨 panic: 위 guard가 아직 안 죽음
    drop(guard);
}
```

컴파일은 통과하지만 `RefCell already borrowed`로 뻗습니다. **해결:** 대여 가드의 수명을 최소화하세요. `borrow_mut()`을 변수에 묶어 오래 들고 있지 말고, 필요한 수정만 한 줄에서 끝내 가드를 즉시 떨어뜨립니다. 위 경우 `guard`를 미리 `drop`하거나, 애초에 두 번째 대여 전에 첫 가드의 스코프가 끝나도록 코드를 재구성합니다.

### 함정 3: 순환 참조로 조용한 메모리 누수

20.7절에서 증명했듯, `Rc`만으로 부모↔자식 양방향 링크를 만들면 GC 없는 Rust에서는 메모리가 영원히 새어 나갑니다. 파이썬에서는 `gc`가 알아서 청소하므로 이 문제를 의식조차 안 해 본 개발자가 많습니다. **해결:** 양방향 링크에서 한쪽(보통 자식→부모, 또는 옵저버→subject)을 반드시 `Weak`로 만드세요. "이 참조가 데이터의 생존을 책임져야 하는가?"를 자문하고, 책임지지 않아도 되는 쪽은 `Weak`입니다. 누수가 의심되면 `Rc::strong_count`로 카운트가 예상대로 0에 수렴하는지 검증하고, `Drop`을 임시로 구현해 해제 로그를 찍어 확인하세요(20.7절 예제처럼).

### 함정 4: `RefCell`을 멀티스레드로

`RefCell`은 `Sync`가 아닙니다. `Arc<RefCell<T>>`로 멀티스레드에서 공유하려 하면 컴파일이 거부합니다. 멀티스레드 내부 가변성은 `Mutex`/`RwLock`이 담당합니다 — `Arc<Mutex<T>>`(Chapter 17). 단일 스레드의 `Rc<RefCell<T>>`와 멀티스레드의 `Arc<Mutex<T>>`는 정확히 대응하는 쌍이라고 기억하세요.

---

지금까지 우리는 파이썬 인터프리터가 모든 객체에 자동으로 부여하던 능력 — 힙 할당, 참조 카운팅, 무제한 가변성, 순환 허용 — 을 Rust의 네 가지 명시적 부품(`Box`, `Rc`, `RefCell`, `Weak`)으로 분해했다가 다시 조립했습니다. 파이썬 객체 하나가 사실은 `Rc<RefCell<T>>`였고, 순환을 청소하던 GC의 자리에 `Weak`라는 설계 책임이 들어선다는 것을 확인했습니다. 그 대가로 우리는 백그라운드 GC 없는 결정론적 메모리 해제와, 대부분의 실수를 컴파일 타임에 잡는 안전망을 얻었습니다.

이로써 Part 7의 고급 언어 탐험을 마칩니다. 우리는 매크로로 메타프로그래밍을, 스마트 포인터로 메모리 그래프를 정복했습니다. 그런데 지금까지 다룬 모든 것은 Rust라는 안전한 성벽 **안쪽**의 이야기였습니다. 다음 장 **Chapter 21: C FFI와 Unsafe Rust**에서는 그 성벽 밖으로 나갑니다 — 컴파일러의 보증이 닿지 않는 영역에서 C 라이브러리와 손을 잡고, `unsafe` 키워드로 직접 포인터를 다루며, 그 위험을 통제하는 법을 배웁니다. 이것이 파이썬과 Rust가 물리적으로 결합하는 다리(Chapter 22의 PyO3)를 놓기 위한 마지막 준비입니다.
