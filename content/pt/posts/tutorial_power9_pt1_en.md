---
title: "Configurando S.O, NVIDIA Drivers, CUDA e CUDNN em um servidor IBM Power9"
date: 2025-06-29 # ano-mês-dia
authors: ["Caio Silva"] # Pode ser uma lista
tags: ["LLM", "Power9", "Drivers", "CUDA", "NVIDIA"]
projects: ["llm-eval"]
translationKey: "tutorial_power9_pt1_en"
summary: "Este post faz parte de uma série de tutoriais que tem como objetivo final construir uma API de Modelos de Linguagem em servidor Power9. Nesta etapa vamos configurar o sistema operacional, instalar drivers NVIDIA, CUDA e CUDNN."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto
Este é o primeiro post de uma série de tutoriais sobre como construir uma API de Modelos de Linguagem em um servidor Power9, desde da configuração do Sistema Operacional, até a API executando inferências de forma remota. 
Esta etapa do tutorial mostra como configurar o Sistema Operacional, instalar os drivers da NVIDIA, CUDA e CUDNN em máquinas com processador IBM Power9 AC922. O foco é garantir que tudo funcione corretamente em arquiteturas ```ppc64le```, comuns em ambientes de alto desempenho.

**IBM Power9**: A IBM Power9 AC922 é uma máquina de alto desempenho usada em tarefas pesadas como inteligência artificial e processamento científico. Ela usa processadores Power9 e trabalha bem com GPUs NVIDIA, oferecendo alta velocidade de comunicação entre CPU e GPU.

**NVIDIA Drivers**: Programas que permitem que o sistema operacional se comunique corretamente com as placas de vídeo da marca. São essenciais para ativar o uso de GPUs.

**CUDA**: Plataforma NVIDIA que permite usar GPUs para acelerar cálculos paralelos. Com essa plataforma é possível rodar algoritmos complexos de forma rápida, como a execução de Grandes Modelos de Linguagem, por exemplo. 

**CUDNN**: Uma biblioteca de primitivas otimizadas para redes neurais profundas (DNNs), desenvolvida pela NVIDIA. Ele oferece implementações de alto desempenho para operações essenciais em DNNs, como convoluções, pooling e normalização, acelerando significativamente o treinamento e a inferência em GPUs.

## TL;DR
- Este post apresenta o passo-a-passo de configurar um servidor Power9 incluindo setup do SO e configurações NVIDIA. 
- O desafio maior é encontrar versões compatíveis com a arquitetura das máquinas Power.

