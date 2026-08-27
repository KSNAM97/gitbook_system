# Rocky Linux 9 및 Bash 실무 명령어 가이드

> [!info] 문서 범위
> Rocky Linux 9 기본 설치와 일반적인 서버 운영에서 사용하는 명령어, Bash 문법, 주요 설정 파일을 정리한 문서다.
>
> 설치한 패키지마다 추가되는 모든 명령어를 문자 그대로 포함할 수는 없으므로, 개별 명령의 전체 옵션은 `man`, `info`, `--help`로 확인한다.

## 표기 규칙

| 표기 | 의미 |
|---|---|
| `<값>` | 환경에 맞게 변경할 값 |
| `$` | 일반 사용자 명령 프롬프트 |
| `#` | root 프롬프트 또는 코드 내부 주석 |
| `sudo` | 관리자 권한 필요 |
| 위험 | 데이터 손실, 접속 단절 또는 부팅 실패 가능 |

> [!danger] 작업 전 주의사항
> `rm -rf`, `mkfs`, `fdisk`, `parted`, `pvremove`, `vgremove`, `lvremove`, `rsync --delete`는 데이터 손실을 일으킬 수 있다.
>
> 디스크, SSH, 네트워크, 방화벽, SELinux, GRUB 설정을 변경하기 전에 백업 또는 스냅샷과 콘솔 접근 수단을 준비한다.

## 공식 문서

- [Rocky Linux 9 Documentation](https://docs.rockylinux.org/9/)
- [Red Hat Enterprise Linux 9 Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9)
- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/)

---

# 1. 도움말과 명령어 검색

```bash
man ls                              # ls 매뉴얼
man 5 passwd                        # passwd 파일 형식 매뉴얼
man -k network                      # 설명에서 키워드 검색
apropos filesystem                  # 관련 매뉴얼 검색
info coreutils                      # GNU Coreutils 상세 문서

ls --help                           # 명령어 자체 도움말
systemctl --help
help cd                             # Bash 내장 명령어 도움말
help test

type cd                             # alias/function/builtin/file 구분
type -a python                      # 같은 이름의 모든 명령 표시
command -v bash                     # 실행할 명령의 경로
whereis bash                        # 실행 파일 및 매뉴얼 위치

compgen -c | sort -u                # 현재 실행 가능한 명령 목록
compgen -b                          # Bash 내장 명령어 목록

dnf provides '*/semanage'           # 명령을 제공하는 패키지 검색
rpm -qf /usr/bin/bash               # 파일을 소유한 패키지 확인
rpm -ql coreutils                   # 패키지가 설치한 파일 목록
```

---

# 2. 시스템 정보

```bash
cat /etc/rocky-release              # Rocky Linux 버전
cat /etc/os-release                 # 배포판 상세 정보

hostname                            # 현재 호스트명
hostnamectl                         # 호스트명, OS, 커널, 가상화 정보
uname -a                            # 전체 커널 정보
uname -r                            # 커널 버전
uname -m                            # CPU 아키텍처

lscpu                               # CPU 상세 정보
nproc                               # 사용 가능한 CPU 개수
free -h                             # 메모리 사용량
cat /proc/meminfo                   # 메모리 상세 정보
uptime                              # 가동 시간과 load average

who                                 # 로그인 사용자
w                                   # 로그인 사용자와 작업
whoami                              # 현재 사용자
id                                  # UID, GID, 그룹
date                                # 현재 시간
timedatectl                         # 시간대와 NTP 상태

lsblk -f                            # 디스크와 파일시스템
df -hT                              # 파일시스템 사용량
findmnt                             # 마운트 구조
lspci                               # PCI 장치
lsusb                               # USB 장치
systemd-detect-virt                 # VM 또는 컨테이너 여부
```

---

# 3. Bash 기본 문법

## 3.1 변수와 환경변수

```bash
VAR="Rocky Linux"                   # = 주변에 공백을 넣지 않는다
echo "$VAR"
echo "${VAR}"

readonly VERSION="9"                # 읽기 전용 변수
unset VAR                           # 변수 삭제

export APP_ENV="production"         # 자식 프로세스에 전달
env                                 # 환경변수 전체
printenv PATH                       # 특정 환경변수

echo '$HOME'                        # 작은따옴표: 변수 확장 안 함
echo "$HOME"                        # 큰따옴표: 변수 확장
```

## 3.2 명령어 치환과 산술 연산

```bash
TODAY=$(date +%F)
CURRENT_DIR=$(pwd)

RESULT=$((10 + 20))
echo $((2 ** 8))
echo $((10 % 3))
```

## 3.3 명령 연결

```bash
cmd1 ; cmd2                         # 성공 여부와 관계없이 순서대로 실행
cmd1 && cmd2                        # cmd1 성공 시 cmd2 실행
cmd1 || cmd2                        # cmd1 실패 시 cmd2 실행
! command                           # 종료 상태 반전

(command1; command2)                # 서브셸에서 실행
{ command1; command2; }             # 현재 셸에서 실행

sleep 60 &
wait
```

## 3.4 특수 변수

```bash
echo $?                             # 직전 명령의 종료 코드
echo $$                             # 현재 셸 PID
echo $!                             # 최근 백그라운드 프로세스 PID

echo "$0"                           # 스크립트 이름
echo "$1"                           # 첫 번째 인자
echo "$#"                           # 인자 개수
printf '%s\n' "$@"                  # 모든 인자를 개별 보존
shift                               # 첫 번째 인자를 제거하고 이동
```

## 3.5 글로빙

```bash
ls *.log                            # .log로 끝나는 모든 파일
ls file?.txt                        # 임의의 문자 한 개
ls file[0-9].txt                    # 숫자 한 개

echo file{1..5}.txt
echo {app,db,web}.conf
```

## 3.6 리다이렉션과 파이프

```bash
command > output.txt                # 표준출력 덮어쓰기
command >> output.txt               # 표준출력 이어쓰기

command 2> error.txt                # 표준에러 덮어쓰기
command 2>> error.txt               # 표준에러 이어쓰기

command > all.txt 2>&1              # stdout과 stderr 저장
command &> all.txt                  # Bash 전용 표기

command < input.txt                 # 파일을 표준입력으로 사용
command > /dev/null 2>&1            # 모든 출력 버리기

command1 | command2                 # stdout 파이프
command1 |& command2                # stdout과 stderr 파이프

command | tee output.txt            # 화면 출력과 파일 저장
command | tee -a output.txt         # 파일에 이어쓰기
```

## 3.7 Here Document

```bash
cat > /tmp/example.conf <<'EOF'
# EOF를 작은따옴표로 감싸면 변수와 명령어를 확장하지 않는다.
name=$USER
path=$(pwd)
EOF
```

