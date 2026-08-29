# 🏗️ Kubernetes - 아키텍처 개요와 핵심 컴포넌트

> **Tag:** #Kubernetes #Architecture #ControlPlane #WorkerNode #부트캠프
> **핵심 요약:** Control Plane과 Worker Node를 구성하는 핵심 컴포넌트(API Server, etcd, Scheduler, Controller Manager, kubelet, kube-proxy, Container Runtime)의 역할과 Pod 생성 전체 흐름(0단계~6단계) 정리

---

## 1. 📖 개요 (Overview)

쿠버네티스 클러스터는 크게 **Control Plane(컨트롤 플레인)** 과 **Worker Node(워커 노드)** 로 나뉜다.

- **마스터(Control Plane) 컴포넌트** — 클러스터 전체를 관리하고 무엇을, 어떻게 실행할지 결정하는 두뇌 역할을 한다. 클러스터의 원하는 상태(Desired State)를 관리하고 실제 상태(Current State)가 원하는 상태와 일치하도록 조정한다.
- **워커 노드(Node) 컴포넌트** — 실제 컨테이너(Pod)를 실행하는 현장 역할을 한다.

개발자는 `kubectl`이나 API 요청을 통해 원하는 상태만 선언하고, 쿠버네티스는 이 상태를 계속 유지하도록 자동으로 동작한다.

---

## 2. 🏗️ 핵심 컴포넌트와 동작 흐름

### 마스터(Control Plane) 컴포넌트

**1) kube-api server**

- 쿠버네티스의 모든 요청이 반드시 거치는 중앙 관문
- `kubectl` 요청 수신
- REST API 형태로 요청 처리
- 인증(Authentication)
- 권한 확인(Authorization)
- 요청 유효성 검사(Validation)
- 최종적으로 etcd에 상태 저장
- 다른 컴포넌트들은 직접 통신하지 않고 API Server를 통해서만 통신한다(쿠버네티스의 단일 진입점).

**2) etcd**

etcd에는 다음과 같은 정보가 저장된다.

- Pod 정보
- Deployment 정보
- ReplicaSet 정보
- Service 정보
- ConfigMap 정보
- Secret 정보
- Node 정보
- Namespace 정보
- 각 리소스의 현재 상태(Current State)
- 사용자가 선언한 원하는 상태(Desired State)

**3) kube-scheduler**

- 새로 생성될 Pod를 어떤 Worker Node에 배치할지 결정하는 컴포넌트
- Pod를 직접 실행하지 않는다.
- 적절한 Node를 선택한 후 해당 정보를 API Server에 반영한다.

판단 기준:
- Node의 CPU / Memory 여유
- Pod의 resource request/limit
- nodeSelector, affinity/Anti-Affinity, taint/toleration
- 이미 실행 중인 Pod 분포

kube-scheduler는 실제로 Pod를 실행하지는 않는다. "Pod는 Node A 또는 Node B에서 실행해라"라고 결정만 한다.

> 예) 회사에서 업무를 직원에게 배정하는 관리자 역할

**4) kube-controller-manager**

- 클러스터의 현재 상태(Current State)를 지속적으로 감시하고 사용자가 선언한 원하는 상태(Desired State)와 비교 후 두 상태가 다르면 원하는 상태가 되도록 자동으로 조정한다.

대표적인 컨트롤러들:
- Deployment Controller
- ReplicaSet Controller
- DaemonSet Controller
- Node Controller
- Job Controller / CronJob Controller

동작 개념:
1. 현재 상태(current state) 확인
2. 원하는 상태(desired state)와 비교
3. 다르면 자동으로 조치

예시: Pod 3개가 필요하다고 선언 → 실제 Pod가 2개만 실행 중 → 현재 상태와 원하는 상태를 비교해서 새 Pod 1개 생성

### 워커 노드(Node) 컴포넌트

실제 Pod와 컨테이너가 실행되는 작업 Node로, Control Plane이 결정한 내용을 실제로 수행하는 역할이다.

**1) kubelet**

- 각 Worker Node에서 실행되는 Kubernetes 에이전트
- 각 Worker Node마다 하나씩 실행되며, 백그라운드에서 지속적으로 동작하는 데몬 형태의 프로세스

주요 역할:
- API Server와 통신하여 자신의 Node에 배정된 Pod 정보를 확인
- API Server의 리소스 변경 사항을 Watch 방식으로 감시 (Watch : "변경사항이 생기면 나한테 알려줘"라고 API Server에 등록해 놓는 방식)
- 자신에게 할당된 Pod가 정상적으로 실행되고 있는지 관리
- Container Runtime에 컨테이너의 생성·실행·중지 등의 작업을 요청
- 컨테이너와 Pod의 상태를 지속적으로 확인
- Pod 및 Node의 상태 정보를 API Server에 보고

