# CKA 완전 정복 학습 가이드
> **기준 버전:** Kubernetes v1.32 | **시험 시간:** 2시간 | **합격 기준:** 66점 이상 | **문제 수:** 15~20문항

---

## 📋 시험 도메인 및 가중치 (2024~2025 최신)

| 도메인 | 가중치 |
|--------|--------|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

---

## 1. Cluster Architecture, Installation & Configuration (25%)

### 1.1 Kubernetes 아키텍처 핵심

#### Control Plane 컴포넌트
```
kube-apiserver      → 모든 API 요청의 진입점, 인증/인가/Admission Control
etcd                → 클러스터 상태 저장 (분산 KV 스토어), Raft 합의 알고리즘
kube-scheduler      → Pod를 노드에 배치 결정 (Filtering → Scoring → Binding)
kube-controller-manager → ReplicaSet, Node, Job 등 컨트롤러 루프 실행
cloud-controller-manager → 클라우드 프로바이더 연동 (LoadBalancer, Volume 등)
```

#### Worker Node 컴포넌트
```
kubelet             → 노드의 에이전트, Pod 생명주기 관리, CRI와 통신
kube-proxy          → 서비스 네트워크 규칙 관리 (iptables/ipvs)
Container Runtime   → containerd, CRI-O (Docker는 v1.24부터 직접 지원 중단)
```

#### 통신 흐름
```
kubectl → kube-apiserver → etcd (상태 저장)
                        → kube-scheduler (Pod 배치)
                        → kube-controller-manager (상태 조정)
                        → kubelet (실제 컨테이너 실행)
```

---

### 1.2 kubeadm을 이용한 클러스터 설치

#### Control Plane 초기화
```bash
# 사전 준비
swapoff -a
sed -i '/swap/d' /etc/fstab

# 커널 모듈 로드
modprobe overlay
modprobe br_netfilter

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# kubeadm 초기화
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP> \
  --kubernetes-version=v1.32.0

# kubeconfig 설정
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# CNI 설치 (Calico 예시)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Worker 노드 조인 토큰 생성
kubeadm token create --print-join-command
```

#### Worker Node 조인
```bash
kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

### 1.3 etcd 백업 및 복원 ⭐⭐⭐ (자주 출제)

```bash
# etcd 백업
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 백업 상태 확인
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd-backup.db --write-out=table

# etcd 복원
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# etcd static pod 수정 (data-dir 경로 변경)
vi /etc/kubernetes/manifests/etcd.yaml
# --data-dir=/var/lib/etcd-restore 로 변경
# volumeMounts, volumes hostPath도 함께 변경

# kubelet 재시작
systemctl restart kubelet
```

---

### 1.4 RBAC (Role-Based Access Control) ⭐⭐⭐

#### RBAC 리소스 구조
```
Role           → 특정 네임스페이스 내 권한 정의
ClusterRole    → 클러스터 전체 또는 비네임스페이스 리소스 권한
RoleBinding    → Role/ClusterRole을 Subject(User/Group/SA)에 바인딩 (네임스페이스)
ClusterRoleBinding → ClusterRole을 Subject에 클러스터 전체 바인딩
```

```yaml
# Role 생성
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

---
# RoleBinding 생성
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# 명령어로 빠르게 생성
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=jane \
  -n dev

# 권한 확인
kubectl auth can-i get pods --as=jane -n dev
kubectl auth can-i '*' '*' --as=system:admin
```

---

### 1.5 클러스터 업그레이드 ⭐⭐⭐

```bash
# Control Plane 업그레이드
apt-get update
apt-cache madison kubeadm  # 사용 가능한 버전 확인

# kubeadm 업그레이드
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.32.0-1.1
apt-mark hold kubeadm

kubeadm upgrade plan
kubeadm upgrade apply v1.32.0

# kubelet, kubectl 업그레이드
kubectl drain <master-node> --ignore-daemonsets
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon <master-node>

# Worker Node 업그레이드 (각 노드 반복)
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data
# 해당 노드에서:
apt-get install -y kubeadm=1.32.0-1.1
kubeadm upgrade node
apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
systemctl daemon-reload && systemctl restart kubelet
# master에서:
kubectl uncordon <worker-node>
```

---

### 1.6 Certificate 관리

```bash
# 인증서 만료일 확인
kubeadm certs check-expiration

# 인증서 갱신
kubeadm certs renew all

# 특정 인증서만 갱신
kubeadm certs renew apiserver

# kubeconfig 인증서 확인
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A 2 Validity

# CSR (Certificate Signing Request) 생성 및 승인
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane/O=dev-group" -out jane.csr

