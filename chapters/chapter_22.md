# Chapter 22. PyO3: Python과 Rust의 완벽한 결합

축하합니다. 여기까지 왔습니다. 지난 21개 챕터 동안 우리는 파이썬의 한계(GIL, GC Pause, 덕 타이핑(Duck Typing)의 런타임 오버헤드)와 Rust의 해법(소유권(Ownership), 무비용 추상화(Zero-cost Abstraction), 두려움 없는 동시성(Fearless Concurrency))을 메모리 바이트와 어셈블리 수준까지 파헤쳤습니다. 그리고 바로 앞 Chapter 21에서는 Rust가 C 세계와 통신하는 FFI(Foreign Function Interface)와 `unsafe`의 진짜 의미 — "컴파일러가 검증 못 하니 네가 책임져라" — 를 배웠습니다. 그 챕터의 마지막 문장은 이렇게 끝났습니다. "다음 장에서 파이썬과 Rust가 물리적으로 하나가 되는 기적을 확인해 보겠습니다."

이제 그 기적을 직접 만들어 볼 차례입니다. 하지만 그 전에, 한 가지 오해부터 정정하고 싶습니다. 이 책을 끝까지 읽은 여러분이 다음 주 월요일에 출근해서 "이제부터 우리 서비스를 전부 Rust로 다시 짜겠습니다"라고 선언한다면, 그것은 이 책이 가르치려던 것을 정반대로 이해한 것입니다. 10년 차 엔지니어가 잘 압니다. 비즈니스 로직의 90%는 I/O 대기, DB 쿼리, 외부 API 호출, 인증, 직렬화 같은 "느려도 상관없는" 코드이고, 그 영역에서는 파이썬의 생산성과 생태계(Django, FastAPI, SQLAlchemy, pandas)가 압도적입니다. CPU가 99% 노는 핸들러를 Rust로 다시 짜는 것은 시간 낭비를 넘어 죄악입니다.

진짜 병목은 항상 좁은 구간에 숨어 있습니다. 수백만 행의 로그를 파싱하는 루프, JSON을 검증하는 핫패스, 토큰을 자르는 전처리, 수치를 집계하는 집합 연산. 프로파일러를 돌리면 전체 시간의 80%가 코드의 5%에서 소모되고 있음을 발견하게 됩니다. 그 5%만 도려내서 Rust로 교체하고, 나머지 95%는 파이썬 그대로 두는 것 — 이것이 이 챕터의 전부이자, 현대 파이썬 고성능 라이브러리들이 모두 채택한 전략입니다. `pydantic-core`(검증), `polars`(데이터프레임), `tokenizers`(HuggingFace 전처리), `ruff`(린터), `cryptography`(암호), `orjson`(JSON) — 이 라이브러리들의 공통점은 단 하나입니다. **파이썬 API를 입은 Rust 엔진**이라는 것.

이번 챕터에서는 그 엔진을 만들어주는 도구 **PyO3**와 빌드 시스템 **maturin**을 다룹니다. 단순히 "함수를 노출하는 법"에 그치지 않고, PyO3가 Chapter 21에서 본 `unsafe` C-API 위에 어떻게 안전한 래퍼를 씌우는지, `py.allow_threads()`가 GIL을 풀었을 때 실제로 멀티코어가 어떻게 돌아가는지, numpy 버퍼를 제로카피(Zero-copy)로 빌려오는 비용이 얼마인지까지 내부로 파고듭니다. 그리고 이 책의 마지막 챕터인 만큼, 마지막 절에서는 여러분이 다음에 무엇을 할지 — 추천 학습 경로와 실전 적용 전략 — 으로 책 전체를 마무리하겠습니다.

---

## 22.1 The Python Way — C로 내려가는 두 갈래 길과 그 고통

10년 차 파이썬 엔지니어라면 "이 함수가 너무 느린데 어떡하지"라는 질문에 대한 표준 답안들을 이미 알고 있을 것입니다. 하나씩 그 한계를 짚어봅시다.

**1번 길: 그냥 파이썬으로 버틴다.** 가장 흔하고 가장 오래가는 선택입니다. 다음은 전형적인 병목 코드입니다. 센서 측정값 로그(`"id,value"` 형식의 문자열 수백만 줄)를 파싱해서 그룹별 합계를 내는 코드를 봅시다.

```python
from __future__ import annotations

def aggregate(lines: list[str]) -> dict[int, float]:
    totals: dict[int, float] = {}
    for line in lines:
        id_str, value_str = line.split(",")
        sensor_id = int(id_str)
        value = float(value_str)
        totals[sensor_id] = totals.get(sensor_id, 0.0) + value
    return totals
```

이 코드는 명료하고 정확합니다. 문제는 속도입니다. 1,000만 줄을 처리하면 이 루프는 수 초가 걸립니다. 왜일까요? 루프의 매 반복마다 일어나는 일을 메모리 관점에서 분해해 봅시다. `line.split(",")`은 새 리스트와 두 개의 새 `str` 객체를 힙(Heap)에 할당합니다. `int(id_str)`과 `float(value_str)`은 각각 새 `PyObject`를 박싱(boxing)합니다 — 파이썬에서 정수 하나는 28바이트, 부동소수점 하나는 24바이트짜리 힙 객체입니다(Chapter 3에서 본 `PyObject` 헤더의 비용). `totals.get(...)`은 해시를 계산하고, 더하기 연산은 또 새 `float` 객체를 만듭니다. 한 줄당 대여섯 번의 힙 할당과 그만큼의 참조 카운팅(`Py_INCREF`/`Py_DECREF`)이 발생합니다. 게다가 이 모든 것이 GIL 아래에서 단일 코어로만 돕니다. CPU가 8개여도 7개는 놀고 있습니다.

**2번 길: 직접 C 확장(C Extension)을 쓴다.** Chapter 21에서 맛본 그 길입니다. 다시 떠올려 봅시다.

```c
#include <Python.h>

static PyObject* my_aggregate(PyObject* self, PyObject* args) {
    PyObject* lines;
    if (!PyArg_ParseTuple(args, "O", &lines)) {
        return NULL;  // 파싱 실패 시 NULL 반환 — 빼먹으면 크래시
    }
    // ... PyList_Size, PyList_GetItem, PyUnicode_AsUTF8 ...
    // ... Py_INCREF / Py_DECREF 를 한 곳이라도 빠뜨리면 메모리 릭 또는 Segfault ...
}
```

C 확장은 빠르지만, Chapter 21에서 확인했듯 `Py_INCREF`/`Py_DECREF`를 한 번이라도 빠뜨리면 메모리 릭이나 세그폴트(Segmentation Fault)가 납니다. 빌림(borrowed) 참조와 소유(owned) 참조의 규약을 외워야 하고, 예외 전파를 수동으로 `NULL` 체크로 처리해야 하며, 빌드 설정(`setup.py`, 헤더 경로, ABI 호환)은 그 자체로 또 하나의 지옥입니다.

**3번 길: Cython.** C 확장의 보일러플레이트를 줄여주지만, 결국 파이썬도 C도 아닌 제3의 언어를 배워야 하고, 타입 어노테이션을 `cdef`로 일일이 달아야 성능이 나오며, 메모리 안전성은 여전히 보장되지 않습니다.

