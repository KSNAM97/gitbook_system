# Kubernetes - Deployment

> **Tag:** #Kubernetes #Deployment #RollingUpdate #ReplicaSet
> **핵심 요약:** Deployment는 Pod 생성, 개수 유지, 무중단 업데이트(Rolling Update), 버전 관리, 롤백을 담당하는 컨트롤러이며, Rolling Update는 maxSurge/maxUnavailable 옵션으로 업데이트 중 여유/위험 수준을 조절한다.

---

## 1. 개요 (Overview)

Deployment는 Pod를 어떻게 만들고, 어떻게 업데이트하고, 문제가 생기면 어떻게 되돌릴지를 쿠버네티스에게 선언적으로 알려주는 컨트롤러다.

Deployment = Pod 생성 + Pod 개수 유지 + 무중단 업데이트 + 버전 관리 + 롤백

### Deployment를 사용하는 이유

**Pod만 사용하면 생기는 문제**

- Pod를 직접 생성하게되면 파드가 죽으면 재시작되지 않고 사라진다.
- 재시작, 개수 유지, 업데이트를 직접 해야 한다.
- 서비스 중 이미지 버전을 바꾸려면 기존 Pod 삭제 → 새 Pod 생성 → 서비스 중단 가능성 발생.
- Deployment는 이 문제를 해결한다.

**Deployment가 해주는 일**

- Pod 개수를 항상 유지.
- 새로운 버전으로 천천히 교체.
- 문제 발생 시 이전 버전으로 복구.
- 그래서 실무에서는 Pod 단독 생성 거의 안 하고 Deployment를 사용한다.

### Deployment의 기본 구조

Deployment는 내부적으로 이런 구조를 가진다.

```
Deployment
└ ReplicaSet
    └ Pod
    └ Pod
    └ Pod
```

즉 Deployment는 직접 Pod를 만들지 않는다. Deployment가 ReplicaSet을 만들고, ReplicaSet이 Pod를 만든다.

### Deployment가 관리하는 핵심 기능

Deployment가 하는 핵심 역할은 5가지다.

1. **Pod 개수 유지** — replicas 값만큼 항상 유지하며, Pod가 죽으면 자동 재생성한다.
2. **버전 관리** — 이미지 변경 시 새로운 ReplicaSet을 생성하고, 이전 ReplicaSet은 기록으로 남는다.
3. **무중단 업데이트 (Rolling Update)** — Pod를 한 번에 다 지우지 않고 하나씩 교체하면서 서비스를 유지한다.
4. **롤백 (Rollback)** — 새 버전에서 문제가 생기면 이전 정상 버전으로 되돌린다.
5. **선언적 관리** — "이 상태가 되길 원한다"만 작성하면 쿠버네티스가 알아서 맞춘다.

---

## 2. Deployment YAML 기본 예제

```yaml
apiVersion: apps/v1
kind: Deployment		# kind가 Deployment인걸 제외하면 나머지는 ReplicaSet과 설정이 동일하다.
metadata:
  name: nginx-deploy

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
      - name: nginx
        image: nginx:1.31
        ports:
        - containerPort: 80
```

- `metadata.name` : Deployment 이름
- `spec.replicas` : 유지할 Pod 개수
- `spec.selector.matchLabels` : 어떤 Pod를 관리할지 기준 라벨
- `spec.template` : 실제 Pod 설계도
- `template.metadata.labels` : selector와 반드시 일치해야 함
- `template.spec.containers` : Pod 안에 들어갈 컨테이너 정의

### template이 중요한 이유

Deployment에서 가장 중요한 부분은 `spec.template`이다. 이 부분은 Pod 설계도이며, Deployment는 template을 보고 Pod를 만든다. template이 바뀌면 새 버전으로 인식한다. 즉 이미지 변경, 환경변수 변경, 포트 변경 = 전부 새로운 ReplicaSet 생성으로 이어진다.

