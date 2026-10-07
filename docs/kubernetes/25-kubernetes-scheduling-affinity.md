# Kubernetes - Pod Scheduling (nodeSelector·Affinity)

> **Tag:** #Kubernetes #PodScheduling #nodeSelector #NodeAffinity #PodAffinity
> **핵심 요약:** Pod가 어느 Node에 배치될지 결정하는 스케줄링 과정, nodeSelector를 이용한 단순 배치 제한, Node Affinity(required/preferred)를 이용한 세밀한 조건 배치, Pod Affinity/Anti-Affinity를 이용한 Pod 간 관계 기반 배치까지 실습 전체 정리

---

## 1. 개요 (Overview)

Pod Scheduling은 Pod를 어느 Node(서버)에 올려서 실행할지 결정하는 과정이다. Pod는 그냥 만들어진다고 바로 실행되지 않는다. (반드시 배치될 Node가 결정되어야 한다.) 이 결정을 하는 역할이 kube-scheduler(스케줄러)다.

```
# Pod = 실행할 프로그램 묶음
# Node = Pod가 실제로 올라가서 실행되는 서버(워커 서버)
# Scheduler	= Pod를 어느 Node에 올릴지 선택하는 관리자
```

쿠버네티스에서 Pod가 계속 Pending(대기) 상태로 남는 경우가 있다. Pending은 Pod가 아직 실행 상태가 되지 않았다는 의미다. 스케줄링 대상 Pod라면 배치할 적절한 Node를 찾지 못한 경우가 대표적인 원인이다. 하지만 이미지 다운로드, 볼륨 연결 등 다른 이유로도 Pending 상태가 유지될 수 있으므로 Pending이 항상 스케줄링 실패를 의미하는 것은 아니다.

Scheduling이 실제로 진행되는 흐름은 다음과 같다.

```
Pod 생성 --> API Server에 Pod 정보 저장 --> kube-scheduler가 Node 선택 --> Pod가 해당 Node에 Bind
   --> 해당 Node의 kubelet이 Pod 실행
```

**스케줄러가 Node를 선택하는 방식**

스케줄러는 단순히 랜덤으로 Node를 고르지 않는다.

1) Pod를 올릴 수 없는 Node를 후보에서 제거한다. (Filtering)
2) 남은 Node의 적합성을 평가한다. (Scoring)
3) 점수가 높은 Node를 최종적으로 선택한다. (Binding)

Filtering에서는 Pod의 요구사항을 만족하지 못하는 Node를 후보에서 제외한다. Filtering 결과 배치 가능한 Node가 0개라면 해당 Pod는 스케줄링되지 못하고 Pending 상태가 유지될 수 있다. 배치 가능한 Node가 여러 개라면 Scoring을 통해 어느 Node가 더 적절한지 판단한다.

즉, 스케줄링은 단순히 "Node 하나를 선택하는 과정"이 아니라 배치할 수 없는 Node를 제거하고 남은 Node의 적합도를 비교한 후 최종 Node를 선택하는 과정이다.

**배치 가능한 Node의 조건**

스케줄러가 Node를 후보에서 제거할 때 확인하는 대표적인 조건은 아래와 같다.

1) 자원이 충분한가 (CPU, 메모리)
- Pod에는 requests(요청 자원)가 설정될 수 있다.
- 스케줄러는 Pod의 requests를 기준으로 Node에 해당 Pod를 배치할 수 있는지 판단한다.
- Node에 Pod가 요청한 만큼의 allocatable 자원이 충분하지 않으면 해당 Node는 후보에서 제외된다.
- requests가 클수록 배치할 수 있는 Node가 줄어들 수 있다.

예시
```
# Pod requests: cpu 1, memory 1Gi
# Node에 배치 가능한 여유 자원: cpu 0.5, memory 2Gi
# CPU 부족 --> 그 Node는 후보에서 제외
```

2) Node 상태가 정상인가
- 일반적으로 Node가 Ready 상태가 아니면 정상적인 Pod 배치가 어렵다.
- Node가 cordon 상태라면 새 Pod를 해당 Node에 배치하지 않는다.
 - cordon = 해당 Node를 Scheduling 대상에서 제외하는 것
 - 기존에 실행 중인 Pod는 그대로 유지될 수 있다.
 - 새 Pod는 다른 Node에 배치된다.

3) Pod가 요구한 노드 조건을 만족하는가
- 대표적으로 nodeSelector / node affinity가 있다.
 - nodeSelector : 특정 라벨이 있는 Node에만 배치
 - node affinity : Node의 라벨을 기준으로 강제 조건 또는 선호 조건으로 배치

4) Node가 Pod를 거부하고 있지는 않은가 (taint)
- taint : 특정 Node에 아무 Pod나 배치되지 못하도록 제한하는 기능
- toleration : 특정 Pod가 해당 taint를 허용할 수 있도록 설정하는 기능
- toleration이 없거나 taint를 만족하지 못하면 해당 Node는 후보에서 제외될 수 있다.

**스케줄링을 직접 제어하는 대표 방법 4가지**

1) nodeSelector (가장 쉬움)
- 라벨이 있는 노드에만 배치

2) Node Affinity (조건을 더 세밀하게)
- required : 조건 불만족이면 절대 배치 안 됨 (Pending 가능)
- preferred : 가능하면 그쪽, 없으면 다른 노드도 가능
- weight : preferred 조건의 선호도를 나타내는 점수

3) Pod Affinity/Anti-Affinity (분산 배치)
- 특정 Pod와 같은 Node 또는 가까운 위치에 배치
- 특정 Pod와 같은 Node에 몰리지 않도록 분산 배치
- 고가용성 및 장애 분산에 활용

4) taint & toleration (특정 노드 보호)
- GPU Node, 중요한 Node 등 특정 Node를 일반 Pod가 사용하지 못하도록 제한
- 필요한 Pod만 toleration을 사용하여 해당 Node에 배치할 수 있도록 허용 (자세한 내용은 Kubernetes - Pod Scheduling (Taint·Toleration) 참고)

---

## 2. nodeSelector

nodeSelector는 Pod를 특정 라벨(label)을 가진 Node에만 배치하도록 강제하는 기능이다. Pod가 스케줄러에게 이렇게 요구하는 것과 같다.

```
이 라벨이 붙은 Node가 아니면 나는 실행되지 않아도 된다.
```

**사용하는 이유**

모든 Node는 성능이 동일하지 않다.
- SSD 서버
- GPU 서버
- 테스트용 서버
- 운영 전용 서버

특정 Pod는 아무 Node에서나 실행되면 안 되는 경우가 있다. nodeSelector는 이런 경우 Pod의 배치 위치를 명확히 제한한다.

**기준은 Node의 라벨(label)**

- nodeSelector는 Node에 붙은 label(key=value)을 기준으로 동작한다.
- 라벨은 Node의 속성을 나타내는 이름표다.
- 예) disktype=ssd, env=prod
- 라벨은 미리 Node에 설정되어 있어야 한다.

**스케줄러가 nodeSelector를 처리하는 방식**