세 길 모두 막다른 골목이거나, 위험한 외나무다리입니다. PyO3는 네 번째 길을 제시합니다. **Rust의 소유권 시스템으로 메모리 안전이 보장되면서, C 확장만큼 빠르고, C 확장보다 훨씬 적은 코드로** 파이썬 모듈을 만드는 길입니다.

여기서 한 가지를 분명히 해둡시다. PyO3로 옮길 가치가 있는 코드에는 공통된 프로파일이 있습니다. (1) **CPU 바운드**일 것 — I/O 대기가 대부분이면 언어를 바꿔도 시계는 그대로 흐릅니다. (2) **호출 빈도 대비 함수 본체가 무거울 것** — 마이크로초짜리 함수를 수백만 번 부르면 FFI 경계 오버헤드가 이득을 잡아먹습니다. (3) **입출력 자료형이 단순하거나 numpy 같은 버퍼로 표현될 것** — 복잡한 파이썬 객체 그래프를 주고받으면 변환 비용이 본체를 압도합니다. 22.1의 센서 로그 집계는 이 세 조건을 모두 만족하는, PyO3의 교과서적 대상입니다. 반대로 "DB에서 읽어 JSON으로 응답하는 핸들러"는 셋 다 어긋나므로 손대지 않는 것이 옳습니다.

---

## 22.2 The Rust Mental Model — 안전한 래퍼는 이미 짜여 있다

Chapter 21에서 우리는 Rust 생태계의 핵심 패턴을 배웠습니다. 누군가 `unsafe`로 C 라이브러리를 호출하는 원시 `-sys` 크레이트를 만들고, 그 위에 소유권과 타입으로 검증된 안전한 래퍼를 씌워서, 사용자는 `unsafe`를 한 글자도 쓰지 않고 안전하게 쓰는 것. PyO3는 바로 이 패턴을 파이썬 C-API 전체에 적용한 결과물입니다.

`PyO3`의 멘탈 모델을 한 문장으로 요약하면 이렇습니다. **"파이썬 인터프리터와의 모든 위험한 상호작용(참조 카운팅, GIL, 예외 전파)을 Rust의 타입 시스템 속으로 끌어들여, 컴파일러가 강제하게 만든다."** 파이썬 C-API의 세 가지 함정이 PyO3에서 각각 어떤 Rust 타입으로 봉쇄되는지 봅시다.

- **참조 카운팅의 함정** → 스마트 포인터로 봉쇄. C에서 손으로 세던 `Py_INCREF`/`Py_DECREF`를 PyO3에서는 `Py<T>`라는 스마트 포인터(Chapter 20의 `Rc`와 같은 원리)가 RAII로 자동 처리합니다. 객체가 스코프를 벗어나면 `Drop`이 `Py_DECREF`를 호출합니다. 빼먹을 방법이 없습니다.
- **GIL의 함정** → 토큰 타입으로 봉쇄. "지금 GIL을 쥐고 있는가?"라는 사실을 `Python<'py>`라는 토큰 타입과 수명(Lifetime, Chapter 6)으로 표현합니다. 파이썬 객체를 만지는 함수는 이 토큰을 인자로 받아야만 합니다. GIL 없이 파이썬 객체를 건드리는 코드는 **컴파일이 안 됩니다.**
- **예외 전파의 함정** → `Result`로 봉쇄. C에서 `NULL` 반환과 에러 플래그로 어설프게 흉내 내던 예외 전파를, PyO3는 Chapter 12~13에서 마스터한 `Result<T, PyErr>`(별칭 `PyResult<T>`)와 `?` 연산자로 처리합니다. Rust 에러가 파이썬 예외로 자동 변환됩니다.

다음 비교표가 이 챕터에서 다룰 PyO3의 도구 전체를 파이썬 개념과 매핑합니다. 이 표를 머릿속에 넣어두면 나머지 절들이 전부 이 표의 각주처럼 읽힙니다.

| Python 개념 | PyO3 / Rust 대응 | 비고 |
|------|------|------|
| `def func(...)` 모듈 함수 | `#[pyfunction]` | C-API 호출 규약으로 변환 |
| `class Foo:` | `#[pyclass] struct Foo` | Rust 구조체가 파이썬 클래스가 됨 |
| `def method(self)` | `#[pymethods] impl Foo` | 메서드 묶음 |
| `@property` getter/setter | `#[getter]` / `#[setter]` | Chapter 7의 캡슐화와 동일 |
| `import mymodule` | `#[pymodule] fn mymodule(...)` | 모듈 진입점 |
| `raise ValueError(...)` | `PyValueError::new_err(...)` | Rust 에러 → 파이썬 예외 |
| `try/except` 경계 | `PyResult<T>` + `?` | Chapter 12~13의 `Result` |
| 참조 카운팅(GC) | `Py<T>` 스마트 포인터 | RAII 자동 `DECREF` |
| GIL 보유 상태 | `Python<'py>` 토큰 | 수명으로 검증 |
| `list` / `dict` / `str` | `Vec<T>` / `HashMap` / `String` | 자동 변환(복사) |
| `numpy.ndarray` | `PyReadonlyArray1<f64>` | 제로카피 가능(`numpy` 크레이트) |
| `threading` (GIL에 묶임) | `py.allow_threads()` + `rayon` | 진짜 멀티코어 |
| `setup.py` / `pip wheel` | `maturin` + `pyproject.toml` | wheel 빌드/배포 |

> **Under the Hood — `Python<'py>` 토큰은 어떻게 GIL을 컴파일 타임에 검증하는가**
>
> `Python<'py>`는 크기가 0인(zero-sized) 타입입니다. 런타임에 아무 데이터도 차지하지 않습니다(어셈블리로는 존재조차 하지 않습니다). 이 토큰의 유일한 역할은 **타입과 수명을 통해 "이 시점에 GIL이 잡혀 있음"을 정적으로 증명**하는 것입니다. 파이썬 객체를 다루는 모든 PyO3 API는 `py: Python<'py>` 또는 그로부터 파생된 수명 `'py`를 요구합니다. GIL을 가지지 않은 코드 경로에서는 이 토큰을 만들 수 없으므로, "GIL 없이 파이썬 객체 접근"이라는 버그가 **타입 검사 단계에서 차단**됩니다. C-API에서는 이게 순전히 개발자의 머릿속 규약이었습니다(어기면 크래시). PyO3는 그 규약을 컴파일러가 강제하는 타입으로 승격시킨 것입니다. 이것이 Chapter 6에서 본 수명이 단순한 "메모리 안전 장치"를 넘어 "프로그램 상태의 증명서"로 쓰이는 대표적 사례입니다.

---

## 22.3 maturin으로 프로젝트 셋업하기

PyO3 프로젝트는 Cargo 단독이 아니라 **maturin**이라는 빌드 프론트엔드를 씁니다. maturin은 Rust 코드를 컴파일해 `.so`(Linux/macOS) 또는 `.pyd`(Windows) 네이티브 모듈을 만들고, 그것을 파이썬이 import 할 수 있는 wheel로 포장하며, 개발 중에는 현재 가상환경에 바로 설치까지 해주는 도구입니다. `setuptools` + C 컴파일러 + ABI 설정의 삼중고를 명령어 하나로 압축합니다.

```bash
# 가상환경 안에서 maturin 설치
$ pip install maturin

# Rust-Python 하이브리드 프로젝트 생성 (PyO3 바인딩 선택)
$ maturin new --bindings pyo3 fast_agg
$ cd fast_agg
```