---

# 4. Bash 조건문과 반복문

## 4.1 조건식

```bash
[[ -z "$VAR" ]]                     # 빈 문자열
[[ -n "$VAR" ]]                     # 빈 문자열이 아님

[[ "$A" == "$B" ]]                  # 문자열 같음
[[ "$A" != "$B" ]]                  # 문자열 다름
[[ "$A" == Rocky* ]]                # 패턴 일치
[[ "$A" =~ ^[0-9]+$ ]]              # 정규식 일치

[[ "$A" -eq "$B" ]]                 # 숫자 같음
[[ "$A" -ne "$B" ]]                 # 숫자 다름
[[ "$A" -gt "$B" ]]                 # 큼
[[ "$A" -ge "$B" ]]                 # 크거나 같음
[[ "$A" -lt "$B" ]]                 # 작음
[[ "$A" -le "$B" ]]                 # 작거나 같음

(( A > B ))                         # Bash 산술 조건
```

## 4.2 파일 조건식

```bash
[[ -e /path ]]                      # 존재
[[ -f /path/file ]]                 # 일반 파일
[[ -d /path/dir ]]                  # 디렉터리
[[ -L /path/link ]]                 # 심볼릭 링크

[[ -r /path/file ]]                 # 읽기 가능
[[ -w /path/file ]]                 # 쓰기 가능
[[ -x /path/file ]]                 # 실행 가능
[[ -s /path/file ]]                 # 크기가 0보다 큼

[[ file1 -nt file2 ]]               # file1이 더 최신
[[ file1 -ot file2 ]]               # file1이 더 오래됨

[[ -f "$FILE" && -r "$FILE" ]]      # AND
[[ "$A" == y || "$A" == yes ]]      # OR
[[ ! -e "$FILE" ]]                  # NOT
```

## 4.3 if 문

```bash
if [[ -f /etc/rocky-release ]]; then
    echo "Rocky Linux"
elif [[ -f /etc/redhat-release ]]; then
    echo "RHEL 계열"
else
    echo "기타 운영체제"
fi
```

## 4.4 case 문

```bash
case "$1" in
    start)
        echo "시작"
        ;;
    stop)
        echo "중지"
        ;;
    restart|reload)
        echo "재시작 또는 재로드"
        ;;
    *)
        echo "사용법: $0 {start|stop|restart|reload}"
        exit 2
        ;;
esac
```

## 4.5 반복문

```bash
for FILE in /var/log/*.log; do
    echo "$FILE"
done

for ((I=0; I<5; I++)); do
    echo "$I"
done

COUNT=1

while (( COUNT <= 5 )); do
    echo "$COUNT"
    ((COUNT++))
done

while IFS= read -r LINE; do
    printf '%s\n' "$LINE"
done < /etc/hosts

break                               # 반복문 종료
continue                            # 현재 반복 건너뛰기
```

## 4.6 함수와 배열

```bash
backup_file() {
    local SOURCE="$1"
    local DEST="${SOURCE}.$(date +%F_%H%M%S).bak"

    [[ -f "$SOURCE" ]] || {
        echo "파일이 없습니다: $SOURCE" >&2
        return 1
    }

    cp -a -- "$SOURCE" "$DEST"
    echo "백업 완료: $DEST"
}

backup_file /etc/hosts
```

```bash
SERVICES=(sshd chronyd firewalld)

for SERVICE in "${SERVICES[@]}"; do
    systemctl is-active "$SERVICE"
done

declare -A PORTS=(
    [ssh]=22
    [http]=80
    [https]=443
)

echo "${PORTS[https]}"
```

---

# 5. Bash 스크립트 기본 템플릿

```bash
#!/usr/bin/env bash

# -e: 처리되지 않은 명령 실패 시 종료
# -u: 선언되지 않은 변수 사용 시 종료
# pipefail: 파이프 중간 명령 실패도 실패로 처리
set -Eeuo pipefail

trap 'echo "오류: line=$LINENO status=$?" >&2' ERR

usage() {
    echo "사용법: $0 <파일>"
}

main() {
    if (( $# != 1 )); then
        usage
        exit 2
    fi

    local file="$1"

    if [[ ! -f "$file" ]]; then
        echo "일반 파일이 아닙니다: $file" >&2
        exit 1
    fi

    printf '파일=%s\n' "$file"
    printf '크기=%s bytes\n' "$(stat -c %s "$file")"
}

main "$@"
```

```bash
chmod +x script.sh
bash -n script.sh                   # 문법 검사
bash -x script.sh                   # 실행 과정 추적
shellcheck script.sh                # shellcheck 패키지 필요
```

> [!warning] Bash 작성 원칙
> - 변수와 경로는 특별한 이유가 없다면 `"$VAR"`처럼 감싼다.
> - 외부 입력을 `eval`에 전달하지 않는다.
> - `set -e`에는 `if`, `&&`, `||`, `!` 등의 예외가 있으므로 맹신하지 않는다.

---

# 6. 히스토리와 작업 제어

```bash
history
history 20
!!
!123
Ctrl+r

alias ll='ls -alF --color=auto'
unalias ll

jobs -l
fg %1
bg %1
Ctrl+z
Ctrl+c

nohup command > app.log 2>&1 &
disown %1
```

---

# 7. 파일과 디렉터리

```bash
pwd

ls
ls -la                              # 숨김 파일 포함
ls -lh                              # 읽기 쉬운 크기
ls -ltr                             # 오래된 순
ls -ld /path/dir                    # 디렉터리 자체 정보

cd /var/log
cd ..
cd -

touch file.txt
mkdir directory
mkdir -p /srv/app/{bin,conf,logs}

cp source destination
cp -a source/ destination/          # 속성 보존
cp -i source destination            # 덮어쓰기 확인

mv old new
mv -i old new

rm file
rm -i file
rm -r directory
rm -rf directory                    # 위험: 경로 재확인
rmdir empty_directory

install -m 0644 app.conf /etc/app/

ln file hardlink
ln -s /real/path symlink
readlink -f symlink
realpath ./relative/path

file filename
stat filename
basename /var/log/messages
dirname /var/log/messages
```

## 안전한 임시 파일

```bash
TMPFILE=$(mktemp)
TMPDIR=$(mktemp -d)

trap 'rm -rf -- "$TMPFILE" "$TMPDIR"' EXIT
```

---

# 8. 파일 내용과 텍스트 처리

```bash
cat file
cat -n file
less file                           # q 종료, / 검색

head -n 20 file
tail -n 50 file
tail -F /var/log/messages           # 실시간 추적

wc -l file                          # 줄 수
wc -w file                          # 단어 수
wc -c file                          # 바이트 수
```

