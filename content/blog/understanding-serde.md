---
title: Understanding Serde
date: "2019-11-17"
---

Serde is one of the most popular Rust crates, and deservedly so. If you aren't familiar, Serde describes itself as "a framework for serializing and deserializing Rust data structures efficiently and generically." What is most impressive to me is how robust the Serde data model has proven to be, allowing it to support human readable protocols like JSON and YAML, but also binary formats like Bincode. Its really a bonus that Serde does this while remaining exceptionally performant.

This blog posts dives into how Serde (along with the ecosystem of Serde data formats) is able to pull this off. To limit the scope of this post I am going to focus on Serde serialization to JSON, and skip any discussion of deserialization. If you are interested in deserialization (or a different data format) I believe you will be able to perform a similar analysis yourself after reading this post.

## The Serde Data Model

One of the things I like to do when I am first trying to reason about a new library is to think about how I might go about implementing it. Sometimes the method I think up is reasonably close, and other times I miss the mark fundamentally. This was a case of the latter, but I think it is educational to present anyway.

After reading about the [Serde data model](https://serde.rs/data-model.html), which is described as "the API by which data structures and data formats interact", I was developing roughly the following mental model of how Serde might work.

```
Rust structure 
  ↓
  -- Serialize --> Structure in terms of the Serde data model
  ↓
  -- Data format (JSON/Bincode/etc) --> Convert the Serde data model to the output format
```

I've included some real Serde example code below, to set some context before diving deeper into how I thought this might be implemented.

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point { x: 1, y: 2 };

    // Convert the Point to a JSON string.
    let serialized = serde_json::to_string(&point).unwrap();

    // Prints serialized = {"x":1,"y":2}
    println!("serialized = {}", serialized);
}
```

Mapping my mental model to this example, I expected `#[derive(Serialize)]` would output some code like:

```rust
impl Serialize for Point {
	fn serialize(&self) -> SerdeDataModel {
		...
	}
}
```

Then I expected `serde_json::to_string` to look roughly like:

```rust
fn to_string<T>(input: T) -> String
	where T: Serialize
{
	let serde_data_model = input.serialize();

	let mut output = String::new();

	// code which traverses the Serde data model
	// representation and builds the JSON
	for elem in serde_data_model {
		match elem {
			struct(content) => //.. serialize the struct into JSON
			_ => // handle all other types in the Serde data model	
		}
	}

	output
}
```

I was starting to feel comfortable with this idea, so I dove into the source to see how close I was. I wanted to start by finding the definition of the Serde data model, which I expected would be a large enum. As you can probably guess, I was not able to find that enum because it doesn't actually exist.

## Into the Code

Unable to confirm my suspicions about how Serde might work, I did try to peek through the code a bit to see if things would start making sense. But the Serde code base makes heavy use of generics (for good reason) and jumps rapidly between the Serde crate, the Serde data format crate, and code generated by the Serde derive macros, so I had a hard time making sense of it. At that time I moved to my second technique for understanding library code: pick an entry point into the library that I am familiar with as a user, and trace a code path through the library starting at that entry point.

Sticking with the example above, lets start with `serde_json::to_string`.

```rust
// https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs
// crate: serde_json

pub fn to_writer<W, T: ?Sized>(writer: W, value: &T) -> Result<()>
where
    W: io::Write,
    T: Serialize,
{
    let mut ser = Serializer::new(writer);
    try!(value.serialize(&mut ser));
    Ok(())
}

pub fn to_vec<T: ?Sized>(value: &T) -> Result<Vec<u8>>
where
    T: Serialize,
{
    let mut writer = Vec::with_capacity(128);
    try!(to_writer(&mut writer, value));
    Ok(writer)
}

pub fn to_string<T: ?Sized>(value: &T) -> Result<String>
where
    T: Serialize,
{
    let vec = try!(to_vec(value));
    let string = unsafe {
        // We do not emit invalid UTF-8.
        String::from_utf8_unchecked(vec)
    };
    Ok(string)
}
```

