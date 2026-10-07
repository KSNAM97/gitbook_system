# Terraform - RDS

이 문서는 Terraform으로 RDS(MySQL)와 Aurora를 구성하는 방법과 서브넷 그룹, 보안 그룹, 접속 확인 실습을 정리한다.

## 1. RDS(Relational Database Service) 서비스 배포

AWS에서는 데이터베이스를 직접 서버에 설치하고 관리하지 않아도 되도록 RDS라는 완전 관리형 데이터베이스 서비스를 제공

일반적으로 기업에서 데이터베이스를 운영하려면 서버 설치, 데이터베이스 설치, 패치 관리, 백업 관리, 장애 복구, 성능 모니터링 등을 직접 수행해야 이러한 작업은 많은 시간과 관리 비용이 필요하다.

RDS는 이러한 데이터베이스 운영 작업을 AWS가 대신 관리해주는 서비스다. 사용자는 데이터베이스 인스턴스를 생성하기만 하면 되고, 실제 운영에 필요한 대부분의 관리 작업은 AWS가 자동으로 수행 따라서 개발자나 운영자는 데이터베이스 인프라 관리보다는 애플리케이션 개발과 서비스 운영에 집중할 수 있다.

RDS는 EC2 위에 데이터베이스를 직접 설치하는 방식과 달리, AWS가 제공하는 관리형 서비스이기 때문에 자동 백업, 장애 복구, 패치 관리, 모니터링 등의 기능이 기본적으로 포함되어 있다. 또한 필요에 따라 스토리지 확장, 읽기 전용 복제본 생성, Multi-AZ 배포 등을 통해 고가용성과 확장성을 확보할 수 있다.

## 2. RDS란?

RDS는 AWS에서 제공하는 완전 관리형 관계형 데이터베이스 서비스다. 관계형 데이터베이스는 테이블 형태로 데이터를 저장하고 SQL을 통해 데이터를 조회하는 구조를 가진 데이터베이스를 의미

RDS의 핵심 목적은 데이터베이스 운영을 자동화하는 것이다. 일반적으로 데이터베이스를 직접 운영할 경우 다음과 같은 작업이 필요하다.

* 데이터베이스 서버 설치
* 운영체제 패치 및 보안 업데이트
* 데이터베이스 소프트웨어 설치 및 업데이트
* 백업 및 복구 관리
* 장애 발생 시 복구 작업
* 성능 모니터링 및 튜닝

이러한 작업은 전문적인 DBA(Database Administrator)가 수행하는 경우가 많다. 하지만 RDS를 사용하면 AWS가 이러한 작업을 자동으로 처리

예를 들어 RDS에서는 다음과 같은 작업이 자동으로 수행된다.

* 정기적인 자동 백업
* 장애 발생 시 자동 복구
* 데이터베이스 소프트웨어 패치
* 스토리지 자동 확장
* 모니터링 및 알림

이 때문에 클라우드 환경에서는 대부분의 서비스가 EC2에 데이터베이스를 직접 설치하기보다 RDS를 사용하는 경우가 많다.

## 3. RDS의 주요 기능

## 4. 자동 백업 및 복구

* RDS는 자동 백업 기능을 제공 사용자가 백업 보존 기간을 설정하면 해당 기간 동안 데이터베이스의 스냅샷과 트랜잭션 로그가 자동으로 저장된다.
* 이 기능을 사용하면 특정 시점으로 데이터베이스를 복원할 수 있다. 이를 Point-in-Time Recovery라고
* 예를 들어 다음과 같은 상황을 가정할 수 있다.
  * 오전 10시에 데이터가 정상 상태
  * 오후 2시에 실수로 데이터 삭제 발생
  * 이 경우 백업 기능을 통해 오전 10시 상태로 데이터베이스를 복구할 수 있다.
  * 이러한 자동 백업 기능은 운영 환경에서 매우 중요한 기능이며, 데이터 손실 위험을 크게 줄여준다.

## 5. 고가용성 및 확장성

RDS는 Multi-AZ 배포 기능을 제공 Multi-AZ는 데이터베이스를 하나의 가용 영역(Availability Zone)이 아니라 여러 가용 영역에 복제하는 구조다.

* 예를 들어 다음과 같은 구조가 된다.
  * Primary DB (AZ-A)
  * Standby DB (AZ-B)
* Primary DB에 장애가 발생하면 AWS가 자동으로 Standby DB를 Primary로 전환 이 과정을 Failover라고
* 이 방식은 다음과 같은 장점이 있다.
  * 장애 발생 시 서비스 중단 최소화
  * 데이터 손실 방지
  * 자동 장애 복구

또한 읽기 전용 복제본(Read Replica)을 생성하여 읽기 트래픽을 분산할 수도 있다. 대규모 서비스에서는 읽기 요청이 많기 때문에 Read Replica를 활용하면 성능을 크게 향상시킬 수 있다.

## 6. 보안 및 관리 기능

* RDS는 다양한 보안 기능을 제공
* 대표적인 보안 기능은 다음과 같다.
  * IAM 연동: AWS IAM과 연동하여 사용자 접근 권한을 제어할 수 있다.
  * VPC 네트워크 격리: RDS 인스턴스는 VPC 내부에 생성되므로 외부 접근을 제한할 수 있다.
  * KMS 암호화: 데이터를 저장할 때 AWS KMS를 이용해 암호화할 수 있다.
  * CloudWatch 모니터링: CPU, 메모리, 디스크 I/O 등의 성능 지표를 모니터링할 수 있다.
  * 이러한 기능을 통해 데이터베이스 보안과 운영 관리를 효율적으로 수행할 수 있다.

## 7. RDS의 지원 데이터베이스 엔진

* RDS는 다양한 데이터베이스 엔진을 지원 사용자는 서비스 요구사항에 따라 적절한 데이터베이스를 선택할 수 있다.

대표적인 데이터베이스 엔진은 다음과 같다.

* Amazon Aurora
* MySQL
* PostgreSQL
* MariaDB
* Oracle
* Microsoft SQL Server
* 이 중에서 클라우드 환경에서 가장 많이 사용되는 엔진은 Aurora와 MySQL이다.

## 8. Amazon Aurora

* Amazon Aurora는 AWS에서 개발한 고성능 클라우드 데이터베이스다. MySQL과 PostgreSQL과 호환되는 구조를 가지고 있다.
* Aurora의 가장 큰 특징은 성능과 확장성이다.
* AWS 공식 자료 기준으로 다음과 같은 성능을 제공
  * MySQL 대비 최대 5배 성능
  * PostgreSQL 대비 최대 3배 성능

Aurora는 기존 데이터베이스와 달리 스토리지 구조가 분리되어 있다. 데이터는 3개의 가용 영역에 총 6개의 복제본으로 저장된다.

즉 다음과 같은 구조를 가진다.

* AZ-A : 2개 복제
* AZ-B : 2개 복제
* AZ-C : 2개 복제
* 이 구조는 다음과 같은 장점을 제공
  * 높은 데이터 안정성
  * 빠른 장애 복구
  * 자동 스토리지 확장
* Aurora는 대규모 트래픽을 처리하는 서비스에서 많이 사용된다.

## 9. RDS MySQL

* RDS MySQL은 AWS에서 제공하는 관리형 MySQL 서비스다. 기존 MySQL 데이터베이스와 동일한 방식으로 사용할 수 있다.
* 이미 MySQL을 사용하고 있는 서비스라면 RDS MySQL로 쉽게 마이그레이션할 수 있다.
* RDS MySQL의 주요 특징은 다음과 같다.
  * 자동 백업
  * 자동 패치
  * 모니터링 기능
  * 스냅샷 기능
  * 스토리지 확장
  * 즉 기존 MySQL 서버를 직접 운영하는 것보다 훨씬 관리가 편리하다.
* 대부분의 중소 규모 서비스에서는 RDS MySQL을 많이 사용하며, 대규모 트래픽 서비스에서는 Aurora를 선택하는 경우가 많다.

## 10. Terraform을 활용한 RDS 배포

Terraform은 인프라를 코드로 관리하기 때문에 반복 가능한 배포와 관리가 가능하다. 즉, AWS 콘솔에서 하나씩 클릭해서 만드는 방식이 아니라, Terraform 코드 파일에 원하는 인프라 구성을 작성한 뒤 명령어를 실행하여 동일한 환경을 다시 만들 수 있다.

* 장점
  * 같은 인프라를 여러 번 동일하게 재현할 수 있다.
  * 누가 어떤 설정을 했는지 코드로 남길 수 있다.
  * 실습, 테스트, 운영 환경을 같은 방식으로 관리할 수 있다.
  * 잘못된 수동 설정을 줄일 수 있다.
  * 삭제(terraform destroy)까지 자동화할 수 있다.
* Terraform은 단순히 서버 한 대를 만드는 도구가 아니라, VPC, Subnet, Security Group, EC2, RDS 같은 AWS 자원 전체를 코드로 연결해 관리하는 도구

## 11. Terraform을 통한 RDS 구성 이해

* 아래 코드는 MySQL 기반 RDS를 생성하는 가장 기본적인 예시이다.

```hcl
resource "aws_db_instance" "my_rds_instance" {
  allocated_storage        = 20                # 인스턴스에 할당할 스토리지 (GB 단위)
  engine                  = "mysql"           # 데이터베이스 엔진 타입 (MySQL)
  engine_version          = "8.0"             # 데이터베이스 엔진 버전 (MySQL 8.0)
  instance_class           = "db.t3.micro"     # 인스턴스 유형 (소형 인스턴스)
  name                   = "mydatabase"      # 데이터베이스 이름
  username               = "admin"           # 데이터베이스 관리자 계정 이름
  password               = "password"        # 데이터베이스 관리자 계정 비밀번호 (보안 유의)
  parameter_group_name = "default.mysql8.0"# 사용할 파라미터 그룹 이름 (MySQL 8.0 기본)
  skip_final_snapshot     = true              # 삭제 시 최종 스냅샷을 생성하지 않음
  publicly_accessible      = false             # 퍼블릭 액세스 허용하지 않음
  multi_az                = true              # 다중 AZ 배포 활성화 (장애 대비)

  tags = {
    Name        = "My-RDS-MySQL"           # 이름 태그
    Environment = "Production"             # 환경 태그
  }
}
```

* 이 코드는 AWS에 MySQL RDS 인스턴스를 하나 생성
* 여기서 중요한 것은 단순히 DB만 만드는 것이 아니라, 운영에 필요한 주요 속성을 함께 정의하고 있다.

## 12. RDS 주요 설정 상세 설명

* allocated\_storage = 20
  * 이 설정은 RDS 인스턴스에 할당할 스토리지 크기를 의미
  * 단위는 GB
  * 즉, 위 설정은 DB 저장 공간을 20GB로 만들겠다는 의미이다.
* engine = "mysql"
  * 이 값은 RDS에서 사용할 데이터베이스 엔진 종류를 의미
  * 여기서는 MySQL을 사용
  * AWS RDS는 여러 DB 엔진을 지원
* mysql
* postgres
* mariadb
* oracle-se2
* sqlserver-ex
* aurora-mysql
* aurora-postgresql
* engine\_version = "8.0"
  * 이 값은 사용할 MySQL 버전을 지정
  * MySQL은 버전에 따라 다음과 같은 차이가 있을 수 있다.
* instance\_class = "db.t3.micro"
  * 이 값은 RDS 인스턴스의 사양을 의미
  * RDS 서버의 CPU, 메모리 수준을 정하는 옵션
  * db.t3.micro : 소규모 테스트용
  * db.t3.small : 조금 더 여유 있는 환경
  * db.t3.medium : 더 높은 리소스
  * db.m 계열, db.r 계열 : 운영/고성능 환경
* name = "mydatabase"
  * 이 설정은 RDS 인스턴스 안에 생성될 기본 데이터베이스 이름
  * 즉, RDS 서버 자체 이름이 아니라, MySQL 내부에 생성될 DB 이름
  * 예를 들어 접속 후 show databases;를 실행했을 때 mydatabase가 보이게 된다.
* username / password
  * username = "admin" (관리자 계정 이름)
  * password = "password"(관리자 비밀번호)
  * mysql -h RDS엔드포인트 -u admin -p
* 운영 환경에서는 비밀번호를 코드에 직접 적는 것은 보안상 좋지 않다.
  * terraform variable로 분리
  * tfvars 파일 사용
* parameter\_group\_name = "default.mysql8.0"
  * 이 설정은 DB 파라미터 그룹을 지정
  * 파라미터 그룹은 MySQL의 동작 방식을 제어하는 설정 묶음입니다.
* skip\_final\_snapshot = true
  * 이 값은 RDS를 삭제할 때 최종 스냅샷을 남길지 여부입니다.
  * true : 삭제 시 최종 스냅샷 생성 안 함
  * false : 삭제 전에 마지막 스냅샷 생성
  * 실습에서는 비용과 시간을 줄이기 위해 true를 많이 사용
  * 하지만 운영에서는 데이터 보호를 위해 false로 두고 최종 백업을 남기는 경우가 많다.
* publicly\_accessible = false
  * 이 설정은 RDS를 퍼블릭 인터넷에서 직접 접근 가능하게 만들지 여부
  * true : 퍼블릭 접근 가능
  * false : 퍼블릭 접근 불가
  * 여기서는 false이므로 외부 인터넷에서 바로 RDS로 붙을 수 없다.
* multi\_az = true
  * 이 설정은 고가용성을 위해 Multi-AZ 구성을 활성화하는 값입니다.
* Multi-AZ가 활성화되면 다음처럼 동작
  * 하나의 AZ에 Primary DB 배치
  * 다른 AZ에 Standby DB 자동 생성
  * 장애 발생 시 Standby로 Failover
  * 즉, 읽기 분산용 복제본이 아니라 장애 대비용 대기 인스턴스

## 13. VPC와 서브넷 그룹

## 14. VPC

* RDS 인스턴스는 AWS VPC 안에서 생성된다.
* 서브넷 그룹을 설정하여 데이터베이스가 위치할 네트워크 영역을 지정할 수 있다.
* 즉, RDS는 아무 네트워크에나 생성되는 것이 아니라, 반드시 특정 VPC와 서브넷 안에 배치된다.
  * 퍼블릭 서브넷: 인터넷 게이트웨이와 연결된 영역
  * 프라이빗 서브넷 : 외부 인터넷에서 직접 접근 불가한 영역
  * DB는 보통 프라이빗 서브넷에 생성
* 또한 RDS는 단일 서브넷이 아니라 DB Subnet Group이라는 형태로 서브넷 목록을 지정해야 합니다.
* 특히 Multi-AZ를 사용할 경우 서로 다른 AZ의 프라이빗 서브넷이 필요합니다.
* 이를 통해 외부 접근을 제어하고 보안을 강화할 수 있다.

## 15. 보안 그룹(Security Groups)

* RDS 인스턴스에 적용되는 보안 그룹은 인바운드 및 아웃바운드 트래픽을 제어합니다.
* 보안 그룹을 통해 특정 IP 또는 네트워크만 데이터베이스에 접근할 수 있도록 설정합니다.

```hcl
resource "aws_security_group" "rds_sg" {
  name        = "rds-security-group"   # 보안 그룹 이름
  description = "Allow database access" # 보안 그룹 설명
  vpc_id      = var.vpc_id             # VPC ID 변수

  ingress {
    from_port   = 3306                 # 시작 포트 (MySQL 기본 포트)
    to_port     = 3306                 # 종료 포트 (MySQL 기본 포트)
    protocol    = "tcp"                # 프로토콜 (TCP)
    cidr_blocks = ["10.0.0.0/16"]      # 내부 VPC 네트워크에만 접근 허용
  }

  egress {
    from_port   = 0               # 모든 포트 허용 (출력 트래픽)
    to_port     = 0                 # 모든 포트 허용 (출력 트래픽)
    protocol    = "-1"              # 모든 프로토콜 허용
    cidr_blocks = ["0.0.0.0/0"]     # 모든 IP에 출력 허용
  }
}
```

## 16. Terraform을 활용한 RDS 인스턴스 배포 및 검증