## grep

```bash
grep root /etc/passwd
grep -in error app.log
grep -Rni 'pattern' /etc
grep -vE '^(#|$)' file              # 주석과 빈 줄 제외
grep -E 'error|fail' app.log
grep -F 'a.b' file                  # 고정 문자열
grep -C 3 error app.log             # 앞뒤 3줄
```

## cut, sort, uniq, tr

```bash
cut -d: -f1 /etc/passwd
sort file
sort -n numbers.txt
sort -u file
sort file | uniq -c
tr 'a-z' 'A-Z' < file
tr -d '\r' < windows.txt > unix.txt
column -t -s: /etc/passwd
```

## 비교와 체크섬

```bash
diff -u old.conf new.conf
cmp file1 file2

sha256sum file
sha256sum -c checksums.txt
```

## sed

```bash
sed -n '1,20p' file
sed 's/old/new/' file
sed 's/old/new/g' file
sed -i.bak 's/old/new/g' file       # 백업 후 직접 수정
sed '/^#/d;/^$/d' file
```

## awk

```bash
awk -F: '{print $1}' /etc/passwd
awk -F: '$3 >= 1000 {print $1,$3}' /etc/passwd
awk '{sum += $1} END {print sum}' numbers.txt

df -P | awk 'NR>1 && $5+0 >= 80 {print $6,$5}'
```

## JSON

```bash
sudo dnf install -y jq

jq '.' data.json
jq -r '.users[].name' data.json
```

---

# 9. 파일 검색

```bash
find /etc -name '*.conf'
find /etc -iname '*.CONF'
find /var/log -type f
find /tmp -type d

find /var/log -type f -size +100M
find /tmp -type f -mtime +7
find /tmp -type f -mmin -30
find /srv -user rocky
find /srv -perm /002

# 삭제 전 대상을 먼저 출력한다.
find /tmp -type f -mtime +7 -print

# 대상 확인 후 삭제한다.
find /tmp -type f -mtime +7 -delete

find /path -type f -exec chmod 640 {} +
find /path -type d -exec chmod 750 {} +

# 공백과 개행이 포함된 파일명도 안전하게 처리
find /path -type f -print0 | xargs -0 sha256sum
```

---

# 10. 압축과 아카이브

```bash
tar -cvf backup.tar /etc
tar -tvf backup.tar
tar -xvf backup.tar

tar -czf backup.tar.gz /etc
tar -xzf backup.tar.gz
tar -xzf backup.tar.gz -C /restore

tar --exclude='*.log' -czf backup.tar.gz /srv/app

gzip -k file
gunzip file.gz
zgrep error app.log.gz
```

```bash
sudo dnf install -y zip unzip

zip -r archive.zip directory/
unzip -l archive.zip
unzip archive.zip -d destination/
```

---

# 11. 권한, 소유권, ACL

## 숫자 권한

| 숫자 | 권한 |
|---:|---|
| `7` | `rwx` |
| `6` | `rw-` |
| `5` | `r-x` |
| `4` | `r--` |
| `0` | `---` |

```bash
chmod 640 file
chmod 600 private.key
chmod 700 private_directory
chmod 755 executable

chmod u+x script.sh
chmod g-w file
chmod -R u=rwX,g=rX,o= directory

chown user file
chown user:group file
chown -R user:group directory
chgrp group file

umask
umask 027
```

## 특수 권한

```bash
chmod u+s executable                # SUID
chmod g+s shared_directory          # SGID 및 그룹 상속
chmod +t shared_directory           # sticky bit

find / -xdev -perm /6000 -type f    # SUID/SGID 파일 점검
```

## ACL

```bash
getfacl file

setfacl -m u:alice:rw file
setfacl -m g:developers:rwx directory
setfacl -d -m g:developers:rwx directory

setfacl -x u:alice file
setfacl -b file
```

## 파일 속성

```bash
lsattr file

chattr +i file                      # 수정 및 삭제 차단
chattr -i file

chattr +a logfile                   # append-only
```

---

# 12. 사용자와 그룹

```bash
getent passwd
getent passwd rocky
getent group wheel

id rocky
groups rocky

sudo useradd -m -s /bin/bash rocky
sudo passwd rocky

sudo passwd -l rocky                # 암호 로그인 잠금
sudo passwd -u rocky                # 잠금 해제

sudo usermod -aG wheel rocky
sudo usermod -s /bin/bash rocky
sudo usermod -e 2026-12-31 rocky

sudo userdel rocky
sudo userdel -r rocky               # 홈 디렉터리 포함 삭제
```

> [!warning]
> `usermod -G`를 `-a` 없이 사용하면 기존 보조 그룹이 제거될 수 있다. 기존 그룹을 유지하며 추가하려면 `usermod -aG`를 사용한다.

```bash
sudo groupadd developers
sudo groupmod -n devops developers
sudo gpasswd -a rocky developers
sudo gpasswd -d rocky developers
sudo groupdel developers
```

## 암호 만료 정책

```bash
chage -l rocky
sudo chage -M 90 -m 1 -W 14 rocky
sudo chage -d 0 rocky               # 다음 로그인 시 암호 변경
```

## 사용자 전환과 sudo

```bash
su - rocky
sudo -i
sudo -u rocky command
sudo -l

visudo
visudo -cf /etc/sudoers
sudo visudo -f /etc/sudoers.d/rocky
```

`/etc/sudoers.d/rocky` 예시:

```sudoers
# rocky가 모든 명령을 비밀번호 확인 후 실행
rocky ALL=(ALL) ALL

# 특정 서비스 재시작만 비밀번호 없이 허용
rocky ALL=(root) NOPASSWD: /usr/bin/systemctl restart httpd
```

> [!danger]
> `NOPASSWD: ALL` 또는 셸을 실행할 수 있는 편집기와 프로그램을 허용하면 사실상 전체 root 권한이 될 수 있다.

---

# 13. 프로세스와 리소스

```bash
ps aux
ps -ef
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu | head
pstree -p

pgrep -a sshd
pidof sshd

top
top -p <PID>

sudo dnf install -y htop
htop
```

## 프로세스 종료

```bash
kill <PID>                          # SIGTERM, 정상 종료 요청
kill -TERM <PID>
kill -HUP <PID>                     # 설정 재로드 등에 사용
kill -KILL <PID>                    # 강제 종료, 마지막 수단

pkill process_name
pkill -u rocky
```

## 우선순위와 제한

```bash
nice -n 10 command
renice 10 -p <PID>
ionice -c2 -n7 command

ulimit -a
watch -n 2 free -h
timeout 30 command
time command
```

## 열린 파일

