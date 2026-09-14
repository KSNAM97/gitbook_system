# CloudFront 실습: Origin·Behavior·버저닝

이 문서는 S3와 EC2/ALB를 Origin으로 하는 CloudFront 배포를 만들고, Origin Group으로 장애 조치를 구성한 뒤, Behavior로 여러 Origin을 경로별로 나누고, 마지막으로 정적 자산 버저닝을 실습하는 과정을 다룬다. 이론 12. Amazon CloudFront 문서와 이어지는 구성이다.

## 1. 아키텍처 개요

- S3 버킷을 Origin으로 하는 CloudFront 배포를 만들어, S3에 직접 접속했을 때와 CloudFront 엣지 로케이션을 거쳐 접속했을 때를 비교한다.
- EC2 2대를 Primary·Secondary Origin으로 하는 Origin Group을 구성해, Primary 장애 시 Secondary의 대체 이미지로 자동 전환되는 것을 확인한다.
- Behavior로 `*`(기본, S3)와 `/api/*`(ALB→EC2)를 나눠 하나의 CloudFront 배포로 정적 파일과 API를 함께 서비스한다.
- 정적 자산 파일명에 해시를 붙이는 버저닝 방식으로 캐시 무효화 없이 새 버전을 배포하는 것을 실습한다.

## 2. EC2에 웹 서버 배포(사용자 데이터)

```bash
#!/bin/bash
sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)

echo "<h1>$INSTANCE_ID</h1>" >> /var/www/html/index.html
```

IMDSv2(토큰 발급 후 메타데이터 조회) 방식으로 Instance ID를 가져와 페이지에 출력한다.

## 3. S3를 Origin으로 CloudFront 배포 생성

1. S3 콘솔에서 [버킷 만들기]를 선택해 버킷을 생성하고, 생성된 버킷으로 들어가 [업로드] → [파일 추가]에서 `cat.jpg`, `dog.jpg` 같은 정적 파일을 업로드한다. 이 버킷은 퍼블릭으로 열지 않고 [모든 퍼블릭 액세스 차단]을 그대로 유지한다.
2. CloudFront 콘솔 왼쪽 탐색 메뉴의 [배포]에서 [배포 생성]을 선택한다.
3. Origin 도메인 입력란을 클릭하면 뜨는 목록에서 방금 만든 S3 버킷을 선택한다.
4. Origin 액세스에서 [Origin access control settings(권장)]을 선택하고, [새 OAC 생성]을 눌러 이름을 입력한 뒤 생성한다.
5. 뷰어 프로토콜 정책은 [HTTP를 HTTPS로 리디렉션]을 선택한다.
6. [배포 생성]을 선택하면, 화면 상단에 "다음 버킷 정책 정책을 복사하여 Amazon S3 버킷 정책에 붙여넣어야 합니다"라는 안내와 함께 정책 예시가 표시된다. 이 정책을 복사해 S3 콘솔의 해당 버킷 → [권한] 탭 → [버킷 정책] → [편집]에서 붙여넣고 저장한다. 정책은 아래와 같은 형태다.

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

7. 저장 후 CloudFront 배포 상세 화면으로 돌아와 배포 상태가 [배포됨]으로 바뀔 때까지 기다린다.

## 4. S3 직접 접속 vs CloudFront 경유 접속 비교

S3 버킷을 퍼블릭으로 열지 않은 상태에서 버킷 주소로 직접 접속하면 다음과 같이 거부된다.

```text
https://<S3 버킷 이름>.s3.ap-northeast-2.amazonaws.com/cat.jpg

<Error>
<Code>AccessDenied</Code>
<Message>Access Denied</Message>
</Error>
```

같은 파일을 CloudFront 배포 도메인으로 요청하면 엣지 로케이션을 거쳐 정상적으로 응답한다.

```text
# CloudFront를 사용하여 엣지 로케이션으로 접속
https://<CloudFront 배포 도메인>.cloudfront.net/cat.jpg
https://<CloudFront 배포 도메인>.cloudfront.net/dog.jpg
```

S3는 Private 상태를 유지하면서, 실제 콘텐츠 제공은 CloudFront(OAC로 인증된 요청)만 가능하도록 구성된 것을 확인할 수 있다.

## 5. Origin Group으로 404 이미지 Failover 구성

Primary·Secondary EC2 각각에 동일한 페이지를 배포하고, 404 상황에서 보여줄 대체 이미지를 준비한다.

