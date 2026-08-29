# 🔐 Kubernetes - Secret

> **Tag:** #Kubernetes #Secret #Base64 #환경변수 #Volume #MySQL #부트캠프
> **핵심 요약:** 비밀번호, 토큰, 인증키 등 민감한 데이터를 Base64 인코딩된 형태로 저장·관리하는 Secret의 개념, 생성 방법, Pod에서 사용하는 방식(환경변수/파일 Mount)과 MySQL 연동 실습으로 정리

---

## 1. 📖 개요 (Overview)

Secret은 비밀번호, 토큰, 인증키 등 민감한 데이터를 저장하기 위한 Kubernetes 오브젝트이다.

ConfigMap과 마찬가지로 Key-Value 형태로 데이터를 저장한다.

```
Secret
  │
  ├── USERNAME = admin
  ├── PASSWORD = 1234
  └── TOKEN    = abc123
```

### Secret을 사용하는 이유

애플리케이션에서 다음과 같은 값을 Deployment YAML이나 Docker Image에 직접 작성하면 보안상 위험하다.

- DB_USERNAME=admin
- DB_PASSWORD=1234
- API_TOKEN=abc123

예를 들어 Deployment에 직접 작성하면:

```yaml
env:
  - name: DB_PASSWORD
    value: "1234"
```

설정값이 YAML에 그대로 노출된다.

Secret을 사용하면 민감한 값을 별도의 Kubernetes 오브젝트로 관리할 수 있다.

```
Secret  -->  Pod  -->  Container
```

### ConfigMap과 Secret의 차이

| 구분 | ConfigMap | Secret |
|---|---|---|
| 목적 | 일반 설정값 | 민감한 데이터 |
| 예 | 환경명, 로그 레벨 | 비밀번호, 토큰 |
| 오브젝트 | O | O |
| Key-Value | O | O |
| 환경변수 사용 | O | O |
| 파일 Mount | O | O |
| 기본 표현 | 일반 문자열 | Base64 |

- ConfigMap : 공개되어도 큰 문제가 없는 설정
- Secret : 보호해야 하는 설정

### Secret의 데이터 구조

Secret도 내부적으로 Key-Value 구조이다.

| Secret: app-secret | Key | Value |
|---|---|---|
| | USERNAME | admin |
| | PASSWORD | 1234 |
| | TOKEN | abc123 |

### Secret의 종류

Secret에는 여러 Type이 있다.

- Opaque
- kubernetes.io/tls
- kubernetes.io/dockerconfigjson
- kubernetes.io/basic-auth

**Opaque**
- 일반적인 Key-Value 데이터를 저장할 때 사용
- `type: Opaque`
- 우리가 실습에서 사용하는 일반적인 Secret

**TLS Secret**
- TLS 인증서와 개인키를 저장할 때 사용
- `tls.crt`
- `tls.key`

**Docker Registry Secret**
- Private Docker Registry 인증 정보를 저장할 때 사용
- `kubernetes.io/dockerconfigjson`

### Secret 생성 방법 1 (--from-literal)

직접 Key-Value를 입력하는 방식

```
kubectl create secret generic app-secret \
  --from-literal=USERNAME=admin \
  --from-literal=PASSWORD=1234
```

### Secret 생성 방법 2 (--from-file)

파일의 내용을 Secret으로 저장하는 방식

```
kubectl create secret generic app-secret  --from-file=password.txt
```

- Key = password.txt
- Value = password.txt 파일 내용

### Secret 생성 방법 2 (YAML)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret

type: Opaque

data:
  USERNAME: YWRtaW4=   # Base64값 직접 입력
  PASSWORD: MTIzNA==   # Base64값 직접 입력
```

**Secret의 Base64**

Secret의 data에 값을 직접 작성할 때는 일반적으로 Base64로 인코딩된 값을 사용한다.

```
echo -n "admin" | base64
YWRtaW4=
```

따라서

```yaml
data:
  USERNAME: YWRtaW4=
```

로 작성한다. (Base64는 암호화가 아니라 인코딩 방식이다.)

### Secret 저장값 확인

- Secret 목록 : `kubectl get secret`
- Secret 정보 : `kubectl describe secret app-secret`
- YAML : `kubectl get secret app-secret -o yaml`
- 특정 값 확인 : `kubectl get secret app-secret -o jsonpath='{.data.PASSWORD}' | base64 -d`

### Secret을 Pod에서 사용하는 방법

ConfigMap과 마찬가지로 크게 두 가지로 사용 가능하다.

**방법 1) 환경변수 (Key를 사용해서 Secret에 저장된 Value 적용)**

```
Secret  -->  Environment Variable  -->  Container
```

특정 Key만 가져오는 예:

```yaml
env:
  - name: PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: PASSWORD
