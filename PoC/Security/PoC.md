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
git clone git@sharedgithub.com:Netcracker/pgskipper-operator.git
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

For information: default policy:

```
# Allow tokens to look up their own properties
path "auth/token/lookup-self" {
    capabilities = ["read"]
}

# Allow tokens to renew themselves
path "auth/token/renew-self" {
    capabilities = ["update"]
}

# Allow tokens to revoke themselves
path "auth/token/revoke-self" {
    capabilities = ["update"]
}

# Allow a token to look up its own capabilities on a path
path "sys/capabilities-self" {
    capabilities = ["update"]
}

# Allow a token to look up its own entity by id or name
path "identity/entity/id/{{identity.entity.id}}" {
  capabilities = ["read"]
}
path "identity/entity/name/{{identity.entity.name}}" {
  capabilities = ["read"]
}


# Allow a token to look up its resultant ACL from all policies. This is useful
# for UIs. It is an internal path because the format may change at any time
# based on how the internal ACL features and capabilities change.
path "sys/internal/ui/resultant-acl" {
    capabilities = ["read"]
}

# Allow a token to renew a lease via lease_id in the request body; old path for
# old clients, new path for newer
path "sys/renew" {
    capabilities = ["update"]
}
path "sys/leases/renew" {
    capabilities = ["update"]
}

# Allow looking up lease properties. This requires knowing the lease ID ahead
# of time and does not divulge any sensitive information.
path "sys/leases/lookup" {
    capabilities = ["update"]
}

# Allow a token to manage its own cubbyhole
path "cubbyhole/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}

# Allow a token to wrap arbitrary values in a response-wrapping token
path "sys/wrapping/wrap" {
    capabilities = ["update"]
}

# Allow a token to look up the creation time and TTL of a given
# response-wrapping token
path "sys/wrapping/lookup" {
    capabilities = ["update"]
}

# Allow a token to unwrap a response-wrapping token. This is a convenience to
# avoid client token swapping since this is also part of the response wrapping
# policy.
path "sys/wrapping/unwrap" {
    capabilities = ["update"]
}

# Allow general purpose tools
path "sys/tools/hash" {
    capabilities = ["update"]
}
path "sys/tools/hash/*" {
    capabilities = ["update"]
}

# Allow checking the status of a Control Group request if the user has the
# accessor
path "sys/control-group/request" {
    capabilities = ["update"]
}

# Allow a token to make requests to the Authorization Endpoint for OIDC providers.
path "identity/oidc/provider/+/authorize" {
    capabilities = ["read", "update"]
}
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
            name: "vault-auth"          # serviceAccount name
EOF
```

Check SecretStore have Ready status

```shell
kubectl get SecretStore -A
```

### Create Vault key/value

```shell
PGUSER=$(kubectl get secrets -n postgres "postgres-credentials" -o go-template='{{.data.user | base64decode}}')
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
    - secretKey: custom-key1
      remoteRef:
        key: postgres/secret1
        property: key1
    - secretKey: custom-key2
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
```

```
key1: value1
key2: value2
postgres.secret2.key3: value3
postgres.secret2.key4: value4
```

### Extract all keys from SecretStore

Create ExternalSecret

```shell
cat <<EOF | kubectl apply -n postgres -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: test-secret-extract
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: secret-store
    kind: SecretStore
  target:
    name: test-secret-extract
  dataFrom:
    - extract:
        key: postgres/secret1
    - extract:
        key: postgres/secret2
EOF
```

Check result:

```shell
kubectl get secret -n postgres test-secret-extract -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```

Result:

```text
key1: value1
key2: value2
key3: value3
key4: value4
```

### Check backoff policy

Backoff policy - What will happen if External Secret CR not able to find если не обнаружит путь (node path)

