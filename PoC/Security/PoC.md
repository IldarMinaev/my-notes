# External Secrets Operator with Vault and PG Skipper Operator

## Install Hashicorp Vault

### 1. Get repo with helm

```shell
git clone git@github.com:hashicorp/vault-helm.git
cd vault-helm/
git checkout $(git describe --tags --abbrev=0)
```

### 2. Install with helm one node Vault cluster

```shell
helm upgrade vault . \
  --install \
  -n vault \
  --create-namespace \
  --set='server.ha.enabled=false'           `# Disable HighAvailability mode` \
  --set='server.ha.raft.enabled=true'       `# Integrated storage (Raft)` \
  --set='server.serviceAccount.create=true' `# Create the Vault service account` \
  --set='server.serviceAccount.name=vault'  `# Service account name Vault uses` \
  --set='csi.enabled=true'                  `# Enable Vault CSI Provider` \
  --set='injector.enabled=true'             `# Enable Vault Agent Injector (Kubernetes mutation webhook)` \
  --wait
```

### 3. Configure Vault

After first installation when pod running (but still not healthy), initialize Vault storage:

```shell
kubectl exec -ti -n vault vault-0 -- vault operator init
```

Do not forget to save securely unseal keys.
Unseal the cluster (use 3 keys, even if no HA mode enabled, run commands in first pod) (see details <https://developer.hashicorp.com/vault/docs/concepts/seal>):

```shell
kubectl exec -ti -n vault vault-0 -- vault operator unseal
```

If HA mode used, run unseal on second and third pods:

```shell
kubectl exec -ti -n vault vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -ti -n vault vault-1 -- vault operator unseal
```

Authorize in Vault cluster using Root token

```shell
kubectl exec -ti -n vault vault-0 -- vault login
```

Check cluster status

```shell
kubectl exec -ti -n vault vault-0 -- vault operator raft list-peers
```

## Install External Secrets Operator

Using helm repo:

```shell
helm repo add external-secrets https://charts.external-secrets.io

helm upgrade external-secrets external-secrets/external-secrets \
  --install \
  -n vault \
  --create-namespace \
  --wait
```

## Install PG Skipper

### 1. Get repo

```shell
git clone git@github.com:Netcracker/pgskipper-operator.git
cd pgskipper-operator
```

### 2. Install manually CRDs

```shell
kubectl apply --server-side --force-conflicts -f ./charts/patroni-core/crds/qubership.org_patronicores.yaml
kubectl apply --server-side --force-conflicts -f ./charts/patroni-services/crds/qubership.org_patroniservices.yaml
```

### 3. Install patroni core

```shell
helm upgrade patroni-core ./charts/patroni-core \
  --install \
  -n postgres \
  --create-namespace \
  -f ./charts/patroni-core/patroni-core-quickstart-sample.yaml \
  --set='storage.type=provisioned'        `# Configure used storage provisioner` \
  --set='storage.size=2Gi'                `# Size of PV requested` \
  --set='storage.storageClass=local-path' `# Default in Rancher Desktop` \
  --wait
```

### 4. Install patroni services

```shell
helm upgrade patroni-services ./charts/patroni-services \
  --install \
  -n postgres \
  --create-namespace \
  -f ./charts/patroni-services/patroni-services-quickstart-sample.yaml \
  --set='storage.type=provisioned'        `# Configure used storage provisioner` \
  --set='storage.size=2Gi'                `# Size of PV requested` \
  --set='storage.storageClass=local-path' `# Default in Rancher Desktop` \
  --wait
```

## Configure use passphrase from SecretManager

### Configure SecretStore

Enable kv secret plugin

```shell
kubectl exec -ti -n vault vault-0 -- vault secrets enable -path shared -version 2 kv
```

Create KV read policy

```shell
kubectl exec -ti -n vault vault-0 -- vault policy write kv-read - << EOF
path "shared/data/*" {
   capabilities = ["list", "read", "create", "update", "delete"]
}
path "shared/metadata/*" {
   capabilities = ["list", "read", "create", "update", "delete"]
}
path "auth/token/renew-self" {
    capabilities = ["update"]
}
EOF
```

Configure Kubernetes authorization (see <https://developer.hashicorp.com/vault/docs/auth/kubernetes>)

```shell
kubectl exec -ti -n vault vault-0 -- vault auth enable kubernetes
KUBERNETES_PORT_443_TCP_ADDR=$(kubectl exec -ti -n vault vault-0 -- sh -c 'echo -n ${KUBERNETES_PORT_443_TCP_ADDR}')
kubectl exec -ti -n vault vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
  disable_iss_validation=true `# Required for Kubernetes v1.21+` \
  issuer="https://kubernetes.default.svc.cluster.local"
```

Create Vault Role (see <https://developer.hashicorp.com/vault/api-docs/auth/kubernetes#create-update-role>)

```shell
kubectl exec -ti -n vault vault-0 -- vault write auth/kubernetes/role/read-secret \
bound_service_account_names=vault-auth \
bound_service_account_namespaces=postgres \
policies=kv-read \
alias_name_source=serviceaccount_name \
ttl=1h
```

Create service account in app namespace to access Vault

```shell
kubectl create serviceaccount vault-auth -n postgres
```

Create Cluster Role Binding for App Vault's service account (see <https://support.hashicorp.com/hc/en-us/articles/4404389946387-Kubernetes-auth-method-Permission-Denied-error>)

```shell
kubectl create clusterrolebinding vault-auth-delegator-postgres \
  --clusterrole=system:auth-delegator \
  --serviceaccount=postgres:vault-auth
