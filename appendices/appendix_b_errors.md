# 부록 B. 컴파일러 에러 해독집

파이썬에서 에러는 **런타임에** 옵니다. `AttributeError: 'NoneType' object has no attribute 'name'`은 배포 후 새벽 2시, 특정 입력에서만 터집니다. Rust에서는 정반대입니다. 에러는 프로그램이 실행되기 전에 컴파일러가 거의 모두 잡습니다. 그래서 학습 초반 체감은 "코드 쓰는 시간보다 컴파일러와 싸우는 시간이 더 길다"입니다. 하지만 이 싸움은 파이썬이라면 런타임에 만났을 버그를 미리 보는 것입니다. **borrow checker와의 싸움에서 이기면 동시에 use-after-free·data race·null 역참조와의 싸움에서 이긴 것입니다.**

이 부록은 시니어 파이썬 개발자가 Rust 첫 몇 주에 가장 자주 만나는 컴파일러 에러를 주제별로 모은 레퍼런스입니다. 각 에러마다 (1) 에러 코드, (2) 실제 `rustc` 메시지 발췌, (3) 원인, (4) 파이썬 멘탈 모델과의 차이, (5) `before / after` 해결 코드를 정리했습니다. 메시지는 모두 실제 컴파일러 출력입니다(Rust 2021 에디션). 출력 라인은 짧게 발췌하기 위해 일부 줄을 생략(`...`)했습니다.

> 💡 이 부록은 처음부터 읽는 글이 아니라 **컴파일이 막혔을 때 에러 코드로 찾아오는 색인**입니다. 각 항목 끝에 깊이 다루는 본문 챕터를 적어 두었으니, 같은 에러를 반복한다면 그 챕터로 돌아가세요.

---

## B.0 에러를 빠르게 읽는 법

Rust 에러는 파이썬의 한 줄짜리 `TypeError`와 다른 종(種)입니다. 길고 화살표가 많아 처음엔 노이즈처럼 보이지만, 구조를 알면 30초 안에 핵심을 짚습니다. 다음 메시지를 해부해 봅시다.

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{}", first);
  |                    ----- immutable borrow later used here
  |
  = note: ...
  = help: ...

For more information about this error, try `rustc --explain E0502`.
```

이 출력은 5개의 구성 요소로 이루어져 있습니다.

| 구성 요소 | 예시 | 읽는 법 |
|---|---|---|
| **에러 코드** | `error[E0502]` | 에러 종류의 안정적 ID. 검색·`--explain`의 키. |
| **한 줄 요약** | `cannot borrow ... as mutable ...` | 어긴 규칙. 보통 이 한 줄이면 충분. |
| **위치(span)** | `--> src/main.rs:4:5` | `파일:줄:열`. `^^^`/`---`로 정확한 토큰 표시. |
| **note** | `= note: ...` | 왜 그런지에 대한 추가 설명(인과관계). |
| **help** | `= help: ...` / 코드 제안 | **수정 방법.** `+`/`-`로 정확한 패치까지. |

**읽는 순서**: ① 한 줄 요약 → ② 빨간 `^^^`(에러의 핵심 위치) → ③ 회색 `---`(연관 위치, "여기서 빌렸다" 등)로 인과관계를 잇고 → ④ `help`의 수정 제안. 파이썬 traceback을 아래에서 위로 읽던 습관을 버리고 **위에서 아래로** 읽으세요. 첫 줄이 가장 중요합니다.

### `rustc --explain`

에러 코드를 만나면 가장 먼저 칠 명령입니다.

```bash
rustc --explain E0502
```

개념 설명, 잘못된 코드 예제, 고치는 방법을 긴 글로 보여 줍니다. 모든 예외에 공식 튜토리얼이 딸려 오는 셈입니다.

> 🚨 **에러 코드는 영원하지 않습니다.** 일부는 시간이 지나며 통합됩니다. `rustc --explain E0412`를 실행하면 첫 줄에 `#### Note: this error code is no longer emitted by the compiler.`가 나옵니다 — 과거 "타입을 찾을 수 없음"은 `E0412`였으나 최신 컴파일러는 `E0425`로 통합해 출력합니다(B.4). 화면의 코드 번호가 책과 달라도 당황하지 마세요. **요약 메시지가 진짜 신호이고, 코드는 검색용 라벨입니다.**

### `cargo clippy`

`rustc`가 "컴파일되는가"를 본다면, `clippy`는 "더 나은 방법이 있는가"를 보는 린터입니다(파이썬 `flake8`/`pylint`에 해당하나 훨씬 교육적).

```bash
cargo clippy            # 린트 검사
cargo clippy --fix      # 자동 수정 가능한 것은 바로 고침
```

경고에는 `#[warn(clippy::needless_borrow)]` 같은 린트 이름이 붙고 `help`로 더 관용적인 코드를 제안합니다. 켜 두면 시니어 동료가 옆에서 리뷰해 주는 것과 같습니다.

### IDE: `rust-analyzer`

`rust-analyzer`(VS Code 확장)는 위 진단을 **저장하기도 전에** 인라인으로 보여 줍니다. 빨간 밑줄에 마우스를 올리면 메시지와 Quick Fix가 뜹니다. 컴파일-수정 루프가 초 단위로 줄어 학습 초기 생산성 차이가 가장 큰 도구입니다.

---

## B.1 소유권과 이동 (Ownership & Move)