* 이 코드는 VPC와 EC2 인스턴스, 그리고 RDS용 보안 그룹 및 서브넷 그룹을 설정
* vpc 모듈을 통해 VPC, 서브넷 및 가용 영역을 구성하고, ec2 모듈로 EC2 인스턴스를 퍼블릭 서브넷에 배포
* 마지막으로, aws\_security\_group과 aws\_db\_subnet\_group 리소스를 통해 데이터베이스 접근을 제어하고, 프라이빗 서브넷에 RDS를 위한 서브넷 그룹을 생성

## 17. 전체 흐름

* VPC 생성
  * Public / Private Subnet 생성
  * EC2를 Public Subnet에 배치
  * RDS는 Private Subnet에 배치
  * Security Group으로 MySQL 포트 제어
  * DB Subnet Group 생성
  * 그 위에 RDS 생성

VPC 모듈

```hcl
module "vpc" {
  source             = "./modules/vpc"     # VPC 모듈 경로
  vpc_name           = var.vpc_name        # VPC 이름
  vpc_cidr           = var.vpc_cidr        # VPC CIDR 블록
  public_subnets     = var.public_subnets  # 퍼블릭 서브넷 리스트
  private_subnets    = var.private_subnets # 프라이빗 서브넷 리스트
  availability_zones = var.availability_zones# 가용 영역 리스트
}
```

* source = "./modules/vpc"
  * 이 모듈 코드가 위치한 경로입니다.
  * 즉, VPC를 직접 여기서 작성하지 않고 별도의 모듈 파일에서 불러온다.
* vpc\_name = var.vpc\_name
  * 생성할 VPC 이름
  * 태그나 리소스 명명에 사용될 수 있다.

vpc\_cidr

* vpc\_cidr = var.vpc\_cidr
  * VPC 전체 네트워크 대역입니다.
  * 예를 들어 10.0.0.0/16 같은 값이 들어갈 수 있다.
* public\_subnets = var.public\_subnets
  * 인터넷 접근이 가능한 퍼블릭 서브넷 목록입니다.
  * 보통 ALB, Bastion Host, 외부 공개 EC2 등이 이곳에 배치
* private\_subnets
  * private\_subnets = var.private\_subnets
  * 인터넷에서 직접 접근할 수 없는 프라이빗 서브넷 목록
  * RDS는 보통 이곳에 배치
* availability\_zones = var.availability\_zones
  * 서브넷을 어느 AZ에 배치할지 지정합니다.
  * Multi-AZ 구성이나 고가용성 구조를 위해 여러 AZ를 사용

## 18. EC2 모듈

```hcl
module "ec2" {
  source = "./modules/ec2" # EC2 모듈 경로

  ami_id          = var.ami_id                 # AMI ID
  instance_type   = var.instance_type          # EC2 인스턴스 타입
  vpc_id          = module.vpc.vpc_id          # VPC ID
  subnet_id       = module.vpc.public_subnets[0] # 퍼블릭 서브넷 ID (첫 번째 서브넷)
  instance_name   = var.instance_name          # 인스턴스 이름
  public_key_path = var.public_key_path        # 공개 키 경로
}
```

* source
  * EC2를 생성하는 모듈 경로입니다.
* ami\_id
  * EC2에 사용할 운영체제 이미지 ID
  * 예를 들면 Amazon Linux, Rocky Linux, Ubuntu 등
* instance\_type
  * EC2 사양입니다.
  * 예: t3.micro, t3.small
* vpc\_id
  * EC2를 어느 VPC에 넣을지 지정합니다.

## 19. RDS 보안 그룹

```hcl
resource "aws_security_group" "rds_sg" {
  name        = "rds-security-group"   # 보안 그룹 이름
  description = "Allow database access" # 보안 그룹 설명
  vpc_id      = module.vpc.vpc_id      # VPC ID

  ingress {
    from_port   = 3306         # 시작 포트 (MySQL 기본 포트)
    to_port     = 3306                 # 종료 포트 (MySQL 기본 포트)
    protocol    = "tcp"                # 프로토콜 (TCP)
    cidr_blocks = [var.allowed_cidr]   # 접근 허용 CIDR
  }

  egress {
    from_port   = 0                # 모든 포트 허용 (출력 트래픽)
    to_port     = 0                # 모든 포트 허용 (출력 트래픽)
    protocol    = "-1"              # 모든 프로토콜 허용
    cidr_blocks = ["0.0.0.0/0"] # 모든 IP에 출력 허용
  }
}

# DB 서브넷 그룹 생성
resource "aws_db_subnet_group" "this" {
  name       = "${var.vpc_name}-db-subnet-group" # DB 서브넷 그룹 이름
  subnet_ids = module.vpc.private_subnets        # 프라이빗 서브넷 ID 목록

  tags = {
    Name = "${var.vpc_name}-db-subnet-group"     # 태그 이름 설정
  }
}
```

* RDS는 일반 EC2처럼 그냥 서브넷 하나 지정해서 넣는 것이 아니라, DB Subnet Group을 통해 여러 서브넷을 묶어서 지정
* 여러개를 지정하는 이유는 Multi-AZ를 위해 다른 AZ의 서브넷이 필요할 수 있고 장애 대비를 위해 최소 2개 이상의 서브넷을 구성하는 경우가 많다. 여기서는 module.vpc.private\_subnets를 사용하므로 RDS는 프라이빗 서브넷에 배치 즉, DB를 외부 인터넷에 공개하지 않겠다는 구조입니다.

## 20. RDS 인스턴스 구성

이 코드는 MySQL 기반 AWS RDS 인스턴스를 생성하고 구성하는 Terraform 코드입니다. aws\_db\_instance 리소스를 통해 인스턴스 스토리지 크기, 데이터베이스 이름, 파라미터 그룹, 보안 그룹 등 다양한 설정을 정의합니다. 또한 백업 및 유지보수 시간, 스토리지 유형 등을 지정하여 인스턴스 운영을 최적화하며, 퍼블릭 액세스를 비활성화하여 보안을 강화합니다.

```hcl
resource "aws_db_instance" "my_rds_instance" {
  allocated_storage     = var.db_allocated_storage      # RDS 인스턴스의 스토리지 크기
  engine                = "mysql"                       # 데이터베이스 엔진 (MySQL)
  engine_version        = var.db_engine_version         # 데이터베이스 엔진 버전
  instance_class        = var.db_instance_class         # 인스턴스 유형
  db_name               = var.db_name                   # 데이터베이스 이름
  username              = var.db_username               # 관리자 계정 이름
  password              = var.db_password               # 관리자 계정 비밀번호
  parameter_group_name  = var.db_parameter_group_name   # 데이터베이스 파라미터 그룹
  skip_final_snapshot   = true                          # 삭제 시 최종 스냅샷 생성하지 않음
  publicly_accessible   = false                         # 퍼블릭 액세스 비활성화
  multi_az              = var.db_multi_az              # 다중 가용 영역 배포 여부
  vpc_security_group_ids = [aws_security_group.rds_sg.id] # 사용할 보안 그룹 ID
  db_subnet_group_name  = aws_db_subnet_group.my_db_subnet_group.name # DB 서브넷 그룹 이름

  # 백업 관련 설정
  backup_retention_period = 7                   # 백업 보존 기간 (일 단위)
  backup_window           = "02:00-03:00"        # 백업 시작 시간 (UTC 기준)

  # 모니터링 및 유지관리
  maintenance_window      = "sun:05:00-sun:06:00" # 유지보수 시간 (UTC 기준)

  # 스토리지 및 암호화 설정
  storage_type            = "gp2"                      # 스토리지 유형 (gp2: 범용 SSD)

  tags = {
    Name        = "My-RDS-MySQL"                       # RDS 인스턴스 이름 태그
    Environment = var.environment                      # 환경 태그
  }
}
```

* allocated\_storage
  * allocated\_storage = var.db\_allocated\_storage
  * RDS 스토리지 크기
  * 직접 숫자를 넣지 않고 변수로 받아서 처리
  * 즉, 개발/실습/운영 환경마다 크기를 다르게 줄 수 있다.
* engine
  * engine = "mysql"
  * MySQL RDS를 사용
* engine\_version
  * engine\_version = var.db\_engine\_version
  * MySQL 버전을 변수로 분리
  * 예를 들면 8.0.35 같은 식으로 넣을 수 있다.
* instance\_class
  * instance\_class = var.db\_instance\_class
  * RDS 서버 사양
  * 예: db.t3.micro, db.t3.small
  * 이 역시 변수화되어 있기 때문에 환경마다 쉽게 바꿀 수 있다.
* db\_name
  * db\_name = var.db\_name
  * RDS 안에 생성될 기본 데이터베이스 이름
* username / password
  * username = var.db\_username
  * password = var.db\_password
  * DB 접속 계정 정보입니다.
  * 실습에서는 변수에 넣어 관리
* parameter\_group\_name
  * parameter\_group\_name = var.db\_parameter\_group\_name
  * DB 동작 세부 설정을 담은 파라미터 그룹입니다.
  * 예를 들어 문자셋, 로그 설정, 타임존, 연결 수 제한 등의 DB 동작을 제어
* skip\_final\_snapshot
  * skip\_final\_snapshot = true
  * 삭제 시 최종 백업 스냅샷을 남기지 않습니다.
  * 실습에서는 빠른 삭제를 위해 true를 많이 사용
* publicly\_accessible
  * publicly\_accessible = false
  * 외부 인터넷 직접 접근 불가 즉, EC2를 통해 내부에서 접근하는 구조입니다.
* multi\_az
  * multi\_az = var.db\_multi\_az
  * 다중 AZ 활성화 여부를 변수로 제어합니다.
  * true: 장애 대비용 스탠바이 인스턴스 생성
  * false: 단일 AZ
* vpc\_security\_group\_ids
  * vpc\_security\_group\_ids = \[aws\_security\_group.rds\_sg.id]
  * RDS에 어떤 보안 그룹을 적용할지 지정
  * 여기서는 위에서 만든 rds\_sg를 연결
  * 즉, 이 보안 그룹 규칙이 RDS 접근 정책이 된다.
* db\_subnet\_group\_name
  * db\_subnet\_group\_name = aws\_db\_subnet\_group.my\_db\_subnet\_group.name
  * RDS가 배치될 DB 서브넷 그룹을 지정
  * 즉, 이 설정 때문에 RDS는 프라이빗 서브넷들에 배치
* 백업 관련 설정
  * backup\_retention\_period = 7(자동 백업을 며칠 동안 보관할지 지정, 7일)
  * backup\_window = "02:00-03:00"(자동 백업이 수행될 시간, UTC 기준 새벽 2시부터 3시 사이)

## 21. 읽기 전용 인스턴스 구성

* 다음 코드를 사용해 읽기 전용 인스턴스를 구성
* RDS에서 읽기 복제본(Read Replica)을 생성할 때, "읽기 전용"은 따로 명시할 필요는 없다.
* 읽기 복제본은 기본적으로 비동기적으로 원본 인스턴스의 데이터를 복제하고, 읽기 요청만 처리하는 인스턴스로 설정

## 22. 읽기 복제본 인스턴스

```hcl
resource "aws_db_instance" "read_replica" {
  engine              = "mysql"        # 데이터베이스 엔진 (MySQL)
  instance_class      = "db.t3.micro"  # 인스턴스 유형
  publicly_accessible = false         # 퍼블릭 액세스 비활성화
  skip_final_snapshot = true          # 최종 스냅샷 미생성

  replicate_source_db = aws_db_instance.my_rds_instance.id# 복제할 원본 인스턴스 ID

  tags = {
    Name        = "My-RDS-Read-Replica"     # 복제본 인스턴스 이름 태그
    Environment = "Production"                 # 환경 태그
  }
}
```

* replicate\_source\_db
  * replicate\_source\_db = aws\_db\_instance.my\_rds\_instance.id
  * 어떤 RDS를 원본으로 복제할지 지정
  * 즉, my\_rds\_instance를 원본으로 하여 읽기 전용 복제본을 하나 만들겠다는 의미
* 읽기 복제본의 특징
  * 원본 DB 데이터를 복제
  * 조회 요청 처리 가능
  * 쓰기 작업 불가
  * 읽기 부하 분산 가능
  * 원본과 약간의 복제 지연이 생길 수 있음
* 중요한 점은 Multi-AZ와 Read Replica는 목적이 다르다는 것입니다.
  * Multi-AZ : 장애 대비
  * Read Replica : 읽기 분산

## 23. 실습: RDS(Relational Database Service) 서비스 배포

AWS에서는 데이터베이스를 직접 서버에 설치하고 관리하지 않아도 되도록 RDS라는 완전 관리형 데이터베이스 서비스를 제공

일반적으로 기업에서 데이터베이스를 운영하려면 서버 설치, 데이터베이스 설치, 패치 관리, 백업 관리, 장애 복구, 성능 모니터링 등을 직접 수행해야 이러한 작업은 많은 시간과 관리 비용이 필요하다.

RDS는 이러한 데이터베이스 운영 작업을 AWS가 대신 관리해주는 서비스다. 사용자는 데이터베이스 인스턴스를 생성하기만 하면 되고, 실제 운영에 필요한 대부분의 관리 작업은 AWS가 자동으로 수행 따라서 개발자나 운영자는 데이터베이스 인프라 관리보다는 애플리케이션 개발과 서비스 운영에 집중할 수 있다.

RDS는 EC2 위에 데이터베이스를 직접 설치하는 방식과 달리, AWS가 제공하는 관리형 서비스이기 때문에 자동 백업, 장애 복구, 패치 관리, 모니터링 등의 기능이 기본적으로 포함되어 있다. 또한 필요에 따라 스토리지 확장, 읽기 전용 복제본 생성, Multi-AZ 배포 등을 통해 고가용성과 확장성을 확보할 수 있다.

## 24. 실습: RDS란?

RDS는 AWS에서 제공하는 완전 관리형 관계형 데이터베이스 서비스다. 관계형 데이터베이스는 테이블 형태로 데이터를 저장하고 SQL을 통해 데이터를 조회하는 구조를 가진 데이터베이스를 의미

RDS의 핵심 목적은 데이터베이스 운영을 자동화하는 것이다. 일반적으로 데이터베이스를 직접 운영할 경우 다음과 같은 작업이 필요하다.

* 데이터베이스 서버 설치
* 운영체제 패치 및 보안 업데이트
* 데이터베이스 소프트웨어 설치 및 업데이트
* 백업 및 복구 관리
* 장애 발생 시 복구 작업
* 성능 모니터링 및 튜닝

이러한 작업은 전문적인 DBA(Database Administrator)가 수행하는 경우가 많다. 하지만 RDS를 사용하면 AWS가 이러한 작업을 자동으로 처리

예를 들어 RDS에서는 다음과 같은 작업이 자동으로 수행된다.

* 정기적인 자동 백업
* 장애 발생 시 자동 복구
* 데이터베이스 소프트웨어 패치
* 스토리지 자동 확장
* 모니터링 및 알림

이 때문에 클라우드 환경에서는 대부분의 서비스가 EC2에 데이터베이스를 직접 설치하기보다 RDS를 사용하는 경우가 많다.

## 25. 실습: RDS의 주요 기능

## 26. 실습: 자동 백업 및 복구

* RDS는 자동 백업 기능을 제공 사용자가 백업 보존 기간을 설정하면 해당 기간 동안 데이터베이스의 스냅샷과 트랜잭션 로그가 자동으로 저장된다.
* 이 기능을 사용하면 특정 시점으로 데이터베이스를 복원할 수 있다. 이를 Point-in-Time Recovery라고
* 예를 들어 다음과 같은 상황을 가정할 수 있다.
  * 오전 10시에 데이터가 정상 상태
  * 오후 2시에 실수로 데이터 삭제 발생
  * 이 경우 백업 기능을 통해 오전 10시 상태로 데이터베이스를 복구할 수 있다.
  * 이러한 자동 백업 기능은 운영 환경에서 매우 중요한 기능이며, 데이터 손실 위험을 크게 줄여준다.

## 27. 실습: 고가용성 및 확장성

RDS는 Multi-AZ 배포 기능을 제공 Multi-AZ는 데이터베이스를 하나의 가용 영역(Availability Zone)이 아니라 여러 가용 영역에 복제하는 구조다.

