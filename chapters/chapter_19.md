# Chapter 19. 매크로 (Macros): 데코레이터와 메타클래스의 대체재

파이썬 개발자라면 함수 하나에 `@`를 붙여 동작을 통째로 바꿔본 경험이 있을 것입니다. `@app.route("/users")`, `@login_required`, `@lru_cache`, `@retry(max=3)`, `@dataclass`. 이 작은 골뱅이 하나로 라우팅이 등록되고, 인증이 강제되고, 캐시가 끼워지고, `__init__`이 자동 생성됩니다. 데코레이터(Decorator)는 파이썬이 자랑하는 메타프로그래밍(Metaprogramming)의 얼굴이고, 그 우아함은 부정할 수 없습니다. 더 깊이 들어가면 메타클래스(Metaclass)가 있습니다. ORM이 모델 클래스를 정의하는 순간 컬럼을 수집하고 테이블 매핑을 만들어내는 그 마법, Django의 `models.Model`이나 SQLAlchemy의 선언적 베이스가 바로 메타클래스의 작품입니다.

하지만 10년 차 엔지니어라면 이 마법의 청구서도 받아봤을 것입니다. `@timer`를 붙인 함수의 스택 트레이스가 갑자기 `wrapper`로 가득 차서 진짜 호출 위치를 찾기 힘들었던 적, `functools.wraps`를 빼먹어서 `func.__name__`이 전부 `"wrapper"`로 나오던 적, 데코레이터가 시그니처를 `(*args, **kwargs)`로 뭉개버려서 `mypy`가 타입 추론을 포기하고 IDE 자동완성이 죽어버린 적. 그리고 무엇보다, **이 모든 일이 런타임에 일어난다**는 사실. 함수가 호출될 때마다 클로저(closure)를 한 겹 더 거치고, 메타클래스는 클래스가 정의되는 순간 인터프리터를 멈춰 세우고 딕셔너리를 뒤집니다.

Rust는 같은 목표 — 반복되는 코드를 자동 생성하고, 동작을 선언적으로 주입하는 것 — 를 정반대 시점에서 달성합니다. **컴파일 타임(Compile Time)**입니다. Rust의 매크로(Macro)는 C의 `#define` 같은 순진한 텍스트 치환이 아닙니다. 소스 코드를 **토큰 스트림(Token Stream)**으로 받아 분석하고 변형한 뒤, 새로운 Rust 코드를 뱉어내는 "코드를 짜는 코드"입니다. 그 결과물은 마치 사람이 손으로 쓴 것처럼 컴파일러의 타입 검사를 그대로 통과하고, 단일형화(Monomorphization)와 인라이닝(Inlining)의 대상이 되며, 런타임에는 **추가 비용이 0**입니다. 데코레이터가 매 호출마다 지불하는 클로저 비용을, 매크로는 컴파일 한 번으로 청산합니다.

이 대비를 한 문장으로 요약하면 이렇습니다. **파이썬의 메타프로그래밍은 실행 중인 객체를 조작하고, Rust의 메타프로그래밍은 컴파일 중인 소스를 조작합니다.** 전자는 유연하지만 비용과 불투명함을 동반하고, 후자는 경직되어 보이지만 타입 안전과 무비용 추상화(Zero-cost Abstraction)를 지킵니다.

Chapter 18을 떠올려보면, 우리는 이미 매크로의 결과물을 잔뜩 써왔습니다. `#[tokio::main]`은 `async fn main`을 일반 `fn main`으로 재작성하는 속성 매크로였고, `#[derive(Debug)]`는 `Debug` 트레이트 구현을 통째로 생성하는 derive 매크로였습니다. `println!`, `vec!`, `?` 뒤의 에러 변환까지 — Rust 프로그램은 매크로 위에서 돌아갑니다. 이번 챕터에서는 그 마법의 커튼을 걷고, 우리 손으로 직접 `macro_rules!`와 절차적 매크로(Procedural Macro)를 작성해봅니다. 그리고 `cargo expand`로 매크로가 무엇으로 펼쳐지는지 두 눈으로 확인합니다.

---

## 19.1 The Python Way — 데코레이터와 메타클래스의 런타임 청구서

먼저 우리가 익숙한 방식을 정확히 해부합시다. 다음은 실무에서 수없이 본 타이밍 데코레이터입니다.

```python
import time
import functools
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")
R = TypeVar("R")

def timer(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)             # 메타데이터 복사 (안 하면 __name__이 깨짐)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        result = func(*args, **kwargs)  # 원본 호출 — 스택이 한 겹 깊어진다
        print(f"{func.__name__} took {time.perf_counter() - start:.6f}s")
        return result
    return wrapper

@timer
def heavy_computation() -> int:
    return sum(i * i for i in range(1_000_000))
```

`@timer`가 하는 일을 정확히 따져봅시다. `heavy_computation`이라는 이름은 이제 원본 함수가 아니라 `wrapper` 함수 객체를 가리킵니다. 즉 모듈이 임포트되는 순간 — 함수가 한 번도 호출되기 전에 — 새로운 함수 객체가 힙에 생성되고, 원본 함수에 대한 참조를 클로저로 붙잡습니다. 이후 `heavy_computation()`을 부를 때마다 다음 비용이 발생합니다.

1. **추가 스택 프레임**: `wrapper`를 거쳐 `func`를 부릅니다. 호출 깊이가 한 겹 늘어납니다. 데코레이터를 여러 개 쌓으면(`@a @b @c`) 그만큼 더 깊어집니다.
2. **`*args, **kwargs` 패킹/언패킹**: 모든 인자가 튜플과 딕셔너리로 한 번 포장되었다가 풀립니다.
3. **타입 정보 손실**: `ParamSpec`이 등장하기 전(Python 3.10 미만) 코드에서는 `wrapper`의 시그니처가 `(*args, **kwargs)`로 뭉개져, 정적 타입 검사기와 IDE가 원래 시그니처를 잃어버립니다. `ParamSpec`으로 많이 개선됐지만, 인자를 변형하는 데코레이터에서는 여전히 한계가 있습니다.
4. **`functools.wraps`를 잊으면**: 스택 트레이스, `__name__`, `__doc__`, `__module__`이 전부 `wrapper`의 것으로 오염됩니다. APM 도구가 모든 함수를 `wrapper`로 집계하는 악몽이 시작됩니다.

메타클래스는 한 단계 더 깊은 흑마술입니다. 클래스가 **정의되는 순간** 개입합니다.

```python
class ModelMeta(type):
    def __new__(mcs, name, bases, namespace):
        # 클래스 본문을 스캔해 Field 인스턴스를 수집한다 (런타임에!)
        fields = {k: v for k, v in namespace.items() if isinstance(v, Field)}
        namespace["_fields"] = fields
        cls = super().__new__(mcs, name, bases, namespace)
        return cls

class Field:
    def __init__(self, col_type: str) -> None:
        self.col_type = col_type

class User(metaclass=ModelMeta):
    id = Field("INTEGER")
    name = Field("TEXT")
    # User._fields 는 클래스 정의가 끝나는 순간 자동으로 채워진다
```

`User` 클래스가 정의되는 그 순간, `ModelMeta.__new__`가 인터프리터에 의해 호출되어 클래스 네임스페이스(`namespace` 딕셔너리)를 헤집고 `_fields`를 주입합니다. 강력합니다. ORM 전체가 이 메커니즘 위에 서 있습니다. 그러나 대가가 있습니다.

- **불투명함**: `User` 클래스 어디에도 `_fields`라는 속성을 선언한 코드가 없습니다. 그 속성이 어디서 왔는지 알려면 메타클래스를 읽어야 합니다. 코드를 읽는 것만으로는 클래스의 진짜 모습을 알 수 없습니다.
- **정적 분석 불가**: `mypy`는 `User._fields`의 존재를 모릅니다. 메타클래스가 런타임에 만든 속성이기 때문입니다. 타입 검사기와 IDE가 장님이 됩니다.
- **다중 상속 충돌**: 두 부모가 서로 다른 메타클래스를 쓰면 `TypeError: metaclass conflict`로 터집니다. MRO(Method Resolution Order)와 얽히면 디버깅이 지옥이 됩니다.

