### 파드 기본 개념
파드는 쿠버네티스에서 생성하고 관리할 수 있는 배포 가능한 가장 작은 단위로 표현한다.
파드는 하나 이상의 컨테이너의 그룹이다. 파드 안에 여러 개의 컨테이너가 있으면 이 컨테이너는 네트워크, 스토리지 같이 같은 환경에서 사용되는 자원은 공유한다.(가능하면 1개의 컨테이너만 실행 권장)
또한 모든 파드는 하나의 워커 노드에서 실행되며, 여러 워커 노드에 의해 실행되지 않는다(즉 2개의 노드에서 1개의 파드에 대해 공유할 수 없다)


### 파드가 필요한 이유
굳이 파드를 사용하지 않고 워커 노드에서 직접 컨테이너를 사용할 수 있다.
하지만 이런 경우 많은 문제가 생긴다.
1. 네트워크 및 스토리지 설정
하나의 노드에서 많은 컨테이너를 사용할 경우 네트워크 및 스토리지 설정을 다 구성해야 하므로 복잡성이 증가한다.
2. 일관된 배포 단위
파드는 여러 컨테이너를 그룹화하여 관리할 수 있다. 예를 들면 주 애플리케이션 컨테이너와 이를 보조하는 `사이드카 컨테이너`를 함께 배치할 수 있다.
3. 확장성
파드는 디플로이먼트, 레플리카셋 기능을 이용해 쉽게 확장할 수 있다. 파드를 직접 확장/축소 하는 작업이 직접 컨테이너를 사용하여 확장/축소를 하는 것보다 더 효율성이 뛰어나다.

결과적으로 컨테이너를 그룹화하여 관리할 수 있는 구조가 필요하며 그게 바로 `파드`이다.

### 파드 내부
파드 내부의 모든 컨테이너는 LAN환경과 같이 접근할 수 있다. localhost를 사용하여 컨테이너간 통신을 할 수 있고, 공유 스토리지 볼륨을 설정하여 데이터를 공유할 수 있다. 단 같은 네트워크를 사용하기에 포트번호가 중복되면 안된다.


### 멀티 컨테이너
쿠버네티스는 파드안에 한 개의 컨테이너 사용을 권장한다. 하지만 상황에 따라 한 파드안에 여러 개의 컨테이너가 사용될 수 있다. 여러 개의 컨테이너를 사용하는 경우 장점과 단점을 알아보자.

### 멀티 컨테이너 장점
1. 통신이 유용함
같은 파드내에 컨테이너들은 별도의 과정없이 로컬 환경에 접근하듯 다른 컨테이너에 접근할 수 있어 속도가 빠르고 네트워크 지연, 트래픽에 대한 비용을 줄여준다.
2. 데이터 공유
공유 볼륨을 사용하여 설정 파일, 로그, 캐시등 공유가 필요한 데이터에 쉽게 접근할 수 있다.
3. 애플리케이션 디자인 유용성
보조 작업을 수행하는 사이드카 컨테이너같은 특정 패턴을 사용하여 컨테이너간 기능을 분리할 수 있다.
4. 개발 및 테스트 용이성
개발 구성 요소를 하나의 파드이 배치하여, 통합된 환경에서 개발 및 테스트를 진행할 수 있다.

### 멀티컨테이너 단점
1. 복잡성 증가
한 파드내에 여러 컨테이너를 관리할 때, 컨테이너간 의존성이 발생하여 복잡해질 수 있다. 그리고 컨테이너가 자원 공유 하므로 파드에서 자원 할당, 모니터링등이 복잡해진다.

2. 오류 격리
한 컨테이너에서 발생한 오류(리소스 포함)가 같은 파드내의 컨테이너에 영향을 미칠 수 있다.

3. 스케일링 제약
같은 그룹으로 스케일링을 편하게 할 수 있는 장점이 있지만 항상 동일한 파드가 스케일링 되므로, 의도치 않는 컨테이너 또한 늘어날 수 있다.