```

**방법 2) 환경변수 (Secret에 환경변수를 적용)**

```yaml
envFrom:
  - secretRef:
      name: app-secret
```

**방법 3) Secret을 파일로 Mount**

```
Secret -->  Volume -->  Container 내부 파일
```

Pod에서 사용할 Volume을 정의한다. app-secret Secret을 Pod에서 사용할 Volume으로 가져오고, 그 Volume의 이름을 secret-volume으로 정의한다.

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

볼륨 적용:

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secret
```

- `/etc/secret/USERNAME` 경로 및 파일 생성
- `/etc/secret/PASSWORD` 경로 및 파일 생성

### Secret의 Key와 파일의 관계

예를 들어 Secret:

```
USERNAME = admin
PASSWORD = 1234
```

을 `/etc/secret`에 Mount하면:

```
/etc/secret
       ├── USERNAME
       └── PASSWORD
```

파일 내용

- USERNAME 파일 : key / admin : value
- PASSWORD 파일 : key / 1234 : value

즉 Secret의 Key가 파일 이름이 되고 Value가 파일 내용이 된다.

### Secret과 환경변수의 관계

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: PASSWORD
```

Secret:
```
PASSWORD = 1234
```

Container:
```
DB_PASSWORD=1234
```

즉 Secret의 Key 이름과 Container 환경변수 이름은 반드시 같을 필요가 없다.

### env와 envFrom 차이

**env** : Secret의 특정 Key만 가져온다.

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: PASSWORD
```

**envFrom** : Secret의 모든 Key를 가져온다.

```yaml
envFrom:
  - secretRef:
      name: app-secret
```

### Secret과 Volume의 관계

Secret 자체가 Volume은 아니다.

- Secret : 설정 데이터를 저장하는 Kubernetes 오브젝트
- Volume : Container에 저장공간이나 파일을 연결하는 방식

---

## 2. 🛠️ 실습 1 — Secret 생성과 Deployment 환경변수 적용

### STEP 1) 실습 디렉터리 생성

```
[root@k8s-master ~]# mkdir secret-practice

[root@k8s-master ~]# cd secret-practice

[root@k8s-master secret-practice]# pwd
/root/secret-practice
```

### STEP 2) Secret 생성

다음 조건에 맞는 Secret을 생성한다.

- Secret 이름 : app-secret
- USERNAME : admin
- PASSWORD : 1234

```
[root@k8s-master secret-practice]# kubectl create secret generic app-secret \
  --from-literal=USERNAME=admin \
  --from-literal=PASSWORD=1234
secret/app-secret created

[root@k8s-master secret-practice]# kubectl get secrets
NAME                                    TYPE                DATA   AGE
app-secret                              Opaque              2      28s
sh.helm.release.v1.nfs-provisioner.v1   helm.sh/release.v1  1      20h

[root@k8s-master secret-practice]# kubectl get secrets app-secret
NAME         TYPE     DATA   AGE
app-secret   Opaque   2      73s
```

| Secret: app-secret | Key | Value |
|---|---|---|
| | USERNAME | admin |
| | PASSWORD | 1234 |

### STEP 3) Secret에 저장된 값 확인

```
[root@k8s-master secret-practice]# kubectl get secrets app-secret -o yaml
apiVersion: v1
data:
  PASSWORD: MTIzNA==
  USERNAME: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2026-08-28T03:14:34Z"
  name: app-secret
  namespace: default
  resourceVersion: "582606"
  uid: 809838ba-4332-4c8c-bb3f-ba7e89110d4e
type: Opaque
```

Secret의 값이 Base64 형태로 표시된다.

- USERNAME=admin → USERNAME=YWRtaW4=
- PASSWORD=1234 → PASSWORD=MTIzNA

Base64는 암호화가 아니라 인코딩이므로 Base64 값만 알고 있어도 쉽게 원래 값을 확인할 수 있다.

Secret의 Base64 값을 원래 값으로 변환:

```
# text --> base64
[root@k8s-master secret-practice]# echo "admin" | base64
YWRtaW4K

# text <-- base64
[root@k8s-master secret-practice]# echo "YWRtaW4K" | base64 -d
admin

# text --> base64
[root@k8s-master secret-practice]# echo "1234" | base64
MTIzNAo=

# text <-- base64
[root@k8s-master secret-practice]# echo "MTIzNAo=" | base64 -d
1234
```

