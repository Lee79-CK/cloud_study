# Terraform 완전 정복: 초급 ~ 고급 강의 및 실전 문제집

> 대상: 초급자 ~ 고급 Cloud Engineer  
> 클라우드: AWS / Azure 기반  
> Terraform 버전: 1.5+ (HCL2)

---

## Part 1. 기초 개념 (Beginner)

---

### 1-1. Infrastructure as Code(IaC)란?

Infrastructure as Code는 인프라를 수동 콘솔 작업이 아닌 **선언적 코드**로 정의하고, 버전 관리·자동화·재현 가능하게 만드는 방법론이다.

**IaC의 핵심 이점:**

- **재현성(Reproducibility):** 동일 코드를 실행하면 동일 인프라가 생성된다.
- **버전 관리:** Git으로 인프라 변경 이력을 추적할 수 있다.
- **자동화:** CI/CD 파이프라인에 통합하여 인프라 배포를 자동화한다.
- **문서화:** 코드 자체가 인프라의 현재 상태를 문서화한다.
- **협업:** 팀원 간 코드 리뷰를 통해 인프라 변경을 검증한다.

**IaC 도구 비교:**

| 도구 | 방식 | 언어 | 상태 관리 | 멀티 클라우드 |
|------|------|------|-----------|---------------|
| Terraform | 선언적 | HCL | State 파일 | ✅ |
| Pulumi | 선언적/명령적 | Python, TS 등 | State 파일 | ✅ |
| CloudFormation | 선언적 | JSON/YAML | AWS 내장 | ❌ (AWS 전용) |
| ARM/Bicep | 선언적 | JSON/Bicep | Azure 내장 | ❌ (Azure 전용) |
| Ansible | 명령적 | YAML | 없음 | ✅ |

---

### 1-2. Terraform 아키텍처와 동작 원리

Terraform은 **선언적(Declarative)** 접근 방식을 사용한다. "원하는 최종 상태"를 정의하면, Terraform이 현재 상태와 비교하여 필요한 변경만 수행한다.

**핵심 구성 요소:**

```
┌──────────────────────────────────────────────────┐
│                Terraform Core                     │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  HCL    │  │  State   │  │  Dependency      │ │
│  │  Parser  │  │  Manager │  │  Graph Builder   │ │
│  └─────────┘  └──────────┘  └──────────────────┘ │
└──────────────────┬───────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
  ┌─────▼─────┐        ┌─────▼─────┐
  │  AWS      │        │  Azure    │
  │  Provider │        │  Provider │
  └─────┬─────┘        └─────┬─────┘
        │                     │
  ┌─────▼─────┐        ┌─────▼─────┐
  │  AWS API  │        │ Azure API │
  └───────────┘        └───────────┘
```

**Terraform 워크플로우:**

1. **Write:** `.tf` 파일에 인프라를 코드로 정의
2. **Init:** `terraform init` — Provider 플러그인 다운로드 및 백엔드 초기화
3. **Plan:** `terraform plan` — 현재 상태와 원하는 상태의 차이를 계산 (Dry Run)
4. **Apply:** `terraform apply` — 실제 인프라 변경 실행
5. **Destroy:** `terraform destroy` — 모든 리소스 삭제

---

### 1-3. HCL(HashiCorp Configuration Language) 기본 문법

#### Block 구조

```hcl
# 기본 Block 구조
<BLOCK_TYPE> "<BLOCK_LABEL>" "<BLOCK_NAME>" {
  # Arguments (Key-Value)
  <IDENTIFIER> = <EXPRESSION>

  # Nested Block
  <BLOCK_TYPE> {
    <IDENTIFIER> = <EXPRESSION>
  }
}
```

#### 데이터 타입

```hcl
# Primitive Types
string_example  = "hello"
number_example  = 42
bool_example    = true

# Collection Types
list_example    = ["a", "b", "c"]           # 순서 있는 배열
map_example     = { key1 = "val1", key2 = "val2" }  # 키-값 쌍
set_example     = toset(["a", "b", "c"])    # 중복 없는 집합

# Structural Types
object_example = {
  name = "server"
  port = 8080
}

# tuple (서로 다른 타입의 순서 있는 시퀀스)
tuple_example = ["hello", 42, true]
```

---

### 1-4. Provider 설정

Provider는 Terraform이 특정 클라우드 플랫폼의 API와 통신하기 위한 플러그인이다.

#### AWS Provider

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # 5.x 최신 버전 사용
    }
  }
}

# provider.tf
provider "aws" {
  region = "ap-northeast-2"   # 서울 리전

  default_tags {
    tags = {
      Environment = "dev"
      ManagedBy   = "terraform"
      Team        = "platform"
    }
  }
}

# 멀티 리전 사용 시 alias 활용
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}
```

#### Azure Provider

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    key_vault {
      purge_soft_delete_on_destroy = false
    }
  }

  # Service Principal 인증 (CI/CD 환경)
  # client_id       = var.arm_client_id
  # client_secret   = var.arm_client_secret
  # tenant_id       = var.arm_tenant_id
  # subscription_id = var.arm_subscription_id
}
```

---

### 1-5. 첫 번째 리소스 생성

#### AWS: VPC + EC2 기본 구성

```hcl
# main.tf

# VPC 생성
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "main-vpc"
  }
}

# 퍼블릭 서브넷
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-2a"
  }
}

# 인터넷 게이트웨이
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# 라우트 테이블
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# 보안 그룹
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # 실서비스에서는 특정 IP 제한 필수
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}

# EC2 인스턴스
resource "aws_instance" "web" {
  ami                    = "ami-0c9c942bd7bf113a2"  # Amazon Linux 2023
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
  EOF

  tags = {
    Name = "web-server"
  }
}

# 출력값
output "instance_public_ip" {
  description = "웹 서버 퍼블릭 IP"
  value       = aws_instance.web.public_ip
}
```

#### Azure: Resource Group + Virtual Network + VM 기본 구성

```hcl
# main.tf

# 리소스 그룹
resource "azurerm_resource_group" "main" {
  name     = "rg-terraform-demo"
  location = "koreacentral"

  tags = {
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

# 가상 네트워크
resource "azurerm_virtual_network" "main" {
  name                = "vnet-main"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

# 서브넷
resource "azurerm_subnet" "web" {
  name                 = "snet-web"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

# 퍼블릭 IP
resource "azurerm_public_ip" "web" {
  name                = "pip-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

# 네트워크 보안 그룹
resource "azurerm_network_security_group" "web" {
  name                = "nsg-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowHTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowSSH"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# NIC
resource "azurerm_network_interface" "web" {
  name                = "nic-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.web.id
  }
}

# NSG-NIC 연결
resource "azurerm_network_interface_security_group_association" "web" {
  network_interface_id      = azurerm_network_interface.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}

# Virtual Machine
resource "azurerm_linux_virtual_machine" "web" {
  name                = "vm-web"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "azureuser"

  network_interface_ids = [azurerm_network_interface.web.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  custom_data = base64encode(<<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    echo "<h1>Hello from Terraform on Azure!</h1>" > /var/www/html/index.html
    systemctl restart nginx
  EOF
  )
}

output "vm_public_ip" {
  value = azurerm_public_ip.web.ip_address
}
```

---

### 1-6. Variables, Outputs, Locals

#### Variables (입력 변수)

```hcl
# variables.tf

variable "environment" {
  description = "배포 환경 (dev, staging, prod)"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment는 dev, staging, prod 중 하나여야 합니다."
  }
}

variable "instance_type_map" {
  description = "환경별 인스턴스 타입 매핑"
  type        = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

variable "allowed_ports" {
  description = "허용할 포트 목록"
  type        = list(number)
  default     = [80, 443, 8080]
}

variable "db_password" {
  description = "데이터베이스 비밀번호"
  type        = string
  sensitive   = true   # plan/apply 출력 시 마스킹
}

variable "vpc_config" {
  description = "VPC 설정 객체"
  type = object({
    cidr_block         = string
    enable_dns         = bool
    public_subnets     = list(string)
    private_subnets    = list(string)
    availability_zones = list(string)
  })
  default = {
    cidr_block         = "10.0.0.0/16"
    enable_dns         = true
    public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
    private_subnets    = ["10.0.101.0/24", "10.0.102.0/24"]
    availability_zones = ["ap-northeast-2a", "ap-northeast-2c"]
  }
}
```

**변수 값 전달 방법 (우선순위 낮→높):**

1. `default` 값
2. 환경 변수: `TF_VAR_environment=prod`
3. `terraform.tfvars` 또는 `*.auto.tfvars` 파일
4. `-var-file="prod.tfvars"` CLI 플래그
5. `-var="environment=prod"` CLI 플래그

```hcl
# prod.tfvars
environment = "prod"
db_password = "SuperSecretP@ss123"

vpc_config = {
  cidr_block         = "10.0.0.0/16"
  enable_dns         = true
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets    = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  availability_zones = ["ap-northeast-2a", "ap-northeast-2b", "ap-northeast-2c"]
}
```

#### Outputs (출력값)

```hcl
# outputs.tf

output "vpc_id" {
  description = "생성된 VPC의 ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  value       = aws_subnet.public[*].id
}

output "db_connection_string" {
  description = "DB 접속 문자열"
  value       = "postgresql://${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  sensitive   = true
}

# 다른 모듈에서 참조: module.vpc.vpc_id
```

#### Locals (지역 변수)

```hcl
# locals.tf

locals {
  # 공통 태그 생성
  common_tags = {
    Environment = var.environment
    Project     = "my-platform"
    ManagedBy   = "terraform"
    CreatedAt   = timestamp()
  }

  # 네이밍 규칙 통일
  name_prefix = "${var.project}-${var.environment}"

  # 조건부 값 설정
  is_production   = var.environment == "prod"
  instance_type   = local.is_production ? "t3.large" : "t3.micro"
  min_capacity    = local.is_production ? 3 : 1
  max_capacity    = local.is_production ? 10 : 3
  multi_az        = local.is_production ? true : false

  # 서브넷-AZ 매핑 (zipmap 활용)
  subnet_az_map = zipmap(
    var.vpc_config.availability_zones,
    var.vpc_config.public_subnets
  )
}
```

---

### 1-7. Data Sources

Data Source는 이미 존재하는 리소스나 외부 데이터를 **읽기 전용**으로 참조한다.

```hcl
# AWS: 최신 Amazon Linux AMI 조회
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# AWS: 현재 사용 중인 계정 정보
data "aws_caller_identity" "current" {}

# AWS: 현재 리전 정보
data "aws_region" "current" {}

# AWS: 기존 VPC 참조
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Azure: 기존 리소스 그룹 참조
data "azurerm_resource_group" "existing" {
  name = "rg-existing-infra"
}

# Azure: 기존 Key Vault의 시크릿 참조
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-admin-password"
  key_vault_id = data.azurerm_key_vault.main.id
}

# 활용 예시
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id   # 항상 최신 AMI
  instance_type = "t3.micro"
  subnet_id     = data.aws_vpc.existing.id

  tags = {
    Account = data.aws_caller_identity.current.account_id
    Region  = data.aws_region.current.name
  }
}
```

---

### 1-8. State 관리 기초

Terraform State는 코드에 정의된 리소스와 실제 클라우드 리소스 간의 **매핑 정보**를 저장한다.

**State가 하는 일:**
- 리소스 ID와 속성값 저장
- 리소스 간 의존성 추적
- 성능 최적화 (API 호출 최소화)
- `plan` 시 변경 사항 계산의 기준점

**로컬 State vs 원격 State:**

```hcl
# 원격 State Backend — AWS S3
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "envs/dev/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-lock"   # State Locking
  }
}

# 원격 State Backend — Azure Blob Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "envs/dev/terraform.tfstate"
  }
}
```

**State Locking이 중요한 이유:** 두 사람이 동시에 `terraform apply`를 실행하면 State 파일이 충돌하여 인프라가 꼬일 수 있다. DynamoDB(AWS) 또는 Blob Lease(Azure)를 통해 동시 접근을 방지한다.

---

## Part 2. 중급 개념 (Intermediate)

---

### 2-1. 반복문: count, for_each, for

#### count

