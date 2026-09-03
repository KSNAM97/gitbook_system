# Kubernetes - AutoScaling

> **Tag:** #Kubernetes #AutoScaling #HPA #VPA #ClusterAutoscaler #MetricsServer
> **핵심 요약:** HPA는 Metrics Server가 제공하는 CPU/Memory 사용률을 기준으로 Deployment의 replicas 값을 자동으로 조절해 Pod 개수를 수평으로 늘리거나 줄이며, VPA는 Pod의 requests 값을 수직으로 조절하고, Cluster Autoscaler는 Pending Pod를 감지해 Node 개수를 조절한다.

---

## 1. 개요 (Overview)

쿠버네티스에서 애플리케이션은 보통 여러 개의 Pod로 실행된다.

문제는 트래픽과 자원 사용량이 항상 일정하지 않다는 점이다.

- 평소에는 사용자 300명
- 점심시간에는 사용자 10,000명
- 밤에는 다시 사용자 200명

Pod를 항상 적은 개수로 유지하면 트래픽이 증가했을 때 응답 속도가 느려지거나 장애가 발생할 수 있다. Pod를 항상 많은 개수로 유지하면 대부분의 시간에 CPU, Memory, Node 자원이 낭비된다. 이 문제를 해결하기 위해 사용하는 개념이 AutoScaling(자동 확장)이다.

**AutoScaling의 핵심 목표**

- 현재 부하에 맞게 필요한 만큼 자원을 자동으로 늘린다.
- 부하가 줄어들면 필요 없는 자원을 자동으로 줄인다.
- 성능과 비용을 동시에 최적화한다.

쿠버네티스에서 AutoScaling은 한 가지 방식만 존재하지 않는다. 대표적으로 다음과 같이 구분할 수 있다.

- Pod 개수를 늘리거나 줄인다.
- Pod가 요청하는 CPU/Memory 크기를 조절한다.
- Pod가 배치되는 Node 자체를 늘리거나 줄인다.

| 구분 | 이름 | 무엇을 조절하는가 |
|---|---|---|
| 1단계 | HPA | Pod 개수 |
| 2단계 | VPA | Pod의 CPU/Memory requests |
| 3단계 | Cluster Autoscaler | Node 개수 |

HPA와 VPA는 서로 목적이 다르다.

- **HPA** : Pod를 여러 개로 늘리는 수평 확장
- **VPA** : Pod 하나가 요청하는 자원 크기를 조절하는 수직 확장
- **Cluster Autoscaler** : Pod를 실행할 Node 자체를 늘리거나 줄이는 확장

실무에서는 HPA와 Cluster Autoscaler를 함께 사용하는 경우가 많다.

1. HPA가 Pod를 증가시킨다.
2. Node 자원이 부족하면 새 Pod가 Pending 상태가 된다.
3. Cluster Autoscaler가 Node를 추가한다.
4. 새 Node에 Pending Pod가 배치된다.

---

## 2. Metrics Server

Metrics Server는 쿠버네티스 클러스터에서 Node와 Pod의 CPU, Memory 사용량을 수집해서 Kubernetes Metrics API로 제공하는 컴포넌트다.

HPA는 스스로 Pod의 CPU/Memory 사용량을 직접 측정하지 않는다. HPA는 Metrics API를 통해 현재 자원 사용량을 조회한다. 일반적인 CPU/Memory 기반 HPA에서는 Metrics Server가 이 데이터를 제공한다. 즉, Metrics Server는 AutoScaling에서 사용할 수 있는 현재 자원 사용량을 제공하는 데이터 제공자 역할을 한다.

**Metrics Server가 제공하는 대표적인 정보**

- Node CPU 사용량
- Node Memory 사용량
- Pod CPU 사용량
- Pod Memory 사용량

Metrics Server는 장기간 데이터를 저장하는 모니터링 시스템이 아니다. 현재에 가까운 CPU/Memory 사용량을 제공하는 용도다. 과거 사용량 분석이나 장기간 Metric 저장은 Prometheus 같은 별도의 모니터링 시스템을 사용한다.

### Metrics Server가 필요한 이유

HPA는 CPU나 Memory 같은 Resource Metric을 기준으로 AutoScaling할 수 있다. 이때 각 Pod가 현재 CPU와 Memory를 얼마나 사용하고 있는지 알아야 한다. 이 정보를 Metrics Server가 제공한다.

HPA의 CPU 사용률은 단순히 Node 전체 CPU 대비 사용률이 아니다. Pod에 설정된 CPU requests 값을 기준으로 계산한다.

예:

- Pod CPU requests : 200m (CPU 0.2)
- Pod 실제 CPU 사용량 : 100m (CPU 0.1)
- Pod 현재 CPU 사용률 : 50%

계산: 100m / 200m × 100 = CPU 사용률 50%

따라서 CPU 사용률 기반 HPA를 정상적으로 사용하려면 일반적으로 Pod에 CPU requests가 설정되어 있어야 한다. Metrics Server가 없으면 기본적인 CPU/Memory Resource Metric을 HPA가 조회할 수 없기 때문에 정상적인 AutoScaling 판단을 할 수 없다.

