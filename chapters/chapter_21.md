# Chapter 21. C FFI와 Unsafe Rust

`import numpy as np`라고 타이핑할 때, 여러분은 사실 무슨 일이 벌어지는지 알고 있나요? 파이썬 인터프리터는 `numpy/core/_multiarray_umath.so`라는 컴파일된 공유 라이브러리(shared object)를 `dlopen`으로 메모리에 올립니다. 그 안에는 C와 일부 어셈블리(SIMD 커널)로 작성된 함수들이 들어 있습니다. 여러분이 `arr @ other`라고 쓰는 순간 호출되는 것은 파이썬 바이트코드가 아니라 BLAS 라이브러리의 `dgemm` 같은 C 함수입니다. 파이썬 인터프리터는 그저 인자를 포장해서 C 함수에 넘기고, 결과 포인터를 받아 파이썬 객체로 다시 감싸는 **얇은 접착제(glue)** 역할을 합니다.

이것이 파이썬이 느린 언어인데도 데이터 과학과 AI의 표준어가 된 비밀입니다. 파이썬은 빠른 언어가 아닙니다. 파이썬은 **세상에서 가장 다루기 쉬운 C 언어 래퍼(Wrapper)**입니다. `numpy`, `pandas`, `torch`, `tensorflow`, `lxml`, `pydantic-core`, 심지어 `re` 모듈의 정규식 엔진까지, 성능이 중요한 모든 것은 C/C++/Rust로 내려가 있고 파이썬은 그 위에 우아한 API를 씌웁니다.

한 언어에서 다른 언어로 작성된 함수를 직접 호출하는 이 인터페이스를 **FFI(Foreign Function Interface, 외부 함수 인터페이스)**라고 부릅니다. FFI의 공용어는 사실상 **C ABI(Application Binary Interface)**입니다. C는 1970년대부터 운영체제와 함께 살아온 언어라, 거의 모든 언어가 "C와 대화하는 법"을 알고 있습니다. 두 언어가 직접 대화하는 대신, 둘 다 C라는 공용어로 번역해서 만나는 셈입니다.

그런데 파이썬에서 C로 내려가는 길은 결코 안전하지 않습니다. C-API로 확장 모듈을 직접 짜본 사람이라면 `Py_INCREF`를 한 번 빠뜨려 세그멘테이션 폴트(Segmentation Fault)로 프로세스가 통째로 죽는 경험, 혹은 `Py_DECREF`를 한 번 더 호출해 멀쩡한 객체가 조기 해제되는 use-after-free를 겪어봤을 겁니다. 파이썬의 안전망은 C 경계를 넘는 순간 사라집니다. 그리고 그 경계 너머에는 파이썬이 자랑하던 어떤 보호 장치도 없습니다.

Rust 역시 C와 완벽하게 통신합니다. 하지만 Rust 컴파일러는 C 함수 내부에서 포인터가 어떻게 움직이는지, 그 함수가 넘겨받은 포인터를 언제까지 유효하다고 가정하는지 알 방법이 없습니다. 컴파일러가 메모리 안전을 *증명할 수 없는* 영역에 들어서는 순간, Rust는 그 사실을 코드에 명시하라고 요구합니다. 그 명시가 바로 **`unsafe`** 키워드입니다. 이번 장에서는 `unsafe`가 정확히 무엇을 잠금 해제하는지, 원시 포인터(raw pointer)의 정체가 무엇인지, 그리고 std 라이브러리조차 내부적으로 쓰는 `unsafe`를 어떻게 안전한 API로 포장하는지를 바이트 단위와 심볼 테이블 단위로 해부합니다.

---

## 21.1 The Python Way — C-API와 참조 카운팅 지옥

먼저 우리가 지금까지 파이썬 생태계에 네이티브 코드를 제공해 온 방식을 정직하게 들여다봅시다. 파이썬에 C 확장을 제공하는 정통 경로는 `Python.h`를 인클루드하고 CPython의 C-API를 직접 호출하는 것입니다.

```c
#include <Python.h>

// 파이썬에서 my_module.add(a, b)로 호출될 함수
static PyObject* my_module_add(PyObject* self, PyObject* args) {
    long a, b;

    // 파이썬 튜플 인자를 C long 두 개로 파싱
    if (!PyArg_ParseTuple(args, "ll", &a, &b)) {
        return NULL;  // 파싱 실패 시 예외 설정 + NULL 반환
    }

    // C long을 파이썬 int 객체로 다시 포장 (새 참조, refcnt = 1)
    return PyLong_FromLong(a + b);
}

static PyMethodDef MyModuleMethods[] = {
    {"add", my_module_add, METH_VARARGS, "두 정수를 더합니다."},
    {NULL, NULL, 0, NULL}  // 센티넬
};

static struct PyModuleDef my_module = {
    PyModuleDef_HEAD_INIT, "my_module", NULL, -1, MyModuleMethods
};

PyMODINIT_FUNC PyInit_my_module(void) {
    return PyModule_Create(&my_module);
}
```

겨우 두 정수를 더하는 함수 하나를 노출하기 위해 이만큼의 보일러플레이트가 필요합니다. 그런데 이 코드의 진짜 무서운 점은 분량이 아니라 **눈에 보이지 않는 계약(invisible contract)**입니다.

**함정 1: 참조 카운트를 손으로 관리한다.** CPython의 모든 객체는 헤더에 `ob_refcnt` 필드를 가집니다. C-API 함수들은 각자 "이 함수는 새 참조를 돌려준다(new reference)", "빌린 참조만 돌려준다(borrowed reference)", "인자의 참조를 가져간다(steals reference)"는 규약을 가지고 있고, 이 규약은 *문서에만* 적혀 있습니다. 컴파일러는 검증해 주지 않습니다.

```c
PyObject* item = PyList_GetItem(list, 0);  // 빌린 참조 — INCREF 하면 안 됨
PyObject* num  = PyLong_FromLong(42);       // 새 참조 — 다 쓰면 DECREF 해야 함
// num을 DECREF 빠뜨리면 → 메모리 누수
// item을 DECREF 하면 → 조기 해제 → use-after-free → 세그폴트
```

`Py_DECREF`를 한 번 빠뜨리면 그 객체는 영원히 해제되지 않습니다(누수). 한 번 더 호출하면 살아 있는 객체가 죽어 다른 곳에서 그 메모리를 건드릴 때 세그멘테이션 폴트가 터집니다. 이 버그는 *재현이 어렵습니다*. 누수는 며칠 운영해야 메모리 그래프에서 드러나고, 조기 해제는 GC가 그 메모리 슬롯을 재사용하는 타이밍에만 터지기 때문입니다.

