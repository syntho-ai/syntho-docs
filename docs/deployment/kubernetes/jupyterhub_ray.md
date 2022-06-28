# Deployment instructions for JupyterHub/Ray Syntho

## Requirements

To install the Syntho Application together with JupyterHub & Ray, the following requirements need to be met:

- A running Kubernetes cluster available
  - Self managed, Azure Kubernetes Services (AKS), Amazon Elastic Kubernetes Service (EKS), or other Kubernetes (managed) solutions running Kubernetes 1.20 or higher.
- `kubectl` installed.
  - For managing the Kubernetes cluster
  - The instances should preferably have SSD storage.
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
│       │   │   └── cluster_crd.yaml
│       │   ├── templates
│       │   │   ├── _helpers.tpl
│       │   │   ├── operator_cluster_scoped.yaml
│       │   │   ├── operator_namespaced.yaml
│       │   │   └── raycluster.yaml
│       │   └── values.yaml
│       └── values.yaml
```

Please also request access to the Docker images for JupyterHub & Ray. These images will have all the necessary software installed to run the Syntho application correctly. The credentials are set in Kubernetes using `ImagePullSecrets` later.

The images necessary for this deployment:

- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application.
- syntho-jupyterhub
  - Version: latest
  - Has the latest version of JupyterHub installed that is compatible with the Syntho Application.

## Deployment using Helm

The application is deployed with JupyterHub and Ray. Both applications are deployed in the same namespace, called `syntho` for now. Please see the section [JupyterHub](#jupyterhub) for the deployment of JupyterHub and the section [Ray](#ray) on the deployment of Ray. Together they form the total application landscape for this deployment method.

If the namespace `syntho` does not exist, create it by running:

```[bash]
kubectl create namespace syntho
```

Please read through the remaining sections to configure JupyterHub and Ray correctly for your environment.

### Setting up a Kubernetes Secret

Depending on the received credentials from Syntho, a Kubernetes `Secret` should be created to pull the latest image from our docker registry. Please read more about creating `Secrets` [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

We will assume that a secret named `syntho-cr-secret` has been created at this point. Please contact the Syntho Support Team for your credentials.

### JupyterHub

Under the folder `helm/jupyterhub`, the files for the JupyterHub deployment can be found. We will need to adjust the file `values.yaml` in the upcoming sections. Once we reach the section [Deploy using helm - JupyterHub](#deploy-using-helm---jupyterhub), the `values.yaml` file should be correctly adjusted for your environment and ready to be used by Helm for deploying the application.

#### Image

The image for Syntho should be set under `singleuser.image.name` and `singleuser.image.tag`. It is important to fill the value of the created secret under `singleuser.image.pullSecrets`. An example of this would be:

```[yaml]
singleuser:
  image:
    name: <name-of-registry>/syntho-jupyterhub
    tag: 0.2.13
    pullPolicy:
    pullSecrets: ["<your-secret-name>"]
```

#### Authentication method

Under `hub.config`, we can set the desired authentication method when using JupyterHub. JupyterHub will create a personal space for each user with preexisting files that can be used to interact with the Syntho Application. An overview of all authentication methods can be found [here](https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/authentication.html#configuring-authenticator-classes).

A YAML example using Azure Active Directory:

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

Depending on the requirement for accessing the application, we can either select a `Loadbalancer` in Kubernetes to create a separate `Loadbalancer` that can be used for accessing the application. If that is the case, the following values should be set like this:

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

The Ingress can then be enabled by adjusting the values for `ingress` as follows:

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
  # Uncomment this part if cert-manager is used
  #tls: [
  #  {
  #    "hosts": [
  #      "<host-name>"
  #    ],
  #    "secretName": "k8s-jupyterhub-ingress-tls-secret"
  #  }
  #]
```

#### Deploy using Helm - JupyterHub

After the values.yaml have been set correctly for JupyterHub, the application can be deployed using helm with the following commands:

```[sh]
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update

helm upgrade --cleanup-on-fail \
  --install jupyterhub-syntho jupyterhub/jupyterhub \
  --namespace syntho \
  --values values.yaml
```

If any issues arise during this step, please contact the Syntho Support Team.

#### Upgrading JupyterHyb Syntho

If updates are available, please adjust the tag under `singleuser.image.tag` to the latest version. Once that is done, the application can be updated using the following command:

```[sh]
helm upgrade --cleanup-on-fail \
  jupyterhub-syntho jupyterhub/jupyterhub \
  --namespace syntho \
  --values values.yaml
```

### Ray

Next we will need to deploy the Ray cluster to be used for scaling the Syntho Application over multiple nodes/workers. For this we will need access to the image `syntho-ray`. A pre-configured values.yaml can be found under `helm/ray/values.yaml`. Once configured, we can deploy using Helm by following the instructions under [Deploy using Helm - Ray](#deploy-using-helm---ray).

#### Setting the image

In the values.yaml file in `helm/ray`, set the following fields to ensure the usage of the correct Docker image:

```[yaml]
operatorImage: <name-of-registry>/syntho-ray:<image-tag>
image: <name-of-registry>/syntho-ray:<image-tag>
```

`<name-of-registry>` and `<image-tag>` will be provided by Syntho for your deployment.

Next to setting the correct Docker image, define the Kubernetes `Secret` that is created under `imagePullSecrets`:

```[yaml]
imagePullSecrets: 
    - name: syntho-cr-secret
```

#### Workers and nodes

Depending on the size and amount of nodes of the cluster, adjust the amount of workers that Ray has available for tasks. Under `podTypes.rayHeadType` we can set the resources for the head node, which we recommend to keep as is in the provided file. This head node will mostly be used for administrative tasks in Ray and the worker nodes will be picking up most of the tasks for the Syntho Application.

We recommend two pools of workers, where the first pool has a higher amount of memory, but a low amount of workers and the second pool with reverse conditions. Depending on the CPUs and Memory available in the node, the amount of CPUs and Memory can be set. An example of a cluster with two node pools, of 1 machine (autoscaling up to 3), with 16 CPUs and 64GB of RAM and another of 1 machine (autoscaling up to 3) with 8 CPUs and 32GB of RAM:

```[yaml]
rayWorkerType:
    # minWorkers is the minimum number of Ray workers of this pod type to keep running.
    minWorkers: 1
    # maxWorkers is the maximum number of Ray workers of this pod type to which Ray will scale.
    maxWorkers: 3
    memory: 50Gi
    CPU: 5
    GPU: 0

rayWorkerType2:
    minWorkers: 3
    maxWorkers: 5
    memory: 8Gi
    CPU: 2
    GPU: 0
```

If autoscaling is enabled in Kubernetes, new nodes will be created once the Ray requirements are higher than the available resources. Please discuss with together with the Syntho Support Team which situation would fit your data requirements.

#### Deploy using Helm - Ray

Once the values have been set correctly in `values.yaml` under `helm/ray`, we can deploy the application to the cluster using the following command:

```[sh]
helm upgrade --cleanup-on-fail ray-cluster ./helm/ray --values values.yaml --namespace syntho 
```

## Testing the application

Once both Ray and JupyterHub have been installed, login into the application in `http://<ip-address-or-dns>/hub`. Once an environment has been created, a simple test to check whether Ray installed correctly and is accessible for JupyterHub, is to run the following command in a `Python Notebook`:

```[python]
import ray

ray.init("ray://ray-cluster-ray-head:10001")
```

This will connect to the Ray cluster and will return something like this:

```[sh]
ClientContext(dashboard_url='10.244.1.21:8265', python_version='3.9.5', ray_version='1.12.1', ray_commit='4863e33856b54ccf8add5cbe75e41558850a1b75', protocol_version='2022-03-16', _num_clients=1, _context_to_restore=<ray.util.client._ClientContext object at 0x7f6c24257b50>)
```
