# AWS VPC - 실습 파일

이론 3. AWS VPC와 가이드 6·7·12에서 사용하는 User Data, MariaDB 접속 계정, VPC Endpoint 구분, EFS 마운트 cloud-config를 모은 문서다.

## 웹 서버 User Data · MariaDB 계정 · Endpoint 유형

**개요**: 프라이빗 서브넷 EC2 구성에 사용하는 User Data, MariaDB 계정 생성, 게이트웨이/인터페이스 엔드포인트 비교 메모.

`02-1) AWS VPC.txt`

```bash
#!/bin/bash
sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo "<h1>$INSTANCE_ID</h1>" >> /var/www/html/index.html 


[ec2-user@ip-10-0-3-134 ~]$ sudo -s
[root@ip-10-0-3-134 ec2-user]# sudo dnf install -y mariadb105-server

[root@ip-10-0-3-134 ec2-user]# systemctl  start  mariadb
[root@ip-10-0-3-134 ec2-user]# systemctl  enable  mariadb


[root@ip-10-0-3-134 ec2-user]# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 4
Server version: 10.5.29-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>


# 1) root 비밀번호 설정
MariaDB [(none)]> ALTER USER 'root'@'localhost' IDENTIFIED BY '1234';
Query OK, 0 rows affected


# 2) testuser 계정 생성 (모든 호스트(%)에서 접속 가능하도록 설정)
MariaDB [(none)]> CREATE USER 'testuser'@'%' IDENTIFIED BY '1234';
Query OK, 0 rows affected


# 3) testuser에게 모든 데이터베이스에 대한 권한 부여
MariaDB [(none)]> GRANT ALL PRIVILEGES ON *.* TO 'testuser'@'%';
Query OK, 0 rows affected


# 4) 권한 적용
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected


# 5) 생성된 계정 확인
MariaDB [(none)]> SELECT User, Host FROM mysql.user;
+-----------------+------------------+
| User        	| Host      	|
+-----------------+------------------+
| testuser    	| %         	|
| mariadb.sys 	| localhost 	|
| mysql       	| localhost 	|
| root        	| localhost	|
+-----------------+------------------+


-게이트웨이 엔드포인트
 # 라우팅 테이블에 특정 Prefix List(S3, DynamoDB)를 목적지로 등록해서 트래픽을 VPC 엔드포인트로 보낸다.
 # 즉, 라우팅 기반 접근 방식.

-인터페이스 엔드포인트
 # 선택한 서브넷에 ENI(Elastic Network Interface, 가상 네트워크 카드)를 하나 만들어서, 
   그 ENI를 통해 AWS 서비스로 트래픽을 전달하는 방식.
 # 즉, ENI 기반 접근 방식
```

## ALB + TG + ASG User Data

**개요**: 시작 템플릿에 넣는 User Data(인스턴스 ID를 HTML로 표시).

`ALB + TG + ASG + VPC + EndPoint.txt`

```bash
	# ALB + TG + ASG + VPC + EndPoint


#!/bin/bash

sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on

TOKEN=$(curl -s -X PUT \
"http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -s \
-H "X-aws-ec2-metadata-token: $TOKEN" \
"http://169.254.169.254/latest/meta-data/instance-id")

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>EC2 Web Server</title>
</head>
<body>

    <h1>EC2 Web Server</h1>

    <h2>Instance ID</h2>

    <p>$INSTANCE_ID</p>

</body>
</html>
EOF
```

## EFS 마운트 cloud-config

**개요**: `#cloud-config`로 nfs-utils·httpd 설치 후 EFS를 `/var/www/html`에 마운트한다. EFS ID는 `fs-xxxxxxxxxxxxxxxxx`로 가렸다.

`EFS.txt`

```bash
#cloud-config
package_upgrade: true

packages:
  - nfs-utils
  - httpd

runcmd:
  - |
      set -e

      EFS_ID="fs-xxxxxxxxxxxxxxxxx"


      if [ -d /var/www/html ]; then
        mkdir -p /var/www/html.bak
        cp -a /var/www/html/.   /var/www/html.bak/ || true
      fi

      mkdir -p /var/www/html

      echo "${EFS_ID}.efs.ap-northeast-2.amazonaws.com:/ /var/www/html nfs4 defaults,_netdev 0 0" >> /etc/fstab

      mount -a

      echo "<h1>Hello world from EFS</h1>" > /var/www/html/index.html

      systemctl enable --now httpd

      mkdir -p /var/www/html/sampledir
      chown -R ec2-user:ec2-user /var/www/html/sampledir
      chmod -R o+rx /var/www/html/sampledir


[root@ip-172-31-43-124 ec2-user]# vi /var/www/html/index.html
<h1>Hello world from EFS</h1>
<h2>Hello world from EFS</h2>
<h3>Hello world from EFS</h3>


#cloud-config
package_upgrade: true

packages:
  - nfs-utils
  - httpd

runcmd:
  - |
      set -e

      # EFS ID 설정 (반드시 수정)
      EFS_ID="fs-xxxxxxxxxxxxxxxxx"


      # 기존 웹루트 백업 (있으면 보관)
      if [ -d /var/www/html ]; then
        mkdir -p /var/www/html.bak
        cp -a /var/www/html/.   /var/www/html.bak/ || true
      fi

      # EFS를 마운트할 웹루트 디렉터리 준비
      mkdir -p /var/www/html

      # fstab 등록: EFS를 /var/www/html로 마운트
      echo "${fs-xxxxxxxxxxxxxxxxx}.efs.ap-northeast-2.amazonaws.com:/ /var/www/html nfs4 defaults,_netdev 0 0" >> /etc/fstab

      # 마운트 실행
      mount -a

      # 테스트용 index.html 생성 (EFS에 저장됨)
      echo "<h1>Hello world from EFS</h1>" > /var/www/html/index.html

      # Apache 시작 및 부팅 자동시작
      systemctl enable --now httpd

      # 샘플 디렉터리 (EFS 공유)
      mkdir -p /var/www/html/sampledir
      chown -R ec2-user:ec2-user /var/www/html/sampledir
      chmod -R o+rx /var/www/html/sampledir
```
