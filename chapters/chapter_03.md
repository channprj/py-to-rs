# Chapter 3. 문법의 이면: 표현식(Expression)과 타입 시스템

파이썬 개발자가 Rust 코드를 처음 마주하면, 중괄호(`{}`)와 세미콜론(`;`)이 여기저기 붙어있는 모습 때문에 C나 Java의 향수를 (혹은 거부감을) 느낄 수 있습니다. 그러나 겉보기에 비슷한 문법 구조 아래 깔린 **철학은 완전히 다릅니다.**

파이썬은 문(Statement) 중심의 언어이며, 제어 흐름(Control Flow)을 조작하고 상태를 변경하기 위해 문장을 나열합니다. 반면 Rust는 OCaml, Haskell 같은 함수형 언어의 영향을 강하게 받은 **표현식(Expression) 기반 언어**입니다. 거의 모든 코드 블록이 평가(Evaluate)되어 값을 반환합니다.

10년 차 파이썬 개발자라면 `mypy`를 도입했다가 `# type: ignore` 주석의 늪에 빠져본 경험, `int`가 어느 순간 `float`로 승격되어 미묘한 버그를 만든 경험, 혹은 수천만 건의 레코드를 처리하는 배치 작업에서 `int` 객체 하나가 28바이트씩 차지하며 메모리를 잡아먹는 모습을 `tracemalloc`으로 목격한 경험이 있을 겁니다. 이 모든 고통의 뿌리는 하나입니다. 파이썬의 타입은 **런타임에만 존재하는 객체의 꼬리표**일 뿐, 메모리 레이아웃을 결정하지 못한다는 것.

이번 챕터에서는 파이썬의 제어 구문과 Rust의 표현식이 어떻게 다르게 동작하는지, 파이썬의 동적 타입(Dynamic Typing)과 Rust의 정적 타입(Static Typing)이 메모리 레이아웃에서 얼마나 다른 물리적 결과를 만드는지, 그리고 파이썬의 임의정밀도 정수가 사라진 자리에 들어온 고정폭 정수의 오버플로 동작을 바이트 단위로 해부합니다. 이 챕터를 마치면 여러분은 Chapter 4의 소유권(Ownership)을 배울 준비, 즉 "이 데이터는 스택 몇 바이트인가, 힙으로 가는가"를 본능적으로 떠올리는 사고방식을 갖추게 됩니다.

---

## 3.1 The Python Way — 문(Statement) 위주의 제어 흐름

먼저 우리가 지금까지 해온 방식을 정직하게 들여다봅시다. 파이썬에서 "조건에 따라 값을 결정"하는 코드는 거의 항상 둘 중 하나입니다.

```python
from __future__ import annotations

def resolve_timeout(env: str) -> int:
    # 방법 1: 외부에 변수를 미리 선언하고 분기마다 덮어쓰기(Mutation)
    # 초기화되지 않은 변수 참조를 피하려고 의미 없는 기본값을 박아넣는 보일러플레이트
    timeout = 0
    if env == "prod":
        timeout = 30
    elif env == "staging":
        timeout = 15
    else:
        timeout = 5
    return timeout

def resolve_timeout_ternary(env: str) -> int:
    # 방법 2: 삼항 연산자. 조건이 셋 이상이면 가독성이 즉시 무너진다.
    return 30 if env == "prod" else (15 if env == "staging" else 5)
```

여기에는 세 가지 함정이 숨어 있습니다.

첫째, **`if`는 값을 반환하지 않습니다.** 파이썬의 `if`, `for`, `while`, `try-except`는 모두 문(Statement)입니다. 따라서 분기 결과를 변수에 담으려면 변수를 먼저 만들어두고 각 분기에서 재할당해야 합니다. `timeout = 0`이라는 가짜 초기값은 논리적으로 의미가 없지만, 이게 없으면 어떤 코드 경로에서 변수가 정의되지 않아 `UnboundLocalError`가 날 수 있습니다. 컴파일러가 없으니 이 위험을 사람이 눈으로 검사해야 합니다.

둘째, **변수가 가변(Mutable)이 됩니다.** `timeout`은 함수 안에서 최소 두 번 값이 바뀝니다. 멀티스레드 환경이나 긴 함수에서 "이 변수의 최종값이 뭐지?"를 추적하려면 모든 분기를 다 읽어야 합니다.

셋째, 그리고 가장 결정적으로, **타입 힌트는 런타임에 무시됩니다.** `resolve_timeout`의 시그니처는 `-> int`라고 약속하지만, 실수로 `timeout = "30"`이라고 쓴 누군가의 PR이 머지되어도 파이썬 인터프리터는 침묵합니다. `mypy`를 CI에 걸어두지 않았다면, 이 버그는 프로덕션에서 `TypeError: unsupported operand`로 터집니다. `mypy`를 걸었더라도, `Any`로 오염된 서드파티 라이브러리 경계, 동적 `getattr`, 메타클래스 마법 앞에서는 무력합니다.

> 💡 **핵심**: 파이썬에서 타입 힌트는 "주석이 달린 문서"에 가깝습니다. 인터프리터는 `def f(x: int) -> int`의 `int`를 보지도 않습니다. 타입은 값이 아니라 *객체*에 붙어 있고, 그 검사는 전적으로 선택적 외부 도구(`mypy`, `pyright`)의 몫입니다.

---

## 3.2 The Rust Mental Model — 모든 것은 값이다

Rust는 위 세 문제를 언어 차원에서 동시에 해결합니다. 핵심 전환은 단 한 문장입니다. **블록 `{}`은 그 자체가 표현식이며, 마지막 줄에 세미콜론이 없으면 그 값을 반환한다.**

```rust
fn resolve_timeout(env: &str) -> u32 {
    // if 전체가 표현식 → 결과를 불변 변수에 바로 바인딩
    let timeout = if env == "prod" {
        30
    } else if env == "staging" {
        15
    } else {
        5
    };
    timeout // 세미콜론 없음 → 함수의 반환값
}
```

`timeout`은 `let`으로 단 한 번 바인딩되고 그 후 절대 바뀌지 않습니다(불변, Immutable). 가짜 초기값이 필요 없습니다. 어떤 분기도 빠뜨릴 수 없습니다(빠뜨리면 타입이 안 맞아 컴파일 에러). 그리고 `env`에 정수를 넘기거나 `u32`가 아닌 값을 반환하려 하면 `cargo build` 단계에서 즉시 거부당합니다. 런타임에 도달하기 전에 말입니다.

이 "표현식 우선" 철학은 단순히 코드를 짧게 만들려는 미학이 아닙니다. **불변성을 기본값으로 만들기 위한 설계**입니다. 분기마다 변수를 덮어쓰는 대신 분기가 값을 *낳게* 하면, 변수는 한 번 정해지고 변하지 않습니다. 이것이 Chapter 16~18에서 다룰 동시성에서 데이터 경합(Data Race)을 원천 차단하는 토대가 됩니다.

