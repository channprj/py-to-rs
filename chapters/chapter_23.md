# Chapter 23. 테스트와 문서화: pytest를 넘어서

![Part 9. 프로덕션으로 가는 길](../images/part-09-production.jpg)


> **Part 9. 프로덕션으로 가는 길 (From Language to Production)**
>
> 여기까지 우리는 소유권·트레이트·에러·동시성·매크로·FFI까지 Rust라는 *언어*를 정복했습니다. 하지만 언어를 안다는 것과 그 언어로 프로덕션 코드를 운영한다는 것은 다른 이야기입니다. 실무에서 코드는 "동작하는가"만큼이나 "동작함을 어떻게 증명하는가", "회귀를 어떻게 막는가", "다른 사람이 어떻게 읽는가"가 중요합니다. Part 9는 바로 이 마지막 한 걸음 — 검증된 코드를 신뢰할 수 있는 소프트웨어로 만드는 엔지니어링 실천 — 을 다룹니다. 그 첫 장은 모든 신뢰의 토대인 **테스트와 문서화**입니다.

10년 차 Python 엔지니어에게 테스트는 익숙한 풍경입니다. `tests/` 디렉터리, `conftest.py`에 모인 픽스처들, `@pytest.mark.parametrize`로 펼친 케이스 표, `unittest.mock`으로 봉합한 외부 의존성, 그리고 CI의 마지막 단계에서 돌아가는 `pytest --cov`. 이 도구들은 강력하지만, 우리는 그 강력함의 대가를 매일 조용히 치르고 있습니다.

가장 큰 대가는 **테스트가 별도 세계에 산다**는 점입니다. `src/`와 `tests/`는 물리적으로 분리되어 있고, 둘을 잇는 것은 `import` 경로와 우리의 규율뿐입니다. 내부 함수를 테스트하려면 `_private_helper`를 어쩔 수 없이 노출하거나, `mock.patch("module._private_helper")`로 문자열 경로를 박아 넣습니다. 그 문자열은 리팩터링 한 번에 조용히 깨지고, 테스트는 "통과하지만 아무것도 검증하지 않는" 좀비가 됩니다. 두 번째 대가는 **문서가 거짓말을 한다**는 점입니다. docstring에 적어둔 `>>> foo(2)` 예제는 실제 `foo`가 바뀌어도 빨갛게 변하지 않습니다. `doctest` 모듈이 있긴 하지만 기본값이 아니라 별도로 켜야 하고, 대부분의 팀은 켜지 않습니다. 그 결과 README의 예제 코드는 6개월만 지나면 박물관 유물이 됩니다.

세 번째 대가는 더 미묘합니다. **예제 기반 테스트(example-based testing)의 맹점**입니다. 우리는 `test_add(2, 3) == 5`를 씁니다. 그런데 `add(-1, sys.maxsize)`는요? `add(0, 0)`은요? 시니어 엔지니어조차 자신이 *생각해낸* 케이스만 테스트합니다. 버그는 언제나 우리가 생각하지 못한 입력 — 빈 리스트, 음수, 오버플로 경계, 유니코드 결합 문자 — 에 숨어 있습니다. Hypothesis를 써본 분이라면 "내가 못 본 반례를 기계가 찾아주는" 그 충격을 기억할 겁니다. 하지만 Hypothesis 역시 Python 생태계에서는 "아는 사람만 쓰는" 선택지에 머물러 있습니다.

Rust는 이 세 가지 대가를 언어와 도구 체인 차원에서 정면으로 다룹니다. 테스트는 `tests/` 디렉터리로 격리되는 게 아니라 **구현과 같은 파일 안에 살면서 private 함수에 자유롭게 접근**합니다(`#[cfg(test)]`). 문서의 코드 예제는 장식이 아니라 **`cargo test`가 실제로 컴파일하고 실행하는 테스트**입니다(doctest) — 문서가 거짓말을 하면 빌드가 빨갛게 터집니다. 그리고 속성 기반 테스트(`proptest`)는 외부 크레이트이긴 하지만, Hypothesis와 정확히 같은 철학을 Rust의 타입 시스템 위에서 더 자연스럽게 구현합니다. Chapter 2에서 "`cargo test`는 테스트·예제·문서를 한 줄로 돌린다"고 예고했던 그 약속을, 이번 장에서 끝까지 풀어냅니다.

이 책의 다른 장과 마찬가지로, 우리는 "어떻게 쓰는가"보다 "왜 그렇게 동작하는가"에 무게를 둡니다. `cargo test`를 치면 컴파일러가 정확히 몇 개의 바이너리를 만드는지, doctest는 어떻게 보이지 않는 `fn main()`으로 감싸지는지, `proptest`가 반례를 어떻게 최소값까지 축소(shrink)하는지를 내부까지 들여다봅니다. 그리고 그 도구들을 앞 장들에서 키워온 `Money`/`Cents` 뉴타입(Chapter 7-8)과 결제 상태 기계(Chapter 10)에 직접 부착해, "테스트가 곧 설계의 일부"가 되는 경험을 만들어 봅니다.

---

## 23.1 The Python Way — `pytest`, 픽스처, 그리고 그 비용

먼저 우리가 익숙한 방식을 정확히 짚겠습니다. 음수가 될 수 없는 금액 타입을 Python으로 만들고 `pytest`로 검증하는 전형적인 코드입니다.

```python
# src/money.py
from __future__ import annotations

def cents_new(value: int) -> int:
    if value < 0:
        raise ValueError(f"금액은 음수일 수 없습니다: {value}")
    return value
```

```python
# tests/test_money.py
import pytest
from src.money import cents_new

class TestCents:
    def test_rejects_negative(self) -> None:
        with pytest.raises(ValueError, match="음수"):
            cents_new(-1)

    @pytest.mark.parametrize("value,expected", [(0, 0), (1, 1), (999_999, 999_999)])
    def test_accepts_nonnegative(self, value: int, expected: int) -> None:
        assert cents_new(value) == expected

@pytest.fixture
def sample_payment() -> dict[str, object]:
    return {"state": "pending", "amount": 50_000}

def test_uses_fixture(sample_payment: dict[str, object]) -> None:
    assert sample_payment["amount"] == 50_000
```

이 코드는 깔끔하고, `pytest`는 훌륭한 도구입니다. `parametrize`로 케이스를 표로 펼치고, 픽스처로 셋업을 재사용하고, `pytest.raises`로 예외를 검증합니다. 하지만 이 익숙함 속에 네 가지 구조적 비용이 숨어 있습니다.

**비용 1 — 분리와 import 경로의 취약함.** `src/`와 `tests/`는 별개의 세계입니다. 둘을 잇는 `from src.money import cents_new`는 프로젝트 루트, `PYTHONPATH`, `conftest.py`의 위치, `pytest.ini`/`pyproject.toml`의 설정에 따라 동작하기도 하고 안 하기도 합니다. "테스트가 import를 못 한다"로 30분을 태운 경험은 누구에게나 있습니다. 더 근본적으로, **private 함수를 테스트하기 어렵습니다.** `_internal_helper`를 검증하려면 그것을 사실상 공개하거나, `module._internal_helper`라는 문자열 경로로 접근해야 합니다.

**비용 2 — mock의 문자열 마법과 그 깨짐.** 외부 의존성을 끊으려면 `mock.patch("src.service.fetch_from_db")`처럼 **문자열로 경로를 지정**합니다. 이 문자열은 타입 체커도, IDE의 리네임 리팩터링도 추적하지 못합니다. `fetch_from_db`를 다른 모듈로 옮기는 순간, mock은 엉뚱한 곳을 패치하거나 조용히 실패하고, 테스트는 "초록불이지만 아무것도 막지 못하는" 상태가 됩니다.

**비용 3 — doctest는 켜야만 돈다.** docstring에 `>>> cents_new(150)` 예제를 적어도, `pytest`는 기본적으로 그것을 실행하지 않습니다. `pytest --doctest-modules`를 명시하거나 설정 파일에 추가해야 하며, 대부분의 팀은 잊습니다. 결과적으로 문서의 예제는 검증되지 않은 채 썩어갑니다.

