# Kubernetes - 통합 정리

> **Tag:** #Kubernetes #통합정리 #요약 #부트캠프
> **핵심 요약:** 31개 Kubernetes 문서에서 실제로 다룬 실습·명령어·트러블슈팅을 주제별로 압축 정리

---

## 1. 핵심 기술 개념 (Concept)

### 실습 클러스터 환경 (전체 문서 공통 기준)

- Rocky Linux 9.8 (Blue Onyx), containerd 2.3.3, kubeadm/kubelet **v1.35.7**
- Master(k8s-master, 192.168.10.100, control-plane) + Worker1(.101) + Worker2(.102)
- CNI: Flannel(Pod CIDR `10.244.0.0/16`), Ingress Controller: `ingress-nginx controller-v1.15.1`(baremetal provider)
- `containerd`는 Docker CE 공식 repo(`download.docker.com/linux/centos/9`)에서 `containerd.io` 패키지로 설치 (Rocky 자체 repo엔 없음)

### 1~8. Pod 기초 흐름

| 주제 | 핵심 내용 |
|---|---|
| 1. 설치 | `kubeadm init --pod-network-cidr=10.244.0.0/16` → Flannel apply → `kubeadm token create --print-join-command` → 워커에서 `kubeadm join`. `/etc/containerd/config.toml`에서 `disabled_plugins`의 `"cri"` 제거 + `SystemdCgroup = true` 필수 |
| 2. Pod 생성 | `kubectl run`으로 만든 Pod는 삭제하면 그대로 사라지지만, `kubectl create deployment`로 만든 Pod를 개별 삭제하면 replicas 유지를 위해 즉시 재생성됨 |
| 3. 아키텍처 | Pod 생성은 kubectl → API Server(Pending 기록) → Scheduler(nodeName 결정) → kubelet(Watch로 감지, 실제 실행) 순서로 6단계 |
| 4. Namespace | `kubectl create`(멱등성 없음, 재실행 시 `AlreadyExists` 에러) vs `kubectl apply`(멱등성 있음, `unchanged`). `kubectl config set-context --namespace=`로 기본 namespace 전환 시 `-n` 없이 `delete pods --all` 실행하면 그 namespace만 삭제됨(사고 위험) |
| 5. ResourceQuota·LimitRange | requests만 있고 limits가 없으면 Quota가 4개 항목(requests/limits × cpu/memory)을 전부 요구해 Pod가 통째로 거부될 수 있음. CPU 초과는 Throttling(컨테이너 생존), Memory 초과는 `OOMKilled`(강제 종료) — `polinux/stress` 이미지로 실증 |
| 6. Pod 구조·생명주기 | Multi-Container Pod는 `kubectl exec -c <컨테이너명>`으로 컨테이너 지정 필요 |
| 7. livenessProbe | 실패 시 컨테이너 강제 종료(exit code **137**, SIGKILL) 후 재시작. Pod IP는 유지됨(Infra/pause 컨테이너가 유지하기 때문) |
| 8. readinessProbe | 실패해도 `STATUS: Running` 유지, RESTARTS 증가 안 함 — Service의 EndpointSlice에서만 제외됨. livenessProbe와의 핵심 차이 |

### 9~16. Controller 계열

