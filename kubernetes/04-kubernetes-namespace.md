# 📦 Kubernetes - Namespace

> **Tag:** #Kubernetes #Namespace #kubectl #부트캠프
> **핵심 요약:** namespace로 클러스터를 여러 가상 공간으로 나눠 리소스를 격리하는 방법, CLI/YAML로 namespace를 생성·삭제하는 방법, `kubectl config`로 base namespace를 전환하는 방법 정리

---

## 1. 📦 namespace — 클러스터 안의 가상 공간

namespace는 한 개의 쿠버네티스 클러스터를 여러 개의 가상의 공간으로 나눠서 리소스를 서로 섞이지 않게 관리하는 기능이다.

대표적인 리소스: Pod, Deployment, Service, ConfigMap, Secret

쿠버네티스를 수업에서 하나의 큰 교실(클러스터)이라고 치면 namespace는 조별로 나눠진 각 조의 책상 구역이다.

namespace가 없으면 모든 조가 같은 공간에 파일(리소스)을 놓는 것과 같아서 이름이 충돌하거나 누가 만든 건지 헷갈리고 실수로 다른 리소스를 지울 수 있다.

namespace가 있으면 1조는 1조 공간에서만 작업, 2조는 2조 공간에서만 작업하고, 같은 이름의 리소스도 각 공간에서 따로 만들 수 있다.

예: 1조 namespace : web Deployment / 2조 namespace : web Deployment — 이게 동시에 가능하다(서로 다른 공간이므로 같은 이름을 사용할 수 있다).

**namespace 특징 3가지**

- **분리(격리)** — 팀/서비스/환경별로 리소스를 분리해서 관리. dev, test, prod를 namespace로 분리.
- **이름 충돌 방지** — 같은 이름을 서로 다른 namespace에 만들 수 있음. 예: dev/web, prod/web.
- **권한과 자원 제한 적용** — 특정 namespace에만 접근 가능하게 권한을 줄 수 있다(RBAC). 특정 namespace는 CPU/메모리를 얼마까지 쓰게 제한할 수 있다(ResourceQuota).

**namespace에 속하는 리소스(대부분)**
- Pod
- Deployment
- ReplicaSet
- Service
- ConfigMap
- Secret
- Ingress(설치 방식에 따라 다를 수 있음)
- Job, CronJob 등

**namespace에 속하지 않는 리소스(클러스터 전체 공용)**
- Node
- PersistentVolume(PV)
- StorageClass
- Namespace 자체
- ClusterRole / ClusterRoleBinding(권한 관련)

### namespace 생성 — CLI 방식과 YAML 방식

네임스페이스는 CLI를 사용하는 방식과 YAML을 사용하는 방식으로 생성이 가능하다.

**CLI 방식**

```
kubectl create namespace soldesk
```

`soldesk`라는 이름의 네임스페이스를 즉시 생성한다. 내부적으로 kube-api-server를 통해 etcd에 저장된다.

생성 확인:

```
kubectl get namespaces
kubectl get ns
```

**YAML 방식**

```
kubectl create namespace soldesk --dry-run=client -o yaml > soldesk-ns.yaml
```

- `create namespace soldesk` — soldesk 네임스페이스를 만들 명령
- `--dry-run=client` — 실제로 만들지는 말고 결과만 출력
- `-o yaml` — 출력 형식을 YAML로 변환
- `soldesk-ns.yaml` — 결과를 파일로 저장

### namespace 확인 및 kube-system 파드 조회

```
[root@k8s-master ~]# kubectl  get  nodes
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   20h     v1.35.7
k8s-worker1   Ready    <none>          20h     v1.35.7
k8s-worker2   Ready    <none>          20h     v1.35.7



[root@k8s-master ~]# kubectl  get  namespaces
NAME              STATUS   AGE
default           Active      20h
kube-flannel      Active      20h
kube-node-lease   Active      20h
kube-public       Active      20h
kube-system       Active      20h



[root@k8s-master ~]# kubectl  get  pod  --namespace kube-system
NAME                                        READY    STATUS    RESTARTS       AGE
coredns-7d764666f9-2dwdd             1/1     Running     1 (137m ago)     20h
coredns-7d764666f9-klkhq             1/1     Running     1 (137m ago)     20h
etcd-k8s-master                      1/1     Running     2 (137m ago)     20h
kube-apiserver-k8s-master            1/1     Running     2 (137m ago)     20h
kube-controller-manager-k8s-master   1/1     Running     2 (137m ago)     20h
kube-proxy-bxjvw                     1/1     Running     1 (157m ago)     20h
kube-proxy-gbr4f                     1/1     Running     1 (157m ago)     20h
kube-proxy-rwhkc                     1/1     Running     1 (137m ago)     20h
kube-scheduler-k8s-master            1/1     Running     2 (137m ago)     20h



[root@k8s-master ~]# kubectl  get  pod  -n kube-flannel
NAME                           READY    STATUS    RESTARTS       AGE
kube-flannel-ds-7hlrp   1/1     Running     1 (138m ago)     20h
kube-flannel-ds-f9wcz   1/1     Running     1 (158m ago)     20h
kube-flannel-ds-z4fbv   1/1     Running     2 (149m ago)     20h
```

`kube-system` 파드들의 역할은 각각 다음과 같다.

