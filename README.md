# k8s-autoheal-operator

<p align="center">
  <img src="https://raw.githubusercontent.com/kubernetes/kubernetes/master/logo/logo.svg" width="100" alt="Kubernetes Logo" />
</p>

<h1 align="center">Kubernetes Autoheal Operator</h1>

<p align="center">
  <em>A Kubernetes operator that automatically detects unhealthy pods and heals them by triggering rolling restarts of their parent Deployments.</em>
</p>

<p align="center">
  <a href="https://github.com/kanika-04/k8s-autoheal-operator">
    <img src="https://img.shields.io/github/stars/kanika-04/k8s-autoheal-operator?style=flat" alt="GitHub Stars" />
  </a>
  <a href="https://github.com/kanika-04/k8s-autoheal-operator">
    <img src="https://img.shields.io/github/last-commit/kanika-04/k8s-autoheal-operator" alt="Last Commit" />
  </a>
  <a href="https://github.com/kanika-04/k8s-autoheal-operator/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg" alt="Apache 2.0 License" />
  </a>
</p>

---

## Overview

**k8s-autoheal-operator** is a Kubernetes operator designed to improve workload reliability by detecting unhealthy pods and initiating recovery actions automatically.

When a pod enters a configured failure state, the operator identifies its parent Deployment through Kubernetes owner references and triggers a rolling restart by updating the Deployment's pod-template annotation.

This approach uses Kubernetes-native rollout behavior instead of manually deleting or recreating Deployments.

### Supported Failure States

- **CrashLoopBackOff** — A container repeatedly crashes and restarts.
- **OOMKilled** — A container is terminated after exceeding its memory limit.
- **Error** — A container exits with an error.

Detection and recovery depend on the operator's implemented logic and the state reported by Kubernetes.

## Architecture

```text
                 KUBERNETES CLUSTER
+------------------------------------------------------+
|                                                      |
|   +------------------+                               |
|   |   Pod Watcher    |                               |
|   |   (Informer)     |                               |
|   +--------+---------+                               |
|            |                                         |
|            v                                         |
|   +------------------+                               |
|   | Failure Detection|                               |
|   +--------+---------+                               |
|            |                                         |
|            v                                         |
|   +------------------+                               |
|   | Rate-Limited Queue|                              |
|   +--------+---------+                               |
|            |                                         |
|            v                                         |
|   +------------------+                               |
|   |   Healer Worker  |                               |
|   +--------+---------+                               |
|            |                                         |
|            v                                         |
|   +------------------+                               |
|   | Parent Deployment|                               |
|   |  Rolling Restart |                               |
|   +--------+---------+                               |
|            |                                         |
|            v                                         |
|      Kubernetes replaces pods                        |
|                                                      |
+------------------------------------------------------+
```

### Components

| Component | File | Responsibility |
|---|---|---|
| Cluster Connection | `internal/clusterConnect.go` | Establishes a connection to the Kubernetes cluster. |
| Pod Watcher | `internal/watcher.go` | Watches pod status changes and detects failure conditions. |
| Healer | `internal/healer.go` | Identifies parent Deployments and triggers recovery actions. |
| Manager | `cmd/main.go` | Starts the operator and manages its configured services. |

## Features

- **Real-time monitoring:** Uses Kubernetes informers for event-driven pod monitoring.
- **Automated recovery:** Triggers rolling restarts when configured failure conditions are met.
- **Deployment-aware healing:** Traces failed pods to their parent Deployments.
- **Duplicate prevention:** Helps prevent repeated restart requests for the same workload.
- **Rate-limited processing:** Controls the processing of recovery requests.
- **Health probes:** Supports health and readiness endpoints when configured.
- **Metrics:** Supports a metrics endpoint when enabled.
- **Leader election:** Supports coordinated operation in highly available deployments when configured.
- **Containerization:** Includes a Dockerfile for building a container image.

## Tech Stack

- **Language:** Go
- **Orchestration:** Kubernetes
- **Kubernetes integration:** Kubernetes API and controller-runtime, as used by the implementation
- **Containerization:** Docker
- **Build automation:** Make
- **Version control:** Git and GitHub

## Prerequisites

Before running the operator, install:

- Go version compatible with `go.mod` (the original project documentation specifies Go 1.24+).
- Docker, if you intend to build container images.
- `kubectl`.
- Access to a Kubernetes cluster.
- A valid Kubernetes configuration for local development.

For a local cluster, you can use [Kind](https://kind.sigs.k8s.io/) or [Minikube](https://minikube.sigs.k8s.io/).

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/kanika-04/k8s-autoheal-operator.git
cd k8s-autoheal-operator
```

### 2. Download dependencies

```bash
go mod download
```

### 3. Verify Kubernetes connectivity

```bash
kubectl cluster-info
```

This command checks whether your configured Kubernetes cluster is reachable.

### 4. Build the project

```bash
go build ./...
```

If your Makefile provides a `build` target, you can also use:

```bash
make build
```

### 5. Run tests

```bash
go test ./...
```

If the project defines additional Makefile targets, consult the Makefile for the supported test commands.

## Running Locally

To run the operator outside a Kubernetes cluster, ensure your kubeconfig points to a working cluster and that your account has the required permissions.

```bash
make run
```

This command assumes that the project defines a `run` target in its Makefile.

## Deploying to Kubernetes

If the project provides the required deployment manifests and Makefile targets, build and push an image to a container registry:

```bash
make docker-build docker-push IMG=<your-registry>/k8s-autoheal-operator:latest
```

Then deploy the operator:

```bash
make deploy IMG=<your-registry>/k8s-autoheal-operator:latest
```

Replace `<your-registry>` with your actual container registry.

**Note:** These commands depend on the targets defined in your Makefile and the deployment configuration included in the repository.

## Verifying the Deployment

If the deployment manifests use the standard Kubebuilder namespace, check the operator pod:

```bash
kubectl get pods -n k8s-autoheal-operator-system
```

View the operator logs:

```bash
kubectl logs -n k8s-autoheal-operator-system \
  -l control-plane=controller-manager \
  --follow
```

The namespace and label selector may differ if your deployment configuration uses custom values.

## Testing the Auto-Healing Workflow

You can test the recovery workflow in a development cluster using a disposable workload that repeatedly fails.

For example:

```bash
kubectl create deployment crasher \
  --image=busybox \
  -- /bin/sh -c "exit 1"
```

Observe the pod state:

```bash
kubectl get pods -w
```

In another terminal, observe the operator logs:

```bash
kubectl logs -n k8s-autoheal-operator-system \
  -l control-plane=controller-manager \
  --follow
```

The operator should initiate recovery only if the failure matches its detection logic, the workload is supported, and the operator has sufficient permissions.

Clean up the test workload when finished:

```bash
kubectl delete deployment crasher
```

## Configuration

The operator's available configuration depends on its implementation and startup flags.

Common settings in Kubebuilder-based projects include:

| Flag | Typical purpose |
|---|---|
| `--metrics-bind-address` | Configures the metrics endpoint. |
| `--health-probe-bind-address` | Configures health and readiness probes. |
| `--leader-elect` | Enables leader election for highly available deployments. |
| `--metrics-secure` | Controls secure serving of metrics. |

Check the project's startup code or run command help to confirm which flags are supported and their actual default values.

## Development

### Build

```bash
make build
```

### Test

```bash
make test
```

### Lint

```bash
make lint
```

### Build a Docker image

```bash
make docker-build IMG=k8s-autoheal-operator:dev
```

These commands require the corresponding targets to be defined in the project's Makefile.

## Security Considerations

- Follow the principle of least privilege when configuring Kubernetes RBAC.
- Never commit passwords, access tokens, private keys, or sensitive cluster configuration.
- Test automatic recovery in a development cluster before using it in production.
- Review restart behavior and safeguards to avoid repeated recovery loops.
- Use appropriate resource limits and monitoring for the operator itself.

## Contributing

Contributions, bug reports, and suggestions are welcome.

1. Fork the repository.
2. Create a feature branch.
3. Implement your changes and add tests.
4. Run the available build and test commands.
5. Submit a pull request describing your changes.

## License

This project references the Apache License 2.0. See the [`LICENSE`](LICENSE) file in the repository for the applicable license terms.

## Author

**Kanika Agrawal**

- GitHub: [@kanika-04](https://github.com/kanika-04)
- Project: [k8s-autoheal-operator](https://github.com/kanika-04/k8s-autoheal-operator)
