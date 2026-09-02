# Table of contents

* [소개](README.md)

## Linux

### 기초 & 환경 구축

* [Linux 기초 & Rocky Linux 초기 구축 (VMware 기반)](linux/basics-setup/01-linux-rocky-initial-setup.md)

### 명령어 정리

* [Rocky Linux 9 및 Bash 실무 명령어 가이드](linux/commands/01-rocky-linux-bash-commands-guide.md)

### 파일 & 디렉터리 관리

* [경로 이동 & 목록 조회 (cd & ls & pwd)](linux/file-directory/01-navigation-cd-ls-pwd.md)
* [디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm)](linux/file-directory/02-create-delete-mkdir-rmdir-rm.md)
* [복사·이동·와일드카드 (cp · mv · glob)](linux/file-directory/03-copy-move-wildcard.md)
* [파일시스템 기본 명령어 통합 정리 — 경로·조회·생성·복사 한눈에](linux/file-directory/04-filesystem-commands-summary.md)
* [파일시스템 기본 명령어 트러블슈팅 치트시트](linux/file-directory/05-filesystem-troubleshooting-cheatsheet.md)
* [파일시스템 기본 명령어 퀵 레퍼런스](linux/file-directory/06-quick-reference.md)
* [디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm)](linux/file-directory/07-lab-project-directory-standardization.md)

### 파일 내용 출력 & 검색

* [파일 내용 출력 6종 (cat · head · tail · more · less · nl)](linux/file-search/01-view-content-cat-head-tail-more-less-nl.md)
* [cat 리다이렉션 & Heredoc](linux/file-search/02-cat-redirection-heredoc.md)
* [파일·디렉터리 검색 (find)](linux/file-search/03-find-search.md)
* [파일 조회·처리·검색 통합 정리](linux/file-search/04-summary.md)
* [파일 조회·처리·검색 트러블슈팅 치트시트](linux/file-search/05-troubleshooting-cheatsheet.md)
* [파일 조회·처리·검색 명령어 퀵 레퍼런스](linux/file-search/06-quick-reference.md)

### VI Editor

* [VI 3-Mode 아키텍처 & 파일 열기](linux/vi-editor/01-vi-3-mode-architecture-file-open.md)
* [VI 편집 명령어 (커서·삭제·복사·Ex Mode 범위 조작)](linux/vi-editor/02-vi-editing-commands-cursor-delete-copy-ex-mode.md)
* [VI 치환](linux/vi-editor/03-vi-substitution.md)
* [VI Shell 연동 & Swap 파일 복구](linux/vi-editor/04-vi-shell-integration-swap-recovery.md)
* [VI 통합 정리](linux/vi-editor/05-integration-summary.md)
* [VI 트러블슈팅 치트시트](linux/vi-editor/06-troubleshooting-cheatsheet.md)
* [VI 명령어 퀵 레퍼런스](linux/vi-editor/07-quick-reference.md)

### 사용자 & 그룹 & 권한

* [리눅스 사용자 계정 관리 (useradd / usermod / userdel / passwd / su)](linux/users-groups/01-user-account-management-useradd-usermod-userdel.md)
* [리눅스 그룹 관리 & UPG 모델 (groupadd & usermod & gpasswd)](linux/users-groups/02-group-management-upg-model-groupadd-gpasswd.md)
* [Root 접속 통제 & Sudo 권한 위임 (SSH Hardening / wheel / sudoers)](linux/users-groups/03-root-access-control-sudo-delegation.md)
* [사용자·그룹·권한 통합 정리 — 계정 라이프사이클 한눈에](linux/users-groups/04-users-groups-permissions-summary.md)
* [사용자·그룹·권한 트러블슈팅 치트시트](linux/users-groups/05-users-groups-troubleshooting-cheatsheet.md)
* [사용자·그룹·권한 명령어 퀵 레퍼런스](linux/users-groups/06-users-groups-quick-reference.md)
* [종합실습 부서별 계정·홈디렉터리·Skel 표준화 시나리오](linux/users-groups/07-department-account-home-skel-standardization.md)

