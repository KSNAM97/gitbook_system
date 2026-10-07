# Amazon Route 53 - 실습 파일

이론 11과 가이드 13에서 사용하는 User Data와 확인 명령을 모은 문서다.

## Route 53 실습 메모

**개요**: Primary/Secondary EC2 User Data와 DNS 확인 명령.

`route53.txt`

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


<h1>hello Soldesk IT Academy</h1>


https://www.whatsmydns.net


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::aws-esk.com/*"
        }
    ]
}


scp  -i  My-EC2-KeyPair.pem  ./index.html  ec2-user@15.164.178.172:~/index.html
scp  -i  My-EC2-KeyPair.pem  ./index.html  ec2-user@15.164.173.40:~/index.html


15.164.178.172
15.164.173.40


-도쿄 = 1 , 싱가포르 = 1 , 미국 = 0 , 유럽 = 1
 # 최소(Minimum)	: 하나라도 0(비정상)이 있으면 결과가 0. 장애를 가장 민감하게 잡는다.
 # 최대(Maximum)	: 하나라도 1(정상)이 있으면 결과가 1. 장애를 덜 민감하게 잡는다.
 # 평균(Average) 	: 전체 검사값의 평균. 예: [1,1,0,1] --> 0.75 부분 장애율을 볼 때 유용하다.

-기간 (Period)
 # CloudWatch가 지표를 집계하는 시간 단위.
 # 보통 1분, 5분 단위로 사용 (1분이면 빠른 감지, 5분이면 안정적 감
```