핵심은 이것입니다. 파이썬의 데코레이터와 메타클래스는 **이미 만들어진 런타임 객체(함수 객체, 클래스 객체)를 사후에 조작**합니다. 그래서 유연하지만, 그 조작의 결과는 소스 코드에 드러나지 않고, 타입 시스템 바깥에 있으며, 매 실행마다 비용을 냅니다. Rust는 같은 일을 **컴파일러가 소스를 보기 전에, 소스 자체를 다시 쓰는 방식**으로 합니다.

---

## 19.2 The Rust Mental Model — 컴파일 타임 코드 생성

Rust 매크로를 이해하는 가장 빠른 길은 컴파일 파이프라인 위에 정확히 어디서 동작하는지 보는 것입니다.

> **Under the Hood: 매크로 확장은 토큰 스트림 위에서, AST 직전에 일어난다**
>
> rustc가 소스 파일을 처리하는 순서는 대략 이렇습니다.
>
> ```text
> 소스 텍스트
>    │  ① 렉싱(Lexing)
>    ▼
> 토큰 스트림(Token Stream)   ← 매크로의 입력이자 출력은 여기다
>    │  ② 파싱(Parsing) + 매크로 확장(Macro Expansion)
>    ▼
> AST (추상 구문 트리)
>    │  ③ 이름 해석 / 타입 검사 / 빌림 검사
>    ▼
> HIR → MIR → LLVM IR → 기계어
> ```
>
> 결정적인 사실: **매크로는 ②단계, 즉 타입 검사보다 한참 전에 토큰을 다룹니다.** 매크로는 `i32`인지 `String`인지 모릅니다. 매크로가 보는 것은 의미(semantics)가 아니라 구문(syntax) 조각 — 식별자, 표현식, 타입 이름 같은 **토큰의 모양**뿐입니다. 매크로가 코드를 뱉어내면, 그 결과는 마치 처음부터 사람이 쓴 코드인 것처럼 ③단계로 넘어가 **정상적인 타입 검사를 전부 받습니다.**
>
> 이것이 파이썬 데코레이터와의 본질적 차이입니다. 데코레이터는 `④ 실행 시점`에 동작하는 일반 함수이고, 매크로는 `② 파싱 시점`에 동작하는 컴파일러 플러그인입니다. 데코레이터의 산출물은 힙 위의 클로저 객체이고, 매크로의 산출물은 컴파일될 소스 코드입니다.

이 차이가 만드는 실질적 결과는 세 가지입니다.

**첫째, 런타임 비용이 0입니다.** 매크로가 생성한 코드는 컴파일 시점에 펼쳐져 그 자리에 인라인됩니다. `@timer`처럼 클로저를 한 겹 두르는 게 아니라, 타이밍 측정 코드가 함수 본문에 직접 박힙니다. 추가 스택 프레임도, 인자 재포장도 없습니다.

**둘째, 타입 안전이 유지됩니다.** 매크로가 만든 코드도 빌림 검사기(Borrow Checker)와 타입 검사기를 통과해야 합니다. 메타클래스가 `mypy`의 눈을 가렸던 것과 정반대로, 매크로 산출물은 `rust-analyzer`가 펼쳐서 분석하므로 자동완성과 타입 검사가 그대로 작동합니다.

**셋째, 변환이 명시적입니다.** 매크로 호출에는 `!`(선언적/함수형)나 `#[...]`(속성/derive)가 붙어, "여기서 코드 생성이 일어난다"는 신호를 줍니다. 그리고 `cargo expand` 한 번이면 펼쳐진 결과를 그대로 볼 수 있습니다. 메타클래스가 숨긴 `_fields`와 달리, 매크로가 만든 코드는 언제든 들춰볼 수 있습니다.

Rust 매크로는 크게 두 갈래입니다.

| 종류 | 호출 형태 | 입력 | 작성 방법 | 대표 예 |
|---|---|---|---|---|
| **선언적 매크로** (Declarative) | `name!(...)` | 토큰 패턴 | `macro_rules!` (같은 크레이트에서) | `vec!`, `println!`, `assert!` |
| **절차적 매크로** (Procedural) | `#[derive(..)]` / `#[attr]` / `name!(...)` | `TokenStream` | 별도 proc-macro 크레이트 + `syn`/`quote` | `#[derive(Debug)]`, `#[tokio::main]`, `sqlx::query!` |

선언적 매크로는 "이 패턴이 보이면 이 코드로 바꿔라"는 **선언적 치환 규칙**입니다. 정규식의 코드 버전이라고 생각하면 됩니다. 절차적 매크로는 입력 토큰을 받아 **임의의 Rust 프로그램으로** 처리하는 본격적인 컴파일러 확장입니다. 데코레이터/메타클래스에 가장 가까운 것은 절차적 매크로지만, 일상 작업의 90%는 선언적 매크로로 충분합니다. 거기서부터 시작합시다.

파이썬의 메타프로그래밍 도구와 Rust 매크로를 개념적으로 정렬하면 다음과 같습니다. 이 표가 이번 챕터 전체의 지도입니다.

| 파이썬 (런타임) | Rust (컴파일 타임) | 동작 시점 | 산출물 | 타입 검사 | 런타임 비용 |
|---|---|---|---|---|---|
| `@decorator` (함수 래핑) | 속성 매크로 `#[attr]` | 실행 / 파싱 | 클로저 객체 / 재작성된 소스 | mypy 부분적 | 124 ns+/call vs 0 |
| `@register`, `@app.route` | 속성 매크로 `#[route]` | 임포트 / 파싱 | 부수효과 등록 / 생성 코드 | 약함 / 강함 | 런타임 / 0 |
| 메타클래스 `__new__` (클래스 조작) | derive 매크로 `#[derive(..)]` | 클래스 정의 / 파싱 | 동적 속성 주입 / impl 추가 | mypy 불가 / 완전 | 클래스 생성 시 / 0 |
| `eval`/`exec`, 코드 생성 | 함수형 매크로 `name!(..)` | 실행 / 파싱 | 동적 코드 / 생성 코드 | 없음 / 완전 | 런타임 / 0 |
| `*args` 가변 인자 | 선언적 매크로 `macro_rules!` | 실행 / 파싱 | 패킹된 튜플 / 펼쳐진 코드 | 약함 / 완전 | 패킹 비용 / 0 |

핵심 패턴이 보입니다. 파이썬 열은 전부 "실행 시점"에 동작하고 런타임 비용을 동반하며 타입 검사가 부분적이거나 불가능합니다. Rust 열은 전부 "파싱 시점"에 동작하고 런타임 비용이 0이며 산출물이 완전한 타입 검사를 받습니다.

---

## 19.3 선언적 매크로: `macro_rules!`의 해부

이름 뒤에 `!`가 붙은 매크로 — `vec!`, `println!` — 의 정체부터 밝힙시다. 왜 이들은 함수가 아닐까요?

파이썬의 `print()`는 인자를 몇 개든 받습니다(`print(a, b, c)`). 가변 인자(`*args`)가 자유롭기 때문입니다. 그러나 Rust 함수는 **가변 인자를 허용하지 않습니다.** 함수의 인자 개수와 타입이 고정되어야 컴파일러가 스택 프레임 크기와 호출 규약(calling convention)을 확정할 수 있기 때문입니다. 그래서 "임의 개수의 인자를 받아 적절한 코드로 펼치는" 일은 함수가 아니라 매크로의 몫입니다.

`vec!`의 단순화된 내부 구현으로 매크로의 해부학을 익혀봅시다.

```rust
macro_rules! my_vec {
    // ┌─ 매처(Matcher): 입력이 이 모양인지 검사
    // │                          ┌─ 트랜스크라이버(Transcriber): 이 코드로 치환
    ( $( $x:expr ),* $(,)? ) => {{
        let mut temp_vec = Vec::new();
        $(
            temp_vec.push($x);   // 입력 원소 개수만큼 이 줄이 반복 생성된다
        )*
        temp_vec
    }};
}

fn main() {
    let v: Vec<i32> = my_vec![1, 2, 3];
    println!("{:?}", v);  // [1, 2, 3]
}
```

이 작은 매크로 안에 선언적 매크로의 모든 핵심 문법이 들어 있습니다. 하나씩 뜯어봅시다.

