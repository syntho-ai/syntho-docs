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

The Syntho Application Helm chart can be found here: [Syntho Charts](https://github.com/syntho-ai/syntho-charts/tree/master/helm). This chart can be used deploy the Syntho Application. For this deployment, we will need the following charts in the folder `helm`:

- syntho-ui
  - Helm chart containing the web UI application and necessary API's
- ray
  - Helm chart for deploying the cluster to be used for parallelizing ML and heavy workloads

Please request access to the Docker images necessary for this deployment. These images will have all the necessary software installed to run the Syntho application correctly. We will set the credentials for pulling them in Kubernetes using `ImagePullSecrets` later.

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

We will assume that a secret named `syntho-cr-secret` has been created at this point. Please contact the Syntho Support Team for your credentials. An example of creating a secret for a docker registry via `kubectl` can be found below:

```[bash]
kubectl create secret docker-registry syntho-cr-secret --namespace syntho --docker-server=<registry> --docker-username=<username> --docker-password=<password>
```

In both the Helm charts for Ray and the Syntho application, we can set the secret under the `imagePullSecrets` section.

```[bash]
imagePullSecrets: 
  - name: syntho-cr-secret
```

## Deployment of Ray using Helm

To power the ML models, we will need to deploy a Ray cluster using Helm for the Core API to connect to. The chart can be found in the repository [here](https://github.com/syntho-ai/syntho-charts/tree/master/helm) or can be supplied by a repo url to be used in Helm directly. Please contact the Syntho team for this repo url.

This part of the documentation will assume access to the folder `helm/ray` in the master branch of the aforementioned github repository.

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

### License key - Ray

The license key can be set under `SynthoLicense` in the `values.yaml` file.. An example of this would be:

```[yaml]
SynthoLicense: <syntho-license-key>
```

Please use the license key provided by Syntho.

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
helm install ray-cluster ./helm/ray --values values.yaml --namespace syntho 
```

Once deployed, we can find the service name in Kubernetes for the Ray application. In the case of using the name `ray-cluster` as is the case in the command above, the service name (and hostname to use in the variable `ray_address` for the Core API values section) is `ray-cluster-ray-head`.

## Deployment of Syntho Application using Helm

To deploy the Syntho Application, we will use the Helm chart in the same repository as mentioned before `syntho-ai/syntho-charts`. The chart can be found in the folder `helm/syntho-ui`. The rest of this paragraph will assume access to the folder `helm/syntho-ui` in the master branch of the aforementioned github repository.

To configure the UI, the `values.yaml` file in `helm/syntho-ui` can be used. The following sections will describe the different fields that can be set.

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
    className: nginx  # Set to class name of the ingress controller you are using
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

If an ingress is not necessary, it can be disabled by setting `ingress.enabled` to false:

```[yaml]
frontend:
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
  db:
    host: <hostname>
    port: <port>
    user: <username>
    password: <password>
    name: <database>
  redis:
    host: redis-svc
    port: 6379
    db: 0
```

The redis section can be set as is defined above, if the redis instance is being used from the Helm chart. The default behavior will deploy the redis instance defined in the chart. If a different redis instance is being used, the host and port need to be changed.

The database section needs to be changed if a different database is being used. The default behavior will deploy the database instance defined in the chart. If a different database outside the Helm chart is being used, the host, port, user, password and database name need to be changed. To disable the usage and deployment of the database instance defined in the chart, the following can be set:

```[yaml]
backend:
  database_enabled: false
```

If the database is being used from the Helm chart, the value `host` can be set to `database` and port to `5432`. The other values can be changed in case a different username, password or database name is preferred. This will automatically adjust the database instance defined in the Helm chart.

If a hostname is available, we recommend setting the ingress for it as well. The process here is similar to setting it for the UI. See:

```[yaml]
backend:
  ingress:
    enabled: true
    name: backend-ingress
    className: nginx  # Set to class name of ingress controller
    annotations: {
      cert-manager.io/cluster-issuer: "",  # Set to issuer defined if using cert-manager for SSL
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

Setting up the credentials for the first administrative user is also necessary. We can define this user in the following way:

```[yaml]
backend:
  user:
    username: admin
    password: password
    email: admin@company.com
```

This user can be used to login into the UI and create other users.

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
core:
  db:
    username: <database-username>
    password: <database-password>
    name: <database-name>
    host: <database-host>
    port: <database-port>
```

The deployment can possibly create the database instance itself. In that case, the `database_enabled` field should be set to `true`:

```[bash]
core:
  database_enabled: true
```

This will create a database with the specified username, password and database name. The host for this database is `backend` and the port will be `5432` as this is a Postgres database.

Lastly we need to set a secret key for encryption purposes, the credentials for a Redis instance and the Ray head IP or hostname to connect to. The Ray head hostname is the one mentioned in the section [Deploy using Helm - Ray](#deploy-using-helm---ray).

```[sh]
secret_key: UNIbrRR0CnhPEB0BXKQSDASaNzT1IYgQWWaLyQ1W1iPg= # Fernet Key: generate by running  python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
redis_host: redis://<redis-hostname-or-ip>:<port>/<redis_db_index> # Set to redis://redis-svc:6379/1 if using the redis instance created by the chart
ray_address: <ray-head-ip-or-hostname>
```

The fernet key can be generated using the `cryptography` library in Python. Running the following command will result in a randomly generated fernet key in your CLI:

```[sh]
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

The redis instance can be set to `redis://redis-svc:6379/1` if the redis instance created by the Helm chart is being used, which is being deployed by default.

### Deploy using Helm - Syntho Application

To deploy the Syntho Application, we will use the Helm chart provided by the Syntho team. The chart can be found in the `helm/syntho` folder.

```[sh]
helm install syntho-ui ./helm/syntho-ui --values values.yaml --namespace syntho 
```

## Testing the deployment

Once both Helm charts are deployed, the application should be reachable on the defined url of the frontend. To test this, we can simply open a browser and navigate to the url of the frontend `frontend_url`. If the deployment was successful, we should see the login page of the Syntho application and should be able to login with the credentials of the admin user we defined in the `values.yaml` file.

## Upgrading the applications
