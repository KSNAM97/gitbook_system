# Terraform - IAM

이 문서는 Terraform으로 IAM 사용자, 역할, 정책과 AssumeRole을 구성하는 방법과 실습을 정리한다.

## 1. IAM(Identity and Access Management) 서비스 이해

AWS 인프라를 운영할 때 가장 중요한 요소 중 하나는 보안 관리이다. AWS 환경에서는 EC2, S3, RDS, VPC와 같은 다양한 리소스를 생성하고 운영하게 되며, 이러한 리소스에 대해 누가 접근할 수 있는지, 어떤 작업을 수행할 수 있는지를 명확하게 관리해야 한다. 이러한 접근 권한을 체계적으로 관리하기 위해 AWS에서 제공하는 서비스가 바로 IAM다.

IAM은 AWS 리소스에 대한 접근을 안전하게 제어하기 위한 서비스로, 사용자와 권한을 관리하는 기능을 제공한다. AWS를 실제로 운영하다 보면 여러 사람이 하나의 AWS 환경을 함께 사용하는 경우가 많다. 예를 들어 개발자, 운영자, 관리자와 같은 다양한 역할을 가진 사용자들이 동일한 AWS 계정을 사용하게 된다. 이때 모든 사람이 동일한 권한을 가지게 되면 보안 문제가 발생할 수 있으며, 실수로 중요한 리소스를 삭제하거나 변경하는 위험도 발생할 수 있다.

이러한 문제를 해결하기 위해 IAM을 사용하면 사용자별로 권한을 분리하고, 필요한 권한만 부여하여 AWS 리소스에 대한 접근을 안전하게 관리할 수 있다. IAM을 통해 조직의 보안 정책을 적용하고, 사용자별 역할에 맞는 권한을 설정함으로써 AWS 환경을 보다 안전하게 운영할 수 있다.

## 2. IAM 유저 기본 이해

IAM은 Identity and Access Management의 약자로, AWS 리소스에 대한 접근을 안전하게 제어할 수 있도록 해주는 서비스이다. AWS 계정에서 개별 사용자 및 그들의 권한을 관리하기 위해 사용된다.

AWS 계정은 기본적으로 하나의 Root 계정을 가지고 시작하지만, 실제 운영 환경에서는 Root 계정을 직접 사용하는 것은 보안상 매우 위험하다. Root 계정은 AWS 계정의 모든 권한을 가지고 있기 때문에 실수나 보안 사고가 발생할 경우 큰 문제가 발생할 수 있다. 따라서 일반적인 운영 환경에서는 Root 계정을 사용하기보다는 IAM 유저를 생성하여 필요한 권한을 부여하는 방식으로 AWS 환경을 관리하는 것이 권장된다.

IAM을 사용하면 사용자별로 AWS 리소스 접근 권한을 설정할 수 있으며, 이를 통해 조직의 역할에 맞는 권한 관리 체계를 구축할 수 있다.

## 3. IAM 유저란?

IAM 유저는 AWS에 접근할 수 있는 개별 사용자 계정이다. 예를 들어, 회사 내에서 개발자, 운영자, 관리자 등 각각의 역할을 가진 사람들을 IAM 유저로 관리할 수 있다.

각 유저는 개별적인 자격 증명(예: Access Key, Secret Key)을 가지며, 이를 통해 AWS 콘솔 또는 API를 통해 AWS 서비스에 접근할 수 있다.

IAM 유저는 실제 사람을 의미하는 경우도 있지만, 프로그램이나 자동화 시스템을 위해 생성되는 경우도 있다. 예를 들어 CI/CD 시스템이나 자동화 스크립트가 AWS 리소스를 제어해야 하는 경우 IAM 유저를 생성하여 접근 권한을 부여할 수 있다.

또한 IAM 유저는 다음과 같은 다양한 방식으로 AWS에 접근할 수 있다.

* AWS Management Console
  * 웹 브라우저를 통해 AWS 관리 콘솔에 로그인하여 리소스를 관리하는 방식
* AWS CLI
  * 터미널에서 명령어를 사용하여 AWS 리소스를 제어하는 방식
* SDK 및 API
  * 프로그램 코드에서 AWS 서비스를 호출하여 리소스를 관리하는 방식

이러한 다양한 접근 방식에서도 동일하게 IAM 유저의 권한이 적용되며, IAM 정책을 통해 어떤 작업을 수행할 수 있는지가 결정된다.

## 4. IAM 유저의 목적과 특징

IAM 유저를 사용하는 가장 큰 목적은 AWS 리소스에 대한 접근 권한을 사용자 단위로 관리하기 위함이다.

## 5. 개별 권한 관리

IAM 유저마다 서로 다른 권한을 부여할 수 있기 때문에 조직 내 역할에 맞는 권한 설정이 가능하다. 예를 들어 개발자는 EC2 인스턴스를 생성하거나 수정할 수 있는 권한이 필요할 수 있지만, 결제 관리나 계정 설정과 같은 권한은 필요하지 않을 수 있다. 이러한 경우 IAM 정책을 사용하여 필요한 권한만 부여할 수 있다.

## 6. 액세스 관리

유저가 AWS 콘솔, CLI, SDK 등을 통해 특정 서비스에 접근할 수 있는지 제어할 수 있다. 예를 들어 특정 사용자는 S3 버킷을 읽기만 할 수 있도록 설정할 수 있으며, 다른 사용자는 EC2 인스턴스를 생성하거나 시작할 수 있도록 설정할 수 있다. 이러한 방식으로 서비스별 접근 권한을 세밀하게 관리할 수 있다.

## 7. 정책의 적용

유저가 어떤 AWS 리소스에 어떤 작업을 할 수 있는지를 정의하는 IAM 정책을 유저에게 부여하여 권한을 관리한다.

일반적으로 모든 사람이 root 계정으로 AWS에 접근하는 것은 보안상 위험하기 때문에, IAM 유저를 생성하고 각 유저에게 적절한 권한을 부여하는 것이 좋다.

Root 계정은 AWS 계정 전체에 대한 모든 권한을 가지고 있기 때문에 일상적인 운영 작업에는 사용하지 않는 것이 좋다. 대신 IAM 유저를 생성하고 각 사용자에게 필요한 권한만 부여하는 방식으로 AWS 환경을 운영하는 것이 보안 측면에서 훨씬 안전하다.

## 8. IAM 정책(Policy)과 권한

IAM 유저의 권한은 IAM 정책(Policy)을 통해 제어된다. 정책은 JSON 형식의 문서로, 유저가 어떤 AWS 서비스에 어떤 작업을 수행할 수 있는지를 정의한다.

IAM 정책은 AWS 권한 관리의 핵심 요소이며, 정책을 통해 사용자가 접근할 수 있는 서비스와 수행 가능한 작업을 매우 세밀하게 설정할 수 있다. IAM 정책을 적절하게 설계하면 조직 내 사용자들이 필요한 작업만 수행하도록 제한할 수 있으며, 이를 통해 AWS 환경의 보안을 강화할 수 있다.

## 9. IAM 정책의 구성 요소

* 정책은 크게 두 가지로 나눌 수 있습니다.

## 10. AWS 관리형 정책

AWS에서 사전 구성하여 제공하는 표준 정책이다. 예를 들어, AmazonS3ReadOnlyAccess 정책은 S3 버킷에 읽기 전용 권한을 부여한다. 이러한 정책은 AWS에서 일반적인 사용 사례를 기반으로 만들어져 있기 때문에 초기 권한 설정 시 매우 편리하게 사용할 수 있다.

## 11. 사용자 정의 정책(커스텀 정책)

유저가 필요에 따라 직접 작성하는 정책으로, JSON 형식으로 구성된다. 특정 리소스에 대해서만 권한을 부여하거나 특정 작업만 허용하는 등 보다 세밀한 권한 제어가 가능하다.

## 12. 정책의 세부 요소

* Effect
  * 권한을 Allow하거나 Deny할지 정의
  * Allow는 해당 작업을 허용하는 것을 의미하며, Deny는 해당 작업을 명시적으로 거부하는 것을 의미
* Action
  * 어떤 AWS 서비스와 작업(Action)에 대한 권한을 부여할 것인지를 정의한다. (예: s3:ListBucket)
* Resource
  * 정책이 적용될 리소스를 명시합니다. (예: 특정 S3 버킷의 ARN)
* Condition
  * 정책의 조건을 설정하여 특정 상황에서만 권한을 부여할 수 있다.
  * 예를 들어 특정 IP 주소에서만 접근을 허용하거나 특정 시간대에만 권한을 허용하는 등의 조건을 설정할 수 있다.

