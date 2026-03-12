# CKAD (Certified Kubernetes Application Developer) 완전 학습 가이드
> 시험 버전: Kubernetes v1.31+ | 시험 시간: 2시간 | 합격 기준: 66점 이상

---

## 📋 시험 도메인 및 가중치 (최신 커리큘럼)

| 도메인 | 가중치 |
|--------|--------|
| 1. Application Design and Build | 20% |
| 2. Application Deployment | 20% |
| 3. Application Observability and Maintenance | 15% |
| 4. Application Environment, Configuration and Security | 25% |
| 5. Services and Networking | 20% |

---

# 🟦 DOMAIN 1: Application Design and Build (20%)

## 1.1 컨테이너 이미지 빌드 및 관리

### 핵심 개념
- **Dockerfile** 기반 이미지 빌드
- **Multi-stage build**: 최종 이미지 경량화
- 이미지 태깅, 레지스트리 push/pull

```dockerfile
# Multi-stage build 예시
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

FROM alpine:3.18
WORKDIR /app
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

```bash
# 이미지 빌드 및 태깅
docker build -t myapp:v1.0 .
docker tag myapp:v1.0 registry.example.com/myapp:v1.0
docker push registry.example.com/myapp:v1.0
```

---

## 1.2 Pod 설계 패턴

### Init Containers
- 메인 컨테이너 실행 전 초기화 작업 수행
- 순차적으로 실행되며, 모두 성공해야 메인 컨테이너 시작

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-db
    image: busybox:1.35
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting; sleep 2; done']
  - name: init-service
    image: busybox:1.35
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: myapp:v1.0
    ports:
    - containerPort: 8080
```

### Sidecar Containers (Kubernetes 1.29+ 네이티브 지원)
- 메인 컨테이너와 함께 실행되는 보조 컨테이너
- 1.29+에서는 `initContainers`에 `restartPolicy: Always`로 정의

```yaml
# Kubernetes 1.29+ Sidecar 패턴
spec:
  initContainers:
  - name: log-collector
    image: fluentd:v1.16
    restartPolicy: Always   # ← 이것이 sidecar로 만드는 핵심
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  containers:
  - name: app
    image: myapp:v1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

### Ambassador / Adapter 패턴
```yaml
# Ambassador: 외부 연결을 대신 처리
spec:
  containers:
  - name: app
    image: myapp:v1.0
  - name: ambassador
    image: nginx:alpine
    # 외부 트래픽을 앱으로 프록시
```

---

## 1.3 Jobs & CronJobs

### Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  completions: 5        # 총 5번 성공해야 완료
  parallelism: 2        # 동시에 2개 Pod 실행
  backoffLimit: 4       # 실패 재시도 횟수
  activeDeadlineSeconds: 300  # 최대 실행 시간(초)
  template:
    spec:
      restartPolicy: Never   # Job에서는 Never 또는 OnFailure
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```

### CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"          # 매일 새벽 2시
  concurrencyPolicy: Forbid       # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: alpine:3.18
            command: ["/bin/sh", "-c", "echo backup done"]
```

---

## 1.4 Volumes

### emptyDir
```yaml
volumes:
- name: shared-data
  emptyDir:
    medium: Memory   # tmpfs 사용 (메모리 기반)
    sizeLimit: 100Mi
```

### hostPath
```yaml
volumes:
- name: host-vol
  hostPath:
    path: /data/myapp
    type: DirectoryOrCreate   # Directory | File | Socket | CharDevice | BlockDevice
```

### PersistentVolume / PersistentVolumeClaim
```yaml
# PV 생성
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-storage
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce         # RWO | ROX | RWX | RWOP
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle
  storageClassName: manual
  hostPath:
    path: /mnt/data

---
# PVC 생성
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

---

# 🟩 DOMAIN 2: Application Deployment (20%)

## 2.1 Deployment 전략

### Rolling Update (기본값)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 최대 추가 Pod 수 (또는 %)
      maxUnavailable: 0     # 최대 불가용 Pod 수 (또는 %)
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.25
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### Recreate 전략
```yaml
strategy:
  type: Recreate   # 모든 기존 Pod 종료 후 새 Pod 생성
```