| 파드 | 역할 |
|---|---|
| coredns-... | 클러스터 내부 DNS 역할. Service 이름을 IP로 해석해 주는 핵심 컴포넌트. Pod 간 서비스 이름 기반 통신에 사용. 두 개가 뜨는 것은 CoreDNS 이중화용 Pod로, 하나에 문제가 생겨도 DNS 서비스가 유지되도록 복수 개 실행 |
| etcd-k8s-master | Kubernetes 클러스터의 모든 상태 정보를 저장하는 Key-Value 저장소. Pod, Deployment, Service, ConfigMap 등의 상태 정보 저장 |
| kube-apiserver-k8s-master | Kubernetes API의 중심. kubectl, Controller, Scheduler 등의 요청을 처리하는 API Server. Kubernetes의 모든 제어 요청이 이 컴포넌트를 통해 전달됨 |
| kube-controller-manager-k8s-master | 원하는 상태(Desired State)를 유지하도록 감시하는 Controller 모음. Pod 개수 유지, Node 상태 감시, 자동 복구 등을 담당 |
| kube-scheduler-k8s-master | 새로 생성되는 Pod를 어떤 Node에 배치할지 결정. 실제 Pod 실행은 하지 않고 배치할 Node만 결정 |
| kube-proxy-* | 각 Node에서 네트워크 통신을 관리하는 컴포넌트. Service로 들어온 트래픽을 실제 Pod로 전달. iptables 또는 nftables 등의 네트워크 규칙을 관리. 노드마다 하나씩 실행됨 |

### namespace 생성/삭제와 Pod 생성 실습

```
[root@k8s-master ~]# vi  nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod

spec:
  containers:
    - name: mynginx
      image: nginx:1.31
      ports:
        - containerPort: 80

:wq




[root@k8s-master ~]# cat nginx.yaml
apiVersion: v1              # 쿠버네티스 API 버전 (Pod는 v1 사용)
kind: Pod                   # 생성할 리소스 종류 (Pod)
metadata:                   # 리소스의 메타데이터 영역 (이 리소스를 쿠버네티스가 식별하고 관리하기 위한 정보)
  name: mypod               # Pod이름(쿠버네티스에서 리소스를 식별하는 기준, 네임스페이스 내에서 유일해야 함)

spec:                       # Pod의 실제 동작 정의 영역
  containers:               # 이 Pod에서 실행될 컨테이너 목록
    - name: nginx             # 컨테이너 이름
      image: nginx:1.31       # 사용할 컨테이너 이미지 (Docker Hub 공식 nginx)
      ports:                  # 컨테이너가 사용하는 포트 정보 (문서화/Service 연동용)
        - containerPort: 80     # 컨테이너 내부에서 사용하는 HTTP 포트




[root@k8s-master ~]# kubectl  get  pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1         Running     0                35s



[root@k8s-master ~]# kubectl  get  pods  -n  default
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1         Running     0                35s



# create로 다시 실행하게되면 이미 mypod 이름의 파드가 있기때문에 에러가 발생한다.
[root@k8s-master ~]# kubectl  create  -f  nginx.yaml
Error from server (AlreadyExists): error when creating "nginx.yaml": pods "mypod" already exists



[root@k8s-master ~]# kubectl  delete  -f  nginx.yaml
pod "mypod" deleted from default namespace
```

### apply로 실행 — create와의 차이

`kubectl create`는 이미 존재하면 에러가 나지만, `kubectl apply`는 없으면 생성하고 있으면 변경 없이 그대로 둔다(멱등성).

```
[root@k8s-master ~]# kubectl  apply  -f  nginx.yaml
pod/mypod created


[root@k8s-master ~]# kubectl  get  pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1         Running     0                35s


[root@k8s-master ~]# kubectl  get  pods  -n  default
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1         Running     0                35s



[root@k8s-master ~]# kubectl  apply  -f  nginx.yaml
pod/mypod unchanged
```

### namespace 생성/삭제

```
					# namespace 생성

[root@k8s-master ~]# kubectl  create  namespace  soldesk
namespace/soldesk created


[root@k8s-master ~]# kubectl  get  namespace
NAME              STATUS   AGE
default           Active      21h
kube-flannel      Active      21h
kube-node-lease   Active      21h
kube-public       Active      21h
kube-system       Active      21h
soldesk        Active      34s



[root@k8s-master ~]# kubectl  create  namespace soldesk  --dry-run=client  -o  yaml
apiVersion: v1
kind: Namespace
metadata:
  name: soldesk
spec: {}
status: {}




	# namespace 삭제
[root@k8s-master ~]# kubectl  delete  namespaces  soldesk
namespace "soldesk" deleted




[root@k8s-master ~]# kubectl  create  namespace soldesk  --dry-run=client  -o  yaml > sol-namespace.yaml



[root@k8s-master ~]# ls -l
합계 16
-rw-------. 1 root root 1027  7월  2 12:55 anaconda-ks.cfg
-rw-r--r--  1 root root   152  8월 12 10:17 nginx-pod.yaml
-rw-r--r--  1 root root   156  8월 12 12:27 nginx.yaml
-rw-r--r--  1 root root    77  8월 12 12:50 sol-namespace.yaml




[root@k8s-master ~]# vi  sol-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: soldesk
spec: {}		# 삭제
status: {}		# 삭제



[root@k8s-master ~]# cat  sol-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: soldesk



[root@k8s-master ~]# kubectl  create  -f  sol-namespace.yaml
namespace/soldesk created




	# 같은 의미
[root@k8s-master ~]# kubectl  run  --image=nginx:latest webserver  --namespace  soldesk
pod/pods created



	# 같은 의미
[root@k8s-master ~]# kubectl  run  --image=nginx:latest webserver  -n soldesk
pod/pods created




	# 같은 의미
[root@k8s-master ~]# kubectl  get  pods  -n  soldesk    --output  wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
webserve    1/1         Running     0                27m    10.244.2.10   k8s-worker2   <none>                   <none>


	# 같은 의미
[root@k8s-master ~]# kubectl  get  pods  -n  soldesk    -o  wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
webserve    1/1         Running     0                27m    10.244.2.10   k8s-worker2   <none>                   <none>
```

