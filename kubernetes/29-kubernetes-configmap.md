# ⚙️ Kubernetes - ConfigMap

> **Tag:** #Kubernetes #ConfigMap #환경변수 #Volume #부트캠프
> **핵심 요약:** 애플리케이션의 민감하지 않은 설정값을 이미지와 분리해서 저장·관리하는 ConfigMap의 개념, 생성 방법, Pod에서 사용하는 두 가지 방식(환경변수/파일 Mount)을 실습으로 정리

---

## 1. 📖 개요 (Overview)

ConfigMap은 애플리케이션에서 사용하는 설정값(Configuration)을 저장하는 Kubernetes 오브젝트이다.

예를 들어 애플리케이션에서 다음과 같은 값을 사용한다고 하자.

- 개발/운영 환경 구분
- DB 주소
- 서버 포트
- 로그 레벨
- 애플리케이션 이름
- 외부 API 주소

이런 민감하지 않은 설정값을 Pod의 이미지에 직접 넣지 않고 ConfigMap에 저장해 사용할 수 있다.

### ConfigMap을 사용하는 이유

**기존 방식** — 애플리케이션 이미지 안에 설정값을 넣으면 환경이 변경될 때마다 이미지를 다시 만들어야 한다.

```
Docker Image
 └─ application
          └─ 설정값
```

개발 환경에서 운영 환경으로 변경하려면 이미지 수정이 필요하다.

**ConfigMap 사용**

```
Docker Image
   └─ application

ConfigMap
  └─ 환경별 설정값
```

애플리케이션이 ConfigMap의 설정값을 외부에서 받아 사용하도록 구성하기 때문에 이미지는 그대로 사용하고 ConfigMap만 변경할 수 있다. 즉, 애플리케이션 코드/이미지와 환경 설정을 분리하기 위해 사용한다.

### ConfigMap에 저장할 수 있는 값

ConfigMap에는 일반적인 문자열 형태의 설정값을 저장할 수 있다.

예:

```
APP_NAME=spring-app
APP_ENV=dev
LOG_LEVEL=INFO
SERVER_PORT=8080
DB_HOST=mysql
```

단, 비밀번호, API Key, 인증서 같은 민감한 정보는 ConfigMap이 아니라 Secret을 사용하는 것이 적절하다.

- ConfigMap : 일반적인 설정값
- Secret : 비밀번호, 인증정보 등 민감한 데이터

### ConfigMap 생성 방법

ConfigMap은 여러 방법으로 생성할 수 있다.

**1) 명령어로 생성**

```
kubectl create configmap <config-map이름> --from-literal=<키>=<값>
```

예:

```
kubectl create configmap app-config \       # 생성될 Config-map 이름 (app-config)
  --from-literal=APP_NAME=spring-app \      # 저장할 Key=Value (APP_NAME=spring-app)
  --from-literal=APP_ENV=dev \              # 저장할 Key=Value (APP_ENV=dev)
  --from-literal=LOG_LEVEL=INFO             # 저장할 Key=Value (LOG_LEVEL=INFO)
```

- app-config : 생성할 ConfigMap 이름
- APP_NAME : Key / spring-app : Value
- APP_ENV : Key / dev : Value
- LOG_LEVEL : Key / INFO : Value

**2) YAML로 생성**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config          # 생성될 Config-map 이름 (app-config)

data:
  APP_NAME: spring-app      # 저장할 Key=Value (APP_NAME=spring-app)
  APP_ENV: dev               # 저장할 Key=Value (APP_ENV=dev)
  LOG_LEVEL: INFO            # 저장할 Key=Value (LOG_LEVEL=INFO)