파이썬에서 `b = a`는 *같은 객체의 또 하나의 이름*을 만들 뿐, 둘은 `id()`를 공유하며 모두 살아 있었습니다. Rust의 `let b = a;`는 `Copy`가 아닌 타입에 대해 **소유권을 이동(Move)**시키고, 이동 후 `a`는 무효화됩니다. 이 한 가지가 초반 에러의 절반을 만듭니다. (본문: **Chapter 4 — 소유권**)

### E0382 — borrow of moved value (이동된 값의 사용)

```text
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:4:20
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     let t = s;
  |             - value moved here
4 |     println!("{}", s);
  |                    ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let t = s.clone();
  |              ++++++++
```

**왜 발생하는가.** `let t = s;`에서 힙 데이터의 소유권이 `t`로 이동했습니다. 소유자는 하나여야 하므로 이동 후 `s`는 죽은 이름이 되고, 4번 줄에서 그것을 써서 거부됩니다.

**파이썬 멘탈 모델 차이.** 파이썬 개발자는 `t = s` 후에도 `s`가 살아 있다고 믿습니다 — 둘 다 같은 객체의 참조니까요. Rust는 이를 "참조 둘"이 아니라 "소유권 이전 하나"로 봅니다. 둘 다 살아 있으면 스코프 끝에서 같은 힙을 두 번 해제(double-free)하므로 원본을 무효화해 차단합니다.

```rust
// before — 🚨 E0382
fn main() {
    let s = String::from("hello");
    let t = s;                  // 소유권이 t로 이동
    println!("{}", s);          // s는 이미 죽음
    println!("{}", t);
}
```

```rust
// after — 의도에 따라 셋 중 하나 선택
fn main() {
    let s = String::from("hello");
    let t = s.clone();          // (1) 깊은 복사: 둘 다 독립 소유 (힙 복사 비용)
    println!("{} {}", s, t);

    // (2) 빌려주기: 소유권은 그대로, 읽기만 빌림 (가장 흔함)
    let u = String::from("world");
    let r = &u;
    println!("{} {}", u, r);
}
```

> 대부분의 E0382는 `.clone()`이 아니라 **`&`(참조)**가 정답입니다. `.clone()`은 힙 복사 비용을 숨기므로, "독립 복사본이 필요한가, 잠깐 읽기만 하면 되는가"를 먼저 자문하세요.

### E0505 — cannot move out because borrowed (빌린 동안 이동 불가)

```text
error[E0505]: cannot move out of `s` because it is borrowed
 --> src/main.rs:4:13
  |
2 |     let s = String::from("hello");
  |         - binding `s` declared here
3 |     let r = &s;
  |             -- borrow of `s` occurs here
4 |     let t = s;
  |             ^ move out of `s` occurs here
5 |     println!("{}", r);
  |                    - borrow later used here
```

**왜 발생하는가.** `r`이 `s`를 빌린 상태에서 `s`를 이동시키려 했습니다. 이동을 허용하면 `r`은 사라진 값을 가리키는 **dangling reference**가 됩니다. 빌림(`r`)이 살아 있는 동안 원본(`s`)의 이동은 금지됩니다.

**파이썬 멘탈 모델 차이.** 파이썬에는 "빌림" 개념이 없습니다 — 객체는 참조 카운트가 0이 될 때까지 살아 있어 별칭을 어디 만들든 안전합니다. Rust는 GC 대신 "이 참조가 가리키는 값이 그보다 오래 사는가"를 컴파일 타임에 증명해야 하므로 빌린 값의 이동을 금지합니다.

```rust
// before — 🚨 E0505
fn main() {
    let s = String::from("hello");
    let r = &s;        // s를 빌림
    let t = s;         // 빌린 동안 이동 시도 -> dangling 위험
    println!("{}", r);
}
```

```rust
// after — 빌림이 끝난 뒤 이동
fn main() {
    let s = String::from("hello");
    {
        let r = &s;
        println!("{}", r);   // 여기서 r 사용 끝(NLL: 마지막 사용 지점에서 빌림 종료)
    }
    let t = s;               // 이제 안전하게 이동 가능
    println!("{}", t);
}
```

> Rust 2018+의 **NLL(Non-Lexical Lifetimes)** 덕분에 빌림은 스코프 끝이 아니라 **마지막 사용 지점**에서 끝납니다. `r`을 `t = s` 위에서 마지막으로 쓰면 명시적 블록 없이도 통과하는 경우가 많습니다.

---

## B.2 대여 (Borrowing)

소유권을 매번 넘기긴 불편하므로 Rust는 "잠깐 빌려주는" 참조를 제공합니다. 철칙은 하나입니다: **`&T`(불변)는 여러 개 가능하지만 `&mut T`(가변)는 단 하나만, 그리고 가변 참조가 있는 동안 다른 어떤 참조도 불가**. 이 규칙(aliasing XOR mutation)이 data race를 컴파일 타임에 제거합니다. (본문: **Chapter 5 — 참조와 대여**)

### E0502 — 가변·불변 동시 대여

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{}", first);
  |                    ----- immutable borrow later used here
