# Kubernetes - Pod Scheduling (Taint·Toleration)

> **Tag:** #Kubernetes #PodScheduling #Taint #Toleration #Cordon #Drain #부트캠프
> **핵심 요약:** Node가 Pod를 거부하도록 막는 Taint와 이를 허용하는 Toleration의 개념·구조·Effect(NoSchedule/PreferNoSchedule/NoExecute), Taint와 Node Affinity의 차이, 그리고 Node 유지보수를 위한 Cordon·Drain의 사용법까지 실습 전체 정리

---

## 1. 개요 (Overview)

Taint & Toleration은 특정 Node에 아무 Pod나 배치되지 못하도록 제한하고, 필요한 Pod만 해당 Node에 배치할 수 있도록 하는 기능이다.

- Taint = Node에 출입 제한을 거는 것
- Toleration = Pod가 그 출입 제한을 견딜 수 있도록 허용하는 것

즉, Node가 "아무 Pod나 들어오지 마"라고 막는다면 Pod가 "나는 이 조건을 수용할 수 있어"라고 설정하면 해당 Pod는 Taint가 있는 Node에 배치될 수 있다.

**Taint를 사용하는 이유**

모든 Node를 모든 Pod가 사용하는 것이 항상 좋은 것은 아니다. 예를 들어 특정 Node를 특정 용도로 사용하고 싶을 수 있다.
- GPU 전용 Node
- 데이터베이스 전용 Node
- 중요한 시스템 Pod용 Node
- 운영 전용 Node
- 특정 팀이나 서비스 전용 Node

이때 해당 Node에 Taint를 설정하면 일반 Pod가 해당 Node에 배치되는 것을 막을 수 있다.

예시
- Node 1 = 일반 Node
- Node 2 = GPU 전용 Node

Node 2에 Taint를 설정한다.
```
kubectl  taint  nodes  k8s-worker2  gpu=true:NoSchedule
```
이제 Node 2에는 해당 Taint를 허용하지 않는 일반 Pod가 새롭게 배치되지 않는다.

**Taint의 기본 구조**

형식 : `key=value:effect`
- key = Taint의 이름
- value = Taint의 값
- effect = 해당 Taint가 Pod에게 적용하는 동작

예시 : `gpu=true:NoSchedule` — "gpu=true라는 제한이 걸려 있으므로 이 조건을 허용하지 않는 Pod는 배치하지 마라" 라는 의미다.

**Taint의 Effect**

Taint에서 가장 중요한 것이 effect다. 대표적으로 3가지가 있다.

- **NoSchedule**
 - Taint를 허용하는 Toleration이 없는 Pod를 새롭게 배치하지 않는다.
 - 기존에 실행 중인 Pod는 그대로 실행될 수 있다.
 - 가장 일반적으로 사용하는 Taint 방식이다.

- **PreferNoSchedule**
 - 가능하면 해당 Taint를 허용하지 않는 Pod를 배치하지 않으려고 한다.
 - 하지만 다른 조건에 따라 해당 Node에 배치될 수도 있다.
 - 강제 차단이 아니라 선호 방식이다.
 - 예를 들어 k8s-worker2에 PreferNoSchedule Taint가 설정되어 있다고 가정한다. 일반 Pod는 이 Taint를 허용하는 Toleration이 없다. 그러면 Kubernetes Scheduler는 가능하면 k8s-worker2를 피하고 k8s-worker1 같은 다른 Node에 Pod를 배치하려고 한다. 하지만 k8s-worker1에 CPU나 Memory가 부족하거나, 다른 스케줄링 조건 때문에 배치할 수 없다면 PreferNoSchedule Taint가 있는 k8s-worker2에 Pod가 배치될 수도 있다.

- **NoExecute**
 - Taint를 허용하는 Toleration이 없는 Pod를 새롭게 배치하지 않는다.
 - Taint를 허용하지 않는 기존 Pod를 해당 Node에서 제거할 수 있다.
 - 실행 중인 Pod에도 영향을 줄 수 있다는 점이 NoSchedule과 다르다.

**NoSchedule과 NoExecute의 차이**

- NoSchedule : 새로운 Pod의 배치를 제한. 기존 Pod에는 직접적인 영향이 없음
- NoExecute : 새로운 Pod의 배치를 제한. 기존 Pod도 제거될 수 있음

즉, NoSchedule = 새로 들어오는 것을 막음 / NoExecute = 새로 들어오는 것도 막고 기존 Pod도 내보낼 수 있다.

**Toleration이란**

Toleration은 Pod가 특정 Taint를 허용할 수 있도록 설정하는 것이다. 중요한 점은 Toleration이 있다고 해서 해당 Node에 반드시 배치되는 것은 아니다. Toleration의 의미는 "이 Pod는 이 Taint를 허용할 수 있다."라는 뜻이다.

- Taint = Node의 출입 제한
- Toleration = Pod가 해당 제한을 허용할 수 있음

하지만 Toleration만 설정했다고 Pod가 해당 Node를 선택하는 것은 아니다. 해당 Node에 배치되려면 Taint를 허용하고 다른 스케줄링 조건도 만족해야 한다. 특정 Node에 반드시 배치하고 싶다면 nodeSelector나 Node Affinity 등을 함께 사용할 수 있다.

**Toleration의 기본 구조**

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

의미 : key=gpu, value=true, effect=NoSchedule → 즉 `gpu=true:NoSchedule` Taint를 허용할 수 있는 Pod가 된다.

**Taint와 Toleration의 관계**

Node에 다음과 같은 Taint가 있다고 가정한다.
```
gpu=true:NoSchedule
```

- 일반 Pod : Toleration 없음 → Taint를 허용하지 못함 → 해당 Node에 새롭게 배치되지 않음
- 다음과 같은 Toleration을 가진 Pod는 gpu=true:NoSchedule Taint를 허용하여 해당 Node에 배치될 수 있음

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

Taint는 "배치 금지", Toleration은 "배치 허용" 하지만 정확하게는 Taint와 Toleration의 관계를 다음과 같이 이해해야 한다.

- Taint : 이 Taint를 허용하지 않는 Pod를 제한
- Toleration : 특정 Taint를 허용할 수 있도록 Pod에 설정

따라서 Toleration은 배치를 강제하는 기능이 아니다. Toleration이 있으면 해당 Node에 들어갈 수 있지만, 반드시 해당 Node에 들어가는 것은 아니다.

**Taint와 Node Affinity의 차이**

Taint와 Node Affinity는 서로 목적이 다르다.

- Taint : Node 입장에서 Pod를 제한
- Node Affinity : Pod 입장에서 원하는 Node를 선택

GPU Node를 일반 Pod가 사용하지 못하도록 하고 싶다면 Node에 Taint를 설정한다: `gpu=true:NoSchedule`

