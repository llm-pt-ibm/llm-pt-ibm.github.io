---
title: "Compilando e Executando o SDG Hub e o Training Hub da Red Hat na IBM Power9"
date: 2026-08-31
authors: ["Lucas Pereira"]
tags: ["LLM", "SDG Hub", "Training Hub", "Power9", "GPU", "Fine-tuning"]
projects: ["multiarq"]
translationKey: "sdg-training-hub"
summary: "Guia para compilar e executar o SDG Hub e o Training Hub da Red Hat na IBM Power9 (ppc64le), abordando a stack de dependências com PyTorch, xFormers, Triton e Ray, além de um ambiente totalmente conteinerizado para geração de dados sintéticos e fine-tuning."
draft: false
---

## Contexto
O [<span class="link-personalizado">*InstructLab*</span>](https://github.com/instructlab) é uma ferramenta para *fine-tuning* de modelos de linguagem de grande porte. Embora ainda esteja em uso, o projeto passou recentemente por uma reorganização. Agora mantido pela *Red Hat*, o *InstructLab* foi dividido em duas ferramentas independentes: o *SDG Hub* (*Synthetic Data Generation Hub*) e o *Training Hub*.

O *SDG Hub* é responsável pela etapa de geração de dados sintéticos, enquanto o *Training Hub* cuida de todo o processo de treinamento dos modelos. O objetivo deste post é demonstrar como compilar e executar essas ferramentas em um sistema IBM Power9, destacando as adaptações necessárias para que funcionem nessa arquitetura, já que algumas de suas dependências ainda não possuem suporte nativo para a plataforma.

## TL;DR
- Compilação e execução do *SDG Hub* na Power9 (*ppc64le*) para geração de dados sintéticos, utilizando o *Conda* para gerenciar *PyArrow*, *Pandas*, *NumPy* e *Matplotlib*.
- Compilação de toda a *stack* de dependências do *Training Hub* na *ppc64le*, incluindo *PyTorch*, *xFormers*, *LLVM*, *Triton*, *TorchVision* e *Ray*.
- Solução para o requisito de *glibc* 2.34 do *Ray*, indisponível no AlmaLinux 8, por meio de uma solução híbrida baseada em *containers*.
- *Fine-tuning* de modelos de linguagem com o *Training Hub* na IBM Power9, utilizando GPUs NVIDIA V100.
- Ambiente totalmente conteinerizado (`build.sh` / `run.sh`) que automatiza a instalação e torna as duas ferramentas reproduzíveis em novos sistemas.

## Ambiente de Execução
O ambiente utilizado para compilar e executar o *SDG Hub* e o *Training Hub* inclui:

- **Arquitetura:** Servidor IBM Power9.
- **Sistema Operacional (SO):** AlmaLinux 8.10 binário compatível com *Red Hat Enterprise Linux (RHEL)* 8.9/8.10.
- **RAM:** 512GB.
- **GPUs:** 4x NVIDIA Tesla V100 SXM2 16GB.

## Entendendo a *Stack* de Dependências
Para o *SDG Hub*, foram necessárias poucas adaptações. Como esse módulo depende majoritariamente de serviços baseados em API para a geração de dados sintéticos, foi suficiente utilizar o *Conda* para instalar as dependências necessárias, incluindo *PyArrow* 25.0.0, *Pandas* 2.3.0, *NumPy* 1.26.4 e *Matplotlib* 3.10.0.

O *Training Hub*, por outro lado, exigiu um trabalho significativamente maior. Suas principais dependências incluem o *PyTorch* 2.7, que fornece o *framework* de *deep learning* utilizado durante o treinamento; o *xFormers* 0.0.35, que oferece implementações otimizadas de mecanismos de atenção; o *LLVM* e o *Triton*, utilizados para compilar *kernels* de GPU altamente otimizados para *hardware* NVIDIA; e o *TorchVision*, exigido por componentes do ecossistema *PyTorch*.

Outra biblioteca que exigiu atenção especial foi o *Ray*, responsável pela execução distribuída e pela orquestração do treinamento. Para a arquitetura *ppc64le*, utilizamos uma *build* fornecida pela IBM, compilada contra a *glibc* 2.34, disponível no AlmaLinux 9, enquanto nosso ambiente alvo executa o AlmaLinux 8. Por conta disso, desenvolvemos uma solução híbrida baseada em *containers* que combina componentes das duas distribuições, garantindo a compatibilidade entre todas as dependências exigidas pelo *Training Hub*.

## Conteinerizando a Solução
Para simplificar a implantação e a reprodutibilidade, desenvolvemos um repositório contendo uma solução totalmente conteinerizada que empacota todas as dependências e configurações necessárias para executar o *SDG Hub* e o *Training Hub* na arquitetura IBM Power9.

Repositório: [<span class="link-personalizado">*sdg_training_hub*</span>](https://github.com/llm-pt-ibm/sdg_training_hub).

O repositório automatiza praticamente todo o processo de instalação e execução. Ao rodar `bash build.sh`, as imagens dos *containers* são construídas e o *Training Hub* 0.9.0 é instalado junto com o *SDG Hub* 0.9.4. Concluída a *build*, o ambiente pode ser iniciado com `bash run.sh`, deixando os módulos de geração de dados sintéticos e de treinamento de modelos imediatamente disponíveis.

{{< figure src="/images/sdg_training_hub_run.gif" alt="Figura 1" caption="Training Hub em execução na arquitetura IBM Power9." >}}

A Figura 1 mostra o *Training Hub* em execução em um servidor IBM Power9.

## Considerações Finais
Este trabalho demonstra como os módulos de geração de dados sintéticos e de treinamento de modelos da *Red Hat* podem ser adaptados para funcionar em servidores IBM Power9. Apesar da falta de suporte oficial para algumas dependências na arquitetura *ppc64le*, as adaptações apresentadas aqui viabilizam o pleno aproveitamento da capacidade computacional da plataforma, utilizando quatro GPUs NVIDIA Tesla V100 tanto para a geração de dados sintéticos quanto para o *fine-tuning* de modelos de linguagem. Além disso, a solução conteinerizada torna o ambiente fácil de reproduzir e simplifica a implantação em novos sistemas.

## *Disclaimer*
Este trabalho não é um lançamento oficial da IBM nem uma distribuição de *software*, e não é desenvolvido ou suportado pela IBM.

Este trabalho foi desenvolvido pela Universidade Federal de Campina Grande (UFCG), universidade pública brasileira, como parte de um projeto de Pesquisa, Desenvolvimento e Inovação conduzido em parceria com a IBM e a Flex Brazil.
