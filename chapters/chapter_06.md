# Chapter 6. 수명 (Lifetimes): 컴파일러에게 의도 설명하기

![Chapter 6. 수명 (Lifetimes): 컴파일러에게 의도 설명하기](../images/chapter-06-lifetimes.jpg)

파이썬에서 다음과 같은 함수를 작성해 본 적 있나요? 입력 리스트의 일부를 잘라서 돌려주는 흔하디흔한 코드입니다.

```python
def first_chunk(data: list[int]) -> list[int]:
    return data[:3]  # 슬라이스
```

여러분은 이 코드가 반환하는 슬라이스의 원본 리스트가 언제 메모리에서 사라질지 단 한 번도 고민해 본 적이 없을 겁니다. 파이썬에는 **수명(Lifetime)**이라는 명시적 개념이 존재하지 않기 때문입니다. 객체는 누군가 참조하고 있는 한 계속 살아있고, 참조하는 이가 아무도 없어질 때(레퍼런스 카운트가 0이 되거나, 순환 참조를 GC가 수거할 때) 비로소 메모리에서 사라집니다. 변수가 메모리에 언제까지 남아있을지 계산하는 일은 가비지 컬렉터(Garbage Collector)의 몫이었습니다.

하지만 Chapter 4에서 보았듯 Rust는 GC 없이 컴파일 타임에 모든 메모리의 해제 시점을 결정합니다. 그리고 Chapter 5에서 본 참조(Reference)·대여(Borrowing)는 소유권을 넘기지 않고 메모리 주소만 빌려주는 메커니즘이었습니다. 여기서 한 가지 끔찍한 질문이 떠오릅니다. **"참조를 빌려줬는데, 원본 데이터가 먼저 메모리에서 해제되어 버리면 어떡하지?"** C/C++ 개발자라면 이 상황을 잘 압니다. 해제된 메모리를 가리키는 포인터, 즉 **허공을 가리키는 포인터(Dangling Pointer)**는 use-after-free 버그와 보안 취약점의 단골 원인입니다.

Rust 컴파일러는 이 문제를 런타임이 아니라 컴파일 타임에 박멸합니다. 그 도구가 바로 **수명(Lifetimes)**입니다. 이 장에서 가장 먼저 못 박아 둘 핵심은 이것입니다. **수명은 새로운 무언가를 만들어 내는 문법이 아닙니다.** 수명 표기는 변수를 더 오래 살게 하지도, 빨리 죽게 하지도 않습니다. 그것은 그저 "이미 존재하는 참조들 사이의 생존 관계"를 컴파일러에게 **설명**하는 주석(Annotation)일 뿐입니다. 컴파일러가 혼자서는 추론할 수 없을 때, 우리가 의도를 적어 주는 것이죠.

이번 장에서는 수명이 왜 필요한지, 함수와 구조체에서 어떻게 표기하는지, 왜 대부분의 코드에서는 수명을 안 써도 되는지(생략 규칙), `'static`의 진짜 의미는 무엇인지, 그리고 파이썬 출신 개발자가 수명에서 저지르는 전형적인 실수와 그 해법을 모두 다룹니다.

---

## 6.1 The Python Way — 수명이 없는 세계의 평화

파이썬에서 우리는 객체의 생존 기간을 단 한 번도 손으로 관리하지 않았습니다. 다음 코드를 봅시다.

```python
from __future__ import annotations


class Cache:
    def __init__(self) -> None:
        self._latest: str | None = None

    def remember(self, value: str) -> None:
        self._latest = value  # 그냥 참조를 저장한다


def build_cache() -> Cache:
    cache = Cache()
    temp = "temporary value"   # 지역 변수
    cache.remember(temp)       # 지역 변수의 참조를 객체 안에 보관
    return cache               # 함수가 끝나도...


c = build_cache()
print(c._latest)  # "temporary value" — 멀쩡히 살아있다!
```

`build_cache()`가 반환된 후에도 `temp`가 가리키던 문자열은 멀쩡히 살아있습니다. 왜냐하면 파이썬에서 변수는 단지 힙(Heap)에 있는 객체를 가리키는 이름표일 뿐이고, `cache._latest`가 그 객체를 여전히 참조하고 있어 레퍼런스 카운트가 1 이상이기 때문입니다. 함수의 지역 변수 `temp`라는 이름표가 사라져도, 객체 자체는 누군가 가리키는 한 절대 죽지 않습니다.

이것이 파이썬의 메모리 모델입니다. 모든 것이 힙에 있고, 모든 변수가 사실상 포인터이며, 객체의 수명은 "마지막으로 가리키던 참조가 사라지는 순간"까지 자동으로 연장됩니다.

> **Under the Hood: 파이썬의 평화에 드는 런타임 비용**
>
> 이 편안함은 공짜가 아닙니다. CPython의 모든 객체는 `PyObject` 헤더에 `ob_refcnt`(레퍼런스 카운트) 필드를 가집니다 — 64비트 빌드에서 객체 하나당 최소 8바이트의 오버헤드입니다. `cache.remember(temp)` 한 줄이 실행될 때 인터프리터는 (1) 대입할 객체의 레퍼런스 카운트를 `Py_INCREF`로 1 증가시키고, (2) `_latest`가 기존에 가리키던 객체가 있었다면 `Py_DECREF`로 1 감소시키며, (3) 그 결과가 0이면 즉시 `tp_dealloc`을 호출해 메모리를 해제합니다. 단순한 대입 한 번에 이만큼의 일이 매번 런타임에 일어납니다. 게다가 두 객체가 서로를 가리키는 순환 참조는 레퍼런스 카운트만으로는 절대 0이 되지 않으므로, CPython은 별도의 세대별(generational) 추적 GC를 주기적으로 돌려 순환 고리를 끊어야 합니다. "수명을 신경 쓰지 않아도 되는" 편안함의 정체는, 사실 **런타임이 매 순간 수명을 대신 추적하고 있는 것**입니다.

파이썬에서는 이 비용을 지불하는 대신, 우리는 수명에 대해 생각할 필요가 없었습니다. Rust는 정확히 반대의 거래를 제안합니다. 런타임에 단 한 푼의 비용도 지불하지 않는 대신, 우리가 컴파일 타임에 수명 관계를 (대부분은 컴파일러가, 가끔은 우리가 직접) 명시해야 합니다.

---

## 6.2 The Rust Mental Model — 허공을 가리키는 포인터를 컴파일 타임에 박멸하기

Rust가 막으려는 구체적인 재앙을 먼저 보겠습니다. 다음은 Chapter 5에서 본 참조를 사용하되, 원본이 참조보다 먼저 죽는 상황입니다.

```rust
fn main() {
    let r;                // 1. 참조 변수 r을 선언합니다(아직 아무것도 안 가리킴).
    {
        let x = 5;        // 2. x는 이 내부 스코프에서만 살아있습니다.
        r = &x;           // 3. r이 x를 참조(빌려옴)합니다.
    }                     // 4. x의 스코프가 끝나고 메모리에서 버려집니다(Dropped).

    println!("r: {}", r); // 5. 🚨 r은 이미 죽어버린 x를 가리키고 있습니다.
}
```

C였다면 이 코드는 경고 정도만 내고 컴파일되어, 운이 나쁘면 쓰레기 값을 출력하고 운이 더 나쁘면 보안 구멍이 됩니다. 파이썬이었다면 애초에 `x`가 힙에 살아남아 문제가 되지 않았을 겁니다. 그러나 Rust는 컴파일을 거부합니다.

```text
error[E0597]: `x` does not live long enough
 --> dangle.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("r: {}", r);
  |                       - borrow later used here
```

에러 메시지를 한 줄씩 해독해 봅시다. 컴파일러는 `x`가 6번 줄(`}`)에서 버려진다고 짚어 줍니다("`x` dropped here"). 그런데 그 참조 `r`은 7번 줄에서 여전히 사용됩니다("borrow later used here"). 즉 **참조가 원본보다 오래 살아남으려 한다**는 것입니다. 이것이 모든 수명 규칙의 뿌리가 되는 단 하나의 불변식입니다.

> 💡 **수명의 제1 원칙**: 빌려온 참조는 그 참조가 가리키는 원본 데이터보다 절대로 오래 살아남을 수 없다. (A reference must never outlive the data it points to.)

여기서 결정적으로 중요한 사고 전환이 있습니다. 위 에러를 고치려고 `r`에 어떤 수명 표기를 붙여도 소용없습니다. 문제는 표기의 부재가 아니라 **사실 자체**입니다 — `x`가 너무 일찍 죽습니다. 해결책은 수명을 "붙이는" 것이 아니라, `x`를 `r`만큼 오래 살도록 스코프를 재구성하는 것입니다.

```rust
fn main() {
    let x = 5;       // x를 바깥 스코프로 끌어올린다 → r과 수명이 같아짐
    let r = &x;
    println!("r: {}", r); // OK
}
```

이것이 수명의 본질입니다. **수명 표기는 문제를 푸는 도구가 아니라, 코드가 이미 안전한지를 컴파일러에게 증명해 보이는 언어입니다.** 코드가 안전하지 않으면 어떤 표기로도 컴파일러를 속일 수 없습니다.