### STEP 4) Deployment YAML 작성

다음 조건에 맞는 Deployment를 작성한다.

- Deployment 이름 : secret-deployment
- Pod 2개
- Container 이름 : nginx
- Image : nginx:1.31
- Secret app-secret 사용
- Secret의 모든 Key-Value를 환경변수로 사용

```
[root@k8s-master secret-practice]# vi secret-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secret

  template:
    metadata:
      labels:
        app: secret
    spec:
      containers:
        - name: nginx
          image: nginx:1.31
          envFrom:
            - secretRef:
                name: app-secret
```

```yaml
envFrom:
  - secretRef:
      name: app-secret
```

app-secret Secret의 모든 Key-Value를 Container의 환경변수로 전달한다.

```
# Secret: app-secret
USERNAME=admin
PASSWORD=1234
          ↓
      envFrom
          ↓
   # Container
USERNAME=admin
PASSWORD=1234
```

### STEP 5) Deployment 생성 및 확인

작성한 Deployment YAML을 Kubernetes에 적용하고 Pod가 정상적으로 생성되었는지 확인한다.

```
[root@k8s-master secret-practice]# kubectl apply -f secret-deployment.yaml
deployment.apps/secret-deployment created

[root@k8s-master secret-practice]# kubectl get deployments secret-deployment
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
secret-deployment   2/2     2            2           18s

[root@k8s-master secret-practice]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
secret-deployment-68f778b75-6ddhh   1/1     Running   0          4s
secret-deployment-68f778b75-qhhd7   1/1     Running   0          4s
```

**Secret 환경변수 확인**

Pod에 Secret의 USERNAME과 PASSWORD가 환경변수로 전달되었는지 확인한다.

```
# 첫 번째 Pod에서 확인
[root@k8s-master secret-practice]# kubectl exec secret-deployment-68f778b75-6ddhh -- /bin/bash -c 'echo $USERNAME'
admin

# 첫 번째 Pod에서 확인
[root@k8s-master secret-practice]# kubectl exec secret-deployment-68f778b75-6ddhh -- /bin/bash -c 'echo $PASSWORD'
1234
```

두 번째 Pod에서도 동일하게 확인한다.

---

## 3. 🛠️ 실습 2 — Secret + MySQL 연동 실습

```
Secret  -->  Deployment  -->  Pod  -->  MySQL
```

Secret에 저장된 DB 접속 정보를 MySQL Container에 환경변수로 전달하여 실제 DB가 생성되는 것을 확인한다.

### STEP 1) 실습 디렉터리 생성

Secret + MySQL 실습을 위한 디렉터리를 생성하고 이동한다.

```
[root@k8s-master secret-practice]# cd

[root@k8s-master ~]# mkdir secret-mysql-practice

[root@k8s-master ~]# cd secret-mysql-practice

[root@k8s-master secret-mysql-practice]# pwd
/root/secret-mysql-practice
```

### STEP 2) MySQL Secret 생성

MySQL에서 사용할 민감한 정보를 Secret으로 생성한다.

- Secret 이름 : mysql-secret
- MYSQL_ROOT_PASSWORD : root1234
- MYSQL_DATABASE : testdb
- MYSQL_USER : appuser
- MYSQL_PASSWORD : app1234

```
[root@k8s-master secret-mysql-practice]# kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=root1234 \
  --from-literal=MYSQL_DATABASE=testdb \
  --from-literal=MYSQL_USER=appuser \
  --from-literal=MYSQL_PASSWORD=app1234
```

| Secret: mysql-secret | Key | Value |
|---|---|---|
| | MYSQL_ROOT_PASSWORD | root1234 |
| | MYSQL_DATABASE | testdb |
| | MYSQL_USER | appuser |
| | MYSQL_PASSWORD | app1234 |

MYSQL_ROOT_PASSWORD, MYSQL_PASSWORD는 민감한 정보이므로 Secret으로 관리한다.

