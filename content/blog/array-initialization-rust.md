---
title: Methods for Array Initialization in Rust
date: "2018-12-22"
---

Arrays in Rust are fixed size, and Rust requires that every element in an array is initialized to a valid value when the array is initialized. The result of these requirements is array initialization in Rust is a much deeper topic than it would seem.

## Array Literals

```rust
let _: [u8; 3] = [1, 2, 3];
let _: [&str; 3] = ["1", "2", "3"];

let _: [String; 3] = [
    String::from("1"),
    String::from("2"),
    String::from("3")
];

let mut rng = rand::thread_rng();
let _: [u8; 3] = [rng.gen(), rng.gen(), rng.gen()];
``` 

The simplest method of creating an array is to explicitly set a value for each element. If this fits your use case, you should use it. Beyond that, there's not much to add here so let's move on. 

## [T; N] where T: Copy

```rust
let _: [u8; 3] = [0; 3];
let _: [Option<u8>; 3] = [None; 3];
let _: [u8; 100000] = [0; 100000]; // this works for any array size
```

If you need an array initialized with the same value repeated for each element, and the type of value contained in the array implements the Copy trait, Rust supports a shorthand syntax shown above. 

```rust
let _: [u8; 3] = [rng.gen(); 3];
```

One important thing to note from this example is that the copy happens after the `rng.gen()` method is evaluated. This means that all three elements in this array contain the same value.

## [T; N] where T: Default (and N <= 32)

```rust
let x: [Option<String>; 3] = Default::default();
assert_eq!(
    [None, None, None],
    x
);
```

A very common use case is initializing an array with `None`. While this can be done using `[None; N]` for `Option<T>` where `T` implements the copy trait, if `T` does not implement copy you can fall back to using the default trait as shown above. 

The primary downside to this method is it only works for arrays up to size 32. 

## Going unsafe

```rust
let _: [Option<String>; 33] = unsafe {
    let mut arr: [Option<String>; 33] = std::mem::uninitialized();
    for item in &mut arr[..] {
        std::ptr::write(item, None);
    }
    arr
};
```

