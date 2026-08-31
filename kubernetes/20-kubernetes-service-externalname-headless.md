# Kubernetes - ExternalName·Headless Service

> **Tag:** #Kubernetes #Service #ExternalName #Headless #StatefulSet #부트캠프
> **핵심 요약:** 셀렉터 없이 외부 DNS를 Service 이름으로 매핑하는 ExternalName 타입과, ClusterIP 없이 Pod IP를 그대로 노출하는 Headless Service 타입을 StatefulSet 연동 실습까지 정리

---

## 1. ExternalName 실습

- 외부 DNS 이름을 Service 이름으로 매핑한다.
- 셀렉터(selector)를 사용하지 않는다. (Pod를 선택하지 않음)
- 클러스터 내부에서 외부 서비스를 내부 서비스처럼 사용하기 위한 목적이다.

```
[root@k8s-master ~]# vi extname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ExternalName
  externalName: example.com

---
apiVersion: v1
kind: Pod
metadata:
  name: tester
spec:
  containers:
  - name: test
    image: busybox:1.36
    command: ["sh","-c","sleep 3600"]
```

service 이름인 `web`이 쿠버네티스 Core DNS에 저장된다 = `web.default.svc.cluster.local`

```
[root@k8s-master ~]# kubectl  apply  -f  extname-service.yaml  --dry-run=client
service/web created (dry run)
pod/tester created (dry run)


[root@k8s-master ~]# kubectl  exec  -it  tester  -- nslookup web
Server:	10.96.0.10
Address:	10.96.0.10:53

** server can't find web.cluster.local: NXDOMAIN

** server can't find web.cluster.local: NXDOMAIN

** server can't find web.svc.cluster.local: NXDOMAIN

** server can't find web.svc.cluster.local: NXDOMAIN

web.default.svc.cluster.local   canonical name = example.com
Name:   example.com
Address: 104.20.23.154
Name:   example.com
Address: 172.66.147.243

web.default.svc.cluster.local   canonical name = example.com
Name:   example.com
Address: 2606:4700:10::6814:179a
Name:   example.com
Address: 2606:4700:10::ac42:93f3

command terminated with exit code 1
```

### 실습 문제 — ExternalName Service를 생성하여 naver라는 이름으로 google.com에 접근하시오

- Service 이름: naver
- Service Type: ExternalName
- 외부 DNS: google.com
- 테스트 Pod에서 DNS 및 접속 확인

```
[root@k8s-master ~]# vi externalname-google.yaml
apiVersion: v1
kind: Service
metadata:
  name: naver                  	# Kubernetes 내부에서 사용할 Service 이름

spec:
  type: ExternalName           	# 외부 DNS 이름과 연결하는 Service
  externalName: google.com	# 실제로 연결할 외부 DNS 이름
```

```
[root@k8s-master ~]# kubectl apply  -f  externalname-google.yaml  --dry-run=client
service/naver created (dry run)


[root@k8s-master ~]# kubectl apply  -f  externalname-google.yaml 
service/naver created


[root@k8s-master ~]# kubectl  get  service
NAME         	TYPE           	CLUSTER-IP   	EXTERNAL-IP   	PORT(S) 	  AGE
kubernetes	ClusterIP      	10.96.0.1		<none>        	43/TCP	  9d
naver        	ExternalName	 <none>       	google.com    	<none> 	  16s


[root@k8s-master ~]# kubectl  run  dnspod  --image=busybox:1.36  --command  -- sleep 3600


[root@k8s-master ~]# kubectl   get  pods
NAME 	READY   STATUS        RESTARTS   AGE
dnspod	1/1         Running         0                93s


	# dnspod로 접속
[root@k8s-master ~]# kubectl  exec  -it  dnspod  -- sh
/ #
/ #
/ # nslookup  naver
Server:	   10.96.0.10
Address:	   10.96.0.10:53

** server can't find naver.cluster.local: NXDOMAIN

** server can't find naver.svc.cluster.local: NXDOMAIN

** server can't find naver.cluster.local: NXDOMAIN

** server can't find naver.svc.cluster.local: NXDOMAIN

naver.default.svc.cluster.local canonical name = google.com
Name:   google.com
Address: 2404:6800:400b:c00c::64
Name:   google.com
Address: 2404:6800:400b:c00c::65
Name:   google.com
Address: 2404:6800:400b:c00c::8b
Name:   google.com
Address: 2404:6800:400b:c00c::71

naver.default.svc.cluster.local canonical name = google.com
Name:   google.com
Address: 142.250.21.101
Name:   google.com
Address: 142.250.21.138
Name:   google.com
Address: 142.250.21.102
Name:   google.com
Address: 142.250.21.113
Name:   google.com
Address: 142.250.21.139
Name:   google.com
Address: 142.250.21.100
```

