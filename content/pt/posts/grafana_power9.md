---
title: "Observabilidade na IBM POWER9 utilizando o Grafana"
date: 2026-09-18 # ano-mês-dia
authors: ["Maria Luísa Gomes"] # Pode ser uma lista
tags: ["Grafana", "Observability", "Power9"]
projects: ["multiarq"]
translationKey: "grafana-ppc64le"
summary: "Este post descreve o processo de compilação do Grafana 13.1.0 a partir do código-fonte em uma máquina IBM Power9 com arquitetura ppc64le"
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto
O Grafana é uma das ferramentas mais adotadas para visualização em *stacks* de observabilidade. No entanto, não são oferecidos binários oficiais para a arquitetura ppc64le nos seus *releases* padrão, e as imagens do Grafana mantidas pela IBM no Docker Hub estão desatualizadas. Portanto, rodar o Grafana em versões atuais nativamente em um servidor IBM POWER9 exige compilá-lo a partir do código-fonte, um processo que envolve ajustes específicos de compilação e dependências que não costumam aparecer nos builds para arquiteturas mais comuns, como x86.

Este post descreve o processo de compilação do Grafana 13.1.0 a partir do código-fonte em uma máquina IBM Power9 com arquitetura ppc64le, gerando os artefatos do *frontend* e *backend*. A partir desse processo, o resultado foi empacotado em uma imagem Docker e publicada no Docker Hub: 

```bash
ufcgibm/grafana-ppc64le:13.1.0-ppc64le
```
Esse processo foi automatizado num Dockerfile multi-stage, o que permite reproduzir o build para versões futuras do Grafana sem repetir a compilação manualmente.