1) 모든 Node를 가져온다.
2) nodeSelector 조건을 확인한다.
3) 조건과 일치하지 않는 Node를 전부 제외한다.
4) 조건을 만족하는 Node만 배치 후보로 남긴다.

결과는 둘 중 하나다.
- 조건을 만족하는 Node가 있음 : 그 Node에 Pod 배치
- 조건을 만족하는 Node가 없음 : Pod는 Pending 상태 유지

nodeSelector는 선호 조건이 아니라 강제 조건이다.

예시
- Node 상태 : Node A(disktype=hdd), Node B(disktype=ssd)
- Pod 설정
```yaml
nodeSelector:
  disktype: ssd
```
- 결과 : Node A는 제외, Node B만 후보로 남음, Pod는 Node B에서 실행. 만약 Node B가 없다면 Pod는 Pending이 된다.

**주의할 점**
- 라벨 key 또는 value가 조금이라도 다르면 매칭되지 않는다.
- nodeSelector를 사용하면 해당 라벨이 없는 Node에는 절대 배치되지 않는다.
- 여러 라벨을 쓰면 모두 만족해야 배치된다 (AND 조건)

**Node 및 Label 확인**

```
[root@k8s-master ~]# kubectl get nodes
NAME           STATUS   ROLES         	AGE   VERSION
k8s-master     Ready      control-plane	14d     v1.35.7
k8s-worker1   Ready      <none>        	14d     v1.35.7
k8s-worker2   Ready      <none>       	14d     v1.35.7


[root@k8s-master ~]# kubectl get nodes --show-labels
NAME          STATUS   ROLES           AGE   VERSION   LABELS
k8s-master    Ready    control-plane   14d   v1.35.7   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-worker1   Ready    <none>          14d   v1.35.7   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,env=prod,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-worker1,kubernetes.io/os=linux
k8s-worker2   Ready    <none>          14d   v1.35.7   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=gpu,env=dev,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-worker2,kubernetes.io/os=linux
```

k8s-worker1, k8s-worker2에 미리 설정되어있는 Label을 삭제한다.

```
# k8s-worker1에 미리 설정되어있는 Label 삭제
# k8s-worker1	: disktype=ssd,env=prod

[root@k8s-master ~]# kubectl  label  node  k8s-worker1 disktype-
node/k8s-worker1 unlabeled

[root@k8s-master ~]# kubectl  label  node  k8s-worker1 env-
node/k8s-worker1 unlabeled


# k8s-worker2에 미리 설정되어있는 Label 삭제
# k8s-worker2	: disktype=gpu,env=dev

[root@k8s-master ~]# kubectl  label  node  k8s-worker2 disktype-
node/k8s-worker2 unlabeled

[root@k8s-master ~]# kubectl  label  node  k8s-worker2 env-
node/k8s-worker2 unlabeled


	OR


[root@k8s-master ~]# kubectl  label  node  k8s-worker{1,2} disktype-
node/k8s-worker1 unlabeled
node/k8s-worker2 unlabeled

[root@k8s-master ~]# kubectl  label  node  k8s-worker{1,2} env-
node/k8s-worker1 unlabeled
node/k8s-worker2 unlabeled


[root@k8s-master ~]# kubectl  get  nodes  -L  disktype
NAME          STATUS   ROLES           AGE   VERSION   DISKTYPE
k8s-master    Ready    control-plane   14d   v1.35.7
k8s-worker1   Ready    <none>          14d   v1.35.7
k8s-worker2   Ready    <none>          14d   v1.35.7


[root@k8s-master ~]# kubectl  get  nodes  -L  env
NAME          STATUS   ROLES           AGE   VERSION   ENV
k8s-master    Ready    control-plane   14d   v1.35.7
k8s-worker1   Ready    <none>          14d   v1.35.7
k8s-worker2   Ready    <none>          14d   v1.35.7
```

**Label 설정 (k8s-worker1: hardware=general, k8s-worker2: hardware=gpu)**

```
[root@k8s-master ~]# kubectl  label  nodes k8s-worker1  hardware=general
node/k8s-worker1 labeled

[root@k8s-master ~]# kubectl  label  nodes k8s-worker2  hardware=gpu
node/k8s-worker2 labeled

[root@k8s-master ~]# kubectl  get  nodes  -L  hardware
NAME          	STATUS	ROLES            AGE   VERSION   HARDWARE
k8s-master    	Ready    	control-plane   14d     v1.35.7
k8s-worker1   	Ready    	<none>           14d     v1.35.7      general
k8s-worker2	Ready    	<none>           14d     v1.35.7      gpu
```

**Pod에 nodeSelector 적용**

```yaml
[root@k8s-master ~]# vi  gpu-pod-nodeselector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod

spec:
  nodeSelector:
    hardware: gpu

  containers:
  - name: nginx-container
    image: nginx:1.31
```

```
[root@k8s-master ~]# kubectl  apply  -f  gpu-pod-nodeselector.yaml
pod/gpu-pod created


[root@k8s-master ~]# kubectl  get  pods  -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
gpu-pod    1/1        Running      0                34s     10.244.2.32   k8s-worker2   <none>           <none>
```

```yaml
nodeSelector:
  hardware: gpu
```

Pod에 위의 nodeSelector가 설정되어 있으므로 Scheduler는 hardware=gpu 라벨을 가진 Node만 선택한다.

**Deployment에 nodeSelector 적용**

```yaml
[root@k8s-master ~]# vi gpu-deploy-nodeselector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-deploy

spec:
  replicas: 3
  selector:
    matchLabels:
      app: gpu-web

  template:
    metadata:
      labels:
        app: gpu-web
    spec:
      nodeSelector:
        hardware: gpu
      containers:
      - name: nginx
        image: nginx:1.31
```

Deployment로 3개의 Pod를 생성해도 모두 k8s-worker2에 생성된다.

```
[root@k8s-master ~]# kubectl  apply  -f  gpu-deploy-nodeselector.yaml
deployment.apps/gpu-deploy created


[root@k8s-master ~]# kubectl  get  deployments gpu-deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
gpu-deploy   3/3     3            3           15s


[root@k8s-master ~]# kubectl  get  pods
NAME                         	READY   STATUS    RESTARTS   AGE
gpu-deploy-8449f8f588-2tnpd 	1/1         Running     0                31s
gpu-deploy-8449f8f588-djgj9 	1/1         Running     0                31s
gpu-deploy-8449f8f588-jkg8k 	1/1         Running     0                31s
gpu-pod                       	1/1         Running     0                7m11s


[root@k8s-master ~]# kubectl  get  pods -o  wide
NAME                          	READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
gpu-deploy-8449f8f588-2tnpd 	1/1         Running     0                5m5s   10.244.2.35   k8s-worker2   <none>           <none>
gpu-deploy-8449f8f588-djgj9 	1/1         Running     0                5m5s   10.244.2.34   k8s-worker2   <none>           <none>
gpu-deploy-8449f8f588-jkg8k	1/1         Running     0                5m5s   10.244.2.33   k8s-worker2   <none>           <none>
gpu-pod                       	1/1         Running     0                11m    10.244.2.32   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  describe  pod  gpu-deploy-8449f8f588-2tnpd | grep Node-
Node-Selectors:              hardware=gpu


[root@k8s-master ~]# kubectl  get  nodes -L hardware
NAME          	STATUS	ROLES           	AGE   VERSION   HARDWARE
k8s-master	Ready    	control-plane	14d     v1.35.7
k8s-worker1   	Ready    	<none>          	14d     v1.35.7      general
k8s-worker2   	Ready    	<none>          	14d     v1.35.7      gpu
```

