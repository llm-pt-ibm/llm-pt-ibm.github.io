---
title: "Compiling and Running Red Hat's SDG Hub and Training Hub on IBM Power9"
date: 2026-08-31
authors: ["Lucas Pereira"]
tags: ["LLM", "SDG Hub", "Training Hub", "Power9", "GPU", "Fine-tuning"]
projects: ["multiarq"]
translationKey: "sdg-training-hub"
summary: "Guide to building and running Red Hat's SDG Hub and Training Hub on IBM Power9 (ppc64le), covering the PyTorch, xFormers, Triton, and Ray dependency stack, plus a fully containerized environment for synthetic data generation and fine-tuning."
draft: false
---

## Background
[<span class="link-personalizado">InstructLab</span>](https://github.com/instructlab) is a tool for fine-tuning large language models. Although it is still in use, the project has recently been reorganized. Now maintained by Red Hat, InstructLab has been split into two independent tools: SDG Hub (Synthetic Data Generation Hub) and Training Hub.

SDG Hub is responsible for the synthetic data generation stage, while Training Hub handles the entire model training process. The purpose of this post is to demonstrate how to build and run these tools on an IBM Power9 system, highlighting the adaptations required to make them work on this architecture, since some of their dependencies do not yet provide native support for the platform.

## TL;DR
- Building and running SDG Hub on Power9 (ppc64le) for synthetic data generation, using Conda to manage PyArrow, Pandas, NumPy, and Matplotlib.
- Building Training Hub's full dependency stack on ppc64le, including PyTorch, xFormers, LLVM, Triton, TorchVision, and Ray.
- Solving Ray's glibc 2.34 requirement, which is not available on AlmaLinux 8, through a hybrid container-based solution.
- Fine-tuning language models with Training Hub on IBM Power9, using NVIDIA V100 GPUs.
- A fully containerized environment (`build.sh` / `run.sh`) that automates installation and makes both tools reproducible on new systems.

## Execution Environment
The environment used to build and run SDG Hub and Training Hub includes:

- **Architecture:** IBM Power9 Server.
- **Operating System (OS):** AlmaLinux 8.10 binary compatible with Red Hat Enterprise Linux (RHEL) 8.9/8.10.
- **RAM:** 512GB.
- **GPUs:** 4x NVIDIA Tesla V100 SXM2 16GB.

## Understanding the Dependency Stack
For SDG Hub, only a few adaptations were required. Since this module mainly relies on API-based services for synthetic data generation, it was sufficient to use Conda to install the required dependencies, including PyArrow 25.0.0, Pandas 2.3.0, NumPy 1.26.4, and Matplotlib 3.10.0.

Training Hub, on the other hand, required significantly more work. Its main dependencies include PyTorch 2.7, which provides the deep learning framework used during training; xFormers 0.0.35, which offers optimized implementations of attention mechanisms; LLVM and Triton, used to compile highly optimized GPU kernels for NVIDIA hardware; and TorchVision, which is required by components of the PyTorch ecosystem.

Another library that required special attention was Ray, which is responsible for distributed execution and training orchestration. For the ppc64le architecture, we used an IBM-provided build, compiled against glibc 2.34, which is available in AlmaLinux 9, whereas our target environment runs AlmaLinux 8. As a result, we developed a hybrid container-based solution that combines components from both distributions, ensuring compatibility across all the dependencies required by Training Hub.

## Containerizing the Solution
To simplify deployment and reproducibility, we developed a repository containing a fully containerized solution that packages all the dependencies and configuration required to run both SDG Hub and Training Hub on the IBM Power9 architecture.

Repository: [<span class="link-personalizado">sdg_training_hub</span>](https://github.com/llm-pt-ibm/sdg_training_hub).

The repository automates nearly the entire installation and execution process. Running `bash build.sh` builds the container images and installs Training Hub 0.9.0 together with SDG Hub 0.9.4. Once the build is complete, the environment can be started by running `bash run.sh`, making both the synthetic data generation and model training modules immediately available.

{{< figure src="/images/sdg_training_hub_run.gif" alt="Figure 1" caption="Training Hub running on the IBM Power9 architecture." >}}

Figure 1 shows Training Hub running on an IBM Power9 server.

## Final Considerations
This work demonstrates how Red Hat's synthetic data generation and model training modules can be adapted to run on IBM Power9 servers. Despite the lack of official support for some dependencies on the ppc64le architecture, the adaptations presented here make it possible to fully leverage the computational capabilities of the platform, using four NVIDIA Tesla V100 GPUs for both synthetic data generation and language model fine-tuning. In addition, the containerized solution makes the environment easy to reproduce and simplifies deployment on new systems.

## Disclaimer
This work is not an official IBM release or software distribution and is not developed or supported by IBM.

This work was developed by the Federal University of Campina Grande (UFCG), a Brazilian public university, as part of a Research, Development, and Innovation project conducted in partnership with IBM and Flex Brazil.
