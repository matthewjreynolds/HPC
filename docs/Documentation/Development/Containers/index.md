---
layout: default
title: Containers Intro
parent: Containers
order: 1
---

# Introduction to containers

## What are containers?
Containers provide a method of packaging your code so that it can be run anywhere you have a container runtime. This enables you to create a container on your local laptop and then run it on Eagle or other computing resources. Containers provide an alternative way of isolating and packaging your code from solutions such as Conda environments. 

## Docker vs. Singularity
The most common container runtime outside of HPC is Docker. Docker is not suited for the HPC environment on Eagle and is therefore not available on the system currently. Singularity is an alternative container tool which is provided. 

## Compatibility 
Singularity is able to run most Docker images, but Docker is unable to run Singularity images. A key consideration when deciding to containerize an application is which container engine to build with. A suggested best practice is to build images with Docker when possible, as this provides more flexibility. Sometimes this is not possible though, and you may have to build with Singularity or maintain separate builds for each container engine. 

## Container advantages
* **Portability**: containers can be run on HPC, locally, and on cloud infrastructure used at NREL. 
* **Reproducibility**: Containers are one option to ensure reproducible research by packaging all necessary software to reproduce an analysis. Containers are also easily versioned using a hash.
* **Workflow integration**: Workflow management systems such as Airflow, Nextflow, Luigi, and others provide built in integration with container engines. 

## HPC hardware
Both Singularity and Docker provide the ability to use hardware based features of Eagle. A common usage for containers is packaging of GPU enabled tools such as TensorFlow. Singularity enables access to the GPU and driver on the host. Likewise the MPI installations available on Eagle can be accessed from correctly configured containers. 

## Building
Containers are built from a container specification file, Dockerfiles for Docker or Singularity Definition File in Singularity. These files specify the steps necessary to create the desired package and the additional software packages to install and configure in this environment. 
```
FROM ubuntu:20.04

RUN apt-get -y update && apt-get install -y python3 
```

The above Dockerfile illustrates the build steps to create a simple image. Images are normally built from a base image indicated by `FROM`, in this case Ubuntu. The ability to use a different base image provides a way to use packages which may work more easily on one Linux Distribution. For example the Linux distribution on Eagle is CentOS, building the above image would allow the user to install packages from Ubuntu repositories. 

The `RUN` portion of the above Dockerfile indicates the command to run, in this example it installs the Python 3 package. Additional commands such as `COPY`, `ENV`, and others enable the customization of your image to suit your compute environment requirements. 