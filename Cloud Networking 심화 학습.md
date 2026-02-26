# AWS · Azure · Kubernetes 네트워크 체계
## — 고급 아키텍처, 심화 개념, 글로벌 빅테크 면접 Q&A —

> 대상 독자: Senior Cloud / MLOps / LLMOps Engineer  
> 수준: 박사급 논문 수준의 기술 심화 문서  
> 언어: 한국어

---

# PART 1. 클라우드 네트워킹의 이론적 기반

## 1.1 Software-Defined Networking (SDN)과 오버레이 네트워크의 본질

클라우드 네트워킹은 물리적 하드웨어를 추상화하는 **Software-Defined Networking** 패러다임 위에 구축된다. SDN의 핵심은 Control Plane(라우팅 결정 로직)과 Data Plane(실제 패킷 포워딩)을 분리하는 것이다. 전통적인 네트워크 장비에서는 이 둘이 단일 장치 안에 결합되어 있었지만, 클라우드 환경에서는 중앙 집중식 컨트롤러가 모든 가상 스위치의 포워딩 테이블을 프로그래밍 방식으로 제어한다.

오버레이 네트워크(Overlay Network)는 기존 물리 네트워크(Underlay) 위에 논리적 터널을 생성하여 테넌트 격리, IP 주소 공간 독립성, 멀티테넌시를 구현한다. AWS VPC는 내부적으로 **Mapping Service**와 **Blackfoot** 엣지 장비를 활용한 커스텀 오버레이를 사용하며, 각 ENI(Elastic Network Interface)에 할당된 Private IP는 물리 호스트의 실제 IP와 완전히 분리된다. 이 매핑 시스템은 EC2 인스턴스가 다른 가용 영역으로 이동하더라도 일관된 IP 주소를 유지할 수 있게 해주는 핵심 인프라다.

**VXLAN(Virtual Extensible LAN)**은 오버레이 구현의 대표적 프로토콜로, L2 프레임을 UDP/IP 패킷으로 캡슐화하여 L3 네트워크를 가로질러 전달한다. VXLAN은 24비트 VNI(VXLAN Network Identifier)를 통해 최대 1,600만 개의 논리적 세그먼트를 지원함으로써, 4,096개로 제한된 VLAN의 확장성 문제를 해결한다. Kubernetes의 Flannel(VXLAN 모드)과 AWS VPC CNI 모두 이 원리를 다르게 활용한다.

## 1.2 BGP와 클라우드 라우팅 프로토콜의 심층 이해

**BGP(Border Gateway Protocol)**는 인터넷의 경로 벡터 라우팅 프로토콜로, 클라우드 환경에서 온프레미스-클라우드 연결의 핵심 프로토콜이다. AWS Direct Connect와 Azure ExpressRoute 모두 BGP를 통해 경로를 교환하며, 이 과정에서 AS(Autonomous System) 경계를 넘나드는 정교한 정책 제어가 가능하다.

eBGP(External BGP)와 iBGP(Internal BGP)의 구분은 실무에서 매우 중요하다. Direct Connect에서 고객 온프레미스 라우터(Customer Gateway)와 AWS Virtual Private Gateway 사이는 eBGP 세션을 형성한다. AS Path Prepending, Local Preference, MED(Multi-Exit Discriminator) 속성을 조작하여 Active/Active 혹은 Active/Passive 이중화 구성에서 트래픽 흐름을 정밀하게 제어할 수 있다.

특히 **Route Reflector** 패턴은 Kubernetes의 Calico BGP 모드에서도 동일하게 등장한다. 모든 BGP 피어가 iBGP full mesh를 형성하면 O(n²)의 세션이 필요하지만, Route Reflector를 도입하면 O(n)으로 줄일 수 있다. 대규모 Kubernetes 클러스터에서 Calico를 사용할 때 이 패턴은 필수적이다.

---

# PART 2. AWS 네트워크 아키텍처 심층 분석

## 2.1 VPC 아키텍처의 고급 설계 패턴

VPC(Virtual Private Cloud)는 단순한 가상 네트워크 이상의 복잡한 추상화 레이어다. 설계 시 고려해야 할 가장 중요한 원칙은 **주소 공간의 비중첩성(Non-overlapping CIDR)**이다. 멀티 계정, 멀티 리전 환경에서 VPC Peering이나 Transit Gateway를 통한 연결을 고려한다면, 전사 IP 주소 관리 계획(IPAM, IP Address Management)이 선행되어야 한다. AWS는 2021년 Amazon VPC IP Address Manager(IPAM)를 출시하여 조직 전체의 IP 풀을 중앙 관리할 수 있게 했다.

**서브넷 계층화 전략**은 보안과 라우팅의 근간이다. 일반적으로 Public Subnet(Internet Gateway 경로 있음), Private Subnet(NAT Gateway를 통한 아웃바운드만 허용), Isolated/DB Subnet(외부 통신 완전 차단)의 3계층 구조를 사용한다. 그러나 고성능이 요구되는 HPC나 대용량 ML 워크로드에서는 **Placement Group**과 함께 **Enhanced Networking(ENA, Elastic Fabric Adapter)**을 활용하는 서브넷 설계가 필요하다.

**EFA(Elastic Fabric Adapter)**는 일반 ENA와 달리 OS-bypass 메커니즘을 제공하여 MPI(Message Passing Interface) 기반 분산 딥러닝 학습 시 노드 간 레이턴시를 획기적으로 낮춘다. NVIDIA의 NVLink가 GPU 간 통신을 가속화하는 것처럼, EFA는 클러스터 내 노드 간 통신을 가속화한다. AWS에서 LLM 학습 클러스터를 운영하는 경우 p4d/p5 인스턴스와 EFA, 그리고 Cluster Placement Group의 조합은 사실상 표준 패턴이다.

## 2.2 Transit Gateway — 허브 앤 스포크의 진화

**Transit Gateway(TGW)**는 AWS 네트워크 아키텍처의 핵심 허브로, 수천 개의 VPC와 온프레미스 네트워크를 단일 허브를 통해 연결한다. TGW 이전에는 VPC Peering의 비전이적 특성(Non-transitive) 때문에 대규모 멀티-VPC 환경에서 메시(Mesh) 토폴로지가 불가피했고, 이는 복잡성을 기하급수적으로 증가시켰다.

TGW의 **Route Table 격리** 기능은 네트워크 세그멘테이션의 핵심 도구다. 예를 들어, 프로덕션 VPC와 개발 VPC를 별도의 TGW Route Table에 연결하면 두 그룹 간의 직접 통신을 차단하면서도 공유 서비스 VPC(DNS, 보안 스캐너, 모니터링)에는 모두 접근 가능하도록 구성할 수 있다. 이를 **세그먼트 격리(Segmented Isolation)** 패턴이라 부른다.

**TGW Network Manager**는 SD-WAN 통합과 글로벌 트랜싯 네트워크 구성을 가능하게 한다. AWS의 글로벌 가속 네트워크인 **Global Accelerator**와 결합하면, 사용자의 요청이 인터넷을 통해 AWS Edge Location에 도달한 후 AWS 백본 네트워크를 타고 목적지 리전까지 이동하게 되어, 퍼블릭 인터넷 경유 대비 레이턴시를 크게 줄일 수 있다.

## 2.3 AWS 네트워크 보안의 심층 메커니즘

**Security Group(SG)**은 상태 저장(Stateful) 방화벽으로, Connection Tracking을 통해 허용된 인바운드 연결의 아웃바운드 응답을 자동으로 허용한다. 반면 **Network ACL(NACL)**은 상태 비저장(Stateless) 방화벽으로 서브넷 경계에 적용된다. 이 두 계층의 상호작용을 이해하는 것이 중요한데, SG가 인스턴스 수준에서 세밀한 제어를 제공하는 동안, NACL은 서브넷 경계에서 광범위한 차단을 수행하는 방어 심층(Defense in Depth) 전략의 일환이다.

**VPC Flow Logs**는 ENI 수준에서 IP 트래픽 메타데이터를 캡처한다. 실제 패킷 내용은 포함되지 않지만, 5-tuple(소스 IP, 목적지 IP, 소스 포트, 목적지 포트, 프로토콜)과 함께 ACCEPT/REJECT 여부, 바이트/패킷 수를 기록한다. 이 로그를 Amazon Athena나 CloudWatch Logs Insights로 분석하면 비정상 트래픽 패턴, 포트 스캔, 데이터 유출 시도를 탐지할 수 있다.

**AWS Network Firewall**은 관리형 네트워크 방화벽 서비스로, Suricata 룰셋 기반의 IPS(Intrusion Prevention System) 기능을 제공한다. 기존 서드파티 방화벽 어플라이언스와 달리, AWS Network Firewall은 자동으로 수평 확장되어 처리량 제한이 없다. 특히 TGW와의 통합을 통해 모든 인터 VPC 트래픽을 중앙 인스펙션 VPC를 통과하도록 강제하는 **Centralized Inspection** 패턴이 엔터프라이즈 보안 아키텍처에서 권장된다.

## 2.4 PrivateLink와 VPC Endpoint — Zero Trust 네트워킹

**AWS PrivateLink**는 서비스 소비자와 제공자 사이에 퍼블릭 인터넷이나 VPC Peering 없이 프라이빗 연결을 구현하는 기술이다. 내부적으로 NLB(Network Load Balancer)와 ENI 기반의 **Interface Endpoint**를 통해 동작하며, 서비스 제공자의 IP 주소나 네트워크 구조가 노출되지 않는다는 점에서 보안성이 탁월하다.

