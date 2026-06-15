# Chapter 26. 성능 엔지니어링: 프로파일링·criterion·SIMD

10년 차 Python 엔지니어라면 "성능 튜닝"이라는 단어에 본능적인 경계심이 있을 것입니다. 좋은 의미에서입니다. 우리는 추측에 기반한 최적화가 거의 항상 시간 낭비라는 것을 비싼 대가를 치르며 배웠습니다. 어느 새벽 p99 레이턴시 알람에 깨어 `cProfile`을 붙여 보면, 우리가 한 달 전 "여기가 느릴 거야"라며 정성껏 캐싱을 박아둔 함수는 전체 시간의 1.2%였고, 진짜 범인은 아무도 의심하지 않던 `json.loads`와 그 뒤의 dict 컴프리헨션이었습니다. 측정 없는 최적화는 죄악이라는 격언(Donald Knuth의 "premature optimization is the root of all evil")은 Python 세계에서 특히 뼈아프게 사실입니다.

그래서 우리는 측정 도구를 사랑하게 됐습니다. `cProfile`로 호출 그래프를 뜨고, `py-spy`로 운영 중인 프로세스에 무중단으로 샘플러를 붙이고, `line_profiler`로 한 줄 한 줄의 비용을 재고, `memory_profiler`와 `tracemalloc`으로 메모리 누수를 추적합니다. 이 도구들은 훌륭합니다. 문제는 측정이 끝난 뒤입니다. 병목을 정확히 찾아냈는데, 그 병목이 순수 Python 인터프리터 루프 안에 있다면, 우리가 할 수 있는 일은 제한적입니다. 알고리즘을 바꾸거나, `numpy`로 벡터화하거나, C 확장으로 도망가거나. [Chapter 1](./chapter_01.md)에서 해부한 그 비용 — 매 반복마다 `PyObject`를 할당하고, 타입 슬롯을 조회하고, refcount를 증감하는 — 은 Python 코드를 아무리 영리하게 다시 짜도 사라지지 않습니다. 이것이 **Python 최적화의 천장**입니다.

Rust에서 성능 엔지니어링은 출발선부터 다릅니다. 인터프리터 오버헤드라는 바닥짐(ballast)이 없으니, 우리가 측정하는 시간은 거의 전부 "CPU가 실제로 한 일"입니다. 그래서 프로파일러가 가리키는 병목은 "어떻게 인터프리터를 피할까"가 아니라 "이 알고리즘을, 이 메모리 레이아웃을, 이 CPU 명령어를 어떻게 더 좋게 만들까"라는, 한 층 더 근본적인 질문으로 이어집니다. 캐시 미스를 줄이고, 자동 벡터화(auto-vectorization)를 유도하고, 할당을 없애는 — Python에서는 라이브러리 작성자만 만지던 영역이 우리 손에 들어옵니다.

이 챕터는 이 책 전체에 흩어져 있던 벤치마크 방법론을 한곳으로 수렴해 정본화(canonicalize)합니다. 여러분은 [Chapter 1](./chapter_01.md)부터 모든 벤치마크 코드에 `std::hint::black_box`가 등장하는 것을 봤을 것입니다. 왜 그것이 필요한지, 왜 그것 없이는 "1000배 빠름" 같은 거짓 수치가 나오는지를 여기서 끝까지 파고듭니다. 통계적으로 정직한 측정을 자동화하는 `criterion`, 가벼운 대안 `divan`, 할당을 추적하는 `dhat`, 핫스팟을 시각화하는 `flamegraph` — 실무에서 쓰는 도구 전체를 다룹니다. 그리고 측정이 끝난 뒤 실제로 무엇을 고칠 수 있는지 — 자동 벡터화와 명시적 SIMD, AoS↔SoA 레이아웃, `#[inline]`/LTO 빌드 튜닝 — 를 어셈블리와 측정값으로 보여줍니다.

이 책의 일관된 태도를 마지막으로 다시 강조합니다. 우리는 "Rust는 빠르다"고 외치지 않습니다. 대신 어셈블리 명령어를 직접 추출해 SIMD 레지스터를 눈으로 확인하고, `criterion`이 토해낸 신뢰구간을 그대로 인용하며, `--emit asm`으로 컴파일러가 정말 벡터화를 했는지 검증합니다. 측정으로 확정한 사실만 적습니다. 이것이 Part 9 "프로덕션으로 가는 길"의 마지막 챕터이자, 이 책이 처음부터 견지해온 "근거로 설득한다"는 원칙의 종착점입니다.

---

## 26.1 The Python Way — 측정의 미덕과 최적화의 천장

Python에서 성능 작업의 첫 단계는 언제나 측정입니다. 추측은 금물입니다. 전형적인 워크플로를 봅시다.

```python
import cProfile
import pstats

def analyze_logs(lines: list[str]) -> dict[str, int]:
    counts: dict[str, int] = {}
    for line in lines:
        parts = line.split(" ")
        if len(parts) < 3:
            continue
        path, status_field = parts[1], parts[2]
        code = status_field.split("=")[1]
        if code.startswith("5"):
            counts[path] = counts.get(path, 0) + 1
    return counts

if __name__ == "__main__":
    lines = ["GET /api/a status=503 12ms"] * 1_000_000
    profiler = cProfile.Profile()
    profiler.enable()
    result = analyze_logs(lines)
    profiler.disable()
    stats = pstats.Stats(profiler).sort_stats("cumulative")
    stats.print_stats(10)
```

`cProfile`은 결정론적 프로파일러(deterministic profiler)입니다. 모든 함수 호출에 후킹해 진입·종료 시각을 기록하므로 정확한 호출 횟수와 누적 시간을 줍니다. 대가는 오버헤드입니다. 후킹 자체가 프로그램을 2~10배 느리게 만들어, 측정값이 실제 운영 성능을 왜곡할 수 있습니다. 함수 호출이 잦은 코드일수록 더 그렇습니다.

그래서 운영 환경에는 **샘플링 프로파일러**인 `py-spy`를 씁니다.

```bash
# 운영 중인 Python 프로세스에 무중단으로 붙어 60초간 샘플링
$ py-spy record -o profile.svg --pid 12345 --duration 60

# 실시간 top 뷰 (어느 함수가 지금 CPU를 먹고 있나)
$ py-spy top --pid 12345
```

`py-spy`는 별도 프로세스에서 일정 주기(기본 100Hz)로 대상의 콜스택을 들여다봅니다. 후킹이 없어 오버헤드가 1% 미만이고, 코드 수정도 재시작도 필요 없습니다. 결과를 플레임그래프(flamegraph)로 뽑으면, 넓은 막대가 곧 시간을 많이 먹는 함수입니다. 한 줄 단위의 정밀 측정이 필요하면 `line_profiler`의 `@profile` 데코레이터를 씁니다.

여기까지는 흠잡을 데 없습니다. 문제는 측정이 가리킨 병목을 고치는 단계입니다. 위 `analyze_logs`를 프로파일링하면 시간의 대부분이 `str.split`, dict 룩업, 그리고 인터프리터의 바이트코드 디스패치 루프 자체에 흩어져 있습니다. 알고리즘은 이미 O(n)으로 최적입니다. 그럼 무엇을 할 수 있을까요?

- **`str.split`을 줄인다**: `parts[2].split("=")[1]`는 매번 새 리스트와 새 문자열 객체들을 할당합니다. 정규식이나 인덱싱으로 줄일 수 있지만, 인터프리터 오버헤드는 그대로입니다.
- **`numpy`/`polars`로 벡터화한다**: 문자열 파싱은 벡터화가 어렵고, 가능하더라도 결국 C/Rust로 짜인 네이티브 코드에 일을 넘기는 것입니다.
- **C 확장이나 PyO3로 도망간다**: 진짜 병목을 들어내는 길. [Chapter 22](./chapter_22.md)에서 본 전략입니다.

핵심은 이것입니다. **순수 Python 코드의 최적화는 "인터프리터 오버헤드를 줄이는" 게임이지 "CPU를 더 잘 쓰는" 게임이 아닙니다.** [Chapter 1](./chapter_01.md)에서 측정한 50M 제곱합 루프의 ~3.0초 중 절대다수는 매 반복마다 `PyObject`를 만들고 타입을 재확인하는 비용이었습니다. 그 루프를 Python 안에서 아무리 다듬어도 CPU가 정말 하고 싶은 `imul` 한 줄로는 결코 내려갈 수 없습니다. 캐시 라인을 의식하거나, SIMD 레지스터를 채우거나, 명령어 수준 병렬성(ILP)을 끌어내는 일은 인터프리터 위에서는 불가능합니다. 이것이 천장입니다.

