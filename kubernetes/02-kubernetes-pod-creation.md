# 📦 Kubernetes - Pod 생성

> **Tag:** #Kubernetes #Pod #Deployment #kubectl #부트캠프
> **핵심 요약:** kubectl run으로 Pod를 직접 생성하는 방법과 Deployment로 Pod를 관리하는 방법, Pod의 개념과 생명주기 정리

---

## 1. 📖 개요 (Overview)

Pod는 Kubernetes에서 컨테이너를 실행하기 위한 가장 작은 단위이며, 쿠버네티스에서 실제 애플리케이션이 실행되는 가장 작은 실행 단위이다. 컨테이너를 담는 상자, 또는 하나의 서비스(웹 서버 등)가 동작하는 기본 단위로 볼 수 있다. Docker에서는 컨테이너가 직접 실행되지만, Kubernetes에서는 컨테이너가 Pod 안에서 실행되며, 쿠버네티스는 Pod 단위로 관리한다.

컨테이너를 직접 쓰지 않고 Pod를 사용하는 이유는 컨테이너를 하나의 실행 단위로 묶기 위해서다. 예를 들어 웹 서버 + 사이드카 로그 수집기(Fluentd)는 반드시 같은 네트워크/스토리지 공유가 필요하므로, Pod 하나 안에 넣어서 한 세트로 움직이게 만든다. 쿠버네티스 스케줄링·관리 단위가 Pod이기 때문에, 쿠버네티스는 컨테이너가 아니라 Pod 단위로 배포·복구·이동을 관리한다.

**Pod의 특징 3가지**

1. **IP를 가진다** — Pod는 생성되면 고유한 Pod IP가 하나 생긴다. Pod 안의 모든 컨테이너는 이 IP를 공유한다.
2. **네임스페이스(네트워크 + 파일시스템) 공유** — Pod 내부의 컨테이너들은 같은 네트워크(IP, 포트), 같은 localhost, 같은 Volume(스토리지)를 공유한다. 즉 Pod = 여러 컨테이너가 한 컴퓨터처럼 행동하는 구조.
3. **사라졌다가 다시 생길 수 있다** — Pod는 고정된 존재가 아니다. 재시작되면 IP도 달라진다. 장애가 나면 자동 복구할 수 있다. 그래서 Pod는 언제든 교체되는 일회성 실행 단위라고 보면 된다.

Pod는 1개 이상의 컨테이너를 포함할 수 있다. 1개가 가장 흔한 형태(nginx 한 개 실행)이며, 2개 이상은 사이드카 패턴(로그 Agent + 메인앱 등)이다. Pod가 컨테이너 모음이라고 설명하는 이유가 바로 이것이지만, 보통은 1컨테이너 = 1개의 Pod 구조로 사용한다.

Pod는 직접 만들지 않는 것이 원칙이다. 실무에서는 거의 절대로 `kubectl run`으로 Pod를 직접 띄우지 않는다. 왜냐하면 Pod는 장애가 발생하면 자동 복구가 안 되고, 수평 확장(Replica 증가/감소)도, 롤링 업데이트도 불가능하기 때문이다. 실무에서는 Deployment라는 컨트롤러가 Pod를 대신 만든다. `kubectl run`으로 직접 Pod를 만드는 방식은 테스트용이고, Deployment로 Pod를 관리하는 방식이 실무 방식이다.

Pod는 일회성 존재이다. 고정 IP가 아니라 재생성되면 IP가 바뀌고, 이름도 변경될 수 있으며(ReplicaSet이 새 Pod 생성), 고장 나면 새 Pod가 자동으로 생성되고(Deployment가 관리할 때), 일정 주기로 재시작될 수도 있다. 그래서 Pod는 고정된 서버라고 보면 안 되고, 언제든 교체되는 실행 단위라고 봐야 한다.

**Pod의 생명주기(Life Cycle)**

Pod는 다음과 같은 상태를 거친다.

```
Pending --> ContainerCreating --> Running --> Succeeded / Failed / CrashLoopBackOff
```

Pod가 Running이어도 내부 컨테이너는 계속 재시작될 수 있다.

---

## 2. 🛠️ Pod 직접 생성과 Deployment를 통한 관리

### 노드 상태 확인

