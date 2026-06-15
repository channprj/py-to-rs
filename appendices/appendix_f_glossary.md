# 부록 F. 용어집 (Glossary)

이 책은 22개 장에 걸쳐 Rust의 핵심 개념을 Python 개발자의 멘탈 모델과 끊임없이 대조하며 전개합니다. 그 과정에서 같은 단어가 장마다 조금씩 다른 결로 등장하고, "소유권"과 "대여"와 "수명"처럼 서로 얽혀 정의되는 용어들이 한꺼번에 쏟아집니다. 10년 차 Python 엔지니어가 Rust에 입문할 때 무너지는 지점은 문법이 아니라 바로 이 **용어들이 가리키는 멘탈 모델의 누적된 무게**입니다. `&`가 "참조"인지 "대여"인지, `Arc`가 Python의 refcount와 무엇이 다른지, `dyn`이 왜 fat pointer인지를 매번 본문으로 되돌아가 찾는 것은 비효율적입니다.

이 부록은 그 비효율을 없애기 위한 **단일 진실 원천(single source of truth)**입니다. 책 전체에서 쓰이는 핵심 용어를 한곳에 모아, 각 항목마다 (1) 한 줄 정의, (2) 그 용어가 처음으로 깊이 있게 다뤄지는 장으로 가는 딥링크, (3) "Python 멘탈 모델과 어떻게 다른가"를 한 줄로 못 박았습니다. 본문을 읽다 막히면 여기로 돌아와 좌표를 확인하고, 다시 본문으로 점프하면 됩니다.

용어집은 사전이지 교재가 아닙니다. 따라서 도입부 이후로는 개념을 처음부터 가르치지 않습니다. 대신 **이미 배운 것을 압축해 다시 꺼내 쓰도록** 설계했습니다. 정의는 짧고, 깊이가 필요하면 딥링크가 해당 장으로 안내합니다. 주제별로 묶되(소유권·타입·동시성·에러·매크로·성능), 마지막에 **영문↔한글 양방향 색인 표**를 두어 어느 방향에서든 탐색이 되도록 했습니다. 책의 표기 규약 — `한국어(English)` 병기, 💡 Under the Hood / 🚨 No Free Lunch 박스, 벤치 측정 환경 표기 — 도 이 부록 안에서 다시 한번 명시합니다.

마지막으로 한 가지 부탁이 있습니다. 정의에 적힌 바이트 수치(`size_of`, `getsizeof`)는 전부 **이 책의 표준 측정 환경에서 직접 측정한 값**입니다. 인터넷에 떠도는 수치는 버전에 따라 다릅니다. 이 부록의 모든 숫자는 의심해도 좋되, 의심한다면 직접 측정해 확인하십시오 — 그것이 이 책 전체를 관통하는 태도입니다.

---

## F.1 표기 규약 (Notation Conventions)

본문 전체에서 일관되게 쓰는 표기를 한곳에 모읍니다. 이 규약은 모든 장과 부록에 적용됩니다.

**용어 병기.** 기술 용어는 처음 등장할 때 `한국어(English)` 형식으로 병기합니다. 예: **소유권(Ownership)**, **단일형화(Monomorphization)**, **무비용 추상화(Zero-cost Abstraction)**. 번역어는 책 전반에서 고정합니다. 특히 다음 네 가지는 흔들리지 않습니다.

| 영문 | 이 책의 고정 번역어 | 쓰지 않는 표현 |
|------|--------------------|----------------|
| Ownership | 소유권 | — |
| Borrowing | 대여 | "빌림"(혼용 금지) |
| Reference | 참조 | "레퍼런스" |
| Lifetime | 수명 | "라이프타임", "생명주기" |
| Trait | 트레이트 | "특성", "성질" |
| Monomorphization | 단일형화 | "단형화" |
| Zero-cost Abstraction | 무비용 추상화 | "제로코스트" |
| Zero-copy | 제로카피 | "무복사" |

**인용 박스 두 종류.** 본문은 두 가지 강조 박스를 씁니다.

> 💡 **Under the Hood**: 메모리 레이아웃(바이트 단위), 디스패치 방식(static/dynamic), 또는 디스어셈블리 관점에서 "내부가 실제로 어떻게 동작하는가"를 파고드는 박스입니다. 추측이 아니라 측정·관찰 가능한 사실을 담습니다.

> 🚨 **No Free Lunch**: "공짜는 없다." 어떤 선택이 가져오는 트레이드오프, 숨은 비용, 또는 흔히 오해하는 함정을 경고하는 박스입니다. Rust가 항상 옳다는 식의 마케팅을 경계하고 균형을 잡습니다.

**코드 언어 태그.** 코드 블록은 언어를 정확히 표기합니다: ` ```python `, ` ```rust `, ` ```toml `, ` ```bash `, ` ```text `. 컴파일되지 않거나 패닉하는 코드는 `// 🚨 컴파일 에러!` 또는 `// panic!` 주석으로 명시합니다.

**버전 핀.** 이 책의 모든 코드와 수치는 다음 환경에 고정됩니다.

| 항목 | 버전 |
|------|------|
| Rust | **1.96.0** (edition 2021) |
| Python | **3.13.6** (CPython) |
| 측정 머신 | Apple Silicon (AArch64), `--release` 빌드 |

**벤치 측정 환경 표기.** 성능 수치를 인용할 때는 항상 (1) 입력 크기, (2) 빌드 모드(`--release` 여부), (3) `black_box`/`criterion` 사용 여부를 함께 명시합니다. 측정 맥락이 없는 "X배 빠름"은 신뢰하지 않습니다. 정직한 범위는 **보통 2~50배, CPU 바운드 핫루프에서 100배+**입니다([Chapter 1](../chapters/chapter_01.md)).

---

## F.2 소유권·메모리 (Ownership & Memory)

Rust의 심장입니다. 이 군집의 용어들은 [Chapter 4](../chapters/chapter_04.md)~[Chapter 6](../chapters/chapter_06.md)에서 집중적으로 다뤄집니다. Python 개발자가 가장 먼저, 가장 세게 부딪히는 벽이기도 합니다.

**소유권(Ownership)** — 모든 값은 단 하나의 소유자(변수)를 가지며, 소유자가 스코프를 벗어나면 값은 그 즉시 결정론적으로 해제된다는 규칙. 최초 상세 등장 [Chapter 4](../chapters/chapter_04.md). · *Python 멘탈모델 대비*: Python은 GC와 참조 카운팅이 수명을 대신 관리하지만, Rust는 컴파일 타임에 "이 데이터의 주인은 누구인가"를 개발자가 직접 답하게 합니다.

**이동(Move)** — 힙 데이터를 가진 값을 대입·전달할 때 소유권이 옮겨가고, 원래 변수는 무효화되어 더는 쓸 수 없게 되는 동작. 최초 상세 등장 [Chapter 4](../chapters/chapter_04.md). · *Python 멘탈모델 대비*: Python의 `b = a`는 같은 객체에 라벨을 하나 더 붙일 뿐 둘 다 멀쩡히 쓰지만, Rust에서 `let b = a`는 `a`를 무효화합니다(`error[E0382]: borrow of moved value`).