GPU Pod가 해당 Node를 사용할 수 있도록 하려면 Pod에 Toleration을 설정한다.

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

그런데 GPU Pod가 반드시 GPU Node에 배치되기를 원한다면 Toleration에 Node Affinity 또는 nodeSelector를 함께 사용할 수 있다.

즉, Taint & Toleration은 "누가 들어올 수 있는가"이고, Node Affinity는 "어디에 들어갈 것인가"이다.

Pod가 생성되면 스케줄러는 Node를 확인한다.
1) Node에 Taint가 있는지 확인
2) Pod에 해당 Taint를 허용하는 Toleration이 있는지 확인
3) 허용하지 못하는 Taint가 있으면 해당 Node를 후보에서 제외
4) 모든 Taint를 허용할 수 있다면 다른 스케줄링 조건을 확인
5) 자원, Affinity 등 다른 조건까지 만족하면 해당 Node가 선택될 수 있음

즉, Taint & Toleration만 보는 것이 아니라 다른 스케줄링 조건과 함께 최종 Node를 결정한다. Node의 Label을 기준으로 한 배치 제어(nodeSelector, Node Affinity)는 Kubernetes - Pod Scheduling (nodeSelector·Affinity) 문서에서 자세히 다룬다.

**전체 흐름 예시**

Node 상태
- k8s-worker1 : 일반 Node, Taint 없음
- k8s-worker2 : GPU Node, Taint: gpu=true:NoSchedule

일반 Pod (Toleration 없음)
- 결과 : worker1 → 배치 가능, worker2 → Taint 때문에 배치 불가

GPU Pod
```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```
- 결과 : worker1 → 배치 가능, worker2 → Taint 허용 가능(worker2에 배치될 수도 있음)

하지만 GPU Node에 반드시 배치하려면 Toleration과 nodeSelector 또는 Node Affinity를 함께 설정해야 한다.

---

## 2. NoSchedule Taint 실습

```
[root@k8s-master ~]# kubectl  describe  nodes  k8s-master  | grep Taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1  | grep Taint
Taints:             <none>

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker2  | grep Taint
Taints:             <none>
```

**Toleration 설정 X**

```yaml
[root@k8s-master ~]# vi taint-step1-noschedule.yaml
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-1
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
---
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-2
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f  taint-step1-noschedule.yaml
pod/taint-step1-pod-1 created
pod/taint-step1-pod-2 created

# k8s-worker1 노드, k8s-worker1 노드에 각 1개씩 생성된다.
[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                  READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
taint-step1-pod-1   1/1         Running     0                10s     10.244.2.23   k8s-worker2   <none>                    <none>
taint-step1-pod-2   1/1         Running     0                10s     10.244.1.22   k8s-worker1   <none>                    <none>


	# 모든 Pod 삭제
[root@k8s-master ~]# kubectl  delete  pods  --all
pod "taint-step1-pod-1" deleted from default namespace
pod "taint-step1-pod-2" deleted from default namespace


	# k8-worker1 노드에 taint 설정
[root@k8s-master ~]# kubectl  taint  nodes  k8s-worker1  role=db:NoSchedule
node/k8s-worker1 tainted


	# k8-worker1 노드의 taint 확인
[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1 | grep Taint
Taints:             role=db:NoSchedule


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                  READY   STATUS    RESTARTS   AGE   IP              NODE          	NOMINATED NODE   READINESS GATES
taint-step1-pod-1   1/1        Running      0                3s      10.244.2.24   k8s-worker2   	<none>           <none>
taint-step1-pod-2   1/1        Running      0                3s      10.244.2.25   k8s-worker2   	<none>           <none>
```

- k8s-worker1 노드 : Taint 설정 O (role=db:NoSchedule). role=db:NoSchedule Taint를 허용하는 Toleration이 없는 Pod는 새롭게 배치될 수 없음
- k8s-worker2 노드 : Taint 설정 X. 일반 Pod도 배치 가능
- 생성한 Pod YAML : Toleration 설정 X. role=db:NoSchedule Taint를 허용하지 못함

결과
- k8s-worker1 : role=db:NoSchedule Taint가 설정되어 있음. Pod에 Toleration이 없으므로 배치 불가
- k8s-worker2 : Taint가 없음. Pod에 Toleration이 없어도 배치 가능

스케줄러가 판단하는 과정
- k8s-worker1 : 테인트 있음 → Pod가 허용 못 함 → 배치 불가
- k8s-worker2 : 테인트 없음 → 아무 제한 없음 → 배치 가능

**Toleration 설정 O**

```yaml
[root@k8s-master ~]# vi taint-step1-noschedule.yaml
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-1
spec:
  tolerations:		# Toleration 추가 설정
  - key: "role"		# Toleration 추가 설정
    operator: "Equal"		# Toleration 추가 설정
    value: "db"		# Toleration 추가 설정
    effect: "NoSchedule"	# Toleration 추가 설정
  containers:
  - name: nginx-container
    image: nginx:1.29.1

---
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-2
spec:
  tolerations:		# Toleration 추가 설정
  - key: "role"		# Toleration 추가 설정
    operator: "Equal"		# Toleration 추가 설정
    value: "db"		# Toleration 추가 설정
    effect: "NoSchedule"	# Toleration 추가 설정
  containers:
  - name: nginx-container
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f  taint-step1-noschedule.yaml  --dry-run=client
pod/taint-step1-pod-1 configured (dry run)
pod/taint-step1-pod-2 configured (dry run)


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                READY   STATUS    RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES
taint-step1-pod-1   1/1     Running   0          7m30s   10.244.2.24   k8s-worker2   <none>                    <none>
taint-step1-pod-2   1/1     Running   0          7m30s   10.244.2.25   k8s-worker2   <none>                    <none>


[root@k8s-master ~]# kubectl  apply  -f  taint-step1-noschedule.yaml
pod/taint-step1-pod-1 configured
pod/taint-step1-pod-2 configured


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                	READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
taint-step1-pod-1 	1/1         Running     0                7m30s   10.244.2.24   k8s-worker2   <none>                   <none>
taint-step1-pod-2 	1/1         Running     0                7m30s   10.244.2.25   k8s-worker2   <none>                   <none>


[root@k8s-master ~]# kubectl  delete  pods  --all
pod "taint-step1-pod-1" deleted from default namespace
pod "taint-step1-pod-2" deleted from default namespace
```

---

## 3. Taint + Toleration 응용 실습

**EX3) 일반 Pod와 전용 Pod를 Taint로 분리 (같은 클러스터에서 특정 Pod만 전용 노드에 배치되도록 제어한다.)**

현재 Node 상태 : k8s-worker1(role=db:NoSchedule Taint 설정), k8s-worker2(Taint 없음)

