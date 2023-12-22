# 함수의 선언부가 가지는 의미

2023.12.20

---

오늘은 어렵지만, Rust를 깊게 이해하기 위해 도움이 될 만한 내용을 소개해 보려 한다. Rust에서는 함수의 선언부가 특별한 의미를 가진다. 먼저 아래의 코드를 살펴보자.

```rust
# fn main() {}
//반환 위치에 placeholder를 써서 타입 추론 시도
fn test() -> _ {
    String::from("asdf")
}
```

결과는 아래와 같다. 컴파일되지 않는다! Rust를 오래동안 해온 사람들에게는 당연하다고 느껴지지만 왜 안되는지는 생각해보지 않았을 것이다.

```
error[E0121]: the placeholder `_` is not allowed within types on item signatures for return types
 --> src/lib.rs:1:14
  |
1 | fn test() -> _ {
  |              ^
  |              |
  |              not allowed in type signatures
  |              help: replace with the correct return type: `String`
```

다음과 같은 상황을 가정해 보자.

```rust
fn main() {
    //testing 모듈의 test를 사용
    let s: String = testing::test();
}

//다른 crate의 모듈이라 가정
mod testing {
    pub fn test() -> _ {
        String::from("asdf")
    }    
}
```

위 코드는 실제 컴파일되지 않는 코드이다. 만약 위와 같은 상황에서 `testing`모듈의 사용자와 컴파일러는 `test()`함수의 반환 타입을 바로 알 수 없다. 반환 타입을 알려면 `testing`모듈의 소스코드를 직접 읽어봐야 한다. 이와 같은 이유로, Rust에서 함수의 선언부는 일종의 계약, 또는 인터페이스의 기능을 한다. 함수의 선언부는 명시적이어야 한다. 모호해서는 안 된다. 다만 예외는 있다. lifetime elision rule과 anonymous lifetime이 있다. lifetime generic parameter를 생략하게 해주는 아주 고마운 기능이다. anonymous lifetime은 함수의 매개변수나 반환값에 generic lifetime parameter를 받는 구조체를 넘길 때 lifetime elision rule을 적용할 수 있게 해준다. 사실, 이 둘을 적용하더라도 lifetime parameter가 아얘 없는 것이 아니라 컴파일러가 적절한 lifetime parameter를 추가해준다. 아래의 예시를 참고하자.

**lifetime elision rule 적용 전**
```rust
fn test(s: &str) -> &str { s }
```

**lifetime elision rule 적용 후(컴파일러에 의해 자동 적용)**
```rust
fn test<'a>(s: &'a str) -> &'a str { s }
```

**anonymous lifetime 적용 전(프로그래머가 직접 lifetime 지정)**
```rust
struct Container<'a>(&'a str);
fn test<'a>(s: &'a str) -> Container<'a> { Container(s) }
```

**anonymous lifetime 적용 후(컴파일러에 의해 추론됨)**
```rust
struct Container<'a>(&'a str);
fn test(s: &str) -> Container<'_> { Container(s) }
```

즉, Rust에서 모든 함수의 선언부에 위치하는 reference는 lifetime 관계를 갖는다. 많은 경우는 lifetime elision rule에 의해 컴파일러가 자동으로 채워주기 때문에 프로그래머가 직접 lifetime을 지정하지 않아도 되는 경우가 많다.

Rust에서 함수의 선언부는 일종의 abstraction boundary(추상화 경계)로 생각할 수 있다. Lifetime, borrowing 규칙은 이 경계를 기준으로 따로 적용된다. 아래에서 몇가지 예를 들어볼 것이다.

## Partial borrow

```rust
fn main() {
    let mut a = A { x: 5, y: 10 };
    //a.x를 mutable borrow
    let x = &mut a.x;
    //a.y를 mutable borrow
    let y = &mut a.y;
    //x를 사용
    *x = 5;
}

struct A {
    x: usize,
    y: usize,
}
```

위의 코드를 보자. Rust는 Partial borrow를 지원한다. 구조체에서 각각의 field를 따로 빌려올 수 있다. 위의 코드를 getter 형태로 바꿔보면 아래와 같다.

```rust
fn main() {
    let mut a = A { x: 5, y: 10 };
    //a.x를 mutable borrow
    let x = a.x_mut();
    //a.y를 mutable borrow
    let y = a.y_mut();
    //x를 사용
    *x = 5;
}

struct A {
    x: usize,
    y: usize,
}

impl A {
    fn x_mut(&mut self) -> &mut usize {
        &mut self.x
    }
    
    fn y_mut(&mut self) -> &mut usize {
        &mut self.y
    }
}

```