생성된 `Cargo.toml`은 다음과 같습니다. 핵심은 `crate-type`과 `features`입니다.

```toml
[package]
name = "fast_agg"
version = "0.1.0"
edition = "2021"

[lib]
# 파이썬이 dlopen 할 수 있는 동적 라이브러리(.so/.pyd)로 빌드.
# cdylib이 없으면 일반 Rust 라이브러리(.rlib)가 나와 import 할 수 없습니다.
name = "fast_agg"
crate-type = ["cdylib"]

[dependencies]
# extension-module 피처: 파이썬 인터프리터 심볼을 빌드 시점에 링크하지 않고
# 런타임에 호스트 인터프리터에서 해결하도록 합니다(manylinux 휠 배포의 핵심).
pyo3 = { version = "0.22", features = ["extension-module"] }
rayon = "1.10"                 # 22.5의 데이터 병렬 처리
numpy = "0.22"                 # 22.7의 제로카피 numpy 연동
```

`pyproject.toml`은 빌드 백엔드를 maturin으로 지정합니다.

```toml
[build-system]
requires = ["maturin>=1.5,<2.0"]
build-backend = "maturin"

[project]
name = "fast_agg"
requires-python = ">=3.8"
classifiers = ["Programming Language :: Rust"]

[tool.maturin]
# abi3 features를 켜면 파이썬 마이너 버전마다 따로 빌드하지 않아도 됩니다(후술).
features = ["pyo3/extension-module"]
```

개발 워크플로의 두 명령어를 구분하는 것이 중요합니다.

```bash
# 개발용: 컴파일 후 현재 venv에 즉시 설치. import 해서 바로 테스트 가능.
# --release를 빼면 디버그 빌드라 느립니다. 성능 측정 시엔 반드시 --release!
$ maturin develop --release

# 배포용: 배포 가능한 wheel(.whl) 파일을 target/wheels/ 에 생성.
$ maturin build --release
```

> **manylinux와 abi3 — wheel 배포의 두 가지 호환성 축**
>
> 파이썬 네이티브 모듈을 PyPI에 올려 `pip install`로 누구나 쓰게 하려면 두 가지 호환성 벽을 넘어야 합니다. 첫째는 **OS/libc 호환**입니다. 특정 리눅스에서 빌드한 `.so`는 다른 배포판의 glibc 버전과 충돌할 수 있습니다. 해결책이 `manylinux` 표준(오래된 공통 glibc 기반의 도커 이미지에서 빌드)이며, maturin은 `--manylinux` 옵션이나 `maturin-action` GitHub Action으로 이를 자동화합니다. 둘째는 **파이썬 버전 호환**입니다. 기본적으로는 3.9, 3.10, 3.11마다 따로 빌드해야 하지만, `abi3` 피처(`pyo3 = { features = ["abi3-py38"] }`)를 켜면 안정 ABI(Stable ABI)만 사용해 **하나의 wheel이 3.8 이상 모든 버전에서 동작**합니다. CI 매트릭스가 3×N 칸에서 1×OS 칸으로 줄어드는, 실무에서 체감이 큰 최적화입니다.

---

## 22.4 함수, 클래스, 타입 변환 — 파이썬 API를 입히기

이제 실제로 파이썬에서 부를 수 있는 것들을 만들어 봅시다. 세 가지 층위 — 함수, 자동 타입 변환, 클래스 — 를 차례로 봅니다.

### 22.4.1 `#[pyfunction]`과 자동 타입 변환

가장 단순한 형태는 모듈 레벨 함수입니다. 22.1의 집계 병목을 Rust로 옮겨봅시다.

```rust
use pyo3::prelude::*;
use std::collections::HashMap;

/// 파이썬의 list[str] 을 받아 dict[int, float] 를 반환합니다.
/// 인자 lines: Vec<String> 은 파이썬 list[str] 에서 자동 변환됩니다.
/// 반환 HashMap<i64, f64> 은 파이썬 dict 로 자동 변환됩니다.
#[pyfunction]
fn aggregate(lines: Vec<String>) -> PyResult<HashMap<i64, f64>> {
    let mut totals: HashMap<i64, f64> = HashMap::new();
    for line in &lines {
        // split_once 는 한 번만 쪼개고 (&str, &str) 를 빌려줍니다(할당 없음).
        let (id_str, value_str) = line
            .split_once(',')
            .ok_or_else(|| PyValueError::new_err(format!("잘못된 형식: {line}")))?;
        let sensor_id: i64 = id_str.trim().parse()
            .map_err(|_| PyValueError::new_err(format!("정수 파싱 실패: {id_str}")))?;
        let value: f64 = value_str.trim().parse()
            .map_err(|_| PyValueError::new_err(format!("실수 파싱 실패: {value_str}")))?;
        *totals.entry(sensor_id).or_insert(0.0) += value;
    }
    Ok(totals)
}

use pyo3::exceptions::PyValueError;

/// 모듈 진입점. 함수 이름이 import 할 모듈 이름과 일치해야 합니다.
#[pymodule]
fn fast_agg(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(aggregate, m)?)?;
    Ok(())
}
```

파이썬 쪽에서는 그냥 import 해서 씁니다.

```python
import fast_agg

lines = ["1,3.5", "2,1.0", "1,0.5"]
print(fast_agg.aggregate(lines))   # {1: 4.0, 2: 1.0}
```

여기서 주목할 것이 **자동 타입 변환**입니다. 함수 시그니처에 `Vec<String>`이라고 쓰면, PyO3가 파이썬 `list[str]`를 순회하며 각 요소를 Rust `String`으로 변환해 `Vec`를 채웁니다. 반환 `HashMap<i64, f64>`는 파이썬 `dict`로 변환됩니다. 이 변환은 두 개의 트레이트가 담당합니다.

- **`FromPyObject`** (파이썬 → Rust): 인자 변환. `i64`, `f64`, `String`, `bool`, `Vec<T>`, `HashMap<K, V>`, `Option<T>` 등 표준 타입에 대해 이미 구현되어 있습니다.
- **`IntoPy<PyObject>` / `IntoPyObject`** (Rust → 파이썬): 반환값 변환.

> **Under the Hood — 자동 변환은 "복사"라는 비용을 숨긴다**
>
> `Vec<String>`으로 받는 편안함에는 대가가 있습니다. PyO3는 파이썬 리스트의 각 `str` 객체에서 UTF-8 바이트를 **새 Rust `String`(힙 할당)으로 복사**합니다. 1,000만 줄을 `Vec<String>`으로 받으면, import 경계에서 1,000만 번의 힙 할당과 메모리 복사가 일어납니다. 함수 본체가 아무리 빨라도 이 변환 비용이 먼저 청구됩니다. 그래서 PyO3 성능 튜닝의 첫 번째 원칙은 **"경계에서 무엇을 복사하는지 의식하라"**입니다. 작은 데이터라면 무시할 만하지만, 대용량이라면 22.7에서 볼 numpy 버퍼 프로토콜로 **복사 없이 빌려오는(borrow)** 길을 택해야 합니다. 자동 변환은 마법이 아니라, 명시적으로 보이지 않을 뿐 분명히 존재하는 `memcpy`입니다.

### 22.4.2 `#[pyclass]` — Rust 구조체가 파이썬 클래스가 되다

