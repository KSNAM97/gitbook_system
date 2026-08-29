# 🔌 Kubernetes - Service 기초와 ClusterIP

> **Tag:** #Kubernetes #Service #ClusterIP #부트캠프
> **핵심 요약:** 계속 바뀌는 Pod IP 문제를 해결하기 위해 고정된 단일 진입점을 제공하는 Service의 개념과 기본 YAML 구조, ClusterIP 타입 실습 정리

---

## 1. 📖 개요 (Overview)

쿠버네티스 Service는 계속 바뀌는 Pod들을 대신해서 고정된 단일 진입점(주소와 이름)을 제공해주는 객체이다.

쿠버네티스에서 실제 애플리케이션은 Pod 안에서 실행된다. 하지만 Pod는 고정된 서버처럼 계속 유지되는 리소스가 아니다.

1) Pod는 언제든지 죽고 새로 만들어진다.
- 노드 장애
- 컨테이너 오류
- 스케일 조정
- 업데이트(Rolling Update)

2) Pod가 새로 만들어질 때마다 IP 주소가 매번 바뀐다.

예) 웹 서버 Pod 3개 실행 중, 각각 IP가 다음과 같다.
- 10.244.1.10
- 10.244.1.11
- 10.244.2.5

즉, 어제 접속하던 Pod IP로 오늘은 접속이 안 되는 상황이 정상적인 동작이다.

이 중 하나의 Pod가 죽고 다시 생성되면 IP는 전혀 다른 값으로 바뀐다. 이 상태에서 DB, 다른 서비스, 외부 사용자가 Pod IP를 직접 사용한다면 서비스는 바로 끊어진다. 그래서 필요한 것이 Service다.

**Service의 핵심 역할**

Service는 다음 역할을 한다.
- 여러 Pod를 하나의 서비스로 묶는다.
- 고정된 접근 지점을 제공한다. (단일 진입점 제공)
- Pod가 바뀌어도 사용자는 신경 쓸 필요가 없다.
- Service 내부에서 자동으로 로드밸런싱을 수행한다.

Service는 Pod들이 바뀌어도 항상 동일한 주소와 이름으로 접근할 수 있게 해주는 중간 관리자다.

**Service가 하는 일 (동작 개념)**

Service는 Pod를 직접 생성하지 않는다. Service는 Pod를 선택만 한다. 선택 기준은 label이다.

예시 개념 흐름:

1) Pod 생성 — label: `app=web`
2) Service 생성 — selector: `app=web`
3) Service는 `app=web` 라벨을 가진 모든 Pod를 자동으로 추적
4) 사용자는 Pod IP가 아니라 Service IP 주소로 접속
5) Service는 내부적으로 여러 개의 Pod 중 하나로 트래픽 전달

Pod가 추가되거나 삭제되면 Service는 자동으로 대상 목록을 갱신한다.

**Service가 Pod를 찾는 방법 (Label & Selector)**

Pod 예시:

```yaml
metadata:
  labels:
    app: web
    tier: frontend
```

Service 예시:

```yaml
spec:
  selector:
    app: web
```

이 Service는 `app=web` 라벨을 가진 Pod만 대상으로 삼는다. 그래서 Service는 Pod 이름, Pod IP를 전혀 몰라도 된다. Label만 맞으면 자동으로 Service와 연결된다.

**Kubernetes Service Type**

쿠버네티스에서 Pod는 생성되고 삭제될 수 있으며, Pod가 다시 생성되면 IP 주소가 변경될 수 있다. 따라서 클라이언트가 Pod의 IP를 직접 사용하면 문제가 발생한다.

Service Type:
- ClusterIP
- NodePort
- LoadBalancer
- ExternalName
- Headless Service

Service는 Pod에 접근하기 위한 고정된 네트워크 엔드포인트를 제공하고, 선택된 Pod로 트래픽을 전달하는 논리적 객체다.

**Service의 주요 타입**

Service에는 여러 종류가 있으며 사용 목적에 따라 다르다.

1) **ClusterIP (기본값)**
- ClusterIP는 Service의 기본 타입이다.
- 클러스터 내부에서만 접근할 수 있는 가상 IP(ClusterIP)를 제공한다.
- 외부에서 직접 접근할 수 없다.
- 주로 클러스터 내부의 마이크로서비스 간 통신에 사용한다.
- 예: Web → API, API → DB, Frontend → Backend

