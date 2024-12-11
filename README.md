# Домашнее задание к занятию «Хранение в K8s. Часть 2»

### Цель задания

В тестовой среде Kubernetes нужно создать PV и продемострировать запись и хранение файлов.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке NFS в MicroK8S](https://microk8s.io/docs/nfs). 
2. [Описание Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). 
3. [Описание динамического провижининга](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). 
4. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1

**Что нужно сделать**

Создать Deployment приложения, использующего локальный PV, созданный вручную.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-hw2
spec:
  selector:
    matchLabels:
      app: volume-hw2
  replicas: 1
  template:
    metadata:
      labels:
        app: volume-hw2
    spec:
      containers:
      - name: busybox
        image: busybox:1.28
        command: ['sh', '-c', 'mkdir -p /volumes && while true; do echo "$(date) - Test message" >> /volumes/success.txt; sleep 5; done']
        volumeMounts:
        - name: volume
          mountPath: /volumes
      - name: multitool
        image: wbitt/network-multitool
        command: ['sh', '-c', 'tail -f /volumes/success.txt']
        volumeMounts:
        - name: volume
          mountPath: /volumes
      volumes:
      - name: volume
        persistentVolumeClaim:
          claimName: pvc-vol
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl apply -f deployment.yml
deployment.apps/volume-hw2 created
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
volume-hw2-5746cfb4d6-2rfps   0/2     Pending   0          7s
```

2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  hostPath:
    path: /data/pvc-first
```

```

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-vol
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl apply -f pv.yml
persistentvolume/local-volume created
user@k8s:/opt/hw_k8s_7$ microk8s kubectl apply -f pvc.yml
persistentvolumeclaim/pvc-vol created
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-volume   1Gi        RWO            Delete           Bound    default/pvc-vol   local-storage   <unset>                          14s
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pvc
NAME      STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
pvc-vol   Bound    local-volume   1Gi        RWO            local-storage   <unset>                 15s
```

3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl exec -ti volume-hw2-5746cfb4d6-2rfps -c multitool -- /bin/bash
volume-hw2-5746cfb4d6-2rfps:/# cd /volumes/
volume-hw2-5746cfb4d6-2rfps:/volumes# tail -f success.txt
Wed Dec 11 11:38:37 UTC 2024 - Test message
Wed Dec 11 11:38:42 UTC 2024 - Test message
Wed Dec 11 11:38:47 UTC 2024 - Test message
Wed Dec 11 11:38:52 UTC 2024 - Test message
Wed Dec 11 11:38:57 UTC 2024 - Test message
Wed Dec 11 11:39:02 UTC 2024 - Test message
Wed Dec 11 11:39:07 UTC 2024 - Test message
Wed Dec 11 11:39:12 UTC 2024 - Test message
Wed Dec 11 11:39:17 UTC 2024 - Test message
Wed Dec 11 11:39:22 UTC 2024 - Test message
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl logs volume-hw2-5746cfb4d6-2rfps multitool
Wed Dec 11 11:28:27 UTC 2024 - Test message
Wed Dec 11 11:28:32 UTC 2024 - Test message
Wed Dec 11 11:28:37 UTC 2024 - Test message
Wed Dec 11 11:28:42 UTC 2024 - Test message
Wed Dec 11 11:28:47 UTC 2024 - Test message
Wed Dec 11 11:28:52 UTC 2024 - Test message
Wed Dec 11 11:28:57 UTC 2024 - Test message
Wed Dec 11 11:29:02 UTC 2024 - Test message
Wed Dec 11 11:29:07 UTC 2024 - Test message
Wed Dec 11 11:29:12 UTC 2024 - Test message
Wed Dec 11 11:29:17 UTC 2024 - Test message
Wed Dec 11 11:29:22 UTC 2024 - Test message
```

4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl delete deployment volume-hw2
deployment.apps "volume-hw2" deleted
user@k8s:/opt/hw_k8s_7$ microk8s kubectl delete pvc pvc-vol
persistentvolumeclaim "pvc-vol" deleted
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-volume   1Gi        RWO            Delete           Failed   default/pvc-vol   local-storage   <unset>                          15m

user@k8s:/opt/hw_k8s_7$ microk8s kubectl describe pv local-volume
Name:            local-volume
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    local-storage
Status:          Failed
Claim:           default/pvc-vol
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:         host_path deleter only supports /tmp/.+ but received provided /data/pvc-first
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /data/pvc-first
    HostPathType:
Events:
  Type     Reason              Age   From                         Message
  ----     ------              ----  ----                         -------
  Warning  VolumeFailedDelete  68s   persistentvolume-controller  host_path deleter only supports /tmp/.+ but received provided /data/pvc-first
```

PV перешел в состояние Failed, т.к. контроллер PV не сумел удалить данные по пути /data/pvc-first. По умолчанию он может удалить только данные по пути /tmp. Если бы там находились файлы, то они были бы утеряны.

5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.

```
user@k8s:/opt/hw_k8s_7$ ls -l /data/pvc-first/success.txt
-rw-r--r-- 1 root root 7788 Dec 11 11:43 /data/pvc-first/success.txt
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl delete pv local-volume
persistentvolume "local-volume" deleted
```

Файл сохранился:

```
user@k8s:/opt/hw_k8s_7$ ls -l /data/pvc-first/success.txt
-rw-r--r-- 1 root root 7788 Dec 11 11:43 /data/pvc-first/success.txt
```

После удаления PV, файл в директории /data/pvc-first останется на месте из-за особенностей работы контроллера PV с hostPath. В случае если в манифесте PV политика persistentVolumeReclaimPolicy будет установлена в Recycle, то файл будет удален.

5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

### Задание 2

**Что нужно сделать**

Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

1. Включить и настроить NFS-сервер на MicroK8S.

```
user@k8s:/opt/hw_k8s_7$ microk8s enable nfs
Infer repository community for addon nfs
Infer repository core for addon helm3
Addon core/helm3 is already enabled
Installing NFS Server Provisioner - Helm Chart 1.4.0

Node Name not defined. NFS Server Provisioner will be deployed on random Microk8s Node.

If you want to use a dedicated (large disk space) Node as NFS Server, disable the Addon and start over: microk8s enable nfs -n NODE_NAME
Lookup Microk8s Node name as: kubectl get node -o yaml | grep 'kubernetes.io/hostname'

Preparing PV for NFS Server Provisioner

persistentvolume/data-nfs-server-provisioner-0 created
"nfs-ganesha-server-and-external-provisioner" has been added to your repositories
Release "nfs-server-provisioner" does not exist. Installing it now.
NAME: nfs-server-provisioner
LAST DEPLOYED: Wed Dec 11 13:15:19 2024
NAMESPACE: nfs-server-provisioner
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NFS Provisioner service has now been installed.
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pv
NAME                            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
data-nfs-server-provisioner-0   1Gi        RWO            Retain           Bound    nfs-server-provisioner/data-nfs-server-provisioner-0                  <unset>                          24m
```

2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-hw2-nfs
  labels:
    app: volume-hw2-nfs
spec:
  selector:
    matchLabels:
      app: volume-hw2-nfs
  replicas: 1
  template:
    metadata:
      labels:
        app: volume-hw2-nfs
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "1180"
        volumeMounts:
        - name: nfs-storage
          mountPath: "/data"
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```



3. Продемонстрировать возможность чтения и записи файла изнутри пода. 
4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
