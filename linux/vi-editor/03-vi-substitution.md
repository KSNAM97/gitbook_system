# 🔁 VI 치환

> **Tag:** #Linux #Vi #Vim #Substitute #Regex #Refactoring  
> **핵심 요약:** VI/Vim의 `:substitute` 명령은 범위, 검색 패턴, 치환 문자열, 플래그를 조합한다. 대량 치환 전에는 백업과 매칭 개수 확인을 수행하고, 운영 파일에서는 `c` 플래그로 각 변경을 확인한다.

---

## 1. 📖 개요 (Overview)

치환 명령의 기본 구조는 다음과 같다.

```vim
:[범위]s/검색패턴/치환문자열/[플래그]
```

예시:

```vim
:%s/linux/soldesk/gc
```

이 예시의 구성은 다음과 같다.

```text
%         문서 전체
s         substitute
linux     검색 패턴
soldesk   치환 문자열
g         각 줄의 모든 일치
c         일치 항목마다 확인
```

범위를 생략하면 현재 커서가 있는 줄만 처리한다.

```vim
:s/linux/soldesk/
```

`g`가 없으면 현재 줄의 첫 번째 일치만 치환한다. 현재 줄의 모든 일치를 치환하려면 다음과 같이 한다.

```vim
:s/linux/soldesk/g
```

Vim 치환과 `sed -i`의 차이는, Vim은 편집 화면에서 결과를 확인할 수 있고 `c` 플래그로 각 치환을 승인할 수 있으며 같은 세션에서 `u`로 실행 취소가 가능하다는 점이다. 반면 `sed -i`는 비대화형 자동화에 적합하며, 실행 전 백업 옵션을 사용하는 것이 안전하다.

```bash
sed -i.bak 's/old/new/g' file
```

---

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

### 2-1. 범위

| 범위 | 의미 |
|---|---|
| 생략 | 현재 줄 |
| `.` | 현재 줄 |
| `$` | 마지막 줄 |
| `%` | 전체 문서 |
| `6` | 6번째 줄 |
| `2,7` | 2~7번째 줄 |
| `.,+5` | 현재 줄부터 아래 5줄 |
| `.,$` | 현재 줄부터 마지막 줄 |

### 2-2. 플래그

| 플래그 | 의미 |
|---|---|
| `g` | 각 줄의 모든 일치 치환 |
| `i` | 대소문자 무시 |
| `c` | 일치할 때마다 확인 |
| `n` | 변경하지 않고 일치 개수 표시 |

확인 프롬프트의 대표 키:

```text
y  현재 항목 변경
n  현재 항목 건너뜀
a  나머지 모두 변경
q  종료
l  현재 항목만 변경하고 종료
```

### 2-3. 기본 치환

현재 줄의 첫 번째 `linux`:

```vim
:s/linux/soldesk/
```

6번째 줄의 첫 번째 `linux`:

```vim
:6s/linux/soldesk/
```

2~7번째 줄에서 줄마다 첫 번째 `linux`:

```vim
:2,7s/linux/soldesk/
```

6번째 줄의 모든 `linux`:

```vim
:6s/linux/CISCO/g
```

2~7번째 줄의 모든 `linux`:

```vim
:2,7s/linux/soldesk/g
```

문서 전체에서 줄마다 첫 번째 `linux`:

```vim
:%s/linux/WIN/
```

문서 전체의 모든 `linux`:

```vim
:%s/linux/WIN/g
```

대소문자를 무시하고 모두 치환:

```vim
:%s/linux/HELLO/gi
```

확인하면서 치환:

```vim
:%s/linux/HELLO/gc
```

### 2-4. 줄 시작과 줄 끝

줄 시작이 `linux`인 경우:

```vim
:%s/^linux/soldesk/
```

대소문자를 무시하려면:

```vim
:%s/^linux/soldesk/i
```

줄 끝이 `linux`인 경우:

```vim
:%s/linux$/soldesk/
```

줄 끝의 공백까지 고려:

```vim
:%s/linux\s*$/soldesk/
```

### 2-5. 정확한 단어 치환

`linux`라는 독립된 단어만 치환:

```vim
:%s/\<linux\>/WindowS/g
```

대소문자를 무시하고 `Linux`, `linux` 모두 처리:

```vim
:%s/\<linux\>/WindowS/gi
```

다음 문자열은 기본적으로 제외된다.

```text
selinux
linuxOS
mylinux
```

> Vim의 `\<`, `\>`는 `'iskeyword'` 설정을 기준으로 단어 경계를 판단한다.

확인:

```vim
:set iskeyword?
```

### 2-6. 주석 처리

줄 시작의 `#` 하나 제거:

```vim
:%s/^#//
```

줄 시작에 `#` 추가:

```vim
:%s/^/#/
```

들여쓰기 뒤의 `#` 제거:

```vim
:%s/^\(\s*\)#/\1/
```

### 2-7. 빈 줄과 공백

완전히 빈 줄 삭제:

```vim
:g/^$/d
```

공백만 있는 줄도 삭제:

```vim
:g/^\s*$/d
```

공백만 있는 줄의 공백을 제거하여 빈 줄로 만들기:

```vim
:%s/^\s*$//
```

> `:%s/^\s*$//`는 줄을 삭제하지 않고 줄 안의 공백만 제거한다.

