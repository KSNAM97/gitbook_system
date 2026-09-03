# Kubernetes - host/path 기반 Ingress

> **Tag:** #Kubernetes #Ingress #hostPath
> **핵심 요약:** host/path 기반 Ingress 실습 — 메인/커리큘럼/강좌/로그인 서비스를 순차적으로 추가하며 Ingress 라우팅 규칙과 롤링 업데이트를 구성

---

## 1. host/path 기반 Ingress 실습

### Deployment/Service YAML 확인

```
[root@k8s-master sol-collection]# ls  -l  /root/webserver-demo/ingress/
합계 12
-rw-r--r-- 1 root root 528 12월 19  2025 curriculum.yaml
-rw-r--r-- 1 root root 523 12월 19  2025 ingress.yaml
-rw-r--r-- 1 root root 514 12월 19  2025 sol-home.yaml
```

**sol-home.yaml — 메인 페이지 Deployment/Service**

```
[root@k8s-master sol-collection]# cat  /root/webserver-demo/ingress/sol-home.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sol-home-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sol-home

  template:
    metadata:
      labels:
        app: sol-home
    spec:
      containers:
        - name: nginx
          image: konan7979/sol-collection:1.0	# 개별 이미지로 수정
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: sol-home-service
spec:
  type: ClusterIP
  selector:
    app: sol-home
  ports:
    - port: 80
      targetPort: 80
```

**curriculum.yaml — 커리큘럼 페이지 Deployment/Service**

```
[root@k8s-master sol-collection]# cat  /root/webserver-demo/ingress/curriculum.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curriculum-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: curriculum
  template:
    metadata:
      labels:
        app: curriculum
    spec:
      containers:
        - name: nginx
          image: konan7979/curriculum-service:1.0
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: curriculum-service
spec:
  type: ClusterIP
  selector:
    app: curriculum
  ports:
    - port: 80
      targetPort: 80
```

**ingress.yaml — 초기 Ingress 규칙**

```
[root@k8s-master sol-collection]# cat  /root/webserver-demo/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress                      		# 리소스 종류: Ingress
metadata:
  name: sol-ingress                	# Ingress 이름

spec:
  ingressClassName: nginx		# nginx Ingress Controller 사용
  rules:
# -host: www.soldesk.com # Domain이 있는경우 Domain 설정
    - http:                        		# HTTP 요청에 대한 라우팅 규칙 설정
        paths:
          - path: /          		# / 로 들어오는 요청 처리 (: http://도메인/)
            pathType: Prefix       	# / 로 시작하는 모든 경로를 매칭
            backend:
              service:
                name: sol-home-service	# 요청을 전달할 Service 이름
                port:
                  number: 80             	# Service의 80번 포트로 전달

          - path: /curriculum		# /curriculum 으로 들어오는 요청 처리 (예: http://도메인/curriculum)
            pathType: Prefix       	# /curriculum으로 시작하는 경로를 매칭
            backend:
              service:
                name: curriculum-service	# curriculum Service로 전달
                port:
                  number: 80              	# Service의 80번 포트로 전달
```

### Deployment, Service 생성

```
[root@k8s-master ~]# kubectl apply -f /root/webserver-demo/ingress/sol-home.yaml
deployment.apps/sol-home-deploy created
service/sol-home-service created

[root@k8s-master sol-collection]# kubectl  get  deployments
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
sol-home-deploy   1/1     1            1           13s

[root@k8s-master sol-collection]# kubectl  get  pods
NAME                                         READY   STATUS    RESTARTS   AGE
sol-home-deploy-745b565968-rl9k7   1/1         Running     0                98s

[root@k8s-master sol-collection]# kubectl  get  service  sol-home-service
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
sol-home-service   ClusterIP   10.97.125.105     <none>             80/TCP   
```

**sol-home-service 서비스의 endPoint 확인**

```
[root@k8s-master sol-collection]# kubectl  get  endpoints  sol-home-service
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME               	ENDPOINTS	AGE
sol-home-service	10.244.2.4:80	9m34s
```

### Ingress 생성

```
[root@k8s-master ~]# kubectl apply -f /root/webserver-demo/ingress/ingress.yaml
ingress.networking.k8s.io/sol-ingress created

[root@k8s-master sol-collection]# kubectl  get  ingress
NAME          CLASS   HOSTS   ADDRESS          PORTS   AGE
sol-ingress    nginx     *       192.168.10.101   80      16s

[root@k8s-master sol-collection]# kubectl  get  service  --namespace  ingress-nginx
NAME                                 	TYPE    	   CLUSTER-IP   	EXTERNAL-IP   PORT(S)                      	AGE
ingress-nginx-controller             	NodePort	   10.105.3.121     	<none>            80:32181/TCP,443:30366/TCP   	86m
ingress-nginx-controller-admission   	ClusterIP	   10.101.171.143	<none>            443/TCP                      	86m
```

