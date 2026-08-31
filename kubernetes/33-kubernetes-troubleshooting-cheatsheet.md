# Kubernetes - 트러블슈팅 치트시트

> **Tag:** #Kubernetes #트러블슈팅 #에러 #치트시트 #부트캠프
> **핵심 요약:** 31개 문서 실습에서 실제로 재현된 오류 시나리오와 원인·해결 방법 정리

---

## 1. 핵심 기술 개념 (Concept)

### 트러블슈팅 핵심 원칙

| 상황 | 우선 확인 명령 |
|---|---|
| Pod 상태 이상 | `kubectl get pods -o wide --watch` / `kubectl describe pod <pod>` |
| Pod 로그 확인 | `kubectl logs <pod>` / `kubectl logs <pod> <init-container명>` |
| Node/스케줄링 문제 | `kubectl get nodes -L <label-key>` / `kubectl describe nodes <node> \| grep Taint` |
| Service 연결 문제 | `kubectl describe svc <svc>` (Endpoints 확인) |
| 이력/변경 확인 | `kubectl rollout history deployment <name>` |

---

## 2. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 시나리오 1. kubeconfig 없이 kubectl 실행 (설치 직후)

**증상:** `The connection to the server localhost:8080 was refused - did you specify the right host or port?`

**원인:** `kubeadm init` 직후 `~/.kube/config`를 설정하지 않아 kubectl이 기본값(localhost:8080)으로 접속 시도

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 시나리오 2. Namespace/리소스 재생성 시 AlreadyExists

**증상:** `kubectl create -f nginx.yaml` 재실행 시 `Error from server (AlreadyExists): pods "mypod" already exists`

**원인:** `create`는 멱등성이 없어 이미 존재하는 리소스에 다시 쓰면 에러. `apply`는 있으면 수정(`unchanged`), 없으면 생성

```bash
kubectl apply -f nginx.yaml   # AlreadyExists 없이 안전하게 재실행 가능
```

### 시나리오 3. ResourceQuota로 인한 Pod 생성 거부

**증상:**
```
Error from server (Forbidden): pods "base-pod3" is forbidden: exceeded quota: rq-pod-count, requested: pods=1, used: pods=2, limited: pods=2
Error from server (Forbidden): ... minimum cpu usage per Container is 200m, but request is 50m
Error from server (Forbidden): ... must specify limits.cpu for: nginx; limits.memory for: nginx; requests.cpu for: nginx; requests.memory for: nginx
```

**원인:** Namespace의 ResourceQuota 초과, LimitRange의 min 미달, 또는 requests/limits 4개 항목 중 일부만 지정

```bash
kubectl get resourcequotas -n <ns>          # used/hard 확인
kubectl describe limitranges <name> -n <ns> # min/max/default 확인
# Pod spec에 requests/limits 4개 항목 모두 명시
```

### 시나리오 4. CPU/Memory 초과 시 동작 차이 (Throttling vs OOMKilled)

**증상:** CPU limit 초과는 컨테이너가 죽지 않고 느려지기만 함(`ps` 상 %CPU가 limit 수준에서 고정) / Memory limit 초과는 `STATUS: OOMKilled`로 즉시 강제 종료

**원인:** CPU는 압축(throttle) 가능한 자원, Memory는 압축 불가능한 자원이라 커널이 OOM Killer로 처리

```bash
# 재현 (polinux/stress 이미지)
kubectl exec <pod> -- stress --cpu 1        # limit 500m → throttle만
kubectl exec <pod> -- stress --vm 1 --vm-bytes 150M   # limit 100Mi → OOMKilled
kubectl describe pod <pod>   # Last State: Terminated, Reason: OOMKilled
```

### 시나리오 5. livenessProbe 실패로 컨테이너 반복 재시작

**증상:**
```
Warning  Unhealthy  ...  Liveness probe failed: HTTP probe failed with statuscode: 404
Normal   Killing    ...  Container nginx failed liveness probe, will be restarted
command terminated with exit code 137
```

**원인:** probe 대상 경로(예: `/health`) 파일 삭제, 또는 컨테이너 내부 프로세스 강제 종료(`nginx -s stop`)

