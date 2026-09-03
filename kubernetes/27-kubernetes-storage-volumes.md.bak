# Kubernetes - Storage 개요와 emptyDir·hostPath

> **Tag:** #Kubernetes #Storage #Volume #emptyDir #hostPath
> **핵심 요약:** 쿠버네티스 스토리지 전체 구조와 핵심 개념 4가지를 정리하고, emptyDir과 hostPath 볼륨을 실습으로 확인

---

## 1. 개요 (Overview)

Pod는 원래 언제든지 사라질 수 있는 존재다. Deployment는 Pod를 필요하면 새로 만들고, 문제가 있으면 재생성한다. 노드를 drain 하면 Pod가 다른 노드로 이동(재생성)하고, 스케줄러는 Pod를 다른 노드에 새로 올릴 수 있다.

컨테이너 내부 저장은 기본적으로 휘발성이다. 컨테이너 안에 파일을 저장해도 Pod가 삭제되면 같이 사라진다. 그래서 DB, 업로드 파일, 로그 같은 데이터는 컨테이너 내부에만 두면 안 된다.

Pod(애플리케이션)와 데이터(저장소)는 반드시 분리해야 한다. Pod는 언제든지 삭제, 재생성, 이동될 수 있는 존재다. 드레인, 장애 복구 과정에서 Pod는 쉽게 바뀐다. 따라서 Pod 안에 데이터를 두면 Pod가 사라질 때 데이터도 함께 사라진다.

Pod는 갈아끼우고, 데이터는 그대로 유지하는 것이 목표다. Pod는 소모품처럼 교체된다. 데이터는 PersistentVolume(PV), PersistentVolumeClaim(PVC)로 외부에 유지된다. Pod가 바뀌어도 동일한 PVC를 다시 마운트하면 기존 데이터를 그대로 사용한다.

### Storage 전체 구조

쿠버네티스 스토리지는 3단계로 구성된다.

1. **Pod 내부 관점**: 어디 경로에 마운트해서 쓰나?
   - `volumeMounts`: 컨테이너 경로(`/data` 등)에 붙여야 한다.
2. **쿠버네티스 리소스 관점**: 저장소를 어떻게 요청/할당하나?
   - `PV`: 실제 저장공간(리소스)
   - `PVC`: Pod가 요청하는 저장공간(주문서)
   - `StorageClass`: 동적 생성 규칙(자동으로 PV 만들어주는 규칙)
3. **실제 인프라 관점**(진짜 디스크가 어디 있나)
   - NFS, iSCSI, Ceph, GlusterFS, 클라우드 디스크(예: AWS EBS), 로컬 디스크 등

### 핵심 개념 4가지

- **Volume**
  - Pod가 사용하는 저장공간 인터페이스
  - Pod yaml에 `volumes`와 `volumeMounts`로 정의한다
  - Pod에 붙는 개념이라 Pod가 없어지면 연결도 끊긴다(데이터는 종류에 따라 유지/삭제)

- **PV (PersistentVolume)**
  - 클러스터에 존재하는 실제 저장공간
  - Pod와 분리되어 존재한다
  - 예: NFS 서버의 특정 경로, Ceph 볼륨, AWS EBS 디스크, Google Persistent Disk, Azure Disk 등

- **PVC (PersistentVolumeClaim)**
  - Pod가 PV를 요청하는 주문서
  - 필요한 용량(예: 10Gi), 접근 방식(예: ReadWriteOnce)을 적어서 요청한다.
  - 쿠버네티스는 조건이 맞는 PV를 찾아 PVC와 연결(binding)한다

- **StorageClass**
  - PVC가 만들어질 때 PV를 자동으로 생성해주는 규칙
  - 이게 있으면 관리자가 PV를 미리 만들어두지 않아도 된다(동적 프로비저닝)

### Volume 종류

1. **emptyDir**
   - 특징: Pod가 살아있는 동안만 존재하는 임시 공간
   - 사용처: 임시 캐시, 컨테이너 간 임시 파일 공유
   - Pod가 삭제되면 데이터도 삭제된다.

