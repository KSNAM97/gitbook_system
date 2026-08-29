# 🌐 Kubernetes - 정규표현식 Ingress·Canary 배포

> **Tag:** #Kubernetes #Ingress #Regex #Canary #부트캠프
> **핵심 요약:** 정규표현식과 rewrite-target을 사용한 Ingress 구성, Canary Deployment 개념과 replicas 조정을 통한 점진적 배포 실습 정리

---

## 1. 🔤 정규표현식 기반 Ingress (rewrite-target)

`path: /login`처럼 접두사(Prefix)로만 매칭하면 `/login/login.html`처럼 하위 경로가 그대로 백엔드에 전달되어 파일 경로가 꼬일 수 있다. 이 문제를 해결하기 위해 정규표현식과 `rewrite-target` 애노테이션을 사용한 Ingress를 별도로 구성한다.

```
[root@k8s-master ~]# vi /root/webserver-demo/ingress/ingress-html.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: sol-ingress-html
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"    	# path에서 정규표현식 사용
    nginx.ingress.kubernetes.io/rewrite-target: /$2		# 두 번째 그룹($2)의 경로만 Backend로 전달

spec:
  ingressClassName: nginx                                 # nginx Ingress Controller 사용

  rules:
  - http:
      paths:

      - path: /curriculum(/|$)(.*)                       	# /curriculum 및 하위 경로 매칭
        pathType: ImplementationSpecific         	# Ingress Controller 방식으로 경로 해석
        backend:
          service:
            name: curriculum-service              	# curriculum-service로 전달
            port:
              number: 80                            	# Service의 80번 포트 사용

      - path: /login(/|$)(.*)                      	# /login 및 하위 경로 매칭
        pathType: ImplementationSpecific
        backend:
          service:
            name: auth-service                        	# auth-service로 전달
            port:
              number: 80

      - path: /class(/|$)(.*)                        	# /class 및 하위 경로 매칭
        pathType: ImplementationSpecific
        backend:
          service:
            name: class-service                      	# class-service로 전달
            port:
              number: 80

      - path: /()(.*)                                  	# 위 경로에 해당하지 않는 요청은 메인 서비스로 전달
        pathType: ImplementationSpecific
        backend:
          service:
            name: sol-home-service                  	# sol-home-service로 전달
            port:
              number: 80
```

**Ingress 정규표현식 경로 `/class(/|$)(.*)`가 무엇을 매칭할지 분해하면 다음과 같다.**

`(/|$)`

- `/` : 슬래시(/)
- `$` : 문자열의 끝
- 즉, `/class` 뒤에 슬래시(/)가 오거나 주소가 끝나는 경우를 허용한다.
  - `/class`
  - `/class/`
  - `/class/index.html/`

`(.*)`

- `.` : 아무 문자 1개
- `*` : 앞의 문자가 0개 이상
- `.*` : 아무 문자가 0개 이상
- `(.*)` : 뒤에 오는 모든 문자열을 하나의 그룹으로 묶는다.

**`rewrite-target: /$2`**

`rewrite-target: /$2`는 매칭된 것 중 무엇을 보낼지(치환)를 작성하는 설정이다.

| 예시 | 요청 | `$2` | rewrite 결과 | 백엔드 nginx 경로 |
|---|---|---|---|---|
| EX1 | `/curriculum` | `""` (빈 문자열) | `/` | `/usr/share/nginx/html/index.html` |
| EX2 | `/curriculum/index.html` | `index.html` | `/index.html` | `/usr/share/nginx/html/index.html` |
| EX3 | `/curriculum/css/style.css` | `css/style.css` | `/css/style.css` | `/usr/share/nginx/html/css/style.css` |

```
[root@k8s-master ~]# vi /root/webserver-demo/ingress/ingress-html.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sol-ingress-html
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2

spec:
  ingressClassName: nginx

  rules:
  - http:
      paths:

      - path: /curriculum(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: curriculum-service
            port:
              number: 80

      - path: /login(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: auth-service
            port:
              number: 80

      - path: /class(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: class-service
            port:
              number: 80

      - path: /()(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: sol-home-service
            port:
              number: 80
```

**정규표현식 Ingress 적용을 위해 각 서비스의 경로를 재구성**

전체 디렉터리 구조:

```
[root@k8s-master ~]# tree  /root/webserver-demo/
/root/webserver-demo/
├── auth
│       ├── Dockerfile
│       └── html
│                 └── login
│                            ├── login.html
│                            └── signup.html
├── class
│       ├── Dockerfile
│       └── html
│                  └── ban
│                            └── index.html
├── curriculum
│       ├── Dockerfile
│       └── index.html
├── deploy
│       ├── auth-deploy.yaml
│       ├── class-deploy.yaml
├── ingress
│       ├── curriculum.yaml
│       ├── ingress-html.yaml
│       ├── ingress.yaml
│       └── sol-home.yaml
├── service
│       ├── auth-svc.yaml
│       └── class-svc.yaml
└── sol-collection
          ├── Dockerfile
          └── html
                     ├── images
                     │   ├── sol_logo.jpg
                     │   └── soldesk.jpg
                     └── index.html
```

