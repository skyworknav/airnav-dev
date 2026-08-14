# Secret Management Guide: Project `airnav`

This document details how database credentials and application secrets are securely managed, synchronized, and injected into the Kubernetes cluster for **`airnav`** (`demo-app-staging` and `demo-app-prod` environments).

---

## 1. Architectural Strategy: Direct `.env` Synchronization

To maximize developer velocity, avoid Vault unsealing headaches, and reduce memory consumption on our K3s cluster, we use **Direct `.env` Secret Synchronization** from host `192.168.8.199`.

### Why No Vault or ESO?
- **Zero Lockouts**: No Vault master keys to unseal after server reboots.
- **Resource Savings**: Saves ~300MB of RAM and CPU on K3s nodes by eliminating External Secrets Operator (ESO) controller pods.
- **Developer Familiarity**: Developers manage plain `.env` files on host `192.168.8.199`.
- **GitOps Hygiene**: Credentials are **NEVER committed to GitHub**. Raw passwords stay strictly on host `192.168.8.199` and are injected dynamically into Kubernetes native `Secrets`.

### Why Push (`kubectl`) Instead of Pull (Network Fetch)?
We explicitly chose a **Push Model** (`192.168.8.199` pushing secrets into K3s via `kubectl`) over a **Pull Model** (VMs reaching across the network to fetch `.env` files from `192.168.8.199`) for 3 critical security and reliability reasons:

1. **Zero Open Ports / Hardened Host Security**: `192.168.8.199` does not need to run an open HTTP server or NFS share. If any worker VM in the cluster is compromised by an attacker, the attacker **cannot** reach across the network to steal raw `.env` files from `192.168.8.199`.
2. **Offline Resilience**: Once secrets are pushed, Kubernetes stores them encrypted in its native memory/etcd storage. If host `192.168.8.199` goes offline for maintenance, application pods in K3s can still restart smoothly without failing network dependencies.
3. **Controlled Ingress**: Secrets only move across network boundaries when an authorized administrator manually triggers `./sync-secrets.sh`.

---

## 2. Initial Setup on Central Host (`192.168.8.199`)

### Why Install `kubectl` on `192.168.8.199`?
Host `192.168.8.199` acts as the **Central Remote Control** for the K3s cluster (`192.168.10.20`). By installing `kubectl` and copying the `kubeconfig` key to `192.168.8.199`, the host server can convert local `.env` files directly into Kubernetes `Secret` objects across the network. This eliminates the need to SSH into the VM or install Vault/ESO!

If setting up a new host server (`192.168.8.199`), run these commands to install `kubectl` and connect it to the K3s cluster master (`192.168.10.20`):

```bash
# Step 1: Install the kubectl binary tool
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv kubectl /usr/local/bin/

# Step 2: Copy the K3s configuration from the Master Node (192.168.10.20)
mkdir -p ~/.kube
scp root@192.168.10.20:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Step 3: Point the configuration across the network to 192.168.10.20
sed -i 's/127.0.0.1/192.168.10.20/g' ~/.kube/config

# Step 4: Verify connectivity
kubectl get nodes
```

---

## 3. Where Credentials Live on Host (`192.168.8.199`)

Sensitive `.env` files are stored securely in `/opt/airnav/` on host `192.168.8.199`:

- **Staging `.env`**: `/opt/airnav/staging.env`
- **Production `.env`**: `/opt/airnav/prod.env`

### File Format (`/opt/airnav/prod.env`)
```ini
POSTGRES_DB=airnav_prod_db
POSTGRES_USER=airnav_prod_user
POSTGRES_PASSWORD=YourChosenSecurePassword123!
```

---

## 3. How to Update or Rotate Credentials

When a developer needs to update or change a password:

### Step 1: Log into Host Server (`192.168.8.199`)
```bash
ssh user@192.168.8.199
```

### Step 2: Edit the Environment File
Edit `/opt/airnav/staging.env` or `/opt/airnav/prod.env`:
```bash
nano /opt/airnav/prod.env
```

