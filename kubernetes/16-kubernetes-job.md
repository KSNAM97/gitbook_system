# Kubernetes - Job

> **Tag:** #Kubernetes #Job #Controller
> **핵심 요약:** 한 번 실행하고 끝나는 작업을 위한 Job Controller의 개념과 completions/parallelism/backoffLimit/restartPolicy/activeDeadlineSeconds 실습 정리

---

## 1. 개요 (Overview)

### 왜 Pod를 직접 `kubectl run`으로 실행하면 계속 재시작되는가

다음 실습으로 시작한다.

```
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch


[root@k8s-master ~]# kubectl  run  jobpod  --image=centos:7  --command  sleep 5


[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME     READY   STATUS    	RESTARTS	AGE   IP       	NODE     		NOMINATED NODE   READINESS GATES
jobpod     0/1        Pending   		0                	0s    <none>   	<none>   		<none>           	<none>
jobpod     0/1        Pending   		0          		0s    <none>   	k8s-worker2   	<none>           	<none>
jobpod     0/1        ContainerCreating	0          		0s    <none>   	k8s-worker2   	<none>           	<none>
jobpod     1/1        Running             	0          		1s    10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     0/1        Completed           	0          		6s    10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     1/1        Running             	1 (2s ago)   	7s    10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     0/1        Completed           	1 (7s ago)   	12s   10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     0/1        CrashLoopBackOff   1 (12s ago)   	23s   10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     1/1        Running             	2 (12s ago)   	23s   10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     0/1        Completed           	2 (17s ago)   	28s   10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     0/1        CrashLoopBackOff   2 (26s ago)   	54s   10.244.2.3   	k8s-worker2   	<none>           	<none>
jobpod     1/1        Running             	3 (26s ago)   	54s   10.244.2.3   	k8s-worker2   	<none>           	<none>



[root@k8s-master ~]# kubectl  delete  pods jobpod
pod "jobpod" deleted from default namespace
```

이 실습은 "쿠버네티스는 프로세스를 관리하는 게 아니라 원하는 상태(desired state)를 관리한다."는 것을 보여주기 위한 예제다.

사용자가 쿠버네티스에게 한 말은 딱 하나다.
- jobpod라는 Pod는 존재해야 한다.
- 쿠버네티스는 이 말을 이렇게 해석한다.
- 그럼 jobpod는 항상 살아 있어야겠네

**왜 Pod가 끝났는데 다시 살아나는가**

실행한 명령: `kubectl run jobpod --image=centos:7 --command sleep 5`

이 명령의 의미는 다음과 같다.
- jobpod라는 Pod를 하나 만들고 그 Pod 안에서 sleep 5를 실행한다.
- `restartPolicy: Always` = 컨테이너가 종료되면 같은 Pod 안에서 다시 실행하라
- sleep 5는 5초 후 정상 종료(exit 0)한다. 하지만 쿠버네티스 입장에서는 정상 종료든 비정상 종료든 상관없이 컨테이너가 멈췄다는 사실만 중요하다. 그래서 kubelet은 이렇게 행동한다 → 컨테이너가 멈췄네? 그럼 다시 실행해서 원하는 상태를 맞춰야지.
- `Completed → Running → Completed`를 계속 반복한다.

이건 버그가 아니라 쿠버네티스가 충실하게 일하고 있다는 증거다. Pod는 존재해야 하고 컨테이너는 계속 실행 상태여야 한다. 그러니 종료되면 다시 실행한다.

이게 바로 "쿠버네티스는 작업이 끝났는지를 보지 않는다."는 특징이다. 쿠버네티스는 기본적으로 서비스를 오래 유지하는 플랫폼이다.

`CrashLoopBackOff`는 에러가 발생했다는 뜻이 아니라 너무 짧은 시간 안에 컨테이너가 계속 종료되고 있다는 뜻이다. kubelet은 이런 반복 종료를 문제 상황으로 판단해, 재시작 간격을 점차 늘려가며 대기 후 재시작한다. 이 동작이 CrashLoopBackOff이다.

그런데 만약 이 작업을 한 번만 실행하고 끝내고 싶다면 어떻게 해야 할까?

### Job Controller

Job Controller는 한 번 실행하고 끝나는 작업을 쿠버네티스에서 실행하기 위한 컨트롤러다.

