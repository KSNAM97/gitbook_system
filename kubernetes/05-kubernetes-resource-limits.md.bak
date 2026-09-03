# Kubernetes - ResourceQuota·LimitRange

> **Tag:** #Kubernetes #ResourceQuota #LimitRange
> **핵심 요약:** ResourceQuota로 namespace 전체의 자원 총량을 제한하고 LimitRange로 개별 파드/컨테이너의 최소·최대·기본 자원값을 제한하는 방법, CPU/Memory Limit 초과 시 실제 동작(throttling vs OOMKilled) 정리

---

## 1. ResourceQuota와 LimitRange로 자원 제한하기

### 왜 자원 제한이 필요한가

쿠버네티스 클러스터는 여러 팀 / 여러 서비스가 같이 쓰는 서버 묶음이다.

자원 제한이 없으면:
- 한 팀이 실수로 파드를 무한히 생성
- 어떤 파드가 메모리, CPU를 과도하게 사용
- 다른 팀 서비스가 갑자기 느려지거나 죽음

그래서 쿠버네티스는 "이 네임스페이스는 이 정도까지만 써라", "이 파드는 최소/최대 이 범위 안에서만 써라"라는 규칙을 걸 수 있게 해준다.

여기서 등장하는 개념이 **ResourceQuota**와 **LimitRange**다.

### ResourceQuota

ResourceQuota는 네임스페이스 전체에 걸리는 제한이다. 쉽게 말하면 "이 네임스페이스는 총량이 여기까지다"라고 말하는 규칙이다.

예를 들면 파드 최대 10개까지만 생성 가능, CPU 요청 총합은 2코어까지만, 메모리 사용 총합은 4Gi까지만 — 이건 개별 파드가 아니라 네임스페이스 전체 합계를 제한한다.

**ResourceQuota가 제한하는 것들**
- pods
- requests.cpu / limits.cpu
- requests.memory / limits.memory
- services

| 항목 | 설명 |
|---|---|
| pods | 해당 네임스페이스에서 생성할 수 있는 파드의 최대 개수를 제한. 파드를 무한히 생성해서 클러스터 자원을 잠식하는 상황을 방지. 예: `pods: 10` (이 네임스페이스에는 파드를 최대 10개까지만 생성 가능) |
| requests.cpu | 파드들이 "최소로 보장받고 싶다"고 요청한 CPU의 총합 제한. 스케줄러가 파드를 노드에 배치할 때 기준으로 사용하는 값. 예: `requests.cpu: 2` (이 네임스페이스의 모든 파드 CPU 요청 합계는 2코어를 넘을 수 없음) |
| limits.cpu | 실제로 파드가 사용할 수 있는 CPU 최대치의 총합 제한. 한 네임스페이스가 CPU를 독점하는 상황 방지. 예: `limits.cpu: 4` (모든 파드의 CPU limit 합계는 4코어를 초과할 수 없음) |
| requests.memory | 파드들이 "최소 보장 메모리"로 요청한 값의 총합 제한. 노드에 파드가 배치될 수 있는지 판단하는 기준. 예: `requests.memory: 2Gi` (모든 파드의 메모리 요청 합계는 2Gi를 넘을 수 없음) |
| limits.memory | 파드들이 실제로 사용할 수 있는 메모리 최대치의 총합 제한. 메모리 폭주로 인한 노드 장애 방지. 예: `limits.memory: 4Gi` (모든 파드의 메모리 limit 합계는 4Gi를 초과할 수 없음) |
| services | 네임스페이스에서 생성 가능한 Service 객체 개수 제한. LoadBalancer, NodePort 등을 무분별하게 생성하는 것을 방지. 예: `services: 5` (이 네임스페이스에는 서비스 객체를 최대 5개까지만 생성 가능) |

이 네임스페이스에서 생성 가능한 자원 총량을 막는 것이다.

### LimitRange

LimitRange는 개별 파드(또는 컨테이너)에 적용되는 규칙이다. 쉽게 말하면 파드 하나당 규칙이다.

예를 들면:
- 파드 하나는 메모리 최소 128Mi 이상
- 파드 하나는 메모리 최대 512Mi 이하
- request를 안 쓰면 기본값으로 자동 지정

**LimitRange가 하는 핵심 역할**
- 너무 작은 값 방지 (말도 안 되는 파드 생성 방지)
- 너무 큰 값 방지 (한 파드가 자원 독점하는 것 방지)
- request / limit 안 적어도 기본값 강제 (아무 생각 없이 만든 파드를 통제)

개발자 실수를 막는 안전장치다.

### ResourceQuota와 LimitRange 차이

| 구분 | ResourceQuota | LimitRange |
|---|---|---|
| 범위 | 네임스페이스 전체 | 파드/컨테이너 하나 |
| 목적 | 전체 자원 총량 통제 | 파드 크기 규격화 |
| 예 | 파드 최대 10개, CPU 총합 2코어 | 파드 메모리 128Mi ~ 512Mi |

- LimitRange만 있으면 파드 하나는 제한되지만, 파드를 100개 만들 수 있다.
- ResourceQuota만 있으면 총량은 제한되지만, 파드 하나의 크기가 커질 수 있다.

LimitRange로 파드 모양을 통제하고, ResourceQuota로 네임스페이스 전체 규모를 통제하는 조합으로 운영 단위를 만든다.

### EX1) 베이스라인 만들기

resource 네임스페이스를 만들고 Quota 없이 파드 3개를 생성해 제한이 없는 상태를 확인한다.

- 네임스페이스: resource
- 파드: base-pod-1 ~ base-pod-3 (nginx)
- namespace 생성: 이후 정책 적용 범위가 되는 운영 단위를 만든다.

