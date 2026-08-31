---
title: "Virtualização em Power9: como estruturamos um ambiente isolado com KVM e Libvirt"
date: 2026-03-27
authors: ["Gabrielly Lima"]
tags: ["Virtualização", "Power9", "KVM", "Libvirt"]
projects: ["multiarq"]
translationKey: "power9-virtualization"
summary: "Neste post, exploramos a construção de um ambiente virtualizado utilizando KVM e Libvirt em um servidor Power9, com foco em isolamento, reprodutibilidade e uso compartilhado entre membros de uma equipe."
draft: false
---

## Contexto
Diante da necessidade de estabelecer ambientes isolados e seguros para a instalação de bibliotecas, frameworks e ferramentas de uso geral, o encapsulamento de um ambiente surgiu como alternativa para resolução desse problema, fazendo-se presente através do KVM gerenciado por meio do virt-manager e do virsh.

A virtualização é amplamente utilizada em ambientes x86, com ferramentas e fluxos bem consolidados. No entanto, quando migramos para arquiteturas como o IBM Power9 (ppc64le), muitos desses processos deixam de ser diretos e exigem adaptações específicas. Abaixo, temos um diagrama que demonstra essa comunicação dividida em 4 camadas.

## Fluxo de comunicação entre Hardware (Power9) e Máquinas Virtuais
O fluxo é organizado nas seguintes camadas:

<!--<div style="text-align: center;">
  <img src="/images/kvm_virtualizacao.png" style="max-width: 90%;">
</div> -->

{{< figure src="/images/kvm_virtualizacao.png" alt="Figura 1: Diagrama que representa a arquitetura de virtualização em 4 camadas." caption="Figura 1: Diagrama que representa a arquitetura de virtualização em 4 camadas." >}}

Neste trabalho, exploramos a construção de um ambiente virtualizado utilizando KVM e Libvirt em um servidor Power9, com foco em isolamento, reprodutibilidade e uso compartilhado entre membros de uma equipe.

## TL;DR
* Implementamos um ambiente virtualizado no Power9 usando KVM + Libvirt.
* Adaptamos fluxos comuns de virtualização para arquitetura ppc64le, resolvendo problemas de permissão, lock de escrita e provisionamento.
* O ambiente permite isolamento seguro entre usuários e fácil gerenciamento de VMs.
* Disponibilizamos imagens prontas com drivers NVIDIA/CUDA para uso imediato.

## Ambiente utilizado
* **Arquitetura**: Servidor IBM Power9 (Arquitetura ppc64le).
* **Sistema Operacional (SO)**: AlmaLinux 8.10 binário compatível com Red Hat Enterprise Linux (RHEL) 8.9/8.10.
* **RAM**: 512GB.
* **Execução**: Virtual Manager para gerenciamento de Máquinas Virtuais (VMs).
* **Hypervisor**: KVM (Kernel-based Virtual Machine) / QEMU.
* **Gerenciamento**: Libvirt (virsh, virt-install, virt-customize).
* **Armazenamento**: Discos virtuais no formato .qcow2.
* **GPUs**: 4x NVIDIA Tesla V100 SXM2 16GB (NVLink2).

## Instalando o ambiente de virtualização (KVM + Libvirt)
Antes de criar qualquer VM, é necessário instalar e configurar o KVM e o Libvirt no servidor Power9.

1. **Instalação dos pacotes**:
```bash
sudo dnf install -y qemu-kvm libvirt libvirt-client libvirt-daemon libvirt-daemon-kvm virt-install virt-viewer guestfs-tools \
libguestfs-tools python3-libvirt
```

2. **Iniciando serviço**:
```bash
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
```

3. **Adicionando o usuário ao grupo libvirt**:
Para que usuários não-root possam gerenciar VMs sem precisar de sudo em todo comando:

Execute o comando abaixo:
```bash
sudo usermod -aG libvirt $(whoami)
```

Faça logout e login novamente para aplicar a mudança.

4. **Verificando instalação**:

Verifique a versão do `virsh`:
```bash
sudo virsh version
```