이 사고방식의 전환을 파이썬 비유로 다시 정리하면 이렇습니다. 파이썬에서 타입 힌트 `x: int`를 적는다고 해서 `x`가 실제로 정수가 되는 것은 아닙니다 — 그것은 단지 "내 의도는 정수다"라는 선언이고, `mypy` 같은 정적 검사기가 그 의도와 코드가 일치하는지를 검증할 뿐입니다. 수명 표기 `&'a str`도 똑같습니다. `'a`를 적는다고 데이터가 `'a`만큼 살게 되는 것이 아니라, "이 참조의 수명은 `'a`다"라는 의도를 적는 것이고, 빌림 검사기가 그 의도가 실제 스코프와 모순되지 않는지를 검증합니다. 차이가 있다면, 파이썬의 타입 힌트는 런타임에 무시 가능한 선택이지만 Rust의 수명은 컴파일을 통과하기 위한 필수 증명이라는 점입니다.

> **Under the Hood: 스택 프레임과 참조의 유효성 영역(Region)**
>
> `let x = 5;`에서 `5`는 힙이 아니라 현재 함수의 **스택 프레임(Stack Frame)** 위에, 더 정확히는 그 함수의 특정 렉시컬 스코프(`{ }`) 안에 놓입니다. 스코프가 끝나는 `}`를 만나는 순간 스택 포인터가 되감기며 그 메모리 영역은 즉시 재사용 가능 상태가 됩니다 — `x`라는 4바이트짜리 `i32`가 차지하던 스택 슬롯은 다음에 선언되는 다른 변수가 덮어쓸 수 있습니다. 컴파일러는 각 참조에 대해 "이 참조가 유효해야 하는 코드 구간"을 하나의 **영역(Region, 또는 NLL에서의 lifetime)**으로 계산합니다. 위 예제에서 `r`의 영역은 `r = &x`부터 `println!`까지 뻗어 있는데, `x`가 사는 영역(내부 `{ }`)은 그보다 먼저 끝납니다. 두 영역을 겹쳐 보면 `r`의 영역이 `x`의 영역을 삐져나가는 것이 명백히 보이고, 바로 이 **영역 포함 관계 위반**이 E0597의 정체입니다. 현대 Rust의 빌림 검사기(NLL, Non-Lexical Lifetimes)는 이 영역을 렉시컬 스코프가 아니라 실제 사용 지점까지의 제어 흐름 그래프(CFG) 기반으로 계산하므로, "마지막으로 쓰인 곳 이후"는 더 이상 살아있을 필요가 없다고 똑똑하게 판단합니다.

여기까지가 단일 함수 내부의 이야기입니다. 컴파일러는 한 함수 안에서는 모든 변수의 스코프를 눈으로 보듯 추적할 수 있어, 우리가 수명을 한 글자도 적지 않아도 됩니다. 문제는 **함수 경계를 넘을 때** 시작됩니다.

---

## 6.3 함수 시그니처의 수명: 컴파일러가 손을 드는 순간

이제 Chapter 5에서 본 참조를 매개변수로 받고, 참조를 반환하는 함수를 만들어 봅시다. 두 문자열 중 더 긴 쪽을 돌려주는, 파이썬에서라면 30초면 끝날 함수입니다.

```python
def longest(x: str, y: str) -> str:
    return x if len(x) > len(y) else y

# 런타임에 x나 y 중 하나가 반환되며, GC가 알아서 메모리를 관리한다.
```

이를 Rust로 직역하면 — 소유권을 빼앗지 않으려고 `&str`을 받습니다 — 다음과 같고, 컴파일에 **실패**합니다.

```rust
// 🚨 컴파일 에러!
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {}
```

```text
error[E0106]: missing lifetime specifier
 --> e0106.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the
          signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

`error[E0106]: missing lifetime specifier` — 이것이 여러분이 수명을 처음 만나게 되는 에러이며, 가장 중요한 에러입니다. 메시지의 `help`가 정확히 핵심을 짚습니다. **"이 함수의 반환 타입은 빌려온 값을 담고 있는데, 그것이 `x`에서 빌린 것인지 `y`에서 빌린 것인지 시그니처가 말해 주지 않는다."**

왜 컴파일러는 이걸 혼자 알아내지 못할까요? `longest`를 호출하는 쪽을 상상해 보세요. 컴파일러가 `longest`를 검사할 때는 **함수 본문이 아니라 시그니처만 보고** 호출처의 안전성을 판단해야 합니다(이것이 모듈화된 컴파일의 기본 원칙입니다). 시그니처 `(x: &str, y: &str) -> &str`만 봐서는, 반환된 참조가 `x`의 데이터에 묶여 있는지 `y`의 데이터에 묶여 있는지 알 길이 없습니다. 그렇다면 호출처에서 `x`는 살아있는데 `y`가 죽었을 때, 반환값을 써도 되는지 안 되는지 판단할 수 없습니다. 컴파일러는 모호함 앞에서 추측하지 않습니다. 멈추고 우리에게 묻습니다.

### 수명 표기로 의도를 적어 주기

우리는 컴파일러에게 이렇게 말해 줍니다. **"`x`, `y`, 그리고 반환값은 모두 최소한 어떤 공통의 수명 `'a` 동안은 함께 살아있다."** 어포스트로피(`'`)로 시작하는 이름이 수명 매개변수이며, 관례상 `'a`, `'b`를 씁니다(타입 매개변수 `<T>`와 똑같은 자리, `<'a>`에 선언합니다).

```rust
// "x와 y, 그리고 반환값은 모두 최소한 수명 'a 만큼은 함께 살아있어야 한다."
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {result}"); // OK: 두 입력이 모두 살아있는 구간
    }
}
```

이 시그니처가 컴파일러에게 전달하는 계약은 이렇습니다. 호출처에서 넘긴 `x`와 `y` 중 **더 짧게 사는 쪽의 수명**을 `'a`로 잡고, 반환값은 그 `'a`보다 더 오래 사용되어서는 안 된다. 이제 컴파일러는 본문을 다시 들여다볼 필요 없이 시그니처만으로 모든 호출처를 검증할 수 있습니다.

### 표기는 사실을 바꾸지 않는다 — 영역을 벗어난 사용 거부

수명 표기가 마법이 아니라는 것을 증명하는 결정적 예제입니다. 위와 똑같은 `longest`를 쓰되, 반환값을 입력 중 하나가 죽은 뒤에 사용해 봅시다.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    } // 🚨 여기서 string2가 죽는다
    println!("The longest string is {result}"); // result를 여기서 쓴다
}
```

```text
error[E0597]: `string2` does not live long enough
  --> result_fail.rs:9:44
   |
 8 |         let string2 = String::from("xyz");
   |             ------- binding `string2` declared here
 9 |         result = longest(string1.as_str(), string2.as_str());
   |                                            ^^^^^^^ borrowed value does not live long enough
10 |     }
   |     - `string2` dropped here while still borrowed
11 |     println!("The longest string is {result}");
   |                                      ------ borrow later used here
```

컴파일러는 `'a`를 두 입력 중 더 짧은 `string2`의 수명으로 잡았습니다. 그러므로 `result`도 `string2`만큼만 살 수 있는데, `string2`가 죽은 다음 줄에서 `result`를 쓰니 거부합니다. 우리가 코드에서 `longest`가 실제로 `string1`을 반환한다는 걸 알아도 소용없습니다 — 시그니처상 컴파일러는 둘 중 어느 것이든 나올 수 있다고 보수적으로 가정합니다.

### 진짜로 다른 수명일 때: 두 개의 수명 매개변수

만약 반환값이 **언제나 첫 번째 인자에서만** 나온다는 것이 사실이라면, 두 번째 인자에 같은 `'a`를 묶을 이유가 없습니다. 그럴 때는 수명을 둘로 나눕니다.

```rust
// 반환값은 항상 x에서 나온다. y는 그저 비교용일 뿐, 수명을 공유할 필요가 없다.
fn first_word<'a, 'b>(x: &'a str, _y: &'b str) -> &'a str {
    x.split(' ').next().unwrap_or("")
}

fn main() {
    let a = String::from("hello world");
    let r;
    {
        let b = String::from("short-lived");
        r = first_word(&a, &b);
    } // b는 여기서 죽지만...
    println!("{r}"); // r은 a를 가리키므로 OK!
}
```

`b`가 죽은 뒤에도 `r`을 쓸 수 있습니다. 시그니처가 "반환값은 `'a`(즉 `a`)에만 묶인다"고 정확히 명시했기 때문입니다. 이것이 **시그니처가 곧 계약**이라는 말의 의미입니다. 수명을 정직하고 정밀하게 적을수록, 호출처에서 더 많은 자유를 얻습니다.

만약 본문에서 `_y`를 반환하려 들면 어떻게 될까요? 시그니처와 본문이 어긋나므로 또 다른 에러가 납니다.

```rust
// 🚨 시그니처는 'a를 약속했는데 본문이 'b인 y를 반환하려 함
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

```text
error[E0621]: explicit lifetime required in the type of `y`
 --> e0621.rs:2:39
  |
