---
title: "[Summary] Just Use a Monorepo"
description: "Summary of pros and cons of monorepo"
date: 2023-01-21T23:06:58+08:00
draft: false
tags: [version control, monorepo, dependency]
---

Link: [Just use a monorepo](https://buttondown.email/blog/just-use-a-monorepo)

Buttondown, a newsletter service, recently migrated their projects from separate microrepositories to a single monorepo. They strongly recommends that a software business use monorepo as early as possible. The followings are pros and cons in the article and its reference.

Pros:
1. API rolls out smoothly with up-to-date documentation. There should be no conflict between front-end and backend of the system.
2. Avoid context-switching among different workspaces, which might have slightly different configurations.
3. Fix dependency issues. Packages and tools now have an universal version number. We don't need to keep track of version number of every dependency.
4. Easier to debug if we know exactly which commit # works fine.
5. Decrease overhead of cross-project changes.
6. Easier to set up dev environment and run all tests.

Cons:
1. Permission management will be hard if you want to hide some parts of the repo.
2. Hard to open-source some parts of the repo.
