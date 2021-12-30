---
title: "101 Chunks of Rust"
date: 2021-12-28T18:31:03+01:00
draft: true
summary: "Keeping a track of usefull Rust snippets."
---

I'm ~~learning~~ writing more & more rust. But I really lack writing idiomatic code. So, I've decided to gather small useful Rust snippets on this page. No particular order, the more I want to write, the more it will be written on it.

Goal: One snippet, one crate/doc page/poc/whatever.

Have fun!


# The Snippets

## Navigating into the command line args

```rust
use std::env;

fn main() {
    println!("There is {} env.", env::args().len());
    for arg in env::args() {
        println!("- {}", arg);
    }
}
```

Also, you can check the [clap crate](https://crates.io/crates/clap).


## Filtering numbers

```rust
use std::env;
use std::str::FromStr;

fn main() {
    let numbers = env::args()
        .skip(1)
        .map(|arg| u64::from_str(&arg))
        .filter(|arg| arg.is_ok())
        .map(|arg| arg.unwrap())
        .collect::<Vec<u64>>();
    println!("{:?}", numbers);
}
```

The [std::str::FromStr](https://doc.rust-lang.org/std/str/trait.FromStr.html) trait documentation.


## Pretty logging

`pretty_env_logger` is a simple logger built on top of `log` and that is allowing to write logs using colors.

```rust
extern crate pretty_env_logger;
#[macro_use] extern crate log;

fn main() {
    pretty_env_logger::init_timed();
    info!("such information");
    warn!("o_O");
    error!("much error");
}
```

Don't forget to use [log](https://crates.io/crates/log) & [pretty_env_logger](https://crates.io/crates/pretty_env_logger) crates to enable this.


## Returning values from a loop

```rust
fn main() {
    let mut count = 0;

    let result = loop {
        if count == 10 {
            break count * 10;
        }
        count += 1;
    };

    println!("result: {:?}", result);
}
```


## Simple password generator

```rust
// Don't forget to add the rand crate in Cargo.toml file:
//
// [dependencies]
// rand = "0.8.4"
//
use rand::prelude::*;

fn generate(size: usize) -> String {
    let mut rng = rand::thread_rng();

    let printable_chars = (0x20u8..=0x7e).map(|c| c as char).collect::<Vec<char>>();

    (0..size)
        .map(|_| rng.gen::<usize>() % printable_chars.len())
        .map(|idx| printable_chars[idx])
        .collect()
}

fn main() {
    for _ in 0..32 {
        println!("{}", generate(16));
    }
}
```

It makes use of the [rand](https://crates.io/crates/rand) crate.


# All the crates

  * [clap](https://crates.io/crates/clap): Command line argument parser;
  * [pretty_env_logger](https://crates.io/crates/pretty_env_logger): Simple logger built on top of env_logger, configurable via environment variables & writing nice colored messages depending their log levels;
  * [rand](https://crates.io/crates/rand): Random number generation, with quite some useful features.
