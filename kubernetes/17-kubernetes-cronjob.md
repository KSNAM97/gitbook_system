# ⏰ Kubernetes - CronJob

> **Tag:** #Kubernetes #Controller #CronJob #배치작업 #부트캠프
> **핵심 요약:** 정기 작업을 예약 실행하는 CronJob의 동작 구조, cron 스케줄 문법, concurrencyPolicy(Allow/Forbid), HTTP 헬스체크 자동화 실습

---

## 1. 📖 개요 (Overview)

실무에서 배치 작업은 대부분 정기적으로 실행된다 — 매일 새벽 2시 DB 백업, 10분마다 로그 정리, 매주 통계 집계, 매일 캐시 초기화 등. 이 작업들의 공통점은 항상 실행 중일 필요가 없고, 사용자의 요청을 기다리는 서버 프로그램이 아니며, 정해진 시간에 실행되고 작업을 수행한 뒤 종료된다는 것이다.

리눅스에서는 이러한 정기적인 작업을 cron으로 처리한다. 쿠버네티스에서는 cron과 같은 역할을 하는 전용 리소스가 CronJob이다. CronJob은 지정된 일정(Schedule)에 따라 Job을 자동으로 생성하는 컨트롤러다.

중요한 점은 CronJob이 실제 작업을 직접 수행하는 것이 아니라는 것이다. CronJob은:
- 백업 작업을 직접 수행하지 않는다.
- 스크립트를 직접 실행하지 않는다.
- 컨테이너를 직접 실행하지 않는다.
- Pod를 직접 관리하여 작업을 수행하는 역할도 하지 않는다.

CronJob이 하는 일은 딱 하나다 — "지금 시간이 됐으니까 Job 하나 만들어야겠다." 실제 작업은 항상 Job이 수행한다.

**실행 구조**
```
1) CronJob → 2) 정해진 시간이 됨 → 3) Job 생성 → 4) Pod 생성 → 5) Container 실행
→ 6) 작업 수행 → 7) Container 종료 → 8) Pod 완료 → 9) Job 완료
```

따라서 CronJob은 "언제 실행할 것인가"를 관리하고, Job은 "무슨 작업을 수행할 것인가"를 관리한다.

**CronJob의 전체 동작 흐름 (매일 새벽 2시에 백업하는 CronJob 예시)**

1. 관리자가 CronJob을 생성한다.
2. 쿠버네티스는 이 CronJob을 기억하고 대기한다. (아직 아무 작업도 실행하지 않는다)
3. 시간이 새벽 2시가 된다.
4. CronJob Controller가 판단한다. (실행 시간이 됐다.)
5. CronJob Controller가 새 Job 하나를 생성한다.
6. Job Controller가 Pod를 생성한다.
7. Pod 안 컨테이너에서 백업 스크립트가 실행된다.
8. 작업이 끝나면 `exit 0`(성공) 또는 `exit 1`(실패)로 종료된다.
9. Job은 성공 또는 실패 상태로 남는다.

다음 날 새벽 2시가 되면 또 다른 새로운 Job이 생성된다 — CronJob은 매번 같은 Job을 재사용하지 않고, 항상 새로운 Job을 생성한다.

**CronJob과 Job의 차이**
- Job: 지금 당장 한 번 실행 (실행 담당)
- CronJob: 이 Job을 언제 실행할지 예약 (시간 담당)

배치 작업 로직은 Job에 있고, 정기 실행은 CronJob이 관리한다.

---

## 2. 🛠️ CronJob 스케줄 문법과 주요 옵션

CronJob은 리눅스 cron과 완전히 같은 형식을 쓴다.

```
Cronjob Schedule: "30 2 1 * *"
- Minutes (from 0 to 59)
- Hours (from 0 to 23)
- Day of the month (from 1 to 31)
- Month (from 1 to 12)
- Day of the week (from 0 to 6)
```

기본 형식: `분 시 일 월 요일`

- `0 2 * * *` — 매일 새벽 2시
- `*/5 * * * *` — 5분마다
- `0 0 * * 0` — 매주 일요일 0시

쿠버네티스 CronJob은 서버 시간 기준이므로 타임존 설정이 매우 중요하다.

**CronJob 주요 옵션**

