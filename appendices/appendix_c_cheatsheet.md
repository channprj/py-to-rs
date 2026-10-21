# 부록 C. Python → Rust 치트시트

이 부록은 본문을 읽으며 체득한 사고 흐름을 **빠르게 찾아보는 레퍼런스**로 압축한 것입니다. 설명은 최소화하고, 왼쪽에 익숙한 Python 관용구, 오른쪽에 그대로 동작하는 Rust 코드를 둡니다. 모든 Rust 코드는 `edition = 2021` 기준이며, 별도 표기가 없으면 표준 라이브러리만으로 컴파일됩니다.

표를 볼 때 두 가지를 기억하세요. 첫째, Python의 한 줄이 Rust에서 한 줄이 아닌 경우는 **거의 항상 소유권(Ownership)·타입(Type)·에러(Error)를 명시적으로 드러내기 때문**입니다. 둘째, Rust의 이터레이터 체인은 `for` 루프와 동일한 어셈블리로 단일형화(Monomorphization)되므로, "함수형 스타일이 느릴까?" 걱정할 필요가 없습니다(Chapter 15).

> 표기 규약: `🚨`는 컴파일되지 않거나 패닉하는 코드, `// ...`는 발췌 생략, `&`는 빌림(Borrow), `?`는 에러 조기 반환입니다.

---

## C.1 기본 문법 — 변수, 상수, 출력, 주석

| Python | Rust |
|---|---|
| `x = 10` | `let x = 10;` (불변) / `let mut x = 10;` (가변) |
| `x: int = 10` | `let x: i64 = 10;` |
| `PI = 3.14` (관례상 상수) | `const PI: f64 = 3.14;` (컴파일 타임 상수) |
| 전역 가변 상태 | `static mut`(unsafe) 또는 `static` + `Mutex`/`OnceLock` |
| `x = 1; x = "a"` (재바인딩) | `let x = 1; let x = "a";` (섀도잉, 타입 변경 가능) |
| `del x` | 스코프 종료 또는 `drop(x);` |
| `# 한 줄 주석` | `// 한 줄 주석` |
| `"""docstring"""` | `/// 문서 주석` (항목 위), `//! ` (모듈 내부) |
| `print("hi")` | `println!("hi");` |
| `print(x, y)` | `println!("{x} {y}");` (인라인 캡처) |
| `print(f"{x=}")` | `println!("{x:?}");` 또는 `dbg!(x);` |
| `print(x, end="")` | `print!("{x}");` (개행 없음) |
| `import sys; sys.stderr.write(...)` | `eprintln!("error: {msg}");` |

Python의 `=`는 이름을 객체에 묶는 행위이지만, Rust의 `let`은 **기본적으로 불변 바인딩을 만들고 값을 그 자리에 둡니다**. 가변성이 필요하면 `mut`를 명시적으로 요청해야 합니다. 이 한 글자가 데이터 경합(Data Race)을 컴파일 타임에 차단하는 출발점입니다(Chapter 4, 17).

섀도잉(Shadowing)은 파이썬 재바인딩과 닮았지만 본질이 다릅니다. 파이썬에서 `x = "a"`는 같은 변수가 새 객체를 가리키게 바꾸는 것이고, Rust의 `let x = "a";`는 이전 `x`를 가린 채 **완전히 새로운 변수를 같은 이름으로 도입**하는 것입니다. 그래서 타입까지 바꿀 수 있습니다. 둘의 차이가 드러나는 곳은 불변성입니다. 파이썬의 변수는 언제든 다시 대입할 수 있지만, Rust에서 `mut` 없는 `x`에 두 번째로 대입하면 `error[E0384]: cannot assign twice to immutable variable`가 뜹니다. 값을 의도적으로 갱신할 때는 `mut`를, 단지 변환 단계를 이어 쓸 때는 섀도잉을 쓰는 것이 관례입니다.

상수에서도 사고방식이 갈립니다. 파이썬의 "상수"는 대문자 이름이라는 신사협정일 뿐 런타임에 얼마든지 바뀌지만, Rust의 `const`는 컴파일 타임에 값이 확정되어 사용처마다 인라인됩니다. 즉 `const`는 메모리 주소를 갖지 않고, `static`은 프로그램 수명 내내 고정된 주소를 갖는다는 점도 함께 기억해 두면 전역 상태를 설계할 때 헷갈리지 않습니다.

---

## C.2 조건, 반복, 함수, 클로저

### 조건과 반복

| Python | Rust |
|---|---|
| `if a: ... elif b: ... else: ...` | `if a { } else if b { } else { }` |
| `x = 1 if c else 2` | `let x = if c { 1 } else { 2 };` (if는 표현식) |
| `for i in range(5):` | `for i in 0..5 {` |
| `for i in range(0, 10, 2):` | `for i in (0..10).step_by(2) {` |
| `for x in xs:` | `for x in &xs {` (빌림 순회) |
| `for i, x in enumerate(xs):` | `for (i, x) in xs.iter().enumerate() {` |
| `while cond:` | `while cond {` |
| `while True:` | `loop {` (무한 루프 전용 키워드) |
| `break` / `continue` | `break;` / `continue;` (라벨: `'outer: loop {`) |
| `for ... else:` | 직접 대응 없음 → `loop`에서 `break value`로 값 반환 |
| `match`(3.10+) | `match`(패턴 매칭, 망라성 검사 필수) |

