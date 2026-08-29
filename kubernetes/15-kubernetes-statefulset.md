# ☸️ Kubernetes - StatefulSet

> **Tag:** #Kubernetes #StatefulSet #Controller #HeadlessService #부트캠프
> **핵심 요약:** 상태(State)가 있는 애플리케이션을 위한 컨트롤러 StatefulSet의 개념, Headless Service와의 관계, scale-out/scale-in 및 롤링 업데이트 실습 정리

---

## 1. 📖 개요 (Overview)

StatefulSet은 상태(State)가 있는 애플리케이션을 안정적으로 운영하기 위한 Kubernetes 컨트롤러이다.

Deployment와 비슷하게 여러 Pod를 관리하지만, StatefulSet은 각 Pod의 고유한 이름과 순서를 유지한다.

상태(state)란 애플리케이션이 실행되면서 계속 기억하거나 유지해야 하는 정보를 말한다.

예:
- 데이터
- 고유한 이름
- Pod의 순서
- 각 인스턴스의 역할
- 재시작 후에도 유지해야 하는 정보

예를 들어 데이터베이스에서는 mysql-0, mysql-1, mysql-2처럼 각각의 인스턴스가 구분될 필요가 있을 수 있다.

Deployment/ReplicaSet은 Pod가 죽으면 새로 만들 때 이름도 바뀌고 어느 노드에 뜰지도 바뀌고 Pod의 정체성이 계속 바뀌어도 상관없는 무상태(Stateless) 서비스에 최적화되어 있다.

반면 StatefulSet은 Pod의 이름이 고정되고 Pod마다 고유한 저장소(PVC)를 붙일 수 있고 Pod를 만들고 지우는 순서가 보장된다. 그래서 DB, 메시지큐, 분산 시스템처럼 각 인스턴스가 구분되어야 하는 서비스에 사용한다.

**Deployment와 StatefulSet 차이**

**(1) Pod 이름(정체성)**

- Deployment: pod 이름이 랜덤 해시가 붙음 (예: `web-7f9d7c9d7f-abcde`). 죽으면 다른 이름으로 다시 생성됨
- StatefulSet: pod 이름이 규칙적으로 고정됨 (예: `db-0, db-1, db-2`). db-1이 죽으면 db-1이라는 이름으로 다시 생성됨. 즉, 정체성이 유지된다.

**(2) 저장소(Persistent Volume)**

- Deployment: 여러 Pod가 같은 PVC를 쓰면 충돌 가능. 보통은 공유 저장소가 아니면 위험
- StatefulSet: 각 Pod마다 전용 PVC를 자동으로 만들 수 있음 (db-0 전용 PVC, db-1 전용 PVC …). Pod가 재생성되어도 그 Pod 번호의 PVC를 다시 붙인다. (그래서 데이터가 유지된다)

**(3) 생성/삭제 순서 보장**

- Deployment: 동시에 여러 Pod가 막 생성/삭제될 수 있음. 순서 보장 없음
- StatefulSet: 기본적으로 순서가 보장됨. 생성은 0 → 1 → 2 순서대로, 삭제는 2 → 1 → 0 역순으로. 앞 Pod가 Ready 상태가 되어야 다음 Pod 생성 같은 흐름이 가능

**(4) 업데이트(롤링업데이트)**

- Deployment: 일반적으로 빠르게 순차 업데이트
- StatefulSet: 업데이트도 순서대로 진행. 데이터가 있는 서비스는 업데이트 시 순서가 중요해서 StatefulSet 방식이 필요할 때가 많다.

**StatefulSet이 꼭 필요한 대표 서비스 예시**

(1) 데이터베이스 — mysql, mariadb, postgresql. DB는 각 인스턴스가 자기 데이터 저장소를 가져야 한다. 죽었다가 살아나도 내 데이터 디스크를 붙여야 한다.

(2) 분산 시스템 / 클러스터형 서비스 — kafka, zookeeper, elasticsearch, redis cluster(일부 구성). 이런 것들은 노드마다 역할이 있고(leader/follower) 특정 노드 이름으로 통신하거나 노드 수를 기준으로 클러스터가 구성된다. 그래서 고정된 정체성 + 고정된 스토리지가 필요하다.

