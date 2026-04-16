---
title: "LLM Inference with Ollama on IBM Power9 Using GPU"
date: 2026-04-16 # year-month-day
authors: ["Maria Luísa Gomes"] # Can be a list
tags: ["LLM", "Ollama", "Power9", "GPU", "Inference"]
projects: ["multiarq"]
translationKey: "ollama-gpu"
summary: "This post documents how to perform LLM inference with Ollama on IBM Power9 using GPU, focusing on a fork compatible with NVIDIA Tesla V100 on POWER9."
draft: false # Change to true if you want the post to remain as a draft
---

## Context
This is the second post in the series about language model inference on POWER9 with [<span class="link-personalizado">*Ollama*</span>](https://ollama.com/). In this article, we will cover how to send requests using GPU, achieving a significant performance gain compared to the CPU approach shown in the [<span class="link-personalizado">*previous post*</span>](https://llm-pt-ibm.github.io/en/posts/ollama_cpu/).

The main challenge is that Ollama does not offer official support for the *ppc64le* architecture with [<span class="link-personalizado">*CUDA*</span>](https://developer.nvidia.com/cuda-12-2-0-download-archive?target_os=Linux&target_arch=ppc64le&Distribution=RHEL&target_version=8&target_type=rpm_local). The solution was found through an official IBM community blog, where a contributor made a [<span class="link-personalizado">*fork*</span>](https://github.com/naveedus/ollama-ppc64le) of Ollama adapted to support NVIDIA GPUs on POWER9 via CUDA, specifically targeting IBM Power AC922 with Tesla V100. However, simply using the repository is not enough — you must build it correctly so Ollama can detect and use the GPU. This tutorial explains exactly how to do that.

## TL;DR
- This post details the environment setup to perform inference using IBM POWER9 infrastructure;
- Ollama does not offer official support for *ppc64le* with CUDA;
- The solution was to use a fork developed by an IBM contributor, found in an official IBM community blog;
- The fork was built from scratch using `make`, pointing to CUDA 12.2 and specifying the V100 architecture (`sm_70`);
- With this, it was possible to run LLM inference on IBM POWER9 with GPU acceleration.

## Environment Used

**Hardware**:
- *ppc64le* architecture;
- Recommended minimum RAM: ~64GB;
- GPU: NVIDIA Tesla V100;
- NVIDIA driver: 535.54.03;
- CUDA: version 12.2.

**Operating System:** Alma Linux 8.10 (*ppc64le*), binary compatible with *Red Hat Enterprise Linux (RHEL)* 8.9/8.10.

## Initial Checks

1. Verify that the driver and GPU are visible:

```bash
nvidia-smi
```

2. Verify that CUDA is installed:

```bash
nvcc --version
```

Note: If nothing appears, try:

```bash
export PATH=/usr/local/cuda-12.2/bin:$PATH
export CUDACXX=/usr/local/cuda-12.2/bin/nvcc
```

3. Also verify that CUDA 12 exists:

```bash
ls -la /usr/local/cuda-12
```

## Running in a Virtual Environment

In this tutorial, we make the necessary configuration inside a virtual environment to isolate execution and settings. This is optional but recommended.

```bash
conda create -n ollamaGPU python=3.11 -y
conda activate ollamaGPU
```

To deactivate the environment:

```bash
conda deactivate
```

## Initial Setup
To run Ollama on the POWER9 architecture, the environment must be prepared with the appropriate dependencies. The first step is to update the system and install the basic utilities:

```bash
sudo dnf update -y
sudo dnf install -y wget git tar make gcc gcc-c++ cmake gcc-toolset-11
```

Although this command installs some dependencies, you must ensure the correct versions are being used.

### Configuring *Go*
Ollama is developed in *Go*, so it is necessary to have the correct version.

**Expected version:** 1.25.7.

#### If it is not installed or if the version differs:

```bash
wget https://go.dev/dl/go1.25.7.linux-ppc64le.tar.gz
sudo tar -C /usr/local -xzf go1.25.7.linux-ppc64le.tar.gz
export PATH=/usr/local/go/bin:$PATH
```

To add it to *PATH* permanently:

```bash
echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verify the version is correct: `go version`

### Configuring *CMake*
Verify the version: `cmake --version`

**Expected version:** 3.26.5.

#### If necessary, install it manually:

```bash
wget https://github.com/Kitware/CMake/releases/download/v3.26.5/cmake-3.26.5.tar.gz
tar -xzf cmake-3.26.5.tar.gz
cd cmake-3.26.5
./bootstrap
make -j$(nproc)
sudo make install
```

### Configuring *GCC*

**Expected version:** 11.2.1.

**Important:** On AlmaLinux 8, the `gcc-toolset` is not enabled automatically. Enabling it depends on whether you are using a conda virtual environment or not:

Without a conda virtual environment:

```bash
scl enable gcc-toolset-11 bash
```

With a conda virtual environment active: the `scl enable` command opens a subshell that conflicts with conda and corrupts `PATH`. Use manual export instead:

```bash
export PATH=/opt/rh/gcc-toolset-11/root/usr/bin:$PATH
export LD_LIBRARY_PATH=/opt/rh/gcc-toolset-11/root/usr/lib64:$LD_LIBRARY_PATH
```

In both cases, verify the version:

```bash
gcc --version
```

Important: GCC activation must be done every time you open a new terminal before compiling.

## Cloning and Building Ollama

With the environment configured, we can build Ollama. Here we clone the fork repository from an IBM contributor, which focuses on GPU support. This fork was found on an official IBM community blog and contains the necessary adjustments for the POWER9 architecture to properly build and detect the GPU:

```bash
git clone https://github.com/naveedus/ollama-ppc64le.git ollama-gpu
cd ollama-gpu
git checkout witherspoon
```

Verify you are on the correct branch: `git branch` (it should show `witherspoon`).

### Building with CUDA support:

```bash
make CUDA_ARCHITECTURES="70"
```

In the previous tutorial, when we built Ollama for CPU, we used `go build`, which only compiles Ollama's Go code and not the CUDA libraries from `llama.cpp`. Here we need a different approach: use `make` to compile the CUDA kernels with `nvcc`, produce the CUDA runner as a shared library, and then call `go build`.

Why `CUDA_ARCHITECTURES="70"`? Each NVIDIA GPU has a specific architecture identified by an `sm_XX` code. The V100 is part of the Volta architecture, whose code is `sm_70`. By default, the build compiles for all supported CUDA architectures (sm_60 through sm_90), which significantly increases compile time. By specifying `CUDA_ARCHITECTURES="70"`, we instruct the build to target only the V100, making the process faster.

The build will take a few minutes. When finished, verify that the CUDA runner was generated:

```bash
ls -lh llama/build/linux-ppc64le/runners/cuda_v12/
```

You should see two files: `libggml_cuda_v12.so` and `ollama_llama_server`. Confirm that the runner is linked to CUDA:

```bash
ldd llama/build/linux-ppc64le/runners/cuda_v12/libggml_cuda_v12.so | grep -i cuda
```

The output should show `libcuda.so`, `libcublas.so`, `libcudart.so`, and `libcublasLt.so`.

## Running inference

With Ollama built, we can start the server:

```bash
./ollama serve
```

To verify it worked: `ps aux | grep ollama`.

In another terminal, wait a few seconds and check the logs to confirm the server detected the GPUs correctly. Look for these lines:

```bash
Dynamic LLM libraries runners="[cpu cuda_v12]"
inference compute ... library=cuda compute=7.0 ... total="15.0 GiB"
```

## Downloading the test model and running inference

For validation, we used the `llama3.1:8b` model. In another terminal, run:

```bash
./ollama pull llama3.1:8b
```

To run inference:

```bash
./ollama run llama3.1:8b "tell me all odd numbers up to 100"
```

## Confirming GPU usage

In another terminal, while inference is running, run:

```bash
nvidia-smi
```

In the processes section, you should see `ollama_llama_server` with memory allocated on one of the GPUs:

{{< figure src="/images/ollama_inference_gpu.png" alt="Figure 1" caption="Ollama using the GPU">}}

## Final Considerations

With the steps presented, it was possible to configure the environment to run LLM inference on an IBM POWER9 machine using NVIDIA Tesla V100 GPUs. With this approach, model inference has a significant performance gain compared to CPU execution. Using the Meta Llama 3.1 8B Instruct model as a reference, GPU execution achieved a higher token generation rate than CPU execution.

Here are the collected results for the same request (`tell me all odd numbers up to 100`) with both types of execution:

| | CPU | GPU |
|----------|-------------|-------------|
| Token generation rate | 0.71 tokens/s | 79.82 tokens/s |
| Total duration | 3m49s | 4.52s |
| Prompt evaluation rate | 10.67 tokens/s | 295.77 tokens/s |

From the table, we see that GPU execution was approximately 112 times faster in token generation, with total response time reduced from 3 minutes and 49 seconds to 4.52 seconds.

## Next Steps

- Evaluate GPU and CPU execution in a comparative post with other architectures;
- Test GPU inference with larger models, above 8 billion parameters;
- Compile Ollama for a newer version to load newer models and evaluate performance;
