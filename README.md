# Настройка External snapshotter для создания снапшотов pvc в Managed Kubernetes

Ниже представлен процесс настройки и использования External snapshotter в Kubernetes для создания снапшотов Persistent Volume Claims (PVC).
External snapshotter - это инструмент, который позволяет создавать и управлять снапшотами PVC в Kubernetes, используя средства облачной платформы на базе OpenStack. Мы рассмотрим шаги по установке и настройке External snapshotter, а также демонстрируем его использование для создания и восстановления снапшотов PVC в Managed Kubernetes. В данном случае, мы будем использовать [CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter).

Вам потребуется установить в кластер external-snapshotter, snapshot-controller и создать Custom Resource Definitions (CRDs) для снапшотов. Для этого выполним команду:

```shell
kubectl apply -f https://raw.githubusercontent.com/selectel/mks-csi-snapshotter/master/deploy/setup-snapshot-controller.yaml
```

После применения манифеста, проверяем, что созданные нами поды корректно развернуты. Для этого выполняем команду:

```shell
kubectl get pod -l app=snapshot-controller --namespace=kube-system
```

Для того, что бы протестировать развернутое нами решение создаем Persistent Volume Claim (PVC) и StorageClass для последующего создания снапшота. В данном случае, мы будем использовать кластер в зоне ru-3b. Создание PVC и StorageClass для ru-3b выполняется при помощи манифеста:

```yaml
---
# pvc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast.ru-3b
provisioner: cinder.csi.openstack.org
parameters:
  type: fast.ru-3b
  availability: ru-3b
  fsType: ext4
allowVolumeExpansion: true

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pv-claim
spec:
  storageClassName: fast.ru-3b
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Применяем манифест командой `kubectl apply -f pvc.yaml`.

Далее, по аналогии с PVC и StorageClass, создаем манифест `VolumeSnapshotClass` и `VolumeSnapshot`:

```yaml
# Snapshot.yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass-v1
driver: cinder.csi.openstack.org
deletionPolicy: Delete
parameters:
  force-create: "true"
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-demo
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass-v1 #<-- Тут необходимо использовать название SnapshotClass'а, созданного нами выше.
  source:
    persistentVolumeClaimName: my-pv-claim #<-- Тут необходимо использовать название PVC, снапшот которого вы собираетесь создать.
```

Применяем созданный нами манифест, выполнив команду `kubectl apply -f snapshot.yaml`.

## Восстановление из снапшота

Для восстановления PVC из созданного ранее снапшота необходимо создать дополнительный манифест.
Пример такого манифеста можно найти в нашем репозитории по ссылке. Давайте рассмотрим его более подробно.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: snapshot-demo-restore #<-- Тут необходимо дать название нового PVC
spec:
  storageClassName: fast.ru-3b
  dataSource:
    name: new-snapshot-demo #<-- Тут необходимо дать название ранее созданного снапшота, из которого выполняется восстановление
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Применяем данный манифест командой `kubectl apply -f snapshot-restore.yaml`.