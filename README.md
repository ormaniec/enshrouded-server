# enshrouded-server

[![Static Badge](https://img.shields.io/badge/DockerHub-blue)](https://hub.docker.com/r/sknnr/enshrouded-dedicated-server) ![Docker Pulls](https://img.shields.io/docker/pulls/sknnr/enshrouded-dedicated-server) [![Static Badge](https://img.shields.io/badge/GitHub-green)](https://github.com/jsknnr/enshrouded-server) ![GitHub Repo stars](https://img.shields.io/github/stars/jsknnr/enshrouded-server)

Run Enshrouded dedicated server in a container. Optionally includes a Helm chart for running in Kubernetes.

**Disclaimer:** This is not an official image. No support, implied or otherwise, is offered to any end user by the author or anyone else. Feel free to do what you please with the contents of this repo.

## General Information

The processes within the container does **not** run as root. Everything runs as the user `steam` (uid:10000/gid:10000 by default). If you exec into the container, you will drop into `/home/steam` as the steam user. Enshrouded is installed to `/home/steam/enshrouded`.

Any persistent volumes should be mounted to `/home/steam/enshrouded/savegame` and be owned by `10000:10000`.

If you need to run the container process with a different uid/gid, you can build a custom image based on the included Dockerfile. See the [custom uid/gid instructions](https://github.com/jsknnr/enshrouded-server/issues/51) for details.

### Ports

| Port       | Protocol | Default |
| ---------- | -------- | ------- |
| Game Port  | UDP      | 15637   |
| Steam Port | UDP      | 27015   |

Both ports must be published and forwarded through your router for external connectivity.

### Environment Variables

| Name              | Description                                                    | Default                    | Required |
| ----------------- | -------------------------------------------------------------- | -------------------------- | -------- |
| `SERVER_NAME`     | Display name for the server                                    | `Enshrouded Containerized` | No       |
| `SERVER_PASSWORD` | Password required to join the server                           |  None                      | No       |
| `PORT`            | Game port                                                      | `15637`                    | No       |
| `STEAM_PORT`      | Port used for Steam queries                                    | `27015`                    | No       |
| `SERVER_SLOTS`    | Number of player slots (max 16)                                | `16`                       | No       |
| `SERVER_IP`       | IP address the server listens on                               | `0.0.0.0`                  | No       |
| `EXTERNAL_CONFIG` | Use a manually supplied config file instead of env vars (0/1)  | `0`                        | No       |

> Note:
>
> `SERVER_IP` is ignored when using Helm, as Kubernetes handles networking differently.

## Running with Docker

```bash
docker volume create enshrouded-persistent-data

docker run \
  --detach \
  --name enshrouded-server \
  --mount type=volume,source=enshrouded-persistent-data,target=/home/steam/enshrouded/savegame \
  --publish 15637:15637/udp \
  --publish 27015:27015/udp \
  --env=SERVER_NAME='Enshrouded Containerized Server' \
  --env=SERVER_PASSWORD='ChangeThisPlease' \
  --env=SERVER_SLOTS=16 \
  --env=PORT=15637 \
  --env=STEAM_PORT=27015 \
  --restart=unless-stopped \
  sknnr/enshrouded-dedicated-server:latest
```

## Running with Docker Compose

Clone this repo or copy the `compose.yaml` file from the `container` directory. Edit the environment variables to your liking, then run:

```bash
docker-compose up -d
```

To stop the server:

```bash
docker-compose down
```

### Standard Compose File

```yaml
services:
  enshrouded:
    image: sknnr/enshrouded-dedicated-server:latest
    ports:
      - "15637:15637/udp"
      - "27015:27015/udp"
    environment:
      - SERVER_NAME=Enshrouded Containerized
      - SERVER_PASSWORD=PleaseChangeMe
      - PORT=15637
      - STEAM_PORT=27015
      - SERVER_SLOTS=16
      - SERVER_IP=0.0.0.0
    volumes:
      - enshrouded-persistent-data:/home/steam/enshrouded/savegame
    restart: unless-stopped

volumes:
  enshrouded-persistent-data:
```

### External Config Compose File

If you prefer full control over the server configuration rather than using environment variables, you can mount your own config file. Set `EXTERNAL_CONFIG=1` and bind-mount your JSON config to replace the default.

> Note:
>
> If you are using custom difficulty settings, you must change `"gameSettingsPreset": "Default"` to
> `"gameSettingsPreset": "Custom"` in your JSON config file, otherwise your difficulty overrides will
> be ignored.

For the full list of available server settings and user group options, see the [official dedicated server configuration guide](https://enshrouded.zendesk.com/hc/en-us/articles/16055441447709-Dedicated-Server-Configuration).

```yaml
services:
  enshrouded:
    image: sknnr/enshrouded-dedicated-server:latest
    ports:
      - "15637:15637/udp"
      - "27015:27015/udp"
    environment:
      - EXTERNAL_CONFIG=1
    volumes:
      - enshrouded-persistent-data:/home/steam/enshrouded/savegame
      - ./enshrouded_server_external.json:/home/steam/enshrouded/enshrouded_server.json
    restart: unless-stopped

volumes:
  enshrouded-persistent-data:
    # Uncomment below to use a host bind mount instead of a Docker-managed volume.
    # Ensure the directory exists and is owned by 10000:10000.
    # driver: local
    # driver_opts:
    #   type: none
    #   device: /path/to/your/host/directory
    #   o: bind
```

## Running with Podman

```bash
podman volume create enshrouded-persistent-data

podman run \
  --detach \
  --name enshrouded-server \
  --mount type=volume,source=enshrouded-persistent-data,target=/home/steam/enshrouded/savegame \
  --publish 15637:15637/udp \
  --publish 27015:27015/udp \
  --env=SERVER_NAME='Enshrouded Containerized Server' \
  --env=SERVER_PASSWORD='ChangeThisPlease' \
  --env=SERVER_SLOTS=16 \
  --env=PORT=15637 \
  --env=STEAM_PORT=27015 \
  docker.io/sknnr/enshrouded-dedicated-server:latest
```

### Quadlet (Podman systemd integration)

To run the container as a systemd service using Podman's quadlet subsystem, create the file `/etc/containers/systemd/enshrouded.container` (when running as root):

```text
[Unit]
Description=Enshrouded Game Server

[Container]
Image=docker.io/sknnr/enshrouded-dedicated-server:latest
Volume=enshrouded-persistent-data:/home/steam/enshrouded/savegame
PublishPort=15637:15637/udp
PublishPort=27015:27015/udp
ContainerName=enshrouded-server
Environment=SERVER_NAME="Enshrouded Containerized Server"
Environment=SERVER_PASSWORD="ChangeThisPlease"
Environment=PORT=15637
Environment=STEAM_PORT=27015
Environment=SERVER_SLOTS=16

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target
```

## Running with Kubernetes (Helm)

A Helm chart is included in the `helm` directory of this repo and is also hosted in the [jsknnr helm-charts repository](https://jsknnr.github.io/helm-charts).

### Install from the Helm repository

```bash
helm repo add jsknnr https://jsknnr.github.io/helm-charts
helm repo update
```

```bash
helm install enshrouded jsknnr/enshrouded-dedicated-server \
  --namespace enshrouded \
  --create-namespace \
  --values myvalues.yaml
```

Where `myvalues.yaml` is your copy of `values.yaml` with any overrides applied. Be sure to create and specify a namespace, as the chart does not provision one by default.

## Troubleshooting

### Connectivity

If you cannot connect to the server after deployment, the issue is almost certainly with your network configuration, not this image. Verify the following:

1. The game port (default 15637) and Steam port (default 27015) are open on your firewall and container host.
2. Both ports are forwarded from your router to the private IP of the machine running the container.
3. Your ISP is not blocking the ports. Contact them if the above steps are confirmed correct and connectivity still fails.

For additional help, see [this discussion thread](https://github.com/jsknnr/enshrouded-server/issues/16) where several users debugged similar issues.

### Storage and Permissions

It is recommended to let Docker or Podman manage the persistent volume. If you must use a bind mount, the host directory must be owned by `10000:10000`:

```bash
chown -R 10000:10000 /path/to/your/directory
```

If ownership is incorrect, the container will fail to start because the server process cannot write savegame data.