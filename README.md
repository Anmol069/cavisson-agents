# Cavisson Agents

A Helm umbrella chart that deploys Cavisson's monitoring and instrumentation agents into a Kubernetes cluster.

**Namespace:** `cavisson`

---

## Components

### CMON (Cluster Monitor)

Runs as a **DaemonSet** so every node is covered. It collects host-level and container-level metrics (CPU, memory, disk, network, pods, nodes, events) and streams them to the Cavisson NetDiagnostics Controller (NDC) over WSS.

Key behaviours:
- Mounts `/proc`, `/sys`, `/var/log`, and the Docker container log directory from the host
- Uses `hostPID: true` to observe all processes on the node
- Tolerates master/control-plane taints so control-plane nodes are also monitored
- Reads TLS credentials (client cert/key, CA bundle) from a Kubernetes Secret
- Port: `7891` (exposed as a `hostPort`)

### CavProxy (Cavisson ND Proxy)

Runs as a **Deployment** (default: 1 replica). It acts as a local proxy between application agents injected into workloads and the remote NDC, so instrumented pods never need a direct outbound connection to the controller.

Key behaviours:
- Exposes a `ClusterIP` Service on port `443` (maps to container port `7923`)
- Communicates upstream to NDC over WSS
- TLS settings (server CA cert, client cert/key, self-signed/expired/revoked cert policies) are all configurable via `values.yaml`
- The Webhook component is pre-configured to point to `cavproxy.cavisson.svc.cluster.local:443`

### Webhook (ND Webhook Injector)

Runs as a **Deployment** that registers a Kubernetes `MutatingWebhookConfiguration`. It automatically injects Cavisson application agents into matching pods at admission time, with no changes required to application manifests.

Key behaviours:
- Matches pods by namespace regex and/or label selectors (e.g. `com.cavisson/inject-<agent>: "true"`)
- Dynamically derives the tier name from pod metadata
- TLS certificates for the webhook server are auto-generated and valid for `certValidityDays` (default: 1024 days)
- Runs with a hardened security context: non-root, read-only filesystem, no privilege escalation
- `failurePolicy: Ignore` — admission continues even if the webhook is unavailable

---

## Prerequisites

- Kubernetes 1.20+
- Helm 3.x
- A TLS Secret named `tls-secret` in the `cavisson` namespace containing:
  - `clientKey` — RSA private key for the agent
  - `clientCert` — Client certificate
  - `serverCaCert` — CA bundle for server certificate verification

---

## Configuration