1. **schedule** — 언제 실행할지 결정. 예: `schedule: "*/5 * * * *"` → 5분마다 Job 생성
2. **jobTemplate** — CronJob이 만들 Job의 템플릿. CronJob 안에는 Job 정의가 들어 있다.
3. **successfulJobsHistoryLimit** — 예: `successfulJobsHistoryLimit: 3` → 성공한 Job을 최대 3개까지 보관
4. **failedJobsHistoryLimit** — 예: `failedJobsHistoryLimit: 2` → 실패한 Job을 최대 2개까지 보관

이 옵션이 없으면 Job이 계속 쌓여서 관리가 어려워질 수 있다.

**Job vs CronJob 구조 비교**
```yaml
# Job
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
  template:
    spec:
      containers:
      - name: hello-busybox
        image: busybox
        args:
        - /bin/sh
        - "-c"
        - "echo Hello; sleep 5; echo Bye"
      restartPolicy: Never
```
```yaml
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-definition
spec:
  schedule: "0 3 1 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello-busybox
            image: busybox
            args:
            - /bin/sh
            - -c
            - "echo Hello; sleep 5; echo Bye"
          restartPolicy: Never
```

---

## 3. 🧪 실습: CronJob 기본 생성과 History 관리

```yaml
[root@k8s-master ~]# vi cronjob-exam.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-exam

spec:
  schedule: "* * * * *"
  startingDeadlineSeconds: 500
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2

  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello-busybox
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello; sleep 10; echo Bye; date
          restartPolicy: Never
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  cronjob-exam.yaml
cronjob.batch/cronjob-exam created

[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME                                  READY	STATUS    	RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
cronjob-exam-29786643-xnhrp   0/1     	Pending   		0                 0s    <none>   <none>   <none>           <none>
cronjob-exam-29786643-xnhrp   1/1     	Running             	0                 3s    10.244.2.17   k8s-worker2   <none>           <none>
cronjob-exam-29786643-xnhrp   0/1     	Completed           	0                 13s   10.244.2.17   k8s-worker2   <none>           <none>
cronjob-exam-29786644-g5dp8   0/1     	Pending             	0                 0s    <none>        <none>        <none>           <none>
cronjob-exam-29786644-g5dp8   1/1     	Running             	0                 3s    10.244.2.18   k8s-worker2   <none>           <none>
```

```bash
[root@k8s-master ~]# kubectl  get  cronjobs.batch
NAME            SCHEDULE    TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob-exam   * * * * *       <none>        False          0           37s                       8m25s
```

```bash
# 성공적으로 완료된 Job을 최대 3개까지 보관한다
[root@k8s-master ~]# kubectl  get  jobs.batch
NAME                         STATUS     COMPLETIONS   DURATION   AGE
cronjob-exam-29786648   Complete   1/1                    15s              2m44s
cronjob-exam-29786649   Complete   1/1                    15s              104s
cronjob-exam-29786650   Complete   1/1                    15s              44s

[root@k8s-master ~]# kubectl get pods
NAME                          	   READY   STATUS      RESTARTS   AGE
cronjob-exam-29786655-7kxj9    0/1        Completed    0                 2m22s
cronjob-exam-29786656-74nxc   0/1        Completed    0                 82s
cronjob-exam-29786657-8pzhx   0/1        Completed    0                 22s

[root@k8s-master ~]# kubectl  logs cronjob-exam-29786655-7kxj9
Thu Aug 20 04:15:02 UTC 2026
Hello
Bye
Thu Aug 20 04:15:12 UTC 2026

[root@k8s-master ~]# kubectl  delete  cronjobs cronjob-exam
cronjob.batch "cronjob-exam" deleted from default namespace
```

---

## 4. 🧪 concurrencyPolicy 비교 실습: Forbid vs Allow

### concurrencyPolicy: Forbid

- 이전 Job이 아직 실행 중이면 다음 스케줄의 Job은 아예 실행하지 않음
- 항상 1개 Job만 실행 (중복 실행 완전 차단)
- 데이터 무결성 보장에 유리, 파일 이동/삭제, 상태 변경 작업에 적합

**동작 개념**
```
03:00 Job A 시작 (실행 중)
03:01 스케줄 도착 : 실행 안 함
03:02 스케줄 도착 : 실행 안 함
03:05 Job A 종료
03:06 다음 스케줄 : Job B 시작
```