```bash
kubectl describe pod <pod> | grep -A 8 Liveness   # probe 설정 확인
kubectl get pods -o wide --watch                  # RESTARTS 증가 관찰
```

### 시나리오 6. readinessProbe 실패 — 재시작은 안 되지만 트래픽만 끊김

**증상:** `READY 0/1`이지만 `STATUS: Running` 유지, RESTARTS는 그대로. `kubectl get endpoints`/`endpointslice`에서 해당 Pod IP가 사라짐

**원인:** readinessProbe 실패 시 Service가 트래픽 대상에서만 제외할 뿐 재시작은 하지 않음(livenessProbe와의 핵심 차이)

```bash
kubectl get endpoints <svc-name>
kubectl get endpointslice -l kubernetes.io/service-name=<svc-name>
kubectl describe pod <pod>   # Readiness probe failed 이벤트 확인
```

### 시나리오 7. Init Container 실패로 메인 컨테이너 미실행

**증상:** Pod가 `Init:0/1` 또는 `Init:Error`에서 멈추고 메인 컨테이너가 시작되지 않음

**원인:** Init Container가 `exit 1`(비정상 종료)로 끝나면 다음 단계로 진행되지 않음

```bash
kubectl get pods -o wide --watch                  # Pending → Init:0/1 → PodInitializing
kubectl logs <pod> <init-container명>              # 실패한 init 컨테이너 로그 직접 확인
```

### 시나리오 8. Static Pod가 삭제해도 계속 재생성됨

**증상:** `kubectl delete pod <static-pod>` 실행 직후 동일한 이름(노드명 접미사 포함)으로 즉시 재생성됨

**원인:** Static Pod는 API Server가 아니라 워커 노드의 kubelet이 로컬 manifest 파일을 직접 감시해서 관리함

```bash
# 반드시 워커 노드에 직접 접속해서 파일을 삭제해야 함
ssh guest@k8s-worker1
sudo rm /etc/kubernetes/manifests/<pod-name>.yaml
```

### 시나리오 9. RC/RS의 template.image 변경이 기존 Pod에 반영 안 됨

**증상:** `kubectl edit rc`로 image를 바꿔도 `kubectl describe pod`에는 이전 이미지 그대로 표시

**원인:** ReplicationController/ReplicaSet은 selector 라벨 개수만 관리할 뿐 이미지 버전 일치 여부는 확인하지 않음(Deployment와의 근본 차이)

```bash
kubectl delete pod <pod-name>   # 강제 삭제해야 새 template.image로 재생성됨
```

### 시나리오 10. 이미지 오타로 인한 Rollout 정지 (ErrImagePull)

**증상:** `kubectl rollout status`가 `"2 out of 3 new replicas have been updated..."`에서 멈추고 `kubectl get pods`에 `ErrImagePull` 다수 발생

**원인:** `kubectl set image`로 지정한 태그가 존재하지 않는 이미지(예: `nginx:not-exist`)

```bash
kubectl rollout status deployment <name>
kubectl get pods                          # ErrImagePull 확인
kubectl rollout undo deployment <name>    # 이전 정상 리비전으로 즉시 복구
```

### 시나리오 11. Job 단일 Pod가 정상 종료 후에도 무한 재시작

**증상:** `kubectl run`으로 만든 Pod가 `sleep 5` 같은 짧은 작업을 정상 종료(exit 0)했는데도 `Completed → Running → CrashLoopBackOff`를 반복

**원인:** `kubectl run`은 기본 `restartPolicy: Always`를 쓰기 때문에 정상 종료도 "다시 실행해야 할 상태"로 간주됨. Job이 아니라 Job의 Pod template에서만 `Never`/`OnFailure` 사용 가능

```bash
# Job으로 만들면 정상 종료 시 재시작하지 않음
kubectl create job my-job --image=centos:7 -- sleep 5
```

### 시나리오 12. Job이 backoffLimit 초과로 완전히 실패

**증상:** `kubectl describe jobs.batch <name>`의 Events에 `BackoffLimitExceeded`

