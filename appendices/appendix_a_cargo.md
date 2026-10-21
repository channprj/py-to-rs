# 부록 A. Cargo & 프로젝트 레퍼런스

이 부록은 본문(특히 Chapter 2)에서 다룬 개념을 빠르게 찾아보기 위한 **레퍼런스**입니다. 명령어의 "왜"는 본문에 있으니, 여기서는 "무엇을 어떻게"에 집중합니다. 표의 플래그 의미는 `cargo 1.96.0`, `rustup 1.28.2`에서 대조한 레퍼런스입니다. 그 가운데 `cargo new`의 기본 에디션만 2026-10-22에 cargo 1.99.1로 다시 확인했고, 생성된 `Cargo.toml`은 `edition = "2024"`입니다. rustup 도구 자체는 1.29.2입니다. 표의 나머지 명령을 1.99.1에서 한 줄씩 다시 실행한 것은 아닙니다. 파이썬 대응은 의미상 가장 가까운 도구를 적었으며, 1:1로 완벽히 같지는 않다는 점을 감안하세요.

---

## A.1 Cargo 명령어 전체 레퍼런스

표준 명령(별도 설치 불필요)입니다. `cargo <cmd> --help`로 언제든 전체 플래그를 확인할 수 있습니다. 파이썬에서 빌드·테스트·포맷·린트·문서·게시를 각각 다른 도구로 처리하던 것과 달리, 아래 명령들은 모두 단일 `cargo` 네임스페이스 아래에 있고 동일한 매니페스트(`Cargo.toml`)와 락 파일(`Cargo.lock`)을 공유합니다. 즉 새 도구를 익힐 때마다 설정 파일이 늘어나는 일이 없습니다.

| 명령 | 설명 | 자주 쓰는 플래그 | Python 대응 |
| :--- | :--- | :--- | :--- |
| `cargo new <name>` | 새 패키지 디렉터리를 스캐폴딩 | `--lib`(라이브러리), `--bin`(기본, 실행 파일), `--edition 2024`(생략해도 현재 stable이 이 값을 씀), `--vcs git` | `poetry new`, `django-admin startproject` |
| `cargo init` | **기존** 디렉터리를 패키지로 초기화 | `--lib`, `--bin`, `--name <n>` | `poetry init`, `pip init`(개념상) |
| `cargo build` | 컴파일만 수행, 산출물 생성 | `--release`, `--target <triple>`, `--all-features`, `-p <pkg>`, `--timings` | (없음 — 파이썬은 빌드 단계 없음) |
| `cargo run` | 빌드 후 바이너리 실행 | `--release`, `--bin <n>`, `--example <n>`, `-- <args>` | `python main.py` |
| `cargo check` | 타입·빌림 검사만, 코드 생성 생략(수 배 빠름) | `--all-targets`, `--workspace` | `mypy`, `pyright` |
| `cargo test` | 단위·통합·문서 테스트를 병렬 실행 | `--release`, `--`(테스트 바이너리 인자), `-- --nocapture`, `--test <n>`, `--doc` | `pytest` |
| `cargo bench` | 벤치마크 실행(기본은 nightly 필요, 보통 `criterion` 사용) | `--bench <n>` | `pytest-benchmark` |
| `cargo fmt` | `rustfmt`로 전체 코드 포매팅 | `--check`(변경 없이 검사만, CI용), `--all` | `black`, `isort`, `ruff format` |
| `cargo clippy` | 관용구·성능·버그 패턴까지 린팅 | `-- -D warnings`(경고를 에러로), `--fix`, `--all-targets` | `flake8`, `pylint`, `ruff check` |
| `cargo fix` | 컴파일러/clippy가 자동 수정 가능한 경고를 일괄 적용 | `--edition`(에디션 마이그레이션), `--allow-dirty` | `ruff check --fix`, `pyupgrade` |
| `cargo doc` | 주석(`///`) 기반 HTML 문서 생성 | `--open`(브라우저로 열기), `--no-deps`, `--document-private-items` | `sphinx-build`, `mkdocs build` |
| `cargo add <crate>` | 의존성 추가 + `Cargo.toml` 자동 수정 | `--features <f>`, `--dev`, `--build`, `--optional`, `--no-default-features` | `poetry add`, `pip install` |
| `cargo remove <crate>` | 의존성 제거(`cargo rm` 별칭) | `--dev`, `--build` | `poetry remove`, `pip uninstall` |
| `cargo update` | `Cargo.lock`을 SemVer 제약 내 최신으로 갱신 | `-p <crate>`(특정 크레이트만), `--dry-run`, `--breaking`(메이저 포함) | `poetry update`, `pip-compile -U` |
| `cargo tree` | 의존성 그래프를 트리로 시각화 | `-i <crate>`(역추적: 누가 끌어오나), `-d`(중복 표시), `-e features` | `pipdeptree` |
| `cargo search <q>` | crates.io에서 크레이트 검색 | `--limit <n>` | `pip search`(폐기됨) |
| `cargo info <crate>` | 크레이트 메타데이터·feature 출력 | `--version <v>` | `pip show` |
| `cargo publish` | crates.io에 게시 | `--dry-run`, `--allow-dirty`, `--token <t>` | `twine upload`, `poetry publish` |
| `cargo yank --vers <v>` | 게시된 버전을 신규 사용 못 하게 철회(기존 락은 유지) | `--undo` | (PyPI엔 직접 대응 없음) |
| `cargo install <crate>` | 실행형 크레이트를 `~/.cargo/bin`에 전역 설치 | `--locked`, `--git <url>`, `--path .`, `--force` | `pipx install` |
| `cargo uninstall <crate>` | 전역 설치 바이너리 제거 | — | `pipx uninstall` |
| `cargo clean` | `target/` 빌드 캐시 삭제 | `--release`, `-p <pkg>`, `--doc` | `rm -rf build/ dist/ *.egg-info` |
| `cargo metadata` | 해석된 의존성을 기계 판독용 JSON으로 출력 | `--format-version 1`, `--no-deps` | `pip list --format=json` |
| `cargo package` | 게시 전 배포용 `.crate` 타르볼 생성·검증 | `--list`, `--no-verify` | `python -m build` |
| `cargo vendor` | 모든 의존성 소스를 로컬에 복사(오프라인/감사용) | `--versioned-dirs` | `pip download`, `pip wheel` |
| `cargo expand` *(확장)* | 매크로(`derive`/`macro_rules!`)를 펼친 코드 출력 | `--bin <n>`, `<path>` | `python -m macropy`(개념상) |

