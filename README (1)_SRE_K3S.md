# DEVOPS / SRE — Monitoring Lab Guide

### Kubernetes Observability with K3s

**Step-by-step installation and configuration**

> **Stack:** Kubernetes (K3s) • Helm • Prometheus • Grafana • Loki • Alloy

---

## Purpose

Build a small Kubernetes observability stack in an iximiuz Labs K3s playground, collect Kubernetes pod logs with Grafana Alloy, send them to Loki, and view metrics/logs in Grafana.

## Environment

- **Client:** Windows 11 + Ubuntu on WSL2 running the `labctl` client.
- **Remote shell:** iximiuz playground shell (prompt: `laborant@dev-machine`) for `kubectl` and Helm commands.

## Status at the End

Prometheus, Grafana, Loki, and Grafana Alloy were installed; all monitoring pods were `Running`; Loki readiness and labels API checks succeeded; Loki was added to Grafana, and the Grafana dashboard opened at `localhost:3000`.

---

## 1. Lab Architecture and Traffic Flow

```
Kubernetes Nodes & Pods  →  Prometheus (Metrics)  →  Grafana (Dashboards)
                         →  Alloy (Pod Logs)       →  Loki (Log Store)  →  Grafana
```

- **Prometheus** scrapes metrics from Kubernetes components and exporters.
- **Grafana** queries Prometheus for metrics.
- **Grafana Alloy** discovers Kubernetes pods and reads their logs through the Kubernetes API.
- **Alloy** pushes those logs to **Loki**.
- **Grafana** queries Loki when you explore logs.

> **Important:** The lab uses a single-replica Alloy Deployment with Kubernetes API log collection. Loki authentication is disabled and Loki storage is ephemeral. These are lab settings, **not** production recommendations.

---

## 2. Prepare Ubuntu WSL2 and Install labctl

Open PowerShell on Windows and check that Ubuntu is installed:

```bash
wsl -l -v
```

If you need to reset the Ubuntu password, open Ubuntu as root from PowerShell:

```bash
wsl -d Ubuntu -u root
passwd jitendra_reddy
```

Return to the normal Ubuntu user and verify:

```bash
whoami
```

Install the iximiuz `labctl` CLI inside Ubuntu WSL:

```bash
curl -sf https://labs.iximiuz.com/cli/install.sh | sh
```

Authenticate:

```bash
labctl auth login
```

> **Expected:** The login command may say `Already logged in` if authentication is already configured.

---

## 3. Start and Enter Your K3s Playground

Start a K3s playground **only if** you do not already have a running one:

```bash
labctl playground start k3s
```

Record the **Run ID** printed by `labctl`. Use your own current Run ID in the next command:

```bash
labctl ssh YOUR_RUN_ID
```

> **Shell distinction:** The Windows/WSL Ubuntu shell runs `labctl`. After SSH, the remote playground prompt appears as `laborant@dev-machine:~$`. Run `kubectl` and Helm commands in this remote shell.
>
> **Avoid:** Do not start another playground if your current one is already running. Reuse its Run ID.

---

## 4. Check the Kubernetes Cluster

Confirm the Kubernetes connection:

```bash
kubectl get nodes -o wide
kubectl get namespaces
```

Check node resource usage and allocatable resources:

```bash
kubectl top nodes
kubectl describe nodes | grep -A 8 "Allocatable:"
```

> The lab cluster previously showed three nodes: `cplane-01`, `node-01`, and `node-02`, with about **8 allocatable CPUs** and **~12 GiB allocatable memory** in total. Your current numbers may differ.

Create the monitoring namespace:

```bash
kubectl create namespace monitoring
```

> If it already exists, Kubernetes will report that the namespace already exists. That is fine; continue.

---

## 5. Install Helm Chart Repositories