```

**왜 발생하는가.** `first`가 0번 요소를 불변으로 빌린 상태에서 `v.push(4)`는 벡터를 가변으로 빌려야 합니다(`&mut self`). 둘은 공존할 수 없어 거부됩니다.

**파이썬 멘탈 모델 차이.** 파이썬에서 `first = lst[0]` 후 `lst.append(4)`는 일상입니다. 하지만 `push`가 재할당을 유발하면 기존 요소는 새 힙 주소로 이사 가고 `first`가 가리키던 옛 주소는 무효가 됩니다(C++의 iterator invalidation). 파이썬은 런타임에 무탈하게 처리하지만, Rust는 **버그가 될 패턴 자체를 컴파일 타임에 차단**합니다.

```rust
// before — 🚨 E0502
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];     // 불변 빌림
    v.push(4);             // 가변 빌림 시도
    println!("{}", first);
}
```

```rust
// after — 값을 복사해 빌림을 끝냄
fn main() {
    let mut v = vec![1, 2, 3];
    let first = v[0];       // i32는 Copy -> 빌림 없음
    v.push(4);              // 이제 가변 빌림 가능
    println!("{}", first);
}
```

### E0499 — 가변 대여 2회

```text
error[E0499]: cannot borrow `v` as mutable more than once at a time
 --> src/main.rs:4:13
  |
3 |     let a = &mut v;
  |             ------ first mutable borrow occurs here
4 |     let b = &mut v;
  |             ^^^^^^ second mutable borrow occurs here
5 |     a.push(4);
  |     - first borrow later used here
```

**왜 발생하는가.** `&mut v`를 두 번 만들었습니다. 가변 참조는 "나만 바꿀 수 있다"는 독점권이라 동시에 둘 존재할 수 없습니다.

**파이썬 멘탈 모델 차이.** 파이썬에는 "쓰기 권한을 가진 별칭" 제약이 없어, 두 변수가 같은 리스트를 가리키고 둘 다 `append`합니다(멀티스레드에서 미묘한 버그의 근원). Rust의 "가변 참조 단 하나"는 본질적으로 **싱글 라이터(single-writer) 원칙**이며 16~17장 fearless concurrency의 토대입니다.

```rust
// before — 🚨 E0499
fn main() {
    let mut v = vec![1, 2, 3];
    let a = &mut v;
    let b = &mut v;   // 두 번째 가변 빌림
    a.push(4);
    b.push(5);
}
```

```rust
// after — 가변 빌림 수명이 겹치지 않게
fn main() {
    let mut v = vec![1, 2, 3];
    {
        let a = &mut v;
        a.push(4);          // a 사용 끝
    }
    let b = &mut v;         // 새 가변 빌림 가능
    b.push(5);
    println!("{:?}", v);
}
```

### E0596 — 가변 대여가 필요한데 불변으로 선언됨

```text
error[E0596]: cannot borrow `v` as mutable, as it is not declared as mutable
 --> src/main.rs:3:5
  |
3 |     v.push(4);
  |     ^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
2 |     let mut v = vec![1, 2, 3];
  |         +++
```

**왜 발생하는가.** `v.push(4)`는 `&mut self`를 요구하는데, `v`가 `let v`(불변)로 선언돼 가변 참조를 만들 수 없습니다.

**파이썬 멘탈 모델 차이.** 파이썬 변수는 기본적으로 변경 가능합니다. Rust는 **불변이 기본값**이고 가변성은 `mut`로 명시적으로 선택(opt-in)합니다 — 의도를 코드에 박는 문서화이자 최적화 단서입니다. `help`가 `let mut v`로 고치라고 알려 주므로 가장 쉬운 에러입니다.

```rust
// before — 🚨 E0596
fn main() {
    let v = vec![1, 2, 3];
    v.push(4);
}
```

```rust
// after — mut 추가
fn main() {
    let mut v = vec![1, 2, 3];
    v.push(4);
    println!("{:?}", v);
}
```

### E0507 — 빌린 값에서 이동 (move out of borrowed content)

```text
error[E0507]: cannot move out of `w.name` which is behind a shared reference
 --> src/main.rs:3:5
  |
3 |     w.name
  |     ^^^^^^ move occurs because `w.name` has type `String`, which does not implement the `Copy` trait
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     w.name.clone()
  |           ++++++++
```

**왜 발생하는가.** 함수가 `&Wrapper`(불변 참조)만 받았는데 그 안의 `String` 필드를 통째로 꺼내 반환하려 했습니다. 빌린 데이터의 일부를 이동시키면 원래 소유자에게 "구멍 난" 값이 남으므로 금지됩니다.

**파이썬 멘탈 모델 차이.** 파이썬에서 `return w.name`은 또 다른 참조를 돌려줄 뿐 원본 `w`는 멀쩡합니다. Rust에서는 참조로 받은 값의 소유권을 빼앗을 수 없습니다 — 빌린 자는 읽거나 빌린 채로 수정할 수 있을 뿐 가져갈 수는 없습니다.

```rust
// before — 🚨 E0507
struct Wrapper { name: String }
fn take(w: &Wrapper) -> String {
    w.name              // 빌린 w에서 String을 이동시키려 함
}
fn main() {
    let w = Wrapper { name: String::from("x") };
    let _ = take(&w);
}
```

```rust
// after — 복제, 소유권 받기, 참조 반환 중 선택
struct Wrapper { name: String }

