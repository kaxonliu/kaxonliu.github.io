# 手动部署  nfs-subdir-external-provisioner

有的时候使用 helm 遇到网络问题无法部署 nfs-subdir-external-provisioner，此时可以手动安装部署。

## 下载部署文件

在 K8S 控制节点上，下载最新版 `nfs-subdir-external-provisioner-4.0.18` Releases 文件，并解压。

~~~bash
# 下载
wget https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/archive/refs/tags/nfs-subdir-external-provisioner-4.0.18.zip

# 解压
unzip nfs-subdir-external-provisioner-4.0.18.zip

# 切目录
cd nfs-subdir-external-provisioner-4.0.18.nfs-subdir-external-provisioner-nfs-subdir-external-provisioner-4.0.18/deploy
~~~



## 创建名称空间

统一管理

~~~bash
kubectl create ns nfs-system
~~~

统一替换名称空间

~~~bash
sed -i "s/namespace:.*/namespace: nfs-system/g" ./rbac.yaml ./deployment.yaml
~~~



## 修改 deployment.yaml 配置

~~~yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.10.85		# 修改为 nfs server ip
            - name: NFS_PATH
              value: /data/nfs			# 修改为 nfs server 共享的路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.10.85		# 修改为 nfs server ip
            path: /data/nfs			    # 修改为 nfs server 共享的路径
~~~



## 修改  class.yaml

定义 Kubernetes Storage Class 的配置文件 `class.yaml`，重点修改储卷删除后的默认策略

- 该值为 false 时，存储卷删除时，在 NFS 上直接删除对应的数据目录
- 该值为 true 时，存储卷删除时，在 NFS 上以 `archived-<volume.Name>` 的命名规则，归档保留原有的数据目录

~~~yaml
# class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
# or choose another name, must match deployment's env PROVISIONER_NAME'
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"	# 存储卷删除时 归档保留原有的数据目录
~~~



## 部署

~~~bash
# 部署 
kubectl apply -k .
~~~