Pod를 실제로 실행하는 것은 kubelet 자체가 아니라 Container Runtime이다.

**2) kube-proxy**

- Kubernetes의 Service 네트워크 기능을 구현하는 데 관여한다.
- Service로 들어온 트래픽을 적절한 Pod로 전달할 수 있도록 Node의 네트워크 규칙을 관리한다.
  - Service 동작 구현
  - iptables / ipvs 규칙 설정
  - Pod 간 통신, 로드밸런싱 지원

예시) Service IP로 요청 : kube-proxy가 규칙에 따라 여러 Pod 중 하나로 트래픽 전달

핵심 포인트:
- L4 레벨 트래픽 제어
- 쿠버네티스 내부 네트워크의 핵심

**3) Container Runtime**

- 실제로 컨테이너를 실행하는 엔진

대표 종류:
- **containerd** — 현재 쿠버네티스에서 가장 널리 쓰이는 컨테이너 런타임
- **CRI-O** — Docker 기능 없이 쿠버네티스 실행에 필요한 최소 기능만 제공
- **(과거) Docker** — 예전 쿠버네티스에서 사용되던 컨테이너 런타임

### 전체 구성요소

- **개발자 PC**
  - Docker(또는 빌드 도구) : 이미지 생성
  - kubectl : 배포 요청 전송
- **이미지 저장소(Registry)**
  - Docker Hub / GitHub Container Registry / ECR
  - 이미지 보관 창고
- **쿠버네티스 Control Plane**
  - API Server : 접수/조회/권한/상태 관리
  - etcd : 상태 저장소
  - Scheduler : 어느 워커에 배치할지 결정
  - Controller Manager : 지금 상태가 사용자가 원한 상태와 다르면 자동으로 계속 수정하는 역할
- **쿠버네티스 Worker Node**
  - kubelet : 현장 에이전트(명령 감시, 실행 지시, 상태 보고)
  - containerd : 컨테이너 런타임(이미지 pull, 컨테이너 실행)
  - CNI(네트워크 플러그인) : Pod IP/통신 구성

### Pod 생성 전체 흐름 (0단계 ~ 6단계)

**0단계) 개발자 로컬 환경 (개발자가 컨테이너 이미지 생성)**

실행 파일 + 라이브러리 + 설정을 하나로 묶어서 이미지로 만든다.
- 쿠버네티스는 소스코드를 실행하지 않고 이미지를 실행한다.
- 즉, 쿠버네티스에서 파드를 실행하려면 반드시 이미지 형태여야 한다.
- 예) nginx 기반 웹 컨테이너

Docker로 이미지를 빌드한다.

```
docker build -t hub.example.com/nginx .
```

워커 노드들이 가져갈 수 있도록 이미지 창고에 올린다. 쿠버네티스 워커 노드는 개발자 PC에 있는 이미지를 직접 볼 수 없기 때문에 공용(또는 사설) Registry에 올려야 한다.

```
docker tag myapp:1.0 hub.example.com/myapp:1.0
docker push hub.example.com/myapp:1.0
```

중요 포인트:
- 쿠버네티스는 이미지를 직접 만드는 도구가 아니다.
- 이미지는 반드시 Docker 같은 도구로 미리 만들어져 있어야 한다.

**1단계) 사용자가 원하는 상태를 선언한다**

쿠버네티스의 시작은 "무엇을 어떻게 실행할지" 직접 명령하는 것이 아니다. 사용자는 "이 애플리케이션을 항상 이런 상태로 유지해줘"라고 요구사항만 선언한다.

예를 들어 nginx 컨테이너를 항상 3개 실행하고 싶다면 `replicas: 3`이라고만 적는다. 이 단계에서 중요한 점은 사용자는 실행 위치, 실행 순서, 실행 방법을 전혀 신경 쓰지 않는다는 것이다.

단지 "목표 상태(원하는 상태)만 말한다." 쿠버네티스는 이 목표를 기준으로 이후 모든 동작을 수행한다.

**2단계) API Server가 요청을 접수하고 상태로 저장한다**

사용자가 kubectl을 통해 요청을 보내면, 그 요청은 가장 먼저 API Server로 들어온다.

API Server는 이 사용자가 권한이 있는지 확인하고, 설정 파일(yaml)이 문법적으로 맞는지 검사하고, 실행 가능한 요청인지 검증한다.

