# PyTorch 완전 가이드: 기본 개념부터 NVIDIA DGX Spark 실습까지

> **강의 대상**: AI/ML 엔지니어, MLOps/LLMOps 실무자, 클라우드 엔지니어  
> **최종 업데이트**: 2026년 4월

---

## 1장. PyTorch란 무엇인가

### 1.1 PyTorch의 정의와 탄생 배경

PyTorch는 Meta(구 Facebook) AI Research 팀이 2016년에 공개한 오픈소스 딥러닝 프레임워크입니다. 기존의 Torch(Lua 기반) 프로젝트를 Python 생태계로 가져오면서, Python 개발자들이 친숙하게 사용할 수 있는 형태로 재탄생시켰습니다.

PyTorch가 등장하기 전에는 Google의 TensorFlow가 사실상 표준이었습니다. 그러나 TensorFlow 1.x의 정적 계산 그래프(Static Computation Graph) 방식은 디버깅이 어렵고, 연구 목적의 빠른 프로토타이핑에 적합하지 않았습니다. PyTorch는 이 문제를 **동적 계산 그래프(Dynamic Computation Graph)** 방식으로 해결했습니다. Python의 제어 흐름(if, for, while)을 그대로 사용하면서 자동 미분이 가능하다는 점이 연구자들 사이에서 폭발적인 인기를 끌었고, 현재는 학계와 산업계 모두에서 가장 널리 사용되는 딥러닝 프레임워크로 자리잡았습니다.

### 1.2 PyTorch의 핵심 설계 철학

PyTorch의 설계 철학은 세 가지 키워드로 요약됩니다.

첫째, **Pythonic**입니다. PyTorch는 Python의 관용적 표현을 최대한 존중합니다. NumPy와 거의 동일한 API를 제공하며, Python 클래스 상속을 통해 모델을 정의합니다. NumPy를 사용해본 개발자라면 학습 곡선이 매우 낮습니다.

둘째, **Imperative(명령형) 실행**입니다. 코드를 한 줄씩 실행하면서 중간 결과를 즉시 확인할 수 있습니다. 이를 "Eager Execution"이라 하며, 디버거(pdb, breakpoint)를 그대로 활용할 수 있어 개발 생산성이 높습니다.

셋째, **확장성**입니다. 단일 GPU부터 수천 개의 GPU 클러스터까지 동일한 코드 베이스로 확장 가능하며, C++ 확장, CUDA 커스텀 커널, 그리고 다양한 하드웨어 백엔드를 지원합니다.

---

## 2장. PyTorch 핵심 개념 심화

### 2.1 Tensor — 모든 것의 기본 단위

Tensor는 PyTorch의 가장 기본적인 데이터 구조입니다. 수학적으로는 다차원 배열(multidimensional array)이며, NumPy의 ndarray와 개념적으로 동일하지만 GPU 가속을 네이티브로 지원합니다.

```python
import torch

# 다양한 방법으로 텐서 생성
x = torch.tensor([1.0, 2.0, 3.0])                  # 리스트로부터
y = torch.zeros(3, 4)                                # 0으로 초기화된 3x4 행렬
z = torch.randn(2, 3, 4)                             # 정규분포 난수, 2x3x4 텐서
w = torch.arange(0, 10, step=2)                      # [0, 2, 4, 6, 8]

# GPU로 이동 (CUDA 사용 가능 시)
if torch.cuda.is_available():
    x_gpu = x.to('cuda')       # 또는 x.cuda()
    print(x_gpu.device)        # cuda:0

# dtype 제어
x_fp16 = x.to(dtype=torch.float16)
x_bf16 = x.to(dtype=torch.bfloat16)   # Blackwell GPU에서 효율적
```

Tensor의 핵심 속성 세 가지를 기억해야 합니다. `shape`(차원과 크기), `dtype`(데이터 타입: float32, float16, bfloat16, int8 등), `device`(cpu 또는 cuda:N)입니다. 이 세 속성을 명확히 관리하는 것이 PyTorch 프로그래밍의 기본입니다.

### 2.2 Autograd — 자동 미분 엔진

딥러닝의 핵심은 역전파(Backpropagation)이고, 역전파의 핵심은 미분입니다. PyTorch의 Autograd 시스템은 텐서 연산의 계산 그래프를 자동으로 구성하고, `backward()`를 호출하면 그래디언트를 자동으로 계산합니다.

```python
# requires_grad=True로 설정하면 해당 텐서에 대한 미분을 추적
x = torch.tensor(3.0, requires_grad=True)
y = x ** 2 + 2 * x + 1    # y = x² + 2x + 1

y.backward()                # dy/dx 계산
print(x.grad)               # tensor(8.) → 2*3 + 2 = 8
```

