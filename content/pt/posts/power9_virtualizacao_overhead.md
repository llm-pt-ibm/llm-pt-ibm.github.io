---
title: "Avaliando o custo de virtualização em IBM Power9: host, KVM e QEMU"
date: 2026-09-14
authors: ["Gabrielly Lima"]
tags: ["Virtualização", "Power9", "KVM", "QEMU", "Benchmark", "GPU"]
projects: ["multiarq"]
translationKey: "power9-virtualization-overhead"
summary: "Medimos o overhead de virtualização em IBM Power9 (ppc64le) comparando o host físico, KVM e QEMU com passthrough de GPU NVIDIA V100, e descrevemos um viés de configuração que faria a virtualização parecer mais rápida que o próprio host."
draft: false
---

## Contexto

Executar cargas de trabalho aceleradas por GPU em sistemas IBM Power9 costuma envolver escolher entre rodar direto no hardware físico ou usar algum nível de virtualização, seja para isolamento, orquestração via Kubernetes, ou reprodutibilidade entre times. Essa escolha tem um custo, mas qual o tamanho dele?

Este post descreve como medimos o overhead de virtualização em IBM Power9 (ppc64le), comparando o host físico (bare-metal), KVM (com aceleração de hardware) e QEMU rodando sob TCG (emulação por software pura, sem aceleração de hardware), com passthrough de GPU NVIDIA V100 nos ambientes que suportam. Também descrevemos uma descoberta metodológica feita no caminho: um viés de configuração que, se não identificado, teria feito a virtualização parecer mais rápida que o próprio host físico.

## O que este trabalho entrega

* Um pipeline reproduzível de benchmark comparando o host físico, KVM e QEMU em ppc64le, cobrindo desempenho de CPU, memória, cache, banda de memória, armazenamento e GPU.
* Um caso documentado mostrando como um viés de configuração no host pode inflar ou até inverter resultados de benchmark de virtualização, e como identificar isso.

## Ambiente experimental

### Hardware do host

* Servidor IBM Power9 (AC922);
* Arquitetura ppc64le (64-bit little-endian);
* 512 GB de RAM;
* 4× NVIDIA Tesla V100 SXM2 (16 GB);
* Armazenamento SSD/NVMe, /dev/sdb1, ext4;
* Sistema operacional: AlmaLinux 8.10;
* Kernel 4.18.0-553.105.1.el8_10.ppc64le.

### Software do host, por função

* Orquestração: Slurm 23.11.7;
* Virtualização: Libvirt 8.0.0, QEMU 6.2.0;
* Stack de GPU: CUDA Toolkit 12.2, cuDNN 9.0, NCCL 2.18.3, driver NVIDIA 535;
* Ferramentas de benchmark: stress-ng 0.15.00, Fio 3.19, Python 3.13.12 (pipeline de análise).

### Configuração das VMs

Cada VM foi configurada com 16 vCPUs e 64 GB de RAM, usando um disco virtio configurado para I/O direto, contornando o cache do lado do host.

## Metodologia

Comparamos os três ambientes usando 15 execuções cada, seguindo um desenho aninhado para KVM e QEMU composto por três máquinas virtuais independentes com cinco réplicas cada, evitando tratar réplicas da mesma VM como amostras independentes. Agendamos as execuções via Slurm, o que garantiu alocação consistente de recursos e evitou execuções sobrepostas.

A suíte de benchmarks cobriu CPU e memória (stress-ng), cache (stress-ng), banda de memória (STREAM), armazenamento (fio) e GPU (matrixMul e NCCL). A Figura 1 mostra o pipeline completo, da coleta de dados à validação estatística.

{{< figure src="/images/evaluation_pipeline.png" alt="Figura 1: Visão geral do pipeline de benchmark." caption="Figura 1. Visão geral do pipeline de benchmark: cada ambiente executa a mesma suíte de benchmarks, 15 execuções são coletadas, e os resultados passam por uma etapa formal de validação estatística antes de produzir as estimativas finais de overhead." >}}