### curl이 되는 이미지로 실제 접속 확인

```
	# curl이 되는 이미지로 수정
[root@k8s-master ~]# kubectl  run curlpod  --image=curlimages/curl:latest  --rm  -it  --restart=Never  -- curl  http://naver
<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <meta name=viewport content="initial-scale=1, minimum-scale=1, width=device-width">
  <title>Error 404 (Not Found)!!1</title>
  <style>
    *{margin:0;padding:0}html,code{font:15px/22px arial,sans-serif}html{background:#fff;color:#222;padding:15px}body{margin:7% auto 0;max-width:390px;min-height:180px;padding:30px 0 15px}* > body{background:url(//www.google.com/images/errors/robot.png) 100% 5px no-repeat;padding-right:205px}p{margin:11px 0 22px;overflow:hidden}ins{color:#777;text-decoration:none}a img{border:0}@media screen and (max-width:772px){body{background:none;margin-top:0;max-width:none;padding-right:0}}#logo{background:url(//www.google.com/images/branding/googlelogo/1x/googlelogo_color_150x54dp.png) no-repeat;margin-left:-5px}@media only screen and (min-resolution:192dpi){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat 0% 0%/100% 100%;-moz-border-image:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) 0}}@media only screen and (-webkit-min-device-pixel-ratio:2){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat;-webkit-background-size:100% 100%}}#logo{display:inline-block;height:54px;width:150px}
  </style>
  <a href=//www.google.com/><span id=logo aria-label=Google></span></a>
  <p><b>404.</b> <ins>That's an error.</ins>
  <p>The requested URL <code>/</code> was not found on this server.  <ins>That's all we know.</ins>
pod "curlpod" deleted from default namespace


	# 정해진 기능을 수행후 --rm 명령어에의해  pod 삭제
[root@k8s-master ~]# kubectl  get pods
No resources found in default namespace.
```

`naver` Service를 통해 `http://naver`로 요청하면 실제로는 `google.com`(루트 경로는 404)에 접속되는 것을 확인할 수 있다. ExternalName Service가 CNAME 방식으로 외부 도메인에 매핑되어 있기 때문이다.

---

## 2. Headless Service 실습

Headless Service는 서비스 IP(ClusterIP)를 아예 만들지 않는 Service다.

일반 Service는 하나의 가상 IP(ClusterIP)를 만들고 그 IP 뒤에서 여러 Pod로 트래픽을 분산한다. Headless Service는 ClusterIP를 만들지 않는다. 대신 Pod 각각의 IP를 그대로 노출한다. 즉, 로드밸런서를 없애고 Pod 하나하나를 직접 보이게 만드는 Service이다.

**일반 Service vs Headless Service 차이**

| 구분 | 일반 Service (ClusterIP) | Headless Service |
|---|---|---|
| Service IP | 있음 (예: 10.96.10.50) | 없음 (`clusterIP: None`) |
| DNS 조회 결과 | Service IP 1개 | Pod IP 목록 여러 개 |
| kube-proxy 개입 | 트래픽을 Pod로 분산 | 개입하지 않음 |
| 클라이언트 인지 | Pod 존재를 모름 | Pod를 직접 선택 |

**Headless Service YAML 구조 예시**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 3306
    targetPort: 3306
