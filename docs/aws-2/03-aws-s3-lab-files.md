# AWS S3 - 실습 파일

이론 4. AWS S3와 가이드 7에서 사용하는 AWS CLI 명령, 버킷 정책 JSON, 이미지 리사이즈 Lambda 코드를 모은 문서다.

## S3 CLI 실습과 IAM 정책

**개요**: `aws s3 ls/cp/sync`, User Data 배포, 사용자별 홈 디렉터리 IAM 정책.

`S3.txt`

```bash
[root@ip-172-31-47-32 ec2-user]# aws  s3  ls  s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/image/
2026-09-04 06:13:58          0
2026-09-04 06:17:05      26655 cat.jpg
2026-09-04 06:17:05      34359 dog.jpg


[root@ip-172-31-47-32 ec2-user]# aws  s3  ls  s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/file/
2026-09-04 06:14:05          0
2026-09-04 06:17:29       11184 aaa.txt
2026-09-04 06:17:29       1185 파이썬 자동화.txt


[root@ip-172-31-47-32 ec2-user]# aws  s3  ls  s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an --recursive
2026-09-04 06:14:05           0 file/
2026-09-04 06:17:29      11184 file/aaa.txt
2026-09-04 06:17:29       1185 file/파이썬 자동화.txt
2026-09-04 06:13:58           0 image/
2026-09-04 06:17:05      26655 image/cat.jpg
2026-09-04 06:17:05      34359 image/dog.jpg
2026-09-04 06:02:18      10221 index.html


	# S3  -->  EC2 파일 다운로드
[root@ip-172-31-47-32 ec2-user]# aws s3  cp  s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/index.html  .
download: s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/index.html to ./index.html

[root@ip-172-31-47-32 ec2-user]# ls  -l
total 12
-rw-r--r--. 1 root root 10221 Sep  4 06:02 index.html


	# S3  <--  EC2 파일 업로드
[root@ip-172-31-47-32 ec2-user]# touch newFile.txt


[root@ip-172-31-47-32 ec2-user]# aws  s3  cp  newFile.txt  s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/
upload: ./newFile.txt to s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/newFile.txt


	# S3  -->  EC2  동기화
[root@ip-172-31-47-32 ec2-user]# aws  s3  sync  s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an  .
download: s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/image/cat.jpg to image/cat.jpg
download: s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/file/파이썬 자동화.txt to file/파이썬 자동화.txt
download: s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/image/dog.jpg to image/dog.jpg
download: s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/file/aaa.txt to file/aaa.txt


[root@ip-172-31-47-32 ec2-user]# ls  -l
total 12
drwxr-xr-x. 2 root root    52 Sep  4 07:14 file
drwxr-xr-x. 2 root root    36 Sep  4 07:14 image
-rw-r--r--. 1 root root 10221 Sep  4 06:02 index.html
-rw-r--r--. 1 root root     0 Sep  4 07:09 newFile.txt


	


#!/bin/bash
sudo -s
dnf install httpd -y
service httpd start
chkconfig httpd on
aws s3 cp s3://my-s3-sol-bucket-123456789012-ap-northeast-2-an/index.html   /var/www/html  --region  ap-northeast-2


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListSpecificBucket",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-test-bucket-home-123456789012-ap-northeast-2-an",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "",
            "home/",
            "home/${aws:username}/*"
          ]
        }
      }
    },
    {
      "Sid": "FullAccessToUserHome",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-test-bucket-home-123456789012-ap-northeast-2-an/home/${aws:username}",
        "arn:aws:s3:::my-test-bucket-home-123456789012-ap-northeast-2-an/home/${aws:username}/*"
      ]
    }
  ]
}


{
    "Version": "2012-10-17",  // 정책 문법 버전
    "Statement": [

        {
            "Sid": "ListSpecificBucket",	// 정책 문장을 구분하기 위한 식별자

            "Effect": "Allow",		// 아래 조건에 해당하는 작업을 허용

            "Principal": "*",		// 모든 Principal을 대상으로 함

            "Action": "s3:ListBucket",	// 버킷 내부 객체 목록을 조회할 수 있는 권한

            "Resource": "arn:aws:s3:::{버킷명}",	// 객체가 아니라 버킷 자체에 대한 작업이므로 버킷 ARN 지정

            "Condition": {		// 권한을 허용할 추가 조건을 설정
                "StringLike": {		// 문자열 패턴을 비교하는 조건 연산자(*같은 와일드카드를 사용할 수 있음)
                    "s3:prefix": [		// s3:ListBucket 실행 시 조회할 객체 경로(prefix)를 제한하는 조건

                        "",			// 버킷의 최상위 경로 조회 허용
                        "home/",			// home/ 경로 조회 허용
                        "home/${aws:username}/*"	// 로그인한 IAM 사용자 이름과 동일한 경로만 접근 허용
                    ]
                }
            }
        },

        {
            "Sid": "FullAccessToUserHome",	// 정책 문장을 구분하기 위한 식별자
            "Effect": "Allow",			// 아래 S3 작업을 허용
            "Principal": "*",			// 모든 Principal을 대상으로 정책을 평가

            "Action": "s3:*",			// 지정된 Resource에 대해 모든 S3 작업 허용

            "Resource": [
                "arn:aws:s3:::{버킷명}/home/${aws:username}",	// 자신의 사용자 이름과 동일한 home 경로
                "arn:aws:s3:::{버킷명}/home/${aws:username}/*"	// 자신의 home 경로 아래에 존재하는 모든 객체
            ]
        }
    ]
}	
```

