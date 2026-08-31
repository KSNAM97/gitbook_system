# ⚡ Kubernetes - 퀵 레퍼런스

> **Tag:** #Kubernetes #퀵레퍼런스 #치트시트 #명령어 #부트캠프
> **핵심 요약:** 코딩·운영 중 바로 찾아보는 kubectl 명령어 전체 패턴 모음

---

## 1. 🎯 핵심 기술 개념 (Concept)

### 자주 헷갈리는 항목

| 항목 | 설명 |
|---|---|
| `kubectl apply` vs `create` | apply: 있으면 수정, 없으면 생성(선언적) / create: 없을 때만 생성, 있으면 에러 |
| ReplicaSet vs Deployment | ReplicaSet: 단순 복제 유지 / Deployment: Rollout·Rollback까지 관리 |
| Deployment vs StatefulSet | Deployment: 무상태, Pod 이름 랜덤 / StatefulSet: 고정 이름·순서·볼륨 |
| Job vs CronJob | Job: 1회성 완료 작업 / CronJob: 스케줄에 따라 Job 반복 생성 |
| ClusterIP vs NodePort vs LoadBalancer | 내부만 / Node IP+포트로 외부 / 클라우드 LB로 외부 |
| livenessProbe vs readinessProbe | 실패 시 재시작 / 실패 시 트래픽에서만 제외 |
| ConfigMap vs Secret | 일반 설정(평문) / 민감 정보(Base64 인코딩) |
| PV vs PVC | PV: 실제 스토리지 자원 / PVC: Pod가 스토리지를 요청하는 명세 |
| emptyDir vs hostPath vs PVC | Pod 삭제 시 소멸 / Node에 고정 / 클러스터 추상화(영구) |
| nodeSelector/Affinity vs Taint/Toleration | Pod가 Node를 선택(끌어당김) / Node가 Pod를 거부(밀어냄) |
| `kubectl delete` vs `kubectl scale --replicas=0` | delete: 리소스 자체 제거 / scale 0: 리소스는 유지, Pod만 0개 |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### Deployment + Service + Probe 기본 템플릿

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:latest
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
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### ConfigMap + Secret + 주입 템플릿

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: "admin1234"
---
# Pod spec에서 사용
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secret
```

### Ingress 기본 템플릿

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

### HPA 기본 템플릿

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
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 클러스터 & Node 명령어

```bash
kubectl cluster-info                     # 클러스터 정보
kubectl get nodes                        # Node 목록
kubectl describe node <node>             # Node 상세(Taint, Allocatable 등)
kubectl top nodes                        # Node 리소스 사용량
kubectl cordon <node>                    # 스케줄링 제외
kubectl drain <node>                     # Pod 대피 후 유지보수
kubectl uncordon <node>                  # 스케줄링 재개
```

### Namespace 명령어

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl config set-context --current --namespace=dev   # 기본 namespace 변경
kubectl get pods -n dev
kubectl get pods --all-namespaces
```

### Pod 명령어

```bash
kubectl get pods                         # Pod 목록
kubectl get pods -o wide                 # IP, Node 포함
kubectl get pods --show-labels           # Label 포함
kubectl get pods -l app=web              # Label Selector로 조회
kubectl describe pod <pod>               # 상세 정보 + Events
kubectl logs <pod>                       # 로그
kubectl logs <pod> -c <container>        # 특정 컨테이너 로그
kubectl logs -f <pod>                    # 실시간 로그
kubectl exec -it <pod> -- /bin/bash      # 컨테이너 접속
kubectl delete pod <pod>                 # Pod 삭제
kubectl port-forward pod/<pod> 8080:80   # 로컬 포트 포워딩
```

### 리소스 생성/적용 명령어

```bash
kubectl apply -f deploy.yaml             # 생성/수정(선언적)
kubectl create -f deploy.yaml            # 생성(있으면 에러)
kubectl delete -f deploy.yaml            # 파일 기준 삭제
kubectl diff -f deploy.yaml              # 적용 전 변경사항 미리보기
kubectl edit deployment <name>           # 즉시 편집
```

### Deployment / Rollout 명령어

```bash
kubectl get deployments
kubectl scale deployment <name> --replicas=5
kubectl rollout status deployment <name>
kubectl rollout history deployment <name>
kubectl rollout undo deployment <name>
kubectl rollout undo deployment <name> --to-revision=2
kubectl set image deployment/<name> <container>=<image>:<tag>
```

### Service / Ingress 명령어

```bash
kubectl get svc
kubectl get endpoints <svc>
kubectl describe svc <svc>
kubectl get ingress
kubectl describe ingress <ingress>
```

### Label / Selector 명령어

```bash
kubectl label pod <pod> env=prod         # Label 추가
kubectl label pod <pod> env-             # Label 제거
kubectl get pods -l env=prod             # Label로 필터
kubectl get pods -l 'env in (prod,dev)'  # 다중 값 필터
```

### ConfigMap / Secret 명령어

```bash
kubectl create configmap app-config --from-literal=APP_MODE=production
kubectl create configmap app-config --from-file=config.txt
kubectl get configmap app-config -o yaml

kubectl create secret generic app-secret --from-literal=DB_PASSWORD=admin1234
kubectl get secret app-secret -o yaml
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### Storage 명령어

```bash
kubectl get pv
kubectl get pvc
kubectl describe pvc <pvc>
kubectl get storageclass
```

### AutoScaling 명령어

```bash
kubectl get hpa
kubectl describe hpa <hpa>
kubectl autoscale deployment <name> --min=1 --max=10 --cpu-percent=50
```

---

> 📌 **핵심 요약**
> - `apply`(선언적, 있으면 수정) vs `create`(명령적, 있으면 에러) — 실무에서는 `apply` 위주로 사용
> - Rollout 계열: `rollout status` / `rollout history` / `rollout undo`로 배포 상태 관리
> - Label Selector(`-l`)는 Pod 조회·필터링의 핵심 — Service/Deployment의 selector와 반드시 일치해야 함
> - Secret 값은 Base64 인코딩일 뿐 암호화가 아니므로 `base64 -d`로 바로 확인 가능
> - 관련: 12. 🎛️ Kubernetes - Deployment · 21. 🌐 Kubernetes - Ingress 기초와 준비 · 32. 🧩 Kubernetes - 통합 정리 · 33. 🚑 Kubernetes - 트러블슈팅 치트시트