---

## 3. Affinity / Anti-Affinity란

Affinity / Anti-Affinity는 Pod를 어떤 Node에 가깝게 또는 떨어지게 배치할지 정하는 규칙이다. Affinity는 특정 조건을 만족하는 대상과 가깝게 배치하도록 하는 개념이다. Anti-Affinity는 특정 조건을 만족하는 대상과 떨어지게 배치하도록 하는 개념이다.

nodeSelector가 특정 Node의 Label을 기준으로 단순하게 배치 위치를 제한한다면, Affinity / Anti-Affinity는 Node 또는 다른 Pod와의 관계를 이용하여 더 세밀하게 배치할 수 있다.

- Affinity = 가까이 배치
- Anti-Affinity = 떨어뜨려 배치

**nodeSelector의 한계**

nodeSelector는 간단하게 특정 Label을 가진 Node를 선택할 수 있지만 한계가 있다.
- 복잡한 조건 표현에 제한이 있음
- 선호(prefer) 표현 불가
- 특정 Pod와 같은 위치에 배치하는 조건 표현 불가
- 특정 Pod와 떨어져 배치하는 조건 표현 불가
- Node와 Pod 사이의 관계를 이용한 배치 표현 불가

Affinity / Anti-Affinity는 이러한 배치 관계를 더 세밀하게 설정하기 위해 사용한다.

**Affinity의 종류**

Affinity는 크게 두 가지 기준으로 나눌 수 있다.

1) Node Affinity
- Node의 라벨을 기준으로 Pod를 어떤 Node에 배치할지 결정
- nodeSelector보다 복잡하고 유연한 조건 설정 가능
- required / preferred를 이용하여 강제 또는 선호 조건 설정 가능

2) Pod Affinity / Pod Anti-Affinity
- 다른 Pod와의 관계를 기준으로 배치
- 특정 Pod와 같은 위치에 배치하거나 떨어뜨려 배치
- 고가용성 및 Pod 분산 배치에 활용

**Node Affinity (nodeSelector의 확장)**

Node Affinity는 Pod를 특정 Node에 배치하고 싶은 조건을 더 자세하게 표현할 수 있다. Node의 Label을 기준으로 어떤 Node에 배치할지 결정한다.

Node Affinity에는 두 가지 방식이 있다.

- requiredDuringSchedulingIgnoredDuringExecution
 - 조건을 반드시 만족해야 배치 (강제 조건)
 - 조건을 만족하지 않는 Node에는 배치하지 않음
 - 만족하는 Node가 없으면 Pod는 Pending 상태가 될 수 있음

- preferredDuringSchedulingIgnoredDuringExecution
 - 가능하면 조건을 만족하는 Node로 배치 (선호 조건)
 - 조건을 만족하는 Node를 우선적으로 선호
 - 조건을 만족하는 Node가 없어도 다른 조건을 만족하는 Node에 배치 가능

이것이 nodeSelector와 Node Affinity의 가장 큰 차이 중 하나다.

- nodeSelector = 단순한 Label 일치 조건
- required Affinity = 반드시 만족해야 하는 조건
- preferred Affinity = 가능하면 만족했으면 하는 조건

Node Affinity의 preferred 조건에서는 weight를 사용할 수 있다.
- weight = 해당 preferred 조건의 선호도
- weight는 1~100 사이의 값을 사용
- weight가 높을수록 해당 조건을 더 선호
- weight가 높다고 해당 Node에 반드시 배치되는 것은 아님

**Pod Affinity / Anti-Affinity**

Pod Affinity / Anti-Affinity는 Node의 Label이 아니라 다른 Pod와의 관계를 기준으로 배치 위치를 결정한다.

1) Pod Affinity
- 특정 Pod와 같은 Node 또는 지정한 토폴로지 영역에 같이 배치하고 싶다
- 특정 Pod가 있는 위치를 기준으로 가까이 배치
- 관련 Pod를 가까이 배치하여 네트워크 지연 등을 줄이는 데 활용
- 예) 웹 Pod와 캐시 Pod를 같은 Node에 배치 — 두 Pod를 가까운 위치에 배치하여 Pod 간 통신 거리를 줄이는 목적

2) Pod Anti-Affinity
- 특정 Pod와 같은 Node 또는 지정한 토폴로지 영역에 배치하지 않도록 설정
- 동일한 서비스의 Pod를 서로 다른 Node에 분산 배치할 때 사용
- 하나의 Node에 장애가 발생해도 다른 Node의 Pod가 서비스를 유지할 수 있도록 함
- 장애 대비 및 고가용성 확보에 중요
- 예) 같은 app Pod를 서로 다른 Node에 분산 배치 — 하나의 Node에 여러 Pod가 몰리는 것을 방지하고, 특정 Node 장애 시 모든 Pod가 동시에 장애나는 상황을 줄임

**강제 vs 선호 개념**

Affinity / Anti-Affinity에도 강제 조건과 선호 조건을 설정할 수 있다.

1) required (강제)
- 조건을 반드시 만족해야 함
- 조건을 만족하지 못하면 해당 Node에 배치할 수 없음
- 만족하는 Node가 없다면 Pod는 Pending 상태가 될 수 있음

2) preferred (선호)
- 가능하면 조건을 만족하는 곳에 배치
- 조건을 만족하지 않아도 다른 Node에 배치 가능
- 스케줄러가 해당 조건을 선호도로 참고
- preferred에서는 weight를 이용하여 선호도를 지정할 수 있음

nodeSelector는 특정 Label을 만족해야 하는 강제적인 Node 선택 방식이다. Affinity / Anti-Affinity는 강제(required)와 선호(preferred)를 모두 설정할 수 있다.

**Affinity / Anti-Affinity를 사용하는 이유**

쿠버네티스에서는 단순히 Pod를 아무 Node에나 배치하는 것보다 Pod와 Node의 특성 및 Pod 간 관계를 고려하여 배치해야 하는 경우가 있다.
- 특정 종류의 Node에 Pod 배치
- 특정 Pod와 가까이 배치
- 특정 Pod와 떨어뜨려 배치
- Pod를 여러 Node에 분산
- 장애 발생 시 서비스 영향 최소화
- 네트워크 지연 최소화
- 고가용성 확보

---

## 4. Node Affinity 실습

**EX1) required 조건으로 특정 환경(env)을 반드시 만족하는 노드에만 Pod 배치하기**