**대여(Borrowing)** — 소유권을 넘기지 않고 값을 참조로 잠시 빌려 쓰는 행위. 불변 대여(`&T`)와 가변 대여(`&mut T`)가 있다. 최초 상세 등장 [Chapter 5](../chapters/chapter_05.md). · *Python 멘탈모델 대비*: Python에는 "빌린다"는 개념 자체가 없습니다. 모든 전달이 사실상 참조 공유이고, 누가 그것을 수정해도 막지 않습니다.

**참조(Reference)** — 값의 메모리 주소를 가리키는 포인터. `&T`(불변)와 `&mut T`(가변)로 표기하며, Rust 참조는 **항상 유효한 주소를 가리킨다**(null 불가)는 불변식을 컴파일러가 보장한다. 최초 상세 등장 [Chapter 5](../chapters/chapter_05.md). · *Python 멘탈모델 대비*: Python의 모든 변수가 사실상 참조지만 그것이 null(`None`)일 수 있고 댕글링될 위험은 GC가 가립니다. Rust 참조는 null도 댕글링도 타입 시스템이 차단합니다.

**XOR 규칙(가변/불변 대여 배타, Aliasing XOR Mutability)** — 어떤 시점에든 한 값에 대해 **여러 개의 불변 참조** 또는 **단 하나의 가변 참조** 중 하나만 존재할 수 있다는 대여 검사기의 핵심 규칙. 공유와 변경을 배타적으로 강제해 데이터 레이스를 원천 차단한다. 최초 상세 등장 [Chapter 5](../chapters/chapter_05.md). · *Python 멘탈모델 대비*: Python은 한 리스트를 여러 곳에서 동시에 참조하며 아무나 수정할 수 있어, 이터레이션 중 변경 같은 버그가 런타임에야 터집니다. Rust는 이를 컴파일 타임에 거부합니다.

> 💡 **Under the Hood: XOR 규칙은 컴파일 타임에 어떻게 잡히는가**
>
> 불변 참조를 살려둔 채 같은 값을 가변으로 건드리면 `rustc`가 거부합니다. 실제 출력입니다(rustc 1.96.0).
>
> ```rust
> let mut v = vec![1, 2, 3];
> let first = &v[0]; // 불변 대여
> v.push(4);         // 🚨 가변 대여 시도 — XOR 규칙 위반
> println!("{first}");
> ```
>
> ```text
> error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
>  --> xor.rs:4:5
>   |
> 3 |     let first = &v[0]; // 불변 대여
>   |                  - immutable borrow occurs here
> 4 |     v.push(4); // 가변 대여 시도 — XOR 규칙 위반
>   |     ^^^^^^^^^ mutable borrow occurs here
> 5 |     println!("{first}"); // 불변 대여를 여기서 사용
>   |                ----- immutable borrow later used here
> ```
>
> 이것은 단순한 까다로움이 아닙니다. `Vec::push`는 용량이 부족하면 내부 버퍼를 재할당(realloc)하고 옛 버퍼를 해제합니다. 그러면 `first`가 가리키던 주소는 해제된 메모리(dangling)가 됩니다. C였다면 use-after-free, Python이었다면 (리스트 재할당은 안전하지만) 이터레이션 중 변경으로 인한 미묘한 논리 버그입니다. `error[E0502]`는 그 버그를 배포 전에 잡아낸 것입니다.

**수명(Lifetime)** — 참조가 유효한 코드 구간(스코프). 컴파일러가 "이 참조가 가리키는 값이 참조보다 먼저 사라지지 않는가"를 검증하기 위해 추적하는 정보로, `'a` 같은 표기로 명시한다. 최초 상세 등장 [Chapter 6](../chapters/chapter_06.md). · *Python 멘탈모델 대비*: Python은 객체가 참조되는 한 GC가 살려두므로 수명을 의식할 일이 없습니다. Rust는 댕글링 참조를 컴파일 타임에 막기 위해 수명을 명시적으로 다룹니다.

**RAII/Drop (Resource Acquisition Is Initialization)** — 값이 스코프를 벗어나는 순간 `Drop` 트레이트의 `drop`이 자동 호출되어 자원(메모리·파일·락)을 결정론적으로 해제하는 패턴. 최초 상세 등장 [Chapter 4](../chapters/chapter_04.md), 심화 [Chapter 20](../chapters/chapter_20.md). · *Python 멘탈모델 대비*: Python의 `__del__`은 GC 타이밍에 의존해 호출 시점이 불확실하지만(그래서 `with`/컨텍스트 매니저를 씁니다), Rust의 `drop`은 스코프 종료 시점에 결정론적으로 일어납니다.

> 💡 **Under the Hood: Drop은 생성의 역순으로 일어난다**
>
> 같은 스코프에서 `a`를 먼저, `b`를 나중에 만들면 해제는 **역순**(b → a)으로 일어납니다. 스택처럼 LIFO입니다.
>
> ```rust
> struct Guard(&'static str);
> impl Drop for Guard {
>     fn drop(&mut self) { println!("drop: {}", self.0); }
> }
> let _a = Guard("a (먼저 생성)");
> let _b = Guard("b (나중 생성)");
> // 출력: drop: b (나중 생성)  /  drop: a (먼저 생성)
> ```
>
> 이 결정론 덕분에 `MutexGuard`는 스코프를 벗어나면 자동으로 unlock되고(unlock을 깜빡할 수 없음, [Chapter 17](../chapters/chapter_17.md)), 파일 핸들은 자동으로 닫힙니다. C++의 소멸자와 같은 계열이며, Python의 `try/finally`나 `with`를 언어 차원에서 강제하는 셈입니다.

**스마트 포인터(Smart Pointer)** — 일반 참조를 넘어 소유권·참조 카운트·내부 가변성 같은 추가 의미를 부여하는 포인터 타입. `Box<T>`, `Rc<T>`, `Arc<T>`, `RefCell<T>` 등. 최초 상세 등장 [Chapter 20](../chapters/chapter_20.md). · *Python 멘탈모델 대비*: Python은 모든 객체가 사실상 refcount된 힙 포인터라 구분이 없지만, Rust는 필요한 의미론을 타입으로 골라 씁니다(`Box`=단일 소유 힙, `Rc`=공유 소유 등).

**내부 가변성(Interior Mutability)** — 불변 참조(`&T`)를 통해서도 내부 값을 변경할 수 있게 하는 패턴. `Cell<T>`, `RefCell<T>`(단일 스레드), `Mutex<T>`/`RwLock<T>`(다중 스레드)가 제공한다. 대여 규칙 검사를 컴파일 타임에서 런타임으로 옮기는 것이 핵심. 최초 상세 등장 [Chapter 20](../chapters/chapter_20.md). · *Python 멘탈모델 대비*: Python에서는 모든 객체가 기본적으로 가변이라 이런 구분이 없습니다. Rust에서는 "불변이 기본"이므로, 불변 인터페이스 뒤에서 변경이 필요할 때만 명시적으로 내부 가변성을 꺼냅니다.

> 💡 **Under the Hood: `RefCell`은 대여 검사를 런타임으로 미룬다**
>
> ```rust
> use std::cell::RefCell;
> let counter = RefCell::new(0_i32);
> let shared: &RefCell<i32> = &counter; // 불변 참조인데도...
> *shared.borrow_mut() += 1;            // ...내부는 수정 가능
> println!("{}", shared.borrow());      // 출력: 1
> ```
>
> `RefCell`은 XOR 규칙을 버리는 게 아니라, 그 검사를 **런타임으로 옮깁니다**. `borrow_mut()`가 이미 빌려준 상태에서 또 호출되면 컴파일은 통과하되 런타임에 `already borrowed: BorrowMutError`로 패닉합니다. 즉 "컴파일 타임 안전 vs 런타임 유연성"의 트레이드오프를 개발자가 의식적으로 선택하는 도구입니다.