## 정적 웹 호스팅 버킷 정책

**개요**: 정적 웹 호스팅용 읽기 전용 버킷 정책(주석 설명 포함).

`static-hosting.txt`

```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::[버킷명]/*"
        }
    ]
}


{
    "Version": "2012-10-17",   		// 정책 문서의 버전. AWS에서 권장하는 고정 값 "2012-10-17" 사용
    "Statement": [             		// 실제 권한 부여 내용을 담는 배열
        {
            "Sid": "Statement",     	// Statement의 식별자(ID). 구분용으로 자유롭게 작성 가능
            "Effect": "Allow",          	// 허용(Allow)인지 거부(Deny)인지 지정
            "Principal": "*",          	// 누가 접근할 수 있는지 지정. "*"는 모든 사용자(공개 접근)를 의미
            "Action": "s3:GetObject",  	// 허용할 작업. 여기서는 객체 다운로드(읽기) 권한만 허용
            "Resource": "arn:aws:s3:::test-static-hosting-123456789012/*"  
            // 권한이 적용되는 대상 자원(Resource)
            // [버킷명]에 있는 모든 객체(*)에 대해 적용됨
        }
    ]
}


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-static-hosting-123456789012-ap-northeast-2-an/*"
        }
    ]
}
```

## 이미지 리사이즈 Lambda

**개요**: S3 업로드 이벤트로 이미지를 리사이즈하는 Lambda 함수.

`index.js`

```javascript
const { S3Client, GetObjectCommand, PutObjectCommand } = require('@aws-sdk/client-s3');
const util = require('util');
const sharp = require('sharp');

// Initialize S3 client
const s3Client = new S3Client();
//람다 사용 전 dependency 인스톨은 npm install --platform=linux --arch=x64 sharp 로 할 것
exports.handler = async (event, context, callback) => {
    // Get the S3 event
    const srcBucket = event.Records[0].s3.bucket.name;
    const srcKey = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " "));
    const dstKey = srcKey;

    // Determine the image type
    const typeMatch = srcKey.match(/\.([^.]*)$/);
    if (!typeMatch) {
        console.log("Could not determine the image type.");
        return;
    }

    // Check if the image type is supported
    const imageType = typeMatch[1].toLowerCase();
    if (imageType != "jpg" && imageType != "png") {
        console.log(`Unsupported image type: ${imageType}`);
        return;
    }

    // Get the image from S3 bucket
    try {
        const params = {
            Bucket: srcBucket,
            Key: srcKey
        };
        const origImageResponse = await s3Client.send(new GetObjectCommand(params));
        const origImage = await streamToBuffer(origImageResponse.Body);


        // Create a thumbnail
        const width = parseInt(process.env.width) || 200;
        const height = parseInt(process.env.height) || 200;
        console.log("width:", width, "height:", height);


        var buffer = await sharp(origImage).resize(width, height).toBuffer();


        const destParams = {
            Bucket: srcBucket,
            Key: "resized/" + dstKey,
            Body: buffer,
            ContentType: "image"
        };

        await s3Client.send(new PutObjectCommand(destParams));
    } catch (error) {
        console.log(error);
        return;
    }

    console.log(`${srcBucket}/${srcKey} 이미지를 리사이징 하여 ${srcBucket}/resized/${dstKey}에 업로드 하였습니다`);
};

// Helper function to convert stream to buffer
const streamToBuffer = async (stream) => {
    return new Promise((resolve, reject) => {
        const chunks = [];
        stream.on('data', (chunk) => chunks.push(chunk));
        stream.on('end', () => resolve(Buffer.concat(chunks)));
        stream.on('error', reject);
    });
};
```