정책 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-bucket"
    }
  ]
}
```

## 13. AssumeRole 권한 부여

* AssumeRole은 AWS에서 다른 IAM Role의 권한을 임시로 빌려서 사용하는 기능이다.
* 쉽게 말하면 "평소에는 권한이 없지만, 필요한 순간에 특정 역할(Role)의 권한을 잠깐 빌려서 작업한다."라는 의미
  * 예를 들어 회사에 직원이 있다고 가정한다.
* 일반 직원
  * 서버를 삭제할 권한이 없음
* 관리자 역할
  * 서버 삭제 권한이 있음
* 일반 직원이 필요한 순간 관리자 역할을 잠깐 빌림
  * 관리자 권한으로 작업
* 작업이 끝나면
  * 다시 일반 직원 권한으로 돌아감
* AWS에서는 이 과정을 AssumeRole이라고 한다.
  * IAM Role
* IAM Role은 AWS에서 사용하는 권한 묶음이다.
* 사용자에게 직접 권한을 계속 주는 것이 아니라, "이 역할을 사용하면 이런 작업을 할 수 있다." 라는 식으로 권한을 만들어 놓는다.
* 예를 들어 다음과 같은 Role을 만들 수 있다.
  * S3 읽기 Role
  * EC2 관리 Role
  * RDS 관리 Role
  * Lambda 실행 Role
  * 관리자 Role
* 그리고 필요할 때 해당 Role을 사용한다.

## 14. AssumeRole의 핵심

* AssumeRole을 사용하려면 기본적으로 2가지를 이해해야 한다.
  *
    1. 누가 이 Role을 사용할 수 있는가?
  *
    2. 이 Role을 사용하면 무엇을 할 수 있는가?
* AWS에서는 이것을 각각 다음과 같이 설정한다.
  * Trust Policy: 누가 Role을 사용할 수 있는지 설정
  * Permission Policy: Role이 어떤 AWS 작업을 할 수 있는지 설정

## 15. 1) Trust Policy

* Trust Policy는 "누가 이 Role을 사용할 수 있는가?" 를 정의한다.
* 예를 들어 EC2가 Role을 사용할 수 있도록 설정하려면 다음과 같다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

"Effect": "Allow"
```

* 허용한다.

```
"Principal": {
  "Service": "ec2.amazonaws.com"
}
```

* EC2 서비스가 이 Role을 사용할 수 있다.

```
"Action": "sts:AssumeRole"
```

* EC2가 이 Role의 권한을 임시로 가져가서 사용할 수 있도록 허용한다.
* 즉 전체 의미는 다음과 같다.
  * EC2 서비스가 이 IAM Role의 권한을 임시로 Assume해서 사용할 수 있도록 허용한다.

## 16. 2) Permission Policy

* Permission Policy는 "이 Role을 사용하면 무엇을 할 수 있는가?" 를 정의한다.
* 예를 들어 Role에 S3 읽기 권한을 부여하려면 다음과 같이 설정할 수 있다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": "*"
    }
  ]
}
```

* 이 Role을 사용하면 S3 Bucket 목록 확인과 Object 조회가 가능하다.
* 정리하면 다음과 같다.
* Trust Policy
  * 누가 Role을 사용할 수 있는가?
* Permission Policy
  * Role을 사용해서 무엇을 할 수 있는가?

## 17. STS (Security Token Service)

* STS는 Security Token Service의 약자이다.
* AWS에서 임시 보안 자격 증명을 만들어주는 서비스이다.
* AssumeRole을 실행하면 STS가 임시로 다음과 같은 정보를 발급한다.
  * AccessKeyId
  * SecretAccessKey
  * SessionToken
  * Expiration
* 이 자격 증명은 영구적으로 사용하는 것이 아니라 일정 시간이 지나면 만료된다.
* 따라서 AssumeRole은 장기간 사용하는 Access Key보다 안전하게 권한을 사용할 수 있다.
  * AssumeRole 동작 과정
* 예를 들어 EC2가 S3에 접근한다고 가정한다.
  * EC2 --> IAM Role 사용 --> STS AssumeRole --> 임시 자격 증명 발급 --> S3 접근
* EC2 안에 AWS Access Key를 직접 저장하지 않아도 된다.

## 18. EC2에서 사용하는 AssumeRole 예제

* EC2가 IAM Role을 사용할 수 있도록 하는 Trust Policy이다.
  * Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

"Principal": { "Service": "ec2.amazonaws.com" },
 #  이 Role을 사용할 수 있는 대상을 지정한다.
 #  "Service": "ec2.amazonaws.com" 는 EC2 서비스를 의미한다.

"Action": "sts:AssumeRole"
 #  EC2가 이 Role을 Assume할 수 있도록 허용한다.
 #  쉽게 말하면 EC2가 이 IAM Role의 권한을 임시로 빌려서 사용할 수 있게 한다.
```