---

## F.3 타입·추상화 (Types & Abstraction)

Rust의 타입 시스템은 Python의 동적 타입과 정반대 방향으로 설계되었습니다 — 가능한 모든 것을 컴파일 타임에 박제합니다. 이 군집은 [Chapter 7](../chapters/chapter_07.md)~[Chapter 11](../chapters/chapter_11.md)에 걸쳐 있습니다.

**트레이트(Trait)** — 타입이 구현해야 할 메서드의 집합을 정의하는 인터페이스. Python의 추상 기반 클래스(ABC)나 프로토콜에 대응하되, 외부 타입에도 나중에 구현을 붙일 수 있다. 최초 상세 등장 [Chapter 9](../chapters/chapter_09.md). · *Python 멘탈모델 대비*: Python의 dunder 메서드(`__add__`, `__len__` 등)가 흩어진 관례라면, Rust의 트레이트(`Add`, `Display`)는 명시적 계약입니다. "이 타입이 이 트레이트를 구현하는가"를 컴파일러가 강제합니다.

**ADT(대수적 데이터 타입, Algebraic Data Type)** — 타입을 곱(product)과 합(sum)으로 조합하는 방식. `struct`는 필드들의 곱 타입, `enum`은 변형(variant)들의 합 타입이다. 최초 상세 등장 [Chapter 10](../chapters/chapter_10.md). · *Python 멘탈모델 대비*: Python에는 진정한 합 타입이 없어 `Union[A, B]`나 `isinstance` 분기로 흉내 냅니다. Rust의 `enum`은 "이 값은 정확히 이 변형들 중 하나"임을 타입으로 보장하고, `match`가 **모든 경우를 다뤘는지(망라성)** 컴파일 타임에 검사합니다.

> 💡 **Under the Hood: enum 변형은 데이터를 품을 수 있다 (Ch10 결제 상태머신)**
>
> ```rust
> enum PaymentState {
>     Pending,
>     Authorized { txn_id: u64 },             // 변형마다 다른 데이터
>     Captured { txn_id: u64, amount: Money },
>     Failed(String),
> }
> ```
>
> `match`로 분기하면 컴파일러가 네 변형을 빠짐없이 다뤘는지 검사합니다. `Failed` 처리를 빠뜨리면 `error[E0004]: non-exhaustive patterns`로 거부합니다. Python의 상태 머신은 문자열 상수나 enum 값으로 표현하지만, "각 상태가 어떤 데이터를 갖는가"를 타입으로 묶지 못해 잘못된 상태-데이터 조합이 런타임에야 드러납니다.

**뉴타입(Newtype)** — 기존 타입을 단일 필드 튜플 구조체로 감싸 새로운 의미를 부여하는 패턴. `struct Money(i64);`처럼. 런타임 표현은 감싼 타입과 동일해 비용이 0이다. 최초 상세 등장 [Chapter 7](../chapters/chapter_07.md), 심화 [Chapter 8](../chapters/chapter_08.md). · *Python 멘탈모델 대비*: Python에서 "금액"과 "수량"을 둘 다 `int`로 두면 섞여도 모릅니다. `Money(i64)`와 `Quantity(i64)`는 타입이 달라 섞으면 컴파일 에러입니다 — 의미를 타입으로 강제합니다.

> 💡 **Under the Hood: 뉴타입은 진짜로 무비용이다**
>
> `struct Meters(f64);`의 크기를 측정하면 정확히 `f64`와 같습니다.
>
> ```rust
> assert_eq!(std::mem::size_of::<Meters>(), std::mem::size_of::<f64>()); // 둘 다 8
> ```
>
> 래퍼가 메모리를 한 바이트도 더 쓰지 않습니다. 컴파일이 끝나면 `Meters`라는 이름은 증발하고 기계어는 그냥 `f64`를 다룹니다. "타입 안전을 얻되 런타임 비용은 0" — 무비용 추상화의 가장 작고 순수한 예입니다.

**제네릭(Generics)** — 타입을 매개변수화해 여러 구체 타입에 대해 같은 코드를 쓰는 기법. `fn add<T: Add>(a: T, b: T)`처럼 트레이트 바운드로 제약한다. 최초 상세 등장 [Chapter 9](../chapters/chapter_09.md). · *Python 멘탈모델 대비*: Python의 함수는 본래 모든 타입을 받지만(덕 타이핑) 런타임에 깨질 수 있습니다. Rust 제네릭은 트레이트 바운드로 "어떤 능력을 가진 타입만" 받도록 컴파일 타임에 제약합니다.

**단일형화(Monomorphization)** — 제네릭 코드를 컴파일러가 실제 사용된 구체 타입마다 전용 함수로 복제 생성하는 기법. 런타임 디스패치 비용 없이 제네릭을 쓰게 해준다. 최초 상세 등장 [Chapter 9](../chapters/chapter_09.md). · *Python 멘탈모델 대비*: Python은 `a + b`마다 타입 슬롯을 런타임에 조회하지만, 단일형화된 Rust 코드는 `i32` 덧셈을 단일 CPU 명령어로 박제합니다([Chapter 1](../chapters/chapter_01.md)의 어셈블리 분석 참조).

**무비용 추상화(Zero-cost Abstraction)** — 제네릭·이터레이터·클로저 같은 고수준 추상이 손으로 짠 저수준 코드와 동일한 기계어로 컴파일되어, 추상화에 따른 런타임 비용이 없는 성질. 최초 상세 등장 [Chapter 1](../chapters/chapter_01.md), 이터레이터 사례 [Chapter 15](../chapters/chapter_15.md). · *Python 멘탈모델 대비*: Python은 추상화할수록 느려집니다(데코레이터·프로퍼티·`__getattr__` 오버헤드). Rust는 추상화의 비용을 컴파일 타임에 정산해 0으로 만듭니다.

**동적 디스패치(dyn, Dynamic Dispatch)** — 호출할 메서드를 런타임에 가상 함수 테이블(vtable)을 통해 결정하는 방식. `dyn Trait`으로 표기하며, 단일형화(정적 디스패치)와 달리 타입을 런타임까지 지운다. 최초 상세 등장 [Chapter 9](../chapters/chapter_09.md), 심화 [Chapter 10](../chapters/chapter_10.md). · *Python 멘탈모델 대비*: Python의 모든 메서드 호출이 사실상 동적 디스패치(딕셔너리 룩업)입니다. Rust는 이를 **선택적**으로만 씁니다 — 기본은 정적, 런타임 다형성이 필요할 때만 `dyn`.

