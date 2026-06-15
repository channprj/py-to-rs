# Chapter 24. CLI 애플리케이션: argparse/click에서 clap으로

데이터 엔지니어든 인프라 엔지니어든, 10년 차쯤 되면 누구나 자기만의 CLI 도구 묘지를 하나씩 갖고 있습니다. 로그를 긁어 집계하는 스크립트, 설정 파일을 검증하는 도구, S3 버킷을 정리하는 일회성 자동화, 배포 파이프라인에 끼워 넣은 헬퍼 — 처음엔 `sys.argv[1]`로 시작했다가, 옵션이 늘면서 `argparse`로 갈아탔고, 데코레이터가 예뻐 보여서 `click`으로 다시 썼을 것입니다. 그리고 그 도구를 동료에게 건넬 때마다 똑같은 대화를 반복합니다. "아 그거? `pip install -r requirements.txt` 먼저 하고, Python 3.11 이상이어야 하고, `PYTHONPATH`를 잡아줘야 돼."

CLI 도구의 본질적 모순이 여기 있습니다. CLI는 "그냥 실행되는 한 개의 명령어"여야 하는데, Python으로 만든 CLI는 사실 **실행 환경 전체를 끌고 다니는 애플리케이션**입니다. 인터프리터, 가상환경, 의존성 트리, 그리고 그 모든 것을 묶는 부적절한 접착제까지. `pyinstaller`나 `shiv`로 단일 파일을 만들어 봐도, 수십 MB짜리 바이너리가 인터프리터를 통째로 품고 콜드 스타트마다 그것을 펼칩니다. CLI 도구를 동료의 머신에, CI 러너에, Docker 베이스 이미지에 배포하는 일은 늘 생각보다 번거롭습니다.

성능도 무시할 수 없습니다. CLI는 대개 짧게 살다 죽습니다. 한 번 실행되고 끝나죠. 그런데 Python CLI는 매 실행마다 인터프리터를 부팅하고, `import`로 모듈 그래프를 끌어오고, `argparse`/`click`의 파서 객체를 런타임에 조립합니다. 이 책의 환경(Python 3.13.6)에서 빈 `argparse` 서브커맨드 파서를 가진 스크립트의 프로세스 시작 시간 중앙값을 측정하면 약 **38ms**, `click`은 약 **41ms**입니다. 한두 번 실행이면 무시할 만하지만, 셸 자동완성이 매 탭마다 CLI를 호출하거나, 셸 스크립트가 루프 안에서 도구를 수천 번 부르거나, `pre-commit` 훅이 파일마다 린터를 띄우는 상황에서는 이 콜드 스타트 비용이 누적되어 체감됩니다.

마지막으로 가장 미묘한 고통은 **타입 검증의 시점**입니다. `argparse`에서 `type=int`를 붙이고 `choices=["json", "text"]`를 지정해도, 그것은 전부 런타임에 평가됩니다. `--count`에 숫자가 아닌 값이 들어오면 사용자가 그 코드 경로를 밟기 전까지 아무도 모릅니다. 서브커맨드를 추가하고 디스패치 `if/elif` 분기에 처리를 깜빡 빠뜨려도, Python은 잠자코 통과시킵니다. `click`의 데코레이터는 이 검증을 함수 호출 시점으로 미룰 뿐, 본질은 같습니다 — 인자 구조가 코드의 정적 구조가 아니라 런타임에 조립되는 데이터이기 때문입니다.

Rust의 `clap`은 이 네 가지 고통 — 배포, 콜드 스타트, 런타임 비용, 검증 시점 — 을 정면으로 다룹니다. 핵심 아이디어는 이 책 전체를 관통하는 그 철학입니다. **인자 구조를 런타임 데이터가 아니라 컴파일 타임 타입으로 표현한다.** `#[derive(Parser)]`를 붙인 `struct`의 필드가 곧 인자이고, `enum`의 변형(variant)이 곧 서브커맨드이며, 타입 파싱·검증·`--help` 생성·버전 출력이 전부 그 구조에서 컴파일 타임에 파생됩니다. 그리고 결과물은 의존성 0개의 단일 정적 바이너리입니다 — 이 챕터에서 만들 도구의 릴리스 바이너리는 약 1MB이고, 시작 시간 중앙값은 **2.5ms**, Python 대비 약 15배 빠릅니다.

이번 챕터에서는 [Chapter 14](./chapter_14.md)의 로그 분석기를 서브커맨드와 파일 입출력을 갖춘 실배포 가능한 CLI로 승격시킵니다. 그 과정에서 [Chapter 10](./chapter_10.md)의 `enum` 망라성, [Chapter 12](./chapter_12.md)의 `Option`/`Result`, [Chapter 13](./chapter_13.md)의 `anyhow`가 어떻게 CLI라는 무대에서 한데 엮이는지 보게 될 것입니다. 그리고 늘 그렇듯, "어떻게 쓰는가"보다 "왜 그렇게 동작하는가"에 무게를 둡니다 — `derive(Parser)`가 `cargo expand`로 펼쳤을 때 무엇을 만들어내는지까지 내려갑니다.

---

## 24.1 The Python Way — `argparse`/`click`과 런타임 조립

먼저 익숙한 방식을 정확히 짚읍시다. 다음은 전형적인 `argparse` 서브커맨드 CLI입니다.

```python
import argparse
import sys

def main() -> int:
    parser = argparse.ArgumentParser(description="HTTP access log analyzer")
    parser.add_argument("--format", choices=["text", "json"], default="text")
    parser.add_argument("-v", "--verbose", action="count", default=0)

    sub = parser.add_subparsers(dest="command", required=True)

    p_stats = sub.add_parser("stats", help="상태 코드별 요청 수")
    p_stats.add_argument("file", nargs="?")  # 선택 인자

    p_top = sub.add_parser("top", help="Top-N 엔드포인트")
    p_top.add_argument("files", nargs="*")   # 다중 인자
    p_top.add_argument("-n", "--number", type=int, default=3)

    args = parser.parse_args()

    if args.command == "stats":
        return run_stats(args.file, args.format)
    elif args.command == "top":
        return run_top(args.files, args.number, args.format)
    # 'tail' 서브커맨드를 나중에 추가하면? 이 분기에 넣는 걸 깜빡해도 아무도 안 막는다.
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

이 코드는 명쾌하고 잘 동작합니다. 하지만 시니어의 눈으로 보면 세 가지 구조적 비용이 보입니다.

**첫째, 파서가 런타임에 조립됩니다.** `add_argument` 호출 하나하나가 실행 시점에 `ArgumentParser` 객체의 내부 딕셔너리에 인자 스펙을 쌓습니다. 인자가 30개라면 30번의 메서드 호출이 매 실행마다 일어나고, 그 모든 객체가 힙에 할당됩니다. `args.file`로 결과를 꺼낼 때 반환되는 것은 동적 `Namespace` 객체이고, 그 속성은 문자열 키로 조회됩니다. 타입 시스템의 도움은 전혀 없습니다 — `args.fil`(오타)라고 써도 IDE도 인터프리터도 정의 시점엔 침묵하다가, 런타임에 `AttributeError`로 터집니다.

**둘째, 검증이 런타임으로 미뤄집니다.** `type=int`, `choices=[...]`, `required=True`는 전부 파싱이 실제로 일어나는 순간 평가됩니다. 잘못된 입력은 사용자가 그 인자를 줄 때 비로소 거부됩니다. 이건 사실 CLI의 본질상 어쩔 수 없는 부분이 있습니다(입력은 런타임에 오니까). 문제는 **인자 구조 자체의 정합성**도 런타임까지 검증되지 않는다는 점입니다. 서브커맨드 디스패치의 `if/elif` 사슬은 그저 평범한 Python 코드라, 분기 하나를 빠뜨려도 컴파일러가 없으니 잡을 길이 없습니다.

**셋째, 분기 망라성을 강제할 수 없습니다.** 위 코드에 `tail` 서브커맨드를 추가한다고 합시다. `sub.add_parser("tail")`을 추가하고 디스패치 `if/elif`에 `elif args.command == "tail"`을 넣어야 하는데, 후자를 깜빡하면 `tail`은 조용히 `return 0`으로 떨어집니다. 사용자는 "왜 아무 일도 안 일어나지?"라고 당황하고, 여러분은 새벽에 그 버그를 추적합니다. Python에는 "모든 서브커맨드를 처리했는지" 확인해줄 메커니즘이 없습니다.

`click`은 데코레이터로 보일러플레이트를 줄여 더 우아해 보입니다.

```python
import click