fn take_clone(w: &Wrapper) -> String {
    w.name.clone()              // (1) 복제: 원본 보존
}
fn take_owned(w: Wrapper) -> String {
    w.name                      // (2) 소유권을 받으면 이동 허용
}
fn take_ref(w: &Wrapper) -> &str {
    &w.name                     // (3) 참조만 반환 (가장 저렴)
}
fn main() {
    let w = Wrapper { name: String::from("x") };
    println!("{}", take_clone(&w));
    println!("{}", take_ref(&w));
    println!("{}", take_owned(w));  // w 소유권 소진
}
```

> 컬렉션에서 요소를 꺼내야 한다면 `std::mem::take`/`replace`, `Option::take`, `Vec::remove`/`HashMap::remove`를 쓰세요. "꺼낸 자리에 기본값을 채워 구멍을 메우는" 방식입니다.

---

## B.3 수명 (Lifetimes)

참조는 가리키는 값보다 오래 살 수 없습니다. 컴파일러가 이를 자동 추론하지 못할 때 **수명 표기(lifetime annotation)**로 의도를 설명하라고 요구합니다. 수명은 새 코드를 만들지 않고, 이미 존재하는 참조들의 관계를 *기술*할 뿐입니다. (본문: **Chapter 6 — 수명**)

### E0106 — missing lifetime specifier (수명 표기 누락)

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

**왜 발생하는가.** 반환 참조가 `x`에서 빌린 것인지 `y`에서 빌린 것인지 시그니처만으론 알 수 없습니다. 호출자는 "반환 참조가 얼마나 유효한가"를 알아야 하므로, 수명 매개변수로 그 관계를 명시하라고 요구합니다.

**파이썬 멘탈 모델 차이.** 파이썬에는 수명 개념이 없습니다 — 반환 객체는 참조 카운트가 살아 있는 한 유효합니다. Rust는 GC가 없어 "반환 참조 유효 기간 = 입력 참조 중 하나의 유효 기간"이라는 계약을 타입 시스템에 박아야 합니다. `<'a>`는 제네릭처럼 동작하는 "수명 매개변수"이고, `help`가 정확한 패치를 보여 줍니다.

```rust
// before — 🚨 E0106
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

```rust
// after — 'a로 입력과 출력의 수명 관계를 명시
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
fn main() {
    let s = longest("apple", "banana");
    println!("{}", s);
}
```

### E0621 — explicit lifetime required (명시적 수명 필요)

```text
error[E0621]: explicit lifetime required in the type of `input`
 --> src/main.rs:3:47
  |
3 |     fn parse(&self, input: &str) -> &'a str { input }
  |                                               ^^^^^ lifetime `'a` required
  |
help: add explicit lifetime `'a` to the type of `input`
  |
