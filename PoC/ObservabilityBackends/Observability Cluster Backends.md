# Prerequisites
## Kubernetes
Everything in this article tested on [[06. Dev packages#Rancher Desktop|Rancher Desktop]] version 1.17.0 on Fedora 41 Linux. This version of Rancher has some issues on MacOs, Windows and some Linux distro. See latest release notes of Rancher for more details.
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
### Install with helm
```shell
helm upgrade -i \
  grafana-operator oci://ghcr.io/grafana/helm-charts/grafana-operator \
  --namespace monitoring \
  --create-namespace \
  --version v5.16.0
```
### Create grafana instance
```shell
kubectl apply -n monitoring -f - <<EOF
---
apiVersion: grafana.integreatly.org/v1beta1
kind: Grafana
metadata:
  name: k8s-grafana
  labels:
    dashboards: "grafana"
spec:
  config:
    security:
      admin_user: root
      admin_password: secret
    log:
      mode: "console"
      level: "error"
  ingress:
    spec:
      ingressClassName: traefik
      rules:
        - host: grafana-monitoring.localhost.localdomain
          http:
            paths:
              - backend:
                  service:
                    name: k8s-grafana-service
                    port:
                      number: 3000
                path: /
                pathType: Prefix
---
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: grafanadatasource-vmsingle
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  datasource:
    name: vmsingle
    type: prometheus
    access: proxy
    url: http://vmsingle-k8s-vmsingle:8429
    isDefault: true
    jsonData:
      "tlsSkipVerify": true
      "timeInterval": "5s"
EOF
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
### Create VMSingle instance
```shell
kubectl apply -n monitoring -f - <<EOF
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMSingle
metadata:
  name: k8s-vmsingle
spec:
  retentionPeriod: 2d
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "1000m"
  storage:
    storageClassName: local-path
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  removePvcAfterDelete: true
  extraArgs:
    dedup.minScrapeInterval: 60s
EOF
```
### Create ingress
```shell
kubectl create ingress k8s-vmsingle -n monitoring --class=traefik --rule="vmsingle-monitoring.localhost.localdomain/*=vmsingle-k8s-vmsingle:8429"
```
### Scrape Metrics Server
```shell
kubectl apply -n monitoring -f - <<EOF
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: metrics-server
spec:
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      kubernetes.io/name: Metrics-server
  endpoints:
  - port: "443"
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view-metrics
rules:
- apiGroups:
    - metrics.k8s.io
  resources:
    - pods
    - nodes
  verbs:
    - get
    - list
    - watch
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view-metrics
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: system:serviceaccount:vm:vmagent
EOF
```
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
  kubectl create -n $NAMESPACE secret docker-registry ghcr-io-secret --docker-username="$GITHUB_USER" --docker-password="$GITHUB_PAT" --docker-server='https://ghcr.io/v1/'
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
  --set 'collector.imagePullSecrets[0].name=ghcr-io-secret' \
  --set 'query.imagePullSecrets[0].name=ghcr-io-secret' \
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
Add helm repo
```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts --force-update
```
Install with helm
```shell
helm upgrade --install \
  opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace opentelemetry-operator-system \
  --create-namespace \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib" \
  --set 'manager.extraArgs={--enable-go-instrumentation=true,--enable-nginx-instrumentation=true}' \
```
### Create an OpenTelemetry Collector
Otelcol instance should be created in required namespace
```shell
NAMESPACE=robot-shop
```

```shell
kubectl apply -n ${NAMESPACE:-default} -f - <<EOF
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
spec:
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        send_batch_size: 10000
        timeout: 10s
    exporters:
      debug: {}
      otlp:
        endpoint: "jaeger-collector.jaeger.svc.cluster.local:4317"
        tls:
          insecure: true
      prometheus:
        endpoint: "0.0.0.0:8889"
      prometheusremotewrite:  
        endpoint: "http://vmsingle-k8s-vmsingle.monitoring.svc.cluster.local:8429/api/v1/write"
        tls:
          insecure: true
        headers:
          Content-Type: application/x-protobuf
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug, otlp]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheusremotewrite, debug]
EOF
```