### `-` 와 `--` 옵션 차이

리눅스 명령어와 kubectl에서는 옵션을 크게 짧은 옵션(short option)과 긴 옵션(long option)으로 구분한다.

**짧은 옵션 (Short Option)**
- 옵션 앞에 `-`를 사용한다.
- 한 글자로 표현하는 경우가 많다.
- 명령어를 빠르게 입력할 수 있다.

예) `-n`, `-o`, `-f`

**긴 옵션 (Long Option)**
- 옵션 앞에 `--`를 사용한다.
- 옵션의 전체 이름을 사용하기 때문에 의미를 이해하기 쉽다.

예) `--namespace`, `--output`, `--filename`, `--image`

**짧은 옵션과 긴 옵션 비교**

| 짧은 옵션 | 긴 옵션 |
|---|---|
| `-n` | `--namespace` |
| `-o` | `--output` |
| `-f` | `--filename` |

예1)

```
kubectl get pods -n soldesk
kubectl get pods --namespace soldesk
```

두 명령어는 같은 의미이며, soldesk Namespace의 Pod를 조회한다.

예2)

```
kubectl get pods -o wide
kubectl get pods --output wide
```

두 명령어는 같은 의미이며, Pod의 IP, Node 등의 추가 정보를 출력한다.

예3)

```
kubectl apply -f nginx.yaml
kubectl apply --filename nginx.yaml
```

두 명령어는 같은 의미이며, nginx.yaml 파일을 사용하여 리소스를 적용한다.

### namespace에 pod 생성 (하나의 YAML에 여러 리소스)

```
			# namespace에 pod 생성

[root@k8s-master ~]#  vi  mynginx.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: studydesk

---
apiVersion: v1
kind: Pod
metadata:
  name: studyweb
  namespace: studydesk

spec:
  containers:
    - name: nginxweb
      image: nginx:1.31
      ports:
        - containerPort: 80

:wq



[root@k8s-master ~]# kubectl  create  -f  mynginx.yaml  --dry-run=client
namespace/studydesk created (dry run)
pod/studyweb created (dry run)


[root@k8s-master ~]# kubectl  create  -f  mynginx.yaml
namespace/studydesk created



[root@k8s-master ~]# kubectl  get  namespaces
NAME              STATUS   AGE
default           Active      23h
kube-flannel      Active      23h
kube-node-lease   Active      23h
kube-public       Active      23h
kube-system       Active      23h
studydesk         Active      8s



[root@k8s-master ~]# kubectl  get  pods  -n studydesk
NAME       READY   STATUS    RESTARTS   AGE
studyweb   1/1         Running     0                109s



[root@k8s-master ~]# kubectl  get  pods  --all-namespaces
NAMESPACE      NAME                                 READY   STATUS    RESTARTS        AGE
kube-flannel   kube-flannel-ds-7hlrp                1/1         Running     1 (4h59m ago)     23h
kube-flannel   kube-flannel-ds-f9wcz                1/1         Running     1 (5h19m ago)     23h
kube-flannel   kube-flannel-ds-z4fbv                1/1         Running     2 (5h10m ago)     23h
kube-system    coredns-7d764666f9-2dwdd             1/1         Running     1 (4h59m ago)     23h
kube-system    coredns-7d764666f9-klkhq             1/1         Running     1 (4h59m ago)     23h
kube-system    etcd-k8s-master                      1/1         Running     2 (4h59m ago)     23h
kube-system    kube-apiserver-k8s-master            1/1         Running     2 (4h59m ago)     23h
kube-system    kube-controller-manager-k8s-master   1/1         Running     2 (4h59m ago)     23h
kube-system    kube-proxy-bxjvw                     1/1         Running     1 (5h19m ago)     23h
kube-system    kube-proxy-gbr4f                     1/1         Running     1 (5h19m ago)     23h
kube-system    kube-proxy-rwhkc                     1/1         Running     1 (4h59m ago)     23h
kube-system    kube-scheduler-k8s-master            1/1         Running     2 (4h59m ago)     23h
studydesk      studyweb                             1/1         Running     0                     3m17s



[root@k8s-master ~]# kubectl  delete  -f  mynginx.yaml
namespace "studydesk" deleted
pod "studyweb" deleted from studydesk namespace
```

### Base namespace 변경 — kubectl config로 기본 namespace 지정

예를 들어 이번주는 계속 soldesk의 namespace에 관련된 작업을 수행해야 하는데 기본값으로 default namespace로 설정되어 있다.

작업 시 namespace를 변경하기 위해서 `-n soldesk`(`--namespace`) 명령어를 계속 사용해야 한다.

변경 방법: k8s의 config 파일에 namespace를 등록해야 한다.

