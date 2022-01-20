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

```ts
const run = (args: string[]) => {
  console.log("run", args);
  return 0;
};
```

It get's called by the main function when the program executes and passes the all command line arguments down as a string array. Executing this script with Deno should print run and all arguments passed via the command line.

```sh
$ deno run ./src/main.ts sh
run [ "sh" ]

$ deno run ./src/main.ts sh
run [ "ps", "aux" ]
```

I highly recommend running any of the following code snippets in a Virtual machine. While 99% of the code we'll write is rather harmless, there is a small section in this post that could cause permanent data loss if you manage to get a few wires crossed. Thats exactly why the `Vagrantfile` is there, it allows me to provide a safe pretested environment. Don't worry if you don't know Vagrant or have never heard about it before. In essence, Vagrant is similar to Docker, but instead of creating containers with Vagrant you end up with full-blown virtual machines. You can easily get it installed from [here](https://www.vagrantup.com/), and a simple `vagrant up` followed by a `vagrant ssh` allows you to follow along with the rest of the article without issues.

## Shielding of a process with namespaces

Namespaces are the Linux kernel feature that enables the containerization of a process. First introduced back in 2002 with Linux 2.4.19, the idea behind a namespace is to wrap specific global system resources in an abstraction layer. This abstraction layer makes it appear as if processes within a namespace have their own isolated set of resources. Different process groups can have different sets of namespaces applied to them. Initially only the mount namespaces existed but over time more of them got added to the kernel and at the moment there are seven distinct namespaces available:

| Namespace | Flag            | Page                   | Isolates                             |
| --------- | --------------- | ---------------------- | ------------------------------------ |
| UTS       | CLONE_NEWUTS    | man uts_namespaces     | Hostname and NIS domain name         |
| Mount     | CLONE_NEWNS     | man mount_namespaces   | Mount points                         |
| PID       | CLONE_NEWPID    | man pid_namespaces     | Process IDs                          |
| Network   | CLONE_NEWNET    | man network_namespaces | Network devices, stacks, ports, etc. |
| IPC       | CLONE_NEWIPC    | man ipc_namespaces     | System V IPC, POSIX message queues   |
| User      | CLONE_NEWUSER   | man user_namespaces    | User and group IDs                   |
| Cgroup    | CLONE_NEWCGROUP | man cgroup_namespaces  | Cgroup root directory                |

Don't worry about these just yet; we'll discuss them in more detail once we start writing some code. But before we can go any further, we'll need a common understand about how processes are created in the Linux kernel. If you've ever programmed in python before, you probably know that you can create a new process by calling `subprocess.Popen` or any of its many derivatives. Many other programming languages offer a similar API like `exec.Command` in golang, `Process.Start` in dotnet or `child_process.spawn` in nodejs. While all these languages offer different API calls they all end up making the same syscalls in the Linux kernel named `fork`, `clone` and `execve`. While technically `fork` isn't actually a syscall but rather a wrapper around `clone`. For simplicity we are going to treat it as such, mainly because you will often find it referenced as a syscall online but it's the easier one to use and implement in Deno. Using these API's to create a new process looks something like this:

![fork-exec-syscall-diagram](./assets/2022-01-18-writing-a-container-runtime-from-scratch-in-typescript/fork-exec.png)

As you might have noticed, the calling process is often referred to as the parent process, whereas the newly created process is often called the child process. A parent will call `fork` or `clone` to create a new process, making an exact copy itself and continuing execution. The key point to understanding `fork` is to realize that two processes exist after it has completed its work, and execution continues from the point where `fork` returns. The return value allows us to determine whether after the fork we are continuing as the child or parent process. We can then use the `wait` syscall in the parent process to block execution and wait for the child process to exit. A child process often calls out into `execve`, which loads a new program into a process's memory. All of this might still sound very abstract at this point. But let's make this a bit more concrete by recording all syscalls while creating a new process in Python. The following one-liner Python script will run the `ls` command and print the output of the command to stdout `import subprocess; print(subprocess.check_output(['ls']).decode('utf-8'))`. With `strace` we can look under the covers and see what API calls Python calls out to create a new process:

```sh
$ strace -f python3 -c "import subprocess; print(subprocess.check_output(['ls']).decode('utf-8'))" 2>&1 | grep "clone\|fork\|exec\|wait"

execve("/bin/python3", ["python3", "-c", "import subprocess; print(subproc"...], 0x7fff592b3688 /* 37 vars */) = 0
<...>
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f3ec675d3d0) = 4411
<...>
[pid  4411] execve("/opt/deno/bin/ls", ["ls"], 0x55ce62f4c3e0 /* 38 vars */) = -1 ENOENT (No such file or directory)
[pid  4411] execve("/opt/deno/bin/ls", ["ls"], 0x55ce62f4c3e0 /* 38 vars */) = -1 ENOENT (No such file or directory)
[pid  4411] execve("/usr/local/sbin/ls", ["ls"], 0x55ce62f4c3e0 /* 38 vars */) = -1 ENOENT (No such file or directory)
[pid  4411] execve("/sbin/ls", ["ls"], 0x55ce62f4c3e0 /* 38 vars */) = -1 ENOENT (No such file or directory)
[pid  4411] execve("/bin/ls", ["ls"], 0x55ce62f4c3e0 /* 38 vars */ <unfinished ...>
[pid  4411] <... execve resumed>)       = 0
[pid  4410] wait4(4411,  <unfinished ...>
<... wait4 resumed>[{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 4411
```

Prefixing a command with `strace` will record all syscalls it makes during execution and print one line for each syscall with argument information to stderr. We'll need to send stderr to stdout with some bash redirection (`2>&1`) to filter the output and make it more digestible. In the example above, you can see the first call to `execve` is starting the Python process itself. This is because when executing this command from bash it will have called fork and the child process it created will then start up Python. `strace` is only able to trace syscalls from the child process, hence why we don't see the fork or clone call before the trace started. After that point, the child process gets replaced by the program passed to execve, and Python will be running. We then see Python calling out to `clone` and creating a new process with ID 4411. As mentioned earlier, `fork` is just a wrapper around `clone`, hence why we'll only find traces of the `clone` syscall in the recordings. We can then see multiple `execve` calls all for the `ls` binary. `execve` requires passing an absolute path and we see Python looking for `ls` for every path in the `PATH` environment variable. Eventually, it finds `ls` at `/bin/ls` and executes the program. The parent process with ID 4410 calls `wait4` and blocks execution until the child process (4411) exits, after which we see execution resuming in the parent process.

Ok, enough talk, let's start writing some code. As mentioned a few times already, we'll need to call `fork`, which is exported from the libc module in `libc/mod.ts`. It will return a number that allows us to determine if we are continuing in the parent, child or if the fork failed. If it returns `-1` the `fork` failed, we'll immediately stop execution by throwing an error saying we could not create a new child process. If the return value is 0, we are running as the child process. When running in the child process, we'll call the `container` function and afterwards immediately call `Deno.exit` to make sure we halt execution. If `fork` returns any other positive return value will be the child process ID indicating we are running in the parent process. In the parent process, we'll call the `waitPid` function and pass the child process ID returned from the `fork` syscall. This will block execution until the child process is finished and return the status code from the child process, which we'll use to determine whether it ran successfully or not.

```ts
import { fork, waitPid } from "../libc/mod.ts";

const run = (args: string[]) => {
  const pid = fork();

  if (pid === -1) throw new Error("Failed to create a new process");

  // Running as the child process
  if (pid === 0) {
    console.log("child", pid, args);
    container(args);
    Deno.exit(0);
  }

  // Running as the parent process
  // pid === to the process ID of the child process
  console.log("parent", pid, args);
  const status = waitPid(pid);
  return status.code;
};

const container = (args: string[]) => {};
```

Executing this script will yield the following result:

```sh
$ deno run -A --unstable ./src/main.ts sh
Check file:///vagrant/src/main.ts
parent 5845 [ "sh" ]
child 0 [ "sh" ]

$ deno run -A --unstable ./src/main.ts sh
parent 5850 [ "sh" ]
child 0 [ "sh" ]
```

