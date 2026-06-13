# Chapter 14. 소유권을 가진 스마트 포인터 컬렉션

파이썬 개발자에게 자료구조란 숨 쉬는 것과 같습니다. `list`, `dict`, `set`, 그리고 `str`은 어떤 타입의 데이터든 집어넣고 길이를 늘이거나 줄일 수 있는 마법의 주머니입니다. `data = []` 한 줄이면 끝이고, `data.append(...)`로 무한히 늘리며, `data[0]`으로 즉시 꺼냅니다. 우리는 이 주머니 안에서 무슨 일이 벌어지는지 거의 생각하지 않습니다.

하지만 10년 차 엔지니어라면 이 편의의 청구서를 본 적이 있을 겁니다. 1억 개의 정수를 담은 `list`가 RAM을 4GB씩 잡아먹어 OOM Killer에게 살해당하던 새벽. `numpy`를 도입하고 나서야 메모리가 1/5로 줄고 순회 속도가 50배 빨라졌던 경험. 혹은 사용자 입력으로 받은 키를 `dict`에 잔뜩 넣었더니 해시 충돌 공격(Hash-Flooding DoS)으로 서버가 멈췄던 사건. 이 모든 고통의 뿌리에는 동일한 사실이 있습니다. **파이썬의 컬렉션은 데이터 자체를 담지 않습니다. 데이터를 가리키는 포인터들을 담습니다.**

파이썬의 리스트는 실제 데이터 배열이 아니라 "메모리 곳곳에 흩어진 객체들의 주소(포인터)를 모아놓은 배열"입니다. `[1, 2, 3]`은 메모리상에서 정수 `1`, `2`, `3`이 나란히 붙어 있는 8바이트 × 3짜리 버퍼가 아닙니다. 그것은 힙(Heap) 여기저기에 흩어진 세 개의 정수 객체를 가리키는 세 개의 포인터 배열입니다. CPU가 이 리스트를 순회하려면 매번 포인터를 따라 캐시 밖으로 점프(pointer chasing)해야 하고, 이것이 캐시 미스(cache miss)의 폭풍을 일으킵니다.

Rust는 정적 타입 언어이면서도 파이썬 못지않게 강력한 가변 길이 자료구조를 제공합니다. 이를 컬렉션(Collections)이라고 부르며, 대표적으로 `Vec<T>`, `String`, `HashMap<K, V>`가 있습니다. 이들의 공통점은 Chapter 4에서 배운 **소유권(Ownership)**과 Chapter 20에서 자세히 다룰 **스마트 포인터(Smart Pointer)**의 결합체라는 것입니다. 셋 모두 힙에 데이터를 할당하고, 그 데이터의 소유권을 가지며, 스코프를 벗어나면 자동으로 메모리를 해제합니다. 가비지 컬렉터도, 참조 카운터도 없이.

이번 장에서는 파이썬의 자료구조와 러스트의 컬렉션이 메모리 상에서 어떻게 다른지를 바이트 단위로 해부합니다. `Vec`의 3워드 헤더와 증폭 성장(amortized doubling), `HashMap`이 왜 기본적으로 느린 해시 함수를 쓰는지 그리고 어떻게 교체하는지, `String`이 왜 `s[0]` 인덱싱을 컴파일조차 거부하는지를 추적합니다. 그 과정에서 실제 `rustc`가 내뱉는 패닉과 컴파일 에러를 직접 마주하고, 파이썬과의 현실적인 성능 격차를 측정합니다.

---

## 14.1 The Python Way — 포인터 배열이라는 진실

먼저 우리가 매일 쓰는 도구의 내부를 정직하게 들여다봅시다. `[1, "two", 3.0]`처럼 타입을 섞을 수 있는 리스트의 유연함이 어디서 나오는지가 핵심입니다.

```python
import sys

my_list: list = [1, "two", 3.0]   # int, str, float가 섞여도 문제없음
print(sys.getsizeof(my_list))      # 80 (포인터 3개 + 헤더 + 여유 슬롯)
print(sys.getsizeof(1))            # 28 (정수 객체 하나)
```

이 유연함의 비밀은 `my_list`가 정수 `1`, 문자열 `"two"`, 실수 `3.0`을 **직접 담지 않는다**는 데 있습니다. 리스트는 세 개의 `PyObject*` 포인터를 담고, 실제 값들은 힙 어딘가에 따로 흩어진 별도의 객체로 존재합니다. 모든 값이 동일한 크기(포인터 8바이트)의 박스에 들어가 있으므로 타입이 섞여도 상관없습니다. 이것이 파이썬 리스트가 이종(heterogeneous) 데이터를 담을 수 있는 메커니즘입니다.

`list.append()`의 성장 방식도 살펴봅시다. CPython의 리스트는 내부 버퍼가 가득 차면 더 큰 버퍼를 새로 할당하고 기존 포인터들을 복사합니다. 성장 패턴은 대략 `0, 4, 8, 16, 25, 35, ...`로, 약 1.125배씩 늘어납니다. 이 덕분에 `append`는 평균적으로 O(1)이지만(amortized), 재할당이 일어나는 순간만큼은 O(n)입니다. 어디서 많이 본 패턴이지요? 곧 Rust의 `Vec`에서 똑같은 원리를 보게 됩니다.

```python
# dict: 파이썬의 진짜 주력 무기
scores: dict[str, int] = {"Blue": 10, "Yellow": 50}
blue = scores.get("Blue", 0)          # 키가 없으면 기본값
scores["Yellow"] = 60                 # 덮어쓰기
scores.setdefault("Red", 30)          # 없을 때만 삽입

from collections import defaultdict
counts: defaultdict[str, int] = defaultdict(int)
for word in "the cat the dog the".split():
    counts[word] += 1                 # 키가 없어도 0에서 시작
```

`dict` 역시 키와 값 모두 포인터로 저장합니다. Python 3.6부터 dict는 삽입 순서를 보존하는데, 이는 내부적으로 "조밀한 엔트리 배열 + 희소한 인덱스 배열"이라는 두 단계 구조를 쓰기 때문입니다. 그리고 모든 키는 `hash()`로 해싱되며, 충돌 시 오픈 어드레싱(open addressing)으로 다음 슬롯을 탐색합니다.

이 모든 편의에는 두 가지 본질적 비용이 따릅니다.

첫째, **메모리 팽창**입니다. 1억 개의 64비트 정수를 담는다고 합시다. 데이터 자체는 800MB면 충분합니다. 그러나 파이썬 `list`는 포인터 배열(800MB) *더하기* 흩어진 정수 객체들(작은 정수 캐싱을 벗어나면 객체당 28바이트 × 1억 = 2.8GB)을 차지합니다. 합쳐서 3.6GB. Rust의 `Vec<i64>`는 정확히 800MB입니다.

둘째, **캐시 지역성(Cache Locality)의 붕괴**입니다. CPU가 리스트를 순회하며 합을 구한다고 합시다. 각 원소를 읽으려면 포인터를 따라 힙의 다른 위치로 점프해야 합니다. 이 객체들은 생성 시점이 제각각이라 메모리에 흩어져 있으므로, 거의 매번 L1/L2 캐시를 빗나갑니다(cache miss). 메인 메모리 접근은 캐시 접근보다 100배 가까이 느립니다. 데이터가 연속으로 붙어 있을 때 CPU가 누리는 프리페치(prefetch)와 SIMD 벡터화의 혜택을 전혀 받지 못합니다.

