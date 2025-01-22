# Prerequisites
- WSL
- Helm
- Rancher-desktop
# Clone git repo
```
git clone  https://github.com/instana/robot-shop.git
cd robot-shop/
```
# Install the application
You can edit K8s/helm/values.yml or use `--set` parameter for `helm` command  or use `kustomize`.
```
helm upgrade --install --create-namespace -n robot-shop robot-shop K8s/helm/ \
  --set redis.storageClassName=local-path
```
# Create ingress
## With nginx ingress controller
[How to setup NGIX-Ingress-Controller on rancher-desktop](https://docs.rancherdesktop.io/how-to-guides/setup-NGINX-Ingress-Controller/)
If you already installed nginx-ingress-cotroller in rancher, you can try to create ingress for web ui:
```
kubectl create ingress robot-shop-web -n robot-shop --class=nginx --rule="robot-shop.<host_address>/*=web:8080"
```
where `host_address` is any preferrable DNS address for your rancher k8s instance.
example:
```
kubectl create ingress robot-shop-web -n robot-shop --class=nginx --rule="robot-shop.demo.rancher-desktop.qubership.org/*=web:8080"
```
Add a record for chosen host address in hosts file to redirect requests to 127.0.0.1 or install and configure local DNS to redirect all *.<host_address> requests to localhost.

## With default traefik in Rancher
On modern Linux distro names with suffixes `.localhost.localdomain` resolved to 127.0.0.1
So with default Rancher-Desktop traefik installation you can use following command to create ingress:
```shell
kubectl create ingress robot-shop-web -n robot-shop --class=traefik --rule="robot-shop.localhost.localdomain/*=web:8080"
```
# Make observable
Make sure observability backends installed and open-telemetry collector created. See [[Observability Cluster Backends#Create an OpenTelemetry Collector]]
## Create instrumentation
```shell
kubectl apply -n robot-shop -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector-collector:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  # Looks like need to cafefully read documentation/agents code. As defaults for protocol (http/protobuf or grpc) is different.
  #python:
  #  env:
  #    - name: OTEL_EXPORTER_OTLP_ENDPOINT
  #      value: http://otel-collector-collector:4318
  #java:
  #  env:
  #    - name: OTEL_EXPORTER_OTLP_ENDPOINT
  #      value: http://otel-collector-collector:4318
  #go:
  #  env:
  #    - name: OTEL_EXPORTER_OTLP_ENDPOINT
  #      value: http://otel-collector-collector:4318
EOF
```
## Patch deployments with annotations
#### For Java applications:
```shell
kubectl patch deploy shipping -n robot-shop -p '{"spec": {"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"true"}}}} }'
```
#### For Go applications:
Enable Go language support in opentelemetry-operator by adding the flag --enable-go-instrumentation=true in operator's manager arguments.
Annotate deployment with inject-go: true and with path to target executable binary:
```shell
kubectl patch deploy dispatch -n robot-shop -p '{"spec": {"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-go":"true","instrumentation.opentelemetry.io/otel-go-auto-target-exe":"/go/bin/dispatch"}}}}}'
```
In Instrumentation CR need to change env variable OTEL_EXPORTER_OTLP_ENDPOINT to send telemetry data to 4318 port on opentelemetry collector and change go-instrumentation image version to v0.12.0-alpha (replace collector endpoint with actual address on your environment):
```shell
kubectl patch Instrumentation otel-instrumentation -n robot-shop --type=merge -p '{"spec":{"go":{"env":[{"name":"OTEL_EXPORTER_OTLP_ENDPOINT","value":"http://qubership-opentelemetry-collector.open-telemetry:4318"}],"image":"ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:v0.12.0-alpha"}}}'
```
#### For Python applications:
```shell
kubectl patch deploy payment -n robot-shop -p '{"spec": {"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":"true"}}}} }'
```
Collector endpoint also should be changed to 4318 port in CR instrumentation and default python instrumentation image to 0.48b0 (replace collector endpoint with actual address on your environment):
```shell
kubectl patch Instrumentation otel-instrumentation -n robot-shop --type=merge -p '{"spec":{"python":{"env":[{"name":"OTEL_EXPORTER_OTLP_ENDPOINT","value":"http://qubership-opentelemetry-collector.open-telemetry:4318"}],"image":"ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.48b0"}}}'
```
TODO add annotations to all other services

> **NOTE**
> You can create multiple instrumentation custom resources with different settings and images and bind to particular microservices with annotation:
> ```shell
> instrumentation.opentelemetry.io/inject-<language>: <your_instrumentation_cr_name>
> ```
> example:
> ```shell
> kubectl patch deploy shipping -n robot-shop -p '{"spec": {"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"my-instrumentation"}}}} }'
> ```

# Run load
```shell
./K8s/autoscale.sh
cd load-gen/; ./load-gen.sh -h http://robot-shop.localhost.localdomain -n 10 -t 1h30m; cd -
```
