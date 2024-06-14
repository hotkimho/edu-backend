### 라이브니스 프로브
컨테이너에 문제가 생겨 제대로 동작되지 않는 경우, `kubectl`이 해당 컨테이너를 재시작한다.
하지만 무한 루프나 데드락 같이 문제가 있지만 정상적으로 동작하는 것처럼 보이는 경우 어떻게 해야할까? 그래서 애플리케이션의 내부에서 문제를 파악하기 보단 외부에서 애플리케이션의 상태를 체크해야 한다.

쿠버네티스는 `라이브니스 프로브`를 통해 컨테이너가 살아 있는지 확인할 수 있다.
주기적으로 라이브니스 프로브를 실행하고 프로브가 실패할 경우, 해당 컨테이너를 다시 시작한다.

세 가지 메커니즘으로 컨테이너에 프로브를 실행한다.
1. GET 프로브
지정된 ip, port, path에  HTTP GET 메소드로 요청하고, status code(200, 300번대)로 문제를 판단한다.
잘못된 응답 코드나, 응답이 없는 경우, 해당 컨테이너를 다시 시작한다.
2. TCP 소켓 프로브
컨테이너 지정된 포트에 TCP연결을 시도하고 연결이 실패한 경우, 해당 컨테이너를 다시 시작한다.
3. Exec 프로브
컨테이너내의 임의의 명령을 실행하고 명령의 종료 상태코드를 확인한다. 0이면 성공, 0이외의 값이면 실패로 간주한다.

결과적으로 컨테이너에서 에러가 발생하거나, 라이브니스 프로브가 실패한 경우 노드의 `kubelet`은 해당 컨테이너를 재시작한다.

### 레플리카셋
레플리카셋은 레플리카 파드 집합의 실행을 안정적으로 유지하는 것으로, 파드를 지속적으로 모니터링하여 파드 수가 의도하는 수와 일치하는치 확인하고 실행하는걸 보장한다.
특정 레이블의 키를 갖는 파드를 매칭시킬 수 있다.
YAML파일을 보면서 필드를 확인하고 어떻게 생성되는지 확인하자.
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```
`kind : ReplicaSet` 이제 Pod가 아닌 레플리카셋으로 명시한다.
`spec.replicas : 3` 몇 개의 파드가 실행될 지 설정한다.
`spec.selector.matchLabels : app:kubia` app:kubia 레이블을 가진 파드에 대해서만 관리한다.
`template` : 실제 파드에서 실행될 컨테이너이며, 추가/삭제될 파드에 대한 템플릿이다.

실제 이 YAML 파일로 레플리카 셋을 만들면 다음과 같이 파드가 만들어진다.
![image](./images/create-replicaset.png)

`kubectl get rs` 명령을 통해 현재 레플리카셋의 파드가 몇 개 생성되어야 하고 몇 개 생성 되어있는지 확인할 수 있다.
`app:kubia`의 레이블을 가지는 파드만 레플리카셋이 관리하기 때문에 이미지가 같은 파드들이 있지만 별도로 3개만 생성된걸 알 수 있다.

![image](./images/detail-replicaset.png)

`kubectl describe rs` 명령을 통해 레플리카셋에 대한 자세한 정보와 어떤 파드가 만들어졌는지 등 이벤트를 확인할 수 있다.

```
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
         - kubia
```
레플리카셋에서 레이블 셀렉터를 이용하여 좀더 파드와의 매칭을 확장할 수 있다.
- `In` : 레이블이 지정된 값 중 하나와 일치
- `NotIn` : 레이블 값이 지정된 값과 일치하지 않아야 함
- `Exists` : 파드는 지정된 값을 키로 가진 레이블을 포함해야 함(이 연산자를 사용할 땐, 값 필드를 지정하지 않아야 함)
- `DoesNotExist` : 파드에 지정된 값을 키로 가진 레이블이 포함되어 있지 않아야함(똑같이 값 필드를 지정하지 않아야 함)
위의 연산자를 활용하여 YAML 매니페스트 파일에서 파드를 매칭할 수 있다.

### 데몬셋
레플리카셋은 노드의 몇 개의 파드가 배치될지 정할 수 없다. 이는 스케줄러에의해 할당된다. 하지만 각각의 노드에서 1개씩 지정된 파드를 실행하려면 어떻게 해야할가?(kube-proxy 모든 노드에 1개씩 있는 느낌)
각각의 노드에서 1개씩 파드를 실행하려면 데몬셋 오브젝트를 생성해야 한다.
생성된 파드는 타겟 노드가 지정되어 있고 스케줄링 기능을 사용하지 않으면 이런 요구 사항을 만족할 수 있다.
데몬셋은 레플리카셋과 다르게 `원하는 파드의 수` 개념이 없다. 파드 셀렉터와 일치하는 파드가 노드에서 실행 중인지 확인하는 것이기 때문에, 복제본 개념이 필요 없다. 그리고 노드가 다운됐다고 해서 다른 노드에 파드를 생성하지 않는다. 대신 새 노드가 추가된 경우 생성된 노드에 파드를 할당한다.

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```
이 매니페스트는 데몬셋을 생성한다
- spec
  - template
    - spec
      - nodeSelector
        - disk: ssd
