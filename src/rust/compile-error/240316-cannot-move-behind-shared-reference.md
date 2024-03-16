# Cannot move out of X which is behind a shared reference

2024.03.16

---

아래의 코드를 컴파일하려면 에러가 발생한다.

```rust
fn main() {
    let mut x: String = "a".to_owned();
    let r: &mut String = &mut x;
    //x의 소유권을 b로 이동. 이 때, mutable reference인 r을 dereference함.
    let b: String = *r;
}
```

아래는 컴파일 에러 메세지이다.

```
error[E0507]: cannot move out of `*r` which is behind a mutable reference
 --> src/main.rs:4:21
  |
4 |     let b: String = *r;
  |                     ^^ move occurs because `*r` has type `String`, which does not implement the `Copy` trait
  |
help: consider removing the dereference here
  |
4 -     let b: String = *r;
4 +     let b: String = r;
  |
```

위 에러 메세지를 보면 알 수 있듯이, Rust는 mutable reference가 가리키는 값의 소유권을 가져와서 빈 상태로 두는 것을 허용하지 않는다.
위 상황이 왜 안되는지 아래의 예시 코드를 통해 알아보자.

다음과 같은 코드를 보자.

```rust
fn main() {
    let mut x: String = "a".to_owned();
    //x를 mutable borrow함.
    moves_x(&mut x);
    //함수가 반환되어 x의 mutable borrow가 종료됨.
    //만약 moves_x 함수에서 x가 참조하는 값의 소유권을 이동하는것을 허용했다면
    //아래의 코드가 실행되면 해제된 메모리를 접근하게 됨.
    x.clear(); //String::clear(&mut x);와 동일
    //implicit call to drop(x);
    //x가 이중으로 drop됨.
}

fn moves_x(x: &mut String) {
    //컴파일 오류가 발생
    let x: String = *x;
    //x가 drop됨.
    //implicit call to drop(x);
}
```

```
error[E0507]: cannot move out of `*x` which is behind a mutable reference
  --> src/main.rs:13:21
   |
13 |     let x: String = *x;
   |                     ^^ move occurs because `*x` has type `String`, which does not implement the `Copy` trait
   |
help: consider removing the dereference here
   |
13 -     let x: String = *x;
13 +     let x: String = x;
   |
```

`moves_x()` 함수는 `&mut String`을 매개변수로 받는다.
이 함수가 호출될 때 mutable borrow를 매개변수로 전달하고, 함수 실행이 끝나면 mutable borrow는 종료된다.
따라서, 함수 실행 종료 이후에는 `x`를 다시 사용할 수 있다고 컴파일러가 판단한다.
이는 정상적인 판단이다. 그러나, `moves_x()` 함수가 매개변수로 받은 `&mut String`의 소유권을 가져온다면, 이러한 판단이 더 이상 유효하지 않게 된다.

따라서 다음과 같은 문제가 발생한다.
1. Double drop이 발생한다.
2. Use after free(or move)가 발생한다.
3. 컴파일러가 값의 이동 sementic을 정상적으로 추적할 수 없다. 즉, `main()` 함수에서 `x`가 이동된 사실을 인지하지 못하고 해당 값이 정상적으로 사용되고 있다.

따라서, Rust에서 모든 값은 항상 초기화된 상태여야 하며, 빈 값으로 두어서는 안된다. 이는 컴파일러가 "모든 값은 초기화 되어 있다"라 하는 가정을 어기게 되어 위와 같은 여러가지 문제를 일으킨다. 따라서, 컴파일러는 mutable reference를 통하여 값을 밖으로 이동하는 것을 허용하지 않는다.

## 해결 방법

위 문제의 본질은 mutable reference를 통하여 값이 이동될 경우 해당 값이 Invalid한 값을 가리키게 된다는 것이다. 이 문제를 해결하기 위한 방법에는 여러가지가 있다.

## 1. 값을 대입한다.

이전 값을 사용할 필요가 없을 경우에는 값을 대입할 수 있다.

```rust
fn main() {
    let mut x: String = "a".to_owned();
    //x를 mutable borrow함.
    clear_x(&mut x);
    //함수가 반환되어 x의 mutable borrow가 종료됨.
    //x에는 새로운 값이 들어가 있기 때문에 문제 없음.
    x.clear(); //String::clear(&mut x);와 동일
    //implicit call to drop(x);
    //새로 교체된 값을 drop하기 때문에 문제되지 않음.
}

fn clear_x(x: &mut String) {
    //x에 새로운 String을 할당하여 저장하였다.
    //이전 값(main함수에서 넘어온 값)은 drop되고 새로운 값으로 대체되었다.
    //컴파일러가 생성한 코드(실제로는 컴파일되지 않음.)
    /*
    drop(*x);
    *x = String::new();
    */
    *x = String::new();
}
```

