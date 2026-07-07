# restore
existingResourcePolicy: none 或 update

## Sample

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: dc1-restore
  namespace: openshift-adp
spec:
  backupName: dc1
  includedResources:
    - virtualmachines.kubevirt.io
    - datavolumes.cdi.kubevirt.io  #不会关联自动备份  
    - Secrets
    - ConfigMaps
    - NetworkAttachmentDefinition
    - virtualmachinesnapshots.snapshot.kubevirt.io
    - VirtualMachineSnapshotContents.snapshot.kubevirt.io  #不会关联自动备份
    - volumesnapshots.snapshot.storage.k8s.io
    - persistentvolumeclaims
    - ControllerRevision
  excludedResources:
    - pods
    - virtualmachineinstances.kubevirt.io
  existingResourcePolicy: update
  includeClusterResources: true
  itemOperationTimeout: 4h0m0s
  namespaceMapping:
    dc1: dc2
  restorePVs: false
  restoreStatus: 
    includedResources:
      - virtualmachinesnapshots.snapshot.kubevirt.io
      - VirtualMachineSnapshotContents.snapshot.kubevirt.io



```

单独恢复存储相关组件
```
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: dc1-restorestorage
  namespace: openshift-adp
spec:
  backupName: dc1
  includedResources:
    - virtualmachinesnapshots.snapshot.kubevirt.io
    - VirtualMachineSnapshotContents.snapshot.kubevirt.io  #不会关联自动备份
    - volumesnapshots.snapshot.storage.k8s.io
    - volumesnapshotContents.snapshot.storage.k8s.io
    - persistentvolumeclaims
    - persistentvolumes
  excludedResources:
    - pods
    - virtualmachineinstances.kubevirt.io
  existingResourcePolicy: none
  includeClusterResources: true
  itemOperationTimeout: 4h0m0s
  namespaceMapping:
    dc1: dc2
  restorePVs: false
```

Restore snapshot

apiVersion: velero.io/v1
kind: Restore
metadata:
  name: dc1-restoresnapshot
  namespace: openshift-adp
spec:
  backupName: dc1
  includedResources:
    - volumesnapshots.snapshot.storage.k8s.io
    - volumesnapshotContents.snapshot.storage.k8s.io
  existingResourcePolicy: update
  includeClusterResources: true
  itemOperationTimeout: 4h0m0s
  namespaceMapping:
    dc1: dc2
