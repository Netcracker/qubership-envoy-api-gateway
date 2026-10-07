---
name: troubleshoot-envoy-api-gateway
description: Diagnose and resolve Envoy Gateway deployment and ingress traffic failures on Kubernetes clusters (KubeMarine clusters and public clouds). Use this skill when troubleshooting envoy-api-gateway and envoy-api-gateway-cr applications issues, KubeMarine envoy-gateway plugin issues, gateway-api-converter issues, Gateway API and Envoy Gateway CRs problems (e.g. GatewayClass, Gateway, HTTPRoute, EnvoyProxy, ClientTrafficPolicy, BackendTrafficPolicy), Envoy Gateway control-plane and data-plane pods problems, TLS/HTTPS setup, Envoy Gateway pods out-of-cluster access (through HAProxy or public cloud LBs), problems with external access to applications.
---

# Troubleshoot Envoy API Gateway

## 1. Components

Following high-level overview of components is useful when troubleshooting problems related to Envoy Gateway:
* `envoy-api-gateway` application, contains the `envoy-gateway` Helm chart, which deploys Envoy Gateway "control-plane" operator Deployment and Gateway API and `gateway.envoyproxy.io` CRDs.
* `envoy-api-gateway-cr` application, contains the `envoy-gateway-cr` Helm chart, which deploys Envoy Gateway data-plane CRs such as `GatewayClass`, `Gateway`, `EnvoyProxy`, `ClientTrafficPolicy`, `BackendTrafficPolicy` and others. Envoy Gateway control-plane pod then creates the data-plane Deployment or DaemonSet.
* `gateway-api-converter` application, contains Helm chart which deploys one operator Deployment that watches `Ingress` cluster-wide and generates `HTTPRoute` and other Envoy Gateway (Gateway API) CRs at runtime. Used to allow Envoy Gateway integration/migration for applications which do not yet provide their own Gateway API resources.
* `ingress-nginx` is deprecated predecessor of Envoy Gateway. We migrate from it to Envoy Gateway. Both could be installed on the cluster at the same time to allow smooth migration, but exactly one of the two is configured to receive edge traffic.
* KubeMarine is Kubernetes installer for on-prem envs, it has `envoy-gateway` plugin which installs both `envoy-gateway` and `envoy-gateway-cr` charts. Uses recommended default parameters for Envoy Gateway charts, additional configuration (e.g. certificates and user overrides) are provided through KubeMarine `cluster.yaml` input configuration. Also configures the HAProxy LB which exposes Envoy Gateway. Not relevant for public clouds.

## 2. Deployment schemas

Identify the Envoy Gateway schema before anything else, it is useful to derive which problems could be relevant and which configuration is required/missing/wrong:
* KubeMarine regular schema. This is schema relevant for most on-prem envs. In this schema Envoy Gateway is installed by KubeMarine as a plugin, most recommended configuration options are already set by KubeMarine by default. Most of the time only certificates are provided by user. Envoy Gateway runs as DaemonSet on worker hostPorts 21080/21443; external HAProxy VMs on ports 80/443 run in front of Envoy Gateway (PROXY protocol is enabled for communication between HAProxy and Envoy Gateway).
* AWS EKS schema. In this schema Envoy Gateway is installed as a plain Helm chart (not ArgoCD or AppDeployer). Envoy Gateway is exposed via AWS ALB targeting Envoy Gateway ClusterIP Service.
* Azure AKS schema. Envoy Gateway also installed as a plain Helm chart. Envoy Gateway is exposed via Azure NLB, using k8s Service of type LoadBalancer.
* Bubble environments. This is a special case of KubeMarine schema where Envoy Gateway runs on ports 80/443. Ingress-NGINX is always scaled down in this schema. In this schema an additional "window" namespace is used to expose applications externally, this namespace contains "external" Ingresses - these Ingresses have external hostnames and are configured to rewrite/resend them to internal hostnames used by other actual application Ingresses in other namespaces of the same cluster. Gateway API Converter converts "window" namespace Ingresses to their HTTPRoute alternatives (Ingresses in other namespaces also could be converted).

In all of the above schemas additionally:
* Both Envoy charts (and converter, if present) are installed in the same namespace, by default `gateway-system`.
* External ingress traffic is served by `external` GatewayClass, Gateway and EnvoyProxy CRs. The `internal` counterparts are not actually used in most cases.
* Gateway API Converter could be present.
* Ingress-NGINX still could be installed if migration is not finished. The migration process is roughly the following:
    * Ingress-NGINX was installed and infrastructure (LB/DNS) was configured to point all external traffic to Ingress-NGINX.
    * First migration step is to install Envoy Gateway alongside Ingress-NGINX, on the same cluster.
    * Then, infrastructure should be reconfigured to point to Envoy Gateway, external access should be verified.
    * Finally, Ingress-NGINX could be deleted, including admission webhook to avoid Ingress issues.

If the schema cannot be determined from the artifacts, ask the user. Following hints can help guess the schema:
* Look at the URL, sometimes it hints AWS/Azure envs.
* If it is on-site env, then it is most likely public cloud, KubeMarine is rarely used on-site (only few on-site on-prem envs).
* If it is internal env which has "SaaS" in URL, then it is definitely KubeMarine schema.
* Bubble environments are rare and are all internal envs, they do not have "SaaS" in the name.