> **Under the Hood: 파이썬 `list`의 메모리 레이아웃**
>
> `[10, 20, 30]`은 메모리상에서 이렇게 생겼습니다. 리스트 객체 본체는 포인터 배열을 가리키고, 그 배열의 각 슬롯은 다시 힙에 흩어진 정수 객체를 가리킵니다.
>
> ```text
> list 객체              ob_item (포인터 배열, 힙)
> ┌──────────────┐      ┌──────┬──────┬──────┬─ (여유) ─┐
> │ ob_refcnt    │      │ ptr0 │ ptr1 │ ptr2 │   ...    │
> │ ob_type      │      └──┬───┴──┬───┴──┬───┴──────────┘
> │ ob_size = 3  │         │      │      │
> │ allocated=4  │         ▼      ▼      ▼
> │ ob_item ─────┼──►   [int 10][int 20][int 30]  ← 힙 곳곳에 흩어짐
> └──────────────┘       28byte  28byte  28byte    각각 refcnt+type 헤더 포함
> ```
>
> 핵심은 두 단계의 포인터 추적(double indirection)입니다. 값 하나를 읽으려면 ① 포인터 배열에서 슬롯을 읽고 ② 그 포인터가 가리키는 객체로 점프해 ③ 객체 헤더를 건너뛴 뒤 실제 값을 읽습니다. 그리고 ③의 점프 목적지가 캐시 라인마다 제각각이라는 것이 순회 성능을 무너뜨리는 결정적 원인입니다.

---

## 14.2 The Rust Mental Model — 컬렉션은 소유권을 가진 연속 버퍼다

Rust의 사고방식은 단순합니다. **"같은 타입의 값들을 메모리에 연속으로 붙여 놓고, 그 버퍼 전체의 소유권을 하나의 변수에 맡긴다."**

여기서 "같은 타입"이 첫 번째 전환점입니다. `Vec<T>`는 제네릭(Generic) 타입 `T`를 가지며, 오직 단 한 가지 타입의 데이터만 담습니다. 컴파일러가 `T`의 정확한 크기를 컴파일 타임에 알기 때문에, 원소 100개짜리 `Vec<i64>`는 `8바이트 × 100 = 800바이트`의 연속된 버퍼 하나입니다. 포인터 배열도, 흩어진 객체도, 객체 헤더도 없습니다. CPU가 이 버퍼를 순회하면 한 캐시 라인(보통 64바이트)을 읽을 때마다 8개의 정수가 한꺼번에 딸려 옵니다. 프리페처가 다음 라인을 미리 당겨오고, 자동 벡터화(auto-vectorization)가 작동합니다.

"소유권"이 두 번째 전환점입니다. `Vec<T>`, `String`, `HashMap<K, V>`은 모두 Chapter 20에서 본격적으로 다룰 **스마트 포인터**입니다. 즉 힙에 버퍼를 할당하고, 그 버퍼를 가리키는 포인터와 메타데이터를 스택에 두며, 자신이 스코프를 벗어나는 순간 `Drop` 트레이트를 통해 힙 버퍼를 자동으로 해제합니다. 이것이 Chapter 4의 소유권 규칙이 컬렉션에 그대로 적용되는 방식입니다. `let v = vec![1, 2, 3];`에서 `v`가 스코프를 벗어나면, 컴파일러가 삽입한 해제 코드가 정확히 그 지점에서 실행됩니다. GC도 refcount도 없습니다.

세 번째 전환점은 **엄격함**입니다. 파이썬이 뒤에서 몰래 처리해 주던 일들을 Rust는 타입 시스템으로 드러냅니다. 인덱스가 범위를 벗어날 수 있다는 사실, 키가 존재하지 않을 수 있다는 사실, 문자열 슬라이싱이 글자 경계를 깰 수 있다는 사실. Rust는 이 위험들을 `Option`, 경계 검사(bounds check), 그리고 컴파일 에러로 표면화합니다.

| 파이썬 개념 | Rust 대응 | 핵심 차이 |
|---|---|---|
| `list` (이종 타입) | `Vec<T>` (단일 타입) | Vec은 연속 버퍼, list는 포인터 배열 |
| `list[int]` (혼합) | `Vec<Enum>` | 혼합 타입은 Enum으로 명시적 표현 |
| `dict` | `HashMap<K, V>` | HashMap 기본 해셔는 DoS 방어용 SipHash |
| `collections.OrderedDict` | `indexmap::IndexMap` | 삽입 순서 보존이 필요할 때 |
| `set` | `HashSet<T>` | 중복 없는 집합 |
| `collections.deque` | `VecDeque<T>` | 양끝 O(1) 큐 |
| 정렬된 dict (`sorted()`) | `BTreeMap<K, V>` | 키 정렬 보존, 범위 쿼리 |
| `heapq` (최소 힙) | `BinaryHeap<Reverse<T>>` | BinaryHeap은 기본이 최대 힙 |
| `str` (불변, 글자 인덱싱) | `String` + `&str` (UTF-8 바이트) | 소유/대여 분리, 바이트 인덱싱 불가 |
| `str.encode()` (bytes) | `&[u8]` / `Vec<u8>` | 바이트 슬라이스 |
| `d.setdefault(k, v)` | `map.entry(k).or_insert(v)` | Entry API |
| `defaultdict(int)` | `*map.entry(k).or_insert(0) += 1` | Entry API + 역참조 |

이제 각 컬렉션을 하나씩 깊이 파고듭니다.

---

## 14.3 `Vec<T>`: 3워드 헤더와 증폭 성장

`Vec<T>`는 Rust에서 가장 많이 쓰는 컬렉션입니다. 파이썬 리스트와 달리, 단일 타입만 담습니다.

```rust
fn main() {
    // 컴파일러가 vec! 매크로 안의 값들을 보고 i32 타입의 Vec임을 추론합니다.
    let mut numbers = vec![1, 2, 3];

    numbers.push(4);

    // 인덱스 접근 (경계 검사가 들어갑니다)
    println!("세 번째 원소: {}", numbers[2]);
}
```

타입을 섞으려 하면 어떻게 될까요? 파이썬에서는 자연스러운 `numbers.push("four")`가 Rust에서는 컴파일 단계에서 거부됩니다.

```rust
fn main() {
    let mut numbers = vec![1, 2, 3];
    numbers.push("four"); // 🚨 컴파일 에러!
}
```

실제 `rustc`의 출력입니다.

```text
error[E0308]: mismatched types
 --> src/main.rs:3:18
  |
3 |     numbers.push("four");
  |             ---- ^^^^^^ expected integer, found `&str`
  |             |
  |             arguments to this method are incorrect
  |
note: method defined here
 --> .../library/alloc/src/vec/mod.rs
  |
  |     pub fn push(&mut self, value: T) {
```

`expected integer, found &str` — 이 한 줄이 핵심입니다. `numbers`의 타입이 `Vec<i32>`로 확정된 순간, `push`의 인자 타입 `T`는 `i32`로 못 박혔습니다. 파이썬이라면 런타임에 무엇이든 받아들였겠지만, Rust는 컴파일 타임에 차단합니다.

그렇다면 정말로 정수와 문자열을 섞어야 한다면? Chapter 10에서 배운 **Enum**을 사용합니다. 이것이 파이썬의 "아무거나 담기"를 Rust답게 표현하는 방식입니다.

```rust
enum SpreadsheetCell {
    Int(i64),
    Float(f64),
    Text(String),
}

fn main() {
    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];

    for cell in &row {
        match cell {
            SpreadsheetCell::Int(i) => println!("정수: {}", i),
            SpreadsheetCell::Float(f) => println!("실수: {}", f),
            SpreadsheetCell::Text(s) => println!("문자열: {}", s),
        }
    }
}
```

차이가 보이나요? 파이썬의 혼합 리스트는 "런타임에 무엇이든 들어올 수 있고 무엇이 들어왔는지는 꺼내봐야 안다"입니다. Rust의 `Vec<SpreadsheetCell>`은 "들어올 수 있는 변형은 Int/Float/Text 셋뿐이고, `match`로 모든 경우를 다루지 않으면 컴파일되지 않는다"입니다. 가능성의 집합이 타입에 박제됩니다. 그리고 `Vec`의 각 슬롯은 가장 큰 변형(여기서는 `String`, 24바이트)에 태그 바이트를 더한 크기로 통일되며, 이 역시 연속 버퍼입니다.

