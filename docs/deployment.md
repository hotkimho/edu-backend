## 디플로이먼트

### 디플로이먼트
디플로이먼트는 애플리케이션을 배포하고 파드와 레플리카셋에 대한 `선언적 업데이트`를 제공합니다.

선언적 업데이트는 `사용자가 의도하는(원하는) 상태`를 달성하기 위해 필요한 모든 작업을 자동으로 수행하는걸 의미합니다.

의도하는 상태는 다음과 같은 요소들입니다.
- 레플리카 수: 몇 개의 파드가 실행되어야 하는 것을 의미합니다.
- 컨테이너 이미지: 어떤 이미지가 사용되어야 하는 것을 의미합니다.
- 업데이트 전략: 새로운 버전을 어떻게 업데이트 할 것인지를 의미합니다.

사용자는 의도하는 상태를 정의하고, 디플로이먼트 컨트롤러는 현재 상태를 모니터링하며 의도하는 상태를 유지합니다.

![image](./images/deployment-image.png)

그림과 같이, 디플로이먼트가 생성되면 레플리카셋이 생성되고, 이어서 파드가 생성이 됩니다.

### 디플로이먼트 생성
직접 디플로이먼트를 생성하고 조회합니다.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
매니페스트 파일에서 각 옵션에 대한 설명입니다.

- .spec
  - replicas
    - 디플로이먼트는 N개의 파드를 가지는 레플리카셋을 생성합니다. 즉 생성 될 파드의 수 입니다.
  - selector
    - matchLabels
      - 생성된 레플리카셋이 관리할 파드를 선택하는 방법을 정의합니다.
      - 이 규칙은 파드의 레이블을 기반으로 합니다.
  - template
    - metadata
      - labels
        - 생성될 파드에 추가할 레이블입니다.
    - spec
      - containers(생성될 파드의 템플릿)
        - name
          - 생성될 파드의 이름입니다.
        - image
          - 생성될 파드의 이미지입니다.
        - ports:
          - containerPort
            - 컨테이너가 사용할 포트입니다.


```
# 디플로이먼트 조회
# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           23s
```
- Name: 디플로이먼트의 이름
- READY: 사용자가 사용할 수 있는 애플리케이션 레플리카의 수 필드는 `ready(사용자가 사용할 수 있는 준비된 파드 수) / desired(디플로이먼트에서 의도한 파드 수)` 패턴을 따릅니다.
- UP-TO-DATE: 의도한 상태를 얻기 위해 업데이트된 레플리카의 수
  - 디플로이먼트에서 최신 상태를 반영하기 위해 업데이트된 파드의 수 입니다.
  - 의도한 레플리카가 3개이고 이 값이 3이면 모든 파드는 최신 상태를 의미합니다.
- AVAILABLE: 사용자가 사용할 수 있는 애플리케이션 레플리카의 수

```
# 레플리카셋 조회
# kubectl get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-cbdccf466   3         3         3       95s
```
- NAME: 레플리카셋의 이름입니다.
  - 레플리카셋의 이름은 항상 `<디플로이먼트이름>-HASH` 형식으로 되어있습니다.
  - HASH 문자열은 레플리카셋과 파드의 `pod-template-hash` 레이블과 같습니다.
- DESIRED: 디플로이먼트에서 원하는 레플리카의 수 입니다.
- CURRENT: 현재 실행중인 레플리카의 수 입니다.
- READY: 사용자가 사용할 수 있는 애플리케이션 레플리카의 수 입니다.
  - 사용할 수 있는 애플리케이션 레플리카는 `프로브`를 통과하여 요청을 수행할 수 있는 컨테이너를 의미합니다.

```
# 파드 조회
# kubect get pod
NAME                               READY   STATUS      RESTARTS          AGE
nginx-deployment-cbdccf466-6nfh2   1/1     Running     0                 116s
nginx-deployment-cbdccf466-7l782   1/1     Running     0                 116s
nginx-deployment-cbdccf466-dq5ft   1/1     Running     0                 116s
```
- NAME: 파드의 이름입니다.
  - 파드의 이름은 항상 `<디플로이먼트이름>-<pod-template-hash>-HASH` 형식으로 되어있습니다.
- READY: 파드 내에서 사용자가 사용할 수 있는 컨테이너의 수 입니다. `ready/total` 형식으로 되어 있습니다.
- STATUS: 파드의 상태입니다.

### 롤아웃
디플로이먼트에서 롤아웃은 애플리케이션의 새로운 버전이 배포되거나 구성 변경을 반영하기 위해 기존 파드를 새로운 파드로 교체하는 과정을 의미합니다. 

디플로이먼트의 `파드 템플릿(.spec.template)이 변경된 경우에만`(레이블 또는 컨테이너 이미지가 변경된 경우) 디플로이먼트의 `롤아웃이 트리거`됩니다. 하지만 `스케일링(레플리카 수 변경) 업데이트는 롤아웃을 트리거 하지 않습`니다.