- 일반 Pod 생성 : normal-pod, Toleration 없음, k8s-worker1에는 배치 불가
- DB Pod 생성 : db-pod, role=db:NoSchedule Toleration 설정, k8s-worker1에도 배치 가능

Toleration은 k8s-worker1에 배치될 수 있도록 허용하는 역할만 한다. 따라서 Toleration만 설정하면 db-pod는 k8s-worker1 또는 k8s-worker2에 배치될 수 있다. db-pod를 반드시 k8s-worker1에 배치하려면 k8s-worker1에 Label을 설정하고 nodeSelector를 함께 사용해야 한다.

```
	# nodeSelector를 사용하기윈 Label 설정
[root@k8s-master ~]# kubectl  label  nodes  k8s-worker1  role=db
node/k8s-worker1 labeled
```

```yaml
[root@k8s-master ~]# vi taint-step3-mixed.yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal-pod
spec:
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  nodeSelector:
    role: db
  tolerations:
  - key: "role"
    operator: "Equal"
    value: "db"
    effect: "NoSchedule"
  containers:
  - name: db
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "root"
    ports:
    - containerPort: 3306
```

```
[root@k8s-master ~]# kubectl  apply  -f  taint-step3-mixed.yaml  --dry-run=client
pod/normal-pod created (dry run)
pod/db-pod created (dry run)


[root@k8s-master ~]# kubectl  apply  -f  taint-step3-mixed.yaml
pod/normal-pod created
pod/db-pod created


[root@k8s-master ~]# kubectl   get  pods  -o  wide
NAME        	READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
db-pod       	1/1        Running     0                13s     10.244.1.24   k8s-worker1   <none>           <none>
normal-pod	1/1        Running     0                13s     10.244.2.28   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  delete  pods  --all
pod "db-pod" deleted from default namespace
pod "normal-pod" deleted from default namespace
```

**EX4) Node마다 서로 다른 Taint를 설정하고 Pod 3개의 Toleration 동작 확인**
- k8s-worker1 : cpu=true:NoSchedule
- k8s-worker2 : gpu=true:NoSchedule
- k8s-worker3 : cpu=true:NoSchedule, gpu=true:NoSchedule (Control-Plane을 임시 Worker-node로 사용)

```
	# Taint 확인
[root@k8s-master ~]# kubectl  describe  nodes  k8s-master | grep Taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1 | grep Taint
Taints:             role=db:NoSchedule

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker2 | grep Taint
Taints:             <none>


	# 설정된 Taint 삭제
[root@k8s-master ~]# kubectl  taint  node  k8s-master node-role.kubernetes.io/control-plane-
node/k8s-master untainted

	# 설정된 Taint 삭제
[root@k8s-master ~]# kubectl  taint  node  k8s-worker1 role-
node/k8s-worker1 untainted


	# Taint 확인
[root@k8s-master ~]# kubectl  describe  nodes  k8s-master | grep Taint
Taints:             <none>

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1 | grep Taint
Taints:             <none>


	# 실습 Taint 설정
[root@k8s-master ~]# kubectl taint node k8s-worker1 cpu=true:NoSchedule
node/k8s-worker1 tainted

[root@k8s-master ~]# kubectl taint node k8s-worker2 gpu=true:NoSchedule
node/k8s-worker2 tainted


[root@k8s-master ~]# kubectl taint node k8s-master cpu=true:NoSchedule
node/k8s-master tainted

[root@k8s-master ~]# kubectl taint node k8s-master gpu=true:NoSchedule
node/k8s-master tainted


	# Taint 확인
[root@k8s-master ~]# kubectl  describe  nodes  k8s-master
Name:               	k8s-master
Roles:              	control-plane
Labels:             	beta.kubernetes.io/arch=amd64
                    	beta.kubernetes.io/os=linux
                    	kubernetes.io/arch=amd64
                    	kubernetes.io/hostname=k8s-master
                    	kubernetes.io/os=linux
                    	node-role.kubernetes.io/control-plane=
                    	node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        	flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"ea:62:ea:c9:e4:f4"}
                    	flannel.alpha.coreos.com/backend-type: vxlan
                    	flannel.alpha.coreos.com/kube-subnet-manager: true
                    	flannel.alpha.coreos.com/public-ip: 192.168.10.100
                    	node.alpha.kubernetes.io/ttl: 0
                    	volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  	Tue, 11 Aug 2026 15:20:57 +0900
Taints:             	cpu=true:NoSchedule
                    	gpu=true:NoSchedule
Unschedulable:      false
~~~~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~~~~


[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1 | grep Taint
Taints:             cpu=true:NoSchedule


[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker2 | grep Taint
Taints:             gpu=true:NoSchedule


	# Toleration 설성 후 특정 Node에 배치하기위한 Label 설정
[root@k8s-master ~]# kubectl  label  node  k8s-worker1  type=cpu
node/k8s-worker1 labeled

[root@k8s-master ~]# kubectl  label  node  k8s-worker2  type=gpu
node/k8s-worker2 labeled

[root@k8s-master ~]# kubectl  label  node  k8s-master  type=cpu-gpu
node/k8s-master labeled


	# 각 Node의 Label 확인
[root@k8s-master ~]# kubectl get nodes -L type
NAME        	STATUS     ROLES    	AGE   VERSION   TYPE
k8s-master    	Ready        control-plane	15d     v1.35.7     cpu-gpu
k8s-worker1   	Ready        <none>           	14d     v1.35.7     cpu
k8s-worker2   	Ready        <none>           	14d     v1.35.7     gpu
```

```yaml
	# Toleration 과 nodeSelector를 적용한 Pod 생성
[root@k8s-master ~]# vi  taint-step4-multi-taint-nodeselector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-pod
spec:
  nodeSelector:
    type: cpu

  tolerations:
  - key: "cpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

  containers:
  - name: nginx
    image: nginx:1.29.1

---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    type: gpu

  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

  containers:
  - name: nginx
    image: nginx:1.29.1

---
apiVersion: v1
kind: Pod
metadata:
  name: cpu-gpu-pod
spec:
  nodeSelector:
    type: cpu-gpu

  tolerations:
  - key: "cpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

  containers:
  - name: nginx
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f  taint-step4-multi-taint-nodeselector.yaml
pod/cpu-pod created
pod/gpu-pod created
pod/cpu-gpu-pod created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME       	READY   STATUS    RESTARTS   AGE   IP               NODE
cpu-gpu-pod	1/1        Running      0               15s     10.244.0.29    k8s-master
cpu-pod       	1/1        Running      0               15s     10.244.1.25    k8s-worker1 
gpu-pod       	1/1        Running      0               15s     10.244.2.29    k8s-worker2
```