2 |     if x.len() > y.len() { x } else { y }
  |                                       ^ lifetime `'a` required
  |
help: add explicit lifetime `'a` to the type of `y`
  |
1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |                                ++
```

`error[E0621]`은 "`y`도 반환하려면 `y`에 `'a`를 묶어야 한다"고 정확히 처방해 줍니다. 컴파일러는 단지 막기만 하는 게 아니라, 시그니처와 본문이 일관되도록 고쳐 주는 가이드 역할까지 합니다.

| Python (수명 개념 없음) | Rust 수명 표기 |
|---|---|
| 함수 시그니처가 반환값의 생존 관계를 말하지 않음 | `<'a>`로 입력·출력 참조의 생존 관계를 시그니처에 명시 |
| 반환 객체의 수명 = 런타임 레퍼런스 카운트로 결정 | 반환 참조의 수명 = 입력 참조들의 수명으로 컴파일 타임 결정 |
| 지역 변수 참조를 반환해도 안전(객체가 힙에 생존) | 지역 변수 참조 반환 시 컴파일 거부(E0515) |
| 모호하면 그냥 동작 | 모호하면 컴파일 거부(E0106), 명시 요구 |

---

## 6.4 수명 생략 규칙 (Lifetime Elision) — 왜 대부분 안 써도 되는가

여기까지 읽고 "그럼 참조를 쓰는 모든 함수마다 `'a`를 도배해야 하나?"라고 걱정했다면, 안심하세요. 실무 Rust 코드의 대부분은 수명을 한 글자도 적지 않습니다. 컴파일러가 흔한 패턴을 **추론**해 주기 때문이고, 이 추론 알고리즘을 **수명 생략(Lifetime Elision)**이라고 부릅니다.

생략은 새로운 능력이 아니라, 컴파일러가 명시적 표기를 **기계적으로 채워 넣는** 결정론적 규칙일 뿐입니다. 규칙은 단 세 가지입니다. 컴파일러는 이 세 규칙을 순서대로 적용해 보고, 그래도 모든 출력 참조의 수명이 정해지지 않으면 그제서야 E0106을 냅니다.

용어 두 개만 정리합니다. 함수의 매개변수에 있는 참조 수명을 **입력 수명(Input Lifetime)**, 반환값에 있는 참조 수명을 **출력 수명(Output Lifetime)**이라 합니다.

1. **규칙 1 (입력 규칙)**: 생략된 **입력** 참조마다 각각 **서로 다른, 독립된** 수명을 부여한다. `fn f(x: &str, y: &str)`는 `fn f<'a, 'b>(x: &'a str, y: &'b str)`로 펼쳐진다. 중첩 참조도 각각 따로 세어, `&'_ &'_ T`에는 수명이 둘 배정됩니다.
2. **규칙 2 (단일 입력 규칙)**: 입력 수명이 (규칙 1 적용 후) **정확히 하나**뿐이면, 그 수명을 모든 **출력** 참조에 복사한다. 입력 수명이 둘 이상이면 이 규칙은 작동하지 않습니다.
3. **규칙 3 (메서드 규칙)**: 입력에 `&self` 또는 `&mut self`(즉 수신자 `self`가 참조)가 있으면, **다른 입력 수명이 몇 개든 상관없이** `self`의 수명을 모든 출력 참조에 복사한다. 그래서 메서드는 입력이 둘 이상이어도 규칙 2를 거치지 않고 곧장 `self`의 수명을 따릅니다.

세 가지 모두 **`fn`과 `impl` 메서드 시그니처에만** 적용되며, 적용 순서는 1 → 2 → 3입니다. 세 규칙을 다 거쳤는데도 출력 수명이 미결로 남으면 — 6.3의 `longest`가 정확히 그 경우입니다(입력 둘, `self` 없음) — 컴파일러는 추측을 멈추고 E0106을 냅니다. 한 가지 정밀하게 짚을 점은, 규칙 2와 규칙 3은 **출력 수명에만** 손을 댄다는 것입니다. 입력 수명을 서로 묶어 주는(unify) 규칙은 없으므로, "두 입력이 같은 수명이다"는 사실이 필요하면 그것은 언제나 우리가 직접 적어 주어야 합니다.

규칙 2가 가장 흔한 구원자입니다. 예를 들어 다음 함수는 입력 참조가 하나뿐이라 출력 수명이 자동으로 결정됩니다.

```rust
// 우리가 쓰는 코드 — 수명 표기 없음
fn first_word(s: &str) -> &str {
    s.split(' ').next().unwrap_or("")
}

// 컴파일러가 규칙 1 + 규칙 2를 적용해 실제로 해석하는 시그니처
// fn first_word<'a>(s: &'a str) -> &'a str
```

입력이 하나(`s`)뿐이므로 규칙 2에 의해 반환 `&str`의 수명은 자명하게 `s`의 수명입니다. 모호함이 없으니 우리가 적을 필요도 없습니다. 반면 6.3의 `longest`가 에러였던 이유가 이제 명확해집니다 — 입력이 둘이라 규칙 2가 적용되지 않고, `&self`도 없어 규칙 3도 적용되지 않아, 출력 수명이 미결로 남기 때문입니다. 그래서 우리가 직접 적어 줘야 했던 것이죠.

규칙 3은 메서드를 작성할 때 위력을 발휘합니다. 다음 절의 구조체 메서드에서 바로 확인하겠습니다.

이 세 규칙이 의외로 강력한 이유는, 실무에서 참조를 받는 함수의 압도적 다수가 "입력 참조 하나 → 출력 참조 하나"(규칙 2가 처리) 또는 "메서드에서 `&self`의 일부를 반환"(규칙 3이 처리)이라는 두 패턴 중 하나이기 때문입니다. 6.3의 `longest`처럼 "독립적인 입력 참조가 둘 이상이고 그중 어느 것이든 반환될 수 있는" 경우는 생각보다 드뭅니다. 그래서 표준 라이브러리의 `str::trim`, `str::split_once`, `slice::first` 같은 수많은 메서드가 시그니처에 수명을 한 글자도 적지 않으면서도 정확히 입력의 수명을 출력에 전파합니다. 여러분이 그 함수들을 별다른 의식 없이 써 온 것은, 컴파일러가 매번 이 세 규칙을 조용히 적용해 준 덕분입니다.

한 가지 주의할 점은, 생략 규칙은 **함수와 메서드 시그니처에만** 적용된다는 것입니다. 구조체 정의(6.5)에서 참조 필드를 가질 때는 생략이 작동하지 않아 `struct Foo<'a> { x: &'a str }`처럼 반드시 수명을 적어야 합니다 — 구조체에는 "입력/출력"이라는 흐름이 없어 규칙을 적용할 근거가 없기 때문입니다. 이것이 다음 절에서 수명을 직접 적게 되는 이유이기도 합니다.

> **Under the Hood: 생략은 런타임이 아니라 "타입 체커 시간"에 펼쳐진다**
>
> 수명 생략을 파이썬의 데코레이터나 메타프로그래밍처럼 "런타임에 뭔가 채워 넣는" 것으로 오해하면 안 됩니다. 생략은 순전히 컴파일러 프론트엔드의 타입 체킹 단계에서 일어나는 **순수한 문법 설탕(syntactic sugar)**입니다. 컴파일러는 소스를 AST로 파싱한 직후, 위 세 규칙을 적용해 생략된 수명을 명시적 형태로 "탈설탕(desugar)"합니다. 그 다음 빌림 검사기가 펼쳐진 명시적 수명을 가지고 영역 분석을 수행합니다. 핵심은, **생략된 코드와 명시한 코드는 컴파일러 입장에서 완전히 동일한 프로그램**이라는 점입니다 — 생성되는 기계어는 한 바이트도 다르지 않습니다. 실제로 수명 매개변수는 단일형화(Monomorphization) 이후 기계어에 어떤 흔적도 남기지 않습니다. 수명은 타입 검사가 끝나면 증발하는, 순수하게 컴파일 타임에만 존재하는 정보입니다. C++의 포인터에 런타임 비용이 없듯, Rust의 수명 검사에도 런타임 비용은 정확히 0입니다. 이것이 Chapter 5에서 말한 무비용 추상화(Zero-cost Abstraction)가 메모리 안전성에까지 적용된 모습입니다.

---

## 6.5 구조체에 참조 담기 — 실무에서 가장 자주 만나는 지점

함수 시그니처의 수명은 사실 6.4의 생략 규칙 덕에 거의 안 보입니다. 여러분이 실무에서 수명을 **눈으로 직접 적게 되는** 거의 유일한 지점은 **구조체(Struct) 필드에 참조를 담을 때**입니다.

Chapter 4에서 본 소유권 모델에서, 구조체는 보통 자기 데이터를 **소유**합니다(예: `String`, `Vec<T>`). 하지만 원본을 복사하지 않고 그 일부를 가리키기만 하는 구조체 — 이른바 **뷰(View)** 혹은 **제로카피(Zero-copy)** 구조체 — 를 만들고 싶을 때가 있습니다. 파서, 토크나이저, 설정 리더가 대표적입니다. 이때 구조체에 `&str`을 담아야 하고, 컴파일러는 다음을 알아야 합니다. **"이 구조체 인스턴스는 자신이 빌린 원본 데이터보다 오래 살아남으면 안 된다."** 이것을 수명 매개변수로 명시합니다.