Rust로 넘어오면 이 천장이 사라집니다. 그런데 천장이 사라진다는 것은 곧 **책임이 우리에게 넘어온다**는 뜻이기도 합니다. 인터프리터가 가려주던 것이 없으니, 우리가 직접 메모리 레이아웃과 명령어 선택을 의식해야 최고 성능이 나옵니다. 그리고 그 모든 결정은 — Python에서 그랬듯 — 측정으로 검증해야 합니다. 다만 이번에는 측정 도구가 훨씬 더 정밀하고, 고칠 수 있는 것이 훨씬 더 깊습니다.

---

## 26.2 The Rust Mental Model — 측정→가설→수정→재측정, 그리고 컴파일러를 의심하라

Rust의 성능 작업은 Python과 같은 과학적 루프를 따릅니다. **측정(measure) → 가설(hypothesize) → 수정(fix) → 재측정(re-measure).** 이 루프 없이 손대는 최적화는 Rust에서도 똑같이 죄악입니다. 다만 Rust 특유의 두 가지 멘탈 모델이 추가됩니다.

첫째, **컴파일러를 신뢰하되 의심하라.** Rust의 최적화 백엔드(LLVM)는 무섭도록 똑똑합니다. 죽은 코드를 지우고(Dead Code Elimination), 상수를 미리 계산하고(Constant Folding), 함수를 인라이닝하고, 루프를 벡터화합니다. 이것은 축복이지만, 벤치마크에서는 저주가 됩니다. "측정하려던 코드를 컴파일러가 통째로 지워버려" 0초가 나오는 것입니다. [Chapter 1](./chapter_01.md)에서 본 `black_box`의 존재 이유가 바로 이것입니다. 우리는 컴파일러가 우리 벤치마크에 무슨 짓을 했는지 의심하고, `--emit asm`으로 실제 생성된 기계어를 확인하는 습관을 들여야 합니다.

둘째, **벤치마크는 통계다.** 한 번의 측정값은 노이즈입니다. CPU 주파수 스케일링, 캐시 상태, OS 스케줄러, 다른 프로세스의 간섭이 매 실행마다 시간을 흔듭니다. Python에서 `timeit`이 여러 번 반복해 최솟값을 취하던 것처럼, Rust에서도 통계적으로 측정해야 합니다. `criterion`은 이것을 자동화합니다 — 워밍업, 다수 샘플 수집, 이상치(outlier) 제거, 신뢰구간 계산, 그리고 이전 실행 대비 회귀 감지까지.

Python의 프로파일링 도구와 Rust의 도구를 한눈에 매핑하면 다음과 같습니다.

| 목적 | Python 도구 | Rust 도구 | 비고 |
|------|-------------|-----------|------|
| 결정론적 함수 프로파일 | `cProfile` / `pstats` | (직접 대응 적음) `perf record`+`cargo flamegraph` | Rust는 샘플링이 표준 |
| 샘플링 프로파일 (무중단) | `py-spy` | `samply`, `cargo flamegraph`, `perf` | 둘 다 콜스택 샘플링 |
| 한 줄 단위 측정 | `line_profiler` | `perf annotate`, `cargo asm` | Rust는 어셈블리 줄 단위 |
| 마이크로벤치마크 | `timeit`, `pytest-benchmark` | `criterion`, `divan` | 통계·회귀 감지 |
| 메모리/할당 프로파일 | `tracemalloc`, `memory_profiler` | `dhat`, `heaptrack`, valgrind massif | 할당 호출 위치 추적 |
| 플레임그래프 시각화 | `py-spy record -o x.svg` | `cargo flamegraph` | 동일 SVG 포맷 |
| 최적화 방벽 | (인터프리터라 불필요) | `std::hint::black_box` | 컴파일러 DCE 방지 |

Python 쪽에 `black_box`가 없다는 점이 의미심장합니다. 인터프리터는 죽은 코드를 지우거나 상수를 폴딩하지 않으므로(매 반복을 곧이곧대로 실행하므로), Python 벤치마크는 컴파일러의 장난을 걱정할 필요가 없습니다. 측정값이 느릴지언정 정직합니다. Rust의 측정값은 더 빠르지만, 컴파일러가 개입할 여지가 크기 때문에 더 조심해야 합니다. "정직한 측정을 위한 방벽"이 Rust 벤치마크의 필수 도구인 이유입니다.

이제 측정 도구부터 하나씩 내부로 들어갑니다.

---

## 26.3 프로파일링 — 핫스팟을 찾는 법

측정의 첫 단계는 "어디가 느린가"입니다. Rust는 네이티브 바이너리를 만들므로, C/C++용 프로파일러를 그대로 쓸 수 있습니다. 가장 보편적인 조합은 Linux의 `perf` + `cargo flamegraph`, macOS·크로스플랫폼의 `samply`입니다.

먼저 프로파일링 가능한 빌드를 만들어야 합니다. 릴리스 최적화는 켜되, 디버그 심볼은 남겨야 콜스택에 함수 이름이 보입니다. `Cargo.toml`에 다음을 추가합니다.

```toml
# 릴리스 성능 + 프로파일링용 심볼을 함께
[profile.release]
debug = true        # 디버그 심볼 포함 (성능에는 영향 없음)
```

> 💡 `debug = true`는 디버그 *정보*만 남길 뿐, 최적화를 끄지 않습니다. `cargo build`(디버그 빌드)와 혼동하지 마세요. 프로파일링은 **반드시 `--release`**로 합니다. 디버그 빌드를 프로파일링하면 최적화되지 않은 코드의 병목을 보게 되어, 릴리스에서는 존재하지도 않는 함수를 잡고 씨름하게 됩니다(26.10 함정 참고).

프로파일링 명령은 다음과 같습니다.

```bash
# 1) cargo-flamegraph: 한 번에 프로파일 + SVG 생성 (perf 래퍼)
$ cargo install flamegraph
$ cargo flamegraph --release --bin perf_demo
#   → flamegraph.svg 생성. 넓은 막대 = 시간 많이 먹는 함수.

# 2) samply: 크로스플랫폼 샘플링 프로파일러 (Firefox Profiler UI)
$ cargo install samply
$ cargo build --release
$ samply record ./target/release/perf_demo

# 3) perf 직접 사용 (Linux)
$ perf record -g ./target/release/perf_demo
$ perf report          # 인터랙티브 콜스택 뷰
```

플레임그래프를 읽는 법은 `py-spy`와 동일합니다. **세로축은 콜스택 깊이, 가로축은 시간 비율(알파벳 순 정렬, 호출 순서 아님).** 가장 넓은 막대가 가장 많은 샘플이 찍힌 — 즉 가장 오래 머문 — 함수입니다. 그 막대 위에 또 다른 넓은 막대가 있으면 그 함수가 호출한 하위 함수가 진짜 병목입니다.

`cargo flamegraph`가 만드는 SVG는 `py-spy record`의 SVG와 같은 Brendan Gregg 포맷이라, Python에서 플레임그래프를 읽어본 사람이라면 추가 학습이 필요 없습니다. 차이는 막대에 찍히는 이름입니다. Python 플레임그래프는 인터프리터 프레임(`_PyEval_EvalFrameDefault` 같은)이 곳곳에 끼지만, Rust 플레임그래프는 여러분이 짠 함수가 직접 보입니다. 단일형화(Monomorphization, [Chapter 1](./chapter_01.md))된 제네릭 함수는 구체 타입이 박힌 긴 심볼 이름(`core::iter::...::sum`)으로 나타나는데, 처음엔 낯설지만 곧 익숙해집니다.

> 🚨 **No Free Lunch — perf 권한과 환경**
>
> Linux에서 `perf`는 커널의 성능 카운터에 접근하므로 권한이 필요합니다. `perf record`가 `permission denied`를 내면 `sudo sysctl kernel.perf_event_paranoid=1`(또는 `-1`)로 완화하거나, `sudo`로 실행해야 합니다. 컨테이너(Docker) 안에서는 `--cap-add SYS_ADMIN` 또는 `--privileged`가 필요할 수 있고, 클라우드 VM에서는 아예 PMU(성능 모니터링 유닛)가 노출되지 않아 하드웨어 카운터를 못 쓰는 경우도 있습니다. macOS에서는 `perf`가 없으므로 `samply`나 Instruments.app(`cargo instruments`)을 씁니다. 프로파일링 환경을 구성하는 이 마찰은 Rust의 잘못이 아니라 OS 수준 도구를 직접 쓰는 대가입니다 — `py-spy`가 이것을 잘 추상화해줬던 것이 새삼 고맙게 느껴질 수 있습니다.

