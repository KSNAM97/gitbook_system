# Docker - YAML 문법

> **Tag:** #Docker #YAML #문법 #DockerCompose #부트캠프
> **핵심 요약:** Docker Compose 작성의 기초, YAML 데이터 표현 방식 핵심 정리

---

## 1. 개요 (Overview)

YAML(YAML Ain't Markup Language)은 사람이 읽기 쉽게 만든 데이터 표현 형식이다. Docker Compose(docker-compose.yml), Kubernetes(deployment.yaml), Ansible(site.yml), GitHub Actions(.github/workflows/*.yml) 등 다양한 인프라 도구에서 공통 설정 언어로 사용된다.

YAML의 핵심 규칙은 세 가지다. 첫째, 들여쓰기로 구조를 표현한다.
```yaml
server:
  host: localhost    # server의 자식
  port: 8080         # server의 자식
```
둘째, 탭(Tab) 사용을 금지하고 공백(Space)만 사용해야 한다. 탭이 섞이면 파서 에러가 발생하므로 공백 2칸 또는 4칸으로 통일한다. 셋째, `key: value` 형식으로 작성한다.
```yaml
name: docker-study
version: "1.0"
```

YAML의 데이터 타입은 세 가지로 나뉜다. 스칼라(scalar)는 문자열, 숫자, 불리언, null 등 단일 값을 나타내고(`port: 8080`), 맵(map)은 key: value 묶음이며(`server: { host: localhost }`), 리스트(list)는 순서 있는 값 목록이다(`- apple`).

스칼라 타입별 예시는 다음과 같다.

```yaml
# 문자열 (따옴표 생략 가능)
message: hello world
user: guest
path: /var/www/html

# 특수문자 포함 시 따옴표 필요
title: "YAML: beginner guide"

# 숫자
port: 8080
pi_approx: 3.14

# 불리언
debug: true
enabled: false

# null (세 가지 표현 모두 동일)
description: null
another: ~
empty:
```

맵(Map) 작성법은 다음과 같다.

```yaml
# 여러 줄 (실무에서 주로 사용)
server:
  host: localhost
  port: 8080
  debug: true

# 한 줄로 작성 (거의 안 씀)
server: { host: localhost, port: 8080, debug: true }
```

리스트(List) 작성법은 다음과 같다.

```yaml
# 여러 줄 (하이픈 -)
fruits:
  - apple
  - banana
  - orange

# 한 줄
fruits: [apple, banana, orange]

# 맵을 포함한 리스트
services:
  - name: web
    port: 80
  - name: db
    port: 3306
```

여러 줄 문자열을 작성하는 방법은 두 가지 스타일이 있다.

#### 리터럴 스타일 (`|`) — 줄바꿈 그대로 유지
```yaml
description: |
  이 서비스는 테스트용 웹 서버입니다.
  두 번째 줄입니다.
  세 번째 줄입니다.
```
→ 실제 값: `이 서비스는...\n두 번째 줄...\n세 번째 줄...`

#### 접힌 스타일 (`>`) — 줄바꿈을 공백으로 합침
```yaml
message: >
  이 문장은 YAML 파일에서
  여러 줄로 써 있지만,
  실제로는 한 줄입니다.
```
→ 실제 값: `이 문장은 YAML 파일에서 여러 줄로 써 있지만, 실제로는 한 줄입니다.`

주석은 다음과 같이 작성한다.

```yaml
server:
  host: localhost    # 개발 환경에서는 localhost 사용
  port: 8080         # 기본 포트

# 이 줄은 전체 주석
```

앵커(&)와 별칭(*)은 반복되는 설정을 재사용하기 위한 기능이다. `&name`은 앵커로 특정 블록에 이름을 붙이고, `*name`은 별칭으로 앵커 내용을 가져오며, `<<: *name`은 앵커 내용을 현재 위치에 합친다(merge).

```yaml
# 공통 환경변수 정의
default-env: &common-env
  DB_HOST: db.example.com
  DB_PORT: 3306

# 재사용
dev:
  environment:
    <<: *common-env    # DB_HOST, DB_PORT 자동 포함
    MODE: dev

prod:
  environment:
    <<: *common-env
    MODE: prod
```

**실제 해석 결과:**
```yaml
dev:
  environment:
    DB_HOST: db.example.com
    DB_PORT: 3306
    MODE: dev

prod:
  environment:
    DB_HOST: db.example.com
    DB_PORT: 3306
    MODE: prod
```

---

>  **핵심 요약**
> - YAML = 사람이 읽기 쉬운 데이터 표현 형식, 탭 금지(공백만 사용)
> - 3가지 타입: 스칼라(단일 값) / 맵(key:value) / 리스트(- 항목)
> - `|`: 줄바꿈 그대로 유지 / `>`: 줄바꿈을 공백으로 합침
> - `&앵커` / `*별칭` / `<<: *merge` — 반복 설정 재사용
> - Docker Compose, Kubernetes, Ansible 등 인프라 도구의 공통 설정 언어
> - 관련: 8.  Docker - Docker Compose · 9.  Docker - 통합 정리