쿠버네티스를 배우다 보면 가장 먼저 접하는 개념이 Deployment다. Deployment는 웹 서버처럼 항상 살아 있어야 하는 서비스를 관리하기 위한 컨트롤러다.

하지만 실무에서는 항상 떠 있을 필요가 없는 작업도 많다.
- DB 마이그레이션을 한 번 실행하는 작업
- 하루의 로그를 모아서 정리하는 작업
- 통계 데이터를 계산해서 결과 파일을 만드는 작업
- 백업 스크립트를 한 번 실행하는 작업

이 작업들의 공통점은 계속 실행될 필요가 없다는 것이다. 성공하면 끝이다(실패 시 다시 시도는 필요). 이런 작업을 Deployment로 실행하면 문제가 생긴다. Deployment는 Pod가 종료되면 장애라고 판단하고 다시 띄우기 때문이다. 그래서 쿠버네티스에는 한 번 실행하고 끝나는 작업을 위한 전용 컨트롤러인 Job Controller가 존재한다.

### Job과 Pod의 역할 관계

Job을 이해할 때 가장 중요한 관점은 Job은 일을 하지 않는다는 점이다. 실제 작업을 수행하는 주체는 Pod이고 Job Controller는 그 Pod를 관리하는 관리자이다.

사람에 비유하면:
- Pod: 현장에서 실제 일을 하는 작업자
- Job: 작업자가 일을 제대로 끝냈는지 확인하는 관리자

1. 작업을 수행할 Pod를 생성
2. Pod가 성공했는지 실패했는지 감시
3. 실패하면 다시 Pod를 생성해서 재시도
4. 성공하면 작업을 종료 처리

즉, Job은 작업 실행 책임자이지 작업 실행자는 아니다.

### Job Controller의 성공과 실패 판단

Job이 성공했는지 실패했는지를 판단하는 기준은 매우 단순하다. 컨테이너의 종료 코드(exit code)를 사용한다. 리눅스에서 프로그램은 종료될 때 숫자를 하나 남긴다.
- exit 0 : 정상 종료
- exit 1 이상 : 비정상 종료
- Job은 오직 종료 코드(exit code)만 본다.

컨테이너가 exit 0으로 종료되면 성공, exit 0이 아니면 실패다.
- 로그 내용은 보지 않는다.
- 에러 메시지가 있었는지도 보지 않는다.
- 출력이 많았는지 적었는지도 상관없다.
- 0으로 끝났냐, 아니냐 이것만으로 판단한다.

그래서 Job에서 실행되는 스크립트나 프로그램은 실패 시 반드시 exit 1 같은 실패 코드로 종료해야 한다.

### Job의 전체 동작 흐름 (시간 순서로 이해)

Job 하나가 실행될 때 실제 내부에서는 다음 순서로 움직인다.

1단계: 사용자가 Job을 생성한다.
2단계: Job Controller가 이 Job을 수행할 Pod가 필요하다고 판단한다.
3단계: Job Controller가 Pod를 하나 생성한다.
4단계: Pod 안의 컨테이너가 실행된다. (배치 스크립트 실행 / 프로그램 수행 / 데이터 처리 작업 수행)
5단계: 컨테이너가 종료된다.
6단계: Job Controller가 종료 코드를 확인한다. exit 0이면 Job은 성공 상태가 된다. exit 0이 아니면 Job은 아직 성공하지 못했다고 판단하고 새로운 Pod를 생성해 작업을 다시 시도한다.

이 과정을 성공할 때까지 반복하는 것이 Job의 핵심 동작이다.

### Job의 주요 개념 옵션

Job에는 자주 등장하는 핵심 개념이 있다.

- **completions** — 이 Job이 총 몇 번 성공해야 끝나는가. 기본적인 Job은 completions 1. 여러 번 성공해야 하는 반복 작업도 가능.
- **parallelism** — 동시에 몇 개의 Pod를 실행할 것인가. 병렬 처리가 필요한 작업에서 사용.
- **backoffLimit** — 실패를 몇 번까지 허용할 것인가. 무한 재시도를 막기 위한 안전장치.

이 옵션들을 조합하면 단일 실행 Job, 병렬 Job, 반복 성공 Job 같은 다양한 작업 패턴을 만들 수 있다.

### Job이 주로 쓰이는 사례

Job은 다음과 같은 상황에서 거의 필수적으로 사용된다.
- DB 백업 작업
- 데이터 정리 및 배치 처리
- 통계 집계
- 캐시 초기화
- 테스트 자동 실행
- 마이그레이션 스크립트 실행