프로파일러가 병목을 가리키면, 다음 단계는 그 병목을 정밀하게, 그리고 통계적으로 측정하는 것입니다. 여기서 `criterion`이 등장합니다.

---

## 26.4 벤치마크 방법론 정본화 — criterion, black_box, divan

이 절이 이 챕터의 심장입니다. 이 책의 모든 챕터에서 벤치마크 코드를 봤지만, 그 방법론을 한곳에 모아 정리하지 않았습니다. 여기서 끝냅니다.

### 왜 `black_box`인가 — 한 번 더, 끝까지

[Chapter 1](./chapter_01.md)에서 `black_box` 없는 루프가 0초가 나온다는 것을 봤습니다. 여기서 그 메커니즘을 마지막으로 못 박습니다. 다음은 직접 측정한 결과입니다(rustc 1.96.0, `-O`, Apple Silicon).

```rust
use std::time::Instant;
fn main() {
    let n: u64 = 1_000_000_000;        // 10억 회
    let start = Instant::now();
    let mut total: u64 = 0;
    for i in 0..n {
        total = total.wrapping_add(i.wrapping_mul(i));
    }
    println!("{:.6}s (total={})", start.elapsed().as_secs_f64(), total);
}
```

```text
$ rustc -O noblackbox.rs && ./noblackbox
0.000000s (total=3338615082255021824)
```

10억 번 루프가 **0.000000초**입니다. `total`을 출력까지 했는데도요. LLVM은 이 합이 입력 `n`에만 의존하는 순수 함수임을 알고, 닫힌 형태(closed-form) 공식으로 컴파일 타임에 결과를 계산해버립니다. 런타임에 루프는 **존재하지 않습니다.** 이런 측정으로 "Rust는 Python보다 10억 배 빠르다"고 말하는 것은 거짓말입니다.

`std::hint::black_box`는 이 장난을 막는 **최적화 방벽(optimization barrier)**입니다.

```rust
use std::hint::black_box;
// total += black_box(i) * black_box(i);
```

`black_box(x)`는 "이 값은 외부에서 관찰될 수 있으니 미리 계산하거나 지우지 마라"는, 컴파일러에게 보내는 불투명한 신호입니다. 정확히는 컴파일러가 인자의 값을 가정하지 못하게 막아, 그것에 의존하는 계산을 실제로 수행하는 코드를 남기게 합니다. 이 방벽을 켜면 같은 머신에서 50M 루프(`u64`)가 약 **0.04초**로 측정됩니다. 이것이 정직한 수치입니다.

`black_box`는 입력과 출력 양쪽에 쓸 수 있습니다.

```rust
// 입력을 방벽으로 감싸 상수 폴딩 방지
let result = my_function(black_box(input));
// 출력을 방벽으로 감싸 DCE 방지 (결과가 안 쓰이면 함수째 지워질 수 있음)
black_box(result);
```

### criterion — 통계적 측정의 표준

`black_box`를 손으로 끼우고 `Instant`로 재는 것은 마이크로벤치마크의 최소 요건일 뿐입니다. 실무에서는 `criterion`을 씁니다. `criterion`은 워밍업, 다수 샘플 수집, 이상치 제거, 신뢰구간 추정, 회귀 감지를 모두 자동화하는 사실상의 표준 벤치마크 하니스입니다(Python의 `pytest-benchmark`에 대응하되 훨씬 정교합니다).

설정은 `Cargo.toml`과 `benches/` 디렉터리로 합니다.

```toml
[dev-dependencies]
criterion = "0.8.2"

[[bench]]
name = "agg_bench"
harness = false        # criterion이 자체 main을 제공하므로 기본 하니스 끔
```

벤치마크 대상은 [Chapter 14](./chapter_14.md)·[Chapter 15](./chapter_15.md)의 로그 분석기, [Chapter 22](./chapter_22.md)의 집계 서비스를 이어받습니다. "status=5xx 라인을 골라 경로별로 집계"하는 핫패스의 두 버전 — Python 직역 스타일(줄마다 `String` 할당)과 제로카피(zero-copy) 슬라이스 버전 — 을 비교합니다.

```rust
use std::collections::HashMap;

// 느린 버전: 줄마다 String 할당 + split (Python 습관의 직역)
pub fn aggregate_5xx_slow(lines: &[String]) -> HashMap<String, u64> {
    let mut counts: HashMap<String, u64> = HashMap::new();
    for line in lines {
        let parts: Vec<String> = line.split(' ').map(|s| s.to_string()).collect();
        if parts.len() < 3 {
            continue;
        }
        let path = parts[1].clone();
        if let Some(eq) = parts[2].find('=') {
            if parts[2][eq + 1..].starts_with('5') {
                *counts.entry(path).or_insert(0) += 1;
            }
        }
    }
    counts
}

// 빠른 버전: 제로카피 슬라이스 파싱 + 키만 소유
pub fn aggregate_5xx_fast(lines: &[String]) -> HashMap<String, u64> {
    let mut counts: HashMap<String, u64> = HashMap::new();
    for line in lines {
        let mut it = line.split(' ');
        let _method = it.next();
        let (Some(path), Some(status)) = (it.next(), it.next()) else {
            continue;
        };
        if let Some(code) = status.strip_prefix("status=") {
            if code.as_bytes().first() == Some(&b'5') {
                *counts.entry(path.to_string()).or_insert(0) += 1;
            }
        }
    }
    counts
}
```

차이는 단 하나, "무엇을 할당하는가"입니다. `slow`는 줄마다 `Vec<String>`을 만들고 각 토큰을 새 `String`으로 복사합니다 — 한 줄에 4~5번의 힙 할당입니다. `fast`는 `split`이 반환하는 `&str` 슬라이스(원본을 가리키는 16바이트 fat pointer, [Chapter 1](./chapter_01.md))를 그대로 쓰고, 카운트가 필요할 때 경로 키 하나만 소유합니다. [Chapter 15](./chapter_15.md)에서 배운 "이터레이터는 게으르고 제로카피"라는 원칙의 실전입니다.

`criterion` 벤치마크 파일은 이렇게 작성합니다.

```rust
use criterion::{criterion_group, criterion_main, BenchmarkId, Criterion};
use perf_demo::{aggregate_5xx_fast, aggregate_5xx_slow, make_lines};
use std::hint::black_box;

fn bench_aggregate(c: &mut Criterion) {
    let lines = make_lines(100_000);    // 합성 로그 10만 줄
    let mut group = c.benchmark_group("aggregate_5xx");

    group.bench_function(BenchmarkId::new("slow", "100k"), |b| {
        b.iter(|| aggregate_5xx_slow(black_box(&lines)));
    });
    group.bench_function(BenchmarkId::new("fast", "100k"), |b| {
        b.iter(|| aggregate_5xx_fast(black_box(&lines)));
    });

    group.finish();
}

criterion_group!(benches, bench_aggregate);
criterion_main!(benches);
```

`b.iter(|| ...)`가 핵심입니다. `criterion`은 이 클로저를 충분히 여러 번 실행해 통계적으로 안정된 측정값을 얻고, 입력은 `black_box`로 감싸 상수 폴딩을 막습니다. `cargo bench`로 돌린 실제 출력입니다.

```text
$ cargo bench --bench agg_bench
aggregate_5xx/slow/100k  time:   [12.866 ms 13.093 ms 13.343 ms]
aggregate_5xx/fast/100k  time:   [2.6827 ms 2.7236 ms 2.7672 ms]
```

세 숫자는 `[하한 추정 중앙값 상한]`으로, 95% 신뢰구간입니다. `slow`는 약 13.1ms, `fast`는 약 2.72ms — **약 4.8배** 차이입니다. 이 배수는 "Rust가 Python보다"가 아니라 "같은 Rust 안에서 할당을 없앴더니"의 차이라는 점이 중요합니다. 마이크로 최적화의 효과를 정직하게 측정한 값입니다.

`criterion`의 진짜 가치는 **회귀 감지**입니다. 한 번 실행하면 결과를 `target/criterion/`에 저장하고, 다음 실행에서 자동으로 비교합니다.

```text
aggregate_5xx/fast/100k  time:   [2.71 ms 2.74 ms 2.78 ms]
                         change: [-1.2% +0.3% +1.9%] (p = 0.71 > 0.05)
                         No change in performance detected.
```

