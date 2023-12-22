# Shadowing의 이해와 활용

2023.12.22

---

오늘은 Shadowing에 대해 알아보려고 한다. 사실, Shadowing은 Rust 외 다른 언어에도 있는 개념이다. Shadowing은 두 변수의 이름이 동일할 때 어떤 변수를 이용할지 결정하는 규칙이다. 
이 글에서는 Shadowing을 크게 두 가지 경우로 나눠 설명하려 한다.

## 서로 다른 scope에서 일어나는 Shadowing

```rust
fn main() {
    let var1 = 123;
    {
        let var1 = 999;
        println!("{var1}");
    }
    println!("{var1}");
}
```

**출력**
```
999
123
```

위의 코드에는 scope(중괄호) 밖의 `var1`과 scope 안의 `var1`이 있다. scope 안에서 정의된 `var1`과 밖에서 정의된 `var1`의 이름이 중복된다.
이런 경우에 shadowing이 발생한다. shadow는 '그림자'라는 의미를 갖는다. scope 안에서 `var1`을 정의할 때, scope 밖의 같은 이름의 변수와 중복되기 때문에 shadow의 의미인 그림자처럼 scope 밖의 `var1`을 가리게 된다. 이 현상을 shadowing 이라 부른다. '그림자'의 의미와 같이, shadowing이 되더라도, 변수의 이름만 잠깐 가릴 뿐, scope 밖에서 정의된 `var1`에는 아무런 영향을 주지 않는다. scope가 끝나면 scope 밖의 `var1`은 다시 접근 가능하게 된다.

Shadowing은 같은 scope 내에서도 일어날 수 있다. 이러한 현상은 다른 언어에서는 거의 찾아볼 수 없다. Rust만의 특별한 기능이고, 언어의 특징에 의한 특별한 쓰임세가 있다.

## 같은 scope 내에서 일어나는 Shadowing

```rust
fn main() {
    //타입 힌트는 가독성을 위해 주석으로 나타냄. 실제로는 없어도 자동으로 추론됨.
    let var1/*: usize*/ = 1234;
    println!("{var1}");
    let var1/*: String*/ = String::from("string");
    println!("{var1}");
}
```

**출력**
```
1234
string
```

위의 코드에서는 같은 scope(`main()`함수) 내에 동일한 이름인 `var1`과 서로 다른 타입 `usize`, `String`을 가진 변수를 선언하였다. 가독성을 위해 타입을 주석으로 표시하였다. 대부분의 언어에서는 위와 같은 shadowing을 허용하지 않는다. Rust는 언어적 특징에 의해서 있는 기능이다. 이것이 무슨 의미인지는 뒤에서 예제와 함께 자세히 알아볼 것이다. 위 코드에서 일어나는 shadowing을 이해하려면 컴파일러가 코드를 어떻게 해석하는지 알아봐야 한다.

## binding이란?

먼저, Binding이 뭔지 알아야 한다. Rust에서는 모든 선언을 binding으로 취급한다. binding은 변수의 이름을 데이터와 연결(bind)하는것을 의미한다. 아래 코드를 보자.

```rust
fn main() {
    let var1/*: usize*/ = 1234;
    //   |        |         |
    // 변수명      |         |
    //         타입(추론됨)  |
    //                    데이터(지역변수, 스택에 할당됨)
}
```

Binding은 위의 코드와 같이, 데이터(메모리)와 변수명을 연결해 준다. 여기서 중요한 점은, 변수명과 데이터(메모리이 할당된 값)은 별개로 취급된다. 이 둘은 `let` 키워드를 통해서 연결되게 된다.
이 개념은 Rust 내에서 엄청 중요하다. 데이터가 drop 되는 시점은 변수명의 의해 결정되는 것이 아니다. 바로, 데이터(메모리)가 위치한 scope에 따라 결정된다. 변수명이 아닌 데이터(메모리)가 위치한 scope를 벗어나기 직전 해당 데이터(메모리)에 대한 drop이 실행되어 메모리가 해제된다. 아래의 예시를 보면 이해하는데 도움이 될 것이다.

