---
type: Architecture Documentation
title: Slurm-in-Docker Architecture
description: "Container architecture for the SLURM workload manager cluster: image hierarchy, startup sequence, authentication flow, and inter-container communication patterns."
tags: [architecture, docker, slurm, container, munger, authentication]
---

# Architecture

Slurm-in-Docker implements a **four-container SLURM cluster** using a layered Docker image hierarchy and shared volumes for configuration and authentication.

## Image Hierarchy

All images derive from a common base that provides SLURM binaries and system dependencies:

```
packages/centos-7 (RPM builder)
       ↓ produces RPMs
scidas/slurm.base:19.05.1 (base image with SLURM, OpenMPI, MUNGE, MariaDB)
       ↓ extends
├── scidas/slurm.controller:19.05.1 (head-node: slurmctld)
├── scidas/slurm.worker:19.05.1 (compute-nodes: slurmd)
└── scidas/slurm.database:19.05.1 (accounting: slurmdbd + MariaDB)
```

- **Base image** (`base/Dockerfile`): Installs SLURM 19.05.1, OpenMPI 3.0.1, Lmod 7.7, MUNGE, MariaDB, and SSH from RPMs. Creates system users: `munge` (UID 981), `slurm` (UID 982), `worker` (UID 1000).
- **Controller image** (`controller/Dockerfile`): Runs `slurmctld` and generates SLURM configuration at startup.
- **Worker image** (`worker/Dockerfile`): Runs `slurmd` and waits for configuration from controller.
- **Database image** (`database/Dockerfile`): Runs `slurmdbd` and MariaDB for job accounting.

## Startup Sequence

Containers start asynchronously but coordinate via shared volumes and polling:

### 1. Controller (bootstrap)
Source: `controller/docker-entrypoint.sh`

1. Starts SSH daemon (`_sshd_host`)
2. Sets up passwordless SSH for the worker user (`_ssh_worker`)
3. Generates MUNGE authentication key (`_munge_start`)
4. Copies keys and SSH config to `/.secret/` volume (`_copy_secrets`)
5. Generates `slurm.conf` from environment variables (`_generate_slurm_conf`)
6. Starts `slurmctld` (SLURM controller daemon)

### 2. Database (depends on controller)
Source: `database/docker-entrypoint.sh`

1. Starts SSH daemon
2. Initializes MariaDB (creates symlink for `resolveip` fix)
3. Polls `/.secret/` for MUNGE key from controller (`_munge_start_using_key`)
4. Polls `/home/` for worker SSH key (`_wait_for_worker`)
5. Generates `slurmdbd.conf` from environment variables (`_generate_slurmdbd_conf`)
6. Starts `slurmdbd` (SLURM database daemon)

### 3. Workers (depend on controller)
Source: `worker/docker-entrypoint.sh`

1. Starts SSH daemon
2. Polls `/.secret/` for MUNGE key from controller (`_munge_start_using_key`)
3. Polls `/home/` for worker SSH key (`_wait_for_worker`)
4. Polls `/.secret/` for `slurm.conf` from controller (`_slurmd`)
5. Starts `slurmd` (SLURM compute daemon)

## Authentication

All inter-container communication uses **MUNGE** authentication:

- The **controller generates** a MUNGE key via `create-munge-key -f` and starts the MUNGE daemon.
- Other containers **poll for** the key file at `/.secret/munge.key`, then copy it to `/etc/munge/munge.key`.
- MUNGE tokens authenticate all SLURM daemon communication (slurmctld ↔ slurmd, slurmctld ↔ slurmdbd).

## Networking and Volumes

### Docker Bridge Network
All containers join the `slurm` bridge network defined in `docker-compose.yml`. Containers communicate via Docker DNS using hostnames (controller.local.dev, database.local.dev, etc.).

### Shared Volumes
| Mount | Source | Purpose |
|---|---|---|
| `/home` | `./home` | Shared user home directories across containers |
| `/.secret` | `./secret` | Authentication keys and configuration files |

The `/.secret` volume is the **primary coordination mechanism** — the controller writes MUNGE keys, SSH keys, and slurm.conf here, and other containers poll for these files before starting their daemons.

## Ports

All containers expose these ports:
- **22**: SSH
- **3306**: MariaDB (database container)
- **6817**: SLURM controller (slurmctld)
- **6818**: SLURM compute (slurmd)
- **6819**: SLURM database (slurmdbd)

See [Configuration](configuration.md) for how port values map to environment variables and SLURM configuration files. See [Testing](testing.md) for how the test suite validates this startup sequence.