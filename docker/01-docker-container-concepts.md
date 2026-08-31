# Docker - 도커와 컨테이너의 이해

> **Tag:** #Docker #컨테이너 #개념 #부트캠프
> **핵심 요약:** Docker의 등장 배경, 컨테이너 개념, Image/Container 관계, 격리 기술 핵심 정리

---

## 1. 개요 (Overview)

Docker는 애플리케이션을 컨테이너(Container) 단위로 패키징하여 어디서든 동일하게 실행할 수 있도록 해주는 오픈소스 플랫폼이다. 개발사는 Docker Inc.이며, 핵심 개념은 컨테이너 기반 가상화이고, OCI(Open Container Initiative) 표준을 따른다. 이를 이해하는 데는 해상 컨테이너 비유가 유용한데, 배(선박)는 서버(운영 환경)에, 컨테이너는 소프트웨어 실행 단위에 해당한다. 컨테이너 규격이 통일되어 어느 배에나 실을 수 있듯, Docker 컨테이너도 규격이 통일되어 있어 어느 서버에서나 실행 가능하다.

전통적인 서버 방식과 비교하면 Docker의 차이가 뚜렷하다. 전통 방식은 서버마다 환경을 직접 설치해야 하는 반면 Docker는 이미지로 패키징한다. 이식성 면에서 전통 방식은 환경에 의존해 낮지만 Docker는 어디서나 동일하게 동작해 높다. 배포 속도는 전통 방식이 느리고 Docker는 빠르며, 리소스 사용은 전통 방식이 OS 전체를 필요로 해 무겁지만 Docker는 OS를 공유해 가볍다. 격리 수준에서도 전통 방식은 프로세스 수준 격리가 불가능한 반면 Docker는 컨테이너 수준 격리를 제공한다.

이를 더 넓은 맥락에서 Bare Metal / VM / Container로 비교할 수 있다. Bare Metal은 물리 서버를 직접 사용하는 방식으로 단일 OS를 쓰고 격리가 없으며 기동 시간이 빠르지만 리소스 효율은 낮다. VM은 하이퍼바이저로 OS를 가상화하는 방식으로 각 VM마다 독립된 OS를 가지며, 하드웨어 수준의 강한 격리를 제공하지만 기동 시간이 분 단위로 느리고 리소스 효율도 낮다. 대표 도구로는 VMware, KVM이 있다. Container는 OS를 공유하면서 프로세스 수준으로 격리하는 방식으로, Host OS를 공유하고 중간 수준의 격리를 가지며 기동 시간이 초 단위로 매우 빠르고 리소스 효율이 높다. 대표 도구는 Docker, Podman이다.

Docker Image와 Container의 관계를 보면, Image는 읽기 전용 템플릿이고 Container는 Image를 실행한 인스턴스이다. 흐름은 다음과 같다.

```
Docker Image (읽기 전용)
    ↓  docker run
Docker Container (실행 중인 인스턴스)
    = Image + 쓰기 가능한 레이어(Write Layer)
```

Image는 정적인 파일 상태이며 수정이 불가능한 Read-Only이고 여러 컨테이너가 공유할 수 있으며, 삭제 시에도 데이터가 유지된다. 반면 Container는 동적으로 실행 중인 상태이며 Write Layer를 통해 수정이 가능하고 각각 독립적이며, 삭제 시 데이터가 사라진다.

Docker Image Layer 구조를 보면, 이미지는 여러 개의 읽기 전용 레이어(Layer)가 쌓여 만들어진다. 예를 들어 Dockerfile에서 `FROM ubuntu`는 Layer 1(Base)을, `RUN apt update`는 Layer 2를, `RUN apt install`은 Layer 3을, `COPY app.py /app`은 Layer 4(최상단)를 생성한다. 각 Layer는 읽기 전용이며, 중복되는 레이어는 재사용되어 다운로드가 생략되고, 여러 이미지가 동일한 레이어를 공유할 수 있다.

Docker Registry는 Docker Image를 저장하고 배포하는 저장소이다. Docker Hub는 공식 퍼블릭 레지스트리(hub.docker.com)이고, Private Registry는 사내에서 자체 구축하는 레지스트리이며, ECR/GCR/ACR은 클라우드 제공업체가 제공하는 레지스트리다. 관련 명령어로는 레지스트리에서 이미지를 가져오는 `docker pull nginx:latest`, 레지스트리에 이미지를 올리는 `docker push myapp:1.0`, 이미지를 검색하는 `docker search nginx`가 있다.

마지막으로 컨테이너 격리 기술은 Linux 커널의 두 가지 기술로 구현된다. Linux Namespaces는 프로세스, 네트워크, 파일시스템 등을 격리하고, cgroups(Control Groups)는 CPU, 메모리 등 리소스 사용량을 제한하며, UnionFS는 여러 레이어를 하나의 파일시스템처럼 마운트한다. Namespace의 종류로는 프로세스를 격리하는 PID, 네트워크를 격리하는 NET, 파일시스템 마운트를 격리하는 MNT, 호스트명을 격리하는 UTS, 프로세스 간 통신을 격리하는 IPC, 사용자 ID를 격리하는 USER가 있다.

---

>  **핵심 요약**
> - Docker = 컨테이너 기반 오픈소스 플랫폼, OCI 표준 준수
> - Image(읽기 전용 템플릿) → docker run → Container(실행 인스턴스)
> - 이미지는 레이어 구조, 중복 레이어 재사용으로 효율적
> - 격리 기술: Linux Namespaces(프로세스/네트워크 격리) + cgroups(리소스 제한)
> - Container는 VM보다 가볍고(OS 공유), 기동 시간이 매우 빠름(초 단위)
> - 관련: 2.  Docker - 컨테이너와 이미지 · 3.  Docker - 컨테이너 사용하기 · 9.  Docker - 통합 정리
