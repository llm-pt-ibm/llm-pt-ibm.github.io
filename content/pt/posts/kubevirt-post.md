---
title: "Executando VMs com KubeVirt na IBM Power9 (ppc64le)"
date: 2026-05-22
draft: false
authors: ["João Ramalho"]
projects: ["multiarq"]
tags: ["KubeVirt", "Power9", "ppc64le", "Kubernetes", "Virtualização"]
summary: "Este post documenta como adaptamos o KubeVirt para criar e gerenciar máquinas virtuais via Kubernetes na IBM Power9 (ppc64le), uma arquitetura não suportada oficialmente pelo projeto."
translationKey: "kubevirt-post"
---

## Contexto
Este post tem como objetivo apresentar o processo de adaptação do [KubeVirt](https://kubevirt.io/) para a arquitetura IBM POWER9 (ppc64le). Serão abordados os principais desafios encontrados, as modificações realizadas no código-fonte, o papel de cada componente e os resultados obtidos ao final do processo.

O KubeVirt é um operador que estende o Kubernetes para gerenciar máquinas virtuais (VMs) como recursos nativos. Em ambientes tradicionais, VMs são gerenciadas por ferramentas como libvirt/virsh, separadas do ecossistema de containers. O KubeVirt elimina essa separação: com ele, é possível criar, iniciar, parar e monitorar VMs usando os mesmos comandos e workflows do Kubernetes — `kubectl`, YAML, namespaces e RBAC. As VMs rodam como processos QEMU/KVM reais dentro de pods gerenciados pelo Kubernetes.

A motivação para este trabalho surgiu no contexto do projeto Multiarq, que mantém uma infraestrutura compartilhada de HPC na IBM POWER9. A possibilidade de gerenciar VMs e containers no mesmo cluster Kubernetes simplifica a administração do ambiente e abre caminho para cenários como GPU passthrough para workloads de AI/ML dentro de VMs, isolamento de ambientes de pesquisa e testes de compatibilidade multi-arquitetura.

O principal desafio é que o KubeVirt **não oferece suporte oficial para ppc64le**. Apenas x86_64 (amd64), arm64 e s390x são suportados. Isso significa que o sistema de build, as validações da API, os defaults de configuração e o pipeline de geração de domínios libvirt não reconhecem ppc64le, tratando tudo como amd64 por padrão.

## TL;DR

- O KubeVirt não suporta ppc64le oficialmente; todo o pipeline assume amd64 como fallback.
- Compilamos os binários Go diretamente, contornando o sistema de build Bazel que não reconhece a arquitetura.
- Foram necessários patches em ~14 arquivos Go e a criação de 4 arquivos novos para adicionar suporte a ppc64le.
- As imagens Docker foram criadas com Dockerfiles customizados e servidas via registry local.
- Com as adaptações, foi possível executar uma VM CirrOS ppc64le via KubeVirt no POWER9, gerenciada inteiramente pelo Kubernetes.

## Ambiente de Execução

- **Arquitetura:** Servidor IBM Power9 (ppc64le).
- **Sistema Operacional:** AlmaLinux 8.10, binário compatível com RHEL 8.9/8.10.
- **GPUs:** 4x NVIDIA Tesla V100-SXM2-16GB.
- **Docker:** Docker CE 26.1.3.
- **Kubernetes:** v1.35.0 via minikube v1.38.0 (driver docker, runtime containerd).
- **KubeVirt:** v1.8.2.
- **Go:** 1.24.9.

## O que é o KubeVirt e como ele funciona

O KubeVirt é composto por vários componentes que trabalham juntos para traduzir um recurso Kubernetes (a VirtualMachineInstance, ou VMI) em uma VM QEMU/KVM real rodando no host.

O **virt-operator** é o ponto de entrada: quando o administrador cria o Custom Resource KubeVirt no cluster, o operator provisiona todos os outros componentes — deployments, daemonsets, services, RBAC. Ele funciona como um instalador permanente que reconcilia o estado desejado.

O **virt-api** recebe as chamadas da API do Kubernetes para os recursos do KubeVirt. Quando o usuário faz `kubectl apply` de uma VMI, o virt-api valida o YAML (ex: a arquitetura é suportada? o machine type existe?) e injeta defaults (ex: firmware UUID, CPU topology).

O **virt-controller** observa as VMIs e decide onde elas devem rodar. Ele cria um pod especial — o virt-launcher — no node adequado, com todas as configurações necessárias (volumes, devices, node selectors).

O **virt-handler** roda como DaemonSet (um por node) e é o agente local que faz a ponte entre Kubernetes e libvirt/QEMU. Quando o pod do virt-launcher aparece no node, o virt-handler lê a spec da VMI, gera o XML do domínio libvirt e instrui o libvirt a criar a VM. Também registra device plugins no kubelet (`/dev/kvm`, `/dev/net/tun`, `/dev/vhost-net`) para que os pods possam acessar os dispositivos necessários.

O **virt-launcher** é o pod que encapsula a VM. Cada VMI gera um pod dedicado com três containers: o compute (QEMU + libvirt), o guest-console-log e o container disk. Dentro do container compute, o processo QEMU roda a VM real — com seu próprio kernel, memória e CPU virtual.

O fluxo completo é:

1. `kubectl apply` → API Server → virt-api (valida, injeta defaults)
2. virt-controller detecta a VMI → cria o pod virt-launcher no node adequado
3. kubelet inicia o pod virt-launcher no node
4. virt-handler detecta o pod → lê a spec da VMI → gera XML libvirt → chama o libvirt
5. libvirt inicia o QEMU → VM roda dentro do pod

É importante ressaltar que a VM **não vira um container** — ela roda como um processo QEMU real dentro de um pod. O Kubernetes gerencia o ciclo de vida do pod, e o KubeVirt traduz entre os dois mundos.

## Desafios e Adaptações

### Sistema de Build

O sistema de build do KubeVirt utiliza Bazel, que não reconhece ppc64le. A função `format_archname` no script de build só aceita `x86_64`, `aarch64` e `s390x`. A solução foi compilar os binários Go diretamente com `go build`, contornando o Bazel.

Uma dependência adicional é o **libnbd**: o virt-launcher requer a versão 1.18+, mas o AlmaLinux 8 só disponibiliza a 1.6. Foi necessário compilar o libnbd 1.20 do fonte. O componente **container-disk** é um programa C (não Go) que precisa de compilação estática para rodar em containers `FROM scratch`.

### Validação da API

O webhook de validação do virt-api rejeita VMIs com arquitetura desconhecida. Sem o patch, uma VMI com `architecture: ppc64le` seria recusada antes mesmo de chegar ao scheduler. Foi necessário adicionar cases no admitter e criar uma função de validação específica para ppc64le.

### Defaults de Configuração

O KubeVirt precisa saber qual machine type usar para cada arquitetura (ex: `pc-q35` para amd64, `virt` para arm64). Para ppc64le, configuramos `pseries` como machine type padrão — o tipo de máquina virtual do POWER.

### Geração do Domínio Libvirt

Este foi o desafio central. O KubeVirt converte a spec da VMI em um XML de domínio libvirt que o QEMU interpreta. Esse pipeline tem duas partes:

O **arch-defaulter** define valores padrão do OS type (`arch` e `machine`) no XML. Sem o patch, ele retornava `x86_64` para ppc64le, fazendo o libvirt tentar criar uma VM x86 numa máquina POWER — resultando no erro `No emulator found for arch 'x86_64'`.

O **converter** é uma interface com ~12 métodos que definem comportamentos específicos de cada arquitetura: se precisa de USB, SMBIOS, PCIe placement, ROM tuning, etc. Existiam implementações para amd64, arm64 e s390x, mas não para ppc64le. O código caía no fallback `converterAMD64`, gerando configurações incompatíveis. Criamos o `converterPPC64LE` com valores adequados para POWER: sem USB, sem SMBIOS, sem PCIe placement, com VirtIO como modelo de disco.

Resolvido o converter, surgiu o erro de **dispositivos USB**: o pipeline de graphics/video não tinha case para ppc64le, fazendo o libvirt adicionar um video device VGA default que dependia de USB — mas o controller USB estava desabilitado (`IsUSBNeeded: false`). A solução foi adicionar um case ppc64le no configurador de video com `virtio` como dispositivo de vídeo, seguindo o mesmo padrão do s390x e arm64.

Por fim, o **CPU model**: o KubeVirt usa `host-model` como default, que não funciona em virtualização aninhada no POWER9. A solução foi especificar `POWER9` como CPU model na VMI.

### Imagens Docker

Sem Dockerfiles no projeto (tudo é gerado pelo Bazel), criamos Dockerfiles customizados para cada componente. Os componentes mais simples (virt-operator, virt-api, virt-controller, virt-exportproxy) usam `ubi8/ubi-minimal` como base. O virt-handler requer ferramentas de sistema adicionais. O virt-launcher é o mais complexo, com `almalinux:8` como base e dependências de qemu-kvm, libvirt e o libnbd compilado. Um registry local (`registry:2` na porta 5000) serve as imagens para o minikube.

### Passo a Passo Técnico
 
Devido à grande quantidade de etapas envolvidas, o passo a passo com os procedimentos detalhados, incluindo todos os patches no código Go, Dockerfiles, comandos de compilação e configuração, está disponível neste link: [guia-instalacao-kubevirt-ppc64le](https://github.com/llm-pt-ibm/kubevirt-ppc64le).

## Resultados

Com todas as adaptações aplicadas, foi possível executar uma VM CirrOS ppc64le via KubeVirt no POWER9, gerenciada inteiramente pelo Kubernetes:

```
$ kubectl get vmi test-vmi -o wide
NAME       AGE     PHASE     IP               NODENAME   READY
test-vmi   2m43s   Running   10.244.120.124   minikube   True
```

Dados coletados de dentro da VM confirmam a execução correta:

|                      | Valor                                  |
|----------------------|----------------------------------------|
| Arquitetura          | ppc64le                                |
| CPU                  | POWER9 (architected), altivec supported|
| Hypervisor           | KVM                                    |
| Plataforma           | pSeries                                |
| Modelo               | IBM pSeries (emulated by qemu)         |
| Kernel               | 5.15.0-71-generic ppc64le              |

Esses dados confirmam que o KubeVirt está gerando o domínio libvirt correto para ppc64le, com machine type `pseries`, CPU POWER9 e virtualização KVM/QEMU com paravirtualização VirtIO.

## Considerações Finais

Com as adaptações realizadas, tornou-se possível utilizar o KubeVirt para criar e gerenciar máquinas virtuais em uma IBM POWER9 via Kubernetes. A VM executada é uma VM KVM/QEMU real — com kernel próprio, memória isolada e CPU virtual — gerenciada como qualquer outro recurso Kubernetes.

No contexto do projeto Multiarq, essa solução permite unificar a gestão de containers e VMs no mesmo cluster, simplificando a administração da infraestrutura compartilhada. Workloads que exigem isolamento de kernel ou acesso direto a hardware (como GPU passthrough) podem ser executados em VMs sem abandonar o ecossistema Kubernetes.

Os patches realizados são potencialmente contribuíveis ao projeto KubeVirt upstream. A arquitetura do KubeVirt já prevê extensibilidade por arquitetura — o padrão de interfaces (Converter, ArchDefaulter) e switches por arch facilita a adição de novas plataformas. O ppc64le segue o mesmo padrão do s390x, que também foi adicionado posteriormente ao projeto.

## Próximos Passos

- Resolver o conflito USB/Graphics para permitir VNC sem o workaround `autoattachGraphicsDevice: false`, habilitando acesso gráfico às VMs;
- Ajustar o CPU model default no código para que ppc64le use `POWER9` automaticamente, sem exigir especificação manual na VMI;
- Explorar GPU passthrough das V100 via KubeVirt para executar workloads de AI/ML dentro de VMs gerenciadas pelo Kubernetes;
- Testar outras distribuições como containerDisk (Fedora, Ubuntu Server, AlmaLinux ppc64le) para validar a compatibilidade além do CirrOS;
- Configurar masquerade networking para habilitar live migration entre nodes;
- Documentar as mudanças em formato de PR para contribuição ao KubeVirt upstream;
- Validar o KubeVirt no Single Node OpenShift (OCP 4.21) já instalado na máquina, usando o OpenShift Virtualization como operador.