```rust
fn main() {
    //var1: 변수명, Test {}: 데이터
    let var1: Test = Test {};
    //var1: 변수명, String: 데이터
    //여기서, var1은 새 데이터(메모리) String으로 다시 binding됨.
    let var1: String = String::from("asdf");
    //var1은 다시 binding된 String의 데이터를 나타냄
    println!("{var1}");
    //위에서 할당된 데이터 Test {}, String이 scope를 벗어남
    //drop(메모리 해제)
}

struct Test {}

impl Drop for Test {
    fn drop(&mut self) {
        println!("dropped test!!!");
    }
}
```

**출력**
```
asdf
dropped test!!!
```

출력을 보면 알수 있는것처럼 `println!` 실행 후 `dropped test!`가 출력된다. `var1`이 `String`으로 다시 binding되더라도, 연결된 메모리는 해당 메모리가 할당된 scope를 벗어나야 해제됨을 알 수 있다.

이제 Binding 개념을 같은 scope 내에서 일어나는 shadowing에 적용해 보자.

## 같은 scope 내에서 일어나는 Shadowing - binding의 관점에서 이해하기

```rust
fn main() {
    //타입 힌트는 가독성을 위해 주석으로 나타냄. 실제로는 없어도 자동으로 추론됨.
    let var1/*: usize*/ = 1234; //A
    println!("{var1}");
    let var1/*: String*/ = String::from("string"); //B
    println!("{var1}");
}
```

**출력**
```
1234
string
```

위의 코드에서 지역변수 `1234`를 `A`, 지역변수 `String::from("string")`을 `B`라 하자.
첫 번째 `let var1/*: usize*/ = 1234;` 코드가 실행된 이후의 binding은 아래와 같다.

| binding 명 | 연결된 데이터 |
| ---------- | ----------- |
| var1       | A           |

이 때 `var1`을 출력하면 연결된 데이터 `A`의 값인 `1234`가 출력된다.

첫 번째 `let var1/*: usize*/ = 1234;` 코드가 실행된 이후의 binding은 아래와 같다.

| binding 명 | 연결된 데이터 |
| ---------- | ----------- |
| var1       | B           |

데이터 `A`가 binding 해제되었다. 그렇다고 해서 데이터 `A`의 lifetime이나 drop(해제)되는 시점은 변하지 않고 데이터가 속해있는 scope(`main()`)를 기준으로 정해진다.
이 때 `var1`을 출력하면 연결된 데이터 `B`의 값인 `string`이 출력된다.

`main()`함수 실행이 종료되었다. `main()`함수 scope에 포함된 데이터 `A`, `B`는 이 시점에서 drop(해제)된다. binding `var1`는 데이터를 연결하는 역활을 할 뿐이지, 해당 데이터의 lifetime과 drop(해제)시점을 결정하지 않는다.

## 예시 - immutable rebinding

아래와 같은 코드가 있다.

```rust
fn main() {
    //숫자 vector
    let mut numbers = vec![1, 4, 2, 5, 3];
    //정렬
    std::slice::sort(&mut numbers);
    //사용자가 실수로 변경함.
    numbers[0] = 333;
}
```

위의 코드는 `numbers` 슬라이스를 정렬하는 예시이다. 슬라이스를 정렬하기 위해서는 `std::slice::sort()`함수애 slice의 mutable reference를 넘겨줘야 한다.
Rust에서는 최대한 `mut`키워드를 자제하는 것이 좋다. 사용자의 의도와는 다르게 실수로 변경할 수 있기 때문이다. 이와 같은 문제를 해결하기 위해 shadowing을 사용할 수 있다.
물론, shadowing 이외 scope를 이용하여 해결하는 방법도 있지만 여기서는 다루지 않겠다.

**std::slice::sort**
```rust
impl<T> [T] {
    pub fn sort(&mut self) where
        T: Ord,
    {
        //...
    }
}
```

```rust
fn main() {
    //숫자 vector
    let mut numbers = vec![1, 4, 2, 5, 3];
    //정렬
    numbers.sort();
    //사용자가 실수로 변경함.
    numbers[0] = 333;
}
```

위의 코드에서는 정렬된 `numbers` vector를 사용자의 의도와 잘못되게 사용된 예이다. 사용자의 의도는 변경 불가능해야 하나, 정렬을 위해서 불가피하기 mutable한 binding으로 선언하였다.

아래는 shadowing을 이용해 mutable한 vector `numbers`의 데이터를 immutable하게 rebinding한 코드이다.
`numbers`가 immutable하게 rebinding 되었기 때문에 해당 코드는 정상적으로 오류를 발생한다.

