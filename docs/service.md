### 서비스
파드는 외부의 요청없이 독립적으로 실행할 수 있지만 대부분의 애플리케이션들은 외부 요청에 응답하기 위한 것이다.
물론 파드도 내부적으로 IP를 가지고 있지만 쿠버네티스의 파드는 굉장히 삭제되고 변경되기 쉽다. 외부에서 특정 파드의 IP를 알고 접속을 한다고 해도 IP가 변경될 수 있어 원하는 파드에 접근한건지 보장할 수 없다. 
그리고 파드가 할당될 때 IP가 부여되므로 미리 파드의 IP를 알 수 없다.
이런 문제를 해결하기 위해 파드 집합에서 실행중인 애플리케이션을 네트워크 서비스로 추상화한 방법이 `서비스`이다.

![image](./images/service-example-image.png)

사용자들이 접근하는 프론트 엔드기능은 3개의 파드(2.1.1.0/24)로 구성되어 있고, 프론트엔드가 접근하는 백엔드 기능은 1개의 파드(2.1.1.4)로 되어 있다.
프론트엔드 서비스, 백엔드 서비스 같이 각각의 파드 집합에 대한 접점을 만든다.
그러면 사용자는 프론트엔드 서비스에 요청 시, 프론트엔드 서비스가 3개의 파드 중 1개로 요청을 하고 백엔드 또한 백엔드 서비스가 백엔드 파드로 요청을 전달한다.
이렇게 되면 사용자는 프론트엔드 서비스의 ip/port만 알면되고 프론트엔드 또한 백엔드 서비스의 ip/port만 알면된다.
그래서 서비스의 ip/port는 파드에 영향을 받아선 안되고 변경되서도 안된다.

### 서비스 생성
서비스를 지원하는 pod를 어떻게 관리할 수 있을까? 이 경우 레이블 셀렉터를 사용하여 어떤 파드가 어떤 서비스에 속하는지 알 수 있다.

![image](.//images/service-label-selector.png)

그림을 보면 pod는 `app:kubia` 라는 라벨을 가지고 서비스에서 `app=kubia`에 대해 셀렉팅을 한다.
이렇게 레이블 셀렉팅 기능을 이용해 서비스에서 파드를 관리할 수 있다.

service yaml
```
apiVersion: v1
kind: Service
metadata:
  name: kimho
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
  - name: kimho-port
    port: 406
    targetPort: kimhotarget
  selector:
    app: kimho
```

pod yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: kimho-pod
  labels:
    app: kimho
spec:
  containers:
  - image: luksa/kubia
    name: kimho-pod
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```

서비스, pod에서 같은 label `app:kimho`를 사용하고 있다.
![image](./images/req-service.png)

서비스를 조회하고 서비스의 할당된 ip에 요청 시, 파드로 요청이 가는걸 볼 수 있다.

### 서비스 검색
서비스는 한번 만들면 고정 ip/port가 생기고 서비스가 유지되는 동안 변경되지 않는다.
하지만 파드는 삭제되고 생성되기를 반복한다. 그럼 이 파드들은 어떻게 서비스의 정보를 알 수 있을까?
파드가 시작되면 `kubectl`을 통해 서비스의 정보를 환경변수로 가져와 저장한다.

쉽게 값을 찾을 수 있게 서비스 매니페스트 파일을 만들었다.
```
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
  - name: kimho-port
    port: 406
    targetPort: kimhotarget
  selector:
```
파드의 환경 변수에 kimho-port인 406번을 찾으면 된다.

`kubectl exec kimho-pod -- env | ag kimho`
명령을 통해 환경변수 리스트에서 kimho가 붙은 리스트를 조회한다.

![image](./images/pod-env.png)
`SERVICE_PORT_KIMHO_PORT=406`을 통해 서비스 매니페스트 파일에 정의된 내용이 파드의 환경변수에 있는걸 확인할 수 있다.


