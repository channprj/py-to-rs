# Chapter 2. 도구 생태계: 패키징 지옥에서 벗어나기

파이썬 생태계에서 프로젝트를 시작할 때 우리는 수많은 선택의 기로에 놓입니다. 
"의존성 관리는 `pip`로 할까, `pipenv`로 할까, 아니면 `poetry`를 쓸까?"
"가상 환경은 `venv`? `conda`?"
"포매터는 `black`, 린터는 `flake8`, 테스트는 `pytest`..."

이처럼 파이썬은 각 영역마다 훌륭한 서드파티 도구들이 존재하지만, 이들을 엮어서 하나의 일관된 개발 파이프라인(CI/CD)을 구축하는 것은 온전히 개발자의 몫입니다. 때로는 버전 충돌이나 운영체제 간의 차이로 인해 배포보다 환경 세팅에 더 많은 시간을 쏟기도 합니다.

Rust는 이 문제를 **Cargo(카고)**라는 공식 빌드 시스템 겸 패키지 매니저 단 하나로 평정했습니다. 이 챕터에서는 파이썬의 파편화된 도구들을 Cargo의 멘탈 모델로 통합해 봅니다.

---

## 2.1 가상 환경(Virtual Environment)의 소멸

파이썬 개발자라면 프로젝트 디렉토리에 진입할 때 숨 쉬듯 `source venv/bin/activate`를 타이핑합니다. 파이썬은 기본적으로 시스템 전역(Global)에 패키지를 설치하려는 습성이 있기 때문에, 프로젝트별로 의존성이 꼬이는 것을 막기 위해 가상 환경을 필수적으로 만들어야 했습니다.

* **The Python Way**:
  ```bash
  $ python -m venv .venv
  $ source .venv/bin/activate
  $ pip install requests
  ```

**Rust의 해결책: 가상 환경은 필요 없다**
Cargo는 시스템 전역에 패키지를 흩뿌리지 않습니다. 다운로드한 패키지 소스 코드는 글로벌 캐시(`~/.cargo/registry`)에 안전하게 저장되지만, 패키지의 **빌드 결과물**은 무조건 해당 프로젝트 폴더 내부의 `target/` 디렉토리에 격리되어 저장됩니다.

* **The Rust Way**:
  ```bash
  $ cargo new my_project
  $ cd my_project
  $ cargo add reqwest
  ```
단지 이 명령어만으로 의존성이 프로젝트에 격리되며, 다른 프로젝트와 절대 충돌하지 않습니다. 가상 환경을 켜고 끄는 개념 자체가 Rust에는 존재하지 않습니다.

---

## 2.2 `pyproject.toml`에서 `Cargo.toml`로

최근 파이썬 생태계도 PEP 518을 통해 `setup.py`나 `requirements.txt` 대신 `pyproject.toml`로 대통합을 이루어가는 추세입니다. 놀랍게도 이 변화는 Rust의 `Cargo.toml` 구조에 큰 영감을 받은 것입니다.

### Python (`pyproject.toml` - Poetry 예시)
```toml
[tool.poetry]
name = "my-python-app"
version = "0.1.0"
description = ""
authors = ["Pythonista <python@example.com>"]

[tool.poetry.dependencies]
python = "^3.10"
fastapi = "^0.95.0"
uvicorn = {extras = ["standard"], version = "^0.22.0"}

[tool.poetry.group.dev.dependencies]
pytest = "^7.3.1"
black = "^23.3.0"
```

### Rust (`Cargo.toml`)
Rust의 `Cargo.toml`은 훨씬 단순하고 언어 차원에서 표준화되어 있습니다. 서드파티 툴(`[tool.poetry]`)이라는 접두사가 필요 없습니다.

```toml
[package]
name = "my_rust_app"
version = "0.1.0"
edition = "2021" # 파이썬 버전 제약과 유사
authors = ["Rustacean <rust@example.com>"]

[dependencies]
# fastapi의 대체재
axum = "0.6.18"
# uvicorn의 대체재이자 비동기 런타임
tokio = { version = "1.28.1", features = ["full"] }
# JSON 직렬화/역직렬화 (pydantic 대체재)
serde = { version = "1.0", features = ["derive"] }

[dev-dependencies]
# pytest 플러그인 등 개발 환경 의존성
tokio-test = "0.4.2"
```

