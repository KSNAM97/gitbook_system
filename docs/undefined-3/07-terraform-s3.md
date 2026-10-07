# Terraform - S3

이 문서는 Terraform으로 S3 버킷, 버저닝, 정적 웹 호스팅을 구성하는 방법과 관련 실습을 정리한다.

## 1. Amazon S3 (Simple Storage Service) 서비스 이해

## 2. Amazon S3란?

* Amazon S3(Simple Storage Service)는 AWS에서 제공하는 객체 스토리지(Object Storage) 서비스이다.
* EC2의 EBS처럼 서버의 디스크로 사용하는 스토리지가 아니라, 파일, 이미지, 동영상, 백업 데이터, 로그 등의 데이터를 객체(Object) 형태로 저장하는 서비스이다.
* S3는 서버를 직접 생성하거나 관리할 필요가 없는 완전 관리형 서비스이다.
* AWS가 저장 장치의 확장, 복제, 장애 처리 등의 인프라를 관리
* 사용자는 저장할 데이터를 S3 Bucket에 업로드하고 필요할 때 다운로드하여 사용하면 된다.
* 대표적인 사용 용도
  * 이미지, 동영상 저장
  * 웹사이트 파일 저장
  * 로그 저장, 데이터 백업
  * 정적 웹사이트 호스팅
  * 애플리케이션 파일 저장
  * 빅데이터 및 데이터 레이크 저장소
* S3의 가장 큰 특징은 서버의 디스크 용량을 직접 관리할 필요가 없다는 것이다.
* EC2에 파일을 저장하는 경우에는 EBS 용량을 직접 결정해야 하지만, S3는 저장되는 데이터 양에 따라 자동으로 확장된다.

## 3. S3와 EBS의 차이

* EBS와 S3는 모두 데이터를 저장하지만 저장 방식과 사용 목적이 다르다.
* EBS는 EC2에 디스크처럼 연결해서 사용하는 Block Storage이다.
* S3는 Network를 통해 Object를 저장하고 가져오는 Object Storage이다.

## 4. 주요 차이

구분EBSS3 저장 방식Block StorageObject Storage 주 용도EC2 DiskFile / Object 저장 EC2 연결필요필요 없음 File System사용사용하지 않음 확장Volume 크기 관리자동 확장 대표 사용OS, DB DiskImage, Backup, Log

## 5. S3 기본 구조

* S3는 크게 다음 구조로 데이터를 관리

S3 Bucket │ ├── images/ │ ├── cat.jpg │ └── dog.jpg │ ├── documents/ │ └── report.pdf │ └── index.html

* S3의 핵심 구성 요소는 다음과 같다.
  * Bucket
  * Object
  * Key
  * Value
  * Metadata

## 6. Bucket

* Bucket은 S3에서 Object를 저장하는 가장 상위의 저장 공간이다.
* 일반적인 File System으로 생각하면 최상위 Directory와 비슷하게 보일 수 있지만, 실제 S3는 일반적인 Directory 기반 File System이 아니다.

```
예)
my-soldesk-bucket
 ├ images/cat.jpg
 ├ images/dog.jpg
 └ index.html
```

* 여기서 my-soldesk-bucket 이 Bucket 이름이다.

## 7. Object

* S3에 실제로 저장되는 데이터를 Object라고 한다.

```
예)
 # image.jpg
 # index.html
 # backup.zip
 # access.log
 # video.mp4
```

* 각 Object는 다음과 같은 정보로 구성된다.
  * Key
  * Value
  * Metadata
  * Version ID

```
예)
Bucket 이름
my-soldesk-bucket
          │
           └── images/aws.png
```

* Bucket: 파일을 저장하는 S3의 저장 공간
* Object: S3에 실제로 저장된 파일 하나
* Key: Bucket 안에서 Object를 구분하는 이름 또는 경로
* Value: 실제 파일 내용
* Metadata: 그 파일에 대한 부가 정보 (파일 크기, 업로드 날짜 ,암호화 정보)

## 8. S3의 Folder 구조

* S3 Console에서는 Folder가 존재하는 것처럼 보인다.

```
예)
images/
documents/
backup/
```

* 하지만 실제 S3는 전통적인 File System의 Directory 구조를 사용하는 것이 아니다.
* 다음 Object가 있다고 가정
  * images/aws.png
* 실제로 Object Key 자체가 "images/aws.png" 이다.
* 즉 "/" 문자를 이용하여 Console에서 Folder처럼 표현하는 것이다.
* 이를 Prefix라고

```
예)
images/aws.png
images/terraform.png
images/linux.png
```

* 위 Object의 공통 Prefix는 “images/” 이다.

## 9. S3 데이터 접근 방식

* S3는 일반적으로 HTTP 또는 HTTPS 기반 API를 통해 데이터에 접근
* AWS CLI를 이용할 수도 있다.
* 현재 AWS Account의 S3 Bucket 목록 확인
  * aws s3 ls
* 특정 Bucket 확인
  * aws s3 ls s3://my-soldesk-bucket

## 10. S3 Bucket 생성

* Terraform에서는 aws\_s3\_bucket Resource를 사용하여 Bucket을 생성할 수 있다.

```hcl
   # main.tf
resource "aws_s3_bucket" "my_bucket" {
  bucket = "soldesk-s3-example-123456789"
  tags = {
    Name        = "soldesk-s3"
    Environment = "dev"
  }
}
```

* 코드 설명
* aws\_s3\_bucket
  * AWS S3 Bucket을 생성하는 Terraform Resource
* my\_bucket
  * Terraform 내부에서 사용하는 Resource 이름
* bucket
  * 실제 AWS에 생성될 S3 Bucket 이름
* tags
  * S3 Bucket 관리에 사용할 Tag

## 11. S3 Bucket 이름

* S3 Bucket 이름은 같은 AWS Account 안에서만 구분되는 이름이 아니다.
* S3 General Purpose Bucket 이름은 전역 Namespace에서 고유해야
* 이미 다른 AWS 사용자가 사용하고 있는 Bucket 이름은 사용할 수 없다.
  * 예) my-s3-bucket
* 다른 사용자가 위 이름을 이미 사용 중이라면 같은 이름으로 Bucket을 생성할 수 없다. 따라서 실습에서는 AWS Account ID를 Bucket 이름에 포함하는 방법을 많이 사용

## 12. AWS Account ID를 이용한 Bucket 이름 생성

```hcl
   # main.tf
# 현재 AWS Account 정보 조회
data "aws_caller_identity" "current" {
}

# Bucket 이름 생성
locals {
  bucket_name = "soldesk-s3-${data.aws_caller_identity.current.account_id}"
}

# S3 Bucket 생성
resource "aws_s3_bucket" "my_bucket" {
  bucket = local.bucket_name
  tags = {
    Name = "soldesk-s3"
  }
}
```

* 예를 들어 Account ID가 “123456789012” 이라면 Bucket 이름은 "soldesk-s3-123456789012"이 된다.

## 13. S3 ARN

* AWS Resource는 ARN(Amazon Resource Name)을 사용하여 식별할 수 있다.
* Bucket ARN예) arn:aws:s3:::soldesk-s3-123456789012
* Object ARN예) arn:aws:s3:::soldesk-s3-123456789012/images/aws.png
* Bucket과 Object의 ARN이 서로 다르다는 점이 중요하다.
  * Bucket: arn:aws:s3:::my-bucket
  * Object: arn:aws:s3:::my-bucket/\*
* IAM Policy나 Bucket Policy를 작성할 때 자주 사용된다.

## 14. S3 Object Upload

* Terraform으로 S3 Bucket에 파일을 업로드할 수도 있다.
* 예를 들어 현재 Terraform Directory에 다음 파일이 있다고 가정

```hcl
 # index.html

   # main.tf
resource "aws_s3_object" "index" {
  bucket= aws_s3_bucket.my_bucket.id
  key    = "index.html"
  source = "${path.module}/index.html"
  content_type = "text/html"
}
```

## 15. 코드 설명

* bucket
  * Object를 업로드할 S3 Bucket
* key
  * S3에서 사용할 Object 이름
* source
  * Local PC에 존재하는 실제 File 경로
* content\_type
  * Object의 Content-Type 지정

## 16. S3 Object 경로

* 위 Terraform으로 생성되는 구조
* 다음처럼 Key를 설정하면

```hcl
key = "images/aws.png"
```

* S3 Console에서는 다음처럼 보인다. images └── aws.png
* 하지만 실제 Object Key는"images/aws.png"이다.

## 17. S3 Versioning

