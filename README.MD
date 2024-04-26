# Example MinIO tenant setup with KES and Certificate Manager

Goal is to automate the entire of TLS certificates for MinIO, KES and the Prometheus with cert manager 100% Infrastructure
as code way so that no manual management or setup is needed to rotate certificates.


## Requirements
- kubectl

## Install Operators

Install MinIO Operator

```shell
kubectl apply -k minio-operator
```

Install Cert Manager Operator

```shell
kubectl apply -k cert-manager-operator
```

Install Prometheus Operator

```shell
kubectl create -f https://github.com/prometheus-operator/prometheus-operator/releases/download/v0.73.2/bundle.yaml
```

Install Cert Manager Issuers

```shell
kubectl apply -k cert-manager-issuers
```

Install CA certificate into MinIO Operator

```shell
kubectl get secret minio-tenant-1-ca-tls -n minio-tenant-1 -ojsonpath="{.data['ca\.crt']}" | base64 --decode > ca.crt
kubectl create secret generic operator-ca-tls-minio-tenant-1 --from-file=ca.crt -n minio-operator
kubectl rollout restart deployment.apps/minio-operator -n minio-operator
```

Install vault

```shell
kubectl apply -f tenant/vault.yaml
```

Get vault token and setup vault

```shell
# Get vault token
VAULT_ROOT_TOKEN=$(kubectl logs -l app=vault -n minio-tenant-1 | grep "Root Token: " | sed -e "s/Root Token: //g")

# Get vault pod name
VAULT_POD=$(kubectl get pods -l app=vault -n minio-tenant-1 | grep -v NAME | awk '{print $1}')

# unseal vault and enable approle
kubectl exec $VAULT_POD -n minio-tenant-1 -- sh -c 'VAULT_TOKEN='$VAULT_ROOT_TOKEN' VAULT_ADDR="http://127.0.0.1:8200" vault auth enable approle'
kubectl exec $VAULT_POD -n minio-tenant-1 -- sh -c 'VAULT_TOKEN='$VAULT_ROOT_TOKEN' VAULT_ADDR="http://127.0.0.1:8200" vault secrets enable kv'

# Copy policy file and write it
kubectl cp tenant/kes-policy.hcl $VAULT_POD:/kes-policy.hcl -n minio-tenant-1
kubectl exec $VAULT_POD -n minio-tenant-1 -- sh -c 'VAULT_TOKEN='$VAULT_ROOT_TOKEN' VAULT_ADDR="http://127.0.0.1:8200" vault policy write kes-policy /kes-policy.hcl'
kubectl exec $VAULT_POD -n minio-tenant-1 -- sh -c 'VAULT_TOKEN='$VAULT_ROOT_TOKEN' VAULT_ADDR="http://127.0.0.1:8200" vault write auth/approle/role/kes-role token_num_uses=0 secret_id_num_uses=0 period=5m policies=kes-policy'

# Get Role ID and secret
ROLE_ID=$(kubectl exec $VAULT_POD -n minio-tenant-1 -- sh -c 'VAULT_TOKEN='$VAULT_ROOT_TOKEN' VAULT_ADDR="http://127.0.0.1:8200" vault read auth/approle/role/kes-role/role-id' | grep "role_id    " | sed -e "s/role_id    //g")
SECRET_ID=$(kubectl exec $VAULT_POD -n minio-tenant-1 -- sh -c 'VAULT_TOKEN='$VAULT_ROOT_TOKEN' VAULT_ADDR="http://127.0.0.1:8200" vault write -f auth/approle/role/kes-role/secret-id' | grep "secret_id             " | sed -e "s/secret_id             //g")
```

Set the App role ID and Approle secret in `kes-configuration-secret.yaml`

Install Tenant and KES

```shell
kubectl apply -k tenant
```

Install prometheus and KES Scrapper
```shell
kubectl apply -k prometheus-kes
```