### 허가권 & 소유권

* [Chmod 계산기](linux/permissions-ownership/00-chmod-calculator.md)
* [허가권 (Permission) — chmod & rwx·UGO 모델](linux/permissions-ownership/01-permissions-chmod.md)
* [허가권 상세 (chmod & 8진수 · 심볼릭 표기)](linux/permissions-ownership/02-chmod-details.md)
* [소유권 (Ownership) — chown & UID·GID 소유 모델](linux/permissions-ownership/03-ownership-chown.md)
* [소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky)](linux/permissions-ownership/04-ownership-special-permissions.md)
* [Umask — 기본 권한 마스크 (User Mask)](linux/permissions-ownership/05-umask.md)
* [특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX)](linux/permissions-ownership/06-special-permission-setgid.md)
* [특수권한 Sticky-bit — 공유 디렉터리 삭제 방지 (1XXX)](linux/permissions-ownership/07-special-permission-sticky-bit.md)
* [특수권한 Set-UID — 실행 중 소유자 권한 위임 (4XXX)](linux/permissions-ownership/08-special-permission-setuid.md)
* [허가권·소유권 통합 정리](linux/permissions-ownership/09-summary.md)
* [허가권·소유권 트러블슈팅 치트시트](linux/permissions-ownership/10-troubleshooting-cheatsheet.md)
* [허가권·소유권 명령어 퀵 레퍼런스](linux/permissions-ownership/11-quick-reference.md)

### 압축 & 아카이브

* [파일 압축 & 아카이브 — gzip · bzip2 · xz · tar](linux/compression-archive/01-compression-tools-gzip-bzip2-xz-tar.md)
* [파일 압축·아카이브 통합 정리](linux/compression-archive/02-summary.md)
* [파일 압축·아카이브 트러블슈팅 치트시트](linux/compression-archive/03-troubleshooting-cheatsheet.md)
* [파일 압축·아카이브 명령어 퀵 레퍼런스](linux/compression-archive/04-quick-reference.md)

### 파티션 & 마운트

* [디스크 타입 & 파티션 구조](linux/partition-mount/01-disk-types-partition-structure.md)
* [파일 시스템 & Format](linux/partition-mount/02-filesystem-format.md)
* [마운트 & umount](linux/partition-mount/03-mount-umount.md)
* [Automount](linux/partition-mount/04-automount.md)
* [파티션·마운트 통합 정리](linux/partition-mount/05-summary.md)
* [파티션·마운트 트러블슈팅 치트시트](linux/partition-mount/06-troubleshooting-cheatsheet.md)
* [Partition & Mount 명령어 퀵 레퍼런스](linux/partition-mount/07-quick-reference.md)
* [종합실습 HDD 추가 파티션 포맷 마운트 Automount](linux/partition-mount/08-lab-hdd-partition-format-mount-automount.md)

### 스토리지 관리 (RAID·LVM)

* [RAID 개념 & Hardware vs Software RAID](linux/storage-raid-lvm/01-raid-concept-hardware-vs-software.md)
* [RAID 레벨별 특징 (Linear·0·1·5·6)](linux/storage-raid-lvm/02-raid-levels.md)
* [mdadm 명령어 & RAID 관리](linux/storage-raid-lvm/03-mdadm-commands.md)
* [종합실습 RAID 5 + Spare 구성 & 장애 복구](linux/storage-raid-lvm/04-lab-raid5-spare-recovery.md)
* [LVM 개념 & 구조 (PV·VG·LV·PE)](linux/storage-raid-lvm/05-lvm-concept.md)
* [LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate)](linux/storage-raid-lvm/06-lvm-configuration.md)
* [종합실습 LVM 구성 확장 축소](linux/storage-raid-lvm/07-lab-lvm-expand-shrink.md)
* [RAID·LVM 통합 정리 — 스토리지 관리 한눈에](linux/storage-raid-lvm/08-raid-lvm-summary.md)
* [RAID·LVM 트러블슈팅 치트시트](linux/storage-raid-lvm/09-raid-lvm-troubleshooting.md)
* [RAID·LVM 명령어 퀵 레퍼런스](linux/storage-raid-lvm/10-raid-lvm-quick-reference.md)