**Gateway Endpoint**는 S3와 DynamoDB에만 적용되며, VPC Route Table에 프리픽스 리스트(Prefix List)를 추가하여 해당 서비스로의 트래픽이 인터넷 게이트웨이나 NAT를 거치지 않고 직접 AWS 백본을 통해 전달되도록 한다. 비용 절감(NAT Gateway 데이터 전송 비용 제거)과 보안(인터넷 노출 없음) 두 가지 이점을 동시에 제공한다.

Zero Trust 아키텍처 관점에서 PrivateLink의 중요성은 아무리 강조해도 지나치지 않는다. SaaS 제공자가 PrivateLink를 통해 서비스를 노출하면, 고객 VPC와 SaaS 백엔드 사이의 트래픽이 완전히 AWS 네트워크 내에서만 이동한다. 이는 금융, 의료 등 엄격한 데이터 주권 요건을 가진 산업에서 핵심 설계 패턴으로 자리잡았다.

---

# PART 3. Azure 네트워크 아키텍처 심층 분석

## 3.1 Azure Virtual Network와 그 고유한 특성

Azure Virtual Network(VNet)는 AWS VPC와 개념적으로 유사하지만 몇 가지 중요한 차이점이 있다. 가장 주목할 만한 차이는 **VNet이 기본적으로 리전 범위(Regional Scope)**라는 점이다. VNet은 단일 리전 내 여러 가용 영역(Availability Zones)에 걸쳐 확장되지만, 리전 간 통신은 별도의 메커니즘이 필요하다.

Azure의 **Application Security Group(ASG)**은 AWS Security Group과 유사한 역할을 하면서도, 논리적 그룹핑을 NIC 태그 기반으로 수행한다는 점이 다르다. 서버 역할(예: WebServer, DBServer)로 ASG를 생성하고, NSG(Network Security Group) 규칙에서 IP 주소 대신 ASG를 참조할 수 있어 동적 환경에서 보안 정책 관리가 매우 편리해진다.

**Accelerated Networking**은 Azure의 SR-IOV(Single Root I/O Virtualization) 구현으로, 네트워크 트래픽이 하이퍼바이저를 완전히 우회하여 NIC에서 직접 VM에 전달된다. 이는 CPU 사용률을 낮추고 레이턴시를 줄이며 처리량을 높이는 효과를 낳는다. 대규모 ML 워크로드나 고성능 데이터베이스 워크로드에서 Accelerated Networking 활성화 여부는 성능에 직접적인 영향을 준다.

## 3.2 Azure Hub-Spoke와 Virtual WAN

Azure의 허브-스포크 토폴로지는 **Hub VNet**에 공유 서비스(Azure Firewall, VPN/ExpressRoute Gateway, DNS)를 배치하고, 각 워크로드 VNet을 스포크로 허브에 피어링하는 구조다. AWS TGW와 달리 Azure VNet Peering도 기본적으로 비전이적(Non-transitive)이므로, 허브를 통한 트랜짓 라우팅을 위해서는 Azure Firewall이나 NVA(Network Virtual Appliance)에서 IP Forwarding을 활성화하거나, **Azure Route Server**를 사용해야 한다.

**Azure Virtual WAN(vWAN)**은 이 복잡성을 해소하는 관리형 서비스다. vWAN은 Microsoft가 관리하는 내부 라우터(Hub Router)를 제공하여, 스포크 VNet들 사이의 Any-to-Any 트랜짓 라우팅을 자동으로 처리한다. vWAN Standard SKU는 VNet-to-VNet, VPN-to-VNet, ExpressRoute-to-VNet 간 트랜짓을 모두 지원하며, BGP-enabled NVA를 허브에 직접 배치하여 커스텀 라우팅을 구현할 수도 있다.

## 3.3 ExpressRoute와 고급 연결 패턴

**Azure ExpressRoute**는 AWS Direct Connect에 대응하는 서비스로, 온프레미스 네트워크와 Azure 데이터센터를 전용 회선으로 연결한다. ExpressRoute는 **Private Peering**(Azure IaaS 서비스, VNet 연결), **Microsoft Peering**(M365, Azure PaaS 서비스)의 두 가지 피어링 타입을 지원한다.

**ExpressRoute Global Reach**는 두 ExpressRoute 회선을 Microsoft 백본을 통해 연결하여, 서로 다른 지역의 온프레미스 데이터센터 간 트래픽이 Microsoft 네트워크를 경유할 수 있게 한다. 이는 글로벌 기업에서 WAN 연결 비용 최적화와 성능 향상 수단으로 활용된다.

**ExpressRoute FastPath**는 데이터 경로에서 ExpressRoute Gateway를 우회하여 온프레미스에서 Azure VM으로의 트래픽이 게이트웨이 없이 직접 전달되게 함으로써 레이턴시를 줄이고 처리량을 높인다. 이는 대용량 데이터 전송이 빈번한 HPC 및 분석 워크로드에서 중요하다.

## 3.4 Azure DNS와 Private DNS의 설계

**Azure Private DNS Zone**은 VNet 내부에서만 해석되는 사설 DNS 네임스페이스를 제공한다. Private DNS Zone을 VNet에 연결(Link)할 때, **Auto-registration**을 활성화하면 해당 VNet의 VM들이 생성될 때 자동으로 A 레코드가 등록된다.

멀티 VNet 환경에서의 DNS 해석은 아키텍처적 복잡성을 가중시킨다. 허브-스포크 토폴로지에서 모든 스포크의 DNS 쿼리를 허브의 **Azure DNS Private Resolver**로 전달하고, 허브에서 온프레미스 DNS와 Azure Private DNS Zone을 중계하는 패턴이 권장된다. Azure DNS Private Resolver는 기존의 커스텀 DNS 서버(VM 기반) 대비 가용성이 높고 운영 부담이 낮다.

---

# PART 4. Kubernetes 네트워크 아키텍처 심층 분석

## 4.1 Kubernetes 네트워킹의 핵심 모델과 CNI

Kubernetes는 네트워킹에 관한 몇 가지 근본적인 가정을 전제로 한다. 첫째, 모든 Pod는 NAT 없이 다른 모든 Pod와 직접 통신 가능해야 한다. 둘째, 모든 노드는 NAT 없이 모든 Pod와 통신 가능해야 한다. 셋째, Pod 내부에서 보이는 자신의 IP는 외부에서 보이는 IP와 동일해야 한다(NAT 없는 flat 네트워크).

이 요건을 충족하는 방법은 CNI(Container Network Interface) 플러그인에 위임된다. CNI 플러그인은 Pod 생성 시 네트워크 인터페이스 생성, IP 주소 할당, 라우팅 설정을 담당한다. 주요 CNI 플러그인별 핵심 구현 방식을 이해하는 것은 트러블슈팅과 성능 최적화의 전제 조건이다.

**AWS VPC CNI**는 Pod에 실제 VPC IP 주소를 직접 할당하는 방식으로, 오버레이 없이 네이티브 VPC 라우팅을 사용한다. 각 노드의 ENI에 Secondary IP를 사전 할당(Warm Pool)하고, Pod 생성 시 이 IP를 Pod의 eth0에 연결한다. 이 방식의 장점은 VPC Flow Logs, Security Group, 라우팅 테이블이 Pod 수준에서 직접 적용된다는 점이지만, 인스턴스 유형별 ENI/IP 수 제한에 의해 노드당 최대 Pod 수가 제한된다는 단점이 있다. 이를 극복하기 위한 **VPC CNI Prefix Delegation** 기능은 /28 CIDR 블록을 ENI에 할당하여 IP 밀도를 크게 높인다.

**Calico**는 L3 라우팅 기반의 BGP를 사용하며, 오버레이 없는(Overlay-free) 순수 IP 라우팅 모드에서 최고의 성능을 발휘한다. 각 노드는 BGP 스피커로 동작하여 자신이 호스팅하는 Pod CIDR을 인접 노드에 광고한다. Calico의 가장 강력한 기능 중 하나는 **Network Policy** 구현으로, Kubernetes 표준 NetworkPolicy API에 더해 Calico 고유의 GlobalNetworkPolicy와 HostEndpoint 정책을 통해 노드 레벨 방화벽까지 구현할 수 있다.

**Cilium**은 eBPF(extended Berkeley Packet Filter)를 활용하여 커널 수준에서 네트워킹, 보안, 관찰 가능성을 구현하는 차세대 CNI다. iptables 기반의 전통적 CNI 대비 O(1) 복잡도의 패킷 처리, 커널 우회 없는 고성능, 그리고 L7 인식 네트워크 정책이 핵심 차별점이다. Google GKE Dataplane V2, AWS EKS에서 Cilium 지원이 공식화되면서 프로덕션 표준으로 빠르게 자리 잡고 있다. Cilium의 **Hubble**은 eBPF 기반의 네트워크 관찰 도구로, 서비스 맵, 흐름 로그, 레이턴시 히스토그램을 실시간으로 제공한다.

## 4.2 kube-proxy와 iptables, IPVS의 비교

**kube-proxy**는 Kubernetes Service 추상화를 구현하는 컴포넌트로, ClusterIP 서비스로 들어오는 트래픽을 백엔드 Pod들로 로드밸런싱한다. 기본 구현은 **iptables** 모드로, 각 서비스와 엔드포인트에 대한 DNAT 규칙을 iptables 체인에 삽입한다. 이 접근 방식의 문제는 서비스 수가 증가할수록 iptables 규칙이 O(n) 선형 검색으로 처리되어 레이턴시가 증가한다는 점이다. 1,000개 이상의 서비스가 존재하는 클러스터에서 이 문제는 무시할 수 없는 성능 저하를 야기한다.

