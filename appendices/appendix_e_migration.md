# 부록 E. Python → Rust 마이그레이션 플레이북

Chapter 22에서 우리는 PyO3로 파이썬 병목 함수 하나를 Rust로 옮겨 7배에서 40배까지 끌어올리는 기술을 익혔습니다. 도구는 이제 충분합니다. 하지만 메스를 다룰 줄 안다고 어느 장기를 절제할지 아는 것은 아닙니다.

이 부록은 "어떻게 옮기는가"가 아니라 **"무엇을, 언제, 어디까지 옮기는가"**에 답합니다. 코드 예제는 최소화하고 **체크리스트와 판단 기준**에 집중합니다. 마이그레이션은 기술 문제이기 이전에 ROI(Return on Investment) 문제이기 때문입니다.

이 부록을 관통하는 단 하나의 원칙은 **"전부 다시 짜지 말고 1%의 핫스팟을 노려라"**입니다. 마지막 절에서 다시 돌아올 이 문장을 읽는 내내 머릿속에 띄워 두시기 바랍니다.

---

## E.1 언제 Rust로 옮길 가치가 있는가 — 프로파일링과 ROI

마이그레이션의 첫 단계는 코드를 짜는 것이 아니라 **측정하는 것**입니다. "이 함수가 느릴 것 같다"는 직감은 거의 항상 틀립니다. 측정 없이 Rust로 옮기는 것은, Chapter 22에서 말했듯, 도박입니다.

### 핫스팟 식별 — 어떤 프로파일러를 언제 쓰는가

파이썬 프로파일러는 목적이 다릅니다. 상황에 맞는 도구를 고르세요.

| 도구 | 방식 | 언제 쓰는가 | 오버헤드 |
|------|------|------------|----------|
| `cProfile` | 결정론적(deterministic) 함수 호출 추적 | 개발 환경에서 함수별 누적 시간 파악 | 큼(2~10배 느려짐) |
| `py-spy` | 샘플링(sampling) 프로파일러 | **운영 중인 프로세스**에 attach, 코드 수정 불필요 | 거의 없음 |
| `scalene` | CPU·메모리·GPU 동시 프로파일링, 라인 단위 | CPU와 메모리 병목을 함께 봐야 할 때 | 중간 |
| `line_profiler` | 함수 내 라인별 시간 | 특정 함수 내부의 어느 줄이 느린지 | 큼 |

실전 순서는 이렇습니다. 먼저 `py-spy`로 운영 프로세스를 건드리지 않고 전체 그림을 봅니다. 코드 수정 없이 살아 있는 서비스에 붙일 수 있다는 점이 결정적입니다.

```bash
# 운영 중인 PID 1234 프로세스를 30초간 샘플링해 flame graph 생성
$ py-spy record -o profile.svg --pid 1234 --duration 30

# 실시간으로 어느 함수가 CPU를 먹는지 top 처럼 관찰
$ py-spy top --pid 1234
```

flame graph에서 가로로 가장 넓은 막대 — 전체 시간의 가장 큰 비중을 차지하는 함수 — 가 후보입니다. 그다음 `scalene`이나 `line_profiler`로 확대해 내부의 어느 라인이 범인인지 좁힙니다.

### CPU 바운드인가, I/O 바운드인가 — 가장 중요한 갈림길

이것이 마이그레이션 의사결정의 핵심 분기입니다. Rust는 **CPU 시간을 줄이는** 도구이지 **대기 시간을 줄이는** 도구가 아닙니다. 함수가 디스크·네트워크·DB 응답을 기다리며 시간을 보낸다면 언어를 바꿔도 시계는 그대로 흐릅니다. PostgreSQL이 200ms 걸려 쿼리를 돌려주는 동안, 기다리는 코드가 파이썬이든 Rust든 200ms는 200ms입니다.

옮길 가치가 있는 코드 vs 손대면 안 되는 코드를 구분하는 표입니다.

| 옮길 가치가 높음 (CPU 바운드) | 옮길 가치가 낮음 (I/O 바운드 · 글루) |
|---|---|
| 수백만 행 로그·CSV 파싱 | DB 쿼리 발행과 결과 대기 |
| JSON/Protobuf 직렬화·역직렬화 핫패스 | 외부 REST API 호출 |
| 토큰화·정규식·문자열 전처리 루프 | 인증·권한 검사 글루 코드 |
| 수치 집계·행렬 연산·시뮬레이션 | 파일 업로드/다운로드 스트리밍 |
| 이미지·오디오 디코딩/인코딩 | 메시지 큐 컨슘 후 라우팅 |
| 암호화·해싱·압축 | 템플릿 렌더링, 설정 로딩 |
| 그래프 탐색·정렬·집합 연산 | 비즈니스 로직 분기(if/else 덩어리) |

판별법은 단순합니다. **CPU 사용률을 보세요.** 함수가 도는 동안 코어 하나가 100%에 붙어 있으면 CPU 바운드, 코어가 놀고 `read()`/`recv()`/`await`에서 시간을 보내면 I/O 바운드입니다. `py-spy`의 `--idle` 옵션은 대기 중인 스레드까지 보여주므로 "겉보기엔 바쁜데 실제로는 대기 중"인 경우를 가려냅니다.