| 주제 | 핵심 내용 |
|---|---|
| 9. Init Container·Static Pod | Init Container가 `exit 1`이면 메인 컨테이너 실행 안 됨. **Static Pod는 `kubectl delete`로 삭제 불가** — kubelet이 즉시 재생성하므로 워커 노드의 `/etc/kubernetes/manifests/`에서 YAML 파일 자체를 지워야 함. Pod 이름 뒤에 노드명이 자동으로 붙음 |
| 10. Controller 개념(RC) | **template의 image를 바꿔도 기존 Pod에는 반영 안 됨** — RC는 selector 라벨만 보고 이미지 버전은 확인하지 않으므로, 기존 Pod를 직접 삭제해야 새 이미지로 재생성됨 |
| 11. ReplicaSet | `matchExpressions(In)`으로 라벨 셀렉터의 OR 조건 표현(`matchLabels`는 AND만 가능). 라벨이 selector 밖으로 바뀐 Pod는 삭제되지 않고 관리 대상에서만 제외됨. `--cascade=orphan` 권장(`--cascade=false`는 deprecated) |
| 12. Deployment | Deployment → ReplicaSet → Pod 계층, Pod 이름에 RS 해시가 붙음(`-5796ddf486-`). `RollingUpdate`(maxSurge/maxUnavailable)가 기본, `Recreate`는 다운타임 발생 |
| 13. Rollout·Rollback | `kubernetes.io/change-cause`는 자동 기록 안 됨 — `annotate`로 매번 남겨야 함. **이미지 오타(`nginx:not-exist`)로 `ErrImagePull` 발생 시 `rollout status`가 멈추고, `rollout undo`로 복구.** 롤백 시 `revisionHistoryLimit` 내에 동일 템플릿의 오래된 ReplicaSet이 남아있으면 새로 안 만들고 그걸 재사용함 |
| 14. DaemonSet | `replicas` 필드 없음. **ReplicaSet이 아니라 ControllerRevision으로 이력 관리** — change-cause를 남기려면 최초 배포 전에 YAML에 `annotations`로 미리 넣어둬야 함(사후 annotate는 Revision 1엔 반영 안 됨) |
| 15. StatefulSet | Pod 이름 고정(`sf-nginx-0,1,2`). 생성은 0→1→2 순서, 삭제/축소/롤링업데이트는 항상 역순(2→1→0). scale-in 시 PVC는 자동 삭제 안 됨(`persistentVolumeClaimRetentionPolicy`로 제어). Headless DNS: `sf-nginx-0.sf-service.default.svc.cluster.local` |
| 16. Job | **성공/실패 판단은 오직 exit code로만**. `restartPolicy`는 `Never`/`OnFailure`만 허용(`Always` 불가 — Job에 안 쓰면 sleep 성공 종료에도 무한 재시작 반복). `backoffLimit` 기본값 6, 초과 시 `BackoffLimitExceeded`로 `Failed` |

### 17~24. Service·Ingress·Label

| 주제 | 핵심 내용 |
|---|---|
| 17. CronJob | Job 이름 뒤 숫자(`cronjob-exam-29786655-...`)는 Unix epoch 기반 스케줄 인덱스. `concurrencyPolicy: Forbid`는 이전 Job 실행 중이면 다음 Job 건너뜀, `Allow`는 동시 실행 허용 |
| 18. Service·ClusterIP | `clusterIP` 필드로 IP 고정 지정 가능(기본 대역 `10.96.0.0/12`). Endpoints가 비면 Selector와 Pod Label 불일치가 가장 흔한 원인 |
| 19. NodePort·LoadBalancer | `nodePort`도 명시적으로 고정 지정 가능(30000-32767 범위). 온프레미스 환경에서 LoadBalancer의 `EXTERNAL-IP`가 `<pending>`으로 남는 것은 클라우드 컨트롤러가 없어 정상 |
| 20. ExternalName·Headless | Headless(`clusterIP: None`)는 StatefulSet의 `serviceName`과 정확히 일치해야 Pod별 고정 DNS가 생성됨. 특정 Pod 조회 시 축약 이름이 아니라 `<Pod>.<Service>.<ns>.svc.cluster.local` 전체 FQDN 필요 |
| 21. Ingress 기초 | `ingress-nginx controller-v1.15.1`을 baremetal provider 매니페스트로 설치 |
| 22. host·path 기반 Ingress | `pathType: Prefix`만 쓰면 하위 경로 문자열이 그대로 백엔드에 전달돼 파일 경로가 어긋나는 문제 발생 → 정규식 rewrite 필요(23번으로 이어짐) |
| 23. 정규식 Ingress·Canary | `nginx.ingress.kubernetes.io/use-regex: "true"` + `rewrite-target: /$2` + `path: /curriculum(/|$)(.*)` 패턴. Canary는 v1/v2 Deployment를 같은 Service selector(`app`만 공통, `version` 라벨은 제외)로 묶고 **replicas 비율로 트래픽 비율 조절**, `replicas: 0`이면 즉시 트래픽 완전 차단 |
| 24. Label | `-l 'app in (web,api)'` 형태의 set-based selector로 OR 조건 표현(쉼표는 AND). Node Label을 바꿔도 이미 배치된 Pod는 재배치되지 않음(nodeSelector는 최초 스케줄링 시점에만 평가) |

### 25~31. Scheduling·Storage·설정·AutoScaling

