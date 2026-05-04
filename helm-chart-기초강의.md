# 🚢 Kubernetes Helm Chart 완전 정복 강의

> **대상**: Kubernetes를 이제 막 시작했거나, kubectl은 써봤지만 Helm은 처음인 분들
> **목표**: Helm의 핵심 개념을 이해하고, 직접 Chart를 만들고 배포할 수 있게 되는 것
> **소요 시간**: 약 60~90분 (실습 포함)

---

## 📚 목차

1. [왜 Helm이 필요한가?](#1-왜-helm이-필요한가)
2. [Helm이란 무엇인가?](#2-helm이란-무엇인가)
3. [Helm 핵심 3대 개념](#3-helm-핵심-3대-개념)
4. [Helm 설치하기](#4-helm-설치하기)
5. [Helm Chart 구조 파헤치기](#5-helm-chart-구조-파헤치기)
6. [Helm 필수 명령어](#6-helm-필수-명령어)
7. [실전 1: 공개 Chart로 Nginx 배포하기](#7-실전-1-공개-chart로-nginx-배포하기)
8. [실전 2: 나만의 Chart 만들기](#8-실전-2-나만의-chart-만들기)
9. [Templating(템플릿) 마스터하기](#9-templating템플릿-마스터하기)
10. [Values 파일 활용 전략](#10-values-파일-활용-전략)
11. [환경별 배포 (dev / staging / prod)](#11-환경별-배포-dev--staging--prod)
12. [LLMOps/MLOps에서의 Helm 활용](#12-llmopsmlops에서의-helm-활용)
13. [자주 하는 실수와 베스트 프랙티스](#13-자주-하는-실수와-베스트-프랙티스)
14. [요약 및 다음 단계](#14-요약-및-다음-단계)

---

## 1. 왜 Helm이 필요한가?

### 🤔 Helm 없이 살아본다고 가정해봅시다

여러분이 회사에서 간단한 웹 애플리케이션을 Kubernetes에 배포한다고 해봅시다. 필요한 리소스가 무엇일까요?

- `Deployment` (앱을 실행할 파드 정의)
- `Service` (외부에서 접근할 수 있게 해주는 네트워크)
- `ConfigMap` (설정 파일)
- `Secret` (비밀번호, API 키 같은 민감 정보)
- `Ingress` (도메인 라우팅)
- `HorizontalPodAutoscaler` (자동 확장)

그러면 YAML 파일이 6개 이상 생깁니다. 이걸 일일이 `kubectl apply -f`로 배포해야 하죠.

### 🔥 그런데 문제가 생기기 시작합니다

**문제 1. 환경별로 값이 다르다**

개발(dev), 스테이징(staging), 운영(prod) 환경마다 이미지 태그, 레플리카 수, 리소스 할당량 등이 다릅니다.

```yaml
# dev 환경
replicas: 1
image: myapp:dev-latest
resources:
  limits:
    memory: "256Mi"

# prod 환경
replicas: 5
image: myapp:v1.2.3
resources:
  limits:
    memory: "2Gi"
```

→ 환경마다 YAML 파일을 복붙해서 만들면? **유지보수 지옥** 🥲

**문제 2. 버전 관리가 어렵다**

"어제 배포했던 버전으로 롤백해주세요!" 라고 하면, 어제 적용했던 YAML 파일들을 다 찾아서 다시 apply해야 합니다.

**문제 3. 다른 사람의 앱을 설치하기 어렵다**

Prometheus, Grafana, Redis 같은 오픈소스를 설치하려면 수십 개의 YAML 파일을 직접 작성하거나 받아야 합니다.

**문제 4. 패키징 표준이 없다**

"이 앱 좀 설치해주세요" 했을 때 어떻게 전달해야 할까요? 압축 파일? Git? 표준이 없습니다.

---

### 💡 그래서 Helm이 등장합니다

Helm은 **Kubernetes의 패키지 매니저**입니다. 위의 모든 문제를 한 번에 해결해줍니다.

| 비유 | 설명 |
|------|------|
| 🍎 `apt` (Ubuntu) | 리눅스 앱 설치 도구 |
| 🍺 `brew` (Mac) | macOS 앱 설치 도구 |
| 🐍 `pip` (Python) | Python 패키지 설치 도구 |
| ⎈ **`Helm`** (K8s) | **Kubernetes 앱 설치 도구** |

> 💬 **한 줄 요약**: kubectl이 "수동으로 하나하나 명령 내리는 도구"라면, Helm은 "패키지 단위로 한 번에 설치/관리하는 도구"입니다.

---

## 2. Helm이란 무엇인가?

### 📖 정의

**Helm**은 Kubernetes 애플리케이션을 정의(define)하고, 설치(install)하고, 업그레이드(upgrade)하기 위한 도구입니다.

### 🎯 Helm이 해주는 일

1. **패키징**: 여러 K8s YAML을 하나의 묶음(Chart)으로 만들기
2. **템플릿화**: 환경별로 값만 바꿔서 재사용 가능하게 하기
3. **버전 관리**: 배포 이력을 자동으로 추적
4. **롤백**: 한 줄 명령어로 이전 버전으로 되돌리기
5. **공유**: Repository를 통해 팀/커뮤니티와 공유

### 🏗️ Helm의 동작 방식 (간단 그림)

```
                      ┌─────────────────┐
                      │  Helm Chart     │
                      │  (템플릿 묶음)   │
                      └────────┬────────┘
                               │
                               │ + values.yaml (환경별 값)
                               ▼
                      ┌─────────────────┐
                      │  Helm Engine    │
                      │  (템플릿 렌더링) │
                      └────────┬────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │ 완성된 K8s YAML 매니페스트  │
                  └────────────┬───────────┘
                               │
                               │ kubectl apply (Helm이 대신 해줌)
                               ▼
                      ┌─────────────────┐
                      │  Kubernetes     │
                      │   Cluster       │
                      └─────────────────┘
```

---

## 3. Helm 핵심 3대 개념

Helm을 이해하려면 딱 3개의 개념만 알면 됩니다.

### 🍱 3-1. Chart (차트)

**Chart는 Kubernetes 애플리케이션의 패키지입니다.**

쉽게 말해, "도시락 세트"입니다. 도시락 안에는 밥, 반찬, 국이 들어있죠? Chart 안에는 Deployment, Service, ConfigMap 같은 K8s 리소스 템플릿들이 들어있습니다.

```
my-chart/
├── Chart.yaml          # 도시락 라벨 (이름, 버전 등)
├── values.yaml         # 양념 농도 조절 (기본 설정값)
└── templates/          # 실제 도시락 내용물
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

### 🚀 3-2. Release (릴리스)

**Release는 클러스터에 설치된 Chart의 인스턴스입니다.**

같은 Chart를 여러 번 설치할 수 있습니다. 각 설치본을 Release라고 부릅니다.

예시:
- `nginx` Chart를 설치할 때 이름을 `web-frontend`로 → Release #1
- 같은 `nginx` Chart를 또 설치하면서 이름을 `api-gateway`로 → Release #2

> 💡 **비유**: Chart가 "라면 봉지(설계도)"라면, Release는 "실제 끓여진 라면 한 그릇(실체)"입니다. 같은 라면 봉지로 여러 그릇을 끓일 수 있죠.

### 📦 3-3. Repository (저장소)

**Repository는 Chart들이 모여있는 공간(서버)입니다.**

GitHub의 `npm registry`나 Docker의 `Docker Hub`처럼, Chart를 모아두는 인터넷상의 창고입니다.

대표적인 공개 Repository:
- **Bitnami**: https://charts.bitnami.com/bitnami (가장 유명, 신뢰도 높음)
- **Artifact Hub**: https://artifacthub.io (검색 사이트)
- **공식 K8s**: https://kubernetes.github.io/ingress-nginx 등

---

### 🔁 세 개념의 관계

```
Repository (창고)  ─────►  Chart (도시락 설계도)  ─────►  Release (실제로 만든 도시락)
   "어디서?"                    "무엇을?"                       "어디에, 어떤 이름으로?"
```

---

## 4. Helm 설치하기

### 🪟 설치 방법

**macOS**
```bash
brew install helm
```

**Linux (Ubuntu/Debian)**
```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

**Windows (Chocolatey)**
```powershell
choco install kubernetes-helm
```

**범용 (스크립트)**
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### ✅ 설치 확인

```bash
helm version
```

출력 예시:
```
version.BuildInfo{Version:"v3.14.0", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.5"}
```

> ⚠️ **주의**: Helm 2와 Helm 3는 다릅니다. 반드시 **Helm 3 이상**을 사용하세요. Helm 3는 Tiller(서버 컴포넌트)가 없어져서 보안과 사용성이 크게 개선되었습니다.

---

## 5. Helm Chart 구조 파헤치기

`helm create my-app` 명령어를 실행하면 다음과 같은 폴더 구조가 만들어집니다.

```
my-app/
├── Chart.yaml              # 📋 Chart 메타데이터 (이름, 버전, 설명)
├── values.yaml             # ⚙️ 기본 설정값
├── charts/                 # 📚 의존하는 다른 Chart들 (서브 Chart)
├── templates/              # 📁 K8s 매니페스트 템플릿들
│   ├── _helpers.tpl        #   🔧 재사용 가능한 템플릿 함수
│   ├── deployment.yaml     #   🏃 Deployment 템플릿
│   ├── service.yaml        #   🌐 Service 템플릿
│   ├── ingress.yaml        #   🚪 Ingress 템플릿
│   ├── serviceaccount.yaml #   👤 ServiceAccount 템플릿
│   ├── hpa.yaml            #   📈 HPA 템플릿
│   ├── NOTES.txt           #   📝 설치 후 안내 메시지
│   └── tests/              #   🧪 테스트
└── .helmignore             # 🙈 패키징 시 제외할 파일 (Git의 .gitignore와 비슷)
```

### 📋 Chart.yaml 자세히 보기

```yaml
apiVersion: v2                  # Helm 3는 v2 (Helm 2는 v1)
name: my-app                    # Chart 이름
description: A Helm chart for my application
type: application               # application 또는 library
version: 0.1.0                  # Chart 자체의 버전 (SemVer)
appVersion: "1.16.0"            # 앱의 버전 (참고용)

dependencies:                   # 다른 Chart 의존성 (선택)
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

> 💡 **`version` vs `appVersion` 헷갈림 주의!**
> - `version`: Chart 패키지 자체의 버전 (도시락 포장지의 버전)
> - `appVersion`: 안에 든 앱의 버전 (도시락 안 음식의 버전)

### ⚙️ values.yaml 자세히 보기

```yaml
replicaCount: 1                 # 파드 개수

image:
  repository: nginx             # 이미지 저장소
  pullPolicy: IfNotPresent      # 이미지 가져오기 정책
  tag: "1.25.0"                 # 이미지 태그

service:
  type: ClusterIP               # Service 타입
  port: 80                      # 포트

ingress:
  enabled: false                # Ingress 사용 여부
  className: ""
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific

resources:                      # 리소스 제한 (비어있으면 무제한)
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

---

## 6. Helm 필수 명령어

실무에서 99% 쓰이는 명령어들입니다. 이것만 외워도 됩니다.

### 🔍 검색 & 정보

```bash
# Repository 추가
helm repo add bitnami https://charts.bitnami.com/bitnami

# Repository 목록 확인
helm repo list

# Repository 업데이트 (apt update와 비슷)
helm repo update

# Chart 검색
helm search repo nginx
helm search hub nginx          # Artifact Hub에서 검색

# Chart 정보 보기
helm show chart bitnami/nginx
helm show values bitnami/nginx # 사용 가능한 values.yaml 확인
helm show all bitnami/nginx
```

### 🚀 설치 & 관리

```bash
# 설치 (가장 기본)
helm install <릴리스이름> <차트이름>
helm install my-nginx bitnami/nginx

# 값을 오버라이드해서 설치
helm install my-nginx bitnami/nginx --set replicaCount=3
helm install my-nginx bitnami/nginx -f my-values.yaml

# 네임스페이스 지정
helm install my-nginx bitnami/nginx -n my-namespace --create-namespace

# 설치된 Release 목록
helm list                       # 현재 네임스페이스
helm list -A                    # 모든 네임스페이스
helm list -n my-namespace       # 특정 네임스페이스

# Release 상태 확인
helm status my-nginx

# 적용된 values 확인
helm get values my-nginx

# 렌더링된 매니페스트 확인
helm get manifest my-nginx
```

### 🔄 업그레이드 & 롤백

```bash
# 업그레이드
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# 설치 또는 업그레이드 (없으면 설치, 있으면 업그레이드)
helm upgrade --install my-nginx bitnami/nginx -f values.yaml

# 변경 이력 보기
helm history my-nginx

# 롤백
helm rollback my-nginx 1        # 1번 리비전으로
helm rollback my-nginx          # 이전 리비전으로
```

### 🗑️ 제거

```bash
# Release 삭제
helm uninstall my-nginx

# 삭제하지만 이력은 남기기
helm uninstall my-nginx --keep-history
```

### 🛠️ 개발 & 디버깅

```bash
# Chart 만들기 (기본 템플릿 생성)
helm create my-app

# Chart 문법 검증
helm lint ./my-app

# 템플릿 렌더링 결과만 보기 (실제 설치 X)
helm template my-app ./my-app
helm template my-app ./my-app -f values-prod.yaml

# 설치 시뮬레이션 (dry-run)
helm install my-app ./my-app --dry-run --debug

# Chart 패키징 (.tgz 파일 생성)
helm package ./my-app
```

---

## 7. 실전 1: 공개 Chart로 Nginx 배포하기

이론은 충분합니다. 이제 직접 해봅시다!

### Step 1. Bitnami Repository 추가

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Step 2. Nginx Chart 검색

```bash
helm search repo bitnami/nginx
```

### Step 3. 설정값 확인

```bash
helm show values bitnami/nginx > nginx-default-values.yaml
```

이 파일을 열어보면 어떤 값들을 바꿀 수 있는지 한눈에 보입니다.

### Step 4. 커스텀 values 파일 작성

`my-nginx-values.yaml`:
```yaml
replicaCount: 2

service:
  type: ClusterIP
  ports:
    http: 8080

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### Step 5. 설치!

```bash
helm install my-nginx bitnami/nginx \
  -f my-nginx-values.yaml \
  -n web --create-namespace
```

### Step 6. 확인

```bash
helm list -n web
kubectl get all -n web
```

### Step 7. 업그레이드 (replica 늘리기)

```bash
helm upgrade my-nginx bitnami/nginx \
  -f my-nginx-values.yaml \
  --set replicaCount=5 \
  -n web
```

### Step 8. 롤백

```bash
helm history my-nginx -n web
helm rollback my-nginx 1 -n web
```

### Step 9. 정리

```bash
helm uninstall my-nginx -n web
```

🎉 축하합니다! Helm으로 첫 배포를 완료했습니다.

---

## 8. 실전 2: 나만의 Chart 만들기

### Step 1. Chart 생성

```bash
helm create my-webapp
cd my-webapp
```

### Step 2. 구조 살펴보기

```bash
tree .
```

### Step 3. values.yaml 수정

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25-alpine"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

# 커스텀 값 추가
appConfig:
  greeting: "Hello, Helm!"
  logLevel: "info"
```

### Step 4. 템플릿 렌더링 확인

```bash
helm template my-webapp . --debug
```

### Step 5. Lint 체크

```bash
helm lint .
```

### Step 6. 설치

```bash
helm install my-webapp . -n test --create-namespace
```

### Step 7. 패키징

```bash
helm package .
# my-webapp-0.1.0.tgz 파일이 생성됨
```

이 `.tgz` 파일을 팀원에게 전달하면, 그 사람은 `helm install <name> my-webapp-0.1.0.tgz`로 똑같이 설치할 수 있습니다.

---

## 9. Templating(템플릿) 마스터하기

Helm의 진짜 강력함은 **템플릿**에 있습니다. Go 템플릿 문법(`{{ }}`)을 사용합니다.

### 🎨 9-1. 기본 변수 치환

`templates/deployment.yaml` 일부:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-webapp     # Release 이름 사용
  labels:
    app: {{ .Chart.Name }}             # Chart 이름 사용
spec:
  replicas: {{ .Values.replicaCount }} # values.yaml의 replicaCount
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 🔑 주요 내장 객체

| 객체 | 설명 | 예시 |
|------|------|------|
| `.Release.Name` | 릴리스 이름 | `my-nginx` |
| `.Release.Namespace` | 네임스페이스 | `default` |
| `.Chart.Name` | Chart 이름 | `nginx` |
| `.Chart.Version` | Chart 버전 | `0.1.0` |
| `.Values.xxx` | values.yaml의 값 | `.Values.image.tag` |
| `.Files.Get` | 파일 내용 읽기 | `.Files.Get "config.txt"` |

### 🔀 9-2. 조건문 (if/else)

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
    {{- end }}
{{- end }}
```

### 🔁 9-3. 반복문 (range)

```yaml
env:
  {{- range $key, $value := .Values.env }}
  - name: {{ $key }}
    value: {{ $value | quote }}
  {{- end }}
```

values.yaml:
```yaml
env:
  LOG_LEVEL: info
  ENVIRONMENT: production
```

### 🎯 9-4. 기본값 처리 (default)

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}"
```

`tag`가 없으면 `"latest"`를 사용합니다.

### 🛠️ 9-5. 파이프 함수 활용

```yaml
name: {{ .Values.appName | upper }}              # 대문자
config: {{ .Values.config | toYaml | indent 4 }} # YAML로 변환 후 들여쓰기
secret: {{ .Values.password | b64enc }}          # Base64 인코딩
```

### 🧰 9-6. _helpers.tpl로 코드 재사용

`templates/_helpers.tpl`:
```yaml
{{/* 공통 라벨 정의 */}}
{{- define "my-webapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

다른 템플릿에서 사용:
```yaml
metadata:
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
```

### ⚠️ 9-7. 공백 제어 (Whitespace Control)

- `{{-` : 앞쪽 공백 제거
- `-}}` : 뒤쪽 공백 제거

```yaml
# 안 좋은 예 (빈 줄이 생김)
{{ if .Values.enabled }}
foo: bar
{{ end }}

# 좋은 예
{{- if .Values.enabled }}
foo: bar
{{- end }}
```

---

## 10. Values 파일 활용 전략

### 📂 우선순위 (높은 순)

```
1. --set 옵션 (CLI에서 직접)        ← 가장 강력
2. -f / --values 파일                
3. Chart의 values.yaml               ← 기본값
```

### 🎛️ --set 사용법

```bash
helm install my-app ./chart \
  --set image.tag=v2.0 \
  --set replicaCount=5 \
  --set env.LOG_LEVEL=debug
```

배열의 경우:
```bash
--set servers={server1,server2,server3}
```

복잡한 값은 `--set-string`, `--set-file`도 활용 가능.

### 📄 -f 옵션 (여러 파일 합치기)

```bash
helm install my-app ./chart \
  -f values.yaml \
  -f values-prod.yaml \
  -f values-secrets.yaml
```

뒤에 오는 파일이 앞 파일을 덮어씁니다. 공통 설정 + 환경별 설정 + 시크릿 등으로 분리할 수 있습니다.

---

## 11. 환경별 배포 (dev / staging / prod)

### 📁 파일 구조

```
my-app/
├── Chart.yaml
├── values.yaml              # 공통 기본값
├── values-dev.yaml          # 개발 환경
├── values-staging.yaml      # 스테이징 환경
├── values-prod.yaml         # 운영 환경
└── templates/
```

### 📝 values-dev.yaml

```yaml
replicaCount: 1
image:
  tag: "dev-latest"
resources:
  limits:
    cpu: 100m
    memory: 128Mi
ingress:
  enabled: true
  hosts:
    - host: dev.myapp.com
```

### 📝 values-prod.yaml

```yaml
replicaCount: 5
image:
  tag: "v1.2.3"
resources:
  limits:
    cpu: 1000m
    memory: 2Gi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
ingress:
  enabled: true
  hosts:
    - host: myapp.com
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.com
```

### 🚀 배포 명령어

```bash
# 개발 환경
helm upgrade --install my-app ./my-app \
  -f values.yaml \
  -f values-dev.yaml \
  -n dev --create-namespace

# 운영 환경
helm upgrade --install my-app ./my-app \
  -f values.yaml \
  -f values-prod.yaml \
  -n prod --create-namespace
```

> 💡 **꿀팁**: `helm upgrade --install`는 "있으면 업그레이드, 없으면 설치"라서 CI/CD 파이프라인에서 자주 사용됩니다.

---

## 12. LLMOps/MLOps에서의 Helm 활용

> 🎓 LLMOps, MLOps, Cloud Engineer를 지향하는 분들을 위한 실전 활용 사례입니다.

### 🤖 12-1. 자주 쓰이는 Helm Chart들

ML/AI 인프라 구축 시 Helm은 거의 필수입니다.

| 도구 | 용도 | Chart |
|------|------|-------|
| **Kubeflow** | ML 파이프라인 플랫폼 | `kubeflow/kubeflow` |
| **KServe** | 모델 서빙 | `kserve/kserve` |
| **MLflow** | 실험 추적, 모델 레지스트리 | `community-charts/mlflow` |
| **Argo Workflows** | 워크플로우 오케스트레이션 | `argo/argo-workflows` |
| **Seldon Core** | 모델 서빙 | `seldonio/seldon-core-operator` |
| **Ray** | 분산 학습/추론 | `kuberay/kuberay-operator` |
| **vLLM** | LLM 추론 서빙 | 커스텀 Chart |
| **Prometheus + Grafana** | 모니터링 | `prometheus-community/kube-prometheus-stack` |
| **NVIDIA GPU Operator** | GPU 관리 | `nvidia/gpu-operator` |

### 🚀 12-2. 실전 예시: vLLM 모델 서빙 Chart

LLM 추론 서버를 Helm으로 패키징한 예시:

```yaml
# values.yaml
model:
  name: "meta-llama/Llama-3-8B-Instruct"
  dtype: "bfloat16"
  maxModelLen: 4096

image:
  repository: vllm/vllm-openai
  tag: "latest"

resources:
  limits:
    nvidia.com/gpu: 1
    memory: "32Gi"
  requests:
    nvidia.com/gpu: 1
    memory: "16Gi"

nodeSelector:
  accelerator: nvidia-a100

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 4
  targetGPUUtilization: 80
```

```yaml
# templates/deployment.yaml (일부)
spec:
  containers:
    - name: vllm
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      args:
        - "--model"
        - "{{ .Values.model.name }}"
        - "--dtype"
        - "{{ .Values.model.dtype }}"
        - "--max-model-len"
        - "{{ .Values.model.maxModelLen | quote }}"
      resources:
        {{- toYaml .Values.resources | nindent 8 }}
```

이렇게 하면, **모델만 다른 여러 LLM 서비스를 같은 Chart로 손쉽게 배포** 가능합니다.

```bash
# Llama 3 8B 배포
helm install llama3-8b ./vllm-chart --set model.name=meta-llama/Llama-3-8B-Instruct

# Mistral 7B 배포  
helm install mistral-7b ./vllm-chart --set model.name=mistralai/Mistral-7B-v0.1
```

### 📊 12-3. 모니터링 스택 한 번에 설치

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

이 한 줄로 Prometheus, Grafana, Alertmanager, 각종 Exporter가 모두 설치됩니다. ML 워크로드의 GPU/메모리/지연시간 모니터링을 즉시 시작할 수 있습니다.

### 🎯 12-4. ArgoCD + Helm = GitOps

운영 환경에서는 보통 **GitOps** 패턴을 사용합니다:

```
Git Repo (Helm Chart + values)
        │
        │ (자동 동기화)
        ▼
    ArgoCD
        │
        ▼
  Kubernetes Cluster
```

ArgoCD는 Helm을 네이티브 지원하므로, Git에 Helm Chart와 values를 저장해두면 자동으로 클러스터에 배포됩니다.

---

## 13. 자주 하는 실수와 베스트 프랙티스

### ❌ 자주 하는 실수

1. **`helm install` 후 `kubectl edit`로 직접 수정**
   - → Helm이 추적 못 함. 다음 `helm upgrade` 시 덮어쓰기 됨.
   - ✅ 항상 `values.yaml`을 수정 후 `helm upgrade`

2. **values.yaml을 Git에 커밋하면서 비밀번호 포함**
   - → 보안 사고
   - ✅ 시크릿은 `Sealed Secrets`, `External Secrets`, `SOPS` 같은 도구 사용

3. **Chart 버전 업데이트 안 함**
   - → 변경사항이 있는데 `version`을 그대로 두면 캐시 문제 발생
   - ✅ `Chart.yaml`의 `version`을 SemVer로 꼬박꼬박 올리기

4. **template에서 들여쓰기 잘못 쓰기**
   - → 렌더링 결과가 깨짐
   - ✅ `nindent`(앞에 줄바꿈 + 들여쓰기) vs `indent`(들여쓰기만) 구분

5. **`--dry-run` 안 써보고 바로 prod 배포**
   - ✅ 배포 전 항상 `helm template` 또는 `helm install --dry-run --debug`

### ✅ 베스트 프랙티스

1. **항상 `helm upgrade --install` 사용 (멱등성)**
2. **모든 값에 합리적인 기본값 (default) 제공**
3. **`helm lint`를 CI 파이프라인에 포함**
4. **`NOTES.txt`에 사용법 안내 작성** (사용자 친화적)
5. **`README.md`에 values 설명 정리**
6. **버전 관리는 SemVer 준수** (Major.Minor.Patch)
7. **시크릿은 Chart 외부에서 관리**
8. **`helm test`로 설치 후 헬스체크 자동화**
9. **공식/유명 Chart는 굳이 다시 만들지 말고 Bitnami 등 활용**
10. **운영 환경은 GitOps (ArgoCD/Flux) + Helm 조합으로**

---

## 14. 요약 및 다음 단계

### 🎯 오늘 배운 것 한 페이지 요약

```
┌────────────────────────────────────────────────────────────────┐
│                    🚢 Helm = K8s 패키지 매니저                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  핵심 3대 개념:                                                │
│    🍱 Chart      = 패키지 (템플릿 묶음)                        │
│    🚀 Release    = 클러스터에 설치된 Chart 인스턴스            │
│    📦 Repository = Chart를 보관하는 저장소                     │
│                                                                │
│  핵심 명령어 5개:                                              │
│    helm install <name> <chart>      # 설치                     │
│    helm upgrade --install ...       # 설치 또는 업그레이드     │
│    helm rollback <name> <revision>  # 롤백                     │
│    helm list -A                     # 목록                     │
│    helm uninstall <name>            # 제거                     │
│                                                                │
│  핵심 워크플로우:                                              │
│    helm create → 수정 → helm lint → helm template →            │
│    helm install --dry-run → helm install → helm upgrade        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 🚀 다음에 공부하면 좋을 주제

1. **Helm 라이브러리 Chart**: 공통 템플릿을 라이브러리로 만들어 재사용
2. **Helm Hooks**: 설치 전/후, 업그레이드 전/후 작업 정의
3. **Helm Tests**: `helm test` 명령어로 설치 후 자동 검증
4. **Helmfile**: 여러 Helm Release를 선언적으로 관리하는 도구
5. **Helm Plugin**: `helm-diff`, `helm-secrets` 등 유용한 플러그인
6. **OCI Registry**: Helm 3.8+에서 컨테이너 레지스트리에 Chart 저장
7. **ArgoCD + Helm**: GitOps로 운영 자동화
8. **Kustomize vs Helm**: 두 도구의 차이와 조합 사용법

### 📚 추천 자료

- 📖 공식 문서: https://helm.sh/docs/
- 🔍 Chart 검색: https://artifacthub.io/
- 📦 Bitnami Charts: https://github.com/bitnami/charts
- 🎓 Helm Best Practices: https://helm.sh/docs/chart_best_practices/

---

## 🎉 마치며

Helm은 처음에는 복잡해 보이지만, **Chart / Release / Repository** 이 3가지 개념과 **install / upgrade / rollback** 이 3가지 명령어만 익히면 80%는 끝입니다.

나머지는 **실전에서 마주치며 천천히 배워나가면 됩니다.** 처음부터 모든 템플릿 문법을 외울 필요는 없습니다. 필요할 때마다 공식 문서를 참고하면 됩니다.

특히 LLMOps/MLOps 분야에서는 Kubeflow, KServe, MLflow, vLLM 등 거의 모든 도구가 Helm Chart로 배포됩니다. **Helm은 선택이 아니라 필수**입니다.

오늘부터 한 번씩 직접 손으로 Chart를 만들어보세요. 처음에는 어색해도, 한 달만 지나면 손에 익을 것입니다. 💪

> _"The best way to learn Helm is to break things with it."_  
> — 익명의 K8s 엔지니어

Happy Helming! ⎈