```bash
sudo dnf install -y lsof

lsof /path/file
lsof -p <PID>
lsof -iTCP -sTCP:LISTEN
```

---

# 14. systemd 서비스

```bash
systemctl status sshd

sudo systemctl start sshd
sudo systemctl stop sshd
sudo systemctl restart sshd
sudo systemctl reload sshd
sudo systemctl reload-or-restart sshd

sudo systemctl enable sshd
sudo systemctl disable sshd
sudo systemctl enable --now sshd
sudo systemctl disable --now sshd

sudo systemctl mask <service>
sudo systemctl unmask <service>

systemctl is-active sshd
systemctl is-enabled sshd
systemctl is-failed sshd

systemctl --failed
systemctl list-units --type=service
systemctl list-unit-files --type=service
systemctl list-dependencies sshd

systemctl cat sshd
systemctl show sshd
systemctl show -p MainPID sshd

sudo systemctl daemon-reload
sudo systemctl reset-failed
```

## 부팅 타깃

```bash
systemctl get-default

sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target

sudo systemctl reboot
sudo systemctl poweroff
```

## 사용자 서비스

```bash
systemctl --user status
systemctl --user enable --now myapp.service

# 로그아웃 후에도 사용자 서비스 유지
loginctl enable-linger rocky
```

## 커스텀 서비스 설정

파일: `/etc/systemd/system/myapp.service`

```ini
[Unit]
# 서비스 설명
Description=My Application

# 네트워크가 온라인 상태가 된 후 시작
Wants=network-online.target
After=network-online.target

[Service]
# 일반 장기 실행 프로세스
Type=simple

# root 대신 전용 계정 사용
User=myapp
Group=myapp

WorkingDirectory=/srv/myapp

# 셸 문법이 자동 적용되지 않으므로 절대 경로 권장
ExecStart=/usr/local/bin/myapp --config /etc/myapp/myapp.conf

# 실패한 경우 5초 뒤 재시작
Restart=on-failure
RestartSec=5s

Environment="APP_ENV=production"

# 파일이 없어도 실패하지 않도록 앞에 - 사용
EnvironmentFile=-/etc/sysconfig/myapp

# 기본 보안 강화
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemd-analyze verify /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
systemctl status myapp
```

패키지가 설치한 원본 unit은 직접 수정하지 않고 override를 사용한다.

```bash
sudo systemctl edit myapp
```

```ini
[Service]
Environment="LOG_LEVEL=debug"
Restart=always
```

---

# 15. 로그와 journalctl

```bash
journalctl
journalctl -b                       # 현재 부팅 로그
journalctl -b -1                    # 이전 부팅 로그

journalctl -u sshd
journalctl -u sshd -f

journalctl -p err
journalctl -p warning..alert

journalctl --since today
journalctl --since "1 hour ago"
journalctl --since "2026-07-22 09:00:00"

journalctl _PID=<PID>
journalctl _UID=<UID>
journalctl -k

journalctl --disk-usage
sudo journalctl --vacuum-time=30d
sudo journalctl --vacuum-size=1G

dmesg -T
dmesg -w
```

## journald 영구 보관 설정

파일: `/etc/systemd/journald.conf.d/retention.conf`

```ini
[Journal]
# /var/log/journal에 영구 보관
Storage=persistent

# 전체 journal 최대 사용량
SystemMaxUse=1G

# 최대 보관 기간
MaxRetentionSec=30day

Compress=yes
```

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo systemctl restart systemd-journald
```

## logrotate 설정

파일: `/etc/logrotate.d/myapp`

```text
/var/log/myapp/*.log {
    # 매일 회전
    daily

    # 최근 14개 보관
    rotate 14

    # 이전 로그 압축
    compress
    delaycompress

    # 파일이 없어도 오류를 내지 않음
    missingok

    # 빈 로그는 회전하지 않음
    notifempty

    # 새 로그의 권한과 소유자
    create 0640 myapp myapp

    # 날짜 확장자 사용
    dateext
}
```

```bash
sudo logrotate -d /etc/logrotate.conf
sudo logrotate -f /etc/logrotate.conf
```

---

# 16. DNF와 RPM

```bash
sudo dnf check-update               # 업데이트가 있으면 종료코드 100 가능
sudo dnf upgrade -y

sudo dnf install -y nginx
sudo dnf reinstall nginx
sudo dnf remove nginx
sudo dnf autoremove

dnf search nginx
dnf info nginx
dnf list installed
dnf provides '*/dig'
dnf repoquery nginx

dnf history
dnf history info <ID>
sudo dnf history undo <ID>

sudo dnf clean all
sudo dnf makecache

dnf repolist
dnf repolist all
```

## CRB와 EPEL

```bash
sudo dnf config-manager --set-enabled crb
sudo dnf install -y epel-release
dnf repolist
```

## 그룹과 버전 고정

```bash
dnf group list
sudo dnf group install -y "Development Tools"

sudo dnf install -y 'dnf-command(versionlock)'
sudo dnf versionlock add nginx
dnf versionlock list
sudo dnf versionlock delete nginx
```

## 재시작 필요 여부

```bash
sudo dnf install -y dnf-utils

sudo dnf needs-restarting
sudo dnf needs-restarting -r
```

## RPM

```bash
rpm -qa
rpm -q bash
rpm -qi bash
rpm -ql bash
rpm -qf /usr/bin/bash

rpm -V bash                         # 설치 당시 상태와 비교
rpm -K package.rpm                  # 서명과 체크섬 확인

sudo dnf install ./package.rpm      # 의존성 처리
```

## 사용자 정의 저장소

파일: `/etc/yum.repos.d/internal.repo`

```ini
[internal]
name=Internal Repository
baseurl=https://repo.example.com/rocky/9/$basearch/
enabled=1

# 패키지와 저장소 메타데이터 서명 검증
gpgcheck=1
repo_gpgcheck=1

gpgkey=https://repo.example.com/RPM-GPG-KEY-internal
```

---

# 17. 네트워크 확인

```bash
ip address show
ip -br address
ip link show
ip route show
ip route get 1.1.1.1
ip neigh show
hostname -I

nmcli general status
nmcli device status
nmcli device show
nmcli connection show
nmcli connection show --active
nmtui

ping -c 4 1.1.1.1
ping -c 4 example.com
tracepath example.com

curl -I https://example.com
curl -fsS https://example.com
curl -L -O <URL>
wget <URL>
```

## 소켓과 포트 확인

```bash
ss -lnt                             # TCP 리스닝 포트
ss -lnu                             # UDP 소켓
ss -lntp                            # 프로세스 포함
ss -tan                             # 전체 TCP 연결
ss -s                               # 소켓 요약
```

## DNS 확인

```bash
getent hosts example.com

