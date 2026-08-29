# ☸️ Kubernetes - DaemonSet

> **Tag:** #Kubernetes #DaemonSet #Controller #부트캠프
> **핵심 요약:** 클러스터의 각 노드마다 특정 Pod를 1개씩 자동으로 유지하는 컨트롤러 DaemonSet의 개념과 노드 선택, 롤링 업데이트/롤백 실습 정리

---

## 1. 📖 개요 (Overview)

DaemonSet은 클러스터의 각 Node마다 특정 Pod를 1개씩(또는 조건에 맞는 노드마다) 자동으로 실행해 주는 컨트롤러이다. 즉, Deployment가 서비스용 Pod를 n개 유지라면, DaemonSet은 노드마다 1개씩 유지한다.

**DaemonSet이 필요한 이유**

모든 노드에서 반드시 돌아야 하는 공통 기능 Pod를 자동 배치/유지한다.

예)
- 로그 수집 에이전트 (각 노드의 로그를 모아서 중앙으로 전송)
- 모니터링 에이전트 (노드 CPU/메모리/디스크/네트워크 수집)
- 네트워크 플러그인 구성 요소 (CNI 관련)
- 노드 보안/감사 에이전트
- 스토리지/CSI 노드 에이전트(환경에 따라)

DaemonSet은 특정 서비스 트래픽을 처리하는 앱이 아니라, 노드 자체에서 데이터를 가져오거나 시스템 기능을 제공해야 하므로 노드마다 1개씩 있어야 한다.

**DaemonSet의 동작 원리**

DaemonSet을 생성하면, 컨트롤러가 다음을 자동으로 한다.

1. 현재 클러스터의 노드 목록을 확인한다.
2. 각 노드마다 DaemonSet용 Pod가 있는지 확인한다.
3. 없으면 그 노드에 DaemonSet용 Pod를 만든다.
4. DaemonSet용 Pod가 죽으면 그 노드에서 다시 만든다.
5. 노드가 새로 추가되면 그 노드에도 자동으로 DaemonSet용 Pod를 만든다.
6. 노드가 삭제되면 그 노드의 DaemonSet용 Pod도 함께 사라진다.

결론적으로 노드 수가 늘면 DaemonSet용 Pod도 같이 늘고 노드 수가 줄면 DaemonSet용 Pod도 같이 줄어든다. 예를 들어 워커 노드가 2개면 DaemonSet용 Pod 2개, 5개면 DaemonSet용 Pod 5개가 생성된다(조건에 맞는 노드만).

**Deployment와 DaemonSet 차이**

| 구분 | Deployment | DaemonSet |
|---|---|---|
| 목적 | 서비스 트래픽 처리용 Pod를 원하는 개수만큼 유지 | 노드마다 반드시 필요한 공통 Pod를 유지 |
| 스케줄링 | 어떤 노드에 올라갈지는 쿠버네티스가 분산 배치 | 각 노드에 1개씩이 원칙 |
| replicas | 사용자가 개수를 지정한다 (예: 3개, 10개) | 사용자가 숫자를 지정하는 개념이 아니다 (노드 수/조건에 의해 자동 결정) |

- Deployment = 서비스 개수 보장
- DaemonSet = 노드당 1개 보장

**모든 노드가 아니라 특정 노드만 돌리고 싶을 때**

DaemonSet은 기본이 모든 노드(정확히는 스케줄 가능한 노드)이지만, 필요하면 조건을 걸어서 일부 노드에만 배치할 수 있다.

대표 방법
- **nodeSelector** — 라벨이 붙은 노드에만 배치
- **nodeAffinity** — 더 복잡한 조건(OR/AND 등)으로 노드 선택
- **taints/tolerations** — 특정 노드(예: control-plane)에 원래는 안 올라가게 막혀있는데 toleration을 넣으면 올라가게 할 수 있다

