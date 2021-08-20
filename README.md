**쿠버네티스 클러스터의 메트릭들을 Prometheus로 수집하고 Grafana web UI를 통해 시각화 시키는 작업**   
**GRU 호롤리님의 블로그를 보고 GCP 클라우드 환경에서 실습을 진행했습니다**

# 1. Prometheus

- **namespace 생성**

```bash
$ kubectl create ns monitoring
```

- **ClusterRole, ClusterRoleBinding 생성**

```yaml
# prometheus-cluster-role.yaml

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: monitoring
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```

- **Configmap 생성**

```yaml
# prometheus-config-map.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: container memory alert
      rules:
      - alert: container memory usage rate is very high( > 55%)
        expr: sum(container_memory_working_set_bytes{pod!="", name=""})/ sum (kube_node_status_allocatable_memory_bytes) * 100 > 55
        for: 1m
        labels:
          severity: fatal
        annotations:
          summary: High Memory Usage on 
          identifier: ""
          description: " Memory Usage: "
    - name: container CPU alert
      rules:
      - alert: container CPU usage rate is very high( > 10%)
        expr: sum (rate (container_cpu_usage_seconds_total{pod!=""}[1m])) / sum (machine_cpu_cores) * 100 > 10
        for: 1m
        labels:
          severity: fatal
        annotations:
          summary: High Cpu Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
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

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

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

      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

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
```

- **deployment 생성**

```yaml
# prometheus-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf

        - name: prometheus-storage-volume
          emptyDir: {}
```

- **node exporter 데몬셋 생성**

```yaml
# prometheus-node-exporter.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - image: prom/node-exporter
        name: node-exporter
        ports:
        - containerPort: 9100
          protocol: TCP
          name: http
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 9100
    nodePort: 31672
    protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
```

- **Service 생성**

**클러스터 외부에서 접속이 가능하도록 Service type을 NodePort에서 LoadBalancer로 변경**

```yaml
# prometheus-svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector:
    app: prometheus-server
  #type: NodePort
  #ports:
  #  - port: 8080
  #    targetPort: 9090
  #    nodePort: 30003
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: 9090
  externalIPs:
    - xx.xx.xxx.xx
    - xx.xx.xxx.xx
```

- **배포**

```bash
$ kubectl apply -f prometheus-cluster-role.yaml
$ kubectl apply -f prometheus-config-map.yaml
$ kubectl apply -f prometheus-deployment.yaml
$ kubectl apply -f prometheus-node-exporter.yaml
$ kubectl apply -f prometheus-svc.yaml
```

- **pod 확인**

**node-exporter가 노드 수 만큼 생성되었는지 확인**

```bash
$ kubectl get pod -n monitoring

NAME                                     READY   STATUS    RESTARTS   AGE
node-exporter-99w2v                      1/1     Running   0          18s
node-exporter-f9q7f                      1/1     Running   0          18s
prometheus-deployment-7bcb5ff899-h4rb7   1/1     Running   0          65s
```

- **service 확인**

**Prometheus 브라우저 접속확인**

```bash
$ kubectl get svc -o wide -n monitoring

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP		PORT(S)          AGE     SELECTOR
prometheus-service   LoadBalancer   10.99.72.195   xx.xx.xxx.xxx   	8080:31213/TCP   7d      app=prometheus-server
```

- **Prometheus 상단 메뉴 확인**

```
상단 메뉴의 Status > Targets 를 들어가서 kube-state-metrics 가 떠있는 확인한다. 
kube-state-metrics는 쿠버네티스 클러스터 내 오브젝트(예를들면 Pod)에 대한 지표정보를 
생성하는 서비스로 Pod 상태정보를 모니터링하기 위해서는 kube-state-metrics가 떠 있어야 한다.
```

- **Kube State Metrics 배포**

