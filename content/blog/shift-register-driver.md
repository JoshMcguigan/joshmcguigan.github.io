---
title: Shift Register Driver
date: "2018-07-04"
---

I recently finished reading the [Discovery book](https://rust-embedded.github.io/discovery/) as an introduction to embedded programming in Rust. The book assumes minimal previous experience so it is a great way to learn embedded programming, as well as expand your Rust skills (I'd recommend at least having completed the [Rust book](https://doc.rust-lang.org/book/second-edition/index.html) before starting).

This [blog post](http://blog.japaric.io/brave-new-io) roughly outlines the current state of the embedded Rust eco-system, and in it Jorge (author of the Discovery book) challenges the community with the [weekly driver initiative](https://github.com/rust-lang-nursery/embedded-wg/issues/39). Creating a driver sounds difficult, but it turns out that for many device types, it is well within reach after completing the Discovery book. The [shift register driver](https://github.com/JoshMcguigan/shift-register-driver) is my first contribution to the initiative.

## What's a shift register?

Shift registers are used as GPIO expanders. Serial-in parallel-out shift registers, which are the type initially supported by this driver, consume three output pins from your controller and in return provide eight (or more, theoretically unlimited) outputs. They do this by converting a series of pulses on the three output pins into a particular state of the shift register output pins. 

## Hardware abstraction layer

The shift register driver is built on top of traits defined in the [embedded-hal](https://github.com/japaric/embedded-hal) crate. This crate defines a set of high-level interfaces to common embedded systems peripherals. For example, it defines a trait `OutputPin` which has only two methods: `set_high()` and `set_low()`. Creating a driver on top of the embedded-hal traits means that any device which has a [hal implementation crate](https://github.com/rust-embedded/awesome-embedded-rust#hal-implementation-crates) can use your driver. 

## Using the driver

The serial-in parallel-out shift register struct is in a module called `sipo` with the expectation that there will eventually be support within this driver for parallel-in serial-out shift registers (which would go in a `piso` module). Constructing a new `ShiftRegister` consumes three output pins (any struct which impl's the embedded-hal trait `OutputPin`). 

From there, you can call `shift_register.decompose()` which returns an array of eight `ShiftRegisterPin` structs, each of which associated with an output on the shift register. The `ShiftRegisterPin` struct impl's the embedded-hal trait `OutputPin` as well, so you can control these pins with `pin.set_high()`, `pin.set_low()`, or pass these pins along to any other driver which expects an embedded-hal `OutputPin`. 

Finally, if you are done with the shift register outputs and you'd like to regain ownership of the original three GPIO pins used to construct the shift register, you can call `shift_register.release()`.

```rust  
use shift_register_driver::sipo::ShiftRegister; 
let shift_register = ShiftRegister::new(clock, latch, data); 
{ 
	let mut outputs = shift_register.decompose();  
	for i in 0..8 {
		outputs[i].set_high(); 
		delay.delay_ms(300u32); 
	}  
	for i in 0..8 { 
		outputs[7-i].set_low(); 
		delay.delay_ms(300u32); 
	}  
} 
// shift_register.release() can optionally be used when the shift register is no 
//      longer needed in order to regain ownership of the original GPIO pins 
let (clock, latch, data) = shift_register.release();
```

## Getting involved

Writing a driver was a great learning experience. If you are interested in embedded Rust, I'd highly recommend that you purchase a `STM32F3DISCOVERY` discovery board and work through the Discovery book. After that, review the code of some of the existing [driver crates](https://github.com/rust-embedded/awesome-embedded-rust#driver-crates), then find a device which doesn't already have a driver and get started!

