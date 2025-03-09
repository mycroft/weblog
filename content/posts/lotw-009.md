---
title: Weekly links (Ed. 9)
date: 2025-03-09T08:40:42+01:00
summary: |
  What I've read, done and learned this week.
tags:
  - golang
  - raft
  - cassandra
  - scylladb
  - kubernetes
---

Hi reader,

We're in March! Soon will come the spring. Cold is gone for now! And I've yet to go outside to verify this. Gosh I wish I had less work to be done.

This week was focused on fixing the fallout of the recent technology migration. I've restarted to write some git management code from scratch, with tests this time. I hope I'll have time to continue on this.

Links I found interesting this week:

- [Engineering Success Starts with Onboarding](https://tech.loveholidays.com/engineering-success-starts-with-onboarding-dc6519c99263) - Companies suck at on-boarding. Managers suck at it too. They fail to enable sharing mandatory knowledge, to mentor clearly newcomers and are not available to solve problems.
- [Designing chat architecture for reliable message ordering at scale](https://ably.com/blog/chat-architecture-reliable-message-ordering) - Fun fact: This week I've seen some random guys talking about cassandra and that using it made very easy building discord over it. Despite [Discord stores trillon of messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) on Discord, the word "easy" triggered me. This is not **easy**. This is **hard**. And the Ably blog post demonstrates how hard it is.
- [Implementing Raft](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/) - I need to read and play this whole article series. I'd like to build a simular thing in Rust or something.
- [Introducing Yoke - The IaC package manager for K8s](https://www.reddit.com/r/kubernetes/comments/1ckxmgm/introducing_yoke_the_iac_package_manager_for_k8s/) - I've been using [cdk8s](https://cdk8s.io/) for years now to maintain my Kubernetes cluster, in addition to [flux](https://fluxcd.io/). However, cdk8s is no longer maintained and is somehow a pain in the ass due to some issues, espacially when importing existing CRDs. I'll take Yoke a look when I'll have some time as I wishes to keep my golang code.

I did not play much this week. I've watched the last Star Trek movies, as well as the first season of Severance. The 2nd will end soon so it'll be time to catch up. I'm currently listening to _Wild Horses_ by The Sundays.

Everyone, have a great week.