**로그인, 회원가입을 새로운 ingress에 맞게 경로 수정**

```
[root@k8s-master ~]# cd  /root/webserver-demo/auth/

[root@k8s-master auth]# ls -l
합계 4
-rw-r--r-- 1 root root 62  8월 24 17:12 Dockerfile
drwxr-xr-x 3 root root 19  8월 24 13:10 html

[root@k8s-master auth]# cp ./html/login/login.html   ./html/
[root@k8s-master auth]# cp ./html/login/signup.html ./html/

[root@k8s-master auth]# ls -l ./html/
합계 8
drwxr-xr-x 2 root root  43  8월 24 14:39 login
-rw-r--r-- 1 root root 329  8월 25 10:19 login.html
-rw-r--r-- 1 root root 376  8월 25 10:20 signup.html

[root@k8s-master auth]# rm  -rf ./html/login

[root@k8s-master auth]# ls -l ./html/
합계 8
-rw-r--r-- 1 root root 329  8월 25 10:19 login.html
-rw-r--r-- 1 root root 376  8월 25 10:20 signup.html
```

**강좌 페이지를 새로운 ingress에 맞게 경로 수정**

```
[root@k8s-master auth]# cd /root/webserver-demo/class/

[root@k8s-master class]# ls -l
합계 4
-rw-r--r-- 1 root root 66  8월 24 14:53 Dockerfile
drwxr-xr-x 3 root root 17  8월 24 15:58 html

[root@k8s-master class]# cp ./html/ban/index.html   ./html/

[root@k8s-master class]# rm  -rf  ./html/ban/

[root@k8s-master class]# ls -l  ./html/
합계 4
-rw-r--r-- 1 root root 779  8월 25 10:22 index.html
```

```
[root@k8s-master ~]# kubectl  get  pods
NAME                                 	READY   STATUS    RESTARTS   AGE
auth-deploy-555dd97c97-cgp25         	1/1         Running     0                109s		# 로그인 및 회원 가입 Pod
auth-deploy-555dd97c97-zhncw         	1/1         Running     0                109s		# 로그인 및 회원 가입 Pod
auth-deploy-555dd97c97-zjbzp         	1/1         Running     0                109s		# 로그인 및 회원 가입 Pod
class-deploy-84cc5b8786-4rhwv        	1/1         Running     0                100s		# 강좌 Pod
class-deploy-84cc5b8786-b4xsx        	1/1         Running     0                100s		# 강좌 Pod
curriculum-deploy-587bbcd4c5-4gnhd   	1/1         Running     0                2m21s	# 커리큘럼 Pod
curriculum-deploy-587bbcd4c5-gz9cs   	1/1         Running     0                2m21s	# 커리큘럼 Pod
sol-home-deploy-745b565968-b67vc     	1/1         Running     0                2m25s	# 메인 페이지 Pod
```

**롤링 업데이트 전 로그인 및 회원가입 Pod의 경로를 확인**

```
[root@k8s-master ~]# kubectl  exec  -it  auth-deploy-555dd97c97-cgp25  -- /bin/bash
root@auth-deploy-555dd97c97-cgp25:/#

root@auth-deploy-555dd97c97-cgp25:/# ls  -l  /usr/share/nginx/html/
total 8
-rw-r--r-- 1 root root 497 Aug 13  2025 50x.html
-rw-r--r-- 1 root root 615 Aug 13  2025 index.html
drwxr-xr-x 2 root root  43 Aug 24 05:39 login		# login 디렉터리 확인

root@auth-deploy-555dd97c97-cgp25:/# ls  -l  /usr/share/nginx/html/login/
total 8	
-rw-r--r-- 1 root root 329 Aug 24 05:38 login.html		# 로그인 페이지
-rw-r--r-- 1 root root 376 Aug 24 05:39 signup.html	# 회원 가입 페이지
```

**로그인, 회원가입 Pod 롤링 업데이트**