```hcl
# 동일한 리소스를 여러 개 생성
resource "aws_subnet" "public" {
  count = length(var.vpc_config.public_subnets)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.vpc_config.public_subnets[count.index]
  availability_zone = var.vpc_config.availability_zones[count.index]

  tags = {
    Name = "${local.name_prefix}-public-${count.index + 1}"
  }
}

# 조건부 리소스 생성 (0 또는 1개)
resource "aws_nat_gateway" "main" {
  count = local.is_production ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}
```

#### for_each (권장)

```hcl
# Map 기반 for_each — 리소스마다 고유 키 부여
variable "subnets" {
  default = {
    "public-2a" = {
      cidr = "10.0.1.0/24"
      az   = "ap-northeast-2a"
      type = "public"
    }
    "public-2c" = {
      cidr = "10.0.2.0/24"
      az   = "ap-northeast-2c"
      type = "public"
    }
    "private-2a" = {
      cidr = "10.0.101.0/24"
      az   = "ap-northeast-2a"
      type = "private"
    }
  }
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.type == "public"

  tags = {
    Name = "${local.name_prefix}-${each.key}"
    Type = each.value.type
  }
}

# 특정 서브넷 참조: aws_subnet.this["public-2a"].id
```

> **count vs for_each:** `count`는 인덱스 기반이라 중간 요소를 삭제하면 뒤의 리소스가 모두 재생성된다. `for_each`는 키 기반이므로 특정 항목만 독립적으로 추가/삭제할 수 있다. **가능하면 항상 `for_each`를 사용하라.**

#### for 표현식

```hcl
# 리스트 변환
locals {
  upper_names = [for name in var.names : upper(name)]

  # 필터링
  prod_instances = [
    for k, v in var.instances : v
    if v.environment == "prod"
  ]

  # Map 생성
  subnet_id_map = {
    for k, subnet in aws_subnet.this : k => subnet.id
  }

  # 중첩 for (Nested Loop)
  sg_rules = flatten([
    for port in var.allowed_ports : [
      for cidr in var.allowed_cidrs : {
        port = port
        cidr = cidr
      }
    ]
  ])
}
```

---

### 2-2. 조건문과 동적 블록

#### 조건 표현식

```hcl
# 삼항 연산자
resource "aws_instance" "app" {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
  monitoring    = var.environment == "prod" ? true : false
}

# 조건부 리소스 생성
resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name          = "${local.name_prefix}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
}
```

#### Dynamic Block

```hcl
# 보안 그룹 규칙을 동적으로 생성
variable "ingress_rules" {
  default = [
    { port = 80,  cidr = "0.0.0.0/0",    description = "HTTP" },
    { port = 443, cidr = "0.0.0.0/0",    description = "HTTPS" },
    { port = 22,  cidr = "10.0.0.0/8",   description = "SSH from VPN" },
    { port = 8080, cidr = "10.0.0.0/8",  description = "App from VPN" },
  ]
}

resource "aws_security_group" "app" {
  name   = "${local.name_prefix}-app-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Azure NSG에서의 dynamic block
resource "azurerm_network_security_group" "app" {
  name                = "nsg-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  dynamic "security_rule" {
    for_each = { for idx, rule in var.ingress_rules : idx => rule }
    content {
      name                       = security_rule.value.description
      priority                   = 100 + security_rule.key
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = tostring(security_rule.value.port)
      source_address_prefix      = security_rule.value.cidr
      destination_address_prefix = "*"
    }
  }
}
```

---

### 2-3. 모듈 (Modules)

모듈은 재사용 가능한 Terraform 코드의 패키지이다.

#### 디렉토리 구조

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── ec2/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── rds/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

#### VPC 모듈 작성

```hcl
# modules/vpc/variables.tf
variable "name_prefix" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "public_subnets" {
  type = list(string)
}

variable "private_subnets" {
  type = list(string)
}

variable "azs" {
  type = list(string)
}

variable "enable_nat_gateway" {
  type    = bool
  default = false
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/vpc/main.tf

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-vpc"
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name_prefix}-igw" })
}

# 퍼블릭 서브넷
resource "aws_subnet" "public" {
  for_each = { for idx, cidr in var.public_subnets : var.azs[idx] => cidr }

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value
  availability_zone       = each.key
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-public-${each.key}"
    Tier = "public"
  })
}

# 프라이빗 서브넷
resource "aws_subnet" "private" {
  for_each = { for idx, cidr in var.private_subnets : var.azs[idx] => cidr }

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value
  availability_zone = each.key

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-private-${each.key}"
    Tier = "private"
  })
}

# NAT Gateway (프로덕션 전용)
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"
  tags   = merge(var.tags, { Name = "${var.name_prefix}-nat-eip" })
}

resource "aws_nat_gateway" "this" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = values(aws_subnet.public)[0].id

  tags = merge(var.tags, { Name = "${var.name_prefix}-nat" })
}

# 퍼블릭 라우트 테이블
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(var.tags, { Name = "${var.name_prefix}-public-rt" })
}

resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

# 프라이빗 라우트 테이블
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.this[0].id
    }
  }

  tags = merge(var.tags, { Name = "${var.name_prefix}-private-rt" })
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}

output "public_subnet_ids" {
  value = [for s in aws_subnet.public : s.id]
}

output "private_subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}

output "vpc_cidr_block" {
  value = aws_vpc.this.cidr_block
}
```

#### 모듈 호출

```hcl
# main.tf (루트 모듈)

module "vpc" {
  source = "./modules/vpc"

  name_prefix        = local.name_prefix
  vpc_cidr           = "10.0.0.0/16"
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets    = ["10.0.101.0/24", "10.0.102.0/24"]
  azs                = ["ap-northeast-2a", "ap-northeast-2c"]
  enable_nat_gateway = local.is_production
  tags               = local.common_tags
}

# 모듈 출력값 참조
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnet_ids[0]
  # ...
}
```

#### Terraform Registry 모듈 사용

```hcl
# 공식 AWS VPC 모듈 사용
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${local.name_prefix}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-2a", "ap-northeast-2c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = !local.is_production
  enable_dns_hostnames = true

  tags = local.common_tags
}
```

---

### 2-4. Provisioners와 대안

Provisioner는 리소스 생성 후 추가 설정을 실행한다. **가능하면 사용하지 않는 것이 좋으며**, 대안을 먼저 고려해야 한다.

```hcl
# Provisioner 사용 예시 (비권장)
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # 파일 전송
  provisioner "file" {
    source      = "scripts/setup.sh"
    destination = "/tmp/setup.sh"

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # 원격 실행
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "sudo /tmp/setup.sh"
    ]

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # 로컬 실행 (리소스 생성 후)
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }

  # 리소스 삭제 시 실행
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance destroyed' >> destroy.log"
  }
}
```

**권장 대안:**
- **user_data / custom_data** — EC2/VM 초기 설정 스크립트
- **Packer** — 사전 구성된 AMI/이미지 빌드
- **Ansible/Chef/Puppet** — 설정 관리 도구와 연동
- **cloud-init** — 클라우드 환경의 표준 초기화 도구

---

### 2-5. 내장 함수 (Built-in Functions)

```hcl
locals {
  # 문자열 함수
  lower_name   = lower("HELLO")           # "hello"
  upper_name   = upper("hello")           # "HELLO"
  trimmed      = trimspace("  hello  ")   # "hello"
  formatted    = format("Hello, %s!", "World")
  joined       = join(", ", ["a", "b", "c"])  # "a, b, c"
  split_result = split(",", "a,b,c")          # ["a", "b", "c"]
  replaced     = replace("hello-world", "-", "_")

  # 숫자 함수
  max_val = max(1, 5, 3)       # 5
  min_val = min(1, 5, 3)       # 1
  ceiling = ceil(4.3)          # 5
  floored = floor(4.7)         # 4

  # 컬렉션 함수
  merged    = merge({ a = 1 }, { b = 2 })          # { a=1, b=2 }
  keys_list = keys({ a = 1, b = 2 })               # ["a", "b"]
  vals_list = values({ a = 1, b = 2 })              # [1, 2]
  looked_up = lookup({ a = 1, b = 2 }, "a", 0)     # 1
  flat      = flatten([["a", "b"], ["c"]])          # ["a", "b", "c"]
  distinct_list = distinct(["a", "a", "b"])         # ["a", "b"]
  zipped    = zipmap(["a", "b"], [1, 2])            # { a=1, b=2 }
  contains_check = contains(["a", "b", "c"], "b")  # true
  len       = length(["a", "b", "c"])               # 3

  # 파일 시스템 함수
  file_content  = file("${path.module}/scripts/init.sh")
  template      = templatefile("${path.module}/templates/config.tpl", {
    db_host = aws_db_instance.main.endpoint
    db_port = 5432
  })
  file_exists   = fileexists("${path.module}/optional.conf")

  # 인코딩 함수
  base64_encoded = base64encode("hello")
  json_encoded   = jsonencode({ key = "value" })
  json_decoded   = jsondecode("{\"key\": \"value\"}")
  yaml_encoded   = yamlencode({ key = "value" })

  # 타입 변환
  to_number = tonumber("42")
  to_string = tostring(42)
  to_list   = tolist(toset(["b", "a", "c"]))
  to_map    = tomap({ a = "1", b = "2" })

  # IP 함수
  cidr_subnet = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
  cidr_host   = cidrhost("10.0.1.0/24", 5)        # "10.0.1.5"

  # 날짜/해시
  current_time = timestamp()
  sha256_hash  = sha256("hello")

  # try — 에러 발생 시 대체값 반환
  safe_lookup = try(var.config.optional_key, "default_value")
}
```

---

### 2-6. 라이프사이클 관리

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  lifecycle {
    # 새 리소스 먼저 생성 후 기존 리소스 삭제 (다운타임 최소화)
    create_before_destroy = true

    # 특정 속성 변경 무시 (외부에서 변경된 태그 등)
    ignore_changes = [
      tags["LastModified"],
      ami,     # AMI 변경을 무시 (별도 롤링 업데이트 사용 시)
    ]

    # 리소스 삭제 방지 (프로덕션 DB 등)
    prevent_destroy = true

    # 사전 조건 검증 (Terraform 1.2+)
    precondition {
      condition     = var.instance_type != "t3.nano"
      error_message = "t3.nano는 이 워크로드에 너무 작습니다."
    }

    # 사후 조건 검증
    postcondition {
      condition     = self.public_ip != ""
      error_message = "인스턴스에 퍼블릭 IP가 할당되지 않았습니다."
    }

    # 리소스 교체 트리거 (Terraform 1.2+)
    replace_triggered_by = [
      aws_security_group.app.id   # SG 변경 시 인스턴스 교체
    ]
  }
}
```

---

### 2-7. State 관리 고급

```bash
# State 내용 확인
terraform state list
terraform state show aws_instance.web

# 리소스 이동 (리팩토링 시)
terraform state mv aws_instance.web aws_instance.app
terraform state mv aws_instance.old module.compute.aws_instance.main

# State에서 리소스 제거 (Terraform 관리 해제, 실제 리소스는 유지)
terraform state rm aws_instance.legacy

# 기존 리소스를 Terraform 관리로 가져오기
terraform import aws_instance.web i-0abc123def456

# State 강제 잠금 해제 (비상 시에만 사용)
terraform force-unlock <LOCK_ID>

# 리소스 오염 표시 (다음 apply 시 재생성)
terraform taint aws_instance.web        # deprecated
terraform apply -replace=aws_instance.web  # 권장 방법
```

#### moved 블록 (Terraform 1.1+)

```hcl
# 리소스 이름 변경 시 State 자동 이전
moved {
  from = aws_instance.web
  to   = aws_instance.app_server
}

# 모듈로 이동 시
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.main
}

# for_each로 전환 시
moved {
  from = aws_subnet.public[0]
  to   = aws_subnet.public["ap-northeast-2a"]
}
```

---

## Part 3. 고급 개념 (Advanced)

---

### 3-1. Workspace를 활용한 멀티 환경 관리

```bash
# Workspace 생성 및 전환
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace select dev
terraform workspace list
```

```hcl
# 워크스페이스별 설정 분기
locals {
  workspace_config = {
    dev = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
      multi_az      = false
    }
    staging = {
      instance_type = "t3.small"
      min_size      = 2
      max_size      = 4
      multi_az      = true
    }
    prod = {
      instance_type = "t3.medium"
      min_size      = 3
      max_size      = 10
      multi_az      = true
    }
  }

  env    = terraform.workspace
  config = local.workspace_config[local.env]
}