Add the Prometheus Community and Grafana repositories, then update the local chart index:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Add the Grafana Community repository for the Loki chart:

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
```

Verify the repositories and chart:

```bash
helm repo list
helm search repo prometheus-community/kube-prometheus-stack
helm search repo grafana/alloy
```

---

## 6. Install Prometheus and Grafana

### 6.1 Create the Values File

Create a YAML file in the remote playground shell:

```bash
nano monitoring-values.yaml
```

Paste the complete configuration below. In nano, save with `Ctrl+O`, press `Enter`, then exit with `Ctrl+X`.

```yaml
alertmanager:
  enabled: false

grafana:
  enabled: true
  adminUser: admin
  adminPassword: "LabGrafana@2026!"
  persistence:
    enabled: false
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 300m
      memory: 384Mi

prometheus:
  enabled: true
  prometheusSpec:
    replicas: 1
    retention: 6h
    retentionSize: "1GB"
    scrapeInterval: 30s
    evaluationInterval: 30s
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        cpu: 1000m
        memory: 1Gi

prometheusOperator:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 300m
      memory: 256Mi

kube-state-metrics:
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi

prometheus-node-exporter:
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi
```

### 6.2 Install Using Helm

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values monitoring-values.yaml \
  --wait \
  --timeout 10m
```

### 6.3 Verify

```bash
helm list -n monitoring
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Wait until the Grafana, Prometheus, Operator, kube-state-metrics, and node-exporter pods show `Running` and their `READY` columns show all containers ready (e.g., `3/3` or `2/2`).

> **Grafana lab login:** Username `admin`; password `LabGrafana@2026!`. This is a temporary lab password. **Do not reuse it in production.**

---

## 7. Install Loki

### 7.1 Create `loki-values.yaml`

```bash
nano loki-values.yaml
```

Paste this single-binary, filesystem-backed configuration. Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```yaml
deploymentMode: Monolithic

loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  storage:
    type: filesystem
  limits_config:
    allow_structured_metadata: true
    retention_period: 24h

singleBinary:
  replicas: 1
  persistence:
    enabled: false
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 1Gi

gateway:
  enabled: false
lokiCanary:
  enabled: true
chunksCache:
  enabled: false
resultsCache:
  enabled: false

backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0
ingester:
  replicas: 0
querier:
  replicas: 0
queryFrontend:
  replicas: 0
queryScheduler:
  replicas: 0
distributor:
  replicas: 0
compactor:
  replicas: 0
indexGateway:
  replicas: 0
bloomPlanner:
  replicas: 0
bloomBuilder:
  replicas: 0
bloomGateway:
  replicas: 0
```

### 7.2 Install Loki

```bash
helm upgrade --install loki grafana-community/loki \
  --namespace monitoring \
  --values loki-values.yaml \
  --wait \
  --timeout 10m
```

### 7.3 Verify Loki Pods and Service

```bash
kubectl get pods -n monitoring
kubectl get svc loki -n monitoring
```

The Loki service is available inside the cluster at:

```
http://loki.monitoring.svc.cluster.local:3100
```

The log push endpoint used by Alloy is:

```
http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push
```

---

## 8. Install Grafana Alloy to Collect Kubernetes Logs

### 8.1 Create `alloy-values.yaml`

```bash
nano alloy-values.yaml
```

Paste the configuration below. This uses one Alloy Deployment replica. Alloy discovers pods through the Kubernetes API and forwards their logs to Loki. Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```yaml
controller:
  type: deployment
  replicas: 1

alloy:
  mounts:
    varlog: false

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

configMap:
  create: true
  content: |
    logging {
      level = "info"
      format = "logfmt"
    }

    discovery.kubernetes "pods" {
      role = "pod"
    }

    loki.source.kubernetes "pods" {
      targets = discovery.kubernetes.pods.targets
      forward_to = [loki.write.default.receiver]
    }

    loki.write "default" {
      endpoint {
        url = "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"
      }
    }
```

### 8.2 Install Alloy

```bash
helm upgrade --install alloy grafana/alloy \
  --namespace monitoring \
  --values alloy-values.yaml \
  --wait \
  --timeout 10m