### ROI 판단 기준 — 옮길지 말지의 정량적 체크리스트

병목을 찾았다고 무조건 옮기는 것이 아닙니다. 다음 다섯 가지를 모두 통과해야 ROI가 양수입니다.

- [ ] **전체 시간의 20% 이상**을 이 구간이 소비하는가? (암달의 법칙(Amdahl's Law): 5%짜리 구간을 무한히 빠르게 만들어도 전체는 최대 5%만 빨라집니다.)
- [ ] **CPU 바운드**인가? (위 표로 확인. 아니면 즉시 중단.)
- [ ] 입출력 자료형이 **단순하거나 numpy 버퍼로 표현** 가능한가? (복잡한 객체 그래프는 변환 비용이 본체를 압도 — Chapter 22.4.1.)
- [ ] 호출 빈도 대비 **함수 본체가 충분히 무거운가**? (마이크로초짜리 함수를 수백만 번 부르면 FFI 경계 오버헤드가 이득을 잡아먹습니다.)
- [ ] **자주 바뀌지 않는 안정된 알고리즘**인가? (매주 스펙이 바뀌는 코드는 빌드·재배포 비용이 매주 발생합니다.)

다섯 항목이 모두 체크되면 ROI는 거의 확실히 양수입니다. 특히 첫 번째(20% 임계)와 두 번째(CPU 바운드)는 타협 불가능한 전제 조건입니다.

> 💡 **암달의 법칙을 마이그레이션 언어로.** 전체의 30%를 먹는 구간을 10배 빠르게 만들면 새 전체 시간은 `0.7 + 0.3/10 = 0.73`(1.37배). 같은 구간이 70%였다면 `0.3 + 0.7/10 = 0.37`(2.7배)입니다. **같은 노력, 같은 10배 가속인데 결과는 두 배 차이.** flame graph의 가로 폭이 곧 ROI입니다.

---

## E.2 세 가지 마이그레이션 전략

병목을 특정하고 ROI가 양수임을 확인했다면, 다음은 **어떤 형태로** 옮길지 결정합니다. 세 가지 전략이 있고 권장 순위가 명확합니다.

### 전략 1 — PyO3 네이티브 확장 (권장 기본값)

핫스팟 함수만 Rust로 짜서 `maturin`으로 wheel을 만들고 파이썬에서 `import`해 쓰는 방식입니다. Chapter 22 전체가 이 전략의 교과서입니다. **특별한 이유가 없는 한 항상 이것부터 검토하세요.**

- **장점**: 기존 코드베이스·배포 파이프라인을 거의 그대로 유지. 같은 프로세스 내 함수 호출이라 네트워크 왕복이 없고, 점진적 도입과 폴백(fallback)이 가장 쉬움. `pydantic-core`·`polars`·`ruff`가 모두 이 길.
- **단점**: Rust 빌드 인프라(CI에서 manylinux/abi3 wheel 빌드)를 새로 갖춰야 함. 경계의 타입 변환 비용을 의식해야 함(E.5). 팀에 Rust 빌드를 이해하는 사람이 최소 1명 필요.
- **언제**: CPU 바운드 핫스팟이 좁고, 같은 프로세스에서 동기 호출되며, 기존 아키텍처를 흔들고 싶지 않을 때. **대부분의 경우가 여기에 해당합니다.**

### 전략 2 — 독립 마이크로서비스로 분리 (gRPC/REST)

병목 기능을 별도의 Rust 서비스로 떼어내고, 파이썬은 네트워크(gRPC/REST/메시지 큐)로 호출합니다.

- **장점**: 언어 경계가 곧 네트워크 경계라 배포·스케일·장애 격리가 독립적. Rust 서비스만 따로 수평 확장(scale-out)하고, 빌드 파이프라인이 분리되며, 팀을 나눠 개발 가능.
- **단점**: 네트워크 왕복(수백 마이크로초~밀리초)과 직렬화 비용이 추가됨 — **PyO3 함수 호출(나노초~마이크로초)보다 수백~수천 배 비싼 경계.** 운영 복잡도(서비스 디스커버리, 모니터링)와 분산 시스템의 어려움(부분 실패, 타임아웃, 재시도)을 떠안음.
- **언제**: (1) 기능이 이미 독립적인 서비스 단위일 때(검색 인덱싱, 추천 엔진), (2) 호출 빈도가 낮고 한 번의 호출이 무거워 네트워크 오버헤드가 무시할 만할 때, (3) 독립적 스케일링·배포 주기가 필요할 때, (4) 여러 언어의 클라이언트(Node·Go 등)가 같은 기능을 호출할 때.

> 🚨 **흔한 오판.** "마이크로서비스가 더 깔끔해 보인다"는 이유만으로 전략 2를 고르는 것은 함정입니다. 마이크로초 단위 핫패스를 네트워크 너머로 보내면 직렬화 + 왕복이 본체를 압도해 **오히려 느려집니다.** 경계가 비쌀수록 한 번에 더 큰 배치를 넘겨야 정당화됩니다.

### 전략 3 — 전면 재작성 (거의 권장하지 않음)

서비스 전체를 Rust로 다시 짜는 것입니다. **이 책 전체가 경계하는 안티패턴**입니다(Chapter 22 도입부의 "월요일에 출근해서 전부 다시 짜겠다는 선언" 참조).

- **왜 거의 항상 나쁜가**: 비즈니스 로직의 90%는 I/O 바운드 글루 코드라 옮겨도 빨라지지 않고(E.1), 그동안 새 기능 개발은 멈추며, 검증된 파이썬 코드의 수많은 엣지 케이스를 처음부터 다시 발견·재현해야 합니다. *Things You Should Never Do*(Joel Spolsky)가 경고한 "처음부터 다시 짜기"의 함정 — 숨은 비즈니스 규칙 소실, 일정 폭주, 기능 동결 — 을 그대로 밟습니다.
- **정당화되는 드문 경우**: (1) 기존 코드베이스가 **소규모**(수천 줄 이하)이고 CPU 바운드 비중이 압도적인 단일 목적 도구, (2) 시작부터 **파이썬으로는 도달 불가능한 지연·메모리 요구사항**(임베디드, 실시간, 마이크로초 SLA)이 명확할 때, (3) 기존 코드가 어차피 폐기 예정이고 새 요구사항이 본질적으로 시스템 프로그래밍일 때. `ruff`는 기존 린터를 "재작성"한 게 아니라 **새 시장을 Rust 네이티브로 개척**한 것에 가깝습니다 — 이 구분이 중요합니다.

의사결정 순서는 명확합니다. **전략 1을 기본값으로 두고, 네트워크 경계가 명백히 정당화될 때만 전략 2로, 전략 3은 "재작성이 아니라 신규 개발"일 때만** 고려하세요.

---

## E.3 단계별 플레이북

전략을 골랐다면(대개 전략 1) 이제 실행입니다. 다음 8단계는 핫스팟 하나를 안전하게 교체하는 표준 절차이며, 각 단계는 **검증 기준(verify)**을 동반합니다 — 검증 없이 다음 단계로 넘어가지 마세요.

1. **측정 (Measure)** → 검증: `py-spy`/`cProfile`로 핫스팟을 특정하고 전체의 20% 이상을 차지함을 수치로 확인. 이 수치가 비교의 기준선(baseline)이 됩니다.

2. **경계 설정 (Define the boundary)** → 검증: Rust로 넘길 함수의 입력·출력을 적습니다. "`list[str]`을 받아 `dict[int, float]`를 반환한다"처럼 한 문장으로 못 쓰면 경계가 잘못 잡힌 것입니다. 경계는 **좁고 굵게** — 한 줄씩이 아니라 배치 단위로.

3. **인터페이스 설계 (Design the interface)** → 검증: 파이썬 시그니처를 먼저 확정하고 그걸 만족하는 Rust 시그니처(`#[pyfunction]`)를 적습니다(타입 매핑은 E.4). 기존 함수와 **동일한 시그니처**를 유지하면 호출부 수정이 없어 폴백이 쉬워집니다.

4. **Rust 모듈 작성 (Implement)** → 검증: `maturin develop`로 컴파일·`import`가 되는지 확인. 이 단계에서는 성능이 아니라 **정확성**만 봅니다.

5. **테스트 — 파이썬 결과와 대조 (Test against the oracle)** → 검증: 기존 파이썬 함수를 **오라클(oracle)**로 삼아, 동일 입력에 두 구현의 출력이 일치하는지 자동 테스트로 확인. 마이그레이션 안전성의 핵심입니다.

   ```python
   # 기존 파이썬 함수(aggregate_py)를 정답지로 삼아 Rust 구현을 검증
   import random, fast_agg
   from legacy import aggregate_py

   def test_rust_matches_python():
       for _ in range(100):
           lines = [f"{random.randint(0, 99)},{random.random():.4f}"
                    for _ in range(1000)]
           expected = aggregate_py(lines)
           actual = fast_agg.aggregate(lines)
           # 부동소수점은 정확 일치 대신 허용 오차로 비교
           assert expected.keys() == actual.keys()
           for k in expected:
               assert abs(expected[k] - actual[k]) < 1e-9
   ```

   > 부동소수점 합산은 연산 순서에 따라 마지막 비트가 달라질 수 있습니다(특히 병렬 `reduce`에서). 정확 일치(`==`)가 아니라 **허용 오차(tolerance)** 비교를 쓰세요. property-based 테스트(`hypothesis`)로 무작위 입력을 던지면 엣지 케이스를 더 잘 잡습니다.

6. **벤치마크 (Benchmark)** → 검증: 반드시 `maturin develop --release`로 빌드한 뒤 1단계의 기준선과 같은 입력 크기로 비교. 디버그 빌드 수치는 무의미하며, 변환 비용을 **포함한** 정직한 비교를 합니다(Chapter 22.7).

7. **점진적 배포 (Roll out gradually)** → 검증: 한 번에 전체 트래픽을 보내지 마세요. 피처 플래그나 환경 변수로 Rust·파이썬 경로를 **런타임에 전환**하고, 카나리(canary)로 일부 트래픽부터 흘립니다.

   ```python
   import os

   if os.getenv("USE_RUST_AGG") == "1":
       from fast_agg import aggregate as aggregate_impl
   else:
       from legacy import aggregate_py as aggregate_impl  # 폴백
   ```

8. **모니터링 (Monitor)** → 검증: 배포 후 지연(p50/p99), 에러율, 메모리, CPU를 기준선과 비교. `pyo3_runtime.PanicException`이 로그에 찍히는지 반드시 감시 — Rust 패닉이 새는 신호입니다. 이상 시 7단계 플래그로 **즉시 파이썬으로 롤백**.

이 8단계의 미덕은 **언제든 되돌릴 수 있다는 것**입니다. 3단계에서 시그니처를 보존하고 7단계에서 플래그를 심었기에, 문제가 생기면 환경 변수 하나로 검증된 파이썬 구현으로 돌아갑니다. 이 안전망이 "한 번에 전부 옮기기"와 "점진적 마이그레이션"을 가르는 결정적 차이입니다.

---

## E.4 타입 매핑 의사결정

동적 타입을 정적 타입으로 옮기는 일은 단순 변환이 아니라 **설계**입니다. 파이썬에서 암묵적이던 것(이 값이 None일 수 있는가? 이 dict의 키는?)을 Rust에서는 타입으로 명시해야 하고, 그 과정에서 코드의 숨은 계약이 드러납니다 — 옮기다가 원래 코드의 버그를 발견하는 일이 흔합니다.

| 파이썬 | Rust | 설계 판단 |
|--------|------|-----------|
| `int` | `i64` / `i32` / `u64` | 음수 가능성·범위를 결정해야 함. 파이썬 `int`는 무한정밀도이므로 오버플로 가능성 검토(큰 수면 `i128`이나 `BigInt`) |
| `float` | `f64` | 거의 항상 `f64`. 파이썬 `float`는 C double과 동일 |
| `str` | `String` / `&str` | 소유가 필요하면 `String`, 빌림이면 `&str`. 경계 복사 비용 의식(Chapter 22.4.1) |
| `bytes` | `Vec<u8>` / `&[u8]` | 제로카피 가능 후보 |
| `None` 가능한 값 | `Option<T>` | 파이썬의 암묵적 None을 **타입으로 명시**. `Optional[X]` → `Option<X>` |
| `Union[A, B]` / `A \| B` | `enum` | 가장 중요한 변환. 합 타입(sum type)으로 표현 (Chapter 10) |
| `dict[str, Any]` (고정 스키마) | `struct` | 키가 미리 정해져 있으면 `struct`. 필드가 곧 키 (Chapter 7) |
| `dict[K, V]` (동적 키) | `HashMap<K, V>` | 키가 런타임에 결정되면 `HashMap` |
| `list[T]` | `Vec<T>` | 동질 컬렉션 |
| `tuple[A, B]` | `(A, B)` | 고정 길이·이질 타입 |
| 예외(raise) | `Result<T, E>` | `try/except` → `Result` + `?` (Chapter 12~13) |
| `@dataclass` | `#[derive(...)] struct` | Pydantic 모델도 동일 (Chapter 7) |

표에서 가장 중요한 네 가지 결정만 짚습니다.

- **Optional → Option**: 파이썬의 암묵적 `None`을 `Option<T>`로 강제. 진짜 가치는 **호출부가 None 처리를 빼먹을 수 없게** 만드는 것입니다. `None.foo()`로 새벽 호출기를 울리던 `AttributeError`가 컴파일 단계에서 차단됩니다.
- **Union → enum (가장 중요)**: `isinstance` 분기를 `enum`(Chapter 10)으로 표현하면 `match`가 모든 변종(variant)을 다뤘는지 **컴파일러가 검사**합니다(exhaustiveness). 새 변종을 추가하면 처리하지 않은 모든 `match`가 컴파일 에러가 되어, `isinstance` 체인 하나를 빠뜨려 조용히 오분기하던 버그가 원천 봉쇄됩니다.
- **dict → struct vs HashMap**: **키가 작성 시점에 고정이면 `struct`, 런타임에 결정되면 `HashMap`**. 파이썬 개발자는 습관적으로 모든 것을 `dict`로 다루지만, `{"name": ..., "age": ...}`처럼 키가 사실상 고정인 dict를 `HashMap`으로 옮기면 (1) 없는 키 접근이 런타임 에러로 남고, (2) 매 접근마다 해시 비용이 들며, (3) IDE 자동완성·타입 검사를 못 받습니다. 이런 건 반드시 `struct`로.
- **예외 → Result**: `try/except`를 `Result<T, E>`와 `?`로 옮깁니다(Chapter 12~13). `From<MyError> for PyErr`를 한 곳에 정의하면 본문은 `?`로 깔끔히 전파하고 경계에서 자동으로 파이썬 예외가 됩니다(Chapter 22.6). 어떤 예외 타입(`ValueError`인가 `RuntimeError`인가)으로 매핑할지 이 지점에서 명시하게 되어, 모호하던 에러 계약이 분명해집니다.

---

## E.5 데이터 경계 설계 — FFI 왕복을 최소화하라

마이그레이션 성능은 함수 본체가 아니라 **경계(boundary)**에서 결정되는 경우가 많습니다. Chapter 22.4.1에서 보았듯 `Vec<String>`으로 1,000만 줄을 받으면 함수가 한 줄도 실행되기 전에 1,000만 번의 힙 할당·`memcpy`가 청구됩니다. 경계 설계의 세 가지 원칙입니다.

### 원칙 1 — 직렬화 비용을 의식하라

파이썬 객체를 Rust 타입으로(혹은 반대로) 옮기는 모든 변환은 비용입니다. 자동 변환(`Vec<String>`, `HashMap`)은 편하지만 보이지 않는 복사를 숨깁니다. 경계를 넘기기 전에 항상 물으세요. **"여기서 몇 바이트가 복사되는가?"** 작은 데이터(수백~수천 개)면 무시해도 되지만 대용량이면 이 비용이 본체를 압도합니다.

### 원칙 2 — 가능하면 제로카피(Zero-copy)

수치·바이트 데이터는 복사하지 말고 **빌려오세요**. `numpy` 크레이트의 `PyReadonlyArray1`은 버퍼 프로토콜(PEP 3118)로 노출된 포인터를 Rust 슬라이스(`&[f64]`)로 재해석할 뿐 한 바이트도 복사하지 않습니다(Chapter 22.7). 80MB 배열을 `Vec<f64>`로 받으면 80MB를 복사하지만 제로카피는 포인터 8바이트만 넘깁니다. `polars`가 Apache Arrow 메모리 위에서 복사 없이 수십 배 빠른 비결입니다.

### 원칙 3 — 배치로 FFI 왕복을 줄여라

FFI 경계는 호출 자체에 고정 오버헤드가 있어, 파이썬 루프 안에서 원소마다 Rust 함수를 부르는 것은 최악의 패턴입니다.

```python
# 🚨 나쁨: 1천만 번의 FFI 경계 왕복. 경계 오버헤드가 본체를 압도.
result = [fast_lib.process_one(x) for x in items]

# ✅ 좋음: 단 한 번의 경계 횡단. 리스트 전체를 넘기고 결과를 한 번 받음.
result = fast_lib.process_batch(items)
```

경계 비용이 클수록(특히 전략 2의 네트워크 경계) 배치 크기를 키워야 정당화됩니다. PyO3 인프로세스 경계는 배치 1,000 정도로 충분하지만, gRPC 경계는 밀리초 단위라 배치를 수만~수십만으로 키워야 할 수 있습니다.

| 경계 종류 | 1회 횡단 비용(대략) | 권장 배치 전략 |
|-----------|---------------------|----------------|
| PyO3 인프로세스 함수 호출 | 나노초~마이크로초 | 자료형 변환만 의식, 배치 1천 내외 |
| 제로카피 numpy 버퍼 | 거의 0 (포인터 전달) | 배열 통째로 한 번 |
| gRPC/REST 네트워크 | 수백 마이크로초~밀리초 | 배치 수만 이상, 직렬화 포맷 최적화 |

---

## E.6 흔한 함정

Chapter 22.8의 함정을 마이그레이션 관점에서 재정리하고, 프로젝트 레벨의 함정을 더합니다.

- **함정 1 — GIL을 안 풀고 멀티코어를 기대한다.** 순수 연산 구간을 `py.allow_threads()`로 감싸지 않으면 `rayon`을 써도 다른 파이썬 스레드가 전부 멈춥니다. "Rust로 바꿨는데 서버 동시성이 더 나빠졌다"의 단골 원인(Chapter 22.5).
- **함정 2 — 과도한 데이터 복사.** 자동 변환의 편리함 뒤에 숨은 경계의 `memcpy`. 대용량이면 제로카피를 검토하세요(E.5).
- **함정 3 — 너무 잘게 쪼갠 FFI 경계.** 원소마다 호출하면 경계 오버헤드가 본체를 압도합니다. 배치로 넘기세요(E.5).
- **함정 4 — 에러 변환 누락.** `From<MyError> for PyErr`를 정의하지 않으면 의미 없는 에러가 파이썬으로 올라갑니다. 어떤 예외 타입으로 매핑할지 명시적으로 설계하세요(E.4, Chapter 22.6).
- **함정 5 — 디버그 빌드로 벤치마크한다.** Rust 디버그 빌드는 릴리스보다 10배 이상 느립니다. 성능 이야기는 무조건 `--release`(Chapter 22.8).
- **함정 6 — 빌드·배포 인프라를 간과한다.** 가장 자주 프로젝트를 좌초시키는 함정입니다. "로컬은 되는데 CI wheel이 안 빌드된다", "운영 glibc와 충돌한다", "3.12 wheel이 없다"가 배포 직전에 터집니다. 해결책: **manylinux**(오래된 공통 glibc 이미지에서 빌드, `maturin-action`으로 자동화), **abi3**(`features = ["abi3-py38"]`로 하나의 wheel이 3.8+ 전부 동작, CI 매트릭스를 `3×N`→`1×OS`로 축소, Chapter 22.3). **CI wheel 빌드를 첫날부터 세팅하세요** — 마지막에 붙이면 반드시 후회합니다.
- **함정 7 — 전부 Rust로 옮기려 한다.** Rust는 CPU 병목 전용 수술 도구이지 만병통치약이 아닙니다(E.1). I/O 바운드 핸들러를 옮기면 코드만 복잡해지고 속도는 그대로입니다.
- **함정 8 — 폴백 경로 없이 배포한다.** 피처 플래그 없이 전체 트래픽을 새 구현으로 보내면 문제 발생 시 재배포까지 서비스가 멈춥니다. E.3의 7단계는 선택이 아니라 필수입니다.

---

## E.7 점진적 도입 체크리스트

코드 한 줄 짜기 전에, 팀과 인프라가 준비되었는지 다음 체크리스트로 점검하세요.

### 팀 역량 및 학습 곡선

- [ ] 팀에 Rust 소유권(Ownership)·대여(Borrowing)·수명(Lifetime)을 이해하는 사람이 **최소 1명** 있는가? (이 책의 Part 2를 읽은 사람)
- [ ] 그 사람이 부재중일 때 Rust 모듈을 **유지보수할 두 번째 사람**이 있는가? (버스 팩터(bus factor) 1을 피하라)
- [ ] 코드 리뷰에서 Rust 코드를 검토할 수 있는가? 아니면 첫 모듈은 의도적으로 작게 시작해 팀이 함께 학습하는가?
- [ ] `clippy`·`rustfmt`를 CI에 넣어 스타일·안티패턴을 자동 검출하는가?

### 빌드 인프라

- [ ] CI에서 Rust 툴체인(`rustup`, `cargo`)을 설치·캐싱하는 파이프라인이 있는가?
- [ ] `maturin`으로 wheel을 빌드하는 CI 잡이 있는가? (`maturin-action` 권장)
- [ ] 빌드 시간이 파이썬-only 대비 얼마나 늘어나는지 측정했는가? (Rust 컴파일은 느립니다. 캐싱 전략 필수)
- [ ] `cargo`/Rust 의존성의 보안 감사(`cargo audit`)를 파이프라인에 넣었는가?

### Wheel 배포

- [ ] 어느 OS·아키텍처(linux x86_64/aarch64, macos, windows)를 지원할지 결정했는가?
- [ ] **manylinux** 표준으로 빌드해 운영 환경 glibc와의 호환을 확보했는가?
- [ ] **abi3**를 켜서 파이썬 마이너 버전마다 따로 빌드하지 않는가? (켜지 않으면 매 버전 wheel 필요)
- [ ] 사내 PyPI(또는 PyPI)에 wheel을 올리고 `pip install`로 설치되는지 **운영과 동일한 환경**에서 검증했는가?
- [ ] 소스 배포(sdist) 폴백이 있는가? (지원하지 않는 플랫폼에서 사용자가 직접 컴파일할 수 있게)

### 폴백 전략

- [ ] Rust 경로와 파이썬 경로를 **런타임에 전환**하는 피처 플래그가 있는가? (E.3 7단계)
- [ ] 순수 파이썬 기준 구현(reference implementation)을 **삭제하지 않고 유지**하는가? (오라클 테스트용 + 폴백용)
- [ ] Rust 모듈 `import` 실패 시 자동으로 파이썬 구현으로 떨어지는 `try/except ImportError` 가드가 있는가?
- [ ] 카나리 배포로 일부 트래픽부터 흘리고, 이상 시 즉시 롤백하는 절차가 문서화되어 있는가?

```python
# import 실패 시에도 죽지 않는 폴백 가드 — wheel이 없는 환경 대비
try:
    from fast_agg import aggregate as _aggregate
except ImportError:
    from legacy import aggregate_py as _aggregate  # 순수 파이썬 폴백

aggregate = _aggregate
```

이 체크리스트의 절반 이상에 체크하지 못한다면 **인프라와 팀을 먼저 준비**하세요. 마이그레이션이 실패하는 가장 흔한 이유는 Rust 코드가 어려워서가 아니라 빌드·배포·운영 준비가 부족해서입니다.

---

## E.8 미니 프로젝트 청사진 — 로그 집계 함수 하나만 옮기기

이제 플레이북을 한 파일 단위로 압축해 봅시다. 목표는 거창한 서비스 재작성도, 새 마이크로서비스도 아닙니다. **순수 파이썬 로그 집계 함수 하나를 Rust로 교체하되, 오라클 테스트와 폴백 경로를 남기는 것**입니다.

기존 파이썬 기준 구현은 다음과 같습니다.

```python
# legacy.py
from __future__ import annotations

def aggregate(lines: list[str]) -> dict[int, float]:
    totals: dict[int, float] = {}
    for line in lines:
        sensor, value = line.split(",", 1)
        sid = int(sensor)
        totals[sid] = totals.get(sid, 0.0) + float(value)
    return totals
```

Rust 구현은 같은 시그니처를 유지합니다. 처음부터 제로카피까지 욕심내지 말고, `list[str] -> dict[int, float]` 자동 변환으로 정확성을 먼저 확보합니다.

```rust
use pyo3::exceptions::PyValueError;
use pyo3::prelude::*;
use std::collections::HashMap;

#[pyfunction]
fn aggregate(lines: Vec<String>) -> PyResult<HashMap<i64, f64>> {
    let mut totals = HashMap::new();

    for line in lines {
        let (sensor, value) = line
            .split_once(',')
            .ok_or_else(|| PyValueError::new_err(format!("invalid line: {line}")))?;
        let sid: i64 = sensor
            .parse()
            .map_err(|_| PyValueError::new_err(format!("invalid sensor id: {sensor}")))?;
        let value: f64 = value
            .parse()
            .map_err(|_| PyValueError::new_err(format!("invalid value: {value}")))?;
        *totals.entry(sid).or_insert(0.0) += value;
    }

    Ok(totals)
}

#[pymodule]
fn fast_agg(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(aggregate, m)?)?;
    Ok(())
}
```

여기서 중요한 것은 `unwrap()`이 하나도 없다는 점입니다. 입력이 깨졌을 때 Rust 패닉이 아니라 파이썬의 `ValueError`로 올라갑니다. 마이그레이션 초기에 가장 많이 놓치는 품질 기준입니다.

오라클 테스트는 기존 파이썬 함수를 정답지로 삼습니다.

```python
# tests/test_fast_agg.py
from __future__ import annotations

import os
import random

from legacy import aggregate as aggregate_py

try:
    from fast_agg import aggregate as aggregate_rust
except ImportError:  # wheel이 없는 개발 환경
    aggregate_rust = None


def make_lines(seed: int, n: int) -> list[str]:
    rng = random.Random(seed)
    return [
        f"{rng.randint(0, 127)},{rng.random():.6f}"
        for _ in range(n)
    ]


def test_rust_matches_python() -> None:
    if aggregate_rust is None:
        return
    for seed in range(20):
        lines = make_lines(seed, 1_000)
        expected = aggregate_py(lines)
        actual = aggregate_rust(lines)
        assert expected.keys() == actual.keys()
        for key in expected:
            assert abs(expected[key] - actual[key]) < 1e-9


def test_fallback_switch() -> None:
    os.environ["USE_RUST_AGG"] = "0"
    from wrapper import aggregate

    assert aggregate(["1,2.5", "1,0.5"]) == {1: 3.0}
```

운영 코드에서는 직접 `fast_agg.aggregate`를 import하지 말고, 얇은 wrapper를 둡니다.

```python
# wrapper.py
from __future__ import annotations

import os

from legacy import aggregate as aggregate_py

if os.getenv("USE_RUST_AGG", "0") == "1":
    try:
        from fast_agg import aggregate as aggregate_impl
    except ImportError:
        aggregate_impl = aggregate_py
else:
    aggregate_impl = aggregate_py


def aggregate(lines: list[str]) -> dict[int, float]:
    return aggregate_impl(lines)
```

이 구조의 장점은 배포 실패를 재배포 없이 되돌릴 수 있다는 것입니다. wheel import가 실패해도 파이썬 구현으로 떨어지고, 성능 또는 정확성 이상이 보이면 `USE_RUST_AGG=0`으로 즉시 롤백합니다. 카나리는 `USE_RUST_AGG=1`을 일부 워커나 일부 트래픽에만 적용하면 됩니다.

벤치마크는 반드시 wrapper가 아니라 양쪽 구현을 직접 비교합니다. 그래야 폴백 로직과 환경 변수 검사 비용이 끼지 않습니다.

```python
# bench.py
from __future__ import annotations

import random
import time

from legacy import aggregate as aggregate_py
from fast_agg import aggregate as aggregate_rust

lines = [
    f"{random.randint(0, 999)},{random.random():.6f}"
    for _ in range(1_000_000)
]


def bench(name: str, fn) -> None:
    start = time.perf_counter()
    result = fn(lines)
    elapsed = time.perf_counter() - start
    print(f"{name:12s} {elapsed:7.3f}s keys={len(result)}")


bench("python", aggregate_py)
bench("rust", aggregate_rust)
```

이 미니 프로젝트는 작지만, 실제 마이그레이션의 핵심을 모두 담습니다.

- 경계가 한 문장으로 설명됩니다: `list[str] -> dict[int, float]`.
- 기존 파이썬 구현이 오라클로 남아 있습니다.
- Rust 구현은 패닉 대신 파이썬 예외를 반환합니다.
- 테스트가 양쪽 결과를 대조합니다.
- wrapper가 폴백과 카나리를 가능하게 합니다.
- 벤치마크가 `--release` wheel을 기준으로 변환 비용까지 포함합니다.

이 단위를 한 번 성공시키면, 그다음 핫스팟도 같은 템플릿으로 반복하면 됩니다. 마이그레이션을 프로젝트가 아니라 **반복 가능한 수술 절차**로 만드는 것이 핵심입니다.

---

## E.9 실전 사례 패턴 — 거인들이 택한 길

업계의 대표적인 Rust 기반 파이썬 라이브러리들의 공통점은 **하나같이 전략 1(PyO3 네이티브 확장)**이며, **파이썬 API를 그대로 유지한 채 엔진만 Rust로 교체**했다는 것입니다.

| 라이브러리 | 무엇을 옮겼나 | 핵심 패턴 | 마이그레이션 교훈 |
|-----------|--------------|-----------|-------------------|
| **pydantic-core** | 검증(validation) 핫패스 | Pydantic v2가 검증 엔진만 Rust로 분리. 파이썬 API는 그대로 | 가장 자주 도는 좁은 핫스팟(검증)만 정확히 도려냄 → v1 대비 수~수십 배 |
| **polars** | 데이터프레임 엔진 전체 | Apache Arrow 메모리 위 제로카피, 쿼리 최적화·병렬 실행 | 데이터 경계를 Arrow로 통일해 복사 비용 제거(E.5 원칙 2) |
| **tokenizers** (HuggingFace) | 토큰화 전처리 루프 | 문자열 처리 핫패스를 Rust로, 파이썬은 얇은 래퍼 | CPU 바운드 문자열 루프의 교과서적 대상(E.1) |
| **ruff** | 린터·포매터 전체 | 기존 도구를 옮긴 게 아니라 Rust 네이티브로 신규 개발 | "재작성"이 아닌 "신규 개발"로 전략 3의 함정을 회피(E.2) |
| **orjson** | JSON 직렬화/역직렬화 | 직렬화 핫패스만 네이티브, 파이썬 `json` 호환 API | 직렬화는 CPU 바운드의 전형. 드롭인(drop-in) 교체 |
| **cryptography** | 암호 연산 | 암호 프리미티브를 Rust로(기존 C 일부 대체) | 정확성이 생명인 영역에서 Rust의 메모리 안전성 활용 |

여기서 읽어낼 패턴은 세 가지이며, 모두 이 부록이 권한 길과 일치합니다. (1) **모두 좁은 핫스팟을 도려냈다** — 서비스 전체를 옮긴 사례는 없습니다(ruff는 신규 개발). (2) **모두 파이썬 API를 보존해** 드롭인(drop-in) 교체가 가능했습니다(E.3 3단계). (3) **데이터 경계를 신중히 설계했다** — polars는 Arrow, pydantic-core는 검증 결과만 주고받아 경계 복사를 최소화했습니다(E.5).

---

## E.10 마이그레이션 마인드셋 — 1%의 핫스팟을 노려라

이 부록을 한 문장으로 압축하면 첫 절에서 띄워 둔 그 문장으로 돌아옵니다. **"전부 다시 짜지 말고 1%의 핫스팟을 노려라."**

새 도구(Rust)를 배우면 모든 것을 그 도구로 풀고 싶어집니다 — "망치를 쥐면 모든 게 못으로 보인다"는 함정입니다. 하지만 이 책 전체가, 그리고 pydantic-core·polars·tokenizers·ruff가 증명하는 진실은 정반대입니다. **Rust는 파이썬을 대체하러 온 것이 아니라 가장 아픈 1%를 수술하러 온 것입니다.** 비즈니스 로직의 90%는 I/O 대기와 글루 코드이고, 그 영역에서는 파이썬의 생산성·생태계가 압도적입니다. 마이그레이션의 기술은 그 1%를 **정확히 찾아(E.1) 안전하게 도려내고(E.3) 되돌릴 수 있게 배포하는(E.7)** 절제의 기술입니다.

다음 주에 출근하면 이 부록의 순서대로 한 걸음만 떼어 보세요. `py-spy`로 가장 느린 함수를 찾고, CPU 바운드이며 입출력이 단순한지 확인하고, 그 함수 하나만 PyO3로 옮겨 파이썬 결과와 대조하고, `--release`로 벤치마크하세요. 50줄짜리 첫 모듈이 7배 빨라지고 나머지 95%를 손대지 않고도 서비스가 빨라지는 경험은 백 마디 설명보다 강력합니다.

절제하는 외과의가 되십시오. 전체를 들어내는 것이 아니라 가장 아픈 한 곳만 정확히. 그것이 마이그레이션의 처음이자 끝입니다.

---

## References

- https://pyo3.rs/
- https://www.maturin.rs/
- https://github.com/benfred/py-spy
- https://nnethercote.github.io/perf-book/
- https://doc.rust-lang.org/cargo/commands/cargo-bench.html
