---
type: Testing Guide
title: Slurm-in-Docker Testing
description: "Test infrastructure for the SLURM cluster: test suite design, test cases for SLURM commands and MPI, Makefile build system, and CI/CD workflow."
tags: [testing, slurm, makefile, ci-cd, test-suite]
---

# Testing

The project includes a Bash test suite and a hierarchical Makefile system for building and testing the SLURM cluster.

## Test Suite

Source: `test/test.sh`

The test suite validates cluster functionality by executing SLURM commands inside the controller container and verifying expected outcomes.

### Test Cases

| Test Function | Description | SLURM Commands |
|---|---|---|
| `check_slurm_status` | Verifies all containers are running | `docker ps`, `sinfo` |
| `check_slurm_srun` | Tests direct job execution across workers | `srun -N 2 hostname` |
| `check_slurm_sbatch` | Tests batch job submission | `sbatch --wrap "hostname"` |
| `check_slurm_sbatch_array` | Tests array job submission | `sbatch --array` |
| `check_slurm_mpi` | Tests MPI job execution with OpenMPI/PMI2 | `mpirun hostname` |

### Test Helper Functions

- `polling()`: Waits for a condition with configurable timeout
- `drun()`: Executes commands inside a container as the worker user
- `is_contain_dead()`: Checks if a container has stopped
- `is_all_completed()`: Verifies all SLURM jobs reached completion state
- `has_tasks()`: Checks for pending jobs in the queue

### Running Tests

```bash
make test
# or directly:
bash test/test.sh
```

## Makefile System

The project uses a hierarchical Makefile system where the root Makefile delegates to subdirectory Makefiles:

### Root Makefile
```makefile
subdir = packages base controller worker database

.PHONY: all build clean lint test $(subdir)

all: build
build: $(subdir)
clean: $(subdir)
test:
	$(MAKE) -C $@
lint:
	shellcheck **/*.sh
```

### Build Dependencies
```
packages → base → controller | worker | database
```

- **packages**: Builds SLURM and OpenMPI RPMs from source
- **base**: Copies RPMs and builds base image
- **controller/worker/database**: Extend base image with component-specific configuration

### Sub-Makefile Pattern
Each component follows the same pattern:
```makefile
all: build
build:
	docker build -t scidas/slurm.<component>:19.05.1 .
clean:
	docker rmi scidas/slurm.<component>:19.05.1
```

## Linting

Shell scripts are linted with `shellcheck`:

```bash
make lint
# Runs: shellcheck **/*.sh
```

## CI/CD

Source: `.github/workflows/openwiki-update.yml`

- **Trigger**: Manual dispatch or daily at 08:00 UTC
- **Purpose**: Updates OpenWiki documentation automatically
- **Provider**: OpenRouter AI via OpenWiki CLI

The repository has no automated test CI; tests must be run manually in a Docker-enabled environment.

The test suite depends on the [environment-driven configuration](configuration.md) to provide valid SLURM settings, and validates the [container startup sequence](architecture.md) that establishes MUNGE authentication and configuration distribution.

See [Architecture](architecture.md) for the container startup sequence that tests validate.