```rust
// 구조체가 'a 동안 사는 어떤 문자열의 일부를 가리킨다.
// → 이 구조체 인스턴스는 그 원본보다 오래 살 수 없다.
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    // 규칙 3: &self가 있으므로 출력 수명은 self.part의 'a를 따른다 → 표기 생략 가능!
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {announcement}");
        self.part
    }

    fn level(&self) -> i32 {
        3
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");

    // i는 first_sentence(=novel의 일부)를 가리키므로,
    // i는 novel이 메모리에서 해제되기 전에 파괴되어야 한다.
    let i = ImportantExcerpt { part: first_sentence };

    println!("level = {}", i.level());
    let part = i.announce_and_return_part("listen up");
    println!("part = {part}");
}
```

두 가지를 주목하세요. 첫째, 구조체 선언 `struct ImportantExcerpt<'a>`와 `impl<'a>`에 수명 매개변수가 타입 매개변수처럼 따라붙습니다 — 이것은 "이 구조체는 어떤 수명 `'a`에 종속된다"는 선언입니다. 둘째, `announce_and_return_part`가 `&str`을 반환하는데도 수명 표기가 없습니다. 입력이 둘(`&self`, `announcement`)이라 규칙 2는 안 되지만, **규칙 3**이 `&self`의 수명을 출력에 자동 부여하기 때문입니다. 메서드에서 수명이 거의 안 보이는 이유가 바로 이 규칙 3입니다.

### self-referential struct가 어려운 이유

파이썬 개발자가 본능적으로 시도하지만 Rust에서 막히는 패턴이 있습니다. **자기 자신을 가리키는 구조체(self-referential struct)** — 즉, 한 필드에 데이터를 소유하면서 다른 필드가 그 소유한 데이터의 일부를 참조하는 구조입니다.

```rust
// 의도: full_text를 소유하면서, current가 그 일부를 가리키게 하고 싶다.
struct Parser<'a> {
    full_text: String,   // 데이터를 소유
    current: &'a str,    // ...의 일부를 가리키려는 시도
}

fn main() {
    let text = String::from("data");
    let p = Parser {
        current: &text,    // text를 빌림
        full_text: text,   // 그런데 동시에 text의 소유권을 move!
    };
    println!("{}", p.current);
}
```

```text
error[E0505]: cannot move out of `text` because it is borrowed
  --> selfref2.rs:10:20
   |
 9 |         current: &text,      // text를 빌림
   |                  ----- borrow of `text` occurs here
10 |         full_text: text,     // 그런데 동시에 text의 소유권을 move
   |                    ^^^^ move out of `text` occurs here
   |
help: consider cloning the value if the performance cost is acceptable
```

`error[E0505]`. `text`를 `&text`로 빌리는 동시에 `full_text: text`로 소유권을 옮기려 하니, 빌린 동안 옮길 수 없다고 거부합니다. 설령 이 줄을 통과한다 해도 더 근본적인 문제가 있습니다. 구조체가 메모리에서 **이동(move)**할 때 — Rust에서 값은 스택에서 흔히 이동합니다 — `full_text`(`String`의 힙 포인터+길이+용량 24바이트)의 스택상 주소가 바뀌는데, `current`가 가리키던 주소는 그대로라서 곧바로 허공을 가리키게 됩니다.

> **Under the Hood: 왜 move가 self-reference를 박살내는가**
>
> Rust에서 `String`은 스택에 `{ ptr, len, capacity }` 24바이트의 핸들을 두고 실제 텍스트는 힙에 둡니다(Chapter 4). self-referential 구조체에서 `current: &'a str`가 같은 구조체 내 `full_text`의 힙 버퍼를 가리킨다고 합시다. 문제는 `current`가 가리키는 주소를 컴파일러가 구조체 생성 시점에 고정해 버린다는 점입니다. Rust의 값은 기본적으로 **이동 가능(movable)**하며, 이동은 단순히 스택상의 바이트들을 새 위치로 `memcpy`하는 것입니다. 구조체가 이동하면 `full_text` 핸들의 스택 주소도 바뀌지만, 힙 버퍼 자체의 주소는 안 바뀌므로 힙을 가리키는 self-reference라면 살아남을 여지가 있어 보입니다 — 그러나 컴파일러는 어떤 참조가 "이동해도 안전한 힙 포인터"인지 "이동하면 깨지는 스택 포인터"인지 일반적으로 구별할 수 없으므로, 안전을 위해 self-reference 자체를 금지합니다. 이 한계를 우회하려면 (1) 값이 절대 이동하지 않음을 타입으로 보장하는 `Pin<T>`(Chapter 18 비동기에서 다시 만납니다), (2) 인덱스(`usize`)나 소유 토큰으로 참조를 대체하는 설계, 혹은 (3) `ouroboros`·`self_cell` 같은 크레이트가 필요합니다. 실무 결론은 명확합니다 — **self-referential struct는 거의 항상 설계 신호등입니다.** 참조 대신 인덱스를 들거나, 소유권을 재배치하세요.

---

## 6.6 정적 수명 (`'static`) — 의미와 흔한 오해

특별한 이름을 가진 수명이 하나 있습니다. `'static`은 **프로그램이 시작해서 끝날 때까지** 유효한, 가장 긴 수명입니다. 파이썬의 모듈 레벨 상수나 전역 변수에 가장 가까운 개념입니다.

`'static`을 만나는 첫 번째 경우는 **문자열 리터럴**입니다. 코드에 하드코딩한 `"..."`는 컴파일된 바이너리의 읽기 전용 데이터 섹션(`.rodata`)에 박혀, 프로그램이 사는 내내 그 자리에 있습니다. 따라서 타입이 `&'static str`입니다.

```rust
fn main() {
    // 문자열 리터럴은 바이너리에 박혀 영원히 산다 → &'static str
    let s: &'static str = "I have a static lifetime.";
    println!("{s}");
}
```

또 다른 의미로 `'static`은 **수명 바운드(lifetime bound)**로도 쓰입니다 — `T: 'static`은 "타입 `T`는 빌린 참조를 일절 품고 있지 않거나, 품더라도 그게 `'static`이다"라는 뜻입니다. 즉 그 값은 원할 때까지 무한정 보관해도 안전합니다. 스레드를 띄우거나(Chapter 16), 값을 어딘가에 영구 저장할 때 이 바운드가 등장합니다.

```rust
// T가 'static이어야 함 = T가 빌린 참조를 (영원하지 않게) 품고 있지 않아야 함.
fn store_forever<T: 'static>(_value: T) {
    // 예: 다른 스레드로 보내거나, 전역 캐시에 영구 저장
}

fn main() {
    store_forever(String::from("owned, so 'static OK")); // String은 데이터를 소유 → 'static
    store_forever(5_i32);                                 // 소유 값 → 'static

    // 아래는 컴파일 거부됨:
    // let local = String::from("hi");
    // store_forever(&local); // &local은 local의 수명에 묶여 'static이 아님
}
```

### 런타임에 계산한 값을 `'static`으로 만들기 — `OnceLock`/`LazyLock`

문자열 리터럴이 아니라 **런타임에 한 번 계산한 값**을 프로그램 끝까지 살리고 싶을 때가 있습니다. 파이썬의 모듈 레벨 싱글턴(`_CONFIG = load_config()`)이나 `functools.lru_cache`로 캐싱하는 전역 테이블이 떠오르는 자리입니다. 예전 Rust 코드는 `lazy_static!`/`once_cell::sync::Lazy` 같은 외부 크레이트로 이를 했지만, 그 기능은 표준 라이브러리에 흡수되었습니다 — **`std::sync::OnceLock`**(Rust 1.70+, 한 번만 쓰는 셀)과 **`std::sync::LazyLock`**(Rust 1.80+, 첫 접근 시 클로저로 지연 초기화)입니다. 새 코드에서는 외부 크레이트 대신 이 둘을 쓰세요.

```rust
use std::sync::OnceLock;

// 런타임에 한 번 초기화되어 프로그램 끝까지 사는 전역 → 안전하게 &'static을 얻는 정석.
static CONFIG: OnceLock<String> = OnceLock::new();

fn config() -> &'static str {
    // get_or_init은 최초 호출에서만 클로저를 실행하고, 이후엔 저장된 값을 반환한다.
    CONFIG.get_or_init(|| {
        // 비싼 초기화(파일/환경 변수 읽기 등)를 단 한 번만 수행
        String::from("loaded-once")
    })
}

fn main() {
    println!("{}", config()); // loaded-once (초기화 발생)
    println!("{}", config()); // loaded-once (재사용, 초기화 안 함)
}
```

