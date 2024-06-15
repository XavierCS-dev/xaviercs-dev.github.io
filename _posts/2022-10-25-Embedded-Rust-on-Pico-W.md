---
title: Running Rust on a PicoW
layout: post
tags: pico-w raspberry-pi rust embedded
author: Xavier
---

The Raspberry Pi PicoW is a powerful and inexpensive microcontroller with wireless support. However, when trying to run the classic "blinky" example from [rp-hal](https://github.com/rp-rs/rp-hal/tree/main/boards/rp-pico), you may have discovered that it does not work. This is because the onboard LED (GPIO25) must be accessed through the wireless controller. A port of the driver is being written [as we speak](https://github.com/embassy-rs/cyw43).

Fret not however, as it is still possible to access most of the other GPIO with rp-hal. In this article, you will learn how to setup a Rust development environment
for the Pi Pico/W, and blink 3 LEDs in a controlled pattern. If you are intent on using the wireless chip, you may want to look into [creating your own bindings](https://rust-lang.github.io/rust-bindgen/) or using [both C and Rust together](https://www.youtube.com/watch?v=zSWkrpu8KBA).

This article assumes you have basic developer utilities such as git, gcc and a text editor / IDE installed. It is also geared towards Linux however you should be able to follow along with Windows and MacOS

## The Setup

In this case I will be using CLion as my IDE, however this does not matter as the setup only depends on regular cargo. We are also going to be using Rustup as part of the installation process. You can install Rustup with:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

or run the installer if you are on windows. You are also going to want to make sure Rustup is up-to-date:

```bash
rustup self update
rustup update stable
```

Now to install the essential components we'll need to compile and run code for the Pi Pico. Firstly we are going to want to add the cross compilation target for ARM Cortex-M0/+/M1 because chances are, you aren't writing a program with a device utilising those cores:

```bash
rustup target add thumbv6m-none-eabi
```

Next we are going to want to install elf2uf2-rs which will automatically convert the ELF files rustc produces into uf2 files and load them onto a pi pico connected to your PC via USB:

```bash
cargo install elf2uf2-rs --locked
```

Make sure to install flip-link, the reason why we need this will be explained further on, but long story short, it stops the stack from running into the data segment.

```bash
cargo install flip-link
```

## The Environment

We will use the cargo project kindly crafted by some fellow Rust developers [here](https://github.com/rp-rs/rp2040-project-template).

First we will make a copy of the template:

```bash
git clone https://github.com/rp-rs/rp2040-project-template
```

Now open the cargo project in your favourite IDE or text editor. You will be met with quite a few different files, some of which may be unfamiliar to you. Don't worry, not all of them are relevant to us and I will try my best to explain what is. The file structure should look something like this:

```
.
├── build.rs
├── .cargo
│   └── config.toml
├── Cargo.lock
├── Cargo.toml
├── CODE_OF_CONDUCT.md
├── debug_probes.md
├── Embed.toml
├── .github
│   └── workflows
│       └── ci_checks.yml
├── memory.x
├── README.md
├── src
│   └── main.rs
└── .vscode
    ├── launch.json
    └── settings.json
```

Firstly I would like to draw your attention to something we are already somewhat familiar with: ".cargo/config.toml". `config.toml` contains the config for the cargo build system for this specific project. We actually want to make a change to this file, because by default, it assumes we are using probe-run as our runner, but we are going to be using elf2uf2-rs. So change:

```toml
runner = "probe-run --chip RP2040"
# runner = "cargo embed"
# runner = "elf2uf2-rs -d"
```

to

```toml
# runner = "probe-run --chip RP2040"
# runner = "cargo embed"
runner = "elf2uf2-rs -d"
```

## What's all this?

If you aren't interested in the details, you may skip ahead to "Breadboard Setup".

Just below where we set our runner, you should see a structure `rustflags = [..]`. These are simply custom flags to pass to the compiler upon being invoked. Firstly you can see each line has 2 arguments, with `-C` being the first on each line. `-C` is a rustc flag that allows you to specify codegen options such as:

- Which core model to use
- How many codegen units the crate is split into
- Platform security features
- Linker arguments and a host of other features.

I will go over the 4 linker arguments at the top and why they are needed. If you wish to learn about the code-size optimisations, there are a tonne of easy to find online resources which you can learn from, as the config.toml file already gives a good high level view of what they are.

### flip-link

Unfortunately, Rust programs using the `cortex-m-rt` crate aren't necessarily memory safe even without `unsafe` blocks. This is because in the default memory layout for programs written for ARM Cortex-M processors, the stack grows downwards which, in the case of an overflow, would cause the stack to collide with the data segment. `flip-link` fixes this problem by placing the stack below the data region, so when the stack overflows, it means memory is full and creates a hardware exception instead. You can learn more about `flip-link` [here](https://github.com/knurling-rs/flip-link).

### nmagic

The nmagic flag tells the linker (lld for ARM platform targets) to not page align sections, only link with static libraries, and to mark the output as NMAGIC if the output supports unix style magic numbers. 

### Tlink.x

Tlink.x is a custom linker script for the `cortex-m-rt` crate, which you can find more about in the "Dependencies" section.

### Tdefmt.x

Tdefmt.x is a custom linker script for the `de-fmt` framework, which you can find more about in the "Dependencies" section.

### Build Target

```toml
[build]
target = "thumbv6m-none-eabi"
```

This is quite simple to understand, `thumbv6m` specifies we are building for an ARM Cortex M0/+/M1 (M0+ in our case), `none` means we are building for a free standing environment (No OS), `eabi` or Embedded Application Binary interface defines things such as how data types are laid out in memory, how program initialisation functions, and how other such things are accessed. EABI is the default tool chain application binary interface for ARM. You can learn more about ABIs [here](https://en.wikipedia.org/wiki/Application_binary_interface).


### Build.rs

The file is already commented with an explanation that explains it fairly well, but to summarise, it copies the `memory.x` file from the project root to where the linker can always find it. In our case we won't really need it, but it is still useful to have for more complicated projects.

### Memory.x

The memory.x file specifies the memory region information of the target platform. The memory available to the device must be specified by the RAM and FLASH regions, but in the case, we also have the BOOT2 region, which is where the rp2040-boot2 second-stage bootloader will go. The text and read-only data will go into the FLASH region,  the blocking starting symbol (bss, static variables which are declared but not have been assigned a value) and data segments will go into the RAM region.

### Dependencies

Now looking at the `/Cargo.toml` file, we have 7 dependencies that come enabled by default. 

- The `cortex-m` [crate](https://docs.rs/cortex-m/latest/cortex_m/) provides low level access to Cortex-M processors allowing access to core peripherals and registers and also allowing the manipulation of interrupt mechanisms.
- The `cortex-m-rt` [crate](https://docs.rs/cortex-m-rt/latest/cortex_m_rt/) provides startup code and a minimal runtime for Cortex-M microcontrollers.
- The `embedded-hal` [crate](https://docs.rs/embedded-hal/latest/embedded_hal/) is a hardware abstraction layer (HAL) that seeks to erase device specific details by providing a minimal, zero-cost API.
    - A HAL allows a programmer to interact with hardware with a simplified interface.
- The `de-fmt` [crate](https://crates.io/crates/defmt) is a (blazingly fast 🚀) logging framework for embedded devices.
- The `de-fmt-rtt` [crate](https://crates.io/crates/defmt-rtt) allows the transmission of `de-fmt` log messages over the Real Time Transfer Protocol (RTP). The RTP uses UDP to transfer data and is usually used for streaming media.
- The `panic-probe`[crate](https://crates.io/crates/panic-probe) is a panic handler which tells our code what to do upon panic. In this case it exists probe-run with an error code. However we will be using elf2uf2 as our runner.
    - Keep in mind we are using elf2uf2 to keep things simple, `probe-run` is an incredibly useful debugging tool, it cannot be overstated, learn more about it [here](https://github.com/knurling-rs/probe-run). If you haven't already, make sure you read the [Rust Embedded Book](https://docs.rust-embedded.org/book) after you have read this article.
- The `rp-pico` [crate](https://crates.io/crates/rp-pico) provides board support for the Pi Pico which includes `rp2040-hal` but also configures the pins of the RP2040 micro-controller to better match how it is wired on the Pico.

## Breadboard Setup

![image](https://www.dropbox.com/s/13slwrt5mo931v4/pico_setup.jpg?raw=1)

Key:
- Wire connection: -
- LED connection: + 
```
GND        USB
 |    GP0       VBUS
 |    GP1       VSYS
 |----GND       GND
 |    GP2       3V3_EN
 |++++GP3       3V3(OUT)
 |++++GP4       ADC_VREF
 |++++GP5       GP28
```

## The Code

You may have alreadt taken a peak at `main.rs`, and wow! There is a lot to get through. What you see here is likely different to what you have in your `main.rs` currently, this is because I have added some code to blink 3 LEDs in a controlled fashion, which I will also explain.

{% highlight rust %}
#![no_std]
#![no_main]
{% endhighlight %}

At the very top of the file you can see two crate level attributes. `!#[no_std]` simply tells Rust to not link the standard library and instead link to the `core` crate. The `libcore` crate is platform-agnostic but only has part of the functionality of the `std` crate.

`!#[no_main]` simply means we will not be using the standard main function, and will specify our own entry point.

{% highlight rust %}
#[entry]
fn main() -> ! {
{% endhighlight %}

Since we don't have the usual Rust entry function, we use the `#[entry]` macro from the `cortex_m_rt` crate to specify one. The entry macro must appear in the dependency graph exactly once. The function specified will be called by the reset handler once RAM has been initialised, and the FPU has been enabled [if applicable](https://docs.rs/cortex-m-rt-macros/0.1.5/cortex_m_rt_macros/attr.entry.html). 

You may notice that our main function returns `!`. If you haveven't seen it before, `!` means the function does not return. In our case, we don't want it to return, because that means the program will end, which is not what we want.

{% highlight rust %}
let mut pac = pac::Peripherals::take().unwrap();
let core = pac::CorePeripherals::take().unwrap();
let mut watchdog = Watchdog::new(pac.WATCHDOG);
let sio = Sio::new(pac.SIO);
{% endhighlight %}

`pac::Peripherals::take().unwrap()` and `let core = pac::CorePeripherals::take().unwrap();` simply get all the peripherals as an instance of a struct, wrapped in an `Option<T>` enum, and returns it. This method can only be called sucessfully once, and includes modules such as the Memory Protection Unit and System Control Block.

 `let mut watchdog = Watchdog::new(pac.WATCHDOG);` gets a `Watchdog` timer instance and is used to detect and recover from malfunctions.

 `let sio = Sio::new(pac.SIO);` Gets a single-cycle I/O block. The Cortex-M0+ implements a memory mapped single-cycle I/O port for access to peripherals, but the I/O port [does *not* support code execution](https://developer.arm.com/documentation/dui0662/b/Cortex-M0--Peripherals/Single-cycle-I-O-Port).

{% highlight rust %}
let external_xtal_freq_hz = 12_000_000u32;
let clocks = init_clocks_and_plls (
    external_xtal_freq_hz,
    pac.XOSC,
    pac.CLOCKS,
    pac.PLL_SYS,
    pac.PLL_USB,
    &mut pac.RESETS,
    &mut Watchdog,
)
.ok()
.unwrap()
{% endhighlight %}

Wow, there is quite a lot to `unpac` here. `let external_xtal_freq_hz = 12_000_000u32;` specifies the crystal frequency of the Pi Pico. The clock of the Pi Pico is controlled by a crystal oscillator, which you can learn more about [here](https://www.electronics-tutorials.ws/oscillator/crystal.html).

`init_clocks_and_plls()` initialises the clocks and PLLs, then returns a `ClocksManager` instance wrapped in a `Result<T,E>` enum. A PLL allows a circuit board to synchronise its clock with an external timing signal.

Overall, into the function we pass: external crystal frequency, the crystal oscillator, the clocks, PLL for the system clock (133MHz) and PLL for the USB reference clock (48MHz). We also pass in the RESETS and watchdog timer. To better understand how all these components work on the Pi Pico, I recommend reading the [datasheet](https://datasheets.raspberrypi.com/pico/pico-datasheet.pdf).

{% highlight rust %}
let mut delay = cortex_m::delay::Delay::new(core.SYST, clocks.system_block.freq().to_HZ());
{% endhighlight %}

The above code creates a delay driver instance using the SysTick which is a timer that is part of the Cortex-M0+ [NVIC](https://www.motioncontroltips.com/what-is-nested-vector-interrupt-control-nvic/) controller.

{% highlight rust %}
let pins = bsp::Pins::new (
    pac.IO_BANK0,
    pac.PADS_BANK0,
    sio.gpio_bank0,
    &mut pac.RESETS
);
{% endhighlight %}

`bsp::Pins::new()` creates a `Pins` instance, used for interfacing with the Raspberry Pi Pico pins. In this case we pass in all the banks, and the RESETS. An IO pin is part of a specific IO bank. For example, the GPIO pins are found in the gpio bank.

{% highlight rust %}
let mut gpio5 = pins.gpio5.into_push_pull_output();
let mut gpio5 = pins.gpio4.into_push_pull_output();
let mut gpio5 = pins.gpio3.into_push_pull_output();
{% endhighlight %}

The above code is rather simple, it just sets 3 pins into push pull output state. This means the pins can either be ON (high) or OFF (low).

{% highlight rust %}
loop {
    gpio5.set_high().unwrap();
    delay.delay_ms(500);
    gpio5.set_low().unwrap();
    gpio4.set_high().unwrap();
    delay.delay_ms(500);
    gpio5.set_high().unwrap();
    delay.delay_ms(500);
    gpio4.set_low().unwrap();
    gpio5.set_low().unwrap();
    gpio3.set_high().unwrap();
    delay.delay_ms(500);
    gpio5.set_high().unwrap();
    delay.delay_ms(500);
    gpio5.set_low().unwrap();
    gpio4.set_high().unwrap();
    delay.delay_ms(500);
    gpio5.set_high().unwrap();
    delay.delay_ms(500);
    gpio3.set_low().unwrap();
    gpio4.set_low().unwrap();
    gpio5.set_low().unwrap();
    delay.delay_ms(500);
}
{% endhighlight %}

This is the main loop that will never terminate. Here we are blinking the LEDs (settings the pins high/low for on/off) in a controlled fashion. The pattern in which they will blink may already be obvious to you just looking at the code. `set_high()` and `set_low()` returns a `Result<T,E>` because writing to the GPIO registers could fail. There is a 500ms delay so we can see the LEDs blinking at a reasonable pace.

## Voila!

Here is the full code:

{% highlight rust %}
//! Blinks the LED on a Pico board
//!
//! This will blink an LED attached to GP25, which is the pin the Pico uses for the on-board LED.
#![no_std]
#![no_main]

use bsp::entry;
use defmt::*;
use defmt_rtt as _;
use embedded_hal::digital::v2::OutputPin;
use panic_probe as _;

// Provide an alias for our BSP so we can switch targets quickly.
// Uncomment the BSP you included in Cargo.toml, the rest of the code does not need to change.
use rp_pico as bsp;
// use sparkfun_pro_micro_rp2040 as bsp;

use bsp::hal::{
    clocks::{init_clocks_and_plls, Clock},
    pac,
    sio::Sio,
    watchdog::Watchdog,
};

#[entry]
fn main() -> ! {
    info!("Program start");
    let mut pac = pac::Peripherals::take().unwrap();
    let core = pac::CorePeripherals::take().unwrap();
    let mut watchdog = Watchdog::new(pac.WATCHDOG);
    let sio = Sio::new(pac.SIO);

    // External high-speed crystal on the pico board is 12Mhz
    let external_xtal_freq_hz = 12_000_000u32;
    let clocks = init_clocks_and_plls(
        external_xtal_freq_hz,
        pac.XOSC,
        pac.CLOCKS,
        pac.PLL_SYS,
        pac.PLL_USB,
        &mut pac.RESETS,
        &mut watchdog,
    )
    .ok()
    .unwrap();

    let mut delay = cortex_m::delay::Delay::new(core.SYST, clocks.system_clock.freq().to_Hz());

    let pins = bsp::Pins::new(
        pac.IO_BANK0,
        pac.PADS_BANK0,
        sio.gpio_bank0,
        &mut pac.RESETS,
    );

    let mut gpio5 = pins.gpio5.into_push_pull_output();
    let mut gpio4 = pins.gpio4.into_push_pull_output();
    let mut gpio3 = pins.gpio3.into_push_pull_output();

    loop {
        gpio5.set_high().unwrap();
        delay.delay_ms(500);
        gpio5.set_low().unwrap();
        gpio4.set_high().unwrap();
        delay.delay_ms(500);
        gpio5.set_high().unwrap();
        delay.delay_ms(500);
        gpio4.set_low().unwrap();
        gpio5.set_low().unwrap();
        gpio3.set_high().unwrap();
        delay.delay_ms(500);
        gpio5.set_high().unwrap();
        delay.delay_ms(500);
        gpio5.set_low().unwrap();
        gpio4.set_high().unwrap();
        delay.delay_ms(500);
        gpio5.set_high().unwrap();
        delay.delay_ms(500);
        gpio3.set_low().unwrap();
        gpio4.set_low().unwrap();
        gpio5.set_low().unwrap();
        delay.delay_ms(500);
    }
}
{% endhighlight %}

Now, plug the Pi Pico into your computer while holding the BOOTSEL button. Once your computer has detected the Pico as a storage device, use `cargo run` and then your Pi should disconnect from your computer and start blinking the LEDs in a predictable fashion, given your board setup is correct.