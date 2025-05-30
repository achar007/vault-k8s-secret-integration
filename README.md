---

## 🔐 Vault Integration with Kubernetes using Agent Injector

###  Prerequisites

* Vault deployed and initialized
* Kubernetes cluster running
* Vault Helm chart (optional)
* `vault` CLI configured to access Vault
* A working namespace (e.g., `my-namespace`)

---

### 1️⃣ Enable Kubernetes Auth in Vault

```sh
vault auth enable kubernetes
```

Check it:

```sh
vault auth list
```

---

### 2️⃣ Configure Kubernetes Auth

Run inside the Vault pod:

```sh
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

---

### 3️⃣ Create Vault Role for App

```sh
vault write auth/kubernetes/role/my-app \
  bound_service_account_names=default \
  bound_service_account_namespaces=my-namespace \
  policies=my-policy \
  ttl=24h
```

---

### 4️⃣ Create Vault Policy

Create a file `my-policy.hcl`:

```hcl
path "kvv2/data/myapp" {
  capabilities = ["read"]
}

path "kvv2/data/myapp/*" {
  capabilities = ["read"]
}
```

Upload it:

```sh
vault policy write my-policy my-policy.hcl
```

---

### 5️⃣ Create Secret in Vault (KV v2)

```sh
vault kv put kvv2/myapp username='appuser' password='s3cr3t'
```

Check it:

```sh
vault kv get kvv2/myapp
```

---

### 6️⃣ Deploy App with Vault Injector

**vault-injector-deployment.yaml**:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-demo-injector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault-demo-injector
  template:
    metadata:
      labels:
        app: vault-demo-injector
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: my-app
        vault.hashicorp.com/agent-inject-secret-config.txt: kvv2/data/myapp
        vault.hashicorp.com/agent-inject-template-config.txt: |
          {{- with secret "kvv2/data/myapp" -}}
          username: {{ .Data.data.username }}
          password: {{ .Data.data.password }}
          {{- end }}
    spec:
      serviceAccountName: default
      containers:
        - name: app
          image: busybox
          command:
            - sleep
            - "3600"

```

Apply it:

```sh
kubectl create namespace my-namespace
kubectl -n my-namespace apply -f vault-injector-deployment.yaml
```

---

### 7️⃣ Verify Secret Injection

```sh
kubectl -n my-namespace exec -it <pod-name> -- cat /vault/secrets/config.txt
```

Expected output:

```
username: appuser
password: s3cr3t
```

---