# CertificateSigningRequest 오브젝트 생성
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(cat jane.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF

kubectl certificate approve jane
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

---

## 2. Workloads & Scheduling (15%)

### 2.1 핵심 워크로드 오브젝트

#### Pod 정의 필수 요소
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: prod
  annotations:
    description: "nginx web server"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
    env:
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
    - name: data-vol
      mountPath: /data
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
  volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: data-vol
    persistentVolumeClaim:
      claimName: my-pvc
  restartPolicy: Always
  nodeSelector:
    disktype: ssd
  serviceAccountName: my-sa
```

---

#### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # 최대 비가용 Pod 수
      maxSurge: 1            # 초과 생성 가능한 Pod 수
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```bash
# Deployment 관리 명령어
kubectl rollout status deployment/nginx-deploy
kubectl rollout history deployment/nginx-deploy
kubectl rollout undo deployment/nginx-deploy
kubectl rollout undo deployment/nginx-deploy --to-revision=2
kubectl set image deployment/nginx-deploy nginx=nginx:1.26
kubectl scale deployment nginx-deploy --replicas=5

# --record는 deprecated, 대신 어노테이션 사용
kubectl annotate deployment nginx-deploy \
  kubernetes.io/change-cause="image update to 1.26"
```

---

### 2.2 스케줄링 심화 ⭐⭐⭐

#### Taints & Tolerations
```bash
# Taint 추가 (effect: NoSchedule | PreferNoSchedule | NoExecute)
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 key=value:NoExecute

# Taint 제거
kubectl taint nodes node1 key=value:NoSchedule-

# Toleration (Pod spec)
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

#### Node Affinity
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # Hard requirement
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd", "nvme"]
          - key: region
            operator: NotIn
            values: ["us-west"]
      preferredDuringSchedulingIgnoredDuringExecution:  # Soft requirement
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["zone-a"]
```

#### Pod Affinity / Anti-Affinity
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["nginx"]
        topologyKey: "kubernetes.io/hostname"  # 동일 노드에 배치 금지
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: cache
          topologyKey: "kubernetes.io/hostname"
```

#### Static Pod & DaemonSet
```bash
# Static Pod - kubelet이 직접 관리 (API 서버 없이도 실행)
# 경로: /etc/kubernetes/manifests/
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml, etcd.yaml, kube-controller-manager.yaml, kube-scheduler.yaml

# Static Pod 경로 확인
cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

```yaml
# DaemonSet - 모든 노드(or 선택된 노드)에 Pod 1개씩 실행
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:v1.14
```

---

### 2.3 Resource Limits & LimitRange & ResourceQuota

```yaml
# LimitRange - 네임스페이스 내 기본 리소스 설정
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:          # 기본 limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # 기본 requests
      cpu: "100m"
      memory: "128Mi"
    max:              # 최대
      cpu: "2"
      memory: "1Gi"
    min:              # 최소
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "4"
      memory: "2Gi"

---
# ResourceQuota - 네임스페이스 전체 리소스 제한
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"
    services.loadbalancers: "2"
```

---

### 2.4 HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: "200Mi"
```

```bash
kubectl autoscale deployment nginx-deploy --cpu-percent=70 --min=2 --max=10
```

---

## 3. Services & Networking (20%)

### 3.1 Service 타입 ⭐⭐⭐

```yaml
# ClusterIP (기본값) - 클러스터 내부에서만 접근
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80          # 서비스 포트
    targetPort: 8080  # Pod 포트

---
# NodePort - 노드의 포트를 통해 외부 접근 (30000-32767)
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # 생략 시 자동 할당

---
# LoadBalancer - 클라우드 LB 프로비저닝
apiVersion: v1
kind: Service
metadata:
  name: my-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080

---
# ExternalName - 외부 DNS 이름으로 라우팅
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com

---
# Headless Service - ClusterIP: None (StatefulSet용)
apiVersion: v1
kind: Service
metadata:
  name: headless-svc
spec:
  clusterIP: None
  selector:
    app: stateful-app
  ports:
  - port: 80
```

---

### 3.2 Ingress ⭐⭐⭐

```yaml
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
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
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

---

### 3.3 NetworkPolicy ⭐⭐⭐

```yaml
# 특정 Pod에 대한 Ingress/Egress 제어
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: prod
      podSelector:              # AND 조건 (같은 항목 내)
        matchLabels:
          app: backend
    - podSelector:              # OR 조건 (별도 항목)
        matchLabels:
          app: admin
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53

---
# Default Deny All (모든 트래픽 차단)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

### 3.4 DNS & CoreDNS

```bash
# Kubernetes DNS 형식
# <service-name>.<namespace>.svc.<cluster-domain>
# 예: my-service.prod.svc.cluster.local

# Pod DNS
# <pod-ip-dashes>.<namespace>.pod.<cluster-domain>
# 예: 10-244-1-5.default.pod.cluster.local

# CoreDNS ConfigMap 확인/수정
kubectl get cm coredns -n kube-system -o yaml

# DNS 디버깅
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup my-service.prod.svc.cluster.local
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

---

## 4. Storage (10%)

### 4.1 PV / PVC / StorageClass ⭐⭐⭐

#### PersistentVolume (PV) - 관리자가 프로비저닝
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce      # RWO: 단일 노드 R/W
  # - ReadOnlyMany     # ROX: 다수 노드 읽기
  # - ReadWriteMany    # RWX: 다수 노드 R/W
  # - ReadWriteOncePod # RWOP: 단일 Pod만 (k8s 1.22+)
  persistentVolumeReclaimPolicy: Retain  # Retain | Recycle | Delete
  storageClassName: manual
  hostPath:
    path: /mnt/data
  # 또는 NFS:
  # nfs:
  #   server: nfs-server.example.com
  #   path: /exports/data
```

#### PersistentVolumeClaim (PVC) - 사용자가 요청
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: dev
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
  selector:
    matchLabels:
      type: fast
```

#### StorageClass (동적 프로비저닝)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

### 4.2 Volume 타입

```yaml
spec:
  volumes:
  # emptyDir - Pod 생명주기와 같음
  - name: temp-dir
    emptyDir:
      medium: Memory  # tmpfs 사용
      sizeLimit: "100Mi"

  # hostPath - 노드 파일시스템 마운트
  - name: host-path
    hostPath:
      path: /var/log/containers
      type: DirectoryOrCreate  # Directory | File | Socket | CharDevice | BlockDevice

  # configMap
  - name: config
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: config/app.properties

  # secret
  - name: secret-vol
    secret:
      secretName: my-secret
      defaultMode: 0400

  # projected - 여러 소스를 하나의 볼륨으로
  - name: projected-vol
    projected:
      sources:
      - configMap:
          name: my-config
      - secret:
          name: my-secret
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
```

---

## 5. Troubleshooting (30%) ⭐⭐⭐ 가장 중요

### 5.1 Pod 트러블슈팅 워크플로우

```bash
# 1단계: 상태 확인
kubectl get pods -A -o wide
kubectl describe pod <pod-name> -n <namespace>

# 2단계: 로그 확인
kubectl logs <pod-name>                        # 현재 컨테이너 로그
kubectl logs <pod-name> -c <container-name>   # 특정 컨테이너
kubectl logs <pod-name> --previous            # 이전 컨테이너 로그 (재시작 후)
kubectl logs <pod-name> --tail=50 -f          # 실시간 로그

# 3단계: 컨테이너 접속
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container> -- /bin/sh

# 4단계: 이벤트 확인
kubectl get events --sort-by='.lastTimestamp' -n <namespace>
kubectl get events --field-selector=involvedObject.name=<pod-name>
```

#### Pod 상태별 원인 분석
```
Pending       → 리소스 부족, Node Selector/Affinity 불일치, PVC 미바인딩
ImagePullBackOff → 이미지 이름/태그 오류, Private Registry 인증 실패
CrashLoopBackOff → 애플리케이션 오류, 설정 파일 오류, OOMKilled
OOMKilled     → 메모리 limits 초과 (kubectl describe로 확인)
Evicted       → 노드 디스크/메모리 부족으로 kubelet이 제거
Terminating   → Finalizer 남아있음, 강제 삭제: kubectl delete pod --force --grace-period=0
ContainerCreating → Volume 마운트 대기, init container 실행 중
```

---

### 5.2 Node 트러블슈팅

```bash
# 노드 상태 확인
kubectl get nodes -o wide
kubectl describe node <node-name>

# 노드 컴포넌트 로그 (systemd)
journalctl -u kubelet -f
journalctl -u kubelet --since "1 hour ago"
systemctl status kubelet
systemctl status containerd

# kubelet 재시작
systemctl daemon-reload
systemctl restart kubelet

# 노드 조건(Condition) 확인
kubectl get node <node> -o jsonpath='{.status.conditions[*].type}'
# Ready | MemoryPressure | DiskPressure | PIDPressure | NetworkUnavailable

# 노드 정보 상세
kubectl top nodes  # metrics-server 필요
kubectl top pods --sort-by=memory -A
```

---

### 5.3 네트워크 트러블슈팅

```bash
# DNS 문제
kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup kubernetes
kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never -- dig my-svc.default.svc.cluster.local

# 서비스 연결 문제
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -O- http://my-service:80
kubectl run debug --image=curlimages/curl --rm -it --restart=Never -- curl -v http://my-service:80

# Endpoint 확인 (서비스 → Pod 연결 확인)
kubectl get endpoints my-service
kubectl get ep -A | grep -v "<none>"

# kube-proxy 상태 확인
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# iptables 규칙 확인 (노드에서)
iptables -L -n | grep <service-ip>
iptables -t nat -L KUBE-SERVICES
```

---

### 5.4 Control Plane 트러블슈팅

```bash
# Static Pod 기반 컴포넌트 (kubeadm 설치)
ls /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# API Server 로그
kubectl logs -n kube-system kube-apiserver-<master>
crictl logs <container-id>  # containerd 직접 확인

# 컨테이너 런타임으로 직접 확인
crictl ps -a
crictl pods
crictl inspect <container-id>

# kube-scheduler, kube-controller-manager 로그
kubectl logs -n kube-system kube-scheduler-<master>
kubectl logs -n kube-system kube-controller-manager-<master>

# etcd 상태 확인
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

### 5.5 Application 트러블슈팅

```bash
# ConfigMap / Secret 확인
kubectl get cm <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d

# ServiceAccount 권한 문제
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa
kubectl get rolebindings,clusterrolebindings -A | grep my-sa

# 리소스 사용량 확인
kubectl top pods -n <namespace> --sort-by=cpu
kubectl describe pod <name> | grep -A 5 "Limits\|Requests"

# 환경변수 확인
kubectl exec <pod> -- env | grep DB_

# Port-forward로 직접 테스트
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<svc-name> 8080:80
```

---

## 6. 핵심 kubectl 명령어 치트시트

### 6.1 빠른 리소스 생성 (--dry-run 활용)

```bash
# Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl run nginx --image=nginx --port=80 --env="ENV=prod" --labels="app=nginx"
kubectl run busybox --image=busybox --command -- sleep 3600
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://service

# Deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml
kubectl create deployment nginx --image=nginx --port=80

# Service
kubectl expose pod nginx --port=80 --target-port=8080 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort --name=nginx-svc

# ConfigMap & Secret
kubectl create configmap app-config --from-literal=key1=val1 --from-file=config.properties
kubectl create secret generic db-secret --from-literal=password=mysecret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass

# ServiceAccount
kubectl create serviceaccount my-sa -n dev

# Namespace
kubectl create namespace staging
```

---

### 6.2 조회 & 필터링

```bash
# 라벨 셀렉터
kubectl get pods -l app=nginx,env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l 'env notin (dev)'

# 출력 형식
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o custom-columns='NAME:.metadata.name,STATUS:.status.conditions[-1].type'
kubectl get pods --sort-by='.metadata.creationTimestamp'

# 특정 필드 확인
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'
kubectl get node worker1 -o jsonpath='{.status.capacity}'

# 모든 네임스페이스
kubectl get pods -A
kubectl get all -A

# 리소스 확인
kubectl api-resources
kubectl api-versions
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy --recursive
```

---

### 6.3 수정 & 관리

```bash
# 즉시 수정
kubectl edit deployment nginx-deploy
kubectl patch deployment nginx-deploy -p '{"spec":{"replicas":5}}'
kubectl patch node worker1 -p '{"spec":{"unschedulable":true}}'

# Label & Annotation
kubectl label pod nginx-pod env=production tier=web
kubectl label pod nginx-pod env-              # 라벨 제거
kubectl annotate pod nginx-pod description="web server"

# 노드 관리
kubectl cordon node1      # 스케줄링 금지 (기존 Pod는 유지)
kubectl uncordon node1    # 스케줄링 재개
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data  # 노드 비우기

# 리소스 삭제
kubectl delete pod nginx --grace-period=0 --force
kubectl delete pods --all -n dev
kubectl delete all --all -n dev  # 주의: PV/PVC/Secret 등은 삭제 안됨

# 네임스페이스 컨텍스트
kubectl config set-context --current --namespace=dev
kubectl config get-contexts
kubectl config use-context kubernetes-admin@kubernetes
```

---

## 7. ConfigMap & Secret 심화

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 단순 키-값
  database_host: "mysql.prod.svc.cluster.local"
  database_port: "3306"
  # 파일 형식
  app.properties: |
    log.level=INFO
    max.connections=100
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend;
      }
    }
