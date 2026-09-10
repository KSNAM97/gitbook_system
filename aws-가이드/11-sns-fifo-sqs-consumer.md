# SNS FIFO 실습: SQS FIFO 소비자 배포

SNS FIFO와 SQS FIFO의 순서 보장·중복 제거 개념(Message Group ID, Deduplication ID 등)은 이론 10. AWS SNS에, SQS의 큐/FIFO/Long Polling 기본 개념은 이론 9. AWS 디커플링 서비스와 Amazon SQS에 정리되어 있다. S3 업로드를 Lambda로 감지해 SNS로 알림을 보내는 기본 파이프라인 실습은 가이드 10. SNS 실습: S3 업로드 알림에서 별도로 다룬다.

이 문서에서는 SNS FIFO Topic에 발행된 메시지가 SQS FIFO 큐를 통해 실제로 어떻게 도착하는지 직접 확인한다. EC2 위에 SQS FIFO 큐를 폴링하는 Python 소비자를 배포하고(실습 1), 실제로 발행된 SNS FIFO 알림이 소비자에게 어떤 형태로 도착하는지 필드 단위로 확인한다(실습 2).

## 1. 실습 1: EC2에 SQS FIFO 소비자 배포하기

**목표**: EC2 인스턴스가 부팅되면서 자동으로 SQS FIFO 큐를 폴링하는 Python 소비자를 설치·기동하도록 user-data 스크립트를 구성한다.

### 1) user-data 스크립트

```bash
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
```

스크립트를 단계별로 보면 다음과 같다.

- `yum update -y` / `yum install -y python3 python3-pip` / `pip3 install boto3`: 인스턴스 부팅 시점에 Python 실행 환경과 AWS SDK(boto3)를 준비한다. 이 세 줄이 없으면 이후 Python 스크립트가 동작할 수 없다.
- `cat << 'EOF' > sqs_consumer.py ... EOF`: 여기 문서(Here Document) 문법으로 Python 소스 코드 전체를 파일로 생성한다. 별도의 파일을 EC2에 미리 올려둘 필요 없이, user-data 스크립트 하나에 설치 과정과 실행할 코드를 동시에 담을 수 있다. `'EOF'`처럼 따옴표로 감싸면 내부의 `$`, 백틱 등을 쉘이 치환하지 않고 그대로 파일에 써준다.
- 폴링 루프(`sqs.receive_message`): `WaitTimeSeconds=10`은 Long Polling을 의미한다 — 큐가 비어 있어도 즉시 빈 응답을 반환하지 않고 최대 10초까지 메시지 도착을 기다렸다가 응답하므로, `WaitTimeSeconds=0`(Short Polling)으로 짧은 간격에 계속 요청을 날리는 것보다 API 호출 횟수를 크게 줄일 수 있다. `VisibilityTimeout=30`은 메시지를 받아간 뒤 30초 동안은 다른 소비자에게 같은 메시지가 보이지 않도록 숨기는 시간으로, 그 사이에 처리와 삭제가 끝나야 중복 처리를 막을 수 있다.
- 처리 후 삭제(`sqs.delete_message`): 메시지를 로그 파일에 기록한 뒤 `ReceiptHandle`로 큐에서 명시적으로 삭제한다. SQS는 메시지를 받았다고 자동으로 지워주지 않으므로, 처리가 끝난 메시지를 직접 삭제해야 같은 메시지가 다시 수신되지 않는다.
- `nohup ... &`: 인스턴스가 부팅되는 동안 실행된 이 스크립트가 세션 종료 후에도 백그라운드에서 계속 동작하도록 한다.

### 2) 실제 기동 확인

user-data로 자동 기동되는지와 별개로, 콘솔 세션에서 직접 스크립트를 실행하고 프로세스가 떠 있는지 확인했다.