**비용 4 — 커버리지의 파편화와 착시.** `pytest-cov`(내부적으로 `coverage.py`)는 또 다른 서드파티이고, 라인 커버리지 95%가 "검증됨"을 의미하지 않습니다. 실행은 됐지만 `assert`가 하나도 없는 라인도 커버리지에는 잡힙니다.

이 비용들의 공통 뿌리는 Chapter 2에서 본 그 주제 — **언어의 책임 범위가 인터프리터에서 끝나기 때문**입니다. 테스트 러너(`pytest`/`unittest`/`nose`), 픽스처 시스템, mock 라이브러리, 커버리지, doctest, 파라미터화가 모두 서로 다른 도구이거나 옵션이고, 그것들을 하나의 일관된 파이프라인으로 엮는 일은 온전히 우리 몫입니다. Rust는 이 영역을 언어와 `cargo`가 직접 가져갑니다.

---

## 23.2 The Rust Mental Model — 테스트는 코드와 함께 산다

Rust의 출발점은 한 문장으로 요약됩니다. **테스트와 문서는 라이브러리의 외부 부속이 아니라, 같은 크레이트 안에서 컴파일러가 직접 챙기는 1급 시민이다.**

이 철학은 세 갈래로 나타납니다.

첫째, **단위 테스트는 구현과 같은 파일에 산다.** 모듈 안에 `#[cfg(test)] mod tests`를 두고, 그 안의 함수에 `#[test]`를 붙입니다. `#[cfg(test)]`는 "이 모듈은 `cargo test`로 빌드할 때만 컴파일하라"는 조건부 컴파일 지시자입니다. 즉 테스트 코드는 릴리스 바이너리에 **단 한 바이트도 들어가지 않습니다.** 그러면서도 같은 파일·같은 모듈 안에 있으니 `super::*`로 private 함수·private 필드에 자유롭게 접근합니다. Python에서 "private을 테스트하려면 공개해야 하는" 딜레마가 애초에 없습니다.

둘째, **문서의 코드는 실행되는 테스트다.** `///` 문서 주석 안의 코드 블록은 `cargo test`가 별도의 작은 프로그램으로 컴파일·실행합니다(doctest). 문서가 `Cents::new(150)`을 보여주는데 실제 시그니처가 바뀌었다면, 테스트가 컴파일 에러로 터집니다. Python `doctest`의 강화판이되, **기본으로 켜져 있고** Rust 타입 시스템의 검사를 그대로 받습니다.

셋째, **검증의 모든 단계가 한 명령어로 통합된다.** `cargo test` 한 줄이 단위 테스트, 통합 테스트, doctest를 각각 컴파일해 병렬로 실행합니다. `pytest` + `pytest-cov` + `--doctest-modules` + `conftest.py` 조합이 `cargo test` 하나로 수렴합니다.

이 세 가지를 Python과 한 표로 정리하면 다음과 같습니다.

| 개념 | Python (`pytest` 생태계) | Rust (`cargo` 내장 + 크레이트) |
|------|--------------------------|-------------------------------|
| 단위 테스트 위치 | 별도 `tests/` 디렉터리 | 구현과 **같은 파일** `#[cfg(test)] mod tests` |
| private 접근 | 어렵다(노출/문자열 경로) | **자유롭다**(`use super::*`) |
| 테스트 함수 표시 | 함수명 `test_` 접두사 규약 | `#[test]` 속성 |
| 동등 단언 | `assert a == b` | `assert_eq!(a, b)` / `assert_ne!` |
| 예외/패닉 검증 | `with pytest.raises(...)` | `#[should_panic(expected = "...")]` |
| 파라미터화 | `@pytest.mark.parametrize` | `rstest`의 `#[case(...)]` |
| 픽스처 | `@pytest.fixture` + 인자 주입 | `rstest`의 `#[fixture]` + 인자 주입 |
| 문서 예제 검증 | `--doctest-modules`(별도 옵션) | **doctest 기본 내장** |
| 속성 기반 테스트 | `hypothesis` | `proptest` / `quickcheck` |
| 스냅샷 테스트 | `syrupy` | `insta` |
| 커버리지 | `pytest-cov`(서드파티) | `cargo llvm-cov`(서드파티) |
| 테스트 러너 | `pytest` / `unittest` | `cargo test` / `cargo nextest` |
| 테스트 코드의 릴리스 포함 | 디렉터리 분리로 회피 | `#[cfg(test)]`로 **컴파일 단계에서 배제** |

표의 마지막 줄이 결정적입니다. 테스트는 코드 *옆에* 있지만 릴리스 빌드에는 *없습니다.* 이 둘을 동시에 만족시키는 것이 `#[cfg(test)]`입니다. 이제 각 항목을 내부까지 파고듭니다.

---

## 23.3 내장 단위 테스트 — `#[cfg(test)]`와 `#[test]`

Chapter 8에서 만든 `Cents` 뉴타입을 가져와, 음수를 거부하는 검증 생성자에 단위 테스트를 붙여 봅시다. 다음은 라이브러리 크레이트 `src/lib.rs`의 일부입니다.

```rust
use std::fmt;

/// 음수가 될 수 없는 금액. 최소 화폐 단위(센트/원)의 정수로 보관합니다.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Cents(i64);

impl Cents {
    pub fn new(value: i64) -> Result<Self, MoneyError> {
        if value < 0 {
            Err(MoneyError::Negative(value))
        } else {
            Ok(Cents(value))
        }
    }

    pub fn value(self) -> i64 {
        self.0
    }

    /// 오버플로를 검사하며 두 금액을 더합니다.
    pub fn checked_add(self, rhs: Cents) -> Option<Cents> {
        self.0.checked_add(rhs.0).map(Cents)
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum MoneyError {
    Negative(i64),
}

impl fmt::Display for MoneyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MoneyError::Negative(v) => write!(f, "금액은 음수일 수 없습니다: {v}"),
        }
    }
}
impl std::error::Error for MoneyError {}

// ── 여기서부터 테스트 ──────────────────────────────────────
#[cfg(test)]            // cargo test 빌드에서만 컴파일된다
mod tests {
    use super::*;        // 부모 모듈의 모든 것(private 포함)을 가져온다

    #[test]
    fn new_rejects_negative() {
        assert_eq!(Cents::new(-5), Err(MoneyError::Negative(-5)));
    }

    #[test]
    fn new_accepts_zero_and_positive() {
        assert_eq!(Cents::new(0).unwrap().value(), 0);
        assert_eq!(Cents::new(42).unwrap().value(), 42);
    }

    #[test]
    fn checked_add_sums() {
        let a = Cents::new(150).unwrap();
        let b = Cents::new(250).unwrap();
        assert_eq!(a.checked_add(b), Some(Cents::new(400).unwrap()));
    }

    #[test]
    fn checked_add_detects_overflow() {
        let max = Cents::new(i64::MAX).unwrap();
        let one = Cents::new(1).unwrap();
        assert_eq!(max.checked_add(one), None); // 조용히 감기지 않는다
    }
}
```

`cargo test`를 돌리면 다음과 같이 나옵니다(실제 출력).

```text
running 4 tests
test tests::checked_add_sums ... ok
test tests::checked_add_detects_overflow ... ok
test tests::new_accepts_zero_and_positive ... ok
test tests::new_rejects_negative ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

세 가지를 짚어 봅시다.

**`use super::*`가 핵심입니다.** `tests` 모듈은 부모 모듈의 자식이므로, `super`로 부모를 가리키고 그 안의 모든 항목 — `pub`이 아닌 private 항목까지 — 을 끌어옵니다. `Cents`의 내부 필드 `.0`은 모듈 바깥에선 보이지 않지만, 같은 크레이트의 자식 모듈인 `tests`에서는 접근 가능합니다. Python에서 `_private`를 테스트하려고 씨름하던 문제가 사라집니다.

**`assert_eq!`는 실패 시 양쪽 값을 출력합니다.** `assert!(a == b)`만 쓰면 "false였다"만 알려주지만, `assert_eq!(a, b)`는 실패 시 `left`와 `right`의 `Debug` 표현을 둘 다 보여줍니다. 그래서 비교 대상 타입에는 보통 `#[derive(Debug, PartialEq)]`가 필요합니다(위 `Cents`, `MoneyError`에 붙어 있습니다). `assert_ne!`는 그 반대입니다.