2. **hostPath**
   - 특징: 노드의 특정 디렉터리를 Pod에 마운트
   - 장점: 쉬움, 로컬 파일 접근 가능
   - 단점
     - Pod가 다른 노드로 이동하면 데이터가 안 보일 수 있다
     - 운영환경에서는 보안/이식성 문제가 커서 제한적으로만 사용

3. **configMap / secret** (설정/키 주입)
   - 파일로 설정값 또는 비밀값을 컨테이너에 제공하는 용도
   - DB 데이터 저장용이 아니다
   - secret은 base64로 저장되며, 민감정보 관리에 사용한다.

4. **PV/PVC 기반 영구 저장소**
   - Pod가 없어져도 데이터 유지
   - DB, 업로드 파일, 중요 로그 저장에 사용

---

## 2. emptyDir 실습

### EX1) emptyDir을 사용한 볼륨 실습

```
[root@k8s-master ~]# vi storage_test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-empt
spec:
  containers:
  - name: nginx-empt
    image: nginx:1.31

    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html

  volumes:
  - name: html
    emptyDir: {}

---	
apiVersion: v1
kind: Pod
metadata:
  name: web-noempt
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    ports:
    - containerPort: 80
      protocol: TCP
```

- emptyDir 임시 저장소를 마운트한다
- nginx가 사용하는 웹 루트 디렉터리를 Pod 전용 임시 공간으로 바꿔서 쓰는 구조
- nginx는 컨테이너 이미지에 들어 있던 기본 html이 아니라 emptyDir 볼륨에 있는 파일을 읽게 된다.

```
[root@k8s-master ~]# kubectl  apply  -f  storage_test.yaml
pod/web-empt created
pod/web-noempt created

[root@k8s-master ~]# kubectl  get  pods
NAME         	READY   STATUS    RESTARTS   AGE
web-empt     	1/1        Running      0                2m10s
web-noempt	1/1        Running      0                2m10s

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME         	READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
web-empt     	1/1         Running     0                2m33s   10.244.1.6   k8s-worker1   <none>           <none>
web-noempt	1/1         Running     0                2m33s   10.244.1.7   k8s-worker1   <none>           <none>

   # web-noemp
[root@k8s-master ~]# curl  http://10.244.1.7
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

   # web-emp
[root@k8s-master ~]# curl  http://10.244.1.6
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.31.3</center>
</body>
</html>
```

- 임시 디렉터리에 아무런 파일이 없기 때문에 결과가 출력되지 않는다.

```
   # web-noempt 컨테이너 접속
[root@k8s-master ~]# kubectl  exec  web-noempt  -it  -- /bin/bash
root@web-noempt:/#
root@web-noempt:/#
root@web-noempt:/# ls  -l  /usr/share/nginx/html/
total 8
-rw-r--r-- 1 root root 497 Jul 15 16:03 50x.html
-rw-r--r-- 1 root root 896 Jul 15 16:03 index.html		<---

   # web-empt 컨테이너 접속
[root@k8s-master ~]# kubectl  exec  web-empt  -it  -- /bin/bash
root@web-empt:/#
root@web-empt:/# ls  -l  /usr/share/nginx/html/
total 0						# index.html 파일이 확인되지 않는다.

root@web-empt:/# mount | grep  /usr
/dev/mapper/rl-root on /usr/share/nginx/html type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)

root@web-empt:/# echo "<h1> Storage EmptDir Test </h1>"  > /usr/share/nginx/html/index.html

[root@k8s-master ~]# curl  http://10.244.1.6
<h1> Storage EmptDir Test </h1>

[root@k8s-master ~]# kubectl  delete pods web-empt
pod "web-empt" deleted from default namespace

[root@k8s-master ~]# kubectl  delete pods web-noempt
pod "web-noempt" deleted from default namespace
```

### EX2) Pod 1개 안에서 nginx 2개를 생성 후 emptyDir로 index.html 공유

