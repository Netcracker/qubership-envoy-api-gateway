# Troubleshooting guide 

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

## Availability Check

### HTTPRoutes and TLSRoutes

The simpliest way to check the HTTPRoute, TLSRoute is to make the request through `curl` utility.

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

```
$ curl -v -H "Host: test.service.envoy-gateway" http://gateway.k8s.local
*   Trying 10.10.0.1:80...
* Connected to gateway.k8s.local (10.10.0.1) port 80 (#0)
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

In case of AWS ALB integration get the URL from Ingress status in the Envoy Gateway namespace:

```yaml
status:
  loadBalancer:
    ingress:
    - ip: internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com
```

Put particular `host` from Ingress into `Host` header:

```
$ curl -v -H "Host: test.service.envoy-gateway" http://internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com
*   Trying 10.10.0.1:80...
* Connected to internal-alb-waf-1023017451.us-east-1.elb.amazonaws.com (10.10.0.1) port 80 (#0)
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

where `tcp.endpoint.domain.local` is FQDN of TCP application. Usually it goes to the one of the IP address that attached to the Load Balancer

The more specific checks depend on particular application and could be performed through the appropriate client tools.

### UDPRoutes

The TFTP endpoint could be check in the following manner:

```
$ curl tftp://udp.endpoint.domain.local:6969/hello-world

Hostname: echoserver-udp-567b76c8b8-lkv5g

Request Information:
        client_address=10.205.6.195
        client_port=59914
        real path=/hello-world
        request_scheme=tftp
```

where `udp.endpoint.domain.local` is FQDN of UDP application. Usually it goes to the one of the IP address that attached to the Load Balancer

There are a lot of applications that are using UDP payload and they could be checked through the specific tools.

## Troubleshooting

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

If the status is different check the `parentRefs` in the `Route` specification. It must refer to the particular Gateway in correct Namespace and must have correct reference to the linster. The listener must match the Route type. Also check the `backendRefs` correctness. It must go to the Service with active Pods.

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

3. If Gateway doesn't have `type: Programmed` check the correctness of `gatewayClassName` option

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

If there is not `type: Accepted` check the `parametersRef` and correct it: 

```yaml
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: external
    namespace: envoy-api-gateway
```

5. If the GatewayClass status is `type: Accepted` check the EnvoyProxy. Check the Service that is used to send traffic to the Envoy e.g.:

```shell
$ kubectl -n envoy-api-gateway get svc
NAME                                                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                    AGE
envoy-envoy-api-gateway-default-external-gateway-0b2e9423   NodePort    172.30.94.122    <none>        80:31861
```

The type of Service must be acceptable for the particular scheme. The `NodePort` type is used when the external Load Balancer is configured to send traffic to particular port. The `LoadBalancer` type is used when the Load Balancer controller supplies integration between Kubernetes cluster and infrastructure. The `ClusterIP` is used when the Load Balancer is running right on Kubernetes cluster.
