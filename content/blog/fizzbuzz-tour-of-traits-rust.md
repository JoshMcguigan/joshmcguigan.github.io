---
title: A FizzBuzzy Tour of Traits in Rust
date: "2018-08-13"
---

Traits are a core part of the Rust programming language, and understanding traits, particularly those which are part of the standard library, is necessary in order to write idiomatic Rust. In this post I'll write several [FizzBuzz](https://en.wikipedia.org/wiki/Fizz_buzz) implementations, each demonstrating the use of a different trait from the Rust standard library. 

While I do think that knowing and using these traits is necessary for writing idiomatic Rust code, I would not claim that any one of these examples is _the best_ way to write FizzBuzz in Rust. [This blog post by Tom Dalling](https://www.tomdalling.com/blog/software-design/fizzbuzz-in-too-much-detail/) provides a great explanation for why 'optimizing' FizzBuzz is a bad idea. 

One last thing I want to mention before getting started is that there are at least a handful of traits which are more common (and arguably more useful) than the traits I am going to discuss here. Many traits which have obvious implementations can be automatically derived by adding `#[derive(TraitName)]` annotations to your code (a list of derive-able traits is available [here](https://doc.rust-lang.org/rust-by-example/trait/derive.html)). In this post, I've chosen to focus on a set of traits that I think are very useful, but cannot be automatically derived. 

## A Starting Point

Before beginning the discussion of Traits, I'll create a baseline implementation of FizzBuzz which will serve as a starting point for all of the examples. It is possible to write FizzBuzz in Rust with a style that would be familiar from many other languages.

```rust
for i in 1u32..=100 {

    let divisible_by_three = i % 3 == 0;
    let divisible_by_five = i % 5 == 0;

    if divisible_by_three && divisible_by_five {
        println!("FizzBuzz");
    } else if divisible_by_three {
        println!("Fizz");
    } else if divisible_by_five {
        println!("Buzz");
    } else {
        println!("{}", i);
    }

}
```

Rust has a feature called pattern matching which can be used to express these kind of if-else statements, while providing a guarantee that you haven't missed an edge case. Using match, you'd get:

```rust
for i in 1u32..=100 {

    let divisible_by_three = i % 3 == 0;
    let divisible_by_five = i % 5 == 0;

    match (divisible_by_three, divisible_by_five) {
        (true, true) => println!("FizzBuzz"),
        (true, false) => println!("Fizz"),
        (false, true) => println!("Buzz"),
        (false, false) => println!("{}", i),
    }

}
```

Even in this simple example, we are already making use of traits from the standard library. One I'd like to discuss in particular is the `Display` trait, which we use in the last arm of the match statement when we write `println!("{}", i)`. In this case, `i` is of type `u32`, and `u32` implements the `Display` trait. The `Display` trait is used whenever you use the format/print/println family of macros with the placeholder `{}`. Later we'll derive the `Display` trait ourselves on a new type.

For now let's move on to the discussion of traits, using this code as a rough starting point.

## Trait Bounds and IntoIterator

Lets say you want to write a function which can take a `vec` of integers and run FizzBuzz on that `vec`.

```rust
fn fizzbuzz_vec(nums: &Vec<u32>) {
    for num in nums {
        let divisible_by_three = num % 3 == 0;
        let divisible_by_five = num % 5 == 0;

        match (divisible_by_three, divisible_by_five) {
            (true, true) => println!("FizzBuzz"),
            (true, false) => println!("Fizz"),
            (false, true) => println!("Buzz"),
            (false, false) => println!("{}", num),
        }
    }
}
```

But what if you also wanted a function to support FizzBuzzing over an array, rather than a vec, of integers? You could write a second function to handle arrays, but this is the perfect use case for trait bounds. 

`IntoIterator` is a trait from the standard library which says a type can be converted into an iterator. The `for-in` syntax we are using requires a type which implements either the `IntoIterator` or the `Iterator` trait, and the standard library implements the `IntoIterator` trait for both vectors and arrays. 

Using this to write a function which takes any type that implements `IntoIterator` for items of type `u32` (meaning the iterator will return items of type `u32`) looks like this:

```rust
fn fizzbuzz<'a, T>(nums: T)
    where T: IntoIterator<Item = &'a u32>
{
    for num in nums {
        let divisible_by_three = num % 3 == 0;
        let divisible_by_five = num % 5 == 0;

        match (divisible_by_three, divisible_by_five) {
            (true, true) => println!("FizzBuzz"),
            (true, false) => println!("Fizz"),
            (false, true) => println!("Buzz"),
            (false, false) => println!("{}", num),
        }
    }
}
```

Either of these functions can be called with a vec as the argument, but the second can also be called with an array.

```rust
let nums : Vec<u32> = vec![1,2,3,4,5,15];
fizzbuzz_vec(&nums);

let nums : Vec<u32> = vec![1,2,3,4,5,15];
fizzbuzz(&nums);

let nums: [u32; 6] = [1,2,3,4,5,15];
fizzbuzz(&nums);
```

## From/Into and Display

The `From` trait is used to convert from one type to another. In this case we'll create a `FizzBuzz` enum, and implement `From` so we can easily convert from a `u32` to a `FizzBuzz`. 

```rust
#[derive(Debug)]
enum FizzBuzz {
    Fizz,
    Buzz,
    FizzBuzz,
    Other(u32),
}

impl From<u32> for FizzBuzz {
    fn from(item: u32) -> Self {
        match (item % 3 == 0, item % 5 == 0) {
            (false, false) => FizzBuzz::Other(item),
            (true, false) => FizzBuzz::Fizz,
            (false, true) => FizzBuzz::Buzz,
            (true, true) => FizzBuzz::FizzBuzz,
        }
    }
}

for i in 1..=100 {
    println!("{:?}", FizzBuzz::from(i));
}
```

An additional benefit of implementing the `From` trait is the `Into` trait is automatically implemented. In most cases, using into requires specifying the type you want to convert to because there will not be enough information for the compiler to infer it. An example of this is below:

```rust
for i in 1..=100 {
    let fizzbuzz : FizzBuzz = i.into();
    println!("{:?}", fizzbuzz);
}
```

Note that while we derive `Debug` on our enumeration and rely on that to print the variants (`{:?}` is used to debug print), we could implement our own display logic with the `Display` trait. While the `Fizz`, `Buzz`, and `FizzBuzz` variants are handled like we'd want already, the `Other` variant should be changed so that it only prints the number (the derived `Debug` implementation prints `Other(x)`). 

```rust
impl Display for FizzBuzz {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        match self {
            FizzBuzz::Other(n) => write!(f, "{}", n),
            _ => write!(f, "{:?}", self),
        }
    }
}
```

## Fn

The `Fn` family of traits, `Fn`/`FnMut`/`FnOnce`, is used to mark that a type can be called like a function. Using this, we can create a struct which takes four functions as constructor arguments, and then exposes an eval method which takes a `u32` as input and runs one of the four functions depending on whether the input is divisible by three, divisible by five, divisible by both three and five, or divisible by neither three nor five. 

```rust
struct FizzBuzzer<FnFizz, FnBuzz, FnFizzBuzz, FnOther> {
    fn_fizz: FnFizz,
    fn_buzz: FnBuzz,
    fn_fizzbuzz: FnFizzBuzz,
    fn_other: FnOther,
}

impl<FnFizz, FnBuzz, FnFizzBuzz, FnOther> FizzBuzzer<FnFizz, FnBuzz, FnFizzBuzz, FnOther>
    where
        FnFizz: Fn(),
        FnBuzz: Fn(),
        FnFizzBuzz: Fn(),
        FnOther: Fn(u32),
{
    fn new(fn_fizz: FnFizz, fn_buzz: FnBuzz, fn_fizzbuzz: FnFizzBuzz, fn_other: FnOther) -> Self {
        Self { fn_fizz, fn_buzz, fn_fizzbuzz, fn_other }
    }

    fn eval(&self, num: u32) {
        match (num % 3 == 0, num % 5 == 0) {
            (false, false) => (self.fn_other)(num),
            (true, false) => (self.fn_fizz)(),
            (false, true) => (self.fn_buzz)(),
            (true, true) => (self.fn_fizzbuzz)(),
        }
    }
}
```

Notice in the `where` clause on the `impl` statement, we specify that `FnFizz`/`FnBuzz`/`FnFizzBuzz` are of type `Fn()`, a function which takes no arguments and provides no return value, and `FnOther` is of type `Fn(u32)`, a function which takes a single `u32` argument and provides no return.

The struct can be instantiated by passing four closures to the `new` method, and used by passing `u32` types to the `eval` method, as shown below.

```rust
let fizzbuzzer = FizzBuzzer::new(
    || println!("Fizz"),
    || println!("Buzz"),
    || println!("FizzBuzz"),
    |num| println!("{}", num),
);

for i in 1..=100 {
    fizzbuzzer.eval(i);
}
```

## Iterator

As mentioned in the section on trait bounds, implementing either the `Iterator` or `IntoIterator` trait in Rust allows for your type to be used in for loops.

To implement `Iterator` on a type only requires implementing a single method, `fn next(&mut self) -> Option<Self::Item>` where `Self::Item` is the type that the iterator will return. To start, we'll create a struct to contain the state of the iterator.

```rust
struct FizzBuzzer {
    next: u32,
    max: u32,
}

impl FizzBuzzer {
    fn new(starting_value: u32, length: u32) -> Self {
        let max = if length > 0 { starting_value + length - 1 } else { 0 }; // protect from underflow
        FizzBuzzer { next: starting_value, max }
    }
}
```

Then we implement `Iterator` for our new type by creating a `next` method which will return an `Option<String>`.

```rust
impl Iterator for FizzBuzzer {
    type Item = String;

    fn next(&mut self) -> Option<Self::Item> {
        if self.next > self.max { return None }

        let s = match (self.next % 3 == 0, self.next % 5 == 0) {
            (false, false) => format!("{}", self.next),
            (true, false) => String::from("Fizz"),
            (false, true) => String::from("Buzz"),
            (true, true) => String::from("FizzBuzz"),
        };

        self.next += 1;

        Some(String::from(s))
    }
}
```

Note, this code could be improved by using the clone on write type, as discussed in [this Rust-based FizzBuzz deep-dive by Chris Morgan](https://chrismorgan.info/blog/rust-fizzbuzz.html).

With `Iterator` implemented on the `FizzBuzzer` type, it can be used as follows.

```rust
for text in FizzBuzzer::new(1, 100) {
    println!("{}", text);
}
``` 

## Summary

Writing idiomatic Rust code requires not only understanding traits on a conceptual level, but also having a familiarity with the traits which are part of the Rust standard library. In typical Rust fashion, many of the common standard library traits can be automatically derived if they have an obvious implementation. In this post, I've covered some of the traits which integrate very tightly with the language, but which cannot be automatically derived. Using these traits, and implementing them on your own types where appropriate, will significantly improve the quality of the Rust code you produce. 

#### Edits

 1. Modified code examples to use `u32` rather than `usize` based on [this discussion](https://www.reddit.com/r/rust/comments/96y6gp/a_fizzbuzzy_tour_of_traits_in_rust/e4491si/)