```
[root@k8s-master ~]# vi pod-emptydir-nginx2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir-nginx2
spec:
  containers:
  - name: nginx-a
    image: nginx:1.31
    ports:
    - containerPort: 80
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html

  - name: nginx-b
    image: nginx:1.31
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html

    command: ["/bin/sh", "-c"]
    args:
    - |
      cat > /etc/nginx/conf.d/default.conf <<'CONF'
      server {
        listen 8080;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        location / {
          try_files $uri $uri/ =404;
        }
      }
      CONF
      nginx -g 'daemon off;'

  volumes:
  - name: html
    emptyDir: {}
```

```
[root@k8s-master ~]# kubectl  apply  -f  pod-emptydir-nginx2.yaml
pod/pod-emptydir-nginx2 created

[root@k8s-master ~]# kubectl  get  pods
NAME                  		READY  	STATUS    RESTARTS   AGE
pod-emptydir-nginx2	2/2     	Running     0                 61s

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                  		READY   STATUS    RESTARTS   AGE    IP             NODE          NOMINATED NODE   READINESS GATES
pod-emptydir-nginx2   	2/2         Running     0                100s    10.244.1.8   k8s-worker1   <none>           <none>

# pod-emptydir-nginx2 Pod안의 nginx-a 컨테이너로 통신
[root@k8s-master ~]# curl  http://10.244.1.8:80
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.31.3</center>
</body>
</html>

# pod-emptydir-nginx2 Pod안의 nginx-b 컨테이너로 통신
[root@k8s-master ~]# curl  http://10.244.1.8:8080
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.31.3</center>
</body>
</html>

	# pod-emptydir-nginx2 Pod안의 nginx-a 컨테이너로 접속
[root@k8s-master ~]# kubectl  exec  -it  pod-emptydir-nginx2 -c nginx-a  --  /bin/bash
root@pod-emptydir-nginx2:/#

	# html 디렉터리에 index.html 파일이 없다.
root@pod-emptydir-nginx2:/# ls  -l  /usr/share/nginx/html/
total 0

	# /usr/share/nginx/html/ 경로에 index.html 파일 생성
root@pod-emptydir-nginx2:/# echo  "<h1> emptDir volumeMount Test NGINX html </h1>"  >  /usr/share/nginx/html/index.html

root@pod-emptydir-nginx2:/# cat  /usr/share/nginx/html/index.html                                    
<h1> emptDir volumeMount Test NGINX html </h1>

	# pod-emptydir-nginx2 Pod안의 nginx-b 컨테이너로 접속
[root@k8s-master ~]# kubectl  exec  -it  pod-emptydir-nginx2 -c nginx-b  --  /bin/bash
root@pod-emptydir-nginx2:/#

root@pod-emptydir-nginx2:/# cat  /usr/share/nginx/html/index.html
<h1> emptDir volumeMount Test NGINX html </h1>

[root@k8s-master ~]# curl  http://10.244.1.8:80
<h1> emptDir volumeMount Test NGINX html </h1>

[root@k8s-master ~]# curl  http://10.244.1.8:8080
<h1> emptDir volumeMount Test NGINX html </h1>

	# 실습 완료 후 Pod 삭제
[root@k8s-master ~]# kubectl  delete  pods  pod-emptydir-nginx2
pod "pod-emptydir-nginx2" deleted from default namespace
```

---

## 3. hostPath 실습

hostPath는 Pod가 실행되는 노드의 특정 디렉터리를 컨테이너 안에 그대로 마운트해서 사용하는 방식이다. Pod가 다른 노드로 이동할 수 있으므로, 모든 노드에 동일한 디렉터리가 존재해야 데이터 불일치 위험을 줄일 수 있다. 특정 Node의 데이터를 사용해야 한다면 nodeSelector나 nodeAffinity를 이용하여 Pod가 특정 Node에서 실행되도록 제한할 수 있다. 이 구조는 노드 종속적이어서 운영 환경에서는 권장되지 않는다.

### EX2) hostPath을 사용한 볼륨 실습

```
[root@k8s-master ~]# ssh  guest@k8s-worker1
guest@k8s-worker1's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Aug 18 11:06:41 2026 from 192.168.10.100

[guest@k8s-worker1 ~]$ sudo  mkdir  /webdata

[root@k8s-worker1 ~]# sudo  vi  /webdata/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>스토리지 테스트 페이지</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>스토리지 테스트 페이지입니다. 워커노드1</p>
</body>
</html>
```

