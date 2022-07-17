---
title: "Best Practices for Deploying High Available Applications in Kubernetes"
date: 2022-07-17T16:23:44+03:00
type: "post"
showTableOfContents: true
tags: ["Kubernetes"]
---

It's not difficult to deploy an application in Kubernetes with minimal working configuration.
But when you want to provide your application with maximum availability and reliability, you will inevitably encounter a
considerable number of pitfalls. In this article, I tried to systematize and describe the most important rules for 
deploying highly available applications in Kubernetes.

Functionality that is not available in Kubernetes "out of the box" will hardly be affected here. Also, I won't be tied 
to specific CD solutions and will omit the issues of templating/generating Kubernetes manifests.

## 1. Number of replicas

It's hard to talk about any availability if the application doesn't work in at least 2 replicas. Why are there problems
when launching an application in 1 replica? Many Kubernetes objects (Node, Pod, ReplicaSet, etc.) can be automatically
deleted/recreated under certain conditions, so Kubernetes cluster and the applications running in it should be ready for
such situations.

For example, when autoscaling nodes down, some nodes along with the Pods running on them will be deleted. If your 
application is running in a single instance on the node being deleted at this time, application will be unavaileble for
some time. In general, when working in the same replica, any abnormal shutdown of the application will mean a downtime.
Thus, **the application must be running in at least 2 replicas**.

*The recommendations are relevant if HorizontalPodAutoscaler is not used. The best option for applications that will
have more than a few replicas is to configure HorizontalPodAutoscaler and forget about specifying the number of replicas
manually.*

## 2. Uniform distribution of replicas by nodes

It's very important to distribute application Pods to different nodes if the application is running in multiple
replicas. To do this, **recommend the scheduler not to run multiple Pods of the same Deployment on the same node**:

```yaml
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: testapp
              topologyKey: kubernetes.io/hostname
```


Use `preferredDuringScheduling` instead of `requiredDuringScheduling`, which may make it impossible to launch new Pods
if there are fewer available nodes than the new Pods require. However, `requiredDuringScheduling` can be useful when the
number of nodes and replicas of the application is precisely known and you need to be sure that two Pods won't be run on
the same node.

## 3. PriorityClass

`priorityClassName` affects which Pods will be scheduled in the first place, as well as which Pods can be evicted by the
scheduler if there's no space left for new Pods on the nodes.

You'll need to create several resources of the
[PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass) type and
associate them with the Pods via `priorityClassName`. A set of PriorityClass may look something like this:

| Name | Value | Description |
|------|:-------:|-------------|
| **Cluster** | 20000 | Critical components for functioning of the cluster, such as kube-api server |
| **Daemonsets** | 10000 | Usually we want DaemonSet's Pods not to be evicted by regular applications |
| **Production-high** | 9000 | Stateful apps |
| **Production-medium** | 8000 | Stateless apps |
| **Default** | 7000 | Non-production apps |

## 4. Update strategy

TODO

## 5. Stopping processes in containers

TODO

## 6. Probes

TODO

### 6.1 Liveness probe

TODO

### 6.2 Readiness probe

TODO

### 6.3 Startup probe

TODO
