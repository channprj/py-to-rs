# 부록 G. 바로 베끼는 예제와 디자인 패턴

![부록 G. 바로 베끼는 예제와 디자인 패턴](../images/appendix-g-patterns.jpg)

이 부록은 장을 순서대로 읽다 말고, 코드를 쓰다 막혔을 때 펼치는 참조입니다. 각 블록은 그 자체로 `fn main`이 있는 프로그램입니다. 발췌가 아닙니다. 본문 예제와 같이 **edition 2021**입니다. `cargo new`가 만들어 주는 2024와 섞지 마십시오. 저장한 뒤 이렇게 실행합니다.

```bash
rustc --edition 2021 example.rs && ./example
```

G.1의 여섯 시나리오는 서로 다른 교훈입니다. G.3은 같은 규칙으로 고른 짧은 실무 장면입니다. 결제 상태 기계, `Vector2D`, 제로카피 설정 파서, 상한이 있는 워커 큐, HTTP 크롤러, 센서 로그 집계, `clap` CLI, Axum 서비스는 이미 본문에 있으므로 여기서 다시 찍지 않습니다. G.2와 G.4는 이름과 권장 형태가 있고, 파이썬에서 그 형태로 손이 가는 지점을 옆에 적었습니다.

| 절 | 알아볼 일 | 영역 |
|----|-----------|------|
| G.1.1 | 돈 계산을 정수 새 타입으로 잠그기 | 데이터 모델링 |
| G.1.2 | 포트 문자열을 `Result`로 거절하기 | 에러 |
| G.1.3 | 순서를 지키며 태그 중복 제거 | 컬렉션·이터레이터 |
| G.1.4 | 스코프 스레드로 조각 길이 합치기 | 동시성 |
| G.1.5 | 표로 돌리는 슬러그 검사 | 테스트 |
| G.1.6 | 짝이 맞는 로그만 만드는 매크로 | 매크로 |
| G.2 | 뉴타입, 타입스테이트, 빌더, 전략, RAII | 패턴 참조 |
| G.3.1 | 이모지가 낀 미리보기를 문자로 자르기 | 문자열 |
| G.3.2 | 자판기 거스름을 큰 동전부터 나누기 | 계산 |
| G.3.3 | 같은 멱등 키는 한 번만 반영하기 | 컬렉션 |
| G.3.4 | 연속 실패 세 번이면 호출을 거절하기 | 상태 |
| G.3.5 | 점수와 이름을 묶어 순위 매기기 | 이터레이터 |
| G.3.6 | 버전을 문자열이 아니라 숫자로 비교하기 | 데이터 모델링 |
| G.4 | 확장 트레이트, 커맨드, 데코레이터, 어댑터, TryFrom | 패턴 참조 |

---

## G.1 시나리오

### G.1.1 원화 잔액은 그냥 `int`가 아닙니다

파이썬에서는 잔액을 `int`로 두고 사용자 아이디와 더하는 사고가 런타임까지 갑니다. 단위가 같은 값만 더하고, 오버플로와 잔액 부족은 `None`이 아니라 호출부가 처리하게 두는 편이 맞습니다. 입력은 시작 잔액 10000원, 결제 3500원, 환불 500원입니다. 출력은 `balance=7000`입니다.

```python
def charge(balance: int, amount: int) -> int:
    if amount > balance:
        raise ValueError("insufficient")
    return balance - amount
```

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
struct Won(u64);

impl Won {
    fn new(value: u64) -> Self {
        Self(value)
    }

    fn checked_add(self, other: Self) -> Option<Self> {
        self.0.checked_add(other.0).map(Self)
    }

    fn checked_sub(self, other: Self) -> Option<Self> {
        self.0.checked_sub(other.0).map(Self)
    }
}

fn main() {
    let start = Won::new(10_000);
    let charged = start.checked_sub(Won::new(3_500)).expect("charge");
    let balance = charged.checked_add(Won::new(500)).expect("refund");
    println!("balance={}", balance.0);
}
```

```text
balance=7000
```

`Won`과 `u64` 사용자 아이디는 다른 타입이므로 서로 더하면 컴파일되지 않습니다. 산술만 새 타입으로 잠급니다. 이메일처럼 "형식 검증이 본업"인 값의 권장 형태는 아래 G.2.1 뉴타입에 따로 있습니다. 구조체로 필드를 숨기는 이야기 자체는 [Chapter 7](../chapters/chapter_07.md)과 [Chapter 8](../chapters/chapter_08.md)에 있습니다.

### G.1.2 포트 문자열은 예외 계층 없이 거절합니다

파이썬 `int(os.environ["PORT"])`는 빈 문자열, `nope`, `0`을 서로 다른 예외로 흩뿌립니다. 호출부가 필요한 정보는 "어떤 문자열이 포트가 아닌가" 하나입니다. 입력은 `"8080"`과 `"nope"`입니다. 출력은 두 줄입니다.

```python
def parse_port(raw: str) -> int:
    port = int(raw)
    if port <= 0 or port > 65535:
        raise ValueError(raw)
    return port
```

```rust
enum PortError {
    Invalid { raw: String },
}