**IPVS(IP Virtual Server) 모드**는 커널 내의 해시 테이블을 사용하여 O(1) 복잡도로 서비스 룩업을 수행한다. 또한 Round-Robin 외에도 Least Connection, Destination Hashing, Source Hashing 등 다양한 로드밸런싱 알고리즘을 지원한다. 10,000개 이상의 서비스가 존재하는 대규모 클러스터에서는 IPVS 모드가 사실상 필수적이다.

**Cilium의 eBPF 기반 접근**은 kube-proxy를 완전히 대체할 수 있다. eBPF 프로그램이 커널의 패킷 처리 경로에 직접 삽입되어, netfilter 스택을 우회함으로써 iptables나 IPVS보다 훨씬 낮은 레이턴시를 달성한다. 이는 kube-proxy 없는(Kube-proxy-free) 클러스터 구성을 가능하게 한다.

## 4.3 Service Mesh — Istio와 Envoy의 아키텍처

**Service Mesh**는 서비스 간 통신을 인프라 계층에서 처리하는 전용 인프라 레이어다. 각 Pod에 주입된 **사이드카 프록시(Sidecar Proxy)**가 모든 인바운드/아웃바운드 트래픽을 가로채고, 상호 TLS, 로드밸런싱, 서킷 브레이커, 트레이싱, 재시도를 투명하게 처리한다.

**Istio**의 아키텍처는 **Istiod**라는 컨트롤 플레인 컴포넌트와 데이터 플레인의 Envoy 프록시로 구성된다. Istiod는 Pilot(트래픽 관리), Citadel(인증서 관리), Galley(설정 검증) 기능을 단일 바이너리에 통합한다. Istiod는 xDS API(Envoy Discovery Service)를 통해 각 Envoy 프록시에 라우팅 규칙, 클러스터 정보, 엔드포인트 목록, 리스너 설정을 배포한다.

**Envoy**의 아키텍처는 스레드 모델 관점에서 이해해야 한다. 메인 스레드가 컨트롤 플레인과의 xDS 통신과 설정 업데이트를 담당하고, Worker 스레드들이 실제 I/O와 패킷 처리를 병렬로 수행한다. 각 Worker 스레드는 독립적인 이벤트 루프를 가지며, 커넥션은 특정 Worker 스레드에 고정(Pinned)된다.

**mTLS(mutual TLS)**는 Istio의 핵심 보안 기능이다. Citadel은 각 서비스 어카운트에 X.509 인증서를 발급하고, 이를 Envoy가 TLS 핸드셰이크에 사용한다. PeerAuthentication 리소스의 STRICT 모드를 설정하면 해당 네임스페이스나 전체 메시에서 mTLS가 강제된다. 이는 평문 텍스트 통신을 차단하고 서비스 간 인증을 보장하는 제로 트러스트 네트워킹의 핵심 구현이다.

**Ambient Mesh**는 Istio의 차세대 모드로, 사이드카 없이 노드 수준의 **ztunnel(Zero-Trust Tunnel)**과 L7 처리를 위한 **waypoint proxy**를 활용한다. 사이드카 모드의 오버헤드(메모리, CPU, 주입 복잡성)를 줄이면서 동등한 기능을 제공하는 방향으로 발전하고 있다.

## 4.4 Ingress와 Gateway API — 진화하는 트래픽 입구

**Kubernetes Ingress**는 클러스터 외부에서 내부 서비스로의 HTTP/HTTPS 트래픽을 관리하는 API 오브젝트다. Ingress Controller(NGINX, AWS ALB, Traefik, Istio Ingress Gateway 등)가 Ingress 리소스를 Watch하여 실제 로드밸런서를 프로그래밍한다.

**Gateway API**는 Ingress의 한계를 극복하기 위해 설계된 차세대 API로, Kubernetes SIG-Network에서 관리한다. Gateway API는 역할 기반(Role-Based) 분리를 제공하여 인프라 팀이 GatewayClass와 Gateway를 관리하고, 애플리케이션 팀이 HTTPRoute, TCPRoute, GRPCRoute를 관리하도록 권한을 분리할 수 있다. 이 API는 점진적으로 Ingress를 대체할 예정으로, Istio, Envoy Gateway, Cilium, AWS Load Balancer Controller 등 주요 구현체들이 지원하고 있다.

**AWS Load Balancer Controller**는 Gateway API를 통해 ALB(HTTP)와 NLB(TCP/UDP)를 프로비저닝한다. 특히 **Target Group Binding** 기능을 통해 Kubernetes Service를 기존 ALB Target Group에 연결하거나, **IP 모드** Target Group을 사용하여 Pod IP를 직접 등록함으로써 NLB-to-Pod 경로에서 NodePort의 추가 홉을 제거할 수 있다.

## 4.5 eBPF — 네트워킹의 패러다임 전환

**eBPF(extended Berkeley Packet Filter)**는 Linux 커널에서 안전하게 실행되는 샌드박스 프로그램을 주입하는 기술로, 클라우드 네이티브 네트워킹의 게임 체인저로 부상했다. eBPF 프로그램은 커널 소스 코드 수정이나 커널 모듈 로드 없이, 커널의 특정 훅 포인트(네트워크 스택의 XDP, TC, 소켓 레이어 등)에 동적으로 주입된다.

**XDP(eXpress Data Path)**는 NIC 드라이버 수준에서 패킷을 처리하는 가장 빠른 eBPF 훅이다. 패킷이 커널 네트워크 스택에 진입하기 전에 처리되므로, DDoS 방어, 로드밸런싱, 패킷 필터링에서 최소한의 레이턴시로 최대 수십 Gbps의 처리량을 달성할 수 있다. Facebook의 Katran 로드밸런서와 Cloudflare의 DDoS 방어 시스템이 XDP 기반으로 구현되어 있다.

**TC(Traffic Control) Hook**은 XDP 대비 커널 스택에서 더 깊은 위치에 있어 패킷의 메타데이터(소켓 정보, 컨테이너 ID 등)에 접근 가능하다. Cilium은 TC Hook을 통해 네트워크 정책 시행, 서비스 로드밸런싱, 암호화를 구현한다.

---

# PART 5. 멀티 클라우드와 하이브리드 네트워킹 패턴

## 5.1 AWS와 Azure의 상호 연결

멀티 클라우드 환경에서 AWS와 Azure 간의 직접 연결은 보통 Equinix나 Megaport 같은 **네트워크 교환소(Network Exchange)**를 통해 구현된다. AWS Direct Connect 연결과 Azure ExpressRoute 연결이 동일한 공동 배치(Colocation) 시설에 종단점을 가질 때, 두 연결을 직접 패치할 수 있어 클라우드 간 트래픽이 공용 인터넷을 경유하지 않는다.

**Megaport Cloud Router(MCR)**는 가상 라우터로, AWS와 Azure를 포함한 다양한 클라우드 제공자의 연결을 단일 논리적 라우팅 도메인 안에서 BGP로 상호 연결할 수 있다. 이는 멀티 클라우드 트랜짓 아키텍처를 구현하는 실용적인 방법이다.

## 5.2 Service Mesh Federation — 멀티 클러스터 메시

대규모 마이크로서비스 환경에서는 단일 Kubernetes 클러스터의 한계를 넘어 여러 클러스터에 걸친 서비스 메시 연합이 필요하다. **Istio Multi-Cluster** 구성에서는 Primary-Remote 모델(단일 컨트롤 플레인이 여러 클러스터의 데이터 플레인을 제어)과 Replicated Control Planes 모델(각 클러스터에 독립적인 Istiod) 두 가지 방식이 있다.

**Cilium Cluster Mesh**는 여러 Kubernetes 클러스터를 단일 네트워크 도메인으로 연합하는 기능으로, 클러스터 간 서비스 디스커버리, 로드밸런싱, 네트워크 정책을 지원한다. 각 클러스터의 Cilium이 etcd를 통해 다른 클러스터의 엔드포인트 정보를 공유하고, eBPF를 통해 클러스터 경계를 넘는 서비스 호출을 투명하게 처리한다.

---

# PART 6. 네트워크 관찰 가능성과 성능 엔지니어링

## 6.1 분산 추적과 네트워크 텔레메트리

현대 클라우드 네이티브 환경에서 네트워크 문제 진단은 단순한 ping/traceroute를 넘어선 **분산 추적(Distributed Tracing)** 기법이 필요하다. **OpenTelemetry**는 트레이스, 메트릭, 로그를 통합하는 표준 프레임워크로, 서비스 간 컨텍스트 전파(Trace Context Propagation)를 통해 요청이 여러 서비스와 네트워크 홉을 거치는 경로를 시각화한다.

**eBPF 기반의 관찰 가능성**은 애플리케이션 코드 수정 없이 커널 수준에서 네트워크 이벤트를 수집한다. **Pixie**(CNCF 프로젝트)는 eBPF를 활용하여 PII 데이터를 외부로 전송하지 않고 클러스터 내에서 자동으로 프로토콜 파싱, 서비스 맵 생성, 성능 메트릭 수집을 수행한다.

## 6.2 네트워크 성능 병목 진단 방법론

네트워크 성능 문제는 주로 대역폭(Bandwidth), 레이턴시(Latency), 패킷 손실(Packet Loss), 지터(Jitter) 네 가지 차원에서 분석된다. 클라우드 환경에서는 여기에 더해 **스로틀링(Throttling)**이 중요한 변수로 추가된다.

