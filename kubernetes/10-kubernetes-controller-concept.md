# 🎛️ Kubernetes - Controller 개념과 ReplicationController

> **Tag:** #Kubernetes #Controller #ReplicationController #ReconciliationLoop #부트캠프
> **핵심 요약:** Pod 생성 시 API Server-Controller-Scheduler-kubelet이 협력하는 동작 순서, Controller의 Reconciliation Loop 개념, ReplicationController의 YAML 구조와 실습

---

## 1. 📖 개요 (Overview)

### 파드 3개 요청시 쿠버네티스 동작 순서

상황: 사용자가 파드 3개를 유지해라라고 요청 (Deployment / ReplicaSet 생성)

**1단계. API Server**

- 사용자의 요청(kubectl apply 등)을 API Server가 가장 먼저 받는다.
- 요청 내용을 검증한다.
- 파드 3개가 필요하다는 정보를 etcd에 저장한다.

핵심
- API Server는 명령을 전달하고 기록만 한다.
- 직접 파드를 만들지는 않는다.

**2단계. Controller**

- 컨트롤러는 API Server를 계속 감시하고 있다.
- etcd에 저장된 정보를 보고 파드가 3개 필요하지만 아직 없다는 것을 확인한다. 그래서 API Server에게 Pod 객체 3개를 생성해 달라고 요청한다.

핵심
- 컨트롤러는 pod 개수 유지가 역할이다.
- 노드 배치나 실행은 하지 않는다.

**3단계. API Server**

- 컨트롤러의 요청을 받아 Pod 객체 3개를 생성한다.
- 생성된 Pod 정보는 etcd에 저장된다.
- 이 시점의 파드는 아직 어느 노드에도 배치되지 않았다.

**4단계. Scheduler**

- 스케줄러는 nodeName이 없는 Pod를 발견한다.
- etcd에 있는 노드 상태(리소스, 사용량 등)를 참고한다.

판단 결과
- 파드 2개 : 노드1
- 파드 1개 : 노드2

- Pod 객체에 nodeName을 기록한다 (배치 결정).

핵심
- 스케줄러는 배치만 결정한다.
- 실제 실행은 하지 않는다.

**5단계. API Server**

- 스케줄러가 기록한 배치 결과를 etcd에 저장한다.
- 이제 해당 파드들은 어느 노드에서 실행할지가 확정된 상태다.

**6단계. kubelet (실제 실행 주체)**

- 각 노드의 kubelet은 API Server를 계속 감시한다.
- 자기 노드로 지정된 Pod를 발견한다.
- 노드1 kubelet : 파드 2개 실행
- 노드2 kubelet : 파드 1개 실행
- 컨테이너 런타임에게 요청해서 이미지 다운로드 → 컨테이너 생성 → 실행을 수행한다.
- 실제로 파드를 만드는 주체는 kubelet이다.

### Kubernetes Controller

Controller는 쿠버네티스에서 현재 상태를 감시하고, 원하는 상태(desired state)로 맞추기 위해 자동으로 조정하는 관리자 역할이다.

쿠버네티스는 사람이 직접 하나하나 관리하는 시스템이 아니라 원하는 상태만 선언하면, 나머지는 컨트롤러가 자동으로 맞춰주는 시스템이다.

이 자동 조정의 핵심 주체가 바로 Controller이다.

**왜 Controller가 필요한가?**

예를 들어 다음과 같은 상황을 생각해보자.
- Pod를 3개 유지하고 싶다 그런데 그 중 1개가 죽었다
- 사람이 직접 확인하고 다시 Pod를 만들어야 할까?

쿠버네티스에서는 사람이 개입하지 않아도 Controller가 자동으로 Pod를 생성/수정/삭제를 자동으로 진행한다.

Controller의 역할은 다음과 같다.
- 상태 감시 (watch)
- 차이 발견 (현재 상태 vs 원하는 상태)
- 자동 복구 (create / delete / restart)

**Controller의 기본 동작 개념**

Controller는 항상 아래 구조로 동작한다.

1단계) 사용자가 YAML로 원하는 상태를 선언한다. (예: Pod를 3개 유지하겠다)

2단계) Controller가 현재 상태를 계속 감시한다 (예: 지금 Pod가 몇 개인지 확인)

3단계) 현재 상태와 원하는 상태를 비교한다