```
   # namespace 생성
[root@k8s-master ~]# kubectl  create  namespace resource
namespace/resource created


   # pod 생성
[root@k8s-master ~]# kubectl  run  base-pod1  --image=nginx:1.31  -n resource
pod/base-pod1 created


[root@k8s-master ~]# kubectl  run  base-pod2  --image=nginx:1.31  -n resource
pod/base-pod2 created


[root@k8s-master ~]# kubectl  run  base-pod3  --image=konan7979/custom-nginx-web:1.31  -n resource
pod/base-pod3 created



[root@k8s-master ~]# kubectl  get  pods   -n  resource
NAME        READY   STATUS    RESTARTS   AGE
base-pod1   1/1         Running     0                4m21s
base-pod2   1/1         Running     0                4m16s
base-pod3   1/1         Running     0                94s



[root@k8s-master ~]# kubectl  get  pods  -o  wide -n  resource
NAME        READY   STATUS    RESTARTS   AGE     IP            NODE            NOMINATED NODE   READINESS GATES
base-pod1   1/1         Running     0                4m4s    10.244.2.3   k8s-worker2   <none>                   <none>
base-pod2   1/1         Running     0                3m59s   10.244.1.4   k8s-worker1   <none>                  <none>
base-pod3   1/1         Running     0                77s       10.244.1.5   k8s-worker1   <none>                  <none>



[root@k8s-master ~]# kubectl  delete  pods  --namespace resource  --all
pod "base-pod1" deleted from resource namespace
pod "base-pod2" deleted from resource namespace
pod "base-pod3" deleted from resource namespace
```

동일한 결과를 하나의 YAML 파일로도 만들 수 있다.

```
[root@k8s-master ~]# vi  step1-baseline.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: resource
---
apiVersion: v1
kind: Pod
metadata:
  name: base-pod1
  namespace: resource
spec:
  containers:
    - name: nginx-web1
      image: nginx:1.31
---
apiVersion: v1
kind: Pod
metadata:
  name: base-pod2
  namespace: resource
spec:
  containers:
    - name: nginx-web2
      image: nginx:1.31
---
apiVersion: v1
kind: Pod
metadata:
  name: base-pod3
  namespace: resource
spec:
  containers:
    - name: nginx-web3
      image: konan7979/custom-nginx-web:1.31



[root@k8s-master ~]# kubectl  apply  -f  step1-baseline.yaml
namespace/resource created
pod/base-pod1 created
pod/base-pod2 created
pod/base-pod3 created


[root@k8s-master ~]# kubectl  get  pods   -n  resource
NAME        READY   STATUS    RESTARTS   AGE
base-pod1   1/1         Running     0                4m21s
base-pod2   1/1         Running     0                4m16s
base-pod3   1/1         Running     0                94s



[root@k8s-master ~]# kubectl  get  pods  -o  wide -n  resource
NAME        READY   STATUS    RESTARTS   AGE     IP            NODE            NOMINATED NODE   READINESS GATES
base-pod1   1/1         Running     0                4m4s    10.244.2.3   k8s-worker2   <none>                   <none>
base-pod2   1/1         Running     0                3m59s   10.244.1.4   k8s-worker1   <none>                  <none>
base-pod3   1/1         Running     0                77s       10.244.1.5   k8s-worker1   <none>                  <none>
```

### EX2) 파드 개수 제한

resource에 `pods=2` ResourceQuota를 걸고 파드 3개 생성 시 3번째 pod부터 거부되는지 확인한다.

- ResourceQuota가 네임스페이스 전체의 오브젝트 개수를 제한
- quota used/hard를 읽을 수 있게 한다.

- ResourceQuota: rq-pod-count
- 네임스페이스: resource
- pod 개수 제한: 2개

```
[root@k8s-master ~]# vi  rq-step2-qouta.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-pod-count
  namespace: resource
spec:
  hard:
    pods: 2



[root@k8s-master ~]# kubectl  apply  -f  rq-step2-qouta.yaml
resourcequota/rq-pod-count created




[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME            REQUEST     LIMIT   AGE
rq-pod-count   pods: 0/2                 51s



[root@k8s-master ~]# kubectl  get  resourcequotas  rq-pod-count  -n  resource
NAME            REQUEST     LIMIT   AGE
rq-pod-count   pods: 0/2                 51s



[root@k8s-master ~]# kubectl  describe  resourcequotas  -n  resource
Name:         rq-pod-count
Namespace:  resource
Resource     Used  Hard
--------      ----  ----
pods           0        2

-Hard = 이 네임스페이스에 허용된 최대 pod 한도(제한값)
-Used = 현재 이 네임스페이스에서 실제로 사용 중인 pod




   # 첫번째 pod 생성
[root@k8s-master ~]# kubectl  run  base-pod1  --image=nginx:1.31  -n resource
pod/base-pod1 created



[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME           REQUEST     LIMIT   AGE
rq-pod-count   pods: 1/2                4m46s



   # 두번째 pod 생성
[root@k8s-master ~]# kubectl  run  base-pod2  --image=nginx:1.31  -n resource
pod/base-pod2 created


[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME            REQUEST     LIMIT   AGE
rq-pod-count   pods: 2/2                 6m4s



   # 세번째 pod 생성
[root@k8s-master ~]# kubectl  run  base-pod3  --image=konan7979/custom-nginx-web:1.31  -n resource
Error from server (Forbidden): pods "base-pod3" is forbidden: exceeded quota: rq-pod-count, requested: pods=1, used: pods=2, limited: pods=2




[root@k8s-master ~]# kubectl  delete  pods  -n resource  --all
pod "base-pod1" deleted from resource namespace
pod "base-pod2" deleted from resource namespace
```

### EX3-1) 요청(requests) 총합 제한

