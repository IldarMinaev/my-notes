Prerequisites:
- WSL
- Helm
- Rancher-desktop
```
git clone  https://github.com/instana/robot-shop.git
cd robot-shop/K8s/helm/
kubectl create ns robot-shop
```
Edit values.yml - change redis pvc storageClass to local-path if it's default storage class in your k8s.
```
helm install robot-shop -n robot-shop .
```
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

[How to setup NGIX-Ingress-Controller on rancher-desktop](https://docs.rancherdesktop.io/how-to-guides/setup-NGINX-Ingress-Controller/)
