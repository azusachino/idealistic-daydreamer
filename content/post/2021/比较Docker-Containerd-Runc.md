---
title: 了解 Docker, Containerd, RunC
description: 自顶向下
date: 2021-12-26
slug: understand-docker-containerd-runc
image: img/2021/12/SiberianSunset.jpg
categories: [Conclusion]
tags: [Conclusion, Docker, Runc]
---

From the graph below, I think everyone could gain a good understanding about container-relate stuffs. Thanks tutorial works, please check reference.

![ ](img/2021/12/docker-relation.jpg)

Both Docker & Kubernetes are classical C/S architectures, e.g. (docker-cli -> docker-daemon, kubernetes-cli -> kube-apiserver)

![ ](img/2021/12/docker.png)

If you like, we can use runc bare in metal to run a container, also it's not recommended.

```bash
mkdir -p /mycontainer/rootfs
cd /mycontainer
# export busybox via podman into the rootfs directory
podman export $(podman create busybox) | tar -C rootfs -xvf -

# create spec
runc spec
runc run mycontainerid
```

## References

- [The differences between Docker, containerd, CRI-O and runc](https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/)
- [OpenContainers](https://github.com/opencontainers)
