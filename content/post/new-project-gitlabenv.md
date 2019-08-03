---
title: "New project: Gitlabenv"
date: "2019-07-14"
draft: false
---

At Artistanbul, we started using Gitlab CI. While building our pipeline, the
hardest part was managing environment variables. Gitlab's interface makes it really hard:

![Gitlab CI](/images/gitlabci.png)

[Gitlabenv](https://github.com/egegunes/gitlabenv) is a little command line application that makes managing this
variables easier. It gets the current state from Gitlab, you modify the
variables and it uploads your modified version to Gitlab.

It doesn't support removing variables yet. I'll add features as I need. If you
see any issues or have feedback [let me
know](https://github.com/egegunes/gitlabenv/issues).
