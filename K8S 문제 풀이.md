### **Section 1: 아키텍처 및 내부 동작 (Architecture & Internals)**

**Q1. Kubernetes Control Plane의 핵심 컴포넌트인 `etcd`의 역할과, 고가용성(HA) 구성 시 홀수(3, 5개) 노드로 구성해야 하는 이유를 설명하시오.**

* **해설:** `etcd`는 클러스터의 모든 상태 데이터(설정, 메타데이터 등)를 저장하는 일관성 있는 고가용성 Key-Value 저장소입니다. 홀수 노드로 구성하는 이유는 분산 합의 알고리즘인 Raft 알고리즘 때문입니다. 클러스터가 정상 동작하려면 과반수(Quorum) 이상의 노드가 살아있어야 하는데, 짝수(예: 4개)일 때와 홀수(예: 3개)일 때 허용 가능한 장애 노드 수가 동일(1개)하여 자원 낭비가 발생하며, 네트워크 파티션 발생 시 Split-Brain(뇌 분리) 현상을 방지하고 리더 선출을 원활하게 하기 위함입니다.

**Q2. `kube-scheduler`가 Pod를 노드에 할당하는 과정(Scheduling Cycle)을 `Filtering`과 `Scoring` 단계로 나누어 설명하시오.**

* **해설:**
1. **Filtering (Predicates):** Pod가 배치될 수 *없는* 노드를 걸러내는 단계입니다. 예를 들어, 노드의 리소스(CPU/Memory) 부족, Taint/Toleration 불일치, Node Selector 불일치 등을 확인하여 부적합한 노드를 제외합니다.
2. **Scoring (Priorities):** Filtering을 통과한 노드들에게 점수를 부여하여 가장 적합한 노드를 선정하는 단계입니다. Image Locality(이미지가 이미 있는지), Least Requested(자원이 많이 남은 곳) 등의 함수를 통해 점수를 매기고, 가장 높은 점수의 노드에 Pod를 바인딩합니다.



**Q3. `Informer`와 `SharedInformer` 패턴이 Controller 개발 시 왜 중요한지, 그리고 이것이 API Server의 부하를 어떻게 줄여주는지 설명하시오.**

* **해설:** Controller가 API Server에 매번 `LIST`나 `WATCH` 요청을 보내면 API Server에 과도한 부하가 발생합니다. `Informer`는 로컬 캐시(Local Store)를 유지하며 API Server의 변경 사항을 감지해 동기화합니다. `SharedInformer`는 여러 Controller가 동일한 리소스를 감시할 때, 하나의 연결만 사용하여 이 캐시를 공유함으로써 메모리 사용량과 API 호출 빈도를 획기적으로 줄여주는 패턴입니다.

**Q4. `Pause Container`의 역할은 무엇이며, Pod 내의 컨테이너들이 네트워크와 스토리지 볼륨을 공유하는 원리는 무엇인가?**

* **해설:** `Pause Container`는 Pod가 생성될 때 가장 먼저 실행되어 Pod의 네트워크 네임스페이스(Network Namespace)와 IPC 등을 생성하고 유지하는 역할을 합니다. Pod 내의 다른 애플리케이션 컨테이너들은 이 `Pause Container`의 네임스페이스에 조인(Join)하는 방식으로 실행되므로, 같은 IP 주소(localhost)와 볼륨을 공유할 수 있게 됩니다.

---

### **Section 2: 자원 최적화 및 오토스케일링 (Resource Optimization)**

**Q5. `Requests`와 `Limits`의 차이를 설명하고, `Guaranteed`, `Burstable`, `BestEffort` QoS(Quality of Service) 클래스가 어떻게 결정되는지 서술하시오.**

* **해설:**
* `Requests`: 스케줄러가 Pod를 배치할 때 보장해야 하는 최소 자원량.
* `Limits`: 컨테이너가 사용할 수 있는 최대 자원량 (초과 시 CPU는 Throttling, Memory는 OOMKilled).
* **QoS 결정:**
* `Guaranteed`: 모든 컨테이너에 Requests와 Limits가 설정되어 있고, 그 값이 동일할 때.
* `Burstable`: 최소 하나 이상의 컨테이너가 Requests/Limits를 가지지만 Guaranteed 조건은 아닐 때.
* `BestEffort`: Requests와 Limits가 전혀 설정되지 않았을 때.





**Q6. HPA(Horizontal Pod Autoscaler)와 VPA(Vertical Pod Autoscaler)의 차이점을 설명하고, 두 오토스케일러를 동시에 사용할 때 발생할 수 있는 문제점과 해결 방안을 제시하시오.**

* **해설:** HPA는 CPU/Mem 사용량에 따라 Pod의 **개수**를 늘리는 반면, VPA는 Pod의 **Requests/Limits 크기**를 조정하여 재생성합니다. 두 기능을 동일한 지표(예: CPU)로 동시에 사용하면 서로 간섭(Thrashing)이 발생할 수 있습니다. 예를 들어, HPA가 부하를 감지해 Pod를 늘리려는데 VPA가 리소스를 늘려버리면 HPA의 조건이 변경될 수 있습니다. 해결책은 HPA는 CPU/Mem 등의 자원 사용량으로, VPA는 OOM 방지용으로 사용하거나, VPA를 `Recommendation` 모드로만 사용하여 자동 적용을 막는 것입니다. 또는 Goldilocks 같은 도구로 적정 리소스를 추천받아 수동 적용합니다.

**Q7. 노드의 리소스 활용률을 높이기 위해 `Overcommitment`를 구성하려 한다. 이때 발생할 수 있는 `Eviction` 정책과 노드 압박(Node Pressure) 상황에서의 동작 방식을 설명하시오.**

* **해설:** Overcommitment는 실제 물리 자원보다 더 많은 양을 Limits의 합계로 설정하는 것입니다. 노드의 실제 사용 가능한 자원이 부족해지면 `kubelet`은 Node Pressure 상태를 감지하고 Eviction(축출)을 시작합니다. 이때 QoS 클래스 순서인 `BestEffort` -> `Burstable` -> `Guaranteed` 순서로 Pod를 종료시켜 자원을 확보합니다. 따라서 중요한 파드는 반드시 Guaranteed로 설정해야 합니다.

---

### **Section 3: 네트워크 및 보안 (Networking & Security)**

**Q8. CNI(Container Network Interface) 플러그인의 역할은 무엇이며, Overlay Network(예: VXLAN)와 Host-Gateway 방식의 차이점을 설명하시오.**

* **해설:** CNI는 컨테이너 간의 네트워크 연결성 제공, IP 할당/회수 등을 표준화한 인터페이스입니다.
* **Overlay (VXLAN, IPIP):** 패킷을 캡슐화하여 기존 L3 네트워크 위에서 가상의 L2 네트워크를 구성합니다. 설정이 쉽고 호환성이 좋으나 캡슐화 오버헤드가 있습니다. (예: Calico VXLAN, Flannel).
* **Host-Gateway (BGP 등):** 캡슐화 없이 라우팅 테이블을 통해 직접 통신합니다. 성능이 우수하나 L2 연결성이 필요하거나 라우터 설정이 필요할 수 있습니다. (예: Calico BGP).



**Q9. `NetworkPolicy`를 사용하여 네임스페이스 간의 격리(Isolation)를 구현하는 방법을 `Ingress`와 `Egress` 관점에서 설명하시오.**

* **해설:** `NetworkPolicy`는 기본적으로 `Allow` 방식이 아닌 `Deny` 기반(정책 생성 시 해당 Pod는 차단됨)으로 동작합니다. 특정 네임스페이스의 Pod들을 선택(PodSelector)한 뒤, `Ingress`(들어오는 트래픽) 규칙에서 허용할 `namespaceSelector`나 `ipBlock`을 명시하고, `Egress`(나가는 트래픽) 또한 필요한 목적지만 허용함으로써 Whitelist 기반의 보안 격리를 구현합니다.

**Q10. Kubernetes `ServiceAccount`와 `UserAccount`의 차이, 그리고 RBAC(Role-Based Access Control)에서 `Role`과 `ClusterRole`의 차이를 설명하시오.**

* **해설:**
* `ServiceAccount`: Pod 내부의 프로세스가 API Server와 통신하기 위한 계정 (네임스페이스에 종속됨).
* `UserAccount`: 사람(개발자, 관리자)을 위한 계정 (쿠버네티스에서 직접 관리하지 않고 외부 IdP 등과 연동).
* `Role`: 특정 네임스페이스 내에서만 유효한 권한 정의.
* `ClusterRole`: 클러스터 전체(모든 네임스페이스, 노드 등)에 유효한 권한 정의.



---

### **Section 4: 스토리지 및 데이터 관리 (Storage & Data)**

**Q11. `PV`(PersistentVolume)와 `PVC`(PersistentVolumeClaim)의 생명주기를 설명하고, `StorageClass`의 `ReclaimPolicy` (`Retain`, `Delete`)가 데이터 보존에 미치는 영향을 서술하시오.**

* **해설:** 사용자가 PVC를 생성하면, 쿠버네티스는 적절한 PV를 찾아 바인딩(Binding)합니다. (동적 프로비저닝의 경우 StorageClass에 의해 PV가 자동 생성됨). 사용이 끝나 PVC를 삭제할 때, `ReclaimPolicy`가 작동합니다.
* `Retain`: PV는 유지되지만 'Released' 상태가 되며, 데이터는 보존되나 다른 PVC가 바로 재사용할 수는 없습니다(수동 정리 필요).
* `Delete`: 연결된 스토리지 자산(AWS EBS 등)과 PV가 함께 삭제됩니다.



**Q12. `StatefulSet`이 `Deployment`와 다른 점 3가지를 들고, 왜 데이터베이스와 같은 애플리케이션에 적합한지 설명하시오.**

* **해설:**
1. **고유한 식별자:** Pod 이름이 `web-0`, `web-1`처럼 고정된 순서와 인덱스를 가집니다.
2. **안정적인 네트워크 ID:** Headless Service를 통해 `web-0.service-name`과 같은 고정 도메인을 가집니다.
3. **안정적인 스토리지:** 각 Pod가 고유한 PVC와 매핑되어 Pod가 재생성되어도 원래의 데이터를 유지합니다.
이러한 특성 때문에 마스터-슬레이브 구조나 데이터 지속성이 중요한 DB 운영에 필수적입니다.



---

### **Section 5: 장애 처리 및 트러블슈팅 (Troubleshooting)**

**Q13. Pod가 `CrashLoopBackOff` 상태에 빠지는 주된 원인 3가지를 나열하고, 이를 디버깅하기 위한 명령어 순서를 설명하시오.**

* **해설:**
* **원인:** 애플리케이션 코드 오류/패닉, 환경변수/설정 파일 누락, Liveness Probe 실패로 인한 반복 재시작.
* **디버깅:**
1. `kubectl describe pod <pod-name>`: 이벤트 로그(Events)를 확인하여 Probe 실패나 종료 코드 확인.
2. `kubectl logs <pod-name> (--previous)`: 현재 또는 직전 컨테이너의 표준 출력 로그 확인.
3. 설정 문제 의심 시 `kubectl get pod <pod-name> -o yaml`로 설정 검토.





**Q14. `ImagePullBackOff` 에러가 발생했을 때 확인해야 할 사항들을 우선순위대로 나열하시오.**

* **해설:**
1. **이미지 이름/태그 오타:** 이미지가 레지스트리에 실제로 존재하는지 확인.
2. **인증 정보(ImagePullSecrets):** Private Registry 사용 시 Secret이 생성되어 있고 Pod/ServiceAccount에 설정되어 있는지 확인.
3. **네트워크 문제:** 노드에서 레지스트리(Docker Hub 등)로의 아웃바운드 통신이 방화벽 등으로 막혀있는지 확인.



**Q15. 노드가 `NotReady` 상태가 되었다. 원인을 파악하기 위해 마스터 노드와 해당 워커 노드에서 각각 확인해야 할 로그와 데몬은 무엇인가?**

* **해설:**
* **마스터 노드:** `kubectl describe node <node-name>`을 통해 Condition 확인.
* **워커 노드:** `kubelet` 데몬의 상태 (`systemctl status kubelet`) 및 로그 (`journalctl -u kubelet`) 확인. 또한 컨테이너 런타임(containerd/docker)의 상태 확인, 디스크 공간(`df -h`), 메모리 부족 여부 등을 점검해야 합니다.