### Metrics Server의 동작 위치

Metrics Server는 쿠버네티스 클러스터 내부에서 Deployment 형태로 실행된다. 일반적으로 kube-system Namespace에 설치된다. Metrics Server는 각 Node의 kubelet으로부터 CPU와 Memory 사용량 정보를 수집한다.

**동작 구조**

1. 각 Node에는 kubelet이 실행된다.
2. kubelet이 Node와 Container의 자원 사용량 정보를 제공한다.
3. Metrics Server가 각 Node의 kubelet로부터 Metric을 수집한다.
4. 수집한 정보를 Kubernetes Metrics API로 제공한다.
5. HPA가 Metrics API를 조회한다.

**전체 흐름**

```
Pod / Node  -->  kubelet  -->  Metrics Server  -->  Metrics API  -->  HPA
```

### HPA와 Metrics Server의 관계

HPA는 Pod 내부에 직접 접속해서 CPU/Memory를 확인하지 않는다. HPA Controller가 Metrics API를 통해 자원 사용량을 조회한다. 일반적인 CPU/Memory 기반 HPA에서는 Metrics Server가 해당 Metric 데이터를 제공한다.

**역할 구분**

- Metrics Server : 현재 CPU/Memory 사용량 제공
- HPA : Metric을 보고 필요한 Pod 개수 결정
- Deployment : 지정된 replicas 개수만큼 Pod 유지

**동작 흐름**

1. Pod에서 CPU/Memory 사용량 발생
2. kubelet이 자원 사용량 정보 제공
3. Metrics Server가 각 Node의 kubelet에서 Metric 수집
4. HPA가 Metrics API를 조회
5. HPA가 현재 Metric과 목표 Metric 비교
6. 필요한 replicas 개수 계산
7. Deployment의 replicas 값을 변경
8. Deployment가 Pod를 생성하거나 삭제

Metrics Server는 데이터 제공자, HPA는 판단자, Deployment는 실제 Pod 개수를 유지하는 실행자다.

---

## 3. HPA (Horizontal Pod Autoscaler)

HPA는 워크로드의 Pod 개수를 자동으로 늘리거나 줄이는 기능이다.

**Horizontal의 의미**

- Pod 하나의 크기를 키우는 것이 아니라
- 동일한 Pod의 개수를 늘리는 방식이다.

예:

- 트래픽 증가 --> Pod 2개 --> Pod 5개 --> Pod 10개
- 트래픽 감소 --> Pod 10개 --> Pod 5개 --> Pod 2개

HPA는 일반적으로 Deployment 또는 StatefulSet 같은 Scale 가능한 워크로드를 대상으로 사용한다. ReplicaSet도 Scale 대상이 될 수 있지만 일반적으로 Deployment를 대상으로 사용하는 경우가 많다.

HPA는 다음과 같은 Metric을 기준으로 판단할 수 있다.

- CPU 사용률
- Memory 사용량

가장 기본적인 기준은 CPU 사용률이다.

예:

- target CPU : 50%, 현재 평균 CPU : 80% → Pod 증가 가능
- target CPU : 50%, 현재 평균 CPU : 20% → 일정 조건이 충족되면 Pod 감소 가능

중요한 점은 목표값을 조금 넘었다고 Pod를 무조건 1개씩 증가시키는 단순한 방식은 아니라는 것이다. HPA는 현재 Metric과 목표 Metric의 비율을 사용해서 필요한 replicas 개수를 계산한다.

개념적으로 다음과 같이 이해할 수 있다.

```
필요한 Pod 수 = 현재 Pod 수 × 현재 Metric / 목표 Metric
```

예:

- 현재 Pod : 2개
- 현재 평균 CPU : 100%
- 목표 CPU : 50%

계산: 2 × 100 / 50 = 4 → 약 4개의 Pod가 필요하다고 판단할 수 있다.

### HPA CPU 사용률의 기준

HPA에서 CPU 사용률을 사용할 때 매우 중요한 것은 requests 값이다.

예:

```yaml
resources:
  requests:
    cpu: "200m"
```

현재 실제 CPU 사용량이 100m라면 100m / 200m × 100 = 50%, CPU 사용률은 50%가 된다. 따라서 CPU Utilization 기반 HPA에서는 requests.cpu가 중요하다. HPA CPU 사용률의 기준은 limits가 아니라 requests이다.

- **requests** : Pod가 실행되기 위해 "최소한 이 정도 자원이 필요하다"고 요청하는 값. Scheduler가 어느 Node에 배치할지 판단할 때 사용.
- **limits** : Container가 사용할 수 있는 최대 자원 한도. 이 값을 넘어서 사용하지 못하도록 제한.

### HPA는 평균값을 사용

HPA는 특정 Pod 하나만 보고 판단하지 않는다. 대상 Pod들의 Metric을 이용해서 전체적으로 필요한 replicas 수를 계산한다.

예:

- Pod 1: CPU 40%
- Pod 2: CPU 60%
- Pod 3: CPU 50%

