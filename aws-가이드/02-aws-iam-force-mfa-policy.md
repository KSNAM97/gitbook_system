# AWS IAM - MFA 강제 정책 적용하기

IAM 사용자가 MFA(다단계 인증)를 활성화하기 전까지는 자신의 비밀번호·MFA 장치 관리 외의 어떤 작업도 할 수 없도록 강제하는 IAM 정책을 만들고, 이를 사용자 그룹에 연결해 특정 권한(EC2 전체 액세스)과 함께 적용하는 절차를 다룬다.

## 1. IAM 개념과 사용자 생성

### IAM이란

AWS Identity and Access Management(IAM)은 AWS 리소스에 대한 액세스를 안전하게 제어할 수 있는 웹 서비스다. AWS 계정을 최초로 생성할 때는 모든 AWS 서비스 및 리소스에 대해 완전한 액세스 권한이 있는 단일 로그인 ID로 시작하는데, 이 자격 증명을 루트 사용자라고 한다.

루트 계정은 모든 권한을 가지고 있기 때문에 공유되어서도 안 되고 보안에 각별히 주의해야 한다. 그래서 루트 사용자를 직접 사용하지 않을 것을 권장하며, IAM을 통해 리소스를 사용할 수 있도록 인증(로그인, 계정) 및 권한 부여된 대상을 제어한다.

### IAM의 기능

- AWS 계정의 리소스를 관리하고 사용할 수 있는 권한을 다른 사람에게 부여할 수 있다.
- 리소스에 따라 여러 사람에게 다양한 권한을 세분화하여 부여할 수 있다.
- Amazon EC2 인스턴스에 대해 IAM 기능을 통해 완전하게 자격 증명을 안전하게 제공할 수 있다.
- 멀티 팩터 인증(MFA)을 제공한다. 계정 소유자나 사용자가 계정을 사용하기 위해 암호, 액세스 키뿐 아니라 특별히 구성된 디바이스 코드도 제공하도록 요구할 수 있다.
- ID 페더레이션을 제공한다.
- PCI DSS를 준수한다.
- 많은 AWS 서비스와 통합이 가능하다.
- 일관성을 제공하고, 기본적으로 추가 비용 없이 무료로 이용할 수 있다.

### IAM 정책(Policy)

IAM 정책(policy)은 IAM 역할(role) 혹은 개인 사용자에게 부여할 수 있다. 사용자에게 직접 정책을 부여할 수도 있지만, 사용자 그룹 단위로 부여할 수도 있다.

### IAM 사용자 추가 절차

1. IAM 대시보드에서 [액세스 관리] > [사용자]를 선택한다.
2. 오른쪽 상단의 [사용자 추가]를 선택한다. 사용자 추가는 사용자 세부 정보 지정, 권한 설정, 검토 및 생성, 암호 검색 4단계로 진행된다.
3. **사용자 세부 정보 지정**: 사용자 이름을 입력한다. AWS Management Console에 대한 사용자 액세스 권한 제공 항목에서, Identity Center를 이용하면 사용자가 콘솔을 액세스할 수 있는 권한을 중앙에서 관리할 수 있다. IAM 사용자를 직접 생성하는 방식은 액세스 키나 서비스별 보안 인증 정보를 통해 프로그래밍 방식 액세스를 활성화해야 하는 경우에만 권장된다. IAM 사용자 생성을 선택하면 콘솔 암호를 AWS에서 자동 생성하거나 직접 지정할 수 있는 옵션이 나타나며, "사용자는 다음 로그인 시 새 암호를 생성해야 합니다" 옵션을 체크하는 것이 권장된다.
4. **권한 설정**: 기존 사용자 그룹에 사용자를 추가하거나 새 그룹을 생성한다. 사용자 그룹은 그룹 단위로 정책(policy)이나 역할(role)을 부여할 수 있기 때문에, 해당 사용자에게 필요한 권한을 고려해 그룹에 추가한다. 한 명의 사용자가 여러 정책이나 역할을 부여받을 수 있고, 두세 개의 사용자 그룹에 동시에 속할 수도 있다. 개인 단위로 권한을 부여할 경우 [직접 정책 연결]을 선택하면, AWS에서 관리하는 권한 정책 목록이 나타난다. 필요한 정책이 없다면 [정책 생성]으로 직접 만들 수도 있다.
5. **검토 및 생성**: 설정한 사용자의 세부 정보와 권한을 확인한 뒤 [사용자 생성]을 선택한다.
6. **암호 검색**: 사용자가 생성되면 콘솔 로그인 URL이 표시된다. 이 화면에서만 암호를 확인·다운로드할 수 있으므로 .csv 파일을 반드시 다운로드해 저장한다. .csv 파일에는 콘솔 로그인 URL, 사용자 이름, 콘솔 암호가 포함된다.

