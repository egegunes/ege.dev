---
title: "watching it shatter"
date: 2026-03-06
tags: [""]
categories: [""]
---

This week, just like the previous, was filled with PostgreSQL (PG). I’m still working on v2.9.0 release of the PG operator.

Last week I merged the implementation of pg_tde to bring data-at-rest encryption to Kubernetes. Then we started doing QA: oh boy, did it ever surface problems!

One of the goals of v2.9.0 is declaring major version upgrades production-ready. So we decided to test the upgrade with TDE enabled. Boom, [data corruption](https://perconadev.atlassian.net/browse/PG-2240). [Point-in-time recovery](https://perconadev.atlassian.net/browse/PG-2239) is problematic, [pg_rewind](https://perconadev.atlassian.net/browse/PG-2234) too…

This is one of my favorite things about Kubernetes. You put something fragile on it and watch it shatter. Of course I need to give credit to our QA engineers: they don’t stop until they watch the software crumbling in pain.

Outside of PG stuff, I opened [a small PR](https://github.com/bergerx/kubectl-status/pull/590) for adding volume mount information to kubectl-status plugin. I needed to quickly check mounts for a few containers while testing PG major upgrades and did a quick implementation with Claude. Before opening the PR, I was unsure whether I should mention that the code was written by Claude. I still feel embarrassed about using AI code generation. In the end I decided to openly admit it because it’s the right thing to do. I also want our contributors to be open about their usage of AI (I know they use AI).

Speaking of AI code generation, I feel that the vibe has shifted significantly. I see the pessimists have accepted defeat and I am one of them: AI can write code and the code is not crap. It’s not 2023 anymore and it looks like it’s time to update my priors. I am trying to integrate Claude Code into my workflows and I had a bunch of positive interactions with it the last few weeks. I am also coding a big feature using Claude just for the sake of the experience.​​​​​​​​​​​​​​​​