# Jenkins

A single-container Jenkins controller run with Docker Compose. It sits behind
[Traefik](https://traefik.io/), which serves the web UI over HTTPS and passes
the inbound agent port (50000) through as plain TCP.

## Requirements

- Docker with the Compose plugin
- A Traefik instance that:
  - has a `websecure` entrypoint with a default certificate covering `DOMAIN`
    (for example a wildcard)
  - has a `tcp50000` entrypoint for inbound agents
  - is attached to the `jenkins` Docker network (this stack creates it)
- A DNS record for `DOMAIN` that points to the Traefik host

## Setup

```bash
cp .env.example .env
chmod 600 .env
$EDITOR .env          # set JENKINS_TAG, DOMAIN and the allowlists
docker compose up -d
```

Wait for the container to report healthy, then get the initial admin password:

```bash
docker compose ps
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open `https://$DOMAIN` and finish the setup wizard.

## Configuration

All settings are read from `.env`.

| Variable                     | Default                | Description                                                     |
|------------------------------|------------------------|-----------------------------------------------------------------|
| `JENKINS_TAG`                | `2.580.1-alpine-jdk25` | Image tag of `jenkins/jenkins`. Pin an exact LTS release.       |
| `DOMAIN`                     | **required**           | Hostname Traefik routes to Jenkins.                             |
| `IPV4_ALLOWLIST`             | `0.0.0.0/0`            | IPv4 ranges allowed to reach the UI and agent port, comma-separated. |
| `IPV6_ALLOWLIST`             | `::/0`                 | IPv6 ranges allowed to reach the UI and agent port, comma-separated. |
| `TZ`                         | `Europe/Athens`        | Container time zone.                                            |
| `JENKINS_HEAP_PERCENT`       | `60`                   | JVM max heap as a percentage of the memory limit.               |
| `JENKINS_CPU_LIMIT`          | `2`                    | CPU limit.                                                      |
| `JENKINS_MEMORY_LIMIT`       | `2G`                   | Memory limit.                                                   |
| `JENKINS_MEMORY_RESERVATION` | `1G`                   | Memory reservation.                                             |
| `JENKINS_PIDS_LIMIT`         | `2048`                 | Process/thread limit. Jenkins uses many threads, so keep this high. |

The default allowlists let every address in. Narrow them before you expose the
instance.

## How it's set up

- **Hardening:** the container drops all Linux capabilities and runs with
  `no-new-privileges`.
- **JVM:** G1GC is set explicitly. The heap is capped at `JENKINS_HEAP_PERCENT`
  of the container's memory limit, which leaves room for metaspace, threads and
  native memory.
- **Health check:** `wget` polls `/health` every 30s, with a 180s start period.
- **Logs:** the json-file driver rotates and compresses them (5 × 10 MB).
- **Data:** everything is stored in the `jenkins-home` named volume, mounted at
  `/var/jenkins_home`.

## Operations

```bash
docker compose logs -f jenkins     # follow logs
docker compose restart jenkins     # restart
docker compose down                # stop (the volume is kept)
```

**Upgrade:** set a new `JENKINS_TAG` in `.env`, then run:

```bash
docker compose pull && docker compose up -d
```

**Back up** the home volume:

```bash
docker run --rm -v jenkins-home:/data -v "$PWD":/backup alpine \
  tar czf /backup/jenkins-home-$(date +%F).tar.gz -C /data .
```

**Restore** a backup into the volume:

```bash
docker compose down
docker run --rm -v jenkins-home:/data -v "$PWD":/backup alpine \
  sh -c 'rm -rf /data/* && tar xzf /backup/jenkins-home-YYYY-MM-DD.tar.gz -C /data'
docker compose up -d
```