resource "aws_instance" "app" {
  instance_type = local.config.instance_type
  tags = {
    Environment = local.env
  }
}
```

> **주의:** Workspace는 간단한 환경 분리에 적합하지만, 프로덕션 환경에서는 **디렉토리 기반 분리** 또는 **Terragrunt**가 더 안전하다. Workspace는 모든 환경이 같은 코드베이스를 공유하므로, 환경별로 다른 리소스 구성이 필요한 경우에는 적합하지 않다.

---

### 3-2. Remote State 참조

다른 Terraform 프로젝트의 State에서 출력값을 읽어온다.

```hcl
# 네트워크 팀의 VPC 출력값 참조
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "company-terraform-state"
    key    = "network/vpc/terraform.tfstate"
    region = "ap-northeast-2"
  }
}

# 사용
resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [
    data.terraform_remote_state.network.outputs.app_security_group_id
  ]
}
```

---

### 3-3. 고급 모듈 패턴

#### 조건부 서브모듈

```hcl
# 모니터링을 선택적으로 활성화
module "monitoring" {
  source = "./modules/monitoring"
  count  = var.enable_monitoring ? 1 : 0

  cluster_name = module.eks.cluster_name
  alert_email  = var.alert_email
}
```

#### 모듈 간 의존성

```hcl
module "vpc" {
  source = "./modules/vpc"
  # ...
}

module "rds" {
  source = "./modules/rds"

  # 명시적 의존성 — vpc 모듈 완료 후 실행
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  depends_on = [module.vpc]  # 암시적 의존성이 부족할 때만 사용
}

module "app" {
  source = "./modules/app"

  vpc_id              = module.vpc.vpc_id
  subnet_ids          = module.vpc.private_subnet_ids
  db_connection_string = module.rds.connection_string
}
```

---

### 3-4. AWS 실전 아키텍처: ECS Fargate + ALB + RDS

```hcl
# --- VPC ---
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${local.name_prefix}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-2a", "ap-northeast-2c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = !local.is_production
  enable_dns_hostnames = true

  tags = local.common_tags
}

# --- ALB ---
resource "aws_lb" "app" {
  name               = "${local.name_prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets

  enable_deletion_protection = local.is_production

  tags = local.common_tags
}

resource "aws_lb_target_group" "app" {
  name        = "${local.name_prefix}-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 5
    matcher             = "200"
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# --- ECS Cluster ---
resource "aws_ecs_cluster" "main" {
  name = "${local.name_prefix}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = local.common_tags
}

# --- ECS Task Definition ---
resource "aws_ecs_task_definition" "app" {
  family                   = "${local.name_prefix}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = local.is_production ? "1024" : "256"
  memory                   = local.is_production ? "2048" : "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "app"
      image     = "${var.ecr_repository_url}:${var.image_tag}"
      essential = true

      portMappings = [{
        containerPort = 8080
        protocol      = "tcp"
      }]

      environment = [
        { name = "APP_ENV", value = var.environment },
        { name = "DB_HOST", value = aws_db_instance.main.endpoint },
        { name = "DB_NAME", value = aws_db_instance.main.db_name },
      ]

      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = aws_secretsmanager_secret.db_password.arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "app"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])

  tags = local.common_tags
}

# --- ECS Service ---
resource "aws_ecs_service" "app" {
  name            = "${local.name_prefix}-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = local.config.min_size
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.ecs.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_maximum_percent         = 200
  deployment_minimum_healthy_percent = 100

  tags = local.common_tags
}

# --- Auto Scaling ---
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = local.config.max_size
  min_capacity       = local.config.min_size
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${local.name_prefix}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# --- RDS ---
resource "aws_db_subnet_group" "main" {
  name       = "${local.name_prefix}-db-subnet"
  subnet_ids = module.vpc.private_subnets
  tags       = local.common_tags
}

resource "aws_db_instance" "main" {
  identifier = "${local.name_prefix}-db"

  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = local.is_production ? "db.r6g.large" : "db.t3.micro"
  allocated_storage    = 20
  max_allocated_storage = local.is_production ? 100 : 50

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az               = local.is_production
  backup_retention_period = local.is_production ? 30 : 7
  deletion_protection    = local.is_production

  skip_final_snapshot       = !local.is_production
  final_snapshot_identifier = local.is_production ? "${local.name_prefix}-db-final" : null

  performance_insights_enabled = local.is_production
  monitoring_interval         = local.is_production ? 60 : 0

  tags = local.common_tags

  lifecycle {
    prevent_destroy = false  # prod에서는 true 설정
  }
}
```

---

### 3-5. Azure 실전 아키텍처: AKS + Azure SQL

```hcl
# --- Resource Group ---
resource "azurerm_resource_group" "main" {
  name     = "rg-${local.name_prefix}"
  location = var.location
  tags     = local.common_tags
}

# --- Virtual Network ---
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${local.name_prefix}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
  tags                = local.common_tags
}

resource "azurerm_subnet" "aks" {
  name                 = "snet-aks"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.0.0/20"]   # /20 = 4096 IPs for pods
}

resource "azurerm_subnet" "db" {
  name                 = "snet-db"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.16.0/24"]

  delegation {
    name = "fs"
    service_delegation {
      name = "Microsoft.DBforPostgreSQL/flexibleServers"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action",
      ]
    }
  }
}

# --- AKS Cluster ---
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-${local.name_prefix}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = local.name_prefix
  kubernetes_version  = var.kubernetes_version

  default_node_pool {
    name                = "system"
    node_count          = local.is_production ? 3 : 1
    vm_size             = local.is_production ? "Standard_D4s_v3" : "Standard_B2s"
    vnet_subnet_id      = azurerm_subnet.aks.id
    os_disk_size_gb     = 50
    max_pods            = 50
    zones               = local.is_production ? ["1", "2", "3"] : null
    enable_auto_scaling = true
    min_count           = local.is_production ? 3 : 1
    max_count           = local.is_production ? 10 : 3

    node_labels = {
      "role" = "system"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
    service_cidr      = "172.16.0.0/16"
    dns_service_ip    = "172.16.0.10"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "5m"
  }

  tags = local.common_tags
}

# 워커 노드풀 추가
resource "azurerm_kubernetes_cluster_node_pool" "worker" {
  name                  = "worker"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = local.is_production ? "Standard_D8s_v3" : "Standard_D2s_v3"
  vnet_subnet_id        = azurerm_subnet.aks.id
  enable_auto_scaling   = true
  min_count             = local.is_production ? 2 : 1
  max_count             = local.is_production ? 20 : 5
  max_pods              = 50
  zones                 = local.is_production ? ["1", "2", "3"] : null

  node_labels = {
    "role" = "worker"
  }

  node_taints = []

  tags = local.common_tags
}

# --- Azure Database for PostgreSQL Flexible Server ---
resource "azurerm_private_dns_zone" "postgres" {
  name                = "${local.name_prefix}.postgres.database.azure.com"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "postgres" {
  name                  = "vnet-link-postgres"
  private_dns_zone_name = azurerm_private_dns_zone.postgres.name
  virtual_network_id    = azurerm_virtual_network.main.id
  resource_group_name   = azurerm_resource_group.main.name
}

resource "azurerm_postgresql_flexible_server" "main" {
  name                   = "psql-${local.name_prefix}"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  version                = "15"
  delegated_subnet_id    = azurerm_subnet.db.id
  private_dns_zone_id    = azurerm_private_dns_zone.postgres.id

  administrator_login    = "dbadmin"
  administrator_password = var.db_password

  storage_mb = local.is_production ? 65536 : 32768
  sku_name   = local.is_production ? "GP_Standard_D4s_v3" : "B_Standard_B1ms"

  zone = local.is_production ? "1" : null

  high_availability {
    mode                      = local.is_production ? "ZoneRedundant" : "Disabled"
    standby_availability_zone = local.is_production ? "2" : null
  }

  backup_retention_days = local.is_production ? 35 : 7

  tags = local.common_tags

  depends_on = [azurerm_private_dns_zone_virtual_network_link.postgres]
}

# --- Log Analytics ---
resource "azurerm_log_analytics_workspace" "main" {
  name                = "log-${local.name_prefix}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = local.is_production ? 90 : 30
  tags                = local.common_tags
}
```

---

### 3-6. Terraform Cloud / Enterprise 연동

```hcl
# HCP Terraform (구 Terraform Cloud) 백엔드 설정
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      tags = ["app:web", "env:prod"]
    }
  }
}
```

**주요 기능:**
- **Remote State 관리** — 안전한 상태 저장소 및 락킹
- **Remote Plan & Apply** — 팀 공유 실행 환경
- **Policy as Code (Sentinel)** — 정책 자동 강제
- **Private Registry** — 내부 모듈 공유
- **VCS 연동** — PR 시 자동 `plan`, merge 시 자동 `apply`
- **Cost Estimation** — 변경 사항의 예상 비용 표시

---

### 3-7. 테스트 전략

#### terraform validate & fmt

```bash
# 구문 유효성 검사
terraform validate

# 코드 포맷팅 (표준 스타일 적용)
terraform fmt -recursive

# 포맷 검사만 (CI/CD 용)
terraform fmt -check -recursive
```

#### tflint

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_unused_declarations" {
  enabled = true
}
```

#### checkov / tfsec (보안 스캔)

```bash
# checkov
checkov -d . --framework terraform

# tfsec (현재 trivy에 통합됨)
trivy config .
```

#### terraform test (Terraform 1.6+)

```hcl
# tests/vpc_test.tftest.hcl

variables {
  environment = "test"
  vpc_cidr    = "10.99.0.0/16"
}

run "vpc_creation" {
  command = plan  # 또는 apply

  assert {
    condition     = aws_vpc.main.cidr_block == "10.99.0.0/16"
    error_message = "VPC CIDR 블록이 올바르지 않습니다."
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS 호스트이름이 활성화되어야 합니다."
  }
}

run "subnet_count" {
  command = plan

  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "퍼블릭 서브넷이 2개여야 합니다."
  }
}
```

---

### 3-8. CI/CD 파이프라인 통합

#### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
    paths: ['infra/**']
  push:
    branches: [main]
    paths: ['infra/**']

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: tflint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          tflint --init
          tflint

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const plan = `${{ steps.plan.outputs.stdout }}`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan\n\`\`\`\n${plan.substring(0, 65000)}\n\`\`\``
            });

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

---

### 3-9. import 블록과 코드 생성 (Terraform 1.5+)

기존 리소스를 Terraform으로 가져오는 선언적 방법.

```hcl
# imports.tf

# 기존 리소스를 import 블록으로 선언
import {
  to = aws_s3_bucket.legacy
  id = "my-legacy-bucket-name"
}

import {
  to = aws_instance.legacy_web
  id = "i-0abc123def456"
}

import {
  to = azurerm_resource_group.legacy
  id = "/subscriptions/xxx/resourceGroups/rg-legacy"
}
```

```bash
# 1. import 블록 작성 후, 코드 자동 생성
terraform plan -generate-config-out=generated.tf

# 2. generated.tf를 검토하고 정리
# 3. terraform plan으로 no changes 확인
# 4. import 블록은 성공 후 제거해도 됨
```

---

### 3-10. 고급 패턴: Terragrunt