```

이렇게 설정하면 Service IP는 생성되지 않고 DNS만 관리하는 Service가 된다.

**Headless Service의 DNS 동작 방식**

일반 Service DNS 조회:

```
myservice.default.svc.cluster.local
# 10.96.10.50
```

Headless Service DNS 조회:

```
my-headless.default.svc.cluster.local
# 10.244.1.10
# 10.244.2.15
# 10.244.3.8
```

즉, 하나의 IP가 아니라 Pod IP 리스트가 그대로 반환된다. 이 리스트를 애플리케이션이 직접 사용하거나 클라이언트가 직접 선택한다.

**Headless Service가 필요한 이유**

일반적인 웹 서비스에서는 어느 Pod로 가든 상관없다. 하지만 이런 경우는 다르다.

- DB 클러스터 — Primary / Replica 구분 필요, 특정 Pod로 반드시 접속해야 함
- Stateful 서비스 — Pod마다 고유 ID 필요, Pod 순서와 이름이 중요

이런 시스템은 "쿠버네티스가 분산하지 말고 내가 직접 관리할게"라는 전제에서 동작한다.

- DB 읽기/쓰기용 — Primary, db-1, db-2는 읽기 전용 Replica로 구분해서 사용할 때, 각 DB Pod를 고정된 이름으로 직접 지정할 수 있다.
- 분산 저장소 구성 — 각 저장소 노드가 서로 다른 데이터를 담당하거나 고유한 역할을 가지는 경우, 특정 노드를 정확하게 지정해서 접근할 수 있다.

### ClusterIP와 Headless Service 비교 실습

```
[root@k8s-master ~]# vi deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dep
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


[root@k8s-master ~]# kubectl  get  deployments  deploy-web-dep
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web-dep   3/3     3            3           23s


