# Build
Locally build all images (make sure docker used context is same as used by your rancher instance)
```shell
git clone git@github.com:Netcracker/qubership-integration-designtime-catalog.git
cd qubership-integration-designtime-catalog
mvn install -Dgpg.skip
docker build --progress=plain -t qubership-integration-platform-qip-design-time-catalog:latest -f ./Dockerfile ./
cd -
git clone git@github.com:Netcracker/qubership-integration-engine.git
cd qubership-integration-engine
docker build --progress=plain -t qubership-integration-platform-qip-engine:latest -f ./Dockerfile ./
cd -
git clone git@github.com:Netcracker/qubership-integration-runtime-catalog.git
cd qubership-integration-runtime-catalog
mvn install -Dgpg.skip -Dmaven.test.skip=true
docker build --progress=plain -t qubership-integration-platform-qip-runtime-catalog:latest -f ./Dockerfile ./
cd -
git clone git@github.com:Netcracker/qubership-integration-sessions-management.git
cd qubership-integration-sessions-management
mvn install -Dgpg.skip
docker build --progress=plain -t qubership-integration-platform-qip-sessions-management:latest -f ./Dockerfile ./
cd -
git clone git@github.com:Netcracker/qubership-integration-variables-management.git
cd qubership-integration-variables-management
mvn install -Dgpg.skip
docker build --progress=plain -t qubership-integration-platform-qip-variables-management:latest -f ./Dockerfile ./
```
# Install
As QIP have not merged helm charts. Download and swithc to helm branch
```shell
git clone git@github.com:Netcracker/qubership-integration-platform.git
cd qubership-integration-platform
git checkout helm-charts
```
Install helm
```shell
helm upgrade --install --create-namespace -n qip qip-k8s-dev qip-dev/
```
# Configure
TODO