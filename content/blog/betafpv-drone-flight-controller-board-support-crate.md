---
title: BetaFPV F3 Drone Flight Controller - Board Support Crate
date: "2018-07-31"
---

In a [previous blog post](/blog/betafpv-drone-flight-controller-hello-rust/)  I described how to get a very simple Rust program compiling for and running on the BetaFPV F3 drone flight controller. Since that time I've been working to create a board support crate to provide a high level API for the board. 

## What is a Board Support Crate?

The BetaFPV F3 board is based on the STM32F303 microcontroller, which has strong support within the embedded Rust community. There is already a [STM32F30X device crate](https://github.com/japaric/stm32f30x) which provides a safe API for direct register level access to the microcontroller. On top of that sits the [stm32f30x-hal crate](https://github.com/japaric/stm32f30x-hal), which implements higher level traits like `OutputPin` and `DelayMs`. Board support crates build on the `device` and `hal` crates, creating an API for the board which can be both more limiting and more expansive than the two crates it is built upon. 

Board support crates attempt to disallow configurations of the board which are not supported by its physical layout. An example of this is that on the BetaFPV F3 board, pin A06 is used to control the motor 1 output. It does not make sense to configure that pin as an input, so the board support crate does not allow that functionality. 

On the other side, the board support crate includes features of the board beyond those offered directly by the MCU. For the BetaFPV F3, this means it provides an API to work with the motion processing unit, as well as potentially the radio receiver in the future. 

When using the board support crate to do LED control, unlike in [my previous post](/blog/betafpv-drone-flight-controller-hello-rust/) which was directly manipulating registers in unsafe Rust, or even building on top of the device or hal crate to safely manipulate pin C15 (the pin which controls the LED), all of the lower level details are abstracted away and we just control an `led` struct.

```rust
let Board {mut led, mut delay, ..} = Board::new();

loop {
    led.set_high();

    delay.delay_ms(500u16);

    led.set_low();

    delay.delay_ms(500u16);
}
```

## Writing a Board Support Crate

The goal of the board support crate is to expose the components of the board through a high level API, requiring as little knowledge of the underlying device as possible. 

Most Rust board support crates are written for development boards which have lots of IO and are very configurable for different use cases. For this reason, users are often required to interact with the underlying device and hal crates to properly configure the board. 

Since the BetaFPV F3 is not a development board it has a much more strictly defined feature set, which makes it easier to provide a high level API to the user without requiring knowledge of the underlying device or hal crates. The BetaFPV F3 board support crate provides a zero-configuration `new` method to create a `Board` struct, as shown below.

```rust
let Board 
    {
        mut led,
        mut mpu,
        mut delay,
        mut motor_1, 
        mut motor_2, 
        mut motor_3, 
        mut motor_4,
    } = Board::new();
    
// do things with the board components here
```

#### GPIO control

Board support crates capture the board's electrical schematic in code. The code sample below comes from the `Board::new()` method and shows how the `Led` and `MotorX` types are wrappers around the standard `embedded-hal` GPIO exposed by the `stm32f30x-hal` crate. It is the responsibility of the board support crate to ensure the correct pins are used for each output, and that those pins are configured appropriately. 

```rust
type Led = PC15<Output<PushPull>>;
type Motor1 = PA6<Output<PushPull>>;
type Motor2 = PA7<Output<PushPull>>;
type Motor3 = PB8<Output<PushPull>>;
type Motor4 = PB9<Output<PushPull>>;

let mut gpioa = dp.GPIOA.split(&mut rcc.ahb);
let mut gpiob = dp.GPIOB.split(&mut rcc.ahb);
let mut gpioc = dp.GPIOC.split(&mut rcc.ahb);

let led = gpioc
    .pc15
    .into_push_pull_output(&mut gpioc.moder, &mut gpioc.otyper);
let motor_1 = gpioa
    .pa6
    .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);
let motor_2 = gpioa
    .pa7
    .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);
let motor_3 = gpiob
    .pb8
    .into_push_pull_output(&mut gpiob.moder, &mut gpiob.otyper);
let motor_4 = gpiob
    .pb9
    .into_push_pull_output(&mut gpiob.moder, &mut gpiob.otyper);
```

#### Motion Processing Unit

Interfacing with the motion processing unit, in this case the MPU6000, is a bit more difficult. The MPU6000 combines a three-axis accelerometer with a three-axis gyrometer and exposes this data via the SPI bus. The job of the board support crate is to correctly configure an MPU6000 device driver instance and expose the resulting device struct via the board support crate API. Unfortunately, there was no existing Rust MPU6000 driver. 

I started writing a driver, but fortunately soon after discovered an embedded Rust driver existed for a related device, the [MPU9250](https://github.com/japaric/mpu9250). Comparing the [MPU6000 register map](https://www.invensense.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf) to the [MPU9250 register map](https://www.invensense.com/wp-content/uploads/2015/02/RM-MPU-9250A-00-v1.6.pdf) revealed a few differences, so I [forked](https://github.com/JoshMcguigan/mpu9250) the MPU9250 driver and tweaked it to work with the MPU6000. 

I also had to [fork](https://github.com/joshmcguigan/stm32f30x-hal) the [original](https://github.com/japaric/stm32f30x-hal) hal crate for the STM32F30X in order to expose the pins needed to communicate with the MPU. This was as simple as uncommenting out a few lines in the original source.

The following code snippet demonstrates reading data from the accelerometer using the board support crate (available in the examples folder as `mpu.rs`).

```rust
let Board {mut led, mut mpu, mut delay, ..} = Board::new();

// https://www.invensense.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf
// expected 0x68 based on register map
// some startup time is required or this assertion fails
delay.delay_ms(1000u16);
assert_eq!(mpu.who_am_i().unwrap(), 0x68);

// seeing a blinking LED means the assertion did not fail
for _i in 0..5 {
    led.set_high();

    delay.delay_ms(500u16);

    led.set_low();

    delay.delay_ms(500u16);
}

// LED controlled by orientation of board
loop {
    let board_up = mpu.accel().unwrap().z > 0;

    if board_up {
        led.set_high();
    } else {
        led.set_low();
    }

    delay.delay_ms(500u16);
}
```   

#### Debugging

It would be very difficult to continue development on this board without some form of feedback (beyond a single LED). To achieve this, I created a new [bit-bang-serial](https://github.com/JoshMcguigan/bit-bang-serial) crate with the goal of using one of the motor output ports to send debugging data back to my development machine. 

Initial attempts to use this crate with the BetaFPV board failed, although it was working when tested with the STM32F3DISCOVERY. Analysis with an oscilloscope revealed the reason for this. While the BetaFPV F3 board outputs had reasonably quick rise times, the fall times were very slow, and the output didn't return all the way back to 0v when commanded off. 

On the BetaFPV board, unlike the STM32F3DISCOVERY, the outputs are first passed through FETs to allow them to drive high power loads (up to 6.3 amps according to the manufacturer). This means the `PushPull` configuration on the output pins of the MCU can't actually pull the output back to ground when commanded off. Experimenting with various pulldown resistors, I found the fall time improved significantly with 10 ohms of resistance between the `tx` and `gnd` pins. I'd guess the results would be even better with less resistance, but I didn't have the resistors around to try that. 

With the pulldown resistor, fall time is on the order of 10ms, and I am able to communicate reliably enough for debugging purposes at 9600 baud. 

## Use

If you have a BetaFPV F3 board, running the examples from the board support crate is straightforward. First you'll need to install [dfu-util](http://dfu-util.sourceforge.net/), then clone the [BetaFPV F3 board support crate](https://github.com/JoshMcguigan/betafpv-f3). Hold the boot button on the board as you connect the USB cable to start the board in bootloader mode. Finally, run `./flash-example led-control` to compile the `led-control` example and flash it to the board. As of this writing there are four examples available:

 * led-control
 * motor-control
 * mpu
 * serial-comm

## Future Work

This board support crate is certainly a work in progress, but much of the future development will likely take place in other crates. Sensor fusion, combining the accelerometer and gyrometer data to accurately measure the orientation of the board, will be my next focus. There is a [madgwick](https://github.com/japaric/madgwick) crate which may provide a starting point, but it expects to also have magnetometer data, which this board does not have.   

Once the sensor fusion implementation is complete (or at least an initial prototype is ready for testing), work can begin on radio receiver support and flight control code. 

It would be helpful to have the USB port available for debugging, although that would require writing a full USB stack in `nostd` Rust, which as far as I know has not been done yet. Alternatively, it may be possible (and easier) to repurpose some of the pins from the USB port and use a different communication protocol, which would probably be sufficient for debugging purposes in the short term. 

## Getting involved

The [BetaFPV F3 board support crate](https://github.com/JoshMcguigan/betafpv-f3) is available on GitHub. The best way to get started is to purchase a BetaFPV F3 board and run the examples. The specific board I bought is available [here](https://www.amazon.com/BETAFPV-Brushed-Controller-Integrated-Receiver/dp/B074GXKKZ7). A slightly more expensive, although potentially more fun, alternative would be to purchase a [BetaFPV drone which includes the F3 flight controller](https://www.amazon.com/Beta85-Micro-Quadcopter-Receiver-8-5x20/dp/B078WRLGPQ). Note that you'll also need a battery charger and radio transmitter if you want to fly the drone with the stock firmware, as it doesn't come with that equipment. 

Drone flight is a complex problem, and I'd welcome additional contributors to this project. Even in these early stages, the work has spanned several crates, and I can see several more coming out of this. If you are interested in getting involved, please feel free to reach out and I'll do what I can to help you get started.  

## References, Resources, and Further Reading

 * Code references
    * [BetaFPV F3 board support crate](https://github.com/JoshMcguigan/betafpv-f3)
    * [The original stm32f30x-hal crate](https://github.com/japaric/stm32f30x-hal) and [my fork adding support for the pins used by the BetaFPV F3 board to communicate to the MPU](https://github.com/joshmcguigan/stm32f30x-hal)
    * [stm32f30x](https://github.com/japaric/stm32f30x)
    * [The original MPU9250 driver](https://github.com/japaric/mpu9250) and [my fork supporting the MPU6000](https://github.com/JoshMcguigan/mpu9250) 
    * [bit-bang-serial](https://github.com/JoshMcguigan/bit-bang-serial)
    * [madgwick](https://github.com/japaric/madgwick)
 * BetaFPV F3 pin configurations
    * [From the manufacturer](https://betafpv.freshdesk.com/support/solutions/articles/27000044289-cli-dump-of-fcs)
    * [From the BetaFlight project](https://github.com/betaflight/betaflight/tree/master/src/main/target/BETAFLIGHTF3)
 * [dfu-util](http://dfu-util.sourceforge.net/) - A utility for flashing embedded devices that support the DFU protocol
 * [Brave new I/O](http://blog.japaric.io/brave-new-io/) - A blog post by Jorge Aparicio which describes the relationship between the device, hal, and board support crates
