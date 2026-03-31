---
title: "Setting Up the OS, NVIDIA Drivers, CUDA, and cuDNN on IBM Power 9 Servers"
date: 2025-06-29 # ano-mês-dia
authors: ["Caio Silva"] # Pode ser uma lista
tags: ["LLM", "Power9", "Drivers", "CUDA", "NVIDIA"]
projects: ["llm-eval"]
translationKey: "tutorial_power9_pt1_en"
summary: "This post is part of a tutorial series aimed at building a Large Language Model API on Power9 servers. In this step, we'll set up the operating system and install NVIDIA drivers, CUDA, and cuDNN."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Background
This is the first post in a tutorial series on how to build a Language Model API on an IBM Power9 server, covering everything from setting up the operating system to having the API running remote inference.
This step of the tutorial shows how to set up the operating system and install NVIDIA drivers, CUDA, and cuDNN on machines with IBM Power9 AC922 processors. The focus is on ensuring everything works correctly on `ppc64le` architectures, which are common in high-performance environments.

**IBM Power9**: The IBM Power9 AC922 is a high-performance machine used for demanding tasks such as artificial intelligence and scientific computing. It uses Power9 processors and works well with NVIDIA GPUs, offering high-speed communication between the CPU and GPU.

**NVIDIA Drivers**: Software that allows the operating system to communicate correctly with NVIDIA GPUs. These drivers are essential to enable GPU acceleration.

**CUDA**: NVIDIA's platform for accelerating parallel computing on GPUs. It lets you run complex algorithms efficiently, such as Large Language Model inference.

**cuDNN**: A GPU-optimized library of primitives for deep neural networks (DNNs) developed by NVIDIA. It offers high-performance implementations of key DNN operations like convolutions, pooling, and normalization, significantly speeding up training and inference on GPUs.



## TL;DR
- This post provides a step-by-step guide on setting up Power9 servers, including the OS and NVIDIA configurations.
- The main challenge is finding compatible versions for the Power9 machine architecture.


## Setting up the Operating System
Let's start with the installation of **Red Hat Enterprise Linux 8.10 (Ootpa)**. On Power systems, the architecture used is `ppc64le` (PowerPC 64-bit little-endian), so it's essential to ensure the .iso image is compatible with this architecture. Otherwise, the Power9's petitboot won't recognize the media and installation won't proceed.

