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
The Ray cluster manager will enable autoscaling based on a given configuration. We will then separately deploy a single instance running the Syntho Application using `docker-compose` in the same network as the Ray cluster. See section Deployment using Ray cluster manager in the [file for the JupyterHub & Ray deployment](./jupyterhub_ray.md#deployment-using-ray-cluster-manager-option-1) (Recommended)

- Option 2: Deploying Ray instances manually using Docker. Autoscaling using Ray will not be available and the nodes need to be connected manually to the head node of Ray. The Syntho Application will still be deployed using `docker-compose`. See section Deployment using manual Ray cluster (Option 2) in the [file for the JupyterHub & Ray deployment](./jupyterhub_ray.md#deployment-using-manual-ray-cluster-option-2)

The Syntho Application itself will be installed using a docker-compose file together with the correct environment variables set. This document will continue to explain the installation of the Syntho Application and not Ray, see the [file for the JupyterHub & Ray deployment](./jupyterhub_ray.md) for more information on the Ray part of the deployment.

## Requirements

To install the Syntho Application, the following requirements need to be met:

- Have at least 3 VM instances running.
  - OS: Any Linux OS capable of running docker & docker-compose.
  - Preferably with SSD storage.
  - Docker 1.13.0+ ​installed.
    - Please see the [official docker documentation](https://docs.docker.com/engine/install/) for the installation instructions.
  - `docker-compose` 2.x.x  installed.
    - Please see the [official docker-compose documentation](https://docs.docker.com/compose/install/) for the installation instructions.
  - `docker-compose` file version will be v3.
- Postgres database
  - Either by including a Postgres database in the deployment or have an external database available. Two different databases need to be created in this instances.
- Redis instance
  - The redis instances can be included when deploying using docker-compose. If this is removed from the docker-compose, a Redis instance needs to be created for the Syntho Application to connect to.
- Reachable IP address and outbound port defined for the Syntho Application (by default this is port 3000).
- Internal networking between 3 VM instances.
- Docker images
  - Either by having access to the container registry.
  - or loading them in manually using `docker load` after receiving the docker images in .tar.gz format.
- Configuration files
  - The Syntho Support Team will provide the required files depending on the chosen deployment option.
- [Optional] DNS zone and DNS record for UI.
  - Example: syntho.company.com be used for hosting the UI.

## Preparations

To prepare this deployment, please download the configuration files provided by Syntho for this particular deployment setup. If option 1 was selected, a Ray configuration file using the YAML format will be provided. We will go over the changes necessary to configure this configuration file correctly.

Please also request access to the necessary Docker images. These images will have all the necessary software installed to run the Syntho application correctly. All of the VM instances need to log into the registry using `docker login`, see the section [Login into container registry](#login-into-container-registry)

The images necessary for this deployment for both Ray and the Syntho Application:

- syntho-ray
  - Version: latest
  - Has the latest Ray version installed that is compatible with the Syntho Application. See the document [Jupyterhub & Ray deployment](./jupyterhub_ray.md#deployment-using-manual-ray-cluster-option-2) for more the deployment instructions for Ray.
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

To connect the Core API to a Ray instance, we need to set the variable `RAY_ADDRESS` in the `.env` file. This should be set to the IP address or hostname of the Ray head instance. This refers back to the [other document](./jupyterhub_ray.md), where we explain how to install Ray. At the end of those steps, the IP or hostname of the Ray head instance should be known.

```[sh]
RAY_ADDRESS=<ray-head-ip-address>
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