```
[root@k8s-master ~]# cd  /root/webserver-demo/auth/

[root@k8s-master auth]# ls  -l
합계 4
-rw-r--r-- 1 root root 62  8월 24 17:12 Dockerfile
drwxr-xr-x 2 root root 43  8월 25 10:20 html

    # 로그인, 회원가입 이미지 생성
[root@k8s-master auth]# docker  build  -t  konan7979/soldesk-auth:1.1  .

    # 로그인, 회원가입 이미지 확인
[root@k8s-master auth]# docker  images
IMAGE                              		ID             	DISK USAGE   CONTENT SIZE   EXTRA
konan7979/curriculum-service:1.0 	2a2b49481bba        276MB         72.3MB
konan7979/sol-collection:1.0       	571a4a896a75        276MB         72.3MB
konan7979/sol-collection:1.1       	50e9203d4706        276MB         72.3MB
konan7979/sol-collection:1.2       	6f9c8602891e        276MB         72.3MB
konan7979/soldesk-auth:1.0         	85d142e226fd        276MB         72.3MB
konan7979/soldesk-auth:1.1         	c2d9d04b0a87        276MB         72.3MB
konan7979/soldesk-class:1.0        	4c3e945db06d        276MB         72.3MB
konan7979/soldesk-class:1.1        	3191088fda2f        276MB         72.3MB

    # 로그인, 회원가입 이미지 PUSH
[root@k8s-master auth]# docker  push  konan7979/soldesk-auth:1.1
The push refers to repository [docker.io/konan7979/soldesk-auth]
077cf53213bf: Pushed
44136fa355b3: Already exists
375a694db734: Layer already exists
5c32499ab806: Layer already exists
5f825f15e2e0: Layer already exists
16d05858bb8d: Layer already exists
08cfef42fd24: Layer already exists
4f4e50e20765: Layer already exists
3cc5fdd1317a: Layer already exists
3c388c881ddd: Pushed
1.1: digest: sha256:c2d9d04b0a870716404c1da825c46a99735c033c9144389940f2551d0b05bc64 size: 856

    # 로그인, 회원가입 Deployment 롤링 업데이트
[root@k8s-master auth]# kubectl  set  image  deployments  auth-deploy  nginx=konan7979/soldesk-auth:1.1
deployment.apps/auth-deploy image updated

    # 버전 관리
[root@k8s-master auth]# kubectl annotate deployment  auth-deploy  \
 kubernetes.io/change-cause="rev2: soldesk-auth:1.0 -> soldesk-auth:1.1 Version Update" --overwrite
deployment.apps/auth-deploy annotated

[root@k8s-master auth]# kubectl  rollout  history  deployment  auth-deploy
deployment.apps/auth-deploy
REVISION  CHANGE-CAUSE
1         <none>
2         rev2: soldesk-auth:1.0 -> soldesk-auth:1.1 Version Update

    # 새로 만든 Ingress 실행
[root@k8s-master auth]# kubectl  apply  -f  /root/webserver-demo/ingress/ingress-html.yaml
ingress.networking.k8s.io/sol-ingress-html created
```

```
https://192.168.10.100:30366/
```

---

## 2. 🐤 Canary Deployment (카나리 배포)

### 개념

Canary Deployment(카나리 배포)는 새로운 버전을 전체 사용자에게 한 번에 적용하지 않고, 일부 Pod 또는 일부 사용자에게만 먼저 적용하여 안정성을 확인한 뒤 점진적으로 확대하는 배포 방식이다.

1. 기존 버전 전체 운영
2. 신규 버전 소수만 배포
3. 문제 없는지 확인
4. 신규 버전 비율 확대
5. 최종적으로 전체 신규 버전 전환

**왜 Canary라고 부르는가?**

과거 광산에서는 유독가스를 감지하기 위해 카나리아 새를 먼저 광산 안으로 데리고 들어갔다. 카나리아가 이상 반응을 보이면 사람이 위험을 미리 알 수 있었다. 소프트웨어 배포도 이와 비슷하게, 신규 버전을 일부에 먼저 적용 후 문제가 있는지 확인하고 문제가 없으면 전체 적용하기 때문에 Canary Deployment라고 부른다.

**기존 배포 방식의 문제점**

예를 들어 현재 웹 서비스가 1.0 버전으로 운영되고 있다고 가정한다.

현재 운영: `web:1.0`, `web:1.0`, `web:1.0`

새로운 2.0 버전을 바로 전체 배포하면: `web:2.0`, `web:2.0`, `web:2.0`

만약 2.0에 치명적인 오류가 있다면 전체 사용자에게 동시에 장애가 발생할 수 있다. Canary Deployment는 이를 방지하기 위해 일부만 먼저 변경한다.

```
web:1.0
web:1.0
web:1.0
web:2.0   <-- Canary
```

이 상태에서 2.0의 동작을 확인한다.

**Canary Deployment의 기본 구조**

예를 들어 총 10개의 Pod가 운영 중이라고 가정한다.

- 기존 버전: `v1 v1 v1 v1 v1 v1 v1 v1 v1 v1`
- 새 버전 v2를 1개만 먼저 배포: `v1 v1 v1 v1 v1 v1 v1 v1 v1 v2(Canary)`