## 3. Blast radius

**Whole cluster** (no hostname works, or all fail identically): fault is in Envoy Gateway, KubeMarine, the converter, or the infrastructure in front.

**One or a few hostnames**: fault is more likely in the route, or the application, or gateway-api-converter (or source Ingress).

**Traffic never reaches Envoy (no access-log lines for the request in data-plane logs)**: suspect infrastructure problems (LB/DNS configuration).

## 4. Inputs to request

* For KubeMarine regular schema, request `cluster.yaml` and KubeMarine version.
* For Envoy Gateway installation as charts (AWS/Azure), request `values.yaml` for `envoy-api-gateway` and `envoy-api-gateway-cr` applications, also request these applications versions.
* If converter is present, request its `values.yaml`, version and pod logs.
* In case of cluster-wide issues, request Gateway API resources statuses (GatewayClass, Gateway, ClientTrafficPolicy, BackendTrafficPolicy).
* In case of problems with particular route, request its HTTPRoute, together with status. If converter is present additionally request original Ingress yaml.
* Request Envoy Gateway control-plane and data-plane pods logs if Gateway API resources statuses are not enough to determine the problem.
* For AWS schema, confirm that ALB exists in front of Envoy Gateway, that it targets correct Service port (80), that DNS points to this ALB.
* For Azure schema, confirm that NLB exists in front of Envoy Gateway, that DNS points to this NLB.
* In case of install / upgrade failures, request deploy logs (KubeMarine job, helm deploy logs, ArgoCD/AppDeployer job logs).

Note that even though versions information may not be used in troubleshooting descriptions, version information still could be useful for further human assessment.

Only request most relevant information, do not overload user immediately with a lot of not-so-necessary and ambigious action items.

## 5. Workflow

1. Identify the schema (§2); ask the user if the artifacts are ambiguous.
2. Determine blast radius (§3).
3. Collect relevant configuration and versions (§4).
4. Collect logs according to schema, blast radius and configuration.
5. Assess collected information, match configuration and logs against known issues (§6, §7).

## 6. Known issues

Installation checks, availability checks, the generic troubleshooting procedure and known issues with their solutions are in `references/Troubleshooting.md`. The file is large and new issues get added over time, so do not rely on memory or read it whole — discover its sections first:

1. List all sections with their line numbers (path is relative to the directory containing SKILL.md):
   ```bash
   grep -n '^## ' references/Troubleshooting.md
   ```
   Sections 1-3 are the common installation checks, availability checks and generic troubleshooting steps. Every later `##` section is a known issue, with subsections.
2. Search the whole file for exact error strings, status codes, header names, parameter names or CR kinds taken from the user's logs and configuration.
3. Read each candidate section in full, from its heading line to the next `## ` heading. Check every known issue whose title matches the symptoms, not just the first one.

## 7. Gateway API Converter

**No route generated for an Ingress** — the `Ingress` carries `gateway-api-converter.netcracker.com/ignore: "true"` or some unsupported annotations (search converter logs by ingress name). That is correct when the application ships its own `HTTPRoute`; if it is not doing so, its gating parameters are missing (`references/Troubleshooting.md`, section 15).

**Duplicate or conflicting routes** on one hostname — the application ships its own `HTTPRoute` but did *not* set the ignore annotation, so the converter generated a competing route. Add the annotation to the `Ingress` and delete the generated route.

**ArgoCD Ingress health check permanently degraded** on a cluster with no Ingress-NGINX — nothing assigns an address, so `Ingress` status never populates. Either add the ignore annotation (it also suppresses the ArgoCD health check), or update the converter and set `annotateMissingLBIntegration: true`.

**`ssl-passthrough` Ingress not working after conversion**, in this case there are two most likely scenarios:
1. `ssl-passthrough` is not needed at all, it was not working even on Ingress-NGINX, because Ingress-NGINX requires additional option `--enable-ssl-passthrough` (could be checked in Ingress-NGINX pod arguments). In this case the best thing to do is set `ignoreSSLPassthrough: true` in converter deploy parameters, converter will also ignore this annotation during conversion.
2. TLS listener is not configured on Gateway for the generated TLSRoute. In this case it is required to ask cluster-admin to add necessary TLS listener configuration.

## 8. Hard rules

- Never recommend a parameter value before establishing the schema (§2). `proxyProtocol`, `hostPorts`, `envoyService.type`, and `numTrustedHops` are schema-dependent and silently break traffic when wrong.
- Never recommend editing a live `Gateway`, `GatewayClass`, `EnvoyProxy`, or `ClientTrafficPolicy`. The operator reconciles them; change the chart value or `cluster.yaml` option and redeploy.
- Only one `ClientTrafficPolicy` may attach to a given Gateway — custom configuration could be provided through chart values.
- Envoy Gateway on OpenShift is not supported.
- Never recommend splitting edge traffic between Ingress-NGINX and Envoy Gateway. If that schema is found, recommend migrating to the target schema with either only Ingress-NGINX, or only Envoy Gateway.