```rust
fn main() {
    // loop는 값을 반환하는 표현식이 될 수 있습니다 (Python에는 없는 패턴)
    let mut n = 0;
    let first_over_100 = loop {
        n += 7;
        if n > 100 {
            break n; // loop 전체가 이 값으로 평가됩니다
        }
    };
    println!("{first_over_100}"); // 105
}
```

### 함수, 람다 → 클로저

| Python | Rust |
|---|---|
| `def add(a: int, b: int) -> int:` | `fn add(a: i64, b: i64) -> i64 {` |
| `return a + b` | `a + b` (마지막 표현식, 세미콜론 없음) |
| `def f(): pass` (None 반환) | `fn f() {}` (유닛 `()` 반환) |
| `f = lambda x: x + 1` | `let f = \|x\| x + 1;` |
| `sorted(xs, key=lambda p: p[1])` | `xs.sort_by_key(\|p\| p.1);` |
| 클로저가 외부 변수 캡처 | `\|x\| x + base` (자동 캡처), `move \|x\| x + base` (소유권 이동) |
| `*args, **kwargs` | 직접 대응 없음 → 슬라이스 `&[T]`, 빌더, 또는 매크로 |
| 기본 인자 `def f(x=10):` | 직접 대응 없음 → `Option<T>` 또는 빌더 패턴(Chapter 11) |

```rust
fn main() {
    let base = 100;
    // Fn / FnMut / FnOnce는 캡처 방식에 따라 컴파일러가 자동 추론합니다
    let add_base = |x: i32| x + base; // base를 불변 빌림으로 캡처
    println!("{}", add_base(5)); // 105

    // move: 캡처한 값의 소유권을 클로저로 이동 (스레드 전달 시 필수, Chapter 16)
    let owned = String::from("hi");
    let printer = move || println!("{owned}");
    printer();
}
```

Python의 함수는 일급 객체인 단일 타입(`function`)이지만, Rust의 모든 클로저는 **각자 고유한 익명 타입**을 가집니다. 그래서 클로저를 인자로 받을 때 제네릭 `fn run<F: Fn(i32) -> i32>(f: F)` 또는 트레이트 객체 `Box<dyn Fn(i32) -> i32>`를 씁니다(Chapter 9, 19).

클로저가 `Fn`, `FnMut`, `FnOnce` 중 무엇이 되는지는 캡처한 값을 어떻게 다루는지에 따라 컴파일러가 결정합니다. 캡처한 변수를 읽기만 하면 `Fn`, 변경하면 `FnMut`, 소비(이동)하면 `FnOnce`입니다. 이 구분은 파이썬에 없는 개념인데, 파이썬은 클로저가 바깥 변수를 자유롭게 읽고 쓰며 그 비용(셀 객체, 참조 카운트)을 런타임에 치르기 때문입니다. Rust는 이 비용을 타입으로 환산해 컴파일 타임에 확정합니다. 그리고 함수가 끝난 뒤에도 살아남아야 하는 클로저(스레드, 콜백 저장)에는 `move`로 소유권을 통째로 옮겨, 빌림이 함수 수명보다 오래 사는 위험을 원천 차단합니다(Chapter 16, 18).

---

## C.3 문자열 — 포매팅과 메서드

### f-string과 포매팅

| Python | Rust |
|---|---|
| `f"{name} is {age}"` | `format!("{name} is {age}")` |
| `f"{x:.2f}"` | `format!("{x:.2}")` |
| `f"{x:>8}"` | `format!("{x:>8}")` (우측 정렬) |
| `f"{x:08b}"` | `format!("{x:08b}")` (2진수, 0 패딩) |
| `f"{x:#x}"` | `format!("{x:#x}")` (0x 접두 16진수) |
| `"%s" % x` (구식) | `format!("{x}")` |
| `repr(x)` | `format!("{x:?}")` (Debug 트레이트) |
| `"".join(parts)` | `parts.join("")` 또는 `parts.concat()` |
| `s += "more"` | `s.push_str("more");` |
| `s += c` (한 글자) | `s.push('c');` |

Python `str`은 불변(Immutable)이라 `+=`마다 새 객체를 만들지만, Rust `String`은 **가변 힙 버퍼**라 `push_str`이 제자리(in-place)에서 자랍니다. 루프 안 문자열 누적은 Rust가 구조적으로 유리합니다. 파이썬에서 `s += x`를 만 번 반복하면 매번 새 문자열을 할당·복사해 O(n²)에 가까워지지만(그래서 우리는 `"".join(parts)` 관용구를 외워 둡니다), Rust의 `push_str`은 용량(capacity)이 부족할 때만 재할당하는 분할 상환 O(1)이라 루프 누적을 안심하고 씁니다.

`&str`과 `String`의 구분도 처음엔 낯설지만 규칙은 단순합니다. `&str`은 **빌린 문자열 조각**(문자열 리터럴, 슬라이스)이고 `String`은 **소유한 힙 버퍼**입니다. 함수 인자는 가능한 한 `&str`로 받아 `String`이든 리터럴이든 모두 받아들이고, 반환하거나 저장할 때는 `String`을 씁니다. 파이썬의 `str` 하나가 두 역할을 겸하던 것을 Rust는 "누가 소유하는가"로 갈라 둔 셈입니다(Chapter 14).

