# Local development proxy

This repository contains a `docker-compose.yaml` file that runs an instance
of [Traefik Proxy](https://traefik.io/traefik/) for local development purposes. This allows you to run multiple web
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