```yaml
[root@k8s-master ~]# vi cronjob-exam.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-exam

spec:
  schedule: "* * * * *"
  startingDeadlineSeconds: 500
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2

  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello-busybox
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello; sleep 100; echo Bye; date
          restartPolicy: Never
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  cronjob-exam.yaml
cronjob.batch/cronjob-exam created

# 아직 Pod가 실행중이므로 상태가 Running으로 확인
[root@k8s-master ~]# kubectl  get  pods
NAME                          	  READY   STATUS    RESTARTS   AGE
cronjob-exam-29786736-z84jv	  1/1         Running    0                 31s

# 작업이 완료되었기때문에 Completed 상태가 된다.
[root@k8s-master ~]# kubectl  get  pods
NAME                          	  READY   STATUS      RESTARTS   AGE
cronjob-exam-29786736-z84jv   0/1        Completed   0                  2m41s
cronjob-exam-29786737-gr978   1/1        Running     0                   57s

[root@k8s-master ~]# kubectl  logs  cronjob-exam-29786736-z84jv
Thu Aug 20 05:36:02 UTC 2026
Hello
Bye
Thu Aug 20 05:37:42 UTC 2026
```
sleep 100초짜리 작업임에도 매분(`* * * * *`) 스케줄이 도착할 때마다 새 Job이 쌓이지 않고, 이전 Job이 끝난 뒤에야 다음 Job이 시작된다 — 정확히 1개씩 순차 실행된다.

### concurrencyPolicy: Allow

- default로 적용
- 이전 Job이 아직 실행 중이어도 스케줄 시간이 되면 새 Job을 또 생성
- 동시에 여러 Job 실행 가능
- 로그 수집, 통계 집계, 단순 스크립트 등 서로 영향을 주지 않는 배치에 적합

**동작 개념**
```
03:00 Job A 시작 (아직 실행 중)
03:01 Job B 시작 (아직 실행 중)
03:02 Job C 시작
```

```yaml
[root@k8s-master ~]# vi cronjob-exam.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-exam

spec:
  schedule: "* * * * *"
  startingDeadlineSeconds: 500
#  concurrencyPolicy: Forbid		# 주석 처리
  concurrencyPolicy: Allow		# 추가 설정
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2

  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello-busybox
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello; sleep 100; echo Bye; date
          restartPolicy: Never
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  cronjob-exam.yaml
cronjob.batch/cronjob-exam created

# Allow이기 때문에 첫 번째 Job이 실행중인데도 두 번째 Job도 생성된다.
[root@k8s-master ~]# kubectl get pods
NAME                          	   READY   STATUS    RESTARTS   AGE
cronjob-exam-29786750-xjmxr   1/1         Running     0                 64s	# 첫 번째 Job (Job 실행중)
cronjob-exam-29786751-btsg6    1/1         Running     0                 4s		# 두 번째 Job
```
첫번째 작업이 Running 중인데도 새로운 Pod(ContainerCreating)가 생성된다.

---

## 5. 🧪 실습: CronJob 기반 HTTP 헬스 체크 자동화

내부 서비스 HTTP 헬스 체크를 CronJob으로 자동화하고, 실패 Job을 근거로 장애 징후를 판단한다. CronJob이 만든 Job이 성공/실패로 명확히 갈리는 흐름을 만들어, 서비스가 정상일 때는 성공, 비정상일 때는 실패가 나는 것을 직접 확인한다.

**Deployment, Service 생성**
```yaml
[root@k8s-master ~]# vi job-deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hc-nginx-ctr
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hc-nginx
  template:
    metadata:
      labels:
        app: hc-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.31
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hc-nginx-svc
spec:
  selector:
    app: hc-nginx		# app: hc-nginx  label을 갖은 Pod를 하나의 Service로 묶어서 단일 진입점을 제공한다.
  ports:
  - port: 80
    targetPort: 80
```

```bash
[root@k8s-master ~]# kubectl  apply  -f  job-deploy-nginx.yaml
deployment.apps/hc-nginx-ctr created
service/hc-nginx-svc created
```

**HTTP 헬스체크 CronJob**
```yaml
[root@k8s-master ~]# vi cron-http-check.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-http-check
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          containers:
          - name: http-check
            image: curlimages/curl:8.5.0
            command:
            - sh
            - -c
            - |
              curl -sSf http://hc-nginx-svc.default.svc.cluster.local >/dev/null
          restartPolicy: Never
```