AWS에서는 **EC2 인스턴스 네트워크 성능 한도**가 있으며, 이는 특정 인스턴스 유형에서 제공되는 최대 네트워크 대역폭과 PPS(Packets Per Second) 한도로 표현된다. 버스트 가능한 인스턴스(T시리즈)의 네트워크 크레딧 메커니즘, 그리고 ENA 드라이버의 RSS(Receive Side Scaling) 설정이 실제 성능에 큰 영향을 미친다.

**MTU(Maximum Transmission Unit)**와 **Jumbo Frame** 설정은 대용량 데이터 전송 성능에 직접적인 영향을 준다. AWS VPC는 기본 1500 MTU를 지원하며, 동일 VPC 내 또는 Direct Connect를 통한 경우 9001 MTU(Jumbo Frame)를 지원한다. Jumbo Frame을 사용하면 패킷 헤더 오버헤드 비율이 줄어들어 대용량 데이터 전송 시 처리량이 향상된다. 그러나 Jumbo Frame은 경로상 모든 링크가 지원해야 하므로, VPC Peering 경계나 인터넷 게이트웨이를 통과하는 경우 자동으로 1500 MTU로 분절(Fragmentation)된다.

---

# PART 7. 글로벌 빅테크 면접 Q&A 30선

> Amazon, Apple, Nvidia, Google, Microsoft, Tesla의 Senior/Staff/Principal Cloud/Network Engineer 면접에 최적화된 질문과 심화 답변

---

## [AWS/Amazon 계열 — 심화]

### Q1. AWS VPC에서 패킷이 EC2 인스턴스 A에서 동일 서브넷의 인스턴스 B로 전달되는 과정을 하이퍼바이저 수준까지 설명하라.

**답변:** EC2 인스턴스의 ENI(Elastic Network Interface)는 물리 호스트의 가상 NIC로 매핑된다. 패킷이 인스턴스 A의 OS에서 생성되면 ENA 드라이버를 통해 Nitro 하이퍼바이저의 가상 스위치로 전달된다. Nitro는 소프트웨어 정의 방식의 전용 ASIC으로, 패킷의 목적지 MAC/IP를 기반으로 VPC Mapping Service에 질의한다. Mapping Service는 목적지 인스턴스 B의 ENI가 어떤 물리 호스트에 위치하는지, 해당 호스트로의 언더레이 IP 주소는 무엇인지를 반환한다. 그 후 원본 L2 프레임이 물리 네트워크 헤더로 캡슐화(오버레이 인캡슐레이션)되어 목적지 물리 호스트로 전달되고, 대상 호스트의 Nitro가 인캡슐레이션을 제거하여 인스턴스 B의 ENI로 패킷을 전달한다. Security Group 규칙은 이 과정에서 Nitro ASIC 내부에서 하드웨어 수준으로 시행된다.

---

### Q2. Transit Gateway와 VPC Peering 중 어느 것을 선택해야 하는가? 트레이드오프를 심층적으로 논하라.

**답변:** VPC Peering은 두 VPC 간 직접 연결로, 대역폭 제한이 없고 추가 홉이 없어 레이턴시가 최소화된다. 비용도 데이터 전송 비용만 부과되어 Transit Gateway의 시간당 attachment 비용이 없다. 그러나 비전이성 때문에 n개의 VPC를 완전 연결하려면 n(n-1)/2 개의 피어링이 필요하고, 최대 125개 피어링이라는 hard limit이 있다. 반면 Transit Gateway는 단일 허브로 수천 개의 VPC와 온프레미스를 연결하고, Route Table 격리를 통한 정교한 세그멘테이션이 가능하다. 단, 모든 트래픽이 TGW를 경유하므로 추가 레이턴시(일반적으로 수십 μs)와 처리량 한도(최대 50 Gbps/AZ per attachment)가 존재한다. 따라서 VPC 수가 소수(10개 미만)이고 단순 연결이 목표라면 VPC Peering이 적합하고, 엔터프라이즈 규모의 멀티 계정 환경이라면 Transit Gateway가 적합하다.

---

### Q3. AWS에서 EKS Pod가 인터넷에 나갈 때의 SNAT 동작을 설명하고, 이것이 대규모 환경에서 왜 문제가 될 수 있는지 설명하라.

**답변:** VPC CNI를 사용하는 EKS에서 Pod는 ENI의 Secondary IP를 직접 사용한다. Pod가 VPC 외부(인터넷)로 나갈 때, Worker Node의 OS에서 iptables SNAT 규칙에 의해 Pod IP가 Worker Node의 Primary ENI IP로 변환된다. 문제는 NAT 연결 추적 테이블(conntrack)의 용량이다. 대규모 클러스터에서 많은 Pod가 단시간에 많은 외부 연결을 맺으면 conntrack 테이블이 고갈되고, 이는 새로운 연결 설정 실패, 기존 연결의 패킷 드롭으로 이어진다. 해결책으로는 NAT Gateway로 트래픽을 오프로드하거나, Elastic IP를 Pod에 직접 할당하거나, VPC CNI의 `AWS_VPC_K8S_CNI_EXTERNALSNAT=true` 플래그로 노드 SNAT를 비활성화하고 NAT Gateway를 사용하는 방법이 있다. 또한 `net.netfilter.nf_conntrack_max` 커널 파라미터를 워크로드에 맞게 조정하는 것이 중요하다.

---

### Q4. AWS PrivateLink 기반 서비스를 설계할 때 고려해야 할 고가용성 패턴은 무엇인가?

**답변:** PrivateLink의 서비스 제공자 측 NLB는 여러 AZ에 걸쳐 배포되어야 하며, Cross-Zone Load Balancing 활성화 여부를 트레이드오프와 함께 결정해야 한다. 활성화 시 AZ 간 불균형 트래픽을 해소할 수 있지만 데이터 전송 비용이 발생한다. Interface Endpoint는 소비자 VPC의 각 AZ 서브넷에 별도 ENI로 생성하여 AZ 격리 장애 시에도 다른 AZ의 Endpoint를 통해 서비스 접근이 가능하도록 해야 한다. Endpoint Service의 DNS 이름은 리전 도메인과 AZ 특정 도메인 모두 사용할 수 있으며, 소비자가 AZ 특정 DNS 이름을 사용하면 트래픽이 해당 AZ 내에서만 처리되어 레이턴시를 최소화한다. 헬스체크는 NLB 타겟 그룹 수준과 Endpoint Service 수준 모두에서 적절히 설정되어야 한다.

---

### Q5. AWS Direct Connect의 가상 인터페이스 유형(Public VIF, Private VIF, Transit VIF)의 차이와 사용 시나리오를 설명하라.

**답변:** Public VIF는 온프레미스에서 AWS의 공개 IP 주소 범위(S3, EC2의 공인 IP, CloudFront 등)로 접근할 때 사용되며, AWS는 자신의 공인 IP 범위 전체를 BGP로 광고한다. Private VIF는 특정 VGW(Virtual Private Gateway)에 연결되어 해당 VGW에 연결된 VPC 내의 사설 IP로만 접근 가능하다. Transit VIF는 TGW에 연결되어 TGW에 attached된 모든 VPC와 온프레미스 사이의 통신을 허용한다. 실제 설계 시 고려할 점은, Private VIF는 VIF당 하나의 VGW에만 연결 가능하므로 연결해야 할 VPC가 많다면 TGW + Transit VIF 조합이 더 확장적이다. 보안 측면에서 Public VIF는 온프레미스 IP가 인터넷에 노출되지 않으면서도 AWS 퍼블릭 서비스에 접근하는 효율적인 방법이지만, BGP 라우트 필터링을 통해 불필요한 트래픽 흐름을 제한해야 한다.

---

### Q6. Amazon이 운영하는 글로벌 서비스에서 Anycast를 어떻게 활용하는지, 그리고 Global Accelerator의 내부 동작 원리를 설명하라.

**답변:** Anycast는 동일한 IP 주소를 여러 위치에서 광고하여 클라이언트가 BGP 라우팅에 의해 가장 가까운 서버로 자동 연결되는 기술이다. AWS Global Accelerator는 두 개의 정적 Anycast IP 주소를 제공하며, 이 IP는 전 세계 AWS Edge Location에서 광고된다. 사용자의 패킷은 인터넷을 통해 가장 가까운 Edge Location에 도달하고, 이후 AWS 글로벌 백본(Tier-1 ISP 수준의 사설 네트워크)을 통해 목적지 리전의 엔드포인트로 전달된다. 이는 퍼블릭 인터넷 경로 전체를 사용하는 것보다 BGP 다중 경로 불안정성, 라우팅 루프, ISP 간 대역폭 제한의 영향을 받지 않아 안정적인 성능을 제공한다. Health Check를 통해 장애 발생 시 트래픽을 수초 내에 건강한 엔드포인트로 전환할 수 있어 DNS 기반 장애 조치보다 훨씬 빠른 복구가 가능하다.

---

## [Google/GCP 계열]

### Q7. Kubernetes NetworkPolicy의 한계는 무엇이며, 이를 극복하기 위한 Cilium의 L7 정책은 어떻게 동작하는가?

**답변:** 표준 Kubernetes NetworkPolicy는 L3/L4 수준(IP, Port, 프로토콜)에서만 트래픽을 제어할 수 있다. 예를 들어 동일 포트(8080)를 사용하는 두 가지 API 엔드포인트(/users와 /admin)를 서로 다른 정책으로 제어하는 것은 표준 NetworkPolicy로는 불가능하다. Cilium의 CiliumNetworkPolicy는 HTTP 메서드, URL 경로, 헤더, gRPC 서비스 메서드까지 정책 기준으로 사용할 수 있다. 내부적으로 Cilium은 eBPF 소켓 레이어 훅을 통해 패킷을 가로채고, 내장된 경량 Envoy 프로세스에 L7 파싱을 위임한 후 정책 결정 결과를 eBPF 맵에 기록한다. 이 방식은 사이드카 없이 노드 에이전트(DaemonSet) 하나로 L7 정책을 구현하므로, Istio 사이드카 대비 리소스 오버헤드가 적다. 단, HTTP/2와 gRPC의 멀티플렉싱으로 인해 커넥션과 스트림 수준에서의 세밀한 정책 시행은 여전히 도전적인 부분이다.