### `with_capacity`와 증폭 성장

`Vec`이 어떻게 자라는지 직접 관찰해 봅시다. 다음 코드는 그대로 컴파일·실행됩니다.

```rust
fn main() {
    let mut v: Vec<i64> = Vec::new();
    let mut last_cap = v.capacity();
    println!("초기: len={} cap={}", v.len(), v.capacity());

    for i in 0..20 {
        v.push(i);
        if v.capacity() != last_cap {
            println!("재할당! len={} cap={}", v.len(), v.capacity());
            last_cap = v.capacity();
        }
    }
}
```

실제 출력입니다(rustc 1.96, `--release` 무관).

```text
초기: len=0 cap=0
재할당! len=1 cap=4
재할당! len=5 cap=8
재할당! len=9 cap=16
재할당! len=17 cap=32
```

용량이 `0 → 4 → 8 → 16 → 32`로, 가득 찰 때마다 정확히 두 배씩 늘어납니다. 이것이 **증폭 더블링(amortized doubling)**입니다. 매 `push`가 O(1)이 아니라, 재할당이 일어나는 순간만 O(n)이고 그 비용을 전체 `push` 횟수에 분산하면 평균 O(1)이 됩니다. 파이썬 리스트가 1.125배씩 자라는 것과 전략은 같지만, Rust는 2배 성장으로 재할당 횟수를 더 공격적으로 줄입니다.

재할당은 공짜가 아닙니다. 새 버퍼를 할당하고, 기존 원소를 전부 복사하고, 옛 버퍼를 해제합니다. 최종 크기를 미리 안다면 `with_capacity`로 단 한 번에 할당하세요.

```rust
fn main() {
    // 100만 개를 담을 것을 안다면 재할당을 0번으로
    let mut v: Vec<i64> = Vec::with_capacity(1_000_000);
    for i in 0..1_000_000 {
        v.push(i); // 재할당 없음. cap이 충분하므로 그냥 쓰기
    }
    println!("len={} cap={}", v.len(), v.capacity());
}
```

파이썬에서 `[0] * n`이나 리스트 컴프리헨션이 빈 리스트 `append` 반복보다 빠른 것과 동일한 원리입니다. 다만 Rust는 `with_capacity`라는 명시적 도구를 줍니다.

### 인덱싱: `[]` vs `get()`

파이썬의 `list[i]`는 범위를 벗어나면 `IndexError`를 던집니다. Rust에는 두 가지 길이 있고, 둘의 차이를 아는 것이 중요합니다.

```rust
fn main() {
    let v = vec![10, 20, 30];

    // 방법 1: [] — 범위를 벗어나면 패닉(panic)
    let x = v[10]; // 🚨 panic!
    println!("{}", x);
}
```

실행하면 프로그램이 즉시 죽습니다.

```text
thread 'main' panicked at src/main.rs:5:13:
index out of bounds: the len is 3 but the index is 10
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`index out of bounds: the len is 3 but the index is 10` — 파이썬의 `IndexError`에 해당합니다. 하지만 파이썬 예외는 `try-except`로 잡을 수 있는 반면, Rust의 패닉은 기본적으로 스레드를 종료시킵니다(잡으려면 `catch_unwind`라는 무거운 도구가 필요하며, 이는 정상 흐름 제어용이 아닙니다).

그래서 "값이 없을 수도 있다"가 정상 시나리오라면, `get()`을 써서 Chapter 12에서 배운 `Option`으로 받습니다.

```rust
fn main() {
    let v = vec![10, 20, 30];

    // 방법 2: get() — Option<&T> 반환, 패닉 없음
    match v.get(10) {
        Some(x) => println!("값: {}", x),
        None => println!("인덱스 10은 범위 밖입니다"),
    }

    // 흔한 관용구: 없으면 기본값 (파이썬의 list[i] if ... else default)
    let val = v.get(10).copied().unwrap_or(-1);
    println!("val = {}", val); // -1
}
```

규칙은 단순합니다. **"인덱스가 범위 안임이 논리적으로 보장될 때만 `[]`를 쓰고, 불확실하면 `get()`을 써라."** `[]`의 경계 검사는 매 접근마다 비용이 들지만(보통 분기 예측으로 거의 무료), 안전을 보장합니다. C++의 `vector::operator[]`가 검사 없이 미정의 동작(UB)을 일으키는 것과 대조적입니다. 성능이 정말 중요하고 안전을 직접 증명할 수 있는 핫 루프에서는 `get_unchecked`(unsafe)도 있지만, 이는 Chapter 21의 영역입니다.

> **Under the Hood: `Vec<T>`는 단 3워드다**
>
> `Vec<T>` 값 자체는 스택에 놓인 24바이트(64비트 기준 3워드)의 작은 구조체입니다. 실제 데이터는 힙에 있습니다.
>
> ```text
> 스택 (Vec 헤더, 24 bytes)        힙 (실제 버퍼)
> ┌──────────────────────┐       ┌────┬────┬────┬─(여유)─┐
> │ ptr      ────────────┼──────►│ 10 │ 20 │ 30 │  ???   │
> │ len      = 3         │       └────┴────┴────┴────────┘
> │ capacity = 4         │         8B   8B   8B   8B
> └──────────────────────┘
> ```
>
> - `ptr`: 힙 버퍼의 시작 주소
> - `len`: 현재 원소 개수 (인덱싱 경계 검사에 사용)
> - `capacity`: 재할당 없이 담을 수 있는 최대 개수
>
> 파이썬 리스트의 "포인터 배열 + 흩어진 객체"(14.1의 다이어그램)와 비교하세요. Rust는 `i64` 값들이 힙 버퍼에 **직접, 연속으로** 누워 있습니다. 포인터 추적이 한 단계(ptr → 버퍼)뿐이고, 그 이후로는 산술 연산(`ptr + i * 8`)으로 i번째 원소에 도달합니다. 이 연속성이 캐시 지역성, 프리페치, SIMD 벡터화를 가능하게 하는 물리적 기반입니다.

### 슬라이스 `&[T]`: 인자로 받을 때의 정답

`Vec<T>`를 함수에 넘길 때 초보자는 흔히 `&Vec<T>`로 받습니다. 동작은 하지만, 관용적이지 않습니다. Rust에는 `&[T]`라는 **슬라이스(slice)** 타입이 있고, 이것이 정답입니다. 슬라이스는 "버퍼의 일부(또는 전체)를 가리키는 ptr + len 두 워드"이며, `Vec<T>`, 배열 `[T; N]`, 다른 슬라이스 어느 것에서든 만들 수 있습니다.

```rust
// ❌ 덜 유연: Vec만 받을 수 있고, 배열이나 부분 구간은 못 받는다
fn sum_vec(data: &Vec<i64>) -> i64 {
    data.iter().sum()
}

// ✅ 관용적: Vec, 배열, 부분 슬라이스 모두 받는다
fn sum_slice(data: &[i64]) -> i64 {
    data.iter().sum()
}