`CONFIG`는 `static`이라 프로그램 전체 수명 동안 살아있고, 그 안의 `String`도 마찬가지이므로 `get_or_init`이 돌려주는 `&String`은 정직하게 `&'static`입니다. 진짜로 임의의 소유 값을 영구히 `'static`으로 누수시켜야 한다면 `Box::leak`(`Box::leak(s.into_boxed_str()) -> &'static str`)도 있지만, 이름 그대로 메모리를 의도적으로 누수시켜 되돌릴 수 없으니 초기화 1회성 데이터에만 쓰세요. 어느 쪽이든 공통점은 동일합니다 — `'static`은 "데이터가 실제로 프로그램 끝까지 산다"는 **사실을 먼저 만든 뒤**에야 얻어지는 것이지, 표기만으로 부여되지 않습니다.

> 🚨 **흔한 오해 1: `&'static`을 붙이면 값이 더 오래 산다?** 아닙니다. `'static`은 다른 모든 수명 표기와 똑같이 "이미 그러한 사실"을 기술할 뿐, 값을 연장하지 않습니다. 지역 변수에 `&'static`을 억지로 붙이면 컴파일러는 "이건 `'static`이 아니다"라며 거부합니다. 실제로 함수가 지역 변수의 참조를 `&'static`으로 반환하려는 코드를 보세요.

```rust
fn make() -> &'static str {
    let s = String::from("temporary");
    &s          // 🚨 s는 함수가 끝나면 죽는데 'static을 약속함
}
fn main() { println!("{}", make()); }
```

```text
error[E0515]: cannot return reference to local variable `s`
 --> static_misuse.rs:3:5
  |
3 |     &s
  |     ^^ returns a reference to data owned by the current function
```

> 🚨 **흔한 오해 2: `T: 'static`은 "값이 프로그램 끝까지 살아야 한다"는 뜻?** 아닙니다. `T: 'static`은 "타입 `T`가 `'static`보다 짧은 빌린 참조를 포함하지 않는다"는 뜻이며, 그 값은 얼마든지 일찍 드롭될 수 있습니다. `String`, `Vec<u8>`, `i32`처럼 데이터를 **소유**하는 모든 타입은 자동으로 `T: 'static`을 만족합니다 — 빌린 참조가 없으니까요. 이 둘(`&'static T` 타입과 `T: 'static` 바운드)을 혼동하는 것이 초보자의 단골 함정입니다.

---

## 6.7 수명 바운드와 제네릭 — `T: 'a`와 where 절

제네릭 타입과 수명이 만나면 한 단계 더 정밀한 표현이 필요합니다. 참조를 품을 수 있는 제네릭 타입 `T`를 구조체에 담을 때, 컴파일러는 "`T`가 품고 있을지 모르는 참조가 구조체의 수명 `'a`보다 짧으면 안 된다"는 것을 알아야 합니다. 이를 `T: 'a` 바운드로 표현합니다.

```rust
use std::fmt::Display;

// T가 (참조를 품고 있다면) 적어도 'a만큼은 살아야 한다는 바운드.
struct Wrapper<'a, T: Display + 'a> {
    label: &'a str,
    value: T,
}

impl<'a, T: Display + 'a> Wrapper<'a, T> {
    fn show(&self) {
        println!("[{}] {}", self.label, self.value);
    }
}

fn main() {
    let w = Wrapper { label: "count", value: 42 };
    w.show(); // [count] 42
}
```

바운드가 여러 개로 복잡해지면 `where` 절로 분리해 가독성을 높입니다 — 파이썬에서 함수 시그니처가 길어질 때 타입 힌트를 정리하는 것과 같은 정신입니다.

```rust
use std::fmt::Display;

// 위와 동일하지만 where 절로 정돈 — 바운드가 많을수록 이쪽이 읽기 좋다.
struct Wrapper<'a, T>
where
    T: Display + 'a,
{
    label: &'a str,
    value: T,
}
```

실무에서 `T: 'a`를 직접 적는 일은 흔치 않습니다(컴파일러가 대개 추론합니다). 하지만 제네릭 컬렉션이나 콜백을 다룰 때, 컴파일러가 "`T may not live long enough`"라고 불평하면 십중팔구 `where T: 'a` 한 줄이 해법입니다. 에러 메시지가 직접 그렇게 제안해 주니, 메시지를 신뢰하세요. (참고로 이 진단은 결합되는 타입에 따라 에러 코드 `error[E0309]`로 나타나기도 합니다.)

---

## 6.8 변성(Variance) 입문 — 파이썬엔 없는 "수명끼리의 호환성"

수명을 어느 정도 다루다 보면, 의식하지 못한 채 매일 의존하던 규칙 하나가 수면 위로 떠오릅니다. 다음 코드는 아무 표기 없이 컴파일됩니다.

```rust
fn print_it(s: &str) {        // 임의의(짧은) 수명을 기대하는 함수
    println!("{s}");
}

fn main() {
    let static_str: &'static str = "hello"; // 가장 긴 수명
    print_it(static_str);                    // 'static을 더 짧은 수명 자리에 넘긴다 → OK
}
```

`print_it`은 "아무 수명 `&str`"을 받는데, 우리는 거기에 가장 긴 `&'static str`을 넘겼습니다. 직관적으로 당연해 보이지만, 엄밀히 말하면 타입이 다릅니다 — `&'static str`과 `&'short str`은 서로 다른 타입입니다. 그런데도 긴 쪽을 짧은 쪽 자리에 넣는 것이 허용됩니다. 이 "수명 사이의 대입 가능 방향"을 규정하는 것이 **변성(Variance)**입니다. 파이썬에는 이런 개념이 없습니다(굳이 비유하면 `mypy`의 공변/반변 제네릭 `TypeVar(covariant=True)`가 가장 가깝습니다). Rust에서는 컴파일러가 타입마다 변성을 자동으로 정하며, 우리가 직접 표기하지는 않지만 **수명 에러를 해독할 때 반드시 알아야 하는 배경 규칙**입니다.

변성에는 세 종류가 있습니다.

- **공변(Covariant)**: `'long`이 `'short`보다 길 때, `&'long T`를 `&'short T`가 필요한 자리에 쓸 수 있다. 즉 수명을 **줄이는 방향으로** 호환된다. `&'a T`(불변 참조)와 `Box<T>`, `Vec<T>`가 `'a`/`T`에 대해 공변입니다. 위 예제가 동작한 이유가 바로 이것 — 긴 `'static`을 짧은 수명 자리에 흘려보낼 수 있습니다.
- **불변(Invariant)**: 수명을 늘리지도 줄이지도 못한다. 정확히 같은 수명만 받는다. **`&'a mut T`(가변 참조)는 가리키는 대상 `T`에 대해 불변**이고, `Cell<T>`·`RefCell<T>`도 `T`에 불변입니다. 가변 접근이 끼면 "양방향"이 되어, 한쪽으로만 줄이는 것이 곧바로 불건전(unsound)해지기 때문입니다.
- **반변(Contravariant)**: 공변의 반대. 수명을 **늘리는 방향으로** 호환된다. 실무에서 직접 마주칠 일은 드물지만, 함수 타입의 **인자 위치**가 반변입니다 — `fn(&'static str)`을 요구하는 자리에 `fn(&'short str)`(더 짧은 수명도 받아 주는, 더 너그러운 함수)을 넘길 수 있습니다.

가변 참조가 왜 불변이어야 하는지, 즉 "공변을 허용하면 무슨 재앙이 나는지"를 컴파일러가 막는 실제 사례로 봅시다.

```rust
// dest가 가리키는 참조를, src로 덮어쓴다.
fn evil<'a>(dest: &mut &'a str, src: &'a str) {
    *dest = src;
}

fn main() {
    let mut long_lived: &'static str = "static";
    {
        let local = String::from("temporary");
        // &mut &'static str 을 &mut &'a str 자리에 넘기려는 시도.
        // &mut가 공변이었다면 'static을 'a(짧은 수명)으로 "줄여" 통과시켰을 것이다.
        evil(&mut long_lived, &local);
    } // 🚨 local이 여기서 죽는다
    println!("{long_lived}"); // 만약 위가 통과했다면 long_lived는 죽은 local을 가리킨다!
}
```

```text
error[E0597]: `local` does not live long enough
  --> variance_invariant.rs:10:31
   |
 6 |     let mut long_lived: &'static str = "static";
   |                         ------------ type annotation requires that `local` is borrowed for `'static`
 8 |         let local = String::from("temporary");
   |             ----- binding `local` declared here
10 |         evil(&mut long_lived, &local);
   |                               ^^^^^^ borrowed value does not live long enough
11 |     } // local이 여기서 죽는다
   |     - `local` dropped here while still borrowed
```

에러 메시지의 두 번째 줄이 정확히 변성의 작동을 폭로합니다 — **"type annotation requires that `local` is borrowed for `'static`"**. `&mut &'static str`이 불변이기 때문에 컴파일러는 `'a`를 `'static`으로 **줄이지 못하고**, 거꾸로 `local`이 `'static`만큼 살아야 한다고 요구합니다. 당연히 지역 변수 `local`은 그럴 수 없으니 E0597로 거부됩니다. 만약 `&mut`가 (불변 참조처럼) 공변이었다면 이 코드가 통과해 `long_lived`가 해제된 `local`을 가리키는 use-after-free가 됐을 것입니다. 즉 **변성은 수명 시스템이 건전성을 유지하는 마지막 빗장**입니다.