Pod와 Service는 Ingress 뒤에 있는 내부 서비스이고, Ingress(정확히는 Ingress Controller)가 외부와 클러스터를 연결하는 관문이다.

```
https://192.168.10.100:30366/
```

### curriculum 서비스 추가

```
[root@k8s-master ~]# cd  /root/webserver-demo/curriculum/

[root@k8s-master curriculum]# ls  -l
합계 8
-rw-r--r-- 1 root root  125 12월 19  2025 Dockerfile
-rw-r--r-- 1 root root 3777 12월 19  2025 index.html

[root@k8s-master curriculum]# docker  build  -t  konan7979/curriculum-service:1.0  .

[root@k8s-master curriculum]# docker  images
IMAGE                               	ID             	DISK USAGE	CONTENT SIZE   EXTRA
custom-nginx-web:1.31               	02218a3d52d6        	348MB           	97MB
konan7979/curriculum-service:1.0    	2a2b49481bba        	276MB         	72.3MB
konan7979/custom-nginx-web:1.31     	02218a3d52d6        	348MB           	97MB
konan7979/nginx-exec-liveness:1.0   	caeb48d1c8b7        	235MB         	63.1MB
konan7979/nginx-liveness:1.0        	75edbab6ae2a        	235MB         	63.1MB
konan7979/sol-collection:1.0        	571a4a896a75        	276MB         	72.3MB
konan7979/ssh-probe:1.0             	6578cbc133e3        	321MB         	81.9MB

[root@k8s-master curriculum]# docker  push  konan7979/curriculum-service:1.0
The push refers to repository [docker.io/konan7979/curriculum-service]
a3362294a34d: Pushed
44136fa355b3: Mounted from konan7979/ssh-probe
375a694db734: Mounted from konan7979/sol-collection
5c32499ab806: Mounted from konan7979/sol-collection
5f825f15e2e0: Mounted from konan7979/sol-collection
16d05858bb8d: Mounted from konan7979/sol-collection
08cfef42fd24: Mounted from konan7979/sol-collection
3cc5fdd1317a: Mounted from konan7979/sol-collection
4f4e50e20765: Mounted from konan7979/sol-collection
24b091158190: Pushed
5f3c66393266: Pushed
1.0: digest: sha256:2a2b49481bba9e7db30203649a8bfaf804aed75d94697cca3350a33b164afe5f size: 856

[root@k8s-master curriculum]# kubectl  apply  -f  /root/webserver-demo/ingress/curriculum.yaml
deployment.apps/curriculum-deploy created
service/curriculum-service created

[root@k8s-master curriculum]# kubectl  get  deployments  curriculum-deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
curriculum-deploy   2/2     2            2           25s

[root@k8s-master curriculum]# kubectl  get  pods
NAME                                 	READY   STATUS    RESTARTS   AGE
curriculum-deploy-587bbcd4c5-5vct9   	1/1        Running     0                 6s
curriculum-deploy-587bbcd4c5-fqqsm   	1/1        Running     0                 6s
sol-home-deploy-745b565968-rl9k7     	1/1        Running     0                 46m

[root@k8s-master curriculum]# kubectl  get  service
NAME                 	TYPE        CLUSTER-IP      	EXTERNAL-IP   PORT(S)   AGE
curriculum-service	ClusterIP    10.110.9.135	<none>            80/TCP     81s
kubernetes       	ClusterIP    10.96.0.1       	<none>            443/TCP    12d
sol-home-service  	ClusterIP    10.97.125.105   	<none>            80/TCP     47m

[root@k8s-master curriculum]# kubectl  get  pods  -o wide
NAME                                 	READY   STATUS    RESTARTS   AGE    IP            NODE          NOMINATED NODE   READINESS GATES
curriculum-deploy-587bbcd4c5-5vct9	1/1        Running     0                 3m8s   10.244.2.5   k8s-worker2   <none>           <none>
curriculum-deploy-587bbcd4c5-fqqsm   	1/1        Running     0                 3m8s   10.244.1.5   k8s-worker1   <none>           <none>
sol-home-deploy-745b565968-rl9k7     	1/1        Running     0                 49m    10.244.2.4   k8s-worker2   <none>           <none>

[root@k8s-master curriculum]# kubectl  get  endpoints  curriculum-service
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                 	ENDPOINTS                      AGE
curriculum-service	10.244.1.5:80,10.244.2.5:80   2m30s

[root@k8s-master curriculum]# cat /root/webserver-demo/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sol-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sol-home-service
                port:
                  number: 80
          - path: /curriculum
            pathType: Prefix
            backend:
              service:
                name: curriculum-service
                port:
                  number: 80
```