**왜 테스트가 구현과 같은 파일에 사는가.** Rust 커뮤니티의 관례는 "단위 테스트는 구현 바로 아래, 통합 테스트는 `tests/`"입니다. 단위 테스트가 같은 파일에 있으면 private 세부 구현을 테스트할 수 있고, 코드를 읽는 사람이 "이 함수가 어떤 경계 조건을 보장하는지"를 바로 옆에서 봅니다. 테스트가 곧 실행 가능한 명세서가 됩니다.

> 💡 패닉을 검증하고 싶을 때는 `#[should_panic]`을 씁니다. Python의 `pytest.raises`에 대응합니다.
>
> ```rust
> #[test]
> #[should_panic(expected = "금액은 음수일 수 없습니다")]
> fn expect_with_message_panics() {
>     // expect의 메시지가 패닉 텍스트에 포함된다.
>     Cents::new(-1).expect("금액은 음수일 수 없습니다: -1");
> }
> ```
>
> `expected = "..."`는 패닉 메시지에 그 부분 문자열이 *포함*되는지 검사합니다(`pytest.raises(..., match=...)`와 동일한 발상). 다만 `#[should_panic]`은 "어딘가에서 패닉했다"만 보장하지 절대 권장되는 검증 방식은 아닙니다 — 검증 실패를 패닉이 아니라 `Result`로 표현하는 게 Rust답고(Chapter 12), 그래서 위 테스트보다는 23.3 초반의 `assert_eq!(Cents::new(-5), Err(...))` 쪽이 정석입니다.

---

## 23.4 통합 테스트 — `tests/` 디렉터리와 공개 API 경계

단위 테스트가 "안에서 들여다보는" 시선이라면, 통합 테스트는 "밖에서 사용자처럼 쓰는" 시선입니다. Rust는 이 둘을 디렉터리로 명확히 가릅니다. 크레이트 루트의 `tests/` 디렉터리에 둔 각 `.rs` 파일은 **독립된 별개의 크레이트로 컴파일되어**, 우리 라이브러리를 외부 의존성처럼 `use`로 가져옵니다.

Chapter 10의 결제 상태 기계를 라이브러리의 공개 API로 노출했다고 합시다. `tests/integration.rs`는 이렇게 생겼습니다.

```rust
// tests/integration.rs — 크레이트 외부에서 공개 API만 사용한다.
use ch23::{Cents, PaymentStatus, TransitionError};

#[test]
fn full_lifecycle_through_public_api() {
    let payment = PaymentStatus::Pending { amount: Cents::new(50_000).unwrap() };
    let authorized = payment.authorize("AUTH-XYZ".into()).unwrap();
    let captured = authorized.capture("RCPT-001".into()).unwrap();
    let refunded = captured.refund(Cents::new(20_000).unwrap()).unwrap();
    assert_eq!(refunded.label(), "Refunded");
}

#[test]
fn refund_exceeding_capture_is_rejected() {
    let captured = PaymentStatus::Pending { amount: Cents::new(10_000).unwrap() }
        .authorize("A".into())
        .unwrap()
        .capture("R".into())
        .unwrap();
    assert_eq!(
        captured.refund(Cents::new(99_999).unwrap()),
        Err(TransitionError::RefundExceedsCapture)
    );
}
```

여기서 결정적인 차이가 드러납니다. **통합 테스트는 `use ch23::...`로 우리 크레이트를 외부 사용자처럼 import합니다.** 따라서 `pub`으로 노출하지 않은 항목은 보이지 않습니다. `Cents`의 private 필드 `.0`에는 손댈 수 없고, 오직 `Cents::new`/`value`/`checked_add`라는 공개 계약만 씁니다. 이 제약이 통합 테스트의 가치입니다 — **공개 API가 실제로 충분하고 일관된지**를 사용자 관점에서 검증합니다. private 필드를 만지작거리던 단위 테스트가 놓치는 "API 설계 결함"을 여기서 잡습니다.

> **💡 Under the Hood — 단위 vs 통합의 경계는 "컴파일 단위"의 경계다**
>
> 둘의 차이는 철학이 아니라 *물리적 컴파일 단위*입니다. 단위 테스트(`#[cfg(test)] mod tests`)는 라이브러리 크레이트 *안에서* 함께 컴파일되므로 private에 접근합니다. 통합 테스트(`tests/*.rs`)는 라이브러리를 빌드한 *뒤*, 그 산출물을 링크하는 *별개의 크레이트*로 컴파일됩니다. 그래서 통합 테스트는 라이브러리의 공개 ABI 경계 밖에 서 있고, `pub`만 봅니다.
>
> 실무 함의가 두 가지입니다. 첫째, `tests/` 안의 파일이 여러 개면 각각이 별도 크레이트라 **별도 바이너리로 컴파일**됩니다 — 파일이 많으면 그만큼 컴파일·링크가 늘어납니다. 둘째, 통합 테스트끼리 헬퍼를 공유하려면 `tests/common/mod.rs`처럼 서브모듈로 둬야 합니다(`tests/common.rs`로 두면 그 자체가 테스트 크레이트로 잡혀 "테스트 0개"라는 빈 결과를 냅니다). 이 사소해 보이는 규칙이 Rust 입문자가 가장 자주 밟는 디렉터리 함정입니다.

판단 기준은 단순합니다. **내부 알고리즘의 경계 조건·private 헬퍼는 단위 테스트로, "사용자가 이 라이브러리를 import해서 쓰는 시나리오"는 통합 테스트로.** 바이너리 크레이트(`src/main.rs`)는 보통 로직을 라이브러리(`src/lib.rs`)로 빼고 얇은 `main`만 남겨, 그 라이브러리를 단위·통합 양쪽에서 테스트하는 구조가 정석입니다.

---

## 23.5 Under the Hood — `cargo test`는 무엇을 컴파일하는가

`cargo test` 한 줄을 치면 무슨 일이 벌어질까요? 출력의 머리글을 자세히 보면 답이 보입니다. 앞서 만든 라이브러리에 통합 테스트와 doctest까지 더하고 `cargo test`를 돌리면, 출력이 이렇게 *구획*으로 나뉩니다(실제 출력을 요약).

```text
     Running unittests src/lib.rs (target/debug/deps/ch23-9af3043721de81df)
test result: ok. 8 passed; 0 failed; ...

     Running tests/integration.rs (target/debug/deps/integration-b1f7c80db00ab99d)
test result: ok. 2 passed; 0 failed; ...

   Doc-tests ch23
test result: ok. 6 passed; 0 failed; ...
```

세 개의 독립된 `Running`/`Doc-tests` 블록이 보이시나요? 이것이 핵심입니다.

> **💡 Under the Hood — 세 종류의 테스트, 세 개(이상)의 바이너리**
>
> `cargo test`는 단일 거대 테스트 실행기가 아니라, **여러 개의 테스트 바이너리를 따로 컴파일해 차례로 실행하는 오케스트레이터**입니다.
>
> 1. **단위 테스트 바이너리.** 라이브러리(`src/lib.rs`)를 `#[cfg(test)]` 코드까지 포함해 컴파일한 **하나의** 실행 파일(`target/debug/deps/ch23-<해시>`). 내부의 `#[test]` 함수들을 수집해 돌립니다.
> 2. **통합 테스트 바이너리들.** `tests/` 아래의 각 `.rs` 파일마다 **별개의** 실행 파일. 파일이 N개면 바이너리도 N개입니다(23.4의 컴파일 단위 경계).
> 3. **doctest 실행기.** 각 `///` 코드 블록을 *임시* 작은 프로그램으로 추출·컴파일해 실행합니다. 최신 툴체인은 이들을 하나로 묶어 컴파일하지만(merged doctests), 의미상 각 예제는 독립 프로그램입니다.
>
> 그리고 각 바이너리 안에서 `#[test]` 함수들은 **기본적으로 여러 스레드에 나뉘어 병렬 실행**됩니다. `pytest`가 기본 단일 프로세스이고 병렬화하려면 `pytest-xdist`가 필요한 것과 대조적입니다. 이 병렬성 때문에 테스트는 서로 독립적이어야 하며, 전역 상태(환경 변수, 같은 파일 경로, 공유 DB)를 만지는 테스트는 서로를 오염시킬 수 있습니다.

