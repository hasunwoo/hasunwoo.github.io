# Generic 매개변수의 lifetime 처리

2023.12.20

---

아래의 코드는 표면적으로는 아무 문제가 없어 보인다. 그러나 컴파일해보면 오류가 발생한다.

```rust
fn main() {
    let owned = "asdf".to_owned();
    let borrowed = returns_asref(&owned);
    println!("{borrowed}");
}

fn returns_asref<'a>(t: impl AsRef<str> + 'a) -> &'a str {
    t.as_ref()
}
```

아래는 컴파일 에러 메세지이다.

```
error[E0515]: cannot return value referencing function parameter `t`
 --> src/main.rs:8:5
  |
8 |     t.as_ref()
  |     -^^^^^^^^^
  |     |
  |     returns a value referencing data owned by the current function
  |     `t` is borrowed here
```

컴파일 에러 메세지를 자세히 살펴보자. `t`가 borrow되었고, 이 함수가 소유하고 있는 값의 참조를 반환했다고 한다. 컴파일러는 `t`를 reference type이 아닌, owned type으로 취급하고 있다. 먼저, 함수의 선언부가 가지는 의미에 대해 알면 이 문제 해결에 도움이 된다.

## 함수의 선언부(Signature)는 계약이다

함수의 선언보는 함수와 호출자와의 계약의 역할을 한다. 컴파일러는 함수의 선언부를 기준으로 동작한다. 함수가 호출될 때, 컴파일러는 호출되는 함수의 선언부를 참고하여 각종 규칙(ownership, borrowing, lifetime, type)을 검사하게 된다. Rust를 처음 접하는 사람들은 함수를 호출할 때, 컴파일러가 해당 함수의 코드를 보고 각종 규칙을 적용한다고 생각할 수 있다. 그러나 컴파일러는 의외로 단순하게 작동한다. 함수의 선언부만 보고 이러한 규칙들을 검사한다. 이 사실을 알면 Rust가 어떻게 작동하는지에 대해 더 깊게 이해할 수 있다. 이러한 이유 때문에 Rust에서는 함수의 반환 타입으로 type inference(_;placeholder)를 사용할 수 없다. 왜냐하면 함수의 선언부가 명확해야 선언부를 기준으로 동작하기 때문이다.

위 내용은 중요한 내용이므로 별도의 글로 자세히 다뤄 보았다.
[보러 가기](../tips/231220-meaning-of-function-signature.md)

먼저 `main`함수부터 분석해 보자.

## main 함수 분석

```rust
fn main() {
    //"asdf" string slice에 to_owned()를 호출하여 owned type으로 변환(String)
    let owned = "asdf".to_owned();
    //returns_asref() 함수에 owned 값을 빌려주고, &str를 받음.
    //이 때, returns_asref()가 &str를 반환했기 때문에, owned를 borrowed 변수에 다시 빌려줌(reborrowing)
    let borrowed = returns_asref(&owned);
    //borrowed 출력
    println!("{borrowed}");
    //여기 이후로 borrowed가 사용되지 않는다. 따라서, reborrow된 값 owned는 여기 이후로 다시 수정가능 상태가 된다.
}
```

먼저, `main`함수를 분석하여 해당 내용을 주석으로 표시하였다. 여기서 2번째 줄은 중요하므로 자세히 분석해 보자. 이 줄에서는 main함수가 소유하고 있는 `String` 값인 `owned` 변수를 `returns_asref()` 함수에 immutable borrow하여 전달한다. 여기서, 컴파일러는 `returns_asref()`함수의 선언부를 참고하여 여러 규칙을 적용하게 된다. 아래는 `returns_asref()` 함수의 선언부이다.

```rust
fn returns_asref<'a>(t: impl AsRef<str> + 'a) -> &'a str
```

