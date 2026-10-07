# Kubernetes - 퀵 레퍼런스

> **Tag:** #Kubernetes #퀵레퍼런스 #치트시트 #명령어
> **핵심 요약:** 31개 문서 실습에서 실제로 사용된 kubectl 명령어 패턴 모음

---

## 1. 핵심 기술 개념 (Concept)

### 자주 헷갈리는 항목

| 항목 | 설명 |
|---|---|
| `kubectl apply` vs `create` | apply: 있으면 수정, 없으면 생성(선언적) / create: 재실행 시 `AlreadyExists` 에러 |
| RC/ReplicaSet의 image 변경 | template의 image를 바꿔도 **기존 Pod에는 반영 안 됨** — 기존 Pod를 직접 삭제해야 새 이미지 적용 |
| Deployment vs DaemonSet 이력 관리 | Deployment: ReplicaSet 기반 / DaemonSet: **ControllerRevision** 기반(change-cause는 최초 배포 전 annotation으로 미리 넣어야 함) |
| `kubectl delete pod`(Static Pod) | API Server가 아니라 kubelet이 관리 — 워커 노드의 `/etc/kubernetes/manifests/`에서 파일 자체를 삭제해야 함 |
| livenessProbe vs readinessProbe | 실패 시 재시작(exit 137) / 실패 시 Endpoint 제외만(재시작 없음) |
| ConfigMap/Secret 주입 방식 | env(`envFrom`): Pod 재시작 필요 / Volume Mount: `..data` symlink로 자동 갱신 |
| `--cascade=orphan` vs 기본 삭제 | orphan: 관리 대상 Pod는 남기고 ReplicaSet만 삭제 / 기본: Pod까지 함께 삭제 |
| Job의 `restartPolicy` | `Never` 또는 `OnFailure`만 허용 (`Always` 사용 불가 — 무한 재시작 유발) |
| NodePort vs LoadBalancer(온프레미스) | NodePort: 즉시 접속 가능 / LoadBalancer: 클라우드 LB 컨트롤러 없으면 `EXTERNAL-IP` 계속 `<pending>` |
| `echo` vs `echo -n` (Secret) | `-n` 없이 base64 인코딩하면 디코딩 값 끝에 개행 문자 포함됨 |

---

## 2. 표준 설정 템플릿 (Configuration)

### Namespace + Pod (멀티 리소스 단일 파일)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: soldesk
---
apiVersion: v1
kind: Pod
metadata:
  name: studyweb
  namespace: soldesk
spec:
  containers:
    - name: nginxweb
      image: nginx:1.31
      ports:
        - containerPort: 80
```

### Deployment + Probe + Rolling Update 템플릿

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
  annotations:
    kubernetes.io/change-cause: "초기 배포"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service (ClusterIP 고정 IP 지정) + NodePort 고정

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service
spec:
  type: ClusterIP
  clusterIP: 10.100.100.100
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30100
```

### Ingress (정규식 rewrite-target + Canary)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sol-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /curriculum(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: curriculum-service
            port:
              number: 80
```

### nodeSelector + Taint/Toleration + Affinity(weight)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  nodeSelector:
    role: db
  tolerations:
  - key: "role"
    operator: "Equal"
    value: "db"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values: [prod]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 70
        preference:
          matchExpressions:
          - key: region
            operator: In
            values: [seoul]
  containers:
  - name: nginx
    image: nginx:1.31
```

### ConfigMap + Secret 주입(env + Volume 동시)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deploy
spec:
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.31
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: web-page
```

### PVC (Dynamic Provisioning, NFS)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-log
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: sc-nfs-multi
  resources:
    requests:
      storage: 1Gi
```

