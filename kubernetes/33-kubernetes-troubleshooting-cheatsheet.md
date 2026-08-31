# 🚑 Kubernetes - 트러블슈팅 치트시트

> **Tag:** #Kubernetes #트러블슈팅 #에러 #치트시트 #부트캠프
> **핵심 요약:** 자주 발생하는 Kubernetes 오류 시나리오와 해결 방법 핵심 정리

---

## 1. 🎯 핵심 기술 개념 (Concept)

### 트러블슈팅 핵심 원칙

| 상황 | 우선 확인 명령 |
|---|---|
| Pod 상태 이상 | `kubectl get pods -o wide` / `kubectl describe pod <pod>` |
| Pod 로그 확인 | `kubectl logs <pod>` / `kubectl logs <pod> --previous` |
| 이벤트 확인 | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| Service 연결 문제 | `kubectl get endpoints <svc>` / `kubectl describe svc <svc>` |
| 리소스 정의 확인 | `kubectl get <resource> <name> -o yaml` |

---

## 2. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 시나리오 1. Pod가 Pending 상태에서 멈춤

**증상:** `kubectl get pods` 결과 `STATUS: Pending`이 오래 유지됨

**원인:** 스케줄링에 필요한 Node 자원 부족, Taint 미허용, PVC 미바인딩

```bash
kubectl describe pod <pod-name>
# Events 하단에서 원인 확인
# 예: Insufficient cpu / Insufficient memory
# 예: 0/3 nodes are available: 3 node(s) had taint

kubectl get nodes -o wide
kubectl describe node <node-name>   # Allocatable, Taints 확인
```

### 시나리오 2. CrashLoopBackOff

**증상:** Pod가 계속 재시작을 반복하며 `STATUS: CrashLoopBackOff`

**원인:** 컨테이너 내부 프로세스가 즉시 종료됨, livenessProbe 반복 실패, 설정값 오류

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous     # 직전에 종료된 컨테이너 로그
kubectl describe pod <pod-name>        # Last State, Exit Code 확인

# 흔한 원인
# - CMD/ENTRYPOINT가 즉시 종료되는 명령
# - livenessProbe의 initialDelaySeconds가 너무 짧음
# - 환경변수/ConfigMap 미존재로 애플리케이션 시작 실패
```

### 시나리오 3. ImagePullBackOff / ErrImagePull

**증상:** `STATUS: ImagePullBackOff`

**원인:** 이미지 이름·태그 오타, Private Registry 인증 누락

```bash
kubectl describe pod <pod-name>
# Failed to pull image "xxx": rpc error ...

# 이미지 이름/태그 확인
# Private Registry 인증 필요 시
kubectl create secret docker-registry regcred \
  --docker-server=<registry> --docker-username=<user> \
  --docker-password=<pass>

# Pod spec에 추가
spec:
  imagePullSecrets:
  - name: regcred
```

### 시나리오 4. Service로 접속이 안 됨

**증상:** `curl <ClusterIP>:포트` 연결 실패 또는 응답 없음

**원인:** Selector와 Pod Label 불일치, targetPort 오타, Pod가 readinessProbe 실패

```bash
# Endpoints가 비어있으면 Selector 불일치
kubectl get endpoints <svc-name>

# Service와 Pod Label 비교
kubectl get svc <svc-name> -o yaml | grep -A 3 selector
kubectl get pods --show-labels

# readinessProbe 실패 시 Endpoints에서 제외됨
kubectl describe pod <pod-name>
```

### 시나리오 5. NodePort/LoadBalancer로 외부 접속 안 됨

**증상:** NodeIP:NodePort 접속 실패

**원인:** 방화벽 미허용, NodePort 범위(30000-32767) 벗어남, 클라우드 LB 미설치 환경

```bash
kubectl get svc <svc-name>          # PORT(S) 컬럼에서 NodePort 확인
firewall-cmd --list-ports           # 방화벽 확인
firewall-cmd --add-port=<port>/tcp --permanent
firewall-cmd --reload

