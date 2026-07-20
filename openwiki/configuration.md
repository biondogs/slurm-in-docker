---
type: Configuration Guide
title: Slurm-in-Docker Configuration
description: "Configuration management for the SLURM cluster: environment variables, slurm.conf generation, slurmdbd.conf, user-provided configurations, and Lmod module system."
tags: [configuration, slurm, environment, lmod, slurm.conf]
---

# Configuration

Slurm-in-Docker supports two configuration approaches: **environment-driven auto-generation** (default) and **user-provided configuration files**.

## Environment-Driven Configuration (Default)

When no custom configuration files are provided, each container generates its SLURM configuration at startup from environment variables defined in `docker-compose.yml`.

### Controller Environment Variables
Source: `controller/docker-entrypoint.sh` → `_generate_slurm_conf()`

| Variable | Default | SLURM Config Key |
|---|---|---|
| `CLUSTER_NAME` | snowflake | ClusterName |
| `CONTROL_MACHINE` | controller | SlurmctldHost |
| `SLURMCTLD_PORT` | 6817 | SlurmctldPort |
| `SLURMD_PORT` | 6818 | SlurmdPort |
| `COMPUTE_NODES` | worker01 worker02 | NodeName |
| `PARTITION_NAME` | docker | PartitionName |
| `USE_SLURMDBD` | true | AccountingStorageType |
| `ACCOUNTING_STORAGE_HOST` | database | AccountingStorageHost |
| `ACCOUNTING_STORAGE_PORT` | 6819 | AccountingStoragePort |

### Database Environment Variables
Source: `database/docker-entrypoint.sh` → `_generate_slurmdbd_conf()`

| Variable | Default | SLURM Config Key |
|---|---|---|
| `DBD_ADDR` | database | DbdAddr |
| `DBD_HOST` | database | DbdHost |
| `DBD_PORT` | 6819 | DbdPort |
| `STORAGE_HOST` | database.local.dev | StorageHost |
| `STORAGE_PORT` | 3306 | StoragePort |
| `STORAGE_USER` | slurm | StorageUser |
| `STORAGE_PASS` | password | StoragePass |

### Worker Environment Variables
Source: `worker/docker-entrypoint.sh`

| Variable | Default | Purpose |
|---|---|---|
| `CONTROL_MACHINE` | controller | Locates controller for MUNGE key and slurm.conf |
| `ACCOUNTING_STORAGE_HOST` | database | Accounting database hostname |
| `COMPUTE_NODES` | worker01 worker02 | Node list for slurm.conf |

## User-Provided Configuration

To override auto-generated configuration with custom SLURM files:

```bash
mkdir -p home/config secret
cp my-slurm.conf home/config/slurm.conf
cp my-slurmdbd.conf home/config/slurmdbd.conf
```

The entrypoint scripts check for these files before generating defaults. If found, they use the user-provided files instead.

### Example Configuration Files
- `config/slurm.conf.example`: Reference SLURM controller configuration
- `config/slurmdbd.conf.example`: Reference SLURM database daemon configuration

## SLURM Configuration Reference

The generated `slurm.conf` includes these key settings:
- **AuthType=auth/munge**: MUNGE-based authentication
- **SchedulerType=sched/backfill**: Backfill scheduling algorithm
- **AccountingStorageType=accounting_storage/slurmdbd**: Database accounting
- **NodeName=worker[01-02]**: Worker node range specification

See [Architecture](architecture.md) for how configuration files are distributed between containers via the `/.secret` volume. See [Testing](testing.md) for how the test suite validates SLURM configuration through job execution tests.

## Lmod Module System

Lmod 7.7 is installed in all images and provides module-based software management. The `docker-compose-lmod.yml` variant mounts pre-compiled module binaries and Lua module definition scripts.

See `using-lmod-with-slurm-in-docker.md` for Lmod setup and `docker-compose-lmod.yml` for the Lmod-enabled compose configuration.