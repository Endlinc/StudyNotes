#### 在Kubernetes用StatefulSet部署ZooKeeper



##### 需要准备的

1. 需要一个至少包含4个节点的集群
1. 准备3个20G的卷

##### 目标

- 如何使用 StatefulSet 部署一个 ZooKeeper ensemble。
- 如何一致性配置 ensemble。
- 如何在 ensemble 中 分布 ZooKeeper 服务器的部署。

##### Zookeeper介绍

[Apache ZooKeeper](https://zookeeper.apache.org/doc/current/) 是一个分布式的开源协调服务，用于分布式系统。详细介绍：https://en.wikipedia.org/wiki/Apache_ZooKeeper

##### 准备工作

首先我们需要创建3个StorageClass为any的pv，这为我们的将要部署的ZooKeeper提供持久的数据储存空间。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-pv-0
  labels:
    type: local
spec:
  storageClassName: any
  capacity:
    storage: 11Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-pv-1
  labels:
    type: local
spec:
  storageClassName: any
  capacity:
    storage: 11Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-pv-2
  labels:
    type: local
spec:
  storageClassName: any
  capacity:
    storage: 11Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

通过执行kubectl apply命令，我们可以创建出上面声明的3个pv。

```shell
kubectl apply -f zookeeper-pv.yaml
```

创建完成后，可以通过kubectl查询pv是否创建成功。

```shell
kubectl get pv -A
```

（worker node没有集群外网落权限情况下）我们需要把ZooKeeper镜像手动挂载到目标集群下所有的节点。下面的命令会非常好用：

```shell
docker pull kubernetes-zookeeper
docker save -o zookeeper.tar kubernets-zookeeper:1.0-3.4
scp zookeeper.tar {user}@{ip}:{port}
docker load < zookeeper.tar
docker tag {old_image_name}:{version} {new_image_name}:{version}
```

##### 创建一个ZooKeeper Ensemble

下面的配置文件包含创建一个无头服务、一个Service、一个PodDisruptionBudget和一个StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  namespace: mae
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  namespace: mae
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
  namespace: mae
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
  namespace: mae
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: IfNotPresent
        image: "k8s.gcr.io/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 0
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "any"
      resources:
        requests:
          storage: 10Gi
```

 打开终端，使用kubectl apply创建出我们在上面配置文件中声明的清单。

```shell
kubectl apply -f zookeeper.yaml
```

将创建出一个叫做zk-hs的无头服务、zk-cs服务、zk-pdb的PodDisruptionBudget和zk的StatefulSet。你讲看到下面回显：

```shell
service/zk-hs created
service/zk-cs created
poddisruptionbudget.policy/zk-pdb created
statefulset.apps/zk created
```

可以通过kubectl get查看StatefulSet创建的Pods:

```shell
kubectl get pods -w -l app=zk -n fst-manage
```

你可能会发现某些zookeeper的节点状态会是pending或者是其他异常状态，你可以通过kubectl describe命令查询这个容器的状态：

```shell
kubectl describe pods zk-0 -n fst-manage
```

通过返回的信息你可以明确Pod执行失败的原因，以下有几种常见情况：

1. pvc连接pv失败。你需要确认StatefulSet中的VolumeClaimTemplate所声明的storageclass、需要的空间大小与你创建的pv是否一致，否则会连接失败。
2. image pull error。可能你的worker node没有外网权限，或者没有找到你在创建服务申明中填入的镜像地址或镜像名。可能的解决方案包括手动把你所需要的镜像挂载到集群的节点上，确认镜像名字与镜像版本与你声明的一致；或者确认你的镜像是否可以通过集群节点拉取。

到这里，你声明的3个zookeeper都已经部署到环境上，同时无头服务和服务都在正常工作。我们通过StatefulSet部署了Zookeeper服务到环境上。



###### Reference：

https://kubernetes.io/zh/docs/tutorials/stateful-application/zookeeper/ 运行 ZooKeeper，一个分布式协调系统-kubernetes博客

https://www.jianshu.com/p/729d755ace65 docker镜像下载到本地，并在其他机器恢复

https://blog.csdn.net/Alavn_/article/details/103799826 Docker load 之后镜像名字为none问题解决

https://feisky.gitbooks.io/kubernetes/content/troubleshooting/network.html 网络异常排错

https://luckymrwang.github.io/2021/03/27/%E7%90%86%E8%A7%A3%E6%9C%89%E7%8A%B6%E6%80%81%E5%BA%94%E7%94%A8%E6%8E%A7%E5%88%B6%E5%99%A8StatefulSet/ 理解有状态应用控制器StatefulSet