### 주요 Deployment 명령어
```bash
# 이미지 업데이트
kubectl set image deployment/web-app web=nginx:1.26 --record

# 롤아웃 상태 확인
kubectl rollout status deployment/web-app

# 롤아웃 히스토리
kubectl rollout history deployment/web-app
kubectl rollout history deployment/web-app --revision=2

# 롤백
kubectl rollout undo deployment/web-app
kubectl rollout undo deployment/web-app --to-revision=1

# 스케일링
kubectl scale deployment/web-app --replicas=6

# 일시 중지 / 재개 (카나리 배포 활용)
kubectl rollout pause deployment/web-app
kubectl rollout resume deployment/web-app
```

---

## 2.2 Helm 기초

```bash
# Helm 차트 기본 명령어
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx

# 설치
helm install my-nginx bitnami/nginx --namespace default
helm install my-app ./mychart --values custom-values.yaml

# 업그레이드
helm upgrade my-nginx bitnami/nginx --set replicaCount=3

# 롤백
helm rollback my-nginx 1

# 목록 / 삭제
helm list -A
helm uninstall my-nginx
```

```yaml
# values.yaml 예시
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

---

## 2.3 HorizontalPodAutoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 200Mi
```

```bash
# HPA 생성 (명령형)
kubectl autoscale deployment web-app --cpu-percent=50 --min=2 --max=10
kubectl get hpa
```

---

# 🟨 DOMAIN 3: Application Observability and Maintenance (15%)

## 3.1 Liveness, Readiness, Startup Probes

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0
    
    # Startup Probe: 초기 기동 시간이 긴 앱
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10       # 최대 300초(30*10) 기다림
    
    # Liveness Probe: 컨테이너 재시작 여부 결정
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1
    
    # Readiness Probe: 트래픽 수신 여부 결정
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      
    # Exec 방식 probe
    # livenessProbe:
    #   exec:
    #     command: ["cat", "/tmp/healthy"]
    #   initialDelaySeconds: 5
    #   periodSeconds: 5
```

---

## 3.2 로깅 및 디버깅

```bash
# 로그 확인
kubectl logs <pod> -c <container>         # 특정 컨테이너
kubectl logs <pod> --previous            # 이전 컨테이너 로그
kubectl logs <pod> -f --tail=100         # 실시간 스트리밍

# Pod 디버깅
kubectl describe pod <pod>               # 이벤트 포함 상세 정보
kubectl exec -it <pod> -- /bin/sh        # 쉘 접속
kubectl exec <pod> -c <container> -- env # 환경변수 확인

# 임시 디버그 컨테이너 (Ephemeral Containers)
kubectl debug -it <pod> --image=busybox:1.35 --target=<container>

# 노드 디버깅
kubectl debug node/<node-name> -it --image=ubuntu

# 리소스 사용량
kubectl top pod
kubectl top node
kubectl top pod --containers
```

---

## 3.3 컨테이너 리소스 관리

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"       # 250 milliCPU = 0.25 CPU
      limits:
        memory: "128Mi"
        cpu: "500m"
# OOMKilled: 메모리 limit 초과 시
# Throttled: CPU limit 초과 시 (종료 안 됨)
```

---

# 🟥 DOMAIN 4: Application Environment, Configuration and Security (25%)

## 4.1 ConfigMap

```yaml
# ConfigMap 생성 (선언형)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  config.yaml: |
    database:
      host: mysql-service
      port: 3306
```

```bash
# ConfigMap 생성 (명령형)
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_PORT=8080

kubectl create configmap app-config \
  --from-file=config.yaml \
  --from-file=configs/   # 디렉토리 전체
```

```yaml
# ConfigMap 사용 - 환경변수
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: SPECIFIC_VAR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_PORT
    
    # ConfigMap 볼륨 마운트
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
      items:                    # 특정 키만 마운트
      - key: config.yaml
        path: app-config.yaml
```

---

## 4.2 Secret

```bash
# Secret 생성 (명령형)
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=p@ssw0rd

kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=         # base64 인코딩
  password: cEBzc3cwcmQ=
stringData:                  # 평문으로 작성 가능
  api-key: "my-api-key-here"
```

```yaml
# Secret 사용
spec:
  containers:
  - name: app
    envFrom:
    - secretRef:
        name: db-secret
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
  imagePullSecrets:
  - name: regcred
```

---

## 4.3 ServiceAccount & RBAC

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
automountServiceAccountToken: false   # 자동 마운트 비활성화

