---
title: "Installing Docker in a ppc64le (Power9) Environment"
date: 2026-04-01
authors: ["Gabrielly Lima"]
tags: ["Docker", "Power9", "ppc64le", "AlmaLinux", "Containers"]
projects: ["multiarq"]
translationKey: "power9-docker-installation"
summary: "In this post, we walk through how to install and configure Docker Engine on IBM Power9 (ppc64le), including removing conflicting packages, validating the installation, and following image compatibility best practices."
draft: false
---

## Context
This post is part of our tutorial series on building a language model infrastructure on an IBM Power9 server. After configuring the operating system and NVIDIA drivers, the next step is installing Docker Engine, an essential tool for packaging and running containerized applications in an isolated and reproducible way.

Docker Engine has official support for Rocky Linux on x86_64, arm64, s390x, and ppc64le architectures, which allows direct use on Power9 without special adaptations. Even so, some care is needed during installation, such as removing tools that conflict with Docker and ensuring selected images are compatible with ppc64le.

## TL;DR
* This post provides a step-by-step guide to installing Docker Engine on Rocky Linux/AlmaLinux with ppc64le architecture.
* You must remove Podman and Buildah before installation, since these packages conflict with Docker.
* Docker Hub images must explicitly support ppc64le to run correctly on Power9.

## Environment Used
* **Architecture**: IBM Power9 server (ppc64le architecture).
* **Operating System (OS)**: AlmaLinux 8.10 binary compatible with Red Hat Enterprise Linux (RHEL) 8.9/8.10.
* **RAM**: 512GB.

## Prerequisites
Before installing Docker, consider an important firewall limitation: when exposing container ports with Docker, those ports bypass default firewalld rules. Verify whether this behavior is acceptable for your environment before proceeding.

It is also important to reinforce that Docker Engine is compatible with Rocky Linux 8 and 9, and with AlmaLinux 8 on ppc64le architecture.

## Installing Docker Engine on Power9
1. **Removing conflicting packages**:
Rocky Linux usually includes Podman and Buildah by default. These packages conflict with Docker Engine and should be removed, along with any older Docker versions that might already exist:

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

2. **Adding the official Docker repository**:
The recommended method is to use the official repository. For RHEL-based distributions such as AlmaLinux, Docker uses the CentOS repository, which is an officially supported flow.

Install the repository management plugin and add the repository:

```
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

3. **Installing Docker packages**:
With the repository configured, install Docker Engine and the build and compose plugins:

```
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

4. **Starting and enabling the service**:
On Rocky Linux/AlmaLinux, the Docker service does not start automatically after installation. Start it manually and enable it at boot:

```
sudo systemctl start docker
sudo systemctl enable docker
```

5. **Verifying the installation**:
To confirm everything was installed correctly, run the test image:

```
sudo docker run hello-world
```

The expected output is a message confirming Docker is working correctly.

## Post-installation Configuration
By default, only root (or users with sudo privileges) can run Docker commands. To use Docker without sudo for every command:

1. **Creating the docker group (if needed)**:

```
sudo groupadd docker
```

2. **Adding your user to the group**:

```
sudo usermod -aG docker $USER
```

You need to log out and back in again for permissions to take effect.

## Tips for Power9 Architecture
On IBM Power9, not all Docker Hub images are compatible with ppc64le. Images published only for x86_64 will fail to run. Always check whether the image explicitly supports ppc64le architecture.

To validate that the Docker daemon is running and correctly recognizing the server architecture, run:

```
docker version --format '{{.Server.Arch}}'
```

The expected output is:

```
ppc64le
```

## Final Considerations
Installing Docker Engine on Rocky Linux/AlmaLinux (ppc64le) follows a straightforward flow, as long as conflicts with Podman and Buildah are resolved before installation.

With official support for ppc64le architecture, Docker provides a stable foundation for running containers on Power9. The main ongoing concern is selecting images that are compatible with this architecture.

With Docker installed and configured, the environment is ready to move forward to the next steps in the language model infrastructure.