```
[root@k8s-master configmap-test]# kubectl get secrets
NAME                                    TYPE                DATA   AGE
app-secret                              Opaque              2      35m
mysql-secret                            Opaque              4      4s
sh.helm.release.v1.nfs-provisioner.v1   helm.sh/release.v1  1      20h

[root@k8s-master configmap-test]# kubectl get secrets mysql-secret
NAME           TYPE     DATA   AGE
mysql-secret   Opaque   4      11s

[root@k8s-master configmap-test]# kubectl describe secrets mysql-secret
Name:         mysql-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
MYSQL_DATABASE:       6 bytes
MYSQL_PASSWORD:       7 bytes
MYSQL_ROOT_PASSWORD:  8 bytes
MYSQL_USER:           7 bytes

[root@k8s-master secret-mysql-practice]# kubectl get secret mysql-secret -o yaml
apiVersion: v1
data:
  MYSQL_DATABASE: dGVzdGRi
  MYSQL_PASSWORD: YXBwMTIzNA==
  MYSQL_ROOT_PASSWORD: cm9vdDEyMzQ=
  MYSQL_USER: YXBwdXNlcg==
kind: Secret
metadata:
  creationTimestamp: "2026-08-28T03:50:13Z"
  name: mysql-secret
  namespace: default
  resourceVersion: "586138"
  uid: 40bfc666-96c8-413e-aff4-cb8aaaf04f8c
type: Opaque
```

### STEP 3) MySQL Deployment YAML 작성

다음 조건에 맞는 MySQL Deployment를 작성한다.

- Deployment 이름 : mysql-deployment (Pod 1개 생성)
- Container 이름 : mysql
- Image : mysql:8.0
- Secret mysql-secret 사용
- Secret의 모든 Key-Value를 환경변수로 사용

```
[root@k8s-master secret-mysql-practice]# vi mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          envFrom:
            - secretRef:
                name: mysql-secret
```

envFrom을 사용하여 mysql-secret의 모든 Key-Value를 MySQL Container의 환경변수로 전달한다.

| Secret: mysql-secret | Key | Value |
|---|---|---|
| | MYSQL_ROOT_PASSWORD | root1234 |
| | MYSQL_DATABASE | testdb |
| | MYSQL_USER | appuser |
| | MYSQL_PASSWORD | app1234 |

```
mysql-secret
 - MYSQL_ROOT_PASSWORD
 - MYSQL_DATABASE
 - MYSQL_USER
 - MYSQL_PASSWORD
          ↓
     envFrom
          ↓
MySQL Container
 - MySQL Container에서는 다음과 같이 환경변수를 사용할 수 있다.
 - MYSQL_ROOT_PASSWORD=root1234
 - MYSQL_DATABASE=testdb
 - MYSQL_USER=appuser
 - MYSQL_PASSWORD=app1234
```

### STEP 5) Deployment 생성

작성한 MySQL Deployment를 Kubernetes에 적용한다.

```
[root@k8s-master secret-mysql-practice]# kubectl apply -f mysql-deployment.yaml
deployment.apps/mysql-deployment created

[root@k8s-master secret-mysql-practice]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
mysql-deployment   0/1     1            0           9s

[root@k8s-master secret-mysql-practice]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
mysql-deployment-595b4d775c-97w6l   1/1     Running   0          26s
```

Deployment가 MySQL Pod 1개를 생성한다.

### STEP 6) Pod에 Secret 환경변수가 전달되었는지 확인

MySQL Pod에 Secret의 환경변수가 정상적으로 전달되었는지 확인한다.

```
[root@k8s-master secret-mysql-practice]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
mysql-deployment-595b4d775c-97w6l   1/1     Running   0          26s

[root@k8s-master secret-mysql-practice]# kubectl exec mysql-deployment-595b4d775c-97w6l -- env | grep MYSQL
MYSQL_MAJOR=8.0
MYSQL_VERSION=8.0.46-1.el9
MYSQL_SHELL_VERSION=8.0.46-1.el9
MYSQL_PASSWORD=app1234
MYSQL_ROOT_PASSWORD=root1234
MYSQL_USER=appuser
MYSQL_DATABASE=testdb
```

Secret에 저장된 값이 MySQL Container의 환경변수로 전달된 것을 확인한다.

```
Secret: mysql-secret
 - MYSQL_ROOT_PASSWORD
 - MYSQL_DATABASE
 - MYSQL_USER
 - MYSQL_PASSWORD
        ↓
    envFrom
        ↓
MySQL Container
 - MYSQL_ROOT_PASSWORD
 - MYSQL_DATABASE
 - MYSQL_USER
 - MYSQL_PASSWORD
```

Secret 자체가 MySQL에 직접 연결되는 것이 아니라 Secret의 값을 환경변수로 전달하고, MySQL이 그 환경변수를 사용하는 구조이다.

### STEP 8) MySQL에 접속

MySQL Container에 접속하여 실제로 testdb 데이터베이스가 생성되었는지 확인한다.