* Versioning은 동일한 Object의 여러 Version을 저장하는 기능이다.
* 예를 들어"index.html"File을 업로드했다고 가정
* 처음 내용

```
<h1>Hello AWS</h1>
```

* 이후 File을 수정

```
<h1>Hello Soldesk</h1>
```

* Versioning을 사용하지 않는 경우 기존 Object가 덮어써진다.

```
index.html

Hello Soldesk
```

* Versioning을 사용하면

```
index.html
 ├ Version 3
 ├ Version 2
 └ Version 1
```

* 처럼 이전 Version을 보관할 수 있다.
* Versioning을 사용하면 다음 상황에서 복구가 가능하다.
  * 실수로 File 덮어쓰기
  * 잘못된 File 업로드
  * Object 삭제
  * 이전 Version 복구

## 18. Terraform으로 Versioning 활성화

```hcl
   # main.tf
resource "aws_s3_bucket_versioning" "my_bucket_versioning" {
  bucket = aws_s3_bucket.my_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

## 19. 코드 설명

* bucket
  * Versioning을 적용할 S3 Bucket
* versioning\_configuration
  * Versioning 설정
* status = "Enabled"
  * S3 Versioning 활성화
* 구조 S3 Bucket

```
index.html
 ├ Version 3
 ├ Version 2
 └ Version 1
```

## 20. Versioning에서 Object 삭제

* Versioning을 사용하는 Bucket에서 Object를 일반적으로 삭제하면 기존 Version 자체가 즉시 완전히 삭제되는 방식과 다르게 동작

```
예)
index.html
 ├ Delete Marker
 ├ Version 3
 ├ Version 2
 └ Version 1
```

* Delete Marker가 현재 Version 역할을 하기 때문에 Object가 삭제된 것처럼 보인다.
* 필요하면 이전 Version을 다시 사용할 수 있다.

## 21. 실습: Amazon S3 (Simple Storage Service) 서비스 이해

## 22. 실습: Amazon S3란?

* Amazon S3(Simple Storage Service)는 AWS에서 제공하는 객체 스토리지(Object Storage) 서비스이다.
* EC2의 EBS처럼 서버의 디스크로 사용하는 스토리지가 아니라, 파일, 이미지, 동영상, 백업 데이터, 로그 등의 데이터를 객체(Object) 형태로 저장하는 서비스이다.
* S3는 서버를 직접 생성하거나 관리할 필요가 없는 완전 관리형 서비스이다.
* AWS가 저장 장치의 확장, 복제, 장애 처리 등의 인프라를 관리
* 사용자는 저장할 데이터를 S3 Bucket에 업로드하고 필요할 때 다운로드하여 사용하면 된다.
* 대표적인 사용 용도
  * 이미지, 동영상 저장
  * 웹사이트 파일 저장
  * 로그 저장, 데이터 백업
  * 정적 웹사이트 호스팅
  * 애플리케이션 파일 저장
  * 빅데이터 및 데이터 레이크 저장소
* S3의 가장 큰 특징은 서버의 디스크 용량을 직접 관리할 필요가 없다는 것이다.
* EC2에 파일을 저장하는 경우에는 EBS 용량을 직접 결정해야 하지만, S3는 저장되는 데이터 양에 따라 자동으로 확장된다.

## 23. 실습: S3와 EBS의 차이

* EBS와 S3는 모두 데이터를 저장하지만 저장 방식과 사용 목적이 다르다.
* EBS는 EC2에 디스크처럼 연결해서 사용하는 Block Storage이다.
* S3는 Network를 통해 Object를 저장하고 가져오는 Object Storage이다.

## 24. 실습: 주요 차이

구분EBSS3 저장 방식Block StorageObject Storage 주 용도EC2 DiskFile / Object 저장 EC2 연결필요필요 없음 File System사용사용하지 않음 확장Volume 크기 관리자동 확장 대표 사용OS, DB DiskImage, Backup, Log

## 25. 실습: S3 기본 구조

* S3는 크게 다음 구조로 데이터를 관리

S3 Bucket │ ├── images/ │ ├── cat.jpg │ └── dog.jpg │ ├── documents/ │ └── report.pdf │ └── index.html

* S3의 핵심 구성 요소는 다음과 같다.
  * Bucket
  * Object
  * Key
  * Value
  * Metadata

## 26. 실습: Bucket

* Bucket은 S3에서 Object를 저장하는 가장 상위의 저장 공간이다.
* 일반적인 File System으로 생각하면 최상위 Directory와 비슷하게 보일 수 있지만, 실제 S3는 일반적인 Directory 기반 File System이 아니다.

```
예)
my-soldesk-bucket
 ├ images/cat.jpg
 ├ images/dog.jpg
 └ index.html
```

* 여기서 my-soldesk-bucket 이 Bucket 이름이다.

## 27. 실습: Object

* S3에 실제로 저장되는 데이터를 Object라고

```
예)
 # image.jpg
 # index.html
 # backup.zip
 # access.log
 # video.mp4
```

* 각 Object는 다음과 같은 정보로 구성된다.
  * Key
  * Value
  * Metadata
  * Version ID

```
예)
Bucket 이름
my-soldesk-bucket

        │
        └── images/aws.png
```

* Bucket: 파일을 저장하는 S3의 저장 공간
* Object: S3에 실제로 저장된 파일 하나
* Key: Bucket 안에서 Object를 구분하는 이름 또는 경로
* Value: 실제 파일 내용
* Metadata: 그 파일에 대한 부가 정보 (파일 크기, 업로드 날짜 ,암호화 정보)

## 28. 실습: S3의 Folder 구조

* S3 Console에서는 Folder가 존재하는 것처럼 보인다.

```
예)
images/
documents/
backup/
```

* 하지만 실제 S3는 전통적인 File System의 Directory 구조를 사용하는 것이 아니다.
* 다음 Object가 있다고 가정
  * images/aws.png
* 실제로 Object Key 자체가 "images/aws.png" 이다.
* 즉 "/" 문자를 이용하여 Console에서 Folder처럼 표현하는 것이다.
* 이를 Prefix라고

```
예)
images/aws.png
images/terraform.png
images/linux.png
```

* 위 Object의 공통 Prefix는 “images/” 이다.

## 29. 실습: S3 데이터 접근 방식

* S3는 일반적으로 HTTP 또는 HTTPS 기반 API를 통해 데이터에 접근
* AWS CLI를 이용할 수도 있다.
* 현재 AWS Account의 S3 Bucket 목록 확인
  * aws s3 ls
* 특정 Bucket 확인
  * aws s3 ls s3://my-soldesk-bucket

## 30. 실습: S3 Bucket 생성

* Terraform에서는 aws\_s3\_bucket Resource를 사용하여 Bucket을 생성할 수 있다.

```hcl
   # main.tf
resource "aws_s3_bucket" "my_bucket" {
  bucket = "soldesk-s3-example-123456789"
  tags = {
    Name        = "soldesk-s3"
    Environment = "dev"
  }
}
```

* 코드 설명
* aws\_s3\_bucket
  * AWS S3 Bucket을 생성하는 Terraform Resource
* my\_bucket
  * Terraform 내부에서 사용하는 Resource 이름
* bucket
  * 실제 AWS에 생성될 S3 Bucket 이름
* tags
  * S3 Bucket 관리에 사용할 Tag

## 31. 실습: S3 Bucket 이름

* S3 Bucket 이름은 같은 AWS Account 안에서만 구분되는 이름이 아니다.
* S3 General Purpose Bucket 이름은 전역 Namespace에서 고유해야
* 이미 다른 AWS 사용자가 사용하고 있는 Bucket 이름은 사용할 수 없다.
  * 예) my-s3-bucket
* 다른 사용자가 위 이름을 이미 사용 중이라면 같은 이름으로 Bucket을 생성할 수 없다. 따라서 실습에서는 AWS Account ID를 Bucket 이름에 포함하는 방법을 많이 사용

## 32. 실습: AWS Account ID를 이용한 Bucket 이름 생성

```hcl
   # main.tf
# 현재 AWS Account 정보 조회
data "aws_caller_identity" "current" {
}

# Bucket 이름 생성
locals {
  bucket_name = "soldesk-s3-${data.aws_caller_identity.current.account_id}"
}