curl 옵션 설명:
- `-s` (silent) — 출력과 진행 표시를 숨김. 정상일 때 아무 로그도 안 남김
- `-S` (show-error) — `-s`와 같이 쓸 때 실패 에러 메시지만 출력
- `-f` (fail) — HTTP 응답 코드가 4xx/5xx면 실패 처리, exit code를 0이 아닌 값으로 반환
- `>/dev/null` — 정상 응답 바디(HTML 등)를 버리고 상태만 체크

```bash
[root@k8s-master ~]# kubectl  apply  -f  cron-http-check.yaml
cronjob.batch/cron-http-check created

# http server로부터 200OK 정보를 받기때문에 Completed로 확인
[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS      RESTARTS   AGE
cron-http-check-29786775-h27fw   	0/1        Completed    0                 95s
cron-http-check-29786776-gl8m7   	0/1        Completed    0                 35s
hc-nginx-ctr-7699b6f665-9fj9p    	1/1        Running       0                 13m
```

**강제로 Fail 발생시키기**
```bash
[root@k8s-master ~]# kubectl  scale  deployment hc-nginx-ctr  --replicas=0

[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
cron-http-check-29786780-6m5s2   	0/1     Pending     0          0s      <none>        <none>        <none>           <none>
cron-http-check-29786780-6m5s2   	0/1     ContainerCreating   0          0s      <none>        k8s-worker2   <none>           <none>
cron-http-check-29786780-6m5s2   	0/1     Error               0          7s      10.244.2.44   k8s-worker2   <none>           <none>

[root@k8s-master ~]# kubectl  get  pods
NAME                             		READY   STATUS              RESTARTS   AGE
cron-http-check-29786777-kn7k9   	0/1        Completed             0                4m
cron-http-check-29786778-4zwlw   	0/1        Completed             0                3m
cron-http-check-29786779-jgzfz   	0/1        Completed             0                2m
cron-http-check-29786780-6m5s2   	0/1        Error                   0                 60s
```
백엔드 Pod를 0개로 줄이면 Service가 트래픽을 전달할 대상이 없어 curl이 실패(4xx/5xx 또는 연결 실패)하고, `-f` 옵션 때문에 non-zero exit code로 종료되어 Job이 `Error` 상태로 남는다 — 이 실패 이력 자체가 장애 감지 신호로 활용된다.

---

## 6. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`을 설정하지 않으면 완료된 Job/Pod가 계속 쌓여 클러스터 리소스를 낭비하므로 반드시 설정하는 것이 좋다.
- `concurrencyPolicy`의 기본값은 Allow이므로, 중복 실행이 위험한 작업(파일 이동/삭제, 상태 변경 등)에는 반드시 Forbid로 명시해야 한다.
- CronJob 스케줄은 클러스터(서버) 시간 기준이므로, 타임존이 다른 환경에서는 `timeZone` 필드(또는 서버 TZ 설정)를 반드시 확인해야 한다.
- `startingDeadlineSeconds`는 스케줄된 시간을 놓쳤을 때(예: 컨트롤러 다운타임) 얼마나 지난 시점까지는 여전히 Job을 시작할지 지정하는 값이다.
- Job이 `Error`/`Completed`로 반복 남는 패턴을 `kubectl get jobs.batch`/`kubectl logs`로 주기적으로 확인하면, CronJob을 간단한 헬스체크·모니터링 도구로도 활용할 수 있다.

---

> 📌 **핵심 요약**
> - CronJob은 "언제 실행할지"만 담당하고 실제 작업 실행은 Job이 담당한다 — CronJob → Job → Pod → Container 순으로 동작한다
> - 스케줄은 리눅스 cron 문법(`분 시 일 월 요일`)을 그대로 사용하며, `concurrencyPolicy`(Allow/Forbid)로 중복 실행 허용 여부를 제어한다
> - `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`으로 이력 보관 개수를 제한해야 Job/Pod가 무한히 쌓이는 것을 막을 수 있다
> - curl `-sSf` 옵션과 CronJob을 조합하면 별도 모니터링 도구 없이도 정기 HTTP 헬스체크를 구현할 수 있다
> - 관련: 16. ⚙️ Kubernetes - Job · 2. 📦 Kubernetes - Pod 생성 · 10. 🎛️ Kubernetes - Controller 개념과 ReplicationController