resource에 `requests.cpu=1`을 제한하고, 요청이 있는 파드로 총합을 채운 뒤 초과 생성이 거부되는지 확인한다.

- requests.* 는 최소 요청(예약) 총합
- 파드 request를 누적해서 quota used 확인

실습 설정:
- 네임스페이스: resource
- ResourceQuota: rq-req-total

제한:
- requests.cpu: "1"
- 파드(각각 request 포함): req-a: cpu 500m / req-b: cpu 400m / req-c: cpu 400m (초과로 실패)

```
[root@k8s-master ~]# vi  rq-step3-requests-qouta.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-req-total
  namespace: resource
spec:
  hard:
    requests.cpu: "1"

: wq

[root@k8s-master ~]# kubectl  apply  -f  rq-step3-requests-qouta.yaml  --dry-run=client
resourcequota/rq-req-total created (dry run)



[root@k8s-master ~]# kubectl  apply  -f  rq-step3-requests-qouta.yaml
resourcequota/rq-req-total created



[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME           REQUEST             LIMIT   AGE
rq-req-total    requests.cpu: 0/1               50s



[root@k8s-master ~]# kubectl  describe  resourcequotas  -n  resource
Name:           rq-req-total
Namespace:    resource
Resource       Used  Hard
--------       ----  ----
requests.cpu   0       1



[root@k8s-master ~]# nproc		# CPU 코어 개수 확인
2




	# 파드 생성

[root@k8s-master ~]# vi  rq-step3-request-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: req-pod1
  namespace: resource
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    resources:
      requests:
        cpu: "500m"
---
apiVersion: v1
kind: Pod
metadata:
  name: req-pod2
  namespace: resource
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    resources:
      requests:
        cpu: "400m"
---
apiVersion: v1
kind: Pod
metadata:
  name: req-pod3
  namespace: resource
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    resources:
      requests:
        cpu: "400m"




[root@k8s-master ~]# kubectl  apply  -f  rq-step3-request-pod.yaml  --dry-run=server
pod/req-pod1 created (server dry run)
pod/req-pod2 created (server dry run)
pod/req-pod3 created (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  rq-step3-request-pod.yaml
pod/req-pod1 created
pod/req-pod2 created
Error from server (Forbidden): error when creating "rq-step3-request-pod.yaml": pods "req-pod3" is forbidden: exceeded quota: rq-req-total, requested: requests.cpu=400m, used: requests.cpu=900m, limited: requests.cpu=1



[root@k8s-master ~]# kubectl  get  resourcequotas  -n resource
NAME           REQUEST                LIMIT   AGE
rq-req-total   requests.cpu: 900m/1              19m



[root@k8s-master ~]# kubectl  describe  resourcequotas  -n resource
Name:           rq-req-total
Namespace:    resource
Resource       Used  Hard
--------       ----  ----
requests.cpu   900m  1
```

### EX3-2) 요청(requests.memory) 총합 제한

resource에 `requests.memory=1Gi`를 제한하고 요청이 있는 파드로 총합을 채운 뒤 초과 생성이 거부되는지 확인한다.

- requests.memory는 최소 요청(예약) 메모리 총합
- 파드 request를 누적하면 quota used가 올라간다.
- 총합을 초과하면 파드 생성이 Forbidden으로 거부된다.

실습 설정:
- 네임스페이스: resource
- ResourceQuota: rq-req-mem-total

제한:
- requests.memory: 512Mi(각 파드) — 총합 1Gi
- 파드(각각 request 포함): mem-pod-1 : 512Mi / mem-pod-2 : 512Mi / mem-pod-3 : 256Mi (초과로 실패)

```
[root@k8s-master ~]# vi rq-step3-2-request-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-req-mem-total
  namespace: resource
spec:
  hard:
    requests.memory: "1Gi"

: wq


[root@k8s-master ~]# kubectl  apply  -f  rq-step3-2-request-quota.yaml  --dry-run=server
resourcequota/rq-req-mem-total created (server dry run)


[root@k8s-master ~]# kubectl  apply  -f  rq-step3-2-request-quota.yaml
resourcequota/rq-req-mem-total created


[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME                  REQUEST                    LIMIT   AGE
rq-req-mem-total   requests.memory: 0/1Gi             16s


[root@k8s-master ~]# kubectl  describe  resourcequotas  -n  resource
Name:                 	rq-req-mem-total
Namespace:          	resource
Resource     	Used  Hard
--------             	--- -  ----
requests.memory  	0       1Gi




[root@k8s-master ~]# vi  rq-step-3-2-request-mem-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mem-pod1
  namespace: resource
spec:
  containers:
  - name: mem-container
    image: nginx:1.31
    resources:
      requests:
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: mem-pod2
  namespace: resource
spec:
  containers:
  - name: mem-container
    image: nginx:1.31
    resources:
      requests:
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: mem-pod3
  namespace: resource
spec:
  containers:
  - name: mem-container
    image: nginx:1.31
    resources:
      requests:
        memory: "256Mi"





[root@k8s-master ~]# kubectl  apply  -f  rq-step-3-2-request-mem-pod.yaml  --dry-run=server
pod/mem-pod1 created (server dry run)
pod/mem-pod2 created (server dry run)
pod/mem-pod3 created (server dry run)




[root@k8s-master ~]#  kubectl  apply  -f   rq-step-3-2-request-mem-pod.yaml
pod/mem-pod1 created
pod/mem-pod2 created
Error from server (Forbidden): error when creating "rq-step-3-2-request-mem-pod.yaml": pods "mem-pod3" is forbidden: exceeded quota: rq-req-mem-total, requested: requests.memory=256Mi, used: requests.memory=1Gi, limited: requests.memory=1Gi



[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME                 REQUEST                    	LIMIT   	AGE
rq-req-mem-total   requests.memory: 1Gi/1Gi           	11m




[root@k8s-master ~]# kubectl  describe  resourcequotas  -n  resource
Name:            	rq-req-mem-total
Namespace:      	resource
Resource         	Used  Hard
--------         	----  ----
requests.memory  	1Gi   1Gi





[root@k8s-master ~]# kubectl  delete  -f  rq-step3-2-request-quota.yaml
resourcequota "rq-req-mem-total" deleted from resource namespace


[root@k8s-master ~]# kubectl  delete  -f  rq-step-3-2-request-mem-pod.yaml
pod "mem-pod1" deleted from resource namespace
pod "mem-pod2" deleted from resource namespace
Error from server (NotFound): error when deleting "rq-step-3-2-request-mem-pod.yaml": pods "mem-pod3" not found
```

