# Terraform - 실습 파일 모음

Terraform 실습 파일은 별도 저장소 [KSNAM97/terraform_practice](https://github.com/KSNAM97/terraform_practice)에서 관리한다. 모든 `.tf` 파일에는 학습용 주석이 그대로 남아 있고, 각 폴더의 `README.md`에 개요가 있다. `terraform.tfstate`는 저장소에 포함하지 않는다.

| 폴더 | 실습 | 핵심 리소스 |
|---|---|---|
| `00-setup` | 환경 설정 | `환경설정.txt`, `terraform-plugin-cache.txt`, `user_daa.txt` |
| `01-vpc/1-1_VPC_data` | VPC 조회 (data 블록) | `data.aws_vpc`, `data.aws_subnets`, `data.aws_subnet`, `data.aws_route_tables`, `data.aws_security_groups` |
| `01-vpc/1-2_VPC_basic` | VPC 기본 구성 | `aws_vpc`, `aws_subnet`, `aws_internet_gateway`, `aws_route_table(_association)`, `aws_security_group`, `aws_instance` |
| `01-vpc/1-3_VPC_total` | VPC 전체 구성 (Public/Private + NAT) | `aws_nat_gateway`, `aws_eip`, `aws_key_pair`, `aws_security_group`(public/private), `aws_instance` |
| `01-vpc/1-4_vpc-count` | count로 반복 생성 | `count`, `locals`, `random_string`, `data.local_file`, `aws_subnet`, `aws_instance` |
| `01-vpc/1-5_vpc_modue` | 모듈로 분리 | `module "network"`, `module "ec2"`, `modules/network`, `modules/ec2` |
| `02-s3/2) S3/2-1_S3_basic` | S3 기본 | `aws_s3_bucket`, `aws_s3_object`, `aws_s3_bucket_public_access_block`, `aws_s3_bucket_policy` |
| `02-s3/2) S3/2-2_S3_versioing` | S3 버저닝 | `aws_s3_bucket`, `aws_s3_bucket_versioning`, `aws_s3_object` |
| `02-s3/2) S3/2-3_S3_static_hosting` | S3 정적 웹 호스팅 + Route 53 | `aws_s3_bucket_website_configuration`, `aws_s3_object`(index/error), `data.aws_route53_zone`, `aws_route53_record` |
| `03-alb-asg` | Packer로 AMI 생성 | `al2023-httpd-ami.pkr.hcl`, `al2023-httpd-ami.json` |
| `04-ec2-basic` | EC2 기본 (최소 구성) | `data.aws_ami`, `aws_instance` |
| `05-iam` | IAM 정책 JSON | `s3-readonly-policy.json` |

## 환경 설정

**개요**: AWS CLI 프로파일, Provider 플러그인 캐시, user_data 예시를 모은 사전 준비 파일이다.

**폴더**: `00-setup`

- AWS CLI 프로파일(`my-profile`) 생성
- `TF_PLUGIN_CACHE_DIR`로 Provider 재다운로드 방지
- EC2 user_data(Nginx 설치) 예시

**핵심 리소스**: `환경설정.txt`, `terraform-plugin-cache.txt`, `user_daa.txt`

## VPC 조회 (data 블록)

**개요**: 새 리소스를 만들지 않고 기본 VPC의 서브넷·라우팅 테이블·보안 그룹을 data 블록으로 조회해 output으로 확인한다.

**폴더**: `01-vpc/1-1_VPC_data`

- `data` 블록은 "조회" 전용임을 이해
- 조회 결과를 `output`으로 출력

**핵심 리소스**: `data.aws_vpc`, `data.aws_subnets`, `data.aws_subnet`, `data.aws_route_tables`, `data.aws_security_groups`

## VPC 기본 구성

**개요**: VPC 1개와 Public Subnet, Internet Gateway, Route Table, 보안 그룹(SSH), EC2를 변수로 구성하는 가장 단순한 VPC 실습이다.

**폴더**: `01-vpc/1-2_VPC_basic`

- 변수(`variables.tf`)로 CIDR·AZ·인스턴스 타입 분리
- 리소스 간 참조로 생성 순서 자동 결정
- `graph.dot`/`graph.png`로 의존 관계 확인

**핵심 리소스**: `aws_vpc`, `aws_subnet`, `aws_internet_gateway`, `aws_route_table(_association)`, `aws_security_group`, `aws_instance`

## VPC 전체 구성 (Public/Private + NAT)

**개요**: Public 2개·Private 2개 서브넷, NAT Gateway, 라우팅 테이블, Key Pair, Public/Private EC2까지 3-Tier 네트워크를 한 번에 구성한다.

**폴더**: `01-vpc/1-3_VPC_total`

- Public/Private 서브넷 분리
- NAT Gateway를 통한 Private 서브넷 외부 통신
- SSH 키페어와 user_data 적용

**핵심 리소스**: `aws_nat_gateway`, `aws_eip`, `aws_key_pair`, `aws_security_group`(public/private), `aws_instance`

## count로 반복 생성

**개요**: 1-3 구성을 `count`와 `local`로 줄여 서브넷 4개를 반복 생성한다. 섹션별 상세 주석이 달려 있다.

**폴더**: `01-vpc/1-4_vpc-count`

- `count`/`count.index`로 중복 코드 제거
- `random_string`, `local_file`(공개키 읽기) 활용
- `environment` 변수로 환경 구분

**핵심 리소스**: `count`, `locals`, `random_string`, `data.local_file`, `aws_subnet`, `aws_instance`

## 모듈로 분리

**개요**: network 모듈(VPC·서브넷·NAT)과 ec2 모듈(키페어·보안 그룹·EC2)로 나누고 root에서 값을 전달·연결한다.

**폴더**: `01-vpc/1-5_vpc_modue`

- root → child 모듈 변수 전달
- 모듈 output을 다른 모듈 input으로 연결
- `terraform.tfvars`로 환경 값 관리

**핵심 리소스**: `module "network"`, `module "ec2"`, `modules/network`, `modules/ec2`

## S3 기본

**개요**: 버킷을 만들고 index.html을 업로드한 뒤 Public Access Block 해제와 버킷 정책으로 공개 읽기를 허용한다.

**폴더**: `02-s3/2) S3/2-1_S3_basic`

- 계정 ID를 이용한 고유 버킷 이름 생성
- `depends_on`으로 Public Access Block → 정책 순서 보장
- index URL output 확인

**핵심 리소스**: `aws_s3_bucket`, `aws_s3_object`, `aws_s3_bucket_public_access_block`, `aws_s3_bucket_policy`

## S3 버저닝

**개요**: 버킷 버저닝을 활성화하고 동일 키의 Object 내용을 바꿔 버전이 쌓이는 것을 확인한다.

**폴더**: `02-s3/2) S3/2-2_S3_versioing`

- `aws_s3_bucket_versioning` 활성화
- 변수 `object_content` 변경 후 재적용으로 버전 생성

**핵심 리소스**: `aws_s3_bucket`, `aws_s3_bucket_versioning`, `aws_s3_object`

## S3 정적 웹 호스팅 + Route 53

**개요**: 정적 웹 사이트 호스팅을 설정하고 Route 53 Alias 레코드로 도메인에 연결한다.

**폴더**: `02-s3/2) S3/2-3_S3_static_hosting`

- `filemd5()`로 파일 변경 감지
- 웹사이트 엔드포인트에 Route 53 A(Alias) 레코드 연결
- fqdn·website_url output

**핵심 리소스**: `aws_s3_bucket_website_configuration`, `aws_s3_object`(index/error), `data.aws_route53_zone`, `aws_route53_record`

## Packer로 AMI 생성

**개요**: Terraform으로 ALB·ASG를 구성하기 전에 Packer로 httpd가 설치된 Amazon Linux 2023 AMI를 만든다(JSON/HCL 두 형식).

**폴더**: `03-alb-asg`

- Packer `required_plugins`, 변수, source 블록
- provisioner로 httpd 설치
- JSON 템플릿과 HCL2 템플릿 비교

**핵심 리소스**: `al2023-httpd-ami.pkr.hcl`, `al2023-httpd-ami.json`

## EC2 기본 (최소 구성)

**개요**: AMI를 data로 조회해 EC2 1대를 만드는 최소 main/variables/outputs 구성이다.

**폴더**: `04-ec2-basic`

- `required_providers` 버전 제약
- `data.aws_ami`로 최신 AL2023 조회
- 변수로 리전·프로파일·인스턴스 타입 분리

**핵심 리소스**: `data.aws_ami`, `aws_instance`

## IAM 정책 JSON

**개요**: S3 읽기 전용 권한을 부여하는 IAM 정책 JSON 예시다. Terraform IAM 실습에서 정책 문서로 사용한다.

**폴더**: `05-iam`

- `s3:ListAllMyBuckets` 등 읽기 전용 Statement 구성

**핵심 리소스**: `s3-readonly-policy.json`
