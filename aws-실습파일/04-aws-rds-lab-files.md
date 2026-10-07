# AWS RDS - 실습 파일

이론 5. AWS RDS와 가이드 8(WordPress 3-Tier)에서 사용하는 SQL·IAM DB 인증 명령, User Data, wp-config 템플릿을 모은 문서다. DB 비밀번호와 엔드포인트는 가렸다.

## RDS 계정 생성과 IAM DB 인증

**개요**: 일반 계정·IAM 인증 계정 생성, `generate-db-auth-token`, SSL 접속.

`RDS.txt`

```bash
# 테스트 유저 생성
CREATE USER 'testuser'@'%' IDENTIFIED BY 'password';				# 일반 계정

CREATE USER 'testuser' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS'; 	# IAM 사용자 인증 계정

ALTER USER 'admin'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';	# admin 계정도 IAM 사용자 인증을 사용시

-CREATE USER 'testuser'	# 새로운 DB 사용자 이름을 testuser 로 생성

-IDENTIFIED WITH AWSAuthenticationPlugin
 # 일반적인 비밀번호 기반 인증이 아니라 AWSAuthenticationPlugin 플러그인을 이용해서 인증
 # 즉, 이 사용자는 IAM 인증 토큰으로만 로그인 가능

-AS 'RDS'		# 플러그인 옵션으로 RDS를 지정, RDS에서 제공하는 IAM 인증 방식을 사용한는 의미


1) Bastion 서버에서 AWS 자격증명(Access Key 또는 IAM Role)을 이용해 RDS 접속용 임시 인증 토큰을 발급
 # aws rds generate-db-auth-token ...
 # AWS는 Bastion 서버가 가진 IAM 자격증명을 확인
 # 해당 IAM 사용자 또는 Role에 rds-db:connect 권한이 있는지 검사
 # 권한이 있으면 약 15분 동안 사용할 수 있는 임시 인증 토큰을 발급

2) Bastion 서버는 발급받은 임시 인증 토큰을 비밀번호처럼 사용해서 RDS에 접속
 # mysql -h <RDS주소> -u testuser -p
 # 비밀번호 입력란에 일반 비밀번호 대신 발급받은 토큰을 입력
 # RDS는 토큰이 정상적으로 AWS에서 발급된 것인지 확인
 # 토큰이 위조되지 않았는지, 15분이 지나 만료되지 않았는지 확인
 # 해당 IAM 사용자 또는 Role이 testuser로 접속할 권한이 있는지 확인
 # 모든 조건이 맞으면 RDS 접속 허용
 # 즉, 일반 비밀번호를 미리 저장해두는 방식이 아니라 AWS IAM 권한으로 
    짧게 사용할 수 있는 임시 비밀번호를 발급받아서 RDS에 접속하는 방식


select user, host, plugin from mysql.user where user='testuser';
# user	host	plugin
testuser	%	AWSAuthenticationPlugin


-user
# DB 계정 이름
# 여기서는 testuser

-host
# %는 어느 호스트에서든 접속 가능하다는 의미
# 단, 실제 접속 가능 여부는 RDS 보안그룹과 네트워크 설정에도 영향을 받음

-plugin
# 인증 방식을 의미
# AWSAuthenticationPlugin이면 IAM Database Authentication 사용


 # testuser 사용자에게 db_my_test의 모든 권한 부여
GRANT ALL PRIVILEGES ON my-db.* TO 'testuser'@'%';


	# 클라우드 쉘
aws rds generate-db-auth-token  --hostname <rds-endpoint>  --port 3306  --region ap-northeast-2  --username testuser

aws rds generate-db-auth-token  --hostname <rds-endpoint>  --port 3306  --region ap-northeast-2  --username testuser

<rds-endpoint>:3306/?Action=connect&DBUser=testuser&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=<...>
```

## 3-Tier User Data (주석 포함)

**개요**: WordPress 서버를 자동 구성하는 User Data.

`demo_3_tier_userdata - 주석.txt`