4단계) 차이가 있으면 자동으로 조정한다
- 부족하면 생성
- 많으면 삭제
- 죽었으면 재생성
- 이 과정을 무한 반복한다.

이 구조를 **Reconciliation Loop(상태 동기화 루프)**라고 부른다.

**Controller가 관리하는 대표적인 대상**

Controller는 보통 직접 Pod를 관리하지 않고, Pod를 관리하는 상위 리소스를 관리한다.

대표적인 Controller들
- ReplicationController
- ReplicaSet Controller
- Deployment Controller
- DaemonSet Controller
- StatefulSet Controller
- Job / CronJob Controller

---

## 2. 🛠️ ReplicationController (RC)

ReplicationController는 Pod의 개수를 항상 일정하게 유지해주는 Controller이다. 즉, Pod가 죽거나 삭제되면 자동으로 다시 생성해서 지정한 개수(replicas)를 보장한다.

### ReplicationController의 역할

ReplicationController는 다음을 보장한다.
- Pod가 지정한 개수만큼 항상 존재
- Pod가 죽으면 자동 재생성
- Pod를 수동으로 삭제해도 다시 생성

중요한 점: ReplicationController는 Pod 자체를 직접 감시하는 것이 아니라 Label을 기준으로 Pod를 관리한다.

### Label 기반 관리 개념

ReplicationController는 아래 두 가지를 항상 확인한다.
- 몇 개를 유지할 것인가? (replicas)
- 어떤 Pod를 관리할 것인가? (selector)

예시 개념
```
label: app=web
replicas: 3
```

의미
- app=web 라벨이 붙은 Pod를
- 항상 3개 유지하겠다

만약 app=web Pod가 2개 → 1개 생성
만약 app=web Pod가 4개 → 1개 삭제

### ReplicationController 동작 흐름

1단계) ReplicationController 생성 / replicas 값 설정 / selector(label) 설정

2단계) Controller가 현재 Pod 개수 확인

3단계) 원하는 개수와 비교

4단계) 부족하면 Pod 생성 / 초과하면 Pod 삭제

- 이 과정을 계속 반복한다.

장애 상황 예시
- replicas = 3
- 현재 Pod = 3

문제 발생
- Pod 1개가 노드 장애로 종료됨

ReplicationController 동작
- 현재 Pod 수 = 2
- 원하는 Pod 수 = 3
- Pod 1개 자동 생성

결과: 사용자는 아무것도 하지 않았지만 Pod는 다시 3개가 된다.

### ReplicationController YAML

기본 구성 요소

- replicas : 유지해야 하는 Pod 개수
- selector : ReplicationController가 관리할 Pod를 선택하는 기준 (라벨)
- template : Pod가 부족할 때 새로 생성할 Pod의 설계도

**replicas**

- 항상 몇 개를 켜두고 있을지 목표 개수다.
- ReplicationController는 실제 개수와 replicas를 계속 비교한다.

동작 예시
- replicas: 3 인데 현재 Pod가 2개면 → 1개를 더 만든다.
- replicas: 3 인데 현재 Pod가 4개면 → 1개를 삭제한다.

**selector**

- 이 RC가 어떤 Pod들을 내 관리 대상으로 볼 것인가를 고르는 필터다.
- 라벨(key: value)이 일치하는 Pod를 관리 대상으로 삼는다.

예시
```
replicas: 3

selector:
  app: web
```

ReplicationController는 현재 실행 중인 Pod들 중에서 라벨이 app: web 인 Pod만을 관리 대상으로 삼는다. 이 관리 대상 Pod의 개수를 지속적으로 확인하면서, 개수가 3개보다 적으면 template에 정의된 설정을 사용해 Pod를 추가로 생성하고, 개수가 3개보다 많으면 초과된 Pod를 삭제한다. 이 과정을 반복함으로써 항상 app: web 라벨을 가진 Pod가 3개 유지되도록 동작한다.

중요한 포인트(중요)
- selector 조건이 너무 넓으면 위험하다.
- 예: app=web 인 Pod가 이미 다른 용도로 존재하는데 RC selector가 app=web이면, RC가 그 Pod까지 자기 관리 대상으로 착각할 수 있다.
- 그래서 보통 RC/RS/Deploy에서 라벨을 목적별로 명확히 잡는다.

**template : Pod가 부족할 때 새로 생성할 Pod의 설계도**

