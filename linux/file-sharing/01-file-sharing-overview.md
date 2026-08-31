# 파일 공유 개요 (NFS vs Samba/SMB)

> **Tag:** #Linux #NFS #Samba #SMB #CIFS #FileSharing #Storage  
> **핵심 요약:** 네트워크 파일 공유의 대표 기술은 NFS와 Samba(SMB)이다. NFS는 Linux/Unix 계열끼리 파일시스템을 공유하는 데 최적화되어 있고, Samba는 Microsoft의 SMB 프로토콜을 Linux에서 사용할 수 있도록 구현한 소프트웨어 모음이다. Samba는 Linux↔Windows뿐 아니라 Linux↔Linux, Windows↔Windows 환경에서도 동작하지만, Linux 환경만 존재한다면 일반적으로 NFS를 더 많이 사용한다. 두 기술 모두 "디스크 공간(스토리지)"을 공유하는 것이지 메모리를 공유하는 것이 아니다.

---

## 1. 개요 (Overview)

NFS(Network File System)는 주로 Linux/Unix 시스템 사이에서 파일과 디렉터리를 공유하기 위해 사용하는 네트워크 파일 공유 기술이다. TCP/IP 네트워크를 통해 원격 서버의 디렉터리를 로컬 디렉터리처럼 마운트해서 사용하며, 다음과 같은 조합에서 쓰인다.

```text
NFS 사용 조합
Linux  <--->  Linux
Unix   <--->  Linux
```

Samba는 Linux/Unix 계열 운영체제에서 Microsoft의 SMB 프로토콜을 사용할 수 있도록 구현한 소프트웨어 모음이다. Samba를 사용하면 Linux와 Windows가 네트워크를 통해 파일, 디렉터리, 프린터 등의 자원을 공유할 수 있다. Samba는 Windows 전용 프로토콜을 Linux에서 사용할 수 있도록 구현한 서비스이므로, 단순히 Linux와 Windows 사이에서만 사용하는 것은 아니다. Linux 시스템끼리도 Samba로 파일을 공유할 수 있지만, Linux 환경에서는 일반적으로 NFS를 더 많이 사용한다.

```text
Samba/SMB 사용 조합
Linux    <--->  Windows
Linux    <--->  Linux
Windows  <--->  Windows
```

SMB와 CIFS 용어를 정리하면 다음과 같다.

| 용어 | 설명 |
|---|---|
| SMB(Server Message Block) | Windows의 기본 파일 공유 프로토콜 |
| CIFS(Common Internet File System) | SMB1 기반 확장 명칭. 현재는 SMB2/SMB3가 주로 사용 |
| SMB2/SMB3 | 성능·보안이 개선된 현행 버전(암호화, 서명 지원) |

CIFS 이후에는 라우터를 넘어 다양한 네트워크에서도 파일 공유가 가능해졌다. 다만 Linux의 마운트 타입 이름은 여전히 `cifs`를 사용하며, 실제 협상 버전은 SMB3(`vers=3.1.1`)인 경우가 많다. SMB1(CIFS)은 보안 취약점 때문에 최신 Windows에서 기본 비활성화되어 있다. `smbclient -L` 실행 시 `SMB1 disabled -- no workgroup available` 메시지가 보이는 것은 정상이다.

NFS와 Samba를 비교하면 다음과 같다.

| 구분 | NFS | Samba(SMB) |
|---|---|---|
| 주 사용 환경 | Linux·Unix 계열 | Windows 혼재 환경 |
| 기본 포트 | TCP 2049 (+ RPC 111) | TCP 445, 139 / UDP 137, 138 |
| 인증 방식 | 기본은 IP·UID 기반(sec=sys) | 사용자 계정·비밀번호 기반 |
| 권한 처리 | Linux 퍼미션·UID/GID 그대로 | SMB 계정 매핑 + 마스크 옵션 |
| 클라이언트 도구 | `nfs-utils` | `samba-client`, `cifs-utils` |
| 마운트 타입 | `-t nfs` | `-t cifs` |

NFS는 UID/GID가 서버·클라이언트 간 일치해야 권한이 자연스럽게 동작하고, Samba는 계정 인증 후 서버가 정한 마스크로 권한이 결정된다는 점이 큰 차이다.

실습 문서에서 흔히 "메모리를 할당받는다"라고 표현하지만, 실제로는 서버의 디스크 공간(스토리지)을 네트워크로 빌려 쓰는 것이다. 실제 데이터는 항상 서버의 HDD/SSD에 저장되며 클라이언트는 원격 접근만 한다.

```text
클라이언트가 쓰는 것 = 서버의 디스크 공간(스토리지)
클라이언트가 쓰는 것 ≠ 서버의 메모리(RAM)
```

---

## 2. 표준 확인 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 설치 여부 확인

```bash
rpm -qa | grep samba                       # Samba 관련 패키지 확인
rpm -qa | grep nfs                         # NFS 관련 패키지 확인
rpm -qa | grep rpcbind                     # rpcbind 확인
rpm -qa | grep cifs-utils                  # cifs 마운트 도구 확인
```

### Step 2. 필요 패키지 설치

```bash
dnf install -y samba samba-client samba-common cifs-utils   # Samba 서버·클라이언트
dnf install -y nfs-utils                                    # NFS 서버·클라이언트
```

`mount -t cifs`를 사용하려면 `cifs-utils`(mount.cifs)가 반드시 필요하다.

---

## 3. 선택 기준 (Verification)

```text
Linux 전용 환경, 대량 I/O, UID 통일 가능   → NFS
Windows 클라이언트 존재, 계정 인증 필요     → Samba(SMB)
공유 프린터까지 필요                        → Samba
방화벽 단순화(포트 최소)                    → NFSv4 (2049 단일)
```

>  **핵심 요약**
> - NFS는 Linux/Unix 중심, Samba는 SMB 프로토콜 구현체
> - Samba는 Linux↔Windows 외 조합에서도 사용 가능
> - Linux 전용 환경에서는 일반적으로 NFS 선호
> - SMB1(CIFS)은 비활성화가 기본, 현행은 SMB2/SMB3
> - 공유되는 자원은 메모리가 아니라 디스크 공간
> - 관련:  Windows SMB 서버 & Linux CIFS 클라이언트 ·  Linux Samba 서버 구축 ·  NFS 개념 & RPC 동작 원리

---