# S3 Bucket 생성
resource "aws_s3_bucket" "my_bucket" {
  bucket = local.bucket_name
  tags = {
    Name = "soldesk-s3"
  }
}
```

* 예를 들어 Account ID가 “123456789012” 이라면 Bucket 이름은 "soldesk-s3-123456789012"이 된다.

## 33. 실습: S3 ARN

* AWS Resource는 ARN(Amazon Resource Name)을 사용하여 식별할 수 있다.
* Bucket ARN예) arn:aws:s3:::soldesk-s3-123456789012
* Object ARN예) arn:aws:s3:::soldesk-s3-123456789012/images/aws.png
* Bucket과 Object의 ARN이 서로 다르다는 점이 중요하다.
  * Bucket: arn:aws:s3:::my-bucket
  * Object: arn:aws:s3:::my-bucket/\*
* IAM Policy나 Bucket Policy를 작성할 때 자주 사용된다.

## 34. 실습: S3 Object Upload

* Terraform으로 S3 Bucket에 파일을 업로드할 수도 있다.
* 예를 들어 현재 Terraform Directory에 다음 파일이 있다고 가정

```hcl
 # index.html

   # main.tf
resource "aws_s3_object" "index" {
  bucket = aws_s3_bucket.my_bucket.id
  key    = "index.html"
  source = "${path.module}/index.html"
  content_type = "text/html"
}
```

## 35. 실습: 코드 설명

* bucket
  * Object를 업로드할 S3 Bucket
* key
  * S3에서 사용할 Object 이름
* source
  * Local PC에 존재하는 실제 File 경로
* content\_type
  * Object의 Content-Type 지정

## 36. 실습: S3 Object 경로

* 위 Terraform으로 생성되는 구조
* 다음처럼 Key를 설정하면

```hcl
key = "images/aws.png"
```

* S3 Console에서는 다음처럼 보인다. images └── aws.png
* 하지만 실제 Object Key는"images/aws.png"이다.

## 37. 실습: S3 Versioning

* Versioning은 동일한 Object의 여러 Version을 저장하는 기능이다.
* 예를 들어"index.html"File을 업로드했다고 가정
* 처음 내용

```
<h1>Hello AWS</h1>
```

* 이후 File을 수정

```
<h1>Hello Soldesk</h1>
```

* Versioning을 사용하지 않는 경우 기존 Object가 덮어써진다.

```
index.html

Hello Soldesk
```

* Versioning을 사용하면

```
index.html
 ├ Version 3
 ├ Version 2
 └ Version 1
```

* 처럼 이전 Version을 보관할 수 있다.
* Versioning을 사용하면 다음 상황에서 복구가 가능하다.
  * 실수로 File 덮어쓰기
  * 잘못된 File 업로드
  * Object 삭제
  * 이전 Version 복구

## 38. 실습: Terraform으로 Versioning 활성화

```hcl
   # main.tf