**함정 2: GIL(Global Interpreter Lock)을 명시적으로 다뤄야 한다.** C 확장에서 무거운 계산이나 블로킹 I/O를 할 때는 `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS` 매크로로 GIL을 풀었다가 다시 잡아야 다른 파이썬 스레드가 진행됩니다. 이 짝을 잘못 맞추면 데드락이거나, GIL을 놓은 채로 파이썬 객체를 건드려 인터프리터 상태를 깨뜨립니다. 더 근본적으로, GIL을 풀지 않은 C 확장은 아무리 빨라도 *CPU 병렬성*을 얻지 못합니다. 코어가 32개여도 GIL을 쥔 채 도는 함수는 한 번에 하나만 실행됩니다. 네이티브 확장의 진짜 가치는 "GIL을 풀고 여러 코어에서 동시에 계산"하는 데 있는데, 이 구간 관리를 손으로 해야 한다는 것이 C-API의 또 다른 부담입니다.

**함정 3: 에러 처리가 NULL 규약과 전역 예외 상태에 묶여 있다.** C 함수는 실패를 `NULL` 반환 + 스레드 전역 예외 상태 설정으로 표현합니다. `NULL` 체크를 빠뜨리면 다음 줄에서 널 포인터 역참조가 일어나고, 예외 설정 없이 `NULL`을 반환하면 인터프리터가 "SystemError: error return without exception set"을 뱉습니다.

> **Under the Hood: CPython 객체 헤더의 실제 무게**
>
> 64비트 CPython에서 모든 객체는 공통 헤더로 시작합니다. Chapter 4에서 본 그 구조입니다.
>
> ```text
> ┌──────────────────────┬──────────────────────┐
> │ ob_refcnt (8 bytes)  │ ob_type   (8 bytes)  │   ← PyObject 헤더 = 16 bytes
> ├──────────────────────┴──────────────────────┤
> │ ... 실제 데이터 ...                            │
> └──────────────────────────────────────────────┘
> ```
>
> C-API로 정수를 주고받을 때 일어나는 일을 추적해 봅시다. `a + b`라는 C 레벨의 덧셈 자체는 단일 CPU 명령어입니다. 하지만 그 결과를 파이썬으로 돌려주려면 `PyLong_FromLong`이 힙에 최소 28바이트짜리 객체를 `malloc`하고, 헤더를 채우고, `ob_refcnt`를 1로 설정해야 합니다. 즉, **연산은 1 사이클인데 그 결과를 파이썬 세계로 포장하는 데 수십~수백 사이클이 듭니다.** 이것이 "파이썬 루프에서 숫자를 하나씩 더하면 느리지만, numpy로 배열 전체를 한 번에 더하면 빠른" 근본 이유입니다. 객체 포장 비용을 배열당 한 번으로 분산시키는 것이죠.

파이썬 C-API의 결론은 분명합니다. 강력하지만, **모든 안전 불변식(invariant)을 사람이 머릿속과 문서로 관리해야 하는** 세계입니다. 컴파일러는 침묵하고, 버그는 런타임에, 그것도 운영 환경에서야 터집니다. 이제 Rust가 같은 문제를 어떻게 바라보는지 봅시다.

---

## 21.2 The Rust Mental Model — `unsafe`는 "위험"이 아니라 "내가 책임진다"는 서명

파이썬 개발자가 `unsafe`라는 단어를 처음 보면 "위험하니까 쓰지 말라는 경고"로 오해합니다. 정확히 반대입니다. Rust에서 `unsafe`의 진짜 의미는 이렇습니다.

> *"컴파일러인 나는 이 블록 안의 메모리 안전성을 증명할 수 없다. 그러니 개발자인 네가, 여기서만큼은, 그 불변식을 직접 보장한다고 서명해라."*

핵심은 **`unsafe`가 안전 검사를 끄는 것이 아니라는 점**입니다. 흔한 착각은 "`unsafe` 블록에 들어가면 빌림 검사기(Borrow Checker)와 타입 검사가 비활성화된다"는 것인데, 사실이 아닙니다. `unsafe` 안에서도 소유권 규칙, 대여 규칙, 타입 검사, 수명(Lifetime) 검사가 *전부 그대로 작동합니다*. `unsafe`는 단지 평소에 잠겨 있던 **다섯 가지 추가 능력**을 잠금 해제할 뿐입니다.

Rust 레퍼런스가 정의하는 `unsafe`의 다섯 가지 "초능력(superpower)"은 다음과 같습니다.

1. **원시 포인터(raw pointer, `*const T` / `*mut T`)의 역참조(dereference).**
2. **`unsafe`로 표시된 함수(C FFI 함수 포함) 호출.**
3. **가변 정적 변수(`static mut`)의 접근 및 수정.**
4. **`union` 필드 접근.**
5. **`unsafe` 트레이트(예: `Send`, `Sync`)의 구현.**

이게 전부입니다. 이 다섯 가지를 제외한 나머지 모든 규칙은 `unsafe` 안에서도 유효합니다. 예를 들어 `unsafe` 블록 안에서 소유권이 이동된 값을 다시 쓰면 여전히 E0382(use of moved value)가 납니다. `unsafe`는 "Rust의 안전 보장 전체를 끄는 스위치"가 아니라, "이 다섯 가지에 한해 컴파일러의 증명 의무를 사람에게 위임하는 좁은 게이트"입니다.

다섯 번째 능력인 **`unsafe` 트레이트 구현**은 파이썬 개발자에게 가장 낯설 테니 따로 짚겠습니다. Chapter 16~17에서 본 `Send`와 `Sync`가 대표적인 `unsafe` 트레이트입니다. `Send`는 "이 타입을 다른 스레드로 *이동*해도 안전하다", `Sync`는 "이 타입의 참조를 여러 스레드가 *공유*해도 안전하다"는 표식입니다. 대부분의 타입은 컴파일러가 이 트레이트들을 자동으로 부여하지만, FFI로 받은 원시 포인터를 담은 구조체처럼 컴파일러가 스레드 안전성을 *증명할 수 없는* 타입에는 자동 부여되지 않습니다. 이때 개발자가 "내가 검증했으니 이 타입은 스레드 안전하다"고 서명하려면 `unsafe impl Send for MyType {}`를 직접 써야 합니다. `impl` 앞에 붙은 `unsafe`가 바로 그 서명입니다. 이를 잘못 선언하면 컴파일러는 데이터 레이스를 막을 수 없게 됩니다 — 표식을 믿고 검사를 면제했기 때문입니다.

