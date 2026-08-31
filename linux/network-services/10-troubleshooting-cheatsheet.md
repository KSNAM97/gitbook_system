# 트러블슈팅 치트시트 (SSH·vsFTP·SFTP·SCP·DHCP·DNS)

> **Tag:** #Linux #Troubleshooting #Cheatsheet #SSH #vsFTP #DHCP #DNS #SELinux #firewalld
> **핵심 요약:** 네트워크 서비스 장애는 대부분 데몬 미동작, 방화벽 미개방, 설정 후 재시작 누락, SELinux 차단, 권한 문제 다섯 가지로 수렴한다. 증상별 원인과 확인 명령을 서비스별로 정리하고, 공통 진단 순서(데몬 → 포트 → 방화벽 → SELinux → 로그)를 기준으로 범위를 좁혀 나간다. 변경 전 백업과 사전 검증 명령(`sshd -t`, `named-checkconf`, `dhcpd -t`) 사용을 습관화한다.

---

## 1. 공통 진단 5단계

```text
1) 데몬이 동작하는가        systemctl status 서비스
2) 포트가 열려 있는가        ss -tlnp | grep 포트
3) 방화벽이 허용하는가       firewall-cmd --list-all
4) SELinux가 막는가          ausearch -m avc -ts recent
5) 로그에 무엇이 남았는가    journalctl -u 서비스 -n 50
```

클라이언트 쪽 확인:

```bash
ping -c 3 서버IP                            # 기본 연결
nc -zv 서버IP 포트                          # 포트 도달 여부
telnet 서버IP 포트                          # 포트 도달 여부(대체)
```

> 서버에서는 정상인데 클라이언트에서만 실패하면 네트워크 경로·방화벽 문제일 가능성이 높고, 서버에서도 실패하면 데몬·설정 문제일 가능성이 높다. 이 구분만으로 원인 범위가 절반으로 줄어든다.

---

## 2. SSH

| 증상 | 원인 | 확인·대응 |
|---|---|---|
| `Connection refused` | sshd 미동작·포트 불일치 | `systemctl status sshd`, `ss -tlnp` |
| `Connection timed out` | 방화벽 차단·네트워크 문제 | `firewall-cmd --list-port`, `ping` |
| `Access denied`(root) | `PermitRootLogin no` | 일반 계정 접속 후 `su -` |
| `Access denied`(일반 계정) | `AllowUsers`·`AllowGroups` 미포함 | `sshd -T \| grep allow` |
| 포트 변경 후 접속 불가 | 방화벽 미개방 | `firewall-cmd --list-port` |
| 3회 실패 후 끊김 | `MaxAuthTries 3` | 의도된 정상 동작 |
| 30초 후 끊김 | `LoginGraceTime 30` | 의도된 정상 동작 |
| 재시작 후 sshd 죽음 | 설정 문법 오류 | `sshd -t`로 사전 검증 |

진단 명령:

```bash
sshd -t                                     # 문법 검사
sshd -T | grep -iE 'port|permitroot|allowusers|allowgroups'
systemctl status sshd
ss -tlnp | grep sshd
tail -f /var/log/secure
```

> 원격에서 SSH 설정을 바꿀 때는 기존 세션을 열어둔 채 새 세션으로 검증한다. 잠기면 콘솔 접근 없이는 복구할 수 없다.

---

## 3. vsFTP

| 증상 | 원인 | 확인·대응 |
|---|---|---|
| 연결 시간 초과 | 방화벽 미개방·데몬 미동작 | `firewall-cmd --list-all`, `systemctl status vsftpd` |
| `530 Login incorrect` | 익명 차단 또는 비밀번호 오류 | `anonymous_enable` 확인 |
| `530 Permission denied`(비번 안 물음) | user_list·ftpusers 등록 | `cat /etc/vsftpd/user_list` |
| `550 Permission denied`(업로드) | `write_enable=NO`·디렉터리 권한 | 설정과 `ls -ld` 확인 |
| `500 OOPS: writable root` | chroot + 쓰기 가능 홈 | `allow_writeable_chroot=YES` |
| 업로드 파일 못 찾음 | 로컬 현재 디렉터리 문제 | 파일이 있는 폴더에서 접속 |
| 설정 바꿔도 그대로 | 데몬 재시작 누락 | `systemctl restart vsftpd` |

