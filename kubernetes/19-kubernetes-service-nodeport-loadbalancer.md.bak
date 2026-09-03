# Kubernetes - NodePort·LoadBalancer

> **Tag:** #Kubernetes #Service #NodePort #LoadBalancer
> **핵심 요약:** 외부에서 Service에 접근할 수 있도록 Node 포트를 여는 NodePort 타입과, 클라우드 환경의 외부 Load Balancer와 연동하는 LoadBalancer 타입을 실습으로 정리

---

## 1. NodePort 실습

- 모든 노드를 대상으로 외부 접속 가능한 포트를 예약한다.
- Default NodePort 범위: 30000–32767
- ClusterIP를 생성 후 NodePort를 예약한다.
- 외부에서 port 번호를 사용해서 접속 가능하다.

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
[root@k8s-master ~]# vi nodeport-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  clusterIP: 10.100.100.200
  selector:
    app: web-dep
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30100
```

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-nginx.yaml
deployment.apps/deploy-web-dep created


[root@k8s-master ~]# kubectl  apply  -f  nodeport-nginx.yaml
service/nodeport-service created


[root@k8s-master ~]# kubectl  get  deployments deploy-web-dep
NAME             	READY   UP-TO-DATE   AVAILABLE   AGE
deploy-web-dep   	3/3         3                    3                 75s


[root@k8s-master ~]# kubectl  get  pods
NAME                              		READY   STATUS    RESTARTS   AGE
deploy-web-dep-578859465c-h88cn   	1/1         Running     0                97s
deploy-web-dep-578859465c-ppkbj   	1/1         Running     0                25s
deploy-web-dep-578859465c-xfbtl   	1/1         Running     0                97s


[root@k8s-master ~]# kubectl  get  pods  -o  wide
NAME                              		READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
deploy-web-dep-578859465c-h88cn  	1/1         Running     0                5m6s    10.244.2.2   k8s-worker2   <none>           	 <none>
deploy-web-dep-578859465c-ppkbj 	1/1         Running     0                3m4s    10.244.1.3   k8s-worker1   <none>           	 <none>
deploy-web-dep-578859465c-xfbtl   	1/1         Running     0                5m6s    10.244.1.2   k8s-worker1   <none>           	 <none>


[root@k8s-master ~]# kubectl  get  service
NAME               	TYPE        CLUSTER-IP  	EXTERNAL-IP   PORT(S)          AGE
kubernetes         	ClusterIP	  10.96.0.1        	<none>            443/TCP     	    9d
nodeport-service	NodePort	  10.100.100.200	<none>            80:30100/TCP	    2m6s


[root@k8s-master ~]# kubectl  describe   service  nodeport-service
Name:                		nodeport-service
Namespace:                	default
Labels:                   	<none>
Annotations:              	<none>
Selector:                 	app=web-dep
Type:                     	NodePort
IP Family Policy:         	SingleStack
IP Families:              	IPv4
IP:                       	10.100.100.200
IPs:                      	10.100.100.200
Port:                     	<unset>  80/TCP
TargetPort:               	80/TCP
NodePort:                 	<unset>  30100/TCP
Endpoints:                	10.244.1.2:80,10.244.2.2:80,10.244.1.3:80
Session Affinity:         	None
External Traffic Policy:	Cluster
Internal Traffic Policy:	Cluster
Events:                   	<none>


[root@k8s-master ~]# kubectl  get  nodes -o  wide
NAME          STATUS   ROLES       	AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                
k8s-master     Ready    control-plane	9d      v1.35.7      192.168.10.100      <none>            Rocky Linux 9.8 (Blue Onyx)   5.14.0-687.36.1.el9_8.x86_64  
k8s-worker1   Ready    <none>          	9d      v1.35.7      192.168.10.101      <none>            Rocky Linux 9.8 (Blue Onyx)   5.14.0-687.36.1.el9_8.x86_64  
k8s-worker2   Ready    <none>          	9d      v1.35.7      192.168.10.102      <none>            Rocky Linux 9.8 (Blue Onyx)   5.14.0-687.36.1.el9_8.x86_64   
```