* 예를 들어 다음과 같은 구조가 된다.
  * Primary DB (AZ-A)
  * Standby DB (AZ-B)
* Primary DB에 장애가 발생하면 AWS가 자동으로 Standby DB를 Primary로 전환 이 과정을 Failover라고
* 이 방식은 다음과 같은 장점이 있다.
  * 장애 발생 시 서비스 중단 최소화
  * 데이터 손실 방지
  * 자동 장애 복구

또한 읽기 전용 복제본(Read Replica)을 생성하여 읽기 트래픽을 분산할 수도 있다. 대규모 서비스에서는 읽기 요청이 많기 때문에 Read Replica를 활용하면 성능을 크게 향상시킬 수 있다.

## 28. 실습: 보안 및 관리 기능

* RDS는 다양한 보안 기능을 제공
* 대표적인 보안 기능은 다음과 같다.
  * IAM 연동: AWS IAM과 연동하여 사용자 접근 권한을 제어할 수 있다.
  * VPC 네트워크 격리: RDS 인스턴스는 VPC 내부에 생성되므로 외부 접근을 제한할 수 있다.
  * KMS 암호화: 데이터를 저장할 때 AWS KMS를 이용해 암호화할 수 있다.
  * CloudWatch 모니터링: CPU, 메모리, 디스크 I/O 등의 성능 지표를 모니터링할 수 있다.
  * 이러한 기능을 통해 데이터베이스 보안과 운영 관리를 효율적으로 수행할 수 있다.

## 29. 실습: RDS의 지원 데이터베이스 엔진

* RDS는 다양한 데이터베이스 엔진을 지원 사용자는 서비스 요구사항에 따라 적절한 데이터베이스를 선택할 수 있다.

대표적인 데이터베이스 엔진은 다음과 같다.

* Amazon Aurora
* MySQL
* PostgreSQL
* MariaDB
* Oracle
* Microsoft SQL Server
* 이 중에서 클라우드 환경에서 가장 많이 사용되는 엔진은 Aurora와 MySQL이다.

## 30. 실습: Amazon Aurora

* Amazon Aurora는 AWS에서 개발한 고성능 클라우드 데이터베이스다. MySQL과 PostgreSQL과 호환되는 구조를 가지고 있다.
* Aurora의 가장 큰 특징은 성능과 확장성이다.
* AWS 공식 자료 기준으로 다음과 같은 성능을 제공
  * MySQL 대비 최대 5배 성능
  * PostgreSQL 대비 최대 3배 성능

Aurora는 기존 데이터베이스와 달리 스토리지 구조가 분리되어 있다. 데이터는 3개의 가용 영역에 총 6개의 복제본으로 저장된다.

즉 다음과 같은 구조를 가진다.

* AZ-A : 2개 복제
* AZ-B : 2개 복제
* AZ-C : 2개 복제
* 이 구조는 다음과 같은 장점을 제공
  * 높은 데이터 안정성
  * 빠른 장애 복구
  * 자동 스토리지 확장
* Aurora는 대규모 트래픽을 처리하는 서비스에서 많이 사용된다.

## 31. 실습: RDS MySQL

* RDS MySQL은 AWS에서 제공하는 관리형 MySQL 서비스다. 기존 MySQL 데이터베이스와 동일한 방식으로 사용할 수 있다.
* 이미 MySQL을 사용하고 있는 서비스라면 RDS MySQL로 쉽게 마이그레이션할 수 있다.
* RDS MySQL의 주요 특징은 다음과 같다.
  * 자동 백업
  * 자동 패치
  * 모니터링 기능
  * 스냅샷 기능
  * 스토리지 확장
  * 즉 기존 MySQL 서버를 직접 운영하는 것보다 훨씬 관리가 편리하다.
* 대부분의 중소 규모 서비스에서는 RDS MySQL을 많이 사용하며, 대규모 트래픽 서비스에서는 Aurora를 선택하는 경우가 많다.

## 32. 실습: Terraform을 활용한 RDS 배포

Terraform은 인프라를 코드로 관리하기 때문에 반복 가능한 배포와 관리가 가능하다. 즉, AWS 콘솔에서 하나씩 클릭해서 만드는 방식이 아니라, Terraform 코드 파일에 원하는 인프라 구성을 작성한 뒤 명령어를 실행하여 동일한 환경을 다시 만들 수 있다.

* 장점
  * 같은 인프라를 여러 번 동일하게 재현할 수 있다.
  * 누가 어떤 설정을 했는지 코드로 남길 수 있다.
  * 실습, 테스트, 운영 환경을 같은 방식으로 관리할 수 있다.
  * 잘못된 수동 설정을 줄일 수 있다.
  * 삭제(terraform destroy)까지 자동화할 수 있다.
* Terraform은 단순히 서버 한 대를 만드는 도구가 아니라, VPC, Subnet, Security Group, EC2, RDS 같은 AWS 자원 전체를 코드로 연결해 관리하는 도구

## 33. 실습: Terraform을 통한 RDS 구성 이해

* 아래 코드는 MySQL 기반 RDS를 생성하는 가장 기본적인 예시이다.

```hcl
resource "aws_db_instance" "my_rds_instance" {
  allocated_storage        = 20                # 인스턴스에 할당할 스토리지 (GB 단위)
  engine                  = "mysql"           # 데이터베이스 엔진 타입 (MySQL)
  engine_version          = "8.0"             # 데이터베이스 엔진 버전 (MySQL 8.0)
  instance_class           = "db.t3.micro"     # 인스턴스 유형 (소형 인스턴스)
  name                   = "mydatabase"      # 데이터베이스 이름
  username               = "admin"           # 데이터베이스 관리자 계정 이름
  password               = "password"        # 데이터베이스 관리자 계정 비밀번호 (보안 유의)
  parameter_group_name = "default.mysql8.0"# 사용할 파라미터 그룹 이름 (MySQL 8.0 기본)
  skip_final_snapshot     = true              # 삭제 시 최종 스냅샷을 생성하지 않음
  publicly_accessible      = false             # 퍼블릭 액세스 허용하지 않음
  multi_az                = true              # 다중 AZ 배포 활성화 (장애 대비)

  tags = {
    Name        = "My-RDS-MySQL"           # 이름 태그
    Environment = "Production"             # 환경 태그
  }
}
```

* 이 코드는 AWS에 MySQL RDS 인스턴스를 하나 생성
* 여기서 중요한 것은 단순히 DB만 만드는 것이 아니라, 운영에 필요한 주요 속성을 함께 정의하고 있다.

## 34. 실습: RDS 주요 설정 상세 설명

* allocated\_storage = 20
  * 이 설정은 RDS 인스턴스에 할당할 스토리지 크기를 의미
  * 단위는 GB
  * 즉, 위 설정은 DB 저장 공간을 20GB로 만들겠다는 의미이다.
* engine = "mysql"
  * 이 값은 RDS에서 사용할 데이터베이스 엔진 종류를 의미
  * 여기서는 MySQL을 사용
  * AWS RDS는 여러 DB 엔진을 지원
* mysql
* postgres
* mariadb
* oracle-se2
* sqlserver-ex
* aurora-mysql
* aurora-postgresql
* engine\_version = "8.0"
  * 이 값은 사용할 MySQL 버전을 지정
  * MySQL은 버전에 따라 다음과 같은 차이가 있을 수 있다.
* instance\_class = "db.t3.micro"
  * 이 값은 RDS 인스턴스의 사양을 의미
  * RDS 서버의 CPU, 메모리 수준을 정하는 옵션
  * db.t3.micro : 소규모 테스트용
  * db.t3.small : 조금 더 여유 있는 환경
  * db.t3.medium : 더 높은 리소스
  * db.m 계열, db.r 계열 : 운영/고성능 환경
* name = "mydatabase"
  * 이 설정은 RDS 인스턴스 안에 생성될 기본 데이터베이스 이름
  * 즉, RDS 서버 자체 이름이 아니라, MySQL 내부에 생성될 DB 이름
  * 예를 들어 접속 후 show databases;를 실행했을 때 mydatabase가 보이게 된다.
* username / password
  * username = "admin" (관리자 계정 이름)
  * password = "password"(관리자 비밀번호)
  * mysql -h RDS엔드포인트 -u admin -p
* 운영 환경에서는 비밀번호를 코드에 직접 적는 것은 보안상 좋지 않다.
  * terraform variable로 분리
  * tfvars 파일 사용
* parameter\_group\_name = "default.mysql8.0"
  * 이 설정은 DB 파라미터 그룹을 지정
  * 파라미터 그룹은 MySQL의 동작 방식을 제어하는 설정 묶음입니다.
* skip\_final\_snapshot = true
  * 이 값은 RDS를 삭제할 때 최종 스냅샷을 남길지 여부입니다.
  * true : 삭제 시 최종 스냅샷 생성 안 함
  * false : 삭제 전에 마지막 스냅샷 생성
  * 실습에서는 비용과 시간을 줄이기 위해 true를 많이 사용
  * 하지만 운영에서는 데이터 보호를 위해 false로 두고 최종 백업을 남기는 경우가 많다.
* publicly\_accessible = false
  * 이 설정은 RDS를 퍼블릭 인터넷에서 직접 접근 가능하게 만들지 여부
  * true : 퍼블릭 접근 가능
  * false : 퍼블릭 접근 불가
  * 여기서는 false이므로 외부 인터넷에서 바로 RDS로 붙을 수 없다.
* multi\_az = true
  * 이 설정은 고가용성을 위해 Multi-AZ 구성을 활성화하는 값입니다.
* Multi-AZ가 활성화되면 다음처럼 동작
  * 하나의 AZ에 Primary DB 배치
  * 다른 AZ에 Standby DB 자동 생성
  * 장애 발생 시 Standby로 Failover
  * 즉, 읽기 분산용 복제본이 아니라 장애 대비용 대기 인스턴스

## 35. 실습: VPC와 서브넷 그룹

## 36. 실습: VPC

* RDS 인스턴스는 AWS VPC 안에서 생성된다.
* 서브넷 그룹을 설정하여 데이터베이스가 위치할 네트워크 영역을 지정할 수 있다.
* 즉, RDS는 아무 네트워크에나 생성되는 것이 아니라, 반드시 특정 VPC와 서브넷 안에 배치된다.
  * 퍼블릭 서브넷: 인터넷 게이트웨이와 연결된 영역
  * 프라이빗 서브넷 : 외부 인터넷에서 직접 접근 불가한 영역
  * DB는 보통 프라이빗 서브넷에 생성
* 또한 RDS는 단일 서브넷이 아니라 DB Subnet Group이라는 형태로 서브넷 목록을 지정해야 합니다.
* 특히 Multi-AZ를 사용할 경우 서로 다른 AZ의 프라이빗 서브넷이 필요합니다.
* 이를 통해 외부 접근을 제어하고 보안을 강화할 수 있다.

## 37. 실습: 보안 그룹(Security Groups)

* RDS 인스턴스에 적용되는 보안 그룹은 인바운드 및 아웃바운드 트래픽을 제어합니다.
* 보안 그룹을 통해 특정 IP 또는 네트워크만 데이터베이스에 접근할 수 있도록 설정합니다.

```hcl
resource "aws_security_group" "rds_sg" {
  name        = "rds-security-group"   # 보안 그룹 이름
  description = "Allow database access" # 보안 그룹 설명
  vpc_id      = var.vpc_id             # VPC ID 변수

  ingress {
    from_port   = 3306                 # 시작 포트 (MySQL 기본 포트)
    to_port     = 3306                 # 종료 포트 (MySQL 기본 포트)
    protocol    = "tcp"                # 프로토콜 (TCP)
    cidr_blocks = ["10.0.0.0/16"]      # 내부 VPC 네트워크에만 접근 허용
  }

  egress {
    from_port   = 0               # 모든 포트 허용 (출력 트래픽)
    to_port     = 0                 # 모든 포트 허용 (출력 트래픽)
    protocol    = "-1"              # 모든 프로토콜 허용
    cidr_blocks = ["0.0.0.0/0"]     # 모든 IP에 출력 허용
  }
}
```

## 38. 실습: Terraform을 활용한 RDS 인스턴스 배포 및 검증

* 이 코드는 VPC와 EC2 인스턴스, 그리고 RDS용 보안 그룹 및 서브넷 그룹을 설정
* vpc 모듈을 통해 VPC, 서브넷 및 가용 영역을 구성하고, ec2 모듈로 EC2 인스턴스를 퍼블릭 서브넷에 배포
* 마지막으로, aws\_security\_group과 aws\_db\_subnet\_group 리소스를 통해 데이터베이스 접근을 제어하고, 프라이빗 서브넷에 RDS를 위한 서브넷 그룹을 생성

## 39. 실습: 전체 흐름

* VPC 생성
  * Public / Private Subnet 생성
  * EC2를 Public Subnet에 배치
  * RDS는 Private Subnet에 배치
  * Security Group으로 MySQL 포트 제어
  * DB Subnet Group 생성
  * 그 위에 RDS 생성

VPC 모듈

```hcl
module "vpc" {
  source             = "./modules/vpc"     # VPC 모듈 경로
  vpc_name           = var.vpc_name        # VPC 이름
  vpc_cidr           = var.vpc_cidr        # VPC CIDR 블록
  public_subnets     = var.public_subnets  # 퍼블릭 서브넷 리스트
  private_subnets    = var.private_subnets # 프라이빗 서브넷 리스트
  availability_zones = var.availability_zones# 가용 영역 리스트
}
```

* source = "./modules/vpc"
  * 이 모듈 코드가 위치한 경로입니다.
  * 즉, VPC를 직접 여기서 작성하지 않고 별도의 모듈 파일에서 불러온다.
* vpc\_name = var.vpc\_name
  * 생성할 VPC 이름
  * 태그나 리소스 명명에 사용될 수 있다.

vpc\_cidr

* vpc\_cidr = var.vpc\_cidr
  * VPC 전체 네트워크 대역입니다.
  * 예를 들어 10.0.0.0/16 같은 값이 들어갈 수 있다.
* public\_subnets = var.public\_subnets
  * 인터넷 접근이 가능한 퍼블릭 서브넷 목록입니다.
  * 보통 ALB, Bastion Host, 외부 공개 EC2 등이 이곳에 배치
* private\_subnets
  * private\_subnets = var.private\_subnets
  * 인터넷에서 직접 접근할 수 없는 프라이빗 서브넷 목록
  * RDS는 보통 이곳에 배치
* availability\_zones = var.availability\_zones
  * 서브넷을 어느 AZ에 배치할지 지정합니다.
  * Multi-AZ 구성이나 고가용성 구조를 위해 여러 AZ를 사용

## 40. 실습: EC2 모듈

```hcl
module "ec2" {
  source = "./modules/ec2" # EC2 모듈 경로

  ami_id          = var.ami_id                 # AMI ID
  instance_type   = var.instance_type          # EC2 인스턴스 타입
  vpc_id          = module.vpc.vpc_id          # VPC ID
  subnet_id       = module.vpc.public_subnets[0] # 퍼블릭 서브넷 ID (첫 번째 서브넷)
  instance_name   = var.instance_name          # 인스턴스 이름
  public_key_path = var.public_key_path        # 공개 키 경로
}
```

* source
  * EC2를 생성하는 모듈 경로입니다.
* ami\_id
  * EC2에 사용할 운영체제 이미지 ID
  * 예를 들면 Amazon Linux, Rocky Linux, Ubuntu 등
* instance\_type
  * EC2 사양입니다.
  * 예: t3.micro, t3.small
* vpc\_id
  * EC2를 어느 VPC에 넣을지 지정합니다.

## 41. 실습: RDS 보안 그룹