---

### Q8. Google의 Andromeda 네트워크 가상화 플랫폼과 이를 Kubernetes에서 사용하는 GKE Dataplane V2의 관계를 설명하라.

**답변:** Andromeda는 Google Cloud의 네트워크 가상화 스택으로, 각 VM의 호스트에 배포된 소프트웨어 데이터플레인이 패킷 포워딩, 보안 정책, 로드밸런싱을 처리한다. Andromeda는 커스텀 Mercury 드라이버를 통해 하이퍼바이저를 우회한 고성능 데이터플레인을 제공한다. GKE Dataplane V2는 Google이 Cilium 기반으로 구현한 Kubernetes 네트워크 플러그인으로, eBPF를 Andromeda 위에서 동작시켜 Pod 간 통신, NetworkPolicy, 서비스 로드밸런싱을 처리한다. 특히 GKE Dataplane V2는 Hubble의 관찰 가능성을 GKE 콘솔에 통합하여 네트워크 흐름 로그를 Pod 레벨에서 시각화한다. Andromeda가 VPC 수준의 네트워크 가상화를 담당하고, GKE Dataplane V2가 Kubernetes 오버레이 네트워크를 담당하는 두 계층이 협력하는 구조다.

---

### Q9. Kubernetes에서 IPAM(IP Address Management) 전략을 설계할 때 고려해야 할 핵심 요소는 무엇인가?

**답변:** IPAM 설계의 첫 번째 고려 사항은 Pod CIDR과 Service CIDR, Node CIDR 사이의 주소 공간 분리와 충분한 크기 확보다. 클러스터 생성 후 이 CIDR을 변경하는 것은 매우 어렵기 때문에 초기 설계가 중요하다. 대규모 클러스터(1,000노드 이상)에서는 /16 Pod CIDR도 부족할 수 있으며, AWS EKS에서는 VPC CIDR 용량이 직접적인 제약이 된다. 이를 해결하는 방법으로는 EKS의 Custom Networking(Pod를 별도 서브넷에 배치)과 Prefix Delegation(/28 블록 단위 IP 할당)이 있다. 멀티 클러스터 환경에서는 클러스터 간 Pod CIDR 중복이 없어야 Cluster Mesh나 직접 통신이 가능하므로, 전사적 CIDR 레지스트리 관리가 필수적이다. IPv4 고갈 문제에 대한 장기적 해결책으로 IPv6 Dual-stack 또는 IPv6-only 클러스터 전환을 중장기 계획에 포함해야 한다.

---

## [Microsoft/Azure 계열]

### Q10. Azure의 AcceleratedNetworking과 DPDK를 활용한 고성능 NVA 설계 방법을 설명하라.

**답변:** NVA(Network Virtual Appliance)는 Azure에서 방화벽, IDS/IPS, WAN 가속기 역할을 하는 VM 기반 네트워크 어플라이언스다. 기본 Linux NVA는 커널 네트워크 스택을 통해 패킷을 처리하므로 컨텍스트 스위칭과 메모리 복사 오버헤드가 크다. DPDK(Data Plane Development Kit)는 이를 해결하기 위해 NIC를 사용자 공간(User Space)에서 직접 제어하고, 폴링 기반으로 패킷을 처리하여 인터럽트 오버헤드를 제거한다. Azure의 Accelerated Networking(SR-IOV)은 DPDK와 함께 사용할 때 최대 효과를 발휘한다. SR-IOV의 VF(Virtual Function)를 DPDK PMD(Poll Mode Driver)가 직접 제어하면, 하이퍼바이저 완전 우회와 제로 카피(Zero-Copy) 패킷 처리가 가능해진다. 실무에서는 FD.io VPP(Vector Packet Processor)를 DPDK 위에 구현하여 수십 Gbps 수준의 L3/L4 처리를 단일 VM에서 달성할 수 있다. 이 구성에서는 NUMA(Non-Uniform Memory Access) 최적화, CPU 코어 핀닝, HugePages 설정이 성능의 관건이다.

---

### Q11. Azure의 UDR(User Defined Route)과 BGP 사이의 우선순위 충돌을 설명하고, Azure Route Server가 이를 어떻게 해결하는지 설명하라.

**답변:** Azure VNet에서 라우팅 우선순위는 System Route보다 UDR이 높고, BGP를 통해 학습된 경로는 UDR보다 낮은 우선순위를 가진다(기본 설정). 이는 NVA를 거치도록 강제하는 UDR이 ExpressRoute BGP 경로를 재정의할 수 있음을 의미한다. 전통적인 허브-스포크에서 NVA를 통한 트랜짓을 구현하려면 스포크마다 NVA를 가리키는 UDR을 수동으로 관리해야 했다. Azure Route Server는 이 문제를 해결하는 관리형 BGP 엔드포인트로, NVA가 Route Server와 iBGP 세션을 맺고 학습된 경로를 VNet에 프로그래밍한다. 즉, NVA가 BGP를 통해 온프레미스로부터 학습한 경로를 Route Server에 재광고하면, Route Server가 VNet의 라우팅 테이블에 자동으로 반영한다. 이를 통해 수동 UDR 관리 없이 동적 라우팅이 가능해지고, 온프레미스 경로 변경이 자동으로 클라우드 라우팅에 반영된다.

---

### Q12. Azure Kubernetes Service(AKS)에서 Azure CNI Overlay와 Kubenet, 전통적 Azure CNI의 차이점과 각각의 선택 기준을 설명하라.

**답변:** 전통적 Azure CNI는 Pod에 VNet IP를 직접 할당하여 VNet 수준에서 Pod가 가시적이지만, 노드당 최대 30~250개의 IP를 VNet CIDR에서 소비하므로 IP 고갈이 빠르게 발생한다. Kubenet은 노드에만 VNet IP를 할당하고 Pod는 노드 로컬 오버레이 CIDR을 사용하므로 VNet IP 소비가 적다. 단, 다른 VNet 리소스에서 Pod에 직접 접근하려면 UDR을 통해 Pod 서브넷 라우팅을 수동 관리해야 한다. Azure CNI Overlay는 두 방식의 장점을 결합한 방식으로, 노드는 VNet IP를 사용하고 Pod는 RFC 1918 오버레이 공간에서 IP를 받는다. 오버레이 트래픽이 노드에서 캡슐화되어 전달되므로 VNet IP 고갈 없이 고밀도 Pod 배포가 가능하다. 선택 기준으로는 네이티브 VNet 통합이 필요(Service Endpoint, 특정 NSG 규칙)하면 전통적 Azure CNI, IP 절약이 최우선이면 Kubenet, 대규모 클러스터에서 오버레이의 유연성과 VNet 통합의 일부를 함께 원한다면 Azure CNI Overlay가 적합하다.

---

## [Nvidia 계열 — HPC/ML 네트워킹]

### Q13. GPU 클러스터 학습 환경에서 RDMA over Converged Ethernet(RoCE)과 InfiniBand의 차이점과 클라우드에서의 구현 방법을 설명하라.

**답변:** InfiniBand는 데이터센터용 고성능 네트워크 표준으로, RDMA(Remote Direct Memory Access)를 네이티브로 지원하고 MPI 워크로드에 최적화된 매우 낮은 레이턴시(< 1μs)와 높은 처리량(최신 NDR: 400 Gbps)을 제공한다. IB 패브릭은 전용 스위치(Quantum 시리즈)와 HCA(Host Channel Adapter)가 필요하며, 이더넷과 완전히 별도의 패브릭이다. RoCE(RDMA over Converged Ethernet)는 기존 이더넷 인프라 위에서 RDMA의 이점을 제공하는 방식으로, RoCEv2는 UDP/IP 위에서 IB Transport를 실행하여 L3 라우팅이 가능하다. AWS에서는 p4d 인스턴스가 4개의 100Gbps EFA 인터페이스를 제공하며, EFA는 내부적으로 SRD(Scalable Reliable Datagram) 프로토콜을 통해 RDMA와 유사한 OS-bypass 통신을 구현한다. Azure에서는 NDv4 시리즈 VM이 InfiniBand HDR(200 Gbps)을 직접 지원하며, InfiniBand 어댑터가 VM에 SR-IOV로 패스스루된다. NCCL(NVIDIA Collective Communications Library)은 이 하드웨어 경로를 자동으로 탐지하고 최적 집합 통신(All-Reduce, All-Gather 등) 알고리즘을 선택한다.

---

### Q14. Kubernetes에서 GPU 간 통신을 최적화하기 위한 Topology-Aware Scheduling 구현 방법을 설명하라.