> 💡 별칭이 기본 제공됩니다: `cargo b`=build, `cargo c`=check, `cargo r`=run, `cargo t`=test, `cargo d`=doc. `~/.cargo/config.toml`의 `[alias]` 섹션에서 직접 별칭을 추가할 수도 있습니다(예: `nt = "nextest run"`).

### 자주 쓰는 조합

```bash
# 편집-저장 루프: 변경마다 빠른 검사 (cargo-watch 필요)
$ cargo watch -x check -x test

# CI 게이트 3종 (black+ruff+mypy+pytest 전체를 대체)
$ cargo fmt --check && cargo clippy -- -D warnings && cargo test

# 특정 테스트 하나만, 출력 캡처 끄고 실행
$ cargo test test_parse_user -- --nocapture

# 실행 파일에 인자 전달 ('--' 뒤는 프로그램 인자)
$ cargo run --release -- --input data.json --verbose

# 누가 이 크레이트를 끌어오는지 역추적
$ cargo tree -i serde
```

---

## A.2 `Cargo.toml` 필드 레퍼런스

매니페스트의 주요 섹션을 한눈에 봅니다. 전체 스키마는 [The Cargo Book](https://doc.rust-lang.org/cargo/reference/manifest.html)에 있습니다.

### `[package]`

```toml
[package]
name = "my_app"              # 크레이트 이름 (필수)
version = "0.1.0"            # SemVer (필수)
edition = "2024"            # 언어 에디션. cargo new 기본값(cargo 1.99.1). 이 책 예제는 "2021"
rust-version = "1.74"       # MSRV: 이 버전 미만이면 빌드 거부
authors = ["Heechan <h@e.io>"]
description = "..."         # crates.io 게시 시 필수
license = "MIT OR Apache-2.0"  # SPDX 식별자
repository = "https://github.com/org/repo"
readme = "README.md"
keywords = ["cli", "json"]  # crates.io 검색 (최대 5개)
categories = ["command-line-utilities"]
default-run = "main_bin"    # bin이 여러 개일 때 cargo run 기본 대상
build = "build.rs"          # 빌드 스크립트 경로 (기본 자동 탐지)
exclude = ["tests/fixtures/*"]  # 패키징에서 제외할 파일
```

| 필드 | 의미 | Python `pyproject.toml` 대응 |
| :--- | :--- | :--- |
| `name` / `version` | 크레이트 식별자 | `project.name` / `project.version` |
| `edition` | 문법·관용구 세대 | (대응 없음 — 언어 차원) |
| `rust-version` | 최소 컴파일러 버전 | `project.requires-python` |
| `license` | SPDX 라이선스 | `project.license` |
| `description`/`keywords`/`categories` | 레지스트리 메타데이터 | `project.description`/`keywords`/`classifiers` |

### `[dependencies]` — 버전 지정 문법

```toml
[dependencies]
serde = "1.0"                                            # 캐럿: >=1.0.0, <2.0.0
serde = { version = "1.0", features = ["derive"] }       # feature 활성화
rand = { version = "0.8", default-features = false }      # 기본 feature 끄기
tokio = { version = "1", features = ["full"], optional = true }  # feature로만 켜지는 선택적 의존성

# 출처 지정
my_lib    = { path = "../my_lib" }                        # 로컬 경로 (pip install -e)
gh_crate  = { git = "https://github.com/o/r" }            # git, 기본 브랜치
gh_branch = { git = "https://github.com/o/r", branch = "dev" }
gh_tag    = { git = "https://github.com/o/r", tag = "v1.2.0" }
gh_rev    = { git = "https://github.com/o/r", rev = "a1b2c3d" }   # 커밋 해시 고정
priv_reg  = { version = "2.0", registry = "my-registry" }  # 사내 레지스트리

# 같은 크레이트를 다른 이름으로 (버전 충돌 회피)
old_rand = { package = "rand", version = "0.7" }

# 워크스페이스 상속
serde = { workspace = true }
```

### `[dev-dependencies]` / `[build-dependencies]`

```toml
[dev-dependencies]              # 테스트·예제·벤치에만. 최종 바이너리에 미포함
criterion = "0.5"
proptest = "1"

[build-dependencies]            # build.rs 컴파일에만 쓰임
cc = "1.0"                      # C 코드 컴파일
tonic-build = "0.12"            # gRPC/protobuf 코드 생성
```

| 섹션 | 빌드에 포함되는 대상 | Python 대응 |
| :--- | :--- | :--- |
| `[dependencies]` | 라이브러리·바이너리 본체 | `[project.dependencies]` |
| `[dev-dependencies]` | 테스트·예제·벤치만 | `[dependency-groups.dev]`, poetry dev group |
| `[build-dependencies]` | `build.rs`만 | `[build-system.requires]`(개념상) |

### `[features]`

```toml
[features]
default = ["std", "json"]         # 아무것도 안 적으면 켜지는 기본 feature
std = []                          # 마커 feature (의존성 없음)
json = ["dep:serde_json"]         # 이 feature가 serde_json 의존성을 활성화
full = ["std", "json", "async"]   # feature가 다른 feature를 켤 수 있음
async = ["dep:tokio", "tokio/rt"] # 의존성의 특정 feature까지 전파
```

- `dep:<crate>`: 동명의 암묵적 feature를 만들지 않고 선택적 의존성만 켭니다(권장 표기).
- `<crate>/<feature>`: 의존성의 특정 feature를 전파합니다.
- feature는 **컴파일 타임 플래그**입니다. 런타임 토글이 아닙니다(Chapter 2.12 참조).

### `[profile.*]`

```toml
[profile.release]               # cargo build --release
opt-level = 3
lto = "fat"

[profile.dev]                   # cargo build (기본)
opt-level = 0

[profile.dev.package."*"]       # 의존성만 최적화, 내 코드는 디버그
opt-level = 3

[profile.bench]                 # cargo bench
[profile.test]                  # cargo test
```

상속 구조: `dev`→디버그 기본, `release`→배포 기본, `test`는 `dev`를, `bench`는 `release`를 상속합니다. 임의 이름의 커스텀 프로파일은 `inherits`로 베이스를 지정합니다(`[profile.dist] inherits = "release"`).

### `[workspace]` / `[[bin]]` / `[lib]` / `[[bench]]`

```toml
# 워크스페이스 루트 Cargo.toml
[workspace]
resolver = "2"                  # feature 해석 알고리즘 v2 (edition 2021+ 권장)
members = ["apps/*", "libs/*"]
exclude = ["legacy/old_crate"]

[workspace.dependencies]        # 멤버가 { workspace = true }로 상속
serde = { version = "1.0", features = ["derive"] }

[workspace.package]             # 멤버가 version.workspace = true 등으로 상속
version = "0.1.0"
edition = "2021"             # 이 책의 예제 에디션. 새 프로젝트의 cargo new는 "2024"
```

```toml
# 한 패키지가 여러 산출물을 가질 때
[lib]
name = "my_lib"
crate-type = ["rlib", "cdylib"]  # cdylib: C/Python에서 부를 동적 라이브러리 (PyO3)

[[bin]]                          # 이중 대괄호 = 배열 (여러 개 가능)
name = "server"
path = "src/bin/server.rs"

[[bin]]
name = "cli"
path = "src/bin/cli.rs"

[[bench]]
name = "throughput"
harness = false                  # criterion 사용 시 기본 테스트 하네스 끔
```

> 💡 `[[...]]`(이중 대괄호)는 TOML의 배열 테이블로, 같은 종류를 여러 개 정의할 때 씁니다(`[[bin]]`, `[[bench]]`, `[[test]]`, `[[example]]`). `[lib]`은 패키지당 하나뿐이라 단일 대괄호입니다.

---

## A.3 의존성 버전 지정(SemVer) 규칙

Cargo의 핵심 직관은 **"기본 표기가 곧 캐럿(`^`)"**이라는 점입니다. 파이썬 `poetry`에서는 `^1.0`처럼 캐럿을 명시해야 했지만, Cargo는 버전 문자열을 그냥 적기만 해도 캐럿 의미가 적용됩니다. 즉 `serde = "1"`은 "SemVer 메이저 1 안에서 가장 높은 호환 버전"을 뜻하며, 실제로 빌드에 고정되는 정확한 버전은 `Cargo.lock`이 따로 기록합니다(Chapter 2.5). 캐럿은 "메이저가 같으면 호환된다"는 SemVer 약속을 신뢰하는 표기이고, 틸드(`~`)는 그보다 보수적으로 패치만 허용합니다.

| 표기 | 명칭 | 허용 범위 | 비고 |
| :--- | :--- | :--- | :--- |
| `"1.2.3"` | 캐럿(기본) | `>=1.2.3, <2.0.0` | 가장 흔함. 메이저 안에서 자동 업데이트 |
| `"1.2"` | 캐럿 | `>=1.2.0, <2.0.0` | |
| `"1"` | 캐럿 | `>=1.0.0, <2.0.0` | |
| `"^1.2.3"` | 캐럿(명시) | `>=1.2.3, <2.0.0` | 위와 동일, 의도 명시 |
| `"~1.2.3"` | 틸드 | `>=1.2.3, <1.3.0` | 패치만 올림 |
| `"~1.2"` | 틸드 | `>=1.2.0, <1.3.0` | |
| `"~1"` | 틸드 | `>=1.0.0, <2.0.0` | |
| `"1.*"` | 와일드카드 | `>=1.0.0, <2.0.0` | crates.io 게시 시 금지 |
| `"*"` | 와일드카드 | 모든 버전 | 위험. 게시 금지 |
| `"=1.2.3"` | 정확히 고정 | `1.2.3`만 | |
| `">=1.2, <1.5"` | 비교 연산 | 명시 범위 | 쉼표=AND |

### `0.x` 특수 규칙 🚨

SemVer에서 `0.x`는 "불안정"을 뜻하므로, Cargo는 **첫 0이 아닌 자리**를 메이저처럼 취급합니다.

| 표기 | 허용 범위 | 설명 |
| :--- | :--- | :--- |
| `"0.7.3"` | `>=0.7.3, <0.8.0` | `0.7`이 메이저 역할 → `0.8`로 안 올라감 |
| `"0.0.5"` | `>=0.0.5, <0.0.6` | `0.0.x`는 패치 하나하나가 메이저 |

따라서 `tokio = "0.7"`은 `0.8`로 자동 업데이트되지 **않습니다.** `0.x` 의존성을 올릴 때는 마이너 번호에 깨지는 변경이 숨어 있으니 변경 로그를 확인하세요.

---

## A.4 프로파일 튜닝 치트시트

`[profile.release]`에 넣는 배포 최적화 옵션입니다. 거의 모든 옵션이 **빌드 시간을 희생해 런타임 속도 또는 바이너리 크기를 얻는** 트레이드오프이므로, 평소 개발 빌드(`dev` 프로파일)에는 적용하지 않고 배포 빌드에만 켜는 것이 보통입니다. 파이썬에는 "빌드 최적화"라는 개념 자체가 없었지만, AOT 컴파일 언어인 Rust에서는 이 한 블록이 최종 산출물의 성능과 크기를 좌우합니다.

| 옵션 | 값 | 효과 | 기본값(release) |
| :--- | :--- | :--- | :--- |
| `opt-level` | `0`~`3` | 최적화 수준(속도 우선) | `3` |
| `opt-level` | `"s"` / `"z"` | 크기 우선 / 크기 극단 우선 | — |
| `lto` | `false`/`"thin"`/`"fat"`(=`true`) | 링크 타임 최적화: 크레이트 경계 인라이닝 | `false` |
| `codegen-units` | `1`~`256` | 낮을수록 최적화↑·빌드↓ | `16` |
| `panic` | `"unwind"` / `"abort"` | abort는 되감기 테이블 제거 → 바이너리↓ | `"unwind"` |
| `strip` | `false`/`true`/`"debuginfo"`/`"symbols"` | 심볼 제거 → 바이너리↓ | `false` |
| `debug` | `0`~`2`/`true`/`false` | 디버그 심볼 포함 정도 | `0` |
| `overflow-checks` | `true`/`false` | 정수 오버플로 패닉 검사 | `false` |
| `incremental` | `true`/`false` | 증분 컴파일(빌드 캐시) | `false` |

### 두 가지 대표 프로파일

```toml
# 1) 최대 속도 (벤치마크·프로덕션 핫패스)
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1

# 2) 최소 크기 (컨테이너·임베디드 배포)
[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

> 💡 의존성은 강하게, 내 코드는 빠르게 빌드하고 싶다면 개발 프로파일에서 의존성만 최적화하세요: `[profile.dev.package."*"] opt-level = 3`. 이미지 처리·압축 같은 무거운 라이브러리를 디버그로 돌릴 때 체감 속도가 크게 오릅니다.

---

## A.5 유용한 Cargo 확장

`cargo install`로 설치하는 서드파티 서브커맨드입니다. 설치하면 PATH의 `cargo-<name>` 바이너리를 Cargo가 자동 인식해 `cargo <name>` 형태로 호출할 수 있습니다. 파이썬에서 `pytest-xdist`, `coverage`, `pip-audit`을 따로 깔아 쓰던 것과 비슷하지만, 모두 동일한 `cargo` 진입점으로 통합된다는 점이 다릅니다. 이 확장들은 표준은 아니지만 사실상 실무 표준에 가깝습니다.

| 확장 | 설치 | 역할 | Python 대응 |
| :--- | :--- | :--- | :--- |
| **cargo-edit** | `cargo install cargo-edit` | `cargo upgrade`(의존성 일괄 상향) 등 추가. `add`/`rm`은 본체 흡수됨 | `pip-tools` |
| **cargo-watch** | `cargo install cargo-watch` | 파일 변경 감지 시 자동 재실행 | `pytest-watch`, `watchdog` |
| **cargo-nextest** | `cargo install cargo-nextest` | 차세대 테스트 러너(병렬·빠른 출력) | `pytest-xdist` |
| **cargo-audit** | `cargo install cargo-audit` | `Cargo.lock`의 알려진 취약점 스캔 | `pip-audit`, `safety` |
| **cargo-outdated** | `cargo install cargo-outdated` | 구버전 의존성 목록 표시 | `pip list --outdated` |
| **cargo-flamegraph** | `cargo install flamegraph` | 프로파일링 플레임그래프 생성 | `py-spy`, `pyinstrument` |
| **cargo-expand** | `cargo install cargo-expand` | 매크로 펼친 코드 출력(매크로 디버깅) | — |
| **cargo-bloat** | `cargo install cargo-bloat` | 어느 의존성/함수가 바이너리를 부풀리나 | — |
| **cargo-deny** | `cargo install cargo-deny` | 라이선스·중복·취약점 정책 강제 | `pip-licenses` |
| **cargo-llvm-cov** | `cargo install cargo-llvm-cov` | 코드 커버리지 측정 | `coverage.py`, `pytest-cov` |

```bash
# 흔한 워크플로
$ cargo watch -x check -x test     # 저장할 때마다 check + test
$ cargo nextest run                 # 빠른 테스트 러너
$ cargo audit                       # 취약점 스캔 (CI 필수)
$ cargo outdated --workspace        # 구버전 의존성 점검
$ cargo flamegraph --bin server     # 프로파일링 (--release 권장)
$ cargo expand --bin cli            # #[derive(...)] 펼쳐보기
```

> 💡 `cargo-expand`와 `cargo-flamegraph`(플레임그래프 생성)는 nightly 툴체인이 필요할 수 있습니다. `cargo +nightly expand`처럼 툴체인을 명시하세요.

---

## A.6 rustup 명령어

툴체인·채널·타깃·컴포넌트를 관리하는 `rustup` 명령입니다(Chapter 2.3 참조). `pyenv`가 인터프리터 버전만 다루던 것과 달리, `rustup`은 (1) 컴파일러 버전·채널, (2) 크로스 컴파일 타깃, (3) `clippy`·`rustfmt` 같은 부속 컴포넌트를 한 도구로 함께 관리합니다. 명령은 대부분 `rustup <명사> <동사>` 구조(예: `toolchain install`, `target add`, `component add`)라 익히기 쉽습니다.

| 명령 | 역할 | pyenv 대응 |
| :--- | :--- | :--- |
| `rustup update` | 모든 툴체인 + rustup 자체를 최신으로 | (pyenv엔 일괄 갱신 없음) |
| `rustup show` | 활성/설치된 툴체인·타깃 표시 | `pyenv versions` |
| `rustup check` | 갱신 가능 여부만 확인 | — |
| `rustup default <toolchain>` | 시스템 기본 툴체인 설정 | `pyenv global` |
| `rustup toolchain install <ch>` | 채널/버전 설치(`stable`, `nightly`, `1.99.1`) | `pyenv install` |
| `rustup toolchain list` | 설치된 툴체인 목록 | `pyenv versions` |
| `rustup toolchain uninstall <ch>` | 툴체인 제거 | `pyenv uninstall` |
| `rustup override set <ch>` | 현재 디렉터리에 툴체인 고정 | `pyenv local` |
| `rustup override unset` | 디렉터리 고정 해제 | — |
| `rustup target add <triple>` | 크로스 컴파일 타깃 추가 | (대응 없음) |
| `rustup target list` | 사용 가능 타깃 목록(`--installed`로 필터) | — |
| `rustup component add <comp>` | 컴포넌트 추가(`clippy`, `rustfmt`, `rust-src`, `rust-analyzer`) | — |
| `rustup component list` | 컴포넌트 목록 | — |
| `rustup run <ch> <cmd>` | 특정 툴체인 환경에서 명령 실행 | — |
| `rustup which <bin>` | 실행될 바이너리 경로 표시 | `pyenv which` |
| `rustup doc --book` | 로컬 The Rust Book 열기 | — |

```bash
# 자주 쓰는 시퀀스
$ rustup default stable
$ rustup toolchain install nightly
$ rustup component add clippy rustfmt rust-src rust-analyzer
$ rustup target add x86_64-unknown-linux-musl   # 정적 리눅스 바이너리용
$ cargo +nightly build                           # 일회성 nightly 사용 (+toolchain 문법)
```

> 💡 `+toolchain` 문법(`cargo +nightly ...`, `rustc +1.99.1 ...`)으로 한 명령만 다른 툴체인으로 실행할 수 있습니다. 디렉터리 단위 고정은 `rust-toolchain.toml`(Chapter 2.3)이 더 견고합니다.

---

## A.7 자주 쓰는 환경변수

빌드·런타임·로깅 동작을 제어하는 환경변수입니다.

### 런타임/디버깅

| 변수 | 값 | 효과 |
| :--- | :--- | :--- |
| `RUST_BACKTRACE` | `0`/`1`/`full` | 패닉 시 백트레이스 출력(`full`은 모든 프레임) |
| `RUST_LOG` | `error`/`warn`/`info`/`debug`/`trace`, `crate=level` | `log`/`tracing`/`env_logger` 로그 레벨. 예: `RUST_LOG=myapp=debug,tokio=warn` |
| `RUST_LOG_STYLE` | `auto`/`always`/`never` | 로그 색상 출력 제어 |

### Cargo / 빌드

| 변수 | 효과 |
| :--- | :--- |
| `CARGO_HOME` | Cargo 데이터 디렉터리(기본 `~/.cargo`) — 레지스트리 캐시·전역 바이너리 위치 |
| `RUSTUP_HOME` | rustup 데이터 디렉터리(기본 `~/.rustup`) |
| `CARGO_TARGET_DIR` | `target/` 위치 변경(여러 프로젝트가 캐시 공유 가능) |
| `RUSTFLAGS` | `rustc`에 전달할 전역 플래그(예: `RUSTFLAGS="-C target-cpu=native"`) |
| `CARGO_BUILD_JOBS` | 병렬 빌드 작업 수(기본 CPU 코어 수) |
| `CARGO_NET_OFFLINE` | `true`면 네트워크 접근 없이 캐시만으로 빌드 |
| `CARGO_INCREMENTAL` | `0`/`1` — 증분 컴파일 강제 on/off |
| `CARGO_TERM_COLOR` | `auto`/`always`/`never` — 출력 색상 |
| `CARGO_REGISTRIES_<NAME>_TOKEN` | 사내 레지스트리 인증 토큰 |

### 컴파일 타임 주입(코드에서 `env!`로 접근)

`build.rs`가 `println!("cargo:rustc-env=KEY=val")`로 주입한 값이나 Cargo가 자동 설정하는 값은 `env!`/`option_env!` 매크로로 **컴파일 타임에** 박을 수 있습니다(런타임 비용 0).

| 변수 | 내용 |
| :--- | :--- |
| `CARGO_PKG_VERSION` | `Cargo.toml`의 패키지 버전 |
| `CARGO_PKG_NAME` | 패키지 이름 |
| `CARGO_MANIFEST_DIR` | `Cargo.toml`이 있는 디렉터리 절대 경로 |
| `OUT_DIR` | `build.rs` 산출물 디렉터리(생성 코드 출력 위치) |

```rust
fn main() {
    // 컴파일 타임에 값이 박힌다 — 런타임 조회 비용 없음
    println!("v{} ({})", env!("CARGO_PKG_VERSION"), env!("GIT_HASH"));
}
```

```bash
# 런타임 사용 예
$ RUST_BACKTRACE=1 RUST_LOG=debug cargo run --release
$ RUSTFLAGS="-C target-cpu=native" cargo build --release   # 현재 CPU에 맞춰 최적화
$ CARGO_TARGET_DIR=/tmp/shared-target cargo build           # target/ 공유로 빌드 캐시 재사용
```

---

## A.8 프로젝트 품질 게이트 레시피

명령어를 아는 것과 팀이 매번 같은 품질 기준으로 검증하는 것은 다릅니다. Python 프로젝트에서 `ruff format && ruff check && pytest`를 Makefile이나 CI에 묶어두듯, Rust 프로젝트도 Cargo 명령을 하나의 게이트로 묶어두는 편이 좋습니다.

### 로컬 사전 점검

가장 작은 기본 게이트입니다.

```bash
$ cargo fmt --all -- --check
$ cargo clippy --all-targets --all-features -- -D warnings
$ cargo test --all-features
```

각 명령의 의미는 다음과 같습니다.

| 명령 | 잡는 문제 | Python 대응 |
|---|---|---|
| `cargo fmt --check` | 포맷 불일치 | `black --check`, `ruff format --check` |
| `cargo clippy -D warnings` | 관용구 위반, 잠재 버그 | `ruff check`, `pylint` |
| `cargo test --all-features` | unit/integration/doctest 실패 | `pytest` + `doctest` |

`--all-targets`는 라이브러리뿐 아니라 예제, 테스트, 벤치 타깃까지 clippy가 보게 합니다. `--all-features`는 optional feature 조합에서만 깨지는 코드를 잡는 데 유용하지만, feature 간 충돌을 의도한 크레이트라면 별도 feature matrix를 구성해야 합니다.

### 문서와 공개 API 확인

라이브러리라면 문서도 빌드 대상입니다.

```bash
$ RUSTDOCFLAGS="-D warnings" cargo doc --no-deps --all-features
$ cargo test --doc
```

Rust 문서 주석의 예제는 `cargo test`에 포함되지만, 문서 생성 자체의 경고(깨진 intra-doc link 등)는 `cargo doc`에서 더 명확히 드러납니다. 공개 API 문서에 잘못된 intra-doc link를 넣으면 `RUSTDOCFLAGS="-D warnings"`가 CI를 빨갛게 만들어줍니다. Python 패키지에서 Sphinx 링크 깨짐을 별도 검사하는 것과 같은 위치입니다.

### 벤치마크와 회귀 감시

성능을 약속하는 크레이트라면 `criterion` 벤치마크를 별도 게이트로 둡니다. 다만 CI 공유 머신의 노이즈가 크므로, pull request마다 절대 수치를 실패 조건으로 삼기보다 추세를 기록하는 편이 안전합니다.

```bash
$ cargo bench --bench aggregate
$ cargo test --release --all-features
```

`cargo test --release`는 드물게 중요합니다. 디버그 빌드에서는 정수 오버플로가 패닉하지만 릴리스 빌드에서는 wrapping 될 수 있고(Chapter 1), 최적화가 켜져야만 드러나는 `unsafe` UB 징후도 있습니다. 모든 테스트를 매번 릴리스로 돌릴 필요는 없지만, 성능·FFI·SIMD·unsafe가 있는 프로젝트는 릴리스 테스트를 주기적으로 포함하세요.

### CI용 최소 워크플로

```yaml
name: rust

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all -- --check
      - run: cargo clippy --all-targets --all-features -- -D warnings
      - run: cargo test --all-features
      - run: RUSTDOCFLAGS="-D warnings" cargo doc --no-deps --all-features
```

이 워크플로는 작지만 강합니다. 포맷, 린트, 테스트, 문서를 모두 같은 툴체인 위에서 검증합니다. PyO3 프로젝트라면 여기에 `maturin build --release`와 wheel import 테스트를 추가하고, 웹 서비스라면 Docker 이미지 빌드와 smoke test를 붙이면 됩니다.

### MSRV와 최신 stable을 분리하라

라이브러리가 `rust-version = "1.80"` 같은 MSRV(Minimum Supported Rust Version)를 선언한다면, CI는 최소 두 축을 가져야 합니다.

```yaml
strategy:
  matrix:
    toolchain: [stable, "1.80.0"]
```

최신 stable은 앞으로 깨질 문제를, MSRV는 "우리가 약속한 최소 컴파일러에서 아직 빌드되는가"를 검증합니다. Python 패키지가 `requires-python = ">=3.12"`를 선언했다면 3.12와 최신 3.x를 모두 테스트하는 것과 같은 원리입니다.

---

이상이 Cargo·rustup의 일상 작업에 필요한 명령과 매니페스트 필드의 핵심 레퍼런스입니다. 개념적 배경과 "왜"는 Chapter 2를, 각 명령의 전체 옵션은 `cargo <cmd> --help`와 [The Cargo Book](https://doc.rust-lang.org/cargo/)을 참조하세요.

---

## References

- https://doc.rust-lang.org/cargo/
- https://doc.rust-lang.org/cargo/commands/
- https://doc.rust-lang.org/cargo/reference/manifest.html
- https://doc.rust-lang.org/cargo/reference/profiles.html
- https://rust-lang.github.io/rustup/
- https://doc.rust-lang.org/rustc/
