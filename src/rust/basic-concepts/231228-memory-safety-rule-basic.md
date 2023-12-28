# 메모리 안전성을 위한 기본 규칙 - Ownership, Borrowing, Ailiasing, Lifetime의 이해

2023.12.28

---

Rust는 메모리 안전성을 컴파일 시점에 보장하기 위해 다양한 규칙을 강제한다. 기본 규칙에는 Ownership, Borrowing, Ailiasing, Lifetime 규칙이 있다.
이 글에서는 이러한 규칙들이 어떻게 적용되는지 하나씩 자세히 알아볼 것이다.

# Ownership(소유권) 규칙

## 1. 모든 값은 Owner(주인)을 갖는다.

Rust에서 모든 값은 Owner(주인)을 갖는다. 여기서 Owner는 해당 값이 생성된 Scope가 된다.
이름이 없거나 `_`인 값은 사용 완료 즉시 drop(해제)된다. 이를 temporarly value(임시 값)이라고 부른다.

```rust
fn main() {
    String::from("123"); //이름이 없는 값이기 때문에 이 문장 실행 후 즉시 해제.
    let _name = String::from("123"); //이름이 `_'가 아니기 때문에 '_name'의 Owner인 main함수의 scope가 종료될 때 해제됨.
    let _ = String::from("123"); //이름이 `_`이기 때문에 이 문장 이후로 즉시 해제됨.
    let value = 123; //값 '123'의 owner는 main함수의 scope가 됨. main() 함수의 scope가 종료될 때 해제됨.
    //main()함수의 scope 종료. '_name', 'value' 가 여기서 drop(해제)됨.
}
```

### 1-2. 값을 비워둘 수 없고, 초기화되기 전에 사용될 수 없다.

```rust
fn main() {
    let name: String;
    //컴파일 에러!! 초기화되지 않은 값 'name'을 사용할 수 없음.
    println!("{name}");
}
```

```rust
fn main() {
    let mut name = String::from("123");
    let r = &mut name;
    let new_name = *r;
}
```

**출력**
```
error[E0507]: cannot move out of `*r` which is behind a mutable reference
 --> src/main.rs:4:20
  |
4 |     let new_name = *r;
  |                    ^^ move occurs because `*r` has type `String`, which does not implement the `Copy` trait
  |
help: consider removing the dereference here
  |
4 -     let new_name = *r;
4 +     let new_name = r;
  |
```

위의 경우, `name`의 값을 `new_name`으로 dereference하여 move(이동)하려 하고 있다. 이동될 경우, `name`이 비워지기 때문에 오류가 발생한다. mutable reference가 참조하는 값이 이동되어 빈 값이 될 경우 컴파일러가 인지하지 못하여 undefined behavior가 발생할 수 있기 때문에 이를 금지한다. 이런 경우에는 `std::mem::take()`를 써서 기본값으로 대체하거나, `std::mem::replace()`를 써서 다른 값을 이동하고 원래 값의 소유권을 가져오거나, `Option<T>`를 써서 빈 값을 저장할 수 있게 해야 한다.

## 2. Ownership(소유권)은 move(이동)될 수 있다.

Ownership은 이동될 수 있다. 크게 두 가지 경우에서 이동이 일어난다. 첫째는 `=` 연산자(assignment operator)를 통해서 이동이 일어날 수 있고, 둘째로는 함수 호출을 통해서 이동이 일어날 수 있다.

아래는 `=` 연산자를 통해 이동이 일어나는 예시이다.

```rust
fn main() {
    let s = String::from("1234");
    let new = s; //s의 소유권이 new로 이동됨. s는 사용할 수 없음.
    println!("{s}"); //컴파일 오류!! use of moved value
    println!("{new}");
}
```

아래는 함수 호출을 통해 이동이 일어나는 예시이다.

```rust
fn main() {
    let s = String::from("1234");
    takes_string(s); //이동이 일어남. takes_string함수에서 String을 인자로 받기 때문.
    println!("{s}"); //컴파일 오류!! use of moved value
}

fn takes_string(s: String) { }
```

Rust에서 값은 크게 `Copy`타입과 아닌 경우로 나눌 수 있다. `Copy`는 marker trait으로, 특정 값이 메모리 복사만으로 이동될 수 있음을 나타낸다. 대표적인 `Copy`타입의 예로는 primitive type에 속하는 정수(`usize`, `isize` 등등)과 실수(`f32`, `f64`), `bool`이다. `Copy`타입의 값들은 단순하고 크기가 작기 때문에 메모리 복사만으로 쉽게 이동할 수 있다. 또한, scope가 종료될 때 `drop()`이 호출되지 않고, `Drop` trait을 구현할 수도 없다.
만약 struct의 모든 field가 `Copy`타입인 경우, `#[derive(Copy)]` 매크로를 이용하여 `Copy`타입으로 만들 수 있다.

