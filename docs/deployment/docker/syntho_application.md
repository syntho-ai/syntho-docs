# Deployment instructions for Syntho Application

## Table of Contents

1. [Introduction](#introduction)
2. [Requirements](#requirements)
3. [Preparations](#preparations)
4. [The Syntho Application deployment](#the-syntho-application-deployment)
5. [Running the application](#running-the-application)

## Introduction

To use the Syntho Application for this specific deployment option, we need to install both Ray and the Syntho Application as part of this Syntho Application. The Syntho Application will be used as the self-service interface for our application. Deploying the Syntho Application and Ray using Docker can be done a few ways:

- Option 1: Depending on whether support for for autoscaling Ray workers is necessary, we can either deploy by using the Ray cluster manager directly in the following cloud providers: AWS, GCP, Azure.
The Ray cluster manager will enable autoscaling based on a given configuration. We will then separately deploy a single instance running the Syntho Application using `docker-compose` in the same network as the Ray cluster. See section [Deployment using Ray cluster manager (Option 1)](#deployment-using-ray-cluster-manager-option-1).

- Option 2: Deploying Ray instances manually using Docker. Autoscaling using Ray will not be available and the nodes need to be connected manually to the head node of Ray. The Syntho Application will still be deployed using `docker-compose`. See section [Deployment using manual Ray cluster (Option 2)](#deployment-using-manual-ray-cluster-option-2).

The Syntho Application itself will be installed using a docker-compose file together with the correct environment variables set. See section [The Syntho Application deployment](#the-syntho-application-deployment).

## Requirements

To install the Syntho Application, the following requirements need to be met:

- Have at least 2 VM instances running.
  - OS: Any Linux OS capable of running docker & docker-compose.
  - Preferably with SSD storage.
  - Docker 1.13.0+ ​installed.
    - Please see the [official docker documentation](https://docs.docker.com/engine/install/) for the installation instructions.
  - `docker-compose` 2.x.x  installed.
    - Please see the [official docker compose documentation](https://docs.docker.com/compose/install/) for the installation instructions.
  - `docker-compose` file version will be v3.
- Postgres database
  - The docker compose file will have a postgres instance included. It is also possible to use an external database. In case an external database is used, two different databases need to be available.
- Redis instance
  - The redis instances is included when deploying using docker compose. If this is removed from the docker-compose, a Redis instance needs to be created for the Syntho Application to connect to.
- Reachable IP address and outbound port defined for the Syntho Application (by default this is port 3000).
- Internal networking between 3 VM instances.
- Docker images
  - Either by having access to the container registry.
  - or loading them in manually using `docker load` after receiving the docker images in .tar.gz format.
- Configuration files
  - The configuration files are available in the Syntho repository on Github called [syntho-charts](https://github.com/syntho-ai/syntho-charts/tree/master/docker-compose). The relevant folders for this deployment are syntho-ui and ray.
- [Optional] DNS zone and DNS record for UI.
  - Example: syntho.company.com be used for hosting the UI.
- [Optional] SSL certificate
  - If the Syntho Application is to be accessed via HTTPS, a SSL certificate is required. This is highly recommended for production environments.

## Preparations

To prepare this deployment, please download the configuration files provided by Syntho for this particular deployment setup. If option 1 was selected, a Ray configuration file using the YAML format will be provided. We will go over the changes necessary to configure this configuration file correctly.

Please also request access to the necessary Docker images. These images will have all the necessary software installed to run the Syntho application correctly. All of the VM instances need to log into the registry using `docker login`, see the section [Login into container registry](#login-into-container-registry)

The images necessary for this deployment for both Ray and the Syntho Application:

- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application.
- syntho-core-api
  - Version: latest
  - The Syntho Core API is responsible for the core operations of the Syntho Platform.
- syntho-frontend
  - Version: latest
  - The Syntho Frontend is a container that contains the web UI for the Syntho Platform.
- syntho-backend
  - Version: latest
  - The Syntho Backend is responsible for user management and workspace management.

### Login into container registry

On each of the VM instances, we need to login into the registry to access the Syntho docker images. Please request your credentials with the Syntho Support Team. Once the credentials have been received, the following command can be used in each VM instance to login into the registry:

```[sh]
docker login <registry> -u <username>
```

The registry, username and password will be provided by the Syntho Support Team.

## Deployment using Ray cluster manager (Option 1)

This deployment option will use the Ray cluster manager to deploy the Ray cluster. The Ray cluster manager will automatically deploy the Ray cluster based on the configuration file provided. The Syntho Application will be deployed using `docker-compose` in the same network as the Ray cluster. The files necessary for the Ray deployment can be found [here](https://github.com/syntho-ai/syntho-charts/tree/master/docker-compose/ray)

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

Remember the IP address of the Ray head node (or the hostname of the machine), so that we can use that later to connect the cluster to our Syntho Application.

## Deployment using manual Ray cluster (Option 2)

For this setup, we will be using the files found in our `syntho-charts` repo, specifically [here](https://github.com/syntho-ai/syntho-charts/tree/master/docker-compose/ray/option-2).

### Creating the head node

On the VM instance that is designated to run as the head node, make sure that Docker is installed and the `docker login` has been executed with the supplied credentials for the container registry. In the folder `docker-compose/ray/option-2`, copy the file `docker-compose-head.yaml` to the head node instance.

First we need to add the environment variable with the head node IP, license key and image name. Copy the `example.env` file to `.env` using `cp example.env .env` and make sure the following variables are set:

```[sh]
SYNTHO_RAY_IMAGE=<syntho-ray-image>
LICENSE_KEY=<license-key>
```

Next, we need to copy the file `docker-compose-head.yaml` to `docker-compose.yaml`. We can do so by running the following command:

```[sh]
cp docker-compose-head.yaml docker-compose.yaml
```

Once the `.env` and `docker-compose.yaml` file is created, we can simply run:

```[sh]
docker compose up -d
```

Once the container is started, look into the logs by using `docker compose logs` and note down the IP address found in this line:

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
SYNTHO_RAY_IMAGE=<syntho-ray-image>
LICENSE_KEY=<license-key>
```

Once this `.env` file is created, we can simply run:

```[sh]
docker compose up -d
```

This should create the worker and connect it to the head node. If any issues arise, make sure that ports 6379 and 10001 are accessible on the head node for all workers.

## The Syntho Application deployment

For this deployment, the configuration files for the Syntho Application will be provided. The Syntho Application will be deployed using `docker-compose`. In order to configure the Syntho Application, we will need to adjust the environment variables in the folder given. An example `.env` will be provided in the configuration files.

### Configuring the Syntho Application

To configure the full application using the `docker-compose` file, we must first change the `.env` file to contain the right variables. Depending on whether the images are pulled via the registry, or loaded in locally, the `.env` file will need to be changed accordingly. Please change the following values to the right values based on the given names of the images:

```[sh]
CORE_IMAGE=<image-name-of-syntho-core-api>
BACKEND_IMAGE=<image-name-of-syntho-backend>
FRONTEND_IMAGE=<image-name-of-syntho-frontend>
```

The next step is to set the hostname for the frontend and backend application. As these application are user-facing, we will need to set a reachable IP or hostname for these applications. Please set the following environment variables to the right values:

```[sh]
BACKEND_HOST=localhost:8000  # The hostname for the backend application.
FRONTEND_HOST=localhost:3000  # The hostname for the frontend application.
BACKEND_PROTOCOL=http # The protocol for the backend application, http of https depending on the availability of a secure connection.
FRONTEND_PROTOCOL=http  # The protocol for the frontend application, http of https depending on the availability of a secure connection.
```

Lastly we will need to set the environment variable for the license key. This license key will be provided by the Syntho Team. Please set the following environment variable to the right value:

```[sh]
LICENSE_KEY=<license-key>
```

For more information, contact the Syntho Team.

### Configuring the Core API

The Core API is responsible for the core operations of the Syntho Platform. This service will need access to a Postgres database, a Redis instance, and a Ray cluster. We will need to set the right credentials for each of these applications.

#### Core database

To start, we need to configure the credentials for the Postgres database. A Postgres instance is included in the `docker-compose` file, but if a separate Postgres instance is used, the credentials need to be set in the `.env` file.

```[sh]
CORE_DATABASE_HOST=<database-host>
CORE_DATABASE_USER=<database-user>
CORE_DATABASE_PASSWORD=<database-password>
CORE_DATABASE_NAME=<database-name>
```

If the Postgres instance in docker-compose is used, these credentials don't need to be changed.

#### Redis

To configure the Redis credentials, the following environment variables can be set in the `.env` file:
  
```[sh]
CORE_CELERY_BROKER_URL=redis://<redis-host>:6379/0
CORE_CELERY_RESULT_BACKEND=redis://<redis-host>:6379/0
```

If the Redis instance defined in the `docker-compose` file is used, then these credentials don't need to be changed.

#### Ray

To connect the Core API to a Ray instance, we need to set the variable `CORE_RAY_ADDRESS` in the `.env` file. This should be set to the IP address or hostname of the Ray head instance.

```[sh]
CORE_RAY_ADDRESS=<ray-head-ip-address>
```

#### Setting additional variables

Import other variables are the `CORE_SECRET_KEY`, which is used as an encryption key. This key can be generated using the following command using python:

```[sh]
# If cryptography is not installed, install it with `pip install cryptography`.
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

In the case that issues with the port of the application arise, the default port can be changed using the `CORE_PORT` variable in the `.env` file.

```[sh]
CORE_PORT=8000
```

### Configuring the Backend

The backend application is mainly responsible for responding to requests from the frontend and the user management. The backend will need to connect to the Core API for this. The backend will also need to connect to a Redis instance and a Postgres instance. In this deployment, we use the postgres instance in `docker-compose` for the backend as well as for the Syntho Core API.

#### Backend database

The backend will need to connect to the Postgres database. The credentials for this database will be set in the `.env` file. We can set the following variables:

```[sh]
BACKEND_DB_HOST=postgres
```

#### Backend redis

For the redis instance used for the backend, we can set the following variables. The redis instance is included in the `docker-compose` file, but if a separate redis instance is used, the credentials need to be set in the `.env` file.

```[sh]
BACKEND_REDIS_HOST=redis # Redis hostname to connect to
BACKEND_REDIS_PORT=6379 # Redis port to connect to
BACKEND_REDIS_DB_INDEX=1 # Redis database index to use
```

#### Admin user credentials

The backend will need to have an admin user to manage the users and workspaces. The credentials for this user can be set in the `.env` file. The following variables can be set:

```[sh]
ADMIN_USERNAME=admin
ADMIN_PASSWORD=somepassword
ADMIN_EMAIL=admin@company.com
```

These credentials can be used in the UI application to login.

#### Additional environment variables

For the backend, some additional environment variables are needed. First of all, we need to define a port for the backend to use. To change the port, we can change the value in `BACKEND_PORT`:

```[sh]
BACKEND_PORT=8000
```

The backend port will need to exposed as the user should be able to access the backend. Whatever value is being chosen either needs to be exposed as an outbound port or port forwarded to another outbound port.

We then have the secret key that the backend uses for encryption purposes. We can set this key to any value, as long as it is a rather long string. Example:

```[sh]
BACKEND_SECRET_KEY="66n6ldql(b2g0jmop(gr)@x0tz!*^7(d1_$2#y&$t&3r$=cr%#"
```

### Configuring the Frontend

The frontend is responsible for the user interface. The frontend will need to connect to the Backend for this. The most important variable that are left for the Frontend is the following:

```[sh]
FRONTEND_PORT=3000
```

This variable sets the port for the frontend, which will need to be exposed for the user to be able to access the application. 

## Running the application

Once all environment variables are set correctly, we can now run the application by running the following command:

```[sh]
docker-compose up -d  # -d for detached mode
```

The application will need a moment to fully spin up. Once the application is running, we can access the application by going to the following address `http(s)://<hostname-or-ip>:3000`.

## Manually saving logs

To create a copy of the last saved logs lines for this application, docker-compose can be used. To create a log file of the last saved lines in docker-compose, we can use the following command:

```[sh]
docker compose logs >> syntho_logs.log
```

This will create a log file with the name `syntho_logs.log` in the current directory.