Terragrunt는 DRY(Don't Repeat Yourself) 원칙으로 Terraform 코드를 관리하는 래퍼 도구다.

```
# Terragrunt 디렉토리 구조
live/
├── terragrunt.hcl              # 루트: 공통 설정
├── dev/
│   ├── env.hcl                 # 환경별 변수
│   ├── vpc/
│   │   └── terragrunt.hcl
│   ├── ecs/
│   │   └── terragrunt.hcl
│   └── rds/
│       └── terragrunt.hcl
├── staging/
│   ├── env.hcl
│   ├── vpc/
│   │   └── terragrunt.hcl
│   └── ...
└── prod/
    ├── env.hcl
    └── ...
```

```hcl
# live/terragrunt.hcl (루트)
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "company-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "ap-northeast-2"
  default_tags {
    tags = {
      ManagedBy = "terragrunt"
    }
  }
}
EOF
}
```

```hcl
# live/dev/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

terraform {
  source = "../../../modules/vpc"
}

inputs = {
  environment    = local.env_vars.locals.environment
  vpc_cidr       = "10.0.0.0/16"
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
}
```

```hcl
# live/dev/ecs/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id             = "vpc-mock"
    private_subnet_ids = ["subnet-mock-1", "subnet-mock-2"]
  }
}

terraform {
  source = "../../../modules/ecs"
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
}
```

---

## Part 4. 베스트 프랙티스와 보안

---

### 4-1. 프로젝트 구조 권장

```
infrastructure/
├── modules/                    # 재사용 가능한 모듈
│   ├── networking/
│   ├── compute/
│   ├── database/
│   ├── monitoring/
│   └── security/
├── environments/               # 환경별 구성
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
├── global/                     # 환경 공통 리소스
│   ├── iam/
│   ├── dns/
│   └── s3-state/
├── .github/
│   └── workflows/
│       └── terraform.yml
├── .tflint.hcl
├── .pre-commit-config.yaml
└── README.md
```

### 4-2. 보안 베스트 프랙티스

```hcl
# 1. 민감 정보는 변수의 sensitive 플래그 사용
variable "db_password" {
  type      = string
  sensitive = true
}

# 2. AWS Secrets Manager / Azure Key Vault에서 시크릿 읽기
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}

# 3. KMS 암호화 키 관리
resource "aws_kms_key" "main" {
  description             = "Main encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = data.aws_iam_policy_document.kms.json
}

# 4. S3 State 암호화
terraform {
  backend "s3" {
    bucket         = "tf-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true                    # AES-256 암호화
    kms_key_id     = "arn:aws:kms:..."       # CMK 사용
    dynamodb_table = "tf-lock"
  }
}

# 5. 최소 권한 원칙 IAM 정책
data "aws_iam_policy_document" "ecs_task" {
  statement {
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
    ]
    resources = [
      "${aws_s3_bucket.app.arn}/*"
    ]
  }

  statement {
    effect = "Allow"
    actions = [
      "secretsmanager:GetSecretValue",
    ]
    resources = [
      aws_secretsmanager_secret.db_password.arn
    ]
  }
}

# 6. .gitignore 필수 항목
# .gitignore:
# *.tfstate
# *.tfstate.*
# .terraform/
# *.tfvars        (시크릿이 포함될 수 있음)
# crash.log
# override.tf
# override.tf.json
# *_override.tf
```

### 4-3. 네이밍 컨벤션

```hcl
# 리소스 네이밍 — 일관된 패턴 사용
locals {
  name_prefix = "${var.project}-${var.environment}"
}

# AWS 리소스 네이밍 예시
# VPC:      myapp-prod-vpc
# Subnet:   myapp-prod-public-2a
# SG:       myapp-prod-web-sg
# EC2:      myapp-prod-web-01
# RDS:      myapp-prod-db
# S3:       myapp-prod-assets-{account_id}

# Azure 리소스 네이밍 예시 (Azure CAF 권장)
# RG:       rg-myapp-prod
# VNet:     vnet-myapp-prod
# Subnet:   snet-web-myapp-prod
# VM:       vm-web-myapp-prod-01
# AKS:      aks-myapp-prod
# SQL:      psql-myapp-prod
```

---

## Part 5. 실전 문제집

---

### 섹션 A: 초급 문제 (10문제)

---

**문제 A-1.**  
다음 중 Terraform의 핵심 워크플로우 순서가 올바른 것은?

(a) plan → init → apply → destroy  
(b) init → apply → plan → destroy  
(c) init → plan → apply → destroy  
(d) apply → init → plan → destroy  

<details>
<summary>정답 및 해설</summary>

**정답: (c)**

Terraform의 핵심 워크플로우는 다음과 같다.

1. `terraform init` — Provider 플러그인 다운로드, 백엔드 초기화, 모듈 다운로드
2. `terraform plan` — 현재 상태(State)와 원하는 상태(코드)를 비교하여 변경 계획 생성 (Dry Run)
3. `terraform apply` — 계획된 변경 사항을 실제 인프라에 적용
4. `terraform destroy` — 관리 중인 모든 리소스 삭제

`init`은 항상 가장 먼저 실행해야 하며, `plan` 없이 바로 `apply`할 수도 있지만, 변경 사항을 사전에 확인하는 것이 베스트 프랙티스이다.
</details>

---

**문제 A-2.**  
Terraform에서 이미 존재하는 AWS 리소스 정보를 읽기 전용으로 참조할 때 사용하는 블록은?

(a) `resource`  
(b) `data`  
(c) `module`  
(d) `variable`

<details>
<summary>정답 및 해설</summary>

**정답: (b)**

`data` 블록(Data Source)은 이미 존재하는 리소스나 외부 데이터를 읽기 전용으로 참조한다. 예를 들어, `data "aws_ami" "latest" { ... }`는 최신 AMI 정보를 조회하지만 새로운 AMI를 생성하지 않는다. `resource` 블록은 새 리소스를 생성·관리하고, `module`은 재사용 가능한 코드 패키지를, `variable`은 입력값을 정의한다.
</details>

---

**문제 A-3.**  
다음 코드에서 `aws_instance.web`의 퍼블릭 IP를 출력하려면 빈칸에 무엇을 넣어야 하는가?

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}

output "web_ip" {
  value = _______________
}
```

<details>
<summary>정답 및 해설</summary>

**정답:** `aws_instance.web.public_ip`

Terraform에서 리소스의 속성을 참조할 때는 `<RESOURCE_TYPE>.<NAME>.<ATTRIBUTE>` 형식을 사용한다. `aws_instance` 리소스는 생성 후 `public_ip`, `private_ip`, `id`, `arn` 등의 속성을 제공한다. 이러한 값은 리소스가 실제로 생성된 후에야 알 수 있으므로 "computed attribute"라고 부른다.
</details>

---

**문제 A-4.**  
변수에 `sensitive = true`를 설정하면 어떤 효과가 있는가?

(a) 변수 값이 자동으로 암호화되어 State에 저장된다  
(b) `terraform plan`과 `apply` 출력에서 변수 값이 마스킹된다  
(c) 변수를 환경 변수로만 전달할 수 있다  
(d) 변수 값이 State 파일에 저장되지 않는다  

<details>
<summary>정답 및 해설</summary>

**정답: (b)**

`sensitive = true`는 `plan`과 `apply`의 콘솔 출력에서 해당 값을 `(sensitive value)`로 마스킹한다. **주의:** State 파일에는 평문으로 저장된다. 따라서 State 파일 자체의 암호화(S3 버킷 암호화, Azure Blob 암호화 등)가 반드시 필요하다. 변수 전달 방식에는 제한이 없다.
</details>

---

**문제 A-5.**  
다음 변수 정의가 유효하지 않은 이유는?

```hcl
variable "name" {
  type    = string
  default = "web-server"
}

variable "name" {
  type    = string
  default = "app-server"
}
```

<details>
<summary>정답 및 해설</summary>

같은 이름의 변수를 두 번 정의하면 **중복 선언 오류**가 발생한다. Terraform에서 모든 변수, 리소스, 데이터 소스, 출력값의 이름은 해당 모듈 내에서 고유해야 한다. `terraform validate` 실행 시 `Duplicate variable "name" definition` 에러가 발생한다.
</details>

---

**문제 A-6.**  
`terraform.tfvars` 파일과 `-var` CLI 플래그를 동시에 사용하여 같은 변수에 다른 값을 지정하면 어떤 값이 적용되는가?

<details>
<summary>정답 및 해설</summary>

**`-var` CLI 플래그의 값이 적용된다.**

변수 우선순위(낮 → 높):
1. `default` 값
2. 환경 변수 (`TF_VAR_xxx`)
3. `terraform.tfvars` / `*.auto.tfvars`
4. `-var-file` CLI 플래그
5. `-var` CLI 플래그

뒤에 오는 것일수록 우선순위가 높다. `-var`가 가장 높으므로, `terraform.tfvars`의 값을 덮어쓴다.
</details>

---

**문제 A-7.**  
다음 코드에서 `terraform apply` 실행 시 EC2 인스턴스는 몇 개 생성되는가?

```hcl
variable "create_instance" {
  default = false
}

resource "aws_instance" "app" {
  count         = var.create_instance ? 3 : 0
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

<details>
<summary>정답 및 해설</summary>

**0개**

`var.create_instance`의 기본값이 `false`이므로 삼항 연산자는 `0`을 반환한다. `count = 0`이면 해당 리소스는 전혀 생성되지 않는다. 이 패턴은 조건부 리소스 생성에 자주 사용된다.
</details>

---

**문제 A-8.**  
Terraform State 파일의 주요 역할이 아닌 것은?

(a) 코드에 정의된 리소스와 실제 클라우드 리소스 간의 매핑 저장  
(b) 리소스 간 의존성 그래프 추적  
(c) 클라우드 Provider의 API 인증 정보 저장  
(d) `plan` 시 변경 사항 계산의 기준점 제공  

<details>
<summary>정답 및 해설</summary>

**정답: (c)**

State 파일은 API 인증 정보를 저장하지 않는다. 인증 정보는 Provider 설정, 환경 변수, 또는 AWS CLI 프로필 등 별도의 인증 메커니즘을 통해 관리된다. State 파일은 리소스 매핑(a), 의존성 추적(b), 변경 계산(d)을 위한 것이다. 단, State 파일에는 리소스 속성값(DB 비밀번호 등)이 평문으로 포함될 수 있으므로 보안에 주의해야 한다.
</details>

---

**문제 A-9.**  
다음 Azure 리소스에서 `location` 값을 Resource Group과 동일하게 설정하려면?

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-demo"
  location = "koreacentral"
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-demo"
  address_space       = ["10.0.0.0/16"]
  location            = _______________
  resource_group_name = azurerm_resource_group.main.name
}
```

<details>
<summary>정답 및 해설</summary>

**정답:** `azurerm_resource_group.main.location`

다른 리소스의 속성을 참조할 때는 `<TYPE>.<NAME>.<ATTRIBUTE>` 형식을 사용한다. 이렇게 하면 Resource Group의 location이 변경될 때 VNet의 location도 자동으로 변경된다. 또한 Terraform은 이 참조를 통해 암시적 의존성을 파악하여, Resource Group이 먼저 생성된 후 VNet을 생성한다.
</details>

---

**문제 A-10.**  
`terraform destroy`를 실행했는데 특정 RDS 인스턴스에서 `prevent_destroy = true`가 설정되어 있으면 어떻게 되는가?

<details>
<summary>정답 및 해설</summary>

**전체 `destroy` 작업이 에러와 함께 중단된다.**

`lifecycle { prevent_destroy = true }`가 설정된 리소스를 삭제하려고 하면 Terraform은 에러를 발생시키고 작업을 중단한다. 이 설정은 프로덕션 데이터베이스 같은 중요 리소스의 실수로 인한 삭제를 방지하는 안전장치이다. 해당 리소스를 정말 삭제하려면 먼저 코드에서 `prevent_destroy = false`로 변경하고 `apply`한 뒤 `destroy`해야 한다.
</details>

---

### 섹션 B: 중급 문제 (10문제)

---

**문제 B-1.**  
`count`와 `for_each`의 핵심 차이점을 설명하고, `for_each`를 사용해야 하는 상황의 예시를 들어라.

<details>
<summary>정답 및 해설</summary>

**핵심 차이점:**

`count`는 **인덱스 기반**이다. 리소스는 `aws_subnet.public[0]`, `aws_subnet.public[1]` 형태로 식별된다. 중간 요소(예: `[1]`)를 삭제하면, `[2]`가 `[1]`로 밀려오면서 **의도치 않게 재생성**된다.

`for_each`는 **키 기반**이다. 리소스는 `aws_subnet.public["2a"]`, `aws_subnet.public["2c"]` 형태로 식별된다. 특정 키를 삭제해도 다른 리소스에 영향이 없다.

**for_each를 사용해야 하는 상황:**

서브넷 3개(`2a`, `2b`, `2c`)를 관리하다가 `2b`를 제거해야 하는 경우, `count`를 사용하면 `2c` 서브넷이 인덱스 변경으로 삭제 후 재생성된다. `for_each`를 사용하면 `2b`만 삭제되고 `2a`, `2c`는 그대로 유지된다.

**원칙:** 리소스가 교환 가능한(interchangeable) 것이 아니라면 `for_each`를 사용하라.
</details>

---

**문제 B-2.**  
다음 코드의 문제점을 찾아 수정하라.

```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "http" {
  type              = "ingress"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
}