```hcl
resource "aws_security_group" "rds_sg" {
  name        = "rds-security-group"   # 보안 그룹 이름
  description = "Allow database access" # 보안 그룹 설명
  vpc_id      = module.vpc.vpc_id      # VPC ID

  ingress {
    from_port   = 3306         # 시작 포트 (MySQL 기본 포트)
    to_port     = 3306                 # 종료 포트 (MySQL 기본 포트)
    protocol    = "tcp"                # 프로토콜 (TCP)
    cidr_blocks = [var.allowed_cidr]   # 접근 허용 CIDR
  }

  egress {
    from_port   = 0                # 모든 포트 허용 (출력 트래픽)
    to_port     = 0                # 모든 포트 허용 (출력 트래픽)
    protocol    = "-1"              # 모든 프로토콜 허용
    cidr_blocks = ["0.0.0.0/0"] # 모든 IP에 출력 허용
  }
}

# DB 서브넷 그룹 생성
resource "aws_db_subnet_group" "this" {
  name       = "${var.vpc_name}-db-subnet-group" # DB 서브넷 그룹 이름
  subnet_ids = module.vpc.private_subnets        # 프라이빗 서브넷 ID 목록

  tags = {
    Name = "${var.vpc_name}-db-subnet-group"     # 태그 이름 설정
  }
}
```

* RDS는 일반 EC2처럼 그냥 서브넷 하나 지정해서 넣는 것이 아니라, DB Subnet Group을 통해 여러 서브넷을 묶어서 지정
* 여러개를 지정하는 이유는 Multi-AZ를 위해 다른 AZ의 서브넷이 필요할 수 있고 장애 대비를 위해 최소 2개 이상의 서브넷을 구성하는 경우가 많다. 여기서는 module.vpc.private\_subnets를 사용하므로 RDS는 프라이빗 서브넷에 배치 즉, DB를 외부 인터넷에 공개하지 않겠다는 구조입니다.

## 42. 실습: RDS 인스턴스 구성

이 코드는 MySQL 기반 AWS RDS 인스턴스를 생성하고 구성하는 Terraform 코드입니다. aws\_db\_instance 리소스를 통해 인스턴스 스토리지 크기, 데이터베이스 이름, 파라미터 그룹, 보안 그룹 등 다양한 설정을 정의합니다. 또한 백업 및 유지보수 시간, 스토리지 유형 등을 지정하여 인스턴스 운영을 최적화하며, 퍼블릭 액세스를 비활성화하여 보안을 강화합니다.

```hcl
resource "aws_db_instance" "my_rds_instance" {
  allocated_storage     = var.db_allocated_storage      # RDS 인스턴스의 스토리지 크기
  engine                = "mysql"                       # 데이터베이스 엔진 (MySQL)
  engine_version        = var.db_engine_version         # 데이터베이스 엔진 버전
  instance_class        = var.db_instance_class         # 인스턴스 유형
  db_name               = var.db_name                   # 데이터베이스 이름
  username              = var.db_username               # 관리자 계정 이름
  password              = var.db_password               # 관리자 계정 비밀번호
  parameter_group_name  = var.db_parameter_group_name   # 데이터베이스 파라미터 그룹
  skip_final_snapshot   = true                          # 삭제 시 최종 스냅샷 생성하지 않음
  publicly_accessible   = false                         # 퍼블릭 액세스 비활성화
  multi_az              = var.db_multi_az              # 다중 가용 영역 배포 여부
  vpc_security_group_ids = [aws_security_group.rds_sg.id] # 사용할 보안 그룹 ID
  db_subnet_group_name  = aws_db_subnet_group.my_db_subnet_group.name # DB 서브넷 그룹 이름

  # 백업 관련 설정
  backup_retention_period = 7                   # 백업 보존 기간 (일 단위)
  backup_window           = "02:00-03:00"        # 백업 시작 시간 (UTC 기준)

  # 모니터링 및 유지관리
  maintenance_window      = "sun:05:00-sun:06:00" # 유지보수 시간 (UTC 기준)

  # 스토리지 및 암호화 설정
  storage_type            = "gp2"                      # 스토리지 유형 (gp2: 범용 SSD)

  tags = {
    Name        = "My-RDS-MySQL"                       # RDS 인스턴스 이름 태그
    Environment = var.environment                      # 환경 태그
  }
}
```

* allocated\_storage
  * allocated\_storage = var.db\_allocated\_storage
  * RDS 스토리지 크기
  * 직접 숫자를 넣지 않고 변수로 받아서 처리
  * 즉, 개발/실습/운영 환경마다 크기를 다르게 줄 수 있다.
* engine
  * engine = "mysql"
  * MySQL RDS를 사용
* engine\_version
  * engine\_version = var.db\_engine\_version
  * MySQL 버전을 변수로 분리
  * 예를 들면 8.0.35 같은 식으로 넣을 수 있다.
* instance\_class
  * instance\_class = var.db\_instance\_class
  * RDS 서버 사양
  * 예: db.t3.micro, db.t3.small
  * 이 역시 변수화되어 있기 때문에 환경마다 쉽게 바꿀 수 있다.
* db\_name
  * db\_name = var.db\_name
  * RDS 안에 생성될 기본 데이터베이스 이름
* username / password
  * username = var.db\_username
  * password = var.db\_password
  * DB 접속 계정 정보입니다.
  * 실습에서는 변수에 넣어 관리
* parameter\_group\_name
  * parameter\_group\_name = var.db\_parameter\_group\_name
  * DB 동작 세부 설정을 담은 파라미터 그룹입니다.
  * 예를 들어 문자셋, 로그 설정, 타임존, 연결 수 제한 등의 DB 동작을 제어
* skip\_final\_snapshot
  * skip\_final\_snapshot = true
  * 삭제 시 최종 백업 스냅샷을 남기지 않습니다.
  * 실습에서는 빠른 삭제를 위해 true를 많이 사용
* publicly\_accessible
  * publicly\_accessible = false
  * 외부 인터넷 직접 접근 불가 즉, EC2를 통해 내부에서 접근하는 구조입니다.
* multi\_az
  * multi\_az = var.db\_multi\_az
  * 다중 AZ 활성화 여부를 변수로 제어합니다.
  * true: 장애 대비용 스탠바이 인스턴스 생성
  * false: 단일 AZ
* vpc\_security\_group\_ids
  * vpc\_security\_group\_ids = \[aws\_security\_group.rds\_sg.id]
  * RDS에 어떤 보안 그룹을 적용할지 지정
  * 여기서는 위에서 만든 rds\_sg를 연결
  * 즉, 이 보안 그룹 규칙이 RDS 접근 정책이 된다.
* db\_subnet\_group\_name
  * db\_subnet\_group\_name = aws\_db\_subnet\_group.my\_db\_subnet\_group.name
  * RDS가 배치될 DB 서브넷 그룹을 지정
  * 즉, 이 설정 때문에 RDS는 프라이빗 서브넷들에 배치
* 백업 관련 설정
  * backup\_retention\_period = 7(자동 백업을 며칠 동안 보관할지 지정, 7일)
  * backup\_window = "02:00-03:00"(자동 백업이 수행될 시간, UTC 기준 새벽 2시부터 3시 사이)

## 43. 실습: 읽기 전용 인스턴스 구성

* 다음 코드를 사용해 읽기 전용 인스턴스를 구성
* RDS에서 읽기 복제본(Read Replica)을 생성할 때, "읽기 전용"은 따로 명시할 필요는 없다.
* 읽기 복제본은 기본적으로 비동기적으로 원본 인스턴스의 데이터를 복제하고, 읽기 요청만 처리하는 인스턴스로 설정

## 44. 실습: 읽기 복제본 인스턴스

```hcl
resource "aws_db_instance" "read_replica" {
  engine              = "mysql"        # 데이터베이스 엔진 (MySQL)
  instance_class      = "db.t3.micro"  # 인스턴스 유형
  publicly_accessible = false         # 퍼블릭 액세스 비활성화
  skip_final_snapshot = true          # 최종 스냅샷 미생성

  replicate_source_db = aws_db_instance.my_rds_instance.id# 복제할 원본 인스턴스 ID

  tags = {
    Name        = "My-RDS-Read-Replica"     # 복제본 인스턴스 이름 태그
    Environment = "Production"                 # 환경 태그
  }
}
```

* replicate\_source\_db
  * replicate\_source\_db = aws\_db\_instance.my\_rds\_instance.id
  * 어떤 RDS를 원본으로 복제할지 지정
  * 즉, my\_rds\_instance를 원본으로 하여 읽기 전용 복제본을 하나 만들겠다는 의미
* 읽기 복제본의 특징
  * 원본 DB 데이터를 복제
  * 조회 요청 처리 가능
  * 쓰기 작업 불가
  * 읽기 부하 분산 가능
  * 원본과 약간의 복제 지연이 생길 수 있음
* 중요한 점은 Multi-AZ와 Read Replica는 목적이 다르다는 것입니다.
  * Multi-AZ : 장애 대비
  * Read Replica : 읽기 분산

## 45. 실습: Terraform RDS MySQL + EC2 Client + Read Replica 실습

* 최종 프로젝트 구조

rds-mysql-service/ │ ├── provider.tf ├── variables.tf ├── main.tf ├── outputs.tf ├── terraform.tfvars │ └── modules/ │ ├── network/ │ ├── variables.tf │ ├── main.tf │ └── outputs.tf │ └── ec2/ │ ├── variables.tf │ ├── main.tf │ └── outputs.tf │ └── rds/ ├── variables.tf ├── main.tf └── outputs.tf

## 46. 실습: STEP 1. Network Child Module 변수 작성

* VPC, Subnet, Internet Gateway, Route Table을 담당할
* Network Child Module의 입력 변수를 작성
  * rds-mysql-service\modules\network\variables.tf

```hcl
variable "vpc_name" {
  type        = string
  default     = "my-vpc"
}

variable "vpc_cidr" {
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  type        = list(string)
  default = [ "10.0.1.0/24", "10.0.2.0/24" ]
}

variable "private_subnets" {
  type        = list(string)
  default = [ "10.0.3.0/24", "10.0.4.0/24" ]
}

variable "availability_zones" {
  type        = list(string)
  default = [ "ap-northeast-2a", "ap-northeast-2b" ]
}
```

* vpc\_name
  * VPC의 Name Tag
* vpc\_cidr
  * VPC 전체 Network 범위
* public\_subnets
  * Public EC2가 배치될 Public Subnet CIDR
* private\_subnets
  * RDS가 배치될 Private Subnet CIDR
* availability\_zones
  * Subnet을 서로 다른 Availability Zone에 분산

## 47. 실습: STEP 2. VPC 생성

* Network Child Module에서 VPC를 생성
* RDS Endpoint와 AWS 내부 DNS를 사용할 수 있도록
* DNS Support와 DNS Hostname을 활성화
  * rds-mysql-service\modules\network\main.tf

## 48. 실습: VPC 생성

```hcl
resource "aws_vpc" "my_vpc" {
  # VPC Network 범위
  cidr_block = var.vpc_cidr

  # AWS 내부 DNS Resolver 사용
  enable_dns_support = true

  # DNS Hostname 사용
  enable_dns_hostnames = true

  tags = {
    Name = var.vpc_name
  }
}
```

* aws\_vpc
  * 새로운 VPC 생성
* cidr\_block
  * VPC에서 사용할 IP Address 범위
* enable\_dns\_support = true
  * VPC 내부에서 AWS DNS Resolver 사용
* enable\_dns\_hostnames = true
  * EC2 등에 DNS Hostname 사용
  * rds-mysql-service\modules\network\outputs.tf

```hcl
output "vpc_id" {
  description = "생성된 VPC ID"
  value = aws_vpc.my_vpc.id
}
```

* Network Child Module에서 생성한 VPC ID를 Root Module로 전달
* Root에서는 module.network.vpc\_id 형태로 사용

## 49. 실습: STEP 3. Public Subnet 생성

* EC2 DB Client가 들어갈 Public Subnet 2개를 생성
  * rds-mysql-service\modules\network\main.tf

## 50. 실습: Public Subnet 생성

```hcl
resource "aws_subnet" "public" {
  count = length(var.public_subnets)# Public Subnet CIDR 개수만큼 생성
  vpc_id = aws_vpc.my_vpc.id # STEP 2에서 생성한 VPC
  cidr_block = var.public_subnets[count.index]# 각 Public Subnet CIDR

  # 서로 다른 Availability Zone에 배치
  availability_zone = element(
    var.availability_zones,
    count.index
  )

  # EC2 생성 시 Public IP 자동 할당
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.vpc_name}-public-${count.index + 1}"
  }
}

   # rds-mysql-service\modules\network\outputs.tf
output "public_subnets" {
  description = "생성된 Public Subnet ID 목록"
  value = aws_subnet.public[*].id
}
```

* Public Subnet ID 전체를 List로 출력
* Root에서는 module.network.public\_subnets 형태로 사용

## 51. 실습: STEP 4. Private Subnet 생성

* RDS를 배치할 Private Subnet 2개를 생성
  * rds-mysql-service\modules\network\main.tf

## 52. 실습: Private Subnet 생성

```hcl
resource "aws_subnet" "private" {
  count = length(var.private_subnets)# Private Subnet CIDR 개수만큼 생성
  vpc_id = aws_vpc.my_vpc.id# 기존 VPC
  cidr_block = var.private_subnets[count.index]# 각 Private Subnet CIDR

  # 서로 다른 Availability Zone에 생성
  availability_zone = element(
    var.availability_zones,
    count.index
  )

  tags = {
    Name = "${var.vpc_name}-private-${count.index + 1}"
  }
}
```

* element()는 Terraform의 함수로 리스트에서 특정 인덱스 번호의 값을 꺼낼 때 사용
* map\_public\_ip\_on\_launch = true 설정을 사용하지 않는다.
  * rds-mysql-service\modules\network\outputs.tf

```hcl
output "private_subnets" {
  description = "생성된 Private Subnet ID 목록"
  value = aws_subnet.private[*].id
}
```

* Private Subnet ID 전체를 List로 출력
* 나중에 RDS DB Subnet Group에 module.network.private\_subnets 전체를 전달

## 53. 실습: STEP 5. Internet Gateway 생성

* Public Subnet의 EC2가 인터넷과 통신할 수 있도록
* Internet Gateway를 생성
  * rds-mysql-service\modules\network\main.tf

## 54. 실습: Internet Gateway 생성

```hcl
resource "aws_internet_gateway" "my_igw" {
  # 생성한 VPC에 연결
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "${var.vpc_name}-igw"
  }
}
```

* Internet Gateway
  * VPC와 Internet을 연결
* 단 Internet Gateway만 생성해서는 Public Subnet의 인터넷 통신이 완성되지 않는다.

## 55. 실습: STEP 6. Public Route Table 생성

* Public Subnet에서 외부로 나가는 Traffic을
* Internet Gateway로 전달
  * rds-mysql-service\modules\network\main.tf

## 56. 실습: Public Route Table

```hcl
resource "aws_route_table" "public" {
  # Route Table이 속할 VPC
  vpc_id = aws_vpc.my_vpc.id

  # 모든 외부 IPv4 Traffic을 Internet Gateway로 전달
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_igw.id
  }

  tags = {
    Name = "${var.vpc_name}-public-rt"
  }
}

# Public Subnet과 Route Table 연결
resource "aws_route_table_association" "public" {
  # Public Subnet 개수만큼 연결
  count = length(aws_subnet.public)

  # 각 Public Subnet
  subnet_id = aws_subnet.public[count.index].id

  # Public Route Table
  route_table_id = aws_route_table.public.id
}

# STEP 7. EC2 Child Module 변수 작성

RDS MySQL에 접속할 Client EC2용 Child Module을 작성

   # rds-mysql-service\modules\ec2\variables.tf
variable "ami_id" {
  description = "EC2에서 사용할 AMI ID"
  type        = string
  default     = "ami-0123456789abcdef0"
}

variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string
  default     = "t3.micro"
}

variable "subnet_id" {
  description = "EC2를 생성할 Public Subnet ID"
  type        = string
  default     = "subnet-0123456789abcdef0"
}

variable "public_key_path" {
  description = "SSH Public Key 경로"
  type        = string
  default     = "~/.ssh/my-key.pub"
}

variable "instance_name" {
  description = "EC2 Name Tag"
  type        = string
  default     = "db_client"
}

variable "vpc_id" {
  description = "Security Group을 생성할 VPC ID"
  type        = string
  default     = "vpc-0123456789abcdef0"
}
```

## 57. 실습: STEP 8. EC2 Key Pair 생성