**동적 계산 그래프의 의미**: Autograd는 forward pass가 실행될 때마다 새로운 계산 그래프를 생성합니다. 이는 매 반복(iteration)마다 다른 구조의 네트워크를 사용할 수 있다는 의미입니다. 자연어 처리에서 가변 길이 시퀀스를 처리하거나, 조건부 분기가 있는 모델을 구현할 때 이 특성이 큰 장점이 됩니다.

**그래디언트 컨텍스트 관리**는 실무에서 매우 중요합니다:

```python
# 추론(inference) 시에는 그래디언트 추적을 끔 → 메모리 절약 + 속도 향상
with torch.no_grad():
    output = model(input_data)

# 또는 데코레이터 방식
@torch.inference_mode()
def predict(model, data):
    return model(data)
```

### 2.3 nn.Module — 모델 설계의 기본 단위

`torch.nn.Module`은 PyTorch에서 모든 신경망 모델의 기본 클래스입니다. 모델을 설계한다는 것은 곧 `nn.Module`을 상속받는 클래스를 작성하는 것입니다.

```python
import torch.nn as nn
import torch.nn.functional as F

class SimpleClassifier(nn.Module):
    def __init__(self, input_dim, hidden_dim, num_classes):
        super().__init__()
        # 레이어 정의
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.bn1 = nn.BatchNorm1d(hidden_dim)
        self.dropout = nn.Dropout(p=0.3)
        self.fc2 = nn.Linear(hidden_dim, num_classes)
    
    def forward(self, x):
        # 순전파 로직 정의
        x = self.fc1(x)
        x = self.bn1(x)
        x = F.relu(x)
        x = self.dropout(x)
        x = self.fc2(x)
        return x

# 모델 인스턴스 생성 및 GPU 이동
model = SimpleClassifier(784, 256, 10).to('cuda')

# 파라미터 수 확인
total_params = sum(p.numel() for p in model.parameters())
print(f"Total parameters: {total_params:,}")
```

`nn.Module`의 핵심 메서드들을 정리하면 다음과 같습니다. `forward()`는 순전파 로직을 정의합니다. `parameters()`는 학습 가능한 모든 파라미터를 반환합니다. `state_dict()`는 모델의 가중치를 딕셔너리 형태로 반환하며 저장/로드에 사용됩니다. `train()`과 `eval()`은 학습/추론 모드를 전환하며, Dropout이나 BatchNorm의 동작을 제어합니다. `to(device)`는 모델 전체를 특정 디바이스로 이동시킵니다.

### 2.4 DataLoader — 효율적인 데이터 파이프라인

실제 학습에서는 대량의 데이터를 미니배치(mini-batch) 단위로 나누어 모델에 공급해야 합니다. `Dataset`과 `DataLoader`가 이 역할을 담당합니다.

```python
from torch.utils.data import Dataset, DataLoader

class CustomDataset(Dataset):
    def __init__(self, data, labels, transform=None):
        self.data = data
        self.labels = labels
        self.transform = transform
    
    def __len__(self):
        return len(self.data)
    
    def __getitem__(self, idx):
        sample = self.data[idx]
        label = self.labels[idx]
        if self.transform:
            sample = self.transform(sample)
        return sample, label

# DataLoader: 배치 단위 공급, 셔플, 병렬 로딩
train_loader = DataLoader(
    dataset=train_dataset,
    batch_size=64,
    shuffle=True,
    num_workers=4,        # 멀티프로세스 데이터 로딩
    pin_memory=True,      # GPU 전송 속도 향상
    drop_last=True        # 마지막 불완전 배치 제거
)
```

**DGX Spark에서의 주의사항**: DGX Spark의 통합 메모리 아키텍처에서는 `num_workers` 설정에 주의해야 합니다. 워커 프로세스가 fork될 때 전체 프로세스 상태를 복제하므로, 대형 모델이 이미 메모리에 로드된 상태에서는 메모리가 빠르게 소진될 수 있습니다. 이 경우 `num_workers=0`으로 설정하여 메인 프로세스에서 데이터를 로딩하는 것이 안전합니다.

### 2.5 학습 루프(Training Loop)의 구조

PyTorch의 학습 루프는 명시적(explicit)입니다. 프레임워크가 학습 과정을 숨기지 않고, 개발자가 모든 단계를 직접 제어합니다. 이것이 PyTorch의 가장 큰 장점이자, 처음 접할 때 약간의 진입 장벽이 되기도 합니다.

```python
# 손실 함수와 옵티마이저 정의
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=100)

# 학습 루프
model.train()
for epoch in range(num_epochs):
    running_loss = 0.0
    for batch_idx, (inputs, targets) in enumerate(train_loader):
        inputs, targets = inputs.to('cuda'), targets.to('cuda')
        
        # 1) 그래디언트 초기화
        optimizer.zero_grad()
        
        # 2) 순전파 (Forward Pass)
        outputs = model(inputs)
        
        # 3) 손실 계산
        loss = criterion(outputs, targets)
        
        # 4) 역전파 (Backward Pass)
        loss.backward()
        
        # 5) 가중치 업데이트
        optimizer.step()
        
        running_loss += loss.item()
    
    scheduler.step()
    avg_loss = running_loss / len(train_loader)
    print(f"Epoch [{epoch+1}/{num_epochs}], Loss: {avg_loss:.4f}")
```