fn parse_port(raw: &str) -> Result<u16, PortError> {
    match raw.parse::<u16>() {
        Ok(port) if port != 0 => Ok(port),
        _ => Err(PortError::Invalid {
            raw: raw.to_string(),
        }),
    }
}

fn report(raw: &str) {
    match parse_port(raw) {
        Ok(port) => println!("{port}"),
        Err(PortError::Invalid { raw }) => println!("invalid port: {raw}"),
    }
}

fn main() {
    report("8080");
    report("nope");
}
```

```text
8080
invalid port: nope
```

`u16` 파싱이 실패하거나 0이면 같은 `Invalid`입니다. `thiserror`로 `Display`를 붙이는 확장은 [Chapter 13](../chapters/chapter_13.md)의 일입니다. 여기서의 교훈은 실패를 값으로 돌려 두 입력을 한 프로그램에서 모두 보여주는 것입니다. `?`가 없는 `main`은 라이브러리 함수의 `Result`를 삼키지 않습니다.

### G.1.3 태그는 처음 나온 순서를 유지한 채 한 번만 남깁니다

파이썬 `list(dict.fromkeys(tags))`와 같은 일입니다. `HashSet`만 쓰면 순서가 사라지고, `Vec`만 쓰면 중복 검사가 매번 선형입니다. 입력은 `rust,python,rust,go,python`입니다. 출력은 `rust,python,go`입니다.

```python
def unique_in_order(tags: str) -> str:
    seen: dict[str, None] = dict.fromkeys(tags.split(","))
    return ",".join(seen)
```

```rust
use std::collections::HashSet;

fn unique_in_order(tags: &str) -> String {
    let mut seen = HashSet::new();
    let mut out = Vec::new();
    for tag in tags.split(',') {
        if seen.insert(tag) {
            out.push(tag);
        }
    }
    out.join(",")
}

fn main() {
    println!("{}", unique_in_order("rust,python,rust,go,python"));
}
```

```text
rust,python,go
```

`HashSet::insert`는 새로 들어갔을 때만 `true`입니다. 출력 순서는 `Vec`이 책임집니다. 이터레이터로 접는 더 큰 파이프는 [Chapter 15](../chapters/chapter_15.md), `HashSet`의 소유권은 [Chapter 14](../chapters/chapter_14.md)를 보십시오. 그룹 집계나 센서 로그와는 다른 문제입니다.

### G.1.4 독립된 조각의 길이는 스코프 안에서만 병렬로 셉니다

로그 조각 세 개의 문자 수는 서로 의존하지 않습니다. 파이썬이라면 `ThreadPoolExecutor`에 함수를 넣고 `shutdown`을 기다립니다. 결과를 모은 뒤 스레드가 바깥 값을 붙잡고 남지 않게 하려면 `std::thread::scope`가 맞습니다. 상한 큐나 채널 워커가 아닙니다. 입력은 `"alpha"`, `"beta"`, `"gamma"`입니다. 출력은 `chars=14`입니다. 5 + 4 + 5입니다.

```python
from concurrent.futures import ThreadPoolExecutor


def total_chars(parts: list[str]) -> int:
    with ThreadPoolExecutor(max_workers=3) as pool:
        return sum(pool.map(len, parts))
```

```rust
fn main() {
    let parts = ["alpha", "beta", "gamma"];
    let total = std::thread::scope(|scope| {
        let mut handles = Vec::new();
        for part in parts {
            handles.push(scope.spawn(move || part.chars().count()));
        }
        handles
            .into_iter()
            .map(|handle| handle.join().expect("worker"))
            .sum::<usize>()
    });
    println!("chars={total}");
}
```

```text
chars=14
```

`scope`가 끝나면 스레드도 끝납니다. 조인 전의 핸들을 스코프 밖에 저장할 수 없습니다. 공유 상태가 필요해지면 [Chapter 16](../chapters/chapter_16.md)과 [Chapter 17](../chapters/chapter_17.md)로 가십시오. 이 입력에는 `Mutex`가 필요 없습니다.

### G.1.5 슬러그 규칙은 표의 행마다 같은 함수로 확인합니다

파이썬 `pytest.mark.parametrize`에 해당합니다. 규칙을 산문으로만 적으면 다음 사람이 공백 두 칸을 다르게 읽습니다. 표가 명세입니다. 입력 세 쌍은 `Hello World` → `hello-world`, `A  B` → `a-b`, 빈 문자열 → 빈 문자열입니다. 세 쌍이 모두 맞으면 출력은 `slug checks=3`입니다. 하나라도 다르면 `assert_eq!`가 패닉합니다.

```python
import re


def slug(value: str) -> str:
    lowered = re.sub(r"[^a-z0-9]+", "-", value.lower()).strip("-")
    return lowered
```

```rust
fn slug(input: &str) -> String {
    let mut out = String::new();
    let mut prev_dash = false;
    for ch in input.chars() {
        if ch.is_ascii_alphanumeric() {
            out.push(ch.to_ascii_lowercase());
            prev_dash = false;
        } else if !prev_dash && !out.is_empty() {
            out.push('-');
            prev_dash = true;
        }
    }
    while out.ends_with('-') {
        out.pop();
    }
    out
}

