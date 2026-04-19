# Velero

### What is Velero?

Velero is an open-source **Kubernetes backup and disaster recovery** tool.  
It helps in backing up and restoring:
- Cluster resources (Deployments, Services, Ingress, etc.)
- Persistent volume data (via **Restic** or **snapshots**)  
It stores these backups in an **object storage backend** (like MinIO, AWS S3, or GCS).

**Use cases:**
- Backup the entire cluster  
- Restore after accidental deletion or failure  
- Migrate workloads between clusters  

---

### How Velero Works (Architecture Overview)

Velero consists of two main components:
1. **Velero CLI** — A command-line tool to trigger backups, restores, and monitor status.  
2. **Velero Server (in-cluster)** — Runs inside Kubernetes as a Deployment (for control logic) and DaemonSet (for Restic node-agent) to perform actual backup and restore operations.

---

### Prerequisites
- Working Kubernetes cluster (`kubectl` access)
- `helm` and `velero` CLI installed
- Object storage (like MinIO or S3) accessible from the cluster

---

### Step 1: Install MinIO (as object storage for Velero)

Use Helm to deploy a standalone **MinIO** server that acts as an S3-compatible storage backend:

```bash
helm repo add minio https://charts.min.io/
helm repo update

helm install my-minio minio/minio \
  --namespace minio \
  --create-namespace \
  --set replicas=1 \
  --set mode=standalone \
  --set persistence.size=3Gi \
  --set rootUser=admin,rootPassword=admin12345 \
  --set resources.requests.memory=512Mi

### Step 1: Expose MinIO API Service
kubectl patch svc my-minio -n minio -p '{"spec":{"type":"NodePort"}}'
kubectl get svc my-minio -n minio
# Use the NodePort displayed as s3Url=http://<NodeIP>:<NodePort> in Velero install

### Step 2: Install Velero CLI
curl -fsSL https://github.com/vmware-tanzu/velero/releases/download/v1.18.0/velero-v1.18.0-linux-amd64.tar.gz -o velero.tar.gz
tar -xzf velero.tar.gz
sudo mv velero-v*/velero /usr/local/bin/
velero version

### Step 3: Prepare a Backup Storage Location
# Create credentials file `credentials-velero`:
[default]
aws_access_key_id=admin
aws_secret_access_key=admin12345
# Create bucket `velero-backups` in MinIO console: http://<NodeIP>:9001

### Step 4: Install Velero in Kubernetes
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://100.64.188.228:31712 \
  --use-node-agent \
  --uploader-type restic \
  --snapshot-location-config region=minio \
  --namespace velero

# Verify
kubectl get pods -n velero
velero backup-location get

### Step 5: Take a Backup
# Namespace backup
velero backup create mybackup --include-namespaces default
# Full cluster backup
# use flag --default-volumes-to-fs-backup for pvc data backup
velero backup create full-cluster-backup --include-namespaces '*' --wait
# Check status
velero backup get
velero backup describe full-cluster-backup --details

### Step 6: Restore from Backup
velero restore create --from-backup mybackup
velero restore get

### Step 7: Schedule Backups Automatically
velero schedule create daily-backup \
  --schedule "0 2 * * *" \
  --include-namespaces default
velero schedule get