공통 특징: 항상 실행될 필요 없음, 결과가 성공/실패로 명확, 실패 시 재시도 필요.

### Job과 CronJob의 관계

여기서 자연스럽게 나오는 개념이 CronJob이다.
- Job: 한 번 실행
- CronJob: 정해진 시간마다 Job을 생성

즉, `CronJob → Job → Pod` 이 순서로 동작한다. CronJob은 직접 일을 하지 않고 Job을 정기적으로 만들어주는 역할만 한다.

### Job을 쓰면 안 되는 경우

다음과 같은 서비스에는 Job을 쓰면 안 된다.
- 웹 서버
- API 서버
- 지속적으로 요청을 받아야 하는 서비스 — Deployment를 사용해야 한다.

Job은 반드시 끝이 있는 작업에만 사용해야 한다.

---

## 2. 실습

### 기본 Job 실행 (completions/parallelism/backoffLimit 주석 처리)

```
[root@k8s-master ~]# vi job-exam.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
# completions: 5
# parallelism: 2
# activeDeadlineSeconds: 5
  template:
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]
        args:
        - "-c"
        - "date; echo 'Hello World'; sleep 20; date; echo 'Bye'"
      restartPolicy: Never
# restartPolicy: OnFailure
# backoffLimit: 3
```

- completions: 이 Job이 총 몇 번 성공해야 끝나는가. 기본적인 Job은 completions 1. 여러 번 성공해야 하는 반복 작업도 가능.
- parallelism: 동시에 몇 개의 Pod를 실행할 것인가. 병렬 처리가 필요한 작업에서 사용.
- backoffLimit: 실패를 몇 번까지 허용할 것인가. 무한 재시도를 막기 위한 안전장치.

```
[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml
job.batch/centos-job created


[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME               	READY   	STATUS    	RESTARTS   AGE   IP       	  NODE     	NOMINATED NODE   READINESS GATES
centos-job-vssh5   	0/1     	Pending   		0                 0s    <none>   	  <none>   	<none>                    <none>
centos-job-vssh5   	0/1     	Pending   		0                 0s    <none>   	  k8s-worker2   	<none>                    <none>
centos-job-vssh5   	0/1     	ContainerCreating	0                 0s    <none>   	  k8s-worker2   	<none>                    <none>
centos-job-vssh5   	1/1     	Running             	0                 1s    10.244.2.5	  k8s-worker2   	<none>                    <none>
centos-job-vssh5   	0/1     	Completed           	0                 21s   10.244.2.5	  k8s-worker2   	<none>                    <none>
centos-job-vssh5   	0/1     	Completed           	0                 22s   10.244.2.5	  k8s-worker2   	<none>                    <none>
centos-job-vssh5   	0/1     	Completed           	0                 23s   10.244.2.5	  k8s-worker2   	<none>                    <none>


[root@k8s-master ~]# kubectl  logs  job/centos-job
Thu Aug 20 01:45:00 UTC 2026
Hello World
Bye
```

### sleep 시간을 60초로 늘려서 Pod를 강제 삭제해보기