The example makes it look like the parent always executes before the child process. But that's not always the case; it's indeterminate which process - the parent or the child - has access to the CPU next. On a multiprocessor system, they could simultaneously access the CPU. In our case, this won't matter but do know that relying on a specific execution order could result in subtle and hard to debug race conditions. But as you can see, we're moving in the right direction. We just need a few more things in place to mimic the `docker run' command. When executing `deno run -A --unstable ./src/main/ts ps aux`, we want the child process to start running the `ps`program and use`aux`as arguments. For this, we can use`exec`, which has the following type definition:

```ts
exec(fileName: string, args: string[], env?: { [key: string]: string; } | undefined) => never`
```

One of the first arguments takes a fileName. This can be a relative or an absolute path. If it's a relative path, the function will look in the `PATH` variable to find the binary. If it can't find the program available on your `PATH` it will stop execution and throw an error. The second argument is a string array that represents the list of arguments. The last one is an optional object that allows us to tweak the programs environment variables. We won't need this one just yet; in doing so, the child process will just inherit all environment variables. Since it replaces the program that called it, a successful `exec` never returns.

```ts
import { fork, exec, waitPid } from "../libc/mod.ts";

const run = (args: string[]) => {
  const pid = fork();

  if (pid === -1) throw new Error("Failed to create a new process");

  // Running as the child process
  if (pid === 0) {
    try {
      container(args);
    } catch (ex) {
      console.error(ex);
      Deno.exit(1);
    }
  }

  // Running as the parent process
  // pid === to the process ID of the child process
  const status = waitPid(pid);
  return status.code;
};

const container = (args: string[]): never => {
  const [program, ...restArgs] = args;
  return exec(program, restArgs);
};
```

Once again, we'll import `exec` from the `libc` module. We'll also remove the console logs we put in just earlier to reduce the noise. In the `container` function, we'll destructure the args array; the first argument will be the program to execute, the rest will be passed as program arguments. We'll call exec and indicate that the `container` function never returns by adding the `never` return type. Given exec only returns to the caller if an error occurs, we'll wrap it up in a `try/catch` clause and log any errors to stderr. If an error does occur, we'll use `Deno.exit()` with exit code of 1 to stop the child process and indicate we terminated abruptly. Let's give it a try:

```sh
$ deno run -A --unstable ./src/main.ts sh
sh-4.4# hostname
rocky
sh-4.4# ls
Makefile  README.md  Vagrantfile  libc  runjs  scripts  src
sh-4.4# pwd
/vagrant
```

Executing our mini container engine now will start a new `sh` session and present us with a shell prompt. The output of any command we run gets printed back to the console we currently have open. This is because `exec` inherits stdio file descriptors of the parent process by default. Obviously, that might not always be what you want, but this will work just fine for now. Actual container engines like docker will change these file descriptors so they can capture output to file. This makes it possible to call `docker logs` on a running container and view everything printed to stdout. From the `waitPid` function, we get the exit code of the child process. Because of this, we can also play around with returning different exit codes from the containerized `sh` session and see them getting reflected back in bash when our container engine exits:

```sh
$ deno run -A --unstable ./src/main.ts sh
sh-4.4# exit 111
exit

$ echo $?
111
```

> The status code returned from waitPid is truncated to an 8bit value, meaning the max return value is 255. Any higher value will get wrapped around.

Yeah, I know that was a lot of stuff to go through before we can start looking at namespaces. It had to be done, but now we are finally ready to move our process in separate namespaces. For now, we'll just focus on a subset of them; we'll dig into the other when it makes sense. We start off with the UTS or Unix time-sharing System namespace. First introduced in Linux 2.6.19 (2006), it allows us to unshare the domain- and hostname from the current host system. The `unshare()` syscall exported from the `libc` module will enable us to disassociate parts of our containers execution context from the parent process:

```ts
unshare(flags: number): void
```

The flags argument is a bitmask that specifies which parts should be unshared and can be combined to apply multiple namespaces. It's just a void function that doesn't return any meaningful value. To decouple our container from the UTS namespace, all we need to do is import the `CLONE_NEWUTS` and call `unshare()` right before we `fork()` our container process:

```ts
import {
  fork,
  exec,
  waitPid,
  unshare,
  CLONE_NEWUTS,
  setHostname,
} from "../libc/mod.ts";

const run = (args: string[]) => {
  unshare(CLONE_NEWUTS);

  const pid = fork();

  if (pid === -1) throw new Error("Failed to create a new process");

  // Running as the child process
  if (pid === 0) {
    try {
      container(args);
    } catch (ex) {
      console.error(ex);
      Deno.exit(1);
    }
  }

  // Running as the parent process
  // pid === to the process ID of the child process
  const status = waitPid(pid);
  return status.code;
};

const container = (args: string[]): never => {
  setHostname("container");

  const [program, ...restArgs] = args;
  return exec(program, restArgs);
};
```

Let's also change the hostname to a different value right before calling `exec()`. In the example above, I got very original and changed it to `container`; feel free to change it to something more exciting if you feel like it. Let's give it a try:

```sh
$ deno run -A --unstable ./src/main.ts sh
Check file:///vagrant/src/main.ts

sh-4.4# hostname
container

sh-4.4# hostname new-hostname

sh-4.4# hostname
new-hostname
```

And when we exit out of the container we can see that at the host level nothing has changed, hooray:

```sh
$ hostname
rocky
```

Next up we have the IPC namespace also introduced in LInux 2.6.19 (2006) to isolate interprocess communication resources. It affects System V IPC objects and POSIX message queues or can be used to separate shared memory (SHM) between two processes to avoid misusage. When an IPC namespace is destroyed then all IPC objects within the namespace automatically get destroyed, too. To add it to our program we just need to OR it with the uts namespace flag:

```ts
...
const run = (args: string[]) => {
  unshare(CLONE_NEWUTS | CLONE_NEWIPC);

  const pid = fork();
  ...
```

The PID namespace was introduced in Linux 2.6.24 (2008) and gives processes an independent set of process identifiers (PIDs). This means that processes which reside in different namespaces can own the same PID. In the end a process has two PIDs: the PID inside the namespace, and the PID outside the namespace on the host system. The PID namespaces can be nested, so if a new process is created it will have a PID for each namespace from its current namespace up to the initial PID namespace.

The first process created in a PID namespace gets the number 1 and gains all the same special treatment as the usual init process. For example, all processes within the namespace will be re-parented to the namespace’s PID 1 rather than the host PID 1. In addition the termination of this process will immediately terminate all processes in its PID namespace and any descendants. Let’s create a new PID namespace:

```ts
...
const run = (args: string[]) => {
  unshare(CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID);

  const pid = fork();
  ...
```

When we run execute our container runtime now we can see that the `sh` process is running as pid 1:

```sh
$ deno run -A --unstable ./src/main.ts sh
sh-4.4# echo $$
1
```

Looks isolated, doesn't it? Not exactly, when we run `ps aux`, we can still get an overview of all processes running on the system. This happens because of something called the proc filesystem, which is mounted at `/proc`. `procfs` is a pseudo filesystem which is just Linux speak for this thing is special. It's the kernel's way of exposing process or system-related information back to user space in a file like manner. The problem here is that we unshared the PID namespace but left the host `/proc` filesystem mounted in the container. If we want to isolate this, we'll need to remount this filesystem again. From within our container, we can do this by running `mount -t proc proc /proc` and that fixes our problem:

```sh
$ deno run -A --unstable ./src/main.ts sh

sh-4.4# mount -t proc proc /proc

sh-4.4# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1  23288  3536 pts/0    S    23:07   0:00 sh
root           4  0.0  0.1  55912  3620 pts/0    R+   23:07   0:00 ps aux
```

But with everything in computing, if you fix one problem, you immediately get presented with another one to fix. When we exit out of the container and run `ps` on the host, we'll get presented with `Error, do this: mount -t proc proc /proc`. Because we remounted `/proc` in the container we are now left with a "broken" `/proc` mount on the host. To fix this we just need to run `mount -t proc proc /proc` again on the host to fix our problem. We can easily put this into code and mount `/proc` before starting our container and then later mount `/proc` again when we exit. But we'll need to handle many edge cases and race conditions, or otherwise, our host will likewise end up with a broken `/proc` mount. There's a better way of fixing this, which is the perfect segway into the next chapter, where we'll dig into pivoting our container into a new filesystem!

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