[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-578859465c-chqzs   	1/1     Running   0          26s
deploy-web-dep-578859465c-qj479   	1/1     Running   0          45s
deploy-web-dep-578859465c-wgtj2   	1/1     Running   0          45s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                              		READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
deploy-web-dep-578859465c-chqzs   	1/1        Running     0                 41s    10.244.1.7   k8s-worker1   <none>           <none>
deploy-web-dep-578859465c-qj479   	1/1        Running     0                 60s    10.244.1.6   k8s-worker1   <none>           <none>
deploy-web-dep-578859465c-wgtj2   	1/1        Running     0                 60s    10.244.2.6   k8s-worker2   <none>           <none>
```

```
	# Service (ClusterIP)
[root@k8s-master ~]# vi clusterip-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service
spec:
  type: ClusterIP
  clusterIP: 10.100.100.100
  selector:
    app: web-dep
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```
	# Service (Headless)
[root@k8s-master ~]# vi headless-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: web-dep
  ports:
  - protocol: TCP
    port: 80
```

```
[root@k8s-master ~]# kubectl  apply  -f   clusterip-nginx.yaml
service/clusterip-service created


[root@k8s-master ~]# kubectl  apply  -f   headless-nginx.yaml
service/headless-service created


[root@k8s-master ~]# kubectl  get  service
NAME              	TYPE        CLUSTER-IP  	EXTERNAL-IP   PORT(S)   AGE
clusterip-service   	ClusterIP    10.100.100.100	<none>            80/TCP     23s
headless-service    	ClusterIP    None             	<none>            80/TCP     20s
kubernetes          	ClusterIP    10.96.0.1        	<none>            443/TCP    9d
```

Headless Service는 CLUSTER-IP가 할당되지 않는다.

```
[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                              		READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
deploy-web-dep-578859465c-chqzs   	1/1        Running     0                 41s    10.244.1.7   k8s-worker1   <none>           <none>
deploy-web-dep-578859465c-qj479   	1/1        Running     0                 60s    10.244.1.6   k8s-worker1   <none>           <none>
deploy-web-dep-578859465c-wgtj2   	1/1        Running     0                 60s    10.244.2.6   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  describe  service  headless-service
Name:                     	headless-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	app=web-dep
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	None
IPs:                      	None
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.2.6:80,10.244.1.7:80,10.244.1.6:80
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                		<none>


[root@k8s-master ~]# kubectl  get  pods  --all-namespaces   -o wide
NAMESPACE      NAME                                 READY   STATUS    RESTARTS       	AGE     IP               NODE          NOMINATED NODE   READINESS GATES
default        deploy-web-dep-578859465c-chqzs	1/1     Running   0                  	19m     10.244.1.7       k8s-worker1   <none>           <none>
default        deploy-web-dep-578859465c-qj479	1/1     Running   0                  	19m     10.244.1.6       k8s-worker1   <none>           <none>
default        deploy-web-dep-578859465c-wgtj2	1/1     Running   0                  	19m     10.244.2.6       k8s-worker2   <none>           <none>
default        testcentos                           	1/1     Running   0                  	4m52s  10.244.2.7       k8s-worker2   <none>           <none>
kube-flannel   kube-flannel-ds-7hlrp                	1/1     Running   8 (153m ago)   	9d       192.168.10.100   k8s-master    <none>           <none>
kube-flannel   kube-flannel-ds-f9wcz                	1/1     Running   7 (153m ago)   	9d       192.168.10.101   k8s-worker1   <none>           <none>
kube-flannel   kube-flannel-ds-z4fbv                	1/1     Running   8 (153m ago)   	9d       192.168.10.102   k8s-worker2   <none>           <none>
kube-system    coredns-7d764666f9-2dwdd           	1/1     Running   8 (153m ago)   	9d       10.244.0.19      k8s-master    <none>           <none>
kube-system    coredns-7d764666f9-klkhq          	1/1     Running   8 (153m ago)   	9d       10.244.0.18      k8s-master    <none>           <none>
kube-system    etcd-k8s-master                    	1/1     Running   9 (153m ago)   	9d       192.168.10.100   k8s-master    <none>           <none>
kube-system    kube-apiserver-k8s-master       	1/1     Running   9 (153m ago)   	9d       192.168.10.100   k8s-master    <none>           <none>
kube-system    kube-controller-manager-k8s-master   1/1     Running   9 (153m ago) 	9d       192.168.10.100   k8s-master    <none>           <none>
kube-system    kube-proxy-bxjvw                 	1/1     Running   7 (153m ago)   	9d       192.168.10.102   k8s-worker2   <none>           <none>
kube-system    kube-proxy-gbr4f                  	1/1     Running   7 (153m ago)   	9d       192.168.10.101   k8s-worker1   <none>           <none>
kube-system    kube-proxy-rwhkc                 	1/1     Running   8 (153m ago)   	9d       192.168.10.100   k8s-master    <none>           <none>
kube-system    kube-scheduler-k8s-master       	1/1     Running   9 (153m ago)   	9d       192.168.10.100   k8s-master    <none>           <none>


[root@k8s-master ~]# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                          AGE
kube-dns    ClusterIP   10.96.0.10        <none>            53/UDP,53/TCP,9153/TCP   9d


	# centos pod를 생성하고 바로 컨테이너로 접속
[root@k8s-master ~]# kubectl  run  testcentos  -it  --image=centos:8  /bin/bash
If you don't see a command prompt, try pressing enter.
[root@testcentos /]#

[root@testcentos /]# cat  /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5


	# dns를 테스트하기위한 pod 생성 (자동 삭제)
[root@k8s-master ~]# kubectl  run  dns-test  --image=busybox:1.36  -it  --rm  --restart=Never  -- sh
All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
/ #
/ #
/ #
/ # nslookup  headless-service
Server:          10.96.0.10
Address:        10.96.0.10:53

** server can't find headless-service.cluster.local: NXDOMAIN
** server can't find headless-service.cluster.local: NXDOMAIN
** server can't find headless-service.svc.cluster.local: NXDOMAIN

Name:   headless-service.default.svc.cluster.local
Address: 10.244.2.6
Name:   headless-service.default.svc.cluster.local
Address: 10.244.1.6
Name:   headless-service.default.svc.cluster.local
Address: 10.244.1.7

** server can't find headless-service.svc.cluster.local: NXDOMAIN


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                              		READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
deploy-web-dep-578859465c-chqzs   	1/1        Running     0                 41s    10.244.1.7   k8s-worker1   <none>           <none>
deploy-web-dep-578859465c-qj479   	1/1        Running     0                 60s    10.244.1.6   k8s-worker1   <none>           <none>
deploy-web-dep-578859465c-wgtj2   	1/1        Running     0                 60s    10.244.2.6   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  delete  deployments  deploy-web-dep
deployment.apps "deploy-web-dep" deleted from default namespace


[root@k8s-master ~]# kubectl  delete service  clusterip-service
service "clusterip-service" deleted from default namespace


[root@k8s-master ~]# kubectl  delete service  headless-service
service "headless-service" deleted from default namespace
```

`nslookup headless-service` 결과로 Service IP가 아니라 3개 Pod의 IP가 그대로 리스트로 반환되는 것을 확인할 수 있다. 이것이 Headless Service의 핵심 동작이다.

### 실습 문제 — Headless Service와 StatefulSet을 연동해서 Pod 3개를 생성하고, 각 Pod가 고정된 이름으로 DNS 조회되는지 확인

- StatefulSet 이름: web-sts
- Pod 개수: 3개
- 이미지: nginx:1.31
- Service 이름: web-headless

각 Pod는 다음 이름으로 생성되어야 한다.
- web-sts-0
- web-sts-1
- web-sts-2

각 Pod를 다음 DNS 이름으로 직접 조회할 수 있어야 한다.
- web-sts-0.web-headless
- web-sts-1.web-headless
- web-sts-2.web-headless

**STEP 1) StatefulSet 생성**