```
[root@k8s-master ~]# kubectl  label  node  k8s-worker1  env=pord  disktype=ssd
node/k8s-worker1 labeled

[root@k8s-master ~]# kubectl  label  node  k8s-worker2  env=dev  disktype=hdd
node/k8s-worker2 labeled
```

- requiredDuringSchedulingIgnoredDuringExecution : 조건을 반드시 만족해야 배치 (강제 조건)
- preferredDuringSchedulingIgnoredDuringExecution : 가능하면 조건을 만족하는 Node로 배치 (선호 조건)

```yaml
[root@k8s-master ~]# vi step1-required.yaml
apiVersion: v1
kind: Pod
metadata:
  name: step1-required
spec:
  containers:
  - name: nginx
    image: nginx:1.31

  affinity:
    nodeAffinity:	
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values: [prod]
```

```
[root@k8s-master ~]# kubectl  apply  -f  step1-required.yaml
pod/step1-required created
```

**nodeSelectorTerms와 matchExpressions의 논리 관계**

- nodeSelectorTerms : 조건 묶음(세트). 여러 개를 나열할 수 있다. 이 묶음들 사이는 OR(또는) 관계다.
- matchExpressions : 한 묶음(term) 안에 들어가는 실제 조건들이다. 여러 개를 나열할 수 있다. 같은 term 안에서는 AND(그리고) 관계다.

```
# - matchExpressions: 한 덩어리가 term 1개다.
# term이 여러 개면 그 term들끼리는 OR다.
```

예제 1) AND만 있는 경우(TERM이 1개)
```
nodeSelectorTerms:
- matchExpressions:
    A
    B
    C

-논리식 : A AND B AND C
-A도 만족하고, B도 만족하고, C도 만족하는 노드만 가능
```

예제 2) OR만 있는 경우(TERM이 여러 개, 각 term 조건 1개)
```
nodeSelectorTerms:
- matchExpressions:
  A
- matchExpressions:
  B
- matchExpressions:
  C

-논리식 : A OR B OR C
-A만 만족해도 되고, B만 만족해도 되고, C만 만족해도 된다.
```

예제 3) AND 묶음 OR AND 묶음 (가장 흔한 형태)
```
nodeSelectorTerms:
- matchExpressions::
    A
    B
- matchExpressions:
    C
    D

-논리식 : (A AND B) OR (C AND D)
# (A와 B를 둘 다 만족하는 노드)이거나
# (C와 D를 둘 다 만족하는 노드)이면 된다
```

```
[root@k8s-master ~]# kubectl  get   pods
NAME             READY   STATUS    RESTARTS   AGE
step1-required   1/1        Running      0                3m44s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME             READY   STATUS    RESTARTS   AGE     IP               NODE            NOMINATED NODE   READINESS GATES
step1-required   1/1        Running     0                 6m20s   10.244.1.28   k8s-worker1   <none>                   <none>


[root@k8s-master ~]# kubectl  get  nodes  -L  env,disktype
NAME          	STATUS   ROLES           	AGE   VERSION   ENV    DISKTYPE
k8s-master    	Ready      control-plane   	14d     v1.35.7
k8s-worker1   	Ready      <none>          	14d     v1.35.7      prod    ssd
k8s-worker2	Ready      <none>          	14d     v1.35.7      dev     hdd


[root@k8s-master ~]# kubectl  describe  nodes k k8s-worker1
Name:     		k8s-worker1
Roles:              	<none>
Labels:             	beta.kubernetes.io/arch=amd64
                    	beta.kubernetes.io/os=linux
                    	disktype=ssd
                    	env=prod
                    	hardware=general
                    	kubernetes.io/arch=amd64
                    	kubernetes.io/hostname=k8s-worker1
                    	kubernetes.io/os=linux
~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~

[root@k8s-master ~]# kubectl  delete  pods  step1-required
pod "step1-required" deleted from default namespace
```

**Deployment에 required Node Affinity 적용**

```yaml
[root@k8s-master ~]# vi step1-required-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: step1-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - prod
```

Deployment를 사용하여 Pod를 3개 생성시 모두 k8s-worker1에 생성된다.

```
[root@k8s-master ~]# kubectl  apply  -f  step1-required-deploy.yaml
deployment.apps/step1-deploy created


[root@k8s-master ~]# kubectl  get  deployments  step1-deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
step1-deploy   3/3     3            3           14s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                            		READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
step1-deploy-74979754fb-72w9x   	1/1        Running     0                 28s    10.244.1.30   k8s-worker1   <none>           <none>
step1-deploy-74979754fb-dczzw   	1/1        Running     0                 28s    10.244.1.31   k8s-worker1   <none>           <none>
step1-deploy-74979754fb-stvtd   	1/1        Running     0                 28s    10.244.1.29   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  get  nodes  -L  env,disktype
NAME           STATUS   ROLES           AGE   VERSION   ENV    DISKTYPE
k8s-master     Ready      control-plane	14d     v1.35.7
k8s-worker1   Ready      <none>          	14d     v1.35.7      prod    ssd
k8s-worker2   Ready      <none>          	14d     v1.35.7      dev     hdd


[root@k8s-master ~]# kubectl  delete  -f  step1-required-deploy.yaml
deployment.apps "step1-deploy" deleted from default namespace
```

**EX2) matchExpressions를 사용하여 여러 Node 조건을 AND 방식으로 동시에 적용해 Pod 배치하기**

```
[root@k8s-master ~]# cp  ./step1-required-deploy.yaml  ./step2-and-required.yaml
```

```yaml
[root@k8s-master ~]# vi step2-and-required.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: step2-deploy	# step1-deploy  -->  step2-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - dev		# prod  -->  dev

              - key: disktype
                operator: In
                values:
                - hdd		# ssd이면 Pending상태가 된다.
```

```
[root@k8s-master ~]# kubectl  apply  -f  step2-and-required.yaml
deployment.apps/step2-deploy created


[root@k8s-master ~]# kubectl  get  pods   -o  wide
NAME                           		READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
step2-deploy-864db8984-584g8   	1/1        Running      0                19s    10.244.2.37   k8s-worker2   <none>                    <none>
step2-deploy-864db8984-pz722   	1/1        Running      0                19s    10.244.2.36   k8s-worker2   <none>                    <none>
step2-deploy-864db8984-zf7h8   	1/1        Running      0                19s    10.244.2.38   k8s-worker2   <none>                    <none>


[root@k8s-master ~]# kubectl  get  nodes  -L  env,disktype
NAME           STATUS   ROLES           AGE   VERSION   ENV    DISKTYPE
k8s-master     Ready      control-plane	14d     v1.35.7
k8s-worker1   Ready      <none>          	14d     v1.35.7      prod    ssd
k8s-worker2   Ready      <none>          	14d     v1.35.7      dev     hdd
```

조건을 dev/ssd로 바꾸면 (기존 hdd에서 ssd로 변경) 조건을 만족하는 Node가 없어 모든 Pod가 Pending 상태가 된다.