```

| ConfigMap: app-config | Key | Value |
|---|---|---|
| | APP_NAME | spring-app |
| | APP_ENV | dev |
| | LOG_LEVEL | INFO |

### ConfigMap 확인 명령어

- ConfigMap 목록 : `kubectl get configmap` / `kubectl get cm`
- 특정 ConfigMap 확인 : `kubectl get configmap app-config` / `kubectl describe configmap app-config`
- YAML 형태로 확인 : `kubectl get configmap app-config -o yaml`

### ConfigMap을 Pod에서 사용하는 방법

ConfigMap의 값을 Pod에서 사용하는 대표적인 방법은 2가지이다.

**방법 1) 환경변수로 전달**

```
ConfigMap  --->  Environment Variable  --->  Container
```

예:

```yaml
env:
  - name: APP_NAME
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_NAME
```

컨테이너 내부에서는 `echo $APP_NAME` 명령어로 확인이 가능하다. 결과 : `APP_NAME: spring-app`

**방법 2) 파일로 Mount**

| ConfigMap: app-config | Key | Value |
|---|---|---|
| | APP_NAME | spring-app |
| | APP_ENV | dev |
| | LOG_LEVEL | INFO |

ConfigMap을 Pod 내부의 파일로 사용할 수도 있다.

```
ConfigMap  --->  Volume  --->  Container 내부 파일
```

예:

```yaml
volumes:                       # Pod에서 사용할 Volume을 정의
  - name: config-volume
    configMap:                 # Volume의 데이터 원본이 ConfigMap이라는 의미
      name: app-config         # 어떤 ConfigMap을 Volume으로 사용할 것인지 지정

volumeMounts:                  # Volume을 Container 내부에 연결(Mount)하는 설정
  - name: config-volume        # 위에서 만든 "config-volume" Volume을 지정
    mountPath: /etc/config     # Container 내부에서 Volume을 어느 디렉터리에 연결할 것인지 지정
```

그러면 ConfigMap의 각 key가 파일 형태로 만들어진다.

- `/etc/config/APP_NAME`
- `/etc/config/APP_ENV`
- `/etc/config/LOG_LEVEL`
- APP_NAME 이름의 파일이 만들어지고 파일이름이 key가 되고 안의 값이 value가 된다.
- APP_ENV 이름의 파일이 만들어지고 파일이름이 key가 되고 안의 값이 value가 된다.
- LOG_LEVEL 이름의 파일이 만들어지고 파일이름이 key가 되고 안의 값이 value가 된다.

---

## 2. 🛠️ 실습

### 1-1) ConfigMap 생성

```
[root@k8s-master ~]# kubectl create configmap app-config \
  --from-literal=APP_NAME=spring-app \
  --from-literal=APP_ENV=dev \
  --from-literal=LOG_LEVEL=INFO

[root@k8s-master ~]# kubectl get configmaps
NAME              DATA   AGE
app-config        3      20s
kube-root-ca.crt  1      16d

[root@k8s-master ~]# kubectl get configmaps app-config
NAME         DATA   AGE
app-config   3      55s
```

### config-map 사용 X (환경변수를 직접 작성)

```
[root@k8s-master ~]# vi configmap-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-configmap-pod

spec:
  containers:
    - name: nginx-container
      image: nginx:1.31

      env:
        - name: APP_NAME
          value: "spring-app"

        - name: APP_ENV
          value: "dev"

        - name: LOG_LEVEL
          value: "INFO"
```

| ConfigMap: app-config | Key | Value |
|---|---|---|
| | APP_NAME | spring-app |
| | APP_ENV | dev |
| | LOG_LEVEL | INFO |

### config-map 사용 O (configMapKeyRef로 참조)

```
[root@k8s-master ~]# vi configmap-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:1.31

      env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_NAME

        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV

        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL

[root@k8s-master ~]# kubectl apply -f configmap-pod.yaml
pod/configmap-pod created

[root@k8s-master ~]# kubectl get pods configmap-pod
NAME            READY   STATUS    RESTARTS   AGE
configmap-pod   1/1     Running   0          22s

[root@k8s-master ~]# kubectl exec -it configmap-pod -- /bin/bash
root@configmap-pod:/#

root@configmap-pod:/# echo $APP_NAME
spring-app

root@configmap-pod:/# echo $APP_ENV
dev

root@configmap-pod:/# echo $LOG_LEVEL
INFO

