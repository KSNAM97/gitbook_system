# Kubernetes - Pod 구조와 생성 · 동작 흐름

> **Tag:** #Kubernetes #Pod #kubelet #Scheduler
> **핵심 요약:** Pod의 개념과 생성 방식(Single/Multi Container), CLI·YAML을 이용한 Pod 생성 실습, Pod 생성 시 API Server→Scheduler→kubelet으로 이어지는 동작 흐름 정리

---

## 1. 개요 (Overview)

Pod는 쿠버네티스에서 컨테이너를 실행하는 가장 작은 단위이다.

쿠버네티스에서는 컨테이너(Docker 컨테이너)를 혼자서 바로 실행하지 않는다. 반드시 Pod라는 그릇 안에 담아서 실행한다. 즉, 컨테이너는 실행 파일이고 Pod는 컨테이너를 담는 실행 단위이다.

쿠버네티스 입장에서 "컨테이너 하나 실행해줘"가 아니라 "Pod 하나 실행해줘"라고 요청한다.

**왜 컨테이너를 바로 실행하지 않고 Pod를 쓰는가**

이유는 여러 컨테이너를 하나의 묶음으로 관리하기 위해서이다. 실제 서비스에서는 컨테이너 하나만 단독으로 동작하는 경우보다 서로 밀접하게 연결된 컨테이너들이 함께 움직이는 경우가 많다.

예를 들어, 웹 서버 컨테이너 + 로그 수집 컨테이너, 이 두 개는 항상 같이 실행되고 같이 종료되고 같은 네트워크를 사용해야 한다.

이때 쿠버네티스는 이 컨테이너들을 각각 따로 관리하지 않고 하나의 Pod 안에 넣어서 하나의 덩어리로 관리한다. 그래서 Pod는 컨테이너들을 묶은 단위이다.

**Pod 안에는 무엇이 들어 있는가**

Pod 안에는 다음과 같은 것들이 들어 있다.
- 하나 이상의 컨테이너
- 공통 네트워크(IP, Port 공간)
- 공통 스토리지(볼륨)
- 컨테이너 실행 규칙

중요한 포인트는 Pod 안의 모든 컨테이너는 같은 IP를 사용한다는 것이다. 그래서 Pod 안의 컨테이너들은 localhost로 서로 통신할 수 있다. 이것이 Pod를 쓰는 핵심 이유 중 하나다.

---

## 2. Pod 구조와 생성 · 동작 흐름

### Single Container Pod와 Multi Container Pod

Pod에는 컨테이너가 1개일 수도 있고 여러 개일 수도 있다.

**Single Container Pod**
- Pod 안에 컨테이너 1개
- 가장 기본적인 형태
- 연습, 테스트, 단순 서비스에서 주로 사용
- 예) nginx 컨테이너 1개만 들어 있는 Pod

**Multi Container Pod**
- Pod 안에 컨테이너 여러 개
- 서로 강하게 의존하는 컨테이너들을 묶는다.
- 예) 웹 서버 컨테이너 + 로그 수집 컨테이너
- 예) 웹 서버 컨테이너 + 프록시 컨테이너

### Pod는 어떻게 생성되는가

Pod는 보통 두 가지 방법으로 생성된다.

**명령어로 생성**
- `kubectl run` 명령을 사용하면 쿠버네티스가 내부적으로 Pod를 만든다.
- 예) `kubectl run webserver --image=nginx:latest`
- 예) `kubectl get pods`

**YAML 파일로 생성**
- Pod의 설정을 YAML 파일로 작성하고 `kubectl apply`로 생성한다.
- 예) `kubectl create -f webserver.nginx.yaml`
- 예) `kubectl get pods`

쿠버네티스에서 가장 정석적인 방법은 YAML이다. YAML 안에는 어떤 이미지로 어떤 컨테이너를 어떤 이름의 Pod로 실행할지 작성되어 있다.

### Pod의 생명 주기

Pod는 영구적인 존재가 아니다. Pod는 언제든지 삭제될 수 있고 재생성될 수 있고 다른 노드로 옮겨질 수 있다.

Pod가 죽으면 같은 Pod가 살아나는 것이 아니라 새 Pod가 만들어진다. 그래서 Pod의 IP도 바뀐다.

이것이 중요한 이유는 Pod 자체를 믿고 서비스하면 안 되기 때문이다. 실무에서는 Pod 위에 Deployment, Service 같은 개념을 얹어서 안정적인 서비스를 만든다.

### Pod와 Node의 관계