예시 상황
- GPU 노드에만 에이전트 설치
- 특정 역할(role=log-node) 라벨이 있는 노드에만 로그 수집기 실행
- control-plane 노드에도 반드시 실행해야 하는 시스템 에이전트 배치

**DaemonSet 업데이트(롤링업데이트)**

DaemonSet도 이미지 버전을 바꾸면 Pod가 노드별로 순차 교체될 수 있다. 즉, 각 노드에 있는 Pod를 새 버전으로 하나씩 바꿔치기한다.

다만 DaemonSet은 노드당 1개가 중요해서, 업데이트 중에도 각 노드에서 에이전트가 완전히 사라지는 시간을 최소화하려고 전략을 사용한다. 설정에 따라 동시에 바꾸는 노드 수를 제한할 수 있다(업데이트 전략).

Deployment는 replicas 기반으로 바꾸고, DaemonSet은 노드 단위로 바뀐다.

**DaemonSet에서 자주 보게 되는 특징/필드들**

1. **selector / template** — 어떤 Pod가 DaemonSet 소속인지 구분하기 위한 라벨 매칭. template은 실제로 노드에 생성될 Pod 모양(컨테이너/이미지/환경변수 등)
2. **nodeSelector / affinity** — 어느 노드에 깔지를 결정
3. **tolerations** — 특정 taint가 있는 노드에도 배치할지 결정. 대표적으로 control-plane 노드(마스터 노드)에 올릴 때 필요할 수 있다
4. **hostPath (에이전트에서 자주 사용)** — 노드의 파일 시스템을 Pod에 마운트해서 로그/메트릭 파일 등을 읽음. 예: `/var/log`를 마운트해서 노드 로그 수집

**언제 DaemonSet을 쓰면 안 되나?**

서비스 애플리케이션(웹 서버, API 서버)을 각 노드에 1개씩 깔 필요가 없다면 Deployment/StatefulSet이 보통 맞다.

예)
- 쇼핑몰 웹 서버: DaemonSet로 깔면 노드마다 1개씩 강제라서 운영이 불편해질 수 있음
- 트래픽에 따라 3개에서 10개로 늘리고 싶은 서비스: Deployment가 맞다
- DaemonSet은 노드 공통 기능에 맞는 도구다.

---

## 2. 🛠️ 실습

### DaemonSet YAML 예제

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-nginx

spec:
  replicas: 3		# ReplicaSet에서 replicas를 제외하면 형식은 동일하다. (DeamonSet은 기본 1개로 설정)
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      name: nginx-pod
      labels:
        app: web
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31
```

### DaemonSet 관련 주요 명령어

```
# DaemonSet 생성 및 확인
# kubectl create -f daemonset-exam.yaml
# kubectl get daemonset
# kubectl get pods

# DaemonSet으로 동작 중인 Pod를 삭제
# kubectl delete pod daemonset-nginx-XXX

# DaemonSet Rolling Update (컨테이너 버전 수정)
# kubectl edit ds daemonset-nginx

# Rolling Back (롤백)
# kubectl rollout undo daemonset daemonset-nginx

# DaemonSet 삭제
# kubectl delete daemonsets.apps daemonset-nginx

# 또는 (축약형)
# kubectl delete ds daemonset-nginx
```

### DaemonSet 생성 및 노드별 Pod 자동 배치 확인

```
[root@k8s-master ~]# vi daemonset-nginx.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-nginx
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      name: nginx-pod
      labels:
        app: web
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31


[root@k8s-master ~]# kubectl  create  -f  daemonset-nginx.yaml  --dry-run=client
daemonset.apps/daemonset-nginx created (dry run)



[root@k8s-master ~]# kubectl  create  -f  daemonset-nginx.yaml
daemonset.apps/daemonset-nginx created



