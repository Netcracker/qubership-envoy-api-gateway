# Troubleshooting Guide

## 1. Common Installation checks

When the charts are installed via ArgoCD, these manual checks are not necessary, as ArgoCD itself checks the health of all resources in the chart.

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
kubectl -n gateway-system get pod

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

NAMESPACE        NAME                       CLASS      ADDRESS         PROGRAMMED   AGE
gateway-system   default-external-gateway   external                   False        12h
gateway-system   default-internal-gateway   internal   172.30.164.73   True         12h
```

A healthy Gateway reports `PROGRAMMED` `True`. In the example above the external Gateway is not programmed; see section 3 for how to investigate it.

The existing `EnvoyProxies`:

```bash
kubectl get envoyproxy -A

NAMESPACE        NAME       AGE
gateway-system   external   12h
gateway-system   internal   12h
```

The existing `ClientTrafficPolicy`

```bash
kubectl get clienttrafficpolicy -A

NAMESPACE        NAME                    AGE
gateway-system   enable-proxy-protocol   12h
```

`enable-proxy-protocol` is the chart default name, even though this policy also holds all other `ctpSpec` settings, not only PROXY protocol. KubeMarine overrides the name to `external-client-traffic-policy`.

## 2. Common availability checks

### HTTPRoutes and TLSRoutes

The simplest way to check an HTTPRoute or TLSRoute is to make a request with the `curl` utility.

#### HTTPRoute

Get the particular endpoint from `spec.hostnames` in HTTPRoute e.g.:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: echoserver
spec:
  hostnames:
  - test.service.envoy-gateway
```

Put it into `Host` header:

```shell
$ curl -v -H "Host: test.service.envoy-gateway" http://gateway.k8s.local
*   Trying 10.10.0.1:80...
* Connected to gateway.k8s.local (10.10.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: test.service.envoy-gateway
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

In case of AWS ALB integration get the ALB address from the status of the ALB Ingress in the Envoy Gateway namespace:

```yaml
status:
  loadBalancer:
    ingress:
    - hostname: internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com
