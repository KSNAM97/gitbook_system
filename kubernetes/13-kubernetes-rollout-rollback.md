# 🔁 Kubernetes - Rollout·Rollback 실습

> **Tag:** #Kubernetes #Deployment #RollingUpdate #Rollback #rollout #부트캠프
> **핵심 요약:** kubectl set image로 이미지를 변경해 롤링 업데이트를 수행하고, kubernetes.io/change-cause annotation으로 배포 이력을 남기며, kubectl rollout undo/pause/resume으로 롤백과 배포 제어를 실습한다.

---

## 1. 📖 개요 (Overview)

Deployment에서 실행 중인 컨테이너 이미지를 새로운 버전으로 변경하는 명령은 다음과 같다.

형식: `kubectl set image deployment <deploy_name> <container_name>=<new_version_image> --record`

이 명령을 실행하면 Deployment가 이를 감지하고 자동으로 롤링 업데이트(Rolling Update)를 수행한다.

- `<deploy_name>` : Deployment 이름
- `<container_name>` : Deployment 안에 정의된 컨테이너 이름
- `<new_version_image>` : 변경할 새 컨테이너 이미지 (예: nginx:1.29.0)

예시: `kubectl set image deployment deploy-nginx nginx-container=nginx:1.29.0 --record`

### RollBack 개념

해당 Deployment의 배포 이력을 확인하면 과거에 어떤 이미지와 설정으로 배포되었는지 버전 목록을 보여준다.

- 형식: `kubectl rollout undo deployment <deploy_name>` — 이전 정상 버전으로 복구
- 형식: `kubectl rollout history deployment <deploy_name>` — 배포 이력 확인

가장 최근 이전 버전으로 Deployment를 되돌린다. 롤링 업데이트 도중 장애가 발생했을 때 사용하는 명령어다.

예시: `kubectl rollout history deployment deploy-nginx`

---

## 2. 🛠️ 배포 이력(change-cause) 기록 방법

### deploy-nginx.yaml 파일 수정

```
[root@k8s-master ~]# vi deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-update	# 수정

spec:
  replicas: 3
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      name: nginx-pod
      labels:
        app: web
    spec:
      containers:
      - name: nginx-web	# 수정
        image: nginx:1.31
```

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-nginx.yaml
deployment.apps/deploy-update created


[root@k8s-master ~]# kubectl  get  pods
NAME                            	     READY   STATUS    RESTARTS   AGE
deploy-update-fdd46fc97-rsvt2     1/1         Running     0                34s
deploy-update-fdd46fc97-x6qw9   1/1         Running     0                34s
deploy-update-fdd46fc97-zxpws   1/1         Running     0                34s


[root@k8s-master ~]# kubectl  describe  deployments.apps  deploy-update | grep Image
    Image:         nginx:1.31


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1         <none>
```

아직 버전 기록이 없다.

```
[root@k8s-master ~]# kubectl  delete  deployments.apps  deploy-update
deployment.apps "deploy-update" deleted from default namespace


[root@k8s-master ~]# kubectl  get  pods
No resources found in default namespace.
```

### 방법 1 (곧 deprecated되므로 비 권장) — `--record` 플래그

```
[root@k8s-master ~]# kubectl  create  -f  deploy-nginx.yaml  --record=true
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/deploy-update created


[root@k8s-master ~]# kubectl  get deployments.apps
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
deploy-update   3/3         3                    3                 75


[root@k8s-master ~]# kubectl  get pods
NAME                            		READY   STATUS    RESTARTS   AGE
deploy-update-fdd46fc97-fc9ht   	1/1         Running     0                54s
deploy-update-fdd46fc97-npm9q   	1/1         Running     0                54s
deploy-update-fdd46fc97-rtvl4   	1/1         Running     0                54s


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              kubectl create --filename=deploy-nginx.yaml --record=true


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update  --revision=1
deployment.apps/deploy-update with revision #1
Pod Template:
  Labels:       app=web
        pod-template-hash=fdd46fc97
  Annotations:  kubernetes.io/change-cause: kubectl create --filename=deploy-nginx.yaml --record=true
  Containers:
   nginx-web:
    Image:      	nginx:1.31
    Port:       	<none>
    Host Port:  	<none>
    Environment: 	<none>
    Mounts:     	<none>
  Volumes:      	<none>
  Node-Selectors:	<none>
  Tolerations:  	<none>


[root@k8s-master ~]# kubectl  delete  deployments.apps  deploy-update
deployment.apps "deploy-update" deleted from default namespace
```

### 방법 2 (annotation 방식)

```
[root@k8s-master ~]# kubectl  create  -f  deploy-nginx.yaml
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/deploy-update created


[root@k8s-master ~]# kubectl  get  deployments.apps  deploy-update
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
deploy-update   3/3     3            3           19s


[root@k8s-master ~]# kubectl  get  pods
NAME                            READY   STATUS    RESTARTS   AGE
deploy-update-fdd46fc97-59p2g   1/1     Running   0          24s
deploy-update-fdd46fc97-k67fr   1/1     Running   0          24s
deploy-update-fdd46fc97-rkdx6   1/1     Running   0          24s


[root@k8s-master ~]# kubectl  annotate  deployments.apps  deploy-update \
> kubernetes.io/change-cause="nginx 1.31 이미지 최초 배포" --overwrite
deployment.apps/deploy-update annotated


[root@k8s-master ~]# kubectl  rollout history deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              nginx 1.31 이미지 최초 배포


[root@k8s-master ~]# kubectl  rollout history deployment  deploy-update  --revision=1
deployment.apps/deploy-update with revision #1
Pod Template:
  Labels:       app=web
        pod-template-hash=fdd46fc97
  Annotations:  kubernetes.io/change-cause: nginx 1.31 이미지 최초 배포
  Containers:
   nginx-web:
    Image:      	nginx:1.31
    Port:       	<none>
    Host Port:  	<none>
    Environment:        	<none>
    Mounts:     	<none>
  Volumes:      	<none>
  Node-Selectors:       	<none>
  Tolerations:  	<none>
```

---

## 3. 🖥️ 실습 — Image Version Up

### 터미널 1 (watch)

```
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
```

형식: `kubectl set image deployment <deploy_name> <container_name>=<new_version_image>`

### 터미널 2 (이미지 변경)

```
# 위의 방법 1
[root@k8s-master ~]# kubectl  set image  deployments  deploy-update  nginx-web=nginx:1.31.1  --record