[root@k8s-master ~]# kubectl exec configmap-pod -- env | grep -E 'APP_NAME|APP_ENV|LOG_LEVEL'
LOG_LEVEL=INFO
APP_NAME=spring-app
APP_ENV=dev
```

### 2) ConfigMap 전체를 환경변수로 사용하는 방법

ConfigMap의 값을 하나씩 지정하는 대신 전체 key-value를 한꺼번에 환경변수로 가져올 수도 있다.

| ConfigMap: app-config | Key | Value |
|---|---|---|
| | APP_NAME | spring-app |
| | APP_ENV | dev |
| | LOG_LEVEL | INFO |

```
[root@k8s-master ~]# vi configmap-all.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-envfrom-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.31

      envFrom:
        - configMapRef:
            name: app-config

[root@k8s-master ~]# kubectl apply -f configmap-all.yaml
pod/configmap-envfrom-pod created

[root@k8s-master ~]# kubectl exec configmap-envfrom-pod -- env | grep -E 'APP_NAME|APP_ENV|LOG_LEVEL'
APP_NAME=spring-app
LOG_LEVEL=INFO
APP_ENV=dev
```

### 3) ConfigMap을 파일로 Mount하는 실습

**3-1) ConfigMap 생성**

```
[root@k8s-master ~]# kubectl create configmap file-config \
  --from-literal=APP_NAME=spring-app \
  --from-literal=APP_ENV=dev \
  --from-literal=LOG_LEVEL=INFO

[root@k8s-master ~]# kubectl get configmaps
NAME              DATA   AGE
app-config        3      39m
file-config       3      6s
kube-root-ca.crt  1      16d

[root@k8s-master ~]# vi configmap-volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.31

      volumeMounts:
        - name: config-volume
          mountPath: /etc/config

  volumes:
    - name: config-volume
      configMap:
        name: file-config
```

**컨테이너에 접속 후 확인**

```
[root@k8s-master ~]# kubectl apply -f configmap-volume-pod.yaml
pod/configmap-volume-pod created

[root@k8s-master ~]# kubectl exec -it configmap-volume-pod -- /bin/bash
root@configmap-volume-pod:/#

root@configmap-volume-pod:/# ls -l /etc/config/
total 0
lrwxrwxrwx 1 root root 14 Aug 28 01:36 APP_ENV -> ..data/APP_ENV
lrwxrwxrwx 1 root root 15 Aug 28 01:36 APP_NAME -> ..data/APP_NAME
lrwxrwxrwx 1 root root 16 Aug 28 01:36 LOG_LEVEL -> ..data/LOG_LEVEL

root@configmap-volume-pod:/# cat /etc/config/APP_ENV
dev

root@configmap-volume-pod:/# cat /etc/config/APP_NAME
spring-app

root@configmap-volume-pod:/# cat /etc/config/LOG_LEVEL
INFO
```

**컨테이너에 접속하지 않고 확인**

```
[root@k8s-master ~]# kubectl exec configmap-volume-pod -- ls -l /etc/config
total 0
lrwxrwxrwx 1 root root 14 Aug 28 01:36 APP_ENV -> ..data/APP_ENV
lrwxrwxrwx 1 root root 15 Aug 28 01:36 APP_NAME -> ..data/APP_NAME
lrwxrwxrwx 1 root root 16 Aug 28 01:36 LOG_LEVEL -> ..data/LOG_LEVEL

[root@k8s-master ~]# kubectl exec configmap-volume-pod -- cat /etc/config/APP_NAME
spring-app

[root@k8s-master ~]# kubectl exec configmap-volume-pod -- cat /etc/config/APP_ENV
dev

[root@k8s-master ~]# kubectl exec configmap-volume-pod -- cat /etc/config/LOG_LEVEL
INFO
```

### ConfigMap 파일 Mount 실습 (index.html)

ConfigMap에 File을 저장한 후 해당 파일을 Key=Value 형태로 사용한다.

```
ConfigMap  -->  Deployment  -->  Pod  -->  Volume  -->  index.html
```

**STEP 1) 실습 디렉터리 생성**

```
[root@k8s-master ~]# mkdir configmap-test

[root@k8s-master ~]# cd configmap-test/

[root@k8s-master configmap-test]# pwd
/root/configmap-test
```

**STEP 2) index.html 파일 생성**

```
[root@k8s-master configmap-test]# vi index.html
<!DOCTYPE html>
<html>
<head>
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
</body>
</html>