***Answer***:
a. ExternalSecret CR have `SecretSyncedError` status
b. Reconcile error in logs: `error processing spec.data[0] (key: postgres.postgres-credentials1), err: Secret does not exist`
c. Requests to SecretsStore is [exponential](https://github.com/external-secrets/external-secrets/blob/002d1b0d2bfe0a874671ecad2b4eaf7960f38802/pkg/controllers/common/common.go#L97-L103):

```shell
# change ExternalSecret with wrong key (postgres.postgres-credentials1)
prev="";
kubectl logs -n vault external-secrets-665666979-bqjs5 | \
  grep postgres.postgres-credentials1 | \
  sed -nE 's/.*"ts":([0-9]+).*/\1/p' | \
  while read -r num; do
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

Same for wrong role in Secret Manager:

```shell
prev="";
kubectl logs -n vault external-secrets-665666979-bqjs5 | \
  grep 'cannot read secret data from Vault' | \
  sed -nE 's/.*"ts":([0-9]+).*/\1/p' | \
  while read -r num; do
    if [[ -n "$prev" ]]; then
      diff=$((num - prev));
      echo "$diff";
    fi;
    prev=$num;
  done
```

Result:

```text
0
1
5
8
17
32
```

Same for wrong auth parameters in SecretStore:

```shell
prev="";
kubectl logs -n vault external-secrets-665666979-bqjs5 | \
  grep 'unable to log in with Kubernetes auth' | \
  sed -nE 's/.*"ts":([0-9]+).*/\1/p' | \
  while read -r num; do
    if [[ -n "$prev" ]]; then
      diff=$((num - prev));
      echo "$diff";
    fi;
    prev=$num;
  done | grep -v '^0$'
```

Result:

```text
1
4
8
16
32
64
128
256
420
420
420
421
420
420
420
```

### Push secrets

Push secrets still in Alpha: https://external-secrets.io/latest/guides/pushsecrets/

Test in separate NS
```shell
kubectl create namespace test-apps
```
Create RO Stores

```shell
kubectl exec -ti -n vault vault-0 -- vault policy write app1-read - << EOF
path "shared/data/app1/*" {
   capabilities = ["list", "read"]
}
path "shared/metadata/app1/*" {
   capabilities = ["list", "read"]
}
EOF

kubectl exec -ti -n vault vault-0 -- vault policy write app2-read - << EOF
path "shared/data/app2/*" {
   capabilities = ["list", "read"]
}
path "shared/metadata/app2/*" {
   capabilities = ["list", "read"]
}
EOF
```

Create RW Stores

```shell
kubectl exec -ti -n vault vault-0 -- vault policy write app1-write - << EOF
path "shared/data/app1/*" {
   capabilities = ["create", "update", "delete"]
}
path "shared/metadata/app1/*" {
   capabilities = ["create", "update", "delete"]
}
EOF

kubectl exec -ti -n vault vault-0 -- vault policy write app2-write - << EOF
path "shared/data/app2/*" {
   capabilities = ["create", "update", "delete"]
}
path "shared/metadata/app2/*" {
   capabilities = ["create", "update", "delete"]
}
EOF
```

Create roles

```shell
kubectl exec -ti -n vault vault-0 -- vault write auth/kubernetes/role/app1-read-secret \
  bound_service_account_names=vault-auth-app1-read \
  bound_service_account_namespaces=test-apps \
  policies=default,app1-read \
  alias_name_source=serviceaccount_name \
  ttl=1h

kubectl exec -ti -n vault vault-0 -- vault write auth/kubernetes/role/app2-read-secret \
  bound_service_account_names=vault-auth-app2-read \
  bound_service_account_namespaces=test-apps \
  policies=default,app2-read \
  alias_name_source=serviceaccount_name \
  ttl=1h

kubectl exec -ti -n vault vault-0 -- vault write auth/kubernetes/role/app1-write-secret \
  bound_service_account_names=vault-auth-app1-write \
  bound_service_account_namespaces=test-apps \
  policies=default,app1-read,app1-write \
  alias_name_source=serviceaccount_name \
  ttl=1h

kubectl exec -ti -n vault vault-0 -- vault write auth/kubernetes/role/app2-write-secret \
  bound_service_account_names=vault-auth-app2-write \
  bound_service_account_namespaces=test-apps \
  policies=default,app2-read,app2-write \
  alias_name_source=serviceaccount_name \
  ttl=1h
```

Create ServiceAccounts

```shell
kubectl create serviceaccount vault-auth-app1-read -n test-apps
kubectl create serviceaccount vault-auth-app2-read -n test-apps
kubectl create serviceaccount vault-auth-app1-write -n test-apps
kubectl create serviceaccount vault-auth-app2-write -n test-apps

kubectl create clusterrolebinding vault-auth-test-apps \
  --clusterrole=system:auth-delegator \
  --serviceaccount=test-apps:vault-auth-app1-read \
  --serviceaccount=test-apps:vault-auth-app2-read \
  --serviceaccount=test-apps:vault-auth-app1-write \
  --serviceaccount=test-apps:vault-auth-app2-write
```

Cluster role `system:auth-delegator` required because Vault validates the Kubernetes service account token using the Kubernetes TokenReview API. This API requires the service account to have permissions to request a token review, which is granted by binding the `system:auth-delegator` ClusterRole to the service account. Without this ClusterRole binding, the service account is not authorized to verify tokens via the TokenReview API, which causes the SecretStore authentication to fail.
See: https://external-secrets.io/latest/provider/hashicorp-vault/

Create SecretStores

```shell
cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: store-app1-read
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "shared"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "app1-read-secret"
          serviceAccountRef:
            name: "vault-auth-app1-read"
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: store-app2-read
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "shared"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "app2-read-secret"
          serviceAccountRef:
            name: "vault-auth-app2-read"
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: store-app1-write
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "shared"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "app1-write-secret"
          serviceAccountRef:
            name: "vault-auth-app1-write"
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: store-app2-write
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "shared"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "app2-write-secret"
          serviceAccountRef:
            name: "vault-auth-app2-write"
EOF
```

Create ExternalSecrets for read

```shell
cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app1-secret-from-store-app1-read
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: store-app1-read
    kind: SecretStore
  target:
    name: app1-secret-from-store-app1-read
  dataFrom:
    - extract:
        key: app1/secret
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app2-secret-from-store-app2-read
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: store-app2-read
    kind: SecretStore
  target:
    name: app2-secret-from-store-app2-read
  dataFrom:
    - extract:
        key: app2/secret
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app1-secret-from-store-app2-read
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: store-app2-read
    kind: SecretStore
  target:
    name: app1-secret-from-store-app2-read
  dataFrom:
    - extract:
        key: app1/secret
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app2-secret-from-store-app1-read
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: store-app1-read
    kind: SecretStore
  target:
    name: app2-secret-from-store-app1-read
  dataFrom:
    - extract:
        key: app2/secret
EOF
```

Create source secrets

```shell
cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: v1
kind: Secret
metadata:
  name: source-secret-app1
stringData:
  key1: "value1"
EOF
cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: v1
kind: Secret
metadata:
  name: source-secret-app2
stringData:
  key2: "value2"
EOF
```

Create pushsecret

```shell
cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: pushsecret-app1
spec:
  updatePolicy: Replace # Policy to overwrite existing secrets in the provider on sync
  deletionPolicy: Delete # the provider' secret will be deleted if the PushSecret is deleted
  refreshInterval: 1h # Refresh interval for which push secret will reconcile
  secretStoreRefs: # A list of secret stores to push secrets to
    - name: store-app1-write
      kind: SecretStore
  selector:
    secret:
      name: source-secret-app1 # Source Kubernetes secret to be pushed. Alternatively, you can point to a generator that produces values to be pushed
  data:
    - conversionStrategy: None # Also supports the ReverseUnicode strategy
      match:
        # The secretKey is used within PushSecret (it should match key under spec.template.data)
        secretKey: key1
        remoteRef:
          remoteKey: app1/secret # The destination secret object name (where the secret is going to be pushed)
          property: key1 # The key within the destination secret object.
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: pushsecret-app2
spec:
  updatePolicy: Replace # Policy to overwrite existing secrets in the provider on sync
  deletionPolicy: Delete # the provider' secret will be deleted if the PushSecret is deleted
  refreshInterval: 1h # Refresh interval for which push secret will reconcile
  secretStoreRefs: # A list of secret stores to push secrets to
    - name: store-app2-write
      kind: SecretStore
  selector:
    secret:
      name: source-secret-app2 # Source Kubernetes secret to be pushed. Alternatively, you can point to a generator that produces values to be pushed
  data:
    - conversionStrategy: None # Also supports the ReverseUnicode strategy
      match:
        # The secretKey is used within PushSecret (it should match key under spec.template.data)
        secretKey: key2
        remoteRef:
          remoteKey: app2/secret # The destination secret object name (where the secret is going to be pushed)
          property: key2 # The key within the destination secret object.
EOF
```

Sync time: based on interval in ExternalSecret and if sync was - up to 420 seconds. For me it was 43 seconds for first secret and 198 for second one.

Test resulted objects:

```shell
kubectl get secretstore -n test-apps; kubectl get ExternalSecret -n test-apps; kubectl get pushsecret -n test-apps; kubectl get secret -n test-apps
```

Result:

```text
NAME               AGE   STATUS   CAPABILITIES   READY
store-app1-read    46h   Valid    ReadWrite      True
store-app1-write   21h   Valid    ReadWrite      True
store-app2-read    21h   Valid    ReadWrite      True
store-app2-write   21h   Valid    ReadWrite      True
NAME                               STORETYPE     STORE             REFRESH INTERVAL   STATUS              READY
app1-secret-from-store-app1-read   SecretStore   store-app1-read   15s                SecretSynced        True
app1-secret-from-store-app2-read   SecretStore   store-app2-read   15s                SecretSyncedError   False
app2-secret-from-store-app1-read   SecretStore   store-app1-read   15s                SecretSyncedError   False
app2-secret-from-store-app2-read   SecretStore   store-app2-read   15s                SecretSynced        True
NAME              AGE     STATUS
pushsecret-app1   11m     Synced
pushsecret-app2   7m15s   Synced
NAME                               TYPE     DATA   AGE
app1-secret-from-store-app1-read   Opaque   1      9m39s
app2-secret-from-store-app2-read   Opaque   1      2m36s
source-secret-app1                 Opaque   1      33m
source-secret-app2                 Opaque   1      33m
```

Sync errors logs:

```shell
kubectl logs -n vault external-secrets-665666979-bqjs5 | grep app1-secret-from-store-app2-read | tail -n 1
```

Result:

```text
{"level":"error","ts":1757070426.4602442,"msg":"Reconciler error","controller":"externalsecret","controllerGroup":"external-secrets.io","controllerKind":"ExternalSecret","ExternalSecret":{"name":"app1-secret-from-store-app2-read","namespace":"test-apps"},"namespace":"test-apps","name":"app1-secret-from-store-app2-read","reconcileID":"08ac487f-7d0b-4475-8b80-153d872917b0","error":"error processing spec.dataFrom[0].extract, err: cannot read secret data from Vault: Error making API request.\n\nURL: GET http://vault.vault:8200/v1/shared/data/app1/secret\nCode: 403. Errors:\n\n* 1 error occurred:\n\t* permission denied\n\n","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).reconcileHandler\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.21.0/pkg/internal/controller/controller.go:353\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).processNextWorkItem\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.21.0/pkg/internal/controller/controller.go:300\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).Start.func2.1\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.21.0/pkg/internal/controller/controller.go:202"}

```

Check value:

```shell
kubectl get secrets -n test-apps app2-secret-from-store-app2-read -o go-template='{{.data.key2 | base64decode}}'; echo
```

Check create only policy

```shell
cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: v1
kind: Secret
metadata:
  name: source-secret-app2-overwrite-test
stringData:
  key2: "overrided"
EOF

cat <<EOF | kubectl apply -n test-apps -f -
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: pushsecret-app2-override
spec:
  updatePolicy: IfNotExists
  deletionPolicy: None
  refreshInterval: 15s
  secretStoreRefs:
    - name: store-app2-write
      kind: SecretStore
  selector:
    secret:
      name: source-secret-app2-overwrite-test
  data:
    - match:
        secretKey: key2
        remoteRef:
          remoteKey: app2/secret
          property: key2
EOF
```

Check synced:

```shell
kubectl get pushsecret -n test-apps pushsecret-app2-override
```

Result:

```
NAME                       AGE    STATUS
pushsecret-app2-override   100s   Synced
```

Check value:

```shell
kubectl get secrets -n test-apps app2-secret-from-store-app2-read -o go-template='{{.data.key2 | base64decode}}'; echo
```

Result:
```text
value2
```

Check removing secret in secretstore:

```shell
kubectl exec -ti -n vault vault-0 -- vault kv delete \
  -mount shared \
  app2/secret
```

Check within 15 seconds it overrided:

```
kubectl get secrets -n test-apps app2-secret-from-store-app2-read -o go-template='{{.data.key2 | base64decode}}'; echo
```

Result:

```
overrided
```

Wait on hour and check it returned back because of update policy of `pushsecret-app2`

```shell
sleep 3600; kubectl get secrets -n test-apps app2-secret-from-store-app2-read -o go-template='{{.data.key2 | base64decode}}'; echo
```

Result:

```
value2
```

## Proof of concepts

### Design

TODO: Add design

### POC Repo

Currently POC repo located in private repository: <https://github.com/IldarMinaev/poc>. Contact Ildar Minaev to get access.