```rust
fn main() {
    //숫자 vector
    let mut numbers = vec![1, 4, 2, 5, 3];
    //정렬
    numbers.sort();
    //immutable rebinding(shadowing)
    //mut numbers를 numbers(immutable)로 rebinding
    let numbers = numbers;
    //사용자가 실수로 변경함.
    //컴파일 오류!! immutable binding을 변경하려고 시도!
    numbers[0] = 333;
}
```

**출력**

```
error[E0596]: cannot borrow `numbers` as mutable, as it is not declared as mutable
  --> src/main.rs:11:5
   |
11 |     numbers[0] = 333;
   |     ^^^^^^^ cannot borrow as mutable
   |
help: consider changing this to be mutable
   |
8  |     let mut numbers = numbers;
   |         +++
```

위와 같이 컴파일에러가 발생한다. Rust 코드를 작성할 때에는 모든 변수가 immutable하다는 가정 하에 작성해야 한다. 만약, 위와 같이 `&mut`참조가 불가피하기 필요할 경우, `&mut`참조의 사용이 완료된 후(위 코드에서는 `sort()`), 같은 변수명으로 rebinding 함으로써 사용자의 실수를 방지할 수 있다. `Vec<T>`와 같은 Owned type 뿐만 아니라, `&mut T`와 같은 mutable reference를 reborrow하는 경우에도 위의 방법을 적용할 수 있다.

반대로, 아래와 같이 immutable한 데이터를 mutable한 데이터로 rebinding할 수도 있다. 이 때는 move가 발생한다. reference인 경우에는 이러한 rebinding이 불가능하다.

```rust
fn main() {
    let s = String::from("asdf");
    //가능함. 같은 이름의 새로운 binding으로 move됨.
    let mut s = s;
}
```


## 예시 - 여러 단계의 연산

하나의 변수를 여러 단계를 거쳐서 변환해야 하는 경우가 있다. 아래의 예시를 보자.

```rust
fn main() {
    //문자열 input 선언
    let input: &str = "1234";
    //input 문자열을 u32로 변환후 5를 더함
    let input_number = input.parse::<u32>().unwrap() + 5;
    //변환되기 전의 문자열을 잘못 사용!! 컴파일러가 잡을 수 없음.
    println!("{input}");
}
```

**출력**
```
1234
```

변환되기 전의 값 `1234`가 잘못 출력되었다.

위의 코드는 `input`문자열을 `u32`형으로 변환하여 5를 더한 다음, `input_number`에 저장하는 예시이다. Rust를 제외한 대부분의 정적 타입 언어에서는 shadowing이 지원되지 않기 때문에 다른 타입의 데이터를 저장하려면 위와 같이 다른 변수명을 지정해야 한다.
이 방식에는 여러가지 문제점이 있는데, 프로그래머가 변환되기 전의 값을 실수로 사용할 수도 있고, 변환 단계가 많아지면 변수 이름을 결정하기가 어려워진다. 같은 scope 내의 variable shadowing 은 정적 타입 언어가 가지는 이런 문제점들을 해결해준다.

위의 코드를 shadowing을 활용하여 수정하면 아래와 같다.

```rust
fn main() {
    //문자열 input 선언
    let input: &str = "1234";
    //input 문자열을 u32로 변환
    //input 변수명을 변환된 값으로 rebinding함.
    let input = input.parse::<u32>().unwrap() + 5;
    //변환된 후의 변수를 정상적으로 사용.
    println!("{input}");
}
```

**출력**
```
1239
```

변환된 후의 값 `1239`가 정상적으로 출력되었다.
Shadowing은 iterator의 중간값 저장(`Iterator::collect()`), 길고 복잡한 연산을 여러 단계로 나눠서 표현할때도 사용할 수 있다.

## 예시 - temporary value dropped while borrowed 해결

아래의 코드를 보자. 파일의 이름을 확장자와 분리하는 예시이다.

```rust
fn main() {
    //파일명을 . 을 기준으로 분리하여 앞의 이름만 추출
    let filename: &str = String::from("123.asdf")
        .split(".")
        .next()
        .unwrap();
    println!("{filename}");
}
```

**출력**