생성된 IAM 사용자로 로그인하려면 콘솔 로그인 URL로 접속하여 계정 ID(또는 별칭), 사용자 이름, 발급받은 암호를 입력한다. 처음 발급받은 암호로 로그인하면 이전 비밀번호를 입력하고 새 비밀번호를 설정하는 화면이 나타나며, 새 비밀번호를 설정하면 루트 계정이 아닌 IAM 사용자 계정으로 접속된다.

## 2. MFA 강제 정책(Force_MFA) 생성

1. IAM 대시보드 왼쪽 메뉴에서 [정책]으로 이동한다.
2. 오른쪽 상단의 [정책 생성]을 선택한다.
3. [JSON] 탭을 선택하고, MFA 인증 없이는 자신의 비밀번호·액세스 키·MFA 장치 관리 외의 모든 작업을 거부하는 다음 정책을 작성한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowViewAccountInfo",
      "Effect": "Allow",
      "Action": [
        "iam:GetAccountPasswordPolicy",
        "iam:ListVirtualMFADevices"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowManageOwnPasswords",
      "Effect": "Allow",
      "Action": [
        "iam:ChangePassword",
        "iam:GetUser"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    },
    {
      "Sid": "AllowManageOwnAccessKeys",
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:ListAccessKeys",
        "iam:UpdateAccessKey"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    },
    {
      "Sid": "AllowManageOwnVirtualMFADevice",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice",
        "iam:DeleteVirtualMFADevice"
      ],
      "Resource": "arn:aws:iam::*:mfa/${aws:username}"
    },
    {
      "Sid": "AllowManageOwnUserMFA",
      "Effect": "Allow",
      "Action": [
        "iam:DeactivateMFADevice",
        "iam:EnableMFADevice",
        "iam:ListMFADevices",
        "iam:ResyncMFADevice"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    },
    {
      "Sid": "DenyAllExceptListedIfNoMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

- **AllowViewAccountInfo**: 계정 비밀번호 정책 조회, 가상 MFA 장치 목록 조회를 허용한다.
- **AllowManageOwnPasswords**: 자신의 비밀번호 변경, 자신의 사용자 정보 조회를 허용한다.
- **AllowManageOwnAccessKeys**: 자신의 액세스 키 생성·삭제·조회·업데이트를 허용한다.
- **AllowManageOwnVirtualMFADevice**: 자신의 가상 MFA 장치 생성·삭제를 허용한다.
- **AllowManageOwnUserMFA**: 자신의 MFA 장치 활성화·비활성화·조회·재동기화를 허용한다.
- **DenyAllExceptListedIfNoMFA**: `aws:MultiFactorAuthPresent` 조건이 `false`인 경우, 즉 MFA로 인증하지 않은 세션에서는 위에 나열된 MFA 관련 작업을 제외한 모든 작업을 거부한다.

4. 정책 검토 화면에서 이름을 `Force_MFA`로 지정하고 설명을 입력한 뒤 [정책 생성]을 선택한다.

정책 목록에 `Force_MFA`가 고객 관리형 정책으로 추가된다.

## 3. 사용자 그룹 생성 및 정책 연결

1. IAM 대시보드에서 [사용자 그룹] 메뉴로 이동한다.
2. 오른쪽 상단의 [그룹 생성]을 선택한다.
3. 그룹 이름을 `EC2MFA`로 지정한다.
4. 권한 정책 연결 단계에서 `EC2Full`을 검색하여 AWS 관리형 정책인 `AmazonEC2FullAccess`를 체크한다.
5. 이어서 `Force`를 검색하여 앞서 만든 `Force_MFA` 정책도 함께 체크하고 [그룹 생성]을 선택한다.

이렇게 하면 EC2MFA 그룹에 속한 사용자는 MFA로 인증하기 전까지 EC2 관련 작업을 포함한 모든 작업이 차단되고, MFA 등록·활성화 관련 작업만 허용된다. MFA 인증을 마친 세션에서는 `DenyAllExceptListedIfNoMFA`의 거부 조건이 해제되어 그룹에 연결된 `AmazonEC2FullAccess` 권한이 정상적으로 적용된다.

## 4. 사용자 생성 및 그룹 할당

1. IAM 대시보드에서 [사용자] 메뉴로 이동한다.
2. 오른쪽 상단의 [사용자 추가]를 선택한다.
3. 사용자 이름을 `MFAUser`로 입력한다. AWS 자격 증명 유형은 "암호 - AWS 관리 콘솔 액세스"를 선택하고, 콘솔 비밀번호는 자동 생성으로 두며, "사용자가 다음에 로그인할 때 새 비밀번호 생성 요청"을 체크한다.
4. 권한 설정 단계에서 [그룹에 사용자 추가]를 선택하고, 앞서 만든 `EC2MFA` 그룹을 체크한 뒤 다음 단계로 진행하여 사용자를 생성한다.

사용자가 생성되면 IAM 대시보드의 리소스 현황(사용자 그룹/사용자/역할/정책 수)에 반영된다.

## 5. MFA 디바이스 등록

`Force_MFA` 정책이 적용된 사용자는 MFA 디바이스를 등록해야 다른 작업을 수행할 수 있다. 계정/패스워드 인증 방식만 사용할 경우 계정 정보 탈취나 서비스 이용 요금 과다 발생 등 AWS 이용에 심각한 위험이 발생할 수 있다. MFA(Multi-Factor Authentication)는 기존 인증 방식 외에 추가 인증 정보 입력을 요구하는 다중 인증 방식으로, AWS 콘솔 로그인 보안을 강화하며 사용 시 추가 비용은 발생하지 않는다.

1. AWS Management Console에 로그인한 뒤 오른쪽 상단의 계정 메뉴에서 [보안 자격 증명]을 선택한다.
2. 보안 자격 증명 페이지의 [멀티 팩터 인증(MFA)] 섹션에서 [MFA 디바이스 할당]을 선택한다.
3. MFA 디바이스 유형을 선택한다. 패스키/보안 키, 인증 관리자 앱, 하드웨어 TOTP 토큰 세 가지 옵션이 있으며, 가장 널리 쓰이는 방식은 모바일 또는 컴퓨터 앱 기반 인증 관리자 앱이다. 디바이스 이름을 입력한다. 사용자는 MFA 디바이스를 최대 8개까지 할당할 수 있으므로 여러 디바이스를 구분할 수 있는 이름을 붙이는 것이 좋다. [인증 관리자 앱]을 선택하고 [다음]을 선택한다.
4. 사용 중인 스마트폰 OS에 따라 Google Play Store 또는 Apple App Store에서 OTP 애플리케이션(Google Authenticator 등)을 설치한다.
5. 가상 MFA 디바이스 설정 화면에서 [QR 코드 표시]를 선택하고, OTP 애플리케이션의 QR 코드 스캔 기능으로 화면에 표시된 QR 코드를 스캔한다.
6. 애플리케이션에 연속으로 표시되는 MFA 코드 2개를 각각 입력한 뒤 [MFA 할당]을 선택하면 설정이 완료된다.
   - **MFA 코드 1**: 현재 확인되는 MFA 코드 6자리
   - **MFA 코드 2**: MFA 코드 1을 입력한 후 갱신되어 표시되는 새 MFA 코드 6자리

설정 완료 후 AWS 콘솔에 로그인하면, 기존 계정 정보로 1차 인증을 마친 뒤 MFA 코드를 입력하는 2차 인증 과정이 추가된다.

> 관련: 이론 1.  AWS - 클라우드 기초 개념 · 이론 2.  AWS EC2 - 배포
