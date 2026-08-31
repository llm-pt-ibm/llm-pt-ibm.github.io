---
title: "Instalação do Docker em ambiente Arquitetura ppc64le (Power9)"
date: 2026-04-01
authors: ["Gabrielly Lima"]
tags: ["Docker", "Power9", "ppc64le", "AlmaLinux", "Contêineres"]
projects: ["multiarq"]
translationKey: "power9-docker-installation"
summary: "Neste post, mostramos o passo a passo para instalar o Docker Engine no AlmaLinux 8.10 em IBM Power9 (ppc64le), incluindo a remoção de pacotes conflitantes, a configuração do repositório oficial e cuidados com compatibilidade de imagens."
draft: false
---

## Contexto
Diante da necessidade de padronizar a execução de softwares no nosso servidor IBM Power9 (ppc64le), o uso de contêineres apresenta-se como uma solução robusta para evitar conflitos de ambiente. Este post dá continuidade à estruturação da nossa infraestrutura, detalhando a instalação do Docker Engine no AlmaLinux. A adoção desta tecnologia é estratégica para garantir o isolamento rigoroso de dependências e a portabilidade das diversas aplicações. Com isso, conseguimos encapsular desde bibliotecas de uso geral até serviços mais complexos, assegurando um ambiente de execução limpo, seguro e altamente reprodutível.

O Docker Engine tem suporte oficial para AlmaLinux nas arquiteturas x86_64, arm64, s390x e ppc64le, o que nos permite utilizá-lo diretamente no Power9 sem adaptações especiais. No entanto, alguns cuidados são necessários antes e durante a instalação, como desinstalar ferramentas que conflitam com o Docker e garantir que as imagens utilizadas sejam compatíveis com a arquitetura ppc64le.

## TL;DR
* Este post apresenta o passo a passo para instalar o Docker Engine no AlmaLinux na arquitetura ppc64le.
* É necessário remover o Podman e o Buildah antes de instalar, pois conflitam com o Docker.
* Imagens do Docker Hub precisam ter suporte explícito a ppc64le para funcionar no Power9.

## Ambiente utilizado
* **Arquitetura**: Servidor IBM Power9 (Arquitetura ppc64le)
* **Sistema Operacional (SO)**: AlmaLinux 8.10 binário compatível com Red Hat Enterprise Linux (RHEL) 8.9/8.10
* **RAM**: 512GB

## Pré-requisitos
Antes de instalar o Docker, é importante estar ciente de uma limitação relacionada ao firewall: ao expor portas de contêineres usando o Docker, essas portas ignoram as regras padrão do firewalld. Certifique-se de que isso não representa um problema para o seu ambiente antes de prosseguir. Além disso, é importante destacar que o Docker Engine é compatível com Rocky Linux 8 e 9 e o AlmaLinux 8 na arquitetura ppc64le.

## Removendo pacotes conflitantes
O AlmaLinux, por padrão, possui o Podman e o Buildah instalados. Esses pacotes conflitam com o Docker Engine e precisam ser removidos antes da instalação. Também é recomendável remover quaisquer versões antigas do Docker que possam estar presentes:

```bash
sudo dnf remove -y podman \
                   buildah \
                   docker \
                   docker-client \
                   docker-client-latest \
                   docker-common \
                   docker-latest \
                   docker-latest-logrotate \
                   docker-logrotate \
                   docker-engine
```

## Adicionando repositório do Docker e instalando pacotes necessários
### Configuração do repositório
O método recomendado de instalação é utilizar o repositório oficial do Docker. Vale mencionar que o Docker utiliza o repositório do CentOS para distribuições baseadas em RHEL — como o AlmaLinux — e isso é oficialmente suportado. Primeiro, instale o pacote dnf-plugins-core e adicione o repositório:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### Instalando o Docker Engine
Com o repositório configurado, instale a versão mais recente do Docker Engine junto com os plugins de build e compose:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Iniciando o serviço
Diferentemente de distribuições baseadas em Debian, como o Ubuntu, o Docker não inicia automaticamente no AlmaLinux após a instalação. É necessário iniciar o serviço manualmente e habilitá-lo para que suba junto com o sistema:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

## Verificando a instalação
Para confirmar que tudo foi instalado corretamente, execute a imagem hello-world. O Docker detectará automaticamente a arquitetura ppc64le e baixará a imagem correta:

```bash
sudo docker run hello-world
```

A saída esperada é uma mensagem que confirme que o Docker está funcionando corretamente.

## Configurações pós-instalação
Por padrão, apenas o usuário root ou usuários com privilégios de sudo podem executar comandos do Docker. Para evitar o uso de sudo em todo comando, adicione seu usuário ao grupo docker. Primeiro, crie o grupo caso ele não exista:

```bash
sudo groupadd docker
```

Em seguida, adicione seu usuário ao grupo:

```bash
sudo usermod -aG docker $USER
```

É necessário deslogar e logar novamente para que as permissões sejam aplicadas.

## Dicas para arquitetura Power9
Como estamos utilizando o IBM Power9, alguns cuidados adicionais são importantes ao trabalhar com o Docker Hub. O primeiro ponto é a compatibilidade de imagens: nem todas as imagens disponíveis no Docker Hub possuem suporte a ppc64le. Imagens exclusivas para x86_64 resultarão em erro de execução no Power9, por isso sempre verifique se a imagem desejada possui a tag ppc64le antes de utilizá-la.

Para validar se o Docker está rodando corretamente e reconhecendo a arquitetura da máquina, use:

```bash
docker version --format '{{.Server.Arch}}'
```

A saída esperada é ppc64le.

## Considerações finais
A instalação do Docker Engine no AlmaLinux (ppc64le) segue um fluxo direto, desde que os conflitos com o Podman e o Buildah sejam resolvidos previamente. O suporte oficial ao ppc64le pelo Docker garante uma experiência estável no Power9, com a ressalva de que a compatibilidade das imagens utilizadas deve ser sempre verificada antes do uso.

Com o Docker instalado e configurado, o ambiente está pronto para executar contêineres e avançar para as próximas etapas da nossa infraestrutura de modelos de linguagem.
