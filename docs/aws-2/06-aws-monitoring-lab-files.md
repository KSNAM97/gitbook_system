# AWS CloudWatch - 실습 파일

이론 6. AWS CloudWatch와 가이드 9에서 사용하는 CloudWatch Agent 설정, 로그 수집 JSON, httpd.conf를 모은 문서다.

## CloudWatch 설정 명령

**개요**: CloudWatch Agent 설치와 설정 적용 명령.

`CloudWatch 설정.txt`

```bash
	# Aphache 서버 설치
[root@ip-172-31-33-227 ec2-user]# dnf  install  -y  httpd

[root@ip-172-31-33-227 ec2-user]# service  httpd  start

[root@ip-172-31-33-227 ec2-user]# chkconfig  httpd  on


	# 로그 디렉터리 생성
[root@ip-172-31-33-227 ec2-user]# mkdir -p  /var/log/www/error
[root@ip-172-31-33-227 ec2-user]# mkdir -p  /var/log/www/access


[root@ip-172-31-33-227 ec2-user]# ls  -l  /etc/httpd/conf
total 32
-rw-r--r--. 1 root root 12532 Jun  9 06:00 httpd.conf
-rw-r--r--. 1 root root 13430 Jun  9 06:01 magic


	# Aphache 설정파일 변경
[root@ip-172-31-33-227 ec2-user]# cp  httpd.conf  /etc/httpd/conf/
cp: overwrite '/etc/httpd/conf/httpd.conf'? y


	# Aphache 서버 재실행
[root@ip-172-31-33-227 ec2-user]# service  httpd   restart
Redirecting to /bin/systemctl restart httpd.service


	# CloudWatch Agent 설치
[root@ip-172-31-33-227 ec2-user]# dnf  install  -y  amazon-cloudwatch-agent


	# Agent 설정 적용
[root@ip-172-31-33-227 ec2-user]# /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config -m ec2 -s -c file:/home/ec2-user/EnablePHPCloudwatchlog.json


# -a fetch-config	: 설정을 가져오거나 로컬 파일을 읽어 반영하는 액션
# -m ec2		: 실행 환경이 EC2임을 지정
# -s		: 설정 적용 후 서비스를 즉시 시작
# -c file:/home/ec2-user/EnablePHPCloudwatchlog.json	: 업로드한 에이전트 설정 JSON을 사용한다
# amazon-cloudwatch-agent-ctl이 EnablePHPCloudwatchlog.json을 읽어서 설정을 적용하고, 그 설정대로 CloudWatch Agent가 동작


[root@ip-172-31-33-227 ec2-user]# systemctl  start  amazon-cloudwatch-agent
```

## EC2 설정 기록

**개요**: Apache·PHP 설치와 로그 수집 준비 과정.

`ec2_설정.txt`