```bash
#!/bin/bash
sudo -s
sudo yum install -y httpd
systemctl start httpd
chkconfig httpd on

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)

echo "<h1>hello,world!</h1>" >> /var/www/html/page.html
echo "<h1>$INSTANCE_ID</h1>" >> /var/www/html/page.html

echo "<h1>OMG its 404</h1>" >> /var/www/html/backup.html
echo "<h1>$INSTANCE_ID</h1>" >> /var/www/html/backup.html
```

로컬에서 준비한 대체 이미지를 각 EC2로 전송한다.

```bash
# Origin Group Primary
scp -i My-EC2-KeyPair.pem ./404-error.jpg ec2-user@<Primary EC2 퍼블릭 IP>:/home/ec2-user/

# Origin Group Secondary
scp -i My-EC2-KeyPair.pem ./404-error.jpg ec2-user@<Secondary EC2 퍼블릭 IP>:/home/ec2-user/
```

각 EC2에 접속해 이미지를 웹 루트로 옮기고, `backup.html`에서 해당 이미지를 출력하도록 수정한다.

```bash
# 이미지를 html 파일 디렉터리로 복사
cp /home/ec2-user/404-error.jpg /var/www/html/

# 404 Error 이미지를 출력하기 위해 html 코드 수정
vi /var/www/html/backup.html
```

```html
<img src="/404-error.jpg" alt="404 Error Image" style="max-width:600px;">
```

Primary·Secondary 두 EC2 모두 같은 작업을 반복한다.

이후 CloudFront 콘솔에서 Origin Group을 구성한다.

1. 대상 CloudFront 배포 상세 화면의 [원본] 탭에서 [원본 생성]을 두 번 선택해, Primary EC2와 Secondary EC2를 각각 Custom Origin으로 등록한다. Origin 도메인에는 각 EC2의 퍼블릭 DNS(또는 퍼블릭 IP)를 입력한다.
2. 같은 배포 상세 화면의 [원본 그룹] 탭에서 [원본 그룹 생성]을 선택한다.
3. 원본 그룹 이름을 입력하고, 원본 1(기본)에는 Primary EC2 Origin을, 원본 2에는 Secondary EC2 Origin을 선택한다.
4. 장애 조치 조건에서 CloudFront가 Secondary로 전환할 HTTP 상태 코드(예: 403, 404, 500, 502, 503, 504)를 체크한다.
5. [원본 그룹 생성]을 선택한다.
6. [동작] 탭에서 `backup.html`을 포함하는 경로 패턴의 동작을 선택해 [편집]을 누르고, 원본 및 원본 그룹에서 방금 만든 원본 그룹을 선택한 뒤 저장한다.

이렇게 구성한 CloudFront 배포 도메인으로 `backup.html`에 접속하면, Primary 장애 시 Secondary가 준비한 404 안내 이미지가 대신 응답한다.

```text
https://<CloudFront 배포 도메인>.cloudfront.net/backup.html
```

## 6. Behavior로 S3와 ALB를 함께 사용하기

정적 페이지는 S3에서, API 요청은 ALB(EC2)에서 처리하도록 하나의 CloudFront 배포에 두 Origin을 등록한다.

EC2에 API용 사용자 데이터를 등록한다.

```bash
#!/bin/bash
yum install -y httpd
systemctl enable httpd
systemctl start httpd
mkdir -p /var/www/html/api
echo "<h1>Hello from EC2 API</h1>" > /var/www/html/api/hello
echo "<h1>EC2 API Origin</h1>" > /var/www/html/index.html
```

1. CloudFront 배포 상세 화면의 [원본] 탭에서 [원본 생성]을 선택해 ALB를 Custom Origin으로 추가한다(Origin 도메인에 ALB의 DNS 이름을 입력, 프로토콜은 HTTP 또는 HTTPS 선택). S3 Origin은 3단계에서 이미 등록되어 있다.
2. [동작] 탭에서 [동작 생성]을 선택한다.
3. 경로 패턴에 `/api/*`를 입력하고, 원본 및 원본 그룹에서 방금 추가한 ALB Origin을 선택한다.
4. 뷰어 프로토콜 정책과 허용 HTTP 메서드(GET, HEAD, OPTIONS 등 API에 필요한 메서드)를 지정하고 [동작 생성]을 선택한다.
5. [동작] 탭 목록에서 `/api/*` 동작이 기본 동작(`*`)보다 위(우선순위 번호가 더 작은 자리)에 있는지 확인한다. CloudFront는 목록 위에서부터 첫 매칭 규칙을 적용하므로, 구체적인 경로 규칙이 기본 규칙보다 항상 위에 있어야 한다. 순서가 다르면 목록에서 해당 동작을 선택해 [이동] 또는 [편집]으로 우선순위를 조정한다.

