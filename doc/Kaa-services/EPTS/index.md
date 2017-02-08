---
layout: page
title: Persistent data handling
permalink: /:path/
sort_idx: 70
---

{% include variables.md %}

* TOC
{:toc}

## Service overview

Endpoint Time Series service (EPTS) is a [Kaa service]()<!--TODO: link to definition--> that receives endpoint-related time series data of arbitrary structure, persists it, and provides a REST API to retrieve it.

EPTS uses the [NATS messaging system](http://nats.io/) to receive data from [Data Collection extension]()<!--TODO: link to definition--> and store it in the [Apache Cassandra](http://cassandra.apache.org/) database.
EPTS sends response confirmation back to Data Collection extension if required.

[EPTS REST API]()<!--TODO: link--> allows making queries to retrieve the persistent data and visualize it using [Kaa Web service]()<!--TODO: link to definition-->

You can deploy multiple [replicas]()<!--TODO: definition--> of EPTS instance within the same [solution]()<!--TODO: link to definition-->.

### Prerequisites

The following components are required to follow this guide:

* [Java 8 JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* [Maven 3](https://maven.apache.org/)
* [Docker](https://www.docker.com/)
* [NATS messaging system](http://nats.io/)
* [Cassandra 3.9](http://cassandra.apache.org/)

## Build and run

Prior to building and running EPTS, make sure you have Java 8, Maven 3 and Docker already installed on your machine.

>**NOTE:** The `docker` command requires `sudo`.
>Examples below are shown without `sudo`.
>To run all docker commands without `sudo` follow [this instruction](http://askubuntu.com/questions/477551/how-can-i-use-docker-without-sudo).
{:.note}

To build your EPTS project:

1. Start the [NATS docker container](https://hub.docker.com/_/nats/).

   ```bash
   docker run -d -p 4222:4222 --name nats --publish-all=true nats:0.9.6
   ```
    >**NOTE:** To view details about the running and stopped containers, run `docker ps -a`.
    {:.note}

2. Start the [Cassandra docker container](https://hub.docker.com/_/cassandra/).

   ```bash
   docker run --name cassandra -d -p 9042:9042 cassandra:3.9
   ```

3. Download the EPTS source files and build the project using Maven.

   ```bash
   mvn clean install
   ```

    To skip the integration tests, run the command below.

   ```bash
   mvn clean install -DskipTests
   ```

4. Run the `.jar` file of your EPTS project.

   ```bash
   $ service/target/epts-service-0.1.0-SNAPSHOT.jar --instanceId=643833
   ```

    To set a startup argument, use the following startup command syntax: `--{parameter name}={custom value}`.
    For example, you can specify service instance ID:

   ```
   --instanceId=46424.
   ```
    See [Configuration](#configuration).

To initialize Cassandra DB schema, run the following CQL script: `dao/src/test/resources/cassandra.cql`.
If you use docker image for Cassandra, run this script in the container.

## Configuration

Use the `.yml` configuration file to set up Cassandra and NATS for your EPTS.
See table below.

| Full parameter name | Short parameter name | Description | Default value |
|---------------------|----------------------|-------------|---------|
| `epts.nats.urls`                     | `nats.urls`     | Comma-separated list of NATS client connection URLs.                                                     | [nats://127.0.0.1:4222](nats://127.0.0.1:4222) |
| `epts.nats.username`                 | `nats.username` | Username for NATS client connection.                                                                     |           |
| `epts.nats.password`                 | `nats.password` | Password for NATS client connection.                                                                     |           |
| `epts.service.instance.id`           | `instanceId`    | Unique identifier of service instance (service plus its configuration).                                  |           |
| `epts.nats.subject.protocol-version` |                 | IPC protocol version.                                                                                    | [See EPTS configuration file](http://gitlab.cybervisiontech.com/kaaiot/EPTS/blob/master/service/src/main/resources/application.yml)<!--TODO: verify link--> |
| `epts.nats.subject.service-name`     |                 | Service name to be used in NATS subjects.                                                                | [See EPTS configuration file](http://gitlab.cybervisiontech.com/kaaiot/EPTS/blob/master/service/src/main/resources/application.yml) |
| `epts.nats.subject.name`             |                 | NATS subject to subscribe to. All service instance replicas are subscribed to this subject.              | [See EPTS configuration file](http://gitlab.cybervisiontech.com/kaaiot/EPTS/blob/master/service/src/main/resources/application.yml) |
| `cassandra.client.clusterName`       | `clusterName`   | Cassandra cluster name.                                                                                  | [See EPTS configuration file](http://gitlab.cybervisiontech.com/kaaiot/EPTS/blob/master/service/src/main/resources/application.yml) |
| `cassandra.client.keyspaceName`      | `keyspaceName`  | Cassandra keyspace name.                                                                                 | [See EPTS configuration file](http://gitlab.cybervisiontech.com/kaaiot/EPTS/blob/master/service/src/main/resources/application.yml) |
| `cassandra.client.nodeList`          | `nodeList`      | Comma-separated list of Cassandra cluster node addresses. List element format: *IP:port* (e.g.: 127.0.0.1:9042).    | 127.0.0.1 |

## Docker setup

After you built the project, you can configure Docker containers for your EPTS:

1. Install [Docker Engine](https://docs.docker.com/engine/installation/).

    If you use Linux, you can install Docker Engine by running the [installation script](https://get.docker.com/).

   ```bash
   wget -qO- https://get.docker.com/ | sh
   ```

   >**IMPORTANT:** If you use Windows, you must also have [Python](https://www.python.org/downloads/) installed.
   {:.important}

2. Install [Docker Compose](https://docs.docker.com/compose/install/).

3. Run the below commands from the `EPTS/containers/docker/` directory.

   ```bash
   docker-compose -f nats-docker-compose.yml up -d
   docker-compose -f cassandra-db-docker-compose.yml up -d
   docker-compose -f epts-docker-compose.yml up -d
   ```

    Run `sudo docker ps -a` to view details about the running and stopped containers.

### Shell access

To access EPTS container shell, run:

```bash
docker exec -it ${epts container id} bash
```

To access Cassandra database container shell, run:

```bash
docker exec -it ${Cassandra-db container id} bash
```

### Local image build

As an alternative to using Docker Compose, you can build image separately.
To do this, run from the EPTS root directory:

```bash
docker build -f containers/docker/epts/Dockerfile --build-arg setupfile=service/target/epts-service-0.1.0-SNAPSHOT.jar -t epts-service:0.1.0 .
```

Run EPTS container with `--network` flag to specify which network you want to run the container on.

```bash
docker run -d --network=kaa-tier --env-file containers/docker/epts/epts-service.env epts-service:0.1.0
```

In the example above, `kaa-tier` is the network used in the NATS docker container.

Environment variables for the EPTS image are located in the `EPTS/containers/docker/epts/epts-service.env` file.
See table below.

| Parameter name | Default value |
|----------------|---------------|
| `INSTANCE_ID`         | eptsInstanceId           |
| `CASSANDRA_HOST_PORT` | cassandra-db:9042        |
| `NATS_HOST_PORT`      | [nats://nats-service:4222](nats://nats-service:4222) |

Environment variables for the Cassandra image are located in the `EPTS/containers/docker/epts/cassandra-db.env` file.
See table below.

| Parameter name | Default value |
|----------------|---------------|
| `CASSANDRA_CLUSTER_NAME` | Kaa Cluster |

## Kubernetes setup

As an option, you can use [Kubernetes](https://kubernetes.io/) to orchestrate your EPTS docker containers.
Prior to installing Kubernetes, make sure you have built the [Docker images](#local-image-build).

### Install

To install Kubernetes, use one of the options:

* [Install using `kubeadm`](https://kubernetes.io/docs/getting-started-guides/kubeadm/) --- for Ubuntu 16.04, CentOS 7, and HypriotOS v1.0.1+.

* Run Kubernetes locally [using Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/).

After installation, start the Kubernetes pods for Cassandra, NATS, and EPTS by running the commands below from the `EPTS/containers/kubernetes` directory:

```
kubectl create -f kube-cassandra.yml
kubectl create -f kube-nats.yml
kubectl create -f kube-epts.yml
```

To check pods status, run:

```
kubectl get pods
```

To check services status, run:

```
kubectl get service
```

For better data visualization, you can [deploy Dashbord UI for Kubernetes](https://kubernetes.io/docs/user-guide/ui/#deploying-the-dashboard-ui).

### Configure

Kubernetes allows binding ports in the 30000-32767 range on localhost.
This allows you to manage your pods from external network.

Environment variables for the EPTS service are located in the `EPTS/containers/kubernetes/kube-epts.yml` file.
See table below.

| Parameter name | Default value |
|---------------------|---------------|
| `GET_HOSTS_FROM`      | dns                                                |
| `INSTANCE_ID`         | eptsInstanceId                                     |
| `CASSANDRA_HOST_PORT` | cassandra.default.svc.cluster.local:9042           |
| `NATS_HOST_PORT`      | [nats://nats-service.default.svc.cluster.local:4222](nats://nats-service.default.svc.cluster.local:4222) |