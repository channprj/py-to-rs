# Chapter 22. PyO3: Python과 Rust의 완벽한 결합

지금까지 파이썬의 한계(GIL, 오버헤드, 타입 불안정성)와 Rust의 해결책(Zero-cost, 소유권, 안전한 동시성)을 깊이 있게 파헤쳤습니다. 

그렇다면 파이썬 생태계를 버리고 모두 Rust로 다시 짜야 할까요? 아닙니다. 백엔드 개발에서 비즈니스 로직(DB ORM, 사용자 인증, 외부 API 연동 등)은 생산성이 높은 파이썬으로 작성하는 것이 훨씬 유리합니다. Rust가 진가를 발휘하는 곳은 **"CPU를 많이 쓰거나, 다량의 데이터를 파싱하거나, 고도의 병렬 처리가 필요한 코어 모듈"**입니다.

이 장에서는 Rust 코드를 파이썬에서 `import` 할 수 있는 네이티브 모듈(Native Extension)로 만들어주는 마법의 도구, **PyO3**와 **Maturin**을 소개합니다. `pydantic-core`, `polars`, `orjson` 등 현대 파이썬 생태계의 고성능 라이브러리들이 모두 이 기술로 만들어졌습니다.

---

## 22.1 `maturin`을 이용한 프로젝트 셋업

PyO3 기반 프로젝트를 셋업할 때는 Cargo 대신 `maturin`이라는 빌드 툴을 사용합니다. 

```bash
# 파이썬 가상환경에서 maturin 설치
$ pip install maturin

# 새로운 Rust-Python 하이브리드 프로젝트 생성
$ maturin new my_rust_module
```

`Cargo.toml` 파일에는 자동으로 PyO3 의존성이 추가됩니다.
```toml
[package]
name = "my_rust_module"
version = "0.1.0"
edition = "2021"

[lib]
# 파이썬 모듈로 빌드하기 위해 'cdylib' (동적 C 라이브러리) 타입을 지정합니다.
crate-type = ["cdylib"] 

[dependencies]
# 파이썬 C-API를 안전하게 감싼 PyO3 크레이트
pyo3 = { version = "0.19", features = ["extension-module"] }
```

---

## 22.2 러스트 함수를 파이썬으로 노출하기

아주 무거운 텍스트 파싱 로직이나 수학 연산이 있다고 가정해 봅시다. 

* **The Rust Implementation (`src/lib.rs`)**
  ```rust
  use pyo3::prelude::*;

  // #[pyfunction] 매크로를 붙이면, 이 함수는 파이썬에서 호출 가능한 C-API로 변환됩니다.
  #[pyfunction]
  fn heavy_computation(a: usize, b: usize) -> PyResult<usize> {
      let mut sum = 0;
      for i in 0..10_000_000 {
          sum += (a + b) % (i + 1);
      }
      Ok(sum) // PyResult를 반환하여 파이썬의 Exception 시스템과 연동합니다.
  }

  // #[pymodule] 매크로는 이 파일 전체를 파이썬의 '모듈(Module)'로 묶어줍니다.
  #[pymodule]
  fn my_rust_module(_py: Python, m: &PyModule) -> PyResult<()> {
      // 파이썬 모듈 객체(m)에 위에서 만든 heavy_computation 함수를 추가합니다.
      m.add_function(wrap_pyfunction!(heavy_computation, m)?)?;
      Ok(())
  }
  ```

### 빌드 및 파이썬에서 실행
코드를 작성한 뒤, `maturin develop` 명령어를 실행하면 Rust 컴파일러가 코드를 빌드하고, 그 결과물( `.so` 또는 `.pyd` 파일)을 현재 파이썬 가상 환경에 자동으로 설치해 줍니다.

```bash
$ maturin develop --release
```

설치가 끝났다면, 파이썬 인터프리터를 열고 평소처럼 `import` 하면 됩니다.
```python
import my_rust_module
import time

start = time.time()
# 런타임은 러스트의 C-Extension 코드를 타고 돌아갑니다!
result = my_rust_module.heavy_computation(10, 20)
print(f"Result: {result}, Took: {time.time() - start:.5f}s")
```
파이썬으로 순수하게 짰을 때보다 최소 수십 배에서 수백 배 빠른 결과를 얻을 수 있습니다.

---

## 22.3 GIL 릴리즈 (진정한 멀티스레딩)

PyO3의 가장 강력한 기능은 **특정 Rust 블록 안에서 파이썬의 GIL(Global Interpreter Lock)을 강제로 풀어버릴 수 있다**는 점입니다.

만약 우리가 Rust 내부에서 스레드를 여러 개 띄워 무언가를 처리하려고 할 때, 파이썬 객체를 건드리지 않는 순수 연산이라면 GIL을 쥐고 있을 필요가 없습니다.

```rust
use pyo3::prelude::*;
use std::thread;

#[pyfunction]
fn parallel_task(py: Python) -> PyResult<usize> {
    // 파이썬 인터프리터의 GIL을 명시적으로 놓아줍니다 (Release).
    // 이 블록 안에서는 다른 파이썬 스레드들이 멈추지 않고 평소처럼 동작할 수 있습니다!
    let result = py.allow_threads(|| {
        // --- 여기는 GIL이 없는 순수한 하드웨어 세상입니다 ---
        let mut handles = vec![];
        for i in 0..4 {
            handles.push(thread::spawn(move || {
                // 무거운 계산...
                i * 100
            }));
        }
        
        let mut sum = 0;
        for h in handles {
            sum += h.join().unwrap();
        }
        sum
        // ------------------------------------------
    }); // 블록이 끝나면 다시 파이썬 GIL을 획득(Acquire)하여 파이썬 세계로 돌아갑니다.

    Ok(result)
}
```

이 기능을 활용하면, 파이썬의 고질적인 문제였던 "CPU 바운드 작업에서의 멀티스레딩 불가" 제약을 완벽하게 부술 수 있습니다.

---

## 22.4 맺음말: 두 세계의 융합

이로써 10년 차 파이썬 개발자를 위한 **"Python to Rust Complete Guide"**의 긴 여정이 끝났습니다.

Rust는 파이썬을 대체하기 위해 등장한 언어가 아닙니다. 오히려 파이썬의 밑단(C/C++)을 가장 안전하고 모던하게 교체하여, **파이썬 생태계를 더욱 빠르고 강력하게 만들어 주는 최고의 파트너**입니다.

* 데이터 모델링은 **Struct와 Trait**으로,
* 에러 핸들링은 **Option과 Result**로,
* 메모리 관리는 **Ownership과 Borrowing**으로,
* 동시성은 **Arc/Mutex와 Tokio**로 대체하는 법을 배웠습니다.

이제 파이썬의 빠른 개발 생산성과 Rust의 극단적인 성능 및 안전성을 결합할 차례입니다. 기존 파이썬 프로젝트에서 가장 느린 병목 지점(Bottleneck)을 찾아 프로파일링한 뒤, 그 부분만 PyO3를 활용해 Rust로 교체하는 작고 즐거운 실험부터 시작해 보시길 권합니다.

환영합니다, 새로운 Rustacean!