평균: (40 + 60 + 50) / 3 = 50%

목표 CPU 사용률이 50%라면 Pod 개수를 크게 변경할 필요가 없다고 판단할 수 있다. 단, 실제 계산은 각 Pod의 requests와 실제 사용량을 기반으로 이루어진다.

### HPA 동작 흐름

1. **사용자 트래픽 증가** : 서비스에 접속하는 사용자나 요청 수가 증가한다.
2. **Pod CPU 사용량 증가** : 요청을 처리하는 Pod의 CPU 사용량이 증가한다.
3. **kubelet** : 각 Node의 kubelet이 해당 Node에서 실행 중인 Pod와 Container의 자원 사용량 정보를 제공한다.
4. **Metrics Server** : 각 Node의 kubelet으로부터 CPU와 Memory 사용량 정보를 수집한다.
5. **Metrics API** : Metrics Server가 수집한 자원 사용량을 Kubernetes Metrics API 형태로 제공한다.
6. **HPA Controller** : HPA Controller가 Metrics API를 통해 현재 Pod의 자원 사용량을 조회한다.
7. **현재 Metric과 목표 Metric 비교** : 현재 CPU 또는 Memory 사용량과 HPA에 설정된 목표값을 비교한다.
   - 예: 목표 CPU 사용률 50%, 현재 평균 CPU 사용률 80% → Pod를 늘릴 필요가 있다고 판단
8. **Deployment replicas 변경** : HPA가 필요한 Pod 개수를 계산하고 Deployment의 replicas 값을 변경한다. (예: replicas: 2 --> replicas: 4)
9. **Deployment / ReplicaSet** : Deployment가 변경된 replicas 값을 기준으로 ReplicaSet을 통해 필요한 Pod 개수를 유지한다.
10. **새 Pod 생성** : ReplicaSet이 부족한 개수만큼 새로운 Pod를 생성한다. 결과적으로 트래픽을 여러 Pod가 나누어 처리하게 된다.

트래픽이 감소하면 반대로 replicas 값을 줄일 수 있다.

### HPA는 Pod를 직접 생성하는가

HPA는 Pod를 직접 생성하지 않는다. HPA는 Scale 대상 리소스의 replicas 값을 변경한다.

예:

```
Deployment
replicas: 2

-HPA가 부하 증가를 감지하면
replicas: 5
```

Deployment Controller가 replicas=5 상태를 맞추기 위해 Pod를 증가시킨다.

따라서 역할은 다음과 같다.

- HPA : 몇 개의 Pod가 필요한지 결정
- Deployment : 해당 개수의 Pod를 유지
- ReplicaSet : 실제 Pod 복제 관리

### HPA와 Deployment를 함께 사용하는 이유

HPA는 일반 Pod 하나를 직접 늘리는 방식으로 사용하는 것이 아니다. Pod 단독 리소스에는 replicas라는 개념이 없다. Deployment는 replicas 값을 가지고 있기 때문에 HPA가 이 값을 조절할 수 있다.

일반적인 구조:

```
HPA  -->  Deployment  -->  ReplicaSet  -->  Pod
```

### HPA의 최소/최대 Pod 수

HPA에는 최소 Pod 수와 최대 Pod 수를 지정할 수 있다.

예:

```yaml
minReplicas: 2
maxReplicas: 10
```

의미:

- 아무리 부하가 낮아도 최소 2개의 Pod는 유지한다.
- 아무리 부하가 높아도 최대 10개까지만 증가한다.

이 값은 무한 확장을 방지하고 서비스의 최소 가용성을 유지하기 위해 중요하다.

### HPA Scale Down 주의점

HPA는 부하가 감소했다고 해서 즉시 Pod를 대량으로 줄이지 않는 경우가 있다. 부하가 일시적으로 감소했을 가능성이 있기 때문이다. 이를 방지하기 위해 Scale Down에는 안정화 시간이 적용될 수 있다.

목적:

- Pod가 계속 늘었다 줄었다 반복하는 현상 방지
- 불필요한 Pod 재생성 방지
- 서비스 안정성 확보

이처럼 자원이 빠르게 증가와 감소를 반복하는 현상을 Flapping 또는 Thrashing이라고 표현하기도 한다.

---

## 4. VPA (Vertical Pod Autoscaler)

VPA는 Vertical Pod Autoscaler의 약자다. VPA는 Pod의 개수를 늘리는 것이 아니라 Pod가 요청하는 CPU와 Memory 값을 조절하는 방식이다.

