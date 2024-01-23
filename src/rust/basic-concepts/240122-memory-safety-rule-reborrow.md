# 메모리 안전성을 위한 심화 규칙 - Reborrowing의 이해

2024.01.22

---

Rust에서는 Reference를 다시 Borrow할 수 있다. 이걸 Reborrowing 이라고 한다. Reborrowing은 크게 두 가지의 상황에서 일어난다. 첫째로는, mutable reference를 reborrowing 하는 것이고, 둘째로는 함수가 Reference를 반환할 때 일어날 수 있다. 이 글에서는 Reborrowing이 정확이 무엇을 의미하는지와 언제, 어떻게 발생하는지 알아볼 것이다.

# Reborrowing 이란?

이미 존재하는 Reference을 Borrow하여 또 다른 Reference를 생성하는 것을 Reborrowing이라고 한다. 이 글에서 '부모'는 Reborrow된 원본의 Reference를, 'Reborrow'는 Reborrow로 생성된 Reference를 의미한다.
어떤 Reference `x`를 reborrow 하여 `y`라고 하는 reborrow를 생성하면, `y`의 부모는 `x`가 된다. 이 때, lifetime, ailiasing rule은 일반적인 값을 borrow했을 때와 비슷하게 적용된다.
`y`는 `x`의 lifetime이 끝나기 전까지만 사용할 수 있고, `y`의 lifetime이 끝나거나 마지막으로 사용되기 전까지(Non Lexical Lifetime) `x`는 수정 불가능 상태가 되고, mutable borrow할 수 없게 된다. 이는 일반적인 값을 borrow 했을때와 비슷하다.
일반적으로 immutable reference에서는 reborrowing이 발생하지 않는다. 이는 여러 개의 immutable reference가 동시에 존재할 수 있기 때문이다. mutable reference를 immutable borrow하거나, mutable borrow하면 reborrow가 발생한다. 이 때, 원래의 mutable reference는 일시적으로 사용 불가능 상태가 된다.

# mutable reference의 Reborrowing

```rust
fn main() {
    let mut s = String::from("asdf");
    //s -> &mut x
    let x: &mut String = &mut s;
    //s -> &mut x -> &mut String
    //x를 reborrow하여 함수로 넘김
    clear_string(x);
    //reborrow 사용 종료
    //x 다시 사용 가능.
    x.push_str("a");
    println!("{x}");
}

fn clear_string(s: &mut String) {
    s.clear();
}
```

위의 코드에서 `clear_string()` 함수를 호출할 때 `x`를 reborrow하고 있다. 함수가 반환하면 reborrow는 종료되고, `x`는 다시 사용할 수 있게 된다.
만약 reborrowing이 발생하지 않고 move가 발생했다면, `x`는 `clear_string()`함수의 실행 종료와 동시에 사용 불가능하게 됬을 것이다. 이처럼, reborrow는 rust의 엄격한 메모리 제한을 보다 유연하게 해 준다. 우리가 프로그래밍을 하면서 인지하지 못하더라도, rust를 보다 쉽게 다룰 수 있도록 도와주고 있다.

위의 경우 뿐만 아니라, mutable reference를 immutable borrow 한 경우도 비슷하게 reborrowing이 발생한다.

# 함수 호출시 발생하는 Reborrowing

아래와 코드를 보자

```rust
fn main() {
    let mut a = String::from("a");
    let mut b = String::from("bb");
    let mut c = String::from("ccc");

    //test 함수의 선언부에 따라서, 반환되는 &str의 lifetime 'a를 공유하는
    //매개변수 a, b를 reborrow함. c의 borrow는 이 함수가 반환될 때 끝남.
    //&a, &b -> &str
    let t: &str = test(&a, &b, &c);

    //c는 수정 가능함. c의 borrow는 위 함수가 반환될 때 끝남.
    c = String::from("ccccc");

    //a, b는 수정 불가능함. reborrow인 t가 살아있는 동안에는 수정 불가능함.
    //rust에서는 immutable reference와 mutable reference가 동시에 존재할 수 없고,
    //a, b를 수정하기 위해서는 mutable borrow가 필요함.
    a = String::from("ccccc");
    b = String::from("ccccc");

    //reborrow가 마지막으로 사용됨.
    println!("{t}");
    //이 이후로는 reborrow가 사용되지 않기 때문에 a, b는 다시 수정가능 상태가 됨.
}

//더 긴 문자열의 reference를 반환
fn test<'a, 'b>(a: &'a str, b: &'a str, c: &'b str) -> &'a str {
    if a.len() < b.len() {
        b
    } else {
        a
    }
}
```