이 5단계(zero_grad → forward → loss → backward → step)는 PyTorch 학습의 "표준 패턴"입니다. 모든 PyTorch 프로젝트에서 이 구조를 반복하게 되며, 이 기본 구조 위에 Mixed Precision Training, Gradient Accumulation, Distributed Training 등의 고급 기법을 쌓아 올립니다.

### 2.6 모델 저장과 로드

학습된 모델을 저장하고 다시 불러오는 것은 운영(Production) 환경에서 필수적인 작업입니다.

```python
# 방법 1: state_dict 저장 (권장)
torch.save(model.state_dict(), 'model_weights.pth')

# 복원
model = SimpleClassifier(784, 256, 10)
model.load_state_dict(torch.load('model_weights.pth', map_location='cuda'))
model.eval()

# 방법 2: 체크포인트 저장 (학습 재개용)
checkpoint = {
    'epoch': epoch,
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
    'loss': loss,
}
torch.save(checkpoint, 'checkpoint.pth')

# 방법 3: TorchScript로 내보내기 (배포용)
scripted_model = torch.jit.script(model)
scripted_model.save('model_scripted.pt')
```

MLOps 관점에서는 체크포인트 저장이 특히 중요합니다. 대형 모델 학습이 중단되었을 때 처음부터 다시 시작하지 않고 마지막 체크포인트에서 재개할 수 있기 때문입니다.

---

## 3장. PyTorch 고급 개념

### 3.1 Mixed Precision Training (혼합 정밀도 학습)

현대 GPU(특히 Tensor Core 탑재 GPU)에서는 FP32 대신 FP16이나 BF16을 사용하면 학습 속도를 크게 향상시킬 수 있습니다. PyTorch의 `torch.amp`(Automatic Mixed Precision) 모듈이 이를 간편하게 지원합니다.

```python
from torch.amp import autocast, GradScaler

scaler = GradScaler('cuda')

for inputs, targets in train_loader:
    inputs, targets = inputs.to('cuda'), targets.to('cuda')
    optimizer.zero_grad()
    
    # autocast 블록 내에서 자동으로 FP16/BF16 적용
    with autocast('cuda'):
        outputs = model(inputs)
        loss = criterion(outputs, targets)
    
    # GradScaler로 loss를 스케일링하여 FP16 언더플로 방지
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

DGX Spark의 Blackwell GPU는 5세대 Tensor Core를 탑재하여 FP4 연산까지 지원합니다. FP4는 추론(inference) 시 특히 유용하며, 이론상 최대 1 PFLOP의 성능을 제공합니다. 학습 시에는 BF16이 가장 범용적으로 사용됩니다.

### 3.2 Distributed Training (분산 학습)

단일 GPU의 메모리나 연산 능력이 부족할 때는 여러 GPU에 학습을 분산시킵니다. PyTorch는 `DistributedDataParallel(DDP)`을 통해 이를 지원합니다.

```python
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

# 분산 환경 초기화
dist.init_process_group(backend='nccl')
local_rank = int(os.environ['LOCAL_RANK'])
torch.cuda.set_device(local_rank)

# 모델을 DDP로 래핑
model = model.to(local_rank)
model = DDP(model, device_ids=[local_rank])

# 학습 루프는 동일 — DDP가 그래디언트 동기화를 자동 처리
```

DGX Spark는 200Gbps ConnectX-7 QSFP 이더넷 포트를 내장하고 있어, 두 대의 Spark를 직접 연결하여 256GB의 통합 메모리 풀에서 최대 405B 파라미터 모델을 분산 추론할 수 있습니다.

### 3.3 FSDP — 초대형 모델 학습

Fully Sharded Data Parallel(FSDP)은 모델 파라미터, 그래디언트, 옵티마이저 상태를 모든 GPU에 분산(sharding)하여 메모리 효율을 극대화하는 기법입니다.

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP

model = FSDP(model, auto_wrap_policy=size_based_auto_wrap_policy)
```

### 3.4 torch.compile — JIT 컴파일을 통한 최적화

PyTorch 2.0에서 도입된 `torch.compile`은 모델의 계산 그래프를 분석하고 최적화된 커널을 생성하여 실행 속도를 향상시킵니다.

```python
# 한 줄로 모델 최적화
compiled_model = torch.compile(model, mode='reduce-overhead')

# 이후 사용법은 동일
output = compiled_model(input_data)
```

`mode` 옵션으로 `default`(범용), `reduce-overhead`(오버헤드 최소화), `max-autotune`(최대 성능 탐색) 중 선택할 수 있습니다. 특히 추론 시 `max-autotune` 모드는 Triton 커널을 자동 생성하여 상당한 성능 향상을 제공합니다.