* 로컬 PC의 Public Key를 AWS EC2 Key Pair로 등록
* Key Pair 이름 중복 방지를 위해 Random Number를 사용
  * rds-mysql-service\modules\ec2\main.tf

## 58. 실습: Key Pair 이름 중복 방지를 위한 Random Number

```hcl
resource "random_integer" "random_number" {
  min = 1000
  max = 9999
}

# EC2 Key Pair 생성
resource "aws_key_pair" "ec2_key_pair" {
  # 예 : ec2-key-pair-5382
  key_name = "ec2-key-pair-${random_integer.random_number.result}"

  # 로컬 Public Key 파일 내용 등록
  public_key = file(
    pathexpand(var.public_key_path)
  )
}
```

* random\_integer
  * 1000 \~ 9999 사이 Random Number 생성
* pathexpand()
  * \~/.ssh/my-key.pub

## 59. 실습: STEP 9. EC2 Security Group 생성

* DB Client EC2용 Security Group을 생성
  * rds-mysql-service\modules\ec2\main.tf

## 60. 실습: DB Client EC2 Security Group

```hcl
resource "aws_security_group" "ec2_sg" {
  vpc_id = var.vpc_id# Network Module에서 생성된 VPC
  name_prefix = "ec2-public-sg-"

  # SSH
  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks= [ "0.0.0.0/0" ]
  }

  # HTTP
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks= [ "0.0.0.0/0" ]
  }

  # 모든 Outbound 허용
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks= [ "0.0.0.0/0" ]
  }
}
```

## 61. 실습: STEP 10. DB Client EC2 생성

* Public Subnet에 RDS 접속용 EC2를 생성
  * rds-mysql-service\modules\ec2\main.tf

## 62. 실습: RDS 접속용 EC2

```hcl
resource "aws_instance" "ec2_instance" {
  ami = var.ami_id# Root에서 전달한 AMI
  instance_type = var.instance_type# EC2 Instance Type
  subnet_id = var.subnet_id# EC2 Instance Type

  vpc_security_group_ids = [ aws_security_group.ec2_sg.id ]# EC2 Security Group
  key_name = aws_key_pair.ec2_key_pair.key_name# 생성한 Key Pair 사용

  associate_public_ip_address = true# public IP address 할당

  tags = {
    Name = var.instance_name
  }
}

   # rds-mysql-service\modules\ec2\outputs.tf
output "instance_id" {
  description = "EC2 Instance ID"
  value = aws_instance.ec2_instance.id
}

output "public_ip" {
  description = "EC2 Public IP"
  value = aws_instance.ec2_instance.public_ip
}

output "public_dns" {
  description = "EC2 Public DNS"
  value = aws_instance.ec2_instance.public_dns
}
```

## 63. 실습: STEP 11. Root Module 기본 변수 작성

* Root Module에서 실제 Network와 EC2 값을 정의
  * rds-mysql-service\variables.tf

```hcl
variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-northeast-2"
}

variable "aws_profile" {
  description = "AWS CLI Profile"
  type        = string
  default     = "my-profile"
}

variable "environment" {
  description = "환경 이름"
  type        = string
  default     = "Production"
}

# Network
variable "vpc_name" {
  description = "VPC 이름"
  type        = string
  default     = "my-vpc"
}

variable "vpc_cidr" {
  description = "VPC CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "Public Subnet CIDR"
  type        = list(string)
  default = [ "10.0.1.0/24", "10.0.2.0/24" ]
}

variable "private_subnets" {
  description = "Private Subnet CIDR"
  type        = list(string)
  default = [ "10.0.3.0/24", "10.0.4.0/24" ]
}

variable "availability_zones" {
  description = "Availability Zone 목록"
  type        = list(string)
  default = [ "ap-northeast-2a", "ap-northeast-2b" ]
}

# EC2
variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string
  default     = "t3.micro"
}

variable "instance_name" {
  description = "EC2 Name Tag"
  type        = string
  default     = "db_client"
}

variable "public_key_path" {
  description = "SSH Public Key 경로"
  type        = string
  default     = "~/.ssh/my-key.pub"
}
```

## 64. 실습: STEP 12. Root Provider 설정

* Root Module에서 AWS Provider와 Random Provider를 설정
  * rds-mysql-service\provider.tf

```hcl
terraform {
  required_version = ">= 1.16.0"
  required_providers {

    aws = {
      source = "hashicorp/aws"
      version = ">= 5.73.0"
    }

    # EC2 Child Module의 random_integer에서 사용
    random = {
      source = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  profile = var.aws_profile
}
```

## 65. 실습: STEP 13. Root에서 Network Child Module 호출

* Root Module에서 module 블록을 사용해 Child Module을 호출한다.
* source는 사용할 Child Module의 경로를 지정한다.
* module 블록 안의 값은 Root Module의 변수 값을 Child Module로 전달하는 역할을 한다.
  * rds-mysql-service\main.tf

## 66. 실습: Network Child Module 호출

```hcl
module "network" {
  source = "./modules/network"# Local Module
  vpc_name        = var.vpc_name# Network 설정 전달
  vpc_cidr          = var.vpc_cidr# Network 설정 전달
  public_subnets   = var.public_subnets# Network 설정 전달
  private_subnets  = var.private_subnets# Network 설정 전달
  availability_zones = var.availability_zones# Network 설정 전달
}

# source = "./modules/network"
#직접 작성한 Network Child Module 호출

   # rds-mysql-service\outputs.tf
output "vpc_id" {
  description = "생성된 VPC ID"
  value = module.network.vpc_id
}

output "public_subnets" {
  description = "생성된 Public Subnet ID 목록"
  value = module.network.public_subnets
}

output "private_subnets" {
  description = "생성된 Private Subnet ID 목록"
  value = module.network.private_subnets
}
```

## 67. 실습: STEP 14. Amazon Linux 2023 AMI 조회

* 고정 AMI ID 대신 Amazon Linux 2023 최신 AMI를 자동 조회
  * rds-mysql-service\main.tf

## 68. 실습: 최신 Amazon Linux 2023 AMI 조회

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners = [ "amazon" ]

  filter {
    name = "name"
    values = [ "al2023-ami-*" ]
  }

  filter {
    name = "architecture"
    values = [
      "x86_64"
    ]
  }
}
```

## 69. 실습: STEP 15. Root에서 EC2 Child Module 호출

* Network Module에서 만든 첫 번째 Public Subnet에
* DB Client EC2를 생성
  * rds-mysql-service\main.tf

## 70. 실습: EC2 Child Module 호출

```hcl
module "ec2" {
  source = "./modules/ec2"
  ami_id = data.aws_ami.al2023.id# Amazon Linux 2023 AMI
  instance_type = var.instance_type# Instance_Type
  vpc_id = module.network.vpc_id# Network Module에서 생성한 VPC

  subnet_id = module.network.public_subnets[0]# 첫 번째 Public Subnet에 EC2 배치
  instance_name = var.instance_name# EC2 Name

  public_key_path = var.public_key_path# SSH Public Key
}

   # rds-mysql-service\outputs.tf
output "public_dns" {
  description = "DB Client EC2 Public DNS"
  value = module.ec2.public_dns
}

terraform  plan
```

## 71. 실습: STEP 16. RDS 변수 작성

* MySQL RDS에 사용할 변수를 Root Module에 추가
  * rds-mysql-service\variables.tf

```hcl
variable "vpc_id" {
  description = "RDS Security Group을 생성할 VPC ID"
  type        = string
}

variable "private_subnets" {
  description = "RDS가 사용할 Private Subnet ID 목록"
  type        = list(string)
}

variable "vpc_name" {
  description = "VPC 이름"
  type        = string
}

variable "allowed_cidr" {
  description = "RDS MySQL 접근을 허용할 CIDR"
  type        = string
}
```

## 72. 실습: STEP 17. RDS Security Group 생성

* RDS MySQL에 접근할 수 있는 Network 범위를 제어하는 Security Group을 생성하는 단계
* MySQL 기본 Port인 TCP 3306을 허용하고, RDS가 생성될 VPC는 Root Module에서 전달받은 vpc\_id를 사용
* RDS MySQL의 TCP 3306 Port를 허용
  * rds-mysql-service\modules\rds\variables.tf

## 73. 실습: RDS Security Group

```hcl
resource "aws_security_group" "rds_sg" {
  name = "rds-security-group"
  vpc_id = module.network.vpc_id# Network Module에서 생성한 VPC

  # MySQL
  ingress {
    from_port = 3306
    to_port = 3306
    protocol = "tcp"

    cidr_blocks = [ var.allowed_cidr ]# 같은 VPC Network에서 접근 허용
  }

  # 모든 Outbound 허용
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = [ "0.0.0.0/0" ]
  }
}

   # rds-mysql-service\modules\rds\outputs.tf
output "rds_security_group_id" {
  description = "RDS Security Group ID"
  value = aws_security_group.rds_sg.id
}
```

## 74. 실습: STEP 18. DB Subnet Group 생성

* RDS가 배치될 Private Subnet들을 하나의 DB Subnet Group으로 묶는 단계
* RDS는 일반 EC2처럼 특정 Subnet 하나를 직접 지정하는 것이 아니라, 여러 Private Subnet을 DB Subnet Group으로 구성해서 사용
  * rds-mysql-service\modules\rds\main.tf

## 75. 실습: RDS DB Subnet Group

```hcl
resource "aws_db_subnet_group" "my_db_subnet_group" {
  name = "${var.vpc_name}-db-subnet-group"# DB Subnet Group 이름
  subnet_ids = var.private_subnets

  tags = {
    Name = "${var.vpc_name}-db-subnet-group"
  }
}

   # rds-mysql-service\modules\rds\outputs.tf
output "db_subnet_group_name" {
  description = "RDS DB Subnet Group 이름"
  value       = aws_db_subnet_group.my_db_subnet_group.name
}
```

## 76. 실습: STEP 19. Primary RDS MySQL 생성

* Private Subnet 환경에 MySQL Primary RDS를 생성
  * rds-mysql-service\variables.tf

```hcl
variable "db_allocated_storage" {
  description = "RDS Storage 크기"
  type        = number
  default     = 20
}