```

### 8.3 Verify Alloy

```bash
kubectl get pods -n monitoring
kubectl get deployments -n monitoring
```

Find the Alloy pod/deployment name. In this lab it was `alloy-679b6cffbf-n2zfb` and the pod showed `2/2 Running`. Read its Alloy container logs:

```bash
kubectl logs deployment/alloy -n monitoring -c alloy --tail=50
```

Look for errors:

```bash
kubectl logs deployment/alloy -n monitoring -c alloy --tail=200 | grep -i error
```

> **If `deployment/alloy` is not found:** Use `kubectl get deployments -n monitoring` and `kubectl get pods -n monitoring` to identify the actual name, then substitute it in the logs command.

---

## 9. Verify Loki Is Ready and Receiving Labels

Run a temporary curl pod to check Loki readiness:

```bash
kubectl run loki-check \
  --namespace monitoring \
  --rm -i \
  --restart=Never \
  --image=curlimages/curl \
  -- curl -s http://loki.monitoring.svc.cluster.local:3100/ready
```

**Expected output:**

```
ready
```

Check labels available in Loki:

```bash
kubectl run loki-label-check \
  --namespace monitoring \
  --rm -i \
  --restart=Never \
  --image=curlimages/curl \
  -- curl -s http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/labels
```

The lab returned:

```json
{"status":"success","data":["instance","job","pod","service_name","stream"]}
```

This confirms Loki has indexed label names from ingested log streams. The exact labels depend on the collection configuration and available logs.

---

## 10. Open Grafana at `localhost:3000`

The Grafana service is inside the Kubernetes cluster. The `kubectl port-forward` command exposes its service port 80 as port 3000 in the remote playground shell:

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Keep this command running. To open the remote playground port from Windows localhost, use the `labctl port-forward` feature from a **separate Ubuntu WSL terminal**, using your current playground Run ID:

```bash
labctl playground list
labctl port-forward YOUR_RUN_ID -m dev-machine -L 3000:3000
```

> **Port-forward note:** The second command is run in Ubuntu WSL, **not** inside the `laborant@dev-machine` remote shell. Keep both forwarding processes running while using Grafana. If the `labctl` syntax differs in a newer CLI version, check `labctl --help` and the playground/port-forward help for that installed version.

Open this address in the Windows browser:

```
http://localhost:3000
```

Sign in with:

| Field    | Value              |
| -------- | ------------------ |
| Username | `admin`            |
| Password | `LabGrafana@2026!` |

The Grafana dashboard opened successfully in the lab.

---

## 11. Add Loki as a Grafana Data Source

1. In Grafana, open **Connections**.
2. Choose **Data sources**.
3. Click **Add data source**.
4. Search for and select **Loki**.
5. In the **URL** field, enter the in-cluster Loki service address:

   ```
   http://loki.monitoring.svc.cluster.local:3100
   ```

6. Leave other settings at their defaults for this lab.
7. Click **Save & test**.

> The user confirmed that Loki connected successfully.

---

## 12. View Kubernetes Logs in Grafana Explore

1. In Grafana, open **Explore**.
2. Choose **Loki** as the data source.
3. Select **Code** mode if the query builder is shown.
4. Set the time range to **Last 15 minutes** (or a suitable range).
5. Enter a LogQL query and click **Run query**.

### 12.1 Useful LogQL Examples

```logql
# Logs from the monitoring namespace
{namespace="monitoring"}

# All streams with a pod label
{pod=~".+"}

# Prometheus-related pod logs
{pod=~".*prometheus.*"}

# Grafana pod logs
{pod=~".*grafana.*"}

# Search log lines containing the word "error"
{pod=~".+"} |= "error"

