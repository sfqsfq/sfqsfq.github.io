---
title: "K8s_pv_pvc"
date: 2023-03-16T17:11:22+08:00
draft: false

tags: ["Kubernetes", "PV", "PVC"]
categories: ["Kubernetes"]
---

- PV 是对底层网络共享存储的抽象
- PVC 是对 PV 的请求
- StorageClass作为对存储资源的抽象定义，对用户设置的PVC申请屏蔽后端存储的细节，一方面减少了用户对于存储资源细节的关注，另一方面减轻了管理员手工管理PV的工作，由系统自动完成PV的创建和绑定，实现了动态的资源供应。
- CSI 是 Container Storage Interface 的缩写，是一个规范，用于定义存储插件与 Kubernetes 之间的接口


## PV 

```yaml
apiVersion: v1
# PV
kind: PersistentVolume
metadata:
  name: pv1
spec:
  # 存储容量
  capacity:
    storage: 1Gi
  # 存储卷模式
  volumeMode: Filesystem
  # 访问模式
  accessModes:
    - ReadWriteOnce
  # 回收策略
  persistentVolumeReclaimPolicy: Recycle
  # 存储类型
  storageClassName: slow
  # 存储位置
  nfs:
    path: /data/pv1
    server: node3.sfq.me
```

## PVC
  
```yaml
apiVersion: v1
# PVC
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  # 资源请求
  resources:
    requests:
      storage: 1Gi
  # 访问模式
  accessModes:
    - ReadWriteOnce
  # 存储卷模式
  # volumeMode: Filesystem
  # 存储类型
  storageClassName: slow
  # PV 的选择条件
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
# 提供者，以 kubernetes.io/ 开头 
provisioner: kubernetes.io/nfs
# 参数
parameters:
  server: "node3.sfq.me"
  share: "/data/pv"
  mountOptions: "vers=4.1"
```

## 动态资源供应模式

### 1. 定义 storageClass

[参考](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)



```bash
sfqfs@sfq:~/k8s/storageClass$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
 nfs.ser>     --set nfs.server=node3.sfq.me \
>     --set nfs.path=/data/pv
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Fri Mar 17 00:38:06 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

sfqfs@sfq:~/k8s/storageClass$ kubectl get sc
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path                           Delete          WaitForFirstConsumer   false                  10d
nfs-client             cluster.local/nfs-subdir-external-provisioner   Delete          Immediate              true                   2m48s
```

### 2. 定义 PVC

```yaml
apiVersion: v1
# PVC
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  # 资源请求
  resources:
    requests:
      storage: 1Gi
  # 访问模式
  accessModes:
    - ReadWriteOnce
  # 存储卷模式
  # volumeMode: Filesystem
  # 存储类型
  storageClassName: nfs-client
```

### 3. Pod使用 PVC 的存储资源

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-test-pv
spec:
  containers:
  - name: pod-test-pv
    image: busybox
    command:
    - sh
    - -c
    - sleep 36000
    volumeMounts:
      - name: mypvc
        mountPath: /mnt/data
  volumes:
  - name: mypvc
    persistentVolumeClaim:
      claimName: pvc1
```



### 4. 测试

```bash
# 当前 pv
sfqfs@sfq:~/k8s/storageClass$ kubectl get pv
No resources found

# 创建 pvc
sfqfs@sfq:~/k8s/storageClass$ kubectl create -f pvc.yaml 
persistentvolumeclaim/pvc1 created
sfqfs@sfq:~/k8s/storageClass$ kubectl get pvc
NAME   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc1   Bound    pvc-74c60ca7-9e55-4a5d-8e1b-1d7e84ad72ba   1Gi        RWO            nfs-client     2s

# 查看 pv
sfqfs@sfq:~/k8s/storageClass$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS   REASON   AGE
pvc-74c60ca7-9e55-4a5d-8e1b-1d7e84ad72ba   1Gi        RWO            Delete           Bound    default/pvc1   nfs-client              15s

# 创建 pod
sfqfs@sfq:~/k8s/storageClass$ kubectl create -f pod.yaml 
pod/pod-test-pv created


sfqfs@sfq:~/k8s/storageClass$ kubectl exec -it pod/pod-test-pv -- ls /mnt/data/
aa.txt
sfqfs@sfq:~/k8s/storageClass$ ssh sfq@node3.sfq.me ls /data/pv
default-pvc1-pvc-74c60ca7-9e55-4a5d-8e1b-1d7e84ad72ba
sfqfs@sfq:~/k8s/storageClass$ ssh sfq@node3.sfq.me ls /data/pv/default-pvc1-pvc-74c60ca7-9e55-4a5d-8e1b-1d7e84ad72ba
aa.txt
```


## 静态资源供应模式

### 1. 手动创建 PV

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-sfq-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: node3.sfq.me
    path: /data/nfs
```

### 2. 创建 PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-sfq-nfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  # 静态存储管理 storageClassName 为空
  storageClassName: ""
  volumeName: pv-sfq-nfs
```


### 3. Pod 使用 PVC 的存储资源

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-sfq-pvc
spec:
  containers:
  - name: pod-sfq-pvc
    image: busybox
    command:
    - sh
    - -c
    - sleep 36000
    volumeMounts:
      - name: mypvc
        mountPath: /mnt/data
  volumes:
  - name: mypvc
    persistentVolumeClaim:
      claimName: pvc-sfq-nfs
```


### 4. 测试

```bash

# pv 创建
sfqfs@sfq:~/k8s/storageClass/static$ kubectl create -f pv.yml 
persistentvolume/pv-sfq-nfs created

sfqfs@sfq:~/k8s/storageClass/static$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM          STORAGECLASS   REASON   AGE
pvc-681662da-ec58-471c-be39-5dd17ffa1a53   1Gi        RWO            Delete           Bound       default/pvc1   nfs-client              10h
pv-sfq-nfs                                 10Gi       RWX            Retain           Available                                          3s


# pvc 创建
sfqfs@sfq:~/k8s/storageClass/static$ kubectl create -f pvc.yaml 
persistentvolumeclaim/pvc-sfq-nfs created

sfqfs@sfq:~/k8s/storageClass/static$ kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc1          Bound    pvc-681662da-ec58-471c-be39-5dd17ffa1a53   1Gi        RWO            nfs-client     10h
pvc-sfq-nfs   Bound    pv-sfq-nfs                                 10Gi       RWX                           2s

# pod 使用 pvc
sfqfs@sfq:~/k8s/storageClass/static$ kubectl get pod -w
NAME                                               READY   STATUS    RESTARTS      AGE
nginx-deployment-7bf89ffbcd-d29qq                  1/1     Running   0             11d
nginx-deployment-7bf89ffbcd-5s6sm                  1/1     Running   0             10d
nfs-subdir-external-provisioner-7757cd775b-hskkp   1/1     Running   0             10h
pod-test-pv                                        1/1     Running   1 (12m ago)   10h
pod-sfq-pvc                                        1/1     Running   0             4s

# 测试
sfqfs@sfq:~/k8s/storageClass/static$ kubectl exec -it pod-sfq-pvc -- touch /mnt/data/test
sfqfs@sfq:~/k8s/storageClass/static$ ssh sfq@node3.sfq.me ls -l /data/nfs/
total 0
-rw-r--r--. 1 root root 0 Mar 16 23:16 test
```