[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                    	READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
daemonset-nginx-k4bqk   	1/1        Runnin  g   0                 12s    10.244.1.54   k8s-worker1   <none>                   <none>
daemonset-nginx-xc4mg   	1/1        Running     0                 12s    10.244.2.37   k8s-worker2   <none>                   <none>
```

이 상태에서 워커노드 1대가 추가되면 해당 워커 노드에도 해당 pod가 자동으로 생성된다.

### DaemonSet Rolling Update (이미지 버전 수정)

```
# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch



# 터미널 2
[root@k8s-master ~]# kubectl  edit  daemonsets  daemonset-nginx
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2026-08-19T07:01:16Z"
  generation: 1
  name: daemonset-nginx
  namespace: default
  resourceVersion: "238778"
  uid: 87489b5f-b630-4780-9f27-101ad790acc6

spec:
  revisionHistoryLimit: 10		# revision을 10개까지 보관
  selector:
    matchLabels:
      app: web			# app: web 라벨을 갖은 pod를 관리 대상으로 인식
  template:
    metadata:
      labels:
        app: web			# 해당 pod 생성시 app: web 라벨을 사용
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.31		# 해당 pod가 사용할 이미지 (image: nginx:1.31  -->  image: nginx:1.31.1)
        imagePullPolicy: IfNotPresent
        name: nginx-container
        resources: {}

~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~



[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                    	READY   	STATUS    	RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
daemonset-nginx-k4bqk   	1/1    	Running   	0                2m53s	  10.244.1.54   k8s-worker1   <none>           <none>
daemonset-nginx-xc4mg   	1/1      	Running   	0                2m53s	  10.244.2.37   k8s-worker2   <none>           <none>

daemonset-nginx-xc4mg   	1/1     	Terminating   	0                9m31s	  10.244.2.37   k8s-worker2   <none>           <none>
daemonset-nginx-xc4mg   	1/1     	Terminating   	0                9m31s	  10.244.2.37   k8s-worker2   <none>           <none>
daemonset-nginx-xc4mg   	0/1     	Completed     	0                9m31s	  10.244.2.37   k8s-worker2   <none>           <none>
daemonset-nginx-4zkth   	0/1     	Pending       	0                0s      	  <none>        <none>         <none>           <none>
daemonset-nginx-4zkth   	0/1     	Pending       	0                0s      	  <none>        k8s-worker2   <none>           <none>
daemonset-nginx-4zkth   	0/1     	ContainerCreating   0                0s      	  <none>        k8s-worker2   <none>           <none>
daemonset-nginx-xc4mg   	0/1     	Completed           	0                9m32s 	  10.244.2.37   k8s-worker2   <none>           <none>
daemonset-nginx-xc4mg   	0/1     	Completed           	0                9m32s	  10.244.2.37   k8s-worker2   <none>           <none>
daemonset-nginx-4zkth   	1/1     	Running             	0                2s   	  10.244.2.38   k8s-worker2   <none>           <none>
daemonset-nginx-k4bqk   	1/1     	Terminating         	0                9m33s   10.244.1.54   k8s-worker1   <none>           <none>
daemonset-nginx-k4bqk   	1/1     	Terminating         	0                9m33s   10.244.1.54   k8s-worker1   <none>           <none>
daemonset-nginx-k4bqk   	0/1     	Completed           	0                9m33s   10.244.1.54   k8s-worker1   <none>           <none>
daemonset-nginx-qsft7   	0/1     	Pending             	0                0s      	  <none>        <none>        <none>           <none>
daemonset-nginx-qsft7   	0/1     	Pending             	0                0s      	  <none>        k8s-worker1   <none>           <none>
daemonset-nginx-qsft7   	0/1     	ContainerCreating	0                0s      	  <none>        k8s-worker1   <none>           <none>
daemonset-nginx-k4bqk   	0/1     	Completed           	0                9m33s	  10.244.1.54   k8s-worker1   <none>           <none>
daemonset-nginx-k4bqk   	0/1     	Completed           	0                9m33s	  10.244.1.54   k8s-worker1   <none>           <none>
daemonset-nginx-qsft7   	1/1     	Running             	0                1s   	  10.244.1.55   k8s-worker1   <none>           <none>



[root@k8s-master ~]# kubectl  describe  daemonsets   daemonset-nginx | grep Image
    Image:         nginx:1.31.1
```

### 롤아웃 히스토리와 ControllerRevision

```
[root@k8s-master ~]# kubectl  rollout  history  daemonset daemonset-nginx
daemonset.apps/daemonset-nginx
REVISION  CHANGE-CAUSE
1              <none>
2              <none>


kubectl annotate daemonset daemonset-nginx \
kubernetes.io/change-cause="nginx 1.31 -> 1.31.1 업데이트" \
--overwrite



# CHANGE-CAUSE 기록이 확인되지 않는다.
[root@k8s-master ~]# kubectl  rollout  history  daemonset daemonset-nginx
daemonset.apps/daemonset-nginx
REVISION  CHANGE-CAUSE
1              <none>
2              <none>
```

DaemonSet은 Deployment처럼 ReplicaSet으로 이력을 관리하지 않고, ControllerRevision을 사용하여 Revision 이력을 관리한다.

ControllerRevision은 DaemonSet의 이전 Pod Template 정보를 저장하는 이력용 리소스이다.

### Daemonset 삭제 후 재생성, change-cause 설정

```
# Daemonset 삭제

[root@k8s-master ~]# kubectl  delete  daemonsets  daemonset-nginx
daemonset.apps "daemonset-nginx" deleted from default namespace



# Daemonset 생성
[root@k8s-master ~]# kubectl  apply  -f  daemonset-nginx.yaml



# change-cause 설정
[root@k8s-master ~]# kubectl annotate daemonset daemonset-nginx \
kubernetes.io/change-cause="nginx 1.31 --> 1.31.1 업데이트" \
--overwrite
daemonset.apps/daemonset-nginx annotated



# 터미널 2
[root@k8s-master ~]# kubectl  edit  daemonsets  daemonset-nginx
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2026-08-19T07:01:16Z"
  generation: 1
  name: daemonset-nginx
  namespace: default
  resourceVersion: "238778"
  uid: 87489b5f-b630-4780-9f27-101ad790acc6

spec:
  revisionHistoryLimit: 10		# revision을 10개까지 보관
  selector:
    matchLabels:
      app: web			# app: web 라벨을 갖은 pod를 관리 대상으로 인식
  template:
    metadata:
      labels:
        app: web			# 해당 pod 생성시 app: web 라벨을 사용
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.31		# 해당 pod가 사용할 이미지 (image: nginx:1.31  -->  image: nginx:1.31.1)
        imagePullPolicy: IfNotPresent
        name: nginx-container
        resources: {}

~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~



[root@k8s-master ~]# kubectl  rollout  history  daemonset daemonset-nginx
daemonset.apps/daemonset-nginx
REVISION  CHANGE-CAUSE
1             <none>
2             nginx 1.31 --> 1.31.1 업데이트
```

첫번째 REVISION은 daemonset이 없는 상태에서 daemonset 버전을 설정할 수 없기때문에 버전관리를 할수 없다.

### YAML에 change-cause를 미리 지정해서 버전 관리하기

```
# Daemonset 삭제

[root@k8s-master ~]# kubectl  delete  daemonsets  daemonset-nginx
daemonset.apps "daemonset-nginx" deleted from default namespace



[root@k8s-master ~]# vi  daemonset-nginx.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-nginx
  annotations:					# 추가 설정
    kubernetes.io/change-cause: "nginx 1.31 첫 배포"	# 추가 설정 (yaml 파일에 버전 관련 설정을 미리 설정)
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      name: nginx-pod
      labels:
        app: web
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31



# Daemonset 생성
[root@k8s-master ~]# kubectl  apply  -f  daemonset-nginx.yaml



# 버전 관리 확인
[root@k8s-master ~]# kubectl  rollout  history  daemonset daemonset-nginx
daemonset.apps/daemonset-nginx
REVISION  CHANGE-CAUSE
1              nginx 1.31 첫 배포


# 버전 관리 설정
[root@k8s-master ~]# kubectl annotate daemonset daemonset-nginx \
kubernetes.io/change-cause="nginx 1.31 --> 1.31.1 업데이트" --overwrite
daemonset.apps/daemonset-nginx annotated



# 터미널 2
[root@k8s-master ~]# kubectl  edit  daemonsets  daemonset-nginx
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2026-08-19T07:01:16Z"
  generation: 1
  name: daemonset-nginx
  namespace: default
  resourceVersion: "238778"
  uid: 87489b5f-b630-4780-9f27-101ad790acc6

spec:
  revisionHistoryLimit: 10		# revision을 10개까지 보관
  selector:
    matchLabels:
      app: web			# app: web 라벨을 갖은 pod를 관리 대상으로 인식
  template:
    metadata:
      labels:
        app: web			# 해당 pod 생성시 app: web 라벨을 사용
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.31		# 해당 pod가 사용할 이미지 (image: nginx:1.31  -->  image: nginx:1.31.1)
        imagePullPolicy: IfNotPresent
        name: nginx-container
        resources: {}

~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~



# 버전 관리 확인
[root@k8s-master ~]# kubectl  rollout  history  daemonset daemonset-nginx            
daemonset.apps/daemonset-nginx
REVISION  CHANGE-CAUSE
1              nginx 1.31 첫 배포
2              nginx 1.31 --> 1.31.1 업데이트



# 현재 사용하는 이미지 확인
[root@k8s-master ~]# kubectl  describe  daemonsets  daemonset-nginx  | grep Image
    Image:         nginx:1.31.1
```

### Rollback 확인

```
# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                    	READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
daemonset-nginx-79mdq   	1/1         Running     0                9m36s   10.244.2.45   k8s-worker2   <none>                   <none>
daemonset-nginx-7z8sj   	1/1         Running     0                9m38s   10.244.1.62   k8s-worker1   <none>                   <none>



# 터미널 2  (이전 버전의 이미지로 롤백)
[root@k8s-master ~]# kubectl  rollout  undo  daemonset  daemonset-nginx



# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                    	READY  	STATUS    	RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
daemonset-nginx-79mdq   	1/1     	Running   	0                9m36s	  10.244.2.45   k8s-worker2   <none>           <none>
daemonset-nginx-7z8sj   	1/1     	Running   	0                9m38s   10.244.1.62   k8s-worker1   <none>           <none>
daemonset-nginx-79mdq   	1/1     	Terminating   	0                11m     10.244.2.45   k8s-worker2   <none>           <none>
daemonset-nginx-79mdq   	1/1     	Terminating   	0                11m     10.244.2.45   k8s-worker2   <none>           <none>
daemonset-nginx-79mdq   	0/1     	Completed     	0                11m     10.244.2.45   k8s-worker2   <none>           <none>
daemonset-nginx-mhfld   	0/1     	Pending       	0                0s      <none>        <none>        <none>           <none>
daemonset-nginx-mhfld   	0/1    	Pending       	0                0s      <none>        k8s-worker2   <none>           <none>
daemonset-nginx-mhfld   	0/1     	ContainerCreating	0                0s      <none>        k8s-worker2   <none>           <none>
daemonset-nginx-mhfld   	1/1     	Running             	0                0s      10.244.2.46   k8s-worker2   <none>           <none>
daemonset-nginx-7z8sj   	1/1     	Terminating         	0                11m     10.244.1.62   k8s-worker1   <none>           <none>
daemonset-nginx-7z8sj   	1/1     	Terminating         	0                11m     10.244.1.62   k8s-worker1   <none>           <none>
daemonset-nginx-79mdq   	0/1     	Completed           	0                11m     10.244.2.45   k8s-worker2   <none>           <none>
daemonset-nginx-79mdq   	0/1     	Completed           	0                11m     10.244.2.45   k8s-worker2   <none>           <none>
daemonset-nginx-7z8sj   	0/1     	Completed           	0                11m     10.244.1.62   k8s-worker1   <none>           <none>
daemonset-nginx-psrvx   	0/1     	Pending             	0                0s      <none>        <none>        <none>           <none>
daemonset-nginx-psrvx   	0/1     	Pending             	0                0s      <none>        k8s-worker1   <none>           <none>
daemonset-nginx-psrvx   	0/1     	ContainerCreating 	0                0s      <none>        k8s-worker1   <none>           <none>
daemonset-nginx-7z8sj   	0/1     	Completed           	0                11m     10.244.1.62   k8s-worker1   <none>           <none>
daemonset-nginx-7z8sj   	0/1     	Completed           	0                11m     10.244.1.62   k8s-worker1   <none>           <none>
daemonset-nginx-psrvx   	1/1     	Running             	0                1s      10.244.1.63   k8s-worker1   <none>           <none>


# nginx:1.31.1 에서 nginx:1.31로 롤백 확인
[root@k8s-master ~]# kubectl  describe  daemonsets  daemonset-nginx  | grep Image
    Image:         nginx:1.31
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- DaemonSet의 Pod 이름은 Deployment처럼 ReplicaSet 해시가 붙는 대신 노드별로 각각 생성되며, `kubectl get pods -o wide`로 NODE 컬럼을 확인하면 각 워커 노드에 정확히 1개씩 배치된 것을 확인할 수 있다.
- `kubectl rollout history daemonset <이름>`의 CHANGE-CAUSE가 `<none>`으로 나오는 경우, `kubectl annotate ... kubernetes.io/change-cause=... --overwrite`를 이미지 변경 전에 미리 실행해두지 않았기 때문이다. YAML의 `metadata.annotations.kubernetes.io/change-cause`에 미리 지정해두면 최초 배포부터 이력이 남는다.
- DaemonSet은 Deployment와 달리 ReplicaSet이 아닌 ControllerRevision으로 이력을 관리하므로, 이력을 직접 조회하려면 `kubectl get controllerrevisions`를 사용해야 한다.
- 롤링 업데이트 중 `kubectl get pods -o wide --watch`로 관찰하면 한 노드씩 순차적으로 Terminating → Completed → Pending → ContainerCreating → Running 상태를 거치는 것을 확인할 수 있으며, 이는 노드당 1개 원칙을 지키면서 업데이트를 진행하기 때문이다.
- `kubectl rollout undo daemonset <이름>`으로 롤백 시에도 동일하게 노드 단위로 순차 교체되며, `kubectl describe daemonsets <이름> | grep Image`로 최종 이미지 버전을 반드시 확인한다.

---

> 📌 **핵심 요약**
> - DaemonSet은 클러스터의 각 노드마다 Pod를 1개씩 자동으로 생성/유지하는 컨트롤러이며, 노드가 추가/삭제되면 Pod도 함께 자동으로 늘거나 줄어든다
> - Deployment는 replicas로 서비스 Pod 개수를 보장하고, DaemonSet은 노드당 1개를 보장한다는 점이 핵심 차이다
> - nodeSelector, nodeAffinity, taints/tolerations로 모든 노드가 아닌 특정 노드(GPU 노드, control-plane 등)에만 배치할 수 있다
> - DaemonSet은 ReplicaSet이 아닌 ControllerRevision으로 롤아웃 이력을 관리하며, `kubectl rollout history`/`kubectl rollout undo`로 이미지 변경 이력 확인과 롤백이 가능하다
> - 관련: 2. 📦 Kubernetes - Pod 생성 · 15. ☸️ Kubernetes - StatefulSet
