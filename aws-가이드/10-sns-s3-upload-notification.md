# SNS 실습: S3 업로드 알림 (Lambda · EventBridge)

SNS(Simple Notification Service)의 기본 개념 — Topic, 구독, Publish/Subscribe 모델, 메시지 포맷 —은 이론 10. AWS SNS에 정리되어 있다. 이 문서에서는 그 개념 중 가장 기본적인 흐름을 손으로 구성한다. S3에 파일이 업로드되면 Lambda가 이를 감지해 SNS로 알림을 발행하는 파이프라인(실습 1), EventBridge의 이벤트 패턴으로 어떤 S3 이벤트만 골라 받을지 필터링하는 방법(실습 2)을 다룬다. SNS FIFO와 SQS FIFO 소비자를 이용한 순서 보장 실습은 가이드 11. SNS FIFO 실습에서 별도로 다룬다.

## 1. 실습 1: S3 업로드를 Lambda로 감지해 SNS 알림 보내기

**목표**: S3 버킷의 `uploads/` 경로에 파일이 올라오면 Lambda가 이를 감지해 SNS Topic으로 알림 메시지를 발행하도록 구성한다.

### 1) IAM 정책 — Lambda에 SNS 발행 권한 부여

Lambda가 SNS Topic에 메시지를 발행하려면 `sns:Publish` 권한이 필요하다. 이 권한은 계정 내 모든 Topic이 아니라 실제로 사용할 Topic의 ARN 하나로 범위를 좁혀서 부여하는 것이 안전하다 — 최소 권한 원칙에 따라, Lambda 실행 역할이 다른 Topic까지 건드릴 수 없도록 `Resource`를 특정 ARN으로 고정한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:ap-northeast-2:<계정 ID>:s3-upload-sms-email"
    }
  ]
}
```

이 정책을 Lambda 실행 역할에 연결해두면, 함수 코드 안에서 `sns.publish()`를 호출할 때 별도 자격 증명 없이 이 역할의 권한으로 동작한다.

### 2) Lambda 함수 — S3 이벤트를 받아 SNS로 발행

S3 버킷에 `ObjectCreated` 이벤트가 발생하면 아래 Lambda 함수가 트리거된다.

```python
import urllib.parse
import boto3

# AWS SNS 서비스를 사용하기 위한 클라이언트 생성 (서울 리전)
sns = boto3.client("sns", region_name="ap-northeast-2")

# 메시지를 전송할 SNS Topic의 ARN
TOPIC_ARN = "arn:aws:sns:ap-northeast-2:<계정 ID>:s3-upload-sms-email"


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

    # 업로드된 파일이 uploads/ 경로가 아니면 SNS 알림을 보내지 않음
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
```

함수 구조를 단계별로 보면 다음과 같다.

- `urllib.parse.unquote_plus(...)`: S3 이벤트의 Object Key는 공백이나 한글 등이 URL 인코딩된 채로 전달될 수 있다(예: 공백이 `+`로 치환). 이를 디코딩하지 않으면 실제 파일 경로와 다른 문자열을 다루게 되므로, 로그·메시지에 정확한 경로를 남기려면 반드시 디코딩을 거쳐야 한다.
- `uploads/` 접두사 검사: S3 버킷 전체에 대해 이벤트를 받되, 실제로 알림이 필요한 것은 `uploads/` 경로로 올라온 파일뿐이라면 이런 코드 레벨 필터가 필요하다. 이 조건에 걸리지 않는 업로드(예: 임시 파일, 다른 경로)는 조용히 무시하고 종료한다.
- 메시지 구성 및 발행: 버킷 이름과 파일 경로를 담은 텍스트 메시지를 만들어 `sns.publish()`로 지정한 Topic에 발행한다. Topic에 Email, SMS 등 구독자가 연결되어 있다면 이 한 번의 `publish` 호출로 모든 구독자에게 알림이 전달된다.

S3 이벤트 → Lambda → SNS로 이어지는 가장 단순한 알림 파이프라인이다. 핵심은 IAM 정책을 Topic 단위로 최소화하는 것과, Lambda 안에서 Object Key 디코딩·경로 필터링을 빠뜨리지 않는 것이다. 경로 필터링을 Lambda 트리거 설정(S3 이벤트 알림의 Prefix 필터)에서 미리 걸 수도 있지만, 이 실습에서는 코드 레벨에서도 한 번 더 확인하는 방식을 택했다.

## 2. 실습 2: EventBridge로 S3 이벤트 필터링하기

**목표**: S3 이벤트를 Lambda가 아니라 EventBridge를 거쳐 받을 때, 이벤트 패턴(Event Pattern)으로 어떤 이벤트만 규칙에 매칭시킬지 좁혀본다.

EventBridge 규칙의 이벤트 패턴은 기본적으로 `source`(이벤트를 발생시킨 서비스), `detail-type`(이벤트 종류), `detail`(이벤트의 세부 필드) 세 부분으로 구성된다. S3 업로드 이벤트라면 `source`는 `aws.s3`, `detail-type`은 `Object Created`가 되고, `detail.bucket.name`으로 버킷을 좁히거나 `detail.object.key`에 `prefix` 조건을 걸어 특정 폴더 아래로 들어온 파일만 매칭시킬 수 있다.

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["<버킷 이름>"]
    },
    "object": {
      "key": [
        { "prefix": "<폴더명>/" }
      ]
    }
  }
}
```

