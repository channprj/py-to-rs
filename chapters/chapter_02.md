# Chapter 2. 도구 생태계: 패키징 지옥에서 벗어나기

새 노트북을 받았다고 상상해 봅시다. 10년 차 파이썬 엔지니어인 여러분이 회사의 기존 서비스를 로컬에서 띄우기까지 몇 단계가 필요할까요? 대략 이렇습니다. `pyenv`로 정확한 패치 버전(3.11.4였던가, 3.11.6이었던가?)을 깔고, `pyenv local`로 디렉터리에 고정하고, `python -m venv`로 가상 환경을 만들고, 활성화한 뒤, `poetry install`을 돌립니다. 그런데 `psycopg2` 빌드가 깨집니다. 시스템에 `libpq`가 없거나, `openssl@3` 헤더 경로를 못 찾았거나, Apple Silicon인데 휠(wheel)이 x86_64용만 있었던 겁니다. Confluence 온보딩 문서의 "Troubleshooting" 섹션을 뒤지며 `brew install`을 몇 번 더 하고, 환경 변수를 export하고, 결국 30분에서 길게는 반나절을 비즈니스 로직과 무관한 환경 세팅에 씁니다.

이것이 파이썬의 **패키징 지옥(Packaging Hell)**입니다. 문제의 본질은 파이썬이 나빠서가 아니라, **언어의 책임 범위가 인터프리터 하나에서 끝나기 때문**입니다. 버전 관리(`pyenv`/`asdf`), 환경 격리(`venv`/`virtualenv`/`conda`), 의존성 해석(`pip`/`pipenv`/`poetry`/`pdm`/`uv`), 락 파일(`Pipfile.lock`/`poetry.lock`), 포매팅(`black`/`isort`), 린팅(`flake8`/`pylint`/`ruff`), 타입 체크(`mypy`/`pyright`), 테스트(`pytest`/`tox`/`nox`), 빌드 백엔드(`setuptools`/`hatchling`/`maturin`) — 이 모든 영역을 **서로 다른 단체와 개인이 만든 서드파티 도구**가 채우고 있습니다. 각각은 훌륭하지만, 이들을 하나의 일관된 파이프라인으로 엮는 일은 온전히 여러분의 몫입니다. 그리고 그 조합은 회사마다, 프로젝트마다, 심지어 팀원마다 미묘하게 다릅니다.

Rust는 이 문제를 정반대 철학으로 풀었습니다. **언어를 만들 때부터 도구 체인을 함께 설계**했습니다. 버전·채널 관리는 `rustup`이, 빌드·의존성·테스트·포맷·문서·벤치마크는 `cargo`가, 일관된 코드 품질은 `rustfmt`와 `clippy`가 담당합니다. 그리고 이 셋은 따로 노는 도구가 아니라 하나의 통합된 경험으로 묶여 있습니다. 신입 팀원에게 건넬 온보딩 문서는 단 한 줄로 줄어듭니다. "클론받고 `cargo run` 하세요."

이 챕터의 목표는 단순히 명령어를 외우는 것이 아닙니다. 여러분의 머릿속에 깊이 각인된 "가상 환경을 켜고 끈다", "버전이 깨질까 봐 두렵다"는 **멘탈 모델 자체를 교체**하는 것입니다. 더 나아가 `Cargo.toml`을 한 줄씩 해부하고, Debug와 Release 빌드가 기계어 수준에서 무엇이 다른지, 정적 링크된 단일 바이너리가 어떻게 파이썬의 배포 지옥을 끝장내는지를 실제 수치와 함께 들여다봅니다.

---

## 2.1 The Python Way — 파편화된 도구들의 오케스트라

파이썬 프로젝트 하나를 운영하려면 머릿속에 동시에 여러 도구의 상태를 떠올려야 합니다. 다음은 전형적인 현대 파이썬 프로젝트의 풀 스택입니다.

```bash
# 1) 파이썬 버전 자체를 관리 (pyenv 또는 asdf)
$ pyenv install 3.11.6
$ pyenv local 3.11.6          # .python-version 파일 생성

# 2) 환경 격리 (venv)
$ python -m venv .venv
$ source .venv/bin/activate   # 셸 세션마다 반복

# 3) 의존성 해석 및 설치 (poetry / uv / pip)
$ poetry install              # poetry.lock 기준 설치

# 4) 코드 품질 도구들 — 각각 별도 설정 파일과 별도 실행
$ black .                     # 포매팅
$ isort .                     # import 정렬
$ ruff check .                # 린팅
$ mypy src/                   # 타입 체크

# 5) 테스트
$ pytest -q
```

각 단계는 독립적이고, **서로의 존재를 모릅니다.** `pytest`는 `mypy`가 통과했는지 알 수 없고, `black`은 `ruff`의 설정과 충돌하지 않도록 여러분이 직접 `pyproject.toml`에서 `line-length`를 맞춰줘야 합니다. 이 조합을 강제하기 위해 `pre-commit` 훅을 깔고, CI에서는 또 별도의 워크플로 YAML을 작성합니다. 도구가 여덟 개라면, 깨질 수 있는 접점도 여덟 개입니다.

> 🚨 **런타임 비용의 함정: 휠(wheel)과 sdist**
> 파이썬 패키지는 두 형태로 배포됩니다. 미리 컴파일된 바이너리인 **휠(`.whl`)**과, 빌드되지 않은 소스 배포본인 **sdist(`.tar.gz`)**입니다. C 확장을 포함한 패키지(`numpy`, `pydantic-core`, `psycopg2`)에서 여러분의 플랫폼(OS × CPU 아키텍처 × 파이썬 ABI)에 맞는 휠이 PyPI에 없으면, `pip`는 sdist를 받아 **여러분의 머신에서 직접 컴파일**합니다. 이때 시스템에 C 컴파일러, 헤더 파일, 링커가 없으면 그 악명 높은 `error: command 'gcc' failed` 메시지가 뜹니다. 즉, 파이썬의 "설치"는 사실 **숨겨진 빌드 단계**를 품고 있으며, 그 빌드는 여러분의 시스템 상태에 의존합니다. "내 컴퓨터에선 됐는데"의 근원이 바로 이것입니다.

핵심 문제는 **재현성(Reproducibility)**입니다. `requirements.txt`에 `requests`라고만 적으면 오늘 설치한 버전과 한 달 뒤 동료가 설치한 버전이 다를 수 있습니다. `pip freeze`로 버전을 못 박아도, 그 라이브러리의 **하위 의존성(transitive dependency)**까지 고정되지는 않습니다. `poetry`/`uv`의 락 파일이 이 문제를 상당 부분 해결했지만, 이는 비교적 최근의 일이며 여전히 어떤 도구를 쓰느냐에 따라 동작이 다릅니다.

---

## 2.2 The Rust Mental Model — "언어가 곧 도구 체인이다"

Rust는 2015년 1.0 출시 시점부터 `cargo`를 함께 내놓았습니다. 이것은 단순한 편의 기능이 아니라 **설계 철학의 선언**입니다. "패키징, 빌드, 테스트, 문서화는 언어의 책임이지 사용자가 조립할 문제가 아니다."

이 철학이 만드는 세 가지 멘탈 모델의 전환을 먼저 짚겠습니다.

**첫째, 전역(Global)이 아니라 프로젝트(Project)가 기본 단위입니다.** 파이썬의 `pip install`은 본능적으로 시스템 전역의 `site-packages`를 노립니다. 그래서 격리를 위해 가상 환경이 필요했습니다. Cargo는 정반대입니다. 의존성의 **빌드 산출물**은 언제나 프로젝트 폴더 안의 `target/`에 격리됩니다. 다운로드한 소스만 전역 캐시(`~/.cargo/registry`)를 공유할 뿐, 프로젝트끼리 빌드 결과가 충돌하는 일은 구조적으로 불가능합니다. "가상 환경을 켠다"는 동작 자체가 사라집니다.

**둘째, 선언(Declarative)이 곧 잠금(Lock)으로 이어집니다.** `Cargo.toml`에 원하는 버전 제약을 선언하면, Cargo가 의존성 그래프 전체를 해석해 `Cargo.lock`에 정확한 버전을 못 박습니다. 이 락 파일은 1.0부터 존재했고, 모든 Rust 프로젝트가 동일하게 사용합니다. 선택지가 없다는 것이 오히려 축복입니다.

**셋째, 컴파일은 숨겨진 비밀이 아니라 일급 시민입니다.** 파이썬에서 빌드는 `pip install` 뒤에 숨어 있다가 실패할 때만 정체를 드러냅니다. Rust에서 빌드는 명시적이고, 두 가지 모드(Debug/Release)로 나뉘며, 모든 단계가 관찰 가능합니다(`cargo build -v`).

이 세 가지가 합쳐지면, 파이썬에서 여러 도구로 흩어져 있던 책임이 단 하나의 명령어 네임스페이스(`cargo ...`)로 수렴합니다.

---

## 2.3 툴체인 관리: `pyenv`에서 `rustup`으로

파이썬 개발자는 프로젝트마다 다른 인터프리터 버전(3.8, 3.11, 3.12) 때문에 `pyenv`로 버전을 스위칭합니다. Rust도 버전 관리가 필요하지만, `rustup`이 이를 단일 도구로 흡수했고, 한 걸음 더 나아가 **채널(Channel)**과 **컴포넌트(Component)**라는 개념을 추가했습니다.

```bash
# The Python Way (pyenv)
$ pyenv install 3.11.6
$ pyenv local 3.11.6           # 현재 디렉터리에 고정
$ pyenv global 3.12.0          # 시스템 기본값 변경
```

```bash
# The Rust Way (rustup)
$ rustup update                # 모든 툴체인을 최신으로
$ rustup default stable        # 기본 채널을 stable로
$ rustup toolchain install nightly   # nightly 채널 추가 설치
$ rustup component add clippy rustfmt rust-src   # 컴포넌트 추가
```

### 채널: stable / beta / nightly

`pyenv`가 관리하는 것은 단순히 버전 번호입니다. `rustup`은 **출시 안정성 등급**인 채널을 관리합니다.