---

## 4장. PyTorch 생태계와 실전 활용

### 4.1 Hugging Face Transformers + PyTorch

현대 NLP와 LLM 개발에서 Hugging Face Transformers 라이브러리는 사실상의 표준입니다. 이 라이브러리는 PyTorch를 기본 백엔드로 사용합니다.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "meta-llama/Llama-3-8B"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.bfloat16,
    device_map="auto"           # 자동으로 GPU 할당
)

inputs = tokenizer("PyTorch는", return_tensors="pt").to('cuda')
outputs = model.generate(**inputs, max_new_tokens=100)
print(tokenizer.decode(outputs[0]))
```

### 4.2 파인튜닝(Fine-tuning) 기법

사전 학습된 대형 모델을 특정 도메인이나 태스크에 맞게 조정하는 파인튜닝은 현대 AI 개발의 핵심 워크플로입니다.

**Full Fine-tuning**: 모델의 모든 파라미터를 업데이트합니다. 가장 높은 성능을 얻을 수 있지만 메모리와 연산량이 많이 필요합니다. DGX Spark의 128GB 통합 메모리에서는 최대 70B 파라미터 모델까지 Full Fine-tuning이 가능합니다.

**LoRA (Low-Rank Adaptation)**: 원본 가중치를 동결하고, 작은 저차원(low-rank) 어댑터만 학습합니다. 메모리 사용량을 대폭 줄이면서도 풀 파인튜닝에 근접하는 성능을 얻을 수 있습니다.

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,                           # 랭크
    lora_alpha=32,                  # 스케일링 팩터
    target_modules=["q_proj", "v_proj"],  # 적용 대상 모듈
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

peft_model = get_peft_model(model, lora_config)
peft_model.print_trainable_parameters()
# → trainable params: 4,194,304 || all params: 8,030,261,248 || 0.05%
```

**QLoRA (Quantized LoRA)**: 기본 모델을 4비트(INT4)로 양자화한 뒤 LoRA를 적용합니다. 메모리 사용량을 더욱 절감할 수 있어, DGX Spark에서 70B 모델 파인튜닝이 실질적으로 가능해집니다.

### 4.3 Computer Vision — torchvision

이미지 분류, 객체 탐지, 세그멘테이션 등 컴퓨터 비전 태스크에서 PyTorch는 `torchvision` 라이브러리와 함께 사용됩니다.

```python
import torchvision.models as models
import torchvision.transforms as transforms

# 사전 학습된 ResNet 불러오기
model = models.resnet50(weights=models.ResNet50_Weights.DEFAULT)
model.fc = nn.Linear(2048, num_custom_classes)  # 출력층 교체
model = model.to('cuda')

# 이미지 전처리 파이프라인
transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),
])
```

### 4.4 실전 활용 사례

**사례 1 — 대규모 언어 모델(LLM) 개발**: OpenAI의 GPT 시리즈, Meta의 LLaMA 시리즈, Google의 Gemma 등 거의 모든 주요 LLM이 PyTorch로 학습됩니다. 최근에는 DeepSeek, Qwen 등 오픈소스 모델들도 PyTorch 기반으로 학습 및 공개되고 있습니다.

**사례 2 — 자율주행**: Tesla의 Autopilot, Waymo의 인식 시스템 등에서 PyTorch가 사용됩니다. 3D 포인트 클라우드 처리, 다중 카메라 퓨전, 실시간 경로 예측 등의 모델이 PyTorch로 구현됩니다.

**사례 3 — 의료 AI**: 의료 영상(X-ray, CT, MRI) 분석, 약물 분자 구조 예측, 유전체 시퀀싱 분석 등에서 PyTorch가 활발히 사용됩니다. DGX Spark의 공식 플레이북에도 단일 세포 RNA 시퀀싱(Single-cell RNA Sequencing) 예제가 포함되어 있습니다.

**사례 4 — 추천 시스템**: Meta의 DLRM(Deep Learning Recommendation Model), Netflix와 Spotify의 개인화 추천 엔진 등이 PyTorch를 활용합니다.

**사례 5 — 로보틱스 및 엣지 AI**: NVIDIA Isaac, Metropolis 프레임워크와 결합하여 로봇 제어, 스마트 시티, 영상 분석 등의 엣지 AI 애플리케이션을 개발할 수 있습니다.

---

## 5장. NVIDIA DGX Spark — 데스크톱 AI 슈퍼컴퓨터

### 5.1 DGX Spark 하드웨어 개요

NVIDIA DGX Spark는 2025년 3월 GTC에서 발표되어 같은 해 10월에 출시된 개인용 AI 슈퍼컴퓨터입니다. 데이터센터급 AI 성능을 150mm × 150mm × 50.5mm의 초소형 폼팩터에 담았습니다.

핵심 사양은 다음과 같습니다:

| 항목 | 사양 |
|------|------|
| SoC | NVIDIA GB10 Grace Blackwell Superchip |
| CPU | 20코어 ARM (10× Cortex-X925 + 10× Cortex-A725) |
| GPU | Blackwell 아키텍처, 5세대 Tensor Core |
| 메모리 | 128GB LPDDR5x 통합 메모리 (CPU/GPU 공유) |
| 스토리지 | 4TB NVMe SSD |
| AI 성능 | 최대 1 PFLOP (FP4 Sparse) |
| 네트워크 | ConnectX-7 200GbE QSFP 듀얼 포트 |
| 전원 | USB-C, 240W 외부 PSU |
| 운영체제 | DGX OS (Ubuntu 기반) |
| 가격 | $3,999~$4,699 (구성에 따라 상이) |

### 5.2 DGX Spark가 PyTorch 개발자에게 의미하는 것

DGX Spark의 가장 큰 특징은 **128GB 통합 메모리**입니다. 일반적인 GPU(RTX 4090의 24GB VRAM 등)와 달리, CPU와 GPU가 동일한 메모리 풀을 공유합니다. 이것은 실무에서 매우 큰 차이를 만들어냅니다.

일반적으로 FP16 정밀도에서 70B 파라미터 모델을 로드하려면 약 140GB의 GPU 메모리가 필요합니다. 기존에는 H100 80GB SXM 두 대(약 $60,000~$70,000 이상)를 NVLink로 연결해야 했지만, DGX Spark에서는 $4,000 내외의 단일 장치로 동일한 모델을 구동할 수 있습니다. 물론 연산 대역폭은 데이터센터급 GPU에 비해 낮지만, 프로토타이핑과 소규모 파인튜닝, 추론 용도로는 충분합니다.

또한 모든 데이터를 로컬에서 처리할 수 있으므로, 환자 데이터, 독점 코드, 미공개 모델 가중치 등 민감한 데이터를 다루는 환경에서 클라우드 대비 보안 이점이 있습니다.

### 5.3 아키텍처 특성과 주의사항

DGX Spark는 **ARM64(aarch64) 아키텍처**를 사용합니다. 대부분의 ML 생태계가 x86_64를 기준으로 구축되었기 때문에, 일부 패키지에서 호환성 이슈가 발생할 수 있습니다. GPU의 CUDA Compute Capability는 sm_121(Blackwell)이며, PyTorch에서는 sm_120 타깃이 바이너리 호환됩니다.

주요 주의사항으로는 다음이 있습니다. 첫째, 안정 버전보다 Nightly 빌드나 NVIDIA 공식 컨테이너가 더 잘 호환될 수 있습니다. 둘째, 일부 서드파티 패키지(특히 C++ 확장이 포함된 것들)는 ARM64용 바이너리가 없어 소스에서 직접 빌드해야 할 수 있습니다. 셋째, 통합 메모리 구조에서 `num_workers`가 높으면 메모리 오버헤드가 급증할 수 있습니다.

---

## 6장. DGX Spark에서 PyTorch 실습

### 6.1 환경 설정 — Docker 컨테이너 방식 (권장)

NVIDIA는 DGX Spark에 최적화된 공식 PyTorch 컨테이너를 제공합니다. Docker를 사용하면 ARM64 호환성, CUDA 드라이버, cuDNN 등의 복잡한 설정을 건너뛸 수 있습니다.

```bash
# 1. Docker 권한 확인
docker ps
# permission denied 에러 발생 시:
sudo usermod -aG docker $USER
newgrp docker

# 2. NVIDIA PyTorch 컨테이너 풀
docker pull nvcr.io/nvidia/pytorch:25.11-py3

# 3. 컨테이너 실행
docker run --gpus all -it --rm --ipc=host \
  -v $HOME/.cache/huggingface:/root/.cache/huggingface \
  -v ${PWD}:/workspace -w /workspace \
  nvcr.io/nvidia/pytorch:25.11-py3

# 4. 컨테이너 내부에서 추가 패키지 설치
pip install transformers peft datasets trl bitsandbytes
```

각 옵션의 의미를 이해하는 것이 중요합니다. `--gpus all`은 컨테이너에서 GPU에 접근할 수 있게 합니다. `--ipc=host`는 호스트와 공유 메모리를 공유하여 DataLoader의 멀티프로세스 로딩에 필수적입니다. `-v` 옵션은 호스트 디렉토리를 컨테이너에 마운트하여 Hugging Face 캐시를 재사용합니다.

### 6.2 환경 설정 — 네이티브 pip 방식

Docker 없이 직접 설치하려면 CUDA 호환 PyTorch를 설치해야 합니다.

```bash
# 가상환경 생성
python3 -m venv ~/pytorch-env
source ~/pytorch-env/bin/activate

# CUDA 13 호환 PyTorch 설치
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130

# GPU 인식 확인
python3 -c "
import torch
print(f'PyTorch version: {torch.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')
print(f'CUDA version: {torch.version.cuda}')
print(f'GPU: {torch.cuda.get_device_name(0)}')
print(f'Memory: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB')
"
```