fn main() {
    let v = vec![1, 2, 3, 4, 5];
    let arr = [10, 20, 30];
    println!("{}", sum_slice(&v));        // Vec → &[i64]
    println!("{}", sum_slice(&arr));      // [i64; 3] → &[i64]
    println!("{}", sum_slice(&v[1..4]));  // 부분 구간 [2, 3, 4] → 9
}
```

`&Vec<T>`는 "ptr → Vec 헤더 → 버퍼"로 한 단계 더 우회하지만, `&[T]`는 버퍼를 직접 가리킵니다. clippy의 `ptr_arg` 린트가 정확히 이 개선을 권합니다. `&str`이 `String`에 대해 갖는 관계가, `&[T]`가 `Vec<T>`에 대해 갖는 관계와 똑같습니다 — 사실 `String`은 본질적으로 `Vec<u8>`이고 `&str`은 `&[u8]`에 UTF-8 보증을 씌운 것이니, 같은 패턴의 두 사례입니다. 이 대칭을 머릿속에 새겨두면 다음 절의 문자열 이야기가 훨씬 자연스럽게 들어옵니다.

---

## 14.4 `String`과 `&str`: UTF-8 바이트의 세계

파이썬에서 문자열은 불변(immutable) 객체 하나(`str`)로 끝납니다. Rust에서는 문자열을 표현하는 방식이 크게 두 가지로 나뉘며, 파이썬 개발자들이 가장 많이 헷갈려하는 부분입니다.

1. **`String`**: 길이를 변경할 수 있고(growable), 내용 변경도 가능하며, **힙에 할당된 UTF-8 텍스트의 소유권을 가진 스마트 포인터**입니다. 사실 내부적으로는 `Vec<u8>`의 얇은 래퍼입니다.
2. **`&str` (string slice)**: 텍스트 데이터의 특정 부분을 가리키는 **참조(대여)**입니다. 스택, 힙, 심지어 실행 파일 내부(바이너리 `.rodata` 섹션) 등 데이터가 어디에 있든 가리킬 수 있습니다.

```rust
fn main() {
    // 1. 바이너리에 하드코딩된 문자열 리터럴을 가리키는 참조 (&'static str)
    //    소유권이 없으므로 길이를 늘리거나 수정할 수 없습니다.
    let greeting: &str = "Hello, Pythonistas!";

    // 2. 힙에 새로 할당한, 소유권을 가진 문자열 (String)
    let mut s = String::from("Hello");
    s.push_str(", world!"); // 내용을 변경(append)할 수 있습니다.

    // 3. String의 일부를 떼어내 &str로 빌리기 (slice)
    let hello: &str = &s[0..5];

    println!("{} / {} / {}", greeting, s, hello);
}
```

함수 시그니처를 쓸 때의 황금률부터 익혀 둡시다. **"읽기만 한다면 `&str`을 받고, 소유가 필요하면 `String`을 받아라."** `&str`은 `String`과 문자열 리터럴 양쪽을 모두 받을 수 있어 가장 유연합니다.

```rust
// 좋음: String이든 &str이든, 부분 슬라이스든 다 받을 수 있다
fn greet(name: &str) -> String {
    format!("안녕하세요, {}님", name)
}

fn main() {
    let owned = String::from("철수");
    println!("{}", greet(&owned));    // String → &str 자동 변환(deref coercion)
    println!("{}", greet("영희"));     // 리터럴도 그대로
}
```

### O(1) 인덱싱이 불가능한 이유

파이썬은 문자열의 길이를 계산하거나 인덱스로 접근할 때 항상 글자(character) 단위를 기준으로 합니다.

```python
text = "안녕하세요"
print(len(text))   # 5 (글자 수)
print(text[0])     # "안" (글자 단위 접근)
```

하지만 Rust의 문자열은 내부에 **UTF-8 인코딩된 바이트 배열**로 저장됩니다. ASCII는 1바이트, 한글은 글자당 3바이트, 일부 이모지는 4바이트를 차지합니다. 그래서 `text[0]`이 무엇을 반환해야 하는지가 모호합니다. 첫 바이트? 첫 글자? Rust는 이 모호함 자체를 컴파일 에러로 막습니다.

```rust
fn main() {
    let text = String::from("안녕하세요");
    let first = text[0]; // 🚨 컴파일 에러! String은 usize로 인덱싱할 수 없습니다.
    println!("{}", first);
}
```

```text
error[E0277]: the type `String` cannot be indexed by `{integer}`
  |
  |     let first = text[0];
  |                 ^^^^^^^ `String` cannot be indexed by `{integer}`
  |
  = help: the trait `Index<{integer}>` is not implemented for `String`
```

왜 막을까요? `text[0]`이 첫 글자 `'안'`을 반환하려면, Rust는 바이트 0부터 시작해 UTF-8 시퀀스를 디코딩해야 합니다. 글자 단위 인덱싱은 본질적으로 **O(n)**입니다(앞에서부터 글자 경계를 세어가야 하므로). 파이썬은 이 O(n) 비용을 뒤에서 몰래 치르거나, 메모리를 더 써서 고정폭 표현(PEP 393의 유연한 표현)으로 우회합니다. Rust는 "인덱싱은 O(1)이어야 한다"는 약속을 깨지 않기 위해, 차라리 인덱싱을 금지합니다.

그렇다면 바이트 범위 슬라이싱(`&text[0..2]`)은 어떨까요? 이건 컴파일은 되지만, 글자 경계를 깨면 런타임 패닉입니다.

```rust
fn main() {
    let text = String::from("안녕하세요");
    let slice = &text[0..2]; // 🚨 panic! '안'은 3바이트인데 2바이트만 자름
    println!("{}", slice);
}
```

```text
thread 'main' panicked at src/main.rs:3:18:
byte index 2 is not a char boundary; it is inside '안' (bytes 0..3 of `안녕하세요`)
```

`byte index 2 is not a char boundary; it is inside '안' (bytes 0..3)` — 에러 메시지가 정확히 무엇이 잘못됐는지 알려줍니다. `'안'`은 바이트 0~3을 차지하는데, 우리가 바이트 2에서 잘라 글자를 두 동강 냈다는 것입니다.

### `.len()`, `.chars()`, `.bytes()`, 그리고 그래핌

문자열의 "길이"를 묻는 세 가지 다른 질문이 있고, Rust는 이를 명확히 구분합니다.

```rust
fn main() {
    let text = String::from("안녕 Rust");

    // .len(): 바이트 수 (UTF-8 인코딩된 길이)
    println!("바이트 수: {}", text.len());          // 11 (한글 3*2=6 + 공백 1 + Rust 4)

    // .chars().count(): 유니코드 스칼라(char) 수
    println!("글자 수: {}", text.chars().count());  // 7 (안 녕 공백 R u s t)

    // .bytes(): 바이트 단위 순회
    let first_byte = text.bytes().next().unwrap();
    println!("첫 바이트: {}", first_byte);           // 236 (UTF-8 '안'의 첫 바이트)

    // .chars(): 글자(char) 단위 순회 — 올바른 "첫 글자" 접근법
    let first_char = text.chars().next().unwrap();
    println!("첫 글자: {}", first_char);             // '안'

    // n번째 글자는 O(n) 순회로
    let third = text.chars().nth(2).unwrap();
    println!("세 번째 글자: {}", third);             // ' ' (공백)
}
```

`.len()`이 바이트 수를 반환한다는 점은 파이썬 개발자가 거의 반드시 한 번 데이는 함정입니다. `len("안녕하세요")`는 파이썬에서 5지만, Rust의 `"안녕하세요".len()`은 15(바이트)입니다.

한 가지 더 깊은 층위가 있습니다. `char`(유니코드 스칼라)조차 사람이 인식하는 "글자"와 다를 수 있습니다. 예를 들어 가족 이모지나 결합 문자(combining character)는 여러 개의 `char`로 이루어진 하나의 **그래핌 클러스터(grapheme cluster)**입니다. `"é"`가 `e`+결합 액센트(2 char)일 수도 있고 단일 `é`(1 char)일 수도 있습니다. 사람이 보는 글자 단위가 필요하다면 표준 라이브러리로는 부족하고, `unicode-segmentation` 크레이트의 `.graphemes(true)`를 써야 합니다.

```rust
// Cargo.toml: unicode-segmentation = "1"
use unicode_segmentation::UnicodeSegmentation;