2) **NodePort**
- NodePort는 각 Node의 특정 포트를 열어 외부에서 Service에 접근할 수 있도록 한다.
- 일반적으로 30000~32767 범위의 포트를 사용한다.
- 흐름: `Node IP:NodePort → Service → Pod`
- 예: Node 1 : 192.168.10.101:30080, Node 2 : 192.168.10.102:30080, Node 3 : 192.168.10.103:30080
- 각 Node에서 동일한 NodePort를 통해 Service에 접근할 수 있다.
- 운영 환경에서는 직접 NodePort를 노출하기보다 LoadBalancer나 Ingress 등을 함께 사용하는 경우가 많다.
- NodePort의 단점: Node의 IP와 포트를 외부에 노출해야 함, 외부 사용자에게 직접 NodePort를 제공하는 구조는 운영 환경에서 관리가 불편할 수 있음, 일반적으로 LoadBalancer나 Ingress 등을 함께 사용

3) **LoadBalancer**
- LoadBalancer는 외부 Load Balancer를 통해 Service에 접근할 수 있도록 하는 타입이다.
- 클라우드 환경에서 주로 사용한다. (AWS, Azure, GCP...)
- 클라우드 환경의 Kubernetes에서 LoadBalancer Service를 생성하면 클라우드 환경에 따라 외부 Load Balancer가 프로비저닝될 수 있다.
- 흐름: 외부 사용자 → Cloud Load Balancer → Service → Pod
- 외부 사용자가 Node의 IP와 NodePort를 직접 사용할 필요를 줄일 수 있다.

4) **ExternalName**
- 쿠버네티스 외부에 있는 서비스의 DNS 이름을 Service 이름으로 매핑한다.
- 즉, Pod를 연결하는 것이 아니라 "Service 이름 → 외부 DNS 이름"을 연결하는 방식이다.
- 셀렉터(selector)를 사용하지 않는다. (연결할 Pod가 없기 때문에 Pod를 선택하지 않는다.)
- 클러스터 내부의 애플리케이션이 외부 서비스를 쿠버네티스 내부의 Service 이름으로 사용할 수 있도록 하기 위한 목적이다.
- 동작 방식: 1) 클러스터 내부에서 Service 이름으로 DNS 조회 → 2) ExternalName Service → 3) 외부 DNS 이름(CNAME) 반환 → 4) 외부 서비스로 접근

예시:

```yaml
apiVersion: v1		# yaml에 pod가 존재하지 않는다.
kind: Service
metadata:
  name: mydb
spec:
  type: ExternalName
  externalName: db.external.com
```

- 외부 DB 주소: `db.external.com`
- ExternalName Service: `mydb`
- 클러스터 내부에서 `mydb.default.svc.cluster.local` ---- CNAME ----> `db.external.com`

```
mydb.default.svc.cluster.local
│     │       │      └───────────── 클러스터 내부 DNS 영역
│     │       └───────────────── Service
│     └───────────────────────── Namespace
└─────────────────────────────── Service 이름
```

`mydb.default.svc.cluster.local`는 Kubernetes DNS가 자동으로 만들어주는 이름이다.

주 사용 사례: 외부 DB, 외부 API, 외부 SaaS 서비스, Kubernetes 외부에서 운영되는 서비스 연동

5) **Headless Service**
- Headless Service는 ClusterIP에 IP 주소가 없는 Service다.
- `clusterIP: None`으로 설정한다.
- 일반적인 Service처럼 하나의 ClusterIP로 트래픽을 전달하지 않는다.
- DNS 조회 시 Service의 ClusterIP 대신 연결된 Pod들의 IP를 조회할 수 있다.
- 주로 개별 Pod에 직접 접근해야 하는 경우 사용한다.
- 예: StatefulSet, 데이터베이스 클러스터, 각 Pod를 개별적으로 식별해야 하는 경우

---

## 2. 🧩 Deployment + Service 기본 YAML 구조