기존:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
```

VPA 적용 후:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

- HPA : Pod 개수 조절
- VPA : Pod 하나당 필요한 CPU/Memory requests 조절

VPA는 일반적으로 limits를 단순히 자동 증가시키는 기능이라고 이해하면 안 된다. 핵심은 워크로드의 실제 자원 사용량을 분석하고 적절한 CPU/Memory requests 값을 추천하거나 적용하는 것이다.

### VPA를 사용하는 이유

HPA는 Pod를 여러 개로 늘려서 부하를 분산하는 방식이다. 하지만 모든 애플리케이션이 Pod 증가만으로 성능이 좋아지는 것은 아니다.

예:

- 하나의 프로세스가 많은 Memory를 필요로 하는 서비스
- JVM Heap이 큰 애플리케이션
- 데이터 처리 작업
- 개별 Pod 자체의 자원이 부족한 애플리케이션
- 여러 Pod로 쉽게 분산하기 어려운 애플리케이션

이 경우 Pod 개수를 늘리는 것보다 Pod 하나에 할당되는 자원을 적절하게 조정하는 것이 더 효과적일 수 있다.

### VPA의 동작 방식

VPA는 워크로드의 자원 사용 패턴을 분석해서 적절한 CPU/Memory requests 값을 추천할 수 있다.

주요 동작 방식:

- 추천값만 제공
- 새 Pod 생성 시 추천값 적용
- 필요에 따라 기존 Pod를 교체해서 추천값 적용

사용하는 VPA 버전과 설정에 따라 세부 동작 방식은 달라질 수 있다.

### VPA의 특징과 주의점

CPU/Memory requests 값을 변경하기 위해 기존 Pod를 재생성해야 하는 경우가 있다. 따라서 서비스 중단에 민감한 환경에서는 설정과 운영 방식을 주의해야 한다. 또한 동일한 CPU/Memory Metric에 대해 HPA와 VPA를 동시에 사용하면 서로 영향을 줄 수 있다.

예:

- HPA는 CPU 사용률을 기준으로 Pod 수 증가
- VPA는 CPU requests 변경
- requests가 변경되면 HPA의 CPU 사용률 계산값도 변경될 수 있음

따라서 동일한 자원 Metric을 기준으로 HPA와 VPA를 함께 사용할 때는 설계가 필요하다.

실무에서는:

- HPA를 기본 확장 방식으로 사용
- VPA는 자원 추천 또는 특정 워크로드에 제한적으로 사용하는 경우가 많다.

---

## 5. Cluster Autoscaler

Cluster Autoscaler는 Pod 개수가 아니라 Node 개수를 자동으로 조절한다.

즉,

- Pod를 배치할 Node 자원이 부족하면 Node 증가
- Node가 장시간 불필요하게 남으면 Node 감소

주로 AWS, GCP, Azure 같은 Cloud 환경에서 사용한다.

### 왜 Node AutoScaling이 필요한가

HPA가 Pod 수를 늘려도 실제로 Pod를 실행할 Node 자원이 없으면 Pod가 정상적으로 실행되지 못한다.

예: HPA replicas: 3 --> replicas: 10

하지만 Node 전체의 CPU와 Memory가 부족하면 일부 Pod는 STATUS: Pending 상태가 된다.

대표적인 원인:

- Insufficient cpu
- Insufficient memory

이 상황을 해결하기 위해 Cluster Autoscaler가 Node 수를 증가시킬 수 있다.

### Cluster Autoscaler의 판단 기준

Cluster Autoscaler는 단순히 "Node CPU 사용률이 90%니까 Node를 하나 추가한다"와 같이 동작하는 것이 아니다. 핵심적으로 Scheduler가 배치하지 못한 Unschedulable Pod를 확인한다.

판단 기준:

- Pending 상태의 Pod가 있는가?
- Scheduler가 자원 부족 등의 이유로 배치하지 못했는가?
- 새 Node가 추가되면 해당 Pod를 배치할 수 있는가?
- 사용 중인 Node Group이 확장 가능한가?

조건이 충족되면 Cloud Provider의 Node Group을 확장한다.

### Node Scale Down

Cluster Autoscaler는 Node를 늘리는 것뿐만 아니라 줄일 수도 있다.

동작 흐름:

```
트래픽 감소  -->  HPA가 Pod 감소  -->  일부 Node의 사용량 감소  -->
해당 Node의 Pod를 다른 Node로 이동 가능한지 확인  -->  안전하게 비울 수 있으면 Node 제거
```

단, 모든 Node를 무조건 제거할 수 있는 것은 아니다.

다음과 같은 조건 때문에 Scale Down이 제한될 수 있다.

- 다른 Node로 이동할 수 없는 Pod
- 특정 Node에 고정된 Pod
- Local Storage 사용
- PodDisruptionBudget 조건
- Node Affinity / Node Selector 조건
- 기타 Scheduling 제약

### Metrics Server와 Cluster Autoscaler의 차이

Metrics Server는 Node와 Pod의 CPU/Memory 사용량 정보를 제공한다. Metrics Server가 직접 Pod 또는 Node 개수를 늘리거나 줄이지는 않는다. Cluster Autoscaler는 Node의 CPU 사용률이 단순히 높다고 해서 Node를 추가하는 방식이 아니다. 주요 판단 기준은 Scheduler가 배치할 수 없는 Pod가 존재하는지 여부이다.

예:

- HPA가 Pod를 3개에서 10개로 증가시킨다.
- 현재 Node에는 추가 Pod가 들어갈 자원이 없다.
- 일부 Pod가 Pending 상태가 된다.
- Cluster Autoscaler가 새 Node를 추가한다.
- Scheduler가 Pending Pod를 새 Node에 배치한다.

Node 축소 시에는 다음과 같은 조건을 확인한다.

- 사용률이 낮은 Node가 있는가?
- 해당 Node의 Pod들을 다른 Node로 옮길 수 있는가?
- 안전하게 Node를 제거할 수 있는가?

### HPA + Cluster Autoscaler 관계

실무에서 자주 함께 사용하는 조합이다. HPA는 Pod 개수를 조절한다. Cluster Autoscaler는 Node 개수를 조절한다.

동작 흐름:

```
트래픽 증가  -->  Pod CPU/Memory 사용량 증가  -->  Metrics Server가 Metric 제공  -->
HPA가 Deployment replicas 증가  -->  새 Pod 생성 시도  -->  현재 Node 자원 부족  -->  일부 Pod Pending
-->  Cluster Autoscaler가 Pending Pod 확인  -->  Cloud 환경에 Node 추가 요청  -->
새 Node가 Kubernetes Cluster에 참여  -->  Scheduler가 Pending Pod를 새 Node에 배치
```

트래픽이 감소하면 반대로 동작할 수 있다.

---

## 6. 실습 EX1) Metrics Server 설치 + CPU 기준 HPA

### Metrics Server 설치 전 확인

```bash
[root@k8s-master ~]# kubectl  top  nodes
error: Metrics API not available
```

### Metrics Server 설치

```bash
[root@k8s-master ~]# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