* 하지만 이것만으로 EC2가 S3를 사용할 수 있는 것은 아니다.
* Role에 실제 S3 권한도 추가해야 한다.
  * Permission Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": "*"
    }
  ]
}
```

* 따라서 두 정책의 역할은 다르다.
* Trust Policy
  * EC2가 Role을 사용할 수 있도록 허용
* Permission Policy
  * Role을 사용해서 S3 작업을 할 수 있도록 허용

## 19. 다른 AWS 계정의 Role 사용

* AssumeRole은 다른 AWS 계정의 Role도 사용할 수 있다.
* 예를 들어 다음과 같이 계정이 있다고 가정한다.
* 회사 A 계정
  * 123456789012
* 회사 B 계정
  * 123456789012
* 회사 A 사용자가 회사 B의 Role을 사용하도록 설정할 수 있다.
* 회사 B Role의 Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

* 이처럼 다른 AWS 계정의 Role을 사용하는 방식을 Cross Account AssumeRole이라고 한다.

## 20. AWS CLI로 AssumeRole 실행

* AWS CLI에서도 직접 AssumeRole을 실행할 수 있다.

```bash
aws sts assume-role \
```

* -role-arn arn:aws:iam::123456789012:role/MyRole \\
* -role-session-name student-session
* 실행하면 대략 다음과 같은 임시 자격 증명이 반환된다.

```json
{
  "Credentials": {
    "AccessKeyId": "ASIAXXXXXXXXXXXX",
    "SecretAccessKey": "xxxxxxxxxxxxxxxx",
    "SessionToken": "xxxxxxxxxxxxxxxx",
    "Expiration": "2026-09-26T13:00:00Z"
  }
}
```

* AccessKeyId
  * 임시로 발급된 Access Key이다.
  * 기존 IAM User의 Access Key와 다른 새로운 임시 Access Key이다.
* SecretAccessKey
  * 위의 임시 Access Key와 함께 사용하는 Secret Key이다.
* SessionToken
  * AssumeRole로 발급된 임시 인증 정보라는 것을 증명하는 Token
  * AssumeRole에서는 AccessKeyId와 SecretAccessKey만 사용하는 것이 아니라 SessionToken까지 같이 사용
* Expiration
  * 임시 자격 증명이 언제 만료되는지를 나타낸다.
  * 이 시간이 지나면 해당 임시 Access Key는 더 이상 사용할 수 없다.
  * AWS CLI Profile에서 AssumeRole 사용
* AWS CLI에서는 Profile을 이용하면 편하게 사용할 수 있다.

```hcl
[profile assume-user]
role_arn = arn:aws:iam::123456789012:role/MyRole
source_profile = default
region = ap-northeast-2
```

* 사용 : aws s3 ls --profile assume-user
* 동작 과정
  * default Profile 인증 --> MyRole AssumeRole --> 임시 권한 획득 --> S3 조회
  * Terraform에서 AssumeRole 사용
* Terraform AWS Provider에서도 AssumeRole을 사용할 수 있다.
* Terraform이 AWS 리소스를 생성할 때 현재 로그인한 사용자 권한을 그대로 사용하는 것이 아니라 특정 IAM Role의 권한을 임시로 빌려서 작업하도록 만들 수 있다.

```hcl
provider "aws" {
  region  = "ap-northeast-2"
  profile = "my-profile"
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformRole"
  }
}
```

* assume\_role
  * 인증한 뒤 다른 IAM Role의 권한을 임시로 사용
* role\_arn
  * Terraform이 사용할 IAM Role을 지정
* 동작 과정
  * my-profile로 AWS 인증 --> TerraformRole Assume --> STS에서 임시 권한 발급 --> TerraformRole의 권한으로 AWS 리소스 생성
* profile
  * 누구로 먼저 로그인할지 지정
* assume\_role
  * 로그인 후 어떤 Role의 권한을 사용할지 지정다.

## 21. 실습: IAM(Identity and Access Management) 서비스 이해

AWS 인프라를 운영할 때 가장 중요한 요소 중 하나는 보안 관리이다. AWS 환경에서는 EC2, S3, RDS, VPC와 같은 다양한 리소스를 생성하고 운영하게 되며, 이러한 리소스에 대해 누가 접근할 수 있는지, 어떤 작업을 수행할 수 있는지를 명확하게 관리해야 한다. 이러한 접근 권한을 체계적으로 관리하기 위해 AWS에서 제공하는 서비스가 바로 IAM다.

IAM은 AWS 리소스에 대한 접근을 안전하게 제어하기 위한 서비스로, 사용자와 권한을 관리하는 기능을 제공한다. AWS를 실제로 운영하다 보면 여러 사람이 하나의 AWS 환경을 함께 사용하는 경우가 많다. 예를 들어 개발자, 운영자, 관리자와 같은 다양한 역할을 가진 사용자들이 동일한 AWS 계정을 사용하게 된다. 이때 모든 사람이 동일한 권한을 가지게 되면 보안 문제가 발생할 수 있으며, 실수로 중요한 리소스를 삭제하거나 변경하는 위험도 발생할 수 있다.

이러한 문제를 해결하기 위해 IAM을 사용하면 사용자별로 권한을 분리하고, 필요한 권한만 부여하여 AWS 리소스에 대한 접근을 안전하게 관리할 수 있다. IAM을 통해 조직의 보안 정책을 적용하고, 사용자별 역할에 맞는 권한을 설정함으로써 AWS 환경을 보다 안전하게 운영할 수 있다.

## 22. 실습: IAM 유저 기본 이해

IAM은 Identity and Access Management의 약자로, AWS 리소스에 대한 접근을 안전하게 제어할 수 있도록 해주는 서비스이다. AWS 계정에서 개별 사용자 및 그들의 권한을 관리하기 위해 사용된다.

AWS 계정은 기본적으로 하나의 Root 계정을 가지고 시작하지만, 실제 운영 환경에서는 Root 계정을 직접 사용하는 것은 보안상 매우 위험하다. Root 계정은 AWS 계정의 모든 권한을 가지고 있기 때문에 실수나 보안 사고가 발생할 경우 큰 문제가 발생할 수 있다. 따라서 일반적인 운영 환경에서는 Root 계정을 사용하기보다는 IAM 유저를 생성하여 필요한 권한을 부여하는 방식으로 AWS 환경을 관리하는 것이 권장된다.

IAM을 사용하면 사용자별로 AWS 리소스 접근 권한을 설정할 수 있으며, 이를 통해 조직의 역할에 맞는 권한 관리 체계를 구축할 수 있다.

## 23. 실습: IAM 유저란?

IAM 유저는 AWS에 접근할 수 있는 개별 사용자 계정이다. 예를 들어, 회사 내에서 개발자, 운영자, 관리자 등 각각의 역할을 가진 사람들을 IAM 유저로 관리할 수 있다.

각 유저는 개별적인 자격 증명(예: Access Key, Secret Key)을 가지며, 이를 통해 AWS 콘솔 또는 API를 통해 AWS 서비스에 접근할 수 있다.

IAM 유저는 실제 사람을 의미하는 경우도 있지만, 프로그램이나 자동화 시스템을 위해 생성되는 경우도 있다. 예를 들어 CI/CD 시스템이나 자동화 스크립트가 AWS 리소스를 제어해야 하는 경우 IAM 유저를 생성하여 접근 권한을 부여할 수 있다.

또한 IAM 유저는 다음과 같은 다양한 방식으로 AWS에 접근할 수 있다.

* AWS Management Console
  * 웹 브라우저를 통해 AWS 관리 콘솔에 로그인하여 리소스를 관리하는 방식
* AWS CLI
  * 터미널에서 명령어를 사용하여 AWS 리소스를 제어하는 방식
* SDK 및 API
  * 프로그램 코드에서 AWS 서비스를 호출하여 리소스를 관리하는 방식

이러한 다양한 접근 방식에서도 동일하게 IAM 유저의 권한이 적용되며, IAM 정책을 통해 어떤 작업을 수행할 수 있는지가 결정된다.

## 24. 실습: IAM 유저의 목적과 특징

IAM 유저를 사용하는 가장 큰 목적은 AWS 리소스에 대한 접근 권한을 사용자 단위로 관리하기 위함이다.

## 25. 실습: 개별 권한 관리

IAM 유저마다 서로 다른 권한을 부여할 수 있기 때문에 조직 내 역할에 맞는 권한 설정이 가능하다. 예를 들어 개발자는 EC2 인스턴스를 생성하거나 수정할 수 있는 권한이 필요할 수 있지만, 결제 관리나 계정 설정과 같은 권한은 필요하지 않을 수 있다. 이러한 경우 IAM 정책을 사용하여 필요한 권한만 부여할 수 있다.

## 26. 실습: 액세스 관리

유저가 AWS 콘솔, CLI, SDK 등을 통해 특정 서비스에 접근할 수 있는지 제어할 수 있다. 예를 들어 특정 사용자는 S3 버킷을 읽기만 할 수 있도록 설정할 수 있으며, 다른 사용자는 EC2 인스턴스를 생성하거나 시작할 수 있도록 설정할 수 있다. 이러한 방식으로 서비스별 접근 권한을 세밀하게 관리할 수 있다.

## 27. 실습: 정책의 적용

유저가 어떤 AWS 리소스에 어떤 작업을 할 수 있는지를 정의하는 IAM 정책을 유저에게 부여하여 권한을 관리한다.

일반적으로 모든 사람이 root 계정으로 AWS에 접근하는 것은 보안상 위험하기 때문에, IAM 유저를 생성하고 각 유저에게 적절한 권한을 부여하는 것이 좋다.

Root 계정은 AWS 계정 전체에 대한 모든 권한을 가지고 있기 때문에 일상적인 운영 작업에는 사용하지 않는 것이 좋다. 대신 IAM 유저를 생성하고 각 사용자에게 필요한 권한만 부여하는 방식으로 AWS 환경을 운영하는 것이 보안 측면에서 훨씬 안전하다.

## 28. 실습: IAM 정책(Policy)과 권한

IAM 유저의 권한은 IAM 정책(Policy)을 통해 제어된다. 정책은 JSON 형식의 문서로, 유저가 어떤 AWS 서비스에 어떤 작업을 수행할 수 있는지를 정의한다.

IAM 정책은 AWS 권한 관리의 핵심 요소이며, 정책을 통해 사용자가 접근할 수 있는 서비스와 수행 가능한 작업을 매우 세밀하게 설정할 수 있다. IAM 정책을 적절하게 설계하면 조직 내 사용자들이 필요한 작업만 수행하도록 제한할 수 있으며, 이를 통해 AWS 환경의 보안을 강화할 수 있다.

## 29. 실습: IAM 정책의 구성 요소

* 정책은 크게 두 가지로 나눌 수 있습니다.

## 30. 실습: AWS 관리형 정책

AWS에서 사전 구성하여 제공하는 표준 정책이다. 예를 들어, AmazonS3ReadOnlyAccess 정책은 S3 버킷에 읽기 전용 권한을 부여한다. 이러한 정책은 AWS에서 일반적인 사용 사례를 기반으로 만들어져 있기 때문에 초기 권한 설정 시 매우 편리하게 사용할 수 있다.

## 31. 실습: 사용자 정의 정책(커스텀 정책)

유저가 필요에 따라 직접 작성하는 정책으로, JSON 형식으로 구성된다. 특정 리소스에 대해서만 권한을 부여하거나 특정 작업만 허용하는 등 보다 세밀한 권한 제어가 가능하다.

## 32. 실습: 정책의 세부 요소

* Effect
  * 권한을 Allow하거나 Deny할지 정의
  * Allow는 해당 작업을 허용하는 것을 의미하며, Deny는 해당 작업을 명시적으로 거부하는 것을 의미
* Action
  * 어떤 AWS 서비스와 작업(Action)에 대한 권한을 부여할 것인지를 정의한다. (예: s3:ListBucket)
* Resource
  * 정책이 적용될 리소스를 명시합니다. (예: 특정 S3 버킷의 ARN)
* Condition
  * 정책의 조건을 설정하여 특정 상황에서만 권한을 부여할 수 있다.
  * 예를 들어 특정 IP 주소에서만 접근을 허용하거나 특정 시간대에만 권한을 허용하는 등의 조건을 설정할 수 있다.

정책 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-bucket"
    }
  ]
}
```

* IAM --> 사용자 --> 사용자 생성

![IAM  -->  사용자  -->  사용자 생성 화면](<../.gitbook/assets/1 (1).png>)

* 사용자 이름: sol-user
* AWS Management Console에 대한 사용자 액세스 권한 제공

![AWS Management Console에 대한 사용자 액세스 권한 제공 화면](<../.gitbook/assets/2 (1).png>)

* 권한 옵션: 직접 정책 연결
* 권한 정책: AdministratorAccess

![권한 정책: AdministratorAccess 화면](<../.gitbook/assets/3 (1).png>)

![권한 정책: AdministratorAccess 화면](<../.gitbook/assets/4 (1).png>)

* IAM --> 정책 --> 정책 생성

![IAM  -->  정책  -->  정책 생성 화면](../.gitbook/assets/5.png)

* 서비스 : S3

![서비스 : S3 화면](../.gitbook/assets/6.png)

* 모든 읽기 권한

![모든 읽기 권한 화면](../.gitbook/assets/7.png)

* 리소스: 모든

![리소스: 모든 화면](../.gitbook/assets/8.png)

* 정책 이름 : S3\_Read\_only

![정책 이름 : S3\_Read\_only 화면](../.gitbook/assets/9.png)

* 생성된 정책 확인

![생성된 정책 확인 화면](../.gitbook/assets/10.png)

## 33. 실습: 사용자 계정 생성시 정책 적용

![생성된 정책 확인 화면](<../.gitbook/assets/1 (1).png>)

