# Local development proxy

Running multiple Docker (microservice) containers in a local environment can introduce some challenges: only one
container can bind to port 80 at any given time, and assuming that every microservice has their
own `docker-compose.yaml`, it can be difficult to make multiple microservices communicate with each other.
In production, you might use a container orchestration tool (such as Kubernetes) which takes care of these issues, but
running Kubernetes on your local machine is usually overkill.

This repository contains a simple `docker-compose.yaml` file that runs an instance of
[Traefik Proxy](https://traefik.io/traefik/) for local development purposes. This allows you to run multiple web
services locally, without port conflicts, each with their own domain name.

## Getting started

### Requirements

- [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/install/)

### Installation

```shell
git clone git@github.com:nicwortel/local-development-proxy.git
cd local-development-proxy
docker-compose up -d
```

After doing this once, the proxy container will start up automatically when Docker starts (for example after rebooting
your machine), thanks to the line
[`restart: always`](https://github.com/nicwortel/local-development-proxy/blob/master/docker-compose.yaml#L13).

### Usage

In order to connect with the reverse proxy and expose your application, add this to the `docker-composer.yaml` of your
project:

```diff
 version: '3.7'

 services:
   web:
     image: my/image
+    # Connect to this project's network (default) AND the local development proxy network (web)
+    networks:
+      - default
+      - web
+    labels:
+      traefik.enable: 'true'
+      traefik.http.routers.myproject.rule: 'Host(`some-application.localhost`)'
+      traefik.http.routers.myproject.entrypoints: web
+      traefik.docker.network: web

   db:
     image: mysql

+networks:
+  web:
+    external: true
```