fn main() {
    let cases = [("Hello World", "hello-world"), ("A  B", "a-b"), ("", "")];
    let mut passed = 0;
    for (input, expected) in cases {
        assert_eq!(slug(input), expected, "input {input:?}");
        passed += 1;
    }
    println!("slug checks={passed}");
}
```

```text
slug checks=3
```

ASCII 문자와 숫자 사이가 아닌 문자는 대시 하나로 접습니다. 앞뒤 대시는 없습니다. 표 주도 테스트와 `assert_eq!`의 자리는 [Chapter 23](../chapters/chapter_23.md)입니다. 여기서는 크레이트 없이 같은 확인을 `main`이 수행합니다.

### G.1.6 로그 필드는 키와 값이 짝일 때만 컴파일됩니다

파이썬 로거에 `extra={"job": "nightly", "status": "ok"}`를 넘기면 키를 하나 빠뜨려도 실행됩니다. 고정된 필드 나열은 선언 매크로가 짝을 세게 할 수 있습니다. 입력은 키 `job`/`status`와 값 `nightly`/`ok`입니다. 출력은 `job=nightly status=ok`입니다.

```python
def kv(**fields: str) -> str:
    return " ".join(f"{key}={value}" for key, value in fields.items())
```

```rust
macro_rules! kv {
    ($($key:literal, $value:expr),+ $(,)?) => {{
        let mut line = String::new();
        $(
            if !line.is_empty() {
                line.push(' ');
            }
            line.push_str($key);
            line.push('=');
            line.push_str(&$value.to_string());
        )+
        line
    }};
}

fn main() {
    println!("{}", kv!("job", "nightly", "status", "ok"));
}
```

```text
job=nightly status=ok
```

인자가 홀수면 이 매크로 규칙에 맞지 않아 컴파일되지 않습니다. 반복과 위생은 [Chapter 19](../chapters/chapter_19.md)에서 다룹니다. 이 예제는 그 장의 예제를 다시 실은 것이 아니라, 로그 한 줄을 만드는 완전한 프로그램입니다.

---

## G.2 디자인 패턴

패턴은 이름보다 **언제 손이 가는가**와 **어떤 형태를 권하는가**가 중요합니다. 아래 코드가 각 패턴의 참조 구현입니다. `// ...`으로 중간을 지우지 않았습니다.

| 패턴 | 파이썬에서 손이 가는 곳 | 권장 형태 |
|------|-------------------------|-----------|
| 뉴타입 | 어디서나 `str`인 이메일 | 비공개 필드를 생성자만 채운다 |
| 타입스테이트 | `"draft"` 문자열을 if로 검사 | 메서드가 다음 타입을 반환한다 |
| 빌더 | 기본값이 많은 키워드 인자 | 소비하는 빌더가 최종 값을 만든다 |
| 전략 | 가격 함수를 인자로 전달 | 트레이트를 제네릭으로 받는다 |
| RAII 가드 | `with`로 진입과 해제를 묶음 | `Drop`은 해제만 한다 |

### G.2.1 뉴타입 (Newtype)

이메일 주소와 사용자 표시 이름은 둘 다 문자열이지만 역할이 다릅니다. 파이썬은 둘 다 `str`이라 인자를 바꿔 넣어도 함수가 받습니다. 권장 형태는 튜플 구조체 하나를 만들고, 소문자화와 `@` 검사를 생성자 안에 두는 것입니다. 필드를 공개하면 호출부가 검사를 우회합니다. 피할 것: `type Email = String` 같은 별칭. 별칭은 새 타입이 아니라서 아무 문자열이나 통과합니다. 입력은 `Ada@Example.com`입니다. 출력은 `ada@example.com`입니다.

```rust
struct Email(String);

impl Email {
    fn parse(raw: &str) -> Option<Self> {
        let normalized = raw.trim().to_ascii_lowercase();
        let at = normalized.find('@');
        let ok = match at {
            Some(index) => index > 0 && index + 1 < normalized.len() && !normalized[index + 1..].contains('@'),
            None => false,
        };
        if ok {
            Some(Self(normalized))
        } else {
            None
        }
    }

    fn as_str(&self) -> &str {
        &self.0
    }
}

fn main() {
    let email = Email::parse("Ada@Example.com").expect("email");
    println!("{}", email.as_str());
}
```

```text
ada@example.com
```

> 💡 **Under the Hood**: `Email`은 런타임에 `String` 하나와 같은 레이아웃입니다. 구분은 타입 검사기에만 있고, 가상 함수 테이블은 없습니다. 래퍼를 벗기는 비용은 없습니다.

새 타입으로 단위를 나누는 앞선 설명은 [Chapter 8](../chapters/chapter_08.md)에 있습니다. G.1.1의 `Won`은 검사 산술이고, 여기의 `Email`은 경계에서의 형식 검증입니다.

### G.2.2 타입스테이트 (Typestate)

초안 문서에 `publish()`를 허용하면, 파이썬은 `if status != "titled": raise`로 막습니다. 그 검사는 빼먹기 쉽습니다. 권장 형태는 상태마다 다른 타입을 두고, 허용된 메서드만 다음 타입을 반환하게 하는 것입니다. `BlankReport`에는 `publish`가 없습니다. 제목을 넣어야 `TitledReport`가 되고, 그 타입에만 `publish`가 있습니다. 결제 상태 기계(Pending, Success, Failed)의 열거형이 아닙니다. 열거형은 [Chapter 10](../chapters/chapter_10.md)의 도구이고, 여기서는 "지금 이 메서드를 호출할 수 있는가"를 타입으로 지웁니다. 입력 제목은 `weekly`입니다. 출력은 `published:weekly`입니다.