```
[root@k8s-master ~]# ssh  guest@k8s-worker2
guest@k8s-worker1's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Aug 18 11:06:41 2026 from 192.168.10.100

[guest@k8s-worker2 ~]$ sudo  mkdir  /webdata

[root@k8s-worker2 ~]# sudo  vi  /webdata/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>스토리지 테스트 페이지</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>스토리지 테스트 페이지입니다. 워커노드2</p>
</body>
</html>
```

```
[root@k8s-master ~]# vi storage_hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-hostpath
spec:
  containers:
  - name: nginx
    image: nginx:1.29.1

    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html

  volumes:
  - name: html
    hostPath:
      path: /webdata
```

```
[root@k8s-master ~]# kubectl  apply  -f  storage_hostpath.yaml
pod/web-hostpath created

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME           	READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
web-hostpath	1/1         Running     0                22s     10.244.1.9   k8s-worker1   <none>           <none>

[root@k8s-master ~]# curl  http://10.244.1.9
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>스토리지 테스트 페이지</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>스토리지 테스트 페이지입니다. 워커노드1</p>
</body>
</html>

[root@k8s-master ~]# kubectl  exec  -it  web-hostpath  --  /bin/bash
root@web-hostpath:/#

root@web-hostpath:/# apt-get  update

root@web-hostpath:/# apt-get  install  -y  vim

root@web-hostpath:/# export LANG=C.UTF-8

root@web-hostpath:/# vim  /usr/share/nginx/html/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>Storage Test Page</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>Storage Test Page k8s-worker1</p>
</body>
</html>

[root@k8s-master ~]# ssh  guest@k8s-worker1
guest@k8s-worker1's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Thu Aug 27 11:00:21 2026 from 192.168.10.100

[guest@k8s-worker1 ~]$ cat /webdata/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>Storage Test Page</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>Storage Test Page k8s-worker1</p>
</body>
</html>
```

### EX) nodeSelector를 이용한 hostPath 실습

- hostPath는 Pod가 실행되는 Node의 로컬 디렉터리를 사용한다.
- 따라서 특정 Node의 디렉터리를 사용하려면 nodeSelector로 Pod를 그 Node에 고정할 수 있다.
- k8s-worker1의 `/webdata`를 Pod의 `/usr/share/nginx/html`에 마운트

```
	# STEP 1) worker1에 Label 추가

   # k8s-worker1에 식별용 Label을 추가
[root@k8s-master ~]# kubectl label node k8s-worker1 storage=hostpath

[root@k8s-master ~]# kubectl get nodes -L storage
NAME           STATUS  	ROLES           AGE   VERSION   STORAGE
k8s-master     Ready	control-plane   15d   v1.35.7
k8s-worker1   Ready	<none>          15d   v1.35.7   hostpath
k8s-worker2   Ready	<none>          15d   v1.35.7
```

```
	# STEP 2) worker1에 hostPath 디렉터리 생성 (위에서 생성한 디렉터리 재사용)

[root@k8s-master ~]# ssh  guest@k8s-worker1
guest@k8s-worker1's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Aug 18 11:06:41 2026 from 192.168.10.100

[guest@k8s-worker1 ~]$ sudo  mkdir  /webdata

[root@k8s-worker1 ~]# sudo  vi  /webdata/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>스토리지 테스트 페이지</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>스토리지 테스트 페이지입니다. 워커노드1</p>
</body>
</html>
```

```
	# STEP 3) nodeSelector + hostPath Deployment 생성

[root@k8s-master ~]# vi hostpath-nodeselector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-hostpath-deploy

spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-hostpath

  template:
    metadata:
      labels:
        app: web-hostpath
    spec:
      nodeSelector:
        storage: hostpath
      containers:
      - name: nginx
        image: nginx:1.31
        ports:
        - containerPort: 80

        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html

      volumes:
      - name: html
        hostPath:
          path: /webdata
          type: Directory

[root@k8s-master ~]# kubectl apply -f hostpath-nodeselector.yaml
deployment.apps/web-hostpath-deploy created

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                   	  READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
web-hostpath-deploy-67957494b6-66gh9	  1/1         Running     0                44s     10.244.1.11   k8s-worker1   <none>           <none>
web-hostpath-deploy-67957494b6-fwxtl	  1/1         Running     0                44s     10.244.1.13   k8s-worker1   <none>           <none>
web-hostpath-deploy-67957494b6-lx26j	  1/1         Running     0                44s     10.244.1.12   k8s-worker1   <none>           <none>
```