CPU까지 함께 요청하도록 수정하면 세 파드 모두 정상 생성된다.

```
[root@k8s-master ~]# vi rq-step-3-2-request-mem-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mem-pod1
  namespace: resource
spec:
  containers:
  - name: mem-container
    image: nginx:1.31
    resources:
      requests:
        memory: "512Mi"
        cpu: "1"		# CPU 추가
---
~~~~~~~~~~ 중간 생략 ~~~~~~~~~~

:wq




[root@k8s-master ~]# kubectl  apply  -f  rq-step-3-2-request-mem-pod.yaml
pod/mem-pod1 created
pod/mem-pod2 created
pod/mem-pod3 created



[root@k8s-master ~]# kubectl  get  pods  -n  resource
NAME        READY   STATUS    RESTARTS   AGE
mem-pod1   1/1        Running     0                 17s
mem-pod2   1/1        Running     0                 17s
mem-pod3   1/1        Running     0                 17s



[root@k8s-master ~]# kubectl  describe  pods  mem-pod1  -n resource
Name:             	mem-pod1
Namespace:        	resource
Priority:         	0
Service Account:  	default
Node:             	k8s-worker1/192.168.10.101
Start Time:       	Thu, 13 Aug 2026 12:44:08 +0900
Labels:           	<none>
Annotations:      	<none>
Status:           	Running
IP:               	10.244.1.22
IPs:
  IP:  10.244.1.22
Containers:
  mem-container:
    Container ID:	containerd://5bb9bc819a2f23fa50daa707491e4f9d7dc0ffe790428179686f14062e326a20
    Image:          	nginx:1.31
    Image ID:       	docker.io/library/nginx@sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8
    Port:           	<none>
    Host Port:      	<none>
    State:          	Running
      Started:      	Thu, 13 Aug 2026 12:44:08 +0900
    Ready:          	True
    Restart Count: 	0
    Requests:
      cpu:        	1
      memory:     	512Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fr7lg (ro)
~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~
```

### Limits-EX1) 제한(limits) 총합 제한

resource에 `limits.cpu=2`, `limits.memory=1Gi`를 걸고 limit이 있는 파드로 총합을 채운 뒤 초과 생성이 거부되는지 확인한다.

- 네임스페이스: resource
- ResourceQuota: rq-limits-total

제한:
- limits.cpu: "2"
- limits.memory: 1Gi
- 파드(각각 limit 포함): limit-a: cpu 1, mem 512Mi / limit-b: cpu 1, mem 512Mi / limit-c: cpu 1, mem 512Mi (초과로 실패)

```
[root@k8s-master ~]# vi  rq-step1-limit-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-limits-total
  namespace: resource
spec:
  hard:
    limits.cpu: "2"
    limits.memory: "1Gi"



[root@k8s-master ~]# kubectl  apply  -f  rq-step1-limit-quota.yaml  --dry-run=server
resourcequota/rq-limits-total created (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  rq-step1-limit-quota.yaml
resourcequota/rq-limits-total created



[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME              REQUEST   	LIMIT                                   	AGE
rq-limits-total             	limits.cpu: 0/2, limits.memory: 0/1Gi	28s



[root@k8s-master ~]# kubectl  describe  resourcequotas  -n  resource
Name:            	rq-limits-total
Namespace:     	resource
Resource         	Used  Hard
--------       	----  ----
limits.cpu     	0       2
limits.memory	0       1Gi



[root@k8s-master ~]# vi  rq-step1-limit-quota-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-a
  namespace: resource
spec:
  containers:
  - name: limit-nginx
    image: nginx:1.31
    resources:
      limits:
        cpu: "1"
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: limit-b
  namespace: resource
spec:
  containers:
  - name: limit-nginx
    image: nginx:1.31
    resources:
      limits:
        cpu: "1"
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: limit-c
  namespace: resource
spec:
  containers:
  - name: limit-nginx
    image: nginx:1.31
    resources:
      limits:
        cpu: "1"
        memory: "512Mi"




[root@k8s-master ~]# kubectl  apply  -f  rq-step1-limit-quota-pods.yaml  --dry-run=server
pod/limit-a created (server dry run)
pod/limit-b created (server dry run)
pod/limit-c created (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  rq-step1-limit-quota-pods.yaml
pod/limit-a created
pod/limit-b created
Error from server (Forbidden): error when creating "rq-step1-limit-quota-pods.yaml": pods "limit-c" is forbidden: exceeded quota: rq-limits-total, requested: limits.cpu=1,limits.memory=512Mi, used: limits.cpu=2,limits.memory=1Gi, limited: limits.cpu=2,limits.memory=1Gi



[root@k8s-master ~]# kubectl  get  resourcequotas  -n  resource
NAME              REQUEST   	LIMIT                                     	 AGE
rq-limits-total             	limits.cpu: 2/2, limits.memory: 1Gi/1Gi	 19m



[root@k8s-master ~]# kubectl  describe  resourcequotas  -n  resource
Name:          	rq-limits-total
Namespace:     	resource
Resource       	Used  Hard
--------       	----  ----
limits.cpu     	2       2
limits.memory  	1Gi   1Gi



	# limit를 사용하여 제한하게되면 limit와 request가 모두 적용된다.
[root@k8s-master ~]# kubectl  describe  pods  limit-a  -n  resource
Name:             limit-a
Namespace:        resource
~~~~~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~~~~~
    Container ID:	containerd://90c29105fe708aa0bd4c8af175aa6d6ab24d130f45c10047e16f6f7e0f1d1cea
    Image:          	nginx:1.31
    Image ID:       	docker.io/library/nginx@sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8
    Port:           	<none>
    Host Port:      	<none>
    State:          	Running
      Started:      	Thu, 13 Aug 2026 13:08:38 +0900
    Ready:          	True
    Restart Count:  	0
    Limits:
      cpu:     	1
      memory:  	512Mi
    Requests:
      cpu:        	1
      memory:     	512Mi
    Environment:  	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-b78wg (ro)
~~~~~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~~~~~



[root@k8s-master ~]# kubectl  delete  -f  rq-step1-limit-quota.yaml
resourcequota "rq-limits-total" deleted from resource namespace



[root@k8s-master ~]# kubectl  delete  -f  rq-step1-limit-quota-pods.yaml
pod "limit-a" deleted from resource namespace
pod "limit-b" deleted from resource namespace
Error from server (NotFound): error when deleting "rq-step1-limit-quota-pods.yaml": pods "limit-c" not found
```