```rust
struct BlankReport;

struct TitledReport {
    title: String,
}

struct PublishedReport {
    title: String,
}

impl BlankReport {
    fn title(self, title: &str) -> TitledReport {
        TitledReport {
            title: title.to_string(),
        }
    }
}

impl TitledReport {
    fn publish(self) -> PublishedReport {
        PublishedReport { title: self.title }
    }
}

impl PublishedReport {
    fn label(&self) -> String {
        format!("published:{}", self.title)
    }
}

fn main() {
    let report = BlankReport.title("weekly").publish();
    println!("{}", report.label());
}
```

```text
published:weekly
```

피할 것: 모든 상태 메서드를 한 구조체에 두고 내부 `enum`을 실행 중에 검사만 하는 형태. 그건 타입스테이트가 아니라 런타임 상태 기계입니다. 호출 순서를 컴파일러가 기억할 필요가 없으면 열거형이면 충분합니다.

### G.2.3 빌더 (Builder)

검색 조건은 필수 검색어와, 빼먹기 쉬운 기본값(상한 10, 오프셋 0)으로 갈립니다. 파이썬은 키워드 인자로 해결합니다. 러스트 함수 인자는 이름만으로 생략되지 않습니다. 권장 형태는 빌더를 값으로 소비하고, `build`가 최종 구조체를 만드는 것입니다. 이 예제는 HTTP 서버가 아닙니다. 질의 문자열을 만들 뿐입니다. 첫 입력은 검색어 `rust`와 상한 5이고 오프셋은 기본값 0입니다. 둘째 입력은 검색어 `rust`와 오프셋 3이고 상한은 기본값 10입니다. 출력은 두 줄입니다.

```rust
struct Query {
    term: String,
    limit: u32,
    offset: u32,
}

struct QueryBuilder {
    term: String,
    limit: u32,
    offset: u32,
}

impl QueryBuilder {
    fn new(term: impl Into<String>) -> Self {
        Self {
            term: term.into(),
            limit: 10,
            offset: 0,
        }
    }

    fn limit(mut self, limit: u32) -> Self {
        self.limit = limit;
        self
    }

    fn offset(mut self, offset: u32) -> Self {
        self.offset = offset;
        self
    }

    fn build(self) -> Query {
        Query {
            term: self.term,
            limit: self.limit,
            offset: self.offset,
        }
    }
}

impl Query {
    fn as_query(&self) -> String {
        format!("q={}&limit={}&offset={}", self.term, self.limit, self.offset)
    }
}

fn main() {
    let limited = QueryBuilder::new("rust").limit(5).build();
    let shifted = QueryBuilder::new("rust").offset(3).build();
    println!("{}", limited.as_query());
    println!("{}", shifted.as_query());
}
```

```text
q=rust&limit=5&offset=0
q=rust&limit=10&offset=3
```

피할 것: 빌더를 `&mut self`로만 두고 호출부가 반쯤 채운 빌더를 최종 값처럼 쓰는 것. 최종 값은 `build`의 반환뿐입니다. 생성 패턴의 배경은 [Chapter 11](../chapters/chapter_11.md)에 있습니다. 여기 프로그램은 그 장의 예제와 다른 질의 객체입니다.

### G.2.4 전략 (Strategy)

정가와 10% 할인은 같은 "가격을 고친다"는 일이지만 규칙이 다릅니다. 파이썬은 함수를 넘깁니다. 권장 형태는 트레이트 하나와, 그 트레이트를 제네릭으로 받는 함수입니다. 구현이 컴파일 시점에 알려져 있으면 `dyn`으로 지울 이유가 없습니다. 입력 정가는 2000이고 할인율은 10입니다. 정수 나눗셈으로 200을 뺍니다. 출력은 `list=2000 off=1800`입니다.

```rust
trait Pricer {
    fn apply(&self, list: u64) -> u64;
}

struct ListPrice;

impl Pricer for ListPrice {
    fn apply(&self, list: u64) -> u64 {
        list
    }
}

struct PercentOff {
    percent: u64,
}

impl Pricer for PercentOff {
    fn apply(&self, list: u64) -> u64 {
        list - list * self.percent / 100
    }
}

fn charge<P: Pricer>(pricer: &P, list: u64) -> u64 {
    pricer.apply(list)
}

fn main() {
    let list = charge(&ListPrice, 2000);
    let off = charge(&PercentOff { percent: 10 }, 2000);
    println!("list={list} off={off}");
}
```

```text
list=2000 off=1800
```

> 💡 **Under the Hood**: `charge`는 `ListPrice`와 `PercentOff`에 대해 각각 구체 함수로 단일형화됩니다. 호출은 정적 디스패치이고, 트레이트 객체의 포인터 쌍이 힙에 생기지 않습니다.