| 채널 | 주기 | 성격 | 파이썬에 비유하면 |
| :--- | :--- | :--- | :--- |
| **stable** | 6주마다 | 프로덕션용. 모든 안정화된 기능. | CPython 정식 릴리스 (3.13.6) |
| **beta** | 6주마다 | 다음 stable의 후보. 회귀 테스트용. | 릴리스 후보(rc) |
| **nightly** | 매일 | 실험적 기능(unstable feature) 사용 가능. | `__future__` 임포트 + 빌드 미리보기 |

Rust는 6주마다 한 칸씩 밀려 올라가는 **열차 모델(train model)**을 씁니다. 오늘의 nightly가 6주 뒤 beta가 되고, 다시 6주 뒤 stable이 됩니다. `#![feature(...)]` 같은 실험적 기능은 nightly에서만 켤 수 있으며, 안정화되기 전까지는 stable 빌드에서 컴파일조차 되지 않습니다. 파이썬에서 `from __future__ import annotations`로 미래 문법을 미리 끌어쓰던 경험과 정확히 같은 개념이되, **컴파일러가 강제**한다는 점이 다릅니다.

#### 컴파일러 에러 시연: stable에서 nightly 기능을 쓰면

말로만 "stable에선 안 된다"고 하면 와닿지 않으니, 실제로 막히는 장면을 보겠습니다. Rust에는 `box` 패턴(`Box`를 패턴 매칭으로 바로 풀어내는 문법)이 있는데, 이 기능은 십수 년째 nightly에 머물러 있고 stable에서는 컴파일되지 않습니다. 무심코 이 문법을 쓰면 어떻게 되는지 봅시다.

```rust
fn main() {
    let boxed: Box<i32> = Box::new(42);
    let box value = boxed;   // 🚨 컴파일 에러! box 패턴은 아직 nightly 전용
    println!("{}", value);
}
```

stable 1.96.0에서 `rustc`를 돌리면 정확히 이렇게 거부합니다.

```text
error[E0658]: box pattern syntax is experimental
 --> src/main.rs:3:9
  |
3 |     let box value = boxed;
  |         ^^^^^^^^^
  |
  = note: see issue #29641 <https://github.com/rust-lang/rust/issues/29641> for more information

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0658`.
```

`error[E0658]`은 "안정화되지 않은(unstable) 기능을 stable 채널에서 사용했다"는 뜻의, Rust 입문자가 만나는 가장 흔한 에러 코드 중 하나입니다. note에 적힌 issue 번호(여기선 #29641)는 그 기능의 안정화 추적 이슈이고, 거기서 "왜 아직 안정화 안 됐는지"의 논의 맥락을 볼 수 있습니다. 해결책은 둘 중 하나입니다. (1) `rustup default nightly`로 nightly 채널에 들어가 파일 맨 위에 `#![feature(box_patterns)]`를 선언하거나, (2) stable에서 동작하는 대안(`let value = *boxed;`)을 쓰는 것입니다. 프로덕션 코드라면 거의 항상 (2)를 택합니다. 참고로 stable에서 `#![feature(...)]` 자체를 적으면 또 다른 에러 `error[E0554]: #![feature] may not be used on the stable release channel`가 뜹니다. 즉 E0658(기능을 씀)과 E0554(기능 게이트를 켬)는 같은 벽의 양면입니다.

### 툴체인 핀(Pin): `rust-toolchain.toml`

특정 프로젝트가 특정 툴체인을 강제하고 싶다면, 프로젝트 루트에 `rust-toolchain.toml`을 둡니다. `pyenv`의 `.python-version`과 같은 역할이지만, 버전뿐 아니라 채널·컴포넌트·타깃까지 못 박을 수 있습니다.

```toml
# rust-toolchain.toml
[toolchain]
channel = "1.96.0"                       # 정확한 버전 핀 (또는 "stable", "nightly-2026-05-01")
components = ["clippy", "rustfmt"]        # 이 프로젝트에 필요한 컴포넌트
targets = ["x86_64-unknown-linux-musl"]  # 빌드 대상 플랫폼까지 명시
profile = "minimal"
```

이 파일이 있으면, 해당 디렉터리에서 `cargo`나 `rustc`를 실행하는 순간 `rustup`이 자동으로 지정된 툴체인으로 전환하고, 없으면 즉시 내려받습니다. 팀원이 "내 로컬 nightly에선 됐는데 CI에선 안 돼요" 같은 상황을 원천 차단합니다. `.python-version`은 버전만 적을 수 있었지만, `rust-toolchain.toml`은 **빌드 환경 전체를 코드로 박제**합니다.

> 💡 `pyenv`가 파이썬 인터프리터 바이너리를 통째로 빌드해 `~/.pyenv/versions/`에 쌓는 반면, `rustup`은 미리 컴파일된 툴체인을 `~/.rustup/toolchains/`에 내려받아 심볼릭 링크 셰임(shim)으로 전환합니다. `cargo`라는 이름의 셰임이 PATH 맨 앞에 있고, 실행될 때 현재 디렉터리의 핀을 읽어 실제 툴체인으로 라우팅합니다. 그래서 버전 전환이 즉각적입니다.

---

## 2.4 가상 환경의 소멸

파이썬 개발자는 디렉터리에 들어서면 숨 쉬듯 `source .venv/bin/activate`를 칩니다. 파이썬이 본능적으로 전역에 패키지를 설치하려 하기 때문에, 프로젝트별 의존성이 꼬이지 않도록 격리된 가짜 환경을 만들어야 했습니다. `poetry shell`이나 `uv run`이 이를 추상화했지만, 내부적으로는 여전히 `.venv` 폴더를 만들고 `sys.path`를 조작합니다.

```bash
# The Python Way
$ python -m venv .venv
$ source .venv/bin/activate    # 셸 프롬프트에 (.venv) 표시
(.venv) $ pip install requests
(.venv) $ deactivate           # 끝나면 비활성화
```

Rust에는 이 절차가 **존재하지 않습니다.**

```bash
# The Rust Way
$ cargo new my_project         # 프로젝트 스캐폴딩
$ cd my_project
$ cargo add reqwest            # 의존성 추가 (Cargo.toml 자동 수정)
$ cargo run                    # 빌드 + 실행
```

활성화도 비활성화도 없습니다. 그 이유를 메모리 모델 수준에서 봅시다.

> **Under the Hood — 의존성은 어디에 사는가**
>
> Cargo는 의존성을 두 곳으로 분리해 다룹니다.
>
> 1. **소스 캐시 (전역, 공유):** `~/.cargo/registry/`
>    - `cache/`: 다운로드한 `.crate` 압축 파일.
>    - `src/`: 압축 해제된 크레이트 소스 코드.
>    - 같은 버전의 `serde 1.0.210`은 머신 전체에서 **딱 한 번만** 내려받아 모든 프로젝트가 공유합니다. 디스크 절약 + 네트워크 절약.
>
> 2. **빌드 산출물 (프로젝트별, 격리):** `<project>/target/`
>    - `debug/` 또는 `release/` 아래에 컴파일된 `.rlib`(Rust 라이브러리), 오브젝트 파일, 최종 바이너리가 쌓입니다.
>    - 프로젝트 A의 `target/`과 프로젝트 B의 `target/`은 완전히 별개입니다.
>
> 파이썬의 `site-packages`는 **소스(=실행 단위)**가 전역에 깔리기 때문에 격리가 필요했습니다. Rust는 **소스는 공유하되 빌드 결과는 격리**합니다. 같은 크레이트라도 프로젝트마다 활성화한 feature나 컴파일 플래그가 다르면 `target/`의 산출물이 달라지므로, 프로젝트 격리가 자동으로 보장됩니다. 켜고 끌 "환경"이라는 개념 자체가 불필요해지는 것입니다.

여기서 따라오는 부수 효과: `target/`은 언제든 `cargo clean`으로 통째로 날려도 안전합니다. 소스 캐시가 그대로이므로 다시 빌드만 하면 됩니다. `.gitignore`에 `/target`만 넣으면 됩니다. 파이썬처럼 `.venv`, `__pycache__`, `*.pyc`, `.mypy_cache`, `.pytest_cache`, `.ruff_cache`를 줄줄이 무시 목록에 적을 필요가 없습니다.

---

## 2.5 `Cargo.toml` 심층 해부

최근 파이썬 생태계도 PEP 518/621을 통해 `setup.py`와 `requirements.txt` 대신 `pyproject.toml`로 수렴하고 있습니다. 흥미롭게도 이 변화는 Cargo의 `Cargo.toml`에서 직접 영감을 받았으며, `poetry.lock`/`Pipfile.lock` 역시 `Cargo.lock`의 결정론적 빌드 철학을 차용한 것입니다. 두 파일을 나란히 놓고 항목별로 해부해 봅시다.

### 비교: `pyproject.toml` vs `Cargo.toml`

```toml
# Python — pyproject.toml (Poetry 스타일)
[tool.poetry]                          # 'tool.poetry' 접두사: 서드파티 도구의 네임스페이스
name = "my-python-app"
version = "0.1.0"
description = "A sample python app"
authors = ["Pythonista <python@example.com>"]

[tool.poetry.dependencies]
python = "^3.10"
fastapi = "^0.95.0"
uvicorn = {extras = ["standard"], version = "^0.22.0"}  # extras = Rust의 features

[tool.poetry.group.dev.dependencies]
pytest = "^7.3.1"
black = "^23.3.0"
```

```toml
# Rust — Cargo.toml
[package]
name = "my_rust_app"
version = "0.1.0"
edition = "2021"                       # 언어 에디션 (문법/관용구 버전)
authors = ["Rustacean <rust@example.com>"]
rust-version = "1.74"                  # MSRV: 이 버전 미만에선 빌드 거부

[dependencies]
axum = "0.7"                           # fastapi 대체 (웹 프레임워크)
tokio = { version = "1.40", features = ["macros", "rt-multi-thread"] }  # 비동기 런타임
serde = { version = "1.0", features = ["derive"] }   # pydantic 대체 (직렬화)

[dev-dependencies]
tokio-test = "0.4"                     # 테스트에서만 쓰는 의존성

[build-dependencies]
cc = "1.0"                             # build.rs에서만 쓰는 의존성 (C 코드 컴파일 등)
```

