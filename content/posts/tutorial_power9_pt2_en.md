---
title: "Setting Up the Conda and PyTorch on IBM Power9 Servers"
date: 2025-06-29 # ano-mês-dia
authors: ["Caio Silva"] # Pode ser uma lista
tags: ["LLM", "Power9", "Conda", "PyTorch"]
summary: "This post is part of a tutorial series aimed at building a Language Model API on Power9 servers. In this step, we'll set up the Conda package manager and the PyTorch library."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Context
This is the second post in a tutorial series on how to build a Language Model API on an IBM Power9 server, covering everything from setting up the operating system to having the API running remote inference. The [first post]({{< relref "tutorial_power9_pt1_en.md" >}}) covers installing the OS and configuring NVIDIA drivers, CUDA, and CUDNN. In this step, we'll show how to set up the Conda package manager and the PyTorch library.

**Conda**: Conda is an open-source, cross-platform package and environment management system. It's like a "toolbox" for data scientists and developers to organize their projects.

**PyTorch**: PyTorch is an open-source machine learning library developed primarily by Facebook AI Research (FAIR). It's especially popular for building deep learning applications, a subfield of machine learning inspired by how the human brain works.

## TLDR
- This post provides a step-by-step guide to installing Conda and PyTorch.
- The main challenge is finding compatible versions for the Power9 machine architecture.

## Setting up the Conda
We'll start with installing **Conda**. On Power systems, the architecture used is `ppc64le` (PowerPC 64-bit little-endian), so it's essential to download the version for this architecture. We'll use **miniconda**, a lighter option that's better suited for custom setups like the Power9 server.

1. To download and install the latest version of Miniconda:
```
sudo wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-ppc64le.sh
bash ~/Miniconda3-latest-Linux-ppc64le.sh
```

2. Check if Conda was activated automatically:
```
conda -–version
```
If it didn't start automatically, you'll need to activate it.

3. To ensure it's automatically activated with each new connection, we will write the command into your `.bashrc` (or `.zshrc`) file.

```
echo 'source ~/miniconda3/etc/profile.d/conda.sh' >> ~/.bashrc
source ~/.bashrc
```
Check again with the command:
```
conda -–version
```
Expected output looks like: ```conda 23.10.0```

## Installing and configuring the PyTorch library
There are no official builds or Conda/PyPi wheels with full support for the **ppc64le** architecture. To install PyTorch, you’ll need to build it manually.

#### (Optional) Creating a Conda virtual environment
It’s recommended to create a dedicated virtual environment to install PyTorch in isolation.  
1. To create and activate the virtual environment, run:

```
conda create -y -n api_llm python=3.10
conda activate api_llm
```
#### Installing prerequisites
We need to install some packages required to properly build PyTorch.

1. First, install the packages using the following commands:

```
conda install -y -c conda-forge openblas libblas cmake ninja python3-devel gcc-c++ rust cargo
```
CMake (the build system used by PyTorch) dropped support for scripts declaring compatibility with older versions (<3.5). To address this, we need to install a version of cmake <3.5 using pip.

2. Run the command:
```
pip install cmake==3.27.7
```

To make sure the correct version was installed, run the command:
```
cmake --version 
```

Expected output: ```cmake version 3.27.7```

#### Building PyTorch
Now let's start the **PyTorch** build process.

1. The first step is to clone the repository and set it up to install version 2.6.0:

```
git clone --recursive https://github.com/pytorch/pytorch
cd pytorch
git checkout v2.6.0 
git submodule sync 
git submodule update --init --recursive 
```

2. To install the required packages via pip, run the following command:
```
pip install -r requirements.txt
```
3. And finally, to build PyTorch, run Python’s setup.py:
```
sudo USE_CUDA=1 USE_DISTRIBUTED=1 USE_NCCL=1 USE_GLOO=1 USE_CUDNN=1 python setup.py install
```
The build process usually takes a while, around 15 minutes.

4. To check if everything worked correctly, create a file named ```test_torch.py```

```
nano test_torch.py
```
This file should contain the following lines:

```python {linenos=inline}
import torch
print(torch.__version__)
print("CUDA available:", torch.cuda.is_available())
print("Number of GPUs:", torch.cuda.device_count())
print("GPU name:", torch.cuda.get_device_name(0))
x = torch.rand(3, 3).cuda()
y = torch.rand(3, 3).cuda()
print("Sum on GPU:", (x + y))
print("cuDNN available:", torch.backends.cudnn.is_available())
print("C extensions loaded:", torch._C._cuda_getDeviceCount() > 0)
```
When you run this file, you’ll check:

- Installed PyTorch version
- CUDA availability
- Number of available GPUs
- GPU name on the Power9 server
- Whether GPU usage is working correctly
- CUDNN availability
- Whether the .so files were compiled correctly

This script simply verifies some CUDA and PyTorch informations and performs a basic addition operation using GPU tensors.

5. Run the file with the command:

```
python test_gpu.py
```

Expected output should look something like:

```bash
2.6.0a0+git1eba9b3
CUDA available: True
Number of GPUs: 4
GPU name: Tesla V100-SXM2-16GB
Sum on GPU: tensor([[1.9163, 1.2208, 0.5998],
        [1.7962, 0.6040, 1.3943],
        [0.9536, 0.8010, 0.0668]], device='cuda:0')
cuDNN available: True
C extensions loaded: True
```

Keep in mind that the output may vary depending on the number and model of GPUs, as well as the tensor sums (due to randomness). What matters is that the boolean outputs in the script return ```True```.

With this, PyTorch is installed and ready to use. In the next tutorial, we’ll run the first Language Model inference on the Power9 server.

