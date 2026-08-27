# 🏗️ 종합실습 Samba 공유 & 권한 제어(Sticky Bit)

> **Tag:** #Linux #Lab #Samba #StickyBit #Permission #ForceGroup #smbconf  
> **핵심 요약:** Samba 공유 디렉터리를 777로 열어 두면 Windows 사용자가 다른 사람이 만든 파일까지 삭제할 수 있다. 이때 Sticky Bit(`chmod 1777`)를 적용하면 파일의 소유자와 디렉터리 소유자, root만 삭제·이름 변경이 가능하고 다른 사용자는 읽기·복사만 할 수 있다. 이 실습은 Samba 공유 생성부터 그룹 연동, Sticky Bit 적용, 실제 삭제 차단 검증까지 진행한다.

---

## 1. 🎯 실습 목표 (Scenario)

### 1-1. 요구사항

```text
[요구사항 1] Linux 사용자가 /SHARE에 만든 파일을 Windows 사용자가 삭제할 수 있는 상태를 확인
[요구사항 2] /SHARE에서 파일을 생성한 소유자만 변경·삭제할 수 있도록 설정
[요구사항 3] 단, 다른 사용자도 읽기 및 복사는 가능해야 함
```

### 1-2. 구성도

```text
Server-A (192.168.10.100)  Samba Server
└─ /SHARE  (공유 디렉터리, 그룹 SG, Sticky Bit)
     ├─ smb.conf [Share] 섹션
     └─ Samba 계정: samba, user1, user2 (모두 SG 그룹)

Windows (192.168.10.131)  Client
└─ \\192.168.10.100\SHARE  → 네트워크 드라이브 Z:
```

---

## 2. 🛠️ 단계별 실습 (Configuration)

### STEP 1. 공유 디렉터리·그룹·계정 준비

```bash
mkdir -p /SHARE                            # 공유 디렉터리
groupadd SG                                # 공유용 그룹

useradd -G SG user1                        # 테스트 계정 1
useradd -G SG user2                        # 테스트 계정 2
useradd -G SG samba                        # 기본 계정

passwd user1
passwd user2
```

Samba DB 등록:

```bash
smbpasswd -a user1
smbpasswd -a user2
smbpasswd -a samba
pdbedit -L                                 # 등록된 Samba 사용자 확인
```

---

### STEP 2. 디렉터리 소유권·권한 설정

```bash
chown root:SG /SHARE                       # 그룹 소유권을 SG로
chmod 777 /SHARE                           # 1차: 전체 쓰기 허용
ls -ld /SHARE                              # drwxrwxrwx 확인
```

---

### STEP 3. smb.conf 공유 섹션 작성

```bash
cp -a /etc/samba/smb.conf /etc/samba/smb.conf.bak
vi /etc/samba/smb.conf
```

```text
[Share]
        path = /SHARE
        writable = yes
        browseable = yes
        guest ok = no
        valid users = @SG
        force group = SG
        create mask = 0666
        directory mask = 0777
```

검증·반영:

```bash
testparm                                   # 문법 확인
systemctl restart smb nmb                  # 데몬 재시작
```

---

### STEP 4. 방화벽·SELinux

```bash
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
firewall-cmd --list-services               # samba 포함 확인
```

```bash
semanage fcontext -a -t samba_share_t "/SHARE(/.*)?"
restorecon -Rv /SHARE
ls -Zd /SHARE                              # samba_share_t 확인
```

---

### STEP 5. Windows에서 네트워크 드라이브 연결

```text
파일 탐색기 → 내 PC → 네트워크 드라이브 연결
드라이브: Z:
폴더:     \\192.168.10.100\SHARE
[다른 자격 증명 사용] 체크 → user1 / 비밀번호
```

---

### STEP 6. 문제 상황 재현 (Sticky Bit 미적용)

Linux에서 파일을 생성한다.

```bash
su - samba -c "touch /SHARE/linux_file.txt"   # samba 계정이 파일 생성
ls -l /SHARE                                  # 소유자 samba 확인
```

이 상태에서 Windows의 `user1`로 접속해 `linux_file.txt`를 삭제하면 삭제가 성공한다. 디렉터리에 쓰기 권한(777)이 있으면 파일 소유자가 아니어도 삭제할 수 있기 때문이다.

```text
파일 삭제 권한 = 파일 자체 권한이 아니라 "상위 디렉터리의 쓰기 권한"
→ 777 디렉터리에서는 누구나 남의 파일 삭제 가능
```

---

### STEP 7. Sticky Bit 적용 (해결)