이 구조를 알면 자주 쓰는 플래그들이 자연스럽게 이해됩니다.

```bash
# 출력 캡처 끄기: 통과한 테스트의 println! 도 화면에 보여준다
$ cargo test -- --nocapture

# 단일 스레드로 직렬 실행: 전역 상태를 건드리는 테스트 디버깅에 유용
$ cargo test -- --test-threads=1

# 이름으로 필터링: 'refund'가 들어간 테스트만
$ cargo test refund

# 특정 통합 테스트 파일만
$ cargo test --test integration

# 라이브러리 단위 테스트만 (doctest·통합 제외)
$ cargo test --lib
```

`--` 뒤의 인자는 `cargo`가 아니라 **테스트 바이너리 자체**에 전달된다는 점에 주의하세요(`--nocapture`, `--test-threads`는 테스트 러너의 옵션입니다). 기본적으로 테스트 실행기는 통과한 테스트의 표준 출력을 *삼킵니다* — 실패한 테스트의 출력만 보여줘서 신호 대 잡음비를 높이려는 설계입니다. `--nocapture`는 그 입막음을 풉니다.

### `#[ignore]` — 평소엔 건너뛰는 무거운 테스트

DB·네트워크가 필요하거나 수 초가 걸리는 테스트는 매번 돌리고 싶지 않습니다. `#[ignore]`를 붙이면 기본 실행에서 빠지고, `--ignored`로 명시할 때만 돕니다.

```rust
#[test]
fn fast() { assert_eq!(2 + 2, 4); }

#[test]
#[ignore = "느린 통합 테스트: cargo test -- --ignored 로만 실행"]
fn slow_integration() { /* ... 무거운 셋업 ... */ }
```

기본 실행과 `--ignored` 실행의 실제 출력입니다.

```text
$ cargo test --test ignore_demo
running 2 tests
test slow_integration ... ignored, 느린 통합 테스트: cargo test -- --ignored 로만 실행
test fast ... ok
test result: ok. 1 passed; 0 failed; 1 ignored; ...

$ cargo test --test ignore_demo -- --ignored
running 1 test
test slow_integration ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 1 filtered out; ...
```

`pytest.mark.skip`/`skipif`에 대응하되, 이유 문자열이 출력에 그대로 찍혀 "왜 건너뛰는지"가 로그에 남습니다.

---

## 23.6 문서화 테스트(doctest) — 거짓말하지 못하는 문서

이제 이 장의 킬러 기능입니다. Rust에서 `///` 문서 주석 안의 코드 블록은 **장식이 아니라 실행되는 테스트**입니다. `cargo test`가 이들을 추출해 컴파일하고 실행합니다.

`Cents`에 문서와 예제를 달아 봅시다.

```rust
/// 음수가 될 수 없는 금액. 최소 화폐 단위(센트/원)의 정수로 보관합니다.
///
/// 생성 시점에 검증을 통과해야 하므로, `Cents` 값을 손에 쥐었다는 사실 자체가
/// "유효한(0 이상) 금액"이라는 증명입니다.
///
/// # Examples
///
/// ```
/// use ch23::Cents;
///
/// let price = Cents::new(1_500).unwrap();
/// assert_eq!(price.value(), 1_500);
/// ```
///
/// 두 금액을 더하면 합계가 나옵니다. 오버플로는 `checked_add`로 막습니다.
///
/// ```
/// use ch23::Cents;
///
/// let total = Cents::new(150).unwrap().checked_add(Cents::new(250).unwrap());
/// assert_eq!(total.map(|c| c.value()), Some(400));
/// ```
///
/// 음수는 만들 수 없습니다 — 검증은 예외가 아니라 `Result`로 표현됩니다.
///
/// ```
/// use ch23::Cents;
///
/// assert!(Cents::new(-1).is_err());
/// ```
pub struct Cents(i64);
```

`cargo test`의 doctest 구획 출력입니다(실제 출력).

```text
   Doc-tests ch23

running 3 tests
test src/lib.rs - Cents (line 12) ... ok
test src/lib.rs - Cents (line 21) ... ok
test src/lib.rs - Cents (line 30) ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

세 예제가 모두 실제로 컴파일·실행되어 통과했습니다. 만약 `value()`를 `amount()`로 이름을 바꾸고 문서를 안 고치면, 이 doctest가 `error[E0599]: no method named 'value'`로 빌드를 빨갛게 터뜨립니다. **문서가 거짓말을 하는 순간 CI가 막습니다.** Python `doctest`와 달리 기본으로 켜져 있고, Rust의 전체 타입 검사를 받습니다.

> **💡 Under the Hood — doctest는 보이지 않는 `fn main()`으로 감싸진다**
>
> doctest 코드 블록은 그대로 컴파일되는 게 아닙니다. `rustdoc`이 각 블록을 **임시 크레이트로 감싸서** 컴파일합니다. 명시적 `fn main()`이 없으면 블록 전체를 자동으로 `fn main() { ... }`로 둘러싸고, `extern crate <크레이트명>;`을 앞에 끼워 넣습니다. 그래서 위 예제들이 `use ch23::Cents;`만 쓰고도 독립 프로그램으로 돕니다.
>
> 이 "자동 `main` 래핑" 덕분에 doctest에서 `?` 연산자(Chapter 12-13)를 쓰려면 약간의 장치가 필요합니다. `?`는 `Result`를 반환하는 함수 안에서만 쓸 수 있으니, 블록의 반환 타입을 알려줘야 합니다. 관례는 `#`으로 시작하는 *숨김 줄*에 `Ok(())`를 넣는 것입니다. `#`로 시작하는 줄은 **컴파일·실행은 되지만 문서에는 표시되지 않습니다.**
>
> ```rust
> /// ```
> /// use std::num::ParseIntError;
> /// let n: i32 = "42".parse()?;   // ?를 쓰려면 main이 Result여야 한다
> /// assert_eq!(n, 42);
> /// # Ok::<(), ParseIntError>(())  // 이 줄은 문서에 안 보이지만 컴파일된다
> /// ```
> ```
>
> 독자는 깔끔한 본문만 보고, 컴파일러는 `Ok(())`로 마무리되는 완전한 프로그램을 봅니다. 거짓말 없는 문서와 읽기 좋은 문서를 동시에 얻는 트릭입니다.

코드 블록 뒤에 **속성**을 붙여 doctest의 동작을 조절합니다. Python에는 대응물이 없는, doctest를 실무에 쓸 수 있게 해주는 핵심 장치입니다.

- ` ```no_run`: **컴파일은 하되 실행은 하지 않는다.** 네트워크 요청, 무한 루프, 파일 쓰기처럼 실행하면 곤란하지만 "이 코드가 컴파일되는가(API가 맞는가)"는 검증하고 싶을 때.
- ` ```ignore`: **컴파일도 실행도 하지 않는다.** 의사 코드나 의도적으로 불완전한 스니펫. 가장 약한 보장이라 남용 금지.
- ` ```compile_fail`: **컴파일 실패해야 통과.** "이 코드는 타입 에러로 막힌다"를 문서이자 테스트로 박제할 때(이 책의 컴파일 에러 시연을 자동 검증하는 데 쓰입니다).
- ` ```should_panic`: 실행 시 패닉해야 통과.

```rust
/// no_run: 컴파일은 하되 실행은 하지 않는다.
///
/// ```no_run
/// // 실제로 실행하면 영원히 멈추겠지만, 컴파일 검사는 받는다.
/// loop {}
/// ```
pub fn doc_norun() {}
```

이 블록의 doctest 출력에는 `... - compile`이라는 꼬리표가 붙어, "컴파일만 했다"가 명시됩니다.

```text
test src/doctest_check.rs - doctest_check::doc_norun (line 14) - compile ... ok
```

문서화의 또 다른 축은 모듈·크레이트 수준 문서입니다. `///`는 *바로 뒤따르는 항목*을 문서화하지만, `//!`는 *자신을 감싼 항목*(파일 맨 위에 쓰면 크레이트/모듈 전체)을 문서화합니다. `cargo doc --open`을 치면 이 주석들이 그대로 HTML 문서로 렌더링됩니다 — Sphinx를 따로 설정할 필요 없이, 코드 옆 주석이 곧 검증된 API 문서가 됩니다.