```

```yaml
# Secret (base64 인코딩)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=      # echo -n 'admin' | base64
  password: cGFzc3dvcmQ=  # echo -n 'password' | base64
stringData:               # 평문으로 작성 (자동 인코딩)
  api-key: "my-secret-api-key"

---
# Secret 타입들
# Opaque                              - 일반 시크릿
# kubernetes.io/service-account-token - SA 토큰
# kubernetes.io/dockerconfigjson      - Docker Registry 인증
# kubernetes.io/tls                   - TLS 인증서
# kubernetes.io/ssh-auth              - SSH 키
```

---

## 8. StatefulSet 핵심

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless  # Headless Service 이름 필수
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
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:          # 각 Pod마다 독립적 PVC 자동 생성
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

```bash
# StatefulSet Pod 이름: <name>-0, <name>-1, <name>-2
# DNS: mysql-0.mysql-headless.default.svc.cluster.local
# 순서대로 생성/삭제 (0→1→2 생성, 2→1→0 삭제)

kubectl get statefulset
kubectl scale statefulset mysql --replicas=5
```

---

## 9. Job & CronJob

```yaml
# Job
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  completions: 5        # 총 완료해야 할 횟수
  parallelism: 2        # 동시 실행 수
  backoffLimit: 4       # 실패 재시도 횟수
  activeDeadlineSeconds: 300  # 최대 실행 시간(초)
  ttlSecondsAfterFinished: 60 # 완료 후 자동 삭제
  template:
    spec:
      restartPolicy: Never  # Never | OnFailure (Always는 불가)
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]