### 강좌 페이지, 로그인/회원가입 페이지 추가 — 디렉터리 구성

```
[root@k8s-master ~]#  mkdir -p  /root/webserver-demo/class/html/ban
```

```
[root@k8s-master ~]# mkdir -p  /root/webserver-demo/auth/html/login
```

```
[root@k8s-master ~]# mkdir -p  /root/webserver-demo/service
```

```
[root@k8s-master ~]# mkdir -p  /root/webserver-demo/deploy
```

```
[root@k8s-master ~]# ls -l  webserver-demo/
합계 0
drwxr-xr-x 3 root root 18   8월 24 13:10 auth
drwxr-xr-x 3 root root 18   8월 24 13:06 class
drwxr-xr-x 2 root root 42 12월 19  2025 curriculum
drwxr-xr-x 2 root root   6  8월 24 13:09 deploy
drwxr-xr-x 2 root root 70 12월 19  2025 ingress
drwxr-xr-x 2 root root   6  8월 24 13:11 service
drwxr-xr-x 3 root root 36 12월 19  2025 sol-collection

[root@k8s-master ~]# tree  /root/webserver-demo/

/root/webserver-demo/
├── auth
│        └── html
│                   └── login
├── class
│        └── html
│                   └── ban
├── curriculum
│        ├── Dockerfile
│        └── index.html
├── deploy
├── ingress
│        ├── curriculum.yaml
│        ├── ingress.yaml
│        └── sol-home.yaml
├── service
└── sol-collection
           ├── Dockerfile
           └── html
                       ├── images
                       │        ├── sol_logo.jpg
                       │        └── soldesk.jpg
                       └── index.html
```

서비스 단위로 소스/도커파일을 분리한다.

- services/class : 강좌 페이지 전용(정적 HTML)
- services/auth : 로그인/회원가입 전용(정적 HTML)
- k8s : 쿠버네티스 배포 매니페스트(Deployment/Service/Ingress)
- konan7979/soldesk-main:1.0 : 메인 HTML만 포함
- konan7979/soldesk-auth:1.0 : 로그인/회원가입 HTML만 포함

**장점**

- 메인 화면만 수정해도 main 이미지만 다시 빌드/배포하면 됨(auth는 그대로)
- 트래픽/복제수(replica)를 서비스별로 다르게 운영 가능(메인은 2개, auth는 3개처럼)
- Ingress 경로 기반 라우팅( `/` 는 main, `/login` 은 auth )이 직관적으로 성립

### 강좌 페이지, 로그인 페이지, 회원가입 페이지 HTML 파일 생성

**강좌 페이지**

```
[root@k8s-master ~]# vi ./webserver-demo/class/html/ban/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>Soldesk Class</title>
</head>
<body>
  <h1>Soldesk Academy</h1>
  <p>강의 종류</p>
  <hr>

  <ul>
    <li><a href="/class/linux.html">리눅스</a></li>
    <li><a href="/class/network.html">네트워크</a></li>
    <li><a href="/class/docker.html">도커</a></li>
    <li><a href="/class/kubernetes.html">쿠버네티스</a></li>
    <li><a href="/class/ansible.html">앤서블</a></li>
    <li><a href="/class/terraform.html">테라폼</a></li>
    <li><a href="/class/aws.html">AWS</a></li>
    <li><a href="/class/elk.html">ELK</a></li>
    <li><a href="/class/project.html">통합 프로젝트</a></li>
  </ul>

  <hr>
  <button onclick="location.href='/'">메인으로</button>
</body>
</html>
```

```
[root@k8s-master ~]# ls  -l  ./webserver-demo/class/html/ban/
합계 4
-rw-r--r-- 1 root root 779  8월 24 14:36 index.html
```

**강좌 페이지를 서비스하기 위한 Dockerfile 생성**

