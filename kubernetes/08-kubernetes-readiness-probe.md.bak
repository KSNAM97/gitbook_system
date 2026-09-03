# Kubernetes - readinessProbe

> **Tag:** #Kubernetes #readinessProbe #Service #Probe
> **핵심 요약:** readinessProbe(httpGet/exec/tcpSocket)로 Pod가 트래픽을 받을 준비가 되었는지 확인하고, 실패 시 컨테이너 재시작 없이 Service Endpoint에서만 제외하는 동작 원리와 실습 정리

---

## 1. 개요 (Overview)

Readiness Probe는 Pod가 현재 사용자 요청을 받을 준비가 되었는지 확인하는 기능이다. 컨테이너가 실행 중이라고 해서 항상 서비스를 제공할 준비가 된 것은 아니다.

예를 들어 애플리케이션이 실행된 후 다음과 같은 초기화 작업이 필요할 수 있다.

```
컨테이너 시작 --> 애플리케이션 실행 --> 환경 설정 로딩 --> DB 연결 --> 캐시 데이터 로딩 --> 서비스 준비 완료
```

이 과정에서 컨테이너의 상태는 Running일 수 있지만 실제 서비스는 아직 준비되지 않았을 수 있다. 이때 Readiness Probe를 사용하면 준비가 완료된 Pod만 Service가 사용하도록 설정할 수 있다.

### Readiness Probe의 동작

Readiness Probe 검사에 성공시

```
Readiness Probe 성공  -->  Pod Ready  -->  Service Endpoint에 포함  -->  사용자 트래픽 전달
Readiness Probe 검사에 실패시 Readiness Probe 실패  -->  Pod Not Ready  -->  Service Endpoint에서 제외
   -->  사용자 트래픽 전달 안 함
```

중요한 점은 Readiness Probe가 실패해도 컨테이너를 재시작하지 않는다는 것이다.

Readiness 실패 시:

```
Pod 삭제 안 함
Container 재시작 안 함
Service 트래픽만 차단
다시 정상 상태가 되면 자동으로 Ready 상태가 되고 Service Endpoint에 다시 포함된다.
```

### Readiness Probe의 목적

Readiness Probe를 사용하는 가장 큰 이유는 정상적으로 서비스할 수 있는 Pod에게만 트래픽을 전달하기 위해서이다.

예를 들어 Service 뒤에 Pod가 3개 있다고 가정한다.

```
                               Service
                                 │
        ┌────────┼────────┐
        ↓                       ↓                       ↓
      Pod1                   Pod2                   Pod3
      Ready                 Ready               NotReady
```

Service가 사용할 수 있는 Pod:

```
Pod1 : 사용
Pod2 : 사용
Pod3 : 제외
따라서 Pod3가 장애 상태이거나 아직 준비되지 않은 경우 사용자 요청이 Pod3으로 전달되지 않는다.
```

### Readiness Probe와 Service의 관계

Readiness Probe는 Service와 매우 밀접한 관계가 있다. Service는 Selector를 이용하여 Pod를 찾지만, 실제 트래픽을 전달할 때는 Ready 상태의 Pod를 Endpoint로 사용한다.

예:

```yaml
# Service
selector:
  app: web
```

```
# Pod:
web-pod1   app=web   Ready
web-pod2   app=web   Ready
web-pod3   app=web   NotReady

# Service Endpoint:
web-pod1
web-pod2
```

web-pod3는 Label이 일치하더라도 Readiness 검사에 실패하면 정상 트래픽 대상에서 제외된다. Readiness Probe가 실패했다고 해서 새로운 Pod를 자동으로 만들지는 않는다. (서비스만 제외)

### Readiness Probe와 Liveness Probe 차이

| 구분 | Readiness Probe | Liveness Probe |
|---|---|---|
| 목적 | 서비스 가능한지 확인 | 컨테이너가 정상적으로 살아있는지 확인 |
| 실패 시 | Service에서 제외 | 컨테이너 재시작 |
| Pod 상태 | Running 유지 가능 | Running 중 컨테이너 재시작 |
| 트래픽 | 차단 | 재시작 동안 차단 |
| 핵심 질문 | 요청을 받아도 되는가? | 살아 있는가? |

---

## 2. Readiness Probe 검사 방법

대표적으로 다음 방법을 사용한다.

**1) HTTP Get 검사**

특정 URL에 HTTP 요청을 보내 정상 응답인지 확인한다.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

동작:

```
Kubelet  -->  http://Pod-IP:80/  -->  정상 응답  -->  Ready
```

**2) Exec 검사**

컨테이너 내부에서 특정 명령어를 실행한다.

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/ready
```

명령어가 성공하면 Ready 상태가 된다. 파일이 존재하지 않아 명령어가 실패하면 NotReady가 된다.

**3) TCP Socket 검사**

특정 TCP 포트에 연결 가능한지 확인한다.

```yaml
readinessProbe:
  tcpSocket:
    port: 80
