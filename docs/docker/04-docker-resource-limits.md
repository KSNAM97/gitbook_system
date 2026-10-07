# Docker - 컨테이너 리소스 제한

> **Tag:** #Docker #리소스제한 #메모리 #CPU #모니터링
> **핵심 요약:** 메모리/CPU 리소스 제한 옵션과 모니터링 방법 핵심 정리

---

## 1. 개요 (Overview)

컨테이너의 메모리는 `docker run` 시 `-m` 또는 `--memory` 옵션으로 제한한다. `-m` / `--memory`는 최대 메모리 사용량을 제한하고(`-m 512m`), `--memory-swap`은 메모리와 스왑을 합산하여 제한하며(`--memory-swap 1g`), `--memory-reservation`은 권고 수준의 소프트 제한을 걸고(`--memory-reservation 256m`), `--oom-kill-disable`은 OOM Killer를 비활성화한다(`--oom-kill-disable`).

```bash
# 메모리 512MB로 제한
docker run -d --name myapp -m 512m nginx:latest

# 메모리 512MB + 스왑 512MB (총 1GB)
docker run -d --name myapp -m 512m --memory-swap 1g nginx:latest

# 메모리 예약 (최솟값 보장)
docker run -d --name myapp -m 512m --memory-reservation 256m nginx:latest

# OOM Kill 비활성화 (주의: 시스템 불안정 가능)
docker run -d --name myapp -m 512m --oom-kill-disable nginx:latest
```

`--memory-swap` 옵션의 동작 방식도 정리해두면, 미설정 시에는 스왑이 메모리와 동일하게 설정되어 기본적으로 2배 효과를 가지며, `--memory-swap` 값이 `-m` 값과 같으면 스왑을 사용하지 않고(swap = 0), `--memory-swap -1`로 설정하면 스왑을 무제한으로 사용한다. 예를 들어 `-m 512m --memory-swap 1g`로 설정하면 메모리 512MB + 스왑 512MB로 총 1GB까지 사용 가능하다.

CPU 제한 옵션으로는 `--cpus`가 사용 가능한 CPU 개수를 제한하고(`--cpus 1.5`), `--cpuset-cpus`가 특정 CPU 코어를 지정하며(`--cpuset-cpus 0,1`), `--cpu-shares`(`-c`)가 CPU 상대적 비율을 지정한다(기본값 1024, `-c 512`).

```bash
# CPU 1.5개 제한
docker run -d --name myapp --cpus 1.5 nginx:latest

# 0번, 1번 CPU 코어만 사용
docker run -d --name myapp --cpuset-cpus 0,1 nginx:latest

# CPU 가중치 (다른 컨테이너 대비 절반만 사용)
docker run -d --name myapp -c 512 nginx:latest
```

컨테이너 리소스 사용량은 다음 방법으로 모니터링한다.

```bash
# 실시간 리소스 사용량 확인 (CPU, 메모리, 네트워크, I/O)
docker stats

# 특정 컨테이너만 확인
docker stats 컨테이너명

# 한 번만 출력 (스냅샷)
docker stats --no-stream

# Docker 이벤트 실시간 확인
docker events

# 특정 컨테이너 이벤트만 필터
docker events --filter container=컨테이너명
```

`docker stats`의 출력 항목은 CPU 사용률을 나타내는 CPU %, 메모리 사용량과 제한을 함께 보여주는 MEM USAGE / LIMIT, 메모리 사용률인 MEM %, 네트워크 입출력인 NET I/O, 디스크 입출력인 BLOCK I/O로 구성된다.

stress 도구로 부하 테스트를 하는 방법은 다음과 같다.

```bash
# stress 패키지 설치 후 사용
docker exec -it 컨테이너명 bash

# 컨테이너 내부에서
apt-get install -y stress

# CPU 2코어 부하 10초
stress --cpu 2 --timeout 10

# 메모리 256MB 부하 10초
stress --vm 1 --vm-bytes 256M --timeout 10
```

메모리 제한을 초과하면 OOM Killer가 동작하여 컨테이너 내 프로세스를 강제 종료한다.

```
컨테이너 메모리 초과
    ↓
Linux OOM Killer 발동
    ↓
컨테이너 내 프로세스 종료 (또는 컨테이너 전체 종료)
    ↓
docker events 로 oom 이벤트 확인 가능
```

---

>  **핵심 요약**
> - 메모리: `-m 512m` / `--memory-swap` / `--memory-reservation` / `--oom-kill-disable`
> - `--memory-swap == -m` → 스왑 사용 안 함, `--memory-swap -1` → 스왑 무제한
> - CPU: `--cpus`(개수 제한) / `--cpuset-cpus`(코어 지정) / `-c`(가중치)
> - 모니터링: `docker stats`(실시간) / `docker events`(이벤트 확인)
> - 메모리 초과 시 Linux OOM Killer 발동 → `docker events`로 oom 이벤트 확인
> - 관련: 3.  Docker - 컨테이너 사용하기 · 9.  Docker - 통합 정리 · 11.  Docker - 퀵 레퍼런스
