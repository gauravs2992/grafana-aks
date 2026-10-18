# Grafana and Prometheus on AKS

This Jenkins pipeline installs the `kube-prometheus-stack` Helm chart into an **existing** AKS cluster. The chart includes Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter. It does not create or destroy the AKS cluster; that lifecycle remains with the Terraform pipeline in `terraform-aks`.

## What is installed

- A provisioned **AKS Infrastructure** Grafana dashboard with cluster CPU and memory, node count/readiness, and node utilization panels.
- Prometheus rules for NotReady nodes, sustained high node CPU, low node memory, CrashLooping pods, and nearly full node filesystems.
- Grafana is exposed by an Azure public LoadBalancer restricted to the Jenkins VM's public outbound IPv4 address (`/32`). Prometheus and Alertmanager remain private in-cluster services.
- Persistent claims for Grafana (10 GiB), Prometheus (20 GiB, 15-day retention), and Alertmanager (2 GiB). The cluster's default storage class must support dynamically provisioned `ReadWriteOnce` volumes.

These are Kubernetes node and workload statistics. AKS control-plane metrics are not exposed by node-exporter or kube-state-metrics; use Azure Monitor managed Prometheus and Azure Managed Grafana if control-plane metrics are required.

## Jenkins setup

Create a Pipeline job whose SCM repository root is `grafana-aks` and whose script path is `Jenkinsfile`. Run it on a **Linux** Jenkins agent with Azure CLI (`az`), `kubectl`, and Helm 3 installed. The agent must be able to reach the AKS API endpoint over HTTPS (TCP 443). Keep the Jenkins agent's Helm repository cache between builds where possible: the pipeline adds the pinned chart repository only when missing and does not refresh the full chart index on every run.

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
| `JENKINS_PUBLIC_IP` | *(required for `DEPLOY`)* | Jenkins VM public outbound IPv4 address; the Grafana service allows only this address |
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
6. Grafana is exposed by an Azure public LoadBalancer, with Kubernetes `loadBalancerSourceRanges` restricting inbound access to the Jenkins VM public outbound IP as a single-address `/32`. Prometheus and Alertmanager are not exposed publicly.

## View the dashboard and alerts

On `DEPLOY`, Jenkins applies the chart, waits up to 5 minutes for monitoring pods to become Ready, then waits up to 5 minutes for Azure to assign the Grafana service's external IP. Each check has its own stage so the build log shows which operation is taking time. On a pod timeout, it prints pod and PVC status for diagnosis. The build prints the Grafana URL:

```text
Grafana GUI URL: http://<EXTERNAL-IP>/
```

Open the printed URL from a client whose outbound internet traffic uses the configured `JENKINS_PUBLIC_IP`, then sign in as `admin` with the password stored in the Jenkins credential `grafana-admin-password`. The dashboard is in **Dashboards → AKS Infrastructure**. You can also retrieve the address later with `kubectl -n monitoring get service aks-observability-grafana`; replace `monitoring` if you selected another namespace.

The Jenkins IP must be the public **outbound/source** address seen by the Azure LoadBalancer, not necessarily an IP attached to the VM if outbound traffic is SNATed through a NAT Gateway, firewall, or proxy. If your browser does not egress through that address, the firewall blocks it; use a browser/network routed through Jenkins' egress or change the allowlist to your browser's fixed public IP. This chart serves Grafana over HTTP, not HTTPS; the restricted allowlist reduces exposure but does not encrypt login traffic. For production use, terminate TLS with a trusted certificate before exposing Grafana.

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
