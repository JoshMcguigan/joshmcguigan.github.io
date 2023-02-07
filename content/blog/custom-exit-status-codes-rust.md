---
title: Custom Exit Status Codes with ? in main
date: "2019-02-09"
---

Rust 1.26 introduced the ability to return a `Result` from the `main` method, which was a great ergonomics improvement especially for small CLI applications. If your application returns an `Ok`, Rust reports a success exit status code to the operating system. Likewise if your application returns an `Err`, Rust reports an error exit status code. 

But what if you want to return a custom exit status error code for each possible error type in your application, to provide some additional feedback to your user? This leads into an exploration of the `Termination` and `Try` traits, and is the topic of this post. 

## Custom exit codes with std::process:exit

The Rust standard library includes `std::process::exit` which allows exiting the process with a custom error code. A simple example is listed below. In a larger program this type of match pattern would get repeated many times, and this is exactly why the `?` operator was introduced.

```rust
use std::fs::File;

fn main() {
    let file : File = match File::open("filename") {
        Ok(file) => file,
        Err(e) => {
            eprintln!("Error: {}", e);
            std::process::exit(9);
        }
    };

    // do something with file
}
```

## Introducing try (?)

To eliminate the boilerplate associated with the approach above, the `?` operator was introduced, and as of Rust 1.26 it is possible to use the `?` operator in `main`.

```rust
use std::fs::File;
use std::io;

fn main() -> Result<(), io::Error> {
    let file : File = File::open("filename")?;

    // do something with file

    Ok(())
}
```

This greatly simplifies error handling, but it comes at the cost of specifying a custom exit status code. Using this pattern, Rust will return the same exit status to the OS no matter what type of error occurred.  

## Termination trait

It actually isn't just `Result` that can be returned from `main`, but anything which implements the `std::process::Termination` trait. The `Termination` trait includes a single method: `fn report(self) -> i32`, which Rust uses to determine which exit status code to return to the OS. 

The standard library includes `impl<E: Debug> Termination for Result<(), E>` which is why we could return a `Result<(), io::Error>` from `main` in the example above. But since the only trait bound on the error type is `Debug` there is no way for Rust to know which particular exit status code you might want for a particular error. For this reason, no matter the error type Rust will return the same exit code. 

## Implementing Termination

It would be nice to have an additional `Termination` impl like `impl<E: Into<i32> + Debug> Termination for Result<(), E>`, which includes a trait bound on the error type to ensure we could convert each error into a custom exit status. However, due to the Rust orphan rules, it is not possible to add that implementation outside of the standard library, and due to the rules on overlapping trait implementations it couldn't even be added within the standard library. 

To get around these issues, we have to create our own type to implement `Termination`, as shown below. 

```rust
pub enum Exit<T> {
    Ok,
    Err(T)
}

impl<T: Into<i32> + Debug> Termination for Exit<T> {
    fn report(self) -> i32 {
        match self {
            Exit::Ok => 0,
            Exit::Err(err) => {
                eprintln!("Error: {:?}", err);
                err.into()
            },
        }
    }
}
```

The additional `Into<i32>` trait bound on the error type allows us to call `err.into()` in the report method to pass back a custom exit status code.

## A simple example

We can use the above implementation of `Exit` to return our own custom exit status codes as shown below.

```rust
#[derive(Debug)]
enum MyErr {
    OneLessThanZero,
    OneEqualsTwo,
}

impl From<MyErr> for i32 {
    fn from(err: MyErr) -> Self {
        match err {
            MyErr::OneLessThanZero => 2,
            MyErr::OneEqualsTwo => 3,
        }
    }
}

fn main() -> Exit<MyErr> {
    if 1 < 0 {
        return Exit::Err(MyErr::OneLessThanZero)
    } else if 1 == 2 {
        return Exit::Err(MyErr::OneEqualsTwo)
    }

    Exit::Ok
}
```

## Try (?) with Exit