resource "aws_s3_bucket_versioning" "my_bucket_versioning" {
  bucket = aws_s3_bucket.my_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

## 39. 실습: 코드 설명

* bucket
  * Versioning을 적용할 S3 Bucket
* versioning\_configuration
  * Versioning 설정
* status = "Enabled"
  * S3 Versioning 활성화
* 구조 S3 Bucket

```
index.html
 ├ Version 3
 ├ Version 2
 └ Version 1
```

## 40. 실습: Versioning에서 Object 삭제

* Versioning을 사용하는 Bucket에서 Object를 일반적으로 삭제하면 기존 Version 자체가 즉시 완전히 삭제되는 방식과 다르게 동작

```
예)
index.html
 ├ Delete Marker
 ├ Version 3
 ├ Version 2
 └ Version 1
```

* Delete Marker가 현재 Version 역할을 하기 때문에 Object가 삭제된 것처럼 보인다.
* 필요하면 이전 Version을 다시 사용할 수 있다.

## 41. 실습: Terraform S3 Static File Public Access 실습

* Terraform을 사용하여 다음 환경을 구성
  *
    1. AWS Provider 설정
  *
    2. S3 Bucket 생성
  *
    3. index.html 파일 업로드
  *
    4. S3 Public Access Block 해제
  *
    5. Bucket Policy를 이용하여 index.html 공개
  *
    6. 브라우저에서 index.html 접속
  * 프로젝트 구조
* terraform-s3-lab/ │ ├── variables.tf ├── main.tf ├── output.tf ├── terraform.tfvars └── index.html

## 42. 실습: STEP 0. 실습 디렉터리 생성

* Terraform 실습을 진행할 디렉터리를 생성
* PowerShell에서 실행
  * mkdir C:\terraform-s3-lab
  * cd C:\terraform-s3-lab

STEP 1. AWS Provider 설정 + S3 Bucket 생성

* 첫 번째 단계에서는 AWS Provider를 설정하고 가장 기본적인 S3 Bucket을를 생성

S3 Bucket 이름은 전 세계에서 유일해야

따라서 현재 AWS 계정 ID와 Region을 Bucket 이름에 포함하여 중복 가능성을 낮춘다.

* terraform-s3-lab\variables.tf

```hcl
variable "aws_region" {
  description = "AWS 리전"
  type        = string
  default     = "ap-northeast-2"
}

variable "aws_profile" {
  description = "AWS CLI Profile 이름"
  type        = string
  default     = "my-profile"
}

variable "bucket_prefix" {
  description = "S3 Bucket 이름 앞부분"
  type        = string
  default     = "soldesk-s3"
}

variable "project_name" {
  description = "프로젝트 이름"
  type        = string
  default     = "s3-lab"
}

variable "environment" {
  description = "환경 이름"
  type        = string
  default     = "training"
}

   # terraform-s3-lab\main.tf
terraform {
  # 사용할 Terraform 최소 버전
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

# 현재 Terraform을 실행하는 AWS 계정 정보 조회
#
account_id를 가져와 S3 Bucket 이름에 사용
data "aws_caller_identity" "current" {
}

   # terraform-s3-lab\output.tf
output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "user_id" {
  value = data.aws_caller_identity.current.user_id
}

output "arn" {
  value = data.aws_caller_identity.current.arn
}

PS C:\my-terraform> terraform plan
Changes to Outputs:
  + account_id = "123456789012"
  + anr        = "arn:aws:iam::123456789012:user/admin"
  + user_id    = "AIDA5A62EVKFSAZ6N4LW2"

   # terraform-s3-lab\main.tf
locals {
  # S3 Bucket 이름 생성
  # 예: soldesk-s3-basic-123456789012-ap-northeast-2
  # S3 Bucket 이름은 전 세계에서 유일해야 하기 때문에 AWS Account ID와 Region을 이름에 포함
  # "soldesk-s3-basic-123456789012-ap-northeast-2"에서 "123456789012"이 값을 조회
  bucket_name = "${var.bucket_prefix}-basic-${data.aws_caller_identity.current.account_id}-${var.aws_region}"
}

# S3 Bucket 생성
resource "aws_s3_bucket" "my_bucket" {
  # locals에서 생성한 Bucket 이름 사용
  bucket = local.bucket_name

  # Resource Tag
  tags = {
    Name = "${var.project_name}-basic""
  }
}
   # terraform-s3-lab\output.tf
output "bucket_name" {
  description = "생성된 S3 Bucket 이름"
  value = aws_s3_bucket.my_bucket.bucket
}

output "bucket_arn" {
  description = "생성된 S3 Bucket ARN"
  value = aws_s3_bucket.my_bucket.arn
}

PS C:\my-terraform> terraform init
PS C:\my-terraform> terraform validate
PS C:\my-terraform> terraform plan
```

## 43. 실습: STEP 2. index.html을 S3 Object로 업로드

* 이번 단계에서는 생성한 S3 Bucket에 index.html 파일을 업로드
* Bucket └── soldesk-s3-basic-123456789012-ap-northeast-2
* Object └── index.html
  * terraform-s3-lab\main.tf

## 44. 실습: index.html 파일을 S3 Bucket에 업로드

```hcl
resource "aws_s3_object" "index" {
  # Object를 저장할 S3 Bucket
  bucket = aws_s3_bucket.my_bucket.id

  key = "index.html"  # S3 내부에서 사용할 Object Key
  source = "${path.module}/index.html"# Terraform 프로젝트 디렉터리에 있는 index.html 업로드
  content_type = "text/html"  # 브라우저가 HTML 문서로 인식하도록 Content-Type 설정
}

PS C:\my-terraform> terraform init
PS C:\my-terraform> terraform validate
PS C:\my-terraform> terraform plan
```

## 45. 실습: STEP 3. S3 Public Access Block 해제

* S3는 기본적으로 외부에서 Public 접근하지 못하도록 Public Access Block 기능이 활성화되어 있다.
* 이번 실습에서는 index.html 파일을 인터넷에서 조회할 것이므로 Public Access Block을 해제
* 주의 : 실무에서는 S3를 무조건 Public으로 설정하지 않는다.
* 이번 설정은 Public Access와 Bucket Policy 동작을 이해하기 위한교육 실습용 설정이다.
  * terraform-s3-lab\main.tf

## 46. 실습: S3 Public Access Block 해제

```hcl
resource "aws_s3_bucket_public_access_block" "public_access" {

  bucket = aws_s3_bucket.my_bucket.id# 설정을 적용할 S3 Bucket
  block_public_acls = false# Public ACL 차단 해제
  ignore_public_acls= false# Public ACL 무시 기능 해제
  block_public_policy = false# Public Bucket Policy 차단 해제
  restrict_public_buckets = false# Public Bucket 제한 해제
}

PS C:\my-terraform> terraform plan
```

* S3 Bucket --> 권한 --> 퍼블릭 액세스 차단(버킷 설정) 확인

![S3 Bucket  -->  권한  -->  퍼블릭 액세스 차단버킷 설정 확인 화면](<../.gitbook/assets/1 (5).png>)

## 47. 실습: STEP 4. Bucket Policy를 이용하여 index.html Public 공개

* Public Access Block을 해제했으므로 이번에는 Bucket Policy를 생성
* Bucket Policy에서 모든 사용자에게 "s3:GetObject" 권한을 허용하면 인터넷 사용자들이 S3 Object를 읽을 수 있게 된다.
  * terraform-s3-lab\main.tf

## 48. 실습: S3 Bucket Policy : 인터넷의 모든 사용자가 Bucket 내부의 Object를 읽을 수 있도록 허용

```hcl
resource "aws_s3_bucket_policy" "public_read" {
  bucket = aws_s3_bucket.my_bucket.id# Policy를 적용할 S3 Bucket

  # Public Access Block 해제가 먼저 완료된 후 Bucket Policy를 생성하도록 의존성을 설정
  depends_on = [ aws_s3_bucket_public_access_block.public_access  ]

  # Terraform Object를 JSON 형식의 IAM Policy로 변환
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "PublicRead" # Policy Statement 이름
        Effect = "Allow"# 허용자
        Principal = "*"# 모든 사용자
        Action = [ "s3:GetObject" ]# S3 Object 읽기 권한
        Resource = "${aws_s3_bucket.my_bucket.arn}/*"# Bucket 내부의 모든 Object
      }
    ]
  })
}

   # terraform-s3-lab\output.tf
# index.html Object URL
output "index_url" {
  description = "index.html 접속 URL"
  value = "https://${aws_s3_bucket.my_bucket.bucket}.s3.${var.aws_region}.amazonaws.com/${aws_s3_object.index.key}"
}

PS C:\my-terraform> terraform plan
PS C:\my-terraform> terraform apply
```

![PS C:\my-terraform> terraform apply 화면](<../.gitbook/assets/2 (5).png>)

## 49. 실습: output으로 출력된 경로로 브라우저를 통해서 접속

https://soldesk-s3-bucket-basic-123456789012-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/index.html

![https://soldesk-s3-bucket-basic-123456789012-ap-northeast-2. 화면](<../.gitbook/assets/3 (5).png>)

## 50. 실습: STEP 5. terraform.tfvars 작성

* variables.tf에는 Variable의 구조와 기본값을 정의
* terraform.tfvars에서는 실제 실습에서 사용할 값을 지정
* terraform.tfvars에 지정된 값이 있으면 variables.tf의 default보다 terraform.tfvars 값이 우선 사용된다.
  * terraform-s3-lab\terraform.tfvars

## 51. 실습: AWS Region

```hcl
aws_region = "ap-northeast-2"

# AWS CLI Profile
aws_profile = "my-profile"

# S3 Bucket 이름 앞부분
bucket_prefix = "soldesk-s3"

# 프로젝트 이름
project_name = "s3-lab"

# 환경 이름
environment = "training"

# STEP 7. AWS CLI로 확인

# 현재 AWS Account의 S3 Bucket 목록을 확인
PS C:\my-terraform> aws s3 ls --profile my-profile

# Bucket 확인
PS C:\trf2\2) S3\2-1_S3_basic> aws s3 ls
2026-09-22 12:04:41 soldesk-s3-bucket-basic-123456789012-ap-northeast-2

# Bucket Object 확인
PS C:\trf2\2) S3\2-1_S3_basic> aws s3 ls s3://soldesk-s3-bucket-basic-123456789012-ap-northeast-2
2026-09-22 12:04:43      20007 index.html
```

* 예: aws s3 ls s3://soldesk-s3-basic-123456789012-ap-northeast-2 --profile my-profile

## 52. 실습: 파일 업로드

형식 :　aws s3 cp ./index.html s3://버킷이름/index.html

```powershell
PS C:\trf2\2) S3\2-1_S3_basic>
 aws  s3  cp  ./route53.html  s3://soldesk-s3-bucket-basic-123456789012-ap-northeast-2/route53.html
upload: .\route53.html to s3://soldesk-s3-bucket-basic-123456789012-ap-northeast-2/route53.html

PS C:\trf2\2) S3\2-1_S3_basic> aws  s3  ls  s3://soldesk-s3-bucket-basic-123456789012-ap-northeast-2
2026-09-22 12:04:43      20007 index.html
2026-09-22 12:12:33     649736 route53.html

# 파일 다운로드
형식 :　aws s3 cp s3://버킷이름/index.html ./index.html

PS C:\trf2\2) S3\2-1_S3_basic>
 aws s3 cp  s3://soldesk-s3-bucket-basic-123456789012-ap-northeast-2/index.html  ./index2.html
download: s3://soldesk-s3-bucket-basic-123456789012-ap-northeast-2/index.html to .\index2.html

# STEP 9. Terraform Resource 삭제

PS C:\my-terraform> terraform destroy
```

## 53. 실습: Terraform을 활용한 S3 Versioning 실습

* S3 Bucket을 Terraform으로 생성
* S3 Bucket에 Versioning을 활성화
* Versioning 테스트를 위해 document.txt Object를 생성
* document.txt의 내용을 변경한 뒤 다시 terraform apply를 실행
* 같은 Key의 Object가 덮어써지는 것이 아니라 새로운 Version으로 저장되는 것을 확인

## 54. 실습: STEP 1. Terraform 기본 설정

* Terraform에서 사용할 AWS Provider와 AWS Region, AWS CLI Profile을 설정
* 이 단계에서는 아직 S3 Bucket을 생성하지 않는다.
* Terraform이 AWS와 통신할 수 있도록 기본 환경만 구성

```hcl
   # variables.tf
variable "aws_region" {
  description = "AWS 리전"
  type        = string
  default     = "ap-northeast-2"
}

variable "aws_profile" {
  description = "AWS CLI Profile 이름"
  type        = string
  default     = "default"
}

   # main.tf
terraform {
  required_version = ">= 1.16.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile
}

   # terraform.tf
aws_region  = "ap-northeast-2"
aws_profile = "default"

terraform init
```

## 55. 실습: STEP 2. 현재 AWS Account 정보 조회

* S3 Bucket 이름은 다른 AWS 사용자와 중복될 수 없다. 따라서 현재 Terraform이 접속한 AWS Account ID를 조회하여 Bucket 이름에 사용할 수 있도록
* data Block은 새로운 AWS Resource를 생성하는 것이 아니라 AWS에 존재하는 정보 또는 현재 Account 정보를 조회

```hcl
   # main.tf
data "aws_caller_identity" "current" {
}

   # outputs.tf
output "account_id" {
  description = "현재 AWS Account ID"
  value       = data.aws_caller_identity.current.account_id
}
```

* 현재 AWS Account 정보를 조회
* Resource를 생성하지 않고 Output을 통해 Account ID를 확인

```hcl
terraform plan
```

## 56. 실습: STEP 3. S3 Bucket 이름 생성

* S3 Bucket 이름은 고유해야
* 실습마다 이름 충돌이 발생하지 않도록 다음 정보를 조합하여 Bucket 이름을 생성
  * Bucket Prefix
  * AWS Account ID
  * AWS Region
* 예)

```hcl
bucket_prefix = "soldesk-s3"
Account ID = 123456789012
Region = ap-northeast-2
```

* 생성 결과 : soldesk-s3-versioning-123456789012-ap-northeast-2

```hcl
   # variables.tf
variable "bucket_prefix" {
  description = "S3 Bucket 이름 Prefix"
  type        = string
  default     = "soldesk-s3"
}

   # main.tf
locals {
  bucket_name = "${var.bucket_prefix}-versioning-${data.aws_caller_identity.current.account_id}-${var.aws_region}"
}

   # outputs.tf
output "bucket_name" {
  description = "생성할 S3 Bucket 이름"
  value       = local.bucket_name
}

PS C:\trf2\2) S3\2-２_S3_versioning>　terraform plan
```

* 다음과 같은 형식으로 Bucket 이름이 생성되는지 확인
  * soldesk-s3-versioning-123456789012-ap-northeast-2

## 57. 실습: STEP 4. S3 Bucket 생성

* 실제 S3 Bucket을 생성
* 앞에서 이해한 Bucket 이름 생성 방식을 적용
* S3 Bucket에는 관리 목적으로 Tag를 추가

```hcl
   # variables.tf
variable "project_name" {
  description = "Project 이름"
  type        = string
  default     = "s3-lab"
}

variable "environment" {
  description = "환경 이름"
  type        = string
  default     = "training"
}

   # main.tf
resource "aws_s3_bucket" "my_bucket" {
  bucket = local.bucket_name

  tags = {
    Name = "${var.project_name}-versioning"
    Environment = var.environment
    ManagedBy = "Terraform"
  }
}

   # terraform.tf
project_name  = "s3-lab"
environment   = "training"

PS C:\trf2\2) S3\2-２_S3_versioning>　terraform plan
PS C:\trf2\2) S3\2-２_S3_versioning>　terraform apply
```

## 58. 실습: STEP 5. S3 Versioning 활성화

* S3 Versioning은 동일한 Key의 Object가 다시 업로드되었을 때 기존 Object를 완전히 덮어쓰지 않고 각각의 Version으로 보관하는 기능이다.
* 예) document.txt │ ├── Version 3 ├── Version 2 └── Version 1
* 이번 STEP에서는 Versioning 기능 자체만 확인
* Version 관리는 기본값으로 비활성화 되어있다.