resource "aws_instance" "web" {
  ami             = "ami-12345678"
  instance_type   = "t3.micro"
  security_groups = [aws_security_group.web.name]
  subnet_id       = aws_subnet.public.id
}
```

<details>
<summary>정답 및 해설</summary>

**문제점:** VPC 내 EC2 인스턴스에서 `security_groups`가 아니라 `vpc_security_group_ids`를 사용해야 한다.

`security_groups`는 EC2-Classic 또는 기본 VPC에서 보안 그룹 **이름**으로 참조할 때 사용한다. 커스텀 VPC에서는 `vpc_security_group_ids`를 사용하여 보안 그룹 **ID**를 전달해야 한다.

**수정 코드:**
```hcl
resource "aws_instance" "web" {
  ami                    = "ami-12345678"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]  # 수정
  subnet_id              = aws_subnet.public.id
}
```

이 실수는 `plan` 단계에서 에러가 발생하지 않을 수 있으나, `apply` 시 보안 그룹이 연결되지 않는 문제가 생긴다.
</details>

---

**문제 B-3.**  
다음 `dynamic` 블록이 생성하는 보안 그룹 규칙의 수는?

```hcl
variable "ports" {
  default = [80, 443, 8080]
}

variable "cidrs" {
  default = ["10.0.0.0/8", "172.16.0.0/12"]
}

locals {
  rules = flatten([
    for port in var.ports : [
      for cidr in var.cidrs : {
        port = port
        cidr = cidr
      }
    ]
  ])
}

resource "aws_security_group" "app" {
  name = "app-sg"

  dynamic "ingress" {
    for_each = local.rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

<details>
<summary>정답 및 해설</summary>

**6개**

`flatten`의 중첩 `for` 루프는 `ports`(3개) × `cidrs`(2개) = 6개의 조합을 생성한다:
- 80 + 10.0.0.0/8, 80 + 172.16.0.0/12
- 443 + 10.0.0.0/8, 443 + 172.16.0.0/12
- 8080 + 10.0.0.0/8, 8080 + 172.16.0.0/12

`flatten` 함수는 중첩 리스트를 1차원으로 펼치고, `dynamic` 블록의 `for_each`가 각 요소마다 하나의 `ingress` 규칙을 생성한다.
</details>

---

**문제 B-4.**  
모듈의 출력값으로 `sensitive = true`인 값을 전달할 때 주의해야 할 점은?

<details>
<summary>정답 및 해설</summary>

`sensitive = true`인 모듈 출력값을 참조하는 쪽에서도 해당 값이 **자동으로 sensitive로 전파**된다. 따라서 이 값을 사용하는 모든 곳에서 출력에 마스킹이 적용된다.

주의할 점:
1. 모듈 output에도 `sensitive = true`를 명시해야 한다. 그렇지 않으면 Terraform이 에러를 발생시킨다.
2. `nonsensitive()` 함수로 명시적으로 해제할 수 있지만, 보안상 위험하므로 신중히 사용해야 한다.
3. **State 파일에는 여전히 평문으로 저장**되므로, State 파일 자체의 보안이 핵심이다.

```hcl
# 모듈 내부
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true   # 필수
}

# 호출 측에서 사용
resource "aws_ssm_parameter" "db_pass" {
  name  = "/app/db/password"
  type  = "SecureString"
  value = module.rds.db_password  # 자동으로 sensitive 처리됨
}
```
</details>

---

**문제 B-5.**  
다음 코드에서 `terraform apply` 시 어떤 문제가 발생하며, 어떻게 해결하는가?

```hcl
resource "aws_instance" "app" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private[count.index].id
}

output "instance_ids" {
  value = aws_instance.app.*.id
}
```

이 상태에서 `count`를 `3`에서 `2`로 변경하면?

<details>
<summary>정답 및 해설</summary>

`count`를 3에서 2로 변경하면 `aws_instance.app[2]`(세 번째 인스턴스)가 **삭제(destroy)**된다.

이 동작 자체는 정상이지만, 문제가 되는 시나리오는 다음과 같다. 만약 `count = 3`인 상태에서 리스트의 첫 번째 서브넷을 제거하면, 인덱스가 밀려서 `app[0]`에는 이전의 `app[1]` 서브넷이, `app[1]`에는 이전의 `app[2]` 서브넷이 할당되어 **기존 인스턴스 2개가 변경(또는 재생성)**된다.

**해결 방법:** `for_each`로 전환하여 키 기반 관리를 사용한다.

```hcl
resource "aws_instance" "app" {
  for_each      = toset(["app-1", "app-2", "app-3"])
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private[each.key].id
}
```

추가로, `aws_instance.app.*.id`는 legacy splat 표현식이다. `[for i in aws_instance.app : i.id]` 또는 `values(aws_instance.app)[*].id`가 권장된다.
</details>

---

**문제 B-6.**  
`terraform plan`에서 `~ (update in-place)`, `+/- (destroy and recreate)`, `+ (create)`, `- (destroy)` 기호의 의미를 각각 설명하라. 또한, 어떤 속성 변경이 `update in-place`이고 어떤 것이 `destroy and recreate`인지 판단하는 기준은?

<details>
<summary>정답 및 해설</summary>

- `+` (create): 새 리소스 생성
- `-` (destroy): 기존 리소스 삭제
- `~` (update in-place): 리소스를 삭제하지 않고 속성만 변경
- `+/-` 또는 `-/+` (destroy and recreate): 리소스 삭제 후 재생성. `-/+`는 `create_before_destroy` 설정에 따라 새 리소스를 먼저 만든 후 기존 것을 삭제

**판단 기준:** Provider가 결정한다. 각 리소스의 속성은 Provider 코드에서 `ForceNew: true`(재생성 필요) 또는 `false`(in-place 업데이트 가능)로 정의되어 있다.

예를 들어 EC2 인스턴스에서:
- `tags` 변경 → update in-place (서버 유지)
- `instance_type` 변경 → stop + update (서버 중지 후 변경)
- `ami` 변경 → destroy and recreate (서버 완전 재생성)
- `subnet_id` 변경 → destroy and recreate

`plan` 출력을 반드시 확인하여 의도치 않은 재생성을 방지해야 한다.
</details>

---

**문제 B-7.**  
다음 `templatefile` 사용법에서 `config.tpl`의 내용을 작성하라.

```hcl
resource "aws_instance" "app" {
  user_data = templatefile("${path.module}/config.tpl", {
    db_host = "mydb.xxx.rds.amazonaws.com"
    db_port = 5432
    app_env = "production"
    features = ["auth", "logging", "metrics"]
  })
}
```

출력은 아래와 같아야 한다:
```
#!/bin/bash
export DB_HOST=mydb.xxx.rds.amazonaws.com
export DB_PORT=5432
export APP_ENV=production
export FEATURES=auth,logging,metrics
```

<details>
<summary>정답 및 해설</summary>

```tpl
#!/bin/bash
export DB_HOST=${db_host}
export DB_PORT=${db_port}
export APP_ENV=${app_env}
export FEATURES=${join(",", features)}
```

`templatefile` 함수는 `${ }` 구문으로 변수를 보간한다. 리스트는 `join()` 함수로 문자열로 변환한다. 반복문도 사용 가능하다:

```tpl
%{ for feature in features ~}
echo "Enabling feature: ${feature}"
%{ endfor ~}
```

`~` 기호는 앞뒤 공백/개행을 제거하는 트림 마커이다.
</details>

---

**문제 B-8.**  
`terraform state mv`와 `moved` 블록의 차이점을 설명하고, 각각 언제 사용하는 것이 적절한가?

<details>
<summary>정답 및 해설</summary>

| 구분 | `terraform state mv` | `moved` 블록 |
|------|---------------------|-------------|
| 실행 방식 | CLI에서 수동 실행 | 코드에 선언, plan/apply 시 자동 |
| 추적 | 실행 기록이 남지 않음 | 코드에 이력이 남음 (Git 추적) |
| 팀 협업 | 모든 팀원이 직접 실행해야 함 | 코드 반영만으로 전파됨 |
| 적용 시점 | 즉시 | 다음 plan/apply 시 |

**`terraform state mv` 사용 시나리오:**
- 긴급 State 복구 작업
- 일회성 마이그레이션
- State 파일 간 리소스 이동

**`moved` 블록 사용 시나리오 (권장):**
- 리소스 이름 변경
- 모듈 구조 리팩토링
- `count`에서 `for_each`로 전환
- 팀 환경에서 안전한 마이그레이션

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.app
}
```
</details>

---

**문제 B-9.**  
다음 코드에서 `cidrsubnet("10.0.0.0/16", 8, 5)`의 결과값은?

<details>
<summary>정답 및 해설</summary>

**결과: `"10.0.5.0/24"`**

`cidrsubnet(prefix, newbits, netnum)` 함수 동작:
1. 기본 CIDR: `10.0.0.0/16` → 서브넷 마스크 16비트
2. `newbits = 8` → 새 서브넷 마스크 = 16 + 8 = `/24`
3. `netnum = 5` → 5번째 서브넷

`/16`에서 `/24`로 나누면 256개의 서브넷이 가능하고, 5번째는:
- 0번: 10.0.0.0/24
- 1번: 10.0.1.0/24
- ...
- 5번: **10.0.5.0/24**

이 함수는 서브넷 CIDR을 계산할 때 매우 유용하며, 하드코딩을 피할 수 있다.
</details>

---

**문제 B-10.**  
Terraform에서 `depends_on`을 명시적으로 사용해야 하는 경우와, 사용하지 않아야 하는 경우를 각각 설명하라.

<details>
<summary>정답 및 해설</summary>

**사용해야 하는 경우:**

Terraform이 코드에서 의존성을 자동으로 감지할 수 없을 때. 예를 들어, IAM 정책이 EC2 인스턴스에 직접 참조되지 않지만, 인스턴스가 기동 시 해당 권한이 필요한 경우:

```hcl
resource "aws_iam_role_policy" "app" {
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.app.json
}

resource "aws_instance" "app" {
  # ... IAM 정책을 직접 참조하지 않지만 필요
  depends_on = [aws_iam_role_policy.app]
}
```

**사용하지 않아야 하는 경우:**

리소스 간에 이미 속성 참조를 통한 **암시적 의존성**이 있을 때. 불필요한 `depends_on`은 병렬 실행을 방해하여 성능을 저하시키고, 코드 가독성을 떨어뜨린다.

```hcl
# 나쁜 예: 불필요한 depends_on
resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id  # 이미 암시적 의존성 존재
  depends_on = [aws_subnet.public]  # 불필요
}
```
</details>

---

### 섹션 C: 고급 문제 (10문제)

---

**문제 C-1.**  
프로덕션 환경에서 Terraform State를 안전하게 관리하기 위한 전략을 5가지 이상 설명하라.

<details>
<summary>정답 및 해설</summary>

1. **원격 백엔드 사용:** S3, Azure Blob, GCS 등 원격 스토리지에 State를 저장하여 팀 공유 및 안전한 접근을 보장한다.

2. **State Locking:** DynamoDB(AWS), Blob Lease(Azure) 등을 통해 동시 수정을 방지한다. 두 명이 동시에 `apply`하면 State가 손상될 수 있다.

3. **State 암호화:** 전송 중(TLS) 및 저장 시(SSE-S3, SSE-KMS, Azure Storage Service Encryption) 모두 암호화한다. State에는 DB 비밀번호 같은 민감 정보가 평문으로 포함될 수 있다.

4. **버전 관리:** S3 버킷 버전 관리를 활성화하여 State 파일의 이전 버전을 복구할 수 있도록 한다.

5. **접근 제어:** IAM 정책으로 State 파일에 대한 접근을 최소 권한 원칙에 따라 제한한다. 읽기/쓰기 권한을 분리한다.

6. **State 분리:** 환경(dev/staging/prod)별, 팀별, 서비스별로 State를 분리하여 blast radius를 줄인다. 하나의 State 변경이 다른 환경에 영향을 미치지 않도록 한다.

7. **State 감사 로깅:** CloudTrail(AWS), Azure Activity Log 등으로 State 파일 접근 이력을 기록한다.

8. **자동 백업:** 정기적인 State 파일 백업과 복구 절차를 문서화한다.
</details>

---

**문제 C-2.**  
다음 시나리오에서 zero-downtime 배포를 달성하기 위한 Terraform 전략을 설계하라.

> ECS Fargate 서비스를 운영 중이며, 새 컨테이너 이미지 배포 시 기존 서비스가 중단되지 않아야 한다.

<details>
<summary>정답 및 해설</summary>

```hcl
resource "aws_ecs_service" "app" {
  name            = "app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3

  # 핵심 1: 롤링 업데이트 설정
  deployment_maximum_percent         = 200   # 최대 6개 태스크 (배포 중)
  deployment_minimum_healthy_percent = 100   # 항상 3개 이상 유지

  # 핵심 2: Circuit Breaker 활성화
  deployment_circuit_breaker {
    enable   = true
    rollback = true   # 실패 시 자동 롤백
  }

  # 핵심 3: 로드밸런서 연동
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }

  # 핵심 4: Force new deployment on task def change
  force_new_deployment = true

  lifecycle {
    # 태스크 수가 오토스케일링에 의해 변경될 수 있으므로 무시
    ignore_changes = [desired_count]
  }
}

