---
title: "Merging gitlabci and gitlabenv"
date: 2019-12-17T22:14:43+03:00
showDate: true
draft: false
---

Recently I announced two new projects:
[gitlabenv](/post/new-project-gitlabenv/) and
[gitlabci](/post/new-project-gitlabci/). They both made for easing the problems
I faced while using Gitlab CI. Now I'm merging two projects into one. Therefore
`gitlabci` is the successor of `gitlabenv`.

After merge;

```
$ gitlabenv list group/project
```

turns to

```
$ gitlabci env list group/project
```

and

```
$ gitlabci list group/project
```

turns to

```
$ gitlabci pipeline list group/project
```

[Download gitlabci](https://github.com/egegunes/gitlabci/releases) here.