### Metrics Server Deployment 수정 (kubelet-insecure-tls 추가)

```bash
[root@k8s-master ~]# kubectl  edit deployments.apps metrics-server  --namespace kube-system
~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls			# 추가 설정
        image: registry.k8s.io/metrics-server/metrics-server:v0.9.0
        imagePullPolicy: IfNotPresent
~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~
```

### Metrics Server Pod 확인

```bash
[root@k8s-master ~]# kubectl get pods  --namespace  kube-system
NAME                                 	READY   STATUS    RESTARTS         AGE
coredns-7d764666f9-2dwdd             	1/1        Running     14 (5h36m ago)   16d
coredns-7d764666f9-klkhq             	1/1        Running     14 (5h36m ago)   16d
etcd-k8s-master                      	1/1        Running     15 (5h36m ago)   16d
kube-apiserver-k8s-master            	1/1        Running     15 (5h36m ago)   16d
kube-controller-manager-k8s-master   	1/1        Running     15 (5h36m ago)   16d
kube-proxy-bxjvw                     	1/1        Running     14 (5h36m ago)   16d
kube-proxy-gbr4f                     	1/1        Running     13 (5h36m ago)   16d
kube-proxy-rwhkc                     	1/1        Running     14 (5h36m ago)   16d
kube-scheduler-k8s-master            	1/1        Running     15 (5h36m ago)   16d
metrics-server-67cbccccd9-n86d6      	1/1        Running     0                     12m		# Metrics Server Pod
```

### 설치 확인 (kubectl top)

```bash
[root@k8s-master ~]# kubectl  top  nodes
NAME          	CPU(cores)	CPU(%)     MEMORY(bytes)   	MEMORY(%)
k8s-master    	110m           	2%           1072Mi                	51%
k8s-worker1	41m            	1%           774Mi                 	42%
k8s-worker2   	29m            	0%           615Mi                 	33%


[root@k8s-master ~]# kubectl  top  pods
No resources found in default namespace.
```

### Deployment, Service 생성

```bash
	# Deploymoent , Service 생성
[root@k8s-master ~]# vi deploy_web_autoscale.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: smlinux/hpa-example
        name: web
        ports:
        - containerPort: 80
        resources: 
          requests: 
            cpu: 200m  
---
apiVersion: v1
kind: Service
metadata:
  name: svc-web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  deploy_web_autoscale.yaml
deployment.apps/deploy-web created
service/svc-web created


[root@k8s-master ~]# kubectl  get  deployments  deploy-web
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web   1/1         1                    1                  35s


[root@k8s-master ~]# kubectl  get  pods
NAME                                  READY   STATUS    RESTARTS   AGE
deploy-web-769979f576-kn8lp   1/1         Running     0                 45s


[root@k8s-master ~]# kubectl  get  service  svc-web
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
svc-web    ClusterIP   10.102.72.98       <none>            80/TCP    78s
```

### HPA 생성

```bash
	# HPA 생성
[root@k8s-master ~]# vi hpa_web.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpe-web
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: deploy-web
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

**hpa_web.yaml 필드 설명**

```yaml
apiVersion: autoscaling/v2    	# HPA의 최신(현행) API 버전
kind: HorizontalPodAutoscaler 	# 파드 개수를 자동으로 조절하는 리소스
metadata:
  name: hpe-web              	# HPA 리소스 이름
