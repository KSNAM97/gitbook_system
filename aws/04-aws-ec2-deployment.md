# AWS EC2 - 배포

## 1. EC2(Elastic Compute Cloud) 기초

- EC2는 안전하고 크기 조정이 가능한 컴퓨팅 파워를 클라우드에서 제공하는 웹 서비스이다.
- 개발자가 더 쉽게 웹 규모의 클라우드 컴퓨팅 작업을 할 수 있도록 설계되었다.
- 클라우드: 빌려쓰기
- 클라우드 컴퓨팅: 컴퓨팅 빌려쓰기
- EC2: 컴퓨팅을 빌려쓰는 AWS 서비스

특징:

- **초 단위 온디맨드 가격 모델**
  - 온디맨드 모델에서는 가격이 초 단위로 결정
  - 서비스 요금을 미리 약정하거나 선입금이 필요 없음
  - 사용한 만큼만 과금되어 비용 효율적 운영 가능
- **빠른 구축 속도와 확장성**
  - 몇 분이면 전 세계에 인스턴스 수백여대를 구축 가능
  - 트래픽 증가나 감소에 따라 유연하게 인프라 확장 및 축소 가능
- **다양한 구성방법 지원**
  - 머신러닝, 웹서버, 게임서버, 이미지처리 등 다양한 용도에 최적화된 서버 구성 가능
  - 다양한 과금 모델 사용 가능
  - 요구사항에 따라 인스턴스 타입, 스토리지, 네트워크 환경을 자유롭게 선택 가능
- **여러 AWS 서비스와 연동**
  - 오토스케일링, Elastic Load Balancer(ELB), CloudWatch, RDS, S3 등 다른 AWS 서비스와 원활하게 통합 가능

## 2. EC2 인스턴스

- EC2에서 컴퓨팅을 담당
- 다양한 유형과 크기로 구성 가능
- 저장을 담당하는 EBS와 네트워크로 연결
- 필요에 따라 CPU, 메모리, 스토리지, 네트워크 성능을 조정 가능

저장 방법에 따라 두 가지로 분류:

- **EBS 연동**: 인스턴스를 종료해도 데이터가 유지됨
- **인스턴스 스토어**: 인스턴스 종료 시 데이터가 삭제되며, 고속 임시 저장소 용도로 사용

하나의 가용영역(AZ)에 존재:

- 특정 AZ 내에서 실행되며, 고가용성을 위해 여러 AZ에 분산 배치 가능
- 다른 AZ로의 이전을 위해 스냅샷 생성 후 복원 가능

예시 - 총 $10로 EC2를 사용할 때 용도별 자원 배분:

| 용도 | CPU | RAM | 그래픽카드 |
|---|---|---|---|
| CPU가 중요한 알고리즘 처리 | $7 | $2 | $1 |
| 메모리 DB | $2 | $7 | $1 |
| 비트코인 채굴 | $4 | $1 | $5 |

이렇게 필요에 따라 각기 다른 인스턴스를 만들 수 있는데 이것을 **인스턴스 유형**이라고 한다.

## 3. 인스턴스 유형(패밀리)

- 인스턴스의 역할에 따라 CPU, 메모리, 스토리지, 네트워크 등을 조합한 구성
- 각 인스턴스 유형별로 사용 목적에 따라 최적화 (예: 메모리 위주, CPU 위주, 그래픽카드 위주 등)
- 유형별로 이름 존재 (예: t유형, m유형, inf유형 등) - 같은 유형의 인스턴스들을 **인스턴스 패밀리**라 부름
- 타입별 세대별로 숫자 부여 (예: m5 = m 인스턴스의 5번째 세대)
- 아키텍처 및 프로세서/추가기술에 따라 접미사 (예: c7gn = c 인스턴스 중 AWS Graviton 프로세서를 사용(g) + Network Optimized(n))

| 타입 | 범주 | 설명 | 예시 |
|---|---|---|---|
| t | 범용 | 저렴한 범용 | 웹서버, DB |
| m | 범용 | 범용 | 애플리케이션 서버 |
| A1 | 범용 | ARM 기반 | ARM 기반 테스트 및 워크로드/웹서버 등 |
| Mac | 범용 | 맥 기반 | macOS 기반 빌드/테스트 |
| c | 컴퓨팅 최적화 | 컴퓨팅 최적화 | CPU 성능이 중요한 어플리케이션/DB |
| f | 컴퓨팅 최적화 | 하드웨어 가속 | 유전자 연구, 금융 분석, 빅데이터 분석 |
| Inf | 컴퓨팅 최적화 | 머신러닝 | 머신러닝 |
| g | 컴퓨팅 최적화 | 그래픽 최적화 | 3D 모델링/인코딩 |
| r | 메모리 최적화 | 메모리 최적화 | 메모리 성능이 중요한 어플리케이션/DB |
| x | 메모리 최적화 | 메모리 최적화 | Spark |
| p | 메모리 최적화 | 그래픽 최적화 | 머신러닝, 비디오 코딩 |
| z | 메모리 최적화 | 고주파수 싱글 스레드 워크로드 | EDA 회로 설계 예측 |
| u-6tb1 | 메모리 최적화 | 대용량 메모리 인스턴스 | 가상화, 인메모리 데이터베이스 |
| h | 저장 최적화 | 디스크 쓰루풋 최적화 | 하둡, 멀티미디어 뉴스 |
| i | 저장 최적화 | 디스크 속도 최적화 | NoSQL, 데이터웨어하우스 |
| d | 저장 최적화 | 디스크 최적화 | 파일 서버, 데이터웨어하우스/하둡 |

제품 세부 정보 (t2 패밀리 예시):

| 이름 | vCPU | RAM(GiB) | CPU 크레딧/시간 | 온디맨드 요금/시간 | 1년 약정 예약 인스턴스 실질 시간 |
|---|---|---|---|---|---|
| t2.nano | 1 | 0.5 | 3 | 0.0058 USD | 0.003 USD |
| t2.micro | 1 | 1.0 | 6 | 0.0116 USD | 0.007 USD |
| t2.small | 1 | 2.0 | 12 | 0.023 USD | 0.014 USD |
| t2.medium | 2 | 4.0 | 24 | 0.0464 USD | 0.031 USD |
| t2.large | 2 | 8.0 | 36 | 0.0928 USD | 0.055 USD |
| t2.xlarge | 4 | 16.0 | 54 | 0.1856 USD | 0.110 USD |
| t2.2xlarge | 8 | 32.0 | 81 | 0.3712 USD | 0.219 USD |

## 4. Amazon EBS(Elastic Block Store)

EC2 인스턴스 저장에는 크게 EBS(Elastic Block Store)와 인스턴스 스토어(Instance Store) 두 가지 방식이 있다.

| 구분 | EBS (Elastic Block Store) | 인스턴스 스토어 (Instance Store) |
|---|---|---|
| 저장 위치 | 네트워크 기반 블록 스토리지 (EC2와 네트워크로 연결) | EC2 호스트 서버의 로컬 디스크 |
| 데이터 지속성 | 인스턴스 종료/중지 후에도 데이터 유지 | 인스턴스 중지·종료 시 데이터 삭제 |
| 크기 변경 | 실행 중에도 확장 가능 | 불가능 (인스턴스 생성 시 고정) |
| 스냅샷 | S3에 스냅샷 생성 가능 | 불가능 |
| 사용 가능 인스턴스 | 대부분의 EC2 인스턴스 유형 | 일부 인스턴스 유형만 지원 |

- Amazon EBS는 AWS 클라우드의 Amazon EC2 인스턴스에서 사용할 영구 블록 스토리지 볼륨을 제공한다.
- 각 Amazon EBS 볼륨은 가용 영역(AZ) 내에서 자동으로 복제되어 하드웨어 장애로부터 데이터를 보호하며, 고가용성과 내구성을 제공한다.

![EBS vs 인스턴스 스토어 개념 비교](assets/ebs-vs-instance-store-compare.jpeg)

EBS란?