### 파일공유 (NFS·Samba)

* [파일 공유 개요 (NFS vs Samba/SMB)](linux/file-sharing/01-file-sharing-overview.md)
* [Windows SMB 서버 & Linux CIFS 클라이언트](linux/file-sharing/02-windows-smb-linux-cifs.md)
* [Linux Samba 서버 구축 (smbd·nmbd·smb.conf)](linux/file-sharing/03-samba-server-setup.md)
* [종합실습 Samba 공유 & 권한 제어(Sticky Bit)](linux/file-sharing/04-lab-samba-sticky-bit.md)
* [NFS 개념 & RPC 동작 원리](linux/file-sharing/05-nfs-concept-rpc.md)
* [NFS 서버 구성 (/etc/exports·exportfs·방화벽)](linux/file-sharing/06-nfs-server-configuration.md)
* [NFS 클라이언트 마운트 & fstab 영구화](linux/file-sharing/07-nfs-client-mount-fstab.md)
* [종합실습 다중 클라이언트 NFS 구성](linux/file-sharing/08-lab-nfs-multi-client.md)
* [종합정리 Samba & NFS](linux/file-sharing/09-samba-nfs-summary.md)
* [Samba·NFS 퀵 레퍼런스](linux/file-sharing/10-samba-nfs-quick-reference.md)
* [Samba·NFS 트러블슈팅 치트시트](linux/file-sharing/11-samba-nfs-troubleshooting.md)

### 네트워크 서비스

* [SSH 개념 & 프로세스·보안 설정 (Telnet·Daemon·sshd_config)](linux/network-services/01-ssh-concept-security.md)
* [SCP 파일 전송 (Linux·Windows)](linux/network-services/02-scp-file-transfer.md)
* [vsFTP 설치 & 접근 제어 (user_list·chroot)](linux/network-services/03-vsftp-setup-access-control.md)
* [SFTP 파일 전송](linux/network-services/04-sftp-file-transfer.md)
* [DHCP 개념 & 서버 구성](linux/network-services/05-dhcp-concept-server.md)
* [DNS 개념 & Master Name Server·Zone 이론](linux/network-services/06-dns-concept-zone-theory.md)
* [종합실습 DNS Master + Web + FTP 통합 구성](linux/network-services/07-lab-dns-web-ftp-integration.md)
* [Zone 파일 레코드 옵션 (TTL·SOA·NS·serial·A·AAAA)](linux/network-services/08-zone-file-records.md)
* [종합정리 네트워크 서비스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS)](linux/network-services/09-network-services-summary.md)
* [트러블슈팅 치트시트 (SSH·vsFTP·SFTP·SCP·DHCP·DNS)](linux/network-services/10-troubleshooting-cheatsheet.md)
* [퀵 레퍼런스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS)](linux/network-services/11-quick-reference.md)


## Shell Script

* [Shell Script 종합 연습문제](shell-script/00-shell-script-exercises.md)
* [Shell Script - 변수와 환경변수 (커널·쉘·쉘스크립트 개념 포함)](shell-script/01-variables-and-environment-variables.md)
* [Shell Script - Metacharacters (메타문자)](shell-script/02-metacharacters.md)
* [Shell Script - expr · let (산술 연산)](shell-script/03-expr-let-arithmetic.md)
* [Shell Script - exit 상태와 test 명령](shell-script/04-exit-status-and-test.md)
* [Shell Script - 조건문 (if · case)](shell-script/05-conditionals-if-case.md)
* [Shell Script - 반복문 (for · while · until)](shell-script/06-loops-for-while-until.md)
* [Shell Script - 배열(Array)과 RANDOM](shell-script/07-arrays-and-random.md)
* [Shell Script - 위치 매개변수 (Positional Parameters)](shell-script/08-positional-parameters.md)
* [Shell Script - cron · anacron (스케줄 자동화)](shell-script/09-cron-anacron-scheduling.md)
* [Shell Script - 통합 정리 (변수부터 cron · anacron까지 한눈에)](shell-script/10-integration-summary.md)
* [Shell Script - 트러블슈팅 치트시트](shell-script/11-troubleshooting-cheatsheet.md)
* [Shell Script - 명령어 퀵 레퍼런스](shell-script/12-quick-reference.md)


