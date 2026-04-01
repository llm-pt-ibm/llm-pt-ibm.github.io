---
title: "Instalação do Docker em ambiente ppc64le (Power9)"
date: 2026-04-01
authors: ["Gabrielly Lima"]
tags: ["Docker", "Power9", "ppc64le", "AlmaLinux", "Contêineres"]
projects: ["multiarq"]
translationKey: "power9-docker-installation"
summary: "Neste post, mostramos o passo a passo para instalar e configurar o Docker Engine no IBM Power9 (ppc64le), incluindo remoção de pacotes conflitantes, validação da instalação e boas práticas para compatibilidade de imagens."
draft: false
---

## Contexto
Este post faz parte da série de tutoriais sobre como construir uma infraestrutura de modelos de linguagem no servidor IBM Power9. Após configurar o sistema operacional e os drivers da NVIDIA, o próximo passo é instalar o Docker Engine, ferramenta essencial para empacotar e executar aplicações em contêineres de forma isolada e reproduzível.

O Docker Engine tem suporte oficial para Rocky Linux nas arquiteturas x86_64, arm64, s390x e ppc64le, o que permite uso direto no Power9 sem adaptações especiais. Ainda assim, alguns cuidados são necessários durante a instalação, como remover ferramentas que conflitam com o Docker e garantir que as imagens escolhidas sejam compatíveis com ppc64le.

## TL;DR
* Este post apresenta o passo a passo para instalar o Docker Engine no Rocky Linux/AlmaLinux na arquitetura ppc64le.
* É necessário remover Podman e Buildah antes da instalação, pois esses pacotes conflitam com o Docker.
* Imagens do Docker Hub precisam ter suporte explícito a ppc64le para funcionar corretamente no Power9.

## Ambiente utilizado
* **Arquitetura**: Servidor IBM Power9 (Arquitetura ppc64le).
* **Sistema Operacional (SO)**: AlmaLinux 8.10 binário compatível com Red Hat Enterprise Linux (RHEL) 8.9/8.10.
* **RAM**: 512GB.

## Pré-requisitos
Antes de instalar o Docker, considere uma limitação importante de firewall: ao expor portas de contêineres usando Docker, essas portas ignoram as regras padrão do firewalld. Verifique se esse comportamento é aceitável para o seu ambiente antes de prosseguir.

Também é importante reforçar que o Docker Engine é compatível com Rocky Linux 8 e 9 e com AlmaLinux 8 na arquitetura ppc64le.

## Instalação do Docker Engine no Power9
1. **Removendo pacotes conflitantes**:
O Rocky Linux geralmente traz Podman e Buildah por padrão. Esses pacotes conflitam com o Docker Engine e devem ser removidos, junto com versões antigas do próprio Docker, caso existam:

```
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

2. **Adicionando repositório oficial do Docker**:
O método recomendado é usar o repositório oficial. Para distribuições baseadas em RHEL, como o AlmaLinux, o Docker utiliza o repositório do CentOS, fluxo oficialmente suportado.

Instale o plugin de gerenciamento de repositórios e adicione o repositório:

```
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

3. **Instalando os pacotes do Docker**:
Com o repositório configurado, instale o Docker Engine e os plugins de build e compose:

```
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

4. **Iniciando e habilitando o serviço**:
No Rocky Linux/AlmaLinux, o serviço do Docker não inicia automaticamente após a instalação. Inicie manualmente e habilite no boot:

```
sudo systemctl start docker
sudo systemctl enable docker
```

5. **Verificando a instalação**:
Para confirmar que tudo foi instalado corretamente, execute a imagem de teste:

```
sudo docker run hello-world
```

A saída esperada é uma mensagem confirmando que o Docker está funcionando corretamente.

## Configurações pós-instalação
Por padrão, apenas root (ou usuários com sudo) podem executar comandos Docker. Para usar Docker sem sudo em todos os comandos:

1. **Criando o grupo docker (se necessário)**:

```
sudo groupadd docker
```

2. **Adicionando o usuário ao grupo**:

```
sudo usermod -aG docker $USER
```

É necessário fazer logout e login novamente para aplicar as permissões.

## Dicas para arquitetura Power9
No IBM Power9, nem todas as imagens do Docker Hub são compatíveis com ppc64le. Imagens publicadas somente para x86_64 irão falhar na execução. Sempre verifique se a imagem possui suporte explícito à arquitetura ppc64le.

Para validar se o daemon Docker está ativo e reconhecendo corretamente a arquitetura do servidor, execute:

```
docker version --format '{{.Server.Arch}}'
```

A saída esperada é:

```
ppc64le
```

## Considerações finais
A instalação do Docker Engine no Rocky Linux/AlmaLinux (ppc64le) segue um fluxo direto, desde que os conflitos com Podman e Buildah sejam resolvidos antes da instalação.

Com suporte oficial à arquitetura ppc64le, o Docker oferece uma base estável para execução de contêineres no Power9. O principal cuidado contínuo está na seleção de imagens compatíveis com a arquitetura.

Com o Docker instalado e configurado, o ambiente fica pronto para avançar para as próximas etapas da infraestrutura de modelos de linguagem.