# 핵심 5: Target Group의 Health Check
resource "aws_lb_target_group" "app" {
  # ...
  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 15
    timeout             = 5
    matcher             = "200"
  }

  # Deregistration delay — 기존 연결 처리 시간
  deregistration_delay = 30
}
```

**동작 흐름:**
1. 새 Task Definition이 등록됨
2. ECS가 새 태스크를 추가로 시작 (최대 200% = 6개)
3. 새 태스크가 ALB Health Check 통과
4. ALB가 트래픽을 새 태스크로 분산
5. 기존 태스크가 draining 후 종료
6. 실패 시 Circuit Breaker가 자동 롤백
</details>

---

**문제 C-3.**  
Terraform으로 멀티 리전 DR(Disaster Recovery) 아키텍처를 구현하는 코드를 작성하라. (AWS, Active-Passive)

<details>
<summary>정답 및 해설</summary>

```hcl
# Provider 설정 — 두 리전
provider "aws" {
  alias  = "primary"
  region = "ap-northeast-2"   # 서울 (Active)
}

provider "aws" {
  alias  = "dr"
  region = "ap-southeast-1"   # 싱가포르 (Passive)
}

# Primary 리전 VPC
module "vpc_primary" {
  source    = "./modules/vpc"
  providers = { aws = aws.primary }

  name_prefix = "myapp-prod-primary"
  vpc_cidr    = "10.0.0.0/16"
  # ...
}

# DR 리전 VPC
module "vpc_dr" {
  source    = "./modules/vpc"
  providers = { aws = aws.dr }

  name_prefix = "myapp-prod-dr"
  vpc_cidr    = "10.1.0.0/16"
  # ...
}

# RDS Cross-Region Read Replica
resource "aws_db_instance" "primary" {
  provider = aws.primary

  identifier        = "myapp-db-primary"
  engine            = "postgres"
  instance_class    = "db.r6g.large"
  allocated_storage = 100

  backup_retention_period = 35
  # ...
}

resource "aws_db_instance" "dr_replica" {
  provider = aws.dr

  identifier          = "myapp-db-dr"
  replicate_source_db = aws_db_instance.primary.arn
  instance_class      = "db.r6g.large"

  # DR 승격 시 바로 사용할 수 있도록 동일 스펙
}

# S3 Cross-Region Replication
resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "myapp-assets-primary"
}

resource "aws_s3_bucket" "dr" {
  provider = aws.dr
  bucket   = "myapp-assets-dr"
}

resource "aws_s3_bucket_replication_configuration" "primary_to_dr" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  role     = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD"
    }
  }
}

# Route53 Health Check + Failover
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.myapp.com"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10
}

resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app.myapp.com"
  type    = "A"

  # Primary
  alias {
    name                   = module.alb_primary.dns_name
    zone_id                = module.alb_primary.zone_id
    evaluate_target_health = true
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  failover_routing_policy {
    type = "PRIMARY"
  }
}

resource "aws_route53_record" "app_dr" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app.myapp.com"
  type    = "A"

  alias {
    name                   = module.alb_dr.dns_name
    zone_id                = module.alb_dr.zone_id
    evaluate_target_health = true
  }

  set_identifier = "dr"

  failover_routing_policy {
    type = "SECONDARY"
  }
}
```

**핵심 포인트:**
- Provider alias로 멀티 리전 리소스를 단일 코드베이스에서 관리
- RDS Cross-Region Read Replica로 데이터 동기화
- Route53 Failover 라우팅으로 자동 전환
- DR 리전도 항상 최소한의 인프라를 유지하여 RTO를 단축
</details>

---

**문제 C-4.**  
Terraform으로 관리 중인 리소스를 `terraform state rm`으로 State에서 제거한 후, 다시 `terraform import`로 가져오면 어떤 일이 발생하는가? 데이터 손실이 있는가?

<details>
<summary>정답 및 해설</summary>

**데이터 손실은 없다.** 이 과정을 단계별로 살펴보면:

1. `terraform state rm aws_db_instance.main`
   - State 파일에서 RDS 매핑 정보만 삭제된다
   - **실제 RDS 인스턴스는 그대로 존재**한다
   - 이 시점에서 Terraform은 해당 리소스의 존재를 모른다

2. `terraform plan`
   - 코드에 `aws_db_instance.main`이 있지만 State에 없으므로 "새로 생성" 계획이 표시된다
   - **이 상태에서 `apply`하면 동일 이름/식별자의 리소스 생성을 시도하여 충돌 에러가 발생**한다

3. `terraform import aws_db_instance.main myapp-db`
   - 기존 실제 리소스의 현재 상태를 읽어와 State에 기록한다
   - 리소스 데이터에는 영향 없다

4. `terraform plan`
   - State와 코드를 비교하여 차이점을 표시한다
   - import는 모든 속성을 가져오지 못할 수 있으므로, 코드와 실제 상태 간 차이가 나타날 수 있다
   - `plan`에서 "no changes"가 될 때까지 코드를 조정해야 한다

**핵심:** `state rm`은 Terraform의 "인식"만 제거하며, 실제 인프라에는 아무런 영향이 없다. 이 작업은 주로 리소스를 다른 State로 옮기거나, Terraform 관리 대상에서 제외할 때 사용한다.
</details>

---

**문제 C-5.**  
대규모 Terraform 프로젝트에서 `plan`/`apply` 시간이 매우 오래 걸린다. 성능을 개선하기 위한 전략 5가지를 제시하라.

<details>
<summary>정답 및 해설</summary>

1. **State 분리(State Splitting):** 하나의 거대한 State를 서비스별, 레이어별로 분리한다. 예: networking, compute, database, monitoring. 각 State가 관리하는 리소스 수가 줄어들어 plan 시간이 단축된다.

2. **`-target` 플래그 (주의해서 사용):** 특정 리소스만 plan/apply한다. 단, 의존성 문제가 발생할 수 있으므로 긴급 상황에서만 사용하고, 이후 전체 plan을 반드시 실행한다.
   ```bash
   terraform plan -target=module.ecs
   ```

3. **`-refresh=false` 옵션:** plan 시 모든 리소스의 현재 상태를 API로 확인하는 refresh 단계를 건너뛴다. State가 최신이라 확신할 때만 사용한다.

4. **`-parallelism` 증가:** 기본값 10을 늘려 동시 API 호출 수를 증가시킨다. Provider의 API rate limit에 주의해야 한다.
   ```bash
   terraform apply -parallelism=30
   ```

5. **Data Source 최소화:** 불필요한 `data` 블록은 매번 API를 호출하므로 제거하거나 변수로 대체한다. 특히 `data "aws_ami"`처럼 조건이 복잡한 조회는 성능에 영향을 줄 수 있다.

6. **Provider 캐싱:** Terraform 1.4+에서 Provider 플러그인 캐시를 설정하여 `init` 시간을 단축한다.

7. **모듈 설계 최적화:** 불필요하게 큰 모듈을 작은 단위로 분리하고, 모듈 간 의존성을 최소화한다.
</details>

---

**문제 C-6.**  
다음 시나리오에 대한 Terraform 코드를 작성하라.

> AWS에서 3개의 서로 다른 계정(dev, staging, prod)에 동일한 인프라를 배포해야 한다. 각 계정에는 서로 다른 IAM Role을 Assume해야 한다.

<details>
<summary>정답 및 해설</summary>

```hcl
# variables.tf
variable "accounts" {
  type = map(object({
    account_id    = string
    role_arn      = string
    instance_type = string
    region        = string
  }))
  default = {
    dev = {
      account_id    = "111111111111"
      role_arn      = "arn:aws:iam::111111111111:role/TerraformRole"
      instance_type = "t3.micro"
      region        = "ap-northeast-2"
    }
    staging = {
      account_id    = "222222222222"
      role_arn      = "arn:aws:iam::222222222222:role/TerraformRole"
      instance_type = "t3.small"
      region        = "ap-northeast-2"
    }
    prod = {
      account_id    = "333333333333"
      role_arn      = "arn:aws:iam::333333333333:role/TerraformRole"
      instance_type = "t3.medium"
      region        = "ap-northeast-2"
    }
  }
}

# providers.tf
provider "aws" {
  alias  = "dev"
  region = var.accounts["dev"].region
  assume_role {
    role_arn = var.accounts["dev"].role_arn
  }
}

provider "aws" {
  alias  = "staging"
  region = var.accounts["staging"].region
  assume_role {
    role_arn = var.accounts["staging"].role_arn
  }
}

provider "aws" {
  alias  = "prod"
  region = var.accounts["prod"].region
  assume_role {
    role_arn = var.accounts["prod"].role_arn
  }
}

# main.tf — 모듈로 각 환경 배포
module "infra_dev" {
  source    = "./modules/infra"
  providers = { aws = aws.dev }

  environment   = "dev"
  instance_type = var.accounts["dev"].instance_type
  tags          = { Account = var.accounts["dev"].account_id }
}

module "infra_staging" {
  source    = "./modules/infra"
  providers = { aws = aws.staging }

  environment   = "staging"
  instance_type = var.accounts["staging"].instance_type
  tags          = { Account = var.accounts["staging"].account_id }
}

module "infra_prod" {
  source    = "./modules/infra"
  providers = { aws = aws.prod }

  environment   = "prod"
  instance_type = var.accounts["prod"].instance_type
  tags          = { Account = var.accounts["prod"].account_id }
}
```

**대안적 접근 (Terragrunt):** 실무에서는 Terragrunt를 사용하여 계정별 디렉토리에서 독립적으로 실행하는 것이 더 안전하다. 하나의 State에 모든 계정의 리소스를 넣으면 blast radius가 커지기 때문이다.
</details>

---

**문제 C-7.**  
Terraform에서 Circular Dependency(순환 의존성)가 발생하는 코드 예시를 보여주고, 해결 방법을 설명하라.

<details>
<summary>정답 및 해설</summary>

**순환 의존성 예시:**

```hcl
# SG A가 SG B를 참조하고, SG B가 SG A를 참조
resource "aws_security_group" "a" {
  name = "sg-a"
  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.b.id]  # B 참조
  }
}