```

80번 포트에 연결할 수 있으면 성공한다.

## 3. Readiness Probe 주요 옵션

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 2
  successThreshold: 1
  failureThreshold: 3
```

- **initialDelaySeconds** — 컨테이너 시작 후 첫 검사를 시작하기까지 기다리는 시간
- **periodSeconds** — 몇 초마다 검사할 것인지 설정
- **timeoutSeconds** — 검사 응답을 몇 초까지 기다릴 것인지 설정
- **successThreshold** — 몇 번 연속 성공해야 정상으로 판단할지 설정
- **failureThreshold** — 몇 번 연속 실패해야 비정상으로 판단할지 설정

## 4. EX1) HTTP Readiness Probe Deployment 생성

**Deployment**

| 항목 | 값 |
|---|---|
| Deployment 이름 | readiness-deploy |
| Pod 개수 | 3 |
| Image | nginx:1.31 |
| Label | app=readiness |
| Container Port | 80 |
| Readiness 방식 | HTTP Get |
| 검사 경로 | / |
| 검사 포트 | 80 |
| initialDelaySeconds | 5 |
| periodSeconds | 5 |

```
[root@k8s-master ~]# kubectl delete deployments drain-test-deploy
deployment.apps "drain-test-deploy" deleted from default namespace


[root@k8s-master ~]# kubectl delete daemonsets  drain-ds
daemonset.apps "drain-ds" deleted from default namespace


[root@k8s-master ~]# vi readiness-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness
  template:
    metadata:
      labels:
        app: readiness
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5


[root@k8s-master ~]# kubectl  apply  -f  readiness-deploy.yaml
deployment.apps/readiness-deploy created


[root@k8s-master ~]# kubectl  get  pods
NAME                                	READY   STATUS    RESTARTS   AGE
readiness-deploy-7b4f58cd8b-nbpj4   	1/1         Running     0                9s
readiness-deploy-7b4f58cd8b-qspn4   	1/1         Running     0                9s
readiness-deploy-7b4f58cd8b-wj858   	1/1         Running     0                9s
```

Nginx는 기본적으로 / 경로에 정상 페이지를 제공한다. 따라서 다음 검사가 성공한다.

```
http://Pod-IP:80/
성공하면 : READY 1/1
```

## 5. EX2) Service 생성

ClusterIP Service를 생성

| 항목 | 값 |
|---|---|
| Service 이름 | readiness-service |
| Selector | app=readiness |
| Port | 80 |
| TargetPort | 80 |

```
[root@k8s-master ~]# vi readiness-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: readiness-service
spec:
  type: ClusterIP
  selector:
    app: readiness
  ports:
  - port: 80
    targetPort: 80



[root@k8s-master ~]# kubectl  apply  -f  readiness-service.yaml
service/readiness-service created



[root@k8s-master ~]# kubectl  get  service
NAME                	TYPE        CLUSTER-IP      	EXTERNAL-IP   	PORT(S)	  AGE
kubernetes          	ClusterIP    10.96.0.1       	<none>        	443/TCP	  15d
readiness-service	ClusterIP    10.111.102.63   	<none>        	80/TCP	  37s


	
	# EndPoint 확인
[root@k8s-master ~]# kubectl  get  endpoints readiness-service
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                	ENDPOINTS                                      	AGE
readiness-service	10.244.1.40:80,10.244.1.41:80,10.244.1.42:80   	19s



	# EndPoint 확인
[root@k8s-master ~]# kubectl get endpointslice  readiness-service-mdr2n
NAME                      	ADDRESSTYPE   	PORTS	ENDPOINTS                             	AGE
readiness-service-mdr2n	IPv4          	80      	10.244.1.40,10.244.1.41,10.244.1.42	3m4s
```

Service는 다음 Label을 가진 Pod를 찾는다.

```
app=readiness
따라서 Deployment가 생성한 Pod 3개가 Service 대상이 된다.
```

```
readiness-service
        │
        ├── Pod1
        ├── Pod2
        └── Pod3
```

## 6. EX3) Readiness Probe 실패 상태 만들기

현재 Readiness Probe는 다음 경로를 검사한다. `/`를 존재하지 않는 경로인 `/notfound`로 변경하여 Readiness Probe를 고의로 실패시키기.

**STEP 1 현재 상태 확인**

```
[root@k8s-master ~]# kubectl  get  pods
NAME                                	READY   STATUS    RESTARTS   AGE
readiness-deploy-7b4f58cd8b-nbpj4   	1/1         Running     0                9s
readiness-deploy-7b4f58cd8b-qspn4   	1/1         Running     0                9s
readiness-deploy-7b4f58cd8b-wj858   	1/1         Running     0                9s
```

```
[root@k8s-master ~]# kubectl  edit  deployments.apps  readiness-deploy
/path
~~~~~~~~~ 중간 생략 ~~~~~~~~~
    spec:
      containers:
      - image: nginx:1.31
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /notfound		# /   -->  /notfound  로 수정 후 저장	
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
~~~~~~~~~ 중간 생략 ~~~~~~~~~
```