### Step 3: Run the 1-Line Secret Synchronization Script
Run the synchronization script to push the updated secrets directly into K3s:
```bash
/opt/airnav/sync-secrets.sh
```

---

## 4. How the Synchronization Script Works

The script `/opt/airnav/sync-secrets.sh` on host `192.168.8.199` uses native `kubectl` secret generation:

```bash
#!/bin/bash

# Ensure target namespaces exist in K3s
kubectl create namespace demo-app-staging --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace demo-app-prod --dry-run=client -o yaml | kubectl apply -f -

# 1. Sync Staging Secret to namespace 'demo-app-staging'
kubectl create secret generic postgres-secret \
  --from-env-file=/opt/airnav/staging.env \
  -n demo-app-staging \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. Sync Production Secret to namespace 'demo-app-prod'
kubectl create secret generic postgres-secret \
  --from-env-file=/opt/airnav/prod.env \
  -n demo-app-prod \
  --dry-run=client -o yaml | kubectl apply -f -

echo "Airnav secrets successfully synchronized to K3s cluster!"
```

---

## 5. How Kubernetes Pods Consume the Secret

Both the application (`cicd-demo-app`) and PostgreSQL (`postgres`) consume `postgres-secret` dynamically without hardcoding values in Git:

### Example: PostgreSQL StatefulSet (`postgres-statefulset.yaml`)
```yaml
env:
  - name: PGDATA
    value: "/var/lib/postgresql/data/pgdata"

  - name: POSTGRES_USER
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: POSTGRES_USER

  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: POSTGRES_PASSWORD

  - name: POSTGRES_DB
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: POSTGRES_DB
```

---

## 6. Password Rotation In-Place (Why `postgres-secret` Name Never Changes)

When you change a password inside `/opt/airnav/staging.env` or `prod.env`:
- **Do NOT create a new secret name**: The secret name `postgres-secret` stays permanent.
- **Do NOT edit application YAMLs**: `deployment.yaml` and `postgres-statefulset.yaml` stay untouched.

Running `/opt/airnav/sync-secrets.sh` uses `kubectl apply`, which **overwrites the secret in-place**. To force running pods to pick up the new password immediately:

```bash
kubectl rollout restart deployment/cicd-demo-app -n demo-app-prod
```

---

## 7. Secret Management Outside Kubernetes (Non-Cluster Setups)

If building non-Kubernetes environments (single Virtual Machines, Docker Compose, or Bare-Metal servers), Vault + ESO is **NOT** the only way to fetch credentials from an outside server. Below are the standard industry alternatives:

| Method | How It Works | Best Use Case |
| :--- | :--- | :--- |
| **Ansible / SSH Remote Push** | Server `192.168.8.199` runs Ansible to push template `.env` files directly to `/opt/app/.env` on target VMs over SSH. | Standard Linux VMs, Systemd services. |
| **SOPS / Mozilla SOPS** | Developers encrypt `.env` files using a private key (`.env.enc`) and commit encrypted files to Git. Target server decrypts at boot. | Open Git repositories, Docker Compose. |
| **REST API Fetch (Vault / AWS Secrets Manager)** | Application startup script calls HTTP API (`curl http://192.168.8.199:8200/v1/secret/...`) to load JSON credentials directly into RAM. | Microservices without K8s. |
| **Systemd `EnvironmentFile`** | Server loads local file `/etc/default/app.env` before launching service binary. | Bare-metal Linux servers. |

---

## 8. Summary Checklist for New Deployments

- [x] Host `192.168.8.199` has `/opt/airnav/staging.env` and `prod.env`.
- [x] Script `/opt/airnav/sync-secrets.sh` ran successfully.
- [x] Native `postgres-secret` exists in `demo-app-staging` and `demo-app-prod`.
- [x] `postgres-statefulset.yaml` contains `PGDATA: /var/lib/postgresql/data/pgdata` fix.
- [x] Argo CD synced application to cluster.
