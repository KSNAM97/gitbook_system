# Route 53 Health Check와 라우팅 정책 실습

이 문서는 서로 다른 리전(또는 서로 다른 EC2)에 동일한 웹 페이지를 배포한 뒤, Route 53 Health Check로 상태를 감시하고 Failover 라우팅 정책으로 장애 조치를 확인하는 실습이다. 이론 11. Amazon Route 53의 Health Check·Routing Policy 설명과 이어지는 구성이다.

## 1. 아키텍처 개요

- Primary EC2와 Secondary EC2 두 대에 동일한 웹 서버(httpd)를 배포한다.
- Route 53 Health Check가 Primary EC2를 주기적으로 점검한다.
- Failover Routing Policy로 평소에는 Primary로, Primary 장애 시 Secondary로 자동 전환되는 레코드를 구성한다.
- DNS 전파 상태는 외부 도구(whatsmydns.net)로 확인한다.

## 2. EC2에 웹 서버 배포(사용자 데이터)

Primary·Secondary EC2 각각의 시작 템플릿(또는 인스턴스 시작 시) 사용자 데이터에 아래 스크립트를 등록한다. IMDSv2 방식(토큰 발급 후 메타데이터 조회)으로 Instance ID를 가져와 페이지에 출력한다.

```bash
#!/bin/bash
sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)

echo "<h1>$INSTANCE_ID</h1>" > /var/www/html/index.html
echo "<h1>hello Soldesk IT Academy</h1>" >> /var/www/html/index.html
```

- `TOKEN`을 먼저 발급받아 헤더에 실어 요청하는 방식이 IMDSv2다. 토큰 없이 바로 메타데이터를 조회하는 IMDSv1보다 SSRF 공격에 안전하므로, 신규 인스턴스에는 이 방식을 사용한다.
- 결과 페이지에는 `<h1>i-xxxxxxxxxxxxxxxxx</h1>`와 `<h1>hello Soldesk IT Academy</h1>`가 순서대로 출력된다.

## 3. 로컬에서 EC2로 파일 직접 전송(scp)

사용자 데이터 대신 로컬에서 작성한 `index.html`을 직접 배포해야 할 때는 `scp`로 각 EC2에 전송한다.

```bash
scp -i My-EC2-KeyPair.pem ./index.html ec2-user@<Primary EC2 퍼블릭 IP>:~/index.html
scp -i My-EC2-KeyPair.pem ./index.html ec2-user@<Secondary EC2 퍼블릭 IP>:~/index.html
```

전송한 뒤에는 각 EC2에 접속해 `/var/www/html/` 경로로 옮기고 웹 서버가 해당 파일을 서비스하도록 권한을 맞춘다.

## 4. S3 버킷 정책으로 정적 파일 공개(선택)

정적 리소스를 S3에서 직접 제공하려면, 버킷 정책으로 `GetObject` 권한을 퍼블릭에 열어준다. Block Public Access를 해제해야 하는 구성이므로, 실제로 공개해도 되는 정적 자산에만 적용한다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<S3 버킷 이름>/*"
        }
    ]
}
```

## 5. Route 53 Health Check 생성

1. Route 53 콘솔 왼쪽 탐색 메뉴의 [상태 확인]에서 [상태 확인 생성]을 선택한다.
2. 이름을 입력하고, 모니터링 대상은 [엔드포인트]를 선택한다.
3. 지정 방법은 [IP 주소] 또는 [도메인 이름] 중 하나를 선택하고, Primary EC2의 퍼블릭 IP(또는 도메인)를 입력한다.
4. 프로토콜은 HTTP, 포트는 80을 지정하고, 필요하면 경로에 `/index.html`을 입력한다.
5. 고급 설정에서 요청 간격(10초 또는 30초)과 실패 임계값을 지정한다.
6. [상태 확인 생성]을 선택한다.

생성 직후에는 전 세계 여러 Health Checker 노드가 점검을 시작하며, [상태 확인] 상세 화면의 [상태 확인 상태] 탭에서 리전별 점검 결과를 확인할 수 있다.

## 6. Failover Routing Policy 레코드 생성

1. Route 53 콘솔의 [호스팅 영역]에서 대상 도메인의 Hosted Zone으로 들어가 [레코드 생성]을 선택한다.
2. 레코드 이름과 유형(A)을 지정하고, 라우팅 정책은 [장애 조치]를 선택한다.
3. 장애 조치 레코드 유형은 [기본]을 선택하고, 값에는 Primary EC2의 퍼블릭 IP를 입력한다. 상태 확인은 5단계에서 만든 Health Check를 연결한다.
4. 같은 레코드 이름으로 [레코드 생성]을 한 번 더 진행하여, 이번에는 장애 조치 레코드 유형을 [보조]로 선택하고 값에 Secondary EC2의 퍼블릭 IP를 입력한다.
5. 두 레코드 모두 저장되면, 평소에는 Primary IP로 응답하다가 Health Check가 Primary를 비정상으로 판정하면 자동으로 Secondary IP로 응답이 바뀐다.

## 7. DNS 전파 확인

레코드를 변경한 뒤 전 세계 DNS 서버에 실제로 반영되었는지는 [whatsmydns.net](https://www.whatsmydns.net)에서 확인할 수 있다. 도메인을 입력하고 레코드 유형(A)을 선택하면, 여러 국가의 DNS 서버가 어떤 IP로 응답하는지 한눈에 볼 수 있다. 지역마다 응답이 다르면 아직 전파가 끝나지 않았거나, TTL 값 때문에 이전 응답이 캐싱되어 있는 상태다.

## 8. CloudWatch로 Health Check 통계 확인

Route 53 Health Check 결과는 CloudWatch Metrics로 전달되며, 여러 리전의 Health Checker 값(정상 1, 비정상 0)을 어떤 통계로 집계하느냐에 따라 해석이 달라진다.

예를 들어 도쿄 1, 싱가포르 1, 미국 0, 유럽 1인 경우:

- **최소(Minimum)**: 하나라도 0(비정상)이 있으면 결과가 0이다. 장애를 가장 민감하게 잡는다.
- **최대(Maximum)**: 하나라도 1(정상)이 있으면 결과가 1이다. 장애를 덜 민감하게 잡는다.
- **평균(Average)**: 전체 검사값의 평균이다(예: `[1,1,0,1] --> 0.75`). 부분 장애율을 볼 때 유용하다.

**기간(Period)**은 CloudWatch가 지표를 집계하는 시간 단위로, 보통 1분·5분 단위를 사용한다(1분이면 빠른 감지, 5분이면 더 안정적인 감지). CloudWatch Alarm에서 이 통계와 기간을 조합해 알림 조건을 설정하면, 일부 리전만 장애인 상황과 전체 장애 상황을 구분해서 대응할 수 있다.

## 9. 리소스 정리

1. Route 53의 Failover 레코드(기본/보조) 2개를 삭제한다.
2. Route 53 Health Check를 삭제한다.
3. 임시로 열어둔 S3 버킷 정책을 원복하거나 버킷을 삭제한다.
4. Primary·Secondary EC2 인스턴스를 종료한다.

> 관련: 이론 11. Amazon Route 53 · 이론 6. AWS CloudWatch · 가이드 12. VPC부터 Route 53까지: 고가용성 웹 서비스 구축