| 주제 | 핵심 내용 |
|---|---|
| 25. nodeSelector·Affinity | `nodeSelectorTerms` 묶음끼리는 OR, 같은 term 안 `matchExpressions`는 AND. `preferred`는 여러 조건의 **weight(1~100)를 합산**해 가장 높은 노드를 우선 배치 |
| 26. Taint·Toleration | `NoSchedule`은 기존 Pod 유지, `NoExecute`는 기존 Pod까지 제거. `cordon`은 내부적으로 `node.kubernetes.io/unschedulable:NoSchedule` Taint로 구현됨. `drain`은 `--ignore-daemonsets` 필수, 이후에도 Cordon 상태는 `uncordon`으로 직접 풀어야 함 |
| 27. Storage(emptyDir·hostPath) | emptyDir을 웹 루트 경로에 마운트하면 기존 이미지의 index.html이 빈 디렉터리로 덮여 **403 Forbidden** 발생. `hostPath.type: Directory`는 경로 없으면 생성 실패, `DirectoryOrCreate`는 생성하되 내용은 비어있음 |
| 28. PV·PVC·StorageClass | NFS + `nfs-subdir-external-provisioner`(Helm) 조합의 Dynamic Provisioning. `reclaimPolicy: Retain`은 PVC 삭제 후에도 PV가 `Released`로 남아 수동 정리 필요, `Delete`는 PVC 삭제 시 PV까지 자동 삭제. Dynamic PV 이름은 `pvc-<UUID>` 자동 생성 |
| 29. ConfigMap | 환경변수(`envFrom`)로 주입한 값은 Pod 재시작 전까지 미반영. **Volume Mount는 `..data` symlink 구조라 Pod 재시작 없이 자동 갱신됨** |
| 30. Secret | Base64는 암호화가 아니라 인코딩 — `echo -n` 없이 인코딩하면 개행 문자까지 포함되는 흔한 실수. `kubectl describe secrets`는 값 대신 byte 수만 표시 |
| 31. AutoScaling | HPA CPU 사용률은 limits가 아니라 **requests 대비**로 계산(`실사용/requests×100`). `필요 Pod 수 = 현재 Pod 수 × 현재 Metric / 목표 Metric` 공식으로 replicas 산정. Metrics Server 설치 후 자체 서명 인증서 환경에서는 `--kubelet-insecure-tls` 옵션 추가가 거의 필수 |

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** kubeadm 기반 Kubernetes 클러스터(Master 1대 + Worker 2대) 설치 환경.

### 클러스터 구성 흐름 요약

```bash
# Master Node
kubeadm init --pod-network-cidr=10.244.0.0/16
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubeadm token create --print-join-command

# Worker Node
kubeadm join 192.168.10.100:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# 확인
kubectl get nodes
kubectl get pods --all-namespaces
```

---

> **핵심 요약**
> - Pod 상태는 kubelet의 Watch 기반 감시로 유지되며, Deployment/RC/RS가 관리하는 Pod를 개별 삭제해도 즉시 재생성된다 (단, Static Pod는 예외 — manifest 파일 자체를 지워야 함)
> - livenessProbe(재시작, exit 137) vs readinessProbe(Endpoint 제외만, 재시작 없음)의 차이는 실습에서 RESTARTS 컬럼 변화 여부로 가장 뚜렷하게 드러난다
> - Controller 계열은 이력 관리 방식이 서로 다르다: Deployment=ReplicaSet, DaemonSet=ControllerRevision, RC/RS=변경 이력 없음(image 변경도 기존 Pod에 자동 반영 안 됨)
> - ConfigMap/Secret은 env 주입 시 Pod 재시작이 필요하지만 Volume Mount는 symlink(`..data`) 구조 덕분에 자동 갱신된다
> - Ingress의 host/path 라우팅은 `pathType: Prefix`만으로는 하위 경로가 깨지기 쉬워 `use-regex` + `rewrite-target: /$2` 조합이 필요하고, Canary는 Deployment replicas 비율로 트래픽을 조절한다
> - HPA는 requests 대비 사용률을 기준으로 `현재 Pod 수 × 현재 Metric / 목표 Metric` 공식으로 replicas를 계산한다
> - 관련: 13. Kubernetes - Rollout·Rollback 실습 · 23. Kubernetes - 정규표현식 Ingress·Canary 배포 · 33. Kubernetes - 트러블슈팅 치트시트 · 34. Kubernetes - 퀵 레퍼런스
