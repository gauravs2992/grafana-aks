# Grafana and Prometheus on AKS

This Jenkins pipeline installs the `kube-prometheus-stack` Helm chart into an **existing** AKS cluster. The chart includes Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter. It does not create or destroy the AKS cluster; that lifecycle remains with the Terraform pipeline in `terraform-aks`.

## What is installed

- A provisioned **AKS Infrastructure** Grafana dashboard with cluster CPU and memory, node count/readiness, and node utilization panels.
- Prometheus rules for NotReady nodes, sustained high node CPU, low node memory, CrashLooping pods, and nearly full node filesystems.
- Private `ClusterIP` services for Grafana, Prometheus, and Alertmanager. Nothing is exposed through a public LoadBalancer or Ingress.
- Persistent claims for Grafana (10 GiB), Prometheus (20 GiB, 15-day retention), and Alertmanager (2 GiB). The cluster's default storage class must support dynamically provisioned `ReadWriteOnce` volumes.

These are Kubernetes node and workload statistics. AKS control-plane metrics are not exposed by node-exporter or kube-state-metrics; use Azure Monitor managed Prometheus and Azure Managed Grafana if control-plane metrics are required.

## Jenkins setup

Create a Pipeline job whose SCM repository root is `grafana-aks` and whose script path is `Jenkinsfile`. Run it on a **Linux** Jenkins agent with Azure CLI (`az`), `kubectl`, and Helm 3 installed. The agent must be able to reach the AKS API endpoint over HTTPS (TCP 443).

Configure these Jenkins credentials:

| Credential ID | Jenkins type | Contents |
| --- | --- | --- |
| `azure-service-principal` | Username with password | Username is the Azure service principal client ID; password is its client secret |
| `azure-subscription-id` | Secret text | Target Azure subscription ID |
| `azure-tenant-id` | Secret text | Service principal tenant ID |
| `grafana-admin-password` | Secret text | Strong Grafana admin password; used only during `DEPLOY` |

The service principal needs the **Azure Kubernetes Service Cluster User Role** on the AKS resource to retrieve user credentials. Its AKS identity must also be authorized to install cluster-scoped Kubernetes resources (CRDs, ClusterRoles, and webhooks) and namespaced resources. For clusters using Azure RBAC for Kubernetes authorization, grant **Azure Kubernetes Service RBAC Cluster Admin** on the cluster to this trusted deployment identity. For Kubernetes RBAC authorization, bind the identity to a suitably privileged Kubernetes role. Do not use cluster-admin for human dashboard viewers. For private AKS, run the Jenkins agent in a network that can route to the cluster VNet and resolve the private API DNS name. Entra-integrated clusters may also require `kubelogin` on the agent.

The Jenkins parameters are:

| Parameter | Default | Meaning |
| --- | --- | --- |
| `ACTION` | `DEPLOY` | `DEPLOY` installs/upgrades the stack; `UNINSTALL` removes the Helm release |
| `AZURE_RESOURCE_GROUP` | `rg-aks-test` | Resource group of the existing AKS cluster |
| `AKS_CLUSTER_NAME` | `aks-test-cluster` | Existing AKS cluster name |
| `NAMESPACE` | `monitoring` | Namespace for all monitoring components |
| `CHART_VERSION` | `72.6.2` | Pinned `kube-prometheus-stack` Helm chart version |
| `DELETE_PVC` | `false` | With `UNINSTALL`, explicitly delete the monitoring PVCs and their data |

Change the resource group and cluster name to match your deployment. The defaults match the Terraform AKS example in `terraform-aks`.

## How cluster connectivity works

1. Jenkins authenticates to Azure Resource Manager using the service principal and selects the configured subscription.
2. `az aks get-credentials` retrieves a **user** kubeconfig for the named AKS cluster. The kubeconfig and Azure CLI profile are stored temporarily in the Jenkins workspace and removed in the pipeline's `post` cleanup.
3. `kubectl` and Helm use that kubeconfig to talk directly to the Kubernetes API endpoint over TLS/HTTPS (TCP 443). Azure login by itself does not provide network access: the agent must have a route to the endpoint and Kubernetes authorization.
4. Helm installs the chart resources into the `monitoring` namespace (or the selected namespace). Prometheus discovers Kubernetes targets and scrapes kubelet/container and node-exporter metrics; kube-state-metrics exposes Kubernetes object state.
5. Grafana queries the in-cluster Prometheus service. Prometheus evaluates the provisioned rules and sends firing alerts to the in-cluster Alertmanager service.
6. Dashboard access is through a deliberate, authenticated operator path such as `kubectl port-forward`; no public service is created by this configuration.

## View the dashboard and alerts

From an authorized machine with cluster network access and a suitable kubeconfig, forward Grafana locally:

```sh
kubectl -n monitoring port-forward service/aks-observability-grafana 3000:80
```

These commands use the default `NAMESPACE=monitoring`; replace `monitoring` if you deployed into a different namespace. Open `http://localhost:3000`, sign in as `admin`, and use the password stored in the Jenkins credential `grafana-admin-password`. The dashboard is in **Dashboards → AKS Infrastructure**.

Inspect active and pending alert rules in Grafana under **Alerting → Alert rules**, or forward Alertmanager:

```sh
kubectl -n monitoring port-forward service/aks-observability-kube-prometheus-alertmanager 9093:9093
```

Alertmanager will receive the included Prometheus alerts, but the default chart receiver does not deliver notifications to an external system. To get email, Teams, Slack, or webhook notifications, configure an Alertmanager receiver and route in `values.yaml` using the chart's `alertmanager.config` settings. Store destination credentials securely (for example, Kubernetes Secrets), never in this repository, then run `DEPLOY` to apply the configuration.

## Customize dashboards and alerts

- Edit `dashboards/aks-infrastructure.json` to add Grafana panels or PromQL queries; the pipeline supplies it to Helm using `--set-file`.
- Edit `additionalPrometheusRulesMap` in `values.yaml` to add or change Prometheus alert rules. Thresholds and `for` durations are examples; tune them to your workload and capacity.
- Change the storage sizes, retention, or chart version in `values.yaml` and the Jenkins `CHART_VERSION` parameter as appropriate.

## Uninstall and data retention

Run the pipeline with `ACTION=UNINSTALL`. Helm removes the release and the Jenkins pipeline removes its Grafana admin secret. Persistent volume claims are intentionally retained by default; they can retain monitoring data and incur storage charges. Set `DELETE_PVC=true` only when you intend to permanently remove **all** PVCs in the selected namespace belonging to this release. The namespace is not deleted.

The pipeline does not delete the AKS cluster, its Terraform state, or other resources in the selected namespace.