이 함수는 lifetime generic parameter `'a`와 `t`를 인자로 받고, `&'a str`을 반환한다. 여기서 매개변수 `t`와 `&'a str`은 같은 lifetime인 `'a`로 묶여있다. 따라서, 컴파일러는 매개변수 `t`를 reborrow하여 `&'a str`을 반환한다. 컴파일러는 반환되는 값의 출처가 매개변수 `t`라 분석한다. 따라서, 반환되는 레퍼런스가 사용되는 동안에는 매개변수로 넘어온 `t`는 수정 불가 상태가 되고, `t`가 drop되면 반환된 `&'a str`도 사용불가 상태가 된다(invalid reference).

`t: impl AsRef<str> + 'a`의 의미에 대해 자세히 알아보자. 매개변수 `t`는 `AsRef<str>`타입을 구현해야 한다. 이러한 타입으로는 `&str`, `String`등이 있다. `+ 'a`는 해당 타입의 모든 reference field의 lifetime이 `'a`와 같거나 길어야 한다는 의미이다.


## returns_asref 함수 분석

```rust
fn returns_asref<'a>(t: impl AsRef<str> + 'a) -> &'a str {
    //t를 빌려서 반환.
    t.as_ref()
}
```

`returns_asref` 함수의 선언부는 위에서 분석하였기에 생략한다. `t.as_ref()`는 매개변수 `t`를 빌려서 반환한다. `as_ref()` 함수는 `AsRef` trait에 정의되어 있는 메서드로, cheap reference-to-reference conversion을 하는 함수이다. 주로, 구조체나 enum의 field의 reference를 제공하기 위해 사용된다. 아래는 `AsRef` trait의 정의이다.

**std::convert::AsRef**

```rust
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}
```

`AsRef<T>`의 선언부를 보자. `<T: ?Sized>`에서 `Sized`는 대부분의 타입들이 구현하는 marker trait이다. 이 trait은 크기가 정해진 타입에 구현된다. `?Sized`는 해당 marker trait이 구현되어 있거나, 없거나 상관하지 않겠다는 의미이다. 즉, 크기가 정해지거나 정해지지 않은 타입 모두를 수용하겠다는 의미이다. `str`같은 경우 Dynamic sized type(DST)이다. 크기가 정해지지 않은 타입이고, `Sized` trait이 구현되어 있지 않다. 정리하자면, `<T: ?Sized>`는 `str`과 같은 크기가 정해지지 않은 타입도 `T`에 올수 있게 하기 위한 trait bound이다.

`as_ref()` 함수는 `&self`를 매개변수로 받고, `&T`를 반환한다. 이는, `self`를 `&T`로 reborrow 하는 것이다.

```rust
fn returns_asref<'a>(t: impl AsRef<str> + 'a) -> &'a str {
    //t를 빌려서 반환.
    t.as_ref()
}
```

`returns_asref` 함수로 돌아오자. 이 함수의 선언부는 lifetime parameter `'a`, `t: impl AsRef<str> + 'a`를 인자로 받고, `&'a str`을 반환한다. 이 함수를 컴파일하면 owned type `t`의 레퍼런스를 반환할 수 없다고 한다. 즉, 컴파일러는 `t`를 owned type으로 여기는 것이다. 만약 `String`과 같은 owned type이 매개변수 `t`로 넘어오게 되면, 소유권도 같이 넘어오고, 이 함수가 반환될 때 drop되기 때문이다. 컴파일러는 매우 보수적으로 우리가 작성한 코드를 분석하여 잠재적으로 문제가 될 수 있는 코드들을 거부한다.

위 문제를 lifetime 관점으로 바라보자.

## lifetime의 관점으로 분석

위 함수의 매개변수인 `t`의 lifetime을 `'input`이라고 하자. 

`t: impl AsRef<str> + 'a`의 의미는 다음과 같다. 매개변수 `t`는 `AsRef<str>`타입을 구현해야 한다. 이러한 타입으로는 `&str`, `String`등이 있다. 실제 어떤 타입인지는 컴파일 시점에서 알 수 있다. `+ 'a`는 해당 타입의 모든 reference field의 lifetime이 `'a`와 같거나 길어야 한다는 의미이다.