```
[root@k8s-master ~]# kubectl  config  --help
Modify kubeconfig files using subcommands like "kubectl config set
current-context my-context".
~~~~~~~~~~ 중간 생략 ~~~~~~~~~~
Available Commands:
  current-context   Display the current-context
  delete-cluster    kubeconfig에서 지정된 클러스터를 삭제합니다
  delete-context    kubeconfig에서 지정된 컨텍스트를 삭제합니다
  delete-user       Delete the specified user from the kubeconfig
  get-clusters      kubeconfig에 정의된 클러스터를 표시합니다
  get-contexts      하나 또는 여러 컨텍스트를 설명합니다
  get-users         Display users defined in the kubeconfig
  rename-context    Rename a context from the kubeconfig file
  set               Set an individual value in a kubeconfig file
  set-cluster       Set a cluster entry in kubeconfig
  set-context       Set a context entry in kubeconfig
  set-credentials   Set a user entry in kubeconfig
  unset             Unset an individual value in a kubeconfig file
  use-context       Set the current-context in a kubeconfig file
  view              병합된 kubeconfig 설정 또는 지정된 kubeconfig 파일을 표시합니
~~~~~~~~~~ 중간 생략 ~~~~~~~~~~



[root@k8s-master ~]# kubectl  config  view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.10.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED



[root@k8s-master ~]# kubectl  config  \
  set-context  \
  soldesk@kubernetes \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --namespace=soldesk
Context "soldesk@kubernetes" created.


# kubectl  config		: kubectl 설정 파일 관리
# set-context		: 새로운 context 생성 또는 수정
# soldesk@kubernete	: 생성할 context 이름 (여러 context를 구분하기위한 이름)
# --cluster=kubernetes	: 해당 context를 생성할 클러스터를 지정
# --user=kubernetes-admin	: kubenetes를 접속시 사용할 계정 (현재 생성되어있는 계정을 그대로 사용)
# --namespace=soldesk	: 해당 context를 사용시 namespace를 soldesk로 지정
```

`set-context`를 실행하면 `kubectl config view`에 새로운 context가 추가된 것을 확인할 수 있다.

