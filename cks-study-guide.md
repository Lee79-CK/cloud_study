# CKS (Certified Kubernetes Security Specialist) 완전 학습 가이드 2025

> **시험 버전**: Kubernetes v1.32 기준  
> **시험 시간**: 2시간  
> **합격 점수**: 67점 이상  
> **형식**: 실기 시험 (핸즈온)

---

## 📚 시험 도메인 & 가중치

| 도메인 | 가중치 |
|--------|--------|
| 1. 클러스터 설정 (Cluster Setup) | 10% |
| 2. 클러스터 강화 (Cluster Hardening) | 15% |
| 3. 시스템 강화 (System Hardening) | 15% |
| 4. 마이크로서비스 취약점 최소화 | 20% |
| 5. 공급망 보안 (Supply Chain Security) | 20% |
| 6. 모니터링, 로깅 및 런타임 보안 | 20% |

---

# 🔐 DOMAIN 1: 클러스터 설정 (Cluster Setup) — 10%

## 1.1 CIS Benchmark

CIS(Center for Internet Security) Benchmark는 Kubernetes 클러스터 보안 설정의 업계 표준입니다.

### kube-bench 사용
```bash
# kube-bench 실행 (전체)
kube-bench run --targets master,node

# 특정 체크만 실행
kube-bench run --check 1.2.1

# JSON 출력
kube-bench run --json > results.json
```

### 주요 CIS 체크 항목
- **1.2.1**: API Server에 anonymous-auth 비활성화
- **1.2.6**: API Server audit log 활성화
- **1.2.16**: admission controller 설정 (PodSecurity 등)
- **1.3.2**: etcd 데이터 암호화
- **4.2.1**: kubelet anonymous 인증 비활성화

---

## 1.2 Kubernetes Dashboard 보안

```yaml
# 안전한 Dashboard 설정
# 읽기 전용 ServiceAccount 사용
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-reader
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-reader-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view  # cluster-admin 절대 사용 금지
subjects:
- kind: ServiceAccount
  name: dashboard-reader
  namespace: kubernetes-dashboard
```

**보안 위험 요소:**
- `--enable-skip-login` 플래그 사용 금지
- `--insecure-port` 사용 금지
- cluster-admin 권한의 ServiceAccount 사용 금지

---

## 1.3 Ingress TLS 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

```bash
# TLS Secret 생성
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n default
```

---

## 1.4 NodeRestriction Admission Controller

```yaml
# kube-apiserver 플래그 (--admission-plugins에 NodeRestriction 포함)
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,PodSecurity,...
```

NodeRestriction은 kubelet이 자신의 노드와 해당 노드에 스케줄된 Pod만 수정할 수 있도록 제한합니다.

---

# 🔒 DOMAIN 2: 클러스터 강화 (Cluster Hardening) — 15%

## 2.1 RBAC (Role-Based Access Control)

### Role과 RoleBinding (네임스페이스 범위)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole과 ClusterRoleBinding (클러스터 범위)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  # resourceNames으로 특정 Secret만 허용 가능
  resourceNames: ["my-secret"]
```

### RBAC 권한 확인
```bash
# 현재 사용자 권한 확인
kubectl auth can-i create pods
kubectl auth can-i delete secrets --namespace production

# 특정 사용자 권한 확인
kubectl auth can-i list pods --as jane
kubectl auth can-i create deployments --as system:serviceaccount:default:my-sa

# 모든 권한 조회
kubectl auth can-i --list
kubectl auth can-i --list --as jane
```

### 최소 권한 원칙 (Least Privilege)
```bash
# 불필요한 ClusterRoleBinding 탐지
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# ServiceAccount에 automountServiceAccountToken 비활성화
kubectl patch serviceaccount default \
  -p '{"automountServiceAccountToken": false}'
```

---

## 2.2 ServiceAccount 보안

```yaml
# automountServiceAccountToken 비활성화 (SA 레벨)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-sa
  namespace: default
automountServiceAccountToken: false  # 중요!
---
# Pod 레벨에서도 비활성화 가능
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: secure-sa
  automountServiceAccountToken: false  # Pod 레벨 설정이 우선
  containers:
  - name: app
    image: nginx:alpine
```

### Projected ServiceAccount Token (Bound Service Account Tokens)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-projected-token
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200  # 2시간 만료
          audience: vault
```

---

## 2.3 API Server 강화

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml 주요 보안 설정
spec:
  containers:
  - command:
    - kube-apiserver
    # 익명 인증 비활성화
    - --anonymous-auth=false
    # 인증 모드 설정
    - --authorization-mode=Node,RBAC
    # Admission Controller 활성화
    - --enable-admission-plugins=NodeRestriction,PodSecurity,PodSecurityPolicy
    # 안전하지 않은 포트 비활성화
    - --insecure-port=0
    # Audit 로깅
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    # TLS 설정
    - --tls-min-version=VersionTLS12
    - --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    # etcd 암호화 통신
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
```

---

## 2.4 kubelet 강화

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 익명 인증 비활성화
authentication:
  anonymous:
    enabled: false  # 중요!
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
# 인증 모드
authorization:
  mode: Webhook  # AlwaysAllow 사용 금지
# 읽기 전용 포트 비활성화
readOnlyPort: 0  # 중요!
# 보호된 커널 기본값
protectKernelDefaults: true
# 스트리밍 연결 타임아웃
streamingConnectionIdleTimeout: 5m
# 이벤트 레코드 QPS
eventRecordQPS: 5
```

```bash
# kubelet 설정 확인
cat /var/lib/kubelet/config.yaml
systemctl status kubelet

# kubelet 인증 테스트
# 익명 접근 시도 (실패해야 함)
curl -k https://node-ip:10250/pods
```

---

## 2.5 etcd 보안

```bash
# etcd 암호화 설정 확인
# /etc/kubernetes/manifests/etcd.yaml에서 확인
grep -i "client-cert-auth" /etc/kubernetes/manifests/etcd.yaml

# etcd 데이터 직접 조회 (보안 테스트)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret
```

### etcd 데이터 암호화 (Encryption at Rest)
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps  # configmap도 암호화 권장
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>  # head -c 32 /dev/urandom | base64
  - identity: {}  # 기존 데이터 읽기용 (새 데이터엔 첫 번째 provider 사용)
```

```bash
# API Server에 암호화 설정 적용
# kube-apiserver.yaml에 추가:
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# 기존 Secret 재암호화
kubectl get secrets -A -o json | kubectl replace -f -

# 암호화 확인 (etcd에서 직접 조회시 암호화된 값이 보여야 함)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret | hexdump -C
```

---

# 🛡️ DOMAIN 3: 시스템 강화 (System Hardening) — 15%

## 3.1 OS 레벨 보안

### 불필요한 서비스 비활성화
```bash
# 실행 중인 서비스 확인
systemctl list-units --type=service --state=running

# 불필요한 서비스 중지 및 비활성화
systemctl stop snapd
systemctl disable snapd
systemctl stop bluetooth
systemctl disable bluetooth

# 열린 포트 확인
ss -tulpn
netstat -tulpn

# 불필요한 패키지 제거
apt remove --purge snapd telnet rsh-client
```

### 사용자 계정 관리
```bash
# 불필요한 사용자 확인
cat /etc/passwd | grep -v nologin | grep -v false

# 빈 패스워드 계정 확인
awk -F: '($2 == "") {print $1}' /etc/shadow

# UID 0 계정 확인 (root 외 존재하면 안 됨)
awk -F: '($3 == "0") {print $1}' /etc/passwd
```

---

## 3.2 AppArmor

AppArmor는 프로세스의 파일시스템/네트워크/시스템콜 접근을 제한하는 MAC(Mandatory Access Control) 시스템입니다.

```bash
# AppArmor 상태 확인
aa-status
apparmor_status

# 프로필 목록
ls /etc/apparmor.d/

# 프로필 로드
apparmor_parser -q /etc/apparmor.d/my-profile