```
[root@k8s-master ~]# vi statefulset-pod.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web-sts
  template:
    metadata:
      labels:
        app: web-sts
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
        ports:
        - containerPort: 80
```

```
[root@k8s-master ~]# kubectl apply -f statefulset-pod.yaml
statefulset.apps/web-sts created
```

**STEP 2) Headless Service 생성**

```
[root@k8s-master ~]# vi headless-sts.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web-sts
  ports:
  - port: 80
    targetPort: 80
```

```
[root@k8s-master ~]# kubectl apply -f headless-sts.yaml
service/web-headless created
```

**STEP 3) Pod, Headless Service 확인**

```
[root@k8s-master ~]# kubectl  get  statefulsets  web-sts
NAME      READY   AGE
web-sts     3/3       2m16


[root@k8s-master ~]# kubectl  get  pods
NAME        READY   STATUS    RESTARTS   AGE
web-sts-0   1/1         Running     0                54s
web-sts-1   1/1         Running     0                52s
web-sts-2   1/1         Running     0                51s


[root@k8s-master ~]# kubectl  get  service  web-headless
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
web-headless   ClusterIP    None             <none>             80/TCP    70s


[root@k8s-master ~]# kubectl  describe  service web-headless
Name:                     	web-headless
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	app=web-sts
Type:                     	ClusterIP
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	None
IPs:                      	None
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
Endpoints:                	10.244.1.9:80,10.244.2.9:80,10.244.1.10:80
Session Affinity:         	None
Internal Traffic Policy:  	Cluster
Events:                   	<none>


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME        READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
web-sts-0   1/1        Running     0                 4m7s   10.244.1.9    k8s-worker1   <none>                  <none>
web-sts-1   1/1        Running     0                 4m5s   10.244.2.9    k8s-worker2   <none>                  <none>
web-sts-2   1/1        Running     0                 4m4s   10.244.1.10   k8s-worker1   <none>                  <none>
```

**STEP 4) Service 이름으로 DNS 조회**

```
[root@k8s-master ~]# kubectl  run  dnspod  --image=busybox:1.36  --rm  -it  --restart=Never  -- nslookup  web-headless
Server:       	10.96.0.10
Address:     	10.96.0.10:53

** server can't find web-headless.cluster.local: NXDOMAIN

** server can't find web-headless.cluster.local: NXDOMAIN

** server can't find web-headless.svc.cluster.local: NXDOMAIN

Name:   web-headless.default.svc.cluster.local
Address: 10.244.2.9
Name:   web-headless.default.svc.cluster.local
Address: 10.244.1.9
Name:   web-headless.default.svc.cluster.local
Address: 10.244.1.10

** server can't find web-headless.svc.cluster.local: NXDOMAIN


pod "dnspod" deleted from default namespace
```

Headless Service 이름으로 해당 Service에 포함된 모든 Pod의 IP 주소를 리턴받을 수 있다.

**STEP 5) 특정 Pod를 이름으로 직접 조회**

```
# busybox는 몇가지 명령어만 실행할수있는 작은 이미지이므로 DNS기능의 한계로 전체 Domain 주소를 입력해야 한다.
[root@k8s-master ~]# kubectl  run  dnspod  --image=busybox:1.36  --rm  -it  --restart=Never  -- nslookup  web-sts-0.web-headless.default.svc.cluster.local
Server:         10.96.0.10
Address:       10.96.0.10:53

Name:   web-sts-0.web-headless.default.svc.cluster.local
Address: 10.244.1.9


# busybox는 몇가지 명령어만 실행할수있는 작은 이미지이므로 DNS기능의 한계로 전체 Domain 주소를 입력해야 한다.
[root@k8s-master ~]# kubectl  run  dnspod  --image=busybox:1.36  --rm  -it  --restart=Never  -- nslookup  web-sts-1.web-headless.default.svc.cluster.local
Server:         10.96.0.10
Address:       10.96.0.10:53

Name:   web-sts-1.web-headless.default.svc.cluster.local
Address: 10.244.2.9


# busybox는 몇가지 명령어만 실행할수있는 작은 이미지이므로 DNS기능의 한계로 전체 Domain 주소를 입력해야 한다.
[root@k8s-master ~]# kubectl  run  dnspod  --image=busybox:1.36  --rm  -it  --restart=Never  -- nslookup  web-sts-2.web-headless.default.svc.cluster.local
Server:         10.96.0.10
Address:       10.96.0.10:53

Name:   web-sts-2.web-headless.default.svc.cluster.local
Address: 10.244.1.10
```

