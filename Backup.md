# Protect VM
- 把vm disk pv的persistentVolumeReclaimPolicy 改为Retain， 防止误删
- VolumeSnapshotContent 需要修改为 deletionPolicy: Retain
- 备份vm的时候还需要备份ControllerRevision，
```
PV_NAME=pvc-4a5ff80c-ff35-41a3-840d-12c86774c6ba
oc patch pv "$PV_NAME" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
VSC_NAME=snapcontent-49018d11-0d9f-435f-b25e-5e39e68cc497
oc patch volumesnapshotcontent "$VSC_NAME" \
  -p '{"spec":{"deletionPolicy":"Retain"}}' --type=merge
```

# backup
  defaultVolumesToFsBackup: false
  snapshotVolumes: false
  snapshotMoveData: false
## include
    - virtualmachines.kubevirt.io
    - virtualmachineinstances.kubevirt.io # 运行中vm备份需要备份vmi
    - datavolumes.cdi.kubevirt.io  #不会关联自动备份
    - pods   # 运行中vm备份需要备份pod    
    - Secrets
    - ConfigMaps
    - NetworkAttachmentDefinition
    - virtualmachinesnapshots.snapshot.kubevirt.io
    - VirtualMachineSnapshotContents.snapshot.kubevirt.io  #不会关联自动备份
    - volumesnapshots.snapshot.storage.k8s.io
    - persistentvolumeclaims
    - ControllerRevision

## Sample
```
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: dc1
  namespace: openshift-adp
  labels:
    app: centos01
spec:
  defaultVolumesToFsBackup: false
  csiSnapshotTimeout: 10m0s
  includedResources:
    - virtualmachines.kubevirt.io
    - virtualmachineinstances.kubevirt.io # 运行中vm备份需要备份vmi
    - datavolumes.cdi.kubevirt.io  #不会关联自动备份
    - pods   # 运行中vm备份需要备份pod    
    - Secrets
    - ConfigMaps
    - NetworkAttachmentDefinition
    - virtualmachinesnapshots.snapshot.kubevirt.io
    - VirtualMachineSnapshotContents.snapshot.kubevirt.io  #不会关联自动备份
    - volumesnapshots.snapshot.storage.k8s.io
    - persistentvolumeclaims
    - ControllerRevision
  ttl: 720h0m0s
  itemOperationTimeout: 4h0m0s
  storageLocation: default
  includeClusterResources: true
  includedNamespaces:
    - dc1
  snapshotVolumes: false
  snapshotMoveData: false

```