```text
[ec2-user@<EC2 프라이빗 호스트명> ~]# python3 /home/ec2-user/sqs_consumer.py

[ec2-user@<EC2 프라이빗 호스트명> ~]# ps -ef | grep sqs_consumer
root        3268    3249  0 06:42 pts/4    00:00:00 python3 /home/ec2-user/sqs_consumer.py
ec2-user    3270    2724  0 06:43 pts/3    00:00:00 grep --color=auto sqs_consumer
```

`ps -ef | grep sqs_consumer` 결과에 `python3 /home/ec2-user/sqs_consumer.py` 프로세스가 살아있는 것이 보이면, 소비자가 정상적으로 백그라운드에서 폴링을 이어가고 있다는 뜻이다.

user-data 스크립트로 패키지 설치·코드 생성·백그라운드 실행까지 한 번에 자동화할 수 있다. 폴링 루프에서는 Long Polling(`WaitTimeSeconds`)으로 호출 횟수를 줄이고, `VisibilityTimeout` 동안 처리를 끝낸 뒤 명시적으로 `delete_message`를 호출하는 삭제 패턴을 지키는 것이 SQS 소비자 구현의 기본이다.

## 2. 실습 2: SNS 알림 메시지 수신 확인하기

**목표**: SNS FIFO Topic에 발행한 메시지가 SQS FIFO 큐를 통해 실제로 어떤 형태로 도착하는지 확인하고, 필드별 의미를 정리한다.

### 1) 테스트 메시지 발행

아래와 같은 간단한 JSON을 원본 메시지로 SNS Topic에 발행했다.

```json
{"orderId":"1001", "status":"new"}
```

### 2) 소비자가 실제로 수신한 로그

실습 1에서 띄워둔 SQS FIFO 소비자가 이 메시지를 수신한 결과는 다음과 같다.

```text
[root@<EC2 프라이빗 호스트명> ec2-user]# python3 /home/ec2-user/sqs_consumer.py
메시지 없음...
메시지 없음...
받은 메시지: {
  "Type" : "Notification",
  "MessageId" : "05ebceec-d15b-5059-98ad-af12051ad22e",
  "SequenceNumber" : "10000000000000003000",
  "TopicArn" : "arn:aws:sns:ap-northeast-1:<계정 ID>:my-test-topic.fifo",
  "Subject" : "order-alarm",
  "Message" : "{\"orderId\":\"1001\" , \"status\":\"NEW\"}",
  "Timestamp" : "2026-02-05T17:23:13.176Z",
  "UnsubscribeURL" : "https://sns.ap-northeast-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:ap-northeast-1:<계정 ID>:my-test-topic.fifo:c5dd8602-0a19-40c5-86dc-d67f42f5693c"
}
```

처음 두 번의 "메시지 없음..." 출력은 실습 1에서 구성한 Long Polling 루프가 메시지가 도착할 때까지 `WaitTimeSeconds=10` 만큼 대기와 재시도를 반복하고 있다는 뜻이며, 이후 실제 메시지가 도착하자 SNS가 감싼 형태 그대로 로그에 출력됐다. Topic 이름이 `my-test-topic.fifo`로 `.fifo` 접미사를 갖고 있다는 점에서 이 Topic이 SNS FIFO로 생성되었음을 확인할 수 있다.

### 3) 필드별 의미

- `Type`: `"Notification"` — 이 메시지가 SNS에서 발행된 "알림(Notification)"이라는 것을 나타낸다.
- `MessageId`: SNS가 이 메시지에 부여한 고유 ID.
- `SequenceNumber`: FIFO Topic에서 메시지의 순서를 추적하기 위한 번호로, 이론 10. AWS SNS에서 다룬 Message Sequence Number가 실제로 이 필드에 담겨 온다.
- `TopicArn`: 메시지가 발행된 SNS Topic의 ARN(고유 식별자).
- `Subject`: 발행 시 지정한 "제목" 값 — 여기서는 `sns.publish()` 호출 시 넘긴 `Subject`와 동일한 역할이다.
- `Message`: 실제로 전달하려던 본문. 원본 JSON(`{"orderId":"1001" , "status":"NEW"}`)이 SNS 봉투 안에 이스케이프 처리된 문자열로 한 번 더 감싸져 들어있는 것을 볼 수 있다.
- `Timestamp`: 메시지가 SNS에서 발행된 시각(UTC 기준).
- `UnsubscribeURL`: 이 링크를 호출하면 해당 구독을 취소할 수 있다.