이 멘탈 모델의 실천적 귀결이 Rust 생태계의 핵심 패턴입니다. **`unsafe`를 최대한 작게 가두고, 그 위를 안전한 API로 덮는다.** Rust 표준 라이브러리 자체가 이 패턴의 거대한 증거입니다. `Vec<T>`는 내부적으로 `unsafe`한 원시 포인터 조작으로 힙 버퍼를 관리하지만, 우리가 `vec.push(x)`를 호출할 때 `unsafe`를 쓸 필요는 없습니다. std가 그 `unsafe`를 검증된 안전 추상화 뒤에 봉인해 두었기 때문입니다.

| 측면 | Safe Rust | Unsafe Rust | C |
|---|---|---|---|
| 메모리 안전 보장 | 컴파일러가 증명 | 다섯 능력에 한해 사람이 보장, 나머진 컴파일러 | 전적으로 사람이 보장 |
| 빌림 검사 / 소유권 | 적용됨 | **여전히 적용됨** (끄지 않음) | 없음 |
| 원시 포인터 역참조 | 금지 (E0133) | 허용 | 항상 허용 |
| 데이터 레이스 | 컴파일 타임 차단 | 사람이 차단 책임 | 차단 장치 없음 |
| use-after-free / double-free | 불가능 | 사람이 회피 책임 | 흔한 버그 원천 |
| 미정의 동작(UB) 가능성 | 없음(건전성 버그 제외) | 규약 위반 시 발생 | 상시 존재 |
| 적용 범위 | 코드의 99% | 좁은 경계 블록 | 코드 전체 |

표의 핵심은 마지막 행입니다. C는 *프로그램 전체*가 안전을 사람에게 의존하지만, Rust는 그 위험을 `unsafe` 블록이라는 **감사 가능한(auditable) 좁은 영역**으로 응축시킵니다. 메모리 버그가 의심될 때 C 프로그램에서는 전체 코드를 의심해야 하지만, Rust에서는 `unsafe` 블록만 들여다보면 됩니다. `grep -rn unsafe`가 곧 보안 감사의 출발점이 됩니다.

---

## 21.3 원시 포인터와 참조의 차이, 그리고 E0133

`unsafe`를 이해하려면 먼저 **원시 포인터(raw pointer)**가 참조(reference)와 무엇이 다른지 알아야 합니다. Chapter 5에서 본 `&T`와 `&mut T`(참조)는 컴파일러가 다음을 *보장하는* 포인터입니다.

- 항상 유효한(살아 있는) 메모리를 가리킨다 (널이 아니고, 댕글링이 아니다).
- 가리키는 타입에 맞게 정렬(align)되어 있다.
- 대여 규칙을 지킨다: 불변 참조는 여러 개, 가변 참조는 단 하나(aliasing XOR mutability).

반면 원시 포인터 `*const T`와 `*mut T`는 이 모든 보장을 **포기한** 포인터입니다. C의 포인터와 정확히 같습니다. 널일 수도, 댕글링일 수도, 정렬이 깨졌을 수도, 같은 메모리를 가리키는 가변 포인터가 열 개여도 컴파일러는 아무 말도 하지 않습니다.

```rust
fn main() {
    let value: i32 = 42;

    // 원시 포인터를 *만드는* 것은 safe다. 가리키기만 할 뿐 메모리를 건드리지 않으므로.
    let r_const: *const i32 = &value;
    let r_mut: *mut i32 = &value as *const i32 as *mut i32;

    println!("주소: {:p}", r_const); // 주소 출력도 safe

    // 하지만 *역참조*는 unsafe다. 이 포인터가 유효한지 컴파일러가 모른다.
    let v = unsafe { *r_const };
    println!("값: {}", v);

    let _ = r_mut;
}
```

여기서 결정적인 비대칭이 있습니다. 원시 포인터를 *만드는 것*과 주소를 출력하는 것은 안전합니다. 위험한 행위는 오직 **역참조(dereference)**, 즉 그 주소의 메모리를 실제로 읽거나 쓰는 순간뿐입니다. Rust는 정확히 그 지점에만 `unsafe`를 요구합니다.

이 규칙을 어기면 어떻게 될까요? 직접 컴파일러에게 물어봅시다.

```rust
fn main() {
    let value: i32 = 42;
    let r: *const i32 = &value;
    let v = *r; // 🚨 컴파일 에러! unsafe 없이 원시 포인터 역참조
    println!("{}", v);
}
```

`rustc`의 응답입니다.

```text
error[E0133]: dereference of raw pointer is unsafe and requires
              unsafe function or block
 --> src/main.rs:4:13
  |
4 |     let v = *r; // 🚨 컴파일 에러!
  |             ^^ dereference of raw pointer
  |
  = note: raw pointers may be null, dangling or unaligned; they can
          violate aliasing rules and cause data races: all of these
          are undefined behavior

For more information about this error, try `rustc --explain E0133`.
```

에러 코드 `E0133`을 해독해 봅시다. 메시지는 두 가지를 말합니다. 첫째, "원시 포인터 역참조는 unsafe하니 `unsafe` 함수나 블록이 필요하다." 둘째, note가 *왜* 위험한지 정확히 열거합니다 — 널일 수도, 댕글링일 수도, 정렬이 깨졌을 수도 있고, 별칭(aliasing) 규칙을 위반해 데이터 레이스를 일으킬 수 있으며, **이 모든 것이 미정의 동작(Undefined Behavior, UB)**이라고. 해결은 `let v = unsafe { *r };`처럼 역참조를 `unsafe` 블록으로 감싸는 것입니다. 그 순간 여러분은 "이 포인터가 유효함을 내가 보증한다"고 서명한 셈입니다.

> **Under the Hood: 참조와 원시 포인터의 런타임 표현은 동일하다**
>
> 흥미로운 사실 하나. `&i32`, `*const i32`, 그리고 C의 `const int*`는 **런타임에 완전히 동일한 비트**입니다. 셋 다 그냥 64비트 메모리 주소일 뿐입니다. 차이는 오직 컴파일 타임에만 존재합니다.
>
> ```text
> let r: &i32         →  [ 0x7ffe...a4c ]   8바이트 주소. 컴파일러가 유효성·대여규칙 검증.
> let p: *const i32   →  [ 0x7ffe...a4c ]   8바이트 주소. 컴파일러 검증 없음.
> ```
>
> 즉 원시 포인터는 "참조에서 컴파일러의 보장만 벗겨낸 것"입니다. 추가 런타임 비용은 0입니다. `&value as *const i32` 같은 변환이 무비용 추상화(Zero-cost Abstraction)인 이유입니다 — 비트는 그대로고, 컴파일러가 부여하던 검증 의무만 사람에게 넘어옵니다. 단, 얇은(thin) 포인터에 한합니다. `&str`이나 `&[T]` 같은 슬라이스 참조는 (주소, 길이) 두 워드짜리 뚱뚱한(fat) 포인터이고, 이를 원시 포인터로 다룰 때는 `*const [T]` 역시 두 워드입니다.