이 구조를 바탕으로 필터 범위를 좁은 것부터 넓은 것까지 세 단계로 나누어 테스트했다.

### 1) 버킷 + 접두사 필터 (가장 좁은 범위)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["<S3 버킷 이름>"]
    },
    "object": {
      "key": [
        { "prefix": "uploads/" }
      ]
    }
  }
}
```

특정 버킷의 `uploads/` 경로로 올라온 파일만 매칭된다. 실습 1의 Lambda 코드 레벨 필터와 동일한 효과를 EventBridge 규칙 단계에서 미리 걸어두는 방식으로, 불필요한 이벤트가 아예 Lambda까지 전달되지 않아 호출 비용과 처리 로직을 줄일 수 있다.

### 2) 버킷만 필터 (접두사 조건 제거)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["<S3 버킷 이름>"]
    }
  }
}
```

버킷 경로와 무관하게 해당 버킷에 생성되는 모든 객체 이벤트를 받는다. 서울 리전에서 이 패턴으로 테스트해 특정 버킷의 업로드를 폴더 구분 없이 전부 수신하는 동작을 확인했다.

### 3) 필터 없음 (가장 넓은 범위)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"]
}
```

`detail` 조건이 전혀 없어 계정 내 모든 S3 버킷의 Object Created 이벤트가 매칭된다. 도쿄 리전에서 이 패턴으로 테스트해, 버킷·경로를 특정하지 않았을 때 얼마나 넓게 이벤트가 잡히는지를 좁은 패턴과 비교했다.

세 패턴을 나란히 두고 보면, 실제 운영 환경에서는 (1)처럼 버킷과 경로까지 좁힌 패턴을 쓰는 것이 일반적이다. (2), (3)처럼 범위를 넓히는 것은 이번처럼 "필터가 정확히 어디까지 걸러주는지" 동작을 검증하거나 여러 리전에서 이벤트 발생 패턴을 비교해볼 때 유용하다.

EventBridge 이벤트 패턴은 `source`/`detail-type`/`detail`의 조합으로 좁게도, 넓게도 구성할 수 있다. 필터를 규칙 단계에서 미리 걸어두면 뒤에 연결된 Lambda나 SNS로 불필요한 이벤트가 흘러가는 것을 막을 수 있으므로, 코드 레벨 필터링과 EventBridge 패턴 필터링을 함께 쓰는 것이 실무에서 흔한 조합이다.

> 관련: 이론 10. AWS SNS · 이론 9. AWS 디커플링 서비스와 Amazon SQS · 가이드 11. SNS FIFO 실습
