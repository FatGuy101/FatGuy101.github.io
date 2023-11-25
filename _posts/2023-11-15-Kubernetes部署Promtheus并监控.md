---
layout: post
title: 使用Prometheus监控Kubernetes集群
subtitle: 多master的Kubernetes集群
categories: markdown
tags: [Kubernetes]
---



### 集群搭建

参考之前写的Kubernetes集群搭建

https://fattt.org.edu.kg/markdown/2023/11/05/K8s%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA.html#h-kubernetes%E5%A4%9Amaster%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA

这里使用多master的k8s集群

| hosts            | ip            |
| ---------------- | ------------- |
| server10(master) | 192.168.25.10 |
| server11(master) | 192.168.25.11 |
| server12(master) | 192.168.25.12 |
| server13         | 192.168.25.13 |

### Kubernetes部署Promtheus

创建目录

~~~shell
mkdir -p /root/prom/{alertmanager,node-exporter,config,grafana} && cd /root/prom/
~~~

#### namespace

创建namespace

~~~shell
cat > prometheus-ns.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  name: prom
EOF
kubectl apply -f prometheus-ns.yaml 
~~~

#### RBAC

配置promtheus RBAC账号权限

~~~shell
cat > prometheus-rbac.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: prom

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
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
  namespace: prom
EOF
kubectl apply -f prometheus-rbac.yaml 
~~~

#### congfigMap

创建prometheus配置文件，以congfigMap方式使用(这里我还监控了ceph，不需要的可删除)

~~~shell
cat > prometheus-config-configMap.yaml << EOF
apiVersion: v1
data:
  prometheus.yaml: |
    global:
      scrape_interval:     15s
      evaluation_interval: 15s
      external_labels:
        cluster: k8s-cluster

    alerting:
      alertmanagers:
      - static_configs:
        - targets:
           - alertmanager.prom.svc:9093

    rule_files:
      - /etc/prometheus-rules/alerts.yaml

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['localhost:9090']

      - job_name: 'kubernetes-node-exporter'
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - source_labels: [__address__]
          regex: '(.*):10250'
          replacement: '${1}:9100'
          target_label: __address__
          action: replace
        - source_labels: [__meta_kubernetes_node_name]
          action: replace
          target_label: node
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - source_labels: [__meta_kubernetes_node_address_InternalIP]
          action: replace
          target_label: ip
          
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-cadvisor'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

      - job_name: 'kubernetes-service-endpoints'
        kubernetes_sd_configs:
        - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name

      - job_name: 'kubernetes-services'
        kubernetes_sd_configs:
        - role: service
        metrics_path: /probe
        params:
          module: [http_2xx]
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
          action: keep
          regex: true
        - source_labels: [__address__]
          target_label: __param_target
        - target_label: __address__
          replacement: blackbox-exporter.example.com:9115
        - source_labels: [__param_target]
          target_label: instance
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          target_label: kubernetes_name

      - job_name: 'kubernetes-ingresses'
        kubernetes_sd_configs:
        - role: ingress
        relabel_configs:
        - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_probe]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
          regex: (.+);(.+);(.+)
          replacement: ${1}://${2}${3}
          target_label: __param_target
        - target_label: __address__
          replacement: blackbox-exporter.example.com:9115
        - source_labels: [__param_target]
          target_label: instance
        - action: labelmap
          regex: __meta_kubernetes_ingress_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_ingress_name]
          target_label: kubernetes_name

      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name

      - job_name: 'ceph'
        static_configs:
        - targets: ['192.168.25.150:9283']

kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: prom
EOF
kubectl apply -f prometheus-config-configMap.yaml
~~~

~~~shell
cat > prometheus-alerts-configMap.yaml << EOF
apiVersion: v1
data:
  alerts.yaml: |
    groups:
     - name: prometheus alert
       rules:
        - alert: "InstanceDown"
          expr: up == 0
          for: 30s
          labels:
            severity: critical
          annotations:
            instance: "{{ $labels.instance }}"
            description: "{{ $labels.job }} 已停止运行超过 30s！"
            
     - name: node-up
       rules:
       - alert: "node-up"
         expr: up{node="server13"} == 0
         for: 15s
         labels:
           severity: critical
         annotations:
           instance: "{{ $labels.instance }}"
           description: "{{ $labels.instance }} 已停止运行超过 15s！"

kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: prom
EOF
kubectl apply -f prometheus-alerts-configMap.yaml
~~~

#### deployment

部署prometheus

~~~shell
cat > prometheus-deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: prom
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus
        name: prometheus
        imagePullPolicy: IfNotPresent
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yaml"
        # 暂时使用自身Pod进行存储 <应该修改为 pv 存储>
        - "--storage.tsdb.path=/prometheus"
        # 数据保留30天
        - "--storage.tsdb.retention=30d"
        # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-admin-api"
        # 支持热更新，直接执行 curl -x POST  ip:9090/-/reload立即生效
        - "--web.enable-lifecycle"

        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/etc/prometheus"
          name: config-volume
        - mountPath: "/etc/prometheus-rules"
          name: alerts-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 1500m
            memory: 1500Mi
      securityContext:
        runAsUser: 0

      volumes:
      - configMap:
          name: prometheus-config
        name: config-volume

      - name: alerts-volume
        configMap:
          name: prometheus-alerts
EOF
kubectl apply -f prometheus-deploy.yaml
~~~

#### service

创建 prometheus service 暴露服务 <如果无需外界访问则无需暴露>

~~~shell
cat > prometheus-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: prom
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
    - name: prometheus-web
      nodePort: 30090
      port: 9090
      targetPort: http
EOF
kubectl apply -f prometheus-svc.yaml
~~~



### Kubernetes部署alertManager

~~~shell
cd /root/prom/alertmanager
~~~

#### configMap

配置configMap配置清单

~~~shell
cat > alertmanager-config.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: prom
data:
  alerts.yaml: |-
    global:
      # 在没有报警的情况下声明为已解决的时间
      resolve_timeout: 5m
      # 配置邮件发送信息
      smtp_smarthost: 'smtp.qq.com:465'
      # 填写自己的qq邮箱
      smtp_from: 'xxxxxx@qq.com'
      # 填写自己的qq邮箱
      smtp_auth_username: 'xxxxxx@qq.com'
      # 填写自己的邮箱授权码，我这里是假的授权码
      smtp_auth_password: 'syiolshahdusgaja'
      smtp_hello: 'qq.com'
      smtp_require_tls: true
    # 所有报警信息进入后的根路由，用来设置报警的分发策略
    route:
      # 这里的标签列表是接收到报警信息后的重新分组标签，例如，接收到的报警信息里面有许多具有 cluster=A 和 alertname=LatncyHigh 这样的标签的报警信息将会批量被聚合到一个分组里面
      group_by: ['alertname', 'cluster']
      # 当一个新的报警分组被创建后，需要等待至少group_wait时间来初始化通知，这种方式可以确保您能有足够的时间为同一分组来获取多个警报，然后一起触发这个报警信息。
      group_wait: 30s
      # 当第一个报警发送后，等待'group_interval'时间来发送新的一组报警信息。
      group_interval: 5m
      # 如果一个报警信息已经发送成功了，等待'repeat_interval'时间来重新发送他们
      repeat_interval: 5m
      # 默认的receiver：如果一个报警没有被一个route匹配，则发送给默认的接收器
      receiver: default
      # 上面所有的属性都由所有子路由继承，并且可以在每个子路由上进行覆盖。
      routes:
      - receiver: email
        group_wait: 10s
        match:
          team: node
    receivers:
    - name: 'default'
      email_configs:
      # 填写自己的qq邮箱
      - to: 'xxxxxxx@qq.com'
        send_resolved: true
    - name: 'email'
      email_configs:
      # 填写别人的qq邮箱
      - to: 'xxxxxx@qq.com'
        send_resolved: true
EOF
kubectl apply -f alertmanager-config.yaml
~~~

#### delpoyment

配置alertmanager Pod

~~~shell
cat > alertmanager-deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: prom
spec:
  selector:
    matchLabels:
      app: alertmanager
  replicas: 2
  template:
    metadata:
      labels:
         app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - name: alert-config
          mountPath: /etc/prometheus-rules
        ports:
        - name: http
          containerPort: 9093
      volumes:
      - name: alert-config
        configMap:
          name: alertmanager-config