---

## 21.4 Rust에서 C 함수 호출하기 — `extern "C"`와 링킹

이제 실전입니다. Rust에서 C 표준 라이브러리 함수를 호출해 봅시다. C 함수를 부르려면 두 가지가 필요합니다. (1) 그 함수의 시그니처를 Rust에게 선언해 주는 `extern "C"` 블록, (2) 호출 지점을 감싸는 `unsafe`.

```rust
// C 표준 라이브러리(libc)의 함수들을 선언한다.
// "C"는 C 호출 규약(calling convention)을 따르라는 지시다.
extern "C" {
    fn abs(input: i32) -> i32;
    fn sqrt(input: f64) -> f64;
}

fn main() {
    let x = -42;

    // 외부 C 함수 호출은 컴파일러가 검증할 수 없으므로 unsafe가 필요하다.
    // 이유: 시그니처가 실제 C 함수와 일치하는지 컴파일러가 보장 못 한다.
    let abs_x = unsafe { abs(x) };
    let root = unsafe { sqrt(16.0) };

    println!("abs({x}) = {abs_x}, sqrt(16.0) = {root}");
}
```

실제로 컴파일하고 실행하면 출력은 다음과 같습니다.

```text
abs(-42) = 42, sqrt(16.0) = 4
```

`extern "C"` 블록 안의 함수들은 **선언만** 합니다. 본문이 없습니다. 컴파일러는 "이 이름의 함수가 어딘가 C 라이브러리에 있고, C 규약으로 호출하면 된다"고만 알고, 실제 구현은 링커(linker)가 libc에서 찾아 연결합니다. libc는 시스템에 항상 링크되므로 별도 설정이 필요 없습니다.

왜 호출에 `unsafe`가 필요할까요? 컴파일러는 `extern` 블록의 선언이 *실제 C 함수와 일치한다고 믿을 뿐 검증하지 못하기* 때문입니다. 만약 여러분이 `fn abs(input: i64) -> i64`로 잘못 선언하면, 컴파일러는 항의하지 않고 그대로 호출 코드를 생성합니다. 그 결과는 잘못된 스택 레이아웃, 쓰레기 값, 혹은 세그폴트 — 즉 UB입니다. 이 시그니처의 정확성을 보장하는 것은 컴파일러가 아니라 **여러분**입니다. 그래서 `unsafe`입니다.

실무에서는 직접 `extern` 선언을 손으로 쓰는 대신 **`libc` 크레이트**를 씁니다. `libc`는 운영체제별 표준 C 함수와 타입(`size_t`, `c_int`, `pid_t`, `stat` 구조체 등)을 플랫폼에 맞게 미리 선언해 둔 크레이트입니다.

```rust
// Cargo.toml에 libc = "0.2" 추가 후
fn main() {
    let pid = unsafe { libc::getpid() };
    println!("현재 프로세스 PID: {pid}");
}
```

> **Under the Hood: 호출 규약(Calling Convention)이란 무엇인가**
>
> `extern "C"`의 `"C"`는 단순한 라벨이 아니라 **호출 규약(Calling Convention)**을 지정합니다. 호출 규약이란 함수를 부를 때 *기계어 수준에서* 지켜야 할 약속입니다. 구체적으로:
>
> - **인자를 어디에 둘 것인가.** x86-64 System V ABI(리눅스/macOS)에서 정수 인자는 차례로 `rdi, rsi, rdx, rcx, r8, r9` 레지스터에, 그 이상은 스택에 넣습니다. 부동소수점은 `xmm0~xmm7`에 넣습니다.
> - **반환값을 어디서 받을 것인가.** 정수 반환은 `rax`, 부동소수점은 `xmm0`.
> - **스택을 누가 정리하고, 어떤 레지스터를 호출자가 보존하는가(caller/callee-saved).**
>
> 호출자(Rust)와 피호출자(C)가 *서로 다른 규약*을 가정하면, 인자가 엉뚱한 레지스터에서 읽히고 스택이 어긋나 즉시 깨집니다. `extern "C"`는 "나는 C가 기대하는 그 규약대로 인자를 배치하겠다"는 선언입니다. Rust의 기본 규약은 `extern "Rust"`(불안정하며 컴파일러가 자유롭게 바꿀 수 있음)이고, FFI 경계에서는 반드시 안정된 `"C"`를 써야 양쪽이 같은 약속 위에서 만납니다. 그래서 FFI 함수 시그니처에는 거의 항상 `extern "C"`가 붙습니다.

---

## 21.5 Rust 함수를 C에 노출하기 — `#[no_mangle]`과 라이브러리 빌드

반대 방향, 즉 Rust로 짠 함수를 C(또는 파이썬, Go, 그 무엇이든 C ABI를 아는 언어)에서 호출하게 만드는 길을 봅시다. 두 가지가 필요합니다.

1. **`extern "C"`** 함수 정의 — C 호출 규약으로 호출되도록.
2. **`#[no_mangle]`** 속성 — 함수 이름을 그대로 심볼 테이블에 노출하도록.

```rust
// C에서 호출 가능한 Rust 함수
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn multiply(a: i32, b: i32) -> i32 {
    a * b
}
```

`#[no_mangle]`이 왜 필요한지가 이 절의 핵심입니다. Rust는 기본적으로 모든 함수 이름을 **이름 맹글링(name mangling)**합니다. 같은 이름의 함수가 다른 모듈/제네릭 인스턴스에 여러 개 존재할 수 있으므로, 컴파일러는 충돌을 막기 위해 크레이트 이름·모듈 경로·타입 정보·해시를 섞어 고유한 심볼 이름을 만듭니다.

> **Under the Hood: 맹글링된 심볼 vs `#[no_mangle]`**
>
> 다음 두 함수를 컴파일해 심볼 테이블을 `nm`으로 들여다보면 차이가 명백합니다.
>
> ```rust
> #[no_mangle]
> pub extern "C" fn add_unmangled(a: i32, b: i32) -> i32 { a + b }
>
> pub fn add_mangled(a: i32, b: i32) -> i32 { a + b }
> ```
>
> ```text
> $ nm mangle_test.o | grep -i add
> T _add_unmangled                                    ← 깔끔한 이름 그대로
> t __ZN11mangle_test11add_mangled17hdbb46d1c2dfb3984E ← 맹글링됨
> ```
>
> 맹글링된 이름 `_ZN11mangle_test11add_mangled17h...E`를 해독하면: `_ZN`(맹글 접두사) + `11mangle_test`(크레이트 이름, 길이 11 접두) + `11add_mangled`(함수 이름, 길이 11) + `17h...`(타입/제네릭 충돌 회피용 16자리 해시) + `E`(종료). C 컴파일러는 이런 이름을 모릅니다. C 코드는 `add`라는 심볼을 찾는데 라이브러리에는 `_ZN11mangle_test...`만 있으면 링커가 "undefined symbol"로 실패합니다. `#[no_mangle]`은 이 변환을 끄고 함수 이름을 *글자 그대로* 심볼로 내보내, C가 `add`로 찾을 수 있게 합니다. C++의 `extern "C" { }`가 C++ 맹글링을 끄는 것과 정확히 같은 역할입니다.
>
> (Rust 1.85+에서는 `#[unsafe(no_mangle)]` 형태도 도입되었으나, edition 2021에서는 `#[no_mangle]`이 표준입니다.)