### 세미콜론의 진짜 의미: 표현식을 문으로 강등하기

파이썬에서 세미콜론은 거의 쓸 일이 없는 "한 줄에 여러 문장" 구분자였습니다. Rust에서 세미콜론은 훨씬 무거운 의미를 가집니다. **세미콜론은 표현식을 문(Statement)으로 강등시키고, 그 값을 버립니다.**

```rust
fn main() {
    let a = {
        let y = 3;
        let z = 4;
        y + z // 세미콜론 없음 → 블록이 7을 반환 → a = 7
    };

    let b = {
        let y = 3;
        let z = 4;
        y + z; // 🚨 세미콜론 있음 → 값 7을 버림 → 블록은 () 반환 → b = ()
    };

    println!("a = {a}");   // a = 7
    // println!("b = {}", b); // b는 () 타입이라 {} 포맷 불가
}
```

`a`와 `b`는 단 한 글자, 세미콜론 하나로 운명이 갈립니다. `a`는 `i32` 값 `7`을 갖고, `b`는 곧 설명할 **유닛 타입 `()`** 을 갖습니다. 이 규칙은 함수 끝에서도 똑같이 적용됩니다.

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b   // 세미콜론 없음 → i32 반환. Idiomatic Rust.
}

fn broken(a: i32, b: i32) -> i32 {
    a + b;  // 🚨 세미콜론 있음 → 값을 버리고 () 반환 → i32와 불일치, 컴파일 에러
}
```

`broken`을 컴파일하면 rustc는 다음과 같이 정확히 어디가 문제인지 짚어줍니다.

```text
error[E0308]: mismatched types
 --> src/main.rs:6:30
  |