줄 끝 공백 제거:

```vim
:%s/\s\+$//
```

연속된 공백을 하나로 변경:

```vim
:%s/ \+/ /g
```

### 2-8. IP 주소 치환

점(`.`)은 정규식에서 임의의 한 문자를 의미하므로 리터럴 점은 `\.`로 작성한다.

```vim
:%s/192\.168\.1/172.16.100/g
```

확인하면서 변경:

```vim
:%s/192\.168\.1/172.16.100/gc
```

마지막 호스트 번호를 유지하는 캡처 예시:

```vim
:%s/192\.168\.1\.\([0-9]\+\)/172.16.100.\1/gc
```

예시 결과:

```text
192.168.1.251 → 172.16.100.251
192.168.1.2   → 172.16.100.2
```

### 2-9. 캡처 그룹과 전체 일치

순서 변경:

```vim
:%s/\(user\)_\(name\)/\2_\1/g
```

```text
user_name → name_user
```

전체 일치 문자열은 `&`로 참조한다.

```vim
:%s/\<error\>/[ERROR: &]/g
```

### 2-10. 구분자 변경

경로처럼 `/`가 많은 문자열은 다른 구분자를 사용할 수 있다.

```vim
:%s#/old/path#/new/path#g
```

또는:

```vim
:%s|/old/path|/new/path|g
```

### 2-11. 검색

```vim
/pattern
?pattern
n
N
*
#
:nohlsearch
```

- `/pattern`: 아래 방향
- `?pattern`: 위 방향
- `n`: 같은 검색 방향의 다음 결과
- `N`: 반대 방향의 결과
- `*`: 커서 아래 단어를 아래 방향으로 검색
- `#`: 커서 아래 단어를 위 방향으로 검색

### 2-12. 치환 전 개수 확인

문서 전체의 `linux` 일치 개수:

```vim
:%s/linux//gn
```

대소문자 무시:

```vim
:%s/linux//gin
```

정확한 단어 개수:

```vim
:%s/\<linux\>//gn
```

> `n` 플래그가 있으면 실제 치환은 수행하지 않고 일치 개수만 보고한다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 안전 치환 절차

현재 버퍼를 백업 파일로 저장:

```vim
:w /tmp/OS.bak
```

매칭 개수 확인:

```vim
:%s/\<linux\>//gin
```

각 항목을 확인하며 치환:

```vim
:%s/\<linux\>/WindowS/gic
```

현재 버퍼와 백업을 비교:

```vim
:w !diff -u /tmp/OS.bak -
```

> 위 명령은 현재 버퍼 내용을 `diff`의 표준 입력으로 전달하므로 아직 원본 파일에 저장하지 않은 변경도 비교할 수 있다.

이상이 없으면 저장:

```vim
:w
```

### 3-2. 실습 파일

```text
Operating System
OS = unixos , linuxos , windowsos
unix = cisco IOS , linux , android
Windows = windows xp , windows 7 , windows 8 , windows 10
cisco IOS 12.4 version
Red Hat Linux = Fedora linux , CentOS linux , RedHat linux
Debian Linux = Ubuntu linux , kali linux
Linux centos 7 = Linux RedHat 7
selinux = only CentOS
selinux = off
linux download = linuxOS
windows10 enterprise downloads
ens160
IPADDR="192.168.1.251"
NETMASK="255.255.255.0"
GATEWAY="192.168.1.2"
DNS1="192.168.1.2"
```

### 3-3. 대표 트러블슈팅

#### 🚨 시나리오 1. `selinux`, `linuxOS`까지 변경되었다

실행 취소:

```vim
u
```

독립된 단어만 변경:

```vim
:%s/\<linux\>/WIN/gc
```

`Linux`도 포함:

```vim
:%s/\<linux\>/WIN/gic
```

#### 🚨 시나리오 2. IP 치환에서 예상하지 않은 문자열까지 변경되었다

잘못된 패턴:

```vim
:%s/192.168.1.1/10.0.0.1/g
```

정규식에서 점이 임의의 한 문자로 처리된다.

수정:

```vim
:%s/192\.168\.1\.1/10.0.0.1/gc
```

더 엄격한 경계를 사용하려면:

```vim
:%s/\(^\|[^0-9.]\)\zs192\.168\.1\.1\ze\([^0-9.]\|$\)/10.0.0.1/gc
```

### 3-4. 치환 후 저장했는데 결과가 잘못되었다

같은 Vim 세션에 Undo 기록이 있으면:

```vim
u
:w
```

저장 전 백업과 현재 버퍼 비교:

```vim
:w !diff -u /tmp/OS.bak -
```

세션 종료 후에는 백업, 버전 관리, persistent undo 등을 확인한다.

> 📌 **핵심 요약**
> - 현재 줄: `:s/old/new/`
> - 전체: `:%s/old/new/g`
> - 확인: `c`
> - 대소문자 무시: `i`
> - 개수 확인: `n`
> - 정확한 단어: `\<word\>`
> - 리터럴 점: `\.`
> - 관련: VI 3-Mode 아키텍처 & 파일 열기 · VI 편집 명령어 (커서·삭제·복사·Ex Mode 범위 조작) · VI Shell 연동 & Swap 파일 복구