```
[root@k8s-master ~]# cd  /root/webserver-demo/class/

[root@k8s-master class]# pwd
/root/webserver-demo/class

[root@k8s-master class]# ls  -lR
.:
합계 0
drwxr-xr-x 2 root root 24  8월 24 14:50 html

./html:
합계 4
-rw-r--r-- 1 root root 779  8월 24 14:48 index.html

   # Dockerfile 생성
[root@k8s-master class]# 	
FROM  nginx:1.29.1
COPY  html/   /usr/share/nginx/html/
EXPOSE 80

[root@k8s-master class]# ls  -lR
.:
합계 4
-rw-r--r-- 1 root root 66  8월 24 14:53 Dockerfile
drwxr-xr-x 2 root root 24  8월 24 14:50 html

./html:
합계 4
-rw-r--r-- 1 root root 779  8월 24 14:48 index.html

   # 이미지 생성
[root@k8s-master class]# docker  build  -t  konan7979/soldesk-class:1.0  .

   # 이미지 hub.docker.com에 PUSH
[root@k8s-master class]# docker  push  konan7979/soldesk-class:1.0
The push refers to repository [docker.io/konan7979/soldesk-class]
1dd9c66811ae: Pushed
44136fa355b3: Mounted from konan7979/ssh-probe
375a694db734: Mounted from konan7979/sol-collection
5c32499ab806: Mounted from konan7979/sol-collectionektl 
16d05858bb8d: Mounted from konan7979/sol-collection
5f825f15e2e0: Mounted from konan7979/sol-collection
08cfef42fd24: Mounted from konan7979/sol-collection
3cc5fdd1317a: Mounted from konan7979/sol-collection
4f4e50e20765: Mounted from konan7979/sol-collection
dc7cb1c8cf65: Pushed
1.0: digest: sha256:d85e63c1753bfeefb177e3b975c4690b3573a63c49fddf2e569cfd4b23ac1f4b size: 856
```

**강좌 페이지 이미지를 사용한 Deployment와 Service YAML 파일 생성**

```
[root@k8s-master class]# vi  /root/webserver-demo/deploy/class-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: class-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: class
  template:
    metadata:
      labels:
        app: class
    spec:
      containers:
      - name: nginx
        image: konan7979/soldesk-class:1.0
        ports:
        - containerPort: 80
```

```
[root@k8s-master auth]# vi  /root/webserver-demo/deploy/class-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: class-service
spec:
  selector:
    app: class
  ports:
  - port: 80
    targetPort: 80
```

```
[root@k8s-master class]# ls  -l  /root/webserver-demo/deploy/
합계 8
-rw-r--r-- 1 root root 327  8월 24 15:06 class-deploy.yaml
-rw-r--r-- 1 root root 135  8월 24 15:06 class-svc.yaml
```

**강좌 페이지로 이동하기 위한 Path 설정**

```
[root@k8s-master class]# vi /root/webserver-demo/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sol-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:

          - path: /
            pathType: Prefix
            backend:
              service:
                name: sol-home-service
                port:
                  number: 80

          - path: /curriculum
            pathType: Prefix
            backend:
              service:
                name: curriculum-service
                port:
                  number: 80

          - path: /ban
            pathType: Prefix
            backend:
              service:
                name: class-service
                port:
                  number: 80
```

```
[root@k8s-master class]# ls -l  /root/webserver-demo/deploy/
합계 8
-rw-r--r-- 1 root root 327  8월 24 15:06 class-deploy.yaml		# 실행
-rw-r--r-- 1 root root 135  8월 24 15:06 class-svc.yaml		# 실행

[root@k8s-master class]# ls -l  /root/webserver-demo/ingress/
합계 12
-rw-r--r-- 1 root root 528 12월 19  2025 curriculum.yaml
-rw-r--r-- 1 root root 523 12월 19  2025 ingress.yaml		# 실행
-rw-r--r-- 1 root root 514 12월 19  2025 sol-home.yaml

[root@k8s-master class]# kubectl  apply  -f  /root/webserver-demo/deploy/class-deploy.yaml

[root@k8s-master class]# kubectl  apply  -f  /root/webserver-demo/deploy/class-svc.yaml

[root@k8s-master class]# kubectl  apply  -f  /root/webserver-demo/ingress/ingress.yaml

[root@k8s-master class]# kubectl  get  deployments  class-deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
class-deploy   2/2         2                    2                 70s

[root@k8s-master class]# kubectl  get  pods
NAME                                 	READY   STATUS    RESTARTS   AGE
class-deploy-5567486569-b62q5        	1/1     Running   0          5m36s
class-deploy-5567486569-qp6lf        	1/1     Running   0          5m36s
curriculum-deploy-587bbcd4c5-5vct9   	1/1     Running   0          161m
curriculum-deploy-587bbcd4c5-fqqsm   	1/1     Running   0          161m
sol-home-deploy-745b565968-rl9k7     	1/1     Running   0          3h27m

[root@k8s-master class]# kubectl  get  service  class-service
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
class-service   ClusterIP   10.97.151.80   <none>        80/TCP    76s

[root@k8s-master class]# kubectl  get  ingress  sol-ingress
NAME          CLASS   HOSTS   ADDRESS          PORTS   AGE
sol-ingress   nginx   *       192.168.10.101   80      3h18m

[root@k8s-master class]# kubectl  describe  ingress  sol-ingress
Name:       	sol-ingress
Labels:           	<none>
Namespace:        	default
Address:          	192.168.10.101
Ingress Class:    	nginx
Default backend:  	<default>
Rules:
  Host		Path  	Backends
  ----  		----  	--------
  *
         		/             	sol-home-service:80 (10.244.2.4:80)
          		/curriculum	curriculum-service:80 (10.244.1.5:80,10.244.2.5:80)
         		/ban        	class-service:80 (10.244.1.6:80,10.244.2.6:80)
Annotations:  	<none>
Events:
  Type    Reason  Age                    	  From                      	Message
  ----    ------  ----                   	  ----                      	-------
  Normal  Sync    2m30s (x3 over 3h19m)  nginx-ingress-controller  	Scheduled for sync
```

