---
title: Weekly links (Ed. 2)
date: 2025-01-12T18:28:01+01:00
summary: |
  What I've read and done this week.
tags:
  - rust
  - tokio
  - postgres
  - sre
---

Hi reader,

This week was busy with work, and gaming as I started [Zelda: TOTK](https://fr.wikipedia.org/wiki/The_Legend_of_Zelda:_Tears_of_the_Kingdom) (and it was better late than never). I also started to work resolving challenges available on [Protohackers](https://protohackers.com/), using Rust & Tokio. Links I found interesting this week were:

- [Postgres UUIDv7 + per-backend monotonicity](https://brandur.org/fragments/uuid-v7-monotonicity) - The UUIDv7 RFC 9562 is not one year old and Postgres maintainers already imported their own implementation in their database engine;
- [The Evolution of SRE at Google](https://www.usenix.org/publications/loginonline/evolution-sre-google); [Causal Analysis based on System Theory](https://github.com/joelparkerhenderson/causal-analysis-based-on-system-theory) looks like a good read, but I haven't had time to read it yet ([hn](https://news.ycombinator.com/item?id=42584750), [lb](https://lobste.rs/s/8vgmiv/evolution_sre_at_google));
- While I'm lookin at the [Tokio](https://tokio.rs/) ecosystem, I found out that [axum 0.8.0](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0) was released. I built some small webservices in Rust in the past, using [Rocket](https://rocket.rs/) or [Actix](https://actix.rs/), and I think I'll give a try to axum soon.