대략 다음과 같은 비율이 된다. Canary v1 : 90%, Canary v2 : 10%. 문제가 없다면 점차 확대한다.

| 단계 | Canary v1 | Canary v2 |
|---|---|---|
| 1단계 | 90% | 10% |
| 2단계 | 70% | 30% |
| 3단계 | 50% | 50% |
| 최종 | 0% | 100% |

**Kubernetes에서 Canary Deployment를 만드는 기본 원리**

K8s에서는 일반적으로 서로 다른 버전의 Deployment를 동시에 실행하여 Canary Deployment를 구현할 수 있다.

- Deployment 1: `web-v1`, replicas: 9, image: `web:1.0`
- Deployment 2: `web-v2`, replicas: 1, image: `web:2.0`

두 Deployment의 Pod가 동일한 Service에 포함되도록 Label을 구성한다.

- Deployment 1의 web-v1 Pod: `app=web`, `version=v1`
- Deployment 2의 web-v2 Pod: `app=web`, `version=v2`

Service는 다음 Label만 선택한다.

```yaml
selector:
  app: web
```

그러면 Service 입장에서는 `app=web, version=v1`(3개)과 `app=web, version=v2`(1개) 모두 자신의 Backend Pod가 된다.

**Service와 Canary Deployment 관계**

Canary Deployment에서 중요한 부분은 Service Selector이다.

- 기존 Pod: `app=web`, `version=v1`
- Canary Pod: `app=web`, `version=v2`

Service가 다음과 같이 설정되어 있다면:

```yaml
selector:
  app: web
```

`version`은 확인하지 않기 때문에 v1과 v2 모두 Service에 포함된다.

```
구조:
                         Service
                            |
                  selector: app=web
                            |
           +-----------+------------+
           |                |                 |
       v1 Pod        v1 Pod           v2 Pod
```

따라서 사용자 요청이 기존 버전과 신규 버전으로 분산될 수 있다.

| 구분 | Rolling Update | Canary Deployment |
|---|---|---|
| 목적 | 순차적으로 새 버전 교체 | 새 버전을 일부에 먼저 테스트 |
| 기존/신규 버전 동시 운영 | 일시적으로 존재 | 의도적으로 일정 기간 함께 운영 |
| 트래픽 검증 | 주 목적이 아님 | 핵심 목적 |
| 장애 영향 최소화 | 가능 | 더욱 세밀하게 가능 |
| 신규 버전 테스트 | 제한적 | 실제 트래픽으로 검증 가능 |

### Canary Deployment 실습

기존 버전 v1과 신규 버전 v2 Deployment를 각각 생성한다. v1과 v2는 서로 다른 HTML 화면을 사용한다. 두 Deployment는 version 라벨로 서로 구분한다. Service는 공통 라벨 `app=web`만 선택하여 v1과 v2를 모두 서비스에 포함한다. 반복 요청을 통해 기존 버전과 Canary 버전으로 요청이 분산되는 것을 확인한다.

다음 조건으로 Canary Deployment 환경을 구성한다.

| 항목 | 기존 버전 | 신규 Canary 버전 |
|---|---|---|
| Deployment 이름 | web-v1 | web-v2 |
| replicas | 5 | 1 |
| image | nginx:1.31 | nginx:1.31 |
| label | app=web, version=v1 | app=web, version=v2 |
| HTML | VERSION V1 | VERSION V2 - CANARY |

Service 조건: `web-service`, selector `app=web`, port 80, targetPort 80.

**STEP 1) v1 HTML, v2 HTML 작성**

```
[root@k8s-master ~]# mkdir  -p  /canary/v1

[root@k8s-master ~]# vi  /canary/v1/index.html
<!DOCTYPE html>
<html>
<head>
  <title>Canary Test</title>
</head>
<body>
  <h1>Canary Version V1</h1>
  <p>Stable Version</p>
</body>
</html>

[root@k8s-master ~]# mkdir  -p  /canary/v2

[root@k8s-master ~]# vi  /canary/v2/index.html
<!DOCTYPE html>
<html>
<head>
  <title>Canary Test</title>
</head>
<body>
  <h1>Canary Version V2</h1>
  <p>Stable Version</p>
</body>
</html>

[root@k8s-master ~]# ls  -l  /canary/
합계 0
drwxr-xr-x 2 root root 24  8월 25 12:38 v1
drwxr-xr-x 2 root root 24  8월 25 12:39 v2
[root@k8s-master ~]# ls  -lR  /canary/
/canary/:
합계 0
drwxr-xr-x 2 root root 24  8월 25 12:38 v1
drwxr-xr-x 2 root root 24  8월 25 12:39 v2

/canary/v1:
합계 4
-rw-r--r-- 1 root root 143  8월 25 12:38 index.html

/canary/v2:
합계 4
-rw-r--r-- 1 root root 143  8월 25 12:39 index.html
```

