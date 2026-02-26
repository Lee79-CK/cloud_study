# 고급 CI/CD 엔지니어링 완전 가이드
## GitHub Actions · Helm Chart · ArgoCD · AKS · AWS CI/CD
### Senior Cloud Engineer를 위한 박사급 기술 레퍼런스

---

# PART 1: CI/CD 아키텍처의 철학적 기반

## 1.1 파이프라인은 상태 기계다

많은 엔지니어들이 CI/CD 파이프라인을 "자동화된 스크립트 모음"으로 이해하지만, 이는 초급 수준의 인식입니다. 박사급 관점에서 파이프라인은 **부수 효과(side effect)를 동반한 유한 상태 기계(Finite State Machine)**입니다. 각 스테이지는 상태 전환이며, 전환 실패 시의 보상 트랜잭션(compensating transaction)까지 설계해야 합니다.

이를 더 깊이 이해하려면 분산 시스템의 **CAP 정리**를 CI/CD에 대입하는 사고 실험이 도움이 됩니다. 파이프라인이 실행 중 네트워크 파티션이 발생했을 때, 일관성(Consistency)과 가용성(Availability) 중 무엇을 선택할 것인가? 이 질문에 답하는 방식이 파이프라인 설계의 근본을 결정합니다.

## 1.2 GitOps의 이론적 토대: 선언적 수렴(Declarative Convergence)

GitOps는 단순한 "Git으로 배포 관리하기"가 아닙니다. 이것은 제어 이론(Control Theory)의 **폐쇄 루프 피드백 시스템(Closed-Loop Feedback System)**을 인프라에 적용한 개념입니다. 목표 상태(desired state)와 현재 상태(observed state)의 차이를 지속적으로 계산하고, 그 차이를 0으로 수렴시키는 것이 핵심 원리입니다.

```
Desired State (Git) → Diff Engine (ArgoCD) → Reconciliation → Observed State
         ↑_________________________피드백 루프______________________________↑
```

이 모델에서 Git 저장소는 단순한 코드 저장소가 아니라 **시스템의 단일 진실 원천(Single Source of Truth)**이 됩니다.

---

# PART 2: GITHUB ACTIONS — 내부 아키텍처와 고급 패턴

## 2.1 런타임 아키텍처 심층 분석

GitHub Actions의 실행 모델은 **Runner Broker 패턴**으로 구성되어 있습니다. GitHub API 서버가 FIFO 큐를 유지하고, Runner(GitHub-hosted 또는 self-hosted)가 롱 폴링(long-polling) 방식으로 작업을 가져갑니다. 이 때 중요한 점은 GitHub-hosted runner는 **매 Job마다 완전히 새로운 VM**을 프로비저닝한다는 것이고, 이것이 보안 격리의 핵심이지만 동시에 캐시 전략을 복잡하게 만드는 원인입니다.

Self-hosted runner를 설계할 때는 러너 레이블(label)과 러너 그룹(group)을 조합하여 **워크로드 격리**를 구현해야 합니다. 예를 들어, GPU 빌드, ARM 크로스컴파일, 고보안 배포 작업 각각에 전용 러너 풀을 할당하는 것이 엔터프라이즈 수준의 설계입니다.

## 2.2 Workflow 고급 패턴

### 재사용 가능한 워크플로우 (Reusable Workflows)

중앙화된 워크플로우 관리의 핵심은 `workflow_call` 이벤트를 활용한 워크플로우 합성(composition)입니다. 이것은 단순한 코드 재사용이 아니라 **조직 차원의 배포 표준을 강제하는 메커니즘**입니다.

```yaml
# .github/workflows/reusable-docker-build.yml
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      dockerfile-path:
        required: false
        type: string
        default: './Dockerfile'
    secrets:
      registry-password:
        required: true
    outputs:
      image-digest:
        description: "빌드된 이미지의 SHA256 다이제스트"
        value: ${{ jobs.build.outputs.digest }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.docker-build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      # BuildKit의 캐시 마운트를 활용하여 레이어 캐시를 최적화
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: |
            image=moby/buildkit:master
            network=host

      - name: Build and Push
        id: docker-build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ inputs.dockerfile-path }}
          # 레지스트리 캐시를 사용하여 레이어 재사용률 극대화
          cache-from: type=registry,ref=${{ inputs.image-name }}:buildcache
          cache-to: type=registry,ref=${{ inputs.image-name }}:buildcache,mode=max
          push: true
          # 이미지 불변성 보장을 위해 태그 대신 다이제스트 사용
          tags: ${{ inputs.image-name }}:${{ github.sha }}
```

### 매트릭스 전략의 고급 활용

단순한 OS/버전 매트릭스를 넘어서, **동적 매트릭스(Dynamic Matrix)**를 활용하면 변경된 서비스만 선택적으로 빌드하는 스마트 파이프라인을 구현할 수 있습니다.

```yaml
jobs:
  # 1단계: 변경된 서비스 목록을 동적으로 계산
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # git diff를 위해 전체 히스토리 필요
      
      - name: Detect changed services
        id: set-matrix
        run: |
          # main 브랜치 대비 변경된 디렉토리를 서비스 목록으로 변환
          CHANGED=$(git diff --name-only origin/main...HEAD | \
            grep -E '^services/' | \
            cut -d/ -f2 | sort -u | \
            jq -R -s -c 'split("\n") | map(select(length > 0))')
          echo "matrix={\"service\":$CHANGED}" >> $GITHUB_OUTPUT

  # 2단계: 동적 매트릭스 기반으로 병렬 빌드 실행
  build:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.matrix != '{"service":[]}' }}
    strategy:
      matrix: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
      fail-fast: false  # 하나 실패해도 나머지는 계속 실행
    uses: ./.github/workflows/reusable-docker-build.yml
    with:
      image-name: myregistry.azurecr.io/${{ matrix.service }}
```

## 2.3 보안 강화 패턴: OIDC와 비밀 관리

GitHub Actions의 가장 중요한 보안 발전은 **OIDC(OpenID Connect) 기반의 임시 자격증명 발급**입니다. 이것은 장기 자격증명(Access Key)을 완전히 제거하고, 워크플로우 실행 시에만 유효한 임시 토큰을 클라우드 프로바이더로부터 직접 발급받는 방식입니다.

```yaml
permissions:
  id-token: write   # OIDC 토큰 요청 권한
  contents: read

jobs:
  deploy:
    steps:
      # AWS에 장기 자격증명 없이 안전하게 인증
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
          # 역할 세션 이름에 워크플로우 정보를 포함시켜 CloudTrail 추적 용이화
          role-session-name: GitHubActions-${{ github.run_id }}
```

AWS IAM 정책에서는 GitHub의 OIDC 프로바이더가 발급한 토큰에 포함된 클레임(sub, repository, ref)을 조건으로 걸어 **최소 권한 원칙**을 정밀하게 구현할 수 있습니다.

## 2.4 Actions 성능 최적화: 캐시 전략

GitHub Actions 캐시는 내부적으로 Azure Blob Storage 기반이며, 캐시 키(cache key)와 복원 키(restore keys)의 계층적 설계가 핵심입니다. 이를 "캐시 폴백 체인(Cache Fallback Chain)"이라고 부르며, 정확한 캐시 히트 → 부분 매칭 → 미스 순으로 탐색합니다.

```yaml
- name: Cache Gradle dependencies (계층적 캐시 키 전략)
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    # 최정밀 키: 락파일 기반
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    # 폴백 체인: OS+도구 버전 → OS만
    restore-keys: |
      ${{ runner.os }}-gradle-
      ${{ runner.os }}-
```

---

# PART 3: HELM CHART — 패키지 관리의 심층 이해

## 3.1 Helm의 릴리스 상태 기계

Helm은 릴리스(Release)라는 개념을 통해 Kubernetes 리소스의 **생명주기를 추상화**합니다. 내부적으로 릴리스 정보는 Kubernetes Secret 오브젝트로 저장되며(`helm.sh/release.v1` 어노테이션), 각 릴리스는 최대 `--history-max` 개수의 리비전을 유지합니다.

```
pending-install → deployed → superseded (이전 버전)
     ↓                ↓
  failed          pending-upgrade → deployed
                       ↓
                    failed (여기서 rollback 가능)
```

이 상태 기계를 이해하면 `helm upgrade --atomic` 플래그의 의미가 명확해집니다. `--atomic`은 업그레이드 실패 시 자동으로 이전 버전으로 롤백하고, 릴리스 상태를 `failed`가 아닌 이전 `deployed` 상태로 복원합니다.

## 3.2 고급 템플릿 패턴

### Named Templates과 헬퍼 함수

