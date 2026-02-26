# AWS · Azure · Kubernetes 리소스 & 비용 최적화
## 박사급 심층 기술 가이드 + 해외 대기업 면접 Q&A 35선

> 대상 독자: Senior Cloud / MLOps / LLMOps Engineer  
> 기준 시점: 2025년 상반기

---

## 목차

1. [이론적 토대 — 클라우드 경제학과 최적화 철학](#1-이론적-토대)
2. [AWS 리소스 최적화 심층 분석](#2-aws-리소스-최적화)
3. [Azure 리소스 최적화 심층 분석](#3-azure-리소스-최적화)
4. [Kubernetes 리소스 최적화 심층 분석](#4-kubernetes-리소스-최적화)
5. [비용 최적화 거버넌스 체계 (FinOps Framework)](#5-finops-거버넌스-체계)
6. [AI/ML·LLM 워크로드 특화 최적화](#6-aiml-llm-워크로드-최적화)
7. [멀티클라우드 비용 통합 아키텍처](#7-멀티클라우드-비용-통합-아키텍처)
8. [해외 대기업 면접 Q&A 35선](#8-면접-qa-35선)

---

## 1. 이론적 토대

### 1.1 클라우드 비용의 본질적 구조

클라우드 비용을 단순히 "청구서 절감"으로 접근하는 것은 피상적인 시각이다. 근본적으로 클라우드 비용은 **컴퓨트 자원 소비 함수(Resource Consumption Function)**로 표현할 수 있다.

```
Total Cost = Σ (Unit Price × Utilization × Time) + Data Transfer + API Calls + Support
```

이 함수에서 최적화 가능한 변수는 세 가지 축으로 나뉜다. 첫째, **Unit Price 최적화**는 Reserved Instance, Savings Plans, Spot 활용, 협상력(EDP/MACC) 등을 통해 단가 자체를 낮추는 전략이다. 둘째, **Utilization 최적화**는 Right-sizing, Auto-scaling, 스케줄링을 통해 실제 소비량을 워크로드 필요에 맞추는 전략이다. 셋째, **Architectural 최적화**는 워크로드 자체를 재설계하여 Time 축(동작 시간)과 소비 패턴을 바꾸는 전략이다. 대부분의 조직이 첫 번째 축에만 집중하고 두 번째와 세 번째 축을 소홀히 하는 경향이 있는데, 실질적인 20-40% 절감은 후자 두 축에서 나온다.

### 1.2 수요-공급 미스매치 이론

클라우드 환경에서 낭비는 본질적으로 **프로비저닝된 용량(Provisioned Capacity)**과 **실제 수요(Actual Demand)** 사이의 미스매치에서 발생한다. 이를 두 가지 유형으로 구분할 수 있다.

**오버 프로비저닝(Over-provisioning)**은 안전 마진을 과도하게 확보하거나, 피크 트래픽 기준으로 항시 용량을 유지하거나, 레거시 사이징 방법론을 그대로 적용하는 경우에 발생한다. 업계 평균 클라우드 VM의 CPU 활용률은 7-15%에 불과하며, 이는 온프레미스 서버 설계 관행이 클라우드로 그대로 이식되기 때문이다.

**언더 프로비저닝(Under-provisioning)**은 덜 주목받지만 비용 구조를 왜곡시킨다. 스로틀링과 재시도로 인한 API 비용 증가, 느린 응답으로 인한 SLA 위반 패널티, 그리고 불필요한 캐시 레이어 추가 비용이 발생한다.

### 1.3 최적화의 4가지 레버 (4-Lever Framework)

업계에서 실증된 최적화 프레임워크는 다음 네 가지 레버를 순서대로 적용하는 것이다.

**Eliminate(제거)**는 가장 높은 ROI를 제공한다. 사용하지 않는 리소스, 고아(orphan) 리소스, 중복 서비스를 발견하고 제거하는 것만으로도 일반적으로 총 비용의 10-20%를 즉시 절감할 수 있다.

**Rightsize(적정화)**는 실제 워크로드 특성에 맞게 리소스 유형과 크기를 조정하는 것이다. 이는 단순히 작게 만드는 것이 아니라, CPU-최적화 vs 메모리-최적화 vs 가속기-최적화 인스턴스 유형 선택까지 포함한다.

**Automate(자동화)**는 수요 변화에 따라 리소스를 탄력적으로 조정하는 것으로, 정적 환경 대비 30-50%의 비용 절감이 가능하다.

**Purchase Optimally(최적 구매)**는 위 세 단계가 적용된 후 안정화된 기저 수요에 대해 약정 기반 할인을 적용하는 것이다. 순서를 지키지 않으면 과도한 RI를 구매하여 오히려 낭비가 발생할 수 있다.

---

## 2. AWS 리소스 최적화

### 2.1 컴퓨트 최적화 심층 분석

#### EC2 인스턴스 패밀리 전략

AWS EC2 인스턴스 패밀리 선택은 단순한 크기 문제가 아니라 **아키텍처 특성 매핑** 문제다. 각 워크로드 유형별 최적 인스턴스 패밀리를 이해하는 것이 핵심이다.

Graviton3/4 기반 인스턴스(C7g, M7g, R7g 등)는 같은 세대 x86 대비 평균 20-40% 낮은 비용으로 동등하거나 우수한 성능을 제공한다. 이는 단순히 ARM 아키텍처의 에너지 효율성 때문만이 아니라, AWS가 자체 실리콘을 설계하면서 클라우드 워크로드에 특화된 최적화를 적용했기 때문이다. 컨테이너 기반 마이크로서비스, Java 애플리케이션, 웹 서버 등 대부분의 범용 워크로드에서 Graviton 마이그레이션은 가장 빠른 비용 절감 수단이다.

AMD EPYC 기반 인스턴스(M6a, C6a, R6a)는 Intel 대비 약 10% 저렴하면서도 동등한 성능을 제공하며, x86 바이너리 호환성이 필요한 경우의 중간 단계 최적화 옵션이다.

**스팟 인스턴스 아키텍처 패턴**은 단순히 저렴한 옵션을 선택하는 것이 아니라, 인터럽트 허용 아키텍처(Interruption-tolerant Architecture)를 설계하는 것이다. 이를 위해서는 스테이트리스 워크로드 설계, 체크포인팅 메커니즘 구현, 2분 인터럽트 경고를 활용한 그레이스풀 드레인, 그리고 여러 인스턴스 패밀리와 가용 영역에 걸친 다양화(diversification) 전략이 필요하다.

```yaml
# EC2 Fleet 다양화 예시 (CloudFormation 일부)
EC2Fleet:
  Type: AWS::EC2::EC2Fleet
  Properties:
    LaunchTemplateConfigs:
      - LaunchTemplateSpecification:
          LaunchTemplateId: !Ref LT
          Version: "$Latest"
        Overrides:
          - InstanceType: m7g.xlarge
            WeightedCapacity: 4
          - InstanceType: m7g.2xlarge
            WeightedCapacity: 8
          - InstanceType: c7g.xlarge
            WeightedCapacity: 4
          - InstanceType: c7g.2xlarge
            WeightedCapacity: 8
    TargetCapacitySpecification:
      TotalTargetCapacity: 100
      OnDemandTargetCapacity: 20      # 기저 20%는 On-Demand
      SpotTargetCapacity: 80          # 나머지 80%는 Spot
      DefaultTargetCapacityType: spot
    SpotOptions:
      AllocationStrategy: price-capacity-optimized  # 2023년 신규 전략
      InstanceInterruptionBehavior: terminate
```

`price-capacity-optimized` 전략은 단순 `lowestPrice`보다 인터럽트율을 크게 낮추면서도 좋은 가격을 유지한다. AWS 내부 데이터에 따르면 이 전략을 사용할 때 스팟 인터럽션율이 `lowestPrice` 대비 평균 3배 낮다.

#### AWS Savings Plans 심층 분석

Savings Plans는 세 가지 유형이 있으며, 선택 전략이 매우 중요하다.

**Compute Savings Plans**는 가장 유연하다. EC2 인스턴스 패밀리, 리전, OS, 테넌시, 컴퓨트 유형(EC2, Lambda, Fargate)에 관계없이 적용되며 최대 66% 할인을 제공한다. 불확실성이 높은 워크로드나 마이그레이션이 예상되는 환경에 적합하다.

**EC2 Instance Savings Plans**는 특정 인스턴스 패밀리와 리전에 고정되지만 최대 72%의 더 큰 할인을 제공한다. 장기적으로 안정적인 워크로드에 적합하다.

**SageMaker Savings Plans**는 ML 트레이닝 및 추론 워크로드를 위한 것으로 최대 64% 할인이다.

최적 구매 전략은 **커버리지 분석**에서 시작한다. 지난 3개월의 온디맨드 사용량 중 안정적인 기저(base) 수요를 식별하고, 그 80-85% 수준에서 Savings Plans를 구매하는 것이 일반적인 권장사항이다. 100% 커버리지를 목표로 하면 사용 패턴 변화 시 유연성을 잃게 된다.

```python
# AWS Cost Explorer API를 활용한 Savings Plans 최적화 분석
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce', region_name='us-east-1')

def analyze_savings_plans_coverage():
    """지난 30일 Savings Plans 커버리지 분석"""
    end = datetime.now().strftime('%Y-%m-%d')
    start = (datetime.now() - timedelta(days=30)).strftime('%Y-%m-%d')
    
    response = ce.get_savings_plans_coverage(
        TimePeriod={'Start': start, 'End': end},
        GroupBy=[{'Type': 'DIMENSION', 'Key': 'SERVICE'}],
        Granularity='MONTHLY'
    )
    
    for group in response['SavingsPlansCoverages']:
        svc = group['Attributes'].get('SERVICE', 'Unknown')
        coverage = group['Coverage']['CoveragePercentage']
        on_demand_cost = group['Coverage']['OnDemandCost']
        
        print(f"서비스: {svc}")
        print(f"  SP 커버리지: {coverage:.1f}%")
        print(f"  미커버 On-Demand 비용: ${float(on_demand_cost):,.2f}")
        
        # 커버리지 70% 미만이면 추가 SP 구매 권고
        if float(coverage) < 70:
            print(f"  ⚠️ 추가 Savings Plans 구매 권고")

def get_rightsizing_recommendations():
    """EC2 Right-sizing 권고사항 조회"""
    response = ce.get_rightsizing_recommendation(
        Service='AmazonEC2',
        Configuration={
            'RecommendationTarget': 'CROSS_INSTANCE_FAMILY',  # 패밀리 간 변경도 허용
            'BenefitsConsidered': True
        }
    )
    
    total_savings = 0
    for rec in response['RightsizingRecommendations']:
        if rec['RightsizingType'] == 'MODIFY':
            current = rec['CurrentInstance']
            target = rec['ModifyRecommendationDetail']['TargetInstances'][0]
            monthly_savings = float(target['EstimatedMonthlySavings'])
            total_savings += monthly_savings
            
            print(f"인스턴스: {current['ResourceId']}")
            print(f"  현재: {current['InstanceType']} → 권고: {target['InstanceType']}")
            print(f"  월 절감액: ${monthly_savings:,.2f}")
            print(f"  CPU 평균 활용률: {current['UtilizationMetrics']['CPU']['Average']:.1f}%")
    
    print(f"\n총 월 절감 가능액: ${total_savings:,.2f}")
    return total_savings

analyze_savings_plans_coverage()
get_rightsizing_recommendations()
```

### 2.2 스토리지 최적화

#### S3 지능형 계층화와 수명 주기 정책

S3 비용 최적화의 핵심은 **데이터 온도(Data Temperature)** 모델로 접근하는 것이다. 데이터는 생성 초기에는 자주 접근되다가 시간이 지남에 따라 접근 빈도가 감소하는 패턴을 보이며, 이를 자동으로 감지하고 적절한 스토리지 클래스로 전환하는 것이 S3 Intelligent-Tiering의 핵심 가치다.

그러나 Intelligent-Tiering이 항상 최선은 아니다. 객체당 모니터링 비용($0.0025/1,000 objects)이 발생하므로, 128KB 미만의 소형 객체나 접근 패턴이 명확히 예측 가능한 데이터에는 명시적 수명 주기 정책이 더 효율적이다.

```json
{
  "Rules": [
    {
      "ID": "ML-Training-Data-Lifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "ml-data/training/" },
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER_IR" },
        { "Days": 180, "StorageClass": "GLACIER" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 7,  "StorageClass": "STANDARD_IA" },
        { "NoncurrentDays": 30, "StorageClass": "GLACIER" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 }
    }
  ]
}
```

#### EBS 최적화 전략

EBS는 클라우드 스토리지 낭비의 주요 원인 중 하나다. 주의해야 할 패턴은 다음과 같다.

gp2 → gp3 마이그레이션은 가장 빠른 EBS 비용 절감 수단이다. gp3는 gp2 대비 20% 저렴하면서 기본 IOPS(3,000)와 처리량(125 MB/s)이 더 높다. gp2는 IOPS가 볼륨 크기에 연동되어 있어 고성능을 위해 불필요하게 큰 볼륨을 구매하게 만들지만, gp3는 크기와 IOPS를 독립적으로 설정할 수 있다.

```bash
# 계정 내 모든 gp2 볼륨을 gp3로 마이그레이션하는 스크립트
#!/bin/bash

# 모든 리전에서 gp2 볼륨 조회
for region in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
    echo "=== Region: $region ==="
    
    volumes=$(aws ec2 describe-volumes \
        --region $region \
        --filters "Name=volume-type,Values=gp2" \
        --query 'Volumes[?State==`available` || State==`in-use`].[VolumeId,Size,Iops,State]' \
        --output json)
    
    echo "$volumes" | jq -c '.[]' | while read -r vol; do
        vol_id=$(echo $vol | jq -r '.[0]')
        size=$(echo $vol | jq -r '.[1]')
        current_iops=$(echo $vol | jq -r '.[2]')
        
        # gp3 IOPS: 최대 16000, 최소 3000. gp2 IOPS = size * 3 (최소 100)
        # gp3로 전환 시 동일 성능을 위해 필요한 IOPS 계산
        required_iops=$((size * 3))
        target_iops=$(( required_iops > 3000 ? required_iops : 3000 ))
        target_iops=$(( target_iops > 16000 ? 16000 : target_iops ))
        
        echo "  Modifying $vol_id (${size}GB, ${current_iops} IOPS) → gp3 with ${target_iops} IOPS"
        
        aws ec2 modify-volume \
            --region $region \
            --volume-id $vol_id \
            --volume-type gp3 \
            --iops $target_iops \
            --throughput 125 \
            --no-cli-pager
    done
done
```

### 2.3 네트워크 비용 최적화

네트워크 비용은 종종 간과되지만 전체 AWS 청구의 15-25%를 차지할 수 있다. 핵심 최적화 영역은 다음과 같다.

**VPC Endpoint 전략**은 S3, DynamoDB, ECR 등으로의 트래픽이 인터넷 게이트웨이나 NAT Gateway를 거치지 않도록 한다. NAT Gateway 비용($0.045/GB)은 대용량 트래픽 환경에서 급격히 증가하며, Gateway Endpoint(무료)를 활용하면 이를 완전히 회피할 수 있다.

**데이터 전송 비용 아키텍처**는 AZ 간 트래픽($0.01/GB 양방향), 리전 간 트래픽($0.02-0.09/GB), 인터넷 이그레스($0.09/GB first 10TB)를 구분하여 아키텍처 수준에서 트래픽 경로를 최적화해야 한다. 특히 Kubernetes 환경에서는 파드 간 통신이 AZ를 넘나드는 경우가 빈번하여 예상치 못한 비용이 발생한다.

---

## 3. Azure 리소스 최적화

### 3.1 Azure 고유 비용 구조 이해

Azure의 비용 구조는 AWS와 유사하지만 몇 가지 고유한 특성이 있다. **Azure Hybrid Benefit(AHB)**은 온프레미스 Windows Server 또는 SQL Server 라이선스를 Azure에서 재사용할 수 있게 하여 최대 85%의 비용 절감을 가능하게 한다. 이는 Microsoft 생태계에서 이전하는 조직에게 강력한 레버다.

**Microsoft Azure Consumption Commitment(MACC)**는 AWS의 EDP에 해당하며, 일정 소비량을 약정하는 대가로 추가 할인과 지원을 받는 계약이다. 연간 Azure 지출이 $500K를 초과하는 조직은 MACC 협상을 적극적으로 검토해야 한다.

### 3.2 Azure Reserved Instances와 Savings Plans

Azure Reserved Instances(RI)는 1년 또는 3년 약정으로 Pay-as-you-go 대비 최대 72% 할인을 제공한다. Azure RI의 차별화 포인트는 **Instance Size Flexibility**인데, 같은 인스턴스 패밀리 내에서 크기 변경 시에도 RI 혜택이 비례적으로 적용된다. 예를 들어 D4s_v5 RI를 구매하면 D2s_v5 두 대에도 적용된다.

Azure Savings Plans는 Compute와 Machine Learning 두 종류가 있으며, 시간당 약정 금액 기반으로 할인이 적용된다. AWS와 달리 Azure Savings Plans는 구독, 리전, 인스턴스 시리즈에 관계없이 유연하게 적용된다.

```python
# Azure Cost Management API를 활용한 비용 분석
from azure.identity import DefaultAzureCredential
from azure.mgmt.costmanagement import CostManagementClient
from azure.mgmt.costmanagement.models import (
    QueryDefinition, QueryTimePeriod, QueryDataset,
    QueryAggregation, QueryGrouping, TimeframeType
)
from datetime import datetime, timedelta

def analyze_azure_costs(subscription_id: str):
    """Azure 서비스별 비용 분석 및 최적화 기회 식별"""
    credential = DefaultAzureCredential()
    client = CostManagementClient(credential)
    
    scope = f"/subscriptions/{subscription_id}"
    
    # 지난 30일 서비스별 비용 집계
    end_date = datetime.now()
    start_date = end_date - timedelta(days=30)
    
    query = QueryDefinition(
        type="Usage",
        timeframe=TimeframeType.CUSTOM,
        time_period=QueryTimePeriod(
            from_property=start_date,
            to=end_date
        ),
        dataset=QueryDataset(
            granularity="None",
            aggregation={
                "totalCost": QueryAggregation(name="Cost", function="Sum"),
                "totalUsage": QueryAggregation(name="UsageQuantity", function="Sum")
            },
            grouping=[
                QueryGrouping(type="Dimension", name="ServiceName"),
                QueryGrouping(type="Dimension", name="ResourceLocation")
            ]
        )
    )
    
    result = client.query.usage(scope, query)
    
    # 결과 분석 및 최적화 기회 출력
    costs = {}
    for row in result.rows:
        service = row[2]  # ServiceName
        location = row[3]  # ResourceLocation
        cost = float(row[0])
        
        if service not in costs:
            costs[service] = 0
        costs[service] += cost
    
    # 비용 기준 상위 10개 서비스
    sorted_costs = sorted(costs.items(), key=lambda x: x[1], reverse=True)
    print("상위 10개 비용 발생 서비스:")
    for svc, cost in sorted_costs[:10]:
        print(f"  {svc}: ${cost:,.2f}/month")
    
    return sorted_costs

def check_underutilized_vms(subscription_id: str):
    """낮은 활용률 VM 식별 (Azure Advisor API 활용)"""
    from azure.mgmt.advisor import AdvisorManagementClient
    
    credential = DefaultAzureCredential()
    advisor_client = AdvisorManagementClient(credential, subscription_id)
    
    # Cost 카테고리의 권고사항 조회
    recommendations = advisor_client.recommendations.list(
        filter="Category eq 'Cost'"
    )
    
    rightsizing_recs = []
    for rec in recommendations:
        if 'virtual machine' in rec.short_description.problem.lower():
            rightsizing_recs.append({
                'resource': rec.resource_metadata.resource_id,
                'impact': rec.impact,
                'savings': rec.extended_properties.get('annualSavingsAmount', 'N/A')
            })
    
    return rightsizing_recs
```

### 3.3 Azure AKS 특화 최적화

**노드 풀 자동 프로비저닝(NAP)**은 AKS의 Karpenter 상당 기능으로, 파드 요구사항에 따라 최적의 VM 크기를 동적으로 선택한다. 기존의 클러스터 오토스케일러(CAS)가 미리 정의된 노드 풀 내에서만 스케일링하는 것과 달리, NAP는 VM SKU 자체를 동적으로 결정한다.

**Azure Spot Virtual Machines**는 AWS Spot과 유사하지만 최대 90%의 더 큰 할인이 가능하다. 단, 30초의 짧은 퇴거 경고가 AWS의 2분보다 짧으므로, 더 공격적인 체크포인팅 전략이 필요하다.

---

## 4. Kubernetes 리소스 최적화

### 4.1 리소스 Request/Limit의 이론적 배경

Kubernetes 스케줄링은 **Bin Packing 알고리즘**의 변형이다. 스케줄러는 파드의 resource request를 기반으로 노드에 파드를 배치하는데, 이때 request 값이 실제 사용량과 얼마나 차이 나는지가 노드 활용률을 결정한다.

중요한 개념 구분이 필요하다. **Request**는 스케줄링에 사용되는 예약 값이며, **Limit**은 실제 소비 한계값이다. 이 두 값의 관계에 따라 파드의 QoS 클래스가 결정된다.

```
Guaranteed: Request == Limit (가장 높은 우선순위, OOM 발생 시 마지막으로 종료)
Burstable:  Request < Limit  (중간 우선순위)
BestEffort: Request 미설정  (가장 낮은 우선순위, 가장 먼저 종료)
```

QoS 클래스는 단순한 분류가 아니라 **OOM Killer 우선순위**와 직결된다. 비용 절감을 위해 limit을 과도하게 낮추면 OOM Killer가 자주 발생하는 불안정한 클러스터가 된다. 이상적인 설정은 실제 p99 사용량을 측정한 후, request는 p50 수준으로, limit은 p99.9 수준으로 설정하는 것이다.

### 4.2 VPA (Vertical Pod Autoscaler) 심층 분석

VPA는 단순히 리소스 request를 자동으로 조정하는 도구가 아니라, 워크로드의 리소스 프로파일을 지속적으로 학습하는 **히스토그램 기반 추천 엔진**이다.

VPA의 내부 동작을 이해하는 것이 올바른 활용의 핵심이다. VPA Recommender는 Metrics Server에서 수집된 CPU/메모리 사용량을 지수 감쇠 히스토그램(Exponential Decay Histogram)으로 관리하며, 최근 데이터에 더 높은 가중치를 부여한다. 이를 통해 워크로드의 계절성(Seasonality)을 반영한 추천을 생성한다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ml-inference-server-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: ml-inference-server
  updatePolicy:
    updateMode: "Auto"       # Off, Initial, Recreate, Auto 중 선택
  resourcePolicy:
    containerPolicies:
      - containerName: inference
        minAllowed:
          cpu: 500m
          memory: 1Gi
        maxAllowed:
          cpu: 8
          memory: 32Gi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits   # 또는 RequestsOnly
      - containerName: sidecar
        mode: "Off"    # 사이드카는 VPA 적용 제외
```

VPA의 가장 큰 한계는 **리소스 변경 시 파드 재시작**이 필요하다는 점이다. 이를 해결하기 위해 VPA In-Place Resource Update(Kubernetes 1.27+ alpha)가 도입되었으며, 이를 통해 재시작 없이 CPU 제한을 조정할 수 있게 되었다.

### 4.3 HPA와 KEDA의 고급 활용

기본 HPA는 CPU/메모리 메트릭만 지원하지만, 실제 워크로드에서는 더 다양한 스케일링 신호가 필요하다. KEDA(Kubernetes Event-driven Autoscaling)는 이를 해결하는 표준 솔루션이다.

**KEDA ScaledObject**를 활용하면 큐 깊이, 카프카 컨슈머 랙, Prometheus 메트릭 등 다양한 이벤트 소스를 기반으로 정밀한 스케일링이 가능하다.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: llm-inference-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: llm-inference-deployment
  pollingInterval: 15        # 메트릭 체크 주기 (초)
  cooldownPeriod: 300        # 스케일 다운 유예 시간 (초) - LLM 특성상 길게 설정
  minReplicaCount: 1         # 비용 절감: 0으로 설정하면 완전한 scale-to-zero 가능
  maxReplicaCount: 50
  advanced:
    restoreToOriginalReplicaCount: false
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 600  # 스케일 다운은 10분 안정화 후
          policies:
            - type: Percent
              value: 10
              periodSeconds: 60            # 1분당 최대 10%씩만 다운
        scaleUp:
          stabilizationWindowSeconds: 0   # 스케일 업은 즉시
          policies:
            - type: Pods
              value: 5
              periodSeconds: 30            # 30초마다 최대 5개 파드 추가
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: llm_queue_pending_requests
        threshold: "3"    # 파드당 처리 중인 요청이 3개 초과 시 스케일 업
        query: sum(llm_inference_queue_size) / sum(kube_deployment_status_replicas_ready{deployment="llm-inference-deployment"})
    - type: aws-sqs-queue
      authenticationRef:
        name: keda-aws-credentials
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/account/inference-queue
        queueLength: "5"
        awsRegion: us-east-1
```

### 4.4 Karpenter를 활용한 노드 최적화

Karpenter는 Kubernetes 클러스터 오토스케일러(CAS)를 대체하는 AWS의 오픈소스 노드 프로비저너로, 기존 CAS 대비 훨씬 정밀한 노드 최적화가 가능하다.

CAS와 Karpenter의 핵심 차이는 **의사결정 단위**에 있다. CAS는 미리 정의된 노드 그룹(ASG) 단위로 스케일링하며, 어떤 인스턴스 타입이 사용될지는 ASG 설정에 고정된다. Karpenter는 스케줄되지 못한 파드(Pending Pods)의 요구사항을 분석하여 가장 효율적인 인스턴스 타입을 실시간으로 선택한다.

```yaml
# Karpenter NodePool - 비용 최적화 설정 예시
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: cost-optimized-pool
spec:
  template:
    metadata:
      labels:
        nodepool: cost-optimized
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]   # spot 우선, 없으면 on-demand
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64", "amd64"]      # Graviton 우선 허용
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]         # 컴퓨트/범용/메모리 패밀리만
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]                   # 6세대 이상만 (비용 효율성)
        - key: karpenter.k8s.aws/instance-cpu
          operator: In
          values: ["4", "8", "16", "32"]  # 적정 크기 범위
      taints:
        - key: "spot-node"
          effect: PreferNoSchedule        # Spot 노드를 soft preference로
  disruption:
    consolidationPolicy: WhenUnderutilized  # 활용률 낮을 때 통합
    consolidateAfter: 30s
    expireAfter: 720h    # 30일 후 노드 갱신 (보안 패치)
  limits:
    cpu: 1000
    memory: 4000Gi

---
# Karpenter EC2NodeClass
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket    # 컨테이너 최적화 OS로 오버헤드 최소화
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "my-cluster"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "my-cluster"
  instanceStorePolicy: RAID0   # NVMe 인스턴스 스토어를 RAID0으로 활용 (I/O 최적화)
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
```

Karpenter의 **Consolidation(통합)** 기능은 활용률이 낮은 여러 노드의 워크로드를 더 적은 수의 노드로 재배치하여 빈 노드를 종료한다. 이는 수동 개입 없이 클러스터 비용을 지속적으로 최적화하는 강력한 메커니즘이다.

### 4.5 Namespace 수준 리소스 할당량과 LimitRange

클러스터 수준의 리소스 거버넌스는 **ResourceQuota**와 **LimitRange**의 조합으로 구현된다.

```yaml
# 팀별 네임스페이스 리소스 할당량
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-ml-quota
  namespace: team-ml
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 400Gi
    limits.cpu: "200"
    limits.memory: 800Gi
    requests.nvidia.com/gpu: "8"    # GPU 할당량 제한
    count/pods: "100"
    count/services: "20"
    count/persistentvolumeclaims: "20"
    requests.storage: 10Ti

---
# 기본 리소스 설정 강제 (Request 없는 파드 방지)
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-ml
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "16"
        memory: 64Gi
      min:
        cpu: 10m
        memory: 32Mi
      maxLimitRequestRatio:
        cpu: "10"       # limit이 request의 10배를 넘을 수 없음
        memory: "4"
```

---

## 5. FinOps 거버넌스 체계

### 5.1 FinOps 성숙도 모델

FinOps Foundation이 정의한 성숙도 모델은 세 단계로 구성된다.

**Crawl(탐색)** 단계는 가시성 확보가 목표다. 모든 리소스에 표준화된 태그를 적용하고, 팀/환경/프로젝트별 비용 분리가 가능한 상태를 만드는 것이다. 이 단계에서는 보통 40-60%의 리소스가 적절히 태깅되지 않아 비용 귀속이 불가능하다.

**Walk(실행)** 단계는 최적화 실행이 목표다. 비용 분석 결과를 기반으로 Right-sizing, RI/SP 구매, 스케줄링 자동화가 이루어진다. 이 단계에서 Unit Economics(요청당 비용, 사용자당 비용) 개념이 도입된다.

**Run(운영)** 단계는 예측적 비용 관리가 목표다. AI 기반 이상 감지, 비용 예측 모델, 엔지니어링 팀의 자율적 비용 최적화 문화가 정착된 상태다.

### 5.2 태그 전략과 비용 배분 모델

효과적인 비용 배분을 위한 표준 태그 체계를 설계하는 것은 FinOps 성공의 전제 조건이다.

```python
# 태그 정책 준수 검사 자동화 (AWS Config 룰 or Python 스크립트)
import boto3
from typing import Dict, List, Set

REQUIRED_TAGS = {
    'Environment': {'production', 'staging', 'development', 'sandbox'},
    'Team': None,          # 자유 형식
    'Project': None,
    'CostCenter': None,
    'Owner': None,
    'ManagedBy': {'terraform', 'cloudformation', 'manual', 'cdk'}
}

def audit_tag_compliance(resource_arns: List[str]) -> Dict:
    """리소스 태그 준수 현황 감사"""
    tagging = boto3.client('resourcegroupstaggingapi')
    
    report = {
        'compliant': [],
        'non_compliant': [],
        'missing_tags_summary': {}
    }
    
    paginator = tagging.get_paginator('get_resources')
    for page in paginator.paginate(
        ResourceARNList=resource_arns if resource_arns else [],
        TagFilters=[]
    ):
        for resource in page['ResourceTagMappingList']:
            arn = resource['ResourceARN']
            existing_tags = {t['Key']: t['Value'] for t in resource.get('Tags', [])}
            
            missing = []
            invalid = []
            
            for tag_key, valid_values in REQUIRED_TAGS.items():
                if tag_key not in existing_tags:
                    missing.append(tag_key)
                    report['missing_tags_summary'][tag_key] = \
                        report['missing_tags_summary'].get(tag_key, 0) + 1
                elif valid_values and existing_tags[tag_key] not in valid_values:
                    invalid.append(f"{tag_key}={existing_tags[tag_key]}")
            
            if missing or invalid:
                report['non_compliant'].append({
                    'arn': arn,
                    'missing': missing,
                    'invalid': invalid
                })
            else:
                report['compliant'].append(arn)
    
    compliance_rate = len(report['compliant']) / \
                     (len(report['compliant']) + len(report['non_compliant'])) * 100
    report['compliance_rate'] = f"{compliance_rate:.1f}%"
    
    return report
```

### 5.3 비용 이상 감지 (Anomaly Detection)

AWS Cost Anomaly Detection과 Azure Cost Alerts만으로는 충분하지 않다. 조직 특성에 맞는 커스텀 이상 감지 로직이 필요하다.

```python
import numpy as np
from scipy import stats
from datetime import datetime, timedelta

class CostAnomalyDetector:
    """통계 기반 비용 이상 감지"""
    
    def __init__(self, baseline_days: int = 30, threshold_sigma: float = 2.5):
        self.baseline_days = baseline_days
        self.threshold_sigma = threshold_sigma
    
    def detect_anomalies(self, daily_costs: list) -> list:
        """
        Z-score와 IQR을 결합한 강건한 이상 감지.
        계절성(주간 패턴)을 보정한 후 이상치를 판별한다.
        """
        if len(daily_costs) < self.baseline_days:
            return []
        
        costs = np.array(daily_costs)
        
        # 주간 계절성 제거 (월요일=0 ... 일요일=6)
        day_of_week = np.arange(len(costs)) % 7
        seasonal_means = np.array([
            np.mean(costs[day_of_week == d]) for d in range(7)
        ])
        overall_mean = np.mean(costs)
        seasonal_factors = seasonal_means / overall_mean
        deseasonalized = costs / seasonal_factors[day_of_week]
        
        # 롤링 Z-score 계산 (최근 baseline_days 기준)
        anomalies = []
        for i in range(self.baseline_days, len(costs)):
            window = deseasonalized[i - self.baseline_days:i]
            z_score = (deseasonalized[i] - np.mean(window)) / (np.std(window) + 1e-10)
            
            if abs(z_score) > self.threshold_sigma:
                anomaly_pct = (costs[i] - np.mean(costs[i-7:i])) / np.mean(costs[i-7:i]) * 100
                anomalies.append({
                    'day_index': i,
                    'cost': costs[i],
                    'z_score': z_score,
                    'change_pct': anomaly_pct,
                    'severity': 'HIGH' if abs(z_score) > 4 else 'MEDIUM'
                })
        
        return anomalies
    
    def forecast_monthly_cost(self, daily_costs: list) -> dict:
        """Prophet 스타일 간이 예측 (선형 트렌드 + 계절성)"""
        costs = np.array(daily_costs[-60:])   # 최근 60일 사용
        x = np.arange(len(costs))
        
        # 선형 트렌드 피팅
        slope, intercept, r_value, p_value, std_err = stats.linregress(x, costs)
        
        # 남은 일수 예측
        days_in_month = 30
        remaining_days = days_in_month - (len(costs) % days_in_month)
        
        forecast = []
        for d in range(remaining_days):
            pred_x = len(costs) + d
            predicted = slope * pred_x + intercept
            forecast.append(max(0, predicted))   # 음수 방지
        
        current_month_cost = sum(costs[-(len(costs) % days_in_month):])
        projected_total = current_month_cost + sum(forecast)
        
        return {
            'current_month_to_date': current_month_cost,
            'projected_month_total': projected_total,
            'trend': 'increasing' if slope > 0 else 'decreasing',
            'daily_trend_delta': slope
        }
```

---

## 6. AI/ML·LLM 워크로드 최적화

### 6.1 GPU 최적화의 특수성

GPU 리소스 최적화는 CPU/메모리 최적화와 근본적으로 다른 접근이 필요하다. GPU는 높은 단가($3-35/hour for A100/H100)와 독점적 사용 특성으로 인해 최적화 효과가 극적으로 크다.

**MIG (Multi-Instance GPU)**는 A100/H100에서 단일 GPU를 논리적으로 분할하여 여러 워크로드가 공유할 수 있게 한다. A100 80GB는 최대 7개의 MIG 인스턴스(각 10GB)로 분할 가능하며, 소형 추론 워크로드에서 GPU 활용률을 10-20%에서 70-90%로 끌어올릴 수 있다.

```yaml
# Kubernetes에서 MIG 분할 GPU 활용
# nvidia-device-plugin이 MIG 모드로 설정된 후
apiVersion: apps/v1
kind: Deployment
metadata:
  name: small-inference-service
spec:
  replicas: 7   # A100 1개를 7개 파드가 나눠 쓰는 경우
  template:
    spec:
      containers:
        - name: inference
          image: my-inference:latest
          resources:
            limits:
              nvidia.com/mig-1g.10gb: 1   # MIG 슬라이스 1개 요청
```

**GPU 시간 공유(Time-slicing)**는 MIG보다 격리는 약하지만 더 많은 GPU 패밀리에서 지원된다. NVIDIA GPU Operator를 통해 단일 GPU를 최대 48개의 프로세스가 시간 분할로 공유할 수 있다.

### 6.2 LLM 추론 비용 최적화

LLM 추론의 비용 구조는 일반 ML과 다르다. 입력 토큰 처리(Prefill Phase)와 출력 토큰 생성(Decode Phase)의 컴퓨트 특성이 완전히 다르기 때문이다.

**Continuous Batching**은 LLM 추론 서버(vLLM, TGI)가 채택한 핵심 최적화다. 전통적인 정적 배치는 배치 내 가장 긴 시퀀스가 완료될 때까지 GPU가 유휴 상태인 문제가 있었다. Continuous Batching은 완료된 시퀀스 자리에 즉시 새 요청을 채워 GPU 활용률을 극대화한다.

**KV-Cache 최적화**는 어텐션 레이어의 키-밸류 캐시를 효율적으로 관리하는 것이다. vLLM의 PagedAttention은 운영체제의 가상 메모리 페이징 개념을 KV 캐시에 적용하여, 메모리 단편화를 4% 미만으로 줄이고 배치 크기를 기존 대비 최대 24배 늘릴 수 있게 했다.

**모델 양자화**는 FP16 → INT8 → INT4 변환을 통해 모델 크기와 추론 비용을 크게 줄인다. GPTQ, AWQ, GGUF 등의 양자화 방식에 따라 성능 저하 폭이 다르므로, 태스크별 품질-비용 트레이드오프 분석이 필요하다.

```python
# 모델 양자화 비용-성능 트레이드오프 분석 프레임워크
from dataclasses import dataclass
from typing import Dict

@dataclass
class QuantizationProfile:
    name: str
    bits: int
    size_reduction_factor: float   # FP16 대비
    latency_reduction_factor: float
    quality_retention: float       # 1.0 = 동일, 0.95 = 5% 저하
    gpu_memory_gb: float

QUANTIZATION_PROFILES = {
    'FP16':  QuantizationProfile('FP16',  16, 1.0,  1.0,  1.000, 14.0),
    'INT8':  QuantizationProfile('INT8',   8, 2.0,  1.4,  0.985,  7.0),
    'INT4':  QuantizationProfile('INT4',   4, 4.0,  2.2,  0.960,  3.5),
    'INT2':  QuantizationProfile('INT2',   2, 8.0,  3.5,  0.890,  1.8),
}

def analyze_quantization_roi(
    baseline_hourly_cost: float,
    requests_per_hour: int,
    quality_threshold: float = 0.95
) -> None:
    """양자화 수준별 ROI 분석"""
    print(f"{'방식':<8} {'시간당 비용':>12} {'처리량 향상':>12} {'품질':>8} {'ROI':>10}")
    print("-" * 55)
    
    for name, profile in QUANTIZATION_PROFILES.items():
        if profile.quality_retention < quality_threshold:
            status = "❌ 품질 기준 미충족"
            continue
        
        adjusted_cost = baseline_hourly_cost / profile.size_reduction_factor
        throughput_gain = profile.latency_reduction_factor
        adjusted_cost_per_request = adjusted_cost / (requests_per_hour * throughput_gain)
        baseline_cost_per_request = baseline_hourly_cost / requests_per_hour
        roi = (baseline_cost_per_request - adjusted_cost_per_request) / baseline_cost_per_request * 100
        
        print(f"{name:<8} ${adjusted_cost:>10.2f}/h {throughput_gain:>11.1f}x "
              f"{profile.quality_retention:>7.1%} {roi:>9.1f}%")

analyze_quantization_roi(
    baseline_hourly_cost=3.0,    # A10G GPU 비용
    requests_per_hour=1000,
    quality_threshold=0.96
)
```

### 6.3 SageMaker 비용 최적화

AWS SageMaker는 ML 플랫폼으로서 강력하지만, 기본 설정으로 운영하면 비용이 급증한다. 주요 최적화 포인트는 다음과 같다.

**SageMaker Training Spot Instances**는 학습 작업에 스팟 인스턴스를 적용하여 최대 90% 비용 절감이 가능하다. 인터럽트 발생 시 체크포인트에서 재개하는 기능을 반드시 구현해야 한다.

**SageMaker Inference Endpoint 최적화**: 실시간 엔드포인트는 24/7 과금되므로, 배치 추론이 가능한 경우 SageMaker Batch Transform을 활용하거나, 낮은 트래픽 시간대에는 Serverless Inference(Lambda 기반)로 전환하는 하이브리드 전략이 효과적이다.

**Multi-Model Endpoint**는 수천 개의 소형 모델을 단일 엔드포인트로 서빙하여 인스턴스 비용을 최대 수백 배 절감할 수 있다.

---

## 7. 멀티클라우드 비용 통합 아키텍처

### 7.1 멀티클라우드 비용 통합의 과제

AWS, Azure, GCP를 동시에 운영하는 환경에서 비용 통합 가시성을 확보하는 것은 기술적으로 복잡하다. 각 클라우드의 비용 데이터 모델이 다르고, 청구 기준과 서비스 분류 체계가 상이하다.

**FOCUS (FinOps Open Cost and Usage Specification)**는 FinOps Foundation이 주도하는 클라우드 비용 데이터 표준화 스펙으로, AWS, Azure, GCP, Oracle이 모두 채택했다. FOCUS 기반으로 비용 데이터를 정규화하면 멀티클라우드 비용 분석이 크게 단순화된다.

### 7.2 OpenCost를 활용한 Kubernetes 비용 배분

OpenCost는 CNCF의 Kubernetes 비용 측정 표준이다. 클라우드 청구 데이터와 Kubernetes 리소스 사용량을 연계하여 네임스페이스, 파드, 레이블 수준의 정확한 비용 배분이 가능하다.

```yaml
# OpenCost Helm 설치 및 설정
# helm repo add opencost https://opencost.github.io/opencost-helm-chart

apiVersion: v1
kind: ConfigMap
metadata:
  name: opencost-conf
  namespace: opencost
data:
  default.json: |
    {
      "provider": "aws",
      "awsSpotDataRegion": "us-east-1",
      "awsSpotDataBucket": "my-spot-data-bucket",
      "projectID": "123456789",
      "awsAthenaProjectID": "123456789",
      "athenaBucketName": "s3://aws-athena-query-results/",
      "athenaRegion": "us-east-1",
      "athenaDatabase": "athenacurcfn_cur_db",
      "athenaTable": "cur_report",
      "masterPayerARN": "arn:aws:iam::master:role/OpenCostRole",
      
      "customPricesEnabled": false,
      "CPU": "0.031611",
      "spotCPU": "0.006655",
      "RAM": "0.004237",
      "spotRAM": "0.000892",
      "GPU": "2.936",
      "storage": "0.00004,
      "zoneNetworkEgress": "0.01",
      "regionNetworkEgress": "0.02",
      "internetNetworkEgress": "0.09"
    }
```

---

## 8. 면접 Q&A 35선

> 대상: Amazon, Apple, Nvidia, Google, Microsoft, Tesla  
> 난이도: Senior Cloud Engineer / Staff Engineer 수준

---

### [AWS 심화 영역 — 10문항]

---

**Q1. EC2 Savings Plans와 Reserved Instances의 차이점을 설명하고, 어떤 상황에서 각각을 선택해야 하는지 설명하세요. 특히 인스턴스 패밀리 마이그레이션이 예정된 환경에서 어떤 전략을 취하겠습니까?**

**A1.** Savings Plans와 Reserved Instances는 모두 약정 기반 할인이지만, 유연성과 최대 할인율 사이의 트레이드오프가 다릅니다. Reserved Instances는 인스턴스 패밀리, 리전, OS, 테넌시를 확정하는 대신 최대 72%의 더 큰 할인을 제공합니다. 반면 Compute Savings Plans는 인스턴스 패밀리, 리전, 컴퓨트 타입에 관계없이 적용되며 최대 66% 할인을 제공합니다.

인스턴스 패밀리 마이그레이션이 예정된 환경(예: x86 → Graviton3 전환 로드맵이 있는 경우)에서는 Compute Savings Plans를 선택하는 것이 합리적입니다. RI를 구매한 상태에서 Graviton으로 전환하면 기존 RI가 낭비됩니다. Savings Plans는 c7g나 m7g처럼 새로운 Graviton 인스턴스에도 자동 적용되기 때문입니다.

전략적으로는, 기저 수요의 70-80%를 Compute Savings Plans로 커버하고, 매우 안정적인 워크로드(예: 특정 리전에 고정된 데이터베이스 서버)에 한해 EC2 Instance Savings Plans나 RI를 적용하는 레이어드 접근이 효과적입니다. 나머지 20-30%는 스팟이나 온디맨드로 유연성을 확보합니다.

---

**Q2. 대규모 EKS 클러스터에서 NAT Gateway 비용이 예상보다 3배 높게 청구되었습니다. 원인을 진단하고 해결하는 방법을 체계적으로 설명하세요.**

**A2.** NAT Gateway 비용 급증은 Kubernetes 환경에서 매우 흔한 문제로, 주요 원인은 다음과 같습니다.

첫째로 가장 흔한 원인은 **AZ 간 트래픽 패턴**입니다. 파드가 다른 AZ의 NAT Gateway를 경유하면 AZ 간 전송 비용($0.01/GB)이 중복 발생합니다. 이는 파드가 어느 AZ에 있는지 관계없이 특정 AZ의 NAT Gateway만 사용하도록 라우팅이 설정된 경우 발생합니다. 해결책은 각 AZ에 NAT Gateway를 배포하고, 파드가 자신의 AZ 내 NAT Gateway를 사용하도록 라우팅 테이블을 설정하는 것입니다.

둘째로 **ECR 이미지 풀 트래픽**이 NAT Gateway를 경유하는 경우입니다. ECR에 VPC Endpoint를 생성하면 이 트래픽이 완전히 우회됩니다. ECR pull 트래픽은 노드 규모에 따라 상당한 볼륨이 될 수 있습니다.

셋째로 S3, DynamoDB, SecretsManager, SSM 등 AWS 서비스 호출이 NAT를 경유하는 경우입니다. Gateway Endpoint(S3, DynamoDB)와 Interface Endpoint를 적절히 배포하여 해결합니다.

진단은 VPC Flow Logs를 활성화하고 Athena로 쿼리하여 NAT Gateway를 경유하는 상위 목적지 IP를 식별하는 것에서 시작합니다. `destination_prefix_list_id`가 없는 트래픽이 모두 NAT를 경유하므로, AWS 서비스 IP 대역과 대조하여 Endpoint 설치 대상을 결정합니다.

---

**Q3. AWS Cost Anomaly Detection을 사용하고 있지만 노이즈가 너무 많아 실제 이상 징후를 놓치고 있습니다. 개선 방안을 제시하세요.**

**A3.** AWS Cost Anomaly Detection은 ML 기반이지만 조직의 특수한 비용 패턴(예: 월말 배치 작업, 계절적 트래픽)을 완전히 학습하지 못해 false positive가 많습니다. 개선 접근은 세 계층으로 구성됩니다.

모니터 설계를 정교화하는 것이 첫 번째입니다. 단일 전체 계정 모니터 대신, 서비스별/링크드 어카운트별로 세분화된 모니터를 생성하고, 각 모니터에 적절한 임계값을 설정합니다. 예를 들어 SageMaker 학습 작업은 일별 변동폭이 크므로 임계값을 높게, RDS 비용은 안정적이므로 낮게 설정합니다.

커스텀 이상 감지 로직을 구축하는 것이 두 번째입니다. Cost Explorer API로 일별 비용 데이터를 수집하여, 주간 계절성을 보정한 Z-score 기반 이상 감지를 구현합니다. 단순 임계값이 아닌 통계적 신뢰구간을 사용하면 false positive를 크게 줄일 수 있습니다.

경보 피로 감소를 위한 알림 계층화가 세 번째입니다. Slack의 낮은 심각도 채널로 모든 이상 징후를 보내되, 실제 사람이 즉시 대응해야 할 HIGH 심각도 이상만 PagerDuty나 SMS로 에스컬레이션하는 구조를 만듭니다. 이상 징후의 원인이 특정 리소스(예: 특정 EC2 인스턴스 ID, 특정 Lambda 함수)에 귀속될 때만 HIGH로 분류하는 규칙도 효과적입니다.

---

**Q4. S3 비용이 갑자기 증가했는데 Replication 관련 비용이 원인이라는 의심이 있습니다. 어떻게 조사하고 최적화하겠습니까?**

**A4.** S3 복제 비용은 여러 구성 요소로 이루어집니다. 원본 PUT/COPY 요청 비용, 복제 데이터 전송 비용(리전 간 복제의 경우 GB당 전송 요금), 목적지 버킷의 스토리지 비용, 그리고 S3 Replication Time Control(RTC) 사용 시 추가 요금이 발생합니다.

조사는 Cost Explorer에서 S3 서비스를 `UsageType`으로 드릴다운하여 `DataTransfer-Out-Bytes` 및 `S3-Replication` 관련 항목이 증가했는지 확인하는 것으로 시작합니다. S3 서버 접근 로깅 또는 CloudTrail을 통해 복제 작업 로그를 분석합니다.

최적화 방안은 다음과 같습니다. 복제가 진짜 필요한지 재검토합니다. 컴플라이언스나 DR 목적이 아닌 단순 백업이라면 S3 Cross-Region Replication 대신 Glacier 아카이빙이 훨씬 저렴합니다. 복제가 필요하다면 수명 주기 필터를 적용하여 모든 객체를 복제하지 않고 특정 접두사 또는 태그가 있는 중요 데이터만 복제합니다. 또한 복제 목적지에도 수명 주기 정책을 적용하여 복제된 데이터가 자동으로 저렴한 스토리지 클래스로 전환되도록 합니다.

---

**Q5. Amazon SageMaker 학습 비용을 최소화하면서도 학습 시간이 과도하게 늘어나지 않게 하는 전략을 설명하세요.**

**A5.** SageMaker 학습 비용 최적화는 세 가지 축으로 접근해야 합니다.

**인스턴스 유형 선택 최적화**가 첫 번째입니다. 모든 학습에 p4d.24xlarge(A100)를 쓰는 것은 과잉이며, 워크로드 특성에 맞는 인스턴스를 선택해야 합니다. 소형 모델이나 프로토타이핑에는 g5 계열, 대형 모델 학습에만 p4d 또는 p5, 분산 학습에는 trn1(Trainium) 인스턴스가 비용 효율적입니다. Trainium은 같은 학습 작업에서 A100 대비 40-50% 저렴합니다.

**스팟 인스턴스 + 체크포인팅**이 두 번째입니다. SageMaker는 `use_spot_instances=True` 설정만으로 스팟을 활용할 수 있으며, `checkpoint_s3_uri`를 설정하면 인터럽트 발생 시 자동 재시작됩니다. 단, 체크포인트 저장 빈도를 에포크 단위가 아닌 스텝 단위로 높이면 재시작 시 낭비를 최소화할 수 있습니다.

**효율적인 분산 학습 전략**이 세 번째입니다. 데이터 병렬, 모델 병렬, 파이프라인 병렬의 적절한 조합을 선택해야 합니다. SageMaker Distributed Library의 Tensor Parallelism과 Pipeline Parallelism을 활용하면 GPU 활용률을 높이고 학습 시간을 단축할 수 있습니다. SageMaker Debugger로 GPU 활용률을 모니터링하여 병목(데이터 로딩, 불균형한 분산 등)을 식별하는 것도 필수입니다.

---

**Q6. 여러 AWS 계정을 AWS Organizations로 관리 중입니다. 연결 계정(Linked Account)들의 RI/SP 공유를 어떻게 최적화하겠습니까?**

**A6.** AWS Organizations에서 RI와 Savings Plans는 기본적으로 퍼어(Payer) 계정에서 모든 연결 계정으로 공유됩니다. 이 기본 설정이 항상 최적은 아니며, 세밀한 제어가 필요합니다.

RI 공유를 비활성화해야 하는 경우가 있습니다. 부서별 정확한 비용 배분이 필요하거나, 특정 팀이 RI를 구매했는데 다른 팀의 온디맨드 사용량에 혜택이 가는 상황을 방지해야 할 때입니다. `ModifyAccount`를 통해 특정 연결 계정의 RI 공유를 비활성화할 수 있습니다.

최적화 전략은 **RI 포트폴리오 중앙 관리**입니다. 모든 RI와 SP를 퍼어 계정에서 구매하고, Cost Allocation Tags와 Cost Categories를 활용하여 실제 사용 계정에 비용을 귀속시킵니다. 이를 통해 중앙에서 RI 활용률을 극대화하면서도 각 팀의 비용 책임성을 유지할 수 있습니다. AWS License Manager와 연계하여 RI 구매 승인 워크플로우도 구현합니다.

---

**Q7. AWS Lambda 함수의 비용이 급증했습니다. Cold Start 문제와 메모리 크기 최적화 관점에서 어떻게 접근하겠습니까?**

**A7.** Lambda 비용은 `요청 수 × 메모리(GB) × 지속 시간(ms)`로 결정됩니다. 이 공식에서 메모리와 지속 시간이 역관계에 있다는 점이 중요합니다. 메모리를 늘리면 CPU 할당량도 같이 늘어 실행 시간이 줄어들므로, 반드시 메모리를 줄이는 것이 비용 절감이 아닙니다.

**AWS Lambda Power Tuning**은 이 최적화를 자동화하는 오픈소스 도구입니다. Step Functions를 활용하여 다양한 메모리 구성(128MB ~ 10GB)으로 Lambda를 반복 실행하고 비용-성능 곡선을 그려주므로, 데이터 기반으로 최적 메모리를 선택할 수 있습니다. 일반적으로 1024MB나 1769MB 설정에서 최적 비용-성능 비율이 나오는 경우가 많습니다.

Cold Start 비용 문제는 실행 시간 자체보다는 SLA 영향이 더 크지만, Provisioned Concurrency 비용이 과도한 경우가 있습니다. Provisioned Concurrency는 초당 요청 수가 예측 가능한 시간대에만 Application Auto Scaling을 활용해 동적으로 적용하면 비용을 크게 줄일 수 있습니다. 트래픽이 없는 새벽 시간에 Provisioned Concurrency를 0으로 스케일 다운하는 스케줄 기반 자동화가 효과적입니다.

---

**Q8. AWS Graviton 인스턴스로 마이그레이션하는 과정에서 성능 회귀(Regression)가 발생했습니다. 어떻게 진단하고 해결하겠습니까?**

**A8.** Graviton(ARM64)으로의 마이그레이션에서 성능 회귀는 주로 세 가지 영역에서 발생합니다.

**JVM 애플리케이션의 JIT 워밍업 차이**가 첫 번째입니다. ARM64용 JVM은 x86 대비 JIT 컴파일 최적화가 일부 다를 수 있습니다. `-XX:+UseG1GC -XX:MaxGCPauseMillis=200` 등의 GC 튜닝이 플랫폼별로 재조정이 필요할 수 있습니다. Amazon Corretto 17 이상을 사용하면 AWS가 Graviton에 최적화한 JVM 빌드를 제공합니다.

**네이티브 라이브러리 재컴파일 필요**가 두 번째입니다. NumPy, TensorFlow, PyTorch 같은 라이브러리들은 x86의 AVX-512 같은 SIMD 명령어를 직접 활용하도록 최적화되어 있습니다. Graviton3는 SVE(Scalable Vector Extension)를 지원하므로, AWS가 제공하는 arm64-optimized 빌드를 사용하거나 소스에서 재컴파일해야 합니다. AWS dlami 이미지는 Graviton에 최적화된 ML 라이브러리를 포함합니다.

**컨텍스트 스위치 비용 차이**가 세 번째입니다. 고도로 병렬화된 워크로드에서 스레드 수 설정이 x86과 달라야 할 수 있습니다. Graviton3의 코어당 성능 특성을 고려해 스레드 풀 크기를 재조정하는 것이 필요합니다.

성능 회귀를 정량화하기 위해 Amazon CloudWatch Container Insights와 애플리케이션 APM(예: AWS X-Ray, Datadog)을 활용하여 플랫폼 간 p50/p95/p99 레이턴시를 비교하고 병목 지점을 식별합니다.

---

**Q9. 데이터 레이크 환경에서 S3 Select와 Athena를 사용 중입니다. 스캔 비용을 최소화하기 위한 데이터 포맷 및 파티션 전략을 설명하세요.**

**A9.** S3 Select와 Athena는 스캔한 데이터 양($5/TB for Athena)에 따라 과금되므로, 데이터 포맷과 파티션 설계가 직접적으로 비용에 영향을 줍니다.

**컬럼형 포맷(Parquet/ORC) 채택**이 가장 기본적이고 임팩트가 큰 최적화입니다. 행 기반 포맷(CSV, JSON) 대비 쿼리에 필요한 컬럼만 읽으므로 스캔량을 80-95% 줄일 수 있습니다. Parquet의 Row Group Size는 기본 128MB로, 데이터 접근 패턴에 맞게 조정합니다.

**파티션 최적화**는 쿼리 패턴에 따라 파티션 키를 설계해야 합니다. `year=/month=/day=/` 구조는 시계열 쿼리에 최적화되어 있으며, 쿼리 시 날짜 범위 조건이 파티션 프루닝을 통해 불필요한 파티션 스캔을 방지합니다. 파티션이 너무 세분화되면 오히려 메타데이터 오버헤드가 커져 역효과가 납니다(일반적으로 파티션당 최소 100MB 이상을 권장).

**데이터 압축**은 Snappy(속도 우선)나 Zstd(압축률 우선)를 Parquet과 함께 사용하면 스토리지 비용과 스캔 비용을 동시에 줄입니다. Zstd는 Snappy보다 압축률이 20-30% 높으면서 디압축 속도도 경쟁력이 있습니다.

**Athena 쿼리 최적화**도 중요합니다. `SELECT *` 대신 필요한 컬럼만 선택하고, WHERE 절에 파티션 키를 포함하며, CTAS(Create Table As Select)로 자주 사용하는 집계 결과를 materialized view처럼 저장하는 패턴이 비용을 크게 줄입니다.

---

**Q10. AWS Cost Explorer API를 활용한 자동화된 비용 최적화 파이프라인을 설계한다면 어떻게 구성하겠습니까?**

**A10.** 완전 자동화된 비용 최적화 파이프라인은 5단계로 구성합니다.

**데이터 수집 계층**은 Cost Explorer API, CUR(Cost and Usage Report), CloudWatch Metrics를 EventBridge Scheduler로 매일 수집하여 S3에 저장하고, Glue Crawler로 카탈로그화합니다.

**분석 계층**은 Athena를 통해 서비스별, 계정별, 태그별 비용 트렌드를 분석하고, SageMaker를 활용한 비용 이상 감지 및 예측 모델을 운영합니다.

**권고 생성 계층**은 Cost Explorer의 Right-sizing 권고, RI/SP 구매 권고, 그리고 커스텀 룰(예: 7일 이상 중지된 EC2, 연결되지 않은 EBS)을 통합하여 우선순위화된 최적화 백로그를 생성합니다.

**자동 실행 계층**은 리스크에 따라 두 트랙으로 분리합니다. 저위험 작업(예: gp2→gp3 변환, 미사용 EIP 해제, S3 수명 주기 정책 적용)은 자동 실행하고, 고위험 작업(예: 인스턴스 타입 변경, RI 구매)은 Slack/JIRA 티켓으로 사람의 승인을 받은 후 실행합니다. AWS SSM Automation Document를 활용하면 승인 게이트와 자동 실행 롤백이 모두 가능합니다.

**리포팅 계층**은 QuickSight 또는 Grafana 대시보드로 실시간 비용 현황을 시각화하고, 월별 절감 효과를 팀별로 귀속시켜 조직 내 비용 최적화 문화를 강화합니다.

---

### [Kubernetes / EKS 심화 영역 — 10문항]

---

**Q11. Kubernetes 클러스터의 CPU 활용률이 평균 15%에 불과합니다. 근본 원인 분석과 개선 방안을 설명하세요.**

**A11.** CPU 활용률 15%는 Request 기반 스케줄링의 전형적인 문제입니다. 근본 원인 분석을 위해 세 가지를 구분해야 합니다.

**Request vs Actual Usage Gap** 확인이 첫 번째입니다. `kubectl top pods --all-namespaces`와 Prometheus의 `container_cpu_usage_seconds_total`을 비교하여, 파드의 CPU request 대비 실제 사용량 비율을 측정합니다. 대부분의 경우 개발자들이 안전 마진으로 request를 과도하게 높게 설정한 것이 원인입니다.

**Node Allocatable vs Requested 비율** 계산이 두 번째입니다. `kubectl describe node`로 각 노드의 Allocated resources 섹션을 확인합니다. 노드의 CPU allocatable 대비 requested 비율이 90%를 넘어도 실제 활용률이 15%라면, request 과다 설정이 확실한 원인입니다.

**개선 방안**으로는 VPA를 Auto 모드로 설정하여 request를 자동 최적화하거나, Goldilocks(FairwindsOps의 오픈소스)를 사용하여 VPA 추천값 기반의 request/limit 설정 가이드를 제공합니다. Karpenter의 Consolidation 기능을 활성화하면 낭비된 노드를 자동으로 통합합니다.

---

**Q12. Kubernetes에서 GPU 리소스를 여러 팀이 공유하는 환경을 설계한다면 어떻게 하겠습니까? 공정한 할당과 비용 배분을 모두 고려하세요.**

**A12.** GPU 공유 환경 설계는 기술적 공유 메커니즘과 거버넌스 두 계층으로 구성합니다.

**기술적 공유 레이어**는 워크로드 특성에 따라 분리합니다. 배치 학습 작업은 namespace별 ResourceQuota로 `requests.nvidia.com/gpu`를 제한하고, 학습 프레임워크(Kubeflow, Volcano)의 Gang Scheduling을 활용합니다. 추론 워크로드는 NVIDIA MIG(A100/H100 전용) 또는 Time-slicing을 적용하여 여러 서비스가 물리 GPU를 공유합니다. Time-slicing은 NVIDIA Device Plugin ConfigMap의 `sharing.timeSlicing.resources` 설정으로 활성화합니다.

**우선순위 및 선점(Preemption) 정책**은 PriorityClass를 활용합니다. 프로덕션 추론 서비스는 높은 우선순위, 실험적 학습 작업은 낮은 우선순위로 설정하여 중요한 워크로드가 항상 GPU를 확보하도록 합니다.

**비용 배분**은 OpenCost + GPU 비용 가중치를 설정하여 네임스페이스별 GPU 사용량을 측정하고, 팀별 차지백 리포트를 생성합니다. GPU는 CPU/메모리 대비 단가가 높으므로(H100의 경우 인스턴스 비용의 95%가 GPU 비용), 정확한 GPU 사용량 측정이 공정한 비용 배분의 핵심입니다.

---

**Q13. Kubernetes HPA와 VPA를 동시에 사용할 때 발생하는 충돌과 해결 방안을 설명하세요.**

**A13.** HPA와 VPA를 동시에 사용하면 충돌이 발생할 수 있습니다. HPA는 파드 수를 조정하고, VPA는 파드당 리소스를 조정하는데, VPA가 리소스를 변경하면 HPA의 스케일링 결정에 영향을 주는 피드백 루프가 생깁니다.

구체적 충돌 시나리오는 다음과 같습니다. CPU 기반 HPA가 활성화된 상태에서 VPA가 CPU request를 낮추면, 실제 CPU 사용률(사용량/request)이 상승하여 HPA가 불필요한 스케일 업을 트리거할 수 있습니다. 반대로 VPA가 CPU request를 높이면 HPA가 스케일 다운할 수 있습니다.

**권장 해결 방안**은 세 가지입니다. 첫째, CPU 기반 HPA와 VPA를 동시에 사용하지 않고, HPA에는 커스텀 메트릭(큐 깊이, RPS 등)을 사용하고 VPA가 CPU/메모리 request를 관리하게 합니다. 둘째, VPA의 `controlledValues`를 `RequestsOnly`로 설정하여 limit은 변경하지 않도록 합니다. 셋째, KEDA를 활용하여 HPA를 대체하고, VPA는 Initial 모드로만 사용하여 배포 시 최적 request를 설정하되 런타임 변경은 막습니다.

Kubernetes 1.32+에서는 APA(Autoscaling Policy API) 로드맵이 HPA와 VPA를 통합하는 방향으로 발전하고 있어, 향후 이 문제는 공식적으로 해결될 전망입니다.

---

**Q14. 대규모 Kubernetes 클러스터에서 네트워크 정책이 없거나 잘못 설정된 경우, 비용과 보안 관점에서 어떤 문제가 발생하고 어떻게 해결하겠습니까?**

**A14.** 네트워크 정책 미설정은 비용과 보안 두 관점에서 모두 문제를 야기합니다.

**비용 관점**에서, 불필요한 서비스 간 통신이 AZ 경계를 넘어 발생하면 AZ 간 전송 비용이 증가합니다. 또한 External DNS 레코드로 해결할 수 있는 내부 통신이 NLB/ALB를 경유하면 불필요한 로드 밸런서 비용이 발생합니다. Cilium의 Hubble을 활용한 네트워크 플로우 가시성으로 실제 통신 패턴을 측정하고 최적화 기회를 식별합니다.

**보안 관점**에서, 파드 간 제로 트러스트가 없으면 내부 이동(Lateral Movement) 위험이 존재합니다. NetworkPolicy로 필요한 통신만 허용하는 최소 권한 원칙을 적용합니다.

**단계적 구현 전략**은 감사 모드(audit mode)에서 시작하는 것입니다. Cilium이나 Calico의 감사 모드로 현재 트래픽 패턴을 수집한 후, 관측된 패턴을 기반으로 NetworkPolicy를 자동 생성하는 도구(Inspektor Gadget, Hubble)를 활용합니다. 처음부터 deny-all 정책을 적용하면 서비스 장애가 발생하므로, 점진적으로 정책을 강화하는 접근이 실무적입니다.

---

**Q15. EKS 클러스터에서 Fargate와 Managed Node Group의 비용을 비교하고, 어떤 워크로드에 각각을 사용하겠습니까?**

**A15.** Fargate와 Managed Node Group은 근본적으로 다른 비용 모델을 가집니다.

**Fargate**는 파드의 CPU와 메모리 request에 대해 초 단위로 과금됩니다($0.04048/vCPU-hour, $0.004445/GB-hour). 오버헤드가 없고 정확히 요청한 리소스만 과금되는 장점이 있습니다. 그러나 node overhead가 없는 대신, 노드 공유로 인한 Bin Packing 효율이 없어 일반적으로 EC2 대비 2-3배 비쌉니다.

**Managed Node Group(EC2)**은 노드 전체에 대해 과금되므로, 빈 용량이 있으면 낭비가 됩니다. 그러나 클러스터 활용률이 높으면 Fargate보다 훨씬 경제적입니다. 또한 Fargate는 DaemonSet 미지원, EFA 미지원, Privileged Container 미지원 등 기능적 제약이 있습니다.

**적합한 워크로드 분류**로는, Fargate가 적합한 경우는 산발적으로 실행되는 크론 작업, 격리가 중요한 멀티테넌트 환경, 노드 관리를 원하지 않는 소규모 팀, 보안 감사 요구사항이 강한 환경입니다. EC2가 적합한 경우는 상시 운영되는 서비스, GPU 워크로드, 고성능 네트워킹(EFA) 필요 시, 대규모 워크로드로 비용 효율이 중요한 경우입니다.

하이브리드 접근이 최선입니다. 기본 서비스는 EC2 노드 그룹 + Karpenter로 운영하고, 격리가 필요한 특정 워크로드(보안 민감 처리, 고객별 격리)는 Fargate로 처리합니다.

---

**Q16. Kubernetes Cluster Autoscaler와 Karpenter의 핵심 차이점을 설명하고, 실제 프로덕션 마이그레이션 경험이나 전략을 이야기하세요.**

**A16.** CAS와 Karpenter의 가장 근본적인 차이는 **스케일링 결정 단위**입니다. CAS는 "어떤 노드 그룹의 수를 늘릴 것인가"를 결정하고, Karpenter는 "이 파드에 가장 적합한 인스턴스 타입을 즉시 생성"합니다.

CAS의 한계는 세 가지입니다. ASG 기반이므로 노드 프로비저닝에 2-3분이 소요됩니다. 여러 노드 그룹 중 어디에 스케일 업할지 결정하는 Expander 로직이 단순합니다. 노드 그룹 간 통합(Consolidation)이 없어 부분적으로 사용되는 노드가 많습니다.

Karpenter의 장점은 EC2 Fleet API를 직접 호출하여 60초 이내 노드를 프로비저닝하고, 파드 요구사항에 최적화된 인스턴스 타입을 동적으로 선택하며, Consolidation으로 노드 수를 지속적으로 최적화한다는 점입니다.

**프로덕션 마이그레이션 전략**은 단계적으로 진행해야 합니다. 먼저 CAS와 Karpenter를 동시에 운영하되, 새 NodePool을 생성하고 특정 네임스페이스의 파드부터 Karpenter 관리로 전환합니다. Karpenter NodePool에 `provisioner: karpenter` 레이블을 설정하고, 해당 레이블이 있는 파드만 Karpenter가 처리하도록 합니다. 충분한 검증 후 모든 워크로드를 전환하고 CAS를 제거합니다.

---

**Q17. 멀티테넌트 Kubernetes 환경에서 팀별 비용 배분을 자동화하는 아키텍처를 설계하세요.**

**A17.** 멀티테넌트 비용 배분 자동화는 측정, 배분, 리포팅 세 계층으로 구성합니다.

**측정 계층**은 OpenCost를 배포하여 네임스페이스 단위의 CPU, 메모리, GPU, 스토리지, 네트워크 비용을 실시간 측정합니다. OpenCost는 클라우드 청구 데이터(CUR)와 연계하여 실제 비용에 기반한 측정이 가능합니다. Shared 리소스(모니터링, 로깅, 인그레스 컨트롤러)는 별도 namespace에 배치하고 비례 배분(프로포셔널 배분)이나 균등 배분 정책을 사전에 정의합니다.

**배분 규칙**은 실제 사용량 기반(Actual Usage Based)이 이상적이지만, 예약된 용량 기반(Reserved Capacity Based)도 팀 예측 가능성을 위해 병행합니다. 예를 들어 팀이 Request한 용량에 대해 70%를 과금하고, 실제 Limit까지 사용한 부분에 대해 추가 30%를 과금하는 하이브리드 모델이 팀 행동 변화를 유도하는 데 효과적입니다.

**리포팅 및 피드백 루프**는 주간 자동 리포트를 팀 Slack 채널로 발송하고, 비용이 예산 대비 80%를 초과하면 경보를 보냅니다. 팀이 직접 Grafana 대시보드로 자신의 비용을 조회할 수 있게 하여 자율적 최적화를 유도합니다.

---

**Q18. Kubernetes 파드의 리소스 Request가 없는 BestEffort 파드가 대량으로 배포된 상황을 발견했습니다. 어떻게 처리하겠습니까?**

**A18.** BestEffort 파드는 스케줄링 효율성과 클러스터 안정성 모두에 악영향을 미칩니다. 스케줄러가 해당 파드를 무시하고 노드 배치를 결정하므로, 실제로 상당한 리소스를 소비하는 파드가 보이지 않는 곳에서 클러스터를 압박합니다.

**즉각적 완화 조치**로는 LimitRange에 `defaultRequest`를 설정하여 신규 파드가 자동으로 최소 request를 갖도록 합니다. 이는 기존 파드에는 소급 적용되지 않으므로, Gatekeeper(OPA)나 Kyverno Policy를 통해 Request가 없는 파드 배포를 CI/CD 단계에서 차단합니다.

```yaml
# Kyverno Policy: CPU/Memory Request 필수화
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-requests
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-resources
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "CPU와 Memory request가 필수입니다."
        pattern:
          spec:
            containers:
              - (name): "*"
                resources:
                  requests:
                    cpu: "?*"
                    memory: "?*"
```

**기존 파드 마이그레이션**은 팀별로 VPA 권고값을 공유하고 2주 일정으로 수정을 요청합니다. 수정하지 않은 팀의 파드는 낮은 우선순위 NodePool로 이동시켜 클러스터 불안정성 위험을 격리합니다.

---

**Q19. Kubernetes Ingress 레이어의 비용 최적화 전략을 설명하세요. 특히 여러 팀이 각자 ALB를 생성하는 환경을 어떻게 개선하겠습니까?**

**A19.** ALB 남발은 Kubernetes 환경에서 흔한 비용 낭비 패턴입니다. ALB는 시간당 $0.008 + 처리된 트래픽에 따라 LCU(Load Balancer Capacity Unit) 비용이 추가되므로, 수십 개의 ALB가 생성되면 월 수백~수천 달러의 불필요한 비용이 발생합니다.

**해결 방안은 공유 Ingress 아키텍처**입니다. 클러스터당 소수의 공유 ALB를 운영하고, Ingress Controller(AWS Load Balancer Controller의 IngressGroup 기능)를 활용하여 여러 Ingress 리소스를 단일 ALB로 통합합니다. `alb.ingress.kubernetes.io/group.name` 어노테이션으로 같은 그룹의 Ingress를 하나의 ALB로 묶습니다.

내부 서비스 간 통신은 ALB를 완전히 제거하고, Kubernetes Service(ClusterIP)와 서비스 메시(Istio, Linkerd)를 활용합니다. 서비스 메시는 비용이 추가되지만 ALB 절감 효과가 더 크고, mTLS 보안까지 제공합니다.

팀의 자율적 Ingress 생성을 제한하기 위해 Gatekeeper/Kyverno로 IngressGroup 어노테이션이 없는 ALB 타입 Ingress 생성을 차단하고, 승인된 그룹명만 허용하는 정책을 시행합니다.

---

**Q20. Kubernetes에서 Stateful 워크로드(데이터베이스, Kafka)의 비용 최적화는 Stateless 워크로드와 어떻게 다른가요?**

**A20.** Stateful 워크로드는 데이터 영속성, 네트워크 안정성, 재시작 비용 측면에서 Stateless와 근본적으로 다른 최적화 접근이 필요합니다.

**스팟 인스턴스 적용 불가**가 가장 큰 차이입니다. 일반적으로 데이터베이스나 Kafka 브로커는 스팟을 직접 사용할 수 없습니다. 대신, DB를 온디맨드(또는 RI)로 운영하고, DB에 연결하는 애플리케이션 레이어만 스팟으로 운영하는 분리 전략이 현실적입니다. 단, Kafka는 복제 팩터와 min.insync.replicas를 적절히 설정하면 일부 브로커를 스팟으로 운영할 수 있습니다.

**스토리지 최적화**가 Stateful 워크로드에서 더 중요합니다. PV(Persistent Volume)에 사용되는 EBS 볼륨은 파드와 독립적으로 과금되므로, 사용하지 않는 PVC와 PV를 주기적으로 정리하는 자동화가 필요합니다. `kubectl get pvc --all-namespaces`로 Bound 상태가 아닌 PVC를 식별하고, `StorageClass`의 `reclaimPolicy`를 Retain에서 Delete로 변경하는 것도 PV 고아 방지에 도움이 됩니다.

**RDS 대 자체 관리 DB 비용 비교**도 중요한 결정입니다. 소규모에서는 자체 관리가 저렴하지만, 운영 인건비와 가용성 구현 비용을 고려하면 RDS가 TCO 측면에서 유리한 경우가 많습니다.

---

### [Azure 심화 영역 — 5문항]

---

**Q21. Azure Reserved Instances와 Savings Plans를 혼합 운영할 때의 최적화 전략을 설명하세요.**

**A21.** Azure에서 RI와 Savings Plans를 혼합 운영하는 이유는 두 제품의 할인 구조가 다르기 때문입니다. RI는 특정 VM 시리즈와 리전에 고정되어 할인율이 높고, Savings Plans는 VM 시리즈와 리전에 무관하게 적용되어 유연합니다.

최적 전략은 **스테이블 기저 워크로드에 RI, 가변적 워크로드에 Savings Plans**를 적용하는 계층 구조입니다. 예를 들어 프로덕션 데이터베이스 서버처럼 항상 같은 VM SKU를 사용하는 경우 3년 RI로 최대 할인을 받고, 모델 학습처럼 VM 크기나 리전이 자주 바뀌는 경우 Compute Savings Plans를 적용합니다.

Azure Hybrid Benefit을 RI/SP와 중첩 적용하는 것도 중요합니다. AHB와 3년 RI를 결합하면 Pay-as-you-go 대비 최대 80-85%의 비용 절감이 가능합니다. Windows Server 라이선스를 보유한 조직은 이 조합이 가장 강력한 비용 절감 레버입니다.

---

**Q22. Azure Policy와 Budgets를 활용하여 팀별 클라우드 비용 거버넌스 체계를 구축하는 방법을 설명하세요.**

**A22.** Azure 비용 거버넌스는 Management Group → Subscription → Resource Group 계층을 기반으로 구축합니다.

**Azure Policy**로 허용되지 않는 리소스 유형 또는 VM SKU를 차단하고(Deny 효과), 필수 태그가 없는 리소스 생성을 거부합니다. `Allowed virtual machine size SKUs` Policy를 통해 과도하게 큰 VM(예: Standard_M Series) 생성을 차단하는 것이 실질적 비용 제어에 효과적입니다.

**Azure Budgets**는 구독 또는 리소스 그룹 수준에서 월별 예산을 설정하고, 50%/75%/90%/100% 임계값에서 이메일 알림과 Action Group(Azure Functions, Logic Apps) 트리거를 설정합니다. 100% 초과 시 자동으로 비개발 환경의 VM을 중지하는 자동화를 구현하면 예산 초과를 선제적으로 방지할 수 있습니다.

---

**Q23. Azure AKS에서 노드 풀을 최적화하는 전략을 설명하세요. 특히 시스템 노드 풀과 사용자 노드 풀 분리의 이유와 방법을 설명하세요.**

**A23.** AKS에서 시스템 노드 풀과 사용자 노드 풀을 분리하는 것은 비용과 안정성 모두에 관련됩니다.

시스템 노드 풀은 CoreDNS, kube-proxy, Azure CNI 등 클러스터 필수 구성 요소를 실행합니다. 이 풀은 항상 온라인이어야 하므로 스팟 인스턴스를 사용할 수 없고, 적정 크기의 안정적인 VM(예: Standard_D4s_v5 2-3개)으로 운영합니다.

사용자 노드 풀은 실제 애플리케이션 워크로드를 위한 것으로, 스팟 인스턴스를 적용하거나 워크로드별 특화 VM(GPU, 고메모리)을 별도 풀로 운영할 수 있습니다. `kubectl taint` + `nodeSelector`/`tolerations`를 활용하여 사용자 워크로드가 사용자 노드 풀에만 배치되도록 합니다.

Node Auto Provisioning(NAP, Karpenter 기반)을 활용하면 개별 노드 풀을 수동 관리할 필요 없이 파드 요구사항에 따라 최적 VM SKU가 자동 선택됩니다. AKS 1.28+ 에서 NAP가 GA되었습니다.

---

**Q24. Azure Monitor와 Log Analytics를 활용한 비용 최적화 인사이트 도출 방법을 설명하세요.**

**A24.** Azure Monitor 자체도 상당한 비용을 발생시킬 수 있습니다(Log Analytics는 GB당 ingestion 비용, 데이터 보존 비용). 따라서 모니터링 비용 최적화도 함께 고려해야 합니다.

Log Analytics의 주요 비용은 데이터 수집량입니다. 기본 설정으로는 VM의 IIS 로그, Windows 이벤트 로그, 각종 진단 로그가 모두 수집되어 불필요하게 많은 데이터가 쌓입니다. Data Collection Rules(DCR)을 사용하여 필요한 로그 타입만 선택적으로 수집하고, 중요도가 낮은 로그는 Basic Logs(저렴하지만 쿼리 기능 제한)로 전환합니다.

비용 최적화 인사이트 측면에서는, KQL(Kusto Query Language)로 VM CPU/메모리 평균 활용률이 낮은 리소스를 식별하는 쿼리를 작성하여 정기 실행하고, 결과를 Azure Dashboard에 시각화합니다. Azure Workbooks를 활용하면 비용 최적화 인사이트를 대화형 리포트로 표현할 수 있습니다.

---

**Q25. Microsoft Azure의 MACC(Microsoft Azure Consumption Commitment)를 협상하는 과정과 전략을 설명하세요.**

**A25.** MACC는 단순한 볼륨 할인 계약이 아니라 Microsoft와의 전략적 파트너십입니다. 협상 전략은 다음과 같습니다.

**사전 준비**로 현재 Azure 지출 데이터와 향후 3년 성장 계획을 정확히 파악해야 합니다. Microsoft 측은 현재 지출의 150-200% 수준의 약정을 제안하는 경향이 있으므로, 실제 성장 계획보다 적은 금액으로 약정하는 것이 안전합니다. 약정 미달 시 차액 패널티가 없는 MACC를 목표로 협상합니다.

**협상 레버**로는 M365, Teams, GitHub Enterprise 등 Microsoft 전체 포트폴리오와의 통합 거래 규모를 활용하거나, 경쟁사(AWS, GCP) 견적을 협상 카드로 활용합니다. 연간 지출 $1M 이상인 경우 전용 CSA(Cloud Solution Architect) 지원, 기술 크레딧, 파일럿 프로그램 지원도 협상 항목에 포함할 수 있습니다.

MACC 체결 후에는 Microsoft Cost Management의 소비 현황을 월별 추적하고, 사용 속도(burn rate)가 예상보다 낮으면 마이그레이션 가속화나 신규 서비스 도입을 검토합니다.

---

### [AI/ML · LLMOps 비용 최적화 — 5문항]

---

**Q26. LLM 추론 서버를 운영하면서 GPU 활용률이 30%에 불과합니다. 어떻게 최적화하겠습니까?**

**A26.** LLM 추론 GPU 활용률 30%는 주로 세 가지 원인으로 발생합니다.

**배칭 비효율**이 첫 번째입니다. 요청이 순서대로 하나씩 처리된다면 GPU는 대부분의 시간을 대기로 보냅니다. vLLM, TGI(Text Generation Inference), TensorRT-LLM으로 전환하여 Continuous Batching을 활성화합니다. 이를 통해 GPU 활용률을 30%에서 70-85%로 끌어올릴 수 있습니다.

**토큰 예산 불균형**이 두 번째입니다. 짧은 응답을 생성하는 요청들이 대부분이라면 배치가 금방 완료되어 GPU가 대기합니다. `max_batch_prefill_tokens`와 `max_total_tokens` 파라미터를 워크로드에 맞게 조정합니다.

**동적 배치 스케줄링**이 세 번째 해결책입니다. NVIDIA Triton Inference Server의 Dynamic Batching을 활용하면 비슷한 길이의 요청을 함께 배치하여 패딩 낭비를 줄입니다.

추가로, 요청이 적은 시간대에는 KEDA로 추론 서버 파드를 scale-to-zero하거나 최소 1개로 유지하여 GPU 비용을 절감합니다. 저전력 GPU(A10G, L4)는 A100보다 적합한 중소형 모델 추론에 사용하고, A100/H100은 대형 모델 전용으로 제한합니다.

---

**Q27. MLflow 실험 추적 및 아티팩트 저장소의 비용을 최적화하는 방법을 설명하세요.**

**A27.** MLflow의 비용은 크게 아티팩트 스토리지(S3/Azure Blob), 추적 서버 컴퓨트, 데이터베이스(RDS/PostgreSQL) 세 부분으로 구성됩니다.

**아티팩트 비용 최적화**가 가장 큰 절감 기회입니다. 모델 체크포인트가 매 에포크마다 저장되면 S3 비용이 빠르게 증가합니다. MLflow에 수명 주기 정책을 적용하여 best N개의 체크포인트만 유지하고 나머지는 자동 삭제합니다. 또한 대형 모델은 S3 Glacier로 아카이빙하고, 현재 사용 중인 모델만 S3 Standard에 유지합니다.

**실험 데이터 정리**도 중요합니다. 팀이 많은 실험을 실행할수록 메타데이터가 축적되어 데이터베이스 비용이 증가합니다. MLflow의 `mlflow gc` 명령어로 삭제된 실험/런의 아티팩트를 정기적으로 정리하고, 30일 이상 된 Failed 실험은 자동 삭제하는 정책을 적용합니다.

**추적 서버 Right-sizing**은 MLflow 서버 자체가 트래픽이 많지 않다면 소형 인스턴스로 충분합니다. Fargate나 Lambda URL 기반의 서버리스 배포를 검토하면 항시 실행 비용을 없앨 수 있습니다.

---

**Q28. SageMaker Pipeline과 Kubeflow Pipeline 중 어떤 것을 선택하겠습니까? 비용 측면을 포함한 트레이드오프를 분석하세요.**

**A28.** 선택은 조직의 클라우드 전략, 팀 역량, 워크로드 특성에 따라 다릅니다.

**SageMaker Pipeline**의 비용 구조는 파이프라인 실행 자체에는 별도 비용이 없고, 각 스텝에서 사용하는 컴퓨트(EC2/Fargate)와 스토리지만 과금됩니다. AWS 서비스와 깊은 통합(S3, ECR, IAM, CloudWatch)이 되어 있어 운영 오버헤드가 낮습니다. 단, AWS에 종속되며 Kubernetes 환경과의 통합이 제한적입니다.

**Kubeflow Pipeline**의 비용 구조는 Pipeline Controller, MinIO(아티팩트 스토어), MySQL(메타데이터)을 직접 운영해야 합니다. 이 컴포넌트들이 상시 실행되므로 소규모 환경에서는 오히려 더 비쌀 수 있습니다. 그러나 멀티클라우드, 온프레미스, Spark 등 다양한 컴퓨트 백엔드를 지원하고 vendor lock-in이 없습니다.

**비용 최적화 관점 권고**로는, AWS 중심 인프라에서 데이터 과학팀의 규모가 크지 않다면 SageMaker Pipeline이 TCO 측면에서 유리합니다. 멀티클라우드 전략이 있거나 이미 Kubernetes 운영 역량이 있는 조직은 Kubeflow가 장기적으로 더 비용 효율적입니다.

---

**Q29. GPU 인스턴스를 AWS Spot으로 운영하는 ML 학습 파이프라인을 설계할 때 체크포인팅 전략을 어떻게 구현하겠습니까?**

**A29.** GPU Spot 학습 파이프라인의 핵심은 인터럽트를 최소한의 진행 손실로 처리하는 것입니다.

**체크포인트 빈도 결정**은 비용-복구 트레이드오프입니다. 너무 자주 저장하면 I/O 오버헤드로 학습이 느려지고, 너무 드물게 저장하면 인터럽트 시 손실이 큽니다. 경험적으로 10-20분 주기가 적절하며, S3 Multi-part Upload를 사용하여 대형 체크포인트를 병렬로 업로드합니다.

**인터럽트 감지 및 처리**는 AWS EC2 Instance Metadata Service(IMDS)의 인터럽트 경고를 폴링합니다. `http://169.254.169.254/latest/meta-data/spot/termination-time` 엔드포인트가 응답하면 2분 이내 인터럽트가 예정된 것입니다. 이 시점에 현재 배치 완료 후 즉시 체크포인트를 저장하고 그레이스풀 종료합니다.

```python
import signal, boto3, time, threading

class SpotInterruptionHandler:
    def __init__(self, model, optimizer, save_path: str):
        self.model = model
        self.optimizer = optimizer
        self.s3 = boto3.client('s3')
        self.save_path = save_path
        self._check_thread = threading.Thread(target=self._poll_interruption, daemon=True)
        self._check_thread.start()
    
    def _poll_interruption(self):
        import urllib.request
        while True:
            try:
                urllib.request.urlopen(
                    "http://169.254.169.254/latest/meta-data/spot/termination-time",
                    timeout=1
                )
                # 응답이 오면 인터럽트 예정
                print("Spot interruption detected! Saving checkpoint...")
                self.save_checkpoint(is_emergency=True)
                break
            except:
                pass
            time.sleep(5)
    
    def save_checkpoint(self, epoch: int = 0, step: int = 0, is_emergency: bool = False):
        import torch
        checkpoint = {
            'epoch': epoch,
            'step': step,
            'model_state': self.model.state_dict(),
            'optimizer_state': self.optimizer.state_dict(),
        }
        local_path = f"/tmp/checkpoint_e{epoch}_s{step}.pt"
        torch.save(checkpoint, local_path)
        # S3에 업로드
        self.s3.upload_file(local_path, 'my-training-bucket', 
                           f"{self.save_path}/checkpoint_e{epoch}_s{step}.pt")
        if is_emergency:
            # latest 포인터 업데이트
            self.s3.put_object(Bucket='my-training-bucket',
                              Key=f"{self.save_path}/latest.txt",
                              Body=f"checkpoint_e{epoch}_s{step}.pt")
```

SageMaker의 `checkpoint_s3_uri`를 사용하면 이 로직을 프레임워크 수준에서 자동화할 수 있습니다.

---

**Q30. Nvidia의 관점에서, GPU 클러스터의 에너지 효율(Performance per Watt)을 최적화하는 방법을 설명하세요. 이는 비용과 지속가능성에 어떻게 연결됩니까?**

**A30.** GPU의 에너지 효율 최적화는 운영 비용(전력 비용)과 직결되며, 대규모 GPU 클러스터에서는 하드웨어 비용보다 전력 비용이 더 클 수 있습니다.

**GPU Power Management**로 MIG나 time-slicing 환경에서 GPU Power Limit을 활용합니다. NVIDIA Management Library(NVML)의 `nvidia-smi -pl <watts>`로 TDP의 70-80% 수준으로 전력 한계를 설정하면, 성능 손실 5-10% 이내로 에너지 소비를 20-30% 줄일 수 있습니다. 이는 수백 개의 GPU 클러스터에서 수십만 달러의 연간 절감으로 이어집니다.

**Mixed Precision 학습(FP16/BF16)**은 FP32 대비 같은 학습 시간 동안 2배 많은 연산을 수행하므로, 단위 성능당 에너지 소비가 크게 낮아집니다. H100의 경우 BF16 Tensor Core가 FP32 대비 6-7배 높은 처리량을 제공합니다.

**클라우드 리전 선택**도 지속가능성에 영향을 미칩니다. AWS의 us-west-2(Oregon), Azure의 Sweden Central은 재생 에너지 비율이 높아 탄소 발자국이 적고, Carbon Credit이 있는 조직은 이런 리전 선택을 의무화하는 정책을 적용합니다.

---

### [멀티클라우드 & 아키텍처 — 5문항]

---

**Q31. Google Cloud의 관점에서, AWS와 GCP를 동시에 사용하는 멀티클라우드 환경에서 데이터 전송 비용을 최소화하는 아키텍처를 설계하세요.**

**A31.** 멀티클라우드 데이터 전송 비용은 아키텍처 결정에 따라 극적으로 달라집니다. 핵심 원칙은 **데이터를 계산이 있는 곳으로 이동하되, 필요한 최소 데이터만 이동**하는 것입니다.

**Cloud Interconnect / ExpressRoute 활용**이 첫 번째 전략입니다. AWS Direct Connect와 GCP Cloud Interconnect를 전용 회선으로 연결하면 인터넷 이그레스 요금($0.09/GB) 대신 훨씬 저렴한 전용 회선 전송 요금을 적용받습니다. 월간 데이터 전송량이 TB 단위 이상이면 ROI가 확실합니다.

**데이터 위치성(Locality) 원칙**이 두 번째입니다. 학습 데이터가 AWS S3에 있다면 모델 학습도 AWS에서, 배포 목표 사용자가 GCP 리전에 있다면 서빙은 GCP에서 하는 방식으로 데이터와 컴퓨트를 같은 클라우드에 배치합니다.

**CQRS 패턴 적용**이 세 번째입니다. 쓰기는 단일 클라우드(master)에서 처리하고, 읽기 전용 복제본을 다른 클라우드에 유지합니다. 양방향 동기화는 전송 비용이 배증하므로 지양합니다.

---

**Q32. Apple의 관점에서, 사용자 데이터 프라이버시를 보장하면서 클라우드 비용을 최적화하는 방법을 설명하세요.**

**A32.** 프라이버시와 비용 최적화는 상충 관계처럼 보이지만, 잘 설계하면 함께 달성할 수 있습니다.

**On-device First 아키텍처**는 프라이버시를 보장하면서 클라우드 비용도 줄이는 전략입니다. 가능한 많은 추론과 처리를 사용자 기기에서 수행하면 클라우드 전송 및 처리 비용이 근본적으로 줄어듭니다. Apple Silicon의 Neural Engine이 이를 가능하게 하며, CoreML 모델 최적화를 통해 클라우드에 의존했던 기능을 온디바이스로 이전합니다.

**Differential Privacy**를 적용한 데이터 수집은 개인 식별이 불가능한 집계 데이터만 클라우드로 전송하므로, 원시 데이터 저장 비용과 GDPR 컴플라이언스 비용을 동시에 줄입니다.

**Private Cloud Compute(PCC)** 아키텍처는 클라우드 처리가 필요할 때 처리 결과만 반환하고 원시 데이터는 저장하지 않는 무상태(stateless) 처리를 기본으로 합니다. 이는 스토리지 비용 최소화와 프라이버시 보장을 동시에 달성합니다.

---

**Q33. Tesla의 관점에서, 대규모 자율주행 데이터(페타바이트급) 파이프라인의 비용 최적화 방법을 설명하세요.**

**A33.** 페타바이트급 자율주행 데이터 파이프라인의 비용은 스토리지, 처리, 레이블링 세 부분으로 구성되며, 각각 다른 전략이 필요합니다.

**계층적 스토리지 전략**이 핵심입니다. 모든 주행 데이터를 동일한 스토리지 클래스에 저장하는 것은 비효율적입니다. 최근 데이터(Hot): S3 Standard, 학습에 사용된 적 있는 데이터(Warm): S3 Standard-IA, 아카이브 데이터(Cold): S3 Glacier 계층으로 자동 전환합니다. 페타바이트 환경에서 이 계층화만으로 연간 수천만 달러의 절감이 가능합니다.

**데이터 선택 전략(Data Selection)**은 모든 주행 데이터가 동등하게 가치 있지 않다는 인식에서 출발합니다. 엣지 케이스(Edge Cases)와 시나리오 다양성이 높은 데이터만 선택적으로 레이블링하고 학습에 사용하는 Curriculum Learning 접근이 레이블링 비용을 크게 줄입니다.

**분산 처리 비용 최적화**로는 Apache Spark on Kubernetes(Spot 인스턴스)를 활용한 데이터 전처리 파이프라인을 구성합니다. Ray를 활용한 분산 추론과 Dojo(자체 학습 슈퍼컴퓨터)로의 학습 워크로드 이전도 장기적 비용 절감 전략입니다.

---

**Q34. Amazon 내부 관점에서, Black Friday와 같은 예측 가능한 트래픽 스파이크를 위한 비용 최적화 전략을 설명하세요.**

**A34.** 예측 가능한 트래픽 스파이크에 대한 최적화는 예측 불가능한 스파이크와 전략이 다릅니다. 사전 준비가 가능하므로, 비용 효율적인 예열(Pre-warming) 전략이 핵심입니다.

**용량 예약(Capacity Reservation) + Savings Plans 조합**을 사용합니다. 이벤트 1-2주 전에 On-Demand Capacity Reservation을 특정 AZ에 예약하여 용량 확보를 보장합니다. 이와 별개로 평소 기저 수요에는 Savings Plans를 유지하고, 스파이크 기간에는 예약된 온디맨드를 추가로 활용합니다.

**Pre-scaling 자동화**는 Historical 패턴 기반의 예측 스케일링을 EventBridge Scheduler로 구현합니다. 단순히 사전에 인스턴스를 더 추가하는 것이 아니라, 서비스별 응답 시간과 학습된 트래픽 패턴을 바탕으로 최소 필요 용량을 정확히 계산합니다.

**캐싱 최대화**는 트래픽 증가분의 실제 백엔드 도달을 줄이는 가장 비용 효율적인 전략입니다. CloudFront의 Cache Hit Ratio를 높이고, ElastiCache의 연결 풀링을 최적화하여 DB 부하를 최소화합니다. 이벤트 기간 동안 스태틱 자산의 Cache TTL을 평소보다 높게 설정합니다.

---

**Q35. Microsoft Azure에서 대규모 AI 워크로드를 운영할 때 Azure OpenAI Service와 자체 호스팅 모델 간의 비용 트레이드오프를 어떻게 분석하겠습니까?**

**A35.** Azure OpenAI Service(AOAI)와 자체 호스팅의 선택은 규모, 품질 요구사항, 운영 역량에 따라 달라집니다.

**Azure OpenAI Service**는 토큰당 과금($0.002-0.06/1K tokens) 모델입니다. 초기 인프라 비용이 없고 운영 오버헤드가 최소화됩니다. 그러나 요청량이 증가할수록 선형적으로 비용이 증가하고, 월 수천만 토큰 이상의 규모에서는 자체 호스팅이 경제적일 수 있습니다. AOAI의 Provisioned Throughput(PTU)는 월정액으로 일정 처리량을 보장받는 모델로, 높은 트래픽에서 토큰 과금보다 저렴합니다.

**자체 호스팅(Llama 3, Mistral 등)**은 GPU 인프라 비용이 고정적으로 발생합니다. A100 80GB GPU 1개의 Azure 비용은 약 $3-4/hour로, 월 $2,160-2,880입니다. 이 GPU가 GPT-3.5 수준의 모델을 서빙한다면 시간당 수백만 토큰 처리가 가능하므로, 높은 볼륨에서는 AOAI 대비 10-100배 저렴해집니다.

**Break-even 분석**: 월 토큰 소비량이 GPT-3.5 turbo 기준으로 약 5억 토큰($1,000) 이상이면 자체 호스팅 검토가 의미 있습니다. 단, 운영 인건비, GPU 장애 처리, 모델 업데이트 비용을 TCO에 반드시 포함해야 합니다. 대부분의 기업에서는 AOAI로 시작하여 볼륨이 일정 수준을 넘을 때 자체 호스팅으로 전환하는 하이브리드 전략이 최적입니다.

---

## 결론: 비용 최적화의 문화적 차원

기술적 최적화 도구와 전략을 아무리 갖춰도, 조직 문화가 뒷받침되지 않으면 지속 가능하지 않습니다. 가장 성숙한 FinOps 조직들은 **엔지니어링 팀이 자신이 사용하는 클라우드 비용에 직접 책임**을 지는 구조를 만들었습니다. 이는 처벌이 아니라, 팀이 비용 데이터에 접근하고, 최적화 권한을 갖고, 절감 성과를 인정받는 인센티브 구조를 통해 달성됩니다.

결국 최고의 클라우드 비용 최적화는 기술과 문화가 함께 발전하는 선순환에서 나옵니다.

---

*문서 버전: 2025.02 | 작성 기준: Kubernetes 1.29+, AWS SDK Python 1.34+, Azure SDK Python 1.21+*
