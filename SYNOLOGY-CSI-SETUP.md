# Synology CSI iSCSI Setup Guide

## Prerequisites

### 1. Create iSCSI LUN on Synology NAS

**Steps:**
1. Log into DSM (Synology DiskStation Manager)
2. Open **Storage Manager** → **iSCSI** → **LUN**
3. Click **Create** → **Create iSCSI LUN (Regular Files)**
4. Configure:
   - **Name**: `k8s-iscsi-lun-01` (or similar)
   - **Location**: `/volume1` (or your preferred volume)
   - **Capacity**: Start with 100GB (can expand later)
   - **Thin Provisioning**: Enabled (recommended)
5. Click **Next** → **Apply**

### 2. Create iSCSI Target (if needed)

1. In **Storage Manager** → **iSCSI** → **Target**
2. Click **Create**
3. Configure:
   - **Name**: `k8s-target`
   - **IQN**: Auto-generated is fine
   - **CHAP Authentication**: Disabled (or configure if needed)
   - **Map LUNs**: Select the LUN created above
4. Click **Apply**

### 3. Verify Synology Credentials in Doppler

The CSI driver needs:
- `SYNOLOGY_HOST` - NAS IP/hostname (e.g., `192.168.1.20`)
- `SYNOLOGY_USERNAME` - Admin user (`clawd`)
- `SYNOLOGY_PASSWORD` - Admin password

These should already exist in Doppler `home/main` project.

## Deployment

After creating the iSCSI LUN:

```bash
# Merge the PR
git merge feat/add-synology-csi-iscsi

# Flux will automatically:
# 1. Add the synology-csi Helm repository
# 2. Deploy the CSI driver to kube-system namespace
# 3. Create the "synology-iscsi" storage class

# Wait for deployment
flux reconcile kustomization --namespace flux-system flux-system
kubectl get pods -n kube-system | grep synology-csi

# Verify storage class
kubectl get sc synology-iscsi
```

## Migrating PostgreSQL to iSCSI

Once the CSI driver is running:

```bash
# 1. Create new PVC using synology-iscsi storage class
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-1-iscsi
  namespace: db
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: synology-iscsi
  resources:
    requests:
      storage: 100Gi
EOF

# 2. Scale down PostgreSQL
kubectl scale cluster/postgres -n db --replicas=0

# 3. Copy data from old PVC to new PVC (using a migration pod)
# 4. Update PostgreSQL cluster to use new PVC
# 5. Scale back up
```

## Storage Class Features

**synology-iscsi** storage class provides:
- **Block storage**: Better for database random I/O
- **Volume expansion**: Can grow volumes online
- **Snapshots**: Volume snapshot support
- **Retain policy**: PVs persist after PVC deletion (data safety)
- **Immediate binding**: Volumes bound immediately on creation

## Troubleshooting

### CSI driver pods not starting
```bash
kubectl logs -n kube-system -l app=synology-csi
```

### Check iSCSI connectivity
```bash
# From a node:
iscsiadm -m discovery -t st -p 192.168.1.20:3260
```

### Verify storage class
```bash
kubectl describe sc synology-iscsi
```

## Performance Benefits vs NFS

**iSCSI advantages:**
- Lower latency (block vs file protocol)
- Better random I/O performance
- No file locking overhead
- More suitable for databases

**NFS advantages:**
- ReadWriteMany support
- Simpler configuration
- Better for shared file storage

For PostgreSQL: **iSCSI is the better choice**.