> 💡 **Under the Hood: `dyn Trait`은 fat pointer다**
>
> 일반 참조 `&T`는 8바이트(주소 하나)지만, 트레이트 객체 `&dyn Trait`은 16바이트입니다. 측정값(rustc 1.96.0):
>
> ```rust
> std::mem::size_of::<&dyn Shape>()   // 16  (데이터 포인터 + vtable 포인터)
> std::mem::size_of::<Box<dyn Shape>>() // 16
> ```
>
> 두 포인터 중 하나는 실제 데이터를, 다른 하나는 그 타입의 메서드 주소들이 담긴 **vtable**을 가리킵니다. 메서드를 호출하면 vtable을 한 번 거쳐 간접 호출(indirect call)합니다 — 정적 디스패치보다 느리고 인라이닝도 막히지만, 서로 다른 타입을 한 컬렉션(`Vec<Box<dyn Shape>>`)에 담을 수 있게 해줍니다. 트레이드오프를 타입으로 드러낸 것입니다.

**타입상태(Typestate)** — 객체의 상태(예: 빌드 단계, 연결 여부)를 타입 파라미터로 인코딩해, 잘못된 상태에서의 연산을 컴파일 타임에 차단하는 패턴. 최초 상세 등장 [Chapter 11](../chapters/chapter_11.md). · *Python 멘탈모델 대비*: Python은 "URL 없이 `build()` 호출" 같은 실수를 런타임 예외로만 잡습니다. 타입상태는 그 메서드를 애초에 잘못된 상태의 타입에 **존재하지 않게** 만들어 컴파일 에러로 차단합니다.

> 💡 **Under the Hood: 타입상태로 잘못된 호출을 컴파일 단계에서 지운다**
>
> ```rust
> struct NoUrl;
> struct HasUrl(String);
> struct RequestBuilder<U> { url: U, timeout: u64 }
>
> impl RequestBuilder<NoUrl> {
>     fn url(self, u: &str) -> RequestBuilder<HasUrl> { /* NoUrl -> HasUrl 전이 */ }
> }
> impl RequestBuilder<HasUrl> {
>     fn build(self) -> String { /* build()는 오직 HasUrl 상태에만 존재 */ }
> }
> // RequestBuilder::new().build();  // 🚨 컴파일 에러: NoUrl에는 build()가 없음
> ```
>
> `build()`는 `RequestBuilder<HasUrl>`에만 구현되어 있으므로, URL을 설정하지 않고 호출하면 "메서드를 찾을 수 없음"으로 컴파일이 거부됩니다. 런타임 검증이 컴파일 타임 보장으로 승격된 것입니다.

**빌더(Builder)** — 복잡한 객체를 단계별 메서드 체인으로 구성하는 생성 패턴. Rust는 키워드 인자·기본값이 없어 빌더가 그 자리를 메운다. 최초 상세 등장 [Chapter 11](../chapters/chapter_11.md). · *Python 멘탈모델 대비*: Python은 `def __init__(self, x=10, *, name=None)`처럼 기본값·키워드 인자로 유연한 생성을 지원하지만, Rust에는 그 문법이 없어 빌더 패턴(+타입상태)으로 명시적으로 표현합니다.

**고아 규칙(Orphan Rule)** — 트레이트나 타입 중 **최소 하나는 현재 크레이트에 정의되어 있어야** 그 트레이트를 그 타입에 구현할 수 있다는 규칙. 외부 트레이트를 외부 타입에 구현하는 것을 금지해 구현 충돌을 막는다. 최초 상세 등장 [Chapter 9](../chapters/chapter_09.md). · *Python 멘탈모델 대비*: Python은 몽키 패칭으로 남의 클래스에 아무 메서드나 붙일 수 있어 충돌·예측 불가가 흔합니다. 고아 규칙은 그런 전역 오염을 컴파일 타임에 차단합니다.

> 💡 **Under the Hood: 고아 규칙 위반의 실제 에러**
>
> 외부 트레이트(`Display`)를 외부 타입(`Vec`)에 구현하려 하면 거부됩니다(rustc 1.96.0).
>
> ```text
> error[E0117]: only traits defined in the current crate can be implemented
>               for types defined outside of the crate
>  --> orphan.rs:4:1
>   |
> 4 | impl fmt::Display for Vec<i32> {
>   | ^^^^^^^^^^^^^^^^^^^^^^--------
>   |                       `Vec` is not defined in the current crate
>   = note: define and implement a trait or new type instead
> ```
>
> 해법은 에러 메시지가 직접 알려줍니다 — **뉴타입으로 감싸라**(`struct MyVec(Vec<i32>);`). 그러면 `MyVec`은 내 크레이트 소유이므로 어떤 트레이트든 구현할 수 있습니다. 고아 규칙과 뉴타입 패턴이 한 쌍으로 맞물리는 지점입니다.

**디슈가링(Desugaring)** — 편의 문법(syntactic sugar)을 컴파일러가 더 기본적인 형태로 풀어내는 과정. `?` 연산자가 `match`로, `for` 루프가 `Iterator` 호출로, `async fn`이 상태 기계로 풀린다. 최초 상세 등장 [Chapter 12](../chapters/chapter_12.md), `for`/이터레이터 [Chapter 15](../chapters/chapter_15.md), `async` [Chapter 18](../chapters/chapter_18.md). · *Python 멘탈모델 대비*: Python에도 디슈가링이 있지만(`with`→`__enter__`/`__exit__`), Rust의 디슈가링은 "고수준 문법이 실제로 어떤 저수준 코드로 변하는가"를 이해해야 성능·`Send` 경계를 다룰 수 있다는 점에서 더 실용적으로 중요합니다.

> 💡 **Under the Hood: `?`는 match로 풀린다**
>
> ```rust
> // 우리가 쓰는 편의 문법
> let cents: i64 = s.parse()?;
>
> // 컴파일러가 푸는 형태 (개념적 등가)
> let cents: i64 = match s.parse() {
>     Ok(v) => v,
>     Err(e) => return Err(e),  // 에러면 즉시 함수 탈출
> };
> ```
>
> `?` 한 글자가 "성공이면 값을 꺼내고, 실패면 에러를 호출자에게 즉시 전파"하는 `match`로 풀립니다([Chapter 12](../chapters/chapter_12.md), [Chapter 13](../chapters/chapter_13.md)). Python의 `try/except`가 예외를 위로 던지는 것과 목적은 같지만, Rust는 그 전파 경로가 타입(`Result`)에 명시되어 컴파일러가 추적합니다.

---

## F.4 동시성 (Concurrency)

GIL 없는 진짜 병렬, 그리고 그것을 안전하게 만드는 타입 시스템. 이 군집은 [Chapter 16](../chapters/chapter_16.md)~[Chapter 18](../chapters/chapter_18.md)에 걸쳐 있습니다.

**두려움 없는 동시성(Fearless Concurrency)** — 데이터 레이스를 컴파일러가 타입 시스템으로 원천 차단해, 멀티스레드 코드를 "컴파일되면 메모리 안전하다"고 신뢰할 수 있는 성질. 최초 상세 등장 [Chapter 1](../chapters/chapter_01.md), 심화 [Chapter 16](../chapters/chapter_16.md)~[Chapter 17](../chapters/chapter_17.md). · *Python 멘탈모델 대비*: Python은 GIL로 데이터 레이스를 회피하는 대신 CPU 병렬을 포기합니다. Rust는 GIL 없이 진짜 병렬을 하면서도 레이스를 컴파일 타임에 막습니다.

