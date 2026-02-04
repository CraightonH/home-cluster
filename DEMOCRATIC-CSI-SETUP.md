# Democratic CSI for Synology iSCSI Storage

## Overview

Democratic CSI provides dynamic provisioning of Synology iSCSI LUNs as Kubernetes persistent volumes. Unlike NFS storage (which can lose data on cluster rebuilds), iSCSI provides block-level storage that persists on the NAS.

## Why Democratic CSI?

- **Block storage**: Better performance for databases (PostgreSQL, etc.)
- **Data persistence**: Survives cluster rebuilds (data lives on NAS)
- **Dynamic provisioning**: Auto-creates iSCSI LUNs on demand
- **Talos compatible**: Handles iSCSI initiator configuration automatically
- **Active development**: Better maintained than official synology-csi

## Architecture

```
┌─────────────────────────────────────────────────┐
│ Kubernetes Cluster (Talos)                      │
│                                                  │
│  ┌──────────────┐        ┌──────────────┐       │
│  │ PostgreSQL   │        │ Other Apps   │       │
│  │ Pod          │        │ Pod          │       │
│  └──────┬───────┘        └──────┬───────┘       │
│         │ PVC                   │ PVC            │
│         │                       │                │
│  ┌──────┴───────────────────────┴───────┐       │
│  │ Democratic CSI Driver (DaemonSet)    │       │
│  │ - Controller: Creates LUNs           │       │
│  │ - Node: Mounts iSCSI volumes         │       │
│  └──────────────┬───────────────────────┘       │
│                 │ iSCSI (port 3260)              │
└─────────────────┼───────────────────────────────┘
                  │
                  ▼
     ┌────────────────────────────┐
     │ Synology NAS               │
     │                            │
     │  ┌──────────────────────┐  │
     │  │ iSCSI Target & LUNs  │  │
     │  │ /volume12            │  │
     │  └──────────────────────┘  │
     └────────────────────────────┘
```

## Prerequisites

### 1. Synology NAS Configuration

**Enable iSCSI Service:**
1. DSM → **Storage Manager** → **iSCSI**
2. Enable **iSCSI service** (port 3260)

**No manual LUN creation needed** - Democratic CSI creates LUNs dynamically.

### 2. Synology Credentials

Ensure these are set in your cluster secrets (Doppler `home/main`):
- `SYNOLOGY_HOST` - NAS IP (e.g., 192.168.1.20)
- `SYNOLOGY_USERNAME` - Admin user (e.g., `clawd`)
- `SYNOLOGY_PASSWORD` - Admin password

### 3. Kubernetes Cluster Node IPs

Democratic CSI will configure iSCSI initiators on each node. Your Talos nodes:
- node1: 192.168.1.241
- node2: 192.168.1.242
- node3: 192.168.1.243
- node4: 192.168.1.244
- node5: 192.168.1.245

## Synology Host Configuration (Access Control)

After the CSI driver deploys, configure iSCSI host access:

### Step 1: Identify Initiator IQNs

After democratic-csi starts, check the node IQNs:

```bash
# On each node (via talosctl):
talosctl -n 192.168.1.241 read /etc/iscsi/initiatorname.iscsi --talosconfig kubernetes/bootstrap/talos/clusterconfig/talosconfig

# Expected format:
# iqn.2005-03.org.open-iscsi:node1
```

### Step 2: Create Host in Synology DSM

1. DSM → **Storage Manager** → **iSCSI** → **Host** tab
2. Click **Add**
3. Fill in host details:
   - **Name**: `k8s-cluster`
   - **Description**: `Kubernetes Talos nodes`
   - **Operating System**: `Linux`
   - **Protocol**: `iSCSI`

4. **Add Initiators**:
   - Enter each node's IQN (from Step 1):
     ```
     iqn.2005-03.org.open-iscsi:node1
     iqn.2005-03.org.open-iscsi:node2
     iqn.2005-03.org.open-iscsi:node3
     iqn.2005-03.org.open-iscsi:node4
     iqn.2005-03.org.open-iscsi:node5
     ```
   - Click **Next**

5. **Select LUNs**:
   - Democratic CSI will create LUNs dynamically
   - Select **"Allow access to all LUNs"** for now
   - Or restrict to specific LUN patterns later
   - Click **Next**

6. **Set Permissions**:
   - **Read/Write** access
   - Click **Next** → **Done**

## Deployment

### 1. Merge PR

Once this PR is merged, Flux will automatically:
- Add democratic-csi Helm repository
- Deploy the CSI driver (controller + node daemonset)
- Create `synology-iscsi` storage class

### 2. Verify Deployment