Valide o suporte à virtualização no processador:
```bash
sudo virt-host-validate
```

## Setup

1. **Preparação de ambiente**:
No KVM, a forma mais rápida de provisionar VMs é clonar uma imagem "semente" (.qcow2) e expandi-la, em vez de fazer uma instalação limpa via ISO. Portanto, para manter a organização, todos os discos virtuais ficarão em um diretório separado:

Baixe a imagem base do Alma Linux 8:
```bash
cd /home/user/
wget https://repo.almalinux.org/almalinux/8/cloud/ppc64le/images/AlmaLinux-8-GenericCloud-latest.ppc64le.qcow2 -O alma8_base.qcow2
```

2. **Gerenciamento do Hipervisor**:
A administração do hipervisor e das instâncias segue protocolos específicos para garantir a estabilidade do sistema. Comandos para o Administrador controlar o serviço no Power9:

Desative o sistema KVM:
```bash
sudo systemctl stop libvirtd
```

Reative o sistema KVM:
```bash
sudo systemctl start libvirtd
```

Habilite no boot:
```bash
sudo systemctl enable libvirtd
```

3. **Resolução de permissões**:
O usuário do sistema que executa o KVM (chamado qemu) precisa ter permissão para acessar os discos da VM. Se o diretório estiver dentro de uma home pessoal, o Linux bloqueará o acesso por padrão. Para permitir que o hipervisor acesse a pasta de discos sem expor seus arquivos pessoais, conceda permissão de execução (o+x) nos diretórios:

Permita que o qemu "atravesse" a home (apenas travessia, não leitura):
```bash
chmod o+x /home/user
```

Permita que o qemu acesse a pasta de discos:
```bash
chmod o+x /home/user/discos
```

4. **Configuração de rede virtual (Libvirt)**:
O Libvirt cria uma rede NAT padrão (default) que coloca as VMs na faixa 192.168.122.0/24. As VMs têm acesso à internet via NAT, mas não são acessíveis diretamente da rede externa sem configuração adicional.

Verifique o status da rede:
```bash
sudo virsh net-list --all
```

Se estiver inativa, inicie e habilite no boot:
```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

Se a rede não existir, defina e inicialize:
```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

Se o XML não for encontrado, instale o pacote de configuração de rede:
```bash
sudo dnf install -y libvirt-daemon-config-network
```

5. **Criando novas VMs**:

Clone a imagem base:
```bash
cp /home/user/alma8_base.qcow2 /home/user/discos/nome_vm.qcow2
```

Expanda o disco (a expansão deve ser feita ANTES de criar a VM):
```bash
qemu-img resize /home/user/discos/nome_vm.qcow2 +100G
```

Crie a VM:
```bash
sudo virt-install \
 --connect qemu:///system \
 --name vm_nome \
 --memory 131072 \
 --vcpus 16 \
 --cpu host \
 --disk path=/home/user/discos/nome_vm.qcow2,format=qcow2 \
 --import \
 --os-variant almalinux8 \
 --network network=default \
 --graphics none \
 --noautoconsole
```

6. **Customização após criar as VMs**:
Após criar a VM, é necessário definir a senha root, pois a imagem cloud vem sem senha por padrão. Utilizamos o virt-customize para isso. **Importante**: A VM deve estar desligada para que o disco possa ser editado em segurança.

Desligue a VM:
```bash
sudo virsh shutdown vm_nome
```

Aguarde o desligamento completo:
```bash
sudo virsh list --all
```

Injete a senha no disco:
```bash
sudo virt-customize -a /home/user/discos/nome_vm.qcow2 \
 --root-password password:senha_desejada
```

Ligue a VM novamente:
```bash
sudo virsh start vm_nome
```

7. **Acessando VMs**:

**Via console serial**

Conecte ao console da VM:
```bash
sudo virsh console vm_nome
```

Para sair do console, use `Ctrl + ]`.

**Via SSH**

Descubra o IP da VM:
```bash
sudo virsh domifaddr vm_nome
```

Acesse via SSH:
```bash
ssh root@<ip_da_vm>
```