**Send** — 값의 소유권을 다른 스레드로 **이동**해도 안전함을 나타내는 마커 트레이트. 컴파일러가 자동 구현 여부를 추론한다. 최초 상세 등장 [Chapter 17](../chapters/chapter_17.md). · *Python 멘탈모델 대비*: Python에는 "이 객체를 스레드 간 옮겨도 되는가"라는 타입 수준 개념이 없습니다(어차피 GIL이 직렬화). Rust는 `Send`로 그 안전성을 타입에 새깁니다.

**Sync** — 값을 여러 스레드에서 **공유 참조(`&T`)**로 동시에 접근해도 안전함을 나타내는 마커 트레이트. `T: Sync`는 곧 `&T: Send`와 동치다. 최초 상세 등장 [Chapter 17](../chapters/chapter_17.md). · *Python 멘탈모델 대비*: Python은 공유 가변 상태를 막지 않아 개발자가 직접 락을 걸어야 합니다. Rust는 `Sync`가 아닌 타입을 여러 스레드에서 공유하려 하면 컴파일을 거부합니다.

> 💡 **Under the Hood: `Rc`는 Send가 아니다 — 그래서 컴파일러가 막는다**
>
> `Rc`의 참조 카운트 증감은 원자적이지 않아 스레드 간 공유 시 레이스가 납니다. 그래서 `Rc`는 `Send`가 아니고, 스레드로 넘기면 컴파일 거부됩니다(rustc 1.96.0).
>
> ```text
> error[E0277]: `Rc<i32>` cannot be sent between threads safely
>  --> send.rs:6:19
>   |
> 6 |       thread::spawn(move || {
>   |       ------------- ^------ ... `Rc<i32>` cannot be sent between threads safely
>   = help: within `{closure}`, the trait `Send` is not implemented for `Rc<i32>`
> ```
>
> 해법은 스레드 안전 버전인 `Arc`(Atomic Reference Counted)를 쓰는 것입니다. 컴파일러가 "이 타입은 스레드를 넘으면 안 된다"를 `Send` 트레이트로 알고 차단했습니다 — Python이었다면 운영 환경에서 가끔 카운트가 꼬이는 하이젠버그가 됐을 버그입니다.

**Arc (Atomic Reference Counted)** — 스레드 간 안전하게 공유 소유권을 갖는 스마트 포인터. 참조 카운트를 원자적으로 증감한다. 가변 공유는 `Arc<Mutex<T>>`처럼 락과 조합해야 한다. 최초 상세 등장 [Chapter 17](../chapters/chapter_17.md). · *Python 멘탈모델 대비*: Python은 *모든* 객체가 항상 refcount를 달지만(28바이트 중 8바이트), `Arc`는 진짜 공유가 필요할 때만 선택적으로 씁니다. 99%의 값은 단일 소유라 카운트 자체가 없습니다([Chapter 1](../chapters/chapter_01.md)).

**Mutex/RwLock** — 공유 데이터에 한 번에 하나의 스레드만(또는 RwLock은 여러 reader 혹은 한 writer만) 접근하도록 강제하는 상호 배제 락. Rust에서는 락이 데이터를 **감싸므로** 락 없이 데이터에 접근하는 것 자체가 불가능하다. 최초 상세 등장 [Chapter 17](../chapters/chapter_17.md). · *Python 멘탈모델 대비*: Python의 `threading.Lock`은 락과 데이터가 분리되어 있어 락을 깜빡 잊고 접근해도 막지 않습니다. Rust의 `Mutex<T>`는 데이터를 락 안에 넣어, `lock()`을 거치지 않으면 데이터에 손댈 수 없습니다.

**work-stealing** — 멀티스레드 스케줄러에서, 자기 작업 큐가 빈 스레드가 다른 바쁜 스레드의 큐 뒤쪽에서 작업을 "훔쳐" 와 부하를 자동으로 균형 잡는 기법. `tokio`의 기본 런타임과 `rayon`이 채택한다. 최초 상세 등장 [Chapter 18](../chapters/chapter_18.md), 데이터 병렬 사례 [Chapter 16](../chapters/chapter_16.md). · *Python 멘탈모델 대비*: Python의 `asyncio` 이벤트 루프는 단일 스레드라 work-stealing이 없습니다. `tokio`는 기본적으로 여러 워커 스레드가 태스크를 훔쳐가며 멀티코어를 채웁니다.

**Future / Poll** — `Future`는 "아직 완료되지 않았을 수도 있는 비동기 연산"을 표현하는 트레이트. `poll` 메서드가 호출되면 가능한 만큼 진행하고 `Poll::Ready(값)` 또는 `Poll::Pending`을 반환한다. 최초 상세 등장 [Chapter 18](../chapters/chapter_18.md). · *Python 멘탈모델 대비*: Python의 코루틴은 `await`을 만나면 **스스로** 양보하는 능동적 주체지만, Rust의 future는 외부 executor가 `poll`을 호출해줘야 전진하는 **수동적 데이터**입니다. 양보의 주도권이 다릅니다.

> 🚨 **No Free Lunch: future는 게으르다 — `.await` 안 하면 한 줄도 안 돈다**
>
> `async fn`을 호출하면 즉시 실행되는 게 아니라 `Future`를 구현한 익명 구조체가 반환될 뿐입니다. 누군가 `poll`(보통 `.await` 또는 `tokio::spawn`)할 때까지 코드는 단 한 줄도 실행되지 않습니다. Python에서 `await`을 빠뜨리면 `RuntimeWarning: coroutine was never awaited`가 뜨지만, Rust에서 `.await`을 빠뜨린 future는 조용히 드롭됩니다 — 부작용도 경고도(컴파일러가 `#[must_use]`로 잡지 않는 한) 없습니다. 이 철저한 게으름이 "수만 개 future를 미리 만들어 `select!`로 고르기" 같은 패턴을 공짜로 가능하게 합니다([Chapter 18](../chapters/chapter_18.md)).

---

## F.5 에러 처리 (Error Handling)

예외가 없는 세계에서 실패를 타입으로 다루는 법. 이 군집은 [Chapter 12](../chapters/chapter_12.md)~[Chapter 13](../chapters/chapter_13.md)에 있습니다.

**Option<T>** — 값이 있을 수도(`Some(T)`) 없을 수도(`None`) 있음을 타입으로 표현하는 enum. null의 안전한 대체물. 최초 상세 등장 [Chapter 12](../chapters/chapter_12.md). · *Python 멘탈모델 대비*: Python의 `None`은 어떤 변수에든 숨어들 수 있어 `AttributeError: 'NoneType'`이 런타임에 터집니다. Rust는 `Option<T>`를 명시해, 값을 꺼내기 전에 `None` 처리를 **컴파일러가 강제**합니다.

**Result<T, E>** — 성공(`Ok(T)`) 또는 실패(`Err(E)`)를 타입으로 표현하는 enum. 함수의 실패 가능성이 시그니처에 드러난다. 최초 상세 등장 [Chapter 12](../chapters/chapter_12.md), 커스텀 에러 설계 [Chapter 13](../chapters/chapter_13.md). · *Python 멘탈모델 대비*: Python은 어떤 함수가 어떤 예외를 던지는지 시그니처에 안 나타나(문서를 믿어야 함), `try/except`를 깜빡하면 호출 스택을 타고 올라가 프로세스를 죽입니다. Rust는 실패가 반환 타입에 박혀 무시할 수 없습니다.