---

### **Section 6: 고급 운영 및 배포 전략 (Advanced Ops)**

**Q16. `Rolling Update`, `Blue/Green`, `Canary` 배포 전략의 차이점과 Kubernetes에서 각각을 구현하는 일반적인 방법을 설명하시오.**

* **해설:**
* **Rolling Update:** 구버전 Pod를 하나씩 줄이고 신버전 Pod를 하나씩 늘리는 방식 (Deployment 기본 전략). 무중단 배포가 가능하나 롤백이 느릴 수 있음.
* **Blue/Green:** 신버전(Green)을 구버전(Blue)만큼 모두 띄운 후 Service의 Selector를 교체하여 트래픽을 일시에 전환. 자원이 2배 필요하나 롤백이 즉시 가능.
* **Canary:** 신버전 Pod를 소수만 띄우고 트래픽의 일부(예: 10%)만 흘려보내 테스트한 후 점차 확대. Ingress Controller나 Istio 같은 Service Mesh를 통해 정교한 트래픽 제어로 구현.



**Q17. GitOps의 개념을 설명하고, ArgoCD와 같은 도구를 사용할 때 기존 `kubectl apply` 방식 대비 장점을 보안과 운영 관점에서 서술하시오.**

* **해설:** GitOps는 Git 리포지토리를 '단일 진실 공급원(Source of Truth)'으로 삼아 인프라와 애플리케이션을 선언적으로 관리하는 방식입니다.
* **장점:**
1. **히스토리 관리 및 롤백:** Git 커밋으로 변경 이력이 남고 `git revert`로 쉬운 롤백 가능.
2. **보안:** 개발자가 클러스터에 직접 접근할 필요 없이 Git 권한만 있으면 되므로 클러스터 자격 증명(Credential) 노출 최소화.
3. **Drift 감지:** 실제 클러스터 상태가 Git 정의와 달라지면(누군가 수동 수정 시) ArgoCD가 이를 감지하고 자동 복구(Self-healing) 가능.





**Q18. `Sidecar` 패턴의 정의와 대표적인 사용 사례(Logging, Proxy)를 설명하시오.**

* **해설:** Sidecar 패턴은 메인 애플리케이션 컨테이너와 동일한 Pod 내에 보조 컨테이너를 함께 배치하여 기능을 확장하는 패턴입니다.
* **Logging:** 앱이 로그를 파일에 쓰면 사이드카가 이를 읽어 중앙 로그 서버(Elasticsearch 등)로 전송.
* **Proxy (Service Mesh):** Envoy 같은 프록시를 사이드카로 두어 mTLS, 트래픽 제어, 서킷 브레이커 등의 네트워크 로직을 앱 수정 없이 처리.



**Q19. Kubernetes `CRD`(Custom Resource Definition)와 `Operator` 패턴이 무엇인지 설명하고, 이것이 필요한 상황을 예로 드시오.**

* **해설:** CRD는 쿠버네티스 API를 확장하여 사용자가 정의한 새로운 리소스 타입을 만드는 것입니다. Operator는 이 CRD를 감시하며 실제 상태를 원하는 상태로 맞추는 사용자 정의 컨트롤러입니다.
* **예시:** 'Prometheus'를 배포할 때 단순 Deployment가 아니라 `Prometheus`라는 CRD를 만들고, Operator가 설정 변경 시 자동으로 설정 리로드나 클러스터링 재구성을 처리하도록 할 때 필요합니다. (복잡한 상태 관리가 필요한 앱의 자동화).



**Q20. `Taint`와 `Toleration` 그리고 `Node Affinity`의 차이를 설명하고, 특정 노드군(예: GPU 노드)에만 특정 Pod를 배치하려면 어떻게 조합해야 하는지 설명하시오.**

* **해설:**
* `Taint`: 노드에 설정하여, 자격 없는 Pod가 오지 못하게 막음 (밀어냄).
* `Toleration`: Pod에 설정하여, 해당 Taint가 있는 노드에 들어갈 자격을 얻음 (들어갈 수 있음, 보장은 아님).
* `Node Affinity`: Pod가 특정 노드를 선호하거나 필수적으로 요구함 (당김).
* **GPU 노드 전용 배치 조합:** GPU 노드에 Taint(`gpu=true:NoSchedule`)를 걸어 일반 Pod를 차단하고, GPU Pod에는 해당 Toleration을 부여합니다. 동시에 GPU Pod에 Node Affinity(`key: gpu, operator: In, values: true`)를 설정하여 반드시 그 노드로 찾아가도록 구성합니다. (Taint만 쓰면 다른 일반 노드로 갈 수도 있기 때문).

시니어 클라우드 엔지니어를 위한 Kubernetes 심화 문제 **21번부터 50번**까지 이어서 제공해 드립니다.

이번 섹션에서는 **보안 심화, 관찰 가능성(Observability), 스토리지 고급, 그리고 패키지 관리 및 운영 자동화** 영역을 중점적으로 다룹니다.

---

### **Section 7: 보안 심화 (Advanced Security)**

**Q21. `SecurityContext`에서 `runAsUser`, `runAsGroup`, `fsGroup`의 차이를 설명하고, `AllowPrivilegeEscalation: false` 설정이 왜 중요한지 서술하시오.**

* **해설:**
* `runAsUser/Group`: 컨테이너 프로세스가 실행될 UID/GID를 지정합니다.
* `fsGroup`: Pod 내에서 마운트된 볼륨의 소유권을 해당 GID로 변경하여 쓰기 권한을 부여합니다.
* `AllowPrivilegeEscalation: false`: 프로세스가 부모 프로세스보다 더 많은 권한(예: SUID 바이너리 실행)을 얻는 것을 방지합니다. 이는 컨테이너 탈옥(Breakout) 공격을 막는 핵심 설정입니다.



**Q22. Kubernetes Secrets는 기본적으로 base64로 인코딩만 되어 있다. etcd에 저장될 때 암호화(Encryption at Rest)를 적용하는 방법(EncryptionConfiguration)을 설명하시오.**

* **해설:** API Server 시작 옵션에 `--encryption-provider-config` 플래그를 추가하고, 해당 설정 파일(YAML)에 암호화 키와 공급자(aescbc, secretbox 등)를 정의해야 합니다. 이 설정을 적용한 후, 기존에 평문으로 저장된 Secret들을 암호화하기 위해선 모든 Secret을 다시 읽어서 쓰는(Rewrite) 작업(`kubectl get secrets --all-namespaces -o json | kubectl replace -f -`)이 필요합니다.

**Q23. OPA(Open Policy Agent) 또는 Kyverno와 같은 Policy Engine이 필요한 이유를 `PodSecurityPolicy(PSP)`의 폐기(Deprecation)와 연관 지어 설명하시오.**

* **해설:** K8s v1.25부터 PSP가 완전히 제거되었습니다. 이를 대체하고 더 유연한 정책(예: 특정 레지스트리 이미지만 허용, Ingress 호스트 중복 방지 등)을 구현하기 위해 OPA Gatekeeper나 Kyverno가 사용됩니다. 이들은 Admission Controller 단계(Validating/Mutating Webhook)에서 요청을 가로채어 정의된 정책(Rego 등)에 따라 리소스 생성을 승인하거나 거부합니다.

**Q24. `ServiceAccount` 토큰의 `Projected Volume` 방식과 기존 Secret 기반 토큰의 차이점, 그리고 이것이 보안에 더 유리한 이유를 설명하시오.**

* **해설:** 기존 방식은 토큰이 만료되지 않는 정적 Secret으로 생성되었습니다. Projected ServiceAccount Token은 `TokenRequest` API를 통해 발급되며, **만료 시간(TTL)**이 있고, **특정 관객(Audience)**에게만 유효하도록 제한할 수 있습니다. 또한 Pod가 삭제되면 토큰도 즉시 무효화되므로 키 유출 시 리스크가 적습니다.

**Q25. 컨테이너 런타임 보안을 위해 `AppArmor` 또는 `Seccomp` 프로파일을 적용하는 원리와, 이를 통해 막을 수 있는 공격 유형을 예로 드시오.**

* **해설:**
* `Seccomp`(Secure Computing Mode): 컨테이너가 호스트 커널에 요청할 수 있는 시스템 콜(System Call)을 제한합니다(예: `reboot`, `sysctl` 호출 차단).
* `AppArmor`: 파일 시스템 경로에 대한 읽기/쓰기/실행 권한을 세밀하게 제어합니다.
* 이를 통해 공격자가 쉘을 획득하더라도 시스템 중요 파일에 접근하거나 커널 취약점을 이용한 권한 상승을 시도하는 것을 차단할 수 있습니다.



---

### **Section 8: 관찰 가능성 (Observability - Monitoring & Logging)**

**Q26. Prometheus의 `Pull` 방식 아키텍처가 `Push` 방식 대비 가지는 장단점을 설명하고, Kubernetes 환경에서 `ServiceMonitor`(Prometheus Operator)가 하는 역할을 서술하시오.**

* **해설:**
* **Pull 방식:** 서버가 타겟(Exporter)에 접속해 메트릭을 가져옵니다. 타겟의 부하를 중앙에서 조절하기 쉽고, 타겟이 죽었는지(Down) 즉시 알 수 있다는 장점이 있지만, 단기 작업(Short-lived Job)의 메트릭 수집은 어렵습니다(PushGateway 필요).
* **ServiceMonitor:** K8s의 Service 리소스를 감지하여, 해당 서비스 뒤에 있는 Pod들의 엔드포인트를 자동으로 Prometheus의 스크래핑 타겟으로 등록해주는 CRD입니다. 이를 통해 Pod가 동적으로 변경되어도 설정 변경 없이 메트릭 수집이 가능합니다.



**Q27. `Liveness`, `Readiness`, `Startup` Probe의 차이를 명확히 구분하고, `Startup Probe`가 필요한 구체적인 시나리오(예: 레거시 앱)를 설명하시오.**

* **해설:**
* `Liveness`: 실패 시 컨테이너를 **재시작**합니다 (교착 상태 해결).
* `Readiness`: 실패 시 서비스 엔드포인트에서 **제외**하여 트래픽을 차단합니다 (부팅 중 트래픽 유입 방지).
* `Startup`: 이 Probe가 성공할 때까지 Liveness/Readiness 검사를 보류합니다.
* **시나리오:** 초기 데이터 로딩이나 웜업으로 부팅에 3분 이상 걸리는 레거시 Java 앱의 경우, Liveness만 설정하면 부팅 도중 계속 재시작되는 루프에 빠집니다. 이때 Startup Probe를 길게 주어 부팅 시간을 벌어줘야 합니다.



**Q28. 로그 수집 아키텍처 중 `Sidecar` 패턴(예: fluent-bit 사이드카)과 `Node-level Agent` 패턴(예: DaemonSet)의 차이와 각각의 장단점을 설명하시오.**

* **해설:**
* **Node-level (DaemonSet):** 노드당 하나의 에이전트가 `/var/log/containers/*`를 수집. 자원 효율적이지만, 로그 형식을 앱별로 파싱하기 까다로울 수 있습니다.
* **Sidecar:** 각 Pod 안에 로깅 에이전트를 함께 배포. 앱별로 커스텀 로그 포맷 처리가 쉽고 로그를 별도 저장소로 보내기 용이하지만, Pod마다 리소스를 추가로 소모하므로 클러스터 전체 자원 사용량이 증가합니다.



**Q29. 분산 트레이싱(Distributed Tracing) 도구인 Jaeger나 OpenTelemetry를 도입할 때, 애플리케이션 코드 레벨에서 `Context Propagation`이 왜 필요한지 설명하시오.**

* **해설:** 마이크로서비스 환경에서 하나의 요청이 여러 서비스를 거쳐갈 때, 이 요청이 연결된 하나의 트랜잭션임을 식별하기 위해 `Trace ID`와 `Span ID`를 HTTP 헤더(예: `traceparent`)에 실어 다음 서비스로 넘겨줘야 합니다. 이를 Context Propagation이라 하며, 이것이 끊기면 트레이싱이 단절되어 전체 흐름을 볼 수 없습니다.

**Q30. Kubernetes 이벤트(`kubectl get events`)는 영구적으로 저장되지 않는다. 이를 장기 보관하고 검색하기 위한 일반적인 아키텍처를 설명하시오.**