fn main() {
    let s = "a\u{0301}"; // 'a' + 결합 액센트 → 화면엔 'á' 하나
    println!("char 수: {}", s.chars().count());                  // 2
    println!("그래핌 수: {}", s.graphemes(true).count());        // 1
}
```

정리하면 길이의 층위는 셋입니다. **바이트(`.len()`) ≥ char(`.chars().count()`) ≥ 그래핌(`.graphemes()`)**. 무엇을 원하는지 항상 자문하세요.

> **Under the Hood: `String`의 UTF-8 바이트 레이아웃**
>
> `String`은 사실상 `Vec<u8>` 위에 "이 바이트들은 항상 유효한 UTF-8"이라는 불변 조건을 씌운 타입입니다. 따라서 `Vec`과 똑같이 3워드(ptr/len/capacity) 헤더를 가집니다.
>
> `String::from("안녕R")`의 메모리:
>
> ```text
> 스택 (String 헤더)            힙 (UTF-8 바이트 버퍼)
> ┌──────────────────┐        ┌────┬────┬────┬────┬────┬────┬────┐
> │ ptr      ────────┼───────►│ EC │ 95 │ 88 │ EB │ 85 │ 95 │ 52 │
> │ len      = 7     │        └────┴────┴────┴────┴────┴────┴────┘
> │ capacity = 8     │          └──── '안' ───┘└──── '녕' ───┘ 'R'
> └──────────────────┘            3바이트         3바이트       1바이트
> ```
>
> `len`은 7(바이트)이지 글자 수 3이 아닙니다. 그리고 `'안'`의 첫 바이트 `0xEC`를 보는 것만으로 "이건 3바이트 시퀀스의 시작"임을 알 수 있습니다(UTF-8의 자기동기화 self-synchronizing 성질). `&s[0..3]`은 char 경계(0, 3, 6, 7)에 맞으므로 성공하지만, `&s[0..2]`는 `'안'`의 한가운데를 가르므로 패닉합니다. 이것이 14.4에서 본 패닉의 물리적 원인입니다.

---

## 14.5 `HashMap<K, V>`: 기본 해셔는 왜 느린가

파이썬의 가장 강력한 무기인 딕셔너리는 내부적으로 해시 테이블입니다. Rust도 동일한 자료구조인 `HashMap`을 제공하지만, 기본 동작에 중요한 차이가 있습니다.

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();

    // String과 i32를 넣으면서 소유권이 HashMap으로 넘어갑니다(Move).
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    // get()은 파이썬과 달리 Option<&V>를 반환합니다.
    // "값이 없을 수 있다"는 사실을 타입으로 강제합니다.
    let blue = scores.get("Blue").copied().unwrap_or(0); // &str로 조회 가능
    println!("Blue = {}", blue);

    // 덮어쓰기
    scores.insert(String::from("Yellow"), 60);

    println!("{:?}", scores);
}
```

여기서 소유권에 주의해야 합니다. `scores.insert(key, 10)`은 `key`의 소유권을 가져갑니다. 파이썬이라면 신경 쓸 일이 없지만, Rust에서는 삽입 후 `key`를 다시 쓰면 컴파일 에러입니다.

```rust
use std::collections::HashMap;

fn main() {
    let key = String::from("Blue");
    let mut scores = HashMap::new();
    scores.insert(key, 10);
    println!("{}", key); // 🚨 컴파일 에러! key는 이미 move됨
}
```

```text
error[E0382]: borrow of moved value: `key`
  |
3 |     let key = String::from("Blue");
  |         --- move occurs because `key` has type `String`, which does not implement the `Copy` trait
4 |     let mut scores = HashMap::new();
5 |     scores.insert(key, 10);
  |                   --- value moved here
6 |     println!("{}", key);
  |                    ^^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
5 |     scores.insert(key.clone(), 10);
```

`borrow of moved value: key` — Chapter 4의 소유권 규칙이 그대로입니다. `String`은 `Copy`가 아니므로 `insert`가 소유권을 가져가고, 그 후의 사용은 차단됩니다. 컴파일러는 친절하게도 `.clone()`을 제안하지만, 정말 필요할 때만 복제하세요. 키를 계속 쓸 거라면 처음부터 `&key`로 조회하거나, 키 타입을 `&str`로 설계하는 편이 낫습니다.

### Entry API: `setdefault`와 `defaultdict`의 대체

파이썬의 `dict.setdefault()`와 `defaultdict`는 "키가 없으면 기본값을 넣고, 있으면 그대로 둔다"는 패턴입니다. Rust의 대응물은 **Entry API**이며, 더 강력합니다.

```python
# Python
from collections import defaultdict
counts = defaultdict(int)
for word in "the cat the dog the bird".split():
    counts[word] += 1
```

```rust
use std::collections::HashMap;

fn main() {
    let sentence = "the cat the dog the bird";
    let mut counts: HashMap<&str, i32> = HashMap::new();

    for word in sentence.split_whitespace() {
        // entry(): 슬롯에 대한 핸들을 얻는다
        // or_insert(0): 없으면 0을 넣고, 그 슬롯의 &mut를 반환
        // *...+= 1: 역참조해서 증가
        *counts.entry(word).or_insert(0) += 1;
    }

    let mut pairs: Vec<_> = counts.iter().collect();
    pairs.sort();
    println!("{:?}", pairs); // [("bird", 1), ("cat", 1), ("dog", 1), ("the", 3)]
}
```

Entry API의 진가는 **해시를 한 번만 계산한다**는 데 있습니다. 파이썬에서 `if k in d: d[k] += 1 else: d[k] = 1`을 쓰면 `in` 검사와 대입에서 해시를 두 번 계산하지만, `entry(word)`는 슬롯을 한 번에 찾아 핸들을 줍니다. 기본값 계산이 비쌀 때는 `or_insert_with(|| expensive())`를 써서 키가 없을 때만 클로저를 실행합니다.

```rust
use std::collections::HashMap;

fn main() {
    let mut groups: HashMap<bool, Vec<i32>> = HashMap::new();
    for n in 1..=6 {
        // 키가 없으면 빈 Vec을 만들고(or_insert_with), 그 Vec에 push
        groups.entry(n % 2 == 0).or_insert_with(Vec::new).push(n);
    }
    println!("짝수: {:?}", groups[&true]);   // [2, 4, 6]
    println!("홀수: {:?}", groups[&false]);  // [1, 3, 5]
}
```

### 기본 해셔 SipHash: 안전하지만 느리다

이제 가장 중요한 차이입니다. Rust의 `HashMap`은 기본적으로 **SipHash 1-3**라는 해시 함수를 씁니다. 이는 의도적인 설계 결정이며, 보안과 성능의 트레이드오프입니다.

10년 차 백엔드 엔지니어라면 "해시 충돌 공격(Hash-Flooding DoS)"을 알 겁니다. 공격자가 같은 해시 버킷에 몰리는 키들을 대량으로 보내면, 해시 테이블이 O(1)에서 O(n)으로 퇴화해 서버가 마비됩니다. 파이썬도 3.3부터 이 공격을 막기 위해 해시 시드를 무작위화(`PYTHONHASHSEED`)했습니다. Rust는 한 걸음 더 나아가, **암호학적으로 강한** SipHash를 기본값으로 채택하고 프로세스마다 무작위 시드를 씁니다. 외부 입력을 키로 받는 웹 서버라면 이 방어가 공짜로 켜져 있는 셈입니다.