```
[root@k8s-master ~]# kubectl api-resources    # 축약 명령어 확인

[root@k8s-master ~]# kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   81m   v1.35.7
k8s-worker1   Ready    <none>          57m   v1.35.7
k8s-worker2   Ready    <none>          56m   v1.35.7

[root@k8s-master ~]# kubectl get nodes -o wide
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                        KERNEL-VERSION                  CONTAINER-RUNTIME
k8s-master    Ready    control-plane   82m   v1.35.7   192.168.10.100    <none>        Rocky Linux 9.8 (Blue Onyx)     5.14.0-687.36.1.el9_8.x86_64    containerd://2.3.3
k8s-worker1   Ready    <none>          58m   v1.35.7   192.168.10.101    <none>        Rocky Linux 9.8 (Blue Onyx)     5.14.0-687.36.1.el9_8.x86_64    containerd://2.3.3
k8s-worker2   Ready    <none>          57m   v1.35.7   192.168.10.102    <none>        Rocky Linux 9.8 (Blue Onyx)     5.14.0-687.36.1.el9_8.x86_64    containerd://2.3.3
```

### `kubectl run`으로 Pod 직접 생성

```
[root@k8s-master ~]# kubectl run webserver --image=nginx:latest --port 80
pod/webserver created

[root@k8s-master ~]# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
webserver   1/1     Running   0          19s

[root@k8s-master ~]# kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE     IP           NODE          NOMINATED NODE   READINESS GATES
webserver   1/1     Running   0          2m28s   10.244.2.2   k8s-worker2   <none>            <none>
```

### `kubectl describe`로 Pod 상세 정보 확인

```
[root@k8s-master ~]# kubectl describe pods webserver
Name:             webserver
Namespace:        default
Priority:         0
Service Account:  default
Node:             k8s-worker2/192.168.10.102
Start Time:       Tue, 11 Aug 2026 16:55:38 +0900
Labels:           run=webserver
Annotations:      <none>
Status:           Running
IP:               10.244.2.2
IPs:
  IP:  10.244.2.2
Containers:
  webserver:
    Container ID:  containerd://ad56aba02ea28c6eb77cea71a14247eaa36c490efbf84e0bd968025553223522
    Image:         nginx:latest
    Image ID:      docker.io/library/nginx@sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8
    Port:          80/TCP
    Host Port:     0/TCP
    State:         Running
      Started:     Tue, 11 Aug 2026 16:55:47 +0900
    Ready:         True
    Restart Count: 0
    Environment:   <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-b89xh (ro)
Conditions:
  Type                       Status
  PodReadyToStartContainers  True
  Initialized                True
  Ready                      True
  ContainersReady            True
  PodScheduled               True
Volumes:
  kube-api-access-b89xh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:               <none>
Tolerations:                  node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                               node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason   Age    From               Message
  ----    ------   ----   ----               -------
  Normal  Scheduled  4m     default-scheduler  Successfully assigned default/webserver to k8s-worker2
  Normal  Pulling    3m59s  kubelet            spec.containers{webserver}: Pulling image "nginx:latest"
  Normal  Pulled     3m51s  kubelet            spec.containers{webserver}: Successfully pulled image "nginx:latest" in 7.815s (7.815s including waiting). Image size: 63135215 bytes.
  Normal  Created    3m51s  kubelet            spec.containers{webserver}: Container created
  Normal  Started    3m51s  kubelet            spec.containers{webserver}: Container started
```

### 두 번째 Pod 생성 및 확인

```
[root@k8s-master ~]# kubectl run webhttp --image=httpd --port 8080
pod/webhttp created

[root@k8s-master ~]# kubectl get pods
NAME        READY   STATUS              RESTARTS   AGE
webhttp     0/1     ContainerCreating   0          7s
webserver   1/1     Running             0          14m

[root@k8s-master ~]# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
webhttp     1/1     Running   0          38s
webserver   1/1     Running   0          15m

[root@k8s-master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   133m

[root@k8s-master ~]# kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
webhttp     1/1     Running   0          25m   10.244.1.2   k8s-worker1   <none>            <none>
webserver   1/1     Running   0          40m   10.244.2.2   k8s-worker2   <none>            <none>
```

### Pod IP로 직접 접속 확인

```
[root@k8s-master ~]# curl http://10.244.2.2
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

CLI 환경에서 텍스트 기반 브라우저로 확인하려면 EPEL 저장소와 elinks를 설치한다.

```bash
# EPEL(Extra Packages for Enterprise Linux) 저장소를 설치한다.
# Rocky Linux 기본 저장소에 없는 추가 패키지를 설치할 수 있도록 저장소를 추가한다.
[root@k8s-master ~]# dnf install -y epel-release

