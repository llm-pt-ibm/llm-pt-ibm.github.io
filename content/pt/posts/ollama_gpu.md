---
title: "Inferência de LLMs com Ollama na IBM Power9 Utilizando GPU"
date: 2026-04-16 # ano-mês-dia
authors: ["Maria Luísa Gomes"] # Pode ser uma lista
tags: ["LLM", "Ollama", "Power9", "GPU", "Inferência"]
projects: ["multiarq"]
translationKey: "ollama-gpu"
summary: "Este post documenta como executar inferência de LLMs com Ollama na IBM Power9 utilizando GPU, com foco em um fork compatível com NVIDIA Tesla V100 em POWER9."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto
Este é o segundo post da série sobre inferência de modelos de linguagem na POWER9 com o [<span class="link-personalizado">*Ollama*</span>](https://ollama.com/). Neste post, abordaremos como enviar requisições utilizando GPU, obtendo um ganho significativo de desempenho em relação à abordagem via CPU apresentada no [<span class="link-personalizado">*post anterior*</span>](https://llm-pt-ibm.github.io/posts/ollama_cpu/).

O principal desafio é que o Ollama não oferece suporte oficial para a arquitetura *ppc64le* com [<span class="link-personalizado">*CUDA*</span>](https://developer.nvidia.com/cuda-12-2-0-download-archive?target_os=Linux&target_arch=ppc64le&Distribution=RHEL&target_version=8&target_type=rpm_local). A solução encontrada foi através de um blog da  [<span class="link-personalizado">*comunidade oficial IBM*</span>](https://community.ibm.com/community/user/blogs/andrey-klyachkin/2025/03/06/run-ollama-on-almalinux-ppc64le-ibm-power), onde um contribuidor disponibilizou um [<span class="link-personalizado">*fork*</span>](https://github.com/naveedus/ollama-ppc64le) do Ollama adaptado para suportar GPUs NVIDIA na POWER9 via CUDA, voltado especificamente para o IBM Power AC922 com Tesla V100. No entanto, o simples uso do repositório não é suficiente — é necessário compilar corretamente para que o Ollama consiga detectar e utilizar a GPU. Este tutorial explica exatamente como fazer esse processo.

## TL;DR
- Este post apresenta detalhes sobre a configuração do ambiente para realizar inferências utilizando a infraestrutura da IBM POWER9;
- O Ollama não oferece suporte oficial para *ppc64le* com CUDA;
- A solução foi utilizar um fork desenvolvido por um contribuidor da IBM, encontrado em um blog oficial da comunidade IBM;
- O fork foi compilado do zero utilizando `make`, apontando para CUDA 12.2 e especificando a arquitetura do V100 (`sm_70`);
- Com isso, foi possível executar inferência de LLMs na IBM POWER9 com aceleração GPU.

## Ambiente utilizado

**Hardware**:
- Arquitetura *ppc64le*;
- RAM: mínimo recomendado de ~64GB;
- GPU: NVIDIA Tesla V100;
- *Driver* NVIDIA: 535.54.03;
- CUDA: versão 12.2.

**Sistema Operacional:** Alma Linux 8.10 (*ppc64le*), binário compatível com *Red Hat Enterprise Linux (RHEL)* 8.9/8.10.

## Verificações iniciais

1. Verificar se o driver e a GPU estão visíveis
```bash
nvidia-smi
```

2. Verificar se o CUDA está instalado
```bash
nvcc --version
```

OBS: Se não aparecer nada, tente:

```bash
export PATH=/usr/local/cuda-12.2/bin:$PATH
export CUDACXX=/usr/local/cuda-12.2/bin/nvcc
```

3. Verifique também se o CUDA 12 existe:

```bash
ls -la /usr/local/cuda-12
```

## Execução em Ambiente Virtual

Neste tutorial, estamos fazendo as configurações necessárias em um ambiente virtual para isolar o ambiente de execução e as configurações utilizadas. Essa execução é opcional, mas recomendada.

```bash
conda create -n ollamaGPU python=3.11 -y
conda activate ollamaGPU
```

Para desativar o ambiente:

```bash
conda deactivate
```

## *Setup* inicial
Para executar o Ollama na arquitetura POWER9, é necessário preparar o ambiente com as dependências adequadas. O primeiro passo é atualizar o sistema e instalar os utilitários básicos:

```bash
sudo dnf update -y
sudo dnf install -y wget git tar make gcc gcc-c++ cmake gcc-toolset-11
```

Embora esse comando instale parte das dependências, é necessário garantir que as versões corretas estejam sendo utilizadas.

### Configuração do *Go*
O Ollama é desenvolvido em *Go*, portanto é necessário garantir a versão adequada.

**Versão esperada:** 1.25.7.

#### Caso não esteja instalado ou esteja com uma versão diferente:

```bash
wget https://go.dev/dl/go1.25.7.linux-ppc64le.tar.gz
sudo tar -C /usr/local -xzf go1.25.7.linux-ppc64le.tar.gz
export PATH=/usr/local/go/bin:$PATH
```

Para adicionar ao *PATH* permanentemente:

```bash
echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verifique se a versão está correta: `go version`

### Configuração do *CMake*
Verifique se a versão está correta: `cmake --version`

**Versão esperada:** 3.26.5.

#### Caso necessário, realize a instalação manual:

```bash
wget https://github.com/Kitware/CMake/releases/download/v3.26.5/cmake-3.26.5.tar.gz
tar -xzf cmake-3.26.5.tar.gz
cd cmake-3.26.5
./bootstrap
make -j$(nproc)
sudo make install
```

### Configuração do *GCC*

**Versão esperada:** 11.2.1.

**Importante:** No AlmaLinux 8, o `gcc-toolset` não é ativado automaticamente. A forma de ativá-lo depende se você está usando ambiente virtual conda ou não:

Sem ambiente virtual conda:

```bash
scl enable gcc-toolset-11 bash
```

Com ambiente virtual conda ativo: o comando `scl enable` abre um *subshell* que conflita com o conda e corrompe o `PATH`. Use o `export` manual:

```bash
export PATH=/opt/rh/gcc-toolset-11/root/usr/bin:$PATH
export LD_LIBRARY_PATH=/opt/rh/gcc-toolset-11/root/usr/lib64:$LD_LIBRARY_PATH
```

Em ambos os casos, verifique a versão:

```bash
gcc --version
```

Importante: a ativação do GCC precisa ser feita toda vez que abrir um novo terminal, antes de compilar.

## Clonando e compilando o Ollama

Com o ambiente configurado, podemos realizar o *build* do Ollama. Aqui vamos clonar o repositório do *fork* de um contribuidor da IBM, que fez um repositório com foco para a GPU. Esse *fork* foi encontrado em um *blog* oficial da comunidade IBM e contém os ajustes necessários para que o *build* e a detecção da GPU funcionem corretamente na arquitetura da POWER9:

```bash
git clone https://github.com/naveedus/ollama-ppc64le.git ollama-gpu
cd ollama-gpu
git checkout witherspoon
```

Verifique se está no branch correto: `git branch` (deve aparecer `witherspoon`).

### Compilando com suporte a CUDA:

```bash
make CUDA_ARCHITECTURES="70"
```

No tutorial anterior, quando compilamos o Ollama para a CPU, usamos o `go build`, que compila apenas o código Go do Ollama, sem as bibliotecas CUDA do `llama.cpp`. Aqui precisamos utilizar uma abordagem diferente: usar o `make` para compilar os *kernels* do CUDA com `nvcc`, gerar o *runner* CUDA como biblioteca compartilhada e só então chamar o `go build`.

Por que `CUDA_ARCHITECTURES="70"`? Cada GPU NVIDIA possui uma arquitetura específica, identificada por um código `sm_XX`. O V100 é da arquitetura Volta, cujo código é `sm_70`. Por padrão, o *build* compila para todas as arquiteturas suportadas pelo CUDA (sm_60 até sm_90), o que aumenta o tempo de compilação significativamente. Ao especificar `CUDA_ARCHITECTURES="70"`, instruímos o *build* a compilar apenas para o V100, tornando o processo um pouco mais rápido.

O *build* vai demorar alguns minutos. Ao finalizar, verifique se o *runner* CUDA foi gerado:

```bash
ls -lh llama/build/linux-ppc64le/runners/cuda_v12/
```

Você deve ver dois arquivos: `libggml_cuda_v12.so` e `ollama_llama_server`. Confirme que o runner está linkado com CUDA:

```bash
ldd llama/build/linux-ppc64le/runners/cuda_v12/libggml_cuda_v12.so | grep -i cuda
```

A saída deve mostrar `libcuda.so`, `libcublas.so`, `libcudart.so` e `libcublasLt.so`.

## Realizando a inferência

Com o Ollama compilado, podemos iniciar o servidor:

```bash
./ollama serve
```

Para verificar se deu certo: `ps aux | grep ollama`.

Em outro terminal, aguarde alguns segundos e verifique os logs para confirmar que o servidor detectou as GPUs corretamente. Procure por estas linhas:

```bash
Dynamic LLM libraries runners="[cpu cuda_v12]"
inference compute ... library=cuda compute=7.0 ... total="15.0 GiB"
```

## Baixar o modelo de teste e executar a inferência

Para validação, utilizamos o modelo `llama3.1:8b`. Para isso, em outro terminal, rode:

```bash
./ollama pull llama3.1:8b
```

Para executar a inferência:

```bash
./ollama run llama3.1:8b "me fale todos os números ímpares até 100"
```

## Confirmar o uso da GPU

Em outro terminal, com a inferência em execução, rode:

```bash
nvidia-smi
```

Na seção de processos, você deve ver o `ollama_llama_server` com memória alocada em uma das GPUs:

{{< figure src="/images/ollama_inference_gpu.png" alt="Figura 1" caption="Ollama usando a GPU">}}

## Considerações finais

Com os passos apresentados, foi possível configurar o ambiente para executar inferências de LLMs em uma máquina IBM POWER9, utilizando as GPUs NVIDIA Tesla V100. Com essa abordagem, a inferência de modelos possui um ganho de desempenho significativo em relação à execução via CPU. Utilizando o modelo Meta Llama 3.1 8B Instruct como referência, a execução via GPU atingiu uma maior geração de tokens por segundo em relação à execução via CPU.

Vejamos os dados coletados para uma mesma requisição (`Me fale todos os números ímpares até 100`) com os dois tipos de execução:

|          | CPU         | GPU         |
|----------|-------------|-------------|
| Taxa de geração de *tokens* | 0.71 tokens/s | 79.82 tokens/s |
| Duração total | 3m49s | 4.52s |
| Taxa de avaliação do *prompt* | 10.67 tokens/s | 295.77 tokens/s |

Com os dados apresentados na tabela, percebemos que a execução com GPU foi aproximadamente 112 vezes mais rápida na geração de *tokens*, com o tempo total de resposta reduzido de 3 minutos e 49 segundos para 4.52 segundos.

## Próximos Passos

- Avaliar a execução com GPU e CPU em um post comparativo e com outras arquiteturas;
- Testar a inferência em GPU com modelos maiores, com mais de 8 bilhões de parâmetros, por exemplo;
- Compilar o Ollama para uma versão mais recente, para carregar modelos novos e avaliar o desempenho;