### HPA (CPU + Memory)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deploy
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 설치 & 클러스터 구성

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubeadm token create --print-join-command
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
kubeadm reset -f                          # 재설치 전 초기화
```

### Namespace & Context

```bash
kubectl create namespace soldesk --dry-run=client -o yaml > soldesk-ns.yaml
kubectl config set-context soldesk@kubernetes --cluster=kubernetes --user=kubernetes-admin --namespace=soldesk
kubectl config use-context soldesk@kubernetes
kubectl get pod --all-namespaces          # -A
```

### Pod / 기본 조회

```bash
kubectl run webserver --image=nginx:latest --port 80
kubectl get pods -o wide --watch
kubectl describe pods <pod>
kubectl exec <pod> -c <container명> -it -- /bin/bash   # Multi-Container Pod
kubectl logs <pod> <init-container명>                   # Init Container 로그
```

### Controller (RC / ReplicaSet / Deployment)

```bash
kubectl scale rc <name> --replicas=2
kubectl edit rc <name>
kubectl delete rs <name> --cascade=orphan          # Pod는 남기고 RS만 삭제
kubectl label pod <pod> <key>=<value> --overwrite   # 관리 대상 라벨 실시간 변경
kubectl get pod <pod> -o yaml | grep -A3 ownerReferences
kubectl get deployments.apps -o wide                # READY/UP-TO-DATE/AVAILABLE
```

### Rollout / Rollback

```bash
kubectl set image deployments <name> <container>=<image>:<tag>
kubectl annotate deployments.apps <name> kubernetes.io/change-cause="..." --overwrite
kubectl rollout history deployment <name> [--revision=N]
kubectl rollout undo deployment <name> [--to-revision=N]
kubectl rollout status deployment <name>
kubectl rollout pause deployment <name>
kubectl rollout resume deployment <name>
```

### DaemonSet / StatefulSet

```bash
kubectl rollout undo daemonset <name>
kubectl get controllerrevisions
kubectl scale statefulset <name> --replicas=5       # 0,1,2,3,4 순서로 증가
kubectl scale statefulset <name> --replicas=2       # 4,3 역순으로 종료
```

### Job / CronJob

```bash
kubectl create job my-job --image=centos:7 -- sleep 5
kubectl logs job/<name>
kubectl describe jobs.batch <name>          # BackoffLimitExceeded 등 Events 확인
kubectl get cronjobs.batch
kubectl get jobs.batch
```

### Service / Ingress

```bash
kubectl describe svc <svc-name>             # Endpoints/Selector 확인
kubectl get endpointslice -l kubernetes.io/service-name=<svc-name>
kubectl get ingress
kubectl describe ingress <name>             # Rules/Backends/Events
```

### Label / Selector

```bash
kubectl get pod -l app=web,env=prod         # AND 조건
kubectl get pod -l 'app in (web,api)'       # OR 조건 (set-based)
kubectl label pod <pod> app-                # 라벨 삭제 (끝에 -)
kubectl get nodes -l disktype=ssd -L disktype,env   # -L로 라벨 값을 컬럼 출력
```

### Scheduling (Affinity / Taint)

```bash
kubectl label node <node> disktype=ssd
kubectl taint nodes <node> gpu=true:NoSchedule
kubectl taint node <node> gpu-              # Taint 제거 (끝에 -)
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
kubectl describe nodes <node> | grep Taint
```

### Storage

```bash
kubectl describe pvc <pvc-name>             # Events에서 Provisioning 단계 확인
kubectl get storageclass
kubectl exec <pod> -it -- mount | grep <mountPath>
```

### ConfigMap / Secret

```bash
kubectl create configmap app-config --from-literal=APP_NAME=spring-app
kubectl create configmap web-page --from-file=index.html
kubectl edit configmaps <name>              # Volume Mount는 재시작 없이 자동 반영
kubectl create secret generic app-secret --from-literal=PASSWORD=1234
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d
echo -n "admin" | base64                    # -n 필수 (개행 방지)
```

### AutoScaling

```bash
kubectl top nodes
kubectl top pods
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl edit deployments.apps metrics-server --namespace kube-system   # --kubelet-insecure-tls 추가
kubectl get hpa
watch -n 2 kubectl get hpa <name>
```

---

> **핵심 요약**
> - `apply`(선언적, 있으면 수정) vs `create`(있으면 `AlreadyExists` 에러) — 재실행 안전성 때문에 실무에서는 `apply` 위주로 사용
> - RC/ReplicaSet은 image 변경이 기존 Pod에 자동 반영되지 않는다 — 반드시 `kubectl delete pod`로 재생성 유도
> - Rollout 계열(`status`/`history`/`undo`/`pause`/`resume`)과 `annotate ... change-cause`는 항상 세트로 사용해야 이력이 남는다
> - Label Selector(`-l`)는 AND(쉼표)와 OR(`in (...)`) 두 문법을 구분해서 사용한다
> - Secret은 `echo -n`으로 인코딩하고 `base64 -d`로 즉시 검증할 수 있다(암호화 아님, 인코딩일 뿐)
> - 관련: 12. Kubernetes - Deployment · 13. Kubernetes - Rollout·Rollback 실습 · 32. Kubernetes - 통합 정리 · 33. Kubernetes - 트러블슈팅 치트시트