---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"    # 매일 새벽 2시
  concurrencyPolicy: Forbid  # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["/bin/sh", "-c", "backup.sh"]
```

---

## 10. 시험 전략 & 팁

### 환경 설정 (시험 시작 직후)
```bash
# alias 설정
alias k=kubectl
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kl='kubectl logs'

export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# bash completion
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# 현재 컨텍스트 확인
kubectl config get-contexts
kubectl config current-context
```

### 네임스페이스 전환
```bash
kubectl config set-context --current --namespace=<namespace>
# 또는
kubectl get pods -n <namespace>
```

### YAML 빠른 생성 패턴
```bash
# 1. dry-run으로 템플릿 생성 후 수정
kubectl run nginx --image=nginx $do > pod.yaml
vi pod.yaml
kubectl apply -f pod.yaml

# 2. edit으로 즉시 수정
kubectl edit deployment nginx

# 3. patch로 빠른 수정
kubectl patch svc my-svc -p '{"spec":{"type":"NodePort"}}'
```

### 자주 사용하는 문서 북마크
- https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/tasks/administer-cluster/

---

# 📝 예상 출제 문제 (60문항)

## 도메인 1: Cluster Architecture, Installation & Configuration

**Q1.** etcd를 `/opt/backup/etcd-snapshot.db`에 백업하라. etcd는 `https://127.0.0.1:2379`에서 실행 중이며, 인증서는 `/etc/kubernetes/pki/etcd/` 디렉토리에 있다.

