---
title: TSL256X Light Intensity Sensor Driver
date: "2018-07-15"
---

It is an exciting time to be working in embedded Rust. After writing [my first driver](/blog/shift-register-driver/), I mostly had the feeling that for driver writers there was a clear expectation and an obvious standard for how things should be done. My experience writing this driver, which uses I2C rather than GPIO, exposed me to a few topics of active discussion within the embedded Rust working group which I hadn't seen before.

## Measuring Light Intensity

Lux is a measure of light intensity normalized based on the perception of the human eye. The TSL256X has two sensors, one measuring a broad spectrum including light from both the visible and IR range, and the other measuring just the IR range. The datasheet includes empirically derived formulas for converting the raw output of these two sensors to a single Lux measurement. 

The TSL256X driver, in its current state, includes methods to get the raw sensor data for each on-board sensor but it does not perform the Lux calculation. This is primarily due to the limited floating point math operations available in the Rust core library. For many use cases, it may be enough to directly use the raw sensor values, as shown below:

```rust
extern crate tsl256x;
use tsl256x::{Tsl2561, SlaveAddr};

let addr = SlaveAddr::default().addr();
let sensor = Tsl2561::new(&mut i2c, addr).unwrap();

sensor.power_on(&mut i2c); 

// Note sensor readings are zero until one integration
//      period (default 400ms) after power on
iprintln!(&mut cp.ITM.stim[0], "IR+Visible: {}, IR Only: {}",
                    sensor.visible_and_ir_raw(&mut i2c).unwrap(),
                    sensor.ir_raw(&mut i2c).unwrap());
```

The driver is available on [crates.io](https://crates.io/crates/tsl256x) and [GitHub](https://github.com/JoshMcguigan/tsl256x).

## The Challenges

The challenges here are primarily non-technical, instead centering around how the driver fits into the overall embedded Rust ecosystem. Ultimately I think there would be benefit in documenting a set of standards for embedded-hal drivers to adhere to, which could answer questions like these.  

#### Crate Naming

> There are only two hard things in Computer Science: cache invalidation and naming things.

The TSL256X family includes two devices, the TSL2560 and TSL2561, which differ only in the communication protocol they support. This brings up questions about how the driver crate(s) should be named. Looking for existing discussions of this revealed [this comment](https://github.com/rust-lang-nursery/embedded-wg/issues/39#issuecomment-370545982) with a few follow up replies, and some discussion in [this issue](https://github.com/rust-lang-nursery/embedded-wg/issues/27), but I don't think a broadly applicable consensus has been reached on this issue.   

While the initial release of this driver crate only supports the TSL2561 (the I2C version), I chose the crate name `tsl256x` based on the general feeling I get that the community is leaning toward single crates with generic names supporting multiple devices. However, as I think about this more, I think I may have made a mistake here, given that drivers are basically an abstraction over the communications requirements of the device and the difference between the two devices in the TSL256X family is the communications protocol supported. Any feedback or guidance here would be much appreciated. 

#### Device Addressing

I2C devices generally support either a single slave address or choosing one of a few addresses. Existing driver crates, from what I have seen, typically hardcode these addresses (actually in a non-exhaustive review of the I2C drivers on [awesome embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust#driver-crates) I could only find examples of devices/drivers which support a single address). The TSL256X family supports choosing one of three addresses, which I certainly wanted to support with the driver. However, there are also device address translators such as the LTC4316, which effectively means that in the general case any device may be accessible at any address, and I also wanted to support this use case. 

For this reason, the `new` method takes the device address as a parameter:

```rust
pub fn new(_i2c: &I2C, address: u8) -> Result<Self, E>
```

But just taking a u8 as the address would not be too friendly to the typical user, who would use one of the three device addresses directly supported by the device. For that, the driver crate also includes an enumeration of those addresses, with a `Default` impl.

```rust
#[allow(dead_code)]
#[allow(non_camel_case_types)]
#[derive(Copy, Clone)]
/// Possible slave addresses
pub enum SlaveAddr {
    /// Default slave address
    ADDR_0x39 = 0x39,
    /// Optional slave address
    ADDR_0x29 = 0x29,
    /// Optional slave address
    ADDR_0x49 = 0x49,
}

impl Default for SlaveAddr {
    fn default() -> Self {
        SlaveAddr::ADDR_0x39
    }
}

impl SlaveAddr {
    /// Get slave address as u8
    pub fn addr(&self) -> u8 {
        *self as u8
    }
}
```

which allows for the following constructor pattern when using the default address:

```rust
Tsl2561::new(&mut i2c, SlaveAddr::default().addr())
```

This works well, but I do wonder if there is an even better solution. I believe it would be possible to build, as a standalone crate, an I2C device address translator struct which consumes an embedded-hal `I2C` type and itself impl's the `I2C` traits, but translates the address of all traffic through it based on a user-configurable address map.

#### To Init or Not to Init?

The TSL256X, like most peripherals, requires some initialization after connecting it to power before it can be used. Most driver crates I've seen do the initialization as part of the `new` method, which is user friendly for most use cases, but what if we are connecting to a device which we know is already initialized, or we want to modify the configuration before initialization, or we don't want to immediately power on the device in order to save energy? 

In this crate, I've chosen to skip initialization in the `new` method, and require the user to explicitly `power_on` the device. Generally, I think either the `new` method should not perform any initialization of the actual device, or there should be a standard method like `new_skip_init` to provide the option of creating a new driver instance without running the initialization code. 

#### I2C Bus Sharing

At the moment, the prevailing standard within the embedded-hal drivers is to consume the I2C bus when creating a new device instance, like this:

```rust
pub fn new(i2c: I2C) -> Result<Self, E>
```

The trouble is, that doesn't allow for multiple devices to share the I2C bus. There is discussion of this exact issue [on GitHub here](https://github.com/japaric/embedded-hal/issues/35). I don't feel like I have a broad enough understanding in this area, particularly around the needs of async, to push for a particular solution here. That said, I still wanted this crate to support sharing the I2C bus. For that reason, the `new` method as well as all other methods on the driver struct borrow (rather than take ownership over) the I2C bus. This solves the bus sharing problem in the short term, and easily refactors to any of the other proposed solutions in the longer term.

## Future Work

Even with the challenges listed above, writing a driver is a great way to get involved in embedded Rust. If there were a driver development document to answer the questions above with the standard community solution, it would make the barrier of entry for new developers even lower, and ensure consistency across drivers. I would be happy to be involved in writing this document, although I think the most important thing is to ensure that the standards reflect the broader community opinion on these issues and they've been properly vetted by both driver users and driver developers.  

I know that not all of the decisions I have made in developing the TSL256X driver would align with the official standard. In those areas, I would rewrite/refactor/update the driver to be compliant. 

## Getting Involved

Writing drivers for embedded Rust has been a great learning experience, and I still think that writing a driver is a great way to get involved as a newcomer to embedded Rust programming, or even embedded programming in general. If you're interested, I'd start by reading [this introductory post on the weekly driver initiative](https://github.com/rust-lang-nursery/embedded-wg/issues/39), as well as reviewing some of the [existing drivers](https://github.com/rust-embedded/awesome-embedded-rust#driver-crates). From there, you can pick a device which doesn't already have a driver and jump in.