`[tool.poetry]`라는 접두사가 필요 없다는 점에 주목하세요. 파이썬에서 `[tool.xxx]`는 "이건 표준이 아니라 xxx라는 도구의 설정"이라는 뜻입니다. Rust에서 `[package]`, `[dependencies]`는 **언어 표준 스키마**입니다. 어떤 Rust 프로젝트를 열어도 구조가 동일합니다.

### `edition` vs `rust-version` — 헷갈리기 쉬운 두 축

- **`edition`** (2015/2018/2021/2024): 언어의 **문법·관용구 세대**입니다. 파이썬 2 → 3 같은 큰 단절을 막으면서도 언어를 진화시키기 위한 장치로, 같은 컴파일러가 여러 에디션을 동시에 빌드할 수 있습니다. 에디션이 달라도 크레이트끼리는 자유롭게 섞입니다.
- **`rust-version`** (MSRV, Minimum Supported Rust Version): 이 크레이트를 빌드하는 데 필요한 **최소 컴파일러 버전**입니다. 파이썬의 `python = "^3.10"`에 대응합니다.

### 의존성의 세 종류

```toml
[dependencies]        # 런타임/라이브러리 본체에 포함됨. requirements.txt의 본문.
[dev-dependencies]    # 테스트·예제·벤치마크 빌드에만. requirements-dev.txt / poetry dev group.
[build-dependencies]  # build.rs(빌드 스크립트) 컴파일에만 쓰임. 파이썬엔 마땅한 대응이 없음.
```

`dev-dependencies`는 최종 바이너리에 **들어가지 않습니다.** 테스트용 mock 라이브러리를 넣어도 프로덕션 바이너리 크기에 영향을 주지 않습니다. 파이썬에서 `pytest`를 운영 도커 이미지에 실수로 포함시키는 사고가 Rust에선 구조적으로 불가능합니다.

### Features — 무비용 조건부 컴파일

`features`는 Cargo의 가장 강력하면서도 파이썬에 정확한 대응물이 없는 기능입니다. 라이브러리가 기능을 **컴파일 타임 플래그**로 쪼개 제공하고, 사용자는 필요한 것만 켭니다.

```toml
[dependencies]
# serde의 derive 매크로 기능만 켠다. 안 켜면 #[derive(Serialize)]가 컴파일 안 됨.
serde = { version = "1.0", features = ["derive"] }

# tokio의 풀 기능 대신 필요한 부분만. 멀티스레드 런타임 + 매크로만.
tokio = { version = "1.40", default-features = false, features = ["macros", "rt-multi-thread"] }

# reqwest에서 무거운 OpenSSL 대신 순수 Rust TLS(rustls)를 선택.
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
```

자기 크레이트에서도 feature를 정의할 수 있습니다.

```toml
[features]
default = ["std"]                      # 아무것도 안 적으면 켜지는 기본 feature
std = []                               # 표준 라이브러리 사용
extra_logging = ["dep:tracing"]        # 이 feature가 켜질 때만 tracing 의존성 활성화
```

```rust
// 코드에서는 cfg 속성으로 조건부 컴파일
#[cfg(feature = "extra_logging")]
fn log_request(req: &str) {
    tracing::info!("request: {}", req);  // extra_logging이 꺼지면 이 함수는 아예 컴파일되지 않음
}
```

> **Under the Hood — features가 "무비용"인 이유**
>
> 파이썬에서 선택적 기능은 보통 런타임에 처리됩니다. `if settings.ENABLE_LOGGING:` 같은 분기는 매 호출마다 조건을 평가하고, 안 쓰는 코드도 인터프리터에 로드됩니다. `try: import ujson except ImportError: import json` 패턴도 런타임 비용입니다.
>
> Rust의 `#[cfg(feature = "...")]`는 **전처리기처럼 컴파일 단계에서 코드를 통째로 포함하거나 제거**합니다. 꺼진 feature의 코드는 어셈블리에 존재하지 않습니다. 즉 분기 비용 0, 바이너리 크기 증가 0, 안 쓰는 의존성 다운로드 0입니다. `reqwest`에서 `rustls-tls`만 켜고 `default-features = false`로 OpenSSL을 빼면, 최종 바이너리에서 OpenSSL 관련 수백 KB의 기계어가 **애초에 생성되지 않습니다.** 이것이 무비용 추상화(Zero-cost Abstraction)가 빌드 시스템 수준에서 구현된 사례입니다.

### SemVer와 캐럿(`^`) 규칙

파이썬의 `^`와 Rust의 버전 문자열은 같은 SemVer 정신을 공유하지만, Cargo는 **버전 번호를 더 엄격하게 해석**합니다. 핵심은 "기본 표기가 곧 캐럿"이라는 점입니다.

| `Cargo.toml` 표기 | 의미 | 허용 범위 |
| :--- | :--- | :--- |
| `serde = "1.2.3"` | 캐럿과 동일 (`^1.2.3`) | `>=1.2.3, <2.0.0` |
| `serde = "1.2"` | `^1.2` | `>=1.2.0, <2.0.0` |
| `serde = "1"` | `^1` | `>=1.0.0, <2.0.0` |
| `serde = "0.7.3"` | 0.x는 마이너가 메이저 취급 | `>=0.7.3, <0.8.0` |
| `serde = "=1.2.3"` | 정확히 고정 | `1.2.3`만 |
| `serde = "~1.2.3"` | 틸드: 패치만 올림 | `>=1.2.3, <1.3.0` |

가장 중요한 함정은 **`0.x` 버전의 특수 규칙**입니다. SemVer에서 0.x는 "아직 불안정"을 뜻하므로, Cargo는 `0.7`의 마이너 번호 `7`을 메이저처럼 취급합니다. 따라서 `tokio = "0.7"`은 `0.8`로 자동 업데이트되지 **않습니다.** 파이썬 `poetry`도 `^0.7`을 `>=0.7.0, <0.8.0`으로 같게 해석하므로, 이 직관은 양쪽에서 통합니다. 다만 그냥 `"0.7"`이라고만 적어도 캐럿이 적용된다는 점이 Cargo의 차이입니다.

### `Cargo.lock` = `poetry.lock`

`Cargo.toml`이 **느슨한 제약**(`^1.0`)을 선언한다면, `Cargo.lock`은 빌드에 실제로 쓰인 **정확한 버전과 의존성 그래프 전체**를 기록합니다.

```toml
# Cargo.lock (발췌, 자동 생성 — 직접 편집 금지)
[[package]]
name = "serde"
version = "1.0.210"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "c8e3592472072e6e22e0a54d5904d9febf8508f65fb8552499a1abc7d1078c3a"
dependencies = ["serde_derive"]
```

`checksum` 필드가 핵심입니다. 이는 다운로드한 크레이트의 SHA-256 해시로, **공급망 공격(supply-chain attack)** 방어선입니다. 누군가 crates.io에 같은 버전 번호로 다른 내용을 올려도(crates.io는 게시된 버전의 덮어쓰기를 금지하지만) 체크섬이 다르면 빌드가 거부됩니다.

> 💡 **커밋 규칙:** 실행 가능한 애플리케이션(binary crate)의 `Cargo.lock`은 **반드시 git에 커밋**합니다. 라이브러리(library crate)의 `Cargo.lock`은 전통적으로 커밋하지 않았으나, 최근에는 CI 재현성을 위해 커밋하는 것이 권장됩니다. 파이썬의 "애플리케이션은 락을 커밋하고 라이브러리는 안 한다"는 관행과 동일합니다.

---

## 2.6 Workspaces, 그리고 로컬 path / git 의존성

파이썬 모노레포에서 API 서버, 워커, 공통 라이브러리를 한 저장소에 두려면 `PYTHONPATH` 조작이나 `pip install -e .`(editable install)의 미로를 헤매야 했습니다. 패키지 간 로컬 참조가 깨지면 디버깅이 고통스럽습니다.

Cargo는 **워크스페이스(Workspace)**를 일급 기능으로 제공합니다.

```toml
# 루트 Cargo.toml — 워크스페이스 정의
[workspace]
resolver = "2"                         # feature 해석 알고리즘 v2 (edition 2021 권장)
members = [
    "apps/api_server",
    "apps/worker",
    "libs/common_utils",
]

# 모든 멤버가 공유하는 의존성 버전 (DRY)
[workspace.dependencies]
tokio = { version = "1.40", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
```

```toml
# apps/api_server/Cargo.toml — 멤버가 워크스페이스 버전을 상속
[package]
name = "api_server"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { workspace = true }           # 루트의 버전을 그대로 사용
common_utils = { path = "../../libs/common_utils" }  # 로컬 라이브러리 참조
```

이렇게 하면 루트에서 `cargo build` 한 번으로 전체가 일관된 의존성으로 빌드되고, 모든 멤버가 **하나의 `Cargo.lock`과 하나의 `target/`을 공유**합니다. 같은 의존성을 여러 번 컴파일하지 않으므로 빌드가 빨라집니다.

### 로컬 path와 git 의존성

PyPI에 올리지 않은 코드를 쓰는 세 가지 방법이 모두 한 줄로 해결됩니다.

```toml
[dependencies]
# 1) crates.io에서 (기본)
serde = "1.0"

# 2) 로컬 경로 (pip install -e ../mylib 대체)
my_lib = { path = "../my_lib" }

# 3) git 저장소 직접 (pip install git+https://... 대체)
some_crate = { git = "https://github.com/org/repo", branch = "main" }
pinned_crate = { git = "https://github.com/org/repo", tag = "v1.2.0" }
exact_crate  = { git = "https://github.com/org/repo", rev = "a1b2c3d" }  # 커밋 해시 고정
```

파이썬에서 `pip install -e`는 패키지를 "편집 가능" 모드로 심볼릭 링크하고, git 의존성은 임시로 클론해 빌드하는 등 동작이 제각각이었습니다. Cargo는 셋 다 동일한 의존성 해석 파이프라인에 넣고, git 의존성의 정확한 커밋 해시까지 `Cargo.lock`에 기록해 재현성을 보장합니다.

