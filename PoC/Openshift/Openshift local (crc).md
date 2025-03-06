First need to register on RedHat and Download crc binary and Pull Secret
https://cloud.redhat.com/openshift/install/crc/installer-provisioned
Binary can be downloaded in shell:
```shell
cd /tmp/
wget https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/crc/2.48.0/crc-linux-amd64.tar.xz
tar -xJvf crc-linux-amd64.tar.xz
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
```