# Chapter 14. 소유권을 가진 스마트 포인터 컬렉션

파이썬 개발자에게 자료구조란 숨 쉬는 것과 같습니다. `list`, `dict`, `set`, 그리고 `str`은 어떤 타입의 데이터든 집어넣고 길이를 늘이거나 줄일 수 있는 마법의 주머니입니다.

하지만 이 마법은 철저히 런타임 힙(Heap) 메모리와 동적 타입(Dynamic Type)에 의존합니다. 파이썬의 리스트는 실제 데이터 배열이 아니라 "메모리 곳곳에 흩어진 객체들의 주소(포인터)를 모아놓은 배열"입니다.

Rust는 정적 타입 언어이면서도 파이썬 못지않게 강력한 가변 길이 자료구조를 제공합니다. 이를 컬렉션(Collections)이라고 부르며, 대표적으로 `Vec`, `String`, `HashMap`이 있습니다. 이 챕터에서는 파이썬의 자료구조와 러스트의 컬렉션이 메모리 상에서 어떻게 다른지 비교합니다.

---

## 14.1 `list` vs `Vec<T>`

파이썬의 리스트는 아무 타입이나 다 담을 수 있습니다.
```python
my_list = [1, "two", 3.0] # int, str, float가 섞여도 문제없음
```

하지만 Rust의 `Vec<T>`(벡터)는 제네릭(Generic) 타입 `T`를 가지며, **오직 단 한 가지 타입의 데이터만** 담을 수 있습니다.

* **The Rust Implementation**
  ```rust
  fn main() {
      // 컴파일러가 매크로 안의 값들을 보고 i32 타입의 Vec임을 추론합니다.
      let mut numbers = vec![1, 2, 3]; 

      // numbers.push("four"); // 🚨 컴파일 에러! i32만 넣을 수 있습니다.
      numbers.push(4);
      
      // 인덱스 접근
      println!("The third element is {}", numbers[2]);
  }
  ```

만약 파이썬처럼 정수와 문자열을 섞어 넣고 싶다면 어떻게 해야 할까요? 바로 앞서 Chapter 10에서 배운 **Enum**을 사용하면 됩니다.

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}

let row = vec![
    SpreadsheetCell::Int(3),
    SpreadsheetCell::Text(String::from("blue")),
    SpreadsheetCell::Float(10.12),
];
// 컴파일 타임에 모든 타입의 크기가 확정되므로 메모리 정렬이 최적화됩니다.
```

---

## 14.2 `str` vs `String` / `&str`

파이썬에서 문자열은 불변(Immutable) 객체 하나(`str`)로 끝납니다. 하지만 Rust에서는 문자열을 표현하는 방식이 크게 두 가지로 나뉘며, 파이썬 개발자들이 가장 많이 헷갈려하는 부분입니다.

1. **`String`**: 길이를 변경할 수 있고(growable), 내용 변경도 가능하며, **Heap 메모리에 할당된 텍스트의 "소유권"을 가진 스마트 포인터**입니다.
2. **`&str` (String Slice)**: 텍스트 데이터의 특정 부분만 가리키는 **참조(Reference, 대여)**입니다. 스택, 힙, 심지어 실행 파일 내부(바이너리) 등 데이터가 어디에 있든 가리킬 수 있습니다.

* **The Rust Implementation**
  ```rust
  fn main() {
      // 1. 바이너리 실행 파일 어딘가에 하드코딩된 문자열을 가리키는 참조 (&str)
      // 소유권이 없으므로 길이를 늘리거나 수정할 수 없습니다.
      let greeting: &str = "Hello, Pythonistas!";

      // 2. Heap에 새롭게 할당한 소유권을 가진 문자열 (String)
      let mut s = String::from("Hello");
      s.push_str(", world!"); // 내용을 변경(Append)할 수 있습니다.

      // 3. String의 일부만 떼어내서 &str로 참조하기 (Slice)
      let hello: &str = &s[0..5];
  }
  ```

### 파이썬의 인덱싱과 Rust의 UTF-8 슬라이싱
파이썬은 문자열의 길이를 계산하거나 인덱스로 접근할 때 항상 글자(Character) 단위를 기준으로 합니다.
```python
text = "안녕하세요"
print(text[0]) # "안"
```

하지만 Rust의 문자열은 내부에 **UTF-8 인코딩된 바이트(Byte) 배열**로 저장됩니다. 한글은 한 글자당 3바이트를 차지합니다.

```rust
let text = String::from("안녕하세요");

// let first_char = text[0]; // 🚨 컴파일 에러! 바이트 인덱싱을 직접 허용하지 않습니다.
// let slice = &text[0..2]; // 🚨 런타임 패닉! 한글은 3바이트인데 2바이트만 자르면 글자가 깨집니다.

// 올바른 접근법: 글자(char) 단위 이터레이터를 사용해야 합니다.
let first_char = text.chars().nth(0).unwrap(); // '안'
```
파이썬은 사용자를 위해 뒤에서 몰래 바이트 크기를 계산해 주지만, Rust는 O(1) 시간 복잡도를 보장할 수 없는 연산(UTF-8 글자 단위 슬라이싱)을 개발자 모르게 수행하지 않습니다. 

---

## 14.3 `dict` vs `HashMap<K, V>`

파이썬의 가장 강력한 무기인 딕셔너리는 내부적으로 해시 맵입니다. Rust도 동일한 자료구조인 `HashMap`을 제공합니다.

* **The Python Way**
  ```python
  scores = {"Blue": 10, "Yellow": 50}
  
  # 값을 찾을 때
  blue_score = scores.get("Blue", 0)
  
  # 값을 덮어쓰거나 추가할 때
  scores["Yellow"] = 60
  scores.setdefault("Red", 30) # Red가 없으면 30을 넣음
  ```

* **The Rust Implementation**
  ```rust
  use std::collections::HashMap;

  fn main() {
      let mut scores = HashMap::new();

      // String과 i32 타입을 넣으면서 소유권이 HashMap으로 넘어갑니다(Move).
      scores.insert(String::from("Blue"), 10);
      scores.insert(String::from("Yellow"), 50);

      // get()은 파이썬과 다르게 Option<&V> 를 반환합니다.
      // 값이 없을 수 있다는 사실(None)을 타입 시스템으로 강제합니다.
      let team_name = String::from("Blue");
      let blue_score = scores.get(&team_name).copied().unwrap_or(0);

      // 덮어쓰기
      scores.insert(String::from("Yellow"), 60);

      // setdefault의 Rust 버전: entry()와 or_insert()의 조합
      scores.entry(String::from("Red")).or_insert(30);

      println!("{:?}", scores);
  }
  ```

Rust의 컬렉션은 파이썬의 컬렉션보다 엄격합니다. 타입을 섞을 수 없고, 인덱싱에 주의해야 하며, 값을 넘길 때 항상 소유권(Move)과 참조(Borrow)를 신경 써야 합니다. 

하지만 이 약간의 엄격함을 대가로, 우리는 파이썬의 백그라운드에서 끊임없이 돌아가는 Reference Counting 연산과 파편화된 메모리 접근 오버헤드를 완전히 걷어낼 수 있습니다. 

다음 장에서는 컬렉션 안에 들어있는 데이터를 우아하고, 그리고 "Loop 문보다도 빠르게" 처리하는 **이터레이터(Iterator)와 클로저(Closure)**에 대해 알아보겠습니다.