### 6.3 실습 1 — GPU 인식 및 기본 연산 확인

DGX Spark에서 가장 먼저 해야 할 일은 GPU가 올바르게 인식되는지 확인하는 것입니다.

```bash
# 터미널에서 GPU 상태 확인
nvidia-smi
# CUDA 버전 13.0 이상이 표시되어야 함
nvcc --version
```

```python
import torch

# 기본 텐서 연산 벤치마크
device = torch.device('cuda')

# 대규모 행렬 곱셈으로 GPU 성능 체감
a = torch.randn(4096, 4096, device=device, dtype=torch.bfloat16)
b = torch.randn(4096, 4096, device=device, dtype=torch.bfloat16)

# 워밍업
for _ in range(10):
    c = torch.matmul(a, b)
torch.cuda.synchronize()

# 벤치마크
import time
start = time.time()
for _ in range(100):
    c = torch.matmul(a, b)
torch.cuda.synchronize()
elapsed = time.time() - start

tflops = 2 * 4096**3 * 100 / elapsed / 1e12
print(f"BF16 MatMul: {tflops:.2f} TFLOPS ({elapsed:.3f}s for 100 iterations)")
```

### 6.4 실습 2 — MNIST 분류 모델 학습

고전적인 MNIST 손글씨 숫자 분류를 DGX Spark에서 학습합니다.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

# 데이터 준비
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])

train_dataset = datasets.MNIST('./data', train=True, download=True, transform=transform)
test_dataset = datasets.MNIST('./data', train=False, transform=transform)

# DGX Spark: num_workers=0 권장 (통합 메모리 아키텍처)
train_loader = DataLoader(train_dataset, batch_size=128, shuffle=True, num_workers=0)
test_loader = DataLoader(test_dataset, batch_size=256, shuffle=False, num_workers=0)

# CNN 모델 정의
class MNISTNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, padding=1)
        self.conv2 = nn.Conv2d(32, 64, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc1 = nn.Linear(64 * 7 * 7, 128)
        self.fc2 = nn.Linear(128, 10)
        self.dropout = nn.Dropout(0.25)
    
    def forward(self, x):
        x = self.pool(torch.relu(self.conv1(x)))
        x = self.pool(torch.relu(self.conv2(x)))
        x = x.view(-1, 64 * 7 * 7)
        x = self.dropout(torch.relu(self.fc1(x)))
        x = self.fc2(x)
        return x

device = torch.device('cuda')
model = MNISTNet().to(device)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=1e-3)

# 학습
for epoch in range(5):
    model.train()
    total_loss = 0
    for batch_data, batch_labels in train_loader:
        batch_data, batch_labels = batch_data.to(device), batch_labels.to(device)
        
        optimizer.zero_grad()
        outputs = model(batch_data)
        loss = criterion(outputs, batch_labels)
        loss.backward()
        optimizer.step()
        total_loss += loss.item()
    
    # 평가
    model.eval()
    correct = 0
    total = 0
    with torch.no_grad():
        for batch_data, batch_labels in test_loader:
            batch_data, batch_labels = batch_data.to(device), batch_labels.to(device)
            outputs = model(batch_data)
            _, predicted = outputs.max(1)
            total += batch_labels.size(0)
            correct += predicted.eq(batch_labels).sum().item()
    
    acc = 100. * correct / total
    avg_loss = total_loss / len(train_loader)
    print(f"Epoch {epoch+1}/5 | Loss: {avg_loss:.4f} | Test Acc: {acc:.2f}%")
```

### 6.5 실습 3 — LLM 추론 (Llama 모델)

DGX Spark의 128GB 통합 메모리를 활용하여 대형 언어 모델을 로컬에서 추론합니다.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model_name = "meta-llama/Llama-3.2-3B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

# 대화형 프롬프트
messages = [
    {"role": "system", "content": "당신은 도움이 되는 AI 어시스턴트입니다."},
    {"role": "user", "content": "PyTorch에서 GPU 메모리를 효율적으로 관리하는 방법을 알려주세요."}
]

inputs = tokenizer.apply_chat_template(
    messages, return_tensors="pt", add_generation_prompt=True
).to(model.device)

with torch.inference_mode():
    outputs = model.generate(
        inputs,
        max_new_tokens=512,
        temperature=0.7,
        do_sample=True
    )

response = tokenizer.decode(outputs[0][inputs.shape[-1]:], skip_special_tokens=True)
print(response)
```

더 큰 모델(70B)을 로드하려면 4비트 양자화를 활용합니다:

```python
from transformers import BitsAndBytesConfig

quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_use_double_quant=True,
)

model_70b = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70B-Instruct",
    quantization_config=quantization_config,
    device_map="auto"
)
```

### 6.6 실습 4 — LoRA 파인튜닝

DGX Spark에서 실제로 모델을 파인튜닝하는 완전한 워크플로입니다.