- cpu-pod : nodeSelector(type=cpu)에 의해 k8s-worker1을 선택하고, cpu=true:NoSchedule Taint에 대한 Toleration이 있으므로 배치가 허용되어 k8s-worker1에 정상 배치된다.
- gpu-pod : nodeSelector(type=gpu)에 의해 k8s-worker2를 선택하고, gpu=true:NoSchedule Taint에 대한 Toleration이 있으므로 배치가 허용되어 k8s-worker2에 정상 배치된다.
- cpu-gpu-pod : nodeSelector(type=cpu-gpu)에 의해 k8s-master를 선택한다. k8s-master에는 cpu=true:NoSchedule, gpu=true:NoSchedule Taint가 모두 설정되어 있고, Pod에도 두 Taint를 모두 허용하는 Toleration이 설정되어 있으므로 배치가 가능해 k8s-master에 정상 배치된다.

```
	# 모든 Pod 삭제
[root@k8s-master ~]# kubectl delete  pods  --all

	# 실습에서 설정한 Taint 삭제
[root@k8s-master ~]# kubectl taint node k8s-worker1 cpu-
node/k8s-worker1 untainted

[root@k8s-master ~]# kubectl taint node k8s-worker2 gpu-
node/k8s-worker2 untainted

[root@k8s-master ~]# kubectl taint node k8s-master cpu-
node/k8s-master untainted

[root@k8s-master ~]# kubectl taint node k8s-master gpu-
node/k8s-master untainted


	# Control-plane 기존 Taint 다시 설정
[root@k8s-master ~]# kubectl taint node k8s-master node-role.kubernetes.io/control-plane=:NoSchedule
node/k8s-master tainted
```

**EX5) NoSchedule과 PreferNoSchedule의 차이 확인**

```
[root@k8s-master ~]# kubectl  describe  nodes  k8s-master | grep Taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1 | grep Taint
Taints:             <none>

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker2 | grep Taint
Taints:             <none>


	# k8-worker1 노드에 taint 설정
[root@k8s-master ~]# kubectl  taint  node  k8s-worker1  role=db:NoSchedule
node/k8s-worker1 tainted

[root@k8s-master ~]# kubectl  describe  nodes  k8s-worker1 | grep Taint
Taints:             role=db:NoSchedule


	# k8-worker2 노드에 condon  설정
	# cordon  = 해당 노드에 새 Pod의 생성을 차단
[root@k8s-master ~]# kubectl  cordon  k8s-worker2
node/k8s-worker2 cordoned


[root@k8s-master ~]# kubectl  get  nodes
NAME          	STATUS                     	ROLES           	AGE   VERSION
k8s-master    	Ready                      	control-plane	15d     v1.35.7
k8s-worker1   	Ready                      	<none>          	15d     v1.35.7
k8s-worker2	Ready,SchedulingDisabled	<none>          	15d     v1.35.7
```

```yaml
[root@k8s-master ~]# vi  taint-prefernoschedule.yaml
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-1
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
---
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-2
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f   taint-prefernoschedule.yaml
pod/taint-step1-pod-1 created
pod/taint-step1-pod-2 created


[root@k8s-master ~]# kubectl  get  pods
NAME                  READY   STATUS    RESTARTS   AGE
taint-step1-pod-1   0/1         Pending     0                 33s
taint-step1-pod-2   0/1         Pending     0                 33s
```

노드 상태
- k8s-worker1 = role=db:NoSchedule (테인트가 걸려 있음)
- k8s-worker2 = cordon 상태 (스케줄링 불가)

Pod 상태 : Pod YAML에 toleration 없음. role=db:NoSchedule을 허용하지 않는다. 2개의 Pod가 모두 Pending 상태가 된다.

**스케줄러가 판단하는 순서**

스케줄러는 갈 수 있는 노드를 하나씩 검사한다.

1) k8s-worker2 : cordon 상태 → 새 Pod 배치 불가 → 후보 탈락
2) k8s-worker1 : NoSchedule 테인트 있음 → Pod가 toleration 없음 → 배치 절대 불가 → 후보 탈락
3) k8s-master : control-plane:NoSchedule 테인트 있음 → Pod toleration 없음 → 배치 불가 → 후보 탈락

결론 : 갈 수 있는 노드가 없기때문에 모든 Pod 상태가 Pending이된다.

```
[root@k8s-master ~]# kubectl  delete  pods  --all
pod "taint-step1-pod-1" deleted from default namespace
pod "taint-step1-pod-2" deleted from default namespace


	# 기존 taint 삭제
[root@k8s-master ~]# kubectl taint node  k8s-worker1 role=db:NoSchedule-
node/k8s-worker1 untainted


	# PreferNoSchedule taint 설정 (PreferNoSchedule = 되도록 피하되, 필요하면 사용 가능)
[root@k8s-master ~]# kubectl taint node k8s-worker1 role=db:PreferNoSchedule
node/k8s-worker1 tainted


[root@k8s-master ~]# kubectl  describe  nodes   | grep Taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule	# Control-Plane 	(Taint)
Taints:             role=db:PreferNoSchedule			# k8s-worker1	(Taint)
Taints:             node.kubernetes.io/unschedulable:NoSchedule	# k8s-worker2	(Cordon)


[root@k8s-master ~]# kubectl  apply  -f  taint-prefernoschedule.yaml
pod/taint-step1-pod-1 created
pod/taint-step1-pod-2 created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                  READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
taint-step1-pod-1   1/1         Running     0                11s     10.244.1.26   k8s-worker1   <none>                  <none>
taint-step1-pod-2   1/1         Running     0                11s     10.244.1.27   k8s-worker1   <none>                  <none>


	# 모든 Pod 삭제
[root@k8s-master ~]# kubectl  delete  pods  --all
pod "taint-step1-pod-1" deleted from default namespace
pod "taint-step1-pod-2" deleted from default namespace


	# PreferNoSchedule taint 설정 삭제
[root@k8s-master ~]# kubectl  taint  node  k8s-worker1  role=db:PreferNoSchedule-
node/k8s-worker1 untainted


	# cordon 설정 삭제
[root@k8s-master ~]# kubectl  uncordon   k8s-worker2
node/k8s-worker2 uncordone


[root@k8s-master ~]# kubectl  get  nodes
NAME          	STATUS   ROLES           AGE   VERSION
k8s-master    	Ready       control-plane   15d    v1.35.7
k8s-worker1	Ready       <none>           15d    v1.35.7
k8s-worker2	Ready       <none>           15d    v1.35.7
```

PreferNoSchedule Taint가 걸린 k8s-worker1이 유일하게 스케줄 가능한 Node였기 때문에, 강제(NoSchedule)가 아닌 선호(PreferNoSchedule)이므로 Toleration 없이도 Pod가 배치될 수 있었다.

---

## 4. Cordon & Drain이란