```bash
chmod 1777 /SHARE                          # Sticky Bit 적용
ls -ld /SHARE                              # drwxrwxrwt 확인
```

```text
drwxrwxrwt   6 root SG 4096  7월 16 10:14 /SHARE
                  ↑
                  t = Sticky Bit
```

Sticky Bit가 걸린 디렉터리에서는 파일의 소유자, 디렉터리 소유자, root만 파일을 삭제하거나 이름을 변경할 수 있다. 다른 사용자는 읽기와 복사는 가능하지만 삭제는 불가능하다.

숫자 대신 기호로도 적용할 수 있다.

```bash
chmod +t /SHARE                            # Sticky Bit 추가
chmod -t /SHARE                            # Sticky Bit 제거
```

---

### STEP 8. 동작 검증

Linux 측에서 각각 다른 소유자로 파일을 생성한다.

```bash
su - user1 -c "touch /SHARE/user1_file.txt"
su - user2 -c "touch /SHARE/user2_file.txt"
ls -l /SHARE                               # 소유자 확인
```

교차 삭제 테스트:

```bash
su - user2 -c "rm -f /SHARE/user1_file.txt"   # 실패해야 정상
```

```text
rm: cannot remove '/SHARE/user1_file.txt': Operation not permitted
```

읽기·복사는 정상 동작한다.

```bash
su - user2 -c "cat /SHARE/user1_file.txt"           # 읽기 가능
su - user2 -c "cp /SHARE/user1_file.txt /tmp/"      # 복사 가능
```

Windows 측에서도 `user2`로 접속해 `user1_file.txt` 삭제를 시도하면 액세스 거부 메시지가 표시되고, 열기·복사는 정상적으로 수행된다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 접속·세션 확인

```bash
smbstatus                                  # 접속 사용자·공유·잠금 확인
smbclient -L localhost -U user1            # 공유 목록 조회
journalctl -u smb -n 50                    # 인증 실패 로그
```

### 3-2. 자주 겪는 문제

| 증상 | 원인·조치 |
|---|---|
| Sticky Bit를 걸어도 삭제됨 | 삭제 시도 계정이 root 또는 디렉터리 소유자인지 확인 |
| Windows에서 계속 삭제 가능 | Windows 자격 증명 캐시 → `net use * /delete` 후 재접속 |
| 파일 그룹이 SG가 아님 | `force group = SG` 누락, `chgrp -R SG /SHARE` |
| 새 파일 권한이 의도와 다름 | `create mask`, `directory mask` 값 확인 |
| 접속 자체가 거부 | `valid users = @SG`에 계정 그룹 포함 여부, `id user1` 확인 |
| 권한·설정 정상인데 거부 | SELinux `samba_share_t` 미적용 → `restorecon -Rv /SHARE` |

### 3-3. Sticky Bit 개념 요약

| 표기 | 의미 |
|---|---|
| `drwxrwxrwx` | Sticky Bit 없음, 누구나 타인 파일 삭제 가능 |
| `drwxrwxrwt` | Sticky Bit 있음, 소유자·디렉터리 소유자·root만 삭제 가능 |
| `drwxrwxrwT` | Sticky Bit는 있으나 other 실행 권한 없음 |

---

## 4. ✅ 최종 체크리스트

```text
[ ] samba 패키지 설치 및 smb·nmb 데몬 enable
[ ] /SHARE 생성, 그룹 SG 연동
[ ] Linux 계정 생성 후 smbpasswd -a 등록
[ ] smb.conf [Share] 섹션 작성 후 testparm 통과
[ ] 방화벽 samba 서비스 permanent 허용 + reload
[ ] SELinux samba_share_t 적용
[ ] Windows 네트워크 드라이브 연결 성공
[ ] chmod 1777 적용 후 drwxrwxrwt 확인
[ ] 타 사용자 삭제 차단 확인
[ ] 타 사용자 읽기·복사 정상 확인
```

> 📌 **핵심 요약**
> - 파일 삭제 권한은 상위 디렉터리의 쓰기 권한에 좌우된다
> - `chmod 1777`(Sticky Bit)로 소유자만 삭제 가능하게 제한
> - Sticky Bit는 읽기·복사는 허용하고 삭제·이름 변경만 차단
> - `force group`으로 공유 파일의 그룹 소유권 통일
> - Windows 자격 증명 캐시 때문에 테스트가 왜곡될 수 있음
> - 관련: 🐧 Linux Samba 서버 구축 · 📚 종합정리 Samba & NFS · 🚨 Samba·NFS 트러블슈팅 치트시트

---