```
[root@k8s-master ~]# vi job-exam.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
# completions: 5
# parallelism: 2
# activeDeadlineSeconds: 5
  template:
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]
        args:
        - "-c"
        - "date; echo 'Hello World'; sleep 60; date; echo 'Bye'"		# 60초로 수정
      restartPolicy: Never
# restartPolicy: OnFailure
# backoffLimit: 3


[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml
job.batch/centos-job created


[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME               	READY   	STATUS    	RESTARTS   AGE   IP       	  NODE     	NOMINATED NODE   READINESS GATES
centos-job-ttw2p   	0/1     	Pending   		0                 0s    <none>   	  <none>   	<none>                    <none>
centos-job-ttw2p   	0/1     	Pending   		0                 0s    <none>   	  k8s-worker2   	<none>                    <none>
centos-job-ttw2p   	0/1     	ContainerCreating	0                 0s    <none>   	  k8s-worker2   	<none>                    <none>
centos-job-ttw2p   	1/1     	Running             	0                 1s    10.244.2.5	  k8s-worker2   	<none>                    <none>


[root@k8s-master ~]# kubectl   delete   pods  centos-job-ttw2p
pod "centos-job-ttw2p" deleted from default namespace


NAME               	READY   	STATUS    	RESTARTS   AGE 	 IP       	  NODE     	NOMINATED NODE   READINESS GATES
centos-job-ttw2p   	0/1     	Pending   		0                 0s    	<none>   	  <none>   	<none>                    <none>
centos-job-ttw2p   	0/1     	Pending   		0                 0s    	<none>   	  k8s-worker2   	<none>                    <none>
centos-job-ttw2p   	0/1     	ContainerCreating	0                 0s    	<none>   	  k8s-worker2   	<none>                    <none>
centos-job-ttw2p   	1/1     	Running             	0                 1s    	10.244.2.5	  k8s-worker2   	<none>                    <none>

centos-job-ttw2p  	1/1     	Terminating         	0                 17s   	10.244.2.7	  k8s-worker2   	<none>                    <none>
centos-job-ttw2p 	1/1     	Terminating         	0                 17s   	10.244.2.7	  k8s-worker2   	<none>                    <none>
centos-job-ttw2p  	1/1     	Terminating         	0                 18s   	10.244.2.7	  k8s-worker2   	<none>                    <none>
centos-job-x8jxz  	0/1     	Pending             	0                 0s    	<none>  	  <none>        	<none>                    <none>
centos-job-x8jxz	0/1     	Pending             	0                 0s    	<none>  	  k8s-worker1   	<none>                    <none>
centos-job-x8jxz 	0/1     	ContainerCreating	0                 0s    	<none>  	  k8s-worker1   	<none>                    <none>
centos-job-x8jxz 	1/1     	Running             	0                 1s    	10.244.1.4	  k8s-worker1   	<none>                    <none>
centos-job-x8jxz   	0/1     	Completed           	0                 60s   	10.244.1.4   k8s-worker1   	<none>                    <none>
centos-job-x8jxz   	0/1     	Completed           	0                 61s   	10.244.1.4   k8s-worker1   	<none>                    <none>
centos-job-x8jxz   	0/1     	Completed           	0                 62s   	10.244.1.4   k8s-worker1   	<none>                    <none>


[root@k8s-master ~]# kubectl  get  pods
NAME               	READY   STATUS      RESTARTS   AGE
centos-job-x8jxz   	0/1        Completed    0                 6m20s


[root@k8s-master ~]# kubectl  get  job
NAME         STATUS     COMPLETIONS   DURATION   AGE
centos-job   Complete    1/1                    89s              7m10s
```

- Complete: job이 정상적으로 작업을 완료한 상태
- 1/1: 목표 작업 1개 완료 작업 1개
- DURATION: job이 생성되어 작업을 완료하는데 걸린 시간

```
[root@k8s-master ~]# kubectl  delete  pods centos-job-x8jxz
pod "centos-job-x8jxz" deleted from default namespace


[root@k8s-master ~]# kubectl  get  pods
No resources found in default namespace.
```

Job은 성공 횟수(= completions)만 채우면 역할이 끝난다. Job의 completions의 default값은 1이기 때문에 성공 Pod 1개가 exit 0으로 끝난다. Job 상태가 Complete(1/1) 되고 그 순간부터 Job 컨트롤러는 더 만들 이유가 없다고 판단한다.

### pod만 삭제 시 job controller는 그대로 동작한다

```
[root@k8s-master ~]# kubectl  get  job
NAME         STATUS     COMPLETIONS   DURATION   AGE
centos-job   Complete   1/1           89s        12m


[root@k8s-master ~]# kubectl  delete jobs.batch centos-job
job.batch "centos-job" deleted from default namespace
```

### completions=3, restartPolicy: OnFailure, backoffLimit=3

```
[root@k8s-master ~]# vi job-exam.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
  completions: 3			# 주석 삭제 후 3으로 수정
# parallelism: 2
# activeDeadlineSeconds: 5
  template:
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]
        args:
        - "-c"
        - "date; echo 'Hello World'; sleep 20; date; echo 'Bye'"
# restartPolicy: Never # 주석 처리
      restartPolicy: OnFailure		# 주석 삭제
  backoffLimit: 3			# 주석 삭제


[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml  --dry-run=client
job.batch/centos-job created (dry run)
```