```python
from transformers import (
    AutoModelForCausalLM, AutoTokenizer,
    TrainingArguments, Trainer, DataCollatorForLanguageModeling
)
from peft import LoraConfig, get_peft_model, TaskType
from datasets import load_dataset
import torch

# 1. 모델 로드 (BF16)
model_name = "meta-llama/Llama-3.2-3B"
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

# 2. LoRA 설정
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,
    lora_alpha=32,
    lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()

# 3. 데이터셋 준비
dataset = load_dataset("tatsu-lab/alpaca", split="train[:5000]")

def format_and_tokenize(example):
    text = f"### Instruction:\n{example['instruction']}\n\n"
    if example.get('input'):
        text += f"### Input:\n{example['input']}\n\n"
    text += f"### Response:\n{example['output']}"
    return tokenizer(text, truncation=True, max_length=512, padding="max_length")

tokenized_dataset = dataset.map(format_and_tokenize, remove_columns=dataset.column_names)

# 4. 학습 설정
training_args = TrainingArguments(
    output_dir="./lora-llama-output",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,     # 실효 배치 크기 = 16
    learning_rate=2e-4,
    bf16=True,                         # BF16 Mixed Precision
    logging_steps=10,
    save_strategy="epoch",
    warmup_ratio=0.03,
    lr_scheduler_type="cosine",
    report_to="none",
)

# 5. Trainer로 학습 실행
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset,
    data_collator=DataCollatorForLanguageModeling(tokenizer, mlm=False),
)

trainer.train()

# 6. LoRA 어댑터 저장
model.save_pretrained("./lora-adapter")
tokenizer.save_pretrained("./lora-adapter")
```

### 6.7 실습 5 — DGX Dashboard와 JupyterLab 활용

DGX Spark에는 웹 기반 DGX Dashboard가 기본 탑재되어 있습니다.

```bash
# 로컬 접속 시
# 데스크톱 하단의 "Show Apps" → "DGX Dashboard" 클릭
# 또는 브라우저에서 http://localhost:11000

# 원격 접속 시 (SSH 터널링)
ssh -L 11000:localhost:11000 username@spark-xxxx.local
# 이후 브라우저에서 http://localhost:11000 접속
```

DGX Dashboard에서는 시스템 모니터링(GPU/CPU/메모리 사용률), 시스템 업데이트, JupyterLab 실행 등을 수행할 수 있습니다. JupyterLab을 시작하면 자동으로 가상 환경이 생성되고 권장 패키지가 설치됩니다.

### 6.8 실습 6 — 듀얼 Spark 분산 추론

두 대의 DGX Spark를 연결하면 256GB 통합 메모리에서 더 큰 모델을 실행할 수 있습니다.

```bash
# 200G QSFP56 DAC 케이블로 두 대의 Spark 연결
# 네트워크 인터페이스 설정

# Spark 1 (Primary)
sudo ip addr add 192.168.100.1/24 dev eth1
# Spark 2 (Secondary)  
sudo ip addr add 192.168.100.2/24 dev eth1

# 연결 확인
ping 192.168.100.2
```

이후 NCCL 기반의 분산 추론이나 vLLM, SGLang 같은 프레임워크를 사용하여 두 Spark에 걸쳐 모델을 분산 배포할 수 있습니다.

---

## 7장. MLOps/LLMOps 관점의 Best Practices

### 7.1 실험 추적과 재현성

PyTorch 프로젝트에서 실험의 재현성을 보장하려면 다음을 관리해야 합니다.

```python
import torch
import random
import numpy as np

def set_seed(seed=42):
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

set_seed(42)
```

Weights & Biases(wandb), MLflow, TensorBoard 등의 실험 추적 도구와 연동하는 것이 운영 환경에서의 표준 관행입니다.

### 7.2 모델 서빙과 배포 파이프라인

DGX Spark에서 개발한 모델을 프로덕션에 배포하는 전형적인 파이프라인은 다음과 같습니다.

첫째, DGX Spark에서 프로토타이핑 및 파인튜닝을 수행합니다. 둘째, 학습된 모델을 TorchScript, ONNX, 또는 TensorRT로 내보냅니다. 셋째, NVIDIA Triton Inference Server나 vLLM을 사용하여 서빙 환경을 구성합니다. 넷째, 클라우드(AWS, GCP, Azure) 또는 온프레미스 DGX 서버에 배포합니다.

```python
# TensorRT로 내보내기 (추론 최적화)
import torch_tensorrt

trt_model = torch_tensorrt.compile(
    model,
    inputs=[torch_tensorrt.Input(shape=[1, 3, 224, 224], dtype=torch.float16)],
    enabled_precisions={torch.float16}
)
torch.jit.save(trt_model, "model_trt.ts")
```

### 7.3 메모리 프로파일링과 최적화

대형 모델 작업 시 메모리 관리는 매우 중요합니다.