## TL;DR
- Compilamos o Grafana 13.1.0 nativamente para ppc64le numa máquina IBM POWER 9;
- O processo envolveu *backend* em Go, *frontend* em Node.js, Yarn, Nx e Webpack, e um ajuste pontual de dependência (@swc/core);
- A imagem está publicada em  [<span class="link-personalizado">ufcgibm/grafana-ppc64le</span>](https://hub.docker.com/r/ufcgibm/grafana-ppc64le);
- Transformamos o processo manual num Dockerfile *multi-stage* reproduzível, ou seja, atualizar para uma versão nova do Grafana agora é um *docker build --build-arg*, não uma recompilação manual do zero.

## Ambiente Utilizado
*Hardware*:
- Arquitetura ppc64le;
- Processador: IBM POWER9
- RAM: ~128 GB;
- 16 CPUs disponíveis
- Sistema Operacional: AlmaLinux 8.10

## Como o Grafana se encaixa na *stack* de observabilidade

O Grafana não coleta métricas sozinho, ele é a camada de visualização de uma *stack* maior. Entender esse *pipeline* ajuda a situar por que um build isolado do Grafana resolve só parte da *stack* de observabilidade em ppc64le.

No ambiente testado, o fluxo funciona assim:
- **node_exporter** roda no *host* da POWER9 e expõe métricas de sistema (uso de CPU, memória, disco, rede) em um *endpoint* HTTP (`/metrics`), no formato que o Prometheus entende;
- **Prometheus** faz *scrape* periódico desse *endpoint*, ou seja, consulta ativamente o `/metrics` em intervalos configuráveis, e armazena os valores coletados como séries temporais na sua própria base de dados (TSDB), mantendo o histórico ao longo do tempo;
- **Grafana** se conecta ao Prometheus como uma fonte de dados (*datasource*) e consulta esses dados usando `PromQL`, a linguagem de consulta do Prometheus, para montar os painéis e gráficos que o usuário efetivamente vê.

{{< figure src="/images/GrafanaStack.png" alt="Figura 1" caption="Stack de observabilidade na Power9">}}

Um ponto importante dessa arquitetura é que, diferente do Grafana, tanto o Prometheus quanto o node_exporter já oferecem binários e imagens docker oficiais para a arquitetura ppc64le, por isso que o esforço de compilação a partir do código fonte descrito neste post foi específico ao Grafana. Ainda sim, os três componentes rodam isolados, cada um em seu próprio *container*, seguindo o modelo de operação padrão desse tipo de *stack*.


## Dependências
Para realizar a compilação, foram utilizadas as seguintes versões:


| Dependência | Versão |
| :---- | :---- |
| Grafana | 13.1.0 |
| GCC  | 11.2.1 |
| Go | 1.26.5 |
| Node.js | 22.22.2 |
| npm | 10.9.7 |
| Yarn | 4.15.0 |
| Python | 3.11 |
| @swc/core | 1.15.40 |


O Grafana 13 requer Node.js 22 ou superior, e o projeto especifica Yarn como seu gerenciador de pacotes. O único ajuste necessário foi na dependência `@swc/core`, usada no build do frontend: a versão fixada pelo Grafana não possui binário pré-compilado para `linux-ppc64le`, e atualizá-la para uma versão que oferece esse binário resolveu o problema.

Com isso, o build gera um binário `bin/grafana` nativo, confirmado como executável ppc64le na versão 13.1.0. O passo a passo completo está documentado no repositório [<span class="link-personalizado">grafana-ppc64le</span>](https://github.com/llm-pt-ibm/grafana-ppc64le/blob/main/TUTORIAL.md) no Github.

## **Utilizando a imagem Docker**

Para quem não deseja realizar a compilação, temos uma imagem disponível no Docker Hub como 
[<span class="link-personalizado">ufcgibm/grafana-ppc64le</span>](https://hub.docker.com/r/ufcgibm/grafana-ppc64le).

Baixe a imagem:

```bash
docker pull ufcgibm/grafana-ppc64le:13.1.0-ppc64le
``` 
Execute:
```bash
docker run -d \
    --name grafana-ppc64le \
    -p 3000:3000 \
    ufcgibm/grafana-ppc64le:13.1.0-ppc64le
``` 

Verifique o container:
```bash
docker ps
``` 

O Grafana estará disponível na porta 3000.

## Compilando novas versões
O tutorial apresentado anteriormente foi feito para a versão 13.1.0 do Grafana. Porém, toda vez que uma nova versão do Grafana é disponibilizada, é preciso repetir esse processo de *build* manualmente. A solução para esse problema foi transformar o processo inteiro em um Dockerfile *multi-stage* que funciona da seguinte forma: temos um primeiro estágio (*builder*) que instala todas as dependências, clona o Grafana na *tag* desejada, aplica o ajuste do @swc/core e roda o *make deps && make build*, tudo dentro do próprio *build* do Docker, e um segundo estágio que copia apenas os binários e os *assets* já compilados no primeiro, gerando a imagem final de *runtime*. 

Agora, para atualizar para uma versão nova do Grafana basta mudar os argumentos:

```bash
docker build \
  --build-arg GRAFANA_VERSION=v13.2.0 \
  --build-arg SWC_CORE_VERSION=1.15.40 \
  -t ufcgibm/grafana-ppc64le:13.2.0-ppc64le .
``` 

O ```SWC_CORE_VERSION``` fica como parâmetro à parte porque pode precisar mudar de novo no futuro. Se isso acontecer, podemos checar o *changelog* do [<span class="link-personalizado">@swc/core</span>](https://github.com/swc-project/swc/releases) no GitHub. A imagem docker mencionada anteriormente foi feita a partir desse Dockerfile, validado em uma máquina IBM Power9.

## Recursos

* Repositório no [<span class="link-personalizado">GitHub</span>](https://github.com/llm-pt-ibm/grafana-ppc64le) (Dockerfile, README, matriz de compatibilidade por versão);  
* Imagem no Docker Hub: [<span class="link-personalizado">ufcgibm/grafana-ppc64le</span>](https://hub.docker.com/r/ufcgibm/grafana-ppc64le);   
* Repositório oficial do [<span class="link-personalizado">Grafana</span>](http://github.com/grafana/grafana);

## Disclaimer

Esse trabalho foi desenvolvido pela Universidade Federal de Campina Grande (UFCG), como parte de um projeto de Pesquisa, Desenvolvimento e Inovação realizado em parceria com a IBM e a Flex Brazil. A imagem e as adaptações apresentadas neste tutorial não são produtos ou distribuições oficiais da IBM ou do Grafana Labs.