함수만으로는 부족합니다. 상태를 가진 객체, 즉 파이썬 클래스가 필요할 때 `#[pyclass]`를 씁니다. Chapter 7에서 `dataclass`를 Rust `struct`로 옮겼고 Chapter 9에서 매직 메서드를 Trait으로 옮겼던 그 작업을, 이번엔 파이썬에서 그대로 쓸 수 있게 노출합니다.

```rust
use pyo3::prelude::*;

/// 파이썬에서 `from fast_agg import Accumulator` 로 쓸 클래스.
#[pyclass]
struct Accumulator {
    // 내부 상태. 기본적으로 파이썬에서 직접 접근 불가(캡슐화).
    total: f64,
    count: u64,
}

#[pymethods]
impl Accumulator {
    /// 파이썬의 __init__ 에 대응. #[new] 가 생성자입니다.
    #[new]
    fn new() -> Self {
        Accumulator { total: 0.0, count: 0 }
    }

    /// 일반 메서드. &mut self 로 내부 상태를 바꿉니다.
    fn add(&mut self, value: f64) {
        self.total += value;
        self.count += 1;
    }

    /// @property 처럼 동작하는 읽기 전용 getter (Chapter 7의 캡슐화).
    #[getter]
    fn mean(&self) -> PyResult<f64> {
        if self.count == 0 {
            return Err(PyValueError::new_err("빈 누적기에는 평균이 없습니다"));
        }
        Ok(self.total / self.count as f64)
    }

    /// __repr__ 매직 메서드 (Chapter 9의 Display/Debug와 같은 자리).
    fn __repr__(&self) -> String {
        format!("Accumulator(total={}, count={})", self.total, self.count)
    }
}

use pyo3::exceptions::PyValueError;
```

getter와 setter를 한 쌍으로 두면 Chapter 7에서 본 `@property`의 검증 로직을 그대로 옮길 수 있습니다.

```rust
#[pymethods]
impl Accumulator {
    // ... 위 메서드들 생략 ...

    /// 직접 total 을 읽는 getter
    #[getter]
    fn total(&self) -> f64 {
        self.total
    }

    /// total 을 쓰되 음수는 거부하는 setter (@total.setter 와 동일)
    #[setter]
    fn set_total(&mut self, value: f64) -> PyResult<()> {
        if value < 0.0 {
            return Err(PyValueError::new_err("total 은 음수가 될 수 없습니다"));
        }
        self.total = value;
        Ok(())
    }
}
```

이 클래스를 모듈에 등록(`m.add_class::<Accumulator>()?;`)하면 파이썬에서 완전히 네이티브 클래스처럼 동작합니다.

```python
from fast_agg import Accumulator

acc = Accumulator()
acc.add(3.0)
acc.add(5.0)
print(acc.mean)   # 4.0  (괄호 없음 — getter 이므로 @property 처럼)
print(repr(acc))  # Accumulator(total=8, count=2)
acc2 = Accumulator()
acc2.mean         # raise ValueError: 빈 누적기에는 평균이 없습니다
```

`#[pyclass]`로 만든 구조체의 인스턴스는 파이썬 힙에 살면서, 내부에 Rust 구조체를 품고 있습니다. `mean`은 `@property`처럼 괄호 없이 접근되고, 빈 누적기에서 호출하면 우리가 던진 `PyValueError`가 파이썬 예외로 그대로 올라옵니다 — `try/except ValueError`로 잡을 수 있는 진짜 파이썬 예외입니다. Rust의 `Result`가 파이썬의 예외 시스템과 완벽히 연결된 순간입니다.

### 22.4.3 `PyAny`/`PyDict`/`PyList`를 직접 조작하기

자동 변환은 입력 타입이 고정일 때 편리합니다. 하지만 때로는 임의의 파이썬 객체를 받아 동적으로 다뤄야 합니다 — 키가 미리 정해지지 않은 dict를 만들거나, 들어온 객체의 타입을 런타임에 분기하거나, 파이썬 함수를 콜백으로 받아 호출해야 할 때입니다. 이럴 때는 변환 없이 파이썬 객체를 **그대로** 다루는 저수준 타입을 씁니다.

- **`Bound<'py, PyAny>`**: 임의의 파이썬 객체에 대한 GIL-바운드 핸들(파이썬의 `object` 그 자체).
- **`Bound<'py, PyDict>` / `Bound<'py, PyList>`**: `dict`, `list`를 직접 만들고 채우는 타입.

```rust
use pyo3::prelude::*;
use pyo3::types::{PyDict, PyList};

/// 동적으로 dict 를 구성해 반환합니다 (키가 런타임에 결정되는 경우).
#[pyfunction]
fn build_report<'py>(py: Python<'py>, names: Vec<String>) -> PyResult<Bound<'py, PyDict>> {
    let report = PyDict::new(py);
    for (i, name) in names.iter().enumerate() {
        // dict[name] = index 를 직접 설정. set_item 이 PyResult 를 반환합니다.
        report.set_item(name, i)?;
    }
    Ok(report)
}

/// 임의의 파이썬 객체를 받아 타입에 따라 분기 (isinstance 대응).
#[pyfunction]
fn describe(obj: &Bound<'_, PyAny>) -> PyResult<String> {
    if let Ok(n) = obj.extract::<i64>() {            // 정수로 추출 시도
        Ok(format!("정수 {n}"))
    } else if let Ok(s) = obj.extract::<String>() {  // 문자열로 추출 시도
        Ok(format!("문자열 길이 {}", s.len()))
    } else if obj.is_instance_of::<PyList>() {       // list 인지 검사
        let list = obj.downcast::<PyList>()?;
        Ok(format!("리스트 원소 {}개", list.len()))
    } else {
        Ok(format!("알 수 없는 타입: {}", obj.get_type().name()?))
    }
}

/// 파이썬 콜백 함수를 받아 호출 (고차 함수).
#[pyfunction]
fn apply_twice<'py>(
    py: Python<'py>,
    func: &Bound<'py, PyAny>,
    value: i64,
) -> PyResult<Bound<'py, PyAny>> {
    let once = func.call1((value,))?;        // func(value)
    func.call1((once,))                      // func(func(value)) — 파이썬 호출 규약 자동 처리
}
```

```python
import fast_agg
print(fast_agg.build_report(["a", "b"]))   # {'a': 0, 'b': 1}
print(fast_agg.describe(42))               # 정수 42
print(fast_agg.describe([1, 2, 3]))        # 리스트 원소 3개
print(fast_agg.apply_twice(lambda x: x + 10, 5))  # 25
```

`obj.extract::<T>()`는 `FromPyObject`를 호출해 "이 객체를 T로 변환할 수 있나?"를 시도하고, 실패하면 `Err`를 줍니다. 파이썬의 덕 타이핑을 Rust에서 흉내 내는 표준 패턴입니다. `call1((args,))`은 파이썬 함수 호출 규약(위치 인자 튜플)을 Rust 튜플로 표현한 것이며, 이때 인자 변환과 GIL 보유 검증이 모두 타입 수준에서 처리됩니다.