```
	# STEP 4) Pod 내부에서 확인

   # 컨테이너 접속
[root@k8s-master ~]# kubectl exec  -it  web-hostpath-deploy-67957494b6-66gh9  --  /bin/bash
root@web-hostpath-deploy-67957494b6-66gh9:/#
root@web-hostpath-deploy-67957494b6-66gh9:/# cat /usr/share/nginx/html/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>Storage Test Page</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>Storage Test Page k8s-worker1</p>
</body>
</html>

   # 컨테이너 접속
[root@k8s-master ~]# kubectl exec  -it  web-hostpath-deploy-67957494b6-fwxtl  --  /bin/bash
root@web-hostpath-deploy-67957494b6-fwxtl:/#
root@web-hostpath-deploy-67957494b6-fwxtl:/# cat /usr/share/nginx/html/index.html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>Storage Test Page</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>Storage Test Page k8s-worker1</p>
</body>
</html>

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                                   	  READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
web-hostpath-deploy-67957494b6-66gh9	  1/1         Running     0                44s     10.244.1.11   k8s-worker1   <none>           <none>
web-hostpath-deploy-67957494b6-fwxtl	  1/1         Running     0                44s     10.244.1.13   k8s-worker1   <none>           <none>
web-hostpath-deploy-67957494b6-lx26j	  1/1         Running     0                44s     10.244.1.12   k8s-worker1   <none>           <none>

[root@k8s-master ~]# curl  http://10.244.1.11
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>Storage Test Page</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>Storage Test Page k8s-worker1</p>
</body>
</html>

[root@k8s-master ~]# curl  http://10.244.1.12
[root@k8s-master ~]# curl  http://10.244.1.13

[root@k8s-master ~]# kubectl   delete  deployments  web-hostpath-deploy
deployment.apps "web-hostpath-deploy" deleted from default namespace
```

nodeSelector로 `storage: hostpath` 라벨이 붙은 k8s-worker1에만 Pod가 스케줄링되었기 때문에 3개 replica 모두 worker1에 생성되었고, worker2로는 스케줄링되지 않아 worker2의 `/webdata`(다른 내용)와는 무관하다는 것을 알 수 있다.

### hostPath 볼륨의 type 필드 옵션

| type | 설명 |
|---|---|
| `DirectoryOrCreate` | 지정한 경로에 디렉터리가 있으면 그대로 사용, 없으면 kubelet이 자동으로 빈 디렉터리 생성(기본 권한 0755) |
| `Directory` | 지정한 경로에 디렉터리가 반드시 존재해야 함. 없으면 Pod 생성 실패, 자동 생성하지 않음 |
| `FileOrCreate` | 지정한 경로에 파일이 있으면 그대로 사용, 없으면 kubelet이 빈 파일 자동 생성(기본 권한 0644). 단, 상위 디렉터리는 자동 생성하지 않음 |
| `File` | 지정한 경로에 파일이 반드시 존재해야 함. 없으면 Pod 생성 실패, 자동 생성하지 않음 |

```
hostPath:
  path: /webdata
  type: DirectoryOrCreate

hostPath:
  path: /webdata
  type: Directory

hostPath:
  path: /webdata/index.html
  type: FileOrCreate

hostPath:
  path: /webdata/index.html
  type: File
```

