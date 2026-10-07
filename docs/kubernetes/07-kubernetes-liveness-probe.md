# Kubernetes - livenessProbe

> **Tag:** #Kubernetes #livenessProbe #SelfHealing #Probe
> **핵심 요약:** livenessProbe(httpGet/tcpSocket/exec) 3가지 방식과 Self-healing Pod 동작 원리, 각 방식별 실습 정리

---

## 1. livenessProbe와 Self-healing Pod

### livenessProbe란 무엇인가

실제 서버에서 가장 무서운 상황은 프로그램이 멈췄는데 서버는 살아 있는 것처럼 보이는 상태다.

예제 상황:
- 웹 서버 프로세스는 떠 있다.
- 포트도 열려 있다.
- 그런데 요청을 보내면 응답이 없다.
- 로그도 안 찍힌다.

이걸 사람 손으로 관리하면 왜 안 되는지를 찾고 시스템을 재부팅하는 상황이 반복될 수 있다. 쿠버네티스는 이걸 사람이 아니라 시스템이 자동으로 처리하도록 설계되어 있다. 그 핵심이 바로 livenessProbe, Self-healing Pod이다.

livenessProbe를 한 문장으로 말하면 "컨테이너가 아직 살아있는 상태인가를 쿠버네티스가 직접 확인하는 방법"이다.

여기서 중요한 포인트는 프로세스가 떠 있냐가 아니라 정상적으로 살아있냐를 본다는 점이다. 프로세스는 살아 있는데 무한 루프에 빠졌거나 데드락 상태이거나 외부 요청에 전혀 응답하지 못하는 상태, 이런 경우는 실제로는 죽은 것과 다름없다.

livenessProbe는 바로 이 상태를 잡아내기 위한 장치다.

### kubelet이 livenessProbe를 사용하는 방식

Pod가 워커 노드에서 실행되면 해당 노드의 kubelet은 가만히 있는 게 아니라, kubelet은 정해진 주기마다 컨테이너에게 정상적으로 동작 중인지를 확인한다.

이 질문을 하는 방식이 바로 livenessProbe다. Probe 결과는 단순하다.
- 요청 메시지 전송 후 응답 메시지를 수신 성공 시 → 정상
- 요청 메시지 전송 후 응답 메시지를 수신 실패 시 → 비정상

livenessProbe가 실패하면 kubelet은 해당 컨테이너는 살아있는 상태가 아니라고 판단하고 해당 컨테이너를 종료시킨 후 컨테이너를 다시 생성한다.

Pod 전체를 지우는 게 아니라 컨테이너만 재시작한다. 즉, Pod IP는 유지, Pod는 그대로 유지, 컨테이너만 새로 생성한다.

이게 바로 자동 복구의 시작점이다.

### Self-healing Pod란 무엇인가

Self-healing Pod는 특정 기능이 아니라 개념이다. 문제가 발생했을 때 사람 개입 없이 쿠버네티스가 스스로 정상 상태로 되돌리는 구조다. Self-healing은 한 가지 기능으로 이루어지지 않는다(여러 요소가 합쳐진 결과).

**Self-healing이 이루어지는 실제 흐름**

하나의 시나리오를 살펴본다.
1. Pod 안의 웹 애플리케이션이 멈춘다
2. 컨테이너는 떠 있지만 응답이 없다
3. kubelet이 livenessProbe 체크
4. probe 실패 감지
5. kubelet이 컨테이너 종료
6. 새 컨테이너 생성
7. 서비스 정상화

이 과정에서 관리자는 아무것도 하지 않았다. 이게 바로 Self-healing Pod의 핵심이다.

### livenessProbe가 Self-healing의 핵심인 이유

livenessProbe가 없다면:
1. 컨테이너는 멈췄다, 하지만 쿠버네티스는 "컨테이너가 떠 있으니까 괜찮겠지"라고 판단할 수 있다.
2. Pod는 계속 Running
3. 서비스는 장애 상태

즉, 자동 복구 자체가 일어나지 않는다.

### livenessProbe와 readinessProbe의 결정적 차이

- **readinessProbe** — "지금 이 컨테이너가 요청을 받아도 돼?"
- **livenessProbe** — "이 컨테이너, 살아있긴 한 거야?"

livenessProbe와 readinessProbe는 질문이 다르기 때문에 결과도 다르다.
- readiness 실패 : 트래픽만 차단
- liveness 실패 : 컨테이너 재시작

Self-healing은 livenessProbe가 있어야만 가능하다.

---

## 2. livenessProbe 메커니즘 — 3가지 방식과 실습

livenessProbe는 컨테이너가 정상적으로 살아있는지를 쿠버네티스가 주기적으로 확인하는 기능이다. 정상으로 판단되지 않으면 해당 컨테이너를 종료하고 다시 시작한다.

**1) httpGet probe**

- 지정한 IP 주소, port, path로 HTTP GET 요청을 보낸다.
- HTTP 응답 코드가 200번대이면 정상으로 판단한다.
- 응답이 없거나 오류 코드가 반환되면 비정상으로 판단하고 컨테이너를 다시 시작한다.
- nginx, apache, spring boot, node.js 같은 웹 애플리케이션은 항상 HTTP 요청을 받아서 응답해야 정상이다.

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

**2) tcpSocket probe**

- 지정된 port로 TCP 연결을 시도 후 TCP 연결이 성공하면 정상으로 판단한다.
- 연결이 실패하면 비정상으로 판단하고 컨테이너를 다시 시작한다.
- HTTP 응답까지는 필요 없고 서비스가 떠 있는지만 확인하면 되는 경우에 사용한다.
  - MySQL, MariaDB, PostgreSQL
  - Redis
  - SSH 데몬
  - 단순 TCP 서버