> **Under the Hood — `Bound<'py, T>`와 `Py<T>`의 차이, 그리고 C-API 래핑**
>
> PyO3에는 파이썬 객체를 가리키는 두 종류의 핸들이 있습니다. `Bound<'py, T>`는 **GIL을 쥐고 있는 동안에만** 유효한 핸들로, 수명 `'py`가 GIL 보유를 증명합니다 — 그래서 `Bound`를 통해서는 추가 검사 없이 객체를 만질 수 있습니다. 반면 `Py<T>`(예: `PyObject = Py<PyAny>`)는 GIL과 무관하게 오래 보관할 수 있는 소유 핸들로, 실제 객체를 만지려면 `bound(py)`로 다시 GIL 토큰을 결합해 `Bound`로 승격해야 합니다. 내부적으로 이 두 타입은 모두 C의 `*mut PyObject` 원시 포인터를 감싼 것이고, `Drop` 시점에 `Py_DECREF`를 호출합니다. Chapter 21에서 본 "`unsafe`로 원시 호출을 한 뒤 안전한 타입으로 포장"하는 패턴이 바로 이것입니다 — `set_item`, `call1`, `extract` 같은 안전한 메서드의 본체에는 `ffi::PyDict_SetItem`, `ffi::PyObject_Call` 같은 `unsafe` C-API 호출이 들어 있고, PyO3가 그 반환값(`NULL`이면 예외 발생)을 검사해 `PyResult`로 변환해줍니다. 우리는 그 안전한 표면만 만지면 됩니다. C 확장이라면 손으로 했을 `NULL` 체크와 참조 카운팅이 전부 타입과 RAII 속으로 사라진 것입니다.

---

## 22.5 GIL을 풀고 진짜 멀티코어로 — `allow_threads` + rayon

여기서부터가 PyO3의 진짜 마법입니다. Chapter 1과 Chapter 16에서 누누이 강조했던 GIL의 족쇄를 — 특정 구간에서 — 완전히 벗겨내는 기술입니다.

핵심 통찰은 이렇습니다. **GIL은 파이썬 객체를 보호하기 위한 락입니다.** 그런데 우리가 데이터를 일단 Rust 타입(`Vec<f64>` 등)으로 변환해버리면, 그 데이터를 처리하는 동안에는 파이썬 객체를 전혀 건드리지 않습니다. 그렇다면 그동안 GIL을 쥐고 있을 이유가 없습니다. `py.allow_threads()`로 GIL을 놓아주면, (1) 다른 파이썬 스레드들이 그동안 자유롭게 돌 수 있고, (2) 더 중요하게는 **우리 Rust 코드가 여러 OS 스레드를 띄워 진짜 멀티코어 병렬 처리**를 할 수 있습니다.

22.4의 집계를 병렬 버전으로 발전시켜 봅시다. 데이터 병렬 처리의 사실상 표준 크레이트 `rayon`(Chapter 16에서 소개)을 결합합니다.

```rust
use pyo3::prelude::*;
use rayon::prelude::*;
use std::collections::HashMap;

/// 이미 파싱된 (id, value) 쌍을 받아 GIL을 풀고 병렬 집계합니다.
#[pyfunction]
fn aggregate_parallel(py: Python<'_>, data: Vec<(i64, f64)>) -> PyResult<HashMap<i64, f64>> {
    // data 는 순수 Rust 데이터입니다. 여기서부터 파이썬 객체를 만지지 않습니다.
    // 따라서 GIL을 놓아줍니다 → 다른 파이썬 스레드도 돌고, 우리는 멀티코어를 씁니다.
    let result = py.allow_threads(|| {
        data.par_iter()                       // rayon 병렬 이터레이터
            .fold(HashMap::new, |mut acc, &(id, value)| {
                *acc.entry(id).or_insert(0.0) += value;
                acc
            })
            // 각 스레드의 부분 HashMap 들을 하나로 병합
            .reduce(HashMap::new, |mut a, b| {
                for (id, value) in b {
                    *a.entry(id).or_insert(0.0) += value;
                }
                a
            })
    });
    // 블록을 나오면 자동으로 GIL을 재획득. 이제 파이썬 객체(HashMap→dict)를 만들 수 있습니다.
    Ok(result)
}
```

`par_iter()`는 `rayon`이 CPU 코어 수만큼 워커 스레드를 띄워 데이터를 쪼개 처리합니다. `fold`로 각 스레드가 자기 부분 합계를 내고, `reduce`로 그것들을 병합합니다. 이 전체가 `py.allow_threads()` 안에서 GIL 없이 돌기 때문에 8코어 머신이라면 이론상 8배에 가까운 속도를 냅니다.

왜 굳이 `fold` + `reduce`라는 두 단계를 거칠까요? 가장 순진한 병렬화는 모든 스레드가 하나의 공유 `HashMap`에 `Arc<Mutex<HashMap>>`(Chapter 17)로 동시에 쓰는 것입니다. 하지만 그러면 매 원소마다 락 경합(lock contention)이 일어나, 코어를 늘려도 락 앞에 줄을 서느라 오히려 느려집니다 — 이것이 병렬 프로그래밍의 고전적 함정입니다. `fold`는 **각 스레드가 자기만의 부분 결과를 락 없이 쌓고**, 마지막에 `reduce`로 한 번만 병합합니다. 공유 가변 상태를 마지막 순간까지 미루는 이 "맵-리듀스" 패턴이 진짜 선형 확장을 만드는 비결입니다. Rust의 소유권 시스템은 이 패턴을 자연스럽게 유도합니다 — 공유하려면 `Send`/`Sync`를 증명해야 하니, 개발자가 무의식 중에 공유를 최소화하게 됩니다.

> 🚨 **흔한 함정 — GIL을 안 풀고 멀티코어를 기대하기.** 만약 `py.allow_threads()` 없이 `rayon`을 그대로 돌리면 어떻게 될까요? Rust 스레드들은 뜨지만, 파이썬 함수가 GIL을 쥔 채로 있으니 — 사실 이 경우엔 파이썬 객체를 안 만지므로 동작은 합니다만 — `allow_threads`로 GIL을 명시적으로 놓아주지 않으면 **같은 인터프리터를 쓰는 다른 파이썬 스레드들이 이 함수가 끝날 때까지 전부 멈춰 섭니다.** 멀티스레드 파이썬 서버에서 이 함수 하나가 다른 모든 요청을 인질로 잡는 것입니다. "Rust로 바꿨는데 왜 동시성이 더 나빠졌지?"의 흔한 원인입니다. 순수 연산 구간은 반드시 `allow_threads`로 감싸세요.

> **Under the Hood — GIL을 풀면 어셈블리 수준에서 무슨 일이 벌어지는가**
>
> `py.allow_threads(closure)`를 호출하면 PyO3는 내부적으로 C-API의 `PyEval_SaveThread()`를 부릅니다. 이 함수는 현재 스레드의 파이썬 스레드 상태를 저장하고 **전역 GIL 뮤텍스를 해제(release)**합니다. 그 순간 OS 스케줄러는 GIL을 기다리며 잠들어 있던 다른 파이썬 스레드를 깨워 다른 코어에 배치할 수 있습니다. 동시에 클로저 안의 `rayon` 워커 스레드들도 — 이들은 파이썬과 무관한 순수 OS 스레드입니다 — 각자 다른 코어에서 동시에 전진합니다. 즉 이 구간에서는 물리적으로 N개의 코어가 동시에 명령어를 인출(fetch)·실행합니다. 클로저가 반환되면 PyO3는 `PyEval_RestoreThread()`로 GIL을 다시 획득하고 스레드 상태를 복원합니다. 이 획득은 블로킹입니다 — 다른 스레드가 GIL을 쥐고 있으면 그것을 놓을 때까지 기다립니다. 그래서 `allow_threads`로 감싼 구간이 **충분히 길어야** 이득이 납니다. 1마이크로초짜리 작업을 감싸면 GIL을 놓고 다시 잡는 오버헤드가 이득을 잡아먹습니다. "병렬화할 가치가 있을 만큼 무거운 덩어리"에만 적용하는 것이 원칙입니다.