```
[root@k8s-master ~]# kubectl  config  view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.10.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: kubernetes
    namespace: soldesk
    user: kubernetes-admin
  name: soldesk@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED


[root@k8s-master ~]# ls  -la
합계 60
dr-xr-x---. 15 root root 4096  8월 12 14:42 .
dr-xr-xr-x. 18 root root  255  7월  2 15:53 ..
-rw-------.  1 root root 3774  8월 12 09:47 .bash_history
-rw-r--r--.  1 root root   18  5월 11  2022 .bash_logout
-rw-r--r--.  1 root root  141  5월 11  2022 .bash_profile
-rw-r--r--.  1 root root  463  8월 11 16:00 .bashrc
drwx------.  8 root root  108  7월  2 15:10 .cache
drwx------. 10 root root 4096  7월  2 15:10 .config
-rw-r--r--.  1 root root  100  5월 11  2022 .cshrc
drwxr-xr-x   3 root root   33  8월 12 15:00 .kube		<----
-rw-------   1 root root   20  8월 11 15:20 .lesshst
drwx------.  3 root root   19  7월  2 15:10 .local
drwx------.  2 root root    6  7월  2 15:08 .ssh
-rw-r--r--.  1 root root  129  5월 11  2022 .tcshrc
-rw-------.  1 root root 1027  7월  2 12:55 anaconda-ks.cfg
-rw-r--r--   1 root root  246  8월 12 14:42 mynginx.yaml
-rw-r--r--   1 root root  152  8월 12 10:17 nginx-pod.yaml
-rw-r--r--   1 root root  156  8월 12 12:27 nginx.yaml
-rw-r--r--   1 root root   57  8월 12 12:51 sol-namespace.yaml
~~~~~~~~~~ 중간 생략 ~~~~~~~~~~



[root@k8s-master ~]# cat  ./.kube/config
apiVersion: v1
clusters:
- cluster:
~~~~~~~~~~ 중간 생략 ~~~~~~~~~~
    server: https://192.168.10.100:6443
  name: kubernetes
contexts:
- context:					# 기본으로 생성된 context
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:					# 새로 생성한 context
    cluster: kubernetes
    namespace: soldesk
    user: kubernetes-admin
  name: soldesk@kubernetes
current-context: kubernetes-admin@kubernetes	# 현재 사용중인 context (current-context를 변경하게되면 base namespace가 변경된다.)
kind: Config
users:
- name: kubernetes-admin
  user:
~~~~~~~~~~ 중간 생략 ~~~~~~~~~~



	# 사용할 context를 변경

[root@k8s-master ~]# kubectl  config  use-context soldesk@kubernetes
Switched to context "soldesk@kubernetes".




[root@k8s-master ~]# kubectl  config   view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.10.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: kubernetes
    namespace: soldesk
    user: kubernetes-admin
  name: soldesk@kubernetes
current-context: soldesk@kubernetes		# current-context가 kubernetes-admin@kubernetes에서 soldesk@kubernetes로 변경된것을 확인
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED



[root@k8s-master ~]# kubectl  create  namespace  soldesk
namespace/soldesk created


[root@k8s-master ~]# kubectl  run  --image=nginx:1.31  nameweb  --port 80
pod/nameweb created


[root@k8s-master ~]# kubectl get  pods
NAME       READY   STATUS    RESTARTS   AGE
nameweb   1/1         Running     0                 31s



[root@k8s-master ~]# kubectl get  pods  --all-namespaces
NAMESPACE      NAME                                 READY   STATUS    RESTARTS        AGE
kube-flannel   kube-flannel-ds-7hlrp                1/1         Running     1 (5h30m ago)     23h
kube-flannel   kube-flannel-ds-f9wcz                1/1         Running     1 (5h50m ago)     23h
kube-flannel   kube-flannel-ds-z4fbv                1/1         Running     2 (5h41m ago)     23h
kube-system    coredns-7d764666f9-2dwdd             1/1         Running     1 (5h30m ago)     23h
kube-system    coredns-7d764666f9-klkhq             1/1         Running     1 (5h30m ago)     23h
kube-system    etcd-k8s-master                      1/1         Running     2 (5h30m ago)     23h
kube-system    kube-apiserver-k8s-master            1/1         Running     2 (5h30m ago)     23h
kube-system    kube-controller-manager-k8s-master   1/1         Running     2 (5h30m ago)     23h
kube-system    kube-proxy-bxjvw                     1/1         Running     1 (5h50m ago)     23h
kube-system    kube-proxy-gbr4f                     1/1         Running     1 (5h50m ago)     23h
kube-system    kube-proxy-rwhkc                     1/1         Running     1 (5h30m ago)     23h
kube-system    kube-scheduler-k8s-master            1/1         Running     2 (5h30m ago)     23h
soldesk        nameweb                              1/1         Running     0                      59s


[root@k8s-master ~]# kubectl  create  namespace  studydesk
namespace/studydesk created

-default namespace
-soldesk namespace
-studydesk namespace


[root@k8s-master ~]# vi nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web1
spec:
  containers:
    - name: mynginx
      image: nginx:1.31
      ports:
        - containerPort: 80

---
apiVersion: v1
kind: Pod
metadata:
  name: web2
  namespace: studydesk
spec:
  containers:
    - name: mynginx
      image: nginx:1.31
      ports:
        - containerPort: 80


---
apiVersion: v1
kind: Pod
metadata:
  name: web3
  namespace: default
spec:
  containers:
    - name: mynginx
      image: nginx:1.31
      ports:
        - containerPort: 80




[root@k8s-master ~]# kubectl  apply  -f  nginx.yaml --dry-run=client
pod/web1 created (dry run)
pod/web2 created (dry run)
pod/web3 created (dry run)



[root@k8s-master ~]# kubectl  apply  -f  nginx.yaml
pod/web1 created
pod/web2 created
pod/web3 created



[root@k8s-master ~]# kubectl  get  pod
NAME  	  READY   STATUS    RESTARTS   AGE
nameweb	  1/1         Running     0                29m
web1 	  1/1         Running     0                3s


[root@k8s-master ~]# kubectl  get  pod  -n  soldesk
NAME  	  READY   STATUS    RESTARTS   AGE
nameweb	  1/1         Running     0                29m
web1 	  1/1         Running     0                3s


[root@k8s-master ~]# kubectl  get  pod  -n  studydesk
NAME   READY   STATUS    RESTARTS   AGE
web2     1/1         Running     0                42s


[root@k8s-master ~]# kubectl  get  pod  -n  default
NAME   READY   STATUS    RESTARTS   AGE
web3     1/1         Running     0                46s



[root@k8s-master ~]# kubectl  get  pod  --all-namespaces
NAMESPACE      	NAME                               		READY   STATUS    RESTARTS        	AGE
default        	web3                                 		1/1         Running     0               	78s
kube-flannel   	kube-flannel-ds-7hlrp                	1/1         Running     1 (6h ago)      	24h
kube-flannel   	kube-flannel-ds-f9wcz                	1/1         Running     1 (6h20m ago)   	24h
kube-flannel   	kube-flannel-ds-z4fbv                	1/1         Running     2 (6h11m ago)   	24h
kube-system    	coredns-7d764666f9-2dwdd             	1/1         Running     1 (6h ago)      	24h
kube-system    	coredns-7d764666f9-klkhq             	1/1         Running     1 (6h ago)      	24h
kube-system    	etcd-k8s-master                      	1/1         Running     2 (6h ago)      	24h
kube-system    	kube-apiserver-k8s-master            	1/1         Running     2 (6h ago)      	24h
kube-system    	kube-controller-manager-k8s-master   	1/1         Running     2 (6h ago)      	24h
kube-system    	kube-proxy-bxjvw                     	1/1         Running     1 (6h20m ago)   	24h
kube-system    	kube-proxy-gbr4f                     	1/1         Running     1 (6h20m ago)   	24h
kube-system    	kube-proxy-rwhkc                     	1/1         Running     1 (6h ago)      	24h
kube-system    	kube-scheduler-k8s-master            	1/1         Running     2 (6h ago)      	24h
soldesk        	nameweb                              	1/1         Running     0               	30m
soldesk        	web1                                 	1/1         Running     0               	78s
studydesk      	web2                                 	1/1         Running     0               	78s



[root@k8s-master ~]# kubectl  delete  pods  --all		# soldesk namespace의 pods들만 삭제된다.
pod "nameweb" deleted from soldesk namespace
pod "web1" deleted from soldesk namespace




[root@k8s-master ~]# kubectl  config  use-context kubernetes-admin@kubernetes
Switched to context "kubernetes-admin@kubernetes".



[root@k8s-master ~]# kubectl  delete  pods  --all
pod "web3" deleted from default namespace



[root@k8s-master ~]# kubectl  get  pods
No resources found in default namespace.
```

context를 `soldesk@kubernetes`로 전환한 상태에서 `-n`을 붙이지 않고 `kubectl delete pods --all`을 실행하면 현재 context의 base namespace인 soldesk의 파드만 삭제되고, 다시 `kubernetes-admin@kubernetes`로 전환하면 default namespace가 대상이 되는 것을 확인할 수 있다.

### 커스텀 Nginx 이미지 생성 및 Kubernetes 배포 실습