Toda comparação seguiu um pipeline estatístico formal. Como não podíamos assumir que as distribuições subjacentes eram normais ou tinham variâncias iguais, testamos as duas suposições (Shapiro-Wilk para normalidade, Levene para homogeneidade de variância) antes de escolher entre ANOVA e Kruskal-Wallis para cada métrica. Quando o teste global mostrava diferença significativa entre os ambientes, usamos o teste post-hoc de Dunn para identificar quais pares específicos diferiam. Como testamos várias métricas ao mesmo tempo, aplicamos a correção de Benjamini-Hochberg entre todas elas para manter a taxa de descoberta falsa sob controle.

## Achados

Duas métricas, desempenho de cache e IOPS aleatório com fila de profundidade 1, inicialmente mostraram o KVM superando o host físico por uma margem larga (mais de 300% no benchmark de cache), um resultado fisicamente impossível já que a virtualização adiciona overhead em vez de removê-lo.

A causa foi uma configuração de pinning de CPU ausente no host: os benchmarks do host rodaram com processos livres para migrar entre todos os núcleos e nós NUMA, enquanto as VMs sempre estiveram fixadas (pinned) a núcleos físicos específicos. Aplicar o mesmo pinning ao host resolveu as duas discrepâncias; todos os demais resultados variaram menos de 4 pontos percentuais, confirmando que o problema estava isolado a essas duas medições. A Figura 2 mostra o efeito da correção nas duas métricas.

{{< figure src="/images/effect_pinning.jpeg" alt="Figura 2: Overhead de cache e IOPS QD=1 antes e depois da correção de pinning." caption="Figura 2. Overhead de cache e IOPS QD=1 antes e depois da correção de pinning. Os valores \"antes\" vêm da baseline enviesada e não corrigida do host, não sendo uma medição válida de desempenho de virtualização. A faixa sombreada marca uma zona de ±5% de variação desprezível." >}}

Nota metodológica: pinning de CPU consistente entre todos os ambientes comparados é essencial para que medições de overhead de virtualização sejam fisicamente significativas. Uma configuração assimétrica, mesmo que acidental, pode inverter completamente o resultado medido.

## Resultados

A Figura 3 mostra o overhead em todos os domínios medidos.

{{< figure src="/images/consolidated_overview.jpeg" alt="Figura 3: Overhead relativo ao host físico em todos os domínios medidos." caption="Figura 3. Overhead relativo ao host físico em todos os domínios medidos (CPU, memória, cache, STREAM, disco e GPU). Cada painel usa sua própria escala; o comprimento das barras não deve ser comparado entre painéis." >}}

Em resumo, o KVM introduz overhead moderado e localizado, afetando principalmente desempenho de cache, banda de memória e IOPS aleatório, um padrão consistente com o custo esperado de virtualização assistida por hardware. O QEMU (TCG) introduz overhead substancial em praticamente toda carga de trabalho, exceto I/O sequencial, como esperado de emulação por software sem aceleração de hardware. O passthrough de GPU não apresentou overhead mensurável no KVM.

## Conclusão

Este trabalho demonstra que é possível medir com rigor o custo de rodar cargas de trabalho aceleradas por GPU sob virtualização em IBM Power9 usando metodologia estatística sólida, e que esse custo é significativamente menor com KVM do que com QEMU, como esperado.

Também demonstra que o benchmark de virtualização é altamente sensível a variáveis de configuração do sistema além da mera presença ou ausência de um hipervisor. Pinning de CPU é um viés bem conhecido em benchmark de virtualização, e teria distorcido as conclusões caso passasse despercebido.

## Disclaimer

This work is not an official IBM release or software distribution and is not developed or supported by IBM.

This work was developed by the Federal University of Campina Grande (UFCG), a Brazilian public university, as part of a Research, Development, and Innovation project conducted in partnership with IBM and Flex Brazil.

## Recursos

* GitHub repository: [<span class="link-personalizado">llm-pt-ibm/ppc64le-virtualization-performance</span>](https://github.com/llm-pt-ibm/ppc64le-virtualization-performance)