```python
# GPU 메모리 사용량 모니터링
print(f"Allocated: {torch.cuda.memory_allocated() / 1e9:.2f} GB")
print(f"Reserved:  {torch.cuda.memory_reserved() / 1e9:.2f} GB")
print(f"Max Allocated: {torch.cuda.max_memory_allocated() / 1e9:.2f} GB")

# 메모리 프로파일러 사용
from torch.profiler import profile, ProfilerActivity

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    profile_memory=True,
    record_shapes=True
) as prof:
    model(input_data)

print(prof.key_averages().table(sort_by="cuda_memory_usage", row_limit=10))
```

---

## 8장. 트러블슈팅 가이드 (DGX Spark)

### 8.1 자주 발생하는 문제와 해결책

**문제: PyTorch에서 CUDA가 인식되지 않음**

DGX Spark의 Blackwell GPU(sm_121)는 비교적 최신 아키텍처이므로, PyTorch 안정 버전에서 지원되지 않을 수 있습니다. NVIDIA 공식 컨테이너를 사용하거나, CUDA 13 호환 wheel을 설치해야 합니다.

```bash
# 해결: CUDA 13 호환 PyTorch 설치
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130
```

**문제: ARM64 패키지 호환성**

일부 Python 패키지가 ARM64(aarch64) 바이너리를 제공하지 않아 설치에 실패할 수 있습니다. Docker에서 Ubuntu 22.04 기반 이미지를 사용하면 호환성이 더 좋은 경우가 많습니다.

```dockerfile
# Dockerfile 예시
FROM nvidia/cuda:12.8.0-devel-ubuntu22.04
# Ubuntu 24.04 대신 22.04를 권장
```

**문제: Docker에서 nvidia 런타임 인식 실패**

```bash
# Error: unknown or invalid runtime name: nvidia
# 해결: NVIDIA Container Toolkit 설정 확인
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**문제: Out of Memory (OOM)**

통합 메모리 아키텍처에서는 CPU와 GPU가 128GB를 공유하므로, 모델 + 데이터 + 시스템이 함께 메모리를 사용합니다. `gradient_checkpointing`을 활성화하거나, 배치 크기를 줄이고, `gradient_accumulation_steps`를 늘려 실효 배치 크기를 유지할 수 있습니다.

```python
model.gradient_checkpointing_enable()
```

---

## 9장. 학습 로드맵

### 9.1 단계별 학습 경로

**1단계 — 기초 (1~2주)**: Python 기본 + NumPy 숙달 → Tensor 연산 익히기 → Autograd 이해 → 간단한 MLP로 MNIST 분류 구현

**2단계 — 중급 (2~4주)**: CNN, RNN, Transformer 구현 → 사전 학습 모델 활용(Transfer Learning) → 데이터 증강, 정규화 기법 → Mixed Precision Training

**3단계 — 고급 (4~8주)**: 분산 학습(DDP, FSDP) → LoRA/QLoRA 파인튜닝 → torch.compile 최적화 → 커스텀 CUDA 커널 작성

**4단계 — 운영 (지속)**: 모델 서빙(Triton, vLLM) → CI/CD 파이프라인 구축 → A/B 테스트 및 모델 모니터링 → 비용 최적화

### 9.2 추천 학습 리소스

공식 문서 및 튜토리얼로는 PyTorch 공식 사이트(pytorch.org/tutorials)와 NVIDIA DGX Spark Playbooks(github.com/NVIDIA/dgx-spark-playbooks)를 추천합니다. NVIDIA의 DGX Spark용 PyTorch 파인튜닝 가이드(build.nvidia.com/spark/pytorch-fine-tune)도 매우 유용합니다.

실습 환경으로는 DGX Spark 내장 JupyterLab, VS Code Remote SSH, 그리고 Claude Code를 활용한 YOLO 모드 개발이 있습니다.

---

## 부록. 주요 명령어 치트시트

```bash
# === DGX Spark 시스템 확인 ===
nvidia-smi                          # GPU 상태
nvcc --version                      # CUDA 버전
uname -m                            # 아키텍처 확인 (aarch64)
cat /etc/os-release                 # OS 정보

# === Docker 관련 ===
docker pull nvcr.io/nvidia/pytorch:25.11-py3
docker run --gpus all -it --rm --ipc=host -v ${PWD}:/workspace nvcr.io/nvidia/pytorch:25.11-py3

# === PyTorch 설치 ===
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130

# === 유용한 Python 스니펫 ===
python3 -c "import torch; print(torch.cuda.is_available())"
python3 -c "import torch; print(torch.cuda.get_device_name(0))"
python3 -c "import torch; print(torch.cuda.mem_get_info())"
```

---

> **이 문서는 PyTorch의 기본 개념부터 DGX Spark에서의 실전 활용까지를 포괄하는 종합 강의 자료입니다.**  
> **각 실습 코드는 DGX Spark 환경에서 직접 실행할 수 있도록 작성되었습니다.**