variable "db_engine_version" {
  description = "MySQL Engine Version"
  type        = string
  default     = "8.0"
}
variable "db_instance_class" {
  description = "RDS Instance Class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_name" {
  description = "Database 이름"
  type        = string
  default     = "mydatabase"
}

variable "db_username" {
  description = "RDS Master Username"
  type        = string
  default     = "admin"
}

variable "db_password" {
  description = "RDS Master Password"
  type        = string
  sensitive   = true
  default     = "admin1234"
}

variable "db_parameter_group_name" {
  description = "DB Parameter Group"
  type        = string
  default     = "default.mysql8.0"
}

variable "db_multi_az" {
  description = "RDS Multi-AZ 활성화 여부"
  type        = bool
  default     = false
}
   # rds-mysql-service\main.tf
# Primary RDS MySQL
resource "aws_db_instance" "my_rds_instance" {
  allocated_storage = var.db_allocated_storage # RDS Storage 크기
  engine = "mysql"# MySQL
  engine_version = var.db_engine_version# MySQL Version
  instance_class = var.db_instance_class# RDS Instance Type
  db_name = var.db_name# 최초 생성할 Database

  username = var.db_username# 관리자 계정
  password = var.db_password# 관리자 Password

  parameter_group_name = var.db_parameter_group_name# Parameter Group

  skip_final_snapshot = true# 교육 실습이므로 destroy 시 Final Snapshot을 생성하지 않음
  publicly_accessible = false# 인터넷에서 직접 접속하지 않도록 설정
  multi_az = var.db_multi_az# Multi-AZ 사용 여부

  vpc_security_group_ids = [ aws_security_group.rds_sg.id ]# RDS Security Group

  # Private Subnet으로 구성된 DB Subnet Group
  db_subnet_group_name = aws_db_subnet_group.my_db_subnet_group.name

  backup_retention_period = 7# 자동 Backup 7일 보존
  backup_window = "02:00-03:00"# Backup 시간 UTC 기준
  maintenance_window = "sun:05:00-sun:06:00"# 유지보수 시간 UTC 기준

  storage_type = "gp3"# General Purpose SSD

  tags = {
    Name = "My-RDS-MySQL"
  }
}

# Primary RDS MySQL 생성

resource "aws_db_instance" "my_rds_instance" {

  # RDS가 사용할 저장 공간 크기 (단위는 GB)
  # 예: 20이면 20GB Storage 생성
  allocated_storage = var.db_allocated_storage

  # 사용할 Database Engine
  # 여기서는 MySQL 사용
  engine = "mysql"

  # 사용할 MySQL Version 지정, 예: "8.0"
  engine_version = var.db_engine_version

  # RDS Instance의 성능 사양 지정
  instance_class = var.db_instance_class

  # RDS 생성 시 MySQL 내부에 같이 생성할 Database 이름 (예: mydatabase)
  db_name = var.db_name

  # RDS MySQL에 접속할 Master 관리자 계정 이름 (예: admin)
  username = var.db_username

  # RDS MySQL Master 관리자 계정의 Password
  password = var.db_password

  # MySQL의 세부 동작 설정을 관리하는 Parameter Group 지정
  # 예: 문자셋, 최대 연결 수, 로그 설정 등의 DB 설정에 사용
  parameter_group_name = var.db_parameter_group_name

  # RDS 삭제 시 마지막 Backup Snapshot 생성 여부
  # true  : Final Snapshot을 생성하지 않고 RDS 삭제
  # false : Final Snapshot을 생성한 후 RDS 삭제
  skip_final_snapshot = true

  # RDS를 Internet에서 직접 접속할 수 있도록 공개할지 설정
  # false : Internet에서 직접 접근 불가
  # 현재 실습에서는 VPC 내부의 EC2를 통해 RDS에 접속
  publicly_accessible = false

  # Multi-AZ 사용 여부
  # true  : 다른 Availability Zone에 Standby DB를 추가로 생성 Primary DB에 장애가 발생하면 Standby DB로 자동 전환
  # false : 하나의 Availability Zone에만 RDS 생성
  multi_az = var.db_multi_az

  # RDS에 적용할 Security Group 지정
  # 이 Security Group의 Inbound Rule을 통해 어떤 Server가 MySQL 3306 Port로 접근할 수 있는지 제어
  vpc_security_group_ids = [ aws_security_group.rds_sg.id ]

  # RDS가 배치될 DB Subnet Group 지정
  # DB Subnet Group에 등록된 Private Subnet 중 하나에 Primary RDS가 배치됨
  # Multi-AZ를 사용하면 다른 AZ의 Private Subnet에 Standby DB도 배치될 수 있음
  db_subnet_group_name = aws_db_subnet_group.my_db_subnet_group.name

  # 자동 Backup 보관 기간
  # 7이면 RDS 자동 Backup을 7일 동안 보관된다. 이 기간 내의 특정 시점으로 복구할 수 있다.
  backup_retention_period = 7

  # 자동 Backup을 수행할 시간대 지정
  # 매일 UTC 02:00 ~ 03:00 사이에 AWS가 자동 Backup 작업을 수행
  # Backup이 진행되는 동안에도 일반적으로 DB는 계속 사용 가능
  backup_window = "02:00-03:00"

  # RDS 유지보수 작업을 적용할 시간대 지정
  # AWS에서 DB Engine Patch, 운영체제 Update, RDS 내부 System Update 등의 유지보수가 필요한 경우
  # 일요일 UTC 05:00 ~ 06:00 사이에 해당 작업을 적용
  # 유지보수 작업이 실제로 있으면 RDS에 Patch / Update 적용
  # 유지보수 작업이 없으면 이 시간에도 DB는 평소처럼 계속 정상 동작
  maintenance_window = "sun:05:00-sun:06:00"

  # RDS에서 사용할 Storage 종류
  # gp3 = General Purpose SSD (일반적인 Database 용도로 사용하는 범용 SSD Storage)
  storage_type = "gp3"

  tags = {
    Name = "My-RDS-MySQL"
    Environment = var.environment
  }
}
```

* RDS 접속에 사용할 Endpoint를 Root Module에서도 사용할 수 있도록 Output으로 전달
  * rds-mysql-service\outputs.tf

```hcl
output "rds_endpoint" {
  description = "Primary RDS MySQL Endpoint"
  value = aws_db_instance.my_rds_instance.endpoint
}
```

## 77. 실습: STEP 20. Read Replica 생성

* Primary RDS의 데이터를 복제하는 Read Replica를 생성
* Read Replica는 Primary RDS의 읽기 Traffic을 분산하기 위해 사용
* Read Replica는 Primary RDS의 데이터를 복제해서 읽기 전용으로 사용하는 RDS 인스턴스로 주요 목적은 SELECT 같은 조회 요청을 분산해서 Primary RDS의 부하를 줄이는 것입니다.
* Primary RDS는 일반적으로 다음과 같은 작업을 처리 INSERT UPDATE DELETE SELECT
* 서비스 규모가 커지면 특히 SELECT 같은 조회 요청이 많이 발생할 수 있다.
* 모든 조회 요청을 Primary RDS 하나에서 처리하면 Database 부하가 증가할 수 있다.
* 이때 Read Replica를 생성하면 일부 조회 요청을 Read Replica로 분산할 수 있다.

Primary RDS

* 쓰기 작업 처리
* INSERT
* UPDATE
* DELETE
* SELECT

Read Replica

* 주로 조회 작업 처리
* SELECT
* 즉, Read Replica의 가장 큰 목적은 읽기 Traffic 분산이다.
  * rds-mysql-service\modules\rds\main.tf

## 78. 실습: MySQL Read Replica

```hcl
resource "aws_db_instance" "read_replica" {

  engine = "mysql"  # Database Engine
  instance_class = "db.t3.micro"# Replica Instance Type
  publicly_accessible = false  # Internet 직접 접근 금지
  skip_final_snapshot = true# 삭제 시 Final Snapshot 생성 안 함
  replicate_source_db = aws_db_instance.my_rds_instance.identifier# Primary RDS를 복제 Source로 지정

  tags = {
    Name = "My-RDS-Read-Replica"
  }
}
```

* replicate\_source\_db
  * Read Replica가 복제할 Primary RDS 지정
* Read Replica
  * 주로 Read Traffic 분산에 사용
  * rds-mysql-service\outputs.tf

```hcl
output "rds_endpoint_read_replica" {
  description = "Read Replica Endpoint"
  value = aws_db_instance.read_replica.endpoint
}
```

## 79. 실습: STEP 21. Root Module의 variables.tf, terraform.tfvars 작성

* 최종 실습에서 사용할 실제 값을 입력
  * rds-mysql-service\variables.tf

```hcl
variable "allowed_cidr" {
  description = "RDS MySQL 접근을 허용할 CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "db_allocated_storage" {
  description = "RDS Storage 크기"
  type        = number
  default     = 20
}

variable "db_engine_version" {
  description = "MySQL Engine Version"
  type        = string
  default     = "8.0"
}

variable "db_instance_class" {
  description = "RDS Instance Class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_name" {
  description = "Database 이름"
  type        = string
  default     = "mydatabase"
}

variable "db_username" {
  description = "RDS Master Username"
  type        = string
  default     = "admin"
}

variable "db_password" {
  description = "RDS Master Password"
  type        = string
  sensitive   = true
  default     = "securepassword123!"
}

variable "db_parameter_group_name" {
  description = "DB Parameter Group"
  type        = string
  default     = "default.mysql8.0"
}

variable "db_multi_az" {
  description = "RDS Multi-AZ 활성화 여부"
  type        = bool
  default     = false
}

   # rds-mysql-service\terraform.tfvars
# AWS
aws_region = "ap-northeast-2"
aws_profile = "my-profile"
environment = "Production"

# Network
vpc_name = "my-vpc"
vpc_cidr = "10.0.0.0/16"
public_subnets = [ "10.0.1.0/24", "10.0.2.0/24" ]
private_subnets = [ "10.0.3.0/24",  "10.0.4.0/24" ]
availability_zones = [  "ap-northeast-2a", "ap-northeast-2b" ]

# RDS
# 같은 VPC 내부에서 MySQL 접근 허용
allowed_cidr = "10.0.0.0/16"
db_allocated_storage = 20
db_engine_version = "8.0"
db_instance_class = "db.t3.micro"
db_name = "mydatabase"

# RDS Master 계정
db_username = "admin"

# 교육 실습용 Password
db_password = "admin1234"
db_parameter_group_name = "default.mysql8.0"

# Multi-AZ 사용
db_multi_az = true

# EC2 DB Client
instance_type = "t2.micro"
instance_name = "db_client"
public_key_path = "~/.ssh/my-key.pub"

   # rds-mysql-service\terraform.tfvars
# AWS
aws_region  = "ap-northeast-2"
aws_profile = "my-profile"
environment = "Production"

# Network
vpc_name = "my-vpc"
vpc_cidr = "10.0.0.0/16"

public_subnets = [ "10.0.1.0/24",  "10.0.2.0/24" ]
private_subnets = [ "10.0.3.0/24",  "10.0.4.0/24" ]

availability_zones = [ "ap-northeast-2a", "ap-northeast-2b" ]

# RDS

# 같은 VPC 내부에서 MySQL 접근 허용
allowed_cidr = "10.0.0.0/16"

db_allocated_storage = 20
db_engine_version    = "8.0"
db_instance_class    = "db.t3.micro"
db_name              = "mydatabase"

# RDS Master 계정
db_username = "admin"

# 교육 실습용 Password
db_password = "admin1234"

db_parameter_group_name = "default.mysql8.0"

# Multi-AZ 사용
db_multi_az = true

# EC2 DB Client
instance_type   = "t2.micro"
instance_name   = "db_client"
public_key_path = "~/.ssh/my-key.pub"

# STEP 22. 전체 실행

PS C:\my-terraform> terraform init
PS C:\my-terraform> terraform plan
PS C:\my-terraform> terraform apply

# STEP 23. Output 확인

PS C:\my-terraform> terraform output
vpc_id
public_subnets
private_subnets
public_dns
rds_security_group_id
rds_endpoint
rds_endpoint_read_replica
```

* VPC 생성 확인

![VPC 생성 확인 화면](<../.gitbook/assets/1 (3).png>)

* 퍼블릭 서브넷 , 프라이빗 서브넷 확인

![퍼블릭 서브넷 , 프라이빗 서브넷 확인 화면](<../.gitbook/assets/2 (3).png>)

* DB로 접속하기위해 생성한 EC2 확인

![DB로 접속하기위해 생성한 EC2 확인 화면](<../.gitbook/assets/3 (3).png>)

* Database 생성 확인

![Database 생성 확인 화면](<../.gitbook/assets/4 (3).png>)

![Database 생성 확인 화면](<../.gitbook/assets/5 (2).png>)

* EC2 인스턴스로 이동

![EC2 인스턴스로 이동 화면](<../.gitbook/assets/6 (2).png>)

나타난 정보를 사용해 ec2로 접속

```powershell
PS C:\Users\soldesk>
ssh -i $home\.ssh\my-key ec2-user@ec2-43-201-97-196.ap-northeast-2.compute.amazonaws.com
   ,     #_
   ~\_  ####_
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   Amazon Linux 2023 (ECS Optimized)
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'

For documentation, visit http://aws.amazon.com/documentation/ecs
[ec2-user@ip-10-0-1-19 ~]$

# mysql client를 구성

# 1. MySQL 클라이언트 설치
[ec2-user@ip-10-0-1-19 ~]# dnf install -y mariadb105

# 2. 설치 확인
[ec2-user@ip-10-0-1-19 ~]# mysql --version

# mysql을 사용해 rds로 접속
# 3. MySQL 클라이언트 설치
[ec2-user@ip-10-0-1-19 ~]$ mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p

[ec2-user@ip-10-0-1-19 ~]#
mysql -h <rds-endpoint> -u admin -p
Enter password: admin1234
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 38
Server version: 8.0.44 Source distribution
Copyright (c) 2000, 2026, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema|
| mydatabase         |
| mysql              |
| performance_schema|
| sys                |
+--------------------+
5 rows in set (0.02 sec)
```

* 워크밴치 Connection Name: Terraform-db Connection Method: Standard TCP/IP over SSH

SSH Hostname: 43.201.97.196# EC2 퍼블릭 IP SSH Username: ec2-user# EC2 접속 계정 SSH Password: 생략

```
SSH Key File : C:\Users\soldesk\.ssh\my-key# 키파일 위치

# RDS 엔드포인트 주소
MySQL Hostname: <rds-endpoint>
MySQL Server Port: 3306
Username: admin
Password: admin1234
```

![Password: admin1234 화면](<../.gitbook/assets/7 (2).png>)

![Password: admin1234 화면](<../.gitbook/assets/8 (2).png>)

![Password: admin1234 화면](<../.gitbook/assets/9 (2).png>)

## 80. 실습: Terraform Aurora MySQL + EC2 Client + Reader 실습

## 81. 실습: 프로젝트 구조

aurora-mysql-service/ │ ├── provider.tf ├── variables.tf ├── main.tf ├── outputs.tf ├── terraform.tfvars │ └── modules/ │ ├── network/ │ ├── variables.tf │ ├── main.tf │ └── outputs.tf │ └── ec2/ │├── variables.tf │├── main.tf │└── outputs.tf └── rds/ ├── variables.tf ├── main.tf └── outputs.tf

## 82. 실습: STEP 1. Network Child Module 변수 작성

* VPC, Public Subnet, Private Subnet, Availability Zone을 생성하기 위해 Network Child Module에서 사용할 입력 변수를 작성
* Public Subnet에는 Aurora에 접속할 DB Client EC2를 배치
* Private Subnet에는 Aurora MySQL Cluster를 배치
  * aurora\modules\network\variables.tf

```hcl
variable "vpc_name" {
  description = "VPC 이름"
  type        = string
  default     = "my-vpc"
}

variable "vpc_cidr" {
  description = "VPC CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "Public Subnet CIDR 목록"
  type        = list(string)
  default = [ "10.0.1.0/24", "10.0.2.0/24" ]
}

variable "private_subnets" {
  description = "Private Subnet CIDR 목록"
  type        = list(string)
  default = [ "10.0.3.0/24", "10.0.4.0/24" ]
}

variable "availability_zones" {
  description = "Subnet을 생성할 Availability Zone 목록"
  type        = list(string)
  default = [ "ap-northeast-2a",  "ap-northeast-2b"
  ]
}

# STEP 2. VPC 생성

[설명]
```

* Aurora MySQL과 EC2가 사용할 VPC를 생성
* Aurora Endpoint와 AWS 내부 DNS를 정상적으로 사용할 수 있도록 DNS Support와 DNS Hostname을 활성화
  * aurora\modules\network\main.tf

## 83. 실습: VPC 생성

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block = var.vpc_cidr
  enable_dns_support = true# AWS 내부 DNS Resolver 사용
  enable_dns_hostnames = true# DNS Hostname 사용

  tags = {
    Name = var.vpc_name
  }
}

   # aurora\modules\network\outputs.tf
output "vpc_id" {
  description = "생성된 VPC ID"
  value = aws_vpc.my_vpc.id
}
```

* 설정 설명
* cidr\_block
  * VPC에서 사용할 IP Address 범위
* enable\_dns\_support
  * VPC 내부에서 AWS DNS Resolver 사용
* enable\_dns\_hostnames
  * EC2와 Aurora Endpoint의 DNS 이름 사용
* vpc\_id
  * 생성된 VPC ID를 Root Module에서 사용할 수 있도록 출력

## 84. 실습: STEP 3. Public Subnet 생성

* Aurora MySQL에 접속하기 위한 DB Client EC2가 배치될 Public Subnet 2개를 생성
* Subnet을 서로 다른 Availability Zone에 생성
  * aurora\modules\network\main.tf

## 85. 실습: Public Subnet 생성

```hcl
resource "aws_subnet" "public" {
  count = length(var.public_subnets)
  vpc_id = aws_vpc.my_vpc.id
  cidr_block = var.public_subnets[count.index]
  availability_zone = element(
    var.availability_zones,
    count.index
  )

  # EC2 생성 시 Public IP 자동 할당
  map_public_ip_on_launch = true
  tags = {
    Name = "${var.vpc_name}-public-${count.index + 1}"
  }
}

   # aurora\modules\network\outputs.tf
output "public_subnets" {
  description = "생성된 Public Subnet ID 목록"
  value = aws_subnet.public[*].id
}
```

* count
  * Public Subnet CIDR 개수만큼 Subnet 생성
* count.index
  * 각 Subnet의 Index 번호
* availability\_zone
  * Public Subnet을 서로 다른 AZ에 생성
* map\_public\_ip\_on\_launch
  * EC2 생성 시 Public IP 자동 할당

## 86. 실습: STEP 4. Private Subnet 생성

* Aurora MySQL Cluster가 사용할 Private Subnet 2개를 생성
* Aurora Database는 외부 Internet에 직접 공개하지 않고 Private Network 내부에 배치
  * aurora\modules\network\main.tf

## 87. 실습: Private Subnet 생성

```hcl
resource "aws_subnet" "private" {
  count = length(var.private_subnets)
  vpc_id = aws_vpc.my_vpc.id
  cidr_block = var.private_subnets[count.index]
  availability_zone = element(
    var.availability_zones,
    count.index
  )
  tags = {
    Name = "${var.vpc_name}-private-${count.index + 1}"
  }
}

   # aurora\modules\network\outputs.tf
output "private_subnets" {
  description = "생성된 Private Subnet ID 목록"
  value = aws_subnet.private[*].id
}
```

* 설정 설명
* private\_subnets
  * Aurora가 배치될 Private Network
* availability\_zone
  * 서로 다른 AZ에 Private Subnet 생성
* map\_public\_ip\_on\_launch를 사용하지 않는다.
  * Private Subnet의 Resource에 Public IP를 자동 할당하지 않음

## 88. 실습: STEP 5. Internet Gateway 생성

* Public Subnet의 EC2가 Internet과 통신할 수 있도록 Internet Gateway를 생성
  * aurora\modules\network\main.tf

## 89. 실습: Internet Gateway 생성

```hcl
resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "${var.vpc_name}-igw"
  }
}
```

## 90. 실습: STEP 6. Public Route Table 생성

* Public Subnet에서 외부로 나가는 Traffic을 Internet Gateway로 전달
* 생성한 Public Route Table을 Public Subnet 2개에 연결
  * aurora\modules\network\main.tf

## 91. 실습: Public Route Table 생성

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.my_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_igw.id
  }
  tags = {
    Name = "${var.vpc_name}-public-rt"
  }
}

# Public Subnet과 Route Table 연결
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  subnet_id = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

* 설정 설명
* 0.0.0.0/0
  * 모든 외부 IPv4 Network
* gateway\_id
  * 외부 Traffic을 Internet Gateway로 전달
* aws\_route\_table\_association
  * Public Subnet과 Public Route Table 연결

## 92. 실습: STEP 7. EC2 Child Module 변수 작성

* Aurora MySQL에 접속할 DB Client EC2를 생성하기 위한 입력 변수를 작성
  * aurora\modules\ec2\variables.tf

```hcl
variable "ami_id" {
  description = "EC2에서 사용할 AMI ID"
  type        = string
}

variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string
  default     = "t3.micro"
}

variable "subnet_id" {
  description = "EC2를 생성할 Public Subnet ID"
  type        = string
  default     = "subnet-0123456789abcdef0"
}

variable "public_key_path" {
  description = "SSH Public Key 경로"
  type        = string
  default     = "~/.ssh/my-key.pub"
}

variable "instance_name" {
  description = "EC2 Name Tag"
  type        = string
  default     = "aurora-db-client"
}

variable "vpc_id" {
  description = "Security Group을 생성할 VPC ID"
  type        = string
  default     = "vpc-0123456789abcdef0"
}
```

* 설정 설명
* ami\_id
  * EC2 운영체제 Image
* instance\_type
  * EC2 Instance 사양
* subnet\_id
  * EC2가 들어갈 Public Subnet
* public\_key\_path
  * SSH Public Key 파일 경로
* vpc\_id
  * Security Group을 생성할 VPC

## 93. 실습: STEP 8. EC2 Key Pair 생성