### 매처와 트랜스크라이버

`macro_rules!`의 규칙은 `( 매처 ) => { 트랜스크라이버 }` 형태입니다. 매처는 입력 토큰이 어떤 모양이어야 하는지 기술하고, 트랜스크라이버는 그 모양일 때 생성할 코드를 기술합니다. 여러 규칙을 `;`로 나열하면, 컴파일러는 위에서부터 차례로 매칭을 시도해 처음 맞는 규칙을 씁니다(패턴 매칭의 `match`와 같은 first-match 원칙).

### 프래그먼트 지정자 (Fragment Specifiers)

`$x:expr`의 `:expr` 부분이 **프래그먼트 지정자**입니다. "이 자리에 올 수 있는 토큰의 문법적 종류"를 못 박습니다. 매크로가 의미가 아니라 구문을 다룬다는 사실이 여기서 드러납니다 — 매크로는 `$x`가 `i32`인지 모르고, 그저 "표현식 모양의 토큰 덩어리"라는 것만 압니다.

| 지정자 | 매칭 대상 | 예 |
|---|---|---|
| `expr` | 표현식 | `1 + 2`, `foo()`, `vec![1]` |
| `ty` | 타입 | `i32`, `Vec<String>`, `&str` |
| `ident` | 식별자 | `foo`, `MyStruct`, `x` |
| `pat` | 패턴 | `Some(x)`, `_`, `0..=9` |
| `path` | 경로 | `std::collections::HashMap` |
| `tt` | 토큰 트리 (가장 유연) | 임의의 단일 토큰 또는 `(...)`/`[...]`/`{...}` 묶음 |
| `literal` | 리터럴 | `42`, `"hi"`, `3.14` |
| `block` | 블록 | `{ let x = 1; x }` |
| `stmt` | 문장 | `let x = 1` |
| `item` | 아이템 | `fn f() {}`, `struct S;` |

`tt`(token tree)는 가장 원시적이면서 강력한 지정자로, 뒤에서 "tt muncher" 기법에 쓰입니다.

### 반복 (Repetition)

`$( ... )*`가 **반복**입니다. 파이썬의 `*args`를 컴파일 타임으로 옮긴 것입니다.

- `$( $x:expr ),*` — `$x` 표현식이 쉼표(`,`)로 구분되어 **0개 이상** 반복. 매처에 쓰면 "이 패턴을 반복해서 매칭"하고, 트랜스크라이버에 쓰면 "이 코드를 매칭된 횟수만큼 반복 생성".
- `*`는 0회 이상, `+`는 1회 이상, `?`는 0회 또는 1회.
- 구분자(separator)는 `*`/`+` 바로 앞에 둡니다. `),*`는 쉼표 구분, `);*`는 세미콜론 구분.
- `$(,)?`는 **선택적 후행 쉼표** 관용구입니다. `my_vec![1, 2, 3,]`처럼 마지막에 쉼표가 와도 받아주려는 장치입니다(이게 없으면 어떻게 되는지는 19.7에서 컴파일 에러로 확인합니다).

트랜스크라이버의 `$( temp_vec.push($x); )*`는 매처의 반복과 짝을 이룹니다. 매처가 `1, 2, 3`을 세 번 매칭했으면, 이 줄도 세 번 생성됩니다. 그래서 `my_vec![1, 2, 3]`은 다음으로 펼쳐집니다.

```rust
{
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```

런타임에는 그저 `Vec::new()` 한 번과 `push` 세 번이 있을 뿐입니다. 데코레이터처럼 클로저를 만드는 비용은 어디에도 없습니다.

### 여러 규칙과 키-값 매크로

매처는 임의의 구분 토큰을 쓸 수 있습니다. `=>`를 구분자로 쓰면 파이썬의 `dict` 리터럴 같은 `HashMap` 생성 매크로를 만들 수 있습니다.

```rust
macro_rules! hashmap {
    ( $( $key:expr => $val:expr ),* $(,)? ) => {{
        let mut map = std::collections::HashMap::new();
        $(
            map.insert($key, $val);
        )*
        map
    }};
}

fn main() {
    // 파이썬: {"one": 1, "two": 2}
    let m = hashmap!{ "one" => 1, "two" => 2 };
    println!("{:?}", m.get("one"));  // Some(1)
}
```

`$key:expr => $val:expr`라는 패턴이 `=>`를 사이에 둔 키-값 쌍을 매칭합니다. 이것만으로 파이썬 딕셔너리 리터럴에 근접한 문법을 얻습니다. 실무에서는 `maplit` 크레이트가 바로 이런 `hashmap!`/`hashset!` 매크로를 제공합니다.

### 재귀 매크로: 컴파일 타임 반복

매크로는 자기 자신을 호출할 수 있습니다. 가변 인자 중 최댓값을 구하는 `my_max!`를 재귀로 만들어봅시다.

```rust
macro_rules! my_max {
    // 기저 사례(base case): 인자가 하나면 그대로 반환
    ($x:expr) => { $x };
    // 재귀 사례: 첫 원소와 "나머지의 최댓값"을 비교
    ($x:expr, $($rest:expr),+) => {{
        let rest_max = my_max!($($rest),+);
        if $x > rest_max { $x } else { rest_max }
    }};
}

fn main() {
    let big = my_max!(3, 7, 2, 9, 1);
    println!("max = {}", big);  // max = 9
}
```

`my_max!(3, 7, 2, 9, 1)`은 컴파일 타임에 `3`과 `my_max!(7, 2, 9, 1)`의 비교로 펼쳐지고, 그 안쪽이 다시 펼쳐지면서 결국 중첩된 `if` 식이 됩니다. 런타임에는 단순한 비교 연산 몇 번뿐입니다. 이 재귀 패턴의 극단적 형태가 **tt muncher**(토큰 트리 갉아먹기)입니다. `tt` 지정자로 입력을 한 토큰씩 떼어내며 처리합니다.

```rust
macro_rules! count_tts {
    () => { 0usize };
    ($head:tt $($rest:tt)*) => { 1usize + count_tts!($($rest)*) };
}

fn main() {
    const N: usize = count_tts!(a b c d e);  // 컴파일 타임 상수!
    println!("count = {}", N);  // count = 5
}
```

`count_tts!(a b c d e)`는 `1 + count_tts!(b c d e)` → ... → `1 + 1 + 1 + 1 + 1 + 0`으로 펼쳐지고, 컴파일러의 상수 폴딩(constant folding)이 이를 `5`로 접어버립니다. `const N`에 쓸 수 있다는 사실이 핵심입니다 — 이 계산은 **컴파일 타임에 끝납니다.** 파이썬이라면 `len([a, b, c, d, e])`를 매 실행마다 셌겠지만, 여기서는 바이너리에 이미 `5`가 박혀 있습니다.

---

## 19.4 위생(Hygiene): 매크로가 변수를 더럽히지 않는 이유

C 매크로를 써본 사람이라면 다음 같은 함정에 데어봤을 것입니다. C의 `#define`은 순진한 텍스트 치환이라, 매크로 내부에서 쓴 임시 변수 이름이 호출 측 변수와 충돌해 조용히 값을 망가뜨립니다. Rust는 이 문제를 **위생(Hygiene)**이라는 메커니즘으로 원천 봉쇄합니다.

위생이란 "매크로가 도입한 식별자와 호출 측의 식별자가 서로 다른 문맥(syntax context)에 속해, 같은 이름이라도 충돌하지 않는다"는 규칙입니다. 직접 확인해봅시다.

```rust
macro_rules! using_a {
    ($e:expr) => {{
        let a = 42;   // 매크로 내부의 a
        $e + a
    }};
}

fn main() {
    let a = 100;             // 호출 측의 a
    let r = using_a!(a);     // $e 자리에 호출 측 a(=100)가 들어간다
    println!("r = {}", r);   // r = 142
}
```

상식적으로 생각하면 매크로가 펼쳐진 코드는 이렇게 보일 것입니다.

```rust
let a = 100;
let r = {
    let a = 42;    // 어? 위의 a를 가렸나?
    a + a          // 42 + 42 = 84? 아니면 100 + 42 = 142?
};
```