- RC가 Pod를 새로 만들어야 할 때 참고하는 Pod 생성 템플릿이다.

template 안에 들어가는 것
- metadata.labels : 새로 만들어질 Pod에 붙일 라벨
- spec.containers : 컨테이너 이름, 이미지, 포트, 환경변수, 볼륨 등 Pod 실행 정보

가장 중요한 규칙
- template.metadata.labels는 selector와 반드시 맞아야 한다.
- 왜냐하면 RC가 Pod를 만들었는데 라벨이 selector와 다르면, RC가 내가 만든 Pod를 자기 Pod로 인식하지 못한다.
- RC는 Pod가 부족하다고 판단해서 계속 새 Pod를 만들 수 있다(무한 생성 상황)

```yaml
apiVersion: v1
kind: ReplicationController

metadata:
  name: <RC 이름>

spec:
  replicas: 3

  selector:
    app: web		<--- 반드시 같은 값을 갖아야 한다. (값이 다를 경우 pod를 무한 생성할 수 있다.)

  template:
    metadata:
      labels:
        app: web	<--- 반드시 같은 값을 갖아야 한다. (값이 다를 경우 pod를 무한 생성할 수 있다.)

    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 실습 — ReplicationController 생성과 동작 확인

```
[root@k8s-master ~]# vi rc-nginx.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-nginx

spec:
  replicas: 3
  selector:
    app: webui

  template:
    metadata:
      name: nginx-pod
      labels:
        app: webui

    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31
        ports:
        - containerPort: 80
```

```
	# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  rc-nginx.yaml  --dry-run=server
replicationcontroller/rc-nginx created (server dry run)



	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch



	# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  rc-nginx.yaml
replicationcontroller/rc-nginx created



	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME              READY   STATUS    		RESTARTS   AGE   IP       	   NODE     	NOMINATED NODE   READINESS GATES
rc-nginx-fq9sk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>          	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>
rc-nginx-fq9sk   0/1        Pending   		0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   k8s-worker2   	<none>           	  <none>
rc-nginx-fq9sk   0/1        ContainerCreating	0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   0/1        ContainerCreating   	0                 0s     <none>   	   k8s-worker2   	<none>           	  <none>
rc-nginx-nlp42   0/1        ContainerCreating   	0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   1/1        Running             	0                 1s     10.244.2.3	   k8s-worker2   	<none>           	  <none>
rc-nginx-nlp42   1/1        Running             	0                 1s     10.244.1.6	   k8s-worker1   	<none>           	  <none>
rc-nginx-fq9sk   1/1        Running             	0                 1s     10.244.1.5	   k8s-worker1   	<none>           	  <none>


[root@k8s-master ~]# kubectl  get  pods
NAME              READY   STATUS    RESTARTS   AGE
rc-nginx-fq9sk   1/1        Running     0                 3m19s
rc-nginx-ggjvk   1/1        Running     0                 3m19s
rc-nginx-nlp42   1/1        Running     0                 3m19s



[root@k8s-master ~]# kubectl  get  replicationcontrollers
NAME       DESIRED   CURRENT   READY   AGE
rc-nginx    3              3               3           3m56


[root@k8s-master ~]# kubectl  get  rc
NAME       DESIRED   CURRENT   READY   AGE
rc-nginx    3              3               3           3m5


-DESIRED	: 사용자가 원하는 Pod 개수 (3)
-CURRENT	: 현재 실제로 존재하는 Pod 개수 (3)
-READY		: 현재 정상적으로 서비스 가능한 Pod 개수 (3)



[root@k8s-master ~]# kubectl  describe  rc  rc-nginx
Name:         	rc-nginx
Namespace:	default
Selector:     	app=webui
Labels:       	app=webui
Annotations:  	<none>
Replicas:     	3 current / 3 desired
Pods Status:  	3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=webui
  Containers:
   nginx-container:
    Image:         	nginx:1.31
    Port:          	80/TCP
    Host Port:     	0/TCP
    Environment:   	<none>
    Mounts:        	<none>
  Volumes:         	<none>
  Node-Selectors:  	<none>
  Tolerations:     	<none>
Events:
  Type     Reason               Age     From                       Message
  ----     ------                ----    ----                        -------
  Normal  SuccessfulCreate  6m37s  replication-controller  Created pod: rc-nginx-fq9sk
  Normal  SuccessfulCreate  6m37s  replication-controller  Created pod: rc-nginx-nlp42
  Normal  SuccessfulCreate  6m37s  replication-controller  Created pod: rc-nginx-ggjvk