### 2-1. `Copy` 타입이 아닌 경우

대표적인 `Copy`타입이 아닌 경우로는 `String`이 있다. 문자열은 크기가 가변적이고, heap에 할당되기 때문에 복잡하고 크기가 크다. 또한, 올바른 메모리의 할당과 해제가 필요하다.
`Copy` 타입이 아닌 경우, 이동이 일어난다. 만약 해당 객체가 `Clone` trait을 구현하면 `clone()`메서드를 호출하여 복제할 수 있다. 이 경우 내부의 리소스가 올바르게 복사된다. 이러한 작업에는 새로운 heap 메모리 블럭을 할당하고 기존의 문자열 데이터를 복사하는 작업이 포함된다. 이 경우, 리소스가 소요되고 무겁기 때문에 명시적으로 `clone()`을 호출하여야 한다.

```rust
fn main() {
    let s = String::from("1234");
    takes_string(s); //이동이 일어남. takes_string함수에서 String을 인자로 받기 때문.
    println!("{s}"); //컴파일 오류!! use of moved value
}

fn takes_string(s: String) { }
```

**`clone()` 사용**

```rust
fn main() {
    let s = String::from("1234");
    takes_string(s.clone()); //이동이 일어남. s를 복제하여 이동함.
    println!("{s}"); //s는 복제되었기 때문에 원본은 이동되지 않음. 따라서 정상적으로 작동함.
}

fn takes_string(s: String) { }
```

### 2-2. `Copy` 타입인 경우
primitive type인 `usize`, `f32`, `bool`과 같은 경우는 크기기 작고, 내부 구조가 복잡하지 않기 때문에 메모리 복사만으로 이동할 수 있고, 리소스 할당 및 해제에 필요한 `Drop` trait이 필요하지 않다. 따라서, `Copy`타입의 값은 이동되지 않고, 복사된다.

```rust
fn main() {
    let num = 5;
    takes_number(num); //복사가 일어남. Copy 타입이기 때문에 원본은 그대로 유지됨.
    takes_number(num); //복사가 일어남. Copy 타입이기 때문에 원본은 그대로 유지됨.
    println!("{num}"); //원본이 이동되지 않고 복사되었기 때문에 정상적으로 동작함.
}

fn takes_number(num: usize) { }
```

## 3. 값의 Owner(주인)은 자신의 생명이 끝날 때 값들을 정리(Drop)해야 한다.

값의 Owner(주인) Scope는 자신이 끝날 때 값들을 정리(Drop)하여 리소스를 해제해야 한다. 이 때, `Copy`타입의 값은 `Drop` trait 구현이 불가능하고, 리소스 할당 및 해제가 필요하지 않기 때문에 `Drop`이 호출되지 않는다.

```rust
fn main() {
    let outer = Test { name: String::from("outer") }; //outer의 주인은 main함수의 scope가 됨.
    {
        let copy_type = 123; // 정수형은 Copy 타입이기 때문에 drop이 호출되지 않는다.
        let inner = Test { name: String::from("inner") }; //inner의 주인은 이 scope가 됨.
        //여기서 inner가 속한 scope가 종료됨. inner의 drop이 호출됨.
    }
    //여기서 main()함수의 scope가 종료됨. outer의 drop이 호출됨.
}

struct Test {
    name: String,
}

impl Drop for Test {
    fn drop(&mut self) {
        println!("{} Dropped", self.name);
    }
}
```

**출력**

```
inner Dropped
outer Dropped
```

`main()`함수의 내부에 정의한 scope가 먼저 종료되어 `inner`가 먼저 drop되었고, 그 다음 `main()`함수의 scope가 종료되어 `outer`가 drop되었다.
값 `copy_type`은 primitive type인 `Copy`타입이기 때문에 리소스 해제가 필요 없으므로 drop이 호출되지 않는다.

# Borrowing(빌림) 규칙

## 1. Borrowing(빌림) 이란?

