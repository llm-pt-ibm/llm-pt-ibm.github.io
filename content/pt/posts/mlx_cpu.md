---
title: "Portando o MLX para a IBM Power9 (ppc64le): do build à inferência de LLM"
date: 2026-08-31
authors: ["João Ramalho"]
projects: ["multiarq"]
translationKey: "mlx-cpu"
tags: ["MLX", "Power9", "ppc64le", "LLM", "Inferência", "Portabilidade"]
summary: "Este post documenta o porte do MLX, o framework de arrays da Apple, para a arquitetura IBM POWER9 (ppc64le) — do build sem patches à inferência de um LLM real, passando pelos problemas de portabilidade encontrados e suas soluções."
---

## Contexto

Este post apresenta o processo de adaptação do [MLX](https://github.com/ml-explore/mlx) para a arquitetura IBM POWER9 (ppc64le). Serão abordados o funcionamento interno do framework, os problemas de portabilidade encontrados, as soluções aplicadas e o resultado final: um LLM gerando texto via MLX em uma máquina POWER9.

O MLX é um framework de arrays para machine learning desenvolvido pela Apple, com API inspirada no NumPy e no PyTorch. Ele foi projetado para Apple Silicon — seu backend principal é Metal, a API de GPU da Apple — mas sua base é um core C++ com backends intercambiáveis (Metal, CUDA e CPU). Sobre esse core existe um ecossistema crescente de inferência, incluindo o [mlx-lm](https://github.com/ml-explore/mlx-lm) (inferência de LLMs) e projetos como o [exo](https://github.com/exo-explore/exo), que usa o MLX como backend para inferência distribuída.

A motivação surgiu justamente pelo exo: ao avaliar sua viabilidade no POWER9 do projeto Multiarq, identificamos que o MLX era a dependência crítica — e que portar o MLX diretamente seria mais valioso do que portar o exo, já que destravaria todo o ecossistema construído sobre ele.

O desafio: o MLX **não oferece suporte a ppc64le**. Os caminhos otimizados do backend CPU assumem NEON (ARM) ou AVX (x86), o JIT de fusão de kernels compila código em tempo de execução assumindo um toolchain específico, e nenhum CI do projeto testa a arquitetura.

## TL;DR

- O core C++ do MLX (CPU-only) **compilou em ppc64le sem nenhum patch de código**, usando um ambiente conda-forge com GCC 14 e OpenBLAS.
- A suíte de testes C++ atingiu **242/244** (3236/3238 assertions) e a suíte Python **721/722** — as 3 falhas restantes foram atribuídas ao libstdc++, não à arquitetura.
- Foram necessárias adaptações no JIT em tempo de execução: um filtro de typedefs `_Float*` da glibc no preâmbulo pré-processado e um contorno para o `g++` hardcoded no código.
- Com o mlx-lm, um LLM real (SmolLM-135M-Instruct) gerou texto na POWER9 a **~15 tokens/s** em float32.
- O fp16 cai em fallback escalar em software (~130x mais lento que fp32), o que torna os modelos fp16 publicados impraticáveis — a mitigação é converter os modelos para float32.
- Cinco achados de portabilidade foram identificados, a maioria independente de arquitetura e potencialmente contribuível ao upstream.

## Ambiente de Execução

- **Arquitetura:** Servidor IBM Power9 (ppc64le, AC922).
- **Sistema Operacional:** AlmaLinux 8.10.
- **Toolchain:** ambiente conda-forge com GCC 14, CMake, Ninja e OpenBLAS (o GCC 8.5 do sistema não atende ao requisito de C++17/20 do MLX).
- **Python:** 3.12 (conda-forge).
- **MLX:** compilado do fonte, CPU-only (`MLX_BUILD_METAL=OFF`, `MLX_BUILD_CUDA=OFF`).

Criação do ambiente:

```bash
conda create -n mlx-p9 --override-channels -c conda-forge \
  python=3.12 gcc_linux-ppc64le=14 gxx_linux-ppc64le=14 \
  cmake ninja openblas
```

## O que é o MLX e como ele funciona

Três características do MLX importam para entender o porte:

**Backends intercambiáveis.** O core define as operações sobre arrays e delega a execução a um backend. No Apple Silicon é o Metal; em GPUs NVIDIA, o CUDA; e existe um backend CPU completo, que é o alvo deste trabalho. O backend CPU usa OpenBLAS para as operações de álgebra linear pesadas — e o OpenBLAS possui kernels otimizados para POWER9, o que garante desempenho competitivo em fp32.

**Camada SIMD com fallback genérico.** As operações elementwise do backend CPU passam por uma abstração `Simd<T, N>` com implementações NEON e AVX — e um fallback escalar genérico (`base_simd`) para arquiteturas sem caminho dedicado. É esse fallback que permite ao MLX compilar em ppc64le sem intrinsics VSX: tudo funciona, mas de forma escalar. A mesma camada implementa fp16/bf16 em software quando não há suporte nativo do compilador — um detalhe que se tornará central na análise de desempenho.

**JIT de fusão de kernels.** Para operações compostas (`mx.compile`, fusões automáticas), o MLX gera código C++ em tempo de execução e o compila em uma biblioteca compartilhada carregada dinamicamente. Para isso, no momento do build ele pré-processa um "preâmbulo" com os headers necessários e, em runtime, invoca um compilador para juntar preâmbulo e kernel gerado. Esse mecanismo — build em duas fases, com um compilador em tempo de execução — foi a origem dos problemas mais interessantes do porte.

## Desafios e Adaptações

### O build em si: nenhum patch necessário

A parte que costuma ser a mais dolorosa foi a mais tranquila: o core C++ compilou de primeira, sem nenhuma modificação de código. O fallback `base_simd` cobriu a ausência de caminho VSX, e o OpenBLAS do conda-forge forneceu o BLAS. Esse resultado diz algo sobre a qualidade da engenharia do MLX: a camada de abstração de arquitetura funciona mesmo em uma arquitetura que o projeto nunca testou.

Os problemas apareceram depois — nos testes e em tempo de execução.

### JIT (1): o compilador errado em runtime

Os primeiros testes de `compile`/fusão falharam com erros de compilação em cadeia. A causa: o MLX invoca `g++` **puro, do PATH**, em tempo de execução (hardcoded em `jit_compiler.cpp`). No nosso ambiente, isso significava o GCC 8.5 do sistema tentando consumir um preâmbulo pré-processado pelo GCC 14 do conda — headers de uma versão de compilador interpretados por outra.

Este é um problema independente de arquitetura: qualquer instalação via conda, Spack ou environment modules — onde o compilador do build não é o `g++` do sistema — está sujeita a ele. O contorno foi um *shim* no PATH apontando `g++` para o compilador do conda (`powerpc64le-conda-linux-gnu-g++`). A correção adequada, candidata a PR, seria o MLX registrar em build time o compilador usado (via `CMAKE_CXX_COMPILER`) e invocá-lo em runtime.

### JIT (2): os typedefs `_Float*` da glibc

Resolvido o compilador, restou um erro genuinamente ppc64le: no preâmbulo pré-processado, a glibc emite typedefs para `_Float32`, `_Float64`, `_Float128`, `_Float32x` e `_Float64x` — que nessa arquitetura **redeclaram tipos built-in do GCC**, quebrando a compilação de todo kernel JIT.

A primeira mitigação foi compilar os kernels com `-fpermissive`, que rebaixa o erro a warning. Funcionou, mas é a solução errada: silencia uma classe inteira de erros para contornar cinco linhas. A solução limpa foi filtrar os typedefs na geração do preâmbulo (`make_compiled_preamble.sh`), com um `sed` no pipe do pré-processamento (`-E -P`). Com o filtro, o `-fpermissive` foi revertido e o resultado se manteve: 242/244.

### RPATH: `lib` vs `lib64`

Nos bindings Python, o MLX assume RPATH `$ORIGIN/lib`, mas o `GNUInstallDirs` do CMake instala em `lib64` em sistemas da família Red Hat — a biblioteca é instalada num lugar e procurada em outro. Contornado com `-DCMAKE_INSTALL_LIBDIR=lib` nos `CMAKE_ARGS`. Também é um achado independente de arquitetura, que afeta qualquer distro RHEL-like.

### As três falhas que não são do MLX

Ao final, restaram 3 falhas em 966 testes — duas em C++ (`exp` e `logaddexp` de complex64 com `-inf`) e uma em Python (`power(0j, nan)`). Todas de números complexos, todas com o mesmo padrão. A investigação isolou a causa fora do MLX e fora da arquitetura: **no mesmo binário e mesma toolchain**, o builtin do compilador acerta enquanto a libstdc++ erra:

| Operação | `__builtin_cexpf` / `__builtin_cpowf` | `std::exp` / `std::pow` |
| --- | --- | --- |
| `exp(-inf + 2i)` | `0, -0` (correto, Annex G) | `nan` |
| `pow(0j, nan)` | `nan` (correto) | `0` |

O dump de macros mostrou `_GLIBCXX98_USE_C99_COMPLEX` definido, mas **não** `_GLIBCXX11_USE_C99_COMPLEX` — o que faz o `<complex>` da libstdc++ cair em implementações ingênuas (via `polar`) em vez de rotear para os builtins C99. Ou seja: MLX correto, arquitetura correta, biblioteca padrão com um modo de configuração degradado no build do conda-forge. Um controle em x86 com a mesma libstdc++ dirá se o problema é específico do build ppc64le.

## Resultados: um LLM rodando na POWER9

Com o build validado, instalamos o mlx-lm (as dependências com componentes Rust — `tokenizers`, `safetensors` — vieram do conda-forge, já que o PyPI não tem wheels ppc64le) e rodamos um modelo real da mlx-community:

```
$ python -m mlx_lm generate \
    --model mlx-community/SmolLM-135M-Instruct-4bit \
    --prompt "..." --max-tokens 50

Generation: 50 tokens, 0.071 tokens-per-sec
```

Funcionou — mas a 0,07 tokens/s, cerca de 14 segundos por token. Um microbenchmark isolou a causa em uma linha:

| Precisão | Matmul 1024×1024 |
| --- | --- |
| float32 | **169,91 GFLOP/s** |
| float16 | 1,31 GFLOP/s |

Uma diferença de **130x**. O fp32 vai para o OpenBLAS, com kernels POWER9 otimizados; o fp16 cai no fallback escalar em software da camada SIMD, elemento a elemento. E os modelos publicados na mlx-community usam fp16 nas ativações por padrão — mesmo os quantizados em 4 bits.

A mitigação é direta: converter o modelo com as ativações em float32:

```
$ python -m mlx_lm convert --hf-path HuggingFaceTB/SmolLM-135M-Instruct \
    --mlx-path ~/models/smollm-135m-f32 --dtype float32

$ python -m mlx_lm generate --model ~/models/smollm-135m-f32 \
    --prompt "..." --max-tokens 50

Generation: 50 tokens, 14.939 tokens-per-sec
```

**De 0,07 para ~15 tokens/s — um ganho de ~210x** trocando apenas o dtype. Para um modelo de 135M em CPU, é um desempenho perfeitamente utilizável, e confirma que o gargalo nunca foi a arquitetura: era um único caminho de código (fp16 em software) no meio de um pipeline saudável.

## Considerações Finais

O balanço do porte é incomum: **zero patches para compilar, 963 de 966 testes passando, e inferência de LLM funcional** — com os problemas reais concentrados não no código de computação, mas na infraestrutura ao redor (JIT em runtime, RPATH, biblioteca padrão) e em um caminho de desempenho (fp16).

Cinco achados foram identificados, e a maioria beneficia outras plataformas além do POWER9:

1. **Typedefs `_Float*` no preâmbulo do JIT** — específico de ppc64le/glibc; correção limpa já implementada no `make_compiled_preamble.sh`.
2. **`g++` hardcoded no JIT** — independente de arquitetura; quebra qualquer ambiente conda/Spack/modules.
3. **RPATH `$ORIGIN/lib` vs `lib64`** — independente de arquitetura; afeta toda a família Red Hat.
4. **Violação de strict-aliasing em `FromFP8`** — independente de arquitetura; visível como warning sob `-O3`.
5. **fp16 em fallback de software** — os modelos fp16 publicados ficam impraticáveis em arquiteturas sem caminho SIMD dedicado; a mitigação é a conversão para float32, e a otimização de longo prazo seria um caminho VSX na camada `Simd<>` ou conversão fp16↔fp32 vetorizada.

## Próximos Passos

- Reportar os achados ao upstream (ml-explore/mlx) em formato de issues e PRs, começando pelo filtro dos typedefs `_Float*`;
- Medir a combinação quantização 4-bit + computação em float32, que deve unir a economia de memória do quantizado ao desempenho do fp32;
- Escalar para modelos maiores (Qwen2.5 0.5B/1.5B) e caracterizar o desempenho com o número de threads do OpenBLAS;
- Validar o exo — a motivação original — sobre o MLX portado;
- Explorar o backend CUDA do MLX com as V100 da máquina (sm_70), avaliando a cobertura dos kernels que assumem bf16 (sm_80+);
- Investigar um caminho VSX para a camada `Simd<>` como contribuição de desempenho ao upstream.

## Disclaimer
This work is not an official IBM release or software distribution and is not developed or supported by IBM.

This work was developed by the Federal University of Campina Grande (UFCG), a Brazilian public university, as part of a Research, Development, and Innovation project conducted in partnership with IBM and Flex Brazil.