**니치 최적화(Niche Optimization)** — 타입이 "절대 가질 수 없는 비트 패턴(니치)"을 enum의 태그로 재활용해 추가 메모리 없이 변형을 구분하는 컴파일러 최적화. 최초 상세 등장 [Chapter 1](../chapters/chapter_01.md), `Option` 맥락 [Chapter 12](../chapters/chapter_12.md). · *Python 멘탈모델 대비*: Python의 `Optional[X]`는 어차피 모든 게 포인터라 표현 최적화가 불가능합니다. Rust는 `Option<&T>`를 null 포인터와 동일한 8바이트로 압축하면서도 null 체크를 강제합니다.

> 💡 **Under the Hood: `Option<&T>`가 8바이트인 이유 (니치)**
>
> 측정값(rustc 1.96.0):
>
> ```rust
> std::mem::size_of::<&i64>()           // 8
> std::mem::size_of::<Option<&i64>>()   // 8   ← 태그가 공짜!
> std::mem::size_of::<Option<Box<i64>>>() // 8 ← 마찬가지
> std::mem::size_of::<Option<i64>>()    // 16  ← i64는 니치가 없어 태그 8바이트 추가
> std::mem::size_of::<Option<bool>>()   // 1   ← bool은 0/1만 쓰므로 나머지 값을 None으로
> ```
>
> 참조와 `Box`는 "절대 null이 아니다"는 불변식이 있어, 주소 0을 `None`의 표식으로 재활용합니다 — 태그 0바이트. 반면 `i64`는 모든 비트 패턴이 유효한 값이라 니치가 없어, 별도 태그 + 정렬 패딩으로 16바이트가 됩니다. "값이 가질 수 없는 구멍이 있는가?"가 판단 기준입니다.

**접착제(Glue) 코드** — 서로 다른 시스템·언어·라이브러리를 이어 붙이는 얇은 연결 코드. 외부 API 호출·일회성 ETL·운영 자동화처럼 실행 시간 대부분이 I/O 대기인 코드. 최초 상세 등장 [Chapter 1](../chapters/chapter_01.md). · *Python 멘탈모델 대비*: Python이 압도적으로 강한 영역입니다. 병목이 CPU가 아니라 네트워크/디스크 대기이므로 Rust로 옮겨도 빨라지지 않습니다 — Rust를 쓰지 *말아야* 할 대표 사례입니다.

**패닉(panic!)** — 복구 불가능한 오류 상황에서 현재 스레드를 즉시 중단(기본적으로 스택 되감기)시키는 메커니즘. `unwrap()`/`expect()`/배열 범위 초과 등이 패닉을 유발한다. 최초 상세 등장 [Chapter 12](../chapters/chapter_12.md). · *Python 멘탈모델 대비*: Python의 예외는 `try/except`로 잡아 복구하는 게 정상이지만, Rust의 패닉은 "여기 도달하면 버그"라는 의미라 보통 잡지 않고 프로그램을 끝냅니다. 복구 가능한 실패는 `Result`로, 진짜 버그는 `panic!`으로 — 둘을 명확히 나눕니다.

---

## F.6 매크로·메타프로그래밍 (Macros & Metaprogramming)

컴파일 타임에 코드를 생성하는 위생적 시스템. 이 군집은 [Chapter 19](../chapters/chapter_19.md)에 있습니다.

**선언적 매크로(Declarative Macro, `macro_rules!`)** — 패턴 매칭으로 토큰을 받아 코드로 확장하는 매크로. `vec![]`, `println!` 등이 이에 속한다. 최초 상세 등장 [Chapter 19](../chapters/chapter_19.md). · *Python 멘탈모델 대비*: Python의 데코레이터는 런타임에 함수를 감싸지만, Rust 매크로는 **컴파일 타임에 코드를 생성**합니다 — 런타임 오버헤드가 없습니다.

**절차적 매크로(Procedural Macro)** — 토큰 스트림을 입력받아 Rust 코드로 변환하는 함수형 매크로. `#[derive(Debug)]`, `#[tokio::main]`, serde의 `#[derive(Serialize)]` 등. 최초 상세 등장 [Chapter 19](../chapters/chapter_19.md). · *Python 멘탈모델 대비*: Python의 메타클래스·데코레이터가 하는 "클래스 자동 변형"을, Rust는 절차적 매크로로 컴파일 타임에 수행합니다. `#[derive(Debug)]`는 `__repr__`을 자동 생성하는 것과 개념이 같되 런타임 비용이 없습니다.

**위생(Hygiene)** — 매크로가 확장될 때 매크로 내부의 식별자와 호출 지점의 식별자가 충돌하지 않도록 컴파일러가 보장하는 성질. 최초 상세 등장 [Chapter 19](../chapters/chapter_19.md). · *Python 멘탈모델 대비*: Python에는 진정한 매크로가 없어(코드를 `exec`로 생성하는 건 위험천만), 이름 충돌을 막아주는 위생 개념도 없습니다. Rust 매크로는 텍스트 치환이 아니라 위생적 토큰 변환이라 변수 캡처 버그가 없습니다.

---

## F.7 성능·FFI (Performance & FFI)

측정 가능한 속도와 Python과의 다리. 이 군집은 [Chapter 1](../chapters/chapter_01.md), [Chapter 14](../chapters/chapter_14.md), [Chapter 21](../chapters/chapter_21.md)~[Chapter 22](../chapters/chapter_22.md)에 걸쳐 있습니다.

**제로카피(Zero-copy)** — 데이터를 새 버퍼로 복사하지 않고 원본을 가리키는 참조/슬라이스로 처리하는 기법. `&str`로 소스 문자열의 일부를 복사 없이 빌려 보는 것이 대표 예. 최초 상세 등장 [Chapter 5](../chapters/chapter_05.md), `String`/`&str` 맥락 [Chapter 14](../chapters/chapter_14.md). · *Python 멘탈모델 대비*: Python의 문자열 슬라이싱(`s[1:5]`)은 새 `str` 객체를 할당해 복사하지만, Rust의 `&s[1..5]`는 원본을 가리키는 16바이트 fat pointer만 만듭니다. `ruff`가 빠른 비결입니다([Chapter 1](../chapters/chapter_01.md)).

> 💡 **Under the Hood: `&str`는 복사하지 않는 16바이트 fat pointer다**
>
> ```rust
> std::mem::size_of::<&str>()  // 16  (데이터 포인터 + 길이)
> std::mem::size_of::<String>() // 24  (포인터 + 길이 + 용량)
> ```
>
> `String`은 힙 데이터를 **소유**하는 24바이트 핸들(ptr·len·cap)이고, `&str`은 그 데이터(또는 일부)를 **빌려 보는** 16바이트 슬라이스(ptr·len)입니다. 함수가 문자열을 읽기만 한다면 `&str`로 받아 복사를 피하는 것이 관용구입니다 — 수백만 행을 파싱할 때 이 "복사 안 함"이 그대로 처리량이 됩니다.

**usize** — 플랫폼 포인터 크기의 부호 없는 정수(64비트 머신에서 8바이트). 슬라이스/`Vec` 인덱싱과 범위(`0..n`)에 쓰인다. 최초 상세 등장 [Chapter 14](../chapters/chapter_14.md). · *Python 멘탈모델 대비*: Python은 `lst[i]`의 `i`가 그냥 정수면 되고 음수면 뒤에서부터 셉니다. Rust는 인덱싱에 `usize`를 요구하며, `i64`로 인덱싱하면 컴파일 거부합니다([Chapter 1](../chapters/chapter_01.md)의 함정 참조).