순진한 텍스트 치환이라면 `$e`(호출 측 `a`)가 매크로 내부의 `let a = 42`에 가려져 `84`가 나와야 합니다. 그러나 실제 결과는 **142**입니다. 매크로가 `let a = 42`로 만든 `a`와, 호출 측에서 `$e`로 넘어온 `a`는 **컴파일러 입장에서 완전히 다른 변수**이기 때문입니다. 매크로가 도입한 식별자에는 보이지 않는 "위생 마크"가 찍혀, 호출 측 식별자와 절대 섞이지 않습니다. 그래서 `$e`는 끝까지 호출 측의 `a`(100)를 가리키고, `+ a`의 `a`는 매크로 내부의 `a`(42)를 가리켜 142가 됩니다.

> **Under the Hood: 위생은 토큰에 붙은 "문맥 태그"다**
>
> rustc는 매크로를 확장할 때 각 토큰에 `SyntaxContext`라는 보이지 않는 꼬리표를 답니다. 매크로 본문에서 생성된 토큰에는 매크로의 문맥 태그가, `$e`처럼 호출 측에서 넘어온 토큰에는 호출 지점의 문맥 태그가 붙습니다. 이름 해석(name resolution) 단계에서 두 식별자가 같은 이름이어도 **문맥 태그가 다르면 별개의 심볼**로 취급합니다. 즉 위생은 "이름이 같은가"가 아니라 "이름 + 문맥이 같은가"로 변수를 식별하는 것입니다. C의 `#define`이 못 했던 것을, Rust는 토큰 메타데이터 한 겹으로 해결합니다.
>
> 위생에는 의도적 예외가 있습니다. 타입, 트레이트 메서드, 그리고 매크로 인자로 명시적으로 넘긴 식별자(`$name:ident`)는 호출 측 문맥을 따릅니다. 그래서 `#[derive(Builder)]`가 생성한 `build()` 메서드를 호출 측에서 자연스럽게 부를 수 있는 것입니다. 위생은 "실수로 인한 충돌"은 막되, "의도된 코드 생성"은 허용하도록 정교하게 설계돼 있습니다.

`using_a!`가 142를 출력한다는 사실은 직접 컴파일해 확인한 것입니다. 위생 덕분에 매크로 작성자는 호출 측이 어떤 변수 이름을 쓰는지 신경 쓸 필요가 없고, 매크로 사용자는 매크로 내부의 임시 변수가 자기 변수를 망가뜨릴까 걱정할 필요가 없습니다. 데코레이터의 클로저가 변수 캡처로 비슷한 격리를 제공하지만, 그것은 런타임 스코프 규칙의 결과인 반면 매크로의 위생은 컴파일러가 토큰 수준에서 강제하는 정적 보장입니다.

---

## 19.5 절차적 매크로: 데코레이터의 진짜 대체재

선언적 매크로는 강력하지만 한계가 있습니다. 입력 토큰을 **패턴으로만** 다룰 수 있어서, "구조체의 모든 필드를 순회하며 코드를 생성"하는 것 같은 복잡한 분석은 불가능합니다. 파이썬의 메타클래스가 클래스 네임스페이스를 자유롭게 헤집었던 것 같은 일을 하려면, 입력을 **진짜 데이터 구조로 파싱해 임의의 코드로 처리**할 수 있어야 합니다. 그것이 **절차적 매크로(Procedural Macro)**입니다.

절차적 매크로는 한 문장으로 정의하면 **"컴파일 타임에 실행되는 별도의 Rust 프로그램"**입니다. 입력으로 `TokenStream`을 받아, 분석·변형한 뒤, 다시 `TokenStream`을 출력합니다. 그 사이의 처리는 무엇이든 가능합니다 — 파일을 읽든, 정규식을 돌리든, SQL을 파싱하든(`sqlx::query!`가 실제로 컴파일 타임에 DB 스키마를 검증하는 방식입니다).

절차적 매크로는 세 종류입니다.

1. **derive 매크로**: `#[derive(Debug)]`처럼 `derive` 속성에 붙어 구조체/열거형에 트레이트 구현을 **추가**합니다. 원본은 건드리지 않고 코드를 덧붙입니다.
2. **속성 매크로(Attribute-like)**: `#[tokio::main]`, `#[route("/")]`처럼 아이템에 붙어 그 아이템을 **재작성**합니다. 파이썬 데코레이터와 시각적으로 가장 닮았습니다.
3. **함수형 매크로(Function-like)**: `sql!(...)`처럼 `name!(...)` 형태로 호출되지만, `macro_rules!`와 달리 입력을 임의의 코드로 처리합니다.

절차적 매크로는 반드시 `proc-macro = true`로 설정된 **별도 크레이트**에 작성해야 합니다. 컴파일러가 본 크레이트를 컴파일하기 전에 이 매크로 크레이트를 먼저 빌드해 실행해야 하기 때문입니다(닭이 먼저냐 달걀이 먼저냐 문제를 크레이트 분리로 푸는 것입니다). 생태계의 세 기둥 크레이트를 쓰게 됩니다.

- **`proc-macro2`**: 표준 `proc_macro::TokenStream`을 감싼, 테스트 가능하고 안정적인 토큰 스트림 타입.
- **`syn`**: `TokenStream`을 Rust 문법 트리(AST)로 **파싱**합니다. `ItemFn`(함수), `DeriveInput`(구조체/열거형) 같은 타입을 제공합니다.
- **`quote`**: 그 반대 방향. Rust 코드를 따옴표 안에 쓰듯 작성해 `TokenStream`으로 **조립**합니다. `#변수` 보간(interpolation)과 `#(...)* ` 반복을 지원합니다.

### 실전: `#[timer]` 속성 매크로 직접 만들기

19.1에서 본 파이썬 `@timer` 데코레이터를, 런타임 오버헤드 없는 Rust 속성 매크로로 다시 구현해봅시다. 먼저 매크로 크레이트의 `Cargo.toml`입니다.

```toml
# timer_macro/Cargo.toml
[package]
name = "timer_macro"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true   # 이 설정이 핵심: 이 크레이트는 컴파일러 플러그인이다

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

매크로 본체입니다.

```rust
// timer_macro/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn timer(_attr: TokenStream, item: TokenStream) -> TokenStream {
    // ① 입력 토큰을 "함수" AST로 파싱한다 (syn)
    let input = parse_macro_input!(item as ItemFn);

    // ② 함수의 구성 요소를 분해한다
    let vis = &input.vis;      // pub 등 가시성
    let sig = &input.sig;      // 시그니처 (이름, 인자, 반환 타입)
    let block = &input.block;  // 함수 본문
    let fn_name = sig.ident.to_string();

    // ③ 새 코드를 조립한다 (quote). 원본 본문을 클로저로 감싸 측정한다.
    let expanded = quote! {
        #vis #sig {
            let __start = std::time::Instant::now();
            let __result = (|| #block)();
            eprintln!("{} took {:?}", #fn_name, __start.elapsed());
            __result
        }
    };

    // ④ TokenStream으로 변환해 반환한다
    expanded.into()
}
```

`#vis`, `#sig`, `#block`, `#fn_name`은 `quote!`의 변수 보간입니다. `syn`이 파싱한 조각들을 새 코드 틀에 꽂아 넣습니다. 사용하는 쪽은 파이썬과 거의 똑같습니다.

```rust
// app/src/main.rs
use timer_macro::timer;

#[timer]   // 파이썬의 @timer 와 동일한 모양
fn heavy_computation() -> i64 {
    (0..1_000_000i64).map(|i| i * i).sum()
}

fn main() {
    let s = heavy_computation();
    println!("sum = {}", s);
}
```

이 코드를 실제로 빌드해 실행하면 다음이 출력됩니다(직접 확인했습니다).

```text
heavy_computation took 10.083334ms
sum = 333332833333500000
```

`#[timer]`는 컴파일 타임에 `heavy_computation`을 다음과 같이 **재작성**합니다.

```rust
// #[timer]가 펼쳐낸 코드 (cargo expand로 확인 가능)
fn heavy_computation() -> i64 {
    let __start = std::time::Instant::now();
    let __result = (|| { (0..1_000_000i64).map(|i| i * i).sum() })();
    eprintln!("heavy_computation took {:?}", __start.elapsed());
    __result
}
```

