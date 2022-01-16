---
id: ea2370bd-e1d1-442d-a8c9-a891bc4baa76
title: "Add styling to an active link in Next.js"
description: 
date: 2022-01-18T20:00:00+01:00
categories: [scratch]
cover: ./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/cover.jpg
---

Containerization and especially Docker, has become immensely popular over the last couple of years. Even though Docker isn't unique or the first of its kind, it quickly became a developer's favourite thanks to its fantastic developer experience. Spawning a new shell with Docker is child's play, a quick `docker run -it ubuntu bash` provides a containerized ubuntu environment in no time. It's fast because there's no VM involved but still offers a good level of security by shielding off other parts of the operating system. Even wrapping up any service or website in an image and shipping it up to production can be done with just a few lines on the command line. Over time whole ecosystems have spun up around it, Kubernetes or Docker Swarm for orchestration, alternative engines like podman, ebpf flavoured networking components, and the list goes on and on. It's mind-boggling how many tools are out there and how amazing this community and ecosystem has become. But have you ever dared to take a step back and wondered how containers work under the hood? Have you ever asked yourself what components make up a container? Have you ever gone hunting down the Linux source code to find that one magical API call that creates a new container? Well, I certainly have, and in this post we are going on a bit of a journey deep down into the Linux kernel. We'll be diving deep into all the nitty-gritty details and see how the sausage gets made by creating a little containerization engine from scratch in TypeScript.

## Setting the scene

Before jumping straight down the deep end, there are a couple of things I would like to get out of the way first. If you already are a seasoned containerization specialist or only care about the code, feel free to jump right into the next section. If not, let's get through some of the basics first and make sure we're all on the same page. Throughout the rest of this article, I'm mainly going to look at things from a Docker perspective, but the core concepts basically apply to any container runtime out there. Using Docker for this exercise makes things easier giving most people will have some level of familiarity with it.

### Containers don't run on docker

