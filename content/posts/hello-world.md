---
title: "Hello World"
date: 2010-05-06
summary:
  Every weblog should have a hello world page.
favorite: true
categories:
- rust
---

Collecting examples for as many languages and related programming environments for “Hello world!” program: http://www2.latech.edu/~acm/HelloWorld.html.

Rosettacode’s idea is to present solutions to the same task in as many different languages as possible, to demonstrate how languages are similar and different: https://rosettacode.org/wiki/Hello_world/Text.

*Updated in 2021: A little rust sample as I'm writing a lot in Rust nowadays:*

```rust
fn main() {
    println!("Hello world!");
}
```

You can even write *Hello World* with [zero line of code](https://codegolf.stackexchange.com/questions/215705/hello-world-in-zero-lines-of-code), and in rust it is a perfect 42 bytes!

```rust
/**/fn main(){print!("Hello, World!")}/**/
```