# Loki-related pod logs
{pod=~".*loki.*"}
```

> **If a query returns no logs:** Widen the time range, check that Alloy is `Running`, inspect Alloy logs, and confirm the label names with the Loki labels API. Label names and values are case-sensitive.

---

## 13. Final Verification Checklist

- [ ] `labctl` authenticated and connected to the intended K3s playground
- [ ] `kubectl` can list cluster nodes
- [ ] `monitoring` namespace exists
- [ ] Prometheus and Grafana Helm release installed
- [ ] Loki Helm release installed
- [ ] Alloy Helm release installed
- [ ] All monitoring pods show `Running` and ready containers
- [ ] Loki `/ready` endpoint returned `ready`
- [ ] Loki labels API returned `status` `success`
- [ ] Grafana opened at `http://localhost:3000`
- [ ] Loki data source **Save & test** succeeded
- [ ] Grafana Explore returned Kubernetes log lines

---

## 14. Troubleshooting Quick Reference

| Problem | Checks / Action |
| ------- | --------------- |
| Pod stuck `Pending` / `ContainerCreating` | `kubectl describe pod POD_NAME -n monitoring`; inspect Events, image pulls, and resource availability. |
| Pod is not ready | `kubectl get pods -n monitoring`; `kubectl describe pod POD_NAME -n monitoring`; inspect container logs. |
| Alloy log errors | `kubectl logs deployment/alloy -n monitoring -c alloy --tail=100`; verify its config and Loki service URL. |
| Loki readiness fails | `kubectl get pods,svc -n monitoring`; `kubectl logs loki-0 -n monitoring --all-containers --tail=100`. |
| Grafana cannot reach Loki | Use the in-cluster URL `http://loki.monitoring.svc.cluster.local:3100` in Grafana, **not** `localhost`. |
| Grafana localhost page unavailable | Ensure `kubectl port-forward` is still running and the `labctl` forwarding process is active in WSL. |
| Explore shows no logs | Increase time range; check Alloy logs; inspect labels from `/loki/api/v1/labels`; try `{pod=~".+"}`. |

---

## 15. Lab Limitations and Security Notes

- The Grafana password is stored as plain text in the lab values file. Use a **secret-management approach** and a strong unique password in production.
- Loki `auth_enabled` is `false` in this isolated lab. Production Loki deployments should be protected by authentication and network controls.
- Loki and Grafana **persistence are disabled** in these values files. Data may be lost if pods are recreated or the playground is deleted.
- The Loki setup uses a **single replica** and filesystem storage. It is for learning, not high availability.
- The playground is temporary. **Save your YAML files outside the playground** if you want to keep them after the lab ends.

---

## 16. Screenshot Checkpoints

No actual screenshots were attached in this conversation, so this guide does not pretend to contain screenshots from your machine. Capture these real screenshots while repeating or reviewing the lab, then insert them at the matching sections:

- WSL terminal showing `labctl` authentication and playground SSH connection (Sections 2–3).
- `kubectl get nodes -o wide` showing the K3s nodes (Section 4).
- `helm list -n monitoring` and `kubectl get pods -n monitoring` showing the installed stack (Sections 6–8).
- Loki `/ready` output showing `ready` and labels API output showing `status success` (Section 9).
- Grafana home page open at `localhost:3000` (Section 10).
- Grafana Loki data source page showing **Save & test** success (Section 11).
- Grafana Explore showing log lines for `{namespace="monitoring"}` (Section 12).

---

## 17. What Has Been Completed and What Comes Next

### Completed in this lab

- K3s playground access through `labctl` CLI.
- Prometheus and Grafana installation through Helm.
- Loki installation through Helm.
- Grafana Alloy installation and Kubernetes pod log collection.
- Loki readiness and label verification.
- Grafana access through `localhost:3000` and Loki data source configuration.
- Kubernetes logs explored using LogQL.

### Suggested next practical exercises

- Build a custom Grafana dashboard for node and pod CPU/memory.
- Create Prometheus alert rules for high CPU, memory pressure, and pod restarts.
- Configure Alertmanager and a notification receiver.
- Practice troubleshooting an application using metrics and logs together.

---

**Stack:** Prometheus • Grafana • Loki • Grafana Alloy
