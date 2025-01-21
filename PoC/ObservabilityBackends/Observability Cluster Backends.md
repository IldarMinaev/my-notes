# Prerequisites
## Cert Manager
See documentation [https://cert-manager.io/docs/installation/helm/](https://cert-manager.io/docs/installation/helm/)
Add helm repo
```shell
helm repo add jetstack https://charts.jetstack.io --force-update
```
Install with helm
```shell
helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.3 \
  --set prometheus.enabled=true \
  --set crds.enabled=true
```
## Cassandra
See documentation [https://docs.k8ssandra.io/install/local/single-cluster-helm/](https://docs.k8ssandra.io/install/local/single-cluster-helm/)
### Add helm repo
```shell
helm repo add k8ssandra https://helm.k8ssandra.io/stable --force-update
```
### Install with helm
```shell
helm upgrade --install \
  k8ssandra-operator k8ssandra/k8ssandra-operator \
  --namespace k8ssandra-operator \
  --create-namespace
```
## Custom Resource Definitions
Install CRDs for servicemonitors and pod monitoris
```shell
curl -s https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/example/prometheus-operator-crd-full/monitoring.coreos.com_servicemonitors.yaml | kubectl apply -f -
curl -s https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/example/prometheus-operator-crd-full/monitoring.coreos.com_podmonitors.yaml | kubectl apply -f -
```
Install CRDs for grafana dashboards
```shell
TODO TBD
```
### Deploy the K8ssandraCluster
```shell
cat <<EOF | kubectl apply -n k8ssandra-operator -f -
apiVersion: k8ssandra.io/v1alpha1
kind: K8ssandraCluster
metadata:
  name: cassandra
spec:
  cassandra:
    serverVersion: "4.0.1"
    datacenters:
      - metadata:
          name: dc1
        size: 1
        storageConfig:
          cassandraDataVolumeClaimSpec:
            storageClassName: local-path
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 5Gi
        config:
          jvmOptions:
            heapSize: 512M
EOF
```
# Back-ends
## Grafana operated
See [https://grafana.github.io/grafana-operator/docs/installation/helm/](https://grafana.github.io/grafana-operator/docs/installation/helm/)
```shell
helm upgrade -i \
  grafana-operator oci://ghcr.io/grafana/helm-charts/grafana-operator \
  --namespace monitoring \
  --create-namespace \
  --version v5.16.0
```
TODO add creation of grafana https://github.com/grafana/grafana-operator/blob/master/examples/grafana_deployment/resources.yaml
## Victoria-Metrics operator
See [https://docs.victoriametrics.com/helm/victoriametrics-operator/index.html](https://docs.victoriametrics.com/helm/victoriametrics-operator/index.html)
Add a chart helm repository
```shell
helm repo add vm https://victoriametrics.github.io/helm-charts/ --force-update
```
Install with helm
```shell
helm upgrade --install \
  vmo vm/victoria-metrics-operator \
  -n monitoring \
  --create-namespace
```
TODO Add creation of VM Stack

## Jaeger
See https://github.com/Netcracker/qubership-jaeger/
Clone repo
```shell
git clone https://github.com/Netcracker/qubership-jaeger/
cd qubership-jaeger/
```
Qubership-jaeger uses integration-tests and readiness probes images published in authorized registry `ghcr.io`. So need to create credentials in kubernetes cluster to access the registry. Put your Github account details. See more details in [[10. Working with GHCR.IO]]
```shell
NAMESPACE=jaeger
kubectl create namespace $NAMESPACE
echo -n Enter your GITHUB_PAT:\  &&\
  read -s GITHUB_PAT && echo && \
  echo -n Enter your GitHub Account:\  && \
  read GITHUB_USER && echo && \
  echo -n Enter your GitHub E-Mail:\  && \
  read GITHUB_EMAIL && echo && \
  kubectl create -n $NAMESPACE secret docker-registry ghcr-io-secret --docker-username="$GITHUB_USER" --docker-password="$GITHUB_PAT" --docker-email="$GITHUB_EMAIL"
```
Install with helm
```shell
CASSANDRA_USER=$(kubectl get secret -n k8ssandra-operator cassandra-superuser -o json | jq -r '.data.username' | base64 -d)
CASSANDRA_PASSWORD=$(kubectl get secret -n k8ssandra-operator cassandra-superuser -o json | jq -r '.data.password' | base64 -d)
CASSANDRA_SVC=cassandra-dc1-service.k8ssandra-operator.svc.cluster.local
helm upgrade --install \
  qubership-jaeger charts/qubership-jaeger/ \
  --namespace jaeger \
  --create-namespace \
  --set jaeger.prometheusMonitoringDashboard=false \
  --set 'collector.readinessProbe.imagePullSecrets[0].name=ghcr-io-secret' \
  --set 'query.readinessProbe.imagePullSecrets[0].name=ghcr-io-secret' \
  --set query.ingress.install=true \
  --set query.ingress.host=query.jaeger.localhost.localdomain \
  --set collector.ingress.install=true \
  --set collector.ingress.host=collector.jaeger.localhost.localdomain \
  --set "cassandraSchemaJob.host=$CASSANDRA_SVC" \
  --set "cassandraSchemaJob.username=$CASSANDRA_USER" \
  --set "cassandraSchemaJob.password=$CASSANDRA_PASSWORD" \
  --set cassandraSchemaJob.datacenter=dc1
```
## Open-telemetry operator
Install with helm
```shell

```
Create observations
```shell

```