```yaml
livenessProbe:
  tcpSocket:
    port: 22
```

**3) exec probe**

- 컨테이너 내부에서 지정한 명령어를 실행한다.
- 명령어 실행 결과의 종료 코드(exit code)가 0이면 정상으로 판단한다.
- 종료 코드가 0이 아니면 비정상으로 판단하고 컨테이너를 다시 시작한다.
- 컨테이너 내부 상태까지 직접 검사해야 할 때 쓰는 방식이다.
  - 특정 파일이 존재해야 정상
  - 특정 프로세스가 떠 있어야 정상
  - 내부 스크립트 결과가 정상이어야 정상

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - cat /data/file
```

### httpGet livenessProbe 실습 — nginx

```
[root@k8s-master ~]# vi pod-nginx-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-liveness
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    ports:
    - containerPort: 80
      protocol: TCP
    livenessProbe:
      httpGet:
        path: /
        port: 80


[root@k8s-master ~]# kubectl  apply  -f  pod-nginx-liveness.yaml  --dry-run=server
pod/nginx-pod-liveness created (server dry run)




[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch



[root@k8s-master ~]# kubectl  apply  -f  pod-nginx-liveness.yaml
pod/nginx-pod-liveness created



[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                 	READY   STATUS    	RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
nginx-pod-liveness	0/1         Pending   		0                0s      <none>        <none>         <none>                   <none>
nginx-pod-liveness	0/1         Pending   		0                0s      <none>        k8s-worker1   <none>                  <none>
nginx-pod-liveness 	0/1         ContainerCreating	0                0s      <none>        k8s-worker1   <none>                  <none>
nginx-pod-liveness	1/1         Running             	0                1s      10.244.1.11   k8s-worker1   <none>                  <none>




[root@k8s-master ~]# kubectl  describe  pods  nginx-pod-liveness | grep Liveness
    Liveness:       http-get http://:80/ delay=0s timeout=1s period=10s #success=1 #failure=3
```

**livenessProbe의 세부 옵션**

| 옵션 | 의미 | 기본값 |
|---|---|---|
| initialDelaySeconds | 컨테이너 시작 후 몇 초 뒤부터 probe 시작 | 0초 |
| timeoutSeconds | 몇 초 안에 응답 없으면 실패로 판단 | 1초 |
| periodSeconds | 몇 초마다 livenessProbe 실행 | 10초 |
| successThreshold | 몇 번 성공하면 정상으로 판단 | 1 |
| failureThreshold | 몇 번 연속 실패하면 컨테이너 재시작 | 3 |

```
[root@k8s-master ~]# vi pod-nginx-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-liveness
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    ports:
    - containerPort: 80
      protocol: TCP
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 1
      failureThreshold: 3
      successThreshold: 1


[root@k8s-master ~]# kubectl  delete  -f  pod-nginx-liveness.yaml
pod "nginx-pod-liveness" deleted from default namespace (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  pod-nginx-liveness.yaml  --dry-run=server
pod/nginx-pod-liveness created (server dry run)



[root@k8s-master ~]# kubectl  apply  -f  pod-nginx-liveness.yaml
pod/nginx-pod-liveness created




[root@k8s-master ~]# kubectl  describe  pods  nginx-pod-liveness | grep Liveness
    Liveness:       http-get http://:80/ delay=10s timeout=1s period=10s #success=1 #failure=3
```

컨테이너 내부에서 nginx 프로세스를 직접 중지시켜 livenessProbe 실패 → 컨테이너 재시작을 확인한다.

```
[root@k8s-master ~]# kubectl  exec  nginx-pod-liveness  -it  -- /bin/bash
root@nginx-pod-liveness:/#
root@nginx-pod-liveness:/# nginx  -s  stop
2026/08/14 04:06:36 [notice] 40#40: signal process started
root@nginx-pod-liveness:/# command terminated with exit code 137



	# 메인 프로세스가 삭제되어 파드가 재시작
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                	READY   STATUS      RESTARTS   	AGE     IP            	  NODE           NOMINATED NODE   READINESS GATES
nginx-pod-liveness	1/1         Running      0          	73s       10.244.1.12	  k8s-worker1   <none>                   <none>
nginx-pod-liveness 	0/1         Completed   0          	2m32s   10.244.1.12	  k8s-worker1   <none>                   <none>
nginx-pod-liveness 	1/1         Running      1 (1s ago)   	2m33s   10.244.1.12	  k8s-worker1   <none>                   <none>
```

path를 실제로 존재하지 않는 경로(`/usr/share/nginx/html/health`)로 바꾸면, 파일을 직접 만들어두지 않는 한 probe가 실패한다.

```
[root@k8s-master ~]# vi pod-nginx-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-liveness
spec:
  containers:
  - name: nginx-container
    image: nginx:1.31
    ports:
    - containerPort: 80
      protocol: TCP
    livenessProbe:
      httpGet:
        path: /usr/share/nginx/html/health		# 경로 변경
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 1
      failureThreshold: 3
      successThreshold: 1



[root@k8s-master ~]# kubectl  exec  nginx-pod-liveness  -- sh -c "echo OK >  /usr/share/nginx/html/health"


[root@k8s-master ~]# kubectl  delete  -f  pod-nginx-liveness.yaml
pod "nginx-pod-liveness" deleted from default namespace


[root@k8s-master ~]# kubectl  apply  -f  pod-nginx-liveness.yaml
pod/nginx-pod-liveness created



[root@k8s-master ~]# kubectl  exec  nginx-pod-liveness  -it  -- /bin/bash


root@nginx-pod-liveness:/# ls -l  /usr/share/nginx/html/
total 12
-rw-r--r-- 1 root root 497 Jul 15 16:03 50x.html
-rw-r--r-- 1 root root   3 Aug 14 04:12 health
-rw-r--r-- 1 root root 896 Jul 15 16:03 index.html


root@nginx-pod-liveness:/# cat  /usr/share/nginx/html/health
OK

root@nginx-pod-liveness:/# exit

[root@k8s-master ~]# kubectl  exec  nginx-pod-liveness  --  rm  /usr/share/nginx/html/health
```

### EX1) 커스텀 이미지에 health 파일을 내장하여 livenessProbe 실습

**EX1-1) nginx:1.31 이미지를 기반으로 사용자 정의 이미지를 생성하고, 이미지 내부에 `/usr/share/nginx/html/health` 파일이 기본으로 존재하도록 Dockerfile을 작성하시오**

```
[root@k8s-master ~]# mkdir liveness-lab


[root@k8s-master ~]# cd liveness-lab


[root@k8s-master liveness-lab]# pwd
/root/liveness-lab



[root@k8s-master liveness-lab]# vi health
health OK



[root@k8s-master liveness-lab]# vi dockerfile
FROM  nginx:1.31

ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

COPY  health  /usr/share/nginx/html/health
```

**EX1-2) EX1-1에서 생성한 이미지를 빌드하고 Docker Hub에 업로드하시오**

```
[root@k8s-master liveness-lab]# docker  build  -t  konan7979/nginx-liveness:1.0  .
[+] Building 1.9s (8/8) FINISHED                                                                                	docker:default
 => [internal] load build definition from dockerfile                                                                	0.0s
 => => transferring dockerfile: 136B                                                                                   	0.0s
 => [internal] load metadata for docker.io/library/nginx:1.31                                                  	1.8s
 => [auth] library/nginx:pull token for registry-1.docker.io                                                  	0.0s
 => [internal] load .dockerignore                                                                                  	0.0s
 => => transferring context: 2B                                                                                   	0.0s
 => [internal] load build context                                                                                         	0.0s
 => => transferring context: 43B                                                                                     	0.0s
 => CACHED [1/2] FROM docker.io/library/nginx:1.31@sha256:8541484afbc9c8a5a8a99b379568e	0.0s
 => => resolve docker.io/library/nginx:1.31@sha256:8541484afbc9c8a5a8a99b379568ebbc957f6585	0.0s
 => [2/2] COPY  health  /usr/share/nginx/html/health                                                         	0.0s
 => exporting to image                                                                                                  	0.1s
 => => exporting layers                                                                                                	0.0s
 => => exporting manifest sha256:a12719dc7b15eba3da57e90bdc60568c95613bcffa582a07ec31f7cb7	0.0s
 => => exporting config sha256:3a5175eb555a100852c3a22fa2583d6f1c9a0812c23aa5f07eda32b1fcab	0.0s
 => => exporting attestation manifest sha256:6e76d6c1769305225cd3afbe40e095a3567dc37528e72f	0.0s
 => => exporting manifest list sha256:75edbab6ae2a0b81b19ee784deddc4cff5afe7c81fcd789c379751	0.0s
 => => naming to docker.io/konan7979/nginx-liveness:1.0                                                       	0.0s
 => => unpacking to docker.io/konan7979/nginx-liveness:1.0                                                     	0.0s



[root@k8s-master liveness-lab]# docker  images
IMAGE                            	 	ID             	DISK USAGE	CONTENT SIZE   EXTRA
custom-nginx-web:1.31             	02218a3d52d6        	348MB           	97MB
konan7979/custom-nginx-web:1.31	02218a3d52d6        	348MB           	97MB
konan7979/nginx-liveness:1.0      	75edbab6ae2a        	235MB         	63.1MB



[root@k8s-master liveness-lab]# docker login
Authenticating with existing credentials... [Username: konan7979]

i Info → To login with a different account, run 'docker logout' followed by 'docker login'

Login Succeeded




[root@k8s-master liveness-lab]# docker  push konan7979/nginx-liveness:1.0
The push refers to repository [docker.io/konan7979/nginx-liveness]
6f8ee52d8f41: Pushed
44136fa355b3: Mounted from konan7979/ssh-probe
3c55dc422a81: Mounted from konan7979/custom-nginx-web
26c307b5e35a: Mounted from konan7979/custom-nginx-web
d84ae7b21412: Mounted from konan7979/custom-nginx-web
c0df8d325117: Mounted from konan7979/custom-nginx-web
b8b80b9bc028: Mounted from konan7979/custom-nginx-web
f5de6e85ac74: Mounted from konan7979/custom-nginx-web
5a4222b844e8: Mounted from konan7979/custom-nginx-web
df2401a872d9: Pushed
1.0: digest: sha256:75edbab6ae2a0b81b19ee784deddc4cff5afe7c81fcd789c379751a1679bf7a5 size: 856
```

**EX1-3) EX1-2에서 업로드한 이미지를 사용하는 Pod를 생성하고, `/health` 경로를 검사하는 livenessProbe를 설정하시오**

**EX1-4) 생성한 Pod가 정상 실행되고 `/health` 경로의 Probe가 성공하는 것을 확인하시오**

조건:

| 항목 | 값 |
|---|---|
| Pod 이름 | nginx-pod-liveness |
| Container 이름 | nginx |
| Probe 방식 | httpGet |
| Path | /health |
| Port | 80 |
| 초기 대기시간 | 10초 |
| 검사 주기 | 5초 |
| Timeout | 1초 |
| 연속 실패 횟수 | 3회 |

```
[root@k8s-master ~]# vi pod-nginx-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-liveness

spec:
  containers:
  - name: nginx
    image: konan7979/nginx-liveness:1.0

    ports:
    - containerPort: 80

    livenessProbe:
      httpGet:
        path: /health
        port: 80
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3


[root@k8s-master liveness-lab]# kubectl  apply  -f  pod-nginx-liveness.yaml  --dry-run=server
pod/nginx-pod-liveness created (server dry run)



[root@k8s-master liveness-lab]# kubectl  apply  -f  pod-nginx-liveness.yaml
pod/nginx-pod-liveness created




[root@k8s-master liveness-lab]# kubectl  get  pods
NAME                 	READY   STATUS                RESTARTS   AGE
nginx-pod-liveness	0/1         ContainerCreating   0                 2s



[root@k8s-master liveness-lab]# kubectl  get  pods
NAME                 READY   STATUS    RESTARTS   AGE
nginx-pod-liveness   1/1      Running     0                 9s



[root@k8s-master liveness-lab]# kubectl  describe  pods | grep Liveness
    Liveness:       http-get http://:80/health delay=10s timeout=1s period=5s #success=1 #failure=3



[root@k8s-master liveness-lab]# kubectl  exec  nginx-pod-liveness   --  curl  http://localhost/health
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    10  100    10    0     0  23696      0 --:--:-- --:--:-- --:--:-- 10000
health OK
```

**EX1-5) 실행 중인 컨테이너에서 `/usr/share/nginx/html/health` 파일을 삭제하여 livenessProbe가 실패하도록 하시오**

```
[root@k8s-master liveness-lab]# watch -n 1 kubectl get pods
Every 1.0s: kubectl get pods                               k8s-master: Fri Aug 14 15:29:19 2026

NAME                    READY   STATUS    RESTARTS   AGE
nginx-pod-liveness   1/1         Running     0                6m48s


		OR

[root@k8s-master liveness-lab]# kubectl get pod nginx-pod-liveness --watch
NAME                    READY   STATUS    RESTARTS   AGE
nginx-pod-liveness   1/1         Running     0                6m48s




	# 컨테이너 접속
[root@k8s-master liveness-lab]# kubectl exec nginx-pod-liveness   -it  --  /bin/bash
root@nginx-pod-liveness:/#
root@nginx-pod-liveness:/# ls  -l /usr/share/nginx/html/
total 12
-rw-r--r-- 1 root root 497 Jul 15 16:03 50x.html
-rw-r--r-- 1 root root  10 Aug 14 06:08 health		<--- 파일 확인
-rw-r--r-- 1 root root 896 Jul 15 16:03 index.html


root@nginx-pod-liveness:/# rm  -rf  /usr/share/nginx/html/health



	# health 파일이 없기때문에 health-check에 실패 (컨테이너 재실행으로 인해 컨테이너에서 자동으로 아웃)
root@nginx-pod-liveness:/# ls  -l /usr/share/nginx/html/
total 12
-rw-r--r-- 1 root root 497 Jul 15 16:03 50x.html
-rw-r--r-- 1 root root 896 Jul 15 16:03 index.html




[root@k8s-master liveness-lab]# kubectl  get  pods
NAME                    READY   STATUS    RESTARTS      AGE
nginx-pod-liveness   1/1         Running     1 (55s ago)      11m

-health-check에 실패했기 때문에 컨테이너를 재실행한다.




[root@k8s-master liveness-lab]# kubectl  describe  pods nginx-pod-liveness | tail -10
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  13m                    default-scheduler  Successfully assigned default/nginx-pod-liveness to k8s-worker1
  Normal   Pulling    13m                    kubelet            spec.containers{nginx}: Pulling image "konan7979/nginx-liveness:1.0"
  Normal   Pulled     12m                    kubelet            spec.containers{nginx}: Successfully pulled image "konan7979/nginx-liveness:1.0" in 6.638s (6.638s including waiting). Image size: 63126097 bytes.
  Normal   Created    2m13s (x2 over 12m)    kubelet            spec.containers{nginx}: Container created
  Normal   Started    2m13s (x2 over 12m)    kubelet            spec.containers{nginx}: Container started
  Warning  Unhealthy  2m13s (x3 over 2m23s)  kubelet            spec.containers{nginx}: Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    2m13s                  kubelet            spec.containers{nginx}: Container nginx failed liveness probe, will be restarted
  Normal   Pulled     2m13s                  kubelet            spec.containers{nginx}: Container image "konan7979/nginx-liveness:1.0" already present on machine and can be accessed by the pod
```

**EX1-6) livenessProbe가 연속 실패한 후 컨테이너가 자동으로 재시작되는 것을 확인하시오**

```
[root@k8s-master liveness-lab]# kubectl exec nginx-pod-liveness   -it  --  /bin/bash
root@nginx-pod-liveness:/#
root@nginx-pod-liveness:/# ls  -l /usr/share/nginx/html/
total 12
-rw-r--r-- 1 root root 497 Jul 15 16:03 50x.html
-rw-r--r-- 1 root root  10 Aug 14 06:08 health
-rw-r--r-- 1 root root 896 Jul 15 16:03 index.html

-다시 컨테이너에 접속하게되면 이미지로 컨테이너를 다시 만들기 때문에 health 파일이 확인된다.
```

### EX2) smlinux/unhealthy 이미지로 실패 누적 후 재시작 확인

`smlinux/unhealthy`는 Kubernetes의 livenessProbe 실습용 교육 이미지다.

일반적인 nginx 이미지처럼 계속 정상 응답하는 것이 아니라, 일정 조건 이후 일부러 비정상 응답을 반환하도록 만들어졌다.

livenessProbe, readinessProbe, self-healing 같은 Kubernetes의 상태 점검과 자동 복구 기능을 설명하기 위한 데모 용도로 사용한다.

실제 서비스 운영용 이미지가 아니라, Kubernetes가 컨테이너 장애를 감지하고 재시작하는 과정을 확인하기 위한 실습용 이미지다.

HTTP 요청이 5번까지는 200 OK로 정상 응답하지만, 6번째 요청부터는 500 Internal Server Error를 반환하도록 동작한다. 따라서 livenessProbe가 주기적으로 HTTP 요청을 보내면, 처음에는 성공하다가 이후 실패가 누적되고, failureThreshold 조건을 만족하면 kubelet이 해당 컨테이너를 재시작한다.

```
[root@k8s-master ~]# vi  pod-nginx-unhealthy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: unhealthy-pod-liveness
spec:
  containers:
  - name: unhealthy-container
    image: smlinux/unhealthy
    ports:
    - containerPort: 80
      protocol: TCP
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3



[root@k8s-master ~]# kubectl  apply -f  pod-nginx-unhealthy.yaml  --dry-run=server
pod/unhealthy-pod-liveness created (server dry run)



[root@k8s-master ~]# kubectl  apply -f  pod-nginx-unhealthy.yaml
pod/unhealthy-pod-liveness created



[root@k8s-master ~]# kubectl  get  pods
NAME                     	READY   	STATUS              	RESTARTS      AGE
nginx-pod-liveness      	1/1     	Running             	1 (24m ago)      34m
unhealthy-pod-liveness	0/1     	ContainerCreating	0                   14s



[root@k8s-master ~]# kubectl  get  pods
NAME                     	READY   	STATUS      RESTARTS     	AGE
nginx-pod-liveness       	1/1     	Running       1 (24m ago)	35m
unhealthy-pod-liveness  	1/1     	Running       0                   	58s



[root@k8s-master ~]# kubectl  get  pods
NAME                     	READY 	STATUS      RESTARTS      	AGE
nginx-pod-liveness       	1/1     	Running       1 (26m ago)   	36m
unhealthy-pod-liveness   	1/1     	Running       1 (5s ago)    	2m15s



[root@k8s-master ~]# kubectl  describe  pods | tail -10
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  3m24s                default-scheduler  Successfully assigned default/unhealthy-pod-liveness to k8s-worker2
  Normal   Pulled     2m53s                kubelet  	  spec.containers{unhealthy-container}: Successfully pulled image "smlinux/unhealthy" in 31.181s (31.181s including waiting). Image size: 263841919 bytes.
  Normal   Pulling    74s (x2 over 3m24s)  kubelet	  spec.containers{unhealthy-container}: Pulling image "smlinux/unhealthy"
  Normal   Created    72s (x2 over 2m53s)  kubelet	  spec.containers{unhealthy-container}: Container created
  Normal   Started    72s (x2 over 2m53s)  kubelet	  spec.containers{unhealthy-container}: Container started
  Normal   Pulled     72s                  kubelet     	  spec.containers{unhealthy-container}: Successfully pulled image "smlinux/unhealthy" in 1.515s (1.515s including waiting). Image size: 263841919 bytes.
  Warning  Unhealthy  4s (x6 over 2m24s)   kubelet	  spec.containers{unhealthy-container}: Liveness probe failed: Get "http://10.244.2.8:80/": dial tcp 10.244.2.8:80: connect: connection refused
  Normal   Killing    4s (x2 over 104s)    kubelet	  spec.containers{unhealthy-container}: Container unhealthy-container failed liveness probe, will be restarted



[root@k8s-master ~]# kubectl  get  pods
NAME                     	READY  	STATUS    RESTARTS      AGE
nginx-pod-liveness       	1/1     	Running     1 (28m ago)      39m
unhealthy-pod-liveness	1/1     	Running     2 (46s ago)       4m36s




[root@k8s-master ~]# kubectl  delete  pods nginx-pod-liveness
pod "nginx-pod-liveness" deleted from default namespace


[root@k8s-master ~]# kubectl  delete  unhealthy-pod-liveness
error: the server doesn't have a resource type "unhealthy-pod-liveness"
```

### EX3) tcpSocket Probe를 이용한 SSH 서비스 장애 감지 및 자동 복구

- SSH 서버가 실행되는 Pod를 생성한다.
- SSH 서비스는 기본적으로 22번 포트를 사용하며, tcpSocket 방식의 livenessProbe를 이용하여 22번 포트가 정상적으로 열려 있는지 확인한다.
- 컨테이너 내부에 접속하여 `/etc/ssh/sshd_config` 파일의 SSH 포트를 수동으로 22 → 2222로 변경하고 SSH 서비스를 다시 시작한다.

1. SSH가 22번 포트를 사용할 때 tcpSocket Probe 성공
2. SSH 포트를 2222번으로 변경
3. tcpSocket Probe는 계속 22번 포트를 검사
4. Probe 실패
5. Kubernetes가 컨테이너를 재시작
6. 컨테이너가 이미지의 원래 설정인 SSH 22번으로 시작
7. tcpSocket Probe 다시 성공

- Pod 이름: ssh-probe
- 컨테이너 이름: ssh-server
- SSH 기본 Port: 22

```
	# 작업 디렉터리 생성

[root@k8s-master ~]# mkdir tcp-probe-lab

[root@k8s-master ~]# cd tcp-probe-lab

[root@k8s-master tcp-probe-lab]# pwd
/root/tcp-probe-lab




	# Dockerfile 생성

[root@k8s-master tcp-probe-lab]# vi dockerfile
FROM rockylinux:9

RUN dnf install -y openssh-server iproute procps-ng vim-minimal && \
    dnf clean all && \
    ssh-keygen -A

EXPOSE 22

CMD ["/bin/bash", "-c", "/usr/sbin/sshd && exec tail -f /dev/null"]

: wq


-tail -f /dev/null = 컨테이너를 계속 실행시키는 메인 프로세스 (실습 중 sshd를 중지해도 컨테이너 자체는 바로 종료되지 않는다.)




	# Image Build 및 hub.docker.com에 이미지 PUSH

[root@k8s-master tcp-probe-lab]# docker  build  -t  konan7979/ssh-probe:1.0  .
[+] Building 1.9s (7/7) FINISHED                                                 docker:default
 => [internal] load build definition from dockerfile                                       0.0s
 => => transferring dockerfile: 248B                                                       0.0s
 => [internal] load metadata for docker.io/library/rockylinux:9                            1.6s
 => [auth] library/rockylinux:pull token for registry-1.docker.io                          0.0s
 => [internal] load .dockerignore                                                          0.0s
 => => transferring context: 2B                                                            0.0s
 => [1/2] FROM docker.io/library/rockylinux:9@sha256:d7be1c094cc5845ee815d4632fe377514ee6  0.0s
 => => resolve docker.io/library/rockylinux:9@sha256:d7be1c094cc5845ee815d4632fe377514ee6  0.0s
 => CACHED [2/2] RUN dnf install -y openssh-server iproute procps-ng vim-minimal &&     d  0.0s
 => exporting to image                                                                     0.3s
 => => exporting layers                                                                    0.0s
 => => exporting manifest sha256:833686fdd0e0174c5900eaf080cb9502ceada2452682ad4d04f3acad  0.0s
 => => exporting config sha256:23bcbc653110eb6719c37f59268047d99985d845ee4d48c7f9186664b6  0.0s
 => => exporting attestation manifest sha256:ee0a74b1440c8d271aa1df2fc613a93598c07529e4f1  0.0s
 => => exporting manifest list sha256:6578cbc133e376b889836ab3406bed26cad7d5941bd3f792c75  0.0s
 => => naming to docker.io/konan7979/ssh-probe:1.0                                         0.0s
 => => unpacking to docker.io/konan7979/ssh-probe:1.0                                      0.3s




[root@k8s-master tcp-probe-lab]# docker  push  konan7979/ssh-probe:1.0
The push refers to repository [docker.io/konan7979/ssh-probe]
4dfcf8f9dcf3: Pushed
44136fa355b3: Already exists
83781c17e4da: Layer already exists
446f83f14b23: Layer already exists
1.0: digest: sha256:6578cbc133e376b889836ab3406bed26cad7d5941bd3f792c75a2ae220c9a86c size: 855
[root@k8s-master tcp-probe-lab]#




	# tcpSocket Probe Pod 작성

[root@k8s-master ~]# vi ssh-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssh-probe

spec:
  containers:
  - name: ssh-server
    image: konan7979/ssh-probe:1.0
    ports:
    - containerPort: 22

    livenessProbe:
      tcpSocket:
        port: 22
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3

: wq



[root@k8s-master tcp-probe-lab]# kubectl  apply  -f  ssh-probe.yaml  --dry-run=server
pod/ssh-probe created (server dry run)


[root@k8s-master tcp-probe-lab]# kubectl  apply  -f  ssh-probe.yaml
pod/ssh-probe created



[root@k8s-master tcp-probe-lab]# kubectl  get  pods
NAME     	  READY	  STATUS      	  RESTARTS    AGE
ssh-probe	  0/1     	  ContainerCreating	  0                 10s


[root@k8s-master tcp-probe-lab]# kubectl  get  pods
NAME     	  READY	  STATUS      RESTARTS    AGE
ssh-probe	  1/1     	  Running       0                  34s



[root@k8s-master tcp-probe-lab]# kubectl  describe  pods | grep Liveness
    Liveness:       tcp-socket :22 delay=10s timeout=1s period=5s #success=1 #failure=3
```

```
	# 컨테이너 접속
[root@k8s-master tcp-probe-lab]# kubectl  exec  ssh-probe -it  -- /bin/bash
[root@ssh-probe /]#


	# 현재 서비스중인 sshd의 port 번호 확인 (TCP 22)
[root@ssh-probe /]# ss  -lntp | grep sshd
LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=8,fd=7))
LISTEN 0      128              [::]:22            [::]:*    users:(("sshd",pid=8,fd=8))


-현재 sshd가 TCP 22 port로 서비스 하기때문에 liveness probe에의해 health-check되고 있다.

-이때 sshd의 서비스 port 번호를  TCP 22번이 아닌 다른 port 번호로 변경하게되면 health-check가 실패하고 컨테이너를 재시작한다.




[root@ssh-probe /]# vi  /etc/ssh/sshd_config		# sshd의 서비스 port번호를 변경하기위해서 sshd_config 파일 변경
      1 #       $OpenBSD: sshd_config,v 1.104 2021/07/02 05:11:21 dtucker Exp $
      2
      3 # This is the sshd server system-wide configuration file.  See
      4 # sshd_config(5) for more information.
      5
      6 # This sshd was compiled with PATH=/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
      7
      8 # The strategy used for options in the default sshd_config shipped with
      9 # OpenSSH is to specify options with their default value where
      10 # possible, but leave them commented.  Uncommented options override the
      11 # default value.
      12
      13 # To modify the system-wide sshd configuration, create a  *.conf  file under
      14 #  /etc/ssh/sshd_config.d/  which will be automatically included below
      15 Include /etc/ssh/sshd_config.d/*.conf
      16
      17 # If you want to change the port on a SELinux system, you have to tell
      18 # SELinux about this change.
      19 # semanage port -a -t ssh_port_t -p tcp #PORTNUMBER
      20 #
      21 Port 2222			# 주석 해제 후 port번호 변경
      22 #AddressFamily any
      23 #ListenAddress 0.0.0.0
      24 #ListenAddress ::

:wq


	# 두번째 터미널에서 확인
[root@k8s-master liveness-lab]# kubectl  get  pods  -o  wide  --watch

NAME        READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
ssh-probe   1/1     Running   0          23m   10.244.1.17   k8s-worker1   <none>           <none>





[root@k8s-master tcp-probe-lab]# kubectl  exec  ssh-probe -it  -- /bin/bash
[root@ssh-probe /]#



[root@ssh-probe /]# pkill  sshd		# sshd 프로세스 중지

[root@ssh-probe /]# /usr/sbin/sshd		# sshd를 다시 실행하게되면 위에서 작성한 TCP 2222가 적용된다.



[root@ssh-probe /]#  ss  -lntp | grep sshd
LISTEN 0      128          0.0.0.0:2222      0.0.0.0:*    users:(("sshd",pid=70,fd=7))
LISTEN 0      128              [::]:2222          [::]:*    users:(("sshd",pid=70,fd=8))




	# 두번째 터미널에서 확인
[root@k8s-master liveness-lab]# kubectl  get  pods  -o  wide  --watch
NAME        READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
ssh-probe   1/1     Running   0          23m   10.244.1.17   k8s-worker1   <none>           <none>
ssh-probe   1/1     Running   1 (0s ago)   28m   10.244.1.17   k8s-worker1   <none>           <none>



root@ssh-probe /]# command terminated with exit code 137		# 컨테이너가 다시 생성되기때문에 접속한 컨테이너에서 강제로 out된다.
[root@k8s-master tcp-probe-lab]#




	# 다시 컨테이너 접속
[root@k8s-master tcp-probe-lab]# kubectl  exec  ssh-probe -it  -- /bin/bash
[root@ssh-probe /]#


	# 현재 서비스중인 sshd의 port 번호 확인 (TCP 22)
[root@ssh-probe /]# ss  -lntp | grep sshd
LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=8,fd=7))
LISTEN 0      128              [::]:22            [::]:*    users:(("sshd",pid=8,fd=8))
```

컨테이너가 재시작되면 이미지의 원래 설정(Port 22)으로 다시 시작되므로 sshd_config에서 변경했던 2222 설정은 사라지고 다시 22번 포트로 서비스된다.

### 실습 3) exec Probe로 healthy 파일 존재 여부 검사

LivenessProbe의 exec 방식을 이용하여 컨테이너 내부의 `/usr/share/nginx/html/healthy` 파일 존재 여부를 검사하고, 파일이 없을 경우 Probe 실패로 컨테이너가 자동 재시작되는 것을 확인한다.

이후 컨테이너 내부에 healthy 파일을 생성하여 Probe가 정상 상태로 복구되고 더 이상 컨테이너가 재시작되지 않는 것을 확인한다.

해당 파일을 다시 삭제하여 livenessProbe 실패와 컨테이너 재시작이 다시 발생하는지 확인한다.

```
	# 실습 디렉터리 생성

[guest@k8s-master ~]# mkdir exec-liveness-lab

[guest@k8s-master ~]# cd exec-liveness-lab


[guest@k8s-master exec-liveness-lab]# pwd
/home/guest/exec-liveness-lab



[root@k8s-master exec-liveness-lab]# vi healthy
Liveness OK


[root@k8s-master exec-liveness-lab]# ls  -l
합계 4
-rw-r--r-- 1 root root 11  8월 14 16:58 healthy




	# Dockerfile 작성
[root@k8s-master exec-liveness-lab]# vi dockerfile
FROM nginx:1.31

COPY healthy /usr/share/nginx/html/healthy

:wq




	# 이미지 빌드 및 이미지 PUSH
[root@k8s-master exec-liveness-lab]# docker  build  -t  konan7979/nginx-exec-liveness:1.0  .




root@k8s-master exec-liveness-lab]# docker  push  konan7979/nginx-exec-liveness:1.0
The push refers to repository [docker.io/konan7979/nginx-exec-liveness]
3c55dc422a81: Mounted from konan7979/nginx-liveness
26c307b5e35a: Mounted from konan7979/nginx-liveness
51a7089ca9bb: Pushed
44136fa355b3: Mounted from konan7979/ssh-probe
d84ae7b21412: Mounted from konan7979/nginx-liveness
c0df8d325117: Mounted from konan7979/nginx-liveness
b8b80b9bc028: Mounted from konan7979/nginx-liveness
f5de6e85ac74: Mounted from konan7979/nginx-liveness
5a4222b844e8: Mounted from konan7979/nginx-liveness
0af710fe68e2: Pushed
1.0: digest: sha256:caeb48d1c8b7df9b94553b4f5197ac52af38dc0e3f0beac7c6a4b22167146166 size: 856




	# exec 방식의 livenessProbe Pod 작성

[root@k8s-master exec-liveness-lab]# cd ..



[root@k8s-master ~]# vi exec-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-exec-liveness

spec:
  containers:
  - name: nginx
    image: konan7979/nginx-exec-liveness:1.0

    livenessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - 'test -f /usr/share/nginx/html/healthy && echo "$(date) : Liveness OK"  >>  /tmp/liveness.log'
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3

:wq


[root@k8s-master ~]# kubectl  apply  -f  exec-liveness.yaml  --dry-run=server
pod/nginx-exec-liveness created (server dry run)



[root@k8s-master ~]# kubectl  get  pods  -n  soldesk
NAME               	  READY	  STATUS    RESTARTS   AGE
nginx-exec-liveness	  1/1    	  Running     0                 23s



[root@k8s-master ~]# kubectl describe pod nginx-exec-liveness | grep Liveness
    Liveness:       exec [/bin/sh -c test -f /usr/share/nginx/html/healthy && echo "Liveness OK"  >>  /tmp/liveness.log] delay=5s timeout=1s period=5s #success=1 #failure=3




	# 두번째 터미널에서 확인
[root@k8s-master ~]# kubectl  get  pods  -o  wide  -n soldesk  --watch
NAME               	  READY	  STATUS    RESTARTS   AGE      IP             NODE           NOMINATED NODE   READINESS GATES
nginx-exec-liveness	  1/1    	  Running     0                 2m24s   10.244.2.9   k8s-worker2   <none>                   <none>




[root@k8s-master ~]# kubectl  exec  nginx-exec-liveness   -it  --  /bin/bash
root@nginx-exec-liveness:/#



root@nginx-exec-liveness:/# ls -l /tmp
total 4
-rw-r--r-- 1 root root 43 Aug 14 08:25 liveness.log



root@nginx-exec-liveness:/# cat /tmp/liveness.log
Fri Aug 14 08:25:22 UTC 2026 : Liveness OK
Fri Aug 14 08:25:27 UTC 2026 : Liveness OK
Fri Aug 14 08:25:32 UTC 2026 : Liveness OK
Fri Aug 14 08:25:37 UTC 2026 : Liveness OK
Fri Aug 14 08:25:42 UTC 2026 : Liveness OK
Fri Aug 14 08:25:47 UTC 2026 : Liveness OK
Fri Aug 14 08:25:52 UTC 2026 : Liveness OK
Fri Aug 14 08:25:57 UTC 2026 : Liveness OK
Fri Aug 14 08:26:02 UTC 2026 : Liveness OK
Fri Aug 14 08:26:07 UTC 2026 : Liveness OK
Fri Aug 14 08:26:12 UTC 2026 : Liveness OK
```

```
	# 두번째 터미널에서 확인
[root@k8s-master ~]#  kubectl  get  pods  -o  wide  --watch
NAME                    READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
nginx-exec-liveness   1/1        Running     0                4m12s   10.244.1.22   k8s-worker1   <none>                    <none>




	# exec Probe 실패 확인

[root@k8s-master ~]# kubectl  exec  nginx-exec-liveness   -it  --  /bin/bash
root@nginx-exec-liveness:/#
root@nginx-exec-liveness:/# rm  -rf  /usr/share/nginx/html/healthy
root@nginx-exec-liveness:/#
root@nginx-exec-liveness:/# command terminated with exit code 137		# exit code 137
[root@k8s-master ~]#




	# 두번째 터미널에서 확인
[root@k8s-master ~]#  kubectl  get  pods  -o  wide  --watch
NAME                    READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
nginx-exec-liveness   1/1        Running     0                4m12s   10.244.1.22   k8s-worker1   <none>                    <none>
nginx-exec-liveness   1/1        Running     1 (0s ago)     5m21s   10.244.1.22   k8s-worker1   <none>                    <none>



	# 컨테이너 재생성 후 확인
[root@k8s-master ~]# kubectl  exec  nginx-exec-liveness   -it  --  /bin/bash
root@nginx-exec-liveness:/#
root@nginx-exec-liveness:/#


root@nginx-exec-liveness:/# ls  -l  /tmp/
total 4
-rw-r--r-- 1 root root 516 Aug 14 08:31 liveness.log



root@nginx-exec-liveness:/# ls  -l  /usr/share/nginx/html/
total 12
-rw-r--r-- 1 root root 497  Jul  15 16:03 50x.html
-rw-r--r-- 1 root root  12  Aug 14 08:01 healthy
-rw-r--r-- 1 root root 896  Jul  15 16:03 index.html
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- livenessProbe가 실패해서 컨테이너가 반복적으로 재시작될 때는 `kubectl describe pods <이름>`의 Events 섹션에서 `Unhealthy`, `Killing` 이벤트와 함께 실패 원인(HTTP status code, connection refused 등)을 확인한다.
- exec probe나 컨테이너 내부 명령으로 강제 종료시킨 프로세스는 `command terminated with exit code 137`처럼 종료 코드 137(SIGKILL)로 표시되며, 이는 kubelet이 컨테이너를 강제 종료했다는 신호다.
- livenessProbe의 `initialDelaySeconds`를 너무 짧게 설정하면 애플리케이션이 아직 완전히 기동되지 않은 상태에서 probe가 실패해 불필요한 재시작 루프에 빠질 수 있으므로, 애플리케이션의 실제 기동 시간을 고려해 값을 설정해야 한다.
- Pod IP는 컨테이너가 재시작돼도 바뀌지 않는다(Infra Container가 유지) — livenessProbe로 인한 재시작은 Pod 재생성이 아니라 컨테이너만 교체되는 것이므로 IP/이름은 그대로다.

---

>  **핵심 요약**
> - livenessProbe는 컨테이너가 "살아있는가"를 kubelet이 주기적으로 확인하는 방식이며, httpGet(HTTP 200번대)·tcpSocket(TCP 연결 성공)·exec(명령어 exit code 0) 3가지 방식을 지원
> - livenessProbe 실패 시 kubelet은 Pod 전체가 아니라 컨테이너만 재시작하며(Pod IP·이름 유지), 이것이 Self-healing Pod의 핵심 메커니즘
> - readinessProbe(트래픽 차단)와 livenessProbe(컨테이너 재시작)는 질문과 결과가 다르다
> - 주요 옵션: initialDelaySeconds(0초)·timeoutSeconds(1초)·periodSeconds(10초)·successThreshold(1)·failureThreshold(3)
> - 관련: 6.  Kubernetes - Pod 구조와 생성·동작 흐름 · 9.  Kubernetes - Init Container·Static Pod · 10.  Kubernetes - Controller 개념과 ReplicationController
