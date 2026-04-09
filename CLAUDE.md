# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SwarmKit is a distributed orchestration toolkit for Docker Swarm, written in Go. It provides Raft-based consensus, mutual TLS communication, task scheduling, and service orchestration. The module path is `github.com/moby/swarmkit/v2`.

## Build & Development Commands

**Prerequisites:** Go 1.24+. Run `make setup` once to install golangci-lint and protobuild.

```bash
make binaries          # Build all binaries (bin/ and swarmd/bin/)
make test              # Unit tests (excludes integration)
make integration-tests # Integration tests
make swarmd-tests      # Swarmd-specific tests (cd swarmd && go test ./...)
make check             # Lint (golangci-lint + proto formatting)
make generate          # Regenerate protobufs and go generate
make all               # check + binaries + test + integration-tests + swarmd-tests
```

**Run a single test:**
```bash
go test -run TestName -v ./path/to/package/
# For swarmd packages:
cd swarmd && go test -run TestName -v ./path/to/package/
```

Test flags: `-parallel 8 --timeout=20m -race` (race on amd64 only).

**Containerized builds:** Set `DOCKER_SWARMKIT_USE_CONTAINER=1` to run all make targets inside Docker.

## Proto Generation

Proto files live in `api/`. The custom protoc plugin `protoc-gen-gogoswarm` generates deepcopy, storeobject, raftproxy, and authenticatedwrapper code. Configuration is in `Protobuild.toml`.

```bash
make protos       # Run protobuild
make generate     # protos + go generate
make checkprotos  # Verify no uncommitted .pb.go changes
```

Proto formatting rules: indent with tabs only; Meta fields must have `(gogoproto.nullable) = false`.

## Architecture

**Two node roles:**
- **Manager**: Runs Raft consensus, orchestrators, scheduler, dispatcher, control API (gRPC), and CA server. Only the Raft leader makes scheduling decisions.
- **Worker (Agent)**: Connects to manager via dispatcher, executes assigned tasks, reports status.

Nodes can be both manager and worker simultaneously.

**Request flow:** Control API → Orchestrator (reconciliation) → Scheduler (placement) → Dispatcher (assignment) → Agent → Executor → Container

**Key packages:**
- `manager/` — Control plane: `controlapi/` (gRPC), `dispatcher/` (task dispatch), `scheduler/` (placement), `state/raft/` (Raft consensus over etcd/raft), `state/store/` (in-memory state), `orchestrator/` (replicated/global/jobs reconciliation)
- `agent/` — Worker: session management, task execution via `exec/` interface
- `ca/` — Certificate authority, mTLS, automatic cert rotation
- `api/` — All protobuf definitions and generated code
- `node/` — Combines manager + agent lifecycle
- `protobuf/plugin/` — Custom protoc plugin code (deepcopy, raftproxy, storeobject, authenticatedwrapper)

**State management:** In-memory store backed by Raft logs (BoltDB). All state mutations go through Raft consensus. Watch API provides real-time change subscriptions.

**Binaries:**
- `swarmd` — Main daemon (manager and/or worker)
- `swarmctl` — CLI for cluster management (uses `SWARM_SOCKET` env var, default `/var/run/swarmd.sock`)
- `swarm-rafttool` — Raft state inspection
- `swarm-bench` — Benchmarking

## Two Go Modules

The repo contains two Go modules: the root module (`github.com/moby/swarmkit/v2`) and `swarmd/` (separate go.mod). The swarmd binaries (`swarmd`, `swarmctl`, `swarm-rafttool`) are built from `swarmd/cmd/`. When working on swarmd code, tests must be run from the `swarmd/` directory.

## Linting

Configured in `.golangci.yml`. Key enabled linters: misspell, ineffassign, revive, unconvert, unused, govet (with nilness). Generated `*.pb.go` files are excluded. Lint runs on both root and `swarmd/` directories.

## Design Documents

The `design/` directory contains architecture docs: raft.md, scheduler.md, orchestrators.md, task_model.md, store.md, raft_encryption.md, topology.md, generic_resources.md, and TLA+ specs.