1. You can download the correct image from the [<span class="link-personalizado">link</span>](https://access.redhat.com/downloads/content/279/ver=/rhel---8/8.10/ppc64le/product-software) provided.
2. In this tutorial, we'll use the **Boot ISO** option and follow the [<span class="link-personalizado">official Red Hat documentation</span>](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/interactively_installing_rhel_from_installation_media/assembly_creating-a-bootable-installation-medium_rhel-installer) to create a bootable USB medium.
3. After inserting the installation media into the Power9 server and rebooting, the system should automatically start petitboot.
4. From there, just follow the [<span class="link-personalizado">official installation guide</span>](https://www.ibm.com/docs/en/linuxonibm/liabw/rhelqs_guide_Power_p9_usb.pdf) to complete the OS setup.



## Setting up NVIDIA Driver and CUDA
#### Checking GPUs and Operating System
To enable the operating system to communicate properly with the server's GPUs, we need to install and configure the NVIDIA driver.

1. First, let's check for the presence of the GPU(s): 
```
lspci | grep -i nvidia
```

The expected output is something like: 

```0004:04:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 SXM2 16GB] (rev a1)```

2. Next, let's check the system architecture and operating system name:
```
uname -m && cat /etc/redhat-release
```
The expected output is: 


```ppc64le Red Hat Enterprise Linux release 8.10 (Ootpa)```

#### Avoiding conflicts

To avoid potential conflicts, it's recommended to disable the `nouveau` driver and `SELinux`.

The `nouveau` driver is an open-source driver for NVIDIA GPUs that replaces the proprietary driver when users want to use only free software without needing high performance.

`SELinux=enable` restricts certain processes from making changes to the system, which can conflict with the installations we'll do in this tutorial.

1. Disable the `nouveau` driver:
```
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/disable-nouveau.conf
```
2. To disable `SELinux`, let's first check its status by running:
```
sestatus
```
If it's active, you'll need to set the `SELINUX=disabled` parameter in the `/etc/selinux/config` file to proceed. Remember that saving changes requires sudo permissions.

3. After that, update the `initramfs` and reboot the machine with the following commands:
```
sudo dracut --force
sudo reboot
```

4. To verify everything worked so far, let's check if `nouveau` is disabled:

```
lsmod | grep nouveau
```
If it's been successfully disabled, there will be no output.

5. To verify the `SELinux`:

```
sestatus
```
If it's disabled, the output will be:  ```SELinux status: disabled```

#### Installing Prerequisites
1. Let's install some prerequisites before starting the actual installation:

```
sudo dnf install pciutils environment-modules
sudo dnf install kernel-devel-$(uname -r) kernel-headers
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf clean all 
sudo dnf install dkms

```
2. We also need to enable some repositories: 
```
sudo subscription-manager repos --enable=rhel-8-for-ppc64le-appstream-rpms
sudo subscription-manager repos --enable=rhel-8-for-ppc64le-baseos-rpms
sudo subscription-manager repos --enable=codeready-builder-for-rhel-8-ppc64le-rpms
```
#### Downloading and Installing CUDA Package Repositories
1. Let's download **CUDA version 12.2** and **NVIDIA Driver 535.54.03-1** with the following command:

```
wget https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda-repo-rhel8-12-2-local-12.2.0_535.54.03-1.ppc64le.rpm

```
2. To install the downloaded package:

```
sudo rpm -i cuda-repo-rhel8-12-2-local-12.2.0_535.54.03-1.ppc64le.rpm
```

3. To install the NVIDIA driver and CUDA, run the following commands:
```
sudo dnf install nvidia-driver-cuda 
sudo dnf clean all 
sudo dnf module reset nvidia-driver 
sudo dnf module enable nvidia-driver:latest-dkms
sudo dnf -y module install nvidia-driver:latest-dkms
sudo dnf -y install cuda      

```

With these commands, the driver and CUDA installation is complete.

#### Post-Installation Steps
1. Let's set the `PATH` and `LD_LIBRARY_PATH` environment variables. To do this, edit the `.bashrc` file and add these two lines:

```
export PATH=/usr/local/cuda/bin:$PATH 
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
```
To update the environment variables, run the following command:

```
source ~/.bashrc
```
We need to make two manual changes because they aren't handled automatically by the CUDA package installation. If these aren't done, the CUDA driver installation will not work properly.

2. The first change is to configure the NVIDIA persistence daemon. First, check its status, and if it's not active, enable it:

```
systemctl status nvidia-persistenced
systemctl enable nvidia-persistenced
```
Some Linux distributions have a udev rule that brings hot-plugged memory online as soon as it's detected, preventing NVIDIA software from correctly configuring GPU memory on Power9.

3. To disable this rule, run the following commands:

```
sudo cp /lib/udev/rules.d/40-redhat.rules /etc/udev/rules.d/
sudo sed -i 's/SUBSYSTEM!="memory",.*GOTO="memory_hotplug_end"/SUBSYSTEM=="*", GOTO="memory_hotplug_end"/' /etc/udev/rules.d/40-redhat.rules
```

#### Installation Check
After completing all these steps, let's reboot the machine and verify the installations:

1. Reboot the machine:

```
sudo reboot
```
2. Check the NVIDIA driver:

```
nvidia-smi
```
The output of the command above should display CUDA compiler information: version and install date. It should also list available devices (GPUs) with details like name, memory, temperature, and other information.

To perform the final check, let's download the `cuda-samples` repository and run the device test.

3. Download the repository and access the `cuda-samples` version matching the installed CUDA:

```
git clone https://github.com/NVIDIA/cuda-samples.git 
cd cuda-samples/Samples/1_Utilities/deviceQuery
git checkout v12.2 
```

4. To build and run the tests:
```
make
./deviceQuery
```
After running this test, you should see `Result = PASS` in the last line. This confirms that the Power9 is set up with the NVIDIA driver and CUDA working correctly.

## Setting up the CUDNN

1. First, we need to download and install the `.rpm` package specific to `ppc64le`.
```
wget https://developer.download.nvidia.com/compute/cudnn/9.0.0/local_installers/cudnn-local-repo-rhel8-9.0.0-1.0-1.ppc64le.rpm
sudo rpm -i cudnn-local-repo-rhel8-9.0.0-1.0-1.ppc64le.rpm
sudo dnf clean all
sudo dnf -y install cudnn
```
2. After installing, set the `CUDNN_LIBRARY` and `CUDNN_INCLUDE_DIR` environment variables directly by adding these lines to your `.bashrc`:

```
echo 'export CUDNN_LIBRARY=/usr/lib64' >> ~/.bashrc    
echo 'export CUDNN_LIBRARY=/usr/lib64' >> ~/.bashrc 
```
After that, the CUDNN installation process is complete.

This is the first part of our tutorial. Once you've finished all the steps in this post, the server will be ready to install the `conda` package manager and the `pytorch` library. You can access the second part of this tutorial at this [<span class="link-personalizado">link</span>]({{< relref "tutorial_power9_pt2_en.md" >}}).