---
# Role (Namespace 범위)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# RBAC 확인
kubectl auth can-i get pods --as system:serviceaccount:default:app-sa
kubectl auth can-i create deployments --as=user1 -n kube-system
kubectl auth can-i '*' '*' --all-namespaces   # 슈퍼유저 확인
```

---

## 4.4 SecurityContext

```yaml
# Pod 레벨 SecurityContext
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp:v1.0
    # 컨테이너 레벨 (Pod 레벨보다 우선)
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE    # 포트 1024 이하 바인딩 허용
      privileged: false
    volumeMounts:
    - name: tmp
      mountPath: /tmp         # readOnlyRootFilesystem 시 임시 디렉토리
  
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## 4.5 ResourceQuota & LimitRange

```yaml
# ResourceQuota - 네임스페이스 전체 제한
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "5"
    persistentvolumeclaims: "5"
    secrets: "10"
    configmaps: "10"

---
# LimitRange - 개별 리소스 기본값/제한
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limit-range
  namespace: dev
spec:
  limits:
  - type: Container
    default:          # limits 기본값
      cpu: 500m
      memory: 256Mi
    defaultRequest:   # requests 기본값
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
  - type: PersistentVolumeClaim
    max:
      storage: 10Gi
    min:
      storage: 1Gi
```

---

## 4.6 NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-netpol
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: production
      podSelector:                  # AND 조건 (같은 항목 내)
        matchLabels:
          role: frontend
    - podSelector:                  # OR 조건 (다른 항목)
        matchLabels:
          role: monitoring
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:                            # DNS 허용 (필수!)
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

# 🟪 DOMAIN 5: Services and Networking (20%)

## 5.1 Services

### ClusterIP (기본값)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - name: http
    port: 80          # 서비스 포트
    targetPort: 8080  # 컨테이너 포트
    protocol: TCP
```

### NodePort
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # 30000-32767 범위
```

### LoadBalancer
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```

### Headless Service (StatefulSet용)
```yaml
spec:
  clusterIP: None    # Headless
  selector:
    app: my-statefulset
```

### ExternalName
```yaml
spec:
  type: ExternalName
  externalName: external-db.example.com   # CNAME 레코드
```

---

## 5.2 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

---

## 5.3 DNS & Service Discovery

```bash
# Pod 간 DNS 형식
<service-name>.<namespace>.svc.cluster.local
<pod-ip-dashes>.<namespace>.pod.cluster.local

# 예시
mysql-service.default.svc.cluster.local
web-app.production.svc.cluster.local

# 같은 네임스페이스에서는 서비스명만으로 접근 가능
curl http://mysql-service:3306
curl http://web-app.production   # 다른 네임스페이스
```

---

# 💡 시험 핵심 팁

## kubectl 필수 명령어 모음

```bash
# ── 리소스 생성/조회 ──────────────────────────────
kubectl run nginx --image=nginx:alpine --restart=Never  # Pod 즉시 생성
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --replicas=3
kubectl expose pod nginx --port=80 --type=ClusterIP --name=nginx-svc
kubectl expose deployment web --port=80 --target-port=8080

# ── 빠른 리소스 편집 ──────────────────────────────
kubectl edit deployment web-app
kubectl patch deployment web-app -p '{"spec":{"replicas":5}}'
kubectl set env deployment/web-app APP_ENV=production
kubectl set image deployment/web-app web=nginx:1.26
kubectl set resources deployment/web-app --limits=cpu=500m,memory=256Mi

# ── 조회 및 필터링 ─────────────────────────────────
kubectl get all -n kube-system
kubectl get pods -A --field-selector=status.phase=Running
kubectl get pods -l app=web-app,env=prod
kubectl get pods -o wide
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# ── 레이블 관리 ────────────────────────────────────
kubectl label pod nginx env=prod
kubectl label pod nginx env-          # 레이블 제거
kubectl annotate pod nginx owner=devteam

# ── 네임스페이스 ──────────────────────────────────
kubectl create ns dev-ns
kubectl config set-context --current --namespace=dev-ns  # 기본 ns 변경

# ── 컨텍스트 전환 ──────────────────────────────────
kubectl config get-contexts
kubectl config use-context dev-cluster
kubectl config current-context

# ── 단축 별칭 (시험장에서 설정) ───────────────────
alias k=kubectl
export do='--dry-run=client -o yaml'
export now='--force --grace-period 0'
```

---

---

# 📝 예상 출제 문제 및 정답/해설

---

## [문제 1] Pod 생성 (난이도: ★☆☆)

**문제:** `nginx` 이미지를 사용하는 `web-pod`라는 이름의 Pod를 `app-ns` 네임스페이스에 생성하세요. 컨테이너 포트는 80이고, 환경변수 `ENV=production`을 설정하세요.

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: app-ns
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    env:
    - name: ENV
      value: "production"
```
```bash
kubectl apply -f pod.yaml
# 또는 명령형
kubectl run web-pod -n app-ns --image=nginx --port=80 --env="ENV=production"
```