### 문자열 메서드 매핑

| Python | Rust | 비고 |
|---|---|---|
| `s.split(",")` | `s.split(',')` | 이터레이터 반환, `.collect::<Vec<_>>()` 필요 시 |
| `s.split()` (공백) | `s.split_whitespace()` | 연속 공백 처리 |
| `s.rsplit(",", 1)` | `s.rsplitn(2, ',')` | |
| `s.strip()` | `s.trim()` | |
| `s.lstrip()` / `s.rstrip()` | `s.trim_start()` / `s.trim_end()` | |
| `s.replace("a", "b")` | `s.replace("a", "b")` | 새 String 반환 |
| `s.upper()` / `s.lower()` | `s.to_uppercase()` / `s.to_lowercase()` | 유니코드 대응 |
| `s.startswith("ab")` | `s.starts_with("ab")` | |
| `s.endswith(".py")` | `s.ends_with(".py")` | |
| `"ab" in s` | `s.contains("ab")` | |
| `s.find("x")` | `s.find('x')` | `Option<usize>` 반환 (없으면 `None`) |
| `len(s)` | `s.len()` / `s.chars().count()` | `len`=바이트 수, `chars().count()`=글자 수 |
| `s * 3` | `s.repeat(3)` | |
| `int(s)` | `s.parse::<i64>()?` | `Result` 반환 |
| `str(n)` | `n.to_string()` | |
| `s[0]` | `s.chars().next()` | 인덱싱 불가, 아래 참고 |

### 🚨 슬라이싱 주의 — Python과 가장 크게 다른 지점

```python
s = "héllo"
print(s[0])      # 'h'  — 코드포인트 단위, 항상 동작
print(s[1:4])    # 'éll'
```

```rust
fn main() {
    let s = "héllo";
    // 🚨 컴파일 에러! String/str은 정수 인덱싱을 지원하지 않습니다
    // let c = s[0];  // error[E0277]: the type `str` cannot be indexed by `{integer}`

    // 바이트 슬라이스는 가능하지만 UTF-8 경계를 벗어나면 런타임 패닉
    let bytes = &s[0..1]; // OK: 'h'는 1바이트
    // let bad = &s[0..2]; // 🚨 panic! end byte index 2 is not a char boundary

    // 안전한 방법: chars() 이터레이터로 접근
    let first: Option<char> = s.chars().next();      // Some('h')
    let sub: String = s.chars().skip(1).take(3).collect(); // "éll"
    println!("{first:?} {sub} {bytes}");
}
```

> **핵심 💡** Rust의 `str`은 **UTF-8 바이트 시퀀스**입니다. `s[i]`가 O(1)이려면 글자가 고정 너비여야 하는데 UTF-8은 그렇지 않으므로, Rust는 아예 정수 인덱싱을 금지합니다. 글자 단위는 `chars()`(O(n) 순회), 바이트 단위는 `as_bytes()`로 명시적으로 선택하게 만듭니다. Python의 `s[i]`가 조용히 O(1)인 척하는 것(실제로는 내부 표현에 따라 다름)과 대조됩니다.

---

## C.4 컬렉션 — 타입 매핑

| Python | Rust | 메모리 |
|---|---|---|
| `list` | `Vec<T>` | 힙, 연속 배열 + len/cap |
| `tuple` | `(T, U, ...)` | 스택, 고정 크기 이종 |
| `dict` | `HashMap<K, V>` | 힙, 해시 테이블 |
| `dict`(삽입 순서 보존 활용) | `IndexMap<K, V>` (indexmap 크레이트) | 순서 보존 해시맵 |
| 정렬된 dict | `BTreeMap<K, V>` | 키 정렬 트리 |
| `set` | `HashSet<T>` | 힙, 해시 셋 |
| 정렬된 set | `BTreeSet<T>` | 키 정렬 트리 |
| `frozenset` | `HashSet<T>`(불변 바인딩으로) | |
| `collections.deque` | `VecDeque<T>` | 링 버퍼 |
| `bytes` | `Vec<u8>` / `&[u8]` | |
| `array.array` | `[T; N]` (고정) / `Vec<T>` | 스택 배열은 컴파일 타임 크기 |

```rust
use std::collections::{HashMap, HashSet, BTreeMap, VecDeque};

fn main() {
    let v: Vec<i32> = vec![1, 2, 3];
    let t: (i32, &str, f64) = (1, "two", 3.0);
    let mut m: HashMap<&str, i32> = HashMap::new();
    m.insert("a", 1);
    let s: HashSet<i32> = [1, 2, 2, 3].into_iter().collect(); // 중복 자동 제거 → {1,2,3}
    let sorted: BTreeMap<i32, &str> = BTreeMap::new();
    let mut dq: VecDeque<i32> = VecDeque::new();
    dq.push_back(1);
    dq.push_front(0);
    println!("{v:?} {t:?} {m:?} {s:?} {sorted:?} {dq:?}");
}
```

---

