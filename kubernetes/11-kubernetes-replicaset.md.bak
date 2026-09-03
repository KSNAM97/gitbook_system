# Kubernetes - ReplicaSet

> **Tag:** #Kubernetes #Controller #ReplicaSet #라벨셀렉터
> **핵심 요약:** ReplicaSet의 개념과 RC와의 차이(set-based selector), matchExpressions(In/NotIn/Exists/DoesNotExist) 문법, selector OR 조건 및 라벨 조작을 통한 편입/제외 실습

---

## 1. 개요 (Overview)

ReplicaSet은 지정한 개수만큼 Pod가 항상 존재하도록 유지하는 컨트롤러다. Pod가 죽거나 삭제되면 자동으로 다시 만들고, Pod가 너무 많으면 줄여서 원하는 개수(replicas)를 맞춘다. 쿠버네티스에서 Pod 개수 보장의 표준 컨트롤러지만, 실무에서는 ReplicaSet을 직접 쓰기보다 Deployment가 ReplicaSet을 만들어서 관리하는 형태가 가장 흔하다.

**ReplicaSet이 필요한 이유**
- Pod는 쉽게 죽을 수 있다 — 컨테이너 오류, 노드 장애, 사람이 `kubectl delete pod` 실행 등으로 Pod는 언제든 사라질 수 있다.
- 서비스는 계속 살아 있어야 한다 — 웹 서버를 3개 띄워 운영 중이라면, 1개가 죽어도 다시 3개로 복구되어야 한다.
- ReplicaSet이 이 역할을 한다 — 항상 Pod 3개 같은 목표를 정해두면, 실제 상태가 바뀌어도 자동으로 원래 상태로 되돌린다.

**ReplicaSet의 핵심 개념 3가지**
- **replicas** — 유지해야 하는 Pod 개수(목표치). 예: `replicas: 3`이면 항상 3개를 유지하려고 한다.
- **selector** — ReplicaSet이 내가 관리할 Pod를 고르는 조건(라벨 조건). selector에 걸리는 Pod를 세어서 replicas에 맞춘다. 매우 중요: selector를 너무 넓게 잡으면 다른 Pod까지 관리 대상으로 잡아버릴 수 있다.
- **template** — Pod가 부족할 때 새로 만들 Pod의 설계도. 어떤 이미지, 포트, 환경변수로 Pod를 만들지 정의한다. `template.metadata.labels`는 selector와 일치해야 한다.
  - 예: `replicas: 3`, `selector: app=webui`라고 했을 때 ReplicaSet은 현재 클러스터에서 label이 `app=webui`인 Pod만 센다.
  - 개수가 3보다 적으면 template으로 Pod를 추가 생성한다.
  - 개수가 3보다 많으면 초과된 Pod를 삭제한다.
  - 이 과정을 계속 반복해서 항상 3개 상태를 유지한다.

**ReplicaSet과 ReplicationController의 차이**
- 기능 목적은 거의 동일하다. (Pod 개수 유지)
- 가장 큰 차이는 selector 표현 범위다.
  - ReplicationController는 보통 equality 기반(`=`) 중심
  - ReplicaSet은 set-based selector(예: `In`, `NotIn`, `Exists` 등)까지 지원해서 더 유연하다
  - 현재 표준은 ReplicaSet이고, 운영에서는 보통 Deployment를 사용한다

---

## 2. ReplicaSet YAML과 matchExpressions

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: rs-webui

spec:
  replicas: 3

  selector:
    matchLabels:
      app: webui
    matchExpressions:
    - key: version    # 검사할 라벨의 key 이름 (version 라벨을 기준으로 판단)
      operator: In    # 조건 연산자: 값이 아래 values 목록에 포함되면 참
      values:
      - "2.1"         # version 라벨 값이 "2.1" 인 Pod만 선택

  template:
    metadata:
      labels:
        app: webui

    spec:
      containers:
      - name: nginx-container
        image: nginx:1.28
        ports:
        - containerPort: 80
