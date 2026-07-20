---
type: Quickstart
title: Slurm-in-Docker Quickstart
description: "Getting-started guide for the Slurm-in-Docker project: a containerized SLURM workload manager cluster for development and testing. Covers build, run, configuration, SLURM commands, and known issues."
resource: https://github.com/SciDAS/slurm-in-docker
tags: [quickstart, slurm, docker, workload-manager, cluster]
---

# Slurm-in-Docker Quickstart

**Slurm-in-Docker** packages the [SLURM](https://slurm.schedmd.com) workload manager into a Docker Compose cluster for development and testing. The project ships **SLURM 19.05.1**, **OpenMPI 3.0.1**, and the **Lmod 7.7** module system, all running on CentOS 7-based images.

The project is marked **WORK IN PROGRESS** and was originally built to learn SLURM basics before extending to more distributed environments. The last code commit was October 2019.

## Cluster Topology

The cluster consists of four containers on a Docker bridge network:

| Container | Function | FQDN | Image |
|---|---|---|---|
| `controller` | SLURM primary controller (slurmctld) | controller.local.dev | scidas/slurm.controller:19.05.1 |
| `database` | SLURM accounting daemon (slurmdbd) + MariaDB | database.local.dev | scidas/slurm.database:19.05.1 |
| `worker01` | SLURM compute node (slurmd) | worker01.local.dev | scidas/slurm.worker:19.05.1 |
| `worker02` | SLURM compute node (slurmd) | worker02.local.dev | scidas/slurm.worker:19.05.1 |

The controller generates configuration and authentication keys, which are shared to other containers via Docker bind mounts (`/.secret` and `/home`).

See [Architecture](architecture.md) for the full container lifecycle and data flow.

## Prerequisites

- Docker Engine + Docker Compose
- CentOS 7-compatible host (images are based on CentOS 7)
- Sufficient RAM: each container needs ~2GB (4 containers × 2GB ≈ 8GB minimum)

## Build

### 1. Build RPM packages

The base image depends on SLURM and OpenMPI RPMs built from source:

```bash
make packages
# or manually:
cd packages/centos-7
docker build -t scidas/slurm.packages:19.05.1 .
docker run --rm -v $(pwd)/rpms:/packages scidas/slurm.packages:19.05.1
```

### 2. Build images in dependency order

The Makefile system handles build order: `packages` → `base` → `controller` | `worker` | `database`.

```bash
make build
```

The base image copies RPMs from `packages/centos-7/rpms` into the Docker build context.

### 3. Verify images

```console
$ docker images | grep slurm
scidas/slurm.packages   19.05.1   ...
scidas/slurm.base       19.05.1   ...
scidas/slurm.controller 19.05.1   ...
scidas/slurm.worker     19.05.1   ...
scidas/slurm.database   19.05.1   ...
```

## Run

Start the cluster with Docker Compose:

```bash
docker-compose up -d
```

All four containers should be running within ~30 seconds:

```console
$ docker ps
CONTAINER ID   IMAGE                             STATUS   NAMES
...            scidas/slurm.worker:19.05.1       Up 30s   worker01
...            scidas/slurm.database:19.05.1     Up 30s   database
...            scidas/slurm.worker:19.05.1       Up 30s   worker02
...            scidas/slurm.controller:19.05.1   Up 31s   controller
```

## Use SLURM Commands

Enter the controller container to issue SLURM commands:

```bash
docker exec -ti controller /bin/bash
```

### Check cluster status

```bash
sinfo -lN
```

Expected output:
```
NODELIST  NODES  PARTITION  STATE   CPUS  MEMORY
worker01      1   docker*     idle     1    1998
worker02      1   docker*     idle     1    1998
```

### Set up accounting (one-time)

```bash
sacctmgr -i add account worker description="worker account" Organization=Slurm-in-Docker
sacctmgr -i create user worker account=worker adminlevel=None
```

### Run a job

```bash
# Direct execution across both workers
srun -N 2 -u worker hostname

# Batch job
sbatch --wrap "hostname" -u worker
```

### Query accounting database

```bash
docker exec -ti database mysql -uslurm -ppassword -hdatabase.local.dev
```

```sql
use slurm_acct_db;
show tables;
```

## Configuration

By default, SLURM configuration files are generated at runtime from environment variables in `docker-compose.yml`. Users can provide custom `slurm.conf` and `slurmdbd.conf` by placing them in `home/config/`:

```bash
mkdir -p home/config secret
cp my-slurm.conf home/config/slurm.conf
cp my-slurmdbd.conf home/config/slurmdbd.conf
```

See [Configuration](configuration.md) for full details on environment variables and configuration options.

## Teardown

```bash
docker-compose down
# or use the provided script:
bash teardown.sh
```

## Known Issues

- **MariaDB resolveip** ([issue #26](https://github.com/SciDAS/slurm-in-docker/issues/26)): MariaDB expects `resolveip` under `/usr/libexec/`. Fixed via a symlink in `database/docker-entrypoint.sh`.
- **SLURM version**: The project ships SLURM 19.05.1 (from August 2019). This is no longer the current SLURM release.
- **ControlMachine** was removed as a configuration option in newer SLURM versions; the entrypoint script marks it as defunct.
- **No release tags**: The repository has no git tags; all builds use the `19.05.1` tag convention.

## Backlog

Areas not yet documented:

- **Lmod integration** — Lmod 7.7 module system support for extending software (Java, Git, Nextflow, iRODS). Source: `using-lmod-with-slurm-in-docker.md`, `docker-compose-lmod.yml`. Deferred because Lmod is optional and has its own tutorial file.
- **Swarm + GlusterFS deployment** — Extended deployment guide for Docker Swarm with GlusterFS shared storage. Source: `slurm-cluster-swarm-glusterfs.md`. Deferred because this is an advanced, optional deployment path.
- **iRODS integration example** — iRODS data management integration with SLURM modules. Source: `lmod-irods-slurm-example.md`. Deferred as a specialized use case.