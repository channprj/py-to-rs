# Chapter 21. C FFI와 Unsafe Rust

파이썬이 오늘날 AI와 데이터 과학의 지배자가 된 이유는 파이썬 인터프리터 자체가 빠르기 때문이 아닙니다. 파이썬은 **"세상에서 가장 다루기 쉬운 C 언어 래퍼(Wrapper)"**이기 때문입니다. `numpy`, `tensorflow`, 심지어 파이썬의 표준 라이브러리 상당수가 내부적으로 C/C++로 작성되어 파이썬의 `C-API`를 통해 연결되어 있습니다.

이처럼 한 언어에서 다른 언어의 함수를 호출하는 인터페이스를 **FFI(Foreign Function Interface)**라고 부릅니다. Rust 역시 C 언어와 완벽하게 통신할 수 있는 FFI 기능을 제공합니다. 하지만 C 언어의 세계로 넘어가는 순간, Rust 컴파일러가 자랑하던 완벽한 메모리 안전성은 보장될 수 없습니다.

이때 등장하는 것이 바로 **`unsafe`** 키워드입니다.

---

## 21.1 `unsafe`의 진정한 의미

파이썬 개발자들이 `unsafe`라는 단어를 들으면 "이 코드는 위험하니까 쓰지 말라"는 뜻으로 오해하곤 합니다. 하지만 Rust에서 `unsafe`의 진짜 의미는 다음과 같습니다.

> *"이 블록 안에서 일어나는 일은 컴파일러인 내가 100% 검증할 수 없다. 그러니 개발자인 네가 메모리 안전성을 직접 책임져라."*

Rust 컴파일러는 C 언어로 작성된 라이브러리 내부에서 포인터가 어떻게 움직이는지 알 턱이 없습니다. 따라서 외부 C 함수를 호출할 때는 무조건 `unsafe` 블록 안에서 호출해야 합니다.

```rust
// C 표준 라이브러리의 abs 함수를 선언합니다.
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    let x = -3;
    // C 함수를 호출하는 것은 컴파일러가 보장할 수 없으므로 unsafe 블록이 필요합니다.
    let abs_x = unsafe {
        abs(x)
    };
    println!("Absolute value of {} is {}", x, abs_x);
}
```

### `unsafe`가 할 수 있는 4가지 마법
`unsafe` 블록 안에서는 평소에 금지되었던 4가지 슈퍼파워를 쓸 수 있습니다.
1. 원시 포인터(Raw Pointer, `*const T`, `*mut T`)의 역참조(Dereference)
2. `unsafe` 함수나 C FFI 함수 호출
3. 가변 정적 변수(`static mut`)의 접근 및 수정
4. `unsafe` 트레이트 구현

---

## 21.2 파이썬 C-API의 한계와 메모리 누수

파이썬 생태계에 내 코드를 C-Extension으로 제공하려면 `Python.h`를 임포트하고 수많은 보일러플레이트를 작성해야 합니다.

* **The Old Python Way (C Extension)**
  ```c
  #include <Python.h>

  static PyObject* my_module_add(PyObject* self, PyObject* args) {
      int a, b;
      // 파이썬 객체 파싱
      if (!PyArg_ParseTuple(args, "ii", &a, &b)) {
          return NULL;
      }
      return PyLong_FromLong(a + b);
  }
  ```
  이 과정에서 `Py_INCREF`나 `Py_DECREF` 같은 레퍼런스 카운팅 함수를 하나라도 빼먹으면 심각한 메모리 릭(Leak)이나 Segmentation Fault가 발생합니다. 파이썬 C-Extension 개발이 악명 높은 이유입니다.

---

## 21.3 안전한 탈출구: Rust가 래핑하는 C-API

Rust 생태계는 `unsafe`의 철학을 극한으로 활용합니다. 

Rust 개발자들은 C 언어 라이브러리(예: `OpenSSL`, `zlib`, 파이썬 C-API 등)를 직접 `unsafe`하게 호출하는 원시 크레이트(`-sys` 크레이트)를 먼저 만듭니다. 그런 다음, 이 원시 호출들을 **Rust의 소유권 모델과 타입 시스템으로 완벽하게 감싸서(Wrapping) 안전한(Safe) 인터페이스**만 외부에 노출시킵니다.

이러한 패턴 덕분에, 현대의 러스트 개발자들은 파이썬 C-API의 포인터나 레퍼런스 카운팅을 직접 만질 필요가 없습니다. 누군가 이미 `unsafe` 블록 안에서 파이썬 인터프리터와의 통신을 완벽하게 검증하고, 우리가 쓰기 편한 안전한 매크로와 구조체로 포장해 두었기 때문입니다.

이 놀라운 포장지의 이름이 바로 이 가이드의 최종 목적지, **`PyO3`**입니다. 다음 장에서 파이썬과 Rust가 물리적으로 하나가 되는 기적을 확인해 보겠습니다.
