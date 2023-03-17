---
title: "K8S ConfigMap Secret"
date: 2023-03-14T10:40:31+08:00
draft: false

tags: ["Kubernetes", "ConfigMap", "Secret"]
categories: ["Kubernetes"]
---

ConfigMap 供容器使用的典型用法
- 生成为容器内的环境变量
- 设置容器启动命令的启动参数（需设置为环境变量）
- 以Volume的形式挂载为容器内部的文件或目录


## 使用YAML文件创建ConfigMap
```bash
# 创建 ConfigMap 资源对象
sfqfs@sfq:~$ cat<<EOF | kubectl create -f -
# cm-appvars.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-appvars
data:
  apploglevel: "info"
  appdatadir: /var/data
EOF

# 获取 configmap 对象
sfqfs@sfq:~$ kubectl get cm/cm-appvars
NAME         DATA   AGE
cm-appvars   2      68s

# 打印 configmap 的详细信息
sfqfs@sfq:~$ kubectl describe cm/cm-appvars
Name:         cm-appvars
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
appdatadir:
----
/var/data
apploglevel:
----
info

BinaryData
====

Events:  <none>
```


## 命令行方式创建

```bash
# 可以指定 key, 没有指定直接使用文件名作为key
# source 可以为目录，此时把当前文件jian的所有文件作为key, 文件内容作为value
kubectl create configmap cm-test1 --from-file=[key=]source --from-file=[key=]source

sfqfs@sfq:~/k8s/StatefulSet$ ls
busybox.yaml                 nginx-sts.yaml          statefulset.md
mongo-headless-service.yaml  statefulset-mongo.yaml  storageclass-fast.yaml
sfqfs@sfq:~/k8s/StatefulSet$ kubectl create configmap cm-files --from-file=.
configmap/cm-files created
sfqfs@sfq:~/k8s/StatefulSet$ kubectl get cm/cm-files
NAME       DATA   AGE
cm-files   6      4s


sfqfs@sfq:~/k8s/ingress-nginx$ kubectl create configmap cm-literals --from-literal=key1=value1
configmap/cm-literals created
sfqfs@sfq:~/k8s/ingress-nginx$ kubectl describe cm/cm-literals
Name:         cm-literals
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
key1:
----
value1

BinaryData
====

Events:  <none>
```


## 使用 env.valueFrom.configMapKeyRef 获取 ConfigMap 中的值到环境变量

```bash
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-env
spec:
  restartPolicy: Never
  containers:
  - name: cm-test
    image: busybox
    command: ["/bin/sh", "-c", "env | grep APP"]
    env:
    # 定义环境变量名称
    - name: APPLOGLEVEL
    # 定义环境变量对应的值
      valueFrom:
        configMapKeyRef:
        # 指定环境变量来自cm/cm-appvars
          name: cm-appvars
        # 指定环境变量在 cm/cm-appvars 中的键
          key: apploglevel
    - name: APPDATADIR
      valueFrom:
        configMapKeyRef:
          name: cm-appvars
          key: appdatadir
EOF

sfqfs@sfq:~/k8s$  kubectl logs pod/pod-cm-env
APPDATADIR=/var/data
APPLOGLEVEL=info xxdsdasdsa adsdsadasdasd
sfqfs@sfq:~/k8s$
```

## 使用 envFrom 获取 ConfigMap 中的值到环境变量

```bash
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-env
spec:
  restartPolicy: Never
  containers:
  - name: cm-test
    image: busybox
    command: ["/bin/sh", "-c", "env | grep app"]
    envFrom:
    - configMapRef:
        name: cm-appvars
EOF

sfqfs@sfq:~/k8s$ kubectl logs pod-cm-env
apploglevel=info xxdsdasdsa adsdsadasdasd
appdatadir=/var/data
```

## 使用 volumeMount 将 ConfigMap 挂载到容器内部的文件

```bash
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-env
spec:
  restartPolicy: Never
  containers:
  - name: cm-test
    image: busybox
    command: ["/bin/sh", "-c", "ls /files; cat /files/*"]
    volumeMounts:
    - name: files
      mountPath: /files
  volumes:
  # 定义 volume 名称
  - name: files
    configMap:
      # 指定使用的 configmap 对象
      name: cm-appvars
      # 将 key 对象的值以 path 指定的文件名进行挂载
      # 如果不指定 items, 则默认将所有 key 对象的值以文件名的形式挂载
      items:
      - key: apploglevel
        path: apploglevel.txt
      - key: appdatadir
        path: appdatadir.txt
EOF

sfqfs@sfq:~/k8s$ kubectl logs pod-cm-env
appdatadir.txt
apploglevel.txt
/var/datainfo xxdsdasdsa adsdsadasdasd
```


# 使用 subPath
```bash
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-env
spec:
  restartPolicy: Never
  containers:
  - name: cm-test
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: files
      mountPath: /etc/apploglevel.txt
      subPath: apploglevel.txt
  volumes:
  # 定义 volume 名称
  - name: files
    configMap:
      # 指定使用的 configmap 对象
      name: cm-appvars
      items:
      - key: apploglevel
        path: apploglevel.txt
EOF
```


## 热更新 ConfigMap

```bash
kubectl  create cm cm-nginx --from-file=nginx.conf --dry-run -oyaml | kubectl replace -f  -
```