---
title: "Realizando Inferências em CPU na Power10"
date: 2025-04-06 # ano-mês-dia
authors: ["André Alves", "Ronaldd Matias"] # Pode ser uma lista
tags: ["LLM", "Power10"]
translationKey: "power10"
summary: "Neste post é ensinado a como executar o modelo Granite-20b-Code-Instruct em CPU em uma máquina Power10. "
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto  
Neste post iremos apresentar a nossa experiência em executar o modelo Granite-20b-Code-Instruct em uma máquina Power10, apresentando os desafios e demais configurações necessárias para realizar inferências utilizando o Llama.cpp, uma das bibliotecas opensource mais populares neste domínio.

## TL;DR  
- Este post apresenta detalhes sobre como configurar e realizar inferências utilizando a infraestrutura da IBM Power 10;
- Nosso maior desafio foi a configuração do Llama cpp, que demandou ajustes como a instalação do Ninja-builder, realização da compilação do OpenBLAS e atualização do compilador C.

## Infraestrutura  
As inferências foram realizadas em uma máquina com arquitetura IBM POWER10, equipada com 750 GB de memória RAM e executando o sistema operacional Red Hat Enterprise Linux 8.10. O acesso ao ambiente é realizado por meio de uma VM, sendo necessário o uso de uma VPN para estabelecer uma comunicação segura e controlada com o sistema, possibilitando a execução das atividades de forma remota e eficiente.

## Setup Inicial  
A biblioteca que nos permite executar LLMs utilizando os recursos computacionais da CPU é o Llama.cpp. Para a sua configuração, foi necessário resolver duas dependências externas: o Ninja-builder e o OpenBLAS. O NinjaBuilder é responsável por otimizar o processo de compilação, enquanto o OpenBLAS é uma biblioteca responsável pelos cálculos matriciais de alto desempenho.

Durante o processo de build do OpenBLAS, identificamos discrepâncias nos testes internos de validação dos cálculos matriciais, indicando um problema de compatibilidade com o compilador C disponível, que estava em uma versão mais antiga, a 8.5.0. A solução, portanto, **foi a atualização do compilador para uma versão mais recente, a 13.2**, garantindo melhor compatibilidade com a arquitetura Power10 e validando a precisão das operações numéricas necessárias para o funcionamento do Llama.cpp. A seguir, apresentamos o passo a passo realizado para viabilizar a compilação das bibliotecas necessárias, bem como a atualização do compilador C.

1. Criando o ambiente de compilação para o builder

```
sudo dnf update -y  && dnf -y groupinstall 'Development Tools' && dnf install -y \ cmake git ninja-build-debugsource.ppc64le \ && dnf clean all
```

2. Atualizando compilador C e definindo variáveis de ambiente

```
scl enable gcc-toolset-13 bash
export CC=/usr/bin/gcc-13
export CXX=/usr/bin/g++-13
```

3. Baixando e compilando o OpenBLAS

```
git clone --recursive https://github.com/DanielCasali/OpenBLAS.git && cd OpenBLAS && \ make -j$(nproc --all) TARGET=POWER10 DYNAMIC_ARCH=1 && \ make PREFIX=/opt/OpenBLAS install && \ cd /
```

4. Baixando e compilando o Llama.cpp usando a biblioteca OpenBLAS que acabamos de baixar

```
 git clone https://github.com/DanielCasali/llama.cpp.git && cd llama.cpp && sed -i "s/powerpc64le/native -mvsx -mtune=native -D__POWER10_VECTOR__/g" ggml/src/CMakeLists.txt && \ mkdir build; \ cd build; \ cmake -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS -DBLAS_INCLUDE_DIRS=/opt/OpenBLAS/include -G Ninja ..; \ cmake --build . --config Release
```

Com todos esses passos realizados com sucesso, o ambiente foi devidamente configurado e otimizado para a execução local do Llama.cpp. Agora, somos capazes de iniciar um servidor para realizar inferências com LLM's de forma eficiente, utilizando exclusivamente os recursos da CPU.


## Realizando Inferência

Nós escolhemos o modelo Granite-20b-code-instruct no formato .GGUF, que é desenvolvido especificamente para otimizar o desempenho de modelos de linguagem em ambientes que utilizam apenas CPU. Esses modelos são quantizados, ou seja, a precisão dos cálculos feitos por eles são reduzidas, e, por conseguinte, o tamanho e consumo de memória também são menores, tornando-os ideais para a execução eficiente com Llama.cpp. Essa abordagem viabiliza inferências locais com alto desempenho, mesmo em arquiteturas baseadas exclusivamente em processadores, como é o caso da POWER10.
O download do modelo foi feito diretamente do Hugging Face. A seguir, mostraremos o passo a passo para realizar o download:

1. Criar um diretório para o modelo no Llama.cpp:

```
mkdir -p /root/llama.cpp/models/granite-20b-code-instruct-8k-GGUF
```

2. Acessar o diretório no Llama.cpp: 

```
cd /root/llama.cpp/models/granite-20b-code-instruct-8k-GGUF
```

3. Baixar o modelo via Hugging Face: 

```
wget https://huggingface.co/ibm-granite/granite-20b-code-instruct-8k-GGUF/resolve/main/granite-20b-code-instruct.Q4_K_M.gguf
```

O último passo pode ser mais demorado a depender da quantidade de parâmetros do modelo. Todavia, após concluir os passos acima, podemos subir um servidor Llama.cpp para que seja possível realizarmos inferências, por padrão, o servidor é exposto na porta 8080 da Power10, mas isso é completamente customizável. O código a seguir ilustra como configurar e executar o servidor Llama:

```
/root/llama.cpp/build/bin/llama-server --host 0.0.0.0 --model /root/llama.cpp/models/granite-20b-code-instruct-8k-GGUF/granite-20b-code-instruct.Q4_K_M.gguf
```

Com o servidor do Llama.cpp executando na porta 8080, agora somos capazes de realizar inferências via requisições HTTP. Neste exemplo, para fins de simplicidade, utilizamos o curl para requisições:


```
curl -X POST http://localhost:8080/completion \
     -H "Content-Type: application/json" \
     -d '{
           "prompt": "Make a hello world program in Java. Your answer should be in Java code only.",
           "max_tokens": 100
         }'

```

A seguir, um exemplo de como a resposta é retornada:

```
{
  "content": "
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

Com isso, agora somos capazes de realizar inferências em CPU. Nossos próximos passos visa realizar essas inferências utilizando o *Framework* de avaliação HELM (*Holistic Evaluation of Language Models*) como mediador. 