문제가 없으면 "nginx Pod 3개를 유지해야 한다"는 정보를 클러스터의 상태 저장소(etcd)에 저장한다.

이 순간부터 쿠버네티스 클러스터는 이 상태를 반드시 유지해야 한다는 목표를 가지게 된다.

**3단계) Controller Manager가 상태를 계속 감시한다**

Controller Manager는 쿠버네티스에서 감시와 조정을 담당하는 관리자다.

이 컴포넌트는 API Server에 저장된 정보를 계속 지켜보면서 다음 질문을 반복한다.
- 지금 클러스터 상태가 사용자가 원한 상태와 같은가?
- 예를 들어 Pod가 3개여야 하는데 실제로는 2개만 실행 중이라면 Controller Manager는 상태가 어긋났다고 판단한다.

중요한 점은 이 감시는 한 번만 이루어지는 것이 아니라 클러스터가 살아 있는 동안 계속 반복된다.

**4단계) 상태가 다르면 Pod를 만들거나 줄이도록 요청한다**

Controller Manager는 컨테이너를 직접 실행하지 않는다. 대신 API Server에게 "Pod를 하나 더 만들어라" 또는 "Pod를 하나 줄여라"라고 요청한다.

이 요청으로 인해 새로운 Pod 객체가 생성된다. 이 Pod는 아직 어느 워커 노드에서 실행될지 정해지지 않았기 때문에 잠시 Pending 상태로 보일 수 있다.

이 단계는 무엇을 실행해야 하는지를 확정하는 단계라고 보면 된다.

**5단계) Scheduler가 Pod를 실행할 워커 노드를 결정한다**

Scheduler는 새로 만들어질 Pod를 보고 이 Pod를 어느 워커 노드에서 실행할지를 결정한다. 이때 Scheduler는 각 워커의 CPU와 메모리 여유, 라벨 조건, 분산 배치 여부 등을 고려한다.

결정을 마치면 Pod 정보에 "이 Pod는 worker1에서 실행한다"라는 정보가 기록된다(실행 장소가 확정).

**6단계) 워커 노드에서 실제 실행과 유지가 이루어진다**

Pod가 특정 워커 노드에 배정되면, 그 노드에서 실행 중인 kubelet이 이를 감지한다.

kubelet은 "이 Pod는 내가 실행해야 한다"는 사실을 확인한 뒤 컨테이너 런타임(containerd 등)에게 컨테이너 실행을 지시한다.

컨테이너 런타임은 이미지를 확인하고, 필요하면 Registry에서 이미지를 내려받은 뒤 컨테이너를 실제로 실행한다.

컨테이너가 정상적으로 실행되면 그 상태가 kubelet을 통해 API Server로 보고되고, Pod 상태는 Running이 된다.

이후에도 컨테이너가 종료되거나 문제가 생기면 이 상태 변화는 다시 감지되어 앞 단계들이 반복되면서 자동 복구가 이루어진다.

**전체 흐름 요약**

1. 개발자 Docker로 이미지 생성
2. Registry에 push
3. kubectl로 배포 요청
4. API Server가 접수
5. Scheduler가 노드 선택
6. kubelet이 명령 수신
7. containerd가 이미지 pull
8. 컨테이너 실행
9. Pod 생성 완료

---


---

> 📌 **핵심 요약**
> - 클러스터는 Control Plane(API Server, etcd, Scheduler, Controller Manager)과 Worker Node(kubelet, kube-proxy, Container Runtime)로 구성되며, 사용자는 원하는 상태(Desired State)만 선언하고 나머지는 쿠버네티스가 자동으로 조정한다
> - API Server는 모든 요청이 반드시 거치는 단일 진입점이고, etcd는 클러스터의 모든 상태를 저장하는 Key-Value 저장소다
> - kube-scheduler는 Pod를 어느 Node에 배치할지만 결정하고, kube-controller-manager는 현재 상태와 원하는 상태를 비교해 자동으로 조치한다
> - kubelet은 자신의 Node에 배정된 Pod를 관리하며 Container Runtime(containerd 등)에 컨테이너 실행을 지시하고, kube-proxy는 Service 트래픽을 Pod로 전달하는 네트워크 규칙을 관리한다
> - Pod 생성은 이미지 빌드/push(0단계) → 상태 선언(1) → API Server 접수(2) → Controller Manager 감시(3) → Pod 생성 요청(4) → Scheduler 노드 결정(5) → kubelet·컨테이너 런타임 실행(6)의 흐름으로 이루어진다
> - 관련: 1. 🔧 Kubernetes - 설치 · 2. 📦 Kubernetes - Pod 생성 · 4. 📦 Kubernetes - Namespace