파이썬 데코레이터와의 차이를 음미해봅시다. 파이썬은 `heavy_computation`이라는 이름을 **별개의 `wrapper` 함수 객체**에 다시 묶었습니다. 호출 시 `wrapper` → `func` 두 프레임을 거칩니다. Rust는 `heavy_computation`이라는 **함수 자체의 본문을 다시 썼습니다.** 별도 함수도, 추가 프레임도 없습니다. 타이밍 코드가 함수 안에 박혀 있을 뿐입니다. 게다가 반환 타입 `i64`가 그대로 보존되어, 호출 측에서 타입 추론과 자동완성이 완벽하게 작동합니다. `functools.wraps`를 잊어 `__name__`이 깨질 일도 없습니다 — 함수 이름은 애초에 바뀌지 않았으니까요.

이 차이는 측정 가능합니다. 아무 일도 안 하는 패스스루(passthrough) 데코레이터, 즉 인자를 그대로 원본에 넘기기만 하는 `functools.wraps` 래퍼의 순수 오버헤드를 재봤습니다(1천만 회 호출, 워밍업 후, CPython 3.x).

```text
plain:     0.515s  ( 51.5 ns/call)   # 데코레이터 없는 함수
decorated: 1.758s  (175.8 ns/call)   # 패스스루 데코레이터 한 겹
호출당 추가 비용: 124.3 ns  (x3.41 느려짐)
```

함수 본문이 `x + 1` 한 줄뿐인 극단적 경우라, 래핑 오버헤드가 상대적으로 도드라져 호출당 3.4배까지 벌어졌습니다. 본문이 무거우면 비율은 줄지만, **호출당 절대 비용 124 ns는 사라지지 않고**, 호출이 잦은 핫 패스에서는 누적됩니다(초당 100만 회 호출이면 데코레이터만으로 0.12초). 데코레이터를 두 겹 쌓으면 이 비용이 두 배가 됩니다. Rust의 `#[timer]`는 이 오버헤드가 정확히 **0**입니다 — 추가 함수 객체도, 추가 스택 프레임도, 인자 재포장도 컴파일된 바이너리에 존재하지 않기 때문입니다. 코드가 함수 본문에 인라인됐을 뿐이고, 옵티마이저는 그것을 손으로 쓴 코드와 동일하게 취급합니다. 이것이 "무비용 추상화(Zero-cost Abstraction)"의 문자 그대로의 의미입니다: 추상화를 위해 지불하는 런타임 비용이 영(零)입니다.

### derive 매크로: 구조체를 분석해 메서드 생성

이번엔 메타클래스에 가장 가까운 일을 해봅시다. 메타클래스가 클래스 필드를 수집해 `_fields`를 만들었듯, 구조체의 모든 필드를 순회해 `HashMap<String, String>`으로 변환하는 메서드를 생성하는 `#[derive(IntoMap)]`을 만듭니다.

```rust
// builder_derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(IntoMap)]
pub fn into_map(input: TokenStream) -> TokenStream {
    // 구조체/열거형 정의를 DeriveInput으로 파싱
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    // 명명된 필드(named fields)만 꺼낸다 — 메타클래스의 namespace 스캔에 해당
    let fields = match &input.data {
        Data::Struct(s) => match &s.fields {
            Fields::Named(named) => &named.named,
            _ => panic!("IntoMap only supports named fields"),
        },
        _ => panic!("IntoMap only supports structs"),
    };

    // 필드마다 map.insert("필드명", 값) 코드를 생성
    let inserts = fields.iter().map(|f| {
        let ident = f.ident.as_ref().unwrap();
        let key = ident.to_string();
        quote! {
            map.insert(#key.to_string(), format!("{:?}", self.#ident));
        }
    });

    // impl 블록 전체를 조립. #(#inserts)* 가 위 코드들을 펼쳐 넣는다
    let expanded = quote! {
        impl #name {
            fn into_map(&self) -> std::collections::HashMap<String, String> {
                let mut map = std::collections::HashMap::new();
                #(#inserts)*
                map
            }
        }
    };
    expanded.into()
}
```

사용 측:

```rust
use builder_derive::IntoMap;

#[derive(IntoMap)]
struct User {
    id: u64,
    name: String,
}

fn main() {
    let u = User { id: 7, name: "Ada".to_string() };
    let mut entries: Vec<_> = u.into_map().into_iter().collect();
    entries.sort();
    println!("{:?}", entries);
    // [("id", "7"), ("name", "\"Ada\"")]
}
```

`#(#inserts)*`는 `quote!`의 반복 보간으로, `inserts` 이터레이터의 각 항목을 차례로 펼쳐 넣습니다 — `macro_rules!`의 `$( ... )*`와 같은 역할을 절차적 매크로 세계에서 합니다. 이것이 `#[derive(Debug)]`, `#[derive(Serialize)]`가 동작하는 정확한 원리입니다. `serde_derive`는 이 derive 메커니즘으로 구조체 필드를 순회해 직렬화/역직렬화 코드를 통째로 생성합니다. Chapter 7에서 `#[derive(Serialize)]` 한 줄로 Pydantic을 대체했던 그 마법의 정체가 바로 이것입니다.

### 함수형 매크로: `name!(...)`의 절차적 버전

세 번째 종류인 **함수형 매크로(Function-like)**는 호출 형태가 `macro_rules!`와 똑같이 `name!(...)`이지만, 입력을 패턴이 아니라 임의의 코드로 처리한다는 점이 다릅니다. 시그니처는 derive와 속성 매크로의 중간 형태입니다.

```rust
// 시그니처만 발췌
use proc_macro::TokenStream;

#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    // input: 괄호 안에 들어온 토큰 전부 (예: SELECT * FROM users)
    // 여기서 SQL을 파싱하고, 컴파일 타임에 문법을 검증하고,
    // 검증된 쿼리를 실행하는 Rust 코드를 생성해 반환한다.
    // ...
    todo!()
}
```

`sqlx::query!` 매크로가 이 부류의 백미입니다. `sqlx::query!("SELECT id, name FROM users WHERE id = ?", user_id)`라고 쓰면, 매크로가 **컴파일 타임에 실제 데이터베이스에 접속해** 쿼리의 문법과 컬럼 타입을 검증하고, 결과를 담을 구조체까지 자동 생성합니다. 잘못된 컬럼명이나 타입 불일치가 **빌드 단계에서** 에러로 잡힙니다. 파이썬의 ORM은 이 검증을 잘해야 런타임 첫 쿼리 시점에, 못하면 프로덕션에서 합니다. 함수형 매크로는 "컴파일 타임에 외부 세계(DB 스키마, 파일, 환경)를 들여다보고 코드를 생성"하는 가장 극적인 사례입니다. 메타클래스로도 도달할 수 없는 영역입니다 — 메타클래스는 어차피 런타임에 도는 코드이니까요.

세 종류를 한눈에 정리하면 이렇습니다.

| 절차적 매크로 종류 | 함수 속성 | 입력 | 원본 처리 | 파이썬 유사물 |
|---|---|---|---|---|
| derive | `#[proc_macro_derive(Name)]` | `DeriveInput` | 보존 + impl 추가 | (직접 대응 없음) |
| 속성(attribute) | `#[proc_macro_attribute]` | 속성 인자 + 아이템 | 소비 후 대체 | `@decorator` |
| 함수형(function-like) | `#[proc_macro]` | 임의 토큰 | 입력을 코드로 변환 | (직접 대응 없음) |