진단 명령:

```bash
systemctl status vsftpd
ss -tlnp | grep :21
grep -vE '^\s*#|^$' /etc/vsftpd/vsftpd.conf   # 유효 설정만 확인
tail -f /var/log/xferlog
tail -f /var/log/secure
journalctl -u vsftpd -n 50
```

SELinux 확인:

```bash
getsebool -a | grep ftp
setsebool -P ftpd_full_access on            # 필요 시(운영은 신중히)
```

---

## 4. SFTP·SCP

| 오류 | 원인 | 대응 |
|---|---|---|
| `not a regular file` | 디렉터리인데 `-r` 누락 | `scp -r` 사용 |
| `stat 파일: No such file` | SFTP 업로드 시 로컬 현재 위치에 파일 없음 | `lcd`로 로컬 디렉터리 이동 |
| `Permission denied`(local) | 로컬 대상 쓰기 권한 부족 | `ls -ld`, 소유권·권한 조정 |
| `dest open ... Permission denied` | 원격 대상 쓰기 권한 부족 | 원격 디렉터리 권한 확인 |
| `No such file or directory` | 경로 오타·상대경로 착각 | `pwd`, `lpwd`로 위치 확인 |
| `Connection refused` | sshd 미동작·포트 불일치 | 포트 지정 확인 |
| `Host key verification failed` | 호스트 키 변경 | `ssh-keygen -R 호스트` |

진단:

```bash
ssh 계정@호스트 'ls -ld /원격/대상'          # 원격 권한 확인
ls -ld /로컬/대상                            # 로컬 권한 확인
```

---

## 5. DHCP

| 증상 | 원인 | 확인·대응 |
|---|---|---|
| IP를 받지 못함 | dhcpd 미동작 | `systemctl status dhcpd` |
| IP를 받지 못함 | 방화벽 67/udp 차단 | `firewall-cmd --list-port` |
| 엉뚱한 대역의 IP | 다른 DHCP 서버 존재 | VMware DHCP 중지 확인 |
| dhcpd 기동 실패 | subnet 대역과 서버 IP 불일치 | `dhcpd -t -cf` |
| dhcpd 기동 실패 | 설정 문법 오류 | `journalctl -u dhcpd` |
| 특정 장비만 못 받음 | range 고갈·MAC 예약 충돌 | `dhcpd.leases` 확인 |
| 클라이언트가 고정 IP | `method=manual` | `nmcli con mod ... ipv4.method auto` |

