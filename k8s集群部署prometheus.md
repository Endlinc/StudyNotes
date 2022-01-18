### 在Kubernetes集群部署Prometheus

##### Reference

[^1]: https://kubernetes.io/zh/docs/tasks/administer-cluster/certificates/
[^2]: https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/
[^3]: https://kubernetes.io/zh/docs/tasks/administer-cluster/dns-debugging-resolution/
[^4]: https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/
[^5]: https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/
[^6]: https://prometheus.io/docs/practices/pushing/



#### 事前准备

1. 确认apiserver已经正确配置了`--authorization-mode`, `--client-ca-file=`, `--enable-aggregator-routing`, `--kubelet-client-certificate`, `--kubelet-client-key`, `--tls-cert-file`, `--tls-private-key-file`. 
   p.s. FusionStage的apiserver配置文件路径为`/opt/kubernetes/manifests/kube-apiserver.manifest`, 原生k8s配置文件路径在`/etc/kubernetes/manifests/kube-apiserver.yaml`
2. apiserver能够正确转发请求给aggregator
3. 做一份证书，用于Prometheus服务网络连接的鉴权。参考Reference[^1]



#### 知识储备

1. 了解角色鉴权，ServiceAccount和RBAC的关系。参考Reference[^2]
2. 域名，如诊断无法访问域名的问题，了解通过`/etc/resolv.conf`设置`kubernetes.default.svc.cluster.local`的ip。参考Reference[^3]
3. 了解daemonset的作用，node-exporter为何能获取**所有**节点的性能指标。参考Reference[^4]



#### 开始安装

因为我们不想要所有人都能够访问集群的性能指标，所以我们设置RBAC鉴权模式中的ClusterRole，对性能指标的权限进行限制，保证只有监控服务可以访问这些资源。

首先我们要声明一个ServiceAccount和一个ClusterRole，分别声明一个服务用户和一个资源权限声明。为了关联这两个声明，我们使用ClusterRoleBinding。模板如下所示：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: fst-manage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  - nodes/metrics
  - nodes/proxy
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: fst-manage

```

接下来我们需要配置一个ConfigMap[^5]，用于设置Prometheus的各种配置项。这里我们可以设置我们需要监控的各种任务`job_name`，k8s服务发现的配置`kubernetes_sd_configs`，TLS请求所需要的设置`tls_config`，对象标记的配置`relabel_configs`等等。总而言之，这里是Prometheus服务的设置中心，这里我们配置如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: fst-manage
data:
  default: |
    global:
      scrape_interval:     10s
      evaluation_interval: 15s
    rule_files:
    scrape_configs:
    - job_name: 'kubernetes-nodes-kubelets'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc.cluster.local:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'kubernetes-nodes-cadvisor'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc.cluster.local:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    - job_name: 'kubernetes-service'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: service

    - job_name: 'kubernetes-endpoints'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

    - job_name: 'kubernetes-ingress'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: ingress

    - job_name: 'kubernetes-pods'
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: pod

    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']

    - job_name: 'pushgateway'
      static_configs:
      - targets: ["172.168.1.21:9091"]

```

接下来我们就开始部署Prometheus服务了，和普通的服务部署没有区别，我们首先声明Service配置容器的网络，再声明Deployment用来控制生成的pod。特别的我们要注意，需要在Deployment中加入我们之前申明的serviceAccountName，表示服务有权限可以访问资源，而且还要把configmap做的Prometheus配置，挂载到生成的pod中。Deployment生成后，会自动生成一个secret，用来存储Prometheus服务证书的位置和Token。模板如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: fst-manage
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
  - protocol: TCP
    port: 9090
    targetPort: 9090
    nodePort: 31110
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: fst-manage
  labels:
    app: prometheus
spec:
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: prometheus
  minReadySeconds: 0
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
        - name: prometheus
          image: prom/prometheus
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              memory: "512Mi"
              cpu: "200m"
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
            initialDelaySeconds: 60
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
            initialDelaySeconds: 60
            periodSeconds: 10
          ports:
            - containerPort: 80
          volumeMounts:
          - name: config
            mountPath: /etc/prometheus/prometheus.yml
            subPath: default
          - name: cert-volume
            mountPath: /var/paas/srv/kubernetes
      securityContext:
        runAsUser: 0
      volumes:
        - name: config
          configMap:
            name: prometheus-config
        - name: cert-volume
          hostPath:
            path: /var/paas/srv/kubernetes

```

执行`kubectl get pods -l app=prometheus -w`显示如下内容后表示Prometheus成功部署到了环境上。

```bash
[root@node0 ~]# kubectl get pods -n monitoring -l app=prometheus -w
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-7494bccb74-nlc56   1/1     Running   0          13d
```



#### 可选配件

##### Pushgateway

这是Prometheus服务对于性能指标获取模型pull model的一种拓展，用来收集生命周期极短工作的指标。相较于等待Prometheus来拉取指标数据，Pushgateway作为中间服务，让各个工作的指标推送到Pushgateway，然后储存、等待Prometheus的下一个周期拉取指标数据。

Pushgateway存在的初衷是为了监控生命周期很短的任务，对于是否选择使用Pushgateway，可以参考Prometheus的文档[^6]。

配置模板如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pushgateway
  namespace: fst-manage
  labels:
    app: pushgateway
spec:
  selector:
    app: pushgateway
  type: NodePort
  ports:
    - name: pushgateway
      port: 9091
      targetPort: 9091
      nodePort: 32000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: fst-manage
  name:  pushgateway
  labels:
    app:  pushgateway
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app:  pushgateway
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: "25%"
      maxUnavailable: "25%"
  template:
    metadata:
      name:  pushgateway
      labels:
        app:  pushgateway
    spec:
      containers:
        - name:  pushgateway
          image: prom/pushgateway:v0.7.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            initialDelaySeconds: 600
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
            httpGet:
              path: /
              port: 9091
          ports:
            - name: "app-port"
              containerPort: 9091
          resources:
            limits:
              memory: "1000Mi"
              cpu: 1
            requests:
              memory: "1000Mi"
              cpu: 1

```

##### node-exporter

Prometheus默认监控的集群和pod的性能指标，如果想要了解节点的性能状况，那么需要使用node-exporter。node-exporter用来提供服务器级别或者操作系统级别的指标，并提供可以配置的指标收集器。

模板配置如下：

```yaml
kind: DaemonSet
apiVersion: apps/v1
metadata:
  labels:
    app: node-exporter
  name: node-exporter
  namespace: fst-manage
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9100
              protocol: TCP
              name:     http
      hostNetwork: true  # 获得Node的物理指标信息
      hostPID: true  # 获得Node的物理指标信息
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: node-exporter
  name: node-exporter-service
  namespace: fst-manage
spec:
  ports:
    - name:     http
      port: 9100
      nodePort: 31672
      protocol: TCP
  type: NodePort
  selector:
    app: node-exporter

```