심볼만 노출했다고 끝이 아닙니다. C가 링크할 수 있는 **라이브러리 형태**로 빌드해야 합니다. `Cargo.toml`에서 크레이트 타입을 지정합니다.

```toml
[package]
name = "mathlib"
version = "0.1.0"
edition = "2021"

[lib]
# cdylib: C에서 동적으로 링크하는 .so/.dylib/.dll
# staticlib: 정적으로 묻는 .a/.lib
crate-type = ["cdylib", "staticlib"]
```

| 크레이트 타입 | 산출물(리눅스/mac/win) | 용도 |
|---|---|---|
| `cdylib` | `.so` / `.dylib` / `.dll` | C/파이썬에서 런타임에 동적 로드. PyO3 확장 모듈이 이것. |
| `staticlib` | `.a` / `.lib` | C 실행 파일에 정적으로 묻어 단일 바이너리로. |
| `rlib` (기본) | `.rlib` | Rust↔Rust 전용. C는 못 읽음. |
| `dylib` | `.so` 등 | Rust↔Rust 동적 링크. C ABI 아님. |

`cargo build --release`를 돌리면 `target/release/`에 `libmathlib.so`(또는 `.dylib`/`.dll`)와 `libmathlib.a`가 생깁니다. C에서 쓰려면 헤더 파일도 필요한데, 이걸 손으로 쓰는 대신 **`cbindgen`** 크레이트가 Rust 소스를 스캔해 `.h` 헤더를 자동 생성해 줍니다. 반대로 C 헤더를 보고 Rust `extern` 선언을 자동 생성하는 도구는 **`bindgen`**입니다. 둘은 짝을 이룹니다.

```text
bindgen   :  C 헤더(.h)  ──→  Rust extern 선언   (C 라이브러리를 Rust에서 쓸 때)
cbindgen  :  Rust 소스   ──→  C 헤더(.h)          (Rust 라이브러리를 C에 줄 때)
```

`numpy`나 `openssl-sys` 같은 거대한 `-sys` 크레이트들은 거의 모두 `bindgen`으로 수천 줄의 `extern` 선언을 자동 생성합니다. 손으로 시그니처를 옮기다 한 글자만 틀려도 UB이므로, 자동화가 사실상 필수입니다.

---

## 21.6 메모리 레이아웃과 C 문자열 — `repr(C)`, `CString`/`CStr`

FFI에서 함수만큼 까다로운 것이 **데이터를 어떻게 주고받느냐**입니다. 두 가지 함정이 있습니다: 구조체의 메모리 레이아웃, 그리고 문자열.

### `repr(C)` — 메모리 레이아웃을 C에 맞추기

Rust는 기본적으로 구조체 필드의 메모리 배치 순서를 **컴파일러 마음대로** 정합니다(`repr(Rust)`). 패딩(padding)을 줄이려고 필드를 재배열하기 때문입니다. 그런데 C는 필드를 *선언 순서 그대로* 배치한다고 약속합니다. 둘이 다르면 같은 바이트를 서로 다르게 해석해 데이터가 깨집니다.

```rust
use std::mem::{size_of, align_of};

struct RustOrder {      // repr(Rust): 컴파일러가 재배열 자유
    a: u8,
    b: u32,
    c: u8,
}

#[repr(C)]
struct COrder {         // repr(C): 선언 순서·패딩 규칙을 C와 동일하게 고정
    a: u8,
    b: u32,
    c: u8,
}

fn main() {
    println!("RustOrder size={}, align={}", size_of::<RustOrder>(), align_of::<RustOrder>());
    println!("COrder    size={}, align={}", size_of::<COrder>(),    align_of::<COrder>());
}
```

실제 실행 결과입니다.

```text
RustOrder size=8, align=4
COrder    size=12, align=4
```

차이가 보이나요? **같은 세 필드인데 크기가 8바이트와 12바이트로 다릅니다.** `repr(Rust)`는 `a`와 `c`(각 1바이트)를 똑똑하게 모아 패딩을 줄여 8바이트로 압축했습니다. `repr(C)`는 C 규칙대로 `a`(1) + 패딩(3) + `b`(4) + `c`(1) + 패딩(3) = 12바이트로, 필드를 절대 재배열하지 않습니다.

> **Under the Hood: 왜 같은 구조체가 크기가 다른가**
>
> ```text
> repr(Rust) — 컴파일러가 a, c를 인접 배치해 패딩 절약:
>   offset 0: b (u32, 4B) │ 4: a (u8) │ 5: c (u8) │ 6~7: 패딩(2B)  →  총 8B
>
> repr(C) — 선언 순서 고정, 정렬 위해 패딩 삽입:
>   offset 0: a (u8) │ 1~3: 패딩(3B) │ 4: b (u32, 4B) │ 8: c (u8) │ 9~11: 패딩(3B)  →  총 12B
> ```
>
> C와 구조체를 주고받을 때 `#[repr(C)]`를 빠뜨리면, C는 12바이트 레이아웃을 기대하는데 Rust는 8바이트로 보내(혹은 필드 오프셋이 달라) `b`를 읽으려다 엉뚱한 바이트를 읽습니다. 이것은 컴파일러가 잡아주지 못하는 조용한 데이터 손상이며 UB입니다. **FFI 경계를 넘나드는 모든 구조체에는 `#[repr(C)]`가 필수입니다.** 같은 이유로 FFI용 열거형에는 `#[repr(C)]` 또는 `#[repr(i32)]`처럼 정수 표현을 명시해, C의 `enum`과 동일한 너비·값을 갖도록 고정합니다.

### C 문자열 — `CString`과 `CStr`

두 번째 함정은 문자열입니다. Rust의 `String`/`str`과 C의 문자열은 근본적으로 다릅니다.