위의 코드에서는 `*x`에 새로운 값을 대입하였다. 이렇게 하면 이전 값을 drop하는 동작이 컴파일러에 의해 추가된다. 이전 값이 필요하지 않을 경우에는 이 방법을 쓸 수 있다.

## 2. 값을 교환한다.(`std::mem::replace()` 또는 `std::mem::swap()` 이용)

이전 값이 필요할 경우에는 `std::mem::replace()` 또는 `std::mem::swap()`를 사용하여 이전 값을 drop하지 않고 소유권을 가져올 수 있다.

**1. `std::mem::replace()`**

```rust
pub fn replace<T>(dest: &mut T, src: T) -> T
```

위 표준 라이브러리 함수는 mutable reference인 `dest`, owned value인 `src`를 매개변수로 받고 owned value T를 반환한다.

위 함수의 개념적인 동작은 아래와 같다. 물론 실제로 컴파일 되지는 않고, unsafe를 사용하여 구현되어 있다.

**`std::mem::replace()` 함수의 개념적 동작**

```rust
pub fn replace<T>(dest: &mut T, src: T) -> T {
    let old = *dest; //실제로는 unsafe 사용하여 구현됨.
    *dest = src;
    old
}
```

**`std::mem::replace()` 사용 예시**

```rust
fn main() {
    let mut x: String = "a".to_owned();
    //x를 mutable borrow함.
    clear_x(&mut x);
    //함수가 반환되어 x의 mutable borrow가 종료됨.
    //x에는 새로운 값이 들어가 있기 때문에 문제 없음.
    x.clear(); //String::clear(&mut x);와 동일
    //implicit call to drop(x);
    //새로 교체된 값을 drop하기 때문에 문제되지 않음.
}

fn clear_x(x: &mut String) {
    //mutable reference인 x에 저장된 값의 소유권을 가져와서 old에 저장하고,
    //x의 위치에는 새로운 값의 소유권을 받아와서 저장한다.
    let old = std::mem::replace(x, String::new());
    println!("{old}");
}
```

이처럼, `std::mem::replace()`를 사용하면 이전 값의 소유권을 가져오는 동시에, 새로운 값의 소유권을 저장하여 문제를 방지할 수 있다.

**`std::mem::swap()` 함수의 개념적 동작**

```rust
pub fn swap<T>(x: &mut T, y: &mut T)
```

위 표준 라이브러리 함수는 mutable reference인 `x`, `y`를 매개변수로 받아와서 두 값을 drop하지 않고 교환한다.

위 함수의 개념적인 동작은 아래와 같다. 물론 실제로 컴파일 되지는 않고, unsafe를 사용하여 구현되어 있다.

**`std::mem::swap()` 함수의 개념적 동작**

```rust
pub fn replace<T>(dest: &mut T, src: T) -> T {
    //실제로는 unsafe 사용하여 구현됨.
    let tmp = *x; 
    *y = *x;
    *x = tmp;
}
```

**`std::mem::replace()` 사용 예시**

```rust
fn main() {
    let mut x: String = "a".to_owned();
    //x를 mutable borrow함.
    clear_x(&mut x);
    //함수가 반환되어 x의 mutable borrow가 종료됨.
    //x에는 새로운 값이 들어가 있기 때문에 문제 없음.
    x.clear(); //String::clear(&mut x);와 동일
    //implicit call to drop(x);
    //새로 교체된 값을 drop하기 때문에 문제되지 않음.
}

fn clear_x(x: &mut String) {
    //소유권의 이동 없이 값만 교환한다.
    let mut v = String::new();
    std::mem::swap(x, &mut v);
    println!("{v}");
}
```

이처럼, `std::mem::swap()`을 사용하면 소유권의 이동 없이 두 값을 교환할 수 있다. 이 방법도 `std::mem::replace()`와 마찬가지로 두 값 모두 초기화된 상태로 유지하여 안전한 방법이다.

## 3. 기본값을 저장한다.(`std::mem::take()` 이용)

이전 값만 필요할 경우에는 `std::mem::take()`를 사용하여 이전 값의 소유권을 가져오고 이전 위치에 기본값을 저장할 수 있다. Rust에서 기본값은 `Default::default()` Trait을 통해서 정의할 수 있다.