### Deployment 업데이트 흐름

이미지를 변경하면 내부에서 일어나는 일:

1. 기존 ReplicaSet (nginx:1.28.4) 유지
2. 새로운 ReplicaSet (nginx:1.29.04) 생성
3. 새 Pod 1개 생성
4. 기존 Pod 1개 종료
5. 이 과정을 반복
6. 최종적으로 새 버전만 남게된다.
7. 이게 바로 Rolling Update다.

중요 포인트: 서비스 중단 없이 교체하며, Pod를 한 번에 다 없애지 않는다.

### Rollback이 가능한 이유

Deployment는 이전 상태를 기록으로 남긴다. 그래서 최신 버전에 문제가 발생하면 이전 안정 버전으로 즉시 복구 가능하다. 이전 버전의 ReplicaSet이 남아 있기 때문에 가능한 구조다.

### Deployment와 ReplicaSet 차이

| 구분 | ReplicaSet | Deployment |
|---|---|---|
| 역할 | Pod 개수 유지 전용 | ReplicaSet을 관리 |
| 업데이트 | 기능 없음 | 업데이트 전략 제공 |
| 롤백 | 기능 없음 | 롤백 가능 |
| 용도 | - | 운영용 표준 방식 |

정리하면 ReplicaSet은 엔진이고, Deployment는 운전자다.

### 언제 Deployment를 쓰는가

**Deployment를 사용하는 경우**

- 웹 서버
- API 서버
- 백엔드 서비스
- 프론트엔드
- 대부분의 일반 서비스

**Deployment를 쓰지 않는 경우**

- DB (StatefulSet 사용)
- 노드마다 1개씩 필요한 경우 (DaemonSet)

---

## 3. Rolling Update 개념

롤링 업데이트는 서비스를 멈추지 않고 기존 Pod를 새 Pod로 하나씩 교체하는 업데이트 방식이다. 즉, 전체 Pod를 한 번에 삭제하지 않는다.

기존 Pod를 유지한 상태에서 새 Pod를 조금씩 늘리고 오래된 Pod를 조금씩 줄인다. 그래서 무중단 업데이트라고 부른다.

### 왜 롤링 업데이트가 필요한가?

- 롤링 업데이트가 없으면 → 모든 Pod 삭제 → 새 Pod 생성 → 그 사이 서비스 중단 발생.
- 운영 환경에서는 몇 초의 중단도 큰 장애가 된다.
- 롤링 업데이트를 사용하면 사용자는 서비스 중단을 느끼지 못한다. (서버를 켠 상태에서 엔진만 교체)

### 롤링 업데이트는 누가 수행하는가

Pod가 직접 하지 않는다. 사용자가 직접 제어하지도 않는다. Deployment가 자동으로 수행한다. Deployment는 새 ReplicaSet을 생성하고 기존 ReplicaSet은 줄이고 새 ReplicaSet은 늘린다.

### 롤링 업데이트 기본 동작 흐름

Pod가 3개 있는 Deployment에서 이미지를 변경시 내부 동작:

1. 기존 Pod 3개 유지
2. 새 Pod 1개 생성
3. 기존 Pod 1개 종료
4. 새 Pod 2개
5. 기존 Pod 2개
6. 반복
7. 최종적으로 새 Pod 3개만 남음

항상 최소 1개 이상의 Pod는 살아 있으며, 서비스는 계속 응답 가능하다.

**롤링 업데이트를 그림처럼 이해하기**

```
업데이트 전
[ OLD ][ OLD ][ OLD ]

업데이트 중
[ NEW ][ OLD ][ OLD ]
[ NEW ][ NEW ][ OLD ]

업데이트 완료
[ NEW ][ NEW ][ NEW ]
```

이 흐름을 자동으로 관리하는 게 Deployment다.

### 롤링 업데이트가 발생하는 조건