문제는 SipHash가 짧은 키(정수, 작은 문자열)에 대해 상대적으로 느리다는 것입니다. 키가 신뢰할 수 있는 내부 데이터(예: 정수 ID, 컴파일 타임 상수)이고 DoS 위협이 없다면, 훨씬 빠른 해시로 교체하는 것이 정석입니다. 실무 표준은 `ahash`와 `fxhash`(러스트 컴파일러 자체가 쓰는 해시)입니다.

```rust
// Cargo.toml:
// ahash = "0.8"
use std::collections::HashMap;
use ahash::RandomState;

fn main() {
    // 기본 해셔(SipHash) 대신 ahash 사용
    let mut map: HashMap<u64, u64, RandomState> =
        HashMap::with_hasher(RandomState::new());

    for i in 0..1_000_000u64 {
        map.insert(i, i * 2);
    }
    println!("{:?}", map.get(&999_999)); // Some(1999998)
}
```

정수 키 100만 개를 삽입·조회하는 마이크로 벤치마크에서, `ahash`나 `fxhash`로 교체하면 기본 SipHash 대비 대략 **2~4배** 빨라지는 것이 일반적입니다(키 분포와 크기에 따라 다름). 더 간편하게는 `hashbrown`(표준 `HashMap`의 실제 구현체) + `ahash` 조합을 쓰거나, 타입 별칭으로 프로젝트 전체에 적용합니다.

```rust
use ahash::AHashMap; // 별칭 타입: HashMap<K, V, ahash::RandomState>

fn main() {
    let mut map: AHashMap<&str, i32> = AHashMap::new();
    map.insert("answer", 42);
    println!("{:?}", map.get("answer"));
}
```

핵심 의사결정: **외부 신뢰 불가 입력이 키라면 기본 SipHash를 유지하라. 내부의 신뢰 가능한 키이고 성능이 중요하다면 ahash/fxhash로 교체하라.**

### 조회가 반환하는 것은 값이 아니라 참조다

파이썬 개발자가 한 번 더 데이는 지점이 있습니다. `scores["Blue"]`는 파이썬에서 값 자체를 줍니다. 하지만 Rust의 `map.get(k)`는 `Option<&V>`, 즉 **값에 대한 참조**를 줍니다. 그리고 `map[k]`(Index 연산)는 키가 없으면 패닉합니다 — `Vec`의 인덱싱과 동일한 위험입니다.

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert("Blue", 10);

    // get(): 안전, Option<&i32>
    if let Some(&v) = scores.get("Blue") {     // &로 역참조 패턴 매칭
        println!("Blue = {}", v);
    }

    // copied()/cloned(): 참조를 복사/복제해 소유값으로
    let blue: i32 = scores.get("Blue").copied().unwrap_or(0);
    println!("{}", blue);

    // map["Red"]  // 🚨 panic! 키가 없으면 죽는다. Vec의 [] 와 같은 함정.

    // 순회: (&K, &V) 쌍으로 빌려서 돈다 (소유권 이동 없음)
    for (team, score) in &scores {
        println!("{}: {}", team, score);
    }

    // 값을 수정하며 순회하려면 &mut
    for (_team, score) in &mut scores {
        *score += 1;
    }
}
```

`get()`이 참조를 반환한다는 사실은 불필요한 복제를 막아주는 설계입니다. 큰 값(예: `Vec`나 `String`)을 맵에 담아두고 `get`으로 꺼낼 때, 파이썬이라면 의식하지 않을 "이게 복사인가 참조인가"를 Rust는 명확히 합니다 — 기본은 참조이고, 복사가 필요하면 `.copied()`/`.cloned()`로 명시합니다. 이 명시성이 14.10에서 다룰 "불필요한 할당" 함정을 구조적으로 줄여줍니다.

> **Under the Hood: `dict`의 오픈 어드레싱 vs `HashMap`의 SwissTable**
>
> 두 언어 모두 오픈 어드레싱(open addressing) 계열이지만 구현이 다릅니다.
>
> CPython `dict`는 "조밀 배열 + 희소 인덱스"의 2단 구조입니다. 충돌 시 의사난수 탐사(perturbation probing)로 다음 슬롯을 찾고, 삽입 순서를 보존하는 엔트리 배열을 따로 둡니다.
>
> Rust `HashMap`은 Google의 **SwissTable**(crate `hashbrown`)을 채택했습니다. 해시값을 두 부분으로 쪼개, 상위 비트로 버킷 그룹을 정하고 하위 7비트(`control byte`)를 별도의 메타데이터 배열에 둡니다. 조회 시 SIMD 명령(SSE2의 `_mm_cmpeq_epi8` 등)으로 **16개 control byte를 한 번에 비교**해 후보를 추립니다.
>
> ```text
> control bytes (메타데이터, SIMD로 16개씩 한 번에 스캔)
> ┌────┬────┬────┬────┬─── ... ───┐
> │ h2 │ EMP│ h2 │ DEL│           │   ← 각 1바이트: 비었음/삭제됨/해시 하위7비트
> └────┴────┴────┴────┴───────────┘
> data buckets (실제 (K, V) 쌍, 연속 배열)
> ┌──────────┬──────────┬──────────┐
> │ (K0, V0) │   ...    │ (Kn, Vn) │
> └──────────┴──────────┴──────────┘
> ```
>
> 이 SIMD 그룹 스캔 덕분에 SwissTable은 캐시 효율과 충돌 처리에서 전통적 탐사보다 빠릅니다. 다만 `dict`와 달리 **삽입 순서를 보존하지 않습니다.** 순서가 필요하면 `indexmap` 크레이트(`IndexMap`)를 쓰세요. 이것이 파이썬 3.7+ `dict`의 순서 보존에 대응합니다.

---

## 14.6 나머지 컬렉션: VecDeque, BTreeMap, HashSet, BinaryHeap

파이썬의 `collections`와 `heapq`에 대응하는 표준 컬렉션들입니다. 하나의 실행 가능한 예제로 묶어 봅니다.

```rust
use std::collections::{VecDeque, BTreeMap, HashSet, BinaryHeap};
use std::cmp::Reverse;

fn main() {
    // VecDeque: 양끝 O(1) 큐 (파이썬 collections.deque)
    let mut dq: VecDeque<i32> = VecDeque::new();
    dq.push_back(1);
    dq.push_front(0);
    println!("앞={:?} 뒤={:?}", dq.front(), dq.back()); // 앞=Some(0) 뒤=Some(1)

    // BTreeMap: 키가 항상 정렬된 맵 (파이썬엔 직접 대응 없음, sorted(dict))
    let mut bt = BTreeMap::new();
    bt.insert("c", 3);
    bt.insert("a", 1);
    bt.insert("b", 2);
    let keys: Vec<_> = bt.keys().collect();
    println!("정렬된 키: {:?}", keys); // ["a", "b", "c"]
    // 범위 쿼리: BTreeMap만의 강점
    for (k, v) in bt.range("a".."c") {
        println!("범위 [a, c): {} = {}", k, v); // a=1, b=2
    }

    // HashSet: 중복 없는 집합 (파이썬 set)
    let mut seen = HashSet::new();
    seen.insert("x");
    seen.insert("x"); // 중복은 무시됨
    seen.insert("y");
    println!("집합 크기: {}", seen.len()); // 2

    // BinaryHeap: 기본은 최대 힙 (파이썬 heapq는 최소 힙)
    let mut max_heap = BinaryHeap::new();
    max_heap.push(3);
    max_heap.push(1);
    max_heap.push(5);
    println!("최댓값 pop: {:?}", max_heap.pop()); // Some(5)

    // 최소 힙이 필요하면 Reverse로 감싼다 (파이썬 heapq처럼)
    let mut min_heap = BinaryHeap::new();
    min_heap.push(Reverse(3));
    min_heap.push(Reverse(1));
    if let Some(Reverse(min)) = min_heap.pop() {
        println!("최솟값 pop: {}", min); // 1
    }
}
```

선택 가이드:

- **`VecDeque`**: 큐(FIFO)나 양끝 삽입/삭제가 필요할 때. `Vec::remove(0)`은 O(n)이지만 `VecDeque::pop_front()`는 O(1)입니다. 파이썬 `deque`와 동일한 동기입니다.
- **`BTreeMap`/`BTreeSet`**: 키 정렬 순회나 범위 쿼리(`range`)가 필요할 때. 균형 B-트리이므로 조회는 O(log n)으로 `HashMap`의 O(1)보다 느리지만, 정렬·범위라는 능력을 얻습니다. 파이썬에는 표준 대응이 없어 보통 `sorted()`나 `sortedcontainers` 라이브러리를 씁니다.
- **`HashSet`/`BTreeSet`**: 집합 연산(`union`, `intersection`, `difference`). 파이썬 `set`/`frozenset`에 대응합니다.
- **`BinaryHeap`**: 우선순위 큐. 파이썬 `heapq`가 리스트 위에서 동작하는 최소 힙인 반면, Rust `BinaryHeap`은 기본이 최대 힙이라 최소 힙은 `Reverse` 래퍼가 필요합니다.

---

## 14.7 실전 예제 (Hands-on): 로그 분석기

지금까지 배운 `Vec`, `String`, `HashMap`, `BTreeMap`을 모두 엮어, 실무에서 흔한 작업인 로그 라인 분석기를 만들어 봅시다. 입력은 가짜 HTTP 액세스 로그이고, 목표는 (1) 상태 코드별 요청 수 집계, (2) 가장 많이 호출된 엔드포인트 Top 3, (3) 상태 코드를 정렬해 출력하는 것입니다. 그대로 `cargo run` 됩니다.

```rust
use std::collections::{HashMap, BTreeMap};

