# 3-Tier Architecture 실습 - 강의 노트

AWS 강의 노트(HWP) 원본을 이미지와 설정 코드, 주석까지 그대로 옮긴 문서다. 정리본은 이론·가이드 문서를 함께 본다.

## WordPress 서버 클러스터 구성

## 3-Tier Architecture (3계층 아키텍처)

- 애플리케이션을 3가지 티어로 나누어 구성하는 방식이다. 이렇게 하면 역할이 명확해지고, 확장/유지보수가 쉬워진다.

- Presentation Tier (프리젠테이션 계층)
  - 사용자가 직접 보는 화면(UI) 부분
  - HTML, CSS, JavaScript 같은 기술로 구성
  - 웹 브라우저에서 동작하며, 사용자의 입력을 받아 Application Tier로 전달

- Application Tier (애플리케이션 계층)
  - 실제 로직(비즈니스 로직)을 처리하는 층
  - 예: 로그인 검증, 주문 처리, 게시글 등록 등
  - PHP, Java, Python 같은 프로그래밍 언어와 서버(EC2 같은 곳)에서 동작

- Data Tier (데이터 계층)
  - 데이터를 저장하고 관리하는 층
  - RDS(MySQL, PostgreSQL 등) 같은 데이터베이스 서비스가 위치
  - Application Tier가 요청한 데이터를 DB에서 읽거나 저장해

- AWS 환경에서의 3-Tier 예시
  - Presentation Tier: ALB(Application Load Balancer)
  - Application Tier: Web EC2 인스턴스
  - Data Tier: Amazon RDS (데이터베이스)

- 즉, ALB는 사용자의 요청을 받아 EC2에 전달하고, EC2는 로직을 처리하면서 RDS에서 데이터를 주고받는 구조
- 정리하면, 3-Tier는 UI(화면), 로직 처리(서버), 데이터(DB)를 분리해놓은 구조이고,
AWS에서는 ALB-EC2-RDS 조합으로 구현하는 경우가 많다.

## 실습

![이미지](assets/09-3tier-practice/1.png)

- 3-Tier Application 구조 기반 WordPress 웹 서버 클러스터
  - Presentation Tier: ALB(Application Load Balancer)가 사용자 요청을 받아 분산 처리
  - Application Tier: EC2 인스턴스(웹 서버)가 Auto Scaling Group으로 묶여 트래픽 변화에 따라 자동 증감
  - Data Tier: Amazon RDS 클러스터가 데이터베이스 역할 수행

- 고가용성 확보
  - ALB를 통해 다수의 EC2 인스턴스에 트래픽을 분산
  - Auto Scaling을 통해 부하 시 자동으로 서버를 늘리고, 부하가 줄면 줄임
  - Multi-AZ 배치를 통해 Availability Zone 장애에도 서비스 지속 가능

- EFS(Elastic File System) 공유 스토리지
  - 여러 EC2 인스턴스가 동일한 워드프레스 파일을 접근할 수 있도록 중앙 스토리지 역할
  - 웹 서버 간 데이터 일관성을 유지
  - 워드프레스 업로드 파일(이미지, 플러그인 등)을 EC2 간 공유 가능

## 워드프레스 서버 클러스터 실습 정리

- 구조
  - ALB : 사용자 요청을 받아서 여러 서버로 로드분산
  - EC2: 웹 서버 역할, 워드프레스 실행, Auto Scaling으로 자동 증감
  - RDS: 데이터베이스 저장
  - EFS: 여러 서버가 공유하는 저장소

- 구성 순서
  - VPC 만들기 (Public Subnet, Private Subnet 나누기)
  - RDS 만들고 DB 준비
  - EFS 만들고 소스코드 저장 공간으로 사용
  - EC2에 Apache, PHP 설치 후 워드프레스 설치
  - S3 버킷 생성 후 wp-config 업데이트 및 업로드
  - EC2를 이미지(AMI)로 만들어서 재사용
  - Auto Scaling Group과 ALB 연결해서 트래픽 자동 분산

- 특징
  - 서버가 많아도 EFS 덕분에 똑같은 소스코드를 공유
  - 사용자가 늘어나면 서버가 자동으로 늘어나고 줄어든다.
  - RDS와 다중 AZ 배치로 장애에도 서비스가 계속 유지된다.

- 핵심 포인트

```
3 Tier 구조 : ALB(분산) + EC2(웹 서버) + RDS(DB)
 # EFS 사용 : 모든 웹 서버가 같은 파일 사용
 # Auto Scaling : 자동으로 서버 수 조절
 # 고가용성 : 장애에도 서비스 멈추지 않음
```

## 워드프레스(WordPress)

- 우리가 인터넷에서 보는 대부분의 홈페이지는 단순한 파일이 아니다.
특히 블로그, 쇼핑몰, 회사 홈페이지처럼 글이 계속 바뀌고, 회원이 있고,
댓글이 달리는 사이트는 프로그램 형태로 만들어진다.

- WordPress는 이런 사이트를 쉽게 만들 수 있도록 만들어진 대표적인 웹 프로그램

- 원래 이런 사이트를 직접 만들려면 HTML, CSS, JavaScript, PHP, 데이터베이스까지 모두 알아야 한다.
하지만 WordPress를 사용하면 이런 복잡한 기술을 몰라도, 화면에서 버튼을 눌러 글을 쓰고
디자인을 바꾸는 방식으로 사이트를 운영할 수 있다.
그래서 WordPress는 홈페이지 제작 도구이자 콘텐츠 관리 시스템(CMS)이라고 불린다.

```
# wp-config.php
```

- WordPress는 실행될 때마다 데이터베이스에 접속해야 한다.
그런데 프로그램은 DB의 위치와 비밀번호등을 알 수 없다.
그래서 해당 정보를 미리 적어둔 파일이 필요하다.

- 그 파일이 바로 wp-config.php이다.

- wp-config.php는 WordPress에게  데이터베이스에, 이 아이디와 비밀번호로 접속해서 데이터를 가져와라.
로그인은 이런 방식으로 암호화해라,  에러는 화면에 보여줄지 말지 이렇게 처리해라.
즉, WordPress의 기본 동작 방식을 미리 정해두는 설정서 역할을 한다.

- 정리
  - 워드프레스: 웹사이트를 손쉽게 만드는 프로그램
  - wp-config.php: 워드프레스가 데이터베이스와 연결되고 제대로 동작할 수 있게 해주는 설정 파일