* **해설:** K8s 이벤트는 etcd에 저장되며 기본 1시간 후 만료됩니다. 이를 보존하기 위해 `event-exporter`나 `fluentd` 등을 사용하여 이벤트를 수집한 뒤, Elasticsearch, Loki, 또는 외부 로깅 시스템으로 전송하여 시계열로 저장하고 검색할 수 있도록 구성해야 합니다.

---

### **Section 9: 고급 스토리지 및 데이터 (Advanced Storage)**

**Q31. `Local Persistent Volume`과 `HostPath`의 결정적인 차이점은 무엇이며, 스케줄러가 `VolumeBindingMode: WaitForFirstConsumer`를 사용할 때 어떻게 동작하는지 설명하시오.**

* **해설:**
* `HostPath`: 노드의 파일 시스템을 직접 쓰지만, 스케줄러가 이 볼륨이 어느 노드에 있는지 모릅니다. Pod가 다른 노드에 뜨면 데이터를 잃습니다.
* `Local PV`: 노드의 로컬 디스크를 사용하면서도 스케줄러가 해당 볼륨이 있는 노드를 인지하고 Pod를 그 노드에 고정(Affinity)합니다.
* `WaitForFirstConsumer`: PVC를 만들 때 바로 PV를 바인딩하지 않고, 해당 PVC를 사용하는 Pod가 생성되어 스케줄링될 때까지 기다렸다가, 그 Pod가 배정된 노드의 적절한 PV와 바인딩합니다. (로컬 볼륨 사용 시 필수 설정).



**Q32. CSI(Container Storage Interface)의 등장 배경과, CSI 드라이버가 `Controller` 플러그인과 `Node` 플러그인으로 나뉘어 동작하는 이유를 설명하시오.**

* **해설:** 과거에는 스토리지 코드가 K8s 코어 내부에 있었으나(In-tree), 유지보수가 어려워 별도 인터페이스(CSI)로 분리했습니다.
* `Controller`: 볼륨 생성/삭제, Attach/Detach 등 클라우드 API 제어 담당 (보통 Deployment/StatefulSet으로 배포).
* `Node`: 각 노드에서 실제 볼륨을 마운트/언마운트하고 포맷하는 역할 담당 (DaemonSet으로 배포되어 모든 노드에서 실행).



**Q33. 데이터베이스와 같은 Stateful 앱 운영 시 `Pod Disruption Budget(PDB)`을 설정해야 하는 이유를 노드 유지보수(Drain) 상황과 연결하여 설명하시오.**

* **해설:** PDB는 자발적인 중단(Voluntary Disruption, 예: 노드 업그레이드, 스케일 다운) 시 동시에 다운될 수 있는 Pod의 수(maxUnavailable)나 유지해야 하는 최소 수(minAvailable)를 제한합니다. 예를 들어 3중화된 DB에서 `minAvailable: 2`를 설정하면, `kubectl drain` 수행 시 쿼럼이 깨지지 않도록 한 번에 하나의 Pod만 축출되도록 강제하여 서비스 가용성을 보장합니다.

---

### **Section 10: 네트워크 심화 (Advanced Networking)**

**Q34. `CoreDNS`에서 발생하는 `ndots:5` 문제(DNS 조회 지연)의 원인과, 이를 해결하기 위한 `dnsConfig` 최적화 방안을 설명하시오.**