## C.5 컬렉션 — 메서드와 컴프리헨션

### 자주 쓰는 메서드 매핑

| Python | Rust |
|---|---|
| `xs.append(x)` | `xs.push(x);` |
| `xs.extend(ys)` | `xs.extend(ys);` |
| `xs.insert(i, x)` | `xs.insert(i, x);` |
| `xs.pop()` | `xs.pop()` (`Option<T>` 반환) |
| `xs.remove(x)`(값) | `xs.retain(\|&v\| v != x);` 또는 위치 찾아 `remove` |
| `del xs[i]` | `xs.remove(i);` |
| `len(xs)` | `xs.len()` |
| `x in xs` | `xs.contains(&x)` |
| `xs.index(x)` | `xs.iter().position(\|&v\| v == x)` (`Option`) |
| `xs.count(x)` | `xs.iter().filter(\|&&v\| v == x).count()` |
| `sorted(xs)` | `{ let mut s = xs.clone(); s.sort(); s }` |
| `xs.sort()` (제자리) | `xs.sort();` |
| `xs.sort(reverse=True)` | `xs.sort_by(\|a, b\| b.cmp(a));` |
| `sorted(xs, key=f)` | `xs.sort_by_key(\|x\| f(x));` |
| `xs.reverse()` | `xs.reverse();` |
| `list(reversed(xs))` | `xs.iter().rev().collect::<Vec<_>>()` |
| `sum(xs)` | `xs.iter().sum::<i64>()` |
| `min(xs)` / `max(xs)` | `xs.iter().min()` / `.max()` (`Option`) |
| `any(p(x) for x in xs)` | `xs.iter().any(\|x\| p(x))` |
| `all(p(x) for x in xs)` | `xs.iter().all(\|x\| p(x))` |
| `zip(a, b)` | `a.iter().zip(b.iter())` |
| `enumerate(xs)` | `xs.iter().enumerate()` |
| `map(f, xs)` | `xs.iter().map(\|x\| f(x))` |
| `filter(p, xs)` | `xs.iter().filter(\|x\| p(x))` |

### 컴프리헨션 → 이터레이터 체인

| Python | Rust |
|---|---|
| `[x*2 for x in xs]` | `xs.iter().map(\|x\| x * 2).collect::<Vec<_>>()` |
| `[x for x in xs if x > 0]` | `xs.iter().filter(\|&&x\| x > 0).copied().collect::<Vec<_>>()` |
| `[f(x) for x in xs if p(x)]` | `xs.iter().filter(\|x\| p(x)).map(\|x\| f(x)).collect()` |
| `{k: v for k, v in pairs}` | `pairs.into_iter().collect::<HashMap<_, _>>()` |
| `{x for x in xs}` | `xs.iter().copied().collect::<HashSet<_>>()` |
| `(x*x for x in xs)` (제너레이터) | `xs.iter().map(\|x\| x * x)` (지연 평가, collect 안 함) |
| `sum(x*x for x in xs)` | `xs.iter().map(\|x\| x * x).sum::<i64>()` |
| 중첩 `[y for x in xss for y in x]` | `xss.iter().flatten().collect::<Vec<_>>()` |
| `[[..] for ..]` 펼치기 | `.flat_map(\|x\| ...)` |

```rust
fn main() {
    let xs = vec![1, 2, 3, 4, 5, 6];

    // [x*x for x in xs if x % 2 == 0]
    let evens_squared: Vec<i32> = xs
        .iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .collect();
    println!("{evens_squared:?}"); // [4, 16, 36]

    // 컴파일러는 이 체인을 손으로 짠 for 루프와 동일한 기계어로 인라이닝합니다 (Chapter 15)
}
```

> **Under the Hood 💡** `filter().map().collect()`는 `collect()`가 호출되는 순간에만 실제 순회가 일어납니다. 중간 `Vec`을 만들지 않고 원소 하나가 파이프라인 전체를 통과합니다. Python의 리스트 컴프리헨션이 즉시 전체 리스트를 메모리에 올리는 것과 달리, 제너레이터 표현식에 가깝게 동작하면서도 인터프리터 오버헤드가 0입니다.

`iter()`, `iter_mut()`, `into_iter()`의 선택은 컴프리헨션을 옮길 때 가장 자주 막히는 지점입니다. `iter()`는 `&T`를, `iter_mut()`는 `&mut T`를, `into_iter()`는 `T`(소유권)를 내놓습니다. 원본을 그대로 두고 읽기만 하면 `iter()`, 결과를 새 컬렉션으로 옮겨 담아 원본이 더 필요 없으면 `into_iter()`를 쓰는 식입니다. 위 예제에서 `filter`가 받는 클로저 인자가 `&&x`(이중 참조)인 이유도 여기 있습니다. `iter()`가 `&i32`를 주고, `filter`는 다시 그것을 빌려 `&&i32`로 넘기기 때문입니다. 이때 `|&&x|`로 패턴 매칭해 풀거나, 정수처럼 `Copy` 타입이면 `.copied()`/`.cloned()`로 값을 복사해 다음 단계의 부담을 덜어 줍니다. 이 한 번의 막힘만 넘기면 나머지 어댑터 체인은 파이썬 제너레이터처럼 술술 읽힙니다(Chapter 15).