# 활성화된 Repository의 패키지 메타데이터를 내려받아 로컬 캐시를 생성/갱신한다.
# 이후 패키지를 검색하거나 설치할 때 저장소 정보를 빠르게 사용할 수 있다.
[root@k8s-master ~]# dnf makecache

# 텍스트 기반 웹 브라우저인 elinks를 설치한다.
# GUI가 없는 CLI 환경에서 웹 페이지에 접속하여 HTTP 서비스의 동작을 확인할 때 사용할 수 있다.
[root@k8s-master ~]# dnf install -y elinks

[root@k8s-master ~]# elinks 10.244.1.2
[root@k8s-master ~]# elinks 10.244.2.2
```

### Pod 삭제

```
[root@k8s-master ~]# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
webhttp     1/1     Running   0          33m
webserver   1/1     Running   0          48m

[root@k8s-master ~]# kubectl delete pods webhttp    # webhttp 이름의 파드 삭제
pod "webhttp" deleted from default namespace

[root@k8s-master ~]# kubectl delete pods webserver    # webserver 이름의 파드 삭제
pod "webserver" deleted from default namespace

[root@k8s-master ~]# kubectl get pods
No resources found in default namespace.
```

### Deployment로 Pod 관리 (실무 방식)

```
[root@k8s-master ~]# kubectl create deployment sol-deploy --image=nginx:latest --replicas=3
deployment.apps/sol-deploy created
```

명령의 각 요소는 다음과 같은 의미를 가진다.

- **deployment** — 리소스 종류(Type). Deployment는 Pod 자동 생성, Pod 개수 유지(replica 관리), Pod 자동 복구, 롤링업데이트, 롤백 기능을 가진 가장 중요한 쿠버네티스 컨트롤러다. 즉, 여러 개의 동일한 파드를 관리하는 상위 관리자다.
- **sol-deploy** — Deployment의 이름(Name). 리소스를 식별하는 고유 이름이며, kubectl 명령에서 `sol-deploy`라고 부르면 해당 Deployment만 조작한다. 예: `kubectl get deployment sol-deploy`, `kubectl delete deployment sol-deploy`.
- **--image=httpd:latest** — Pod 내부에서 실행할 컨테이너 이미지 지정. `httpd:latest`는 Apache 웹서버 최신 버전 이미지를 사용한다는 뜻이며, Deployment는 내부적으로 Pod 템플릿을 만들면서 컨테이너 image를 지정한다. Pod 3개가 생성되면 각 Pod 안에서 httpd 컨테이너가 실행된다.
- **--replicas=3** — Deployment가 생성해야 할 Pod 개수(Replica 수) 지정. `replicas=3`은 항상 3개의 Pod를 유지하라는 뜻이다. Pod 하나가 죽으면 새로운 Pod가 자동 생성되고, 노드가 죽어도 다른 노드에 다시 배치되며, 확장이 필요하면 `--replicas` 값을 변경할 수 있다.

```
[root@k8s-master ~]# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
sol-deploy-956c888c6-c9hng    1/1     Running   0          11s
sol-deploy-956c888c6-flnf6    1/1     Running   0          11s
sol-deploy-956c888c6-kcv2w    1/1     Running   0          11s

[root@k8s-master ~]# kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE          NOMINATED NODE   READINESS GATES
sol-deploy-956c888c6-c9hng    1/1     Running   0          4m22s   10.244.1.4   k8s-worker1   <none>            <none>
sol-deploy-956c888c6-flnf6    1/1     Running   0          5m59s   10.244.2.3   k8s-worker2   <none>            <none>
sol-deploy-956c888c6-kcv2w    1/1     Running   0          5m59s   10.244.1.3   k8s-worker1   <none>            <none>

[root@k8s-master ~]# kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
sol-deploy   3/3     3            3           8m16s
```

`kubectl get deployments` 출력의 각 컬럼은 다음을 의미한다.

| 컬럼 | 의미 |
|---|---|
| NAME | Deployment(배포)의 이름 |
| READY | 현재 Running 중인 파드 / 목표 파드 수 |
| UP-TO-DATE | 최신 버전으로 업데이트된 파드 수 |
| AVAILABLE | 외부 요청을 받을 준비가 된 파드 수, 건강한(Healthy) 파드의 개수 |
| AGE | Deployment가 생성된 시간 |

```
[root@k8s-master ~]# kubectl get pods sol-deploy-956c888c6-c9hng
NAME                          READY   STATUS    RESTARTS   AGE
sol-deploy-956c888c6-c9hng    1/1     Running   0          8m28s