> **Under the Hood: derive 매크로는 원본을 "추가"하고, 속성 매크로는 "대체"한다**
>
> 두 절차적 매크로의 결정적 차이는 원본 코드의 운명입니다. `#[derive(IntoMap)]`은 `User` 구조체 정의를 **그대로 두고**, 생성한 `impl User { ... }` 블록을 옆에 **덧붙입니다**. 그래서 한 구조체에 `#[derive(Debug, Clone, Serialize, IntoMap)]`처럼 여러 derive를 쌓아도 서로 충돌하지 않습니다 — 각자 별도의 `impl` 블록을 추가할 뿐이니까요.
>
> 반면 `#[timer]` 같은 속성 매크로는 입력 아이템을 **소비하고 그 자리를 출력으로 대체**합니다. `timer` 함수가 받은 `item: TokenStream`이 원본 함수이고, 반환한 `TokenStream`이 그것을 완전히 대신합니다. 원본을 보존하려면 매크로 안에서 명시적으로 다시 포함시켜야 합니다(우리 예제에서는 `#sig`와 `#block`을 새 본문에 다시 넣었습니다). 이 "대체" 모델이 파이썬 데코레이터(`name = decorator(name)`로 이름을 재바인딩)와 의미적으로 정확히 일치합니다. derive의 "추가" 모델은 파이썬에 직접적 대응물이 없는, Rust 특유의 안전한 확장 방식입니다.

---

## 19.6 `cargo expand`: 매크로의 X-레이

매크로의 가장 큰 단점은 "내가 쓴 코드가 실제로 무엇으로 변하는지 보이지 않는다"는 점입니다. 메타클래스가 `_fields`를 숨겼듯, 매크로도 생성 코드를 숨깁니다. 하지만 메타클래스와 달리, 매크로에는 **X-레이 도구**가 있습니다. `cargo expand`입니다.

```bash
cargo install cargo-expand   # 최초 1회 설치
cargo expand                 # 모든 매크로를 펼친 소스를 출력
cargo expand path::to::item  # 특정 아이템만
```

`#[derive(Debug)]`를 붙인 간단한 구조체를 `cargo expand`로 펼치면, 컴파일러가 자동 생성한 `impl Debug` 블록이 실제 Rust 코드로 드러납니다.

```rust
// 원본
#[derive(Debug)]
struct Point { x: i32, y: i32 }
```

```rust
// cargo expand 결과 (발췌)
struct Point { x: i32, y: i32 }
impl ::core::fmt::Debug for Point {
    fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        ::core::fmt::Formatter::debug_struct_field2_finish(
            f, "Point", "x", &self.x, "y", &&self.y,
        )
    }
}
```

`#[derive(Debug)]` 한 줄이 손으로 쓰면 십수 줄짜리 `impl Debug`로 펼쳐진다는 것을 두 눈으로 확인할 수 있습니다. 매크로 디버깅의 첫걸음은 언제나 `cargo expand`입니다. "매크로가 왜 컴파일이 안 되지?"의 90%는 펼친 결과를 보면 즉시 풀립니다. 파이썬에서 메타클래스가 만든 속성을 알아내려면 `dir()`과 디버거로 런타임을 헤집어야 했던 것과 비교하면, 컴파일 타임 코드 생성의 투명성이 얼마나 큰 이점인지 드러납니다.

---

## 19.7 컴파일러 에러 시연: 매처 불일치와 위생

매크로는 강력하지만, 잘못 쓰면 평범한 코드보다 에러 메시지가 난해해질 수 있습니다. 컴파일러가 실제로 어떤 에러를 내뱉는지 두 가지 실패를 직접 일으켜 봅시다. 둘 다 실제 `rustc` 출력입니다.

### 함정 1: 후행 쉼표를 받지 못하는 매처

19.3의 `my_vec!`에서 선택적 후행 쉼표 장치 `$(,)?`를 일부러 빼보겠습니다.

```rust
macro_rules! my_vec {
    ( $( $x:expr ),* ) => {{          // $(,)? 가 없다
        let mut temp_vec = Vec::new();
        $( temp_vec.push($x); )*
        temp_vec
    }};
}

fn main() {
    // 🚨 컴파일 에러! 마지막 원소 뒤의 쉼표를 매처가 받지 못한다
    let v: Vec<i32> = my_vec![1, 2, 3,];
}
```

`rustc`의 출력:

```text
error: unexpected end of macro invocation
  --> src/main.rs:11:39
   |
 1 | macro_rules! my_vec {
   | ------------------- when calling this macro
...
11 |     let v: Vec<i32> = my_vec![1, 2, 3,];
   |                                       ^ missing tokens in macro arguments
   |
note: while trying to match meta-variable `$x:expr`
  --> src/main.rs:2:10
   |
 2 |     ( $( $x:expr ),* ) => {{
   |          ^^^^^^^
```

해독: 매처는 `1, 2, 3,`을 보고 마지막 쉼표 뒤에 **또 다른 `$x:expr`이 오리라** 기대했습니다(`),*` 패턴상 쉼표 다음엔 표현식이 와야 하니까요). 그런데 표현식 없이 `]`로 끝나버려 "토큰이 부족하다(missing tokens)"고 항의합니다. `note`가 친절하게 "`$x:expr`을 매칭하려던 중이었다"고 알려줍니다. 해결책은 매처에 `$(,)?`를 넣어 후행 쉼표를 명시적으로 허용하는 것입니다. 표준 라이브러리의 `vec!`이 후행 쉼표를 받아주는 이유가 바로 이 장치 덕분입니다.

### 함정 2: 위생이 막는 변수 누출

이번엔 위생 규칙을 정면으로 위반해봅시다. 매크로가 내부에서 변수를 선언하고, 호출 측이 그 변수를 보길 기대하는 코드입니다.

```rust
macro_rules! declare_x {
    () => { let x = 5; };
}

fn main() {
    declare_x!();
    // 🚨 컴파일 에러! 매크로가 만든 x는 위생 때문에 여기서 안 보인다
    println!("{}", x);
}
```

`rustc`의 출력:

```text
error[E0425]: cannot find value `x` in this scope
 --> src/main.rs:7:20
  |
7 |     println!("{}", x);
  |                    ^ not found in this scope
  |
help: an identifier with the same name is defined here, but is not
      accessible due to macro hygiene
 --> src/main.rs:3:17
  |
3 |     () => { let x = 5; };
  |                 ^
```

이것이 위생의 실증입니다. `declare_x!()`는 분명 `let x = 5;`를 만들었지만, 그 `x`는 매크로의 문맥에 속해 호출 측 `main`의 스코프에서는 보이지 않습니다. 컴파일러의 `help`가 정곡을 찌릅니다 — "같은 이름의 식별자가 여기 정의돼 있지만, **매크로 위생 때문에 접근할 수 없다**(not accessible due to macro hygiene)." 만약 위생이 없는 C 매크로였다면 `x`가 그대로 누출되어 `5`가 출력됐을 것이고, 운 나쁘면 호출 측의 다른 `x`를 덮어써 버그를 심었을 것입니다. Rust는 이것을 **컴파일 에러로** 잡아냅니다.

이 에러를 의도적으로 우회하고 싶다면(즉 진짜로 호출 측에 변수를 만들고 싶다면), 변수 이름을 `$name:ident`로 매크로 인자를 통해 받아야 합니다. 그러면 그 식별자는 호출 측 문맥을 따라, 호출자가 명시적으로 지정한 이름으로 만들어집니다. 위생은 "실수로 인한 누출"은 막되 "의도적인 식별자 전달"은 허용한다는 19.4의 원칙이 여기서 실전으로 확인됩니다.

---

## 19.8 실전 Hands-on — `dbg!` 스타일 디버그 매크로 만들기

배운 것을 엮어, 실무에서 바로 쓸 법한 디버그 매크로를 직접 만들어봅시다. 파이썬 개발자가 디버깅할 때 흔히 쓰는 패턴은 이것입니다.

```python
x = compute()
print(f"x = {x}")   # 변수 이름과 값을 함께, 매번 손으로
```

변수 이름을 문자열로 다시 타이핑하는 게 번거롭습니다. Rust 표준 라이브러리에는 이를 해결하는 `dbg!` 매크로가 있는데, 그 단순화 버전을 직접 구현하면서 `stringify!`, `file!`, `line!` 같은 내장 매크로의 활용을 익혀봅시다.

```rust
/// 표현식을 평가해 "파일:줄 표현식 = 값"을 stderr에 찍고, 값을 그대로 반환한다.
macro_rules! debug_print {
    ($val:expr) => {{
        // stringify! : 토큰을 그대로 문자열 리터럴로 (컴파일 타임)
        // file!/line! : 현재 위치 (컴파일 타임 상수)
        let tmp = $val;  // 한 번만 평가 (부작용 있는 식의 중복 평가 방지)
        eprintln!("[{}:{}] {} = {:?}", file!(), line!(), stringify!($val), &tmp);
        tmp
    }};
}

fn main() {
    let x = 21 * 2;
    debug_print!(x);              // 변수 출력
    let y = debug_print!(x + 1);  // 값을 그대로 반환하므로 체이닝 가능
    println!("y = {}", y);
}
```