```yaml
[root@k8s-master ~]# vi step2-and-required.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: step2-deploy2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.29.1

      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - dev

              - key: disktype
                operator: In
                values:
                - ssd		# hdd에서 ssd로 변경
```

```
[root@k8s-master ~]# kubectl  delete  -f  step2-and-required.yaml
deployment.apps "step2-deploy" deleted from default namespace


[root@k8s-master ~]# kubectl  apply  -f  step2-and-required.yaml
deployment.apps/step2-deploy created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                            		READY   STATUS    RESTARTS   AGE   IP         NODE     NOMINATED NODE   READINESS GATES
step2-deploy-5d65846458-9k2dd   	0/1         Pending     0                76s    <none>   <none>   <none>           	  <none>
step2-deploy-5d65846458-mdhjf   	0/1         Pending     0                76s    <none>   <none>   <none>           	  <none>
step2-deploy-5d65846458-rdd4b   	0/1         Pending     0                76s    <none>   <none>   <none>           	  <none>


[root@k8s-master ~]# kubectl  delete  -f  step2-and-required.yaml
deployment.apps "step2-deploy" deleted from default namespace
```

**EX3) matchExpressions를 사용하여 여러 Node 조건을 OR 방식으로 동시에 적용해 Pod 배치하기**

```
[root@k8s-master ~]# cp  ./step1-required-deploy.yaml  ./step3-or-required.yaml
```

```yaml
[root@k8s-master ~]# vi  step3-or-required.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: step3-deploy	# step1-deploy  -->  step3-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:

            - matchExpressions:
              - key: env
                operator: In
                values:
                - prod

            - matchExpressions:		#  추가 설정
              - key: disktype		#  추가 설정
                operator: In		#  추가 설정
                values:			#  추가 설정
                - hdd			#  추가 설정
```

```
[root@k8s-master ~]# kubectl  apply  -f  step3-or-required.yaml
deployment.apps/step3-deploy created


[root@k8s-master ~]# kubectl  get  nodes  -L  env,disktype
NAME          	STATUS	   ROLES           	AGE   VERSION   ENV    DISKTYPE
k8s-master    	Ready    	   control-plane   	14d     v1.35.7
k8s-worker1   	Ready    	   <none>          	14d     v1.35.7      prod    ssd
k8s-worker2	Ready    	   <none>          	14d     v1.35.7      dev     hdd


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                           		READY   STATUS    RESTARTS   AGE   IP               NODE          	NOMINATED NODE   READINESS GATES
step3-deploy-c87c87575-7gvw2   	1/1         Running     0                30s     10.244.2.40   k8s-worker2   	<none>           	  <none>
step3-deploy-c87c87575-9p79j   	1/1         Running     0                30s     10.244.1.32   k8s-worker1   	<none>           	  <none>
step3-deploy-c87c87575-rsjbp   	1/1         Running     0                30s     10.244.2.39   k8s-worker2   	<none>           	  <none>


[root@k8s-master ~]# kubectl  label  nodes  k8s-worker1 env- disktype-
node/k8s-worker1 unlabeled


[root@k8s-master ~]# kubectl  label  nodes  k8s-worker2 env- disktype-
node/k8s-worker2 unlabeled
```

**EX4) 강제 조건(required)과 선호 조건(preferred)을 함께 사용하는 스케줄링**

```
[root@k8s-master ~]# kubectl  label  nodes  k8s-worker1 env=prod  region=seoul
node/k8s-worker1 labeled

[root@k8s-master ~]# kubectl  label  nodes  k8s-worker2 env=prod  region=busan
node/k8s-worker2 labeled

[root@k8s-master ~]# kubectl  label  nodes  k8s-worker3 env=dev  region=seoul
node/k8s-worker2 labeled

[root@k8s-master ~]# kubectl  get  nodes  -L  env,region
NAME          	STATUS	ROLES           AGE   VERSION   ENV    REGION
k8s-master	Ready    	control-plane   14d     v1.35.7
k8s-worker1   	Ready    	<none>           14d     v1.35.7    prod     seoul
k8s-worker2   	Ready    	<none>           14d     v1.35.7     prod    busan
```

```yaml
[root@k8s-master ~]#  vi step4-1-preferred.yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-required-preferred
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values: [prod]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 70
        preference:
          matchExpressions:
          - key: region
            operator: In
            values: [seoul]
  containers:
  - name: nginx
    image: nginx
```

- k8s-worker1 env=prod region=seoul	: required에의한 후보1 , preferred에 의해 weight: 70
- k8s-worker2 env=prod region=busan	: required에의한 후보2 , preferred에 의해 weight: 0
- k8s-worker3 env=dev   region=seoul	: required에의해 탈락

| 노드 | required(env=prod) | preferred(region=seoul) | 점수 |
|---|---|---|---|
| worker1 | 만족 | 만족 | +70 |
| worker2 | 만족 | 불만족 | +0 |
| worker3 | 불만족 | 만족 | 후보 제외 |

```
[root@k8s-master ~]# kubectl  apply  -f  step4-1-preferred.yaml
pod/affinity-required-preferred created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                          	READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
affinity-required-preferred	1/1        Running      0                71s     10.244.1.4   k8s-worker1   <none>                    <none>


[root@k8s-master ~]# kubectl  delete  -f  step4-1-preferred.yaml
pod "affinity-required-preferred" deleted from default namespace
```

Deployment로 동일한 조건을 적용해도 결과는 같다.

```yaml
[root@k8s-master ~]# vi step4-2-preferred.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-required-preferred
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx-affinity
  template:
    metadata:
      labels:
        app: nginx-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values: [prod]
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 70
            preference:
              matchExpressions:
              - key: region
                operator: In
                values: [seoul]
      containers:
      - name: nginx
        image: nginx:1.29.1
        ports:
        - containerPort: 80
```

```
[root@k8s-master ~]# kubectl  apply  -f  step4-2-preferred.yaml
deployment.apps/affinity-required-preferred created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                           		READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
affinity-required-preferred-7c478d5784-4vnjf   	1/1         Running     0                3s      10.244.1.5   k8s-worker1   <none>           <none>
affinity-required-preferred-7c478d5784-fstvm   	1/1         Running     0                3s      10.244.1.8   k8s-worker1   <none>           <none>
affinity-required-preferred-7c478d5784-vg2xb   	1/1         Running     0                3s      10.244.1.7   k8s-worker1   <none>           <none>
affinity-required-preferred-7c478d5784-xqwld   	1/1         Running     0                3s      10.244.1.6   k8s-worker1   <none>           <none>
affinity-required-preferred-7c478d5784-zgsd2   	1/1         Running     0                3s      10.244.1.9   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  delete  -f  step4-2-preferred.yaml
```

모든 후보 노드를 대상으로 각 preferred 조건을 검사하고, 조건을 만족할 때마다 weight를 더한다. 최종 점수가 가장 높은 노드를 선택한다.