---

## 2.3 빌드 파이프라인: Debug vs Release

파이썬 코드는 실행할 때 컴파일과 런타임이 거의 동시에 일어납니다. 반면 Rust는 AOT(Ahead-of-Time) 컴파일 언어이므로, 소스코드를 기계어로 번역하는 빌드(Build) 과정이 필수적입니다. 이 빌드 과정은 두 가지 모드로 나뉩니다.

### 2.3.1 디버그 빌드 (개발 모드)
개발 중에는 컴파일 속도가 빨라야 하고, 에러가 났을 때 디버깅 정보가 풍부해야 합니다.
```bash
# 코드 빌드 및 실행
$ cargo run
```
* 특징: 컴파일이 빠름. 코드는 최적화되지 않아 상대적으로 느리게 동작함. 메모리 초과 연산이나 오버플로우가 발생하면 즉시 패닉(Panic)을 일으킴.

### 2.3.2 릴리즈 빌드 (운영 모드)
코드를 프로덕션(Production) 서버에 배포할 때는 극단적인 성능 최적화가 필요합니다.
```bash
$ cargo build --release
```
* 특징: LLVM 백엔드를 사용하여 코드 실행 경로를 철저하게 분석하고 최적화함. (인라인 함수 풀기, 사용하지 않는 변수 제거 등). 컴파일 시간은 길어지지만 런타임 속도는 C/C++에 필적함. 빌드된 실행 파일은 `target/release/my_rust_app`에 생성되며, **단일 바이너리(Single Binary)**이므로 서버에 배포할 때 파이썬이나 기타 의존성을 설치할 필요가 전혀 없음.

---

## 2.4 파이썬의 도구들, Cargo로 1:1 매핑하기

| 역할 | Python 생태계 | Rust 생태계 (`Cargo`) |
| :--- | :--- | :--- |
| **패키지 관리** | `pip`, `poetry add` | `cargo add` |
| **빌드/실행** | `python main.py` | `cargo run` |
| **테스트** | `pytest` | `cargo test` |
| **코드 포맷팅** | `black`, `yapf` | `cargo fmt` |
| **정적 분석(Linter)** | `flake8`, `pylint`, `mypy` | `cargo clippy` |
| **문서화** | `Sphinx`, `MkDocs` | `cargo doc --open` |

이 중 10년 차 파이썬 개발자들이 가장 감탄하는 도구는 바로 **`cargo clippy`**입니다. 
파이썬의 `flake8`이나 `pylint`가 주로 코딩 컨벤션이나 간단한 문법 오류를 잡아준다면, `clippy`는 **"네가 작성한 코드는 메모리 비효율이 발생하니, 이렇게 고치는 게 성능이 더 좋다"**라고 코딩의 본질적인 구조를 지적하고 리팩토링 방향을 가르쳐주는 수준입니다. Clippy는 러스트를 독학할 때 가장 훌륭한 스승이 됩니다.

---

## 2.5 요약

파이썬 환경에서는 새 팀원이 합류하면 "문서 보고 의존성 설치하시고, pre-commit 훅 설정하시고..."와 같은 온보딩 과정이 필요합니다. 

Rust 환경에서는 팀원에게 단 한 줄만 알려주면 됩니다.
**"Git 클론 받으신 후에 `cargo run` 하세요."**

의존성 다운로드, C 컴파일러 연동, 최적화 링킹 등 모든 과정을 Cargo가 알아서 처리해 줍니다. 이제 파이썬 생태계의 복잡한 툴 체인을 잊고, Rust의 우아하고 통합된 개발 경험으로 넘어갈 준비가 완료되었습니다. 다음 챕터에서는 본격적으로 파이썬의 구문이 Rust의 표현식(Expression)으로 어떻게 매핑되는지 살펴보겠습니다.
