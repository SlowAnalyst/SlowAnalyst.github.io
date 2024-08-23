---
layout: post
title: "Analysis about Fractional GPU on Kubernetes"
summary: ""
author: dps0340
date: '2024-06-23 14:50:00 +0900'
category: ['learning']
thumbnail: /assets/img/posts/default.png
keywords: devops, kubernetes, gpu
usemathjax: false
permalink: /blog/Analysis-about-Fractional-GPU-on-Kubernetes
---


Contents
--------
1. Introduction
2. PCI Passthrough
3. Axiom of gpu-operator
4. Workarounds with gpu-operator
    1. Use GPU devices on node without fixed allocation
    2. Nvidia Time-Slicing
    3. Nvidia MIG
5. Workarounds without gpu-operator
    1. gpushare-scheduler-extender
    2. nvshare
7. Conclusion
0. References



## Introduction
본 글에서는 gpu-operator에서 구성할 수 있는 GPU Passthrough 방법에 대한 공리와 공리를 우회할 수 있는 방법에 대해 논하고, gpu-operator를 차용하거나 차용하지 않는 환경에서의 한계가 존재하는 대안들을 설명한다.

## PCI Passthrough
GPU 자원은 Host OS에 구성되어 있는 반면, Container의 경우 pivot_root, cgroup, namespace에 의해 격리되는 Process이다. GPU 자원은 pci에 의해 관리되므로 Container Runtime등을 통해 해당 pci Device를 연동해야 한다.

GPU 연동을 제공하는 컴포넌트로 Docker 환경에서는 nvidia-docker, nvidia container toolkit이 있고, Kubernetes에서는 gpu-operator가 있다. gpu-operator의 경우 기존 Nvidia에서 제공하는 GPU 관련 컴포넌트를 하나의 Helm Chart로 통합한 개념이며, 컴포넌트 설명의 경우 글의 목적을 벗어나므로 생략한다.

## Axiom of gpu-operator
gpu-operator의 공리이자 제약조건은 하나의 Pod이 고정적으로 하나의 GPU를 차지하는 것이다.

하나의 GPU가 단일 프로세스에 대해 Core / VRAM을 고정적으로 제공하는 구조가 아니므로 하나의 GPU가 여러 프로세스를 수행할 수 있지만, gpu-operator의 경우 Node [Extended Resource](https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/)를 통해 Pod가 고정적으로 사용할 수 있는 GPU를 정의하고, kube-scheduler에 의해 PodSpec 내 requests 및 limits에 대응되는 Resource를 할당할 수 있는 (allocatable - requests or limits) Node를 찾아 Pod를 배치하는 구조를 통해 GPU 자원을 격리시킨다.

즉, 실제 GPU는 여러 Process를 수용하지만, gpu-operator의 제약조건을 차용하게 되어 하나의 Pod만 특정 GPU를 사용할 수 있는 것이다. Node 내 일반 Process의 경우, 해당 제약조건 없이 GPU를 사용할 수 있다.

## Workarounds with gpu-operator
고려사항은 다음과 같다.
* GPU 내 Core / VRAM 단위의 자원 격리가 이루어지는가?
* GPU:Pod = 1:N 연동이 구성될 경우, 대응되는 Pod의 개수가 추가 / 제거됨에 따라 Pod에 할당된 Core / VRAM이 변동되지 않는가?

