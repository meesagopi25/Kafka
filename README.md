# Kafka

Below is a **ready-to-run operational backup and restore runbook** for a **Kafka cluster deployed on Azure Red Hat OpenShift (ARO) using KRaft mode**.

It is written as a **production-grade runbook**: actionable, ordered, and safe.

---

# Kafka (KRaft) Backup & Restore Runbook

**Platform:** Azure Red Hat OpenShift (ARO)
**Kafka Mode:** KRaft (No ZooKeeper)
**Deployment:** StatefulSets (Controllers + Brokers)
**Storage:** Azure Managed Disks (CSI)

---

## 1. Purpose

This document defines the **standard operating procedure (SOP)** for **backup and restore** of a Kafka cluster running in **KRaft mode** on **ARO**.

It ensures:

* Metadata safety (KRaft controllers)
* Topic data protection (brokers)
* Predictable recovery (preserved cluster identity)
* Minimal downtime

---

## 2. Scope

✔ Kafka **KRaft mode only**
✔ OpenShift / ARO
✔ Bitnami / Helm-based deployments
❌ ZooKeeper-based Kafka (out of scope)

---

## 3. Architecture Overview (KRaft)

| Component   | Responsibility                               |
| ----------- | -------------------------------------------- |
| Controllers | Metadata, Raft quorum, leader election       |
| Brokers     | Topic data, partitions, client traffic       |
| PVCs        | Persistent storage for controllers & brokers |
| Cluster ID  | Immutable identifier for the Kafka cluster   |

> **Critical Rule:**
> **Controllers must always be backed up and restored before brokers.**

---

## 4. Preconditions

Before performing any backup or restore:

* Cluster is **Healthy**
* No under-replicated partitions
* No controller quorum issues
* Snapshot support enabled in ARO

Verify snapshot capability:

```bash
oc get volumesnapshotclass
```

---

## 5. Cluster Health Validation (MANDATORY)

### 5.1 Check KRaft Quorum

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-metadata-quorum.sh \
--bootstrap-server localhost:9092 \
--describe
```

Expected:

* One leader
* All voters in sync
* No lag

---

### 5.2 Check Topic Replication

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

Ensure:

* `UnderReplicatedPartitions: 0`

---

## 6. Logical Metadata Backup (ALWAYS DO)

### 6.1 Topics & Configurations

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-topics.sh --bootstrap-server localhost:9092 --list > topics.txt
```

```bash
for t in $(cat topics.txt); do
  oc exec -n kafka kafka-broker-0 -- \
  kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name "$t" \
  --describe
done > topic-configs.txt
```

---

### 6.2 ACLs

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-acls.sh --bootstrap-server localhost:9092 --list > acls.txt
```

---

### 6.3 Consumer Group Offsets

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-consumer-groups.sh \
--bootstrap-server localhost:9092 \
--all-groups --describe > consumer-offsets.txt
```

---

### 6.4 Helm & Kubernetes State

```bash
helm get values kafka -n kafka > helm-values-backup.yaml
oc get sts -n kafka -o yaml > kafka-statefulsets.yaml
oc get secrets -n kafka -o yaml > kafka-secrets.yaml
```

Store these artifacts in **secure object storage**.

---

## 7. Persistent Volume Backup (KRaft-Safe Order)

### Backup Order (STRICT)

1. Controllers (one at a time)
2. Brokers (one at a time)

---

## 8. Controller PVC Backup (CRITICAL)

### 8.1 Stop One Controller Pod

```bash
oc delete pod kafka-controller-0 -n kafka --grace-period=60
```

> Quorum remains available (2/3).

---

### 8.2 Create VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: kafka-controller-0-snap
  namespace: kafka
spec:
  volumeSnapshotClassName: azuredisk-csi-snapshot
  source:
    persistentVolumeClaimName: data-kafka-controller-0
```

```bash
oc apply -f controller-snapshot.yaml
```

---

### 8.3 Verify Snapshot

```bash
oc get volumesnapshot kafka-controller-0-snap -n kafka
```

Ensure:

```text
readyToUse: true
```

---

### 8.4 Repeat

Repeat **Steps 8.1 – 8.3** for:

* `kafka-controller-1`
* `kafka-controller-2`

---

## 9. Broker PVC Backup (Rolling)

Repeat **one broker at a time**:

```bash
oc delete pod kafka-broker-1 -n kafka --grace-period=60
```

Create snapshot:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: kafka-broker-1-snap
  namespace: kafka
spec:
  volumeSnapshotClassName: azuredisk-csi-snapshot
  source:
    persistentVolumeClaimName: data-kafka-broker-1
```

Verify snapshot readiness, then proceed to next broker.

---

## 10. Restore Procedure (Disaster Recovery)

### ⚠️ Preconditions

* Same Kafka version
* Same replica counts
* **Same KRaft clusterId**
* Controllers restored first

---

## 11. Restore Controller PVCs (FIRST)

### 11.1 Create PVC from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-kafka-controller-0
  namespace: kafka
spec:
  storageClassName: managed-premium
  dataSource:
    name: kafka-controller-0-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Repeat for all controllers.

---

### 11.2 Start Controllers

```bash
oc scale sts kafka-controller -n kafka --replicas=3
```

Validate quorum again.

---

## 12. Restore Broker PVCs

Restore broker PVCs from snapshots using **original PVC names**, then:

```bash
oc scale sts kafka-broker -n kafka --replicas=3
```

---

## 13. Post-Restore Validation

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

```bash
oc exec -n kafka kafka-broker-0 -- \
kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

Confirm:

* Topics exist
* No under-replicated partitions
* Controllers healthy

---

## 14. Reapply Logical Metadata (If Required)

* Recreate ACLs from `acls.txt`
* Reset consumer offsets if needed
* Reapply topic configs

---

## 15. DOs & DON’Ts (KRaft-Specific)

### ✅ DO

* Backup controllers first
* Preserve clusterId
* Restore controllers before brokers
* Use rolling snapshots
* Test restore quarterly

### ❌ DON’T

* Change clusterId
* Restore brokers before controllers
* Mix snapshots from different clusters
* Load-balance controllers
* Skip quorum validation

---

## 16. Incident Checklist (Quick)

* [ ] Cluster health verified
* [ ] Logical metadata backed up
* [ ] Controller PVC snapshots complete
* [ ] Broker PVC snapshots complete
* [ ] Restore tested in non-prod
* [ ] Runbook updated

---

## 17. Key Summary

> “In KRaft mode on OpenShift, Kafka metadata resides in controller disks, so backup and restore must prioritize controller PVC snapshots before broker data, while preserving the immutable clusterId.”

---

## 18. Ownership

**Platform:** ARO
**Kafka:** Platform Engineering
**Runbook Version:** 1.0
**Review Cycle:** Quarterly

---