sudo dnf install -y bind-utils

dig example.com
dig +short example.com
dig @1.1.1.1 example.com
dig -x 192.0.2.10
host example.com
```

---

# 18. NetworkManager 설정

> [!warning]
> SSH로 접속한 상태에서 IP, 게이트웨이 또는 연결 프로파일을 변경하면 즉시 접속이 끊길 수 있다.

## DHCP 연결

```bash
sudo nmcli connection add \
    type ethernet \
    ifname enp1s0 \
    con-name lan-dhcp \
    ipv4.method auto \
    ipv6.method auto

sudo nmcli connection up lan-dhcp
```

## 고정 IPv4 연결

```bash
sudo nmcli connection add \
    type ethernet \
    ifname enp1s0 \
    con-name lan-static \
    ipv4.method manual \
    ipv4.addresses 192.0.2.10/24 \
    ipv4.gateway 192.0.2.1 \
    ipv4.dns "1.1.1.1 8.8.8.8" \
    ipv6.method disabled

sudo nmcli connection modify lan-static connection.autoconnect yes
sudo nmcli connection up lan-static
```

## 기존 연결 수정

```bash
sudo nmcli connection modify lan-static ipv4.addresses 192.0.2.20/24
sudo nmcli connection modify lan-static +ipv4.dns 9.9.9.9
sudo nmcli connection modify lan-static -ipv4.dns 8.8.8.8
sudo nmcli connection modify lan-static ipv4.ignore-auto-dns yes

sudo nmcli connection down lan-static
sudo nmcli connection up lan-static

sudo nmcli connection delete lan-static
```

## DNS와 keyfile

```bash
nmcli device show enp1s0 | grep -E 'IP4.DNS|IP6.DNS'
cat /etc/resolv.conf

