# Amazon CloudFront - 실습 파일

이론 12와 가이드 14에서 사용하는 명령, Origin 테스트 페이지, 버저닝 정적 파일을 모은 문서다.

## CloudFront 실습 메모

**개요**: Origin·Behavior·캐시 무효화 실습 명령과 URL.

`Cloud-Front.txt`

```bash
#!/bin/bash
sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo "<h1>$INSTANCE_ID</h1>" >> /var/www/html/index.html


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::[버킷명]/*"
        }
    ]
}


https://버킷이름.s3.ap-northeast-2.amazonaws.com


https://my-test-origin-bucket-123456789012-ap-northeast-2-an.s3.ap-northeast-2.amazonaws.com

This XML file does not appear to have any style information associated with it. The document tree is shown below.
<Error>
<Code>AccessDenied</Code>
<Message>Access Denied</Message>
<RequestId>R1KFX66HR8G0PNCC</RequestId>
<HostId>DoA0RGP7Evu0GQcV7vabAmkjPGqe2Im2TCt1FNlNbsI2vZjmhlSUjd9qXjGML0JlseOJnXKVoqAWaballY2Fe40w3LHPaDEy</HostId>
</Error>


	# S3로 직접 접속
https://my-test-origin-bucket-123456789012-ap-northeast-2-an.s3.ap-northeast-2.amazonaws.com/cat.jpg
https://my-test-origin-bucket-123456789012-ap-northeast-2-an.s3.ap-northeast-2.amazonaws.com/dog.jpg


	# CloudFront를 사용하여 엣지 로케이션으로 접속
https://d1yzaizq7gqoij.cloudfront.net/cat.jpg
https://d1yzaizq7gqoij.cloudfront.net/dog.jpg


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


	# CDN Origin-Group Primary (CloudShell)
~ $ scp  -i  My-EC2-KeyPair.pem  ./404-error.jpg  ec2-user@54.180.108.147:/home/ec2-user/


	# CDN Origin-Group Secondary (CloudShell)
~ $ scp  -i  My-EC2-KeyPair.pem  ./404-error.jpg  ec2-user@13.209.7.144:/home/ec2-user/


	# CDN Origin-Group Primary (EC2)

   # Image 확인
[root@ip-172-31-44-122 ec2-user]# ls -l
total 56
-rw-r--r--. 1 ec2-user ec2-user 56914 Sep 14 03:57 404-error.jpg


   # Image를 html 파일 디렉터리로 복사
[root@ip-172-31-44-122 ec2-user]# cp  /home/ec2-user/404-error.jpg   /var/www/html/


   # Image 복사 확인
[root@ip-172-31-44-122 ec2-user]# ls  -l  /var/www/html/
total 60
-rw-r--r--. 1 root root 56914 Sep 14 04:02 404-error.jpg
-rw-r--r--. 1 root root    12 Sep 14 03:29 backup.html


   # 404 Error 이미지를 출력하기위해서 html 코드 수정
[root@ip-172-31-44-122 ec2-user]# vi  /var/www/html/backup.html
<img src="/404-error.jpg" alt="404 Error Image" style="max-width:600px;">


	# CDN Origin-Group Secondary (EC2)
[root@ip-172-31-39-84 ec2-user]# ls -l
total 56
-rw-r--r--. 1 ec2-user ec2-user 56914 Sep 14 03:58 404-error.jpg


[root@ip-172-31-39-84 ec2-user]# cp  /home/ec2-user/404-error.jpg   /var/www/html/


[root@ip-172-31-39-84 ec2-user]# ls  -l  /var/www/html/
total 60
-rw-r--r--. 1 root root 56914 Sep 14 04:02 404-error.jpg
-rw-r--r--. 1 root root    12 Sep 14 03:29 backup.html


[root@ip-172-31-44-122 ec2-user]# vi  /var/www/html/backup.html
<img src="/404-error.jpg" alt="404 Error Image" style="max-width:600px;">


https://dwchfhf0jusbw.cloudfront.net/backup.html


https://www.naver.com
http://www.naver.com


https://www.naver.com/


#!/bin/bash
yum install -y httpd
systemctl enable httpd
systemctl start httpd
mkdir -p /var/www/html/api
echo "<h1>Hello from EC2 API</h1>I" > /var/www/html/api/hello
echo "<h1>EC2 API Origin</h1>" > /var/www/html/index.html


https://d1dawzgq0ppta9.cloudfront.net/index.html		# S3

https://d1dawzgq0ppta9.cloudfront.net/api/hello		# ALB


	# EC2에 2대에 설정 (/var/www/html/api/main.html)

[ec2-user@ip-172-31-31-241 ~]$ sudo -s


[root@ip-172-31-31-241 ec2-user]# vi /var/www/html/api/main.html

<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Soldesk</title>
</head>
<body>

    <h1>Hello, Soldesk!</h1>
    <p>AWS CloudFront 테스트 페이지입니다.</p>

    <hr>

    <h2>서비스 정보</h2>
    <p>정적 웹 페이지 테스트</p>

</body>
</html>


http://my-api-alb-1212556668.ap-northeast-2.elb.amazonaws.com/api/main.html		# 경로에 /api/ 가 있기 때문에 ALB(EC2)로 서비스된다.
Hello, Soldesk!
AWS CloudFront 테스트 페이지입니다.

서비스 정보
정적 웹 페이지 테스트
```