### Limits-EX2) requests/limits quota와 resources 미기재 파드

resource에 requests/limits quota를 걸고 resources 없는 파드(bad-pod)가 거부되는지 확인하고, resources 있는 파드(good-pod)는 성공하는지 확인한다.

- 운영에서 리소스 미기재 파드 생성 금지를 왜 하는지 확인
- quota가 계산을 위해 requests/limits 명시를 사실상 강제하는 상황을 확인

- 네임스페이스: resource
- ResourceQuota: rq-require-compute

제한:
- requests.cpu: "1"
- requests.memory: 512Mi
- limits.cpu: "2"
- limits.memory: 1Gi

- bad-pod: resources 없음(실패)
- good-pod: requests/limits 있음(성공)

```
[root@k8s-master ~]# vi rq-step2-limit-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-request-limits-total
  namespace: resource
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "512Mi"
    limits.cpu: "2"
    limits.memory: "1Gi"



[root@k8s-master ~]# kubectl  apply  -f  rq-step2-limit-quota.yaml  --dry-run=server
resourcequota/rq-limits-total created (server dry run)

-pod-a : CPU 300M : 생성 O
-pod-b : CPU 300M : 생성 O
-pod-c : CPU 300M : 생성 O
-pod-d : CPU 300M : 생성 X  = request.cpu 1000M을 초과

-pod-a, pod-b, pod-c의 CPU 총 합이 2000M을 초과할 수 없다.
```

**실습 1 (request 설정 범위 이내에서 pod 생성)**

- requests.cpu = 400m + 300m + 300m = 1000m
- requests.memory = 256Mi + 128Mi + 128Mi = 512Mi
- limits.cpu = 600m + 600m + 600m = 1800m
- limits.memory = 512Mi + 256Mi + 256Mi = 1024Mi

```
[root@k8s-master ~]# vi rq-step2-limit-quota-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "400m"
        memory: "256Mi"
      limits:
        cpu: "600m"
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "300m"
        memory: "128Mi"
      limits:
        cpu: "600m"
        memory: "256Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-c
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "300m"
        memory: "128Mi"
      limits:
        cpu: "600m"
        memory: "256Mi"


:wq 



[root@k8s-master ~]# kubectl apply -f rq-step2-limit-quota-pod.yaml  --dry-run=server
pod/pod-a created (server dry run)
pod/pod-b created (server dry run)
pod/pod-c created (server dry run)




[root@k8s-master ~]# kubectl apply -f rq-step2-limit-quota-pod.yaml
pod/pod-a created
pod/pod-b created
pod/pod-c created


[root@k8s-master ~]# kubectl  get  resourcequotas -n resource
NAME                      	REQUEST                                           	  LIMIT                                         		AGE
rq-request-limits-total	requests.cpu: 1/1, requests.memory: 512Mi/512Mi	  limits.cpu: 1800m/2, limits.memory: 1Gi/1Gi	3m29s




[root@k8s-master ~]# kubectl delete -f rq-step2-limit-quota-pod.yaml
pod "pod-a" deleted from resource namespace
pod "pod-b" deleted from resource namespace
pod "pod-c" deleted from resource namespace
```

**실습 2 (request 설정 범위 초과로 pod 생성)**

- requests.cpu = 400m + 400m + 300m = 1100m
- requests.memory = 256Mi + 256Mi + 128Mi = 640Mi
- limits.cpu = 600m + 600m + 600m = 1800m
- limits.memory = 512Mi + 512Mi + 256Mi = 1280Mi