피할 것: 가격 종류를 문자열 `"list"` / `"off"`로 받아 함수 안에서 `if`를 늘리는 것. 규칙을 추가할 때마다 그 함수가 깨집니다. 트레이트로 동작을 고정하는 설명은 [Chapter 9](../chapters/chapter_09.md)에 있습니다.

### G.2.5 RAII 가드 (Guard)

파이썬 `with`는 진입과 해제를 한 블록에 묶습니다. 러스트에서 같은 자리는 `Drop`입니다. 권장 형태는 가드 값이 생기는 순간 진입을 기록하고, 스코프가 끝나는 순간 해제만 기록하는 것입니다. 입력은 바깥 구간 이름 `outer`와 안쪽 구간 이름 `inner`이고, 안쪽에서 `work`를 한 줄 찍습니다. 출력은 아래 다섯 줄입니다.

```rust
struct Section {
    name: String,
}

impl Section {
    fn enter(name: &str) -> Self {
        println!("enter {name}");
        Self {
            name: name.to_string(),
        }
    }
}

impl Drop for Section {
    fn drop(&mut self) {
        println!("leave {}", self.name);
    }
}

fn main() {
    let outer = Section::enter("outer");
    {
        let inner = Section::enter("inner");
        println!("work");
        drop(inner);
    }
    drop(outer);
}
```

```text
enter outer
enter inner
work
leave inner
leave outer
```

피할 것: `Drop` 안에서 결제를 확정하거나 네트워크 요청을 보내는 것. 해제 중 패닉은 과정을 삼키기 쉽고, 성공과 실패를 구분하지도 못합니다. `Drop`은 락을 풀고 파일을 닫는 일까지만 맡기십시오. 결정론적 해제의 설명은 [Chapter 4](../chapters/chapter_04.md)와 [Chapter 20](../chapters/chapter_20.md)에 있습니다.

---

## G.3 바로 마주치는 장면

G.1과 교훈이 겹치지 않는 짧은 프로그램입니다. 표준 라이브러리만 쓰고, 각각 `fn main`이 있습니다.

### G.3.1 미리보기는 바이트 칸이 아니라 문자로 자릅니다

파이썬 `text[:4]`는 코드 포인트를 셉니다. 러스트 `&text[..4]`는 바이트 칸입니다. `러` 한 글자가 3바이트라, 4바이트에서 자르면 문자 경계가 아니라 패닉입니다. 권장 형태는 `chars().take(n)`입니다. 입력은 `러스트🦀파이썬`의 앞 4글자입니다. 출력은 `prefix=러스트🦀 chars=4 bytes=13`입니다.

```python
def preview(text: str, n: int) -> str:
    return text[:n]
```

```rust
fn preview(text: &str, n: usize) -> String {
    text.chars().take(n).collect()
}

fn main() {
    let text = "러스트🦀파이썬";
    let head = preview(text, 4);
    let chars = head.chars().count();
    let bytes = head.len();
    println!("prefix={head} chars={chars} bytes={bytes}");
}
```

```text
prefix=러스트🦀 chars=4 bytes=13
```

`chars`는 4이고 `bytes`는 13입니다. 게 이모지 하나가 4바이트를 차지합니다. 바이트 길이로 제목을 자르는 습관은 [Chapter 14](../chapters/chapter_14.md)의 문자열 인덱싱과 같은 함정입니다.

### G.3.2 거스름은 큰 동전부터 나눕니다

자판기에 1300원을 넣고 2000원을 냈습니다. 거스름 700원을 500원, 100원, 50원, 10원 순서로 나눕니다. 파이썬 `//`와 러스트 정수 나눗셈은 같은 방향으로 버립니다. 입력은 가격 1300과 지불 2000입니다. 출력은 `500x1,100x2`입니다.

```python
def change(due: int) -> str:
    parts: list[str] = []
    for coin in (500, 100, 50, 10):
        count, due = divmod(due, coin)
        if count:
            parts.append(f"{coin}x{count}")
    return ",".join(parts)
```

```rust
fn change(price: u32, paid: u32) -> String {
    let mut due = paid - price;
    let mut parts = Vec::new();
    for coin in [500, 100, 50, 10] {
        let count = due / coin;
        if count > 0 {
            parts.push(format!("{coin}x{count}"));
            due -= count * coin;
        }
    }
    parts.join(",")
}

fn main() {
    println!("{}", change(1_300, 2_000));
}
```

```text
500x1,100x2
```

이 입력은 50원과 10원 칸이 0이라 출력에 나오지 않습니다. 동전이 액면가를 나누지 못하면 나머지가 남습니다. 그 경우는 이 프로그램이 다루지 않습니다. 잔액 타입 자체는 G.1.1의 `Won`이고, 여기서의 교훈은 나누는 순서입니다.

### G.3.3 같은 멱등 키는 두 번째 금액을 버립니다

파이썬 API에서 `Idempotency-Key`를 딕셔너리에 넣어 재시도를 막는 패턴입니다. 같은 키의 두 번째 호출은 저장해 둔 금액을 돌려주고, 새로 들어온 금액은 반영하지 않습니다. 입력은 `pay-1`에 10, 다시 `pay-1`에 99, 이어서 `pay-2`에 3입니다. 출력은 세 줄입니다.