- AWS에서 제공하는 가상 하드드라이브로, EC2 인스턴스에 연결해 사용할 수 있는 영구 블록 스토리지
- 인스턴스와 독립적으로 존재하므로, 인스턴스를 중지하거나 종료해도 데이터 유지 가능
- 단, 루트 볼륨으로 사용 시 인스턴스 종료와 함께 삭제되지만, 설정을 통해 EBS만 존속 가능
- 용량과 성능을 자유롭게 조정 가능 (확장/축소 및 IOPS 조정 가능)
- 스토리지 유형 선택 가능 - SSD 기반(고성능 IOPS 제공) / HDD 기반(저비용 대용량 저장)
- EBS Multi Attach 기능을 통해 하나의 EBS를 여러 EC2 인스턴스에 장착 가능 (특정 유형에 한함)
- EC2 인스턴스와 같은 가용영역(AZ) 내에서만 존재하며, 다른 AZ로 직접 이동 불가 (스냅샷을 통해 복제 가능)
- 가용영역 내 자동 분산 저장으로 99.999%의 가용성을 목표로 설계됨
- 인스턴스와 EBS는 네트워크로 연결되어있기 때문에 다른 인스턴스로 교체가 가능하다.

![EBS와 EC2 인스턴스의 네트워크 연결 구조](assets/ebs-network-connection.jpeg)

EBS의 유형:

- **범용(General Purpose 또는 GP) - SSD**: 가장 일반적으로 사용되는 SSD 타입. 가격과 성능의 균형이 좋으며, 대부분의 워크로드에 적합 (예: 웹 서버, 개발/테스트 환경, 소규모 데이터베이스)
- **프로비저닝 된 IOPS(Provisioned IOPS 또는 io) - SSD**: 높은 IOPS(입출력 속도)가 필요한 워크로드에 최적화. 대규모 트랜잭션 처리 데이터베이스나 미션 크리티컬 애플리케이션에 적합 (예: Oracle, SQL Server, MySQL 등 OLTP 환경)
- **쓰루풋 최적화(Throughput Optimized HDD 또는 st) - HDD**: 대규모 순차적 데이터 처리에 유리한 HDD 타입 (예: 빅데이터 분석, 데이터 웨어하우스, 로그 처리)
- **콜드 HDD(SC) - HDD**: 접근 빈도가 낮은 데이터를 장기 저장할 때 사용. 저비용이 장점이지만 성능은 낮음 (예: 백업, 아카이브 데이터)
- **마그네틱(Standard) - HDD**: 구형 표준 HDD 타입으로, 현재는 잘 사용되지 않음. 저비용이지만 성능이 낮아 특정 레거시 환경에서만 사용

### Snapshot

- 스냅샷: EBS의 특정 시점을 저장한 이미지. 이후 EBS로 다시 복구 가능하며, EBS의 백업 용도로 활용 가능
- 증분식: 바뀐 부분만 저장 (예: 100GB 볼륨의 스냅샷을 5번 찍어도 500GB가 아닌 100GB + 4번의 변경 부분만 저장, 비용 역시 최적화)
- S3에 저장: 99.999999999% 내구성
- 자동화 생성 가능: Data Lifecycle Manager / AWS Backup 등을 통해 자동화 가능

## 5. AMI(Amazon Machine Image)

- AMI: EC2 인스턴스를 실행하기 위해 필요한 정보를 모아 둔 템플릿
- 운영체제(OS), 애플리케이션 서버, 애플리케이션, 설정 파일 등이 포함됨
- 동일한 환경의 EC2 인스턴스를 여러 개 생성할 때 사용

구성 요소:

- 1개 이상의 EBS 스냅샷 (루트 볼륨 포함)
- 사용 권한 (어떤 AWS 계정이 사용할 수 있는지 정의)
- 블록 디바이스 매핑 (EC2 인스턴스를 위한 볼륨 정보 - EBS 용량, 개수, 연결 위치 등)

특징:

- 필요에 따라 Private으로 설정하거나 Public으로 공개 가능
- 사용자 지정(Custom AMI) 생성 가능 → 특정 설정이나 애플리케이션이 사전 설치된 상태로 배포 가능
- AWS Marketplace에서 제공하는 다양한 AMI 사용 가능 (상용/무료 포함)
- AMI를 이용하면 배포 시간 단축, 환경 표준화, 확장성 향상 가능

### AMI를 사용한 인스턴스 생성 실습 절차

1. Amazon EC2 프로비전 (t3.micro 타입 - 프리티어)
2. AMI 선택 (Amazon Linux)
3. 보안그룹 설정 (80, 22 포트 허용)
4. 아파치 설치 및 설정
5. 웹 서버 확인
6. AMI 생성
7. 기존 인스턴스 종료
8. AMI로부터 새로운 EC2 인스턴스 생성
9. 신규 인스턴스 확인 및 종료

## 6. EC2 요금 모델

> AWS 비용 백서 한줄 요약: "미리 내지 않고, 사용한 만큼 내고, 많이 쓸수록 적게 내고, 예약할수록 더 적게 낸다"

AWS의 가격 정책:

- "미리 내지 않고, 사용한 만큼 내고" - 미리 예치금 등이 없고 사용한 만큼(On-Demand)만 요금 지불
- "많이 쓸수록 적게 내고" - 사용하면 사용할수록 단위 가격이 더 저렴해짐
- "예약할수록 더 적게 낸다" - 미리 요금을 낼 필요는 없지만, 미리 예약하면 더 많이 저렴함
- 즉 "OnDemand는 유용하지만 비싸다" (약정을 통해 많은 할인 혜택 적용 가능)

EC2의 요금 구성:

- 인스턴스 요금
- 데이터 전송
- Public IPv4 (ELB, CloudWatch, EBS 등)

EC2 요금 모델 종류:

- **On-Demand**: 사용한 시간 만큼만 요금 지불
- **Spot Instances**: 남는 인스턴스를 저렴하게 사용
- **Reserved Instances**: 인스턴스 사용 기간을 약정
- **Dedicated**: 물리적인 전용 인스턴스 임대 (Dedicated Instance/Dedicated Host)
- **Savings Plan**: AWS의 컴퓨팅 사용량을 약정

### On-Demand

- 실행하는 인스턴스에 따라 초당 혹은 시간당 컴퓨팅 파워로 측정된 가격을 지불
- 약정 필요 없음
- 수요 예측이 힘들거나 유연하게 EC2를 사용하고 싶을 때
- 개발용 또는 테스트용도로 사용하고 싶을 때

### 예약 인스턴스(Reserved Instances)

- EC2 인스턴스를 일정 기간 약정하여 요금을 할인 받는 방식
- 온디맨드 EC2 사용 요금을 할인 받는 방식으로 적용
- 할인 받고 싶은 EC2 인스턴스와 같은 리전, 유형 구매 필요
- 약정 기간이 길수록 더 큰 할인율 적용 (수요 예측이 가능할 때 사용한다)
- 1년 혹은 3년 선택 가능, 최대 72% 저렴

### Savings Plan

- 컴퓨팅 파워의 사용량을 일정 기간(1년 또는 3년) 약정하여 할인받는 요금 모델
- 온디맨드 대비 최대 72%까지 저렴
- 약정은 시간당 일정 금액(예: $10/hour) 형태로 설정되며, 약정한 사용량 이내에서는 할인이 적용되고, 초과분은 온디맨드 요금으로 청구

주요 종류:

- **Compute Savings Plans**: 특정 인스턴스 유형이나 리전에 종속되지 않음. EC2, Lambda, Fargate 등 다양한 서비스에서 유연하게 사용 가능
- **EC2 Instance Savings Plans**: 특정 리전과 인스턴스 패밀리를 지정하여 약정. 해당 패밀리 내에서 인스턴스 크기 변경 가능 (예: m5.large → m5.xlarge). 특정 워크로드에 장기적으로 동일 인스턴스를 사용할 때 유리

### 스팟 인스턴스(Spot Instances)

- AWS에서 사용하지 않고 남아 있는 컴퓨팅 용량을 경매 방식으로 매우 저렴하게 제공하는 인스턴스 요금 모델
- 가용영역(AZ)별, 인스턴스 유형별로 별도의 풀(Pool)로 관리

비용 절감 효과:

- 온디맨드 대비 최대 90%까지 절약 가능
- 가격은 수요와 공급 상황에 따라 실시간 변동하지만, 항상 On-Demand 가격 이하를 보장

사용 시 주의점:

- AWS가 해당 리소스를 회수해야 하는 경우, 인스턴스가 예고 없이 종료될 수 있음 (Spot Instance Interruption)
- 종료 2분 전에 알림이 오며, 이 시간을 활용해 작업을 저장하거나 종료 절차 수행 가능

적합한 사용 사례:

- 중단되어도 무방한 작업 (예: 대규모 데이터 분석, 머신러닝 학습, 분산 처리 작업)
- 유연한 스케줄이 가능한 백그라운드 작업
- 단기적인 대규모 처리량이 필요한 경우

### 전용 인스턴스(Dedicated Instance)

- AWS가 제공하는 물리적 서버를 다른 고객과 공유하지 않고 특정 고객 전용으로 할당하여 EC2 인스턴스를 실행하는 방식
- 인스턴스/호스트 단위로 격리된 환경 제공
- 물리적 하드웨어는 단일 고객 전용이지만, 그 위에서 여러 개의 가상 CPU(vCPU) 인스턴스를 실행 가능

![전용 인스턴스의 물리 CPU와 vCPU 구성](assets/dedicated-instance-vcpu-structure.jpeg)

특징:

- **라이선스 이슈 해결**: 특정 소프트웨어 라이선스(예: Oracle, Windows Server)가 물리적 하드웨어 단위로 계산될 때 유리
- **성능 안정성**: 다른 고객의 워크로드 영향 없이 일관된 성능 보장
- **보안/규제 준수**: 금융, 의료, 공공기관 등 물리적 격리를 요구하는 규제 환경에서 사용 가능

활용 사례: 하드웨어 종속성이 있는 레거시 시스템 / 고성능 연산을 위한 전용 환경 / 라이선스 비용 최적화를 위한 배포

## 7. 보안그룹

- EC2 인스턴스에 대한 인바운드(외부에서 인스턴스로 들어오는 트래픽) 및 아웃바운드(인스턴스에서 외부로 나가는 트래픽)를 제어하는 가상 방화벽 역할
- 보안 그룹은 상태 저장(Stateful) 방식으로 동작하여, 인바운드에서 허용한 연결은 아웃바운드에서 자동 허용됨
- 네트워크 ACL과 달리 보안 그룹은 인스턴스 단위에서 적용됨

Port 허용:

- 기본적으로 모든 포트는 비활성화되어 있어, 명시적으로 허용 규칙을 추가해야 트래픽 통과 가능
- 선택적으로 트래픽이 지나갈 수 있는 Port(예: 22, 80, 443)와 Source(IP 주소, CIDR, 다른 보안 그룹)를 설정 가능
- Deny(차단) 규칙은 불가능하며, 허용(Allow) 규칙만 설정 가능

인스턴스 단위:

- 하나의 인스턴스에 하나 이상의 보안 그룹을 설정 가능하여, 복합적인 보안 규칙 적용 가능
- 인스턴스에 여러 보안 그룹이 적용될 경우 모든 보안 그룹의 허용 규칙이 합산되어 적용됨 (가장 개방적인 규칙이 우선)

![보안 그룹별로 허용된 포트 구성 예시](assets/security-group-example.jpeg)

![사용자 요청이 각 보안 그룹의 허용 포트를 통해 전달되는 흐름](assets/security-group-port-flow.jpeg)

## 8. EC2 접속 방법

EC2로 접속하는 방법은 여러 가지가 있지만 가장 대표적인 방법 3가지: SSH 연결(putty) / EC2 인스턴스 연결 / EC2 직렬 콘솔

### SSH 연결

| 구분 | 내용 |
|---|---|
| 연결 방법 | SSH 연결 |
| 동작방식 | SSH |
| 연결 인증 방식 | SSH 키 페어로 인증 |
| 감사 방법 | 직접 로깅 혹은 기타 3rd Party 애플리케이션 필요 |
| 연결 요구사항 | 인터넷 연결 필요(Security Group, Public Subnet, Bastion Host 등) |
| 주요 기능 | 기본적인 SSH 통신 |

SSH 키 페어:

- AWS EC2 인스턴스에 SSH 또는 FTP로 접속할 때 사용하는 보안 자격 증명 집합
- 프라이빗 키(Private Key)와 퍼블릭 키(Public Key)로 구성
- 퍼블릭 키: AWS에 저장되어 인스턴스와 연동
- 프라이빗 키: 로컬 PC에 저장, 실제 접속 시 인증에 사용
- 다시 발급 불가 (프라이빗 키를 분실하면 복구 불가능)
- 복구하려면 해당 인스턴스의 EBS 볼륨을 분리하여 다른 인스턴스에 연결 후 authorized_keys 파일을 수정하거나 또는 스냅샷 생성(새 인스턴스에 복원 방식으로 재구성 필요)
- 리전 단위 관리 - 키 페어는 생성한 리전에서만 유효, 다른 리전에서 사용하려면 키 페어를 export/import 해야 함
- 프라이빗 키는 생성 시 1회만 다운로드 가능 (반드시 안전한 위치에 백업)
- 키 분실 시 AWS는 재발급 불가하므로, 키 관리 정책을 사전에 수립해야 함

AMI별 기본 유저 이름:

| AMI | 기본 유저 이름 |
|---|---|
| Amazon Linux | ec2-user |
| CentOS | centos or ec2-user |
| Debian | admin |
| Fedora | fedora or ec2-user |
| RHEL | ec2-user or root |
| SUSE | ec2-user or root |
| Ubuntu | ubuntu |
| Oracle Linux | ec2-user |
| Bitnami | bitnami |

### EC2 인스턴스 연결

| 구분 | 내용 |
|---|---|
| 연결 방법 | EC2 인스턴스 연결 |
| 동작방식 | 임시 SSH키를 생성해서 EC2로 밀어 넣어 연결하는 방식 |
| 연결 인증 방식 | IAM 인증 |
| 감사 방법 | 연결 기록만 감사 가능(CloudTrail) |
| 연결 요구사항 | 인터넷 연결 필요, EC2-instance-connect 에이전트 설치 필요 |
| 주요 기능 | 기본적인 SSH 통신 |

### EC2 직렬 콘솔

| 구분 | 내용 |
|---|---|
| 연결 방법 | EC2 직렬 콘솔 |
| 동작방식 | EC2 시리얼 포트로 연결 |
| 연결 인증 방식 | IAM 인증 + Root Password |
| 감사 방법 | 연결 기록만 감사 가능(CloudTrail) |
| 연결 요구사항 | 계정 단위 활성화, OS Password 설정, 특정 인스턴스 타입/리전에서만 사용 가능 (예: t2 시리즈 사용 불가능) |
| 주요 기능 | 직접 컴퓨터에 모니터와 키보드를 붙인 것 같이 동작, 부팅 및 네트워크 문제 해결 |

EC2로 SSH 접속, FTP 접속에 쓰이는 도구:

- FileZilla (FTP): https://filezilla-project.org/
- PuTTY (SSH): https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html

### 키 파일 형식: PuTTYgen / PEM / PPK

**PuTTYgen(PUTTYGEN.EXE)**

- PuTTY Key Generator의 약자
- SSH 접속에 사용할 공개키(Public Key)와 개인키(Private Key)를 생성하는 프로그램
- AWS에서 다운로드한 .pem 형식의 프라이빗 키를 PuTTY에서 사용할 수 있는 .ppk 형식으로 변환할 때 사용
- PuTTY는 기본적으로 .ppk 형식의 키 파일을 사용
- AWS EC2에서 키 페어 생성 시 다운로드한 .pem 파일을 PuTTYgen으로 불러온 후 .ppk 파일로 저장
- 변환된 .ppk 파일은 PuTTY에서 EC2 SSH 접속 시 인증키로 사용
- 주요 기능: SSH 키 생성 / .pem 키 불러오기 / .ppk 형식으로 변환 및 저장 / Public Key 확인 / Private Key 관리

**PEM(.pem)**

- Privacy Enhanced Mail 형식에서 시작된 텍스트 기반 인증서/키 저장 형식
- AWS EC2에서 키 페어를 생성할 때 프라이빗 키를 .pem 파일로 다운로드할 수 있음
- OpenSSH, Linux, macOS 등에서 SSH 접속 시 주로 사용
- 예: `ssh -i mykey.pem ec2-user@EC2_PUBLIC_IP`
- 파일 내부는 Base64 형태의 텍스트로 저장됨

**PPK(.ppk)**

- PuTTY Private Key의 약자
- PuTTY 프로그램에서 사용하는 프라이빗 키 파일 형식
- Windows에서 PuTTY로 SSH 접속할 때 주로 사용
- AWS에서 받은 .pem 파일을 PuTTYgen을 이용하여 .ppk로 변환해서 사용 가능