```bash
#!/bin/bash

# Apache 웹 서버 설치
dnf install httpd -y

# PHP 8.2, PHP-MySQL 모듈, MariaDB 클라이언트 설치
dnf install -y php8.2 php8.2-mysqlnd mariadb105

# Apache 웹 서버 재시작
systemctl restart httpd

# Apache를 부팅 시 자동 시작하도록 설정
chkconfig httpd on

# /var/www/html 디렉터리 소유자를 ec2-user로 변경
chown -R ec2-user:ec2-user /var/www/html

# WordPress를 설치할 디렉터리 생성
mkdir -p /var/www/html/wordpress

# EC2 Instance Metadata Service(IMDSv2) 접근을 위한 토큰 발급
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# 현재 EC2의 가용 영역(AZ)을 조회하여 EFS DNS 주소 생성 후 /etc/fstab에 등록
# {efs_id} 부분은 자신의 EFS ID로 변경
echo "$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
"http://169.254.169.254/latest/meta-data/placement/availability-zone").{efs_id}.efs.ap-northeast-2.amazonaws.com:/    \
/var/www/html/wordpress nfs4    defaults" >> /etc/fstab

# /etc/fstab에 등록된 EFS 마운트
mount -a

# WordPress 최신 버전 다운로드
wget https://wordpress.org/latest.tar.gz

# WordPress 압축 파일 해제
tar -xzf latest.tar.gz

# 압축 해제된 WordPress 디렉터리를 /var/www/html로 복사
cp wordpress /var/www/html -r

# WordPress 디렉터리 소유자를 ec2-user로 변경
chown ec2-user /var/www/html/wordpress

# WordPress 디렉터리에 다른 사용자 읽기 권한 부여
chmod -R o+r /var/www/html/wordpress

# S3에서 wp-config.php 등의 파일을 WordPress 디렉터리로 복사
# {s3_url} 부분은 자신의 S3 파일 경로로 변경
aws s3 cp {s3_url} /var/www/html/wordpress --region ap-northeast-2
```

## WordPress DB 수정 SQL

**개요**: WordPress 사이트 URL 등을 DB에서 수정하는 명령 기록.

`wordpress-DB 수정.txt`