Key values to override per component (see each sub-chart's `values.yaml` for the full list):

| Key | Default | Description |
|-----|---------|-------------|
| `cmon.controller` | `""` | NDC host and port (`host:443`) |
| `cmon.tier` | `""` | Tier name reported to the controller |
| `cmon.clusterName` | `""` | Cluster name reported to the controller |
| `cmon.ndcCommProtocol` | `WSS` | Protocol for NDC communication |
| `cmon.tlsSecretName` | `tls-secret` | Secret containing TLS credentials |
| `cavproxy.ndcHost` | `""` | NDC hostname for the proxy to connect to |
| `cavproxy.ndcPort` | `443` | NDC port |
| `cavproxy.replicas` | `1` | Number of CavProxy replicas |
| `cavproxy.tlsSecretName` | `tls-secret` | Secret containing TLS credentials |
| `webhook.certValidityDays` | `1024` | Validity period for the auto-generated webhook TLS cert |
| `webhook.failurePolicy` | `Ignore` | Webhook failure policy (`Ignore` or `Fail`) |
| `webhook.NDController.proxyip` | `cavproxy.cavisson.svc.cluster.local` | CavProxy service address |
| `cmon.enabled` | `true` | Deploy CMON |
| `cavproxy.enabled` | `true` | Deploy CavProxy |
| `webhook.enabled` | `true` | Deploy Webhook |

---

## Install

```bash
helm install cavisson-agents . -n cavisson --create-namespace
```

### Installing into an existing namespace

If the `cavisson` namespace already exists (created manually or by another tool), Helm will throw an ownership conflict error during install. This happens because the namespace lacks the label and annotations Helm uses to track resource ownership.

Run these commands first to hand the namespace over to Helm:

```bash
kubectl label namespace cavisson app.kubernetes.io/managed-by=Helm
kubectl annotate namespace cavisson meta.helm.sh/release-name=cavisson-agents
kubectl annotate namespace cavisson meta.helm.sh/release-namespace=cavisson
```

- `app.kubernetes.io/managed-by=Helm` — marks the namespace as a Helm-managed resource
- `meta.helm.sh/release-name` — tells Helm which release owns this namespace
- `meta.helm.sh/release-namespace` — tells Helm in which namespace the release lives

After applying these, the install command above will work without errors.

## Upgrade

```bash
helm upgrade cavisson-agents . -n cavisson
```

## Uninstall

```bash
helm uninstall cavisson-agents -n cavisson
```

---

## Using Custom Values

To override default configuration, create a custom values file (e.g. `custom-values.yaml`) and pass it with `-f`.

### Install with Custom Values

```bash
helm install cavisson-agents . -n cavisson --create-namespace -f custom-values.yaml
```

### Upgrade with Custom Values

```bash
helm upgrade cavisson-agents . -n cavisson -f custom-values.yaml
```

### Uninstall

```bash
helm uninstall cavisson-agents -n cavisson
```

---

## values.yaml

The top-level `values.yaml` is the **default configuration file for this Helm chart**. It controls the target namespace and which components are deployed.

```yaml
namespace: cavisson

cmon:
  enabled: true
  image:
    repository: docker.artifactory.gmfinancial.com/cavissonsystem/cmon_nfagent-u24
    tag: "4.15.0.101"
    pullPolicy: Always

cavproxy:
  enabled: true
  image:
    repository: cavissonsystem/cavndproxy-alpine
    tag: 4.15.0.B123
    pullPolicy: Always

webhook:
  enabled: true
  image:
    repository: cavissonsystem/nd_webhook_injector
    tag: 4.15.0.B123
    pullPolicy: Always
  instrumentationTemplates:
    - name: dotnet-default-debian
      injectionRules:
        technology: dotnet
        image: cavissonsystem/dotnet-agent-ubuntu:4.15.0.B123
        TierNameSource: auto
        env:
          - name: NDHOME
            value: /opt/cavisson/netdiagnostics
          - name: CORECLR_PROFILER
            value: "{918728DD-259F-4A6A-AC2B-B85E1B658318}"
          - name: CORECLR_ENABLE_PROFILING
            value: "1"
          - name: CORECLR_PROFILER_PATH
            value: /opt/cavisson/netdiagnostics/DotNetC/dll/ILRewriteProfiler.so
          - name: LD_LIBRARY_PATH
            value: /opt/cavisson/netdiagnostics/DotNetC/lib:$(LD_LIBRARY_PATH)
          - name: DOTNET_STARTUP_HOOKS
            value: /opt/cavisson/netdiagnostics/DotNetC/dll/ProfilerHelper.dll
          - name: CAV_APP_AGENT_ENABLE_MONITORING
            value: "1"
          - name: CAV_APP_AGENT_AGENTLOGGINGMODE
            value: "1"
          - name: OTEL_DOTNET_AUTO_EXCLUDE_PROCESSES
            value: ThreadDumpTool
    - name: dotnet-default-alpine
      injectionRules:
        technology: dotnet
        image: cavissonsystem/dotnet-agent-alpine:4.15.0.B123
        TierNameSource: auto
        env:
          - name: NDHOME
            value: /opt/cavisson/netdiagnostics
          - name: CORECLR_PROFILER
            value: "{918728DD-259F-4A6A-AC2B-B85E1B658318}"
          - name: CORECLR_ENABLE_PROFILING
            value: "1"
          - name: CORECLR_PROFILER_PATH
            value: /opt/cavisson/netdiagnostics/DotNetC/dll/ILRewriteProfiler.so
          - name: LD_LIBRARY_PATH
            value: /opt/cavisson/netdiagnostics/DotNetC/lib:$(LD_LIBRARY_PATH)
          - name: DOTNET_STARTUP_HOOKS
            value: /opt/cavisson/netdiagnostics/DotNetC/dll/ProfilerHelper.dll
          - name: CAV_APP_AGENT_ENABLE_MONITORING
            value: "1"
          - name: CAV_APP_AGENT_AGENTLOGGINGMODE
            value: "1"
          - name: OTEL_DOTNET_AUTO_EXCLUDE_PROCESSES
            value: ThreadDumpTool
    - name: python-default
      injectionRules:
        technology: python
        image: cavissonsystem/python-agent:4.15.0.B117
        TierNameSource: auto
    - name: java-default
      injectionRules:
        technology: java
        image: cavissonsystem/java-agent:4.15.0.131
        javaEnvVar: JAVA_TOOL_OPTIONS
        TierNameSource: auto
        env:
          - name: NDHOME
            value: /opt/cavisson/netdiagnostics
          - name: CAV_APP_AGENT_AGENTLOGGINGMODE
            value: "ALL"
          - name: CAV_APP_AGENT_NDCHOST
            value: "netcloud4-15-u24.cav-test.com"
          - name: CAV_APP_AGENT_NDCPORT
            value: "4433"
          - name: CAV_APP_AGENT_NDC_COMM_PROTOCOL
            value: "WSS"
    - name: nodejs-default
      injectionRules:
        technology: nodejs
        image: cavissonsystem/nodejs-agent:4.15.0.B132
        TierNameSource: auto
        env:
          - name: CAV_APP_AGENT_AGENTLOGGINGMODE
            value: "ALL"
          - name: CAV_APP_AGENT_NDCHOST
            value: "netcloud4-15-u24.cav-test.com"
          - name: CAV_APP_AGENT_NDCPORT
            value: "4433"
          - name: CAV_APP_AGENT_NDC_COMM_PROTOCOL
            value: "WSS"
```

| Field | Default | Description |
|-------|---------|-------------|
| `namespace` | `cavisson` | Kubernetes namespace where all chart resources are deployed |
| `cmon.enabled` | `true` | Deploy the CMON (Cavisson Monitor) DaemonSet |
| `cavproxy.enabled` | `true` | Deploy the CavProxy Deployment and Service |
| `webhook.enabled` | `true` | Deploy the Webhook Injector and MutatingWebhookConfiguration |

All image tags and version-sensitive env paths for every component — including the agent images injected by the webhook — are managed from this file. To upgrade to a new version, update the relevant `image.tag` (and `env` paths if they changed) and run `helm upgrade cavisson-agents . -n cavisson`.

To selectively disable a component at install time:

```bash
# Deploy only CavProxy (skip CMON and Webhook)
helm install cavisson-agents . -n cavisson --create-namespace \
  --set cmon.enabled=false \
  --set webhook.enabled=false
```