![Version 관리는 기본값으로 비활성화 되어있다. 화면](<../.gitbook/assets/4 (5).png>)

```hcl
   # main.tf
resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.my_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

   # outputs.tf
output "versioning_status" {
  value = aws_s3_bucket_versioning.versioning.versioning_configuration[0].status
}

PS C:\trf2\2) S3\2-２_S3_versioning>　terraform plan
```

* 다음 Resource를 확인
  * aws\_s3\_bucket.my\_bucket
  * aws\_s3\_bucket\_versioning.versioning

```powershell
PS C:\trf2\2) S3\2-２_S3_versioning>　terraform apply
```

* S3 Bucket을 생성하고 해당 Bucket에 Versioning을 활성화

![S3 Bucket을 생성하고 해당 Bucket에 Versioning을 활성화 화면](<../.gitbook/assets/5 (4).png>)

## 59. 실습: STEP 6. Versioning 테스트 Object 생성

* Versioning이 실제로 동작하는지 확인하기 위해 document.txt Object를 생성
* Object의 실제 내용은 Variable을 사용
* 처음에는 다음 값을 사용
  * S3 Version 1

Bucket │ └── document.txt │ └── S3 Version 1

```hcl
   # variables.tf
variable "object_content" {
  description = "Versioning 확인용 document.txt 내용"
  type        = string
  default     = "S3 Version 1"
}

   # main.tf
# Versioning 테스트용 Object 생성
resource "aws_s3_object" "document" {
  # Object를 저장할 S3 Bucket 지정 앞에서 생성한 aws_s3_bucket.my_bucket을 사용
  bucket = aws_s3_bucket.my_bucket.id

  # S3 Bucket 안에서 사용할 Object 이름, document.txt가 Object의 Key가 된다.
  key = "document.txt"

  # document.txt 파일 안에 저장될 실제 내용, variables.tf의 object_content 변수 값을 사용
  content = var.object_content

  # Object의 Content-Type 설정, 일반 Text File이므로 text/plain 사용
  content_type = "text/plain"

  # S3 Versioning 설정이 먼저 완료된 후 document.txt Object가 생성되도록 의존성 지정
  depends_on = [ aws_s3_bucket_versioning.versioning ]
}

   # outputs.tf
output "object_key" {
  description = "Versioning 확인용 Object Key"
  value       = aws_s3_object.document.key
}

output "object_content" {
  description = "Versioning 확인용 Object Content"
  value = aws_s3_object.document.content
}

   # terraform.tf
object_content = "S3 Bucket Version 1 Test"
```

* key = "document.txt"
  * S3에 저장되는 Object의 Key이다.
  * 쉽게 말하면 S3에서 Object를 구분하는 이름이다.
* content = var.object\_content
  * document.txt 안에 저장될 실제 내용을 지정
  * S3 Bucket Version 1 Test
* content\_type = "text/plain"
  * 일반 Text File이라는 것을 의미
* depends\_on
  * S3 Versioning이 활성화된 후 Object를 생성하도록 순서를 지정

```powershell
PS C:\trf2\2) S3\2-２_S3_versioning>　terraform plan
Changes to Outputs:
  + object_content    = "S3 Bucket Version 1 Test"
  + object_key        = "document.txt"

PS C:\trf2\2) S3\2-２_S3_versioning>　terraform apply
```

* document.txt 파일을 다운로드 후 파일 내용 확인

```powershell
PS C:\trf2\2) S3\2-2_S3_versioing>
 aws s3 cp  s3://soldesk-s3-versioning-123456789012-ap-northeast-2/document.txt  ./document.txt
download: s3://soldesk-s3-versioning-123456789012-ap-northeast-2/document.txt to .\document.txt
```

![download: s3://soldesk-s3-versioning-123456789012-ap-northea 화면](<../.gitbook/assets/6 (3).png>)

## 60. 실습: STEP 7. Version 2 생성

* 동일한 Object Key인 document.txt의 내용만 변경
* Key는 변경하지 않는다.
* 기존 : document.txt
* 내용 : S3 Version 1
* 변경 : document.txt
* 내용 : S3 Version 2
* Versioning이 활성화되어 있기 때문에 Version 1을 덮어쓰는 것이 아니라 새로운 Version이 생성된다.

```hcl
   # terraform.tf
object_content = "S3 Bucket Version 2 Test 2"

PS C:\my-terraform> terraform plan
Plan: 0 to add, 1 to change, 0 to destroy.

Changes to Outputs:
  ~ object_content    = "S3 Bucket Version 1 Test" -> "S3 Bucket Version 2 Test 2"

PS C:\my-terraform> terraform apply
Plan: 0 to add, 1 to change, 0 to destroy.

Changes to Outputs:
  ~ object_content    = "S3 Bucket Version 1 Test" -> "S3 Bucket Version 2 Test 2"
```

* AWS Console에서
  * S3 --> Bucket3 --> Objects3 --> Show versions 활성화
* 다음과 같이 Version이 2개 존재하는지 확인 document.txt ├── Version 2 └── Version 1

## 61. 실습: STEP 8. Version 3 생성

* 같은 방법으로 Object 내용을 한 번 더 변경
* 이번에도 Key는 document.txt 그대로 유지

```hcl
   # terraform.tf
object_content = "S3 Bucket Version 3 Test 3"
```

* document.txt 내용이 Version 2에서 Version 3으로 변경되는 것을 확인

```powershell
PS C:\my-terraform> terraform apply
Changes to Outputs:
  ~ object_content    = "S3 Bucket Version 2 Test 2" -> "S3 Bucket Version 3 Test 3"
```

* AWS Console로 접속해서 확인해보면 3개의 버전이 확인된다.

![AWS Console로 접속해서 확인해보면 3개의 버전이 확인된다. 화면](<../.gitbook/assets/7 (3).png>)

## 62. 실습: STEP 10. AWS CLI를 이용한 Delete Marker 생성 및 복구 실습

* 이번 단계에서는 Versioning이 활성화된 S3 Bucket에서 document.txt Object를 삭제한다.
* Versioning이 활성화된 Bucket에서는 Version ID를 지정하지 않고 Object를 삭제하면 기존 Version을 실제로 삭제하지 않고 Delete Marker를 생성한다.
* Delete Marker가 생성되면 Delete Marker가 최신 Version이 되기 때문에 일반적인 S3 Object 조회에서는 document.txt가 삭제된 것처럼 보인다.
* 이후 Delete Marker의 Version ID를 확인하고 Delete Marker 자체를 삭제하여 기존 document.txt가 다시 보이는 것을 확인한다.

\[현재 상태] document.txt

├── Version 3 ← Latest ├── Version 2 └── Version 1

## 63. 실습: STEP 10-1. 현재 Object 확인

* 먼저 document.txt가 정상적으로 존재하는지 확인한다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioning>
aws  s3  ls  s3://soldesk-s3-versioning-123456789012-ap-northeast-2
2026-09-22 13:18:18         26 document.txt

# 현재 document.txt 내용을 다운로드해서 확인
PS C:\trf2\2) S3\2-2_S3_versioning>
aws s3 cp s3://soldesk-s3-versioning-123456789012-ap-northeast-2/document.txt ./document-current.txt
download: s3://soldesk-s3-versioning-123456789012-ap-northeast-2/document.txt to .\document.txt