```

Put the hostname from the HTTPRoute into `Host` header. The ALB redirects HTTP to HTTPS, so send the request to port 443 (`-k` is needed because the certificate does not cover the ALB address):

```shell
$ curl -vk -H "Host: test.service.envoy-gateway" https://internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com
*   Trying 10.10.0.1:443...
* Connected to internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com (10.10.0.1) port 443 (#0)
...
> GET / HTTP/1.1
> Host: test.service.envoy-gateway
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

#### TLSRoute

Get the particular endpoint from `spec.hostnames` in TLSRoute e.g.:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha3
kind: TLSRoute
metadata:
  name: test
  namespace: default
spec:
  hostnames:
    - some.endpoint.domain.local
```

```shell
$ curl -vk https://some.endpoint.domain.local
*   Trying 10.20.30.40:443...
* Connected to some.endpoint.domain.local (10.20.30.40) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=some.endpoint.domain.local
*  start date: Jun  1 00:00:00 2026 GMT
*  expire date: Jun  1 23:59:59 2027 GMT
*  issuer: CN=Example Internal CA
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://some.endpoint.domain.local/
* [HTTP/2] [1] :method: GET
* [HTTP/2] [1] :scheme: https
* [HTTP/2] [1] :authority: some.endpoint.domain.local
* [HTTP/2] [1] :path: /
> GET / HTTP/2
> Host: some.endpoint.domain.local
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/2 200
< content-type: application/json
< content-length: 52
< date: Thu, 18 Jun 2026 12:00:00 GMT
<
{"status":"ok","message":"service is reachable"}
* Connection #0 to host some.endpoint.domain.local left intact
```

### TCPRoutes

The TCPRoute could be checked by `netcat` e.g.:

```shell
nc -v tcp.endpoint.domain.local 5432
Connection to tcp.endpoint.domain.local (10.20.30.155) 5432 port [tcp/*] succeeded!
```

where `tcp.endpoint.domain.local` is the FQDN of the TCP application. Usually it resolves to one of the IP addresses attached to the Load Balancer.

The more specific checks depend on particular application and could be performed through the appropriate client tools.

### UDPRoutes

A UDPRoute to a TFTP endpoint can be checked in the following manner:

```shell
$ curl tftp://udp.endpoint.domain.local:6969/hello-world

Hostname: echoserver-udp-567b76c8b8-lkv5g

Request Information:
        client_address=10.205.6.195
        client_port=59914
        real path=/hello-world
        request_scheme=tftp
```

where `udp.endpoint.domain.local` is the FQDN of the UDP application. Usually it resolves to one of the IP addresses attached to the Load Balancer.

Other UDP applications can be checked with their protocol-specific tools.

## 3. Common Troubleshooting steps

If the response is not expected go through the following pattern to identify the issue.

1. Check the Route status. The valid status looks like the following:

```yaml
  status:
    parents:
    - conditions:
      - lastTransitionTime: "2026-05-22T12:25:21Z"
        message: Route is accepted
        observedGeneration: 1
        reason: Accepted
        status: "True"
        type: Accepted
      - lastTransitionTime: "2026-05-22T12:25:21Z"
        message: Resolved all the Object references for the Route
        observedGeneration: 1
        reason: ResolvedRefs
        status: "True"
        type: ResolvedRefs
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
      parentRef:
        group: gateway.networking.k8s.io
        kind: Gateway
        name: default-external-gateway
        namespace: gateway-system
```

If the status is different, check the `parentRefs` in the Route specification (see section 11). It must refer to the particular Gateway in the correct namespace and must correctly reference the listener. The listener protocol must match the Route type. Also check that `backendRefs` are correct: they must point to a Service with ready Pods.

2. Check the Gateway status. The valid status looks like the following:

```yaml
status:
  addresses:
  - type: IPAddress
    value: 192.168.1.1
  - type: IPAddress
    value: 192.168.1.2
  conditions:
  - lastTransitionTime: "2026-06-17T20:37:36Z"
    message: The Gateway has been scheduled by Envoy Gateway
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted
  - lastTransitionTime: "2026-06-17T20:37:36Z"
    message: Address assigned to the Gateway, 2/2 envoy replicas available
    observedGeneration: 1
    reason: Programmed
    status: "True"
    type: Programmed
...
```

The most important message is 'Address assigned to the Gateway'. That means the Gateway has been integrated correctly.

3. If the Gateway does not have condition `type: Programmed` with `status: "True"`, check the correctness of the `gatewayClassName` option.

4. If the `gatewayClassName` refers to correct GatewayClass check the GatewayClass status, e.g.:

```yaml
status:
  conditions:
  - lastTransitionTime: "2026-06-17T16:49:41Z"
    message: Valid GatewayClass
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted

```

If there is no condition `type: Accepted` with `status: "True"`, check the `parametersRef` and correct it:

```yaml
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: external
    namespace: gateway-system
```

5. If the GatewayClass is accepted, check the EnvoyProxy. Check the Service that is used to send traffic to Envoy, e.g.:

```shell
$ kubectl -n gateway-system get svc
NAME                                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
envoy-gateway-system-default-external-gateway-0b2e9423   ClusterIP   172.30.94.122   <none>        80/TCP,443/TCP               12h
```

The Service type must match the schema (`gatewayClasses.external.envoyService` in `envoy-gateway-cr` values):
* AWS EKS schema: `ClusterIP` with a non-empty `name`; the ALB targets this Service (see section 17).
* Azure AKS schema: `LoadBalancer` (chart default); the Service must have an `EXTERNAL-IP` assigned by the Azure LB.
* KubeMarine and Bubble schemas: Envoy runs as a DaemonSet with host ports and HAProxy targets the node host ports directly, so the Service type is not relevant for edge traffic.

## 4. Header with underscores is dropped or request is rejected

### Description

A request carrying a header whose name contains `_` is rejected, or the header does not reach the application. Ingress-NGINX passed such headers through, while Envoy Gateway `ClientTrafficPolicy` defaults `withUnderscoresAction` to `RejectRequest`.

### How to Solve

Set the option on `defaultGateways.external.ctpSpec` in `envoy-gateway-cr` values (it renders the `ClientTrafficPolicy` resource) and redeploy:

```yaml
defaultGateways:
  external:
    ctpSpec:
      headers:
        withUnderscoresAction: "Allow"
```

## 5. Protocol error (426 Upgrade Required)

### Description

HTTP/1.0 clients or probes get a protocol error (`426 Upgrade Required`). Ingress-NGINX accepted HTTP/1.0 by default, Envoy Gateway does not.

### How to Solve

Enable HTTP/1.0 on `defaultGateways.external.ctpSpec` in `envoy-gateway-cr` values and redeploy:

```yaml
defaultGateways:
  external:
    ctpSpec:
      http1:
        http10: {}
```

## 6. Redirect to incorrect URLs (X-Forwarded-Host)

### Description

The application builds wrong absolute URLs (for example in redirects), because it relies on the `X-Forwarded-Host` header which Ingress-NGINX set and Envoy Gateway does not set by default. A related parity gap: request IDs are regenerated on every hop instead of being preserved.

### How to Solve

Set the headers options on `defaultGateways.external.ctpSpec` in `envoy-gateway-cr` values and redeploy:

```yaml
defaultGateways:
  external:
    ctpSpec:
      headers:
        requestID: "PreserveOrGenerate"
        earlyRequestHeaders:
          add:
          - name: "X-Forwarded-Host"
            value: '%REQ(Host)%'
```

## 7. Redirect to incorrect protocol (HTTP instead of HTTPS)

### Description

The application redirects to `http://` URLs, because it does not receive the `X-Forwarded-Proto` header with `https` value. This happens when Envoy does not trust the `X-Forwarded-*` headers set by the proxy in front of it.

### How to Solve

`numTrustedHops` must equal the number of proxies in front of Envoy: `1` for AWS ALB schema. For other schemas it is currently not relevant.

```yaml
defaultGateways:
  external:
    ctpSpec:
      clientIPDetection:
        xForwardedFor:
          numTrustedHops: 1
```

### Recommendations

`numTrustedHops` is schema-dependent; establish the schema before choosing the right a value. Currently applicable only for AWS schema.

## 8. All requests fail due to PROXY protocol misconfiguration (enabled/disabled on wrong schema)

### Description

Every request fails at connection level; data-plane logs show malformed or truncated requests and no HTTP status.

`defaultGateways.external.proxyProtocol` makes Envoy *require* a PROXY protocol header on every connection, for HTTP, TLS, and TCP routes alike. So the value must match what the proxy in front of Envoy actually sends:
* KubeMarine regular schema: HAProxy sends PROXY protocol, so it must be enabled. KubeMarine enables it by default.
* AWS ALB and Azure NLB schemas: they do not use PROXY protocol, so it must be `false`.
* Bubble environments: PROXY protocol is not used either; KubeMarine should normally disable it by default.

### How to Solve

For AWS and Azure schemas, set `defaultGateways.external.proxyProtocol: false` in `envoy-gateway-cr` values and redeploy. For KubeMarine schemas, check the `plugins.envoy-gateway` section in `cluster.yaml` for user overrides that change the KubeMarine default.

### Recommendations

`proxyProtocol` is schema-dependent; establish the schema before choosing the value.

## 9. All traffic fails with data-plane error "filter chain 'EmptyCluster' has the same matching rules defined as" (TLS listener without TLSRoute)

### Description

This is a bug in Envoy Gateway which happens when there is a TLS listener configured on the Gateway, but there is no corresponding TLSRoute for its hostname.

### Stack Trace

Errors like the following in Envoy Gateway data-plane pods logs (match on the substring):

```
filter chain 'EmptyCluster' has the same matching rules defined as 
```

### How to Solve

Either create the TLSRoute for the listener hostname, or remove the TLS listener from the Gateway configuration (via chart values, not the live Gateway).

### Recommendations

This issue could be confused with section 11, when HTTPRoutes and TLSRoutes reference incorrect parents. If the issue from section 11 is present, it most likely should be fixed first: it could be the reason why TLSRoutes are not attached to the TLS listener.

## 10. HTTPS listener certificate issues

### Description

TLS handshake failures on the HTTPS hostname, or HTTPS not listening at all, or certificate errors in the Gateway CR status. Causes, on `envoy-gateway-cr` values:

- `defaultGateways.external.httpsPort: ""` — HTTPS is disabled entirely.
- `defaultGateways.external.secret.create: false` with no pre-existing Secret named `defaultGateways.external.secret.name`.
- `defaultGateways.external.secret.create: true` but the certificate is not a single-line Base64 string — Secret creation fails.

This is not applicable for AWS schema, since in AWS Envoy Gateway listens only on port 80 and TLS is terminated on the ALB (see section 17).

### How to Solve

- HTTPS disabled: set `defaultGateways.external.httpsPort` to a port.
- Secret missing: set `secret.create: true` with `crt`/`key`, or pre-create the Secret.
- Secret creation fails: provide the certificate and key as single-line Base64 strings.

KubeMarine schemas set the certificate in `cluster.yaml` instead: `plugins.envoy-gateway.externalGateway.certificate.cert` / `.key`.

### Recommendations

**Bubble environments schema** additionally requires a certificate whose SAN list covers **both** the external and the internal (bubble) hostnames — one listener serves both, so a certificate covering only one set breaks the other.

## 11. HTTPRoute not working due to incorrect parentRef

### Description

A single hostname returns 404 from Envoy and its `HTTPRoute` status shows it is not accepted by the expected Gateway.

Related: an `Ingress` with a `spec.tls` section implies an automatic HTTP→HTTPS redirect. The converted route keeps that redirect, so if the Gateway has no HTTPS listener, the hostname redirects and then fails to connect.

### How to Solve

- `spec.parentRefs[].name` / `.namespace` must match the external Gateway — defaults `default-external-gateway` in `gateway-system`, overridable via `defaultGateways.external.name`.
- `spec.parentRefs[].port`, when set, must match an existing listener. An HTTP→HTTPS redirect route targets port `80` and the main route `443`; a route pinned to `443` where `httpsPort` is `""` attaches to nothing.
- If the schema does not use HTTPS listener (AWS) and/or converter is in use, then the problem most likely is because Ingress contains unnecessary HTTPS redirect. In this case Ingress should be changed to remove the redirect, or service should provide their own HTTPRoute, since converter currently always respects HTTPS redirect.

### Recommendations

If the HTTPRoute uses an incorrect Gateway name or namespace, make sure that the application was deployed with correct deploy parameters. Most applications use contracted deploy parameters `GATEWAY_SYSTEM_NAME` and `GATEWAY_SYSTEM_NAMESPACE` (see section 15); gateway-api-converter also uses these parameters when generating HTTPRoutes.

## 12. ArgoCD reports Gateway API or Envoy Gateway resources permanently OutOfSync

### Description

An ArgoCD application stays `OutOfSync` on `HTTPRoute` or other Gateway API resources with a diff the user did not author, because Envoy Gateway admission sets defaults explicitly on apply ([argoproj/argo-cd#22151](https://github.com/argoproj/argo-cd/issues/22151)).

### How to Solve

The fix is an application-side chart change: include the admission-applied defaults in the rendered manifest.

## 13. Traffic still working through Ingress-NGINX, not switched to Envoy Gateway

### Description

Envoy Gateway is healthy and routes are accepted, but responses still show NGINX behavior (e.g. stop working when Ingress-NGINX is scaled down). The infrastructure was never repointed to Envoy Gateway.

### How to Solve

- For KubeMarine schema: `services.loadbalancer.target_backend` in `cluster.yaml` is still `nginx`. Set it to `envoy` and apply by running the KubeMarine `install` procedure with task `deploy.loadbalancer.haproxy.configure`. Default ports: 20080/20443 NGINX, 21080/21443 Envoy.
- For AWS schema: the ALB still targets the NGINX target group, or DNS still resolves to the NGINX ALB. Repoint it to the Envoy Gateway ALB.
- For Azure schema: the Network LB still targets the NGINX, or DNS still resolves to the NGINX Network LB. Repoint it to the Envoy Gateway Network LB.

### Recommendations

In KubeMarine schema HAProxy sends *all* traffic to one backend — per-hostname splitting is not possible. Splitting ingress traffic between Ingress-NGINX and Envoy Gateway is not recommended/supported.

## 14. Ingress apply fails on admission webhook after Ingress-NGINX removal

### Description

After NGINX removal, every `Ingress` apply fails — including deploys that touch Ingress only incidentally. The `ingress-nginx-admission` ValidatingWebhookConfiguration survived and points at a Service that no longer exists.

### How to Solve

Delete the `ingress-nginx-admission` ValidatingWebhookConfiguration.

When NGINX was installed by KubeMarine, removal is manual and must also delete the `ingress-nginx` namespace, the `nginx` IngressClass, and the `ingress-nginx` / `ingress-nginx-admission` ClusterRoles and ClusterRoleBindings.

### Recommendations

To disable NGINX only *temporarily*, back the webhook up before deleting it and add a non-existent `nodeSelector` to its DaemonSet rather than uninstalling.

## 15. Application does not deploy Gateway API or Envoy Gateway resources

### Description

An application deploys no `HTTPRoute` at all, or one whose `parentRefs` point at a non-existent Gateway, because the environment (cloud passport) does not supply the parameters the chart gates on:

| Parameter | Default | Effect when wrong or missing |
|---|---|---|
| `GATEWAY_SYSTEM_TYPE` | `legacy-ingress` | without `gateway-api-default` in the value, no Gateway API resources render |
| `GATEWAY_SYSTEM_NAMESPACE` | `gateway-system` | `parentRefs[].namespace` points at the wrong namespace |
| `GATEWAY_SYSTEM_NAME` | `default-external-gateway` | `parentRefs[].name` points at a non-existent Gateway |

### How to Solve

Set the parameters in the environment and redeploy the application. `GATEWAY_SYSTEM_TYPE` is a comma-separated list, so `legacy-ingress,gateway-api-default` is valid and renders both Ingress and Gateway API resources.

### Recommendations

The same parameters must be consistent with the converter's configuration when the converter is installed.

## 16. Envoy Gateway converting encoded slash symbol to real slash

### Description

Requests containing `%2F` behave differently than under Ingress-NGINX — Envoy Gateway decodes encoded slashes automatically and merges adjacent slashes.

### How to Solve

The following configuration in the `envoy-api-gateway-cr` application makes Envoy Gateway behave like Ingress-NGINX:

```yaml
defaultGateways:
  external:
    ctpSpec:
      path:
        escapedSlashesAction: KeepUnchanged
        disableMergeSlashes: true
```

## 17. AWS ALB does not reach Envoy Gateway

### Description

In AWS schema, the ALB in front of Envoy Gateway must terminate TLS on 443 with the certificate from `gatewayClasses.external.ingress.annotations."alb.ingress.kubernetes.io/certificate-arn"`, redirect 80 to 443, and forward to the Envoy Service on its HTTP port (80 by default, `defaultGateways.external.httpPort`). Health check: port `19002`, path `/healthz` (annotations `alb.ingress.kubernetes.io/healthcheck-port` and `-path`).

### How to Solve

- `gatewayClasses.external.envoyService.type` must be `ClusterIP` with a non-empty `.name` — the ALB controller cannot bind to an unnamed Service, and type `LoadBalancer` would create an NLB instead.
- Check the certificate ARN and health check annotations above.

### Recommendations

One ALB cannot serve both Ingresses and Gateways; each controller needs its own.