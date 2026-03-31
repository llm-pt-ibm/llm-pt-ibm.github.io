---
title: "Performing CPU Inference on Power10"
date: 2025-04-06 # ano-mês-dia
authors: ["André Alves", "Ronaldd Matias"] # Pode ser uma lista
tags: ["LLM", "Power10"]
projects: ["llm-eval"]
translationKey: "power10"
summary: "This post explains how to run the Granite-20b-Code-Instruct model on CPU on a Power10 machine."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Background
In this post, we will share our experience running the Granite-20b-Code-Instruct model on a Power10 machine, describing the challenges and the necessary configurations to perform inference using Llama.cpp, one of the most popular open-source libraries in this domain.


## TL;DR
- This post provides details on how to set up and run inference using IBM Power10 infrastructure.
- Our main challenge was configuring Llama.cpp, which required adjustments such as installing Ninja-builder, compiling OpenBLAS, and updating the C compiler.

## Infrastructure
Inference was performed on a machine with IBM POWER10 architecture, equipped with 750 GB of RAM and running Red Hat Enterprise Linux 8.10. Access to the environment was provided through a VM, requiring the use of a VPN to establish secure and controlled communication with the system, enabling remote and efficient execution of activities.

## Initial Setup
The library that enables run LLMs using CPU resources is Llama.cpp. To set it up, we needed to resolve two external dependencies: Ninja-builder and OpenBLAS. Ninja-builder optimizes the compilation process, while OpenBLAS is a high-performance library for matrix computations.

During the OpenBLAS build process, we identified discrepancies in the internal tests validating matrix calculations, indicating a compatibility problem with the available C compiler, which was an older version (8.5.0). The solution was to **update the compiler to a newer version, 13.2**, ensuring better compatibility with the Power10 architecture and validating the accuracy of the numerical operations required for Llama.cpp. Below, we present the step-by-step process used to enable the compilation of the required libraries and update the C compiler.

1. Creating the build environment for the builder


```
sudo dnf update -y  && dnf -y groupinstall 'Development Tools' && dnf install -y \ cmake git ninja-build-debugsource.ppc64le \ && dnf clean all
```

2. Updating the C Compiler and Setting Environment Variables

```
scl enable gcc-toolset-13 bash
export CC=/usr/bin/gcc-13
export CXX=/usr/bin/g++-13
```

3. Downloading and Building OpenBLAS

```
git clone --recursive https://github.com/DanielCasali/OpenBLAS.git && cd OpenBLAS && \ make -j$(nproc --all) TARGET=POWER10 DYNAMIC_ARCH=1 && \ make PREFIX=/opt/OpenBLAS install && \ cd /
```

4. Downloading and Building Llama.cpp using the OpenBLAS library

```
 git clone https://github.com/DanielCasali/llama.cpp.git && cd llama.cpp && sed -i "s/powerpc64le/native -mvsx -mtune=native -D__POWER10_VECTOR__/g" ggml/src/CMakeLists.txt && \ mkdir build; \ cd build; \ cmake -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS -DBLAS_INCLUDE_DIRS=/opt/OpenBLAS/include -G Ninja ..; \ cmake --build . --config Release
```

With all these steps completed successfully, the environment was properly configured and optimized for running Llama.cpp locally. We are now able to start a server to perform inference with LLMs efficiently, using only CPU resources.


## Performing Inference

We chose the Granite-20b-code-instruct model in the .GGUF format, which is specifically designed to optimize the performance of language models in CPU-only environments. These models are quantized, meaning their calculation precision is reduced, which in turn lowers their size and memory consumption, making them ideal for efficient execution with Llama.cpp. This approach enables high-performance local inference even on processor-only architectures such as POWER10.
The model was downloaded directly from Hugging Face. Below, we show the step-by-step process to download it:

1. Create a directory for the model in Llama.cpp:

```
mkdir -p /root/llama.cpp/models/granite-20b-code-instruct-8k-GGUF
```

2. Access the directory in Llama.cpp:

```
cd /root/llama.cpp/models/granite-20b-code-instruct-8k-GGUF
```

3. Download the model from Hugging Face:

```
wget https://huggingface.co/ibm-granite/granite-20b-code-instruct-8k-GGUF/resolve/main/granite-20b-code-instruct.Q4_K_M.gguf
```

The last step can take longer based on the model’s number of parameters.. However, once the steps above are completed, we can start a Llama.cpp server to perform inference. By default, the server is exposed on port 8080 of the Power10 machine, but this is fully customizable. The following code illustrates how to configure and run the Llama server:

```
/root/llama.cpp/build/bin/llama-server --host 0.0.0.0 --model /root/llama.cpp/models/granite-20b-code-instruct-8k-GGUF/granite-20b-code-instruct.Q4_K_M.gguf
```

With the Llama.cpp server running on port 8080, we can now perform inference via HTTP requests. In this example, for simplicity, we use curl to make the requests:

```
curl -X POST http://localhost:8080/completion \
     -H "Content-Type: application/json" \
     -d '{
           "prompt": "Make a hello world program in Java. Your answer should be in Java code only.",
           "max_tokens": 100
         }'

```

Below is an example of how the response is returned:

```
{
  "content": "
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

With this setup, we are now able to perform inference on CPU. Our upcoming posts will focus on running these inferences using the HELM (*Holistic Evaluation of Language Models*) framework as the intermediary.
