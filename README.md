# System Wiki (Full)

> Rocky Linux 9 기반 시스템 관리, 쉘 스크립트, 데이터베이스, HTML, Docker, Kubernetes 학습 자료 전체 모음

부트캠프에서 다룬 System 전 과정을 항목별로 정리했습니다. 단순 이론 요약이 아니라 각 주제마다 **개념 개요 → 표준 설정/명령어 템플릿 → 실습 예제 → 검증·트러블슈팅**까지 실무에서 바로 찾아볼 수 있는 흐름으로 구성했고, 대부분의 대분류 끝에는 통합 정리·트러블슈팅 치트시트·퀵 레퍼런스 문서를 따로 두어 필요할 때 빠르게 훑어볼 수 있게 했습니다.

## 구성

- **Linux** (12개 소분류, 85개 문서) — Rocky Linux 9 기반 시스템 관리 전체: 기초 & 초기 구축, 명령어 정리, 파일/디렉터리 관리, 파일 내용 출력·검색, VI Editor, 사용자·그룹·권한, 허가권·소유권(SUID·SGID·Sticky bit 포함), 압축·아카이브, 파티션·마운트, 스토리지 관리(RAID·LVM), 파일공유(NFS·Samba), 네트워크 서비스(SSH·SCP·SFTP·vsFTP·DHCP·DNS)
- **Shell Script** (13개 문서) — 변수·환경변수부터 메타문자, 산술 연산, 조건문·반복문, 배열, 위치 매개변수, cron·anacron 스케줄 자동화까지
- **Database** (10개 문서) — MariaDB/MySQL 설치, SQL 문법(DDL·DML·DCL), SELECT·JOIN·GROUP BY 실습, 제약조건(PK·UNIQUE·FK)
- **HTML** (10개 문서) — 기본 구조, 텍스트·인라인 요소, 이미지·링크·입력 태그
- **Docker** (11개 문서) — 컨테이너·이미지 개념, 리소스 제한, 스토리지·네트워크, YAML 문법, Docker Compose
- **Kubernetes** (31개 문서) — 설치와 아키텍처부터 Pod 생명주기, Controller(ReplicaSet·Deployment·DaemonSet·StatefulSet·Job·CronJob), Service·Ingress, Label & Scheduling, Storage(PV·PVC), ConfigMap·Secret, AutoScaling까지 실무 흐름 순서로 정리

## 이렇게 활용하세요

왼쪽 목차(SUMMARY)를 따라 대분류 → 소분류 순서대로 읽으면 처음 학습하는 흐름 그대로 따라갈 수 있고, 이미 아는 주제라면 검색이나 목차에서 바로 원하는 문서로 이동해 명령어·설정 템플릿만 참고할 수도 있습니다. 각 대분류 끝의 **통합 정리** 문서는 전체 흐름 복습용, **트러블슈팅 치트시트**는 에러 발생 시 원인 진단용, **퀵 레퍼런스**는 명령어만 빠르게 찾을 때 사용하세요.
