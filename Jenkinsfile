pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    timestamps()
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['DEPLOY', 'UNINSTALL'],
      description: 'Install or upgrade monitoring, or uninstall it from the target AKS cluster.'
    )
    string(
      name: 'AZURE_RESOURCE_GROUP',
      defaultValue: 'rg-aks-test',
      description: 'Azure resource group containing the existing AKS cluster.'
    )
    string(
      name: 'AKS_CLUSTER_NAME',
      defaultValue: 'aks-test-cluster',
      description: 'Name of the existing AKS cluster.'
    )
    string(
      name: 'NAMESPACE',
      defaultValue: 'monitoring',
      description: 'Kubernetes namespace for Prometheus, Grafana, and Alertmanager.'
    )
    string(
      name: 'CHART_VERSION',
      defaultValue: '72.6.2',
      description: 'Pinned prometheus-community/kube-prometheus-stack Helm chart version.'
    )
    booleanParam(
      name: 'DELETE_PVC',
      defaultValue: false,
      description: 'On UNINSTALL only, also delete monitoring PVCs and their stored data.'
    )
  }

  environment {
    AZURE_CONFIG_DIR = "${WORKSPACE}/.azure"
    KUBECONFIG       = "${WORKSPACE}/.kubeconfig"
    RELEASE_NAME     = 'aks-observability'
  }

  stages {
    stage('Validate agent') {
      steps {
        sh '''
          set -eu
          command -v az >/dev/null
          command -v kubectl >/dev/null
          command -v helm >/dev/null
          test -f values.yaml
          test -f dashboards/aks-infrastructure.json
          test -n "$AZURE_RESOURCE_GROUP"
          test -n "$AKS_CLUSTER_NAME"
          test -n "$NAMESPACE"
          test -n "$CHART_VERSION"
        '''
      }
    }

    stage('Connect to AKS') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'azure-service-principal',
            usernameVariable: 'ARM_CLIENT_ID',
            passwordVariable: 'ARM_CLIENT_SECRET'
          ),
          string(credentialsId: 'azure-subscription-id', variable: 'ARM_SUBSCRIPTION_ID'),
          string(credentialsId: 'azure-tenant-id', variable: 'ARM_TENANT_ID')
        ]) {
          sh '''
            set -eu
            set +x
            umask 077
            mkdir -p "$AZURE_CONFIG_DIR"
            az login --service-principal \
              --username "$ARM_CLIENT_ID" \
              --password "$ARM_CLIENT_SECRET" \
              --tenant "$ARM_TENANT_ID" \
              --output none
            az account set --subscription "$ARM_SUBSCRIPTION_ID"
            az aks get-credentials \
              --resource-group "$AZURE_RESOURCE_GROUP" \
              --name "$AKS_CLUSTER_NAME" \
              --file "$KUBECONFIG" \
              --overwrite-existing
            kubectl cluster-info
          '''
        }
      }
    }

    stage('Deploy monitoring') {
      when {
        expression { params.ACTION == 'DEPLOY' }
      }
      steps {
        withCredentials([
          string(credentialsId: 'grafana-admin-password', variable: 'GRAFANA_ADMIN_PASSWORD')
        ]) {
          sh '''
            set -eu
            set +x
            test -n "$GRAFANA_ADMIN_PASSWORD"
            kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
            kubectl create secret generic grafana-admin \
              --namespace "$NAMESPACE" \
              --from-literal=admin-user=admin \
              --from-literal=admin-password="$GRAFANA_ADMIN_PASSWORD" \
              --dry-run=client -o yaml | kubectl apply -f -

            helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
            helm repo update prometheus-community
            helm upgrade --install "$RELEASE_NAME" \
              prometheus-community/kube-prometheus-stack \
              --namespace "$NAMESPACE" \
              --version "$CHART_VERSION" \
              --values values.yaml \
              --set-file grafana.dashboards.default.aks-infrastructure.json=dashboards/aks-infrastructure.json \
              --wait \
              --timeout 15m
          '''
        }
      }
    }

    stage('Verify monitoring') {
      when {
        expression { params.ACTION == 'DEPLOY' }
      }
      steps {
        sh '''
          set -eu
          kubectl get pods --namespace "$NAMESPACE"
          kubectl get services --namespace "$NAMESPACE"
          kubectl get prometheusrules --namespace "$NAMESPACE"
        '''
      }
    }

    stage('Uninstall monitoring') {
      when {
        expression { params.ACTION == 'UNINSTALL' }
      }
      steps {
        sh '''
          set -eu
          helm uninstall "$RELEASE_NAME" --namespace "$NAMESPACE" --wait --timeout 10m
          kubectl delete secret grafana-admin --namespace "$NAMESPACE" --ignore-not-found
          if [ "$DELETE_PVC" = "true" ]; then
            kubectl delete pvc --namespace "$NAMESPACE" \
              --selector "app.kubernetes.io/instance=$RELEASE_NAME" \
              --ignore-not-found
          fi
        '''
      }
    }
  }

  post {
    always {
      sh '''
        rm -rf "$AZURE_CONFIG_DIR"
        rm -f "$KUBECONFIG"
      '''
    }
  }
}