**Headless Service (헤드리스 서비스)**

StatefulSet Pod들은 서로를 고정된 DNS 이름으로 찾아야 하는 경우가 많다. 예를 들어 db-0이 db-1에게 연결할 때 IP를 쓰면 Pod 재생성 때 IP가 바뀔 수 있다. 그래서 DNS 이름으로 통신해야 안정적이다.

Headless Service는 LoadBalancer처럼 하나의 IP로 묶어주는 서비스가 아니라 각 Pod의 DNS를 만들어주는 서비스다.

StatefulSet DNS의 전형적인 형태:
- `<pod이름>.<서비스이름>.<네임스페이스>.svc.cluster.local`
- 예) `db-0.db-headless.default.svc.cluster.local`
- 예) `db-1.db-headless.default.svc.cluster.local`
- 즉, Pod 개별 주소록을 만들어주는 역할을 한다.

**StatefulSet 구성 요소**

StatefulSet을 만들 때 보통 3개가 같이 온다.

1. **StatefulSet 오브젝트** — 몇 개 Pod를 유지할지(replicas), pod template(컨테이너 정의), volumeClaimTemplates(Pod별 PVC 자동 생성 템플릿)
2. **Headless Service** — serviceName으로 StatefulSet과 연결, Pod DNS 생성
3. **PV/PVC** — 각 Pod가 쓰는 저장소. StatefulSet이 만든 PVC는 Pod가 살아있든 죽었든 남는다. (Pod 삭제해도 데이터가 남는 게 핵심)

가장 중요한 개념 3개
1. **고정된 이름** — db-0, db-1, db-2 처럼 번호 붙은 고정 Pod 이름
2. **Pod별 전용 디스크** — 각 Pod마다 PVC가 별도로 생성되어 데이터 유지
3. **순서 보장** — 0부터 만들고, 역순으로 지움. 업데이트도 순서대로 진행

**실무 상황 예시**

상황: mysql을 3개로 운영(클러스터 or 복제 구조). mysql-0, mysql-1, mysql-2을 서비스 하는 상황에서 mysql-1이 장애로 죽었다.

Deployment라면 새 Pod가 mysql-xxxxx 이런 이름으로 생기고 어떤 디스크를 붙여야 하는지 불명확해지거나 재구성 과정이 복잡해진다.

StatefulSet이라면 mysql-1이라는 이름 그대로 다시 살아난다. mysql-1 전용 PVC를 다시 붙인다. 클러스터 입장에서 "mysql-1이 다시 돌아왔네"처럼 안정적으로 처리 가능하다.

**StatefulSet 쓸 때 주의점**

(1) 스토리지가 중요하다 — StatefulSet은 결국 디스크가 핵심이다. 따라서 StorageClass, PV 동작을 이해해야 운영이 된다.

(2) Pod를 지워도 PVC는 남는다 — Pod 삭제는 데이터 삭제가 아니다. 재배포/재구성할 때 PVC가 남아 있어서 이전 데이터가 그대로 붙을 수 있다. 필요하면 PVC까지 삭제해야 완전 초기화된다.

(3) 무상태 서비스에 쓰면 오히려 불편 — 단순 웹서버 nginx 같은 건 Deployment가 더 적합하다. StatefulSet은 고정성 때문에 유연성이 떨어질 수 있다. StatefulSet은 상태(state)가 있는 애플리케이션을 쿠버네티스에서 안정적으로 운영하기 위한 컨트롤러다.

---

## 2. 🛠️ 실습

### StatefulSet YAML 예제

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sf-nginx

spec:
  replicas: 3
  serviceName: sf-service
  podManagementPolicy: OrderedReady
# podManagementPolicy: Parallel
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

**podManagementPolicy: OrderedReady (default)**
- Pod를 순서대로 생성한다.
- 앞 번호의 Pod가 Running/Ready 상태가 되어야 다음 Pod 생성을 진행한다.
- Pod 간 순서가 중요할 때 사용한다.
- 앞 번호의 Pod가 정상 Ready 상태가 된 뒤 다음 번호의 Pod를 생성해야 하는 경우에 적합하다.