---

## C.6 딕셔너리 관용구

| Python | Rust |
|---|---|
| `d[k]` | `d[&k]` (없으면 panic) 또는 `d.get(&k)` (`Option`) |
| `d.get(k)` | `d.get(&k)` → `Option<&V>` |
| `d.get(k, default)` | `d.get(&k).copied().unwrap_or(default)` |
| `d[k] = v` | `d.insert(k, v);` |
| `k in d` | `d.contains_key(&k)` |
| `del d[k]` | `d.remove(&k);` |
| `d.setdefault(k, []).append(x)` | `d.entry(k).or_default().push(x);` |
| `d.get(k, 0) + 1` 후 대입 | `*d.entry(k).or_insert(0) += 1;` |
| `d.keys()` | `d.keys()` |
| `d.values()` | `d.values()` |
| `d.items()` | `d.iter()` → `(&K, &V)` |
| `for k, v in d.items():` | `for (k, v) in &d {` |
| `dict(zip(ks, vs))` | `ks.into_iter().zip(vs).collect::<HashMap<_,_>>()` |
| `d.update(other)` | `d.extend(other);` |
| `defaultdict(list)` | `HashMap` + `entry(k).or_default()` |
| `defaultdict(int)` | `HashMap` + `entry(k).or_insert(0)` |
| `collections.Counter(xs)` | `entry` 루프 (아래 C.11) |

```rust
use std::collections::HashMap;

fn main() {
    let mut groups: HashMap<&str, Vec<i32>> = HashMap::new();
    let data = [("a", 1), ("b", 2), ("a", 3)];

    // Python: groups.setdefault(k, []).append(v)
    for (k, v) in data {
        groups.entry(k).or_default().push(v);
    }
    println!("{groups:?}"); // {"a": [1, 3], "b": [2]}
}
```

`entry` API는 Python의 `setdefault`/`defaultdict`를 대체하는 핵심 도구입니다. 키를 한 번만 해싱하고, 존재 여부에 따라 분기하는 비용을 한 번의 룩업으로 합칩니다(Chapter 14).

---

## C.7 파일 I/O

| Python | Rust |
|---|---|
| `open(p).read()` | `std::fs::read_to_string(p)?` |
| `open(p, "rb").read()` | `std::fs::read(p)?` → `Vec<u8>` |
| `open(p, "w").write(s)` | `std::fs::write(p, s)?` |
| `with open(p) as f:` | `let f = File::open(p)?;` (스코프 종료 시 자동 close) |
| `for line in f:` | `for line in BufReader::new(f).lines() {` |
| `f.readlines()` | `reader.lines().collect::<Result<Vec<_>, _>>()?` |
| `f.write(s)` (반복) | `BufWriter` + `write_all` |
| `os.path.exists(p)` | `std::path::Path::new(p).exists()` |
| `json.load(f)` | `serde_json::from_reader(f)?` |

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader, BufWriter, Write};

fn main() -> io::Result<()> {
    // with open("in.txt") as f: for line in f: ...
    let file = File::open("in.txt")?;
    let reader = BufReader::new(file);

    let out = File::create("out.txt")?;
    let mut writer = BufWriter::new(out); // 버퍼링: syscall 횟수를 줄임

    for line in reader.lines() {
        let line = line?; // 각 줄도 io::Result
        if !line.trim().is_empty() {
            writeln!(writer, "{}", line.to_uppercase())?;
        }
    }
    writer.flush()?; // 명시적 flush (Drop 시 자동이지만 에러를 잡으려면 권장)
    Ok(())
}
```

Python의 `with`는 `__exit__`로 파일을 닫지만, Rust는 `File`이 스코프를 벗어나면 `Drop`이 자동으로 닫습니다. `?` 연산자는 Python의 예외 전파를 대신해 에러를 호출자에게 올려보냅니다(Chapter 12).

---

## C.8 에러 처리

| Python | Rust |
|---|---|
| `try: ... except E: ...` | `match expr { Ok(v) => ..., Err(e) => ... }` |
| `try: x = f() except: x = d` | `let x = f().unwrap_or(d);` |
| `raise ValueError("msg")` | `return Err(MyError::Invalid("msg".into()));` |
| `raise` (재전파) | `?` 연산자 또는 `return Err(e);` |
| 예외 전파(자동) | `let v = f()?;` (명시적) |
| `finally:` | `Drop` 트레이트 (RAII) 또는 스코프 가드 |
| `x = d.get(k)` (없으면 None) | `let x: Option<_> = d.get(&k);` |
| `assert cond` | `assert!(cond);` |
| `try: ... except (A, B):` | `enum`으로 에러 통합 후 `match` |
| `except E as e: log(e)` | `Err(e) => log(e),` |
| 컨텍스트 추가 (`raise ... from`) | `.map_err(\|e\| ...)?` 또는 anyhow `.context(...)` |

```rust
use std::num::ParseIntError;

// "예외를 던진다"가 아니라 "실패를 타입으로 반환한다"
fn parse_and_double(s: &str) -> Result<i64, ParseIntError> {
    let n: i64 = s.parse()?; // 실패 시 즉시 Err 반환 (Python의 raise)
    Ok(n * 2)
}