**StatefulSet Pod 내부에서 다른 Pod의 DNS 확인**

```
[root@k8s-master ~]# kubectl  get  pods
NAME        READY   STATUS    RESTARTS   AGE
web-sts-0   1/1         Running     0                54s
web-sts-1   1/1         Running     0                52s
web-sts-2   1/1         Running     0                51s


[root@k8s-master ~]# kubectl  exec  -it  web-sts-0  -- /bin/bash
root@web-sts-0:/# 

root@web-sts-0:/# getent  hosts  web-sts-1.web-headless
10.244.2.9      web-sts-1.web-headless.default.svc.cluster.local


root@web-sts-0:/# getent  hosts  web-sts-2.web-headless
10.244.1.10     web-sts-2.web-headless.default.svc.cluster.local


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME        READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
web-sts-0   1/1        Running     0                 4m7s   10.244.1.9    k8s-worker1   <none>                  <none>
web-sts-1   1/1        Running     0                 4m5s   10.244.2.9    k8s-worker2   <none>                  <none>
web-sts-2   1/1        Running     0                 4m4s   10.244.1.10   k8s-worker1   <none>                  <none>
```

`web-sts-0` Pod 내부에서 `getent hosts web-sts-1.web-headless`, `getent hosts web-sts-2.web-headless`를 실행하면 각 Pod의 실제 IP가 정확히 반환된다. StatefulSet + Headless Service 조합으로 각 Pod를 이름 기반의 고정 주소로 개별 식별할 수 있음을 확인할 수 있다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- ExternalName Service는 Pod를 선택하지 않으므로 `kubectl describe svc`에 `Selector`, `Endpoints`가 나타나지 않는다. 대신 `TYPE`이 `ExternalName`이고 `EXTERNAL-IP` 자리에 매핑된 외부 도메인이 표시된다.
- Headless Service(`clusterIP: None`)는 `kubectl get svc`의 `CLUSTER-IP` 컬럼이 `None`으로 표시되며, `nslookup`으로 조회 시 Service IP 대신 Pod IP 목록이 그대로 반환된다.
- busybox 이미지의 `nslookup`은 축약된 이름(`web-headless`)만으로는 특정 Pod를 조회할 수 없는 경우가 있으므로, `<Pod이름>.<Service이름>.<네임스페이스>.svc.cluster.local` 형태의 전체 도메인을 입력해야 한다.
- Deployment가 관리하는 Pod에 새 label을 추가/변경하면 즉시 Service의 Endpoints 목록에 반영되므로, `kubectl describe svc` 결과를 재조회해서 `+ N more...` 표기로 대상 Pod 수 변화를 확인할 수 있다.
- StatefulSet과 Headless Service를 연동할 때는 StatefulSet의 `spec.serviceName`이 Headless Service의 이름과 정확히 일치해야 각 Pod가 `<Pod이름>.<Service이름>` 형태의 고정 DNS를 갖는다.

---

>  **핵심 요약**
> - **ExternalName**은 selector 없이 "Service 이름 → 외부 DNS 이름"을 CNAME으로 매핑하는 타입으로, 외부 DB·API·SaaS 연동에 사용한다
> - ExternalName Service는 Pod를 전혀 선택하지 않으며, 클러스터 내부에서 Service 이름으로 조회하면 외부 도메인의 CNAME이 그대로 반환된다
> - **Headless Service**(`clusterIP: None`)는 ClusterIP를 만들지 않고 DNS 조회 시 연결된 Pod들의 IP 목록을 그대로 반환하며, kube-proxy가 로드밸런싱에 개입하지 않는다
> - Headless Service는 StatefulSet과 결합해 `<Pod이름>.<Service이름>` 형태의 고정 DNS로 개별 Pod에 직접 접근할 때 사용하며, DB 클러스터·Stateful 서비스처럼 특정 Pod로 반드시 접속해야 하는 경우에 적합하다
> - 관련: 18.  Kubernetes - Service 기초와 ClusterIP · 2.  Kubernetes - Pod 생성 · 10.  Kubernetes - Controller 개념과 ReplicationController