```
[root@k8s-master ~]# vi rq-step2-limit-quota-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "400m"
        memory: "256Mi"
      limits:
        cpu: "600m"
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "400m"
        memory: "256Mi"
      limits:
        cpu: "600m"
        memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-c
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "300m"
        memory: "128Mi"
      limits:
        cpu: "600m"
        memory: "256Mi"


[root@k8s-master ~]# kubectl apply -f rq-step2-limit-quota-pod.yaml
pod/pod-a created
pod/pod-b created
Error from server (Forbidden): error when creating "rq-step2-limit-quota-pod.yaml": pods "pod-c" is forbidden: exceeded quota: rq-request-limits-total, requested: limits.memory=256Mi,requests.cpu=300m,requests.memory=128Mi, used: limits.memory=1Gi,requests.cpu=800m,requests.memory=512Mi, limited: limits.memory=1Gi,requests.cpu=1,requests.memory=512Mi



[root@k8s-master ~]# kubectl describe  resourcequotas  rq-request-limits-total  --namespace resource 
Name:            	rq-request-limits-total
Namespace:       	resource
Resource         	Used     Hard
--------         	----     ----
limits.cpu       	1200m   2
limits.memory    	1Gi      1Gi
requests.cpu     	800m    1
requests.memory	512Mi   512Mi



[root@k8s-master ~]# kubectl get resourcequotas rq-request-limits-total  --namespace resource
NAME                      	REQUEST                                              		LIMIT                                         		AGE
rq-request-limits-total	 requests.cpu: 800m/1, requests.memory: 512Mi/512Mi	limits.cpu: 1200m/2, limits.memory: 1Gi/1Gi	11m




[root@k8s-master ~]# kubectl delete -f rq-step2-limit-quota-pod.yaml
pod "pod-a" deleted from resource namespace
pod "pod-b" deleted from resource namespace
Error from server (NotFound): error when deleting "rq-step2-limit-quota-pod.yaml": pods "pod-c" not found




# resources 없는 파드 생성
[root@k8s-master ~]# vi rq-step2-no-limit-quota-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx

: wq


[root@k8s-master ~]# kubectl  apply  -f  rq-step2-no-limit-quota-pod.yaml
Error from server (Forbidden): error when creating "rq-step2-no-limit-quota-pod.yaml": pods "bad-pod" is forbidden: failed quota: rq-request-limits-total: must specify limits.cpu for: nginx; limits.memory for: nginx; requests.cpu for: nginx; requests.memory for: nginx
```

- ResourceQuota가 보는 항목은 4개다: requests.cpu, requests.memory, limits.cpu, limits.memory
- 파드에 resources가 없으면(= requests/limits 미기재) 쿠버네티스 입장에서는 이 값들을 알 수 없다.
- ResourceQuota는 새 파드가 생성될 때, 현재 네임스페이스의 사용량(Used)에 새 파드의 requests/limits 값을 더해서 설정된 최대값(Hard)을 초과하는지 검사한다. 그런데 파드에 requests/limits 값이 없으면 ResourceQuota가 요구하는 자원 값을 계산할 수 없기 때문에 해당 파드의 생성이 Forbidden으로 거부된다.
- 즉, ResourceQuota가 CPU/Memory의 requests와 limits를 관리하는 네임스페이스에서는 파드도 해당 resources 값을 명시해야 한다.

### LimitRange 실습

LimitRange는 Namespace 안에서 생성되는 Pod/Container의 CPU와 Memory에 기본값, 최소값, 최대값 등을 설정하는 Kubernetes 리소스다.

- ResourceQuota: Namespace 전체가 사용할 수 있는 총량 제한
- LimitRange: Namespace 안에서 생성되는 개별 Pod의 자원 사용 규칙 설정

LimitRange를 사용하는 이유는 Pod를 만들 때 사용자가 resources를 아예 작성하지 않을 수도 있기 때문이다.

예:

```
containers:
- name: nginx
  image: nginx
```

이런 Pod가 많아지면 CPU/Memory 관리가 어려워질 수 있다.

LimitRange를 설정하지 않으면:
- resources를 안 적은 Pod → LimitRange의 default / defaultRequest 값이 자동 적용
- resources를 직접 적은 Pod → 사용자가 적은 requests / limits 값 사용. 단, LimitRange의 min / max 범위를 벗어나면 생성 거부

**LimitRange의 주요 설정값**

| 설정값 | 의미 |
|---|---|
| default | Pod/Container에 limits를 작성하지 않았을 때 자동으로 적용되는 기본 Limit 값 |
| defaultRequest | Pod/Container에 requests를 작성하지 않았을 때 자동으로 적용되는 기본 Request 값 |
| min | 사용자가 설정할 수 있는 최소 requests/limits 값. 이 값보다 작게 설정하면 Pod 생성이 거부될 수 있음 |
| max | 사용자가 설정할 수 있는 최대 requests/limits 값. 이 값보다 크게 설정하면 Pod 생성이 거부됨 |

### EX1) LimitRange 최소값 제한

limit의 min CPU 200m, min Memory 128Mi를 설정하고 그보다 작은 파드를 생성해 실패를 확인한다.

- 최소값: cpu 200m, memory 128Mi

```
[root@k8s-master ~]# vi  step1-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-min-max-default
  namespace: resource

spec:
  limits:
  - type: Container
    min:
      cpu: "200m"
      memory: "128Mi"

:wq


[root@k8s-master ~]# kubectl  apply  -f  step1-limitrange.yaml  --dry-run=server
limitrange/lr-min-max-default created (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  step1-limitrange.yaml
limitrange/lr-min-max-default created


[root@k8s-master ~]# kubectl  get  limitranges lr-min-max-default  -n  resource
NAME                 	  CREATED AT
lr-min-max-default	  2026-08-13T08:32:06Z


[root@k8s-master ~]# kubectl  describe  limitranges lr-min-max-default  -n  resource
Name:       lr-min-max-default
Namespace:  resource
Type     	   Resource  Min     Max  Default Request   Default Limit    Max Limit/Request Ratio
---- 	   --------  ---     ---   ---------------  -------------  -----------------------
Container	   cpu   	   200m   -      200m                 -                   -
Container	   memory	   128Mi  -      128Mi                -                   -
```