# 위의 방법 2 (실습 진행)
[root@k8s-master ~]# kubectl  set image  deployments  deploy-update  nginx-web=nginx:1.31.1
```

터미널 1(watch)에서 관찰되는 롤링 업데이트 진행 과정:

```
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                            		READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
deploy-update-fdd46fc97-84882   	1/1     Running   0          4s    10.244.1.11   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-lssnv   	1/1     Running   0          4s    10.244.1.12   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-thtfw   	1/1     Running   0          4s    10.244.2.8    k8s-worker2   <none>           <none>
deploy-update-64f9b77664-cznwv   	0/1     Pending   0          0s    <none>        <none>        <none>           <none>
deploy-update-64f9b77664-cznwv   	0/1     Pending   0          0s    <none>        k8s-worker2   <none>           <none>
deploy-update-64f9b77664-cznwv   	0/1     ContainerCreating   0          0s    <none>        k8s-worker2   <none>           <none>
deploy-update-64f9b77664-cznwv   	1/1     Running             0          11s   10.244.2.9    k8s-worker2   <none>           <none>
deploy-update-fdd46fc97-thtfw    	1/1     Terminating         0          77s   10.244.2.8    k8s-worker2   <none>           <none>
deploy-update-64f9b77664-tcr24   	0/1     Pending             0          0s    <none>        <none>        <none>           <none>
deploy-update-fdd46fc97-thtfw    	1/1     Terminating         0          77s   10.244.2.8    k8s-worker2   <none>           <none>
deploy-update-64f9b77664-tcr24   	0/1     Pending             0          0s    <none>        k8s-worker1   <none>           <none>
deploy-update-64f9b77664-tcr24   	0/1     ContainerCreating   0          0s    <none>        k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-thtfw    	0/1     Completed           0          77s   10.244.2.8    k8s-worker2   <none>           <none>
deploy-update-fdd46fc97-thtfw    	0/1     Completed           0          78s   10.244.2.8    k8s-worker2   <none>           <none>
deploy-update-fdd46fc97-thtfw    	0/1     Completed           0          78s   10.244.2.8    k8s-worker2   <none>           <none>
deploy-update-64f9b77664-tcr24   	1/1     Running             0          10s   10.244.1.13   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-84882    	1/1     Terminating         0          87s   10.244.1.11   k8s-worker1   <none>           <none>
deploy-update-64f9b77664-sqhmb   	0/1     Pending             0          0s    <none>        <none>        <none>           <none>
deploy-update-fdd46fc97-84882    	1/1     Terminating         0          87s   10.244.1.11   k8s-worker1   <none>           <none>
deploy-update-64f9b77664-sqhmb   	0/1     Pending             0          0s    <none>        k8s-worker2   <none>           <none>
deploy-update-64f9b77664-sqhmb   	0/1     ContainerCreating   0          0s    <none>        k8s-worker2   <none>           <none>
deploy-update-fdd46fc97-84882    	0/1     Completed           0          87s   10.244.1.11   k8s-worker1   <none>           <none>
deploy-update-64f9b77664-sqhmb  	1/1     Running             0          1s    10.244.2.10   k8s-worker2   <none>           <none>
deploy-update-fdd46fc97-lssnv    	1/1     Terminating         0          88s   10.244.1.12   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-lssnv    	1/1     Terminating         0          88s   10.244.1.12   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-lssnv    	0/1     Completed           0          88s   10.244.1.12   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-lssnv    	0/1     Completed           0          88s   10.244.1.12   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-lssnv    	0/1     Completed           0          88s   10.244.1.12   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-84882    	0/1     Completed           0          88s   10.244.1.11   k8s-worker1   <none>           <none>
deploy-update-fdd46fc97-84882    	0/1     Completed           0          88s   10.244.1.11   k8s-worker1   <none>           <none>
```

```
[root@k8s-master ~]# kubectl  annotate  deployments.apps  deploy-update \
> kubernetes.io/change-cause="2026-0819 10:54  nginx 1.31.1 이미저 버전업" --overwrite
deployment.apps/deploy-update annotated


