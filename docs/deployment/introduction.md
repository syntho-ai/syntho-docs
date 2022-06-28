# Introduction to the deployment documentation

<div id="top"></div>

<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/syntho-ai/syntho-docs/tree/master/docs/deployment">
    <img src="https://www.syntho.ai/wp-content/uploads/2021/03/cropped-Syntho_logo_wide.png" alt="Logo" height="80">
  </a>

<h3 align="center">Syntho Deployment Documentation</h3>

  <p align="center">
    General repository for saving Syntho documentation in Markdown
    <br />
    <a href="https://github.com/syntho-ai/syntho-docs/tree/master/docs/deployment/docker"><strong>Explore the docker-compose deployment documentation »</strong></a>
    <br />
    <a href="https://github.com/syntho-ai/syntho-docs/tree/master/docs/deployment/kubernetes"><strong>Explore the Kubernetes deployment documentation »</strong></a>
    <br />
    <br />
  </p>
</div>

## Introduction

Welcome to the deployment manual for the Syntho Application. These markdown files will contain the instructions to deploy the Syntho Application using either docker-compose or Kubernetes.

## Accessing Docker images

### Using an internet connection

When connected to the internet, the docker images can be accessed using our private container registry. This registry will have the latest versions and previous versions available. Access to this registry is possible using a token provided by Syntho.

An example of using the Docker CLI with the provided token:

```[sh]
TOKEN_NAME=SynthoToken
TOKEN_PWD=<provided-token>

echo $TOKEN_PWD | docker login --username $TOKEN_NAME --password-stdin synthoregistry.azurecr.io
```

Depending on which deployment form is chosen, different images need to be installed. Each deployment will outline the exact images necessary and can be pulled from the instance or cluster once the Docker credentials have been provided in the appropriate matter.

### Without an internet connection

It is also possible to retrieve the docker images without an internet connection. The docker containers have to be loaded in manually using the Docker CLI.

Given the image in a `.tar` or `.tar.gz` file, we can use the Docker CLI:

```[sh]
docker load < syntho-image.tar.gz
# OR
docker load --input syntho-image.tar.gz
```

In the case of loading the Docker image files, each machine/VM instance needs to have the appropriate files loaded into the Docker environment. Please contact the Syntho Team in case this is the preferred way of retrieving the Docker images to receive the correct files with their corresponding version. Again, theses images might differ depending on the chosen deployment method.

## What's next

Depending on the chosen deployment option, the documentation can be found in the corresponding folder. See the overview of all the methods and their locations below:

- Deploying using Docker
  - [JupyterHub - Ray](/docs/deployment/docker/jupyterhub_ray.md)
  - [Syntho Application with UI](/docs/deployment/docker/)
- Deploying using Kubernetes
  - [JupyterHub - Ray](/docs/deployment/kubernetes/jupyterhub_ray.md)
  - [Syntho Application with UI](/docs/deployment/kubernetes/syntho_application.md)