**STEP 2) v1 이미지 생성**

```
# Dockerfile 생성
[root@k8s-master ~]# vi  /canary/v1/Dockerfile
FROM nginx:1.31
COPY index.html /usr/share/nginx/html/index.html

[root@k8s-master ~]# cd  /canary/v1

[root@k8s-master v1]# ls  -l
합계 8
-rw-r--r-- 1 root root  65  8월 25 12:42 Dockerfile
-rw-r--r-- 1 root root 143  8월 25 12:38 index.html

# Canary web1  이미지 생성
[root@k8s-master v1]# docker  build  -t  konan7979/canary-web:v1.0  .

[root@k8s-master v1]# docker  images | grep canary
konan7979/canary-web:v1.0          e098fee9e533        236MB         63.3MB

Docker Hub에 업로드
[root@k8s-master v2]# docker  push  konan7979/canary-web:v1.0
```

**STEP 3) v2 이미지 생성**

```
# Dockerfile 생성
[root@k8s-master ~]# vi  /canary/v2/Dockerfile
FROM nginx:1.31
COPY index.html /usr/share/nginx/html/index.html

[root@k8s-master v1]# cd /canary/v2

# Canary web2  이미지 생성
[root@k8s-master v2]# docker build  -t konan7979/canary-web:v2.0  .

[root@k8s-master v1]# docker  images | grep canary
konan7979/canary-web:v1.0          e098fee9e533        236MB         63.3MB
konan7979/canary-web:v2.0          18d7f06eb446        236MB         63.3MB

Docker Hub에 업로드
[root@k8s-master v2]# docker push konan7979/canary-web:v2.0
```

**STEP 4) Canary v1 Deployment 생성**

```
[root@k8s-master ~]# cd ~

[root@k8s-master ~]# vi canary-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v1
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web
      version: v1

  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
      - name: nginx
        image: knan7979/canary-web:v1.0
        ports:
        - containerPort: 80

[root@k8s-master ~]# kubectl  apply -f  canary-v1.yaml
deployment.apps/web-v1 created

[root@k8s-master ~]# kubectl  get  deployments  web-v1
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
web-v1    5/5        5                     5                13s

[root@k8s-master ~]# kubectl  get  pods
NAME                      	READY   STATUS    RESTARTS   AGE
web-v1-5dbbd65577-4cw9l   	1/1     Running   0          2s
web-v1-5dbbd65577-7dm44   	1/1     Running   0          2s
web-v1-5dbbd65577-85gcp   	1/1     Running   0          2s
web-v1-5dbbd65577-mk825   	1/1     Running   0          2s
web-v1-5dbbd65577-w22p6   	1/1     Running   0          2s
```

**STEP 5) Canary v2 Deployment 생성**

```
[root@k8s-master ~]# vi canary-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v2
spec:
  replicas: 1

  selector:
    matchLabels:
      app: web
      version: v2

  template:
    metadata:
      labels:
        app: web
        version: v2
    spec:
      containers:
      - name: nginx
        image: konan7979/canary-web:v2.0
        ports:
        - containerPort: 80

[root@k8s-master ~]# kubectl  apply -f  canary-v2.yaml
deployment.apps/web-v2 created

[root@k8s-master ~]#  kubectl  get  deployments  web-v2
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
web-v2    1/1        1                    1                  12s

[root@k8s-master ~]#  kubectl  get  pods
NAME                      	READY   STATUS    RESTARTS   AGE
web-v1-5dbbd65577-4cw9l   	1/1        Running      0                2m6s
web-v1-5dbbd65577-7dm44   	1/1        Running      0                2m6s
web-v1-5dbbd65577-85gcp   	1/1        Running      0                2m6s
web-v1-5dbbd65577-mk825   	1/1        Running      0                2m6s
web-v1-5dbbd65577-w22p6   	1/1        Running      0                2m6s
web-v2-5d5dd8c4b9-wgcgn   	1/1        Running      0                19s

[root@k8s-master ~]# kubectl  get  pods  --show-labels
NAME                      	READY   STATUS    RESTARTS   AGE     LABELS
web-v1-5dbbd65577-4cw9l   	1/1     Running   0          3m28s   app=web,pod-template-hash=5dbbd65577,version=v1
web-v1-5dbbd65577-7dm44   	1/1     Running   0          3m28s   app=web,pod-template-hash=5dbbd65577,version=v1
web-v1-5dbbd65577-85gcp   	1/1     Running   0          3m28s   app=web,pod-template-hash=5dbbd65577,version=v1
web-v1-5dbbd65577-mk825   	1/1     Running   0          3m28s   app=web,pod-template-hash=5dbbd65577,version=v1
web-v1-5dbbd65577-w22p6   	1/1     Running   0          3m28s   app=web,pod-template-hash=5dbbd65577,version=v1
web-v2-5d5dd8c4b9-wgcgn   	1/1     Running   0          101s    app=web,pod-template-hash=5d5dd8c4b9,version=v2

[root@k8s-master ~]# kubectl get pods -L version,app
NAME                     	 READY   STATUS    RESTARTS   AGE     VERSION   APP
web-v1-5dbbd65577-4cw9l   	1/1          Running     0                6m3s     v1            web
web-v1-5dbbd65577-7dm44   	1/1          Running     0                6m3s     v1            web
web-v1-5dbbd65577-85gcp   	1/1          Running     0                6m3s     v1            web
web-v1-5dbbd65577-mk825   	1/1          Running     0                6m3s     v1            web
web-v1-5dbbd65577-w22p6   	1/1          Running     0                6m3s     v1            web
web-v2-5d5dd8c4b9-wgcgn   	1/1          Running     0                4m16s    v2           web
```