- completions: 5 — default = 1, job 전체 기준으로 성공한 pod 수가 5개가 되면 job의 상태를 complete로 전환
- parallelism: 2 — default = 1, job을 동시에 실행할 pod의 개수
- restartPolicy: Never — 컨테이너가 작업 실패시 pod안에서 컨테이너를 재시작하지 않는다. 필요시 Job Controller가 pod를 다시 생성해서 실행
- restartPolicy: OnFailure — 컨테이너가 작업 실패시 pod안에서 컨테이너를 재시작. Pod는 그대로 유지
- backoffLimit: 3 — default = 6, Job이 작업을 실패시 재시도 횟수를 작성

```
[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml
job.batch/centos-job created
```

해당 작업을 성공한 pod가 3개이면 작업 성공으로 판단한다.

```
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME               	READY   	STATUS    	RESTARTS   AGE	IP       	  NODE     	NOMINATED NODE   READINESS GATES
centos-job-v649p   	0/1     	Pending   		0                 0s	<none>	  <none>   	<none>           <none>
centos-job-v649p   	0/1     	Pending   		0                 0s 	<none>   	  k8s-worker2   	<none>           <none>
centos-job-v649p   	0/1     	ContainerCreating 	0                 0s  	<none>   	  k8s-worker2   	<none>           <none>
centos-job-v649p   	1/1     	Running             	0                 1s    	10.244.2.9	  k8s-worker2   	<none>           <none>
centos-job-v649p   	0/1     	Completed           	0                 20s   	10.244.2.9	  k8s-worker2   	<none>           <none>
centos-job-v649p   	0/1     	Completed           	0                 21s   	10.244.2.9	  k8s-worker2   	<none>           <none>

centos-job-5vvlh   	0/1     	Pending             	0                 0s    	<none>   	  <none>        	<none>           <none>
centos-job-5vvlh   	0/1     	Pending             	0                 0s    	<none>      k8s-worker2   	<none>           <none>
centos-job-v649p   	0/1     	Completed           	0                 22s   	10.244.2.9   k8s-worker2   	<none>           <none>
centos-job-5vvlh   	0/1     	ContainerCreating	0                 0s    	<none>       k8s-worker2   	<none>           <none>
centos-job-5vvlh   	1/1     	Running             	0                 1s    	10.244.2.10   k8s-worker2   	<none>           <none>
centos-job-5vvlh   	0/1     	Completed           	0                 21s   	10.244.2.10   k8s-worker2   	<none>           <none>
centos-job-5vvlh   	0/1     	Completed           	0                 23s   	10.244.2.10   k8s-worker2   	<none>           <none>

centos-job-djh5p   	0/1     	Pending             	0                 0s    	<none>        <none>        	<none>           <none>
centos-job-djh5p   	0/1     	Pending             	0                 0s    	<none>        k8s-worker2   	<none>           <none>
centos-job-5vvlh   	0/1     	Completed           	0                 24s   	10.244.2.10   k8s-worker2   	<none>           <none>
centos-job-djh5p   	0/1     	ContainerCreating 	0                 0s    	<none>        k8s-worker2   	<none>           <none>
centos-job-djh5p   	1/1     	Running             	0                 1s    	10.244.2.11   k8s-worker2   	<none>           <none>
centos-job-djh5p   	0/1     	Completed           	0                 21s   	10.244.2.11   k8s-worker2   	<none>           <none>
centos-job-djh5p   	0/1     	Completed           	0                 22s   	10.244.2.11   k8s-worker2   	<none>           <none>
centos-job-djh5p   	0/1     	Completed           	0                 23s   	10.244.2.11   k8s-worker2   	<none>           <none>
```

### Job Controller 삭제

```
[root@k8s-master ~]# kubectl  delete jobs.batch centos-job
job.batch "centos-job" deleted from default namespace
```

### 특정 사건으로 작업이 실패 시 컨테이너를 재시작 (restartPolicy: OnFailure) — 명령어 오타로 실패 유도

