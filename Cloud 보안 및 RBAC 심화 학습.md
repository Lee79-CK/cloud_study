# AWS · Azure · Kubernetes 보안 체계 및 RBAC 심화 가이드
### Senior Cloud Engineer를 위한 박사급 참고 자료

---

## 목차

1. [클라우드 보안의 철학적 기반: Zero Trust Architecture](#1)
2. [AWS 보안 체계 심화](#2)
3. [Azure 보안 체계 심화](#3)
4. [Kubernetes 보안 체계 심화](#4)
5. [RBAC 설계 원칙 및 고급 패턴](#5)
6. [멀티클라우드 통합 보안 아키텍처](#6)
7. [대기업 면접 Q&A 30+](#7)

---

## 1. 클라우드 보안의 철학적 기반: Zero Trust Architecture {#1}

### 1.1 Zero Trust의 인식론적 전환

전통적인 경계 기반 보안(Perimeter Security)은 "내부는 안전하다"는 암묵적 신뢰를 전제로 합니다. 이는 성벽 안에 있으면 신뢰한다는 중세적 패러다임과 동일합니다. 반면 Zero Trust는 **"Never Trust, Always Verify"** 원칙 아래, 네트워크 위치와 무관하게 모든 접근 요청을 지속적으로 검증합니다.

Google의 BeyondCorp 연구(2014)가 기폭제가 되었고, NIST SP 800-207(2020)이 표준화 프레임워크를 제시했습니다. Zero Trust의 7대 원칙은 다음과 같습니다.

첫째, 모든 데이터 소스와 컴퓨팅 서비스는 리소스로 간주됩니다. 둘째, 네트워크 위치와 무관하게 모든 통신은 보호되어야 합니다. 셋째, 개별 엔터프라이즈 리소스에 대한 접근은 세션 단위로 부여됩니다. 넷째, 리소스 접근은 클라이언트 아이덴티티, 애플리케이션, 자산 상태, 기타 행동 속성을 포함하는 동적 정책으로 결정됩니다. 다섯째, 엔터프라이즈는 모든 소유 및 연관 자산의 무결성과 보안 상태를 모니터링합니다. 여섯째, 모든 리소스 인증과 인가는 동적으로 이루어지며 접근 허용 전 엄격히 실행됩니다. 일곱째, 엔터프라이즈는 자산, 네트워크 인프라, 통신에 대한 가능한 많은 정보를 수집하여 보안 태세를 개선합니다.

### 1.2 Policy Decision Point (PDP) vs Policy Enforcement Point (PEP)

Zero Trust 아키텍처에서 PDP는 접근 허용 여부를 결정하는 두뇌 역할을 합니다. AWS의 경우 IAM Policy Evaluation Engine이 PDP에 해당하고, Azure의 경우 Conditional Access Policy Engine이 이 역할을 합니다. Kubernetes에서는 OPA(Open Policy Agent)나 Kyverno가 PDP 역할을 수행합니다.

PEP는 PDP의 결정을 실제로 집행하는 역할로, AWS의 VPC Security Group, Azure의 NSG, Kubernetes의 Network Policy가 이에 해당합니다. 이 두 컴포넌트의 명확한 분리가 고급 보안 아키텍처의 핵심입니다.

---

## 2. AWS 보안 체계 심화 {#2}

### 2.1 IAM의 내부 동작 원리: Policy Evaluation Logic

AWS IAM의 정책 평가 로직은 단순히 "Allow/Deny"를 반환하는 것이 아니라 6단계의 계층적 평가 프로세스를 거칩니다.

**1단계: Implicit Deny** — 기본적으로 모든 요청은 거부됩니다. 어떠한 정책도 없으면 요청은 차단됩니다.

**2단계: SCP(Service Control Policy) 평가** — AWS Organizations의 SCP가 Allow를 명시하지 않으면, 하위 계정의 어떠한 정책도 해당 권한을 부여할 수 없습니다. SCP는 권한의 천장(ceiling)을 정의합니다. 이 개념이 매우 중요한데, SCP는 "부여"가 아니라 "제한"입니다.

**3단계: Resource-based Policy** — S3 버킷 정책, KMS 키 정책, SQS 큐 정책 등이 여기에 해당합니다. 교차 계정(Cross-Account) 접근의 경우, 리소스 기반 정책과 아이덴티티 기반 정책 모두에서 Allow가 있어야 접근이 허용됩니다.

**4단계: Permission Boundary** — IAM 엔티티(사용자, 역할)에 설정된 권한 경계입니다. 실제 부여된 권한과 Permission Boundary의 교집합만이 유효 권한이 됩니다. 이를 수식으로 표현하면 `Effective Permissions = Identity Policy ∩ Permission Boundary`입니다.

**5단계: Session Policy** — `sts:AssumeRole` 호출 시 인라인으로 전달되는 정책입니다. 임시 자격증명의 권한을 세션 수준에서 추가 제한할 수 있어, 최소 권한 원칙(PoLP)의 동적 구현에 활용됩니다.

**6단계: Identity-based Policy** — 마지막으로 IAM 사용자나 역할에 직접 부여된 정책이 평가됩니다.

```
최종 결정 = NOT(명시적 Deny 존재) AND (모든 층위에서 Allow 존재)
```

### 2.2 AWS IAM Identity Center (SSO)와 엔터프라이즈 IdP 통합

대규모 조직에서 개별 IAM 사용자를 생성하는 것은 안티패턴입니다. IAM Identity Center는 외부 IdP(Okta, Azure AD, Ping Identity 등)와 SAML 2.0 또는 SCIM(System for Cross-domain Identity Management)을 통해 통합됩니다.

SCIM의 핵심 가치는 IdP에서 사용자가 해제(offboard)될 때 자동으로 AWS의 접근 권한도 제거된다는 점입니다. 이는 수동 관리의 지연으로 인한 보안 취약점을 원천 차단합니다.

Permission Set은 SAML 역할 매핑의 고급 추상화로, 하나의 Permission Set을 여러 AWS 계정에 다중 할당(Multi-Account Assignment)할 수 있습니다. 이는 계정이 수백 개인 대기업 환경에서 운영 효율성을 극적으로 향상시킵니다.

```json
// Permission Set 예시: 최소 권한 DevOps 역할
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "ap-northeast-2"]
        }
      }
    }
  ]
}
```

### 2.3 AWS STS와 역할 체인(Role Chaining)의 보안 함의

역할 체인은 `RoleA`가 `AssumeRole`을 통해 `RoleB`를 수임하고, `RoleB`가 다시 `RoleC`를 수임하는 패턴입니다. AWS는 역할 체인 시 임시 자격증명의 최대 세션 시간을 1시간으로 제한합니다(단일 수임은 최대 12시간). 이는 장기 실행 파이프라인 설계 시 반드시 고려해야 합니다.

또한 역할 체인에서는 `aws:SourceAccount`, `aws:SourceArn` 조건 키를 활용하여 Confused Deputy 공격을 방지해야 합니다. Confused Deputy는 권한이 높은 서비스가 의도치 않게 낮은 권한의 주체를 대신하여 자원에 접근하는 취약점입니다.

```json
// Confused Deputy 방지 조건 예시
{
  "Effect": "Allow",
  "Principal": {
    "Service": "lambda.amazonaws.com"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "ArnLike": {
      "aws:SourceArn": "arn:aws:lambda:ap-northeast-2:123456789012:function:my-function"
    },
    "StringEquals": {
      "aws:SourceAccount": "123456789012"
    }
  }
}
```

### 2.4 AWS Organizations와 다중 계정 거버넌스

현대적 AWS 아키텍처는 단일 계정이 아니라 Landing Zone 패턴을 따릅니다. AWS Control Tower는 이를 자동화하는 서비스로, 다음의 계정 구조를 권장합니다.

Management Account는 Organizations 관리와 SCP 적용만을 담당하며, 실제 워크로드는 절대 실행하지 않습니다. Log Archive Account는 모든 계정의 CloudTrail, Config 로그를 중앙화합니다. Security Tooling Account는 GuardDuty, Security Hub, Inspector의 위임 관리자(Delegated Admin) 역할을 합니다.

SCP 설계에서 중요한 원칙은 "Allowlist vs Denylist" 전략입니다. 민감한 계정(Production)에는 Allowlist 방식으로 허용된 서비스만 명시하고, 개발 계정에는 Denylist 방식으로 위험한 작업만 차단하는 방식이 효율적입니다.

```json
// Guardrails SCP: 루트 사용자 액션 방지
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    },
    {
      "Sid": "DenyLeaveOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

### 2.5 AWS KMS와 암호화 컨텍스트(Encryption Context)

KMS는 단순한 키 관리 서비스가 아니라 암호화 워크플로우의 중앙 오케스트레이터입니다. 암호화 컨텍스트(Encryption Context)는 추가 인증 데이터(AAD, Additional Authenticated Data)로 기능하여, 복호화 시 동일한 컨텍스트가 제공되어야만 성공합니다. 이를 통해 데이터의 문맥적 무결성(Contextual Integrity)을 보장합니다.

```python
import boto3

kms = boto3.client('kms')

# 암호화 컨텍스트로 데이터의 "목적"을 명시
encryption_context = {
    'Environment': 'production',
    'Application': 'payment-service',
    'DataClassification': 'PII'
}

# 암호화 시 컨텍스트 포함
ciphertext = kms.encrypt(
    KeyId='arn:aws:kms:ap-northeast-2:123456789012:key/...',
    Plaintext=b'sensitive_data',
    EncryptionContext=encryption_context
)

# 복호화 시 동일한 컨텍스트 없으면 실패
plaintext = kms.decrypt(
    CiphertextBlob=ciphertext['CiphertextBlob'],
    EncryptionContext=encryption_context  # 반드시 일치해야 함
)
```

KMS의 키 정책(Key Policy)은 IAM 정책과 독립적으로 작동합니다. 중요한 점은 KMS 키 정책이 해당 키에 대한 IAM 정책 평가를 명시적으로 활성화해야 한다는 것입니다. `kms:EnableIAMUserPermissions` 원칙을 설정하지 않으면 계정 루트를 제외한 어떠한 IAM 엔티티도 키에 접근할 수 없습니다.

### 2.6 VPC와 네트워크 보안의 심층 분석

Security Group은 Stateful 방화벽으로, 반환 트래픽을 자동으로 허용합니다. 반면 Network ACL은 Stateless로 인바운드와 아웃바운드를 독립적으로 평가합니다. 이 차이는 대규모 트래픽 제어에서 중요한 설계 결정 요소입니다.

VPC Endpoint를 통한 PrivateLink 아키텍처는 인터넷을 경유하지 않고 AWS 서비스에 접근하는 표준 패턴입니다. Gateway Endpoint(S3, DynamoDB)와 Interface Endpoint(대부분의 서비스)의 차이를 이해하는 것이 중요합니다. Gateway Endpoint는 라우팅 테이블 기반이고, Interface Endpoint는 ENI(Elastic Network Interface) 기반이어서 보안 그룹을 적용할 수 있습니다.

---

## 3. Azure 보안 체계 심화 {#3}

### 3.1 Azure Active Directory(Entra ID)와 인증 프로토콜

Azure AD는 단순한 디렉토리 서비스가 아니라 클라우드 네이티브 IAM 플랫폼입니다. 핵심 인증 프로토콜로 OpenID Connect(OIDC), OAuth 2.0, SAML 2.0, WS-Federation을 지원합니다.

Microsoft Identity Platform의 토큰 구조를 이해하는 것이 핵심입니다. Access Token은 리소스 접근에 사용되고, ID Token은 사용자 인증에 사용됩니다. Refresh Token은 새로운 Access Token을 발급받는 데 사용됩니다. 

토큰의 수명 주기 관리에서 CAE(Continuous Access Evaluation)는 혁신적입니다. 기존 토큰 기반 인증은 토큰 만료 전까지 취소가 불가능했으나, CAE는 사용자 계정 비활성화, 비밀번호 변경, 세션 취소 이벤트를 실시간으로 리소스 서버에 전파하여 접근을 즉시 차단합니다.

### 3.2 Azure RBAC의 계층 구조와 상속 메커니즘

Azure RBAC는 Management Group → Subscription → Resource Group → Resource의 4계층 범위(Scope) 구조에서 작동합니다. 상위 범위에서 부여된 역할은 하위 범위에 상속됩니다. 이 상속 메커니즘은 대규모 조직 거버넌스에 강력하지만, 과도한 권한 상속을 방지하는 설계가 중요합니다.

역할 정의(Role Definition)는 Actions, NotActions, DataActions, NotDataActions의 4가지 구성요소로 이루어집니다. Actions는 관리 평면(ARM API) 작업이고, DataActions는 데이터 평면(Storage, Cosmos DB 등) 작업입니다. 이 둘의 분리는 관리자가 리소스를 관리하더라도 데이터에는 접근하지 못하도록 하는 정밀한 제어를 가능하게 합니다.

```json
// 커스텀 역할 정의 예시: 읽기 전용 + 특정 데이터 작업 허용
{
  "Name": "DataReader-StorageBlob",
  "Description": "Storage 관리는 불가하나 Blob 데이터 읽기 가능",
  "Actions": [
    "Microsoft.Storage/storageAccounts/read",
    "Microsoft.Storage/storageAccounts/listKeys/action"
  ],
  "NotActions": [],
  "DataActions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"
  ],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/{subscription-id}/resourceGroups/data-rg"
  ]
}
```

### 3.3 Azure Conditional Access: 동적 접근 제어의 구현

Conditional Access는 Zero Trust의 실질적 구현체로, 접근 요청의 신호(Signal)를 기반으로 접근 제어(Control)를 동적으로 적용합니다.

신호(Signal)에는 사용자 및 그룹 멤버십, IP 위치 정보, 디바이스 컴플라이언스 상태(Intune), 애플리케이션, 실시간 리스크 신호(Identity Protection)가 포함됩니다. 제어(Control)에는 MFA 요구, 컴플라이언트 디바이스 요구, 접근 차단, 세션 제한이 있습니다.

Named Location을 활용한 IP 기반 제어보다 Hybrid Azure AD Join 또는 Intune Compliant 디바이스 기반 제어가 더 강력합니다. IP는 VPN으로 우회 가능하지만 디바이스 인증서는 위조가 어렵기 때문입니다.

```
// Conditional Access 정책 설계 원칙
신호: (사용자 = 관리자 그룹) AND (위치 = 신뢰되지 않은 IP) AND (디바이스 = 비관리 디바이스)
제어: 접근 차단

신호: (사용자 = 일반 직원) AND (앱 = Microsoft 365)
제어: MFA 요구 + 세션 4시간 제한
```

### 3.4 Azure Managed Identity와 워크로드 아이덴티티

Managed Identity는 Azure 리소스에 자동으로 관리되는 아이덴티티를 부여합니다. 시크릿 없는 인증(Secretless Authentication)의 핵심 메커니즘으로, IMDS(Instance Metadata Service)를 통해 토큰을 발급받습니다.

System-assigned Managed Identity는 리소스와 수명 주기를 공유하고, User-assigned Managed Identity는 독립적으로 존재하여 여러 리소스에 공유 가능합니다. 대규모 환경에서는 User-assigned가 권한 재사용과 감사 추적에 유리합니다.

Workload Identity Federation은 한 걸음 더 나아가, Azure 외부의 워크로드(GitHub Actions, Kubernetes, AWS Lambda 등)가 OIDC 토큰을 통해 Azure AD에서 토큰을 교환받을 수 있습니다. 이로써 외부 워크로드에서도 비밀 없이 Azure 리소스에 접근할 수 있습니다.

```yaml
# GitHub Actions에서 Azure Workload Identity Federation 사용
- name: Azure Login (No Secrets!)
  uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    # client-secret 불필요! OIDC 토큰 자동 사용
```

### 3.5 Azure Privileged Identity Management (PIM)

PIM은 Just-in-Time(JIT) 권한 접근 패턴의 구현체입니다. 영구적 고권한 역할 할당(Permanent Privileged Assignment)을 최소화하고, 필요 시 활성화(Activation)하는 방식으로 공격 표면을 최소화합니다.

PIM의 핵심 기능은 활성화 요구사항 구성(MFA, 승인 워크플로우, 활성화 사유 기록), 시간 제한 활성화(1~72시간), 활성화 알림 및 감사 로그입니다. 이는 AWS의 Session Policy와 유사하지만 워크플로우 자동화가 내장된 형태입니다.

---

## 4. Kubernetes 보안 체계 심화 {#4}

### 4.1 Kubernetes 보안의 4C 계층 모델

Kubernetes 보안은 Cloud, Cluster, Container, Code의 4계층으로 구성됩니다. 각 계층은 독립적으로 보호되어야 하며, 하위 계층의 침해는 상위 계층 보호를 무력화합니다. 예를 들어 etcd에 암호화 없이 시크릿이 저장된다면(Cloud/Cluster 층 취약점) 아무리 강력한 RBAC 정책을 구성해도 직접 etcd 접근으로 시크릿을 탈취할 수 있습니다.

### 4.2 Kubernetes RBAC의 완전한 이해

Kubernetes RBAC는 4가지 API 오브젝트로 구성됩니다.

**Role**은 특정 네임스페이스 내의 리소스에 대한 권한을 정의합니다. **ClusterRole**은 클러스터 전체 또는 네임스페이스를 초월하는 리소스(Node, PV, CRD 등)에 대한 권한을 정의합니다. **RoleBinding**은 Role이나 ClusterRole을 특정 네임스페이스 내의 주체(Subject)에 바인딩합니다. **ClusterRoleBinding**은 ClusterRole을 클러스터 전체에 걸쳐 바인딩합니다.

중요한 설계 패턴으로, ClusterRole을 정의하고 RoleBinding으로 특정 네임스페이스에만 바인딩하면, 동일한 권한 정의를 여러 네임스페이스에서 재사용할 수 있어 역할 정의의 중복을 피할 수 있습니다.

```yaml
# ClusterRole 정의: 재사용 가능한 Pod 뷰어 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "watch", "list"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: []  # exec 명시적 금지
---
# RoleBinding: 특정 네임스페이스에만 적용
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production  # 이 네임스페이스에만 적용
subjects:
  - kind: ServiceAccount
    name: monitoring-agent
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4.3 ServiceAccount와 워크로드 아이덴티티의 보안 강화

ServiceAccount는 파드에 부여되는 Kubernetes 아이덴티티입니다. 기본적으로 Kubernetes는 `default` ServiceAccount를 모든 파드에 마운트하는데, 이는 보안 안티패턴입니다. 반드시 `automountServiceAccountToken: false`를 기본 설정으로 하고, 필요한 파드에만 명시적으로 마운트해야 합니다.

IRSA(IAM Roles for Service Accounts)는 EKS에서 ServiceAccount에 AWS IAM 역할을 연결하는 메커니즘입니다. 내부적으로는 OIDC(OpenID Connect)를 활용합니다. EKS 클러스터가 OIDC 제공자로 등록되면, Kubernetes가 발행한 ServiceAccount 토큰을 AWS STS가 검증하고 임시 자격증명을 교환해줍니다.

```yaml
# IRSA 설정: ServiceAccount에 IAM 역할 연결
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3ReaderRole
    eks.amazonaws.com/token-expiration: "3600"  # 1시간 토큰 만료
```

Azure에서는 Azure Workload Identity를 통해 유사한 패턴을 구현합니다. GKE에서는 Workload Identity가 내장되어 있습니다.

### 4.4 Pod Security Standards와 보안 컨텍스트

PSS(Pod Security Standards)는 PSP(Pod Security Policy)의 후계자로, Privileged, Baseline, Restricted의 3단계 프로파일을 제공합니다. Restricted는 가장 엄격한 설정으로, 루트 실행 금지, 특권 컨테이너 금지, 특정 볼륨 유형 제한, seccomp 프로파일 강제 등을 포함합니다.

```yaml
# 네임스페이스 레벨에서 PSS 적용
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# SecurityContext 심화 설정
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault  # OCI 기본 seccomp 프로파일
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]  # 모든 Linux 커널 능력 제거
        add: ["NET_BIND_SERVICE"]  # 1024 이하 포트 바인딩에만 필요한 경우
```

### 4.5 Kubernetes Network Policy와 마이크로세그멘테이션

기본적으로 Kubernetes 파드는 클러스터 내 모든 파드와 통신 가능합니다. Network Policy를 통해 마이크로세그멘테이션(Micro-segmentation)을 구현해야 합니다.

중요한 원칙은 먼저 모든 트래픽을 차단하는 Default Deny 정책을 적용하고, 그 후 필요한 통신만 명시적으로 허용하는 것입니다.

```yaml
# Default Deny All: 모든 인바운드/아웃바운드 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # 모든 파드에 적용
  policyTypes:
  - Ingress
  - Egress
---
# 특정 통신만 허용: frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
```

Cilium과 같은 eBPF 기반 CNI는 표준 Network Policy를 넘어 L7(HTTP 메서드, 경로) 레벨 정책을 지원합니다. 이는 서비스 메시 없이도 세밀한 트래픽 제어를 가능하게 합니다.

### 4.6 OPA(Open Policy Agent)와 Admission Control

OPA는 Kubernetes Admission Webhook과 통합되어 리소스 생성/수정 시 정책을 강제합니다. Rego 언어로 작성된 정책은 선언적으로 표현되며, Gatekeeper를 통해 Kubernetes 네이티브 CRD로 관리됩니다.

```rego
# Rego 정책: 모든 Deployment는 resource limits을 설정해야 함
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Deployment"
  container := input.request.object.spec.template.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container '%v'에 memory limit이 설정되지 않았습니다", [container.name])
}

deny[msg] {
  input.request.kind.kind == "Deployment"
  container := input.request.object.spec.template.spec.containers[_]
  not container.resources.limits.cpu
  msg := sprintf("Container '%v'에 CPU limit이 설정되지 않았습니다", [container.name])
}
```

Kyverno는 Rego 대신 YAML로 정책을 작성할 수 있어 진입 장벽이 낮고, 이미지 서명 검증(Sigstore/Cosign 통합), 리소스 자동 수정(Mutate) 기능을 내장하고 있어 최근 채택이 급증하고 있습니다.

### 4.7 Secrets 관리: External Secrets Operator와 Vault 통합

Kubernetes의 기본 Secret은 base64 인코딩에 불과하며 암호화가 아닙니다. etcd 암호화를 활성화하더라도, 시크릿 관리의 모범 사례는 외부 시크릿 관리 시스템(HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)과 통합하는 것입니다.

External Secrets Operator(ESO)는 외부 시크릿 스토어에서 시크릿을 Kubernetes Secret으로 자동 동기화합니다. 시크릿의 원본은 항상 외부 시스템에 있고, Kubernetes Secret은 캐시에 불과하게 됩니다.

```yaml
# ExternalSecret: AWS Secrets Manager에서 시크릿 동기화
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials  # 생성될 Kubernetes Secret 이름
    creationPolicy: Owner
  data:
  - secretKey: password       # K8s Secret의 키
    remoteRef:
      key: prod/database      # AWS Secrets Manager 경로
      property: password      # JSON 필드
```

### 4.8 Service Mesh와 mTLS: Istio와 Linkerd의 보안 아키텍처

서비스 메시는 애플리케이션 코드 수정 없이 서비스 간 통신에 mTLS(Mutual TLS)를 강제할 수 있습니다. Istio의 경우, 사이드카 프록시(Envoy)가 모든 인바운드/아웃바운드 트래픽을 가로채어 인증서 기반 인증을 수행합니다.

PeerAuthentication으로 mTLS 모드를 설정하고, AuthorizationPolicy로 서비스 간 접근을 제어합니다.

```yaml
# Istio PeerAuthentication: Strict mTLS 강제
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # mTLS 없는 트래픽 거부
---
# AuthorizationPolicy: 서비스 간 접근 제어
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

---

## 5. RBAC 설계 원칙 및 고급 패턴 {#5}

### 5.1 역할 폭발(Role Explosion) 문제와 ABAC 하이브리드 접근

나이브한 RBAC 구현은 조직이 성장할수록 역할의 수가 지수적으로 증가하는 "역할 폭발" 문제에 직면합니다. 이는 n명의 사용자와 m개의 리소스가 있을 때 O(n×m)의 역할 조합이 필요해지는 조합론적 문제입니다.

이에 대한 해법은 ABAC(Attribute-Based Access Control)와의 하이브리드 설계입니다. 역할은 큰 범주를 정의하고, 속성(태그)으로 세밀한 제어를 수행합니다.

```json
// AWS Tag-based Access Control (ABAC)
// 태그로 RBAC의 역할 폭발 문제 해결
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:ResourceTag/Environment": "${aws:PrincipalTag/Environment}",
      "ec2:ResourceTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
// 동일 Environment와 Team 태그가 있는 EC2만 제어 가능
// 사용자 수와 무관하게 단 하나의 정책으로 관리
```

### 5.2 최소 권한 원칙(PoLP)의 실용적 구현

최소 권한은 철학이지만 구현은 공학적 문제입니다. IAM Access Analyzer의 정책 생성 기능은 CloudTrail 기반으로 실제 사용된 API 호출을 분석하여 최소 권한 정책을 자동 생성합니다. 이는 "필요한 권한을 사전에 예측"하는 것이 아니라 "실제 사용된 권한만 유지"하는 데이터 주도 접근법입니다.

Amazon Access Advisor는 마지막 서비스 접근 시간을 보여주어 사용되지 않는 권한을 식별합니다. 90일 이상 미사용 권한은 제거하는 것이 권장됩니다.

### 5.3 권한 위임(Delegation)과 경계 설정

Permission Boundary는 관리자가 다른 팀에게 IAM 역할 생성 권한을 위임할 때 특히 중요합니다. 팀에게 자체 IAM 역할을 생성할 수 있는 권한을 주되, 자신들의 권한보다 높은 역할은 만들지 못하도록 경계를 설정합니다. 이를 "위임된 관리"(Delegated Administration) 패턴이라 합니다.

---

## 6. 멀티클라우드 통합 보안 아키텍처 {#6}

### 6.1 CSPM(Cloud Security Posture Management)

CSPM 도구(Wiz, Prisma Cloud, Lacework)는 멀티클라우드 환경에서 보안 구성 오류를 자동으로 탐지합니다. 단순한 취약점 스캔을 넘어, 클라우드 리소스 간 공격 경로(Attack Path)를 그래프 기반으로 시각화하여 "S3 버킷 → EC2 역할 → 관리자 권한"과 같은 복합 취약점 체인을 파악합니다.

### 6.2 SIEM 통합과 보안 이벤트 상관 분석

CloudTrail(AWS), Azure Monitor(Azure), Cloud Audit Logs(GCP)의 로그를 중앙 SIEM(Splunk, Microsoft Sentinel, Elastic SIEM)으로 집계하고, 상관 분석 규칙을 통해 위협을 탐지합니다. 특히 MITRE ATT&CK 프레임워크를 클라우드 환경에 적용한 MITRE ATT&CK Cloud Matrix는 탐지 규칙 설계의 표준 참조 자료입니다.

---

## 7. 대기업 면접 Q&A 30+ {#7}

> 다음 Q&A는 Amazon, Apple, Nvidia, Google, Microsoft, Tesla의 Senior Cloud Engineer / Security Engineer 포지션을 기준으로 구성되었습니다.

---

### [Amazon/AWS 중심 질문]

---

**Q1. AWS IAM 정책 평가에서 "Explicit Deny"와 "Implicit Deny"의 차이는 무엇이며, 어떤 시나리오에서 각각을 활용해야 하나요?**

**A.** Implicit Deny는 AWS의 기본 상태로, 어떠한 Allow 정책도 없을 때 자동으로 적용되는 거부입니다. 반면 Explicit Deny는 `"Effect": "Deny"`를 정책에 명시한 것으로, 이는 다른 모든 Allow 정책을 무력화합니다. 즉, 평가 계층 어디에서든 하나의 Explicit Deny가 존재하면 해당 요청은 무조건 차단됩니다.

실용적 활용 측면에서, Explicit Deny는 SCP에서 특정 리전 외 서비스 사용 차단, 루트 사용자 액션 차단, CloudTrail 비활성화 방지처럼 조직 전체에 예외 없이 적용해야 하는 가드레일에 사용합니다. Implicit Deny는 최소 권한 설계의 기반이 됩니다. 권한이 없으면 기본 차단되므로 명시적으로 필요한 것만 Allow하는 방식입니다. 중요한 점은 Permission Boundary가 설정된 경우, Permission Boundary에 해당 권한이 없으면 Identity 정책의 Allow는 Implicit Deny로 처리된다는 점입니다.

---

**Q2. S3 버킷에 교차 계정(Cross-Account) 접근을 구현할 때 필요한 정책의 조합과 그 이유를 설명하세요.**

**A.** 교차 계정 S3 접근에는 두 개의 정책이 모두 필요합니다. 첫째, 버킷 소유 계정(Account A)의 S3 버킷 정책에서 외부 계정(Account B)의 IAM 주체를 명시적으로 Allow해야 합니다. 둘째, Account B의 IAM 사용자/역할에 S3 접근 권한이 부여되어 있어야 합니다.

이유는 AWS의 교차 계정 신뢰 모델에 있습니다. Account B의 IAM 정책이 있더라도 Account A가 동의하지 않으면 접근이 불가하고, 반대로 버킷 정책에서 허용해도 Account B 내에서 해당 사용자에게 권한이 없으면 접근이 불가합니다. 양쪽 모두의 신뢰가 있어야 성립합니다. 예외로 리소스 기반 정책에서 전체 계정을 허용하는 경우(`Principal: {"AWS": "arn:aws:iam::ACCOUNT-B:root"}`), Account B의 내부 IAM 정책만으로도 접근이 가능합니다.

---

**Q3. EKS에서 IRSA와 Pod Identity의 차이점은 무엇이며, 각각 언제 사용하는 것이 적절한가요?**

**A.** IRSA(IAM Roles for Service Accounts)는 OIDC 토큰을 활용하며, ServiceAccount 어노테이션과 IAM 역할 신뢰 정책에서 `sub` 클레임(네임스페이스:서비스어카운트 형식)을 기반으로 바인딩됩니다. IRSA는 EKS 1.13부터 지원하는 성숙한 솔루션이지만, 클러스터당 하나의 OIDC 제공자 URL을 사용하므로 역할 신뢰 정책이 특정 클러스터에 종속됩니다.

EKS Pod Identity(2023년 출시)는 이 한계를 해결합니다. 클러스터와 IAM 역할 간의 연결을 EKS API 레벨에서 관리하므로, 역할 신뢰 정책 수정 없이 여러 클러스터에 동일 역할 재사용이 가능합니다. 또한 토큰 갱신이 더 투명하게 이루어지고 감사 추적이 개선됩니다. 신규 프로젝트에는 Pod Identity를 권장하지만, IRSA가 이미 구축된 환경에서는 마이그레이션 비용 대비 이득을 평가해야 합니다.

---

**Q4. AWS Organizations에서 SCP를 설계할 때 "Allowlist" 방식과 "Denylist" 방식의 트레이드오프를 설명하고, 각 환경에서의 권장 전략을 제시하세요.**

**A.** Allowlist 방식은 기본 `Deny *`를 설정하고 허용 서비스만 명시합니다. 이는 허가된 것만 사용 가능한 최강 보안 태세로, 규정 준수(Compliance) 요구가 강한 금융, 의료 등 규제 산업의 Production 계정에 적합합니다. 단점은 새로운 서비스 사용 시마다 SCP 업데이트가 필요하여 개발 민첩성이 저하됩니다.

Denylist 방식은 기본 `Allow *` 상태에서 특정 위험 행위만 차단합니다. 개발 자유도를 유지하면서 핵심 가드레일(CloudTrail 삭제 방지, 루트 계정 활동 차단, 승인되지 않은 리전 사용 방지 등)을 보장합니다. 개발 계정이나 샌드박스 환경에 적합합니다. 실무에서는 두 방식을 계층화합니다. Management Group → Organizational Unit 계층에 따라 Production OU에는 Allowlist, Sandbox OU에는 Denylist를 적용하는 방식이 균형잡힌 전략입니다.

---

**Q5. Confused Deputy 공격이 무엇인지 설명하고, AWS 환경에서 이를 방지하는 방법을 코드 레벨로 설명하세요.**

**A.** Confused Deputy 공격은 권한이 높은 주체(AWS 서비스)가 낮은 권한 주체의 요청에 의해 의도치 않게 다른 계정의 리소스에 접근하도록 유도되는 취약점입니다. 예를 들어 악의적인 계정이 Lambda 역할 신뢰 정책에서 `Principal: {"Service": "lambda.amazonaws.com"}`만 명시한 역할을 만들면, AWS Lambda 서비스를 신뢰하는 모든 역할이 잠재적 공격 대상이 됩니다.

방지책은 `aws:SourceArn`과 `aws:SourceAccount` 조건 키를 역할 신뢰 정책에 명시하는 것입니다. `aws:SourceArn`으로 특정 리소스(예: 특정 Lambda 함수 ARN)를 지정하면, 오직 그 리소스만 역할을 수임할 수 있습니다. 이 두 조건을 함께 사용하는 이유는, ARN은 글로벌 유일이지만 계정 검증을 추가함으로써 ARN 예측 공격의 가능성까지 차단하기 위함입니다. 앞서 제시한 JSON 코드 예시가 표준 패턴입니다.

---

**Q6. AWS KMS의 CMK(Customer Managed Key)와 AWS Managed Key의 차이를 설명하고, 각각의 사용 케이스를 제시하세요.**

**A.** AWS Managed Key는 AWS가 자동으로 생성하고 관리하며, 고객이 키 정책을 수정할 수 없습니다. 비용이 없고 자동 교체(연 1회)가 기본 활성화되어 있습니다. 단일 계정, 단일 서비스 내의 단순 암호화에 적합합니다.

CMK는 고객이 생성하고 키 정책을 완전히 제어합니다. 교차 계정 접근, 교차 서비스 접근, 암호화 컨텍스트 활용, 키 삭제 스케줄 설정, HSM 기반의 CloudHSM 연계 등 세밀한 제어가 필요할 때 사용합니다. 비용은 월 $1/키 + API 호출 비용이 발생합니다. 규제 준수 환경에서는 CMK를 통해 "키 소유권"을 명확히 하고, 키 접근 감사 로그(CloudTrail)로 누가 언제 어떤 데이터를 복호화했는지 추적하는 것이 중요합니다.

---

**Q7. AWS VPC에서 Security Group과 Network ACL의 차이를 설명하고, DDoS 방어 아키텍처에서 각각의 역할을 논하세요.**

**A.** Security Group은 ENI(탄력적 네트워크 인터페이스) 레벨에서 작동하는 Stateful 방화벽입니다. 반환 트래픽을 자동으로 허용하여 아웃바운드 규칙을 단순화할 수 있습니다. 허용 규칙만 가능하고 거부 규칙은 존재하지 않습니다.

Network ACL은 서브넷 레벨에서 작동하는 Stateless 방화벽으로, 인바운드와 아웃바운드를 독립적으로 평가합니다. 규칙 번호 순서대로 평가하고 명시적 거부(Deny)가 가능합니다. DDoS 방어 측면에서 NACL은 대규모 공격에서 첫 번째 방어선으로 특정 IP 대역의 트래픽을 서브넷 레벨에서 차단합니다. NACL은 상태를 추적하지 않으므로 대용량 연결 상태 테이블 고갈 공격에 더 내성이 있습니다. Security Group은 개별 인스턴스 레벨에서 세밀한 제어를 담당하고, AWS Shield Advanced + WAF는 L3/L4/L7 DDoS를 종합적으로 방어합니다.

---

### [Google Cloud / Kubernetes 중심 질문]

---

**Q8. Kubernetes RBAC에서 최소 권한 원칙을 구현할 때 발생하는 실제 도전 과제와 해결 방법은?**

**A.** 가장 큰 도전은 필요한 최소 권한을 사전에 정확히 파악하기 어렵다는 점입니다. 개발자들은 편의를 위해 과도한 권한을 요청하는 경향이 있습니다.

해결책으로는 첫째, `kubectl auth can-i --list --as=system:serviceaccount:ns:sa`로 특정 ServiceAccount의 현재 권한을 열거하여 정기적으로 감사합니다. 둘째, rbac-tool이나 Fairwinds Polaris 같은 도구로 불필요한 ClusterRole 바인딩을 탐지합니다. 셋째, 처음에는 넓은 권한을 부여하고 실제 사용 로그를 분석한 후 최소화하는 점진적 축소 전략을 사용합니다. 넷째, Namespace에 ResourceQuota와 LimitRange를 함께 설정하여 RBAC 단독이 아니라 다계층 방어를 구성합니다. 가장 흔한 안티패턴인 `cluster-admin` ClusterRole을 ServiceAccount에 바인딩하는 것은 절대 허용하지 않는 OPA/Kyverno 정책을 기본 설치해야 합니다.

---

**Q9. Kubernetes Secret은 기본적으로 안전하지 않습니다. etcd 암호화를 활성화한 이후에도 완전한 시크릿 보안을 위해 추가로 필요한 조치들을 설명하세요.**

**A.** etcd 암호화는 저장된 시크릿의 암호화를 보장하지만, 여러 다른 공격 벡터가 존재합니다.

메모리 상의 시크릿은 암호화되지 않습니다. 파드에 마운트된 시크릿은 tmpfs에 평문으로 존재하며, `kubectl describe pod`나 node의 `/proc` 파일시스템으로 접근 가능합니다. 따라서 RBAC로 `secrets` 리소스에 대한 `get`, `list`, `watch` 동사를 엄격히 제한해야 합니다. `list`만 있어도 네임스페이스의 모든 시크릿 이름을 열거할 수 있어 공격자에게 정보를 제공합니다.

완전한 보안을 위해서는 External Secrets Operator + 외부 비밀 관리 시스템(Vault, AWS Secrets Manager)으로 Kubernetes Secret을 임시 캐시로만 사용하고, CSI(Container Storage Interface) Secret Store 드라이버로 시크릿을 환경변수가 아닌 볼륨으로 마운트하여 메모리 노출을 최소화하며, Secret 접근에 대한 감사 로그를 활성화하고 이상 패턴 알람을 설정하는 것이 필요합니다.

---

**Q10. Kubernetes에서 멀티테넌시(Multi-tenancy)를 구현하는 세 가지 모델을 비교하고, 각 모델의 보안 수준과 트레이드오프를 분석하세요.**

**A.** 첫째, 클러스터 수준 격리(Cluster per Tenant)는 테넌트마다 별도의 Kubernetes 클러스터를 운영합니다. 가장 강한 격리를 제공하지만 운영 비용과 복잡성이 가장 높습니다. 커널 취약점이 타 테넌트에 영향을 미치지 않습니다. 주로 금융, 의료 등 규제 요구가 강한 환경이나 대형 엔터프라이즈 고객 간 격리에 사용됩니다.

둘째, 네임스페이스 수준 격리(Namespace per Tenant)는 동일 클러스터를 네임스페이스로 분리합니다. Network Policy로 네트워크 격리, RBAC로 API 서버 접근 격리, ResourceQuota로 리소스 격리를 구현합니다. 비용 효율적이지만 커널을 공유하므로 컨테이너 탈출(Container Escape) 취약점이 전체 클러스터 위협이 됩니다. CNCF의 Hierarchical Namespaces로 더 세밀한 네임스페이스 계층 구조 관리가 가능합니다.

셋째, vCluster(가상 클러스터)는 실제 클러스터 내에 격리된 Kubernetes 컨트롤 플레인을 실행합니다. 테넌트에게 마치 자신만의 클러스터처럼 보이면서 실제 노드는 공유합니다. 관리 오버헤드와 격리 수준이 위 두 방식의 중간입니다. SaaS 플랫폼이나 개발 환경 제공에 적합합니다.

---

**Q11. OPA Gatekeeper와 Kyverno를 비교하고, 어떤 기준으로 선택해야 하는지 설명하세요.**

**A.** OPA Gatekeeper는 Rego 언어를 사용하여 강력하고 유연한 정책을 표현할 수 있습니다. 복잡한 논리(조건 분기, 데이터 조회, 집합 연산)를 자유롭게 표현하고, OPA 에코시스템(Conftest 등)과 통합하여 CI/CD에서 로컬 정책 테스트도 가능합니다. 단점은 Rego의 학습 곡선이 가파르다는 것입니다.

Kyverno는 YAML 네이티브로 Kubernetes 운영자에게 더 친숙합니다. 검증(Validate), 변형(Mutate), 생성(Generate) 세 종류의 정책을 지원하고, Sigstore/Cosign을 통한 이미지 서명 검증이 내장되어 있어 Supply Chain Security 구현이 용이합니다. 단점은 복잡한 비즈니스 로직 표현의 한계가 있습니다.

선택 기준으로, 이미 OPA 에코시스템이 있거나 복잡한 커스텀 정책이 필요하면 Gatekeeper, 빠른 도입과 이미지 서명 검증이 필요하면 Kyverno를 선택합니다. 둘을 동시에 사용하는 조직도 많으며, 각각의 강점을 용도에 맞게 활용합니다.

---

### [Microsoft Azure 중심 질문]

---

**Q12. Azure Entra ID의 Conditional Access와 AWS의 IAM Condition의 근본적인 차이는 무엇인가요?**

**A.** AWS IAM Condition은 API 요청 시점의 정적 속성(IP, 시간대, MFA 여부, 리소스 태그)을 평가하는 선언적 조건입니다. 이는 개별 API 호출 레벨에서 작동합니다.

Azure Conditional Access는 훨씬 더 동적이고 컨텍스트 풍부한 접근 제어입니다. 사용자의 실시간 리스크 점수(Identity Protection의 ML 기반 이상 탐지), 디바이스 준수 상태(Intune 통합), 세션 내 지속적 평가(CAE)가 가능합니다. 즉, 로그인 시점뿐만 아니라 세션 중에도 지속적으로 접근을 재평가합니다. AWS에서는 최근 IAM Roles Anywhere와 STS의 세션 조건으로 유사한 기능을 구현할 수 있지만, Azure의 실시간 위험 기반 적응적 인증에는 아직 미치지 못합니다.

---

**Q13. Azure Managed Identity와 Service Principal의 차이는 무엇이며, 보안 관점에서 어떤 것을 선호해야 하나요?**

**A.** Service Principal은 애플리케이션을 위한 Azure AD 아이덴티티로, 인증에 클라이언트 시크릿(비밀번호) 또는 인증서를 사용합니다. 시크릿은 만료 관리, 로테이션, 누출 위험이라는 운영 부담을 수반합니다.

Managed Identity는 Azure 플랫폼이 자격증명을 자동으로 관리하므로 애플리케이션이 자격증명을 보유하지 않습니다. IMDS(인스턴스 메타데이터 서비스)를 통해 토큰을 자동 갱신하므로 자격증명 만료나 누출 위험이 없습니다.

보안 관점에서 Managed Identity를 항상 선호해야 합니다. 시크릿이 없으면 누출될 시크릿도 없기 때문입니다. Managed Identity를 지원하지 않는 서비스(Azure 외부 서비스, 온프레미스)에 한해서만 Service Principal을 사용하되, 인증서를 시크릿보다 우선 선택하고 Azure Key Vault로 자격증명을 관리해야 합니다.

---

**Q14. Azure PIM(Privileged Identity Management)을 운영할 때 Just-in-Time 접근 요청의 승인 워크플로우를 설계하는 방법과 모범 사례를 설명하세요.**

**A.** PIM 승인 워크플로우 설계에서 핵심 원칙은 승인자와 요청자의 분리입니다. Production 환경의 Owner 역할 활성화는 반드시 다른 팀의 관리자가 승인해야 하며, 자가 승인(Self-approval)은 구조적으로 차단해야 합니다.

활성화 요구사항으로 Contributor 이상 역할에는 MFA를 필수화하고, Global Administrator 같은 최고 권한 역할에는 다중 승인자(Multi-approver) 설정을 권장합니다. 활성화 사유(Justification)를 필수 입력으로 설정하면 감사 추적 시 맥락 파악에 도움이 됩니다.

활성화 시간은 업무 필요성에 따라 최소화합니다. 예를 들어 긴급 장애 대응은 2시간, 정기 배포는 4시간, 감사 활동은 8시간으로 차등화합니다. 활성화 알림은 Slack/Teams/Email로 실시간 전파하여 비인가 접근을 즉시 감지할 수 있어야 합니다. PIM Audit Logs는 Sentinel과 연동하여 비정상 활성화 패턴(업무 시간 외 접근, 빈번한 단기 활성화)을 탐지해야 합니다.

---

**Q15. Azure Policy와 Azure RBAC의 차이를 설명하고, 둘을 함께 사용하는 거버넌스 아키텍처를 설계하세요.**

**A.** Azure RBAC는 "누가 무엇을 할 수 있는가"를 제어합니다. 특정 사용자나 서비스 주체에게 리소스 생성, 수정, 삭제 권한을 부여합니다. Azure Policy는 "어떻게 리소스를 만들어야 하는가"를 제어합니다. 권한이 있더라도 정책 조건을 위반하면 작업이 거부되거나, 기존 리소스가 비준수(Non-compliant) 상태로 표시됩니다.

통합 거버넌스 아키텍처에서 RBAC로 역할별 작업 범위를 정의하고, Azure Policy로 태그 강제(`Environment`, `CostCenter` 필수 태그), 리소스 위치 제한(특정 리전만 허용), SKU 제한(비용 통제), 암호화 강제(디스크 암호화 필수) 등을 구현합니다. 정책 이니셔티브(Initiative)로 관련 정책들을 묶어 CIS Benchmark, SOC 2, ISO 27001 등 컴플라이언스 표준 전체를 한 번에 적용할 수 있습니다. Management Group에 정책을 적용하면 하위 모든 구독에 자동 상속됩니다.

---

### [고급 시스템 설계 / 아키텍처 질문]

---

**Q16. 수천 개의 마이크로서비스가 운영되는 환경에서 서비스 간 인증을 어떻게 설계하겠습니까? mTLS와 JWT Bearer Token 방식의 트레이드오프를 분석하세요.**

**A.** mTLS는 양방향 인증서 기반 인증으로 전송 계층에서 신원 검증이 이루어집니다. 장점은 별도 인증 로직 없이 네트워크 레벨 신원 보증이 가능하고, 암호화가 내장되어 있다는 점입니다. 단점은 인증서 관리(발급, 갱신, 폐기) 인프라(PKI)의 운영 복잡성입니다. Istio나 Linkerd 같은 서비스 메시를 사용하면 이 복잡성을 인프라로 추상화할 수 있습니다.

JWT Bearer Token은 서비스가 IdP(예: Vault, SPIRE)에서 서비스 ID 토큰을 발급받아 요청 헤더에 포함합니다. 장점은 클레임 기반으로 세밀한 권한 정보를 토큰에 내장할 수 있고, HTTP 레이어에서 작동하여 네트워크 인프라에 독립적이라는 점입니다. 단점은 토큰 만료 전 탈취 시 차단이 어렵고, 모든 서비스가 토큰 검증 로직을 구현해야 합니다.

대규모 환경에서 권장하는 조합은 mTLS로 전송 계층 기본 인증을 보장하고(인프라 레이어), JWT로 서비스 수준의 세밀한 권한 제어를 수행하는(애플리케이션 레이어) 이중 방어 모델입니다. SPIFFE/SPIRE 프레임워크는 플랫폼 독립적 워크로드 아이덴티티를 제공하여 mTLS와 JWT를 통합합니다.

---

**Q17. Zero Trust Network Access(ZTNA) 아키텍처를 멀티클라우드 환경에서 구현하기 위한 전략을 설명하세요.**

**A.** ZTNA 구현의 핵심은 네트워크 위치 기반 신뢰를 사용자/디바이스/워크로드 아이덴티티 기반 신뢰로 대체하는 것입니다.

아이덴티티 레이어에서는 중앙 IdP(Okta, Azure AD)를 단일 진실 원천으로 설정하고, 모든 클라우드의 IAM 시스템을 페더레이션합니다. AWS IAM Identity Center, Azure AD, GCP Workforce Identity Federation이 각 클라우드의 연결 지점입니다.

디바이스 신뢰 레이어에서는 MDM(Intune, Jamf)으로 디바이스 컴플라이언스를 관리하고, 디바이스 인증서를 발급하여 접근 시 검증합니다.

네트워크 레이어에서는 사용자 트래픽은 Cloudflare Access, Zscaler Private Access, BeyondCorp Enterprise 같은 ZTNA 프록시를 통과시킵니다. 직접 VPN 접근 대신 애플리케이션 레벨 프록시를 사용하여 내부 네트워크 전체 노출을 방지합니다. 서비스 간 통신은 서비스 메시의 mTLS로 보호합니다.

---

**Q18. 대규모 Kubernetes 클러스터에서 이미지 공급망 보안(Supply Chain Security)을 구현하는 end-to-end 파이프라인을 설계하세요.**

**A.** 공급망 보안은 이미지가 생성되고 배포되는 전체 생명 주기에 걸친 무결성 보증 체계입니다.

소스 레벨에서 브랜치 보호 규칙, 코드 리뷰 필수화, Dependabot/Renovate로 취약한 의존성 자동 탐지 및 업데이트를 구성합니다. 빌드 레벨에서 SLSA(Supply-chain Levels for Software Artifacts) 프레임워크를 적용합니다. 빌드 환경을 격리하고 빌드 출처(Provenance)를 생성하여 "이 이미지가 어떤 코드에서, 어떤 환경에서 빌드되었는가"를 증명합니다.

이미지 레벨에서 Trivy 또는 Grype로 취약점 스캔을 수행하고, 임계 취약점이 있으면 파이프라인을 차단합니다. Cosign으로 이미지에 디지털 서명을 추가하고, Rekor(투명성 로그)에 서명 기록을 게시합니다.

배포 레벨에서 Kyverno의 이미지 검증 정책으로 서명되지 않거나 특정 레지스트리 외부의 이미지 배포를 차단합니다. AWS ECR, Azure ACR, Google Artifact Registry의 취약점 스캔 기능을 활성화하고, "승인된 이미지 레지스트리 목록" 이외의 소스를 거부하는 Admission Webhook을 구성합니다.

---

**Q19. Kubernetes에서 etcd 백업 및 암호화가 중요한 이유와, etcd가 침해되었을 때의 전체적인 보안 영향을 분석하세요.**

**A.** etcd는 Kubernetes의 두뇌로 모든 클러스터 상태(시크릿, 서비스어카운트 토큰, 네트워크 정책, RBAC 규칙, 파드 스펙 등)를 저장합니다. etcd 침해는 단순한 데이터 유출이 아니라 클러스터 완전 탈취입니다.

침해 시나리오를 분석하면, 공격자가 etcd에 직접 접근하면 Secret 오브젝트를 base64 디코딩만으로 평문 시크릿을 획득합니다. ServiceAccount 토큰을 탈취하여 `cluster-admin` 권한을 가진 ServiceAccount의 토큰으로 API 서버에 접근하면 클러스터 완전 장악입니다. 또한 악의적인 파드 스펙을 직접 주입하여 특권 컨테이너를 실행하면 노드 탈출이 가능합니다.

방어 전략으로는 먼저 etcd는 반드시 별도의 격리된 노드에서 실행하고, 클라이언트 인증서로만 접속을 허용하며, etcd 포트(2379, 2380)는 컨트롤 플레인 노드 외부에서 절대 접근 불가하게 네트워크 정책을 구성합니다. etcd 데이터 암호화(`--encryption-provider-config`)를 활성화하여 시크릿을 저장 시 암호화합니다. 백업은 암호화된 형태로 외부 저장소에 보관하고, Velero 같은 도구로 정기적 백업과 복원 테스트를 수행해야 합니다.

---

**Q20. Production 환경에서 Kubernetes 노드가 침해되었을 때의 인시던트 대응(IR) 플레이북을 단계적으로 설명하세요.**

**A.** **탐지 단계**: Falco, Sysdig Secure, 또는 클라우드 제공자의 위협 탐지 서비스(AWS GuardDuty EKS Protection, Microsoft Defender for Containers)의 알림을 통해 이상 행동을 감지합니다. 지표로는 예상치 않은 시스템 호출(ptrace, nsenter), 컨테이너에서의 쉘 실행, 비정상 네트워크 연결 등이 있습니다.

**격리 단계**: 침해된 노드를 즉시 `kubectl cordon`으로 스케줄링 차단하고, 새로운 파드 배치를 막습니다. Network Policy로 해당 노드의 모든 이그레스 트래픽을 차단합니다. 클라우드 레벨에서 Security Group/NSG로 노드를 격리합니다. 노드를 즉시 종료하는 것은 포렌식 증거 확보 전에는 자제합니다.

**증거 확보 단계**: 메모리 덤프, 컨테이너 파일시스템 스냅샷, 감사 로그(API Server Audit Log, CloudTrail), 컨테이너 런타임 로그를 수집합니다. 침해된 파드에서 사용된 ServiceAccount 토큰을 즉시 무효화합니다(토큰 재생성).

**복구 단계**: 노드를 클러스터에서 제거하고 새 노드로 교체합니다. 침해 경로로 사용된 취약점을 패치한 새 AMI/이미지로 노드를 재구성합니다. 영향받은 네임스페이스의 모든 시크릿을 로테이션합니다.

**사후 분석**: 루트 원인 분석(RCA)을 통해 초기 침해 경로를 파악하고 보안 정책에 반영합니다.

---

### [심화 개념 / 보안 설계 질문]

---

**Q21. SPIFFE/SPIRE 프레임워크가 무엇인지 설명하고, 대규모 마이크로서비스 환경에서의 활용 방안을 제시하세요.**

**A.** SPIFFE(Secure Production Identity Framework For Everyone)는 플랫폼 독립적 워크로드 아이덴티티 표준입니다. SVID(SPIFFE Verifiable Identity Document)라는 단기 수명의 X.509 인증서 또는 JWT 토큰으로 워크로드 아이덴티티를 표현합니다. SPIRE(SPIFFE Runtime Environment)는 이 표준의 레퍼런스 구현체입니다.

멀티클라우드, 멀티플랫폼 환경에서 SPIFFE의 가치는 이기종 환경(EC2, GCE, 베어메탈, VM, Kubernetes 파드)의 워크로드에 동일한 아이덴티티 프레임워크를 적용할 수 있다는 점입니다. AWS IAM 역할, Kubernetes ServiceAccount, VM 인스턴스 모두 SPIFFE ID로 추상화됩니다.

실제 구현에서 SPIRE Agent가 각 노드에서 실행되며 워크로드를 검증하고 SVID를 발급합니다. Istio는 내부적으로 SPIFFE를 사용합니다. 클라우드 공급자 간 서비스 신뢰를 SPIFFE 연합(Federation)으로 구현하면 교차 클라우드 mTLS가 단일 아이덴티티 프레임워크로 통합됩니다.

---

**Q22. 시크릿 로테이션(Secret Rotation)의 자동화가 어려운 이유와, 무중단 로테이션을 구현하기 위한 패턴을 설명하세요.**

**A.** 시크릿 로테이션의 기술적 어려움은 "이전 시크릿을 사용하는 기존 세션"과 "새 시크릿을 사용하는 신규 세션"이 동시에 존재하는 전환 기간 처리에 있습니다. 서비스를 재시작하는 순간 인증 실패가 발생할 수 있습니다.

무중단 로테이션 패턴으로, 첫째 이중 유효 시크릿(Dual Active Secret) 패턴을 사용합니다. 새 시크릿을 발급하고 데이터베이스/서비스에 등록한 후, 일정 시간 동안 이전 시크릿과 새 시크릿 모두 유효하도록 유지합니다. 모든 클라이언트가 새 시크릿으로 전환된 것을 확인한 후 이전 시크릿을 폐기합니다.

둘째 Sidecar 패턴에서 Vault Agent나 ESO(External Secrets Operator)가 사이드카로 동작하여 시크릿을 주기적으로 갱신하고 파일시스템으로 마운트합니다. 애플리케이션은 파일을 실시간 감시하여 변경 시 새 시크릿을 자동으로 로드합니다. 재시작 없이 시크릿이 갱신됩니다.

AWS Secrets Manager의 자동 로테이션 Lambda는 이 패턴의 구현체로, RDS 비밀번호 로테이션을 무중단으로 처리합니다.

---

**Q23. Kubernetes에서 eBPF(Extended Berkeley Packet Filter)를 활용한 보안 모니터링과 정책 강제의 원리를 설명하세요.**

**A.** eBPF는 사용자 공간 프로그램을 수정하거나 커널 모듈을 로드하지 않고 커널 이벤트에 프로그램을 삽입할 수 있는 혁신적 기술입니다. 보안 관점에서 eBPF는 시스템 호출 레벨에서 모든 프로세스 활동을 추적할 수 있습니다.

Falco는 전통적으로 커널 모듈을 사용했으나 eBPF 드라이버로 전환 중이며, Cilium은 eBPF를 CNI와 보안 정책 모두에 활용합니다. Tetragon(Isovalent 개발)은 eBPF로 프로세스 실행, 네트워크 연결, 파일 시스템 접근을 실시간으로 모니터링하고 정책 위반 시 즉시 프로세스를 종료할 수 있습니다. 이는 기존 탐지 후 대응(Detect and Respond)에서 탐지와 동시 방지(Detect and Prevent)로의 패러다임 전환입니다.

eBPF의 보안 관점에서 주의사항은, eBPF 프로그램 자체가 커널에서 실행되므로 악의적인 eBPF 프로그램은 강력한 루트킷이 될 수 있습니다. 따라서 Kubernetes 환경에서 권한 없는 사용자가 eBPF를 로드하지 못하도록 `CAP_BPF` 능력을 엄격히 제한해야 합니다.

---

**Q24. IAM 권한의 Blast Radius(피해 반경)를 최소화하는 설계 원칙과 실제 구현 전략을 설명하세요.**

**A.** Blast Radius는 특정 아이덴티티가 침해되었을 때 공격자가 접근할 수 있는 리소스의 범위입니다. 이를 최소화하는 핵심 전략들이 있습니다.

계정 경계 활용에서 동일 계정 내 모든 리소스는 잠재적으로 같은 Blast Radius 내에 있습니다. Production/Staging/Development를 별도 계정으로 분리하면 계정 간 Blast Radius 전파를 차단합니다.

시간적 제한(Temporal Isolation)에서 임시 자격증명을 최소한의 시간만 유효하게 설정합니다. 15분짜리 STS 토큰은 침해되어도 공격자의 기회 창이 매우 좁습니다.

범위 제한(Scope Isolation)에서 Session Policy로 AssumeRole 시 동적으로 권한을 추가 제한합니다. CI/CD 파이프라인에서 특정 환경 배포 역할을 수임할 때, 해당 배포에 필요한 최소 권한만 Session Policy로 명시합니다.

리소스 기반 제한에서 S3 버킷 정책에 `aws:ResourceAccount` 조건을 추가하여 특정 계정에서만 접근 가능하게 하면, 자격증명이 유출되어도 다른 계정에서는 사용 불가능합니다.

---

**Q25. HashiCorp Vault의 동적 시크릿(Dynamic Secret)의 원리와 Kubernetes 통합 방법을 설명하세요.**

**A.** Vault의 동적 시크릿은 요청 시마다 새로운 자격증명을 실시간으로 생성하고, 정해진 TTL(Time-to-Live) 후 자동 폐기합니다. 데이터베이스 시크릿을 예로 들면, 파드가 Vault에 데이터베이스 자격증명을 요청하면 Vault가 데이터베이스에 직접 연결하여 고유한 임시 사용자를 생성하고, TTL 만료 시 해당 사용자를 자동 삭제합니다.

이 접근법의 보안적 가치는 각 파드가 고유한 자격증명을 사용하므로 한 자격증명 유출이 다른 파드에 영향을 미치지 않습니다. 또한 자격증명 유출을 알아채지 못해도 TTL이 지나면 자동 무효화됩니다.

Kubernetes 통합에서 Vault Agent Injector는 파드 생성 시 Vault Agent 사이드카를 자동 주입하여 Vault 인증(Kubernetes Auth Method)과 시크릿 획득을 처리합니다. Vault의 Kubernetes Auth Method는 파드의 ServiceAccount 토큰을 Kubernetes API로 검증하여 Vault 토큰을 발급합니다. 이를 통해 파드가 자신의 ServiceAccount 아이덴티티만으로 Vault에서 필요한 시크릿을 획득합니다.

---

### [DevSecOps / 운영 관련 질문]

---

**Q26. GitOps 환경에서 시크릿 관리의 모범 사례는 무엇이며, Sealed Secrets와 External Secrets Operator를 비교하세요.**

**A.** GitOps의 근본 원칙은 Git이 유일한 진실 원천이어야 한다는 것입니다. 그러나 평문 시크릿을 Git에 저장하는 것은 절대 불가합니다.

Sealed Secrets는 공개키 암호화로 시크릿을 암호화된 `SealedSecret` 오브젝트로 변환하여 Git에 저장합니다. 클러스터의 Sealed Secrets 컨트롤러만이 개인키로 복호화할 수 있습니다. 장점은 GitOps 원칙을 완전히 준수하고 추가 외부 의존성이 없습니다. 단점은 시크릿 로테이션 시 암호화 파일을 재생성해야 하고, 컨트롤러 개인키 유출 시 모든 저장된 시크릿이 위험에 처합니다.

External Secrets Operator는 시크릿 원본을 외부 시스템(AWS Secrets Manager, Vault 등)에 유지하고, Git에는 어디서 시크릿을 가져올지를 정의한 `ExternalSecret` 매니페스트만 저장합니다. 장점은 중앙화된 시크릿 관리, 감사 추적, 로테이션 자동화가 용이합니다. 단점은 외부 시스템에 대한 의존성이 생깁니다.

엔터프라이즈 환경에서는 External Secrets Operator + AWS Secrets Manager/Vault 조합이 더 성숙한 선택입니다. Sealed Secrets는 소규모 또는 외부 시스템 구축이 어려운 환경에서 실용적입니다.

---

**Q27. 클라우드 환경에서 Security Logging의 완전성(Completeness)을 보장하기 위한 아키텍처를 설계하고, 탐지 불가능한 로그 삭제를 방지하는 방법을 설명하세요.**

**A.** 로그 완전성의 가장 큰 위협은 공격자가 흔적을 지우기 위해 로그를 삭제하는 것입니다. 이를 방지하기 위한 불변성(Immutability) 보장 전략이 필요합니다.

AWS 환경에서 CloudTrail 로그를 S3에 저장할 때 S3 Object Lock(WORM: Write Once Read Many) 모드를 활성화하면 설정된 보존 기간 동안 누구도 삭제할 수 없습니다. 루트 계정조차도 불가능합니다. S3 버킷은 별도의 보안 계정(Log Archive Account)에 위치시키고, 운영 계정에서는 쓰기 권한만 부여합니다.

CloudTrail의 로그 파일 무결성 검증(Log File Integrity Validation)을 활성화하면 각 로그 파일의 SHA-256 해시와 디지털 서명이 포함된 다이제스트 파일이 생성됩니다. 공격자가 로그를 수정하면 다이제스트 파일과 불일치가 발생하여 변조를 탐지할 수 있습니다.

이벤트 브리지(EventBridge)로 CloudTrail 삭제, 로깅 비활성화 시도를 실시간으로 탐지하고 Security Hub/SNS로 즉각 알림합니다. AWS Config 규칙으로 CloudTrail 비활성화를 자동 탐지하고 자동 재활성화(Auto-remediation)할 수 있습니다.

---

**Q28. FinOps와 SecOps의 교차점에서, 비용 최적화를 위한 리소스 정리가 보안 취약점으로 이어지는 사례와 예방 방법을 설명하세요.**

**A.** 비용 절감을 위해 "사용되지 않는" 보안 컴포넌트를 제거할 때 취약점이 발생합니다.

실제 사례들을 살펴보면, VPC Flow Log를 비용 절감 목적으로 비활성화하면 비용은 절약되지만 네트워크 이상 트래픽 탐지 능력이 사라집니다. GuardDuty를 비활성화하면 위협 탐지 비용은 절약하지만 침해 탐지 시간이 급증합니다. "미사용 IAM 사용자" 정리 시 실제로 자동화 스크립트가 사용 중인 서비스 계정을 삭제하면 서비스 중단이 발생합니다.

예방 전략으로, Security 관련 리소스에는 `SecurityBaseline: required` 같은 필수 태그를 붙이고, FinOps 도구에서 이 태그가 있는 리소스는 비용 최적화 권고 대상에서 제외하도록 구성합니다. IAM 사용자 삭제 전에는 최소 90일의 비활성 기간과 서비스 임팩트 분석이 필요합니다. GuardDuty, CloudTrail, Config의 비활성화 시도는 SCP로 차단하거나 즉시 알림을 발생시킵니다. 보안 예산을 "비용 절감 대상 외" 고정 예산으로 분리하는 조직적 거버넌스가 기술적 통제만큼 중요합니다.

---

**Q29. Kubernetes에서 CIS(Center for Internet Security) Benchmark를 자동으로 적용하고 컴플라이언스를 지속적으로 모니터링하는 시스템을 설계하세요.**

**A.** CIS Kubernetes Benchmark는 API 서버 설정, etcd 보안, 컨트롤러 매니저, 스케줄러, 노드 설정, 정책 등 6개 영역에 걸쳐 수십 개의 보안 권고사항을 포함합니다.

자동화 파이프라인에서 클러스터 프로비저닝 단계에서는 `kube-bench`를 CI/CD에 통합하여 노드가 생성될 때마다 CIS 벤치마크 검사를 실행하고, 통과하지 못한 노드는 자동으로 격리합니다. Terraform/Pulumi로 클러스터를 IaC로 관리하면 모든 설정이 코드로 리뷰 가능하고 CIS 기준 충족 여부를 PR 단계에서 검증할 수 있습니다.

지속적 모니터링에서 Starboard(현 Trivy Operator)는 클러스터 내에서 지속적으로 CIS 벤치마크를 실행하고 결과를 CRD로 저장합니다. Prometheus로 컴플라이언스 점수를 메트릭으로 수집하고, Grafana 대시보드로 시각화하며, 점수가 임계값 아래로 떨어지면 PagerDuty 알림을 발생시킵니다. 관리형 EKS/AKS/GKE 환경에서는 클라우드 제공자가 컨트롤 플레인 CIS 항목을 기본으로 처리하므로 노드와 정책 레벨 항목에 집중합니다.

---

**Q30. 내부자 위협(Insider Threat)을 감지하고 대응하기 위한 클라우드 보안 아키텍처를 설계하세요.**

**A.** 내부자 위협은 기술적 통제만으로 완전히 막을 수 없습니다. 탐지, 예방, 억제를 계층적으로 구현해야 합니다.

탐지 시스템에서 CloudTrail 이벤트를 ML 기반 이상 탐지 서비스(AWS GuardDuty, Azure Sentinel의 UEBA)로 분석합니다. 탐지 대상 패턴으로는 업무 외 시간 대량 데이터 다운로드, 평소 접근하지 않던 리소스에 대한 갑작스러운 접근, 짧은 시간 내 다수 리전에서의 활동, 권한 에스컬레이션 시도 등이 있습니다.

기술적 예방 통제로 데이터 유출 방지(DLP)를 구현합니다. S3에서 민감 데이터(PII, 카드 정보)를 포함한 파일의 외부 공유를 차단하고, AWS Macie로 S3 버킷의 민감 데이터를 자동 분류합니다. VPC Endpoint로 S3 접근을 내부로 제한하고, S3 버킷 정책으로 특정 VPC 외부에서의 직접 접근을 차단합니다.

최소 권한 + 직무 분리(SoD, Separation of Duties)로 공격 표면을 최소화합니다. 프로덕션 데이터 접근과 인프라 관리 권한을 같은 사람이 갖지 못하도록 구조적으로 분리합니다.

---

**Q31. 클라우드 환경에서 암호화 키 계층(Key Hierarchy)를 설계하는 방법과 각 계층의 역할을 설명하세요.**

**A.** 암호화 키 계층은 단일 마스터 키의 위험을 분산시키고 성능과 보안을 동시에 달성하기 위한 구조입니다.

3계층 구조가 표준입니다. 최상위에 HSM(Hardware Security Module) 또는 KMS CMK인 루트 키(KEK: Key Encrypting Key)가 위치합니다. 루트 키는 절대 데이터를 직접 암호화하지 않으며 오직 하위 키를 암호화하는 데만 사용됩니다. AWS KMS CMK, Azure Key Vault 키가 이에 해당합니다.

중간 계층은 데이터 암호화 키(DEK: Data Encrypting Key)로, 실제 데이터를 암호화하는 대칭 키(AES-256)입니다. DEK는 루트 키로 암호화된 형태(Envelope Encryption)로 데이터 옆에 저장됩니다. 데이터 복호화 시 루트 키로 DEK를 먼저 복호화하고, 복호화된 DEK로 데이터를 복호화합니다. DEK는 메모리에서만 존재하고 디스크에는 암호화된 형태로만 보관됩니다.

이 설계의 장점은 루트 키를 교체(Key Rotation)할 때 데이터를 전부 재암호화할 필요 없이 DEK만 재암호화하면 된다는 점입니다. 또한 루트 키를 HSM에 격리하면 성능 병목 없이 보안을 극대화합니다.

---

**Q32. Kubernetes에서 멀티 클러스터 접근 관리(Multi-cluster Access Management)를 보안적으로 구현하는 방법을 설명하세요.**

**A.** 멀티 클러스터 환경에서 각 클러스터에 별도의 kubeconfig와 ServiceAccount를 관리하는 방식은 자격증명 분산이라는 운영 악몽을 초래합니다.

중앙화된 접근 관리로 Teleport나 Boundary 같은 접근 프록시를 도입하면 단일 접근 포인트를 통해 모든 클러스터에 접근할 수 있습니다. 사용자는 클러스터 자격증명이 아닌 중앙 IdP 자격증명으로 인증하고, 단기 수명 인증서로 클러스터 API를 호출합니다.

EKS에서는 `aws eks get-token` 명령으로 IAM 자격증명을 Kubernetes 인증 토큰으로 변환합니다. 이를 통해 IAM 역할 기반 접근 제어가 Kubernetes에 그대로 적용됩니다. `aws-auth` ConfigMap에서 IAM 역할과 Kubernetes RBAC 그룹을 매핑합니다. 주의사항은 `aws-auth` ConfigMap이 잘못 구성되면 클러스터 접근이 완전 차단될 수 있으므로, EKS Access Entries(2024년 출시, IAM 기반 접근 관리의 개선판)로 마이그레이션을 권장합니다.

ArgoCD나 Flux 같은 GitOps 도구가 멀티 클러스터를 관리할 때는 각 클러스터에 최소 권한의 ServiceAccount를 사용하고, 클러스터 간 권한을 상호 격리합니다.

---

**Q33. OWASP Top 10 2021의 "Broken Access Control"이 클라우드 환경에서 구체적으로 어떻게 나타나는지, 그리고 IAM 설계로 이를 방지하는 방법을 설명하세요.**

**A.** 클라우드 환경에서 Broken Access Control의 대표적 패턴들을 살펴보면, 과도하게 허용적인 S3 버킷 정책(`Principal: *`, `Action: "s3:*"`)은 전체 인터넷이 데이터에 접근할 수 있게 만드는 가장 흔한 사례입니다. S3 퍼블릭 엑세스 차단(Public Access Block)을 계정 레벨에서 활성화하고, SCP로 이를 비활성화하지 못하도록 차단해야 합니다.

임시 IAM 역할을 잘못 구성하여 `Principal: *`(임의의 주체)가 역할을 수임할 수 있게 하면 권한 상승 경로가 됩니다. 역할 신뢰 정책의 Principal은 항상 가장 구체적인 ARN으로 지정해야 합니다.

Kubernetes에서 `automountServiceAccountToken: true` 기본 설정으로 인해 파드가 Kubernetes API를 자동으로 호출할 수 있는 토큰을 보유하며, 이 토큰이 컨테이너 탈출 시 내부 정보 수집에 악용됩니다.

감사 도구로 AWS IAM Access Analyzer는 외부 접근 가능한 리소스를 자동으로 탐지합니다. CSPM 도구는 클라우드 전체의 접근 제어 오류를 지속적으로 스캔합니다.

---

**Q34. 서비스 메시(Istio)의 mTLS 구현에서 PERMISSIVE 모드와 STRICT 모드의 차이, 그리고 PERMISSIVE에서 STRICT로 마이그레이션하는 안전한 전략을 설명하세요.**

**A.** PERMISSIVE 모드는 mTLS와 평문 트래픽을 모두 허용합니다. 기존 서비스가 사이드카 없이 통신하는 환경에서 Istio를 점진적으로 도입할 때 사용합니다. 모든 트래픽이 암호화되지 않으므로 실제 보안 보장은 없습니다.

STRICT 모드는 mTLS 트래픽만 허용하고 평문을 거부합니다. 모든 서비스가 Envoy 사이드카를 통해 통신해야 합니다.

안전한 마이그레이션 전략은 점진적 접근입니다. 먼저 Kiali나 Jaeger로 현재 트래픽 흐름을 시각화하고 mTLS가 없는 통신 경로를 파악합니다. 네임스페이스 단위로 STRICT를 적용하되, 영향이 적은 테스트 환경에서 시작합니다. 각 서비스에 사이드카가 주입되고 mTLS로 통신하는 것을 확인한 후 STRICT로 전환합니다. Istio의 `DestinationRule`로 특정 서비스 쌍 간의 TLS 모드를 개별 제어하며 단계적으로 마이그레이션합니다. Istio 메트릭에서 `istio_requests_total`의 security_policy 레이블로 mTLS vs 평문 비율을 모니터링합니다.

---

**Q35. 클라우드 보안 사고 대응(Cloud IR)에서 증거 보전(Evidence Preservation)과 포렌식 수집의 모범 사례를 설명하세요.**

**A.** 클라우드 환경의 포렌식은 전통적 디지털 포렌식과 다릅니다. 인스턴스가 언제든 종료될 수 있고, 메모리는 스냅샷으로 캡처하기 어려우며, 로그가 여러 서비스에 분산되어 있습니다.

즉각 수행할 증거 보전 작업으로, 영향받은 EC2 인스턴스의 EBS 볼륨 스냅샷을 즉시 생성합니다(인스턴스 종료 전). 스냅샷은 삭제되지 않도록 태그로 표시하고, 별도의 격리된 계정으로 복사합니다. VPC Flow Logs, CloudTrail, Application Logs를 즉시 S3에 보존하고, S3 Object Lock으로 보호합니다. 관련 IAM 자격증명의 접근 이력을 `aws iam get-credential-report`와 `aws cloudtrail lookup-events`로 수집합니다. 침해된 역할의 세션을 `sts:RevokeSession`으로 무효화하고, 새 자격증명 발급을 차단합니다.

AWS EC2 Systems Manager Session Manager 로그가 활성화되어 있다면 원격 세션 명령 이력도 확보합니다. CloudTrail의 `userIdentity.sessionContext`는 누가 어떤 역할을 수임하여 작업했는지의 전체 체인을 보여주는 핵심 포렌식 데이터입니다.

---

## 마무리: Senior Cloud Engineer가 갖추어야 할 보안 사고의 틀

이 문서에서 다룬 내용의 핵심은 보안을 사후 추가(Bolt-on Security)가 아닌 아키텍처의 첫 번째 클래스 시민으로 설계하는 것입니다. AWS, Azure, Kubernetes 각각의 보안 메커니즘은 독립적으로 이해하는 것보다, 이 모두를 관통하는 원칙인 **최소 권한, 아이덴티티 중심 접근 제어, 깊이 있는 방어(Defense in Depth), 지속적 검증**을 이해하는 것이 더 중요합니다.

대기업 면접에서 기술 지식뿐만 아니라 "왜 이 설계를 선택했는가", "트레이드오프는 무엇인가", "실제 장애 시 어떻게 대응했는가"를 설명할 수 있는 것이 Senior Engineer를 구별하는 핵심입니다.