### 사이드카 컨테이너
파드안에 한 개의 컨테이너를 권장하지만 보조 기능을 제공하는 컨테이너를 사용할 수 있다 이를 `사이드카 컨테이너`라고 하고 멀티 컨테이너 파드의 디자인 패턴중 하나이다.

![image](./images/sidecar-container.png)
그림을 보면 Application Pod는 web server container와 log saving container를 가지고 있다.

웹 서버 컨테이너에서 발생한 표준 출력, 표준 에러에 대한 로그를 파일 시스템에 저장하면 log saving 컨테이너가 파일 시스템의 로그에 접근한다. 이런 방식으로 메인 컨테이너와 보조 컨테이너를 사용하는 패턴을 사이드카 패턴, 사이드카 컨테이너 라고 한다.

### yaml 매니페스트 구조
파드를 포함한 쿠버네티스의 자원들은 REST API 엔드포인트에 JSON, YAML 매니페스트를 전송해 생성한다.
yaml은 버전 정의, 리소스를 명시 한다음 크게 3가지 부분으로 구분된다.
- metadata : 이름, 네임스페이스, 레이블, 파트 정보를 갖는다.
- spec : 파드 컨테이너, 볼륨, 파드에 대한 실제 명세를 가진다.
- Status : 파드 상태, 컨테이너 설명과 상태, 파드 내부 IP등 실행 중인 파드에 관한 정보를 갖는다.

```
apiVersion : v1
kind : Pod
metadata:
  name: nginx-pod # 파드 이름
spec:
  containers:
  - image: nginx
    name: nginx-container # 컨테이너 이름
    ports:
    - containerPort: 8080
      protocol: TCP
```

작성한 YAML을 통해 파드를 생성하려면 아래와 같이 명령어를 작성한다


```
# kubectl create -f {YAML파일이름}
kubectl create -f 

# 파드 리스트 확인
kubectl get pods -o wide

# 생성된 파드 정보 상세 조회
kubectl describe pod nginx-pod
```

![image](images/nginx-pod.png)
![image](images/nginx-pod-detail2.png)

파드 리스트 조회를 통해 생성된 걸 확인하고, 파드를 상세 조회를 통해 YAML파일을 통해 생성된 걸 확인할 수 있다.

### 파드 로그
![image](./images/logging-node-level.png)

파드의 로그는 각 컨테이너에 출력되는 `표준 출력`, `표준 에러` 로그들을 기록한다.

로그를 저장하는 위치는 컨테이너 런타임에 따라 다르지만 containerd를 사용하는 경우 `/var/log/pods/{pod-uid}/{container-name}/` 경로에 생성된다(실제로 확인한 결과 `/var/log/nginx`의 경로에 있었다)

파드의 로그를 조회하는 방법은 아래와 같다.
```
kubectl logs {pod 이름}

kubectl logs {pod 이름} -c {컨테이너 이름}

# 리눅스의 tail 명령 처럼 마지막 N줄의 로그 조회
kubectl logs --tail=N {pod 이름}

# 실시간으로 출력되는 로그 조회
kubectl logs -f {pod 이름}
```

해당 예제는 `nginx-pod` 라는 파드이름을 갖고 파드 안에서 `nginx-container`라는 이름의 컨테이너로 실행된다. 그래서 결과적으로 같은 로그가 출력된다.

![image](images/nginx-pod-log.png)

### 파드 요청(port-forward)
파드에 디버깅 목적으로 직접 접속할 경우 포트 포워딩을 이용해 접근할 수 있다. 포트 포워딩을 이용하여 서비스를 만들지 않고 직접 파드에 접속한다.

```
# kubectl port-forward {pod 이름} {port:port}
kubectl port-forward nginx-pod 8888:80
```

![image](./images/port-forward.png)

포트 포워드 명령을 통해 `nginx-pod`는 localhost:8888로 접근할 수 있게 되었다.

![image](./images/req-port-forward.png)

직접 `localhost:88888`로 접근했을 때 요청이 성공한걸 볼 수 있다.

이렇게 서비스를 사용하지 않고 개발 및 디버깅 용도로 사용할 수 있다.

### 파드 요청 과정
`kubectl get pods`