The [common approach](https://doc.rust-lang.org/std/mem/fn.uninitialized.html) for initializing arrays larger than 32 elements, or containing types which do not implement the default trait, is shown above. A simple demonstration of how this could be unsafe follows:

```rust
fn builder(rng: &mut rand::prelude::ThreadRng) -> Option<String> {
    // do anything here, potentially even panic
    if rng.gen::<u8>() > 254u8 {
        panic!("Builder failed");
    }
    None
}
let mut rng = rand::thread_rng();
let _: [Option<String>; 33] = unsafe {
    let mut arr: [Option<String>; 33] = std::mem::uninitialized();
    for item in arr.iter_mut() {
        std::ptr::write(item, builder(&mut rng));
    }
    arr
};
```

If the builder method panics partway though initializing the array, the array will be dropped, triggering a drop of each element of the array. But some of the array elements will not yet be initialized, and calling drop on uninitialized memory is undefined behavior. 

Beyond that issue there are further problems with `mem::uninitialized`, which have led to it being deprecated in favor of `mem::MaybeUninit::uninitialized`, although there isn't yet consensus on [how to safely initialize an array with MaybeUninit](https://github.com/rust-lang/rust/issues/54542). 

### Update for Rust 1.36

MaybeUninit was stabilized in Rust 1.36, and `mem::uninitialized` will be deprecated as of Rust 1.38. The [MaybeUninit docs](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html) include an example of array initialization, which I'll provide an abbreviated copy of below.

```rust
let data = {
    let mut data: [std::mem::MaybeUninit<Vec<u32>>; 1000] = unsafe {
        std::mem::MaybeUninit::uninit().assume_init()
    };

    for elem in &mut data[..] {
        unsafe { std::ptr::write(elem.as_mut_ptr(), vec![42]); }
    }

    unsafe { std::mem::transmute::<_, [Vec<u32>; 1000]>(data) }
};

assert_eq!(&data[0], &[42]);
```

## Back to safety

```rust
let _: [Option<String>; 33] = [
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng), builder(&mut rng), builder(&mut rng), builder(&mut rng),
    builder(&mut rng)
];
```

It may not be fun to write, but the unsafe block in the section above can be replaced entirely with safe Rust. 

## Introducing arr!

```rust
use arr_macro::arr;
let _: [Option<String>; 33] = arr![builder(&mut rng); 33];
```

With a function-like procedural macro, we can have the best of both worlds (safety and concise code). I created a crate, [arr_macro](https://github.com/JoshMcguigan/arr_macro), which does exactly this using 100% safe Rust.

```rust
enum MyEnum {
    A,
    B
}
let _: [MyEnum; 33] = arr![MyEnum::A; 33];
```

It works with all enum types (implementing copy/default is not required).

```rust
#[derive(Debug)]
struct MyStruct {
    member: u16,
}
impl MyStruct {
    fn new(member: u16) -> Self {
        MyStruct { member }
    }
}
let mut i = 0u16;
let x: [MyStruct; 33] = arr![MyStruct::new({i += 1; i - 1}); 33];

assert_eq!(0, x[0].member);
assert_eq!(1, x[1].member);
assert_eq!(2, x[2].member);
```

You can even use it with a counter to initialize each array element to a different value.

#### Nightly Required

~~`arr_macro` requires nightly Rust in order to enable the `proc_macro_hygiene` feature.~~ 

Update - Thanks to [this PR](https://github.com/JoshMcguigan/arr_macro/pull/1) by [David Tolnay](https://github.com/dtolnay), `arr_macro` now works on stable! If your procedural macro crate requires nightly due to `proc_macro_hygiene`, check out his crate [proc-macro-hack](https://github.com/dtolnay/proc-macro-hack).

## Prior Art

There are several other crates which solve this problem, each with their own trade-offs. 

#### array-init

```rust
use array_init::array_init;
let _: [MyStruct; 512] = array_init(|i| MyStruct::new(i as u16));
```
 * https://github.com/Manishearth/array-init
 * uses unsafe Rust internally
 * works for arrays up to size 512
 * Nice syntax, but further from the normal [T; N]

#### init-with-rs

```rust
use init_with::InitWith;
let _: [MyStruct; 32] = <[MyStruct; 32]>::init_with_indices(|i| MyStruct::new(i as u16));
```
 * https://github.com/QuietMisdreavus/init-with-rs
 * 100% safe Rust internally
 * works for arrays up to size 32
 * Nice syntax, but further from the normal [T; N]

#### array-macro

```rust
use array_macro::array;
let mut i = 0u16;
let _: [MyStruct; 33] = array![MyStruct::new({i += 1; i - 1}); 33];
```
 * https://github.com/xfix/array-macro
 * uses unsafe Rust internally
 * works for any array size
 
#### General thoughts on the prior art

 * In the case of initializing an array with non-constant values, as demonstrated in the examples, using a syntax further from `[T; N]` may actually be considered a benefit rather than a drawback. This syntactical difference would make it easier to differentiate a copy initialization from an initialization which evaluates an initialization function several times.
 * Many high quality crates provide safe wrappers around unsafe code, but in this case there is [some debate](https://www.reddit.com/r/rust/comments/95vxdy/understanding_ub_with_stdmemuninitialized/) over whether any use of `std::mem::uninitialized` can be considered safe.
 * In practice, the array size limits of `array-init` and `init-with-rs` may not be a big deal for two reasons. First, arrays are stack allocated, so there is some practical limit to how large an array can be. Second, it is trivial to modify either crate to work with larger array sizes.

## Future Work

It is my hope that solutions to these problems will be pulled in to the Rust language / standard library as appropriate. There is an approved RFC for supporting [constants in array repeat expressions](https://github.com/rust-lang/rust/issues/49147). That solves the common use case of `[None; N]`, but it does not resolve the non-constant use case (as expressed above with the examples using `MyStruct`). Perhaps the [work being done on the MaybeUninit type](https://github.com/rust-lang/rust/issues/53491) could be extended to solve that use case?   

## Conclusion

That something as pedestrian as array initialization could lead us down a path of such complexity suggests to me that this is one of the remaining rough edges of the Rust language. Indeed there are at least two approved RFCs in progress to improve things in this area. I look forward to the new language/standard library features which will eventually obsolete all of the workarounds above.
