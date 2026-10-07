# AWS 디커플링(SQS · SNS) - 실습 파일

이론 9·10과 가이드 10·11에서 사용하는 Lambda 코드, EventBridge 이벤트 패턴, SNS 메시지 봉투 예시를 모은 문서다. 계정 ID는 `123456789012`로 가렸다.

## SNS 실습 코드와 메시지 예시

**개요**: S3 업로드 알림 Lambda, EventBridge 패턴, SNS FIFO 메시지 구조.

`SNS.txt`

```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:ap-northeast-2:본인계정ID:s3-upload-sms-email"
    }
  ]
}


import urllib.parse
import boto3

sns = boto3.client("sns", region_name="ap-northeast-2")

TOPIC_ARN = "arn:aws:sns:ap-northeast-2:123456789012:s3-upload-sms-email"


def lambda_handler(event, context):

    record = event["Records"][0]

    bucket = record["s3"]["bucket"]["name"]

    key = urllib.parse.unquote_plus(
        record["s3"]["object"]["key"]
    )

    if not key.startswith("uploads/"):
        return {
            "status": "ignored",
            "key": key
        }

    message = (
        f"S3 파일 업로드 알림\n"
        f"버킷: {bucket}\n"
        f"파일: {key}"
    )

    sns.publish(
        TopicArn=TOPIC_ARN,
        Subject="S3 Upload Alert",
        Message=message
    )

    return {
        "status": "ok",
        "bucket": bucket,
        "key": key
    }


`


import urllib.parse
import boto3

# AWS SNS 서비스를 사용하기 위한 클라이언트 생성
# 서울 리전(ap-northeast-2)의 SNS를 사용
sns = boto3.client("sns", region_name="ap-northeast-2")

# 메시지를 전송할 SNS Topic의 ARN
TOPIC_ARN = "arn:aws:sns:ap-northeast-2:본인계정ID:s3-upload-sms-email"


def lambda_handler(event, context):

    # S3 이벤트에서 첫 번째 이벤트 정보를 가져옴
    record = event["Records"][0]

    # 파일이 업로드된 S3 버킷 이름 추출
    bucket = record["s3"]["bucket"]["name"]

    # 업로드된 파일의 경로 및 파일명 추출
    # S3 이벤트의 Object Key는 URL 인코딩되어 있을 수 있으므로 디코딩
    key = urllib.parse.unquote_plus(
        record["s3"]["object"]["key"]
    )

    # 업로드된 파일이 images/ 경로가 아니면 SNS 알림을 보내지 않음
    if not key.startswith("uploads/"):
        return {
            "status": "ignored",
            "key": key
        }

    # SNS로 전송할 알림 메시지 작성
    message = (
        f"S3 파일 업로드 알림\n"
        f"버킷: {bucket}\n"
        f"파일: {key}"
    )

    # 지정한 SNS Topic으로 메시지 발행
    # SNS Topic에 연결된 Email, SMS 등의 구독자에게 알림 전송
    sns.publish(
        TopicArn=TOPIC_ARN,
        Subject="S3 Upload Alert",
        Message=message
    )

    # Lambda 함수 정상 처리 결과 반환
    return {
        "status": "ok",
        "bucket": bucket,
        "key": key
    }


=========================================================================================================================================

	# event bridge


{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["버킷이름"]
    },
    "object": {
      "key": [
        { "prefix": "폴더명/" }
      ]
    }
  }
}


{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["my-s3-eb-sns-email-sms-123456789012-ap-northeast-2-an"]
    },
    "object": {
      "key": [
        { "prefix": "uploads/" }
      ]
    }
  }
}


	# Seoul Json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["my-s3-eb-sns-email-sms-123456789012-ap-northeast-2-an"]
    }
  }
}


	# Tokyo Json
{  "source": ["aws.s3"],
  "detail-type": ["Object Created"]
}


#!/bin/bash

yum update -y

yum install -y python3 python3-pip

pip3 install boto3

cd /home/ec2-user

cat << 'EOF' > sqs_consumer.py
import boto3, time

sqs = boto3.client("sqs", region_name="ap-northeast-2")

QUEUE_NAME = "my-test-queue.fifo"

queue_url = sqs.get_queue_url(
    QueueName=QUEUE_NAME
)["QueueUrl"]

while True:
    resp = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=1,
        WaitTimeSeconds=10,
        VisibilityTimeout=30
    )

    if "Messages" in resp:
        for msg in resp["Messages"]:
            print("받은 메시지:", msg["Body"])

            with open("/home/ec2-user/messages.log", "a") as f:
                f.write(msg["Body"] + "\n")

            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=msg["ReceiptHandle"]
            )
    else:
        print("메시지 없음...")
        time.sleep(1)