`_helpers.tpl`은 단순한 레이블 생성기가 아니라, **조직 차원의 쿠버네티스 리소스 표준을 인코딩하는 도구**입니다.

```yaml
{{/*
공통 레이블 정의 - Kubernetes 권장 레이블 스펙을 준수
https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
환경별 리소스 제한을 동적으로 계산하는 헬퍼
개발환경은 최소 리소스, 프로덕션은 values.yaml 기반으로 분기
*/}}
{{- define "myapp.resources" -}}
{{- if eq .Values.environment "production" }}
resources:
  requests:
    cpu: {{ .Values.resources.requests.cpu | default "100m" }}
    memory: {{ .Values.resources.requests.memory | default "128Mi" }}
  limits:
    cpu: {{ .Values.resources.limits.cpu | default "500m" }}
    memory: {{ .Values.resources.limits.memory | default "512Mi" }}
{{- else }}
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
{{- end }}
{{- end }}
```

### Schema Validation으로 values.yaml 계약 강제화

`values.schema.json`은 Helm 3의 강력한 기능으로, 잘못된 values 입력을 **배포 전에 컴파일 타임에 차단**합니다. 이것은 차트를 사용하는 팀과의 API 계약을 형식화합니다.

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "title": "MyApp Helm Chart Values",
  "type": "object",
  "required": ["image", "replicaCount"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 50,
      "description": "파드 복제본 수 (1-50 범위)"
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": {
          "type": "string",
          "pattern": "^[a-z0-9]+([._-][a-z0-9]+)*(/[a-z0-9]+([._-][a-z0-9]+)*)*$"
        },
        "tag": {
          "type": "string",
          "description": "이미지 태그. latest 사용 금지 - 불변성 원칙"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    }
  }
}
```

## 3.3 Helm Hooks와 배포 오케스트레이션

Helm Hook은 릴리스 생명주기의 특정 시점에 Job을 실행하는 메커니즘입니다. 고급 패턴으로는 `pre-upgrade` 훅에서 데이터베이스 마이그레이션을 실행하고, 실패 시 배포를 자동으로 중단하는 패턴이 있습니다.

```yaml
# templates/db-migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-db-migration
  annotations:
    # 업그레이드 전에 실행되어야 DB 스키마가 코드보다 먼저 적용됨
    "helm.sh/hook": pre-upgrade,pre-install
    # 이 훅이 완료되어야 다음 단계로 진행 (가중치로 순서 제어)
    "helm.sh/hook-weight": "-5"
    # 성공 시 Job을 자동으로 삭제하여 클러스터 정리
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  # 마이그레이션 실패 시 재시도 없이 즉시 실패 → 배포 중단
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["python", "manage.py", "migrate", "--noinput"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ .Release.Name }}-db-secret
              key: url
```

## 3.4 OCI 레지스트리와 Helm Chart 버전 관리

Helm 3.8부터 OCI(Open Container Initiative) 레지스트리를 차트 저장소로 사용할 수 있습니다. 이것은 차트와 컨테이너 이미지를 **동일한 레지스트리 인프라에서 통합 관리**하는 패턴을 가능하게 합니다.

```bash
# Azure Container Registry에 차트 푸시
helm package ./mychart
helm push mychart-1.2.3.tgz oci://myregistry.azurecr.io/helm

# 다이제스트 기반 참조로 불변성 보장
helm install myapp oci://myregistry.azurecr.io/helm/mychart \
  --version 1.2.3 \
  --set image.tag=$IMAGE_DIGEST
```

---

# PART 4: ARGOCD — GITOPS 제어 플레인의 심층 이해

## 4.1 ArgoCD의 내부 컴포넌트 아키텍처

ArgoCD는 단일 바이너리가 아닌, 책임이 명확히 분리된 **마이크로서비스 아키텍처**로 구성되어 있습니다.

**argocd-server**는 API 서버이자 Web UI 서버입니다. gRPC와 HTTP를 동시에 처리하며, CLI와 UI 모두 이 서버를 통합니다.

**argocd-repo-server**는 Git 저장소를 클론하고 매니페스트를 생성하는 독립 프로세스입니다. 이것을 별도 컴포넌트로 분리한 이유는 중요합니다. 매니페스트 생성 작업(Helm 렌더링, Kustomize 빌드)은 CPU 집약적이고 잠재적으로 위험한 코드(Custom Plugin)를 실행하므로, Application Controller와 격리해야 합니다.

**argocd-application-controller**는 쿠버네티스 컨트롤러 패턴으로 동작하며 실제 동기화 로직을 담당합니다. 내부적으로 **두 개의 상태를 지속적으로 비교**합니다: `target state`(Git에서 렌더링된 상태)와 `live state`(클러스터에서 실제로 관찰된 상태).

## 4.2 동기화 전략의 깊은 이해

### Sync Phases와 Waves

복잡한 애플리케이션 배포에서는 리소스 간의 **의존성 순서**를 제어해야 합니다. ArgoCD는 `sync-wave` 어노테이션으로 이를 구현합니다.

```yaml
# 1단계: 네임스페이스와 CRD (웨이브 -2)
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
---
# 2단계: 시크릿과 컨피그맵 (웨이브 -1)
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
---
# 3단계: 데이터베이스 마이그레이션 (웨이브 0, 기본값)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: Sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    argocd.argoproj.io/sync-wave: "0"
---
# 4단계: 애플리케이션 Deployment (웨이브 1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

### Resource Health Check 커스터마이징

ArgoCD의 내장 헬스 체크는 표준 쿠버네티스 리소스에만 적용됩니다. Canary, CronJob, 외부 DB 등의 커스텀 리소스에 대한 헬스 로직은 **Lua 스크립트**로 확장할 수 있습니다.

```yaml
# argocd-cm ConfigMap에 커스텀 헬스 체크 추가
data:
  resource.customizations.health.argoproj.io_Rollout: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.phase == "Degraded" then
        hs.status = "Degraded"
        hs.message = obj.status.message
        return hs
      end
      if obj.status.phase == "Paused" then
        hs.status = "Suspended"
        hs.message = "카나리 배포가 일시 중지됨. 수동 승인 대기 중"
        return hs
      end
    end
    hs.status = "Healthy"
    return hs
```

## 4.3 ApplicationSet — 멀티 클러스터/멀티 환경 자동화

`ApplicationSet`은 하나의 템플릿으로 여러 환경, 클러스터, 또는 Git 리포지토리에 걸쳐 ArgoCD Application을 **동적으로 생성**하는 컨트롤러입니다. 이것은 플랫폼 팀이 수백 개의 서비스를 중앙에서 관리할 때 핵심 도구입니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-apps
  namespace: argocd