Cordon과 Drain은 특정 Node에 새로운 Pod가 배치되지 않도록 하거나, 기존에 실행 중인 Pod를 다른 Node로 이동시키기 위해 사용하는 Node 관리 기능이다.

주로 다음과 같은 상황에서 사용한다.
- Node를 비우고 유지보수
- OS 업데이트
- Kubernetes 업그레이드
- 하드웨어 교체
- Node를 잠시 Scheduling에서 제외

- Cordon = 새 Pod를 받지 않도록 Node를 막음
- Drain = Node를 비우기 위해 기존 Pod를 다른 곳으로 이동시킴

**Cordon**

Cordon은 특정 Node를 Scheduling 대상에서 제외하는 기능이다. 즉, 해당 Node에 새로운 Pod가 배치되지 않도록 설정한다.
- 새로운 Pod만 해당 Node에 배치되지 않는다.
- 기존에 실행 중인 Pod는 그대로 실행된다.

쉽게 말하면 "지금 실행 중인 Pod는 그대로 두고, 새로운 Pod는 받지 마"라는 의미다.

형식 : `kubectl cordon <Node이름>`
예시 : `kubectl cordon k8s-worker1`

결과
- k8s-worker1이 Scheduling 대상에서 제외됨
- 기존 Pod는 계속 실행
- 새로운 Pod는 k8s-worker1에 배치되지 않음

Cordon 상태 확인
```
kubectl get nodes
NAME          STATUS                     		ROLES
k8s-worker1   Ready,SchedulingDisabled   	<none>
k8s-worker2   Ready                      		<none>
```
SchedulingDisabled가 표시되면 해당 Node가 Cordon 상태라는 의미다.

**Cordon 해제**

Cordon을 해제하면 다시 새로운 Pod를 해당 Node에 배치할 수 있다.

형식 : `kubectl uncordon <Node이름>`
예시 : `kubectl uncordon k8s-worker1`

결과
- k8s-worker1이 다시 Scheduling 대상이 됨
- 새로운 Pod가 해당 Node에 배치될 수 있음

uncordon은 기존 Pod를 다시 배치하는 명령이 아니다. 단지 해당 Node를 다시 Scheduling 가능 상태로 변경한다.

**Drain**

Drain은 특정 Node에서 실행 중인 Pod를 정상적으로 종료하고 다른 Node에서 다시 실행될 수 있도록 Node를 비우는 기능이다. 주로 Node를 유지보수하거나 종료하기 전에 사용한다.

쉽게 말하면 "이 Node를 비워야 하니까 현재 Pod들을 다른 곳으로 보내자"라는 의미다.

형식 : `kubectl drain <Node이름>`
예시 : `kubectl drain k8s-worker1`

일반적으로 Drain을 수행하면 해당 Node가 먼저 Scheduling 불가 상태가 되고, 기존 Pod를 종료하여 다른 Node에서 다시 생성되도록 처리한다.

```
Drain  -->  새로운 Pod 배치 차단  -->  기존 Pod 종료/퇴출  -->  Controller가 관리하는 Pod라면 다른 Node에 새로운 Pod 생성  -->  Node가 비워짐
```

Drain은 Pod를 단순히 삭제하는 명령어가 아니다. Deployment, ReplicaSet 등의 Controller가 관리하는 Pod라면 기존 Pod를 Node에서 종료하고, ReplicaSet 등이 원하는 Replica 수를 유지하기 위해 다른 Node에 새로운 Pod를 생성한다. 따라서 서비스가 여러 Replica로 구성되어 있다면 다른 Node에서 서비스가 계속 실행될 수 있다.

Drain 전
```
worker1
 ├─ Pod A
 └─ Pod B

worker2
 └─ Pod C
```

worker1을 Drain
```
worker1
 └─ 비어 있음

worker2
 ├─ Pod C
 ├─ Pod A
 └─ Pod B
```

단, 실제 배치는 자원, Affinity, Taint 등의 조건에 따라 달라질 수 있다.

**Cordon과 Drain의 차이**

- Cordon : 새로운 Pod 배치 차단 / 기존 Pod 유지 / Node를 Scheduling에서 제외
- Drain : 새로운 Pod 배치 차단 / 기존 Pod를 종료·퇴출 / Controller가 관리하는 Pod는 다른 Node에서 다시 생성될 수 있음

Node를 비우는 목적
- Cordon : "새로운 Pod만 받지 마"
- Drain : "새로운 Pod도 받지 말고, 기존 Pod도 다른 곳으로 보내서 이 Node를 비워"

**Cordon과 Drain의 관계**

Drain은 Node를 비우는 과정에서 새로운 Pod가 해당 Node로 들어오지 못하도록 먼저 Scheduling을 막는다. 따라서 개념적으로는 다음과 같다.

```
Cordon  -->  새로운 Pod 배치 차단  -->  Drain  -->  기존 Pod 퇴출  -->  Node 비움
```

실제로 `kubectl drain` 명령을 실행하면 Drain 과정에서 해당 Node를 Scheduling 불가 상태로 만드는 동작도 수행한다. 따라서 보통 Node를 완전히 비우고 유지보수해야 하는 경우에는 Drain을 사용한다.

**Drain에서 자주 사용하는 옵션**

실제 환경에서는 Node에 어떤 종류의 Pod가 있는지에 따라 Drain 명령어가 바로 실행되지 않을 수 있다.

```
kubectl  drain  k8s-worker1  --ignore-daemonsets
```
- `--ignore-daemonsets` : DaemonSet이 관리하는 Pod는 Drain 과정에서 무시한다. DaemonSet Pod는 각 Node에 존재하는 것을 목적으로 하기 때문에 일반적인 Drain으로 다른 Node에 옮기는 대상이 아니다. 따라서 DaemonSet Pod가 있는 Node를 Drain할 때 자주 사용한다.

```
kubectl  drain  k8s-worker1  --ignore-daemonsets  --delete-emptydir-data
```
- `--delete-emptydir-data` : emptyDir 볼륨을 사용하는 Pod의 데이터를 삭제할 수 있도록 허용한다. emptyDir 데이터는 Node의 Pod가 삭제되면 유지되지 않는다. 따라서 해당 데이터가 삭제되어도 되는지 확인한 후 사용해야 한다.

**PodDisruptionBudget과 Drain**

운영 환경에서는 PodDisruptionBudget(PDB)도 고려해야 한다. PDB는 유지보수나 Drain처럼 자발적인 Pod 중단(Voluntary Disruption) 상황에서 동시에 중단되는 Pod 수를 제한하는 기능이다.

예를 들어 동일한 애플리케이션 Pod가 3개 실행 중일 때 최소 2개는 항상 실행되도록 PDB를 설정할 수 있다. 이 경우 Node Drain 과정에서 애플리케이션의 가용성을 유지하는 데 도움을 줄 수 있다.