이렇게 SQS가 SNS 메시지를 원본 그대로가 아니라 `Type`, `MessageId`, `TopicArn` 등의 메타데이터로 한 번 감싼 "봉투(envelope)" 형태로 전달하는 것이 SNS→SQS 구독의 기본 동작이다. 이론 10. AWS SNS에서 다룬 SNS 메시지 래핑 개념이 여기서 실제 로그로 확인된 셈이다. 만약 이 봉투 없이 원본 `Message` 내용만 그대로 받고 싶다면 구독 설정에서 **Raw Message Delivery**를 활성화하면 되는데, 이 경우 소비자 코드에서 `msg["Body"]`를 더 이상 SNS 포맷으로 파싱할 필요 없이 원본 JSON으로 바로 다룰 수 있게 된다.

SNS FIFO Topic으로 발행한 메시지는 구독자(SQS FIFO)에게 그대로 전달되는 것이 아니라 `Type`/`MessageId`/`SequenceNumber`/`TopicArn` 등을 포함한 봉투 형태로 감싸져 도착하며, `SequenceNumber` 필드를 통해 FIFO 특유의 순서 추적이 실제로 동작함을 확인할 수 있었다. 소비자 코드를 작성할 때는 `msg["Body"]`가 SNS 봉투인지, 원본 메시지인지(Raw Message Delivery 여부)를 먼저 확인하고 그에 맞게 파싱해야 한다는 점을 기억해둘 만하다.

## 3. 실습 3: Message Group ID로 순서 보장 확인하기

**목표**: 같은 Topic·같은 Message Group ID로 여러 메시지를 연달아 발행했을 때, 소비자가 발행한 순서 그대로 메시지를 받는지 확인한다.

이론 10. AWS SNS에서 다룬 Message Group ID는 "같은 그룹 ID를 가진 메시지끼리만 순서가 보장된다"는 개념이었다. 이를 검증하기 위해 같은 Topic(`my-sns-alarm`)과 같은 Message Group ID(`my-test-group`)로 4건의 메시지를 순서대로 발행했다.

```json
// Topic: my-sns-alarm / Message Group ID: my-test-group
{"orderId":"1003", "status":"NEW3"}
{"orderId":"1004", "status":"NEW4"}
{"orderId":"1005", "status":"NEW5"}
{"orderId":"1006", "status":"NEW6"}
```

같은 Message Group ID로 보낸 메시지이므로, 이론에서 설명한 대로 SQS FIFO는 이 그룹 안에서는 반드시 하나씩 순차적으로만 처리한다 — 즉 `orderId 1003`이 소비자에서 처리·삭제되기 전까지는 `1004`가 먼저 넘어갈 수 없다. 실습 2에서 확인한 봉투 형태(`Type`/`MessageId`/`SequenceNumber` 등) 그대로 4건이 도착하며, `SequenceNumber` 값이 각 메시지마다 증가하는 것으로 발행 순서를 재확인할 수 있다.

같은 소비자 스크립트를 다시 실행하면 Long Polling 특성상 처음 몇 번은 "메시지 없음..." 이 반복되다가, 발행한 4건이 `1003 → 1004 → 1005 → 1006` 순서 그대로 하나씩 수신된다. 만약 서로 다른 Message Group ID로 나누어 보냈다면 그룹 간에는 순서가 섞일 수 있지만, 같은 그룹 안에서는 예외 없이 발행 순서가 유지된다는 점이 이 실습의 핵심이다.

> 관련: 이론 10. AWS SNS · 이론 9. AWS 디커플링 서비스와 Amazon SQS · 가이드 10. SNS 실습: S3 업로드 알림