현재 `sol-collection/html/index.html` 코드에는 강좌 페이지로 이동할 방법이 없다. 강좌 페이지로 이동할 코드를 생성해야 이동이 가능하다.

**강좌로 이동하기 위해서 sol-collection 안의 html 코드 수정**

```
[root@k8s-master class]# ls -l  /root/webserver-demo/sol-collection/html/
합계 4
drwxr-xr-x 2 root root  45 12월 19  2025 images
-rw-r--r-- 1 root root 341  1월 13  2026 index.html
```

```
[root@k8s-master class]# vi  /root/webserver-demo/sol-collection/html/index.html
<html>
<head>
  <meta charset="UTF-8">
  <title>soldesk</title>
</head>
<body>
  <center>
    <img src="images/sol_logo.jpg"><br>
    <p style="color:red;">Soldesk Academy/AWS Cloud</p><br>
    <img src="images/soldesk.jpg"><br>
    <button onclick="location.href='/curriculum/index.html'">커리큘럼</button>
    <button onclick="location.href='/ban/index.html'">강좌 페이지</button>		# 버튼추가
  </center>
</body>
</html>
```

**변경사항을 반영하기 위한 롤링 업데이트**

```
[root@k8s-master class]# cd  /root/webserver-demo/sol-collection/

[root@k8s-master sol-collection]# pwd
/root/webserver-demo/sol-collection

[root@k8s-master sol-collection]# ls  -l
합계 4
-rw-r--r-- 1 root root 151 12월 19  2025 Dockerfile
drwxr-xr-x 3 root root  38   8월 24 16:21 html

    # sol-collection:1.0  -->  sol-collection:1.1 로 버전업
[root@k8s-master sol-collection]# docker  build  -t  konan7979/sol-collection:1.1  .

[root@k8s-master sol-collection]# docker push konan7979/sol-collection:1.1
The push refers to repository [docker.io/konan7979/sol-collection]
762b6005bb6c: Pushed
44136fa355b3: Already exists
375a694db734: Layer already exists
5c32499ab806: Layer already exists
5f825f15e2e0: Layer already exists
16d05858bb8d: Layer already exists
08cfef42fd24: Layer already exists
3cc5fdd1317a: Layer already exists
4f4e50e20765: Layer already exists
9a6d0ff9649d: Pushed
1.1: digest: sha256:50e9203d47063d65fb5e293c35d665f891aa3293ba52c71d5ef3f7ea7174d366 size: 856

    # 롤링 업데이트
[root@k8s-master sol-collection]# kubectl  set  image  deployments sol-home-deploy  nginx=konan7979/sol-collection:1.1
deployment.apps/sol-home-deploy image updated

    # 버전 관리
[root@k8s-master sol-collection]# kubectl annotate deployment  sol-home-deploy  \
> kubernetes.io/change-cause="rev2: sol-collection:1.0 -> sol-collection:1.1" --overwrite
deployment.apps/sol-home-deploy annotated

[root@k8s-master sol-collection]# kubectl  rollout  history  deployment  sol-home-deploy
deployment.apps/sol-home-deploy
REVISION  CHANGE-CAUSE
1         	  <none>
2              rev2: sol-collection:1.0 -> sol-collection:1.1
```

### 로그인/회원가입 페이지 추가

**로그인 페이지**

```
[root@k8s-master ~]# vi  ./webserver-demo/auth/html/login/login.html
<!doctype html>
<html lang="ko">
<head>
  <title>Login</title>
  <meta charset="utf-8">
</head>
<body>
  <h1>로그인 페이지</h1>

  <form>
    ID: <input type="text"><br><br>
    PW: <input type="password"><br><br>
    <input type="submit" value="로그인">
  </form>

  <hr>
  <a href="/">메인으로</a>
</body>
</html>
```

**회원가입 페이지**

```
[root@k8s-master ~]# vi  ./webserver-demo/auth/html/login/signup.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>Signup</title>
</head>
<body>
  <h1>회원가입 페이지</h1>

  <form>
    ID: <input type="text"><br><br>
    PW: <input type="password"><br><br>
    Email: <input type="email"><br><br>
    <input type="submit" value="회원가입">
  </form>

  <hr>
  <a href="/">메인으로</a>
</body>
</html>
```

