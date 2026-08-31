# 🧩 Kubernetes - 통합 정리

> **Tag:** #Kubernetes #통합정리 #요약 #부트캠프
> **핵심 요약:** Kubernetes 전체 개념 흐름 요약 및 핵심 비교표 모음

---

## 1. 🎯 핵심 기술 개념 (Concept)

### Kubernetes 전체 학습 흐름

```
1. 설치 & 아키텍처
   └─ kubeadm init/join, Control Plane(API Server/etcd/Scheduler/Controller Manager) vs Node(kubelet/kube-proxy/Container Runtime)

2. Pod 기초
   └─ Pod 생성, Pod 구조와 생성·동작 흐름, Namespace, ResourceQuota·LimitRange

3. Pod 상태 관리
   └─ livenessProbe(재시작), readinessProbe(트래픽 제외), Init Container·Static Pod

4. Controller
   └─ ReplicationController → ReplicaSet → Deployment(Rollout·Rollback)
   └─ DaemonSet(노드마다 1개), StatefulSet(고정 아이덴티티), Job/CronJob(배치 작업)

5. Service & Ingress
   └─ ClusterIP(내부) → NodePort/LoadBalancer(외부 노출) → ExternalName/Headless
   └─ Ingress 기초 → host·path 기반 라우팅 → 정규표현식·Canary 배포

6. Label & Scheduling
   └─ Label/Selector, nodeSelector·Affinity(끌어당김), Taint·Toleration(밀어냄)

7. Storage & 설정
   └─ emptyDir·hostPath → PV·PVC·StorageClass(Dynamic Provisioning)
   └─ ConfigMap(일반 설정) / Secret(민감 정보)

8. AutoScaling
   └─ HPA(Pod 개수) / VPA(requests 값) / Cluster Autoscaler(Node 개수)
```

### 핵심 비교표: Controller 종류

| 구분 | 관리 대상 | 개수 특징 | 대표 용도 |
|---|---|---|---|
| ReplicationController | Pod 복제 (구버전) | 지정 개수 유지 | 레거시, 거의 미사용 |
| ReplicaSet | Pod 복제 | 지정 개수 유지, Selector 기반 | Deployment 내부에서 사용 |
| Deployment | ReplicaSet | Rollout/Rollback 지원 | 무상태(Stateless) 앱 |
| DaemonSet | Pod | 모든(또는 특정) Node에 1개씩 | 로그 수집기, 모니터링 에이전트 |
| StatefulSet | Pod | 고정된 이름·순서·볼륨 유지 | DB, 분산 스토리지 |
| Job | Pod | 완료 후 종료 | 배치/일회성 작업 |
| CronJob | Job | 스케줄에 따라 Job 생성 | 정기 배치 작업 |

### 핵심 비교표: Probe 종류

| 구분 | 실패 시 동작 | 목적 |
|---|---|---|
| livenessProbe | 컨테이너 재시작 | 죽은 프로세스(Deadlock 등) 감지 |
| readinessProbe | Service 트래픽 대상에서 제외 (재시작 안 함) | 준비 안 된 상태에서 트래픽 차단 |

### 핵심 비교표: Service 종류

| 구분 | 노출 범위 | 특징 |
|---|---|---|
| ClusterIP | 클러스터 내부 | 기본값, 외부 노출 없음 |
| NodePort | 각 Node IP + 고정 포트(30000-32767) | 모든 Node에서 접근 가능 |
| LoadBalancer | 클라우드 LB를 통한 외부 IP | Cloud Provider 필요 |
| ExternalName | 클러스터 외부 DNS로 CNAME 연결 | 외부 서비스 참조용 |
| Headless (clusterIP: None) | 개별 Pod IP 직접 조회 | StatefulSet과 함께 주로 사용 |

### 핵심 비교표: Scheduling 제어

| 구분 | 주체 | 동작 방식 |
|---|---|---|
| nodeSelector | Pod | 지정한 Label을 가진 Node에만 배치 |
| Node Affinity | Pod | nodeSelector보다 유연한 조건(선호/필수) 지정 |
| Taint | Node | 특정 Toleration 없는 Pod를 밀어냄(배치 거부) |
| Toleration | Pod | 특정 Taint를 허용해 배치 가능하게 함 |

### 핵심 비교표: Storage

| 구분 | 데이터 유지 | 특징 |
|---|---|---|
| emptyDir | Pod 삭제 시 사라짐 | 같은 Pod 내 컨테이너 간 임시 공유 |
| hostPath | Node에 유지 (Pod 재스케줄 시 문제) | 특정 Node 자원 접근용 |
| PV / PVC | 볼륨 정책에 따라 유지 | 클러스터 자원으로 스토리지 추상화 |
| StorageClass | Dynamic Provisioning | PVC 생성 시 PV 자동 생성 |

### 핵심 비교표: ConfigMap vs Secret

| 항목 | ConfigMap | Secret |
|---|---|---|
| 용도 | 일반 설정값 | 비밀번호, 토큰 등 민감 정보 |
| 인코딩 | 평문 | Base64 인코딩(암호화 아님) |
| 사용 방법 | env / volume 마운트 | env / volume 마운트 동일 |

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** kubeadm 기반 Kubernetes 클러스터(Master + Worker Node) 설치 환경.

### 클러스터 구성 흐름 요약

```bash
# Master Node
kubeadm init --pod-network-cidr=10.244.0.0/16
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Worker Node
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# 확인
kubectl get nodes
kubectl get pods --all-namespaces
```

---

> 📌 **핵심 요약**
> - Kubernetes 리소스는 Pod(최소 실행 단위) → Controller(복제·관리) → Service/Ingress(네트워크 노출) 순으로 계층을 이룬다
> - Probe는 livenessProbe(재시작 트리거) vs readinessProbe(트래픽 제어)로 목적이 다르다
> - Controller는 워크로드 특성에 따라 선택: 무상태(Deployment), 노드당 1개(DaemonSet), 고정 아이덴티티(StatefulSet), 배치(Job/CronJob)
> - Scheduling은 nodeSelector·Affinity(끌어당김)와 Taint·Toleration(밀어냄)을 조합해 세밀하게 제어한다
> - Storage는 emptyDir/hostPath(단순) → PV·PVC·StorageClass(추상화·Dynamic Provisioning) 순으로 발전한다
> - AutoScaling은 HPA(Pod 수) · VPA(requests 값) · Cluster Autoscaler(Node 수) 3단계로 구분된다
> - 관련: 3. 🏗️ Kubernetes - 아키텍처 개요와 핵심 컴포넌트 · 12. 🎛️ Kubernetes - Deployment · 33. 🚑 Kubernetes - 트러블슈팅 치트시트 · 34. ⚡ Kubernetes - 퀵 레퍼런스