@click.group()
@click.option("--format", type=click.Choice(["text", "json"]), default="text")
@click.pass_context
def cli(ctx, format):
    ctx.obj = {"format": format}

@cli.command()
@click.argument("file", required=False)
@click.pass_context
def stats(ctx, file):
    run_stats(file, ctx.obj["format"])

@cli.command()
@click.argument("files", nargs=-1)
@click.option("-n", "--number", type=int, default=3)
@click.pass_context
def top(ctx, files, number, ):
    run_top(files, number, ctx.obj["format"])
```

`click`은 서브커맨드 디스패치를 자동화한다는 점에서 `argparse`보다 한 걸음 낫습니다 — `@cli.command()`로 등록된 함수가 자동으로 라우팅되니, 분기 누락 문제는 줄어듭니다. 하지만 대가가 있습니다. 각 데코레이터는 **런타임에** 함수를 감싸 `Command` 객체를 만들고, 인자 메타데이터를 클로저와 객체 속성으로 누적합니다. `ctx.obj`로 전역 상태를 주고받는 패턴은 타입이 `dict`라 IDE 자동완성도, 정적 검증도 받지 못합니다. `@click.pass_context`를 빠뜨리거나 `ctx.obj` 키 이름을 틀려도 런타임 `KeyError`로만 드러납니다. 데코레이터는 마법처럼 보이지만, 그 마법은 전부 실행 시점에 풀립니다.

정리하면 Python CLI의 구조적 한계는 세 가지입니다. (1) 파서가 런타임에 객체로 조립되어 시작 비용이 들고, (2) 인자 결과가 동적 `Namespace`/`dict`라 타입 안전성이 없으며, (3) 서브커맨드 분기 망라성을 언어가 강제하지 못합니다. `clap`은 이 셋을 각각 컴파일 타임 코드 생성, `struct` 필드 타입, `enum` 매칭 망라성으로 정면 돌파합니다.

---

## 24.2 The Rust Mental Model — 인자는 타입이고, CLI는 구조다

Rust는 이 문제를 정반대 방향에서 접근합니다. Python이 "런타임에 파서 객체를 조립하고 동적 네임스페이스를 꺼낸다"면, Rust의 `clap`은 "원하는 결과 타입을 먼저 `struct`로 선언하면, 그 타입을 채우는 파서가 컴파일 타임에 생성된다"입니다. 방향이 뒤집혀 있습니다 — 파서를 만들고 결과를 꺼내는 게 아니라, **결과의 모양을 선언하면 파서가 따라옵니다.**

핵심은 단 하나의 파생 매크로, `#[derive(Parser)]`입니다.

```rust
use clap::Parser;

/// HTTP 액세스 로그 분석기
#[derive(Parser, Debug)]
#[command(name = "logcli", version, about = "HTTP access log analyzer")]
struct Cli {
    /// 분석할 로그 파일 (생략 시 stdin)
    file: Option<std::path::PathBuf>,

    /// 출력 형식
    #[arg(long, default_value = "text")]
    format: String,

    /// 상세 로그 (-v, -vv 로 단계 조절)
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
}

fn main() {
    let cli = Cli::parse();   // 파싱 실패 시 친절한 에러 출력 후 종료
    println!("{cli:?}");
}
```

이 `struct`를 읽으면 CLI의 전체 인터페이스가 한눈에 들어옵니다. **필드 하나가 인자 하나**입니다. 그리고 그 매핑 규칙이 놀랍도록 직관적입니다.

- `file: Option<PathBuf>` → **선택적** 위치 인자. `Option<T>`라는 타입이 "있어도 되고 없어도 된다"를 그대로 표현합니다([Chapter 12](./chapter_12.md)의 그 `Option`입니다).
- `format: String` → 값을 받는 옵션. 기본값이 있으니 필수가 아닙니다.
- `verbose: u8` + `ArgAction::Count` → `-v`, `-vv`, `-vvv`를 세는 카운터.
- 필드 위의 `///` 문서 주석(doc comment)이 **그대로 `--help` 텍스트가 됩니다.** 별도로 help 문자열을 쓸 필요가 없습니다.

여기서 가장 중요한 인식 전환이 일어납니다. Python에서 `args`는 무엇이든 될 수 있는 동적 `Namespace`였습니다. Rust에서 `cli`는 **컴파일러가 그 모양을 완전히 아는 `Cli` 타입의 값**입니다. `cli.fil`이라고 오타를 내면 컴파일이 거부됩니다. `cli.verbose`가 `u8`이라는 것도 타입으로 박제되어 있습니다. 인자 파싱의 결과가 동적 데이터가 아니라 **여러분이 직접 설계한 정적 타입**이라는 것 — 이것이 `clap` derive의 본질입니다.

타입과 인자의 매핑을 Python과 한 표로 정리하면 다음과 같습니다.

| Rust 필드 타입 | clap에서의 의미 | Python `argparse` 대응 |
|----------------|-----------------|------------------------|
| `String` (기본값 없음) | **필수** 옵션/위치 인자 | `required=True` |
| `Option<T>` | **선택** 인자 (없으면 `None`) | `default=None`, `nargs="?"` |
| `Vec<T>` | **다중** 값 (0개 이상) | `nargs="*"` / `action="append"` |
| `bool` | 플래그 (있으면 `true`) | `action="store_true"` |
| `u8` + `ArgAction::Count` | 반복 카운트 (`-vvv`) | `action="count"` |
| `enum` + `ValueEnum` | 고정된 선택지 | `choices=[...]` |
| `PathBuf` | 파일 경로 (비-UTF-8 안전) | `str` (경로도 그냥 문자열) |
| 별도 `struct` + `#[derive(Subcommand)]` | 서브커맨드 트리 | `add_subparsers` |

이 표가 의미하는 바는 분명합니다. Python에서는 `add_argument`의 키워드 인자(`nargs`, `action`, `required`, `default`)로 **명령형으로** 인자의 성질을 지정했다면, Rust에서는 그 성질이 **필드의 타입 자체로 선언적으로** 표현됩니다. "선택적이다"는 `Option`이, "여러 개다"는 `Vec`이, "정해진 선택지다"는 `enum`이 말해줍니다. 타입이 곧 명세(spec)입니다.

그리고 결정적으로, 이 모든 것이 **컴파일 타임에 코드로 변환**됩니다. Python의 `argparse`가 런타임에 파서를 조립하는 것과 달리, `clap` derive는 빌드 시점에 파싱 함수를 통째로 생성합니다. 다음 절에서 그 생성된 코드를 직접 펼쳐 봅니다.

---

## 24.3 Under the Hood — `derive(Parser)`가 펼치면 무엇이 나오는가

이 챕터에서 단 하나만 기억해야 한다면 이것입니다. **`#[derive(Parser)]`는 런타임 인트로스펙션이 아니라 컴파일 타임 코드 생성입니다.** Python의 데코레이터가 런타임에 함수를 감싸고 객체를 만드는 것과 근본적으로 다릅니다. clap의 파생 매크로는 빌드할 때 여러분의 `struct`를 읽고, 그에 맞는 파싱 코드를 소스 레벨에서 만들어 컴파일러에 넘깁니다.

[Chapter 19](./chapter_19.md)에서 본 절차적 매크로(procedural macro)가 바로 이 일을 합니다. 그리고 `cargo expand` 도구로 매크로가 펼쳐진 후의 코드를 실제로 볼 수 있습니다. 앞서 본 간단한 `struct`를 펼쳐 봅시다.

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// 이름
    name: String,
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}
```

`cargo expand`를 돌리면(이 책의 환경: clap 4.6.1) 약 300줄의 코드가 생성됩니다. 그중 핵심 골격을 발췌하면 이렇습니다.

```rust
// cargo expand 출력 (발췌, 실제 생성 코드)
#[automatically_derived]
impl clap::Parser for Cli {}

#[automatically_derived]
impl clap::CommandFactory for Cli {
    fn command<'b>() -> clap::Command {
        let __clap_app = clap::Command::new("expanddemo");
        <Self as clap::Args>::augment_args(__clap_app)
    }
}

