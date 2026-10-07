# Terraform - 환경 구축 (Windows + AWS CLI + VS Code)

> **Tag:** #Terraform #AWS #환경설정 #AWSCLI #VSCode
> **핵심 요약:** Windows에서 Terraform 실행 환경(AWS CLI 프로파일, 플러그인 캐시, VS Code 포맷터)을 구성하고 첫 `main.tf`를 작성하는 절차

---

## 1. 핵심 기술 개념 (Concept)

| 항목 | 설명 |
|---|---|
| Terraform | HashiCorp의 IaC 도구. `.tf` 파일(HCL)로 인프라를 선언 |
| Provider | AWS 등 외부 서비스와 통신하는 플러그인 (`hashicorp/aws`) |
| AWS CLI Profile | Access Key를 이름으로 분리 저장 (`default`, `my-profile`) |
| Plugin Cache | Provider 바이너리를 공용 폴더에 두고 프로젝트 간 재사용 |

설치: <https://developer.hashicorp.com/terraform/install#windows>

---

## 2. 표준 설정 템플릿 (Configuration)

### 2-1. AWS CLI 프로파일

```powershell
aws configure                       # default 프로파일
aws configure --profile my-profile  # 별도 프로파일
aws configure list-profiles
```

입력 항목: Access Key ID, Secret Access Key, Region(`ap-northeast-2`), Output format(공란)

> 키 값은 문서/저장소에 기록하지 않는다.

### 2-2. Provider 플러그인 캐시 (관리자 CMD)

```bat
mkdir C:\terraform-plugin-cache
setx TF_PLUGIN_CACHE_DIR "C:\terraform-plugin-cache"
```

### 2-3. VS Code 설정 (`settings.json`)

```json
"[terraform]": {
  "editor.formatOnSave": true,
  "editor.formatOnSaveMode": "file",
  "editor.defaultFormatter": "hashicorp.terraform",
  "editor.tabSize": 2
},
"[terraform-vars]": {
  "editor.formatOnSave": true,
  "editor.formatOnSaveMode": "file",
  "editor.defaultFormatter": "hashicorp.terraform",
  "editor.tabSize": 2
}
```

### 2-4. main.tf 기본 골격

```hcl
terraform {
  required_version = ">= 1.15.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.62.0"
    }
  }
}

provider "aws" {
  region  = "ap-northeast-2"
  profile = "my-profile"
}
```

### 2-5. EC2 user_data 예시 (Nginx)

```hcl
user_data = <<-EOF
            #!/bin/bash
            dnf install -y nginx
            systemctl start nginx
            EOF
```

---

## 3. 실행 순서

```bash
terraform init      # Provider 다운로드
terraform plan      # 변경 사항 미리보기
terraform apply     # 적용
terraform destroy   # 삭제
```

---

## 4. 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `No valid credential sources` | `profile` 이름 불일치 → `aws configure list-profiles` 확인 |
| init 때마다 Provider 재다운로드 | `TF_PLUGIN_CACHE_DIR` 미설정 또는 새 터미널 미적용 |
| 저장 시 포맷 안 됨 | HashiCorp Terraform 확장 미설치 |