[root@k8s-master configmap-test]# cat index.html
```

index.html을 ConfigMap에 저장. ConfigMap으로 생성하면 다음과 같은 구조가 된다.

```
ConfigMap : web-page
   Key
    └── index.html
   Value
    └── index.html 파일 내용
```

- Key = index.html
- Value = HTML 코드 전체

| ConfigMap: web-page | Key | Value |
|---|---|---|
| | index.html | index.html 파일 내용 |

```
[root@k8s-master configmap-test]# kubectl create configmap web-page --from-file=index.html
configmap/web-page created

[root@k8s-master configmap-test]# kubectl get configmaps
NAME              DATA   AGE
app-config        3      57m
file-config       3      17m
kube-root-ca.crt  1      16d
web-page          1      15s
```

**STEP 4) ConfigMap 내용 확인**

web-page ConfigMap에 index.html이 정상적으로 저장되었는지 확인

```
Name:         web-page
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
index.html:
----
<!DOCTYPE html>
<html>
<head>
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
</body>
</html>


BinaryData
====

Events:  <none>
```

```
[root@k8s-master configmap-test]# kubectl get configmaps web-page -o yaml
apiVersion: v1
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>ConfigMap 실습</title>
    </head>
    <body>
        <h1>쿠버네티스 ConfigMap 실습</h1>

        <p>ConfigMap으로 전달된 index.html입니다.</p>
    </body>
    </html>
kind: ConfigMap
metadata:
  creationTimestamp: "2026-08-28T01:51:14Z"
  name: web-page
  namespace: default
  resourceVersion: "574377"
  uid: 983de534-262d-4925-816f-cd19af8b187e
```

**STEP 5) local-config ConfigMap 생성**

Container의 문자 환경을 UTF-8로 설정하기 위한 값을 ConfigMap에 저장한다. Container에서 사용할 LANG과 LC_ALL 환경변수를 ConfigMap으로 생성한다.

- ConfigMap 이름 : local-config
- Key = LANG, Value = C.UTF-8
- Key = LC_ALL, Value = C.UTF-8

```
[root@k8s-master configmap-test]# kubectl create configmap local-config \
  --from-literal=LANG=C.UTF-8 \
  --from-literal=LC_ALL=C.UTF-8
```

| ConfigMap: local-config | Key | Value |
|---|---|---|
| | LANG | C.UTF-8 |
| | LC_ALL | C.UTF-8 |

```
[root@k8s-master configmap-test]# kubectl get configmap
NAME              DATA   AGE
app-config        3      63m
file-config       3      24m
kube-root-ca.crt  1      16d
local-config      2      11s
web-page          1      6m38s
```

**모든 ConfigMap의 상세 정보 확인**

```
[root@k8s-master configmap-test]# kubectl describe configmaps
```

**특정 ConfigMap의 상세 정보 확인**

```
[root@k8s-master configmap-test]# kubectl describe configmaps local-config
Name:         local-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
LANG:
----
C.UTF-8

LC_ALL:
----
C.UTF-8


BinaryData
====

Events:  <none>
```

**STEP 6) Deployment YAML 작성**

다음 조건에 맞는 Deployment를 작성한다.

- Deployment 이름 : web-deployment (Pod 2개)
- Container 이름 : nginx-container
- Image : nginx:1.31
- ConfigMap web-page를 Volume으로 사용
- `/usr/share/nginx/html`에 Mount
- ConfigMap local-config를 환경변수로 사용

```
[root@k8s-master configmap-test]# vi web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx-container
          image: nginx:1.31

          envFrom:
            - configMapRef:
                name: local-config

          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html

      volumes:
        - name: html-volume
          configMap:
            name: web-page
```

local-config ConfigMap의 모든 Key-Value를 Container의 환경변수로 사용한다.

```yaml
envFrom:
  - configMapRef:
      name: local-config
```

따라서 Container에는 다음 환경변수가 적용된다.

```
LANG=C.UTF-8
LC_ALL=C.UTF-8
```

web-page ConfigMap을 html-volume이라는 Volume으로 사용한다.

```yaml
volumes:
  - name: html-volume
    configMap:
      name: web-page
```

html-volume을 Container의 `/usr/share/nginx/html`에 Mount 한다.

```yaml
volumeMounts:
  - name: html-volume
    mountPath: /usr/share/nginx/html