```
[root@k8s-master ~]# kubectl  get  pods
NAME                                	READY   STATUS    RESTARTS   AGE
readiness-deploy-5bc4488b49-2tpxx   	0/1         Running     0                3s
readiness-deploy-7b4f58cd8b-7pd7n   	0/1         Running     0                2s
readiness-deploy-7b4f58cd8b-8hkk2   	0/1         Running     0                3s
readiness-deploy-7b4f58cd8b-bfrtd   	0/1         Running     0                2s
```

```
[root@k8s-master ~]# kubectl  describe  pods  readiness-deploy-5bc4488b49-2tpxx
~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~
Events:
  Type     Reason     Age                 	 From               Message
  ----     ------     ----                	 ----               -------
  Normal   Scheduled  3m10s            	default-scheduler  Successfully assigned default/readiness-deploy-5bc4488b49-2tpxx to k8s-worker1
  Normal   Pulled     3m10s                	kubelet               spec.containers{nginx}: Container image "nginx:1.31" already present on machine and can be accessed by the pod
  Normal   Created    3m10s                	kubelet               spec.containers{nginx}: Container created
  Normal   Started    3m10s     	kubelet               spec.containers{nginx}: Container started
  Warning  Unhealthy  65s 		kubelet               spec.containers{nginx}: Readiness probe failed: HTTP probe failed with statuscode: 404
```

```
[root@k8s-master ~]# kubectl  get  service
NAME                	TYPE        CLUSTER-IP      	EXTERNAL-IP   PORT(S)   AGE
kubernetes          	ClusterIP   10.96.0.1       	<none>        443/TCP   15d
readiness-service	ClusterIP   10.111.102.63   	<none>        80/TCP    14m
```

```
[root@k8s-master ~]# curl  http://10.111.102.63/notfound
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.31.3</center>
</body>
</html>
```

**다시 경로를 변경해서 readinessProbe를 성공으로 수정**

```
[root@k8s-master ~]# kubectl  edit  deployments.apps  readiness-deploy
/path
~~~~~~~~~ 중간 생략 ~~~~~~~~~
    spec:
      containers:
      - image: nginx:1.31
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /notfound		# /notfound   -->  /  로 수정 후 저장	
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
~~~~~~~~~ 중간 생략 ~~~~~~~~~
```

```
[root@k8s-master ~]# kubectl  get  pods
NAME                                	READY   STATUS    RESTARTS   AGE
readiness-deploy-7b4f58cd8b-7pd7n   	1/1        Running      0                7m18s
readiness-deploy-7b4f58cd8b-8hkk2   	1/1        Running      0                7m19s
readiness-deploy-7b4f58cd8b-bfrtd   	1/1        Running      0                7m18s
```

---

## 7. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- readinessProbe가 실패하면 `kubectl get pods`에서 `READY` 컬럼이 `0/1`로 표시되지만 `STATUS`는 여전히 `Running`으로 남는다 — livenessProbe 실패(재시작으로 RESTARTS 증가)와 구분되는 핵심 신호다.
- `kubectl describe pods <이름>`의 Events에서 `Unhealthy` 이벤트와 함께 `Readiness probe failed: HTTP probe failed with statuscode: 404` 같은 실패 원인이 기록되므로, 원인 파악의 첫 단계로 사용한다.
- readinessProbe 실패는 Service의 Endpoint(EndpointSlice)에서만 해당 Pod IP가 빠지는 것이므로, `kubectl get endpoints`/`kubectl get endpointslice`로 실패한 Pod가 실제로 대상에서 제외됐는지 함께 확인한다.
- Pod가 계속 `0/1`로 남아있다면 컨테이너 재시작으로는 해결되지 않으므로(readinessProbe는 재시작을 유발하지 않음) 애플리케이션 자체의 초기화 지연, 잘못된 path/port 설정, initialDelaySeconds 부족 등을 점검해야 한다.
- 하나의 컨테이너에 readinessProbe와 livenessProbe를 함께 설정하는 것이 일반적이며, 두 Probe는 독립적으로 동작한다(readiness 실패는 트래픽만 차단, liveness 실패는 컨테이너 재시작).

---

>  **핵심 요약**
> - readinessProbe는 Pod가 트래픽을 받을 준비가 되었는지 확인하며, httpGet·exec·tcpSocket 3가지 검사 방식을 지원
> - readinessProbe 실패 시 Pod와 컨테이너는 그대로 유지되고 Service Endpoint에서만 제외되어 트래픽만 차단된다(재시작 없음)
> - livenessProbe(컨테이너 재시작)와 달리 readinessProbe는 "요청을 받아도 되는가?"를 묻고 실패해도 자동 복구되면 다시 Endpoint에 포함된다
> - 주요 옵션: initialDelaySeconds·periodSeconds·timeoutSeconds·successThreshold·failureThreshold
> - 관련: 7.  Kubernetes - livenessProbe · 2.  Kubernetes - Pod 생성 · 9.  Kubernetes - Init Container·Static Pod
