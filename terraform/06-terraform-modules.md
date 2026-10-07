# Terraform - Modules

## 이론

#### Terraform Module

#### Terraform Module이란?

- Terraform Module은 여러 Terraform 설정과 Resource를 하나의 기능 단위로 묶어
재사용할 수 있도록 만든 코드 집합

- 예를 들어 VPC 환경을 직접 구성하려면 다음과 같은 Resource를 각각 작성해야 한다.
  - VPC
  - Subnet
  - Internet Gateway
  - NAT Gateway
  - Route Table

  - 직접 작성하는 경우:

```hcl
resource "aws_vpc" "example" {
  ...
}

resource "aws_subnet" "example" {
  ...
}

resource "aws_internet_gateway" "example" {
  ...
}

resource "aws_nat_gateway" "example" {
  ...
}
```

- 이처럼 관련 Resource가 많아지면 코드가 길어지고 반복되는 설정도 많아진다.
이러한 관련 Resource를 하나의 묶음으로 만든 것이 Module이다.

#### Module을 사용하는 이유

Module을 사용하면 다음과 같은 장점이 있다.
  - 코드 재사용
  - 유지보수 편리
  - 기능별 코드 분리
  - 반복 코드 감소
  - 프로젝트 구조 단순화

- 예를 들어 VPC 관련 설정을 하나의 Module로 만들면
VPC Module
│
├─ VPC
├─ main.tf
├─ variables.tf
├─ outputs.tf
├─ ASG
├─ main.tf
├─ variables.tf
├─ outputs.tf
├─ ALB
├─ main.tf
├─ variables.tf
├─ outputs.tf

- 처럼 VPC 관련 Resource를 하나의 단위로 관리할 수 있다.

#### Module 기본 형식

- Terraform에서 Module은 "module" 블록으로 호출한다.

```hcl
module "vpc" {
  source = "./modules/vpc"
}

   # 구조
module "vpc"
         │
         └─ 현재 Terraform 코드에서 사용할 Module 이름

source
        │
        └─ Module 코드가 있는 위치
```

- module "vpc" 는 사용할 Module의 이름이고, source = "./modules/vpc" 는 Module 코드가 어디에 있는지를 지정한다.

#### Root Module과 Child Module

- Terraform에서는 현재 실행하는 최상위 Terraform 프로젝트를 Root Module이라고 한다.

- terraform-project/
│
├─ main.tf
├─ variables.tf
├─ outputs.tf
└─ terraform.tfvars

- 이 디렉터리가 Root Module이다.
- Root Module에서 다른 Module을 호출할 수 있다.

예

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

- 그러면 "Root Module  -->  VPC Module" 형태가 된다.
- Root Module에서 호출되는 Module을 일반적으로 Child Module이라고 한다.

#### Local Module

- 현재 프로젝트 내부에 직접 만든 Module을 Local Module이라고 할 수 있다.

```hcl
terraform-project/
│
├─ main.tf
│
└─ modules/
    └─ vpc/
            ├─ main.tf
            ├─ variables.tf
            └─ outputs.tf
```

- Root에서 호출

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

- 여기서 source = "./modules/vpc" 는 현재 프로젝트 내부의 modules/vpc 디렉터리에 있는 Module을 사용한는 의미

#### source란?

- source는 Module 코드를 어디에서 가져올 것인지 지정하는 설정이다.

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

- 이 경우 Local Module을 사용한다.

또 다른 예

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
}
```

- 이 경우 Terraform Registry에 공개된 Module을 사용한다.
즉 source는 이 Module의 코드는 어디에 있는가를 Terraform에게 알려주는 역할을 한다.

#### Terraform Registry Module

- Terraform Registry에는 다른 사용자가 미리 만들어 공개한 Module들이 있다.

예:

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
}
```

- 이 코드는 Terraform 자체 기능을 호출하는 것이 아니다.
의미는 Terraform Registry에 공개된 AWS VPC용 Module을 가져와 사용한다.

구조:
사용자가 작성한 Terraform 코드
│
▼

```hcl
module "vpc"
        │
        ▼
source = "terraform-aws-modules/vpc/aws"
        │
        ▼
Terraform Registry
        │
        ▼
VPC Module
        │
        ├─ VPC
        ├─ Subnet
        ├─ Internet Gateway
        ├─ NAT Gateway
        ├─ Route Table
        └─ 기타 Resource
        │
        ▼

AWS에 Resource 생성
```

#### terraform-aws-modules/vpc/aws 의미

- 코드 : source = "terraform-aws-modules/vpc/aws"

각 부분은 다음 의미를 가진다.

```hcl
terraform-aws-modules
           │
           └─ Module을 제공하는 조직 또는 Namespace
vpc
 │
 └─ Module 이름

aws
 │
 └─ 대상 Provider
```

- 즉 terraform-aws-modules에서 제공하는 AWS용 vpc Module을 사용한다.

#### Terraform Registry Module 방식 (VPC + EC2)

  - 구조

```hcl
terraform-vpc-module/
│
├── main.tf
├── variables.tf
├── outputs.tf
│
└── modules/
    │
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    └── ec2/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

- Root Module : terraform-vpc-module/
  - AWS Provider 설정
  - 실제 변수값 관리
  - Network Module 호출
  - EC2 Module 호출
  - Module 간 Output/Input 연결

- Child Module 1 : modules/network/
  - VPC
  - Public Subnet 2개
  - Private Subnet 2개
  - Internet Gateway
  - Public Route Table
  - NAT Gateway
  - Private Route Table

- Child Module 2 : modules/ec2/
  - Amazon Linux 2023 AMI
  - SSH Key Pair
  - Public EC2 Security Group
  - Private EC2 Security Group
  - Public EC2
  - Private EC2
