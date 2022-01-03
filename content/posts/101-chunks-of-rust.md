---
title: "101 Chunks of Rust"
date: 2021-12-28T18:31:03+01:00
summary: "Keeping a track of usefull Rust snippets."
---

I'm ~~learning~~ writing more & more rust. But I really lack writing idiomatic code. So, I've decided to gather small useful Rust snippets on this page. No particular order, the more I want to write, the more it will be written on it.

Goal: One snippet, one crate/doc page/poc/whatever.

This page was last updated on 2022-01-03 and it is still WIP!

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


## Swapping variables

The following snippet will swap 2 variables (which type has `Copy` trait), and will print their address.

```rust
fn swap<T: Copy>(a: &mut T, b: &mut T) {
    let z = *a;

    *a = *b;
    *b = z;
}

fn main() {
    let mut a = 1;
    let mut b = 2;

    println!("ptr: {:p}: val: {} // ptr: {:p}; val: {}", &a, a, &b, b);
    swap(&mut a, &mut b);
    println!("ptr: {:p}: val: {} // ptr: {:p}; val: {}", &a, a, &b, b);
}
```


## The Itertools crate

```rust
use itertools::Itertools;
use itertools::{merge,partition,zip};

fn main() {
    // Get all permutations
    println!("{:?}", (0..5).permutations(5).collect_vec());

    // Get all uniques
    println!("{:?}", vec![1, 2, 3, 2, 1].iter().unique().collect_vec());

    // combine elements from first array with second's 
    println!("{:?}", vec![0, 1, 2].iter().zip(vec![4, 5, 6]).collect_vec());
    println!("{:?}", zip(vec![0, 1, 2], vec![0, 1, 2]).collect_vec());

    // partition elements of array into 2.
    let mut l = vec![1, 2, 3, 4, 5, 6];
    partition(&mut l, |z| { z % 2 == 0 });
    println!("{:?}", l); // [6, 2, 4, 3, 5, 1]

    // merge two array into an iterator
    println!("{:?}", merge(&[0, 1, 2], &[4, 5, 6]).collect_vec()); // [0, 1, 2, 4, 5, 6]
    println!("{:?}", merge(&[0, 1, 2], &[0, 1, 2]).collect_vec()); // [0, 0, 1, 1, 2, 2]
}
```

Check the [itertools functions](https://docs.rs/itertools/latest/itertools/index.html) & [itertools trait](https://docs.rs/itertools/latest/itertools/trait.Itertools.html).


## Implementing Display & FromStr traits

```rust
use std::str::FromStr;
use std::num::ParseIntError;
use std::fmt;

#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "P <{},{}>", self.x, self.y)
    }
}

impl FromStr for Point {
    type Err = ParseIntError;

    fn  from_str(s: &str) -> Result<Self, Self::Err> {
        let parts = s.split(",")
            .collect::<Vec<&str>>()
            .iter().map(|&x| x.parse::<i32>().unwrap())
            .collect::<Vec<i32>>();

        Ok(Self {
            x: parts[0],
            y: parts[1],
        })
    }
}

fn main() {
    let p: Point = Point::from_str("-1,-1").unwrap();
    println!("{}", p);
}
```

As seen as on [Rust By Example](https://doc.rust-lang.org/rust-by-example/) ([Display](https://doc.rust-lang.org/rust-by-example/hello/print/print_display.html) of official doc); [FromStr](https://doc.rust-lang.org/std/str/trait.FromStr.html).


# Testing out Iterator traits

```rust
use rand::Rng;

#[derive(Clone, Copy, Debug)]
struct Random {
}

impl Iterator for Random {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        let n = rand::thread_rng().gen::<u32>();
        if n % 100 == 0 {
            None
        } else {
            Some(n)
        }
    }
}

fn main() {
    let random_numbers = Random{};

    for v in random_numbers {
        println!("- {}", v);
    }

    println!("There is {} elements in this run.", random_numbers.count());
}
```

The [Iterator trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html) documentation. [Rust By Example's](https://doc.rust-lang.org/rust-by-example/trait/iter.html).


## Generic structures & enums

```rust
struct Testaroo<T> {
    val0: T,
}

enum MyOption<T> {
    None,
    Some(Testaroo<T>)
}

fn main() {
    // All the following code is valid and can be used as it!
    let _ = Testaroo{ val0: "a" };
    let _ : Testaroo<u64> = Testaroo{ val0: 42 };
    let _ = Testaroo{ val0: 0.0 };

    let _ : MyOption<bool> = MyOption::None;
    let _ = MyOption::Some(Testaroo{ val0: true });
}
```


## Arrays, vec & slices

```rust
fn main() {
    let array : [u32; 4] = [1, 2, 3, 4];
    let array_vec = array.to_vec();
    let slice0 = &array[2..=3];
    let slice1 = &array_vec[1..=2];

    // Outputs: array:[1, 2, 3, 4] vec:[1, 2, 3, 4] slice:[3, 4] slice:[2, 3]
    println!("array:{:?} vec:{:?} slice:{:?} slice:{:?}",
        array, array_vec, slice0, slice1);

    // auto-filled array declaration:
    let vbool = [false; 100];

    let vec: Vec<u32> = Vec::with_capacity(256);
    println!("len:{} capacity:{}", vec.len(), vec.capacity());

    let s0: &[bool] = &vbool;
    println!("len:{}", s0.len());

    let print = |n: &[bool]| {
        for el in n {
            println!("{:?}", el);
        }
    };

    print(s0); // This will write a lot of "false".
    print(&s0[..42]); // First 42 elements
    print(&s0[99..]); // last element
}
```

More about [arrays](https://doc.rust-lang.org/std/primitive.array.html), [slices](https://doc.rust-lang.org/std/primitive.slice.html) & [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html).


# All the crates

  * [clap](https://crates.io/crates/clap): Command line argument parser;
  * [itertools](https://crates.io/crates/itertools): Extra iterator adaptors, functions and macros;é
  * [pretty_env_logger](https://crates.io/crates/pretty_env_logger): Simple logger built on top of env_logger, configurable via environment variables & writing nice colored messages depending their log levels;
  * [rand](https://crates.io/crates/rand): Random number generation, with quite some useful features.