```bash
# 1) 권한 상승
sudo -s                                  # 루트 셸로 전환한다


# 2) Apache HTTP Server 설치 및 기동
dnf install httpd -y              	# httpd 패키지를 설치한다
service httpd start                 # Apache를 즉시 시작한다 (AL2023에서는 systemctl start httpd 권장)
chkconfig httpd on             	# 부팅 시 자동 시작 설정이다 (AL2023에서는 systemctl enable httpd 권장)


# 3) 로그 디렉터리 준비
sudo mkdir -p /var/log/www/error         # 애플리케이션 에러 로그 디렉터리를 생성
sudo mkdir -p /var/log/www/access        # 애플리케이션 액세스 로그 디렉터리를 생성


# 4) Apache 설정 반영
cp httpd.conf /etc/httpd/conf		# 현재 디렉터리의 httpd.conf를 시스템 설정 위치로 복사

service httpd restart			# 설정을 반영하기 위해 Apache를 재시작


# 5) CloudWatch Agent 설치
sudo dnf install amazon-cloudwatch-agent -y		# CloudWatch 에이전트를 설치


# 6) 에이전트 설정 적용 및 시작
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/home/ec2-user/EnablePHPCloudwatchlog.json
	# -a fetch-config	: 설정을 가져오거나 로컬 파일을 읽어 반영하는 액션
	# -m ec2		: 실행 환경이 EC2임을 지정
	# -s		: 설정 적용 후 서비스를 즉시 시작
	# -c file:/home/ec2-user/EnablePHPCloudwatchlog.json	: 업로드한 에이전트 설정 JSON을 사용한다
	# amazon-cloudwatch-agent-ctl이 EnablePHPCloudwatchlog.json을 읽어서 설정을 적용하고,  그 설정대로 CloudWatch Agent가 동작

# 7) 서비스 상태 보장 (위 -s가 시작까지 해주지만, 시작 상태를 보장하기 위해 한 번 더 실행)
sudo systemctl start amazon-cloudwatch-agent 


	{ $.status = "404" }


	# 상태/헬스체크
-StatusCheckFailed (0 또는 1)
 : 인스턴스 상태 점검 실패의 종합 결과(아래 두 개 중 하나라도 실패하면 1).

-StatusCheckFailed_System (0/1)
 : AWS 인프라 측 문제로 인스턴스에 접근 불가(호스트 장애, 네트워크/전원 등).

-StatusCheckFailed_Instance (0/1)
 : OS 내부 문제로 에이전트/네트워크 응답 없음(커널 패닉, 과부하 등).


	# CPU & 버스트 크레딧(T 계열 전용)

-CPUUtilization (%)
 : vCPU 사용률(평균). 지속적으로 높으면 과부하/스케일 필요 신호.

-CPUCreditUsage (크레딧/기간)
 : 해당 기간 동안 쓴 CPU 크레딧 개수.

-CPUCreditBalance (크레딧)
 : 남아 있는 CPU 크레딧 잔액. 0 근처면 성능이 베이스라인으로 제한될 수 있음.

-CPUSurplusCreditBalance (크레딧, Unlimited 모드)
 : 잔액 0인데 추가로 쓴 초과 크레딧 누적치(아직 요금 청구 전).

-CPUSurplusCreditsCharged (크레딧, Unlimited 모드)
 : 요금 청구된 초과 크레딧(비용 발생).


	# 네트워크

-NetworkIn / NetworkOut (Bytes)
 : 수신/송신 바이트 수. 대역폭 사용량 트렌드 파악.

-NetworkPacketsIn / NetworkPacketsOut (패킷 수)
 : 초당/기간당 패킷 개수. 작은 페이로드 트래픽 특징 파악에 유용.


	# EBS(블록 스토리지) 관측치

-EBSReadBytes / (EBSWriteBytes) (Bytes)
 : EBS로부터 읽은(쓴) 바이트 수
  (WriteBytes는 화면에 없지만 보통 짝으로 존재, "얼마나 많이 읽고/썼냐"는 용량(바이트) 합계)

-EBSReadOps / EBSWriteOps (IOPS, 요청 수)
 : 읽기/쓰기 I/O 요청 개수 ("몇 번 읽고/썼냐"는 요청(횟수)


1) status 값이 문자열 "404"인 로그만 매칭.
{ $.status = "404" }

3) status 값이 400, 404, 500 중 하나라도 맞으면 매칭.
{ $.status = 400 || $.status = 404 || $.status = 500 }

3) status 값이 4xx 전 범위 매칭(숫자로 기록될 때만 가능).
{ $.status >= 400 && $.status < 500 }

4) status 값이 500이면서 메서드가 GET인 로그만.
{ $.status = 500 && $.method = "GET" }


-이름 필터링 : My-WEB-404-NAMESPACE
 # 이 Metric Filter 자체의 이름
 # CloudWatch Logs에서 어떤 필터인지 구분하기 위한 관리용 이름
 # 실제 지표 이름은 아님

-지표 네임스페이스 : 
 # 생성되는 Custom Metric이 들어갈 분류/폴더 같은 개념
 # CloudWatch → Metrics에서 이 Namespace 아래에 지표가 생성됨

-지표 이름 : WEB-404
 # 실제 생성되는 CloudWatch Metric의 이름
 # 나중에 Alarm 만들 때 이 지표를 선택함

-지표 값 : 1
 # 필터 패턴과 일치하는 로그 1건마다 지표에 더할 값
 # 즉 404 로그 1개 발견 : 1

404 로그 1개  -->  +1
404 로그 1개  -->  +1
404 로그 1개  -->  +1
404 로그 1개  -->  +1
404 로그 1개  -->  +1

결과
test-404-metric = 5
```

## CloudWatch Agent 설정 JSON

**개요**: Apache 액세스·에러 로그를 CloudWatch Logs로 보내는 Agent 설정(주석 포함).

`EnablePHPCloudwatchlog - 주석.json`