일반적으로 파드를 조회할 때 위의 명령어를 사용한다. 그럼 위의 명령어는 어떻게 파드의 정보를 가져올까?
어떻게 동작되는지 확인하기 위해 명령의 로그 상세 레벨을 확인하는 명령어가 있다.
```
kubectl get pods --v={1~9} # 숫자가 클수록 자세하게 로깅
```
![alt text](./images/pod-logging.png)
굉장히 여러 단계에 걸치고 값이 많지만 하나씩 단계를 확인한다.

1. kubeconfig 파일 로드

API 서버와 통신하기 위한 설정을 불러온다. 서버 정보, 클러스터 정보, 인증서 등등을 가지며 아래에 있는 내용은 실제 kubeconfig 파일의 내용이다.
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: C:\Users\kimho\.minikube\ca.crt
    extensions:
    - extension:
        last-update: Thu, 13 Jun 2024 20:44:08 KST
        provider: minikube.sigs.k8s.io
        version: v1.33.1
      name: cluster_info
    server: https://127.0.0.1:50581
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Thu, 13 Jun 2024 20:44:08 KST
        provider: minikube.sigs.k8s.io
        version: v1.33.1
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: C:\Users\kimho\.minikube\profiles\minikube\client.crt
    client-key: C:\Users\kimho\.minikube\profiles\minikube\client.key

```

2. 인증서 회전
```
I0613 22:21:58.114634    9156 cert_rotation.go:137] Starting client certificate rotation controller
```
인증서는 유효기간을 가지고 있어 보안을 위해 주기적으로 갱신을 해야한다. `인증서 회전`이라는 과정을 통해 인증서를 갱신하며, `인증서 회전 컨트롤러`가 이 역할을 수행한다.

3. API 서버 요청 및 응답 확인
```
curl -v -XGET  -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" -H "User-Agent: kubectl.exe/v1.29.1 (windows/amd64) kubernetes/bc401b9" 'https://127.0.0.1:50581/api/v1/namespaces/default/pods?limit=500'

HTTP Trace: Dial to tcp:127.0.0.1:50581 succeed

GET https://127.0.0.1:50581/api/v1/namespaces/default/pods?limit=500 200 OK in 5 milliseconds
```
`https://127.0.0.1:50581/api/v1/namespaces/default/pods?limit=500` 주소로 요청을 보냈고, 200 status code를 통해 요청이 성공적으로 이루어진걸 볼 수 있다.

4. 응답
  
응답된 response body를 보면 굉장히 많은 데이터가 왔지만 json parser을 통해 필요한 부분만 확인한다.
![image](./images/response-body.png)

json을 확인하면 실제로 파드에 조회되는 정보들(cells), 파드 이름, 파드에서 실행되는 container name까지 확인할 수 있다.

### 레이블
레이블은 쿠버네티스 오브젝트(pod, deployment)에 첨부된 (키, 값) 형태의 값이다.
어떻게 보면 각 오브젝트에 할당된 태그와 같다.
이 값들은 레이블 셀렉터를 사용해 리소스를 선택할 때 활용된다.
![image](./images/label-pic.png)

그림을 보면
- app={ui, as, pc, sc, os}
- rel={beta, stable, canary}
형태로 레이블을 갖고 오브젝트들은 1개 이상의 레이블을 가질 수 있다.
레이블은 파드를 생성할 때 할당할 수 있고 도중에 수정할 수 있다.

```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```
이 YAML을 보면 metadata에 레이블을 2개 추가 했다.

```
# 레이블 조회
kubectl get pod --show-labels

# created_method, env 레이블을 포함하는 파드 조회
kubectl get pod -L created_method,env

# 파드내 라벨 수정
kubectl label pod nginx-pod created_method=manual --overwrite
```

![image](./images/labels-cmd.png)

2개의 pod의 레이블을 포함하는 파드를 조회하고, 수정하는 명령어다.

### 레이블 셀렉터
레이블 셀렉터를 이용하면 조건에 맞는 리소스를 가져올 수 있다.
```
# creation_method, manual의 (키, 값)을 가지는 파드 필터링
kubectl get pod -l creation_method=manual

# env 레이블을 가지는 파드 조회
kubectl get pod -l env
```