위 코드는 컴파일 오류를 발생한다.

```
error[E0499]: cannot borrow `a` as mutable more than once at a time
 --> src/main.rs:6:13
  |
4 |     let x = a.x_mut();
  |             - first mutable borrow occurs here
5 |     //a.y를 mutable borrow
6 |     let y = a.y_mut();
  |             ^ second mutable borrow occurs here
7 |     //x를 사용
8 |     *x = 5;
  |     ------ first borrow later used here
```

오류 메세지를 살펴보자. `a`를 여러번 mutable로 borrow할 수 없다고 한다. 위 코드의 `x_mut()`, `y_mut()` 함수에서는 분명히 `x`, `y`만 mutable borrow한다. 컴파일러는 함수를 호출할 때 함수의 내부 구현을 보지 않고 선언부만 확인한다. 이 함수들은 매개변수로 `&mut self`를 받는다. 따라서, 이 함수를 호출하면 실제 사용되는 field인 `x` 또는 `y`만 borrow되는 것이 아니라 `self` 전체를 mutable borrow한다. 그리고 `&mut self`를 reborrow(다시 빌려서)해서 각각의 field `x`, `y`의 mutable reference를 반환한다.
즉, 함수의 내부 구현은 호출되는 부분과는 별개이다. 이 둘을 서로 연결시켜주는 것은 함수의 선언부이다.
이는 Rust 언어의 한계점이기도 하다. 아직까지는 함수에 넘어가는 `&self`에서 특정 field만 빌려오는 문법이 없다.
이러한 방식은 Rust 프로그래머와 컴파일러가 함수의 입력과 출력에 대해 쉽게 예측할 수 있도록 한다. 함수를 사용하는 곳과, 함수의 내부는 각각 공통된 함수의 선언부의 규약에 맞게 행동해야 하고, 컴파일러는 이를 검사하게 된다. 만약, `x_mut()`에서 `self`보다 lifetime이 짧은 레퍼런스를 반환하면 해당 함수 내에서 오류가 발생한다. `main`함수에서 `a`가 mutable borrow된 상태에서 `x_mut()`를 호출하려고 하면 borrowing rule에 의해 컴파일 오류가 발생한다.

## generic lifetime parameter

컴파일러는 함수의 선언부만 보고 lifetime을 검사한다. 아래의 코드를 보자. 

```rust
fn main() {
    let s1 = "12345";
    let s2 = "1234567";
    let longest = get_longest(s1, s2);
    println!("{longest}");
}

fn get_longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() < b.len() {
        b
    } else {
        a
    }
}
```

위의 `get_longest()` 함수에서는 어떤 값이 반환될 지 실행하기 전까지는 알 수 없다. `a`, `b` 중 길이가 긴 문자열 슬라이스의 레퍼런스가 반환된다. 위의 경우와 같이, 함수의 매개변수로 2개 이상의 reference type이 있을때는 lifetime elision rule이 적용되지 않는다. 즉, generic lifetime(`'a`)을 생략할 수 없다. 왜냐하면, 매개변수로 들어오는 두 개 이상의 lifetime중 어떤 lifetime이 반환되는지 모호하기 때문이다.

위 코드의 컴파일 단계는 크게 두 단계로 나눌 수 있다.

**1. `get_longest()` 함수의 컴파일**

먼저, 컴파일러는 함수의 선언부를 확인해서 다음의 정보를 얻는다. 함수는 `'a` generic lifetime parameter를 갖고, 매개변수로 `a: &'a str, b: &'a str`를 받고, `&'a str`을 반환한다. 즉, `a`와 `b`의 공통 lifetime 을 반환해야 한다는 정보를 파악한다. 그리고, 문자열 슬라이스 `a`와 `b`를 immutable borrow 하고, 이것을 reborrow해서 문자열 슬라이스를 반환한다. 이 정보를 토대로, 함수가 반환하는 reference의 lifetime이 `'a`에 들어가는지 검사한다. 여기서 `'static` lifetime을 반환할 수도 있다. variance 규칙에 의해서 `'static` 은 모든 lifetime중 가장 길기 때문에 `'a`를 대체할 수 있다. 그러나 아래와 같은 코드는 컴파일이 되지 않는다.

```rust
fn get_longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    let s = "asdf".to_owned();
    &s
}
```