spec:
  minReplicas: 1              	# 최소 유지할 Pod 개수 (이 값 아래로는 줄지 않음)
  maxReplicas: 10              	# 최대 확장 가능한 Pod 개수
  scaleTargetRef:            	# 어떤 리소스를 기준으로 스케일링할지 지정
    apiVersion: apps/v1     	# 대상 리소스의 API 버전
    kind: Deployment          	# 대상 리소스 종류 (Deployment 기준으로 스케일)
    name: deploy-web        	# 스케일링 대상 Deployment 이름
  metrics:                    	# 스케일링 판단에 사용할 메트릭 목록
  - type: Resource          	# 쿠버네티스 기본 리소스(CPU, Memory 등) 기준
    resource:
      name: cpu             	# CPU 사용률 기준으로 스케일링
      target:
        type: Utilization      	# requests 대비 사용률(%) 기준
        averageUtilization: 50 	# 평균 CPU 사용률이 50%를 넘으면 확장
                             	# (각 Pod의 cpu request 대비 평균값)
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  hpa_web.yaml
horizontalpodautoscaler.autoscaling/hpe-web created


[root@k8s-master ~]# kubectl  get  horizontalpodautoscalers.autoscaling
NAME      REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
hpe-web   Deployment/deploy-web   cpu: 0%/50%   1               10              1               21s


[root@k8s-master ~]# kubectl  get  hpa
NAME      REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
hpe-web   Deployment/deploy-web   cpu: 0%/50%   1               10              1               21s
```

### 부하 테스트용 이미지 (smlinux/hpa-example)

```
-image: smlinux/hpa-example


$ mkdir horizontal
$ vim index.php
<?php
$x = 0.0001;
for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
}
echo "OK!";
?>

$ vim dockerfile
FROM php:5-apache
ADD index.php /var/www/html/index.php
RUN chmod a+x index.php

