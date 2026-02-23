This guide provides the installation information for Envoy API Gateway.

# Envoy Gateway Helm Chart

The current Helm chart is based on the [https://github.com/envoyproxy/gateway](https://github.com/envoyproxy/gateway).

## Overview

The Gateway concept is described in the following page: [Gateway API](https://gateway-api.sigs.k8s.io/). Basically, it is a new generation of Kubernetes Ingress, Load Balancing, and Service Mesh. In comparing with Ingress controller, Gateway controller is more flexible, secure, and mature. The Gateway allows to manage HTTP, gRPC, TCP, and UDP endpoints. Envoy Gateway one of the Gateway API implementation. It dynamically creates the Kubernetes resources that send traffic to backend application. Therefore, it makes impossible to use Envoy Gateway in the production solution with current NC on-premises Kubernetes or OCP4 clusters, because of their Load Balancers that could be managed only manually.

## Installation Prerequisites

The installation prerequisites are listed below.

* Kubernetes cluster version 1.29+
* Cluster admin permissions

**Warning**: Currently, OCP4 is not supported due to [the changes in OCP4.19](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/updating_clusters/index#nw-ingress-gateway-api-manage-succession_updating-cluster-prepare). The Envoy Gateway usage on OCP4 previous to 4.19 leads to complete Envoy Gateway removal during the upgrade from 4.18 to 4.19.

## Charts Structure

The current Helm Chart consists of two separate charts. The first one is a community Envoy Gateway, that manages only Envoy Gateway controller. The second chart is a set of NC default custom resources: `EnvoyProxies`, `GatewayClasses`, `Gateways`, and so on. The folder structure is as follows:

```
charts/
├── envoy-gateway
│   ├── charts
│   │   └── gateway-helm
│   │       ├── crds
│   │       │   └── generated
│   │       └── templates
│   ├── resource-profiles
│   └── templates
└── envoy-gateway-cr
    ├── resource-profiles
    └── templates
```

The `envoy-gateway` chart must be installed at first. Only after that it's possible to install the `envoy-gateway-cr` chart. Both charts should be installed in the same namespace.

## Installation via Helm

The installation procedure via Helm is given below.

1. Create namespace for Envoy Gateway, for example:

```sh
kubectl create ns envoy-gateway
```

3. Navigate in `envoy-gateway` directory and change the configuration in `values.yaml` as it is needed. Full values options list is described in [Values](#envoy-gateway-chart-values) section.
4. Run the installation of the Envoy Gateway chart:

```sh
helm install envoy-gateway . -n envoy-gateway
```

5. Navigate in `envoy-gateway-cr` directory and change the configuration in `values.yaml` as it is needed. Full values options list is described in [CR Values](#envoy-gateway-cr-chart-values) section.
6. Run the installation of the Custom Resources chart:

```sh
helm install envoy-gateway-cr . -n envoy-gateway
```

### `envoy-gateway` Chart Values

The `envoy-gateway` values are specified in the below table.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| certgen | object | `{"job":{"affinity":{},"annotations":{},"args":[],"nodeSelector":{},"pod":{"annotations":{},"labels":{}},"resources":{},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":65532,"runAsNonRoot":true,"runAsUser":65532,"seccompProfile":{"type":"RuntimeDefault"}},"tolerations":[],"ttlSecondsAfterFinished":30},"rbac":{"annotations":{},"labels":{}}}` | Certgen is used to generate the certificates required by EnvoyGateway. If you want to construct a custom certificate, you can generate a custom certificate through Cert-Manager before installing EnvoyGateway. Certgen will not overwrite the custom certificate. Please do not manually modify `values.yaml` to disable certgen, it may cause issues in the expected working of EnvoyGateway OIDC, OAuth2, and so on. |
| config.envoyGateway | object | `{"extensionApis":{},"gateway":{"controllerName":"gateway.envoyproxy.io/gatewayclass-controller"},"logging":{"level":{"default":"info"}},"provider":{"type":"Kubernetes"}}` | EnvoyGateway configuration. Visit https://gateway.envoyproxy.io/docs/api/extension_types/#envoygateway to view all options. |
| createNamespace | bool | `false` |  |
| deployment.annotations | object | `{}` |  |
| deployment.envoyGateway.image.repository | string | `""` |  |
| deployment.envoyGateway.image.tag | string | `""` |  |
| deployment.envoyGateway.imagePullPolicy | string | `""` |  |
| deployment.envoyGateway.imagePullSecrets | list | `[]` |  |
| deployment.envoyGateway.resources.limits.memory | string | `"1024Mi"` |  |
| deployment.envoyGateway.resources.requests.cpu | string | `"100m"` |  |
| deployment.envoyGateway.resources.requests.memory | string | `"256Mi"` |  |
| deployment.envoyGateway.securityContext.allowPrivilegeEscalation | bool | `false` |  |
| deployment.envoyGateway.securityContext.capabilities.drop[0] | string | `"ALL"` |  |
| deployment.envoyGateway.securityContext.privileged | bool | `false` |  |
| deployment.envoyGateway.securityContext.runAsGroup | int | `65532` |  |
| deployment.envoyGateway.securityContext.runAsNonRoot | bool | `true` |  |
| deployment.envoyGateway.securityContext.runAsUser | int | `65532` |  |
| deployment.envoyGateway.securityContext.seccompProfile.type | string | `"RuntimeDefault"` |  |
| deployment.pod.affinity | object | `{}` |  |
| deployment.pod.annotations."prometheus.io/port" | string | `"19001"` |  |
| deployment.pod.annotations."prometheus.io/scrape" | string | `"true"` |  |
| deployment.pod.labels | object | `{}` |  |
| deployment.pod.nodeSelector | object | `{}` |  |
| deployment.pod.tolerations | list | `[]` |  |
| deployment.pod.topologySpreadConstraints | list | `[]` |  |
| deployment.ports[0].name | string | `"grpc"` |  |
| deployment.ports[0].port | int | `18000` |  |
| deployment.ports[0].targetPort | int | `18000` |  |
| deployment.ports[1].name | string | `"ratelimit"` |  |
| deployment.ports[1].port | int | `18001` |  |
| deployment.ports[1].targetPort | int | `18001` |  |
| deployment.ports[2].name | string | `"wasm"` |  |
| deployment.ports[2].port | int | `18002` |  |
| deployment.ports[2].targetPort | int | `18002` |  |
| deployment.ports[3].name | string | `"metrics"` |  |
| deployment.ports[3].port | int | `19001` |  |
| deployment.ports[3].targetPort | int | `19001` |  |
| deployment.priorityClassName | string | `nil` |  |
| deployment.replicas | int | `1` |  |
| global.imagePullSecrets | list | `[]` | Global override for image pull secrets |
| global.imageRegistry | string | `""` | Global override for image registry |
| global.images.envoyGateway.image | string | `nil` |  |
| global.images.envoyGateway.pullPolicy | string | `nil` |  |
| global.images.envoyGateway.pullSecrets | list | `[]` |  |
| global.images.ratelimit.image | string | `"envoyproxy/ratelimit:99d85510"` |  |
| global.images.ratelimit.pullPolicy | string | `"IfNotPresent"` |  |
| global.images.ratelimit.pullSecrets | list | `[]` |  |
| hpa.behavior | object | `{}` |  |
| hpa.enabled | bool | `false` |  |
| hpa.maxReplicas | int | `1` |  |
| hpa.metrics | list | `[]` |  |
| hpa.minReplicas | int | `1` |  |
| kubernetesClusterDomain | string | `"cluster.local"` |  |
| podDisruptionBudget.minAvailable | int | `0` |  |
| service.annotations | object | `{}` |  |
| service.trafficDistribution | string | `""` |  |
| topologyInjector.annotations | object | `{}` |  |
| topologyInjector.enabled | bool | `true` |  |

### `envoy-gateway-cr` Chart Values

The `envoy-gateway-cr` values are specified in the below table.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| gatewayClasses | object | `{"internal": {"name": "internal","envoyProxy": {"name": "internal","logging": "warn"},"envoyDeployment": {"daemonset": "false","replicas": 1,"resources": {"requests": {"cpu": "150m","memory": "640Mi"},"limits": {"cpu": "500m","memory": "1Gi"}}},"envoyService": {"type": "ClusterIP","name": "","externalTrafficPolicy": "Local"}},"external": {"name": "external","envoyProxy": {"name": "external","logging": "warn"},"envoyDeployment": {"daemonset": "false","replicas": 1,"resources": {"requests": {"cpu": "150m","memory": "640Mi"},"limits": {"cpu": "500m","memory": "1Gi"}}},"envoyService": {"type": "LoadBalancer","name": "","externalTrafficPolicy": "Local"},"ingress": {"create": false,"name": "alb","annotations": {"kubernetes.io/ingress.class": "alb","alb.ingress.kubernetes.io/load-balancer-name": "alb","alb.ingress.kubernetes.io/scheme": "internal","alb.ingress.kubernetes.io/target-type": "ip","alb.ingress.kubernetes.io/healthcheck-port": "19002","alb.ingress.kubernetes.io/healthcheck-path": "/healthz"}}}}` | Describes default (`internal` and `external`) GatewayClasses |
| defaultGateways | object | `{"internal":{"name":"default-internal-gateway","httpPort":80,"httpsPort":"","secret":{"create":false,"name":"internal-certificate"}},"external":{"name":"default-external-gateway","proxyProtocol":true,"underscoresAction":"RejectRequest","ctpName":"enable-proxy-protocol","ctpSpec":{},"httpPort":80,"httpsPort":"","secret":{"create":false,"name":"external-certificate"},"tcp":[],"udp":[],"hostPorts":"false"}}`       | Describes default (`internal` and `external`) Gateways |
| upgradeJob | object | `{"image":"ghcr.io/netcracker/qubership-docker-kubectl:0.0.6","pullPolicy":"IfNotPresent","resources":{"requests":{"cpu":"100m","memory":"128Mi"}},"nodeSelector":{"kubernetes.io/os":"linux"},"tolerations":[],"securityContext":{"runAsNonRoot":true,"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"seccompProfile":{"type":"RuntimeDefault"}}}` | Describes pre-upgrade job properties |
| global.images.envoyGateway.image | string | `"envoyproxy/envoy:distroless-v1.36.2"` |  |
| config.envoyGateway | object | `{"gateway":{"controllerName":"gateway.envoyproxy.io/gatewayclass-controller"},"provider":{"type":"Kubernetes"}}` | EnvoyGateway configuration. Must be equal to `envoy-gateway` values.yaml |


#### HTTPS Listeners

HTTPS listeners are **optional**.  
- Set `defaultGateways.internal.httpsPort` or `defaultGateways.external.httpsPort` to `""` to disable HTTPS.  
- TLS secrets are created **only if HTTPS is enabled** and `secret.create=true`.

Example:

```yaml
defaultGateways:
  external:
    httpsPort: ""  # disables HTTPS
    secret:
      create: true  # will be ignored if httpsPort is empty
```

#### HTTPS Listeners Certificates

HTTPS listeners certificates are optional. Certificates are managed by `secret.create` options in
`defaultGateways.internal` and `defaultGateways.external` sections.  
- Secrets will be created **only if** HTTPS listener is enabled (`httpsPort` is not empty) **and** `secret.create=true`.  
- Set `httpsPort` to `""` to disable HTTPS and skip secret creation.
- Certificates (`crt` and `key`) must be single-line Base64 strings if `secret.create=true`.

#### TLS Listeners

TLS listeners are required to set the `passthrough` type of HTTPS proxing on external `Gateway`, for example:

```yaml
defaultGateways:
  external:
    tls:
      - name: tls-1
        port: 4443
        hostname: namespace-3.cluster.example
        namespaceLabel:
          kubernetes.io/metadata.name: namespace-3
```

* `port` must defer to HTTPS listeners ports, if they are used at the same time.
* `namespaceLabel` could be `key: value` notation to allow particular namespaces label or `All` string to allow routes from all of the namespaces.

#### TCP and UDP Listeners Settings

It is possible to set TCP and UDP listeners on `external` gateway. For instance:

```yaml
defaultGateways:
  external:
    tcp:
      - name: sftp
        namespaceLabel:
          kubernetes.io/metadata.name: namespace-1
        port: 2222
```

As the result, the following listener will be created inside the external `Gateway`:

```yaml
listeners:
  - allowedRoutes:
      kinds:
      - group: gateway.networking.k8s.io
        kind: TCPRoute
      namespaces:
        from: Selector
        selector:
          matchLabels:
            kubernetes.io/metadata.name: namespace-1
    name: sftp
    port: 2222
    protocol: TCP
```

**Note**: The `port` option value must be unique within the protocol description.

#### ClientTrafficPolicy

The [ClientTrafficPolicy](https://gateway.envoyproxy.io/docs/api/extension_types/#clienttrafficpolicy) allows the user to configure the behavior of the connection between the downstream client and Envoy Proxy listener. Currently it's possible to manage only the following options:

```yaml
defaultGateways:
  external:
    proxyProtocol: true
    underscoresAction: RejectRequest
    ctpName: enable-proxy-protocol
    ctpSpec:
      timeout:
        http:
          idleTimeout: 1h
          requestReceivedTimeout: 30s
          streamIdleTimeout: 5m  
```

* `proxyProtocol` enables and disables the ProxyProtocol
* `underscoresAction` defines the actions under the HTTP Headers with underscores
* `ctpName` sets the name of resource
* `ctpSpec` set all of the `ClientTrafficPolicy` [spec](https://gateway.envoyproxy.io/docs/api/extension_types/#clienttrafficpolicyspec). It overrides the `proxyProtocol` and `underscoresAction` options if they are set in both parts. By default, `ctpSpec` is empty.

**Note**: Only one ClientTrafficPolicy could be attached to particular Gateway, so if the you are going to use some custom ClientTrafficPolicy the default one must be deleted

#### Deployments and DaemonSets

It's possible to use `DaemonSets` instead of `Deployments` resources for Envoy processes. 

If the options are set as the following:

```yaml
gatewayClasses:
  internal:
    envoyDeployment:
      daemonset: true
  external:
    envoyDeployment:
      daemonset: true
```

the `DaemonsSets` will be created:

```
$ kubectl -n envoy-gateway get daemonset
NAME                                                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
envoy-envoy-gateway-default-external-gateway-11a05f95   3         3         2       3            2           <none>          33s
envoy-envoy-gateway-default-internal-gateway-f9db644f   3         3         2       3            2           <n
```

#### Service Annotations

The `Services` that are being created during `Gateways` reconciliation could have custom annotations. It's usefull in public clouds installations, e.g.:

```yaml
gatewayClasses:
  external:
    envoyService:
      annotations:
        service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

The `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` annotation creates non-public Load Balancer for the particular `Service` in Azure infrastructure.

#### Hostports

There is a posibility to use `hostPort` on Envoy pods. It's more convenient way to send traffic from load balancer to `hostPort` instead of `nodePort`, because `hostPort` is not changed after `Pod` recreation. 

The `values.yaml` part example is as follows:

```yaml
defaultGateways:
  external:
    hostPorts: true
```

#### Gateway Configuration

The `config` section must be the same as described in `envoy-gateway` chart. The default version is the following:

```yaml
config:
  envoyGateway:
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    provider:
      type: Kubernetes
```

## Installation Check

Checking installation via ArgoCD is not necessary as ArgoCD checks all of the resources in chart itself.

### `envoy-gateway` Chart

After the installation all of the CRDs must be created:

```bash
kubectl get crd | grep 'gateway'
backends.gateway.envoyproxy.io                        2026-01-15T07:55:30Z
backendtlspolicies.gateway.networking.k8s.io          2026-01-15T07:54:53Z
backendtrafficpolicies.gateway.envoyproxy.io          2026-01-15T07:55:30Z
clienttrafficpolicies.gateway.envoyproxy.io           2026-01-15T07:55:31Z
envoyextensionpolicies.gateway.envoyproxy.io          2026-01-15T07:55:31Z
envoypatchpolicies.gateway.envoyproxy.io              2026-01-15T07:55:31Z
envoyproxies.gateway.envoyproxy.io                    2026-01-15T07:55:32Z
gatewayclasses.gateway.networking.k8s.io              2026-01-15T07:54:54Z
gateways.gateway.networking.k8s.io                    2026-01-15T07:54:54Z
grpcroutes.gateway.networking.k8s.io                  2026-01-15T07:54:54Z
httproutefilters.gateway.envoyproxy.io                2026-01-15T07:55:32Z
httproutes.gateway.networking.k8s.io                  2026-01-15T07:55:19Z
referencegrants.gateway.networking.k8s.io             2026-01-15T07:54:55Z
securitypolicies.gateway.envoyproxy.io                2026-01-15T07:55:33Z
tcproutes.gateway.networking.k8s.io                   2026-01-15T07:54:56Z
tlsroutes.gateway.networking.k8s.io                   2026-01-15T07:54:56Z
udproutes.gateway.networking.k8s.io                   2026-01-15T07:54:57Z
xbackendtrafficpolicies.gateway.networking.x-k8s.io   2026-01-15T07:54:57Z
xlistenersets.gateway.networking.x-k8s.io             2026-01-15T07:54:57Z
xmeshes.gateway.networking.x-k8s.io                   2026-01-15T07:54:57Z
```

Envoy Gateway operator must be up and running:

```bash
kubectl -n envoy-gateway get pod

NAME                                                              READY   STATUS    RESTARTS   AGE
envoy-gateway-7c865fb9c4-78cvd                                    1/1     Running   0          12h
```

### `envoy-gateway-cr` Chart

During the installation, several custom resources are being created. It is possible to check them by the following commands:

The existing `GatewayClasses`:

```bash
kubectl get gatewayclasses

NAME       CONTROLLER                                      ACCEPTED   AGE
external   gateway.envoyproxy.io/gatewayclass-controller   True       42h
internal   gateway.envoyproxy.io/gatewayclass-controller   True       42h
```

The existing `Gateways`:

```bash
kubectl get gateways.gateway.networking.k8s.io -A

NAMESPACE       NAME                       CLASS      ADDRESS         PROGRAMMED   AGE
envoy-gateway   default-external-gateway   external                   False        12h
envoy-gateway   default-internal-gateway   internal   172.30.164.73   True         12h
```

The existing `EnvoyProxies`:

```bash
kubectl get envoyproxy -A

NAMESPACE       NAME       AGE
envoy-gateway   external   12h
envoy-gateway   internal   12h
```

The existing `ClientTrafficPolicy`

```bash
kubectl get clienttrafficpolicy -A

NAMESPACE       NAME                    AGE
envoy-gateway   enable-proxy-protocol   12h
```

## AWS Application Load Balancer (ALB) Integration

The information for AWS Application Load Balancer (ALB) integration for Envoy API Gateway is provided below.

### Problem Statement

AWS as a public cloud provider has Web Application Firewall (WAF) in its services scope. It could be attached to ALB only. The [aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) creates ALBonly for `Ingress` resources. This makes it impossible to send the traffic to Envoy Gateway directly, since it uses only `Services` and aws-load-balancer-controller creates Network Load Balancer (NLB) in that case. To solve this problem, it is necessary to change the type of related `Service` and create `Ingress` that points to Envoy Gateway `Service`. After that measures, NLB gets destroyed and ALB will be created.

### Implementaion Steps

The implementation steps are specified below.

1. Set `gatewayClasses.external.envoyService.type` to `ClusterIP` and set `gatewayClasses.external.envoyService.name`, it must be not empty.
2. Set `defaultGateways.external.proxyProtocol` to  `false`.
3. Set the `gatewayClasses.external.ingress.create` option to `true`. The `alb.ingress.kubernetes.io/certificate-arn` annotation enables HTTPS and points to a TLS certificate ARN that could be taken from AWS Certificate Manager, set it. The `gatewayClasses.external.ingress` section should be like the following:

```
    ingress:
      create: true
      name: alb
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/load-balancer-name: alb
        alb.ingress.kubernetes.io/scheme: internal
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/healthcheck-port: '19002'
        alb.ingress.kubernetes.io/healthcheck-path: /healthz
        alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxxxx:certificate/xxxxxxx
```

4. Install the Envoy Gateway.
5. Get the URL from `Ingress` status in the Envoy Gateway namespace:

```yaml
status:
  loadBalancer:
    ingress:
    - ip: internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com
```

6. Check the availability of the `HTTPRoute` (put particular endpoint into `Host` header):

```
$ curl -v -H "Host: test.service.envoy-gateway" http://internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com
*   Trying 10.98.237.56:80...
* Connected to internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com (10.98.237.56) port 80 (#0)
> GET /admin HTTP/1.1
> Host: waf.eks.envoy-gateway
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Thu, 14 Aug 2025 05:26:51 GMT
< Content-Type: text/plain
< Transfer-Encoding: chunked
< Connection: keep-alive
< server: echoserver
<
...
```

## Upgrade

Basically, upgrade procedure similar to installation, except the one thing. All of the CRDs must be upgraded before the other resources in case of native Helm usage.
