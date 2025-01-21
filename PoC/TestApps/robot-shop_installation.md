## Prerequisites
- WSL
- Helm
- Rancher-desktop
## Clone git repo
```
git clone  https://github.com/instana/robot-shop.git
cd robot-shop/
```
You can edit K8s/helm/values.yml or use `--set` parameter for `helm` command  or use `kustomize`.
```
helm upgrade --install --create-namespace -n robot-shop robot-shop K8s/helm/ \
  --set redis.storageClassName=local-path
```
## Create ingress
### With nginx ingress controller
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

### With default traefik in Rancher
On modern Linux distro names with suffixes `.localhost.localdomain` resolved to 127.0.0.1
So with default Rancher-Desktop traefik installation you can use following command to create ingress:
```shell
kubectl create ingress robot-shop-web -n robot-shop --class=traefik --rule="robot-shop.localhost.localdomain/*=web:8080"
```

## Run load
```shell
cd load-gen/; ./load-gen.sh -h http://robot-shop.localhost.localdomain -n 10 -t 1h30m
```