Pod는 반드시 Node(워커 노드) 위에서 실행된다. 하지만 사용자는 "이 Pod를 이 노드에 실행해라"라고 직접 지정하지 않는다.

쿠버네티스의 Scheduler가 현재 여유 있는 노드를 골라 Pod를 배치한다. 즉, Pod는 실행 단위, Node는 실행 장소라고 생각하면 된다.

### CLI를 사용한 pod 생성

```
	# CLI를 사용한  pod 생성
[root@k8s-master ~]# kubectl  run  web1  --image=nginx:1.31  --port 80
pod/web1 created


[root@k8s-master ~]# kubectl  get  namespace
NAME              STATUS   AGE
default           Active      2d19h
kube-flannel      Active      2d19h
kube-node-lease   Active      2d19h
kube-public       Active      2d19h
kube-system       Active      2d19h
resource          Active      24h
soldesk           Active      43h
studydesk         Active      43h



[root@k8s-master ~]# kubectl  get  pods
NAME   READY   STATUS    RESTARTS   AGE
web1     1/1        Running     0                 3s
```

### YAML 파일을 사용한 pod 생성

```
	# YAML 파일을 사용한 pod 생성
[root@k8s-master ~]# vi pod-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
    ports:
    - containerPort: 80
      protocol: TCP



[root@k8s-master ~]# kubectl  apply  -f   pod-nginx.yaml  --dry-run=server
pod/nginx-pod created (server dry run)


[root@k8s-master ~]# kubectl  get  pods
NAME   	   READY   STATUS    RESTARTS   AGE
nginx-pod	   1/1        Running      0                22s
web1   	   1/1        Running      0                3m25s
```

### CLI로 생성한 Pod를 YAML로 추출해서 재사용

`kubectl get pods <이름> -o yaml`로 이미 생성된 Pod의 스펙을 YAML로 추출한 뒤, 이를 참고해 새로운 Pod를 만들 수 있다.

```
	# CLI를 YAML 파일로 생성해서 pod 생성

[root@k8s-master ~]# kubectl  get  pods  web1  -o  yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2026-08-14T02:00:47Z"
  generation: 1
  labels:
    run: web1
  name: web1
  namespace: default
  resourceVersion: "112221"
  uid: 2c5ecc32-463c-4ba4-89f7-3c844722d26e
spec:
  containers:
  - image: nginx:1.31
    imagePullPolicy: IfNotPresent
    name: web1
    ports:
    - containerPort: 80
      protocol: TCP
~~~~~~~ 중간 생략 ~~~~~~~



[root@k8s-master ~]# vi  web1-copy-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web1-copy
  namespace: default
spec:
  containers:
  - image: nginx:1.31
    name: web1
    ports:
    - containerPort: 80
      protocol: TCP

: wq



[root@k8s-master ~]# kubectl  apply  -f  web1-copy-nginx.yaml
pod/web1-copy created
```

### Multi Container Pod 생성 및 컨테이너별 접속

```
	# multi container pod 생성


[root@k8s-master ~]# vi  multi-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multipod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.29.1
    ports:
    - containerPort: 80
  - name: centos-container
    image: centos:7
    command:
    - sleep
    - "10000"


[root@k8s-master ~]# kubectl  apply  -f  multi-pod.yaml


[root@k8s-master ~]# kubectl  get pods
NAME     		READY   STATUS              	RESTARTS   AGE
multipod    	0/2         ContainerCreating	0                5s
nginx-pod   	1/1         Running             	0                12m
web1        	1/1         Running             	0                15m
web1-copy	1/1         Running             	0                8m



[root@k8s-master ~]# kubectl  get  pods
NAME        	READY   STATUS 	RESTARTS   AGE
multipod    	2/2        Running   	0                 13s
nginx-pod   	1/1        Running 	0                 15m
web1        	1/1        Running 	0                 18m
web1-copy   	1/1        Running 	0                 10m




[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME        	READY   STATUS 	RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
multipod    	2/2         Running	0                73s     10.244.2.5   k8s-worker2   <none>                   <none>
nginx-pod   	1/1         Running 	0                16m    10.244.2.2   k8s-worker2   <none>                   <none>
web1        	1/1         Running  	0                19m    10.244.1.6   k8s-worker1   <none>                   <none>
web1-copy	1/1         Running 	0                11m    10.244.1.7   k8s-worker1   <none>                   <none>



[root@k8s-master ~]# kubectl  describe  pods
Name:             	multipod
Namespace:    	default
Priority:         	0
Service Account: 	default
Node:             	k8s-worker2/192.168.10.102
Start Time:       	Fri, 14 Aug 2026 11:19:08 +0900
Labels:           	<none>
Annotations:      	<none>
Status:           	Running
IP:               	10.244.2.5
IPs:
  IP:  10.244.2.5
Containers:
  nginx-container:
    Container ID:	containerd://9f2e78cad9c6ae234e6533bc906827b6df03fd6d7d6905653f0b6cd2e499f796
    Image:          	nginx:1.29.1
    Image ID:       	docker.io/library/nginx@sha256:8adbdcb969e2676478ee2c7ad333956f0c8e0e4c5a7463f4611d7a2e7a7ff5dc
    Port:           	80/TCP
    Host Port:     	 0/TCP
    State:          	Running
      Started:      	Fri, 14 Aug 2026 11:19:09 +0900
    Ready:          	True
    Restart Count:  	0
    Environment:    	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-26k72 (ro)
  centos-container:
    Container ID:  	containerd://92d426f49ba7bc00d545b37d1b71f216ab22af336117404436dc676eb05627d9
    Image:         	centos:7
    Image ID:      	docker.io/library/centos@sha256:be65f488b7764ad3638f236b7b515b3678369a5124c47b8d32916d6487418ea4
    Port:          	<none>
    Host Port:     	<none>
    Command:
      sleep
      10000
    State:          	Running
      Started:      	Fri, 14 Aug 2026 11:19:17 +0900
    Ready:          	True
    Restart Count:  	0
    Environment:    	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-26k72 (ro)
~~~~~~~~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~~~~~~~~ 
```