**해설:** `kubectl run`은 Pod를 즉시 생성하는 명령형 방식. `--dry-run=client -o yaml`로 YAML 생성 후 수정하는 방법도 효과적.

---

## [문제 2] Multi-container Pod (난이도: ★★☆)

**문제:** 다음 조건으로 Pod를 생성하세요:
- Pod명: `multi-pod`
- 컨테이너 1: `nginx:alpine` 이미지, `/usr/share/nginx/html` 에 볼륨 마운트
- 컨테이너 2: `busybox` 이미지, 10초마다 `/data/index.html`에 날짜를 기록하는 명령 실행
- 두 컨테이너는 `shared-data`라는 emptyDir 볼륨을 공유

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["while true; do date > /data/index.html; sleep 10; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

**해설:** emptyDir 볼륨은 Pod 생애주기와 함께하며 컨테이너 간 데이터 공유에 사용. `mountPath`가 컨테이너별로 달라도 같은 물리적 디렉토리를 공유.

---

## [문제 3] Init Container (난이도: ★★☆)

**문제:** Init Container를 활용하여 `/app/data/config.txt` 파일이 존재할 때까지 대기한 후, 메인 컨테이너(`nginx:alpine`)가 실행되도록 Pod를 구성하세요. Init Container는 `busybox` 이미지를 사용하고 파일 존재를 5초 간격으로 확인하세요.

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: wait-for-config
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["until [ -f /app/data/config.txt ]; do echo 'waiting for config'; sleep 5; done"]
    volumeMounts:
    - name: app-data
      mountPath: /app/data
  containers:
  - name: web
    image: nginx:alpine
    volumeMounts:
    - name: app-data
      mountPath: /app/data
  volumes:
  - name: app-data
    emptyDir: {}
```

**해설:** Init Container는 순서대로 실행되며 각각 성공해야 다음으로 진행. `restartPolicy`가 `Never`이면 실패 시 Pod 실패. `Always`면 재시작.

---

## [문제 4] Deployment + Rolling Update (난이도: ★★☆)

**문제:**
1. `web-deployment`라는 Deployment를 생성하세요 (replicas: 4, image: `nginx:1.24`)
2. `maxSurge: 1`, `maxUnavailable: 1`로 Rolling Update 정책 설정
3. 이미지를 `nginx:1.25`로 업데이트하고 진행 상태를 확인하세요
4. 업데이트 후 이전 버전으로 롤백하세요

**정답:**
```yaml
# 1. Deployment 생성
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
```
```bash
kubectl apply -f deployment.yaml

# 3. 이미지 업데이트
kubectl set image deployment/web-deployment nginx=nginx:1.25
kubectl rollout status deployment/web-deployment

# 4. 롤백
kubectl rollout undo deployment/web-deployment
kubectl rollout status deployment/web-deployment
```

**해설:** `rollout undo`는 이전 ReplicaSet으로 되돌림. `--to-revision=N`으로 특정 버전 지정 가능. `rollout history`로 리비전 확인.

---

## [문제 5] ConfigMap & Secret (난이도: ★★☆)

**문제:** 다음 조건으로 리소스를 생성하세요:
- ConfigMap `app-config`: key=`DB_HOST`, value=`mysql-svc`
- Secret `db-secret`: key=`DB_PASSWORD`, value=`mypassword`
- 이를 환경변수로 주입하는 Pod `config-pod` (이미지: `busybox`, 명령: `env | grep DB` 실행 후 종료)

**정답:**
```bash
kubectl create configmap app-config --from-literal=DB_HOST=mysql-svc
kubectl create secret generic db-secret --from-literal=DB_PASSWORD=mypassword
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c", "env | grep DB"]
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
```

**해설:** ConfigMap은 평문, Secret은 base64 인코딩으로 저장. `envFrom`으로 전체 주입, `env[].valueFrom`으로 개별 주입. Secret은 메모리(tmpfs)에 마운트하는 것이 보안상 안전.

---

## [문제 6] SecurityContext (난이도: ★★★)

**문제:** 다음 보안 요구사항을 만족하는 Pod를 생성하세요:
- Pod명: `secure-pod`, 이미지: `nginx:alpine`
- UID 1000으로 실행
- Root로 실행 금지
- 권한 상승(privilege escalation) 금지
- 읽기 전용 루트 파일시스템
- `/tmp` 는 쓰기 가능하도록 emptyDir 마운트

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
  containers:
  - name: nginx
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp-vol
      mountPath: /tmp
  volumes:
  - name: tmp-vol
    emptyDir: {}
```

**해설:** `runAsNonRoot: true`는 UID 0으로 실행 시도하면 컨테이너 시작 실패. `readOnlyRootFilesystem`은 보안 강화에 필수적이지만 `/tmp`, `/var/run` 등 쓰기 필요 경로는 emptyDir로 해결.

---

## [문제 7] RBAC (난이도: ★★★)

**문제:** `dev` 네임스페이스에서 다음 권한을 가진 환경을 구성하세요:
- ServiceAccount `dev-sa` 생성
- `pods`와 `pods/log`에 대해 `get`, `list`, `watch` 가능
- `deployments`에 대해 `get`, `list`, `create`, `update` 가능
- 위 권한을 `dev-sa`에 바인딩

**정답:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-sa
  namespace: dev

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-sa
  namespace: dev
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```
```bash
# 검증
kubectl auth can-i list pods --as=system:serviceaccount:dev:dev-sa -n dev
kubectl auth can-i delete pods --as=system:serviceaccount:dev:dev-sa -n dev
```

**해설:** Pod는 core API group(`""`), Deployment는 apps API group(`"apps"`). 리소스의 subresource(`pods/log`)는 별도 명시 필요. ClusterRole/ClusterRoleBinding은 클러스터 전체 범위.

---

## [문제 8] NetworkPolicy (난이도: ★★★)

**문제:** `production` 네임스페이스의 `app: backend` Pod에 대해:
- `app: frontend` 레이블의 Pod에서만 TCP 8080 포트 인그레스 허용
- 외부로의 MySQL(TCP 3306) 이그레스만 허용
- 그 외 모든 트래픽 차단

**정답:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:          # DNS 허용
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

**해설:** `policyTypes`에 Ingress/Egress 모두 명시하면 명시되지 않은 모든 트래픽 차단. DNS 쿼리(UDP/TCP 53)를 별도 허용하지 않으면 서비스 디스커버리 실패. `namespaceSelector`와 `podSelector` 같은 `-` 항목 내에 있으면 AND 조건.

---

## [문제 9] PersistentVolume & PVC (난이도: ★★☆)

**문제:** 다음을 구성하세요:
- `pv-log`: 1Gi, hostPath `/var/log/app`, ReadWriteMany, Retain 정책
- `pvc-log`: 500Mi, ReadWriteMany 요청
- `log-pod`: PVC를 `/var/log`에 마운트, `nginx` 이미지 사용

**정답:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /var/log/app

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-log
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Mi

---
apiVersion: v1
kind: Pod
metadata:
  name: log-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: log-storage
      mountPath: /var/log
  volumes:
  - name: log-storage
    persistentVolumeClaim:
      claimName: pvc-log
```

**해설:** PVC는 요청 크기 이상의 PV에 바인딩. accessMode가 일치해야 함. `storageClassName`을 명시하지 않으면 기본 StorageClass 사용. Retain 정책은 PVC 삭제 후에도 PV의 데이터 보존.

---

## [문제 10] CronJob (난이도: ★★☆)

**문제:** 매 5분마다 `date`를 출력하는 CronJob을 생성하세요:
- 이름: `date-printer`, 이미지: `busybox`
- 동시 실행 금지 (Forbid)
- 성공 이력 3개, 실패 이력 1개만 보존

**정답:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: date-printer
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: date
            image: busybox
            command: ["/bin/sh", "-c", "date"]
```

**해설:** Cron 표현식: `*/5 * * * *` = 매 5분. `concurrencyPolicy: Forbid`는 이전 Job이 실행 중이면 새 Job 건너뜀. `Replace`는 기존 Job 삭제 후 새로 시작. `Allow`는 동시 실행 허용.

---

## [문제 11] Liveness & Readiness Probe (난이도: ★★☆)

**문제:** 다음 Probe를 가진 Deployment를 생성하세요:
- HTTP GET `/healthz` (포트 8080) Liveness Probe, 초기 지연 10초, 주기 15초
- TCP 8080 Readiness Probe, 초기 지연 5초, 주기 10초, 실패 임계값 3

**정답:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probe-app
  template:
    metadata:
      labels:
        app: probe-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
```

**해설:** Liveness 실패 → 컨테이너 재시작. Readiness 실패 → 서비스 엔드포인트에서 제거(트래픽 미수신). Startup Probe는 느리게 시작하는 앱의 초기 기동 시간 확보용.

---

## [문제 12] Ingress (난이도: ★★★)

**문제:** 다음 Ingress를 생성하세요:
- `myapp.example.com/api` → `api-service:8080` (Prefix)
- `myapp.example.com/` → `web-service:80` (Prefix)
- TLS 시크릿 `myapp-tls` 적용
- ingressClassName: `nginx`

**정답:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**해설:** `pathType`: Exact(완전 일치), Prefix(접두사 일치), ImplementationSpecific. 더 구체적인 경로가 우선 매칭. TLS Secret은 `type: kubernetes.io/tls`여야 함.

---

## [문제 13] HPA (난이도: ★★☆)

**문제:** `cpu-app` Deployment에 HPA를 설정하세요:
- 최소 2개, 최대 8개 레플리카
- CPU 사용률 60% 기준으로 스케일링

**정답:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```
```bash
# 명령형 방식
kubectl autoscale deployment cpu-app --min=2 --max=8 --cpu-percent=60
```

**해설:** HPA는 Deployment의 Pod에 resource requests가 반드시 설정되어 있어야 작동. metrics-server가 클러스터에 설치되어 있어야 함.

---

## [문제 14] ResourceQuota & LimitRange (난이도: ★★★)

**문제:** `staging` 네임스페이스에 다음을 적용하세요:
- ResourceQuota: CPU 총 requests 2코어, 총 limits 4코어, Pod 최대 10개
- LimitRange: 컨테이너 기본 CPU request 100m/limit 200m, 최대 CPU 1코어

**정답:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "2"
    limits.cpu: "4"
    pods: "10"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limitrange
  namespace: staging
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
    defaultRequest:
      cpu: 100m
    max:
      cpu: "1"
```

**해설:** LimitRange는 request/limit 미설정 시 기본값 주입. ResourceQuota는 네임스페이스 전체 할당량 제한. LimitRange 없이 ResourceQuota만 있으면 request/limit 미설정 Pod 생성 실패.

---

## [문제 15] Sidecar / Multi-container (Kubernetes 1.29+) (난이도: ★★★)

**문제:** 다음 조건으로 Pod를 구성하세요:
- 메인 컨테이너 `nginx` 이미지: `/var/log/nginx`에 로그 기록
- Sidecar `busybox` 이미지: 로그 디렉토리를 `/logs`로 마운트하여 `tail -f /logs/access.log` 실행
- Kubernetes 1.29+ 네이티브 Sidecar 패턴 사용

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  initContainers:
  - name: log-sidecar
    image: busybox
    restartPolicy: Always       # ← 네이티브 sidecar
    command: ["/bin/sh", "-c", "tail -f /logs/access.log 2>/dev/null || sleep infinity"]
    volumeMounts:
    - name: log-vol
      mountPath: /logs
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: log-vol
      mountPath: /var/log/nginx
  volumes:
  - name: log-vol
    emptyDir: {}
```

**해설:** K8s 1.29+의 네이티브 sidecar는 `initContainers` 내에 `restartPolicy: Always`로 정의. 메인 컨테이너보다 먼저 시작되고, 메인 컨테이너 종료 후에도 계속 실행.

---

## [문제 16] Job - 병렬 처리 (난이도: ★★☆)

**문제:** 총 6번 작업을 수행하되 동시에 2개씩 처리하는 Job을 생성하세요. 각 작업은 `echo "processing item"` 명령을 실행하며, 실패 시 재시도는 2번까지 허용하세요.

**정답:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 6
  parallelism: 2
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox
        command: ["/bin/sh", "-c", "echo 'processing item'"]
```

**해설:** `completions`: 전체 성공 횟수. `parallelism`: 동시 실행 Pod 수. `backoffLimit`: Pod 실패 재시도 한도. `restartPolicy: Never` → Pod 실패 시 새 Pod 생성. `OnFailure` → 같은 Pod 재시작.

---

## [문제 17] Ephemeral Container로 디버깅 (난이도: ★★★)

**문제:** 실행 중인 `debug-target` Pod에 `busybox` 이미지의 임시 컨테이너를 붙여 `/proc/1/environ` 파일을 확인하는 명령을 실행하세요.

**정답:**
```bash
kubectl debug -it debug-target \
  --image=busybox:1.35 \
  --target=<main-container-name> \
  -- /bin/sh

# 접속 후
cat /proc/1/environ | tr '\0' '\n'
```

또는 디버그용 복사본 Pod 생성:
```bash
kubectl debug debug-target \
  --image=busybox:1.35 \
  --copy-to=debug-copy \
  -it \
  -- /bin/sh
```

**해설:** Ephemeral Container는 실행 중인 Pod에 임시로 컨테이너 추가. 기존 컨테이너에 shell이 없거나 distroless 이미지 사용 시 유용. `--target`으로 프로세스 네임스페이스 공유.

---

## [문제 18] Service 생성 및 DNS 확인 (난이도: ★★☆)

**문제:** `backend` 네임스페이스에 있는 `api-deployment` (레이블: `app: api`)를 TCP 8080 포트로 서비스 `api-svc`를 생성하세요. 그리고 `default` 네임스페이스에서 이 서비스에 접근하는 전체 DNS 주소를 작성하세요.

**정답:**
```bash
kubectl expose deployment api-deployment \
  -n backend \
  --name=api-svc \
  --port=8080 \
  --target-port=8080 \
  --type=ClusterIP
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: backend
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
```

**DNS 주소:**
```
api-svc.backend.svc.cluster.local
```

**해설:** 다른 네임스페이스에서 접근 시 `<service>.<namespace>.svc.cluster.local` 형식 사용. 같은 네임스페이스에서는 서비스명만으로 접근 가능.

---

## [문제 19] ConfigMap을 볼륨으로 마운트 (난이도: ★★☆)

**문제:** 다음 내용으로 ConfigMap을 생성하고 Pod에 볼륨으로 마운트하세요:
- ConfigMap `nginx-conf`: key=`nginx.conf`, value는 기본 nginx 설정
- Pod `nginx-custom`: ConfigMap을 `/etc/nginx/conf.d/`에 마운트

**정답:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: nginx-config-vol
      mountPath: /etc/nginx/conf.d/
      readOnly: true
  volumes:
  - name: nginx-config-vol
    configMap:
      name: nginx-conf
```

**해설:** ConfigMap 볼륨 마운트 시 key가 파일명이 됨. `items`로 특정 키만 선택 가능. 마운트 디렉토리의 기존 파일들은 숨겨지므로 `subPath`를 사용하면 특정 파일만 오버라이드 가능.

---

## [문제 20] Taint & Toleration (난이도: ★★★)

**문제:** 다음 조건으로 구성하세요:
- `gpu-node` 노드에 taint 추가: `hardware=gpu:NoSchedule`
- GPU 작업을 실행하는 Pod `gpu-pod`에 이 taint를 허용하는 toleration 추가 (`nvidia/cuda:12.0` 이미지 사용)

**정답:**
```bash
# 노드에 taint 추가
kubectl taint nodes gpu-node hardware=gpu:NoSchedule
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "hardware"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: cuda-app
    image: nvidia/cuda:12.0-base
```

**해설:** Taint 종류: `NoSchedule`(스케줄 안 됨), `PreferNoSchedule`(가능하면 피함), `NoExecute`(기존 Pod도 퇴출). `operator: Exists`는 value 무관하게 허용. Toleration은 허용이지, 특정 노드로의 스케줄 보장은 아님(NodeSelector/Affinity 사용).

---

## [문제 21] Node Affinity (난이도: ★★★)

**문제:** `disktype=ssd` 레이블이 있는 노드에만 스케줄되어야 하는 Pod를 생성하세요. 해당 노드가 없으면 스케줄되지 않아야 합니다.

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx
```

**해설:** `requiredDuringSchedulingIgnoredDuringExecution`: 필수 조건 (hard). `preferredDuringSchedulingIgnoredDuringExecution`: 선호 조건 (soft). `operator`: In, NotIn, Exists, DoesNotExist, Gt, Lt.

---

## [문제 22] StatefulSet (난이도: ★★★)

**문제:** 다음 조건의 StatefulSet을 생성하세요:
- 이름: `mysql-ss`, 레플리카: 3
- 이미지: `mysql:8.0`, 환경변수: `MYSQL_ROOT_PASSWORD=secret`
- Headless Service `mysql-headless` 연동
- 각 Pod에 1Gi PVC 자동 생성 (`/var/lib/mysql` 마운트)

**정답:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-ss
spec:
  serviceName: "mysql-headless"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "secret"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

**해설:** StatefulSet의 Pod는 `mysql-ss-0`, `mysql-ss-1`, `mysql-ss-2`처럼 순서 있는 이름을 가짐. DNS: `mysql-ss-0.mysql-headless.default.svc.cluster.local`. `volumeClaimTemplates`로 각 Pod마다 별도 PVC 자동 생성.

---

## [문제 23] 네임스페이스 리소스 조회 및 필터링 (난이도: ★☆☆)

**문제:** 모든 네임스페이스에서 `app=frontend` 레이블을 가진 Pod 목록을 이름과 네임스페이스, 노드 정보를 포함하여 출력하세요.

**정답:**
```bash
kubectl get pods -A -l app=frontend -o wide
```

또는:
```bash
kubectl get pods --all-namespaces -l app=frontend \
  -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,NODE:.spec.nodeName
```

**해설:** `-A` = `--all-namespaces`. `-o wide`는 IP, Node 등 추가 정보 표시. `-o custom-columns`로 원하는 필드 추출. `-o jsonpath`로 스크립트용 출력 가능.

---

## [문제 24] Pod 강제 삭제 및 재생성 (난이도: ★☆☆)

**문제:** `stuck-pod`가 Terminating 상태에서 멈춰 있습니다. 강제 삭제하세요.

**정답:**
```bash
kubectl delete pod stuck-pod --force --grace-period=0
```

**해설:** `--grace-period=0`은 graceful shutdown 없이 즉시 삭제. StatefulSet의 Pod를 강제 삭제할 때는 주의 필요 (데이터 일관성 문제). `--force` 없이 `--grace-period=0`만으로도 작동.

---

## [문제 25] 종합 문제 - 마이크로서비스 배포 (난이도: ★★★★)

**문제:** 다음 아키텍처를 Kubernetes에 배포하세요:
1. `frontend` Deployment (nginx:alpine, 2 replicas), Service(ClusterIP, 80포트)
2. `backend` Deployment (node:18-alpine, 2 replicas), Service(ClusterIP, 3000포트)
3. `frontend`에서 `backend:3000`으로만 Egress 허용하는 NetworkPolicy
4. `backend`는 `frontend`에서만 3000포트 Ingress 허용
5. Ingress: `app.example.com` → `frontend-svc:80`

**정답:**
```yaml
# frontend Deployment + Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
# backend Deployment + Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: node
        image: node:18-alpine
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
---
# NetworkPolicy for frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 3000
  - to:
    ports:
    - protocol: UDP
      port: 53
---
# NetworkPolicy for backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 3000
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

---

# 🚀 시험 환경 설정 및 시간 관리 팁

## 시험 시작 시 즉시 설정
```bash
# 1. 자동완성 설정 (반드시!)
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# 2. 별칭 설정
alias k=kubectl
complete -o default -F __start_kubectl k

# 3. 자주 쓰는 단축 변수
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# 4. vim 설정 (~/.vimrc)
# set ts=2 sts=2 sw=2 et ai
```

## YAML 빠른 생성 패턴
```bash
# Pod
k run mypod --image=nginx $do > pod.yaml

# Deployment
k create deploy myapp --image=nginx --replicas=3 $do > deploy.yaml

# Service
k expose deploy myapp --port=80 --target-port=8080 $do > svc.yaml

# ConfigMap
k create cm myconfig --from-literal=key=value $do > cm.yaml

# Secret
k create secret generic mysecret --from-literal=pass=1234 $do > secret.yaml

# Job
k create job myjob --image=busybox -- echo hello $do > job.yaml

# CronJob
k create cronjob mycron --image=busybox --schedule="*/5 * * * *" -- echo hello $do > cj.yaml

# ServiceAccount
k create sa mysa $do > sa.yaml

# Role
k create role myrole --verb=get,list --resource=pods $do > role.yaml

# RoleBinding
k create rolebinding myrb --role=myrole --serviceaccount=default:mysa $do > rb.yaml
```

## 네임스페이스 전환
```bash
kubectl config set-context --current --namespace=<namespace>
# 이후 모든 명령에서 -n 생략 가능
```

## 핵심 암기 사항
| 항목 | 기억할 것 |
|------|----------|
| Job restartPolicy | Never 또는 OnFailure (Always 불가) |
| PV accessModes | RWO, ROX, RWX, RWOP |
| CronJob schedule | `"분 시 일 월 요일"` |
| Service 포트 | port(서비스) → targetPort(컨테이너) → nodePort |
| Secret type | Opaque, kubernetes.io/tls, kubernetes.io/dockerconfigjson |
| Probe 종류 | httpGet, tcpSocket, exec, grpc |
| Sidecar (1.29+) | initContainers + restartPolicy: Always |
| apiGroups | core API = `""`, apps = `"apps"`, batch = `"batch"` |