/// "GET /api/users 200" 형태의 한 줄을 (메서드, 경로, 상태코드)로 파싱.
/// 형식이 깨졌으면 None을 반환 — Chapter 12의 Option으로 실패를 표현.
fn parse_line(line: &str) -> Option<(&str, &str, u16)> {
    let mut parts = line.split_whitespace();
    let method = parts.next()?;
    let path = parts.next()?;
    let status: u16 = parts.next()?.parse().ok()?; // 파싱 실패도 None으로
    Some((method, path, status))
}

fn main() {
    let logs = "\
GET /api/users 200
POST /api/users 201
GET /api/users 200
GET /api/orders 500
GET /api/users 200
POST /api/login 200
GET /api/orders 200
BROKEN LINE
GET /api/login 404";

    // path별 호출 횟수: Entry API로 집계 (defaultdict(int)에 대응)
    let mut path_counts: HashMap<&str, u32> = HashMap::new();
    // 상태 코드별 횟수: BTreeMap으로 자동 정렬
    let mut status_counts: BTreeMap<u16, u32> = BTreeMap::new();

    for line in logs.lines() {
        match parse_line(line) {
            Some((_method, path, status)) => {
                *path_counts.entry(path).or_insert(0) += 1;
                *status_counts.entry(status).or_insert(0) += 1;
            }
            None => eprintln!("⚠️  파싱 실패, 건너뜀: {:?}", line),
        }
    }

    // 상태 코드별 집계 (BTreeMap이므로 코드 오름차순 보장)
    println!("== 상태 코드별 요청 수 ==");
    for (status, count) in &status_counts {
        println!("  {} → {}건", status, count);
    }

    // Top 3 엔드포인트: HashMap을 Vec으로 옮겨 정렬
    let mut ranked: Vec<(&str, u32)> = path_counts.into_iter().collect();
    // 횟수 내림차순, 동률이면 경로 오름차순으로 안정 정렬
    ranked.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));

    println!("\n== Top 3 엔드포인트 ==");
    for (rank, (path, count)) in ranked.iter().take(3).enumerate() {
        println!("  {}. {} ({}회)", rank + 1, path, count);
    }
}
```

실행 결과입니다.

```text
⚠️  파싱 실패, 건너뜀: "BROKEN LINE"
== 상태 코드별 요청 수 ==
  200 → 5건
  201 → 1건
  404 → 1건
  500 → 1건

== Top 3 엔드포인트 ==
  1. /api/users (4회)
  2. /api/login (2회)
  3. /api/orders (2회)
