---
title: "Observability on IBM POWER9 Using Grafana"
date: 2026-09-18 # year-month-day
authors: ["Maria Luísa Gomes"] # Can be a list
tags: ["Grafana", "Observability", "Power9"]
projects: ["multiarq"]
translationKey: "grafana-ppc64le"
summary: "This post describes the process of building Grafana 13.1.0 from source on an IBM Power9 machine with ppc64le architecture"
draft: false # Change to true if you want the post to remain as a draft
---

## Context
Grafana is one of the most widely adopted tools for visualization in observability stacks. However, official binaries for the ppc64le architecture are not provided in the standard releases, and the Grafana images maintained by IBM on Docker Hub are outdated. Therefore, running Grafana in current versions natively on an IBM POWER9 server requires compiling it from source, a process that involves specific build adjustments and dependencies that do not usually appear in builds for more common architectures, such as x86.

This post describes the process of compiling Grafana 13.1.0 from source on an IBM Power9 machine with the ppc64le architecture, generating both frontend and backend artifacts. Based on this process, the result was packaged in a Docker image and published on Docker Hub:

```bash
ufcgibm/grafana-ppc64le:13.1.0-ppc64le
```

This process was automated in a multi-stage Dockerfile, which makes it possible to reproduce the build for future Grafana versions without repeating the manual compilation.

## TL;DR
- We compiled Grafana 13.1.0 natively for ppc64le on an IBM POWER 9 machine;
- The process involved the Go backend, the Node.js frontend, Yarn, Nx, and Webpack, plus a specific dependency adjustment (@swc/core);
- The image is published at [<span class="link-personalizado">ufcgibm/grafana-ppc64le</span>](https://hub.docker.com/r/ufcgibm/grafana-ppc64le);
- We transformed the manual process into a reproducible multi-stage Dockerfile, which means updating to a new Grafana version is now a matter of running a docker build --build-arg, not a manual recompilation from scratch.

## Environment Used
*Hardware*:
- Architecture: ppc64le;
- Processor: IBM POWER9
- RAM: ~128 GB;
- 16 CPUs available
- Operating System: AlmaLinux 8.10

## How Grafana fits into the observability stack

Grafana does not collect metrics by itself; it is the visualization layer of a larger stack. Understanding this pipeline helps explain why a standalone Grafana build only solves part of the observability stack on ppc64le.

In the tested environment, the flow works like this:
- **node_exporter** runs on the POWER9 host and exposes system metrics (CPU, memory, disk, and network usage) through an HTTP endpoint (`/metrics`), in the format Prometheus understands;
- **Prometheus** periodically scrapes this endpoint, i.e., it actively queries `/metrics` at configurable intervals and stores the collected values as time series in its own TSDB database, keeping the history over time;
- **Grafana** connects to Prometheus as a data source and queries this data using `PromQL`, the Prometheus query language, to build the dashboards and charts the user actually sees.

{{< figure src="/images/GrafanaStack.png" alt="Figure 1" caption="Observability stack on Power9">}}

An important point of this architecture is that, unlike Grafana, both Prometheus and node_exporter already provide official binaries and Docker images for the ppc64le architecture. That is why the effort of compiling from source described in this post was specific to Grafana. Even so, the three components run independently, each in its own container, following the standard operating model for this type of stack.


## Dependencies
To perform the build, the following versions were used:


| Dependency | Version |
| :---- | :---- |
| Grafana | 13.1.0 |
| GCC  | 11.2.1 |
| Go | 1.26.5 |
| Node.js | 22.22.2 |
| npm | 10.9.7 |
| Yarn | 4.15.0 |
| Python | 3.11 |
| @swc/core | 1.15.40 |


Grafana 13 requires Node.js 22 or higher, and the project specifies Yarn as its package manager. The only necessary adjustment was in the `@swc/core` dependency, which is used in the frontend build: the version pinned by Grafana does not have a prebuilt binary for `linux-ppc64le`, and updating it to a version that includes this binary resolved the issue.

With this, the build generates a native `bin/grafana` binary, confirmed as a ppc64le executable in version 13.1.0. The complete step-by-step guide is documented in the repository [<span class="link-personalizado">grafana-ppc64le</span>](https://github.com/llm-pt-ibm/grafana-ppc64le/blob/main/TUTORIAL.md) on GitHub.

## Using the Docker image

For those who do not want to perform the compilation themselves, we provide an image on Docker Hub as [<span class="link-personalizado">ufcgibm/grafana-ppc64le</span>](https://hub.docker.com/r/ufcgibm/grafana-ppc64le).

Pull the image:

```bash
docker pull ufcgibm/grafana-ppc64le:13.1.0-ppc64le
``` 
Run it:
```bash
docker run -d \
    --name grafana-ppc64le \
    -p 3000:3000 \
    ufcgibm/grafana-ppc64le:13.1.0-ppc64le
``` 

Check the container:
```bash
docker ps
``` 

Grafana will be available on port 3000.

## Building new versions
The tutorial described above was made for Grafana version 13.1.0. However, every time a new Grafana version is released, it is necessary to repeat this manual build process. The solution was to turn the entire process into a multi-stage Dockerfile that works as follows: we have a first stage (builder) that installs all dependencies, clones Grafana at the desired tag, applies the @swc/core adjustment, and runs `make deps && make build`, all within the Docker build itself; then a second stage copies only the already compiled binaries and assets from the first stage, generating the final runtime image.

Now, to update to a new Grafana version, it is enough to change the arguments:

```bash
docker build \
  --build-arg GRAFANA_VERSION=v13.2.0 \
  --build-arg SWC_CORE_VERSION=1.15.40 \
  -t ufcgibm/grafana-ppc64le:13.2.0-ppc64le .
``` 

The `SWC_CORE_VERSION` is kept separate because it may need to change again in the future. If that happens, we can check the [<span class="link-personalizado">@swc/core</span>](https://github.com/swc-project/swc/releases) changelog on GitHub. The Docker image mentioned above was built from this Dockerfile and validated on an IBM Power9 machine.

## Resources

* Repository on [<span class="link-personalizado">GitHub</span>](https://github.com/llm-pt-ibm/grafana-ppc64le) (Dockerfile, README, compatibility matrix by version);
* Image on Docker Hub: [<span class="link-personalizado">ufcgibm/grafana-ppc64le</span>](https://hub.docker.com/r/ufcgibm/grafana-ppc64le);
* Official Grafana repository: [<span class="link-personalizado">Grafana</span>](http://github.com/grafana/grafana);

## Disclaimer

This work was developed by the Federal University of Campina Grande (UFCG), as part of a Research, Development, and Innovation project carried out in partnership with IBM and Flex Brazil. The image and the adaptations presented in this tutorial are not products or official distributions of IBM or Grafana Labs.