# 파일 내용 확인
PS C:\trf2\2) S3\2-2_S3_versioning> Get-Content .\document-current.txt
S3 Bucket Version 3 Test 3
```

## 64. 실습: STEP 10-2. 현재 Object Version 목록 확인

* Version을 삭제하기 전에 현재 Version 정보를 확인한다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioning> aws s3api list-object-versions \
```

* -bucket soldesk-s3-versioning-123456789012-ap-northeast-2 --prefix document.txt
* aws s3api
  * AWS CLI에서 S3의 세부 API 기능을 사용하는 명령어
  * aws s3보다 Versioning 같은 세밀한 기능을 확인할 때 많이 사용
* list-object-versions
  * Bucket 안에 저장된 Object Version 목록을 조회
  * 일반 Object뿐 아니라 Delete Marker도 같이 확인할 수 있다.
* -bucket soldesk-s3-versioning-123456789012-ap-northeast-2
  * 어떤 S3 Bucket을 조회할지 지정
* -prefix document.txt
  * Key가 document.txt로 시작하는 Object만 조회
  * document.txt의 Version만 보기 위해 넣은 옵션
* 다음과 같이 여러 Version이 존재하는 것을 확인한다.
* 각 Version에는 고유한 VersionId가 존재한다.

```
예)
Version 3
"Key": "document.txt"
"VersionId": "XXXXXXXX-Version3"
"IsLatest": true

Version 2
"Key": "document.txt"
"VersionId": "XXXXXXXX-Version2"
"IsLatest": false

Version 1
"Key": "document.txt"
"VersionId": "XXXXXXXX-Version1"
"IsLatest": false
```

## 65. 실습: STEP 10-3. document.txt 삭제

* Version ID를 지정하지 않고 document.txt를 삭제한다.
* Versioning이 활성화된 Bucket이기 때문에 Version 3 자체가 삭제되는 것이 아니라 Delete Marker가 생성된다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioning>
aws s3api delete-object --bucket soldesk-s3-versioning-123456789012-ap-northeast-2 --key document.txt
{
    "DeleteMarker": true,
    "VersionId": "b3IX2coFcHORhThgw.99mufbsxiw6k4r"
}
```

* 실행 결과에서 DeleteMarker와 VersionId를 확인할 수 있다.

document.txt ├── Delete Marker ← Latest ├── Version 3 ├── Version 2 └── Version 1

* 실제 Version 1, Version 2, Version 3은 삭제되지 않는다.
* Delete Marker만 새로운 최신 Version으로 추가된다.

## 66. 실습: STEP 10-4. 일반 AWS CLI에서 document.txt가 안 보이는지 확인

* Delete Marker가 최신 Version이 되었기 때문에 일반 Object 목록에서는 document.txt가 존재하지 않는 것처럼 보인다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioning> aws s3 ls s3://soldesk-s3-versioning-123456789012-ap-northeast-2
```

* document.txt가 출력되지 않는 것을 확인
* aws s3 ls --> 현재 Object 기준으로 조회 --> Delete Marker가 Latest --> document.txt가 보이지 않음
* document.txt 파일이 확인되지 않는다.

![document.txt 파일이 확인되지 않는다. 화면](<../.gitbook/assets/8 (3).png>)

* 버전 표시를 활성화하면 확인된다.

![버전 표시를 활성화하면 확인된다. 화면](<../.gitbook/assets/9 (3).png>)

## 67. 실습: STEP 10-5. document.txt 다운로드 테스트

* 일반적인 방법으로 document.txt 다운로드를 시도한다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioning>
aws s3 cp s3://soldesk-s3-versioning-123456789012-ap-northeast-2/document.txt ./document-delete-test.txt
fatal error: An error occurred (404) when calling the HeadObject operation: Key "document.txt" does not exist
```

* Delete Marker가 현재 Version이기 때문에 다운로드할 수 없다.
* 이것은 Version 1, Version 2, Version 3이 실제로 삭제되었다는 의미가 아니다.
* Delete Marker가 현재 Version이기 때문에 일반적인 방법으로 Object를 조회할 수 없는 상태이다.

## 68. 실습: STEP 10-6. Version 목록을 조회해서 Delete Marker 확인

* 일반 aws s3 ls에서는 document.txt가 보이지 않지만 list-object-versions 명령어를 사용하면 기존 Version과 Delete Marker를 확인할 수 있다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioing>
aws  s3api  list-object-versions  --bucket  `
soldesk-s3-versioning-123456789012-ap-northeast-2  --prefix  document.txt
~~~~~~~~~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~~~~~~~~~
    "DeleteMarkers": [
        {
            "Owner": {
                "ID": "fe1fd93ff3db2c0bcb438c698d146a0463d03da28b8e68bd892fab97c1279de0"
            },
            "Key": "document.txt",
            "VersionId": "b3IX2coFcHORhThgw.99mufbsxiw6k4r",
            "IsLatest": true,
            "LastModified": "2026-09-22T05:43:59+00:00"
        }
    ],
    "RequestCharged": null,
    "Prefix": "document.txt"
}

[현재 구조]
document.txt

   ├── Delete Marker    ← IsLatest = true
   ├── Version 3        ← IsLatest = false
   ├── Version 2        ← IsLatest = false
   └── Version 1        ← IsLatest = false
```

## 69. 실습: STEP 10-7 Delete Marker 삭제

* 이번에는 document.txt를 삭제하는 것이 아니라 Delete Marker 자체를 삭제한다.
* 따라서 반드시 Delete Marker의 VersionId를 지정해야 한다.

```powershell
PS C:\trf2\2) S3\2-2_S3_versioing>
aws  s3api  list-object-versions  --bucket  `
soldesk-s3-versioning-123456789012-ap-northeast-2  --prefix  document.txt
~~~~~~~~~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~~~~~~~~~
    "DeleteMarkers": [
        {
            "Owner": {
                "ID": "fe1fd93ff3db2c0bcb438c698d146a0463d03da28b8e68bd892fab97c1279de0"
            },
            "Key": "document.txt",
            "VersionId": "b3IX2coFcHORhThgw.99mufbsxiw6k4r",
            "IsLatest": true,
            "LastModified": "2026-09-22T05:43:59+00:00"
        }
    ],
    "RequestCharged": null,
    "Prefix": "document.txt"
}

형식)
aws s3api delete-object --bucket <Bucket-Name> --key document.txt --version-id <Delete-Marker-Version-ID>

PS C:\trf2\2) S3\2-2_S3_versioing>
aws  s3api  delete-object  --bucket  soldesk-s3-versioning-123456789012-ap-northeast-2  `
```

* -key document.txt --version-id b3IX2coFcHORhThgw.99mufbsxiw6k4r

```json
{
    "DeleteMarker": true,
    "VersionId": "b3IX2coFcHORhThgw.99mufbsxiw6k4r"
}

Delete Marker 삭제 전
document.txt

   ├── Delete Marker    ← Latest
   ├── Version 3
   ├── Version 2
   └── Version 1

Delete Marker 삭제 후
document.txt

   ├── Version 3        ← 다시 현재 Object
   ├── Version 2
   └── Version 1

# STEP 10-9. document.txt가 다시 보이는지 확인

PS C:\trf2\2) S3\2-2_S3_versioing> aws s3 ls s3://soldesk-s3-versioning-123456789012-ap-northeast-2
2026-09-22 13:18:18         26 document.txt

# STEP 10-10. document.txt 다운로드 및 내용 확인

PS C:\trf2\2) S3\2-2_S3_versioning>
aws s3 cp s3://soldesk-s3-versioning-123456789012-ap-northeast-2/document.txt ./document-restored.txt

# 모든 리소스 삭

PS C:\trf2\2) S3\2-2_S3_versioing> terraform  destroy  -auto-approve

# Terraform S3 Static Website + Route 53 실습

Terraform을 사용하여 S3 Static Website를 구성하고
Route 53 도메인과 연결하여 웹사이트에 접속할 수 있도록 구성한다.