control-plane에서 직접 Nginx 커스텀 이미지를 생성한 후 Docker Hub에 Push하고, Kubernetes의 Worker Node가 해당 이미지를 Pull하여 Pod와 Deployment를 실행하는 전체 과정을 실습한다.

Kubernetes에서 Pod를 생성할 때 control-plane의 로컬 이미지를 직접 사용하는 것이 아니라, Docker Hub와 같은 Container Registry에 저장된 이미지를 Worker Node가 Pull하여 실행하는 구조를 이해한다.

**흐름: 이미지 생성 → 태그 변경 → Docker Hub Push → Kubernetes 배포 → Pod 실행 → 확인**

**1단계. control-plane에서 이미지 빌드 후 docker tag로 태그 변경하여 Hub push**

```
[root@k8s-master ~]# mkdir  -p  /root/nginx-image-lab


[root@k8s-master ~]# cd ./nginx-image-lab/


[root@k8s-master nginx-image-lab]# pwd
/root/nginx-image-lab
```

**2단계. index.html 작성 (이미지에 들어갈 실제 콘텐츠)**

```
[root@k8s-master nginx-image-lab]# vi index.html 
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>custom nginx</title>
</head>
<body>
  <h1>custom nginx image</h1>
  <p>built on control-plane</p>
</body>
</html>
```

**3단계. Dockerfile 작성 (이미지 설계도)**

```
[root@k8s-master nginx-image-lab]# vi dockerfile
FROM  nginx:1.31


RUN  apt-get update && \
apt-get install -y \
curl \
iputils-ping \
iproute2 \
vim

ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

COPY index.html  /usr/share/nginx/html/index.html

:wq



[root@k8s-master nginx-image-lab]# cat dockerfile
FROM  nginx:1.31


RUN  apt-get update && \
apt-get install -y \
curl \
iputils-ping \
iproute2 \
vim

ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

COPY index.html  /usr/share/nginx/html/index.html
```

**4단계. 이미지 빌드 (로컬 태그로 이미지 생성)**

- 이미지 이름 : custom-nginx-web
- 태그 : 1.31

```
[root@k8s-master nginx-image-lab]# docker  build  -t  custom-nginx-web:1.31  .
[+] Building 15.6s (8/8) FINISHED                                                                docker:default
 => [internal] load build definition from dockerfile                                             0.0s
 => => transferring dockerfile: 229B                                                              0.0s
 => [internal] load metadata for docker.io/library/nginx:1.31                                     4.2s
 => [internal] load .dockerignore                                                                 0.0s
 => => transferring context: 2B                                                                   0.0s
 => [internal] load build context                                                                 0.0s
 => => transferring context: 1.05kB                                                                0.0s
 => [1/3] FROM docker.io/library/nginx:1.31@sha256:8541484afbc9c8a5a8a99b379568ebbc957f6           4.0s
 => => resolve docker.io/library/nginx:1.31@sha256:8541484afbc9c8a5a8a99b379568ebbc957f6           0.0s
 => => sha256:f5de6e85ac74fc63dae94ac7ac121d8e3755f1743efac65eeaef983b35 1.21kB / 1.21kB           0.8s
 => => sha256:5a4222b844e843499b76e3eb9f0088b1812e432c6965a1f50d48efc4d9 1.40kB / 1.40kB           0.4s
 => => sha256:c0df8d325117373948c15350cc4887825c8708961670514c384a9f6ba86403 954B / 954B           0.9s
 => => sha256:26c307b5e35a59ce911f5fde5b9458120ec8734e831ea2da5649a9ad 29.78MB / 29.78MB           1.6s
 => => extracting sha256:26c307b5e35a59ce911f5fde5b9458120ec8734e831ea2da5649a9ad14abfd3           0.6s
 => => extracting sha256:3c55dc422a8172d2ad90565ea86c2211e5d9b1c854db7ecf506ece263e0d3fe           0.7s
 => => extracting sha256:f5de6e85ac74fc63dae94ac7ac121d8e3755f1743efac65eeaef983b35767b2           0.0s
 => => extracting sha256:5a4222b844e843499b76e3eb9f0088b1812e432c6965a1f50d48efc4d99cd0c           0.0s
 => [2/3] RUN  apt-get update && apt-get install -y curl iputils-ping iproute2 vim                 4.5s
 => [3/3] COPY index.html  /usr/share/nginx/html/index.html                                        0.1s
 => exporting to image                                                                             2.7s
 => => exporting layers                                                                            2.1s
 => => exporting manifest sha256:1f4fdb7cd6780b18a1ad0f8ff1ac2d3abc1289a90cad3b306d6e8cd            0.0s
 => => exporting config sha256:843b19a146c3b4a2b847828bf90560692bb44d1187380abe6823da6d3            0.0s
 => => exporting attestation manifest sha256:56519eed496931fa952d2b471c3fc27f0da6ea2b8a9            0.0s
 => => exporting manifest list sha256:02218a3d52d69c10c071c3d5a426cffb6e56d61a4f72d5373b            0.0s
 => => naming to docker.io/library/custom-nginx-web:1.31                                           0.0s
 => => unpacking to docker.io/library/custom-nginx-web:1.31                                        0.6s




[root@k8s-master ~]# docker  images
IMAGE                             	ID             	DISK USAGE   CONTENT SIZE   EXTRA
custom-nginx-web:1.31             	02218a3d52d6        	348MB            97MB
```

**5단계. docker tag로 Hub용 이미지 태그 생성**

