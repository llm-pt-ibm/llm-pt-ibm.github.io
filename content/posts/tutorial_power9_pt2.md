---
title: "Configurando Conda e PyTorch em um servidor IBM Power9"
date: 2025-06-29 # ano-mês-dia
authors: ["Caio Silva"] # Pode ser uma lista
tags: ["LLM", "Power9", "Conda", "PyTorch"]
summary: "Este post faz parte de uma série de tutoriais que tem como objetivo final construir uma API de Modelos de Linguagem em servidor Power9. Nesta etapa vamos configurar o gerenciador de pacotes Conda e a biblioteca PyTorch."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto
Este é o segundo post de uma série de tutoriais que vamos mostrar o passo-a-passo de como construir uma API de Modelos de Linguagem em um servidor Power9, desde da configuração do Sistema Operacional, até a API executando inferências de forma remota. O [primeiro post]({{< relref "tutorial_power9_pt1.md" >}}) mostra como instalar o S.O e configurar drivers NVIDIA, CUDA e CUDNN. Nesta etapa do tutorial vamos mostrar a configuração do gerenciador de pacotes Conda e da biblioteca PyTorch


**Conda**: Conda é um sistema de gerenciamento de pacotes e ambientes de código aberto e multiplataforma. Ele funciona como uma "caixa de ferramentas" para cientistas de dados e desenvolvedores, ajudando a organizar seus projetos.

**PyTorch**: PyTorch é uma biblioteca de código aberto para aprendizado de máquina, desenvolvida principalmente pelo Facebook AI Research (FAIR). Ela é especialmente popular para o desenvolvimento de aplicações de deep learning (aprendizado profundo), um subcampo do aprendizado de máquina que se inspira no funcionamento do cérebro humano.


## TLDR
- Este post apresenta o passo-a-passo para a instalação do Conda e PyTorch. 
- O desafio maior é encontrar versões compatíveis com a arquitetura das máquinas Power.

## Configurando Conda
Vamos começar com a instalação do **Conda**. Em sistemas Power, a arquitetura usada é a ppc64le (PowerPC 64 bits little-endian), por isso é essencial que a versão baixada seja para esta arquitetura. Para isso, vamos utilizar o **miniconda**, uma versão mais leve e direta para setups customizados como o servidor Power9.

1. Para baixar e instalar a versão mais atualizada do miniconda:

```
sudo wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-ppc64le.sh
bash ~/Miniconda3-latest-Linux-ppc64le.sh
```

2. Verifique se a instalação ativou o conda automático:
```
conda –version
```
Caso não tenha iniciado automaticamente, o Conda precisa ser ativado. 


3. Para  não precisar ativar sempre que realizar uma nova conexão, vamos escrever o comando no bashrc (ou zshrc):

```
echo 'source ~/miniconda3/etc/profile.d/conda.sh' >> ~/.bashrc
source ~/.bashrc
```
Verifique novamente com o comando: 
```
conda –version
```
A saída esperada é algo semelhante a: ```conda 23.10.0```

## Instalando e configurando a biblioteca PyTorch
Não existem builds oficiais ou wheels Conda/PyPi com suporte completo para a arquitetura ppc64le, sendo assim, para instalar o PyTorch precisamos buildar manualmente.

#### (Opcional) Criação de ambiente virtual Conda
Para iniciarmos a instalação é aconselhável criar um ambiente virtual para instalar o pytorch apenas nele. 
1. Para criar e ativar o ambiente virtual executamos os comandos:
```
conda create -y -n api_llm python=3.10
conda create activate api_llm
```

#### Instalando pré-requisitos
Precisamos instalar alguns pacotes necessários para realizar o build do PyTorch da forma correta.

1. Inicialmente, vamos instalar os pacotes com os seguintes comandos: 

```
conda install -y -c conda-forge openblas libblas cmake ninja python3-devel gcc-c++ rust cargo
```
O CMake (sistema de build utilizado pelo PyTorch) removeu o suporte a scripts que declaram compatibilidade com versões antigas (<3.5). Para resolver isso, precisamos instalar via pip uma versão do cmake <3.5. 

2. Executamos o comando:
```
pip install cmake==3.27.7
```

Para garantir que a versão correta foi instalada, executamos o comando:
```
cmake --version 
```

A saída esperada é: ```cmake version 3.27.7```

#### Build do Pytorch
Agora vamos iniciar o processo de build do **PyTorch**.

1. O primeiro passo é clonar o repositório e configurar para instalar a versão 2.6.0:

```
git clone --recursive https://github.com/pytorch/pytorch
cd pytorch
git checkout v2.6.0 
git submodule sync 
git submodule update --init --recursive 
```

2. Para instalar os pacotes necessários via pip executamos o seguinte comando:
```
pip install -r requirements.txt
```
3. E, finalmente, para realizar o build do PyTorch, executamos o setup.py do python:
```
sudo USE_CUDA=1 USE_DISTRIBUTED=1 USE_NCCL=1 USE_GLOO=1 USE_CUDNN=1 python setup.py install
```
O processo de build geralmente demora um tempo considerável, cerca de 15 minutos.

4. Para testar se tudo ocorreu certo, vamos criar um arquivo chamado ```test_torch.py```

```
nano test_torch.py
```
Esse arquivo deve conter as seguintes linhas: 
```python {linenos=inline}
import torch
print(torch.__version__)
print("CUDA disponível:", torch.cuda.is_available())
print("Número de GPUs:", torch.cuda.device_count())
print("Nome da GPU:", torch.cuda.get_device_name(0))
x = torch.rand(3, 3).cuda()
y = torch.rand(3, 3).cuda()
print("Soma na GPU:", (x + y))
print("cuDNN disponível:", torch.backends.cudnn.is_available())
print("Extensões C carregadas:", torch._C._cuda_getDeviceCount() > 0)
```

Ao executar esse arquivo, saberemos:
- Versão instalada do pytorch
- Disponibilidade do CUDA
- Quantidade de GPUs disponíveis
- Nome da GPU no servidor Power9
- Se a utilização da GPU está acontecendo de forma correta
- Disponibilidade do CUDNN
- Se os arquivos .so foram compilados corretamentes

5. Vamos executar o arquivo com o comando:
```
python test_gpu.py
```

A saída deve ser algo semelhante a: 
```bash
2.6.0a0+git1eba9b3
CUDA disponível: True
Número de GPUs: 4
Nome da GPU: Tesla V100-SXM2-16GB
Soma na GPU: tensor([[1.9163, 1.2208, 0.5998],
        [1.7962, 0.6040, 1.3943],
        [0.9536, 0.8010, 0.0668]], device='cuda:0')
cuDNN disponível: True
Extensões C carregadas: True
```

É importante lembrar que as saídas podem ser diferentes em relação ao número e modelo das GPUs e a soma de tensores (devido a aleatoriedade). É importante que as saídas booleanas do código que executamos tenham resultados igual a ```True```.

Com isso, a biblioteca PyTorch está instalada e configurada para ser utilizada. No próximo tutorial vamos realizar a primeira inferência de um Modelo de Linguagem no servidor Power9.