---

## 23.7 속성 기반 테스트(proptest) — 반례를 기계가 찾아 축소한다

예제 기반 테스트의 한계로 돌아갑시다. 우리는 `checked_add_sums`에서 `150 + 250 == 400`을 검증했습니다. 그런데 `0 + 0`은? `i64::MAX + 1`은? `999_999 + 1`은? 시니어 엔지니어조차 자신이 *떠올린* 케이스만 씁니다. **속성 기반 테스트(Property-based Testing)**는 발상을 뒤집습니다 — 구체적 입력 대신 **불변식(invariant)**을 선언하면, 프레임워크가 수백 개의 무작위 입력을 생성해 그 불변식을 깨려 시도합니다. Python의 `hypothesis`를 써봤다면 정확히 그 도구의 Rust 판이 `proptest`입니다.

`Cargo.toml`에 dev-dependency로 추가합니다(`cargo add proptest --dev`).

```toml
[dev-dependencies]
proptest = "1.12"
```

`tests/properties.rs`에 `Cents`의 불변식 네 가지를 선언합니다.

```rust
use ch23::Cents;
use proptest::prelude::*;

proptest! {
    // 불변식 1: 음수 아닌 어떤 정수로 만든 Cents든, value()는 원래 값과 같다(왕복).
    #[test]
    fn value_roundtrips(n in 0i64..=i64::MAX) {
        let c = Cents::new(n).unwrap();
        prop_assert_eq!(c.value(), n);
    }

    // 불변식 2: 덧셈은 교환법칙을 만족한다.
    #[test]
    fn add_is_commutative(a in 0i64..1_000_000, b in 0i64..1_000_000) {
        let ca = Cents::new(a).unwrap();
        let cb = Cents::new(b).unwrap();
        prop_assert_eq!(ca.checked_add(cb), cb.checked_add(ca));
    }

    // 불변식 3: 합이 i64::MAX를 넘으면 None, 아니면 정확히 a+b. (조용한 오버플로 없음)
    #[test]
    fn add_never_silently_overflows(a in 0i64..=i64::MAX, b in 0i64..=i64::MAX) {
        let result = Cents::new(a).unwrap().checked_add(Cents::new(b).unwrap());
        match a.checked_add(b) {
            Some(sum) => prop_assert_eq!(result.map(|c| c.value()), Some(sum)),
            None => prop_assert_eq!(result, None),
        }
    }

    // 불변식 4: 음수 입력은 항상 Err.
    #[test]
    fn negative_always_errors(n in i64::MIN..0) {
        prop_assert!(Cents::new(n).is_err());
    }
}
```

`n in 0i64..=i64::MAX`가 **전략(Strategy)**입니다 — "이 범위에서 값을 뽑아라"라는 생성 규칙입니다. `proptest!` 매크로 안의 `#[test]`는 각 불변식마다 기본 256개의 무작위 입력을 생성해 실행합니다. `prop_assert!`/`prop_assert_eq!`는 일반 `assert`와 달리, 실패 시 패닉 대신 축소 절차를 시작합니다. 실제로 네 불변식 모두 통과합니다.

```text
running 4 tests
test add_never_silently_overflows ... ok
test add_is_commutative ... ok
test negative_always_errors ... ok
test value_roundtrips ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

여기서 진짜 마법은 **반례 축소(Shrinking)**입니다. 일부러 틀린 불변식 — "모든 `Cents` 값은 1000 미만이다" — 을 선언해 봅시다.

```rust
proptest! {
    #[test]
    fn all_cents_under_1000_FALSE(n in 0i64..1_000_000) {
        let c = Cents::new(n).unwrap();
        prop_assert!(c.value() < 1000);
    }
}
```

`proptest`는 무작위로 큰 값(가령 `743_812`)을 만들어 반례를 발견한 뒤, 그 값을 **불변식이 여전히 깨지는 가장 작은 값까지 축소**해 보고합니다. 실제 출력입니다.

```text
thread 'all_cents_under_1000_FALSE' panicked at tests/shrink_demo.rs:7:1:
Test failed: assertion failed: c.value() < 1000 at tests/shrink_demo.rs:11.
minimal failing input: n = 1000
	successes: 0
	local rejects: 0
	global rejects: 0
```

`minimal failing input: n = 1000`. 처음 발견한 거대한 무작위 값이 아니라, 불변식을 깨는 **경계값 1000**까지 깎아낸 것입니다. 이 한 줄은 2026-10-22에 proptest 1.12.0, rustc 1.99.1에서 다시 확인했습니다. 위의 파일 줄 번호는 이 책 스니펫 기준입니다. 디버깅 관점에서 이것은 엄청난 가치입니다 — "743812에서 실패"는 막막하지만 "1000에서 실패"는 즉시 "아, 경계가 999가 아니라 1000이었구나"를 알려줍니다.

> **💡 Under the Hood — Shrinking은 어떻게 동작하는가**
>
> `proptest`(그리고 `hypothesis`)의 축소는 단순하지만 강력한 알고리즘입니다. 반례를 하나 찾으면, 프레임워크는 그 값을 "더 단순한" 후보들로 줄여 봅니다 — 정수면 0 쪽으로 이분 탐색하듯 줄이고, 벡터면 원소를 빼보고, 문자열이면 짧게 잘라봅니다. 줄인 값이 *여전히 불변식을 깨면* 그쪽으로 더 파고들고, 깨지 않으면 한 단계 되돌립니다. 이 과정을 더 줄일 수 없을 때까지 반복해, "불변식을 깨는 최소(국소적으로) 입력"에 수렴합니다.
>
> 그리고 `proptest`는 발견한 실패 케이스를 `proptest-regressions/` 디렉터리에 **시드로 저장**합니다. 다음 실행 때 그 반례를 *먼저* 재현하므로, 한 번 발견된 버그는 무작위성에 의존하지 않고 회귀 테스트로 영구히 고정됩니다. "무작위 테스트는 재현이 안 돼서 못 믿겠다"는 오해를 정면으로 해소하는 설계입니다.

`hypothesis`와의 가장 큰 인식 차이는 **타입과의 결합도**입니다. `proptest`의 전략은 Rust 타입에 자연스럽게 붙습니다. `any::<i64>()`, `any::<String>()`, `prop::collection::vec(any::<u8>(), 0..100)`처럼 타입에서 전략을 끌어내고, `#[derive(Arbitrary)]`(별도 `proptest-derive` 크레이트)로 커스텀 구조체에 대한 생성기를 자동 도출할 수도 있습니다. 동적 타입 위에서 런타임에 데이터를 빚어내는 `hypothesis`와 달리, `proptest`는 컴파일 타임 타입 정보를 그대로 활용합니다.

---

## 23.8 실전 Hands-on — 결제 상태 기계에 테스트 3종을 부착하다

이제 모든 도구를 묶어, Chapter 10의 결제 상태 기계에 **단위 테스트 + doctest + 속성 테스트 + 파라미터화 + 스냅샷**을 전부 부착합니다. 같은 도메인 객체를 다섯 가지 렌즈로 검증하는 경험입니다.

먼저 상태 기계를 라이브러리로 노출합니다(Chapter 10에서 `Money`를 `Cents`로 단일화한 버전).