생성: `pod-0 → pod-1 → pod-2` / 삭제: `pod-2 → pod-1 → pod-0` — StatefulSet을 축소하거나 삭제할 때는 높은 번호부터 역순으로 처리한다.

**podManagementPolicy: Parallel**
- Pod를 순서대로 기다리지 않고 병렬로 생성한다.
- 앞 Pod가 Ready 상태가 될 때까지 기다리지 않고 다른 Pod도 생성한다.
- Pod 간 생성 순서가 중요하지 않을 때 사용한다.
- 각 Pod가 서로 독립적으로 실행될 수 있고 빠르게 여러 Pod를 생성해야 하는 경우에 적합하다.

생성: pod-0, pod-1, pod-2 생성 요청이 병렬로 진행되며, 각 Pod는 독립적으로 Running/Ready 상태가 된다. Scale Down 시에도 OrderedReady처럼 앞 Pod의 상태를 기다리는 순차적인 관리 제약을 적용하지 않고 병렬적으로 처리할 수 있다.

### StatefulSet 생성

```
# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  statefulset-exam.yaml  --dry-run=client
statefulset.apps/sf-nginx created (dry run)


# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch



# 터미널 2
[root@k8s-master ~]# kubectl  apply  -f  statefulset-exam.yaml
statefulset.apps/sf-nginx created



# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME         READY      STATUS    		RESTARTS   AGE   IP              NODE     	NOMINATED NODE   READINESS GATES
sf-nginx-0   0/1            Pending   		0          	     0s     <none>        <none>   	<none>           <none>
sf-nginx-0   0/1            Pending   		0          	     0s     <none>        k8s-worker2   	<none>           <none>
sf-nginx-0   0/1            ContainerCreating	0                 0s     <none>        k8s-worker2   	<none>           <none>
sf-nginx-0   1/1            Running             	0                 1s     10.244.2.47   k8s-worker2   	<none>           <none>

sf-nginx-1   0/1            Pending             	0                 0s     <none>        <none>        	<none>           <none>
sf-nginx-1   0/1            Pending             	0                 0s     <none>        k8s-worker1   	<none>           <none>
sf-nginx-1   0/1            ContainerCreating   	0                 0s     <none>        k8s-worker1   	<none>           <none>
sf-nginx-1   1/1            Running             	0                 1s     10.244.1.64   k8s-worker1   	<none>           <none>

sf-nginx-2   0/1            Pending             	0                 0s     <none>        <none>        	<none>           <none>
sf-nginx-2   0/1            Pending             	0                 0s     <none>        k8s-worker1   	<none>           <none>
sf-nginx-2   0/1            ContainerCreating   	0                 0s     <none>        k8s-worker1   	<none>           <none>
sf-nginx-2   1/1            Running             	0                 1s     10.244.1.65   k8s-worker1   	<none>           <none>
```

Pod가 sf-nginx-0 → sf-nginx-1 → sf-nginx-2 순서대로, 각 Pod가 Running 상태가 된 뒤에 다음 Pod가 생성되는 것을 확인할 수 있다.

### scale-out (확장)