```
[root@k8s-master ~]# vi job-exam.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
  completions: 3			# 주석 삭제 후 3으로 수정
# parallelism: 2
# activeDeadlineSeconds: 5
  template:
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bashz"]		<--- bashc 강제 명령어 오타
        args:
        - "-c"
        - "date; echo 'Hello World'; sleep 20; date; echo 'Bye'"
# restartPolicy: Never
      restartPolicy: OnFailure		<--- Test (작업 실패시 컨테이너 재시작)
  backoffLimit: 3			<--- Test (3번 연속 실패시 작업 실패로 간주)


[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml  --dry-run=client
job.batch/centos-job created (dry run)


[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME               READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
centos-job-msqkl   0/1     Pending   0          0s    <none>   <none>   <none>           <none>
centos-job-msqkl   0/1     Pending   0          0s    <none>   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     ContainerCreating   0          0s    <none>   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     RunContainerError   0          1s    10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     RunContainerError   1 (1s ago)   2s    10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     CrashLoopBackOff    1 (13s ago)   14s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     RunContainerError   2 (0s ago)    14s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     CrashLoopBackOff    2 (22s ago)   36s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     RunContainerError   3 (0s ago)    36s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     Terminating         3 (1s ago)    37s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     Terminating         3 (1s ago)    37s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     Terminating         3             37s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     StartError          3             37s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     StartError          3             38s   10.244.2.16   k8s-worker2   <none>           <none>
centos-job-msqkl   0/1     StartError          3             38s   10.244.2.16   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  get  job  centos-job
NAME         STATUS   COMPLETIONS   DURATION   AGE
centos-job   Failed       0/3                    5m3s           5m3s


[root@k8s-master ~]# kubectl describe  jobs.batch  centos-job
Name:        	centos-job
Namespace:        	default
Selector:         	batch.kubernetes.io/controller-uid=31513b28-7dfa-4cb1-aae7-26f5ce3aa21             4
Labels:           	batch.kubernetes.io/controller-uid=31513b28-7dfa-4cb1-aae7-26f5ce3aa21             4
                  	batch.kubernetes.io/job-name=centos-job
                  	controller-uid=31513b28-7dfa-4cb1-aae7-26f5ce3aa214
                  	job-name=centos-job
Annotations:      	<none>
Parallelism:      	1
Completions:      	3
Completion Mode:  	NonIndexed
Suspend:          	false
Backoff Limit:    	3
Start Time:      	Thu, 20 Aug 2026 12:04:00 +0900
Pods Statuses:    	0 Active (0 Ready) / 0 Succeeded / 1 Failed
Pod Template:
  Labels:	batch.kubernetes.io/controller-uid=31513b28-7dfa-4cb1-aae7-26f5ce3aa214
           	batch.kubernetes.io/job-name=centos-job
           	controller-uid=31513b28-7dfa-4cb1-aae7-26f5ce3aa214
           	job-name=centos-job
  Containers:
   centos-container:
    Image:      	centos:7
    Port:       	<none>
    Host Port:  	<none>
    Command:
      bashz
    Args:
      -c
      date; echo 'Hello World'; sleep 20; echo 'Bye'
    Environment:	<none>
    Mounts:        	<none>
  Volumes:         	<none>
  Node-Selectors:  	<none>
  Tolerations:     	<none>
Events:
  Type 	 Reason                		Age        From               Message
  ----  	 ------                		----       ----                -------
  Normal	 SuccessfulCreate      	6m23s    job-controller    Created pod: centos-job-msqkl
  Normal	 SuccessfulDelete      	5m46s    job-controller    Deleted pod: centos-job-msqkl
  Warning	 BackoffLimitExceeded	5m45s    job-controller    Job has reached the specified ba
```

- BackoffLimitExceeded: BackoffLimit에 의해 Job이 Failed됨을 의미 (실패 한도 초과)

### completions=5, parallelism=2

```
[root@k8s-master ~]# vi job-exam.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
  completions: 5			# 3에서 5로 수정
  parallelism: 2			# 주석 제거
# activeDeadlineSeconds: 5
  template:
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]
        args:
        - "-c"
        - "date; echo 'Hello World'; sleep 20; date; echo 'Bye'"
# restartPolicy: Never
      restartPolicy: OnFailure
  backoffLimit: 3


[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml
job.batch/centos-job created
```

- completions: 5 — 5번 작업을 성공해야 completed가 된다.
- parallelism: 2 — 한번에 2개의 pod를 사용하여 작업 수행