**원인:** 컨테이너 명령어 오타(예: `bashz`) 등으로 `RunContainerError → CrashLoopBackOff → StartError`가 반복되며 재시도 횟수(`backoffLimit`, 기본 6)를 초과

```bash
kubectl describe jobs.batch <name>   # Events 하단에서 BackoffLimitExceeded 확인
kubectl logs job/<name>
```

### 시나리오 13. Service로 접속이 안 됨 (Endpoints 비어있음)

**증상:** `curl <ClusterIP>` 무응답, `kubectl describe svc`의 Endpoints가 비어있음

**원인:** Service의 selector와 Pod의 Label이 불일치하거나, readinessProbe 실패로 Pod가 Endpoint에서 빠진 상태

```bash
kubectl get pods --show-labels
kubectl describe svc <svc-name> | grep -E 'Selector|Endpoints'
kubectl label pod <pod> <key>=<value> --overwrite   # 라벨 정정
```

### 시나리오 14. LoadBalancer의 EXTERNAL-IP가 계속 `<pending>`

**증상:** `kubectl get svc` 결과 `TYPE: LoadBalancer`의 `EXTERNAL-IP`가 `<pending>`으로 남음

**원인:** 온프레미스(kubeadm 자체 구축) 클러스터에는 클라우드 LB 컨트롤러가 없어 정상적인 상태 — MetalLB 등 별도 컴포넌트 필요

```bash
kubectl describe svc <svc-name>   # NodePort는 자동 할당돼 있으므로 그걸로 접속 가능
```

### 시나리오 15. Headless Service에서 특정 Pod DNS 조회 실패

**증상:** `nslookup web-headless`로는 개별 Pod를 찾을 수 없음

**원인:** 축약된 Service 이름만으로는 Pod 단위 DNS가 해석되지 않음 — 전체 FQDN이 필요

```bash
kubectl exec -it <pod> -- getent hosts <pod-name>.<service-name>
# 예: web-sts-1.web-headless
```

### 시나리오 16. Ingress path 하위 경로가 깨짐

**증상:** `/login`은 되는데 `/login/login.html` 같은 하위 리소스가 404 또는 엉뚱한 경로로 요청됨

**원인:** `pathType: Prefix`만 쓰면 매칭된 prefix 이후 문자열이 그대로 백엔드로 전달되어 nginx 내부 실제 파일 경로와 어긋남

```yaml
# 정규식 캡처 그룹 + rewrite-target으로 해결
annotations:
  nginx.ingress.kubernetes.io/use-regex: "true"
  nginx.ingress.kubernetes.io/rewrite-target: /$2
# path: /login(/|$)(.*)
```

### 시나리오 17. Pending Pod — nodeSelector/Affinity 조건 불만족

**증상:** `kubectl get pods` 결과 `STATUS: Pending`이 계속 유지, `kubectl describe pod`에 `0/3 nodes are available` 류의 메시지

**원인:** required Node/Pod Affinity 조건을 만족하는 노드(또는 Pod)가 클러스터에 존재하지 않음

```bash
kubectl get nodes -L <label-key>,<label-key2>   # 실제 노드 라벨 확인
kubectl label nodes <node> <key>=<value> --overwrite
```

### 시나리오 18. Pending Pod — Taint로 인한 배치 거부

**증상:** Toleration 없는 Pod가 모든 노드에서 Pending

**원인:** 모든 워커 노드가 `NoSchedule` Taint 또는 `cordon`(내부적으로 `node.kubernetes.io/unschedulable:NoSchedule` Taint) 상태

```bash
kubectl describe nodes <node> | grep Taint
kubectl taint node <node> <key>-        # Taint 제거 (끝에 - 필수)
kubectl uncordon <node>                 # Cordon 해제
```

### 시나리오 19. emptyDir 마운트 후 403 Forbidden

**증상:** 정상 동작하던 nginx 이미지에 emptyDir을 웹 루트(`/usr/share/nginx/html`)로 마운트했더니 403 응답

**원인:** emptyDir이 빈 디렉터리로 시작하기 때문에 이미지에 원래 있던 `index.html`을 가려버림

