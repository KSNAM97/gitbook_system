# Docker - 통합 정리

> **Tag:** #Docker #통합정리 #요약 #부트캠프
> **핵심 요약:** Docker 전체 개념 흐름 요약 및 핵심 비교표 모음

---

## 1. 핵심 기술 개념 (Concept)

### Docker 전체 학습 흐름

```
1. 도커와 컨테이너의 이해
   └─ Bare Metal / VM / Container 비교
   └─ Image / Container 관계
   └─ Registry (Docker Hub)
   └─ Linux Namespaces / cgroups / UnionFS

2. 컨테이너와 이미지
   └─ Dockerfile 명령어 (FROM, RUN, COPY, CMD, ENTRYPOINT...)
   └─ 이미지 빌드 (docker build)
   └─ 이미지 레이어 구조

3. 컨테이너 사용하기
   └─ 생명주기 (pull→create→start→exec→stop→rm)
   └─ 주요 명령어 (run, exec, logs, inspect, top)

4. 리소스 제한
   └─ 메모리: -m, --memory-swap, --memory-reservation
   └─ CPU: --cpus, --cpuset-cpus, --cpu-shares
   └─ 모니터링: docker stats, docker events

5. 스토리지
   └─ Image Layer (읽기 전용)
   └─ Container Write Layer (휘발성)
   └─ Volume / Bind Mount (영구 보존)
   └─ :ro 옵션 (읽기 전용 마운트)

6. 네트워크
   └─ docker0 브릿지 (기본)
   └─ veth pair
   └─ 포트 포워딩 (-p, -P)
   └─ User-Defined Bridge (사용자 정의 네트워크)

7. YAML 문법
   └─ 스칼라 / 맵 / 리스트
   └─ 들여쓰기 규칙 (탭 금지)
   └─ 앵커(&) / 별칭(*) / <<: merge

8. Docker Compose
   └─ docker-compose.yaml 옵션
   └─ docker compose up/down/ps/logs
   └─ depends_on / healthcheck
   └─ 리버스 프록시 / 로드밸런싱
```

### 핵심 비교표: Bare Metal vs VM vs Container

| 구분 | Bare Metal | VM | Container |
|---|---|---|---|
| OS | 단일 OS | 각각 독립 OS | Host OS 공유 |
| 격리 수준 | 없음 | 강함 | 중간 |
| 기동 시간 | 빠름 | 느림 (분) | 매우 빠름 (초) |
| 리소스 효율 | 낮음 | 낮음 | 높음 |
| 이식성 | 낮음 | 낮음 | 높음 |

### Image vs Container 핵심 비교

| 항목 | Image | Container |
|---|---|---|
| 상태 | 정적 (파일) | 동적 (실행 중) |
| 수정 | 불가 (Read-Only) | 가능 (Write Layer) |
| 삭제 | 데이터 유지 | 데이터 사라짐 |
| 공유 | 여러 컨테이너가 공유 | 각각 독립 |

### Dockerfile 명령어 요약

| 명령어 | 레이어 생성 | 역할 |
|---|---|---|
| FROM |  | 베이스 이미지 |
| RUN |  | 빌드 시 명령 실행 |
| COPY |  | 파일 복사 |
| ADD |  | 파일 복사 + URL/압축 지원 |
| WORKDIR | — | 작업 디렉터리 (레이어 영향 미미) |
| ENV | — | 환경 변수 (레이어 영향 미미) |
| EXPOSE |  | 포트 문서화 |
| VOLUME |  | 마운트 포인트 |
| USER |  | 실행 사용자 |
| CMD |  | 기본 실행 명령 (덮어쓰기 가능) |
| ENTRYPOINT |  | 고정 실행 명령 |
| HEALTHCHECK |  | 상태 확인 |
| LABEL |  | 메타데이터 |

> 소스에서 명시: "FROM, RUN, COPY, ADD 등이 새로운 레이어를 생성한다"

### 컨테이너 생명주기 요약

```
pull → create → start → pause ↔ unpause
                   ↓
                  exec
                   ↓
              stop / kill → rm
```

### 리소스 제한 옵션 요약

| 항목 | 옵션 | 예시 |
|---|---|---|
| 메모리 제한 | `-m` / `--memory` | `-m 512m` |
| 메모리+스왑 제한 | `--memory-swap` | `--memory-swap 1g` |
| 메모리 예약 | `--memory-reservation` | `--memory-reservation 256m` |
| CPU 개수 제한 | `--cpus` | `--cpus 1.5` |
| CPU 코어 지정 | `--cpuset-cpus` | `--cpuset-cpus 0,1` |
| CPU 가중치 | `-c` / `--cpu-shares` | `-c 512` |

### 스토리지 저장 방식 비교

| 방식 | 데이터 유지 | 속도 | 용도 |
|---|---|---|---|
| Container Write Layer | 컨테이너 삭제 시 사라짐 | 느림 | 임시 파일, 로그 |
| Bind Mount (`-v /host:/container`) | 호스트에 유지 | 빠름 | 개발용, 설정 파일 |
| Named Volume (`-v name:/container`) | 볼륨 삭제 전까지 유지 | 빠름 | DB, 영구 데이터 |

### 네트워크 방식 비교

| 방식 | 설명 | DNS 지원 |
|---|---|---|
| 기본 bridge (docker0) | Docker 자동 생성, 172.17.x.x |  |
| User-Defined Bridge | 사용자 생성, DNS 자동 지원 |  |
| host | 호스트 네트워크 직접 사용 | — |
| none | 네트워크 없음 | — |

### -p 포트 옵션 비교

| 옵션 | 설명 |
|---|---|
| `-p 8080:80` | 호스트 8080 → 컨테이너 80 직접 연결 |
| `-p 80` | 호스트 랜덤 포트 → 컨테이너 80 |
| `-P` | Dockerfile EXPOSE 포트 자동 매핑 |

### Docker Compose 핵심 옵션 요약

| 옵션 | 역할 |
|---|---|
| `services` | 컨테이너 정의 영역 |
| `build` | Dockerfile로 이미지 빌드 |
| `image` | 기존 이미지 사용 |
| `ports` | 포트 포워딩 |
| `volumes` | 볼륨 마운트 |
| `environment` | 환경 변수 |
| `restart` | 재시작 정책 |
| `depends_on` | 실행 순서 제어 |
| `networks` | 사용자 정의 네트워크 |

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Docker Engine (Compose V2 내장) 설치 환경.

### Docker 설치 요약 (Rocky Linux)

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

---

>  **핵심 요약**
> - Docker = 컨테이너 기반 플랫폼: Image(템플릿) → Container(실행 인스턴스)
> - 레이어 생성 명령: FROM / RUN / COPY / ADD → 이미지 크기에 영향
> - 영구 데이터: Container Write Layer(휘발) vs Volume/Bind Mount(영구)
> - 네트워크: 기본 bridge(이름 통신 불가) vs User-Defined(DNS 자동 지원)
> - Docker Compose: 멀티 컨테이너를 단일 YAML로 관리, depends_on으로 순서 제어
> - 관련: 1.  Docker - 도커와 컨테이너의 이해 · 8.  Docker - Docker Compose · 11.  Docker - 퀵 레퍼런스