실행 결과(직접 확인했습니다):

```text
[src/main.rs:15] x = 42
[src/main.rs:16] x + 1 = 43
y = 43
```

이 작은 매크로 안에 챕터의 핵심이 모두 들어 있습니다.

- **`stringify!($val)`**: `$val` 토큰을 **컴파일 타임에** 문자열로 박제합니다. `debug_print!(x + 1)`은 `"x + 1"`이라는 문자열 리터럴이 됩니다. 파이썬에서 변수 이름을 손으로 다시 적던 일을, 매크로가 토큰을 보고 자동으로 합니다.
- **`file!()`, `line!()`**: 호출 위치를 컴파일 타임 상수로 박습니다. 런타임에 스택을 들여다보는 비용이 0입니다(파이썬의 `inspect`나 `traceback`은 런타임 비용이 큽니다).
- **`let tmp = $val;`**: `$val`을 단 한 번만 평가합니다. 만약 `eprintln!`에서 `$val`을 두 번 썼다면, `debug_print!(expensive_call())`이 함수를 두 번 호출하는 버그가 생겼을 것입니다. 매크로 작성의 핵심 규율 — **인자를 정확히 한 번만 평가하라** — 가 여기 적용됐습니다.
- **값을 그대로 반환**: 블록의 마지막 표현식이 `tmp`이므로, `debug_print!`는 감싼 값을 그대로 돌려줍니다. 덕분에 `let y = debug_print!(x + 1)`처럼 기존 코드 한가운데 끼워 넣어도 동작이 바뀌지 않습니다. 표준 `dbg!`가 바로 이 설계라, 디버깅 후 매크로만 지우면 끝입니다.

전체 코드는 `fn main`을 포함해 그대로 `cargo run` 됩니다. 의존성도 필요 없습니다 — 모두 표준 라이브러리의 내장 매크로(`eprintln!`, `stringify!`, `file!`, `line!`)와 우리가 만든 `macro_rules!`만으로 동작합니다. 이것이 선언적 매크로의 위력입니다. 별도 크레이트도, `syn`/`quote`도 없이 강력한 디버깅 도구를 같은 파일 안에서 만들었습니다.

---

## 19.9 매크로 검증 — `trybuild`와 compile-fail 테스트

매크로는 테스트 방식도 일반 함수와 다릅니다. 함수는 "이 입력을 넣으면 이 값이 나온다"를 테스트하면 됩니다. 매크로는 거기에 더해 **잘못 썼을 때 좋은 컴파일 에러가 나는가**를 테스트해야 합니다. 파이썬 데코레이터는 런타임에 실패하지만, Rust 매크로의 사용자 경험은 컴파일 에러 메시지 품질에 달려 있습니다.

절차적 매크로 크레이트에서 표준처럼 쓰이는 도구가 `trybuild`입니다. `tests/ui/*.rs`에 "컴파일되어야 하는 예제"와 "컴파일에 실패해야 하는 예제"를 두고, stderr 스냅샷까지 검증합니다.

```toml
[dev-dependencies]
trybuild = "1"
```

```rust
// tests/trybuild.rs
#[test]
fn ui_tests() {
    let t = trybuild::TestCases::new();
    t.pass("tests/ui/pass_timer.rs");
    t.compile_fail("tests/ui/fail_timer_on_struct.rs");
}
```

성공해야 하는 예제는 실제 사용자가 README에서 복사할 법한 최소 코드입니다.

```rust
// tests/ui/pass_timer.rs
use my_macros::timer;

#[timer]
fn work() -> u32 {
    42
}

fn main() {
    assert_eq!(work(), 42);
}
```

실패해야 하는 예제는 매크로의 계약을 일부러 어깁니다.

```rust
// tests/ui/fail_timer_on_struct.rs
use my_macros::timer;

#[timer]
struct NotAFunction;

fn main() {}
```

좋은 속성 매크로라면 이때 `syn`의 내부 파싱 에러를 그대로 흘리지 않고, 호출자에게 명확한 메시지를 줘야 합니다.

```text
error: #[timer] can only be applied to functions
 --> tests/ui/fail_timer_on_struct.rs:3:1
  |
3 | #[timer]
  | ^^^^^^^^
```

이런 메시지를 만들려면 절차적 매크로 내부에서 `syn::Error`를 `compile_error!` 토큰으로 바꿔 반환합니다.

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Item, ItemFn};

#[proc_macro_attribute]
pub fn timer(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let parsed = parse_macro_input!(item as Item);

    let function = match parsed {
        Item::Fn(function) => function,
        other => {
            return syn::Error::new_spanned(
                other,
                "#[timer] can only be applied to functions",
            )
            .to_compile_error()
            .into();
        }
    };

    expand_timer(function).into()
}