```bash
kubectl exec <pod> -- ls /usr/share/nginx/html   # 비어있는지 확인
# 별도 Init Container나 command로 콘텐츠를 채워 넣어야 함
```

### 시나리오 20. PVC가 Pending 상태

**증상:** `kubectl get pvc` 결과 `STATUS: Pending`

**원인:** static이면 조건에 맞는 PV가 없음, dynamic이면 provisioner Pod(`nfs-provisioner-...`)가 정상 동작하지 않음

```bash
kubectl describe pvc <pvc-name>              # Events에서 원인 확인
kubectl get pods -n <provisioner-ns>         # provisioner Pod 상태 확인
kubectl get storageclass
```

### 시나리오 21. ConfigMap/Secret 수정 후 반영 안 됨 (env 방식)

**증상:** ConfigMap/Secret 값을 바꿨는데 애플리케이션 동작이 그대로임

**원인:** `envFrom`으로 주입된 값은 Pod 재시작 전까지 갱신되지 않음. **Volume Mount 방식은 `..data` symlink 구조라 재시작 없이 자동 갱신됨** — 방식에 따라 대응이 다름

```bash
kubectl rollout restart deployment <name>   # env 방식은 재시작 필요
kubectl exec -it <pod> -- ls -l /etc/config # volume 방식은 심링크 자동 갱신 확인
```

### 시나리오 22. Secret Base64 인코딩 시 개행 문자 포함

**증상:** `echo "admin" | base64`로 만든 값을 디코딩하면 끝에 개행이 붙어 비교/로그인 실패

**원인:** `-n` 옵션 없이 `echo`를 사용하면 개행 문자까지 인코딩됨

```bash
echo -n "admin" | base64        # 올바른 방법
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d
```

### 시나리오 23. HPA TARGETS가 `<unknown>`으로 표시

**증상:** `kubectl get hpa` 결과 `TARGETS` 컬럼이 `<unknown>/50%`

**원인:** Metrics Server 미설치, 또는 대상 Pod에 `resources.requests.cpu` 미설정 (Utilization 기반 HPA는 requests가 반드시 필요)

```bash
kubectl top nodes                           # error 나오면 Metrics Server 문제
kubectl get pods -n kube-system | grep metrics-server
```

### 시나리오 24. Metrics Server가 kubelet과 TLS 통신 실패

**증상:** Metrics Server Pod는 Running인데 `kubectl top nodes/pods`가 계속 실패하거나 데이터가 안 보임

**원인:** 자체 서명된 kubelet 인증서 클러스터에서 metrics-server가 TLS 검증에 실패

```bash
kubectl edit deployments.apps metrics-server --namespace kube-system
# containers.args에 --kubelet-insecure-tls 추가
```

---

> **핵심 요약**
> - Pending은 스케줄링 실패(Affinity/Taint/자원/PVC), CrashLoopBackOff는 컨테이너 내부 실행 실패(또는 Job에 `restartPolicy: Always`를 잘못 씀), ErrImagePull은 이미지 태그 오류로 구분한다
> - livenessProbe 실패는 재시작(exit 137)을 유발하지만 readinessProbe 실패는 Endpoint 제외만 할 뿐 재시작하지 않는다 — RESTARTS 컬럼으로 구분
> - Static Pod, RC/RS의 image 변경처럼 "kubectl delete/edit이 안 먹히는 것처럼 보이는" 상황은 대부분 관리 주체(kubelet vs Controller)와 감시 기준(파일 경로 vs 라벨)을 오해한 경우다
> - ConfigMap/Secret은 env 주입 시 재시작이 필요하지만 Volume Mount는 symlink 구조로 자동 갱신된다
> - HPA가 `<unknown>`이면 Metrics Server 설치 여부와 `--kubelet-insecure-tls`, 대상 Pod의 requests 설정을 순서대로 확인한다
> - 관련: 7. Kubernetes - livenessProbe · 13. Kubernetes - Rollout·Rollback 실습 · 18. Kubernetes - Service 기초와 ClusterIP · 34. Kubernetes - 퀵 레퍼런스
