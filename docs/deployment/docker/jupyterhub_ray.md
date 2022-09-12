# Deployment instructions for JupyterHub/Ray Syntho with Docker

## Table of Contents

1. [Introduction](#introduction)
2. [Requirements](#requirements)
3. [Preparations](#preparations)
4. [Deployment using Ray cluster manager (Option 1)](#deployment-using-ray-cluster-manager-option-1)
5. [Deployment using Ray cluster manager (Option 2)](#deployment-using-ray-cluster-manager-option-2)
6. [Testing the application](#testing-the-application)
7. [Upgrading the application](#upgrading-the-application)

## Introduction

To use the Syntho Application for this specific deployment option, we need to install both Ray and JupyterHub as part of this Syntho Application. JupyterHub will be used as the interface running Python Notebooks that run our Syntho Application on the Ray cluster. Deploying the Syntho Application with this JupyterHub/Ray solution using Docker can be done a few ways:

- Option 1: Depending on whether support for for autoscaling Ray workers is necessary, we can either deploy by using the Ray cluster manager directly in the following cloud providers: AWS, GCP, Azure.
The Ray cluster manager will enable autoscaling based on a given configuration. We will then separately deploy a single instance running JupyterHub using `docker-compose` in the same network as the Ray cluster. See section [Deployment using Ray cluster manager](#deployment-using-ray-cluster-manager-option-1) (Recommended)

- Option 2: Deploying Ray instances manually using Docker. Autoscaling using Ray will not be available and the nodes need to be connected manually to the head node of Ray. JupyterHub will still be deployed using `docker-compose`. See section [Deployment using manual Ray cluster (Option 2)](#deployment-using-manual-ray-cluster-option-2)

## Requirements

To install the Syntho Application together with JupyterHub & Ray, the following requirements need to be met:

- Have at least 3 VM instances running.
  - OS: Ubuntu 18.04 or higher.
  - Preferably with SSD storage.
  - Docker 1.13.0+ ​installed.
    - Please see the [official docker documentation](https://docs.docker.com/engine/install/) for the installation instructions.
  - `docker-compose` 2.x.x  installed.
    - Please see the [official docker-compose documentation](https://docs.docker.com/compose/install/) for the installation instructions.
  - `docker-compose` file version will be v3.
- Reachable IP address and outbound port defined for JupyterHub instance (by default this is port 8080).
- Internal networking between 3 VM instances.
- [Optional] DNS zone and record for JupyterHub.
  - A reverse proxy should be setup for the JupyterHub instance in this case.
  - Example: syntho.company.com be used for hosting the interface of JupyterHub.
- Docker images
  - Either by having access to the container registry.
  - or loading them in manually using `docker load` after receiving the docker images in .tar.gz format.
- Configuration files
  - The Syntho Support Team will provide the required files depending on the chosen deployment option.

## Preparations

To prepare this deployment, please download the configuration files provided by Syntho for this particular deployment setup. If option 1 was selected, a Ray configuration file using the YAML format will be provided. We will go over the changes necessary to configure this configuration file correctly.

Please also request access to the Docker images for JupyterHub & Ray. These images will have all the necessary software installed to run the Syntho application correctly. All of the VM instances need to log into the registry using `docker login`, see the section [Login into container registry](#login-into-container-registry)

The images necessary for this deployment for both:

- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application.
- syntho-jupyterhub
  - Version: latest
  - Has the latest version of JupyterHub installed that is compatible with the Syntho Application.

### Login into container registry

On each of the VM instances, we need to login into the registry to access the Syntho docker images. Please request your credentials with the Syntho Support Team. Once the credentials have been received, the following command can be used in each VM instance to login into the registry:

```[sh]
docker login <registry> -u <username>
```

The registry, username and password will be provided by the Syntho Support Team.

## Deployment using Ray cluster manager (Option 1)

We will be deploying the application with JupyterHub and Ray. We will reserve 2 or more VM instances for Ray and 1 for JupyterHub.

Please see the section [JupyterHub](#jupyterhub) for the deployment of JupyterHub and the section [Ray](#ray) on the deployment of Ray. Together they form the total application landscape for this deployment scenario.

### Configuring the Ray configuration file

We will go through the sections necessary to adjust in the `configuration.yaml` file for Ray. Once completed, we can create the cluster by running `ray up configuration.yaml`.

#### Setting the name of the cluster

The name of the cluster can be set using the `cluster_name` block. See:

```[yaml]
cluster_name: syntho_cluster
```

#### Setting the image

In the `configuration.yaml` file, we need to set the right image for Ray to use. Under `docker.image` we can set the image to use. Example:

```[yaml]
docker:
    image: "<registry>/syntho-ray:latest" 
```

#### Configuring the nodes

Under the section `available_node_types` we can set the amount of nodes that we want to have available. Each section under `available_node_types` defines the amount of workers and the configuration of the node. We will always start with a configuration block for the head node, followed by the workers. An example configuration is:

```[yaml]

# The maximum number of workers nodes to launch in addition to the head
# node.
max_workers: 4

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

The Syntho Support Team can help for deciding what the optimal cluster configuration should be. Please request an initial cluster configuration based on the data requirements with them.

In this case, we have set the node configuration for Azure instances. The head node uses 1 Standard_D4s_v3 instance with a pre-configured Ubuntu 18.04 image. This image has Docker pre-installed.

For the workers, we have set the default amount of workers to be 2 (`available_node_types.ray_worker_default.min_workers`). In case that the resources are being over-used, this configuration will auto-scale up to 3 workers (`available_node_types.ray_worker_default.max_workers`). To keep costs lower, we used spot instances (`available_node_types.ray_worker_default.node_config.azure_arm_parameters.priority`).

We also need to set the amount of workers on `max_workers`, which represents the amount of workers that can be spawned (not counting the head node).

We can adjust the amount of workers Ray will spawn by adjusting the `resources` parameter. For now we have set this value to be the same as the amount of CPUs available for the given instances.

#### Configuring file mounts

To be able to connect to the head node, it is recommended that a SSH public key is uploaded on the machines from the machine running the Ray commands. We can also upload other files if desired. Here's an example of configuring the file mounts:

```[yaml]
file_mounts: {
#    "/path1/on/remote/machine": "/path1/on/local/machine",
     "~/.ssh/id_rsa.pub": "~/.ssh/id_rsa.pub",
}
```

This example will upload the public key `~/.ssh/id_rsa.pub` on the local machine to all nodes. Please adjust this to include a public key of your choice that will be uploaded to the Ray nodes. This key can be used for diagnostic purposes, like retrieving the logs from a certain node. Any other file that is necessary to be on all the machines can be added here as well.

#### Setting up the registry credentials

In this setup, the Ray cluster manager will create all the instances. We have to provide the correct command and credentials for docker to work. We will do so by setting up the block `initialization_commands`. An example of this block:

```[yaml]
initialization_commands:
    # enable docker setup
    - sudo usermod -aG docker $USER || true
    - sleep 10  # delay to avoid docker permission denied errors
    # get rid of annoying Ubuntu message
    - touch ~/.sudo_as_admin_successful
    - docker login <registry> -u <username> -p <password>
```

To prevent an issues, we first add the current user to the docker group and remove the standard Ubuntu messages. Once that is done, we call `docker login` to connect to the registry. We included the password flag `-p` here, since we can't provide CLI input with the password in this way.

### Creating the Ray cluster

Once the `configuration.yaml` has been setup correctly, we can use the following command to create the Ray cluster:

```[sh]
ray up configuration.yaml
```

Remember the IP address of the Ray head node (or the hostname of the machine), so that we can use that later to connect to the cluster from JupyterHub.

### Setting up JupyterHub

We will be setting up JupyterHub using docker-compose in the dedicated instance for JupyterHub. The creation of the Ray cluster will have setup a virtual network that the Ray cluster nodes will use. It is important that the instance running JupyterHub is either added to or created in that network.

We will then configure the environment variables and the JupyterHub configuration file in the next steps for the folder containing the docker-compose file for JupyterHub.

#### Docker image

In the environment variables, we need to set the following variable to use the correct docker image:

```[sh]
DOCKER_NOTEBOOK_IMAGE=<registry>/syntho-jupyter:latest
```

This will create a Docker environment for every user that logs in, using the syntho-jupyter image. 

#### Application access

Depending on how the application needs to be accessed, a simple IP address or DNS record can be used. Please remember the private or public IP address of the instance to assign it to a DNS record.

If a DNS record (public or private) is used, it is recommended to setup HTTPS using SSL certificates. If there are certificates, they can be uploaded to the same directory as the docker-compose file. Next uncomment the following lines in `jupyterhub_config.py`:

```[python]
#c.JupyterHub.ssl_key = os.environ['SSL_KEY']
#c.JupyterHub.ssl_cert = os.environ['SSL_CERT']
```

Next set the environment variables `SSL_KEY` and `SSL_CERT` to their correct values.

#### Authentication method

Next we will define the authentication method for the JupyterHub environment. There are a multitude of choices possible, see the [JupyterHub documentation](https://jupyterhub.readthedocs.io/en/stable/reference/authenticators.html) for all possibilities.

We can define this in the file `jupyterhub_config.py`. We recommend a more secure option, like Azure AD. See an example of that here:

```[python]
import os
from oauthenticator.azuread import AzureAdOAuthenticator
c.JupyterHub.authenticator_class = AzureAdOAuthenticator

c.Application.log_level = 'DEBUG'

c.AzureAdOAuthenticator.tenant_id = os.environ.get('AAD_TENANT_ID')

c.AzureAdOAuthenticator.oauth_callback_url = 'http://{your-domain}/hub/oauth_callback'
c.AzureAdOAuthenticator.client_id = '{AAD-APP-CLIENT-ID}'
c.AzureAdOAuthenticator.client_secret = '{AAD-APP-CLIENT-SECRET}'
```

The preferred method of authentication could differ of course. For more help on setting the Authentication method, please contact the Syntho Support Team.

#### Deploy using docker-compose - JupyterHub

Once the configuration is done, we can run the containers in detached mode using the following command:

```[sh]
docker-compose up -d
```

If any issues arise during this step, please contact the Syntho Support Team.

## Deployment using manual Ray cluster (Option 2)

The deployment of Option 1 overlaps with Option 2 when it comes to deploying JupyterHub. **Please refer to section [Setting up JupyterHub](#setting-up-jupyterhub) for the instructions on how to deploy JupyterHub.** We will use the remaining VM instances to run certain Docker commands on to create the head node and the corresponding worker nodes. Please make sure that they are part of the same network, so that we can reach the other machines on most ports.

### Creating the head node

On the VM instance that is designated to run as the head node, make sure that Docker is installed and the `docker login` has been executed with the supplied credentials for the container registry. In the folder `docker-compose/ray/option-2`, copy the file `docker-compose-head.yaml` to the head node instance.

No configuration needs to be adjusted for this, we can now simply run:

```[sh]
docker-compose up
```

Once the container is started, look into the logs by using `docker-compose logs` and note down the IP address found in this line:

```[sh]
ray-head  | 2022-06-02 00:37:05,751     INFO scripts.py:744 -- To connect to this Ray runtime from another node, run
ray-head  | 2022-06-02 00:37:05,752     INFO scripts.py:747 --   ray start --address='<ip-address>:6379'
```

This IP address should be the private IP from within the created network. We will need this IP as an environment variable for the worker to correctly connect to the head node instance.

### Creating the worker nodes

On the VM instances that are assigned as the workers, we need to use the file `docker-compose-worker.yaml` in the folder `docker-compose/ray/option-2` to run the worker. Also make sure that `docker login` has been run on this machine, so that we have access to the registry containing the container.

First we need to add the environment variable with the head node IP. Please create the file `.env` and add:

```[sh]
RAY_HEAD_IP=<ip-of-head-node>
```

Once this `.env` file is created, we can simply run:

```[sh]
docker-compose up
```

This should create the worker and connect it to the head node. If any issues arise, make sure that ports 6379 and 10001 are accessible on the head node for all workers.

## Testing the application

Once both Ray and JupyterHub have been installed using either Option 1 or Option 2, login into the JupyterHub application in `http://<ip-address-or-dns-of-jupyterhub>/hub`. Logging in will trigger the creation of an environment. Once an environment is created, a simple test to check whether Ray installed correctly and is accessible for JupyterHub, is to run the following command in a `Python Notebook`:

```[python]
import ray

ray.init("ray://<ip-or-hostname-of-ray-cluster-head>:10001")
```

Please change `<ip-or-hostname-of-ray-cluster-head>` into the IP or Hostname of the Ray Head instance. This will connect to the Ray cluster and will return something like this:

```[sh]
ClientContext(dashboard_url='10.244.1.21:8265', python_version='3.9.5', ray_version='1.12.1', ray_commit='4863e33856b54ccf8add5cbe75e41558850a1b75', protocol_version='2022-03-16', _num_clients=1, _context_to_restore=<ray.util.client._ClientContext object at 0x7f6c24257b50>)
```

To check whether all workers are connected probably, we can print the worker information from this notebook with the following code:

```[python]
print(ray.nodes())
```

This will print out a list of dictionaries with information about each node in the cluster.

## Upgrading the application

In cases of upgrades, we can provide the latest version of the application using our container registry or by providing the docker image as a file. Please contact the Syntho Support Team for more information.

### Upgrade requirements

Please make sure that the latest version of the docker image is pulled or downloaded and saved using the `docker load` command. The following images a eligible for updates:

- `syntho-ray`
- `syntho-jupyterhub`

### JupyterHub

In both options, we will have provided a JupyterHub docker-compose template in which we can change the image version. Please make sure the correct image is either loaded in using `docker load` or already pulled using `docker pull`. The following command will pull the latest version of the image using the container registry:

```[sh]
docker pull synthoregistry.azurecr.io/syntho-jupyterhub:<latest-image-version>
```

In case the image is not accessible via the container registry, the Syntho Support Team will provide an updated image in `.tar.gz` file. As stated in the [introduction](../introduction.md), we can use the following command to load the image:

```[sh]
docker load < syntho-image.tar.gz
# OR
docker load --input syntho-image.tar.g
```

In the folder `docker-compose/jupyterhub`, we can now change the image version in the `.env` file to the latest version:

```[sh]
DOCKER_NOTEBOOK_IMAGE=synthoregistry.azurecr.io/syntho-jupyterhub:<latest-image-version>
```

Afterwards we can then upgrade the docker-compose instance by running the following command:

```[sh]
docker-compose restart
```

### Ray

For option 1, we can change the image version in the `configuration.yaml` file under the following part of the YAML file:

```[yaml]
docker:
    image: "synthoregistry.azurecr.io/syntho-ray:latest" 
```

We can then upgrade the cluster using the following command:

```[sh]
ray up configuration.yaml
```

For option 2, we will have provided a Ray docker-compose template in which we can change the image version. Please make sure the correct image is either loaded in using `docker load` or already pulled using `docker pull`. We can change the following part of the `docker-compose-head.yaml` file. For the head node, we will change the image version to the latest version:

```[yaml]
services:
    ray-head:
        image: "synthoregistry.azurecr.io/syntho-ray:<latest-image-version>"
```

For each worker, we can adjust the `docker-compose-worker.yaml` file to change the image version like this:

```[yaml]
services:
    ray-worker:
        image: "synthoregistry.azurecr.io/syntho-ray:<latest-image-version>"
```

On each node, run the following command after upgrading the image version:

```[sh]
docker-compose up -d
```

## Manually saving logs

To create a copy of the last saved logs lines for this application, docker-compose can be used. To create a log file of the last saved lines in docker-compose, we can use the following command:

```[sh]
docker compose logs >> syntho_logs.log
```

This will create a log file with the name `syntho_logs.log` in the current directory.