다음 중 하나라도 바뀌면 롤링 업데이트가 발생한다.

- 컨테이너 이미지 변경
- 환경변수 변경
- 포트 변경
- 커맨드 변경
- 볼륨 설정 변경

공통점: `spec.template` 안의 내용이 변경됨. 즉, Deployment는 template이 바뀌면 새 버전으로 인식한다.

---

## 4. 롤링 업데이트 전략 옵션

Deployment에는 롤링 업데이트 속도와 안정성을 조절하는 옵션이 있다. 대표 옵션 2가지는 `maxSurge`, `maxUnavailable`이다. 이 옵션들은 얼마나 여유 있게 업데이트할 것인가를 정한다.

### maxSurge

maxSurge는 기존 Pod 개수보다 초과해서 만들 수 있는 Pod 수다.

예시: `replicas: 3`, `maxSurge: 1`

의미:

- 최대 4개 Pod까지 동시에 존재 가능
- 새 Pod를 먼저 하나 만들고
- 기존 Pod를 나중에 하나 줄임
- 즉, 안전하지만 자원을 조금 더 사용한다.

### maxUnavailable

maxUnavailable은 업데이트 중 사용할 수 없게 되어도 되는 Pod 수다.

예시: `replicas: 3`, `maxUnavailable: 1`

의미:

- 동시에 최대 1개 Pod까지는 내려가도 허용
- 즉, 자원은 적게 쓰지만 조금 더 위험할 수 있다.

**RollingUpdate 설정 예제**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

Pod를 하나 더 만들 수 있고, 동시에 하나까지는 내려도 된다. 가장 일반적인 기본 설정이다.

### 롤링 업데이트 vs 재생성 방식

| 방식 | 동작 | 서비스 영향 |
|---|---|---|
| Recreate | 기존 Pod 전부 삭제 → 새 Pod 전부 생성 | 서비스 중단 발생 |
| RollingUpdate | Pod를 하나씩 교체 | 서비스 유지, 운영 환경에 적합 |

Deployment 기본값은 RollingUpdate다.

**1단계: 새 파드 생성 (maxSurge)**

```
old-pod-1 (v1)  서비스 중
old-pod-2 (v1)  서비스 중
old-pod-3 (v1)  서비스 중
new-pod-1 (v2)  생성됨
```

총 파드: 4. 여기까지는 모든 pod가 동작한다. 서비스 파드 수: 4 (readiness 통과 기준)

**2단계: 기존 파드 1개 종료 (maxUnavailable)**

```
old-pod-2 (v1)  서비스 중
old-pod-3 (v1)  서비스 중
new-pod-1 (v2)  서비스 중
```

old-pod-1이 제거됨. 총 파드: 3, 서비스 파드 수: 3. pod-1이 업데이트 된 게 아니라 old-pod-1이 삭제되고 new-pod-1이 서비스한다.

**3단계: 다시 새 파드 생성**

```
old-pod-2 (v1)  서비스 중
old-pod-3 (v1)  서비스 중
new-pod-1 (v2)  서비스 중
new-pod-2 (v2)  생성됨
```

총 파드: 4

**4단계: 또 기존 파드 1개 종료**

```
old-pod-3 (v1)  서비스 중
new-pod-1 (v2)  서비스 중
new-pod-2 (v2)  서비스 중
```

**5단계: 반복**

```
new-pod-1 (v2)  서비스 중
new-pod-2 (v2)  서비스 중
new-pod-3 (v2)  서비스 중
```

---

## 5. 실습 — Deployment 생성/확인/삭제

```
[root@k8s-master ~]# vi deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx

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
      - name: nginx-container
        image: nginx:1.31
```

```
[root@k8s-master ~]# kubectl  apply  -f  deploy-nginx.yaml  --dry-run=client
deployment.apps/deploy-nginx created (dry run)


[root@k8s-master ~]# kubectl  apply  -f  deploy-nginx.yaml
deployment.apps/deploy-nginx created
```