Rust에서는 값을 move(이동)하는것 뿐만 아니라 borrow(빌림)할 수 있다. 값을 빌리게 되면 해당 값의 메모리 주소가 전달되게 된다. 이는 c언어의 포인터나 c++언어의 레퍼런스와 비슷한 개념이나, Rust에서는 메모리 안전성을 위해 엄격한 규칙이 적용된다.

## 2. Mutability(가변성) 이란?

Mutability는 값을 변경할 수 있는지의 여부를 뜻한다. Rust에서 모든 값 및 reference는 immutable by default(기본적으로 변경 불가)이다. 이는 값의 안전한 사용과 컴파일러 최적화를 위함이다.

`let mut xxx = xxx;`를 통해 수정 가능한 상태의 값을 할당할 수 있다. `&mut xxx`를 통해 값의 mutable reference(수정 가능한 레퍼런스)를 얻어올 수 있다. 만약 할당된 값이 `let xxx = xxx;`와 같이, `mut`키워드 없이 할당된 경우, mutable reference를 얻어올 수 없다.

## 3. Ailiasing Rule 알아보기

하나의 데이터는 여러개의 immutable borrow를 가질 수 있다. 그러나, immutable borrow가 1개라도 있는 경우, mutable borrow를 가질 수 없다. 즉, 하나의 데이터는 오직 하나의 mutable borrow를 가질 수 있다. 데이터가 immutable borrow를 하나라도 가지고 있는 경우, 원래 데이터는 수정할 수 없고 mutable borrow를 가질 수 없다. 데이터가 mutable borrow를 가지고 있는 경우, 원래 데이터는 수정할 수 없고 immutable borrow를 가질 수 없다.

```rust
fn main() {
    //mut 값 선언
    let mut value = 10;
    //여러 개의 immutable borrow 가능함
    let r1 = &value;
    let r2 = &value;
    //컴파일 오류! 하나의 immutable borrow라도 있으면 mutable borrow 불가
    let r3 = &mut value;
    println!("{r1}");
    println!("{r2}");
}
```

```rust
fn main() {
    //mut 값 선언
    let mut value = 10;
    //하나의 mutable borrow만 가능
    let r1 = &mut value;
    //컴파일 오류! mutable borrow와 immutable borrow는 동시에 불가능
    let r2 = &value;
    //r1값을 mutable borrow 써서 수정
    *r1 = 20;
    //컴파일 오류! mutable borrow 또는 immutable borrow가 있으면 원래 값 수정 불가
    value = 30;
    println!("{r1}");
}
```

컴파일러는 immutable borrow나 mutable borrow가 마지막으로 사용되는 시점을 기준으로 해당 borrow를 비활성화(Invalidate)한다. 이를 NLL(Non Lexical Lifetime)이라고 한다. 따라서, 모든 immutable borrow가 마지막으로 사용된 이후에는 원래 값을 수정하거나 mutable borrow를 가져올 수 있다. mutable borrow가 마지막으로 사용된 이후에는 원래 값을 수정하거나, immutable borrow를 가져올 수 있다. 위 규칙이 제대로 작동하지 않는 경우도 가끔 있다. 아직 Rust 컴파일러가 완벽하지 않기 때문이다. 이럴 때는 `drop()`을 호출하여 사용 완료된 reference를 직접 비활성화 할 수 있다.

예전의 Rust 컴파일러는 Lexical Lifetime를 사용하였다. Lexical Liftime은 해당 변수가 선언된 scope를 기준으로 lifetime을 결정한다. 그러나, 최근 도입된 NLL(Non Lexical Lifetime)은 scope 기준이 아닌, 해당 변수의 사용 패턴을 기준으로 lifetime을 결정한다. NLL(Non Lexical Lifetime)은 Rust 2018 에디션에서 도입된 기능으로, 별도의 글에서 자세히 다룰 예정이다. 아래는 Reference에서 해당 규칙이 어떻게 적용되는지에 대한 예시이다.

```rust
fn main() {
    //mut 값 선언
    let mut value = 10;
    //여러 개의 immutable borrow 가능함
    let r1 = &value;
    let r2 = &value;
    println!("{r1}");
    println!("{r2}");
    //r1, r2 비활성화됨. 여기 이후로 사용되지 않으므로 lifetime이 종료됨. 따라서, value는 다시 활성화됨
    //value의 immutable borrow가 전부 비활성화 되었으므로, mutable borrow가 가능해짐 
    let r3 = &mut value;
    *r3 = 30;
    println!("{r3}");
}
```