```rust
/// 결제 상태 기계 (Chapter 10에서 이어짐).
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum PaymentStatus {
    Pending { amount: Cents },
    Authorized { amount: Cents, auth_code: String },
    Captured { amount: Cents, auth_code: String, receipt_id: String },
    Refunded { original: Cents, refunded: Cents, receipt_id: String },
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TransitionError {
    InvalidTransition { from: &'static str, action: &'static str },
    RefundExceedsCapture,
}

impl PaymentStatus {
    pub fn label(&self) -> &'static str {
        match self {
            PaymentStatus::Pending { .. } => "Pending",
            PaymentStatus::Authorized { .. } => "Authorized",
            PaymentStatus::Captured { .. } => "Captured",
            PaymentStatus::Refunded { .. } => "Refunded",
        }
    }

    pub fn authorize(self, auth_code: String) -> Result<Self, TransitionError> {
        match self {
            PaymentStatus::Pending { amount } => {
                Ok(PaymentStatus::Authorized { amount, auth_code })
            }
            other => Err(TransitionError::InvalidTransition {
                from: other.label(),
                action: "authorize",
            }),
        }
    }

    pub fn capture(self, receipt_id: String) -> Result<Self, TransitionError> {
        match self {
            PaymentStatus::Authorized { amount, auth_code } => {
                Ok(PaymentStatus::Captured { amount, auth_code, receipt_id })
            }
            other => Err(TransitionError::InvalidTransition {
                from: other.label(),
                action: "capture",
            }),
        }
    }

    pub fn refund(self, refund_amount: Cents) -> Result<Self, TransitionError> {
        match self {
            PaymentStatus::Captured { amount, receipt_id, .. }
                if refund_amount.value() <= amount.value() =>
            {
                Ok(PaymentStatus::Refunded {
                    original: amount,
                    refunded: refund_amount,
                    receipt_id,
                })
            }
            PaymentStatus::Captured { .. } => Err(TransitionError::RefundExceedsCapture),
            other => Err(TransitionError::InvalidTransition {
                from: other.label(),
                action: "refund",
            }),
        }
    }
}
```

### 렌즈 1 — 단위 테스트로 행복 경로와 잘못된 전이를

`#[cfg(test)] mod tests` 안에서, 정상 경로와 잘못된 전이를 검증합니다. `?`를 쓰는 테스트도 함께 봅시다 — 테스트 함수가 `Result`를 반환하면 중간 단계의 `Result`를 `?`로 깔끔히 전파할 수 있습니다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn happy_path_transitions() {
        let p = PaymentStatus::Pending { amount: Cents::new(50_000).unwrap() };
        let a = p.authorize("AUTH-1".into()).unwrap();
        let c = a.capture("RCPT-1".into()).unwrap();
        assert_eq!(c.label(), "Captured");
    }

    #[test]
    fn invalid_transition_is_value_not_panic() {
        let p = PaymentStatus::Pending { amount: Cents::new(100).unwrap() };
        // Pending에서 capture는 잘못된 전이 — 패닉이 아니라 Err 값으로
        let err = p.capture("RCPT".into()).unwrap_err();
        assert_eq!(
            err,
            TransitionError::InvalidTransition { from: "Pending", action: "capture" }
        );
    }

    // 테스트가 Result를 반환하면 ?로 중간 단계를 전파할 수 있다.
    #[test]
    fn refund_within_capture() -> Result<(), TransitionError> {
        let captured = PaymentStatus::Pending { amount: Cents::new(10_000).unwrap() }
            .authorize("A".into())?
            .capture("R".into())?;
        let refunded = captured.refund(Cents::new(3_000).unwrap())?;
        assert_eq!(refunded.label(), "Refunded");
        Ok(())
    }
}
```

`refund_within_capture`가 `Result<(), TransitionError>`를 반환하는 데 주목하세요. `?`로 전이 실패를 전파하다가, `Err`가 나오면 테스트가 자동으로 실패 처리됩니다. `unwrap()`을 줄줄이 다는 것보다 의도가 명확하고, 실패 지점의 에러 타입이 그대로 출력됩니다. 이것이 23.10에서 다룰 "테스트에서 `unwrap` 남발 대신 `?`" 패턴의 출발점입니다.

### 렌즈 2 — 파라미터화 테스트(rstest)로 케이스 표를

`@pytest.mark.parametrize`에 대응하는 `rstest`로, 여러 입력을 한 표로 펼칩니다(`cargo add rstest --dev`).

```rust
// tests/parametrized.rs
use ch23::{Cents, PaymentStatus, TransitionError};
use rstest::rstest;

#[rstest]
#[case(0, true)]
#[case(1, true)]
#[case(999_999, true)]
#[case(-1, false)]
#[case(i64::MIN, false)]
fn cents_new_accepts_nonnegative(#[case] input: i64, #[case] ok: bool) {
    assert_eq!(Cents::new(input).is_ok(), ok);
}

// Refunded 상태에서 더 이상의 전이는 모두 잘못된 전이여야 한다.
#[rstest]
#[case("authorize")]
#[case("capture")]
fn refunded_state_rejects_further_transitions(#[case] action: &str) {
    let refunded = PaymentStatus::Refunded {
        original: Cents::new(100).unwrap(),
        refunded: Cents::new(50).unwrap(),
        receipt_id: "R".into(),
    };
    let result = match action {
        "authorize" => refunded.authorize("X".into()),
        "capture" => refunded.capture("Y".into()),
        _ => unreachable!(),
    };
    assert!(matches!(
        result,
        Err(TransitionError::InvalidTransition { from: "Refunded", .. })
    ));
}
```

각 `#[case(...)]`가 별개의 테스트로 펼쳐집니다 — 출력에 `case_1`, `case_2`...로 따로 잡혀서, 어느 케이스가 깨졌는지 즉시 보입니다.

```text
running 7 tests
test cents_new_accepts_nonnegative::case_1 ... ok
test cents_new_accepts_nonnegative::case_2 ... ok
test cents_new_accepts_nonnegative::case_3 ... ok
test cents_new_accepts_nonnegative::case_4 ... ok
test cents_new_accepts_nonnegative::case_5 ... ok
test refunded_state_rejects_further_transitions::case_1 ... ok
test refunded_state_rejects_further_transitions::case_2 ... ok

test result: ok. 7 passed; 0 failed; ...
```

`rstest`는 `#[fixture]`로 픽스처도 제공합니다 — 함수에 `#[fixture] fn pool() -> DbPool { ... }`를 정의하고 테스트 인자로 `pool: DbPool`을 받으면 자동 주입됩니다. `pytest`의 픽스처 의존성 주입과 같은 발상을, 매크로와 타입으로 구현한 것입니다.

### 렌즈 3 — 스냅샷 테스트(insta)로 구조 전체를 박제

복잡한 구조체의 출력 전체를 손으로 `assert_eq!`하기는 고통스럽습니다. `insta`(Python `syrupy`에 대응)는 출력을 파일에 **스냅샷**으로 저장하고, 이후엔 그 스냅샷과 비교합니다(`cargo add insta --dev`).

```rust
// tests/snapshot.rs
use ch23::{Cents, PaymentStatus};

#[test]
fn refunded_debug_snapshot() {
    let refunded = PaymentStatus::Pending { amount: Cents::new(50_000).unwrap() }
        .authorize("AUTH-XYZ".into()).unwrap()
        .capture("RCPT-001".into()).unwrap()
        .refund(Cents::new(20_000).unwrap()).unwrap();

    // 기대값을 손으로 쓰지 않는다. 구조 전체를 스냅샷으로 박제.
    insta::assert_debug_snapshot!(refunded);
}
```

첫 실행 때 `insta`는 `.snap.new` 파일을 만들고 테스트를 *실패*시킵니다 — "새 스냅샷을 확인하고 승인하라"는 뜻입니다. `cargo insta review`(또는 `INSTA_UPDATE=always`)로 승인하면 `tests/snapshots/`에 다음이 저장됩니다(실제 생성된 파일).

```text
---
source: tests/snapshot.rs
expression: refunded
---
Refunded {
    original: Cents(
        50000,
    ),
    refunded: Cents(
        20000,
    ),
    receipt_id: "RCPT-001",
}
```

이후로는 `refund` 로직이 의도치 않게 바뀌면 — 가령 `original`에 환불액을 잘못 넣으면 — 스냅샷 diff가 빨갛게 뜹니다. 출력 구조가 의도대로 바뀐 거라면 `cargo insta review`로 새 스냅샷을 승인하면 그만입니다. "기대 출력을 일일이 타이핑하는" 노동을 없애면서도 회귀를 잡습니다.

