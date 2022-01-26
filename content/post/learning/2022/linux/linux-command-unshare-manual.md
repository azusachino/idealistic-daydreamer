---
title: Unshare Manual
description: The standpoint of rootless container
date: 2022-01-26
slug: linux-command-unshare-manual
image: img/2022/01/MeotoIwa.jpg
categories:
  - learning
tags:
  - linux
---

## Definition

unshare - run program in new namespaces

## Usage

`unshare [options] [program [arguments]]`

## Description

The `unshare` command creates new namespaces and then executes the specified program. If program is not given, then `${SHELL}` is run (default: `/bin/sh`).

By default, a new namespace persists only as long as it has member processes. A new namespace can be made persistent even when it has no member processes by bind mounting `/proc/pid/ns/type` files to a filesystem path. A namespace that has been made persistent in this way can subsequently be entered with `nsenter` even after the program terminates (except `PID namespaces` where a permanently running init process is required). Once a persistent namespace is no longer needed, it can be un-persisted by using `unmount` to remove the bind mount.

## Namespaces (can be created with unshare)

### mount namespace

Mounting and unmounting file systems will not affect the rest of the system, except for file systems which are explicitly marked as shared (with `mount --make-shared`)

### UTS namespace

Setting hostname or domain name will not affect the rest of the system.

### IPC namespace

The process will have an independent namespace for POSIX message queues as well as System V message queues, semaphore sets and shared memory segments.

### network namespace

The process will have independent IPv4 and IPv6 stacks, IP routing tables, firewall rules, the `/proc/net` and `/sys/class/net` directory trees, sockets, etc.

### PID namespace

Children will have a distinct set of PID-to-process mappings from their parent.

### cgroup namespace

The process will have a virtualized view of `/proc/self/cgroup`, and new cgroup mounts will be rooted at the namespace cgroup root.

### user namespace

The process will have a distinct set of UIDs, GIDs and capabilities. (Important for Rootless Container)

### time namespace

The process will have a distinct view of `CLOCK_MONOTONIC` and/or `CLOCK_BOOTTIME` which can be changed using `/proc/self/timens_offsets`.

## Options

If file is specified, then a persistent namespace is created by a bind mount.

```s
-i, --ipc[=file]
    Unshare the IPC namespace.

-m, --mount[=file]
    Unshare the mount namespace.

-n, --net[=file]
    Unshare the network namespace.

-p, --pid[=file]
    Unshare the PID namespace.
    (Creation of a persistent PID namespace
    will fail if the --fork option is not also specified.)

-u, --uts[=file]
    Unshare the UTS namespace.

-U, --user[=file]
    Unshare the user namespace.

-C, --cgroup[=file]
    Unshare the cgroup namespace.

-T, --time[=file]
    Unshare the time namespace. The --monotonic
    and --boottime options can be used to specify the
    corresponding offset in the time namespace.

-f, --fork
    Fork the specified program as a child process of unshare
    rather than running it directly. This is useful when creating
    a new PID namespace. Note that when unshare is waiting for
    the child process, then it ignores SIGINT and SIGTERM and
    does not forward any signals to the child. It is necessary to
    send signals to the child process.

--keep-caps
    When the --user option is given, ensure that capabilities
    granted in the user namespace are preserved in the child
    process.

--kill-child[=signame]
    When unshare terminates, have signame be sent to the forked
    child process. Combined with --pid this allows for an easy
    and reliable killing of the entire process tree below
    unshare. If not given, signame defaults to SIGKILL. This
    option implies --fork.

--mount-proc[=mountpoint]
    Just before running the program, mount the proc filesystem at
    mountpoint (default is /proc). This is useful when creating a
    new PID namespace. It also implies creating a new mount
    namespace since the /proc mount would otherwise mess up
    existing programs on the system. The new proc filesystem is
    explicitly mounted as private (with MS_PRIVATE|MS_REC).

--map-user=uid|name
    Run the program only after the current effective user ID has
    been mapped to uid. If this option is specified multiple
    times, the last occurrence takes precedence. This option
    implies --user.

--map-group=gid|name
    Run the program only after the current effective group ID has
    been mapped to gid. If this option is specified multiple
    times, the last occurrence takes precedence. This option
    implies --setgroups=deny and --user.

-r, --map-root-user
    Run the program only after the current effective user and
    group IDs have been mapped to the superuser UID and GID in
    the newly created user namespace. This makes it possible to
    conveniently gain capabilities needed to manage various
    aspects of the newly created namespaces (such as configuring
    interfaces in the network namespace or mounting filesystems
    in the mount namespace) even when run unprivileged.
    This option implies --setgroups=deny and --user.
    This option is equivalent to --map-user=0 --map-group=0.

-c, --map-current-user
    Run the program only after the current effective user and
    group IDs have been mapped to the same UID and GID in the
    newly created user namespace. This option implies
    --setgroups=deny and --user. This option is equivalent to
    --map-user=$(id -ru) --map-group=$(id -rg).

--propagation private|shared|slave|unchanged
    Recursively set the mount propagation flag in the new mount
    namespace. The default is to set the propagation to private.
    It is possible to disable this feature with the argument
    unchanged. The option is silently ignored when the mount
    namespace (--mount) is not requested.

--setgroups allow|deny
    Allow or deny the setgroups(2) system call in a user
    namespace.

-R, --root=dir
    run the command with root directory set to dir.

-w, --wd=dir
    change working directory to dir.

-S, --setuid uid
    Set the user ID which will be used in the entered namespace.

-G, --setgid gid
    Set the group ID which will be used in the entered namespace
    and drop supplementary groups.

--monotonic offset
    Set the offset of CLOCK_MONOTONIC which will be used in the
    entered time namespace. This option requires unsharing a time
    namespace with --time.

--boottime offset
    Set the offset of CLOCK_BOOTTIME which will be used in the
    entered time namespace. This option requires unsharing a time
    namespace with --time.

-V, --version
    Display version information and exit.

-h, --help
    Display help text and exit.
```

## Practice

### PID

readlink - print resolved symbolic links or canonical file names

```sh
> unshare --fork --pid --mount-proc readlink /proc/self
1
```

### user

As an unprivileged user, create a new user namespace where the user's credentials are mapped to the root IDs inside the namespace:

```sh
> id -u; id -g
1000
1000
> unshare --user --map-root-user \
        sh -c ''whoami; cat /proc/self/uid_map /proc/self/gid_map''
root
    0    1000    1
    0    1000    1
```

### UTS

nsenter - run program in different namespaces (`nsenter [options] [program [arguments]]`)

```shell
> touch /root/uts-ns
> unshare --uts=/root/uts-ns hostname FOO
> nsenter --uts=/root/uts-ns hostname
FOO
> unmount /root/uts-ns
```

### mount

```sh
> mount --bind /root/namespaces /root/namespaces
> mount --make-private /root/namespaces
> touch /root/namespaces/mnt
> unshare --mount=/root/namespaces/mnt
```

## Reference

- [unshare(1) - Linux manual page](https://man7.org/linux/man-pages/man1/unshare.1.html)
