# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

A Helm chart distribution for deploying **Envoy Gateway** on Kubernetes clusters (1.29+). It wraps the upstream [envoyproxy/gateway](https://github.com/envoyproxy/gateway) project and adds production-ready defaults, resource profiles, and opinionated custom resources.

There is no application source code — this repo is purely Helm charts, CI/CD workflows, and a minimal Docker image used to distribute chart artifacts.

## Two-Chart Architecture

The deployment requires two interdependent charts installed in order:

1. **`charts/envoy-gateway/`** — installs the Envoy Gateway controller and CRDs. Wraps the community `gateway-helm` chart (pinned in `Chart.yaml`) and configures it via `values.yaml`.

2. **`charts/envoy-gateway-cr/`** — installs custom Kubernetes resources on top of the running controller: `GatewayClass`, `Gateway`, `EnvoyProxy`, and `ClientTrafficPolicy` objects. All meaningful configuration lives in `values.yaml` here.

The `envoy-gateway-cr` chart contains a pre-upgrade Job (`templates/pre-upgrade/`) that patches the gateway namespace with `pod-security.kubernetes.io/enforce=privileged` when hostPorts are enabled. This runs as a Helm hook before install/upgrade.

## Build & Release

There is no local build tool (no Makefile, npm, gradle). The entire release pipeline runs in GitHub Actions:

- **`.github/workflows/push.yml`** — on every push, builds and pushes the `qubership-envoy-api-gateway-transfer` Docker image (linux/amd64 + linux/arm64) to ghcr.io. This is a scratch image that bundles chart `.tgz` files and docs for artifact distribution.
- **`.github/workflows/helm-charts-release.yaml`** — manually triggered release: updates chart versions in `Chart.yaml`/`values.yaml`, packages charts, builds the transfer image, creates a GitHub release.
- **Release configuration**: `.qubership/helm-charts-release-config.yaml` maps chart names to values files; `.qubership/docker-build-config.cfg` defines image components and security scanning.

## Linting

Linting is enforced via GitHub Actions (`.github/workflows/super-linter.yaml`) using `super-linter`. There are no local lint commands, but the configuration can be used to run yamllint locally:

```bash
yamllint -c .github/linters/.yaml-lint.yml charts/
```

Key rules: 2-space YAML indentation, no line length limit, spaces inside braces/brackets.

Editor standards (`.editorconfig`): UTF-8, LF line endings, 4-space indent for shell and Dockerfile.

## Validation After Install

There are no automated tests. Post-install verification is done with kubectl:

```bash
kubectl get crd | grep gateway
kubectl -n gateway-system get pod
kubectl get gatewayclasses
kubectl get gateways.gateway.networking.k8s.io -A
kubectl get envoyproxy -A
kubectl get clienttrafficpolicy -A
```

## Key Configuration Areas in `envoy-gateway-cr/values.yaml`

- **`gatewayClass`** — defines `internal` (ClusterIP) and `external` (LoadBalancer) gateway classes
- **`gateway`** — configures listeners (HTTP on 80, HTTPS on 443, TCP/UDP/TLS); `daemonset: true` switches EnvoyProxy from Deployment to DaemonSet
- **`proxy`** — resource limits/requests, logging levels, telemetry, topology spread constraints
- **`clientTrafficPolicy`** — proxy protocol, HTTP/1 underscore header handling, connection timeouts
- **`resourceProfiles`** — predefined memory/CPU presets (`dev`, `small`, `medium`, `large`, `prod`) referenced from `charts/envoy-gateway/resource-profiles/`

## Notable Constraints

- Requires Kubernetes 1.29+ and cluster-admin permissions for install
- OCP 4.19+ is **not supported** (upstream Envoy Gateway removal issue)
- AWS ALB integration requires setting `service.type: ClusterIP` plus an Ingress resource — see README for full config
- The two charts must be installed in sequence; `envoy-gateway-cr` will fail if the CRDs from `envoy-gateway` are not yet present

## Documentation Update

Each time the @charts/envoy-gateway/values.yaml and @charts/envoy-gateway-cr/values.yaml have changed, update the documentation accordingly.

## Schema update

Each time the @charts/envoy-gateway/values.yaml and @charts/envoy-gateway-cr/values.yaml have changed, the @charts/envoy-gateway/values.schema.json and @charts/envoy-gateway-cr/values.schema.json must be updated accordingly.

## Chart Test

Changes in chart must be tested by `helm template`. The fist run with default values.yaml the second must include changes in the following fields:
* `defaultGateways.external.tcp`
* `defaultGateways.external.udp`
* `defaultGateways.external.tls`
* `gatewayClasses.internal.envoyDeployment.daemonset`
* `gatewayClasses.external.envoyDeployment.daemonset`
* `defaultGateways.external.hostPorts`
