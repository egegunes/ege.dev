---
title: "New project: gitlabci"
date: 2019-12-02T00:58:39+03:00
draft: false
---

At Artistanbul, we usually have multiple repositories for a client and a
release often requires running a pipeline on some or all of them. Besides the
[pain of managing environment variables](/post/new-project-gitlabenv/), the
second annoying thing about Gitlab CI is lack of a dashboard to see all pipelines
of a group. There is a 4 years old
[issue](https://gitlab.com/gitlab-org/gitlab/issues/7861) about it (still
active though).

I created [gitlabci](https://github.com/egegunes/gitlabci) for this reason. You
can [see the
usage](https://github.com/egegunes/gitlabci/blob/master/README.md), [download
the binary](https://github.com/egegunes/gitlabci/releases) for your OS,
[submit](https://github.com/egegunes/gitlabci/issues) bugs and feature request
on Github. All contributions are welcome!
