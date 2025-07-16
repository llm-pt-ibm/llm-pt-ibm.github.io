---
title: "Evaluating Small-Scale LLMs (up to 8B) on PT-BR Benchmarks"
date: 2025-06-02 
authors: ["Lucas Pereira"]
tags: ["LLM", "HELM", "PT-BR"]
translationKey: "experimentos_benchmarks_pt_br"
summary: "In this post, we present the results of evaluating small-scale LLMs on sentiment analysis and MQA tasks in Brazilian Portuguese, using the HELM framework."
draft: false 
---

## Background

This is the first of two posts in this series, aimed at providing a summary of the investigation we conducted using the [<span class="link-personalizado">HELM</span>](https://github.com/stanford-crfm/helm) (*Holistic Evaluation of Language Models*) evaluation framework to assess the [<span class="link-personalizado">Granite</span>](https://huggingface.co/ibm-granite) family of models, the [<span class="link-personalizado">Llama-3.1-8B</span>](https://huggingface.co/meta-llama/Llama-3.1-8B) model, and the [<span class="link-personalizado">DeepSeek-R1-Distill-Llama-3.1-8B</span>](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Llama-8B) model. The evaluations cover both Portuguese-language *benchmarks* and code generation tasks. In this first part, the focus is on evaluating model performance in Brazilian Portuguese (PT-BR) for **sentiment analysis** and **MQA** (*Multiple-Choice Question Answering*) tasks. The second part, to be published soon, will present the evaluation results for code generation tasks.

The use of English-language datasets for evaluating language models is common practice. However, to evaluate this models across different languages and cultural contexts, it is important to test them on *benchmarks* in other languages. In the case of PT-BR, which typically represents a smaller share of the data used to train multilingual models, understanding model behavior is an important step in evaluating their suitability for tasks and contexts specific to this language. In this sense, this post aims to contribute to that understanding by highlighting both the advances and the remaining challenges in these LLMs’ performance on tasks in the PT-BR context.

## TL;DR

<div style="text-align: justify;">

- We evaluated the models: Granite, Llama-3.1-8B, and DeepSeek-R1-Distill-Llama-3.1-8B on the ENEM Challenge, TweetSent-Br, and IMDB *benchmarks*.
- Our method involved experimentation supported by the HELM framework, which we describe in detail in this document.
- The results show that the models accurately classify sentiments in movie reviews in PT-BR.


</div>

## Method

### Execution Environment and Tool Used

We used HELM as the evaluation tool. HELM is an LLM evaluation *framework* developed by researchers at Stanford University. It includes a variety of *benchmarks*, such as sentiment analysis, code generation, and multiple-choice question answering. Using these *benchmarks*, we evaluated and compared the performance of the Granite (8B), Llama-3.1-8B, and DeepSeek-R1-Distill-Llama-3.1-8B models.

For running the experiments, we used Google Colab as the environment, which provides access to an A100 GPU. In this setup, we were able to clone the HELM repository and run models with 8 billion parameters. All configuration and testing were carried out on this platform, ensuring convenience and access to the necessary computational resources.

In a future post, we will go into more detail about LLM evaluation strategies and tools, with a deeper focus on HELM’s capabilities and operation.

### *Benchmarks* and Models

To run tests in Brazilian Portuguese scenarios, it was necessary to extend HELM by adding new *benchmarks*, since the tool did not previously support this language. This effort represented a direct contribution to HELM, adding three *benchmarks*:

- [<span class="link-personalizado">**ENEM Challenge**</span>](https://huggingface.co/datasets/eduagarcia/enem_challenge): built from questions from the Exame Nacional do Ensino Médio (ENEM), designed to evaluate LLMs ability to handle MQA tasks across various knowledge areas, including Humanities, Natural Sciences, Languages, and Mathematics.

- [<span class="link-personalizado">**TweetSent-Br**</span>](https://huggingface.co/datasets/eduagarcia/tweetsentbr_fewshot): composed of *tweets*, specifically for sentiment analysis tasks. The dataset is organized into three main classes: positive (*tweets* expressing a positive reaction about the main topic), negative (*tweets* expressing a negative reaction), and neutral (*tweets* that don’t fit the other categories).

- [<span class="link-personalizado">**IMDB**</span>](https://huggingface.co/datasets/maritaca-ai/imdb_pt): made up of movie reviews written in Brazilian Portuguese. This *benchmark* also focuses on sentiment classification tasks, but uses longer-form review texts, in contrast to TweetSent-Br’s shorter posts.

About the models, selection was guided by compatibility with the available execution environment and by citation relevance and performance. This included the Granite family of models developed by IBM; the Llama models from Meta; and the DeepSeek-R1-Distill-Llama-8B, a compact, optimized version derived from Llama 3.1. This choice enabled a fair and practical comparison among the models.

## Results

Below, we present the results obtained, along with charts developed by the team to make it easier to visualize and understand the models’ performance on the evaluated tasks.

- **ENEM Challenge**:

<!--<div style="text-align: center;">
  <img src="/images/experimentos_benchmarks_pt_br_image001.png" style="max-width: 90%;">
</div> -->


Os resultados indicam que os modelos apresentaram desempenhos semelhantes, com uma leve vantagem para o Llama. Os modelos alcançaram uma média de acerto de 62,53%, esse percentual sugere que, embora os modelos demonstrem algum nível de compreensão das questões, ainda não possuem aptidão suficiente para responder de forma satisfatória às provas do ENEM, ou seja, para selecionar a alternativa correta. Há, portanto, um espaço para melhorias, especialmente no que diz respeito à capacidade de raciocínio e interpretação em língua portuguesa.

- **TweetSent-Br**:

<div style="text-align: center;">
  <img src="/images/experimentos_benchmarks_pt_br_image002.png" style="max-width: 90%;">
</div>


Nesse *benchmark*, assim como observado no ENEM Challenge, os resultados também foram semelhantes entre os modelos. Isso reforça a percepção de que ainda existem lacunas no desempenho dos modelos em tarefas relacionadas  à classificação de sentimentos em português. Classificar uma mensagem como positiva, negativa ou neutra ainda representa um desafio para esses modelos, especialmente diante das nuances e ambiguidades da linguagem.

- **IMDB**:

<div style="text-align: center;">
  <img src="/images/experimentos_benchmarks_pt_br_image003.png" style="max-width: 90%;">
</div>


No IMDB os resultados foram bastante positivos, os modelos apresentaram taxas de acerto superiores a 90%, demonstrando boa performance na tarefa de classificação de sentimentos. O destaque foi o modelo Granite com 8B de parâmetros, que teve uma leve superioridade em relação aos demais. Esses resultados indicam que os modelos conseguem categorizar com facilidade as críticas de filmes em português, mostrando maior domínio nesse tipo de tarefa.

## Conclusão

Com este estudo, foi possível obter uma visão mais clara sobre o desempenho dos modelos de linguagem em PT-BR, por meio da avaliação em três *benchmarks* distintos. Os resultados indicam que os modelos analisados possuem desempenho razoável ao selecionar uma alternativa para áreas do conhecimento do ENEM, e evidenciam que ainda há espaço para melhorias. Por outro lado, em tarefas de análise de sentimentos no *benchmark* IMDB, os modelos de pequeno porte demonstraram boa capacidade de classificação.

A equipe planeja, em estudos futuros, conduzir experimentos com modelos de grande porte, a fim de possibilitar comparações mais amplas de desempenho e eficiência. Isso permitirá uma análise detalhada dos erros cometidos por cada modelo, contribuindo para uma compreensão mais aprofundada de seus pontos fortes e limitações.