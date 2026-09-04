# S3 + VPC Endpoint로 프라이빗 EC2에 정적 사이트 배포하기

이 문서는 정적 웹 페이지 파일을 S3에 올려두고, NAT 게이트웨이 없이 VPC Endpoint(Gateway Endpoint)를 통해 프라이빗 서브넷의 EC2가 S3에서 파일을 가져와 배포하는 절차를 다룬다. 이론 3. AWS VPC 문서의 13. VPC Endpoint 섹션, 가이드 6. AWS ALB + Auto Scaling + 대상 그룹 통합 가이드와 이어지는 구성이다.

## 1. 아키텍처 개요

- **S3**: 배포할 정적 파일(`index.html` 등)을 저장하는 버킷.
- **S3 Gateway Endpoint**: 프라이빗 서브넷의 라우팅 테이블에 연결해, EC2가 인터넷 게이트웨이나 NAT 게이트웨이 없이도 S3에 접근할 수 있게 한다.
- **EC2(프라이빗 서브넷)**: 시작 템플릿의 사용자 데이터(User Data)에서 S3의 파일을 내려받아 웹 서버 루트에 배치한다.
- **ALB + Auto Scaling 그룹**: 가이드 6. AWS ALB + Auto Scaling + 대상 그룹 통합 가이드에서 구성한 것을 그대로 사용한다.

이 구성을 사용하면 EC2 인스턴스에 퍼블릭 IP나 NAT 아웃바운드 경로가 전혀 없어도, S3에 있는 콘텐츠를 배포·갱신할 수 있다.

## 2. S3 버킷 준비와 기본 CLI 사용법

1. S3 콘솔에서 버킷을 생성한다. 버킷 이름은 전역적으로 고유해야 하므로 계정을 특정할 수 있는 값 대신 임의의 접미사를 붙인다(예: `my-static-site-<임의문자열>`).
2. 배포할 `index.html`을 버킷에 업로드한다.
3. EC2에 연결된 IAM Role에 해당 버킷에 대한 읽기 권한(`s3:GetObject`, `s3:ListBucket` 등)을 부여한다. IAM Role 연동 방법은 이론 3. AWS VPC 문서의 14. EC2와 IAM Role 연동, 이론 2. AWS EC2 - 배포 문서의 12. EC2 권한 부여 섹션을 참고한다.

자주 쓰는 S3 CLI 명령어는 다음과 같다.

```bash
# 버킷 내용 확인 (하위 폴더까지 전부 나열)
aws s3 ls s3://my-static-site-bucket --recursive

# S3 -> 로컬(EC2) 파일 다운로드
aws s3 cp s3://my-static-site-bucket/index.html .

# 로컬(EC2) -> S3 파일 업로드
aws s3 cp newFile.txt s3://my-static-site-bucket/

# S3 <-> 로컬 디렉터리 동기화 (변경된 파일만 전송)
aws s3 sync s3://my-static-site-bucket .
```

`cp`는 지정한 파일 하나만 전송하고, `sync`는 원본과 대상을 비교해 변경되거나 없는 파일만 전송한다. 정적 사이트 전체를 배포·갱신할 때는 `sync`가, 특정 파일 하나만 가져올 때는 `cp`가 적합하다.

## 3. S3 Gateway Endpoint 생성

1. VPC 콘솔의 [엔드포인트]에서 [엔드포인트 생성]을 선택한다.
2. 서비스 카테고리는 [AWS 서비스]를 선택하고, 서비스 이름에서 `com.amazonaws.<리전>.s3`(Gateway 유형)를 검색해 선택한다.
3. 엔드포인트를 연결할 VPC를 선택한다.
4. 라우팅 테이블에서 EC2가 속한 프라이빗 서브넷의 라우팅 테이블을 선택한다. 엔드포인트를 생성하면 이 라우팅 테이블에 S3로 향하는 경로가 자동으로 추가된다.
5. 정책은 기본값(전체 액세스)으로 두거나, 필요하면 특정 버킷만 허용하도록 커스텀 정책을 지정한다.
6. [엔드포인트 생성]을 선택한다.

생성이 완료되면 프라이빗 서브넷의 라우팅 테이블에 대상이 `vpce-xxxxxxxx`(S3 프리픽스 리스트)인 경로가 추가된 것을 확인할 수 있다. 이 경로 덕분에 S3로 향하는 트래픽만 NAT 게이트웨이를 거치지 않고 곧바로 S3로 전달된다.

## 4. 시작 템플릿 User Data에서 S3 파일 배포

가이드 6에서 만든 시작 템플릿의 사용자 데이터에 다음 스크립트를 등록한다.

```bash
#!/bin/bash
sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on

# S3에 올려둔 정적 파일을 웹 루트로 복사
aws s3 cp s3://my-static-site-bucket/index.html /var/www/html --region ap-northeast-2
```

인스턴스가 시작될 때마다 S3의 최신 `index.html`을 받아오므로, S3의 파일만 갱신하면 이후 새로 생성되는 인스턴스(Auto Scaling으로 늘어나는 인스턴스 포함)에는 별도 배포 작업 없이 최신 콘텐츠가 자동으로 반영된다.

인스턴스 ID처럼 인스턴스별로 달라지는 값을 페이지에 표시하고 싶다면, IMDSv2 토큰을 발급받아 메타데이터를 조회한 뒤 페이지에 삽입하는 방식을 함께 사용할 수 있다.

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/instance-id")

echo "<p>Instance ID: $INSTANCE_ID</p>" >> /var/www/html/index.html
```

## 5. 동작 확인

1. Auto Scaling 그룹의 인스턴스가 정상적으로 실행 중인지, 대상 그룹의 상태 검사가 healthy인지 확인한다.
2. ALB의 DNS 이름으로 접속해 S3에 올려둔 정적 콘텐츠가 정상적으로 표시되는지 확인한다.
3. 프라이빗 서브넷의 EC2에는 퍼블릭 IP도, NAT를 통한 아웃바운드 경로도 없다는 점을 라우팅 테이블에서 다시 한번 확인한다. S3 접근만 Gateway Endpoint를 통해 이루어진다.
4. S3의 `index.html`을 수정한 뒤 인스턴스를 새로 교체(Instance Refresh 또는 종료 후 재생성)하면, 새 인스턴스에 수정된 콘텐츠가 반영되는 것을 확인할 수 있다.

> 관련: 이론 3.  AWS VPC · 가이드 6.  AWS ALB + Auto Scaling + 대상 그룹 통합 가이드
