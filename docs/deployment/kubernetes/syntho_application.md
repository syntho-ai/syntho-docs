# Deployment instructions for Syntho Application

## Requirements

To install the Syntho Application, the following requirements need to be met:

- Have a running Kubernetes cluster available/
  - Self managed, Azure Kubernetes Services (AKS), Amazon Elastic Kubernetes Service (EKS), or other Kubernetes (managed) solutions running Kubernetes 1.20 or higher.
  - The instances should preferably have SSD storage.
- kubectl installed.
  - For managing the Kubernetes cluster
- Helm v3 installed.
  - See instructions on how to install Helm [here](https://helm.sh/docs/intro/install/).
- Postgres database
  - Either by including a Postgres database in the deployment or have an external database available. Two different databases need to be created in this instances.
- Redis instance
  - The redis instances can be included when deploying using Helm. If this is disabled, a Redis instance needs to be created for the Syntho Application to connect to.
- [Optional] DNS zone and DNS record for UI.
  - Example: syntho.company.com be used for hosting the UI.

## Preparations

The Syntho Application Helm chart can be requested from the Syntho Support. This chart can be used deploy the Syntho Application. Please also request access to the Docker images necessary for this deployment. These images will have all the necessary software installed to run the Syntho application correctly. We will set the credentials for pulling them in Kubernetes using `ImagePullSecrets` later.

The images necessary for this deployment:

- syntho-core-api
  - Version: latest
  - The Syntho Core API is responsible for the core operations of the Syntho Platform.
- syntho-frontend
  - Version: latest
  - The Syntho UI is a container that contains the web UI fro the Syntho Platform.
- syntho-backend
  - Version: latest
  - The Syntho Backend is responsible for user management and workspace management.

## Deployment using Helm

We will deploy the application in a dedicated namespace in Kubernetes, which we call `syntho` for now. If the namespace does not exist, create it by running:

```[bash]
kubectl create namespace syntho
```

The remaining sections will be focused on configuration the Helm chart for your environment.

### Setting up a Kubernetes Secret

Depending on the received credentials from Syntho, a Kubernetes `Secret` should be created to use to pull the latest image from our docker registry. Please read more about creating `Secrets` [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

We will assume that a secret named `syntho-cr-secret` has been created at this point. Please contact the Syntho Support for your credentials.

### Configuring the UI

### Configuring the Backend

### Configuring the Core API