#[automatically_derived]
impl clap::FromArgMatches for Cli {
    fn from_arg_matches_mut(
        __clap_arg_matches: &mut clap::ArgMatches,
    ) -> Result<Self, clap::Error> {
        let v = Cli {
            name: __clap_arg_matches
                .remove_one::<String>("name")
                .ok_or_else(|| clap::Error::raw(
                    clap::error::ErrorKind::MissingRequiredArgument,
                    "the following required argument was not provided: name",
                ))?,
            count: __clap_arg_matches.remove_one::<u8>("count").ok_or_else(/* ... */)?,
        };
        Ok(v)
    }
    // ... update_from_arg_matches 등 생략
}
```

그리고 인자 정의를 빌더 호출로 변환한 부분(`augment_args`)은 이렇습니다.

```rust
// cargo expand 출력 (발췌) — 인자 하나하나가 빌더 호출로 펼쳐진다
fn augment_args<'b>(__clap_app: clap::Command) -> clap::Command {
    let __clap_app = __clap_app.arg({
        let arg = clap::Arg::new("name")
            .value_name("NAME")
            .required(true && clap::ArgAction::Set.takes_values())
            .value_parser(/* String용 자동 추론된 파서 */)
            .action(clap::ArgAction::Set);
        let arg = arg.help("이름").long_help(None);   // ← doc 주석이 help가 됨
        arg
    });
    let __clap_app = __clap_app.arg({
        let arg = clap::Arg::new("count")
            .value_name("COUNT")
            .required(false && clap::ArgAction::Set.takes_values())
            .value_parser(/* u8용 파서 */)
            .action(clap::ArgAction::Set);
        // ... default_value 등
        arg
    });
    __clap_app
}
```

여기서 세 가지 결정적 통찰이 나옵니다.

**첫째, derive는 builder API의 설탕(sugar)입니다.** clap에는 derive API와 builder API 두 가지가 있는데, derive가 컴파일 타임에 builder 호출(`Command::new().arg(...)`)로 펼쳐집니다. Python의 `add_argument`와 똑같은 빌더 패턴이 **런타임이 아니라 컴파일 타임에, 그리고 여러분이 손으로 쓰는 대신 매크로가 생성**해주는 것입니다. 직접 builder API를 써도 되지만, derive를 쓰면 타입 안전한 결과 `struct`를 공짜로 얻습니다.

**둘째, 결과 추출이 타입으로 고정됩니다.** `from_arg_matches_mut`를 보세요. `remove_one::<String>("name")`, `remove_one::<u8>("count")` — 각 필드를 정확한 타입으로 꺼냅니다. Python의 `args.name`이 무엇이든 될 수 있는 동적 속성이었다면, 여기서는 컴파일러가 `name`이 `String`, `count`가 `u8`임을 보증합니다. `ArgMatches`라는 동적 중간 표현은 derive를 쓰면 사용자에게 노출조차 되지 않습니다.

**셋째, doc 주석이 help가 됩니다.** `.help("이름")`을 보세요. 우리가 `/// 이름`이라고 쓴 문서 주석이 컴파일 타임에 추출되어 `--help` 텍스트로 박혔습니다. 문서와 도움말이 절대 어긋나지 않습니다 — 같은 소스에서 나오니까요.

> **💡 Under the Hood — 런타임 인트로스펙션 vs 컴파일 타임 코드 생성**
>
> 이 차이를 메모리와 시간 관점에서 못 박아 둡시다. Python의 `argparse`는 매 실행마다 `ArgumentParser()` 객체를 힙에 만들고, `add_argument`마다 인자 스펙 딕셔너리를 채우며, 파싱 결과를 동적 `Namespace` 객체로 반환합니다. 이 모든 객체(`PyObject` 헤더 + 딕셔너리)가 매 프로세스 시작 시 할당됩니다. `click`은 여기에 데코레이터가 만든 `Command`/`Option` 객체와 클로저 체인까지 더합니다.
>
> Rust의 clap derive는 이 작업을 **컴파일 타임에 한 번** 합니다. 생성된 `command()`와 `from_arg_matches`는 평범한 함수이고, 컴파일러가 인라이닝·최적화합니다. 런타임에 남는 것은 인자 스펙을 담은 `Command` 구조체(스택/정적 데이터에 가깝게 배치)를 한 번 만들고 입력을 매칭하는 비용뿐입니다. 동적 딕셔너리 룩업도, `PyObject` 할당도, 인터프리터 디스패치도 없습니다.
>
> 측정으로 확인합시다. 이 챕터의 logcli를 `--release`로 빌드한 단일 바이너리(약 1,064,400바이트 ≈ 1MB)의 프로세스 시작 시간 중앙값은 **2.5ms**(최소 2.2ms)입니다. 동일 작업을 하는 `argparse` 스크립트는 **38ms**(중앙값), `click`은 **41ms**입니다. 약 15배 차이이며, 그 대부분은 인터프리터 부팅과 모듈 import입니다. CLI가 "한 번 실행되고 죽는" 워크로드라는 점을 떠올리면, 이 콜드 스타트 차이가 곧 사용자 체감 속도입니다.

derive가 컴파일 타임 코드 생성이라는 사실은 단순한 성능 이야기가 아닙니다. 인자 구조가 **타입 시스템 안에 존재**한다는 뜻이고, 그래서 다음 절의 서브커맨드 망라성 검사 같은 일이 가능해집니다.

---

## 24.4 서브커맨드는 enum이다 — 망라성으로 분기 누락 차단

`git commit`, `docker run`, `cargo build`처럼 현대 CLI는 서브커맨드 트리를 갖습니다. Python에서는 `add_subparsers`로 만들고 문자열 비교(`if args.command == "stats"`)로 분기했습니다. Rust에서 서브커맨드는 **`enum`**이고, 분기는 **`match`**입니다. 그리고 [Chapter 10](./chapter_10.md)에서 본 그 `match`의 망라성(exhaustiveness)이 여기서 빛을 발합니다.

```rust
use clap::{Args, Parser, Subcommand};
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(name = "logcli", version, about = "HTTP access log analyzer")]
struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand, Debug)]
enum Command {
    /// 상태 코드별 요청 수를 집계한다
    Stats(StatsArgs),
    /// 가장 많이 호출된 엔드포인트 Top-N
    Top(TopArgs),
}

#[derive(Args, Debug)]
struct StatsArgs {
    /// 분석할 로그 파일 (생략 시 stdin)
    file: Option<PathBuf>,
}

#[derive(Args, Debug)]
struct TopArgs {
    /// 분석할 로그 파일들 (여러 개 가능)
    files: Vec<PathBuf>,
    /// 상위 몇 개를 보여줄지
    #[arg(short, long, default_value_t = 3)]
    number: usize,
}
```

각 서브커맨드는 `enum`의 변형이고, 그 변형은 자신만의 인자 묶음(`StatsArgs`, `TopArgs`)을 데이터로 품습니다. `#[derive(Subcommand)]`가 이 `enum`을 서브커맨드 트리로, `#[derive(Args)]`가 각 인자 묶음을 인자 그룹으로 변환합니다. 디스패치는 평범한 `match`입니다.

```rust
fn run(cli: Cli) {
    match &cli.command {
        Command::Stats(args) => { /* stats 처리 */ }
        Command::Top(args)   => { /* top 처리 */ }
    }
}
```

이제 핵심입니다. `tail` 서브커맨드를 추가한다고 합시다. `enum`에 변형을 추가하고 `match`에 처리를 깜빡하면 어떻게 될까요? Python이었다면 조용히 통과했을 그 버그를, Rust 컴파일러가 거부합니다. 실제로 변형만 추가하고 `match` 가지를 빼 본 결과입니다.

```rust
#[derive(Subcommand)]
enum Command {
    Stats,
    Top,
    Tail, // 새 서브커맨드를 추가했다
}

fn dispatch(cmd: &Command) {
    match cmd {
        Command::Stats => println!("stats"),
        Command::Top => println!("top"),
        // Tail 처리를 깜빡함!
    }
}
```

`cargo build`가 내뱉는 실제 에러입니다.