**STEP 6) Service 생성**

Service는 version 라벨을 사용하지 않는다. 공통 라벨: `app=web`.

```
[root@k8s-master ~]# vi canary-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service

spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80

[root@k8s-master ~]# kubectl  apply  -f  canary-service.yaml
service/web-service created

[root@k8s-master ~]# kubectl  get  service  web-service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-service   ClusterIP   10.109.198.12   <none>        80/TCP    10s

[root@k8s-master ~]# kubectl get pods  -L app
NAME                      	READY   STATUS    RESTARTS   AGE     APP
web-v1-5dbbd65577-4cw9l   	1/1         Running     0                10m      web
web-v1-5dbbd65577-7dm44   	1/1         Running     0                10m      web
web-v1-5dbbd65577-85gcp   	1/1         Running     0                10m      web
web-v1-5dbbd65577-mk825   	1/1         Running     0                10m      web
web-v1-5dbbd65577-w22p6   	1/1         Running     0                10m      web
web-v2-5d5dd8c4b9-wgcgn   	1/1         Running     0                8m6s    web

[root@k8s-master ~]# kubectl  describe  service  web-service
Name:                     	web-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	app=web
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	10.109.198.12
IPs:                      	10.109.198.12
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.27:80,10.244.2.29:80,10.244.1.23:80 + 3 more...
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                   	<none>

[root@k8s-master ~]# kubectl  get endpoints  web-service
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME           ENDPOINTS                                                           AGE
web-service   10.244.1.22:80,10.244.1.23:80,10.244.1.24:80 + 3 more...   2m56s

[root@k8s-master ~]# kubectl  get  endpointslices
NAME                	ADDRESSTYPE   PORTS   ENDPOINTS                                    	AGE
kubernetes          	IPv4       	6443 	192.168.10.100                                    	13d
web-service-5s6z6	IPv4      	80    	10.244.2.27,10.244.2.29,10.244.1.23 + 3 more...	6m8s

[root@k8s-master ~]# kubectl  get  endpointslices  web-service-5s6z6
NAME                	ADDRESSTYPE   PORTS   ENDPOINTS                                    	AGE
web-service-5s6z6	IPv4      	80    	10.244.2.27,10.244.2.29,10.244.1.23 + 3 more...	6m8s

[root@k8s-master ~]# kubectl  get  endpointslices  -l  kubernetes.io/service-name=web-service
NAME                ADDRESSTYPE   PORTS   ENDPOINTS                                    	AGE
web-service-5s6z6   IPv4          80      10.244.2.27,10.244.2.29,10.244.1.23 + 3 more...	7m

# v1 Pod 확인
root@k8s-master ~]# kubectl  get  pods  -l  version=v1  -o  wide
NAME                      	READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
web-v1-5dbbd65577-4cw9l   	1/1         Running     0                87m    10.244.2.28   k8s-worker2   <none>           <none>
web-v1-5dbbd65577-7dm44   	1/1         Running     0                87m    10.244.1.23   k8s-worker1   <none>           <none>
web-v1-5dbbd65577-85gcp   	1/1         Running     0                87m    10.244.2.27   k8s-worker2   <none>           <none>
web-v1-5dbbd65577-mk825   	1/1         Running     0                87m    10.244.2.29   k8s-worker2   <none>           <none>
web-v1-5dbbd65577-w22p6   	1/1         Running     0                87m    10.244.1.22   k8s-worker1   <none>           <none>

[root@k8s-master ~]# curl  http://10.244.2.28
<!DOCTYPE html>
<html>
<head>
  <title>Canary Test</title>
</head>
<body>
  <h1>Canary Version V1</h1>
  <p>Stable Version</p>
</body>
</html>

# v2 Pod 확인
[root@k8s-master ~]# kubectl  get  pods  -l  version=v2  -o  wide
NAME                      	READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
web-v2-5d5dd8c4b9-wgcgn     1/1        Running      0                87m    10.244.1.24   k8s-worker1   <none>                  <none>

[root@k8s-master ~]# curl  http://10.244.1.24
<!DOCTYPE html>
<html>
<head>
  <title>Canary Test</title>
</head>
<body>
  <h1>Canary Version V2</h1>
  <p>Stable Version</p>
</body>
</html>
```