$ docker build -t smlinux/hpa-example .
$ docker images
```

**index.php 코드 설명**

- `$x = 0.0001;` : 아주 작은 실수값으로 시작
- `for ($i = 0; $i <= 1000000; $i++)` : 100만 번 반복 (반복 횟수를 늘려 CPU 연산량이 증가)
- `$x += sqrt($x);` : 매 반복마다 제곱근 계산 수행 (sqrt 연산은 CPU를 사용하는 연산). 누적 계산으로 CPU 사용률을 계속 유지
- `echo "OK!";` : 연산이 끝나면 브라우저에 OK 출력
- 즉 OK 메시지 한번당 위의 연산을 1번 수행

### 부하 발생 및 HPA 동작 확인

```bash
[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                          	READY   STATUS    RESTARTS   AGE   IP              NODE
deploy-web-769979f576-kn8lp	1/1        Running      0                17m   10.244.2.11   k8s-worker2


[root@k8s-master ~]# curl  10.244.2.11
OK!


[root@k8s-master ~]# while  true
> do
> curl  10.244.2.11
> done
```

```bash
[root@k8s-master ~]# kubectl  apply  -f   hpa_web.yaml
horizontalpodautoscaler.autoscaling/hpe-web configured


[root@k8s-master ~]# watch kubectl  get  hpa
Every 2.0s: kubectl get hpa                                     k8s-master: Fri Aug 28 16:31:46 2026

NAME      REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
hpe-web   Deployment/deploy-web   cpu: 0%/50%    2              10              2               35m
```

### 리소스 정리

```bash
	# Deployment 삭제
[root@k8s-master ~]# kubectl  delete  deployments.apps  deploy-web
deployment.apps "deploy-web" deleted


	# service 삭제
[root@k8s-master ~]# kubectl  delete  service  svc-web
service "svc-web" deleted


	# hpa 삭제
[root@k8s-master ~]# kubectl  delete  hpa  hpe-web
horizontalpodautoscaler.autoscaling "hpe-web" deleted
```

---

## 7. 실습 EX2) HPA v2(CPU + Memory)로 파드 확장 + 서비스 분산 처리 확인

autoscaling/v2 HPA 적용 후 replicas가 실제로 늘어나는 것을 확인한다.

### 베이스 리소스 생성 (Deployment + Service)

```bash
	# 베이스 리소스 생성(Deployment + Service)
[root@k8s-master ~]# vi step2-1-deploy-svc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-web-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-v2
  template:
    metadata:
      labels:
        app: web-v2
    spec:
      containers:
      - name: web
        image: smlinux/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: svc-web-v2
spec:
  type: ClusterIP
  selector:
    app: web-v2
  ports:
  - port: 80
    targetPort: 80
```

### HPA v2 생성 (CPU + Memory)

```bash
	# HPA v2 생성(CPU+Memory)
[root@k8s-master ~]# vi step2-1-hpa-cpu-mem.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-web-v2
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: deploy-web-v2
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60
```

- metrics(cpu/memory) : 평균 사용률 목표. 둘 중 하나라도 초과하면 확장 방향으로 동작

### 적용 및 확인

```bash
[root@k8s-master ~]# kubectl  apply -f  step2-1-deploy-svc.yaml
deployment.apps/deploy-web-v2 created
service/svc-web-v2 created


[root@k8s-master ~]# kubectl  apply -f  step2-1-hpa-cpu-mem.yaml
horizontalpodautoscaler.autoscaling/hpa-web-v2 created


[root@k8s-master ~]# kubectl  get  deployments  deploy-web-v2
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web-v2   1/1     1            1           33s


[root@k8s-master ~]# kubectl  get  pods
NAME                             READY   STATUS    RESTARTS   AGE
deploy-web-v2-65f8fbb986-pcmxg   1/1     Running   0          43s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
deploy-web-v2-65f8fbb986-pcmxg   1/1     Running   0          51s   10.244.2.16   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  get  service svc-web-v2
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
svc-web-v2   ClusterIP   10.107.60.9   <none>        80/TCP    68s


[root@k8s-master ~]# kubectl  get  hpa  hpa-web-v2
NAME         REFERENCE                  TARGETS                       MINPODS   MAXPODS   REPLICAS   AGE
hpa-web-v2   Deployment/deploy-web-v2   cpu: 0%/50%, memory: 7%/60%   1         10        1          2m27s
```

### 부하 발생 및 확장 관찰 (터미널 3개 사용)

**터미널 1 — HPA 상태 모니터링**

```bash
	# 터미널 1
[root@k8s-master ~]# watch -n 2 kubectl get hpa hpa-web-v2
Every 2.0s: kubectl get hpa hpa-web-v2                                                                      k8s-master: Fri Aug 28 16:42:49 2026

NAME          REFERENCE                      TARGETS                              MINPODS   MAXPODS   REPLICAS   AGE
hpa-web-v2   Deployment/deploy-web-v2   cpu: 0%/50%, memory: 7%/60%   1             10               1              5m19s
```

**터미널 2 — Pod 목록 모니터링**

```bash
	# 터미널 2
[root@k8s-master ~]# watch -n 2 kubectl get pods -l app=web-v2 -o wide
Every 2.0s: kubectl get pods -l app=web-v2 -o wide                                                          k8s-master: Fri Aug 28 16:42:57 2026

NAME                                       READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
deploy-web-v2-65f8fbb986-pcmxg   1/1        Running     0                 4m24s   10.244.2.16   k8s-worker2   <none>                    <none>
```

**터미널 3 — 부하 발생 (curl)**

```bash
	# 터미널 3
[root@k8s-master ~]# curl 10.244.2.16
OK![



[root@k8s-master ~]# curl 10.244.2.16
OK!
[root@k8s-master ~]# curl 10.244.2.16
OK!
[root@k8s-master ~]# curl 10.244.2.16
OK!
[root@k8s-master ~]# curl 10.244.2.16
OK!
   ~~~~~~~~~~~~~
[root@k8s-master ~]# curl 10.244.2.16
OK!
[root@k8s-master ~]# curl 10.244.2.16
OK!


[root@k8s-master ~]# watch -n 2 kubectl get hpa hpa-web-v2
Every 2.0s: kubectl get hpa hpa-web-v2                                                                      k8s-master: Fri Aug 28 16:44:35 2026

NAME           REFERENCE                     TARGETS                                MINPODS   MAXPODS   REPLICAS   AGE
hpa-web-v2   Deployment/deploy-web-v2   cpu: 28%/50%, memory: 9%/60%   1             10               1               7m5s
```

**터미널 3 — 지속적인 부하 (while true)**

```bash
	# 터미널 3
[root@k8s-master ~]# while  true
> do
> curl 10.244.2.16
> done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!
```

**터미널 1 — Pod 9개까지 확장됨**

```bash
	# 터미널 1
[root@k8s-master ~]# watch -n 2 kubectl get hpa hpa-web-v2
Every 2.0s: kubectl get hpa hpa-web-v2                                                                      k8s-master: Fri Aug 28 16:47:56 2026

NAME         REFERENCE                       TARGETS                                MINPODS   MAXPODS   REPLICAS   AGE
hpa-web-v2   Deployment/deploy-web-v2   cpu: 49%/50%, memory: 7%/60%   1              10              9               10m
```

**터미널 2 — 실제 Pod 9개로 분산 배치 확인**

```bash
	# 터미널 2
[root@k8s-master ~]# watch -n 2 kubectl get pods -l app=web-v2 -o wide
Every 2.0s: kubectl get pods -l app=web-v2 -o wide                                                          k8s-master: Fri Aug 28 16:47:03 2026

NAME                             		READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
deploy-web-v2-65f8fbb986-27s2g   	1/1        Running   0          33s     10.244.2.17   k8s-worker2   <none>           <none>
deploy-web-v2-65f8fbb986-6br76   	1/1        Running   0          18s     10.244.1.16   k8s-worker1   <none>           <none>
deploy-web-v2-65f8fbb986-8rq9p   	1/1        Running   0          18s     10.244.2.19   k8s-worker2   <none>           <none>
deploy-web-v2-65f8fbb986-gss7d   	1/1        Running   0          33s     10.244.1.14   k8s-worker1   <none>           <none>
deploy-web-v2-65f8fbb986-j4dv2   	1/1        Running   0          18s     10.244.2.18   k8s-worker2   <none>           <none>
deploy-web-v2-65f8fbb986-kc4d9   	1/1        Running   0          3s      10.244.1.17   k8s-worker1   <none>           <none>
deploy-web-v2-65f8fbb986-pcmxg   	1/1        Running   0          8m30s   10.244.2.16   k8s-worker2   <none>           <none>
deploy-web-v2-65f8fbb986-q75ds   	1/1        Running   0          33s     10.244.1.15   k8s-worker1   <none>           <none>
deploy-web-v2-65f8fbb986-rv88t   	1/1        Running   0          3s      10.244.2.20   k8s-worker2   <none>           <none>
```

CPU 사용률이 목표치(50%)에 근접하면서 replicas가 1개에서 9개까지 자동으로 확장되었고, 새로 생성된 Pod들이 k8s-worker1과 k8s-worker2에 나뉘어 분산 배치된 것을 확인할 수 있다.

### 메모리 부하를 사용한 auto-scaling

동일한 Deployment/Service 구성(cpu/memory requests 포함)을 기준으로 Memory 부하 방식으로도 동일하게 auto-scaling 테스트를 진행할 수 있다.

```bash
	# 메모리 부하를 사용한 auto-scaling
[root@k8s-master ~]# vi step2-1-deploy-svc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-web-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-v2
  template:
    metadata:
      labels:
        app: web-v2
    spec:
      containers:
      - name: web
        image: smlinux/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: svc-web-v2
spec:
  type: ClusterIP
  selector:
    app: web-v2
  ports:
  - port: 80
    targetPort: 80
```

---

## 8. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `kubectl top nodes` 실행 시 `error: Metrics API not available`가 나오면 Metrics Server가 설치되어 있지 않거나 정상 동작하지 않는 것이므로, `kubectl get pods --namespace kube-system`으로 metrics-server Pod의 상태를 먼저 확인한다.
- 사설 인증서(kubelet 자체 서명 인증서)를 사용하는 클러스터에서는 metrics-server가 kubelet과 TLS 통신에 실패할 수 있으므로 `kubectl edit deployments.apps metrics-server --namespace kube-system`으로 `--kubelet-insecure-tls` 옵션을 추가해야 정상 동작하는 경우가 많다.
- HPA의 TARGETS 값이 계속 `<unknown>`으로 보이면 Metrics Server가 아직 Metric을 수집하지 못했거나, 대상 Pod에 `resources.requests.cpu`가 설정되어 있지 않은 경우다. CPU Utilization 기반 HPA는 requests 값이 반드시 필요하다.
- HPA가 예상만큼 Pod를 늘리지 않는다면 `minReplicas`/`maxReplicas` 범위와 목표 Utilization 값을 확인하고, 부하가 실제로 목표치를 넘고 있는지 `kubectl get hpa`의 TARGETS 컬럼(cpu: 현재%/목표%)으로 확인한다.
- Scale Down이 느리게 느껴지는 것은 버그가 아니라 Flapping/Thrashing 방지를 위한 정상적인 안정화 동작이다. 부하가 감소해도 일정 시간 관찰 후 서서히 줄어든다.
- HPA로 Pod가 늘었는데 일부가 `Pending` 상태로 남아있다면 Node 자원 부족(Insufficient cpu/memory) 가능성이 높다 — `kubectl describe pod <pod-name>`으로 원인을 확인하고, Cloud 환경이라면 Cluster Autoscaler 동작 여부를 점검한다.
- HPA와 VPA를 동일한 CPU/Memory Metric으로 동시에 적용하면 VPA가 requests를 바꾸는 순간 HPA의 사용률 계산 기준도 바뀌어 예측 불가능한 동작이 발생할 수 있으므로, 같은 리소스에 대해 동시 사용은 신중히 설계해야 한다.

---

>  **핵심 요약**
> - AutoScaling은 목적에 따라 HPA(Pod 개수), VPA(Pod의 CPU/Memory requests), Cluster Autoscaler(Node 개수) 3단계로 구분된다
> - HPA는 Metrics Server가 kubelet으로부터 수집해 Metrics API로 제공하는 CPU/Memory 사용률을 기준으로 판단하며, CPU 사용률은 limits가 아니라 requests 대비로 계산된다
> - HPA는 `필요한 Pod 수 = 현재 Pod 수 × 현재 Metric / 목표 Metric` 공식으로 replicas를 계산해 Deployment의 replicas 값을 변경하고, 실제 Pod 생성/삭제는 Deployment·ReplicaSet이 담당한다
> - VPA는 Pod 개수 대신 requests 값 자체를 조절하는 수직 확장이며, HPA와 동일 Metric을 동시에 사용하면 서로 영향을 줄 수 있어 설계가 필요하다
> - Cluster Autoscaler는 Node CPU 사용률이 아니라 Scheduler가 배치하지 못한 Pending Pod 존재 여부를 기준으로 Node를 늘리거나 줄이며, 실무에서는 HPA와 함께 조합해 사용하는 경우가 많다
> - 관련: 12.  Kubernetes - Deployment · 13.  Kubernetes - Rollout·Rollback 실습 · 2.  Kubernetes - Pod 생성