```json
{
	"agent": {
		// CloudWatch Agent 자체의 기본 수집 주기
		// 60초마다 메트릭을 수집
		"metrics_collection_interval": 60
	},

	"logs": {
		"logs_collected": {
			"files": {
				"collect_list": [

					{
						// Apache 에러 로그 파일 경로
						// 해당 경로 아래의 모든 파일을 수집
						"file_path": "/var/log/www/error/*",

						// CloudWatch Logs에 생성할 로그 그룹 이름
						"log_group_name": "apache/error",

						// 로그 스트림 이름
						// EC2 Instance ID를 로그 스트림 이름으로 사용
						"log_stream_name": "[{instance_id}]"
					},

					{
						// Apache 접근 로그 파일 경로
						"file_path": "/var/log/www/access/*",

						// CloudWatch Logs의 로그 그룹 이름
						"log_group_name": "apache/access",

						// EC2 Instance ID를 로그 스트림 이름으로 사용
						"log_stream_name": "[{instance_id}]"
					}
				]
			}
		}
	},

	"metrics": {

		// CloudWatch에 전송되는 메트릭에
		// EC2 관련 정보를 Dimension으로 추가
		"append_dimensions": {

			// EC2가 Auto Scaling Group에 속한 경우
			// 해당 Auto Scaling Group 이름 추가
			"AutoScalingGroupName": "${aws:AutoScalingGroupName}",

			// EC2가 사용하는 AMI ID 추가
			"ImageId": "${aws:ImageId}",

			// EC2 Instance ID 추가
			"InstanceId": "${aws:InstanceId}",

			// EC2 Instance Type 추가
			// 예: t3.micro, t3.small
			"InstanceType": "${aws:InstanceType}"
		},

		"metrics_collected": {

			"disk": {

				// 디스크 사용률(%) 수집
				"measurement": [
					"used_percent"
				],

				// 60초마다 디스크 사용률 수집
				"metrics_collection_interval": 60,

				// 모든 디스크/파일시스템을 대상으로 수집
				"resources": [
					"*"
				]
			},

			"mem": {

				// 메모리 사용률(%) 수집
				"measurement": [
					"mem_used_percent"
				],

				// 60초마다 메모리 사용률 수집
				"metrics_collection_interval": 60
			},

			"statsd": {

				// StatsD 메트릭을 60초 단위로 집계
				"metrics_aggregation_interval": 60,

				// StatsD 데이터를 10초마다 수집
				"metrics_collection_interval": 10,

				// UDP 8125 포트에서 StatsD 메트릭 수신
				"service_address": ":8125"
			}
		}
	}
}
```

## httpd.conf

**개요**: CloudWatch Agent가 수집하기 쉽도록 로그 형식을 조정한 설정.

`httpd.conf`

```apache
ServerRoot "/etc/httpd"

Listen 80

Include conf.modules.d/*.conf

User apache
Group apache

ServerAdmin root@localhost

<Directory />
    AllowOverride none
    Require all denied
</Directory>

DocumentRoot "/var/www/html"

<Directory "/var/www">
    AllowOverride None
    Require all granted
</Directory>

<Directory "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

<IfModule dir_module>
    DirectoryIndex index.html
</IfModule>

<Files ".ht*">
    Require all denied
</Files>

ErrorLog "/var/log/www/error/error_log"
ErrorLogFormat "{\"time\":\"%{%usec_frac}t\", \"function\" : \"[%-m:%l]\",\"process\" : \"[pid%P]\" ,\"message\" : \"%M\"}"

LogLevel warn

<IfModule log_config_module>

    LogFormat "%h %l %u %t \"%r\" %>s %b" common
    LogFormat "{ \"time\":\"%{%Y-%m-%d}tT%{%T}t.%{msec_frac}tZ\", \"process\":\"%D\", \"filename\":\"%f\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\",\"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\",\"referer\":\"%{Referer}i\"}" cloudwatch

    <IfModule logio_module>
    </IfModule>

    CustomLog "logs/access_log" common
    CustomLog "/var/log/www/access/access_log" cloudwatch

</IfModule>

<IfModule alias_module>

    ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"

</IfModule>

<Directory "/var/www/cgi-bin">
    AllowOverride None
    Options None
    Require all granted
</Directory>

<IfModule mime_module>

    TypesConfig /etc/mime.types

    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz

    AddType text/html .shtml
    AddOutputFilter INCLUDES .shtml

</IfModule>

AddDefaultCharset UTF-8

<IfModule mime_magic_module>

    MIMEMagicFile conf/magic

</IfModule>

EnableSendfile on

<IfModule mod_http2.c>
    Protocols h2 h2c http/1.1
</IfModule>

IncludeOptional conf.d/*.conf
```