6 | fn broken(a: i32, b: i32) -> i32 {
  |    ------                    ^^^ expected `i32`, found `()`
  |    |
  |    implicitly returns `()` as its body has no tail expression
7 |     a + b;
  |          - help: remove this semicolon to return this value
```

`help: remove this semicolon to return this value` — 컴파일러가 직접 고치는 법까지 알려줍니다. 파이썬이라면 이 함수는 조용히 `None`을 반환하고, 호출부에서 `None + 5`를 시도하다 한참 뒤에 `TypeError`로 폭발했을 겁니다.

### 유닛 타입 `()`: 파이썬 `None`의 0바이트 사촌

파이썬에서 값을 반환하지 않는 함수는 암묵적으로 `None`을 반환합니다. 그런데 `None`은 사실 `NoneType`의 싱글턴 *객체*이며, 다른 모든 파이썬 객체처럼 힙에 존재하고 참조 카운트를 가집니다(다만 싱글턴이라 인스턴스는 하나뿐).

Rust에서 "아무것도 반환하지 않음"은 유닛 타입 `()`(빈 튜플)으로 표현됩니다. 그리고 이것의 크기는 정확히 **0바이트**입니다.

```rust
use std::mem::size_of;

fn main() {
    println!("size of () = {}", size_of::<()>()); // size of () = 0
}
```

> **Under the Hood — 0바이트 타입의 의미**
>
> `size_of::<()>() == 0`이라는 사실은 단순한 트리비아가 아닙니다. `()`는 *타입 시스템에는 존재하지만 메모리에는 존재하지 않는* 타입입니다. 예를 들어 `HashMap<K, ()>`은 값 자리에 아무 데이터도 저장하지 않으므로 사실상 `HashSet<K>`이 됩니다(실제로 표준 라이브러리의 `HashSet`이 이렇게 구현되어 있습니다). `Result<(), Error>`는 "성공 시 의미 있는 값은 없지만 실패할 수는 있는" 연산을 0바이트 오버헤드로 표현합니다. 파이썬의 `None`은 항상 객체 포인터(8바이트) + 그 객체의 존재를 끌고 다니지만, `()`는 컴파일 후 완전히 증발합니다.

---

## 3.3 `let`, `mut`, `const`, `static`, 그리고 섀도잉

파이썬에서 `x = 5`는 단순합니다. 이름 `x`를 객체 `5`에 묶는 것뿐입니다. Rust에는 비슷해 보이지만 메모리 의미가 전혀 다른 다섯 가지 바인딩 방식이 있습니다. 이것을 구분하지 못하면 Rust의 절반을 잃습니다.

### `let` vs `let mut` — 불변이 기본값

```rust
fn main() {
    let x = 5;
    // x = 6; // 🚨 error[E0384]: cannot assign twice to immutable variable `x`

    let mut y = 5;
    y = 6; // OK. mut을 명시했으므로 재할당 허용
    println!("{x} {y}"); // 5 6
}
```

파이썬에서 모든 변수는 사실상 가변이고, "불변"을 강제하려면 `@property`로 setter를 막거나 `Final` 타입 힌트(역시 런타임 무시)에 기대야 합니다. Rust는 정반대입니다. **기본이 불변이고, 변경이 필요하면 `mut`으로 명시적 허가를 받아야 합니다.** 이 한 글자의 차이가 코드 리뷰어에게 "이 변수는 바뀐다"는 신호를 보냅니다.

### 섀도잉(Shadowing) — 같은 이름, 새로운 변수

파이썬에서 `x = "5"` 다음에 `x = int(x)`를 쓰면 이름 `x`가 가리키는 객체만 바뀝니다. Rust의 섀도잉은 표면적으로 비슷해 보이지만 **완전히 새로운 변수를 만듭니다.**

```rust
fn main() {
    let spaces = "   ";        // &str 타입 (문자열 슬라이스)
    let spaces = spaces.len(); // usize 타입. 같은 이름이지만 새 변수, 타입까지 바뀜
    println!("spaces len = {spaces}"); // spaces len = 3
}
```

`let mut spaces = "   "; spaces = spaces.len();`은 **컴파일되지 않습니다.** `mut`은 같은 타입 안에서의 재할당만 허용하기 때문입니다. 섀도잉은 새 `let`이므로 타입을 바꿔도 됩니다. 이 덕분에 파이썬에서 `raw_input`, `parsed_input`, `validated_input`처럼 단계마다 변수 이름을 새로 짓던 습관을 버리고, 입력을 정제하는 파이프라인 전체에서 의미 있는 이름 하나(`input`)를 재사용할 수 있습니다.

### `const` vs `static` — 컴파일 타임 상수와 고정 메모리

```rust
const MAX_RETRIES: u32 = 3;       // 컴파일 타임 상수. 사용처마다 인라인됨
static GREETING: &str = "hello";  // 프로그램 수명 내내 고정된 단 하나의 메모리 주소

fn main() {
    println!("retries={MAX_RETRIES}, greeting={GREETING}");
}
```

둘 다 파이썬의 "대문자 상수 관례"(`MAX_RETRIES = 3` — 사실은 그냥 변경 가능한 전역 변수)와는 차원이 다릅니다.

| 항목 | `const` | `static` |
|------|---------|----------|
| 타입 명시 | 필수 | 필수 |
| 메모리 주소 | 없음(사용처에 값 복사·인라인) | 단 하나의 고정 주소(`'static`) |
| 가변성 | 항상 불변 | 기본 불변, `static mut`은 `unsafe` |
| 평가 시점 | 컴파일 타임 | 컴파일 타임 초기화 |
| 비유 | C의 `#define` / `constexpr` | C의 전역 `static` 변수 |

`const`는 컴파일러가 사용되는 모든 자리에 값을 그대로 박아 넣습니다(인라인). 그래서 별도의 메모리 주소를 가지지 않습니다. 반면 `static`은 바이너리의 데이터 영역에 단 하나의 실체로 존재하며 고정된 주소를 가집니다. 대부분의 경우 `const`를 쓰고, 진짜로 "프로그램 전체에서 공유되는 단일 메모리 위치"가 필요할 때만 `static`을 씁니다.

---

## 3.4 타입 추론: Hindley-Milner vs 런타임에 무시되는 타입 힌트

파이썬 시니어들이 Rust를 처음 보고 흔히 오해하는 것이 있습니다. "타입을 다 적어야 한다니, Java처럼 장황하겠군." 정반대입니다. Rust의 타입 추론은 파이썬의 타입 힌트보다 *훨씬 강력하고*, 실제로 적어야 하는 타입은 함수 시그니처 정도뿐입니다.

```rust
fn main() {
    // 어디에도 타입을 안 적었지만, 컴파일러가 전부 추론한다.
    let nums = vec![1, 2, 3, 4, 5];          // Vec<i32> (정수 리터럴 기본값)
    let doubled: Vec<_> = nums.iter()
        .map(|x| x * 2)
        .collect();                           // Vec<i32>
    let total: i64 = nums.iter().map(|&x| x as i64).sum();

    println!("{doubled:?}, total={total}");   // [2, 4, 6, 8, 10], total=15
}
```

여기서 주목할 점은 `let nums: Vec<i32> = (1..=5).collect();`처럼 쓸 때, 컴파일러가 `collect()`의 **반환 타입을 보고 거꾸로** 이터레이터의 타입을 결정한다는 것입니다. 파이썬의 타입 힌트는 절대 이렇게 추론을 *양방향*으로 전파하지 못합니다.

Rust의 추론 엔진은 **Hindley-Milner(힌들리-밀너) 타입 시스템** 계열에 뿌리를 둡니다. ML, OCaml, Haskell이 쓰는 그 알고리즘입니다. 핵심 아이디어는 이렇습니다. 컴파일러는 프로그램 전체를 "타입 변수"가 가득한 방정식 집합으로 보고, 사용된 연산·함수·리터럴이 부과하는 제약(Constraint)을 모아 단일화(Unification)로 풉니다. 그 결과 *지역적으로는 타입을 한 글자도 안 적어도* 전역적으로 유일한 타입이 결정됩니다.

> 💡 **핵심**: 결정적 차이는 *검사 시점*입니다.
> - 파이썬 타입 힌트: 런타임에 **무시됨**. `mypy`/`pyright`라는 *별도 도구*가 *선택적으로* 정적 검사. 서드파티 `Any`, 동적 속성, 메타클래스 앞에서 한계.
> - Rust 타입 추론: 컴파일의 **필수 단계**. 타입이 맞지 않으면 바이너리 자체가 만들어지지 않음. 우회로 없음.

그래서 Rust에서는 "타입을 안 적어서 편하다"와 "타입이 100% 보장된다"가 동시에 성립합니다. 파이썬에서는 둘 중 하나를 포기해야 했습니다. 타입 힌트 없이 편하게 쓰거나, `mypy --strict`로 무장하고 끝없는 `# type: ignore`와 싸우거나.

### 추론이 멈추는 곳: 명시가 필요한 순간

추론도 만능은 아닙니다. 빈 컬렉션을 `collect`하거나 `parse`처럼 같은 입력에서 여러 타입이 나올 수 있을 때는 사람이 의도를 알려줘야 합니다.

```rust
fn main() {
    // 🚨 error[E0282]: type annotations needed
    // let parsed = "42".parse().unwrap();

    let parsed: i32 = "42".parse().unwrap(); // 타입 명시로 해결
    // 또는 터보피쉬(turbofish) 문법으로:
    let parsed2 = "42".parse::<i32>().unwrap();
    println!("{parsed} {parsed2}");
}
```

`parse()`는 `FromStr`를 구현한 *어떤* 타입으로든 변환할 수 있어서, `i32`인지 `f64`인지 `u8`인지 컴파일러가 알 수 없습니다. 이때는 `let parsed: i32`로 좌변에 타입을 주거나, `parse::<i32>()`라는 **터보피쉬(`::<>`)** 로 우변에 직접 못 박습니다.

---

## 3.5 메모리 레이아웃: PyObject 헤더 vs Rust 스칼라

이제 이 챕터의 심장부입니다. 파이썬 개발자가 가장 충격받는 지점, 타입 시스템이 메모리에 미치는 *물리적* 영향을 바이트 단위로 봅시다.

### 파이썬: 정수 하나에 28바이트

파이썬의 모든 값은 객체이고, 모든 객체는 `PyObject` 헤더를 답니다. CPython에서 정수 `1` 하나의 실제 크기를 측정해 봅시다.

```python
import sys
print(sys.getsizeof(1))     # 28  (작은 정수)
print(sys.getsizeof(10**30)) # 44  (임의정밀도: 자릿수가 늘면 객체도 커짐)
print(sys.getsizeof(True))   # 28
```

`int` 하나가 **28바이트**입니다. 이 28바이트는 다음으로 구성됩니다(64비트 CPython 기준).

```text
PyObject 헤더:
  ob_refcnt   : 8 bytes   (참조 카운트)
  ob_type     : 8 bytes   (타입 객체 포인터 → "이건 int야"를 가리킴)
PyLongObject 추가:
  ob_size     : 8 bytes   (자릿수/부호 정보)
  ob_digit[0] : 4 bytes   (실제 정수 데이터, 30비트 단위 "자릿수")
  ----------------------
  합계         : 28 bytes
```

그리고 이 28바이트짜리 객체는 **힙(Heap)** 에 있습니다. `a = 1; b = 1`이라고 쓰면 두 이름은 같은 캐시된 객체를 가리키는 두 개의 포인터(각 8바이트)일 뿐입니다. 즉, "정수 변수 하나"의 실질 비용은 스택의 포인터 8바이트 + 힙의 객체 28바이트입니다.

여기에 더해, 사용자 정의 클래스의 인스턴스는 보통 **`__dict__`** 라는 해시맵을 하나씩 더 들고 다닙니다. `p.x = 10`은 내부적으로 `p.__dict__["x"] = <int 객체 10>`이며, 이는 문자열 `"x"`의 해시 계산 → 버킷 탐색 → 포인터 연결이라는 일련의 작업을 매번 수행합니다. `__slots__`로 이 `__dict__`를 제거할 수 있지만, 대부분의 코드베이스는 그렇게 하지 않습니다.

### Rust: 정수는 정확히 그 바이트만큼

Rust의 원시 타입(Primitive Type)에는 헤더가 없습니다. `i32`는 정확히 4바이트, 그 이상도 이하도 아닙니다. `std::mem::size_of`로 직접 확인해 봅시다. 아래는 실제로 컴파일·실행한 결과입니다.

```rust
use std::mem::size_of;

fn main() {
    println!("i8={}, i32={}, i64={}, i128={}",
        size_of::<i8>(), size_of::<i32>(), size_of::<i64>(), size_of::<i128>());
    println!("bool={}, char={}, f32={}, f64={}",
        size_of::<bool>(), size_of::<char>(), size_of::<f32>(), size_of::<f64>());
    println!("usize={}, isize={}", size_of::<usize>(), size_of::<isize>());
}
```

```text
i8=1, i32=4, i64=8, i128=16
bool=1, char=4, f32=4, f64=8
usize=8, isize=8
```

몇 가지 짚어둘 점이 있습니다.

- **`bool`은 1바이트**입니다. 비트 하나면 충분할 것 같지만, CPU는 바이트 단위로 주소를 매기므로 1바이트가 최소 단위입니다.
- **`char`는 4바이트**입니다. C의 `char`(1바이트)와 헷갈리면 안 됩니다. Rust의 `char`는 **유니코드 스칼라 값(Unicode Scalar Value)**, 즉 `'a'`부터 `'😀'`, `'한'`까지 어떤 코드포인트든 담을 수 있는 4바이트(U+0000 ~ U+10FFFF)입니다. 파이썬 3의 문자열도 유니코드지만, Rust는 *한 글자*의 크기를 타입 수준에서 4바이트로 못 박았습니다.
- **`usize`/`isize`는 플랫폼의 포인터 크기**입니다. 64비트 머신에서 8바이트. 배열 인덱싱과 컬렉션 길이는 항상 `usize`를 씁니다. "이 정수가 메모리 주소를 표현할 수 있는가"를 타입으로 구분하는 셈입니다.

> **Under the Hood — 28바이트 vs 4바이트, 그리고 캐시**
>
> 정수 100만 개를 담는다고 합시다. 파이썬 `list`는 100만 개의 *포인터*(각 8바이트 = 8MB) + 각 포인터가 가리키는 힙의 `int` 객체(각 28바이트 = 28MB), 도합 약 36MB를 힘 곳곳에 파편화해서 씁니다. Rust의 `Vec<i32>`는 100만 × 4바이트 = **정확히 4MB**를 *연속된* 메모리 블록 하나에 담습니다. 9배 차이는 메모리만의 문제가 아닙니다. Rust의 연속 배열은 CPU가 한 번에 캐시 라인(보통 64바이트 = `i32` 16개)으로 끌어오므로, 순회 시 캐시 미스(Cache Miss)가 거의 없습니다. 파이썬의 파편화된 포인터 추격(Pointer Chasing)은 매 원소마다 캐시 미스를 유발할 수 있습니다. `numpy`가 빠른 이유가 바로 이것입니다. `numpy`는 내부적으로 Rust의 `Vec<i32>`와 똑같은 "헤더 없는 연속 배열"을 C로 구현한 것이니까요.

### 정렬(Alignment)과 패딩(Padding): 구조체는 단순한 합이 아니다

이제 가장 미묘한 부분입니다. 튜플과 구조체의 크기는 *필드 크기의 단순 합이 아닙니다.* CPU는 데이터를 특정 경계(보통 자기 크기의 배수 주소)에 정렬해야 효율적으로 읽기 때문에, 컴파일러가 빈틈(패딩)을 끼워 넣습니다.

기존 챕터에서 본 `(i32, f64)` 튜플을 실제로 측정하면 이렇습니다.

```rust
use std::mem::{size_of, align_of};

fn main() {
    println!("(i32, f64) size={} align={}",
        size_of::<(i32, f64)>(), align_of::<(i32, f64)>());
    // (i32, f64) size=16 align=8
}
```

`i32`(4) + `f64`(8) = 12바이트일 것 같지만 결과는 **16바이트**입니다. `f64`는 8바이트 정렬을 요구하므로, `i32` 뒤에 4바이트 패딩이 들어가 `f64`가 8바이트 경계에서 시작하게 만듭니다.

```text
(i32, f64)의 실제 스택 배치:
[ i32: 4B ][ 패딩: 4B ][ f64: 8B ]
0          4          8          16
↑ 전체 16바이트, 8바이트 정렬
```

더 흥미로운 건 **필드 순서가 크기를 바꾼다**는 점입니다. 다음 두 구조체를 실제로 측정해 봅시다.

```rust
use std::mem::size_of;

struct BadLayout {  // 순서대로라면 1+4+1 = 6, 패딩 끼면 12가 될 법한데...
    a: u8,
    b: u32,
    c: u8,
}

struct GoodLayout {
    b: u32,
    a: u8,
    c: u8,
}

fn main() {
    println!("BadLayout  = {}", size_of::<BadLayout>());  // 8
    println!("GoodLayout = {}", size_of::<GoodLayout>()); // 8
}
```

> **Under the Hood — Rust는 필드를 자동으로 재배치한다**
>
> C에서 `BadLayout`(u8, u32, u8)을 그대로 두면 `[u8][pad 3][u32][u8][pad 3]` = **12바이트**가 됩니다. 그런데 Rust의 기본 표현(`repr(Rust)`)은 **필드 순서를 보장하지 않으며**, 컴파일러가 패딩을 최소화하도록 자유롭게 재배치합니다. 그래서 `BadLayout`조차 내부적으로 큰 필드부터 정렬해 `[u32][u8][u8][pad 2]` = **8바이트**로 최적화합니다. 위 측정에서 두 구조체가 똑같이 8바이트인 이유입니다. 만약 C ABI 호환(FFI, Chapter 21)이 필요해 `#[repr(C)]`를 붙이면 그때는 선언 순서가 강제되어 `BadLayout`이 다시 12바이트가 됩니다. 즉, Rust는 *기본적으로는* 메모리를 알아서 빽빽하게 채우고, 여러분이 명시적으로 ABI를 요구할 때만 손을 뗍니다. 파이썬에서는 이런 레이아웃을 *생각할 필요조차 없었지만*, 그 대가로 모든 필드가 힙 위 객체 포인터였습니다.

이 차이를 표로 정리하면 다음과 같습니다.

| 개념 | 파이썬(CPython) | Rust |
|------|----------------|------|
| 정수 `int` | 힙 객체, 최소 28바이트 + 8바이트 포인터 | 스택, `i32`=4바이트(헤더 없음) |
| 불리언 | `True`/`False` 싱글턴 객체(28바이트) | `bool` = 1바이트 |
| 한 글자 | `str` 객체(49바이트~) | `char` = 4바이트(유니코드 스칼라) |
| "값 없음" | `None` 싱글턴 객체(포인터 8바이트) | `()` = 0바이트 |
| 객체 속성 | `__dict__` 해시맵 조회 | 구조체 필드, 컴파일 타임 오프셋 |
| 컬렉션 | 포인터 배열 + 파편화된 힙 객체 | 연속 메모리 블록(`Vec<T>`) |
| 타입 검사 | 런타임 무시(`mypy`는 선택) | 컴파일 필수, 우회 불가 |

---

## 3.6 정수 타입 체계와 오버플로: 임의정밀도의 종말

파이썬의 `int`는 **임의정밀도(Arbitrary Precision)** 입니다. `2 ** 1000`을 계산해도 자동으로 메모리를 늘려 정확한 값을 줍니다. 우아하지만, 모든 정수 연산이 "오버플로 검사 + 필요 시 재할당" 분기를 거친다는 숨은 비용이 있습니다. 그리고 임베디드, 네트워크 프로토콜, 바이너리 포맷처럼 "이 값은 정확히 8비트"여야 하는 영역에서는 오히려 거추장스럽습니다.

Rust의 정수는 모두 **고정폭(Fixed-width)** 입니다. `u8`은 0~255, `i32`는 약 ±21억. 폭을 넘으면 어떻게 될까요? 여기서 Rust의 가장 실용적인 설계 결정 중 하나가 등장합니다.

### Debug에서 패닉, Release에서 래핑

```rust
use std::hint::black_box;

fn main() {
    let x: u8 = black_box(255); // black_box로 컴파일러의 상수 폴딩을 막음
    let y: u8 = black_box(1);
    let z = x + y;              // 255 + 1 = ?
    println!("{z}");
}
```

이 코드를 두 가지 빌드 모드로 돌리면 **동작이 다릅니다.** 실제 실행 결과입니다.

```text
# 디버그 빌드 (cargo run)
thread 'main' panicked at src/main.rs:5:13:
attempt to add with overflow

# 릴리스 빌드 (cargo run --release)
0
```

> **Under the Hood — 왜 모드마다 다른가**
>
> 디버그 빌드는 모든 산술 연산에 오버플로 검사 코드를 삽입합니다. 넘치면 즉시 패닉(`attempt to add with overflow`)을 내서 *개발 중에* 버그를 잡습니다. 릴리스 빌드(`--release`)는 이 검사를 제거하고, 하드웨어의 기본 동작인 **2의 보수 래핑(wrapping)** 을 그대로 씁니다. `255 + 1`은 8비트 안에서 한 바퀴 돌아 `0`이 됩니다. 이건 "릴리스에서는 오버플로를 무시한다"는 위험한 타협처럼 보이지만, 실은 명확한 계약입니다. *오버플로가 일어날 수 있는 곳에서는 추측하지 말고 아래의 명시적 메서드를 써라.* C/C++에서 부호 있는 정수 오버플로는 **정의되지 않은 동작(Undefined Behavior)** 이라 컴파일러가 코드를 멋대로 삭제할 수도 있는 반면, Rust의 릴리스 래핑은 *항상 2의 보수로 정의된* 동작입니다. 이미 이것만으로도 C보다 안전합니다.

### 의도를 표현하는 네 가지 메서드

"넘칠 수도 있다"를 안다면, 추측 대신 의도를 코드로 못 박으세요. Rust의 모든 정수 타입에는 네 가지 산술 메서드 군이 있습니다. 아래는 `u8`(0~255)에서 `250`에 더하는 실제 결과입니다.

```rust
fn main() {
    let x: u8 = 250;

    // 1) checked_*: 넘치면 None, 아니면 Some(값) → Option으로 강제 처리
    println!("{:?}", x.checked_add(10)); // None
    println!("{:?}", x.checked_add(5));  // Some(255)

    // 2) wrapping_*: 모듈러 연산(한 바퀴 돌기). 해시·체크섬·링버퍼에 적합
    println!("{}", x.wrapping_add(10));  // 4   (260 mod 256)

    // 3) saturating_*: 한계값에 달라붙음. 게이지·카운터·클램핑에 적합
    println!("{}", x.saturating_add(10)); // 255 (상한에 고정)

    // 4) overflowing_*: (결과값, 넘쳤는지 bool) 튜플 반환
    println!("{:?}", x.overflowing_add(10)); // (4, true)
}
```

| 메서드 | 반환 | 250 + 10 (u8) | 언제 쓰나 |
|--------|------|---------------|-----------|
| `checked_add` | `Option<u8>` | `None` | 오버플로를 에러로 다뤄야 할 때(금액 합산 등) |
| `wrapping_add` | `u8` | `4` | 모듈러가 *의도*일 때(해시, CRC, 링버퍼) |
| `saturating_add` | `u8` | `255` | 상/하한에 고정하고 싶을 때(진행률, HP) |
| `overflowing_add` | `(u8, bool)` | `(4, true)` | 값과 넘침 여부를 동시에 알아야 할 때 |

파이썬에는 이런 구분 자체가 없습니다. 임의정밀도라 넘칠 일이 없으니까요. 하지만 그 편리함은 "이 카운터는 절대 `u32`를 넘으면 안 된다"는 도메인 제약을 *코드로 표현할 수단이 없다*는 뜻이기도 합니다. Rust에서는 `checked_add`가 반환한 `Option`을 처리하지 않으면 컴파일러가 경고하고, `?` 연산자(Chapter 12)로 에러를 전파하게 만들 수 있습니다.

### 형변환 `as`의 위험성과 명시적 변환 철학

파이썬에서 `True + 1`은 `2`입니다. `bool`이 `int`의 서브클래스라 암묵적으로 승격되기 때문입니다. `3 + 2.0`은 `5.0`이고요. Rust는 이런 **암묵적 형변환을 일절 허용하지 않습니다.** 서로 다른 정수 타입을 더하는 것조차 컴파일 에러입니다(3.7절에서 시연).

타입을 바꾸려면 `as` 키워드로 명시해야 합니다. 그런데 `as`는 *조용히 데이터를 손상시킬 수 있어* 신중히 써야 합니다. 실제 결과를 봅시다.

```rust
fn main() {
    let big: i32 = 300;
    println!("300 as u8 = {}", big as u8);   // 44   ← 300 - 256, 상위 비트 잘림!

    let neg: i32 = -1;
    println!("-1 as u8 = {}", neg as u8);     // 255  ← 2의 보수 비트 그대로 해석

    let f: f64 = 3.99;
    println!("3.99 as i32 = {}", f as i32);   // 3    ← 반올림 아님, 소수부 절단

    let huge: f64 = 1e20;
    println!("1e20 as i32 = {}", huge as i32); // 2147483647 ← i32::MAX로 포화(saturating)
}
```

`300 as u8`이 `44`가 되는 건 상위 비트를 그냥 버리기 때문이고, `3.99 as i32`가 `4`가 아니라 `3`인 건 `as`가 반올림이 아니라 소수부 절단(truncation)이기 때문입니다. 이런 잠재적 데이터 손실 때문에, 실무에서는 손실 없는 변환을 보장하는 `From`/`Into` 트레이트(Chapter 9)나, 손실 가능성을 `Result`로 돌려주는 `TryFrom`/`TryInto`를 선호합니다.

```rust
fn main() {
    let small: i32 = 100;
    let widened: i64 = small.into();          // 손실 없음 → 항상 안전

    let big: i32 = 300;
    let narrowed: Result<u8, _> = u8::try_from(big); // 300은 u8 범위 밖
    println!("{widened}, {narrowed:?}");      // 100, Err(TryFromIntError(()))
}
```

> 💡 **핵심**: `as`는 "내가 데이터 손실 가능성을 알고도 강제 변환한다"는 선언입니다. 넓히는 변환(`i32→i64`)은 `.into()`로, 좁히는 변환(`i32→u8`)은 `try_from`으로 실패를 명시적으로 다루는 것이 Idiomatic Rust입니다. `as`는 비트 조작이 *의도*인 저수준 코드에 아껴 쓰세요.

부동소수점도 잠깐 짚으면, Rust의 `f32`/`f64`는 IEEE 754 표준을 그대로 따릅니다. `0.1 + 0.2 != 0.3`인 것은 파이썬과 똑같습니다(같은 IEEE 754니까요). 다만 Rust는 `f64`에 `==`를 직접 쓰면 clippy가 `clippy::float_cmp` 경고를 띄워, 부동소수점 비교는 허용 오차(epsilon)를 두라고 권합니다.

---

## 3.7 제어 흐름: match 망라성, if let, while let, loop, 라벨

표현식 기반 사고를 제어 흐름 전체로 확장해 봅시다. Rust의 제어 구조는 거의 모두 값을 반환하며, 컴파일러가 빠진 경우를 잡아냅니다.

### `match`의 망라성(Exhaustiveness) — 컴파일러가 케이스를 센다

파이썬 3.10의 `match-case`는 강력하지만 여전히 문(Statement)이고, *모든 경우를 다뤘는지 검사하지 않습니다.* `case`를 빠뜨려도 조용히 아무 일도 안 일어나고 통과합니다. Rust의 `match`는 표현식이며, **모든 가능한 값을 다루지 않으면 컴파일을 거부합니다.**

```rust
enum Direction { North, South, East, West }

fn main() {
    let d = Direction::North;
    let s = match d {
        Direction::North => "N",
        Direction::South => "S",
        // East, West를 빠뜨림
    };
    println!("{s}");
}
```

```text
error[E0004]: non-exhaustive patterns: `Direction::East` and `Direction::West` not covered
 --> src/main.rs:6:19
  |
6 |     let s = match d {
  |                   ^ patterns `Direction::East` and `Direction::West` not covered
  |
note: `Direction` defined here
  = note: the matched value is of type `Direction`
```

이게 왜 혁명적일까요? 새로운 상태를 추가하는 상황을 떠올려 보세요. 결제 시스템에 `Refunded` 상태를 새로 추가했다고 합시다. 파이썬이라면 이 상태를 처리하지 않은 수십 개의 `if/elif` 사슬과 `match-case`가 *조용히 잘못된 기본 분기로 빠집니다.* Rust라면 컴파일러가 `Refunded`를 다루지 않은 *모든* `match`를 정확히 짚어 컴파일을 거부합니다. **타입이 곧 체크리스트**가 됩니다. 이 위력은 Chapter 10의 열거형 상태 기계에서 본격적으로 펼쳐집니다. 와일드카드 `_`를 남발하면 이 안전망을 스스로 무력화하는 셈이니, "정말 나머지 전부를 같게 처리해도 되는가"를 자문하세요.

### `if let`과 `while let` — 한 패턴만 신경 쓸 때

전체 `match`가 과한 상황, 즉 "딱 한 가지 경우에만 관심 있을 때"를 위한 축약형입니다.

```rust
fn main() {
    let config: Option<u32> = Some(8080);

    // match 대신 if let: Some일 때만 처리
    if let Some(port) = config {
        println!("포트 {port} 사용");
    } else {
        println!("기본 포트 사용");
    }

    // while let: 패턴이 매칭되는 동안 반복. 스택을 비울 때 전형적
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() { // pop()은 Option 반환, None이면 종료
        print!("{top} ");               // 3 2 1
    }
    println!();
}
```

`while let Some(top) = stack.pop()`은 파이썬의 `while stack: top = stack.pop()` 패턴을 한 줄로, 그것도 빈 스택 처리까지 안전하게 표현합니다. `pop()`이 `None`을 반환하는 순간 루프가 깔끔하게 끝납니다.

### `loop` + `break value` — 값을 반환하는 무한 루프

파이썬에서 "결과를 얻을 때까지 반복하고 그 결과를 변수에 담기"는 항상 외부 변수 + `while True` + `break` 조합이었습니다. Rust의 `loop`는 `break`에 값을 실어 보낼 수 있어, 루프 자체가 표현식이 됩니다.

```rust
fn main() {
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2; // 이 값이 loop 표현식의 결과가 됨
        }
    };
    println!("result = {result}"); // result = 20
}
```

`result`는 가짜 초기값 없이, 루프가 *낳은* 값 `20`으로 한 번에 바인딩됩니다. 재시도 로직, 폴링 루프처럼 "성공할 때까지 돌리고 결과를 받는" 패턴에 완벽히 들어맞습니다.

### 라벨 루프(Labeled Loop) — 중첩 루프 정밀 제어

파이썬에서 중첩 루프를 한 번에 탈출하려면 플래그 변수를 두거나 함수로 빼서 `return`하는 우회가 필요했습니다. Rust는 `'name:` 라벨로 *어느 루프를* break/continue할지 직접 지정합니다.

```rust
fn main() {
    let mut found = None;
    'outer: for i in 0..5 {
        for j in 0..5 {
            if i * j == 6 {
                found = Some((i, j));
                break 'outer; // 안쪽이 아니라 바깥 루프까지 한 번에 탈출
            }
        }
    }
    println!("found = {found:?}"); // found = Some((2, 3))
}
```

### 그리고 — 암묵적 형변환 금지의 컴파일러 에러

3.6절에서 예고한, 서로 다른 정수 타입을 섞을 때의 에러를 직접 봅시다. 파이썬 습관으로 `i32`와 `i64`를 함수에 그냥 넘기면 이렇게 됩니다.

```rust
fn needs_i64(n: i64) -> i64 { n * 2 }

fn main() {
    let count: i32 = 10;
    let result = needs_i64(count); // 🚨 i32를 i64 자리에 넣음
    println!("{result}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:5:28
  |
5 |     let result = needs_i64(count);
  |                  --------- ^^^^^ expected `i64`, found `i32`
  |                  |
  |                  arguments to this function are incorrect
  |
note: function defined here
 --> src/main.rs:1:4
  |
1 | fn needs_i64(n: i64) -> i64 { n * 2 }
  |    ^^^^^^^^^ ------
help: you can convert an `i32` to an `i64`
  |
5 |     let result = needs_i64(count.into());
  |                                 +++++++
```

파이썬이었다면 `10`이 자연스럽게 흘러 들어가 아무 문제 없었을 코드입니다. Rust는 거부합니다. 그리고 단순히 "안 된다"가 아니라, `help: you can convert an i32 to an i64`와 함께 `count.into()`라는 정확한 수정안까지 제시합니다. 이 엄격함이 처음엔 답답해도, "두 정수가 사실은 다른 단위(바이트 수 vs 밀리초)였다"는 부류의 버그를 컴파일 타임에 박멸합니다.

---

## 3.8 실전 예제 (Hands-on): HTTP 상태 코드 분류기

지금까지의 모든 조각 — 표현식 `match`, 망라성, 고정폭 정수, 안전한 캐스팅, `loop + break`, `if let` — 을 하나의 컴파일 가능한 프로그램으로 엮어 봅시다. 흔한 실무 시나리오인 "HTTP 응답을 분류하고, 지연 시간을 집계하고, 실패 시 재시도하는" 로직입니다.

```rust
/// HTTP 상태 코드 분류기: 표현식, match 망라성, 정수 타입, 캐스팅을 한데 묶은 예제.
#[derive(Debug, PartialEq)]
enum StatusClass {
    Informational, // 1xx
    Success,       // 2xx
    Redirect,      // 3xx
    ClientError,   // 4xx
    ServerError,   // 5xx
    Invalid,       // 그 외
}

/// u16 상태 코드를 분류한다. match가 표현식이므로 결과가 그대로 반환된다.
fn classify(code: u16) -> StatusClass {
    match code {
        100..=199 => StatusClass::Informational,
        200..=299 => StatusClass::Success,
        300..=399 => StatusClass::Redirect,
        400..=499 => StatusClass::ClientError,
        500..=599 => StatusClass::ServerError,
        _ => StatusClass::Invalid, // 망라성을 만족시키는 와일드카드
    }
}

/// 평균 지연 시간(ms)을 계산한다. u32 합산 오버플로를 u64 승격으로 예방한다.
fn average_latency_ms(samples: &[u32]) -> f64 {
    if samples.is_empty() {
        return 0.0; // 조기 반환에는 명시적 return을 사용
    }
    // 각 u32를 u64로 넓혀(into-안전한 방향) 합산해야 큰 표본에서 오버플로가 없다.
    let total: u64 = samples.iter().map(|&x| x as u64).sum();
    total as f64 / samples.len() as f64 // 마지막 표현식이 반환값
}

fn main() {
    let codes = [101u16, 200, 301, 404, 503, 700];
    for code in codes {
        let class = classify(code);
        // if let으로 ServerError만 골라 경고 처리
        if let StatusClass::ServerError = class {
            eprintln!("[ALERT] {code} -> {class:?}");
        } else {
            println!("{code} -> {class:?}");
        }
    }

    let latencies = [120u32, 95, 210, 4_000_000, 88];
    println!("avg latency = {:.2} ms", average_latency_ms(&latencies));

    // loop + break value로 재시도 로직 구현
    let mut attempt = 0u32;
    let outcome: Result<u32, &str> = loop {
        attempt += 1;
        // 처음 두 번은 503, 세 번째부터 200을 받았다고 시뮬레이션
        let simulated = classify(if attempt < 3 { 503 } else { 200 });
        if simulated == StatusClass::Success {
            break Ok(attempt); // 성공: 몇 번째 시도였는지 반환
        }
        if attempt >= 5 {
            break Err("max retries exceeded");
        }
    };
    println!("outcome after retries = {outcome:?}");
}
```

`cargo run`으로 실행한 실제 출력입니다(`[ALERT]`는 stderr로 나갑니다).

```text
101 -> Informational
200 -> Success
301 -> Redirect
404 -> ClientError
[ALERT] 503 -> ServerError
700 -> Invalid
avg latency = 800102.60 ms
outcome after retries = Ok(3)
```

이 50줄짜리 프로그램이 보여주는 것:

- `classify`는 `match` 표현식의 결과를 `return` 없이 그대로 반환합니다. `100..=199` 같은 **범위 패턴(Range Pattern)** 으로 깔끔하게 구간을 나눕니다.
- 만약 `_ => StatusClass::Invalid`를 빠뜨리면 `u16`의 나머지 값들(600~65535 등)이 안 다뤄져 E0004로 컴파일이 막힙니다.
- `average_latency_ms`는 `samples.iter().map(|&x| x as u64)`로 *넓히는* 캐스팅을 먼저 해서, 표본 합이 `u32` 범위(약 42억)를 넘어도 안전합니다. 표본에 `4_000_000`이 있는데 이런 게 1000개만 있어도 `u32`라면 넘칩니다.
- 재시도 로직은 외부 가변 플래그 없이 `loop`가 `Ok`/`Err`를 *낳아서* `outcome`에 직접 바인딩됩니다.

같은 로직을 파이썬으로 짠다면 `classify`는 `if code < 200: ... elif ...` 사슬이 되고, 새 구간을 추가할 때 어디를 빠뜨렸는지 컴파일러가 알려주지 않으며, `int` 합산은 오버플로 걱정이 없는 대신 표본 100만 개에서 객체 오버헤드로 수십 MB를 더 씁니다.

---

## 3.9 Best Practices & 안티패턴

**이렇게 하세요.**

- ✅ 분기 결과는 `let x = if/match { ... }`로 바인딩하세요. 외부에 가변 변수를 두고 분기마다 덮어쓰지 마세요.
- ✅ 변수는 기본적으로 `let`(불변)으로 선언하고, 변경이 *진짜* 필요할 때만 `mut`을 붙이세요. `mut`이 보이면 리뷰어가 "여기서 상태가 바뀐다"를 인지합니다.
- ✅ 타입 변환은 의도에 맞는 도구로. 넓히기는 `.into()`, 좁히기는 `try_from`(실패를 `Result`로), 비트 조작이 의도일 때만 `as`.
- ✅ 오버플로 가능성이 있는 산술은 `checked_*`/`saturating_*`/`wrapping_*` 중 *의도에 맞는 것*을 명시하세요. 기본 `+`에 의존하면 디버그 패닉 / 릴리스 래핑으로 동작이 갈립니다.
- ✅ `match`에서 와일드카드 `_`는 신중히. 미래에 변형(variant)이 추가될 때 컴파일러가 잡아주는 안전망을 스스로 끄는 행위입니다. 정말 "나머지 전부 동일"일 때만 쓰세요.
- ✅ 함수의 반환값은 마지막 표현식(세미콜론 생략)으로, 조기 종료에만 `return`을 쓰세요. `cargo clippy`의 `needless_return` 린트가 이를 강제합니다.

**이건 피하세요.**

- ❌ C 습관으로 `return value;`를 마지막 줄에 쓰는 것 — 동작은 하지만 비관용적이고 clippy가 경고합니다.
- ❌ `let mut result = Vec::new();` 후 루프에서 `push`하는 명령형 패턴 — 대부분 `iter().map().collect()`(Chapter 15)로 더 짧고 빠르게 표현됩니다.
- ❌ `f64`를 `==`로 직접 비교 — `clippy::float_cmp` 경고. 허용 오차를 두세요.
- ❌ 의미 없이 큰 정수 타입 사용 — 0~100 범위 값에 `i64`를 쓰면 메모리와 캐시를 낭비합니다. 도메인에 맞는 폭(`u8`, `u16`)을 고르세요. 단, 인덱스/길이는 항상 `usize`.

---

## 3.10 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: "타입은 알아서 맞춰지겠지" — 암묵적 형변환 기대**

파이썬의 `True + 1 == 2`, `3 + 2.0 == 5.0`에 익숙해진 손은 Rust에서 즉시 막힙니다. `let x: i32 = 10; let y: i64 = 20; let z = x + y;`는 E0308입니다. **해결:** 의도한 타입으로 명시 변환(`x as i64 + y` 또는 `i64::from(x) + y`). 애초에 한 함수 안의 정수 타입을 통일하세요.

**함정 2: 세미콜론 하나로 함수가 `()`를 반환**

파이썬에서 마지막 줄 표현식은 그냥 평가만 되고 버려집니다(반환하려면 `return` 필요). Rust 초심자는 반대 실수를 합니다. 반환하려던 마지막 줄에 무심코 `;`를 붙여 값을 버리고 `()`를 반환하게 만드는 것. **해결:** `error[E0308]: expected i32, found ()` + `help: remove this semicolon`을 보면 마지막 표현식의 세미콜론을 지우세요.

**함정 3: 오버플로를 "어차피 큰 수니까 괜찮다"고 무시**

파이썬 `int`는 무한하니 누적 합산을 마음 놓고 했습니다. Rust에서 `u32` 카운터에 계속 더하다 21억을 넘기면, 디버그에서 패닉, 릴리스에서 조용히 0으로 래핑됩니다. 금융·집계 코드에서 특히 치명적입니다. **해결:** 합산 대상을 `u64`/`i64`로 넓히거나, `checked_add`로 오버플로를 `Option`/`Result`로 다루세요(3.8의 `as u64` 패턴 참고).

**함정 4: `as`를 만능 변환기로 남용**

`value as u8`이 항상 "u8로 바꿔준다"고 믿는 것. 실제로는 `300 as u8 == 44`처럼 *조용히 데이터를 손상*시킵니다. **해결:** 좁히는 변환은 `u8::try_from(value)`로 범위를 벗어나면 `Err`를 받으세요. `as`는 손실을 *의도*했을 때만.

**함정 5: 가변 변수 + `if` 문으로 값 만들기**

```rust
// 안티패턴: 파이썬 습관 그대로
let mut grade = String::new();
if score >= 90 { grade = "A".to_string(); }
else if score >= 80 { grade = "B".to_string(); }
// ... grade가 빈 문자열로 남을 위험

// Idiomatic: 표현식으로
let grade = if score >= 90 { "A" }
            else if score >= 80 { "B" }
            else { "F" };
```

표현식 버전은 가짜 초기값이 없고, 모든 경로가 값을 만들도록 컴파일러가 강제합니다.

---

이번 챕터에서 우리는 Rust의 문법이 단순히 "C처럼 생긴 파이썬"이 아니라, 표현식 기반의 사고와 메모리를 바이트 단위로 통제하는 타입 시스템 위에 세워졌음을 확인했습니다. `i32`는 정확히 4바이트이고, `()`는 0바이트이며, 구조체의 크기는 정렬과 패딩으로 결정되고, 정수는 폭을 넘으면 명시적으로 처리해야 합니다. 이 모든 것이 가리키는 한 가지 질문이 있습니다. **"이 데이터는 정확히 메모리 어디에, 얼마나, 언제까지 존재하는가?"**

파이썬은 이 질문을 가비지 컬렉터에게 떠넘겼습니다. 편리했지만, 예측 불가능한 GC 일시정지와 객체 헤더의 무게를 대가로 치렀습니다. Rust는 이 질문에 *컴파일 타임에* 답합니다. 그 답을 내는 메커니즘이 바로 다음 챕터의 주제, **소유권(Ownership)** 입니다. 스택에 사는 4바이트 `i32`와 힙으로 가는 `String`이 변수 할당 시 각각 어떻게 다르게 움직이는지, GC 없이도 메모리가 정확히 0의 런타임 오버헤드로 해제되는 원리를 Chapter 4에서 파헤쳐 보겠습니다.

---

## References

- https://doc.rust-lang.org/book/ch03-00-common-programming-concepts.html
- https://doc.rust-lang.org/book/ch03-02-data-types.html
- https://doc.rust-lang.org/book/ch03-03-how-functions-work.html
- https://doc.rust-lang.org/reference/expressions.html
- https://doc.rust-lang.org/reference/type-layout.html
- https://doc.rust-lang.org/std/mem/fn.size_of.html
