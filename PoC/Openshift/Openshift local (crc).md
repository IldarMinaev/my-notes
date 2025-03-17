Read more: https://www.redhat.com/en/blog/codeready-containers
# Installation
First need to register on RedHat and Download crc binary and Pull Secret
https://cloud.redhat.com/openshift/install/crc/installer-provisioned
Binary can be downloaded in shell:
```shell
cd /tmp/
wget https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/crc/2.48.0/crc-linux-amd64.tar.xz
tar -xJvf crc-linux-amd64.tar.xz
mkdir -p ~/.local/bin/
cp crc-linux-2.48.0-amd64/crc ~/.local/bin/
```
Run installation
```shell
crc setup
```
Start instance after successful setup
```shell
crc start
```
During first start, `crc` ask for secret. Put content of downloaded `pull-secret.txt` file from RedHat web site.
After first start, `crc` show admin credentials of started OpenShift instance. Save it.
If you forget the credentials, you can get it by command
```shell
crc console --credentials
```
Prepare the environment
```shell
eval $(crc oc-env)
eval $(crc podman-env)
```
Now you can use OpenShift CLI tool. Login with command:
```shell
oc login -u kubeadmin -p <PASSWORD> https://api.crc.testing:6443
```
Or switch context of `kubectl` and use it to connect to OpenShift instance
```shell
kubectl config get-contexts
kubectl config use-context crc-admin
```
Or use Web Console https://console-openshift-console.apps-crc.testing/dashboards
# Useful commands
To stop OpenShift instance
```shell
crc stop
```
To delete the instance run
```shell
crc delete
```
To cleanup the installation run
```shell
crc cleanup
rm -rf ~/.crc/
```
By default `crc` create VM with 31GB disk size. To resize the VM use command
```shell
crc config set disk-size 40 && crc stop && crc start
```
By default crc create VM with 10GB RAM. To resize memory use command
```shell
crc config set memory 15360 && crc stop && crc start
```
# Use for devtest your images
To use docker from `crc VM`. Note, VM should have enough memory to run build.
```shell
eval $(crc oc-env)
eval $(crc podman-env)
docker buildx build --progress=plain -t ghcr.io/netcracker/qubership-env-checker:main -f Dockerfile ./
```
You also can use your local docker instance (not from `crc VM`).
Build your latest version of docker image
```shell
unset DOCKER_HOST CONTAINER_SSHKEY CONTAINER_HOST
docker context use default
docker buildx build --progress=plain -t ghcr.io/netcracker/qubership-env-checker:main -f Dockerfile ./
```
Save it to file
```shell
docker save ghcr.io/netcracker/qubership-env-checker:none -o /tmp/qubership-env-checker_none
```
In separate terminal session run Openshift node shell
```shell
oc node-shell crc
```
From first terminal copy saved file to node-shell
```shell
oc cp /tmp/qubership-env-checker_none $(oc get pod -n default | grep nsenter | awk '{print $1}'):/tmp/
```
In OpenShift node shell use podman to load image to Openshift registry
```shell
podman load -i /tmp/qubership-env-checker_none
podman images | grep qubership
podman images | head
```
Helm installation of env-checker
```shell
helm upgrade --install --create-namespace \
  --namespace=cse-toolset \
  qubership-env-checker charts/env-checker \
  --set NAMESPACE=cse-toolset \
  --set CLOUD_PUBLIC_HOST=localhost.localdomain
```