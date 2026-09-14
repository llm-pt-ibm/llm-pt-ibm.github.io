---
title: "Evaluating Virtualization Cost on IBM Power9: Host, KVM, and QEMU"
date: 2026-09-14
authors: ["Gabrielly Lima"]
tags: ["Virtualization", "Power9", "KVM", "QEMU", "Benchmark", "GPU"]
projects: ["multiarq"]
translationKey: "power9-virtualization-overhead"
summary: "We measured virtualization overhead on IBM Power9 (ppc64le) comparing the bare-metal host, KVM, and QEMU with NVIDIA V100 GPU passthrough, and describe a configuration bias that would have made virtualization appear faster than the host itself."
draft: false
---

## Context

Running GPU-accelerated workloads on IBM Power9 systems often involves choosing between running directly on physical hardware and using some level of virtualization, for isolation, Kubernetes orchestration, or reproducibility across teams. This choice comes at a cost, but how large is it?

This post describes how we measured virtualization overhead on IBM Power9 (ppc64le), comparing the bare-metal host, KVM (with hardware acceleration), and QEMU running under TCG (pure software emulation, no hardware acceleration), with NVIDIA V100 GPU passthrough in the environments that support it. We also describe a methodological discovery made along the way: a configuration bias that, if left unidentified, would have made virtualization appear faster than the bare-metal host itself.

## What This Work Delivers

* A reproducible benchmarking pipeline comparing the bare-metal host, KVM, and QEMU on ppc64le, covering CPU, memory, cache, memory bandwidth, storage, and GPU performance.
* A documented case showing how a host configuration bias can inflate or even reverse virtualization benchmark results, and how to identify it.

## Experimental Environment

### Host hardware

* IBM Power9 (AC922) server;
* ppc64le (64-bit little-endian) architecture;
* 512 GB RAM;
* 4× NVIDIA Tesla V100 SXM2 (16 GB);
* SSD/NVMe storage, /dev/sdb1, ext4;
* Operating system: AlmaLinux 8.10;
* Kernel 4.18.0-553.105.1.el8_10.ppc64le.

### Host software, by role

* Orchestration: Slurm 23.11.7;
* Virtualization: Libvirt 8.0.0, QEMU 6.2.0;
* GPU stack: CUDA Toolkit 12.2, cuDNN 9.0, NCCL 2.18.3, NVIDIA Driver 535;
* Benchmark tools: stress-ng 0.15.00, Fio 3.19, Python 3.13.12 (analysis pipeline).

### VM configuration

Each VM was configured with 16 vCPUs and 64 GB of RAM, using a virtio disk configured for direct I/O, bypassing host-side caching.

## Methodology

We compared the three environments using 15 executions each, following a nested design for KVM and QEMU consisting of three independent virtual machines with five replicas each, avoiding treating replicas from the same VM as independent samples. We scheduled runs through Slurm, which ensured consistent resource allocation and prevented overlapping executions.

The benchmark suite covered CPU and memory (stress-ng), cache (stress-ng), memory bandwidth (STREAM), storage (fio), and GPU (matrixMul and NCCL). Figure 1 shows the full pipeline, from data collection to statistical validation.

{{< figure src="/images/evaluation_pipeline.png" alt="Figure 1: Overview of the benchmarking pipeline." caption="Figure 1: Overview of the benchmarking pipeline: each environment runs the same benchmark suite, 15 executions are collected, and results pass through a formal statistical validation step before producing the final overhead estimates." >}}

Every comparison followed a formal statistical pipeline. Since we could not assume the underlying distributions were normal or had equal variances, we tested both assumptions (Shapiro-Wilk for normality, Levene for homogeneity of variance) before choosing between ANOVA and Kruskal-Wallis for each metric. When the global test showed a significant difference between environments, we used Dunn's post-hoc test to identify which specific pairs differed. Because we tested many metrics at once, we applied Benjamini-Hochberg correction across all of them to keep the false discovery rate under control.

## Findings

Two metrics, cache performance and random IOPS at queue depth 1, initially showed KVM outperforming the bare-metal host by a wide margin (over 300% on the cache benchmark), a result that is not physically possible since virtualization adds overhead rather than removing it.

The cause was a missing CPU pinning configuration on the host: the host benchmarks ran with processes free to migrate across all cores and NUMA nodes, while the VMs were always pinned to fixed physical cores. Applying the same pinning to the host resolved both discrepancies; all other results changed by less than 4 percentage points, confirming the issue was isolated to these two measurements. Figure 2 shows the effect of the fix on both metrics.

{{< figure src="/images/effect_pinning.jpeg" alt="Figure 2: Cache and IOPS QD=1 overhead before and after the pinning fix." caption="Figure 2. Cache and IOPS QD=1 overhead before and after the pinning fix. The \"before\" values come from the biased, uncorrected host baseline, not a valid measurement of virtualization performance. The shaded band marks a ±5% zone of negligible variation." >}}

Methodological note: consistent CPU pinning across all compared environments is essential for virtualization overhead measurements to be physically meaningful. An asymmetric configuration, even an accidental one, can fully invert the measured result.

## Results

Figure 3 shows the overhead across all measured domains.

{{< figure src="/images/consolidated_overview.jpeg" alt="Figure 3: Overhead relative to the bare-metal host across all measured domains." caption="Figure 3. Overhead relative to the bare-metal host across all measured domains (CPU, memory, cache, STREAM, disk, and GPU). Each panel uses its own scale; bar length should not be compared across panels." >}}

In summary, KVM introduces moderate and localized overhead, mainly affecting cache performance, memory bandwidth, and random IOPS, a pattern consistent with the expected cost of hardware-assisted virtualization. QEMU (TCG) introduces substantial overhead across nearly every workload except sequential I/O, as expected from software emulation without hardware acceleration. GPU passthrough exhibited no measurable overhead in KVM.

## Conclusion

This work demonstrates that it is possible to rigorously measure the cost of running GPU-accelerated workloads under virtualization on IBM Power9 using sound statistical methodology, and that this cost is significantly lower with KVM than with QEMU, as expected.

It also demonstrates that virtualization benchmarking is highly sensitive to system configuration variables beyond the mere presence or absence of a hypervisor. CPU pinning is a well-known confound in virtualization benchmarking, and it would have distorted the conclusions had it gone unnoticed.

## Disclaimer

This work is not an official IBM release or software distribution and is not developed or supported by IBM.

This work was developed by the Federal University of Campina Grande (UFCG), a Brazilian public university, as part of a Research, Development, and Innovation project conducted in partnership with IBM and Flex Brazil.

## Resources

* GitHub repository: [<span class="link-personalizado">llm-pt-ibm/ppc64le-virtualization-performance</span>](https://github.com/llm-pt-ibm/ppc64le-virtualization-performance)