**답변:** GPU 집합 학습 시 통신 성능은 노드 내 GPU 토폴로지(NVLink, PCIe 도메인), 노드 간 네트워크 토폴로지(같은 Top-of-Rack 스위치 여부, 같은 Placement Group 여부)에 크게 의존한다. Kubernetes의 **Topology Manager**는 CPU, NUMA 노드, 장치의 토폴로지 정합성을 고려하여 Pod를 스케줄링하지만, 기본적으로 멀티 노드 수준의 네트워크 토폴로지를 고려하지 않는다. 이를 해결하기 위해 NVIDIA는 **GPU Feature Discovery(GFD)**와 **Topology-aware GPU scheduling**을 개발했다. NVIDIA의 gang scheduling 확장과 결합하면, All-Reduce 통신 그룹의 모든 Pod가 동일 ToR 스위치나 클러스터 배치 그룹 내에 배치되도록 강제할 수 있다. AWS EKS에서는 EC2 인스턴스의 플레이스먼트 그룹 정보를 노드 레이블로 노출하고, Affinity/Anti-affinity 규칙으로 스케줄링 제약을 표현한다. 최근에는 **NVIDIA Topology Scheduler Plugin**이 Kubernetes Scheduler Extender로 구현되어 실제 NVLink 토폴로지를 기반으로 GPU 할당을 최적화한다.

---

### Q15. 대규모 LLM 학습 클러스터에서 발생하는 네트워크 병목의 원인과 DCQCN(Data Center Quantized Congestion Notification) 메커니즘을 설명하라.

**답변:** LLM 학습에서 All-Reduce 연산은 모든 GPU가 동시에 그라디언트를 교환하는 동기적 집합 통신이다. 이 과정에서 수십~수백 개의 GPU가 동시에 특정 스위치 포트로 트래픽을 보내면 일시적인 마이크로버스트(Microburst)가 발생하고, 스위치 버퍼가 넘쳐 패킷 드롭이 발생한다. RDMA 환경에서 패킷 드롭은 Go-Back-N 재전송을 유발하여 전체 통신 진행이 멈추는 심각한 성능 저하를 초래한다. DCQCN은 이를 방지하기 위해 ECN(Explicit Congestion Notification), PFC(Priority Flow Control), 전송률 감소를 결합한 congestion control 알고리즘이다. 스위치가 큐 점유율이 임계값을 넘으면 패킷에 ECN 마킹을 추가하고, 수신 NIC가 ECN을 탐지하면 CNP(Congestion Notification Packet)를 송신자에게 전송한다. 송신 NIC의 RNI(Rate controller)는 CNP를 받으면 전송률을 단계적으로 감소시키고, 혼잡이 해소되면 점차 전송률을 회복한다. PFC(802.1Qbb)는 비상 수단으로, 스위치 버퍼가 임박하면 상류 장치에 Pause 프레임을 보내 일시 정지시킨다. 단, PFC는 Head-of-Line Blocking 문제를 일으킬 수 있어 세밀한 튜닝이 필요하다.

---

## [심화 Kubernetes 문제]

### Q16. kube-apiserver와 노드 사이의 통신에서 발생하는 인증서 교체(Certificate Rotation) 과정을 설명하고, 이 과정이 네트워크 정책에 미치는 영향을 논하라.

**답변:** Kubernetes에서 모든 컴포넌트 간 통신은 TLS로 암호화되며, kubelet의 클라이언트 인증서가 만료되기 전에 자동으로 교체되어야 한다. kubelet의 **TLS Bootstrapping** 과정은 노드가 처음 참가할 때 bootstrap token을 사용해 초기 인증서를 발급받고, 이후 **Certificate Signing Request(CSR)** API를 통해 정기적으로 갱신한다. kube-controller-manager의 `--cluster-signing-duration` 플래그가 인증서 유효 기간을 결정하며, `rotateCertificates: true` kubelet 설정으로 자동 교체가 활성화된다. 인증서 교체 중 kubelet이 잠깐 kube-apiserver와의 연결을 끊고 재연결할 때, CNI 플러그인이 Node 오브젝트의 준비 상태를 어떻게 해석하느냐에 따라 해당 노드의 Pod에 대한 NetworkPolicy 적용이 영향을 받을 수 있다. Calico의 경우 Felix 에이전트가 kube-apiserver와의 연결이 끊기면 기존 iptables 규칙을 유지하므로 트래픽에 영향이 없다. 그러나 Cilium의 경우 etcd가 직접적인 상태 저장소이고 kube-apiserver 연결 없이도 로컬 eBPF 맵을 유지하므로 정책 시행이 중단되지 않는다.

---

### Q17. Kubernetes Service의 ExternalTrafficPolicy: Local 설정이 레이턴시와 가용성에 미치는 영향을 분석하라.

**답변:** 기본값인 `externalTrafficPolicy: Cluster`에서 NodePort로 들어온 트래픽은 어떤 노드든 수신 가능하고, kube-proxy가 DNAT를 통해 다른 노드의 Pod로 트래픽을 전달할 수 있다. 이 과정에서 소스 IP가 SNAT되어 애플리케이션이 원본 클라이언트 IP를 잃게 된다. `externalTrafficPolicy: Local`로 설정하면 수신 노드에 해당 서비스의 Pod가 없을 경우 트래픽을 드롭하고, Pod가 있는 노드에서만 트래픽을 받아 SNAT 없이 원본 IP를 보존한다. 레이턴시 측면에서는 추가 홉이 없어 유리하지만, Pod 분포가 불균일하면 노드 간 부하가 불균형해진다. 예를 들어 한 노드에 Pod가 2개, 다른 노드에 1개 있다면 로드밸런서 기준으로는 각 노드에 1/3 트래픽이 가지만 실제 Pod 입장에서는 1:2 비율로 불균형이 발생한다. 이를 해결하기 위해 **Pod Topology Spread Constraints**로 Pod를 노드 전체에 균일하게 분산시키거나, NLB의 **cross-zone load balancing**을 비활성화하고 AZ별 Pod 수를 균일하게 유지하는 접근이 필요하다.

---

### Q18. Kubernetes 환경에서 mTLS를 Istio 없이 구현하는 방법과 그 한계를 논하라.

**답변:** Istio 없이 mTLS를 구현하는 방법으로는 애플리케이션 수준 TLS, SPIFFE/SPIRE를 통한 워크로드 ID 발급, 그리고 Linkerd의 경량 사이드카 방식이 있다. **SPIFFE(Secure Production Identity Framework for Everyone)**는 워크로드 ID 표준을 정의하고, **SPIRE(SPIFFE Runtime Environment)**는 SVID(SPIFFE Verifiable Identity Document) 형태의 X.509 인증서나 JWT 토큰을 각 워크로드에 발급한다. SPIRE Agent가 각 노드에서 Pod의 신원을 증명하는 Attestor 플러그인(Kubernetes Workload Attestor)을 통해 서비스 어카운트, 네임스페이스, 파드 레이블 기반으로 ID를 확인하고 SVID를 발급한다. 애플리케이션이 SPIRE Agent로부터 SVID를 받아 직접 TLS 핸드셰이크에 사용하면 사이드카 없이 mTLS가 가능하다. 한계는 모든 애플리케이션이 SPIRE API를 직접 호출하도록 코드를 수정해야 하거나, 언어별 SDK를 통합해야 한다는 점이다. 또한 Istio가 제공하는 L7 트래픽 관리(재시도, 타임아웃, 서킷 브레이커)는 별도 구현이 필요하다.

---

### Q19. eBPF를 활용하여 커널을 수정하지 않고 커스텀 네트워크 정책을 구현하는 방법을 설명하라.

**답변:** eBPF 프로그램은 BPF 검증기(Verifier)가 안전성을 검증한 후 JIT 컴파일되어 커널에 로드된다. 커스텀 네트워크 정책 구현의 첫 단계는 올바른 BPF Hook 선택이다. 인그레스/이그레스 필터링에는 TC(Traffic Control) Hook이 적합하며, 소켓 수준의 정책에는 `BPF_PROG_TYPE_SOCK_OPS`와 `BPF_PROG_TYPE_SK_MSG`를 사용한다. BPF 맵(HashMap, LRU HashMap, LPM Trie 등)을 통해 허용/차단 규칙을 커널과 사용자 공간 간에 공유한다. 예를 들어 IP 기반 ACL을 구현하려면 LPM Trie 맵에 허용 CIDR 목록을 적재하고, TC Hook에서 패킷의 목적지 IP를 룩업하여 DROP 또는 PASS 판정을 내린다. Go 언어에서는 **cilium/ebpf** 라이브러리나 **libbpf-go**를 사용하면 eBPF 프로그램 로딩, 맵 조작, 이벤트 처리를 편리하게 구현할 수 있다. CO-RE(Compile Once, Run Everywhere) 기술을 사용하면 BTF(BPF Type Format) 정보를 활용하여 커널 버전에 독립적인 eBPF 바이너리를 배포할 수 있다.

---

### Q20. Kubernetes에서 대규모 클러스터(5,000노드 이상)의 네트워크 컨트롤 플레인 확장성 문제와 해결 방안을 논하라.

**답변:** 대규모 Kubernetes 클러스터에서 네트워크 컨트롤 플레인의 확장성 병목은 주로 세 곳에서 발생한다. 첫째, **Endpoint 오브젝트의 크기 폭발**이다. 단일 서비스에 수천 개의 Pod가 있으면 Endpoints 오브젝트의 크기가 etcd 1MB 제한에 근접하며, 모든 엔드포인트 변경이 전체 Endpoints 오브젝트를 노드 전체에 재전송하게 된다. **EndpointSlice**는 이를 해결하기 위해 각 슬라이스에 최대 100개의 엔드포인트를 담고, 변경된 슬라이스만 전송함으로써 API 서버와 etcd 부하를 크게 줄인다. 둘째, **kube-proxy iptables 업데이트 지연**이다. 수천 개의 서비스와 수만 개의 엔드포인트가 있는 환경에서 단일 엔드포인트 변경이 전체 iptables를 재생성하게 되어 수십 초의 지연이 발생할 수 있다. IPVS 모드나 Cilium eBPF로 전환하면 증분 업데이트(Incremental Update)가 가능하여 지연을 밀리초 수준으로 낮출 수 있다. 셋째, **CNI IPAM의 응답 지연**이다. 고밀도 Pod 배포 시 IPAM의 IP 할당 속도가 Pod 시작 속도를 제한할 수 있다. Cilium의 경우 노드 로컬 IPAM 모드에서 kube-apiserver 없이 노드가 독립적으로 IP를 관리하여 이 병목을 해소한다.