8. **Gerenciar e apagar VMs**:
Se você precisar destruir um ambiente para recriá-lo do zero, siga os 3 passos obrigatórios para limpar tudo:

Force o desligamento da VM:
```bash
sudo virsh destroy nome_da_vm
```

Remova a definição da VM do Libvirt:
```bash
sudo virsh undefine nome_da_vm
```

Apague o disco virtual para liberar espaço no Power9:
```bash
rm -f /home/user/discos/nome_da_vm.qcow2
```

9. **Criar VM a partir de imagem existente (clonagem)**:
Para criar uma nova VM a partir de uma imagem já configurada, como as imagens prontas com drivers NVIDIA:

Opção A: clonar via `qemu-img` (mantém a imagem original intacta):
```bash
qemu-img create -f qcow2 -b imagem-base.qcow2 -F qcow2 nova-vm.qcow2
```

Opção B: clonar via `virt-clone`:
```bash
virt-clone \
 --original vm-base \
 --name vm-nova \
 --file /home/user/discos/nova-vm.qcow2
```
Caso seja necessário, pode-se executar o passo de excluir a VM e recriá-la conforme a etapa 5.

## Imagens prontas com drivers NVIDIA
Para facilitar o uso das GPUs Tesla V100 presentes no servidor, disponibilizamos imagens .qcow2 pré-configuradas com os drivers NVIDIA, CUDA e cuDNN instalados. Isso elimina a necessidade de configurar o ambiente base a cada novo uso.

1. **Imagens disponíveis**:
| Imagem | Conteúdo |
| :--- | :--- |
| AlmaLinux-8-Power9-NVIDIA-drivers.qcow2.xz | AlmaLinux 8.10 + drivers NVIDIA 535 + CUDA 12.2 + cuDNN 9.0 | 
| InstructLab-Power9-0.25.0.qcow2.xz | AlmaLinux 8.10 + InstructLab 0.25.0 + dependências necessárias para execução no Power9 (ppc64le). |

2. **Como usar imagens pré-configuradas**:

Baixe a imagem da pasta compartilhada e descompacte:
```bash
pip install --user gdown
gdown --folder "https://drive.google.com/drive/u/1/folders/1WM8fHKWaMu-NJOzwqh6cdcET7mNE50du"
xz -d InstructLab-Power9-0.25.0.qcow2.xz
```

Mova para o diretório de discos e crie a VM a partir dela:
```bash
cp InstructLab-Power9-0.25.0.qcow2 /home/user/discos/minha-vm-gpu.qcow2
```

Crie a VM normalmente:
```bash
sudo virt-install \
 --connect qemu:///system \
 --name vm_gpu \
 --memory 131072 \
 --vcpus 16 \
 --cpu host \
 --disk path=/home/user/discos/minha-vm-gpu.qcow2,format=qcow2 \
 --import \
 --os-variant almalinux8 \
 --network network=default \
 --graphics none \
 --noautoconsole
```

Para que a VM tenha acesso às GPUs físicas, é necessário configurar o passthrough PCIe conforme descrito no próximo post desta série.

3. **Como gerar nova imagem a partir de VM configurada**:
Após instalar drivers ou qualquer software dentro de uma VM, você pode exportar o estado atual como nova imagem para reuso:

Desligue a VM:
```bash
sudo virsh shutdown vm_nome
```

Converta e compacte a imagem (remove espaço não utilizado):

```bash
qemu-img convert -O qcow2 -c \
 /home/user/discos/vm_nome.qcow2 \
 /home/user/discos/AlmaLinux-8-Power9-minha-imagem.qcow2
```

Comprima para distribuição:
```bash
xz -T0 -v /home/user/discos/AlmaLinux-8-Power9-minha-imagem.qcow2
```

Saída esperada: `AlmaLinux-8-Power9-minha-imagem.qcow2.xz`.

Verifique a integridade:
```bash
qemu-img check AlmaLinux-8-Power9-minha-imagem.qcow2
qemu-img info AlmaLinux-8-Power9-minha-imagem.qcow2
```