resource "aws_security_group" "b" {
  name = "sg-b"
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.a.id]  # A 참조 → 순환!
  }
}
```

Terraform은 `Error: Cycle` 에러를 발생시킨다.

**해결 방법: 보안 그룹과 규칙을 분리**

```hcl
# 1. 보안 그룹만 먼저 생성 (규칙 없이)
resource "aws_security_group" "a" {
  name = "sg-a"
}

resource "aws_security_group" "b" {
  name = "sg-b"
}

# 2. 규칙을 별도 리소스로 분리
resource "aws_security_group_rule" "a_from_b" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
}

resource "aws_security_group_rule" "b_from_a" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  security_group_id        = aws_security_group.b.id
  source_security_group_id = aws_security_group.a.id
}
```

이렇게 하면 SG 생성은 독립적이고, 규칙이 이미 존재하는 SG ID를 참조하므로 순환이 깨진다.
</details>

---

**문제 C-8.**  
Terraform 1.5+의 `check` 블록과 `precondition`/`postcondition`의 차이점을 코드 예시와 함께 설명하라.

<details>
<summary>정답 및 해설</summary>

**precondition/postcondition:** 리소스의 lifecycle 블록 안에서 사용하며, 조건이 실패하면 **plan/apply를 중단**시킨다.

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
      error_message = "허용되지 않는 인스턴스 타입입니다."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "퍼블릭 IP 할당에 실패했습니다."
    }
  }
}
```

**check 블록 (Terraform 1.5+):** 리소스와 독립적으로 존재하며, 조건 실패 시 **경고(warning)만 표시**하고 apply는 계속 진행된다. 모니터링/감사 목적에 적합하다.

```hcl
check "website_health" {
  data "http" "app_health" {
    url = "https://app.mycompany.com/health"
  }

  assert {
    condition     = data.http.app_health.status_code == 200
    error_message = "앱 헬스 체크 실패! 상태 코드: ${data.http.app_health.status_code}"
  }
}

check "certificate_expiry" {
  data "aws_acm_certificate" "app" {
    domain = "app.mycompany.com"
  }

  assert {
    condition = timecmp(
      data.aws_acm_certificate.app.not_after,
      timeadd(timestamp(), "720h")  # 30일 후
    ) > 0
    error_message = "SSL 인증서가 30일 이내에 만료됩니다!"
  }
}
```

| 구분 | precondition/postcondition | check 블록 |
|------|---------------------------|-----------|
| 실패 시 | plan/apply 중단 (에러) | 경고만 표시 (계속 진행) |
| 위치 | 리소스의 lifecycle 내부 | 최상위 레벨 (독립적) |
| 용도 | 배포 안전장치 | 인프라 상태 모니터링 |
| Data Source | 사용 불가 | 자체 data 블록 사용 가능 |
</details>

---

**문제 C-9.**  
Terraform으로 Blue-Green 배포를 구현하는 전략을 설명하고, 핵심 코드를 작성하라.

<details>
<summary>정답 및 해설</summary>

Blue-Green 배포는 두 개의 동일한 환경(Blue/Green)을 유지하고, 트래픽을 전환하는 방식이다.

```hcl
variable "active_color" {
  description = "현재 활성 환경 (blue 또는 green)"
  type        = string
  default     = "blue"

  validation {
    condition     = contains(["blue", "green"], var.active_color)
    error_message = "active_color는 blue 또는 green이어야 합니다."
  }
}

variable "blue_image_tag" {
  default = "v1.0.0"
}

variable "green_image_tag" {
  default = "v1.1.0"
}

# Blue 환경
module "blue" {
  source = "./modules/app-environment"

  name        = "blue"
  image_tag   = var.blue_image_tag
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnets
  desired_count = var.active_color == "blue" ? 3 : 1  # 비활성 환경은 최소 유지
}

# Green 환경
module "green" {
  source = "./modules/app-environment"

  name        = "green"
  image_tag   = var.green_image_tag
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnets
  desired_count = var.active_color == "green" ? 3 : 1
}

# ALB Listener — 활성 환경으로 트래픽 전달
resource "aws_lb_listener" "app" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = var.acm_cert_arn

  default_action {
    type             = "forward"
    target_group_arn = var.active_color == "blue" ? module.blue.target_group_arn : module.green.target_group_arn
  }
}

# 가중치 기반 점진적 전환 (Canary)
resource "aws_lb_listener_rule" "weighted" {
  listener_arn = aws_lb_listener.app.arn
  priority     = 1

  action {
    type = "forward"
    forward {
      target_group {
        arn    = module.blue.target_group_arn
        weight = var.active_color == "blue" ? 100 : 0
      }
      target_group {
        arn    = module.green.target_group_arn
        weight = var.active_color == "green" ? 100 : 0
      }
    }
  }

  condition {
    path_pattern { values = ["/*"] }
  }
}
```

**배포 프로세스:**
1. 비활성 환경(green)에 새 버전 배포: `green_image_tag = "v1.1.0"`
2. 비활성 환경 테스트 (별도 테스트용 리스너 또는 헤더 기반 라우팅)
3. 트래픽 전환: `active_color = "green"` → `terraform apply`
4. 문제 발생 시 즉시 롤백: `active_color = "blue"` → `terraform apply`
5. 안정화 후 이전 환경 축소
</details>

---

**문제 C-10.**  
다음 상황을 해결하기 위한 Terraform 코드와 전략을 작성하라.

> 기존에 수동으로 생성된 50개 이상의 AWS 리소스(VPC, 서브넷, EC2, RDS, S3 등)를 Terraform 관리로 전환해야 한다. 실제 인프라에 변경 없이 안전하게 마이그레이션하는 방법은?

<details>
<summary>정답 및 해설</summary>

**단계별 전략:**

**1단계: 현재 인프라 문서화**
```bash
# AWS CLI로 현재 리소스 목록 확인
aws ec2 describe-vpcs --output json > vpc_inventory.json
aws ec2 describe-instances --output json > ec2_inventory.json

# 또는 AWS Config / Resource Explorer 활용
```

**2단계: import 블록 작성 (Terraform 1.5+)**
```hcl
# imports.tf

# VPC
import {
  to = aws_vpc.main
  id = "vpc-0abc123def"
}

# 서브넷
import {
  to = aws_subnet.public["2a"]
  id = "subnet-0aaa111"
}

import {
  to = aws_subnet.public["2c"]
  id = "subnet-0bbb222"
}

# EC2 인스턴스
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}

# RDS
import {
  to = aws_db_instance.main
  id = "myapp-db-prod"
}

# S3
import {
  to = aws_s3_bucket.assets
  id = "myapp-prod-assets"
}
```

**3단계: 코드 자동 생성**
```bash
terraform plan -generate-config-out=generated_resources.tf
```

**4단계: 생성된 코드 정리 및 모듈화**
```bash
# generated_resources.tf를 검토하고 정리
# - 불필요한 computed attribute 제거
# - 변수화할 값 식별
# - 모듈 구조로 재구성
```

**5단계: No-Change 검증**
```bash
# 가장 중요한 단계: plan에서 변경 사항이 없어야 함
terraform plan

# 기대 출력:
# No changes. Your infrastructure matches the configuration.
```

**6단계: 반복적 정제**
```hcl
# plan에서 차이가 나는 경우 코드를 조정
# 예: 태그가 코드에 없으면 추가
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "prod-vpc"   # 실제 리소스에 있는 태그와 일치시킴
  }

  lifecycle {
    ignore_changes = [
      tags["LastModifiedBy"],  # 외부에서 변경되는 태그 무시
    ]
  }
}
```

**핵심 주의사항:**
- 한 번에 모든 리소스를 import하지 말고, 레이어별(네트워크 → 컴퓨트 → 데이터베이스)로 진행
- 각 단계에서 반드시 `plan`으로 "No changes" 확인
- 팀에 공유하고 코드 리뷰 진행
- 비프로덕션 환경에서 먼저 연습
- `import` 블록은 성공 후 제거해도 되지만, Git 이력에 남기는 것이 좋다
</details>

---

### 섹션 D: 시나리오 기반 서술형 문제 (5문제)

---

**문제 D-1.**  
당신의 팀에서 Terraform을 도입하려고 합니다. 다음 상황에서 Terraform의 디렉토리 구조, State 관리 전략, CI/CD 파이프라인 설계를 상세히 작성하세요.

> - 3개 환경 (dev, staging, prod)
> - 2개 AWS 계정 (dev/staging 공유, prod 별도)
> - 마이크로서비스 3개 (API, Worker, Frontend)
> - 공유 인프라 (VPC, RDS, ElastiCache)

<details>
<summary>모범 답안</summary>

**디렉토리 구조:**
```
terraform/
├── modules/                          # 재사용 모듈
│   ├── networking/                   # VPC, 서브넷, SG
│   ├── ecs-service/                  # ECS 서비스 템플릿
│   ├── rds/                          # RDS 클러스터
│   ├── elasticache/                  # ElastiCache
│   └── alb/                          # ALB + Target Group
│
├── shared-infra/                     # 공유 인프라 (VPC, DB 등)
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.tf                # s3://tf-state/shared/dev/
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── backend.tf                # s3://tf-state/shared/staging/
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── backend.tf                # s3://tf-state-prod/shared/prod/
│       └── terraform.tfvars
│
├── services/                         # 마이크로서비스별
│   ├── api/
│   │   ├── dev/    → backend: s3://tf-state/services/api/dev/
│   │   ├── staging/
│   │   └── prod/   → backend: s3://tf-state-prod/services/api/prod/
│   ├── worker/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   └── frontend/
│       ├── dev/
│       ├── staging/
│       └── prod/
│
└── global/                           # 계정 레벨 (IAM, Route53 등)
    ├── dev-account/
    └── prod-account/
```

**State 관리 전략:**

- Dev/Staging 계정: `s3://company-tf-state-dev` (하나의 버킷, 키로 분리)
- Prod 계정: `s3://company-tf-state-prod` (별도 계정의 별도 버킷)
- 각 State는 DynamoDB로 Locking
- S3 버킷 버전 관리 활성화
- Cross-account 접근은 IAM Role Assume

**State 분리 기준:** 공유 인프라와 서비스를 분리하여, 서비스 배포가 네트워크에 영향을 주지 않도록 한다. 서비스 간에도 독립 State를 사용하여 blast radius를 최소화한다. 서비스에서 공유 인프라의 값이 필요하면 `terraform_remote_state` data source로 참조한다.

**CI/CD 파이프라인:**

```
PR 생성
  → terraform fmt -check
  → terraform validate
  → tflint
  → checkov (보안 스캔)
  → terraform plan (PR 코멘트로 출력)
  → 코드 리뷰 필수 (CODEOWNERS)

PR Merge (main 브랜치)
  → dev: 자동 terraform apply
  → staging: 자동 apply + 통합 테스트
  → prod: 수동 승인 후 apply (GitHub Environment Protection)
```

각 서비스는 독립적인 워크플로우를 가지며, `paths` 필터로 변경된 디렉토리만 실행한다.
</details>

---

**문제 D-2.**  
금요일 오후 5시, `terraform apply` 실행 중 네트워크 오류로 작업이 중단되었습니다. State 파일이 잠겨있고, 일부 리소스는 생성되었지만 일부는 생성되지 않았습니다. 이 상황을 어떻게 복구하시겠습니까?

<details>
<summary>모범 답안</summary>

**1. 침착하게 현재 상태 파악**

```bash
# Lock 상태 확인
terraform plan
# Error: Error locking state: ...
# Lock Info:
#   ID:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#   Path:      s3://bucket/path/terraform.tfstate
```

**2. Lock 해제**
```bash
# 다른 사람이 작업 중이 아닌지 확인 (팀 채널에 공유)
# Lock을 잡고 있는 프로세스가 확실히 죽었는지 확인
terraform force-unlock xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**3. 현재 State 상태 확인**
```bash
# State에 기록된 리소스 목록 확인
terraform state list

# 실제 클라우드 리소스와 비교
terraform plan
```

**4. 시나리오별 대응:**

**시나리오 A:** Plan에서 "일부 리소스 생성 필요" 표시
→ 이미 생성된 리소스는 State에 있고, 미생성 리소스만 plan에 나타남
→ `terraform apply`를 다시 실행하면 나머지만 생성

**시나리오 B:** State가 손상된 경우
```bash
# S3 버전 관리에서 이전 State 복구
aws s3api list-object-versions --bucket tf-state --prefix path/terraform.tfstate
aws s3api get-object --bucket tf-state --prefix path/terraform.tfstate \
  --version-id "이전_버전_ID" terraform.tfstate.backup

