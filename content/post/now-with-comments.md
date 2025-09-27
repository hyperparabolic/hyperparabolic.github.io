---
title: 'Now With Comments'
date: 2025-09-27
author: Spencer Balogh
description: Comment section powered by giscus.
tags:
  - blog.decent.id
---

I've wanted comments here for a second, but this is a static site (GitHub pages) and I've always had some dealbreaker turnoff with every embeddable option I've looked into. I saw [giscus](https://giscus.app) recently and thought it looks pretty nice.

<!--more-->

Anything you comment here will automatically open a [discussion in my blog's repo](https://github.com/hyperparabolic/hyperparabolic.github.io/discussions/categories/comments).

## Pros

- This uses GitHub Discussions as a backend. A barrier to entry is necessary, but it's not a hard one.
- Possible to self host if giscus needs to implement limits.
- Reasonable to soft migrate. Even if this blog switches to self hosting it looks like discussions from the repo can still be used as long as titles still match.
- Reasonable to hard migrate. Discussions can be queried with the GraphQL API.

## Cons

- It does require a GitHub account. Sorry if you don't like GitHub. I do understand, but it's an ecosystem that I'm pretty well invested into at this point.
- The OAuth permissions screen looks gnarly. "Act on my behalf" is a terribly vague way to describe permissions. However, this can't exceed the app permissions, which only include `Read access to metadata` and `Read and write access to discussions`, and only for this blog's repo.
