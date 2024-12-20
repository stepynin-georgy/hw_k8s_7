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
  name: busybox-multitool
  labels:
    app: busy-multi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busy-multi
  template:
    metadata:
      labels:
        app: busy-multi
    spec:
      containers:
      - name: app1
        image: busybox
        command: ['sh', '-c', 'while true; do echo "Домашнее задание к занятию «Хранение в K8s. Часть 2»" >> /output/success.txt; sleep 5; done']
        volumeMounts:
        - mountPath: /output
          name: example-volume


      - name: multitool
        image: wbitt/network-multitool
        command: ['sh', '-c', 'tail -f /input/success.txt']
        volumeMounts:
        - name: example-volume
          mountPath: /input
        ports:
        - containerPort: 8080
        env: 
          - name: HTTP_PORT
            value: "9080"
      volumes:
      - name: example-volume
        persistentVolumeClaim:
          claimName: example-pvc
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl apply -f deployment.yml
deployment.apps/busybox-multitool created
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
busybox-multitool-5fdfbf7979-md9pm   0/2     Pending   0          17s
```

2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/mnt/data"
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
# namespace: web
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
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
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
example-pv   1Gi        RWO            Retain           Bound    default/example-pvc                  <unset>                          8m56s
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pvc
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
example-pvc   Bound    example-pv   1Gi        RWO                           <unset>                 8m51s
```

3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl exec -ti busybox-multitool-5fdfbf7979-md9pm  -c multitool -- /bin/bash
busybox-multitool-5fdfbf7979-md9pm:/# cd /input/
busybox-multitool-5fdfbf7979-md9pm:/input# tail -f success.txt
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl logs busybox-multitool-5fdfbf7979-md9pm  multitool
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
```

4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl delete deployment busybox-multitool
deployment.apps "busybox-multitool" deleted
user@k8s:/opt/hw_k8s_7$ microk8s kubectl delete pvc example-pvc
persistentvolumeclaim "example-pvc" deleted

```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
example-pv   1Gi        RWO            Retain           Released   default/example-pvc                  <unset>                          13m

user@k8s:/opt/hw_k8s_7$ microk8s kubectl describe pv example-pv
Name:            example-pv
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:
Status:          Released
Claim:           default/example-pvc
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /mnt/data
    HostPathType:
Events:            <none>
```

Удалил Deployment и PVC. PV поменял статус с Bound на Released. До удаления PV был связан с PVC и Deployment, поэтому статус был Bound. А после удаления Deployment и PVC он уже ни с чем не связан, поэтому статус поменялся на Released.

5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.

```
user@k8s:/opt/hw_k8s_7$ tail -f /mnt/data/success.txt
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
Домашнее задание к занятию «Хранение в K8s. Часть 2»
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl delete pv example-pv
persistentvolume "example-pv" deleted
```

Файл сохранился:

```
user@k8s:/opt/hw_k8s_7$ ls -l /mnt/data/success.txt
-rw-r--r-- 1 root root 12880 Dec 20 13:01 /mnt/data/success.txt
```

Удалил PV, файл также сохранился на локальном диске ноды потому что значение в PV у persistentVolumeReclaimPolicy: Retain - это значит, что после удаления PV ресурсы из внешних провайдеров автоматически не удаляются.

5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

### Задание 2

**Что нужно сделать**

Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

1. Включить и настроить NFS-сервер на MicroK8S.

```

user@k8s:/opt/hw_k8s_7$ microk8s enable hostpath-storage
Infer repository core for addon hostpath-storage
Enabling default storage class.
WARNING: Hostpath storage is not suitable for production environments.
         A hostpath volume can grow beyond the size limit set in the volume claim manifest.

deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon.
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS        VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-5c00d1f6-ef4f-4b0c-8ffd-e059ecd93203   1Gi        RWX            Delete           Bound    default/example-pvc   microk8s-hostpath   <unset>                          13s
```

2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-multitool
  labels:
    app: d-multi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: d-multi
  template:
    metadata:
      labels:
        app: d-multi
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: example-volume
          mountPath: /input
      volumes:
      - name: example-volume
        persistentVolumeClaim:
          claimName: example-pvc
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "microk8s-hostpath"
  resources:
    requests:
      storage: 1Gi
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl apply -f deployment-nfs.yml
deployment.apps/dep-multitool created
user@k8s:/opt/hw_k8s_7$ microk8s kubectl apply -f pvc-nfs.yml
persistentvolumeclaim/example-pvc created
```

```
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE
example-pvc   Bound    pvc-5c00d1f6-ef4f-4b0c-8ffd-e059ecd93203   1Gi        RWX            microk8s-hostpath   <unset>                 3m51s
user@k8s:/opt/hw_k8s_7$ microk8s kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
dep-multitool-55c4b4867f-ml7pp   1/1     Running   0          4m3s
```

3. Продемонстрировать возможность чтения и записи файла изнутри пода.

```
dep-multitool-55c4b4867f-ml7pp:/# echo "test" > /input/test.txt
dep-multitool-55c4b4867f-ml7pp:/# cat /input/test.txt
test
```

4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