Kubernetes가 해당 LimitRange에서 min만 있고 defaultRequest가 없을 때, 최소 요청값을 기본 request로 취급한다.

```
	# 최소보다 큰 pod 생성
[root@k8s-master ~]# vi  step1-normal-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal-pod
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"



[root@k8s-master ~]# kubectl  apply -f  step1-normal-pod.yaml  --dry-run=server
pod/normal-pod created (server dry run)


[root@k8s-master ~]# kubectl  apply -f  step1-normal-pod.yaml
pod/normal-pod created




	# 최소보다 작은 pod 생성
[root@k8s-master ~]# vi  step1-min-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: min-pod
  namespace: resource
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    resources:
      requests:
        cpu: "50m"
        memory: "256Mi"


[root@k8s-master ~]# kubectl  apply -f  step1-min-pod.yaml
Error from server (Forbidden): error when creating "step1-min-pod.yaml": pods "min-pod" is forbidden: minimum cpu usage per Container is 200m, but request is 50m




[root@k8s-master ~]# kubectl  delete  -f  step1-limitrange.yaml


[root@k8s-master ~]# kubectl  delete  -f  step1-normal-pod.yaml
```

### EX2) LimitRange min/max/default/defaultRequest 종합 설정

min(cpu 150m, memory 128Mi)보다 작은 requests/limits를 가진 컨테이너는 생성 거부되고, max(cpu 300m, memory 512Mi)보다 큰 requests/limits를 가진 컨테이너도 생성 거부되는 것을 확인한다.

```
[root@k8s-master ~]# vi  step2-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-min-max-default
  namespace: resource

spec:
  limits:
  - type: Container
    min:
      cpu: "150m"
      memory: "128Mi"
    max:
      cpu: "300m"
      memory: "512Mi"
    default:		# limit 미 설정시 적용
      cpu: "250m"
      memory: "256Mi"
    defaultRequest:		# request 미 설정시 적용
      cpu: "200m"
      memory: "128Mi"

:wq
```

---


## 2. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### CPU / Memory Limit 초과 동작 확인


resource 네임스페이스에 limit-test-pod를 생성하고, CPU와 Memory Limit의 실제 동작을 확인한다.

- 네임스페이스: resource
- Pod: limit-test-pod
- 컨테이너: stress
- CPU Limit: 500m
- Memory Limit: 100Mi

CPU 1개를 최대한 사용하도록 부하를 발생시키고 CPU Limit을 초과했을 때 컨테이너가 종료되는지 확인한다. 그 후 같은 Pod 설정을 메모리 테스트용으로 변경하여 150Mi의 메모리를 사용하게 하고, Memory Limit 100Mi를 초과했을 때 OOMKilled가 발생하는지 확인한다.

```
[root@k8s-master ~]# vi  cpu-list-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-test-pod
  namespace: resource
spec:
  containers:
  - name: limt-test-container
    image: polinux/stress
    resources:
      limits:
        cpu: "500m"
        memory: "100Mi"
    command: ["stress"]
    args: ["--cpu", "1"]
```

- `polinux/stress`는 CPU, Memory 등에 강제로 부하를 발생시키는 stress 프로그램이 포함된 이미지다.
- 일반 nginx 이미지는 웹 서버를 실행하는 이미지라서 CPU나 메모리를 의도적으로 많이 사용하는 실습에는 적합하지 않다.

```
[root@k8s-master ~]# nproc		# CPU 코어 개수 확인
4


# cpu: "500m"
# memory: "100Mi"
# Docker 명령어 : stress --cpu 1
```

worker-node 1개가 CPU를 계속 최대한 사용하도록 부하를 발생시키지만, stress가 원하는 CPU는 약 1 CPU인데 Pod CPU Limit은 0.5 CPU이므로 실제로는 CPU Limit에 의해 제한된다.

```
[root@k8s-master ~]# kubectl  apply  -f  cpu-list-test.yaml  --dry-run=server
pod/limit-test-pod created (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  cpu-list-test.yaml
pod/limit-test-pod created



[root@k8s-master ~]# kubectl  get  pods  -n  resource
NAME             READY   STATUS    RESTARTS   AGE
limit-test-pod   1/1         Running     0                17s




[root@k8s-master ~]# kubectl  get  pods  -n  resource  -o wide
NAME             READY   STATUS    RESTARTS   AGE     IP             NODE            NOMINATED NODE   READINESS GATES
limit-test-pod   1/1         Running     0                3m38s   10.244.1.2   k8s-worker1   <none>                    <none>




[root@k8s-master ~]# kubectl  describe  pods  -n  resource
Name:             	limit-test-pod
Namespace:        	resource
Priority:         	0
Service Account:  	default
Node:             	k8s-worker1/192.168.10.101
Start Time:       	Fri, 14 Aug 2026 09:52:17 +0900
Labels:           	<none>
Annotations:      	<none>
Status:           	Running
IP:               	10.244.1.2
~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~
    State:          Running
      Started:      Fri, 14 Aug 2026 09:52:24 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  100Mi
    Requests:
      cpu:        500m
      memory:     100Mi
    Environment:  <none>
~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  43s   default-scheduler  Successfully assigned resource/limit-test-pod to k8s-worker1
  Normal  Pulling    42s   kubelet                 spec.containers{limt-test-container}: Pulling image "polinux/stress"
  Normal  Pulled     36s   kubelet                 spec.containers{limt-test-container}: Successfully pulled image "polinux/stress" in 5.718s (5.718s including waiting). Image size: 4041495 bytes.
  Normal  Created    36s   kubelet                spec.containers{limt-test-container}: Container created
  Normal  Started    36s   kubelet                 spec.containers{limt-test-container}: Container started




[root@k8s-worker1 ~]# ps -eo pid,ppid,comm,args,%cpu,%mem --sort=-%cpu | head -10
    PID   PPID	COMMAND	  COMMAND                   	  %CPU	%MEM
   8598   8586 	stress          	  stress --cpu 1              	  49.9  	0.0
   2258       1 	kubelet         	  /usr/bin/kubelet --bootstra	  1.1  	4.3
    945       1 	containerd      	  /usr/bin/containerd          	  0.6  	3.5
   2307       1 	containerd-shim	  /usr/bin/containerd-shim-ru	  0.1  	0.8
   2312       1 	containerd-shim	  /usr/bin/containerd-shim-ru	  0.1  	0.9
   2637   2307 	flanneld        	  /opt/bin/flanneld --ip-masq	  0.1  	2.5
      1        0 	systemd         	  /usr/lib/systemd/systemd --	  0.0  	0.7
      2        0 	kthreadd        	  [kthreadd]                   	  0.0  	0.0
      3        2 	pool_workqueue_	  [pool_workqueue_]            	  0.0  	0.0


# ps	: 현재 실행 중인 프로세스 정보를 확인하는 명령입니다.
# -e	: 시스템의 모든 프로세스를 출력합니다.
# -o	: 출력할 항목을 직접 지정합니다.

# pid,ppid,comm,args,%cpu,%mem

# pid = 프로세스 ID
# ppid = 부모 프로세스 ID
# comm = 실행 중인 명령어 이름
# args = 실제 실행 명령어와 옵션
# %cpu = CPU 사용률
# %mem	= 메모리 사용률



[root@k8s-master ~]# kubectl  get  pods  -n  resource
NAME             READY   STATUS    RESTARTS   AGE
limit-test-pod   1/1         Running     0                12m



[root@k8s-master ~]# kubectl  delete  -f  cpu-list-test.yaml
pod "limit-test-pod" deleted from resource namespace
```