EOF

chown ec2-user:ec2-user /home/ec2-user/sqs_consumer.py

nohup python3 /home/ec2-user/sqs_consumer.py > /home/ec2-user/consumer.log 2>&1 &


[ec2-user@ip-172-31-24-44 ~]# python3  /home/ec2-user/sqs_consumer.py


[ec2-user@ip-172-31-24-44 ~]# ps -ef | grep sqs_consumer
root        3268    3249  0 06:42 pts/4    00:00:00 python3 /home/ec2-user/sqs_consumer.py
ec2-user    3270    2724  0 06:43 pts/3    00:00:00 grep --color=auto sqs_consumer


	# 전송 메세지

{"orderId":"1001", "status":"new"}


[root@ip-172-31-33-40 ec2-user]# python3 /home/ec2-user/sqs_consumer.py
메시지 없음...
메시지 없음...
받은 메시지: {
  "Type" : "Notification",
  "MessageId" : "05ebceec-d15b-5059-98ad-af12051ad22e",
  "SequenceNumber" : "10000000000000003000",
  "TopicArn" : "arn:aws:sns:ap-northeast-1:123456789012:my-test-topic.fifo",
  "Subject" : "order-alarm",
  "Message" : "{\"orderId\":\"1001\" , \"status\":\"NEW\"}",
  "Timestamp" : "2026-02-05T17:23:13.176Z",
  "UnsubscribeURL" : "https://sns.ap-northeast-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:ap-northeast-1:123456789012:my-test-topic.fifo:c5dd8602-0a19-40c5-86dc-d67f42f5693c"
}


# 이 메시지가 SNS에서 발행된 "알림(Notification)"이라는 뜻
Type : "Notification"

# SNS에서 발급한 메시지 고유 ID
MessageId : "73d35f07-2356-5837-9d98-3e3d7ef6e836"

# FIFO 주제에서 메시지 순서를 추적하는 번호
SequenceNumber : "10000000000000003000"
# 메시지가 발행된 SNS 주제의 ARN (고유 식별자)
TopicArn : "arn:aws:sns:ap-northeast-2:123456789012:my-test-topic.fifo"

# 메시지 발행할 때 지정한 “제목” 값
Subject : "order-alarn"

# 실제 전달하고 싶은 본문 내용
Message : "{\"orderId\":\"1001\",\"status\":\"NEW\"}"

# 메시지가 SNS에서 발행된 시간 (UTC 기준)
Timestamp : "2025-09-09T16:52:48.168Z"

# 이 링크를 누르면 구독을 취소할 수 있음
UnsubscribeURL" : "https://sns.ap-northeast-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:ap-northeast-1:123456789012:my-test-topic.fifo:c5dd8602-0a19-40c5-86dc-d67f42f5693c"


[root@ip-172-31-24-44 ec2-user]# python3  /home/ec2-user/sqs_consumer.py
/usr/local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
메시지 없음...
메시지 없음...
메시지 없음...
메시지 없음...
메시지 없음...
받은 메시지: {
  "Type" : "Notification",
  "MessageId" : "127b0a27-2681-57c7-9765-693b7337fbcb",
  "SequenceNumber" : "10000000000000016000",
  "TopicArn" : "arn:aws:sns:ap-northeast-2:123456789012:my-test-topic.fifo",
  "Subject" : "order-alarm",
  "Message" : " {\"orderId\":\"1002\" , \"status\":\"NEW2\"}",
  "Timestamp" : "2026-09-10T06:51:34.316Z",
  "UnsubscribeURL" : "https://sns.ap-northeast-2.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:ap-northeast-2:123456789012:my-test-topic.fifo:2dbc3d73-fc12-485e-a4f0-9ae3a258e2cf"
}
메시지 없음...
메시지 없음...
메시지 없음...
메시지 없음...
메시지 없음...
메시지 없음...
메시지 없음...


	#  /home/ec2-user/sqs_consumer.py  실행 중지


my-sns-alarm
my-test-group

{"orderId":"1003" , "status":"NEW3"}


my-sns-alarm
my-test-group

{"orderId":"1004" , "status":"NEW4"}


my-sns-alarm
my-test-group

{"orderId":"1005" , "status":"NEW5"}


my-sns-alarm
my-test-group

{"orderId":"1006" , "status":"NEW6"}


	#  /home/ec2-user/sqs_consumer.py  실행 
```