```
error[E0716]: temporary value dropped while borrowed
 --> src/main.rs:2:20
  |
2 |     let filename: &str = String::from("123.asdf").split(".").next().unwrap();
  |                    ^^^^^^^^^^^^^^^^^^^^^^^^                           - temporary value is freed at the end of this statement
  |                    |
  |                    creates a temporary value which is freed while still in use
3 |     println!("{filename}");
  |               ---------- borrow later used here
  |
help: consider using a `let` binding to create a longer lived value
  |
2 ~     let binding = String::from("123.asdf");
3 ~     let filename = binding.split(".").next().unwrap();
```

위의 코드는 컴파일이 되지 않는다. 표면적으로는 아무 문제가 없어 보인다. 컴파일 오류 메세지를 보자. `String::from("123.asdf")`가 임시의 값을 만들고 사용되는 동안 해제되었다고 한다. 그리고, 위 문장이 끝날 때 임시의 값이 해제되었다고 한다.
Rust에서 문장의 중간에서 값을 생성하는 경우 임시의 값으로 취급되어 해당 문장이 끝날 때 해제된다.

`split()` 함수의 선언부를 보자.

**std::str::split()**
```rust
pub fn split<'a, P>(&'a self, pat: P) -> Split<'a, P> where
    P: Pattern<'a>,
```

**std::str::Split 구조체의 Iterator 구현부**
```rust
impl<'a, P> Iterator for Split<'a, P>
where P: Pattern<'a>
{
   //...생략 
   fn next(&mut self) -> Option<&'a str> {
        //...
    }
   //...생략 
}
```

위의 선언부에서 borrowing 관계를 중점으로 확인해보자. `std::str::split()`함수는 매개변수로 `&'a self`를 받는다. 여기서 `self`는 `str`이다.
위 함수는 `&'a str`을 borrow해서 `Split<'a, P>` 구조체를 반환한다. 구조체에도 generic lifetime parameter가 포함되어 있기 때문에 문자열 슬라이스(`&'a str`)를 reborrow하게 된다.
`Iterator` 구현부의 `next()`함수를 살펴보자. `next()` 함수는 `'a` lifetime인 문자열 슬라이스를 반환한다. 즉, 원본 문자열의 reborrow를 반환한다, `split()`의 경우에는, 원본 문자열에서 분리된 부분을 참조하게 된다. 문자열을 복사하지 않고 원본 문자열에 대한 참조를 반환하기 때문에, 메모리를 효율적으로 사용한다.

위의 문제를 해결하기 위해서는 문장이 끝난 뒤에도 `String::from("123.asdf")`에서 생성한 값이 해제되지 않아야 한다. 왜냐하면, 반환되는 `filename` 가 `&str`이고, 생성된 문자열에서 reborrow 하기 때문에다. 따라서, 문장이 끝난 후에 해제되지 않아야 한다.

문자열을 할당한 다음, 분리된 문자열 슬라이드를 rebinding하여서 문제를 해결하였다.

```rust
fn main() {
    //문자열 할당
    let filename = String::from("123.asdf"); //'a lifetime이라 가정
    //파일명을 . 을 기준으로 분리하여 앞의 이름만 추출
    //추출된 문자열 슬라이스(위에서 할당된 문자열로부터 reborrow됨)를 같은 이름으로 rebinding함
    //반환된 filename의 lifetime은 'a
    let filename: &str = filename
        .split(".")
        .next()
        .unwrap();
    println!("{filename}");
    //여기서 할당된 문자열(첫번째 filename 할당)이 drop(해제)됨. 해당 데이터가 생성된 곳을 기준으로 해제됨. 'a lifetime 종료
}
```

**첫번째 `let filename = ...` 직후 binding 상태**

