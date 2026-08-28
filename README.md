# Local development proxy

Running multiple (Docker or Podman) microservice containers on the same local machine can introduce some challenges:
only one container can bind to port 80 at any given time, and assuming that every microservice has their
own `compose.yaml`, it can be difficult to make multiple microservices communicate with each other.
In production, you might use a container orchestration tool (such as Kubernetes) which takes care of these issues, but
running Kubernetes on your local machine is usually overkill.

This repository contains a simple `compose.yaml` file that runs an instance of
[Traefik Proxy](https://traefik.io/traefik/) for local development purposes. This allows you to run multiple web
services locally, without port conflicts, each with their own domain name.

## How it works

The `compose.yaml` in this repository creates a new Docker network and starts an instance of Traefik, a reverse
proxy. Port 80 of the Traefik container is mapped to port 80 of the host. Other applications (with their
own `compose.yaml`) can connect to the shared network and will be proxied by Traefik instead of publishing their
ports on the host machine.

Traefik can be configured using Docker labels, so exposing a new application is a matter of adding a network and some
labels to the application's `compose.yaml`.

## Getting started

### Requirements

Either:

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/)
- or [Podman](https://podman.io/) with a Compose provider (`podman-compose` or `docker-compose`), rootless

### Installation (Docker)

```shell
git clone git@github.com:nicwortel/local-development-proxy.git
cd local-development-proxy
docker compose up -d
```

After doing this once, the proxy container will start up automatically when Docker starts (for example after rebooting
your machine), thanks to the line
[`restart: always`](https://github.com/nicwortel/local-development-proxy/blob/master/compose.yaml#L13).

### Installation (Podman)

> [!WARNING]
> Podman support in this repository is still experimental.

Rootless Podman requires a few one-time changes on the host.

Allow unprivileged processes to bind to port 80 (this makes all ports starting from 80 unprivileged):

```shell
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-unprivileged-ports.conf
sudo sysctl --system
```

Enable Podman's API socket. This is exposed to Traefik to discover containers and their labels, and the `docker-compose`
provider uses it to talk to Podman:

```shell
systemctl --user enable --now podman.socket
```

Enable the `podman-restart.service`, so that the container restarts after a reboot.

```shell
systemctl --user enable podman-restart.service
```

Then start the proxy, including the Podman override file which points Traefik at the Podman socket instead of the
Docker one:

```shell
git clone git@github.com:nicwortel/local-development-proxy.git
cd local-development-proxy
podman compose -f compose.yaml -f compose.podman.yaml up -d
```

> [!NOTE]
> If you also want containers to survive logout or to start before you log in, run `loginctl enable-linger $USER`.

### Usage

In order to connect with the reverse proxy and expose your application, add this to the `compose.yaml` of your
project:

```diff
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

### Domain name resolution

Traefik will use the `Host` HTTP header to route requests to the right container, but in order for your request to be
handled by Traefik you need to ensure that your host machine can resolve the application's domain name to 127.0.0.1.

There are a few ways to achieve this:

- There is a [draft RFC](https://datatracker.ietf.org/doc/html/draft-ietf-dnsop-let-localhost-be-localhost) to have
  any *.localhost domain resolve to 127.0.0.1.
  [systemd-resolved](https://manpages.ubuntu.com/manpages/bionic/man8/systemd-resolved.service.8.html) (used by Ubuntu
  Linux) and certain browsers (Chrome) already implement this and resolve all domain names ending with `.localhost` to
  127.0.0.1.
- You can use dnsmasq and route all domain names ending with `.localhost` to 127.0.0.1.
- You can manually add the domain name(s) to your hosts file.