**출력**
```
error[E0506]: cannot assign to `a` because it is borrowed
  --> src/main.rs:17:5
   |
9  |     let t: &str = test(&a, &b, &c);
   |                        -- `a` is borrowed here
...
17 |     a = String::from("ccccc");
   |     ^ `a` is assigned to here but it was already borrowed
...
21 |     println!("{t}");
   |               --- borrow later used here

error[E0506]: cannot assign to `b` because it is borrowed
  --> src/main.rs:18:5
   |
9  |     let t: &str = test(&a, &b, &c);
   |                            -- `b` is borrowed here
...
18 |     b = String::from("ccccc");
   |     ^ `b` is assigned to here but it was already borrowed
...
21 |     println!("{t}");
   |               --- borrow later used here
```

위의 코드에서 `test()`를 호출할 때 컴파일러는 함수의 선언부를 중심으로 함수의 몸체 부분과 사용되는 부분의 메모리 안전성을 보장한다.

### 1. `test()`함수의 선언부 분석

먼저 컴파일러는 `test()`함수의 선언부를 분석하여 아래와 같은 정보를 얻는다.

```rust
fn test<'a, 'b>(a: &'a str, b: &'a str, c: &'b str) -> &'a str
```

#### 1. 이 함수는 두 개의 lifetime annotation을 사용한다(`'a`, `'b`).
#### 2. 이 함수는 `&'a str`를 반환한다. 따라서, 반환되는 reference는 매개변수로 들어온 reference 중 `'a` lifetime을 공유하는 `a`와 `b`를 reborrow 한다.

### 2. `test()`함수의 몸체 부분 분석

```rust
fn test<'a, 'b>(a: &'a str, b: &'a str, c: &'b str) -> &'a str {
    if a.len() < b.len() {
        b
    } else {
        a
    }
}
```

컴파일러는 이 함수의 몸체 부분의 반환값을 검사한다. `'a` lifetime에 속한 reference를 반환하고 있는지 검사하여, 아니다면 오류를 발생한다. 또한, dangling reference를 반환하는지도 검사한다.
위 코드에서는 `'a` lifetime에 속한 reference인 a 또는 b를 반환하기에 정상적으로 컴파일 된다.

### 3. 함수의 사용 위치 분석

컴파일러는 함수를 호출한 위치에서 반환된 reference가 잘 사용되고 있는지 검사한다. 이 때, 함수의 선언부를 분석하여 얻은 정보를 사용하고, 몸체를 분석하면서 얻은 정보는 고려되지 않는다. 두 부분은 완전히 별개로 컴파일러에 의해 검사된다.

```rust
fn main() {
    let mut a = String::from("a");
    let mut b = String::from("bb");
    let mut c = String::from("ccc");

    //test 함수의 선언부에 따라서, 반환되는 &str의 lifetime 'a를 공유하는
    //매개변수 a, b를 reborrow함. c의 borrow는 이 함수가 반환될 때 끝남.
    //&a, &b -> &str
    let t: &str = test(&a, &b, &c);

    //c는 수정 가능함. c의 borrow는 위 함수가 반환될 때 끝남.
    c = String::from("ccccc");

    //a, b는 수정 불가능함. reborrow인 t가 살아있는 동안에는 수정 불가능함.
    //rust에서는 immutable reference와 mutable reference가 동시에 존재할 수 없고,
    //a, b를 수정하기 위해서는 mutable borrow가 필요함.
    a = String::from("ccccc");
    b = String::from("ccccc");

    //reborrow가 마지막으로 사용됨.
    println!("{t}");
    //이 이후로는 reborrow가 사용되지 않기 때문에 a, b는 다시 수정가능 상태가 됨.
}
```