```
[root@k8s-master class]# ls  -l  /webserver-demo/auth/html/login
합계 8
-rw-r--r-- 1 root root 329  1월 13 23:28 login.html
-rw-r--r-- 1 root root 376  1월 13 23:28 signup.html
```

**로그인, 회원가입용 이미지 생성**

```
[root@k8s-master class]# cd   /root/webserver-demo/auth/

[root@k8s-master auth]# pwd
/root/webserver-demo/auth

[root@k8s-master soldesk-ingress]# vi Dockerfile
FROM nginx:1.29.1
COPY html/ /usr/share/nginx/html/
EXPOSE 80

[root@k8s-master auth]# ls -lR
.:
합계 4
-rw-r--r-- 1 root root 62  8월 24 17:12 Dockerfile
drwxr-xr-x 3 root root 19  8월 24 13:10 html

./html:
합계 0
drwxr-xr-x 2 root root 43  8월 24 14:39 login

./html/login:
합계 8
-rw-r--r-- 1 root root 329  8월 24 14:38 login.html
-rw-r--r-- 1 root root 376  8월 24 14:39 signup.html

    # 회원가입, 로그인을 서비스하기위한 이미지 생성
[root@k8s-master auth]# docker  build  -t  konan7979/soldesk-auth:1.0 .

[root@k8s-master auth]# docker images
IMAGE                              		ID             	DISK USAGE   CONTENT SIZE   EXTRA
konan7979/curriculum-service:1.0   	2a2b49481bba        276MB           72.3MB
konan7979/sol-collection:1.0       	571a4a896a75        276MB           72.3MB
konan7979/sol-collection:1.1       	50e9203d4706        276MB           72.3MB
konan7979/soldesk-auth:1.0         	85d142e226fd        276MB           72.3MB
konan7979/soldesk-class:1.0        	4c3e945db06d        276MB           72.3MB
konan7979/soldesk-class:1.1        	3191088fda2f       276MB           72.3MB

[root@k8s-master auth]# docker push konan7979/soldesk-auth:1.0
```

**회원가입, 로그인을 서비스하기 위한 Deployment/Service 생성**

```
[root@k8s-master auth]# vi  /root/webserver-demo/deploy/auth-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
      - name: nginx
        image: konan7979/soldesk-auth:1.0
        ports:
        - containerPort: 80
```

```
[root@k8s-master auth]# ls  -l   /root/webserver-demo/deploy/
합계 12
-rw-r--r-- 1 root root 323  8월 24 17:20 auth-deploy.yaml
-rw-r--r-- 1 root root 327  8월 24 17:05 class-deploy.yaml
-rw-r--r-- 1 root root 135  8월 24 15:06 class-svc.yaml
```

```
[root@k8s-master auth]# vi  /root/webserver-demo/service/auth-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth
  ports:
  - port: 80
    targetPort: 80
```

```
[root@k8s-master auth]# ls  -l  /root/webserver-demo/service/
합계 8
-rw-r--r-- 1 root root 133  8월 24 17:22 auth-svc.yaml
-rw-r--r-- 1 root root 135  8월 24 17:21 class-svc.yaml
```

**Ingress에 /login 경로 추가**

```
[root@k8s-master auth]# vi  /root/webserver-demo/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sol-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sol-home-service
                port:
                  number: 80
          - path: /curriculum
            pathType: Prefix
            backend:
              service:
                name: curriculum-service
                port:
                  number: 80

          - path: /ban
            pathType: Prefix
            backend:
              service:
                name: class-service
                port:
                  number: 80

          - path: /login
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 80
```

