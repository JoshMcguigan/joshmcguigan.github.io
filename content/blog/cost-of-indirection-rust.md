---
title: The Cost of Indirection
date: "2020-02-29"
---

Providing zero cost abstractions is a goal of the Rust programming language. In this post we'll explore the perforamance costs of various methods of indirection.

## The benchmark

I wanted a benchmark which emphasized the cost of indirection, so I needed to minimize the amount of work done in any single call of a function. I settled on solving the nth odd number problem (a play on the nth prime problem), using a shockingly inefficient algorithm.

```rust
use std::time::Instant;

fn main() {
    let start_time = Instant::now();
    let result = find_nth_odd(1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}

fn find_nth_odd(n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

fn is_odd(n: u64) -> bool {
    n % 2 == 1
}
```

This finds the billionth odd number in 1042 ms. All performance numbers reported in this post are the average of 5 runs, using the 1.41 stable compiler running in release mode (i.e. `cargo run --release --bin $MY_BIN`).

## Struct impl

The simplest indirection is moving the `is_odd` function into a struct impl block.

```rust
use std::time::Instant;

fn main() {
    let finder = OddFinder;

    let start_time = Instant::now();
    let result = find_nth_odd(finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}

fn find_nth_odd(odd_finder: OddFinder, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

struct OddFinder;

impl OddFinder {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}
```

This is a great demonstration of Rust's zero costs abstractions, finding the billionth odd number in 1032 ms.

## Trait

What happens if the `is_odd` method is part of a trait, rather than directly on the struct?

```rust
use std::time::Instant;

fn main() {
    let finder = OddFinderOne;

    let start_time = Instant::now();
    let result = find_nth_odd(finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}

fn find_nth_odd(odd_finder: OddFinderOne, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

trait OddFinder {
    fn is_odd(&self, n: u64) -> bool;
}

struct OddFinderOne;

impl OddFinder for OddFinderOne {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}
```

Surprisingly moving this method behind a trait is not a zero cost abstraction, even though the `find_nth_odd` function still takes the specific `OddFinderOne` implementation. This version finds the billionth prime in 1340 ms.

## Generics

If multiple structs exist which are capable of finding odd numbers, we can use generics to allow our `find_nth_odd` method to use any of them.

```rust
use std::time::Instant;

fn main() {
    let finder = OddFinderOne;

    let start_time = Instant::now();
    let result = find_nth_odd(finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}

fn find_nth_odd<T: OddFinder>(odd_finder: T, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

trait OddFinder {
    fn is_odd(&self, n: u64) -> bool;
}

struct OddFinderOne;

impl OddFinder for OddFinderOne {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}

struct OddFinderTwo;

impl OddFinder for OddFinderTwo {
    fn is_odd(&self, _n: u64) -> bool {
        unimplemented!()
    }
}
```

This is another positive example of zero cost abstractions. It runs at essentially the same speed as the initial non-generic trait-based version of the program (above), finding the billionth prime in 1322 ms.

## Trait Object

Rather than making our `find_nth_odd` function generic, an alternate approach to allow it to accept any `OddFinder` implementation is to use trait objects.

```rust
use std::time::Instant;

fn main() {
    let finder = OddFinderOne;

    let start_time = Instant::now();
    let result = find_nth_odd(&finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}


fn find_nth_odd(odd_finder: & dyn OddFinder, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

trait OddFinder {
    fn is_odd(&self, n: u64) -> bool;
}

struct OddFinderOne;

impl OddFinder for OddFinderOne {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}

struct OddFinderTwo;

impl OddFinder for OddFinderTwo {
    fn is_odd(&self, _n: u64) -> bool {
        unimplemented!()
    }
}
```

I expected this to perform worse than the generic implementation, since trait objects should require dynamic dispatch (the program will have to look up the location of the correct `is_odd` implementation at run time), but it seems the compiler was able to see that `finder` is always `OddFinderOne` and perform some additional optimizations, because this calculated the billionth odd number in 1039 ms (average of 5 runs). This is not only better than the generic implementation but it is actually on par with the baseline, which was a very surprising outcome to me.

## Boxed Trait Object

The other way to use trait objects is by passing a `Box<dyn Trait>` (as opposed to `& dyn Trait` shown above).

```rust
use std::time::Instant;

fn main() {
    let finder: Box<dyn OddFinder> = Box::new(OddFinderOne);

    let start_time = Instant::now();
    let result = find_nth_odd(finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}


fn find_nth_odd(odd_finder: Box<dyn OddFinder>, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

trait OddFinder {
    fn is_odd(&self, n: u64) -> bool;
}

struct OddFinderOne;

impl OddFinder for OddFinderOne {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}

struct OddFinderTwo;

impl OddFinder for OddFinderTwo {
    fn is_odd(&self, _n: u64) -> bool {
        unimplemented!()
    }
}
```

This calculated the billionth odd number in 1314 ms.

## Runtime dynamism

To this point we haven't given the compiler any abstraction it couldn't see through at compile time. Even though we had two `OddFinder` implementations, we were always hard coding the program to use `OddFinderOne`. We'll change that here, to see what happens when the compiler cannot make this type of optimization.