### 직렬 콘솔

- EC2 직렬 콘솔(AWS EC2 Serial Console)은 네트워크 연결이 불가능한 상황에서도 EC2 인스턴스에 접속할 수 있는 콘솔 환경을 제공하는 기능
- 쉽게 말해서, "가상 머신의 모니터와 키보드에 직접 연결하는 것"과 비슷한 역할
- 일반적으로 EC2 인스턴스에 접속할 때는 SSH(리눅스)나 RDP(윈도우)를 사용한다. 하지만 아래 상황에서는 네트워크 기반 접속이 불가능해진다.
  - 보안 그룹(Security Group)에서 포트를 막아버린 경우
  - 방화벽 설정 잘못으로 SSH/RDP 차단된 경우
  - 네트워크 인터페이스(ENI) 설정 오류
  - OS 설정을 잘못 변경해서 부팅 불가 상태에 빠진 경우
- 이럴 때 직렬 콘솔을 사용하면, 네트워크 연결 없이도 인스턴스에 직접 접속해서 문제를 수정할 수 있다.

### 실습: 아파치 웹서버 설치 및 배포

```bash
# 관리자 권한 전환
sudo -s

# 아파치 설치
dnf install -y httpd

# 아파치 웹서버 실행
systemctl start httpd
# OR
service httpd start

# EC2 재부팅시 아파치 웹서버 자동 실행
systemctl enable httpd
# OR
chkconfig httpd on

# 상태 확인
systemctl status httpd
```

상태 확인 출력 예시:

```
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-08-31 05:48:51 UTC; 45s ago
       Docs: man:httpd.service(8)
   Main PID: 26322 (httpd)
      Tasks: 177 (limit: 1059)
     Memory: 13.9M
        CPU: 93ms
     CGroup: /system.slice/httpd.service
             ├─26322 /usr/sbin/httpd -DFOREGROUND
             ├─26323 /usr/sbin/httpd -DFOREGROUND
             ├─26324 /usr/sbin/httpd -DFOREGROUND
             ├─26327 /usr/sbin/httpd -DFOREGROUND
             └─26333 /usr/sbin/httpd -DFOREGROUND
```

index.html 직접 작성:

```bash
vi /var/www/html/index.html
```

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>AWS Web Server</title>
</head>
<body>
    <h1>Hello AWS</h1>
    <h2>EC2 Web Server</h2>
    <p>This is a simple HTML page.</p>
    <hr>
    <p>Server Status: Running</p>
    <p>Web Server: Apache</p>
    <p>Region: ap-northeast-2</p>
    <hr>
    <h3>Welcome to my website!</h3>
</body>
</html>
```

로컬에서 작성한 index.html을 웹 루트로 복사:

```bash
cp /home/ec2-user/index.html /var/www/html/index.html
```

CloudShell에서 EC2로 SCP 전송 (키 파일 권한 설정 후 전송):

```bash
chmod 400 My-EC2-KeyPair.pem
scp -i My-EC2-KeyPair.pem index.html ec2-user@3.37.62.182:/home/ec2-user/index.html
```

> SCP 실행 시 EC2에 등록된 SSH 프라이빗 키(.pem)가 없으면 `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)` 오류가 발생한다. 반드시 인스턴스와 매칭되는 키 페어(-i 옵션)를 지정해야 한다.

## 9. EC2 생명주기

### 중지(Stop)

**요금 청구**

- 중지 상태에서는 인스턴스 사용 요금은 부과되지 않음
- 단, EBS 요금과 Elastic IP 등 다른 리소스 요금은 계속 부과됨 (예: EBS 스토리지 용량, 연결된 Elastic IP 미사용 시에도 비용 발생)

**IP 변경**

- 퍼블릭 IP를 사용 중인 경우, 중지 후 다시 시작하면 퍼블릭 IP가 변경됨 (탄력적 IP를 사용하면 IP 고정 가능)

**중지 가능 조건**

- EBS 기반 인스턴스만 중지가 가능
- 인스턴스 스토어 기반 인스턴스는 중지 불가 (중지 대신 종료만 가능)

**데이터 보존**

- EBS 루트 볼륨과 연결된 추가 EBS 볼륨의 데이터는 유지됨
- 단, 인스턴스 스토어는 휘발성이므로 중지/종료 시 데이터가 삭제된다.

### 인스턴스 상태 전이

![EC2 인스턴스 생명주기 상태 다이어그램 (pending → running → stopping/shutting-down → stopped/terminated)](assets/ec2-lifecycle-states.jpeg)

| 인스턴스 상태 | 설명 | 인스턴스 사용 요금 |
|---|---|---|
| pending | 인스턴스가 running 상태로 될 준비 | 미청구 |
| running | 인스턴스 사용 중 | 청구 |
| stopping | 인스턴스가 중지 또는 최대 절전 모드로 전환 중 | 미청구 / 최대절전시 청구 |
| stopped | 인스턴스가 중지 상태, 재시작 가능 | 미청구 |
| shutting-down | 인스턴스가 종료 중 | 미청구 |
| terminated | 인스턴스가 영구적으로 삭제됨 | 미청구 |

## 10. AWS ENI(Elastic Network Interface)와 Elastic IP

### ENI (Elastic Network Interface)

- EC2 인스턴스에 연결되는 가상 네트워크 카드
- 물리 서버의 LAN 카드와 비슷한 역할
- 하나의 인스턴스에 여러 개의 ENI를 붙일 수도 있다 (하나의 인스턴스에 한 개 이상의 IP를 보유 가능)
- 인스턴스 유형 및 사이즈에 따라 최대 보유 가능한 IP주소가 변동
- 내부적으로는 보안 그룹은 ENI에 부착

주요 특징 및 고유 속성:

- MAC 주소
- 프라이빗 IPv4 주소(기본 1개, 추가 가능)
- 하나 이상의 보안 그룹(Security Group)
- Elastic IP(있다면 연결 가능)
- 인스턴스를 중지/재시작해도 ENI 자체는 유지
- 다른 인스턴스로 이동 가능 (IP 주소 변경 없이 서버 이전 가능)

사용 예:

- 이중화(HA) 구성 시 장애 인스턴스에서 예비 인스턴스로 ENI를 옮겨 서비스 중단 최소화
- 하나의 인스턴스에 여러 네트워크 인터페이스 연결해서 서로 다른 서브넷/보안그룹 사용

### Elastic IP (탄력적 IP)

- 사용자가 1.1.1.1로 서비스를 받고 있다.
- `EC2-ENI 1.1.1.1 <--- User`
- 이때 EC2 인스턴스가 Stop(중지)시킨 후 다시 인스턴스를 Start(시작)상태로 변경하면 공인 IP주소가 변경된다.
- `EC2-ENI 2.2.2.2 <--- User`
- 기존 유저는 해당 서비스를 받기위해서 IP주소로 접속해도 해당 IP 주소로는 서비스를 받을 수 없다.
- `EC2-ENI-Elastic IP address 1.1.1.1 <--- User`
- Elastic IP를 사용하게되면 인스턴스를 중지후 다시 시작해도 퍼블릭 IP 주소가 변경않기 때문에 공인 IP 주소를 그대로 사용할 수 있다.

Elastic IPv4 주소:

- AWS에서 소유하고, 계정에 할당하여 EC2 인스턴스나 ENI에 연결 가능
- 인스턴스를 중지/재시작해도 IP 주소가 변하지 않는다.
- EC2이외의 다른 서비스에서도 사용할 수 있다.(EX: NLB)
- 연결하지 않고 보유하기만해도 비용이 발생한다.

사용 예: 고정 IP가 필요한 서비스(도메인 고정, 방화벽 화이트리스트 등록 등) / EC2 인스턴스 교체 시에도 동일한 IP 유지

## 11. 유저 데이터(User Data)와 메타 데이터(Meta Data)

유저 데이터(User Data)와 메타 데이터(Meta Data)는 AWS EC2에서 인스턴스를 설정하거나 정보를 얻을 때 자주 쓰이는 기능이다.

- **User Data**: 인스턴스에게 "처음 실행할 작업"을 전달
- **Meta Data**: 인스턴스가 "자기 자신의 정보"를 조회

### 유저 데이터(User Data)

- EC2 인스턴스를 생성할 때 자동으로 실행할 스크립트나 설정 정보를 전달하는 기능
- 주로 인스턴스 초기 환경 구성을 자동화할 때 사용
- 기본적으로 인스턴스를 처음 시작할 때 한 번 실행
- Linux에서는 Bash Script, cloud-init 등을 사용할 수 있다.
- Windows에서는 PowerShell 등을 사용할 수 있다.

특징:

- 보통 초기 환경 구성 자동화에 사용
- Bash 스크립트(Linux)나 PowerShell(Windows)로 작성 가능
- 예: 필요한 패키지 설치 / 애플리케이션 코드 배포 / 설정 파일 자동 작성 / 초기 웹 페이지 생성 및 웹 서버 자동 시작

User Data 실행 흐름:

```
EC2 생성 → 인스턴스 부팅 → User Data 실행 → 패키지 설치/환경 설정/서비스 시작 → EC2 사용 준비
```

예시 (Linux):

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd
echo "Hello AWS" > /var/www/html/index.html
```