```

이 작은 예제에 이번 장의 핵심이 모두 들어 있습니다. `split_whitespace()`로 `&str` 슬라이스를 얻어 **불필요한 `String` 할당 없이** 파싱하고(소유 대신 대여), `HashMap::entry().or_insert()`로 집계하며, 정렬이 필요한 상태 코드는 `BTreeMap`에 맡기고, Top-N 랭킹은 `HashMap → Vec` 변환 후 `sort_by`로 처리했습니다. 파싱 실패는 패닉이 아니라 `Option::None`으로 우아하게 흘려보냅니다. 파이썬으로 짰다면 더 짧았겠지만, Rust 버전은 모든 실패 경로가 타입으로 드러나고, 컴파일되는 순간 인덱스 에러·키 부재·타입 혼동이 모두 사라집니다.

---

## 14.8 벤치마크: 연속성이 만드는 격차

과장 없이 현실적인 수치를 봅시다. 측정 환경은 `--release` 빌드, 입력 1,000만 개, Apple Silicon급 데스크톱 기준의 대표적 결과입니다(절대값은 머신마다 다르나 배수는 일관적입니다).

**벤치 1: `Vec<i64>` 합산 vs `list` 합산.**

```python
# Python
data = list(range(10_000_000))
total = sum(data)   # 약 60~90ms
```

```rust
// Rust (--release)
fn main() {
    let data: Vec<i64> = (0..10_000_000).collect();
    let total: i64 = data.iter().sum();   // 약 3~5ms
    println!("{}", total);
}
```

대략 **15~25배** 차이입니다. 이유는 이번 장 내내 강조한 캐시 지역성입니다. `Vec<i64>`는 연속 버퍼라 CPU가 한 캐시 라인에 8개씩 당겨오고 SIMD로 더합니다. 파이썬 `list`는 포인터를 따라 흩어진 정수 객체로 점프하며 캐시 미스를 양산하고, 매 덧셈마다 객체를 언박싱합니다. `numpy`의 `arr.sum()`을 쓰면 파이썬도 연속 버퍼가 되어 격차가 크게 좁혀집니다 — 그것이 바로 `numpy`가 하는 일이고, Rust의 `Vec`은 기본형이 이미 그렇습니다.

**벤치 2: `HashMap` 조회 vs `dict` 조회.** 정수 키 100만 개를 삽입한 뒤 100만 번 조회:

| 구현 | 상대 속도 |
|---|---|
| Python `dict` | 기준 (1.0x) |
| Rust `HashMap` (기본 SipHash) | 약 1.5~3x 빠름 |
| Rust `HashMap` + `ahash` | 약 4~8x 빠름 |

흥미로운 점은 Rust의 기본 `HashMap`이 파이썬 `dict`보다 압도적으로 빠르지는 않다는 것입니다. SipHash의 암호학적 강도 비용 때문입니다. `ahash`로 바꾸는 순간 격차가 크게 벌어집니다. 14.5에서 강조한 "기본 해셔는 느리다"가 수치로 확인되는 지점입니다.

**벤치 3: 문자열 처리(공백으로 split 후 단어 수 집계).** 대용량 텍스트에서 단어 빈도를 셀 때, Rust는 `&str` 슬라이스로 **할당 없이** 토큰을 순회할 수 있습니다. 파이썬의 `str.split()`은 매 토큰마다 새 `str` 객체를 힙에 만듭니다. 대략 **10~30배** 차이가 나며, 특히 `String`을 불필요하게 만들지 않는 것이 핵심입니다. 만약 Rust 코드에서 토큰마다 `.to_string()`을 호출한다면 이 이점의 상당 부분을 스스로 날려버립니다(다음 절의 함정 참고).

수치를 신뢰성 있게 재려면 `criterion` 크레이트를 쓰세요. 워밍업, 이상치 제거, 통계적 신뢰구간을 자동으로 처리해 줍니다. 단순 `Instant::now()` 측정은 JIT 없는 Rust에서도 캐시 상태와 빌드 모드(`--release` 필수)에 크게 흔들립니다.

---

## 14.9 Best Practices & 안티패턴

실무 체크리스트입니다.

**이렇게 하라:**

- ✅ 최종 크기를 알면 `Vec::with_capacity(n)` / `HashMap::with_capacity(n)`으로 재할당을 없앤다.
- ✅ 함수 인자는 소유가 필요 없으면 `&str`, `&[T]`, `&HashMap`으로 빌려 받는다. `String`/`Vec<T>`를 통째로 받는 것은 소유권 이전이 정말 필요할 때만.
- ✅ "없으면 넣기" 패턴은 `entry().or_insert()` / `or_insert_with()`로. 해시를 한 번만 계산한다.
- ✅ 인덱스 범위가 불확실하면 `get()`(→ `Option`)을, 보장되면 `[]`를 쓴다.
- ✅ 외부 입력이 키가 아니고 성능이 중요하면 `ahash`/`fxhash`로 해셔를 교체한다.
- ✅ 삽입 순서 보존이 필요하면 `indexmap::IndexMap`, 키 정렬·범위 쿼리가 필요하면 `BTreeMap`.
- ✅ 문자열 길이의 의미를 명확히 한다: 바이트는 `.len()`, 글자는 `.chars().count()`, 사람이 보는 글자는 `unicode-segmentation`.

**이건 피하라 (clippy가 잡아준다):**

- ❌ 루프 안에서 `vec.remove(0)`으로 큐 흉내내기 → O(n²). `VecDeque::pop_front()` 사용. (clippy: 직접 잡진 않지만 성능 함정)
- ❌ `&Vec<T>`나 `&String`을 인자로 받기 → clippy `ptr_arg`가 `&[T]`/`&str`을 권한다.
- ❌ `map.contains_key(k)` 후 `map.get(k)` → 조회 두 번. `entry`나 단일 `get`으로.
- ❌ 이터레이터에서 `.collect::<Vec<_>>()` 후 다시 순회 → 중간 할당 낭비. 가능하면 체인 유지(Chapter 15).

---

## 14.10 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: 문자열을 인덱스로 자르려는 습관.**

```python
text = "hello"
print(text[0])      # 'h' — 파이썬에선 당연
```

```rust
let text = "hello";
let c = text[0];    // 🚨 컴파일 에러! String/&str은 usize 인덱싱 불가
```

파이썬의 `text[0]`은 Rust에서 컴파일조차 안 됩니다(`Index<{integer}> is not implemented`). 첫 글자가 필요하면 `text.chars().next().unwrap()`, 바이트가 필요하면 `text.as_bytes()[0]`을 쓰세요. ASCII 텍스트라면 `&text[0..1]`도 가능하지만, 입력에 한글·이모지가 섞이는 순간 14.4에서 본 char boundary 패닉을 맞습니다. 사용자 입력 문자열은 항상 멀티바이트를 가정하세요.

**함정 2: 불필요한 `String` 할당.**

```rust
// ❌ 안티패턴: 읽기만 하는데 소유 문자열을 받고, 또 복제한다
fn first_word(s: String) -> String {
    s.split_whitespace().next().unwrap().to_string()
}

// ✅ 빌려서 받고, 슬라이스로 반환 — 힙 할당 0회
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}
```

파이썬은 모든 문자열이 불변 객체라 "복제"라는 개념을 의식하지 않습니다. 그래서 Rust에서도 무심코 `.to_string()`, `.clone()`, `String::from()`을 남발하기 쉽습니다. 이는 14.8 벤치마크에서 본 성능 이점을 직접 깎아먹는 행위입니다. 원칙은 단순합니다. **빌릴 수 있으면 빌려라(`&str`, `&[T]`). 복제는 소유가 진짜 필요할 때 마지막 수단으로.** clippy의 `clippy::redundant_clone` 린트가 명백한 경우를 잡아줍니다.

**함정 3: 기본 해셔 성능을 모르고 쓰기.**

파이썬 `dict`가 빠르다는 인상 때문에, Rust `HashMap`도 당연히 빠를 거라 가정하고 정수 키 수천만 개로 핫 루프를 돌리다가 의외로 느린 결과에 당황하는 경우입니다. 원인은 기본 해셔 SipHash의 암호학적 강도 비용입니다(14.5). 외부 입력이 키가 아닌 내부 정수/짧은 문자열 키라면 `ahash`로 교체해 2~8배를 회수하세요. 반대로, 외부 신뢰 불가 입력을 키로 받으면서 성능을 위해 `fxhash`로 바꾸는 것은 DoS 문을 여는 일이니 절대 하지 마세요.

**함정 4: 순회 중 컬렉션 수정.**

```python
# Python — 런타임 RuntimeError 가능
for k in d:
    if cond(k):
        del d[k]   # dict changed size during iteration
```

Rust에서는 이것이 **컴파일 에러**입니다. `for x in &v`로 불변 대여 중인데 `v.push(...)`를 시도하면 "이미 빌려준 것을 변경할 수 없다"는 차용 검사기(Borrow Checker, Chapter 5) 에러가 납니다. 파이썬은 런타임에 가끔 잡고 가끔 놓치지만, Rust는 컴파일 타임에 항상 막습니다. 해법은 `retain`(조건부 보존)이나, 삭제할 인덱스/키를 먼저 모은 뒤 별도 패스에서 처리하는 것입니다.

```rust
let mut v = vec![1, 2, 3, 4, 5];
v.retain(|&x| x % 2 == 0); // 짝수만 남김 → [2, 4]
```

---

지금까지 우리는 Rust의 컬렉션이 파이썬의 자료구조보다 엄격하다는 사실을 확인했습니다. 타입을 섞을 수 없고, 인덱싱에 주의해야 하며, 값을 넘길 때 항상 소유권(Move)과 대여(Borrow)를 신경 써야 합니다. 문자열은 UTF-8 바이트의 세계라 인덱싱조차 다시 배워야 했습니다.

하지만 이 약간의 엄격함을 대가로, 우리는 파이썬의 백그라운드에서 끊임없이 돌아가는 참조 카운팅 연산과 포인터 추적으로 파편화된 메모리 접근 오버헤드를 완전히 걷어냈습니다. `Vec<i64>`는 그 자체로 `numpy` 배열이고, `HashMap`은 SIMD로 가속된 SwissTable이며, `String`은 글자 경계를 함부로 깨지 않는 안전한 UTF-8 버퍼입니다. 무엇보다 이 모든 것이 무비용 추상화(Zero-cost Abstraction)로, 런타임 비용 없이 컴파일 타임에 검증됩니다.

이 컬렉션들은 그저 데이터를 담아두는 그릇에 그치지 않습니다. 다음 Chapter 15에서는 이 그릇 안의 데이터를 **루프보다도 빠르게, 그리고 우아하게** 처리하는 방법을 다룹니다. 파이썬의 제너레이터(`yield`)와 리스트 컴프리헨션을 뛰어넘는 **이터레이터(Iterator)와 클로저(Closure)**가 그 주인공입니다. 방금 만든 로그 분석기의 `for` 루프와 `sort_by`가, 이터레이터 체이닝으로 어떻게 더 선언적이면서도 어셈블리 레벨에서 동일하거나 더 빠른 코드로 컴파일되는지 보게 될 것입니다.