**Q2.** `/opt/backup/etcd-snapshot.db` 파일로 etcd를 복원하라. 복원 데이터 디렉토리는 `/var/lib/etcd-new`로 설정하라.

**Q3.** kubeadm을 사용하여 클러스터를 현재 버전에서 v1.32.0으로 업그레이드하라. Control Plane 노드(`master`)를 먼저 업그레이드하고, Worker 노드(`worker1`)도 업그레이드하라.

**Q4.** 다음 조건으로 ClusterRole과 ClusterRoleBinding을 생성하라:
- ClusterRole 이름: `deployment-manager`
- 권한: `deployments` 리소스에 대해 `get`, `list`, `create`, `update`, `delete`
- ClusterRoleBinding 이름: `deployment-manager-binding`
- Subject: User `john`

**Q5.** 네임스페이스 `app-ns`에 다음 조건의 Role과 RoleBinding을 생성하라:
- Role 이름: `app-role` / 권한: pods - get, list, watch / configmaps - get
- RoleBinding 이름: `app-rolebinding`
- Subject: ServiceAccount `app-sa` (같은 네임스페이스)

**Q6.** 사용자 `developer`를 위한 CSR(CertificateSigningRequest)을 생성하고 승인하라. 키 파일은 `/root/developer.key`로 저장하라.

**Q7.** `worker1` 노드의 kubelet이 중단되었다. 원인을 파악하고 복구하라.

**Q8.** 다음 조건으로 ServiceAccount를 생성하고 Pod에 할당하라:
- ServiceAccount: `monitoring-sa` (namespace: `monitoring`)
- ClusterRole: `view` (기존 ClusterRole 사용)
- Pod: `monitor-pod` (image: nginx, 해당 SA 사용)