### 다섯 렌즈를 한 명령으로

이 모든 테스트 — 단위 8개, 통합 2개, 파라미터화 7개, 속성 4개, 스냅샷 1개, doctest 6개 — 를 `cargo test` 한 줄이 컴파일하고 실행합니다. 실제 측정한 최종 집계입니다.

```text
unittests src/lib.rs:    8 passed
tests/integration.rs:    2 passed
tests/parametrized.rs:   7 passed
tests/properties.rs:     4 passed
tests/snapshot.rs:       1 passed
Doc-tests ch23:          6 passed
```

총 28개. `pytest` + `pytest-xdist` + `hypothesis` + `syrupy` + `--doctest-modules`를 따로 설정해 엮었어야 할 일을, 하나의 도구 체인이 통합합니다.

---

## 23.9 Best Practices & 안티패턴

실무 체크리스트입니다.

**이렇게 하라**

- **단위 테스트는 구현 파일 안 `#[cfg(test)] mod tests`에, 통합 테스트는 `tests/`에.** 전자는 private 경계 조건을, 후자는 공개 API 시나리오를 검증합니다.
- **`assert_eq!`/`assert_ne!`를 기본으로.** 실패 시 양쪽 값을 출력하므로 디버깅이 빠릅니다. 비교 타입엔 `#[derive(Debug, PartialEq)]`를 답니다.
- **공개 API에는 doctest를 달아라.** 문서이자 회귀 테스트입니다. `cargo doc`으로 그대로 HTML이 되고, 거짓말하면 빌드가 막습니다.
- **불변식이 있는 코드(파서·직렬화·수치·상태 기계)에는 `proptest`를.** "왕복(roundtrip)", "교환법칙", "절대 패닉하지 않음" 같은 속성을 선언하면 경계 버그를 기계가 찾습니다.
- **느린 테스트는 `#[ignore]`로 분리**하고 CI의 별도 잡(`-- --ignored`)에서 돌립니다.
- **CI에 `cargo test`(doctest 포함)와 커버리지(`cargo llvm-cov`)를 넣어라.** `-D warnings`로 경고를 에러 승격하면 doctest 누락도 막을 수 있습니다.
- **큰 구조 출력은 `insta` 스냅샷으로.** 기대값을 손으로 쓰지 말고 리뷰로 승인하세요.

**이건 피하라(안티패턴)**

- 통합 테스트에서 private 세부를 검증하려 애쓰기 — 그건 단위 테스트의 일입니다. 통합 테스트는 공개 계약에 집중하세요.
- `tests/common.rs`에 헬퍼 두기 — 그 자체가 빈 테스트 크레이트로 잡힙니다. `tests/common/mod.rs`로 두세요(23.4 박스).
- `#[should_panic]`을 검증의 기본으로 삼기 — "어딘가에서 패닉"만 보장합니다. 검증 실패는 `Result`로, 단언은 `assert_eq!(result, Err(...))`로.
- doctest를 ` ```ignore`로 도배하기 — 가장 약한 보장이라 "검증되는 척하는 문서"가 됩니다. 실행 곤란하면 `no_run`(컴파일은 검증)을 먼저 고려하세요.
- 무작위 시드를 매번 새로 뽑아 스냅샷·속성 테스트를 비결정적으로 만들기 — `proptest`의 회귀 시드를 커밋해 재현성을 지키세요.

> 💡 `cargo nextest`는 차세대 테스트 러너입니다. 각 테스트를 **별도 프로세스로 격리**해 실행하고(한 테스트의 패닉이 다른 테스트 프로세스를 오염시키지 않음), 출력이 더 명확하며 대규모 스위트에서 더 빠릅니다(`cargo install cargo-nextest` 후 `cargo nextest run`). 단, 별도 프로세스 모델이라 **doctest는 실행하지 않으므로**, `cargo nextest run`과 `cargo test --doc`를 CI에서 함께 돌리는 것이 정석입니다.

---

## 23.10 흔한 함정 — Python 습관이 부르는 실수

**함정 1 — 테스트에서 `unwrap`/`expect` 남발.**

Python에서는 테스트 안에서 예외가 나면 그 자체로 테스트가 실패하며 traceback이 찍힙니다. Rust에서 `unwrap()`은 패닉이라 테스트를 실패시키긴 하지만, 어느 단계에서 왜 실패했는지가 흐려집니다. **테스트 함수가 `Result`를 반환하게 하고 `?`로 전파**하면 의도가 명확하고 에러 타입이 그대로 출력됩니다(23.8 렌즈 1의 `refund_within_capture`). "셋업"이 실패하면 안 되는 곳에는 `unwrap()`도 괜찮지만, "검증 대상"의 결과는 `assert_eq!(x, Err(...))`로 *값으로* 확인하세요 — 패닉으로 흘리지 말고.

**함정 2 — 부동소수점을 `assert_eq!`로 비교.**

Python `assert 0.1 + 0.2 == 0.3`이 거짓이라는 건 다들 압니다. Rust도 똑같습니다. `0.1 + 0.2`는 양쪽 언어에서 정확히 `0.30000000000000004`입니다(직접 측정). 그래서 `assert_eq!(sum, 0.3)`은 실패합니다. **근사 비교**를 써야 합니다.

```rust
let sum: f64 = 0.1 + 0.2;
assert!((sum - 0.3).abs() < 1e-10);   // OK: 근사 비교 (epsilon)
// assert_eq!(sum, 0.3);              // 🚨 실패: 0.30000000000000004
```

`pytest.approx`에 대응하는 것이 이 `(a - b).abs() < epsilon` 패턴이며, 실무에서는 `approx` 크레이트의 `assert_relative_eq!`를 쓰면 상대 오차까지 깔끔히 다룰 수 있습니다. 그리고 이 함정의 사촌 — Chapter 8에서 본 대로, `f64`를 품은 뉴타입에 `#[derive(Eq, Ord)]`를 붙이면 `error[E0277]: the trait bound 'f64: Eq' is not satisfied`로 *컴파일 단계에서* 막힙니다. 부동소수점 동등성의 위험을 타입 시스템이 미리 경고하는 셈입니다(그래서 이 장의 `Cents`는 `i64` 기반이라 `Eq`/`Ord`가 안전합니다).

**함정 3 — doctest에서 타입 추론이 막힌다.**

doctest는 독립 프로그램으로 컴파일되므로, 본문 코드보다 타입 추론에 쓸 단서가 적습니다. 예컨대 부동소수점 리터럴만으로는 `f64`인지 `f32`인지 컴파일러가 못 정해 막힙니다. 실제로 위 함정 2의 doctest를 `let sum = 0.1 + 0.2;`(타입 표기 없이)로 쓰면 다음 에러가 납니다.

```text
error[E0689]: can't call method `abs` on ambiguous numeric type `{float}`
 --> src/doctest_check.rs:24:21
  |
4 | assert!((sum - 0.3).abs() < 1e-10);
  |                     ^^^
```

해법은 `let sum: f64 = ...`처럼 타입을 명시하는 것입니다. doctest를 쓸 때는 "이 코드가 *독립적으로* 컴파일되는가"를 의식하세요.

**함정 4 — private을 테스트하려고 무리하게 `pub`을 단다.**

Python의 "private 테스트하려면 공개" 습관이 남아, Rust에서도 내부 함수를 `pub`으로 열려는 충동이 듭니다. 그럴 필요가 없습니다. **같은 파일의 `#[cfg(test)] mod tests`는 `use super::*`로 private에 자유롭게 접근**합니다(23.3). private은 private으로 두고, 단위 테스트를 그 옆에 두세요.

**함정 5 — `tests/common.rs`의 빈 테스트 크레이트 함정.**

통합 테스트끼리 헬퍼를 공유하려고 `tests/common.rs`를 만들면, `cargo`가 그것을 독립 테스트 크레이트로 컴파일해 "running 0 tests"라는 어리둥절한 줄을 출력합니다. 헬퍼는 `tests/common/mod.rs`(서브모듈)로 두고 각 테스트 파일에서 `mod common;`으로 가져오세요. 모듈을 디렉터리로 표현하는 Rust 규칙(Chapter 2)이 테스트 디렉터리에도 그대로 적용됩니다.