`docker tag`는 이미지를 새로 만드는 게 아니라 기존 이미지에 이름표를 하나 더 붙이는 것이다.

- Docker Hub ID 사용
- 형식: `<dockerhub_id>/custom-nginx-web:1.31`

```
[root@k8s-master ~]# docker tag  custom-nginx-web:1.31  konan7979/custom-nginx-web:1.31



[root@k8s-master ~]# docker  images
IMAGE                             	ID             	DISK USAGE   CONTENT SIZE   EXTRA
custom-nginx-web:1.31             	02218a3d52d6        	348MB            97MB
konan7979/custom-nginx-web:1.31   	02218a3d52d6        	348MB            97MB
```

**6단계. Docker Hub 로그인**

```
root@k8s-master nginx-image-lab]# docker login

USING WEB-BASED LOGIN

i Info → To sign in with credentials on the command line, use 'docker login -u <username>'


Your one-time device confirmation code is: DXCW-QFWC
Press ENTER to open your browser or submit your device code here: https://login.docker.com/activate

Login Succeeded
```

**7단계. Docker Hub로 이미지 push**

docker tag로 만든 Hub용 태그를 push한다.

```
[root@k8s-master nginx-image-lab]# docker images
IMAGE            			ID       	DISK USAGE   CONTENT SIZE   EXTRA
konan7979/custom-nginx-web:1.31   	5da1d829ff98	222MB         59.8MB
custom-nginx-web:v1   		5da1d829ff98	222MB         59.8MB



[root@k8s-master nginx-image-lab]# docker  push konan7979/custom-nginx-web:1.31
The push refers to repository [docker.io/konan7979/custom-nginx-web]
1f7f5124d211: Pushed
44136fa355b3: Pushed
3c55dc422a81: Pushed
26c307b5e35a: Pushed
c0df8d325117: Pushed
d84ae7b21412: Pushed
b8b80b9bc028: Pushed
5a4222b844e8: Pushed
f5de6e85ac74: Pushed
59b9be530d2f: Pushed
79f88e26950e: Pushed
1.31: digest: sha256:02218a3d52d69c10c071c3d5a426cffb6e56d61a4f72d5373b5319a587e1e358 size: 856



-Docker Hub 웹 페이지에서 이미지 확인



[root@k8s-master ~]# docker  search  konan7979/custom-nginx-web
 NAME                                     DESCRIPTION                                      STARS     OFFICIAL
konan7979/custom-nginx-web                                                                0
```

**8단계. Kubernetes Pod 생성 (워커 노드에서 실행됨)**

방금 push한 이미지를 사용한다. control-plane에는 파드 생성 안 함 (기본 taint로 워커에 스케줄).

```
[root@k8s-master nginx-image-lab]# kubectl  run  custom-nginx-web-pod  --image=konan7979/custom-nginx-web:1.31  --port 80
pod/custom-nginx-web-pod created


[root@k8s-master nginx-image-lab]# kubectl  get  pods
NAME                   	READY   STATUS    RESTARTS   AGE
custom-nginx-web-pod   	1/1        Running      0                38s



[root@k8s-master nginx-image-lab]# kubectl  get  pods  -o  wide
NAME                   	READY   STATUS    RESTARTS   AGE   IP                NODE            NOMINATED NODE   READINESS GATES
custom-nginx-web-pod   	1/1         Running     0                57s     10.244.2.16    k8s-worker2   <none>                    <none>




[root@k8s-master nginx-image-lab]# curl http://10.244.2.16
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>custom nginx</title>
</head>
<body>
  <h1>custom nginx image</h1>
  <p>built on control-plane</p>
</body>
</html>



	# pod 삭제
[root@k8s-master nginx-image-lab]# kubectl  delete  pods  custom-nginx-web-pod
pod "custom-nginx-web-pod" deleted from default namespace
```

특정 namespace(`soldesk`)에 커스텀 이미지로 YAML 기반 Pod를 배포할 수도 있다.

```
[root@k8s-master ~]# vi nginx-custom-pod.yam
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom-pod
  namespace: soldesk
spec:
  containers:
  - name: nginx-custom-container
    image: konan7979/custom-nginx-web:1.31
    ports:
    - containerPort: 80



[root@k8s-master nginx-image-lab]# kubectl  apply  -f  nginx-custom-pod.yam
pod/nginx-custom-pod created



[root@k8s-master nginx-image-lab]# kubectl  get  pods  -n  soldesk
NAME               	READY   STATUS    RESTARTS   AGE
nginx-custom-pod	1/1         Running     0                14s


[root@k8s-master nginx-image-lab]# kubectl  get  pods  -o wide  -n  soldesk
NAME               	READY   STATUS    RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
nginx-custom-pod 	1/1         Running     0                86s     10.244.2.17   k8s-worker2   <none>                    <none>



[root@k8s-master nginx-image-lab]# curl http://10.244.2.17
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>custom nginx</title>
</head>
<body>
  <h1>custom nginx image</h1>
  <p>built on control-plane</p>
</body>
</html>
```

커스텀 이미지로 Deployment를 생성하고, `replicas` 값을 수정해 Pod 개수를 늘리는 것도 확인한다.