fn main() {
    match parse_and_double("21") {
        Ok(v) => println!("ok: {v}"),     // ok: 42
        Err(e) => eprintln!("err: {e}"),
    }
    // unwrap_or: 예외를 삼키고 기본값
    let v = parse_and_double("oops").unwrap_or(-1);
    println!("{v}"); // -1
}
```

> **핵심 💡** Rust에는 `try/except`가 없습니다. 실패 가능성은 반환 타입 `Result<T, E>`에 박혀 있어 **시그니처만 봐도 실패 여부를 압니다**. `Option<T>`는 "값이 없을 수 있음"(Python의 `None`), `Result<T, E>`는 "연산이 실패할 수 있음"을 표현합니다(Chapter 12, 13).

---

## C.9 타입과 클래스

| Python | Rust |
|---|---|
| `@dataclass class P:` | `struct P { ... }` + `#[derive(...)]` |
| `class P: def __init__(self, ...)` | `impl P { fn new(...) -> Self }` |
| `self.x` | `self.x` (메서드 내) |
| 메서드 `def m(self):` | `fn m(&self)` (불변) / `fn m(&mut self)` (가변) |
| 정적 메서드 `@staticmethod` | 연관 함수 `fn f()` (self 없음) |
| 클래스 메서드 `@classmethod` | 연관 함수 `fn from_x(...) -> Self` |
| `@property` | getter 메서드 `fn x(&self) -> &T { &self.x }` |
| `Enum` | `enum E { A, B, C }` |
| 데이터 있는 Enum | `enum E { Move { x: i32 }, Quit }` (대수적 자료형) |
| `isinstance(x, Dog)` | `match x { Animal::Dog(d) => ... }` |
| 상속 `class B(A):` | 컴포지션 + 트레이트 (상속 없음, Chapter 8) |
| 추상 클래스 `ABC` | `trait` |
| `__repr__` | `#[derive(Debug)]` |
| `__eq__` | `#[derive(PartialEq)]` |
| `__hash__` | `#[derive(Hash)]` |
| `__lt__` 등 비교 | `#[derive(PartialOrd, Ord)]` |
| 복사 `copy.copy` | `#[derive(Clone)]` + `.clone()` |
| `__call__` | `Fn`/`FnMut`/`FnOnce` 트레이트 구현 |

```rust
// @dataclass(frozen=True) 의 Rust 대응
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

impl Point {
    // @classmethod / __init__ 대응
    fn new(x: f64, y: f64) -> Self {
        Point { x, y }
    }
    // @property 대응 (읽기 전용 접근자)
    fn magnitude(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}

// isinstance 분기 → enum + match
enum Shape {
    Circle(f64),
    Rect { w: f64, h: f64 },
}

fn area(s: &Shape) -> f64 {
    match s {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rect { w, h } => w * h,
    }
}

fn main() {
    let p = Point::new(3.0, 4.0);
    println!("{}", p.magnitude()); // 5
    println!("{}", area(&Shape::Circle(1.0)));
}
```

`isinstance` 체인은 Rust에서 **`enum` + `match`**로 바꾸는 것이 정석입니다. 컴파일러가 모든 변형(Variant)을 다뤘는지 망라성(Exhaustiveness)을 검사하므로, 새 타입을 추가하면 처리를 빠뜨린 곳이 컴파일 에러로 드러납니다(Chapter 10).

상속을 컴포지션과 트레이트로 옮기는 작업은 클래스 기반 설계에 익숙한 시니어 파이썬 개발자가 가장 크게 사고를 전환해야 하는 부분입니다. 파이썬에서 `class Manager(Employee)`로 동작과 데이터를 함께 물려받던 것을, Rust는 두 축으로 분해합니다. 공유할 **데이터**는 필드로 품고(컴포지션), 공유할 **동작**은 `trait`로 추출해 각 타입이 구현합니다. "is-a" 관계를 "has-a"와 "can-do"로 쪼개는 셈입니다. 다형성이 필요하면 정적 디스패치(제네릭 `<T: MyTrait>`)와 동적 디스패치(`Box<dyn MyTrait>`) 중 고르는데, 전자는 단일형화로 인라이닝되어 가상 호출 비용이 없고 후자는 파이썬의 메서드 조회처럼 런타임에 결정됩니다. 대부분은 정적 디스패치로 충분하고, 이질적 타입을 한 컬렉션에 담아야 할 때만 `dyn`을 꺼냅니다(Chapter 8, 9).

---

## C.10 함수형 — 이터레이터 어댑터