```
    # 로그인, 회원가입 Deployment 실행
[root@k8s-master auth]# kubectl  apply  -f  /root/webserver-demo/deploy/auth-deploy.yaml
deployment.apps/auth-deploy created

    # 로그인, 회원가입 Service 실행
[root@k8s-master auth]# kubectl  apply  -f  /root/webserver-demo/service/auth-svc.yaml
service/auth-service created

   # ingress적용을 위한 재시작
[root@k8s-master auth]# kubectl  apply  -f  /root/webserver-demo/ingress/ingress.yaml
ingress.networking.k8s.io/sol-ingress configured

[root@k8s-master auth]# kubectl  get  deployments  auth-deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
auth-deploy   3/3     3            3           73s

[root@k8s-master auth]# kubectl  get  pods
NAME                                 READY   STATUS    RESTARTS   AGE
auth-deploy-555dd97c97-2d57n         1/1     Running   0          88s
auth-deploy-555dd97c97-f5cdv         1/1     Running   0          88s
auth-deploy-555dd97c97-vz62h         1/1     Running   0          88s
class-deploy-84cc5b8786-bsw9s        1/1     Running   0          21m
class-deploy-84cc5b8786-scrlm        1/1     Running   0          21m
curriculum-deploy-587bbcd4c5-5vct9   1/1     Running   0          4h48m
curriculum-deploy-587bbcd4c5-fqqsm   1/1     Running   0          4h48m
sol-home-deploy-66bfd9c66c-q6cbx     1/1     Running   0          57m

[root@k8s-master auth]# kubectl  get  service
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
auth-service         ClusterIP   10.107.138.137   <none>        80/TCP    71s
class-service        ClusterIP   10.97.151.80     <none>        80/TCP    132m
curriculum-service   ClusterIP   10.110.9.135     <none>        80/TCP    4h48m
kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP   13d
sol-home-service     ClusterIP   10.102.49.106    <none>        80/TCP    68m

    # 외부포트 확인
[root@k8s-master auth]# kubectl  get  svc  -A | grep ingress
ingress-nginx   ingress-nginx-controller             	   NodePort    10.105.3.121     <none>        80:32181/TCP,443:30366/TCP   6h49m
ingress-nginx   ingress-nginx-controller-admission   ClusterIP    10.101.171.143   <none>        443/TCP                      6h49m

[root@k8s-master auth]# kubectl  describe  ingress
Name:       	sol-ingress
Labels:        	<none>
Namespace:        	default
Address:          	92.168.10.101
Ingress Class:    	nginx
Default backend:  	<default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
            	/             	sol-home-service:80 (10.244.1.8:80)
           	/curriculum   	curriculum-service:80 (10.244.1.5:80,10.244.2.5:80)
            	/ban          	class-service:80 (10.244.1.12:80,10.244.2.12:80)
           	/login        	auth-service:80 (10.244.2.13:80,10.244.1.13:80,10.244.2.14:80)
Annotations:  <none>
Events:
  Type    Reason  Age                    From                      Message
  ----    ------  ----                   ----                      -------
  Normal  Sync    4m43s (x5 over 5h32m)  nginx-ingress-controller  Scheduled for sync

[root@k8s-master auth]# kubectl  get  pods  -o  wide
NAME                                 	READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
auth-deploy-555dd97c97-2d57n         	1/1     Running   0          6m38s   10.244.2.13   k8s-worker2   <none>           <none>
auth-deploy-555dd97c97-f5cdv         	1/1     Running   0          6m38s   10.244.1.13   k8s-worker1   <none>           <none>
auth-deploy-555dd97c97-vz62h         	1/1     Running   0          6m38s   10.244.2.14   k8s-worker2   <none>           <none>
class-deploy-84cc5b8786-bsw9s        	1/1     Running   0          27m     10.244.2.12   k8s-worker2   <none>           <none>
class-deploy-84cc5b8786-scrlm        	1/1     Running   0          27m     10.244.1.12   k8s-worker1   <none>           <none>
curriculum-deploy-587bbcd4c5-5vct9 	1/1     Running   0          4h53m   10.244.2.5    k8s-worker2   <none>           <none>
curriculum-deploy-587bbcd4c5-fqqsm   	1/1     Running   0          4h53m   10.244.1.5    k8s-worker1   <none>           <none>
sol-home-deploy-66bfd9c66c-q6cbx     	1/1     Running   0          62m     10.244.1.8    k8s-worker1   <none>           <none>
```

로그인, 회원가입이 추가되었으나 main 페이지에서 로그인, 회원가입으로 이동할 기능이 없다. 이동할 기능을 추가하기 위해서 html 코드를 수정 후 롤링 업데이트를 실시한다.

```
[root@k8s-master ingress]# vi  /root/webserver-demo/sol-collection/html/index.html
<html>
<head>
  <meta charset="UTF-8">
  <title>soldesk</title>
</head>
<body>
  <center>
    <img src="images/sol_logo.jpg"><br>
    <p style="color:red;">Soldesk Academy/AWS Cloud</p><br>
    <img src="images/soldesk.jpg"><br>

    <button onclick="location.href='/curriculum/index.html'">커리큘럼</button>

    <br><br>							# 추가 설정
    <button onclick="location.href='/login/login.html'">로그인</button>		# 추가 설정
    <button onclick="location.href='/login/signup.html'">회원가입</button>	# 추가 설정
  </center>

</body>
</html>
```

**변경사항을 반영하기 위한 롤링 업데이트**

```
[root@k8s-master ingress]# cd  /root/webserver-demo/sol-collection

[root@k8s-master sol-collection]# ls  -l
합계 4
-rw-r--r-- 1 root root 151 12월 19  2025 Dockerfile
drwxr-xr-x 3 root root  38  8월 24 17:37 html

    # main 페이지 이미지 생성
[root@k8s-master sol-collection]# docker  build -t  konan7979/sol-collection:1.2  .

    # main 페이지 이미지 PUSH
[root@k8s-master sol-collection]# docker  push  konan7979/sol-collection:1.2

    # 롤링 업데이트
[root@k8s-master sol-collection]# kubectl  set  image  deployments  sol-home-deploy  nginx=konan7979/sol-collection:1.2
deployment.apps/sol-home`-deploy image updated

    # 롤링 업데이트 버전 관리