```bash
# Check CSI driver pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=democratic-csi-iscsi

# Check storage class
kubectl get sc synology-iscsi

# Check CSI driver
kubectl get csidriver org.democratic-csi.iscsi
```

### 3. Test with Sample PVC

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-iscsi-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: synology-iscsi
  resources:
    requests:
      storage: 10Gi
```

Apply and verify:
```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc test-iscsi-pvc

# Should show Bound status
# Check Synology DSM - should see new LUN created
```

## Storage Class Details

**Name**: `synology-iscsi`

**Features**:
- **Reclaim Policy**: `Retain` (PV survives PVC deletion)
- **Volume Expansion**: Enabled (can grow volumes)
- **Binding Mode**: `Immediate` (provisions on PVC creation)
- **Filesystem**: `ext4` (default)
- **Access Modes**: `ReadWriteOnce` (single node access)

**Location**: `/volume12` on Synology NAS

## Migration Path: PostgreSQL from Longhorn

Once tested, migrate PostgreSQL from Longhorn to iSCSI:

1. **Backup PostgreSQL data**
2. **Create new PVC** with `synology-iscsi` storage class
3. **Update CloudNativePG cluster** to use new PVC
4. **Restore data** to new volume
5. **Verify** database functionality
6. **Delete old Longhorn PVC**

## Troubleshooting

### Pods stuck in ContainerCreating

Check CSI driver logs:
```bash
kubectl logs -n kube-system -l app=democratic-csi-iscsi-controller
kubectl logs -n kube-system -l app=democratic-csi-iscsi-node
```

Common issues:
- Synology credentials incorrect
- iSCSI service not enabled on NAS
- Network connectivity issues (firewall blocking port 3260)
- Node IQN not added to Synology host

### LUN not created on Synology

- Check CSI controller logs for API errors
- Verify `SYNOLOGY_HOST`, `SYNOLOGY_USERNAME`, `SYNOLOGY_PASSWORD`
- Ensure admin user has permissions to create iSCSI LUNs

### Mount errors on nodes

- Check node daemonset logs
- Verify iSCSI initiator is configured: `talosctl -n <node> read /etc/iscsi/initiatorname.iscsi`
- Check Synology host configuration includes node's IQN

## Configuration Details

### Democratic CSI Driver Config

Key configuration in `helmrelease.yaml`:

```yaml
driver:
  config:
    driver: synology-iscsi
    
    httpConnection:
      protocol: https
      host: "${SYNOLOGY_HOST}"
      port: 5001
      
    synology:
      locations:
        - "/volume12"  # Where LUNs are stored
        
    iscsi:
      targetPortal: "${SYNOLOGY_HOST}:3260"
      baseiqn: "iqn.2000-01.com.synology:k8s."
      
      # Templates for LUN/target names
      lunTemplate: "{{ .parameters.[csi.storage.k8s.io/pvc/namespace] }}-{{ .parameters.[csi.storage.k8s.io/pvc/name] }}"
      targetTemplate: "{{ .parameters.[csi.storage.k8s.io/pvc/namespace] }}-{{ .parameters.[csi.storage.k8s.io/pvc/name] }}"
```

### LUN Naming Convention

Democratic CSI creates LUNs with this pattern:
```
csi-<namespace>-<pvc-name>
```

Example:
- PVC `postgres-data` in namespace `db` → LUN name: `csi-db-postgres-data`

## Security Considerations

- **Access Control**: Configure Synology host to restrict access by IQN
- **CHAP Authentication**: Optional, can be enabled in driver config
- **Network Isolation**: Keep iSCSI traffic on trusted network (VLAN)
- **TLS**: Driver uses HTTPS to Synology API (port 5001)

## Performance Notes

- **RAID Type**: SHR on Synology provides redundancy with slight performance trade-off
- **Network**: 1Gbit sufficient for most workloads; 10Gbit for high throughput
- **Volume Size**: Start small, expand as needed (expansion is supported)

## Resources

- [Democratic CSI GitHub](https://github.com/democratic-csi/democratic-csi)
- [Democratic CSI Docs](https://github.com/democratic-csi/democratic-csi/tree/master/docs)
- [Synology iSCSI Guide](https://kb.synology.com/en-global/DSM/tutorial/What_is_iSCSI)
- [Talos Linux Storage Docs](https://www.talos.dev/latest/talos-guides/configuration/storage/)

## Next Steps

After successful deployment:

- [ ] Test PVC creation and mounting
- [ ] Verify LUN appears in Synology DSM
- [ ] Migrate PostgreSQL from Longhorn
- [ ] Consider removing Longhorn from cluster
- [ ] Monitor performance and adjust as needed