**Q9.** `kube-apiserver`의 `--audit-log-path` 플래그를 `/var/log/kubernetes/audit.log`로 설정하라.

**Q10.** 현재 클러스터의 모든 노드에서 실행 중인 kubelet 버전을 확인하고 파일에 저장하라.

---

## 도메인 2: Workloads & Scheduling

**Q11.** 다음 조건으로 Deployment를 생성하라:
- 이름: `web-deploy` / Namespace: `prod`
- Image: `nginx:1.24` / Replicas: 4
- Label: `app=web, tier=frontend`
- Resources: requests(cpu:100m, memory:128Mi), limits(cpu:200m, memory:256Mi)

**Q12.** `web-deploy` Deployment를 `nginx:1.25`로 롤링 업데이트하고, 변경 사유를 어노테이션으로 기록하라. 업데이트 후 이전 버전으로 롤백하라.

**Q13.** `node1` 노드에 Taint `env=production:NoSchedule`을 추가하라. 그리고 해당 Taint를 허용하는 Pod `prod-pod` (image: nginx)를 생성하라.

**Q14.** 다음 조건으로 Pod를 생성하라:
- 이름: `affinity-pod` / Image: `nginx`
- Node Affinity: `disktype=ssd` 라벨이 있는 노드에만 스케줄링 (required)
- Pod Anti-Affinity: `app=nginx` 라벨의 Pod와 동일 노드에 배치 금지

**Q15.** 다음 조건으로 Static Pod를 생성하라:
- 이름: `static-nginx` / Image: `nginx:1.25`
- `worker1` 노드에 생성 (해당 노드의 staticPodPath에 YAML 파일 배치)

**Q16.** 네임스페이스 `quota-ns`에 다음 조건의 ResourceQuota를 생성하라:
- pods: 10 / requests.cpu: 2 / requests.memory: 4Gi
- limits.cpu: 4 / limits.memory: 8Gi

**Q17.** 네임스페이스 `limit-ns`에 LimitRange를 생성하라:
- Container 기본 request: cpu=50m, memory=64Mi
- Container 기본 limit: cpu=200m, memory=256Mi

**Q18.** `worker2` 노드에 스케줄링되지 않도록 표시(cordon)하고, 현재 실행 중인 Pod를 모두 다른 노드로 이동(drain)하라.

**Q19.** 다음 조건으로 DaemonSet을 생성하라:
- 이름: `log-collector` / Image: `fluentd:v1.14`
- 모든 노드(control-plane 포함)에 배포
- hostPath `/var/log`를 `/logs`로 마운트

**Q20.** `cpu-deploy` Deployment에 대해 HPA를 설정하라:
- 최소 replicas: 2 / 최대: 8
- CPU 사용률 80% 초과 시 스케일 아웃

**Q21.** 다음 조건으로 Job을 생성하라:
- 이름: `batch-job` / Image: `busybox`
- Command: `echo "processing" && sleep 5`
- completions: 6 / parallelism: 3

**Q22.** 매주 월요일 오전 9시에 실행되는 CronJob을 생성하라:
- 이름: `weekly-report` / Image: `busybox`
- Command: `date >> /tmp/report.txt`
- 동시 실행 정책: Forbid

**Q23.** 다음 조건으로 init container가 포함된 Pod를 생성하라:
- Pod 이름: `init-pod`
- Init Container: `init-c` (image: busybox) - `/work-dir/index.html`에 "Hello" 작성
- Main Container: `main-c` (image: nginx) - emptyDir 볼륨을 `/usr/share/nginx/html`에 마운트

---

## 도메인 3: Services & Networking

**Q24.** 다음 조건으로 Service를 생성하라:
- 이름: `web-svc` / Type: ClusterIP
- Selector: `app=web`
- Port: 80 → targetPort: 8080

**Q25.** `web-svc` Service를 NodePort 타입으로 변경하고 nodePort를 `31000`으로 지정하라.

**Q26.** 다음 조건으로 Ingress를 생성하라:
- 이름: `web-ingress` / IngressClass: nginx
- Host: `myapp.example.com`
- Path `/api` → `api-svc:8080`
- Path `/` → `web-svc:80`

**Q27.** 다음 조건으로 NetworkPolicy를 생성하라:
- 이름: `db-network-policy` / Namespace: `database`
- 대상 Pod: `app=mysql`
- `backend` 네임스페이스의 `app=api` Pod에서만 포트 3306 접근 허용
- 나머지 모든 Ingress 트래픽 차단

