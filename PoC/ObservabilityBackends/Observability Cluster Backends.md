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
curl -s https://raw.githubusercontent.com/Netcracker/qubership-monitoring-operator/refs/heads/main/charts/qubership-monitoring-operator/charts/grafana-operator/crds/integreatly.org_grafanadashboards.yaml | kubectl apply -f -
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

#### Troubleshooting

1) _Problem description_: docker-credential-secretservice: error while loading shared libraries: libsecret-1.so.0:
   cannot open shared object file: No such file or directory  
   _Solution_: run command
   ```shell
   sudo apt-get update
   sudo apt-get install libsecret-1-0
   ```
2) _Problem description_: Error: error getting credentials - err: exec: "docker-credential-wincred.exe": executable file
   not found in PATH, out: ``  
   _Solution_: need to configure credential helper for Docker in Windows environment. Install docker-credential-wincred.exe then set the path to $PATH variable.
   ```shell
   echo 'export PATH=$PATH:"/mnt/c/Program Files/Rancher Desktop/resources/resources/win32/bin"' >> ~/.bashrc
   source ~/.bashrc
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
#### Troubleshooting

1) _Problem description_:
   ```yaml
   Error: Unable to continue with install: CustomResourceDefinition "vmauths.operator.victoriametrics.com" in namespace
   "" exists and cannot be imported into the current release: invalid ownership metadata; label validation error:
   missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key
   "meta.helm.sh/release-name": must be set to "vmo"; annotation validation error: missing key
   "meta.helm.sh/release-namespace": must be set to "monitoring"
   ```
   _Reason_: The problem was incompatibility or conflict of CRD and resource versions  
   _Solution_: Try to run [Script For Deleting CRDs](https://docs.percona.com/everest/uninstall/uninstallEverest.html#remove-all-the-crds)
   and rerun helm install command above

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
### Create VMAgent instance
```shell
kubectl apply -n monitoring -f - <<EOF
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAgent
metadata:
  name: sample
spec:
  selectAllByDefault: true
  replicaCount: 1
  resources:
    requests:
      cpu: "50m"
      memory: "350Mi"
    limits:
      cpu: "500m"
      memory: "850Mi"
  extraArgs:
    memory.allowedPercent: "40"
  remoteWrite:
  - url: "http://vmsingle-k8s-vmsingle.monitoring.svc.cluster.local:8429/api/v1/write"
EOF
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
  --set jaeger.prometheusMonitoring=false \
  --set query.ingress.install=true \
  --set query.ingress.host=query.jaeger.localhost.localdomain \
  --set collector.ingress.install=true \
  --set collector.ingress.host=collector.jaeger.localhost.localdomain \
  --set "cassandraSchemaJob.host=$CASSANDRA_SVC" \
  --set "cassandraSchemaJob.username=$CASSANDRA_USER" \
  --set "cassandraSchemaJob.password=$CASSANDRA_PASSWORD" \
  --set cassandraSchemaJob.datacenter=dc1 #\
#  --set integrationTests.install=true
#  --set 'collector.imagePullSecrets[0].name=ghcr-io-secret' \
#  --set 'query.imagePullSecrets[0].name=ghcr-io-secret' \

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
## Graylog logging-operator
### Opensearch install
To successfully deploy the logging operator, will need an opensearch database. MongoDB will setup while deploying 
graylog service. Update the value for *opensearch.master.persistence.storageClass* in advance and a quick installation 
can be performed based on a ready-made configuration file (just specify it as the -f flag): 
[config-file](https://github.com/Netcracker/qubership-opensearch/blob/main/charts/helm/opensearch-service/example.yaml)

Install with helm:
```shell
helm upgrade --install opensearch-service \
  --namespace=opensearch \
  --create-namespace \
  --set opensearch.master.persistence.storageClass=local-path \
  --set opensearch.master.replicas=1 \
  --set opensearch.tls.enabled=false \
  charts/helm/opensearch-service \
  -f charts/helm/opensearch-service/example.yaml
```

### Install the logging operator. 
If opensearch is built according to the instructions above, then replacing 
graylog.elasticsearchHost is not required. Here is an example of a helm command to install logging-operator with 
**fluentbit**:

```shell
helm upgrade --install qubership-logging-operator \
  --namespace=logging \
  --create-namespace \
  --set graylog.install=true \
  --set graylog.storageSize=10Gi \
  --set graylog.host=http://graylog.demo.qubership.org \
  --set graylog.initContainerDockerImage=alpine:3.17.2 \
  --set graylog.mongoStorageClassName=local-path \
  --set graylog.graylogStorageClassName=local-path \
  --set graylog.elasticsearchHost=http://admin:admin@opensearch.opensearch:9200 \
  --set fluentbit.install=true \
  --set fluentbit.graylogHost=graylog-service.logging \
  --set fluentbit.graylogPort=12201 \
  --set fluentbit.systemAuditLogging=false \
  --set cloudEventsReader.install=false \
  charts/qubership-logging-operator
```

Example of a helm command to install logging-operator with **fluentd**:

```shell
helm upgrade --install qubership-logging-operator \
  --namespace=logging \
  --create-namespace \
  --set graylog.install=true \
  --set graylog.storageSize=10Gi \
  --set graylog.host=http://graylog.demo.qubership.org \
  --set graylog.initContainerDockerImage=alpine:3.17.2 \
  --set graylog.mongoStorageClassName=local-path \
  --set graylog.graylogStorageClassName=local-path \
  --set graylog.elasticsearchHost=http://admin:admin@opensearch.opensearch:9200 \
  --set fluentd.install=true \
  --set fluentd.graylogHost=graylog-service.logging \
  --set fluentd.graylogPort=12201 \
  --set cloudEventsReader.install=false \
  charts/qubership-logging-operator
```