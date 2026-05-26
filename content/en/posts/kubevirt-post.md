---
title: "Running VMs with KubeVirt on IBM Power9 (ppc64le)"
date: 2026-05-22
draft: false
authors: ["João Ramalho"]
projects: ["multiarq"]
tags: ["KubeVirt", "Power9", "ppc64le", "Kubernetes", "Virtualization"]
translationKey: "kubevirt-post"
summary: "This post documents how we adapted KubeVirt to create and manage virtual machines via Kubernetes on IBM Power9 (ppc64le), an architecture not officially supported by the project."
---

## Context

This post aims to present the process of adapting [KubeVirt](https://kubevirt.io/) for the IBM POWER9 (ppc64le) architecture. It covers the main challenges encountered, the modifications made to the source code, the role of each component, and the results obtained at the end of the process.

KubeVirt is an operator that extends Kubernetes to manage virtual machines (VMs) as native resources. In traditional environments, VMs are managed by tools like libvirt/virsh, separate from the container ecosystem. KubeVirt eliminates this separation: with it, you can create, start, stop, and monitor VMs using the same Kubernetes commands and workflows — `kubectl`, YAML, namespaces, and RBAC. VMs run as real QEMU/KVM processes inside pods managed by Kubernetes.

The motivation for this work arose in the context of the Frente Multiarq project, which maintains a shared HPC infrastructure on IBM POWER9. The ability to manage VMs and containers in the same Kubernetes cluster simplifies environment administration and opens the door to scenarios such as GPU passthrough for AI/ML workloads inside VMs, isolation of research environments, and multi-architecture compatibility testing.

The main challenge is that KubeVirt **does not officially support ppc64le**. Only x86_64 (amd64), arm64, and s390x are supported. This means the build system, API validations, configuration defaults, and the libvirt domain generation pipeline do not recognize ppc64le, defaulting everything to amd64.

## TL;DR

- KubeVirt does not officially support ppc64le; the entire pipeline assumes amd64 as a fallback.
- We compiled Go binaries directly, bypassing the Bazel build system that does not recognize the architecture.
- Patches were required across ~14 Go files and 4 new files were created to add ppc64le support.
- Docker images were built with custom Dockerfiles and served via a local registry.
- With these adaptations, it was possible to run a CirrOS ppc64le VM via KubeVirt on POWER9, managed entirely by Kubernetes.

## Execution Environment

- **Architecture:** IBM Power9 server (ppc64le).
- **Operating System:** AlmaLinux 8.10, binary compatible with RHEL 8.9/8.10.
- **GPUs:** 4x NVIDIA Tesla V100-SXM2-16GB.
- **Docker:** Docker CE 26.1.3.
- **Kubernetes:** v1.35.0 via minikube v1.38.0 (docker driver, containerd runtime).
- **KubeVirt:** v1.8.2.
- **Go:** 1.24.9.

## What KubeVirt Is and How It Works

KubeVirt is composed of several components that work together to translate a Kubernetes resource (the VirtualMachineInstance, or VMI) into a real QEMU/KVM VM running on the host.

The **virt-operator** is the entry point: when the administrator creates the KubeVirt Custom Resource in the cluster, the operator provisions all other components — deployments, daemonsets, services, RBAC. It acts as a permanent installer that reconciles the desired state.

The **virt-api** handles Kubernetes API calls for KubeVirt resources. When the user runs `kubectl apply` on a VMI, virt-api validates the YAML (e.g., is the architecture supported? does the machine type exist?) and injects defaults (e.g., firmware UUID, CPU topology).

The **virt-controller** watches VMIs and decides where they should run. It creates a special pod — the virt-launcher — on the appropriate node, with all necessary configurations (volumes, devices, node selectors).

The **virt-handler** runs as a DaemonSet (one per node) and is the local agent that bridges Kubernetes and libvirt/QEMU. When the virt-launcher pod appears on the node, virt-handler reads the VMI spec, generates the libvirt domain XML, and instructs libvirt to create the VM. It also registers device plugins with the kubelet (`/dev/kvm`, `/dev/net/tun`, `/dev/vhost-net`) so pods can access the required devices.

The **virt-launcher** is the pod that encapsulates the VM. Each VMI generates a dedicated pod with three containers: compute (QEMU + libvirt), guest-console-log, and container disk. Inside the compute container, the QEMU process runs the actual VM — with its own kernel, memory, and virtual CPU.

The full flow is:

1. `kubectl apply` → API Server → virt-api (validates, injects defaults)
2. virt-controller detects the VMI → creates the virt-launcher pod on the appropriate node
3. kubelet starts the virt-launcher pod on the node
4. virt-handler detects the pod → reads the VMI spec → generates libvirt XML → calls libvirt
5. libvirt starts QEMU → VM runs inside the pod

It is important to note that the VM **does not become a container** — it runs as a real QEMU process inside a pod. Kubernetes manages the pod lifecycle, and KubeVirt translates between the two worlds.

## Challenges and Adaptations

### Build System

KubeVirt's build system uses Bazel, which does not recognize ppc64le. The `format_archname` function in the build script only accepts `x86_64`, `aarch64`, and `s390x`. The solution was to compile the Go binaries directly with `go build`, bypassing Bazel.

An additional dependency is **libnbd**: virt-launcher requires version 1.18+, but AlmaLinux 8 only provides 1.6. It was necessary to compile libnbd 1.20 from source. The **container-disk** component is a C program (not Go) that requires static compilation to run in `FROM scratch` containers.

### API Validation

The virt-api validation webhook rejects VMIs with an unknown architecture. Without the patch, a VMI with `architecture: ppc64le` would be rejected before even reaching the scheduler. It was necessary to add cases in the admitter and create a specific validation function for ppc64le.

### Configuration Defaults

KubeVirt needs to know which machine type to use for each architecture (e.g., `pc-q35` for amd64, `virt` for arm64). For ppc64le, we configured `pseries` as the default machine type — the virtual machine type for POWER.

### Libvirt Domain Generation

This was the central challenge. KubeVirt converts the VMI spec into a libvirt domain XML that QEMU interprets. This pipeline has two parts:

The **arch-defaulter** sets default OS type values (`arch` and `machine`) in the XML. Without the patch, it returned `x86_64` for ppc64le, causing libvirt to attempt creating an x86 VM on a POWER machine — resulting in the error `No emulator found for arch 'x86_64'`.

The **converter** is an interface with ~12 methods that define architecture-specific behaviors: whether USB is needed, SMBIOS, PCIe placement, ROM tuning, etc. Implementations existed for amd64, arm64, and s390x, but not for ppc64le. The code fell back to `converterAMD64`, generating incompatible configurations. We created `converterPPC64LE` with values appropriate for POWER: no USB, no SMBIOS, no PCIe placement, with VirtIO as the disk model.

After resolving the converter, an **USB device** error appeared: the graphics/video pipeline had no case for ppc64le, causing libvirt to add a default VGA video device that depended on USB — but the USB controller was disabled (`IsUSBNeeded: false`). The solution was to add a ppc64le case in the video configurator with `virtio` as the video device, following the same pattern as s390x and arm64.

Finally, the **CPU model**: KubeVirt uses `host-model` as the default, which does not work with nested virtualization on POWER9. The solution was to specify `POWER9` as the CPU model in the VMI.

### Docker Images

With no Dockerfiles in the project (everything is generated by Bazel), we created custom Dockerfiles for each component. The simpler components (virt-operator, virt-api, virt-controller, virt-exportproxy) use `ubi8/ubi-minimal` as the base. virt-handler requires additional system tools. virt-launcher is the most complex, using `almalinux:8` as the base and dependencies on qemu-kvm, libvirt, and the compiled libnbd. A local registry (`registry:2` on port 5000) serves the images to minikube.

### Technical Step-by-Step Guide

Due to the large number of steps involved, a detailed step-by-step guide with all patches to the Go code, Dockerfiles, compilation commands, and configuration is available at this link:
[kubevirt-ppc64le-installation-guide](https://github.com/llm-pt-ibm/kubevirt-ppc64le).

## Results

With all adaptations applied, it was possible to run a CirrOS ppc64le VM via KubeVirt on POWER9, managed entirely by Kubernetes:

```
$ kubectl get vmi test-vmi -o wide
NAME       AGE     PHASE     IP               NODENAME   READY
test-vmi   2m43s   Running   10.244.120.124   minikube   True
```

Data collected from inside the VM confirms correct execution:

|                      | Value                                  |
|----------------------|----------------------------------------|
| Architecture         | ppc64le                                |
| CPU                  | POWER9 (architected), altivec supported|
| Hypervisor           | KVM                                    |
| Platform             | pSeries                                |
| Model                | IBM pSeries (emulated by qemu)         |
| Kernel               | 5.15.0-71-generic ppc64le              |

These results confirm that KubeVirt is generating the correct libvirt domain for ppc64le, with machine type `pseries`, POWER9 CPU, and KVM/QEMU virtualization with VirtIO paravirtualization.

## Final Considerations

With the adaptations made, it became possible to use KubeVirt to create and manage virtual machines on an IBM POWER9 via Kubernetes. The VM that runs is a real KVM/QEMU VM — with its own kernel, isolated memory, and virtual CPU — managed like any other Kubernetes resource.

In the context of the Multiarq project, this solution allows unifying the management of containers and VMs in the same cluster, simplifying administration of the shared infrastructure. Workloads that require kernel isolation or direct hardware access (such as GPU passthrough) can run in VMs without leaving the Kubernetes ecosystem.

The patches made are potentially contributable to the upstream KubeVirt project. KubeVirt's architecture already provides for extensibility by architecture — the interface pattern (Converter, ArchDefaulter) and per-arch switches make it straightforward to add new platforms. ppc64le follows the same pattern as s390x, which was also added to the project at a later stage.

## Next Steps

- Resolve the USB/Graphics conflict to allow VNC without the `autoattachGraphicsDevice: false` workaround, enabling graphical access to VMs;
- Adjust the default CPU model in the code so that ppc64le automatically uses `POWER9` without requiring manual specification in the VMI;
- Explore GPU passthrough of the V100s via KubeVirt to run AI/ML workloads inside VMs managed by Kubernetes;
- Test other distributions as containerDisk (Fedora, Ubuntu Server, AlmaLinux ppc64le) to validate compatibility beyond CirrOS;
- Configure masquerade networking to enable live migration between nodes;
- Document the changes in PR format for contribution to the KubeVirt upstream;
- Validate KubeVirt on the Single Node OpenShift (OCP 4.21) already installed on the machine, using OpenShift Virtualization as the operator.
