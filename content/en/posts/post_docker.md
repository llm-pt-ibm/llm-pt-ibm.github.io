---
title: "Installing Docker in an Architecture ppc64le (Power9) Environment"
date: 2026-04-01
authors: ["Gabrielly Lima"]
tags: ["Docker", "Power9", "ppc64le", "AlmaLinux", "Containers"]
projects: ["multiarq"]
translationKey: "power9-docker-installation"
summary: "This post walks through how to install Docker Engine on AlmaLinux 8.10 on IBM Power9 (ppc64le), including removing conflicting packages, configuring the official repository, and handling image compatibility concerns."
draft: false
---

## Context
Given the need to standardize software execution on our IBM Power9 (ppc64le) server, containers are a robust solution for avoiding environment conflicts. This post continues the work of structuring our infrastructure by detailing the installation of Docker Engine on AlmaLinux. Adopting this technology is strategically important for ensuring strict dependency isolation and portability across applications. With it, we can package everything from general-purpose libraries to more complex services, ensuring a clean, secure, and highly reproducible runtime environment.

Docker Engine has official support for AlmaLinux on the x86_64, arm64, s390x, and ppc64le architectures, which allows us to use it directly on Power9 without special adaptations. However, some care is required before and during installation, such as uninstalling tools that conflict with Docker and ensuring the images used are compatible with ppc64le.

## TL;DR
* This post presents the step-by-step process for installing Docker Engine on AlmaLinux in the ppc64le architecture.
* You must remove Podman and Buildah before installing, because they conflict with Docker.
* Docker Hub images need explicit ppc64le support to work on Power9.

## Environment Used
* **Architecture**: IBM Power9 server (ppc64le architecture)
* **Operating System (OS)**: AlmaLinux 8.10 binary compatible with Red Hat Enterprise Linux (RHEL) 8.9/8.10
* **RAM**: 512GB

## Prerequisites
Before installing Docker, it is important to be aware of a firewall limitation: when exposing container ports with Docker, those ports bypass the default firewalld rules. Make sure this does not pose a problem for your environment before proceeding. It is also important to note that Docker Engine is compatible with Rocky Linux 8 and 9 and AlmaLinux 8 on the ppc64le architecture.

## Removing Conflicting Packages
AlmaLinux includes Podman and Buildah by default. These packages conflict with Docker Engine and must be removed before installation. It is also recommended to remove any older Docker versions that might be present:

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

## Adding the Docker Repository and Installing Required Packages
### Repository Setup
The recommended installation method is to use Docker's official repository. It is worth mentioning that Docker uses the CentOS repository for RHEL-based distributions such as AlmaLinux, and this is officially supported. First, install the dnf-plugins-core package and add the repository:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### Installing Docker Engine
With the repository configured, install the latest version of Docker Engine along with the build and compose plugins:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Starting the service
Unlike Debian-based distributions such as Ubuntu, Docker does not start automatically on AlmaLinux after installation. You need to start the service manually and enable it so it comes up with the system:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

## Verifying the Installation
To confirm that everything was installed correctly, run the hello-world image. Docker will automatically detect the ppc64le architecture and pull the correct image:

```bash
sudo docker run hello-world
```

The expected output is a message confirming that Docker is working correctly.

## Post-installation Configuration
By default, only the root user or users with sudo privileges can run Docker commands. To avoid using sudo on every command, add your user to the docker group. First, create the group if it does not already exist:

```bash
sudo groupadd docker
```

Then add your user to the group:

```bash
sudo usermod -aG docker $USER
```

You need to log out and log back in for the permissions to take effect.

## Tips for Power9 Architecture
Because we are using IBM Power9, a few additional considerations matter when working with Docker Hub. The first point is image compatibility: not all images available on Docker Hub support ppc64le. Images built only for x86_64 will fail on Power9, so always verify that the desired image has the ppc64le tag before using it.

To validate that Docker is running correctly and recognizing the machine architecture, use:

```bash
docker version --format '{{.Server.Arch}}'
```

The expected output is ppc64le.

## Final Considerations
Installing Docker Engine on AlmaLinux (ppc64le) follows a straightforward path as long as conflicts with Podman and Buildah are resolved beforehand. Official ppc64le support from Docker provides a stable experience on Power9, with the important caveat that image compatibility must always be checked before use.

With Docker installed and configured, the environment is ready to run containers and move on to the next steps in our language model infrastructure.