`serde_json` provides a number of entry points depending on exactly how you plan to use the resulting JSON. In our case we wanted to trace the `to_string` path, but we can quickly see that it just dispatches to `to_vec`, which itself dispatches to `to_writer`, which is where the first interesting work happens.

A `Serializer` is created, which takes ownership of an `io::Write` (which is really an `&mut Vec<u8>` in our case. Then a mutable reference to that `Serializer` is passed to the `serialize` method on our `Point` struct with `value.serialize(&mut ser)`.

The `serialize` method is part of the `Serialize` trait. The trait definition is in [the Serde crate](https://github.com/serde-rs/serde/blob/4eb580790dd6c96089b92942a5f481b21df4feaf/serde/src/ser/mod.rs#L246), but right now I'm interested in the trait implementation for our `Point` struct, which is generated because of the `#[derive(Serialize)]` attribute. Using [cargo-expand](https://github.com/dtolnay/cargo-expand) allows you to see the output of the derive macro.

```rust
// crate: Sample application
// Code generated by the #[derive(Serialize)] macro

use serde::{Serialize, Serializer, ser::SerializeStruct};

impl serde::Serialize for Point {
	fn serialize<S>(&self, serializer: S) -> serde::export::Result<S::Ok, S::Error>
	where
		S: Serializer,
	{
		let mut serde_state = match Serializer::serialize_struct(
			serializer,
			"Point",
			false as usize + 1 + 1,
		) {
			serde::export::Ok(val) => val,
			serde::export::Err(err) => {
				return serde::export::Err(err);
			}
		};
		match SerializeStruct::serialize_field(&mut serde_state, "x", &self.x) {
			serde::export::Ok(val) => val,
			serde::export::Err(err) => {
				return serde::export::Err(err);
			}
		};
		match SerializeStruct::serialize_field(&mut serde_state, "y", &self.y) {
			serde::export::Ok(val) => val,
			serde::export::Err(err) => {
				return serde::export::Err(err);
			}
		};
		SerializeStruct::end(serde_state)
	}
}
```

Before diving into this code, I want to note that I did modify it a small amount to improve readability. Serde uses several tricks to ensure that the code it generates works in all environments. While those tricks are interesting, they are not the focus of today's investigation.

There is one trick I did leave in place however, and that is the way Serde calls trait methods. You can see this in the very first line of the method where `Serializer::serialize_struct` is called and `serializer` is passed in, as oppposed to the more common `serializer.serialize_struct`. This disambiguates the `Serializer::serialize_struct` method from any other `serialize_struct` method which may exist, and I left it in place because changing it felt like it moved the demo code too far away from the actual code.

Getting back to our analysis now, we were tracing the call in `serde_json` to `&point.serialize(&mut serializer)` where `serializer` is a `serde_json` specific implementation of the `Serializer` trait. The first thing that happens in this function is it calls the `serialize_struct` method on the serializer, passing it some information about this struct (the name and the number of fields in the struct). If you are familiar with other programming languages, you may recognize this information as things you could get from a type at runtime via reflection. The `#[derive(Serialize)]` macro exists basically as a high performance work around to the fact that this type information isn't available at runtime in Rust.

```rust
// https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L427
// crate: serde-json
impl serde::Serialize for Serializer {
	// ..many methods omitted

    fn serialize_struct(self, name: &'static str, len: usize) -> Result<Self::SerializeStruct> {
        match name {
            _ => self.serialize_map(Some(len)),
        }
    }
}
```

As you are likely aware, JSON does not have any way to serialize a named struct, so the `serialize_struct` method on the `serde_json` `Serializer` simply dispatches to `self.serialize_map`.

```rust
// https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L394
// crate: serde-json
impl serde::Serialize for Serializer {
	// ..many methods omitted

    fn serialize_map(self, len: Option<usize>) -> Result<Self::SerializeMap> {
        if len == Some(0) {
			// .. omitted code to build an empty JSON object '{}'
        } else {
            try!(self
                .formatter
                .begin_object(&mut self.writer)
                .map_err(Error::io));
            Ok(Compound::Map {
                ser: self,
                state: State::First,
            })
        }
    }
}

// https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L1852
trait Formatter {
	// ..many methods omitted

    fn begin_object<W: ?Sized>(&mut self, writer: &mut W) -> io::Result<()>
    where
        W: io::Write,
    {
        writer.write_all(b"{")
    }
}
```

A keen eye may have noticed that when we called `serialize_map` we passed in the number of fields in the struct. This is a bit odd since JSON doesn't need this information for serialization, and indeed we can see now that unless the length is zero the length information is ignored.

We are now ready to serialize our first byte. `self.formatter.begin_object` takes a mutable reference to our `Vec<u8>` and writes a single character, the open curly brace, which represents the start of a JSON map.

The `serialize_map` method finishes by creating a `Compound::Map` which stores the serializer itself as well as a state enum with the value `State::First`. The important thing is that this return type implements the [serde::ser::SerializeStruct trait](https://github.com/serde-rs/serde/blob/4eb580790dd6c96089b92942a5f481b21df4feaf/serde/src/ser/mod.rs#L1865).

```rust
// crate: Sample application
// Code generated by the #[derive(Serialize)] macro
// Repeated from above for clarity

use serde::{Serialize, Serializer, ser::SerializeStruct};

impl serde::Serialize for Point {
	fn serialize<S>(&self, serializer: S) -> serde::export::Result<S::Ok, S::Error>
	where
		S: Serializer,
	{
		let mut serde_state = match Serializer::serialize_struct(
			serializer,
			"Point",
			false as usize + 1 + 1,
		) {
			serde::export::Ok(val) => val,
			serde::export::Err(err) => {
				return serde::export::Err(err);
			}
		};
		match SerializeStruct::serialize_field(&mut serde_state, "x", &self.x) {
			serde::export::Ok(val) => val,
			serde::export::Err(err) => {
				return serde::export::Err(err);
			}
		};
		match SerializeStruct::serialize_field(&mut serde_state, "y", &self.y) {
			serde::export::Ok(val) => val,
			serde::export::Err(err) => {
				return serde::export::Err(err);
			}
		};
		SerializeStruct::end(serde_state)
	}
}
```

Popping off the stack now we are back to our `serde::Serialize` impl for `Point`, which I've repeated here for clarity. We now know `serde_state` is a `Compound::Map` from `serde-json`. Up next are two calls to `serialize_field` and then a call to `end`.

```rust
// https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L755
// crate: serde-json

impl<'a, W, F> ser::SerializeStruct for Compound<'a, W, F>
where
    W: io::Write,
    F: Formatter,
{
    type Ok = ();
    type Error = Error;

    fn serialize_field<T: ?Sized>(&mut self, key: &'static str, value: &T) -> Result<()>
    where
        T: Serialize,
    {
        match *self {
            Compound::Map { .. } => {
                try!(ser::SerializeMap::serialize_key(self, key));
                ser::SerializeMap::serialize_value(self, value)
            }
			// .. omitted other enum options
        }
    }

    fn end(self) -> Result<()> {
        match self {
            Compound::Map { .. } => ser::SerializeMap::end(self),
			// .. omitted other enum options
        }
    }
}
```

As with the `Serializer`, the `SerializeStruct` methods do nothing more than dispatch to the `SerializeMap` implementations.

```rust
// https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L673
// crate: serde-json

impl<'a, W, F> ser::SerializeMap for Compound<'a, W, F>
where
    W: io::Write,
    F: Formatter,
{
    type Ok = ();
    type Error = Error;

    fn serialize_key<T: ?Sized>(&mut self, key: &T) -> Result<()>
    where
        T: Serialize,
    {
        match *self {
            Compound::Map {
                ref mut ser,
                ref mut state,
            } => {
                try!(ser
                    .formatter
                    .begin_object_key(&mut ser.writer, *state == State::First)
                    .map_err(Error::io));
                *state = State::Rest;

                try!(key.serialize(MapKeySerializer { ser: *ser }));

                try!(ser
                    .formatter
                    .end_object_key(&mut ser.writer)
                    .map_err(Error::io));
                Ok(())
            }
			// .. omitted other enum options
		}
    }

    fn serialize_value<T: ?Sized>(&mut self, value: &T) -> Result<()>
    where
        T: Serialize,
    {
        match *self {
            Compound::Map { ref mut ser, .. } => {
                try!(ser
                    .formatter
                    .begin_object_value(&mut ser.writer)
                    .map_err(Error::io));
                try!(value.serialize(&mut **ser));
                try!(ser
                    .formatter
                    .end_object_value(&mut ser.writer)
                    .map_err(Error::io));
                Ok(())
            }
			// .. omitted other enum options
		}
    }

    fn end(self) -> Result<()> {
        match self {
            Compound::Map { ser, state } => {
                match state {
                    State::Empty => {}
                    _ => try!(ser.formatter.end_object(&mut ser.writer).map_err(Error::io)),
                }
                Ok(())
            }
			// .. omitted other enum options
		}
    }
}
```

That is a big code block, so even though we will jump back to it a few times I am not going to duplicate it. Instead, from here on out, rather than inlining the code I'm just going to link to it. I encourage you to follow along by clicking through the links and reviewing the code nonetheless.

We enter here though the `serialize_key` method. The first method call of interest is to [begin\_object\_key](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L1871) on our formatter. Interestingly, this method uses the state enum we saw earlier to determine whether it should write a "," to our `Vec<u8>` (you don't need a comma before the first field).

Next we call `key.serialize` and pass a `MapKeySerializer`, which [implements Serializer](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L869). In all cases `key` is a `&'static str` (you can see this in the `impl SerializeStruct for Compound` block, but intuitively it is because the struct field names are known at compile time). `key.serialize` immediately calls back to our `MapKeySerializer` with `serializer.serialize_str` [as shown in the impl Serialize for str block](https://github.com/serde-rs/serde/blob/4eb580790dd6c96089b92942a5f481b21df4feaf/serde/src/ser/impls.rs#L43) which [dispatches](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L879) back to our [root serializer's serialize_str method](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L223) which itself calls [format\_escaped\_str](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L2065) to write the actual bytes, `"x"`, to our `Vec<u8>`.  

The `serialize_key` method ends with a call to our formatter's [end\_object\_key method](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L1886), which does nothing. If you are curious, it is the [begin\_object\_value method](https://github.com/serde-rs/json/blob/10132f800fd1223ac698fa8c41b201dca152c413/src/ser.rs#L1897), called at the start of `serialize_value` which writes the colon that is required between the key and the value in JSON maps.

At this point things start getting a bit repetitive. The `serialize_value` method works nearly identically to the `serialize_key` method. Then both methods are repeated for the `y` field on our `Point`, then we ask the formatter to print the closing curly brace.

## Conclusion

I realize that this post explains more of the 'how' than the 'why', and that may not be satisfying to some readers. While I can trace through the program mechanically, I am only starting to become comfortable with it on a conceptual level. Certainly I need to mull things over a bit more before I could claim to fully understand why things are the way they are.

But what about my original guess for the implementation? One thing I have taken away from this is that Serde is very focused on performance. My original approach would have involved allocating an intermediate struct, which is likely a deal breaker when compared to the performance of the actual implementation.

The thing I missed originally was that the Serde data model doesn't come in the form of a struct or enum, but rather in the form of functions which are implemented by each data format as the Serializer trait. The derive macro generates an implementation of the Serialize (not Serializer) trait, which drives the serializer by calling the appropriate methods on the serializer based on the type of Rust data structure being serialized. Beyond that, its all implementation details.