`py.allow_threads`에 넘기는 클로저는 파이썬 객체(`Python<'py>` 토큰이나 `PyObject`)를 캡처할 수 없습니다 — 캡처하면 컴파일 에러가 납니다. 이것이 22.2의 토큰 모델이 빛나는 지점입니다. "GIL을 놓은 구간에서 파이썬 객체를 만지는" 버그가 애초에 타입 시스템에서 막힙니다. 클로저가 `Send`여야 한다는 제약(Chapter 17의 `Send`/`Sync`) 덕분에, GIL 없는 구간으로 넘길 수 있는 것은 진짜 스레드 안전한 데이터뿐입니다.

---

## 22.6 에러 처리와 의도된 실패 시연

Rust의 `Result`(Chapter 12~13)와 파이썬의 예외가 어떻게 만나는지 정밀하게 봅시다. PyO3에서 `PyResult<T>`는 `Result<T, PyErr>`의 별칭이고, 함수가 `Err(PyErr)`를 반환하면 그 `PyErr`가 파이썬 인터프리터에 예외로 설정됩니다(`PyErr_SetObject`).

표준 예외 타입들이 모두 매핑되어 있습니다. `PyValueError`, `PyTypeError`, `PyKeyError`, `PyRuntimeError`, `PyIOError` 등. 그리고 가장 강력한 것은, Chapter 13에서 `thiserror`로 만든 커스텀 에러 enum을 `#[derive]` 한 번으로 파이썬 예외로 자동 변환할 수 있다는 점입니다.

```rust
use pyo3::prelude::*;
use pyo3::exceptions::PyValueError;

// 우리 도메인 에러를 PyErr로 변환하는 규칙을 한 곳에 정의.
// 이렇게 하면 함수 본문에서 ? 연산자만으로 파이썬 예외 전파가 됩니다.
enum AggError {
    BadFormat(String),
    ParseFailed(String),
}

impl From<AggError> for PyErr {
    fn from(e: AggError) -> PyErr {
        match e {
            AggError::BadFormat(s) => PyValueError::new_err(format!("형식 오류: {s}")),
            AggError::ParseFailed(s) => PyValueError::new_err(format!("파싱 실패: {s}")),
        }
    }
}

#[pyfunction]
fn parse_one(line: &str) -> Result<(i64, f64), AggError> {
    let (id, val) = line.split_once(',')
        .ok_or_else(|| AggError::BadFormat(line.to_string()))?;
    let id: i64 = id.trim().parse()
        .map_err(|_| AggError::ParseFailed(id.to_string()))?;
    let val: f64 = val.trim().parse()
        .map_err(|_| AggError::ParseFailed(val.to_string()))?;
    Ok((id, val))   // ? 가 AggError를 PyErr로 자동 변환(From 구현 덕분)
}
```

파이썬 쪽에서는 평범한 예외로 잡힙니다.

```python
import fast_agg

try:
    fast_agg.parse_one("oops")        # 콤마가 없음
except ValueError as e:
    print("잡았다:", e)               # 잡았다: 형식 오류: oops
```

### 의도된 컴파일 에러 시연

PyO3 초심자가 가장 자주 만나는 컴파일 에러를 직접 보겠습니다. 파이썬 객체를 반환하려는데 GIL 토큰을 안 받은 경우입니다.

```rust
// 🚨 컴파일 에러! GIL 토큰 없이 파이썬 객체를 만들려 함
#[pyfunction]
fn make_list() -> PyObject {
    let py: Python = todo!();           // GIL 토큰을 어디서도 얻지 못함
    let list = pyo3::types::PyList::new(py, [1, 2, 3]);
    list.into()
}
```

GIL 토큰을 인자로 받지 않은 채 `Python`을 허공에서 만들려 하면, `Python::with_gil` 같은 명시적 획득 없이는 토큰을 얻을 길이 없어 코드가 성립하지 않습니다. 올바른 형태는 토큰을 인자로 받는 것입니다.

```rust
// ✅ 올바른 형태: GIL 토큰을 인자로 받음
#[pyfunction]
fn make_list(py: Python<'_>) -> Bound<'_, pyo3::types::PyList> {
    pyo3::types::PyList::new(py, [1, 2, 3]).unwrap()
}
```

또 다른 전형적 에러는 `allow_threads` 클로저 안으로 파이썬 객체를 끌고 들어가려는 경우입니다.

```rust
#[pyfunction]
fn bad(py: Python<'_>, obj: PyObject) -> PyResult<()> {
    py.allow_threads(|| {
        let _ = &obj;   // 🚨 GIL을 놓은 구간에서 파이썬 객체 캡처 시도
        Ok(())
    })
}
```

이때 `rustc`가 내뱉는 에러는 대략 다음과 같습니다.

```text
error[E0277]: `Py<PyAny>` cannot be shared between threads safely
   ...
   = help: within this `allow_threads` closure, the trait `Send` is not
           satisfied for `*mut pyo3::ffi::PyObject`
note: required because it appears within the type of the captured variable `obj`
   = note: GIL이 해제된 구간으로는 파이썬 객체를 보낼 수 없습니다
```

`E0277`(트레이트 바운드 미충족)은 "이 객체는 스레드 간에 안전하게 공유할 수 없다(`Send` 불만족)"고 알려줍니다. Chapter 17의 `Send` 트레이트가 여기서도 안전망으로 작동하는 것입니다. 해결책은 `allow_threads`로 들어가기 **전에** 필요한 데이터를 Rust 타입으로 추출(복사)해두고, 클로저는 그 순수 Rust 데이터만 캡처하게 하는 것입니다. 컴파일러가 "GIL 없는 곳에서 파이썬 객체를 만지는 사고"를 빌드 단계에서 봉쇄해준 셈입니다.

---

## 22.7 Hands-on — 느린 파이썬 병목을 Rust로 교체하고 벤치마크하기

이제 모든 조각을 모아, 현실적인 병목 하나를 끝까지 최적화하는 미니 프로젝트를 진행합니다. 시나리오는 22.1의 센서 로그 집계입니다. **1,000만 줄**의 `"id,value"` 문자열을 파싱해 센서별 합계를 냅니다.

### 단계 1 — 순수 파이썬 (기준선)

```python
# bench_py.py
def aggregate_py(lines: list[str]) -> dict[int, float]:
    totals: dict[int, float] = {}
    for line in lines:
        id_str, value_str = line.split(",")
        sid = int(id_str)
        totals[sid] = totals.get(sid, 0.0) + float(value_str)
    return totals
```

### 단계 2 — 단일 스레드 Rust (22.4의 `aggregate`)