`-c` 옵션으로 Multi Container Pod 안의 특정 컨테이너를 지정해 접속할 수 있다.

```
	# 컨테이너로 접속

[root@k8s-master ~]# cat  multi-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multipod		# pod 명
spec:
  containers:
  - name: nginx-container	# container 명
    image: nginx:1.29.1
    ports:
    - containerPort: 80
  - name: centos-container	# container 명
    image: centos:7
    command:
    - sleep
    - "10000"


[root@k8s-master ~]# kubectl  exec  multipod  -c  nginx-container  -it --  /bin/bash
root@multipod:/#
root@multipod:/# cat  /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>




root@multipod:/# apt-get  update

root@multipod:/# apt-get  install  -y  vim

root@multipod:/# vi  /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>My Web Page</title>
</head>
<body>

    <h1>Hello Kubernetes</h1>
    <p>간단한 테스트 페이지입니다.</p>

</body>
</html>



root@multipod:/# exit



[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME        	READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
multipod    	2/2     Running   0          24m   10.244.2.5   k8s-worker2   <none>           <none>
nginx-pod   	1/1     Running   0          39m   10.244.2.2   k8s-worker2   <none>           <none>
web1        	1/1     Running   0          42m   10.244.1.6   k8s-worker1   <none>           <none>
web1-copy   	1/1     Running   0          34m   10.244.1.7   k8s-worker1   <none>           <none>
[root@k8s-master ~]#



[root@k8s-master ~]# curl  http://10.244.2.5
<!DOCTYPE html>
<html>
<head>
    <title>My Web Page</title>
</head>
<body>

    <h1>Hello Kubernetes</h1>
    <p>간단한 테스트 페이지입니다.</p>

</body>
</html>



[root@k8s-master ~]# kubectl  logs  multipod -c  nginx-container
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/08/14 02:19:09 [notice] 1#1: using the "epoll" event method
2026/08/14 02:19:09 [notice] 1#1: nginx/1.29.1
2026/08/14 02:19:09 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14+deb12u1)
2026/08/14 02:19:09 [notice] 1#1: OS: Linux 5.14.0-687.36.1.el9_8.x86_64
2026/08/14 02:19:09 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1024:524288
2026/08/14 02:19:09 [notice] 1#1: start worker processes
2026/08/14 02:19:09 [notice] 1#1: start worker process 29
2026/08/14 02:19:09 [notice] 1#1: start worker process 30
2026/08/14 02:19:09 [notice] 1#1: start worker process 31
2026/08/14 02:19:09 [notice] 1#1: start worker process 32
10.244.0.0 - - [14/Aug/2026:02:41:38 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.76.1" "-"
10.244.0.0 - - [14/Aug/2026:02:43:38 +0000] "GET / HTTP/1.1" 200 175 "-" "curl/7.76.1" "-"
10.244.0.0 - - [14/Aug/2026:02:48:45 +0000] "GET / HTTP/1.1" 200 175 "-" "curl/7.76.1" "-"
```

### Pod 동작 flow — 사용자 요청부터 Running까지