3 |     fn parse(&self, input: &'a str) -> &'a str { input }
  |                             ++
```

**왜 발생하는가.** 반환 타입에 `&'a str`("이 참조는 `'a`만큼 산다")이라 선언해 놓고, 정작 `'a`와 무관한(더 짧은) `input: &str`을 반환하려 했습니다. 컴파일러는 `input`의 수명이 `'a` 이상임을 보장하라고 요구합니다.

**파이썬 멘탈 모델 차이.** 파이썬이라면 "들어온 걸 그대로 돌려준다"가 끝입니다. Rust는 반환 참조가 약속한 수명(`'a`)을 지키는지까지 추적합니다. 해결은 `input`도 `'a`에 묶는 것입니다.

```rust
// before — 🚨 E0621
struct Parser<'a> { data: &'a str }
impl<'a> Parser<'a> {
    fn parse(&self, input: &str) -> &'a str { input }
}
```

```rust
// after — input을 'a에 묶음 (input이 구조체만큼 오래 산다)
struct Parser<'a> { data: &'a str }
impl<'a> Parser<'a> {
    fn parse(&self, input: &'a str) -> &'a str { input }
}
fn main() {
    let p = Parser { data: "ctx" };
    let out = p.parse("hello");
    println!("{}", out);
}
```

### E0597 — borrowed value does not live long enough (수명이 짧음)

```text
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("{}", r);
  |                    - borrow later used here
```

**왜 발생하는가.** 안쪽 블록의 지역 변수 `x`를 빌려 바깥 변수 `r`에 저장했는데, 블록이 끝나면 `x`는 해제됩니다. `r`은 해제된 메모리를 가리키는 dangling reference가 됩니다. 컴파일러는 "`x`가 빌린 채로 죽었다(`dropped here while still borrowed`)"고 지적합니다.

**파이썬 멘탈 모델 차이.** 파이썬에서는 블록이 끝나도 객체가 참조되는 한 살아 있습니다. Rust는 스택 변수가 스코프 끝에서 결정적으로 해제되므로 **그것을 가리키는 참조가 더 오래 살면 안 됩니다.** 이것이 C/C++의 가장 흔한 버그(스택 변수 주소 반환)를 막는 지점입니다.

```rust
// before — 🚨 E0597
fn main() {
    let r;
    {
        let x = 5;
        r = &x;          // x를 빌려 바깥 r에 저장
    }                    // 여기서 x 해제 -> r은 dangling
    println!("{}", r);
}
```

```rust
// after — 빌리지 말고 수명을 맞춤
fn main() {
    let x = 5;           // x를 r과 같은(또는 더 긴) 스코프에 둠
    let r = &x;
    println!("{}", r);
}
```

---

## B.4 타입과 트레이트 (Types & Traits)

파이썬의 덕 타이핑("`.quack()`이 있으면 오리다")은 런타임에 메서드를 찾습니다. Rust는 컴파일 타임에 타입과 트레이트 경계(trait bound)를 검증합니다. 이 섹션은 "이 타입이 그 일을 할 능력(트레이트)을 갖췄음을 증명하라"는 요구입니다. (본문: **Chapter 3 — 타입 시스템**, **Chapter 9 — Trait**)

### E0308 — mismatched types (타입 불일치)

```text
error[E0308]: mismatched types
 --> src/main.rs:6:11
  |
6 |     greet(n);
  |     ----- ^ expected `String`, found `&str`
  |     |
  |     arguments to this function are incorrect
  |
note: function defined here
 --> src/main.rs:1:4
  |
1 | fn greet(name: String) {
  |    ^^^^^ ------------
help: try using a conversion method
  |
6 |     greet(n.to_string());
```

**왜 발생하는가.** 함수는 소유 문자열 `String`을 요구하는데 리터럴 `n`은 `&str`(빌린 슬라이스)입니다. 둘은 명백히 다른 타입입니다 — 전자는 힙을 소유하고 후자는 빌린 뷰입니다.

**파이썬 멘탈 모델 차이.** 파이썬에는 문자열이 한 종류(`str`)뿐이고 정수·실수도 자유롭게 섞입니다(`1 + 2.0`). Rust는 **암묵적 변환(coercion)을 거의 하지 않아** `i32`와 `i64`도, `String`↔`&str`도 명시적 변환(`.to_string()`, `.as_str()`, `&`)이 필요합니다. 가장 흔한 변형은 반환 타입 불일치입니다(`expected u32, found usize` — 정수 폭 차이).

```rust
// before — 🚨 E0308
fn greet(name: String) {
    println!("hi {}", name);
}
fn main() {
    let n = "world";   // n: &str
    greet(n);
}
```

```rust
// after — 시그니처를 &str로 받기 (더 유연)
fn greet(name: &str) {
    println!("hi {}", name);
}
fn main() {
    let n = "world";
    greet(n);                 // &str 그대로 OK
    let owned = String::from("there");
    greet(&owned);            // String -> &str 자동 빌림(deref coercion)
}
```

> 함수 인자는 가능하면 `String`보다 `&str`, `Vec<T>`보다 `&[T]`로 받으세요. 호출자가 소유·빌림 어느 쪽이든 넘길 수 있어 유연합니다.

### E0277 — trait bound not satisfied (트레이트 경계 미충족)

```text
error[E0277]: `Point` doesn't implement `std::fmt::Display`
 --> src/main.rs:4:20
  |
4 |     println!("{}", p);
  |                    ^ `Point` cannot be formatted with the default formatter
  |
  = help: the trait `std::fmt::Display` is not implemented for `Point`
  = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
```

**왜 발생하는가.** `{}` 포맷은 `Display` 트레이트를 요구하는데 `Point`는 구현하지 않았습니다.

**파이썬 멘탈 모델 차이.** 파이썬은 어떤 객체든 `print()`에 넘기면 `__str__`/`repr`이 동작합니다. Rust는 사람이 읽을 표현(`Display`)을 **명시적으로 구현**하도록 요구합니다. `note`가 "`{:?}`을 쓰려면 `Debug`를 derive하라"고 안내합니다. 이 에러는 `where T: Trait` 경계가 안 맞을 때도 동일하게 나타납니다.

```rust
// before — 🚨 E0277
struct Point { x: i32, y: i32 }
fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{}", p);
}
```

```rust
// after — 디버그 출력은 derive로, 사용자 표현은 직접 구현
use std::fmt;

#[derive(Debug)]                 // {:?} 가능
struct Point { x: i32, y: i32 }

impl fmt::Display for Point {    // {} 가능
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{}", p);           // Display
    println!("{:?}", p);         // Debug
}
```

### E0599 — method not found (메서드 없음)

```text
error[E0599]: no method named `lenght` found for struct `Vec<{integer}>` in the current scope
 --> src/main.rs:3:15
  |
3 |     let s = v.lenght();
  |               ^^^^^^
  |
help: there is a method `len` with a similar name
  |
3 -     let s = v.lenght();
3 +     let s = v.len();
```

**왜 발생하는가.** `Vec`에 `lenght`는 없습니다(오타). 또는 메서드가 트레이트에 있는데 그 트레이트를 `use`하지 않았을 때도 발생합니다.

**파이썬 멘탈 모델 차이.** 파이썬에서 오타 난 메서드는 **런타임에** `AttributeError`로 터집니다. Rust는 컴파일 타임에 잡고 편집 거리가 가까운 이름(`len`)까지 제안합니다. 트레이트 메서드가 안 보이면 `help`에 "`use some::Trait`을 추가하라"고 나옵니다 — 파이썬에 없는, "메서드는 보이는 트레이트에서만 호출 가능"이라는 Rust 특유의 규칙입니다.

```rust
// before — 🚨 E0599
fn main() {
    let v = vec![1, 2, 3];
    let s = v.lenght();
    println!("{}", s);
}
```

```rust
// after
fn main() {
    let v = vec![1, 2, 3];
    let s = v.len();          // 올바른 메서드명
    println!("{}", s);
}
```

> 트레이트 메서드가 "없다"고 나오면 십중팔구 `use`가 빠진 것입니다. 예: `read_line`엔 `use std::io::BufRead;`, `rayon`의 `.par_iter()`엔 `use rayon::prelude::*;`가 필요합니다.

### E0428 / E0425 — 중복 정의 & 타입을 찾을 수 없음

**E0428 (중복 정의)**

```text
error[E0428]: the name `User` is defined multiple times
 --> src/main.rs:2:1
  |
1 | struct User { id: u32 }
  | ----------- previous definition of the type `User` here
2 | struct User { name: String }
  | ^^^^^^^^^^^ `User` redefined here
  |
  = note: `User` must be defined only once in the type namespace of this module
```

**왜 발생하는가.** 같은 이름의 타입(또는 모듈)을 한 스코프에서 두 번 정의했습니다.

**파이썬 멘탈 모델 차이.** 파이썬에서 `class User:`를 두 번 정의하면 두 번째가 첫 번째를 *조용히 덮어씁니다* — 에러조차 없습니다(무서운 버그의 근원). Rust는 동명 타입의 공존을 명시적으로 거부합니다. 이름을 바꾸거나, 의도가 "확장"이면 enum variant나 별도 모듈로 분리하세요.

```rust
// before — 🚨 E0428
struct User { id: u32 }
struct User { name: String }   // 중복
```

```rust
// after
struct User { id: u32, name: String }   // 하나로 합치거나
// 또는 이름을 구분: struct UserId { id: u32 } / struct UserName { name: String }
fn main() { let _ = User { id: 1, name: String::from("a") }; }
```

**E0425 (예전 코드 E0412, 타입을 찾을 수 없음)** — 선언/임포트하지 않은 타입명을 시그니처 등에 썼을 때입니다.

```text
error[E0425]: cannot find type `Widget` in this scope
 --> src/main.rs:1:18
  |
1 | fn process(item: Widget) -> u32 { 0 }
  |                  ^^^^^^ not found in this scope
```

> 🚨 **에러 코드 진화 주의.** 위 진단은 rustc 1.99.1 edition 2021의 **`E0425`**입니다. 예전 코드 `E0412`는 더 이상 나오지 않고, `rustc --explain E0412`의 첫 줄은 `#### Note: this error code is no longer emitted by the compiler.`입니다. 원인은 동일합니다: **타입명 오타, 정의 누락, `use` 누락.** `use crate::widget::Widget;`를 추가하거나, 제네릭이었다면 `fn process<T>(item: T)`처럼 타입 매개변수를 선언하세요.

---

## B.5 패턴 매칭 (Pattern Matching)

`match`는 **모든 경우를 빠짐없이(exhaustive)** 다루도록 강제됩니다. 이것이 enum 기반 상태 기계를 "잊은 케이스 없이" 안전하게 만듭니다. (본문: **Chapter 10 — 열거형 상태 기계**)

### E0004 — non-exhaustive patterns (불완전한 패턴)

```text
error[E0004]: non-exhaustive patterns: `Direction::East` and `Direction::West` not covered
 --> src/main.rs:4:11
  |
4 |     match d {
  |           ^ patterns `Direction::East` and `Direction::West` not covered
  |
note: `Direction` defined here
 --> src/main.rs:1:6
  |
1 | enum Direction { North, South, East, West }
  |      ^^^^^^^^^                 ----  ---- not covered
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern, ...
```

**왜 발생하는가.** `Direction`에 variant가 4개인데 `match`가 2개만 처리했습니다. 컴파일러는 빠진 `East`, `West`를 정확히 지목합니다.

**파이썬 멘탈 모델 차이.** 파이썬의 `match`(3.10+)나 if/elif 사슬은 경우를 빠뜨려도 경고하지 않습니다. Rust는 enum에 variant를 **추가**할 때 빛납니다: 새 variant를 추가하는 순간, 처리하지 않은 모든 `match`가 컴파일 에러를 냅니다. 컴파일러가 "이 새 상태를 처리해야 할 모든 곳"의 목록을 자동으로 뽑아 주는 셈이라, 거대한 리팩토링이 두렵지 않습니다.

```rust
// before — 🚨 E0004
enum Direction { North, South, East, West }
fn main() {
    let d = Direction::North;
    match d {
        Direction::North => println!("n"),
        Direction::South => println!("s"),
    }
}
```

```rust
// after — 모든 경우를 다룸
enum Direction { North, South, East, West }
fn main() {
    let d = Direction::North;
    match d {
        Direction::North => println!("n"),
        Direction::South => println!("s"),
        Direction::East  => println!("e"),
        Direction::West  => println!("w"),
        // 또는 나머지를 한 번에: _ => println!("other"),
    }
}
```

> `_ => ...` 와일드카드는 편하지만 남용하면 E0004의 안전망을 스스로 끄는 셈입니다 — variant가 추가돼도 경고하지 않기 때문입니다. 상태가 유한하고 중요하면 모든 variant를 명시적으로 나열하세요.

---

## B.6 동시성 (Concurrency)

Rust의 "fearless concurrency"는 두 마커 트레이트의 결과입니다: **`Send`**(다른 스레드로 소유권 이동 가능)와 **`Sync`**(여러 스레드가 참조 공유 안전). 스레드 경계를 넘는 타입이 이를 만족하지 않으면 컴파일러가 data race를 **컴파일 타임에** 거부합니다. (본문: **Chapter 16~17 — 멀티스레딩과 공유 상태**)

### E0277 — Send 미충족 (Rc를 스레드로)

```text
error[E0277]: `Rc<Vec<i32>>` cannot be sent between threads safely
 --> src/main.rs:6:19
  |
6 |     thread::spawn(move || {
  |     ------------- ^------ ... `Rc<Vec<i32>>` cannot be sent between threads safely
  |     |
  |     required by a bound introduced by this call
  |
  = help: within `{closure...}`, the trait `Send` is not implemented for `Rc<Vec<i32>>`
note: required by a bound in `spawn`
```

**왜 발생하는가.** `Rc<T>`는 **싱글 스레드용** 참조 카운트 포인터입니다. 카운트 증감에 원자적 연산을 쓰지 않아 두 스레드가 동시에 건드리면 race로 메모리가 일찍 해제되거나 누수됩니다. 그래서 `Rc`는 `Send`를 구현하지 않고, `thread::spawn`은 클로저가 캡처하는 모든 것이 `Send`이길 요구합니다.

**파이썬 멘탈 모델 차이.** 파이썬은 GIL이 참조 카운트를 보호해 어떤 객체든 스레드 간에 그냥 넘길 수 있습니다(대신 GIL이 CPU 병렬성을 죽입니다). Rust는 GIL이 없어 진짜 병렬이 가능한 대가로 "이 타입이 스레드 간 공유에 안전한가"를 타입 시스템이 따집니다. 해결은 **원자적** 참조 카운트 `Arc<T>`(Atomic Rc)입니다.

```rust
// before — 🚨 E0277 (Rc는 Send가 아님)
use std::rc::Rc;
use std::thread;
fn main() {
    let data = Rc::new(vec![1, 2, 3]);
    let d = Rc::clone(&data);
    thread::spawn(move || {
        println!("{:?}", d);
    });
}
```

```rust
// after — Arc로 교체 (스레드 간 공유 안전)
use std::sync::Arc;
use std::thread;
fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    let d = Arc::clone(&data);
    let h = thread::spawn(move || {
        println!("{:?}", d);     // Arc<Vec<i32>>는 Send
    });
    h.join().unwrap();
}
```

> 쓰기까지 공유한다면 `Arc<Mutex<T>>` 또는 `Arc<RwLock<T>>`를 씁니다 — `Arc`는 "여러 소유자", `Mutex`는 "한 번에 한 명만 쓰기"를 보장합니다. `Send` vs `Sync`의 분리는 17장에서 다룹니다.

---

## B.7 재귀 타입 (Recursive Types)

### E0072 — recursive type has infinite size (무한 크기 재귀 타입)

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

**왜 발생하는가.** Rust는 컴파일 타임에 모든 타입의 크기(바이트)를 알아야 합니다. 그런데 `Node`가 자기 자신을 직접 포함하면 `size(Node) = 4 + size(Node) = ...`로 무한 발산합니다. 컴파일러는 크기를 정할 수 없어 거부합니다.

**파이썬 멘탈 모델 차이.** 파이썬의 링크드 리스트·트리는 자연스럽습니다 — 모든 객체가 힙에 있고 변수는 포인터일 뿐이라 `self.next = node`에 크기 문제가 없습니다. Rust의 `struct` 필드는 기본적으로 **인라인(구조체 내부에 직접 박힘)**이라 재귀 구조에는 명시적 간접 참조(indirection)가 필요합니다. `Box<Node>`는 크기가 고정된(8바이트) 힙 포인터라 무한 발산을 끊습니다. `help`가 `Box`를 제안합니다.

```rust
// before — 🚨 E0072
struct Node {
    value: i32,
    next: Option<Node>,      // 자기 자신을 인라인 포함 -> 무한 크기
}
```

```rust
// after — Box로 힙 간접 참조 삽입
struct Node {
    value: i32,
    next: Option<Box<Node>>, // Box는 8바이트 포인터 -> 크기 확정
}
fn main() {
    let list = Node {
        value: 1,
        next: Some(Box::new(Node { value: 2, next: None })),
    };
    println!("{}", list.value);
}
```

> 공유가 필요한 그래프라면 `Rc<RefCell<Node>>`(싱글 스레드)나 `Arc<Mutex<Node>>`(멀티 스레드)를 씁니다. `Box`/`Rc`/`RefCell`의 역할 분담은 20장에서 다룹니다.

---

## B.8 Unsafe

Rust의 안전 보장은 대부분 자동이지만, 일부 연산(생 포인터 역참조, 가변 정적 변수 접근, FFI 호출 등)은 컴파일러가 안전을 증명할 수 없습니다. 이때 `unsafe` 블록으로 책임 선언을 요구합니다. (본문: **Chapter 21 — C FFI와 Unsafe Rust**)

### E0133 — operation requires unsafe (unsafe 필요)

```text
error[E0133]: use of mutable static is unsafe and requires unsafe function or block
 --> src/main.rs:3:5
  |
3 |     COUNTER += 1;
  |     ^^^^^^^ use of mutable static
  |
  = note: mutable statics can be mutated by multiple threads: aliasing violations or data races will cause undefined behavior
```

**왜 발생하는가.** 가변 정적 변수(`static mut`)는 전역 공유라 여러 스레드가 동기화 없이 동시에 쓰면 data race(UB)가 됩니다. 컴파일러가 이 위험을 자동으로 막을 수 없으므로, `unsafe`로 "내가 안전을 보장한다"고 선언하길 요구합니다.

**파이썬 멘탈 모델 차이.** 파이썬에는 전역 가변 상태(`global x`)가 흔하고 GIL이 바이트코드 단위 원자성을 어느 정도 보장합니다. Rust에는 GIL이 없어 그런 보호가 없습니다. `unsafe`는 "위험한 코드 스위치"가 아니라 **"이 작은 영역의 안전성 증명을 컴파일러가 아니라 내가 한다"는 계약**입니다. 대부분 `static mut`는 안전한 대안(`AtomicUsize`, `Mutex<T>`, `OnceLock`)으로 대체하는 것이 옳습니다.

```rust
// before — 🚨 E0133
static mut COUNTER: u32 = 0;
fn main() {
    COUNTER += 1;
    println!("{}", COUNTER);
}
```

```rust
// after (1) — unsafe가 필요하면 명시적으로 책임 선언
static mut COUNTER: u32 = 0;
fn main() {
    unsafe {
        COUNTER += 1;
        println!("{}", COUNTER);
    }
}
```

```rust
// after (2) — 더 나은 해법: 원자 타입으로 unsafe 제거
use std::sync::atomic::{AtomicU32, Ordering};
static COUNTER: AtomicU32 = AtomicU32::new(0);
fn main() {
    COUNTER.fetch_add(1, Ordering::SeqCst);   // 스레드 안전, unsafe 불필요
    println!("{}", COUNTER.load(Ordering::SeqCst));
}
```

> 🚨 `unsafe`는 borrow checker를 끄지 않습니다. 단지 (1) 생 포인터 역참조, (2) `unsafe fn` 호출, (3) 가변 정적 접근, (4) union 필드 접근, (5) `unsafe trait` 구현 — 이 다섯만 허용합니다. 나머지 안전 규칙은 `unsafe` 안에서도 그대로 적용됩니다. "안전한 추상화로 unsafe를 감싸라"는 원칙은 21장에서 다룹니다.

---

## B.9 자주 만나는 에러 → 본문 챕터 매핑

특정 에러를 반복한다면 증상만 고치지 말고 해당 챕터로 돌아가 멘탈 모델을 다시 잡으세요.

| 에러 코드 | 한 줄 요약 | 핵심 원인 | 본문 챕터 |
|---|---|---|---|
| **E0382** | borrow of moved value | 이동 후 원본 사용 | 4. 소유권 |
| **E0505** | cannot move out because borrowed | 빌린 동안 원본 이동 | 4. 소유권 / 5. 대여 |
| **E0502** | 가변·불변 동시 대여 | aliasing XOR mutation 위반 | 5. 대여 |
| **E0499** | 가변 대여 2회 | single-writer 위반 | 5. 대여 |
| **E0596** | 불변인데 가변 대여 시도 | `mut` 선언 누락 | 5. 대여 |
| **E0507** | move out of borrowed | 참조 통해 소유권 탈취 | 5. 대여 |
| **E0106** | missing lifetime specifier | 반환 참조의 출처 불명 | 6. 수명 |
| **E0621** | explicit lifetime required | 약속한 수명 미보장 | 6. 수명 |
| **E0597** | does not live long enough | dangling reference | 6. 수명 |
| **E0308** | mismatched types | 암묵적 변환 없음 | 3. 타입 시스템 |
| **E0277** | trait bound not satisfied | 필요한 트레이트 미구현 | 9. Trait / 13. 에러 |
| **E0599** | method not found | 오타 또는 `use` 누락 | 9. Trait / 15. 이터레이터 |
| **E0428** | name defined multiple times | 이름 중복 정의 | 7. Struct |
| **E0412 / E0425** | cannot find type/value | 정의·임포트 누락 | 2. 모듈 / 7. Struct |
| **E0004** | non-exhaustive patterns | `match` 케이스 누락 | 10. 열거형 상태 기계 |
| **E0277** (Send/Sync) | cannot be sent between threads | `Rc` 등 비-`Send` 타입 | 16~17. 동시성 |
| **E0072** | recursive type infinite size | 간접 참조 누락 | 20. 스마트 포인터 |
| **E0133** | requires unsafe | unsafe 연산 비명시 | 21. Unsafe |

---

## 맺음말

이 부록의 에러들은 한 발 물러나 보면 **두 개의 질문**으로 수렴합니다.

1. **"이 값을 누가 소유하고 누가 얼마나 오래 빌리는가?"** — E0382·E0505·E0502·E0499·E0596·E0507·E0106·E0621·E0597·E0072.
2. **"이 타입이 그 일을 할 능력을 증명했는가?"** — E0308·E0277·E0599·E0004·E0133.

파이썬은 이 답을 런타임이 떠맡았습니다 — GC가 수명을, 덕 타이핑이 능력을 확인했고, 틀리면 새벽 2시에 알려 주었습니다. Rust는 같은 질문을 컴파일 타임으로 끌어와 답을 못 내면 바이너리를 만들지 않습니다. 초반엔 깐깐한 게이트키퍼처럼 느껴지지만, 익숙해지면 **가장 빠르고 정확한 페어 프로그래머**임을 알게 됩니다. `help` 한 줄이 곧 정답이고, `rustc --explain`은 무료 과외 교사입니다.

같은 에러를 세 번 만났다면 문법 실수가 아니라 멘탈 모델의 빈틈입니다. 표의 챕터로 돌아가세요. 그 빈틈을 메우는 순간, 컴파일러와의 싸움은 협업으로 바뀝니다.

---

## References

- https://doc.rust-lang.org/error_codes/error-index.html
- https://doc.rust-lang.org/error_codes/E0382.html
- https://doc.rust-lang.org/error_codes/E0499.html
- https://doc.rust-lang.org/error_codes/E0277.html
- https://doc.rust-lang.org/error_codes/E0308.html
- https://doc.rust-lang.org/rustc/
