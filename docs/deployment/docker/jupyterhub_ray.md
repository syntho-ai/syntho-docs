# Deployment instructions for JupyterHub/Ray Syntho with Docker

## Introduction

Deploying the Syntho Application with JupyterHub/Ray solution using Docker can be done a few ways:

- Option 1: Depending on whether support for for autoscaling Ray workers is necessary, we can either deploy by using the Ray cluster manager directly in the following cloud providers: AWS, GCP, Azure.
The Ray cluster manager will enable autoscaling based on a given configuration. We will then separately deploy a single instance running JupyterHub using `docker-compose` in the same network as the Ray cluster. See section [Deployment using Ray cluster manager](#deployment-using-ray-cluster-manager-option-1) (Recommended)

- Option 2: Deploying Ray instances manually using Docker. Autoscaling using Ray will not be available and the nodes need to be connected manually to the head node of Ray. JupyterHub will still be deployed using `docker-compose`. See section [Deployment]

## Requirements

To install the Syntho Application together with JupyterHub & Ray, the following requirements need to be met:

- Have at least 3 VM instances running.
  - OS: Ubuntu 18.04 or higher.
  - Docker 1.13.0+ ​installed.
  - `docker-compose` 2.x.x  installed.
  - `docker-compose` file version will be v3.
- One available IP address for the JupyterHub instance.
- Internal networking between 3 VM instances.
- [Optional] DNS zone and record for JupyterHub.
  - Example: syntho.company.com be used for hosting the interface of JupyterHub.

## Preparations

To prepare this deployment, please download the configuration files provided by Syntho for this particular deployment setup. If option 1 was selected, a Ray configuration file using the YAML format will be provided. We will go over the changes necessary to configure this configuration file correctly.

Please also request access to the Docker images for JupyterHub & Ray. These images will have all the necessary software installed to run the Syntho application correctly. We will need to log into the registry on the 

The images necessary for this deployment for both:

- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application.
- syntho-jupyterhub
  - Version: latest
  - Has the latest version of JupyterHub installed that is compatible with the Syntho Application.

### Login into container registry

On each of the VM instances, we need to login into the registry to access the Syntho docker images. Please request your credentials with the Syntho Support. Once the credentials have been received, the following command can be used in each VM instance to login into the registry:

```[sh]
docker login <registry> -u <username>
```

The registry, username and password will be provided by the Syntho Support.

## Deployment using Ray cluster manager (Option 1)

We will be deploying the application with JupyterHub and Ray. We will reserve 2 or more VM instances for Ray and 1 for JupyterHub.

Please see the section [JupyterHub](#jupyterhub) for the deployment of JupyterHub and the section [Ray](#ray) on the deployment of Ray. Together they form the total application landscape for this deployment scenario.

### Configuring the Ray configuration file

We will go through the sections necessary to adjust in the `configuration.yaml` file for Ray. Once completed, we can create the cluster by running `ray up configuration.yaml`.

#### Setting the image

In the `configuration.yaml` file, we need to set the right image for Ray to use. Under `docker.image` we can set the image to use. Example:

```[yaml]
docker:
    image: "<registry>/syntho-ray:latest" 
```

#### Configuring the nodes

Under the section `available_node_types` we can set the amount of nodes that we want to have available. Each section under `available_node_types` defines the amount of workers and the configuration of the node. We will always start with an configuration block for the head node, followed by the workers. An example configuration is:

```[yaml]
available_node_types:
    ray.head.default:
        # The resources provided by this node type.
        resources: {"CPU": 4}
        # Provider-specific config, e.g. instance type.
        node_config:
            azure_arm_parameters:
                vmSize: Standard_D4s_v4
                # List images https://docs.microsoft.com/en-us/azure/virtual-machines/linux/cli-ps-findimage
                imagePublisher: microsoft-dsvm
                imageOffer: ubuntu-1804
                imageSku: 1804-gen2
                imageVersion: latest
    ray.worker.default:
        # The minimum number of worker nodes of this type to launch.
        # This number should be >= 0.
        min_workers: 2
        # The maximum number of worker nodes of this type to launch.
        # This takes precedence over min_workers.
        max_workers: 3
        # The resources provided by this node type.
        resources: {"CPU": 32}
        # Provider-specific config, e.g. instance type.
        node_config:
            azure_arm_parameters:
                vmSize: Standard_D32s_v4
                # List images https://docs.microsoft.com/en-us/azure/virtual-machines/linux/cli-ps-findimage
                imagePublisher: microsoft-dsvm
                imageOffer: ubuntu-1804
                imageSku: 1804-gen2
                imageVersion: latest
                # optionally set priority to use Spot instances
                priority: Spot
                # set a maximum price for spot instances if desired
                # billingProfile:
                #     maxPrice: -1
```

In this case, we have set the node configuration for Azure instances. The head node uses 1 Standard_D4s_v3 instance with a pre-configured Ubuntu 18.04 image. This image has Docker pre-installed.

For the workers, we have set the default amount of workers to be 2 (`available_node_types.ray_worker_default.min_workers`). In case that the resources are being over-used, this configuration will auto-scale up to 3 workers (`available_node_types.ray_worker_default.max_workers`). To keep costs lower, we used spot instances (`available_node_types.ray_worker_default.node_config.azure_arm_parameters.priority`).

We can adjust the amount of workers Ray will spawn by adjust the `resources` parameter. For now we have set this value to be the same as the amount of CPU's available for the given instances.

#### Configuring file mounts

To be able to connect to the head node, it is recommended that a SSH public key is uploaded on the machines from the machine running the Ray commands. We can also upload other files if desired. Here's an example of configuring the file mounts:

```[yaml]
file_mounts: {
#    "/path1/on/remote/machine": "/path1/on/local/machine",
     "~/.ssh/id_rsa.pub": "~/.ssh/id_rsa.pub",
}
```

This example will upload the public key `~/.ssh/id_rsa.pub` on the local machine to all nodes.

#### Setting up the registry credentials

In this setup, the Ray cluster manager will create all the instances. We have to provide the correct command and credentials for docker to work 

### Installing JupyterHub

Under the folder `helm/jupyterhub`, the files for the JupyterHub deployment can be found. We will need to adjust the file `values.yaml` in the upcoming sections. Once we reach the section [Deploy using helm - JupyterHub](#deploy-using-helm---jupyterhub), the `values.yaml` file should be correctly adjusted for your environment and ready to be used by Helm for deploying the application.

#### Image

The image for Syntho should be set under `singleuser.image.name` and `singleuser.image.tag`. It is also important to fill the value of the created secret under `singleuser.image.pullSecrets`. An example of this would be:

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


#### Deploy using Helm - JupyterHub

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