```

전체 흐름

- web-page ConfigMap --> html-volume --> /usr/share/nginx/html
- local-config ConfigMap --> 환경변수 --> Container

**STEP 7) Deployment 생성 및 확인**

```
[root@k8s-master configmap-test]# kubectl apply -f web-deployment.yaml
deployment.apps/web-deployment created

[root@k8s-master configmap-test]# kubectl get deployments web-deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
web-deployment   2/2     2            2           35s

[root@k8s-master configmap-test]# kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
web-deployment-6d5df754c5-2hxsn   1/1     Running   0          115s
web-deployment-6d5df754c5-gqkn9   1/1     Running   0          115s
```

Deployment가 생성되고 `replicas: 2`에 따라 Pod 2개를 생성한다. Deployment의 Pod Template에 local-config와 web-page 사용 설정이 적용된다.

**STEP 8) Container의 LANG, LC_ALL 확인**

```
[root@k8s-master configmap-test]# kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
web-deployment-6d5df754c5-2hxsn   1/1     Running   0          115s
web-deployment-6d5df754c5-gqkn9   1/1     Running   0          115s

[root@k8s-master configmap-test]# kubectl exec web-deployment-6d5df754c5-2hxsn -- /bin/bash -c 'echo $LANG'
C.UTF-8

[root@k8s-master configmap-test]# kubectl exec web-deployment-6d5df754c5-2hxsn -- /bin/bash -c 'echo $LC_ALL'
C.UTF-8

[root@k8s-master configmap-test]# kubectl exec web-deployment-6d5df754c5-2hxsn -- env | grep -E 'LANG|LC_ALL'
LANG=C.UTF-8
LC_ALL=C.UTF-8
```

local-config ConfigMap의 "LANG=C.UTF-8", "LC_ALL=C.UTF-8" 값이 envFrom을 통해 Container의 환경변수로 적용된다.

**STEP 9) Pod 내부의 index.html 확인**

```
[root@k8s-master configmap-test]# kubectl exec web-deployment-6d5df754c5-2hxsn -- ls -l /usr/share/nginx/html
total 0
lrwxrwxrwx 1 root root 17 Aug 28 02:13 index.html -> ..data/index.html
```

구조

```
ConfigMap
   └── index.html
                ↓
            Volume
                ↓
Container
   └── /usr/share/nginx/html/index.html
```

```
[root@k8s-master configmap-test]# kubectl exec web-deployment-6d5df754c5-2hxsn -- cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
</body>
</html>

[root@k8s-master configmap-test]# kubectl exec -it web-deployment-6d5df754c5-2hxsn -- /bin/bash
root@web-deployment-6d5df754c5-2hxsn:/#

root@web-deployment-6d5df754c5-2hxsn:/# cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
</body>
</html>
```

모든 Pod에 동일한 web-page ConfigMap이 Mount된다.

```
                               Deployment
                                    │
        ┌─────────┴─────────┐
        ↓                   ↓
      Pod 1               Pod 2
        │                   │
        ↓                   ↓
   index.html          index.html
```

```
[root@k8s-master configmap-test]# kubectl get pods -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
web-deployment-6d5df754c5-2hxsn   1/1     Running   0          14m   10.244.2.6    k8s-worker2   <none>            <none>
web-deployment-6d5df754c5-gqkn9   1/1     Running   0          14m   10.244.1.5    k8s-worker1   <none>            <none>

[root@k8s-master configmap-test]# curl http://10.244.2.6
<!DOCTYPE html>
<html>
<head>
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
</body>
</html>

[root@k8s-master configmap-test]# curl http://10.244.1.5
<!DOCTYPE html>
<html>
<head>
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
</body>
</html>
```

**STEP 10) ConfigMap의 index.html 수정**

ConfigMap은 Kubernetes API 리소스이므로 etcd에 저장된다.

web-page ConfigMap의 index.html에 다음 내용을 추가한다.

```
[root@k8s-master configmap-test]# kubectl edit configmaps web-page
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">                                       # 추가 설정
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>

    <h2>ConfigMap 수정 테스트</h2>                                 # 추가 설정
    <p>이 내용은 ConfigMap을 수정한 후 추가되었습니다.</p>              # 추가 설정
    <p>현재 버전: Version 2</p>                                    # 추가 설정