* 사용자 PC의 SSH Public Key를 AWS EC2 Key Pair로 등록
* Key Pair 이름 중복을 방지하기 위해 Random Number를 사용
  * aurora\modules\ec2\main.tf

## 94. 실습: Key Pair 이름 중복 방지를 위한 Random Number

```hcl
resource "random_integer" "random_number" {
  min = 1000
  max = 9999
}

# EC2 Key Pair 생성
resource "aws_key_pair" "ec2_key_pair" {
  key_name = "aurora-key-pair-${random_integer.random_number.result}"
  public_key = file(
    pathexpand(var.public_key_path)
  )
}
```

* 설정 설명
* random\_integer
  * 1000 \~ 9999 사이 Random Number 생성
* pathexpand()
  * \~/.ssh 경로를 현재 사용자 Home Directory 경로로 변환
* file()
  * Public Key 파일 내용을 읽어 AWS Key Pair에 등록

## 95. 실습: STEP 9. EC2 Security Group 생성

* DB Client EC2에 적용할 Security Group을 생성
* 사용자 PC에서 SSH 접속할 수 있도록 TCP 22 Port를 허용
  * aurora\modules\ec2\main.tf

## 96. 실습: DB Client EC2 Security Group

```hcl
resource "aws_security_group" "ec2_sg" {
  vpc_id = var.vpc_id
  name_prefix = "aurora-client-sg-"
  # SSH
  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = [  "0.0.0.0/0" ]
  }

  # 모든 Outbound 허용
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = [  "0.0.0.0/0"  ]
  }

  tags = {
    Name = "aurora-client-sg"
  }
}
```

* 설정 설명
* TCP 22
  * SSH 기본 Port
* 0.0.0.0/0
  * 실습을 위해 모든 IP에서 SSH 접근 허용
* egress
  * EC2에서 외부로 나가는 모든 Traffic 허용

## 97. 실습: STEP 10. DB Client EC2 생성

* Public Subnet에 Aurora MySQL 접속용 EC2를 생성
* 이 EC2를 통해 Private Subnet에 있는 Aurora MySQL에 접속
  * aurora\modules\ec2\main.tf

## 98. 실습: Aurora MySQL 접속용 EC2

```hcl
resource "aws_instance" "ec2_instance" {
  ami = var.ami_id
  instance_type = var.instance_type
  subnet_id = var.subnet_id
  vpc_security_group_ids = [   aws_security_group.ec2_sg.id  ]
  key_name = aws_key_pair.ec2_key_pair.key_name
  associate_public_ip_address = true

  tags = {
    Name = var.instance_name
  }
}

   # aurora\modules\ec2\outputs.tf
output "instance_id" {
  description = "EC2 Instance ID"
  value = aws_instance.ec2_instance.id
}

output "public_ip" {
  description = "EC2 Public IP"
  value = aws_instance.ec2_instance.public_ip
}

output "public_dns" {
  description = "EC2 Public DNS"
  value = aws_instance.ec2_instance.public_dns
}
```

## 99. 실습: STEP 11. Root Module 기본 변수 작성

* Root Module에서 Network와 EC2 Child Module에 전달할 실제 기본값을 정의
  * aurora\variables.tf

```hcl
variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-northeast-2"
}

variable "aws_profile" {
  description = "AWS CLI Profile"
  type        = string
  default     = "my-profile"
}

variable "environment" {
  description = "환경 이름"
  type        = string
  default     = "Production"
}

# Network
variable "vpc_name" {
  description = "VPC 이름"
  type        = string
  default     = "my-vpc"
}

variable "vpc_cidr" {
  description = "VPC CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "Public Subnet CIDR"
  type        = list(string)
  default = [  "10.0.1.0/24",  "10.0.2.0/24" ]
}

variable "private_subnets" {
  description = "Private Subnet CIDR"
  type        = list(string)
  default = [ 10.0.3.0/24",  "10.0.4.0/24" ]
}

variable "availability_zones" {
  description = "Availability Zone 목록"
  type        = list(string)
  default = [ "ap-northeast-2a", "ap-northeast-2b" ]
}

# EC2
variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string
  default     = "t3.micro"
}

variable "instance_name" {
  description = "EC2 Name Tag"
  type        = string
  default     = "aurora-db-client"
}

variable "public_key_path" {
  description = "SSH Public Key 경로"
  type        = string
  default     = "~/.ssh/my-key.pub"
}
```

* 설정 설명
* Root Module에서 실제 Network와 EC2 설정 값을 관리
* Child Module의 변수에 Root Module의 값을 전달하여 실제 AWS Resource를 생성

## 100. 실습: STEP 12. Root Provider 설정

* Terraform에서 AWS Provider와 Random Provider를 사용하도록 설정
  * aurora\provider.tf

```hcl
terraform {
  required_version = ">= 1.16.0"
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = ">= 6.37.0"
    }
    random = {
      source = "hashicorp/random"
      version = "~> 3.9"
    }
  }
}

provider "aws" {
  region = var.aws_region
  profile = var.aws_profile
}
```

* aws Provider
  * AWS Resource 생성
* random Provider
  * EC2 Key Pair 이름 Random Number 생성
* region
  * AWS Seoul Region
* profile
  * AWS CLI Profile 사용

## 101. 실습: STEP 13. Root에서 Network Child Module 호출

* 앞에서 작성한 Network Child Module을 Root Module에서 호출
* VPC와 Public / Private Subnet을 실제로 생성할 값을 전달
* 모듈 variables.tf에는 default값이 없기 때문에 root에서 vpc를 호출해서 root 변수에 있는값으로 실행한다.
  * aurora\main.tf

## 102. 실습: Network Child Module 호출

```hcl
module "network" {
  source = "./modules/network"
  vpc_name = var.vpc_name
  vpc_cidr = var.vpc_cidr
  public_subnets = var.public_subnets
  private_subnets = var.private_subnets
  availability_zones = var.availability_zones
}

   # aurora\outputs.tf
output "vpc_id" {
  description = "생성된 VPC ID"
  value = module.network.vpc_id
}

output "public_subnets" {
  description = "생성된 Public Subnet ID 목록"
  value = module.network.public_subnets
}

output "private_subnets" {
  description = "생성된 Private Subnet ID 목록"
  value = module.network.private_subnets
}
```

* source: 직접 작성한 Network Child Module 경로
* module.network.vpc\_id: Network Module에서 생성된 VPC ID
* module.network.public\_subnets: Public Subnet ID List
* module.network.private\_subnets: Private Subnet ID List

## 103. 실습: STEP 14. Amazon Linux 2023 AMI 조회

* 고정된 AMI ID를 사용하지 않고 현재 Region의 최신 Amazon Linux 2023 AMI를 자동 조회
  * aurora\main.tf

## 104. 실습: 최신 Amazon Linux 2023 AMI 조회

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners = [ "amazon" ]

  filter {
    name = "name"
    values = [ "al2023-ami-2023.*-x86_64" ]
  }

  filter {
    name = "architecture"
    values = [ "x86_64" ]
  }

  filter {
    name = "virtualization-type"
    values = [ "hvm" ]
  }
}
```

## 105. 실습: STEP 15. Root에서 EC2 Child Module 호출

* Network Module에서 생성한 첫 번째 Public Subnet에 Aurora MySQL DB Client EC2를 생성
  * aurora\main.tf

## 106. 실습: EC2 Child Module 호출

```hcl
module "ec2" {
  source = "./modules/ec2"                           # EC2 Child Module 경로
  ami_id = data.aws_ami.al2023.id                   # 조회한 최신 Amazon Linux 2023 AMI ID 전달
  instance_type = var.instance_type                  # Root Module의 EC2 Instance Type 값 전달
  vpc_id = module.network.vpc_id                     # Network Module에서 생성한 VPC ID 전달
  subnet_id = module.network.public_subnets[0]       # 첫 번째 Public Subnet에 EC2 배치
  instance_name = var.instance_name                 # Root Module의 EC2 Name Tag 값 전달
  public_key_path = var.public_key_path           # SSH 접속에 사용할 Public Key 경로 전달
}
```

* data.aws\_ami.al2023.id
  * 자동 조회한 Amazon Linux 2023 AMI
* public\_subnets\[0]
  * 첫 번째 Public Subnet에 EC2 생성
* module.network.vpc\_id
  * Network Module에서 생성한 VPC 사용
  * aurora\outputs.tf

```hcl
output "public_dns" {
  description = "DB Client EC2 Public DNS"
  value = module.ec2.public_dns
}

output "public_ip" {
  description = "DB Client EC2 Public IP"
  value = module.ec2.public_ip
}
```

## 107. 실습: STEP 16. Aurora MySQL 변수 작성

* Aurora MySQL Cluster와 Writer / Reader Instance를 생성하기 위해 필요한 변수들을 Root Module에 추가
* Aurora에서는 일반 RDS MySQL처럼 Primary RDS와 Read Replica를 각각 aws\_db\_instance로 생성하지 않는다.
* Aurora Cluster를 먼저 생성하고 Cluster 내부에 Writer와 Reader Instance를 생성
  * aurora\modules\rds\variables.tf

```hcl
variable "vpc_id" {
  type = string
}

variable "private_subnets" {
  type = list(string)
}

variable "vpc_name" {
  type = string
}

variable "environment" {
  type = string
}

variable "allowed_cidr" {
  type = string
}

variable "aurora_cluster_identifier" {
  type = string
}

variable "aurora_engine_version" {
  type = string
}

variable "aurora_instance_class" {
  type = string
}

variable "db_name" {
  type = string
}

variable "db_username" {
  type = string
}

variable "db_password" {
  type      = string
  sensitive = true
}
```

## 108. 실습: STEP 17. Aurora Security Group 생성

* Aurora MySQL Cluster에 적용할 Security Group을 생성
* MySQL 기본 Port인 TCP 3306을 허용하여 같은 VPC 내부의 EC2에서 Aurora에 접근할 수 있도록
  * aurora\modules\rds\main.tf

## 109. 실습: Aurora MySQL Security Group

```hcl
resource "aws_security_group" "aurora_sg" {
  name = "aurora-mysql-security-group"
  vpc_id = var.vpc_id

  # MySQL
  ingress {
    from_port = 3306
    to_port = 3306
    protocol = "tcp"
    cidr_blocks = [ var.allowed_cidr ]
  }

  # 모든 Outbound 허용
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = [ "0.0.0.0/0" ]
  }

  tags = {
    Name = "aurora-mysql-security-group"
  }
}

   # aurora\modules\rds\outputs.tf