### 메타 데이터 (Meta Data)

- 실행 중인 EC2 인스턴스가 자기 자신의 정보를 조회할 수 있도록 AWS가 제공하는 정보
- IMDS(Instance Metadata Service)를 사용하여 조회
- 별도의 AWS CLI 명령 없이 HTTP 요청으로 조회 가능
  - IPv4 주소: `169.254.169.254`
  - 일반 인터넷을 통해 외부에서 직접 접근하는 주소가 아님
- 일반적으로 해당 EC2 인스턴스 내부에서 IMDS에 접근하여 사용
- HTTP 요청으로 `http://169.254.169.254/latest/meta-data/`에 접속해서 확인

메타 데이터에서 확인 가능한 대표적인 정보: Instance ID / AMI ID / Instance Type / Private IP / Public IPv4 주소(있는 경우) / Hostname / MAC 주소 / 네트워크 인터페이스 관련 정보 / Security Group 정보 / IAM Role 관련 정보 / Availability Zone / Region 관련 정보 / Block Device Mapping 정보

### IMDS(Instance Metadata Service)

EC2 메타데이터를 조회하기 위한 서비스로, 두 가지 방식이 존재: IMDSv1 / IMDSv2

**IMDSv1**

- 별도의 세션 토큰 없이 HTTP 요청으로 메타데이터 조회
- Request / Response 방식
- 사용 방법이 간단하지만 IMDSv2보다 보안 수준이 낮음

```bash
# Instance ID 조회
curl http://169.254.169.254/latest/meta-data/instance-id

# Private IP 조회
curl http://169.254.169.254/latest/meta-data/local-ipv4

# Public IPv4 조회
curl http://169.254.169.254/latest/meta-data/public-ipv4

# AMI ID 조회
curl http://169.254.169.254/latest/meta-data/ami-id

# Instance Type 조회
curl http://169.254.169.254/latest/meta-data/instance-type

# 사용 가능한 메타데이터 목록 확인
curl http://169.254.169.254/latest/meta-data/
```

**IMDSv2**

- IMDSv1보다 보안이 강화된 방식
- 먼저 세션 토큰을 발급받은 후 해당 토큰을 이용하여 메타데이터를 조회 (Token 기반 Session 방식)
- 토큰의 유효시간(TTL)은 1초 ~ 최대 6시간(21,600초)까지 지정 가능
- IMDSv1보다 SSRF 등의 공격에 대한 방어 기능이 강화됨
- 보안을 위해 IMDSv2 사용을 권장
- 계정, AMI, 인스턴스 설정 등에 따라 IMDSv2 사용을 강제할 수 있음
- `HttpTokens = required`로 설정하면 IMDSv2만 사용 가능

조회 과정:

```bash
# STEP 1. Token 발급
TOKEN=$(curl -X PUT \
"http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# STEP 2. 발급받은 Token을 사용하여 메타데이터 조회
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

명령어 설명:

- `TOKEN=`: TOKEN이라는 Shell 변수에 값을 저장
- `$( ... )`: 괄호 안의 명령어 실행 결과를 변수에 저장 (curl 명령의 결과인 "토큰 문자열"을 TOKEN 변수에 저장)
- `curl`: HTTP 요청을 보내는 명령어
- `-X PUT`: HTTP 요청 방식을 PUT으로 지정 (IMDSv2에서는 토큰을 발급받을 때 PUT 요청을 사용)
- `-H`: HTTP Header를 추가하는 curl 옵션
- `X-aws-ec2-metadata-token-ttl-seconds: 21600`: 발급받을 토큰의 유효시간을 21600초(6시간) 지정

추가 조회 예시:

```bash
# Public IPv4 조회
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4

# Private IPv4 조회
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4

# Instance Name 태그 조회 (Instance Metadata에서 Tag 조회 기능이 활성화되어 있어야 함)
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name
```

### 실습: User Data + IMDSv2로 자기소개 웹페이지 자동 구성

EC2 인스턴스가 부팅될 때 자동으로 웹 서버를 설치하고, 해당 인스턴스의 고유 식별자(인스턴스 ID)와 메타데이터를 웹 페이지(index.html)에 표시하여 접속 시 자신의 인스턴스를 바로 확인할 수 있도록 구성하는 User Data 스크립트:

```bash
#!/bin/bash
sudo -s

# Apache 웹서버 설치 및 실행
dnf install -y httpd
service httpd start
chkconfig httpd on

# 토큰 발급
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# 메타데이터 조회
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AMI_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id)
TAG_NAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name)
Pub_IPv4=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)

# index.html 작성
echo "<h1>INSTANCE-ID : $INSTANCE_ID</h1>" > /var/www/html/index.html
echo "<h1>AMI-ID : $AMI_ID</h1>" >> /var/www/html/index.html
echo "<h1>TAG-NAME : $TAG_NAME</h1>" >> /var/www/html/index.html
echo "<h1>Public-IPv4 : $Pub_IPv4</h1>" >> /var/www/html/index.html
```

Node.js 설치 스크립트 예시 (NodeSource 저장소 사용):

```bash
#!/bin/bash
curl --silent --location https://rpm.nodesource.com/setup_20.x | bash -
dnf -y install nodejs
```

## 12. EC2 권한 부여

EC2 인스턴스에서 다른 AWS 서비스(S3, DynamoDB 등)를 사용하려면 자격 증명이 필요하다. 방법은 크게 두 가지다.

**1) IAM 자격 증명을 등록**

- IAM 사용자를 생성하고, 해당 사용자에 대한 IAM 자격 증명을 발급받아 EC2 인스턴스에 직접 등록
- AWS CLI의 `aws configure` 명령어를 통해 자격 증명을 `~/.aws/credentials` 파일에 저장
- 단점: 관리가 어렵고, 변경이 번거로움
- 예: EC2 100대에 등록된 자격 증명을 모두 교체해야 하는 상황이라면, 각 인스턴스마다 수동으로 변경해야 함

```bash
aws configure
# AWS Access Key ID [None]: <ACCESS_KEY_ID>
# AWS Secret Access Key [None]: <SECRET_ACCESS_KEY>
# Default region name [None]: ap-northeast-2
# Default output format [None]:
```

> 실습에서 `aws configure`로 발급받은 Access Key/Secret Key는 자격 증명이 코드나 설정 파일에 그대로 남는 구조이므로, 실전에서는 아래 IAM 역할 방식이 권장된다.

![여러 EC2 인스턴스에 IAM 사용자 자격 증명을 직접 배포하는 구조](assets/ec2-role-credential-rotation.jpeg)

**2) IAM 역할을 부여**

- 필요한 권한이 포함된 IAM 역할(Role)을 생성하고 이를 EC2 인스턴스에 부여
- 장점: 관리와 교체가 간편함
- 인스턴스에 연결된 역할을 변경하면, 인스턴스 내부에서 자동으로 새로운 자격 증명을 사용
- 내부적으로 AWS가 주기적으로 자격 증명을 자동 갱신하므로 별도 관리 필요 없음
- 보안성 향상: 자격 증명이 코드나 파일로 저장되지 않으므로 유출 위험이 크게 줄어든다.

![EC2 인스턴스에 부여된 하나의 IAM Role을 여러 인스턴스가 공유하는 구조](assets/iam-role-permission-result.jpeg)

### EC2에서 IAM 역할로 다른 AWS 서비스 호출 (Node.js SDK 예시)

역할이 부여된 EC2 인스턴스 위에서 애플리케이션이 Access Key 없이 S3, DynamoDB, SageMaker 등에 접근하는 구조:

![EC2 인스턴스의 Node.js 애플리케이션이 S3/DynamoDB/SageMaker에 접근하는 구조](assets/ec2-app-aws-service-integration.jpeg)

IAM 사용자/역할 목록을 조회하는 AWS SDK v3(Node.js) 예시:

```javascript
// AWS SDK v3에서 IAM 클라이언트 클래스를 가져옵니다.
const { IAMClient, ListUsersCommand, ListRolesCommand } = require("@aws-sdk/client-iam");