spec:
  generators:
  # Git 저장소의 디렉토리 구조를 기반으로 Application 자동 생성
  - matrix:
      generators:
      # 첫 번째 차원: 클러스터 목록
      - clusters:
          selector:
            matchLabels:
              environment: production
      # 두 번째 차원: 서비스 디렉토리 목록
      - git:
          repoURL: https://github.com/myorg/platform-config
          revision: HEAD
          directories:
          - path: services/*
  template:
    metadata:
      # 클러스터 이름과 서비스 이름을 조합하여 고유 Application 이름 생성
      name: '{{name}}-{{path.basename}}'
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/platform-config
        targetRevision: HEAD
        path: '{{path}}'
        helm:
          valueFiles:
          - values.yaml
          # 클러스터별 오버라이드 값 파일 (없으면 무시)
          - 'values-{{name}}.yaml'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

## 4.4 ArgoCD Image Updater와 CI/CD 통합

ArgoCD Image Updater는 컨테이너 레지스트리를 모니터링하여 새 이미지가 푸시되면 자동으로 Git 커밋을 생성하는 도구입니다. 이것은 CI(GitHub Actions)와 CD(ArgoCD) 사이의 **이벤트 브리지** 역할을 합니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  annotations:
    # 이미지 업데이터 활성화
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry.azurecr.io/myapp
    # 시맨틱 버저닝 기반 자동 업데이트 (patch 버전만)
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^1\.2\.\d+$
    # Git write-back: values.yaml을 직접 수정하여 GitOps 원칙 유지
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

---

# PART 5: AKS (Azure Kubernetes Service) — 고급 운영 패턴

## 5.1 AKS의 컨트롤 플레인 아키텍처

AKS는 컨트롤 플레인(API 서버, etcd, 스케줄러, 컨트롤러 매니저)을 **Microsoft가 완전 관리**하는 모델입니다. 이것의 심층 의미는 컨트롤 플레인 SLA가 데이터 플레인(노드)과 별도로 계산된다는 것입니다. etcd의 백업, API 서버의 고가용성, 쿠버네티스 버전 업그레이드가 모두 관리형 서비스로 처리됩니다.

## 5.2 Workload Identity와 OIDC의 연동

AKS의 **Workload Identity**는 쿠버네티스 Service Account와 Azure AD Managed Identity를 OIDC 연합을 통해 바인딩합니다. 이것은 Pod 내부에서 Azure Key Vault, Storage Account 등에 접근할 때 시크릿 없이 인증할 수 있게 해줍니다.

```bash
# AKS 클러스터에 OIDC 발급자 활성화
az aks update -g myRG -n myAKS --enable-oidc-issuer --enable-workload-identity

# 페더레이션 자격증명 생성: SA ↔ Managed Identity 바인딩
az identity federated-credential create \
  --name myapp-federated-credential \
  --identity-name myapp-identity \
  --resource-group myRG \
  --issuer $(az aks show -g myRG -n myAKS --query oidcIssuerProfile.issuerUrl -o tsv) \
  --subject "system:serviceaccount:default:myapp-sa" \
  --audience api://AzureADTokenExchange
```

## 5.3 Karpenter와 노드 프로비저닝 최적화

AKS에서 Karpenter(또는 KEDA 기반 KARPENTER)를 사용하면 Cluster Autoscaler 대비 훨씬 정밀한 노드 프로비저닝이 가능합니다. Karpenter는 **Pending Pod의 리소스 요구사항을 직접 분석**하여 최적의 VM SKU를 실시간으로 결정합니다.

```yaml
apiVersion: karpenter.azure.com/v1alpha2
kind: AKSNodeClass
metadata:
  name: gpu-nodeclass
spec:
  imageFamily: AzureLinux
  # GPU 워크로드를 위한 NVidia 드라이버 자동 설치
  osDiskSizeGB: 128
---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-pool
spec:
  template:
    spec:
      nodeClassRef:
        name: gpu-nodeclass
      requirements:
      - key: karpenter.azure.com/sku-family
        operator: In
        values: ["NCas_T4_v3", "NC_A100_v4"]  # T4 또는 A100 GPU
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]  # 스팟 인스턴스 우선 시도
  # 비용 최적화: 노드 미사용 시 30분 후 자동 종료
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30m
```

## 5.4 AKS CI/CD 파이프라인 통합 패턴

AKS 배포 파이프라인의 핵심은 **Blue-Green 및 Canary 배포 전략**을 Kubernetes-native 방식으로 구현하는 것입니다. Azure Deployment Environments와 연동하면 PR별 임시 환경을 자동으로 프로비저닝할 수 있습니다.

```yaml
# GitHub Actions에서 AKS에 Helm 배포
- name: Deploy to AKS
  run: |
    # kubelogin을 통한 AAD 인증 (워크로드 아이덴티티 사용)
    kubelogin convert-kubeconfig -l workloadidentity
    
    helm upgrade --install myapp ./charts/myapp \
      --namespace production \
      --create-namespace \
      --set image.tag=${{ github.sha }} \
      --set ingress.host=myapp.example.com \
      --values ./charts/myapp/values-production.yaml \
      --atomic \           # 실패 시 자동 롤백
      --timeout 10m \      # 배포 타임아웃
      --wait \             # 모든 리소스가 Ready 상태가 될 때까지 대기
      --wait-for-jobs      # Job 완료까지 대기 (DB 마이그레이션 포함)
```

---

# PART 6: AWS CI/CD 서비스 — 심층 아키텍처

## 6.1 AWS CodePipeline의 아티팩트 스토어 모델

CodePipeline의 핵심 개념은 **아티팩트(Artifact)**입니다. 각 스테이지는 S3 버킷에 저장된 아티팩트를 입력으로 받고 새로운 아티팩트를 출력합니다. 이 설계는 파이프라인을 **비동기적이고 재실행 가능한 상태 기계**로 만듭니다.

중요한 고급 개념은 `cross-account` 배포입니다. CodePipeline이 한 AWS 계정(tooling account)에서 실행되면서 다른 계정(production account)에 배포할 때, KMS 키 정책과 IAM 역할 연쇄(role chaining)를 통한 아티팩트 암호화가 필수입니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::TOOLING_ACCOUNT:role/CodePipelineRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

## 6.2 AWS CodeBuild 고급 최적화

CodeBuild의 빌드 성능 최적화는 두 가지 축으로 생각해야 합니다: **컴퓨팅 스펙 선택**과 **캐시 전략**입니다.

로컬 캐시(Local Cache)는 CodeBuild 플릿 내에서만 유효하고, S3 캐시는 플릿 간 공유가 가능하지만 S3 전송 오버헤드가 있습니다. 엔터프라이즈 패턴으로는 **캐시 전용 도커 이미지를 ECR에 유지**하는 방식이 가장 효율적입니다.

```yaml
# buildspec.yml
version: 0.2

env:
  variables:
    ECR_CACHE_IMAGE: 123456789.dkr.ecr.us-east-1.amazonaws.com/build-cache:latest

phases:
  pre_build:
    commands:
      # ECR 로그인
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_CACHE_IMAGE
      # 이전 빌드 캐시 레이어 풀 (실패해도 계속 진행)
      - docker pull $ECR_CACHE_IMAGE || true

  build:
    commands:
      - |
        docker build \
          --cache-from $ECR_CACHE_IMAGE \
          --build-arg BUILDKIT_INLINE_CACHE=1 \
          -t myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION \
          -t $ECR_CACHE_IMAGE \
          .

  post_build:
    commands:
      # 새 캐시 이미지 푸시 (다음 빌드를 위해)
      - docker push $ECR_CACHE_IMAGE
      # 최종 이미지는 SHA 태그로 불변성 보장
      - docker push myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION
```

## 6.3 Amazon EKS와 GitOps 통합

EKS에서 ArgoCD를 운영할 때 핵심 설계 결정은 **허브-스포크 모델**입니다. 단일 ArgoCD 인스턴스(허브 클러스터)가 여러 EKS 클러스터(스포크)를 관리하는 구조입니다.

클러스터 등록 시 IRSA(IAM Roles for Service Accounts)와 연동하면 ArgoCD가 각 스포크 클러스터에 접근하는 자격증명을 Secrets Manager에서 동적으로 갱신할 수 있습니다.

## 6.4 AWS CodeDeploy와 ECS의 Blue-Green 배포

ECS와 CodeDeploy를 결합한 Blue-Green 배포는 **트래픽 이동(traffic shifting)**을 세밀하게 제어할 수 있습니다. `Linear10PercentEvery1Minute` 같은 배포 구성을 사용하면 카나리 배포를 자동화할 수 있으며, CloudWatch 알람과 연동하면 에러율 증가 시 자동 롤백이 트리거됩니다.

```json
{
  "deploymentConfigName": "Custom-Canary",
  "trafficRoutingConfig": {
    "type": "TimeBasedCanary",
    "timeBasedCanary": {
      "canaryPercentage": 10,
      "canaryInterval": 5
    }
  },
  "minimumHealthyHosts": {
    "type": "FLEET_PERCENT",
    "value": 85
  }
}
```

---

# PART 7: 고급 패턴 — 엔터프라이즈 CI/CD 아키텍처

## 7.1 진보적 배포 전략 (Progressive Delivery)

카나리, 블루-그린을 넘어선 **Feature Flag 기반 Progressive Delivery**는 Flagger와 Argo Rollouts를 통해 구현됩니다. 이 패턴에서는 배포와 릴리스가 분리됩니다: 코드는 프로덕션에 배포되지만, 기능은 점진적으로 사용자에게 노출됩니다.

```yaml
# Argo Rollouts Canary 전략
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      # 카나리 분석 템플릿 참조
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 2
        args:
        - name: service-name
          value: myapp-canary
      steps:
      - setWeight: 5       # 5% 트래픽
      - pause: {duration: 5m}
      - setWeight: 20      # 20% 트래픽
      - pause: {duration: 5m}
      - setWeight: 50      # 50% 트래픽
      - pause: {}          # 수동 승인 대기
      - setWeight: 100
---
# 성공률 기반 자동 승인/롤백
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    # 5분 동안 3번 연속 성공률 < 95%이면 자동 롤백
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{
            service="{{args.service-name}}",
            status!~"5.."
          }[2m])) /
          sum(rate(http_requests_total{
            service="{{args.service-name}}"
          }[2m]))
    successCondition: result[0] >= 0.95
```

## 7.2 Policy as Code — OPA와 Kyverno

엔터프라이즈 배포 파이프라인에서 정책 검증은 **shift-left** 원칙에 따라 최대한 일찍 이루어져야 합니다. Kyverno를 사용하면 Helm 렌더링 이후 ArgoCD 동기화 이전에 정책을 강제할 수 있습니다.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: 리소스 제한 필수화
    policies.kyverno.io/description: |
      프로덕션 워크로드는 반드시 CPU와 메모리 제한을 설정해야 합니다.
      이는 노이지 네이버(noisy neighbor) 문제를 방지합니다.
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-container-resources
    match:
      any:
      - resources:
          kinds: [Deployment, StatefulSet]
          namespaces: ["production", "staging"]
    validate:
      message: "모든 컨테이너는 resources.limits.cpu와 resources.limits.memory를 지정해야 합니다."
      pattern:
        spec:
          template:
            spec:
              containers:
              - name: "*"
                resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
```

---

# PART 8: 대기업 면접 Q&A 30선

> **대상 기업:** Amazon, Apple, Nvidia, Google, Microsoft, Tesla
> **대상 직급:** Senior Cloud Engineer / Staff Engineer / Principal Engineer

---

## Q1. GitHub Actions에서 OIDC를 사용한 AWS 인증을 구현했을 때, 보안 측면에서 Access Key 방식과의 근본적인 차이점은 무엇이며, 이를 설계할 때 고려해야 할 잠재적 취약점은 무엇인가요?

**핵심 답변:** OIDC 방식의 근본적인 차이는 **비밀 공유(secret sharing)의 제거**입니다. Access Key는 장기 자격증명으로, 한 번 유출되면 폐기될 때까지 악용 가능합니다. 반면 OIDC는 JWT 기반 단기 토큰(기본 1시간)을 사용합니다. GitHub이 발급한 OIDC 토큰에는 `sub` 클레임에 `repo:org/repo:ref:refs/heads/main` 형태의 정보가 포함되어, AWS IAM에서 특정 브랜치의 특정 워크플로우에서만 역할을 수임할 수 있도록 조건을 걸 수 있습니다.

잠재적 취약점으로는, `sub` 클레임을 `repo:org/repo:*`처럼 와일드카드로 설정하면 해당 저장소의 어떤 브랜치에서도 역할을 수임할 수 있어 악의적인 PR을 통한 공격이 가능합니다. 또한 GitHub 자체가 타협될 경우 신뢰 체인 전체가 손상될 수 있으므로, 고보안 환경에서는 `aws:RequestedRegion` 조건을 추가하는 것이 권장됩니다.

---

## Q2. Helm Chart의 `values.yaml`을 여러 환경(dev/staging/prod)에서 관리할 때, 어떤 계층화 전략을 사용하시나요? 그리고 시크릿 관리는 어떻게 접근하시나요?

**핵심 답변:** 계층화 전략은 **기본값 → 환경 오버라이드 → 클러스터별 오버라이드**의 3단계 계층을 사용합니다. `values.yaml`이 기본값을, `values-staging.yaml`이 환경별 오버라이드를, `values-staging-us-east-1.yaml`이 클러스터별 세부 오버라이드를 담당합니다.

시크릿 관리는 절대 `values.yaml`에 평문으로 저장하지 않습니다. 두 가지 접근법이 있습니다: 첫째, **Helm Secrets 플러그인**을 사용하여 `sops`(Mozilla SOPS)로 암호화된 values 파일을 Git에 커밋하는 방식입니다. 둘째, **External Secrets Operator**를 사용하여 AWS Secrets Manager나 Azure Key Vault에서 런타임에 시크릿을 동적으로 가져오는 방식입니다. 엔터프라이즈 환경에서는 두 번째 방식이 선호됩니다. Git에 어떠한 형태로도 시크릿 데이터가 존재하지 않기 때문입니다.

---

## Q3. ArgoCD에서 `selfHeal`과 `automated sync`를 활성화했을 때, 운영 중에 예상치 못한 드리프트(drift)가 감지될 경우 어떻게 처리하시겠습니까? 그리고 이로 인해 발생할 수 있는 운영상의 위험은 무엇인가요?

**핵심 답변:** `selfHeal`의 핵심 위험은 **정당한 긴급 수동 변경을 자동으로 되돌리는 것**입니다. 예를 들어, 인시던트 대응 중 kubectl로 직접 ConfigMap을 수정했을 때 ArgoCD가 이를 드리프트로 감지하고 즉시 롤백합니다. 이를 방지하기 위해 인시던트 대응 SOP(표준 운영 절차)에 "ArgoCD에서 해당 앱을 먼저 sync 비활성화"를 의무화해야 합니다.

또한, `ignoreDifferences` 설정을 통해 HPA(Horizontal Pod Autoscaler)의 `replicas` 필드처럼 런타임에 동적으로 변경되는 필드는 드리프트 감지에서 제외해야 합니다. 그렇지 않으면 HPA가 스케일아웃한 파드 수를 ArgoCD가 지속적으로 Git의 값으로 되돌리는 충돌이 발생합니다.

---

## Q4. EKS에서 멀티 테넌트 환경을 구성할 때, 네임스페이스 수준의 격리와 클러스터 수준의 격리 중 어느 것을 선택하겠습니까? 그 근거와 트레이드오프는 무엇인가요?

**핵심 답변:** 이것은 **Kubernetes의 소프트 멀티테넌시 한계**에 대한 이해를 묻는 질문입니다. 네임스페이스 격리는 동일한 컨트롤 플레인과 데이터 플레인을 공유하므로 진정한 보안 격리가 아닙니다. CVE로 인한 컨테이너 탈출이 가능하며, API 서버 DDoS가 전체 테넌트에 영향을 줍니다.

선택 기준은 **규제 요건**과 **폭발 반경**입니다. 금융, 의료 데이터처럼 규제 격리가 필요하면 클러스터 수준, 동일 조직 내 팀 격리이면 네임스페이스 수준이 비용 효율적입니다. 중간 지점으로는 EKS의 **노드 그룹 격리**와 **NetworkPolicy + PodSecurityAdmission**을 결합하여 소프트 멀티테넌시를 강화하는 패턴이 있습니다. Kubernetes의 공식 문서도 "Kubernetes는 하드 멀티테넌시를 위해 설계되지 않았다"고 명시합니다.

---

## Q5. GitHub Actions의 Composite Actions와 Reusable Workflows의 차이점을 설명하고, 각각의 적절한 사용 사례를 제시하세요.

**핵심 답변:** 두 개념의 근본적인 차이는 **실행 컨텍스트**입니다. Composite Actions는 호출자(caller) 워크플로우의 동일한 runner Job 내에서 실행됩니다. 즉, 호출자의 환경 변수, 시크릿, 파일시스템에 직접 접근할 수 있습니다. 반면 Reusable Workflows는 완전히 독립된 새로운 Job으로 실행됩니다. 별도의 runner를 할당받고, 시크릿은 명시적으로 전달해야 하며, 출력(outputs)도 명시적으로 정의해야 합니다.

Composite Actions의 적절한 사용 사례는 "Docker 빌드 + 태그 + 푸시"처럼 동일 Job 컨텍스트에서 여러 스텝을 하나로 묶는 경우입니다. Reusable Workflows는 "전체 배포 파이프라인 표준"처럼 독립적인 보안 경계와 Job 레벨 승인이 필요한 경우에 사용합니다. 예를 들어, 프로덕션 배포는 별도 Job에서 Environment approval을 받아야 할 때 Reusable Workflow가 적합합니다.

---

## Q6. 대규모 모노레포(monorepo)에서 GitHub Actions를 최적화할 때 어떤 전략을 사용하시나요? (Nvidia, Google 수준의 규모 가정)

**핵심 답변:** 핵심 전략은 **Affected Graph Analysis**입니다. 변경된 파일의 의존성 그래프를 분석하여 영향받는 서비스만 빌드/테스트합니다. `nx affected`, `turborepo`, 또는 자체 구현한 `git diff` 기반 스크립트를 사용합니다.

두 번째 전략은 **Build Caching at Scale**입니다. Bazel(Google의 오픈소스 빌드 도구)은 각 빌드 타겟의 입력 해시를 계산하여 원격 캐시에서 아티팩트를 재사용합니다. 코드가 변경되지 않은 서비스는 단 1초도 빌드하지 않습니다.

세 번째는 **Self-hosted Runner Auto-scaling**입니다. GitHub Actions Runner Controller(ARC)를 사용하여 Kubernetes 위에서 runner를 Pod으로 실행하고, 큐의 대기 작업 수에 따라 자동으로 스케일합니다. 이렇게 하면 피크 시간에만 컴퓨팅 비용이 발생합니다.

---

## Q7. Helm Rollback 중에 데이터베이스 마이그레이션이 이미 적용된 상황에서 어떻게 처리하시겠습니까?

**핵심 답변:** 이것은 **가장 어려운 배포 시나리오** 중 하나입니다. Helm Rollback은 쿠버네티스 리소스만 이전 상태로 되돌립니다. 데이터베이스는 롤백하지 않습니다. 따라서 DB 스키마와 코드 사이의 **하위 호환성(backward compatibility)**을 유지하는 것이 유일한 근본적 해결책입니다.

구체적인 패턴으로는 **Expand-Contract 마이그레이션**이 있습니다. 새 컬럼을 추가할 때 즉시 NOT NULL로 추가하지 않고, 먼저 NULL 허용으로 추가(Expand), 애플리케이션 코드가 두 버전 모두 지원하는 상태에서 배포, 이후 충분한 시간이 지난 뒤에 NULL 제거(Contract)하는 3단계 과정을 거칩니다. 이렇게 하면 언제든 코드를 롤백해도 DB 스키마와 호환됩니다.

---

## Q8. ArgoCD의 Application of Applications 패턴과 ApplicationSet의 차이점은 무엇이며, 언제 어느 것을 선택하나요?

**핵심 답변:** App of Apps는 **부모 ArgoCD Application이 다른 Application YAML들이 있는 Git 경로를 가리키는** 재귀적 패턴입니다. 이것은 간단하지만 정적입니다. 새 클러스터나 환경을 추가하려면 Git에서 파일을 수동으로 생성해야 합니다.

ApplicationSet은 **동적 생성**을 위한 것입니다. Generator(clusters, git, matrix, pull request 등)가 새로운 상태(새 클러스터 등록, 새 디렉토리 추가, PR 오픈)를 감지하면 Application을 자동으로 생성합니다. PR Preview Environment처럼 **이벤트 드리븐으로 Application이 생성/삭제**되어야 하는 경우에는 ApplicationSet의 PullRequest Generator가 유일한 실용적 해결책입니다.

---

## Q9. AKS에서 Private Cluster를 구성했을 때, CI/CD 파이프라인이 kubectl 명령을 실행하기 위해 어떤 네트워킹 아키텍처가 필요한가요?

**핵심 답변:** AKS Private Cluster는 API 서버 엔드포인트가 VNet 내부에만 노출됩니다. 인터넷에서 직접 접근이 불가능합니다. CI/CD 파이프라인 접근을 위해서는 세 가지 옵션이 있습니다.

첫째, **Self-hosted Runner를 동일 VNet에 배치**하는 방법입니다. Runner VM이 VNet 내부에 있으므로 Private Endpoint를 통해 API 서버에 직접 접근합니다. 이것이 가장 단순하고 보안이 강한 방법입니다.

둘째, **Azure Private Link Service + GitHub Actions hosted runner**를 사용하는 방법으로, Azure의 `command invoke` API(`az aks command invoke`)를 사용하여 클러스터 내부에서 kubectl을 대리 실행합니다. 간단하지만 `command invoke`의 로그가 제한적입니다.

셋째, **Jump Server(Bastion) + SSH 터널링** 방식입니다. 이것은 레거시 방식으로 관리 부담이 큽니다.

---

## Q10. AWS CodePipeline에서 Cross-Account 배포를 구현할 때 보안 아키텍처를 어떻게 설계하시겠습니까?

**핵심 답변:** Cross-Account 배포의 핵심 보안 문제는 **아티팩트 접근 권한과 암호화**입니다. 설계 원칙은 "Tooling Account에서 Production Account로 최소 권한만 위임"입니다.

구체적 아키텍처: Tooling Account의 CodePipeline이 S3 버킷에 빌드 아티팩트를 저장할 때 **고객 관리형 KMS 키(CMK)**로 암호화합니다. Production Account의 배포 역할(CodeDeploy/CloudFormation)에 이 KMS 키에 대한 `kms:Decrypt` 권한을 부여합니다. Tooling Account의 CodePipeline 실행 역할이 `sts:AssumeRole`로 Production Account의 배포 역할을 수임합니다. 이 신뢰 관계에 `aws:PrincipalOrgID` 조건을 추가하여 동일 AWS Organizations 내부에서만 역할 수임이 가능하게 합니다. AWS Organizations SCP(Service Control Policy)로 Production Account에서 생성 가능한 IAM 역할의 권한 상한을 제한합니다.

---

## Q11. Kubernetes에서 ConfigMap이나 Secret이 변경되었을 때 Pod를 자동으로 재시작하는 방법과 그 트레이드오프를 설명하세요.

**핵심 답변:** 기본적으로 Kubernetes는 ConfigMap/Secret 변경 시 Pod를 자동 재시작하지 않습니다. 해결책은 여러 가지입니다.

가장 단순한 방법은 `checksum` 어노테이션 패턴으로, Helm 템플릿에서 ConfigMap의 해시값을 Deployment 어노테이션에 주입하는 것입니다. ConfigMap이 변경되면 해시가 바뀌고, Deployment spec이 변경되므로 Rolling Update가 트리거됩니다. 두 번째는 `Reloader`(Stakater)와 같은 오퍼레이터를 사용하는 방법으로, 어노테이션 기반으로 ConfigMap/Secret 변경을 감시하고 자동 재시작합니다. 세 번째는 CSI Secret Store 드라이버를 사용하여 시크릿을 Volume으로 마운트하면, `autoRotation` 기능으로 새 값이 파일시스템에 자동 반영되어 애플리케이션이 파일 변경을 감지하여 설정을 리로드할 수 있습니다. 마지막 방법은 재시작 없이 시크릿이 갱신되므로 제로 다운타임에 유리합니다.

---

## Q12. CI/CD 파이프라인에서 공급망 보안(Supply Chain Security)을 어떻게 구현하시나요? SLSA 프레임워크를 기준으로 설명하세요.

**핵심 답변:** SLSA(Supply-chain Levels for Software Artifacts)는 빌드 무결성을 4단계로 정의합니다. 대기업 수준의 목표는 SLSA Level 3-4입니다.

SLSA Level 2를 위해서는 **빌드 출처(Provenance)**를 생성해야 합니다. GitHub Actions의 `slsa-framework/slsa-github-generator`를 사용하면 빌드에 서명된 SLSA Provenance 문서가 자동 생성됩니다. 이 문서는 "어떤 소스 코드가, 어떤 빌드 환경에서, 어떤 빌드 도구로 만들어졌는지"를 암호학적으로 증명합니다.

Level 3를 위해서는 **이미지 서명(cosign)**과 **SBOM(Software Bill of Materials) 생성**이 필요합니다. Syft로 SBOM을 생성하고, cosign으로 이미지와 SBOM에 서명한 뒤, Rekor 투명성 로그에 기록합니다. ArgoCD 배포 시 `cosign verify`를 통해 서명 검증을 강제할 수 있습니다.

---

## Q13. Helm Chart 테스트를 위한 전략을 설명하세요. Unit Test와 Integration Test를 어떻게 구현하시나요?

**핵심 답변:** Helm Chart 테스트는 세 레벨로 구성해야 합니다.

Unit Test로는 `helm-unittest` 플러그인을 사용하여 실제 클러스터 없이 렌더링 결과를 검증합니다. 다양한 values 조합으로 템플릿이 올바르게 렌더링되는지, 필수 레이블이 있는지, 리소스 제한이 설정되는지 등을 테스트합니다.

Linting으로는 `helm lint`, `helm template | kubeval` (스키마 검증), `helm template | checkov` (보안 정책 검증), `helm template | kube-score` (베스트 프랙티스 검증)를 파이프라인에 통합합니다.

Integration Test로는 `helm test` 명령과 실제 쿠버네티스 클러스터(KinD나 k3s)를 사용합니다. `helm.sh/hook: test` 어노테이션이 달린 Pod를 배포 후 실행하여 서비스 응답, DB 연결, 엔드포인트 접근 가능성을 검증합니다.

---

## Q14. GitOps 환경에서 긴급 핫픽스(hotfix)를 배포하는 프로세스를 어떻게 설계하시겠습니까? 속도와 안전성의 균형을 어떻게 맞추시나요?

**핵심 답변:** 이것은 "GitOps의 모든 변경이 Git을 통해야 한다"는 원칙과 "즉각적인 프로덕션 수정 필요"라는 현실 사이의 긴장을 다루는 질문입니다.

핵심은 **핫픽스도 Git을 통해야 하되, 속도를 최대화**하는 것입니다. 핫픽스 브랜치를 `hotfix/*` 패턴으로 만들면, GitHub Actions 파이프라인이 일반 PR 대비 **빠른 트랙(fast track)**으로 실행됩니다. 즉, 단위 테스트만 통과하면(통합 테스트 스킵) 즉시 배포 가능하도록 별도 워크플로우를 구성합니다. ArgoCD는 `hotfix/*` 브랜치를 바라보는 별도 Application을 두거나, 기존 Application의 Target Revision을 임시로 변경합니다.

중요한 것은, 핫픽스 배포 후 반드시 **Post-Mortem**을 통해 main 브랜치에 포팅하고 E2E 테스트를 강화하는 루프를 만드는 것입니다.

---

## Q15. Kubernetes에서 PodDisruptionBudget(PDB)과 관련하여 배포 전략을 설계할 때 어떤 점을 고려해야 하나요?

**핵심 답변:** PDB는 **볼런터리 중단(voluntary disruption)**에 대한 가용성 보증입니다. 노드 드레인, 클러스터 업그레이드, kubectl delete pod 같은 의도적 중단으로부터 최소 가용 파드 수를 보장합니다. 단, 노드 하드웨어 장애 같은 인볼런터리 중단에는 적용되지 않습니다.

배포와의 관계에서 주의할 점은 PDB가 Rolling Update를 차단할 수 있다는 것입니다. 예를 들어, `replicas: 3`, `maxUnavailable: 0` PDB를 설정하면 순수한 Recreate 방식 배포는 불가능합니다. 따라서 PDB 설정과 Deployment의 `rollingUpdate.maxUnavailable` 설정이 **수학적으로 호환**되어야 합니다. `minAvailable`을 퍼센트로 지정하면(`80%`) 스케일 다운 시에도 적절한 가용성을 보장할 수 있습니다. 또한 클러스터 업그레이드 시 PDB가 너무 엄격하면 노드 드레인이 영구 차단될 수 있으므로, `maxUnavailable: 1` 이상을 반드시 허용해야 합니다.

---

## Q16. AWS Lambda 기반 서버리스 애플리케이션의 CI/CD 파이프라인과 컨테이너 기반 파이프라인의 가장 큰 아키텍처적 차이점은 무엇인가요?

**핵심 답변:** 핵심 차이는 **배포 단위의 크기와 상태 관리**입니다. 컨테이너 파이프라인은 전체 애플리케이션을 하나의 이미지로 배포하고, Kubernetes가 상태(롤링 업데이트, 헬스체크)를 관리합니다. Lambda 파이프라인은 개별 함수 단위로 배포하며, 때로는 수백 개의 함수를 독립적으로 버전 관리해야 합니다.

Lambda 고유의 도전 과제로는 **Cold Start 관리**가 있습니다. 새 버전 배포 후 Provisioned Concurrency를 예열(warm-up)하는 단계를 파이프라인에 포함해야 합니다. 또한 Lambda의 **Alias 기반 트래픽 이동**(`aws lambda update-alias --routing-config`)을 사용하면 Canary 배포가 가능하며, CloudWatch 알람과 CodeDeploy를 연동하면 자동 롤백이 트리거됩니다. Infrastructure as Code(SAM, Serverless Framework, CDK)와의 통합도 중요한 차이점으로, Lambda 함수 코드와 인프라 변경이 항상 원자적으로 배포되어야 합니다.

---

## Q17. ArgoCD에서 대규모 클러스터를 관리할 때 Application Controller의 성능 문제를 어떻게 해결하시나요?

**핵심 답변:** Application Controller는 기본적으로 **단일 인스턴스**로 실행되며, 모든 Application의 상태를 주기적으로(기본 3분) 재조정합니다. 수백 개의 Application을 관리하면 컨트롤러 큐가 포화 상태가 됩니다.

해결책은 **샤딩(Sharding)**입니다. ArgoCD 2.0부터 Application Controller를 여러 replicas로 실행하고, 각 replica가 클러스터 집합의 부분집합을 담당하게 할 수 있습니다. `ARGOCD_CONTROLLER_SHARDING_ALGORITHM`을 `consistent-hashing`으로 설정하면 클러스터가 균등하게 분산됩니다.

추가 튜닝으로는 `--status-processors`와 `--operation-processors` 값을 늘려 병렬 처리를 향상시키고, `--repo-cache-expiration`을 조정하여 불필요한 Git 폴링을 줄입니다. Resource exclusion을 통해 ArgoCD가 무시해야 할 리소스(고빈도 변경 CRD 등)를 지정하는 것도 중요합니다.

---

## Q18. Tesla와 같은 IoT/엣지 컴퓨팅 환경에서 CI/CD 파이프라인을 설계할 때 클라우드 환경과 다른 고려사항은 무엇인가요?

**핵심 답변:** 엣지 배포는 **불안정한 네트워크, 제한된 대역폭, 롤백의 어려움**이라는 세 가지 근본적 도전이 있습니다.

차분 업데이트(Delta Update)가 핵심입니다. 전체 이미지를 전송하는 대신 변경된 레이어만 전송합니다. OSTree나 Docker layer diff를 활용합니다. 배포 전 체크섬 검증이 필수적이며, 부분적으로 다운로드된 업데이트로 인한 손상을 방지해야 합니다.

Over-the-Air(OTA) 업데이트에서 **A/B 파티션 전략**이 중요합니다. 새 버전을 비활성 파티션에 기록하고, 검증 후 활성화합니다. 차량처럼 롤백이 물리적으로 어려운 환경에서는 업데이트 실패 시 이전 파티션으로 자동 부팅하는 기능이 안전상 필수입니다. 배포 순서도 중요합니다. 일반 앱이 아닌 안전 시스템(브레이크, 스티어링)은 별도의 인증과 검증 파이프라인을 거쳐야 하며 ISO 26262 기능 안전 표준을 준수해야 합니다.

---

## Q19. Helm Chart에서 SubChart(Dependencies)를 관리할 때 버전 고정과 유연성 사이의 균형을 어떻게 유지하시나요?

**핵심 답변:** `Chart.yaml`의 `dependencies`에서 버전을 `~1.2.0`(패치 버전만 유동)으로 고정하는 것이 기본입니다. `^1.0.0`처럼 마이너 버전까지 유동으로 두면 예기치 않은 API 변경에 취약합니다.

`Chart.lock` 파일은 반드시 Git에 커밋해야 합니다. 이것이 의존성의 정확한 버전을 고정하며 재현 가능한 빌드를 보장합니다. `helm dependency update` 실행 시에만 lock 파일이 변경됩니다.

자체 관리하는 내부 차트(데이터베이스, 캐시 등)는 Helm 레포지토리에 버전을 명시적으로 고정하고, Renovate Bot이나 Dependabot을 통해 의존성 업데이트 PR을 자동 생성하도록 설정합니다. 이렇게 하면 보안 취약점이 있는 서브차트를 빠르게 업데이트하면서도 의도치 않은 자동 업그레이드를 방지합니다.

---

## Q20. Kubernetes에서 Horizontal Pod Autoscaler(HPA)와 Cluster Autoscaler, KEDA의 상호 작용을 설명하고, CI/CD 파이프라인에서 이 세 가지를 어떻게 고려해야 하나요?

**핵심 답변:** 세 컴포넌트는 **서로 다른 스케일링 레이어**에서 동작합니다. HPA는 기존 노드 내에서 Pod 수를 조정하고, Cluster Autoscaler는 노드 수를 조정하며, KEDA는 이벤트 기반 커스텀 메트릭으로 HPA를 확장합니다.

CI/CD 파이프라인 설계 시 고려사항으로는, 새 버전 배포 시 Rolling Update가 진행되는 동안 HPA가 동시에 스케일 다운을 시도할 수 있습니다. 이를 방지하려면 배포 중에는 HPA의 minReplicas를 일시적으로 높이거나, Deployment 어노테이션에 스케일링 일시 중지를 표시하는 패턴이 있습니다. 또한 Cluster Autoscaler가 새 노드를 프로비저닝하는 데 1-5분이 소요되므로, 배포 타임아웃을 충분히 여유있게 설정해야 합니다. KEDA를 사용할 때는 카나리 배포 중 이벤트 소스(메시지 큐 등)가 카나리와 스테이블 파드에 어떻게 분산되는지 명확히 이해해야 합니다.

---

## Q21. GitHub Actions에서 환경(Environment)을 활용한 배포 승인 워크플로우를 설계할 때, 대기업의 컴플라이언스 요구사항(SOC2, ISO27001)을 어떻게 충족시키나요?

**핵심 답변:** GitHub Environments는 **배포 승인 게이트와 감사 로그**를 제공하는 네이티브 메커니즘입니다. Production 환경에 `required_reviewers`를 설정하면 모든 프로덕션 배포가 사람의 승인을 받아야 합니다.

SOC2 Type II와 ISO27001에서 요구하는 **변경 관리(Change Management) 증거**로는 GitHub Environments의 배포 히스토리가 감사 로그로 활용됩니다. 승인자 정보, 승인 시간, 배포된 커밋이 자동으로 기록됩니다. 추가적으로, Deployment Protection Rules를 통해 커스텀 webhook 서버를 호출하여 ITSM 시스템(ServiceNow, Jira)과 연동할 수 있습니다. 이렇게 하면 변경 티켓 번호가 있는 경우에만 배포가 허용되는 컴플라이언스 게이트를 구현할 수 있습니다.

---

## Q22. Docker 이미지 빌드에서 멀티 스테이지 빌드(Multi-stage Build)를 사용할 때의 보안 및 성능 이점을 설명하고, 빌드 시간을 최소화하기 위한 레이어 캐시 전략을 상세히 설명하세요.

**핵심 답변:** 멀티 스테이지 빌드의 핵심 보안 이점은 **공격 표면(attack surface) 축소**입니다. 최종 프로덕션 이미지에는 빌드 도구(gcc, make, npm)가 포함되지 않아 이미지 크기가 줄고 CVE 노출이 감소합니다. Python 예시에서 빌드 스테이지에만 있는 `pip`와 `gcc`가 최종 이미지에는 없습니다.

레이어 캐시 최적화의 핵심 원칙은 **변경 빈도가 낮은 레이어를 먼저 배치**하는 것입니다. 의존성 파일(`requirements.txt`, `package.json`)을 소스 코드보다 먼저 COPY하면, 소스 코드 변경 시에도 의존성 레이어는 캐시 히트가 됩니다. BuildKit의 `--mount=type=cache`를 사용하면 레이어 캐시와 별개로 빌드 도구의 캐시 디렉토리를 영구적으로 유지할 수 있습니다. 이것은 특히 npm, pip, cargo처럼 의존성을 인터넷에서 다운로드하는 도구에서 극적인 빌드 시간 단축 효과를 냅니다.

---

## Q23. Google의 Site Reliability Engineering 원칙에서 Error Budget을 CI/CD 파이프라인과 어떻게 연동하시겠습니까?

**핵심 답변:** Error Budget 기반 배포 게이팅은 SLO(Service Level Objective) 기반 CI/CD의 핵심입니다. 아이디어는 단순합니다: 서비스의 Error Budget이 소진되고 있을 때는 새로운 배포를 자동으로 차단합니다. 신규 기능 배포는 추가 리스크를 수반하기 때문입니다.

구현 방법으로는, 배포 파이프라인의 프로덕션 배포 단계 앞에 **Error Budget Check** 스텝을 추가합니다. Prometheus나 Datadog에서 지난 7일간의 Error Budget 소비율을 쿼리하고, 소비율이 20% 이상이면 배포를 차단하거나 추가 승인을 요구합니다. 이것은 SRE 팀과 개발팀 사이의 **자동화된 신뢰 계약**입니다. "Error Budget이 여유로울 때는 빠르게 배포하고, 없을 때는 안정화에 집중한다"는 SRE의 핵심 철학을 파이프라인에 코드로 표현한 것입니다.

---

## Q24. Amazon의 "Two-Pizza Team" 원칙에 따라 독립적인 팀들이 각자의 마이크로서비스를 배포할 때, 플랫폼 팀이 제공해야 할 황금 경로(Golden Path)와 가드레일(Guardrail)을 어떻게 설계하시겠습니까?

**핵심 답변:** 황금 경로는 **팀이 올바른 방식이 가장 쉬운 방식이 되도록** 설계되어야 합니다. 플랫폼 팀은 세 가지를 제공해야 합니다.

첫째, **서비스 카탈로그와 프로젝트 스캐폴딩**입니다. `cookiecutter` 또는 Backstage 템플릿으로 새 서비스 생성 시 표준 CI/CD 파이프라인, 표준 Helm Chart, 표준 모니터링 설정이 자동으로 포함됩니다. 개발팀은 비즈니스 로직만 작성하면 됩니다.

둘째, **Policy as Code 가드레일**입니다. OPA Gatekeeper나 Kyverno로 최소 리소스 제한, 이미지 서명 검증, 승인된 레지스트리만 사용 등을 클러스터 레벨에서 자동 강제합니다. 팀이 실수로 정책을 우회하는 것을 구조적으로 불가능하게 만드는 것이 목표입니다.

셋째, **셀프 서비스 플랫폼**입니다. 팀이 플랫폼 팀의 승인 없이도 새 환경을 만들고, 배포하고, 모니터링을 설정할 수 있어야 합니다. 그렇지 않으면 플랫폼 팀이 병목(bottleneck)이 됩니다.

---

## Q25. Nvidia와 같이 GPU 집약적 워크로드를 위한 CI/CD 파이프라인에서 ML 모델 훈련과 추론 서비스 배포의 특수한 고려사항은 무엇인가요?

**핵심 답변:** ML CI/CD(MLOps)의 일반 CI/CD와의 근본적 차이는 **코드뿐만 아니라 데이터와 모델도 버전 관리**가 되어야 한다는 점입니다.

파이프라인은 세 단계로 나뉩니다: 데이터 검증 → 모델 훈련 및 평가 → 서빙 배포. 데이터 드리프트 감지(Great Expectations, Evidently AI)를 파이프라인 초입에 두어 데이터 품질 문제를 조기에 차단해야 합니다.

모델 레지스트리(MLflow, AWS SageMaker Model Registry)는 모델 아티팩트, 메트릭, 하이퍼파라미터를 함께 버전 관리합니다. CI/CD 파이프라인에서 새 모델 버전의 AUC, F1 Score 등 KPI가 현재 프로덕션 모델을 초과할 때만 배포를 승인하는 **자동화된 챔피언/챌린저 비교**를 구현합니다.

GPU 특유의 고려사항으로는, 추론 서비스의 카나리 배포 시 GPU 메모리 누수가 점진적으로 나타날 수 있으므로 일반 CPU 메트릭 외에 DCGM(Data Center GPU Manager) 메트릭도 롤백 트리거로 활용해야 합니다.

---

## Q26. 쿠버네티스 클러스터 업그레이드(버전 업)를 CI/CD 파이프라인으로 자동화할 때 어떤 전략을 사용하시겠습니까?

**핵심 답변:** 클러스터 업그레이드는 일반 애플리케이션 배포보다 훨씬 높은 위험을 수반합니다. 전략의 핵심은 **Blue-Green Cluster 전략**입니다.

현재 운영 중인 클러스터(Blue)와 동일한 구성으로 새 버전 클러스터(Green)를 프로비저닝합니다. ArgoCD의 멀티 클러스터 기능을 활용하여 Green 클러스터에 모든 워크로드를 배포합니다. 카나리 트래픽(5%)을 Green으로 전환하고 메트릭을 관찰합니다. 문제가 없으면 점진적으로 100%로 전환하고 Blue를 종료합니다.

이 방법이 불가능한 경우(비용, 스테이트풀 워크로드 등) 인플레이스 업그레이드 전략을 사용합니다. 이때 컨트롤 플레인을 먼저 업그레이드하고(AKS/EKS는 관리형으로 자동), 노드는 `surge upgrade`(Blue-Green과 유사하게 새 노드 추가 후 구 노드 드레인)로 처리합니다. 모든 과정을 Terraform 또는 Pulumi로 코드화하여 재현 가능하게 만들어야 합니다.

---

## Q27. CI/CD 파이프라인의 관찰 가능성(Observability)을 어떻게 구현하시나요? 파이프라인 자체를 어떻게 모니터링하시나요?

**핵심 답변:** 파이프라인 관찰 가능성은 종종 간과되지만, 대규모 조직에서 파이프라인 자체가 핵심 인프라입니다.

지표(Metrics) 측면에서는 **DORA 메트릭**(배포 빈도, 변경 리드 타임, 변경 실패율, 복구 시간)을 측정하는 것이 가장 중요합니다. GitHub Actions의 경우 Webhook 이벤트를 수집하여 파이프라인 실행 시간, 성공/실패율, 큐 대기 시간을 Prometheus로 스크래핑할 수 있습니다(GitHub Actions Exporter).

추적(Tracing) 측면에서 OpenTelemetry를 파이프라인에 통합하면 단일 배포의 소스 커밋부터 프로덕션 배포까지의 전체 trace를 Jaeger나 Grafana Tempo에서 확인할 수 있습니다. 어느 단계에서 시간이 많이 소요되는지 병목 분석이 가능합니다.

로그(Logs) 측면에서는 배포 이벤트를 구조화된 로그로 중앙 집중화하고, Kubernetes 감사 로그와 상관시키면 "누가, 언제, 무엇을 배포했는지"를 추적할 수 있습니다.

---

## Q28. Microsoft Azure DevOps와 GitHub Actions를 동시에 사용하는 하이브리드 환경에서 CI/CD를 어떻게 통합 설계하시겠습니까?

**핵심 답변:** 마이그레이션 중이거나 레거시 시스템이 Azure DevOps를 사용하는 경우 하이브리드 환경이 일반적입니다. 핵심 설계 원칙은 **두 시스템이 중복 기능을 수행하지 않도록** 책임을 명확히 분리하는 것입니다.

권장 패턴은 GitHub Actions가 CI(빌드, 테스트, 이미지 빌드)를 담당하고, Azure DevOps Pipelines가 CD(특히 Azure 리소스 배포, Azure Artifacts 연동, Azure Test Plans 통합)를 담당하는 **하이브리드 CI/CD** 모델입니다. 두 시스템은 컨테이너 이미지 태그(또는 다이제스트)를 공유 상태로 사용하여 연결됩니다. GitHub Actions가 이미지를 빌드하고 ACR에 푸시하면, Azure DevOps Pipeline이 ACR 업데이트 이벤트를 트리거로 배포를 시작합니다.

장기적으로는 단일 플랫폼으로 마이그레이션하는 로드맵을 수립하는 것이 유지보수성 측면에서 필요하며, GitHub Actions + Azure로의 통합이 현재 Microsoft의 권장 방향입니다.

---

## Q29. ArgoCD에서 Secret 관리를 위해 Sealed Secrets vs External Secrets Operator를 어떻게 선택하시나요? 각 접근법의 보안 모델 차이를 설명하세요.

**핵심 답변:** 두 접근법의 근본적인 차이는 **비밀의 위치**입니다. Sealed Secrets는 암호화된 시크릿을 Git에 저장하고(kubeseal로 암호화), 클러스터 내 컨트롤러가 복호화합니다. 비밀이 Git 기록에 존재하며(암호화된 형태로), Sealed Secrets 컨트롤러의 개인 키가 손상되면 이전 커밋의 모든 시크릿도 노출됩니다. External Secrets Operator는 실제 시크릿 값을 Git에 저장하지 않고, 외부 비밀 저장소(AWS Secrets Manager, Azure Key Vault, HashiCorp Vault)에 대한 참조(reference)만 Git에 저장합니다. 비밀 로테이션, 감사, 세밀한 접근 제어가 외부 저장소에서 중앙 관리됩니다.

기업 환경에서는 External Secrets Operator가 더 권장됩니다. 이미 존재하는 비밀 관리 인프라(Vault, Secrets Manager)와 통합되고, 비밀 로테이션이 Kubernetes 재배포 없이 가능하며, 감사 로그가 비밀 저장소에서 중앙 관리됩니다. Sealed Secrets는 단순성이 장점으로, 외부 의존성 없이 Git만으로 전체 시스템을 관리할 수 있어 소규모 팀에 적합합니다.

---

## Q30. Apple과 같은 엄격한 보안 요구사항이 있는 환경에서 Supply Chain Attack을 방어하기 위해 CI/CD 파이프라인에 어떤 레이어를 추가하시겠습니까?

**핵심 답변:** Supply Chain Attack은 빌드 파이프라인의 어느 지점에서나 발생할 수 있습니다. 소스 코드 → 의존성 → 빌드 환경 → 이미지 레지스트리 → 배포 각 단계마다 방어 레이어가 필요합니다.

소스 코드 레이어에서는 **서명된 커밋 강제화**(GPG 키 또는 SSH 서명)와 **비밀 스캐닝**(GitHub Secret Scanning, git-secrets)이 필요합니다.

의존성 레이어에서는 **Private Mirror Registry**를 운영하여 모든 의존성(npm, pip, Maven)이 승인된 내부 레지스트리를 통해서만 다운로드됩니다. 의존성 검증(Subresource Integrity, 패키지 해시 고정)과 취약점 스캐닝(Snyk, Dependabot, OWASP Dependency Check)을 파이프라인에 통합합니다.

빌드 환경 레이어에서는 **Hermetic Build**(외부 인터넷 접근 없는 빌드 환경)를 구현하고, 빌드 컨테이너 이미지 자체도 서명 검증 후 사용합니다.

이미지 레지스트리 레이어에서는 **Notary v2/cosign 기반 이미지 서명**, **OPA/Kyverno로 서명되지 않은 이미지 배포 차단**, **Trivy/Grype로 CVE 스캔 결과가 게이팅 조건**이 됩니다.

배포 레이어에서는 **Kubernetes Admission Controller**가 최후 방어선으로, 서명 검증을 통과하지 못한 이미지는 클러스터에 절대 배포되지 않습니다.

---

## Q31. ArgoCD + GitHub Actions를 통합한 CI/CD 파이프라인에서 Pull Request Preview Environment(PR별 임시 환경)를 어떻게 구현하시나요?

**핵심 답변:** PR Preview Environment는 개발자 생산성을 극적으로 향상시키는 패턴으로, 각 PR이 독립된 쿠버네티스 네임스페이스에 완전한 애플리케이션 스택을 배포합니다.

구현 흐름은 다음과 같습니다. GitHub Actions가 PR 오픈 이벤트를 감지하여 컨테이너 이미지를 빌드하고 레지스트리에 푸시합니다. 이어서 `gitops-config` 저장소에 `pr-{PR_NUMBER}` 디렉토리를 생성하고 커스텀 values(이미지 태그, 인그레스 호스트명 `pr-123.preview.example.com`)를 커밋합니다. ArgoCD의 ApplicationSet이 이 새 디렉토리를 감지하여 자동으로 Application을 생성합니다. PR 코멘트에 미리보기 URL을 자동으로 게시합니다. PR이 닫히면 GitHub Actions가 디렉토리를 삭제하고, ArgoCD가 Application과 네임스페이스를 자동으로 정리합니다. 비용 최적화를 위해 PR 환경은 야간에 자동으로 파드를 0으로 스케일 다운하고 낮에 복원하는 스케줄을 적용합니다.

---

## Q32. Kubernetes에서 StatefulSet 업데이트 시 Rolling Update의 한계와 이를 극복하기 위한 전략은 무엇인가요?

**핵심 답변:** StatefulSet은 Deployment와 달리 파드가 순서가 있고(pod-0, pod-1...) 각자 고유한 영구 스토리지를 가집니다. Rolling Update 시 `pod-n`부터 `pod-0` 순서로 역순으로 업데이트되며, 각 파드가 `Ready` 상태가 되어야 다음 파드를 업데이트합니다.

이것의 핵심 한계는 **복잡한 스테이트풀 애플리케이션(예: Cassandra, Kafka, etcd)에서 리더 선출 재조정으로 인한 다운타임**입니다. Kafka에서 `pod-0`(리더 브로커)이 업데이트되면 파티션 리더 선출이 재조정되고, 이 시간 동안 일부 파티션이 일시적으로 오프라인 상태가 됩니다.

Partition 기반 업데이트로 이를 완화할 수 있습니다. `spec.updateStrategy.rollingUpdate.partition: 2`로 설정하면 pod-2 이상만 업데이트됩니다. 단계적으로 partition 값을 낮추면서 검증 후 진행하는 **카나리 유사 업데이트**가 가능합니다. 또한 파드 중단 예산(PDB)을 신중하게 설정하여 최소 quorum(예: Kafka의 경우 ISR(In-Sync Replicas) 과반수)을 항상 유지해야 합니다.

---

# 마무리: 면접 전략과 사고 프레임워크

이 30개 이상의 질문들을 관통하는 핵심 원칙이 있습니다. **트레이드오프 기반 사고**입니다. 대기업 Senior 면접에서 "이 방법이 최고입니다"라는 답변보다 "이 상황에서는 A가 낫고, 저 상황에서는 B가 낫습니다. 그 이유는..."이라는 답변이 훨씬 높은 평가를 받습니다.

또한 설계 결정을 이야기할 때는 항상 **실패 모드(failure mode)와 회복 전략(recovery strategy)**을 함께 언급하는 것이 박사급 사고의 표지입니다. "이 시스템이 실패하면 어떻게 됩니까?"에 대한 명확한 답변이 Senior Engineer와 Staff/Principal Engineer를 구분하는 기준이 됩니다.