```yaml
	# Deployment YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webui
  template:
    metadata:
      name: nginx-pod
      labels:
        app: webui		# 아래 Service의 selector와 연결되는 핵심 기준
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31
        ports:
        - containerPort: 80
```

```yaml
	# Service YAML
apiVersion: v1
kind: Service
metadata:
  name: webui-svc
spec:
  selector:
    app: webui		# app=webui 라벨을 가진 Pod만 이 Service의 대상으로 선택함
  ports:
  - protocol: TCP
    port: 80    		# Service가 클라이언트의 요청을 받는 포트
    targetPort: 80		# Service가 선택된 Pod로 트래픽을 전달할 포트
```

---

## 3. 🛠️ ClusterIP 실습

- selector의 label이 동일한 파드들의 그룹으로 묶어 단일 진입점(Virtual IP)을 생성한다.
- 클러스터 내부에서만 사용 가능하다.
- type 생략 시 default 값으로 10.96.0.0/12 범위에서 랜덤하게 할당된다.

### Deployment 생성

```
[root@k8s-master ~]# vi deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-web-dep
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-dep
  template:
    metadata:
      name: nginx-pod
      labels:
        app: web-dep
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31
```

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-nginx.yaml
deployment.apps/deploy-web-dep created


[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-747497b79d-bptqs   	1/1         Running     0                4s
deploy-web-dep-747497b79d-gdrnq   	1/1         Running     0                4s
deploy-web-dep-747497b79d-lnw72   	1/1         Running     0                4s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                              		READY   STATUS    RESTARTS   AGE   IP              NODE          	NOMINATED NODE   READINESS GATES
deploy-web-dep-747497b79d-bptqs   	1/1        Running     0                 63s    10.244.2.95   k8s-worker2   	<none>                    <none>
deploy-web-dep-747497b79d-gdrnq   	1/1        Running     0                 63s    10.244.2.94   k8s-worker2   	<none>                    <none>
deploy-web-dep-747497b79d-lnw72   	1/1        Running     0                 63s    10.244.1.18   k8s-worker1   	<none>                    <none>
```

### ClusterIP Service 생성

```
[root@k8s-master ~]# vi clusterip-nginx.yaml
apiVersion: v1                 	# 쿠버네티스 core API 버전 (Service는 v1)
kind: Service                  	# 생성할 리소스 종류가 Service임을 의미
metadata:
  name: clusterip-service 	# Service 이름 (클러스터 내부 DNS 이름으로 사용됨)
spec:
  type: ClusterIP              	# Service 타입 (ClusterIP는 클러스터 내부에서만 접근 가능한 서비스)
  clusterIP: 10.100.100.100    	# Service에 할당할 가상 IP(Virtual IP), 명시하지 않으면 쿠버네티스가 자동 할당
  selector:
    app: web-dep              	# app=web-dep 라벨을 가진 Pod들을 이 Service의 대상으로 선택
                               	# 해당 라벨이 있는 Pod들만 트래픽을 받음
  ports:
  - protocol: TCP           	# 통신 프로토콜 (일반적인 웹 서비스는 TCP)
    port: 80               	# Service가 외부(클러스터 내부)에 제공하는 포트 (다른 Pod들은 이 포트로 Service에 접속)
    targetPort: 80          	# 실제 Pod(컨테이너)가 리스닝 중인 포트
                            	# Service로 들어온 요청이 이 포트로 전달됨
```

```
[root@k8s-master ~]# kubectl  apply  -f  clusterip-nginx.yaml  --dry-run=client
service/clusterip-service created (dry run)


[root@k8s-master ~]# kubectl  apply  -f  clusterip-nginx.yaml
service/clusterip-service created


[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-747497b79d-bptqs   	1/1         Running     0                4s
deploy-web-dep-747497b79d-gdrnq   	1/1         Running     0                4s
deploy-web-dep-747497b79d-lnw72   	1/1         Running     0                4s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                              		READY   STATUS    RESTARTS   AGE   IP              NODE          	NOMINATED NODE   READINESS GATES
deploy-web-dep-747497b79d-bptqs   	1/1        Running     0                 63s    10.244.2.95   k8s-worker2   	<none>                    <none>
deploy-web-dep-747497b79d-gdrnq   	1/1        Running     0                 63s    10.244.2.94   k8s-worker2   	<none>                    <none>
deploy-web-dep-747497b79d-lnw72   	1/1        Running     0                 63s    10.244.1.18   k8s-worker1   	<none>                    <none>


[root@k8s-master ~]# kubectl  get  service
NAME                	TYPE        CLUSTER-IP     	EXTERNAL-IP     PORT(S)   	AGE
clusterip-service   	ClusterIP   10.100.100.100	<none>               80/TCP    	11s
kubernetes          	ClusterIP   10.96.0.1        	<none>               443/TCP	9d


[root@k8s-master ~]# kubectl  get  svc
NAME                	TYPE        CLUSTER-IP     	EXTERNAL-IP     PORT(S)   	AGE
clusterip-service   	ClusterIP   10.100.100.100	<none>               80/TCP    	11s
kubernetes          	ClusterIP   10.96.0.1        	<none>               443/TCP	9d


[root@k8s-master ~]# kubectl  get  svc   svc clusterip-service
NAME                	TYPE        CLUSTER-IP     	EXTERNAL-IP     PORT(S)   	AGE
clusterip-service   	ClusterIP   10.100.100.100	<none>               80/TCP    	11s


[root@k8s-master ~]# kubectl  describe  svce   svc clusterip-service
Name:                  		clusterip-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	app=web-dep
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	10.100.100.100
IPs:                      	10.100.100.100
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.94:80,10.244.1.18:80,10.244.2.95:80
Session Affinity:         	None
Internal Traffic Policy:	Cluster
Events:                   	<none>
```

### ClusterIP로 접속 확인

```
[root@k8s-master ~]# curl  10.100.100.100
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
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

- deploy-web-dep-747497b79d-bptqs
- deploy-web-dep-747497b79d-gdrnq
- deploy-web-dep-747497b79d-lnw72

3개의 Pod 중 어떤 pod에 접속했는지 확인할 수 없다.

### 어떤 Pod로 접속했는지 확인할 수 있도록 index.html 코드 수정

```
   # 첫번째 Pod 수정
[root@k8s-master ~]# kubectl  exec  deploy-web-dep-747497b79d-bptqs  -it  -- /bin/bash
root@deploy-web-dep-747497b79d-bptqs:/#


root@deploy-web-dep-747497b79d-bptqs:/# ls  -l  /usr/share/nginx/html/
total 8
-rw-r--r-- 1 root root 497 Jul 15 16:03 50x.html
-rw-r--r-- 1 root root 896 Jul 15 16:03 index.html


root@deploy-web-dep-747497b79d-bptqs:/# cd  /usr/share/nginx/html/


root@deploy-web-dep-747497b79d-bptqs:/usr/share/nginx/html#  echo "<h1> ClusterIP Service Web-dep-1 </h1>"  >  index.html


root@deploy-web-dep-747497b79d-bptqs:/usr/share/nginx/html# cat index.html                     
<h1> ClusterIP Service Web-dep-1 </h1>


   # 두번째 Pod 수정
[root@k8s-master ~]# kubectl  exec  deploy-web-dep-747497b79d-gdrnq  -it  -- /bin/bash
root@deploy-web-dep-747497b79d-gdrnq:/#

root@deploy-web-dep-747497b79d-gdrnq:/# cd  /usr/share/nginx/html/

root@deploy-web-dep-747497b79d-gdrnq:/usr/share/nginx/html#  echo "<h1> ClusterIP Service Web-dep-2 </h1>"  >  index.html


root@deploy-web-dep-747497b79d-gdrnq:/usr/share/nginx/html# cat index.html                     
<h1> ClusterIP Service Web-dep-2 </h1>


   # 세번째 Pod 수정
[root@k8s-master ~]# kubectl  exec  deploy-web-dep-747497b79d-lnw72  -it  -- /bin/bash
root@deploy-web-dep-747497b79d-lnw72:/#

root@deploy-web-dep-747497b79d-lnw72:/# cd  /usr/share/nginx/html/

root@deploy-web-dep-747497b79d-lnw72:/usr/share/nginx/html#  echo "<h1> ClusterIP Service Web-dep-3 </h1>"  >  index.html

root@deploy-web-dep-747497b79d-lnw72:/usr/share/nginx/html# cat index.html                     
<h1> ClusterIP Service Web-dep-3 </h1>
```

### 로드밸런싱 확인 — 요청마다 다른 Pod로 분산된다

```
[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-3 </h1>


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-2 </h1>


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-3 </h1>


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-1 </h1>


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-2 </h1>
```

### 스케일 아웃 후 새 Pod도 자동으로 대상에 추가

```
[root@k8s-master ~]# kubectl  scale  deployment  deploy-web-dep  --replicas=4
deployment.apps/deploy-web-dep scaled


[root@k8s-master ~]# kubectl  describe  service  clusterip-service
Name:                     	clusterip-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	app=web-dep
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	10.100.100.100
IPs:                      	10.100.100.100
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.94:80,10.244.1.18:80,10.244.2.95:80 + 1 more...
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                   	<none>


[root@k8s-master ~]# kubectl  get  pods
NAME                              READY   STATUS    RESTARTS   AGE
deploy-web-dep-747497b79d-bptqs   1/1     Running   0          44m
deploy-web-dep-747497b79d-gdrnq   1/1     Running   0          44m
deploy-web-dep-747497b79d-lnw72   1/1     Running   0          44m
deploy-web-dep-747497b79d-ndr9b   1/1     Running   0          7s


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-1 </h1>


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-2 </h1>


[root@k8s-master ~]# curl  http://10.100.100.100
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
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


[root@k8s-master ~]# curl  http://10.100.100.100
<h1> ClusterIP Service Web-dep-3 </h1>
```

새로 추가된 Pod(`deploy-web-dep-747497b79d-ndr9b`)는 아직 index.html을 수정하지 않았기 때문에 nginx 기본 페이지가 응답으로 나온다. 즉 신규 Pod도 Selector 조건만 맞으면 Service 대상에 자동으로 포함된다.

### selector를 별도 label(`service: nginx`)로 분리해서 재구성

```
[root@k8s-master ~]# vi deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-web-dep
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-dep
  template:
    metadata:
      name: nginx-pod
      labels:
        app: web-dep
        service: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31
```

```
[root@k8s-master ~]# vi clusterip-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service 
spec:
  type: ClusterIP
  clusterIP: 10.100.100.100
  selector:
    service: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-nginx.yaml
deployment.apps/deploy-web-dep created


[root@k8s-master ~]# kubectl  apply  -f  clusterip-nginx.yaml
service/clusterip-service created


[root@k8s-master ~]# kubectl  get  deployments  deploy-web-dep
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web-dep   2/2         2                    2                  5m30s


[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-578859465c-rfghn   	1/1     Running   0          3m20s
deploy-web-dep-578859465c-vfblj   	1/1     Running   0          3m20s


[root@k8s-master ~]# kubectl  describe  service  clusterip-service
Name:                     	clusterip-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	service=nginx
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	10.100.100.100
IPs:                      	10.100.100.100
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.96:80,10.244.1.21:80
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                   	<none>
```

### Service에 포함되는 새로운 Pod 생성 (label이 일치하는 경우)

```
[root@k8s-master ~]# vi svc-in-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: svc-in-pod
  labels:
    service: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    ports:
    - containerPort: 80
```

```
[root@k8s-master ~]# kubectl  apply  -f   svc-in-pod.yaml
pod/svc-in-pod created


	# Deployment에의해 관리되는 Pod는 2개
[root@k8s-master ~]# kubectl  get  deployments  deploy-web-dep
NAME             	READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web-dep   	2/2         2                    2                 8m11s


	# 총 Pod는 3개
[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-578859465c-rfghn   	1/1     Running   0          8m6s
deploy-web-dep-578859465c-vfblj   	1/1     Running   0          8m6s
svc-in-pod                        		1/1     Running   0          8s


	# ClusterIP Service에의해 서비스 되는 Pod는 3개
[root@k8s-master ~]# kubectl  describe  service  clusterip-service
Name:                     	clusterip-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	service=nginx
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	10.100.100.100
IPs:                      	10.100.100.100
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.96:80,10.244.1.21:80,10.244.2.97:80
Session Affinity:        	 None
Internal Traffic Policy:  	Cluster
Events:                   	<none>
```

### Service에 포함되지 않는 새로운 Pod 생성 (label이 불일치하는 경우)

```
[root@k8s-master ~]# vi svc-out-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: svc-out-pod
  labels:
    service: apache
spec:
  containers:
  - name: nginx
    image: nginx:1.31
    ports:
    - containerPort: 80
```

```
[root@k8s-master ~]# kubectl apply  -f  svc-out-pod.yaml
pod/svc-out-pod created


[root@k8s-master ~]# kubectl  get  deployments  deploy-web-dep
NAME             	READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web-dep	2/2         2                    2                 35m


[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-578859465c-rfghn   	1/1        Running      0                35m
deploy-web-dep-578859465c-vfblj   	1/1        Running      0                35m
svc-in-pod                        		1/1        Running      0                27m
svc-out-pod                       		1/1        Running      0                10s


[root@k8s-master ~]# kubectl  describe  service  clusterip-service
Name:                     	clusterip-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	service=nginx
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:             	IPv4
IP:                       	10.100.100.100
IPs:                      	10.100.100.100
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.96:80,10.244.1.21:80,10.244.2.97:80
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                   	<none>
```

`svc-out-pod`는 label이 `service=apache`이므로 Endpoints 목록에 포함되지 않는다.

### 특정 pod에 label 추가하여 Service에 편입

```
[root@k8s-master ~]# kubectl  label  pod  svc-out-pod  service=nginx  --overwrite
pod/svc-out-pod labeled


[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME                              		READY   STATUS    RESTARTS   AGE    LABELS
deploy-web-dep-578859465c-rfghn   	1/1         Running     0                42m    app=web-dep,pod-template-hash=578859465c,service=nginx
deploy-web-dep-578859465c-vfblj   	1/1         Running     0                42m    app=web-dep,pod-template-hash=578859465c,service=nginx
svc-in-pod                        		1/1         Running     0                35m    service=nginx
svc-out-pod                       		1/1         Running     0                8m7s   service=nginx


[root@k8s-master ~]# kubectl  describe  service  clusterip-service
Name:                     	clusterip-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	service=nginx
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:             	IPv4
IP:                       	10.100.100.100
IPs:                      	10.100.100.100
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.96:80,10.244.1.21:80,10.244.2.97:80 + 1 more...
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                   	<none>
```

label을 `service=nginx`로 덮어쓰자 `svc-out-pod`도 즉시 Endpoints에 추가된다(`+ 1 more...`).

### 실습 리소스 정리

```
[root@k8s-master ~]# kubectl  delete  deployments  deploy-web-dep
deployment.apps "deploy-web-dep" deleted from default namespace


[root@k8s-master ~]# kubectl  delete  service  clusterip-service
service "clusterip-service" deleted from default namespace


[root@k8s-master ~]# kubectl  delete  pod svc-in-pod
pod "svc-in-pod" deleted from default namespace


[root@k8s-master ~]# kubectl  delete  pod svc-out-pod
pod "svc-out-pod" deleted from default namespace
```

---

> 📌 **핵심 요약**
> - Service는 계속 바뀌는 Pod IP 문제를 해결하기 위해 label selector로 Pod들을 묶어 고정된 단일 진입점(IP/이름)과 로드밸런싱을 제공하는 객체
> - Service는 Pod를 직접 생성하지 않고 label selector로만 대상을 선택하므로, Pod 이름·IP를 몰라도 label만 맞으면 자동으로 연결된다
> - Service Type은 ClusterIP·NodePort·LoadBalancer·ExternalName·Headless 5가지가 있으며, ClusterIP는 클러스터 내부 전용 기본 타입이다
> - ClusterIP Service는 동일 label의 Pod 그룹에 단일 Virtual IP를 부여하고 요청마다 다른 Pod로 자동 로드밸런싱하며, 새 Pod나 label이 일치하는 Pod는 즉시 Endpoints에 자동 추가된다
> - 관련: 2. 📦 Kubernetes - Pod 생성 · 10. 🎛️ Kubernetes - Controller 개념과 ReplicationController · 19. 🚪 Kubernetes - NodePort·LoadBalancer
