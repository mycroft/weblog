---
title: "Advent of Code 2022"
date: 2022-12-02T20:46:16+01:00
summary: My journey doing the AOC 2022
---

On this page, I'll sum up daily my experience doing the [Advent of Code 2022](https://adventofcode.com/2022).

## Day 1: Counting calories

That is the easy introduction day. Nothing much to say: Lists, doing sums. Then sorting, and picking 3 numbers.

```rust
    for line in lines {
        if line.is_empty() {
            all_cals.push(current_cal);
            current_cal = 0;
        } else {
            current_cal += line.parse::<i32>().expect("need a mumber");
        }
    }

    all_cals.push(current_cal);

    all_cals.sort();
    all_cals.reverse();
```

I should refactor the `sort` & `reverse` calls. I'm pretty sure this could be a 1 liner...


## Day 2: Paper, Scissors, Rock or Rock, Paper, Scissors or ...

I was too lazy. I'll be honest: I've written too much `match` in my solution. I should refactor the thing to do 0. But that was end of the week and I wanted to get a walk.

```rust
    let playtype = match a {
        Play::Rock => {
            match b {
                Play::Paper => Type::Loss,
                Play::Scissors => Type::Win,
                Play::Rock => Type::Draw,
            }
        },
```

Wah I've just seen this [blog post](https://fasterthanli.me/series/advent-of-code-2022/part-2) with quite better parsing using `try_from`. I really should use those.


## Day 3: Intersections

Today's exercise was about doing intersections of sets. Well, rust happens to have a very nice collection type for this: HashSet. And it also happens that this HashSet type has an `intersection` function. 

```rust
    while groups.len() > 0 {
        let inter = groups.pop().unwrap().intersection(&groups.pop().unwrap()).cloned().collect();
        let inter: HashSet<char> = groups.pop().unwrap().intersection(&inter).cloned().collect();

        for c in inter.iter() {
            score_part2 += badge_to_number(*c);
        }
    }
```