위 코드는 컴파일 오류가 발생한다. 위의 코드에서 `s`의 lifetime은 이 함수 자체의 scope이다. `s`의 lifetime은 함수를 반환할 때 종료된다. 따라서 `'a`와 연결되어 있지 않기 때문에 컴파일이 되지 않는다. 만약에 컴파일이 된다면 drop된 값의 reference를 반환하는 꼴이기 때문에 use after free 메모리 위반이 발생한다.
따라서 이 함수가 컴파일되는 과정에서 함수의 선언부에 정의된 규약과 제약사항이 검사된다. 이 함수를 호출하는 사람은 내부 구현을 알 필요 없이 함수의 선언부만 보고도 lifetime과 borrowing에 대한 정보를 알 수 있다. 함수의 내부 구현을 숨기고 선언부만 보고 안전하게 호출할 수 있기에 이를 safe abstraction의 관점에서도 바라볼 수 있다.

**1. `main()` 함수의 컴파일**

`main` 함수에서 `get_longest()` 함수가 호출될 때를 살펴보자. `get_longest()` 함수가 호출될 때 컴파일러는 이 함수의 내부를 분석하여 lifetime과 borrowing 규칙을 검사하지 않는다. 이 함수의 선언부만 확인한다. 이 함수는 `'a` generic lifetime parameter를 갖고, 매개변수로 `a: &'a str, b: &'a str`를 전달받고, `&'a str`을 반환 받는다는 정보를 파악한다. 또한, generic lifetime parameter인 `'a`를 보고, 매개변수로 전달되는 `a`와 `b`의 lifetime이 반환되는 문자열 슬라이스의 lifetime과 같거나 더 길어야 한다는 정보를 파약한다. 함수의 선언부만 보고도 이러한 정보를 얻을 수 있다. `get_longest()`가 호출되고 나면, 문자열 `s1`, `s2`는 반환되는 문자열 슬라이스인 `&'a str`로 reborrow 된다. 함수의 선언부에 정의된 generic lifetime parameter `'a`에 의해 입력과 출력 레퍼런스의 라이프타입과 borrowing 관계가 결정된다.
따라서, `println!("{longest}");`가 실행될 때, `longest`를 reborrow해준 부모 레퍼런스 `s1`, `s2`가 살아있기 때문에 정상적으로 컴파일된다.
`get_longest()`함수가 호출될 때, 이 함수의 선언부만 보고 이러한 정보를 파악하여 lifetime과 borrowing rule을 체크한다.

위의 과정과 같이 Rust는 함수의 선언부를 기준으로 컴파일한다는 것을 알 수 있다. 위에서 말했던 것과 같이, Rust에서 함수의 선언부는 일종의 추상화로 볼 수 있다. 함수를 사용하는 쪽과 작성하는 쪽이 이 선언부에 정의된 규칙을 모두 따라야 한다. 컴파일러는 함수를 사용하는 쪽이 올바르게 사용하고 있는지, 내부가 올바르게 구현되어 있는지 위에서 2단계로 설명한 거와 같이 꼼꼼히 확인한다. 따라서, 함수를 사용하는 쪽과 함수를 구현하는 쪽 둘 중 어느 한 곳이라도 잘못된 행동을 하면 컴파일 오류가 발생한다.

## 결론
1. 컴파일러는 함수의 선언부에 정의되어 있는 규칙을 기준으로 하여, 컴파일 시 함수를 사용하는 쪽, 함수를 구현하는 쪽 각각 따로 검사한다.
2. 컴파일러는 함수가 호출되는 부분을 컴파일할 때 해당 함수의 내부를 확인하지 않는다. 해당 함수의 내부는 해당 함수가 컴파일될 때 별개로 확인된다.
3. 함수의 선언부만 보고도 함수의 입력, 출력의 lifetime과 borrowing이 어떻게 연결되는지 파악할 수 있다. 따라서, 내부 구현을 몰라도 함수를 안전하게 호출하는데 큰 기여를 한다.

오늘은 그동안 Rust를 공부하면서 중요하다고 생각했던 내용을 정리해 봤다. 내가 Rust를 처음 공부했을 때는 lifetime과 borrowing 규칙을 이해하는것만 해도 버거웠던 거 같다. 이러한 규칙이 함수의 선언부를 기준으로 적용된다는 사실을 최근에야 알게 되었는데, Rust를 공부하는 모든 사람이 알면 좋을것 같은 내용이라서 이렇게 정리해 봤다. 앞으로 Rust를 사용할 때 컴파일러의 입장에 한 발작 더 다가갈 수 있을거 같다.

## 참고하면 좋을 학습자료
* [Rust's Golden Rule](https://web.archive.org/web/20230412050320/https://steveklabnik.com/writing/rusts-golden-rule)
* [Lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html)