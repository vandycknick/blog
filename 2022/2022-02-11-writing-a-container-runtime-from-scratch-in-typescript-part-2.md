---
id: ea2370bd-e1d1-442d-a8c9-a891bc4baa76
title: "Writing a container runtime from scratch in TypeScript, Part 2 Pivoting"
description:
date: 2022-02-11T20:00:00+01:00
categories: [scratch, containers, typescript]
cover: ./assets/2022-02-11-writing-a-container-runtime-from-scratch-in-typescript-part-2/cover.jpg
draft: true
---

PART 2: JUST SOME IDEAS AT THE MOMENT

- Use chroot to change to a new filessytem
- Explain issues with chroot
- Use pivot_root to move to a new filesystem
- Look into OCI bundles (skopeo)
- Not OCI compliant but look at overlay2 and

## Intro/Recap

## Pivoting into a new filesystem

While we end up with an isolated process at the moment, I guess many will argue that this isn't a container just yet. And that's indeed the case it's still missing some important features like an isolated filesystem.

What is needed to run a chroot environment? Not that much, since something like this already works:

```sh
$ mkdir -p new-root/{bin,lib64}
$ cp /bin/bash new-root/bin
$ cp /lib64/{ld-linux-x86-64.so*,libc.so*,libdl.so.2,libreadline.so*,libtinfo.so*} new-root/lib64
$ sudo chroot new-root
```

We create a new root directory, copy a bash shell and its dependencies in and run `chroot`. This jail is pretty useless: All we have at hand is bash and its builtin functions like `cd` and `pwd`.
