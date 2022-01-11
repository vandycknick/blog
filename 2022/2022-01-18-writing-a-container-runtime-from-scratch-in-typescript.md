---
id: ea2370bd-e1d1-442d-a8c9-a891bc4baa76
title: "Add styling to an active link in Next.js"
description: 
date: 2022-01-18T20:00:00+01:00
categories: [scratch]
cover: ./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/cover.jpg
---

Containerization, especially Docker, has become immensely popular over the last couple of years. Even though Docker isn't unique or the first of its kind, it quickly became the developer favourite thanks to its fantastic developer experience. Spawning a new shell with Docker is child's play, a quick `docker run -it ubuntu bash` gives access to a shell in a containerized ubuntu environment in no time. Even wrapping up your personal website or blog, ... in an image and shipping it up to production can be with just a few lines on the command line. It has become so popular right now that whole ecosystems have spun up around it, Kubernetes or Docker Swarm for orchestration, podman, ebpf flavoured networking components and the list goes on. It's mind-boggling how many tools are out there and how amazing this community and ecosystem is. But have you ever dared to take a step back and wondered how containers work under the hood? Have you ever asked yourself what components make up a container? Have you ever gone hunting down the Linux source code to find that one magical API call that creates a new container? Well, I have done all of that, and to save you some time in this post, I want to go deep into the nitty-gritty details and show you how the sausage is made by creating a little containerization engine from scratch in TypeScript.

## Setting the scene
But before we can get our hands dirty, there are a couple of things I would like to get out of the way first. If you already are a seasoned containerization specialist or only care about the code, feel free to jump right into the next section. If not, let's get through some of the basics first and make sure we are all on the same page. Throughout the blog post, I'm mainly going to look at things from a Docker perspective, but the core concepts basically apply to any container runtime out there. Using Docker for this exercise makes things easier giving most people will have some familiarity with it.

### What is docker?

Often thought of as cheap VMs, containers are just isolated groups of processes running on a single host. That isolation leverages several underlying technologies built into the Linux kernel: namespaces, cgroups, chroots, capabilities and lots of terms you've probably heard before. It basically boils down to these six things that allow us to create isolated processes:

1. Namespaces
2. Chroot/Pivot Root
3. Copy On Write
4. CGroups
5. Capabilities
6. Seccomp
7. SELinux / AppArmor

![tweak this image](./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/docker-linux-interfaces.png)

###

## Getting ready for development

AAll the code is available at https://github.com/vandycknick/runjs, which has a couple of branches one for each of the following sections. Hopefully, this makes it easier to follow along or start from a working state for each section if you get stuck. The runtime is called runjs and it's a little wordplay on `runc`, yes I know the c actually stands for container, still I thought it was funny.

If you want to follow along the root of the repo contains a `Vagrantfile`. Don't worry if you don't know Vagrant or have never heard about it before. In essence, Vagrant is similar to Docker, but instead of creating containers with Vagrant, you end up with full-blown virtual machines. You can easily get it installed from [here](https://www.vagrantup.com/), and a simple `vagrant up` followed by a `vagrant ssh` allows you to follow along with the rest of the article without issues. This will give you access to a Rocky Linux VM with Docker, deno and a few extra tools preinstalled that we'll need along the way. If you prefer to set up your own VM, feel free to do so, but do know that the rest of the article is tested and known to be working on this preconfigured machine. Just make sure you don't run this on your main OS, while 99% of the code we'll write is rather harmless, there is a part where if you do make a mistake, it can lead to data loss. I'll definitely mention it when we reach that part, but running this in a VM also ensures you don't clutter your main OS.

## Shielding of our process with namespaces

## Pivoting into a new filesystem