sudo chmod 600 /etc/NetworkManager/system-connections/*.nmconnection
sudo nmcli connection reload
```

> [!note]
> `/etc/resolv.conf`는 NetworkManager가 덮어쓸 수 있으므로 DNS는 `nmcli` 연결 프로파일에서 설정하는 것이 권장된다.

---

# 19. firewalld

```bash
sudo systemctl enable --now firewalld

firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-services
sudo firewall-cmd --zone=public --list-all
```

## 런타임 설정

즉시 적용되지만 reload 또는 재부팅 후 사라진다.

```bash
sudo firewall-cmd --zone=public --add-service=http
sudo firewall-cmd --zone=public --add-port=8080/tcp
```

## 영구 설정

`--reload` 후 런타임에도 적용된다.

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp

sudo firewall-cmd --reload
```

```bash
sudo firewall-cmd --runtime-to-permanent

sudo firewall-cmd --permanent --zone=public --remove-port=8080/tcp
sudo firewall-cmd --reload

sudo firewall-cmd --zone=public --query-service=ssh
```

## 소스 기반 허용

```bash
sudo firewall-cmd --permanent \
    --zone=trusted \
    --add-source=192.0.2.0/24

sudo firewall-cmd --reload
```

## Rich rule

```bash
sudo firewall-cmd --permanent \
    --zone=public \
    --add-rich-rule='rule family="ipv4" source address="192.0.2.10/32" port port="8443" protocol="tcp" accept'

sudo firewall-cmd --reload
```

> [!danger]
> SSH 포트 또는 zone을 변경하기 전에 현재 SSH 연결이 계속 허용되는지 확인한다.

---

# 20. SSH

## 클라이언트

```bash
ssh user@server
ssh -p 2222 user@server
ssh -i ~/.ssh/id_ed25519 user@server
ssh -v user@server

ssh -J jump@jump-host user@server

scp file user@server:/tmp/
scp -r directory user@server:/srv/
sftp user@server
```

## 키 생성

```bash
ssh-keygen -t ed25519 -a 100 -C "admin@example.com"
ssh-keygen -lf ~/.ssh/id_ed25519.pub
ssh-copy-id user@server

chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/config
```

## SSH 클라이언트 설정

파일: `~/.ssh/config`

```sshconfig
Host production
    HostName server.example.com
    User rocky
    Port 22

    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

    ServerAliveInterval 60
    ServerAliveCountMax 3
```

```bash
ssh production
```

## SSH 서버

```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd

sudo sshd -t                        # 설정 문법 검사
sshd -T                             # 실제 적용 설정
journalctl -u sshd
```

파일: `/etc/ssh/sshd_config.d/10-hardening.conf`

```sshconfig
# root 직접 로그인 차단
PermitRootLogin no

# 공개키 인증
PubkeyAuthentication yes

# 공개키 로그인을 별도 세션에서 확인한 후 비활성화
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no

X11Forwarding no
MaxAuthTries 4

ClientAliveInterval 300
ClientAliveCountMax 2

# 필요한 경우 사용자 또는 그룹 제한
# AllowUsers rocky deploy
# AllowGroups sshusers
```

```bash
sudo sshd -t && sudo systemctl reload sshd
```

## SSH 포트 변경 순서

1. SELinux 포트 타입을 등록한다.
2. 방화벽에서 새 포트를 연다.
3. SSH 설정을 변경한다.
4. 문법 검사 후 reload한다.
5. 새 터미널에서 접속을 확인한다.
6. 확인 후 기존 포트를 닫는다.

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

```sshconfig
Port 2222
```

```bash
sudo sshd -t && sudo systemctl reload sshd
ssh -p 2222 user@server
```

---

# 21. SELinux

```bash
getenforce
sestatus

ls -Z /var/www/html
ps -eZ
id -Z

sudo setenforce 0                   # 일시적 Permissive
sudo setenforce 1                   # Enforcing
```

파일: `/etc/selinux/config`

```ini
# enforcing  : 정책 위반 차단
# permissive : 차단하지 않고 기록
# disabled   : SELinux 비활성화, 일반적으로 비권장
SELINUX=enforcing

SELINUXTYPE=targeted
```

## 파일 컨텍스트

```bash
sudo restorecon -Rv /var/www/html

sudo dnf install -y policycoreutils-python-utils

sudo semanage fcontext \
    -a \
    -t httpd_sys_content_t \
    '/srv/www(/.*)?'

sudo restorecon -Rv /srv/www
```

웹서버 쓰기 디렉터리:

```bash
sudo semanage fcontext \
    -a \
    -t httpd_sys_rw_content_t \
    '/srv/www/uploads(/.*)?'

sudo restorecon -Rv /srv/www/uploads
```

## 포트 타입과 boolean

```bash
sudo semanage port -l | grep http_port_t
sudo semanage port -a -t http_port_t -p tcp 8080
sudo semanage port -d -t http_port_t -p tcp 8080

getsebool httpd_can_network_connect
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_can_network_connect off
```

## 거부 로그

```bash
sudo ausearch -m AVC,USER_AVC -ts recent
sudo ausearch -m AVC -ts today
```

> [!warning]
> SELinux 문제 해결을 위해 전체 SELinux를 비활성화하지 않는다.
>
> 다음 순서로 확인한다.
>
> 1. 파일 컨텍스트
> 2. SELinux boolean
> 3. 서비스 포트 타입
> 4. AVC 감사 로그
>
> 영구 파일 컨텍스트에는 `chcon`보다 `semanage fcontext`와 `restorecon`을 사용한다.

---

# 22. 시간과 chrony

```bash
date '+%F %T %Z'

timedatectl
timedatectl list-timezones
sudo timedatectl set-timezone Asia/Seoul
sudo timedatectl set-ntp true

sudo systemctl enable --now chronyd

chronyc tracking
chronyc sources -v
chronyc sourcestats -v
sudo chronyc makestep

hwclock --show
sudo hwclock --systohc
```

파일: `/etc/chrony.conf`

```conf
# iburst는 초기 동기화를 빠르게 수행
pool 2.rocky.pool.ntp.org iburst

# 초기 3회 측정에서 1초 이상 차이가 나면 즉시 보정
makestep 1.0 3

# 하드웨어 시계 동기화
rtcsync

driftfile /var/lib/chrony/drift
logdir /var/log/chrony

# 내부 NTP 서버로 제공할 때만 추가
# allow 192.0.2.0/24
```

```bash
sudo systemctl restart chronyd
chronyc tracking
```

---

# 23. 디스크와 파일시스템

```bash
lsblk -f
blkid
sudo fdisk -l
sudo parted -l

df -hT
df -i

du -sh /var/log
du -xhd1 /var | sort -h

findmnt
mount
```

> [!danger]
> 다음 파티션 및 파일시스템 생성 명령은 기존 데이터를 삭제할 수 있다. 장치명을 반드시 확인한다.

```bash
sudo fdisk /dev/sdb

sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary xfs 1MiB 100%
sudo partprobe /dev/sdb

sudo mkfs.xfs /dev/sdb1
# sudo mkfs.ext4 /dev/sdb1
```

## 마운트

```bash
sudo mkdir -p /data
sudo mount /dev/sdb1 /data
sudo umount /data

sudo fuser -vm /data                # 사용 중인 프로세스 확인
sudo blkid /dev/sdb1
```

파일: `/etc/fstab`

```fstab
# 장치/UUID       마운트점  타입  옵션              dump fsck
UUID=<실제-UUID>  /data     xfs   defaults,nofail  0    0
```

옵션 설명:

| 옵션 | 설명 |
|---|---|
| `defaults` | 기본 마운트 옵션 |
| `nofail` | 장치가 없어도 부팅 계속 |
| `_netdev` | 네트워크 스토리지 |
| `nodev` | 장치 파일 해석 차단 |
| `nosuid` | SUID/SGID 무시 |
| `noexec` | 실행 차단, 애플리케이션과 충돌 가능 |

```bash
sudo findmnt --verify --verbose
sudo mount -a
```

## 파일시스템 확장과 복구

```bash
sudo xfs_growfs /data               # XFS 확장, 축소 불가
sudo resize2fs /dev/mapper/vg-lv    # ext4 확장
```

일반적으로 검사와 복구는 마운트 해제 후 수행한다.

```bash
sudo xfs_repair -n /dev/sdb1        # 읽기 전용 점검
sudo xfs_repair /dev/sdb1

sudo e2fsck -f /dev/sdb1
```

## SWAP 파일

```bash
swapon --show

sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

`/etc/fstab`에 추가:

```fstab
/swapfile none swap defaults 0 0
```

---

# 24. LVM

```bash
sudo pvs
sudo vgs
sudo lvs
sudo lvs -a -o +devices
```

## LVM 생성

```bash
sudo pvcreate /dev/sdb1
sudo vgcreate vgdata /dev/sdb1
sudo lvcreate -n lvapp -L 20G vgdata

sudo mkfs.xfs /dev/vgdata/lvapp
sudo mkdir -p /srv/app
sudo mount /dev/vgdata/lvapp /srv/app
```

## LVM 확장

```bash
sudo lvextend -r -L +10G /dev/vgdata/lvapp
sudo lvextend -r -l +100%FREE /dev/vgdata/lvapp
```

## 디스크 추가와 제거

```bash
sudo pvcreate /dev/sdc1
sudo vgextend vgdata /dev/sdc1

sudo pvmove /dev/sdb1 /dev/sdc1
sudo vgreduce vgdata /dev/sdb1
sudo pvremove /dev/sdb1
```

## 스냅샷

```bash
sudo lvcreate \
    -s \
    -n lvapp_snap \
    -L 5G \
    /dev/vgdata/lvapp

sudo lvs
sudo lvremove /dev/vgdata/lvapp_snap
```

> [!warning]
> LVM 스냅샷은 원본 변경량만큼 공간을 사용하며 별도 백업을 대체하지 않는다.

---

# 25. 성능 확인

```bash
free -h
vmstat 1 5
uptime
cat /proc/loadavg

sudo dnf install -y sysstat tuned
sudo systemctl enable --now sysstat tuned

iostat -xz 1 5
mpstat -P ALL 1 5
pidstat 1 5
sar -u 1 5
sar -r
sar -n DEV

tuned-adm active
tuned-adm list
tuned-adm recommend
sudo tuned-adm profile throughput-performance
```

CPU와 메모리 상위 프로세스:

```bash
ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
ps -eo pid,user,%cpu,%mem,cmd --sort=-%mem | head
```

---

# 26. 커널 모듈과 sysctl

```bash
lsmod
modinfo br_netfilter

sudo modprobe br_netfilter
sudo modprobe -r br_netfilter

sysctl -a
sysctl net.ipv4.ip_forward

# 재부팅 전까지만 적용
sudo sysctl -w net.ipv4.ip_forward=1
```

파일: `/etc/sysctl.d/99-custom.conf`

```conf
# 일반 서버에서는 라우팅 비활성화
net.ipv4.ip_forward = 0

# SYN flood 대응
net.ipv4.tcp_syncookies = 1

# 잘못된 ICMP 오류 응답 무시
net.ipv4.icmp_ignore_bogus_error_responses = 1

# 링크 보호
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
```

```bash
sudo sysctl --system
```

부팅 시 모듈 로드:

```bash
echo 'br_netfilter' \
    | sudo tee /etc/modules-load.d/br_netfilter.conf
```

---

# 27. 커널, GRUB, 부팅

```bash
rpm -q kernel

grubby --default-kernel
grubby --default-index
grubby --info=ALL

cat /proc/cmdline
```

```bash
sudo grubby --update-kernel=ALL --args="audit=1"
sudo grubby --update-kernel=ALL --remove-args="audit=1"
```

## 부팅 분석

```bash
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain

journalctl -b -p err
journalctl -b -1 -p err
```

## 종료와 재부팅

```bash
shutdown -r now
shutdown -h now
shutdown -r +10 "10분 후 재부팅"
shutdown -c
```

> [!danger]
> 커널 또는 GRUB 인자 변경은 부팅 실패를 일으킬 수 있다. 원격 서버에서는 콘솔, 복구 커널, 백업 또는 스냅샷을 준비한다.

---

# 28. cron, at, systemd timer

## cron

```bash
crontab -l
crontab -e

sudo crontab -u rocky -l
sudo crontab -u rocky -e
```

cron 형식:

```text
분 시 일 월 요일 명령
0  2  *  *  *   /usr/local/sbin/backup.sh >> /var/log/backup.log 2>&1
```

파일: `/etc/cron.d/myjob`

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

# 매일 02:00에 root로 실행
0 2 * * * root /usr/local/sbin/backup.sh >> /var/log/backup.log 2>&1
```

```bash
sudo chmod 644 /etc/cron.d/myjob
```

> [!note]
> `/etc/cron.d/` 형식에는 실행 사용자 필드가 추가된다. cron 환경은 제한적이므로 명령어의 절대 경로를 사용하는 것이 좋다.

## at

```bash
sudo dnf install -y at
sudo systemctl enable --now atd

echo '/usr/local/sbin/job.sh' | at 23:00
atq
atrm <작업번호>
```

## systemd timer

파일: `/etc/systemd/system/mybackup.service`

```ini
[Unit]
Description=Backup job

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup.sh
```

파일: `/etc/systemd/system/mybackup.timer`

```ini
[Unit]
Description=Run backup daily

[Timer]
OnBootSec=10min
OnUnitActiveSec=24h

# 서버가 꺼져 있던 동안 놓친 작업을 부팅 후 실행
Persistent=true

# 실행 시간을 분산
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mybackup.timer

systemctl list-timers
journalctl -u mybackup.service
```

---

# 29. rsync와 백업

> [!note]
> `source/`는 source 디렉터리 내부를 복사하고, `source`는 source 디렉터리 자체를 대상 아래에 복사한다.

```bash
sudo dnf install -y rsync

rsync -avh source/ destination/
rsync -avh --progress source/ destination/

# 실제 삭제 전 dry-run
rsync -avhn --delete source/ destination/

# 대상의 불필요한 파일도 삭제
rsync -avh --delete source/ destination/

rsync -aHAX --numeric-ids source/ destination/

rsync -avh \
    -e ssh \
    /srv/data/ \
    user@server:/backup/data/
```

제외 패턴:

```bash
rsync -avh \
    --exclude='*.tmp' \
    --exclude='cache/' \
    source/ destination/
```

## 백업과 검증

```bash
tar -czf "/backup/etc-$(date +%F).tar.gz" /etc

sha256sum "/backup/etc-$(date +%F).tar.gz" \
    > "/backup/etc-$(date +%F).tar.gz.sha256"
```

```bash
sha256sum -c /backup/etc-<날짜>.tar.gz.sha256
tar -tzf /backup/etc-<날짜>.tar.gz | head
```

> [!important]
> 복구 테스트를 하지 않은 백업은 신뢰할 수 없다.

---

# 30. Podman

```bash
sudo dnf install -y podman

podman info
podman version
podman search nginx
podman pull docker.io/library/nginx:latest

podman images
podman ps
podman ps -a
```

## 컨테이너 실행

```bash
podman run --rm \
    docker.io/library/alpine:latest \
    echo hello
```

```bash
podman run -d \
    --name web \
    -p 8080:80 \
    docker.io/library/nginx:latest
```

```bash
podman logs web
podman logs -f web
podman inspect web
podman stats
podman exec -it web /bin/sh

podman stop web
podman start web
podman restart web
podman rm web
```

## 볼륨과 SELinux

```bash
podman volume create webdata
podman volume ls
```

```bash
# :Z는 해당 컨테이너 전용 SELinux 라벨
podman run -d \
    --name web \
    -p 8080:80 \
    -v /srv/web:/usr/share/nginx/html:Z \
    docker.io/library/nginx:latest
```

- `:Z`: 특정 컨테이너 전용 SELinux 라벨
- `:z`: 여러 컨테이너가 공유하는 SELinux 라벨

## Containerfile

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest

RUN microdnf install -y httpd \
    && microdnf clean all

COPY index.html /var/www/html/index.html

EXPOSE 80

CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

```bash
podman build -t localhost/myweb:1.0 .

podman run -d \
    --name myweb \
    -p 8080:80 \
    localhost/myweb:1.0
```

## 정리

```bash
podman system df

podman container prune
podman image prune
podman system prune
```

---

# 31. OpenSSL과 인증서

```bash
openssl version
openssl rand -hex 32
openssl dgst -sha256 file

openssl x509 -in cert.pem -text -noout
openssl x509 -in cert.pem -noout -subject -issuer -dates

openssl pkey -in private.key -check
openssl req -in request.csr -text -noout
```

서버 인증서 확인:

```bash
openssl s_client \
    -connect example.com:443 \
    -servername example.com \
    </dev/null
```

## RSA 개인키

```bash
openssl genpkey \
    -algorithm RSA \
    -pkeyopt rsa_keygen_bits:3072 \
    -out server.key

chmod 600 server.key
```

## CSR

```bash
openssl req \
    -new \
    -key server.key \
    -out server.csr
```

## 테스트용 자체 서명 인증서

```bash
openssl req \
    -x509 \
    -newkey rsa:3072 \
    -nodes \
    -keyout test.key \
    -out test.crt \
    -days 365 \
    -subj '/CN=test.example.com'

chmod 600 test.key
```

## 내부 CA 등록

```bash
sudo cp internal-ca.crt \
    /etc/pki/ca-trust/source/anchors/

sudo update-ca-trust
trust list | grep -i internal
```

---

# 32. 호스트명과 로케일

```bash
hostname
hostnamectl

sudo hostnamectl set-hostname server01.example.com
```

파일: `/etc/hosts`

```text
# IP           FQDN                   짧은 이름
192.0.2.10     server01.example.com   server01
```

```bash
localectl
localectl list-locales

sudo localectl set-locale LANG=ko_KR.UTF-8
sudo localectl set-keymap us
```

---

# 33. 로그인과 기본 보안 감사

```bash
last                                # 최근 로그인
lastlog                             # 사용자별 마지막 로그인
sudo lastb                          # 실패 로그인
who
w

faillock
sudo faillock --user rocky
sudo faillock --user rocky --reset

sudo ausearch -m USER_LOGIN -ts today
sudo ausearch -m AVC -ts recent

authselect current
authselect list
authselect check
```

## 파일 권한 점검

```bash
# 다른 사용자가 쓸 수 있는 홈 디렉터리 파일
find /home -xdev -type f -perm /002 -ls

# SUID/SGID 파일
find / -xdev -type f -perm /6000 -ls

# 소유자 또는 그룹이 없는 파일
find / -xdev \( -nouser -o -nogroup \) -ls
```

## 서비스 노출 점검

```bash
ss -lntup
sudo firewall-cmd --list-all
getenforce
systemctl --failed
```

---

# 34. 문제 해결 순서

## 서비스가 시작되지 않을 때

```bash
systemctl status <service>
journalctl -u <service> -b --no-pager
systemctl cat <service>
systemctl show <service>
sudo systemctl reset-failed <service>
```

## 서비스 포트에 접속할 수 없을 때

```bash
ss -lntp
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
getenforce
sudo ausearch -m AVC -ts recent
```

확인 순서:

1. 프로세스가 실행 중인지 확인한다.
2. 대상 주소와 포트에서 리스닝하는지 확인한다.
3. firewalld 규칙을 확인한다.
4. SELinux AVC 로그를 확인한다.
5. 애플리케이션 로그를 확인한다.

## DNS가 동작하지 않을 때

```bash
ip address
ip route

ping -c 2 1.1.1.1
getent hosts example.com
dig example.com

nmcli device show | grep -E 'DNS|DOMAIN'
cat /etc/resolv.conf
```

## 디스크가 찼을 때

```bash
df -hT
df -i

du -xhd1 /var | sort -h
journalctl --disk-usage

# 삭제됐지만 프로세스가 계속 열고 있는 파일
sudo lsof +L1
```

## 부팅 문제

```bash
journalctl -b -p err
journalctl -b -1 -p err

systemctl --failed
systemd-analyze critical-chain

sudo findmnt --verify --verbose
```

## CPU와 메모리 문제

```bash
uptime
free -h
vmstat 1 5

ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
ps -eo pid,user,%cpu,%mem,cmd --sort=-%mem | head
```

## 패키지 또는 설정 파일 문제

```bash
rpm -V <패키지>
rpm -qf <파일>
sudo dnf reinstall <패키지>
```

---

# 35. 안전한 설정 파일 수정 절차

## 1단계: 원본 백업

```bash
sudo cp -a \
    /etc/<설정파일> \
    "/etc/<설정파일>.$(date +%F_%H%M%S).bak"
```

## 2단계: 설정 편집

```bash
sudoedit /etc/<설정파일>
```

## 3단계: 문법 검사

```bash
sudo sshd -t
sudo visudo -cf /etc/sudoers
sudo nginx -t
sudo httpd -t
sudo named-checkconf

sudo systemd-analyze verify \
    /etc/systemd/system/<서비스>.service

sudo findmnt --verify --verbose
```

## 4단계: 설정 반영

가능하면 프로세스 중단이 없는 reload를 우선 사용한다.

```bash
sudo systemctl reload <서비스>
```

reload를 지원하지 않는 경우:

```bash
sudo systemctl restart <서비스>
```

## 5단계: 상태와 로그 확인

```bash
systemctl status <서비스>
journalctl -u <서비스> -n 100 --no-pager
```

---

# 36. 자주 사용하는 명령어 조합

## 현재 부팅의 오류 로그

```bash
journalctl -p err -b --no-pager
```

## 실패한 서비스

```bash
systemctl --failed
```

## CPU 사용량 상위 10개

```bash
ps -eo pid,user,%cpu,%mem,cmd \
    --sort=-%cpu \
    | head -n 11
```

## 메모리 사용량 상위 10개

```bash
ps -eo pid,user,%cpu,%mem,cmd \
    --sort=-%mem \
    | head -n 11
```

## 사용량 80% 이상 파일시스템

```bash
df -P \
    | awk 'NR>1 && $5+0 >= 80 {print $5,$6}'
```

## 100MiB 이상 파일

```bash
find /var \
    -xdev \
    -type f \
    -size +100M \
    -printf '%s %p\n' \
    | sort -n \
    | tail
```

## 설정 파일에서 문자열 검색

```bash
grep -Rni \
    --include='*.conf' \
    'Listen' \
    /etc
```

## 서비스 재시작 후 즉시 확인

```bash
sudo systemctl restart <서비스> \
    && systemctl --no-pager --full status <서비스>
```

## 명령어가 속한 패키지 확인

```bash
rpm -qf "$(command -v <명령어>)"
```

## 명령 성공 여부 처리

```bash
if command; then
    echo "성공"
else
    STATUS=$?
    echo "실패: $STATUS" >&2
fi
```

---

# 37. 최종 체크리스트

> [!check] 설정 변경 체크리스트
> - [ ] 현재 설정 파일을 백업했는가?
> - [ ] 백업 또는 스냅샷이 존재하는가?
> - [ ] 원격 서버라면 콘솔 접근 수단이 있는가?
> - [ ] 설정 파일의 문법 검사를 수행했는가?
> - [ ] 방화벽과 SELinux 영향을 확인했는가?
> - [ ] `restart` 전에 `reload` 지원 여부를 확인했는가?
> - [ ] 변경 후 서비스 상태와 로그를 확인했는가?
> - [ ] SSH 변경 후 기존 세션을 닫기 전에 새 세션 접속을 확인했는가?
> - [ ] 백업 데이터의 복구 테스트를 수행했는가?

> [!summary] 핵심 원칙
> 1. 백업 후 변경한다.
> 2. 명령을 실행하기 전에 대상 경로와 장치명을 확인한다.
> 3. 설정을 반영하기 전에 문법 검사를 수행한다.
> 4. SELinux를 끄기 전에 컨텍스트, boolean, 포트 타입과 AVC 로그를 확인한다.
> 5. 방화벽 영구 설정은 `--reload` 전까지 런타임에 적용되지 않는다.
> 6. systemd 원본 unit 대신 `systemctl edit`으로 override를 작성한다.
> 7. DNS는 `/etc/resolv.conf` 직접 수정 대신 NetworkManager 프로파일로 설정한다.
> 8. 스냅샷은 별도 백업을 대체하지 않는다.
> 9. 상세 옵션은 `man <명령어>`와 `<명령어> --help`로 확인한다.