**Memory 테스트**

```
[root@k8s-master ~]# vi cpu-list-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-test-pod
  namespace: resource
spec:
  containers:
  - name: limt-test-container
    image: polinux/stress
    resources:
      limits:
        cpu: "500m"
        memory: "100Mi"
    command: ["stress"]
    args:
    - "--vm"
    - "1"
    - "--vm-bytes"
    - "150M"
    - "--vm-hang"
    - "1"


 "--vm 1"		: stress를 사용해서 메모리 부하를 발생시키는 vm 워커 1개 생성
 "--vm-bytes 150M"	: vm 워커가 150M의 부하를 발생

-전체 명령어 : stress  --vm 1  --vm-bytes 150M  --vm-hang 1


[root@k8s-master ~]# kubectl  apply  -f  cpu-list-test.yaml  --dry-run=server
pod/limit-test-pod created (server dry run



[root@k8s-master ~]# kubectl  get  pods  -n  resource
NAME             READY   STATUS       RESTARTS      AGE
limit-test-pod   0/1         OOMKilled    2 (23s ago)       27s


[root@k8s-master ~]# kubectl  get  pods  -o  wide  -n  resource
NAME             READY   STATUS      RESTARTS      AGE    IP             NODE           NOMINATED NODE   READINESS GATES
limit-test-pod   0/    1     OOMKilled   4 (67s ago)      116s     10.244.1.3   k8s-worker1   <none>                  <none>
```

메모리 요청량(150M)이 Memory Limit(100Mi)을 초과하면 커널의 OOM Killer가 컨테이너를 강제 종료시키고, Pod STATUS는 `OOMKilled`로 표시된다. CPU Limit 초과는 컨테이너가 죽지 않고 CPU 사용량만 제한(throttling)되는 반면, Memory Limit 초과는 컨테이너 자체가 종료(OOMKilled)된다는 점이 결정적인 차이다.

### 그 외 트러블슈팅 포인트

- `kubectl get resourcequotas`의 REQUEST/LIMIT 컬럼에서 `used/hard` 형식으로 현재 사용량과 한도를 바로 확인할 수 있다. 파드 생성이 Forbidden으로 거부되면 먼저 이 값을 확인한다.
- 파드에 `resources`(requests/limits)를 명시하지 않은 상태에서 ResourceQuota가 requests/limits 항목을 관리하고 있으면 파드 생성 자체가 거부되므로, 반드시 LimitRange의 default/defaultRequest를 설정하거나 파드에 직접 resources를 명시해야 한다.

---

>  **핵심 요약**
> - ResourceQuota는 namespace 전체의 CPU/Memory/Pod 개수 등 자원 총량을 제한하고, LimitRange는 개별 파드/컨테이너의 최소/최대/기본(default, defaultRequest) 자원값을 제한한다 — 둘을 함께 써서 운영 단위를 통제한다
> - ResourceQuota가 requests/limits 항목을 관리 중인 namespace에서는 파드에 `resources`를 명시하지 않으면 생성 자체가 Forbidden으로 거부되므로, LimitRange의 default/defaultRequest를 설정하거나 파드에 직접 resources를 명시해야 한다
> - `kubectl get resourcequotas`의 REQUEST/LIMIT 컬럼에서 `used/hard` 형식으로 현재 사용량과 한도를 바로 확인할 수 있다. 파드 생성이 Forbidden으로 거부되면 먼저 이 값을 확인한다
> - CPU Limit 초과는 컨테이너가 죽지 않고 CPU 사용량만 제한(throttling)되지만, Memory Limit 초과는 커널의 OOM Killer가 컨테이너를 강제 종료시켜 Pod STATUS가 `OOMKilled`로 표시된다 — 이 둘의 동작 차이가 결정적이다
> - 관련: 1.  Kubernetes - 설치 · 2.  Kubernetes - Pod 생성 · 4.  Kubernetes - Namespace