---

## 2.7 빌드 파이프라인: Debug vs Release

파이썬은 소스를 바이트코드(`.pyc`)로 컴파일하면서 실행하므로 빌드 단계가 거의 보이지 않습니다. Rust는 AOT(Ahead-of-Time) 컴파일 언어라, 소스를 기계어로 번역하는 빌드가 필수이며 두 모드로 나뉩니다. 이 둘의 차이는 단순히 "빠르다/느리다"가 아니라, **생성되는 기계어와 안전성 검사 자체가 다릅니다.**

```bash
$ cargo build              # Debug 빌드 → target/debug/
$ cargo run                # Debug 빌드 후 실행
$ cargo build --release    # Release 빌드 → target/release/
$ cargo run --release      # Release 빌드 후 실행
```

| 항목 | Debug (`dev` 프로파일) | Release (`release` 프로파일) |
| :--- | :--- | :--- |
| 최적화 수준 (`opt-level`) | `0` (없음) | `3` (최대) |
| 컴파일 속도 | 빠름 | 느림 (수 배~수십 배) |
| 런타임 속도 | 느림 | C/C++에 필적 |
| 디버그 심볼 | 포함 | 기본 미포함(설정 가능) |
| **정수 오버플로** | **패닉(panic)으로 즉시 중단** | **wrapping(2의 보수 순환)** |
| `debug_assert!` | 활성 | 무시됨 |

### 정수 오버플로 — 모드에 따라 동작이 다른 안전장치

이것이 가장 중요하고 가장 놀라운 차이입니다.

```rust
fn main() {
    let x: u8 = 255;       // u8의 최댓값
    let y = x + 1;         // 256은 u8에 안 들어감
    println!("{}", y);
}
```

- **Debug 빌드**에서 실행하면 패닉으로 즉시 중단됩니다.

```text
thread 'main' panicked at src/main.rs:3:13:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

- **Release 빌드**에서 실행하면 패닉 없이 `y = 0`이 됩니다(2의 보수 순환, wrapping).

개발 중엔 오버플로 버그를 즉시 잡아내고, 운영 중엔 매 산술 연산마다 오버플로 검사를 넣는 비용을 없애겠다는 설계입니다. 파이썬의 `int`는 임의 정밀도라 오버플로가 없지만, 그 대가로 모든 정수가 힙에 할당된 `PyObject`이며 산술마다 객체를 새로 만듭니다(이 메모리 비용은 Chapter 3에서 바이트 단위로 해부합니다). Rust는 고정 크기 정수의 속도를 취하되, 의도치 않은 순환을 명시적으로 다루도록 강제합니다. 순환이 의도된 동작이라면 `wrapping_add`, 검사를 원하면 `checked_add`(`Option` 반환), 포화를 원하면 `saturating_add`를 명시적으로 씁니다.

### `[profile.release]` 튜닝

빌드 동작은 `Cargo.toml`에서 프로파일로 세밀하게 조정합니다. 다음은 "최소 크기 + 최대 속도" 배포용 설정입니다.

```toml
[profile.release]
opt-level = 3          # 0~3, 또는 "s"(크기 우선), "z"(크기 극단 우선)
lto = "fat"            # Link-Time Optimization: 크레이트 경계를 넘어 인라이닝/최적화
codegen-units = 1      # 코드 생성 단위를 1로 → 최적화 극대화(빌드는 느려짐)
panic = "abort"        # 패닉 시 스택 되감기(unwinding) 제거 → 바이너리 축소
strip = true           # 심볼 테이블 제거 → 바이너리 추가 축소
```

각 옵션을 풀어보면:

- **`lto`(Link-Time Optimization):** 기본적으로 각 크레이트는 따로 최적화됩니다. LTO를 켜면 링크 단계에서 **크레이트 경계를 허물고** 전역적으로 인라이닝·죽은 코드 제거를 수행합니다. `"thin"`(빠른 근사)과 `"fat"`(느리지만 철저)이 있습니다.
- **`codegen-units`:** 컴파일러는 병렬 빌드를 위해 크레이트를 여러 단위로 쪼개는데, 단위가 많으면 빌드는 빠르지만 단위 간 최적화 기회를 놓칩니다. `1`로 두면 단위 간 최적화가 가능해져 더 빠른 코드가 나오지만 빌드는 느려집니다.
- **`panic = "abort"`:** 기본값 `"unwind"`는 패닉 시 스택을 거슬러 올라가며 `Drop`을 호출하는 되감기 로직을 바이너리에 심습니다. `"abort"`는 그냥 프로세스를 죽입니다. 되감기 테이블이 사라지므로 바이너리가 작아지고, 일부 핫패스가 빨라집니다(단, `catch_unwind`가 무력화됨).

> **Under the Hood — Release가 빠른 진짜 이유와 실측**
>
> Rust 코드는 LLVM IR(중간 표현)로 낮춰진 뒤 LLVM 백엔드가 기계어로 번역합니다. `opt-level = 0`에서 LLVM은 IR을 거의 그대로 직역하여, 모든 지역 변수가 스택에 저장되고 함수 호출이 전부 실제 `call` 명령으로 남습니다. `opt-level = 3`에서는 LLVM이 인라인 확장, 루프 전개(loop unrolling), 자동 벡터화(SIMD), 죽은 코드 제거, 레지스터 할당 최적화를 수행합니다. 작은 함수는 호출 자체가 사라지고, 반복문은 CPU의 벡터 레지스터를 쓰도록 다시 쓰입니다.
>
> 간단한 실측 예: 1억 번 누적 합산하는 루프.
> ```rust
> fn main() {
>     let mut sum: u64 = 0;
>     for i in 0..100_000_000u64 {
>         sum = sum.wrapping_add(i);
>     }
>     println!("{}", sum);
> }
> ```
> 같은 코드를 측정하면 Debug는 대략 250~400ms, Release는 대략 0~5ms가 나옵니다(LLVM이 루프 전체를 닫힌 형태 공식으로 접거나 벡터화하기 때문). 차이는 수십 배에서, 루프가 통째로 상수 폴딩되면 수백 배까지 벌어집니다. **벤치마크나 성능 비교를 할 때 `--release` 없이 측정하면 의미가 없습니다.** 파이썬과 Rust를 비교한답시고 Debug 빌드로 재면, Rust를 부당하게 깎아내리는 셈입니다.

빌드 전 과정을 관찰하고 싶다면 `cargo build -v`로 `rustc`에 넘기는 실제 플래그를, `cargo build --timings`로 크레이트별 컴파일 시간 HTML 리포트를 볼 수 있습니다.

### 증분 컴파일과 `.fingerprint` — 두 번째 빌드는 왜 빠른가

"Rust는 컴파일이 느리다"는 말을 들어보셨을 겁니다. 절반만 맞습니다. **처음 한 번**은 느립니다. 하지만 두 번째부터는 이야기가 완전히 달라집니다. 실제로 측정해 봅시다. criterion 의존성을 가진 작은 프로젝트를 깨끗한 상태에서 빌드하면 의존성 트리(criterion, plotters, regex, serde 등 수십 개 크레이트)를 전부 컴파일하느라 14초가 걸립니다. 그런데 아무것도 바꾸지 않고 다시 빌드하면:

```text
$ cargo build --benches      # 첫 빌드: 의존성 전체 컴파일
   Compiling criterion v0.5.1
   ...
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 14.05s

$ cargo build --benches      # 두 번째 빌드: 변경 없음
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.05s
```

14.05초가 0.05초로 줄었습니다. 약 280배입니다. `rustc`가 아무 일도 하지 않았기 때문입니다. 그 판단의 근거가 `target/`에 숨어 있는 **`.fingerprint`** 디렉터리입니다.

> **Under the Hood — `.fingerprint`가 캐시를 무효화하는 방식**
>
> Cargo는 컴파일 단위(crate × profile × target)마다 `target/<profile>/.fingerprint/<crate>-<hash>/` 아래에 작은 메타데이터 파일들을 남깁니다. 그 안의 JSON을 직접 열어보면 이렇게 생겼습니다(들여쓰기는 편의상 추가).
>
> ```json
> {
>   "rustc": 9783247635632824192,    // 컴파일러 버전+커밋 해시
>   "features": "[]",                // 활성화된 feature 집합
>   "target": 12368643011565581116, // 빌드 타깃 트리플
>   "profile": 6675295047989516842, // opt-level, debuginfo, lto 등 프로파일 설정
>   "path": 4942398508502643691,    // 소스 경로
>   "deps": [],                     // 의존 크레이트들의 fingerprint
>   "rustflags": [],                // RUSTFLAGS
>   "config": 2069994364910194474   // 관련 cargo 설정
> }
> ```
>
> 빌드를 호출할 때마다 Cargo는 이 입력들을 다시 해싱한 새 fingerprint와 저장된 값을 비교합니다. **하나라도 다르면** 그 크레이트만 다시 컴파일합니다. 여기에 더해, 각 컴파일 단위에는 그것이 의존하는 모든 소스 파일 목록(`dep-info`)과 mtime이 기록되어 있어, 소스의 수정 시각이 바뀌면 재컴파일 대상이 됩니다.
>
> 그래서 `rustc` 버전을 올리거나(`rustc` 해시 변경), `Cargo.toml`에서 feature를 켜거나(`features` 변경), `--release`로 바꾸거나(`profile` 변경), `RUSTFLAGS`를 건드리면(`rustflags` 변경) **그 변경의 영향을 받는 부분만** 다시 빌드됩니다. 의존성은 그대로이므로 두 번째 빌드가 0.05초로 끝나는 것입니다. 파이썬의 `.pyc` 캐시(`__pycache__`)가 소스 mtime만 보고 단순 무효화하는 것과 달리, Cargo의 fingerprint는 **컴파일러 버전·feature·플래그·의존성 그래프 전체**를 입력으로 잡아 정밀하게 무효화합니다.
>
> 이것이 `target/`을 절대 `.gitignore` 밖에 두면 안 되는 이유이기도 합니다. 이 디렉터리는 단순한 산출물 더미가 아니라 머신·컴파일러 버전에 강하게 결합된 **재현 불가능한 빌드 캐시**입니다. 다른 머신에 그대로 복사해도 fingerprint가 어긋나 대부분 무효화됩니다. 반면 `cargo clean` 후 다시 빌드하면 동일한 결과가 나옵니다 — 재현성은 `Cargo.lock`과 소스가 보장하지, `target/`이 보장하는 게 아닙니다.

증분 컴파일은 자기 크레이트 **내부**에서도 작동합니다. `target/debug/incremental/`에는 함수 단위로 쪼갠 중간 산출물이 캐시되어, 한 함수만 고치면 그 함수가 속한 코드 생성 단위만 다시 만듭니다. 이 때문에 편집-저장-`cargo check` 루프가 첫 빌드 이후로는 거의 즉각적으로 느껴집니다.

---

## 2.8 타깃 아키텍처, 크로스 컴파일, 그리고 단일 바이너리 배포

이제 파이썬 배포 지옥과 가장 극명하게 대비되는 지점입니다. 파이썬 애플리케이션을 배포하려면 대상 서버에 호환되는 파이썬 인터프리터, 모든 의존성(맞는 휠), 그리고 그 의존성들의 C 라이브러리까지 갖춰야 합니다. 그래서 도커 이미지에 `python:3.11-slim` 베이스(150MB+)를 깔고, `apt-get install`로 시스템 라이브러리를 넣고, `pip install`로 의존성을 쌓는 다층 레이어가 필요합니다.

Rust는 **단 하나의 정적 링크된 바이너리**로 이 모든 것을 대체합니다.

### 크로스 컴파일

```bash
# 현재 머신의 기본 타깃 확인
$ rustc -vV | grep host
host: aarch64-apple-darwin