노드 셀렉터는 위와 같이 되어 있다. 이 뜻은 `disk:ssd` 레이블을 갖는 노드에만 해당 파드를 할당한다는 의미이다.

`disk:ssd` 레이블을 가진 노드가 없기 때문에 생성되어 있는 노드에 레이블을 추가한다.

```
# kubectl label node {노드이름} disk=ssd
kubectl label node hkim-host-node disk=ssd
```

![alt text](./images/before-daemonset.png)
노드에 레이블을 추가 하기 전 파드 조회


![alt text](./images/add-daemonset-label.png)
노드에 `disk:ssd` 레이블 추가

![alt text](./images/after-daemonset.png)
데몬셋을 생성하고 데몬셋의 노드 셀렉터에 만족하는 레이블을 노드에 추가하니 데몬셋에 의해 파드가 생성되었다.

### 잡
이전에 작성했던 레플리카셋, 데몬셋은 완료됐다는 속성없이 지속적인 태스크를 실행한다. 예를 들어 API서버의 경우 작업이 종료되는게 아닌 무한히 요청을 대기 한다.
하지만 완료 가능한 태스크가 필요하며, 이런 태스크는 작업이 완료되면 소멸된다. 쿠버네티스는 이런 리소스를 잡을 통해 제공한다.

잡은 파드의 `컨테이너 내부에서 실행 중인 프로세스가 완료되면 컨테이너를 다시 시작하지 않는 파드`를 실행할 수 있다. 프로세스가 완료되면 파드는 완료된 것으로 간주한다.
노드에 장애가 발생한 경우 잡은 레플리카셋과 동일하게 파드에 다시 스케줄링된다. 하지만 프로세스 자체에 장애가 발생한 경우, 잡에서 컨테이너를 다시 시작할 것인지 설정할 수 있다.

그럼 잡은 어떤 상황에 유용할까?
잡은 작업이 완료되는 것이 중요한 임시 작업에 유용하다. `관리되지 않은 임시 작업이 장애가 발생하여 중단된 경우, 수동으로 다시 생성해주어야 하며 장애 발생에 대한 감시가 이루어져야 하기에 관리자가 직접 하는건 굉장히 힘든일이다.`
그래서 잡을 통해 임시 작업을 관리하여 노드에 장애가 발생한 경우 잡 파드를 다시 생성하여 실행이 완료할 수 있게 보장해주어야 한다.
예를 들어 데이터를 저장하고, 저장된 데이터를 변환해서 어딘가로 전송해야 하는 일정 시간 이상이 걸리는 작업인 경우 잡을 통해 관리하면 매우 유용하다.

```
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```
이 매니페스트 파일을 통해 잡을 생성할 수 있다.
- spec
  - template
    - spec
      - restartPolicy
        - OnFailure값은 비정상적으로 종료되면 다시 시작하는 옵션이다
        - Always(Deault) 정상적으로 종료 되었다고 해도 다시 시작하는 정책이다.
        - Never 성공/실패 유무에 관계없이 재시작 하지 않는 정책이다.


잡은 여러 번 동작하게 설정할 수 있고 병렬로도 실행이 가능하다.
```
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  parallelism: 2
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

- spec
  - completions
    - 이 속성은 총 5번의 작업이 완료되어야 하고, `parallelism` 속성이 없으면 한 작업씩 순차적으로 이루어진다. 만약 중간에 실패하면 새 파드를 생성하여 작업을 계속한다. 즉 최소 5번 이상의 파드가 만들어진다.
  - parallelism
    - 이 속성은 병렬로 최대 n개의 파드가 생성되어 작업을 병렬로 처리한다는 의미이다. 이 옵션은 잡이 실행되는 동안 변경할 수 있으며 증가한 경우 바로 파드를 만들어 실해한다.

### 크론 잡
주기적으로(지정된 간격) 반복 실행되는 작업을 리눅스에서 `크론(cron)` 이라고 한다. 크론 잡은 일정한 간격으로 잡을 실행한다.
크론의 주기는 5개의 시간으로 이루어진다
*(분) *(시간) *(일) *(월) *(요일)
ex)
- 0,20,40 * * * * : 매일 매시간 0, 20, 40분에 실행
- 45 5 * * 5 : 매 주 금요일 오전 5시 45분에 실행


```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```
- spec
  - schedule
    - 0, 15, 30, 45 * * * * 설정은 매일 매시간 0, 15, 30, 45분에 실행한다는 의미이다.

이 크론잡은 15분 간격으로 잡 리소스르를 생성합니다.

추가적으로 `startingDeadlineSeconds` 옵션을 할당할 수 있다. 이 옵션은 지정된 초 이내로 파드가 시작해야하는 옵션이다.

주의할 점은 이 옵션이 굉장히 작은 시간(10초 미만)으로 설정되어 있으면 크론잡이 스케줄 되지 않을 수 있다.
크론잡 컨트롤러는 10초마다 마지막 부터 현재까지 얼마나 많은 크론잡이 누락되었는지 확인한다. 만약 일정 수 이상의 일정이 누락되면 잡을 실행하지 않고 에어 로그를 남긴다. 