- **Rust `str`**: (주소, 길이) 쌍으로 길이를 *명시적으로* 저장. UTF-8. 내부에 `\0`이 있어도 됨.
- **C 문자열**: 그냥 `char*`. 길이 정보가 없고, **널 종료 바이트(`\0`)**가 나올 때까지가 문자열. 내부에 `\0`이 있으면 거기서 잘림.

이 간극을 메우는 타입이 `std::ffi::CString`(소유, 힙 할당)과 `CStr`(빌린 참조)입니다.

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

fn main() {
    // Rust String → C 문자열: 끝에 \0을 붙이고, 내부에 \0이 있으면 에러 반환
    let rust_str = "안녕, FFI";
    let c_string = CString::new(rust_str).expect("내부에 NUL 바이트 없어야 함");

    // C 함수에 넘길 포인터. c_string이 살아있는 동안만 유효!
    let ptr: *const c_char = c_string.as_ptr();

    // C 문자열 포인터 → Rust: \0까지 읽어 &CStr로
    let back: &CStr = unsafe { CStr::from_ptr(ptr) };
    println!("round-trip: {}", back.to_str().unwrap());

    // c_string은 여기서 살아있으므로 ptr는 유효했다.
}
```

실행 결과는 `round-trip: 안녕, FFI`입니다. `CString::new`가 `Result`를 반환하는 이유는, 입력 문자열 중간에 `\0`이 있으면 C 문자열로 변환할 수 없기 때문입니다(거기서 잘려버리므로). Rust는 이 위험을 타입으로 강제해 컴파일 타임이 아니라 변환 시점에 명시적으로 처리하게 만듭니다.

여기 **가장 악명 높은 FFI 함정**이 숨어 있습니다. `CString`의 수명 관리입니다.

```rust
use std::ffi::CString;
use std::os::raw::c_char;

// 🚨 위험한 패턴! 댕글링 포인터를 반환한다
fn dangling() -> *const c_char {
    let s = CString::new("hello").unwrap();
    s.as_ptr() // s는 이 함수가 끝나는 순간 drop → 그 포인터는 해제된 메모리를 가리킴
} // <- 여기서 s가 해제됨. 반환된 포인터는 댕글링.
```

`s.as_ptr()`은 `CString`이 소유한 힙 버퍼의 주소를 줍니다. 그런데 함수가 끝나면 `s`가 `Drop`되어 그 버퍼가 해제됩니다. 반환된 포인터는 이미 죽은 메모리를 가리키는 **댕글링 포인터**입니다. 이걸 C가 역참조하면 use-after-free, 즉 UB입니다. 무서운 점은 **이 코드가 컴파일된다**는 것입니다. `as_ptr`은 원시 포인터를 *만들기*만 하므로 safe이고, Rust의 수명 검사는 원시 포인터까지는 추적하지 못합니다. 올바른 패턴은 `CString`을 호출자가 살려두거나, 소유권을 C로 넘긴 뒤(`CString::into_raw`) 나중에 반드시 `CString::from_raw`로 회수해 해제하는 것입니다. 이 함정은 21.9에서 다시 다룹니다.

---

## 21.7 Hands-on — 안전한 추상화로 unsafe 봉인하기

지금까지의 조각을 모아 실전 패턴을 완성합시다. 목표는 std와 똑같은 철학입니다. **`unsafe`를 함수 내부 깊숙이 가두고, 외부에는 안전하고 평범한 Rust API만 노출한다.** C 라이브러리에서 받은 정수 배열의 합을 구하는 함수를 Rust로 안전하게 감싸는 시나리오입니다.

C 라이브러리가 다음과 같은 함수를 제공한다고 합시다(`extern` 선언 형태). 이 함수는 `(포인터, 길이)` 쌍으로 배열을 받는 전형적인 C 스타일입니다.

```rust
use std::os::raw::{c_int, c_double};
use std::slice;

// --- 외부 C 라이브러리 선언 (실제로는 -sys 크레이트나 bindgen 생성물) ---
// double sum_array(const double* data, size_t len);
extern "C" {
    fn sum_array(data: *const c_double, len: usize) -> c_double;
}

// --- 안전한 래퍼: 외부에 노출되는 것은 이것뿐 ---
// unsafe가 함수 내부에 완전히 봉인되어 있다.
pub fn safe_sum(numbers: &[f64]) -> f64 {
    // 슬라이스는 (포인터, 길이)를 이미 안다. 이를 C 규약으로 풀어 넘긴다.
    // 안전 보장: numbers는 살아있는 유효한 슬라이스이고,
    //           .as_ptr()와 .len()이 일관되므로 C 함수의 계약을 충족한다.
    unsafe { sum_array(numbers.as_ptr(), numbers.len()) }
}

// --- 반대 방향: Rust 함수를 C에 노출하면서 내부에서 원시 포인터를 안전하게 복원 ---
// C가 (포인터, 길이)를 주면, Rust 슬라이스로 되살려 안전 코드로 처리한다.
#[no_mangle]
pub unsafe extern "C" fn rust_sum(data: *const c_int, len: usize) -> c_int {
    // 호출자(C)가 보장해야 할 계약을 문서화한다.
    // # Safety
    // - data는 len개의 유효한 i32를 담은, 정렬된 메모리를 가리켜야 한다.
    // - 호출 동안 data가 가리키는 메모리는 변경/해제되지 않아야 한다.
    if data.is_null() {
        return 0;
    }
    // 단 한 줄의 unsafe로 원시 포인터를 안전한 슬라이스로 승격.
    let slice = slice::from_raw_parts(data, len);
    // 이 아래부터는 100% 안전한 Rust. iterator, sum 등 마음껏.
    slice.iter().sum()
}
```

이 코드의 설계 의도를 읽어봅시다.

`safe_sum`은 호출자에게 `unsafe`를 강요하지 않습니다. 시그니처는 평범한 `pub fn safe_sum(numbers: &[f64]) -> f64`입니다. 위험한 C 호출은 함수 몸통의 단 한 줄, `unsafe { sum_array(...) }`에 갇혀 있습니다. 이 한 줄이 안전한 이유를 우리는 *증명할 수 있습니다*: `numbers`는 살아 있는 유효한 슬라이스이고, `as_ptr()`가 주는 포인터와 `len()`이 주는 길이는 정의상 일관됩니다. 따라서 C 함수가 `[ptr, ptr+len)` 범위를 읽는 것은 안전합니다. 이 추론을 한 번 검증해 두면, 이후 `safe_sum`을 호출하는 수백 곳은 안전을 신경 쓸 필요가 없습니다.

`rust_sum`은 반대 방향의 정석입니다. C에서 넘어온 `*const c_int`는 신뢰할 수 없습니다. 그래서 (1) 널 체크를 먼저 하고, (2) `slice::from_raw_parts`로 원시 포인터를 슬라이스로 *승격*시킨 다음, (3) 그 아래부터는 `iter().sum()` 같은 100% 안전한 Rust로 처리합니다. 여기서 `from_raw_parts`는 `unsafe` 함수입니다 — 포인터와 길이가 실제로 유효한 메모리를 가리킨다는 보장은 컴파일러가 못 하고 호출자(여기서는 C)가 해야 하므로. 그래서 함수 전체를 `unsafe extern "C" fn`으로 선언하고, `# Safety` 주석으로 **호출자가 지켜야 할 계약**을 명문화했습니다. 이것이 FFI 함수의 표준 문서화 관례입니다.