- key!=value : 키 레이블을 가지는 파드중 값이 value가 아닌 파드 조회
- key in (v1, v2) : 키 레이블을 가지는 파드중 v1 또는 v2값을 가지는 파드 조회

이런식으로 원하는 레이블을 가진 파드를 조회할 수 있다.

### 네임스페이스
쿠버네티스의 네임스페이스는 오브젝트를 그룹화한 것이다.
리눅스의 격리 수준의 네임스페이스가 아닌 오브젝트 이름 범위를 제공한다.

그럼 왜 네임스페이스를 사용하여 오브젝트를 그룹화 하는 것인가??
복잡한 구성 요소를 관리하기 용이하게 작은 개별 그룹으로 분할할 수 있다. 이는 개발, 운영, QA등 다양한 기준으로 나뉜다.

모든 오브젝트가 네임스페이스에 속한 것은 아니다 `노드 리소스`의 경우 네임스페이스에 얽매이지 않고 전역적인 속성을 가진다.

![image](./images/get-namespace.png)
```
kubectl get namespace
```
명령을 통해 네임스페이스를 조회할 수 있다.

![image](./images/get-kube-system.png)
`kube-system` 네임스페이스를 조회하면 이미 배웠던 컨트롤러, 매니저, etcd등 실제로 노드에서 동작하는 리소스들이 조회된다.

이렇게 서로 관계없는 리소스를 그룹으로 분리하여 여러 사용자 또는 그룹이 고유한 네임스페이스를 사용하여, 다른 사람의 리소스에 접근 범위를 제한해야 한다.

### 네임스페이스 생성
네임스페이스 생성은 YAML 파일 또는 kubectl 명령을 통해 CLI로 만들 수 있다.

```
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-namespace
```

```
# YAML로 네임스페이스 생성
kubectl create -f {yaml 파일}

# CLI로 네임스페이스 생성
kubectl create namespace test-namespace
```

YAML 매니페스트를 서버에 전송해서 네임스페이스를 생성/읽기/갱신/삭제 등 API 서버에 직접 전송해 실행할 수 있다.

### 파드 삭제
파드를 삭제할 때 `그레이스풀, 논 그레이스풀` 삭제 방식이 있다.
그레이스풀 종료는 파드 내의 컨테이너가 종료 신호를 받으면, 상태를 저장하고 연결을 정리할 시간을 갖고 종료하는 작업을 말합니다.
종료 시간은 기본적으로 `30초`이며 이는 명령어를 통해 수정할수 있습니다.

그레이스풀 종료 방식은 다음과 같습니다.
1. 파드 삭제 요청
    
`kubectl delete pod {pod name}` 명령어가 실행되면, 쿠버네티스 API 서버에 파드 요청을 보냄

2. API 서버 요청 처리

API서버에 파드 삭제 요청이 오면 API 서버는 etcd에 해당 파드의 상태를 업데이트하며, 파드를 가지고 있는 노드의 `kubelet`에 파드 삭제 신호를 보낸다. kubelet는 파드내의 모든 컨테이너에 `TERM`신호를 보낸다.

3. 컨테이너 종료

TERM 신호를 받은 컨테이너는 프로세스를 종료하며, 이때 연결을 종료하고 리소스를 정리한 후 프로세스를 종료한다.
이는 그레이스풀 종료 시간(기본 30초)동안 종료되지 않으면 kubelet은 `kill`신호를 보내 강제로 종료한다.

4. 종료 확인

모든 컨테이너가 종료되면 kubelet은 API서버에 삭제가 완료되었다고 알리고 API 서버는 etcd에서 파드 객체를 삭제한다.
이후 워커노드는 해당 파드가 사용중인 리소스를 해제한다.

논 그레이스풀 종료는 위의 `3번과정`에서 컨테이너들을 강제로 kill 신호를 통해 즉시 종료하며 이 과정중 데이터의 손실이 발생할 수 있다.
빠르게 컨테이너(파드)를 삭제하고 리소스를 해제할 순 있지만 데이터가 손실될 우려가 있다.




