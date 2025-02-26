Prerequisites:
kind, kubectl, helm installed

System requirements:
https://docs.cilium.io/en/stable/operations/system_requirements/

Cilium requires some kernel modules to be included or loaded dynamically. So before installing cilium need to build a kernel with all required modules listed in the above link with system requirements and proceed according to either of these 2 articles:

https://wsl.dev/wslcilium/

https://jackliusr.github.io/posts/2024/03/setup-kind-cluster-using-cilium-cni-in-wsl2/

### Kind cluster installation
#### Create caching proxy registry for docker images
```sh
docker run -d --name proxy --restart=always --net=kind \
  -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io registry:2
docker run -d --name registry --restart=always --net=kind \
  -p 5000:5000 registry:2
```

#### Create cluster with kind
``` bash
kind create cluster --config=/kind-config.yaml
```
kind-config.yaml:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  # port forward 80 on the host to 80 on this node
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
    listenAddress: 127.0.0.1
  - containerPort: 443
    hostPort: 443
    protocol: TCP
    listenAddress: 127.0.0.1
  # Port forward for http on shared ingress LoadBalancer service
  - containerPort: 30000
    hostPort: 30000
    listenAddress: 127.0.0.1 
  # Port forward for https on shared ingress LoadBalancer service
  - containerPort: 32767
    hostPort: 32767
    listenAddress: 127.0.0.1
  # Port forward for gatewayAPI test purposes
  - containerPort: 30001
    hostPort: 30001
    listenAddress: 127.0.0.1      
networking:
  disableDefaultCNI: true
  kubeProxyMode: "none"
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["http://proxy:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
      endpoint = ["http://proxy:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."ghcr.io"]
      endpoint = ["http://proxy:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5000"]
      endpoint = ["http://registry:5000"]
```
#### Export kind kubeconfig
```sh
kind get kubeconfig > ~/.kube/kind-cluster.yaml
export KUBECONFIG=~/.kube/kind-cluster.yaml
```
### Cilium installation and configuration
#### Install v1.2.0 gatewayAPI crds:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_grpcroutes.yaml
```
#### Install cilium:

``` bash 
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" --values - <<EOF
kubeProxyReplacement: true
k8sServiceHost: kind-control-plane
k8sServicePort: 6443
devices: 'eth0'
ipv6:
  enabled: false
l2announcements:
  enabled: true
ingressController:
  enabled: true
  loadbalancerMode: 'shared'
hostServices:
  enabled: false
externalIPs:
  enabled: true
nodePort:
  enabled: true
hostPort:
  enabled: true
image:
  pullPolicy: IfNotPresent
ipam:
  mode: 'cluster-pool'
gatewayAPI:
  enabled: true
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    ingress:
      enabled: true
      hosts:
        - hubble-ui.demo.qubership.org
  metrics:
    enableOpenMetrics: true
prometheus:
  enabled: true
operator:
  prometheus:
    enabled: true
EOF
```
#### Create IP pool for LoadBalancer services
Choose any IP range for LoadBalancer services (e.g. from kind docker network 172.19.0.0/24)
```sh
kubectl apply -f - <<EOF
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
    - start: "172.19.0.10"
      stop: "172.19.0.100"
EOF
```
#### Modify ingressClassName for hubble-ui ingress:
``` sh
kubectl patch ingress hubble-ui -n kube-system -p '{"spec":{"ingressClassName":"cilium"}}'
```
#### Change node ports for shared cilium ingress service:
``` sh
kubectl patch svc cilium-ingress -n kube-system --type='json' -p '[{"op":"replace","path":"/spec/ports/0/nodePort","value":30000},{"op":"replace","path":"/spec/ports/1/nodePort","value":32767}]'
```
#### Install demo prometheus with grafana to check available metrics and dashboards:
```sh 
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.1/examples/kubernetes/addons/prometheus/monitoring-example.yaml
```

#### Add label ``app.kubernetes.io/name`` for microservices in robot-shop 
```sh

for i in $(kubectl get deploy -n robot-shop --no-headers -o custom-columns='NAME:.metadata.name'); do kubectl patch deploy $i -n robot-shop --type='merge' -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"app.kubernetes.io/name\":\"$i\"}}}}}"
done
kubectl patch sts redis -n robot-shop --type='merge' -p '{"spec":{"template":{"metadata":{"labels":{"app.kubernetes.io/name":"redis"}}}}}'
```

#### Apply permissive CiliumNetworkPolicy with L7 rules in robot-shop namespace:
```sh
cat <<EOF | kubectl apply -n robot-shop -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-policy-l7
  namespace: robot-shop
spec:
  egress:
    - toEndpoints:
        - {}
    - toEntities:
        - cluster
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: '53'
              protocol: UDP
          rules:
            dns:
              - matchPattern: '*'
    - toEndpoints:
        - matchLabels:
          io.kubernetes.pod.namespace: robot-shop
      toPorts:  
        - ports:
            - port: '80'
              protocol: TCP
            - port: '8080'
              protocol: TCP
          rules:
            http: [{}]
    - toEndpoints:
        - matchLabels:
          io.kubernetes.pod.namespace: opentelemetry
      toPorts:
        - ports:
            - port: '4317'
              protocol: TCP
            - port: '4318'
              protocol: TCP
          rules:
            http: [{}]
    - toEntities:
        - world
  endpointSelector: {}
  ingress:
    - fromEntities:
        - cluster
    - fromEndpoints:
        - {}
    - fromEntities:
        - world
EOF
```

#### Adding scrape configs to existing VictoriaMetrics server
1) Create 2 VMServiceScrape custom resources in monitoring namespace:

envoy-metrics:
``` 
kubectl apply -n monitoring -f - <<EOF
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: envoy-metrics
spec:
  endpoints:
    - port: envoy-metrics
      relabelConfigs:
        - action: keep
          regex: 'true'
          source_labels:
            - __meta_kubernetes_service_annotation_prometheus_io_scrape
        - action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: "\${1}:\${2}"
          source_labels:
            - __address__
            - __meta_kubernetes_service_annotation_prometheus_io_port
          target_label: __address__
        - action: keep
          regex: cilium-envoy
          source_labels:
            - __meta_kubernetes_service_label_k8s_app
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - action: replace
          source_labels:
            - __meta_kubernetes_namespace
          target_label: namespace
        - action: replace
          source_labels:
            - __meta_kubernetes_service_name
          target_label: service
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app.kubernetes.io/name: cilium-envoy
EOF
```
hubble-metrics:
```
kubectl apply -n monitoring -f - <<EOF
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: hubble-metrics
spec:
  endpoints:
    - port: hubble-metrics
      relabelConfigs:
        - action: keep
          regex: 'true'
          source_labels:
            - __meta_kubernetes_service_annotation_prometheus_io_scrape
        - action: replace
          regex: (.+)(?::\d+);(\d+)
          replacement: "\${1}:\${2}"
          source_labels:
            - __address__
            - __meta_kubernetes_service_annotation_prometheus_io_port
          target_label: __address__
        - action: keep
          regex: hubble
          source_labels:
            - __meta_kubernetes_service_label_k8s_app
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - action: replace
          source_labels:
            - __meta_kubernetes_namespace
          target_label: namespace
        - action: replace
          source_labels:
            - __meta_kubernetes_service_name
          target_label: service
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app.kubernetes.io/name: hubble
EOF  
```

2) Create 2 VMPodScrape custom resources:

cilium-metrics:
```sh
kubectl apply -n monitoring -f - <<EOF
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
metadata:
  name: cilium-metrics
spec:
  namespaceSelector:
    matchNames:
      - kube-system
  podMetricsEndpoints:
    - port: prometheus
      relabelConfigs:
        - action: keep
          regex: 'true'
          source_labels:
            - __meta_kubernetes_pod_annotation_prometheus_io_scrape
        - action: replace
          regex: (.+):(?:\d+);(\d+)
          replacement: "\${1}:\${2}"
          source_labels:
            - __address__
            - __meta_kubernetes_pod_annotation_prometheus_io_port
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - action: replace
          source_labels:
            - __meta_kubernetes_namespace
          target_label: namespace
        - action: replace
          source_labels:
            - __meta_kubernetes_pod_name
          target_label: pod
        - action: keep
          regex: \d+
          source_labels:
            - __meta_kubernetes_pod_container_port_number
  selector:
    matchLabels:
      app.kubernetes.io/name: cilium-agent
EOF
```
cilium-operator-metrics:
```sh
kubectl apply -n monitoring -f - <<EOF
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
metadata:
  name: cilium-operator-metrics
spec:
  namespaceSelector:
    matchNames:
      - kube-system
  podMetricsEndpoints:
    - port: prometheus
      relabelConfigs:
        - action: keep
          regex: 'true'
          source_labels:
            - __meta_kubernetes_pod_annotation_prometheus_io_scrape
        - action: replace
          regex: (.+):(?:\d+);(\d+)
          replacement: "\${1}:\${2}"
          source_labels:
            - __address__
            - __meta_kubernetes_pod_annotation_prometheus_io_port
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - action: replace
          source_labels:
            - __meta_kubernetes_namespace
          target_label: namespace
        - action: replace
          source_labels:
            - __meta_kubernetes_pod_name
          target_label: pod
        - action: keep
          regex: \d+
          source_labels:
            - __meta_kubernetes_pod_container_port_number
  selector:
    matchLabels:
      app.kubernetes.io/name: cilium-operator
EOF
```
3) Import grafana dashboards from cilium demo monitoring and change datasource to Prometheus, uid to the actual one, which can extracted from URL grafana_URL/connections/datasources/edit/<uid> when datasource settings are opened.