호출 측 데모와 함께 완전한 실행 가능 예제로 묶으면 다음과 같습니다(C 함수 대신 Rust로 구현한 더미를 같은 ABI로 제공해 단독 실행되게 한 버전).

```rust
use std::os::raw::c_double;
use std::slice;

// 데모용: 실제 C 함수 자리에 동일 ABI의 Rust 구현을 놓아 단독 실행 가능하게.
#[no_mangle]
pub unsafe extern "C" fn sum_array(data: *const c_double, len: usize) -> c_double {
    if data.is_null() || len == 0 {
        return 0.0;
    }
    slice::from_raw_parts(data, len).iter().sum()
}

// 외부엔 안전한 API만 노출.
pub fn safe_sum(numbers: &[f64]) -> f64 {
    unsafe { sum_array(numbers.as_ptr(), numbers.len()) }
}

fn main() {
    let data = vec![1.5, 2.5, 3.0, 4.0];
    let total = safe_sum(&data); // 호출자는 unsafe를 전혀 보지 않는다
    println!("합계 = {total}"); // 합계 = 11
}
```

`cargo run`하면 `합계 = 11`이 출력됩니다. 핵심 교훈: **`unsafe`의 표면적을 최소화하라.** `unsafe`는 한 줄, 안전한 API는 그 위를 넓게 덮는다. 이것이 `Vec`, `String`, `Mutex`, 그리고 다음 장의 PyO3가 모두 따르는 패턴입니다.

성능 관점에서 이 래핑은 **무비용(zero-cost)**입니다. `safe_sum`은 인라인되면 `sum_array` 직접 호출과 동일한 기계어로 컴파일됩니다. 안전 추상화를 위해 런타임에 추가로 지불하는 비용은 없습니다. 데이터 과학 워크로드에서 파이썬 루프 대비 Rust FFI 호출이 보통 한 자릿수~수십 배 빠른 이유는, 객체 포장 비용(21.1의 Under the Hood)이 사라지고 연속 메모리를 캐시 친화적으로 순회하기 때문입니다. (정확한 배수는 입력 크기와 `--release` 빌드 여부에 달려 있으니, 과장된 "100배" 같은 주장은 경계하십시오.)

---

## 21.8 Best Practices & 안티패턴

`unsafe`와 FFI는 Rust에서 가장 날카로운 도구입니다. 실무 체크리스트로 정리합니다.

**이렇게 하라.**

- **`unsafe`를 최소 표면적으로 가두라.** 함수 전체가 아니라 위험한 연산 한 줄만 `unsafe` 블록으로 감싸고, 그 위를 안전한 API로 덮으십시오. `grep -rn unsafe`로 감사 대상을 좁힐 수 있어야 합니다.
- **모든 `unsafe fn`과 `unsafe` 블록에 `// SAFETY:` 주석을 달라.** *왜* 이 `unsafe`가 실제로는 안전한지(어떤 불변식이 성립하므로)를 적으십시오. Clippy의 `clippy::undocumented_unsafe_blocks` 린트가 이를 강제할 수 있습니다.
- **FFI 경계의 모든 구조체에 `#[repr(C)]`를, 열거형에 `#[repr(C)]`/`#[repr(i32)]`를 붙이라.** 레이아웃 호환은 선택이 아니라 필수입니다.
- **C 함수 시그니처는 `bindgen`으로 자동 생성하라.** 손으로 옮기면 한 글자 오타가 UB입니다.
- **Rust→C 노출 함수는 `extern "C"` + `#[no_mangle]` + `pub`** 세 가지를 모두 갖추라.
- **C로 넘긴 소유권은 반드시 회수하라.** `CString::into_raw`로 넘겼으면 `from_raw`로, `Box::into_raw`로 넘겼으면 `Box::from_raw`로 짝을 맞춰 해제하십시오.
- **`std::os::raw`/`libc`의 C 타입 별칭을 쓰라.** `c_int`, `c_char`, `c_void`는 플랫폼별로 너비가 다를 수 있으니 직접 `i32`로 박지 마십시오.

**이건 피하라(안티패턴).**

- **귀찮다고 함수 전체를 `unsafe`로 만들지 말라.** 위험 영역이 넓어져 감사가 불가능해집니다.
- **`unsafe`로 빌림 검사기를 "이기려" 하지 말라.** "컴파일러가 자꾸 막으니 `unsafe`로 우회한다"는 거의 항상 설계 오류의 신호입니다. `unsafe`는 검사기를 끄지도 않습니다(21.2).
- **`transmute`를 함부로 쓰지 말라.** 타입 간 비트 재해석은 FFI에서도 거의 필요 없고, 레이아웃 가정이 틀리면 UB입니다. 대부분 `as` 캐스트나 `from_raw_parts`로 충분합니다.
- **`static mut`을 새 코드에 쓰지 말라.** 데이터 레이스를 컴파일러가 막아주지 못합니다. `OnceLock`, `LazyLock`, 혹은 `AtomicUsize` 같은 안전한 대안을 쓰십시오(21.9 참고).
- **C에서 받은 포인터를 널 체크 없이 역참조하지 말라.**

> **Under the Hood: 미정의 동작(UB)은 "그냥 크래시"가 아니다**
>
> 파이썬 개발자에게 가장 직관에 반하는 개념이 UB입니다. UB는 "프로그램이 죽는다"가 아니라, **"컴파일러가 이 상황은 절대 일어나지 않는다고 가정하고 최적화할 권리를 갖는다"**는 뜻입니다. 예를 들어 댕글링 포인터 역참조가 UB라면, 옵티마이저는 "역참조했으니 그 포인터는 유효하다"고 *역으로 추론*해 주변 코드를 재배열하거나, 널 체크를 제거하거나, 무관해 보이는 변수를 망가뜨릴 수 있습니다. UB의 증상은 디버그 빌드에서는 멀쩡하다가 `--release`에서만, 혹은 특정 CPU에서만, 혹은 함수 호출 순서를 바꾸면 사라지는 식으로 나타납니다. 그래서 UB는 디버깅이 악몽입니다. `unsafe`의 책임이란 결국 **"UB를 절대 일으키지 않겠다"는 약속**입니다. 이 약속을 검증하는 도구가 있습니다: `cargo +nightly miri run`은 인터프리터로 코드를 돌리며 UB를 탐지하고, `cargo test`를 ASan/TSan과 함께 돌리면 메모리·스레드 오류를 잡습니다. `unsafe` 코드를 작성했다면 Miri로 검증하는 것이 사실상의 표준입니다.

