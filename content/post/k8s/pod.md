---
title: Pod
description: Everything I know about Pod
date: 2021-09-30
slug: k8s/pod
image: img/2021/09/MackenzieRiver.jpg
categories:
  - k8s
  - source
---

Pod is the smallest unit in kubernetes.

## A tiny sample

```yml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    app: busybox
spec: # desired state
  containers:
    - image: busybox
      command:
        - sleep
        - "3600"
      imagePullPolicy: IfNotPresent
      name: busybox
  restartPolicy: Always
```

## `PodSpec` in source code

Everything that consists a pod is as followed, it's easy to understand 

```go
type PodSpec struct {
    Volumes []Volume // volume configuration
	InitContainers []Container // init containers (optional)
	Containers []Container // container configuration (necessary)
    // List of ephemeral containers run in this pod. Ephemeral containers may be run in an existing
	// pod to perform user-initiated actions such as debugging. This list cannot be specified when
	// creating a pod, and it cannot be modified by updating the pod spec. In order to add an
	// ephemeral container to an existing pod, use the pod's ephemeralcontainers subresource.
	// This field is alpha-level and is only honored by servers that enable the EphemeralContainers feature.
	EphemeralContainers []EphemeralContainer
	RestartPolicy RestartPolicy
	TerminationGracePeriodSeconds *int64
	ActiveDeadlineSeconds *int64
	DNSPolicy DNSPolicy
	NodeSelector map[string]string
	ServiceAccountName string
	AutomountServiceAccountToken *bool
	NodeName string
	SecurityContext *PodSecurityContext // SecurityContext holds pod-level security attributes and common container settings.
	ImagePullSecrets []LocalObjectReference
	Hostname string
	Subdomain string
	SetHostnameAsFQDN *bool // If true the pod's hostname will be configured as the pod's FQDN, rather than the leaf name (the default).
	Affinity *Affinity
	SchedulerName string
	Tolerations []Toleration
	HostAliases []HostAlias
	PriorityClassName string
	Priority *int32
	PreemptionPolicy *PreemptionPolicy
	DNSConfig *PodDNSConfig
	// A pod is ready when all its containers are ready AND
	// all conditions specified in the readiness gates have status equal to "True"
	ReadinessGates []PodReadinessGate
	RuntimeClassName *string
	Overhead ResourceList
	EnableServiceLinks *bool
	TopologySpreadConstraints []TopologySpreadConstraint
}
```

## Container

Everything Pod will have at least two containers, one is `k8s.gcr.io/pause`, another is user-defined, also we can define some init-containers to do some configuration before main container starts.

## Reference

- [详解 Kubernetes Pod 的实现原理](https://draveness.me/kubernetes-pod/)