</body>
</html>
```

ConfigMap은 Kubernetes API 리소스이므로 생성하거나 수정한 내용은 kube-apiserver를 통해 etcd에 저장된다. 즉, `kubectl edit configmap web-page` 명령으로 내용을 수정하면 변경된 ConfigMap 데이터가 Kubernetes의 상태 정보로 etcd에 저장된다.

**컨테이너에 접속 후 변경 확인**

```
[root@k8s-master configmap-test]# kubectl exec -it web-deployment-6d5df754c5-2hxsn -- /bin/bash
root@web-deployment-6d5df754c5-2hxsn:/#
root@web-deployment-6d5df754c5-2hxsn:/#

root@web-deployment-6d5df754c5-2hxsn:/# cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
    <h2>ConfigMap 수정 테스트</h2>
    <p>이 내용은 ConfigMap을 수정한 후 추가되었습니다.</p>
    <p>현재 버전: Version 2</p>

</body>
</html>

[root@k8s-master configmap-test]# curl http://10.244.2.6
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
    <h2>ConfigMap 수정 테스트</h2>
    <p>이 내용은 ConfigMap을 수정한 후 추가되었습니다.</p>
    <p>현재 버전: Version 2</p>

</body>
</html>

[root@k8s-master configmap-test]# curl http://10.244.1.5
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>ConfigMap 실습</title>
</head>
<body>
    <h1>쿠버네티스 ConfigMap 실습</h1>

    <p>ConfigMap으로 전달된 index.html입니다.</p>
    <h2>ConfigMap 수정 테스트</h2>
    <p>이 내용은 ConfigMap을 수정한 후 추가되었습니다.</p>
    <p>현재 버전: Version 2</p>

</body>
</html>
```

Pod를 재시작하지 않아도 ConfigMap을 Volume으로 Mount한 경우 변경 내용이 Pod 내부 파일에도 반영되는 것을 확인할 수 있다(단, 환경변수로 주입한 값은 Pod가 재생성되어야 반영된다).

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- ConfigMap을 환경변수(`env`/`envFrom`)로 주입한 경우, ConfigMap을 수정해도 이미 실행 중인 Pod의 환경변수는 즉시 반영되지 않는다. 새 값을 적용하려면 Pod를 재시작(재생성)해야 한다.
- ConfigMap을 Volume으로 Mount한 경우, `kubectl edit configmaps <이름>`으로 수정하면 파일 내용이 일정 시간 내에 자동으로 갱신된다(symlink 방식으로 `..data`를 가리키기 때문).
- `kubectl exec <pod> -- env | grep <KEY>`로 컨테이너에 실제로 전달된 환경변수를 빠르게 확인할 수 있다.
- ConfigMap을 `--from-file`로 생성하면 파일 이름이 Key, 파일 내용이 Value가 되므로, 여러 파일을 Volume Mount할 때 원하는 파일명이 유지되는지 확인해야 한다.
- `kubectl get pods`의 STATUS가 `ContainerCreating`에서 멈춰 있고 ConfigMap을 참조하는 경우, 참조한 ConfigMap 이름/Key가 실제로 존재하는지 `kubectl describe pods <이름>`의 Events에서 확인한다(존재하지 않는 ConfigMap을 참조하면 Pod가 정상적으로 시작되지 않을 수 있다).

---

> 📌 **핵심 요약**
> - ConfigMap은 민감하지 않은 설정값을 애플리케이션 이미지와 분리해서 저장하는 Kubernetes 오브젝트
> - 생성 방법: `--from-literal`(명령어), `--from-file`(파일), YAML 직접 작성
> - Pod에서 사용하는 방법 2가지: 환경변수(`env`/`envFrom`), 파일 Mount(`volumes`/`volumeMounts`)
> - Volume으로 Mount한 ConfigMap은 수정 시 파일 내용이 자동 갱신되지만, 환경변수로 주입한 값은 Pod 재생성이 필요
> - 관련: 30. 🔐 Kubernetes - Secret · 12. 🎛️ Kubernetes - Deployment · 5. 🎚️ Kubernetes - ResourceQuota·LimitRange