```
[root@k8s-master ~]# vi storage_directoryorcreate.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-directory
spec:
  containers:
  - image: nginx:1.31
    name: nginx

    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html

  volumes:
  - name: html
    hostPath:
      path: /webdata2
      type: DirectoryOrCreate

[root@k8s-master ~]# kubectl  apply  -f  storage_directoryorcreate.yaml
pod/web-directory created

[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME            	READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
web-directory	1/1         Running     0                25s     10.244.2.3   k8s-worker2   <none>                    <none>

# 디렉터리는 만드어졌지만 index.html이 없기 때문에 403에러
[root@k8s-master ~]# curl http://10.244.2.3
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.31.3</center>
</body>
</html>

[root@k8s-master ~]# ssh  guest@k8s-worker2
guest@k8s-worker2's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Thu Aug 27 11:02:53 2026 from 192.168.10.100
[guest@k8s-worker2 ~]$

[guest@k8s-worker2 ~]$ ls  -ld  /webdata*
drwxr-xr-x 2 root root 24  8월 27 11:03 /webdata
drwxr-xr-x 2 root root  6  8월 27 12:06 /webdata2

	# /webdata/index.html 파일을   /webdata2로 복사
[guest@k8s-worker2 ~]$ sudo  cp  /webdata/index.html   /webdata2/index.html
[guest@k8s-worker2 ~]$ exit

[root@k8s-master ~]# curl http://10.244.2.3
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <title>스토리지 테스트 페이지</title>
</head>
<body>
  <h1>Hello, Kubernetes!</h1>
  <p>스토리지 테스트 페이지입니다. 워커노드2</p>
</body>
</html>
```

---

## 4. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- 두 nginx Pod(`web-empt`, `web-noempt`)가 같은 이미지인데도 응답이 다른 이유는 emptyDir이 웹 루트를 완전히 빈 디렉터리로 덮어썼기 때문이다. `mount | grep /usr`로 실제 마운트 여부를 확인할 수 있다.
- 사이드카 컨테이너 간 파일 공유가 안 될 때는 두 컨테이너의 `volumeMounts.name`이 `volumes.name`과 정확히 일치하는지, `mountPath`가 올바른지 먼저 확인한다. `kubectl exec -it <pod> -c <container> -- /bin/bash`로 컨테이너별 파일시스템을 직접 비교하면 빠르다.
- hostPath 사용 시 Pod가 예상한 노드에 없으면(`kubectl get pods -o wide`로 NODE 컬럼 확인) 다른 노드의 디렉터리를 읽게 되어 데이터가 다르게 보일 수 있다. 특정 노드 고정이 필요하면 `kubectl label node <노드> <key>=<value>` 후 `nodeSelector`로 묶는다.
- `hostPath.type: Directory`/`File`은 경로가 없으면 Pod 생성 자체가 실패하므로, 운영 전에 모든 대상 노드에 동일한 디렉터리·파일을 미리 만들어 두어야 한다(`DirectoryOrCreate`/`FileOrCreate`를 쓰면 자동 생성되지만 초기 콘텐츠는 없다는 점에 주의).
- emptyDir·hostPath 모두 Pod 삭제 시점에 데이터 보존 여부가 다르다는 점을 헷갈리지 않아야 한다: emptyDir은 Pod 삭제 시 항상 소멸, hostPath는 Pod가 삭제돼도 노드 디스크에는 남는다(단, Pod가 다른 노드로 재생성되면 접근할 수 없다).

---

>  **핵심 요약**
> - 쿠버네티스 스토리지는 Pod 내부(volumeMounts) → 리소스(PV/PVC/StorageClass) → 실제 인프라(NFS 등) 3단계 구조로 이해한다
> - emptyDir은 Pod 생명주기와 함께하는 임시 저장소로, 삭제 시 데이터도 함께 사라지며 같은 Pod 내 컨테이너 간 파일 공유에 사용
> - hostPath는 노드의 로컬 디렉터리를 그대로 마운트하는 방식으로, 노드 종속적이라 nodeSelector로 특정 노드에 Pod를 고정해야 안정적으로 동작하며 운영 환경에서는 권장되지 않음
> - hostPath의 type 옵션(DirectoryOrCreate/Directory/FileOrCreate/File)에 따라 경로 자동 생성 여부와 Pod 생성 실패 여부가 달라짐
> - 관련: 28.  Kubernetes - PV·PVC와 StorageClass·Dynamic Provisioning