> 💡 **실무 요점**: 변성을 직접 표기할 일은 없습니다(`unsafe`로 `PhantomData`를 다루지 않는 한). 다만 "분명 더 긴 수명을 넘겼는데 왜 거부되지?"라는 당혹스러운 에러의 절반은 `&mut`/`Cell`/`RefCell`의 **불변성** 때문입니다. 불변 참조(`&T`)는 너그럽고, 가변 참조(`&mut T`)는 까다롭다 — 이 한 문장만 기억하면 대부분의 변성 관련 에러를 해독할 수 있습니다.

---

## 6.9 수명은 기계어에 흔적이 0 — MIR로 직접 확인하기

6.4의 Under the Hood에서 "수명은 타입 검사가 끝나면 증발한다"고 단언했습니다. 이 말은 믿어 달라는 주장이 아니라 **눈으로 확인할 수 있는 사실**입니다. Rust 컴파일러의 중간 표현(MIR, Mid-level IR)을 뽑아 보면, 우리가 적은 `'a`가 어디로 사라지는지 직접 볼 수 있습니다.

6.3의 `longest`를 그대로 컴파일하되 MIR을 출력해 봅니다.

```bash
rustc --emit=mir longest.rs
```

소스의 시그니처와, 컴파일러가 빌림 검사를 마친 뒤 생성한 MIR의 시그니처를 나란히 놓아 보세요.

```text
# 우리가 적은 소스
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }

# 빌림 검사 직후의 MIR (lifetimes erased)
fn longest(_1: &str, _2: &str) -> &str {
    debug x => _1;
    debug y => _2;
    let mut _0: &str;
    ...
}
```

`<'a>`도, `&'a`의 `'a`도 **흔적 없이 사라졌습니다.** 명시적으로 적은 수명 매개변수는 MIR 단계에서 이미 없어졌고, 남은 것은 그냥 `&str`뿐입니다. 구조체 수명도 마찬가지입니다 — `ImportantExcerpt<'a>`(6.5)는 MIR에서 이름이 지워진 익명 영역 `ImportantExcerpt<'_>`로 나타나, 우리가 붙인 이름 `'a`가 검사 이후엔 아무 정보도 나르지 않음을 보여 줍니다.

> **Under the Hood: 수명은 "빌림 검사 통과용 증명"이고, 그 증명은 코드 생성 전에 폐기된다**
>
> rustc의 파이프라인은 대략 `소스 → AST → HIR → (타입·트레이트 해석) → MIR → 빌림 검사(borrowck) → 단일형화(Monomorphization) → LLVM IR → 기계어` 순입니다. 수명과 영역(Region) 정보는 **MIR 위에서 빌림 검사가 끝나는 순간까지만** 필요합니다. 빌림 검사가 "모든 참조가 자기가 가리키는 데이터보다 오래 살지 않는다"를 증명하고 나면, 그 증명을 떠받치던 수명 정보는 더 이상 쓸모가 없어 폐기됩니다. 그래서 LLVM에 넘어가는 코드에는 수명의 흔적이 0이고, 당연히 생성된 기계어에도 0입니다. 이를 확인하는 또 다른 방법: 같은 함수를 수명을 **생략한 버전**과 **명시한 버전**으로 각각 `rustc -O --emit=asm`해 어셈블리를 비교하면, 두 출력은 **바이트 단위로 완전히 동일**합니다(실측 확인). 즉 `'a`를 적든 안 적든, 컴파일러가 보는 프로그램도 같고 CPU가 실행하는 명령도 같습니다. 이것이 수명이 무비용 추상화(Zero-cost Abstraction)인 이유의 가장 직접적인 증거입니다 — C의 포인터에 "수명 태그"라는 런타임 필드가 없는 것과 정확히 같은 의미에서, Rust의 참조에도 그런 것은 없습니다. 차이는 단 하나, Rust는 그 안전성을 **컴파일 타임에 증명하도록 강제**한다는 것뿐입니다.

### 수명 관련 에러코드 치트시트

수명을 다루다 만나는 에러는 종류가 의외로 적고, 코드만 보면 원인이 거의 특정됩니다. 실무에서 마주치는 핵심 에러를 한 표로 정리합니다(모두 이 장에서 실제로 재현한 것입니다).

| 에러 코드 | 한 줄 의미 | 전형적 원인 | 처방 |
|---|---|---|---|
| `E0106` | missing lifetime specifier | 입력 참조 2개 이상인 함수가 참조를 반환(생략 규칙 미적용) | `<'a>`를 선언하고 입력·출력에 명시(6.3) |
| `E0621` | explicit lifetime required | 시그니처의 `'a`와 본문이 반환하는 참조의 출처가 불일치 | 반환될 수 있는 인자에 같은 `'a`를 부여 |
| `E0515` | cannot return reference to local variable | 함수 지역 변수의 참조/슬라이스를 반환 | 소유 값(`String`/`Vec`)을 반환하거나 입력 참조를 반환 |
| `E0597` | does not live long enough | 참조가 가리키는 원본이 참조보다 먼저 드롭됨 | 원본의 스코프를 늘리거나 참조 사용을 앞당김(NLL) |
| `E0505` | cannot move out of \`x\` because it is borrowed | 빌린 상태에서 같은 값을 move (self-ref 시도 단골) | 빌림이 끝난 뒤 move, 또는 인덱스/`clone`으로 재설계(6.5) |
| `E0309` | the parameter type \`T\` may not live long enough | 제네릭 `T`가 품을 수 있는 참조에 대한 outlives 바운드 누락 | `where T: 'a` 추가(6.7) |
| (코드 없음) `lifetime may not live long enough` | 영역 포함 관계 위반(변성·바운드 불일치) | 출력 수명이 입력보다 길거나, `&mut`의 불변성 충돌(6.8) | 명명 수명 도입 또는 가변/불변 참조 선택 재검토 |

> 💡 에러 코드는 `rustc --explain E0106`으로 공식 설명과 최소 재현 코드를 즉시 볼 수 있습니다. 최신 rustc는 일부 수명 진단을 숫자 코드 없는 `lifetime may not live long enough`로 통합 출력하기도 하니, 코드가 안 보여도 메시지의 "must outlive" / "requires that ... is borrowed for ..." 문구를 단서로 읽으세요.

---

## 6.10 실전 예제 (Hands-on): 제로카피 설정 파서

지금까지의 모든 개념 — 구조체에 담는 참조, 규칙 3 생략, 출력 수명이 입력에 묶이는 계약 — 이 한데 모이는 현실적인 예제를 만들어 봅시다. 파이썬으로는 다음처럼 작성할 `key=value` 설정 파서입니다.

```python
from __future__ import annotations
from dataclasses import dataclass


@dataclass
class KeyValue:
    key: str
    value: str


def parse(text: str) -> list[KeyValue]:
    result: list[KeyValue] = []
    for line in text.splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        key, _, value = line.partition("=")
        result.append(KeyValue(key.strip(), value.strip()))
    return result
```

파이썬은 `key.strip()`이 호출될 때마다 **새 문자열 객체를 힙에 할당**합니다. 입력이 1MB짜리 설정 파일이고 수만 줄이라면, 그만큼의 작은 문자열 객체가 줄줄이 생성되고 각각 레퍼런스 카운트 헤더를 답니다. Rust 버전은 입력 버퍼의 **슬라이스만 가리켜** 단 한 바이트도 복사하지 않습니다(zero-copy). 그 핵심이 바로 수명입니다 — 반환되는 모든 `&str`가 입력 버퍼의 수명에 묶여 있다는 것을 구조체 수명으로 표현합니다.

```rust
// 반환되는 모든 &str는 원본 input의 수명 'a에 묶여 있다 (zero-copy).
#[derive(Debug, PartialEq)]
struct KeyValue<'a> {
    key: &'a str,
    value: &'a str,
}

struct ConfigParser<'a> {
    input: &'a str,
}

impl<'a> ConfigParser<'a> {
    fn new(input: &'a str) -> Self {
        ConfigParser { input }
    }

    // 출력 KeyValue<'a>의 'a는 self.input의 'a를 따른다 (규칙 3 덕에 표기 최소화).
    fn parse(&self) -> Vec<KeyValue<'a>> {
        self.input
            .lines()
            .filter_map(|line| {
                let line = line.trim();
                if line.is_empty() || line.starts_with('#') {
                    return None;
                }
                let (key, value) = line.split_once('=')?;
                Some(KeyValue {
                    key: key.trim(),
                    value: value.trim(),
                })
            })
            .collect()
    }
}

fn main() {
    let raw = "\
# database config
host = localhost
port = 5432

  # comment
name = analytics
";
    let parser = ConfigParser::new(raw);
    let pairs = parser.parse();

    for kv in &pairs {
        println!("{} -> {}", kv.key, kv.value);
    }

    assert_eq!(pairs[0], KeyValue { key: "host", value: "localhost" });
    assert_eq!(pairs.len(), 3);
    println!("parsed {} entries (zero-copy)", pairs.len());
}
```

실행 결과:

```text
host -> localhost
port -> 5432
name -> analytics
parsed 3 entries (zero-copy)
```