[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              nginx 1.31 이미지 최초 배포
2              2026-0819 10:54  nginx 1.31.1 이미저 버전업


[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update  --revision=2
deployment.apps/deploy-update with revision #2
Pod Template:
  Labels:       app=web
        pod-template-hash=64f9b77664
  Annotations:  kubernetes.io/change-cause: 2026-0819 10:54  nginx 1.31.1 이미저 버전업
  Containers:
   nginx-web:
    Image:      	nginx:1.31.1
    Port:       	<none>
    Host Port:  	<none>
    Environment:        	<none>
    Mounts:     	<none>
  Volumes:      	<none>
  Node-Selectors:       	<none>
  Tolerations:  	<none>
```

```
[root@k8s-master ~]# kubectl  describe  deployments.apps
Name:                   	deploy-update
Namespace:            	default
CreationTimestamp:      	Wed, 19 Aug 2026 10:53:52 +0900
Labels:                 		<none>
Annotations:            	deployment.kubernetes.io/revision: 2
                        		kubernetes.io/change-cause: 2026-0819 10:54  nginx 1.31.1 이미저 버전업
Selector:               	app=web
Replicas:               	3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           	RollingUpdate
MinReadySeconds:        	0
RollingUpdateStrategy:  	25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=web
  Containers:
   nginx-web:
    Image:         	nginx:1.31.1
    Port:          	<none>
    Host Port:     	<none>
    Environment:   	<none>
    Mounts:        	<none>
  Volumes:         	<none>
  Node-Selectors:  	<none>
  Tolerations:     	<none>
Conditions:
  Type           	Status  	Reason
  ----          	------	------
  Available      	True    	MinimumReplicasAvailable
  Progressing    	True    	NewReplicaSetAvailable
OldReplicaSets:  	deploy-update-fdd46fc97 (0/0 replicas created)
NewReplicaSet:	deploy-update-64f9b77664 (3/3 replicas created)
Events:
  Type    Reason                  Age    From                        Message
  ----    ------                  ----    ----                         -------
  Normal  ScalingReplicaSet  4m45s  deployment-controller  Scaled up replica set deploy-update-fdd46fc97 from 0 to 3
  Normal  ScalingReplicaSet  4m24s  deployment-controller  Scaled up replica set deploy-update-64f9b77664 from 0 to 1
  Normal  ScalingReplicaSet  4m23s  deployment-controller  Scaled down replica set deploy-update-fdd46fc97 from 3 to 2
  Normal  ScalingReplicaSet  4m23s  deployment-controller  Scaled up replica set deploy-update-64f9b77664 from 1 to 2
  Normal  ScalingReplicaSet  4m22s  deployment-controller  Scaled down replica set deploy-update-fdd46fc97 from 2 to 1
  Normal  ScalingReplicaSet  4m22s  deployment-controller  Scaled up replica set deploy-update-64f9b77664 from 2 to 3
  Normal  ScalingReplicaSet  4m21s  deployment-controller  Scaled down replica set deploy-update-fdd46fc97 from 1 to 0
```

```
[root@k8s-master ~]# kubectl  get pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-64f9b77664-9kp96   	1/1     Running   0          10m
deploy-update-64f9b77664-c4tgn   	1/1     Running   0          10m
deploy-update-64f9b77664-trnd9   	1/1     Running   0          10m


[root@k8s-master ~]# kubectl  describe  pods  deploy-update-64f9b77664-9kp96


[root@k8s-master ~]# kubectl  delete  deployments.apps  deploy-update
deployment.apps "deploy-update" deleted from default namespace
```

---

## 4. ↩️ Rollback 실습

- 형식: `kubectl rollout undo deployment <deploy_name>` — 이전 정상 버전으로 복구
- 형식: `kubectl rollout history deployment <deploy_name>` — 배포 이력 확인

### 배포 이력 확인

```
[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1         nginx 1.31 이미지 최초 배포
2         2026-0819 10:54  nginx 1.31.1 이미저 버전업


[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update  --revision=1
[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update  --revision=2
```

### Rollback

```
[root@k8s-master ~]# kubectl rollout undo deployment  deploy-update
deployment.apps/deploy-update rolled back


[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
2             2026-0819 10:54  nginx 1.31.1 이미지 버전업
3             nginx 1.31 이미지 최초 배포
```

```
1 = nginx:1.31
2 = nginx:1.31.1
3 = nginx:1.31 (Revision 1의 설정으로 Rollback하면서 동일한 설정이 새로운 Revision 3으로 기록되고,
                   Revision 1은 history 목록에서 사라진다.)
```

```
[root@k8s-master ~]# kubectl  describe  deployments.apps  deploy-update | grep Image
    Image:         nginx:1.31


[root@k8s-master ~]# kubectl annotate deployment deploy-update \
> kubernetes.io/change-cause="Revision 2에서 nginx 1.31.1 버전을 nginx 1.31로 Rollback" \
> --overwrite


[root@k8s-master ~]# kubectl  rollout  history deployment  deploy-update                               
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
2              2026-0819 10:54  nginx 1.31.1 이미저 버전업
3              Revision 2에서 nginx 1.31.1 버전을 nginx 1.31로 Rollback
```

```
[root@k8s-master ~]# kubectl delete deployments.apps deploy-update
deployment.apps "deploy-update" deleted
```

---

## 5. 🧪 실습 — Deployment 롤링업데이트 + 히스토리(change-cause) 기록 + 롤백

```
[root@k8s-master ~]# vi deploy-update-step2.yaml
apiVersion: apps/v1    	# Deployment에서 사용하는 API 버전
kind: Deployment         	# 생성할 리소스 종류
metadata:
  name: deploy-update 	# Deployment 이름

spec:
  replicas: 3                 	# 유지할 Pod 개수
  revisionHistoryLimit: 5   	# 이전 ReplicaSet(Revision) 이력을 최대 5개까지 보관
  strategy:
    type: RollingUpdate     	# 배포 전략을 RollingUpdate 방식으로 설정
    rollingUpdate:
      maxSurge: 1        	# 업데이트 중 원하는 Pod 수보다 최대 1개까지 추가 생성 가능
      maxUnavailable: 1      	# 업데이트 중 최대 1개의 Pod까지 사용 불가 상태 허용
  selector:
    matchLabels:
      app: web              	# app=web 라벨을 가진 Pod를 관리 대상으로 선택

  template:
    metadata:
      labels:
        app: web       		# Deployment selector와 반드시 일치해야 하는 라벨
        track: stable
    spec:
      containers:
      - name: nginx-container 		# 컨테이너 이름
        image: nginx:1.31     		# 사용할 nginx 이미지 버전
        ports:
        - containerPort: 80    		# 컨테이너가 사용하는 포트 정보
```

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-update-step2.yaml  --dry-run=client
deployment.apps/deploy-update created (dry run)


[root@k8s-master ~]# kubectl  apply  -f  deploy-update-step2.yaml
deployment.apps/deploy-update created


[root@k8s-master ~]# kubectl  get  deployments.apps  deploy-update
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
deploy-update   3/3         3                    3                  6s


[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-6747747c6f-mf2mh	1/1         Running     0                17s
deploy-update-6747747c6f-q5pvz   	1/1         Running     0                17s
deploy-update-6747747c6f-zpl4k   	1/1         Running     0                17s        
```

### 초기 배포 revision에 대한 변경 사유 기록

```
[root@k8s-master ~]# kubectl  annotate deployment deploy-update \
kubernetes.io/change-cause="revision 1 : initial deploy nginx 1.31" --overwrite
deployment.apps/deploy-update annotated
```

### 롤링 업데이트 버전 확인

```
[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              revision 1 : initial deploy nginx 1.31


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update  --revision=1
deployment.apps/deploy-update with revision #1
Pod Template:
  Labels: 		app=web
        pod-template-hash=6747747c6f
        track=stable
  Annotations:  kubernetes.io/change-cause: revision 1 : initial deploy nginx 1.31
  Containers:
   nginx-container:
    Image:      	nginx:1.31
    Port:       	80/TCP
    Host Port:	0/TCP
    Environment:        	<none>
    Mounts:     	<none>
  Volumes:      	<none>
  Node-Selectors:       	<none>
  Tolerations:  	<none>


[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-6747747c6f-mf2mh	1/1         Running     0                17s
deploy-update-6747747c6f-q5pvz   	1/1         Running     0                17s
deploy-update-6747747c6f-zpl4k   	1/1         Running     0                17s  
```

### 이미지 버전 업 (Rolling Update)

```
[root@k8s-master ~]# kubectl  set  image  deployments  deploy-update  nginx-container=nginx:1.13.1
deployment.apps/deploy-update image updated
```

### 변경 이력(change-cause) 기록

```
[root@k8s-master ~]# kubectl  annotate  deployments  deploy-update  \
> kubernetes.io/change-cause="revision 2 : nginx:1.31  -->  nginx:1.13.1 version update" \
> --overwrite
deployment.apps/deploy-update annotated
```

### Deployment의 롤링 업데이트 진행 상태를 확인

```
[root@k8s-master ~]# kubectl  rollout  status  deployment  deploy-update
deployment "deploy-update" successfully rolled out
```

### 롤아웃 상태 및 히스토리 확인

```
[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              revision 1 : initial deploy nginx 1.31
2              revision 2 : nginx:1.31  -->  nginx:1.13.1 version update


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update  --revision=2
deployment.apps/deploy-update with revision #2
Pod Template:
  Labels:       	app=web
        pod-template-hash=59c675765b
        track=stable
  Annotations:  kubernetes.io/change-cause: revision 2 : nginx:1.31  -->  nginx:1.13.1 version update
  Containers:
   nginx-container:
    Image:      	nginx:1.13.1
    Port:       	80/TCP
    Host Port:  	0/TCP
    Environment:        	<none>
    Mounts:     	<none>
  Volumes:      	<none>
  Node-Selectors:       	<none>
  Tolerations:  	<none>


[root@k8s-master ~]# kubectl  describe  deployments.apps  deploy-update  | grep Image
    Image:         nginx:1.13.1
```

### 롤백

```
# 이전 revision의로 롤백
[root@k8s-master ~]# kubectl  rollout  undo  deployment  deploy-update
deployment.apps/deploy-update rolled back


# 특정 revision의로 롤백
[root@k8s-master ~]# kubectl  rollout  undo  deployment  deploy-update  --to-revision=1
deployment.apps/deploy-update rolled back
```

### 히스토리 확인

```
[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
2         revision 2 : nginx:1.31  -->  nginx:1.13.1 version update--overwrite
3         revision 1 : initial deploy nginx 1.31
```

### 히스토리 수정

```
[root@k8s-master ~]# kubectl annotate deployment deploy-update \
kubernetes.io/change-cause="revision 3 : rollback to nginx 1.31"  --overwrite
deployment.apps/deploy-update annotated
```

### 히스토리 확인

```
[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update               
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
2         revision 2 : nginx:1.31  -->  nginx:1.13.1 version update--overwrite
3         revision 3 : rollback to nginx 1.31
```

### 이미지 변경 확인

```
[root@k8s-master ~]# kubectl  describe  deployments.apps  deploy-update  | grep Image
    Image:         nginx:1.31


[root@k8s-master ~]# kubectl  get  pods
NAME                             READY   STATUS    RESTARTS   AGE
deploy-update-6747747c6f-b9t8x   1/1     Running   0          4m49s
deploy-update-6747747c6f-c9ld5   1/1     Running   0          4m49s
deploy-update-6747747c6f-z4klk   1/1     Running   0          4m49s
```

### 다시 최신 버전으로 롤링 업데이트

```
[root@k8s-master ~]# kubectl  rollout  undo  deployment  deploy-update --to-revision=2
deployment.apps/deploy-update rolled back
```

Revision 1 (nginx:1.31)

```
[root@k8s-master ~]# kubectl  get  pods
NAME                             READY   STATUS    RESTARTS   AGE
deploy-update-6747747c6f-b9t8x   1/1     Running   0          4m49s
deploy-update-6747747c6f-c9ld5   1/1     Running   0          4m49s
deploy-update-6747747c6f-z4klk   1/1     Running   0          4m49s
```

Revision 2 (nginx:1.31.1)

```
[root@k8s-master ~]# kubectl  get  pods
NAME                             READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-5snbr   1/1     Running   0          11s
deploy-update-59c675765b-l72sc   1/1     Running   0          11s
deploy-update-59c675765b-p5xlm   1/1     Running   0          11s


[root@k8s-master ~]# kubectl  describe  deployments.apps  deploy-update  | grep Image
    Image:         nginx:1.13.1
```

### 히스토리 수정

```
[root@k8s-master ~]# kubectl annotate deployment deploy-update \
kubernetes.io/change-cause="revision 4 : rollback to revision 2 nginx 1.31.1" \
--overwrite
```

### 히스토리 확인

```
[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
3              revision 3 : rollback to nginx 1.31
4              revision 4 : rollback to revision 2 nginx 1.31.1
```

---

## 6. ⚠️ revision이 증가하지 않는 변경 (annotation-only 변경)

모든 Deployment 변경이 Revision을 증가시키는 것은 아니다. Deployment 자체의 `metadata.annotation`만 변경하면 Pod Template(`spec.template`)은 변경되지 않는다. Pod Template이 변경되지 않으면 새로운 ReplicaSet이 생성되지 않고 Revision도 증가하지 않는다. 이미지, 환경변수, Pod Template의 label/annotation 등이 변경되면 새로운 ReplicaSet이 생성되고 Revision이 증가한다.

### 아무 annotation 설정

```
[root@k8s-master ~]# kubectl  annotate  deployments.apps  deploy-update \
> test-annotation=hello  --overwrite
deployment.apps/deploy-update annotated


[root@k8s-master ~]# kubectl  get  deployments.apps  deploy-update  -o  yaml | grep ann
  annotations:
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"deploy-update","namespace":"default"},"spec":{"replicas":3,"revisionHistoryLimit":5,"selector":{"matchLabels":{"app":"web"}},"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":1},"type":"RollingUpdate"},"template":{"metadata":{"labels":{"app":"web","track":"stable"}},"spec":{"containers":[{"image":"nginx:1.31","name":"nginx-container","ports":[{"containerPort":80}]}]}}}}
    test-annotation: hello
```

### 아무 annotation을 사용하면 Revision이 변경되지 않는다.

```
[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
3              revision 3 : rollback to nginx 1.31
4              revision 4 : rollback to revision 2 nginx 1.31.1
```

### kubernetes.io/change-cause 설정

```
[root@k8s-master ~]# kubectl  annotate  deployments  deploy-update \
kubernetes.io/change-cause="revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정" --overwrite
deployment.apps/deploy-update annotated


[root@k8s-master ~]#  kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
3         revision 3 : rollback to nginx 1.31
4         revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정
```

이미지나 환경변수 변경 없이 annotation을 작성하면 새로운 버전이 저장되는 것이 아니라 덮어쓰기된다. 즉 새로운 ReplicaSet이 생성되지 않으면 덮어쓰기된다. (새 ReplicaSet 생성 → revision 증가)

### change-cause를 남겼을 때 vs 안 남겼을 때

Deployment의 Revision 자체는 Pod Template이 변경되면 자동으로 증가한다. 하지만 CHANGE-CAUSE는 자동으로 작성되는 값이 아니다. `kubernetes.io/change-cause` annotation을 별도로 작성하지 않으면 rollout history의 CHANGE-CAUSE 값은 `<none>`으로 표시될 수 있다. 따라서 운영 환경에서는 어떤 변경을 했는지 알 수 있도록 change-cause를 남겨두는 것이 좋다.

### 롤링 업데이트 (change-cause 없이 업데이트)

```
[root@k8s-master ~]# kubectl  set  image  deployments  deploy-update  nginx-container=nginx:1.31.3
deployment.apps/deploy-update image updated


[root@k8s-master ~]# kubectl  get  pods
NAME                             READY   STATUS    RESTARTS   AGE
deploy-update-7c85497f86-8fq57   1/1     Running   0          4s
deploy-update-7c85497f86-fwsk9   1/1     Running   0          3s
deploy-update-7c85497f86-vdv7d   1/1     Running   0          4s


[root@k8s-master ~]#  kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
3         revision 3 : rollback to nginx 1.31
4         revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정
5         revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정
```

### 롤링 업데이트 (change-cause 없이 업데이트, 2회차)

```
[root@k8s-master ~]# kubectl  set  image  deployments  deploy-update  nginx-container=nginx:1.30
deployment.apps/deploy-update image updated
```

롤링업데이트 후 CHANGE-CAUSE를 설정하지 않으면 관리가 어려워질 수 있다. 롤링 업데이트인지 롤백인지 확인할 수 없다.

```
[root@k8s-master ~]#  kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
3              revision 3 : rollback to nginx 1.31
4              revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정
5              revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정
6              revision 4 nginx:1.31.1  -->  nginx:1.31.2로 수정
```

### 변경 이력(change-cause) 기록

```
[root@k8s-master ~]# kubectl annotate deployment deploy-update \
kubernetes.io/change-cause="revision 6: nginx:1.31.3  -->  nginx:1.30로 수정" --overwrite


[root@k8s-master ~]#  kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
3              revision 3 : rollback to nginx 1.31
4              revision 4 : nginx:1.31.1  -->  nginx:1.31.2로 수정
5              revision 4 : nginx:1.31.1  -->  nginx:1.31.2로 수정
6              revision 6 : nginx 1.31.3  -->  nginx:1.30로 수정
```

---

## 7. 🚨 실습 — rollout status가 멈추는 상황 만들기

잘못된 이미지로 Rolling Update를 수행하여 Deployment가 정상적으로 완료되지 않는 상태를 확인한다. `rollout status` 명령어가 무엇을 기다리는지 확인한다. 새 ReplicaSet은 생성되지만, 새 Pod가 정상적으로 Ready 상태가 되지 않으면 Rolling Update는 완료되지 않는다.

### 1) 잘못된 이미지로 업데이트

```
[root@k8s-master ~]# kubectl  set  image  deployments  deploy-update  nginx-container=nginx:not-exist
```

### 2) 롤링 업데이트 상태 확인

```
[root@k8s-master ~]# kubectl  rollout  status  deployment  deploy-update
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...


[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY      STATUS         	RESTARTS   AGE
deploy-update-5b7ddfdd86-cnf26   	1/1            Running          	0                13m
deploy-update-5b7ddfdd86-mrbzj   	1/1            Running        	0                13m
deploy-update-c95df6845-7cm6z    	0/1            ErrImagePull   	0                47s		# image Pull 실패
deploy-update-c95df6845-rxqft    	0/1            ErrImagePull	0                47s		# image Pull 실패
```

잘못된 이미지로 Rolling Update 할 때 2개의 Pod가 ErrImagePull 상태가 될 수 있는 이유:

- 기존 Pod가 3개 실행 중인 상태에서 Rolling Update를 시작한다.
- 첫 번째 새 Pod가 생성되고 잘못된 이미지를 Pull하려고 시도한다.
- 이미지가 존재하지 않으면 첫 번째 새 Pod는 ErrImagePull 또는 ImagePullBackOff 상태가 된다.
- 이 과정에서 기존 Pod 1개가 Terminating 상태로 들어갈 수 있다.
- 그러면 Deployment Controller는 원하는 새 버전 Pod 개수를 맞추기 위해 새 Pod를 하나 더 생성할 수 있다.
- 두 번째 새 Pod 역시 같은 잘못된 이미지를 사용하므로 ErrImagePull 상태가 된다.

### 3) 롤백으로 복구

```
# 이전 Revision으로 버전으로 롤백
[root@k8s-master ~]# kubectl  rollout  undo  deployment  deploy-update
deployment.apps/deploy-update rolled back


# 특정 Revision으로 버전으로 롤백
[root@k8s-master ~]# kubectl  rollout  undo  deployment  deploy-update  --to-revision=4
deployment.apps/deploy-update rolled back
```

### 4) rollout 상태 확인

```
[root@k8s-master ~]# kubectl  rollout  status  deployment  deploy-update
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment spec update to be observed...
Waiting for deployment spec update to be observed...
Waiting for deployment "deploy-update" rollout to finish: 0 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 0 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-update" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-update" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-update" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-update" rollout to finish: 1 old replicas are pending termination...
deployment "deploy-update" successfully rolled out


[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl   	1/1     Running   0          14s
deploy-update-59c675765b-mmnrv   	1/1     Running   0          15s
deploy-update-59c675765b-wkt9k   	1/1     Running   0          15s


[root@k8s-master ~]# kubectl  describe  deployments  deploy-update | grep Image
    Image:         nginx:1.13.1


[root@k8s-master ~]# cat  deploy-update-step2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-update
spec:
  replicas: 3
  revisionHistoryLimit: 5	# ReplicaSet 5개까지 기록
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
~~~~~~ 중간 생략 ~~~~~~


[root@k8s-master ~]# kubectl  get  rs
NAME                       	DESIRED   CURRENT   READY   AGE
deploy-update-59c675765b   	3              3               3          155m
deploy-update-5b7ddfdd86   	0              0               0          96m
deploy-update-6747747c6f   	0              0               0          163m
deploy-update-7c85497f86   	0              0               0          101m
deploy-update-c95df6845    	0              0               0          83m


[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl  	1/1         Running     0                80m
deploy-update-59c675765b-mmnrv   	1/1         Running     0                80m
deploy-update-59c675765b-wkt9k   	1/1         Running     0                80m
```

오래된 ReplicaSet에 파드가 생성되어있다. 잘못된 이미지명으로 롤링업데이트 중 롤백했기 때문에 오래된 ReplicaSet에 파드가 생성되어있다.

---

## 8. ⏸️ 실습 — rollout pause / resume

`rollout pause`와 `rollout resume`은 Deployment의 Rolling Update를 일시 중지했다가 다시 진행할 때 사용하는 명령어다. 일반적인 Rolling Update는 Deployment의 이미지나 환경변수 등 Pod Template이 변경되면 즉시 새로운 ReplicaSet을 생성하고 새로운 Pod를 배포한다.

Deployment 설정 변경(Pod Template) 흐름:

```
새 ReplicaSet 생성  -->  새 Pod 생성  -->  새 Pod Ready 확인  -->  기존 Pod 제거  -->  Rolling Update 완료
```

### rollout pause

형식: `kubectl rollout pause deployment <deployment_name>`

Deployment의 Rolling Update를 일시 중지한다. 중요한 점은 pause가 현재 실행 중인 Pod를 정지시키는 명령어가 아니라는 것이다.

- 기존 Pod 실행 O
- 서비스 계속 O
- 새 ReplicaSet 생성 X
- 새 Pod 생성 X
- 기존 Pod 교체 X

pause 상태에서도 Deployment의 설정은 변경할 수 있다. 이미지 설정 자체는 변경되지만, pause 상태이므로 새로운 ReplicaSet을 생성하거나 Pod를 교체하지 않는다.

```
pause  -->  이미지 변경  -->  Deployment 설정 변경 O  -->  실제 Rolling Update X
```

### rollout resume

형식: `kubectl rollout resume deployment <deployment_name>`

pause 상태를 해제하고 중지되어 있던 Rolling Update를 다시 진행한다.

```
resume  -->  pause 중 변경했던 Pod Template 확인  -->  새 ReplicaSet 생성  -->  새 Pod 생성
   -->  새 Pod Ready 확인  -->  기존 Pod 제거  -->  Rolling Update 완료
```

### 일반 Rolling Update와 pause/resume의 차이

- **일반 Rolling Update**: 이미지 변경 → 즉시 Rolling Update 시작 → 새 ReplicaSet 생성 → 새 Pod 생성 → 기존 Pod 교체
- **pause / resume 사용**: pause → 이미지, 환경변수 등 변경 → 실제 Pod 교체는 아직 하지 않음 → resume → 변경된 Rolling Update 시작

### pause / resume은 언제 사용하는가?

여러 설정을 한꺼번에 변경한 뒤 한 번의 Rolling Update로 적용하고 싶을 때 사용할 수 있다.

예를 들어 다음 세 가지를 변경한다고 가정한다.

1. nginx 이미지 변경
2. 환경변수 변경
3. CPU/Memory 설정 변경

pause 없이 각각 수정하면 변경할 때마다 rollout이 발생할 수 있다.

```
pause   -->  이미지, 환경변수, 리소스 설정 변경   -->  설정 확인   -->  resume   -->  한 번의 Rolling Update
```

운영 중 변경 사항을 먼저 작성한 뒤 검토가 끝난 후 실제 배포하고 싶을 때도 유용하다.

### 5-1) 현재 Deployment와 Pod 확인

```
[root@k8s-master ~]# kubectl  get  deployments  deploy-update
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
deploy-update   3/3         3                    3                 175m


[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl  	1/1         Running     0                80m
deploy-update-59c675765b-mmnrv   	1/1         Running     0                80m
deploy-update-59c675765b-wkt9k   	1/1         Running     0                80m
```

### 5-2) Deployment rollout 일시 중지

```
[root@k8s-master ~]# kubectl  rollout  pause  deployment  deploy-update
deployment.apps/deploy-update paused
```

### 5-3) pause 상태에서 이미지 변경

```
[root@k8s-master ~]# kubectl  describe  deployments  deploy-update | grep Image
    Image:         nginx:1.13.1


[root@k8s-master ~]# kubectl  get  pods  --watch
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl   	1/1         Running     0                91m
deploy-update-59c675765b-mmnrv   	1/1         Running     0                92m
deploy-update-59c675765b-wkt9k   	1/1         Running     0                92m


[root@k8s-master ~]# kubectl  set  image  deployments  deploy-update  nginx-container=nginx:1.31.3
deployment.apps/deploy-update image updated


# 롤링업데이트되지 않는다.
[root@k8s-master ~]# kubectl  get  pods  --watch
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl   	1/1         Running     0                91m
deploy-update-59c675765b-mmnrv   	1/1         Running     0                92m
deploy-update-59c675765b-wkt9k   	1/1         Running     0                92m
```

### 5-4) Deployment에 설정된 이미지 확인

```
[root@k8s-master ~]# kubectl  describe  deployments  deploy-update  | grep Image
    Image:         nginx:1.31.3
```

현재 이미지가 nginx:1.31.1에서 nginx:1.31.3으로 변경되어있다. 하지만 롤링 업데이트는 발생하지 않는다.

### 5-5) 현재 Pod 및 ReplicaSet 확인

```
[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl   	1/1         Running     0                91m
deploy-update-59c675765b-mmnrv   	1/1         Running     0                92m
deploy-update-59c675765b-wkt9k   	1/1         Running     0                92m


[root@k8s-master ~]# kubectl  describe  pods  deploy-update-59c675765b-fn7bl | grep  Image
Image
    Image:     	nginx:1.13.1
    Image ID:	docker.io/library/nginx@sha256:72c7191585e9b79cde433c89955547685db00f3a8595a750339549f6acef7702


[root@k8s-master ~]# kubectl  get  rs
NAME                       	DESIRED   CURRENT   READY   AGE
deploy-update-59c675765b   	3         3         3       174m
deploy-update-5b7ddfdd86   	0         0         0       114m
deploy-update-6747747c6f   	0         0         0       3h2m
deploy-update-7c85497f86   	0         0         0       120m
deploy-update-c95df6845    	0         0         0       102m
```

### 5-6) rollout 재개

```
[root@k8s-master ~]# kubectl  rollout  resume  deployment  deploy-update
deployment.apps/deploy-update resumed


[root@k8s-master ~]# kubectl  get  pods  --watch
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-59c675765b-fn7bl   	1/1     Running   0          97m
deploy-update-59c675765b-mmnrv   	1/1     Running   0          97m
deploy-update-59c675765b-wkt9k   	1/1     Running   0          97m
deploy-update-59c675765b-mmnrv   	1/1     Terminating   0          98m
deploy-update-7c85497f86-svlqn   	0/1     Pending       0          0s
deploy-update-7c85497f86-svlqn   	0/1     Pending       0          0s
deploy-update-59c675765b-mmnrv   	1/1     Terminating   0          98m
deploy-update-7c85497f86-svlqn   	/1     ContainerCreating   0          0s
deploy-update-7c85497f86-hhqvh   	0/1     Pending             0          0s
deploy-update-7c85497f86-hhqvh   	0/1     Pending             0          0s
deploy-update-7c85497f86-hhqvh   	0/1     ContainerCreating   0          0s
deploy-update-59c675765b-mmnrv   	0/1     Completed           0          98m
deploy-update-7c85497f86-svlqn   	1/1     Running             0          0s
deploy-update-59c675765b-wkt9k   	1/1     Terminating         0          98m
deploy-update-7c85497f86-sncvt   	0/1     Pending             0          0s
deploy-update-59c675765b-wkt9k   	1/1     Terminating         0          98m
deploy-update-7c85497f86-sncvt   	0/1     Pending             0          0s
deploy-update-7c85497f86-sncvt   	0/1     ContainerCreating   0          0s
deploy-update-59c675765b-wkt9k   	0/1     Completed           0          98m
deploy-update-7c85497f86-hhqvh   	1/1     Running             0          0s
deploy-update-59c675765b-mmnrv   	0/1     Completed           0          98m
deploy-update-59c675765b-mmnrv   	0/1     Completed           0          98m
deploy-update-59c675765b-fn7bl   	1/1     Terminating         0          98m
deploy-update-59c675765b-fn7bl   	1/1     Terminating         0          98m
deploy-update-59c675765b-fn7bl  	0/1     Completed           0          98m
deploy-update-59c675765b-wkt9k   	0/1     Completed           0          98m
deploy-update-59c675765b-wkt9k   	0/1     Completed           0          98m
deploy-update-7c85497f86-sncvt   	1/1     Running             0          1s
deploy-update-59c675765b-fn7bl   	0/1     Completed           0          98m
deploy-update-59c675765b-fn7bl   	0/1     Completed           0          98m
```

### 4-7) ReplicaSet 다시 확인

```
[root@k8s-master ~]# kubectl  get  rs
NAME                       	DESIRED   CURRENT   READY   AGE
deploy-update-59c675765b   	0              0               0           179m
deploy-update-5b7ddfdd86   	0              0               0           120m
deploy-update-6747747c6f   	0              0               0           3h7m
deploy-update-7c85497f86   	3              3               3           125m
deploy-update-c95df6845    	0              0               0           108m
```

예전 어느 시점:

- nginx:1.31.3 설정으로 ReplicaSet 생성
- deploy-update-7c85497f86 생성
- AGE는 그때부터 계속 누적

그 이후 다른 이미지로 업데이트:

- 7c85497f86은 replicas 0으로 내려감
- ReplicaSet 자체는 revisionHistoryLimit 때문에 남아 있음

이번에 pause 상태에서 다시 nginx:1.31.3 설정:

- resume
- Deployment Controller가 확인
- "이 Pod Template과 동일한 기존 ReplicaSet이 있네?"
- 새 ReplicaSet 생성 안 함
- 기존 deploy-update-7c85497f86을 다시 scale up
- replicas 3

```
# deployment 삭제
[root@k8s-master ~]# kubectl  delete  deployments.apps  deploy-update
deployment.apps "deploy-update" deleted from default namespace
```

---

## 9. 🧮 실습 — maxSurge/maxUnavailable 값을 2로 늘려 8개 Pod 운영

**EX)** Deployment의 Pod를 8개로 실행하고, Rolling Update 시 maxSurge: 2, maxUnavailable: 2를 설정하여 새 버전 Pod가 최대 2개 추가되고 기존 Pod도 최대 2개까지 사용할 수 없는 상태를 확인

```
[root@k8s-master ~]# vi deploy-update-step2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-update
spec:
  replicas: 6		# 3에서 6로 수정
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2		# 2로 수정
      maxUnavailable: 2	# 2로 수정
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        track: stable
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.31
        ports:
        - containerPort: 80
```

maxSurge는 업데이트 중 원래 replicas 수보다 몇 개까지 Pod를 더 만들 수 있는가를 의미한다.

예를 들어:

- replicas: 3
- maxSurge: 1
- 기본 Pod 수 = 3개
- 추가 생성 가능 = 1개
- 최대 Pod 수 = 4개
- 즉 새 버전 Pod를 먼저 하나 더 만들어 놓을 수 있다.

maxUnavailable은 업데이트 중 원하는 replicas 수에서 몇 개까지 서비스 불가 상태를 허용할 것인가를 의미한다.

예를 들어:

- replicas: 3
- maxUnavailable: 1
- 원하는 Pod = 3개
- Unavailable 허용 = 1개
- 최소 Available Pod = 2개
- 즉 업데이트 도중 최소 2개는 정상 서비스 가능한 상태로 유지해야 한다.

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-update-step2.yaml  --dry-run=client
deployment.apps/deploy-update configured (dry run)


[root@k8s-master ~]# kubectl  apply  -f  deploy-update-step2.yaml
deployment.apps/deploy-update created


[root@k8s-master ~]# kubectl  get  pods  --watch
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-6747747c6f-f9mrz   	1/1     Running   0          9s
deploy-update-6747747c6f-k9rcc   	1/1     Running   0          9s
deploy-update-6747747c6f-ksm6s   	1/1     Running   0          9s
deploy-update-6747747c6f-nbvhc   	1/1     Running   0          9s
deploy-update-6747747c6f-q2sdl   	1/1     Running   0          9s
deploy-update-6747747c6f-rzd7d   	1/1     Running   0          9s
deploy-update-6747747c6f-t9nqj   	1/1     Running   0          9s
deploy-update-6747747c6f-whx2k   	1/1     Running   0          9s


[root@k8s-master ~]# kubectl  annotate  deployments  deploy-update \
> kubernetes.io/change-cause="Revistion 1 2026-08-19 : nginx:1.31 버전 최초 배포" --overwrite
deployment.apps/deploy-update annotated


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              Revistion 1 2026-08-19 : nginx:1.31 버전 최초 배포


[root@k8s-master ~]# kubectl  set  image  deployments  deploy-update   nginx-container=nginx:1.31.1


[root@k8s-master ~]# kubectl  get  pods  --watch
NAME                             		READY   STATUS    RESTARTS   AGE
deploy-update-6747747c6f-f9mrz   	1/1     Running   0          9s
deploy-update-6747747c6f-k9rcc   	1/1     Running   0          9s
deploy-update-6747747c6f-ksm6s   	1/1     Running   0          9s
deploy-update-6747747c6f-nbvhc   	1/1     Running   0          9s
deploy-update-6747747c6f-q2sdl   	1/1     Running   0          9s
deploy-update-6747747c6f-rzd7d   	1/1     Running   0          9s
deploy-update-6747747c6f-t9nqj   	1/1     Running   0          9s
deploy-update-6747747c6f-whx2k   	1/1     Running   0          9s

deploy-update-6cbccd8b88-rhx9k   	0/1     Pending   0          0s
deploy-update-6cbccd8b88-2ncl6   	0/1     Pending   0          0s

deploy-update-6cbccd8b88-rhx9k   	0/1     Pending   0          0s
deploy-update-6cbccd8b88-2ncl6   	0/1     Pending   0          0s

deploy-update-6747747c6f-q2sdl   	1/1     Terminating   0          7m15s
deploy-update-6747747c6f-f9mrz   	1/1     Terminating   0          7m15s

deploy-update-6cbccd8b88-rhx9k   	0/1     ContainerCreating   0          0s
deploy-update-6cbccd8b88-2ncl6   	0/1     ContainerCreating   0          0s
~~~~~~~~~~~~~~~~~~ 중간 생략 ~~~~~~~~~~~~~~~~~~


[root@k8s-master ~]# kubectl  annotate  deployments  deploy-update \
 kubernetes.io/change-cause="Revistion 2 2026-08-19 : nginx:1.31 버전  --> nginx:1.31.1 버전  업데이트" --overwrite
deployment.apps/deploy-update annotated


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update
deployment.apps/deploy-update
REVISION  CHANGE-CAUSE
1              Revistion 1 2026-08-19 : nginx:1.31 버전 최초 배포
2              Revistion 2 2026-08-19 : nginx:1.31 버전  --> nginx:1.31.1 버전  업데이트


[root@k8s-master ~]# kubectl  rollout  history  deployment  deploy-update  --revision=2
deployment.apps/deploy-update with revision #2
Pod Template:
  Labels:       	app=web
        pod-template-hash=6cbccd8b88
        track=stable
  Annotations:  kubernetes.io/change-cause: Revistion 2 2026-08-19 : nginx:1.31 버전  --> nginx:1.31.1 버전  업데이트
  Containers:
   nginx-container:
    Image:      	nginx:1.31.1
    Port:       	80/TCP
    Host Port:  	0/TCP
    Environment:        	<none>
    Mounts:     	<none>
  Volumes:      	<none>
  Node-Selectors:       	<none>
  Tolerations:  	<none>


[root@k8s-master ~]# kubectl  get  rs
NAME                       	DESIRED   CURRENT   READY   AGE
deploy-update-6747747c6f   	0             0                0           12m
deploy-update-6cbccd8b88   	8             8                8           5m35s


[root@k8s-master ~]# kubectl  delete  deployments  deploy-update
deployment.apps "deploy-update" deleted from default namespace
```

---

## 10. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- **change-cause가 `<none>`으로 보인다** — `kubernetes.io/change-cause` annotation을 별도로 남기지 않으면 rollout history에 사유가 기록되지 않는다. `--record`(deprecated)나 `kubectl annotate deployment <name> kubernetes.io/change-cause="..." --overwrite`로 매 배포마다 기록해야 한다.
- **annotation만 바꿨는데 Revision이 그대로다** — `spec.template`(이미지/환경변수/label 등)이 바뀌지 않으면 새 ReplicaSet이 생성되지 않아 Revision이 증가하지 않는다. 이 상태에서 change-cause annotation을 다시 걸면 새 Revision이 생기는 게 아니라 기존 Revision의 CHANGE-CAUSE 값이 덮어쓰기된다.
- **`kubectl rollout status`가 "N out of M new replicas have been updated..."에서 멈춘다** — 잘못된 이미지 태그 등으로 새 Pod가 ErrImagePull/ImagePullBackOff에 빠지면 롤링 업데이트가 완료되지 않는다. `kubectl get pods`로 STATUS를 확인하고, `kubectl rollout undo deployment <name>` (또는 `--to-revision=<N>`)으로 이전 정상 버전으로 되돌린다.
- **롤백 후에도 오래된 ReplicaSet에 Pod가 남아 있다** — 롤링 업데이트 도중 롤백하면 진행 중이던 새 ReplicaSet이 아니라 그 전 정상 ReplicaSet으로 scale up되므로, `kubectl get rs`로 확인했을 때 예상과 다른(더 오래된) ReplicaSet에 Pod가 떠 있을 수 있다.
- **pause 상태에서 이미지를 바꿨는데 Pod가 그대로다** — 정상 동작이다. `kubectl describe deployments <name> | grep Image`로 보면 스펙상 이미지는 바뀌어 있지만, 실제 Pod 교체는 `kubectl rollout resume deployment <name>` 실행 전까지 일어나지 않는다.
- **resume 후 예상과 다른 ReplicaSet이 재사용된다** — resume 시 Deployment Controller는 현재 Pod Template과 동일한 설정의 기존 ReplicaSet이 남아 있으면(revisionHistoryLimit 이내) 새로 만들지 않고 그 ReplicaSet을 scale up한다. `kubectl get rs`의 AGE 값이 예상보다 오래된 이유가 여기에 있다.
- **rollout history에 여러 Revision의 CHANGE-CAUSE가 똑같이 찍혀 있다** — annotate 없이 여러 번 `set image`를 실행하면 새 Revision들이 이전 CHANGE-CAUSE 문구를 그대로 물려받아 보일 수 있다. 배포마다 change-cause를 갱신하는 습관이 필요하다.

---

> 📌 **핵심 요약**
> - `kubectl set image deployment <name> <container>=<image>`로 이미지를 바꾸면 자동으로 Rolling Update가 시작되며, `kubectl rollout status`로 진행 상태를 확인할 수 있다
> - `kubernetes.io/change-cause` annotation을 매 배포마다 남겨야 `kubectl rollout history`에서 각 Revision의 변경 사유를 추적할 수 있다 (자동 기록 아님)
> - `kubectl rollout undo deployment <name>` (또는 `--to-revision=<N>`)으로 이전 정상 버전으로 즉시 롤백할 수 있으며, 롤백도 새로운 Revision으로 기록된다
> - 잘못된 이미지로 인해 rollout이 멈추면(ErrImagePull) `kubectl rollout undo`로 복구하고, `kubectl rollout pause`/`resume`으로 여러 변경 사항을 모아 한 번에 배포할 수 있다
> - 관련: 12. 🎛️ Kubernetes - Deployment · 2. 📦 Kubernetes - Pod 생성