진단 명령:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf           # 문법 검사
systemctl status dhcpd
journalctl -u dhcpd -f                      # DORA 과정 실시간 확인
cat /var/lib/dhcpd/dhcpd.leases
ss -ulnp | grep :67
```

클라이언트 재시도:

```bash
nmcli con down ens160 && nmcli con up ens160   # Linux(SSH 세션 중이면 접속 끊김 주의)
```

```text
ipconfig /release && ipconfig /renew           # Windows
```

---

## 6. DNS (BIND)

| 증상 | 원인 | 확인·대응 |
|---|---|---|
| named 기동 실패 | named.conf 문법 오류 | `named-checkconf` |
| zone 로딩 실패 | zone 파일 문법 오류 | `named-checkzone` |
| zone 로딩 실패 | 파일 권한 문제 | `chown root:named`, `chmod 640` |
| `NXDOMAIN` | 레코드 없음·zone 이름 오류 | zone 선언·파일명 확인 |
| 이름이 중복되어 조회됨 | 도메인 끝 점(`.`) 누락 | zone 파일 수정 |
| 변경이 반영 안 됨 | serial 미증가 | serial 증가 후 재시작·`rndc reload` |
| 실제 사이트로 연결됨 | 클라이언트가 외부 DNS 사용 | `/etc/resolv.conf` 확인 |
| 외부에서 조회 불가 | 방화벽·listen-on·allow-query | 세 곳 모두 확인 |
| 역방향 조회 실패 | in-addr.arpa 옥텟 순서 오류 | zone 이름 재확인 |

진단 명령:

```bash
named-checkconf
named-checkzone soldesk.com /var/named/soldesk.com.db
named-checkzone 10.168.192.in-addr.arpa /var/named/soldesk.com.rev
journalctl -u named -n 50
ls -l /var/named/
ss -tulnp | grep :53
```

조회 검증:

```bash
nslookup 도메인 127.0.0.1                   # 서버 자체 검증
nslookup IP 127.0.0.1                       # 역방향 검증
```

---

## 7. SELinux 공통

```bash
getenforce                                  # Enforcing / Permissive
ausearch -m avc -ts recent                  # 최근 차단 기록
```

문제 확인용 임시 전환:

```bash
setenforce 0                                # 일시적으로 Permissive
# 테스트 후
setenforce 1                                # 반드시 원복
```

포트 관련 등록:

```bash
semanage port -l | grep 서비스               # 등록된 포트 확인
semanage port -a -t ssh_port_t -p tcp 2002  # 새 포트 등록
```

> SELinux를 영구 비활성화하는 것은 권장하지 않는다. 임시로 Permissive로 두고 원인을 찾은 뒤, 올바른 포트·컨텍스트·boolean을 등록해 Enforcing 상태로 되돌린다.

---

## 8. 방화벽 공통

```bash
firewall-cmd --state                        # 방화벽 동작 여부
firewall-cmd --list-all                     # 현재 zone 전체 규칙
firewall-cmd --permanent --add-service=이름
firewall-cmd --permanent --add-port=포트/tcp
firewall-cmd --reload                       # 반드시 실행
```

자주 하는 실수:

```text
--permanent 없이 추가        → 재부팅 시 사라짐
--permanent 후 --reload 누락 → 즉시 반영 안 됨
서비스만 열고 포트 누락      → 비표준 포트 차단
```

---

## 9. 증상별 빠른 인덱스

| 증상 키워드 | 우선 확인 |
|---|---|
| 연결 시간 초과 | 방화벽 → 데몬 → 네트워크 |
| Connection refused | 데몬 미동작 → 포트 불일치 |
| Permission denied | 파일 권한 → 설정 옵션 → SELinux |
| 설정 바꿨는데 그대로 | 데몬 재시작 → 문법 오류 |
| 포트 변경 후 안 됨 | 방화벽 → SELinux → 설정값 |
| 이름 조회 실패 | resolv.conf → zone 파일 → serial |
| IP 못 받음 | dhcpd → 67/udp → range |
| 재부팅 후 안 됨 | enable 여부 → `--permanent` |

---

## 10. 사고 예방 체크리스트

```text
[ ] 설정 파일 수정 전 백업(cp 파일 파일.bak.$(date +%F))
[ ] SSH 설정은 sshd -t로 사전 검증
[ ] DNS 설정은 named-checkconf / named-checkzone로 검증
[ ] DHCP 설정은 dhcpd -t -cf로 검증
[ ] 원격 작업 시 기존 세션 유지한 채 새 세션 테스트
[ ] 방화벽은 --permanent + --reload 세트로 실행
[ ] 서비스는 enable로 자동 시작 등록
[ ] zone 파일 수정 시 serial 증가
[ ] 변경 후 로그(journalctl, /var/log/secure) 확인
```

>  **핵심 요약**
> - 진단 순서는 데몬 → 포트 → 방화벽 → SELinux → 로그
> - 대부분의 장애는 재시작 누락·방화벽 미개방·SELinux 차단
> - 오류 메시지를 그대로 읽고 원인 범위를 좁히는 것이 최우선
> - 변경 전 백업, 변경 전 문법 검증을 습관화
> - 원격 작업은 기존 세션을 유지한 채 검증
> - 관련:  종합정리 네트워크 서비스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS) ·  퀵 레퍼런스

---