| binding 명 | 연결된 데이터 |
| ---------- | ----------- |
| filename   | String 객체('a lifetime) |

**두번째 `let filename = ...` 직후 binding 상태**

| binding 명 | 연결된 데이터 |
| ---------- | ----------- |
| filename   | &'a str (이전 filename binding에 연결된 String 객체의 reference) |

위 코드는 정상적으로 컴파일된다. 위 예제에서는 filename의 부분 문자열의 slice를 원본 문자열로부터 borrow하여 rebinding 하였다. 위의 경우에 shadowing를 활용하여 중간 단계를 저장하는 변수명을 짓지 않아도 되고, 파일명을 추출하기 전의 원본 filename을 잘못 접근해 사용하는 상황을 막을 수 있다. 여기서, `String::from("123.asdf")`에서 할당된 문자열은 해당 데이터가 속한 scope(`main()`)이 종료될 때 drop(해제)된다. binding인 `filename`이 변경되더라도 해제되지 않기 때문에 두 번째 `let` 문장에서 rebinding된 원본 문자열을 가리키는 borrow는 유효한 lifetime을 갖게 된다.

위에서 본 것과 같이 shadowing은 lifetime문제를 해결함과 동시에, 프로그래머가 잘못된 값(변환되기 전의 값)에 접근할 수 없도록 한다.

## 예시 - stack pinning

pinning은 복잡하고 어려운 주제이기 때문에 별도의 글에서 자세히 다루도록 하겠다. 이해되지 않으면 이 챕터는 그냥 넘기고 나중에 이해해도 좋다.

shadowing은 stack pinning에서도 사용할 수 있다. pinning은 값을 움직일(move) 수 없도록 메모리의 특정한 위치에 고정하는 것이다. 한번 pin된 값은 다른 위치로 움직이면 안 되고, `Pin<T>` 포인터 타입을 반환하게 된다. 이 포인터는 메모리에 고정된 값을 가리킨다. pinning은 self-referential data structure(자기 참조 자료구조)를 가지는 곳에서 사용되며, Rust에서는 비동기 프로그래밍에서 쓰이는 `Future`에서 주로 사용된다.
heap에서의 pinning은 간단하다. `Box<T>`를 활용하면 고정된 메모리 위치에 값이 저장되기 때문이다. 그러나, 비동기 프로그램에서 항상 `Box<T>`를 활용하는 것은 오버헤드가 크다. heap 메모리 할당에는 CPU자원이 많이 소모된다. 따라서, stack pinning을 주로 활용하여 함수 내에서 `Future`를 사용(`poll()` 호출)하기 전에 pinning을 한다.
이를 가능하게 해주는 것이 scope내 shadowing이다. Rust에는 stack pinning을 위한 `pin!()` 매크로가 있는데, 이는 pinning할 값을 `Pin<T>` 포인터로 shadowing하여 원래 값에 직접적으로 접근하여 움직이지 못하게 막는다. 이를 위해서는 `unsafe`를 사용하여 값이 확실히 메모리에 고정되어 있다는 것을 표현하는데, `unsafe`를 직접 쓰지 않고도 `pin!()` 매크로를 활용하여 stack pinning을 할 수 있다. 이러한 것을 safe abstraction이라 부른다. `unsafe`코드를 안전하게 사용할 수 있게 하는 추상화이다.

아래는 `pin!` 매크로의 예시이다.

```rust
use tokio::pin;

fn main() {
    //future 정의
    let future = async { 12 };
    //future를 stack pinning함
    pin!(future);
}
```

**매크로 확장 후(보기 좋게 편집됨)**

```rust
use tokio::pin;

fn main() {
    //future 정의
    let future = async { 12 };
    //future를 mut로 rebinding함
    let mut future = future;
    #[allow(unused_mut)]
    //future에 대한 Pin 포인터를 생성함.
    //unsafe를 쓰는것이 안전함. future는 shadowing되어서 Pin 포인터 이외의 방법으로 접근 불가능함.
    let mut future = unsafe { std::pin::Pin::new_unchecked(&mut future) };
}
```

stack pinning에서의 활용은 고급 내용이기 때문에 지금 당장 이해하지 못하는 것이 당연하다. 이해되지 않으면 그냥 넘겨도 좋다.
나중에 pinning을 학습한 이후 이 내용을 다시 보면 이해할 수 있을 것이다.

## 결론

Shadowing에 대해 자세히 알아보고, 왜 쓰이고 어디에 쓰이는지 깊게 알아보았다.

1. Rust에서 Shadowing이란 이미 binding 되어있는 변수의 binding을 바꾸는 것이다.
2. binding이 바뀌어도 해당 데이터의 lifetime과 borrowing rule은 해당 데이터가 생성된 scope의 영향을 받는다.
3. Shadowing은 여러 단계의 연산 과정이 필요한 곳, mutability가 더이상 필요하지 않은 곳에서 쓰인다.

오늘은 variable shadowing에 대해 자세히 알아보았고, 다른 언어와는 다른 Rust의 특수성에 의한 사용법을 알아보았다. variable shadowing을 적재적소에 사용하면 보다 안전하고 가독성이 좋은 Rust 코드를 작성하는데 분명히 도움이 될 것이다.