- Drain : Node를 비우기 위한 작업
- PDB : Drain 등으로 Pod가 중단될 때 서비스 가용성을 보호하는 장치

**Cordon / Drain / Uncordon 전체 흐름**

Node 유지보수를 해야 한다고 가정한다.

1) Cordon : `kubectl cordon k8s-worker1` — 새로운 Pod 배치 차단, 기존 Pod는 계속 실행
2) Drain : `kubectl drain k8s-worker1 --ignore-daemonsets` — 새로운 Pod 배치 차단, 기존 Pod를 종료/퇴출, 관리되는 Pod는 다른 Node에서 다시 생성, Node를 비움
3) Node 유지보수 : OS 업데이트, Kubernetes 구성 변경, 하드웨어 점검, 기타 유지보수
4) Uncordon : `kubectl uncordon k8s-worker1` — Scheduling 다시 허용, 새로운 Pod가 해당 Node에 배치될 수 있음

**Cordon과 Drain을 사용하는 이유**

Node를 바로 종료하거나 유지보수하면 해당 Node에서 실행 중인 Pod에 장애가 발생할 수 있다. 따라서 먼저 Pod를 안전하게 다른 Node로 이동시킨 후 Node를 점검하는 것이 좋다.

일반적인 유지보수 흐름
```
01) Node 유지보수 필요
02) Cordon
03) 새로운 Pod 배치 차단
04) Drain
05) 기존 Pod 퇴출
06) 다른 Node에서 Pod 재생성
07) Node 비움
08) 유지보수
09) Uncordon
10) 다시 Scheduling 허용
```

**Cordon과 Taint의 차이**

Cordon과 Taint는 둘 다 Pod의 배치를 제한하지만 목적과 방식이 다르다.

- Cordon : Node 전체를 Scheduling에서 제외. 새로운 Pod가 해당 Node에 배치되지 않음. 별도의 Pod 설정 필요 없음
- Taint : Node에 특정 제한을 설정. 해당 Taint를 허용하지 않는 Pod만 배치 제한. Toleration이 있는 Pod는 해당 Taint를 통과할 수 있음

- Cordon : "현재 이 Node에는 새로운 Pod를 전부 받지 않음"
- Taint : "이 조건을 허용하지 않는 Pod는 받지 않음"

---

## 5. Cordon / Drain 실습

**EX1) cordon을 사용해 신규 Pod 배치를 차단**

cordon은 기존 Pod에는 영향을 주지 않고 새로운 Pod만 배치되지 않게 만든다는 것을 확인한다.

```yaml
[root@k8s-master ~]# vi cordon-step1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cordon-test-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.29.1
```

```
[root@k8s-master ~]# kubectl  apply  -f  cordon-step1.yaml
pod/cordon-test-pod created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME              	READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
cordon-test-pod   	1/1        Running      0                25s    10.244.2.30   k8s-worker2   <none>                  <none>


[root@k8s-master ~]# kubectl  get  nodes
NAME          	STATUS     ROLES           	AGE   VERSION
k8s-master    	Ready        control-plane   	15d     v1.35.7
k8s-worker1   	Ready        <none>          	15d     v1.35.7
k8s-worker2	Ready        <none>          	15d     v1.35.7


	# Cordon 설정 (해당 Pod가 설정된 Node를 설정해야 한다.)
[root@k8s-master ~]# kubectl  cordon  k8s-worker2
node/k8s-worker2 cordoned


[root@k8s-master ~]# kubectl  get  nodes
NAME          	STATUS                     	ROLES           	AGE   VERSION
k8s-master    	Ready                      	control-plane	15d     v1.35.7
k8s-worker1  	Ready                      	<none>          	15d     v1.35.7
k8s-worker2	Ready,SchedulingDisabled	<none>          	15d     v1.35.7
```

```yaml
[root@k8s-master ~]# cat taint-prefernoschedule.yaml
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-1
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
---
apiVersion: v1
kind: Pod
metadata:
  name: taint-step1-pod-2
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
```

```
	# 새로운 Pod 2개 생성
[root@k8s-master ~]# kubectl  apply  -f  taint-prefernoschedule.yaml
pod/taint-step1-pod-1 created
pod/taint-step1-pod-2 created


	# 기존의 Pod는 유지되지만 새로운 Pod의 생성은 차단된다.
[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                	READY   STATUS    RESTARTS   AGE	  IP               NODE            NOMINATED NODE   READINESS GATES
cordon-test-pod	1/1        Running      0                4m	  10.244.2.30   k8s-worker2   <none>           <none>
taint-step1-pod-1	1/1        Running      0                19s	  10.244.1.29   k8s-worker1   <none>           <none>
taint-step1-pod-2	1/1        Running      0                19s	  10.244.1.28   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  uncordon  k8s-worker2
node/k8s-worker2 uncordoned


[root@k8s-master ~]# kubectl  delete  pods  --all
pod "cordon-test-pod" deleted from default namespace
pod "taint-step1-pod-1" deleted from default namespace
pod "taint-step1-pod-2" deleted from default namespace
```

cordon-test-pod(k8s-worker2에 기존 배치)는 그대로 Running 상태를 유지했지만, cordon 이후 apply한 새 Pod 2개는 모두 k8s-worker1에만 생성되고 k8s-worker2에는 배치되지 않았다. cordon이 기존 Pod에는 영향을 주지 않고 신규 배치만 차단한다는 것을 확인할 수 있다.

**EX2) drain으로 기존 Pod를 안전하게 이동**

drain은 노드를 비우기 위해 기존 Pod를 종료하고 다른 노드에서 재생성한다.