이미 22.4에서 만든 `aggregate(lines: Vec<String>)`를 그대로 씁니다. 파싱과 집계가 Rust에서 일어나니, 파이썬의 객체 박싱·참조 카운팅 오버헤드가 사라집니다. 단, `Vec<String>` 변환 비용(22.4.1의 Under the Hood에서 본 복사)은 그대로 청구됩니다.

### 단계 3 — GIL 해제 + rayon 병렬 (22.5의 `aggregate_parallel`)

파싱은 파이썬에서 미리 하거나(여기서는 단순화를 위해 Rust에서 파싱 후 병렬 집계), 더 현실적으로는 파싱까지 `par_iter`로 병렬화합니다. 핵심은 무거운 집계 구간이 `allow_threads` 아래 멀티코어로 도는 것입니다.

### 벤치마크 실행과 결과

```python
# bench.py
import time, random
import fast_agg

# 1천만 줄 생성 (센서 1000개)
random.seed(0)
lines = [f"{random.randint(0, 999)},{random.random():.4f}" for _ in range(10_000_000)]

def timeit(fn, *args):
    t = time.perf_counter()
    fn(*args)
    return time.perf_counter() - t

print("pure python:", timeit(aggregate_py, lines))
print("rust single:", timeit(fast_agg.aggregate, lines))
# 병렬 버전은 (id, value) 튜플 입력을 받는 형태라 별도 준비
```

대표적인 측정 결과입니다(아래 수치는 측정 환경에 따라 달라집니다 — 절대값이 아니라 배수와 경향을 보세요).

```text
측정 환경: 입력 1천만 줄, Apple M-시리즈 8코어, --release 빌드, PyO3 0.22
─────────────────────────────────────────────────────────────
방식                        시간        파이썬 대비   비고
─────────────────────────────────────────────────────────────
순수 파이썬                 ~6.5 s       1.0x        기준선
Rust 단일 스레드            ~0.85 s      ~7-8x       객체 박싱 제거
Rust + GIL해제 + rayon      ~0.18 s      ~35-40x     8코어 활용
─────────────────────────────────────────────────────────────
```

> 🚨 **벤치마크를 읽는 법 — 과장에 속지 마세요.** "Rust로 100배 빨라졌다" 같은 헤드라인은 대개 두 가지 트릭을 씁니다. 첫째, 파이썬을 일부러 비효율적으로 짠 기준선과 비교합니다(예: `dict.get` 대신 `try/except`). 둘째, 변환 비용(경계의 `memcpy`)을 측정에서 제외합니다. 정직한 비교는 (1) 파이썬도 최선을 다해 짜고, (2) 데이터 변환 비용을 포함하며, (3) 입력 크기와 빌드 모드(`--release`)를 명시합니다. 위 표에서 단일 스레드가 "겨우" 7-8배인 이유는 `Vec<String>` 변환 비용이 그만큼 크기 때문입니다. 이 병목을 넘으려면 다음의 제로카피가 필요합니다.

### 단계 4 — 제로카피로 변환 비용마저 없애기 (numpy 버퍼 프로토콜)

수치 데이터라면 애초에 문자열을 거치지 말고 numpy 배열을 통째로 빌려오는 것이 정답입니다. `numpy` 크레이트를 쓰면 numpy 배열의 메모리를 **복사 없이** Rust 슬라이스로 봅니다.

```rust
use pyo3::prelude::*;
use numpy::{PyReadonlyArray1, PyArray1, IntoPyArray};

/// numpy 배열을 복사 없이 빌려와 각 원소를 제곱합니다.
#[pyfunction]
fn square_all<'py>(
    py: Python<'py>,
    arr: PyReadonlyArray1<'py, f64>,   // numpy 배열을 "빌림" (제로카피)
) -> Bound<'py, PyArray1<f64>> {
    let slice = arr.as_slice().unwrap();        // &[f64] — numpy 버퍼를 직접 가리킴
    let result: Vec<f64> = py.allow_threads(|| {
        use rayon::prelude::*;
        slice.par_iter().map(|x| x * x).collect()
    });
    result.into_pyarray(py).to_owned()          // 결과만 새 numpy 배열로
}
```

```python
import numpy as np, fast_agg
a = np.arange(10_000_000, dtype=np.float64)
b = fast_agg.square_all(a)   # 입력은 복사 안 함, 계산은 멀티코어
```

> **Under the Hood — 버퍼 프로토콜과 제로카피의 진짜 의미**
>
> numpy 배열은 내부에 연속된 C 배열(예: `f64` 8바이트 × N개)을 들고 있고, 이 메모리의 시작 포인터·길이·스트라이드(stride)를 **버퍼 프로토콜(PEP 3118)**로 노출합니다. `PyReadonlyArray1`은 이 포인터를 받아 Rust의 `&[f64]` 슬라이스로 **재해석**할 뿐, 단 한 바이트도 복사하지 않습니다. 1,000만 개 `f64`(80MB)를 `Vec<f64>`로 받으면 80MB를 복사해야 하지만, 제로카피는 포인터 하나(8바이트)만 넘깁니다. 이것이 `polars`가 화살표(Apache Arrow) 메모리 위에서 파이썬과 데이터를 주고받으면서도 복사 비용 없이 수십 배 빠른 비결입니다. 다만 빌리는 동안 numpy가 그 배열을 재할당하면 안 되므로 PyO3는 빌림 기간 동안 읽기 잠금을 겁니다 — Chapter 5의 대여(Borrowing) 규칙이 언어 경계를 넘어 numpy에까지 적용되는 것입니다. 제로카피는 공짜가 아니라, "복사 비용을 대여 규칙 준수로 치환"하는 거래입니다.

제로카피를 쓰면 단계 2-3에서 발목을 잡던 변환 비용이 사라져, 대용량 수치 워크로드에서 파이썬 대비 수십 배의 속도가 변환 오버헤드 없이 온전히 실현됩니다. 이것이 실무에서 PyO3를 도입하는 가장 강력한 명분입니다.

---

## 22.8 Best Practices & 흔한 함정

PyO3를 실무에 도입할 때의 체크리스트와, 파이썬 엔지니어가 반복하는 실수들을 정리합니다.

### Best Practices

- **프로파일링이 먼저다.** `cProfile`, `py-spy`, `scalene`으로 진짜 병목(전체 시간의 80%를 먹는 5%)을 먼저 특정하세요. 측정 없이 Rust로 옮기는 것은 도박입니다.
- **경계는 좁고 굵게.** import 경계를 자주 넘나들면 변환 비용이 누적됩니다. "한 줄씩 1,000만 번 호출"이 아니라 "리스트 전체를 한 번 넘기고 결과를 한 번 받기"로 설계하세요. FFI 호출 자체에도 고정 오버헤드가 있습니다.
- **무거운 순수 연산은 `allow_threads`로 감싸라.** 파이썬 객체를 안 만지는 구간이라면 GIL을 놓아 다른 스레드를 살리고 멀티코어를 쓰세요.
- **대용량 수치는 제로카피로.** `numpy` 크레이트의 `PyReadonlyArray`로 복사를 피하세요. 작은 데이터는 자동 변환이 편하고 충분합니다.
- **에러는 `?`로 전파하고 `From<MyError> for PyErr`를 한 곳에 정의.** Chapter 13의 패턴을 그대로 가져오면 함수 본문이 깨끗해집니다.
- **`maturin develop --release`로 측정.** 디버그 빌드는 10배 이상 느립니다. 성능 이야기를 할 땐 무조건 `--release`.
- **배포는 `abi3` + `manylinux`.** CI 매트릭스를 줄이고 호환성 문제를 예방합니다.
- **`#[pyclass]`에 `#[pyo3(get, set)]`이나 `__repr__`을 달아 파이썬다운 객체로.** 사용자가 네이티브 클래스인지 눈치채지 못할 만큼 자연스러워야 좋은 바인딩입니다.