```python
def apply(done: dict[str, int], key: str, amount: int) -> tuple[str, int]:
    if key in done:
        return ("replay", done[key])
    done[key] = amount
    return ("applied", amount)
```

```rust
use std::collections::HashMap;

struct Ledger {
    done: HashMap<String, u32>,
}

impl Ledger {
    fn apply(&mut self, key: &str, amount: u32) -> (&'static str, u32) {
        if let Some(saved) = self.done.get(key) {
            return ("replay", *saved);
        }
        self.done.insert(key.to_string(), amount);
        ("applied", amount)
    }
}

fn main() {
    let mut ledger = Ledger {
        done: HashMap::new(),
    };
    for (key, amount) in [("pay-1", 10), ("pay-1", 99), ("pay-2", 3)] {
        let (kind, value) = ledger.apply(key, amount);
        println!("{kind} {key}={value}");
    }
}
```

```text
applied pay-1=10
replay pay-1=10
applied pay-2=3
```

두 번째 `pay-1`의 99는 장부에 들어가지 않습니다. 이 `HashMap`은 프로세스가 끝나면 사라집니다. 요청 사이까지 지키려면 저장소를 밖에 둡니다. 맵을 소유하는 구조는 [Chapter 14](../chapters/chapter_14.md)에 있습니다. G.1.3은 순서를 남기는 중복 제거이고, 여기의 교훈은 재시도가 합계를 두 번 더하지 않는 것입니다.

### G.3.4 세 번 실패하면 다음 호출은 열지도 않습니다

파이썬에서 실패 횟수를 인스턴스 속성에 세고, 임계값을 넘으면 함수 본문에 들어가지 않게 막는 회로 차단기입니다. 이 예제는 시계가 없습니다. 입력 네 번은 실패, 실패, 실패, 성공 시도입니다. 세 번째에서 열리고, 네 번째는 본문을 실행하지 않습니다. 출력은 네 줄입니다.

```python
class Breaker:
    def __init__(self) -> None:
        self.failures = 0
        self.open = False

    def call(self, ok: bool) -> str:
        if self.open:
            return "rejected"
        if ok:
            self.failures = 0
            return "ok"
        self.failures += 1
        if self.failures >= 3:
            self.open = True
            return "opened"
        return "fail"
```

```rust
struct Breaker {
    failures: u32,
    open: bool,
}

impl Breaker {
    fn call(&mut self, ok: bool) -> &'static str {
        if self.open {
            return "rejected";
        }
        if ok {
            self.failures = 0;
            "ok"
        } else {
            self.failures += 1;
            if self.failures >= 3 {
                self.open = true;
                "opened"
            } else {
                "fail"
            }
        }
    }
}

fn main() {
    let mut breaker = Breaker {
        failures: 0,
        open: false,
    };
    for ok in [false, false, false, true] {
        println!("{}", breaker.call(ok));
    }
}
```

```text
fail
fail
opened
rejected
```

성공 시도가 `rejected`인 이유는 이미 열려 있기 때문입니다. 시간이 지난 뒤 한 번만 시험하는 반열림은 이 프로그램에 없습니다. 그 상태가 필요하면 타임스탬프를 필드에 더합니다. 실행 중에 세는 차단기와, 호출 자체를 타입으로 지우는 상태 기계는 다른 도구입니다. 후자는 [Chapter 10](../chapters/chapter_10.md)에 있습니다.

### G.3.5 같은 점수는 이름 순으로 줄을 세웁니다

파이썬 `zip` 후 `sorted`에 점수 내림차순, 이름 오름차순 키를 주는 일과 같습니다. 입력은 ada 40, lin 90, bea 90입니다. 90점 동점은 이름이 앞선 bea가 위입니다. 출력은 세 줄입니다.

```python
def board(names: list[str], scores: list[int]) -> list[tuple[str, int]]:
    return sorted(zip(names, scores), key=lambda row: (-row[1], row[0]))
```

```rust
fn main() {
    let names = ["ada", "lin", "bea"];
    let scores = [40, 90, 90];
    let mut rows: Vec<(&str, u32)> = names.into_iter().zip(scores).collect();
    rows.sort_by(|left, right| right.1.cmp(&left.1).then(left.0.cmp(right.0)));
    for (name, score) in rows {
        println!("{name} {score}");
    }
}
```

```text
bea 90
lin 90
ada 40
```

`zip`은 짧은 쪽이 끝나면 멈춥니다. 이 입력은 양쪽 길이가 같습니다. 이터레이터로 두 줄을 묶는 설명은 [Chapter 15](../chapters/chapter_15.md)에 있습니다.

### G.3.6 `1.10.0`은 `1.9.0`보다 뒤입니다

파이썬에서 `"1.10.0" < "1.9.0"`은 참입니다. 첫 글자 `1` 다음의 `1`이 `9`보다 앞이기 때문입니다. 버전은 문자 순서가 아니라 `(major, minor, patch)` 숫자 순서입니다. 입력은 `1.9.0`과 `1.10.0`입니다. 출력은 `1.9.0 < 1.10.0`입니다.

```python
def parse(raw: str) -> tuple[int, int, int]:
    major, minor, patch = raw.split(".")
    return (int(major), int(minor), int(patch))
```

