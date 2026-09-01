---
title: "Porting MLX to the IBM Power9 (ppc64le): from build to LLM inference"
date: 2026-08-31
authors: ["João Ramalho"]
projects: ["multiarq"]
translationKey: "mlx-cpu"
tags: ["MLX", "Power9", "ppc64le", "LLM", "Inference", "Portability"]
summary: "This post documents the port of MLX, Apple's array framework, to the IBM POWER9 (ppc64le) architecture — from a patch-free build to real LLM inference, covering the portability issues found along the way and how they were solved."
---

## Context

This post presents the process of adapting [MLX](https://github.com/ml-explore/mlx) to the IBM POWER9 (ppc64le) architecture. It covers how the framework works internally, the portability issues we found, the solutions applied, and the end result: an LLM generating text through MLX on a POWER9 machine.

MLX is an array framework for machine learning developed by Apple, with an API inspired by NumPy and PyTorch. It was designed for Apple Silicon — its primary backend is Metal, Apple's GPU API — but at its core it is a C++ library with interchangeable backends (Metal, CUDA, and CPU). On top of that core sits a growing inference ecosystem, including [mlx-lm](https://github.com/ml-explore/mlx-lm) (LLM inference) and projects such as [exo](https://github.com/exo-explore/exo), which uses MLX as its backend for distributed inference.

The motivation came precisely from exo: while evaluating its viability on the Multiarq project's POWER9, we identified MLX as the critical dependency — and that porting MLX directly would be more valuable than porting exo, since it would unlock the entire ecosystem built on top of it.

The challenge: MLX has **no ppc64le support**. The optimized paths in the CPU backend assume NEON (ARM) or AVX (x86), the kernel-fusion JIT compiles code at runtime assuming a specific toolchain, and no CI in the project tests the architecture.

## TL;DR

- The MLX C++ core (CPU-only) **compiled on ppc64le without a single code patch**, using a conda-forge environment with GCC 14 and OpenBLAS.
- The C++ test suite reached **242/244** (3236/3238 assertions) and the Python suite **721/722** — the 3 remaining failures were attributed to libstdc++, not to the architecture.
- Runtime JIT adaptations were needed: a filter for glibc's `_Float*` typedefs in the preprocessed preamble and a workaround for a hardcoded `g++` in the code.
- Using mlx-lm, a real LLM (SmolLM-135M-Instruct) generated text on the POWER9 at **~15 tokens/s** in float32.
- fp16 falls back to a scalar software implementation (~130x slower than fp32), which makes the published fp16 models impractical — the mitigation is converting models to float32.
- Five portability findings were identified, most of them architecture-independent and potentially contributable upstream.

## Execution Environment

- **Architecture:** IBM Power9 server (ppc64le, AC922).
- **Operating System:** AlmaLinux 8.10.
- **Toolchain:** conda-forge environment with GCC 14, CMake, Ninja, and OpenBLAS (the system's GCC 8.5 does not meet MLX's C++17/20 requirement).
- **Python:** 3.12 (conda-forge).
- **MLX:** built from source, CPU-only (`MLX_BUILD_METAL=OFF`, `MLX_BUILD_CUDA=OFF`).

Environment creation:

```bash
conda create -n mlx-p9 --override-channels -c conda-forge \
  python=3.12 gcc_linux-ppc64le=14 gxx_linux-ppc64le=14 \
  cmake ninja openblas
```

## What MLX is and how it works

Three characteristics of MLX matter for understanding the port:

**Interchangeable backends.** The core defines array operations and delegates execution to a backend. On Apple Silicon that is Metal; on NVIDIA GPUs, CUDA; and there is a complete CPU backend, which is the target of this work. The CPU backend uses OpenBLAS for the heavy linear algebra — and OpenBLAS ships optimized POWER9 kernels, which guarantees competitive fp32 performance.

**SIMD layer with a generic fallback.** The CPU backend's elementwise operations go through a `Simd<T, N>` abstraction with NEON and AVX implementations — plus a generic scalar fallback (`base_simd`) for architectures without a dedicated path. That fallback is what allows MLX to compile on ppc64le with no intrinsics work: everything runs, just scalar. The same layer implements fp16/bf16 in software when the compiler offers no native support — a detail that will become central to the performance analysis.

**Kernel-fusion JIT.** For composite operations (`mx.compile`, automatic fusions), MLX generates C++ code at runtime and compiles it into a dynamically loaded shared library. To do so, at build time it preprocesses a "preamble" containing the required headers and, at runtime, invokes a compiler to combine the preamble with the generated kernel. This mechanism — a two-phase build with a compiler in the runtime loop — was the source of the most interesting problems in the port.

## Challenges and Adaptations

### The build itself: no patches needed

The part that is usually the most painful turned out to be the smoothest: the C++ core compiled on the first attempt, with no code modifications. The `base_simd` fallback covered the absence of a VSX path, and conda-forge's OpenBLAS provided the BLAS. This says something about the quality of MLX's engineering: the architecture abstraction layer works even on an architecture the project has never tested.

The problems showed up later — in the tests and at runtime.

### JIT (1): the wrong compiler at runtime

The first `compile`/fusion tests failed with cascading compilation errors. The cause: MLX invokes a **bare `g++` from PATH** at runtime (hardcoded in `jit_compiler.cpp`). In our environment, that meant the system's GCC 8.5 trying to consume a preamble preprocessed by conda's GCC 14 — headers from one compiler version being interpreted by another.

This is an architecture-independent problem: any installation through conda, Spack, or environment modules — where the build compiler is not the system `g++` — is exposed to it. The workaround was a *shim* on PATH pointing `g++` to the conda compiler (`powerpc64le-conda-linux-gnu-g++`). The proper fix, a PR candidate, would be for MLX to record the compiler used at build time (via `CMAKE_CXX_COMPILER`) and invoke that one at runtime.

### JIT (2): glibc's `_Float*` typedefs

With the compiler sorted, one genuinely ppc64le error remained: in the preprocessed preamble, glibc emits typedefs for `_Float32`, `_Float64`, `_Float128`, `_Float32x`, and `_Float64x` — which on this architecture **redeclare GCC built-in types**, breaking the compilation of every JIT kernel.

The first mitigation was compiling the kernels with `-fpermissive`, which downgrades the error to a warning. It worked, but it is the wrong solution: it silences an entire class of errors to work around five lines. The clean fix was filtering the typedefs during preamble generation (`make_compiled_preamble.sh`), with a `sed` in the preprocessing pipe (`-E -P`). With the filter in place, `-fpermissive` was reverted and the result held: 242/244.

### RPATH: `lib` vs `lib64`

In the Python bindings, MLX assumes an RPATH of `$ORIGIN/lib`, but CMake's `GNUInstallDirs` installs to `lib64` on Red Hat family systems — the library gets installed in one place and searched for in another. Worked around with `-DCMAKE_INSTALL_LIBDIR=lib` in `CMAKE_ARGS`. This is also an architecture-independent finding, affecting any RHEL-like distro.

### The three failures that are not MLX's

In the end, 3 failures remained out of 966 tests — two in C++ (`exp` and `logaddexp` of complex64 with `-inf`) and one in Python (`power(0j, nan)`). All involving complex numbers, all with the same pattern. The investigation isolated the cause outside MLX and outside the architecture: **in the same binary, with the same toolchain**, the compiler builtin gets it right while libstdc++ gets it wrong:

| Operation | `__builtin_cexpf` / `__builtin_cpowf` | `std::exp` / `std::pow` |
| --- | --- | --- |
| `exp(-inf + 2i)` | `0, -0` (correct, Annex G) | `nan` |
| `pow(0j, nan)` | `nan` (correct) | `0` |

A macro dump showed `_GLIBCXX98_USE_C99_COMPLEX` defined but **not** `_GLIBCXX11_USE_C99_COMPLEX` — which makes libstdc++'s `<complex>` fall back to naive implementations (via `polar`) instead of routing to the C99 builtins. In other words: MLX correct, architecture correct, standard library in a degraded configuration mode in the conda-forge build. An x86 control run with the same libstdc++ will tell whether the issue is specific to the ppc64le build.

## Results: an LLM running on the POWER9

With the build validated, we installed mlx-lm (the dependencies with Rust components — `tokenizers`, `safetensors` — came from conda-forge, since PyPI ships no ppc64le wheels) and ran a real model from mlx-community:

```
$ python -m mlx_lm generate \
    --model mlx-community/SmolLM-135M-Instruct-4bit \
    --prompt "..." --max-tokens 50

Generation: 50 tokens, 0.071 tokens-per-sec
```

It worked — but at 0.07 tokens/s, roughly 14 seconds per token. A microbenchmark isolated the cause in a single line:

| Precision | 1024×1024 matmul |
| --- | --- |
| float32 | **169.91 GFLOP/s** |
| float16 | 1.31 GFLOP/s |

A **130x** gap. fp32 goes through OpenBLAS, with optimized POWER9 kernels; fp16 falls into the SIMD layer's scalar software fallback, element by element. And the models published on mlx-community use fp16 activations by default — even the 4-bit quantized ones.

The mitigation is straightforward: convert the model with float32 activations:

```
$ python -m mlx_lm convert --hf-path HuggingFaceTB/SmolLM-135M-Instruct \
    --mlx-path ~/models/smollm-135m-f32 --dtype float32

$ python -m mlx_lm generate --model ~/models/smollm-135m-f32 \
    --prompt "..." --max-tokens 50

Generation: 50 tokens, 14.939 tokens-per-sec
```

**From 0.07 to ~15 tokens/s — a ~210x gain** from changing nothing but the dtype. For a 135M model on CPU, that is perfectly usable performance, and it confirms the bottleneck was never the architecture: it was a single code path (software fp16) in the middle of an otherwise healthy pipeline.

## Final Remarks

The outcome of this port is unusual: **zero patches to compile, 963 of 966 tests passing, and working LLM inference** — with the real problems concentrated not in the compute code, but in the infrastructure around it (runtime JIT, RPATH, standard library) and in one performance path (fp16).

Five findings were identified, and most of them benefit platforms beyond POWER9:

1. **`_Float*` typedefs in the JIT preamble** — ppc64le/glibc-specific; a clean fix is already implemented in `make_compiled_preamble.sh`.
2. **Hardcoded `g++` in the JIT** — architecture-independent; breaks any conda/Spack/modules environment.
3. **RPATH `$ORIGIN/lib` vs `lib64`** — architecture-independent; affects the entire Red Hat family.
4. **Strict-aliasing violation in `FromFP8`** — architecture-independent; visible as a warning under `-O3`.
5. **Software-fallback fp16** — published fp16 models become impractical on architectures without a dedicated SIMD path; the mitigation is converting to float32, and the long-term optimization would be a VSX path in the `Simd<>` layer or vectorized fp16↔fp32 conversion.

## Next Steps

- Report the findings upstream (ml-explore/mlx) as issues and PRs, starting with the `_Float*` typedef filter;
- Measure the combination of 4-bit quantization + float32 compute, which should pair the quantized model's memory savings with fp32 performance;
- Scale up to larger models (Qwen2.5 0.5B/1.5B) and characterize performance against the OpenBLAS thread count;
- Validate exo — the original motivation — on top of the ported MLX;
- Explore MLX's CUDA backend with the machine's V100s (sm_70), assessing the coverage of kernels that assume bf16 (sm_80+);
- Investigate a VSX path for the `Simd<>` layer as a performance contribution upstream.

Disclaimer
This work is not an official IBM release or software distribution and is not developed or supported by IBM.

This work was developed by the Federal University of Campina Grande (UFCG), a Brazilian public university, as part of a Research, Development, and Innovation project conducted in partnership with IBM and Flex Brazil.