**반례: `'input`이 함수의 생명주기와 같은 경우(Owned type)**

`t`가 owned type이라고 가정하다. 아래는 `t`의 예시 중 하나이다.
```rust
# fn main() {}
struct ExampleType<'b> {
    owned: String,
    borrowed: &'b str,
}

impl<'b> AsRef<str> for ExampleType<'b> {
    fn as_ref(&self) -> &str {
        &self.owned
    }
}
```
`ExampleType`은 하나의 owned field와, 하나의 reference field로 이뤄진다. 
만약 `ExampleType`이 `return_asref()`의 인자로 넘어온다면 이 함수로 이동된다. `t: impl AsRef<str> + 'a` 에서 `'a`는 `ExampleType`의 field 중 reference field인 `borrowed`의 lifetime이다. 즉, `'b`와 같다. 이 때, `as_ref`를 호출하면, `&self`를 매개변수로 넘겨주는데, 위의 경우에서는 `owned`가 반환된다. `owned`는 `return_asref()` 함수가 반환될 때, `ExampleType`과 같이 drop된다. 따라서, dangling reference, use after free와 같은 메모리 위반이 발생한다. 컴파일러는 `as_ref()`가 반환하는 값의 lifetime이 함수의 scope보다 긴지 짧은지 알 수 없다. 따라서, 짧다고 가정(owned type)한다.

컴파일러는 위와 같은 상황을 막기 위해서 매개변수로 전달되는 generic type을 owned type으로 간주한다. 물론, reference type가 넘어오면 drop이 실행되지는 않는다. 그러나 generic type이 매개변수에 있다면, 해당 타입을 보수적으로 해석하여 lifetime을 분석한다. `t`의 lifetime이 가장 짧은 경우가 owned type인 경우이기 때문에 이 경우를 기준으로 분석하게 된다.

# 위 오류를 고치려면?

우리가 표현하고 싶은 것은 아래와 같다
1. lifetime `'a` 정의
1. `AsRef::as_ref(&self)` 가 반환하는 `&str`의 lifetime이 `'a`이다.
1. `return_asref()`가 반환하는 `&str`의 lifetime이 `'a`이다.

안타깝게도 위의 조건을 Rust 문법으로 표현하는 것은 불가능하다. 따라서 명시적으로 `&str`을 매개변수로 받게 작성해야 한다. 이를 적용하면 아래와 같다.

```rust
fn returns_reference(t: &str) -> &str {
    //t를 빌려서 반환.
    t
}
```

## 결론
1. 함수를 호출할 때 컴파일러는 함수의 선언부(Signature)를 기준으로 lifetime과 borrowing rule을 적용한다. 호출되는 함수의 내부를 분석하여 규칙을 적용하지 않는다.
2. Rust 컴파일러는 최대한 보수적으로 우리가 작성한 코드를 분석한다. 예를 들어, `t: impl AsRef<str>`에서 `t`의 lifetime은 함수의 scope보다 길 수도(reference), 짧을 수도(owned) 있다. 컴파일러는 가능한 모든 lifetime 중 가장 짧은 lifetime을 기준으로 규칙을 적용한다. 이는 메모리 안전성을 지키기 위함이다.
3. Rust 컴파일러는 generic 타입의 매개변수의 lifetime을 분석할 때 가능한 모든 경우의 수를 고려한다. generic 타입은 해당 조건만 만족하면 그 어떤 타입도 올 수 있기 때문이다.

이 문제에 대한 결론을 내리기까지 상당한 시간이 소요되었다. 컴파일러처럼 생각하는 연습을 더 해야겠고, 생각보다 Rust 컴파일러가 엄격하다는 것을 느끼게 되었다. 사실 이렇게 엄격하기 때문에 `unsafe`를 쓰지 않는 이상 모든 메모리 위반을 컴파일할때 걸러낼 수 있다는 생각이 들었다. C와 같은 언어로 메모리를 잘 다루는 것이 왜 어려운지 간접적으로 느끼게 되었다.