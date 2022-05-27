# Deployment instructions for JupyterHub/Ray Syntho

## Requirements

To install the Syntho Application together with JupyterHub & Ray, the following requirements need to be met:

- Have a running Kubernetes cluster available/
  - Self managed, Azure Kubernetes Services (AKS), Amazon Elastic Kubernetes Service (EKS), or other Kubernetes (managed) solutions running Kubernetes 1.20 or higher.
- kubectl installed.
  - For managing the Kubernetes cluster
- Helm v3 installed.
  - See instructions on how to install Helm [here](https://helm.sh/docs/intro/install/).
- [Optional] DNS zone and record for JupyterHub.
  - Example: syntho.company.com be used for hosting the interface of JupyterHub.

## Preparations

To prepare this deployment, please download the configuration files provided by Syntho for this particular deployment setup. The files will contain the necessary Helm charts and a pre-configured value.yaml files to be used together with these deployment.

The structure of these files will look as follows:

```[sh]
.
├── README.md
├── helm
│   ├── jupyterhub
│   │   ├── README.md
│   │   └── values.yaml
│   └── ray
│       ├── README.md
│       ├── chart
│       │   ├── Chart.yaml
│       │   ├── README.md
│       │   ├── crds
│       │   │   ├── README.md
│       │   │   └── cluster_crd.yaml
│       │   ├── templates
│       │   │   ├── README.md
│       │   │   ├── _helpers.tpl
│       │   │   ├── operator_cluster_scoped.yaml
│       │   │   ├── operator_namespaced.yaml
│       │   │   └── raycluster.yaml
│       │   └── values.yaml
│       └── values.yaml
```

Please also request access to the Docker images for JupyterHub & Ray. These images will have all the necessary software installed to run the Syntho application correctly. We will set the credentials for them in Kubernetes using `ImagePullSecrets` later.

The images necessary for this deployment:

- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application.
- syntho-jupyterhub
  - Version: latest
  - Has the latest version of JupyterHub installed that is compatible with the Syntho Application.

## Deployment using Helm

We will be deploying the application with JupyterHub and Ray. We will deploy both applications in the same namespace, which we call `syntho` for now. Please see the section [JupyterHub](#jupyterhub) for the deployment of JupyterHub and the section 

If the namespace does not exist, create it by running:

```[bash]
kubectl create namespace syntho
```

### Setting up PullSecret

Depending on the received credentials from Syntho, a Kubernetes `Secret` should be created to use to pull the latest image from our docker registry. Please read more about creating `Secrets` [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

We will assume that a secret named `syntho-cr-secret` has been created at this point. Please contact the Syntho Support for your credentials.

### JupyterHub

Under the folder `helm/jupyterhub`, the files for the JupyterHub deployment can be found. We will need to adjust the file `values.yaml` in the upcoming sections. Once we reach the section [Deploy using helm](#deploy-using-helm), the `values.yaml` file should be correctly adjusted for your environment and ready to be used by Helm for deploying the application.

#### Image

The image for Syntho should be set under `singleuser.image.name` and `singleuser.image.tag`. It is also important is to fill the value of the created secret under `singleuser.image.pullSecrets`. An example of this would be:

```[yaml]
singleuser:
  image:
    name: synthoregistry.azurecr.io/syntho-jupyterhub
    tag: 0.2.13
    pullPolicy:
    pullSecrets: ["<your-secret-name>"]
```

#### Authentication method

Under `hub.config`, we can set the desired authentication method when using JupyterHub. JupyterHub will create a personal space for each user with preexisting files that can be used to interact with the Syntho Application. An overview of all authentication methods can be found [here](https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/authentication.html#configuring-authenticator-classes).

An YAML example using Azure Active Directory:

```[yaml]
hub:
  config:
    AzureAdOAuthenticator:
      client_id: your-client-id
      client_secret: your-client-secret
      oauth_callback_url: https://your-jupyterhub-domain/hub/oauth_callback
      tenant_id: your-tenant-id
    JupyterHub:
      authenticator_class: azuread
```

#### Application access

Depending on the requirement for accessing the application, we can either select a `Loadbalancer` in Kubernetes to create a seperate `Loadbalancer` that can be used for accessing the application. If that is the case, the following values should be set like this:

```[yaml]
proxy:
  service:
    type: LoadBalancer
    labels: {}
    annotations: {}
    nodePorts:
      http:
      https:
    disableHttpPort: false
    extraPorts: []
    loadBalancerIP:
    loadBalancerSourceRanges: []
```

To use an `Ingress` of any kind in Kubernetes to be able to assign a DNS record, please configure your Ingress Controller (see overview of Ingress Controllers [here](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)). We recommend the NGINX controller, which can additionally be installed using Helm (see [this link](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)).

In the case of using an ingress controller, please adjust the values of `proxy.service.type` as follows:

```[yaml]
proxy:
  service:
    type: ClusterIP
    labels: {}
    annotations: {}
    nodePorts:
      http:
      https:
    disableHttpPort: false
    extraPorts: []
    loadBalancerIP:
    loadBalancerSourceRanges: []
```

The Ingress can then be enabled by adjust the values for `ingress` as follows:

```[yaml]
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    # If cert-manager is used for HTTPS certs, enable this line
    # cert-manager.io/cluster-issuer: "<issuer-name>"
  hosts: ["<host-name>"]
  pathSuffix:
  pathType: Prefix
  tls: [
    {
      "hosts": [
        "<host-name>"
      ],
      "secretName": "k8s-jupyterhub-ingress-tls-secret"
    }
  ]
```

#### Deploy using Helm

After the values.yaml have been set correctly for JupyterHub, we can deploy the application using helm with the following commands:

```[sh]
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update

helm upgrade --cleanup-on-fail \
  --install jupyterhub-syntho jupyterhub/jupyterhub \
  --namespace syntho \
  --values values.yaml
```

If any issues arise during this step, please contact the Syntho Support.

#### Upgrading JupyterHyb Syntho

If updates are available, please adjust the tag under `singleuser.image.tag` to the latest version. Once that is done, the application can be updated using the following command:

```[sh]
helm upgrade --cleanup-on-fail \
  jupyterhub-syntho jupyterhub/jupyterhub \
  --namespace syntho \
  --values values.yaml
```

### Ray