```rust
fn parse(raw: &str) -> (u32, u32, u32) {
    let mut parts = raw.split('.');
    let major = parts.next().unwrap().parse().unwrap();
    let minor = parts.next().unwrap().parse().unwrap();
    let patch = parts.next().unwrap().parse().unwrap();
    (major, minor, patch)
}

fn main() {
    let older = parse("1.9.0");
    let newer = parse("1.10.0");
    let order = if older < newer {
        "<"
    } else if older > newer {
        ">"
    } else {
        "="
    };
    println!("1.9.0 {order} 1.10.0");
}
```

```text
1.9.0 < 1.10.0
```

사전 릴리스(`-rc.1`)와 빌드 메타데이터는 이 세 칸에 없습니다. 그 문법이 필요하면 파서를 칸 단위로 늘립니다. 세 숫자를 이름으로 묶고 싶으면 [Chapter 7](../chapters/chapter_07.md)의 구조체로 올리면 됩니다.

---

## G.4 이어서 베끼는 패턴

G.2에 없는 이름입니다. 각 블록이 참조 구현이고, 피할 형태를 문장으로 적었습니다.

| 패턴 | 파이썬에서 손이 가는 곳 | 권장 형태 |
|------|-------------------------|-----------|
| 확장 트레이트 | `str`에 메서드가 없어 함수로 호출 | 로컬 트레이트를 외래 타입에 구현 |
| 커맨드 | `"add"` 문자열을 if로 분기 | 열거형 하나가 명령과 인자를 같이 담음 |
| 데코레이터 | 호출 횟수를 세는 래퍼 | 클로저를 가진 제네릭 구조체 |
| 어댑터 | `"9:15"`를 분으로 직접 계산 | `From`이 경계에서 구조체를 만듦 |
| TryFrom | 0에서 100을 넘는 정수를 그대로 캐스팅 | 실패하는 변환은 `TryFrom` |

### G.4.1 확장 트레이트 (Extension Trait)

`"  ".strip() == ""`를 메서드처럼 쓰고 싶을 때, 파이썬은 함수를 하나 더 두거나 문자열을 상속합니다. `str`은 외래 타입이라 러스트에서 `impl str`에 메서드를 추가할 수 없습니다. 권장 형태는 로컬 트레이트를 만들어 `&str`에 구현하는 것입니다. 입력은 공백만 있는 문자열과 `"ok"`입니다. 출력은 `true`와 `false` 두 줄입니다.

```rust
trait Blank {
    fn is_blank(self) -> bool;
}

impl Blank for &str {
    fn is_blank(self) -> bool {
        self.trim().is_empty()
    }
}

fn main() {
    println!("{}", "  ".is_blank());
    println!("{}", "ok".is_blank());
}
```

```text
true
false
```

> 💡 **Under the Hood**: `is_blank`는 `&str`에 대해 구체 함수로 단일형화됩니다. 가상 함수 테이블이 없고, 자유 함수 하나를 메서드 문법으로 부른 것과 호출 비용이 같습니다.

피할 것: 메서드 하나 때문에 `struct Text(String)`을 만들고 모든 문자열을 그 안으로 넣는 것. 형식 검증이 필요한 값은 G.2.1 뉴타입이고, 술어 하나에는 확장 트레이트가 맞습니다. 트레이트 구현은 [Chapter 9](../chapters/chapter_09.md)에 있습니다.

### G.4.2 커맨드 (Command)

잔액을 바꾸는 요청이 문자열 `"add"` / `"sub"`로 들어오면, 철자 하나가 조용히 다른 분기를 탑니다. 권장 형태는 명령과 숫자를 한 열거형으로 받는 것입니다. 입력은 0에서 시작해 40을 더하고 15를 뺀 뒤 잔액을 찍습니다. 출력은 `balance=25`입니다. 결제 상태 기계가 아닙니다. 그쪽은 상태이고, 여기는 이미 해석된 명령입니다.

```rust
enum Command {
    Add(i32),
    Sub(i32),
    Show,
}

fn apply(balance: &mut i32, command: Command) {
    match command {
        Command::Add(amount) => *balance += amount,
        Command::Sub(amount) => *balance -= amount,
        Command::Show => println!("balance={balance}"),
    }
}

fn main() {
    let mut balance = 0;
    apply(&mut balance, Command::Add(40));
    apply(&mut balance, Command::Sub(15));
    apply(&mut balance, Command::Show);
}
```

```text
balance=25
```

> 💡 **Under the Hood**: `Command`는 태그 정수와 페이로드입니다. `Add`와 `Sub`만 `i32`를 붙이고 `Show`는 태그만 있습니다. 새 변형을 더하고 `match`에 팔을 빼먹으면 컴파일이 멈춥니다.

피할 것: `if command == "add"`처럼 명령을 문자열로 남기는 것. 열거형으로 분기를 닫는 설명은 [Chapter 10](../chapters/chapter_10.md)에 있습니다. G.2.2 타입스테이트는 "이 메서드를 지금 호출할 수 있는가"를 타입으로 지우고, 커맨드는 이미 허용된 동작을 값으로 실어 나릅니다.

### G.4.3 데코레이터 (Decorator)