```rust
pub fn take<T>(dest: &mut T) -> T
where
    T: Default,
```

위 표준 라이브러리 함수는 mutable reference인 `dest`를 매개변수로 받고 `dest`는 `Default` trait를 구현해야 한다. 위 함수는 `dest`가 참조하는 값의 소유권을 가져오는 동시에 기본값(`dest::default()`)을 `dest`에 저장한다.

**`std::mem::take()` 함수의 개념적 동작**

```rust
pub fn take<T>(dest: &mut T) -> T
where
    T: Default
{
    //실제로는 unsafe 사용하여 구현됨.
    let old = *dest;
    *dest = Default::default();
    old
}
```

**`std::mem::take()` 사용 예시**

```rust
fn main() {
    let mut x: String = "a".to_owned();
    //x를 mutable borrow함.
    clear_x(&mut x);
    //함수가 반환되어 x의 mutable borrow가 종료됨.
    //x에는 새로운 값이 들어가 있기 때문에 문제 없음.
    x.clear(); //String::clear(&mut x);와 동일
    //implicit call to drop(x);
    //새로 교체된 값을 drop하기 때문에 문제되지 않음.
}

fn clear_x(x: &mut String) {
    //x의 소유권을 이전받고, 원래 x의 위치에는 기본값(Default::default())를 넣는다.
    let old = std::mem::take(x);
    println!("{old}");
}
```

이처럼, `std::mem::take()`를 사용하면 해당 mutable reference의 소유권을 가져오는 동시에 기본값을 저장하여 mutable reference가 가리키는 값이 유효하도록 한다.

## 4. `Option<T>`를 사용한다.

만약 특정 값의 비어있는 상태를 명시적으로 나타내고 싶거나, 새로운 객체 생성에 많은 비용이 들 경우 `Option<T>` enum을 사용할 수 있다. 또한, 객체를 생성할 수 없을 때 유용하게 사용할 수 있다.

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` enum은 값이 존재하는 상태와 값이 없는 상태 중 하나를 나타낸다. 이를 사용하여 값이 비어있는 상태를 명시적으로 표현할 수 있다.

`std::mem::take()`와 비슷하게, 표준 라이브러리 메서드인 `Option::take()`를 사용하면 `Option::Some(T)`가 저장되어 있을 경우 `T`의 소유권을 가져오고 그 대신 `None`을 저장한다.
이 방법은 기본값을 생성하여 저장하지 않고도 빈 값을 효율적으로 나타낼 수 있다.

**`Option::take()` 함수의 개념적 동작**

```rust
impl<T> Option<T> {
    fn take(&mut self) -> Option<T> {
        //&mut self에 None을 저장하고 이전 값을 가져온다.
        std::mem::replace(self, None)
    }
}
```


**`Option::take()` 사용 예시**

```rust
fn main() {
    let mut x: Option<String> = Some("a".to_owned());
    //x를 mutable borrow함.
    clear_x(&mut x);
    //함수가 반환되어 x의 mutable borrow가 종료됨.
    match x {
        Some(x) => x.clear(), //String::clear(&mut x);와 동일
        None => {}, //x에 값이 저장되어 있지 않음. 해당 경우 처리.
    }
}

fn clear_x(x: &mut Option<String>) {
    //x의 소유권을 이전받고 x에 None을 저장.
    let old = x.take();
    println!("{old}");
}
```

## 결론

1. Rust 컴파일러는 모든 값이 항상 초기화된 상태라는걸 가정한다.
1. 따라서, Rust에서는 mutable reference가 가리키는 값의 소유권을 함부로 이동할 수 없다. 이동이 가능하다면 해당 값이 비어있다는 사실을 추적할 수 없어서 여러가지 메모리 안전성 문제가 발생한다.
1. 해당 문제를 해결하기 위해서는 `std::mem::replace()`, `std::mem::swap()`, `std::mem::take()`를 사용하여 소유권을 가져오는 동시에 mutable reference가 빈 값을 가리키지 않게 빈 공간을 다른 값으로 대체할 수 있다.
1. 위의 방법이 적절하지 않거나, 값이 빈 상태를 명시적으로 표현하고 싶다면 `Option<T>`를 사용할 수 있다.

이 글에서는 왜 mutable reference가 가리키는 값의 소유권을 함부로 가져올 수 없는지, 이에 따라 발생할 수 있는 여러 메모리 안전성 문제와 해결방법에 대해 알아보았다.