### Use GPU devices on node without fixed allocation
gpu-operator의 경우 Pod에 대한 `nvidia.com/gpu` Resource Requests를 구성하지 않았을 경우 [기본값으로 Node에 존재하는 모든 GPU를 사용할 수 있다.](https://github.com/NVIDIA/k8s-device-plugin/issues/61)
따라서, gpu-operator의 제약조건을 차용하지 않고 여러 Pod에 대응되는 Process가 GPU를 동시에 사용하는 구성이다.

### Nvidia Time-Slicing
Nvidia Time-Slicing을 통해 GPU를 time-slicing하여 사용할 수 있다.
고정적으로 GPU의 자원을 1:N으로 분리하지만, 자원 격리가 Strict하게 구성되지 않아 VRAM 제한을 tensorflow / pytorch 코드 레벨에서 구성하지 않았다면 VRAM OOM이 발생할 수 있다.

### Nvidia MIG
vGPU의 일종으로, Nvidia MIG (Multi-Instance GPU)를 제공하는 GPU를 통해 vGPU를 구성할 수 있다.
A30의 경우 최대 1:4, A100, H100의 경우 1:7의 GPU : vGPU 대응을 구성할 수 있다.
gpu-operator를 기반으로 간편하게 활성화하고 사용할 수 있지만, A30, A100, H100의 GPU에서만 사용할 수 있다.

## Workarounds without gpu-operator

### gpushare-scheduler-extender
Alibaba Cloud에서 개발한 gpushare-scheduler-extender의 경우 VRAM 단위의 Node Extended Resource를 제공한다.
다만, 구성상 VRAM 단위의 자원 격리가 이루어지지 않으며, 단순히 gpu-operator의 제약조건을 VRAM 단위로 변환시킨 정도에 불과하다. 따라서 pytorch / tensorflow등 code 레벨의 VRAM limit 구성을 요한다.
또한, scheduling을 제공하기 위해 kube-scheduler 내 args 추가 및 volumes 추가를 요해, k3s 혹은 microk8s등의 single node k8s에서는 사용할 수 없다.

### nvshare
nvshare의 경우 GPU:Pod = 1:N 연동을 구성하여 VRAM을 분할한다.
다만, GPU에 대응되는 Pod가 증감할 경우 Pod가 사용할 수 있는 VRAM이 증감하는 이슈가 존재하므로, 자원 격리는 존재하지만 고정적인 자원을 제공하지 못하는 이슈에 따라 프로덕션에서 적합하지 않다.

## Conclusion
이렇게 gpu-operator의 제약조건과, 해당 제약조건을 극복하기 위해 상용 소프트웨어를 차용하지 않고 구성할 수 있는 workaround들을 나열하여 분석하였다.

Core / VRAM 단위의 자원 격리의 경우 Open Source 기반으로 연동할 수 있는 방법을 찾지 못해, 다음과 같은 상황별 recommendation들을 구성하며 글을 맺는다.

* Pod가 고정적으로 Core / VRAM 자원이 격리되는 하나 이상의 GPU를 사용해야 하는 경우 gpu-operator + `nvidia.com/gpu` 명시해서 사용
* Pod가 Core / VRAM에 대한 noisy neighbor 현상을 용인할 수 있으며, 하나의 GPU에 여러 Pod를 사용해야 할 경우 gpu-operator + `nvidia.com/gpu` Resource 정의 없이 사용

## References

[https://github.com/NVIDIA/nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

[https://github.com/NVIDIA/nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit)

[https://github.com/NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator)

[https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.10.0/user-guide.html#gpu-enumeration](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.10.0/user-guide.html#gpu-enumeration)

[https://github.com/NVIDIA/gpu-operator/blob/main/deployments/gpu-operator/values.yaml#L258](https://github.com/NVIDIA/gpu-operator/blob/main/deployments/gpu-operator/values.yaml#L258)

[https://docs.run.ai/v2.17/Researcher/scheduling/GPU-time-slicing-scheduler/](https://docs.run.ai/v2.17/Researcher/scheduling/GPU-time-slicing-scheduler/)

[https://developer.nvidia.com/blog/improving-gpu-utilization-in-kubernetes/](https://developer.nvidia.com/blog/improving-gpu-utilization-in-kubernetes/)

[https://docs.nvidia.com/datacenter/tesla/mig-user-guide/](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

[https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html](https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html)

[https://github.com/grgalex/nvshare](https://github.com/grgalex/nvshare)

[https://github.com/AliyunContainerService/gpushare-scheduler-extender](https://github.com/AliyunContainerService/gpushare-scheduler-extender)