| Python | Rust |
|---|---|
| `map(f, xs)` | `xs.iter().map(f)` |
| `filter(p, xs)` | `xs.iter().filter(\|x\| p(x))` |
| `functools.reduce(f, xs, init)` | `xs.iter().fold(init, f)` |
| `functools.reduce(f, xs)` | `xs.iter().copied().reduce(f)` (`Option`) |
| `sorted(xs, key=f)` | `xs.sort_by_key(\|x\| f(x))` |
| `sorted(xs, key=f, reverse=True)` | `xs.sort_by_key(...); xs.reverse();` 또는 `sort_by` |
| `itertools.chain(a, b)` | `a.iter().chain(b.iter())` |
| `itertools.zip_longest` | `itertools::zip_longest` (itertools 크레이트) |
| `itertools.groupby` | `itertools::Itertools::chunk_by` (정렬 후) |
| `itertools.islice(xs, n)` | `xs.iter().take(n)` |
| `itertools.count()` | `(0..)` (무한 범위) |
| `itertools.cycle(xs)` | `xs.iter().cycle()` |
| `itertools.takewhile(p, xs)` | `xs.iter().take_while(\|x\| p(x))` |
| `itertools.dropwhile(p, xs)` | `xs.iter().skip_while(\|x\| p(x))` |
| `itertools.accumulate` | `xs.iter().scan(0, \|s, &x\| { *s += x; Some(*s) })` |
| `itertools.product` | `itertools::iproduct!` (itertools 크레이트) |
| `enumerate(xs, start=1)` | `xs.iter().enumerate().map(\|(i, x)\| (i + 1, x))` |
| `xs[::-1]` | `xs.iter().rev()` |
| `xs[:n]` | `xs.iter().take(n)` / `&xs[..n]` |
| `xs[n:]` | `xs.iter().skip(n)` / `&xs[n..]` |

```rust
fn main() {
    let nums = vec![3, 1, 4, 1, 5, 9, 2, 6];

    // functools.reduce(lambda a, b: a + b, nums, 0)
    let total: i32 = nums.iter().fold(0, |acc, &x| acc + x);

    // sorted(nums, key=lambda x: -x)[:3]  → 상위 3개
    let mut sorted = nums.clone();
    sorted.sort_by(|a, b| b.cmp(a)); // 내림차순
    let top3: Vec<i32> = sorted.into_iter().take(3).collect();

    println!("{total} {top3:?}"); // 31 [9, 6, 5]
}
```

---

## C.11 자주 쓰는 실전 패턴

데이터 처리 코드에서 거의 매번 등장하는 네 가지 관용구입니다. 파이썬에서는 `collections`가 한 줄로 처리해 주던 것들인데, Rust에서는 `HashMap`의 `entry` API와 이터레이터 조합으로 똑같이 간결하게, 그러나 할당 없이 풀어냅니다.

### 카운터 (Counter → HashMap entry)

```python
from collections import Counter
counts = Counter(words)
# 또는: counts[w] = counts.get(w, 0) + 1
```

```rust
use std::collections::HashMap;

fn count_words(words: &[&str]) -> HashMap<&str, usize> {
    let mut counts = HashMap::new();
    for &w in words {
        // Counter의 핵심: 없으면 0으로 초기화 후 증가
        *counts.entry(w).or_insert(0) += 1;
    }
    counts
}

fn main() {
    let words = ["a", "b", "a", "c", "a", "b"];
    let counts = count_words(&words);
    println!("{counts:?}"); // {"a": 3, "b": 2, "c": 1}
}
```

### 정렬 + 다중 키 (sort with multiple keys)

```python
# 이름 오름차순, 같으면 나이 내림차순
people.sort(key=lambda p: (p.name, -p.age))
```

```rust
#[derive(Debug)]
struct Person { name: String, age: u32 }

fn main() {
    let mut people = vec![
        Person { name: "Bob".into(), age: 30 },
        Person { name: "Bob".into(), age: 25 },
        Person { name: "Amy".into(), age: 40 },
    ];
    // 이름 오름차순, 같으면 나이 내림차순
    people.sort_by(|a, b| {
        a.name.cmp(&b.name).then(b.age.cmp(&a.age))
    });
    println!("{people:?}");
    // Amy(40), Bob(30), Bob(25)
}
```

### 그룹핑 (groupby)

```python
from collections import defaultdict
groups = defaultdict(list)
for item in items:
    groups[item.key].append(item)
```

```rust
use std::collections::HashMap;

fn main() {
    let items = [(1, "a"), (2, "b"), (1, "c"), (2, "d")];
    let mut groups: HashMap<i32, Vec<&str>> = HashMap::new();
    for (key, val) in items {
        groups.entry(key).or_default().push(val);
    }
    println!("{groups:?}"); // {1: ["a", "c"], 2: ["b", "d"]}
}
```

### 중복 제거 (dedup)

```python
unique = list(set(xs))          # 순서 미보존
unique = list(dict.fromkeys(xs)) # 순서 보존
```

```rust
use std::collections::HashSet;

fn main() {
    let xs = vec![1, 3, 2, 3, 1, 4];

    // 순서 미보존
    let unique: HashSet<i32> = xs.iter().copied().collect();

    // 순서 보존 (이미 본 값은 건너뜀)
    let mut seen = HashSet::new();
    let ordered: Vec<i32> = xs
        .iter()
        .copied()
        .filter(|x| seen.insert(*x)) // insert는 새 값이면 true
        .collect();

    println!("{:?} {ordered:?}", unique.len()); // 4 [1, 3, 2, 4]
}
```

---

## C.12 표준 라이브러리 / 모듈 매핑