```yaml
[root@k8s-master ~]# vi drain-ex2-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drain-test-deploy
spec:
  replicas: 4
  selector:
    matchLabels:
      app: drain-test
  template:
    metadata:
      labels:
        app: drain-test
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
[root@k8s-master ~]# kubectl  apply  -f  drain-ex2-deploy.yaml
deployment.apps/drain-test-deploy created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
drain-test-deploy-789567f66-2qrs4	1/1         Running     0                6s      10.244.2.31   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-bqfpk   	1/1         Running     0                6s      10.244.1.30   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-h7vch   	1/1         Running     0                6s      10.244.2.32   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-tjcxr   	1/1         Running     0                6s      10.244.1.31   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  drain  k8s-worker1  --ignore-daemonsets --delete-emptydir-data
node/k8s-worker1 cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-f9wcz, kube-system/kube-proxy-gbr4f
evicting pod default/drain-test-deploy-789567f66-bqfpk
evicting pod ingress-nginx/ingress-nginx-controller-f85ff6d7d-wvdpf
evicting pod default/drain-test-deploy-789567f66-tjcxr
pod/drain-test-deploy-789567f66-bqfpk evicted
pod/drain-test-deploy-789567f66-tjcxr evicted
pod/ingress-nginx-controller-f85ff6d7d-wvdpf evicted
node/k8s-worker1 drained


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
drain-test-deploy-789567f66-2qrs4   	1/1         Running     0                4m5s   10.244.2.31   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-2rqvk   	1/1         Running     0                24s     10.244.2.33   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-bckdv   	1/1         Running     0                24s     10.244.2.35   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-h7vch   	1/1         Running     0                4m5s   10.244.2.32   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  get  nodes
NAME          	STATUS                     	ROLES           AGE   VERSION
k8s-master    	Ready                      	control-plane   15d    v1.35.7
k8s-worker1  	Ready,SchedulingDisabled   	<none>           15d    v1.35.7
k8s-worker2	Ready                      	<none>           15d    v1.35.7


[root@k8s-master ~]# kubectl  delete  deployments  drain-test-deploy
deployment.apps "drain-test-deploy" deleted from default namespace


	# drain을 삭제하는 명령어는 별도로 존재하지않는다. (uncordon으로 삭제)
[root@k8s-master ~]# kubectl  uncordon  k8s-worker1
node/k8s-worker1 uncordoned
```

drain은 k8s-worker1을 자동으로 cordon 상태로 만든 뒤, DaemonSet Pod는 무시하고 Deployment가 관리하는 Pod만 evict했다. evict된 Pod는 ReplicaSet에 의해 곧바로 k8s-worker2에서 재생성되어 최종적으로 replicas 4개가 모두 k8s-worker2에서 실행되었다.

**EX2 유지보수 실습: cordon + drain + uncordon + 파드 재배치 확인**
- cordon으로 새 파드 스케줄을 차단하는 것을 확인
- drain으로 디플로이먼트 파드가 다른 노드로 이동되는 것을 확인
- DaemonSet 파드는 --ignore-daemonsets 옵션으로 남는 것을 확인
- uncordon으로 노드를 복구한 뒤 다시 스케줄 가능한 상태를 확인

```
[root@k8s-master ~]# cat drain-ex2-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drain-test-deploy
spec:
  replicas: 4
  selector:
    matchLabels:
      app: drain-test
  template:
    metadata:
      labels:
        app: drain-test
    spec:
      containers:
      - name: nginx
        image: nginx


[root@k8s-master ~]# kubectl  apply  -f  drain-ex2-deploy.yaml
deployment.apps/drain-test-deploy created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
drain-test-deploy-789567f66-2qrs4	1/1         Running     0                6s      10.244.2.31   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-bqfpk   	1/1         Running     0                6s      10.244.1.30   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-h7vch   	1/1         Running     0                6s      10.244.2.32   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-tjcxr   	1/1         Running     0                6s      10.244.1.31   k8s-worker1   <none>           <none>
```

DaemonSet은 각 노드에 1개씩 떠 있어야 한다. k8s-worker2에도 drain-ds 파드가 떠 있어야 한다.

```yaml
[root@k8s-master ~]# vi step2-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: drain-ds
spec:
  selector:
    matchLabels:
      app: drain-ds
  template:
    metadata:
      labels:
        app: drain-ds
    spec:
      containers:
      - name: agent
        image: busybox:1.36
        command: ["/bin/sh","-c"]
        args:
        - |
          while true; do
            echo "$(date) node=$(cat /etc/hostname)" >> /var/log/agent.log
            sleep 10
          done
```

```
[root@k8s-master ~]# kubectl  apply  -f  step2-daemonset.yaml
daemonset.apps/drain-ds created


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
drain-ds-7z2qs                      	1/1         Running     0                7s        10.244.2.38   k8s-worker2   <none>           <none>
drain-ds-tvcxq                      	1/1         Running     0                7s        10.244.1.34   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-4f745   	1/1         Running     0                2m12s   10.244.2.36   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-ddn6m   	1/1         Running     0                2m12s   10.244.1.33   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-tlr4m   	1/1         Running     0                2m12s   10.244.2.37   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-w6h9n	1/1         Running     0                2m12s   10.244.1.32   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  get  daemonsets  drain-ds
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
drain-ds    2              2               2            2                    2                 <none>                  79s


		# k8s-worker2에 Cordon을 설정
[root@k8s-master ~]# kubectl  cordon  k8s-worker2
node/k8s-worker2 cordoned


[root@k8s-master ~]# kubectl  get  nodes
NAME          	STATUS                     	ROLES           AGE   VERSION
k8s-master    	Ready                      	control-plane   15d    v1.35.7
k8s-worker1   	Ready                      	<none>           15d    v1.35.7
k8s-worker2	Ready,SchedulingDisabled	<none>           15d    v1.35.7


	# 기존의 Pod의 생성은 차단하지만 기존의 Pod는 유지되기때문에 변화는 없다.
[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
drain-ds-7z2qs                      	1/1         Running     0                7s        10.244.2.38   k8s-worker2   <none>           <none>
drain-ds-tvcxq                      	1/1         Running     0                7s        10.244.1.34   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-4f745   	1/1         Running     0                2m12s   10.244.2.36   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-ddn6m   	1/1         Running     0                2m12s   10.244.1.33   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-tlr4m   	1/1         Running     0                2m12s   10.244.2.37   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-w6h9n	1/1         Running     0                2m12s   10.244.1.32   k8s-worker1   <none>           <none>


	# replicas을 6으로 변경 (2개의 Pod가 생성된다, Cordon에의해 k8s-worker1에만 생성된다.)
[root@k8s-master ~]# kubectl  scale  deployment  drain-test-deploy  --replicas=6
deployment.apps/drain-test-deploy scaled


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
drain-ds-7z2qs                      	1/1         Running     0                7m4s   10.244.2.38   k8s-worker2   <none>           <none>
drain-ds-tvcxq                      	1/1         Running     0                7m4s   10.244.1.34   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-4f745   	1/1         Running     0                9m9s   10.244.2.36   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-ddn6m   	1/1         Running     0                9m9s   10.244.1.33   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-gmzn7   	1/1         Running     0                64s     10.244.1.35   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-rcgsn   	1/1         Running     0                64s     10.244.1.36   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-tlr4m   	1/1         Running     0                9m9s   10.244.2.37   k8s-worker2   <none>           <none>
drain-test-deploy-789567f66-w6h9n   	1/1         Running     0                9m9s   10.244.1.32   k8s-worker1   <none>           <none>


[root@k8s-master ~]# kubectl  drain  k8s-worker2  --ignore-daemonsets  --force=false
node/k8s-worker2 already cordoned
Warning: ignoring DaemonSet-managed Pods: default/drain-ds-7z2qs, kube-flannel/kube-flannel-ds-z4fbv, kube-system/kube-proxy-bxjvw
evicting pod ingress-nginx/ingress-nginx-controller-f85ff6d7d-9lzcj
evicting pod default/drain-test-deploy-789567f66-4f745
evicting pod default/drain-test-deploy-789567f66-tlr4m
pod/drain-test-deploy-789567f66-tlr4m evicted
pod/drain-test-deploy-789567f66-4f745 evicted
pod/ingress-nginx-controller-f85ff6d7d-9lzcj evicted
node/k8s-worker2 drained
```