## Database

* [DB - 데이터와 데이터베이스 기초 (MariaDB 설치 포함)](database/01-data-and-database-basics.md)
* [DB - SQL 문법 (DDL · DML · DCL)](database/02-sql-syntax-ddl-dml-dcl.md)
* [DB - SELECT · WHERE · ORDER BY · LIKE 실습 (emp · dept · member · product_catalog)](database/03-select-where-orderby-like.md)
* [emp · dept 테이블 정의 및 샘플 데이터](database/04-emp-dept-table-definitions.md)
* [DB - 제약조건 (PK · UNIQUE · FK)](database/05-constraints-pk-unique-fk.md)
* [DB - INNER JOIN 실습 (customer · orders)](database/06-inner-join-practice.md)
* [DB - GROUP BY · HAVING · 집계함수](database/07-groupby-having-aggregate-functions.md)
* [DB - 통합 정리 (데이터 기초부터 INNER JOIN·GROUP BY·HAVING까지 한눈에)](database/08-integration-summary.md)
* [DB - 트러블슈팅 치트시트](database/09-troubleshooting-cheatsheet.md)
* [DB - SQL 퀵 레퍼런스](database/10-quick-reference.md)


## HTML

* [HTML - HTML 기초와 기본구조](html/01-html-basics-and-structure.md)
* [HTML - 텍스트 표시 방법](html/02-html-text-display.md)
* [HTML - 태그의 구분 · 인라인 텍스트 요소](html/03-html-tag-types-inline-text.md)
* [HTML - 이미지 태그](html/04-html-image-tag.md)
* [HTML - 컨테이너 태그](html/05-html-container-tag.md)
* [HTML - 링크](html/06-html-link.md)
* [HTML - 입력태그](html/07-html-input-tag.md)
* [HTML - 통합 정리](html/08-html-integration-summary.md)
* [HTML - 트러블슈팅 치트시트](html/09-html-troubleshooting-cheatsheet.md)
* [HTML - 퀵 레퍼런스](html/10-html-quick-reference.md)


## Docker

* [Docker - 도커와 컨테이너의 이해](docker/01-docker-container-concepts.md)
* [Docker - 컨테이너와 이미지](docker/02-docker-container-and-image.md)
* [Docker - 컨테이너 사용하기](docker/03-docker-using-containers.md)
* [Docker - 컨테이너 리소스 제한](docker/04-docker-resource-limits.md)
* [Docker - 스토리지](docker/05-docker-storage.md)
* [Docker - 컨테이너 네트워크](docker/06-docker-container-network.md)
* [Docker - YAML 문법](docker/07-docker-yaml-syntax.md)
* [Docker - Docker Compose](docker/08-docker-compose.md)
* [Docker - 통합 정리](docker/09-docker-integration-summary.md)
* [Docker - 트러블슈팅 치트시트](docker/10-docker-troubleshooting-cheatsheet.md)
* [Docker - 퀵 레퍼런스](docker/11-docker-quick-reference.md)


## Kubernetes