* 사용자 이름: sol-user1
* 사용자 지정 암호: soluser1iam!@#$

![사용자 지정 암호: soluser1iam!@#$ 화면](../.gitbook/assets/11.png)

![사용자 지정 암호: soluser1iam!@#$ 화면](../.gitbook/assets/12.png)

![사용자 지정 암호: soluser1iam!@#$ 화면](../.gitbook/assets/13.png)

```
IAM  -->  사용자  -->  sol-user1
```

![IAM  -->  사용자  -->  sol-user1 화면](../.gitbook/assets/14.png)

## 34. 실습: AssumeRole 권한 부여

* AssumeRole은 AWS에서 다른 IAM Role의 권한을 임시로 빌려서 사용하는 기능이다.
* 쉽게 말하면 "평소에는 권한이 없지만, 필요한 순간에 특정 역할(Role)의 권한을 잠깐 빌려서 작업한다."라는 의미
  * 예를 들어 회사에 직원이 있다고 가정한다.
* 일반 직원
  * 서버를 삭제할 권한이 없음
* 관리자 역할
  * 서버 삭제 권한이 있음
* 일반 직원이 필요한 순간 관리자 역할을 잠깐 빌림
  * 관리자 권한으로 작업
* 작업이 끝나면
  * 다시 일반 직원 권한으로 돌아감
* AWS에서는 이 과정을 AssumeRole이라고 한다.
  * IAM Role
* IAM Role은 AWS에서 사용하는 권한 묶음이다.
* 사용자에게 직접 권한을 계속 주는 것이 아니라, "이 역할을 사용하면 이런 작업을 할 수 있다." 라는 식으로 권한을 만들어 놓는다.
* 예를 들어 다음과 같은 Role을 만들 수 있다.
  * S3 읽기 Role
  * EC2 관리 Role
  * RDS 관리 Role
  * Lambda 실행 Role
  * 관리자 Role
* 그리고 필요할 때 해당 Role을 사용한다.

## 35. 실습: AssumeRole의 핵심

* AssumeRole을 사용하려면 기본적으로 2가지를 이해해야 한다.
  *
    1. 누가 이 Role을 사용할 수 있는가?
  *
    2. 이 Role을 사용하면 무엇을 할 수 있는가?
* AWS에서는 이것을 각각 다음과 같이 설정한다.
  * Trust Policy: 누가 Role을 사용할 수 있는지 설정
  * Permission Policy: Role이 어떤 AWS 작업을 할 수 있는지 설정

## 36. 실습: 1) Trust Policy

* Trust Policy는 "누가 이 Role을 사용할 수 있는가?" 를 정의한다.
* 예를 들어 EC2가 Role을 사용할 수 있도록 설정하려면 다음과 같다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

"Effect": "Allow"
```

* 허용한다.

```
"Principal": {
  "Service": "ec2.amazonaws.com"
}
```

* EC2 서비스가 이 Role을 사용할 수 있다.

```
"Action": "sts:AssumeRole"
```

* EC2가 이 Role의 권한을 임시로 가져가서 사용할 수 있도록 허용한다.
* 즉 전체 의미는 다음과 같다.
  * EC2 서비스가 이 IAM Role의 권한을 임시로 Assume해서 사용할 수 있도록 허용한다.

## 37. 실습: 2) Permission Policy

* Permission Policy는 "이 Role을 사용하면 무엇을 할 수 있는가?" 를 정의한다.
* 예를 들어 Role에 S3 읽기 권한을 부여하려면 다음과 같이 설정할 수 있다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": "*"
    }
  ]
}
```

* 이 Role을 사용하면 S3 Bucket 목록 확인과 Object 조회가 가능하다.
* 정리하면 다음과 같다.
* Trust Policy
  * 누가 Role을 사용할 수 있는가?
* Permission Policy
  * Role을 사용해서 무엇을 할 수 있는가?

## 38. 실습: STS (Security Token Service)

* STS는 Security Token Service의 약자이다.
* AWS에서 임시 보안 자격 증명을 만들어주는 서비스이다.
* AssumeRole을 실행하면 STS가 임시로 다음과 같은 정보를 발급한다.
  * AccessKeyId
  * SecretAccessKey
  * SessionToken
  * Expiration
* 이 자격 증명은 영구적으로 사용하는 것이 아니라 일정 시간이 지나면 만료된다.
* 따라서 AssumeRole은 장기간 사용하는 Access Key보다 안전하게 권한을 사용할 수 있다.
  * AssumeRole 동작 과정
* 예를 들어 EC2가 S3에 접근한다고 가정한다.
  * EC2 --> IAM Role 사용 --> STS AssumeRole --> 임시 자격 증명 발급 --> S3 접근
* EC2 안에 AWS Access Key를 직접 저장하지 않아도 된다.

## 39. 실습: EC2에서 사용하는 AssumeRole 예제

* EC2가 IAM Role을 사용할 수 있도록 하는 Trust Policy이다.
  * Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

"Principal": { "Service": "ec2.amazonaws.com" },
 #  이 Role을 사용할 수 있는 대상을 지정한다.
 #  "Service": "ec2.amazonaws.com" 는 EC2 서비스를 의미한다.

"Action": "sts:AssumeRole"
 #  EC2가 이 Role을 Assume할 수 있도록 허용한다.
 #  쉽게 말하면 EC2가 이 IAM Role의 권한을 임시로 빌려서 사용할 수 있게 한다.
```

* 하지만 이것만으로 EC2가 S3를 사용할 수 있는 것은 아니다.
* Role에 실제 S3 권한도 추가해야 한다.
  * Permission Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": "*"
    }
  ]
}
```

* 따라서 두 정책의 역할은 다르다.
* Trust Policy
  * EC2가 Role을 사용할 수 있도록 허용
* Permission Policy
  * Role을 사용해서 S3 작업을 할 수 있도록 허용

## 40. 실습: 다른 AWS 계정의 Role 사용

* AssumeRole은 다른 AWS 계정의 Role도 사용할 수 있다.
* 예를 들어 다음과 같이 계정이 있다고 가정한다.
* 회사 A 계정
  * 123456789012
* 회사 B 계정
  * 123456789012
* 회사 A 사용자가 회사 B의 Role을 사용하도록 설정할 수 있다.
* 회사 B Role의 Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

* 이처럼 다른 AWS 계정의 Role을 사용하는 방식을 Cross Account AssumeRole이라고 한다.

## 41. 실습: AWS CLI로 AssumeRole 실행

* AWS CLI에서도 직접 AssumeRole을 실행할 수 있다.

```bash
aws sts assume-role \
```

* -role-arn arn:aws:iam::123456789012:role/MyRole \\
* -role-session-name student-session
* 실행하면 대략 다음과 같은 임시 자격 증명이 반환된다.

```json
{
  "Credentials": {
    "AccessKeyId": "ASIAXXXXXXXXXXXX",
    "SecretAccessKey": "xxxxxxxxxxxxxxxx",
    "SessionToken": "xxxxxxxxxxxxxxxx",
    "Expiration": "2026-09-26T13:00:00Z"
  }
}
```

* AccessKeyId
  * 임시로 발급된 Access Key이다.
  * 기존 IAM User의 Access Key와 다른 새로운 임시 Access Key이다.
* SecretAccessKey
  * 위의 임시 Access Key와 함께 사용하는 Secret Key이다.
* SessionToken
  * AssumeRole로 발급된 임시 인증 정보라는 것을 증명하는 Token
  * AssumeRole에서는 AccessKeyId와 SecretAccessKey만 사용하는 것이 아니라 SessionToken까지 같이 사용
* Expiration
  * 임시 자격 증명이 언제 만료되는지를 나타낸다.
  * 이 시간이 지나면 해당 임시 Access Key는 더 이상 사용할 수 없다.
  * AWS CLI Profile에서 AssumeRole 사용
* AWS CLI에서는 Profile을 이용하면 편하게 사용할 수 있다.

```hcl
[profile assume-user]
role_arn = arn:aws:iam::123456789012:role/MyRole
source_profile = default
region = ap-northeast-2
```

* 사용 : aws s3 ls --profile assume-user
* 동작 과정
  * default Profile 인증 --> MyRole AssumeRole --> 임시 권한 획득 --> S3 조회
  * Terraform에서 AssumeRole 사용
* Terraform AWS Provider에서도 AssumeRole을 사용할 수 있다.
* Terraform이 AWS 리소스를 생성할 때 현재 로그인한 사용자 권한을 그대로 사용하는 것이 아니라 특정 IAM Role의 권한을 임시로 빌려서 작업하도록 만들 수 있다.

```hcl
provider "aws" {
  region  = "ap-northeast-2"
  profile = "my-profile"
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformRole"
  }
}
```

* assume\_role
  * 인증한 뒤 다른 IAM Role의 권한을 임시로 사용
* role\_arn
  * Terraform이 사용할 IAM Role을 지정
* 동작 과정
  * my-profile로 AWS 인증 --> TerraformRole Assume --> STS에서 임시 권한 발급 --> TerraformRole의 권한으로 AWS 리소스 생성
* profile
  * 누구로 먼저 로그인할지 지정
* assume\_role
  * 로그인 후 어떤 Role의 권한을 사용할지 지정다.

## 42. 실습: 실습

EX) IAM User는 S3 접근 불가, Role Assume 후 S3 접근 가능

* 실습 목표 : 이번 실습의 목표는 다음 4가지를 직접 확인하는 것이다.
  * 첫째, IAM User 자체에는 S3 권한이 없으면 S3에 접근할 수 없다는 점
  * 둘째, IAM Role에는 별도의 권한을 붙일 수 있다는 점
  * 셋째, IAM User가 Role을 Assume하면 자신의 원래 권한 대신 Role 권한으로 동작한다는 점
  * 넷째, 콘솔과 CLI 모두 결국 AssumeRole 구조로 동작한다는 점

## 43. 실습: 1단계 S3 버킷 만들기

* 먼저 Role 권한으로 조회할 대상이 있어야 하므로 S3 버킷을 생성

![먼저 Role 권한으로 조회할 대상이 있어야 하므로 S3 버킷을 생성 화면](../.gitbook/assets/15.png)

* 버킷 이름 : iam-assume-bucket-123456789012 (버킷 생성)

![버킷 이름 : iam-assume-bucket-123456789012 버킷 생성 화면](../.gitbook/assets/16.png)

* S3에 파일 1개 업로드

![S3에 파일 1개 업로드 화면](../.gitbook/assets/17.png)

## 44. 실습: 2 단계 IAM User 만들기

* S3 권한이 없는 사용자 생성
* IAM --> 사용자 --> 사용자 생성

![IAM  -->  사용자  -->  사용자 생성 화면](../.gitbook/assets/18.png)

* 사용자 이름: assume-user
* 콘솔 엑세스 권한 : O
* 사용자 지정 암호: admin1234!@#$

![사용자 지정 암호: admin1234!@#$ 화면](../.gitbook/assets/19.png)

* 지금은 어떠한 권한도 주지 않을것이기 때문에 바로 다음 단계로 진행한다.

![지금은 어떠한 권한도 주지 않을것이기 때문에 바로 다음 단계로 진행한다. 화면](../.gitbook/assets/20.png)

* 사용자 생성

![사용자 생성 화면](../.gitbook/assets/21.png)

* 엣지에서 assume-user 계정으로 로그인

![엣지에서 assume-user 계정으로 로그인 화면](../.gitbook/assets/22.png)

* S3로 이동하게되면 버킷에 관한 아무런 권한이 없기 때문에 목록도 확인되지 않는다.

![S3로 이동하게되면 버킷에 관한 아무런 권한이 없기 때문에 목록도 확인되지 않는다. 화면](../.gitbook/assets/23.png)

* IAM --> 사용자 --> assume-user

![IAM  -->  사용자  -->  assume-user 화면](../.gitbook/assets/24.png)

* Command Line Interface (CLI)

![Command Line Interface CLI 화면](../.gitbook/assets/25.png)

* CSV 파일로 다운로드

![CSV 파일로 다운로드 화면](../.gitbook/assets/26.png)

```powershell
PS C:\Users\ryu> aws configure --profile assume-user
AWS Access Key ID [None]: <ACCESS_KEY_ID> key ID
AWS Secret Access Key [None]: <SECRET_ACCESS_KEY> access key
Default region name [None]: ap-northeast-2
Default output format [None]: json

PS C:\Users\ryu> aws configure list-profiles
default
assume-user

PS C:\Users\ryu> aws s3 ls --profile assume-user
An error occurred (AccessDenied) when calling the ListBuckets operation:
User: arn:aws:iam::123456789012:user/assume-user is not authorized to perform:
s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action

# my-profile은 admin 권한이므로 확인이 가능하다.
PS C:\Users\soldesk> aws  s3  ls  --profile  my-profile
2026-09-29 10:05:41 my-assume-bucket-123456789012-ap-northeast-2-an

PS C:\Users\soldesk>
aws  s3  ls  s3://my-assume-bucket-123456789012-ap-northeast-2-an  --profile  my-profile
2026-09-29 10:06:06        242 AWS-test-file.txt
```

* IAM --> 역할 --> 역할 생성

![IAM  -->  역할  -->  역할 생성 화면](../.gitbook/assets/27.png)

## 45. 실습: Terraform IAM User + AssumeRole + S3 ReadOnly 실습

* Terraform을 사용하여 IAM User를 생성
* IAM User가 AWS CLI에서 사용할 Access Key를 생성한
* S3를 읽을 수 있는 IAM Policy를 생성
* S3 ReadOnly 권한을 사용할 IAM Role을 생성
* IAM User가 해당 Role을 Assume할 수 있도록 Trust Policy를 설정
* IAM User의 Access Key로 AWS CLI에 로그인
* AWS STS AssumeRole을 실행하여 임시 자격 증명을 발급
* 임시 자격 증명을 사용하여 S3의 Bucket과 Object를 조회

## 46. 실습: 전체 동작 구조

IAM User │ │ Access Key │ ▼ AWS CLI │ │ sts:AssumeRole ▼ S3ReadOnlyRole │ │ S3ReadOnlyPolicy ▼ Amazon S3 │ ├── Bucket 조회 ├── Object 목록 조회 └── Object 다운로드

* IAM User 자체에는 S3 ReadOnly 권한이 없다.
* IAM User는 S3ReadOnlyRole을 Assume한다.
* 실제 S3 접근 권한은 Role에 연결된 S3ReadOnlyPolicy에서 가져온다.

## 47. 실습: 최종 프로젝트 구조

iam-assumerole-s3/ │ ├── provider.tf ├── variables.tf ├── main.tf ├── outputs.tf ├── terraform.tfvars │ └── s3-readonly-policy.json

## 48. 실습: STEP 1. Provider 변수 작성

* Terraform에서 AWS에 접속하기 위해 사용할 AWS Region과 AWS CLI Profile 변수를 생성한다.
* 실제 값은 나중에 terraform.tfvars에서 입력한다.
  * iam-assumerole-s3\variables.tf

```hcl
variable "aws_region" {
  description = "AWS Resource를 관리할 Region"
  type = string
}

variable "aws_profile" {
  description = "Terraform 인증에 사용할 AWS CLI Profile"
  type = string
}
```

* aws\_region
  * Terraform이 AWS API 요청을 보낼 기본 Region을 지정한다.
  * IAM은 Global Service이지만 Terraform AWS Provider를 사용하기 위해
  * 기본 Region 설정이 필요하다.
* aws\_profile
  * Terraform이 AWS에 인증할 때 사용할 AWS CLI Profile 이름이다.
  * 예를 들어 my-profile을 사용하면
  * \~/.aws/credentials에 저장된 my-profile의 자격 증명을 사용한다.

## 49. 실습: STEP 2. IAM User 변수 작성

* 새로 생성할 IAM User의 이름을 변수로 정의한다.
* 이 IAM User는 직접 S3 권한을 가지는 것이 아니라 나중에 S3ReadOnlyRole을 Assume하는 사용자로 사용한다.
  * iam-assumerole-s3\variables.tf

```hcl
variable "user_name" {
  description = "생성할 IAM User 이름"
  type = string
  default = "s3_read_user"
}
```

* user\_name
  * Terraform으로 생성할 IAM User의 이름이다.
  * 현재 기본값은 s3\_read\_user이다.
  * 이후 Role의 Trust Policy에서 이 User를 Role을 사용할 수 있는 Principal로 지정한다.

## 50. 실습: STEP 3. AWS Provider 설정

* Terraform에서 사용할 Terraform Version과 AWS Provider Version을 정의한다.
* AWS Provider는 terraform.tfvars에서 지정한 Region과 Profile을 사용한다.
  * iam-assumerole-s3\provider.tf

```hcl
terraform {
  required_version = ">= 1.16.0"
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  profile = var.aws_profile
}
```

## 51. 실습: STEP 4. IAM User 생성

* S3ReadOnlyRole을 Assume할 IAM User를 생성한다.
* 이 단계에서는 User만 생성하며 S3 접근 권한은 아직 부여하지 않는다.
  * iam-assumerole-s3\main.tf

## 52. 실습: IAM User 생성

```hcl
resource "aws_iam_user" "example_user" {
  name = var.user_name
  path = "/"
}
```

* aws\_iam\_user
  * AWS IAM User를 생성하는 Terraform Resource이다.
* name
  * 생성할 IAM User 이름이다.
  * variables.tf의 user\_name 값을 사용한다.
* path = "/"
  * IAM User가 생성될 IAM Path로 "/"는 기본 Root Path를 의미한다.
  * IAM의 기본 Root 경로에 User를 생성

## 53. 실습: STEP 5. IAM User Access Key 생성

* 생성한 IAM User가 AWS CLI에서 사용할 Access Key를 생성한다.
* Access Key는 다음 두 값으로 구성된다.
  * Access Key ID
  * Secret Access Key
* 이 값을 사용하면 IAM User 자격으로 AWS CLI API 요청을 실행할 수 있다.
  * iam-assumerole-s3\main.tf

## 54. 실습: IAM User Access Key 생성

```hcl
resource "aws_iam_access_key" "example_user_key" {
  user = aws_iam_user.example_user.name
}
```

* user
  * 어느 IAM User의 Access Key를 생성할지 지정한다.
  * STEP 5에서 생성한 example\_user의 이름을 사용한다.
* Access Key ID
  * AWS가 사용자를 식별할 때 사용하는 값이다.
* Secret Access Key
  * Access Key ID와 함께 인증에 사용하는 비밀 값이다.
  * Password처럼 외부에 노출하지 않아야 한다.

## 55. 실습: STEP 7. S3 ReadOnly Policy JSON 작성

* Role에 부여할 S3 읽기 전용 권한을 정의한다.
* 이번 실습에서는 모든 S3 Bucket을 대상으로 다음 작업만 허용한다.
  * Bucket 목록 확인
  * Bucket 내부 Object 목록 확인
  * Object 다운로드
  * iam-assumerole-s3\variables.tf

```hcl
variable "s3_policy_file" {
  description = "S3 ReadOnly Policy JSON 파일 경로"
  type = string
  default = "s3-readonly-policy.json"
}
```

* Object Upload와 Delete 권한은 부여하지 않는다.
  * iam-assumerole-s3\s3-readonly-policy.json

```json
{
  "Version": "2012-10-17",# IAM Policy Language의 Version이다.
  "Statement": [
    {
      "Sid": "ListAllBuckets",# 식별자 이름
      "Effect": "Allow",# 지정한 Action을 허용
      "Action": ["s3:ListAllMyBuckets"],# 현재 AWS Account의 S3 Bucket 목록을 조회
      "Resource": "*"# 특정 Bucket 하나가 아니라 Bucket목록을 조회하는 작업이므로 "*"를 사용
    },
    {
      "Sid": "ListBuckets",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],# 특정 Bucket 안의 Object 목록을 확인(예 : aws s3 ls s3://my-bucket)
      "Resource": ["arn:aws:s3:::*"]# 모든 S3 Bucket Resource를 의미한다. (Bucket 자체에 대한 권한이다.)
    },   (예: aws s3 cp s3://my-bucket/test.txt .)
    {
      "Sid": "ReadObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],# S3 Bucket 안에 저장된 Object를 읽거나 다운로드할 수 있다.
      "Resource": ["arn:aws:s3:::*/*"]# 모든 S3 Bucket 내부의 모든 Object를 의미
    }
  ]
}
```

* Version = "2012-10-17"
  * IAM Policy Language의 Version이다.
* Effect = "Allow"
  * 지정한 Action을 허용한다.
* s3:ListAllMyBuckets
  * 현재 AWS Account의 S3 Bucket 목록을 조회할 수 있다.
  * aws s3 ls
* Resource = "\*"
  * ListAllMyBuckets는 특정 Bucket 하나가 아니라
  * Account의 Bucket 목록을 조회하는 작업이므로 "\*"를 사용한다.
* s3:ListBucket
  * 특정 S3 Bucket 안의 Object 목록을 확인할 수 있다.
  * 예: aws s3 ls s3://my-bucket
* arn:aws:s3:::\*
  * 모든 S3 Bucket Resource를 의미한다.
  * Bucket 자체에 대한 권한이다.
* s3:GetObject
  * S3 Bucket 안에 저장된 Object를 읽거나 다운로드할 수 있다.
  * 예: aws s3 cp s3://my-bucket/test.txt .
* arn:aws:s3:::_/_
  * 모든 S3 Bucket 내부의 모든 Object를 의미한다.

## 56. 실습: STEP 7. S3 ReadOnly IAM Policy 생성

* STEP 8에서 작성한 JSON 파일을 읽어 AWS IAM Managed Policy를 생성한다.
* 이 Policy는 아직 User에게 직접 연결하지 않는다.
* 나중에 S3ReadOnlyRole에 연결한다.
  * iam-assumerole-s3\main.tf

## 57. 실습: S3 ReadOnly Managed Policy 생성

```hcl
resource "aws_iam_policy" "s3_readonly_policy" {
  name = "S3ReadOnlyPolicy"
  policy = file(var.s3_policy_file)
}
```

* aws\_iam\_policy: 재사용할 수 있는 IAM Managed Policy를 생성한다.
* name: AWS IAM Console에 생성될 Policy 이름이다.
* policy: 실제 IAM Permission Policy 내용이다.
* file(var.s3\_policy\_file)
  * variables.tf의 s3\_policy\_file에 지정된
  * JSON 파일을 읽는다.
  * 즉 다음 파일의 내용이 IAM Policy가 된다.

```
 # s3-readonly-policy.json
```

## 58. 실습: STEP 8. 현재 AWS Account ID 조회

* IAM Role의 Trust Policy에서 생성한 IAM User의 ARN을 만들어야 한다.
* IAM ARN에는 AWS Account ID가 필요하므로 현재 Terraform이 인증된 AWS Account ID를 자동으로 조회한다.
  * iam-assumerole-s3\main.tf

## 59. 실습: 현재 AWS Account 정보 조회

```hcl
data "aws_caller_identity" "current" {}
```

* aws\_caller\_identity
  * 현재 Terraform을 실행하고 있는 AWS 인증 주체의 Account 정보를 조회한다.
* account\_id
  * 현재 AWS Account ID를 가져올 수 있다.
  * data.aws\_caller\_identity.current.account\_id
  * 예 : 123456789012

## 60. 실습: STEP 9. S3 ReadOnly IAM Role 생성

* S3 ReadOnly 권한을 사용할 IAM Role을 생성한다.
* Role의 Trust Policy에는 STEP 5에서 생성한 IAM User를 Principal로 지정한다.
* 따라서 해당 IAM User가 S3ReadOnlyRole을 Assume할 수 있도록 신뢰 관계를 설정한다.
  * iam-assumerole-s3\main.tf

## 61. 실습: S3 ReadOnly IAM Role 생성

```hcl
resource "aws_iam_role" "s3_read_role" {
  name = "S3ReadOnlyRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:
                       \user/${aws_iam_user.example_user.name}"}# 2줄 입력 X
        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

* aws\_iam\_role: IAM Role을 생성
* name = "S3ReadOnlyRole": 생성할 Role 이름
* assume\_role\_policy: 이 Role을 누가 사용할 수 있는지를 정의하는 Trust Policy
* jsonencode(): Terraform의 Map/Object 형식을 IAM이 사용하는 JSON 형식으로 변환
* Principal: 이 Role을 Assume할 수 있는 대상을 지정
* AWS: AWS IAM Principal을 지정
* 현재 Principal: arn:aws:iam:::user/s3\_read\_user
* 즉 STEP 5에서 생성한 IAM User만 이 Role을 Assume할 수 있도록 설정한다.
* Action = "sts:AssumeRole"
  * 지정된 Principal이 이 Role로 전환할 수 있도록 허용한다.
  * AWS STS가 임시 자격 증명을 발급한다.

## 62. 실습: STEP 10. S3 ReadOnly Policy를 Role에 연결

* S3 ReadOnly 권한을 가진 IAM Policy를 앞에서 생성한 S3ReadOnlyRole에 연결
* 현재까지는 IAM Role과 IAM Policy가 각각 따로 생성된 상태이다.
  * Role만 생성했다고 해서 S3를 읽을 수 있는 것은 아니다.
  * 실제 S3 읽기 권한을 사용하려면 S3 ReadOnly Policy를 Role에 연결해야 한다.
* 이 설정이 완료되면 S3ReadOnlyRole은 다음과 같은 S3 읽기 권한을 가지게 된다.
  * S3 Bucket 목록 조회
  * S3 Bucket 내부 Object 목록 조회
  * S3 Object 읽기 및 다운로드
* 이후 IAM User가 S3ReadOnlyRole을 Assume하면 해당 User는 자신의 기존 권한이 아니라 Role에 연결된 S3 ReadOnly 권한을 임시로 사용할 수 있다.
  * iam-assumerole-s3\main.tf

## 63. 실습: S3 ReadOnly Policy를 Role에 연결

```hcl
resource "aws_iam_role_policy_attachment" "s3_policy_attach" {
  role = aws_iam_role.s3_read_role.name
  policy_arn = aws_iam_policy.s3_readonly_policy.arn
}
```

* aws\_iam\_role\_policy\_attachment : AM Managed Policy를 IAM Role에 연결한다.
* role : Policy를 연결할 Role 이름이다. (S3ReadOnlyRole을 사용한다.)
* policy\_arn
  * Role에 연결할 IAM Managed Policy의 ARN이다. (STEP 8에서 만든 S3ReadOnlyPolicy를 사용)
* 최종 권한 구조 S3ReadOnlyRole │ Policy Attachment ▼ S3ReadOnlyPolicy │ ├── s3:ListAllMyBuckets ├── s3:ListBucket └── s3:GetObject
* 따라서 Role 자체가 S3 ReadOnly 권한을 가지게 된다.

## 64. 실습: STEP 11. IAM User 정보 Output 작성

* Terraform으로 생성한 IAM User 이름을 terraform output 명령으로 확인할 수 있도록 설정한다.
  * iam-assumerole-s3\outputs.tf

```hcl
output "user_name" {
  description = "생성된 IAM User 이름"
  value = aws_iam_user.example_user.name
}
```

* user\_name
  * Terraform으로 생성된 IAM User 이름을 출력한다.
  * 예상 값
  * s3\_read\_user

## 65. 실습: STEP 12. IAM Role ARN Output 작성

* AWS CLI에서 sts assume-role을 실행할 때 Role ARN이 필요하다.
* Terraform Output을 통해 생성된 Role ARN을 확인한다.
  * iam-assumerole-s3\outputs.tf

```hcl
output "s3_read_role_arn" {
  description = "S3 ReadOnly Role ARN"
  value = aws_iam_role.s3_read_role.arn
}
```

* role ARN
  * AWS Resource에서 IAM Role을 고유하게 식별하는 주소이다.
  * 예 : arn:aws:iam::123456789012:role/S3ReadOnlyRole
* AssumeRole 실행 시
  * \--role-arn 옵션에 이 값을 사용한다.

## 66. 실습: STEP 13. Access Key Output 작성

* 생성한 IAM User의 Access Key ID와 Secret Access Key를 Terraform Output으로 확인한다.
* 이 값은 AWS CLI Profile을 생성할 때 사용한다.
  * iam-assumerole-s3\outputs.tf

```hcl
output "user_access_key_id" {
  description = "IAM User Access Key ID"
  value = aws_iam_access_key.example_user_key.id
}

output "user_secret_access_key" {
  description = "IAM User Secret Access Key"
  value = aws_iam_access_key.example_user_key.secret
  sensitive = true
}
```

* user\_access\_key\_id
  * IAM User의 Access Key ID를 출력한다.
* user\_secret\_access\_key
  * IAM User의 Secret Access Key를 출력한다.
* sensitive = true
  * terraform apply 또는 terraform output에서
  * Secret Access Key가 일반 문자열로 그대로 표시되지 않도록 한다.
* 주의
  * sensitive는 Terraform 화면 출력을 숨기는 기능이다.
  * Terraform State에는 Secret 값이 저장될 수 있으므로 terraform.tfstate 파일 역시 안전하게 관리해야 한다.

## 67. 실습: STEP 14. terraform.tfvars 작성

* iam-assumerole-s3\terraform.tfvars

```hcl
aws_region = "ap-northeast-2"
aws_profile = "my-profile"
user_name = "s3_read_user"
s3_policy_file = "s3-readonly-policy.json"
```

* aws\_region
  * Terraform AWS Provider Region
* aws\_profile
  * Terraform Resource 생성 권한을 가진 기존 AWS CLI Profile
* user\_name
  * Terraform으로 생성할 IAM User
* s3\_policy\_file
  * IAM S3 ReadOnly Policy JSON 파일

## 68. 실습: STEP 15. Terraform 초기화 및 검증

```
[설명]
```

* Terraform Provider를 설치하고 작성한 코드에 문제가 없는지 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> terraform init

PS C:\terraform\iam-assumerole-s3> terraform plan

PS C:\terraform\iam-assumerole-s3> terraform apply
Enter a value: yes

[생성 Resource]

# STEP 16. Terraform Output 확인

[설명]
```

* 생성된 IAM User와 Role 정보를 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> terraform output

[확인]
user_name
s3_read_role_arn
user_access_key_id
user_secret_access_key
```

## 69. 실습: STEP 17. Access Key 확인

* IAM User를 AWS CLI Profile에 등록하기 위해 Access Key 값을 확인한다.

```
[Access Key ID]
PS C:\terraform\iam-assumerole-s3> terraform output -raw user_access_key_id
AKIAXXXXXXXXXXXXXXXX

[Secret Access Key]
PS C:\terraform\iam-assumerole-s3> terraform output -raw user_secret_access_key
JTYWA5OMa6RtaWRhGZaqBlwgTGRCSINk6KXEVlD4
```

* Secret Access Key는 외부에 노출하지 않는다.

## 70. 실습: STEP 18. IAM User AWS CLI Profile 생성

* Terraform에서 생성한 IAM User의 Access Key를 사용하여 별도의 AWS CLI Profile을 만든다.
* 이 Profile은 앞으로 IAM User 권한으로 AWS CLI 명령을 실행할 때 사용한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> aws configure --profile s3-read-user
Tip: You can deliver temporary credentials to the AWS CLI using your AWS Console session by running the command 'aws login'.

AWS Access Key ID [None]: <ACCESS_KEY_ID>
AWS Secret Access Key [None]: <SECRET_ACCESS_KEY>
Default region name [None]: ap-northeast-2
Default output format [None]:

PS C:\trf2\5) IAM> aws  configure  list-profiles
default
my-profile
assume-user
s3-read-user
```

## 71. 실습: STEP 19. IAM User 인증 확인

* 새로 생성한 AWS CLI Profile이 실제 IAM User로 인증되는지 확인한다.

```powershell
PS C:\terraform\iam-assumerole-s3> aws  sts  get-caller-identity --profile s3-read-user

[확인 예]
{
    "UserId": "AIDAxxxxxxxxxxxx",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/s3_read_user"
}
```

* get-caller-identity
  * 현재 AWS CLI 요청을 실행하는
  * AWS Identity 정보를 확인한다.
* Arn
  * 현재 IAM User가 s3\_read\_user인지 확인한다.

## 72. 실습: STEP 20. 테스트용 S3 Bucket 생성

* AssumeRole 전과 후의 S3 접근 권한 차이를 확인하기 위해 테스트용 S3 Bucket을 생성한다.
* 현재 실습에서는 IAM User 자체에는 S3 권한이 없고 S3ReadOnlyRole에만 S3 읽기 권한이 있다.
* 따라서 먼저 접근 대상이 되는 S3 Bucket을 생성한 뒤 IAM User 상태와 AssumeRole 상태에서 각각 접근을 테스트
* S3 Bucket 이름은 전 세계에서 중복될 수 없기 때문에 현재 AWS Account ID를 Bucket 이름에 포함
  * iam-assumerole-s3\main.tf

## 73. 실습: AssumeRole 테스트용 S3 Bucket 생성

```hcl
resource "aws_s3_bucket" "assume_test_bucket" {
  bucket = "iam-assume-bucket-${data.aws_caller_identity.current.account_id}"
  tags = {
    Name = "IAM AssumeRole Test Bucket"
  }
}

   # iam-assumerole-s3\outputs.tf
output "s3_bucket_name" {
  description = "AssumeRole 테스트용 S3 Bucket 이름"
  value = aws_s3_bucket.assume_test_bucket.bucket
}
```

* aws\_s3\_bucket
  * AWS S3 Bucket을 생성하는 Terraform Resource이다.
  * 이번 실습에서는 IAM User와 S3ReadOnlyRole의
  * 권한 차이를 확인하기 위한 테스트 대상으로 사용한다.
* bucket
  * 생성할 S3 Bucket 이름이다.
  * S3 Bucket 이름은 AWS 전체에서 중복될 수 없기 때문에
  * 현재 AWS Account ID를 이름에 포함한다.
* data.aws\_caller\_identity.current.account\_id
  * STEP 8에서 조회한 현재 AWS Account ID를 사용한다.

## 74. 실습: STEP 21. 테스트용 S3 Object 업로드

* S3 ReadOnlyRole의 조회와 다운로드 권한을 테스트하기 위해 앞에서 생성한 S3 Bucket에 테스트 파일을 하나 생성
* 별도의 로컬 파일을 준비하지 않아도 실습할 수 있도록 Terraform의 content 속성을 사용하여 test.txt 파일을 생성
  * 이 파일은 나중에 AssumeRole 전과 후의 권한을 비교할 때 사용한다.
* AssumeRole 후에는 다음 작업을 테스트한다.
  * Bucket 내부 Object 목록 확인
  * test.txt 다운로드
* 반대로 Upload와 Delete 권한은 부여하지 않았기 때문에 새로운 Object Upload와 기존 Object Delete는 실패
  * iam-assumerole-s3\main.tf

## 75. 실습: AssumeRole 테스트용 Object 생성

```hcl
resource "aws_s3_object" "test_object" {
  bucket = aws_s3_bucket.assume_test_bucket.id
  key = "test.txt"
  content = "IAM AssumeRole S3 ReadOnly Test"
  content_type = "text/plain"
}

   # iam-assumerole-s3\outputs.tf
output "s3_test_object" {
  description = "AssumeRole 테스트용 S3 Object 이름"
  value = aws_s3_object.test_object.key
}

PS C:\terraform\iam-assumerole-s3> terraform plan
PS C:\terraform\iam-assumerole-s3> terraform apply
```

* 현재 IAM User에는 S3 ReadOnly Policy를 직접 연결하지 않았다.
* 따라서 Role을 Assume하기 전에는 앞에서 생성한 S3 Bucket에 접근할 수 없어야 한다.
* 먼저 테스트용 Bucket 이름을 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> terraform output -raw s3_bucket_name
iam-assume-bucket-123456789012
```

* IAM User의 s3-read-user Profile을 사용하여 S3 Bucket 목록 조회를 시도한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> aws s3 ls --profile s3-read-user
aws: [ERROR]: An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::123456789012:user/s3_read_user is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
```

* 특정 Bucket 내부 조회도 시도한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> $BUCKET = terraform output -raw s3_bucket_name

PS C:\terraform\iam-assumerole-s3> aws s3 ls s3://$BUCKET --profile s3-read-user
aws: [ERROR]: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: User: arn:aws:iam::123456789012:user/s3_read_user is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::iam-s3-bucket-123456789012" because no identity-based policy allows the s3:ListBucket action
```

* IAM User 자체에는 S3 Permission이 없다.
* S3 Bucket과 test.txt는 정상적으로 생성되어 있지만 s3\_read\_user에게는 S3 접근 권한이 없다.
* S3 읽기 권한은 S3ReadOnlyRole에만 연결되어 있다.
* 따라서 이후 S3ReadOnlyRole을 Assume한 뒤 동일한 Bucket에 다시 접근하여 성공하는지 확인한다.

## 76. 실습: STEP 22 S3ReadOnlyRole ARN 확인

* AssumeRole을 실행하기 위해 Terraform에서 생성한 Role ARN을 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> terraform output -raw s3_read_role_arn
arn:aws:iam::123456789012:role/S3ReadOnlyRole
```

## 77. 실습: STEP 23. AssumeRole 임시 Credential을 PowerShell 변수에 저장

* STEP 22에서 AssumeRole을 실행하면 AWS STS가 S3ReadOnlyRole 권한을 사용할 수 있는 임시 자격 증명을 반환한다.
* 하지만 반환된 값을 화면에서 확인만 해서는 이후 AWS CLI 명령이 자동으로 그 Role 권한을 사용하는 것은 아니다.
* 따라서 이번 단계에서는 AssumeRole 결과를 PowerShell 변수에 저장하고, 그 안에 들어 있는 임시 자격 증명을 Windows 환경 변수에 등록한다.
* 등록하는 값은 다음 3개이다.
  * AWS\_ACCESS\_KEY\_ID
  * AWS\_SECRET\_ACCESS\_KEY
  * AWS\_SESSION\_TOKEN
* 이 3개의 환경 변수가 설정되면 현재 PowerShell Session에서 실행하는 AWS CLI 명령은 기존 IAM User의 Access Key가 아니라 S3ReadOnlyRole의 임시 자격 증명을 사용하게 된다.
* 즉, 실제로 Role 권한을 AWS CLI에서 사용하도록 설정하는 단계이다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> $ROLE_ARN = terraform output -raw s3_read_role_arn

PS C:\terraform\iam-assumerole-s3> $creds = aws sts assume-role `
>> --role-arn $ROLE_ARN `
>> --role-session-name s3-read-session `
>> --profile s3-read-user | ConvertFrom-Json

PS C:\terraform\iam-assumerole-s3> $env:AWS_ACCESS_KEY_ID = $creds.Credentials.AccessKeyId

PS C:\terraform\iam-assumerole-s3> $env:AWS_SECRET_ACCESS_KEY = $creds.Credentials.SecretAccessKey

PS C:\terraform\iam-assumerole-s3> $env:AWS_SESSION_TOKEN = $creds.Credentials.SessionToken
```

* ConvertFrom-Json
  * AWS CLI에서 반환된 JSON 결과를
  * PowerShell Object로 변환한다.
* $creds
  * AssumeRole 결과를 저장한다.
* AWS\_ACCESS\_KEY\_ID
  * 임시 Access Key ID
* AWS\_SECRET\_ACCESS\_KEY
  * 임시 Secret Access Key
* AWS\_SESSION\_TOKEN
  * AssumeRole Session Token
* 이 세 값을 Environment Variable로 설정하면 AWS CLI는 임시 Role Credential을 사용하게 된다.

## 78. 실습: STEP 26. AssumeRole 성공 확인

* 현재 AWS CLI가 IAM User가 아니라 S3ReadOnlyRole 자격으로 실행되고 있는지 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> aws sts get-caller-identity
{
    "UserId": "AROA5A62EVKFYMKNZ454X:s3-read-session",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/S3ReadOnlyRole/s3-read-session"
}
```

* 기존
  * arn:aws:iam::123456789012:user/s3\_read\_user
* AssumeRole 후
  * arn:aws:sts::123456789012:assumed-role/S3ReadOnlyRole/s3-read-session
* 즉 IAM User에서 Role Session으로 Identity가 변경되었다.

## 79. 실습: STEP 27. S3 Bucket 목록 조회

* S3ReadOnlyRole에 포함된 s3:ListAllMyBuckets 권한을 확인한다.

```
[PowerShell]
PS C:\trf2\5) IAM> aws  s3  ls
2026-09-29 11:47:20 iam-s3-bucket-123456789012
2026-09-29 10:05:41 my-assume-bucket-123456789012-ap-northeast-2-an
```

* 현재 AWS Account에서 접근 가능한 S3 Bucket 목록이 출력되는지 확인한다.

## 80. 실습: STEP 28. S3 Bucket 내부 Object 조회

* S3ReadOnlyRole의 s3:ListBucket 권한을 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> aws  s3  ls  s3://$BUCKET
2026-09-29 11:53:37         29 test.txt
```

* Bucket 내부의 Object 목록이 출력되는지 확인한다.

## 81. 실습: STEP 29. S3 Object 다운로드

* S3ReadOnlyRole의 s3:GetObject 권한을 확인한다.
* S3의 Object를 사용자 PC로 다운로드한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> aws s3 cp s3://<Bucket-이름>/<파일이름> .

[예]
PS C:\terraform\iam-assumerole-s3> aws s3 cp s3://my-test-bucket/test.txt .
download: s3://my-test-bucket/test.txt to .\test.txt
```

* 현재 Role에는 GetObject 권한이 있으므로 Object 다운로드가 가능하다.

## 82. 실습: STEP 30. S3 Upload 권한 차단 확인

* 현재 Role은 ReadOnly Role이다.
* s3:PutObject 권한을 부여하지 않았기 때문에 S3로 새로운 파일을 Upload할 수 없어야 한다.

\[테스트 파일 생성]

```powershell
PS C:\terraform\iam-assumerole-s3> "S3 Write Test" > upload-test.txt

[Upload 시도]
PS C:\terraform\iam-assumerole-s3> aws  s3  cp  .\upload-test.txt  s3://$BUCKET
upload failed: .\upload-test.txt to s3://iam-s3-bucket-123456789012/upload-test.txt An error occurred (AccessDenied) when calling the PutObject operation: User: arn:aws:sts::123456789012:assumed-role/S3ReadOnlyRole/s3-read-session is not authorized to perform: s3:PutObject on resource: "arn:aws:s3:::iam-s3-bucket-123456789012/upload-test.txt" because no identity-based policy allows the s3:PutObject action
```

* 읽기는 가능하지만 쓰기는 불가능하다.

## 83. 실습: STEP 31. S3 Delete 권한 차단 확인

* ReadOnly Role에는 s3:DeleteObject 권한도 없다.
* 따라서 기존 Object 삭제가 차단되는지 확인한다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> aws  s3  rm  s3://$BUCKET/upload-test.txt
delete failed: s3://iam-s3-bucket-123456789012/upload-test.txt An error occurred (AccessDenied) when calling the DeleteObject operation: User: arn:aws:sts::123456789012:assumed-role/S3ReadOnlyRole/s3-read-session is not authorized to perform: s3:DeleteObject on resource: "arn:aws:s3:::iam-s3-bucket-123456789012/upload-test.txt" because no identity-based policy allows the s3:DeleteObject action
```

## 84. 실습: STEP 32. 임시 Credential 제거

* 실습이 끝난 후 PowerShell에 설정한 AssumeRole 임시 Credential을 제거한다.
* 제거하지 않으면 현재 PowerShell Session에서 계속 Role Credential을 사용하게 된다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> Remove-Item Env:AWS_ACCESS_KEY_ID
PS C:\terraform\iam-assumerole-s3> Remove-Item Env:AWS_SECRET_ACCESS_KEY
PS C:\terraform\iam-assumerole-s3> Remove-Item Env:AWS_SESSION_TOKEN

# 환경변수를 삭제하게되면 default profile이 적용되므로 upload와 delete가 적용된다.
PS C:\trf2\5) IAM> aws s3  cp  .\upload-test.txt  s3://$BUCKET
upload: .\upload-test.txt to s3://iam-s3-bucket-123456789012/upload-test.txt

PS C:\trf2\5) IAM> aws  s3  rm  s3://$BUCKET/upload-test.txt
delete: s3://iam-s3-bucket-123456789012/upload-test.txt
```

## 85. 실습: STEP 35. Terraform Resource 삭제

* 실습이 끝난 후 Terraform으로 생성한 IAM Resource를 삭제한다.
* AWS CLI Profile의 Access Key는 Terraform이 삭제하는 IAM User와 함께 더 이상 사용할 수 없게 된다.

```
[PowerShell]
PS C:\terraform\iam-assumerole-s3> terraform destroy

Enter a value: yes

[삭제 Resource]
```