**Q28.** 네임스페이스 `secure-ns`에 Default Deny All NetworkPolicy를 적용하라. 그리고 같은 네임스페이스 내 Pod들끼리는 모든 통신을 허용하는 정책을 추가하라.

**Q29.** CoreDNS가 정상 동작하는지 확인하고, `busybox` Pod를 실행하여 `kubernetes.default` 서비스의 DNS 조회 결과를 `/root/dns-result.txt`에 저장하라.

**Q30.** 네임스페이스 `frontend`의 Pod가 네임스페이스 `backend`의 `api-service`에만 접근할 수 있도록 NetworkPolicy를 구성하라.

**Q31.** `app=web` Pod 그룹에서만 포트 443으로 Egress를 허용하고 나머지는 차단하는 NetworkPolicy를 생성하라.

**Q32.** 다음 조건의 Service를 생성하고 Pod와 연결을 확인하라:
- Headless Service 이름: `stateful-svc`
- Selector: `app=stateful`
- Port: 80

---

## 도메인 4: Storage

**Q33.** 다음 조건으로 PersistentVolume을 생성하라:
- 이름: `pv-log` / 용량: 100Mi
- AccessMode: ReadWriteMany
- hostPath: `/pv/log`
- StorageClass: `manual`
- ReclaimPolicy: Retain

**Q34.** 다음 조건으로 PVC를 생성하고 Pod에 마운트하라:
- PVC 이름: `pvc-data` / 요청 용량: 50Mi
- AccessMode: ReadWriteOnce / StorageClass: `manual`
- Pod 이름: `pvc-pod` (image: nginx)
- 마운트 경로: `/var/data`

**Q35.** 기존 PVC `app-pvc`의 용량을 500Mi에서 1Gi로 확장하라. (StorageClass에 `allowVolumeExpansion: true` 설정 필요)

**Q36.** 다음 조건으로 StorageClass를 생성하라:
- 이름: `local-storage`
- Provisioner: `kubernetes.io/no-provisioner`
- VolumeBindingMode: `WaitForFirstConsumer`
- ReclaimPolicy: `Delete`

**Q37.** 현재 클러스터에서 `Released` 상태인 PV를 찾아 재사용 가능하도록 `Available` 상태로 변경하라.

**Q38.** `emptyDir` 볼륨을 공유하는 sidecar 패턴의 Pod를 생성하라:
- Container 1: `writer` (busybox) - 매초 `/shared/log`에 타임스탬프 기록
- Container 2: `reader` (busybox) - `/shared/log`를 tail -f로 출력

---

## 도메인 5: Troubleshooting

**Q39.** `broken-deploy` Deployment의 Pod들이 모두 `Pending` 상태이다. 원인을 파악하고 수정하라.
*(힌트: 존재하지 않는 노드 셀렉터 또는 리소스 부족)*

**Q40.** `crash-pod` Pod가 `CrashLoopBackOff` 상태이다. 이전 컨테이너 로그를 확인하고 원인을 파악하라.

**Q41.** `worker2` 노드가 `NotReady` 상태이다. 원인을 파악하고 복구하라.
*(힌트: kubelet 서비스 중단, containerd 설정 오류 등)*

**Q42.** `app-deploy` Deployment의 Pod가 `ImagePullBackOff` 상태이다. 올바른 이미지 이름(`nginx:1.25`)으로 수정하라.

**Q43.** `api-service`에 접근이 안된다. Service의 Selector와 Pod의 Label이 일치하는지 확인하고 수정하라.

**Q44.** `web-pod` Pod가 `OOMKilled`로 재시작을 반복한다. 메모리 limits를 `512Mi`로 증가시켜라.

**Q45.** `kube-scheduler`가 실행되지 않아 Pod가 `Pending` 상태로 남아있다. Static Pod 매니페스트를 확인하고 수정하라.

**Q46.** 모든 네임스페이스에서 `Pending` 상태인 Pod 목록을 `/root/pending-pods.txt`에 저장하라.

**Q47.** `kube-apiserver`에 접근할 수 없다. `/etc/kubernetes/manifests/kube-apiserver.yaml`을 확인하고 오류를 수정하라.
*(힌트: 잘못된 etcd 엔드포인트, 인증서 경로 오류 등)*

**Q48.** `monitoring` 네임스페이스의 `prometheus-pod`가 다른 Pod의 메트릭을 수집하지 못한다. RBAC 문제를 파악하고 필요한 권한을 부여하라.

**Q49.** `worker1` 노드에서 `/var/log/pods` 디렉토리가 가득 차 있다. 디스크 압박(DiskPressure) 상태를 해소하고 노드를 정상화하라.