**FFI (Foreign Function Interface)** — 다른 언어로 작성된 코드(주로 C ABI)를 호출하거나 노출하는 인터페이스. `extern "C"`, `#[repr(C)]`로 메모리 레이아웃을 맞춘다. 최초 상세 등장 [Chapter 21](../chapters/chapter_21.md). · *Python 멘탈모델 대비*: Python의 `ctypes`/`cffi`가 하는 일을 Rust는 `extern` 블록으로 합니다. 차이는 Rust가 경계에서 타입·레이아웃을 컴파일 타임에 검증한다는 점입니다.

**unsafe** — 컴파일러의 메모리 안전 보장을 일부 끄고, 안전성 책임을 개발자가 지는 코드 블록/함수. 원시 포인터 역참조, FFI 호출 등에 필요하다. 최초 상세 등장 [Chapter 21](../chapters/chapter_21.md). · *Python 멘탈모델 대비*: Python에는 "안전/비안전" 구분이 없습니다(전부 런타임 검사). Rust의 `unsafe`는 "여기는 컴파일러가 못 지켜주니 내가 책임진다"고 **명시적으로 표시한 작은 섬**이며, 그 바깥은 여전히 안전합니다.

**PyO3** — Rust로 Python 확장 모듈을 만들거나 Rust에서 Python을 호출하게 해주는 크레이트. `pydantic-core`·`polars` 등이 Python에 노출되는 다리. 최초 상세 등장 [Chapter 22](../chapters/chapter_22.md). · *Python 멘탈모델 대비*: "Python으로 99%, 핫스팟 1%만 Rust로"라는 하이브리드 전략의 실현 도구입니다([Chapter 1](../chapters/chapter_01.md)). Python을 버리는 게 아니라, Python이 끝나는 지점에서 Rust를 `import`해 씁니다.

> 🚨 **No Free Lunch: "Rust로 짜면 무조건 빠르다"는 미신**
>
> 이 부록의 모든 성능 용어를 관통하는 경고입니다. (1) I/O 바운드 코드(접착제)는 Rust로 옮겨도 안 빨라집니다 — 병목이 CPU가 아니니까요. (2) 이미 NumPy/Polars가 네이티브로 처리하는 워크로드도 마찬가지입니다. (3) `--release` 없는 디버그 빌드는 릴리스보다 10~100배 느려 "Rust가 Python보다 느리다"는 황당한 결론을 부릅니다. Rust의 100배는 **순수 CPU 인터프리터 핫루프**라는 Python에 가장 불리한 시나리오에서 나온 수치입니다. 먼저 측정(`py-spy`/`cProfile`)하고, 진짜 병목이 CPU일 때만 Rust를 꺼내십시오([Chapter 1](../chapters/chapter_01.md), [부록 E](./appendix_e_migration.md)).

---

## F.8 핵심 수치 빠른 참조 (Quick Reference: Measured Sizes)

본문 곳곳에 흩어진 메모리 수치를 한 표에 모읍니다. Python은 `sys.getsizeof()`(CPython 3.13.6), Rust는 `std::mem::size_of()`(rustc 1.96.0, 64비트)로 **직접 측정**한 값입니다.

| 데이터 | Python (CPython 3.13.6) | Rust (rustc 1.96.0, 64bit) |
|--------|--------------------------|----------------------------|
| 정수 1개 | `int(1)` = **28**바이트 | `i64` = **8**바이트 |
| 빈 문자열 | `str('')` = **41**바이트 | `String::new()` = **24**바이트 핸들 |
| n글자 ASCII 문자열 | `str` = **41 + n**바이트 (`'hello'`=46) | `String` = **24**바이트 핸들 + 힙 데이터 |
| 문자열 슬라이스 | (개념 없음) | `&str` = **16**바이트 fat pointer |
| 빈 리스트/벡터 | `list([])` = **56**바이트 | `Vec<i64>` = **24**바이트 핸들 (cap 0) |
| 빈 딕셔너리 | `dict({})` = **64**바이트 | `HashMap` = **48**바이트 핸들 |
| 옵셔널 참조 | (전부 포인터, 최적화 불가) | `Option<&T>` = **8**바이트 (니치) |
| 박스 포인터 | (개념 없음) | `Box<T>` = **8**바이트 |
| 트레이트 객체 | (전부 동적) | `&dyn Trait` = **16**바이트 (fat pointer) |

이 표가 의미하는 바는 분명합니다. 같은 데이터를 다룰 때 Rust 프로세스는 통상 Python의 1/3~1/5 메모리를 씁니다. 수백만 행 워크로드에서는 이 차이가 "32GB 인스턴스가 필요하냐, 8GB로 되냐"를 가릅니다([Chapter 1](../chapters/chapter_01.md)).

---

## F.9 영문 ↔ 한글 양방향 색인 (Bidirectional Index)

어느 방향에서 찾든 용어를 빠르게 위치시키기 위한 색인입니다. "딥링크" 열은 그 용어가 처음으로 깊이 다뤄지는 장입니다.

### 영문 → 한글 (알파벳순)

| English | 한국어 | 딥링크 |
|---------|--------|--------|
| ADT (Algebraic Data Type) | 대수적 데이터 타입 | [Ch10](../chapters/chapter_10.md) |
| Arc | 원자적 참조 카운팅 | [Ch17](../chapters/chapter_17.md) |
| Borrowing | 대여 | [Ch5](../chapters/chapter_05.md) |
| Builder | 빌더 | [Ch11](../chapters/chapter_11.md) |
| Declarative Macro | 선언적 매크로 | [Ch19](../chapters/chapter_19.md) |
| Desugaring | 디슈가링 | [Ch12](../chapters/chapter_12.md) |
| Dynamic Dispatch (dyn) | 동적 디스패치 | [Ch9](../chapters/chapter_09.md) |
| Drop / RAII | 드롭 / RAII | [Ch4](../chapters/chapter_04.md) |
| Fearless Concurrency | 두려움 없는 동시성 | [Ch16](../chapters/chapter_16.md) |
| FFI | 외부 함수 인터페이스 | [Ch21](../chapters/chapter_21.md) |
| Future / Poll | 퓨처 / 폴 | [Ch18](../chapters/chapter_18.md) |
| Generics | 제네릭 | [Ch9](../chapters/chapter_09.md) |
| Glue code | 접착제 코드 | [Ch1](../chapters/chapter_01.md) |
| Hygiene | 위생 | [Ch19](../chapters/chapter_19.md) |
| Interior Mutability | 내부 가변성 | [Ch20](../chapters/chapter_20.md) |
| Lifetime | 수명 | [Ch6](../chapters/chapter_06.md) |
| Monomorphization | 단일형화 | [Ch9](../chapters/chapter_09.md) |
| Move | 이동 | [Ch4](../chapters/chapter_04.md) |
| Mutex / RwLock | 뮤텍스 / 읽기-쓰기 락 | [Ch17](../chapters/chapter_17.md) |
| Newtype | 뉴타입 | [Ch7](../chapters/chapter_07.md) |
| Niche Optimization | 니치 최적화 | [Ch1](../chapters/chapter_01.md) |
| Option | 옵션 | [Ch12](../chapters/chapter_12.md) |
| Orphan Rule | 고아 규칙 | [Ch9](../chapters/chapter_09.md) |
| Ownership | 소유권 | [Ch4](../chapters/chapter_04.md) |
| panic! | 패닉 | [Ch12](../chapters/chapter_12.md) |
| Procedural Macro | 절차적 매크로 | [Ch19](../chapters/chapter_19.md) |
| PyO3 | PyO3 | [Ch22](../chapters/chapter_22.md) |
| Reference | 참조 | [Ch5](../chapters/chapter_05.md) |
| Result | 결과 | [Ch12](../chapters/chapter_12.md) |
| Send | Send (스레드 이동 안전) | [Ch17](../chapters/chapter_17.md) |
| Smart Pointer | 스마트 포인터 | [Ch20](../chapters/chapter_20.md) |
| Sync | Sync (스레드 공유 안전) | [Ch17](../chapters/chapter_17.md) |
| Trait | 트레이트 | [Ch9](../chapters/chapter_09.md) |
| Typestate | 타입상태 | [Ch11](../chapters/chapter_11.md) |
| unsafe | 언세이프 | [Ch21](../chapters/chapter_21.md) |
| usize | usize | [Ch14](../chapters/chapter_14.md) |
| work-stealing | 작업 훔치기 | [Ch18](../chapters/chapter_18.md) |
| XOR Rule | XOR 규칙 (대여 배타) | [Ch5](../chapters/chapter_05.md) |
| Zero-copy | 제로카피 | [Ch5](../chapters/chapter_05.md) |
| Zero-cost Abstraction | 무비용 추상화 | [Ch1](../chapters/chapter_01.md) |