`change`가 직전 실행 대비 변화율이고, `p` 값으로 통계적 유의성을 판단합니다. 코드를 고친 뒤 `Performance has regressed`가 뜨면, CI 단계에서 성능 회귀를 잡을 수 있습니다. Python의 `pytest-benchmark`도 비슷한 기능이 있지만, `criterion`의 통계 모델(부트스트랩 신뢰구간, Tukey 이상치 분류)이 더 견고합니다.

### divan — 가벼운 대안

`criterion`은 강력하지만 의존성이 무겁고(통계·플로팅 라이브러리 다수) 보일러플레이트가 있습니다. 더 가볍고 선언적인 대안이 `divan`입니다. 같은 벤치마크를 `divan`으로 쓰면 이렇습니다.

```rust
use perf_demo::{aggregate_5xx_fast, aggregate_5xx_slow, make_lines};

fn main() {
    divan::main();
}

#[divan::bench]
fn slow(bencher: divan::Bencher) {
    let lines = make_lines(100_000);
    bencher.bench_local(|| aggregate_5xx_slow(divan::black_box(&lines)));
}

#[divan::bench]
fn fast(bencher: divan::Bencher) {
    let lines = make_lines(100_000);
    bencher.bench_local(|| aggregate_5xx_fast(divan::black_box(&lines)));
}
```

`#[divan::bench]` 어트리뷰트가 함수를 벤치마크로 등록합니다 — `pytest`가 `test_` 함수를 자동 수집하는 것과 같은 느낌입니다. `divan`도 자체 `black_box`를 제공합니다. 실제 출력입니다.

```text
$ cargo bench --bench divan_bench
Timer precision: 41 ns
divan_bench  fastest    │ slowest    │ median     │ mean       │ samples
├─ fast      2.346 ms   │ 3.043 ms   │ 2.493 ms   │ 2.526 ms   │ 100
╰─ slow      11.04 ms   │ 12.56 ms   │ 11.4 ms    │ 11.47 ms   │ 100
```

`divan`은 중앙값 2.49ms(fast) vs 11.4ms(slow), 약 **4.6배** — `criterion`과 독립적으로 같은 결론을 줍니다. 두 도구가 일치한다는 것은 측정이 견고하다는 강한 증거입니다. 선택 기준은 단순합니다. **회귀 감지·상세 통계·플롯이 필요하면 `criterion`, 빠르고 가볍게 비교만 하면 `divan`.** 둘 다 `black_box`를 강제한다는 공통점이 핵심입니다.

> 💡 **이 책의 모든 벤치마크가 `black_box`를 쓴 이유, 여기서 수렴합니다.** [Chapter 1](./chapter_01.md)의 제곱합 루프, [Chapter 15](./chapter_15.md)의 이터레이터 파이프라인, [Chapter 16](./chapter_16.md)의 멀티스레드 측정 — 전부 `black_box`(또는 `criterion`)를 거쳤습니다. 컴파일러가 측정 대상을 지우거나 미리 계산하면 수치가 거짓이 되기 때문입니다. 앞으로 어떤 Rust 벤치마크를 보든, `black_box`가 없으면 그 수치를 의심하십시오. 그것이 이 책이 가르친 측정 정직성의 마지막 원칙입니다.

---

## 26.5 Under the Hood — 자동 벡터화(Auto-vectorization) 확인하기

측정으로 병목을 찾았고 통계적으로 확정했습니다. 이제 무엇을 고칠 수 있을까요? Python에서는 막혔던 영역, **CPU의 SIMD(Single Instruction Multiple Data) 유닛**을 쓰는 것부터 봅니다.

현대 CPU는 한 명령어로 여러 데이터를 동시에 처리하는 벡터 레지스터를 갖습니다. x86-64의 SSE/AVX(128/256/512비트), ARM의 NEON(128비트)이 그것입니다. 128비트 레지스터 하나에 `f32`(4바이트) 4개가 들어가므로, `fmul.4s` 같은 한 명령어가 곱셈 4개를 동시에 수행합니다. Python의 `numpy`가 빠른 이유의 큰 부분이 바로 이것입니다 — numpy는 C/Fortran으로 짜인 SIMD 최적화 커널에 일을 넘깁니다. 인터프리터 위의 순수 Python 루프는 SIMD를 결코 쓸 수 없습니다.

Rust에서는 **컴파일러가 적절한 루프를 자동으로 벡터화**합니다. [Chapter 15](./chapter_15.md)에서 본 이터레이터 체인이 그 대표적 대상입니다. 두 함수를 봅시다.

```rust
#[inline(never)]
pub fn sum_f32(data: &[f32]) -> f32 {
    data.iter().sum()
}

#[inline(never)]
pub fn dot_product(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}
```

"정말 벡터화됐을까?"를 믿지 말고 어셈블리를 직접 추출해 확인합니다. `--emit asm`은 rustc 내장 기능입니다.

```bash
$ rustc -O --crate-type=lib --emit asm vec.rs -o vec.s
$ grep -E "fmul|ldp\s+q" vec.s
```

추출된 `dot_product`의 핵심 본문입니다(Apple Silicon / AArch64).

```text
    ldp     q1, q2, [x9, #-32]      ; 128비트 레지스터 2개 로드 (f32 8개)
    ldp     q17, q18, [x9], #64     ; 다음 8개
    ...
    fmul.4s v1, v1, v5             ; f32 4개를 한 번에 곱셈 (SIMD!)
    fmul.4s v2, v2, v6             ; 또 4개
    fmul.4s v3, v3, v7
    fmul.4s v4, v4, v16
```

`q1`, `q2`는 128비트 SIMD 레지스터(`q` = quadword)이고, `fmul.4s`는 "4개의 단정밀도(single) `f32`를 동시에 곱하는" NEON 명령어입니다. 게다가 컴파일러는 **4-way 언롤링**까지 했습니다 — `v1`~`v4` 네 개의 누산기를 병렬로 굴려 명령어 수준 병렬성(ILP)을 끌어냅니다. 즉 이 루프는 한 사이클에 `f32` 곱셈 16개(4 레인 × 4 누산기)를 향해 달립니다. 우리가 짠 것은 평범한 `.iter().zip().map().sum()` 한 줄인데, 손으로 SIMD를 짠 것과 같은 기계어가 나온 것입니다. 이것이 [Chapter 1](./chapter_01.md)에서 말한 **무비용 추상화(Zero-cost Abstraction)**의 극단적 사례입니다.

> 💡 **Under the Hood — 왜 어떤 루프는 벡터화되고 어떤 루프는 안 되는가**
>
> 자동 벡터화에는 조건이 있습니다. 각 반복이 **서로 독립적**이어야 합니다. 반복 `i`의 결과가 반복 `i+1`의 입력이 되는 직렬 의존(serial dependency)이 있으면 벡터화가 불가능합니다. 다음 루프를 어셈블리로 추출해 비교했습니다.
>
> ```rust
> #[inline(never)]
> pub fn serial_dep(data: &[u32]) -> u32 {
>     let mut acc: u32 = 1;
>     for &x in data {
>         acc = acc.wrapping_mul(x).wrapping_add(acc >> 1); // acc가 다음 입력
>     }
>     acc
> }
> ```
>
> 측정 결과: `serial_dep`의 어셈블리에는 벡터 레지스터(`.4s`)가 **0개**, `sum_f32`에는 벡터 명령(`.4s`)이 **5개** 나타났습니다. `acc`가 매 단계 이전 결과에 의존하므로 컴파일러가 병렬화할 수 없는 것입니다.
>
> 한 가지 더 미묘한 점: `f32` 덧셈은 결합법칙이 깨집니다(`(a+b)+c ≠ a+(b+c)`, 반올림 오차 때문). 그래서 컴파일러는 기본적으로 부동소수점 합산의 순서를 함부로 바꾸지 못합니다. 그럼에도 위에서 `sum_f32`가 벡터화된 것은, LLVM이 "여러 누산기로 부분합을 만들고 마지막에 합치는" 변형이 충분히 안전하다고 판단하는 휴리스틱을 갖기 때문입니다. 더 공격적인 재정렬이 필요하면 `-C target-feature` 또는 명시적 SIMD를 써야 합니다. **자동 벡터화는 강력하지만 마법이 아닙니다. 의존성을 끊고, 컴파일러가 분석하기 쉬운 단순한 루프를 짜는 것이 우리가 할 일입니다.**

자동 벡터화를 유도하는 실무 체크리스트입니다.

