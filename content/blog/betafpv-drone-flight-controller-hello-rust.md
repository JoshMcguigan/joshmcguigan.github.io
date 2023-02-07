---
title: BetaFPV F3 Drone Flight Controller - Hello Rust
date: "2018-07-11"
---

One of the most exciting areas of hobbyist embedded programming, in my opinion, is flight controllers for remote controlled aircraft. In the particular case of a multi-rotor drone, the flight controller is responsible for converting the `UP` command from the transmitter into specific outputs for each of the motors. Maintaining the stability of a drone involves carefully adjusting the output of each motor thousands of times per second based on feedback from on-board sensors. 

There are several great C-based open source drone flight controller firmware projects, but as far as I can see there are none written in Rust. The good news is that most drone flight controllers are based on STM32 MCUs, which Rust has strong support for. Robust flight controller firmware is quite complex, and there are a number of challenges to be solved before even getting the rotors spinning. The first of those challenges is building a Rust project for a particular flight controller board, and flashing the board with the compiled code. A single blinking LED is our goal for today.

## Why the BetaFPV F3?

There are many options when it comes to drone flight controller hardware, so why pick the BetaFPV F3? My main concern was choosing a board with a supported and relatively popular (in the embedded Rust community) MCU. Most flight controllers are based on some STM32 variant, and this controller is no exception. It is based on the STM32F303, which is supported by an existing [hal implementation crate](https://github.com/japaric/stm32f30x-hal). Beyond compatibility with Rust, my other goal was to find a platform that would allow for an easily re-creatable drone build. Unlike many manufacturers, BetaFPV sells a [complete drone kit](https://betafpv.com/products/beta75-bnf-tiny-whoop-quadcopter) based on this controller. This allows for another developer to quickly and easily recreate, test, and contribute to this work.

## Project Structure

The structure I setup to build a Cargo project for the flight controller is based heavily on the [STM32 Quickstart Guide](http://blog.japaric.io/quickstart/) by Jorge Aparicio, but there are some minor changes. Below I'll outline all of the required steps, starting right after `cargo new`.

1.   In the root of your project, create a new directory and file `.cargo/config`
```toml
[target.thumbv7em-none-eabihf]
runner = "arm-none-eabi-gdb"
rustflags = [
    "-C", "link-arg=-Tlink.x",
    "-C", "link-arg=-nostartfiles"
]
[build]
target = "thumbv7em-none-eabihf"
```
1. Also in the root of your project, create a new file `memory.x`
```
    MEMORY
    {
        /* NOTE K = KiBi = 1024 bytes */
        FLASH : ORIGIN = 0x08000000, LENGTH = 256K
        RAM : ORIGIN = 0x20000000, LENGTH = 40K
    }
    _stack_start = ORIGIN(RAM) + LENGTH(RAM);
```

1. Add these dependencies to `Cargo.toml`
```toml
    [dependencies]
    cortex-m = "*"
    cortex-m-rt = "*"
    panic-semihosting = "*"
    
    [dependencies.stm32f30x-hal]
    features = ["rt"]
    version = "*"
```

1. Update `src/main.rs`

```rust
#![no_std]
#![no_main]

#[macro_use(entry, exception)]
extern crate cortex_m_rt as rt;
extern crate cortex_m;
extern crate panic_semihosting;
extern crate stm32f30x_hal;

use rt::ExceptionFrame;
use core::ptr;
use cortex_m::asm::nop;

entry!(main);

fn main() -> ! {
    const RCC_AHBENR: u32 = 0x40021014;
    // page 51
    const GPIOC_MODER: u32 = 0x48000800;
    // page 51 for start of GPIOC plus offset 18 on page 240
    const GPIOC_BSRR: u32 = 0x48000818;

    unsafe {
        // page 148
        *(RCC_AHBENR as *mut u32) = 1 << 19;

        // page 237
        *(GPIOC_MODER as *mut u32) = 1 << 30;

        loop {
            // page 240
            ptr::write_volatile(GPIOC_BSRR as *mut u32, 1 << (15+16));

            for _i in 0..100_000 {
                nop();
            }

            ptr::write_volatile(GPIOC_BSRR as *mut u32, 1 << (15));

            for _i in 0..100_000 {
                nop();
            }
        }
    }
}

exception!(HardFault, hard_fault);

fn hard_fault(ef: &ExceptionFrame) -> ! {
    panic!("{:#?}", ef);
}

exception!(*, default_handler);

fn default_handler(irqn: i16) {
    panic!("Unhandled exception (IRQn = {})", irqn);
}
```

## What does it do?

The code above directly modifies register values on the STM32F3 in order to flash the LED which is built into the flight controller. Given this board is already supported by the `stm32f30x` and `stm32f30x-hal` crates, this code is lower level than necessary, but it's good practice understanding the ST documentation. If you'd like to follow along, the page numbers in the code comments are from [this manual](https://www.st.com/content/ccc/resource/technical/document/reference_manual/4a/19/6e/18/9d/92/43/32/DM00043574.pdf/files/DM00043574.pdf/jcr:content/translations/en.DM00043574.pdf).

The other knowledge necessary to write this code is which pin the LED is connected to. I have not been able to track down a full schematic for this board, but I've been getting by so far with the following resources:

   [Implementation code for this flight controller from the beta flight project](https://github.com/betaflight/betaflight/tree/master/src/main/target/BETAFLIGHTF3)
   
   [Helpdesk article from the manufacturer](https://betafpv.freshdesk.com/support/solutions/articles/27000044289-cli-dump-of-fcs)
 
 Both of these links say there is a beeper on pin C15, and that the beeper signal is inverted. It turns out that there is no beeper on this board, but there is an LED attached to that pin. The signal to the LED is inverted, so turning the pin off turns the LED on. 

## Building & Flashing

The flight controller has a micro-usb port which can be used for communications or programming via the DFU bootloader. To flash the chip you'll need [dfu-util](http://dfu-util.sourceforge.net/), or some other DFU tool. Press and hold the boot button on the flight controller while inserting the USB cable and the board will startup in bootloader mode. This is indicated by a solid blue LED on the bottom of the board (the red LED on the bottom of the board should be off). From there, it takes only a few commands to build the Rust binary and load the program.

```bash
cargo build --release
arm-none-eabi-objcopy -O binary ./target/thumbv7em-none-eabihf/release/drone drone.bin
dfu-util -D drone.bin --alt 0 -R -s 0x08000000
```

After the loading is complete, restart the controller by unplugging and re-plugging it into the USB port and you should see the red LED on the bottom of the board blinking. 

Note, the top of the board has a red LED which always blinks (even in bootloader mode). The bottom of the board has one red LED and one blue LED. I'm not sure at this point how to control the blue LED, or if it is even possible. 

## Future Work

The next steps for development on this hardware are to continue to reverse engineer the board layout, as well as to develop drivers for the on-board devices. The initial goal would be creation of a [board support crate](https://github.com/rust-embedded/awesome-embedded-rust#board-support-crates), as well as a driver for the gyroscope / accelerometer.   

If you'd like to work alongside me, the exact board I purchased is available via [Amazon](https://www.amazon.com/BETAFPV-Brushed-Controller-Integrated-Receiver/dp/B074GXKKZ7).

##### Update - July 31, 2018

The [board support crate for the BetaFPV F3](https://github.com/JoshMcguigan/betafpv-f3) is now available on GitHub. It is a work in progress but it currently provides high level abstractions for the LED, motors, and motion processing unit. I also wrote a blog post [introducing the BetaFPV F3 board support crate](/blog/betafpv-drone-flight-controller-board-support-crate/).


 
