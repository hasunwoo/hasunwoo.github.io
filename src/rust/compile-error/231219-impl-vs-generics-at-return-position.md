# 반환 위치에서의 impl Trait vs Trait bounds

2023.12.19

---

아래의 두 함수를 비교해 보자.

```rust
# fn main() {}
fn returns_iterator_bounds<O>() -> O
where
    O: Iterator<Item = (usize, I)>,
{
    0..100 // Range<usize> 구조체
}
```

```rust
# fn main() {}
fn returns_iterator_impls() -> impl Iterator<Item = usize> {
    0..100 // Range<usize> 구조체
}
```

위의 두 함수는 같은 역할을 하는 함수이다. 그러나 첫번째 함수 `returns_iterator_bounds()`를 컴파일하면 아래와 같은 에러가 발생한다.
```
error[E0308]: mismatched types
 --> src/main.rs:6:5
  |
2 | fn returns_iterator_bounds<O>() -> O
  |                            -       -
  |                            |       |
  |                            |       expected `O` because of return type
  |                            |       help: consider using an impl return type: `impl Iterator<Item = (usize, I)>`
  |                            this type parameter
...
6 |     0..100 // Range<usize> 구조체
  |     ^^^^^^ expected type parameter `O`, found `Range<{integer}>`
  |
  = note: expected type parameter `O`
                     found struct `std::ops::Range<{integer}>`
```

위 에러 메세지를 자세히 보자. Type parameter `O`를 반환해야 하는데, `Range<{integer}>` 타입을 반환하므로 타입 불일치가 발생하였다. 컴파일러는 친절하게 `impl Iterator<Item = (usize, I)>`를 반환 타입으로 사용하라고 한다. 두 번째 함수인 `returns_iterator_impls()`와 동일한 방법이다.

위의 두 가지 방식의 차이를 이해하기 위해서 [Rust Reference의 'Impl trait type' 챕터](https://doc.rust-lang.org/stable/reference/types/impl-trait.html#differences-between-generics-and-impl-trait-in-return-position)를 학습하였다. 해당 내용을 아래에 정리하였다.

## generics과 impl Trait의 반환 위치에서의 차이

매개변수 위치에서는, `impl Trait`과 generic type parameter는 거이 동일한 역할을 한다. 그러나, 반환 위치에서는 이 둘에 큰 차이가 있다. `impl Trait`는 generic type parameter과 다르게, 함수가 반환 타입을 결정하고, 호출자는 반환 타입을 결정할 수 없다.
```rust
# fn main() {}
fn foo<T: Trait>() -> T {
    // ...
}
```
위의 함수 `foo()`의 반환 타입은 호출자가 결정할 수 있고, 함수는 `T`를 반환한다. 여기서 추가로 설명하자면, 해당 함수를 호출할 때, 호출자는 `foo::<Type>()`와 같은 turbofish syntax를 사용하여 generic type parameter를 지정할 수 있다. generic type parameter는 컴파일러에 의해 monomorphization 되어서 해당 타입이 적용된 버전의 함수가 개별로 만들어진다.

```rust
# fn main() {}
fn foo() -> impl Trait {
    // ...
}
```
위의 함수의 반환 타입은 호출자가 결정할 수 없다. 대신, 함수가 반환 타입을 결정하게 되고, 반환하는 값이 `Trait`을 구현한다는 것만 결정된다.

## 다시 살펴보기

이제 관련 내용을 공부했으니 위에서 제시한 두 함수를 다시 살펴보자.

```rust
# fn main() {}
fn returns_iterator_bounds<O>() -> O
where
    O: Iterator<Item = (usize, I)>,
{
    0..100 // Range<usize> 구조체
}
```
위의 함수는 컴파일 에러가 발생한다. 이 함수는 generic type parameter `O`를 반환한다. generic type parameter가 반환 위치에서 사용되면 함수의 호출자가 반환 타입을 결정한다. 그러나 위 함수는 `Range<usize>` 구조체를 임의로 반환하려 하기 때문에 반환 타입이 불일치하다는 컴파일 에러가 발생한다.

```rust
# fn main() {}
fn returns_iterator_impls() -> impl Iterator<Item = usize> {
    0..100 // Range<usize> 구조체
}
```
위의 함수는 정상적으로 컴파일된다. 이 함수는 반환 타입으로 `impl Iterator<Item=usize>`를 사용하므로, 호출자가 아닌 함수가 반환 타입을 결정한다. 호출자는 반환되는 타입이 `Iterator<Item=usize>`를 구현한다는 것만 알 수 있다. 이 타입이 정확히 뭔지는 숨겨져 있다(type erasure이 발생함). 위 코드에서 `Range<usize>` 구조체는 해당 Trait를 구현하고 있기 때문에 컴파일 에러가 발생하지 않는다.

## 결론

1. generic type parameter를 반환하는 함수의 반환 타입은 함수의 호출자가 결정한다.
1. `impl Trait`를 반환하는 함수의 반환 타입은 함수가 결정한다.

위에서 알아본 것과 같이, 함수의 반환 위치에서의 generic type parameter와 `impl Trait`는 엄청난 차이를 갖는다. 내가 Rust를 공부했을 때 저 내용은 못본거 같은데 오늘 Rust 코드를 작성하다가 알게 되어서 공부 후 정리해보았다.
