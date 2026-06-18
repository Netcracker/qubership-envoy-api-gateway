# Envoy API Gateway Helm Chart

The current Helm chart is based on the [https://github.com/envoyproxy/gateway](https://github.com/envoyproxy/gateway).

# Overview

The Gateway concept is described in the following page: [Gateway API](https://gateway-api.sigs.k8s.io/). Basically, it is a new generation of Kubernetes Ingress, Load Balancing, and Service Mesh. In comparing with Ingress controller, Gateway controller is more flexible, secure, and mature. The Gateway allows to manage HTTP, gRPC, TCP, and UDP endpoints. Envoy Gateway one of the Gateway API implementation. It dynamically creates the Kubernetes resources that send traffic to backend application.

* [Installation Guide](docs/public/Installation.md)
* [Troubleshooting Guide](docs/public/Troubleshooting.md)