# 복구된 State를 업로드하고 부분 생성된 리소스를 import
```

**시나리오 C:** 부분 생성된 리소스가 State에 없는 경우
```bash
# 수동으로 import
terraform import aws_instance.web i-0abc123
terraform import aws_subnet.new subnet-0xyz789
```

**5. 검증**
```bash
terraform plan
# "No changes" 또는 예상 가능한 변경만 있어야 함
```

**6. 재발 방지**
- S3 버킷 버전 관리 활성화 확인
- DynamoDB Lock 테이블 모니터링 알람 설정
- 금요일 오후에는 대규모 인프라 변경을 하지 않는 팀 규칙 수립
- CI/CD를 통한 apply를 표준으로 하여 로컬 실행 최소화
</details>

---

**문제 D-3.**  
Terraform으로 관리하는 프로덕션 RDS 인스턴스의 `engine_version`을 업그레이드해야 합니다. 다운타임을 최소화하면서 안전하게 업그레이드하는 전체 프로세스를 설명하세요.

<details>
<summary>모범 답안</summary>

**사전 준비 단계:**

1. **호환성 확인:** AWS 문서에서 현재 버전에서 목표 버전으로의 업그레이드 경로 확인
2. **스냅샷 생성:** 수동 스냅샷을 먼저 생성하여 롤백 가능하도록 준비
3. **유지보수 창 설정 확인:** `maintenance_window` 값 확인
4. **파라미터 그룹 호환성 확인:** 새 버전에 맞는 파라미터 그룹이 필요한지 확인

**Terraform 코드 변경:**

```hcl
resource "aws_db_instance" "main" {
  identifier     = "myapp-db-prod"
  engine         = "postgres"
  engine_version = "15.4"  # 14.x → 15.4로 변경

  # 핵심: 즉시 적용하지 않고 다음 유지보수 창에서 적용
  apply_immediately = false

  # 업그레이드 시 자동 마이너 버전 업그레이드 비활성화
  auto_minor_version_upgrade = false

  # 파라미터 그룹도 새 버전에 맞게 변경 (필요 시)
  parameter_group_name = aws_db_parameter_group.postgres15.name

  lifecycle {
    prevent_destroy = true

    # 중요: 업그레이드 중 일시적으로 ignore_changes에서
    # engine_version을 제거해야 함
  }
}

# 새 버전용 파라미터 그룹
resource "aws_db_parameter_group" "postgres15" {
  name   = "myapp-postgres15-params"
  family = "postgres15"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }
}
```

**실행 프로세스:**

1. `terraform plan`으로 변경 사항 확인
   - `~ engine_version: "14.10" → "15.4"` 표시
   - **destroy/recreate가 아닌 update in-place인지 반드시 확인**

2. 비프로덕션 환경에서 먼저 테스트
   ```bash
   cd environments/staging
   terraform apply
   # 업그레이드 완료 후 앱 테스트
   ```

3. 프로덕션 적용 (유지보수 창)
   ```bash
   cd environments/prod
   terraform apply
   # apply_immediately = false이므로 다음 maintenance_window에 실행
   ```

4. 즉시 적용이 필요한 경우
   ```hcl
   apply_immediately = true  # 주의: 다운타임 발생
   ```

5. **Multi-AZ 인스턴스의 경우:**
   - Standby가 먼저 업그레이드됨
   - Failover 수행 (약 60-120초 다운타임)
   - 이전 Primary가 업그레이드됨
   - 총 다운타임: 약 1-2분

**롤백 계획:**
- 메이저 버전 업그레이드는 롤백 불가 → 스냅샷에서 새 인스턴스 복원 필요
- 마이너 버전은 이전 버전으로 다시 변경 가능한 경우도 있음
- 롤백 시 새 RDS를 스냅샷에서 생성하고 DNS 전환하는 것이 가장 안전
</details>

---

**문제 D-4.**  
Terraform 모듈의 버전 관리 전략과 Breaking Change 처리 방법을 설명하세요.

<details>
<summary>모범 답안</summary>

**버전 관리 전략 — Semantic Versioning:**

```
v{MAJOR}.{MINOR}.{PATCH}

MAJOR: Breaking Changes (변수 삭제, 리소스 재구성 등)
MINOR: 새 기능 추가 (하위 호환)
PATCH: 버그 수정
```

**모듈 버전 게시 (Git Tags):**
```bash
git tag -a v1.0.0 -m "Initial stable release"
git push origin v1.0.0

git tag -a v1.1.0 -m "Add monitoring support"
git push origin v1.1.0

git tag -a v2.0.0 -m "BREAKING: Restructure subnet naming"
git push origin v2.0.0
```

**모듈 호출 시 버전 고정:**
```hcl
module "vpc" {
  source  = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0"
  # 또는 Terraform Registry
  source  = "app.terraform.io/company/vpc/aws"
  version = "~> 1.2"    # 1.2.x 최신, 2.0은 불가
}
```

**Breaking Change 처리 프로세스:**

1. **CHANGELOG 작성:** 변경 사항, 마이그레이션 가이드 상세 기술
2. **마이그레이션 가이드 제공:**
   ```markdown
   ## v2.0.0 Migration Guide

   ### Breaking Changes
   - `subnet_names` 변수가 `subnets` (map of objects)로 변경됨
   - Output `subnet_ids` → `subnet_id_map`으로 변경

   ### Migration Steps
   1. `moved` 블록 추가:
      moved {
        from = aws_subnet.main[0]
        to   = aws_subnet.main["public-2a"]
      }
   2. 변수 값 형식 변경
   3. terraform plan으로 no-change 확인
   ```

3. **Deprecation 기간:** v1.x에서 deprecated 경고를 먼저 추가하고, 충분한 기간 후 v2.0 릴리스

4. **자동화된 테스트:** 모듈 변경 시 CI에서 모든 사용처에 대해 `terraform plan` 실행

```hcl
# 모듈 내부에서 deprecation 경고
variable "old_name" {
  type    = string
  default = null

  validation {
    condition     = var.old_name == null
    error_message = "DEPRECATED: 'old_name' 변수는 v2.0에서 제거됩니다. 'new_name'을 사용하세요."
  }
}
```
</details>

---

**문제 D-5.**  
Terraform과 Kubernetes를 함께 사용할 때의 경계(boundary)를 어떻게 설정하는 것이 좋은가? Terraform으로 관리해야 할 것과 kubectl/Helm/ArgoCD로 관리해야 할 것을 구분하고 그 이유를 설명하세요.

<details>
<summary>모범 답안</summary>

**Terraform으로 관리해야 할 것 (인프라 레이어):**

- **클러스터 자체:** EKS/AKS 클러스터, 노드 그룹/풀, 네트워크 설정
- **클러스터 기반 인프라:** VPC, 서브넷, NAT Gateway, DNS
- **IAM/RBAC 기반 설정:** IRSA(IAM Roles for Service Accounts), Azure AD 연동
- **클러스터 애드온:** CoreDNS, kube-proxy, CNI 플러그인, CSI 드라이버
- **외부 의존성:** RDS, ElastiCache, S3, SQS 등 클러스터 외부 리소스
- **Namespace 생성:** 기본 네임스페이스 구조

```hcl
# Terraform: EKS 클러스터 + IRSA
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  # ...
}

# Terraform: 앱이 사용할 S3 버킷
resource "aws_s3_bucket" "app_data" {
  bucket = "myapp-data"
}

# Terraform: IRSA로 Pod에 AWS 권한 부여
module "irsa_app" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  role_name = "app-s3-access"
  # ...
}
```

**Helm/ArgoCD로 관리해야 할 것 (애플리케이션 레이어):**

- **애플리케이션 배포:** Deployment, Service, Ingress
- **애플리케이션 설정:** ConfigMap, Secret (외부 시크릿 매니저와 연동)
- **인그레스 컨트롤러:** nginx-ingress, ALB Ingress Controller
- **모니터링 스택:** Prometheus, Grafana, Loki
- **서비스 메시:** Istio, Linkerd
- **GitOps 워크플로우:** ArgoCD Application 정의

```yaml
# ArgoCD: 애플리케이션 매니페스트
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/company/k8s-manifests
    path: apps/myapp
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**경계 설정의 이유:**

1. **배포 주기 차이:** 인프라는 드물게, 앱은 하루 수십 회 배포. Terraform의 plan-apply 사이클은 빈번한 앱 배포에 부적합하다.

2. **상태 관리 방식:** Kubernetes는 자체 선언적 상태 관리(Reconciliation Loop)가 있다. Terraform이 K8s 리소스를 관리하면 두 시스템이 상태를 놓고 충돌할 수 있다.

3. **롤백 속도:** ArgoCD/Helm은 Git revert로 즉시 롤백 가능하지만, Terraform은 plan → apply 과정이 필요하다.

4. **팀 책임 분리:** 플랫폼 팀(Terraform) ↔ 개발 팀(Helm/ArgoCD)의 명확한 경계.

**예외:** Helm Provider를 통해 클러스터 부트스트랩 시 필수 컴포넌트(cert-manager, external-dns 등)를 Terraform으로 설치하는 것은 합리적이다.

```hcl
# 클러스터 부트스트랩용 Helm 릴리스 (Terraform 관리)
resource "helm_release" "cert_manager" {
  name       = "cert-manager"
  repository = "https://charts.jetstack.io"
  chart      = "cert-manager"
  namespace  = "cert-manager"
  version    = "v1.13.0"

  set {
    name  = "installCRDs"
    value = "true"
  }

  depends_on = [module.eks]
}
```

**핵심 원칙:** "Terraform은 플랫폼을 만들고, ArgoCD/Helm은 플랫폼 위에서 앱을 운영한다."
</details>

---

## 부록: 자주 사용하는 명령어 치트시트

```bash
# 초기화 및 기본 워크플로우
terraform init                          # Provider/모듈 다운로드, 백엔드 초기화
terraform init -upgrade                 # Provider/모듈을 최신 허용 버전으로 업그레이드
terraform init -migrate-state           # 백엔드 변경 시 State 마이그레이션
terraform validate                      # 구문 유효성 검사
terraform fmt -recursive                # 코드 포맷팅
terraform plan                          # 변경 사항 미리보기
terraform plan -out=tfplan              # 계획을 파일로 저장
terraform apply tfplan                  # 저장된 계획 적용
terraform apply -auto-approve           # 확인 없이 적용 (CI/CD용)
terraform destroy                       # 모든 리소스 삭제

# State 관리
terraform state list                    # State 내 리소스 목록
terraform state show <resource>         # 특정 리소스 상세 정보
terraform state mv <from> <to>          # 리소스 이동/이름 변경
terraform state rm <resource>           # State에서 제거 (리소스 유지)
terraform state pull                    # 원격 State 로컬로 출력
terraform state push                    # 로컬 State를 원격으로 업로드

# Import
terraform import <resource> <id>        # 기존 리소스 가져오기
terraform plan -generate-config-out=x.tf # import 블록 기반 코드 생성

# 디버깅
terraform console                       # 대화형 표현식 평가
terraform graph | dot -Tpng > graph.png # 의존성 그래프 시각화
terraform output                        # 출력값 확인
terraform output -json                  # JSON 형태로 출력
TF_LOG=DEBUG terraform plan             # 상세 로그 활성화

# Workspace
terraform workspace list                # 워크스페이스 목록
terraform workspace new <name>          # 새 워크스페이스 생성
terraform workspace select <name>       # 워크스페이스 전환

# 고급
terraform apply -target=<resource>      # 특정 리소스만 적용
terraform apply -replace=<resource>     # 리소스 강제 재생성
terraform apply -refresh=false          # refresh 건너뛰기
terraform apply -parallelism=30         # 병렬 처리 수 변경
terraform force-unlock <lock_id>        # State 강제 잠금 해제
terraform providers                     # 사용 중인 Provider 목록
terraform test                          # 테스트 실행 (1.6+)
```
