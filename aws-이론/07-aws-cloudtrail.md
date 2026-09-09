# AWS CloudTrail

## 1. CloudTrail 개요

AWS CloudTrail은 AWS 계정에서 누가, 언제, 어떤 작업을 했는지 기록하는 서비스다. 쉽게 말하면 AWS 계정 활동을 기록하는 CCTV 역할을 한다.

**예**

- 누가 EC2를 종료했는지
- 누가 S3 버킷을 생성하거나 삭제했는지
- 누가 IAM 사용자를 변경했는지
- 누가 AWS 콘솔에 로그인했는지

**주요 기능: 계정 활동 기록**

CloudTrail은 다음 경로로 발생하는 작업을 이벤트로 기록한다.

- AWS Management Console
- AWS CLI
- AWS SDK
- AWS 서비스 간 API 호출

**기록되는 정보**: 누가 실행했는지, 언제 실행했는지, 어떤 작업을 했는지, 어떤 리소스에 작업했는지, 성공했는지 실패했는지, 어떤 IP에서 요청했는지가 함께 남는다.

**활용 목적**: 보안 감사, 문제 원인 추적, 규정 준수, 비정상적인 활동 확인에 사용한다.

**저장**: CloudTrail의 Event history에서는 최근 90일의 관리 이벤트를 조회할 수 있다. Trail을 만들면 이벤트 로그를 S3에 저장해서 장기간 보관할 수 있고, CloudWatch Logs와 연동하면 로그를 모니터링하고 경보를 설정할 수 있다.

**활용 예시**

- EC2가 갑자기 삭제된 경우: CloudTrail에서 누가 `TerminateInstances` 작업을 실행했는지 확인한다.
- S3 버킷 설정이 변경된 경우: CloudTrail에서 누가 설정을 변경했는지 확인한다.

## 2. Trail

Trail은 CloudTrail 이벤트를 계속 수집해서 저장하도록 만드는 설정이다. CloudTrail이 AWS 활동을 기록하는 서비스라면, Trail은 기록한 로그를 어디에 저장하고 관리할지 정하는 설정이다. Trail을 만들면 로그를 S3에 장기 저장할 수 있다.

**S3 저장**: Trail 생성 시 S3 버킷을 지정하면 CloudTrail 이벤트가 해당 S3 버킷에 로그 파일로 저장된다. 흐름은 `AWS 활동 발생 → CloudTrail Event 생성 → Trail이 이벤트 수집 → S3 버킷에 저장` 순이다.

**리전 설정**

- **단일 리전 Trail**: 특정 리전에서 발생한 이벤트만 기록한다.
- **다중 리전 Trail**: 여러 리전에서 발생한 이벤트를 한 곳에 수집한다. 일반적으로 다중 리전 Trail을 많이 사용한다.

**다른 서비스와 연동**

- **CloudWatch Logs**: CloudTrail 이벤트를 CloudWatch Logs로 전달해 로그 검색 및 모니터링이 가능해진다.
- **Metric Filter**: 특정 이벤트를 찾아 숫자 형태의 지표로 변환한다.
- **CloudWatch Alarm**: 특정 이벤트가 발생하면 경보를 발생시킨다.
- **EventBridge**: 특정 AWS 이벤트가 발생하면 자동 작업을 실행한다.

예를 들어 EC2가 종료되면 CloudTrail에 기록되고, EventBridge가 그 이벤트를 감지해 Lambda를 실행하는 자동화 흐름을 구성할 수 있다.

## 3. CloudTrail Event

CloudTrail Event는 AWS에서 발생한 하나의 작업 기록이다. 쉽게 말하면 CloudTrail 로그의 한 줄 한 줄의 기록이며, JSON 형식으로 기록된다.

**예**: 사용자가 EC2를 종료하면 CloudTrail Event가 생성되고, 다음과 같은 필드가 함께 기록된다.

- `eventName`: `TerminateInstances`
- `userIdentity`: 실행한 사용자
- `eventTime`: 실행 시간
- `sourceIPAddress`: 요청 IP

**CloudTrail Event 종류(3가지)**

### 1) Management Event

AWS 리소스를 생성·변경·삭제·관리하는 작업을 기록한다. 쉽게 말하면 AWS 인프라를 관리한 기록이다.

- 예: EC2 시작/중지/삭제, VPC 생성/삭제, IAM Role 생성/삭제, 보안 그룹 변경, Trail 생성/삭제
- 활용: 누가 리소스를 만들었는지, 누가 설정을 변경했는지, 누가 리소스를 삭제했는지 확인할 수 있다.

### 2) Data Event

리소스 안에 있는 데이터에 접근하거나 작업한 기록이다. 쉽게 말하면 리소스 내부 데이터 사용 기록이다.

- 예: S3 객체 다운로드(`GetObject`), S3 객체 업로드(`PutObject`), S3 객체 삭제(`DeleteObject`), Lambda 함수 호출, DynamoDB 데이터 접근
- 특징: Management Event보다 훨씬 많은 로그가 발생할 수 있고, 기본적으로 별도 설정이 필요하며, 추가 비용이 발생할 수 있다.

### 3) Insight Event

평소와 다른 비정상적인 API 활동 패턴을 탐지한다. 쉽게 말하면 CloudTrail이 평소와 다른 이상 행동을 찾아주는 기능이다.

- 예: 짧은 시간에 API 호출이 갑자기 급증, 평소보다 삭제 작업이 크게 증가, 실패하거나 거부되는 API 요청이 갑자기 증가
- 특징: 별도 활성화가 필요하고, 추가 비용이 발생할 수 있다.

## 4. CloudTrail 실습 흐름

1. **CloudTrail Trail 생성 및 모니터링 확인**: 새로운 Trail을 만들어 로그를 S3 버킷에 저장한다. 이때 Data Event 수집 활성화 설정을 켜면 S3 객체 단위 동작이나 Lambda 호출 같은 세부 이벤트도 기록된다. 생성된 Trail이 정상적으로 로그를 남기고 있는지 확인한다.
2. **CloudShell에서 EC2 정보 조회 테스트**: AWS CloudShell을 열고 EC2 인스턴스 정보를 가져오기 위한 API를 호출한다(예: `aws ec2 describe-instances`). API 호출 로그가 CloudTrail에 기록되며, 이벤트 기록에는 호출한 시간·호출자·API 이름(`DescribeInstances`)·실행 결과 등이 남는다.
3. **S3 Object 생성 및 요청 이벤트 확인**: S3 버킷에 객체를 업로드하고(예: `aws s3 cp file.txt s3://<S3 버킷 이름>/`), 다운로드(Get) 혹은 삭제(Delete) 같은 요청을 실행한다. Data Event가 활성화되어 있다면 S3 객체 단위의 동작이 CloudTrail에 기록되며, 이벤트에는 요청한 사용자/서비스 정보, 요청 시간 및 지역(Region), 동작 종류(`PutObject`, `GetObject`, `DeleteObject` 등), 요청 결과(성공/실패)가 담긴다.

> 관련: 이론 6. AWS CloudWatch · 이론 8. AWS KMS(Key Management Service) · 이론 2. AWS EC2 - 배포