// AWS SDK v3 클라이언트는 모듈화되어 있습니다. 각 서비스마다 자신의 클라이언트 모듈이 있습니다.

async function runTest() {
    const client = new IAMClient({
        region: "ap-northeast-2", // 리전 설정
    });
    // 사용자 목록
    console.log("사용자 목록 출력");
    try {
        const dataUser = await client.send(new ListUsersCommand({}));
        dataUser.Users.forEach((element) => {
            console.log(element.UserName); // 사용자 이름 출력
        });
    } catch (error) {
        console.error(error); // 오류 처리
    }
}
```

### 역할(Role)

- IAM 역할(Role)은 AWS 리소스에 임시로 권한을 부여하기 위한 AWS의 보안 주체(Security Principal)
- 사용자(User)와 비슷하게 권한을 가질 수 있지만, 장기 자격 증명(Access Key, Secret Key)을 사용하지 않고, 필요할 때만 임시 자격 증명을 발급받아 사용한다.

**1. 역할(Role)의 특징**

- 장기 자격 증명 없음: IAM 사용자와 달리 Access Key나 Secret Key를 발급받아 보관하지 않음. 대신, AWS에서 필요할 때 임시 자격 증명을 자동 발급
- 권한 부여 방식: 역할에는 정책(Policy)을 연결해 권한 범위를 정의. 해당 역할을 누가/무엇이 사용할 수 있는지 신뢰 정책(Trust Policy)로 지정
- 사용 주체: AWS 서비스(EC2, Lambda 등) / 다른 AWS 계정의 사용자 / AWS SSO 사용자
- 자동 만료: 발급된 임시 자격 증명은 지정된 시간이 지나면 자동 만료. 재발급은 AWS에서 자동 처리

**2. 역할(Role)의 활용 예시**

| 활용 시나리오 | 설명 |
|---|---|
| EC2에서 S3 접근 | EC2 인스턴스에 역할을 부여하면, 인스턴스 내부 애플리케이션이 Access Key 없이 S3에 접근 가능 |
| 계정 간 접근 | 다른 AWS 계정의 리소스에 접근할 때, 해당 계정에서 역할을 만들고 권한 부여 후, 내 계정에서 AssumeRole 사용 |
| AWS 서비스 권한 위임 | Lambda 함수가 DynamoDB나 SQS에 접근할 수 있도록 역할을 부여 |
| 임시 작업자 계정 | 외부 협력업체에 장기 사용자 계정 대신 기간 한정 역할 제공 |

**3. 사용자(User)와 역할(Role)의 차이**

| 구분 | IAM 사용자(User) | IAM 역할(Role) |
|---|---|---|
| 자격 증명 | 장기 Access Key / Secret Key | 임시 자격 증명 |
| 대상 | 사람 또는 애플리케이션(고정) | AWS 서비스, 다른 계정 사용자, 임시 작업자 |
| 만료 | 없음 (직접 삭제 전까지 유지) | 있음 (자동 만료, 재발급 가능) |
| 보안성 | 키 유출 시 위험 | 유출 위험 낮음 (임시 발급) |
| 관리 | 직접 키 관리 필요 | AWS에서 자동 관리 |

## 13. 클라우드 환경의 EC2 활용

### 수직 확장(Vertical Scale, Scale Up)

- 수직 확장은 하나의 서버(인스턴스)의 성능을 높이는 방식으로 시스템 용량을 확장하는 방법
- 즉, 서버 자체의 CPU, 메모리, 디스크 속도 등을 업그레이드해서 처리 능력을 높이는 것을 의미

개념: 기존 서버를 더 강력한 사양으로 변경 (예: CPU 1개, RAM 1GB인 서버 → CPU 16개, RAM 16GB 서버로 업그레이드). 서버의 물리적 또는 가상 자원을 늘려 처리 성능을 높임

![수직 확장 시 성능과 비용 증가 비교(CPU x1 → x16, 비용 x30)](assets/vertical-scale-cost-diagram.jpeg)

장점:

- 구성 단순: 서버 개수 변화 없이 기존 시스템을 업그레이드하므로 아키텍처 변경이 거의 없음
- 관리 편리: 서버가 한 대이므로 관리·모니터링이 쉬움
- 적은 변경으로 성능 향상: 애플리케이션 수정 없이 자원 업그레이드만으로 성능 개선 가능

단점:

- 비용 증가 폭 큼: 성능이 16배 늘어도 비용은 30배 오를 수 있음
- 한계 존재: 서버 하드웨어 사양에는 물리적 한계가 있어 무한 확장 불가능
- 장애 시 전체 서비스 중단: 서버가 하나이므로 해당 서버 장애 시 서비스 전체가 중단됨

활용 예시: 초기 트래픽이 적고, 단일 서버로도 충분히 서비스가 가능한 스타트업 / 복잡한 분산 구조 없이 빠르게 성능을 높여야 하는 경우 / 데이터베이스 서버(MySQL, PostgreSQL) 등에서 성능 병목을 줄이기 위해 고성능 인스턴스로 교체할 때

### 수평 확장(Horizontal Scale, Scale Out)

- 수평 확장은 서버(혹은 인스턴스)의 개수를 늘려서 전체 시스템 성능과 처리량을 향상시키는 방식
- 즉, 한 대의 서버를 업그레이드하는 대신 여러 대의 서버를 병렬로 운영해 부하를 분산하는 구조

개념: 동일하거나 유사한 사양의 서버를 여러 대 추가하여 부하를 분산 (예: CPU 1개, RAM 1GB 서버 1대를 → 동일 사양 서버 16대 운영). 주로 로드 밸런서(Load Balancer)를 사용하여 요청을 여러 서버로 분산

![수평 확장 시 동일 사양 서버 16대로 부하를 분산하는 구조](assets/horizontal-scale-cost-diagram.jpeg)

장점:

- 확장성 우수: 서버 개수를 늘리기만 하면 성능 확장 가능
- 비용 효율성: 소형 인스턴스를 여러 대 사용하는 것이, 대형 인스턴스 1대보다 비용 대비 성능이 나은 경우가 많음
- 장애 대응성: 일부 서버가 장애를 일으켜도 나머지 서버가 서비스 유지 가능 (고가용성)
- 무중단 확장 가능: 서버 추가·제거가 비교적 쉬움

단점:

- 아키텍처 복잡도 증가: 로드 밸런싱, 데이터 동기화, 세션 관리 등 추가 설계 필요
- 데이터베이스 병목: 서버가 늘어나도 DB가 단일 인스턴스면 병목 발생 가능
- 애플리케이션 설계 요구: 서버 간 상태 공유가 최소화된(Stateless) 구조가 필요

활용 예시: 웹 서비스 트래픽이 급증하는 경우 (예: 쇼핑몰, 뉴스 사이트) / 마이크로서비스 아키텍처(MSA) 기반 서비스 / CDN, API 서버, 채팅 서버 등 대규모 동시 접속이 필요한 환경

수평 확장은 동일한 서버를 여러 대 만들어 운영하는 방식으로, 로드 분산과 관리가 복잡해지는 단점이 있다. 서버 수가 많아질수록 아키텍처 설계와 관리 포인트가 증가한다. 하지만 비용 대비 성능 확장성이 뛰어나며, 필요 시 서버 수를 무한히 늘릴 수 있다. 예를 들어 성능이 100만 배 필요하다면 100만 개의 인스턴스를 배포하면 된다.

클라우드 환경에서는 이러한 수평 확장이 기본 전략이며, 하나의 대형 서버로 처리하는 방식보다 다수의 소형 서버로 나누어 처리하는 것이 확장성과 안정성 측면에서 유리하다. 따라서 수평 확장은 대규모 트래픽 처리와 안정적인 서비스 운영을 위한 클라우드 환경의 핵심 철학이다.

### 클라우드 환경에서 고려해야 할 주요 요소

- **탄력성**: 수요에 따라 인스턴스를 늘리거나 줄이는 기능이 필요하다. 이는 사람이 수동으로 조절하는 것이 아니라 자동화된 환경에서 통제되어야 한다. (탄력성: 수요에 따라 컴퓨팅 파워/용량을 확장하거나 축소할 수 있는 능력)
- **안정성**: 클라우드 환경에서는 장애가 언제든 발생할 수 있기 때문에 이를 대비한 설계가 필요하다. 이는 클라우드의 단점이 아니라, 그 상황을 대비하여 구조를 설계해야 한다는 의미다.

온프레미스 환경과 달리 클라우드에서는 각 인스턴스를 소모품처럼 다루어야 하며, 장애나 통제에 의해 인스턴스가 종료되는 것은 정상적인 상황이다. 따라서 예고 없이 인스턴스가 종료될 수 있다는 전제로 아키텍처를 설계해야 한다.

온프레미스에서 클라우드로 이전할 때는 리소스를 이전하는 것뿐 아니라, 필요에 따라 자원을 쉽게 늘리고 줄일 수 있는 환경을 직접 구축해야 한다. 이를 위해 다음과 같은 두 가지 환경 구축이 중요하다.

1. **스테이트리스(Stateless) 환경 구축**: 인스턴스가 상태에 의존하지 않도록 설계
2. **고가용성(High Availability) 환경 구축**: 일부 인스턴스가 장애를 일으켜도 서비스가 지속되도록 설계

Stateless 환경은 인스턴스의 특정 상태나 데이터를 로컬에 저장하지 않고, 외부 저장소나 다른 서비스에 상태를 보관하는 방식이다. 이를 통해 인스턴스가 종료되거나 교체되어도 서비스가 지속적으로 운영될 수 있다.

결론적으로, 클라우드 환경에서는 탄력성과 안정성을 기본 전제로 하고, 인스턴스가 언제든 종료될 수 있다는 가정하에 고가용성과 스테이트리스 구조를 갖춘 설계를 하는 것이 필수적이다.

그래서 EC2를 잘 활용하려면 언제나 인스턴스가 떨어질 수 있다는 전제하에 다음과 같은 방법이 필요하다.

- 인스턴스를 자동으로 프로비저닝할 수 있는 방법
- 인스턴스를 가용 영역별로 분산할 수 있는 방법
- 인스턴스 클러스터에 트래픽을 분산할 수 있는 방법
- 인스턴스가 상태를 저장하지 않도록 하여 언제든 삭제되거나 추가되어도 무방한 구조를 만드는 방법

> 관련 서비스 예시: Auto Scaling, Elastic Load Balancer, RDS, S3

## 14. AWS EC2 Auto Scaling

- AWS EC2 Auto Scaling은 애플리케이션을 모니터링하여 트래픽 변화나 부하 상황에 맞춰 EC2 인스턴스 수를 자동으로 조정하는 서비스이다.
- 필요 시 서버를 자동으로 추가하고, 부하가 줄면 불필요한 서버를 종료해 비용을 절감하면서 예측 가능한 성능을 유지한다.

목적:

- 정확한 수의 EC2 인스턴스를 유지하도록 보장 (최소 인스턴스 수 이하로 내려가지 않도록 인스턴스 추가 / 최대 인스턴스 수 이상으로 늘어나지 않도록 인스턴스 삭제)
- 다양한 스케일링 정책 적용 (CPU 부하, 네트워크 트래픽, 요청량 등에 따라 인스턴스 크기 및 개수 조정 / 특정 시간에 인스턴스 개수를 늘리고, 다른 시간에 줄이기 / 여러 가용 영역(AZ)에 인스턴스를 분산 배치하여 안정성 확보)

주요 기능:

- **자동 확장(Scale Out)**: CPU 사용률, 네트워크 트래픽, 요청 수 등 지정 지표가 기준 이상이면 인스턴스를 자동으로 추가
- **자동 축소(Scale In)**: 부하가 줄어 지정 지표가 기준 이하이면 인스턴스를 자동으로 종료
- **예측 스케일링(Predictive Scaling)**: 과거 트래픽 패턴을 학습해 미래 부하를 예측하고 사전에 인스턴스를 확장
- **고가용성 보장**: 장애 발생 시 인스턴스를 자동으로 교체해 서비스 중단 최소화
- **다중 가용 영역 지원**: 여러 AZ에 인스턴스를 분산 배치하여 장애에 강한 구조 제공

장점: 비용 최적화(필요한 만큼만 인스턴스를 운영하여 불필요한 비용 절감) / 유연성(갑작스러운 트래픽 증가나 감소에 실시간 대응 가능) / 안정성(인스턴스 장애 시 자동 복구) / 운영 효율성(정책 기반 자동 실행으로 수동 관리 부담 감소)

### 구성 요소 및 동작 구조

- Auto Scaling Group(ASG) 생성: 관리할 인스턴스 집합 정의
- 시작 템플릿(Launch Template) 또는 시작 구성(Launch Configuration) 설정: 인스턴스 타입, AMI, 보안 그룹, 키 페어, IAM 역할, 유저 데이터(자동 스크립트) 등 정의
- 스케일링 정책 설정 (예: CPU 사용률이 70% 이상이면 2대 추가, 30% 이하이면 1대 축소)
- 모니터링 및 상태 확인: CloudWatch 및 ELB와 연계하여 지표 기반 정책 자동 실행

기타 설정 사항:

- 종료 정책(Scale-In 시 인스턴스 종료 순서): 인스턴스가 2개 이상인 AZ에서 가장 오래된 시작 템플릿(동일하면 과금 종료 시간 가장 가까운 인스턴스 종료)
- 커스텀 종료 정책: 가장 예전 시작 템플릿부터 / 가장 오래된 인스턴스부터 / 가장 최근 인스턴스부터

예시 시나리오: 온라인 쇼핑몰에서 세일 이벤트 시작 → 트래픽 급증 → 인스턴스 5대에서 20대로 자동 확장. 세일 종료 후 트래픽 감소 → 불필요한 인스턴스 자동 종료 → 다시 5대로 축소

Auto Scaling Group 내부에서는 인스턴스마다 서로 다른 IP를 부여받아 관리된다.

![Auto Scaling Group 내 여러 EC2 인스턴스가 각각 다른 IP를 부여받는 구조](assets/autoscaling-group-ip-pool.jpeg)

## 15. ELB (Elastic Load Balancer)

**ELB를 사용하지 않은 경우**

- 사용자(User)는 직접 Auto Scaling Group 내부의 EC2 인스턴스들에 개별 접속하게된다.
- 각 인스턴스는 고유한 퍼블릭 IP를 가진다.
- 클라이언트가 어떤 서버에 접속할지는 사용자가 직접 선택해야 한다.
- 서버 부하 분산이 자동으로 이뤄지지 않으며, 특정 서버에만 접속자가 몰릴 수 있다.
- 특정 인스턴스가 장애가 나면, 해당 IP를 접속하던 사용자는 서비스 이용이 불가능하다.

**ELB를 사용한 경우**

- 사용자(User)는 개별 EC2 인스턴스의 IP로 접속하지 않고, 로드 밸런서의 도메인 주소로 접속
- 로드 밸런서(ELB)는 들어오는 요청을 Auto Scaling Group 내의 모든 EC2 인스턴스에 자동으로 분산한다.
- 트래픽이 균등하게 분배되어 특정 서버에 과부하가 걸리는 것을 방지한다.
- 장애가 발생한 인스턴스는 자동으로 연결 대상에서 제외
- Auto Scaling과 연동되어, 서버가 자동으로 늘어나거나 줄어들어도 로드 밸런서 주소 하나로 서비스 이용이 가능

![로드 밸런서 도메인 주소를 통해 여러 EC2 인스턴스로 트래픽이 분산되는 구조](assets/elb-load-balancer-diagram.jpeg)

### ELB란?

- 다수의 EC2에 트래픽을 분산 시켜주는 서비스
- 총 4가지 종류: Application Load Balancer / Network Load Balancer / Classic Load Balancer / Gateway Load Balancer
- Health Check: 직접 트래픽을 발생시켜 인스턴스가 살아있는지 체크
- Autoscaling과 연동 가능
- 지속적으로 IP 주소가 바뀌며 IP 고정 불가능 (항상 도메인 기반으로 사용)

**1. Application Load Balancer(ALB)**

- HTTP/HTTPS 트래픽에 최적화된 7계층(L7) 로드 밸런서
- URL 경로 기반, 호스트 기반 라우팅 가능
- 트래픽을 모니터링하여 라우팅 가능 (예: image.sample.com → 이미지 서버, web.sample.com → 웹 서버)
- 웹 애플리케이션, 마이크로서비스 구조에 주로 사용
- 장점: 세밀한 라우팅 규칙, 다양한 HTTP 기능 지원

**2. Network Load Balancer(NLB)**

- TCP/UDP 트래픽 처리에 최적화된 4계층(L4) 로드 밸런서
- 매우 빠른 트래픽 분산, 초고성능 처리 가능
- Elastic IP 할당 가능 → IP 고정 가능
- 금융, 게임 서버 등 대규모 네트워크 처리에 적합
- 장점: 저지연, 고성능, 대규모 연결 처리

**3. Classic Load Balancer(CLB)**

- L4/L7 혼합 지원하는 구형 로드 밸런서
- 예전에 사용되던 타입으로 현재는 신규 구축에 잘 사용하지 않음
- 기능 제한적, 레거시 환경 유지 보수 시 사용
- 장점: 오래된 AWS 환경과의 호환성

**4. Gateway Load Balancer(GWLB)**

- 네트워크 트래픽을 제3자 보안 장비(방화벽, IDS/IPS 등)에 전달
- 3계층(L3) 기반, 패킷 단위 트래픽 처리 가능
- 보안 어플라이언스 통합 및 확장성 제공
- 가상 어플라이언스 배포·확장 관리에 적합
- 장점: 보안 및 트래픽 검사에 특화

### ELB + Auto Scaling

- Auto Scaling을 통해 EC2 인스턴스의 개수를 자동으로 조정하고, ELB를 사용해 각 인스턴스로 트래픽을 고르게 분산 처리한다.
- Auto Scaling으로 인스턴스가 증가하거나 감소하면, 해당 인스턴스가 자동으로 ELB에 등록되거나 해제되어 무중단 서비스 운영이 가능하다.
- 이를 통해 트래픽 급증 시에도 안정적인 서비스 제공이 가능하며, 트래픽이 줄면 불필요한 인스턴스를 줄여 비용을 절감한다.
- ELB와 Auto Scaling의 연동은 고가용성과 확장성을 동시에 확보하는 핵심적인 클라우드 아키텍처 패턴이다.

![ELB와 Auto Scaling Group이 연동되어 트래픽을 분산하는 구조](assets/elb-autoscaling-integration.jpeg)

### 대상 그룹(Target Group)

- 대상 그룹(Target Group)은 ELB(ALB/NLB)가 트래픽을 실제로 전달할 서버나 서비스들을 묶어놓은 논리적인 묶음
- 로드밸런서는 요청이 들어오면, 바로 서버로 보내지 않고 먼저 어떤 대상 그룹으로 보낼지를 결정한 뒤, 그 대상 그룹 안에 있는 대상 중 하나에게 요청을 전달한다.

```
로드밸런서 → 대상 그룹 → 실제 서버
```

**대상 유형(Target Type)**

대상 그룹에는 여러 종류의 대상(Target)을 등록할 수 있다.

1. **Instance 타입**: EC2 인스턴스를 직접 대상으로 등록하는 방식. EC2 인스턴스 ID 기준으로 관리되며, Auto Scaling Group과 함께 사용하기 가장 좋다. 서버가 늘어나거나 줄어들면 자동 반영
2. **IP 타입**: 특정 IP 주소를 대상으로 등록하는 방식 (온프레미스 서버 / 다른 VPC 서버 / 외부 서버). EC2가 아닌 서버를 연결할 때 사용하며, 하이브리드 환경에서 자주 사용
3. **Lambda 타입**: 대상으로 Lambda 함수를 지정하는 방식 (ALB → Lambda 직접 호출). 서버 없이 API 구성 가능, 서버리스 API 구조에서 활용
4. **ALB 타입**: 다른 Application Load Balancer를 대상으로 연결하는 방식 (ALB → ALB → 서버 형태의 다단계 구조). 대규모 시스템에서 계층 분리할 때 사용

**프로토콜 및 포트 설정**

- 대상 그룹은 반드시 통신 방식과 포트를 함께 설정해야 한다.
- 설정 항목: 프로토콜(HTTP, HTTPS, TCP, gRPC 등) / 포트(80, 443, 8080 등)
- 예: HTTP + 80, HTTPS + 443, TCP + 3306
- 로드밸런서는 이 설정에 맞춰서 대상 서버로 요청을 전달한다.

**트래픽 분산 방식**

대상 그룹 안에 여러 대상이 있을 때, 어느 서버로 보낼지를 정하는 방식이 필요하다.

1. **라운드 로빈(Round Robin)**: 가장 기본적인 방식. 요청을 순서대로 나눠서 전달한다. (예: 서버 A → B → C → A → B → C)
2. **고정 세션(Sticky Session)**: 한 사용자를 특정 서버에 계속 연결시키는 방식. 한 번 연결된 사용자는 계속 같은 서버로 이동. 쇼핑몰 장바구니, 로그인 유지 같은 서비스에서 자주 사용된다.

**헬스 체크(Health Check)**

- 대상 그룹의 가장 중요한 기능 중 하나가 헬스 체크이다.
- 헬스 체크는 "이 서버 살아 있나?"를 자동으로 확인하는 기능이다.
- 로드밸런서는 일정 시간마다 대상 서버에 요청을 보내서 상태를 검사한다.

여러 대상 그룹을 도메인/경로별로 라우팅하는 예시(웹서버, 이미지서버, 대시보드, 람다 함수):

![ALB가 도메인별로 여러 대상 그룹에 트래픽을 라우팅하는 구조](assets/alb-target-group-routing.jpeg)

### 리스너(Listener)란?

- 정의: ALB(Application Load Balancer)로 들어오는 요청을 처리하는 주체로, 트래픽의 프로토콜 + 포트 단위로 구성됨.
- 역할: 로드 밸런서가 어떤 요청을 받을지, 받은 요청을 어떻게 처리할지 규칙(rule)으로 결정.

**1. 규칙(Rule) 기능**

- ALB가 특정 프로토콜과 포트로 들어온 요청을 어떻게 처리할지 정의.
- 예시: `http:8080` 요청 → A 대상 그룹의 80번 포트로 전달 / `https:443` 요청 → B 대상 그룹의 80번 포트로 전달 / POST 요청 → 지정된 응답(에러페이지 등) 반환

**2. 규칙 조건**

- 요청의 세부 요소를 조건으로 활용 가능: Header, QueryString, Source IP, HTTP Method 등

**3. 요청 처리 방식**

- Forward: 대상 그룹으로 요청 전달
- Redirect: 다른 URL로 리다이렉트
- Fixed Response: 지정된 응답 반환

리스너별로 조건(method/host)에 따라 서로 다른 대상 그룹으로 라우팅하는 예시:

![리스너(80/8080/443)별 규칙 조건에 따라 이미지서버/웹서버 대상 그룹으로 라우팅되는 구조](assets/alb-listener-rules.jpeg)

### ALB 규칙 (Rules)

**1) 조건 (Conditions, 모두 만족 필요)**

- ALB는 요청이 특정 규칙과 일치하는지 확인 후 동작을 결정함
- 주요 조건 요소: 호스트 헤더(Host Header), HTTP 메서드(Method: GET, POST 등), Source IP(요청자의 IP 주소), HTTP Header(요청 헤더 값), QueryString(쿼리 파라미터)

**2) 작업 유형 (Actions)**

- 규칙이 만족될 경우 ALB가 수행하는 동작
- 대표적인 작업: 대상 그룹(Target Group) 전달(최대 5개 그룹까지 전달 가능, 각 그룹에 가중치(0~999) 부여하여 트래픽 비율 조정 가능) / URL Redirection / 고정 응답(Fixed Response, 예: 404 - ALB가 직접 지정된 메시지를 반환, 예: "서비스 점검 중")

**3) 규칙 순위 (Priority)**

- 규칙은 1~50,000 범위 내의 우선순위 번호를 가짐
- 낮은 숫자일수록 우선순위가 높음
- 요청이 들어오면 순서대로 평가하며, 가장 먼저 일치하는 규칙이 적용됨

> 관련: 3.  AWS - 클라우드 기초 개념