```text
error[E0004]: non-exhaustive patterns: `&Command::Tail` not covered
  --> src/main.rs:17:11
   |
17 |     match cmd {
   |           ^^^ pattern `&Command::Tail` not covered
   |
note: `Command` defined here
  --> src/main.rs:10:6
   |
10 | enum Command {
   |      ^^^^^^^
...
13 |     Tail, // 새 서브커맨드를 추가했다
   |     ---- not covered
   = note: the matched value is of type `&Command`
help: ensure that all possible cases are being handled by adding a match arm
      with a wildcard pattern or an explicit pattern as shown
   |
19 ~         Command::Top => println!("top"),
20 ~         &Command::Tail => todo!(),
   |
```

**이것이 Python CLI 대비 가장 큰 안전성 이득입니다.** 서브커맨드를 추가했는데 그 처리를 빠뜨리면, 코드가 컴파일조차 되지 않습니다. 디스패치 누락이라는 버그 클래스가 통째로 사라집니다. `argparse`의 `if/elif` 사슬에서, 혹은 `click`에서 데코레이터를 빼먹어서 생기던 그 사고를, Rust는 `error[E0004]`로 배포 전에 잡습니다. 이것은 우연이 아니라 24.3에서 본 "인자 구조가 타입 시스템 안에 존재한다"의 직접적 귀결입니다 — 타입(`enum`) 안에 있으니, 타입 검사기가 빠짐을 안다는 것입니다.

### 환경 변수·기본값·ValueEnum

실무 CLI는 옵션의 기본값을 환경 변수에서 가져오는 경우가 많습니다(12-factor app 스타일). clap은 이것을 어트리뷰트로 선언합니다.

```rust
use clap::{Parser, ValueEnum};

#[derive(Copy, Clone, Debug, PartialEq, Eq, ValueEnum)]
enum Format {
    Text,
    Json,
}

#[derive(Parser, Debug)]
struct Cli {
    /// 출력 형식 (LOGCLI_FORMAT 환경 변수로도 지정 가능)
    #[arg(long, value_enum, default_value_t = Format::Text, env = "LOGCLI_FORMAT")]
    format: Format,
}
```

`#[derive(ValueEnum)]`은 `enum`을 `--format text|json` 같은 고정 선택지로 만듭니다(Python `choices`의 타입 안전 버전). `default_value_t`는 기본값, `env = "LOGCLI_FORMAT"`은 "CLI 인자가 없으면 이 환경 변수를 본다"는 선언입니다. 우선순위는 CLI 인자 > 환경 변수 > 기본값 순입니다. 실제 동작을 확인하면, 생성된 `--help`가 환경 변수 출처까지 친절히 표시합니다.

```text
      --format <FORMAT>  출력 형식 [env: LOGCLI_FORMAT=] [default: text]
                         [possible values: text, json]
```

```bash
$ LOGCLI_FORMAT=json logcli stats sample.log   # 환경 변수로 json 선택됨
$ logcli --format json stats sample.log         # CLI 인자가 우선 (둘 다면 인자가 이김)
```

### ArgGroup과 충돌하는 옵션

서로 배타적인 옵션(예: `--quiet`와 `--verbose`)은 `conflicts_with`로, 여러 옵션 중 하나만 허용하려면 `ArgGroup`으로 표현합니다.

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(group(
    clap::ArgGroup::new("output").required(false).args(["json", "text"]),
))]
struct Cli {
    /// JSON으로 출력
    #[arg(long)]
    json: bool,
    /// 사람이 읽는 텍스트로 출력
    #[arg(long)]
    text: bool,
    /// 조용히 (verbose와 충돌)
    #[arg(short, long, conflicts_with = "verbose")]
    quiet: bool,
    /// 상세 로그
    #[arg(short, long)]
    verbose: bool,
}
```

실제로 충돌하는 조합을 주면 clap이 종료 코드 2로 거부합니다.

```text
$ logcli --json --text
error: the argument '--json' cannot be used with '--text'

$ logcli -q -v
error: the argument '--quiet' cannot be used with '--verbose'
```

Python `argparse`에서 상호 배타를 표현하려면 `add_mutually_exclusive_group()`을 명령형으로 호출하고, 그 검증은 런타임에 일어납니다. clap에서는 이 제약이 어트리뷰트로 선언되고 `Command` 구조에 박혀, 같은 파서 코드 경로에서 일관되게 강제됩니다.

---

## 24.5 검증과 String vs PathBuf — 타입으로 입력을 정제하기

CLI 인자의 가장 흔한 함정 두 가지가 여기 있습니다. **숫자 범위 검증**과 **경로를 String으로 받는 실수**입니다.

### value_parser로 검증을 타입에 붙이기

`-n` 옵션이 1에서 100 사이여야 한다고 합시다. Python에서는 파싱 후 `if not 1 <= n <= 100: parser.error(...)`를 손으로 써야 합니다. clap에서는 `value_parser`로 검증을 인자 정의에 붙입니다.

```rust
use clap::Parser;

#[derive(Parser, Debug)]
struct Cli {
    /// 1..=100 사이만 허용
    #[arg(short, long, value_parser = clap::value_parser!(u16).range(1..=100))]
    number: u16,
}
```

범위를 벗어난 값을 주면 clap이 파싱 단계에서 거부합니다 — 우리 코드의 본문은 실행되지도 않습니다.

```text
$ logcli -n 200
error: invalid value '200' for '--number <NUMBER>': 200 is not in 1..=100

$ logcli -n 50    # 통과: Cli { number: 50 }
```

검증을 인자 정의에 붙인다는 것은, "유효한 값만 우리 로직에 도달한다"는 보장을 타입 경계에서 얻는다는 뜻입니다. [Chapter 7](./chapter_07.md)·[Chapter 8](./chapter_08.md)에서 본 뉴타입(newtype)으로 도메인 제약을 인코딩하던 그 발상과 같습니다 — `Money`가 음수를 허용하지 않도록 생성자에서 검증했듯, CLI 인자도 파서가 검증을 통과시킨 값만 넘겨줍니다. 더 복잡한 검증(예: 커스텀 형식 문자열)이 필요하면 `value_parser`에 함수를 넘겨 `Result<T, String>`을 반환하게 할 수 있습니다.

### String이 아니라 PathBuf를 써라

파일 경로를 받는 인자는 `String`이 아니라 **`PathBuf`**로 받아야 합니다. 이유는 두 가지입니다.

첫째, **모든 경로가 유효한 UTF-8은 아닙니다.** Linux 파일명은 NUL과 `/`를 제외한 임의의 바이트 시퀀스일 수 있고, Windows 경로는 UTF-16 기반입니다. `String`은 UTF-8을 강제하므로, 비-UTF-8 경로를 받으면 정보가 손실되거나 거부됩니다. `PathBuf`는 OS의 네이티브 인코딩을 보존합니다.

둘째, **`PathBuf`는 경로 조작 API(`.join()`, `.extension()`, `.parent()`)를 제공**하고, `&Path`로 빌려 파일 시스템 함수에 바로 넘길 수 있습니다. `String`으로 받으면 매번 변환해야 하고, 그 과정에서 타입 불일치 에러를 만납니다. 실제로 경로를 `String`으로 받아 `&Path`가 필요한 곳에 쓰려 하면 이런 에러가 납니다.

```rust
use std::path::Path;