```
[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME               	READY   	STATUS    	RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
centos-job-2rjjn   	0/1     	Pending   		0                 0s    <none>   <none>   <none>           <none>
centos-job-2rjjn   	0/1     	Pending   		0                 0s    <none>   k8s-worker2   <none>           <none>
centos-job-t2b4t   	0/1     	Pending   		0                 0s    <none>   <none>        <none>           <none>
centos-job-t2b4t   	0/1     	Pending   		0                 0s    <none>   k8s-worker1   <none>           <none>
centos-job-2rjjn   	0/1     	ContainerCreating	0                 0s    <none>   k8s-worker2   <none>           <none>
centos-job-t2b4t   	0/1     	ContainerCreating	0                 0s    <none>   k8s-worker1   <none>           <none>
centos-job-2rjjn   	1/1     	Running             	0                 1s    10.244.2.12   k8s-worker2   <none>           <none>
centos-job-t2b4t   	1/1     	Running             	0                 1s    10.244.1.5    k8s-worker1   <none>           <none>
centos-job-t2b4t   	0/1     	Completed           	0                 20s   10.244.1.5    k8s-worker1   <none>           <none>
centos-job-2rjjn   	0/1     	Completed           	0                 21s   10.244.2.12   k8s-worker2   <none>           <none>
centos-job-t2b4t   	0/1     	Completed           	0                 21s   10.244.1.5    k8s-worker1   <none>           <none>
centos-job-2rjjn   	0/1     	Completed           	0                 22s   10.244.2.12   k8s-worker2   <none>           <none>

centos-job-cgtjt   	0/1     	Pending             	0                 0s    <none>        <none>        <none>           <none>
centos-job-wmnrd	0/1     	Pending             	0                 0s    <none>        <none>        <none>           <none>
centos-job-cgtjt   	0/1     	Pending             	0                 0s    <none>        k8s-worker2   <none>           <none>
centos-job-wmnrd 	0/1     	Pending             	0                 0s    <none>        k8s-worker1   <none>           <none>
centos-job-2rjjn   	0/1     	Completed           	0                 22s   10.244.2.12   k8s-worker2   <none>           <none>
centos-job-cgtjt   	0/1     	ContainerCreating	0                 0s    <none>        k8s-worker2   <none>           <none>
centos-job-t2b4t   	0/1     	Completed           	0                 22s   10.244.1.5    k8s-worker1   <none>           <none>
centos-job-wmnrd	0/1     	ContainerCreating	0                 0s    <none>        k8s-worker1   <none>           <none>
centos-job-wmnrd	1/1     	Running             	0                 1s    10.244.1.6    k8s-worker1   <none>           <none>
centos-job-cgtjt   	1/1     	Running             	0                 2s    10.244.2.13   k8s-worker2   <none>           <none>
centos-job-wmnrd	0/1     	Completed           	0                 21s   10.244.1.6    k8s-worker1   <none>           <none>
centos-job-cgtjt   	0/1     	Completed           	0                 22s   10.244.2.13   k8s-worker2   <none>           <none>
centos-job-wmnrd	0/1     	Completed           	0                 23s   10.244.1.6    k8s-worker1   <none>           <none>
centos-job-cgtjt   	0/1     	Completed           	0                 23s   10.244.2.13   k8s-worker2   <none>           <none>

centos-job-w766f   	0/1     	Pending             	0                 0s    <none>        <none>        <none>           <none>
centos-job-w766f   	0/1    	Pending             	0                 0s    <none>        k8s-worker2   <none>           <none>
centos-job-cgtjt   	0/1     	Completed           	0                 23s   10.244.2.13   k8s-worker2   <none>           <none>
centos-job-wmnrd 	0/1     	Completed           	0                 23s   10.244.1.6    k8s-worker1   <none>           <none>
centos-job-w766f   	0/1     	ContainerCreating	0                 0s    <none>        k8s-worker2   <none>           <none>
centos-job-w766f   	1/1     	Running             	0                 2s    10.244.2.14   k8s-worker2   <none>           <none>
centos-job-w766f   	0/1     	Completed           	0                 22s   10.244.2.14   k8s-worker2   <none>           <none>
centos-job-w766f   	0/1     	Completed           	0                 23s   10.244.2.14   k8s-worker2   <none>           <none>
centos-job-w766f   	0/1     	Completed           	0                 24s   10.244.2.14   k8s-worker2   <none>           <none>


[root@k8s-master ~]# kubectl  delete jobs.batch centos-job
job.batch "centos-job" deleted from default namespace
```

### activeDeadlineSeconds — Job 최대 실행 시간 제한

```
[root@k8s-master ~]# vi job-exam.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: centos-job
spec:
# completions: 5 # 주석 처리
# parallelism: 2 # 주석 처리
  activeDeadlineSeconds: 10		# 주석 삭제
  template:
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]
        args:
        - "-c"
        - "date; echo 'Hello World'; sleep 20; echo 'Bye'"
# restartPolicy: Never
      restartPolicy: OnFailure
  backoffLimit: 3
```