```
[root@k8s-master ~]# kubectl label node k8s-worker1 env=prod region=seoul disk=ssd --overwrite

[root@k8s-master ~]# kubectl label node k8s-worker2 env=prod region=busan disk=ssd --overwrite

[root@k8s-master ~]# kubectl label node k8s-worker3 env=dev region=seoul disk=hdd --overwrite
```

```yaml
[root@k8s-master ~]# vi step4-3-preferred.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-multi-weight
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-affinity-mw
  template:
    metadata:
      labels:
        app: nginx-affinity-mw
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values: [prod]

          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            preference:
              matchExpressions:
              - key: region
                operator: In
                values: [seoul]

          - weight: 30
            preference:
              matchExpressions:
              - key: disk
                operator: In
                values: [ssd]

      containers:
      - name: nginx
        image: nginx:1.29.1
        ports:
        - containerPort: 80
```

Example01) pod 10개를 생성해야 한다.

- k8s-worker1 : env=prod(required 통과), region=seoul(+50), disk=ssd(+30) → 총점 80
- k8s-worker2 : env=prod(required 통과), region=busan(+0), disk=ssd(+30) → 총점 30
- k8s-worker3 : env=prod(required 통과), region=seoul(+50), disk=hdd(+0) → 총점 50

우선순위 : k8s-worker1(80) > k8s-worker3(50) > k8s-worker2(30)

따라서 Pod는 가장 높은 점수인 k8s-worker1에 우선적으로 생성된다. 만약 Pod를 계속 생성하다가 k8s-worker1의 리소스가 부족해 더 이상 Pod를 생성할 수 없다면 다음으로 점수가 높은 k8s-worker3(Weight 50)에 Pod가 생성된다. 7번째 pod 생성시 k8s-worker1의 리소스가 부족하면 다음으로 가중치가 높은 k8s-worker3 노드에 pod가 생성된다.

Example02) pod 10개를 생성해야 한다.

- k8s-worker1 : env=prod(required 통과), region=seoul(+50), disk=ssd(+30) → 총점 80
- k8s-worker2 : env=prod(required 통과), region=busan(+0), disk=ssd(+30) → 총점 30
- k8s-worker3 : env=dev(required 조건 불만족) → 후보에서 제외

우선순위 : k8s-worker1(80) > k8s-worker2(30) > k8s-worker3(제외)

따라서 Pod는 k8s-worker1에 우선적으로 생성된다.

```
[root@k8s-master ~]# kubectl  apply  -f  step4-3-preferred.yaml


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                     	READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
affinity-multi-weight-6f759c5f74-4cp6k	1/1        Running     0                 2s      10.244.1.10   k8s-worker1   <none>           <none>
affinity-multi-weight-6f759c5f74-gd55g	1/1        Running     0                 2s      10.244.1.11   k8s-worker1   <none>           <none>
affinity-multi-weight-6f759c5f74-vsph8	1/1        Running     0                 2s      10.244.1.12   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  delete  -f  step4-3-preferred.yaml
```

---

## 5. Pod Affinity 실습

**EX1) 프론트엔드 서버와 캐시 서버의 관계 이해 (Pod Affinity 미적용)**

- 프론트엔드 서버 : 사용자의 요청을 직접 받는다. HTML, API 요청, 이미지 등을 처리한다. 요청 처리 중 자주 캐시 서버를 호출한다.
- 캐시 서버 (Redis, Memcached 등) : DB 대신 빠르게 값을 꺼내주는 역할. 프론트엔드가 요청할 때마다 계속 접근한다. 캐시 서버는 응답 속도가 핵심이다.
- 두 서버의 관계 : 통신 빈도가 매우 높다. 응답 지연(latency)에 매우 민감하다. 즉, 같은 노드에 있으면 성능이 유리한 관계다.

```yaml
[root@k8s-master ~]# vi affinity-step1-cache.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache
  labels:
    app: cache
spec:
  containers:
  - name: nginx
    image: nginx:1.29.1
```

```yaml
[root@k8s-master ~]# vi affinity-step1-frontend.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f  affinity-step1-cache.yaml
pod/cache created

[root@k8s-master ~]# kubectl  apply  -f  affinity-step1-frontend.yaml
pod/frontend created

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME  	READY   STATUS    RESTARTS   AGE   IP               NODE          	 NOMINATED NODE   READINESS GATES
cache   	1/1        Running      0                17s     10.244.2.9    k8s-worker2	<none>                    <none>
frontend	1/1        Running      0                14s     10.244.1.18   k8s-worker1   	<none>                    <none>
```

affinity를 사용하지 않으면 두 개의 서버가 서로 다른 노드에 생성된다. (상황에 따라 같은 노드에 생성될 수 도 있다.)

**EX2) Pod Affinity 적용 - frontend를 cache와 같은 Node에 배치**

```
   # 기존 frontend Pod 삭제
[root@k8s-master ~]# kubectl  delete  pod  frontend
pod "frontend" deleted from default namespace
```

```yaml
   # frontend에 Pod Affinity 설정
[root@k8s-master ~]# vi affinity-step2-frontend.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    app: frontend

spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:	# required = 강제
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [cache]
        topologyKey: kubernetes.io/hostname		# app: cache lable을 갖은 Pod와 Hostname이 같은 노드에 배치

  containers:

  - name: nginx
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  get pods  -o  wide
NAME    READY   STATUS    RESTARTS   AGE   IP             NODE            NOMINATED NODE   READINESS GATES
cache     1/1        Running      0                30m    10.244.2.9   k8s-worker2   <none>                    <none>


[root@k8s-master ~]# kubectl  apply  -f  affinity-step2-frontend.yaml
pod/frontend created


	# app: cache Pod가 있는 노드에 frontend Pod가 생성된다.
[root@k8s-master ~]# kubectl  get pods  -o  wide
NAME  	READY   STATUS    RESTARTS   AGE	  IP              NODE            NOMINATED NODE   READINESS GATES
cache 	1/1        Running      0                33m	  10.244.2.9    k8s-worker2   <none>           <none>
frontend	1/1        Running      0                22s 	  10.244.2.10   k8s-worker2   <none>           <none>
```

**EX3) cache Pod가 없는 상태에서 frontend Pod 생성 후 확인**

```
	# cache Pod 삭제
[root@k8s-master ~]# kubectl  delete  pods  cache
pod "cache" deleted from default namespace

	# frontend Pod 삭제
[root@k8s-master ~]# kubectl  delete  pods  frontend
pod "frontend" deleted from default namespace

	# frontend Pod만 생성
[root@k8s-master ~]# kubectl  apply  -f  affinity-step2-frontend.yaml
pod/frontend created


[root@k8s-master ~]# kubectl  get pods  -o  wide
NAME       READY   STATUS    RESTARTS   AGE   IP         NODE     NOMINATED NODE   READINESS GATES
frontend    0/1         Pending     0                2s      <none>   <none>   <none>                    <none>
```