파이썬 `@wraps`로 호출 횟수를 세는 래퍼와 같은 자리입니다. 러스트에서 그 자리가 늘 매크로일 필요는 없습니다. 권장 형태는 클로저를 필드로 가진 제네릭 구조체입니다. 입력은 `"pong"`을 돌려주는 함수를 두 번 호출하는 것입니다. 출력은 `pong pong calls=2`입니다.

```rust
struct Counted<F> {
    inner: F,
    calls: u32,
}

impl<F> Counted<F>
where
    F: FnMut() -> &'static str,
{
    fn call(&mut self) -> &'static str {
        self.calls += 1;
        (self.inner)()
    }
}

fn main() {
    let mut ping = Counted {
        inner: || "pong",
        calls: 0,
    };
    let first = ping.call();
    let second = ping.call();
    println!("{first} {second} calls={}", ping.calls);
}
```

```text
pong pong calls=2
```

피할 것: 호출 횟수를 `static mut`이나 프로세스 전역에 두는 것. 횟수는 래퍼 값의 필드입니다. 코드를 생성하는 데코레이터의 자리는 [Chapter 19](../chapters/chapter_19.md)의 매크로이고, 횟수를 세는 일에는 이 구조체로 충분합니다.

### G.4.4 어댑터 (Adapter, `From`)

파이썬은 `"9:15".split(":")`를 호출부마다 반복합니다. 권장 형태는 경계에서 한 번 `From<&str>`로 분 단위 구조체를 만드는 것입니다. 입력은 `9:15`와 `10:00`입니다. 출력은 `span=45`입니다.

```rust
struct Clock {
    minutes: u32,
}

impl From<&str> for Clock {
    fn from(raw: &str) -> Self {
        let (hour, minute) = raw.split_once(':').expect("clock");
        let minutes =
            hour.parse::<u32>().expect("hour") * 60 + minute.parse::<u32>().expect("minute");
        Self { minutes }
    }
}

fn main() {
    let start = Clock::from("9:15");
    let end = Clock::from("10:00");
    println!("span={}", end.minutes - start.minutes);
}
```

```text
span=45
```

`From`은 실패하지 않는 변환입니다. 이 입력은 둘 다 `시:분`이라 `expect`가 패닉하지 않습니다. 피할 것: 콜론이 없는 문자열까지 `From`으로 받고 `expect`에 맡기는 것. 거절해야 하는 입력은 다음 절의 `TryFrom`입니다. 변환 트레이트는 [Chapter 9](../chapters/chapter_09.md)에 있습니다.

### G.4.5 범위 관문 (`TryFrom`)

파이썬 `percent: int`는 150도 받습니다. `as u8`은 범위를 검사하지 않고 비트를 자릅니다. 권장 형태는 `TryFrom<u16>`에서 100을 넘는 값을 `Err`로 돌리는 것입니다. 입력은 15와 150입니다. 출력은 `rate=15`와 `out of range` 두 줄입니다.

```rust
struct Percent(u8);

impl TryFrom<u16> for Percent {
    type Error = &'static str;

    fn try_from(value: u16) -> Result<Self, Self::Error> {
        if value > 100 {
            Err("out of range")
        } else {
            Ok(Self(value as u8))
        }
    }
}

fn main() {
    match Percent::try_from(15u16) {
        Ok(Percent(value)) => println!("rate={value}"),
        Err(message) => println!("{message}"),
    }
    match Percent::try_from(150u16) {
        Ok(Percent(value)) => println!("rate={value}"),
        Err(message) => println!("{message}"),
    }
}
```

```text
rate=15
out of range
```

피할 것: 호출부가 `if value <= 100`을 기억하리라 믿고 `as u8`만 남기는 것. 문자열 포트 거절은 G.1.2이고, 이메일 형식은 G.2.1입니다. 여기서의 교훈은 실패하는 숫자 변환을 `TryFrom` 하나로 모으는 것입니다. `Result`로 거절을 돌려주는 설명은 [Chapter 12](../chapters/chapter_12.md)에 있습니다.

---

## References

- [Chapter 4](../chapters/chapter_04.md), [Chapter 20](../chapters/chapter_20.md) — `Drop`과 RAII
- [Chapter 8](../chapters/chapter_08.md) — 뉴타입
- [Chapter 9](../chapters/chapter_09.md) — 트레이트와 정적 디스패치
- [Chapter 10](../chapters/chapter_10.md) — 열거형 상태 기계. 이 부록의 타입스테이트와 다른 도구입니다
- [Chapter 11](../chapters/chapter_11.md) — 빌더
- [Chapter 13](../chapters/chapter_13.md), [Chapter 14](../chapters/chapter_14.md), [Chapter 15](../chapters/chapter_15.md) — 에러, 컬렉션, 이터레이터
- [Chapter 16](../chapters/chapter_16.md), [Chapter 17](../chapters/chapter_17.md) — 스레드와 공유 상태
- [Chapter 19](../chapters/chapter_19.md), [Chapter 23](../chapters/chapter_23.md) — 매크로와 테스트
- [Chapter 7](../chapters/chapter_07.md), [Chapter 12](../chapters/chapter_12.md), [Chapter 14](../chapters/chapter_14.md) — 구조체, `Result`, 문자열과 맵. G.3과 G.4가 가리키는 자리입니다