```yaml
# kube-state-cluster-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources: ["configmaps", "secrets", "nodes", "pods", "services", "resourcequotas", "replicationcontrollers", "limitranges", "persistentvolumeclaims", "persistentvolumes", "namespaces", "endpoints"]
  verbs: ["list","watch"]
- apiGroups:
  - extensions
  resources: ["daemonsets", "deployments", "replicasets", "ingresses"]
  verbs: ["list", "watch"]
- apiGroups:
  - apps
  resources: ["statefulsets", "daemonsets", "deployments", "replicasets"]
  verbs: ["list", "watch"]
- apiGroups:
  - batch
  resources: ["cronjobs", "jobs"]
  verbs: ["list", "watch"]
- apiGroups:
  - autoscaling
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "watch"]
- apiGroups:
  - authentication.k8s.io
  resources: ["tokenreviews"]
  verbs: ["create"]
- apiGroups:
  - authorization.k8s.io
  resources: ["subjectaccessreviews"]
  verbs: ["create"]
- apiGroups:
  - policy
  resources: ["poddisruptionbudgets"]
  verbs: ["list", "watch"]
- apiGroups:
  - certificates.k8s.io
  resources: ["certificatesigningrequests"]
  verbs: ["list", "watch"]
- apiGroups:
  - storage.k8s.io
  resources: ["storageclasses", "volumeattachments"]
  verbs: ["list", "watch"]
- apiGroups:
  - admissionregistration.k8s.io
  resources: ["mutatingwebhookconfigurations", "validatingwebhookconfigurations"]
  verbs: ["list", "watch"]
- apiGroups:
  - networking.k8s.io
  resources: ["networkpolicies"]
  verbs: ["list", "watch"]
```

```yaml
# kube-state-svcaccount.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: kube-system
```

```yaml
# kube-state-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      containers:
      - image: quay.io/coreos/kube-state-metrics:v1.8.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: kube-state-metrics
```

```yaml
# kube-state-svc.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app: kube-state-metrics
```

```bash
$ kubectl apply -f kube-state-cluster-role.yaml
$ kubectl apply -f kube-state-deployment.yaml
$ kubectl apply -f kube-state-svcaccount.yaml
$ kubectl apply -f kube-state-svc.yaml
```

```bash
$ kubectl get pod -n kube-system

NAME                                       READY   STATUS    RESTARTS   AGE
...
kube-state-metrics-59bd4d9d-nbfrq          1/1     Running   0          50s
```

# 2. Grafana 연동

- **Grafana 설치**

**클러스터 외부에서 접속이 가능하도록 Service type을 NodePort에서 LoadBalancer로 변경**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - name: grafana
          containerPort: 3000
        env:
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          value: /
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '3000'
spec:
  selector:
    app: grafana
  #type: NodePort
  #ports:
  #  - port: 3000
  #    targetPort: 3000
  #    nodePort: 30004
	type: LoadBalancer
  ports:
    - port: 3000
      targetPort: 3000
  externalIPs:
    - xx.xx.xxx.xx
    - xx.xx.xxx.xx
```

- **Grafana pod 확인**

```bash
$ kubectl apply -f grafana.yaml

$ kubectl get pod -n monitoring
NAME                                     READY   STATUS    RESTARTS   AGE
grafana-799c99855d-kxhkm                 1/1     Running   0          16s
node-exporter-99w2v                      1/1     Running   0          66m
node-exporter-f9q7f                      1/1     Running   0          66m
prometheus-deployment-7bcb5ff899-h4rb7   1/1     Running   0          67m
```

- **Grafana Service 확인**

Grafana **브라우저 접속확인**

```bash
$ kubectl get svc -n monitoring
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP		PORT(S)          AGE
grafana              LoadBalancer   10.109.82.53   xx.xx.xxx.xxx   	3000:30837/TCP   6d23h
prometheus-service   LoadBalancer   10.99.72.195   xx.xx.xxx.xxx   	8080:31213/TCP   7d
```

### 참고
[Kubernetes Monitoring - Prometheus 실습](https://gruuuuu.github.io/cloud/monitoring-02/#)  
[프로메테우스와 그라파나로 개발 서버 모니터링하기](https://essem-dev.medium.com/%ED%94%84%EB%A1%9C%EB%A9%94%ED%85%8C%EC%9A%B0%EC%8A%A4%EC%99%80-%EA%B7%B8%EB%9D%BC%ED%8C%8C%EB%82%98%EB%A1%9C-%EA%B0%9C%EB%B0%9C-%EC%84%9C%EB%B2%84-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81%ED%95%98%EA%B8%B0-8942aea724b3)    
[Prometheus 모니터링 실무 적용기 2탄](https://medium.com/coinone/prometheus-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81-%EC%8B%A4%EB%AC%B4-%EC%A0%81%EC%9A%A9%EA%B8%B0-2%ED%83%84-8a210a7f2ff)  

[SISIPAPA 노트연결](https://sisipapa.github.io/blog/2021/01/01/k8s-prometheus-grafana/)