`--force=false` : 컨트롤러가 관리하지 않는 일반 Pod가 Node에 존재하면 강제로 삭제하지 않는다. 일반 Pod는 삭제되어도 자동으로 재생성되지 않을 수 있으므로, 서비스 중단을 방지하기 위해 drain이 이런 Pod의 강제 삭제를 막는다.

`--force=true` : Deployment, ReplicaSet 등의 컨트롤러가 관리하지 않는 일반 Pod도 삭제를 허용한다. 일반 Pod는 삭제 후 자동으로 다시 생성되지 않을 수 있으므로 주의해야 한다.

```
	# k8s-worker2에는 DaemonSet에의해 만들어진 Pod 1개만 실행되어야 한다.
[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                	READY   STATUS    RESTARTS   AGE     IP              NODE          NOMINATED NODE   READINESS GATES
drain-ds-7z2qs                      	1/1         Running     0                10m     10.244.2.38    k8s-worker2   <none>           <none>
drain-ds-tvcxq                      	1/1         Running     0                10m     10.244.1.34    k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-ddn6m   	1/1         Running     0                12m     10.244.1.33    k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-fhbhf   	1/1         Running     0                24s       10.244.1.38   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-gmzn7   	1/1         Running     0                4m16s   10.244.1.35   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-rcgsn   	1/1         Running     0                4m16s   10.244.1.36   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-w64bw   	1/1         Running     0                24s       10.244.1.37   k8s-worker1   <none>           <none>
drain-test-deploy-789567f66-w6h9n   	1/1         Running     0                12m     10.244.1.32    k8s-worker1   <none>           <none>
```

Drain은 1회성 동작이기때문에 현재 k8s-worker2는 Cordon 상태이다. `--ignore-daemonsets` 옵션 덕분에 drain-ds(DaemonSet) Pod는 evict되지 않고 k8s-worker2에 그대로 남아있으며, Deployment가 관리하는 나머지 Pod는 모두 evict되어 k8s-worker1에서 재생성되었다. Node를 다시 스케줄 가능하게 만들려면 `kubectl uncordon k8s-worker2`를 실행해야 한다.

---

## 6. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- Pod가 Pending 상태이면 `kubectl describe pod <이름>`의 Events에서 `FailedScheduling` 사유를 확인한다. Taint를 허용하지 못하는 경우 `node(s) had untolerated taint` 메시지가 표시된다.
- `kubectl describe nodes <이름> | grep Taint` 또는 `kubectl describe nodes | grep Taint`로 각 Node에 걸린 Taint를 확인하고, Pod의 `tolerations`가 key/value/effect까지 정확히 일치하는지 대조한다.
- Taint를 제거할 때는 `kubectl taint node <Node이름> <key>=<value>:<effect>-` 처럼 뒤에 `-`를 붙인다. value 없이 `<key>:<effect>-` 형태로도 삭제할 수 있다.
- NoSchedule Taint는 기존 Pod에 영향이 없지만, NoExecute Taint는 Toleration이 없는 기존 Pod까지 제거할 수 있다는 점을 유지보수 계획 시 반드시 구분해야 한다.
- Toleration만 설정하고 nodeSelector/Node Affinity를 함께 쓰지 않으면 Pod가 원하는 특정 Node가 아니라 Toleration을 만족하는 아무 Node에나 배치될 수 있다. 특정 Node에 반드시 배치하려면 Toleration과 nodeSelector(또는 Node Affinity)를 함께 사용한다.
- `kubectl cordon`은 새 Pod 배치만 차단하고 기존 Pod는 그대로 둔다. Node를 완전히 비우려면 `kubectl drain --ignore-daemonsets`까지 실행해야 하며, DaemonSet Pod가 있는 Node에서는 `--ignore-daemonsets` 없이 drain하면 실패하거나 경고가 발생한다.
- emptyDir을 사용하는 Pod가 있는 Node를 drain할 때는 `--delete-emptydir-data`가 필요하며, 이 옵션은 해당 데이터를 삭제하므로 사전에 데이터 보존 여부를 확인해야 한다.
- 컨트롤러가 관리하지 않는 일반(bare) Pod가 있는 Node는 기본적으로 drain이 거부된다. 강제로 진행하려면 `--force`를 사용하되, 해당 Pod는 삭제 후 자동으로 재생성되지 않는다는 점을 주의한다.
- drain은 1회성 동작이며 완료 후에도 Node는 Cordon(SchedulingDisabled) 상태로 남는다. 유지보수가 끝나면 반드시 `kubectl uncordon <Node이름>`으로 복구해야 새 Pod가 다시 배치된다.
- 운영 환경에서는 PodDisruptionBudget(PDB)을 설정해 Drain 등 자발적 중단(Voluntary Disruption) 상황에서도 최소 가용 Pod 수를 보장하는 것이 안전하다.

---

>  **핵심 요약**
> - Taint는 Node가 Pod를 거부하는 설정(`key=value:effect`)이고, Toleration은 Pod가 그 Taint를 허용하겠다고 선언하는 설정이다 — Toleration이 있다고 반드시 그 Node에 배치되는 것은 아니다
> - Effect는 NoSchedule(신규 배치만 차단), PreferNoSchedule(가능하면 회피, 강제 아님), NoExecute(신규 배치 차단 + 기존 Pod도 제거 가능) 세 가지가 대표적이다
> - Taint & Toleration은 "누가 들어올 수 있는가"(Node 입장의 제한)를 다루고, Node Affinity/nodeSelector는 "어디에 들어갈 것인가"(Pod 입장의 선택)를 다루므로 특정 Node에 반드시 배치하려면 둘을 함께 사용한다
> - Cordon은 새 Pod 배치만 차단(`SchedulingDisabled`)하고 기존 Pod는 유지하며, Drain은 Cordon 후 기존 Pod까지 evict해 Controller가 다른 Node에 재생성하도록 한다
> - `kubectl drain <Node> --ignore-daemonsets [--delete-emptydir-data] [--force]`로 유지보수 대상 Node를 비우고, 작업 후 `kubectl uncordon <Node>`로 복구하는 것이 표준 유지보수 흐름이다
> - 관련: 2.  Kubernetes - Pod 생성 · 24.  Kubernetes - Label · 25.  Kubernetes - Pod Scheduling (nodeSelector·Affinity)