## Configurando Sistema Operacional
Vamos começar com a instalação do **Red Hat Enterprise Linux 8.10 (Ootpa)**. Em sistemas Power, a arquitetura usada é a ```ppc64le``` (PowerPC 64 bits little-endian), por isso é essencial que a imagem .iso seja compatível com essa arquitetura. Caso contrário, o petitboot da Power9 não reconhecerá a mídia e a instalação não poderá continuar.
1. Você pode baixar a imagem correta pelo [<span class="link-personalizado">link</span>](https://access.redhat.com/downloads/content/279/ver=/rhel---8/8.10/ppc64le/product-software) indicado. 
2. Neste tutorial, usaremos a opção **Boot ISO** e seguiremos as instruções da [<span class="link-personalizado">documentação oficial da Red Hat</span>](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/interactively_installing_rhel_from_installation_media/assembly_creating-a-bootable-installation-medium_rhel-installer) para criar uma mídia USB inicializável.
3. Após inserir a mídia de instalação no servidor Power 9 e reiniciar a máquina, o sistema deve iniciar automaticamente no petitboot. 
4. A partir desta etapa, basta seguir o [<span class="link-personalizado">guia de instalação</span>](https://www.ibm.com/docs/en/linuxonibm/liabw/rhelqs_guide_Power_p9_usb.pdf) oficial para concluir a configuração do sistema. 


## Configurando Driver NVIDIA e CUDA
#### Checagem de GPUs e Sistema Operacional


Para o sistema operacional realizar comunicação correta com as GPUs do servidor, precisamos instalar e configurar o driver da NVIDIA. 

1. Inicialmente, vamos checar a presença da(s) GPU(s): 

```
lspci | grep -i nvidia
```

A saída esperada é algo como: 

```0004:04:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 SXM2 16GB] (rev a1)```

2. Após isso, vamos verificar arquitetura e nome do sistema operacional: 
```
uname -m && cat /etc/redhat-release
```
A saída esperada é: 


```ppc64le Red Hat Enterprise Linux release 8.10 (Ootpa)```

#### Evitando interferências

Para evitar algumas interferências, é recomendável desativar o driver ```nouveau``` e ```SELinux```.

O ```noveau``` é um driver de código aberto para GPUs NVIDIA que subsitui o driver proprietário quando o usuário quer apenas usar o software livre, sem necessidade de de alto desempenho.

O ```SELinux=enable``` restringe alguns processos de aplicarem mudanças no sistema, podendo conflitar com as instalações que vamos fazer neste tutorial.

1. Desative o driver ```nouveau```: 
```
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/disable-nouveau.conf
```
2. Para desativar o ```SELinux```, primeiro vamos checar o status executando: 
```
sestatus
```
Caso esteja ativo, será preciso setar o parâmetro ```SELINUX=disabled``` no arquivo ```/etc/selinux/config``` para prosseguir. É importante lembrar que a edição só será salva com permissão sudo. 


3. Após isso, vamos atualizar o ```initrafms``` e reiniciar a máquina com os seguintes comandos: 

```
sudo dracut --force
sudo reboot
```

4. Para checar se tudo deu certo até agora, vamos checar se o ```nouveau``` foi desabilitado: 

```
lsmod | grep nouveau
```
Caso tenha sido desabilitado, não terá saída. 


5. Para checar o SELinux:

```
sestatus
```
Caso tenha sido desabilitado, a saída será:  ```SELinux status: disabled```

#### Instalando pré-requisitos
1. Vamos instalar alguns pré-requisitos antes de iniciar a instalação de fato:

```
sudo dnf install pciutils environment-modules
sudo dnf install kernel-devel-$(uname -r) kernel-headers
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf clean all 
sudo dnf install dkms

```
2. Também precisamos habilitar alguns repositórios: 
```
sudo subscription-manager repos --enable=rhel-8-for-ppc64le-appstream-rpms
sudo subscription-manager repos --enable=rhel-8-for-ppc64le-baseos-rpms
sudo subscription-manager repos --enable=codeready-builder-for-rhel-8-ppc64le-rpms
```

#### Baixando e instalando repositórios dos pacotes CUDA
1. Vamos baixar a versão **12.2 do CUDA** e o **Driver NVIDIA 535.54.03-1** com o comando seguinte: 

```
wget https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda-repo-rhel8-12-2-local-12.2.0_535.54.03-1.ppc64le.rpm

```
2. Para instalar o pacote baixado:

```
sudo rpm -i cuda-repo-rhel8-12-2-local-12.2.0_535.54.03-1.ppc64le.rpm
```

3. Para instalar o driver NVIDIA e o CUDA, os seguintes comandos serão executados:
```
sudo dnf install nvidia-driver-cuda 
sudo dnf clean all 
sudo dnf module reset nvidia-driver 
sudo dnf module enable nvidia-driver:latest-dkms
sudo dnf -y module install nvidia-driver:latest-dkms
sudo dnf -y install cuda      

```

Com esses comandos a instalação do driver e do CUDA estão finalizadas.

#### Processos pós-instalação
1. Vamos declarar as variáveis de ambiente ```PATH``` e ```LD_LIBRARY_PATH```. Para isso, deve-se editar o arquivo ```.bashrc``` e adicionar essas duas linhas:

```
export PATH=/usr/local/cuda/bin:$PATH 
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
```
Para atualizar as variáveis de ambiente, vamos executar o comando: 
```
source ~/.bashrc
```
Precisamos realizar duas mudanças de forma manual, pois não são tratadas de forma automática pela instalação dos pacotes CUDA. Caso não sejam realizadas, a instalação do driver CUDA ficará inoperante. 

2. A primeira mudança será configurar o ```deamon``` de persistência da NVIDIA. Primeiro vamos verificar o status e caso não esteja ativo, vamos ativar:
```
systemctl status nvidia-persistenced
systemctl enable nvidia-persistenced
```

Algumas distros Linux possuem uma regra do udev que coloca a memória hot-plug em estado online assim que é detectada fisicamente, impedindo que o software da NVIDIA configure a memória da GPU com os parâmetros corretos no Power9.


3. Para desativar esta regra, vamos executar os comandos: 
```
sudo cp /lib/udev/rules.d/40-redhat.rules /etc/udev/rules.d/
sudo sed -i 's/SUBSYSTEM!="memory",.*GOTO="memory_hotplug_end"/SUBSYSTEM=="*", GOTO="memory_hotplug_end"/' /etc/udev/rules.d/40-redhat.rules
```

#### Checagem de instalação
Após realizar todos esses procedimentos, vamos reiniciar a máquina e checar as instalações:

1. Reiniciando a máquina:

```
sudo reboot
```

2. Checagem de driver NVIDIA:
```
nvidia-smi
```
A saída do comando acima deve mostrar informações do compilador CUDA: versão e data de instalação. Além de mostrar os dispositivos (GPUs) disponíveis com nome, memória, temperatura entre outras informações.


Para realizar a última checagem, vamos baixar o repositório ```cuda-samples``` e executar o teste de dispositivos. 

3. Baixando o repositório e acessando a versão do ```cuda-samples``` referente ao CUDA instalado:

```
git clone https://github.com/NVIDIA/cuda-samples.git 
cd cuda-samples/Samples/1_Utilities/deviceQuery
git checkout v12.2 
```

4. Para buildar e executar os testes:
```
make
./deviceQuery
```

Após executar este teste, espera-se que na última linha contenha: ```Result = PASS```. Com isso, a Power9 está configurada, com driver NVIDIA e CUDA funcionando corretamente.

## Configurando CUDNN

1. Inicialmente, precisamos baixar e instalar o ```.rpm``` específico para ```ppc64le```.

```
wget https://developer.download.nvidia.com/compute/cudnn/9.0.0/local_installers/cudnn-local-repo-rhel8-9.0.0-1.0-1.ppc64le.rpm
sudo rpm -i cudnn-local-repo-rhel8-9.0.0-1.0-1.ppc64le.rpm
sudo dnf clean all
sudo dnf -y install cudnn
```

2. Após a instalação, precisamos configurar as variáveis de ambiente ```CUDNN_LIBRARY``` e ```CUDNN_INCLUDE_DIR```: (De uma forma mais direta do que fizemos anteriormente)

```
echo 'export CUDNN_LIBRARY=/usr/lib64' >> ~/.bashrc    
echo 'export CUDNN_LIBRARY=/usr/lib64' >> ~/.bashrc 
```

Após isso, o processo de instalação do CUDNN está finalizado. 

Esta é a primeira parte do nosso tutorial. Uma vez que todas as etapas mostradas neste post foram finalizadas, o servidor está pronto para ter o gerenciador de pacotes ```conda``` e a biblioteca ```pytorch``` instaladas, você pode acessar a segunda parte deste tutorial neste [<span class="link-personalizado">link</span>]({{< relref "tutorial_power9_pt2_en.md" >}}).