- **슬라이스를 직접 순회**하라(`for &x in slice` 또는 `.iter()`). 인덱스 접근(`slice[i]`)은 경계 검사(bounds check)가 끼어 벡터화를 방해할 수 있습니다.
- **루프 안에서 분기·조기 반환을 피하라.** 단순한 산술 루프가 가장 잘 벡터화됩니다.
- **`-C target-cpu=native`**로 빌드하면 현재 CPU의 최신 SIMD(AVX2/AVX-512)를 쓸 수 있습니다(단, 배포 대상 CPU가 그 명령어를 지원해야 합니다).
- 확신이 안 서면 `--emit asm`이나 [Compiler Explorer](https://godbolt.org)로 **직접 확인**하라.

---

## 26.6 명시적 SIMD — `std::simd`(portable SIMD)

자동 벡터화가 실패하거나 더 강한 제어가 필요할 때, Rust는 **명시적 SIMD**를 제공합니다. 두 갈래가 있습니다. 아키텍처별 인트린식(`std::arch::x86_64`, `std::arch::aarch64` — `unsafe`, [Chapter 21](./chapter_21.md))과, 이식 가능한(portable) `std::simd`입니다. 후자는 한 번 짜면 x86이든 ARM이든 적절한 SIMD로 컴파일되는 추상화로, numpy의 "플랫폼 무관 벡터 연산"과 철학이 같습니다.

`std::simd`는 2026년 현재 nightly 전용입니다(`#![feature(portable_simd)]`). 다음은 점곱(dot product)을 명시적 SIMD로 짠, **nightly에서 컴파일·실행 검증한** 코드입니다.

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;
use std::simd::f32x8;

// 8개의 f32를 한 레인(lane)에 묶어 처리
pub fn dot_simd(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());
    let mut acc = f32x8::splat(0.0);   // 8레인 누산기를 0으로
    let chunks = a.len() / 8;

    for i in 0..chunks {
        let va = f32x8::from_slice(&a[i * 8..]);
        let vb = f32x8::from_slice(&b[i * 8..]);
        acc += va * vb;                // 8개 곱셈+누산을 한 번에
    }

    let mut total = acc.reduce_sum();  // 벡터 누산기를 가로 합으로 접기

    // 8의 배수가 아닌 꼬리(tail) 처리
    for i in (chunks * 8)..a.len() {
        total += a[i] * b[i];
    }
    total
}
```

검증을 위해 스칼라 버전과 결과를 대조했습니다.

```text
$ rustup run nightly rustc -O simd_demo.rs && ./simd_demo
dot_simd = 499167
scalar   = 499167
```

명시적 SIMD와 평범한 `.iter().zip().map().sum()`이 동일한 값(499167)을 냅니다. 코드 구조를 뜯어봅시다.

- **`f32x8`**: 8개의 `f32`를 담는 SIMD 벡터 타입. 128비트 레지스터 2개 또는 256비트(AVX) 1개에 매핑됩니다.
- **`splat(0.0)`**: 8레인 전부를 같은 값으로 채웁니다(numpy의 broadcast).
- **`va * vb`**: 8개 곱셈을 한 명령어로(레인별 곱셈).
- **`reduce_sum()`**: 8개 부분합을 하나로 접는 가로 합(horizontal reduction).
- **꼬리 처리**: 데이터 길이가 8의 배수가 아니면 남는 원소를 스칼라로 처리. SIMD 코드의 영원한 숙제입니다.

> 💡 **Under the Hood — numpy의 벡터화 vs Rust의 SIMD**
>
> numpy에서 `np.dot(a, b)`를 호출하면, Python 인터프리터는 즉시 C로 짜인 BLAS 커널(OpenBLAS, MKL 등)로 제어를 넘깁니다. 그 커널은 손으로 튜닝한 SIMD 어셈블리이고, Python은 그저 "큰 배열 두 개의 포인터를 넘기고 결과를 기다리는" 호출자(orchestrator)일 뿐입니다. 그래서 numpy는 빠르지만, **그 빠름은 numpy 작성자의 것이지 여러분의 것이 아닙니다.** numpy가 제공하지 않는 연산(예: 도메인 특화된 커스텀 집계)을 SIMD로 가속하려면, 여러분은 다시 C/Cython으로 내려가거나 PyO3로 Rust를 불러야 합니다.
>
> Rust의 `std::simd`는 그 SIMD 제어를 **여러분의 코드 안으로** 가져옵니다. 임의의 커스텀 루프를 이식 가능한 SIMD로 짤 수 있고, 컴파일러가 타깃 아키텍처에 맞는 명령어를 골라줍니다. 비용 모델도 정직합니다 — `f32x8`은 어셈블리로 정확히 벡터 명령 몇 개로 컴파일되며, 숨은 할당이나 디스패치가 없습니다. 다만 트레이드오프가 있습니다. (1) nightly 의존성, (2) 꼬리 처리·정렬(alignment) 같은 저수준 세부를 직접 다뤄야 함, (3) **대부분의 경우 자동 벡터화로 충분**하다는 것. 명시적 SIMD는 자동 벡터화가 명백히 실패했고, 프로파일러가 그 루프를 가리켰고, 어셈블리로 확인까지 한 뒤에 꺼내는 마지막 카드입니다. 안정 버전이 필요하면 `wide`나 `pulp` 같은 크레이트가 stable에서 portable SIMD를 제공합니다.

명시적 SIMD를 stable에서 쓰는 가장 실용적인 길은, 직접 인트린식을 짜는 대신 검증된 크레이트에 기대는 것입니다. 행렬·신호처리는 `nalgebra`/`ndarray`, 범용 SIMD 추상은 `wide`/`pulp`가 있습니다. "내가 SIMD를 직접 짜야 하나?"라는 질문에 대한 답은 보통 "아니오, 먼저 자동 벡터화를 유도하고, 안 되면 크레이트를 찾아라"입니다.

---

## 26.7 메모리와 캐시 — 할당 프로파일, SoA vs AoS, 빌드 튜닝

CPU 속도와 메모리 속도의 격차는 갈수록 벌어지고 있습니다. [Chapter 1](./chapter_01.md)에서 본 그 숫자 — L1 캐시 ~1ns, 메인 메모리 ~100ns — 가 현대 성능 작업의 중심입니다. 알고리즘이 같다면, **승부는 캐시에서 갈립니다.**

### 할당 프로파일 — dhat

먼저 "어디서 얼마나 할당하는가"를 측정합니다. Python의 `tracemalloc`에 대응하는 도구가 `dhat`입니다. 전역 할당자(global allocator)를 교체해 모든 힙 할당을 추적합니다.

```rust
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn build_strings(n: usize) -> Vec<String> {
    (0..n).map(|i| format!("item-{}", i)).collect()
}

fn main() {
    // 이 핸들이 살아있는 동안 할당이 기록되고, drop 시 dhat-heap.json을 쓴다.
    let _profiler = dhat::Profiler::new_heap();
    let v = build_strings(10_000);
    println!("built {} strings", v.len());
}
```

실행 결과입니다.

```text
$ cargo run --release
built 10000 strings
dhat: Total:     341,088 bytes in 10,003 blocks
dhat: At t-gmax: 341,088 bytes in 10,003 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

