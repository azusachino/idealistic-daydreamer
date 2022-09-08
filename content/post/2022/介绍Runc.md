---
title: Brief Introduction about runc
description: My current status of learning Containerization
date: 2022-01-16
slug: introduction-runc
image: img/2022/01/KjellHenriksen.jpg
categories: [Conclusion]
tags: [Conclusion, Runc]
---

其实，对于底层 Linux 的 syscall 还是不甚了解的；暂时只能从一个比较高的层面去看 runc 是怎么创建并运行一个 Container 的。

## 概念定义

[runC](https://github.com/opencontainers/runc)是一个遵循 OCI 标准的用来运行容器的命令行工具，也是一个 [Runtime Spec](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md) 的实现。

## 开始使用

### 创建一个 OCI Bundle

`OCI Bundle` 是指满足 OCI 标准的一系列文件，这些文件包含了运行容器所需要的所有数据，存放在一个共同的目录；该目录包含以下两项内容：

1. `config.json`：容器的运行时配置
2. `rootfs`: 容器的 filesystem

我们可以通过 `docker export` 或者 `podman export` 将已有镜像导出为 `OCI Bundle` 的格式。

```sh
# create the bundle folder
mkdir ~/demo
cd ~/demo

# create the root filesystem
mkdir rootfs
podman export $(podman create busybox) | tar -C rootfs -xvf -
```

### 运行 Container

```sh
# create the config.json
runc spec

# run as root (change config.json#process#args to `"sleep", "5"` for better interaction)
runc create demo
runc start demo

# check container status
runc ps

# delete the container
runc delete demo
```

## `runc run` 的大致流程

1. 读取 `config.json` 文件，作为 Container 的 `RuntimeSpec` 配置来源
2. 建立 Socket，作为 runc 和 Container 之间的交互桥梁
3. 创建 Container (linuxContainer)
   1. 创建 libcontainer 所需的配置
   2. 创建 cgroup 需要的一些内容
4. 绑定 Container 和 Socket
5. 启动 Container
   1. 检查 terminal 是否可用
   2. 创建新的 Process
   3. 创建 handler，以接受 setupIO 后的回调
   4. setupIO，生成 tty 对象
   5. 启动真正的 Container 对象
      1. 创建 Exec FIFO 管道
      2. 创建父进程 (parentProcess)
      3. 启动父进程
      4. 读取 Exec FIFO 管道中的内容
   6. TTY 相关交互
6. Destroy 相关操作

**源码中的主要流程如下**:

```go
main.go#runCommand
├── run.go#Action
│   │── utils_linux.go#startContainer(*cli.Context, CtAct, *libcontainer.CriuOpts) (int, error)
│   │   │── utils.go#revisePidFile(*cli.Context) error
│   │   │── utils.go#setupSpec(*cli.Context) (*specs.Spec, error)
│   │   │    └── spec.go#loadSpec(string) (*specs.Spec, error)
│   │   │── notify_socket.go#newNotifySocket(*cli.Context, string, string) *notifySocket
│   │   │── notify_socket.go#setupSpec(*specs.Spec)
│   │   │── utils_linux.go#createContainer(*cli.Context, string, *specs.Spec) (libcontainer.Container, error)
│   │   │   │── rootless_linux.go#shouldUseRootlessCgroupManager(*cli.Context) (bool, error)
│   │   │   │── spec_linux.go#CreateLibcontainerConfig(*CreateOpts) (*configs.Config, error)
│   │   │   │   │── spec_linux.go#createLibcontainerMount(string, specs.Mount) (*configs.Mount, error)
│   │   │   │   │── spec_linux.go#createDevices(*specs.Spec, *configs.Config) ([]*devices.Device, error)
│   │   │   │   │── spec_linux.go#CreateCgroupConfig(*CreateOpts, []*devices.Device) (*configs.Cgroup, error)
│   │   │   │   │── spec_linux.go#initMaps()  // namespaces, mountPropagation, mountFlags, recAttrFlags
│   │   │   │   │── spec_linux.go#setupUserNamespace(*specs.Spec, *configs.Config) error
│   │   │   │   │── spec_linux.go#SetupSeccomp(*specs.LinuxSeccomp) (*configs.Seccomp, error)
│   │   │   │   └── spec_linux.go#createHooks(rspec *specs.Spec, config *configs.Config)
│   │   │   │── utils_linux.go#loadFactory(*cli.Context) (libcontainer.Factory, error)
│   │   │   │   └── factory_linux.go#New(string, ...func(*LinuxFactory) error) (Factory, error)
│   │   │   │── factory_linux.go#Create(string, *configs.Config) (Container, error)
│   │   │   │   │── factory_linux.go#validateId(string) (error)
│   │   │   │   │── configs/validate/validator.go#Validate(*configs.Config) (error)
│   │   │   │   │── cgroups/manager/new.go#NewWithPaths(config *configs.Cgroup, paths map[string]string) (cgroups.Manager, error)
│   │   │   │   │── cgroups/fs2/fs2.go#GetFreezerState() (config.FreezerState, error)
│   │   │   │   └── factory_linux.go#&linuxContainer{} // create linuxContainer as Container implementation
│   │   │── notify_socket.go#setupSocketDirectory()
│   │   │── notify_socket.go#bindSocket()
│   │   │── utils_linux.go#run(*specs.Process) (int, error)
│   │   │   │── utils_linux.go#defer *runner.destroy()
│   │   │   │── utils_linux.go#checkTerminal(*spec.Process) error
│   │   │   │── utils_linux.go#newProcess(specs.Process) (*libcontainer.Process, error)
│   │   │   │── signals.go#newSignalHandler(bool, *notifySocket) *signalHandler
│   │   │   │── utils_linux.go#setupIO(*libcontainer.Process,int, int, bool, bool, string) (*tty, error)
│   │   │   │── container_linux.go#Run(*process.Process) error
│   │   │   │   │── container_linux.go#Start()
│   │   │   │   │   │── container_linux.go#createExecFifo() error
│   │   │   │   │   │── container_linux.go#start(process *Process) (retErr error)
│   │   │   │   │   │   │── container_linux.go#newParentProcess(p *Process) (parentProcess, error)
│   │   │   │   │   │   │── process_linux.go#forwardChildLogs() chan error
│   │   │   │   │   │   │── process_linux.go#start() (retErr error) // runc init
│   │   │   │   │   │   │── container_linux.go#currentOCIState() (*specs.State, error)
│   │   │   │   │   │   └── container_linux.go#ignoreTerminateErrors(err error) error
│   │   │   │   │   │── container_linux.go#deleteExecFifo()
│   │   │   │   │── container_linux.go#exec()
│   │   │   │   │   │── container_linux.go#awaitFifoOpen(path string) <-chan openResult
│   │   │   │   │   └── container_linux.go#handleFifoResult(result openResult) error
│   │   │   │── tty.go#waitConsole() error
│   │   │   │── tty.go#ClosePostStart(*spec.Process) error
│   │   │   │── utils_linux.go#createPidFile(path string, process *libcontainer.Process) error
│   │   │   └── signals.go#forward(process *libcontainer.Process, tty *tty, detach bool) (int, error)
```

## config.json

此次展示 runc 生成的默认`config.json`，以供后续参考查看。

```json
{
  "ociVersion": "1.0.2-dev",
  "process": {
    "terminal": true,
    "user": {
      "uid": 0,
      "gid": 0
    },
    "args": ["sh"],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ],
    "cwd": "/",
    "capabilities": {
      "bounding": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "effective": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "inheritable": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "permitted": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "ambient": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"]
    },
    "rlimits": [
      {
        "type": "RLIMIT_NOFILE",
        "hard": 1024,
        "soft": 1024
      }
    ],
    "noNewPrivileges": true
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "runc",
  "mounts": [
    {
      "destination": "/proc",
      "type": "proc",
      "source": "proc"
    },
    {
      "destination": "/dev",
      "type": "tmpfs",
      "source": "tmpfs",
      "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]
    },
    {
      "destination": "/dev/pts",
      "type": "devpts",
      "source": "devpts",
      "options": [
        "nosuid",
        "noexec",
        "newinstance",
        "ptmxmode=0666",
        "mode=0620",
        "gid=5"
      ]
    },
    {
      "destination": "/dev/shm",
      "type": "tmpfs",
      "source": "shm",
      "options": ["nosuid", "noexec", "nodev", "mode=1777", "size=65536k"]
    },
    {
      "destination": "/dev/mqueue",
      "type": "mqueue",
      "source": "mqueue",
      "options": ["nosuid", "noexec", "nodev"]
    },
    {
      "destination": "/sys",
      "type": "sysfs",
      "source": "sysfs",
      "options": ["nosuid", "noexec", "nodev", "ro"]
    },
    {
      "destination": "/sys/fs/cgroup",
      "type": "cgroup",
      "source": "cgroup",
      "options": ["nosuid", "noexec", "nodev", "relatime", "ro"]
    }
  ],
  "linux": {
    "resources": {
      "devices": [
        {
          "allow": false,
          "access": "rwm"
        }
      ]
    },
    "namespaces": [
      {
        "type": "pid"
      },
      {
        "type": "network"
      },
      {
        "type": "ipc"
      },
      {
        "type": "uts"
      },
      {
        "type": "mount"
      }
    ],
    "maskedPaths": [
      "/proc/acpi",
      "/proc/asound",
      "/proc/kcore",
      "/proc/keys",
      "/proc/latency_stats",
      "/proc/timer_list",
      "/proc/timer_stats",
      "/proc/sched_debug",
      "/sys/firmware",
      "/proc/scsi"
    ],
    "readonlyPaths": [
      "/proc/bus",
      "/proc/fs",
      "/proc/irq",
      "/proc/sys",
      "/proc/sysrq-trigger"
    ]
  }
}
```

## 总结

学习结果不是很理想，还需继续精进。

## References

- [runC(上)](https://switch-router.gitee.io/blog/runc-1/)
- [runC(下)](https://switch-router.gitee.io/blog/runc-2/)