**함정 6 — 테스트 병렬성을 잊고 전역 상태를 공유.**

`cargo test`는 기본적으로 한 바이너리 안의 테스트들을 여러 스레드로 병렬 실행합니다(23.5). `pytest`의 기본 직렬 실행에 익숙하면, 같은 임시 파일 경로·같은 환경 변수·같은 DB를 만지는 테스트들이 서로를 오염시키는 사고를 냅니다. 테스트는 독립적으로 설계하고, 불가피하면 `-- --test-threads=1`로 직렬화하거나, 리소스마다 고유 이름(임시 디렉터리 등)을 쓰세요.

이 여섯 함정을 관통하는 교훈은 하나입니다. 파이썬에서 테스트는 "코드를 다 짠 뒤 별도로 붙이는 안전망"이었지만, Rust에서 테스트와 문서는 **언어와 컴파일러가 코드 옆에서 함께 챙기는 1급 시민**입니다. 처음에는 `#[cfg(test)]`나 doctest의 자동 `main` 래핑이 낯설지만, 그 하나하나가 "테스트가 코드와 어긋날 수 없게", "문서가 거짓말할 수 없게" 만드는 장치임을 깨닫게 됩니다. 테스트를 설계의 일부로, 문서를 검증의 일부로 받아들이는 순간, `cargo test`의 초록불은 단순한 통과 신호가 아니라 "이 크레이트의 모든 약속이 지금 이 순간 사실이다"라는 증명서가 됩니다.

---

## 연습문제

**문제 1 — 뉴타입에 doctest 달기.** Chapter 7의 `Price` 뉴타입(`new(value: f64) -> Result<Self, String>`)에 세 개의 doctest를 다세요: (a) 정상 생성, (b) 0 이하 거부, (c) `?` 연산자를 쓰는 예제. (c)에서는 숨김 줄(`#`)로 `Ok(())`를 마무리하세요.

> **정답 스케치.** (a) `let p = Price::new(10.5)?; assert!((p.get() - 10.5).abs() < 1e-9);` — `f64`라 근사 비교. (b) `assert!(Price::new(-1.0).is_err());`. (c) 블록 마지막에 `# Ok::<(), String>(())`를 두면 `?`가 `Result<_, String>` 문맥에서 동작합니다. 부동소수점이므로 doctest에서 `get()` 비교는 반드시 `abs() < epsilon`으로(함정 2). `cargo test --doc`로 셋 다 통과하는지 확인하세요.

**문제 2 — 상태 기계 불변식을 proptest로.** `PaymentStatus`에 대해 "`Pending`에서 시작해 `authorize` → `capture`를 거치면 *항상* `Captured`가 되고, 그 `amount`는 처음 금액과 같다"는 불변식을 `proptest`로 작성하세요. 금액 전략은 `0i64..1_000_000`.

> **정답 스케치.**
> ```rust
> proptest! {
>     #[test]
>     fn capture_preserves_amount(n in 0i64..1_000_000) {
>         let captured = PaymentStatus::Pending { amount: Cents::new(n).unwrap() }
>             .authorize("A".into()).unwrap()
>             .capture("R".into()).unwrap();
>         if let PaymentStatus::Captured { amount, .. } = captured {
>             prop_assert_eq!(amount.value(), n);
>         } else {
>             prop_assert!(false, "must be Captured");
>         }
>     }
> }
> ```
> 핵심은 "전이가 금액을 보존한다"를 *모든* 입력에 대해 선언한 것입니다. 만약 `capture` 구현이 실수로 금액을 0으로 만들면, proptest가 작은 반례(`n = 1`)로 즉시 잡아냅니다.

**문제 3 — 잘못된 전이 표를 rstest로.** "각 상태에서 어떤 액션이 잘못된 전이인가"를 `#[rstest]` + `#[case(...)]` 표로 작성하세요. 최소 4개 케이스: (Pending, capture), (Pending, refund), (Authorized, refund), (Captured, authorize).

> **정답 스케치.** `#[case("Pending", "capture")]`처럼 (시작 상태 라벨, 액션)을 받아, 해당 상태를 만들고 액션을 호출한 뒤 `assert!(matches!(result, Err(TransitionError::InvalidTransition { from, action } ) if from == expected_from))`로 검증합니다. 상태 생성은 `match start { "Pending" => PaymentStatus::Pending {...}, ... }`로 분기. 각 `#[case]`가 `case_1`, `case_2`...로 펼쳐져 어느 조합이 깨졌는지 즉시 보입니다.

**문제 4 — 통합 vs 단위 경계 판단.** 다음 셋을 단위 테스트와 통합 테스트 중 어디에 둘지 고르고 이유를 한 줄로: (a) `Cents`의 private 필드 정규화 로직, (b) "사용자가 `ch23::PaymentStatus`를 import해 결제 전체 흐름을 돈다", (c) `label()`이 각 variant에 올바른 문자열을 반환하는지.

> **정답 스케치.** (a) **단위** — private 세부라 `use super::*`로만 접근 가능. (b) **통합** — `use ch23::...`로 외부 사용자 관점, 공개 API의 충분함을 검증. (c) **단위** 권장 — variant별 경계 조건이고 구현 옆에 두면 명세서 역할. (c)를 통합에 둬도 동작은 하지만(공개 메서드이므로), 구현 세부와 가까운 검증은 단위가 자연스럽습니다(23.4 판단 기준).

**문제 5 — `no_run` vs `ignore` 선택.** 외부 HTTP API를 호출하는 함수 `fetch_rate(url: &str) -> Result<Cents, _>`의 doctest를 작성한다면 ` ```no_run`과 ` ```ignore` 중 무엇을 쓸지, 그리고 그 이유는?

> **정답 스케치.** **`no_run`**. 실행하면 네트워크에 의존해 CI가 불안정해지지만, *컴파일은* 해야 "함수 시그니처·반환 타입이 문서와 일치하는가"를 검증할 수 있습니다. `ignore`는 컴파일조차 안 하므로 시그니처가 바뀌어도 문서가 조용히 거짓이 됩니다 — 가장 약한 보장이라 최후의 수단입니다(23.6, 23.9 안티패턴).

---

지금까지 Part 9 "프로덕션으로 가는 길"의 첫 조각을 맞췄습니다. 우리는 테스트가 `tests/` 디렉터리에 유배되는 대신 구현 옆에서 private까지 검증하고(`#[cfg(test)]`), 문서의 예제가 거짓말하지 못하도록 컴파일러가 실행하며(doctest), 우리가 떠올리지 못한 반례를 기계가 찾아 최소값까지 축소하는(`proptest`) 세계를 보았습니다. `pytest` 생태계가 여러 도구의 조합으로 풀던 일을, Rust는 `cargo test` 하나와 소수의 잘 설계된 크레이트(`rstest`/`insta`/`proptest`)로 통합합니다.

핵심은 도구의 개수가 아니라 **신뢰의 출처**입니다. 파이썬에서 "초록불"은 "우리가 작성한 케이스가 통과했다"는 뜻이었지만, Rust에서 `cargo test`의 초록불은 "구현·통합·문서·속성이 *전부* 이 순간 일관되다"는 더 강한 명제입니다. 검증된 코드는 신뢰할 수 있는 소프트웨어의 절반입니다. 나머지 절반 — 그 코드를 어떻게 빌드·배포하고, 어떻게 다른 언어와 안전하게 연결하며, 어떻게 운영 환경에서 관찰하는가 — 가 Part 9의 다음 장들을 기다리고 있습니다.

---

## References

- https://doc.rust-lang.org/book/ch11-00-testing.html
- https://doc.rust-lang.org/book/ch11-03-test-organization.html
- https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html
- https://doc.rust-lang.org/cargo/commands/cargo-test.html
- https://doc.rust-lang.org/reference/attributes/testing.html
- https://docs.rs/proptest/latest/proptest/
- https://docs.rs/rstest/latest/rstest/
- https://docs.rs/insta/latest/insta/
- https://nexte.st/