`nodeport-service`는 `80:30100/TCP`로 표시되며, 클러스터 내부에서는 `port: 80`으로, 클러스터 외부에서는 임의의 Node IP와 `NodePort: 30100`으로 접속할 수 있다.

---

## 2. LoadBalancer 실습

- Public 클라우드(AWS, Azure, GCP 등)에서 운영 가능하다.
- LoadBalancer를 자동으로 구성 요청한다.
- NodePort를 예약 후 해당 nodeport로 외부 접근을 허용한다.
- 현재(온프레미스 실습 환경)는 구성해도 사용할 수는 없다.
- AWS, Azure, GCP 등과 연계해서 사용해야 한다.

```
[root@k8s-master ~]# vi loadbalancer-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: web-dep
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```
[root@k8s-master ~]# kubectl  apply  -f  loadbalancer-nginx.yaml
service/loadbalancer-service created


[root@k8s-master ~]# kubectl  get  service  loadbalancer-service
NAME                     TYPE             CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
loadbalancer-service   LoadBalancer   10.111.152.137     <pending>         80:31378/TCP   25s
```

EXTERNAL-IP가 `<pending>` 상태로 남아있는 것을 볼 수 있는데, 이는 외부 Load Balancer를 프로비저닝해줄 클라우드 컨트롤러가 없는 온프레미스 환경이기 때문이다. 실제 클라우드(AWS/Azure/GCP) 환경이라면 이 자리에 클라우드가 할당한 외부 IP가 채워진다.

```
[root@k8s-master ~]# kubectl  delete   service  loadbalancer-service
service "loadbalancer-service" deleted from default namespace
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `kubectl get svc`의 EXTERNAL-IP가 `<pending>`으로 계속 남아있으면(LoadBalancer 타입) 클러스터가 클라우드 Load Balancer를 프로비저닝할 수 없는 온프레미스 환경이라는 뜻이다. 실제 클라우드 환경에서만 정상적으로 외부 IP가 할당된다.
- `kubectl describe svc <이름>`의 `Endpoints` 필드가 비어 있다면 Service의 `selector`와 Pod의 `labels`가 일치하지 않는 것이므로, `kubectl get pods --show-labels`로 라벨을 확인하고 selector를 맞추거나 `kubectl label pod <이름> <key>=<value> --overwrite`로 라벨을 수정한다.
- NodePort Service는 `kubectl get svc`에서 `PORT(S)` 컬럼이 `80:30100/TCP`처럼 `서비스포트:노드포트` 형태로 표시된다. 이 노드포트는 모든 Node의 IP에서 동일하게 열린다.
- NodePort는 30000~32767 범위 밖의 값을 `nodePort`에 지정하면 `apply` 시 에러가 발생하므로, YAML 작성 시 해당 범위를 벗어나지 않는지 확인해야 한다.
- LoadBalancer 타입도 내부적으로는 NodePort를 함께 예약하므로, `kubectl describe svc`로 확인하면 NodePort 값이 함께 표시된다.

---

>  **핵심 요약**
> - **NodePort**는 모든 Node의 30000~32767 범위 포트를 열어 `Node IP:NodePort → Service → Pod` 흐름으로 외부 접근을 허용하며, `kubectl get svc`에는 `서비스포트:노드포트` 형태로 표시된다
> - **LoadBalancer**는 클라우드 환경의 외부 Load Balancer와 연동하는 타입으로, `외부 사용자 → Cloud Load Balancer → Service → Pod` 흐름을 가지며 내부적으로 NodePort도 함께 예약한다
> - 온프레미스 환경에서는 LoadBalancer 타입을 생성해도 EXTERNAL-IP가 `<pending>`으로 남는데, 이는 클라우드 컨트롤러가 없어 외부 IP를 프로비저닝할 수 없기 때문이며 정상 동작이다
> - 운영 환경에서는 NodePort를 직접 외부에 노출하기보다 LoadBalancer나 Ingress를 함께 사용하는 경우가 많다
> - 관련: 18.  Kubernetes - Service 기초와 ClusterIP · 10.  Kubernetes - Controller 개념과 ReplicationController · 21.  Kubernetes - Ingress 기초와 준비
