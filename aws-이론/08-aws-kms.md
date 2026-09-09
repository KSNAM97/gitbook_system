# AWS KMS(Key Management Service)

## 1. KMS 개요

AWS KMS는 AWS에서 사용하는 암호화 키를 생성하고 관리하는 서비스다. AWS 서비스의 데이터를 암호화할 때 사용하는 키를 중앙에서 관리할 수 있다.

**대표적으로 다음과 같은 서비스와 연동해서 사용한다.**

- Amazon EBS 볼륨 암호화
- Amazon S3 객체 암호화
- Amazon RDS 데이터 암호화
- Amazon EFS 파일 암호화

KMS를 사용하면 사용자가 직접 암호화 키 파일을 서버에 저장하지 않고 AWS에서 안전하게 관리할 수 있다.

## 2. KMS Key

KMS에서 생성하는 키를 KMS Key라고 하며, 실제 데이터를 암호화하거나 데이터 키를 보호하는 데 사용한다.

실제 KMS 키 ID는 다음과 같은 형식이라 그대로는 기억하기 어렵다.

```
<KMS 키 ID 예시 형식>: 1234abcd-12ab-34cd-56ef-1234567890ab
```

그래서 별칭(Alias)을 붙여서 사용한다.

- `my-app-key`
- `rds-key`
- `s3-encryption-key`

> 관련: 이론 4. AWS S3 · 이론 5. AWS RDS · 이론 6. AWS CloudWatch