This is a general misconception and the internet in generally wrong on this one. Ever looked up [docker architecture](https://www.google.com/search?q=docker+architecture&tbm=isch&ved=2ahUKEwjBm8z14qr1AhXTuqQKHUjwBLYQ2-cCegQIABAA&oq=docker+archit&gs_lcp=CgNpbWcQARgAMgQIABBDMgUIABCABDIFCAAQgAQyBQgAEIAEMgUIABCABDIFCAAQgAQyBggAEAUQHjIGCAAQBRAeMgYIABAFEB4yBggAEAUQHjoHCCMQ7wMQJzoICAAQsQMQgwE6CAgAEIAEELEDUMIGWL4RYI4aaABwAHgAgAHQAYgBsAmSAQYxMS4yLjGYAQCgAQGqAQtnd3Mtd2l6LWltZ8ABAQ&sclient=img&ei=EgjeYcHKOdP1kgXI4JOwCw&bih=911&biw=1904) on google. Most of the architecture diagrams here come straight from the docker marketing department. BUt containers do not run ON docker. Containers are just processes that run on the Linux kernel (or windows if thats what you fancy). 


Often thought of as cheap VMs, containers are just isolated groups of processes running on a single host. That isolation leverages several underlying technologies built into the Linux kernel: namespaces, cgroups, chroots, capabilities and lots of terms you've probably heard before. It basically boils down to these six things that allow us to create isolated processes:

![container engine architecture](./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/container-architecture.png)

1. Namespaces
2. Chroot/Pivot Root
3. Copy On Write
4. CGroups
5. Capabilities
6. Seccomp
7. SELinux / AppArmor

### Processes vs containers.

### Rise of the container engine's

![Container Engine Comparison](./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/ce-engine-comparison.png)

Are you also seeing a pattern here? Exactly, all these different container engines use the same core component called [runc](https://github.com/opencontainers/runc). Runc does all the heavy lifting of containerizing a process, and with its simple interface, a whole ecosystem of container engines have spun up around it. There's a bit of background here as to why runc became what it is today, but I'll save you the history lesson jump right into the TLDR. Docker used to be built as a monolithic application, all of that changed in 2015 when a considerable initiative resulted in docker becoming more modular and tools like containerd and runc saw the light of day. Remnants of this old past can still be found all over the internet: [oci initiative blog post](https://www.docker.com/blog/open-container-project-foundation), [runc announcement by solomon](https://www.docker.com/blog/runc), [initial libcontainer library before moving into runbc](https://github.com/docker-archive/libcontainer).

## Getting ready for development

The code used throughout this post will be available at https://github.com/vandycknick/runjs, the repo is chopped up into multiple branches matching up to the starting point of each of the upcoming sections. This should make it easy to jump back and forth and be able to start from a know working state. The structure of the repo look something like this:

```sh
├── libc
│   ├── mod.ts
│   └── ...
├── scripts
│   ├── download_image.sh
│   └── setup.sh
├── src
│   ├── deps.ts
│   └── main.ts
├── Makefile
├── README.md
└── Vagrantfile
```

The `libc` folder contains all the interop code required to interface with the Linux kernel. It leverages the [FFI](https://deno.land/manual@v1.17.3/runtime/ffi_api) available in Deno as of 1.13 to be able to call out into native code. When you hear native code, you might think unsafe code and dealing with pointers and memory. But that's not the case. Every function exported from `libc/mod.ts` are safe JavaScript wrappers that API wise closely match up to the native Linux syscall, which should make it easier to explore these API's online or through the man pages.

I'll mainly focus on `src/main.ts`, 

I highly recommend running any of the following code snippets in a Virtual machine. While 99% of the code we'll write is rather harmless, there is a small section in this post that could cause permanent data loss if you manage to get a few wires crossed. Thats exactly why the `Vagrantfile` is there, it allows me to provide a safe pretested environment. Don't worry if you don't know Vagrant or have never heard about it before. In essence, Vagrant is similar to Docker, but instead of creating containers with Vagrant you end up with full-blown virtual machines. You can easily get it installed from [here](https://www.vagrantup.com/), and a simple `vagrant up` followed by a `vagrant ssh` allows you to follow along with the rest of the article without issues. 

## Shielding of a process with namespaces

Namespaces are the Linux kernel feature that enables the containerization of a process. First introduced back in 2002 with Linux 2.4.19, the idea behind a namespace is to wrap specific global system resources in an abstraction layer. This abstraction layer makes it appear as if processes within a namespace have their own isolated set of resources. Different process groups can have different sets of namespaces applied to them. Initially only the mount namespaces existed but over time more of them got added to the kernel and at the moment there are seven distinct namespaces available:

- uts
- mnt
- pid
- net
- ipc
- user
- cgroup

Don't worry about these just yet; we'll discuss them more in detail once we start writing some code. But before we can go any further, we'll need to understand how processes are created in Linux. If you've ever programmed in python before, you probably know that you can create a new process by calling `subprocess.Popen` or any of its many derivatives. Many other programming languages offer a similar API like `exec.Command` in golang, `Process.Start` in dotnet or `child_process.spawn` in nodejs. While all these languages offer different API calls they all end up making the same syscalls in the Linux kernel named `fork`, `clone` and `execve`. While technically `fork` isn't actually a syscall but rather a wrapper around `clone`. For simplicity we are going to treat it as such, mainly because you will often find it referenced as a syscall online but it's also the simpler one of the two to call out to from Deno. Using these API's to create a new process looks something like this:

![fork-exec-syscall-diagram](./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/fork-exec.png)

```sh
strace -f python3 -c "import subprocess; print(subprocess.check_output(['ls']).decode('utf-8'))" 2>&1 | grep "clone\|fork\|exec\|wait"
```

## Pivoting into a new filesystem

While we end up with an isolated process at the moment, I guess many will argue that this isn't a container just yet. And that's indeed the case it's still missing some importanting features like an isolated filesystem.

What is needed to run a chroot environment? Not that much, since something like this already works:

```sh
$ mkdir -p new-root/{bin,lib64}
$ cp /bin/bash new-root/bin
$ cp /lib64/{ld-linux-x86-64.so*,libc.so*,libdl.so.2,libreadline.so*,libtinfo.so*} new-root/lib64
$ sudo chroot new-root
```

We create a new root directory, copy a bash shell and its dependencies in and run `chroot`. This jail is pretty useless: All we have at hand is bash and its builtin functions like `cd` and `pwd`.