* [Kubernetes - 입문 기초이론](kubernetes/00-kubernetes-intro-basics.md)
* [Kubernetes - 설치](kubernetes/01-kubernetes-installation.md)
* [Kubernetes - Pod 생성](kubernetes/02-kubernetes-pod-creation.md)
* [Kubernetes - 아키텍처 개요와 핵심 컴포넌트](kubernetes/03-kubernetes-architecture-overview.md)
* [Kubernetes - Namespace](kubernetes/04-kubernetes-namespace.md)
* [Kubernetes - ResourceQuota·LimitRange](kubernetes/05-kubernetes-resource-limits.md)
* [Kubernetes - Pod 구조와 생성·동작 흐름](kubernetes/06-kubernetes-pod-lifecycle.md)
* [Kubernetes - livenessProbe](kubernetes/07-kubernetes-liveness-probe.md)
* [Kubernetes - readinessProbe](kubernetes/08-kubernetes-readiness-probe.md)
* [Kubernetes - Init Container·Static Pod](kubernetes/09-kubernetes-init-static-pod.md)
* [Kubernetes - Controller 개념과 ReplicationController](kubernetes/10-kubernetes-controller-concept.md)
* [Kubernetes - ReplicaSet](kubernetes/11-kubernetes-replicaset.md)
* [Kubernetes - Deployment](kubernetes/12-kubernetes-deployment.md)
* [Kubernetes - Rollout·Rollback 실습](kubernetes/13-kubernetes-rollout-rollback.md)
* [Kubernetes - DaemonSet](kubernetes/14-kubernetes-daemonset.md)
* [Kubernetes - StatefulSet](kubernetes/15-kubernetes-statefulset.md)
* [Kubernetes - Job](kubernetes/16-kubernetes-job.md)
* [Kubernetes - CronJob](kubernetes/17-kubernetes-cronjob.md)
* [Kubernetes - Service 기초와 ClusterIP](kubernetes/18-kubernetes-service-basics.md)
* [Kubernetes - NodePort·LoadBalancer](kubernetes/19-kubernetes-service-nodeport-loadbalancer.md)
* [Kubernetes - ExternalName·Headless Service](kubernetes/20-kubernetes-service-externalname-headless.md)
* [Kubernetes - Ingress 기초와 준비](kubernetes/21-kubernetes-ingress-basics.md)
* [Kubernetes - host·path 기반 Ingress](kubernetes/22-kubernetes-ingress-routing.md)
* [Kubernetes - 정규표현식 Ingress·Canary 배포](kubernetes/23-kubernetes-ingress-regex-canary.md)
* [Kubernetes - Label](kubernetes/24-kubernetes-label.md)
* [Kubernetes - Pod Scheduling (nodeSelector·Affinity)](kubernetes/25-kubernetes-scheduling-affinity.md)
* [Kubernetes - Pod Scheduling (Taint·Toleration)](kubernetes/26-kubernetes-scheduling-taint-toleration.md)
* [Kubernetes - Storage 개요와 emptyDir·hostPath](kubernetes/27-kubernetes-storage-volumes.md)
* [Kubernetes - PV·PVC와 StorageClass·Dynamic Provisioning](kubernetes/28-kubernetes-storage-pv-pvc.md)
* [Kubernetes - ConfigMap](kubernetes/29-kubernetes-configmap.md)
* [Kubernetes - Secret](kubernetes/30-kubernetes-secret.md)
* [Kubernetes - AutoScaling](kubernetes/31-kubernetes-autoscaling.md)
* [Kubernetes - 통합 정리](kubernetes/32-kubernetes-integration-summary.md)
* [Kubernetes - 트러블슈팅 치트시트](kubernetes/33-kubernetes-troubleshooting-cheatsheet.md)
* [Kubernetes - 퀵 레퍼런스](kubernetes/34-kubernetes-quick-reference.md)

## AWS

### AWS 이론

* [AWS - 클라우드 기초 개념](aws-이론/01-aws-cloud-fundamentals.md)
* [AWS EC2 - 배포](aws-이론/02-aws-ec2-deployment.md)
* [AWS VPC](aws-이론/03-aws-vpc.md)

### AWS 가이드

* [AWS 프리 티어 가입 방법](aws-가이드/01-aws-signup.md)
* [AWS IAM - MFA 강제 정책 적용하기](aws-가이드/02-aws-iam-force-mfa-policy.md)
* [AWS EC2 설정](aws-가이드/03-aws-ec2-setup.md)
* [EC2 인스턴스 접속하기](aws-가이드/04-ec2-instance-connect.md)
* [AWS 탄력적 IP(Elastic IP) 적용하기](aws-가이드/05-aws-elastic-ip.md)
* [AWS ALB + Auto Scaling + 대상 그룹 통합 가이드](aws-가이드/06-aws-elb-https-setup.md)
