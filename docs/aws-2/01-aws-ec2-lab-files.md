# AWS EC2 - 실습 파일

이론 2. AWS EC2 - 배포와 가이드 3~5에서 사용하는 명령어·User Data·SDK 예제 파일을 모은 문서다. 인증 정보 값은 `<ACCESS_KEY_ID>`, `<SECRET_ACCESS_KEY>`로 가렸다.

## IMDSv2 메타데이터 조회와 User Data

**개요**: 인스턴스 메타데이터(IMDSv2) 조회, 부팅 시 웹 서버 설치 User Data, Node.js 설치, aws configure 과정을 정리한 실습 기록.

`02-1) AWS EC2.txt`

```bash
[ec2-user@ip-172-31-45-51 ~]$ sudo -s


[root@ip-172-31-45-51 ec2-user]# TOKEN=$(curl -X PUT  "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    56 100    56   0     0 33392     0  --:--:-- --:--:-- --:--:-- 56000


[root@ip-172-31-45-51 ec2-user]# echo $TOKEN
<IMDS_TOKEN>


# 인스턴스 ID 조회
[root@ip-172-31-45-51 ec2-user]# curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
i-078dae3d648fcbac9


# AMI ID 조회
# 해당 인스턴스를 생성할 때 사용한 AMI의 ID 반환
[root@ip-172-31-45-51 ec2-user]# curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id
ami-00b5b2470beafd65f


# Name 태그 값 조회
# EC2 콘솔에서 Name 태그가 설정되어 있고
# 메타데이터 옵션에서 "인스턴스 메타데이터 태그"가 활성화돼 있어야 정상 반환
[root@ip-172-31-45-51 ec2-user]# curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name
My-EC2-Meatadata


# Public IPv4
[root@ip-172-31-45-51 ec2-user]# curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
13.209.17.211


-EC2 인스턴스가 부팅될 때 자동으로 웹 서버를 설치하고, 해당 인스턴스의 고유 식별자(인스턴스 ID)를 
 웹 페이지(index.html)에 표시하여 접속 시 자신의 인스턴스를 바로 확인할 수 있도록 구성


#!/bin/bash
sudo -s

# Apache 웹서버 설치 및 실행
dnf  install -y  httpd
service  httpd  start
chkconfig  httpd  on

# 토큰 발급
TOKEN=$(curl -X PUT  "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# 메타데이터 조회

# INSTANCE-ID 조회
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)

# AMI ID 조회
AMI_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id)

# Name 태그 값 조회
TAG_NAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name)

# Public IPv4
Pub_IPv4=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)


# index.html 작성
echo "<h1>INSTANCE-ID : $INSTANCE_ID</h1>"  >  /var/www/html/index.html
echo "<h1>AMI-ID : $INSTANCE_ID</h1>"  >>  /var/www/html/index.html
echo "<h1>TAG-NAME : $TAG_NAME</h1>"  >>  /var/www/html/index.html
echo "<h1>Public-IPv4 : $Pub_IPv4</h1>"  >>  /var/www/html/index.html


#!/bin/bash
sudo -s

dnf  install -y  httpd
service  httpd  start
chkconfig  httpd  on

TOKEN=$(curl -X PUT  "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AMI_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id)
Pub_IPv4=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)
TAG_NAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name)


# index.html 작성
echo "<h1>INSTANCE-ID : $INSTANCE_ID</h1>"  >  /var/www/html/index.html
echo "<h1>AMI-ID : $INSTANCE_ID</h1>"  >>  /var/www/html/index.html
echo "<h1>TAG-NAME : $TAG_NAME</h1>"  >>  /var/www/html/index.html
echo "<h1>Public-IPv4 : $Pub_IPv4</h1>"  >>  /var/www/html/index.html


#!/bin/bash
curl --silent --location https://rpm.nodesource.com/setup_20.x | bash -
dnf -y install nodejs
```

## AWS SDK v3 IAM 조회 예제

**개요**: AWS 자격 증명 체인 실습에서 사용하는 Node.js 스크립트(IAM 사용자·역할 목록 출력).

`test.js`

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

runTest();
```