```

### operator 종류와 의미

**1) In**
```yaml
- key: version
  operator: In
  values:
  - "2.1"
  - "2.2"
```
- version 라벨이 있고 그 값이 2.1 또는 2.2 중 하나면 참
- 선택되는 Pod: `version=2.1`, `version=2.2`
- 선택 안 됨: `version=1.0`, version 없음

**2) NotIn**
```yaml
- key: env
  operator: NotIn
  values:
  - dev
```
- env 라벨이 있고 값이 dev가 아니면 참
- 선택되는 Pod: `env=prod`, `env=stage`
- 선택 안 됨: `env=dev`, env 없음

**3) Exists**
```yaml
- key: tier
  operator: Exists
```
- tier 라벨이 존재하기만 하면 참 (값은 상관없음)
- 선택되는 Pod: `tier=frontend`, `tier=backend`, `tier=anything`
- 선택 안 됨: tier 라벨 없음

**4) DoesNotExist**
```yaml
- key: debug
  operator: DoesNotExist
```
- debug 라벨이 없으면 참
- 선택되는 Pod: (debug 라벨 없음)
- 선택 안 됨: `debug=true`, `debug=false`

---

## 3. 실습: ReplicaSet 기본 생성과 Scale

```bash
[root@k8s-master ~]# vi  rs-nginx.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-nginx

spec:
  replicas: 3
  selector:
    matchLabels:
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
```

```bash
# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  rs-nginx.yaml  --dry-run=client
replicaset.apps/rs-nginx created (dry run)

# 터미널 1
[root@k8s-master ~]# watch  kubectl  get  pods  -o   wide

# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  rs-nginx.yaml
replicaset.apps/rs-nginx created

# 터미널 1
[root@k8s-master ~]# watch  kubectl  get  pods  -o   wide
Every 2.0s: kubectl get pods -o wide                                                                      k8s-master: Tue Aug 18 13:08:58 2026
NAME             	READY   STATUS    RESTARTS   AGE   IP             NODE          	NOMINATED NODE   READINESS GATES
rs-nginx-845g8	1/1         Running     0                16s    10.244.2.6   k8s-worker2   	<none>                    <none>
rs-nginx-9bkmw   	1/1         Running     0                16s    10.244.1.9   k8s-worker1   	<none>                    <none>
rs-nginx-vjpv7   	1/1         Running     0                16s    10.244.1.8   k8s-worker1   	<none>                    <none>

[root@k8s-master ~]# kubectl  get  replicasets.apps  rs-nginx
NAME       DESIRED   CURRENT   READY   AGE
rs-nginx    3              3               3            71s

[root@k8s-master ~]# kubectl  get  rs  rs-nginx
NAME       DESIRED   CURRENT   READY   AGE
rs-nginx    3              3               3            71s