이러한 규칙은 하나의 데이터가 여러 부분에서 수정되는 것을 방지하여, race condition, invalid(or dangling) reference와 같은 문제를 방지한다. `Vec<T>`와 같은 경우, mutable reference를 이용하여 값을 삽입할 때 할당되어 있는 메모리가 부족한 경우 새로운 메모리 블럭을 할당받아 이전의 데이터를 새로운 메모리 블럭으로 옮긴 후 이전 메모리 블럭을 해제한다. 이 때, 이전 데이터의 immutable reference는 해제된 메모리를 가리키게 되어 dangling reference가 된다. 이를 사용하려고 할 경우 use after free 위반이 발생하게 된다. 아래는 Ailiasing Rule가 어떻게 이러한 문제를 해결하는지를 보여준다.

```rust
fn main() {
    //vector 선언
    let mut v = vec![1, 2];
    let first: &usize = &v[0]; //&v -> &usize. v를 borrow하여 첫번째 값의 reference를 가져온다.
    //컴파일 오류! immutable borrow `first`가 활성화 되어있기 때문에 push()를 호출하기 위한 &mut v를 요청할 수 없음
    v.push(3); //3을 추가함. 공간이 부족하면 더 큰 메모리 블럭을 재할당하여 이동함
    println!("{first}"); //메모리 위반(use after free)!! 메모리 블럭이 재할당되면 first가 해제된 값을 가리키게 됨.
    //first가 여기 이후로 사용되지 않음. first 비활성화됨
}
```

위 코드는 컴파일되지 않는다. 만약 컴파일된다면 use after free 위반이 발생할 가능성이 크다. 

또한, rust 컴파일러가 사용하고 있는 llvm 백엔드에 noalias 정보를 제공하여 최적화를 가능하게 한다. llvm 컴파일러는 이러한 정보를 이용하여 최적화된 기계어를 생성한다. 예를 들어, immutable reference가 함수의 매개변수로 들어오는 경우, 해당 값을 여러 번 참조하는 대신 해당 값을 최초 한 번만 읽어서 저장 후 재사용(캐싱)할 수 있다. 또한, `Copy`타입의 값이나, 충분히 크기가 작을 경우 참조를 전달하는 대신 값을 통채로 stack에 복사하여 call by value로 전달할 수 있다. 이러한 최적화 때문에 unsafe 코드 작성시 특히 유의해야 한다. 이는 추후 unsafe관련 글에서 따로 다룰 예정이다.

```rust
fn main() {
    let num = 1;
    print_num(&num);
}

fn print_num(num: &usize) {
    println!("{num}");
}
```

위의 `print_num()` 함수를 아래와 같이 컴파일러가 최적화할 수 있다. 만약 `num`이 mutable reference면 이러한 최적화는 불가능하다. 따라서 mutable reference와 immutable reference를 구분해서 사용하는 것은 최적화에도 도움이 된다.

```rust
fn main() {
    let num = 1;
    //num이 수정될 수 없기 때문에 값으로 전달. 충분히 작고, Copy타입이기 때문.
    //간접 참조가 발생하지 않아서 성능이 향상됨.
    print_num(num);
}

fn print_num(num: usize) {
    println!("{num}");
}
```

# Lifetime(생명) 규칙

Lifetime에 대해 알아보자. Rust 컴파일러는 모든 reference에 대한 수명을 추적한다. 이는 dangling reference를 컴파일 시점에 발견하여 use after free와 같은 위험한 메모리 사용을 방지한다. 수명이 끝난 값은 사용할 수 없다.
아래의 코드를 보자.

```rust
fn main() {
    let s: &str; //레퍼런스 s 정의
    {
        let owned = String::from("asdf"); //owned 값 저장
        s = &owned; //s에 owned의 레퍼런스를 저장
        println!("{s}"); //정상적으로 컴파일됨. owned가 살아있기 때문.
        //owned가 drop됨.
    }
    println!("{s}"); //컴파일 오류! s에 저장된 owned의 레퍼런스가 유효하지 않음.
}
```

**출력**
```
error[E0597]: `owned` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let owned = String::from("asdf");
  |             ----- binding `owned` declared here
5 |         s = &owned;
  |             ^^^^^^ borrowed value does not live long enough
6 |         println!("{s}");
7 |     }
  |     - `owned` dropped here while still borrowed
8 |     println!("{s}");
  |               --- borrow later used here
```

위의 경우에 컴파일러는 lifetime을 추적하여 `s`가 참조하는 값 `owned`의 수명이 끝났음을 인지하여 오류를 발생시킨다.