01) s3-bucket
```

* S3 Bucket을 생성한다.
* Bucket 이름은 사용할 도메인 이름과 동일하게 설정한다.
* 예: www.aws-esk.com

```
02) s3-website
```

* 생성한 S3 Bucket에 Static Website Hosting을 설정한다.
* 기본 문서는 index.html로 설정한다.
* 오류 문서는 error.html로 설정한다.

```
03) s3-public
```

* S3 Public Access Block을 해제한다.
* Bucket Policy를 생성한다.
* 인터넷 사용자가 S3 Object를 읽을 수 있도록 s3:GetObject 권한을 허용한다.

```
04) s3-object
```

* index.html 파일을 S3 Bucket에 업로드한다.
* error.html 파일을 S3 Bucket에 업로드한다.
* S3 Static Website Endpoint를 이용하여 정상 페이지와 오류 페이지가 출력되는지 확인한다.

```
05) route53-s3
```

* 기존 Route 53 Hosted Zone을 Data Source로 조회한다.
* www.aws-esk.com A Alias Record를 생성한다.
* Route 53 Record를 S3 Static Website Endpoint에 연결한다.
* 최종적으로 다음 주소로 접속되는지 확인한다.

http://www.aws-esk.com

## 70. 실습: STEP 1. S3 정적 웹사이트용 Bucket 생성

* Route 53까지 연결할 예정이므로 S3 Bucket 이름을 최종 도메인과 동일하게 생성한다.
* 최종 접속 주소 : http://www.aws-esk.com
* 따라서 Bucket 이름 : www.aws-esk.com
  * 01\_s3-bucket\variables.tf

```hcl
variable "aws_region" {
  type        = string
  default     = "ap-northeast-2"
}

variable "aws_profile" {
  type        = string
  default     = "default"
}

variable "domain_name" {
  description = "Route 53에 등록된 기본 도메인"
  type        = string
  default     = "aws-esk.com"
}

variable "subdomain" {
  description = "S3 정적 웹사이트에서 사용할 서브도메인"
  type        = string
  default     = "www"
}

variable "project_name" {
  description = "프로젝트 이름"
  type        = string
  default     = "s3-website"
}

variable "environment" {
  description = "환경 이름"
  type        = string
  default     = "training"
}

   # 01_s3-bucket\main.tf
terraform {
  required_version = ">= 1.16.0"

  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# AWS Provider 설정
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile
}

locals {
  # Route 53에서 사용할 도메인과 S3 Bucket 이름을 동일하게 생성
  # 결과 : www.aws-esk.com
  bucket_name = "${var.subdomain}.${var.domain_name}"
}

# S3 Bucket 생성
resource "aws_s3_bucket" "website" {

  # Route 53 Record 이름과 동일한 Bucket 이름 사용
  bucket = local.bucket_name

  tags = {
    Name = var.project_name
    Environment = var.environment
    ManagedBy = "Terraform"
  }
}
```

* 여러 Variable을 조합하여 사용할 값을 만든다.
  * "${var.subdomain}.${var.domain\_name}"
* 결과 : www.aws-esk.com
  * 01\_s3-bucket\output.tf

```hcl
output "bucket_name" {
  description = "생성된 S3 Bucket 이름"
  value = aws_s3_bucket.website.bucket
}

output "bucket_arn" {
  description = "생성된 S3 Bucket ARN"
  value = aws_s3_bucket.website.arn
}

PS C:\my-terraform> terraform init

PS C:\my-terraform> plan
Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_arn  = (known after apply)
  + bucket_name = "www.aws-esk.com"
  + domain_name = "www.aws-esk.com"

PS C:\my-terraform> apply
Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_arn  = (known after apply)
  + bucket_name = "www.aws-esk.com"
  + domain_name = "www.aws-esk.com"
```

## 71. 실습: STEP 2. S3 Static Website Hosting 설정

* STEP 1에서 생성한 S3 Bucket에 정적 웹사이트 기능을 추가한다.
* STEP 1에서 생성한 aws\_s3\_bucket.website Resource를 그대로 사용한다.

```hcl
   # main.tf
# 기존 S3 Bucket에 Static Website Hosting 설정
resource "aws_s3_bucket_website_configuration" "website" {

  # STEP 1에서 생성한 기존 S3 Bucket 사용
  bucket = aws_s3_bucket.website.id

  # 웹사이트 기본 페이지
  index_document {
    suffix = "index.html"
  }

  # 오류 발생 시 출력할 페이지
  error_document {
    key = "error.html"
  }
}
```

* aws\_s3\_bucket\_website\_configuration
  * 기존 S3 Bucket에 Static Website Hosting 기능을 설정한다.
* bucket = aws\_s3\_bucket.website.id
  * STEP 1에서 생성한 S3 Bucket ID를 사용한다.
* index\_document
  * suffix = "index.html"
  * 사용자가 웹사이트의 루트 주소로 접속했을 때 index.html을 기본 페이지로 출력한다.
* error\_document
  * key = "error.html"
  * 존재하지 않는 페이지에 접근하면 error.html을 출력한다.

```hcl
   # output.tf
output "website_endpoint" {
  description = "S3 Static Website Endpoint"
  value = aws_s3_bucket_website_configuration.website.website_endpoint
}
```

* website\_endpoint
  * S3가 제공하는 정적 웹사이트 접속 주소를 출력한다.
  * 예 : www.aws-esk.com.s3-website.ap-northeast-2.amazonaws.com

```powershell
PS C:\my-terraform> plan
PS C:\my-terraform> apply
```

![PS C:\my-terraform> apply 화면](<../.gitbook/assets/11 (2).png>)

![PS C:\my-terraform> apply 화면](<../.gitbook/assets/12 (2).png>)

## 72. 실습: STEP 3. S3 Public Access 설정

* S3 Static Website는 인터넷 사용자가 index.html과 error.html을 읽을 수 있어야 한다.
* 이번 단계에서는:
  *
    1. Public Access Block 해제
  *
    2. Bucket Policy 설정

```hcl
   # main.tf
# S3 Public Access Block 해제
# aws_s3_bucket_public_access_block은S3의 Public Access Block 설정을 관리한다.
resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket                  = aws_s3_bucket.website.id# STEP 1에서 생성한 기존 Bucket
  block_public_acls       = false# Public ACL 차단 해제
  ignore_public_acls      = false# Public ACL 무시 기능 해제
  block_public_policy     = false# Public Bucket Policy 차단 해제
  restrict_public_buckets = false# Public Bucket 제한 해제
}

# S3 Bucket Policy
resource "aws_s3_bucket_policy" "public_read" {

  bucket = aws_s3_bucket.website.id  # STEP 1에서 생성한 기존 Bucket

  # Public Access Block 설정 완료 후 Bucket Policy 적용
  # aws_s3_bucket_policy는 S3 Bucket에 접근 정책을 적용한다.
  depends_on = [ aws_s3_bucket_public_access_block.public_access ]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "PublicReadGetObject"
        Effect = "Allow"# 접근 허용
        Principal = "*"# 모든 사용자
        Action = "s3:GetObject"# S3 Object 읽기 권한
        Resource = "${aws_s3_bucket.website.arn}/*"        # 해당 Bucket 내부의 모든 Object
      }
    ]
  })
}
   # output.tf
output "bucket_policy_resource" {
  description = "Public Read가 허용된 S3 Object ARN"
  value = "${aws_s3_bucket.website.arn}/*"
}

[코드 설명]
```

* Bucket Policy가 어느 Resource에 적용되는지 확인한다.
  * 예: arn:aws:s3:::www.aws-esk.com/\*

```powershell
PS C:\my-terraform> plan
PS C:\my-terraform> apply
```

## 73. 실습: STEP 4. index.html / error.html 업로드

* 이번 단계에서는 기존 S3 Bucket에 index.html, error.html 두 개의 HTML 파일을 업로드한다.
* 현재 프로젝트 구조:

```hcl
terraform-s3-website
│
├── variables.tf
├── main.tf
├── output.tf
├── terraform.tfvars
├── index.html
└── error.html
```

* etag = filemd5("${path.module}/index.html")
  * index.html 파일의 MD5 해시값을 계산해서 ETag로 사용하는 설정이다.
  * 파일 내용이 변경되면 MD5 값도 변경되므로, Terraform이 파일 변경 여부를 감지할 수 있다.
  * path.module: 현재 Terraform 모듈 경로
  * filemd5() : 파일의 MD5 해시값 계산
  * etag : 파일 변경 여부 확인에 사용

```hcl
   # main.tf
