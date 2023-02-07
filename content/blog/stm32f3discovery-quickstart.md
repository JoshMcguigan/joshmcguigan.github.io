---
title: STM32F3DISCOVERY Quick Start
date: "2018-07-05"
---

The [Discovery book](https://rust-embedded.github.io/discovery/) is a great getting started guide for new-comers to embedded programming in Rust. If you haven't read it yet, I highly recommend it. However, it stops short of providing step-by-step instructions for setting up a cargo project targeting the STM32F3DISCOVERY board from scratch. And even if there was such a guide it would be [many steps](http://blog.japaric.io/quickstart/), and I was hoping for something as simple as `cargo new`.

It was for that reason I created [stm32f3discovery-quickstart](https://github.com/JoshMcguigan/stm32f3discovery-quickstart), a Bash script plus a few auxiliary files which make creating a new cargo project targeting the STM32F3DISCOVERY development board a one-liner. 

```bash
$ git clone https://github.com/JoshMcguigan/stm32f3discovery-quickstart.git
    Cloning into 'stm32f3discovery-quickstart'...
$ ./stm32f3discovery-quickstart/init
    Project Name: newest-micro-project
        Created binary (application) `newest-micro-project` project
$ cd newest-micro-project/
$ cargo run
    Updating registry `https://github.com/rust-lang/crates.io-index`
    Compiling cc v1.0.17                                                         
    ...
    Compiling newest-micro-project v0.1.0 (file:///Users/josh/Projects/newest-micro-project)
     Finished dev [unoptimized + debuginfo] target(s) in 41.82s
      Running `arm-none-eabi-gdb target/thumbv7em-none-eabihf/debug/newest-micro-project`
```

Once you have cloned the [stm32f3discovery-quickstart](https://github.com/JoshMcguigan/stm32f3discovery-quickstart) repository, creating a new STM32F3DISCOVERY project is as simple as `./stm32f3discovery-quickstart/init`. The script asks what name you'd like to give your project, and that's it. If you just wanted a simple way to generate a new STM32F3DISCOVERY project, you can stop reading here. If you're interested in how the script works, or you'd like to manually setup a cargo project to compile and run on the STM32F3DISCOVERY then read on. 

## How it works

The core of [stm32f3discovery-quickstart](https://github.com/JoshMcguigan/stm32f3discovery-quickstart) is a Bash script named `init`. I'll explain each part of the script below.
1. Asking the user for the project name, and creating the cargo binary
```bash
read -p 'Project Name: ' name
cargo new $name
```
1. Getting the directory that the `init` script is in (not the directory that the script is running from) - credit for this goes to [this stackoverflow answer](https://stackoverflow.com/a/246128)
```bash
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
```
1. Copying the cargo and GDB configuration files from the stm32f3discovery-quickstart directory into the new project directory
```bash
cp -r $DIR/.cargo ./$name/.cargo
cp $DIR/.gdbinit ./$name/.gdbinit
```
1. Overwriting `main.rs` with the [roulette example](https://github.com/japaric/f3/blob/master/examples/roulette.rs)
```bash
cp $DIR/src/main.rs ./$name/src/main.rs
```
1. Adding the necessary dependencies to `cargo.toml`
```bash
cat $DIR/dependencies >> ./$name/Cargo.toml
```

## Requirements

If you haven't read the Discovery book, I'd recommend working through chapters 3-5 before starting your own project using this tool. At a minimum, you'll need to follow the steps in chapter 3 to [set up your development environment](https://rust-embedded.github.io/discovery/03-setup/index.html).

If you have worked through the book, you already have everything you need.

## Future Improvements

I think it'd be great to see a [cargo subcommand](https://github.com/rust-lang/cargo/wiki/Third-party-cargo-subcommands) which encapsulates not only this behavior, but expands it to support many other ARM development boards. If you have any interest in either using or contributing to something like this, [join the discussion here](https://github.com/JoshMcguigan/stm32f3discovery-quickstart/issues/1).

## Acknowledgement

This leans heavily on a number of previous works by [Jorge Aparicio](http://japaric.io). Without his dedication to the embedded Rust community, I would not have even known where to start. 