```
[root@k8s-master ~]# kubectl  get  deployments.apps
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
deploy-nginx   3/3         3                    3                  32s
```

- **READY : 3/3** — 정상 준비된 Pod 수 / 원하는 Pod 수. 현재 3개를 원하고, 3개 모두 정상 준비 완료된 상태.
- **UP-TO-DATE : 3** — 현재 Deployment에 설정된 최신 설정(template)으로 실행 중인 Pod가 3개라는 의미. 예를 들어 이미지를 변경해서 롤링 업데이트 중이면 일시적으로 READY와 숫자가 다를 수 있다.
- **AVAILABLE : 3** — 실제로 서비스 가능한 상태의 Pod가 3개라는 의미. 즉, 트래픽을 받을 수 있는 Pod가 3개.

```
[root@k8s-master ~]# kubectl  get  deployments.apps  -o  wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS        IMAGES       SELECTOR
deploy-nginx   3/3         3                    3                  5m25s   nginx-container      nginx:1.31    app=web
```

```
[root@k8s-master ~]# kubectl  get  pods
NAME                            		READY   STATUS    RESTARTS   AGE
deploy-nginx-5796ddf486-7cgpx   	1/1        Running     0                 3m31s
deploy-nginx-5796ddf486-fz9mg   	1/1        Running     0                 3m31s
deploy-nginx-5796ddf486-xlb2m  	1/1        Running     0                 3m31s
```

- `deploy-nginx` : Deployment
- `deploy-nginx-5796ddf486` : ReplicaSet
- `deploy-nginx-5796ddf486-7cgpx`, `-fz9mg`, `-xlb2m` : Pod

```
[root@k8s-master ~]# kubectl  delete  deployments.apps deploy-nginx
deployment.apps "deploy-nginx" deleted from default namespace


[root@k8s-master ~]# kubectl  get  pods
No resources found in default namespace.
```

---

## 6. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `kubectl get deployments`의 READY가 목표값과 다르면 새 Pod가 아직 Ready 상태가 아니라는 뜻이므로 `kubectl get pods`와 `kubectl describe pods <이름>`으로 원인을 확인한다.
- template(spec.template)이 바뀌지 않으면 롤링 업데이트가 발생하지 않는다 — 이미지/환경변수/포트/커맨드/볼륨 중 아무것도 안 바뀌면 새 ReplicaSet이 생성되지 않는다.
- maxSurge와 maxUnavailable은 서로 다른 축이다. maxSurge는 "얼마나 더 만들 수 있는가"(자원 사용량), maxUnavailable은 "얼마나 내려가도 되는가"(가용성 리스크)를 의미하므로 두 값을 함께 보고 튜닝해야 한다.
- Recreate 전략은 서비스 중단이 발생하므로 운영 환경의 무중단 서비스에는 적합하지 않다 — 반드시 다운타임을 감수할 수 있는 경우에만 사용한다.

---

>  **핵심 요약**
> - Deployment = Pod 생성 + 개수 유지 + 무중단 업데이트 + 버전 관리 + 롤백을 담당하는 컨트롤러이며, 내부적으로 Deployment → ReplicaSet → Pod 구조를 가진다
> - spec.template이 변경되면(이미지/환경변수/포트/커맨드/볼륨) 새로운 ReplicaSet이 생성되며 이것이 Rolling Update를 유발한다
> - Rolling Update는 maxSurge(추가로 만들 수 있는 Pod 수)와 maxUnavailable(내려가도 되는 Pod 수)로 여유/위험 수준을 조절한다
> - Recreate는 전체 Pod를 삭제 후 재생성해 서비스 중단이 발생하지만, RollingUpdate는 하나씩 교체해 서비스를 유지한다
> - 관련: 13.  Kubernetes - Rollout·Rollback 실습 · 2.  Kubernetes - Pod 생성