컴파일러는 함수의 선언부 분석 정보를 바탕으로 사용된 위치에서 올바르게 사용되었는지 검사한다. 위에서 `test()`함수가 호출될 때, 컴파일러는 반환되는 reference의 lifetime `'a`를 사용하는 매개변수로 들어온 reference `a`, `b`를 `t`와 연결시킨다. 이 과정을 reborrow라고 한다. 이 때, lifetime 분석 뿐만 아니라, ailiasing rule도 분석한다.
따라서, `t`의 lifetime이 유효하려면, `a`, `b`가 살아 있어야 하고, `t`가 사용되고 있는 동안에는 `a`, `b`를 수정할 수 없다.
이 때, `c`는 `'b` lifetime을 사용하였고, 반환되는 reference에 `'b` lifetime은 사용되지 않았기 때문에 `c`의 borrow는 `test()` 함수의 반환과 동시에 종료된다. 따라서 함수 호출 이후 즉시 수정 가능하다. 그러나, `a`, `b`는 `t`가 유효할 동안, 즉, `t`가 마지막으로 사용된 이후까지 수정할 수 없다.


## Reborrowing의 또 다른 예시: `&*`의 사용

deref coercion이 발생하지 않을 경우에는 수동으로 reborrowing할 수 있다. deref coercion에 대해서는 다른 글에서 자세히 알아볼 예정이다.

아래의 코드를 보자.

```rust
fn main() {
    let a = String::new();
    //Deref coercion이 발생함. 컴파일러는 아래와 같은 코드를 생성함.
    //let s: &str = core::ops::Deref::deref(&a);
    let s: &str = &a;
}
```

위의 예시에서는 deref coercion이 발생하여 컴파일러에 의해 `deref`가 호출된다. 그러나, 모든 경우에서 deref coercion을 컴파일러가 적용해줄 수 있는건 아니다.
이런 경우에는 `core::ops::Deref::deref()` trait을 직접 호출해줄수 있으나, 코드가 장황해진다. 이 때, `&*`를 사용하여 해결할 수 있다.
아래는 적용 예시이다.

```rust
fn main() {
    let a = String::new();
    //만약, 컴파일러가 deref coercion을 추론할 수 없을 경우에는 아래와 같이 수동으로 작성할 수 있다.
    let s: &str = &*a;
}
```

위 코드를 분해해 보면 다음과 같다.

1. `*a`: dereference operator이다. `deref()`가 호출되서 `str`이 된다. `str`은 크기가 유동적인 타입이라서 reference 형태로만 사용될 수 있다.
2. `&(*deref())`: reference operator이다. `&str`이 된다.
2. `deref()`: 결과적으로 `deref()`함수를 호출한 것과 같다.

대부분의 경우에는 `&*`을 사용할 일이 없을 것이다. 컴파일러는 똑똑하다. 그러나, 컴파일러가 deref coercion을 추론하지 못하는 특정 상황에서는 위의 방법을 사용해야 한다.
`DerefMut`에 대해서 사용하려면 `&mut *a` 형태로 쓰면 된다. 이 방법은 `deref()` 또는 `deref_mut()`를 직접 호출하는것 보다 간결하게 사용할 수 있다.

## 결론

오늘은 rust의 심화 개념인 reborrowing에 대해 알아보았고, 언제 일어나는지 알아보았다.

1. reborrowing 덕분에 우리는 mutable reference를 좀 더 쉽고 유연하게 사용할 수 있다.
2. 함수가 reference를 반환할 때 발생하는 reborrowing에 대해 알아보았고, 이 과정이 함수의 선언부를 중심으로 일어난다는 것을 보았다. 따라서, 함수의 선언부를 작성할 때는 lifetime에 대한 깊은 이해를 바탕으로 작성해야, 컴파일 오류를 방지할 수 있고, 지나치게 제약된 규칙 적용을 방지할 수 있다.
3. deref coercion을 수동으로 처리해야 하는 경우 `&*s` 또는 `&mut *s`를 사용하여 처리할 수 있다.

따라서, reborrowing에 대한 내용은 rust의 매모리 안전성 시스템을 이해하는데 매우 중요하다.

## 참고하면 좋을 학습자료
* [함수의 선언부가 가지는 의미](./rust/tips/231220-meaning-of-function-signature.md)