---

## 21.9 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1: `unsafe`가 검사기를 끈다고 착각한다.** 가장 흔한 오해입니다. "빌림 검사기가 막으니 `unsafe`로 감싸면 되겠지"라고 생각하지만, `unsafe` 안에서도 소유권과 대여 규칙은 그대로 작동합니다(21.2). `unsafe`는 21.2에 나열한 다섯 능력만 잠금 해제할 뿐입니다. 빌림 에러를 `unsafe`로 우회하려는 시도는 컴파일조차 안 됩니다. 해결책은 `unsafe`가 아니라 설계를 고치는 것입니다.

**함정 2: `CString`을 너무 빨리 떨어뜨린다(가장 악명 높은 FFI 버그).** 21.6에서 본 댕글링 포인터입니다. 파이썬에서 `c_string = create_string(); ptr = ffi.cast(...)` 식으로 임시 객체의 포인터를 받아 쓰는 습관이 그대로 옮겨오면 사고가 납니다.

```rust
use std::ffi::CString;

extern "C" {
    fn puts(s: *const std::os::raw::c_char) -> i32;
}

fn main() {
    // 🚨 미묘한 버그! as_ptr 결과만 살리고 CString은 즉시 drop
    let ptr = CString::new("hello").unwrap().as_ptr();
    //         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 이 임시 CString은 이 줄 끝에서 drop
    unsafe { puts(ptr); } // ptr는 이미 해제된 메모리를 가리킴 → UB

    // 올바른 방법: CString을 바인딩해 살려둔다
    let c = CString::new("world").unwrap(); // c가 살아있음
    unsafe { puts(c.as_ptr()); }            // 안전: c는 다음 줄까지 유효
}
```

규칙은 단순합니다. **C에 포인터를 넘기는 동안 그 포인터가 가리키는 `CString`을 명명된 변수로 살려두라.** 임시 표현식 끝에 `.as_ptr()`를 붙이는 순간 함정입니다.

**함정 3: `static mut`으로 전역 카운터를 만든다.** 파이썬의 전역 변수 습관이 부르는 사고입니다. Rust에서 `static mut`은 접근 자체가 `unsafe`이며, 멀티스레드에서 데이터 레이스로 직행합니다.

```rust
static mut COUNTER: u32 = 0;

fn main() {
    COUNTER += 1; // 🚨 컴파일 에러!
    println!("{}", COUNTER);
}
```

```text
error[E0133]: use of mutable static is unsafe and requires
              unsafe function or block
 --> src/main.rs:4:5
  |
4 |     COUNTER += 1;
  |     ^^^^^^^ use of mutable static
  |
  = note: mutable statics can be mutated by multiple threads:
          aliasing violations or data races will cause undefined behavior
```

`E0133`은 21.3의 원시 포인터 역참조와 같은 에러 코드입니다 — 둘 다 "사람이 안전을 보장해야 하는 연산"이기 때문입니다. `unsafe`로 감싸면 컴파일은 되지만, 그건 데이터 레이스를 합법화할 뿐입니다. 올바른 해법은 안전한 동시성 프리미티브입니다(Chapter 17 참고).

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0); // mut 아님, unsafe 불필요

fn main() {
    COUNTER.fetch_add(1, Ordering::Relaxed); // 원자적, 스레드 안전
    println!("{}", COUNTER.load(Ordering::Relaxed));
}
```

**함정 4: C 타입 너비를 잘못 가정한다.** C의 `long`은 64비트 리눅스에서 8바이트지만 윈도우에서는 4바이트입니다. `int`를 `i64`로 선언하거나 `size_t`를 `u32`로 박으면 플랫폼에 따라 조용히 깨집니다. 항상 `c_int`, `c_long`, `usize` 같은 정확한 별칭을 쓰십시오.

**함정 5: 소유권을 C에 넘기고 회수를 잊는다.** `Box::into_raw`나 `CString::into_raw`로 Rust 객체의 소유권을 C로 넘기면, Rust의 자동 `Drop`이 *멈춥니다*. C가 그 포인터를 다 쓴 뒤 반드시 Rust로 돌려주어 `from_raw`로 회수하고 해제해야 합니다. 이 짝을 맞추지 않으면 GC 없는 세계의 고전적 메모리 누수입니다. 파이썬 C-API의 `Py_DECREF` 누락과 정확히 같은 종류의 버그가, 형태만 바꿔 다시 나타나는 셈입니다.

---

이번 장에서 우리는 Rust의 안전 보장이 끝나는 경계로 걸어가, 그 너머의 C 세계와 대화하는 법을 배웠습니다. `unsafe`는 안전을 끄는 스위치가 아니라 다섯 가지 능력을 잠금 해제하는 좁은 게이트이며, 원시 포인터는 컴파일러의 보장만 벗겨낸 C 포인터이고, `extern "C"`·`#[no_mangle]`·`#[repr(C)]`는 호출 규약·이름 맹글링·메모리 레이아웃이라는 ABI 3대 약속을 양쪽에서 맞추는 장치였습니다. 그리고 그 모든 위험을 안전한 추상화 뒤에 봉인하는 패턴 — std가, 그리고 곧 만날 PyO3가 따르는 그 패턴 — 이 핵심이었습니다.

20장에서 본 `Box`/`Rc`/`RefCell` 같은 스마트 포인터가 안전한 Rust 안에서 메모리를 다루는 도구였다면, 이번 장의 `unsafe`와 FFI는 그 안전 영역 *바깥*과 통신하는 도구였습니다. 이제 두 세계를 잇는 다리를 직접 놓을 차례입니다. 21.3에서 본 파이썬 C-API의 참조 카운팅 지옥, 21.6의 `repr(C)`와 `CString` 줄다리기, 21.7의 안전 래퍼 패턴 — 이 모든 것을 누군가 이미 `unsafe` 블록 안에 완벽하게 검증해 봉인하고, 우리에게는 매크로 한 줄과 안전한 구조체만 건네준다면 어떨까요? 그 포장지의 이름이 바로 이 가이드의 최종 목적지, **PyO3**입니다. 다음 장에서 파이썬과 Rust가 물리적으로 하나의 `.so` 파일이 되는 과정을 직접 만들어 보겠습니다.