[root@k8s-master sol-collection]# kubectl annotate deployment  sol-home-deploy  \
kubernetes.io/change-cause="rev3: sol-collection:1.1 -> sol-collection:1.2" --overwrite
deployment.apps/sol-home-deploy annotated

    # 버전 확인
[root@k8s-master sol-collection]# kubectl  rollout  history  deployment  sol-home-deploy
deployment.apps/sol-home-deploy
REVISION  CHANGE-CAUSE
1              <none>
2              rev2: sol-collection:1.0 -> sol-collection:1.1
3              rev3: sol-collection:1.1 -> sol-collection:1.2
```

```
[root@k8s-master auth]# vi  /root/webserver-demo/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sol-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sol-home-service
                port:
                  number: 80
          - path: /curriculum
            pathType: Prefix
            backend:
              service:
                name: curriculum-service
                port:
                  number: 80

          - path: /ban
            pathType: Prefix
            backend:
              service:
                name: class-service
                port:
                  number: 80

          - path: /login		# /login/login.html
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 80
```

```
https://192.168.10.100:30366/login/login.html
```

---

## 2. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- Ingress를 생성해도 트래픽이 라우팅되지 않는다면, Ingress Controller(`ingress-nginx-controller`)가 정상적으로 떠 있는지 `kubectl get pods -n ingress-nginx`, `kubectl get svc -n ingress-nginx`로 먼저 확인한다. Ingress 리소스는 규칙 정의일 뿐, 실제 트래픽 처리는 Controller가 담당하기 때문이다.
- `kubectl get ingress`의 `ADDRESS` 컬럼이 비어 있으면 아직 Ingress Controller가 규칙을 반영하지 않은 상태다. `kubectl describe ingress <이름>`의 `Rules`와 `Events`를 확인해 백엔드 Service·포트가 올바른지, `Sync` 이벤트가 발생했는지 점검한다.
- `path: /login`처럼 `pathType: Prefix`만 사용하면 `/login/login.html`의 하위 경로 문자열이 그대로 백엔드로 전달되어 nginx 내부 파일 경로와 어긋날 수 있다. 이 문제는 정규표현식 기반 rewrite-target으로 해결하며, 자세한 내용은 "Kubernetes - 정규표현식 Ingress·Canary 배포" 문서에서 다룬다.
- Ingress는 Service까지만 연결하고 Pod를 직접 알지 못한다. 따라서 라우팅이 실패하면 Ingress → Service 연결(`backend.service.name`, `port.number`)과 Service → Pod 연결(Service의 `selector`와 Pod의 `labels` 일치 여부)을 순서대로 나눠서 점검해야 한다.
- `kubectl get endpoints <service>` 또는 `kubectl get endpointslices -l kubernetes.io/service-name=<service>`로 Service 뒤에 실제로 연결된 Pod IP 목록을 확인하면, Ingress→Service→Pod 경로 중 어느 구간에서 문제가 발생했는지 빠르게 좁힐 수 있다(단, v1 Endpoints API는 v1.33부터 Deprecated이므로 `discovery.k8s.io/v1 EndpointSlice` 사용을 권장한다는 경고가 함께 출력된다).

---

>  **핵심 요약**
> - Ingress 규칙(`rules.http.paths`)의 `path`와 `pathType: Prefix`로 도메인/경로 조건을 정의하고, `backend.service`로 매칭된 요청을 전달할 Service를 지정한다
> - 메인(`/`) → 커리큘럼(`/curriculum`) → 강좌(`/ban`) → 로그인(`/login`) 순서로 서비스를 추가할 때마다 Deployment/Service를 생성하고 Ingress `ingress.yaml`을 업데이트해 경로를 추가한다
> - 화면 간 이동(버튼 링크)이 필요하면 HTML을 수정한 뒤 새 이미지 태그로 빌드·push하고 `kubectl set image`로 롤링 업데이트하며, `kubectl annotate ... change-cause`로 배포 이력을 남긴다
> - `pathType: Prefix`만으로는 `/login/login.html` 같은 하위 경로가 그대로 백엔드에 전달되어 파일 경로가 꼬일 수 있어 정규표현식 기반 Ingress가 필요하다
> - 관련: 21.  Kubernetes - Ingress 기초와 준비 · 23.  Kubernetes - 정규표현식 Ingress·Canary 배포 · 18.  Kubernetes - Service 기초와 ClusterIP
