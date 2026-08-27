# 허가권·소유권 통합 정리

Linux 접근 권한은 프로세스의 UID·GID, 상위 경로의 탐색 권한, 객체 소유권, Owner·Group·Other 권한, 특수 권한, ACL, SELinux 및 마운트 상태를 종합하여 판정한다.

## 목차

1. [개요 (Overview)](#개요-overview)
2. [표준 구성 예시 (Configuration)](#표준-구성-예시-configuration)
3. [검증 및 대표 함정](#검증-및-대표-함정)

---

## 개요 (Overview)

**권한 문제는 어떤 순서로 판정하는가?**

```text
현재 프로세스 UID/GID 확인
→ 상위 경로의 x 권한 확인
→ Owner·Group·Other 범주 선택
→ rwx 권한 적용
→ Set-UID·Set-GID·Sticky 확인
→ ACL 확인
→ SELinux 확인
→ 마운트 및 파일시스템 상태 확인
```

**파일과 디렉터리 권한의 차이는?**

| 권한 | 파일 | 디렉터리 |
|---|---|---|
| `r` | 내용 읽기 | 이름 목록 조회 |
| `w` | 내용 수정 | 생성·삭제·이름 변경 |
| `x` | 실행 | 경로 탐색·접근 |

**특수 권한은 언제 사용하는가?**

| 목적 | 권한 |
|---|---|
| 실행 파일 소유자 권한으로 실행 | Set-UID |
| 공유 디렉터리 그룹 상속 | Set-GID |
| 타인 파일 삭제·이름 변경 제한 | Sticky-bit |

## 표준 구성 예시 (Configuration)

개인 디렉터리:

```bash
mkdir /private
chmod 700 /private
```

팀 공유:

```bash
groupadd project
mkdir /project
chown root:project /project
chmod 2770 /project
```

팀 공유 + 삭제 보호:

```bash
chmod 3770 /project
```

공개 임시 공유:

```bash
mkdir /public-tmp
chown root:root /public-tmp
chmod 1777 /public-tmp
```

파일 권한:

```bash
chmod 600 private.key
chmod 640 config.conf
chmod 644 document.txt
chmod 750 admin-script
```

소유권:

```bash
chown user1:project file
chgrp project file
```

---

## 검증 및 대표 함정

필수 명령:

```bash
id
namei -l /전체/경로
stat <경로>
getfacl <경로>
ls -lZ <경로>
findmnt -T <경로>
```

대표 함정:

| 함정 | 결과 | 올바른 접근 |
|---|---|---|
| `chmod 777` 남용 | 타인 파일 삭제·변조 가능 | 그룹·Sticky 설계 |
| 파일 `w`만 보고 삭제 판단 | 잘못된 분석 | 상위 디렉터리 `w+x` 확인 |
| Set-GID만 설정 | 그룹 쓰기 불가 가능 | umask·ACL 확인 |
| 그룹 추가 후 즉시 테스트 | 기존 세션 미반영 | 재로그인 |
| `chown -R` 무분별 사용 | 소유권·특수 권한 손상 | 범위 사전 확인 |
| root면 항상 접근 가능하다고 판단 | RO·SELinux 등 누락 | 전체 계층 확인 |

## 요약

- 권한은 UID/GID 숫자로 판정
- 디렉터리 삭제는 상위 디렉터리 권한이 핵심
- 공유 그룹 상속은 Set-GID
- 타인 파일 삭제 방지는 Sticky-bit
- 기본 생성 권한은 umask
- 관련: **허가권 (Permission) — chmod & rwx·UGO 모델** · **소유권 (Ownership) — chown & UID·GID 소유 모델** · **Umask — 기본 권한 마스크 (User Mask)** · **허가권·소유권 트러블슈팅 치트시트**