## Behavior 테스트 index.html

**개요**: 정적 콘텐츠 Behavior 확인용 페이지.

`index.html`

```html
<!doctype html>
<html lang="ko">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>CloudFront 버저닝 실습 v2</title>
    <!-- 실습 가이드에서는 S3 루트에 업로드한다고 가정해 절대 경로를 사용합니다 -->
    <link rel="stylesheet" href="/app-v2.abc456.css" />
  </head>
  <body>
    <header class="site-header">
      <h1>CloudFront 버저닝 실습 v2</h1>
    </header>
    <main class="container">
      <section class="card">
        <h2>정적 자산 참조 확인</h2>
        <p>이 페이지는 <code>app-v2.abc456.css</code> 와 <code>app-v2.abc456.js</code> 를 참조합니다</p>
        <p id="runtime-info">로딩 중</p>
        <!-- 로고 크기는 app-v2.abc456.css의 .logo에서 고정 -->
        <img src="logo.png" alt="logo" class="logo" />
      </section>

      <section class="card">
        <h3>체크리스트</h3>
        <ul>
          <li>Network 탭에서 CSS, JS 요청의 파일명을 확인하세요</li>
          <li>X-Cache 와 Age 헤더를 확인하세요</li>
        </ul>
      </section>
    </main>

    <script src="/app-v2.abc456.js"></script>
  </body>
</html>
```

## app-v1.abc123.css

**개요**: 버저닝 파일명 실습용 CSS(v1).

`app-v1.abc123.css`

```css
/* app-v1.abc123.css - 기본 스타일 */

:root {
  --bg: #0b1220;
  --card: #111a2b;
  --text: #e6eefc;
  --muted: #9bb0d3;
  --accent: #4da3ff;
}

* {
  box-sizing: border-box;
}

html, body {
  margin: 0;
  padding: 0;
  background: var(--bg);
  color: var(--text);
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto,
    Helvetica, Arial, Apple SD Gothic Neo, Noto Sans KR, "나눔고딕",
    "맑은 고딕", sans-serif;
  line-height: 1.6;
}

.site-header {
  padding: 24px 20px;
  border-bottom: 1px solid #1d2a44;
}

h1, h2, h3 {
  margin: 0 0 12px;
}

.container {
  max-width: 900px;
  margin: 24px auto;
  padding: 0 16px;
}

.card {
  background: var(--card);
  border: 1px solid #1d2a44;
  border-radius: 12px;
  padding: 16px 18px;
  margin-bottom: 16px;
  box-shadow: 0 6px 16px rgba(0,0,0,0.25);
}

.logo {
  display: block;
  width: 120px;
  height: 70px;
  object-fit: contain;
  margin-top: 8px;
  filter: drop-shadow(0 8px 18px rgba(77,163,255,0.35));
}

code {
  background: rgba(255,255,255,0.06);
  padding: 2px 6px;
  border-radius: 6px;
  color: var(--accent);
}

ul {
  margin: 8px 0 0;
  padding-left: 18px;
}

li {
  color: var(--muted);
}
```

## app-v2.abc456.css

**개요**: 버저닝 파일명 실습용 CSS(v2).

`app-v2.abc456.css`

```css
/* app-v1.abc123.css - 기본 스타일 */
:root {
  --bg: #0b1220;
  --card: #111a2b;
  --text: #e6eefc;
  --muted: #9bb0d3;
  --accent: #4da3ff;
}

* { box-sizing: border-box; }

html, body {
  margin: 0;
  padding: 0;
  background: var(--bg);
  color: var(--text);
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, Apple SD Gothic Neo, Noto Sans KR, "나눔고딕", "맑은 고딕", sans-serif;
  line-height: 1.6;
}

.site-header {
  padding: 24px 20px;
  border-bottom: 1px solid #1d2a44;
}

h1, h2, h3 { margin: 0 0 12px; }

.container {
  max-width: 900px;
  margin: 24px auto;
  padding: 0 16px;
}

.card {
  background: var(--card);
  border: 1px solid #1d2a44;
  border-radius: 12px;
  padding: 16px 18px;
  margin-bottom: 16px;
  box-shadow: 0 6px 16px rgba(0,0,0,0.25);
}

.logo {
  display: block;
  width: 160px;
  height: auto;
  margin-top: 8px;
  filter: drop-shadow(0 8px 18px rgba(77,163,255,0.35));
}

code {
  background: rgba(255,255,255,0.06);
  padding: 2px 6px;
  border-radius: 6px;
  color: var(--accent);
}

ul {
  margin: 8px 0 0;
  padding-left: 18px;
}

li { color: var(--muted); }
```