fn expand_timer(function: ItemFn) -> proc_macro2::TokenStream {
    let name = &function.sig.ident;
    let vis = &function.vis;
    let sig = &function.sig;
    let block = &function.block;

    quote! {
        #vis #sig {
            let __start = ::std::time::Instant::now();
            let __result = (|| #block)();
            eprintln!("{} took {:?}", stringify!(#name), __start.elapsed());
            __result
        }
    }
}
```

여기서 `new_spanned(other, "...")`가 중요합니다. 에러의 위치(span)를 사용자가 붙인 잘못된 아이템 전체에 걸어 주므로, 컴파일러가 정확한 위치를 밑줄로 표시합니다. 매크로 작성자는 단지 코드를 생성하는 사람이 아니라, **컴파일러 에러 UX를 설계하는 사람**입니다.

> **💡 Under the Hood: `compile_error!`는 실패를 토큰으로 표현한다**
>
> 절차적 매크로는 성공하면 Rust 코드를 토큰으로 반환합니다. 실패도 같은 방식입니다. `syn::Error::to_compile_error()`는 내부적으로 `compile_error!("...")` 매크로 호출을 만들어 반환합니다. 그러면 rustc는 매크로 확장 결과를 다시 파싱하다가 `compile_error!`를 만나 의도된 컴파일 에러를 출력합니다. 즉 "매크로가 실패했다"는 예외나 패닉이 아니라, **실패를 나타내는 Rust 코드 조각**입니다. 이 구조 덕분에 실패 위치(span)와 메시지를 사용자가 읽기 좋게 제어할 수 있습니다.

CI에서는 다음 순서를 권합니다.

```bash
$ cargo test
$ cargo expand --test trybuild >/tmp/expanded.rs   # 필요할 때만 수동 확인
```

`trybuild`의 첫 실행은 stderr 스냅샷을 생성하라고 안내합니다. 메시지를 검토한 뒤 승인하면, 이후에는 매크로 에러 메시지가 바뀌는 것도 회귀로 잡힙니다. 라이브러리 사용자에게는 "컴파일 실패도 API"입니다. 깨끗한 매크로 라이브러리는 성공 경로만큼 실패 경로를 테스트합니다.

---

## 19.10 Best Practices & 안티패턴

### 이렇게 하라

- **먼저 함수/제네릭을 시도하라.** 매크로는 함수로 표현할 수 없는 것(가변 인자, 컴파일 타임 코드 생성, 토큰 조작)에만 쓰는 최후의 도구입니다. 같은 일을 제네릭 함수나 트레이트로 할 수 있다면 그쪽이 거의 언제나 낫습니다 — 타입 에러가 명확하고, IDE 지원이 좋고, 읽기 쉽습니다.
- **`$(,)?`로 후행 쉼표를 허용하라.** 표준 매크로처럼 `my_vec![1, 2, 3,]`을 받아주는 것이 사용자 친화적입니다(19.7 함정 1).
- **인자를 정확히 한 번만 평가하라.** 표현식 인자(`$x:expr`)를 트랜스크라이버에서 여러 번 쓰면 부작용이 중복됩니다. `let tmp = $x;`로 한 번 묶어 쓰세요(19.8).
- **선언적으로 충분하면 절차적 매크로를 만들지 마라.** `syn`/`quote` 기반 절차적 매크로는 별도 크레이트, 느린 빌드, 가파른 학습 곡선을 동반합니다. `macro_rules!`로 풀리는 일을 굳이 절차적으로 만들지 마세요.
- **매크로를 문서화하고 `cargo expand`로 검증하라.** 매크로는 사용법이 직관적이지 않을 수 있습니다. 예제를 doc 주석에 넣고, 펼친 결과를 한 번은 확인하세요.
- **공개 매크로는 경로를 절대화하라.** 라이브러리 매크로가 `Vec::new()` 대신 `::std::vec::Vec::new()`처럼 절대 경로(`$crate::` 포함)를 쓰면, 호출 측의 임포트 상황과 무관하게 동작합니다.

### 이건 피하라

- **🚨 매크로 남용.** "이것도 매크로로 하면 멋지겠다"는 유혹을 경계하세요. 매크로로 도배된 코드베이스는 새 팀원에게 외계어처럼 보이고, IDE의 "정의로 이동(Go to Definition)"이 매크로 본문으로 점프해 흐름을 끊습니다. 매크로는 보일러플레이트를 줄이지만, 과하면 가독성이라는 더 큰 자산을 태웁니다.
- **🚨 거대한 절차적 매크로에 비즈니스 로직을 욱여넣기.** 절차적 매크로는 디버깅이 어렵습니다(런타임 디버거를 못 붙입니다). 핵심 로직은 일반 함수에 두고, 매크로는 얇은 어댑터로 유지하세요.
- **🚨 매처를 너무 영리하게.** 수십 개의 규칙과 깊은 tt muncher 재귀로 짠 매크로는, 6개월 뒤의 자신조차 못 읽습니다. 매크로 재귀가 깊어지면 컴파일러의 재귀 한계(`recursion_limit`)에 부딪히고 컴파일 시간도 폭증합니다.
- **clippy를 무시하지 마라.** `clippy`는 `crate_in_macro_def`, `single_component_path_imports` 등 매크로 관련 린트를 제공합니다. 경고를 흘려보내지 마세요.

---

## 19.11 흔한 함정 (파이썬 개발자가 자주 하는 실수)

**함정 1 — 매크로를 함수처럼 생각한다.**

`my_macro!(some_expr())`에서 `some_expr()`이 한 번 호출되리라 가정하지만, 매크로가 `$x`를 본문에서 두 번 쓰면 두 번 호출됩니다. 매크로는 값을 받는 게 아니라 **토큰을 받아 펼친다**는 점을 잊지 마세요. 파이썬 함수는 인자를 평가해 값으로 받지만, 매크로는 평가 전의 코드 조각을 받습니다.

**함정 2 — 데코레이터처럼 "어디에나" 붙일 수 있다고 생각한다.**

파이썬 데코레이터는 함수든 클래스든 자유롭게 붙습니다. Rust 속성 매크로는 입력 아이템의 종류(`ItemFn`, `ItemStruct` 등)를 매크로가 명시적으로 파싱하므로, 함수용 매크로를 구조체에 붙이면 `syn`의 파싱 단계에서 에러가 납니다. 한 매크로는 자기가 다룰 수 있는 아이템에만 붙습니다.

**함정 3 — `expr` 다음에 아무 토큰이나 올 수 있다고 생각한다.**

선언적 매크로의 프래그먼트 지정자에는 **follow-set 제약**이 있습니다. `$x:expr` 뒤에는 `=>`, `,`, `;`만 올 수 있습니다. `$x:expr + $y:expr` 같은 매처를 쓰면 다음 에러가 납니다(실제 `rustc` 출력입니다).

```text
error: `$x:expr` is followed by `+`, which is not allowed for `expr` fragments
 --> src/main.rs:3:15
  |
3 |     ( $x:expr + $y:expr ) => { $x + $y };
  |               ^ not allowed after `expr` fragments
  |
  = note: allowed there are: `=>`, `,` or `;`
```

이유는 모호성 방지입니다. `expr`은 욕심껏(greedily) 토큰을 먹어치우므로, `+` 뒤에 무엇이 표현식의 일부인지 매처의 일부인지 컴파일러가 구분할 수 없습니다. 해결책은 두 인자를 쉼표로 구분(`$x:expr, $y:expr`)하거나, 더 원시적인 `tt`를 쓰는 것입니다.

**함정 4 — 위생을 잊고 변수가 누출되길 기대한다.**

19.7 함정 2에서 본 그대로입니다. C 매크로에 익숙한 사람이 "매크로 안에서 변수를 만들면 밖에서 쓸 수 있겠지"라고 가정하면 `E0425`를 만납니다. Rust 매크로는 위생적이라, 내부 변수는 밖으로 새지 않습니다. 의도적으로 변수를 만들려면 이름을 `$name:ident`로 받으세요.

**함정 5 — 절차적 매크로의 컴파일 비용을 과소평가한다.**

`syn`은 무거운 의존성입니다. 작은 프로젝트에 절차적 매크로 크레이트를 하나 추가하면 빌드 시간이 눈에 띄게 늘 수 있습니다(`syn`+`quote`+`proc-macro2`가 함께 컴파일되니까요). 파이썬에서는 데코레이터를 추가해도 빌드 개념 자체가 없었지만, Rust에서는 컴파일 타임에 대가를 치릅니다. 다만 이것은 **컴파일러가 한 번 내는 비용**이지, 데코레이터처럼 **매 실행마다 내는 런타임 비용이 아니라**는 점에서 거래가 유리합니다.

**함정 6 — 매크로 에러 메시지가 항상 명확하리라 기대한다.**

복잡한 매크로의 에러는 펼쳐진 코드를 가리켜, 원본과 동떨어진 위치를 지적할 때가 있습니다. 이럴 때 `cargo expand`로 펼친 결과를 직접 보면 대부분 즉시 풀립니다(19.6). 절차적 매크로는 `compile_error!`나 `syn::Error::to_compile_error()`로 친절한 에러를 직접 만들어 줄 수도 있습니다.

---

지금까지 파이썬이 런타임에 부리던 마법 — 데코레이터의 함수 래핑, 메타클래스의 클래스 조작 — 을 Rust가 어떻게 컴파일 타임으로 끌어올렸는지 보았습니다. `macro_rules!`로 가변 인자와 컴파일 타임 코드 생성을 다루고, 위생으로 변수 충돌을 원천 차단하며, 절차적 매크로로 구조체를 분석해 트레이트 구현을 통째로 만들어냈습니다. 핵심 교훈은 하나입니다. **파이썬의 메타프로그래밍은 실행 중인 객체를 조작해 유연함을 얻는 대신 런타임 비용과 불투명함을 지불하고, Rust의 메타프로그래밍은 컴파일 중인 소스를 조작해 무비용 추상화와 타입 안전을 지킵니다.** `#[derive(Serialize)]` 한 줄이 Pydantic 전체를 대체하면서도 런타임 오버헤드가 0인 이유, `#[tokio::main]`이 `async fn`을 일반 함수로 둔갑시키는 원리가 이제 더 이상 마법으로 보이지 않을 것입니다.

매크로는 "코드를 짜는 코드"였습니다. 다음 Chapter 20에서는 다시 메모리의 세계로 내려가, 컴파일러의 엄격한 소유권 규칙을 런타임으로 살짝 미뤄주는 또 다른 도구 — `Box<T>`, `Rc<T>`, `RefCell<T>` 같은 **스마트 포인터(Smart Pointer)**를 다룹니다. 파이썬이 GC로 손쉽게 만들던 순환 참조 그래프와 양방향 연결 리스트를, Rust에서 안전하게 — 그리고 그 비용을 정확히 의식하며 — 손수 구현해볼 차례입니다.

---

## References

- https://doc.rust-lang.org/book/ch20-05-macros.html
- https://doc.rust-lang.org/reference/macros-by-example.html
- https://doc.rust-lang.org/reference/procedural-macros.html
- https://doc.rust-lang.org/rust-by-example/macros.html
- https://docs.rs/syn/latest/syn/
- https://docs.rs/quote/latest/quote/
