# Chapter 6. 수명 (Lifetimes): 컴파일러에게 의도 설명하기

파이썬에는 '수명(Lifetime)'이라는 명시적인 개념이 없습니다. 객체는 누군가 참조하고 있는 한 계속 살아있고, 참조하는 이가 아무도 없어질 때(Reference Count == 0) 비로소 메모리에서 사라집니다. 파이썬 개발자는 변수가 메모리에 언제까지 남아있을지 굳이 계산할 필요가 없습니다.

하지만 Rust는 가비지 컬렉터(GC) 없이 컴파일 타임에 모든 메모리의 해제 시점을 결정해야 합니다. "참조(Reference)를 빌려줬는데, 원본 데이터가 먼저 메모리에서 해제되어 버리면 어떡하지?" 이 끔찍한 런타임 버그(Dangling Pointer)를 막기 위해 Rust 컴파일러가 도입한 개념이 바로 **수명(Lifetimes)**입니다.

---

## 6.1 허공을 가리키는 포인터 (Dangling Pointer)

만약 아래와 같은 상황이 발생한다면 어떻게 될까요? 파이썬이라면 예외가 발생하거나 알아서 GC가 처리해주겠지만, C/C++에서는 쓰레기 메모리 값을 읽어오면서 프로그램이 죽거나 해킹에 취약해집니다.

```rust
fn main() {
    let r;                // 1. 참조 변수 r을 선언합니다.
    {
        let x = 5;        // 2. x는 이 내부 스코프에서만 살아있습니다.
        r = &x;           // 3. r이 x를 참조(빌려옴)합니다.
    }                     // 4. x의 스코프가 끝나고 메모리에서 버려집니다(Dropped).

    // println!("r: {}", r); // 5. 🚨 컴파일 에러! r은 이미 죽어버린 x를 가리키고 있습니다.
}
```

Rust 컴파일러(Borrow Checker)는 `x`의 수명이 `r`의 수명보다 짧다는 것을 컴파일 타임에 알아채고 에러를 뱉습니다. "빌려온 참조(`r`)는 원본 데이터(`x`)보다 오래 살아남을 수 없다"는 가장 기본적인 수명 규칙입니다.

---

## 6.2 함수에서의 수명: 컴파일러의 한계

단일 함수 내에서는 컴파일러가 변수의 스코프를 쉽게 추적합니다. 하지만 참조를 매개변수로 받는 함수를 만들 때는 문제가 복잡해집니다.

* **파이썬에서의 문자열 비교**
  ```python
  def longest(x: str, y: str) -> str:
      return x if len(x) > len(y) else y
      
  # 런타임에 x나 y 중 하나가 반환되며, 가비지 컬렉터가 알아서 메모리를 관리함.
  ```

* **Rust로 직역한 코드 (컴파일 실패)**
  ```rust
  // 🚨 컴파일 에러 발생: "missing lifetime specifier"
  fn longest(x: &str, y: &str) -> &str {
      if x.len() > y.len() {
          x
      } else {
          y
      }
  }
  ```

**왜 에러가 날까요?**
Rust 컴파일러는 위 함수를 보고 패닉에 빠집니다. 반환되는 참조(`&str`)가 매개변수 `x`를 가리킬지, `y`를 가리킬지 런타임이 되기 전에는 알 수 없기 때문입니다. 컴파일러 입장에서는 "반환된 참조가 도대체 언제까지 메모리에 살아있을지 확신할 수 없으니 빌드를 멈추겠다"고 선언하는 것입니다.

---

## 6.3 수명 명시 (Lifetime Annotation)

우리는 컴파일러에게 **"이 참조들이 최소한 언제까지는 같이 살아있을 거야"**라고 의도를 명시적으로 적어주어야 합니다. 이것을 수명 명시(Lifetime Annotation)라고 부르며, 어포스트로피(`'`)를 사용하여 표기합니다. (주로 `'a`, `'b` 처럼 알파벳을 씁니다.)

```rust
// "x와 y, 그리고 반환값은 모두 최소한 수명 'a 만큼은 살아있어야 한다"고 명시
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

이제 컴파일러는 안심합니다. 함수를 호출하는 쪽에서 넘겨주는 `x`와 `y` 중 **더 수명이 짧은 쪽의 수명**을 `'a`로 간주하고, 반환된 값이 그보다 오래 쓰이지 않는지만 검사하면 되기 때문입니다.

### 수명은 값을 더 살려두지 않습니다
주의할 점은, 수명 명시를 적는다고 해서 변수가 메모리에 더 오래 살아남는 것은 아니라는 점입니다. 단지 "컴파일러에게 여러 참조 간의 **수명 관계(관계성)**를 설명해 주는 힌트"일 뿐입니다. 

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    
    {
        let string2 = String::from("xyz");
        // 컴파일 에러! 
        // string2의 수명이 더 짧으므로, 반환값 result의 수명도 string2와 같아집니다.
        // 하지만 result는 바깥 스코프에서 프린트되고 있습니다.
        // result = longest(string1.as_str(), string2.as_str()); 
    }
    // println!("The longest string is {}", result); 
}
```

---

## 6.4 구조체(Struct)와 수명

구조체 안에 데이터의 '원본'이 아니라 '참조'를 저장하고 싶을 때도 반드시 수명을 명시해야 합니다. 파이썬에서는 객체 안에 다른 객체를 마음대로 넣었지만, Rust에서는 구조체가 자기가 들고 있는 참조 원본보다 오래 살아남는 것을 막아야 하기 때문입니다.

```rust
// ImportantExcerpt 구조체는 자기가 들고 있는 part 참조보다 
// 더 오래 살아남을 수 없음을 명시
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    
    // 구조체 i가 생성됨. first_sentence의 참조를 들고 있으므로,
    // i는 novel이 메모리에서 해제되기 전에 파괴되어야 함.
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

---

## 6.5 정적 수명 (`'static`)

파이썬의 모듈 레벨 전역 변수처럼, 프로그램이 시작해서 끝날 때까지 영원히 살아있는 특별한 수명이 있습니다. 바로 `'static` 입니다. 우리가 코드에 하드코딩한 문자열 리터럴은 모두 `'static` 수명을 가집니다.

```rust
let s: &'static str = "I have a static lifetime.";
```

수명(Lifetime)은 C/C++ 출신 개발자들조차 낯설어하는 Rust만의 고유한 문법입니다. 처음에는 모든 곳에 `'a`를 붙여야 할 것 같아 두렵겠지만, 다행히 Rust 컴파일러는 자주 쓰이는 수명 패턴을 스스로 추론(Lifetime Elision)하여 우리가 일일이 수명을 적지 않아도 되도록 진화해 왔습니다. 

지금까지 우리는 Rust가 메모리를 지배하는 방식(소유권, 참조, 수명)을 모두 배웠습니다. 이 단단한 기반 위에서, 다음 장부터는 파이썬의 풍부한 객체 지향 문법(클래스, 상속, 매직 메서드)이 Rust에서 데이터 모델(Struct/Trait)로 어떻게 탈바꿈하는지 살펴보겠습니다.