## 함수 호출에서의 lifetime 처리

Rust 컴파일러는 함수의 선언부를 확인하여 lifetime과 borrowing 규칙을 분석한다.
함수가 reference를 반환할 경우, 해당 reference의 lifetime은 매개변수로 들어오는 lifetime 중 하나와 연결된다. 이 때 reborrow가 일어나고, borrowing rule도 동일하게 적용된다. 아래의 예를 보자.

```rust
fn main() {
    let s = dangling();
    println!("{s}");
}

fn dangling() -> &str {
    let s = String::from("asdf"); //문자열 s 할당. 이 함수가 끝날때 drop됨
    &s //drop될 값을 반환하려고함. 오류 발생.
}
```

**출력**

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:6:18
  |
6 | fn dangling() -> &str {
  |                  ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
6 | fn dangling() -> &'static str {
  |                   +++++++

For more information about this error, try `rustc --explain E0106`.
error: could not compile `playground` (bin "playground") due to previous error
```

위의 코드에서 함수 `dangling()`은 해당 scope가 끝날 때 drop될 값인 지역변수 `s`를 반환하려고 하고 있다. 이는 dangling reference를 반환하여 해제된 메모리의 참조를 사용하는 문제가 발생한다. 여기서 주목할 점은 컴파일러 오류이다. 컴파일러는 함수의 선언부를 지적하고 있다. 함수의 반환 타입이 borrowed value인데, 빌려올 곳이 없다고 한다. 저 경우 `'static` lifetime annotation을 추가하여 해결할 수도 있다. `'static` lifetime을 가진 reference는 프로그램의 전체 실행 시간 동안 유효하다. 그러나 저 경우에서 반환 타입의 lifetime을 `'static`으로 변경하면 함수의 구현부에서 오류가 발생한다. 함수가 반환하려는 값의 lifetime은 `'static`보다 짧은 함수가 반환될 때 drop되기 때문에 오류가 발생한다.

lifetime annotation은 함수, 구조체에서 lifetime 관계를 표시할 때 사용한다. 함수의 선언부에서 매개변수로 받은 reference와 반환되는 reference의 lifetime 관계를 표시하고, 구조체에서는 구조체가 reference를 포함하고 있는 경우, 해당 reference의 lifetime을 나타낼 때 사용한다. 이는 reference가 포함된 구조체가 함수의 반환형으로 사용될 때, 컴파일러가 lifetime을 올바르게 추적할 수 있도록 도와준다.
함수가 reference를 반환할 경우, 반환되는 reference는 동일한 lifetime을 가진 매개변수로 들어온 reference를 reborrow(다시 빌림)한다. 이는 dangling reference를 방지하고, borrowing rule이 함수의 매개변수로 들어온 reference와 반환되는 reference 사이에서도 지켜질 수 있도록 돕는다.
또한, 함수의 body가 컴파일 될 때, 해당 함수의 선언부의 lifetime 관계를 확인하여 올바른 reference를 반환하는지 검사한다.

lifetime annotation과 reborrow는 복잡한 주제이기 때문에 다른 글에서 자세히 다룰 예정이다.

## 결론

위에 제시된 Ownership, Borrowing, Ailiasing, lifetime 규칙은 Rust 초보자에게는 어려울 수도 있다. 그러나 Rust 언어를 효과적으로 사용하기 위해서는 꼭 알아야 한다.

1. 모든 값에는 Owner가 있다. Owner는 scope가 종료될 때 자기가 소유하고 있는 값을 drop해야 한다.
2. `Copy`타입이 아닌 경우 move가 일어나고, scope가 종료될때 drop이 호출된다. 값을 이동하고 싶지 않다면 빌리거나 `Clone` trait을 구현하는 경우 `clone()`을 호출하여 복제해야 한다. `Copy`타입인 경우 move 대신 복사가 일어나고, scope가 종료되더라도 drop이 호출되지 않는다.
3. Rust에서는 값을 빌릴 수 있다. 한번에 하나의 mutable reference 또는 여러개의 immutable reference만 빌릴 수 있다.
4. Lifetime은 해당 값의 수명을 나타내는 개념이다. 해당 값이 언제까지 유효한 지 컴파일러가 파악할 때 쓴다. 함수나 구조체의 선언부에 lifetime annotation을 사용하여 반환되는 reference가 어떤 매개변수와 연결되어 있는지 나타낼 수 있다.