### 한글 → 영문 (가나다순)

| 한국어 | English | 딥링크 |
|--------|---------|--------|
| 고아 규칙 | Orphan Rule | [Ch9](../chapters/chapter_09.md) |
| 내부 가변성 | Interior Mutability | [Ch20](../chapters/chapter_20.md) |
| niche 최적화 | Niche Optimization | [Ch1](../chapters/chapter_01.md) |
| 단일형화 | Monomorphization | [Ch9](../chapters/chapter_09.md) |
| 대수적 데이터 타입 | ADT | [Ch10](../chapters/chapter_10.md) |
| 대여 | Borrowing | [Ch5](../chapters/chapter_05.md) |
| 동적 디스패치 | Dynamic Dispatch (dyn) | [Ch9](../chapters/chapter_09.md) |
| 두려움 없는 동시성 | Fearless Concurrency | [Ch16](../chapters/chapter_16.md) |
| 드롭 / RAII | Drop / RAII | [Ch4](../chapters/chapter_04.md) |
| 디슈가링 | Desugaring | [Ch12](../chapters/chapter_12.md) |
| 뉴타입 | Newtype | [Ch7](../chapters/chapter_07.md) |
| 무비용 추상화 | Zero-cost Abstraction | [Ch1](../chapters/chapter_01.md) |
| 뮤텍스 / 읽기-쓰기 락 | Mutex / RwLock | [Ch17](../chapters/chapter_17.md) |
| 빌더 | Builder | [Ch11](../chapters/chapter_11.md) |
| 선언적 매크로 | Declarative Macro | [Ch19](../chapters/chapter_19.md) |
| 소유권 | Ownership | [Ch4](../chapters/chapter_04.md) |
| 수명 | Lifetime | [Ch6](../chapters/chapter_06.md) |
| 스마트 포인터 | Smart Pointer | [Ch20](../chapters/chapter_20.md) |
| 언세이프 | unsafe | [Ch21](../chapters/chapter_21.md) |
| 옵션 | Option | [Ch12](../chapters/chapter_12.md) |
| 외부 함수 인터페이스 | FFI | [Ch21](../chapters/chapter_21.md) |
| 원자적 참조 카운팅 | Arc | [Ch17](../chapters/chapter_17.md) |
| 위생 | Hygiene | [Ch19](../chapters/chapter_19.md) |
| 이동 | Move | [Ch4](../chapters/chapter_04.md) |
| 작업 훔치기 | work-stealing | [Ch18](../chapters/chapter_18.md) |
| 절차적 매크로 | Procedural Macro | [Ch19](../chapters/chapter_19.md) |
| 접착제 코드 | Glue code | [Ch1](../chapters/chapter_01.md) |
| 제네릭 | Generics | [Ch9](../chapters/chapter_09.md) |
| 제로카피 | Zero-copy | [Ch5](../chapters/chapter_05.md) |
| 참조 | Reference | [Ch5](../chapters/chapter_05.md) |
| 타입상태 | Typestate | [Ch11](../chapters/chapter_11.md) |
| 트레이트 | Trait | [Ch9](../chapters/chapter_09.md) |
| 패닉 | panic! | [Ch12](../chapters/chapter_12.md) |
| 퓨처 / 폴 | Future / Poll | [Ch18](../chapters/chapter_18.md) |
| 결과 | Result | [Ch12](../chapters/chapter_12.md) |
| Arc | Arc | [Ch17](../chapters/chapter_17.md) |
| PyO3 | PyO3 | [Ch22](../chapters/chapter_22.md) |
| Send / Sync | Send / Sync | [Ch17](../chapters/chapter_17.md) |
| usize | usize | [Ch14](../chapters/chapter_14.md) |
| XOR 규칙 | XOR Rule | [Ch5](../chapters/chapter_05.md) |

---

용어집은 한 번 읽고 덮는 글이 아닙니다. 본문을 읽다 막힐 때마다 돌아와 좌표를 확인하고, 정의 한 줄과 "Python 멘탈모델 대비" 한 줄로 감을 되살린 뒤, 딥링크로 깊은 설명에 다시 잠수하십시오. 22개 장에 흩어진 개념들이 이 한 장 안에서 서로 어떻게 연결되는지 — 소유권이 `Send`를 낳고, `Send`가 두려움 없는 동시성을 떠받치며, 니치 최적화가 `Option`을 공짜로 만들고, 뉴타입이 고아 규칙의 짝이 되는 — 그 그물을 한눈에 보게 될 때, Rust의 설계가 우연이 아니라 **일관된 하나의 철학**임이 드러납니다.

그 철학은 [Chapter 1](../chapters/chapter_01.md)에서 말한 한 문장으로 압축됩니다: **"가능한 모든 결정을 컴파일 타임으로 끌어올려라."** 이 부록의 모든 용어는 그 한 문장의 서로 다른 얼굴입니다. 본문으로 돌아가 그 얼굴들을 하나씩 다시 만나 보십시오.

---

## References

- https://doc.rust-lang.org/book/ — The Rust Programming Language (공식 책)
- https://doc.rust-lang.org/reference/ — The Rust Reference (언어 명세)
- https://doc.rust-lang.org/std/ — 표준 라이브러리 API 문서
- https://doc.rust-lang.org/nomicon/ — The Rustonomicon (unsafe·메모리 심화)
- https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules — 고아 규칙 명세
- https://doc.rust-lang.org/std/option/enum.Option.html — `Option` 및 니치 최적화
- https://rust-lang.github.io/async-book/ — Asynchronous Programming in Rust (Future/Poll)
- https://doc.rust-lang.org/nomicon/send-and-sync.html — `Send`/`Sync` 명세
- https://pyo3.rs/ — PyO3 공식 가이드
