---
title: "K8s_volume"
date: 2023-03-16T14:34:46+08:00
draft: false

tags: ["Kubernetes", "Volumes"]
categories: ["Kubernetes"]
---

需要持久化数据的程序会用到 Volumes。

- 日志收集需求，在应用程序的容器里面加一个 sidecar, 通过共享卷的方式，将日志收集到 sidecar 容器里面，再通过 sidecar 容器将日志收集到外部的日志收集系统。


## hostPath

hostPath 是最简单的 Volume 类型，它直接将 Pod 的 Volume 挂载到宿主机的目录上。

```bash
# 创建 Pod 资源对象
sfqfs@sfq:~$ cat<<EOF | kubectl create -f -
# pod-hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: tmp
      mountPath: /test
  volumes:
  # 将宿主的 /tmp 目录挂载到容器的 /test 目录
  - name: tmp
    hostPath:
      path: /tmp
  nodeSelector:
  # 挂载到 node4 节点上的目录
    node: node4
EOF


# 在 Pod 中创建文件，在宿主机上查看
sfqfs@sfq:~/k8s$ kubectl exec -it pod-hostpath -- touch /test/file-create-in-pod
sfqfs@sfq:~/k8s$ ssh sfq@node4.sfq.me ls /tmp
file-create-in-pod
systemd-private-7f857c3921084bd1ab8676ed07a91071-chronyd.service-8J5JY3
```



## emptyDir

emptyDir 是一个临时目录，当 Pod 被删除时，emptyDir 也会被删除。
用于存放 Pod 运行时的数据，比如日志文件，用于 Pod 中不同 Container 之间的数据共享。

```bash
# 创建 Pod 资源对象，容器 busybox-1 和 busybox-2 共享一个 emptyDir 卷
# 两个容器都可以读写这个卷
sfqfs@sfq:~$ cat<<EOF | kubectl create -f -
# pod-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir
spec:
  containers:
  - name: busybox-1
    image: busybox
    command: ["sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: shared
      mountPath: /test-1
  - name: busybox-2
    image: busybox
    command: ["sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: shared
      mountPath: /test-2
  volumes:
  # 共享卷
  - name: shared
    emptyDir: {}
EOF

sfqfs@sfq:~/k8s$ kubectl exec -it  pod-emptydir -c busybox-1 -- touch /test-1/file-from-busybox-1
sfqfs@sfq:~/k8s$ kubectl exec -it  pod-emptydir -c busybox-2 -- ls /test-2
file-from-busybox-1
```

## NFS


```bash
# 创建 NFS Volume 
sfqfs@sfq:~$ cat<<EOF | kubectl create -f -
# nfs-server.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-server
spec:
  containers:
  - name: pod-nfs-volume
    image: busybox
    command: ["sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: nfs-volume
      mountPath: /data
  volumes:
  - name: nfs-volume
    nfs:
      server: node3.sfq.me
      path: /data
EOF

sfqfs@sfq:~/k8s$ kubectl exec -it nfs-server -- df -Th
Filesystem           Type            Size      Used Available Use% Mounted on
overlay              overlay        50.0G      4.8G     45.2G  10% /
tmpfs                tmpfs          64.0M         0     64.0M   0% /dev
tmpfs                tmpfs           2.4G         0      2.4G   0% /sys/fs/cgroup
node3.sfq.me:/data   nfs4           50.0G      4.9G     45.1G  10% /data


sfqfs@sfq:~/k8s$ kubectl exec -it nfs-server -- touch /data/file-create-in-pod
sfqfs@sfq:~/k8s$ ssh sfq@node3.sfq.me ls /data
file-create-in-pod
```