# 다른 플랫폼용 타깃 추가
$ rustup target add x86_64-unknown-linux-musl

# 그 타깃으로 빌드 (Apple Silicon Mac에서 리눅스 x86_64 바이너리 생성)
$ cargo build --release --target x86_64-unknown-linux-musl
```

`--target`을 지정하면 산출물이 `target/x86_64-unknown-linux-musl/release/`에 떨어집니다. 파이썬에서 "맥에서 리눅스용으로 빌드"는 거의 불가능에 가까웠지만(C 확장 때문에 결국 도커 빌드로 우회), Rust는 표준 절차입니다. 실무에서는 C 의존성의 링킹 문제를 깔끔히 처리하기 위해 `cross`(도커 기반 크로스 컴파일 도구)를 흔히 씁니다.

### glibc vs musl — 정적 링크의 핵심

`x86_64-unknown-linux-gnu`(기본 리눅스 타깃)는 시스템의 **glibc에 동적 링크**합니다. 즉 대상 서버에 호환되는 glibc 버전이 있어야 합니다. 반면 `x86_64-unknown-linux-musl`은 **musl libc를 정적 링크**하여, 외부 C 라이브러리에 전혀 의존하지 않는 **완전 정적 바이너리**를 만듭니다.

```bash
# musl로 빌드한 바이너리의 의존성 확인 → 외부 동적 라이브러리 없음
$ ldd target/x86_64-unknown-linux-musl/release/my_rust_app
        statically linked
```

이 정적 바이너리는 커널만 호환되면 어떤 리눅스 배포판에서도 그대로 실행됩니다. Alpine, Ubuntu, CentOS, 심지어 빈 컨테이너에서도 동작합니다.

> **Under the Hood — `FROM scratch` 도커 이미지와 바이너리 크기**
>
> 정적 바이너리는 다음과 같은 극단적으로 작은 도커 이미지를 가능하게 합니다.
> ```dockerfile
> # 1단계: 빌드
> FROM rust:1.96 AS builder
> WORKDIR /app
> COPY . .
> RUN rustup target add x86_64-unknown-linux-musl && \
>     cargo build --release --target x86_64-unknown-linux-musl
>
> # 2단계: 텅 빈 이미지에 바이너리 하나만
> FROM scratch
> COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/my_rust_app /
> ENTRYPOINT ["/my_rust_app"]
> ```
> `scratch`는 파일이 0개인 빈 이미지입니다. OS도, 셸도, 인터프리터도 없습니다. 최종 이미지 크기는 **바이너리 크기와 거의 같습니다.**
>
> 같은 "Hello, axum" 웹 서버를 기준으로 한 대략적 비교:
>
> | 배포 방식 | 이미지/산출물 크기(개략) |
> | :--- | :--- |
> | 파이썬(`python:3.11-slim` + FastAPI + uvicorn) | 180~250 MB |
> | Rust(`debian:slim` + 동적 링크 바이너리) | 80~90 MB |
> | Rust(`scratch` + musl 정적 바이너리) | 8~20 MB |
> | Rust(musl + `strip` + `opt-level="z"` + LTO) | 2~6 MB |
>
> 바이너리 크기를 더 줄이려면 `strip = true`로 심볼을 제거하고, `opt-level = "z"`(크기 극단 최적화), `panic = "abort"`로 되감기 테이블을 빼고, 필요하면 `cargo-bloat`로 어느 의존성이 바이너리를 부풀리는지 추적합니다. "Hello World" 수준의 단순 프로그램은 수백 KB까지 내려갑니다. 공격 표면(attack surface)도 극적으로 줄어듭니다. 셸이 없으니 컨테이너에 침투해도 `sh`를 띄울 수 없습니다.

배포가 `scp my_rust_app server:/usr/local/bin/` 한 줄로 끝나는 경험은, `requirements.txt`와 시스템 라이브러리 버전을 맞추느라 씨름하던 파이썬 개발자에게 거의 충격에 가깝습니다.

---

## 2.9 crates.io, 그리고 `cargo`라는 단일 명령어 우주

파이썬의 PyPI에 대응하는 것이 Rust의 **crates.io**입니다. `cargo add reqwest`는 PyPI의 `pip install`처럼 동작하되, 자동으로 `Cargo.toml`을 수정하고 최신 호환 버전을 캐럿 표기로 넣어줍니다. 둘의 가장 큰 문화적 차이는 **품질의 상향 평준화**입니다. 거의 모든 인기 크레이트가 `cargo doc`으로 생성된 표준 형식의 문서를 docs.rs에 자동 게시하므로, 라이브러리마다 문서 스타일이 제각각인 파이썬과 달리 탐색 경험이 일관됩니다.

> **Under the Hood — sparse 레지스트리 프로토콜과 "왜 빨라졌는가"**
>
> Cargo가 의존성을 해석하려면 먼저 "crates.io에 어떤 크레이트의 어떤 버전이 있는가"라는 **인덱스(index)**를 알아야 합니다. 2023년 이전(Cargo 1.70 미만)에는 이 인덱스가 하나의 거대한 **git 저장소**였습니다. `cargo`는 이 저장소를 통째로 `~/.cargo/registry/index/github.com-<hash>/`에 클론하고, 새 크레이트가 나올 때마다 `git fetch`로 받아왔습니다. crates.io에 16만 개가 넘는 크레이트가 쌓이면서 이 git 인덱스는 수백 MB로 불어났고, 깨끗한 머신에서 첫 `cargo build`가 인덱스 클론에만 수십 초~수 분을 잡아먹는 악명 높은 병목이 되었습니다.
>
> 1.70부터 기본값이 된 **sparse 프로토콜**은 이 구조를 버렸습니다. 이제 인덱스는 git이 아니라 평범한 HTTP 정적 파일 서버(`https://index.crates.io/`)이고, Cargo는 **자기 프로젝트가 실제로 의존하는 크레이트의 메타데이터만** 개별 HTTPS GET으로 가져옵니다. 로컬 인덱스 디렉터리 이름이 바뀐 것으로 이 전환을 직접 확인할 수 있습니다.
>
> ```text
> # 과거(git 프로토콜)
> ~/.cargo/registry/index/github.com-1ecc6299db9ec823/
>
> # 현재(sparse 프로토콜, 1.70+)
> ~/.cargo/registry/index/index.crates.io-1949cf8c6b5b557f/
> #                       └ "index.crates.io" 접두사가 sparse임을 가리킴
> ```
>
> 그 안의 `config.json`은 단 네 줄입니다.
>
> ```json
> { "dl": "https://static.crates.io/crates", "api": "https://crates.io" }
> ```
>
> Cargo는 `serde`가 필요하면 `index.crates.io/se/rd/serde` 같은 경로로 그 크레이트의 버전 목록 한 줄만 받아오고, 전체 인덱스를 복제하지 않습니다. 받은 메타데이터는 `.cache/`에 크레이트별 파일로 저장되어 다음 빌드에서 재사용됩니다.
>
> **이것이 정확히 uv가 파이썬에서 빠른 이유와 같은 원리입니다.** 전통적인 `pip`는 의존성을 풀 때 후보 버전마다 sdist/휠 메타데이터를 따로 내려받아 느렸습니다. uv는 PyPI의 JSON API와 HTTP Range 요청으로 **필요한 메타데이터만** 골라 받고, 그것을 전역 캐시에 저장해 재사용합니다. "전체를 복제하지 말고, 필요한 조각만 HTTP로 가져와 캐시하라"는 동일한 설계 철학이, Rust에서는 sparse 레지스트리로, 파이썬에서는 uv로 각각 구현된 셈입니다. 거대 모노레포의 CI에서 sparse 전환 후 인덱스 갱신 시간이 분 단위에서 초 단위로 떨어진 것은 흔한 경험입니다.

핵심 명령어를 파이썬 대응과 함께 정리합니다.