**STEP 7) Service를 통한 Canary 확인**

```
[root@k8s-master ~]# kubectl  get  svc  web-service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-service   ClusterIP   10.109.198.12     <none>              80/TCP    82m

[root@k8s-master ~]# for  i  in  {1..10}
> do
>    curl -s  http://10.109.198.12 | grep "<h1>";
> done
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>	<--- Canary 배포
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>	<--- Canary 배포
  <h1>Canary Version V1</h1>
```

10개 Pod 중 v2가 1개(약 1/6 비율)뿐이지만 요청이 랜덤하게 v1/v2로 분산되는 것을 확인할 수 있다.

**STEP 8) Canary Pod 비율 변경**

Pod 개수: v1 5개, v2 1개.

```
                                            web-service
                                                   |
	               +------------------------+-----------------------+
	               |                               	                                |
	   Deployment web-v1                                            Deployment web-v2
	      replicas: 3                                                             replicas: 1
	               |                               	                                 |
	      +-----+-----+-----+-----+                         	      |
	      |       |        |       |       |                          	      |
	     v1      v1      v1     v1     v1                                           v2

	     VERSION V1                                                 VERSION V2 - CANARY
```

```
# Canary Pod 비율 변경
[root@k8s-master ~]# kubectl  scale  deployment  web-v2  --replicas=3
deployment.apps/web-v2 scaled

[root@k8s-master ~]# kubectl  get  pods  -L  version
NAME                      	READY   STATUS    RESTARTS   AGE    VERSION
web-v1-5dbbd65577-4cw9l   	1/1     Running   0          100m   v1
web-v1-5dbbd65577-7dm44   	1/1     Running   0          100m   v1
web-v1-5dbbd65577-85gcp   	1/1     Running   0          100m   v1
web-v1-5dbbd65577-mk825   	1/1     Running   0          100m   v1
web-v1-5dbbd65577-w22p6   	1/1     Running   0          100m   v1
web-v2-5d5dd8c4b9-7wzsp   	1/1     Running   0          40s    v2
web-v2-5d5dd8c4b9-jcklb   	1/1     Running   0          40s    v2
web-v2-5d5dd8c4b9-wgcgn   	1/1     Running   0          98m    v2

[root@k8s-master ~]# for  i  in  {1..10}
> do
>    curl -s  http://10.109.198.12 | grep "<h1>";
> done

[root@k8s-master ~]# for  i  in  {1..10}; do   curl -s  http://10.109.198.12 | grep "<h1>"; done
  <h1>Canary Version V2</h1>	<--- Canary 배포
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>	<--- Canary 배포
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>	<--- Canary 배포
  <h1>Canary Version V2</h1>	<--- Canary 배포
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
```

이전보다 "VERSION V2 - CANARY" 응답을 볼 가능성이 증가한다.

**STEP 9) Canary에 문제가 발생한 경우**

신규 버전 v2에서 오류가 발생했다고 가정한다. v2를 즉시 0개로 줄인다.

```
# Canary Pod 비율 변경
[root@k8s-master ~]#  kubectl  scale  deployment  web-v2  --replicas=0
deployment.apps/web-v2 scaled

[root@k8s-master ~]# kubectl  get  pods  -L  version
NAME                      	READY   STATUS    RESTARTS   AGE    VERSION
web-v1-5dbbd65577-4cw9l   	1/1        Running     0                 104m    v1
web-v1-5dbbd65577-7dm44   	1/1        Running     0                 104m    v1
web-v1-5dbbd65577-85gcp   	1/1        Running     0                 104m    v1
web-v1-5dbbd65577-mk825   	1/1        Running     0                 104m    v1
web-v1-5dbbd65577-w22p6   	1/1        Running     0                 104m    v1

[root@k8s-master ~]# for  i  in  {1..10}; do   curl -s  http://10.109.198.12 | grep "<h1>"; done
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
```

`replicas: 0`으로 즉시 스케일 인하면 신규 버전 트래픽이 완전히 차단되어 장애 확산을 막을 수 있다. 이것이 Canary Deployment의 핵심 이점 중 하나다.

### web-ingress로 외부 노출