---

## [보안/Zero Trust 계열]

### Q21. 클라우드 네트워크에서 Zero Trust 아키텍처를 구현할 때 SPIFFE/SPIRE, mTLS, OPA(Open Policy Agent)를 통합하는 방법을 설명하라.

**답변:** Zero Trust의 핵심 원칙은 네트워크 위치(같은 VPC, 같은 서브넷)가 신뢰의 기준이 되어서는 안 되며, 모든 요청이 명시적으로 인증·인가되어야 한다는 것이다. SPIFFE/SPIRE는 워크로드 ID 레이어를 담당한다. 각 마이크로서비스는 SPIFFE ID(URI 형식: spiffe://trust-domain/namespace/service-name)를 가지며, SPIRE가 발급한 SVID(X.509 인증서 또는 JWT)를 통해 자신의 ID를 증명한다. mTLS는 서비스 간 통신에서 양방향 인증을 강제하여 "A가 B를 신뢰하는 것"과 "B가 A를 신뢰하는 것"을 모두 암호학적으로 보장한다. OPA(Open Policy Agent)는 인가(Authorization) 레이어로, Envoy의 External Authorization Filter를 통해 각 요청의 SPIFFE ID, HTTP 경로, 메서드, 헤더를 입력으로 받아 Rego 정책으로 허용/차단을 결정한다. 이 세 계층의 통합은 네트워크 방화벽 규칙 없이도 워크로드 수준의 세밀한 접근 제어를 구현한다. AWS에서는 이 패턴을 IAM Roles for Service Accounts(IRSA), EKS Pod Identity, Security Group for Pods와 결합하여 클라우드 IAM과 워크로드 ID를 통합할 수 있다.

---

### Q22. 클라우드 네트워크에서 데이터 유출(Data Exfiltration) 방지 아키텍처를 설계하라.

**답변:** 데이터 유출 방지는 다계층 방어가 필요하다. 첫 번째 방어선은 **네트워크 레벨 이그레스 제어**다. AWS에서는 모든 아웃바운드 트래픽이 NAT Gateway → AWS Network Firewall → 허용 목록 도메인만 허용 되도록 설계한다. Network Firewall의 도메인 기반 필터링(Firewall Manager + FQDN 필터링 규칙)을 통해 승인된 외부 서비스(특정 SaaS API 도메인)만 허용하고 나머지를 차단한다. 두 번째는 **DNS 기반 제어**다. 모든 DNS 쿼리를 Route 53 Resolver로 강제하고, Route 53 Resolver DNS Firewall을 통해 악성 도메인이나 데이터 유출에 사용되는 DNS 터널링 도메인을 차단한다. 세 번째는 **DLP(Data Loss Prevention) 레이어**다. 클라우드 접근 보안 브로커(CASB) 또는 인라인 SSL 검사(SSL Inspection)를 통해 민감 데이터 패턴(정규 표현식 기반 신용카드, 개인정보, 내부 코드 등)을 탐지한다. 네 번째는 **VPC Endpoint를 통한 서비스 잠금**이다. S3, DynamoDB 등 데이터 저장소로의 접근은 VPC Endpoint를 통해서만 허용하고, Endpoint Policy로 특정 계정의 특정 버킷만 접근 가능하도록 제한하여 "confused deputy" 공격과 크로스 계정 데이터 유출을 방지한다.

---

## [고급 아키텍처/설계 계열]

### Q23. 클라우드 네이티브 환경에서 BGP Anycast를 사용한 글로벌 로드밸런싱 시스템을 설계하라.

**답변:** 글로벌 BGP Anycast 설계의 핵심은 동일 IP 프리픽스를 여러 지역에서 광고하는 것이다. 온프레미스 데이터센터나 클라우드 엣지에서 BGP로 서비스 IP(/32 또는 /128)를 업스트림 ISP에 광고하면, 클라이언트의 ISP 라우터는 BGP 경로 선택 알고리즘(AS Path 길이, IGP 비용 등)에 따라 가장 가까운 광고 지점으로 트래픽을 보낸다. 클라우드에서 이를 구현하는 실용적 방법은 Cloudflare Spectrum이나 자체 엣지 PoP 운영이다. Kubernetes 환경에서 MetalLB를 L3 모드로 사용하면 LoadBalancer 서비스 IP를 BGP로 업스트림 라우터에 광고할 수 있다. 단일 클러스터 내에서 여러 노드가 동일 서비스 IP를 광고하면 ECMP(Equal-Cost Multi-Path) 로드밸런싱이 적용된다. 이 설계의 도전 과제는 BGP 세션 플랩(Flap) 시 수렴 시간 동안의 서비스 중단, ECMP의 연결 해싱 일관성 문제(노드 추가/제거 시 기존 연결의 재해싱)가 있다. 연결 일관성은 Maglev 해싱 알고리즘을 사용하여 노드 변경 시 최소한의 연결만 재해싱되도록 개선할 수 있으며, Cilium과 MetalLB 모두 이를 지원한다.

---

### Q24. Kubernetes 클러스터 간 서비스 연결에서 East-West 트래픽과 North-South 트래픽을 분리하여 최적화하는 아키텍처를 설계하라.

**답변:** East-West 트래픽(서비스 간 내부 통신)과 North-South 트래픽(외부 클라이언트 → 서비스)은 요구사항이 다르므로 분리된 경로로 처리해야 한다. East-West를 위해서는 서비스 메시(Istio 또는 Cilium Cluster Mesh)를 통해 클러스터 간 서비스 디스커버리와 mTLS를 구현한다. Cilium Cluster Mesh에서는 `global` 서비스 어노테이션을 통해 특정 서비스를 멀티 클러스터에 걸쳐 노출하고, 소비자가 서비스 이름으로 호출하면 Cilium이 가장 가까운 엔드포인트(로컬 클러스터 우선)로 라우팅한다. North-South를 위해서는 클러스터 앞에 글로벌 레이어(AWS Global Accelerator, Azure Front Door, 또는 Cloudflare)를 배치하여 지리적 라우팅, 헬스체크, DDoS 방어를 처리한다. 이 두 경로의 중요한 분리 지점은 Ingress Gateway다. Istio Ingress Gateway는 North-South 트래픽을 처리하고, 클러스터 내부와 클러스터 간의 East-West는 Istio의 내부 서비스 레지스트리와 사이드카 메시가 처리한다. 네트워크 정책 측면에서는 Ingress Controller가 위치한 네임스페이스만 외부에서 인바운드를 받을 수 있도록 NetworkPolicy를 설정하고, 모든 East-West는 레이블 셀렉터 기반 정책으로 세밀하게 제어한다.

---

### Q25. IPv4 고갈 환경에서 Kubernetes 클러스터의 IPv6 전환(Dual-Stack 및 IPv6-only) 전략을 설명하라.

**답변:** Kubernetes의 Dual-Stack 지원은 v1.21부터 GA되어, 각 Pod와 서비스가 IPv4와 IPv6 주소를 동시에 가질 수 있다. Dual-Stack 전환의 첫 단계는 VPC/VNet 수준에서 IPv6 CIDR을 할당하는 것이다. AWS VPC는 /56 IPv6 CIDR을 서브넷에 분배하여 /64 서브넷으로 사용한다. EKS에서 IPv6 모드를 활성화하면 Pod는 IPv6 주소만 받고 IPv4와의 통신은 NAT64/DNS64를 통해 처리된다. AWS는 EC2 인스턴스에 NAT64 기능을 내장하여 IPv6-only Pod가 IPv4 서비스에 접근할 때 6-to-4 변환을 투명하게 처리한다. 마이그레이션 전략에서 중요한 것은 애플리케이션의 IPv6 지원 여부 확인이다. 하드코딩된 IPv4 주소, IPv4-only 소켓 바인딩(`0.0.0.0` 대신 `::` 사용 필요) 등을 점검해야 한다. 완전한 IPv6-only 환경에서는 서비스 메시의 mTLS 인증서에 IPv6 SAN을 포함시키고, 네트워크 모니터링 도구와 보안 도구가 IPv6 트래픽을 올바르게 처리하는지 검증이 필요하다.

---

## [Tesla/자동차/엣지 계열]

### Q26. 엣지 컴퓨팅 환경에서 Kubernetes 클러스터(예: 차량 내 인포테인먼트 또는 Tesla Autopilot과 유사한 시스템)의 네트워크 격리와 보안 설계를 논하라.

**답변:** 차량 또는 엣지 디바이스에서의 Kubernetes 네트워크 설계는 클라우드와 달리 **제한된 리소스, 신뢰할 수 없는 물리 환경, 간헐적 클라우드 연결**을 전제로 한다. K3s나 MicroK8s 같은 경량 Kubernetes 배포판을 사용하며, CNI로는 Flannel(경량)이나 Cilium(보안 필요 시) 중 선택한다. 네트워크 격리 설계에서 핵심은 **Air-gapped 환경에서도 동작하는 로컬 정책**이다. Cilium eBPF 기반 NetworkPolicy는 클라우드 컨트롤 플레인 연결 없이도 로컬 에이전트가 정책을 시행하므로, 클라우드 연결이 끊겨도 보안 정책이 유지된다. 차량 내 서비스들(센서 수집, ADAS 처리, 엔터테인먼트)은 별도 네임스페이스와 강력한 NetworkPolicy로 격리해야 하며, 안전 크리티컬 서비스(Autopilot)가 다른 서비스의 네트워크 장애에 영향받지 않도록 **QoS(Quality of Service) 클래스** 설정이 중요하다. OTA(Over-the-Air) 업데이트 채널은 별도의 암호화된 터널(WireGuard 또는 DTLS)로 격리하고, 업데이트 패키지의 서명 검증을 네트워크 수신 즉시 수행한다.

---

## [네트워크 트러블슈팅 계열]

### Q27. Kubernetes Pod 간 간헐적인 네트워크 패킷 드롭이 발생한다. 시스템적인 디버깅 방법론을 단계별로 설명하라.

**답변:** 간헐적 패킷 드롭 디버깅은 레이어별 체계적 접근이 필요하다. 1단계는 **문제 재현과 데이터 수집**이다. `ping`과 함께 `tcpdump`를 양쪽 Pod의 eth0 인터페이스에서 동시에 캡처하여 드롭이 발생하는 방향과 레이어를 확인한다. Cilium Hubble이 있다면 `hubble observe --from-pod <pod> --to-pod <pod>` 명령으로 Drop 이벤트와 Drop Reason을 확인한다. 2단계는 **커널 네트워크 스택 지표 확인**이다. `netstat -s`, `ss -s`, `/proc/net/snmp`에서 RetransSegs, TCPLostRetransmit, TCPTimeouts를 확인하고, `ethtool -S <interface>`로 NIC 드라이버 레벨의 rx/tx dropped 카운터를 확인한다. 3단계는 **conntrack 상태 확인**이다. `conntrack -S`로 conntrack 테이블 통계를 확인하고, `nf_conntrack_max`에 근접했는지 확인한다. 4단계는 **MTU 불일치 확인**이다. 오버레이 네트워크(VXLAN, Geneve)를 사용하는 경우 오버레이 헤더만큼 MTU가 줄어든다. `ip link show`로 각 인터페이스의 MTU를 확인하고, `ping -M do -s 1450 <target>`으로 PMTUD(Path MTU Discovery)가 올바르게 동작하는지 확인한다. 5단계는 **CPU RX 큐 불균형 확인**이다. `mpstat -P ALL 1`로 CPU별 소프트웨어 인터럽트 부하를 확인하고, RSS(Receive Side Scaling) 설정으로 인터럽트를 여러 CPU에 분산시킨다.

---

### Q28. AWS Direct Connect를 통한 온프레미스-클라우드 연결에서 비대칭 라우팅 문제가 발생하는 시나리오와 해결 방법을 설명하라.

**답변:** 비대칭 라우팅은 패킷이 A→B와 B→A 경로가 다를 때 발생한다. Direct Connect 환경에서 자주 발생하는 시나리오는 다음과 같다. 온프레미스에서 AWS로의 트래픽은 Direct Connect를 통해 VGW로 들어오지만, AWS에서 온프레미스로의 응답 트래픽은 VPN을 통해 나가는 경우다. 이는 Security Group의 상태 추적(Stateful)에서 문제가 되는데, SG는 인바운드 연결의 리턴 트래픽이 동일 인터페이스를 통해 나간다고 가정하기 때문이다. 반면 NACL은 Stateless이므로 다른 인터페이스를 통해 나가는 리턴 트래픽도 적절한 이그레스 규칙이 있으면 통과한다. VPC 수준에서 비대칭 라우팅은 일반적으로 허용되지만, Network Firewall 같은 상태 저장 인스펙션 어플라이언스를 경유하는 경우 연결 추적이 실패하여 세션이 끊긴다. 해결 방법은 BGP 정책(Local Preference, MED)을 조정하여 인바운드와 아웃바운드 경로를 동일하게 유지하거나, 방화벽을 비대칭 경로가 발생하지 않도록 토폴로지를 재설계하는 것이다. BGP AS Path Prepending을 통해 특정 Direct Connect 회선의 우선순위를 낮추는 방법도 효과적이다.

---

### Q29. Kubernetes에서 DNS 해석 실패로 인한 서비스 간 통신 장애를 진단하고 해결하는 방법을 설명하라.

**답변:** Kubernetes DNS 문제의 가장 빈번한 원인은 **CoreDNS 과부하**, **ndots 설정으로 인한 불필요한 쿼리**, **노드 DNS 설정 충돌**, 그리고 **UDP 패킷 드롭**이다. 1단계 진단: `kubectl run -it --rm debug --image=nicolaka/netshoot -- bash`로 임시 Pod를 생성하고 `dig kubernetes.default.svc.cluster.local @<CoreDNS ClusterIP>`로 DNS 응답 시간과 정상 응답 여부를 확인한다. 2단계: CoreDNS Pod 로그에서 `SERVFAIL`, `timeout` 패턴을 확인하고, `kubectl top pod -n kube-system`으로 CoreDNS CPU/메모리 사용률을 확인한다. 3단계: Pod의 `/etc/resolv.conf`를 확인하여 `ndots:5` 설정이 불필요한 도메인 검색(예: `redis` → `redis.namespace.svc.cluster.local`, `redis.svc.cluster.local`, `redis.cluster.local`, `redis.svc.domain`를 순서대로 시도)을 유발하는지 확인한다. 완전 도메인 이름(FQDN) 뒤에 `.`을 추가하거나, Pod 스펙에 `dnsConfig.options: [{name: ndots, value: "1"}]`로 ndots를 줄이면 불필요한 쿼리를 크게 줄일 수 있다. 4단계: CoreDNS의 자동 확장 설정(**cluster-proportional-autoscaler**)을 통해 클러스터 크기에 비례하여 CoreDNS 레플리카를 자동 조정하고, CoreDNS cache 플러그인의 TTL을 적절히 설정하여 업스트림 쿼리를 줄인다.

---

### Q30. 네트워크 성능 회귀(Performance Regression) 테스트 프레임워크를 클라우드 네이티브 환경에서 구축하는 방법을 설계하라.

**답변:** 네트워크 성능 회귀 테스트는 CI/CD 파이프라인에 통합되어 인프라 변경이 네트워크 성능에 미치는 영향을 자동으로 감지해야 한다. 프레임워크의 구성은 다음과 같다. **벤치마크 도구 계층**으로는 `iperf3`(TCP/UDP 처리량), `netperf`(레이턴시, 작은 패킷 성능), `sockperf`(레이턴시 분포, 지터), `k6` 또는 `wrk`(HTTP 레이어 성능), `DPDK testpmd`(패킷 처리 성능)를 워크로드 특성에 맞게 선택한다. **테스트 시나리오 계층**에서는 동일 노드 Pod 간, 다른 노드 Pod 간, AZ 간, 리전 간, Service ClusterIP를 통한, LoadBalancer를 통한 경우 등 다양한 네트워크 경로를 커버한다. **자동화 계층**에서는 Argo Workflows나 Tekton으로 테스트를 오케스트레이션하고, 결과를 Prometheus에 시계열로 저장하여 Grafana에서 기준선(Baseline)과 비교한다. **회귀 탐지 계층**에서는 통계적 방법(Z-score, Welch's t-test)으로 성능 저하가 통계적으로 유의미한지 판정하고, 특정 임계값(예: P99 레이턴시 10% 이상 증가)을 초과하면 CI 파이프라인을 실패로 처리한다. 이 프레임워크는 Kubernetes 버전 업그레이드, CNI 플러그인 변경, 커널 업데이트, EC2 인스턴스 유형 변경 시 성능 변화를 객관적으로 측정하는 기준이 된다.

---

# 부록: 핵심 용어 정리 및 참고 자키텍처 패턴

## 자주 혼동되는 개념 정리

**SNAT vs MASQUERADE:** SNAT는 정적 소스 IP로 변환하고, MASQUERADE는 인터페이스의 현재 IP를 동적으로 사용한다. 클라우드 환경에서 Elastic IP가 고정된 경우 SNAT를, DHCP로 IP가 변동되는 경우 MASQUERADE가 적합하다.

**Overlay vs Underlay:** Underlay는 물리 네트워크 인프라(실제 스위치, 라우터, 물리 링크)이고, Overlay는 그 위에 논리적으로 생성된 가상 네트워크다. VXLAN, GRE, Geneve는 모두 오버레이 프로토콜이다.

**L4 vs L7 Load Balancing:** L4 LB는 TCP/UDP 레벨에서 5-tuple 기반으로 라우팅하며 패킷 내용을 보지 않는다. L7 LB는 HTTP 헤더, URL 경로, 쿠키를 기반으로 콘텐츠 인식 라우팅을 수행한다. AWS에서 NLB는 L4, ALB는 L7이다.

**Control Plane vs Data Plane vs Management Plane:** Control Plane은 라우팅 결정 로직(BGP, OSPF, Kubernetes kube-controller-manager), Data Plane은 실제 패킷 포워딩(NIC, ASIC, eBPF, iptables), Management Plane은 설정 및 모니터링 인터페이스(AWS Console, kubectl, Ansible)다.

## 추천 심화 학습 자료

심화 학습을 위해서는 다음 자료들을 참고하기를 권장한다. AWS의 공식 "VPC Networking Workshop", Cilium의 공식 문서와 eBPF.io, Istio의 Envoy xDS API 문서, Brendan Gregg의 "Systems Performance" (BPF Performance Tools 포함), 그리고 IETF RFC 문서들(RFC 7348 VXLAN, RFC 4271 BGP, RFC 7950 YANG)이 이론적 기반을 강화하는 데 도움이 된다.
