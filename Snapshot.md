# 从存储中恢复出volumesnapshot，volumesnapshotContent
## VolumeSnapshotContent
由于 VolumeSnapshotContent 是集群级别的资源，你需要确保它的 spec.volumeSnapshotRef 正确填写了即将创建的 VolumeSnapshot 的名称和命名空间。
VolumeSnapshotContent 动态制备模式和静态制备模式，区分这两种模式的依据完全取决于 VolumeSnapshotContent 的 spec.source 填充了什么字段：

- **静态（Pre-provisioned）**：spec.source 必须使用 snapshotHandle（指向存储厂商端已经存在的快照 ID）。
- **动态（Dynamically provisioned）**：spec.source 使用的是 volumeHandle（指向一个现有的真实数据卷，意图从这个卷动态创建出新快照）。


```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: my-existing-content-xyz # 你已有的 VolumeSnapshotContent 名称
spec:
  deletionPolicy: Retain # 建议设为 Retain，防止对应的 VolumeSnapshot 被删时底层快照被意外删除
  driver: openshift-storage.rbd.csi.ceph.com # 对应的 CSI 驱动名称，需与底层一致
  source:
    snapshotHandle: 0001-0011-openshift-storage-0000000000000001-7985a117-d4ef-453c-99ea-c9cbce44f3dd # 底层存储厂商真实的快照 ID
  volumeSnapshotClassName: ocs-storagecluster-rbdplugin-snapclass  
  volumeSnapshotRef:
    name: my-static-snapshot # 【核心】指定即将绑定的 VolumeSnapshot 的名字
    namespace: production # 【核心】指定即将绑定的 VolumeSnapshot 所在的命名空间
```    

## VolumeSnapshot
在定义 VolumeSnapshot 时，不要指定 volumeSnapshotClassName，而是要在 spec.source.volumeSnapshotContentName 中直接填入上一步的 VolumeSnapshotContent 的名称。

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-static-snapshot # 必须与上面 volumeSnapshotRef.name 一致
  namespace: production # 必须与上面 volumeSnapshotRef.namespace 一致
spec:
  source:
    volumeSnapshotContentName: my-existing-content-xyz # 【核心】指定已有的 VolumeSnapshotContent 名称

```

# vmsnapshot
在恢复vm之前恢复vmsnapshot 不会创建在恢复vm之前恢复vmsnapshotcontent， 这时候可以
oc patch vmsnapshot <vmsnapshot-name> -n <namespace> --subresource=status \
 --type=merge -p '{"status":{"virtualMachineSnapshotContentName":"vmsnapshot-content-a0798d49-1aa3-4d9f-96ff-21aebd1e44cd"}}'