배포가 반영되면 요청 경로에 따라 서로 다른 Origin이 응답한다.

```text
https://<CloudFront 배포 도메인>.cloudfront.net/index.html      # S3에서 응답
https://<CloudFront 배포 도메인>.cloudfront.net/api/hello       # ALB(EC2)에서 응답
```

경로에 `/api/`가 없으면 기본 Behavior가 적용되어 S3로, `/api/`로 시작하면 `/api/*` Behavior가 적용되어 ALB로 요청이 전달되는 것을 직접 확인할 수 있다. 참고로 ALB에 등록한 EC2에 별도의 HTML을 배포해두면(`/var/www/html/api/main.html`), ALB DNS 이름으로 직접 접속했을 때도 같은 내용을 확인할 수 있다.

```text
http://<ALB DNS 이름>/api/main.html   # 경로에 /api/가 있으므로 ALB(EC2)로 서비스된다.
```

## 7. 정적 자산 버저닝 실습

CloudFront 캐시 무효화(Invalidation) 없이 새 버전을 배포하는 버저닝 방식을 직접 확인한다. S3 버킷 루트에 아래와 같이 파일명에 해시가 붙은 정적 자산을 준비한다.

**v1 (최초 배포)**

- `app-v1.abc123.css`, `app-v1.abc123.js`, `logo.png`, `favicon.ico`
- `index.html`에서 `<link rel="stylesheet" href="/app-v1.abc123.css" />`, `<script src="/app-v1.abc123.js"></script>`로 참조한다.

**v2 (수정 배포)**

- `app-v2.abc456.css`, `app-v2.abc456.js`(파일명 자체가 바뀐 새 버전)
- `index.html`도 `/app-v2.abc456.css`, `/app-v2.abc456.js`를 참조하도록 함께 교체한다.

1. S3 콘솔에서 대상 버킷으로 들어가 [업로드] → [파일 추가]로 v1 파일 세트(`index.html`, `app-v1.abc123.css`, `app-v1.abc123.js`, `logo.png`, `favicon.ico`)를 업로드한다.
2. CloudFront 배포 도메인으로 접속해 브라우저 개발자 도구를 열고(F12) [Network] 탭에서 `app-v1.abc123.css`·`app-v1.abc123.js` 요청을 찾아 응답 헤더의 `X-Cache`, `Age` 값을 확인한다.
3. S3 콘솔에서 같은 버킷에 [업로드] → [파일 추가]로 v2 파일 세트(파일명이 바뀐 `app-v2.abc456.css`, `app-v2.abc456.js`와, 그 파일을 참조하도록 수정된 `index.html`)를 업로드해 기존 파일 위에 덮어쓴다.
4. 다시 CloudFront 배포 도메인으로 접속해 새로고침한다. 파일명이 달라졌기 때문에 CloudFront는 v2 파일을 완전히 새로운 캐시 대상으로 인식하여, Invalidation 없이도 [Network] 탭에 즉시 `app-v2.abc456.css`·`app-v2.abc456.js` 요청이 잡히는 것을 확인한다.
5. 반대로 파일명을 그대로 두고 `index.html` 내용만 바꾸는 상황을 재현하려면, S3에 같은 파일명으로 덮어쓴 뒤 CloudFront 콘솔 왼쪽 탐색 메뉴의 [무효화]에서 [무효화 생성]을 선택하고 객체 경로에 `/index.html`(또는 전체를 지우려면 `/*`)을 입력한 뒤 [생성]을 누른다. 무효화 상태가 [완료]로 바뀐 후 새로고침하면 최신 내용이 반영되는 것을 확인할 수 있다.

## 8. 리소스 정리

1. CloudFront 배포를 비활성화한 뒤 삭제한다(배포 삭제 전 비활성화 상태로 전환하는 대기 시간이 필요하다).
2. Origin Group에 연결했던 Primary·Secondary EC2 인스턴스를 종료한다.
3. ALB와 대상 그룹을 삭제한다.
4. 실습에 사용한 S3 버킷의 객체를 비우고 버킷을 삭제한다.
5. 버킷 정책·OAC 설정 등 임시로 추가한 리소스를 정리한다.

> 관련: 이론 12. Amazon CloudFront · 이론 4. AWS S3 · 이론 2. AWS EC2 - 배포 · 가이드 12. VPC부터 Route 53까지: 고가용성 웹 서비스 구축