```
`[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME         READY      STATUS    RESTARTS   AGE     IP              NODE          	NOMINATED NODE   READINESS GATES
sf-nginx-0   1/1            Running     0                 1s       10.244.2.47   k8s-worker2   	<none>           <none>
sf-nginx-1   1/1            Running     0                 1s       10.244.1.64   k8s-worker1   	<none>           <none>
sf-nginx-2   1/1            Running     0                 1s       10.244.1.65   k8s-worker1   	<none>           <none>



[root@k8s-master ~]# kubectl  scale  statefulset  sf-nginx  --replicas=5
statefulset.apps/sf-nginx scaled



# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME         READY      STATUS    		RESTARTS   AGE   IP              NODE     	NOMINATED NODE   READINESS GATES
sf-nginx-0   1/1            Running             	0                 1s     10.244.2.47   k8s-worker2   	<none>           <none>
sf-nginx-1   1/1            Running             	0                 1s     10.244.1.64   k8s-worker1   	<none>           <none>
sf-nginx-2   1/1            Running             	0                 1s     10.244.1.65   k8s-worker1   	<none>           <none>

sf-nginx-3   0/1            Pending   		0                 0s      <none>        <none>        	<none>           <none>
sf-nginx-3   0/1            Pending   		0                 0s      <none>        k8s-worker2   	<none>           <none>
sf-nginx-3   0/1            ContainerCreating   	0                 0s      <none>        k8s-worker2   	<none>           <none>
sf-nginx-3   1/1            Running             	0                 1s      10.244.2.48   k8s-worker1   	<none>           <none>

sf-nginx-4   0/1            Pending             	0                 0s      <none>        <none>        	<none>           <none>
sf-nginx-4   0/1            Pending             	0                 0s      <none>        k8s-worker1   	<none>           <none>
sf-nginx-4   0/1            ContainerCreating   	0                 0s      <none>        k8s-worker1   	<none>           <none>
sf-nginx-4   1/1            Running             	0                 1s      10.244.1.66   k8s-worker1   	<none>           <none>
```

replicas를 5로 늘리면 sf-nginx-3, sf-nginx-4가 순서대로 추가 생성된다.

### scale-in (축소)

```
# 터미널 1
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME         READY      STATUS    	RESTARTS   AGE    IP              NODE     	NOMINATED NODE   READINESS GATES
sf-nginx-0   1/1            Running         	0                 1s       10.244.2.47   k8s-worker2   	<none>           <none>
sf-nginx-1   1/1            Running       	0                 1s       10.244.1.64   k8s-worker1   	<none>           <none>
sf-nginx-2   1/1            Running      	0                 1s       10.244.1.65   k8s-worker1   	<none>           <none>
sf-nginx-3   1/1            Running       	0                 1s       10.244.2.48   k8s-worker1   	<none>           <none>
sf-nginx-4   1/1            Running  	0                 1s       10.244.1.66   k8s-worker1   	<none>           <none>


[root@k8s-master ~]# kubectl  scale  statefulset  sf-nginx  --replicas=2
statefulset.apps/sf-nginx scaled


[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME         READY      STATUS    	RESTARTS   AGE    IP              NODE     	NOMINATED NODE   READINESS GATES
sf-nginx-0   1/1            Running         	0                 1s       10.244.2.47   k8s-worker2   	<none>           <none>
sf-nginx-1   1/1            Running       	0                 1s       10.244.1.64   k8s-worker1   	<none>           <none>
sf-nginx-2   1/1            Running      	0                 1s       10.244.1.65   k8s-worker1   	<none>           <none>
sf-nginx-3   1/1            Running       	0                 1s       10.244.2.48   k8s-worker1   	<none>           <none>
sf-nginx-4   1/1            Running  	0                 1s       10.244.1.66   k8s-worker1   	<none>           <none>


sf-nginx-4   1/1            Terminating   	0                 3m1s    10.244.1.66   k8s-worker1   <none>           <none>
sf-nginx-4   1/1            Terminating   	0                 3m1s    10.244.1.66   k8s-worker1   <none>           <none>
sf-nginx-4   0/1            Completed     	0                 3m1s    10.244.1.66   k8s-worker1   <none>           <none>
sf-nginx-4   0/1            Completed     	0                 3m1s    10.244.1.66   k8s-worker1   <none>           <none>
sf-nginx-4   0/1            Completed     	0                 3m1s    10.244.1.66   k8s-worker1   <none>           <none>

sf-nginx-3   1/1            Terminating   	0                 3m2s    10.244.2.48   k8s-worker2   <none>           <none>
sf-nginx-3   1/1            Terminating   	0                 3m2s    10.244.2.48   k8s-worker2   <none>           <none>
sf-nginx-3   0/1            Completed     	0                 3m2s    10.244.2.48   k8s-worker2   <none>           <none>
sf-nginx-3   0/1            Completed     	0                 3m3s    10.244.2.48   k8s-worker2   <none>           <none>
sf-nginx-3   0/1            Completed     	0                 3m3s    10.244.2.48   k8s-worker2   <none>           <none>

sf-nginx-2   1/1            Terminating   	0                 7m29s   10.244.1.65   k8s-worker1   <none>           <none>
sf-nginx-2   1/1            Terminating   	0                 7m29s   10.244.1.65   k8s-worker1   <none>           <none>
sf-nginx-2   0/1            Completed     	0                 7m29s   10.244.1.65   k8s-worker1   <none>           <none>
sf-nginx-2   0/1            Completed     	0                 7m29s   10.244.1.65   k8s-worker1   <none>           <none>
sf-nginx-2   0/1            Completed     	0                 7m29s   10.244.1.65   k8s-worker1   <none>           <none>
```

replicas를 2로 줄이면 sf-nginx-4 → sf-nginx-3 → sf-nginx-2 순서로, 가장 높은 번호부터 역순으로 종료된다.

### 롤링 업데이트

```
[root@k8s-master ~]# kubectl  edit  statefulsets  sf-nginx
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  podManagementPolicy: OrderedReady
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: web
  serviceName: sf-service
  template:
    metadata:
      labels:
        app: web
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.31.1 		#  nginx:1.31  -->  nginx:1.31.1
        imagePullPolicy: IfNotPresent
        name: nginx-container
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~



[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME         READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
sf-nginx-0   1/1     Running   0          16m     10.244.2.47   k8s-worker2   <none>           <none>
sf-nginx-1   1/1     Running   0          5m45s   10.244.1.68   k8s-worker1   <none>           <none>
sf-nginx-2   1/1     Running   0          6m12s   10.244.1.67   k8s-worker1   <none>           <none>
sf-nginx-2   1/1     Terminating   0          8m21s   10.244.1.67   k8s-worker1   <none>           <none>
sf-nginx-2   1/1     Terminating   0          8m21s   10.244.1.67   k8s-worker1   <none>           <none>
sf-nginx-2   0/1     Completed     0          8m21s   10.244.1.67   k8s-worker1   <none>           <none>
sf-nginx-2   0/1     Completed     0          8m22s   10.244.1.67   k8s-worker1   <none>           <none>
sf-nginx-2   0/1     Completed     0          8m22s   10.244.1.67   k8s-worker1   <none>           <none>
sf-nginx-2   0/1     Pending       0          0s      <none>        <none>        <none>           <none>
sf-nginx-2   0/1     Pending       0          0s      <none>        k8s-worker1   <none>           <none>
sf-nginx-2   0/1     ContainerCreating   0          0s      <none>        k8s-worker1   <none>           <none>
sf-nginx-2   1/1     Running             0          1s      10.244.1.69   k8s-worker1   <none>           <none>
sf-nginx-1   1/1     Terminating         0          7m56s   10.244.1.68   k8s-worker1   <none>           <none>
sf-nginx-1   1/1     Terminating         0          7m56s   10.244.1.68   k8s-worker1   <none>           <none>
sf-nginx-1   0/1     Completed           0          7m56s   10.244.1.68   k8s-worker1   <none>           <none>
sf-nginx-1   0/1     Completed           0          7m57s   10.244.1.68   k8s-worker1   <none>           <none>
sf-nginx-1   0/1     Completed           0          7m57s   10.244.1.68   k8s-worker1   <none>           <none>
sf-nginx-1   0/1     Pending             0          0s      <none>        <none>        <none>           <none>
sf-nginx-1   0/1     Pending             0          0s      <none>        k8s-worker1   <none>           <none>
sf-nginx-1   0/1     ContainerCreating   0          0s      <none>        k8s-worker1   <none>           <none>
sf-nginx-1   1/1     Running             0          1s      10.244.1.70   k8s-worker1   <none>           <none>
sf-nginx-0   1/1     Terminating         0          18m     10.244.2.47   k8s-worker2   <none>           <none>
sf-nginx-0   1/1     Terminating         0          18m     10.244.2.47   k8s-worker2   <none>           <none>
sf-nginx-0   0/1     Completed           0          18m     10.244.2.47   k8s-worker2   <none>           <none>
sf-nginx-0   0/1     Completed           0          18m     10.244.2.47   k8s-worker2   <none>           <none>
sf-nginx-0   0/1     Completed           0          18m     10.244.2.47   k8s-worker2   <none>           <none>
sf-nginx-0   0/1     Pending             0          0s      <none>        <none>        <none>           <none>
sf-nginx-0   0/1     Pending             0          0s      <none>        k8s-worker2   <none>           <none>
sf-nginx-0   0/1     ContainerCreating   0          0s      <none>        k8s-worker2   <none>           <none>
sf-nginx-0   1/1     Running             0          1s      10.244.2.49   k8s-worker2   <none>           <none>
```

StatefulSet의 롤링 업데이트는 scale-in과 마찬가지로 가장 높은 번호(sf-nginx-2)부터 역순으로 sf-nginx-1, sf-nginx-0 순서로 교체되며, 각 Pod가 Running 상태가 된 뒤 다음 번호의 Pod 업데이트가 진행된다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `kubectl get pods -o wide --watch`로 관찰 시, StatefulSet의 생성/롤링 업데이트는 항상 낮은 번호부터(`0 → 1 → 2`), 삭제/롤링 업데이트 시 종료는 높은 번호부터(`2 → 1 → 0`) 순서대로 진행되는지 확인한다. 순서가 지켜지지 않으면 `podManagementPolicy`가 `Parallel`로 설정되어 있는지 점검한다.
- `kubectl scale statefulset <이름> --replicas=N`으로 축소했을 때 PVC는 자동 삭제되지 않는다. 재확장 시 이전 데이터가 그대로 붙게 되므로, 완전 초기화가 필요하면 PVC를 직접 삭제해야 한다(`persistentVolumeClaimRetentionPolicy` 설정으로 동작을 제어할 수 있다).
- Pod 간 통신이 IP 기반으로 되어 있어 재생성 후 연결이 끊긴다면 Headless Service(`clusterIP: None`)를 통해 `<pod이름>.<서비스이름>.<네임스페이스>.svc.cluster.local` 형태의 고정 DNS로 통신하도록 구성했는지 확인한다.
- StatefulSet은 `serviceName` 필드로 반드시 Headless Service와 연결되어야 하며, 이 값이 실제 존재하는 Service 이름과 일치하지 않으면 Pod DNS가 정상적으로 생성되지 않는다.
- 롤링 업데이트 중 문제가 발생하면 `kubectl rollout status statefulset <이름>` 및 `kubectl describe statefulsets <이름>`으로 현재 어느 Pod에서 멈췄는지 확인하고, `podManagementPolicy: OrderedReady`인 경우 앞 Pod가 Ready 상태가 되지 못하면 다음 Pod로 진행되지 않는다는 점을 고려한다.

---

> 📌 **핵심 요약**
> - StatefulSet은 상태(State)가 있는 애플리케이션(DB, 메시지큐, 분산 클러스터)을 위해 Pod의 이름·저장소·생성삭제 순서를 고정하여 관리하는 컨트롤러다
> - Pod 이름은 `db-0, db-1, db-2`처럼 고정되고, 각 Pod마다 전용 PVC가 생성되어 재생성되어도 같은 데이터가 다시 붙는다
> - 생성은 `0 → 1 → 2` 순서, 삭제/축소/롤링 업데이트는 `2 → 1 → 0` 역순으로 진행되며, `podManagementPolicy`(OrderedReady/Parallel)로 이 동작을 제어할 수 있다
> - StatefulSet은 Headless Service(serviceName)와 함께 사용해 각 Pod에 고정된 DNS 주소를 부여하며, Pod 삭제 후에도 PVC는 남아 데이터가 유지된다
> - 관련: 2. 📦 Kubernetes - Pod 생성 · 14. ☸️ Kubernetes - DaemonSet · 20. 🌐 Kubernetes - ExternalName·Headless Service