`10,003 blocks` = 10,000개의 `String` + `Vec`의 재할당들. 총 341,088바이트를 할당했습니다. `dhat`은 `dhat-heap.json`도 생성하는데, [DHAT 뷰어](https://nnethercote.github.io/dh_view/dh_view.html)에 올리면 "어느 호출 위치(call site)가 얼마나 할당했는가"를 트리로 봅니다. 26.4의 `aggregate_5xx_slow`를 이걸로 프로파일하면 `String` 할당이 압도적임이 한눈에 드러납니다 — 그래서 `fast` 버전이 4.8배 빨랐던 것입니다. **할당 프로파일은 마이크로 최적화의 나침반입니다.** 더 무거운 도구로는 `heaptrack`(Linux), valgrind massif가 있습니다.

### 캐시 친화적 레이아웃 — SoA vs AoS

데이터 구조의 메모리 배치가 캐시 효율을 좌우합니다. 입자 100만 개를 다루는 시뮬레이션을 생각해봅시다. 자연스러운 설계는 **AoS(Array of Structs)** — 구조체의 배열입니다.

```rust
#[derive(Clone, Copy)]
struct ParticleAoS {
    x: f32, y: f32, z: f32,   // 위치
    mass: f32, charge: f32,    // 물리량
    flags: u32,
}
// size_of::<ParticleAoS>() == 24바이트 (측정 확인)
let particles: Vec<ParticleAoS> = /* 100만 개 */;
```

메모리에는 `[x0 y0 z0 m0 c0 f0][x1 y1 z1 m1 c1 f1]...`처럼 한 입자의 모든 필드가 뭉쳐 놓입니다. 그런데 "모든 입자의 x좌표 평균"만 계산한다면? CPU는 캐시 라인(64바이트) 단위로 메모리를 읽는데, `x0`를 읽으면 그 라인에 `y0 z0 m0 c0 f0`와 다음 입자 일부가 따라옵니다. 우리가 원한 건 `x`뿐인데, 캐시 라인의 5/6이 쓸데없는 데이터로 채워집니다. **캐시 대역폭이 낭비됩니다.**

**SoA(Struct of Arrays)** — 배열의 구조체 — 는 필드별로 배열을 분리합니다.

```rust
struct ParticlesSoA {
    x: Vec<f32>, y: Vec<f32>, z: Vec<f32>,
    mass: Vec<f32>, charge: Vec<f32>,
    flags: Vec<u32>,
}
```

이제 `x`들이 `[x0 x1 x2 x3 ...]`로 연속됩니다. x좌표만 순회하면 캐시 라인 64바이트에 `f32` 16개가 빽빽이 들어차고, 한 번의 메모리 페치로 16개를 처리합니다. 게다가 연속된 동일 타입 배열이라 26.5에서 본 자동 벡터화가 더 잘 걸립니다. 두 레이아웃의 x좌표 평균(centroid)을 측정했습니다.

```rust
fn centroid_x_aos(ps: &[ParticleAoS]) -> f32 {
    ps.iter().map(|p| p.x).sum::<f32>() / ps.len() as f32
}
fn centroid_x_soa(ps: &ParticlesSoA) -> f32 {
    ps.x.iter().sum::<f32>() / ps.x.len() as f32
}
```

두 함수는 같은 값(`499940.38`, 100만 입자)을 내지만, SoA 버전이 캐시 효율과 벡터화 양쪽에서 유리합니다. [Chapter 1](./chapter_01.md)에서 "Python 리스트는 포인터 배열이라 pointer chasing으로 캐시 미스를 일으킨다"고 했던 그 문제의 Rust 버전이 바로 AoS↔SoA 선택입니다. 차이는 Python에서는 선택지가 없었지만(모든 것이 포인터), Rust에서는 우리가 레이아웃을 통제한다는 것입니다.

> 💡 **Under the Hood — SoA는 게임 엔진과 데이터프레임의 비밀**
>
> SoA가 추상적으로 들린다면, 여러분이 매일 쓰는 도구를 떠올리면 됩니다. `polars`와 `pandas`(내부 numpy)의 **컬럼 지향(columnar) 저장**이 정확히 SoA입니다. "한 행의 모든 컬럼"이 아니라 "한 컬럼의 모든 행"을 연속으로 저장하기에, `df["price"].mean()` 같은 컬럼 집계가 캐시·SIMD 친화적으로 빠른 것입니다. 게임 엔진의 ECS(Entity Component System)도 같은 이유로 컴포넌트를 SoA로 배치합니다. AoS는 코드가 직관적이지만(`particle.x`), SoA는 성능이 좋습니다. 실무 지침: **객체 단위로 접근하면 AoS, 필드 단위로 대량 순회하면 SoA.** Rust에는 이 변환을 매크로로 자동화하는 `soa_derive` 크레이트도 있습니다.

### 빌드 튜닝 — #[inline], LTO, codegen-units

마지막으로, 같은 소스라도 빌드 설정이 성능을 바꿉니다. 세 가지 손잡이가 핵심입니다([부록 A](../appendices/appendix_a_cargo.md)의 프로파일 튜닝과 연계).

- **`#[inline]`**: 함수 호출을 호출 지점에 펼쳐 호출 오버헤드를 없애고, 인라인된 코드에 추가 최적화(상수 전파, 벡터화)를 가능하게 합니다. 작고 자주 불리는 함수에 유효합니다. 단, **크레이트 경계를 넘는 인라이닝**은 기본적으로 안 되므로, 라이브러리의 공개 핫함수에 `#[inline]`을 붙여야 사용자 크레이트에서 인라인됩니다. 남용하면 코드 비대화(code bloat)와 컴파일 시간 증가를 부릅니다.
- **LTO(Link-Time Optimization)**: `lto = "fat"`은 링크 시점에 모든 크레이트를 통째로 다시 최적화해, 크레이트 경계를 넘는 인라이닝과 죽은 코드 제거를 가능하게 합니다. 런타임 성능을 보통 수 %~십수 % 올리지만, 빌드 시간이 크게 늘어납니다. `"thin"`은 그 절충안입니다.
- **`codegen-units`**: rustc는 컴파일을 병렬화하려고 크레이트를 여러 코드 생성 단위로 쪼갭니다(기본 16). 단위가 많으면 빌드가 빠르지만 단위 간 최적화 기회를 놓칩니다. `codegen-units = 1`은 전체를 한 단위로 최적화해 성능을 짜내되 빌드를 느리게 합니다.

프로덕션 핫패스용 최대 속도 프로파일은 [부록 A](../appendices/appendix_a_cargo.md)에서 본 그대로입니다.

```toml
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
```

> 🚨 **No Free Lunch — 빌드 튜닝은 측정 후에**
>
> `lto`와 `codegen-units = 1`은 공짜가 아닙니다. 빌드 시간이 2~5배 늘 수 있습니다. 개발 중 매번 이걸 켜면 컴파일-테스트 루프가 고통스러워집니다. 그래서 실무에서는 평소 개발은 기본 `release`(또는 `dev`)로, 최종 배포 빌드와 벤치마크에서만 `lto`/`codegen-units=1`을 켭니다. 그리고 이 모든 설정의 효과는 — 이 챕터의 다른 모든 것처럼 — **반드시 `criterion`으로 전후를 측정해 확인**해야 합니다. "LTO를 켰으니 빨라졌겠지"는 추측이고, 추측은 죄악입니다. 어떤 워크로드에서는 LTO 효과가 측정 노이즈 안에 묻혀 빌드 시간만 손해 보기도 합니다.

---

## 26.8 실전 Hands-on — 로그 집계기의 핫패스 최적화 전체 사이클

지금까지의 도구를 하나의 시나리오로 묶습니다. [Chapter 14](./chapter_14.md)·[Chapter 15](./chapter_15.md)의 로그 분석기, [Chapter 22](./chapter_22.md)의 집계 서비스를 이어받아, **측정→가설→수정→재측정**의 완전한 사이클을 돕니다.

### 1단계 — 측정: 어디가 느린가

10만 줄 합성 로그에 대해 두 버전을 `criterion`으로 측정한 결과(26.4 재인용)입니다.

```text
aggregate_5xx/slow/100k  time:   [12.866 ms 13.093 ms 13.343 ms]
aggregate_5xx/fast/100k  time:   [2.6827 ms 2.7236 ms 2.7672 ms]
```

`divan`으로 교차 검증도 했습니다(중앙값 slow 11.4ms / fast 2.49ms). 두 도구 모두 `slow`가 4.5~4.8배 느리다고 말합니다.

### 2단계 — 가설: 할당이 범인이다

`slow`를 `dhat`으로 프로파일하면 줄마다 `Vec<String>` + 토큰별 `String` 할당이 폭증합니다. 26.7에서 본 10k 문자열 = 341KB 할당의 패턴이 줄 수만큼 곱해집니다. **가설: 힙 할당이 병목이다.** 알고리즘(O(n) 한 번 순회)은 동일하니, 차이는 순전히 할당량입니다.

### 3단계 — 수정: 제로카피와 레이아웃

`fast` 버전은 두 가지를 바꿨습니다. (1) `split`이 주는 `&str` 슬라이스를 복사 없이 쓰고(제로카피), (2) `String::strip_prefix`와 바이트 비교로 추가 할당을 없앴습니다. 카운트 키만 `to_string()`으로 소유합니다. 한 줄당 할당이 4~5번에서 (5xx일 때만) 1번 이하로 줄었습니다.

만약 경로의 종류가 한정적(고정된 엔드포인트 집합)이라면 한 걸음 더 나아갈 수 있습니다. `HashMap<String, u64>` 대신 키를 인덱스로 매핑하거나, 키를 `&str`로 두고 수명([Chapter 6](./chapter_06.md))으로 원본을 빌려 카운트만 따로 모으는 식입니다. 입력 전체가 한 버퍼에 있다면 [Chapter 20](./chapter_20.md)에서 언급한 아레나 할당(`bumpalo` 등)으로 중간 객체들을 한 번에 할당·해제해 할당자 호출 자체를 줄일 수도 있습니다. 다만 이런 최적화는 코드 복잡도를 올리므로, **프로파일러가 그 지점을 다시 가리킬 때만** 적용합니다.

### 4단계 — 재측정: 효과를 숫자로 확인

수정 후 `criterion`을 다시 돌리면 `fast`가 약 2.72ms로, `slow` 대비 4.8배 개선이 확정됩니다. 그리고 `criterion`의 회귀 감지가 이 개선을 자동으로 기록하므로, 이후 누군가 실수로 할당을 다시 들여오면 CI에서 `Performance has regressed`로 잡힙니다.

이 사이클의 교훈은 Python 시절과 똑같습니다. **추측하지 말고 측정하라.** 다른 점은 두 가지입니다. 첫째, Rust에서는 인터프리터 천장이 없으므로 할당·캐시·SIMD 같은 더 깊은 곳을 고칠 수 있습니다. 둘째, `criterion`/`dhat`/`--emit asm`이라는 정밀 도구로 그 효과를 바이트와 명령어 수준에서 검증할 수 있습니다.

마지막으로 정직하게 짚습니다. 이 4.8배는 "Rust vs Python"이 아니라 "Rust 안에서 잘 짠 것 vs 못 짠 것"입니다. 이 `fast` 집계기를 [Chapter 22](./chapter_22.md)의 PyO3로 감싸 Python에서 호출하면, 순수 Python 구현 대비로는 훨씬 큰 배수(보통 10~50배, 입력이 크고 CPU 바운드일수록 더)가 나옵니다 — 인터프리터 오버헤드까지 함께 걷어내기 때문입니다. 도구별로 정직한 배수를 구분해 보고하는 것, 그것이 이 챕터가 가르친 마지막 규율입니다.

---

## 26.9 Best Practices & 안티패턴

실무 체크리스트입니다.

**이렇게 하라**

- **항상 `--release`로 측정하라.** 디버그 빌드는 릴리스보다 통상 10배 이상 느립니다(직접 측정: 같은 합산 루프 디버그 0.51s vs 릴리스 0.05s). 디버그 빌드 벤치마크는 의미가 없습니다.
- **`black_box`(또는 `criterion`/`divan`)를 빠짐없이 써라.** 입력은 상수 폴딩 방지, 출력은 DCE 방지로 감쌉니다. 안 그러면 "0초" 가짜 측정이 나옵니다.
- **측정 맥락을 명시하라.** 입력 크기, 빌드 모드, CPU, 반복 횟수를 벤치마크 결과에 함께 적습니다. 절대 수치보다 *배수*로 보고하는 것이 환경 차이에 강건합니다.
- **고치기 전에 프로파일하라.** `cargo flamegraph`/`samply`로 핫스팟을 먼저 찾습니다. 시간의 80%가 코드의 5%에 있습니다([Chapter 22](./chapter_22.md)).
- **할당을 의심하라.** `dhat`으로 할당 핫스팟을 보고, 불필요한 `String`/`Vec`/`clone`을 제로카피 슬라이스로 바꿉니다([Chapter 5](./chapter_05.md), [Chapter 15](./chapter_15.md)).
- **자동 벡터화를 먼저, 명시적 SIMD는 나중에.** 단순한 루프를 짜 컴파일러가 벡터화하게 하고, `--emit asm`으로 확인합니다. 그래도 안 되고 프로파일러가 가리킬 때만 `std::simd`/SIMD 크레이트를 꺼냅니다.
- **빌드 튜닝은 배포·벤치 빌드에만, 그리고 측정으로 검증.** `lto`/`codegen-units=1`은 빌드를 느리게 하니 개발 중엔 끕니다.

**이건 피하라(안티패턴)**

- **마이크로벤치마크를 맹신하기.** 인라이닝·상수 폴딩으로 격리된 함수가 실제 코드에서와 다르게 동작할 수 있습니다. 가능하면 현실적 입력으로 더 큰 단위를 측정하세요.
- **한 번 측정하고 결론 내기.** 통계적 변동을 무시하면 노이즈를 신호로 착각합니다. `criterion`이 여러 샘플과 신뢰구간을 주는 이유입니다.
- **`clone()` 남발로 borrow checker 회피하기.** [Chapter 1](./chapter_01.md)의 함정 그대로 — 컴파일은 되지만 매번 힙 복사로 느려집니다.
- **추측에 기반한 SIMD/unsafe 도입.** 측정도 없이 `unsafe` 인트린식을 쓰면 메모리 안전성([Chapter 21](./chapter_21.md))을 잃으면서 성능 이득도 불확실합니다.

`clippy`도 성능 린트를 일부 제공합니다(`clippy::needless_collect`, `clippy::redundant_clone` 등). CI에서 `cargo clippy -- -D warnings`로 켜두면 불필요한 할당을 자동으로 잡아줍니다.

---

## 26.10 흔한 함정 (Python 개발자가 자주 하는 실수)

**함정 1 — 디버그 빌드로 벤치마크하기.**

가장 흔하고 가장 치명적입니다. `cargo run`(디버그)으로 측정하고 "Rust가 Python보다 느리네?"라고 결론 내립니다. 디버그 빌드는 최적화가 꺼져 있습니다. 직접 측정한 차이입니다(같은 100회 합산 루프).

```text
=== DEBUG build (no -O) ===  0.5089s
=== RELEASE build (-O) ===   0.0517s
```

약 **10배** 차이입니다. 벤치마크는 반드시 `cargo bench`(자동으로 release 상속) 또는 `cargo run --release`로 하십시오.

**함정 2 — `black_box` 누락으로 0초가 나온다.**

26.4에서 본 그대로. 측정하려던 루프를 컴파일러가 통째로 지웁니다.

```text
$ ./noblackbox    # black_box 없는 10억 회 루프
0.000000s         # 루프가 사라짐!
```

"천 배 빠르다"는 거의 항상 이 가짜입니다. 입력과 출력을 `black_box`로 감싸십시오.

**함정 3 — 마이크로벤치마크의 인라이닝·상수 폴딩 함정.**

작은 함수를 격리해 측정하면, 컴파일러가 그 함수를 호출 지점에 인라인하고 인자가 상수면 결과를 미리 계산합니다. 그래서 격리된 벤치마크가 "빠르다"고 나와도, 실제 코드 맥락에서는 인라인이 안 되거나 다른 최적화에 막혀 다르게 동작할 수 있습니다. `#[inline(never)]`로 측정 대상을 강제로 함수 경계에 묶거나, `black_box`로 인자를 불투명하게 만들어 상수 폴딩을 막아야 정직한 측정이 됩니다. 이 챕터의 `sum_f32`/`dot_product`에 `#[inline(never)]`를 붙인 이유가 이것입니다(어셈블리를 추출하려면 함수가 사라지면 안 되니까요).

**함정 4 — 디버그 빌드를 프로파일링하기.**

벤치마크와 같은 실수가 프로파일링에서도 일어납니다. 디버그 빌드를 `perf`로 프로파일하면, 릴리스에서는 인라인되어 사라질 함수들이 콜스택에 잔뜩 나타나, 존재하지도 않는 병목을 잡고 씨름하게 됩니다. 반드시 `--release` + `debug = true`(심볼용) 조합으로 프로파일하십시오.

**함정 5 — perf 권한·환경 문제를 Rust 탓으로.**

`perf record`가 `permission denied`를 내거나 컨테이너에서 PMU가 안 보이는 것은 OS·하이퍼바이저 수준 문제입니다(26.3의 No Free Lunch 박스). `kernel.perf_event_paranoid` 설정, 컨테이너 권한, 클라우드 VM의 PMU 노출 여부를 확인하세요. macOS면 `samply`나 `cargo instruments`로 우회합니다.

**함정 6 — "Rust니까 무조건 빠르다"는 미신.**

[Chapter 1](./chapter_01.md)에서 경고한 그대로입니다. I/O 바운드 코드([Chapter 18](./chapter_18.md)에서 본 네트워크 대기), 또는 이미 numpy/polars가 네이티브로 처리하는 워크로드를 Rust로 옮겨도 빨라지지 않습니다. 병목이 CPU 인터프리터 루프이거나 캐시·할당 효율일 때만 큰 배수가 나옵니다. 그리고 그것을 아는 유일한 방법은 — 다시 한번 — **측정**입니다.

이 여섯 함정을 관통하는 한 문장은 [Chapter 1](./chapter_01.md)의 그 격언입니다. **측정 없이 최적화하지 말고, 측정값을 의심하라.** Python에서 우리는 전반부(측정하라)를 배웠습니다. Rust는 후반부(측정값을 의심하라)를 추가합니다 — 똑똑한 컴파일러가 벤치마크를 왜곡할 수 있으니까요. `black_box`, `--release`, `--emit asm`, `criterion`은 그 의심을 도구로 체계화한 것입니다.

---

## 연습문제

성능 작업은 글로 배울 수 없습니다. 직접 측정하고, 어셈블리를 눈으로 확인하고, 숫자가 바뀌는 것을 봐야 몸에 익습니다. 아래 네 문제는 모두 **자기검증**이 가능합니다 — 정답인지 아닌지를 사람이 아니라 측정값과 컴파일러가 말해 줍니다.

**문제 1 — `black_box`가 사라지면 시간도 사라진다.** 다음 벤치마크는 1억 번의 제곱합을 측정하려 합니다. 그런데 `--release`로 빌드하면 출력이 `0.000000s`로 나옵니다. (a) 왜 0초가 나오는지 [26.4](#264-벤치마크-방법론-정본화--criterion-black_box-divan)의 논리로 설명하고, (b) `std::hint::black_box`를 어디에 끼워 넣어야 정직한 측정이 되는지 고치십시오.

```rust
use std::time::Instant;
fn main() {
    let t = Instant::now();
    let mut total: u64 = 0;
    for i in 0..100_000_000u64 {
        total += i * i;            // 결과를 아무도 쓰지 않는다
    }
    println!("{total} in {:?}", t.elapsed());
}
```

> **정답 스케치.** (a) `total`이 출력에 쓰이긴 하지만, 컴파일러는 루프 전체가 부수효과 없는 순수 계산임을 알고 닫힌 형태로 접거나(상수 폴딩) 통째로 제거합니다. (b) 입력과 출력 양쪽을 가립니다: `total += black_box(i) * black_box(i);` 그리고 `black_box(total)`. 자기검증 — 수정 전 `0.000000s`, 수정 후 수십 ms 단위의 0이 아닌 값이 나오면 성공입니다. 진짜 벤치마크라면 손으로 `Instant`를 재지 말고 `criterion`에 맡기십시오.

**문제 2 — 컴파일러가 SIMD를 쓰는지 눈으로 확인하라.** 아래 `sum_slice`가 자동 벡터화(auto-vectorization)되어 SIMD 명령으로 컴파일되는지 [26.5](#265-under-the-hood--자동-벡터화auto-vectorization-확인하기)의 방법으로 확인하십시오.

```rust
#[inline(never)]
#[no_mangle]
pub fn sum_slice(xs: &[f64]) -> f64 {
    xs.iter().sum()
}
```

> **정답 스케치.** `rustc -O --emit asm`(또는 godbolt)으로 `sum_slice`를 뽑아, AArch64라면 `fadd v0.2d`/`faddp`, x86-64(AVX)라면 `addpd`/`vaddpd` 같은 **벡터 레지스터·벡터 덧셈** 명령이 보이면 벡터화된 것입니다. 스칼라 `fadd d0, d0, d1`만 보인다면 아닙니다 — `f64` 합산은 부동소수 덧셈의 결합법칙이 성립하지 않아([Chapter 16](./chapter_16.md)) 기본적으로 벡터화가 막힐 수 있습니다. `-C target-cpu=native`를 주거나 `f32`로 바꿔 차이를 관찰해 보십시오.

**문제 3 — AoS vs SoA를 측정으로 증명하라.** 입자 100만 개의 속력 합 `Σ√(vx²+vy²)`을 두 방식으로 작성하고 `criterion`으로 비교하십시오. (a) AoS: `Vec<Particle>`에서 `vx`, `vy`만 읽기. (b) SoA: `vx`, `vy`를 각각 별도 `Vec<f64>`로.

```rust
struct Particle { x: f64, y: f64, vx: f64, vy: f64 }
fn speed_aos(ps: &[Particle]) -> f64 {
    ps.iter().map(|p| (p.vx * p.vx + p.vy * p.vy).sqrt()).sum()
}
fn speed_soa(vx: &[f64], vy: &[f64]) -> f64 {
    vx.iter().zip(vy).map(|(a, b)| (a * a + b * b).sqrt()).sum()
}
```

> **정답 스케치.** 두 함수의 결과값은 **같아야** 합니다(자기검증 1: `assert!((speed_aos(&ps) - speed_soa(&vx,&vy)).abs() < 1e-6)`). 하지만 SoA가 같거나 더 빠릅니다(자기검증 2: criterion 중앙값 비교). 이유는 [Chapter 1](./chapter_01.md)의 캐시 라인 논리입니다 — AoS는 한 입자당 32바이트(`x,y,vx,vy`)를 캐시 라인에 끌어오지만 그중 절반(`x,y`)은 안 씁니다. SoA는 `vx`, `vy`만 연속으로 읽어 캐시 라인을 100% 활용하고 벡터화도 쉬워집니다. 데이터가 클수록 격차가 벌어집니다.

**문제 4 — 최적화가 진짜였는지 되돌려 증명하라.** 문제 3에서 SoA가 빠르다고 결론 냈다고 합시다. 동료가 "그건 그냥 측정 노이즈 아니냐"고 묻습니다. `criterion`의 어떤 출력으로 이 의심을 반박하시겠습니까? 그리고 개선을 커밋한 뒤 [회귀(regression)를 자동으로 잡으려면](#264-벤치마크-방법론-정본화--criterion-black_box-divan) 무엇을 CI에 넣어야 합니까?

> **정답 스케치.** criterion은 표본을 반복 측정해 **신뢰구간(confidence interval)**과 이상치 비율을 보고하고, 직전 실행 대비 변화가 통계적으로 유의한지(`Performance has improved`/`regressed`/`No change`)를 판정합니다. 두 벤치의 신뢰구간이 겹치지 않으면 노이즈가 아닙니다. CI에는 `cargo bench`의 baseline 저장(`--save-baseline`)과 비교(`--baseline`)를 넣고, 임계치를 넘는 회귀에서 빌드를 실패시키면([부록 A](../appendices/appendix_a_cargo.md)의 CI 게이트와 같은 원리) 성능 회귀가 코드 회귀처럼 막힙니다.

---

이 챕터로 우리는 Part 9 "프로덕션으로 가는 길"을, 그리고 이 책이 처음부터 견지해온 측정 정직성의 원칙을 한곳으로 수렴했습니다. [Chapter 1](./chapter_01.md)의 제곱합 루프부터 [Chapter 15](./chapter_15.md)의 이터레이터 파이프라인, [Chapter 16](./chapter_16.md)의 멀티스레드 측정, [Chapter 22](./chapter_22.md)의 PyO3 가속까지 — 모든 수치 뒤에는 `black_box`와 `--release`와 통계적 측정이 있었습니다. 이제 여러분은 그 방법론을 처음부터 끝까지 손에 쥐었습니다.

Python에서 성능 작업은 "측정하고, 인터프리터를 피해 도망가는" 게임이었습니다. Rust에서는 "측정하고, CPU에 더 가까이 내려가 캐시와 SIMD와 명령어를 직접 다루는" 게임입니다. 그 깊이가 부담이 아니라 자유로 느껴지는 순간 — `--emit asm`으로 `fmul.4s`를 눈으로 확인하고, `criterion`의 신뢰구간으로 개선을 증명하고, `dhat`으로 사라진 할당을 세어보는 그 순간 — 여러분은 라이브러리를 쓰는 사람에서 라이브러리를 만드는 사람으로 넘어가 있을 것입니다. Pydantic·Polars·ruff의 작성자들이 선 그 자리입니다. 측정으로 시작해 측정으로 끝내는 이 규율이, 그 자리로 가는 가장 확실한 길입니다.

---

## References

- https://doc.rust-lang.org/std/hint/fn.black_box.html
- https://bheisler.github.io/criterion.rs/book/
- https://docs.rs/criterion/latest/criterion/
- https://docs.rs/divan/latest/divan/
- https://docs.rs/dhat/latest/dhat/
- https://github.com/flamegraph-rs/flamegraph
- https://github.com/mstange/samply
- https://doc.rust-lang.org/std/simd/index.html
- https://rust-lang.github.io/packed_simd/perf-guide/
- https://nnethercote.github.io/perf-book/
- https://doc.rust-lang.org/cargo/reference/profiles.html