```bash
[root@ip-10-0-15-68 ec2-user]# mariadb -h [RDS EndPoint 주소] -u admin -p
Enter password:


MySQL [(none)]> SHPW DATABASES;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'SHPW DATABASES' at line 1
MySQL [(none)]> SHOW DATABASES;
+---------------------------+
| Database           		|
+---------------------------+
| information_schema 	|
| mysql              		|
| performance_schema 	|
| sys                		|
| wordpress          		|
+---------------------------+


MySQL [(none)]> USE wordpress;


MySQL [wordpress]> SHOW TABLES;
+---------------------------+
| Tables_in_wordpress   	|
+---------------------------+
| wp_commentmeta        	|
| wp_comments           	|
| wp_links              	|
| wp_options            	|
| wp_postmeta           	|
| wp_posts              	|
| wp_term_relationships	|
| wp_term_taxonomy      	|
| wp_termmeta           	|
| wp_terms              	|
| wp_usermeta           	|
| wp_users              	|
+---------------------------+


MySQL [wordpress]> SELECT  option_name,  option_value  FROM  wp_options  WHERE  option_name  IN ('siteurl','home');
+-----------------+-------------------------------------------------------------------------+
| option_name 	| option_value                                                             			|
+-----------------+-------------------------------------------------------------------------+
| home          	| http://my-3tier-wordpress-ALB-287717672.ap-northeast-2.elb.amazonaws.com	|
| siteurl         	| http://my-3tier-wordpress-ALB-287717672.ap-northeast-2.elb.amazonaws.com	|
+-----------------+-------------------------------------------------------------------------+


MySQL [wordpress]> UPDATE wp_options SET option_value='http://my-3tier-wordpress-ALB-287717672.ap-northeast-2.elb.amazonaws.com/wordpress'
 WHERE option_name='home';


MySQL [wordpress]> UPDATE wp_options SET option_value='http://my-3tier-wordpress-ALB-287717672.ap-northeast-2.elb.amazonaws.com/wordpress'
 WHERE option_name='siteurl';


MySQL [wordpress]> SELECT option_name, option_value  FROM wp_options  WHERE option_name IN ('siteurl','home');
+------------------+---------------------------------------------------------------------------------+
| option_name 	| option_value                                                                       			|
+------------------+---------------------------------------------------------------------------------+
| home        	| http://my-3tier-wordpress-ALB-287717672.ap-northeast-2.elb.amazonaws.com/wordpress |
| siteurl     	| http://my-3tier-wordpress-ALB-287717672.ap-northeast-2.elb.amazonaws.com/wordpress |
+------------------+---------------------------------------------------------------------------------+


MySQL [wordpress]> SELECT ID, post_title, post_date FROM wp_posts WHERE post_type='post' ORDER BY ID DESC;
+----+---------------------+----------------------------+
| ID | post_title                 	| post_date           		|
+----+---------------------+----------------------------+
| 11 | 이거슨 첫글입니다.      	| 2026-09-08 07:08:38 	|
|  8 | new title               	| 2026-09-08 06:48:46 	|
|  6 | 새로은 글 작성         	| 2026-09-08 06:47:40 	|
|  5 | Auto Draft              	| 2026-09-08 04:15:55 	|
|  1 | Hello world!            	| 2026-09-08 04:15:40 	|
+----+---------------------+----------------------------+


MySQL [wordpress]> SELECT ID, post_title, post_content post_date FROM wp_posts WHERE post_type='post' ORDER BY ID DESC;
+----+----------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ID | post_title                 | post_date                                                                                                                                                                                 |
+----+----------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 11 | 이거슨 첫글입니다.         | <!-- wp:paragraph -->
<p>아마 첫글일 꺼에요</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->                                                              |
|  8 | new title                  | <!-- wp:paragraph -->
<p>new content</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph {"className":"is-style-text-display"} -->
<p class="is-style-text-display"></p>
<!-- /wp:paragraph --> |
|  6 | 새로은 글 작성             | <!-- wp:paragraph -->
<p>새로운글을 작성합니다.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->                                                          |
|  5 | Auto Draft                 |                                                                                                                                                                                           |
|  1 | Hello world!               | <!-- wp:paragraph -->
<p>Welcome to WordPress. This is your first post. Edit or delete it, then start writing!</p>
<!-- /wp:paragraph -->                                                 |
+----+----------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## wp-config.php 템플릿

**개요**: DB 접속 정보를 넣는 wp-config.php(주석 포함).

`wp-config - 주석.php`

```php
<?php

// 데이터베이스 이름
define( 'DB_NAME', '[데이터베이스 이름]' );

// 데이터베이스 사용자명
define( 'DB_USER', '[DB 사용자명]' );

// 데이터베이스 사용자 비밀번호
define( 'DB_PASSWORD', '<DB_PASSWORD>' );

// 데이터베이스 호스트
define( 'DB_HOST', '[RDS 엔드포인트 주소]' );

// 데이터베이스 문자셋
define( 'DB_CHARSET', 'utf8' );

// 데이터베이스 정렬 방식
define( 'DB_COLLATE', '' );

// 파일 시스템 접근 방식
// direct : WordPress가 파일을 직접 생성/수정
define( 'FS_METHOD', 'direct' );

// 보안 키와 솔트 값
// 실제 서비스에서는 임의의 복잡한 문자열로 변경 권장
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );

// 데이터베이스 테이블 접두사
// 하나의 DB에 여러 WordPress를 설치할 경우 구분할 때 사용
$table_prefix = 'wp_';

// 디버그 모드
// true  : 오류 및 디버그 정보 확인
// false : 일반 운영 모드
define( 'WP_DEBUG', false );

// WordPress 설치 경로 설정
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

// WordPress 핵심 파일 불러오기
require_once ABSPATH . 'wp-settings.php';
```