파이썬의 강력함은 "배터리 포함(Batteries Included)" 표준 라이브러리에서 나옵니다. Rust는 표준 라이브러리를 의도적으로 작게 유지하고, 직렬화·정규식·날짜·HTTP 같은 영역을 잘 다듬어진 크레이트 생태계에 맡깁니다. 그래서 모듈 매핑의 절반은 "표준 라이브러리 어디"가 아니라 "어떤 크레이트"인지를 아는 일입니다. 아래 표의 오른쪽 칸을 `Cargo.toml`의 `[dependencies]`에 추가하는 것이 파이썬 `import`에 대응합니다.

| Python 모듈 | Rust 크레이트/모듈 | 비고 |
|---|---|---|
| `json` | `serde_json` | `#[derive(Serialize, Deserialize)]` |
| `re` | `regex` | 컴파일된 정규식, 백트래킹 없음(선형 시간) |
| `datetime` / `time` | `chrono` / `std::time` | `std::time`은 단조 시계/Duration |
| `os.environ` | `std::env::var` | `Result` 반환 |
| `sys.argv` | `std::env::args` | |
| `sys.exit(1)` | `std::process::exit(1)` | |
| `os.path` / `pathlib` | `std::path::{Path, PathBuf}` | |
| `logging` | `log` + `env_logger` / `tracing` | `tracing`은 구조적·비동기 친화 |
| `subprocess` | `std::process::Command` | |
| `random` | `rand` | `rand::random`, `thread_rng` |
| `math` | `f64` 메서드 / `std::f64::consts` | `(x).sqrt()`, `PI` |
| `argparse` / `click` | `clap` | `#[derive(Parser)]` |
| `requests` / `httpx` | `reqwest` | async 기본, blocking 옵션 |
| `pytest` | `#[test]` + `cargo test` | 내장 테스트 러너 |
| `dataclasses` | `struct` + `#[derive]` | |
| `enum` | `enum` | |
| `typing` | 타입 시스템 내장 | `Option`, 제네릭 |
| `collections.Counter` | `HashMap` + `entry` | C.11 참고 |
| `itertools` | `std::iter` + `itertools` 크레이트 | |
| `functools.lru_cache` | `once_cell` / `memoize` 크레이트 | |
| `asyncio` | `tokio` / `async-std` | Chapter 18 |
| `multiprocessing` | `std::thread` / `rayon` | 진짜 병렬, GIL 없음 (Chapter 16) |
| `csv` | `csv` | serde 연동 |
| `hashlib` | `sha2` / `md-5` | |
| `base64` | `base64` | |
| `uuid` | `uuid` | |

```rust
// 표준 라이브러리만으로: 환경 변수 + 인자 + 수학 + 프로세스
fn main() {
    let path = std::env::var("PATH").unwrap_or_default(); // os.environ.get
    let args: Vec<String> = std::env::args().collect();   // sys.argv
    let root = (2.0_f64).sqrt();                          // math.sqrt
    println!("args={} path_len={} sqrt2={root:.4}", args.len(), path.len());
    // std::process::exit(0); // sys.exit
}
```

### serde_json 한눈에

```python
import json
data = json.loads('{"name": "Amy", "age": 30}')
text = json.dumps(data)
```

```rust
// Cargo.toml: serde = { version = "1", features = ["derive"] }, serde_json = "1"
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u32,
}

fn main() -> Result<(), serde_json::Error> {
    let data: User = serde_json::from_str(r#"{"name": "Amy", "age": 30}"#)?;
    let text = serde_json::to_string(&data)?;
    println!("{data:?}\n{text}");
    Ok(())
}
```

---

## C.13 빠른 의사결정 가이드

마지막으로, Python 습관에서 막힐 때 떠올릴 한 줄 규칙입니다.

| 상황 | Python 본능 | Rust 정답 |
|---|---|---|
| 값이 없을 수도 있다 | `None` 반환 | `Option<T>` |
| 실패할 수도 있다 | 예외 발생 | `Result<T, E>` |
| 함수에 값을 넘기는데 계속 쓰고 싶다 | 그냥 넘김 | 빌림 `&x` |
| 여러 타입 중 하나다 | `isinstance` 분기 | `enum` + `match` |
| 공통 동작을 공유한다 | 상속 | `trait` (Chapter 8, 9) |
| 컬렉션을 변형한다 | 컴프리헨션 | 이터레이터 체인 + `collect` |
| 키가 없으면 기본값 | `defaultdict` / `setdefault` | `entry().or_default()` |
| 문자열 i번째 글자 | `s[i]` | `s.chars().nth(i)` |
| 전역 가변 상태 | 그냥 전역 변수 | `Mutex`/`OnceLock`(Chapter 17) |
| 병렬 처리 | `multiprocessing` | `rayon`의 `par_iter` (Chapter 16) |

치트시트는 출발점일 뿐입니다. 표의 오른쪽 칸이 "왜 그렇게 생겼는지"는 본문 각 챕터가 메모리 레이아웃과 어셈블리 수준에서 증명합니다. 변환하다 막히면, 그 막힘은 거의 언제나 **소유권을 누가 갖는가**라는 질문으로 환원됩니다. 그 질문에 답하는 순간 Rust 코드는 자연스럽게 흘러나옵니다.

---

## References

- https://doc.rust-lang.org/book/
- https://doc.rust-lang.org/rust-by-example/
- https://doc.rust-lang.org/std/index.html
- https://cheats.rs/
- https://doc.rust-lang.org/reference/
