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



# All the crates

  * [clap](https://crates.io/crates/clap): Command line argument parser