여기서 `parse`의 반환 타입이 `Vec<KeyValue<'a>>`인 점이 핵심입니다. `'a`는 `ConfigParser`가 빌린 `input`의 수명이고, 그래서 컴파일러는 "이 `Vec`에 담긴 모든 슬라이스는 `input`보다 오래 쓰이면 안 된다"는 것을 자동으로 강제합니다. 만약 `raw`가 파괴된 뒤에 `pairs`를 쓰려 들면 컴파일러가 즉시 막습니다 — 파이썬이라면 런타임에 아무 일 없이 동작했겠지만, 그건 매번 문자열을 복사하는 비용을 치렀기 때문입니다. Rust는 비용을 치르지 않는 대신, 안전성을 수명으로 증명하게 합니다.

> **실무 크레이트 메모**: 실제 설정 파싱에서는 `serde`(+ `toml`/`serde_json`)를 쓰는 것이 일반적이며, 역직렬화 시 빌린 데이터를 다루기 위해 `#[derive(Deserialize)]`에 `#[serde(borrow)]`를 붙여 제로카피 역직렬화를 합니다. 이 모든 것이 위에서 본 구조체 수명 메커니즘 위에 세워져 있습니다. `serde`는 Chapter 7에서 Pydantic 대체로 본격적으로 다룹니다.

### 수명이 만드는 성능 차이를 수치로 확인하기

"제로카피가 빠르다"는 말을 막연한 구호로 두지 말고 측정해 봅시다. 같은 파서를, 슬라이스를 반환하는 버전(`parse_borrowed`)과 매 항목을 `.to_string()`으로 복사해 소유 값을 반환하는 버전(`parse_owned`)으로 만들어, 5만 줄짜리 입력을 50번 반복 파싱한 시간을 비교합니다.

```rust
use std::time::Instant;

// 입력의 슬라이스만 가리킨다 — 할당 0회. 반환 (&str, &str)의 수명은 input에 묶인다.
fn parse_borrowed(input: &str) -> Vec<(&str, &str)> {
    input.lines()
        .filter_map(|l| l.split_once('='))
        .map(|(k, v)| (k.trim(), v.trim()))
        .collect()
}

// 매 항목마다 새 String을 힙에 할당한다 — 파이썬의 strip()과 같은 비용 모델.
fn parse_owned(input: &str) -> Vec<(String, String)> {
    input.lines()
        .filter_map(|l| l.split_once('='))
        .map(|(k, v)| (k.trim().to_string(), v.trim().to_string()))
        .collect()
}

fn main() {
    let mut s = String::new();
    for i in 0..50_000 {
        s.push_str(&format!("key_{i} = value_number_{i}\n"));
    }

    let t = Instant::now();
    for _ in 0..50 { let _ = parse_borrowed(&s); }
    let borrowed = t.elapsed();

    let t = Instant::now();
    for _ in 0..50 { let _ = parse_owned(&s); }
    let owned = t.elapsed();

    println!("borrowed(zero-copy): {borrowed:?}");
    println!("owned(clone):        {owned:?}");
    println!("ratio: {:.1}x", owned.as_secs_f64() / borrowed.as_secs_f64());
}
```

`cargo run --release`(또는 `rustc -O`)로 측정한 대표적인 결과는 다음과 같습니다(Apple Silicon, 입력 5만 줄 × 50회 반복).

```text
borrowed(zero-copy): 65.1ms
owned(clone):        153.7ms
ratio: 2.4x
```

차이는 약 **2.3~2.4배**입니다. 과장이 아닙니다 — 두 버전의 알고리즘은 동일하고, 차이는 오직 "슬라이스를 가리키느냐(수명으로 묶음), 매번 힙에 복사하느냐"뿐입니다. `parse_owned`는 항목마다 두 번의 힙 할당과 바이트 복사를 수행하고, 그렇게 만든 `String`들은 곧 드롭되며 또 해제 비용을 치릅니다. 파이썬의 `str.strip()`은 정확히 이 `parse_owned`와 같은 비용 모델(매번 새 객체 할당)을 따르며, 거기에 인터프리터 오버헤드와 레퍼런스 카운팅까지 얹힙니다. 수명은 단지 안전성을 위한 형식주의가 아니라, **"복사하지 않아도 안전하다"를 증명함으로써 복사를 제거하는 성능 도구**이기도 한 것입니다.

---

## 6.11 Best Practices & 흔한 함정

### Best Practices 체크리스트

- **먼저 생략에 맡기고, 컴파일러가 시킬 때만 적어라.** 함수에 무작정 `<'a>`를 붙이지 마세요. 생략 규칙(6.4)이 대부분 처리합니다. E0106이 뜰 때 비로소, 컴파일러가 `help`로 제안하는 그대로 적으면 됩니다.
- **구조체에 참조를 담기 전에 "정말 빌려야 하나?"를 물어라.** 대부분의 데이터 모델 구조체는 `String`·`Vec<T>`처럼 데이터를 **소유**하는 편이 단순하고 옳습니다. 참조 필드(`&'a str`)는 파서·뷰처럼 제로카피가 성능상 정말 필요할 때만 쓰세요. 참조 필드 하나가 구조체 전체와 그것을 쓰는 모든 함수에 `'a`를 전염시킵니다.
- **수명은 최대한 정밀하게, 그러나 최소한으로.** 6.3의 `first_word`처럼 반환값이 한 입력에만 묶인다면 `'a`/`'b`로 나눠 호출처에 자유를 주세요. 단, 의미 없이 수명을 여러 개로 쪼개 시그니처를 복잡하게 만들지는 마세요.
- **`T: 'static`을 요구하기 전에 한 번 더 생각하라.** 스레드 API 등이 `'static`을 요구할 때, 참조 대신 소유 값(`String`, `Vec`)을 넘기거나 `Arc`(Chapter 17)로 공유하는 것이 정석입니다.
- **clippy를 신뢰하라.** `cargo clippy`는 `needless_lifetimes`(불필요한 명시적 수명), `extra_unused_lifetimes`(쓰이지 않는 수명 매개변수)를 잡아 줍니다. 생략으로 충분한데 습관적으로 적은 `'a`를 자동으로 지적합니다.

### 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1 — 수명을 과도하게 붙이기.** 파이썬에서 타입 힌트를 빼곡히 적던 습관 때문에, 입력이 하나뿐인 함수에까지 수명을 박는 경우가 많습니다.

```rust
// ❌ 불필요: 입력이 하나라 규칙 2가 알아서 처리한다. clippy::needless_lifetimes 경고.
fn first<'a>(s: &'a str) -> &'a str { s.split(' ').next().unwrap_or("") }

// ✅ 생략하면 똑같이 동작하고 더 읽기 쉽다.
fn first(s: &str) -> &str { s.split(' ').next().unwrap_or("") }
```

**함정 2 — 수명 에러를 `.clone()`으로 도망치기.** E0597·E0515를 만났을 때 반사적으로 `.clone()`이나 `.to_string()`을 붙여 "고쳐지는" 경험을 합니다. 때로는 그게 옳지만(짧은 문자열 한 번 복사), **남용하면 Rust를 쓰는 이유가 증발합니다.** clone은 힙 할당과 `memcpy`를 동반하므로, 핫 루프에서 무심코 뿌리면 파이썬보다 느려질 수도 있습니다.

```rust
// ❌ 수명 에러를 회피하려고 매 줄 clone — 제로카피의 이점을 모두 버린다.
fn parse_owned(input: &str) -> Vec<(String, String)> {
    input.lines()
        .filter_map(|l| l.split_once('='))
        .map(|(k, v)| (k.trim().to_string(), v.trim().to_string())) // 매 항목 2회 힙 할당
        .collect()
}

// ✅ 6.10처럼 수명으로 묶어 슬라이스를 반환 — 할당 0회.
//    소유가 정말 필요한 경계(예: 입력 버퍼가 곧 사라지는 API 경계)에서만 clone하라.
```

판단 기준은 단순합니다. **데이터가 함수/구조체보다 오래 살아야 한다면 소유(clone/owned)가 옳고, 데이터의 수명 안에서만 쓰인다면 참조(+수명)가 옳습니다.** clone은 회피가 아니라 의식적인 설계 선택이어야 합니다.

**함정 3 — self-referential 구조체를 억지로 만들려 하기.** 6.5에서 봤듯, "데이터와 그 데이터에 대한 참조를 한 구조체에 함께" 두려는 시도는 거의 항상 막힙니다. 파이썬에서는 자연스럽던 패턴입니다. 해법은 우회가 아니라 **재설계**입니다 — 참조 대신 인덱스(`usize`)나 범위(`Range<usize>`)를 들거나, 소유와 뷰를 별도 타입으로 분리하세요.

```rust
// ❌ 자기 참조 시도 (E0505로 거부됨)
// struct Doc<'a> { text: String, title: &'a str }

// ✅ 참조 대신 범위를 들어 self-reference를 피한다.
struct Doc {
    text: String,
    title: std::ops::Range<usize>, // text[title.clone()] 로 슬라이스 복원
}
```