frontend Pod가 Pending되는 이유는 app=cache 라벨을 가진 Pod가 현재 존재하지 않기 때문이다. requiredDuringSchedulingIgnoredDuringExecution은 강제 조건이므로, frontend Pod는 반드시 app=cache Pod가 실행 중인 노드와 같은 노드에 배치되어야 한다. 하지만 조건을 만족하는 cache Pod가 없기 때문에 스케줄러가 배치할 노드를 찾지 못하고 Pending 상태가 된다.

**EX4) 다시 cache Pod를 다시 생성하여 Pending frontend가 실행되는지 확인**

```
[root@k8s-master ~]# kubectl apply -f affinity-step1-cache.yaml
pod/cache created


[root@k8s-master ~]# kubectl  get pods  -o  wide
NAME   	READY   STATUS    RESTARTS   AGE	  IP               NODE           NOMINATED NODE   READINESS GATES
cache  	1/1         Running     0                8s  	  10.244.2.12   k8s-worker2   <none>           <none>
frontend	1/1         Running     0                3m7s	  10.244.2.11   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  delete  pods --all
pod "cache" deleted from default namespace
pod "frontend" deleted from default namespace
```

**EX5) Preferred Pod Affinity + weight 실습**
- cache Pod의 Label은 app=cache
- frontend Pod는 app=cache Pod가 있는 Node를 선호한다.
- preferredDuringSchedulingIgnoredDuringExecution 사용, weight: 80, topologyKey: kubernetes.io/hostname
- frontend Pod는 2개 생성한다.

```yaml
[root@k8s-master ~]# vi affinity-step5-cache.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache
  labels:
    app: cache
spec:
  containers:
  - name: nginx
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl apply -f affinity-step5-cache.yaml
pod/cache created


[root@k8s-master ~]# kubectl get pod cache -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
cache    1/1         Running      0                4s      10.244.2.13   k8s-worker2   <none>                  <none>
```

```yaml
[root@k8s-master ~]# vi affinity-step5-frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend

  template:
    metadata:
      labels:
        app: frontend

    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:

          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [cache]
              topologyKey: kubernetes.io/hostname

      containers:
      - name: nginx
        image: nginx:1.29.1
```

```
	# Deployment로 Pod 생성전 cache Pod는 k8s-worker2 노드에 생성되어있다.
[root@k8s-master ~]# kubectl get pod cache -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
cache    1/1         Running      0                4s      10.244.2.13   k8s-worker2   <none>                  <none>


[root@k8s-master ~]# kubectl apply -f affinity-step5-frontend.yaml
deployment.apps/frontend created


[root@k8s-master ~]# kubectl get pods -o wide
NAME                        	READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
cache                       	1/1         Running    0                 8m45s   10.244.2.13   k8s-worker2   <none>                   <none>
frontend-5555496466-gqxz	1/1         Running    0                 2m30s   10.244.2.15   k8s-worker2   <none>                   <none>
frontend-5555496466-h9j6n	1/1         Running    0                 2m30s   10.244.2.14   k8s-worker2   <none>                   <none>
```

cache Pod가 현재 k8s-worker2에서 실행 중이기 때문에 frontend Pod 2개도 k8s-worker2에 배치되는 것을 우선적으로 선호한다. weight: 80은 k8s-worker2에 스케줄링 점수 80점을 추가로 부여한다는 의미다. 따라서 일반적으로 k8s-worker2에 Pod가 생성될 가능성이 높다. 하지만 preferred는 강제 조건이 아니라 선호 조건이기때문에 CPU/Memory 부족이나 다른 스케줄링 조건 때문에 k8s-worker2가 적합하지 않다면 frontend Pod가 k8s-worker1에 배치될 수도 있다.

```
[root@k8s-master ~]# kubectl  scale  deployment  frontend  --replicas=5
deployment.apps/frontend scaled


[root@k8s-master ~]# kubectl get pods -o wide
NAME                        	READY   STATUS    RESTARTS   AGE    IP             NODE          NOMINATED NODE   READINESS GATES
cache                       	1/1         Running    0                 12m    10.244.2.13   k8s-worker2   <none>           <none>
frontend-5555496466-gqxzr 	1/1         Running    0                 6m9s   10.244.2.15   k8s-worker2   <none>           <none>
frontend-5555496466-h9j6n   	1/1         Running    0                 6m9s   10.244.2.14   k8s-worker2   <none>           <none>
frontend-5555496466-kbxbr   	1/1         Running    0                 3s      10.244.2.16   k8s-worker2   <none>           <none>
frontend-5555496466-vbk9z   	1/1         Running    0                 3s      10.244.2.17   k8s-worker2   <none>           <none>
frontend-5555496466-wth9h   	1/1         Running    0                 3s      10.244.2.18   k8s-worker2   <none>           <none>
```

Pod Affinity는 같이 모이는것을 선호하기 때문에 cache Pod가 있는 Node로 frontend Pod들이 모이는 방향으로 배치된다.

```
[root@k8s-master ~]# kubectl  delete   pods cache
pod "cache" deleted from default namespace

[root@k8s-master ~]# kubectl  delete  deployments  frontend
deployment.apps "frontend" deleted from default namespace
```

---

## 6. Pod Anti-Affinity 실습

복제 Pod가 같은 노드에 같이 있지 못하도록 강제한다. 고가용성(HA)에서 가장 기본이 되는 분산 규칙을 만든다.

**EX1) required Pod Anti-Affinity로 2개 Pod를 반드시 다른 노드로 분산시키기**
- Deployment 이름은 web-antiaffinity로 생성한다.
- Pod의 라벨은 app=web으로 설정한다.
- 복제본 수는 2개로 설정한다.
- 같은 app=web 라벨을 가진 Pod끼리는 반드시 서로 다른 Node에 배치되도록 설정한다.
- requiredDuringSchedulingIgnoredDuringExecution을 사용한다.
- topologyKey는 kubernetes.io/hostname을 사용한다.
- 컨테이너 이미지는 nginx:1.29.1을 사용한다.
- 생성 후 2개의 Pod가 서로 다른 Node에 배치되었는지 확인한다.

```yaml
[root@k8s-master ~]# vi anti-step1-required.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-antiaffinity

spec:
  replicas: 2
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [web]

            topologyKey: kubernetes.io/hostname

      containers:
      - name: nginx
        image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f  anti-step1-required.yaml
deployment.apps/web-antiaffinity created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
web-antiaffinity-6fc4ddbfd7-dxzrv   	1/1        Running      0                24s    10.244.2.19   k8s-worker2   <none>                   <none>
web-antiaffinity-6fc4ddbfd7-vppn5   	1/1        Running      0                24s    10.244.1.19   k8s-worker1   <none>                   <none>
```

app=web Pod 2개가 서로 다른 Node에 배치된 것을 확인한다. required Pod Anti-Affinity 조건이 정상적으로 적용된 상태이며, 같은 app=web Pod끼리는 같은 Node에 배치될 수 없다.