# LoadBalancer 타입인데 <pending> 상태면
# 클라우드 환경이 아니거나 MetalLB 등 별도 구성 필요
```

### 시나리오 6. Ingress 접속 시 404 / default backend

**증상:** Ingress Host/Path로 접속 시 404 Not Found 또는 default backend 응답

**원인:** Ingress Controller 미설치, host/path 규칙 불일치, Service 이름 오타

```bash
kubectl get ingress
kubectl describe ingress <ingress-name>

# Ingress Controller Pod 확인
kubectl get pods -n ingress-nginx

# /etc/hosts에 도메인 매핑 확인 (테스트 환경)
cat /etc/hosts
```

### 시나리오 7. PVC가 Pending 상태

**증상:** `kubectl get pvc` 결과 `STATUS: Pending`

**원인:** 일치하는 PV 없음, StorageClass 미지정/오타, Dynamic Provisioner 미설치

```bash
kubectl describe pvc <pvc-name>
kubectl get pv
kubectl get storageclass

# StorageClass 이름 확인 후 PVC에 정확히 지정
spec:
  storageClassName: <sc-name>
```

### 시나리오 8. ConfigMap/Secret 수정 후 반영 안 됨

**증상:** ConfigMap을 수정했는데 애플리케이션 동작이 그대로임

**원인:** env로 주입된 값은 Pod 재시작 전까지 반영되지 않음 (volume 마운트는 자동 갱신되나 앱이 재로드해야 함)

```bash
# env 방식은 Pod 재생성 필요
kubectl rollout restart deployment <deployment-name>

# volume 마운트 방식은 파일은 갱신되지만 앱이 재읽기 하지 않으면 미반영
kubectl exec -it <pod-name> -- cat /etc/config/<key>
```

### 시나리오 9. HPA TARGETS가 `<unknown>`으로 표시

**증상:** `kubectl get hpa` 결과 TARGETS 컬럼이 `<unknown>/50%`

**원인:** Metrics Server 미설치, 대상 Pod에 `resources.requests` 미설정

```bash
kubectl top nodes                   # error 나오면 Metrics Server 문제
kubectl get pods -n kube-system | grep metrics-server
kubectl describe hpa <hpa-name>

# Deployment에 requests.cpu 설정 필수
resources:
  requests:
    cpu: 200m
```

### 시나리오 10. Node가 NotReady 상태

**증상:** `kubectl get nodes` 결과 `STATUS: NotReady`

**원인:** kubelet 중지, 네트워크 플러그인(CNI) 문제, 디스크/메모리 부족

```bash
kubectl describe node <node-name>   # Conditions 하단 원인 확인

# 해당 Node에서 직접 확인
systemctl status kubelet
journalctl -u kubelet -f
df -h                                # DiskPressure 확인
```

---

> 📌 **핵심 요약**
> - Pod 문제는 항상 `kubectl describe pod` → Events 순으로 원인을 좁혀나간다
> - Pending은 스케줄링 실패(자원·Taint·PVC), CrashLoopBackOff는 컨테이너 내부 실행 실패, ImagePullBackOff는 이미지/인증 문제로 구분
> - Service 연결 실패는 `kubectl get endpoints`가 비어있는지부터 확인 (Selector 불일치 또는 readinessProbe 실패)
> - ConfigMap/Secret은 env 주입 시 Pod 재시작이 필요하다
> - HPA가 `<unknown>`을 보이면 Metrics Server 설치 여부와 Pod의 requests 설정을 먼저 확인한다
> - 관련: 7. 💓 Kubernetes - livenessProbe · 18. 🔌 Kubernetes - Service 기초와 ClusterIP · 28. 💽 Kubernetes - PV·PVC와 StorageClass·Dynamic Provisioning · 34. ⚡ Kubernetes - 퀵 레퍼런스