```
# 컨테이너 접속
[root@k8s-master secret-mysql-practice]# kubectl exec -it mysql-deployment-595b4d775c-97w6l -- /bin/bash
bash-5.1#

# MySql 로그인
bash-5.1# mysql -u appuser -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.46 MySQL Community Server - GPL

Copyright (c) 2000, 2026, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SHOW DATABASES;
+---------------------+
| Database            |
+---------------------+
| information_schema  |
| performance_schema  |
| testdb               |
+---------------------+
3 rows in set (0.01 sec)

mysql> SELECT USER();
+--------------------+
| USER()             |
+--------------------+
| appuser@localhost  |
+--------------------+
1 row in set (0.00 sec)

mysql> USE testdb;
Database changed

# 테이블 생성
MySql> CREATE TABLE member (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

mysql> SHOW TABLES;
+-------------------+
| Tables_in_testdb  |
+-------------------+
| member            |
+-------------------+
1 row in set (0.00 sec)

mysql> INSERT INTO member VALUES(1, 'Kubernetes');
Query OK, 1 row affected (0.01 sec)

mysql> SELECT * FROM member;
+----+-------------+
| id | name        |
+----+-------------+
|  1 | Kubernetes  |
+----+-------------+
1 row in set (0.00 sec)
```

---

## 4. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- Secret의 `data` 값은 Base64로 인코딩되어 저장된다. Base64는 암호화가 아니라 단순 인코딩이므로 값을 알면 누구나 `base64 -d`로 원본을 복원할 수 있다. 접근 권한(RBAC)으로 Secret 조회 자체를 통제해야 실질적인 보안이 된다.
- `kubectl get secret <이름> -o jsonpath='{.data.<KEY>}' | base64 -d`로 특정 Key의 원본 값을 빠르게 확인할 수 있다.
- YAML의 `data`에 직접 값을 넣을 때는 반드시 Base64로 인코딩한 값을 넣어야 한다(`echo -n "값" | base64`). `-n` 옵션을 빠뜨리면 개행 문자까지 인코딩되어 디코딩한 값 끝에 개행이 포함될 수 있다.
- Secret을 환경변수(`env`/`envFrom`)로 주입한 경우, ConfigMap과 마찬가지로 Secret을 수정해도 이미 실행 중인 Pod에는 즉시 반영되지 않는다. Pod를 재시작(재생성)해야 새 값이 적용된다.
- Secret을 Volume으로 Mount한 경우 파일 내용은 symlink(`..data`) 방식으로 자동 갱신되지만, 환경변수는 갱신되지 않는다.
- `kubectl exec <pod> -- env | grep <KEY>`로 컨테이너에 실제로 전달된 환경변수를 확인할 수 있다.
- `kubectl describe secrets <이름>`은 Value를 직접 노출하지 않고 바이트 수(`N bytes`)만 표시한다. 실제 값 확인은 `-o yaml` 또는 `jsonpath` + `base64 -d` 조합을 사용해야 한다.
- Secret을 참조하는 Deployment/Pod가 `ContainerCreating` 상태에서 멈춰 있으면 참조한 Secret 이름/Key가 실제로 존재하는지 `kubectl describe pods <이름>`의 Events에서 확인한다.
- MySQL 등 공식 이미지 컨테이너에 접속해 로그인할 때는 `mysql -u <user> -p` 실행 후 프롬프트(`Enter password:`)에 비밀번호를 입력하며, 이 비밀번호는 Secret에서 전달된 환경변수(`MYSQL_PASSWORD`)와 동일한 값이어야 한다.

---

> 📌 **핵심 요약**
> - Secret은 비밀번호, 토큰, 인증키 등 민감한 데이터를 저장하는 Kubernetes 오브젝트이며 ConfigMap과 달리 값이 기본적으로 Base64로 인코딩되어 저장된다(암호화 아님)
> - 생성 방법: `--from-literal`(명령어), `--from-file`(파일), YAML 직접 작성(Base64 인코딩 필요)
> - 대표 Type: Opaque(일반 Key-Value), kubernetes.io/tls(인증서), kubernetes.io/dockerconfigjson(레지스트리 인증)
> - Pod에서 사용하는 방법: 환경변수(`env`의 `secretKeyRef` 또는 `envFrom`의 `secretRef`), 파일 Mount(`volumes`의 `secret`/`volumeMounts`)
> - MySQL 등 DB Deployment의 접속 정보(ROOT_PASSWORD, USER, PASSWORD 등)를 Secret + `envFrom`으로 전달하는 것이 실전 패턴
> - 관련: 29. ⚙️ Kubernetes - ConfigMap · 12. 🎛️ Kubernetes - Deployment · 2. 📦 Kubernetes - Pod 생성