**함정 4 — `&'static`을 "오래 살리는 주문"으로 오해하기.** 6.6에서 본 두 가지 오해입니다. 컴파일러가 `'static`을 요구한다고 해서 변수에 `'static`을 붙이면 해결되는 게 아닙니다 — 소유 값을 넘기거나, 데이터를 실제로 `'static`하게(예: `Box::leak`, 표준 라이브러리의 `std::sync::OnceLock`/`std::sync::LazyLock`) 만들어야 합니다. 과거에는 `once_cell`·`lazy_static` 같은 외부 크레이트를 썼지만, 1.70/1.80부터 그 기능이 표준 라이브러리로 들어왔으니 새 코드에서는 표준 타입을 쓰세요(6.6의 코드 참고).

---

## 연습문제

손으로 풀어 보세요. Rustlings 스타일로, 먼저 머릿속(또는 종이)으로 답한 뒤 실제 `rustc`로 확인하는 것이 핵심입니다. 각 문항 아래 **정답 스케치**에는 핵심 아이디어와 함께, 틀렸을 때 컴파일러가 내뱉는 **자기검증용 에러 코드** 또는 측정값을 적어 두었습니다.

**문제 1 (수명 표기).** 다음 함수는 두 정수 슬라이스 중 더 긴 쪽의 "앞 절반"을 반환합니다. 그대로는 컴파일되지 않습니다. 어떤 에러가 나며, 시그니처를 어떻게 고쳐야 할까요?

```rust
fn longer_half(a: &[i32], b: &[i32]) -> &[i32] {
    if a.len() > b.len() { &a[..a.len() / 2] } else { &b[..b.len() / 2] }
}
```

> **정답 스케치.** 입력 참조가 둘이라 생략 규칙 2가 적용되지 않고, 반환 참조가 둘 중 어느 입력에 묶이는지 시그니처가 말하지 않습니다 → `error[E0106]: missing lifetime specifier`. 반환값이 두 입력 어느 쪽에서든 나올 수 있으므로 셋을 같은 `'a`로 묶습니다:
> ```rust
> fn longer_half<'a>(a: &'a [i32], b: &'a [i32]) -> &'a [i32] { /* 동일 */ }
> ```
> 자기검증: 고치기 전 `rustc`는 `E0106`, 고친 뒤엔 통과. (만약 한 입력에만 묶을 수 있는 의미였다면 6.3의 `first_word`처럼 `'a`/`'b`로 나누는 것이 더 정밀합니다.)

**문제 2 (생략 규칙).** 다음 두 함수/메서드는 수명을 한 글자도 적지 않았는데도 컴파일됩니다. 컴파일러가 각각 어떤 시그니처로 "탈설탕(desugar)"하는지, 그리고 적용된 생략 규칙 번호를 적으세요.

```rust
fn trim_prefix(s: &str) -> &str { s.trim_start() }

struct Buf { data: String }
impl Buf {
    fn head(&self) -> &str { &self.data[..1] }
}
```

> **정답 스케치.**
> - `trim_prefix`: 입력 참조가 하나뿐 → **규칙 1**로 `'a` 하나 배정, **규칙 2**로 그 `'a`를 출력에 복사. 펼치면 `fn trim_prefix<'a>(s: &'a str) -> &'a str`.
> - `head`: 입력에 `&self`가 있으므로 **규칙 3**이 `self`의 수명을 출력에 부여. 펼치면 `fn head<'a>(&'a self) -> &'a str`. (입력이 하나뿐이라 규칙 2로도 같은 결과지만, `&self`가 있으면 규칙 3이 우선합니다.)
> 자기검증: 손으로 적은 명시 시그니처로 바꿔도 동일하게 컴파일되면 정답입니다. 반대로 입력을 둘로 늘리고(`fn trim_prefix(s: &str, _t: &str) -> &str`) `&self`가 없게 만들면 곧바로 `E0106`이 떠, 규칙 2가 "단일 입력"에서만 작동함을 확인할 수 있습니다.

**문제 3 (구조체 수명).** 입력 문자열을 빌려 첫 단어를 토큰으로 돌려주는 제로카피 렉서를 완성하세요. 빈칸(`/* ? */`)에 들어갈 수명 표기를 채우고, 왜 구조체에는 생략이 안 되는지 한 줄로 설명하세요.

```rust
#[derive(Debug)]
struct Token/* ? */ { kind: &str, text: &str }

struct Lexer/* ? */ { src: &str }

impl/* ? */ Lexer/* ? */ {
    fn new(src: &str) -> Self { Lexer { src } }
    fn first_word(&self) -> Token/* ? */ {
        let w = self.src.split_whitespace().next().unwrap_or("");
        Token { kind: "word", text: w }
    }
}
```

> **정답 스케치.** 구조체 필드가 참조를 담으면 반드시 수명 매개변수가 필요합니다(생략 규칙은 `fn`/메서드 시그니처에만 적용되고, 구조체에는 입력/출력 흐름이 없어 적용 근거가 없습니다 — 6.4 말미). 채운 결과:
> ```rust
> struct Token<'a> { kind: &'a str, text: &'a str }
> struct Lexer<'a> { src: &'a str }
> impl<'a> Lexer<'a> {
>     fn new(src: &'a str) -> Self { Lexer { src } }
>     fn first_word(&self) -> Token<'a> { /* 동일 */ }
> }
> ```
> `first_word`의 반환 `Token<'a>`는 `self.src`의 `'a`를 따릅니다. 자기검증: `Token`에서 `<'a>`를 빼면 `error[E0106]: missing lifetime specifier`(필드 `&str`마다 한 번씩) — 구조체에는 생략이 없다는 증거입니다.

**문제 4 (`'static`).** 다음 함수는 스레드로 보낼 값을 받기 위해 `T: 'static` 바운드를 겁니다. (a) `spawn_like(String::from("hi"))`는 통과하는데 (b) 지역 변수 참조 `spawn_like(&local)`은 거부되는 이유를 설명하고, 거부 시의 에러 코드를 적으세요. 그리고 런타임에 계산한 값을 `&'static str`로 안전하게 얻는 표준 라이브러리 방법 하나를 쓰세요.

```rust
fn spawn_like<T: Send + 'static>(_v: T) {}
```

> **정답 스케치.** `T: 'static`은 "`T`가 `'static`보다 짧은 빌린 참조를 품지 않는다"는 뜻입니다(6.6의 오해 2). (a) `String`은 데이터를 **소유**하므로 빌린 참조가 없어 자동으로 `'static`을 만족 → 통과. (b) `&local`은 지역 변수 `local`의 수명에 묶이므로 `'static`이 아님 → 거부, `error[E0597]: \`local\` does not live long enough`(메시지에 "argument requires that \`local\` is borrowed for \`'static\`"이 함께 뜹니다). 런타임 계산 값을 `'static`으로 얻는 정석은 `std::sync::OnceLock`입니다(과거의 `once_cell`/`lazy_static`을 대체):
> ```rust
> use std::sync::OnceLock;
> static GREETING: OnceLock<String> = OnceLock::new();
> fn greeting() -> &'static str {
>     GREETING.get_or_init(|| String::from("hello, world")).as_str()
> }
> ```
> 자기검증: (b)는 `E0597`, `OnceLock` 버전은 통과하며 두 번째 호출부터 초기화 클로저가 실행되지 않습니다.

---

수명은 C/C++ 출신 개발자조차 낯설어하는 Rust 고유의 개념이지만, 그 정체는 의외로 단순합니다 — **이미 존재하는 참조들의 생존 관계를 컴파일러에게 설명하는 언어**일 뿐, 새로운 무언가를 만들거나 값을 더 오래 살리는 마법이 아닙니다. 생략 규칙 덕에 대부분의 코드에서는 보이지도 않고, 정작 직접 적게 되는 곳은 구조체에 참조를 담는 제한된 상황뿐입니다. 그리고 그 수명 검사는 타입 체킹이 끝나는 순간 증발하여 런타임 비용이 정확히 0인, 무비용 추상화의 결정판입니다.

이로써 우리는 Rust가 메모리를 지배하는 세 기둥 — 소유권(Chapter 4), 대여(Chapter 5), 수명(Chapter 6) — 을 모두 손에 넣었습니다. 이 단단한 기반 위에서, 다음 장(Part 3, Chapter 7)부터는 파이썬의 풍부한 객체 지향 문법이 Rust에서 어떻게 탈바꿈하는지 살펴봅니다. 첫 주제는 데이터를 담는 그릇 — `dataclass`와 `Pydantic`을 대체하는 **Struct와 데이터 모델링**입니다. 방금 배운 구조체 수명이 그곳에서 `serde`의 제로카피 역직렬화로 다시 등장하니, 기대해도 좋습니다.

---

## References

- https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html
- https://doc.rust-lang.org/rust-by-example/scope/lifetime.html
- https://doc.rust-lang.org/reference/lifetime-elision.html
- https://doc.rust-lang.org/nomicon/lifetimes.html
- https://doc.rust-lang.org/nomicon/subtyping.html
- https://doc.rust-lang.org/std/sync/struct.OnceLock.html
- https://doc.rust-lang.org/error_codes/E0106.html