```
[root@k8s-master ~]# cat web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress

spec:
  ingressClassName: nginx

  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix

        backend:
          service:
            name: web-service
            port:
              number: 80

[root@k8s-master ~]# kubectl  get  service  -n ingress-nginx
NAME                                 	TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                              AGE
ingress-nginx-controller             	NodePort    10.105.3.121     	<none>            80:32181/TCP,443:30366/TCP   28h
ingress-nginx-controller-admission	ClusterIP   10.101.171.143	<none>            443/TCP                              28h

# 웹브라우저로 접속
https://192.168.10.100:30366/
```

10회 접속 시 v2 replicas가 0이면 모두 v1으로 접속된다.

```
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
```

```
# Scale Out
[root@k8s-master ~]#  kubectl  scale  deployment  web-v2  --replicas=3
deployment.apps/web-v2 scaled
```

다시 10회 접속 시:

```
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V1</h1>
  <h1>Canary Version V2</h1>
  <h1>Canary Version V1</h1>
```

`web-v2`의 replicas를 3으로 다시 늘리면 Ingress를 거친 외부 접속에서도 v1/v2가 다시 분산되는 것을 확인할 수 있다. Ingress → Service(`web-service`) → v1/v2 Pod로 이어지는 구조이므로, Service 뒤의 Pod 비율만 바꾸면 Ingress 설정을 변경하지 않고도 Canary 트래픽 비율을 조절할 수 있다.


---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `path: /login`처럼 `pathType: Prefix`만 사용하면 `/login/login.html`의 하위 경로 문자열이 그대로 백엔드로 전달되어 nginx 내부 파일 경로와 어긋날 수 있다. 하위 경로까지 깔끔하게 전달하려면 `nginx.ingress.kubernetes.io/use-regex: "true"`와 `rewrite-target: /$2`를 사용한 정규표현식 경로(`/login(/|$)(.*)`)로 전환한다.
- 정규표현식 Ingress에서 그룹 번호(`$1`, `$2`)와 `rewrite-target`이 어긋나면 엉뚱한 경로가 백엔드로 전달된다. `path: /class(/|$)(.*)`처럼 그룹을 두 개(`(/|$)`, `(.*)`) 구성했다면 `rewrite-target`은 두 번째 그룹인 `/$2`를 가리켜야 한다.
- Canary Deployment에서 v1/v2 Pod가 같은 Service에 묶이지 않는다면, Service의 `selector`가 `version` 라벨까지 포함하고 있지 않은지 확인한다. Canary 구성에서 Service selector는 반드시 공통 라벨(`app=web`)만 사용해야 하며 `version` 라벨은 selector에서 제외해야 한다.
- 신규 버전(Canary)에서 장애가 확인되면 `kubectl scale deployment <신규 Deployment> --replicas=0`으로 즉시 트래픽을 차단할 수 있다. Deployment를 삭제하지 않고 `replicas: 0`으로 낮추는 것만으로 신규 버전 Pod가 Service의 Endpoint에서 빠지므로 빠른 롤백에 유리하다.
- `kubectl get endpoints <service>` 또는 `kubectl get endpointslices -l kubernetes.io/service-name=<service>`로 Service 뒤에 실제로 연결된 Pod IP 목록을 확인하면, Ingress→Service→Pod 경로 중 어느 구간에서 문제가 발생했는지 빠르게 좁힐 수 있다(단, v1 Endpoints API는 v1.33부터 Deprecated이므로 `discovery.k8s.io/v1 EndpointSlice` 사용을 권장한다는 경고가 함께 출력된다).

---

> 📌 **핵심 요약**
> - `use-regex: "true"`와 `rewrite-target: /$2` 애노테이션을 사용하면 `/login(/|$)(.*)` 같은 정규표현식 경로로 하위 경로까지 정확히 백엔드로 rewrite해서 전달할 수 있다
> - Canary Deployment는 새 버전을 전체가 아닌 일부 Pod(또는 일부 사용자)에만 먼저 배포해 안정성을 확인한 뒤 점진적으로 확대하는 배포 방식이다
> - `app` 라벨은 공통, `version` 라벨은 구분하는 두 Deployment(web-v1/web-v2)를 같은 Service selector(`app`만 사용)로 묶으면 `--replicas` 조정만으로 트래픽 비율을 점진적으로 조절할 수 있다
> - 문제가 발생하면 `kubectl scale deployment web-v2 --replicas=0`으로 즉시 신규 버전 트래픽을 차단해 빠르게 롤백할 수 있다
> - 관련: 21. 🌐 Kubernetes - Ingress 기초와 준비 · 22. 🌐 Kubernetes - host·path 기반 Ingress · 18. 🔌 Kubernetes - Service 기초와 ClusterIP