# 프로필 강제 적용 모드로 로드
apparmor_parser -r /etc/apparmor.d/my-profile
```

### AppArmor 프로필 예시
```
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # 쓰기 작업 거부
  deny /** w,
}
```

### Pod에 AppArmor 적용 (Kubernetes 1.30+)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-apparmor-example-deny-write
  containers:
  - name: app
    image: nginx
    # 컨테이너 레벨 오버라이드도 가능
    securityContext:
      appArmorProfile:
        type: RuntimeDefault  # 런타임 기본 프로필 사용
```

> **주의**: Kubernetes 1.30 이전에는 annotation으로 설정  
> `container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/<profile-name>`

---

## 3.3 Seccomp

Seccomp(Secure Computing Mode)은 프로세스가 사용할 수 있는 시스템 콜을 필터링합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # 런타임 기본 Seccomp 프로필 사용
  containers:
  - name: app
    image: nginx
---
# 커스텀 Seccomp 프로필 사용 (노드의 /var/lib/kubelet/seccomp/에 위치)
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-custom-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json  # /var/lib/kubelet/seccomp/profiles/audit.json
  containers:
  - name: app
    image: nginx
```

### Seccomp 프로필 예시
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["accept4", "bind", "connect", "read", "write", "close",
                "socket", "openat", "getdents64", "stat", "fstat",
                "exit_group", "epoll_wait", "epoll_ctl"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

## 3.4 Linux Capabilities

```yaml
# 특정 capability 추가/제거
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL          # 모든 capability 제거
        add:
        - NET_BIND_SERVICE  # 필요한 것만 추가
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
```

### 주요 Linux Capabilities
| Capability | 설명 |
|---|---|
| NET_ADMIN | 네트워크 관리 |
| SYS_ADMIN | 시스템 관리 (매우 강력, 사용 금지 권장) |
| SYS_PTRACE | 프로세스 추적 |
| NET_RAW | RAW 소켓 사용 |
| DAC_OVERRIDE | 파일 권한 무시 |
| SETUID/SETGID | UID/GID 변경 |

---

# 🔑 DOMAIN 4: 마이크로서비스 취약점 최소화 — 20%

## 4.1 Pod Security Standards (PSS)

Kubernetes 1.25에서 PodSecurityPolicy가 제거되고 Pod Security Admission으로 대체되었습니다.

### 3가지 보안 레벨

| 레벨 | 설명 | 사용 상황 |
|------|------|-----------|
| **privileged** | 제한 없음 | 시스템/인프라 워크로드 |
| **baseline** | 최소 제한 | 일반 애플리케이션 |
| **restricted** | 강력한 제한 | 보안 중요 애플리케이션 |

### 3가지 모드

| 모드 | 동작 |
|------|------|
| **enforce** | 위반 시 Pod 생성 거부 |
| **audit** | 위반 시 감사 로그에 기록 (생성은 허용) |
| **warn** | 위반 시 경고 메시지 출력 (생성은 허용) |

```yaml
# 네임스페이스 레벨 PSS 설정
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.32
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.32
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.32
```

```bash
# 네임스페이스 레이블로 PSS 적용
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# PSS 위반 테스트
kubectl run privileged-test --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","securityContext":{"privileged":true}}]}}' \
  -n secure-namespace
```

### restricted 정책 요구사항
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault  # 또는 Localhost
  volumes:
  - name: data
    emptyDir: {}
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false  # 필수
      readOnlyRootFilesystem: true      # 권장
      runAsNonRoot: true                # 필수
      capabilities:
        drop:
        - ALL                           # 필수
```

---

## 4.2 NetworkPolicy

```yaml
# 기본 거부 정책 (Default Deny)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # 모든 Pod에 적용
  policyTypes:
  - Ingress
  - Egress
---
# 특정 트래픽만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: production
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
  # DNS 허용 (필수)
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### AND vs OR 조건 이해
```yaml
# AND 조건: 같은 from 항목 안에 여러 selector
ingress:
- from:
  - podSelector:          # podSelector AND namespaceSelector 모두 충족해야 함
      matchLabels:
        role: frontend
    namespaceSelector:
      matchLabels:
        env: production

# OR 조건: 별도의 from 항목
ingress:
- from:
  - podSelector:          # OR
      matchLabels:
        role: frontend
  - namespaceSelector:    # OR
      matchLabels:
        env: production
```

---

## 4.3 OPA Gatekeeper

OPA(Open Policy Agent) Gatekeeper는 Kubernetes의 Admission Control을 위한 정책 엔진입니다.

```yaml
# ConstraintTemplate 정의
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      
      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Pod에 필수 레이블이 없습니다: %v", [missing])
      }
---
# Constraint 인스턴스 적용
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    labels: ["team", "app", "version"]
```

---

## 4.4 Container Image 보안

```bash
# 이미지 취약점 스캔 (Trivy)
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest
trivy image --exit-code 1 --severity CRITICAL nginx:latest  # CI/CD에서 사용

# Dockerfile 스캔
trivy config Dockerfile

# 파일시스템 스캔
trivy fs /path/to/project
```

### 안전한 Dockerfile 작성
```dockerfile
# 멀티스테이지 빌드
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o app .

# 최소 이미지 사용
FROM gcr.io/distroless/static:nonroot  # distroless 이미지
COPY --from=builder /app/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

---

# 📦 DOMAIN 5: 공급망 보안 (Supply Chain Security) — 20%

## 5.1 이미지 서명 및 검증 (Cosign)

```bash
# cosign으로 이미지 서명
cosign sign --key cosign.key nginx:latest

# 서명 검증
cosign verify --key cosign.pub nginx:latest

# SBOM(Software Bill of Materials) 생성
cosign attest --predicate sbom.json --type cyclonedx nginx:latest
```

### Sigstore/Cosign Admission Controller
```yaml
# Policy Controller로 서명 검증
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: image-policy
spec:
  images:
  - glob: "gcr.io/my-project/**"
  authorities:
  - keyless:
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: https://accounts.google.com
        subject: ci-service@my-project.iam.gserviceaccount.com
```

---

## 5.2 Admission Controller 심화

```yaml
# ImagePolicyWebhook 설정
# kube-apiserver.yaml에 추가
- --enable-admission-plugins=ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/admission-config.yaml

# /etc/kubernetes/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/image-policy-webhook-kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false  # 기본적으로 모든 이미지 거부
```

---

## 5.3 Static Analysis - Kubesec

```bash
# kubesec으로 YAML 보안 스캔
kubesec scan pod.yaml
kubesec scan deployment.yaml

# 출력 예시:
# {
#   "score": 3,
#   "passing": [...],
#   "advise": [...],
#   "critical": [...]
# }

# HTTP API로 사용
curl -sSX POST --data-binary @pod.yaml https://v2.kubesec.io/scan
```

---

## 5.4 Audit Logging

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# 전체 규칙에 적용될 기본 레벨
omitStages:
- "RequestReceived"
rules:
# Secrets에 대한 모든 요청 기록 (민감 데이터)
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Pod 변경 기록
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["pods"]

# 시스템 컴포넌트 불필요한 로그 제외
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services", "services/status"]

# 헬스체크 제외
- level: None
  nonResourceURLs:
  - "/healthz"
  - "/readyz"
  - "/livez"

# 그 외 메타데이터만 기록
- level: Metadata
```

### Audit 레벨 정리
| 레벨 | 기록 내용 |
|------|----------|
| None | 기록하지 않음 |
| Metadata | 메타데이터만 (요청 헤더 등) |
| Request | 메타데이터 + 요청 바디 |
| RequestResponse | 메타데이터 + 요청/응답 바디 |

---

# 🔍 DOMAIN 6: 모니터링, 로깅 및 런타임 보안 — 20%

## 6.1 Falco

Falco는 런타임 보안 도구로 비정상적인 시스템 콜을 감지합니다.

```bash
# Falco 설치 확인
systemctl status falco

# Falco 로그 확인
journalctl -fu falco
cat /var/log/falco.log

# 실시간 이벤트 모니터링
falco -r /etc/falco/falco_rules.yaml
```

### Falco 규칙 작성
```yaml
# /etc/falco/falco_rules.yaml 또는 /etc/falco/rules.d/
- rule: Detect Shell in Container
  desc: 컨테이너 내에서 shell이 실행됨
  condition: >
    spawned_process
    and container
    and shell_procs
  output: >
    컨테이너 내 shell 실행 감지
    (user=%user.name container=%container.id
     image=%container.image.repository:%container.image.tag
     cmd=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, mitre_execution]

- rule: Write Below Binary Dir
  desc: 바이너리 디렉토리에 쓰기 시도
  condition: >
    open_write
    and bin_dir
    and not package_mgmt_procs
  output: >
    바이너리 디렉토리 쓰기 시도 (user=%user.name
    command=%proc.cmdline file=%fd.name)
  priority: ERROR

- rule: Suspicious Network Activity
  desc: 컨테이너에서 예상치 못한 네트워크 연결
  condition: >
    outbound
    and container
    and fd.sport != 443
    and fd.sport != 80
    and not k8s_containers
  output: >
    의심스러운 네트워크 활동
    (user=%user.name command=%proc.cmdline
     connection=%fd.name)
  priority: NOTICE
```

```bash
# 커스텀 규칙 추가 후 재시작
vi /etc/falco/rules.d/custom-rules.yaml
systemctl restart falco

# Falco 설정 파일
cat /etc/falco/falco.yaml
```

---

## 6.2 Immutable Container

```yaml
# 읽기 전용 루트 파일시스템 + 필요한 디렉토리만 쓰기 가능
apiVersion: v1
kind: Pod
metadata:
  name: immutable-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      readOnlyRootFilesystem: true   # 루트 파일시스템 읽기 전용
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 101
    volumeMounts:
    - name: nginx-cache
      mountPath: /var/cache/nginx  # nginx가 쓰기 필요한 디렉토리
    - name: nginx-run
      mountPath: /var/run          # PID 파일용
    - name: tmp
      mountPath: /tmp              # 임시 파일용
  volumes:
  - name: nginx-cache
    emptyDir: {}
  - name: nginx-run
    emptyDir: {}
  - name: tmp
    emptyDir: {}
```

---

## 6.3 mTLS (Mutual TLS) with Service Mesh

### Istio PeerAuthentication
```yaml
# 네임스페이스 전체에 STRICT mTLS 적용
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # PERMISSIVE, STRICT, DISABLE
---
# 특정 워크로드에만 적용
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: workload-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: DISABLE  # 특정 포트는 mTLS 제외
```

---

## 6.4 Runtime Class

```yaml
# RuntimeClass 정의
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor  # gVisor 샌드박스 런타임
handler: runsc   # 컨테이너 런타임 핸들러
---
# Kata Containers
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-containers
handler: kata
---
# Pod에서 RuntimeClass 사용
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-pod
spec:
  runtimeClassName: gvisor  # 보안 강화된 런타임 사용
  containers:
  - name: app
    image: nginx
```

---

# 📝 예상 출제 문제 (100문항)

## 🟢 기초 문제 (1-30)

---

**Q1.** kube-bench를 사용하여 마스터 노드의 CIS 벤치마크 체크를 실행하고, 결과를 `/tmp/kube-bench-master.txt`에 저장하시오.

**정답:**
```bash
kube-bench run --targets master > /tmp/kube-bench-master.txt
# 또는
kube-bench run --targets controlplane > /tmp/kube-bench-master.txt
```

**해설:** kube-bench는 CIS Kubernetes Benchmark 기준으로 클러스터를 검사하는 도구입니다. `--targets` 옵션으로 검사 대상을 지정하며, master/controlplane/node/etcd/policies 등을 대상으로 할 수 있습니다.

---

**Q2.** `production` 네임스페이스에서 `developer` 사용자가 Pod를 조회(`get`, `list`, `watch`)할 수 있는 Role과 RoleBinding을 생성하시오.

**정답:**
```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n production

kubectl create rolebinding developer-pod-reader \
  --role=pod-reader \
  --user=developer \
  -n production
```

**해설:** Role은 특정 네임스페이스 내의 리소스에 대한 권한을 정의하고, RoleBinding은 이 Role을 특정 사용자/그룹/ServiceAccount에 바인딩합니다. ClusterRole/ClusterRoleBinding은 클러스터 전체 범위에 적용됩니다.

---

**Q3.** `default` 네임스페이스의 `default` ServiceAccount에 대해 자동 마운트를 비활성화하시오.

**정답:**
```bash
kubectl patch serviceaccount default \
  -n default \
  -p '{"automountServiceAccountToken": false}'
```
또는 YAML:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
automountServiceAccountToken: false
```

**해설:** `automountServiceAccountToken: false`를 설정하면 Pod 생성 시 ServiceAccount 토큰이 자동으로 마운트되지 않습니다. 이는 불필요한 API 접근 권한을 제거하는 중요한 보안 조치입니다.

---

**Q4.** `developer` 사용자가 `kube-system` 네임스페이스에서 Secret을 생성할 수 있는지 확인하는 명령어를 작성하시오.

**정답:**
```bash
kubectl auth can-i create secrets \
  --namespace kube-system \
  --as developer
```

**해설:** `kubectl auth can-i` 명령어는 특정 사용자/ServiceAccount가 특정 리소스에 대한 특정 동작을 수행할 수 있는지 확인합니다. `--as` 플래그로 다른 사용자를 가장(impersonate)할 수 있습니다.

---

**Q5.** `/etc/kubernetes/manifests/kube-apiserver.yaml`에서 anonymous 인증을 비활성화하시오.

**정답:**
```bash
# kube-apiserver.yaml의 command 섹션에 추가 또는 수정
- --anonymous-auth=false
```

**해설:** `--anonymous-auth=false`를 설정하면 인증되지 않은 요청이 `system:anonymous` 사용자로 처리되지 않고 거부됩니다. CIS Benchmark 1.2.1 항목에 해당합니다.

---

**Q6.** `nginx` 네임스페이스에 `pod-security.kubernetes.io/enforce: baseline` 레이블을 추가하시오.

**정답:**
```bash
kubectl label namespace nginx \
  pod-security.kubernetes.io/enforce=baseline
```

**해설:** Pod Security Admission(PSA)은 네임스페이스 레이블을 통해 보안 정책을 적용합니다. `enforce`는 위반 시 Pod 생성을 거부하고, `audit`은 로그에 기록하며, `warn`은 경고를 표시합니다.

---

**Q7.** `secure-ns` 네임스페이스에서 모든 Ingress와 Egress를 기본으로 차단하는 NetworkPolicy를 생성하시오.

**정답:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**해설:** `podSelector: {}`는 네임스페이스의 모든 Pod에 적용됩니다. `policyTypes`에 Ingress와 Egress를 모두 명시하면 두 방향의 트래픽이 모두 차단됩니다. 이후 필요한 통신만 별도 NetworkPolicy로 허용합니다.

---

**Q8.** 다음 중 Kubernetes에서 `restricted` PSS(Pod Security Standard)가 요구하는 설정이 아닌 것은?

a) `allowPrivilegeEscalation: false`  
b) `runAsNonRoot: true`  
c) `capabilities.drop: ["ALL"]`  
d) `hostNetwork: false`  

**정답:** d

**해설:** `restricted` PSS는 a, b, c와 함께 seccompProfile 설정을 요구합니다. `hostNetwork: false`는 `baseline` PSS에서 요구하는 항목으로, `restricted`에서는 이미 포함되어 있지만 별도 설정 항목은 아닙니다. restricted의 핵심 요구사항은 privilegeEscalation 금지, nonRoot 실행, 모든 capability 제거, seccomp 프로필 적용입니다.

---

**Q9.** Falco가 설치된 노드에서 컨테이너 내에서 shell을 실행하는 이벤트를 감지하는 규칙이 동작하는지 확인하고, 로그를 확인하시오.

**정답:**
```bash
# Falco 서비스 상태 확인
systemctl status falco

# Falco 로그 실시간 확인
journalctl -fu falco

# 테스트: 컨테이너에서 shell 실행
kubectl exec -it <pod-name> -- /bin/sh

# Falco 로그 파일 확인
cat /var/log/falco.log | grep "shell"
```

**해설:** Falco는 커널 수준의 시스템 콜을 모니터링하여 비정상적인 행동을 감지합니다. 컨테이너 내에서 shell 실행, 파일 시스템 변조, 네트워크 연결 등을 감지하는 기본 규칙을 제공합니다.

---

**Q10.** `myapp` Pod의 컨테이너에 `readOnlyRootFilesystem: true`를 설정하고, `/tmp` 디렉토리만 쓰기 가능하도록 구성하시오.

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
  volumes:
  - name: tmp-volume
    emptyDir: {}
```

**해설:** `readOnlyRootFilesystem: true`는 컨테이너의 루트 파일시스템을 읽기 전용으로 만들어 악성 코드 실행 또는 파일 변조를 방지합니다. 필요한 쓰기 가능 디렉토리는 emptyDir 볼륨으로 마운트합니다.

---

**Q11.** etcd Secret 데이터를 암호화하기 위한 EncryptionConfiguration을 작성하고 API Server에 적용하시오.

**정답:**
```bash
# 암호화 키 생성
head -c 32 /dev/urandom | base64
# 출력: <BASE64_KEY>
```
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <BASE64_KEY>
  - identity: {}
```
```bash
# kube-apiserver.yaml에 추가
- --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# 기존 Secret 재암호화
kubectl get secrets -A -o json | kubectl replace -f -
```

**해설:** EncryptionConfiguration은 etcd에 저장되는 데이터를 암호화합니다. `aescbc`는 AES-CBC 방식의 암호화를 사용하며, `identity` 프로바이더는 암호화하지 않습니다. providers 배열의 첫 번째 항목이 새 데이터 암호화에 사용됩니다.

---

**Q12.** `app=frontend` 레이블의 Pod가 `app=backend` 레이블의 Pod의 포트 8080으로만 통신할 수 있도록 NetworkPolicy를 작성하시오.

**정답:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: default
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
      port: 8080
```

**해설:** NetworkPolicy는 `podSelector`로 정책이 적용될 Pod를 선택하고, `ingress.from`으로 허용할 소스를 정의합니다. 이 정책은 backend Pod에 대해 frontend Pod에서 오는 8080 포트 트래픽만 허용합니다.

---

**Q13.** kubelet의 읽기 전용 포트를 비활성화하는 방법을 설명하시오.

**정답:**
```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
readOnlyPort: 0
```
```bash
systemctl daemon-reload
systemctl restart kubelet
```

**해설:** kubelet의 기본 읽기 전용 포트(10255)는 인증 없이 Pod, 노드 정보를 제공하므로 보안상 반드시 비활성화해야 합니다(`readOnlyPort: 0`). CIS Benchmark 4.2.4 항목에 해당합니다.

---

**Q14.** `my-pod` Pod에 AppArmor 프로필 `runtime/default`를 적용하시오. (Kubernetes 1.30 이전 방식)

**정답:**
```yaml
# Kubernetes 1.30 이전 방식 (annotation)
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
spec:
  containers:
  - name: app
    image: nginx
```

**해설:** Kubernetes 1.30 이전에는 AppArmor를 annotation으로 설정했습니다. 형식은 `container.apparmor.security.beta.kubernetes.io/<컨테이너명>: <프로필>`입니다. 1.30+에서는 `securityContext.appArmorProfile`을 사용합니다.

---

**Q15.** Trivy를 사용하여 `nginx:1.19` 이미지의 HIGH, CRITICAL 취약점만 스캔하시오.

**정답:**
```bash
trivy image --severity HIGH,CRITICAL nginx:1.19
```

**해설:** Trivy는 컨테이너 이미지, 파일시스템, Git 저장소 등의 취약점을 스캔하는 도구입니다. `--severity` 옵션으로 특정 심각도의 취약점만 표시할 수 있으며, CI/CD 파이프라인에서 `--exit-code 1`을 사용하면 취약점 발견 시 빌드를 실패시킬 수 있습니다.

---

**Q16.** 다음 RBAC 설정에서 문제점을 찾고 수정하시오.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-app-binding
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

**정답/해설:**
문제점: `cluster-admin` ClusterRole은 모든 리소스에 대한 모든 권한을 부여합니다. 최소 권한 원칙에 위배됩니다.

수정: 애플리케이션에 필요한 최소한의 권한만 부여
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
roleRef:
  kind: Role
  name: my-app-role
  apiGroup: rbac.authorization.k8s.io
```

---

**Q17.** 모든 Kubernetes Audit 이벤트 중 Secret에 대한 요청은 `RequestResponse` 레벨로, 그 외는 `Metadata` 레벨로 기록하는 Audit Policy를 작성하시오.

**정답:**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
- level: Metadata
```

**해설:** Audit Policy의 규칙은 순서대로 적용되며, 첫 번째로 매칭되는 규칙이 적용됩니다. Secret에 대한 규칙을 먼저 정의하고, 나머지는 Metadata 레벨로 처리합니다. `level: Metadata`는 요청/응답 바디 없이 메타데이터만 기록합니다.

---

**Q18.** Pod가 실행될 때 `/proc/sys/kernel` 경로에 쓰기를 금지하는 Seccomp 프로필을 작성하시오.

**정답:**
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["write", "openat"],
      "action": "SCMP_ACT_ERRNO",
      "args": []
    }
  ]
}
```

실제로는 Seccomp는 syscall 기반이므로 파일 경로 필터링은 AppArmor가 더 적합합니다:
```
profile deny-proc-write {
  deny /proc/sys/kernel/** w,
}
```

**해설:** Seccomp는 시스템 콜 단위로 필터링하고, AppArmor는 파일 경로/네트워크/capability 단위로 필터링합니다. 특정 파일 경로에 대한 쓰기 제한은 AppArmor가 더 적합합니다.

---

**Q19.** `restricted` PSS가 적용된 네임스페이스에서 Pod가 실행되려면 최소한 어떤 securityContext 설정이 필요한가?

**정답:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      runAsNonRoot: true
```

**해설:** restricted PSS의 필수 요구사항:
1. `allowPrivilegeEscalation: false` (컨테이너 레벨)
2. `capabilities.drop: ["ALL"]` (컨테이너 레벨)
3. `runAsNonRoot: true` (Pod 또는 컨테이너 레벨)
4. seccompProfile 설정 (RuntimeDefault 또는 Localhost)
5. volume 유형 제한 (hostPath 불가)

---

**Q20.** cluster-admin 권한을 가진 모든 ClusterRoleBinding을 조회하시오.

**정답:**
```bash
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# 또는
kubectl get clusterrolebindings \
  -o custom-columns='NAME:.metadata.name,ROLE:.roleRef.name,SUBJECTS:.subjects' | \
  grep cluster-admin
```

**해설:** cluster-admin 권한은 클러스터의 모든 리소스에 대한 모든 권한을 가지므로, 이를 가진 ServiceAccount/사용자를 정기적으로 감사해야 합니다.

---

## 🟡 중급 문제 (31-65)

---

**Q21.** 다음 Falco 규칙을 해석하고, 어떤 상황에서 알림이 발생하는지 설명하시오.

```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name %container.info shell=%proc.name
    parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
```

**정답/해설:**
이 규칙은 다음 조건을 모두 만족할 때 알림을 발생시킵니다:
- `spawned_process`: 새 프로세스가 생성됨
- `container`: 컨테이너 내부에서 발생
- `shell_procs`: 프로세스가 shell(bash, sh, zsh 등)
- `proc.tty != 0`: 터미널(TTY)이 연결된 상태

즉, **`kubectl exec -it pod -- /bin/bash`처럼 대화형 shell이 컨테이너에서 실행될 때** 경고를 발생시킵니다. 공격자가 침투 후 컨테이너에서 명령을 실행하는 상황을 감지합니다.

---

**Q22.** gVisor를 사용하는 RuntimeClass를 생성하고, 이를 사용하는 Pod를 배포하시오.

**정답:**
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-nginx
spec:
  runtimeClassName: gvisor
  containers:
  - name: nginx
    image: nginx:alpine
```

**해설:** gVisor는 컨테이너와 호스트 커널 사이에 샌드박스 레이어를 제공합니다. `runsc`는 gVisor의 런타임 핸들러입니다. RuntimeClass를 사용하면 다양한 격리 수준의 런타임을 선택적으로 사용할 수 있습니다.

---

**Q23.** `production` 네임스페이스에서 `frontend` Pod들이 외부 인터넷(0.0.0.0/0)에 접근하는 것을 막고, 같은 네임스페이스의 `backend` Pod에만 접근 가능하도록 NetworkPolicy를 작성하시오.

**정답:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: backend
  # DNS 허용 필수
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

**해설:** Egress NetworkPolicy를 사용하여 외부 통신을 차단하고 내부 backend Pod로의 통신만 허용합니다. DNS 쿼리(포트 53)는 서비스 이름 해석을 위해 반드시 허용해야 합니다.

---

**Q24.** OPA Gatekeeper를 사용하여 `latest` 태그의 이미지 사용을 금지하는 ConstraintTemplate과 Constraint를 작성하시오.

**정답:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snolatesttag
spec:
  crd:
    spec:
      names:
        kind: K8sNoLatestTag
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8snolatesttag
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        endswith(container.image, ":latest")
        msg := sprintf("latest 태그 사용 금지: %v", [container.image])
      }
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not contains(container.image, ":")
        msg := sprintf("태그 없는 이미지 금지: %v", [container.image])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoLatestTag
metadata:
  name: no-latest-tag
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

**해설:** OPA의 Rego 언어를 사용하여 정책을 정의합니다. `violation` 규칙이 참이면 해당 리소스의 생성/수정이 거부됩니다. 태그가 없는 이미지도 기본적으로 `latest`를 사용하므로 함께 차단해야 합니다.

---

**Q25.** Kubernetes API Server에서 Audit 로그 설정이 올바른지 확인하고, 최근 Secret 관련 Audit 이벤트를 조회하시오.

**정답:**
```bash
# Audit 로그 설정 확인
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep audit

# Audit 로그에서 Secret 관련 이벤트 검색
cat /var/log/kubernetes/audit.log | jq '. | select(.objectRef.resource=="secrets")'

# 특정 시간 이후의 이벤트
cat /var/log/kubernetes/audit.log | \
  jq '. | select(.objectRef.resource=="secrets" and .verb=="get")'

# GET 이외의 Secret 접근 (write 작업)
cat /var/log/kubernetes/audit.log | \
  jq '. | select(.objectRef.resource=="secrets" and (.verb=="create" or .verb=="update" or .verb=="delete"))'
```

---

**Q26.** `private-registry.company.com` 레지스트리의 이미지만 사용하도록 강제하는 OPA Gatekeeper 정책을 작성하시오.

**정답:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedrepos
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not startswith(container.image, input.parameters.repos[_])
        msg := sprintf("허용되지 않은 레지스트리: %v", [container.image])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    repos:
    - "private-registry.company.com/"
```

---

**Q27.** 다음 Pod 스펙에서 보안 취약점을 모두 찾으시오.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
spec:
  hostNetwork: true
  hostPID: true
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      privileged: true
      runAsUser: 0
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
```

**정답/해설:**
취약점 목록:
1. **`hostNetwork: true`**: Pod가 호스트 네트워크 네임스페이스를 공유 → 호스트 네트워크 인터페이스 접근 가능
2. **`hostPID: true`**: Pod가 호스트 PID 네임스페이스를 공유 → 호스트의 모든 프로세스 조회/조작 가능
3. **`image: nginx:latest`**: latest 태그 사용 → 버전 관리 불가, 재현 불가능
4. **`privileged: true`**: 모든 Linux capability 부여 → 컨테이너 탈출 가능
5. **`runAsUser: 0`**: root 사용자로 실행 → 권한 상승 위험
6. **`hostPath: /`**: 호스트 루트 파일시스템 마운트 → 호스트 완전 접근 가능

---

**Q28.** `etcd` 데이터가 암호화되어 있는지 확인하는 방법을 설명하시오.

**정답:**
```bash
# 1. API를 통해 Secret 생성
kubectl create secret generic test-secret \
  --from-literal=key=supersecret

# 2. etcd에서 직접 조회
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C | head -20

# 암호화된 경우: k8s:enc:aescbc:v1:key1: 로 시작
# 암호화 안 된 경우: 평문으로 확인됨
```

**해설:** etcd에서 직접 Secret을 조회했을 때 값이 `k8s:enc:aescbc:v1:key1:`로 시작하면 암호화가 적용된 것입니다. 암호화되지 않은 경우 `supersecret` 값이 평문으로 보입니다.

---

**Q29.** ServiceAccount 토큰을 특정 대상(audience)에만 유효하도록 설정한 Projected Volume을 작성하시오.

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: token-projection-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: app-token
  volumes:
  - name: app-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600  # 1시간 만료
          audience: "https://my-api-server.example.com"  # 특정 audience
```

**해설:** Projected ServiceAccount Token은 만료 시간과 대상(audience)을 지정할 수 있어 기존 무기한 토큰보다 훨씬 안전합니다. audience를 지정하면 해당 서비스에서만 유효한 토큰이 생성됩니다.

---

**Q30.** `kubesec`으로 Pod YAML을 스캔하고 점수를 확인하시오.

**정답:**
```bash
# 파일로 스캔
kubesec scan pod.yaml

# stdin으로 스캔
cat pod.yaml | kubesec scan -

# HTTP API 사용
curl -sSX POST --data-binary @pod.yaml https://v2.kubesec.io/scan | jq

# 점수 확인 (score > 0 이면 기본적으로 통과)
kubesec scan pod.yaml | jq '.[0].score'
```

---

**Q31.** NetworkPolicy에서 AND 조건과 OR 조건의 차이를 예시로 설명하시오.

**정답:**
```yaml
# AND 조건: 같은 배열 항목 내에 여러 selector (모두 만족해야 함)
ingress:
- from:
  - podSelector:         # frontend Pod이면서
      matchLabels:
        app: frontend
    namespaceSelector:   # AND production 네임스페이스인 경우만 허용
      matchLabels:
        env: production

# OR 조건: 별도 배열 항목으로 분리 (하나만 만족해도 됨)
ingress:
- from:
  - podSelector:         # frontend Pod이거나
      matchLabels:
        app: frontend
  - namespaceSelector:   # OR production 네임스페이스의 모든 Pod
      matchLabels:
        env: production
```

**해설:** 이 차이는 CKS 시험에서 자주 출제됩니다. 같은 `-from` 항목 안에 여러 selector를 쓰면 AND, 별도의 `-from` 항목으로 쓰면 OR 조건입니다.

---

**Q32.** `system:masters` 그룹 멤버십의 위험성을 설명하고, 이를 확인하는 방법을 작성하시오.

**정답/해설:**
`system:masters` 그룹은 RBAC를 완전히 우회하고 모든 리소스에 대한 접근을 허용합니다. 이는 RBAC 정책보다 상위 레벨에서 동작하므로, ClusterRoleBinding을 삭제해도 권한이 유지됩니다.

```bash
# 인증서에서 system:masters 그룹 확인
openssl x509 -in /etc/kubernetes/pki/admin.crt -text | grep "Subject:"
# Subject: O=system:masters, CN=kubernetes-admin 형식으로 표시

# kubeconfig에서 확인
kubectl config view --raw | grep -A 5 "user:"

# 현재 사용자 확인
kubectl auth whoami
```

---

**Q33.** Pod Security Admission을 사용하여 특정 Pod만 예외적으로 허용하는 방법을 설명하시오.

**정답:**
```yaml
# 방법 1: 네임스페이스 레이블로 예외 처리 (다른 네임스페이스 사용)
# privileged 작업을 위한 별도 네임스페이스 생성
apiVersion: v1
kind: Namespace
metadata:
  name: privileged-workloads
  labels:
    pod-security.kubernetes.io/enforce: privileged

# 방법 2: Pod 레벨 예외 (Exemptions 설정)
# kube-apiserver의 AdmissionConfiguration에 설정
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "restricted"
    exemptions:
      usernames: ["system:serviceaccount:kube-system:replicaset-controller"]
      runtimeClasses: []
      namespaces: ["kube-system", "monitoring"]
```

---

**Q34.** Kubernetes에서 컨테이너 탈출(Container Escape)을 방지하는 방법을 설명하시오.

**정답/해설:**
컨테이너 탈출 방지 방법:

1. **privileged 모드 금지**: `securityContext.privileged: false`
2. **capabilities 최소화**: `capabilities.drop: [ALL]`
3. **hostPath 마운트 금지**: NetworkPolicy로 제한
4. **hostPID/hostNetwork 금지**: PSS baseline 이상 적용
5. **readOnlyRootFilesystem**: `readOnlyRootFilesystem: true`
6. **seccomp 프로필 적용**: 위험한 시스템 콜 차단
7. **AppArmor 프로필 적용**: 파일시스템/네트워크 접근 제한
8. **gVisor/Kata Containers 사용**: RuntimeClass로 샌드박스 적용
9. **allowPrivilegeEscalation 비활성화**: `allowPrivilegeEscalation: false`

---

**Q35.** Kubernetes 버전 업그레이드 전 보안 관점에서 확인해야 할 사항을 설명하시오.

**정답/해설:**
1. **Deprecation 확인**: PodSecurityPolicy(1.25에서 제거) → PSA로 마이그레이션
2. **RBAC 변경사항**: API 그룹 변경으로 인한 권한 정책 검토
3. **API 버전 변경**: 사용 중인 API 버전이 제거되는지 확인
4. **기존 Admission Controller**: 새 버전과 호환성 확인
5. **CVE 패치 확인**: 새 버전에서 해결된 보안 취약점 목록 확인
6. **etcd 백업**: 업그레이드 전 반드시 etcd 백업 수행

```bash
# API 사용 현황 확인
kubectl api-versions
kubectl deprecations  # pluto 도구 사용
```

---

**Q36.** Kubernetes에서 Secret을 안전하게 관리하는 방법을 5가지 이상 설명하시오.

**정답/해설:**
1. **etcd 암호화**: EncryptionConfiguration으로 저장 시 암호화
2. **RBAC 접근 제한**: Secret에 대한 get/list 권한 최소화
3. **외부 Secret 관리자 사용**: Vault, AWS Secrets Manager + ESO(External Secrets Operator)
4. **automountServiceAccountToken 비활성화**: 불필요한 SA 토큰 마운트 방지
5. **시크릿 로테이션**: 정기적인 비밀값 교체
6. **감사 로깅**: Secret 접근에 대한 RequestResponse 레벨 감사
7. **readOnlyRootFilesystem**: Secret이 마운트된 경로 이외 쓰기 방지
8. **Sealed Secrets**: GitOps 환경에서 암호화된 형태로 Git 저장소에 저장

---

**Q37.** `kube-system` 네임스페이스에서 `system:anonymous` 사용자의 접근을 확인하고 차단하는 방법을 설명하시오.

**정답:**
```bash
# system:anonymous 관련 ClusterRoleBinding 확인
kubectl get clusterrolebindings -o yaml | \
  grep -B5 "system:anonymous"

kubectl get rolebindings -A -o yaml | \
  grep -B5 "system:anonymous"

# API Server에서 anonymous 비활성화
# /etc/kubernetes/manifests/kube-apiserver.yaml
- --anonymous-auth=false

# 또는 RBAC로 system:anonymous의 불필요한 권한 제거
kubectl get clusterrolebinding system:public-info-viewer
kubectl delete clusterrolebinding system:public-info-viewer
```

---

**Q38.** Istio를 사용하지 않고 Pod 간 mTLS를 구현하는 방법을 설명하시오.

**정답/해설:**
Istio 없이 mTLS를 구현하는 방법:

1. **cert-manager 사용**: Certificate 리소스로 자동 인증서 관리
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-cert
spec:
  secretName: app-tls-secret
  issuerRef:
    name: cluster-issuer
    kind: ClusterIssuer
  dnsNames:
  - app.production.svc.cluster.local
```

2. **애플리케이션 레벨 mTLS**: 앱 코드에서 직접 클라이언트 인증서 검증
3. **SPIFFE/SPIRE**: 워크로드 ID 관리 시스템

---

**Q39.** `node01`에서 실행 중인 컨테이너의 모든 파일시스템 접근을 모니터링하는 Falco 규칙을 작성하시오.

**정답:**
```yaml
- rule: Container Filesystem Access on node01
  desc: node01의 컨테이너에서 파일시스템 접근 감지
  condition: >
    open_read
    and container
    and k8s.node.name = "node01"
  output: >
    파일 접근 (user=%user.name container=%container.name
    file=%fd.name cmd=%proc.cmdline)
  priority: INFO
  tags: [filesystem, container]
```

---

**Q40.** `admission-controller` 설정에서 `AlwaysAdmit`과 `AlwaysDeny`를 사용하면 안 되는 이유를 설명하시오.

**정답/해설:**
- **AlwaysAdmit**: 모든 요청을 검증 없이 허용 → 보안 정책이 전혀 적용되지 않음
- **AlwaysDeny**: 모든 요청을 거부 → 클러스터가 완전히 동작 불가

실제 CKS 시험에서는 이 두 컨트롤러가 활성화되어 있는지 확인하고 제거하는 문제가 출제될 수 있습니다:
```bash
# 현재 활성화된 Admission Controller 확인
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission-plugins
# AlwaysAdmit이 있으면 제거하고 저장
```

---

**Q41.** Pod의 capabilities를 확인하는 방법을 설명하시오.

**정답:**
```bash
# Pod 내부에서 확인
kubectl exec pod-name -- cat /proc/1/status | grep Cap

# capsh로 해석
kubectl exec pod-name -- capsh --decode=00000000a80425fb

# amicontained 도구 사용 (컨테이너 보안 정보 확인)
kubectl run tmp --rm -i --tty --image=r.j3ss.co/amicontained -- amicontained
```

---

**Q42.** Kubernetes에서 Pod가 호스트 파일시스템을 마운트하는 것을 감지하는 방법을 설명하시오.

**정답:**
```bash
# hostPath 볼륨을 사용하는 Pod 찾기
kubectl get pods -A -o json | jq '.items[] | 
  select(.spec.volumes[]?.hostPath != null) | 
  {name: .metadata.name, namespace: .metadata.namespace, 
   volumes: [.spec.volumes[] | select(.hostPath != null)]}'

# 특정 경로를 마운트하는 Pod 찾기
kubectl get pods -A -o json | jq '.items[] | 
  select(.spec.volumes[]?.hostPath.path == "/") | 
  .metadata.name'
```

---

**Q43.** 다음 Audit Policy에서 잘못된 부분을 찾고 수정하시오.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: None
  resources:
  - group: ""
    resources: ["secrets"]
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
```

**정답/해설:**
문제점: 규칙은 위에서 아래로 순서대로 매칭됩니다. 첫 번째 규칙이 Secret을 `None`(기록 안 함)으로 설정하므로 두 번째 규칙은 절대 실행되지 않습니다.

수정:
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse  # 더 구체적인 규칙을 먼저
  resources:
  - group: ""
    resources: ["secrets"]
- level: Metadata          # 나머지는 Metadata 레벨
```

---

**Q44.** `image-scanner` ServiceAccount가 모든 네임스페이스의 Pod를 조회하되, Secret은 조회할 수 없도록 RBAC를 설정하시오.

**정답:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: image-scanner
  namespace: security
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
# Secret은 명시적으로 제외 (포함하지 않음)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: image-scanner-pod-viewer
subjects:
- kind: ServiceAccount
  name: image-scanner
  namespace: security
roleRef:
  kind: ClusterRole
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

**해설:** RBAC는 화이트리스트 방식입니다. 명시적으로 허용하지 않은 리소스는 자동으로 접근이 거부됩니다. Secret을 명시적으로 deny할 필요 없이, 그냥 포함하지 않으면 됩니다.

---

## 🔴 심화 문제 (66-100)

---

**Q45.** 다음과 같은 시나리오에서 침해 사고를 조사하는 절차를 설명하시오: "production 네임스페이스에서 비정상적인 네트워크 연결이 감지됨"

**정답/해설:**
```bash
# 1. 의심스러운 Pod 식별
kubectl get pods -n production -o wide
kubectl top pods -n production  # 리소스 사용량 확인

# 2. Pod의 네트워크 연결 확인
kubectl exec -n production <suspicious-pod> -- ss -tulpn
kubectl exec -n production <suspicious-pod> -- netstat -an

# 3. 프로세스 확인
kubectl exec -n production <suspicious-pod> -- ps aux

# 4. 최근 파일 변경 확인
kubectl exec -n production <suspicious-pod> -- find / -newer /tmp -not -path /proc -not -path /sys 2>/dev/null

# 5. 환경 변수 확인 (민감 정보 노출 여부)
kubectl exec -n production <suspicious-pod> -- env

# 6. Falco 로그 확인
journalctl -fu falco | grep <container-id>

# 7. Audit 로그 확인
cat /var/log/kubernetes/audit.log | \
  jq '. | select(.objectRef.namespace=="production")'

# 8. Pod 격리 (필요시)
kubectl label pod <pod-name> -n production quarantine=true
# NetworkPolicy로 격리된 Pod 트래픽 차단
```

---

**Q46.** Supply Chain 공격을 방어하기 위한 Kubernetes 파이프라인 보안 전략을 설명하시오.

**정답/해설:**
```
Supply Chain 보안 레이어:

1. 소스 코드 레벨:
   - SAST(Static Application Security Testing)
   - 의존성 취약점 스캔 (npm audit, go mod verify)
   - Git 커밋 서명 (GPG)

2. 빌드 레벨:
   - 신뢰된 베이스 이미지만 사용
   - 멀티스테이지 빌드
   - 빌드 환경 격리
   - SBOM 생성 (syft, cosign attest)

3. 이미지 레벨:
   - 취약점 스캔 (Trivy, Grype)
   - 이미지 서명 (cosign sign)
   - Distroless/Alpine 최소 이미지 사용
   - latest 태그 금지

4. 레지스트리 레벨:
   - 프라이빗 레지스트리 사용
   - 이미지 서명 검증
   - 취약한 이미지 격리

5. 배포 레벨:
   - ImagePolicyWebhook으로 서명 검증
   - OPA Gatekeeper로 정책 적용
   - Admission Controller로 이미지 허용 목록 관리

6. 런타임 레벨:
   - Falco로 런타임 이상 감지
   - 읽기 전용 파일시스템
   - 네트워크 정책 적용
```

---

**Q47.** Kubernetes 클러스터에서 Zero Trust 보안 모델을 구현하는 방법을 설명하시오.

**정답/해설:**
```
Zero Trust = "절대 신뢰하지 않고, 항상 검증"

1. 아이덴티티 검증:
   - ServiceAccount + JWT 토큰
   - mTLS (Istio PeerAuthentication STRICT)
   - SPIFFE/SPIRE 워크로드 아이덴티티

2. 최소 권한:
   - RBAC로 필요한 권한만 부여
   - automountServiceAccountToken: false
   - Projected SA Token (만료 시간 설정)

3. 마이크로 세그멘테이션:
   - NetworkPolicy로 Pod 간 통신 제한
   - default-deny-all 정책
   - 필요한 통신만 명시적 허용

4. 암호화:
   - etcd 데이터 암호화
   - TLS 1.2+ 통신
   - Secret 암호화

5. 지속적 검증:
   - Audit 로깅
   - Falco 런타임 모니터링
   - OPA 정책 적용

6. 환경 격리:
   - Namespace 분리
   - RuntimeClass (gVisor, Kata)
   - PSS restricted 적용
```

---

**Q48.** Kubernetes Secrets를 HashiCorp Vault와 통합하는 방법을 설명하시오.

**정답/해설:**
```bash
# 방법 1: Vault Agent Injector (Sidecar)
# Pod annotation으로 설정
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"

# 방법 2: External Secrets Operator (ESO)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-k8s-secret
  data:
  - secretKey: db-password
    remoteRef:
      key: secret/myapp
      property: db-password

# 방법 3: CSI Secret Store Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-database
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault:8200"
    roleName: "database"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/db"
        secretKey: "password"
```

---

**Q49.** CKS 시험에서 자주 틀리는 실수 TOP 10을 설명하시오.

**정답/해설:**
1. **NetworkPolicy AND/OR 혼동**: 같은 `from` 항목 = AND, 별도 항목 = OR
2. **PSS와 PSP 혼동**: PSP는 1.25에서 제거됨, PSS/PSA 사용
3. **AppArmor 버전 혼동**: 1.30+ 에서 securityContext 사용, 이전 버전은 annotation
4. **Audit Policy 규칙 순서**: 위에서 아래로 첫 번째 매칭 규칙 적용
5. **RBAC deny 불가**: RBAC은 허용(allow)만 가능, deny 없음
6. **Seccomp vs AppArmor 용도 혼동**: Seccomp=syscall 필터, AppArmor=파일/네트워크 필터
7. **암호화 키 base64 인코딩**: 32바이트 랜덤 → base64 인코딩 후 사용
8. **kubelet 재시작 누락**: 설정 변경 후 `systemctl restart kubelet` 필수
9. **static Pod 수정**: /etc/kubernetes/manifests/ 디렉토리의 YAML 직접 수정
10. **DNS 허용 누락**: Egress NetworkPolicy에서 포트 53 UDP/TCP 허용 필수

---

**Q50.** 다음 시나리오를 해결하시오: "누군가 클러스터에 접근하여 Secret을 읽었다는 의심이 있다. Audit 로그에서 이를 확인하시오."

**정답:**
```bash
# Secret 조회 관련 Audit 이벤트 검색
cat /var/log/kubernetes/audit.log | \
  jq '. | select(.objectRef.resource=="secrets" and .verb=="get")' | \
  jq '{time: .requestReceivedTimestamp, user: .user.username, 
       secret: .objectRef.name, ns: .objectRef.namespace, 
       sourceIP: .sourceIPs}'

# 특정 시간 이후 Secret 접근
cat /var/log/kubernetes/audit.log | \
  jq '. | select(
    .objectRef.resource=="secrets" 
    and (.verb=="get" or .verb=="list") 
    and .requestReceivedTimestamp > "2024-01-01T00:00:00Z"
  )'

# 외부 IP에서의 접근 확인
cat /var/log/kubernetes/audit.log | \
  jq '. | select(.objectRef.resource=="secrets")' | \
  jq 'select(.sourceIPs[] | startswith("10.") | not)'
```

---

**Q51.** 컨테이너 내에서 특정 syscall을 감지하는 Falco 규칙과 해당 syscall을 차단하는 Seccomp 프로필을 작성하시오. (ptrace 시스템 콜 대상)

**정답:**
```yaml
# Falco 규칙 (감지)
- rule: Ptrace Anti-Debug
  desc: ptrace 시스템 콜 사용 감지 (디버깅/인젝션 시도)
  condition: >
    evt.type=ptrace
    and container
  output: >
    ptrace 감지 (user=%user.name container=%container.id
    proc=%proc.cmdline)
  priority: WARNING
```

```json
// Seccomp 프로필 (차단)
// /var/lib/kubelet/seccomp/profiles/no-ptrace.json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["ptrace"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

```yaml
# Pod에 적용
apiVersion: v1
kind: Pod
metadata:
  name: no-ptrace-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/no-ptrace.json
  containers:
  - name: app
    image: nginx
```

---

**Q52.** Kubernetes 클러스터에서 TLS 인증서 만료를 관리하는 방법을 설명하시오.

**정답:**
```bash
# 인증서 만료 확인
kubeadm certs check-expiration

# 출력 예시:
# CERTIFICATE          EXPIRES                  RESIDUAL TIME
# admin.conf           Jan 01, 2025 00:00 UTC   350d
# apiserver            Jan 01, 2025 00:00 UTC   350d

# 모든 인증서 갱신 (kubeadm)
kubeadm certs renew all

# 특정 인증서만 갱신
kubeadm certs renew apiserver
kubeadm certs renew apiserver-kubelet-client

# 갱신 후 컨트롤 플레인 재시작
# static Pod는 manifests 파일을 수정하면 자동 재시작
# 또는
systemctl restart kubelet

# cert-manager로 자동 갱신 (애플리케이션 인증서)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
spec:
  renewBefore: 720h  # 만료 30일 전 갱신
```

---

**Q53.** Kubernetes 취약점 CVE-2018-1002105 (Privilege Escalation via aggregated API Server)의 원리와 대응 방법을 설명하시오.

**정답/해설:**
**취약점 원리:**
- aggregated API Server 기능을 통해 승인되지 않은 API 요청 전달 가능
- 낮은 권한의 사용자가 API Server를 통해 높은 권한의 요청 전송 가능

**대응 방법:**
1. Kubernetes 1.11.3+, 1.10.7+, 1.9.11+ 이상으로 업그레이드
2. aggregated API Server 사용하지 않는 경우 비활성화
3. 최소 권한 원칙으로 RBAC 설정
4. `kubectl auth can-i`로 권한 검증

---

**Q54.** 다음 명령어의 보안 위험을 설명하시오.
```bash
kubectl create clusterrolebinding anonymous-admin \
  --clusterrole=cluster-admin \
  --user=system:anonymous
```

**정답/해설:**
이 명령어는 **인증되지 않은 모든 사용자(anonymous)에게 cluster-admin 권한을 부여**합니다. 결과:
- 모든 API 엔드포인트가 인증 없이 접근 가능
- 모든 리소스 생성/수정/삭제 가능
- 클러스터 완전 장악 가능

```bash
# 즉시 제거
kubectl delete clusterrolebinding anonymous-admin

# anonymous 사용자의 모든 권한 확인
kubectl get clusterrolebindings -o yaml | grep -B5 "system:anonymous"
kubectl get rolebindings -A -o yaml | grep -B5 "system:anonymous"
```

---

**Q55.** Kubernetes에서 Secret을 Git에 안전하게 저장하는 Sealed Secrets의 동작 원리를 설명하시오.

**정답/해설:**
```
Sealed Secrets 동작 원리:

1. 컨트롤러 설치: sealed-secrets-controller가 클러스터에 설치됨
2. 키 쌍 생성: 컨트롤러가 RSA 키 쌍 자동 생성 (공개키/개인키)
3. 암호화: kubeseal 도구로 공개키를 사용해 Secret 암호화
   kubeseal --cert=<public-key> < secret.yaml > sealedsecret.yaml
4. Git 저장: SealedSecret YAML을 Git에 저장 (암호화된 상태)
5. 복호화: 컨트롤러만이 개인키로 복호화하여 실제 Secret 생성

장점:
- Git에 암호화된 형태로 저장 가능 (GitOps 워크플로우)
- 특정 네임스페이스/클러스터에서만 복호화 가능
- 컨트롤러 외에는 개인키 접근 불가
```

---

**Q56.** Pod가 실행 중인 노드의 `/etc/passwd` 파일에 접근하려 할 때 이를 방어하는 모든 방법을 설명하시오.

**정답/해설:**
```
방어 레이어:

1. PSS (Pod Security Standards):
   - hostPath 볼륨 마운트 금지 (baseline 이상)
   - privileged 모드 금지

2. AppArmor:
   profile deny-host-passwd {
     deny /host/** r,
   }

3. Seccomp:
   - openat 시스템 콜 필터링 (특정 경로)

4. NetworkPolicy:
   - 외부 데이터 전송 차단

5. OPA Gatekeeper:
   - hostPath 사용 Pod 생성 거부 정책

6. Falco:
   condition: open_read and fd.name="/etc/passwd" and container
   → 실시간 감지

7. ReadOnlyRootFilesystem:
   - 컨테이너 내부 파일 변조 방지

8. RBAC:
   - hostPath 볼륨을 사용하는 Pod 생성 권한 제한
```

---

**Q57.** Kubernetes 1.30+에서 AppArmor를 securityContext로 설정하는 방법과, annotation 방식과의 차이를 설명하시오.

**정답:**
```yaml
# Kubernetes 1.30+ (securityContext 방식)
apiVersion: v1
kind: Pod
metadata:
  name: new-apparmor-pod
spec:
  securityContext:
    appArmorProfile:
      type: RuntimeDefault  # 런타임 기본 프로필
  containers:
  - name: app
    image: nginx
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: my-custom-profile  # 컨테이너별 오버라이드
---
# Kubernetes 1.30 이전 (annotation 방식)
apiVersion: v1
kind: Pod
metadata:
  name: old-apparmor-pod
  annotations:
    # 컨테이너 이름별로 annotation
    container.apparmor.security.beta.kubernetes.io/app: localhost/my-custom-profile
spec:
  containers:
  - name: app
    image: nginx
```

**차이점:**
- 1.30+: `securityContext.appArmorProfile`로 Pod/컨테이너 레벨 모두 설정 가능
- 1.30 이전: 반드시 `annotation`으로만 설정, 컨테이너 이름을 annotation 키에 포함

---

**Q58.** Kubernetes 클러스터에 대한 침투 테스트(Penetment Test) 시나리오를 설명하고, 각 공격 벡터에 대한 방어 방법을 서술하시오.

**정답/해설:**
```
공격 벡터 1: API Server 익명 접근
공격: curl https://k8s-api:6443/api/v1/secrets
방어: --anonymous-auth=false, RBAC 설정

공격 벡터 2: ServiceAccount 토큰 탈취
공격: cat /var/run/secrets/kubernetes.io/serviceaccount/token
방어: automountServiceAccountToken: false, 최소 권한 SA

공격 벡터 3: etcd 직접 접근
공격: etcdctl get /registry/secrets/...
방어: etcd 암호화, 방화벽으로 2379 포트 접근 제한

공격 벡터 4: 컨테이너 탈출
공격: privileged 컨테이너에서 호스트 접근
방어: PSS restricted, seccomp, AppArmor, gVisor

공격 벡터 5: 권한 상승
공격: RBAC 설정 오류 악용
방어: 최소 권한 RBAC, 정기 감사

공격 벡터 6: 이미지 공급망 공격
공격: 악성 이미지 배포
방어: cosign 서명 검증, 프라이빗 레지스트리, Trivy 스캔

공격 벡터 7: 네트워크 측면 이동
공격: 침해된 Pod에서 다른 서비스로 이동
방어: NetworkPolicy default-deny, mTLS
```

---

**Q59.** `kube-apiserver`에서 특정 사용자의 모든 API 요청을 차단하는 방법을 설명하시오. (RBAC 외의 방법)

**정답/해설:**
```bash
# 방법 1: 인증서 취소 (CRL - Certificate Revocation List)
# Kubernetes는 기본적으로 CRL을 지원하지 않음
# → 인증서 재발급 후 기존 인증서 만료까지 기다려야 함

# 방법 2: kubeconfig에서 자격증명 제거
# 중앙 집중식 인증(OIDC, LDAP)을 사용하는 경우 해당 시스템에서 계정 비활성화

# 방법 3: Webhook Authentication
# 외부 인증 서버에서 해당 사용자 거부 응답

# 방법 4: RBAC으로 모든 권한 제거
kubectl delete rolebinding --all --as=username
kubectl delete clusterrolebinding --all (해당 사용자 관련)

# 방법 5: 네트워크 레벨 차단
# 해당 사용자 IP를 방화벽에서 차단

# 실무 권장: OIDC + 중앙 IdP(Identity Provider) 사용
# IdP에서 계정을 비활성화하면 즉시 접근 차단 가능
```

---

**Q60.** Kubernetes에서 `PodDisruptionBudget`과 보안의 관계를 설명하시오.

**정답/해설:**
```
PDB 자체는 가용성 도구이지만 보안과 연관성:

1. DoS 방지: PDB로 최소 가용 Pod 수 보장
   → 악의적 Pod 삭제 시도에도 서비스 유지

2. RBAC와 결합:
   - PDB 삭제 권한을 제한하여 서비스 중단 방지
   - 공격자가 PDB를 삭제 후 Pod를 중단시키는 시나리오 방지

3. 유지보수 창 보안:
   - 노드 드레인 시 PDB가 있어야 안전한 롤링 업데이트 가능
   - 보안 패치 적용 시 다운타임 없이 업그레이드

4. 보안 패치 적용:
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # 최소 2개 Pod 유지
  selector:
    matchLabels:
      app: critical-service
```

---

**Q61.** Kubernetes의 TokenRequest API와 일반 ServiceAccount 토큰의 차이를 설명하시오.

**정답/해설:**
```
기존 ServiceAccount 토큰:
- 만료 없음 (무기한)
- 특정 대상(audience) 없음
- Secret에 저장됨
- 취약점: 탈취 시 영구적으로 사용 가능

TokenRequest API 토큰 (Bound Service Account Token):
- 만료 시간 설정 가능 (expirationSeconds)
- 특정 대상(audience) 바인딩
- 특정 Pod에 바인딩 (Pod 삭제 시 무효화)
- Kubernetes 1.22+에서 기본 사용

확인:
kubectl create token my-sa --duration=1h --audience=my-service

Projected Volume:
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        expirationSeconds: 3600
        audience: my-api
```

---

**Q62.** 다음 시나리오를 완료하시오: "모든 Pod에서 `/etc/shadow` 파일 읽기를 Falco로 감지하고, 동시에 AppArmor로 차단하시오."

**정답:**
```yaml
# Falco 규칙 (감지)
- rule: Read Shadow File in Container
  desc: 컨테이너에서 /etc/shadow 읽기 감지
  condition: >
    open_read
    and container
    and fd.name in (/etc/shadow, /etc/gshadow)
  output: >
    /etc/shadow 읽기 시도 감지
    (user=%user.name container=%container.name
     file=%fd.name proc=%proc.cmdline)
  priority: CRITICAL
```

```
# AppArmor 프로필 (차단)
# /etc/apparmor.d/k8s-deny-shadow
profile k8s-deny-shadow flags=(attach_disconnected) {
  #include <abstractions/base>
  
  file,
  
  # shadow 파일 접근 거부
  deny /etc/shadow r,
  deny /etc/gshadow r,
  deny /etc/shadow- r,
}
```

```bash
# 프로필 로드
apparmor_parser -r /etc/apparmor.d/k8s-deny-shadow
```

```yaml
# Pod에 적용 (Kubernetes 1.30+)
apiVersion: v1
kind: Pod
metadata:
  name: secured-pod
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-deny-shadow
  containers:
  - name: app
    image: nginx
```

---

**Q63.** Kubernetes에서 사용되는 PKI 구조를 설명하고, 각 인증서의 역할을 서술하시오.

**정답/해설:**
```
/etc/kubernetes/pki/ 구조:

루트 CA:
├── ca.crt / ca.key
│   └── kubernetes 클러스터 CA
│       ├── apiserver.crt - API Server 서빙 인증서
│       ├── apiserver-kubelet-client.crt - kubelet 클라이언트 인증서
│       ├── apiserver-etcd-client.crt - etcd 클라이언트 인증서
│       └── admin.crt - 관리자 인증서
│
├── etcd/
│   ├── ca.crt / ca.key - etcd 전용 CA
│   ├── server.crt - etcd 서버 인증서
│   ├── peer.crt - etcd 피어 인증서
│   └── healthcheck-client.crt
│
├── front-proxy-ca.crt - API Server 프록시 CA
├── front-proxy-client.crt - 프록시 클라이언트
└── sa.key / sa.pub - ServiceAccount 서명 키

인증서 확인:
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
kubeadm certs check-expiration
```

---

**Q64.** Kubernetes에서 `ValidatingAdmissionWebhook`과 `MutatingAdmissionWebhook`의 차이와 보안 활용 방법을 설명하시오.

**정답/해설:**
```
MutatingAdmissionWebhook:
- 요청을 수정(변경)할 수 있음
- 실행 순서: 먼저 실행
- 활용: 기본값 주입, 사이드카 컨테이너 자동 추가
  예) Istio sidecar injector, Vault agent injector

ValidatingAdmissionWebhook:
- 요청을 검증만 함 (수정 불가)
- 실행 순서: Mutating 이후 실행
- 활용: 정책 검증, 이미지 서명 확인
  예) OPA Gatekeeper, ImagePolicyWebhook

보안 활용:
1. 이미지 서명 검증 (Cosign Policy Controller)
2. 레이블/어노테이션 강제 적용 (OPA)
3. 보안 설정 자동 주입 (MutatingWebhook)
4. 허용된 레지스트리 검증 (ValidatingWebhook)

실패 정책 (failurePolicy):
- Fail: 웹훅 실패 시 요청 거부 (보안 강화)
- Ignore: 웹훅 실패 시 요청 허용 (가용성 우선)
→ 보안 중요 웹훅은 Fail 설정 권장
```

---

**Q65.** 다음 취약한 Pod를 `restricted` PSS를 준수하도록 수정하시오.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fix-me
spec:
  containers:
  - name: app
    image: ubuntu:latest
    command: ["sleep", "3600"]
```

**정답:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fix-me
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: ubuntu:22.04   # latest 태그 제거
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
```

---

**Q66.** Kubernetes 클러스터에서 보안 감사(Security Audit)를 수행하는 체크리스트를 작성하시오.

**정답/해설:**
```
Kubernetes 보안 감사 체크리스트:

[ ] API Server
  [ ] --anonymous-auth=false
  [ ] --authorization-mode=Node,RBAC
  [ ] --enable-admission-plugins에 NodeRestriction 포함
  [ ] --audit-log-path 설정됨
  [ ] --insecure-port=0
  [ ] --tls-min-version=VersionTLS12

[ ] etcd
  [ ] --client-cert-auth=true
  [ ] --auto-tls=false
  [ ] 방화벽으로 외부 접근 차단
  [ ] EncryptionConfiguration 적용

[ ] kubelet
  [ ] --anonymous-auth=false
  [ ] --authorization-mode=Webhook
  [ ] readOnlyPort=0
  [ ] protectKernelDefaults=true

[ ] RBAC
  [ ] cluster-admin 바인딩 최소화
  [ ] ServiceAccount 최소 권한 확인
  [ ] automountServiceAccountToken: false

[ ] 네트워크
  [ ] NetworkPolicy default-deny 적용
  [ ] 필요한 통신만 허용

[ ] Pod 보안
  [ ] PSS restricted 또는 baseline 적용
  [ ] privileged Pod 없음
  [ ] hostPath 마운트 없음

[ ] 이미지 보안
  [ ] latest 태그 사용 없음
  [ ] 취약점 스캔 완료
  [ ] 프라이빗 레지스트리 사용

[ ] 런타임 보안
  [ ] Falco 실행 중
  [ ] Audit 로깅 활성화
  [ ] 인증서 만료 확인
```

---

**Q67.** 다중 테넌트(Multi-tenant) Kubernetes 환경에서 테넌트 간 격리를 보장하는 방법을 설명하시오.

**정답/해설:**
```
하드 멀티 테넌시 (Hard Multi-tenancy):
- 테넌트별 별도 클러스터 (가장 강한 격리)
- vCluster 사용 (가상 클러스터)

소프트 멀티 테넌시 (Soft Multi-tenancy):
1. 네임스페이스 기반 격리
   - 테넌트별 네임스페이스 할당
   - LimitRange로 리소스 제한
   - ResourceQuota로 총 사용량 제한
   - RBAC로 네임스페이스 간 접근 차단
   
2. 네트워크 격리
   - NetworkPolicy default-deny-all
   - 테넌트 간 통신 차단

3. 런타임 격리
   - RuntimeClass (gVisor) 적용
   - 테넌트별 노드 그룹 (nodeSelector/taints)

4. 저장소 격리
   - 테넌트별 StorageClass
   - 볼륨 암호화

5. 정책 격리
   - OPA Gatekeeper로 테넌트별 정책
   - PSS restricted 적용

6. 관측 격리
   - 테넌트별 로그/메트릭 접근 제한
   - Audit 로그 분리
```

---

**Q68.** Kubernetes에서 RBAC escalation 취약점을 설명하고, 방어 방법을 서술하시오.

**정답/해설:**
```
RBAC Escalation 취약점:

1. RoleBinding 생성 권한을 통한 권한 상승
   - rolebindings에 create 권한이 있으면 
     자신이 가진 Role을 다른 사람에게 부여 가능
   - 방어: rolebindings 생성 권한 최소화

2. ClusterRole + RoleBinding 조합
   - ClusterRole을 RoleBinding으로 바인딩 가능
   - 방어: ClusterRole 사용 제한

3. ServiceAccount 토큰 탈취
   - 높은 권한의 SA 토큰으로 API 호출
   - 방어: automountServiceAccountToken: false

4. Impersonation 권한
   - users를 impersonate 할 수 있는 권한이 있으면 
     다른 사용자로 API 호출 가능
   - 방어: impersonate verb 권한 최소화

5. kubectl 플러그인을 통한 권한 우회
   - 방어: 신뢰된 플러그인만 사용

감사 명령:
# 위험한 권한 조합 탐지
kubectl get clusterroles -o yaml | \
  grep -A5 "escalate\|bind\|impersonate"
```

---

**Q69.** Kubernetes 클러스터에서 이미지 취약점 스캔을 자동화하는 파이프라인을 설계하시오.

**정답/해설:**
```
자동화 파이프라인 설계:

1. CI 단계 (빌드 시):
   - Trivy로 이미지 스캔
   - CRITICAL 취약점 발견 시 빌드 실패
   
   # GitHub Actions 예시
   - name: Scan image
     uses: aquasecurity/trivy-action@main
     with:
       image-ref: ${{ env.IMAGE }}
       format: 'sarif'
       exit-code: '1'
       severity: 'CRITICAL,HIGH'

2. 레지스트리 단계:
   - Harbor: 내장 취약점 스캐너
   - ECR: Inspector 연동
   - 취약한 이미지 자동 격리

3. 배포 전 단계 (Admission):
   - ImagePolicyWebhook으로 스캔 결과 확인
   - Cosign으로 서명된 이미지만 허용

4. 런타임 단계:
   - Falco로 런타임 이상 감지
   - Trivy Operator: 클러스터 내 이미지 지속 스캔

5. 정기 스캔:
   - CronJob으로 실행 중인 이미지 스캔
   - Starboard/Trivy Operator 사용
   
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: image-scan
   spec:
     schedule: "0 2 * * *"  # 매일 새벽 2시
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: trivy
               image: aquasec/trivy
               command: ["trivy", "image", "--exit-code", "0", 
                        "production-image:latest"]
```

---

**Q70.** Kubernetes 1.32 기준 최신 보안 기능과 변경사항을 설명하시오.

**정답/해설:**
```
Kubernetes 1.32 주요 보안 관련 변경사항:

1. Pod Security Standards (Stable):
   - PSS가 GA로 안정화
   - PodSecurityPolicy 완전 제거 (1.25에서 제거됨)

2. AppArmor (GA in 1.30):
   - securityContext.appArmorProfile이 안정화
   - annotation 방식 deprecated

3. Encrypted Secrets (개선):
   - KMS v2 provider 안정화
   - 외부 KMS와의 통합 개선

4. Sigstore 통합:
   - 이미지 서명 검증 생태계 성숙

5. Node Authorization 개선:
   - NodeRestriction 플러그인 강화

6. CEL (Common Expression Language):
   - ValidatingAdmissionPolicy에 CEL 사용
   - 웹훅 없이 정책 검증 가능

7. ValidatingAdmissionPolicy (GA in 1.30):
   apiVersion: admissionregistration.k8s.io/v1
   kind: ValidatingAdmissionPolicy
   metadata:
     name: no-latest-tag
   spec:
     failurePolicy: Fail
     matchConstraints:
       resourceRules:
       - apiGroups: [""]
         apiVersions: ["v1"]
         operations: ["CREATE", "UPDATE"]
         resources: ["pods"]
     validations:
     - expression: >
         object.spec.containers.all(c, 
           !c.image.endsWith(':latest'))
       message: "latest 태그 사용 금지"
```

---

# 📌 시험 합격 전략

## 핵심 파일 위치 암기

| 파일/경로 | 용도 |
|----------|------|
| `/etc/kubernetes/manifests/kube-apiserver.yaml` | API Server 설정 |
| `/etc/kubernetes/manifests/etcd.yaml` | etcd 설정 |
| `/var/lib/kubelet/config.yaml` | kubelet 설정 |
| `/etc/kubernetes/audit-policy.yaml` | Audit Policy |
| `/etc/kubernetes/encryption-config.yaml` | 암호화 설정 |
| `/etc/apparmor.d/` | AppArmor 프로필 |
| `/var/lib/kubelet/seccomp/` | Seccomp 프로필 |
| `/var/log/kubernetes/audit.log` | Audit 로그 |

## 필수 명령어 정리

```bash
# RBAC 검증
kubectl auth can-i --list --as <user>
kubectl auth can-i <verb> <resource> --as <user>

# 인증서 관리
kubeadm certs check-expiration
kubeadm certs renew all

# 보안 스캔
kube-bench run --targets master
trivy image --severity HIGH,CRITICAL <image>
kubesec scan <yaml-file>

# AppArmor
aa-status
apparmor_parser -r <profile>

# Falco
systemctl status falco
journalctl -fu falco

# etcd
ETCDCTL_API=3 etcdctl --endpoints=... get /registry/secrets/...

# 네임스페이스 보안 레이블
kubectl label namespace <ns> pod-security.kubernetes.io/enforce=restricted
```

## 시험 팁

1. **시간 관리**: 각 문제에 배점이 다름. 쉬운 문제 먼저 풀고 어려운 문제는 나중에
2. **kubectl explain 활용**: 필드명 모를 때 `kubectl explain pod.spec.securityContext`
3. **공식 문서 활용**: kubernetes.io 북마크 숙지 (시험 중 사용 가능)
4. **작업 컨텍스트 확인**: 매 문제 시작 전 `kubectl config use-context` 확인
5. **YAML 검증**: `kubectl apply --dry-run=client -f file.yaml`로 사전 검증
6. **자동 완성 설정**: `source <(kubectl completion bash)` 항상 설정
```

---
*작성일: 2025년 | Kubernetes v1.32 기준 | CKS 최신 시험 도메인 반영*