[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME              READY   STATUS    RESTARTS   AGE     LABELS
rc-nginx-fq9sk   1/1        Running     0                 9m41s   app=webui
rc-nginx-ggjvk   1/1        Running     0                 9m41s   app=webui
rc-nginx-nlp42   1/1        Running     0                 9m41s   app=webui
```

### RC selector와 같은 라벨을 가진 Pod를 별도로 생성했을 때

```
[root@k8s-master ~]# kubectl  run  httpd-web  --image=httpd:latest  --labels=app=webui  -o  yaml  >  rc-httpd.yaml



[root@k8s-master ~]# ls  -l  rc-httpd.yaml
-rw-r--r-- 1 root root 1583  8월 18 12:12 rc-httpd.yaml



[root@k8s-master ~]# vi  rc-httpd.yaml  (수정 버전)
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: webui
  name: httpd-web
  namespace: default
spec:
  containers:
  - image: httpd:latest
    name: httpd-web



	# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  rc-httpd.yaml  --dry-run=server
pod/httpd-web created (server dry run)



	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch



	# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  rc-httpd.yaml
pod/httpd-web created




	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME              READY   STATUS    		RESTARTS   AGE   IP       	   NODE     	NOMINATED NODE   READINESS GATES
rc-nginx-fq9sk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>          	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>
rc-nginx-fq9sk   0/1        Pending   		0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   k8s-worker2   	<none>           	  <none>
rc-nginx-fq9sk   0/1        ContainerCreating	0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   0/1        ContainerCreating   	0                 0s     <none>   	   k8s-worker2   	<none>           	  <none>
rc-nginx-nlp42   0/1        ContainerCreating   	0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   1/1        Running             	0                 1s     10.244.2.3	   k8s-worker2   	<none>           	  <none>
rc-nginx-nlp42   1/1        Running             	0                 1s     10.244.1.6	   k8s-worker1   	<none>           	  <none>
rc-nginx-fq9sk   1/1        Running             	0                 1s     10.244.1.5	   k8s-worker1   	<none>           	  <none>

httpd-web         0/1        Pending             	0                 0s    <none>   	   <none>        	<none>           	  <none>
httpd-web         0/1        Pending             	0                 0s    <none>  	   k8s-worker2   	<none>           	  <none>
httpd-web         0/1        ContainerCreating   	0                 0s    <none>   	   k8s-worker2   	<none>           	  <none>
httpd-web         0/1        Terminating         	0                 0s    <none> 	   k8s-worker2   	<none>           	  <none>
httpd-web         0/1        Terminating         	0                 1s    <none>  	   k8s-worker2   	<none>           	  <none>


-pod를 3개만 유지해야하는데 같은 LABEL로 pod를 생성하게되면 rc-controller에 의해 바로 삭제된다.
```

### ReplicationController pod 개수 수정 (`kubectl edit`)

```
[root@k8s-master ~]# kubectl  edit  rc  rc-nginx
  namespace: default
  resourceVersion: "167372"
  uid: 5e20c4e1-df53-4e74-8259-387ed5594e44
spec:
  replicas: 5	# 3에서 5로 수정
  selector:
    app: webui
  template:
    metadata:
      labels:
        app: webui
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.31
        imagePullPolicy: IfNotPresent
        name: nginx-container
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~



	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME              READY   STATUS    		RESTARTS   AGE   IP       	   NODE     	NOMINATED NODE   READINESS GATES
rc-nginx-fq9sk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>          	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>
rc-nginx-fq9sk   0/1        Pending   		0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   k8s-worker2   	<none>           	  <none>
rc-nginx-fq9sk   0/1        ContainerCreating	0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   0/1        ContainerCreating   	0                 0s     <none>   	   k8s-worker2   	<none>           	  <none>
rc-nginx-nlp42   0/1        ContainerCreating   	0                 0s     <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-ggjvk   1/1        Running             	0                 1s     10.244.2.3	   k8s-worker2   	<none>           	  <none>
rc-nginx-nlp42   1/1        Running             	0                 1s     10.244.1.6	   k8s-worker1   	<none>           	  <none>
rc-nginx-fq9sk   1/1        Running             	0                 1s     10.244.1.5	   k8s-worker1   	<none>           	  <none>

httpd-web         0/1        Pending             	0                 0s    <none>   	   <none>        	<none>           	  <none>
httpd-web         0/1        Pending             	0                 0s    <none>  	   k8s-worker2   	<none>           	  <none>
httpd-web         0/1        ContainerCreating   	0                 0s    <none>   	   k8s-worker2   	<none>           	  <none>
httpd-web         0/1        Terminating         	0                 0s    <none> 	   k8s-worker2   	<none>           	  <none>
httpd-web         0/1        Terminating         	0                 1s    <none>  	   k8s-worker2   	<none>           	  <none>


rc-nginx-2sn7n   0/1     Pending                  	0                 0s    <none>  	   <none>        	<none>           	  <none>
rc-nginx-gr57d   0/1     Pending                  	0                 0s    <none>    	   <none>        	<none>           	  <none>
rc-nginx-2sn7n   0/1     Pending                  	0                 0s    <none>    	   k8s-worker2   	<none>           	  <none>
rc-nginx-gr57d   0/1     Pending                  	0                 0s    <none>    	   k8s-worker1   	<none>           	  <none>
rc-nginx-2sn7n   0/1     ContainerCreating        	0                 0s    <none>  	   k8s-worker2   	<none>           	  <none>
rc-nginx-gr57d   0/1     ContainerCreating        	0                 0s    <none>   	   k8s-worker1   	<none>           	  <none>
rc-nginx-gr57d   1/1     Running                  	0                 1s    10.244.1.7	   k8s-worker1   	<none>           	  <none>
rc-nginx-2sn7n   1/1     Running                  	0                 1s    10.244.2.4	   k8s-worker2   	<none>           	  <none>




[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME               	READY   STATUS    RESTARTS   AGE    LABELS
rc-nginx-2sn7n	1/1         Running     0                2m8s   app=webui
rc-nginx-fq9sk   	1/1         Running     0                36m    app=webui
rc-nginx-ggjvk   	1/1         Running     0                36m    app=webui
rc-nginx-gr57d   	1/1         Running     0                2m8s   app=webui
rc-nginx-nlp42   	1/1         Running     0                36m    app=webui
```

### ReplicationController pod 개수 수정 (명령어 `kubectl scale`)

```
[root@k8s-master ~]# kubectl  scale  rc  rc-nginx  --replicas=2


	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
rc-nginx-gr57d   1/1     Terminating              0          4m3s   10.244.1.7   k8s-worker1   <none>           <none>
rc-nginx-2sn7n   1/1     Terminating              0          4m3s   10.244.2.4   k8s-worker2   <none>           <none>
rc-nginx-fq9sk   1/1     Terminating              0          38m    10.244.1.5   k8s-worker1   <none>           <none>
rc-nginx-gr57d   1/1     Terminating              0          4m3s   10.244.1.7   k8s-worker1   <none>           <none>
rc-nginx-2sn7n   1/1     Terminating              0          4m3s   10.244.2.4   k8s-worker2   <none>           <none>
rc-nginx-fq9sk   1/1     Terminating              0          38m    10.244.1.5   k8s-worker1   <none>           <none>
rc-nginx-2sn7n   0/1     Completed                0          4m3s   10.244.2.4   k8s-worker2   <none>           <none>
rc-nginx-gr57d   0/1     Completed                0          4m3s   10.244.1.7   k8s-worker1   <none>           <none>
rc-nginx-fq9sk   0/1     Completed                0          38m    10.244.1.5   k8s-worker1   <none>           <none>
rc-nginx-2sn7n   0/1     Completed                0          4m3s   10.244.2.4   k8s-worker2   <none>           <none>
rc-nginx-2sn7n   0/1     Completed                0          4m3s   10.244.2.4   k8s-worker2   <none>           <none>
rc-nginx-fq9sk   0/1     Completed                0          38m    10.244.1.5   k8s-worker1   <none>           <none>
rc-nginx-fq9sk   0/1     Completed                0          38m    10.244.1.5   k8s-worker1   <none>           <none>
rc-nginx-gr57d   0/1     Completed                0          4m3s   10.244.1.7   k8s-worker1   <none>           <none>
rc-nginx-gr57d   0/1     Completed                0          4m3s   10.244.1.7   k8s-worker1   <none>           <none>



[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME               	READY   STATUS    RESTARTS   AGE    LABELS
rc-nginx-ggjvk   	1/1         Running     0                36m    app=webui
rc-nginx-nlp42   	1/1         Running     0                36m    app=webui



-특정 시간대에 트래픽이 몰리면 pod를 확장하고 특정 시간대에 트래픽이 줄어들면 축소할 수 있다.
```

### ReplicationController의 template 이미지 변경 — 기존 Pod에는 반영되지 않는다

```
[root@k8s-master ~]# kubectl  edit  rc  rc-nginx
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
spec:
  replicas: 2
  selector:
    app: webui
  template:
    metadata:
      labels:
        app: webui
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.31		<--- 현재 이미지 = nginx:1.31  -->  nginx:1.31.3으로 수정
        imagePullPolicy: IfNotPresent
        name: nginx-container
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
:wq



	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME              READY   STATUS    		RESTARTS   AGE   IP       	   NODE     	NOMINATED NODE   READINESS GATES
rc-nginx-ggjvk   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>          	  <none>
rc-nginx-nlp42   0/1        Pending   		0                 0s     <none>   	   <none>   	<none>           	  <none>


[root@k8s-master ~]# kubectl describe pod rc-nginx-ggjvk	
Name:             	rc-nginx-ggjvk
Namespace:        	default
Priority:         	0
Service Account:  	default
Node:             	k8s-worker2/192.168.10.102
Start Time:       	Tue, 18 Aug 2026 12:00:23 +0900
Labels:           	app=webui
Annotations:      	<none>
Status:           	Running
IP:               	10.244.2.3
IPs:
  IP:           	10.244.2.3
Controlled By:  ReplicationController/rc-nginx
Containers:
  nginx-container:
    Container ID:	containerd://5f3d6f6ee1d25afd61e71caf18176f36ac7904f0be62054563e5a00959242e57
    Image:          	nginx:1.31
    Image ID:       	docker.io/library/nginx@sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8
    Port:          	80/TCP
    Host Port:      	0/TCP
    State:          	Running
      Started:      	Tue, 18 Aug 2026 12:00:24 +0900
    Ready:          	True
    Restart Count:  	0
    Environment:    	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zd69m (ro)
Conditions:
  Type                        	Status
  PodReadyToStartContainers	True
  Initialized                 	True
  Ready                       	True
  ContainersReady             	True
  PodScheduled                	True
Volumes:
  kube-api-access-zd69m:
    Type:                    	Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:      	kube-root-ca.crt
    Optional:                	false
    DownwardAPI:             	true
QoS Class:                   	BestEffort
Node-Selectors:              	<none>
Tolerations:                 	node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             	node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  44m   default-scheduler  Successfully assigned default/rc-nginx-ggjvk to k8s-worker2
  Normal  Pulled     44m   kubelet            spec.containers{nginx-container}: Container image "nginx:1.31" already present on machine and can be accessed by the pod
  Normal  Created    44m   kubelet            spec.containers{nginx-container}: Container created
  Normal  Started    44m   kubelet            spec.containers{nginx-container}: Container started



-지금은 image를 1.31버전을 사용하고 있지만 만약 1.31버전을 사용하다가 최신 버전인
 1.31.3으로 버전을 변경하게되면 pod에는 변화가 없다

-왜냐하면 ReplicationController은 select의 key: value만으로 pod를 관리하기 때문에 버전은 확인하지 않는다.



	# 터미널 2
[root@k8s-master ~]# kubectl  delete  pods  rc-nginx-ggjvk
pod "rc-nginx-ggjvk" deleted from default namespace



	# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME              READY   STATUS    		RESTARTS   AGE   IP             NODE          	NOMINATED NODE   READINESS GATES
rc-nginx-ggjvk   1/1        Running   		0                41m	10.244.2.3	   k8s-worker2   	<none>                    <none>
rc-nginx-nlp42   1/1        Running   		0                41m   	10.244.1.6	   k8s-worker1   	<none>                    <none>
rc-nginx-ggjvk   1/1        Terminating   		0                47m   	10.244.2.3	   k8s-worker2   	<none>                    <none>
rc-nginx-sd45g   0/1        Pending       		0                0s    	<none>  	   <none>        	<none>                    <none>
rc-nginx-ggjvk   1/1        Terminating   		0                47m   	10.244.2.3	   k8s-worker2   	<none>                    <none>
rc-nginx-sd45g   0/1        Pending       		0                0s    	<none> 	   k8s-worker2   	<none>                    <none>
rc-nginx-sd45g   0/1        ContainerCreating	0                0s    	<none> 	   k8s-worker2   	<none>                    <none>
rc-nginx-ggjvk   0/1        Completed           	0                47m   	10.244.2.3	   k8s-worker2   	<none>                    <none>
rc-nginx-sd45g   1/1        Running             	0                3s    	10.244.2.5	   k8s-worker2   	<none>                    <none>


-ReplicationController에의해 새로 만들어진 pod = rc-nginx-sd45g 




	# 이미지가 1.31에서 1.31.3 버전으로 변경 확인
Image size: 63135215 bytes.
[root@k8s-master ~]# kubectl describe pod rc-nginx-sd45g  | grep -i  image
    Image:          	nginx:1.31.3
    Image ID:       	docker.io/library/nginx@sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8
  Normal  Pulling    3m43s  kubelet            spec.containers{nginx-container}: Pulling image "nginx:1.31.3"
  Normal  Pulled     3m42s  kubelet            spec.containers{nginx-container}: Successfully pulled image "nginx:1.31.3" in 1.618s (1.618s including waiting). Image size: 63135215 bytes.



	# 이미지 그대로 사용
[root@k8s-master ~]# kubectl describe pod rc-nginx-nlp42  | grep -i  image
    Image:          	nginx:1.31
    Image ID:       	docker.io/library/nginx@sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8
  Normal  Pulled  	51m   kubelet            spec.containers{nginx-container}: Container image "nginx:1.31" already present on machine and can be accessed by the pod



-ReplicationController의 모든 pod를 삭제할때에는 pod가 아니라 Controller를 삭제해야 한다.

[root@k8s-master ~]# kubectl  delete  rc  rc-nginx
replicationcontroller "rc-nginx" deleted from default namespace


[root@k8s-master ~]# kubectl  get  pods
No resources found in default namespace.
```

### ReplicationController의 한계

- ReplicationController는 개념 이해에는 좋지만 실무에서는 거의 사용하지 않는다.

이유
- Rolling Update 기능이 부족함
- 더 발전된 ReplicaSet / Deployment를 사용

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `template.metadata.labels`가 `selector`와 다르면 RC가 자신이 만든 Pod를 인식하지 못해 Pod를 무한 생성할 수 있다. 두 값이 반드시 일치하는지 항상 확인한다.
- RC의 selector와 같은 라벨을 가진 Pod를 별도로(`kubectl apply`, `kubectl run` 등) 만들면 RC가 초과분으로 인식해 즉시 삭제한다. 관리 대상과 겹치지 않는 라벨 설계가 필요하다.
- `kubectl edit rc` 또는 `kubectl scale rc --replicas=N`으로 Pod 개수를 늘리거나 줄일 수 있으며, 두 방식 모두 동일하게 selector에 걸리는 Pod 개수만 기준으로 동작한다.
- template의 image를 변경해도 이미 존재하는 Pod에는 반영되지 않는다. RC는 selector의 key:value 라벨만으로 Pod를 관리하며 버전(이미지)은 확인하지 않기 때문이다. 새 이미지를 적용하려면 기존 Pod를 삭제해 재생성을 유도해야 한다.
- RC가 관리하는 모든 Pod를 삭제하려면 Pod 개별 삭제가 아니라 `kubectl delete rc <이름>`으로 Controller 자체를 삭제해야 한다.

---

> 📌 **핵심 요약**
> - Pod 생성은 API Server(요청 접수·기록) → Controller(개수 판단) → API Server(Pod 객체 생성) → Scheduler(노드 배치 결정) → API Server(배치 기록) → kubelet(실제 실행) 순서로 진행된다
> - Controller는 Reconciliation Loop(감시 → 비교 → 조정)를 무한 반복하며 원하는 상태(desired state)를 유지하는 관리자다
> - ReplicationController는 replicas(목표 개수)·selector(관리 대상 라벨)·template(생성 설계도)로 구성되며, template.labels는 selector와 반드시 일치해야 한다
> - RC는 라벨 기반 관리이므로 template의 image를 바꿔도 기존 Pod는 그대로이며, 실무에서는 Rolling Update가 부족해 ReplicaSet/Deployment로 대체된다
> - 관련: 11. 🎛️ Kubernetes - ReplicaSet