**1) 사용자가 Pod 생성 요청**

```
# kubectl apply -f pod.yaml

	또는

# kubectl run nginx --image=nginx
```

사용자가 "Pod 만들어줘"라고 API Server에게 요청한다. kubectl은 Pod를 직접 만들지 않는다. kubectl은 그냥 요청 도구이다.

**2) API Server가 kubectl 요청을 받는다**

API Server 역할:
- 요청 문법 검사
- YAML 문법 맞는지
- apiVersion, kind, spec 구조 맞는지

권한 검사:
- 이 사용자가 Pod 만들 권한 있는지

이상 없으면 etcd에 저장한다.

이 상태의 Pod는 상태: Pending, 이유: 아직 실행할 노드가 정해지지 않음이다.

**3) Scheduler가 노드를 선택**

Scheduler는 아직 노드가 정해지지 않은 Pod 있나 계속 확인한다.

노드가 정해지지 않은 Pod가 있으면 아래 기준으로 노드를 고른다.
- CPU, 메모리 여유
- nodeSelector, affinity 조건
- taint / toleration
- 기존 Pod 분포

위의 기준에 의해 Pod는 worker-node-X를 선택하고 그 결과를 API Server에 다시 알려준다.

**4) 선택된 노드의 kubelet이 감지**

worker-node-X에는 kubelet이 항상 실행 중이다.

kubelet은 "내 노드에서 실행해야 할 Pod 있나" 계속 감시한다.

API Server에서 "worker-node-X 이 Pod 실행해"라는 정보를 받으면 실행을 시작한다.

**5) 컨테이너 생성 준비**

kubelet이 하는 일:
- Pod 스펙 읽기
- "파드가 이렇게 생겼어", "컨테이너는 몇 개고", "이미지는 뭐고", "포트, 볼륨은 이렇게 써"를 판단하고 지시하는 프로그램이다.

kubelet은 이미지 다운로드, 컨테이너 프로세스 생성, 네임스페이스 / cgroup 설정, 실제 리눅스 프로세스 실행 같은 저수준 리눅스 작업을 직접 하지 않는다.

컨테이너 런타임은 컨테이너 만드는 전문 엔진이다.

예시: containerd, CRI-O

컨테이너 런타임은 이미지 pull, 컨테이너 생성, 컨테이너 실행, 컨테이너 중지 / 삭제 같은 작업을 수행한다.

**6) 이미지 다운로드**

컨테이너 런타임 동작: 이미지가 로컬에 이미 있으면 바로 사용하고 없으면 레지스트리에서 pull한다. 이미지 다운로드 완료 후 컨테이너를 생성하고 실행한다.

**7) Pod Running 상태**

컨테이너가 정상 실행되면 kubelet → API Server에 상태를 보고한다.

Pod 상태 변화: Pending → Running

결과:

```
kubectl get pods
STATUS: Running
```

**Pod 실행 이후 지속 동작**

Pod가 실행된 이후에도 kubelet은 계속 감시한다.
- 컨테이너 죽었나? (컨테이너 프로세스가 종료되었는지 확인)
- readiness probe 실패? (이 컨테이너가 서비스 요청을 받을 준비가 되었는지 확인)
- liveness probe 실패? (이 컨테이너가 정상적으로 살아있는지 확인) — 프로세스는 살아 있지만 응답이 없는 경우, liveness probe 실패 시 kubelet이 컨테이너를 삭제 후 컨테이너를 다시 생성한다.

상황에 따라 컨테이너 재시작, Pod 재생성(Deployment라면)이 일어난다.

---

>  **핵심 요약**
> - Pod는 컨테이너를 담는 실행 단위이며, Pod 안의 모든 컨테이너는 같은 IP·네트워크·볼륨을 공유한다
> - Pod는 Single Container Pod와 Multi Container Pod로 나뉘며, `kubectl run` 또는 YAML(`kubectl apply`)로 생성한다
> - `kubectl get pods <이름> -o yaml`로 기존 Pod의 스펙을 추출해 새로운 YAML을 만들 수 있고, Multi Container Pod는 `-c <컨테이너명>` 옵션으로 특정 컨테이너에 접속·로그 확인이 가능하다
> - Pod 생성은 kubectl 요청 → API Server 접수(Pending) → Scheduler 노드 결정 → kubelet 감지·실행 지시 → 컨테이너 런타임이 이미지 pull 및 실행 → Running의 흐름을 따른다
> - 관련: 2.  Kubernetes - Pod 생성 · 7.  Kubernetes - livenessProbe · 9.  Kubernetes - Init Container·Static Pod