| 역할 | Python 생태계 | Rust (`cargo`) | 비고 |
| :--- | :--- | :--- | :--- |
| 패키지 추가 | `pip install`, `poetry add` | `cargo add` | `Cargo.toml` 자동 수정 |
| 빌드/실행 | `python main.py` | `cargo run` | AOT 컴파일 후 실행 |
| 빌드만 | (없음) | `cargo build` | 산출물만 생성 |
| 빠른 타입체크 | `mypy` | `cargo check` | 코드 생성 생략, 컴파일 에러만 빠르게 |
| 테스트 | `pytest` | `cargo test` | 자동 병렬 실행, 문서 예제까지 테스트 |
| 포매팅 | `black`, `isort` | `cargo fmt` | 공식 컨벤션, 설정 거의 불필요 |
| 정적 분석 | `flake8`, `pylint`, `ruff` | `cargo clippy` | 관용구·성능까지 지적 |
| 문서화 | `Sphinx`, `MkDocs` | `cargo doc --open` | 주석 기반 자동 생성 |
| 벤치마크 | `pytest-benchmark` | `cargo bench` | 통계적 측정 |
| 의존성 트리 | `pipdeptree` | `cargo tree` | 의존성 그래프 시각화 |
| 빌드 스크립트 | 커스텀 `setup.py` | `build.rs` | C 링킹·코드 생성 |
| 게시 | `twine upload` | `cargo publish` | crates.io에 배포 |

### `cargo check` — 파이썬에 없는 무기

`cargo check`는 **타입 검사와 빌림 검사(borrow check)까지만 수행하고 기계어 생성은 건너뜁니다.** 코드가 컴파일될 수 있는지를 `cargo build`보다 수 배 빠르게 확인합니다. 편집기에서 저장할 때마다 도는 것이 보통 이 `cargo check`입니다. 파이썬의 `mypy`가 하는 일과 비슷하지만, Rust에선 타입 검사가 별도 도구가 아니라 컴파일러의 일부라는 점이 다릅니다.

### 생태계 도구들

표준 도구 외에, `cargo install`로 설치하는 서브커맨드들이 경험을 완성합니다.

```bash
$ cargo install cargo-watch cargo-nextest cargo-edit cargo-audit

$ cargo watch -x check -x test    # 파일 변경 감지 시 자동 재실행 (pytest-watch 대체)
$ cargo nextest run               # 차세대 테스트 러너 (더 빠르고 출력이 명확)
$ cargo audit                     # Cargo.lock의 알려진 보안 취약점 스캔 (pip-audit 대체)
$ cargo tree -i serde             # serde를 누가 끌어오는지 역추적
```

`cargo-edit`는 `cargo add`/`cargo rm`을 제공했는데, 인기가 많아 Cargo 본체에 흡수되었습니다. `cargo add`는 1.62부터, `cargo remove`는 1.66부터 외부 크레이트 설치 없이 바로 쓸 수 있습니다(둘 다 `cargo add --help`/`cargo remove --help`로 확인됩니다). 다시 말해 의존성 추가/제거를 위해 더는 `cargo install cargo-edit`를 할 필요가 없습니다. 생태계의 좋은 아이디어가 표준으로 승격되는 흐름이 파이썬과 다른 점입니다.

### 흔한 함정: `cargo install`로 라이브러리를 설치하려는 시도

`pip install requests`가 손에 익은 파이썬 개발자가 Rust에서 가장 먼저 저지르는 실수가 이것입니다. `serde`를 쓰고 싶어서 `cargo install serde`를 칩니다. 그러면 이런 에러가 돌아옵니다.

```text
$ cargo install serde
    Updating crates.io index
error: there is nothing to install in `serde v1.0.228`, because it has no binaries
`cargo install` is only for installing programs, and can't be used with libraries.
To use a library crate, add it as a dependency to a Cargo project with `cargo add`.
```

핵심은 `pip install`과 `cargo install`이 **이름만 비슷할 뿐 전혀 다른 도구**라는 점입니다. `pip install`은 라이브러리든 CLI 도구든 가리지 않고 현재 환경에 설치합니다. 반면 `cargo install`은 오직 **실행 파일(바이너리 크레이트)**을 `~/.cargo/bin/`에 전역 설치하는, 파이썬으로 치면 `pipx install`에 해당하는 명령입니다. `ripgrep`, `cargo-nextest`, `cargo-audit` 같은 개발자 도구를 까는 용도이지, 프로젝트 의존성을 추가하는 용도가 아닙니다.

라이브러리를 프로젝트에 추가하는 올바른 방법은 `cargo add`입니다.

```bash
# 🚨 틀림: serde는 라이브러리라 설치할 바이너리가 없다
$ cargo install serde

# ✅ 맞음: 현재 프로젝트의 의존성으로 추가 (Cargo.toml 자동 수정)
$ cargo add serde --features derive
$ cargo add serde_json
```

이 구분이 처음에는 불편하게 느껴지지만, 사실 파이썬의 고질병 하나를 구조적으로 막아줍니다. 파이썬에서 CLI 도구(`black`, `httpie`)를 `pip install`로 깔면 프로젝트 의존성과 전역 도구가 같은 `site-packages`에서 뒤엉켜 버전 충돌을 일으킵니다(그래서 `pipx`가 등장했습니다). Cargo는 이 둘을 애초에 다른 명령(`cargo add` vs `cargo install`)으로 분리해, 도구와 의존성이 섞일 여지를 없앴습니다.

### Clippy — "가르치는 린터"

10년 차 파이썬 개발자가 가장 감탄하는 도구는 단연 **Clippy**입니다. `flake8`이 줄 길이와 미사용 변수를 잡고, `ruff`가 거기에 속도를 더한다면, Clippy는 **코드의 구조와 관용구**를 교정합니다. 의도적으로 비효율적인 코드를 보여드리죠.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let mut sum = 0;
    // C 스타일 인덱스 루프 — Rust답지 않음
    for i in 0..numbers.len() {
        sum += numbers[i];
    }
    println!("{}", sum);
}
```

`cargo clippy`를 돌리면:

```text
warning: the loop variable `i` is only used to index `numbers`
 --> src/main.rs:5:14
  |