**Q50.** 실행 중인 Pod `debug-pod`의 네트워크 설정(IP, 라우팅 테이블)을 확인하고 외부 DNS(`8.8.8.8`)로 ping 테스트 결과를 `/root/network-test.txt`에 저장하라.

---

## 종합 & 심화 문제

**Q51.** 다음 조건의 완전한 애플리케이션 스택을 배포하라:
- Namespace: `myapp`
- Deployment: `frontend` (nginx:1.25, 3 replicas)
- ConfigMap: `frontend-config` (nginx.conf 포함)
- Service: `frontend-svc` (ClusterIP, port 80)
- NetworkPolicy: `frontend` 네임스페이스 내부 통신만 허용

**Q52.** StatefulSet으로 3개 replica의 nginx를 배포하고, 각 Pod에 독립적인 PVC(1Gi)가 할당되도록 설정하라. Headless Service도 함께 생성하라.

**Q53.** 현재 클러스터의 모든 `Running` 상태 Pod의 이름과 IP를 JSON 형식으로 `/root/running-pods.json`에 저장하라.

**Q54.** `multi-container-pod`를 생성하라:
- Container 1: `nginx` - nginx:1.25 (포트 80)
- Container 2: `sidecar` - busybox (nginx 로그를 stdout으로 출력)
- 공유 볼륨: `/var/log/nginx`

**Q55.** 기존 Deployment `legacy-app`을 다음과 같이 수정하라:
- Blue/Green 배포를 위해 새 버전 Deployment `legacy-app-v2` 생성
- `app-service`의 트래픽을 v2로 전환 (selector 변경)
- v1 Deployment를 0개 replica로 스케일 다운

**Q56.** 다음 조건으로 Pod Security 설정을 적용하라:
- 네임스페이스 `secure-app`에 `restricted` Pod Security Standard 적용
- 이를 위반하는 기존 Pod를 수정하여 non-root로 실행

**Q57.** 클러스터의 모든 노드 정보(이름, OS, 커널 버전, 컨테이너 런타임)를 확인하여 `/root/node-info.txt`에 저장하라.

**Q58.** `kube-system` 네임스페이스에서 가장 많은 CPU를 사용 중인 Pod를 찾아 이름을 `/root/top-cpu-pod.txt`에 저장하라.

**Q59.** 다음 조건으로 Pod를 특정 노드(`node01`)에 강제 배치하라:
- nodeName을 직접 지정하는 방법
- nodeSelector를 사용하는 방법
- Node Affinity(required)를 사용하는 방법
각각의 YAML을 작성하라.

**Q60.** 장애 시나리오: 클러스터에서 다음 증상이 발생했다. 각각 원인과 해결 방법을 기술하라:
- 새로운 Pod가 계속 Pending 상태
- Service에 접근되지 않음 (Endpoint는 있음)
- 노드가 주기적으로 NotReady 전환
- etcd 리더 선출 실패

---

## 📌 시험 핵심 체크리스트

### 반드시 숙지해야 할 명령어
```bash
✅ kubectl run / create / apply / edit / patch / delete
✅ kubectl get / describe / logs / exec / explain
✅ kubectl rollout status/history/undo
✅ kubectl scale / autoscale
✅ kubectl cordon / uncordon / drain / taint
✅ kubectl top / auth can-i
✅ kubectl config use-context / set-context
✅ etcdctl snapshot save / restore
✅ kubeadm upgrade plan / apply
✅ kubeadm certs check-expiration / renew
✅ journalctl -u kubelet / systemctl restart kubelet
✅ crictl ps / logs / inspect
```

### 자주 틀리는 포인트
```
❗ NetworkPolicy - AND vs OR 조건 구분 (같은 from 항목 = AND, 다른 항목 = OR)
❗ PVC가 PV에 바인딩되려면 StorageClass, AccessMode, 용량이 맞아야 함
❗ Taint 제거 시 마지막에 '-' 추가
❗ CronJob schedule: "분 시 일 월 요일"
❗ Secret 값은 base64 인코딩 (stringData 사용 시 자동 인코딩)
❗ Static Pod 경로: /etc/kubernetes/manifests/ (kubelet config.yaml로 확인)
❗ kubeadm upgrade 순서: kubeadm → drain → kubelet/kubectl → uncordon
❗ etcdctl 사용 시 반드시 인증서 3개 (cacert, cert, key) 지정
❗ RoleBinding과 ClusterRoleBinding의 roleRef는 수정 불가 (삭제 후 재생성)
❗ imagePullPolicy: Always (latest 태그), IfNotPresent (나머지 기본값)
```