The example above is quite contrived and really not any better than directly calling `std::process::exit`. The real benefit comes when using `?` with our new `Exit` type. In order to use the `?` operator in a function (including `main`) which returns `Exit`, we must implement the `Try` trait for `Exit`.

```rust
impl<T> Try for Exit<T> {
    type Ok = ();
    type Error = T;

    fn into_result(self) -> Result<Self::Ok, Self::Error> {
        match self {
            Exit::Ok => Ok(()),
            Exit::Err(err) => Err(err)
        }
    }

    fn from_error(err: Self::Error) -> Self {
        Exit::Err(err)
    }

    fn from_ok(_: Self::Ok) -> Self {
        Exit::Ok
    }
}
```

Now that `Exit` implements `Try`, we can demonstrate a slightly more realistic example, including using the `?` operator. Notice here that we also have the flexibility to specify a specific exit status code for each error type in the `impl From<MyErr> for i32` block.

```rust
#[derive(Debug)]
enum MyErr {
    MissingArg,
    ParseIntError,
}

impl From<MyErr> for i32 {
    fn from(err: MyErr) -> Self {
        match err {
            MyErr::MissingArg => 2,
            MyErr::ParseIntError => 3,
        }
    }
}

impl From<option::NoneError> for MyErr {
    fn from(_: option::NoneError) -> Self {
        MyErr::MissingArg
    }
}

impl From<num::ParseIntError> for MyErr {
    fn from(_: num::ParseIntError) -> Self {
        MyErr::ParseIntError
    }
}

fn main() -> Exit<MyErr> {
    let num_string= env::args().skip(1).next()?;
    let num : u32 = num_string.parse()?;

    println!("Hello, user #{}!", num);

    Exit::Ok
}
```

## Mapping the error for additional flexibility

A downside to the approach above is that `option::NoneError` and `num::ParseIntError` must map one-to-one with a variant of `MyErr`. This means that if I have two operations that return `num::ParseIntError` I can't have a specific `MyErr` variant for each of them. This can be resolved using `map_err` as shown below.

```rust
#[derive(Debug)]
enum MyErr {
    MissingArg,
    ParseErrorUserNum,
    ParseErrorGroupNum,
}

impl From<MyErr> for i32 {
    fn from(err: MyErr) -> Self {
        match err {
            MyErr::MissingArg => 2,
            MyErr::ParseErrorUserNum => 3,
            MyErr::ParseErrorGroupNum => 4,
        }
    }
}

impl From<option::NoneError> for MyErr {
    fn from(_: option::NoneError) -> Self {
        MyErr::MissingArg
    }
}

fn main() -> Exit<MyErr> {
    let user_num_string : String = env::args().skip(1).next()?;
    let group_num_string : String = env::args().skip(2).next()?;

    let user_num : u32 = user_num_string.parse()
        .map_err(|_| MyErr::ParseErrorUserNum)?;
    let group_num : u32 = group_num_string.parse()
        .map_err(|_| MyErr::ParseErrorGroupNum)?;

    println!("Hello, user #{} from group #{}!", user_num, group_num);

    Exit::Ok
}
```

Note that each call to `parse` gets its own custom error type, and thus its own exit status code.

## Introducing Exit

I have created a crate, [exit](https://github.com/JoshMcguigan/exit), which implements the `Exit` struct as demonstrated in this post. The crate is in its very early stages, and is really just intended as a proof of concept and sample implementation for the ideas discussed here, but I'd be interested in hearing your ideas (via the [issue tracker](https://github.com/JoshMcguigan/exit/issues)) about how this crate could be made more useful. 

#### Nightly Required 

For now, the `Exit` crate requires nightly Rust, for the following features: `try_trait`, `termination_trait_lib`.

## Conclusion

As of Rust 1.26, it is possible to use the `?` operation within `main`. Like many other features of Rust, this was accomplished through the trait system, specifically the `Try` and `Termination` traits. In the standard implementation, errors returned from `main` all return the same exit status code to the OS. The [Exit crate](https://github.com/JoshMcguigan/exit) demonstrates an implementation of the `Try` and `Termination` traits allowing custom exit status codes for each error type.