5 |     for i in 0..numbers.len() {
  |              ^^^^^^^^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#needless_range_loop
  = note: `#[warn(clippy::needless_range_loop)]` on by default
help: consider using an iterator
  |
5 |     for <item> in &numbers {
  |
```

Clippy는 단순히 "잘못됐다"가 아니라, **왜 그런지(인덱스 i는 색인에만 쓰임)**, **어떻게 고치는지(이터레이터를 쓰라)**, 그리고 **더 배울 링크**까지 제시합니다. `numbers.iter().sum()`이라는 더 빠르고 안전한 관용구로 이끄는 것이죠. Clippy의 700개 넘는 린트는 그 자체가 Rust 베스트 프랙티스 교과서입니다. 독학자에게는 24시간 대기하는 시니어 멘토와 같습니다. CI에서 `cargo clippy -- -D warnings`로 모든 경고를 에러로 격상시키면, 팀 전체의 코드 품질을 자동으로 끌어올릴 수 있습니다.

### `build.rs` — 빌드 그 자체를 코드로

위 매핑 표에서 파이썬의 커스텀 `setup.py`에 대응하는 것으로 `build.rs`를 들었습니다. 이는 **빌드 직전에 컴파일·실행되는 Rust 프로그램**입니다. 프로젝트 루트에 `build.rs`를 두면 Cargo가 본 빌드 전에 이를 먼저 컴파일해 실행하고, 그 출력으로 빌드 동작을 조정합니다. 파이썬에서 `setup.py`가 C 확장을 컴파일하거나 버전 문자열을 주입하던 역할과 같되, **임의의 빌드 스크립트가 아니라 타입 안전한 Rust 코드**라는 점이 다릅니다.

```rust
// build.rs — 본 빌드 전에 실행됨
use std::process::Command;

fn main() {
    // 1) 빌드 시점의 git 커밋 해시를 환경 변수로 주입
    let output = Command::new("git")
        .args(["rev-parse", "--short", "HEAD"])
        .output()
        .expect("git 실행 실패");
    let hash = String::from_utf8(output.stdout).unwrap();
    println!("cargo:rustc-env=GIT_HASH={}", hash.trim());

    // 2) 특정 파일이 바뀔 때만 이 스크립트를 재실행하도록 캐시 힌트
    println!("cargo:rerun-if-changed=proto/schema.proto");
}
```

`cargo:`로 시작하는 출력은 Cargo에 대한 지시입니다. `rustc-env`로 환경 변수를 주입하면, 본 코드에서 `env!("GIT_HASH")` 매크로로 **컴파일 타임에** 그 값을 박아넣을 수 있습니다(런타임 비용 0). C 라이브러리 링킹(`cargo:rustc-link-lib`), Protobuf/gRPC 코드 생성(`tonic-build`, `prost-build`), C 소스 컴파일(`cc` 크레이트) 등이 모두 `build.rs`를 통해 이루어집니다. 파이썬에서 빌드 커스터마이즈가 `setuptools`의 난해한 API에 의존했다면, 여기선 그냥 평범한 Rust입니다.

### 수치로 보는 차이: 빌드 모드와 언어의 성능

마지막으로 모든 것을 관통하는 수치 하나를 봅시다. 동일한 알고리즘(1부터 N까지 소수를 세는 단순 시행 나눗셈)을 파이썬과 Rust로 작성해 측정하면, 대략 다음과 같은 양상이 나옵니다(N = 1,000,000 기준, 측정 환경에 따라 배수는 달라짐).

| 실행 방식 | 상대 속도(개략) | 비고 |
| :--- | :--- | :--- |
| CPython 3.13.6 | 1× (기준) | 인터프리터 + `PyObject` 박싱 |
| Rust `cargo run` (Debug) | 3~8× 빠름 | `opt-level=0`, 최적화 거의 없음 |
| Rust `--release` (`opt-level=3`) | 30~100× 빠름 | LLVM 인라이닝·벡터화 |
| Rust `--release` + LTO + `codegen-units=1` | 위에서 추가로 5~20% | 크레이트 경계 최적화 |

여기서 두 가지 교훈이 나옵니다. 첫째, **Debug 빌드만으로도 파이썬보다 빠를 때가 많지만**, 그것을 Rust의 진짜 실력으로 오해하면 안 됩니다. Release와 Debug 사이에 한 자릿수 배수의 차이가 흔합니다. 둘째, 마케팅에서 흔히 보는 "Rust는 파이썬보다 100배 빠르다" 같은 문구는 **특정 워크로드의 Release 빌드 수치**이며, I/O 바운드나 이미 C로 구현된 라이브러리(`numpy`)를 호출하는 코드에서는 격차가 훨씬 줄어듭니다. 우리는 항상 측정 맥락(입력 크기, 빌드 모드)을 명시하는 정직한 비교를 추구합니다. 동시성·이터레이터·PyO3 챕터에서 이 수치들을 워크로드별로 더 정밀하게 다시 측정할 것입니다.

### criterion으로 정직하게 측정하기 — 미니 벤치 하니스

위 표의 "개략" 배수가 미덥지 않다면, 직접 재면 됩니다. 파이썬에서 `pytest-benchmark`나 `timeit`을 쓰듯, Rust에는 통계적으로 엄밀하게 측정해 주는 사실상의 표준 크레이트 **criterion**이 있습니다. `cargo bench`의 기본 하니스(`#[bench]`)는 아직 nightly 전용이라, 실무에서는 stable에서 동작하는 criterion을 씁니다. 워밍업, 다회 샘플링, 이상치 탐지, 회귀 비교까지 자동으로 해줍니다.

소수를 세는 함수 하나로 미니 하니스를 만들어 봅시다. 측정 대상은 라이브러리로 분리합니다(벤치가 `pub` 함수를 호출해야 하므로).

```toml
# Cargo.toml
[package]
name = "prime_bench"
version = "0.1.0"
edition = "2021"

[dev-dependencies]            # 벤치는 테스트처럼 dev에서만 빌드됨
criterion = "0.5"

[[bench]]
name = "primes"
harness = false               # 기본 하니스 끄고 criterion이 main을 제공
```

```rust
// src/lib.rs — 측정 대상
/// 시행 나눗셈으로 n 미만의 소수 개수를 센다.
pub fn count_primes(n: u64) -> u64 {
    let mut count = 0;
    for candidate in 2..n {
        let mut is_prime = true;
        let mut d = 2;
        while d * d <= candidate {
            if candidate % d == 0 {
                is_prime = false;
                break;
            }
            d += 1;
        }
        if is_prime {
            count += 1;
        }
    }
    count
}
```

```rust
// benches/primes.rs — criterion 하니스
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use prime_bench::count_primes;

fn bench_count_primes(c: &mut Criterion) {
    c.bench_function("count_primes 100_000", |b| {
        // black_box: 입력을 "컴파일러가 미리 계산할 수 없는 값"으로 위장.
        // 이게 없으면 LLVM이 결과를 상수 폴딩해 0ns가 측정될 수 있다.
        b.iter(|| count_primes(black_box(100_000)))
    });
}

criterion_group!(benches, bench_count_primes);
criterion_main!(benches);
```

`cargo bench`를 돌리면 criterion이 워밍업 후 100개 샘플을 수집해 신뢰구간과 함께 보고합니다.

```text
$ cargo bench
count_primes 100_000    time:   [2.1039 ms 2.2047 ms 2.3649 ms]
Found 8 outliers among 100 measurements (8.00%)
```

가운데 값이 중앙 추정치(약 2.2 ms)이고, 양옆은 신뢰구간 하한·상한입니다. 같은 알고리즘을 파이썬으로 그대로 옮겨 측정하면(본문 측정 환경: Python 3.13.6) 약 99.6 ms가 나옵니다.

```python
# 동일 알고리즘, 동일 입력 (n = 100,000)
def count_primes(n: int) -> int:
    count = 0
    for candidate in range(2, n):
        is_prime = True
        d = 2
        while d * d <= candidate:
            if candidate % d == 0:
                is_prime = False
                break
            d += 1
        if is_prime:
            count += 1
    return count
# 측정: 약 99.6 ms (5회 best-of)
```

99.6 ms / 2.2 ms ≈ **45배**. 마케팅 슬로건의 "100배"가 아니라, 이 워크로드·이 입력·Release 빌드 기준의 **측정된 45배**입니다. `black_box`를 빼면 LLVM이 결과를 상수로 접어버려 0에 가까운 수치가 나오는데, 그 숫자로 "1000배 빠르다"고 주장하는 것이 바로 우리가 피하려는 거짓 비교입니다. criterion은 이런 함정을 막아주는 도구이자, 정직한 수치를 강제하는 규율입니다.

---

## 2.10 실전: 빈 폴더에서 정적 배포 바이너리까지

이론을 하나의 흐름으로 묶어, 의존성 있는 작은 CLI를 만들고 최적화·정적 빌드까지 가는 전 과정을 보여드립니다. JSON 한 줄을 받아 파싱해 출력하는 도구입니다.

```bash
# 1. 프로젝트 생성 (가상 환경? 없음)
$ cargo new json_echo
$ cd json_echo

# 2. 의존성 추가 — Cargo.toml이 자동 수정됨
$ cargo add serde --features derive
$ cargo add serde_json
```

```rust
// src/main.rs
use serde::{Deserialize, Serialize};

// pydantic 모델에 대응 — 컴파일 타임에 직렬화 코드가 생성됨 (Chapter 7에서 심화)
#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u8,
}

fn main() {
    let raw = r#"{"name": "Heechan", "age": 35}"#;

    // 역직렬화: 실패하면 Result로 에러 반환 (Chapter 12에서 심화)
    let user: User = serde_json::from_str(raw).expect("invalid JSON");
    println!("parsed: {:?}", user);

    // 다시 직렬화
    let back = serde_json::to_string(&user).unwrap();
    println!("re-serialized: {}", back);
}
```

```bash
# 3. 개발 중: 빠른 빌드로 동작 확인
$ cargo run
   Compiling json_echo v0.1.0 (/.../json_echo)
    Finished `dev` profile [unoptimized + debuginfo]
     Running `target/debug/json_echo`
parsed: User { name: "Heechan", age: 35 }
re-serialized: {"name":"Heechan","age":35}

# 4. 코드 품질 점검 (이 흐름이 곧 CI 파이프라인)
$ cargo fmt           # 포매팅
$ cargo clippy        # 린팅
$ cargo test          # 테스트 (지금은 없지만 한 줄로 추가 가능)
```

배포용 최적화 빌드 프로파일을 추가합니다.

```toml
# Cargo.toml 끝에 추가
[profile.release]
opt-level = "z"        # 크기 최우선
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

```bash
# 5. 정적 단일 바이너리 빌드 (리눅스 x86_64 대상)
$ rustup target add x86_64-unknown-linux-musl
$ cargo build --release --target x86_64-unknown-linux-musl

# 6. 산출물 확인 — 외부 의존성 0, 단일 파일
$ ldd target/x86_64-unknown-linux-musl/release/json_echo
        statically linked
$ ls -lh target/x86_64-unknown-linux-musl/release/json_echo
# 수 MB 단위의 단일 실행 파일. 이 파일 하나를 scratch 이미지에 넣으면 배포 끝.
```

새 팀원의 온보딩은 이게 전부입니다. `git clone` 후 `cargo run`. `pyenv`, `venv` 활성화, `poetry install`, 시스템 라이브러리 설치, pre-commit 설정의 행렬이 **단 두 명령**으로 압축됩니다.

---

## 2.11 Best Practices 체크리스트

실무에서 바로 적용할 수 있는 권장 사항입니다.

- [ ] **`Cargo.lock`을 커밋하라.** 애플리케이션은 필수, 라이브러리도 CI 재현성을 위해 권장됩니다.
- [ ] **`rust-toolchain.toml`로 툴체인을 핀하라.** 팀과 CI가 동일한 컴파일러·컴포넌트를 쓰도록 박제합니다.
- [ ] **CI에 `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`를 넣어라.** 이 세 줄이 파이썬의 black+ruff+mypy+pytest 조합 전체를 대체합니다.
- [ ] **`default-features = false`로 의존성을 다이어트하라.** 필요한 feature만 켜면 빌드 시간·바이너리 크기·공격 표면이 모두 줄어듭니다.
- [ ] **편집 중에는 `cargo check`/`cargo clippy`로, 성능 측정은 반드시 `--release`로.** Debug 수치로 성능을 논하지 마세요.
- [ ] **배포는 정적 바이너리(musl) + `scratch`/`distroless` 이미지로.** 수십 MB → 수 MB로 줄고 공격 표면이 사라집니다.
- [ ] **모노레포는 처음부터 `[workspace]`로 구성하라.** 공유 `Cargo.lock`과 `target/`이 빌드를 빠르게 합니다.
- [ ] **`cargo audit`을 CI에 추가하라.** `Cargo.lock`의 알려진 취약점을 자동 차단합니다(파이썬 `pip-audit` 대응).
- [ ] **`.gitignore`에는 `/target`만 넣으면 된다.** 파이썬의 길고 긴 무시 목록이 한 줄로 끝납니다.

---

## 2.12 흔한 함정 (파이썬 개발자가 자주 하는 실수)

오랜 파이썬 습관이 Rust에서 헛수고나 오해로 이어지는 대표 사례들입니다.

**1) "가상 환경을 만들어야지" — 안 만들어도 됩니다.**
`python -m venv .venv`에 해당하는 명령을 찾으려 하지 마세요. Cargo는 프로젝트별로 `target/`을 격리하므로 활성화/비활성화 개념이 없습니다. 이 한 가지 습관을 버리는 것이 멘탈 모델 전환의 출발점입니다.

**2) Debug 빌드로 성능을 측정하는 실수.**
`cargo run`으로 벤치마크를 돌려놓고 "Rust가 생각보다 안 빠르네?"라고 결론짓는 경우가 흔합니다. Debug는 `opt-level=0`이라 파이썬과 비슷하거나 더 느릴 수도 있습니다. **반드시 `--release`로 측정하세요.** Release에서 LLVM의 인라이닝·벡터화가 들어가야 비로소 진짜 성능이 나옵니다.

**3) `target/`을 git에 커밋하는 실수.**
`target/`은 수 GB까지 부풀 수 있는 빌드 캐시입니다. `.gitignore`에 `/target`을 넣으세요. 반대로 `Cargo.lock`은 **커밋해야** 합니다. 둘을 헷갈리지 마세요.

**4) "최신 버전이 좋겠지" 하며 `*`로 버전을 여는 실수.**
`serde = "*"`는 가능은 하지만 위험합니다. 캐럿 기본 동작(`serde = "1"`)을 믿으세요. SemVer 메이저 안에서만 자동 업데이트되어 호환성이 보장됩니다.

**5) `0.x` 버전을 메이저처럼 다루지 않는 실수.**
`tokio = "0.7"`은 `0.8`로 올라가지 **않습니다.** 0.x에선 마이너가 메이저 역할을 합니다. `0.x` 의존성을 올릴 때는 변경 로그를 반드시 확인하세요. 깨지는 변경이 마이너 번호에 숨어 있습니다.

**6) feature를 런타임 설정으로 착각하는 실수.**
`features`는 환경 변수나 설정 파일이 아니라 **컴파일 타임 플래그**입니다. 실행 시점에 켜고 끌 수 없습니다. 런타임 토글이 필요하면 일반적인 `if`나 설정 구조체를 쓰세요. feature는 "이 코드를 아예 빌드에 포함할지"를 결정합니다.

**7) `expect`/`unwrap`을 `try-except`처럼 남발하는 실수.**
파이썬의 `try/except`에 익숙해 `let x = result.unwrap();`을 습관적으로 쓰면, 에러 시 패닉으로 프로세스가 죽습니다. 이는 `Result`를 무시하는 것입니다. 올바른 에러 전파는 `?` 연산자이며, 이는 Chapter 12~13에서 본격적으로 다룹니다. 예제·프로토타입에서만 `unwrap`을 쓰세요.

**8) `cargo build`와 `cargo check`를 구분하지 않는 실수.**
타입·빌림 오류만 빠르게 확인하고 싶을 땐 기계어를 생성하지 않는 `cargo check`가 훨씬 빠릅니다. 편집-저장 루프에서는 `cargo check`(또는 `cargo clippy`)를, 실제 실행할 때만 `cargo build`/`cargo run`을 쓰세요.

---

## 2.13 연습문제

Rustlings 스타일로, 손으로 직접 풀어 보세요. 각 문항 아래 '정답 스케치'를 접어두고, 먼저 머리로(또는 실제 셸로) 답을 만든 뒤 펼쳐서 대조하면 학습 효과가 큽니다. 자기검증용으로 **컴파일러 에러 코드** 또는 **측정값**을 함께 적었습니다.

**문제 1 — 증분 컴파일.** criterion 의존성을 가진 프로젝트를 처음 `cargo build`하면 14초가 걸렸는데, 곧바로 다시 `cargo build`하면 0.05초로 끝났습니다. Cargo는 무엇을 근거로 "다시 컴파일할 필요가 없다"고 판단할까요? 또, 단지 `rustc` 버전만 올렸을 때(소스·`Cargo.toml`은 그대로) 두 번째 빌드는 빨라질까요, 느려질까요?

> **정답 스케치:** 근거는 `target/<profile>/.fingerprint/` 안의 fingerprint입니다. Cargo는 컴파일 단위마다 `rustc` 버전 해시, 활성 feature, 타깃, 프로파일, 소스 경로, 의존성 fingerprint, `RUSTFLAGS`, 관련 설정을 해싱해 저장하고, 다음 빌드에서 이 값들을 다시 계산해 비교합니다. 하나라도 다르면 그 단위만 재컴파일합니다. `rustc` 버전을 올리면 fingerprint JSON의 `"rustc"` 필드가 바뀌므로 **모든 크레이트가 무효화**되어 다시 14초대로 느려집니다(컴파일러가 바뀌면 산출물의 ABI 호환을 보장할 수 없으니 당연합니다). 자기검증: `cat target/debug/.fingerprint/<crate>-<hash>/*.json`으로 `"rustc"` 필드의 정수 해시를 직접 확인하세요.

**문제 2 — sparse 레지스트리.** 깨끗한 CI 러너에서 `cargo build`가 의존성 인덱스를 받는 데 예전(1.70 미만)에는 수십 초가 걸렸지만 지금은 훨씬 빠릅니다. 무엇이 바뀌었나요? 그리고 파이썬의 uv가 빠른 이유와 어떤 공통점이 있나요? 로컬에서 이 전환을 어떻게 눈으로 확인할 수 있을까요?

> **정답 스케치:** 인덱스 프로토콜이 git(전체 인덱스 저장소 클론)에서 **sparse**(필요한 크레이트의 메타데이터만 HTTPS GET)로 바뀌었습니다. 1.70부터 기본값입니다. uv도 PyPI 전체를 받지 않고 필요한 패키지 메타데이터만 HTTP로 골라 받아 캐시하는 동일한 "부분 페치 + 캐시" 철학을 씁니다. 자기검증: `ls ~/.cargo/registry/index/`를 실행해 디렉터리 이름이 옛날 `github.com-...`이 아니라 `index.crates.io-...`로 시작하면 sparse 프로토콜입니다. 그 안의 `config.json`에는 `"dl"`/`"api"` 두 키만 있습니다.

**문제 3 — `cargo install`의 함정.** 동료가 "프로젝트에 `serde`를 쓰려고 `cargo install serde`를 했는데 에러가 나요"라고 합니다. 무슨 에러가 나며, 올바른 명령은 무엇인가요? `cargo install`과 `cargo add`는 각각 파이썬의 무엇에 대응하나요?

> **정답 스케치:** `serde`는 라이브러리(바이너리 없음)라 `cargo install`이 거부합니다. 에러 메시지: `there is nothing to install in \`serde v1.0.228\`, because it has no binaries` / `cargo install is only for installing programs, and can't be used with libraries.` 올바른 명령은 `cargo add serde --features derive`입니다. 대응: `cargo install`은 전역 CLI 도구 설치이므로 파이썬의 `pipx install`에, `cargo add`는 프로젝트 의존성 추가이므로 `poetry add`/`pip install`(프로젝트 맥락)에 해당합니다.

**문제 4 — nightly 기능과 컴파일러 에러.** stable 1.96.0에서 아래 코드를 컴파일하면 어떤 에러 코드가 나오나요? 두 가지 해결책을 제시하세요.

```rust
fn main() {
    let boxed: Box<i32> = Box::new(42);
    let box value = boxed;
    println!("{}", value);
}
```

> **정답 스케치:** `error[E0658]: box pattern syntax is experimental`. box 패턴은 nightly 전용이기 때문입니다. 해결책 (1) nightly 채널 + 파일 상단에 `#![feature(box_patterns)]` 선언, (2) stable에서 그냥 역참조: `let value = *boxed;`. 프로덕션은 (2). 참고로 stable에서 `#![feature(...)]`를 적으면 별개의 에러 `error[E0554]`가 뜹니다. 자기검증: `rustc --explain E0658`로 의미를 확인하세요.

**문제 5 — criterion과 정직한 측정.** 아래 벤치에서 `black_box`를 제거하면 측정 시간이 비현실적으로 0에 가까워질 수 있습니다. 왜 그럴까요? `count_primes(100_000)`을 Release로 측정하면 약 2.2 ms, 동일 알고리즘의 파이썬은 약 99.6 ms입니다. 배수는 얼마이며, 이 수치를 "100배"라고 부르면 왜 부정직한가요?

```rust
b.iter(|| count_primes(black_box(100_000)))
```

> **정답 스케치:** `black_box` 없이 입력이 컴파일 타임 상수면, LLVM이 `count_primes(100_000)`의 결과를 **상수 폴딩**해 루프 자체를 제거할 수 있습니다. 그러면 호출이 사라져 ~0ns가 측정됩니다. `black_box`는 값을 "컴파일러가 미리 알 수 없는 것"으로 위장해 실제 계산을 강제합니다. 배수는 99.6 / 2.2 ≈ **45배**입니다. "100배"는 이 워크로드·이 입력·Release 기준의 측정값(45배)을 부풀린 것이며, 측정 맥락(입력 크기, 빌드 모드) 없이 단일 숫자를 인용하는 것이 부정직의 핵심입니다.

**문제 6 — `cargo add`/`cargo remove`의 출처.** 의존성을 추가/제거하려고 예전 블로그에서 본 `cargo install cargo-edit`를 먼저 깔아야 할까요? 각 명령이 언제 Cargo 본체에 들어왔는지 떠올려 보세요.

> **정답 스케치:** 아닙니다. `cargo add`(1.62+)와 `cargo remove`(1.66+) 모두 Cargo 본체에 흡수되어, 별도 설치 없이 바로 씁니다. 한때 `cargo-edit` 크레이트가 제공하던 기능이 표준으로 승격된 사례입니다. 자기검증: `cargo remove --help`가 "command not found" 없이 사용법을 출력하면 본체에 내장된 것입니다.

---

지금까지 본 것처럼 Rust의 도구 생태계는 파이썬에서 여러 도구로 흩어져 있던 책임을 `rustup`과 `cargo`라는 두 축으로 통합하고, 결정론적 빌드와 정적 단일 바이너리로 배포 지옥을 끝냅니다. 더 깊은 차원에서 이것은 단순한 편의가 아니라, **"안전성과 성능은 빌드 타임에 결정된다"**는 Rust 철학의 첫 번째 발현입니다. 빌드가 일급 시민이기에, 컴파일러는 정수 오버플로를 잡고 feature로 코드를 깎아내고 LLVM으로 기계어를 최적화할 기회를 얻습니다.

이제 환경이라는 무대가 정리되었으니, 다음 챕터에서는 무대 위에서 펼쳐지는 본 공연으로 들어갑니다. Chapter 3에서는 파이썬의 동적 타입과 문(Statement) 중심 구문이, Rust의 정적 타입과 표현식(Expression) 중심 구문으로 어떻게 번역되는지를 살펴봅니다. 특히 파이썬의 `int` 하나가 메모리에서 28바이트의 `PyObject`를 차지하는 반면 Rust의 `i64`가 정확히 8바이트인 이유를, 바이트 단위로 해부할 것입니다.

---

## References

- https://doc.rust-lang.org/cargo/
- https://doc.rust-lang.org/cargo/guide/
- https://doc.rust-lang.org/book/ch01-03-hello-cargo.html
- https://doc.rust-lang.org/cargo/reference/profiles.html
- https://rust-lang.github.io/rustup/
- https://crates.io/