```
[root@k8s-master nginx-image-lab]# kubectl  create  deployment  nginx-custom-pod --image=konan7979/custom-nginx-web:1.31 --replicas=3
deployment.apps/nginx-custom-pod created



[root@k8s-master nginx-image-lab]# kubectl  get  pods
NAME                               		READY   STATUS    RESTARTS   AGE
nginx-custom-pod-b5c4b9774-6vbl5   	1/1         Running     0                17s
nginx-custom-pod-b5c4b9774-s8shp   	1/1         Running     0                17s
nginx-custom-pod-b5c4b9774-t258r   	1/1         Running     0                17s




[root@k8s-master nginx-image-lab]# kubectl  get  pods  -o  wide
NAME                               		READY   STATUS    RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
nginx-custom-pod-b5c4b9774-6vbl5  	1/1         Running     0                66s     10.244.2.19   k8s-worker2    <none>                   <none>
nginx-custom-pod-b5c4b9774-s8shp   	1/1         Running     0                66s     10.244.2.18   k8s-worker2    <none>                   <none>
nginx-custom-pod-b5c4b9774-t258r   	1/1         Running     0                66s     10.244.1.20   k8s-worker1    <none>                   <none>




	# deployments 수정 (edit)
[root@k8s-master nginx-image-lab]# kubectl  edit  deployments.apps nginx-custom-pod




[root@k8s-master nginx-image-lab]# kubectl  get  pods
NAME                               		READY   STATUS    RESTARTS   AGE
nginx-custom-pod-b5c4b9774-6vbl5   	1/1         Running     0                5m54s
nginx-custom-pod-b5c4b9774-mj2gk   	1/1         Running     0                10s
nginx-custom-pod-b5c4b9774-nm5c6   	1/1         Running     0                10s
nginx-custom-pod-b5c4b9774-s8shp   	1/1         Running     0                5m54s
nginx-custom-pod-b5c4b9774-t258r   	1/1         Running     0                5m54s




	# deployments 수정 (scale)
[root@k8s-master nginx-image-lab]# kubectl  scale  deployment  nginx-custom-pod  --replicas=3
deployment.apps/nginx-custom-pod scaled



[root@k8s-master nginx-image-lab]# kubectl  get  pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-custom-pod-b5c4b9774-6vbl5   1/1     Running   0          10m
nginx-custom-pod-b5c4b9774-s8shp   1/1     Running   0          10m
nginx-custom-pod-b5c4b9774-t258r   1/1     Running   0          10m




[root@k8s-master nginx-image-lab]# kubectl  delete deployment  nginx-custom-pod
```

YAML로 Deployment 리소스를 직접 정의할 수도 있다.

```
[root@k8s-master nginx-image-lab]# vi custom-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-custom-deploy
spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx-custom

  template:
    metadata:
      labels:
        app: nginx-custom
    spec:
      containers:
        - name: nginx
          image: konan7979/custom-nginx-web:1.31



[root@k8s-master nginx-image-lab]# kubectl get pods
NAME                                   	  READY   STATUS    RESTARTS   AGE
nginx-custom-deploy-6ccb7f6479-82b6k	  1/1         Running     0                8s
nginx-custom-deploy-6ccb7f6479-shjxl	  1/1         Running     0                8s
nginx-custom-deploy-6ccb7f6479-tcgtx 	  1/1         Running     0                8s
```

---


## 2. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- namespace를 매번 `-n`으로 지정하기 번거로우면 `kubectl config set-context`로 context를 만들고 `kubectl config use-context`로 전환해 base namespace를 바꿔두면 이후 명령어가 해당 namespace를 기본 대상으로 삼는다. 다만 이 상태에서 `kubectl delete pods --all`처럼 `-n` 없이 실행하는 명령은 현재 context의 namespace에만 적용되므로 의도치 않은 namespace의 리소스가 삭제되지 않도록 현재 context를 항상 확인해야 한다.
- `kubectl get pods --all-namespaces`(또는 `-A`)로 모든 namespace의 리소스를 한 번에 조회할 수 있으며, 특정 namespace의 리소스만 보려면 `-n <namespace>`를 명시해야 한다. namespace를 지정하지 않으면 현재 context의 base namespace(기본값 `default`)가 대상이 된다.
- `kubectl delete -f <yaml>`로 namespace와 그 안의 리소스를 함께 정의한 YAML을 삭제하면, 파일에 정의된 순서와 무관하게 namespace 및 그에 속한 리소스가 함께 삭제된다.

---

> 📌 **핵심 요약**
> - namespace는 하나의 클러스터를 여러 가상 공간으로 나눠 팀/서비스/환경별로 리소스를 분리하고, 같은 이름의 리소스도 서로 다른 namespace에 만들 수 있게 해준다
> - Pod, Deployment, Service, ConfigMap, Secret 등은 namespace에 속하지만, Node, PersistentVolume, StorageClass, ClusterRole 등은 클러스터 전체 공용 리소스로 namespace에 속하지 않는다
> - namespace는 `kubectl create namespace`(CLI) 또는 `kubectl create namespace --dry-run=client -o yaml`로 생성한 YAML을 `kubectl apply -f`/`kubectl create -f`로 적용하는 방식(YAML)으로 생성할 수 있다
> - `kubectl create`는 이미 존재하면 에러가 나지만 `kubectl apply`는 없으면 생성하고 있으면 그대로 두는 멱등성을 가진다
> - `kubectl config set-context`로 namespace가 포함된 context를 만들고 `kubectl config use-context`로 전환하면 매번 `-n` 옵션을 쓰지 않고도 해당 namespace를 기본 대상으로 사용할 수 있다
> - 관련: 1. 🔧 Kubernetes - 설치 · 2. 📦 Kubernetes - Pod 생성 · 3. 🏗️ Kubernetes - 아키텍처 개요와 핵심 컴포넌트 · 5. 🎚️ Kubernetes - ResourceQuota·LimitRange