```rust
use std::{env, time::Instant};

fn main() {
    // I don't intend to ever set this environment variable. The purpose
    // of this is only to ensure the compiler won't be able to know
    // which of the two OddFinder implementations we will use.
    let finder: Box<dyn OddFinder> = match env::var("USE_FINDER_TWO") {
        Ok(use_finder_two) if use_finder_two == String::from("True")
            => Box::new(OddFinderTwo),
        _ => Box::new(OddFinderOne),
    };

    let start_time = Instant::now();
    let result = find_nth_odd(finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}


fn find_nth_odd(odd_finder: Box<dyn OddFinder>, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

trait OddFinder {
    fn is_odd(&self, n: u64) -> bool;
}

struct OddFinderOne;

impl OddFinder for OddFinderOne {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}

struct OddFinderTwo;

impl OddFinder for OddFinderTwo {
    fn is_odd(&self, _n: u64) -> bool {
        unimplemented!()
    }
}
```

As expected this version is much slower, as we can be sure dynamic dispatch is required here to determine which `is_odd` function to call, based on whichever `OddFinder` implementation is being used. This version found the billionth prime in 3424 ms.

## Enum

Rather than taking in a trait object, we can modify our `find_nth_odd` function to take an enum containing any possible `OddFinder` implementation.

```rust
use std::{env, time::Instant};

fn main() {
    // I don't intend to ever set this environment variable. The purpose
    // of this is only to ensure the compiler won't be able to know
    // which of the two OddFinder implementations we will use.
    let finder: OddFinders = match env::var("USE_FINDER_TWO") {
        Ok(use_finder_two) if use_finder_two == String::from("True")
            => OddFinders::Two(OddFinderTwo),
        _ => OddFinders::One(OddFinderOne),
    };

    let start_time = Instant::now();
    let result = find_nth_odd(finder, 1_000_000_000);
    let elapsed_time = start_time.elapsed().as_millis();

    println!("{} in {} ms", result, elapsed_time);
}


fn find_nth_odd(odd_finder: OddFinders, n: u64) -> u64 {
    let mut i = 0;
    let mut odd_count = 0;

    while odd_count != n {
        i += 1;
        if odd_finder.is_odd(i) {
            odd_count += 1;
        }
    }

    i
}

trait OddFinder {
    fn is_odd(&self, n: u64) -> bool;
}

struct OddFinderOne;

impl OddFinder for OddFinderOne {
    fn is_odd(&self, n: u64) -> bool {
        n % 2 == 1
    }
}

struct OddFinderTwo;

impl OddFinder for OddFinderTwo {
    fn is_odd(&self, _n: u64) -> bool {
        unimplemented!()
    }
}

enum OddFinders {
    One(OddFinderOne),
    Two(OddFinderTwo),
}

impl OddFinder for OddFinders {
    fn is_odd(&self, n: u64) -> bool {
        match self {
            OddFinders::One(finder) => finder.is_odd(n),
            OddFinders::Two(finder) => finder.is_odd(n),
        }
    }
}
```

Relative to the boxed trait object, here we are trading dynamic dispatch for a runtime enum match statement. This turns out to be a big performance win, allowing this version to find the billionth prime in 1366 ms.

## Inline never

For fun, I re-ran the baseline with `#[inline(never)]` on the `is_odd` function. This increased the time required to calculate the billionth odd number to 3378 ms.

## Conclusion

Making performance predications in the presence of an optimizing compiler is difficult. Extrapolating performance results for one workload to predict the performance against a different workload is also challenging.

Those disclaimers aside I think there are some interesting findings here. The first is that moving a method behind a trait increases the runtime cost, even when called directly on the struct (as opposed to through a trait object). The second is that, at least in this benchmark, creating an enum of all possible trait implementors and passing that into a function is significantly (roughly 3x here) faster than using a boxed trait object. 

All code is available [here](https://github.com/JoshMcguigan/cost-of-indirection). If you discover issues with any of these benchmarks feel free to open issues/pull requests on GitHub.

#### Update #1

Reddit user boscop [suggested I repeat the trait object test using `std::hint::blackbox`](https://www.reddit.com/r/rust/comments/fbev13/the_cost_of_indirection/fj42x2n?utm_source=share&utm_medium=web2x). This has an effect which is similar to the environment variable check in the boxed trait test, and with the black box the `& dyn OddFinder` performs similarly to the `Box<dyn OddFinder>`, finding the billionth prime in 3243 ms.

#### Update #2

Reddit user Shnatsel [suggested I submit a bug report against the Rust compiler](https://www.reddit.com/r/rust/comments/fbev13/the_cost_of_indirection/fj43llt?utm_source=share&utm_medium=web2x), based on the result of the benchmark that [moved the `is_odd` method onto a trait, while `find_nth_odd` still took a concrete struct as an arg](https://github.com/JoshMcguigan/cost-of-indirection/blob/master/src/bin/trait.rs). I've created that bug report in the form of [an issue with the bug tag on the main rust-lang repo](https://github.com/rust-lang/rust/issues/69593).