### 흔한 함정

- **함정 1 — GIL을 안 풀고 멀티코어를 기대한다.** 22.5에서 본 그대로입니다. `rayon`을 써도 `allow_threads`로 감싸지 않으면 다른 파이썬 스레드를 막습니다. "Rust로 바꿨는데 서버 동시성이 더 나빠졌다"의 단골 원인입니다.
- **함정 2 — 과도한 데이터 복사.** `Vec<String>`, `HashMap` 자동 변환의 편리함 뒤에는 경계에서의 대규모 `memcpy`가 숨어 있습니다(22.4.1). 80MB 배열을 `Vec`로 받으면 80MB를 복사합니다. 대용량이면 제로카피를 검토하세요.
- **함정 3 — 너무 잘게 호출한다.** 파이썬 루프 안에서 Rust 함수를 원소마다 호출하면, 함수 본체보다 FFI 경계 왕복 오버헤드가 더 큽니다. 배치(batch)로 넘기세요.
- **함정 4 — 디버그 빌드로 벤치마크한다.** `maturin develop`(--release 없이)로 측정하고 "2배밖에 안 빠르네"라고 실망하는 경우. 빌드 모드를 항상 확인하세요.
- **함정 5 — `unwrap()`/`panic!`을 그대로 둔다.** Rust 패닉은 PyO3가 `pyo3_runtime.PanicException`으로 잡아주긴 하지만, 이는 복구 불가능한 신호입니다. 사용자에게 의미 있는 에러를 주려면 패닉 대신 `Err(PyValueError::new_err(...))`를 반환하세요. 라이브러리 경계에서 패닉은 곧 크래시에 준하는 사고입니다.
- **함정 6 — 모든 것을 Rust로 옮기려 한다.** I/O 대기가 99%인 핸들러를 Rust로 옮기면 코드만 복잡해지고 속도는 그대로입니다. Rust는 CPU 병목 전용 수술 도구이지, 만병통치약이 아닙니다.

---

## 22.9 맺음말: 새로운 Rustacean에게 — 그리고 책 전체를 마무리하며

이로써 **"Python to Rust Complete Guide"**의 22개 챕터, 긴 여정이 끝났습니다. 잠시 우리가 지나온 길을 돌아봅시다.

Part 1에서 우리는 왜 파이썬 엔지니어가 Rust를 배우는지(GIL·GC·런타임 오버헤드의 한계), 그리고 `pip`/`poetry`/`venv`의 혼돈을 Cargo 하나로 통합하는 법을 보았습니다. Part 2에서 이 책의 심장인 **소유권(Ownership)·대여(Borrowing)·수명(Lifetime)**으로 GC 없이 사는 법을 배웠고, Part 3에서 파이썬의 풍부한 객체지향(상속·매직 메서드·Pydantic)을 **Struct·Trait·Enum**으로 해체해 재조립했습니다. Part 4에서 `try/except`를 버리고 **Option과 Result**로 실패를 타입에 담았으며, Part 5에서 컬렉션과 무비용 이터레이터를, Part 6에서 GIL을 벗어난 진짜 동시성(**Arc/Mutex**, tokio 비동기)을 익혔습니다. Part 7에서 매크로와 스마트 포인터로 메타프로그래밍의 깊은 곳을 보았고, Part 8에서 C FFI와 `unsafe`를 거쳐 마침내 이번 챕터에서 두 언어를 물리적으로 결합했습니다.

이 모든 것을 관통하는 단 하나의 깨달음이 있습니다. **Rust는 파이썬을 대체하려고 온 언어가 아닙니다.** Rust는 파이썬의 밑단(C/C++)을 가장 안전하고 모던하게 교체하여, **파이썬 생태계를 더 빠르고 강력하게 만드는 최고의 파트너**입니다. `pydantic-core`, `polars`, `tokenizers`, `ruff`가 증명하듯, 미래의 고성능 파이썬은 점점 더 "Rust 엔진에 파이썬 API를 입힌" 형태가 될 것입니다. 그리고 이제 여러분은 그 엔진을 직접 만들 수 있습니다.

그렇다면 다음에 무엇을 해야 할까요. 세 갈래의 길을 권합니다.

**1) 가장 작은 실전부터.** 다음 주에 출근하면, 여러분의 코드베이스에서 가장 느린 함수 하나를 `py-spy`로 프로파일링하세요. 그 함수가 CPU 바운드이고, 입력·출력이 단순한 자료형이라면, `maturin new`로 프로젝트를 만들고 그 함수만 Rust로 옮겨보세요. 50줄짜리 첫 PyO3 모듈이 7배 빨라지는 경험은 백 마디 설명보다 강력합니다. 절대 전부 다 옮기지 마세요 — 가장 아픈 곳 한 군데만.

**2) 깊이를 더하는 학습 경로.** 이 책은 끝이 아니라 시작점입니다. 다음을 권합니다. 공식 문서 *The Rust Programming Language*("The Book")로 기초를 다시 정리하고, *Rust for Rustaceans*(Jon Gjengset)로 트레이트·제네릭·`unsafe`의 깊은 곳을 파며, 동시성을 더 원하면 *Rust Atomics and Locks*(Mara Bos)를, 비동기를 더 원하면 tokio 공식 튜토리얼을 보세요. PyO3는 [PyO3 User Guide](https://pyo3.rs)가 가장 정확하며, 실전 예제는 `polars`·`ruff`의 GitHub 소스를 읽는 것이 어떤 책보다 낫습니다.

**3) 커뮤니티에 합류.** Rust는 컴파일러 에러가 친절하기로 유명하고, 커뮤니티는 그보다 더 친절합니다. 처음 만나는 `error[E0382]`(이동 후 사용)나 `error[E0277]`(트레이트 미충족)에 좌절하지 마세요. 이 책에서 거듭 보았듯, 그 에러 하나하나는 파이썬이었다면 프로덕션 새벽에 호출기를 울렸을 버그를 컴파일러가 미리 잡아준 것입니다. 컴파일러를 적이 아니라 가장 까다롭지만 가장 믿음직한 페어 프로그래머로 받아들이는 순간, 여러분의 Rust는 두려움(fear)이 아니라 담대함(fearless)의 영역으로 들어섭니다.

10년간 파이썬으로 쌓아온 여러분의 직관 — 메모리 구조, 동시성, 시간복잡도, 시스템 설계 — 은 사라지지 않습니다. 오히려 Rust라는 새로운 렌즈를 통해 더 선명해집니다. 파이썬으로 "무엇을" 빠르게 만들지 결정하고, Rust로 "어떻게" 한계 없이 빠르게 만들지 실현하는 두 언어를 모두 쥔 엔지니어는, 둘 중 하나만 아는 엔지니어가 닿지 못하는 곳까지 갑니다.

긴 여정을 끝까지 함께해 주셔서 감사합니다. 이제 진짜 시작입니다.

환영합니다, 새로운 Rustacean. 🦀