```
	# 파드 개수 증가
[root@k8s-master ~]# kubectl  scale  deployment  web-antiaffinity  --replicas=3
deployment.apps/web-antiaffinity scaled


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS	 RESTARTS   AGE     IP               NODE            NOMINATED NODE   READINESS GATES
web-antiaffinity-6fc4ddbfd7-dxzrv   	1/1        Running	 0                 3m47s   10.244.2.19   k8s-worker2   <none>                    <none>
web-antiaffinity-6fc4ddbfd7-vlqqb   	0/1        Pending   	 0                 13s       <none>       <none>          <none>                    <none>
web-antiaffinity-6fc4ddbfd7-vppn5   	1/1        Running   	 0                 3m47s   10.244.1.19   k8s-worker1   <none>                    <none>
```

기존 Pod 2개는 각각 k8s-worker1, k8s-worker2에서 정상 실행 중이다. Worker Node가 2개뿐이고, required Pod Anti-Affinity에 의해 같은 app=web Pod를 같은 Node에 배치할 수 없다. 따라서 세 번째 Pod가 배치될 수 있는 Node가 없어 IP와 NODE가 `<none>`으로 표시된다. Node가 하나 더 추가되거나 Anti-Affinity 조건이 변경되기 전까지 세 번째 Pod는 Pending 상태를 유지한다.

```
[root@k8s-master ~]# kubectl  delete  deployments  web-antiaffinity
deployment.apps "web-antiaffinity" deleted from default namespace
```

**EX2) preferred Pod Anti-Affinity로 2개 Pod를 반드시 다른 노드로 분산시키기**
- Pod 라벨은 app=web
- 같은 app=web Pod끼리는 가능하면 서로 다른 Node에 배치
- preferredDuringSchedulingIgnoredDuringExecution 사용, weight: 80, topologyKey: kubernetes.io/hostname
- nginx:1.29.1 사용

```yaml
[root@k8s-master ~]# vi anti-step2-required.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anti-aff

spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-step2

  template:
    metadata:
      labels:
        app: web-step2

    spec:
      containers:
      - name: nginx
        image: nginx
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [web-step2]
              topologyKey: kubernetes.io/hostname
```

```
[root@k8s-master ~]# kubectl  apply  -f  anti-step2-required.yaml
deployment.apps/web-step2 created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                        	READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
anti-aff-9c44899d7-46gd5	1/1         Running     0                6s      10.244.2.20   k8s-worker2   <none>                   <none>
anti-aff-9c44899d7-qzd8n 	1/1         Running     0                6s      10.244.1.20   k8s-worker1   <none>                   <none>


	# 파드 개수 증가
[root@k8s-master ~]# kubectl  scale  deployment  web-step2  --replicas=5
deployment.apps/web-step2 scaled


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                        	READY   STATUS    RESTARTS   AGE   IP              NODE             NOMINATED NODE   READINESS GATES
anti-aff-9c44899d7-46gd5 	1/1         Running     0                5m     10.244.2.20   k8s-worker2     <none>                   <none>
anti-aff-9c44899d7-bj8sf 	1/1         Running     0                7s      10.244.2.22   k8s-worker2     <none>                   <none>
anti-aff-9c44899d7-l2sjp 	1/1         Running     0                7s      10.244.1.21   k8s-worker1     <none>                   <none>
anti-aff-9c44899d7-llzlr 	1/1         Running     0                7s      10.244.2.21   k8s-worker2     <none>                   <none>
anti-aff-9c44899d7-qzd8n	1/1         Running     0                5m     10.244.1.20   k8s-worker1     <none>                   <none>


[root@k8s-master ~]# kubectl  delete  deployments  anti-aff
deployment.apps "web-step2" deleted from default namespace
```

preferred는 강제 조건이 아니므로, Worker Node가 2개뿐인 상태에서 replicas를 5로 늘리면 required와 달리 Pending 없이 Node에 몰려서 배치되는 Pod도 발생할 수 있다. required Pod Anti-Affinity처럼 절대 같은 Node에 배치되지 않는 것이 아니라, 가능하면 분산하려는 선호로 동작한다.

---

## 7. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- Pod가 계속 `Pending` 상태라면 먼저 `kubectl describe pod <이름>`의 Events를 확인해 스케줄링 실패 사유(FailedScheduling)를 본다. nodeSelector/required Affinity 조건을 만족하는 Node가 없는 경우가 가장 흔한 원인이다.
- `kubectl get nodes -L <key>`로 각 Node의 실제 라벨 값을 확인해, nodeSelector나 matchExpressions의 key/value 오타 여부를 점검한다. 라벨은 대소문자와 철자가 정확히 일치해야 매칭된다.
- `nodeSelectorTerms` 사이는 OR, 같은 term 안의 `matchExpressions`는 AND라는 점을 혼동하면 의도와 다른 Node에 배치되거나 조건이 지나치게 좁아져 Pending이 발생할 수 있다.
- required Node Affinity는 만족하는 Node가 하나도 없으면 즉시 Pending이 되므로, 운영 환경에서는 required보다 preferred + weight 조합으로 유연성을 확보하는 것이 안전할 때가 많다.
- Pod Affinity의 required 조건은 대상 Pod(labelSelector로 지정한 app 등)가 아직 존재하지 않으면 Pending이 된다. 대상 Pod를 먼저 생성한 뒤 Affinity Pod를 생성하거나, preferred로 완화하는 방식을 고려한다.
- required Pod Anti-Affinity는 topologyKey 기준으로 같은 위치에 중복 배치를 막기 때문에, Node 수보다 replicas가 많아지면 초과분이 Pending 상태로 남는다. Node를 늘리거나 preferred로 전환해 해결한다.
- Node의 Label을 나중에 바꾸거나 삭제해도 이미 스케줄링되어 실행 중인 Pod는 자동으로 재배치되지 않는다. nodeSelector/Affinity는 최초 스케줄링 시점에만 평가된다.
- `kubectl describe pod <이름> | grep Node-` 또는 `kubectl describe pod <이름>`의 Affinity 항목으로 실제 적용된 조건을 다시 확인할 수 있다.

---

>  **핵심 요약**
> - kube-scheduler는 Filtering(배치 불가 Node 제거) → Scoring(적합도 평가) → Binding(최종 선택) 순서로 Pod가 실행될 Node를 결정한다
> - nodeSelector는 Node의 Label을 기준으로 하는 가장 단순한 강제 배치 조건이며, 조건을 만족하는 Node가 없으면 Pod는 Pending 상태로 남는다
> - Node Affinity는 requiredDuringSchedulingIgnoredDuringExecution(강제)과 preferredDuringSchedulingIgnoredDuringExecution(선호, weight 기반 점수제)으로 nodeSelector보다 세밀한 조건을 표현한다
> - nodeSelectorTerms 사이는 OR, 같은 term의 matchExpressions끼리는 AND로 평가된다
> - Pod Affinity는 다른 Pod와 가까이(같은 topologyKey), Pod Anti-Affinity는 다른 Pod와 떨어뜨려 배치하는 규칙이며, 고가용성 확보와 네트워크 지연 최소화에 활용된다
> - 관련: 2.  Kubernetes - Pod 생성 · 24.  Kubernetes - Label · 26.  Kubernetes - Pod Scheduling (Taint·Toleration)
