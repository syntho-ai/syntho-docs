# Deployment instructions for Syntho Application

## Requirements

To install the Syntho Application, the following requirements need to be met:

- Have a running Kubernetes cluster available
  - Self managed, Azure Kubernetes Services (AKS), Amazon Elastic Kubernetes Service (EKS), or other Kubernetes (managed) solutions running Kubernetes 1.20 or higher.
  - The instances should preferably have SSD storage.
- `kubectl` installed.
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

The Syntho Application Helm chart can be requested from the Syntho Support Team. This chart can be used deploy the Syntho Application. Please also request access to the Docker images necessary for this deployment. These images will have all the necessary software installed to run the Syntho application correctly. We will set the credentials for pulling them in Kubernetes using `ImagePullSecrets` later.

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
- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application.

## Deployment using Helm

We will deploy the application in a dedicated namespace in Kubernetes, which we call `syntho` for now. If the namespace does not exist, create it by running:

```[bash]
kubectl create namespace syntho
```

The remaining sections will be focused on configuration the Helm chart for your environment.

### Setting up a Kubernetes Secret

Depending on the received credentials from Syntho, a Kubernetes `Secret` should be created to use to pull the latest image from our docker registry. Please read more about creating `Secrets` [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

We will assume that a secret named `syntho-cr-secret` has been created at this point. Please contact the Syntho Support Team for your credentials.

In the Helm chart, we can set the secret under the `imagePullSecrets` section.

```[bash]
imagePullSecrets: 
  - name: syntho-cr-secret
```

### Configuring the UI

For the UI, we will need to set the image repository and tag first:

```[yaml]
image:
  repository: synthoregistry.azurecr.io/syntho-core-frontend
  tag: latest
```

We will also need to set the domain name for the UI.

```[yaml]
frontend_url: <hostname-or-ip>
frontend_protocol: https # http or https depending on the availability of an SSL certificate
```

If a DNS hostname is available, we can set the ingress configuration as follows for the UI:

```[yaml]
ingress:
    enabled: true
    name: frontend-ingress
    className: nginx  # Set to classname of the ingress controller you are using
    annotations: {
      cert-manager.io/cluster-issuer: "", # In case cert-manager is used for SSL
      nginx.ingress.kubernetes.io/proxy-buffer-size: "32k",
      nginx.ingress.kubernetes.io/affinity: "cookie",
      nginx.ingress.kubernetes.io/rewrite-target: /,
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "600",
      nginx.ingress.kubernetes.io/proxy-read-timeout: "600",
      nginx.ingress.kubernetes.io/proxy-send-timeout: "600",
      nginx.ingress.kubernetes.io/proxy-body-size: "512m",
    }
    hosts:
      - host: <hostname>
        paths:
          - path: /
            pathType: Prefix
    
    tls:  # In case SSL is not used, the tls section can be removed
      - hosts:
        - <hostname>
        secretName: frontend-tls
```

If no ingress is needed, it can be disabled by setting:

```[yaml]
ingress:
    enabled: false
```

### Configuring the Backend

The backend is responsible for user management and workspace management. We need to set a few variables correctly. To start off, we need to set the image:

```[yaml]
backend:
  image:
    repository: synthoregistry.azurecr.io/syntho-core-backend
    tag: latest
```

We then need to set the database credentials and Redis credentials. In the case that the instances defined in the Helm chart themselves are being used, no changes are needed there. Otherwise the following need to be changed:

```[yaml]
backend:
  database:
    host: <hostname>
    port: <port>
    username: <username>
    password: <password>
    database: <database>
  redis:
    host: <hostname>
    port: <port>
    db: <db_index>
```

If a hostname is available, we recommend setting the ingress for it as well. The process here is similar to setting it for the UI. See:

```[yaml]
backend:
  ingress:
    enabled: true
    name: backend-ingress
    className: nginx  # Set to class name of ingress controller
    annotations: {
      cert-manager.io/cluster-issuer: "",  # Set to issuer defined if using cert-maanger for SSL
      nginx.ingress.kubernetes.io/proxy-buffer-size: "32k",
      nginx.ingress.kubernetes.io/affinity: "cookie",
      nginx.ingress.kubernetes.io/rewrite-target: /,
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "600",
      nginx.ingress.kubernetes.io/proxy-read-timeout: "600",
      nginx.ingress.kubernetes.io/proxy-send-timeout: "600",
      nginx.ingress.kubernetes.io/proxy-body-size: "512m"
    }
    hosts:
      - host: <hostname>
        paths:
          - path: /
            pathType: Prefix
    tls:
      - hosts:
        - <hostname>
        secretName: backend-tls
```

Lastly we need to set an additional variable, as defined in the block below:

```[yaml]
secret_key: (^ky&f)l&$3sqf2tctv-(pgzvh!+9$j%b5xe2y@&%p2ay*h$$a  # Random string to use as a secret key
```

### Configuring the Core API

To configure the Core API, we will need to first set the correct image. To set the image we will use the `image` field in the `core` section.

```[bash]
image:
  repository: synthoregistry.azurecr.io/syntho-core-api
  tag: latest
```

Furthermore, we need to set database hostname and credentials:

```[bash]
db:
    username: <database-username>
    password: <database-password>
    name: <database-name>
    host: <database-host>
```

The deployment can possibly create the database already, in which case the credentials do not need to be set.

Lastly we need to set a secret key for encryption purposes, the credentials for a Redis instance and the Ray head IP or hostname to connect to. We will acquire the hostname later in the process by going through the steps of the section [Deployment of Ray using Helm](#deployment-of-ray-using-helm). 

```[sh]
secret_key: UNIbrRR0CnhPEB0BXKQSDASaNzT1IYgQWWaLyQ1W1iPg= # Fernet Key
redis_host: redis://<redis-hostname-or-ip>:<port>/<redis_db_index>
ray_address: <ray-head-ip-or-hostname>
```

## Deployment of Ray using Helm

To power the ML models, we will need to deploy a Ray cluster using Helm for the Core API to connect to. The chart to deploy Ray will be provided by the Syntho team.

### Setting the image

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

### Workers and nodes

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

If autoscaling is enabled in Kubernetes, new nodes will be created once the Ray requirements are higher than the available resources. Please discuss with together with the Syntho Team which situation would fit your data requirements.

### Deploy using Helm - Ray

Once the values have been set correctly in `values.yaml` under `helm/ray`, we can deploy the application to the cluster using the following command:

```[sh]
helm upgrade --cleanup-on-fail ray-cluster ./helm/ray --values values.yaml --namespace syntho 
```

Once deployed, we can find the service name in Kubernetes for the Ray application. In the case of using the name `ray-cluster` as is the case in the command above, the service name (and hostname to use in the variable `ray_address` for the Core API values section) is `ray-cluster-ray-head`.

## Testing the deployment

Once both Helm charts are deployed, the application should be reachable on the defined url of the frontend. To test this, we can simply open a browser and navigate to the url.