```

Create SecretStore (see <https://external-secrets.io/main/api/secretstore/> and <https://external-secrets.io/main/api/spec/#external-secrets.io/v1.VaultProvider>)

```shell
cat <<EOF | kubectl apply -n postgres -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: secret-store
spec:
  provider:
    vault:
      server: "http://vault.vault:8200" # Installed Vault service URL
      path: "shared"                    # Name of kv
      version: "v2"                     # Version of kv
      auth:
        kubernetes:                     # Authorization method
          mountPath: "kubernetes"
          role: "read-secret"           # Vault Role for authorization
          serviceAccountRef:
            name: "vault-auth"               # serviceAccount name
EOF
```

Check SecretStore have Ready status

```shell
kubectl get SecretStore -A
```

### Create Vault key/value

```shell
PGUSER=$(kubectl get secrets -n postgres "postgres-credentials" -o go-template='{{.data.username | base64decode}}')
PGPASSWORD=$(kubectl get secrets -n postgres "postgres-credentials" -o go-template='{{.data.password | base64decode}}')

kubectl exec -ti -n vault vault-0 -- vault kv put \
  -mount shared \
  postgres.postgres-credentials \
  username=${PGUSER} password=${PGPASSWORD}
```

Check get

```shell
kubectl exec -ti -n vault vault-0 -- vault kv get \
  -mount shared \
  postgres.postgres-credentials
```

### Create ExternalSecret

Create External Secret (see <https://external-secrets.io/main/api/externalsecret/>)

```shell
cat <<EOF | kubectl apply -n postgres -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: postgres-credentials
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: secret-store                 # Name of used SecretStore
    kind: SecretStore
  target:
    name: postgres-credentials        # Kubernetes Secret Name
  data:
    - secretKey: user                  # Key in K8s Secret
      remoteRef:
        key: postgres.postgres-credentials   # Path to secret in Vault
        property: username             # Key of secret in Vault
    - secretKey: password
      remoteRef:
        key: postgres.postgres-credentials
        property: password
EOF
```

Check sync status of ExternalSecret:

```shell
kubectl get ExternalSecret -A
```

## Rotate the password

Create port forwarding for Vault UI

```shell
kubectl port-forward vault-0 8200:8200 -n vault
```

In Browser:
1. Open URL <http://localhost:8200/ui/vault/secrets/shared/kv/postgres.postgres-credentials/details?version=1>
2. Click `Create new version` button
3. Change value of `password`
4. Click `Save`
Check Secret updated:Open URL <http://localhost:8200/ui/vault/secrets/shared/kv/postgres.postgres-credentials/details?version=1>
5. Click `Create new version` button
6. Change value of `password`
7. Click `Save`

```shell
kubectl get secrets -n postgres "postgres-credentials" -o go-template='{{.data.password | base64decode}}'
```

Check Postgres logs

```shell
kubectl logs -n postgres $(kubectl get pod -n postgres -o name -l app=patroni,pgtype=master)
```

Currently it shows error:

```text
[2025-09-02 17:26:40.998 UTC][source=postgresql]ERROR:  syntax error at or near "'p@ssWOrD2'" at character 22
[2025-09-02 17:26:40.998 UTC][source=postgresql]STATEMENT:  ALTER ROLE  PASSWORD 'p@ssWOrD2'
```

## Experiments

### Create different keys

```shell
kubectl exec -ti -n vault vault-0 -- vault kv put \
  -mount shared \
  postgres/secret1 \
  key1=value1 key2=value2

kubectl exec -ti -n vault vault-0 -- vault kv put \
  -mount shared \
  postgres/secret2 \
  key3=value3 key4=value4
```


```shell
cat <<EOF | kubectl apply -n postgres -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: test-secret
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: secret-store
    kind: SecretStore
  target:
    name: test-secret
  data:
    - secretKey: key1
      remoteRef:
        key: postgres/secret1
        property: key1
    - secretKey: key2
      remoteRef:
        key: postgres/secret1
        property: key2
    - secretKey: postgres.secret2.key3
      remoteRef:
        key: postgres/secret2
        property: key3
    - secretKey: postgres.secret2.key4
      remoteRef:
        key: postgres/secret2
        property: key4
EOF
```

Check in secret:

```shell
kubectl get secret -n postgres test-secret -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'

key1: value1
key2: value2
postgres.secret2.key3: value3
postgres.secret2.key4: value4
```

## Check backoff policy

Backoff policy - What will happen if External Secret CR not able to find если не обнаружит путь (node path)

***Answer***:
a. ExternalSecret CR have `SecretSyncedError` status
b. Reconcile error in logs: `error processing spec.data[0] (key: postgres.postgres-credentials1), err: Secret does not exist`
c. Requests to SecretsStore is [exponential](https://github.com/external-secrets/external-secrets/blob/002d1b0d2bfe0a874671ecad2b4eaf7960f38802/pkg/controllers/common/common.go#L97-L103):
```shell
# change ExternalSecret with wrong key (postgres.postgres-credentials1)
prev=""; k logs -n vault external-secrets-665666979-bqjs5 | grep postgres.postgres-credentials1 | sed -nE 's/.*"ts":([0-9]+).*/\1/p' | while read -r num; do
  if [[ -n "$prev" ]]; then
    diff=$((num - prev))
    echo "$diff"
  fi            
  prev=$num
done
```
Result:
```text
1
1
4
8
16
33
64
128
257
```