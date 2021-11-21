---
title: Kubernetes Control Plane
description:
date: 2021-09-30
slug: k8s/control-plane-theory
image: img/2021/09/MackenzieRiver.jpg
categories:
  - k8s
  - theory
---

My **Kubernetes Trip 04**, current kubernetes version: [v1.22]

## Definition

The control plane's components make global decisions about the cluster, detecting and responding to cluster events.

## Components

### etcd

Consistent and high available key-value store used as Kubernetes' backing store for all cluster data.

### kube-apiserver

The Api server is a component of the Kubernetes control plane that exposes the Kubernetes API.

### kube-controller manager

Control plane component that runs controller processes

### kube-scheduler

Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on.

Factors taken into account for scheduling decisions include: individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference and deadlines.

## Reference

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