# index.html 업로드
resource "aws_s3_object" "index" {
  bucket = aws_s3_bucket.website.id# STEP 1에서 생성한 기존 S3 Bucket
  key = "index.html"# S3 내부 Object 이름
  source = "${path.module}/index.html"# 현재 Terraform 프로젝트의 index.html 사용
  content_type= "text/html; charset=utf-8"# 브라우저가 HTML 문서로 인식
  etag = filemd5("${path.module}/index.html")# index.html 내용이 변경되면 Terraform에서 변경사항 감지
}

# error.html 업로드
resource "aws_s3_object" "error" {
  bucket = aws_s3_bucket.website.id# STEP 1에서 생성한 기존 S3 Bucket
  key = "error.html"# 오류 페이지 Object 이름
  source = "${path.module}/error.html"# 현재 Terraform 프로젝트의 error.html 사용
  content_type = "text/html; charset=utf-8"# 브라우저가 HTML 문서로 인식
  etag = filemd5("${path.module}/error.html")# error.html 내용 변경 감지
}

[코드 설명]
```

* aws\_s3\_object: 기존 S3 Bucket에 파일을 Object로 업로드
* bucket:STEP 1에서 만든 동일한 S3 Bucket을 사용
* key: S3 내부에서 사용할 파일 이름
* source: 로컬 컴퓨터에서 업로드할 파일
* path.module: 현재 Terraform 프로젝트 디렉터리를 의미
* filemd5(): 파일 내용의 MD5 값을 계산한다.
  * HTML 파일 내용이 변경되면 Terraform이 변경사항을 감지할 수 있다.

```hcl
   # output.tf
output "index_object" {
  description = "업로드된 index.html Object"
  value = aws_s3_object.index.key
}

output "error_object" {
  description = "업로드된 error.html Object"
  value = aws_s3_object.error.key
}

PS C:\my-terraform> plan
PS C:\my-terraform> apply
```

* 다음 두 Object가 추가되는지 확인한다.
  * aws\_s3\_object.index
  * aws\_s3\_object.error
* index.html, error.html 파일이 업로드되어 있다.

![index.html, error.html 파일이 업로드되어 있다. 화면](<../.gitbook/assets/10 (2).png>)

## 74. 실습: STEP 5. S3 Static Website 접속 테스트

* Route 53을 연결하기 전에 S3 Static Website 자체가 정상 동작하는지 확인한다.

```hcl
   # output.tf
output "website_url" {
  description = "S3 Static Website 접속 URL"
  value = "http://${aws_s3_bucket_website_configuration.website.website_endpoint}"
}

   # 브라우저에서 접속
http://www.aws-esk.com.s3-website.ap-northeast-2.amazonaws.com/
```

![http://www.aws-esk.com.s3-website.ap-northeast-2.amazonaws.c 화면](<../.gitbook/assets/13 (1).png>)

* 존재하지 않는 페이지에 접속한다. http://www.aws-esk.com.s3-website.ap-northeast-2.amazonaws.com/test.html

![http://www.aws-esk.com.s3-website.ap-northeast-2.amazonaws.c 화면](<../.gitbook/assets/14 (1).png>)

* 메인 페이지로 이동을 클릭하게되면 다시 index.html로 이동한다.

## 75. 실습: STEP 6. Route 53 기존 Hosted Zone 조회

* Route 53에 이미 존재하는 "aws-esk.com" Public Hosted Zone을 Terraform에서 조회한다.
* 새로운 Hosted Zone을 생성하지 않는다.

```hcl
   # main.tf
# 기존 Route 53 Public Hosted Zone 조회
data "aws_route53_zone" "main" {
  name = var.domain_name# STEP 1에서 정의한 기본 도메인
  private_zone = false# Public Hosted Zone 조회
}

   코드 설명
```

* data "aws\_route53\_zone": Route 53에 이미 존재하는 Hosted Zone을 조회한다.
* resource가 아니라 data이므로 새로운 Hosted Zone을 만들지 않는다.
* name = var.domain\_name: aws-esk.com을 조회한다.
* private\_zone = false: Private Hosted Zone이 아니라 Public Hosted Zone을 조회한다.

```hcl
   # output.tf
output "hosted_zone_id" {
  description = "기존 Route 53 Hosted Zone ID"
  value = data.aws_route53_zone.main.zone_id
}

PS C:\my-terraform> plan
```

* 다음 Data Source가 정상 조회되는지 확인한다.
  * data.aws\_route53\_zone.main

## 76. 실습: STEP 7. Route 53 A Alias Record 생성

* Route 53에서 "www.aws-esk.com" A Record를 생성한다. 그리고 이 Record를 S3 Static Website 로 연결한다.

```hcl
   # main.tf
# www.aws-esk.com A Alias Record 생성
resource "aws_route53_record" "website" {
  zone_id = data.aws_route53_zone.main.zone_id# STEP 6에서 조회한 aws-esk.com Hosted Zone

  name = "${var.subdomain}.${var.domain_name}"# 최종 웹사이트 주소 : www.aws-esk.com
  type = "A"# IPv4 DNS Record

  # S3 Static Website로 Alias 연결
  alias {
    # STEP 2에서 생성된 S3 Website Domain
    name = aws_s3_bucket_website_configuration.website.website_domain
    zone_id = "Z3W03O7B5YMIYP"# 서울 리전 S3 Website Endpoint의 고정 Hosted Zone ID
    evaluate_target_health = false# S3 Website는 Target Health 평가를 사용하지 않음
  }
}
```

* aws\_route53\_record: Route 53 Hosted Zone에 DNS Record를 생성한다.
* zone\_id: STEP 6에서 조회한 aws-esk.com Hosted Zone ID
* name: "${var.subdomain}.${var.domain\_name}" (결과 : www.aws-esk.com)
* type = "A": IPv4 주소 계열 DNS Record
* alias: 실제 IP 주소를 직접 등록하는 대신 AWS Resource를 Target으로 지정한다.
* zone\_id: Z3W03O7B5YMIYP (ap-northeast-2 Seoul Region의 S3 Website Hosted Zone ID)
* evaluate\_target\_health = false : S3 Website Endpoint에 대해 Route 53 Target Health 평가를 사용하지 않는다.

```hcl
   # output.tf
output "route53_record" {
  description = "생성된 Route 53 Record"
  value = aws_route53_record.website.fqdn
}

# fqdn(Fully Qualified Domain Name)
# fqdn : Route53 Record의 전체 도메인을 의미한다.
output "final_website_url" {
  description = "Route 53을 이용한 최종 웹사이트 URL"
  value = "http://${aws_route53_record.website.fqdn}"
}

PS C:\my-terraform> terraform plan

PS C:\my-terraform> terraform apply
Outputs:

aws_route53_record = "www.aws-esk.com"
bucket_arn = "arn:aws:s3:::www.aws-esk.com"
bucket_name = "www.aws-esk.com"
bucket_policy_resource = "arn:aws:s3:::www.aws-esk.com"
domain_name = "www.aws-esk.com"
hosted_zone_id = "Z05574351P6F1W0OG1QHW"
website_endpoint = "www.aws-esk.com.s3-website.ap-northeast-2.amazonaws.com"
website_url = "http://www.aws-esk.com"
```

* www.aws-esk.com DNS Record가 정상적으로 조회되는지 확인한다.

또는 PowerShell: Resolve-DnsName www.aws-esk.com

![Resolve-DnsName www.aws-esk.com 화면](<../.gitbook/assets/15 (1).png>)

## 77. 실습: STEP 9. 최종 웹사이트 확인

\[정상 페이지] http://www.aws-esk.com

결과 : index.html 출력

\[오류 페이지] http://www.aws-esk.com/test.html

결과 : error.html 출력

```hcl
   # terraform.tfvars
# AWS Region
aws_region = "ap-northeast-2"

# AWS CLI Profile
aws_profile = "default"

# Route 53에 등록된 기본 도메인
domain_name = "aws-esk.com"

# S3 Static Website에서 사용할 서브도메인
subdomain = "web"# www  -->  web

# 프로젝트 이름
project_name = "s3-website"

# 환경 이름
environment = "training"

PS C:\my-terraform> terraform apply
Outputs:

aws_route53_record = "web.aws-esk.com"
bucket_arn = "arn:aws:s3:::web.aws-esk.com"
bucket_name = "web.aws-esk.com"
bucket_policy_resource = "arn:aws:s3:::web.aws-esk.com"
domain_name = "web.aws-esk.com"
hosted_zone_id = "Z05574351P6F1W0OG1QHW"
website_endpoint = "web.aws-esk.com.s3-website.ap-northeast-2.amazonaws.com"
website_url = "http://web.aws-esk.com"
```

* dns가 변경되면 S3 버킷까지 삭제 후 다시 생성되어야 하기 때문에 전체가 삭제되고 다시 만들어진다.