[root@k8s-master ~]# kubectl get pods sol-deploy-956c888c6-c9hng -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE          NOMINATED NODE   READINESS GATES
sol-deploy-956c888c6-c9hng    1/1     Running   0          8m54s   10.244.1.4   k8s-worker1   <none>            <none>
```

### Deployment의 자동 복구 확인 — Pod를 강제 삭제해도 개수가 유지된다

```
[root@k8s-master ~]# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
sol-deploy-956c888c6-c9hng    1/1     Running   0          11s
sol-deploy-956c888c6-flnf6    1/1     Running   0          11s
sol-deploy-956c888c6-kcv2w    1/1     Running   0          11s

[root@k8s-master ~]# kubectl delete pods sol-deploy-956c888c6-c9hng
pod "sol-deploy-956c888c6-c9hng" deleted from default namespace

[root@k8s-master ~]# kubectl get pods
NAME                          READY   STATUS              RESTARTS   AGE
sol-deploy-956c888c6-flnf6    1/1     Running             0          11m
sol-deploy-956c888c6-kcv2w    1/1     Running             0          11m
sol-deploy-956c888c6-pthxd    0/1     ContainerCreating   0          3s
```

Pod를 하나 강제로 삭제해도, Deployment가 `replicas: 3` 상태를 유지하기 위해 즉시 새 Pod(`sol-deploy-956c888c6-pthxd`)를 생성하는 것을 확인할 수 있다.

### `kubectl edit`으로 replicas 값 수정

```
[root@k8s-master ~]# kubectl edit deployments.apps sol-deploy
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2026-08-11T08:47:53Z"
  generation: 1
  labels:
    app: sol-deploy
  name: sol-deploy
  namespace: default
  resourceVersion: "14719"
  uid: 83f4e5e4-4792-4176-82ce-f39aa2cd41fe
spec:
  progressDeadlineSeconds: 600
  replicas: 5            # replicas를 3에서 5로 수정
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: sol-deploy
  strategy:

:wq
```

```
[root@k8s-master ~]# kubectl get pods
NAME                          READY   STATUS              RESTARTS   AGE
sol-deploy-956c888c6-bk8hc    0/1     ContainerCreating   0          2s      # pod 생성
sol-deploy-956c888c6-bvbqc    0/1     ContainerCreating   0          2s      # pod 생성
sol-deploy-956c888c6-flnf6    1/1     Running             0          17m
sol-deploy-956c888c6-kcv2w    1/1     Running             0          17m
sol-deploy-956c888c6-pthxd    1/1     Running             0          5m42s
```

### Deployment 삭제

```
[root@k8s-master ~]# kubectl delete deployments.apps sol-deploy
deployment.apps "sol-deploy" deleted from default namespace

[root@k8s-master ~]# kubectl get pods
No resources found in default namespace
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `kubectl get pods`의 STATUS가 `ContainerCreating`에서 멈춰 있으면 이미지 pull이 진행 중이거나 실패한 것이므로 `kubectl describe pods <이름>`의 Events 섹션을 확인한다.
- Pod를 `kubectl run`으로 직접 생성한 경우 삭제하면 그대로 사라지지만, Deployment가 관리하는 Pod는 삭제해도 즉시 새 Pod가 재생성되므로 개수 유지를 확인하려면 Deployment 자체를 삭제해야 한다(`kubectl delete deployments.apps <이름>`).
- Pod의 IP는 재생성 시마다 바뀌므로 `curl`이나 `elinks`로 직접 Pod IP에 접속해 테스트하는 것은 실습 목적일 뿐이며, 실제 서비스 노출에는 Service 리소스를 사용해야 한다.

---

> 📌 **핵심 요약**
> - Pod는 Kubernetes 최소 실행 단위이며 IP를 가지고, 네트워크·파일시스템을 공유하며, 언제든 재생성될 수 있는 일회성 존재
> - Pod 생명주기: `Pending → ContainerCreating → Running → Succeeded / Failed / CrashLoopBackOff`
> - `kubectl run`으로 Pod를 직접 생성하는 것은 테스트용이며, 자동 복구·확장·롤링 업데이트가 불가능
> - 실무에서는 `kubectl create deployment`로 Deployment를 생성해 Pod 개수(`--replicas`)를 유지·자동 복구하며 관리
> - 관련: 1. 🔧 Kubernetes - 설치 · 3. 🏗️ Kubernetes - 아키텍처 개요와 핵심 컴포넌트 · 10. 🎛️ Kubernetes - Controller 개념과 ReplicationController