EOF
kubectl apply -f alertmanager-deploy.yaml
~~~

#### service

暴露alertmanager service

~~~shell
cat > alertmanager-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-svc
  namespace: prom
  annotations:
    prometheus.io/scrape: "true"
spec:
  selector:
    app: alertmanager
  ports:
  - name: http
    port: 9093
EOF
kubectl apply -f alertmanager-svc.yaml
~~~



### Kubernetes部署node-exporter

~~~shell
cd /root/prom/node-exporter
~~~

#### node-exporter

部署安装node-exporter

~~~shell
cat > node-exporter.yaml << EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: prom
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
      name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15
        securityContext:
          privileged: true
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
EOF
kubectl apply -f node-exporter.yaml
~~~



### Kubernetes部署grafana

~~~shell
cd /root/prom/grafana
mkdir -p /root/prom/grafana/data
~~~

#### 部署 kafana

 PS: 这里配置securityContext的原因是从v5.1.0开始，grafana的运行用户ID（userid）和组ID（groupid）已经改成了472。而另外两个env是用来配置Grafana的管理员用户和密码。

~~~shell
cat > grafana-deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: prom
  labels:
    app: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  replicas: 1
  revisionHistoryLimit: 10
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:9.5.10
        imagePullPolicy: IfNotPresent
        ports:
        - name: grafana
          containerPort: 3000
        env:
        #指定 grafana的初始 admin用户和密码
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin
        readinessProbe:
          failureThreshold: 10
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 30
        livenessProbe:
          failureThreshold: 10
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - name: storage
          mountPath: /var/lib/grafana
          subPath: grafana
      securityContext:
        fsGroup: 472
        runAsUser: 472
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana
EOF
kubectl apply -f grafana-deploy.yaml
~~~

#### pvpv

动态制备卷

~~~shell
cat > grafana-pvpv.yaml << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # 这里指定你的NFS服务器地址
    server: xxx.xx.xxx.xx
    path: /root/prom/grafana/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
  namespace: kube-ops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
kubectl apply -f grafana-pvpv.yaml
~~~

#### service

暴露kafana的服务(因为没指定端口，所以它自己生成了端口号)

~~~
cat > grafana-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: prom
  labels:
    app: grafana
spec:
  selector:
    app: grafana
  type: NodePort
  ports:
  - name: grafana
    port: 3000
EOF
kubectl apply -f grafana-svc.yaml
~~~



### 热部署

查看service

~~~shell
kubectl get -n prom svc
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
alertmanager-svc   ClusterIP   10.1.210.101   <none>        9093/TCP         19h
grafana            NodePort    10.1.154.58    <none>        3000:31431/TCP   19h
prometheus         NodePort    10.1.106.7     <none>        9090:30090/TCP   20h
~~~

更新Prometheus数据

~~~shell
curl -X POST http://10.1.106.7:9090/-/reload
~~~



### 浏览器访问

Prometheus

http://192.168.25.10:30090

Grafana

http://192.168.25.10:31431





alertManager警报生效

![](https://picss.sunbangyan.cn/2023/11/25/00451ba6d08d58dc7809afaff21ab60e.jpeg)

警告规则生效

![](https://picst.sunbangyan.cn/2023/11/25/4386a4b9c082678dbf341c9215fae1c7.jpeg)

自动发现服务生效

![](https://picdm.sunbangyan.cn/2023/11/25/35dd824d017e556d7e28e9868e457a02.jpeg)

targets生效

![](https://picdm.sunbangyan.cn/2023/11/25/7174d0a5dc3b005750efe9f224a11ec5.jpeg)

grafana使用配置好的账号密码登录

账号：admin

密码：admin

导入Prometheus数据时要注意，Prometheus和grafana都在k8s集群内部

填写url就指定service名字和集群内部的端口

![](https://picdm.sunbangyan.cn/2023/11/25/457f2230c5b532123a2794ab906d53fd.jpeg)

监控node-exporter

![](https://picst.sunbangyan.cn/2023/11/25/6a63e820cb6e6243b0f0d14174551d66.jpeg)

监控模板

![](https://picss.sunbangyan.cn/2023/11/25/a1d7e7ad82a4b8d67a39208f10c976cd.jpeg)