* **해설:** 리눅스(glibc)는 도메인 조회 시 점(.) 개수가 `ndots` 설정값(기본 5)보다 적으면 `search domain` 목록(namespace.svc.cluster.local 등)을 차례로 붙여서 먼저 조회합니다. 외부 도메인(예: https://www.google.com/search?q=google.com)을 조회할 때도 불필요한 내부 도메인 조회가 3~4회 먼저 발생하여 지연이 생깁니다. 해결책은 Pod의 `dnsConfig`에서 `ndots`를 1~2로 줄이거나, FQDN(마지막에 점 `.` 포함)을 사용하는 것입니다.

**Q35. Ingress Controller에서 `Nginx Ingress`와 AWS `ALB Ingress`의 동작 방식 차이(L4/L7 처리 위치)를 설명하시오.**

* **해설:**
* `Nginx Ingress`: NLB(L4)를 통해 트래픽을 클러스터 내부의 Nginx Pod로 받은 후, Nginx가 내부 로직으로 L7 라우팅을 처리합니다.
* `AWS ALB Ingress`: AWS의 ALB(L7) 리소스가 직접 생성되며, ALB가 트래픽을 받아 클러스터 내의 각 Pod IP(Target Group)로 직접 라우팅(IP 모드)하거나 NodePort로 라우팅합니다.



**Q36. Service Mesh(Istio 등)를 도입했을 때 얻을 수 있는 이점인 `mTLS`, `Circuit Breaking`, `Traffic Shifting`에 대해 설명하시오.**

* **해설:**
* `mTLS`: 서비스 간 통신을 자동으로 암호화하고 상호 인증하여 내부 보안(Zero Trust)을 강화합니다.
* `Circuit Breaking`: 장애가 발생한 서비스로 가는 요청을 즉시 차단하여 전체 시스템으로 장애가 전파되는 것을 막습니다.
* `Traffic Shifting`: Canary 배포 시 헤더나 가중치 기반으로 트래픽의 1%만 신규 버전에 보내는 등의 정교한 제어가 가능합니다.



---

### **Section 11: 패키지 관리 및 운영 자동화 (Helm & GitOps)**

**Q37. Helm Chart에서 `values.yaml`의 역할과, `_helpers.tpl` (Named Templates)을 사용하여 차트의 재사용성을 높이는 방법을 설명하시오.**

* **해설:** `values.yaml`은 사용자 정의 설정값(이미지 태그, 리소스 양 등)을 제공하는 인터페이스입니다. `_helpers.tpl`은 중복되는 로직(예: 앱 이름 생성, 공통 라벨 생성)을 함수처럼 정의해두고, 여러 템플릿 파일에서 `{{ include "chart.fullname" . }}` 형태로 호출하여 코드 중복을 줄이고 일관성을 유지하게 합니다.

**Q38. Helm의 `3-way merge` 패치 방식이 Helm v2의 `2-way merge`와 다른 점, 그리고 이것이 `kubectl edit` 등으로 수동 변경된 리소스를 롤백할 때 어떤 이점이 있는지 서술하시오.**

* **해설:** Helm v3는 (1) 이전 릴리즈의 매니페스트, (2) 현재 차트의 매니페스트, (3) **현재 클러스터의 실제 상태(Live State)** 3가지를 비교합니다. 이를 통해 누군가 `kubectl edit`으로 긴급 패치한 사항을 덮어쓰지 않거나, 롤백 시 실제 상태를 반영하여 충돌 없이 더 정확하게 원복할 수 있습니다.

**Q39. Kustomize가 Helm과 다른 `Overlay` 방식의 설정 관리 철학을 설명하고, `Base`와 `Overlay` 구조를 통해 환경별(Dev/Prod) 설정을 분리하는 방법을 예시로 드시오.**

* **해설:** Helm이 템플릿 치환 방식이라면, Kustomize는 원본 YAML(`Base`)을 건드리지 않고, 그 위에 변경 사항(`Overlay`)을 덧입히는 패치 방식입니다. `Base` 폴더에 공통 Deployment를 두고, `Overlays/dev`에는 `replicas: 1`을, `Overlays/prod`에는 `replicas: 3`과 `resource limits`를 추가하는 패치(patchesStrategicMerge)를 두어 관리합니다.

**Q40. Kubernetes Operator 패턴의 성숙도 모델(Capability Levels) 5단계를 순서대로 나열하고, `Level 4: Deep Insights`와 `Level 5: Auto Pilot`이 의미하는 바를 설명하시오.**

* **해설:** * 단계: Basic Install -> Seamless Upgrades -> Full Lifecycle -> Deep Insights -> Auto Pilot.
* `Level 4 (Deep Insights)`: 메트릭, 로그, 분석을 통해 앱의 상태를 깊이 있게 모니터링하고 경고하는 단계.
* `Level 5 (Auto Pilot)`: 수집된 데이터를 바탕으로 수평/수직 스케일링, 튜닝, 비정상 동작 감지 및 자가 치유(Self-healing)를 사람 개입 없이 자동으로 수행하는 단계입니다.



---

### **Section 12: 클러스터 유지보수 및 트러블슈팅 심화**

**Q41. `etcd` 데이터베이스의 스냅샷을 백업하고 복구하는 명령어(`etcdctl`) 사용 시 주의해야 할 점과, 복구 후 API Server가 정상 동작하기 위해 확인해야 할 사항은?**

* **해설:** 백업 시 `ETCDCTL_API=3` 환경변수와 인증서 경로(--cacert, --cert, --key)를 정확히 지정해야 합니다. 복구 시에는 기존 데이터 디렉토리(`--data-dir`)를 사용하지 말고 새로운 디렉토리에 복구한 뒤, etcd Pod의 마니페스트를 수정하여 새 디렉토리를 바라보게 해야 합니다. 모든 etcd 멤버가 정상적으로 클러스터를 다시 형성했는지 확인해야 합니다.

**Q42. `kubectl drain <node-name>` 명령어를 실행했을 때, `DaemonSet` Pod와 로컬 스토리지를 사용하는 Pod에서 발생하는 에러와 이를 해결하기 위한 옵션을 설명하시오.**

* **해설:** `drain`은 기본적으로 DaemonSet Pod를 삭제할 수 없어 실패하며, 로컬 데이터를 가진 Pod를 삭제하면 데이터가 날아가므로 경고하고 멈춥니다. 이를 강제 진행하려면 `--ignore-daemonsets` (DaemonSet 무시)와 `--delete-emptydir-data` (로컬 데이터 삭제 감수) 옵션을 추가해야 합니다.

**Q43. `Static Pod`의 정의와 생성 방법, 그리고 API Server를 통해 조회는 가능하지만 제어(삭제/수정)는 불가능한 이유를 설명하시오.**

* **해설:** Static Pod는 API Server가 아닌 각 노드의 `kubelet`이 직접 관리하는 Pod입니다(보통 `/etc/kubernetes/manifests` 경로의 YAML 파일). Kubelet이 해당 파일을 감시하며 Pod를 생성하고, API Server에 Mirror Pod를 만들어 상태만 보여줍니다. 따라서 API Server에서 삭제 명령을 내려도 Kubelet이 원본 YAML 파일이 있는 한 다시 생성해버립니다.

**Q44. 클러스터 인증서(Certificates)가 만료되었을 때 갱신하는 방법(`kubeadm` 기준)과 갱신 후 필요한 조치를 설명하시오.**

* **해설:** `kubeadm certs check-expiration`으로 확인 후, `kubeadm certs renew all` 명령어로 갱신합니다. 중요한 것은 **갱신 후 Control Plane 컴포넌트(API Server, Controller Manager, Scheduler)와 Kubelet을 반드시 재시작**해야 갱신된 인증서를 로드하여 정상 통신이 가능합니다.

**Q45. `Zombie Process` (Defunct process)가 컨테이너 내부에서 다수 발생했을 때의 원인(PID 1 문제)과 이를 해결하기 위한 `tini` 같은 init 프로세스의 역할을 설명하시오.**

* **해설:** 컨테이너의 진입 프로세스(PID 1)가 자식 프로세스의 종료 시그널(SIGCHLD)을 적절히 처리(Reaping)하지 않으면 좀비 프로세스가 쌓여 PID 고갈을 유발합니다. `tini`나 `dumb-init` 같은 경량 init 프로세스를 PID 1로 실행하면, 이들이 자식 프로세스들을 관리하고 좀비 프로세스를 수거(Reap)하여 시스템 안정성을 보장합니다.

---

### **Section 13: 클라우드 네이티브 아키텍처 패턴**

**Q46. `Leader Election` 패턴이 필요한 이유와 Kubernetes 클라이언트 라이브러리(client-go)를 사용하여 이를 구현하는 원리(Lease 리소스 활용)를 간단히 설명하시오.**

* **해설:** 여러 복제본(Replica) 중 단 하나만 활성화되어야 하는 작업(예: 스케줄러, 배치 작업)을 위해 필요합니다. K8s에서는 `Lease`라는 리소스를 사용하여, 각 인스턴스가 주기적으로 갱신을 시도합니다. 갱신에 성공한 인스턴스가 리더가 되고, 실패하면 리더 권한을 잃은 것으로 간주하여 다른 인스턴스가 리더를 획득합니다.

**Q47. `Ambassador` 패턴과 `Adapter` 패턴의 차이를 Pod 내 컨테이너 구성 관점에서 설명하시오.**

* **해설:** 둘 다 멀티 컨테이너 패턴입니다.
* **Ambassador:** 외부 서비스에 연결할 때 복잡성(재시도, 인증 등)을 대신 처리해주는 프록시 역할 (예: 로컬호스트로 DB 접속 요청 -> 앰배서더가 실제 DB로 연결).
* **Adapter:** 애플리케이션의 출력(로그, 메트릭) 형식을 표준화하여 내보내는 역할 (예: 앱의 텍스트 로그 -> 어댑터가 JSON으로 변환하여 모니터링 시스템 전송).



**Q48. `Immutable Infrastructure`의 개념을 Kubernetes 배포 전략(이미지 태그 관리)과 연관 지어 설명하시오.**

* **해설:** 한번 배포된 인프라(컨테이너)는 수정하지 않고, 변경이 필요하면 완전히 새로운 것으로 교체한다는 원칙입니다. K8s에서는 `latest` 태그 사용을 지양하고, `v1.0.1`, `git-hash` 등 고유한 태그를 사용하여 배포해야 합니다. 만약 실행 중인 컨테이너에 패치(apt-get install 등)를 하면 불변성이 깨지고 일관성을 잃게 됩니다.

**Q49. `CQRS`(Command Query Responsibility Segregation) 패턴을 마이크로서비스 아키텍처에 적용할 때 발생할 수 있는 데이터 일관성 문제(Eventual Consistency)와 해결 방안을 설명하시오.**

* **해설:** 읽기(Query)와 쓰기(Command) 모델을 분리하면 성능은 좋아지지만, 쓰기 DB의 변경 사항이 읽기 DB로 전파되는 데 시차가 발생합니다. 이를 '결과적 일관성'이라 하며, 비즈니스 로직에서 이를 허용하거나, 중요한 데이터는 실시간 조회를 하도록 설계하거나, 이벤트 스트리밍(Kafka 등)의 순서를 보장하여 동기화 속도를 높여야 합니다.

**Q50. `Saga Pattern`이 분산 트랜잭션 처리에 사용되는 방식을 `Choreography`와 `Orchestration` 방식으로 나누어 비교하시오.**

* **해설:** 분산 환경에서는 2PC(Two-Phase Commit)가 비효율적입니다.
* **Choreography:** 중앙 제어 없이 각 서비스가 이벤트를 주고받으며 다음 단계를 수행합니다. 결합도가 낮지만 프로세스 추적이 어렵습니다.
* **Orchestration:** 중앙의 오케스트레이터(Saga Manager)가 각 서비스에 명령을 내리고 상태를 관리합니다. 구현이 복잡하지만 흐름 파악과 관리가 용이합니다.

---

### **Section 14: 고급 네트워크 및 CNI 심층 분석 (CNI & eBPF)**

**Q51. 기존 `iptables` 기반의 `kube-proxy` 모드와 비교하여, `IPVS` 모드가 가지는 성능상의 이점과 트래픽 라우팅 알고리즘의 차이를 설명하시오.**

* **해설:**
* **iptables:** 규칙(Rule)이 순차적(Sequential)으로 평가되므로 서비스 개수(O(n))가 늘어날수록 성능이 급격히 저하됩니다.
* **IPVS:** 커널 레벨의 해시 테이블(Hash Table)을 사용하여 서비스가 수만 개여도 O(1)에 가까운 일관된 성능을 보입니다. 또한 Round Robin 외에도 Least Connection, Locality-Based Least Connection 등 정교한 로드밸런싱 알고리즘을 지원합니다.



**Q52. Cilium과 같은 최신 CNI에서 사용하는 `eBPF`(Extended Berkeley Packet Filter) 기술이 쿠버네티스 네트워킹과 보안(Observability)에 혁신을 가져온 이유를 설명하시오.**

* **해설:** eBPF는 커널 소스 코드를 수정하거나 모듈을 로드하지 않고도 샌드박스 환경에서 프로그램을 실행할 수 있게 합니다. 이를 통해 `iptables`를 거치지 않고 패킷을 초고속으로 처리(XDP)하거나, 커널 레벨의 모든 시스템 콜과 네트워크 패킷을 가시화하여 사이드카 없이도 L7 모니터링 및 보안 정책 적용이 가능합니다.

**Q53. `MetalLB`가 온프레미스 환경에서 `LoadBalancer` 타입 서비스를 구현하는 방식을 `L2 모드`와 `BGP 모드`로 나누어 설명하시오.**

* **해설:**
* **L2 모드 (ARP):** 하나의 노드가 리더로 선출되어 LoadBalancer IP에 대한 ARP 요청에 응답합니다. 설정이 쉽지만 리더 노드에 트래픽이 몰리고, 장애 시 페일오버 시간(몇 초)이 소요됩니다.
* **BGP 모드:** 클러스터 노드들이 라우터와 BGP 피어링을 맺고 라우팅 정보를 교환합니다. ECMP(Equal-Cost Multi-Path)를 통해 진정한 로드 밸런싱과 빠른 장애 조치가 가능하지만 네트워크 장비 설정이 필요합니다.



**Q54. `Dual Stack`(IPv4/IPv6) 네트워킹을 구성할 때, Pod와 Service가 각각 IP를 할당받는 방식과 `ipFamilyPolicy` 필드의 역할(PreferDualStack 등)을 설명하시오.**

* **해설:** Dual Stack을 활성화하면 Pod는 인터페이스에 IPv4, IPv6 주소를 모두 가질 수 있습니다. `ipFamilyPolicy`는 Service가 어떤 IP를 선호할지 결정합니다.
* `SingleStack`: 기본값(IPv4).
* `PreferDualStack`: 클러스터가 지원하면 둘 다 할당, 아니면 IPv4.
* `RequireDualStack`: 반드시 둘 다 할당. 이를 통해 IPv4 주소 고갈 문제를 해결하고 레거시 시스템과의 호환성을 유지합니다.



---

### **Section 15: 멀티 클러스터 및 하이브리드 클라우드 (Multi-Cluster)**

**Q55. 멀티 클러스터 환경에서 `KubeFed`(Federation v2)의 개념과 `Host Cluster`, `Member Cluster` 간의 관계, 그리고 리소스 전파(Propagation) 방식을 설명하시오.**

* **해설:**
* **Host Cluster:** 연합(Federation) 제어 평면이 실행되는 관리 클러스터.
* **Member Cluster:** 실제 워크로드가 배포될 대상 클러스터들.
* **리소스 전파:** Host에 `FederatedDeployment` 같은 템플릿을 정의하고, `Placement` 정책(어느 클러스터에 배포할지)과 `Override` 설정(클러스터별 복제본 수 등 차이)을 통해 여러 클러스터에 설정을 동기화합니다.



**Q56. `Submariner` 또는 `Linkerd`의 멀티 클러스터 기능을 사용하여 서로 다른 클러스터(또는 VPC) 간의 Pod 통신을 연결하는 `Gateway` 방식의 아키텍처를 설명하시오.**

* **해설:** 각 클러스터에 Gateway 노드를 지정하고, 이들 간에 암호화된 터널(IPSec/WireGuard)을 뚫어 L3 네트워크를 연결합니다. 서비스 디스커버리는 `global` 도메인(예: `mysvc.default.svc.cluster-b.global`)을 사용하여 다른 클러스터의 서비스를 로컬 DNS처럼 조회하고 통신할 수 있게 합니다.

**Q57. GitOps 도구인 ArgoCD를 사용하여 멀티 클러스터를 관리할 때, `App of Apps` 패턴 또는 `ApplicationSet`을 활용하여 수백 개의 클러스터에 동일한 앱을 배포하는 전략을 설명하시오.**

* **해설:**
* **App of Apps:** 하나의 루트 앱이 다른 여러 앱(Child Apps)을 관리하는 구조.
* **ApplicationSet:** 클러스터 목록(List), Git 디렉토리 구조, 또는 라벨 셀렉터(Generator)를 기반으로 템플릿을 자동 생성합니다. 예를 들어, "AWS 리전 라벨이 있는 모든 클러스터에 Prometheus를 배포하라"는 규칙 하나로 수백 개 클러스터의 배포를 자동화하고 동기화 상태를 유지합니다.



---

### **Section 16: AI/ML 워크로드 및 GPU 최적화 (AI & Specialized Workloads)**

**Q58. Kubernetes에서 GPU 자원을 스케줄링하기 위해 `Device Plugin`이 필요한 이유와, Nvidia GPU의 `MIG`(Multi-Instance GPU) 기능을 통해 자원 활용률을 높이는 방법을 설명하시오.**

* **해설:** 기본 스케줄러는 CPU/RAM만 알지 GPU 존재를 모릅니다. `Device Plugin`이 GPU를 감지하여 '[nvidia.com/gpu]()' 같은 자원으로 등록해줍니다. MIG는 하나의 물리 GPU(A100 등)를 최대 7개의 독립된 인스턴스(메모리, 캐시 격리)로 나누어 여러 Pod가 공유하게 함으로써, 작은 모델 추론(Inference) 시 GPU 낭비를 줄입니다.

**Q59. AI 학습(Training) 작업과 같이 배치 성격이 강한 워크로드를 처리할 때, 기본 스케줄러 대신 `Volcano`나 `Kueue`와 같은 배치 스케줄러를 사용하는 이유(Gang Scheduling)를 설명하시오.**

* **해설:** 분산 학습(Distributed Training)은 워커 Pod 10개 중 1개만 실행되지 않아도 전체 작업이 돕니다(All-or-Nothing). 기본 스케줄러는 순차적으로 자원을 점유하므로 교착 상태(Deadlock)가 발생할 수 있습니다. `Gang Scheduling`은 필요한 모든 자원이 확보될 때까지 대기했다가, 확보되면 모든 Pod를 동시에 실행하여 이를 방지합니다.

**Q60. `Kubeflow` 파이프라인(KFP)이 AI 모델의 생명주기(데이터 전처리 -> 학습 -> 평가 -> 배포)를 관리하는 방식과, 이를 통해 실험의 재현성(Reproducibility)을 보장하는 원리를 서술하시오.**

* **해설:** 각 단계를 컨테이너 기반의 컴포넌트로 정의하고, 이를 DAG(Directed Acyclic Graph)로 연결하여 워크플로우를 구성합니다. 모든 단계의 입력값(파라미터), 코드 버전, 출력 데이터(Artifact)가 메타데이터 스토어에 버전 관리되므로, 언제든지 과거의 실험 결과를 동일한 조건에서 재생산하거나 추적할 수 있습니다.

---

### **Section 17: 공급망 보안 및 규정 준수 (Supply Chain Security)**

**Q61. 소프트웨어 공급망 보안을 위해 `Sigstore`의 `Cosign`을 사용하여 컨테이너 이미지를 서명(Signing)하고 검증(Verification)하는 전체 흐름을 설명하시오.**

* **해설:** 개발자는 개인키로 이미지를 서명하고 서명 데이터를 OCI 레지스트리에 함께 업로드합니다. 배포 시 Kubernetes의 Admission Controller(예: Kyverno, Policy Controller)가 공개키를 사용하여 해당 이미지의 서명이 유효한지 검증합니다. 서명이 없거나 위변조된 이미지는 배포를 차단하여 신뢰할 수 있는 코드만 실행되도록 보장합니다.

**Q62. `SBOM`(Software Bill of Materials)의 개념과, 이를 생성하는 도구(Syft 등)를 사용하여 취약점(Log4j 등) 발생 시 신속하게 영향도를 파악하는 방법을 설명하시오.**

* **해설:** SBOM은 소프트웨어 구성 요소(라이브러리 버전, 패키지 정보 등) 명세서입니다. CI 파이프라인에서 이미지를 빌드할 때 SBOM을 생성해 둡니다. 제로데이 취약점이 발표되면, 보안팀은 전체 이미지를 스캔할 필요 없이 중앙 저장된 SBOM 데이터베이스만 쿼리하여 취약한 라이브러리를 포함한 애플리케이션을 즉시 식별할 수 있습니다.

**Q63. 런타임 보안 도구인 `Falco`가 시스템 콜을 분석하여 비정상 행위(예: 쉘 실행, 민감 파일 접근)를 탐지하는 메커니즘과, 이를 알림으로 연동하는 방법을 설명하시오.**

* **해설:** Falco는 커널 모듈이나 eBPF 프로브를 통해 실시간 시스템 콜 스트림을 수집합니다. 정의된 규칙(예: "웹 서버 컨테이너에서 `bash` 실행 금지")과 패턴이 일치하면 이벤트를 발생시킵니다. 이 이벤트는 `Falco Sidekick`을 통해 Slack, Elasticsearch, AWS Lambda 등으로 전달되어 즉각적인 대응을 가능하게 합니다.

---

### **Section 18: 서버리스 및 오토스케일링 심화 (Serverless & Scaling)**

**Q64. `KEDA`(Kubernetes Event-driven Autoscaling)가 HPA(CPU/Mem 기반)의 한계를 극복하고, Kafka Lag이나 RabbitMQ 큐 길이를 기반으로 스케일링을 수행하는 원리를 설명하시오.**

* **해설:** HPA는 기본적으로 리소스 사용량만 봅니다. KEDA는 `Scaler`라는 인터페이스를 통해 외부 이벤트 소스(Kafka, AWS SQS 등)의 메트릭(예: 처리되지 않은 메시지 수)을 조회합니다. 이 값을 HPA가 이해할 수 있는 Custom Metric으로 변환하여 제공함으로써, CPU 부하가 오기 전에 큐에 쌓인 작업량에 따라 선제적으로 Pod를 늘립니다.

**Q65. `Knative Serving`의 `Scale-to-Zero` 기능이 리소스 비용 절감에 기여하는 방식과, '0'에서 '1'로 스케일 업 될 때 발생하는 `Cold Start` 지연 문제를 완화하기 위한 방안을 설명하시오.**

* **해설:** 요청이 없으면 Pod를 0개로 줄여 유휴 자원 비용을 없앱니다. 요청이 들어오면 `Activator`가 요청을 잠시 버퍼링하고 Pod를 띄운 뒤 트래픽을 보냅니다. Cold Start 완화를 위해 `minScale: 1`을 설정하거나, 런타임이 가벼운 언어(Go, Rust) 사용, 또는 GraalVM을 이용한 네이티브 이미지 빌드를 고려해야 합니다.

**Q66. AWS 환경에서 `Karpenter`가 기존의 `Cluster Autoscaler` 대비 노드 프로비저닝 속도가 빠르고 비용 효율적인 이유를 `Provisioner`(NodePool) 개념과 연관 지어 설명하시오.**

* **해설:** Cluster Autoscaler는 Node Group(ASG)의 크기만 조정하므로 반응이 느리고 인스턴스 타입이 고정적입니다. Karpenter는 Pending Pod의 리소스 요구사항(CPU, GPU 등)을 직접 분석하여, 해당 시점에 가장 저렴하고 적합한 인스턴스 타입(Spot 포함)을 계산해 EC2 Fleet API로 즉시 노드를 생성(JIT)합니다.

---

### **Section 19: 데이터 관리 및 백업/복구 (Data & DR)**

**Q67. `Velero`를 사용하여 클러스터 전체 백업을 수행할 때, etcd 스냅샷과의 차이점과 `Restic`(또는 Kopia) 통합을 통해 PV(Persistent Volume) 데이터까지 백업하는 방식을 설명하시오.**

* **해설:** etcd 스냅샷은 클러스터 설정만 저장하며 볼륨 데이터는 포함하지 않습니다. Velero는 K8s API 리소스(YAML)를 백업함과 동시에, 스토리지 스냅샷 API를 호출하거나 `Restic`을 사용하여 파일 시스템 레벨에서 PV의 실제 데이터를 오브젝트 스토리지(S3 등)로 복사합니다. 이를 통해 설정과 데이터를 포함한 완전한 복구가 가능합니다.

**Q68. 데이터베이스 운영 시 `Operator` 패턴(예: Postgres Operator)이 `StatefulSet` 단독 사용보다 고가용성(HA) 구성, 백업, 버전 업그레이드 관점에서 유리한 구체적인 이유를 설명하시오.**

* **해설:** StatefulSet은 순서와 스토리지만 보장할 뿐 DB 내부 로직은 모릅니다. Operator는 DB 전용 지식을 코드로 구현하여, 장애 시 리더 선출(Failover), 정기적인 WAL(Write-Ahead Logging) 백업 및 시점 복구(PITR), 마이너 버전 롤링 업데이트 시 데이터 마이그레이션 등을 자동으로 수행합니다.

**Q69. `Ceph` 기반의 `Rook` 스토리지를 사용할 때, `Block Storage`(RBD), `File System`(CephFS), `Object Storage`(RGW) 모드가 각각 어떤 워크로드에 적합한지 Kubernetes 관점에서 설명하시오.**

* **해설:**
* **RBD (Block):** `ReadWriteOnce`. 고성능이 필요한 DB나 단일 Pod 전용 볼륨.
* **CephFS (File):** `ReadWriteMany`. 여러 Pod가 공유해야 하는 웹 서버 컨텐츠, CI/CD 아티팩트 공유 폴더.
* **RGW (Object):** S3 호환 API. 백업 데이터 저장, 로그 아카이빙, 정적 웹 호스팅 등 대용량 비정형 데이터.



---

### **Section 20: 클러스터 업그레이드 및 마이그레이션 Strategy**

**Q70. `In-place Upgrade`(기존 노드 업그레이드)와 `Blue/Green Cluster Upgrade`(신규 클러스터 교체) 전략의 장단점을 운영 리스크와 롤백 용이성 측면에서 비교하시오.**

* **해설:**
* **In-place:** 비용이 저렴하고 설정 유지가 쉽지만, 업그레이드 실패 시 복구가 매우 어렵고 노드별 순차 진행으로 시간이 오래 걸립니다.
* **Blue/Green:** 신규 버전 클러스터를 옆에 구축하고 트래픽만 전환(DNS/GSLB)합니다. 비용은 2배 들지만, 문제 발생 시 즉시 구 버전 클러스터로 트래픽을 돌려(Switch back) 롤백할 수 있어 리스크가 가장 낮습니다.



**Q71. API 버전 변경(Deprecation)으로 인해 제거된 API(예: `v1beta1`)를 사용하는 매니페스트를 찾고, `kubent`(Kube No Trouble)나 `pluto` 같은 도구를 사용하여 사전에 대응하는 방법을 설명하시오.**

* **해설:** 제거된 API를 호출하면 배포가 실패합니다. `kubent`는 클러스터 내의 실행 중인 리소스와 Helm 릴리즈, Git 리포지토리를 스캔하여, 다음 K8s 버전에서 제거될 API를 사용 중인 리소스를 찾아내고 대체할 정식 버전(`v1`)을 알려주어 업그레이드 전 사전 조치를 가능하게 합니다.

---

### **Section 21: 테넌트 격리 및 비용 관리 (FinOps)**

**Q72. 멀티 테넌트 환경에서 `Namespace` 격리만으로 부족하여 `vCluster`(Virtual Cluster)를 도입해야 하는 상황은 언제이며, 이것이 Control Plane 격리를 제공하는 방식을 설명하시오.**

* **해설:** 네임스페이스는 CRD나 클러스터 전역 리소스(Node, PV 등)를 공유하므로 완전한 권한 분리가 어렵습니다. `vCluster`는 실제 클러스터의 네임스페이스 안에 가상의 API Server와 Controller Manager를 띄워, 테넌트에게 마치 전용 클러스터 전체를 가진 것 같은 환경(Admin 권한 부여 가능)을 제공하면서도 실제 리소스는 호스트 클러스터에서 관리합니다.

**Q73. `OpenCost` 또는 `Kubecost`를 활용하여 네임스페이스, 라벨, 서비스별로 정확한 비용을 산정(Showback/Chargeback)하는 원리와, 공유 리소스(Shared GPU/Storage) 비용 배분 방식을 설명하시오.**

* **해설:** 클라우드 청구서 데이터와 K8s 리소스 사용량 메트릭을 결합하여 비용을 계산합니다. 공유 리소스(예: 모니터링 네임스페이스)나 유휴 자원(Idle) 비용은 각 팀의 실제 사용량 비율(CPU/RAM Request)에 따라 비례 배분(Proportional Distribution)하여 실제 사용자가 부담해야 할 비용을 투명하게 리포팅합니다.

---

### **Section 22: 차세대 기술 및 트렌드 (Emerging Tech)**

**Q81. `WebAssembly(WASM)`가 컨테이너 기술의 대안 혹은 보완재로 주목받는 이유를 `Cold Start` 속도와 `Sandboxing` 보안 관점에서 설명하시오.**

* **해설:** WASM 모듈은 수 밀리초(ms) 단위의 극도로 빠른 부팅 속도를 제공하여 서버리스 환경(Scale-to-Zero)에 최적화되어 있습니다. 또한, OS 레벨의 가상화가 아닌 메모리 격리 기반의 샌드박스 모델을 사용하여, 컨테이너보다 훨씬 가볍고 안전한 격리 환경을 제공합니다. 이는 엣지 컴퓨팅이나 고밀도 멀티 테넌트 환경에서 유리합니다.

**Q82. Kubernetes의 `Gateway API`가 기존의 `Ingress` 리소스를 대체하려는 이유와, `GatewayClass`, `Gateway`, `HTTPRoute`로 리소스를 분리함으로써 얻을 수 있는 조직적 이점(Role-oriented)을 설명하시오.**

* **해설:** Ingress는 기능이 단순하여 벤더별로 주석(Annotation)에 의존해야 했습니다. Gateway API는 이를 표준화하고 확장성을 높였습니다.
* **GatewayClass:** 인프라팀이 로드밸런서 종류(L4/L7)를 정의.
* **Gateway:** 클러스터 운영자가 리스너(포트, 프로토콜)를 설정.
* **HTTPRoute:** 개발자가 실제 라우팅 규칙(경로, 헤더)을 정의.
* 이처럼 역할을 분리하여 인프라 팀과 개발 팀이 서로 간섭 없이 독립적으로 네트워크 설정을 관리할 수 있습니다.



**Q83. `Istio Ambient Mesh`나 `Cilium Service Mesh`와 같은 `Sidecar-less` 아키텍처가 등장한 배경과, 이것이 기존 사이드카 패턴의 성능 오버헤드 및 유지보수 문제를 어떻게 해결하는지 설명하시오.**

* **해설:** 기존 사이드카 패턴은 모든 Pod에 프록시 컨테이너를 주입해야 하므로 리소스 소모가 크고, 프록시 업그레이드 시 앱 재시작이 필요했습니다. Sidecar-less 방식은 노드 당 하나의 공유 프록시(Ztunnel 등)나 커널 레벨(eBPF)에서 L4/L7 처리를 수행하여, 애플리케이션 변경 없이 메쉬 기능을 적용하고 리소스 사용량을 획기적으로 줄입니다.

---

### **Section 23: 플랫폼 엔지니어링 (Platform Engineering)**

**Q84. `Backstage`와 같은 IDP(Internal Developer Platform)를 구축하여 Kubernetes 복잡성을 추상화할 때, `Software Templates`와 `Catalog` 기능이 개발자 생산성에 미치는 영향을 서술하시오.**

* **해설:** 개발자가 직접 Helm 차트나 K8s YAML을 작성하는 대신, 검증된 템플릿(Golden Path)을 통해 클릭 몇 번으로 CI/CD, 모니터링, 보안 설정이 완비된 마이크로서비스를 생성할 수 있습니다. Catalog는 분산된 서비스들의 소유권, API 문서, 종속성을 한곳에서 시각화하여 서비스 식별과 관리를 용이하게 합니다.

**Q85. `Crossplane`을 사용하여 AWS RDS나 S3와 같은 외부 클라우드 자원을 Kubernetes 매니페스트(`Kind: RDSInstance`)로 관리하는 방식(Infrastructure as Data)의 장점과 `Compositions`의 역할을 설명하시오.**

* **해설:** 테라폼(Terraform)과 달리 Crossplane은 K8s 컨트롤 루프를 사용하여 외부 자원의 상태를 지속적으로 감시하고 조정(Reconcile)합니다. `Composition`은 복잡한 클라우드 리소스 설정(VPC, Subnet, SG, RDS)을 하나의 추상화된 CRD(예: `PostgresDB`)로 묶어 개발자에게 제공함으로써, 개발자가 인프라 세부 사항을 몰라도 자원을 요청할 수 있게 합니다.

---

### **Section 24: 실전 장애 처리 시나리오 (War Room Scenarios)**

**Q86. 특정 노드에 배포된 Pod들이 외부 통신 시 간헐적으로 연결 실패(Timeout)를 겪고 있다. 원인이 `SNAT Port Exhaustion`(Source NAT 포트 고갈)일 때, 이를 확인하는 방법과 해결책을 설명하시오.**

* **해설:**
* **확인:** 해당 노드에서 `conntrack -L` 또는 커널 로그(`dmesg`)에서 `nf_conntrack: table full` 에러를 확인합니다. 모니터링 시스템에서 노드의 아웃바운드 연결 수를 확인합니다.
* **해결:** `AWS NAT Gateway` 등을 사용하여 공인 IP를 늘리거나, Pod가 호스트 네트워크를 타지 않고 전용 CNI IP를 사용하도록 설정합니다. 또는 `sysctl`로 `net.netfilter.nf_conntrack_max` 값을 늘려주지만, 근본적으로는 연결 풀링(Connection Pooling)을 점검해야 합니다.



**Q87. Java 애플리케이션 Pod가 `OOMKilled` 되었으나, 모니터링 도구에서는 메모리 사용량이 `Limit`에 도달하지 않은 것으로 보인다. `Container Memory`와 `JVM Heap`의 괴리, 그리고 `Native Memory` 누수 가능성을 설명하시오.**

* **해설:** 컨테이너 메모리 제한(Limit)은 JVM 힙(Heap)뿐만 아니라 Metaspace, Thread Stack, Direct Buffer, GC 오버헤드 등 네이티브 메모리 영역을 모두 포함합니다. 힙 메모리가 여유 있어도 네이티브 메모리(예: Netty의 Direct Buffer)가 누수되거나 과도하게 사용되면 전체 컨테이너 사용량이 Limit을 초과하여 커널에 의해 강제 종료됩니다.

**Q88. Ingress를 통해 서비스되는 웹 애플리케이션에서 간헐적으로 `502 Bad Gateway` 오류가 발생한다. 백엔드 Pod는 정상일 때, Ingress Controller와 애플리케이션 간의 `Keep-Alive Timeout` 불일치 문제를 설명하시오.**

* **해설:** Ingress(Nginx 등)가 백엔드 Pod와의 연결을 유지하려는데(Keep-Alive), 애플리케이션(Node.js, Spring Boot)의 타임아웃 설정이 더 짧아서 연결을 먼저 끊어버리는 경우(Race Condition) 발생합니다. Ingress가 끊긴 소켓으로 요청을 보내면 502 에러가 납니다. **Ingress의 타임아웃보다 애플리케이션의 타임아웃을 항상 길게 설정**해야 합니다.

**Q89. `CrashLoopBackOff` 상태인 Pod의 로그를 보려 했으나 `logs` 명령어도 실패한다. 이때 `ephemeral-containers` (kubectl debug) 기능을 사용하여 실행 중인 Pod에 디버깅용 컨테이너를 붙여 문제를 진단하는 과정을 설명하시오.**

* **해설:** Pod의 메인 컨테이너가 시작하자마자 죽으면 쉘 접속(`exec`)이 불가능합니다. `kubectl debug -it <pod-name> --image=busybox --target=<container-name>` 명령어를 사용하면, 기존 Pod의 프로세스 네임스페이스를 공유하는 임시 컨테이너를 주입할 수 있습니다. 이를 통해 파일 시스템을 탐색하거나 프로세스 상태를 확인하여 원인을 파악합니다.

---

### **Section 25: 아키텍트 관점의 설계 및 의사결정 (Architecture Decision)**

**Q90. 조직의 멀티 테넌시(Multi-tenancy) 전략을 수립할 때, `Soft Isolation`(Namespace 격리)과 `Hard Isolation`(별도 클러스터 또는 vCluster)을 선택하는 기준을 보안, 비용, 운영 복잡성 측면에서 비교하시오.**

* **해설:**
* **Soft Isolation:** 비용 효율적이고 관리가 쉽지만, 커널 공유로 인한 보안 취약점과 `Neighbour Noise`(자원 경합) 문제가 있습니다. (내부 팀 간 공유에 적합).
* **Hard Isolation:** 물리적/논리적으로 완벽히 격리되어 보안과 성능 보장이 확실하지만, 클러스터가 늘어날수록 관리 비용(Control Plane 비용, 업그레이드 공수)이 급증합니다. (규제 준수가 필요한 금융/의료 데이터나 외부 고객용 서비스에 적합).



**Q91. 재해 복구(DR) 전략에서 `Active-Passive`와 `Active-Active` 구성을 Kubernetes로 구현할 때, 가장 큰 걸림돌인 `Stateful Workload`(데이터 동기화) 문제를 해결하기 위한 스토리지 복제 기술(Async vs Sync)과 RPO/RTO 트레이드오프를 설명하시오.**

* **해설:**
* **Active-Active:** 양쪽 클러스터에서 동시에 쓰기 작업을 하려면 양방향 동기화가 필요한데, 이는 레이턴시와 충돌 해결이 매우 어렵습니다. (보통 Stateless 앱만 Active-Active로 구성).
* **Storage Replication:** `Async`(비동기)는 성능 영향이 적지만 데이터 유실 가능성(RPO > 0)이 있고, `Sync`(동기)는 데이터 정합성은 보장(RPO=0)되지만 거리(Distance)에 따른 성능 저하가 큽니다. 따라서 DR은 보통 비동기 복제를 사용하는 Active-Passive 구성을 기본으로 합니다.



**Q92. 비용 최적화(FinOps)를 위해 `Spot Instance`를 운영 환경(Production)에 도입하려 한다. 이때 워크로드의 종류(Stateless/Stateful)에 따른 적합성 여부와, `Karpenter`의 `consolidation` 기능을 활용한 비용 절감 전략을 설명하시오.**

* **해설:** Spot 인스턴스는 언제든 회수될 수 있으므로, 종료 시 안전하게 재시작 가능한 Stateless 웹 서버나 배치 작업에 적합합니다. DB 같은 Stateful 앱에는 부적합합니다. Karpenter의 `consolidation`은 빈 공간이 많은 노드의 Pod를 다른 노드로 옮기고 비싼 노드를 종료하거나, 더 저렴한 인스턴스 타입으로 실시간 교체하여 클러스터 비용을 최소화합니다.

---

### **Section 26: 보안 및 거버넌스 심화 (Advanced Governance)**

**Q93. `External Secrets Operator`(ESO)를 사용하여 AWS Secrets Manager나 HashiCorp Vault와 Kubernetes를 연동하는 아키텍처와, 이를 통해 GitOps 환경에서 `Secret Zero` 문제를 해결하는 방법을 설명하시오.**

* **해설:** GitOps 리포지토리에는 암호화되지 않은 Secret을 절대 올리면 안 됩니다. ESO는 K8s 내에 `ExternalSecret` CRD(참조 정보만 포함)를 생성하면, 실제 값은 외부 Vault에서 가져와 K8s Secret으로 자동 변환해 줍니다. 이를 통해 Git에는 민감 정보 없이 참조값만 저장하고, 실제 비밀 정보는 보안 전용 저장소에서 중앙 관리할 수 있습니다.

**Q94. Kubernetes 클러스터의 규정 준수(PCI-DSS, HIPAA)를 위해 `OPA Gatekeeper`나 `Kyverno`를 활용하여 `Policy-as-Code`를 구현하는 예시(예: LoadBalancer 생성 금지, 특정 레지스트리만 허용)를 드시오.**

* **해설:**
* "모든 이미지는 사내 레지스트리(harbor.internal)에서만 가져와야 한다."
* "Public LoadBalancer 서비스 생성을 차단한다."
* "Pod는 반드시 리소스 Limit을 설정해야 한다."
* 이러한 정책을 코드로 작성하여 배포 시점에 강제함으로써, 개발자의 실수나 보안 위규를 원천 차단하고 감사(Audit) 기록을 남깁니다.



---

### **Section 27: 네트워크 및 연결성 심화 (Advanced Connectivity)**

**Q95. IPv4 주소 고갈 문제를 해결하기 위해 `IPv6 Single Stack` 클러스터로 전환하려 할 때, 외부의 IPv4 클라이언트가 접근할 수 있도록 하는 `NAT64/DNS64` 메커니즘을 설명하시오.**

* **해설:** 클러스터 내부는 IPv6로만 통신하더라도, 외부 레거시 시스템(IPv4)과의 통신은 필요합니다. `NAT64` 게이트웨이는 IPv6 패킷을 IPv4로 변환해 줍니다. `DNS64`는 IPv4 전용 도메인을 조회했을 때, 가상의 IPv6 주소(NAT64 프리픽스 포함)를 반환하여 IPv6-only 클라이언트가 IPv4 서버와 통신할 수 있게 해 줍니다.

**Q96. `Ingress Controller`의 단일 장애점(SPOF) 문제를 해결하기 위해, DNS 로드밸런싱(GSLB)이나 클라우드 로드밸런서(NLB)와 결합하여 고가용성(HA)을 구성하는 아키텍처를 설명하시오.**

* **해설:** Ingress Controller Pod가 죽으면 전체 서비스가 중단될 수 있습니다. 이를 방지하기 위해 Ingress Controller를 DaemonSet으로 띄우거나 `replicas`를 늘리고, 그 앞단에 Cloud NLB를 두어 트래픽을 여러 노드의 Ingress로 분산합니다. 리전 장애 대비를 위해서는 GSLB를 최상단에 두어 헬스 체크 실패 시 다른 리전의 NLB로 트래픽을 우회시킵니다.

---

### **Section 28: Day 2 운영 및 유지보수 (Long-term Ops)**

**Q97. etcd 데이터베이스의 크기가 계속 증가하여 성능 저하가 발생할 때, `Defrag` 작업이 필요한 이유와 이를 무중단으로 수행하기 위한 절차를 설명하시오.**

* **해설:** etcd는 MVCC(Multi-Version Concurrency Control)를 사용하므로 키를 삭제하거나 수정해도 이전 버전이 공간을 차지합니다(Compaction 후 빈 공간 발생). 이 단편화(Fragmentation)를 제거하여 디스크 공간을 회수하려면 `defrag`가 필요합니다. 리더 노드에서 수행 시 잠시 멈춤 현상이 발생할 수 있으므로, 팔로워 노드들부터 순차적으로 수행하고 마지막에 리더를 넘기고(Move Leader) 수행하는 것이 안전합니다.

**Q98. Kubernetes 버전 업그레이드 시 `Drain` 명령어를 사용하여 노드를 비울 때, `PodDisruptionBudget`(PDB)이 설정된 애플리케이션이 업그레이드 속도에 미치는 영향과 운영자의 대응 방안을 설명하시오.**

* **해설:** PDB가 `minAvailable: 90%` 등으로 타이트하게 잡혀있다면, Drain 시 PDB 조건을 만족할 때까지 Pod가 축출되지 않아 업그레이드가 무한 대기할 수 있습니다. 운영자는 업그레이드 창(Maintenance Window) 동안 일시적으로 PDB를 완화하거나, 배포 전략을 점검하여 업그레이드가 진행될 수 있도록 해야 합니다.

---

### **Section 29: 엔지니어링 철학 및 소프트 스킬 (Soft Skills & Philosophy)**

**Q99. "Kubernetes가 과연 필요한가?"라는 질문에 대해, 모놀리식(Monolithic) 아키텍처와 단순 EC2 배포가 더 적합한 경우(Over-engineering 방지)를 비용과 운영 복잡성 관점에서 논하시오.**

* **해설:** 초기 스타트업이나 트래픽이 적고 변경이 드문 서비스, 혹은 개발팀 규모가 작아 K8s 운영 인력이 없는 경우에는 K8s 도입이 오히려 독이 됩니다(배보다 배꼽이 더 큼). 단순함이 최고의 미덕일 때는 PaaS(Heroku, Vercel)나 단순 VM 배포가 비용 효율적이고 개발 속도도 빠릅니다. K8s는 **복잡성을 제어할 필요가 있을 만큼 규모가 커졌을 때** 도입해야 합니다.

**Q100. 시니어 엔지니어로서 팀원들에게 `Cattle vs Pets` (가축 대 애완동물) 개념을 교육하고, 이를 통해 `Immutable Infrastructure` 문화를 정착시키기 위한 구체적인 실천 방안(SSH 접속 금지 등)을 제시하시오.**

* **해설:** 서버를 애지중지하며 이름 붙이고 수동으로 관리(Pets)하는 것이 아니라, 언제든 폐기하고 새로 띄울 수 있는 소모품(Cattle)으로 취급해야 합니다.
* **실천 방안:** 운영 환경에 대한 SSH 접속 권한을 회수합니다. 모든 변경은 코드(IaC, Dockerfile)로만 이루어지게 하고, 문제 발생 시 서버를 고치려 하지 말고(Patch) 새 버전으로 재배포(Replace)하도록 가이드합니다. 로그와 메트릭은 외부 시스템으로 전송하여 서버 접속 없이도 디버깅 가능하게 합니다.


---

### **Bonus Track: Extreme Edge Cases & Niche Tech (101~110)**

**Q101. Kubernetes 클러스터에 Linux 노드와 Windows 노드가 섞여 있는 `Hybrid OS` 환경을 구성했다. 이때 Linux 전용 DaemonSet(예: Node Exporter)이 Windows 노드에 뜨지 않게 하고, 반대로 Windows 앱은 Windows 노드에만 뜨게 강제하기 위한 `RuntimeClass`와 `NodeSelector` 설정 전략을 설명하시오.**

* **해설:**
* OS가 섞여 있으면 스케줄러가 OS를 구분하지 못해 에러가 발생합니다.
* **DaemonSet 제어:** Linux용 DaemonSet에는 `nodeSelector: {kubernetes.io/os: linux}`를 명시하여 Windows 노드를 배제해야 합니다.
* **Windows 앱 제어:** 단순히 `nodeSelector`만 쓰는 것보다, `RuntimeClass`를 정의하여 Windows 컨테이너 런타임 설정을 캡슐화하거나, `Taint`(`os=windows:NoSchedule`)를 Windows 노드에 걸고 앱에 `Toleration`을 주는 방식이 더 안전합니다.



**Q102. `API Aggregation Layer`의 개념을 설명하고, `Metrics Server`나 `Prometheus Adapter`가 이 계층을 통해 커스텀 API(`metrics.k8s.io`)를 확장하는 원리를 서술하시오.**

* **해설:**
* Kubernetes API 서버는 모든 로직을 혼자 처리하지 않고, 특정 API 그룹(예: `/apis/custom.metrics.k8s.io`)에 대한 요청을 외부의 별도 서비스(Extension API Server)로 위임(Proxy)할 수 있습니다.
* 이를 Aggregation Layer라고 하며, Metrics Server는 이 기능을 이용해 API 서버에 합쳐진(Aggregated) 형태로 등록됩니다. 따라서 사용자가 `kubectl get --raw /apis/metrics.k8s.io/...`를 호출하면, 실제 처리는 Metrics Server Pod가 수행하고 결과만 API 서버를 통해 반환됩니다.



**Q103. `Chaos Engineering` 도구인 `Chaos Mesh`나 `Litmus`를 도입하여 프로덕션 환경에서 카오스 테스트를 수행하려 한다. 이때 `Blast Radius`(폭발 반경)를 제어하기 위해 네임스페이스와 라벨 셀렉터를 어떻게 활용해야 하며, 긴급 중단(Kill Switch)은 어떻게 확보하는가?**

* **해설:**
* 프로덕션 카오스 테스트는 매우 위험하므로, 실험 대상을 명확히 격리해야 합니다. Chaos Mesh의 실험 정의(YAML)에서 `namespace`와 `labelSelectors`를 엄격하게 지정하여 특정 카나리 배포 버전이나 테스트용 Pod만 타겟팅합니다.
* **Kill Switch:** 모든 카오스 실험은 CRD로 관리되므로, 장애가 예상 범위를 벗어나면 즉시 해당 실험 CR을 `delete`하거나, 카오스 컨트롤러 매니저의 `pause` 기능을 통해 모든 주입을 즉각 중단할 수 있는 프로세스를 마련해야 합니다.



**Q104. CI/CD 파이프라인에서 `Multi-Architecture` 이미지(AMD64 + ARM64)를 빌드하여 GKE(x86)와 AWS Graviton(ARM) 노드 모두에서 실행되게 하려 한다. Docker의 `Buildx`와 `Manifest List`가 작동하는 원리를 설명하시오.**

* **해설:**
* `docker buildx`는 QEMU 에뮬레이터를 사용하여 하나의 빌드 머신에서 여러 아키텍처용 바이너리를 컴파일합니다.
* 빌드가 완료되면 각 아키텍처별 이미지(Layer)와 이를 묶는 `Manifest List`(Index)가 레지스트리에 업로드됩니다.
* K8s 런타임(containerd)이 이미지를 당길(Pull) 때, 노드의 아키텍처(예: ARM64)에 맞는 해시값을 Manifest List에서 찾아 자동으로 해당 레이어만 다운로드합니다.



**Q105. `OIDC`(OpenID Connect)를 사용하여 Kubernetes 인증을 Google이나 Okta 계정과 연동했다. 사용자가 `kubectl`을 칠 때 `id_token`이 전달되는 흐름과, API 서버가 이 토큰의 유효성을 검증(서명 확인)하는 과정을 설명하시오.**

* **해설:**
* 사용자는 로그인(`kubelogin` 등) 후 IDP로부터 `id_token`(JWT)을 발급받아 kubeconfig에 저장합니다.
* `kubectl` 요청 시 헤더에 이 토큰을 실어 보냅니다.
* API 서버는 IDP와 통신하지 않고(Stateless), 사전에 설정된 `--oidc-issuer-url`의 공개키(JWKS)를 사용하여 JWT의 서명이 올바른지, 만료되지 않았는지(`exp`), 발급자(`iss`)가 맞는지 로컬에서 검증하여 인증을 처리합니다.



**Q106. `Dynamic Admission Control`에서 `MutatingWebhook`과 `ValidatingWebhook`의 실행 순서가 중요한 이유와, 만약 Mutating 웹훅이 무한 루프(재귀 호출)를 일으키지 않게 하기 위한 방어 기제(ReinvocationPolicy)를 설명하시오.**

* **해설:**
* **순서:** Mutating(변경)이 먼저 실행되어 객체를 수정(예: 사이드카 주입)한 후, Validating(검증)이 실행되어 최종 상태를 확인합니다. 순서가 바뀌면 검증 후 내용이 바뀌어 위험할 수 있습니다.
* **무한 루프 방지:** Mutating 웹훅이 객체를 수정하면, 수정된 객체에 대해 다시 웹훅이 트리거될 수 있습니다. 이를 방지하기 위해 `reinvocationPolicy: IfNeeded`를 신중히 설정하거나, 웹훅 로직 내에서 "이미 주입된 경우 건너뜀(Idempotent)" 처리를 반드시 구현해야 합니다.



**Q107. 대규모 클러스터에서 `DNS Throttling` 문제가 발생하여 `NodeLocal DNSCache`를 도입하려 한다. 이것이 iptables 규칙을 어떻게 우회하여 성능을 향상시키는지, 그리고 `force_tcp` 설정이 필요한 상황은 언제인지 설명하시오.**

* **해설:**
* 기본 CoreDNS로 가는 트래픽은 iptables(Conntrack)를 거치며 병목이 됩니다. `NodeLocal DNSCache`는 각 노드에 DNS 캐시 Pod를 DaemonSet으로 띄우고, 가상 IP를 통해 iptables를 거치지 않고 로컬 소켓으로 직접 통신하게 하여 레이턴시를 획기적으로 줄입니다.
* **force_tcp:** UDP 패킷 드롭이 심한 불안정한 네트워크 환경이거나, DNS 응답이 커서 단편화(Fragmentation)가 자주 발생할 때 TCP 연결을 강제하여 신뢰성을 높이기 위해 사용합니다.



**Q108. `Pod Lifecycle`에서 `PreStop Hook`을 사용하여 `Graceful Shutdown`을 구현할 때, 애플리케이션의 종료 로직(SIGTERM 처리)과 서비스 엔드포인트 제거 사이의 타이밍 이슈(Race Condition)를 해결하는 구체적인 설정을 설명하시오.**

* **해설:**
* Pod가 Terminating 되면 K8s는 (1) 엔드포인트 제거와 (2) SIGTERM 전송을 동시에 수행합니다. 이때 앱이 너무 빨리 죽으면, iptables가 갱신되기 전에 들어온 트래픽이 유실됩니다.
* **해결:** `PreStop Hook`에 `sleep 5` 같은 명령어를 넣어, 앱이 SIGTERM을 받기 전에 잠시 대기하게 합니다. 이 시간 동안 로드밸런서와 iptables에서 IP가 완전히 제거될 여유를 주고, 그 후 앱이 남은 요청을 처리하고 종료하도록 합니다.



**Q109. `Headless Service`를 사용할 때 DNS 조회 결과가 일반 서비스와 어떻게 다른지 설명하고, 이를 활용하여 `gRPC Load Balancing`이나 `Custom Sharding` 로직을 클라이언트 사이드에서 구현하는 방법을 서술하시오.**

* **해설:**
* 일반 서비스는 ClusterIP(VIP) 하나만 반환하지만, Headless Service(`ClusterIP: None`)는 연결된 **모든 Pod의 IP 목록(A 레코드 리스트)**을 직접 반환합니다.
* **Client-side LB:** gRPC 클라이언트 등은 이 IP 목록을 받아, 중간 로드밸런서 없이 직접 각 Pod와 연결(Channel)을 맺고 Round Robin이나 자체 알고리즘으로 트래픽을 분산 처리할 수 있습니다.



**Q110. `eBPF` 기반의 모니터링 도구인 `Pixie`나 `Hubble`을 사용할 때, 애플리케이션 코드 수정(Instrumentation) 없이 어떻게 HTTPS 암호화 트래픽의 내부(Payload)를 들여다볼 수 있는지 설명하시오.**

* **해설:**
* 기존에는 SSL/TLS가 적용되면 패킷 스니핑으로 내용을 볼 수 없었습니다.
* eBPF는 커널 레벨에서 `OpenSSL`이나 `Go crypto/tls` 라이브러리의 시스템 콜(예: `SSL_write`, `SSL_read`)을 후킹(kprobe/uprobe)합니다. 즉, 데이터가 암호화되기 직전(Plaintext)이나 복호화된 직후의 메모리 데이터를 가로채서 분석하므로, 인증서 키 없이도 가시성을 확보할 수 있습니다.



---

### **Extra Section: etcd Deep Dive (Internal & Tuning)**

**Q111. `etcd`는 MVCC(Multi-Version Concurrency Control) 모델을 사용한다. 키를 삭제(`DELETE`)했을 때 디스크 공간이 즉시 줄어들지 않는 이유를 `Revision` 개념과 `Compaction`, `Defrag`의 차이를 들어 설명하시오.**

* **해설:**
* **MVCC & Revision:** `etcd`는 데이터를 덮어쓰거나 삭제하지 않고, 새로운 `Revision`(버전)을 계속 쌓아 나갑니다. `DELETE` 연산조차도 "삭제되었다"는 마킹(Tombstone)을 가진 새로운 버전을 기록하는 것입니다. 따라서 물리적 데이터 크기는 계속 증가합니다.
* **Compaction:** 특정 Revision 이전의 오래된 버전을 논리적으로 지웁니다. 하지만 이는 내부 B+Tree(boltDB)의 페이지를 "사용 가능(Free)" 상태로 마킹할 뿐, 파일 시스템상의 파일 크기를 줄여주지는 않습니다.
* **Defrag:** 실제 디스크 공간을 회수하려면 `defrag`를 수행해야 합니다. 이는 마치 디스크 조각 모음처럼, 듬성듬성한 데이터를 새 파일로 옮겨 적어 빈 공간을 OS에 반환하는 작업입니다. (운영 중 수행 시 I/O 블로킹 주의).



**Q112. 프로덕션 환경에서 `etcd` 성능 저하의 가장 큰 원인은 'Disk I/O Latency'이다. `WAL(Write Ahead Log)`과 `fsync` 동작 원리를 바탕으로, 왜 `etcd`에는 반드시 SSD/NVMe를 사용해야 하는지 설명하시오.**

* **해설:**
* **WAL & fsync:** `etcd`는 데이터의 일관성을 보장하기 위해 모든 변경 사항을 먼저 WAL 파일에 기록하고, `fsync` 시스템 콜을 통해 디스크에 완전히 저장되었음을 확인한 후에야 클라이언트에 성공 응답을 보냅니다.
* **Latency 민감도:** `fsync`가 느리면 리더 노드의 응답이 늦어지고, 팔로워 노드들의 복제도 지연되어 클러스터 전체의 쓰기 성능이 급격히 떨어집니다. HDD는 `fsync` 지연 시간이 길어 리더 선출 타임아웃을 유발할 수 있으므로, 낮은 레이턴시를 보장하는 SSD가 필수적입니다. (권장: 10ms 이하).



**Q113. Kubernetes에서 `ConfigMap`이나 `Secret`에 1MB가 넘는 대용량 데이터를 저장하는 것을 권장하지 않는 이유를 `etcd`의 기본 요청 크기 제한(`--max-request-bytes`)과 Raft 합의 과정의 관점에서 설명하시오.**

* **해설:**
* **기본 제한:** `etcd`의 기본 최대 요청 크기는 1.5MB입니다. 이를 초과하면 API 서버에서 거부됩니다. 설정을 늘릴 수는 있지만 권장하지 않습니다.
* **Raft 성능 저하:** Raft 알고리즘은 로그를 모든 멤버에게 복제해야 합니다. 큰 데이터가 네트워크를 타고 복제되는 동안 다른 작은 요청들이 블로킹되거나(Head-of-line blocking), 네트워크 대역폭을 점유하여 하트비트 실패를 유발, 불필요한 리더 선출이 발생할 수 있습니다. 따라서 대용량 데이터는 S3나 별도 스토리지에 저장하고 URL만 참조해야 합니다.



**Q114. 서로 다른 지역(Region)에 걸쳐 `etcd` 클러스터를 구성(Multi-region Deployment)할 때, 네트워크 지연으로 인한 잦은 리더 변경(Flapping)을 막기 위해 튜닝해야 할 파라미터인 `Heartbeat Interval`과 `Election Timeout`의 관계를 설명하시오.**

* **해설:**
* **Heartbeat Interval:** 리더가 팔로워에게 "나 살아있다"고 보내는 신호 주기(기본 100ms).
* **Election Timeout:** 팔로워가 리더의 신호를 받지 못했을 때, 리더가 죽었다고 판단하고 새 선거를 시작하기까지 기다리는 시간(기본 1000ms).
* **튜닝:** 지역 간 RTT(Round Trip Time)가 길다면, 하트비트가 도착하기 전에 타임아웃이 발생할 수 있습니다. 따라서 `Heartbeat Interval`을 RTT보다 충분히 길게 잡고, `Election Timeout`은 `Heartbeat Interval`의 최소 5~10배 이상(예: 5초)으로 설정하여, 일시적인 네트워크 지연이 전체 클러스터의 리더 재선출로 이어지지 않도록 해야 합니다.



**Q115. Kubernetes Controller들이 수천 개의 리소스를 감시(`Watch`)하고 있음에도 `etcd`가 효율적으로 동작할 수 있는 이유인 `Watch Stream`과 `gRPC Server`의 동작 방식을 설명하시오.**

* **해설:**
* **Polling vs Streaming:** 클라이언트가 계속 물어보는 폴링 방식이 아니라, 연결을 맺고 있으면 변경 사항이 발생했을 때 서버가 밀어주는(Push) 스트리밍 방식을 사용합니다.
* **이벤트 멀티플렉싱:** `etcd`는 내부적으로 변경 이벤트를 감지하는 하나의 채널을 두고, 수천 개의 클라이언트 `Watch` 요청을 효율적으로 관리합니다. 즉, 데이터 변경 시 해당 키를 구독하고 있는 모든 gRPC 스트림에 이벤트를 브로드캐스트하므로, 클라이언트 수가 늘어나도 리소스 소모가 선형적으로 급증하지 않습니다.



**Q116. `etcd` 클러스터의 멤버 3개 중 2개가 영구적으로 손상되어 쿼럼(Quorum)이 깨진 재해 상황(Disaster Recovery)이다. 살아남은 1개의 멤버에서 데이터를 복구하고 다시 3노드 클러스터로 복원하는 절차(`--force-new-cluster`)를 설명하시오.**

* **해설:**
* 과반수(2개)가 죽었으므로 쓰기 작업은 불가능하지만, 읽기는 가능할 수 있습니다.


1. 살아남은 노드의 `etcd` 프로세스를 중지합니다.
2. 데이터 디렉토리의 백업본을 뜹니다(snapshot).
3. `etcd`를 다시 시작할 때 `--force-new-cluster` 플래그를 사용하여, "기존 클러스터 정보는 무시하고 내가 새로운 리더가 되어 1인 클러스터를 시작한다"고 선언합니다.
4. 정상 구동되면 나머지 2개의 노드를 초기화하고 `member add` 명령어로 하나씩 추가하여 3노드 클러스터를 다시 구성합니다.


---

### **Section: Prometheus Deep Dive & Monitoring Strategy**

**Q117. Prometheus의 4가지 메트릭 타입(Counter, Gauge, Histogram, Summary) 중 `Histogram`과 `Summary`의 결정적인 차이를 `Aggregation`(집계) 가능 여부와 클라이언트/서버 부하 관점에서 설명하시오.**

* **해설:**
* **Histogram:** 클라이언트는 단순히 버킷(Bucket)별 횟수만 세고, 실제 분위수(Quantile, 예: P99) 계산은 서버(Prometheus)에서 `histogram_quantile` 함수로 수행합니다. 따라서 **여러 인스턴스의 메트릭을 합쳐서(Aggregation) 전체 평균이나 P99를 계산할 수 있습니다.** 서버 부하가 약간 증가합니다.
* **Summary:** 클라이언트 SDK가 직접 P50, P90 등을 계산해서 보냅니다. 서버 부하는 적지만, 이미 계산된 값이므로 **여러 인스턴스의 값을 합쳐서 다시 평균을 낼 수 없습니다(Not Aggregatable).** (예: A서버의 P99와 B서버의 P99 평균이 전체의 P99가 아님).



**Q118. Prometheus는 기본적으로 `Pull` 방식을 사용한다. 하지만 배치 작업(Batch Job)이나 일회성 스크립트의 메트릭을 수집하기 위해 `Pushgateway`를 사용할 때 발생하는 '데이터 영속성 문제(Zombie Metrics)'와 SPOF 위험성을 설명하시오.**

* **해설:**
* **Zombie Metrics:** Pushgateway는 메트릭을 받아서 캐싱하고, Prometheus가 긁어가게 합니다. 만약 배치 작업이 실패하거나 죽어도, Pushgateway는 마지막으로 받은 값을 계속 들고 있습니다. Prometheus는 작업이 죽었는지 모르고 계속 "성공"했던 마지막 값을 수집하게 되어 모니터링에 혼선을 줍니다. (반드시 작업 종료 시 메트릭을 삭제하도록 코딩해야 함).
* **SPOF:** 수천 개의 배치 작업이 하나의 Pushgateway에 의존하면, 그 게이트웨이가 죽었을 때 모든 모니터링이 중단됩니다. 따라서 Pushgateway는 제한적으로 사용해야 합니다.



**Q119. Prometheus 서버 하나가 감당할 수 없을 만큼 데이터가 늘어났을 때(Scale-out), `Federation` 방식의 한계점과 이를 극복하기 위한 `Thanos`나 `Cortex(Mimir)`의 아키텍처 차이(Remote Write/Sidecar)를 설명하시오.**

* **해설:**
* **Federation:** 상위 Prometheus가 하위 Prometheus의 데이터를 긁어가는 방식입니다. 데이터 양이 많으면 상위 서버도 결국 터집니다(수직 확장 한계). 또한, 장기 보관을 위한 솔루션이 아닙니다.
* **Thanos (Sidecar):** Prometheus 옆에 Sidecar를 띄워 데이터를 객체 스토리지(S3)로 업로드합니다. 조회 시 Store Gateway가 S3와 로컬 데이터를 합쳐서 보여줍니다. Prometheus 자체의 부하를 줄이고 무제한 장기 보관이 가능합니다.
* **Cortex/Mimir (Remote Write):** Prometheus가 수집한 데이터를 즉시 외부 쓰기 전용 클러스터로 쏘아 보냅니다. Prometheus는 단순 수집기로 전락하고, 중앙 저장소에서 모든 부하를 처리합니다.



**Q120. `High Cardinality` (고카디널리티) 문제가 Prometheus 메모리 사용량 폭증의 주원인이다. 이것이 의미하는 바와, `label` 설계 시 피해야 할 안티 패턴(Anti-pattern)을 예시(IP, UserID 등)를 들어 설명하시오.**

* **해설:**
* **의미:** 카디널리티는 유니크한 라벨 값의 조합(Time Series) 개수입니다. Prometheus는 각 조합마다 인덱스를 생성하고 메모리를 점유합니다.
* **안티 패턴:** 라벨 값으로 무제한으로 늘어날 수 있는 데이터(Unbounded Data)를 넣으면 안 됩니다.
* `user_id="user_1234"` (사용자가 100만 명이면 시계열도 100만 개)
* `client_ip="192.168.0.1"`
* `email="test@example.com"`


* 이런 데이터는 메트릭 라벨이 아니라 로그(Logging)나 트레이싱(Tracing)에 남겨야 합니다.



**Q121. `rate()` 함수와 `irate()` 함수의 차이점을 '계산 범위'와 '그래프의 급격한 변화(Spike) 표현' 관점에서 설명하고, 알람(Alerting) 규칙에는 왜 `irate`보다 `rate`를 권장하는지 서술하시오.**

* **해설:**
* **rate():** 지정된 범위(예: `[5m]`) 내의 평균 변화율을 계산합니다. 그래프가 부드럽게 그려지며, 스파이크가 뭉개질 수 있습니다.
* **irate():** 범위 내에서 *가장 마지막 두 샘플* 간의 차이만 계산합니다. 순간적인 변화(Micro-burst)를 감지하는 데 유리합니다.
* **Alerting 권장:** `irate`는 스크랩 주기에 따라 값이 너무 들쑥날쑥(Volatile)하여 오경보(Flapping)를 유발할 수 있습니다. `rate`는 추세를 기반으로 안정적인 알람을 제공하므로 알람 규칙에는 `rate`가 적합합니다.



**Q122. Prometheus 고가용성(HA)을 위해 두 개의 서버(A, B)를 동일한 설정으로 띄웠을 때, `Alertmanager`가 중복된 알람(Duplicate Alerts)을 사용자에게 보내지 않도록 처리하는 `Gossip Protocol` 기반의 클러스터링 메커니즘을 설명하시오.**

* **해설:**
* Prometheus A와 B는 서로 통신하지 않고 독립적으로 데이터를 수집하고 알람을 쏩니다. (Shared Nothing).
* 대신, 알람을 받는 **Alertmanager**들을 클러스터로 묶습니다. Alertmanager A가 알람을 받으면, Gossip Protocol을 통해 Alertmanager B, C에게 "나 이 알람 받았어"라고 전파합니다.
* 다른 Alertmanager들은 동일한 알람이 들어오면 "이미 처리 중"임을 인지하고 사용자(Slack, PagerDuty)에게 중복 전송을 하지 않습니다(Deduplication).



**Q123. `Counter` 메트릭을 사용할 때 앱이 재시작되면 카운터가 0으로 초기화된다(Counter Reset). `rate()` 함수가 이를 자동으로 처리하는 원리(Extrapolation)를 설명하시오.**

* **해설:**
* Prometheus의 `rate()` 함수는 타임스탬프를 비교하다가 값이 갑자기 줄어들면(예: 100 -> 0) "아, 재시작되었구나(Reset)"라고 판단합니다.
* 이때 단순히 0으로 계산하는 것이 아니라, 직전 값(100)을 유지한 상태에서 새로운 값(0부터 증가분)을 더하여, 마치 카운터가 계속 증가했던 것처럼 **보정(Extrapolation)**하여 변화율을 계산합니다. 덕분에 재시작 시에도 그래프가 마이너스로 떨어지지 않고 올바른 속도를 보여줍니다.



---