fn open_it(path: String) {
    let _p: &Path = path; // 🚨 컴파일 에러: 타입 불일치
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:6:21
  |
6 |     let _p: &Path = path; // 🚨 타입 불일치
  |             -----   ^^^^ expected `&Path`, found `String`
  |             |
  |             expected due to this
```

해법은 처음부터 `PathBuf`로 받는 것입니다. clap은 `PathBuf`를 1급으로 지원하므로(`value_parser`가 자동 추론됩니다), 필드 타입만 `Option<PathBuf>`로 두면 됩니다. 메모리 크기도 동일합니다 — 이 책의 환경에서 측정하면 `String`과 `PathBuf` 모두 **24바이트**(ptr·len·cap의 3워드 핸들)이고, `Option<PathBuf>` 역시 **24바이트**입니다([Chapter 1](./chapter_01.md)에서 본 니치 최적화로 `None` 태그가 공짜입니다). 즉 `PathBuf`를 쓴다고 더 무거워지지 않으면서, OS 네이티브 경로 안전성과 풍부한 경로 API를 공짜로 얻습니다.

> **💡 Under the Hood — `Vec<PathBuf>` 다중 인자의 메모리**
>
> `top` 서브커맨드의 `files: Vec<PathBuf>`는 여러 파일을 받습니다. 이 `Vec<PathBuf>`의 핸들 자체는 **24바이트**(ptr·len·cap)이고, 그 안에 `PathBuf`(각 24바이트 핸들)들이 연속으로 놓이며, 각 `PathBuf`가 가리키는 실제 경로 바이트는 힙에 따로 있습니다. Python의 `args.files`가 `list[str]`이라면, 그 리스트는 [Chapter 1](./chapter_01.md)에서 해부한 대로 `PyObject*` 포인터 배열이고 각 `str`은 41+n바이트짜리 힙 객체입니다. clap이 파싱한 `Vec<PathBuf>`는 포인터 추적(pointer chasing)이 한 겹 적고 헤더 오버헤드가 없습니다. CLI에서 인자가 수백 개일 일은 드무니 이 차이가 성능을 좌우하진 않지만, "동적 객체 배열 vs 연속 버퍼"라는 이 책 전체의 메모리 대비가 인자 파싱에서도 그대로 반복된다는 점은 짚어둘 만합니다.

---

## 24.6 에러 처리와 종료 코드 — anyhow와 ExitCode

CLI는 셸 파이프라인의 시민입니다. `set -e` 스크립트, `&&` 체인, CI 스텝 — 이 모두가 종료 코드(exit code)로 성공/실패를 판단합니다. 그래서 CLI는 두 가지를 잘해야 합니다. **사람에게는 읽기 쉬운 에러 메시지**를, **스크립트에는 정확한 종료 코드**를 줘야 합니다.

clap 자체의 인자 파싱 에러는 자동으로 종료 코드 2를 냅니다(`Cli::parse()`가 처리). 우리가 신경 쓸 것은 **애플리케이션 로직의 에러**(파일 없음, 파싱 실패 등)입니다. 여기서 [Chapter 13](./chapter_13.md)의 `anyhow`가 등장합니다.

```rust
use std::process::ExitCode;
use anyhow::{Context, Result};

fn run(cli: Cli) -> Result<()> {
    // ... 실제 작업. ? 로 에러를 위로 전파
    Ok(())
}

fn main() -> ExitCode {
    let cli = Cli::parse();   // 파싱 에러는 여기서 자동 종료(코드 2)
    match run(cli) {
        Ok(()) => ExitCode::SUCCESS,        // 0
        Err(e) => {
            eprintln!("error: {e:#}");      // stderr로, {:#}는 원인 체인까지 출력
            ExitCode::from(2)                // 실패 종료 코드
        }
    }
}
```

세 가지 설계 결정이 중요합니다.

**첫째, `main`이 `ExitCode`를 반환합니다.** Rust의 `main`은 `()`, `Result<(), E>`, 또는 `ExitCode`를 반환할 수 있습니다. `ExitCode`를 직접 반환하면 종료 코드를 명시적으로 제어할 수 있습니다. `main() -> Result<(), E>`로 두면 에러 시 자동으로 코드 1과 `Debug` 출력이 나오는데, 이건 사용자에게 보이는 메시지로는 투박합니다(`Debug` 포맷이라 `Error: ...{ ... }` 식으로 나옴). CLI에서는 메시지 포맷과 종료 코드를 직접 통제하는 게 낫습니다.

**둘째, 에러는 stderr로, `{:#}` 포맷으로.** `eprintln!`은 stdout이 아니라 stderr로 보냅니다 — 그래야 `logcli stats x.log > out.txt`로 결과를 리다이렉트해도 에러 메시지가 출력 파일을 오염시키지 않습니다. `{e:#}`의 `#`는 anyhow의 **원인 체인(cause chain)**까지 펼쳐 출력하라는 뜻입니다.

**셋째, `Context`로 에러에 맥락을 더합니다.** [Chapter 13](./chapter_13.md)에서 본 `.with_context()`로 "무엇을 하다 실패했는지"를 입힙니다.

```rust
use std::fs;
use anyhow::{Context, Result};
use std::path::Path;

fn read_log(path: &Path) -> Result<String> {
    fs::read_to_string(path)
        .with_context(|| format!("failed to read {}", path.display()))
}
```

존재하지 않는 파일을 주면 사용자는 이런 메시지를 받습니다.

```text
$ logcli stats nope.log
error: failed to read nope.log: No such file or directory (os error 2)
$ echo $?
2
```

`failed to read nope.log`는 우리가 `with_context`로 더한 맥락이고, `No such file or directory (os error 2)`는 원본 `std::io::Error`입니다. anyhow가 둘을 `:` 로 연결해 한 줄로 보여줍니다. 그리고 `$?`로 확인한 종료 코드는 정확히 2입니다 — 셸 스크립트가 이 실패를 감지할 수 있습니다. Python에서 `sys.exit(2)`와 `print(..., file=sys.stderr)`를 손으로 챙기던 일을, Rust는 타입(`Result`, `ExitCode`)과 `?` 연산자로 구조화합니다.

> 💡 실무 지침: 종료 코드 관례를 따르십시오. **0 = 성공, 1 = 일반 오류, 2 = 사용법/인자 오류**가 널리 쓰입니다(BSD `sysexits.h`는 더 세분화된 코드를 정의합니다). clap의 파싱 에러가 기본 2를 쓰므로, 애플리케이션 에러는 1로 두는 선택도 합리적입니다. 핵심은 "성공과 실패를 종료 코드로 명확히 구분하고, 일관되게 유지하라"입니다.

---

## 24.7 실전 Hands-on — 로그 분석기를 실배포 CLI로 승격

이제 모든 조각을 엮어, [Chapter 14](./chapter_14.md)의 로그 분석기를 서브커맨드·파일 입출력·stdin 파이프·종료 코드를 갖춘 실배포 CLI로 키웁니다. 14장의 핵심 로직(`parse_line`, `HashMap`/`BTreeMap` 집계)을 그대로 이어받아, 그 위에 clap의 인터페이스와 anyhow의 에러 처리를 입힙니다.

먼저 `Cargo.toml`입니다.

```toml
[package]
name = "logcli"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1"
clap = { version = "4", features = ["derive", "env"] }
```

`features = ["derive", "env"]`가 핵심입니다. `derive`가 없으면 `#[derive(Parser)]`가 동작하지 않고, `env`가 없으면 `#[arg(env = ...)]`가 컴파일되지 않습니다(24.9의 함정 참조). 이제 전체 `src/main.rs`입니다. 그대로 `cargo run`·`cargo build --release` 됩니다.

```rust
use std::collections::{BTreeMap, HashMap};
use std::fs;
use std::io::{self, Read};
use std::path::PathBuf;
use std::process::ExitCode;

use anyhow::{Context, Result};
use clap::{Args, Parser, Subcommand, ValueEnum};

/// HTTP 액세스 로그를 분석하는 CLI 도구.
#[derive(Parser, Debug)]
#[command(name = "logcli", version, about = "HTTP access log analyzer")]
struct Cli {
    /// 출력 형식
    #[arg(long, value_enum, default_value_t = Format::Text, env = "LOGCLI_FORMAT")]
    format: Format,

    /// 상세 로그 (-v, -vv 로 단계 조절)
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,

    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand, Debug)]
enum Command {
    /// 상태 코드별 요청 수를 집계한다
    Stats(StatsArgs),
    /// 가장 많이 호출된 엔드포인트 Top-N
    Top(TopArgs),
}

#[derive(Args, Debug)]
struct StatsArgs {
    /// 분석할 로그 파일 (생략 시 stdin)
    file: Option<PathBuf>,
}

#[derive(Args, Debug)]
struct TopArgs {
    /// 분석할 로그 파일들 (여러 개 가능, 생략 시 stdin)
    files: Vec<PathBuf>,
    /// 상위 몇 개를 보여줄지
    #[arg(short, long, default_value_t = 3)]
    number: usize,
}

#[derive(Copy, Clone, Debug, PartialEq, Eq, ValueEnum)]
enum Format {
    Text,
    Json,
}

/// "GET /api/users 200" 한 줄을 (메서드, 경로, 상태코드)로 파싱. (Chapter 14에서 이어받음)
fn parse_line(line: &str) -> Option<(&str, &str, u16)> {
    let mut parts = line.split_whitespace();
    let method = parts.next()?;
    let path = parts.next()?;
    let status: u16 = parts.next()?.parse().ok()?;
    Some((method, path, status))
}

/// 경로가 있으면 파일을, 없으면 stdin을 읽는다 (파이프 처리).
fn read_source(path: &Option<PathBuf>) -> Result<String> {
    match path {
        Some(p) => fs::read_to_string(p)
            .with_context(|| format!("failed to read {}", p.display())),
        None => {
            let mut buf = String::new();
            io::stdin()
                .read_to_string(&mut buf)
                .context("failed to read stdin")?;
            Ok(buf)
        }
    }
}

fn cmd_stats(args: &StatsArgs, fmt: Format) -> Result<()> {
    let text = read_source(&args.file)?;
    let mut status_counts: BTreeMap<u16, u32> = BTreeMap::new();
    for line in text.lines() {
        if let Some((_m, _p, status)) = parse_line(line) {
            *status_counts.entry(status).or_insert(0) += 1;
        }
    }
    match fmt {
        Format::Text => {
            for (status, count) in &status_counts {
                println!("{status} -> {count}");
            }
        }
        Format::Json => {
            let body: Vec<String> = status_counts
                .iter()
                .map(|(s, c)| format!("\"{s}\":{c}"))
                .collect();
            println!("{{{}}}", body.join(","));
        }
    }
    Ok(())
}

fn cmd_top(args: &TopArgs, fmt: Format) -> Result<()> {
    let mut path_counts: HashMap<String, u32> = HashMap::new();
    let sources: Vec<Option<PathBuf>> = if args.files.is_empty() {
        vec![None] // 파일 없으면 stdin 하나
    } else {
        args.files.iter().cloned().map(Some).collect()
    };
    for src in &sources {
        let text = read_source(src)?;
        for line in text.lines() {
            if let Some((_m, path, _s)) = parse_line(line) {
                *path_counts.entry(path.to_string()).or_insert(0) += 1;
            }
        }
    }
    let mut ranked: Vec<(String, u32)> = path_counts.into_iter().collect();
    // 횟수 내림차순, 동률이면 경로 오름차순 (Chapter 14의 정렬 규칙)
    ranked.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    match fmt {
        Format::Text => {
            for (rank, (path, count)) in ranked.iter().take(args.number).enumerate() {
                println!("{}. {} ({})", rank + 1, path, count);
            }
        }
        Format::Json => {
            let body: Vec<String> = ranked
                .iter()
                .take(args.number)
                .map(|(p, c)| format!("[\"{p}\",{c}]"))
                .collect();
            println!("[{}]", body.join(","));
        }
    }
    Ok(())
}

fn run(cli: Cli) -> Result<()> {
    if cli.verbose > 0 {
        eprintln!("verbosity level: {}", cli.verbose);
    }
    match &cli.command {
        Command::Stats(args) => cmd_stats(args, cli.format),
        Command::Top(args) => cmd_top(args, cli.format),
    }
}

fn main() -> ExitCode {
    let cli = Cli::parse();
    match run(cli) {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("error: {e:#}");
            ExitCode::from(2)
        }
    }
}
```

`sample.log`에 14장과 같은 입력을 넣고 실행한 실제 결과입니다.

```bash
$ cat sample.log
GET /api/users 200
POST /api/users 201
GET /api/users 200
GET /api/orders 500
GET /api/users 200
POST /api/login 200
GET /api/orders 200
BROKEN LINE
GET /api/login 404
```

```text
$ logcli stats sample.log
200 -> 5
201 -> 1
404 -> 1
500 -> 1

$ cat sample.log | logcli stats          # stdin 파이프
200 -> 5
201 -> 1
404 -> 1
500 -> 1

$ logcli top -n 2 sample.log
1. /api/users (4)
2. /api/login (2)

$ logcli --format json stats sample.log
{"200":5,"201":1,"404":1,"500":1}
```

`--help`는 우리가 한 줄도 작성하지 않았는데 doc 주석에서 자동 생성됩니다.

```text
$ logcli --help
HTTP access log analyzer

Usage: logcli [OPTIONS] <COMMAND>

Commands:
  stats  상태 코드별 요청 수를 집계한다
  top    가장 많이 호출된 엔드포인트 Top-N
  help   Print this message or the help of the given subcommand(s)

Options:
      --format <FORMAT>  출력 형식 [env: LOGCLI_FORMAT=] [default: text]
                         [possible values: text, json]
  -v, --verbose...       상세 로그 (-v, -vv 로 단계 조절)
  -h, --help             Print help
  -V, --version          Print version
```

이 한 파일에 챕터의 모든 개념이 들어 있습니다. `#[derive(Parser)]`로 인자 구조를, `#[derive(Subcommand)]` enum으로 서브커맨드를, `Option<PathBuf>`/`Vec<PathBuf>`로 선택·다중 파일을, `ValueEnum`으로 `--format` 선택지를, `env`로 환경 변수를, `anyhow`로 에러 맥락을, `ExitCode`로 종료 코드를 다뤘습니다. 그리고 [Chapter 14](./chapter_14.md)의 `parse_line`과 집계 로직은 토씨 하나 안 바꾸고 그대로 재사용했습니다 — 14장에서 만든 핵심이 실배포 도구의 심장으로 자랐습니다.

---

## 24.8 생태계 크레이트 — 자동완성, 진행바, 통합 테스트

clap 하나로 견고한 CLI를 만들 수 있지만, 실배포 도구에는 몇 가지 동반 크레이트가 거의 필수입니다.

### clap_complete — 셸 자동완성

사용자가 `logcli sta<TAB>`을 누르면 `stats`로 완성되는 그 자동완성을, clap은 셸 스크립트로 생성해줍니다. 우리가 이미 인자 구조를 타입으로 갖고 있으니, 그 구조에서 bash/zsh/fish/PowerShell 완성 스크립트를 파생하는 것입니다.

```rust
use std::io;
use clap::{CommandFactory, Parser};
use clap_complete::{generate, Shell};

#[derive(Parser)]
#[command(name = "logcli")]
struct Cli {
    /// 이 셸용 자동완성 스크립트를 stdout에 출력한다
    #[arg(long, value_enum)]
    completions: Option<Shell>,
}

fn main() {
    let cli = Cli::parse();
    if let Some(shell) = cli.completions {
        let mut cmd = Cli::command();           // CommandFactory가 Command를 돌려줌
        let name = cmd.get_name().to_string();
        generate(shell, &mut cmd, name, &mut io::stdout());
    }
}
```

`logcli --completions bash`를 실행하면 실제 bash 완성 함수가 출력됩니다(발췌).

```bash
$ logcli --completions bash
_logcli() {
    local i cur prev opts cmd
    COMPREPLY=()
    if [[ "${BASH_VERSINFO[0]}" -ge 4 ]]; then
        cur="$2"
    ...
```

이 스크립트를 셸 설정에 source하거나 패키지의 completion 디렉터리에 설치하면 탭 자동완성이 동작합니다. Python의 `argcomplete`도 비슷한 일을 하지만, 매 탭마다 Python 인터프리터를 띄워야 해서 느립니다(24.1에서 본 콜드 스타트). clap은 정적 스크립트를 한 번 생성해 두므로 셸이 그것을 직접 평가합니다.

### indicatif — 진행바

큰 로그 파일 수십 개를 처리할 때 진행 상황을 보여주려면 `indicatif`를 씁니다(Python `tqdm`에 대응).

```rust
use std::time::Duration;
use indicatif::{ProgressBar, ProgressStyle};

fn main() {
    let total = 5u64;
    let pb = ProgressBar::new(total);
    pb.set_style(
        ProgressStyle::with_template("{bar:40} {pos}/{len} {msg}").unwrap(),
    );
    for i in 0..total {
        pb.set_message(format!("file {i}"));
        std::thread::sleep(Duration::from_millis(1)); // 실제로는 파일 처리
        pb.inc(1);
    }
    pb.finish_with_message("done");
}
```

`indicatif`는 터미널이 아닐 때(파이프로 리다이렉트될 때) 자동으로 출력을 억제하는 등 실무 디테일을 챙깁니다.

### assert_cmd — CLI 통합 테스트

CLI는 "프로세스를 실행하고 stdout/stderr/종료 코드를 검증"하는 식으로 테스트해야 합니다. 단위 테스트(함수 호출)만으로는 인자 파싱과 종료 코드를 검증할 수 없으니까요. `assert_cmd`가 바로 이것을 합니다([Chapter 23](./chapter_23.md)에서 테스트 전략을 더 깊이 다룹니다).

```rust
// tests/cli.rs
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn stats_from_stdin() {
    let mut cmd = Command::cargo_bin("logcli").unwrap();
    cmd.arg("stats")
        .write_stdin("GET /a 200\nGET /b 404\nGET /c 200\n")
        .assert()
        .success()
        .stdout(predicate::str::contains("200 -> 2"))
        .stdout(predicate::str::contains("404 -> 1"));
}

#[test]
fn missing_file_exits_with_code_2() {
    let mut cmd = Command::cargo_bin("logcli").unwrap();
    cmd.args(["stats", "does-not-exist.log"])
        .assert()
        .failure()
        .code(2)
        .stderr(predicate::str::contains("failed to read"));
}

#[test]
fn unknown_flag_is_rejected() {
    let mut cmd = Command::cargo_bin("logcli").unwrap();
    cmd.args(["stats", "--bogus"]).assert().failure();
}
```

`Command::cargo_bin("logcli")`가 빌드된 바이너리를 찾아 실행하고, `.write_stdin()`으로 파이프 입력을 주며, `.assert()` 체인으로 종료 코드(`.success()`/`.code(2)`)와 출력(`predicate::str::contains`)을 검증합니다. `cargo test`로 돌린 실제 결과입니다.

```text
running 3 tests
test stats_from_stdin ... ok
test missing_file_exits_with_code_2 ... ok
test unknown_flag_is_rejected ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured
```

CLI의 핵심 계약(인자 파싱, 종료 코드, stdin 파이프, 에러 메시지)을 모두 프로세스 레벨에서 검증했습니다. Python에서 `subprocess.run`으로 CLI를 띄워 출력을 검증하던 그 패턴을, `assert_cmd`가 타입 안전한 빌더로 제공합니다.

마지막으로 Python 도구와 clap 생태계를 한눈에 비교합니다.

| 관심사 | Python | Rust (clap 생태계) |
|--------|--------|--------------------|
| 인자 파싱 (명령형) | `argparse` | clap builder API |
| 인자 파싱 (선언형) | `click` 데코레이터 | clap derive API |
| 인자 검증 시점 | 런타임 | 런타임(입력) + 컴파일 타임(구조) |
| 서브커맨드 분기 누락 | 잡지 못함 | `match` 망라성 (E0004) |
| 셸 자동완성 | `argcomplete` (느림) | `clap_complete` (정적 스크립트) |
| 진행바 | `tqdm` | `indicatif` |
| CLI 통합 테스트 | `subprocess` + 수동 검증 | `assert_cmd` + `predicates` |
| 배포물 | 인터프리터 + 의존성 | 단일 정적 바이너리 (~1MB) |
| 콜드 스타트 (이 책 측정) | ~38ms (argparse) | ~2.5ms (release) |

---

## 24.9 Best Practices & 안티패턴

실무 체크리스트입니다.

**이렇게 하라**

- **`Cargo.toml`에 `features = ["derive"]`를 빠뜨리지 마라.** clap의 기본 의존성에는 derive 매크로가 포함되지 않습니다. 환경 변수를 쓰면 `"env"`도 추가합니다. 이 한 줄을 빠뜨려 시작부터 막히는 경우가 가장 흔합니다(아래 함정 1).
- **파일 경로는 `PathBuf`/`Option<PathBuf>`로 받아라.** `String`이 아닙니다. OS 네이티브 인코딩 보존과 경로 API를 공짜로 얻습니다(24.5).
- **서브커맨드는 `enum` + `match`로 디스패치하라.** 변형 추가 시 `match` 망라성이 분기 누락을 컴파일 타임에 잡습니다. 와일드카드 `_ =>`를 남용하면 이 안전장치가 무력화되니 주의하십시오.
- **에러는 stderr로, 출력은 stdout으로.** `eprintln!`(에러) vs `println!`(결과)을 명확히 분리해야 파이프와 리다이렉트가 깨지지 않습니다.
- **종료 코드를 의도적으로 설계하라.** `main() -> ExitCode`로 성공/실패를 명시하고, 관례(0/1/2)를 일관되게 지키십시오.
- **`assert_cmd`로 통합 테스트를 작성하라.** 인자 파싱·종료 코드·stdin 파이프는 프로세스 레벨에서만 검증됩니다([Chapter 23](./chapter_23.md)).
- **doc 주석을 help로 활용하라.** `///` 주석이 그대로 `--help`가 되니, 문서와 도움말을 한 소스로 유지하십시오.

**이건 피하라(안티패턴)**

- **builder API와 derive API를 어설프게 섞기** — 둘 다 가능하고 혼용도 되지만, 작은 CLI는 derive 하나로 끝내는 게 읽기 쉽습니다. 동적으로 인자를 생성해야 하는 드문 경우에만 builder를 섞으십시오.
- **`unwrap()`으로 에러를 패닉으로 흘리기** — CLI가 패닉하면 사용자는 Rust 백트레이스를 봅니다. 이건 버그처럼 보입니다. 예상 가능한 실패(파일 없음 등)는 `Result`로 처리해 깔끔한 메시지를 주십시오.
- **결과를 stdout, 에러도 stdout** — 파이프라인을 깨는 전형적 실수입니다.
- **모든 옵션을 필수로** — `Option<T>`와 `default_value`를 적극 활용해, 흔한 경우는 인자 없이도 동작하게 하십시오.

`clippy`는 여기서도 도움이 됩니다. `cargo clippy`를 켜면 불필요한 `clone`, 비효율적 패턴 등을 잡아주고, 이 챕터의 logcli는 clippy 경고 0개로 빌드됩니다.

---

## 24.10 흔한 함정 — Python 습관이 부르는 사고

**함정 1 — derive feature를 빠뜨린다.**

가장 흔한 시작 장애물입니다. `clap = "4"`만 적고 `#[derive(Parser)]`를 쓰면 컴파일이 거부됩니다.

```text
error: cannot find derive macro `Parser` in this scope
 --> src/main.rs:3:10
  |
3 | #[derive(Parser)]
  |          ^^^^^^
  |
note: `Parser` is imported here, but it is only a trait, without a derive macro
 --> src/main.rs:1:5
  |
1 | use clap::Parser;
  |     ^^^^^^^^^^^^
```

`Parser`는 트레이트로는 import됐지만, 그것을 자동 구현해주는 **파생 매크로**는 `derive` feature 뒤에 숨어 있습니다. 해법은 `Cargo.toml`을 `clap = { version = "4", features = ["derive"] }`로 고치는 것입니다. 같은 이유로 `env = "..."`를 쓰면 `"env"` feature를, `clap_complete`를 쓰면 그 크레이트를 추가해야 합니다. Python의 `pip install click` 한 줄에 익숙한 사람에게는 "왜 feature를 따로 켜야 하지?" 싶지만, 이것은 [Chapter 2](./chapter_02.md)에서 본 Cargo의 feature 시스템 — 쓰지 않는 기능은 컴파일하지 않아 빌드 시간과 바이너리 크기를 아끼는 설계 — 의 일부입니다.

**함정 2 — 경로를 String으로 받는다.**

24.5에서 본 그 함정입니다. Python에서 경로는 그냥 `str`이라, 무심코 `file: String`으로 받으면 비-UTF-8 경로에서 정보가 깨지고, `&Path`가 필요한 곳에서 `error[E0308]: mismatched types`를 만납니다. 처음부터 `PathBuf`로 받으십시오. clap이 1급으로 지원하니 타입만 바꾸면 됩니다.

**함정 3 — stdin 파이프를 잊는다.**

유닉스 CLI의 미덕은 파이프 조립입니다. `cat huge.log | logcli stats`가 동작해야 하고, `logcli stats file.log`도 동작해야 합니다. Python에서 `sys.stdin.read()`와 파일 읽기를 분기하듯, Rust에서도 `Option<PathBuf>`가 `None`이면 stdin을, `Some`이면 파일을 읽도록 분기해야 합니다(24.7의 `read_source`). 이걸 잊으면 파이프라인의 시민이 되지 못합니다. 실전 예제의 `read_source`가 이 패턴의 정석입니다.

**함정 4 — 서브커맨드 디스패치에 `_ =>`를 남발한다.**

`match`의 망라성 검사가 분기 누락을 잡아주는데, 귀찮다고 `_ => {}` 와일드카드를 넣으면 그 안전장치가 꺼집니다. 새 서브커맨드를 추가해도 `_`가 조용히 삼켜버립니다 — Python의 `if/elif` 누락과 똑같은 버그가 되살아납니다. 서브커맨드 디스패치에서는 **모든 변형을 명시적으로 나열**해, 컴파일러가 빠짐을 잡게 두십시오. `todo!()`로 임시 처리를 남기더라도 와일드카드보다 낫습니다.

**함정 5 — 결과와 에러를 한 스트림에 섞는다.**

Python에서 `print(error)`로 에러를 stdout에 찍던 습관이 그대로 넘어오면, `logcli stats x.log | jq`가 에러 메시지까지 jq에 먹여 깨집니다. 결과는 `println!`(stdout), 에러·진단은 `eprintln!`(stderr)로 엄격히 분리하십시오. 24.6에서 본 `eprintln!("error: {e:#}")`가 그 원칙입니다.

이 다섯 함정을 관통하는 교훈은 하나입니다. Python CLI는 "느슨하게 만들어도 일단 돌아가는" 편의의 영역이었지만, Rust CLI는 **타입과 컴파일러가 인터페이스의 정합성을 지키는 구조물**입니다. 처음에는 feature 플래그나 `PathBuf` 같은 제약이 성가시게 느껴지지만, 그 제약 하나하나가 — 서브커맨드 누락, 경로 인코딩 깨짐, 파이프 오염 같은 — 사용자가 마주칠 버그를 빌드 단계에서 봉쇄한 것입니다.

---

## 24.11 연습문제

읽기만 해서는 손에 붙지 않습니다. 아래 문제들은 직접 컴파일·실행해 자기 검증하도록 설계했습니다.

### 문제 ① — 필드 타입으로 인자 성질 예측하기

다음 `struct`가 만드는 CLI의 인자 성질을 **먼저 손으로 예측**한 뒤, 실제로 빌드해 `--help`로 맞춰 보십시오. 각 필드가 필수인지/선택인지/다중인지, 그리고 기본값은 무엇인지 답하십시오.

```rust
#[derive(Parser)]
struct Cli {
    input: PathBuf,                                  // (a)
    output: Option<PathBuf>,                         // (b)
    #[arg(short, long)]
    exclude: Vec<String>,                            // (c)
    #[arg(long, default_value_t = false)]
    dry_run: bool,                                   // (d)
}
```

> **정답 스케치**
> (a) `input` — 타입에 `Option`/`Vec`이 없고 기본값도 없으므로 **필수 위치 인자**. (b) `output` — `Option<PathBuf>`이므로 **선택 위치 인자**(없으면 `None`). (c) `exclude` — `Vec<String>` + `short, long`이므로 `-e/--exclude`를 **여러 번** 줄 수 있는 다중 옵션(0개 이상). (d) `dry_run` — `bool`이라 플래그이고, `--dry-run`을 주면 `true`(필드명의 `_`가 `--dry-run`으로 변환됨). 핵심: 인자의 성질이 키워드 인자가 아니라 **필드 타입**에서 파생된다는 점을 확인하십시오.

### 문제 ② — E0004로 서브커맨드 누락 잡기

아래 `enum`에 `Validate` 변형을 추가하되, `match`에는 처리를 넣지 마십시오. 빌드하면 어떤 에러가 나는지 **에러 코드를 먼저 맞히고**, 그 에러가 왜 Python `argparse`에서는 발생하지 않는지 한 줄로 설명하십시오.

```rust
#[derive(Subcommand)]
enum Command { Stats, Top }

fn dispatch(c: &Command) {
    match c {
        Command::Stats => {}
        Command::Top => {}
    }
}
```

> **정답 스케치**
> `Validate`를 추가하면 `error[E0004]: non-exhaustive patterns: \`&Command::Validate\` not covered`가 납니다. 컴파일이 거부됩니다. Python에서는 서브커맨드 디스패치가 평범한 `if/elif` 문자열 비교라, 분기를 빠뜨려도 인터프리터가 검증할 방법이 없습니다(서브커맨드가 타입이 아니라 런타임 문자열이므로). Rust는 서브커맨드를 `enum` 타입으로 표현해, 타입 검사기가 모든 변형 처리 여부를 컴파일 타임에 강제합니다.

### 문제 ③ — 종료 코드와 stderr 분리 검증하기

`logcli stats /no/such/file 2>/dev/null` 와 `logcli stats /no/such/file >/dev/null` 의 출력 차이를 예측하고, `echo $?`로 확인되는 종료 코드를 답하십시오. 두 리다이렉트가 각각 무엇을 버리는지 설명하십시오.

> **정답 스케치**
> 에러 메시지(`error: failed to read ...`)는 **stderr**로 갑니다. `2>/dev/null`은 stderr를 버리므로 **아무것도 안 보이고**, `>/dev/null`(= `1>/dev/null`)은 stdout만 버리므로 **에러 메시지는 여전히 보입니다**. 종료 코드는 두 경우 모두 **2**(`echo $?` → 2)입니다 — 리다이렉트는 종료 코드에 영향을 주지 않습니다. 핵심: 결과(stdout)와 진단(stderr)을 분리했기 때문에 셸이 둘을 독립적으로 다룰 수 있다는 점입니다.

### 문제 ④ — value_parser로 검증 옮기기

`-n` 옵션을 `usize`로 받되, 1 이상 1000 이하만 허용하도록 `value_parser`를 추가하십시오. 그리고 범위를 벗어난 값을 줬을 때 (1) 종료 코드와 (2) 우리 코드의 본문이 실행되는지/안 되는지를 답하십시오.

> **정답 스케치**
> `#[arg(short, long, value_parser = clap::value_parser!(usize).range(1..=1000))]`. 범위 밖 값(`-n 2000`)을 주면 clap이 **파싱 단계에서** 거부하고 종료 코드 **2**로 끝납니다. 우리의 `cmd_top` 본문은 **실행되지 않습니다** — 검증이 인자 정의에 붙어 있어, 유효한 값만 우리 로직에 도달하기 때문입니다. 이것이 "검증을 타입/파서 경계로 끌어올린다"의 실제 효과입니다(Python에서는 파싱 후 `if`로 손수 검사해야 함).

---

CLI는 Rust가 Python 대비 가장 즉각적이고 논쟁의 여지 없는 우위를 보이는 영역 중 하나입니다. 단일 정적 바이너리라는 배포 모델, 15배 빠른 콜드 스타트, 그리고 무엇보다 인자 구조가 타입으로 존재해 서브커맨드 누락·경로 인코딩·검증 시점 같은 버그 클래스를 컴파일 타임에 봉쇄한다는 점 — 이 모두가 "한 번 실행되고 죽는 작은 도구"라는 CLI의 본질과 정확히 맞아떨어집니다. 그래서 여러분이 매일 쓰는 `ruff`, `uv`, `rg`, `fd`, `bat` 같은 도구들이 약속이라도 한 듯 Rust로 만들어진 것입니다([Chapter 1](./chapter_01.md)에서 본 그 수렴입니다).

이번 챕터에서 우리는 [Chapter 14](./chapter_14.md)의 로그 분석기를, [Chapter 10](./chapter_10.md)의 enum 망라성과 [Chapter 12](./chapter_12.md)의 `Option`, [Chapter 13](./chapter_13.md)의 `anyhow`를 엮어 실배포 CLI로 키웠습니다. 도구는 만들었으니, 이제 그것을 어떻게 견고하게 유지하고 세상에 내보낼지가 남았습니다. 다음 [Chapter 23](./chapter_23.md)에서는 이 CLI를 포함한 Rust 코드 전반의 **테스트 전략** — 단위 테스트, 통합 테스트, 그리고 방금 맛본 `assert_cmd`까지 — 을 본격적으로 다루며, 프로덕션으로 가는 길의 다음 관문을 통과합니다.

---

## References

- https://docs.rs/clap/latest/clap/
- https://docs.rs/clap/latest/clap/_derive/index.html
- https://docs.rs/clap/latest/clap/_tutorial/index.html
- https://docs.rs/clap_complete/latest/clap_complete/
- https://docs.rs/indicatif/latest/indicatif/
- https://docs.rs/assert_cmd/latest/assert_cmd/
- https://docs.rs/anyhow/latest/anyhow/
- https://doc.rust-lang.org/std/process/struct.ExitCode.html
- https://doc.rust-lang.org/std/path/struct.PathBuf.html