- activeDeadlineSeconds: 10 — Job이 실행되는 최대시간을 초단위로 제한하는 옵션. job이 실행된 후 10초가 지나면 실행중인 pod들을 강제 종료

```
[root@k8s-master ~]# kubectl  apply  -f  job-exam.yaml
job.batch/centos-job created


[root@k8s-master ~]# kubectl  get  pods  -o  wide  --watch
NAME               	READY	STATUS    	RESTARTS   AGE   IP             NODE     	NOMINATED NODE   READINESS GATES
centos-job-d8q4z   	0/1     	Pending   		0                 0s    <none>        <none>   	<none>           <none>
centos-job-d8q4z   	0/1     	Pending   		0                 0s    <none>        k8s-worker2   	<none>           <none>
centos-job-d8q4z   	0/1     	ContainerCreating	0                 0s    <none>        k8s-worker2   	<none>           <none>
centos-job-d8q4z   	1/1     	Running             	0                 1s    10.244.2.15   k8s-worker2   	<none>           <none>
centos-job-d8q4z   	1/1     	Terminating         	0                 10s   10.244.2.15   k8s-worker2   	<none>           <none>
centos-job-d8q4z   	1/1     	Terminating         	0                 10s   10.244.2.15   k8s-worker2   	<none>           <none>
centos-job-d8q4z   	1/1     	Terminating         	0                 10s   10.244.2.15   k8s-worker2   	<none>           <none>
```

sleep 20으로 작업 종료 전에 `activeDeadlineSeconds: 10`이 지나면 Job이 실행 중인 Pod를 즉시 `Terminating` 상태로 강제 종료한다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

- `kubectl run`으로 만든 단일 Pod는 기본 `restartPolicy: Always`가 적용되어 종료돼도 계속 재시작(`Completed → Running` 반복)된다. 한 번만 끝내려는 작업은 반드시 Job으로 만들어야 한다.
- Job의 `template.spec.restartPolicy`는 `Never` 또는 `OnFailure`만 허용된다(Always는 사용 불가). `Never`는 실패 시 Job Controller가 새 Pod를 생성해서 재시도하고, `OnFailure`는 같은 Pod 안에서 컨테이너만 재시작한다.
- `kubectl get job`의 STATUS가 `Failed`로 남으면 `kubectl describe jobs.batch <이름>`의 Events에서 `BackoffLimitExceeded`를 확인한다 — `backoffLimit`에 지정된 횟수만큼 재시도했는데도 exit 0을 받지 못했다는 뜻이다.
- 명령어 오타(`bashz` 등)처럼 컨테이너 자체가 뜨지 못하는 경우 `RunContainerError → CrashLoopBackOff → StartError` 순으로 상태가 순환하다가 backoffLimit 초과 시 Job이 `Failed`로 확정된다.
- `activeDeadlineSeconds`는 completions/backoffLimit과 무관하게 전체 Job의 최대 실행 시간을 강제한다. 시간이 지나면 진행 중인 Pod도 즉시 `Terminating` 처리된다.
- Job이 만든 Pod는 Job을 삭제해도(`kubectl delete jobs.batch`) 함께 삭제되지만, Pod만 개별 삭제하면 Job Controller가 `completions` 목표를 채우기 위해 새 Pod를 다시 생성할 수 있다.

---

>  **핵심 요약**
> - Job Controller는 한 번 실행하고 끝나는(성공/실패로 명확히 갈리는) 배치 작업을 위한 컨트롤러이며, exit code(0=성공)만으로 성공/실패를 판단한다
> - 핵심 옵션: `completions`(성공해야 할 횟수, 기본 1), `parallelism`(동시 실행 Pod 수, 기본 1), `backoffLimit`(재시도 허용 횟수, 기본 6), `activeDeadlineSeconds`(전체 실행 시간 제한)
> - `restartPolicy`는 `Never`(Job이 새 Pod 생성) 또는 `OnFailure`(같은 Pod 내 컨테이너 재시작)만 사용 가능
> - CronJob → Job → Pod 순서로 동작하며, Job은 "무슨 작업"을, CronJob은 "언제 실행"을 담당
> - 관련: 17. ⏰ Kubernetes - CronJob · 2.  Kubernetes - Pod 생성 · 10.  Kubernetes - Controller 개념과 ReplicationController
