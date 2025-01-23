# Installation
```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts --force-update
helm install --create-namespace -n otel-demo my-otel-demo open-telemetry/opentelemetry-demo
```
# Demo
Run proxy
```shell
kubectl --namespace otel-demo port-forward svc/my-otel-demo-frontendproxy 8080:8080
```

| Demo resource     | Link                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| Webstore          | [http://localhost:8080/](http://localhost:8080/)                     |
| Jaeger UI         | [http://localhost:8080/jaeger/ui/](http://localhost:8080/jaeger/ui/) |
| Grafana           | [http://localhost:8080/grafana/](http://localhost:8080/grafana/)     |
| Load Generator UI | [http://localhost:8080/loadgen/](http://localhost:8080/loadgen/)     |
| Feature Flags UI  | [http://localhost:8080/feature/](http://localhost:8080/feature/)     |