output "aurora_security_group_id" {
  description = "Aurora Security Group ID"
  value = aws_security_group.aurora_sg.id
}
```

* 설정 설명
* 3306
  * MySQL 기본 Port
* allowed\_cidr
  * 같은 VPC Network에서 Aurora 접근 허용
* vpc\_id
  * Aurora Security Group이 생성될 VPC

## 110. 실습: STEP 18. Aurora DB Subnet Group 생성

* Aurora가 사용할 Private Subnet 2개를 DB Subnet Group으로 묶는다.
* Aurora Writer와 Reader는 이 DB Subnet Group에 포함된 Private Subnet에 배치된다.
  * aurora\modules\rds\main.tf

## 111. 실습: Aurora DB Subnet Group

```hcl
resource "aws_db_subnet_group" "aurora_subnet_group" {
  name = "${var.vpc_name}-aurora-subnet-group"
  subnet_ids = var.private_subnets

  tags = {
    Name = "${var.vpc_name}-aurora-subnet-group"
  }
}
```

* 설정 설명
* subnet\_ids
  * Network Module에서 생성한 Private Subnet 전체 사용
* module.network.private\_subnets
  * 서로 다른 AZ의 Private Subnet List
* DB Subnet Group
  * Aurora DB Instance가 배치될 Network 영역 지정

## 112. 실습: STEP 19. Aurora MySQL Cluster 생성

* Aurora MySQL의 중심이 되는 DB Cluster를 생성
* Aurora에서는 먼저 Cluster를 생성하고 그 Cluster에 Writer와 Reader DB Instance를 추가
* Cluster에는 Database Engine, Database 이름, 관리자 계정, Backup, Network 등의 공통 설정을 정의
  * aurora\modules\rds\main.tf

## 113. 실습: Aurora MySQL Cluster

```hcl
resource "aws_rds_cluster" "aurora_mysql" {

  cluster_identifier = var.aurora_cluster_identifier# Aurora Cluster 이름
  engine = "aurora-mysql"# Aurora MySQL 사용
  engine_version = var.aurora_engine_version# Aurora MySQL Version
  database_name = var.db_name# 최초 생성할 Database
  master_username = var.db_username# 관리자 계정
  master_password = var.db_password# 관리자 Password

  # Private Subnet으로 구성된 DB Subnet Group
  db_subnet_group_name = aws_db_subnet_group.aurora_subnet_group.name
  vpc_security_group_ids = [ aws_security_group.aurora_sg.id ]# Aurora Security Group
  backup_retention_period = 7# 자동 Backup 7일 보관
  preferred_backup_window = "02:00-03:00"# Backup 시간
  preferred_maintenance_window = "sun:05:00-sun:06:00"# 유지보수 시간
  storage_encrypted = true# Storage 암호화
  deletion_protection = false# 실습에서는 삭제 보호 사용 안 함
  skip_final_snapshot = true# destroy 시 Final Snapshot 생성 안 함

  tags = {
    Name = "My-Aurora-MySQL"
    Environment = var.environment
  }
}
```

* 설정 설명
* cluster\_identifier
  * 생성되는 Aurora DB Cluster를 식별하기 위한 이름이다.
  * 예를 들어 값이 "my-aurora-cluster"이면 AWS에 해당 이름을 가진 Aurora Cluster가 생성된다.
  * Aurora Cluster 안에는 이후 Writer Instance와 Reader Instance가 연결된다.
* engine = "aurora-mysql"
  * 생성할 Database Engine 종류를 지정한다.
  * 일반 RDS MySQL에서는 주로 aws\_db\_instance를 사용하지만, Aurora에서는 aws\_rds\_cluster를 먼저 생성하고 그 Cluster 안에 aws\_rds\_cluster\_instance를 추가하는 구조를 사용한다.
* engine\_version
  * Aurora MySQL에서 사용할 Engine Version을 지정한다.
  * Cluster와 이후 생성하는 Writer/Reader Instance는 호환되는 동일한 Aurora MySQL Engine Version을 사용해야 한다.
* database\_name
  * Aurora Cluster가 처음 생성될 때 같이 생성할 Database 이름이다.
  * 예를 들어 값이 "mydatabase"이면 Aurora 생성 후 MySQL에 접속하여 SHOW DATABASES;를 실행했을 때 mydatabase를 확인할 수 있다.
  * Aurora Cluster 자체의 이름이 아니라 MySQL 내부에 생성되는 Database 이름이다.
* master\_username
  * Aurora MySQL에 접속할 Master 관리자 계정 이름이다.
  * 이후 EC2에서 Aurora에 접속할 때 다음과 같이 사용한다.
  * mysql -h -u admin -p
  * 여기서 admin에 해당하는 값이 master\_username이다.
* master\_password
  * Master 관리자 계정에서 사용할 Password이다.
  * Aurora MySQL에 최초 접속할 때 필요한 관리자 Password이며 master\_username과 함께 Database 관리 권한을 가진다.
* db\_subnet\_group\_name
  * Aurora Cluster가 어느 Network 영역에 배치될지 지정한다.
  * 앞에서 생성한 DB Subnet Group을 Aurora Cluster에 연결한다.
  * DB Subnet Group에는 서로 다른 Availability Zone에 위치한 Private Subnet 2개가 등록되어 있다.
  * 따라서 Aurora의 Writer와 Reader Instance가 Public Subnet이 아니라 Private Network 영역에 배치될 수 있다.
* vpc\_security\_group\_ids
  * Aurora Cluster에 적용할 Security Group을 지정한다.
  * Security Group은 Aurora에 어떤 Network Traffic이 접근할 수 있는지를 제어한다.
  * 현재 실습에서는 TCP 3306 Port를 허용한 aws\_security\_group.aurora\_sg를 연결한다.
  * 따라서 Security Group에서 허용한 Network만 Aurora MySQL Endpoint의 3306 Port로 접근할 수 있다.
* backup\_retention\_period = 7
  * Aurora의 자동 Backup을 며칠 동안 보관할지 지정한다. (값이 7이므로 자동 Backup을 7일 동안 보관)
  * 이 기간 내에서는 특정 시점으로 Database를 복구하는 Point-in-Time Recovery 기능을 사용할 수 있다.
  * 예를 들어 오늘 실수로 데이터를 삭제했다면 Backup 보존 기간 안의 이전 시점으로 복구할 수 있다.
* preferred\_backup\_window = "02:00-03:00"
  * Aurora의 자동 Backup 작업을 수행할 선호 시간대를 지정한다.
  * AWS RDS의 시간 설정은 UTC 기준이다.
  * 따라서 "02:00-03:00"은 매일 UTC 02:00 \~ 03:00 사이에 Backup 작업을 수행하도록 지정
  * 이 시간에 반드시 정확히 02:00에 시작하는 것이 아니라 지정된 Window 안에서 AWS가 Backup 작업을 수행한다.
* preferred\_maintenance\_window = "sun:05:00-sun:06:00"
  * Aurora Cluster의 유지보수 작업을 수행할 선호 시간대를 지정한다.
  * Engine Patch, System Update 등 AWS에서 필요한 유지보수 작업이 있을 경우 이 시간대를 사용한다.
  * "sun:05:00-sun:06:00"은 일요일 UTC 05:00 \~ 06:00를 의미한다.
  * 유지보수가 항상 매주 실행된다는 뜻은 아니며, 적용할 유지보수 작업이 있을 때 이 시간대를 우선 사용한다.
* storage\_encrypted = true
  * Aurora Cluster Storage에 저장되는 데이터를 암호화한다.
  * true로 설정하면 Database Storage가 암호화되어 저장된다.
  * Database Data뿐 아니라 Backup, Snapshot 등 Aurora Cluster와 관련된 저장 데이터도 암호화된 형태로 관리된다.
  * Database에 중요한 데이터를 저장할 경우 보안을 위해 사용하는 설정
* deletion\_protection = false
  * Aurora Cluster의 삭제 보호 기능을 사용할지 결정한다.
  * true이면 실수로 Aurora Cluster를 삭제하는 것을 방지할 수 있다.
  * 하지만 true 상태에서는 terraform destroy를 실행해도 Aurora Cluster 삭제가 제한될 수 있다.
  * 운영 환경에서는 실수로 Database를 삭제하지 않도록 true를 사용하는 경우가 많다.
* skip\_final\_snapshot = true
  * Aurora Cluster를 삭제할 때 마지막 Snapshot을 생성할지 결정한다.
  * true : Final Snapshot을 생성하지 않고 Cluster를 삭제한다.
  * false: Cluster를 삭제하기 전에 마지막 Snapshot을 생성한다.
  * 실습에서는 빠르게 Resource를 삭제하고 불필요한 Snapshot 비용이 발생하지 않도록 true를 사용한다.

## 114. 실습: STEP 20. Aurora Writer Instance 생성

* Aurora Cluster 내부에 실제 Database 작업을 처리할 Writer Instance를 생성
* Writer Instance는 읽기와 쓰기 작업을 모두 수행할 수 있다.
  * SELECT
  * INSERT
  * UPDATE
  * DELETE
  * CREATE TABLE
* Aurora Cluster에는 하나의 Writer가 존재
* Application에서 데이터를 저장하거나 수정하는 작업은 Writer Endpoint를 통해 처리
  * aurora\modules\rds\main.tf

## 115. 실습: Aurora Writer Instance

```hcl
resource "aws_rds_cluster_instance" "writer" {
  identifier = "${var.aurora_cluster_identifier}-writer"# Writer Instance 이름
  cluster_identifier = aws_rds_cluster.aurora_mysql.id# STEP 19에서 생성한 Aurora Cluster에 연결
  instance_class = var.aurora_instance_class# Aurora Instance 사양
  engine = aws_rds_cluster.aurora_mysql.engine# Cluster와 동일한 Engine 사용
  engine_version = aws_rds_cluster.aurora_mysql.engine_version# Cluster와 동일한 Engine Version 사용
  db_subnet_group_name = aws_db_subnet_group.aurora_subnet_group.name# Aurora DB Subnet Group
  publicly_accessible = false# Internet 직접 접근 금지
  promotion_tier = 0# Failover 우선순위

  tags = {
    Name = "My-Aurora-Writer"
    Environment = var.environment
  }
}
```

## 116. 실습: STEP 21. Aurora Reader Instance 생성

* Aurora Cluster에 읽기 작업을 처리할 Reader Instance를 생성
* Reader는 Aurora Replica 역할을
* 주로 SELECT와 같은 조회 Traffic을 처리
* Aurora에서는 일반 RDS MySQL Read Replica처럼 replicate\_source\_db를 지정하지 않는다.
* Writer와 Reader가 같은 Aurora Cluster에 포함되며 Aurora가 Storage와 Replication을 관리
  * aurora\modules\rds\main.tf

## 117. 실습: Aurora Reader Instance

```hcl
resource "aws_rds_cluster_instance" "reader" {
  identifier = "${var.aurora_cluster_identifier}-reader"# Reader Instance 이름
  cluster_identifier = aws_rds_cluster.aurora_mysql.id# Writer와 동일한 Aurora Cluster에 연결
  instance_class = var.aurora_instance_class# Reader Instance 사양

  engine = aws_rds_cluster.aurora_mysql.engine# Cluster와 동일한 Engine
  engine_version = aws_rds_cluster.aurora_mysql.engine_version# Cluster와 동일한 Engine Version
  db_subnet_group_name = aws_db_subnet_group.aurora_subnet_group.name# 동일한 DB Subnet Group 사용

  publicly_accessible = false# Internet 직접 접근 금지
  promotion_tier = 1# Writer 장애 시 Reader 승격 우선순위

  tags = {
    Name = "My-Aurora-Reader"
    Environment = var.environment
  }
}
```

* 설정 설명
* Reader
  * Aurora Cluster의 읽기 전용 Replica
* SELECT
  * 조회 Traffic 처리
* cluster\_identifier
  * Writer와 동일한 Aurora Cluster 사용
* promotion\_tier = 1
  * Writer 장애 발생 시 Reader가 Writer로 승격될 수 있음

## 118. 실습: STEP 22. Aurora Endpoint 출력

* Aurora에는 일반 RDS와 달리 여러 종류의 Endpoint가 있다.
* Cluster Endpoint는 현재 Writer Instance로 연결된다.
* Reader Endpoint는 Aurora Reader Instance로 읽기 요청을 분산
  * aurora\modules\rds\outputs.tf

```hcl
output "aurora_writer_endpoint" {
  description = "Aurora Writer Endpoint"
  value       = aws_rds_cluster.aurora_mysql.endpoint
}

output "aurora_reader_endpoint" {
  description = "Aurora Reader Endpoint"
  value       = aws_rds_cluster.aurora_mysql.reader_endpoint
}

output "aurora_port" {
  description = "Aurora MySQL Port"
  value       = aws_rds_cluster.aurora_mysql.port
}
```

* aurora\_writer\_endpoint
  * INSERT / UPDATE / DELETE / SELECT 작업에 사용
* aurora\_reader\_endpoint
  * SELECT 등의 읽기 작업에 사용
* aurora\_port
  * Aurora MySQL 기본 Port인 3306 출력

STEP 22-1. Root Module에 Aurora 변수 추가

* Child rds Module에 전달할 Aurora 관련 변수를 Root Module에 작성
  * aurora\variables.tf

```hcl
variable "allowed_cidr" {
  description = "Aurora MySQL 접근을 허용할 CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "aurora_cluster_identifier" {
  description = "Aurora Cluster Identifier"
  type        = string
  default     = "my-aurora-cluster"
}

variable "aurora_engine_version" {
  description = "Aurora MySQL Engine Version"
  type        = string
  default     = "8.0.mysql_aurora.3.10.5"
}

variable "aurora_instance_class" {
  description = "Aurora DB Instance Class"
  type        = string
  default     = "db.t3.medium"
}

variable "db_name" {
  description = "최초 생성할 Database 이름"
  type        = string
  default     = "mydatabase"
}

variable "db_username" {
  description = "Aurora Master Username"
  type        = string
  default     = "admin"
}

variable "db_password" {
  description = "Aurora Master Password"
  type        = string
  sensitive   = true
  default     = "admin1234"
}

STEP 22-2. Root에서 RDS Child Module 호출
```

* Network Module에서 생성한 VPC와 Private Subnet을 RDS Module에 전달
* Aurora Cluster는 Private Subnet에 생성
  * aurora\main.tf

## 119. 실습: RDS Child Module 호출

```hcl
module "rds" {
  source = "./modules/rds"

  vpc_id          = module.network.vpc_id
  private_subnets = module.network.private_subnets

  vpc_name    = var.vpc_name
  environment = var.environment

  allowed_cidr = var.allowed_cidr

  aurora_cluster_identifier = var.aurora_cluster_identifier
  aurora_engine_version     = var.aurora_engine_version
  aurora_instance_class     = var.aurora_instance_class

  db_name     = var.db_name
  db_username = var.db_username
  db_password = var.db_password
}

STEP 22-3. Root에서 Aurora Output 출력
```

* Child rds Module의 Output을 Root Module에서도 확인할 수 있도록 출력
  * aurora\outputs.tf

```hcl
output "aurora_security_group_id" {
  description = "Aurora Security Group ID"
  value       = module.rds.aurora_security_group_id
}

output "aurora_writer_endpoint" {
  description = "Aurora Writer Endpoint"
  value       = module.rds.aurora_writer_endpoint
}

output "aurora_reader_endpoint" {
  description = "Aurora Reader Endpoint"
  value       = module.rds.aurora_reader_endpoint
}

output "aurora_port" {
  description = "Aurora MySQL Port"
  value       = module.rds.aurora_port
}

   # aurora\terraform.tfvars
# AWS
aws_region  = "ap-northeast-2"
aws_profile = "my-profile"
environment = "Production"

# Network
vpc_name = "my-vpc"
vpc_cidr = "10.0.0.0/16"

public_subnets = [  "10.0.1.0/24", "10.0.2.0/24" ]
private_subnets = [ "10.0.3.0/24", "10.0.4.0/24" ]
availability_zones = [  "ap-northeast-2a",  "ap-northeast-2b" ]

# EC2
instance_type   = "t3.micro"
instance_name   = "aurora-db-client"
public_key_path = "~/.ssh/my-key.pub"

# Aurora
allowed_cidr              = "10.0.0.0/16"
aurora_cluster_identifier = "my-aurora-cluster"
aurora_engine_version     = "8.0.mysql_aurora.3.10.5"
aurora_instance_class     = "db.t3.medium"

# Database
db_name     = "mydatabase"
db_username = "admin"
db_password = "admin1234"
```

## 120. 실습: STEP 23. Terraform 전체 실행

* 작성한 Terraform 코드를 초기화하고 실행 계획을 확인한 뒤 실제 AWS Resource를 생성

```
[PowerShell]
PS C:\my-terraform\aurora-mysql-service> terraform init
PS C:\my-terraform\aurora-mysql-service> terraform plan
PS C:\my-terraform\aurora-mysql-service> terraform apply
```

## 121. 실습: STEP 24. Terraform Output 확인

* Terraform으로 생성된 VPC, Subnet, EC2, Aurora Writer Endpoint와 Reader Endpoint를 확인

```
[PowerShell]
PS C:\my-terraform\aurora-mysql-service> terraform output
vpc_id
public_subnets
private_subnets
public_dns
public_ip
aurora_security_group_id
aurora_writer_endpoint
aurora_reader_endpoint
aurora_port

# DB 접속

PS C:\Users\soldesk>
ssh -i $home\.ssh\my-key ec2-user@ec2-43-201-97-196.ap-northeast-2.compute.amazonaws.com
   ,     #_
   ~\_  ####_
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   Amazon Linux 2023 (ECS Optimized)
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'

For documentation, visit http://aws.amazon.com/documentation/ecs
[ec2-user@ip-10-0-1-19 ~]$

# mysql client를 구성

# 1. MySQL 클라이언트 설치
[ec2-user@ip-10-0-1-19 ~]# dnf install -y mariadb105

# 2. 설치 확인
[ec2-user@ip-10-0-1-19 ~]# mysql --version

# mysql을 사용해 rds로 접속
# 3. MySQL 클라이언트 설치
[ec2-user@ip-10-0-1-19 ~]$ mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p

[ec2-user@ip-10-0-1-19 ~]#
mysql -h <rds-endpoint> -u admin -p
Enter password: admin1234
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 38
Server version: 8.0.44 Source distribution
Copyright (c) 2000, 2026, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema|
| mydatabase         |
| mysql              |
| performance_schema|
| sys                |
+--------------------+
5 rows in set (0.02 sec)
```

* 워크밴치 Connection Name: Terraform-db Connection Method: Standard TCP/IP over SSH

SSH Hostname: 43.201.97.196# EC2 퍼블릭 IP SSH Username: ec2-user# EC2 접속 계정 SSH Password: 생략

```
SSH Key File : C:\Users\soldesk\.ssh\my-key# 키파일 위치

# RDS 엔드포인트 주소
MySQL Hostname: <rds-endpoint>
MySQL Server Port: 3306
Username: admin
Password: admin1234
```

![Password: admin1234 화면](<../.gitbook/assets/7 (2).png>)

![Password: admin1234 화면](<../.gitbook/assets/8 (2).png>)

![Password: admin1234 화면](<../.gitbook/assets/9 (2).png>)

## 122. 실습: STEP 31. Table 및 Data 생성

```
[설명]
```

* Aurora Writer에서 Table을 생성하고 Data를 입력
* Writer는 읽기와 쓰기 작업을 모두 처리할 수 있다.

```
[MySQL]
USE mydatabase;

CREATE TABLE students (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  course VARCHAR(50)
);

INSERT INTO students (name, course)
VALUES
('student1', 'AWS'),
('student2', 'Terraform'),
('student3', 'Aurora');

SELECT * FROM students;

[확인 예]
+------+-----------+-----------+
| id     | name      | course     |
+------+-----------+-----------+
|  1    | student1   | AWS       |
|  2    | student2   | Terraform  |
|  3    | student3   | Aurora     |
+------+-----------+-----------+
```

## 123. 실습: STEP 32. Aurora Reader Endpoint 확인

* Aurora Reader Endpoint를 확인
* Reader Endpoint는 Aurora Cluster의 Reader Instance로 읽기 요청을 전달

```
[PowerShell]
PS C:\my-terraform\aurora-mysql-service> terraform output -raw aurora_reader_endpoint

[예]
my-aurora-cluster.cluster-ro-xxxxxxxx.ap-northeast-2.rds.amazonaws.com
```

## 124. 실습: STEP 33. Aurora Reader 접속

* EC2에서 Reader Endpoint로 접속
* Writer에서 입력한 Data가 Reader에서도 정상적으로 조회되는지 확인

\[EC2에서 실행]

```
mysql -h <AURORA-READER-ENDPOINT> -P 3306 -u admin -p

[접속 후]
USE mydatabase;

SELECT * FROM students;

[확인]
+----+-------+-------+
| id     | name      | course     |
+----+-------+-------+
|  1    | student1   | AWS       |
|  2    | student2   | Terraform  |
|  3    | student3   | Aurora     |
+----+-------+-------+
```

* Writer에서 입력한 Data가 Reader에서도 조회되면 Aurora Replication이 정상적으로 동작하고 있는 것이다.

## 125. 실습: STEP 34. Resource 삭제

* 실습 완료 후 비용 발생을 방지하기 위해 Terraform으로 생성한 AWS Resource를 삭제

```
[PowerShell]
PS C:\my-terraform\aurora-mysql-service> terraform destroy

[확인]
Plan: 0 to add, 0 to change, XX to destroy.
[삭제 승인]
Enter a value: yes
```

* Aurora Writer
* Aurora Reader
* Aurora Cluster
* DB Subnet Group
* Security Group
* EC2
* Key Pair
* Subnet
* Route Table
* Internet Gateway
* VPC

순서에 따라 Terraform이 의존성을 확인하여 Resource를 삭제
