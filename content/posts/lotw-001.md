---
title: Weekly links (Ed. 1)
date: 2025-01-04T14:28:59+01:00
summary: |
  What I've read and done this week.
tags:
- adventofcode
- rust
- observability
- tracing
- s3
- git
---

Hi reader,

As always, it's been a while, but I was busy finishing gathering the [Advent of code 2025 stars](https://adventofcode.com/). Except the usual 2/3 "you're not smart enough to solve those by yourself" problems, I'm quite happy with the [solutions I wrote, in Rust, as always](https://github.com/mycroft/challenges/tree/master/advent-of-code/2024).

This week, I've read a bit about [summary metrics](https://grafana.com/blog/2022/03/01/how-summary-metrics-work-in-prometheus/), especially the [Effective Computation of Biased Quantiles over Data Streams](http://dimacs.rutgers.edu/~graham/pubs/papers/bquant-icde.pdf) (aka CKMS algorithm), that is implemented in the golang or java prometheus client libraries. However, it seems the [python prometheus_client library](https://github.com/prometheus/client_python) does not implement this, yet?

The objects subjects I took a look at:

- [How I use git](https://registerspill.thorstenball.com/p/how-i-use-git), as I plan to suggest at work to work differently on internal repositories. It is always helpful to bring articles like this, that describes main guidelines on how to simply but efficiently work with git;
- [Getting started with tracing in Rust](https://www.shuttle.dev/blog/2024/01/09/getting-started-tracing-rust);
- [Building and operating a pretty big storage system called S3](https://www.allthingsdistributed.com/2023/07/building-and-operating-a-pretty-big-storage-system.html).