[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME             	READY   STATUS    RESTARTS   AGE     LABELS
rs-nginx-845g8   	1/1         Running     0                3m22s   app=webui
rs-nginx-9bkmw   	1/1         Running     0                3m22s   app=webui
rs-nginx-vjpv7   	1/1         Running     0                3m22s   app=webui
```

```bash
# ReplicaSet Controller에 의해 생성된 pod 삭제
[root@k8s-master ~]# kubectl  delete  pods  rs-nginx-vjpv7
pod "rs-nginx-vjpv7" deleted from default namespace

[root@k8s-master ~]# kubectl  get  pods
NAME             	READY   STATUS    RESTARTS   AGE
rs-nginx-845g8   	1/1        Running      0                2m35s
rs-nginx-9bkmw   	1/1        Running      0                2m35s
rs-nginx-h9tf5       	1/1        Running      0                42s		# 새로운 pod 생성
```

**Scale-out (pod 확장)**
```bash
[root@k8s-master ~]# kubectl  edit  rs   rs-nginx
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
spec:
  replicas: 3		# 3을 5로 변경
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~

	OR

# 터미널 2
[root@k8s-master ~]# kubectl  scale  rs  rs-nginx --replicas=5
replicaset.apps/rs-nginx scaled
```

**Scale-in (pod 축소)**
```bash
[root@k8s-master ~]# kubectl  edit  rs   rs-nginx
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
spec:
  replicas: 3		# 5를 2로 변경
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~

	OR

# 터미널 2
[root@k8s-master ~]# kubectl  scale  rs  rs-nginx --replicas=2
replicaset.apps/rs-nginx scaled
```

**ReplicaSet Controller 삭제**
```bash
[root@k8s-master ~]# kubectl  delete  rs  rs-nginx
             OR
[root@k8s-master ~]# kubectl  delete  -f  rs-nginx.yaml

# Controller를 삭제하면 해당 Controller가 관리하는 pod도 같이 삭제된다.
[root@k8s-master ~]# kubectl  get  pods
No resources found in default namespace.
```

**Pod는 남기고 Controller만 삭제 (Orphan)**
```bash
[root@k8s-master ~]# kubectl  apply  -f  rs-nginx.yaml
replicaset.apps/rs-nginx created

[root@k8s-master ~]# kubectl  delete  rs  rs-nginx  --cascade=false		# 현재 비 권장 방식
warning: --cascade=false is deprecated (boolean value) and can be replaced with --cascade=orphan.
replicaset.apps "rs-nginx" deleted from default namespace

	~~~~~~~~~~ OR ~~~~~~~~~~

[root@k8s-master ~]# kubectl  delete  rs  rs-nginx  --cascade=orphan		# 현재 권장 방식
replicaset.apps "rs-nginx" deleted from default namespace

[root@k8s-master ~]# kubectl  get  rs			# 컨트롤러가 확인되지 않는다.
No resources found in default namespace

[root@k8s-master ~]# kubectl  get  pods		# Pod는 확인된다.
NAME             	READY   STATUS    RESTARTS   AGE
rs-nginx-dfks9   	1/1         Running     0                5m10s
rs-nginx-hcvv2   	1/1         Running     0                5m10s
rs-nginx-nz7pl   	1/1         Running     0                5m10s
```

---

## 4. EX1) selector 기본 원리 + 포함/제외 + scale 실습

- selector는 "라벨 조건"이다.
- ReplicaSet은 selector에 걸리는 Pod 개수만 유지한다.
- 라벨 변경이 관리 대상 포함/제외를 즉시 변경한다.

```bash
[root@k8s-master ~]# vi rs-lab-a.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-lab-a

spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      env: prod

  template:
    metadata:
      labels:
        app: web
        env: prod
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
```

```bash
[root@k8s-master ~]# kubectl  apply  -f rs-lab-a.yaml
replicaset.apps/rs-lab-a created

[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME             	READY   STATUS    RESTARTS   AGE     LABELS
rs-lab-a-hr6kn   	1/1         Running     0                2m19s   app=web,env=prod,tier=frontend
rs-lab-a-jvz5j   	1/1         Running     0                2m19s   app=web,env=prod,tier=frontend
rs-lab-a-m8v2j   	1/1         Running     0                2m19s   app=web,env=prod,tier=frontend
```

### 1) Pod 하나를 selector에서 제외시키기

EX1-1) rs-lab-a가 만든 Pod 중 하나의 env를 prod에서 dev 로 변경

```bash
# rs-lab-a-hr6kn pod의 label을 변경
[root@k8s-master ~]# kubectl  label  pod  rs-lab-a-hr6kn env=dev  --overwrite

[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME             	READY   STATUS    RESTARTS   AGE     LABELS
rs-lab-a-hr6kn   	1/1         Running     0                6m34s   app=web,env=dev,tier=frontend
rs-lab-a-jvz5j   	1/1         Running     0                6m34s   app=web,env=prod,tier=frontend
rs-lab-a-m8v2j   	1/1         Running     0                6m34s   app=web,env=prod,tier=frontend
rs-lab-a-wlj8h   	1/1         Running     0                66s       app=web,env=prod,tier=frontend		# 새로운 pod 생성

[root@k8s-master ~]# kubectl  get  rs rs-lab-a
NAME       DESIRED   CURRENT   READY   AGE
rs-lab-a     3             3                3           8m16s
```
`env=dev`로 바뀐 Pod는 selector(`app=web, env=prod`)에서 벗어나므로 ReplicaSet은 여전히 자신이 관리하는 것으로 세지 않고, 대신 부족해진 1개를 새로 생성한다.

**Scale-out 반응**
```bash
[root@k8s-master ~]# kubectl  scale  rs  rs-lab-a  --replicas=5
replicaset.apps/rs-lab-a scaled

[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME             	READY   STATUS    RESTARTS   AGE     LABELS
rs-lab-a-7xwmd   	1/1     Running   0          43s      app=web,env=prod,tier=frontend
rs-lab-a-hr6kn   	1/1     Running   0          11m     app=web,env=dev,tier=frontend	# label match 안 됨(유지되지만 count 제외)
rs-lab-a-jvz5j   	1/1     Running   0          11m     app=web,env=prod,tier=frontend	# label match
rs-lab-a-m8v2j   	1/1     Running   0          11m     app=web,env=prod,tier=frontend	# label match
rs-lab-a-n8ztl   	1/1     Running   0          43s      app=web,env=prod,tier=frontend	# label match
rs-lab-a-wlj8h   	1/1     Running   0          5m39s   app=web,env=prod,tier=frontend	# label match
```

### 3) 수동 Pod를 selector에 끼워 넣기

EX1-3) 수동 Pod를 하나 만들되, ReplicaSet Controller `rs-lab-a` selector에 매치되어야 함

```bash
[root@k8s-master ~]# kubectl  run  maunal-pod  --image=nginx:1.31  --labels app=web,env=prod,tier=backend
pod/maunal-pod created

# tier=backend 라벨이 확인되지 않는다. (ReplicaSet Controller에 의해 삭제)
[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME             	READY   STATUS    RESTARTS   AGE     LABELS
rs-lab-a-7xwmd   	1/1     Running   0          43s      app=web,env=prod,tier=frontend
...(생략)
```
ReplicaSet Controller에 의해 만들어진 pod가 아니어도 label만 매치되면 관리대상으로 포함되며, 이미 원하는 개수(replicas)를 채우고 있으면 초과분은 자동으로 삭제된다.

### 4) ownerReferences로 RS Pod/수동 Pod 구분

`ownerReferences`는 이 리소스는 누가 만든 자식 객체인지 쿠버네티스가 내부적으로 관리하기 위한 연결 정보로, 부모-자식 관계를 명확하게 기록해 두는 메타데이터다. 단독으로 실행되는 pod에는 `ownerReferences`가 없다.

```bash
[root@k8s-master ~]# kubectl  get  pod  rs-lab-a-7xwmd  -o  yaml | nl
     1  apiVersion: v1
     2  kind: Pod
     3  metadata:
     4    creationTimestamp: "2026-08-18T06:10:36Z"
     5    generateName: rs-lab-a-
     6    generation: 1
     7    labels:
     8      app: web
     9      env: prod
    10      tier: frontend
    11    name: rs-lab-a-7xwmd
    12    namespace: default
    13    ownerReferences:
    14    - apiVersion: apps/v1
    15      blockOwnerDeletion: true
    16      controller: true
    17      kind: ReplicaSet
    18      name: rs-lab-a
    19      uid: f127b14a-eb5a-40bc-80af-bf900ff43f2e
    20    resourceVersion: "183395"
    21    uid: 054c33cb-c1f6-4f2d-b83c-d018befb114f
    22  spec:
    23    containers:
    24    - image: nginx:1.31
    25      imagePullPolicy: IfNotPresent
    26      name: nginx
~~~~~~~~~ 중간 생략 ~~~~~~~~~
```

---

## 5. EX2) selector OR 조건 실습 (matchExpressions In)

쿠버네티스 라벨 셀렉터에는 OR 문법이 없지만, `matchExpressions(In)` 하나로 OR 조건을 구성할 수 있다. 수동 Pod 투입, 라벨 변경 편입/제외, scale 반응, 실수 상황을 YAML 1개로 실습한다.

- 쿠버네티스 라벨 셀렉터에 OR 문법이 없고 `matchExpressions(In)`으로 OR 문법을 대체함을 확인
- 라벨 변경으로 관리 대상이 실시간으로 편입/제외됨을 확인
- scale out/in이 selector에 걸리는 Pod만 기준으로 동작함을 확인

```yaml
[root@k8s-master ~]# vi rs-lab-a-or.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-lab-a-or

spec:
  replicas: 2
  selector:
    matchExpressions:
    - key: env
      operator: In
      values:
      - prod
      - stage

  template:
    metadata:
      labels:
        app: web
        env: prod
        lab: rs-lab-a-or
    spec:
      containers:
      - name: nginx
        image: nginx:1.31

# env=prod OR env=stage
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  rs-lab-a-or.yaml
replicaset.apps/rs-lab-a-or created

[root@k8s-master ~]# kubectl  get  rs  rs-lab-a-or
NAME          DESIRED   CURRENT   READY   AGE
rs-lab-a-or   2               2               2          13s

[root@k8s-master ~]# kubectl  describe  rs  rs-lab-a-or
Name:         	rs-lab-a-or
Namespace:    	default
Selector:     	env in (prod,stage)
Labels:       	<none>
Annotations:  	<none>
Replicas:     	2 current / 2 desired
Pods Status:  	2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=web
             env=prod
             lab=rs-lab-a-or
  Containers:
   nginx:
    Image:         	nginx:1.31
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  3m20s  replicaset-controller  Created pod: rs-lab-a-or-56l95
  Normal  SuccessfulCreate  3m20s  replicaset-controller  Created pod: rs-lab-a-or-bsr78

[root@k8s-master ~]# kubectl  get  rs  rs-lab-a-or  -o  wide
NAME          DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES       SELECTOR
rs-lab-a-or    2              2               2           5m32s   nginx              nginx:1.31     env in (prod,stage)
```

### STEP 2. 수동 Pod 5개를 서로 다른 env로 생성 (분류 실습)

```bash
[root@k8s-master ~]# kubectl  run  p-prod  --image=nginx:1.31  --labels env=prod,lab=rs-lab-a-or
# env=prod는 selector(env in prod,stage)에 해당 → 관리 대상. replicas 초과 시 자동 축소됨.

[root@k8s-master ~]# kubectl  run  p-stage  --image=nginx:1.31  --labels env=stage,lab=rs-lab-a-or
# env=stage도 selector에 해당 → 관리 대상. 초과분 자동 삭제.

[root@k8s-master ~]# kubectl  run  p-dev  --image=nginx:1.31  --labels env=dev,lab=rs-lab-a-or
# env=dev는 selector 조건(prod, stage)에 해당하지 않는다. → 관리 대상 아님. 독립적으로 계속 실행.

[root@k8s-master ~]# kubectl  run  p-test  --image=nginx:1.31  --labels env=test,lab=rs-lab-a-or
# env=test 역시 조건에 해당하지 않는다 → 관리 대상 아님.

[root@k8s-master ~]# kubectl  run  p-none  --image=nginx:1.31  --labels  lab=rs-lab-a-or
# env 라벨 자체가 없으므로 selector 조건에 필요한 key가 없다 → 관리 대상 아님.

[root@k8s-master ~]# kubectl get  pods  --show-labels
NAME                	READY   STATUS    RESTARTS   AGE     LABELS
p-dev               	1/1        Running     0                 4m44s   env=dev,lab=rs-lab-a-or
p-none              	1/1        Running     0                 109s     lab=rs-lab-a-or
p-test              	1/1        Running     0                 4m33s   env=test,lab=rs-lab-a-or
rs-lab-a-or-56l95   	1/1        Running     0                 21m      app=web,env=prod,lab=rs-lab-a-or
rs-lab-a-or-bsr78	1/1        Running     0                 21m      app=web,env=prod,lab=rs-lab-a-o
```

### STEP 3. 제외된 Pod를 라벨 변경으로 편입시키기

EX) p-dev를 관리 대상에 편입시켜라(env를 stage 또는 prod로 변경)

```bash
[root@k8s-master ~]# kubectl  label  pod  p-dev  env=stage  --overwrite
pod/p-dev labeled

[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
p-dev               	1/1     Terminating   0          7m28s   ...
p-dev               	0/1     Completed     0          7m29s   ...

# p-dev가 확인되지 않는다.
[root@k8s-master ~]# kubectl get pods --show-labels
NAME                	READY   STATUS    RESTARTS   AGE     LABELS
p-none              	1/1     Running   0          6m10s   lab=rs-lab-a-or
p-test              	1/1     Running   0          8m54s   env=test,lab=rs-lab-a-or
rs-lab-a-or-56l95	1/1     Running   0          25m     app=web,env=prod,lab=rs-lab-a-or
rs-lab-a-or-bsr78	1/1     Running   0          25m     app=web,env=prod,lab=rs-lab-a-or
```
라벨을 변경하면 selector에 매치되어 관리 대상이 되고, 이미 replicas 개수를 채우고 있었으므로 편입된 p-dev는 곧바로 종료(Completed)된다.

### STEP 4. 포함된 Pod를 라벨 변경으로 제외시키기

EX) p-prod를 관리 대상에서 제외시켜라(정답: env를 dev 등으로 바꿈)

```bash
[root@k8s-master ~]# kubectl  label  pod  rs-lab-a-or-56l95  env=test  --overwrite
pod/rs-lab-a-or-56l95 labeled

[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
rs-lab-a-or-rwm5l	0/1     Pending   0          0s    <none>        <none>        <none>           <none>
rs-lab-a-or-rwm5l	1/1     Running             0          0s    10.244.2.14   k8s-worker2   <none>           <none>

[root@k8s-master ~]# kubectl  get  rs  rs-lab-a-or
NAME          DESIRED   CURRENT   READY   AGE
rs-lab-a-or   2               2               2          32m
```
라벨이 바뀐 Pod는 selector 밖으로 벗어나 관리 대상에서 제외되고(단, Pod 자체는 삭제되지 않고 계속 실행됨), ReplicaSet은 부족해진 1개를 새로 생성해 replicas 개수를 다시 채운다.

---

## 6. 실습 B) selector 사고(너무 넓음/너무 좁음) 재현과 복구

ReplicaSet selector를 너무 넓게/너무 좁게 설계했을 때 생기는 대표 사고를 YAML 1개로 만든 뒤 라벨 조작만으로 재현하고 원인 설명과 복구까지 단계별로 수행한다.

- selector가 너무 넓으면 다른 용도의 Pod까지 관리 대상으로 보일 수 있다.
- selector와 template.labels가 불일치하거나 selector가 너무 좁으면 Pod가 생성되지 않는 문제가 발생한다.

```bash
# namespace 생성 후 기본 namespace로 전환
[root@k8s-master ~]# kubectl create namespace selector-lab-b
[root@k8s-master ~]# kubectl config  set-context  --current  --namespace=selector-lab-b
# kubectl config set-context	: kubectl Context 설정 변경
# --current : 현재 사용 중인 Context를 수정
# --namespace=selector-lab-b	: 기본 Namespace를 selector-lab-b로 설정
```

```yaml
[root@k8s-master ~]# vi rs-lab-b.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-b

spec:
  replicas: 2
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web
        team: a
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  rs-lab-b.yaml
replicaset.apps/rs-b created
```

### STEP 2. ReplicaSet에 포함되지 않는 다른 용도의 Pod로 selector 사고 재현

```bash
[root@k8s-master ~]# kubectl  run  otherteam  --image=nginx:1.31  --labels app=web,team=b

[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME         	READY   STATUS    RESTARTS   AGE   LABELS
rs-b-6p6rl   	1/1        Running   0          3m5s   app=web,team=a
rs-b-fhcwp   	1/1        Running   0          3m5s   app=web,team=a
otherteam    	1/1        Running   0          2s     app=web,team=b

[root@k8s-master ~]# kubectl  get  rs  rs-b
NAME   DESIRED   CURRENT   READY   AGE
rs-b      2              2              2            8m44s
```
`otherteam` Pod는 ReplicaSet Controller `rs-b`가 만든 Pod가 아니다. 하지만 `app=web` 라벨 때문에 selector(`app=web`)에 매치되기 때문에 라벨 충돌 사고의 출발점이다. `rs-b`의 selector가 `app=web` 하나뿐이라 위험하다.

### STEP 3. selector를 안전한 라벨 설계로 개선

실행(교체 방식: 기존 RS 삭제 후 새 RS 생성) — selector를 `app=web AND team=a`로 바꾼 안전한 RS로 교체한다.

```bash
[root@k8s-master ~]# kubectl  delete  rs  rs-b
```

```yaml
[root@k8s-master ~]# vi  rs-lab-b.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-b

spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      team: a	# label 추가

  template:
    metadata:
      labels:
        app: web
        team: a
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  rs-lab-b.yaml
replicaset.apps/rs-b created

# RS가 관리대상으로 포함할 label 확인
[root@k8s-master ~]# kubectl  get  rs  rs-b  -o  wide
NAME   DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES       SELECTOR
rs-b       2             2               2           111s     nginx              nginx:1.31    app=web,team=a

# 수동으로 team=b Pod 생성 → 이번에는 selector(app=web,team=a)에 걸리지 않아 관리 대상 밖
[root@k8s-master ~]# kubectl  run  otherteam  --image=nginx:1.31  --labels app=web,team=b

[root@k8s-master ~]# kubectl  delete  rs  rs-b
[root@k8s-master ~]# kubectl  delete  pods  otherteam
[root@k8s-master ~]# kubectl config  set-context  --current  --namespace=default
```

selector를 여러 라벨의 AND 조건(`matchLabels`에 키를 추가)으로 좁히면, 우연히 같은 라벨 하나를 공유하는 다른 팀의 Pod가 관리 대상에 잘못 포함되는 사고를 막을 수 있다.

---

## 7. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- selector가 template.labels와 불일치하면 ReplicaSet은 Pod를 생성하지 못하고 `kubectl describe rs`의 Events에 에러가 남는다. selector와 template.labels는 항상 일치해야 한다.
- `kubectl get rs <name> -o wide`의 SELECTOR 컬럼으로 현재 관리 대상 조건을 바로 확인할 수 있다.
- `ownerReferences`가 있는 Pod는 ReplicaSet이 생성한 Pod, 없는 Pod는 수동으로 생성된 독립 Pod다. `kubectl get pod <name> -o yaml`로 확인한다.
- 라벨을 바꿔 Pod를 selector 밖으로 빼도 Pod 자체는 삭제되지 않고 계속 실행된다 — 단지 ReplicaSet의 관리(개수 카운트) 대상에서 제외될 뿐이다.
- `kubectl delete rs <name> --cascade=orphan`으로 Controller만 지우고 Pod는 살려둘 수 있다(`--cascade=false`는 deprecated).

---

>  **핵심 요약**
> - ReplicaSet은 replicas·selector·template 3요소로 Pod 개수를 항상 목표치에 맞추며, RC보다 유연한 set-based selector(`matchExpressions`: In/NotIn/Exists/DoesNotExist)를 지원한다
> - selector는 라벨 조건일 뿐이므로, ReplicaSet이 만들지 않은 Pod라도 라벨만 일치하면 관리 대상에 포함되고 초과분은 자동 삭제된다 — selector를 너무 넓게 설계하면 사고로 이어진다
> - 라벨을 변경하는 것만으로 실시간으로 관리 대상 편입/제외가 가능하며, 실무에서는 ReplicaSet을 직접 쓰기보다 Deployment가 이를 생성·관리하는 형태가 표준이다
> - 관련: 10.  Kubernetes - Controller 개념과 ReplicationController · 12.  Kubernetes - Deployment · 24.  Kubernetes - Label
