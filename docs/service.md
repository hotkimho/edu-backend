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

### 서비스 검색(환경변수)
서비스는 한번 만들면 고정 ip/port가 생기고 서비스가 유지되는 동안 변경되지 않는다.
하지만 파드는 삭제되고 생성되기를 반복한다. 그럼 이 파드들은 어떻게 서비스의 정보를 알 수 있을까?
파드가 시작되면 `kubelet`을 통해 서비스의 정보를 환경변수로 가져와 저장한다.

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

### 서비스 검색(DNS)
파드에서 환경변수를 통해 서비스의 정보를 알 수 있지만 도메인을 사용할 수도 있다.
`kube-system` 안에 dns 서비스가 있고, 쿠버네티스의 모든 파드는 이 서비스를 사용하도록 구성된다.

![image](./images/dns.png)
사진을 보면 `kube-system` 네임스페이스 안에 `kube-dns` 서비스가 있고, 실제 실행된 파드에서 `/etc/resolv.conf` 경로에 기본 dns 서버로 구성된걸 확인할 수 있다.

그래서 파드에서 실행중인 프로세스에서 수행된 모든 dns 쿼리는 kube-system에서 실행중인 모든 서비스를 알고 있는 쿠버네티스의 자체 dns 서버로 처리된다.

파드가 내부 dns 서버를 사용할지 설정할 수 있는 옵션이 있는데, 파드를 정의할 때 스펙의 dnsPolicy속성으로 설정할 수 있다.

그래서 각 서비스는 내부 dns서버에서 dns 항목을 가져오고, 자기랑 매핑(셀렉터에 의해)된 서비스의 정보를 아는 파드는 환경변수 대신 `FQDN`으로 액세스할 수 있다.

### 서비스 검색(FQDN)
`FQDN(Fully Qualified Domain Name)`은 정규화된 도메인으로 정확한 호스트의 전체 도메인을 의미한다.
예를 들어 `www.naver.com`의 주소가 있을 때, www.naver.com이 FQDN이고, 이는 www(호스트), naver.com(도메인)이 결합된 형태이다.

쿠버네티스에서 FQDN 다음과 같이 정의된다.
```
# {service-name}.namespace.svc(일반적으로 클러스터내의 서비스를 의미).cluster.local
```
현재 내가 실행중인 파드에서 접속해보면 어떨까?
`kimho.default.svc.cluster.local` 이 주소로 요청을 보낸다.

![image](./images/req-fqdn.png)

결국 이렇게 주소를 보내면 kimho 서비스를 통해 실행된 pod에 요청하게 되므로, kimho(service), kimho-pod(pod) 인 상황에선 자기자신에게 요청을 보낸것과 같다.

### 엔드 포인트
서비스는 파드에 직접 연결되지 않고 그 중간에 엔드포인트 리소스가 있다. 실제로 서비스를 조회하면 엔드포인트 리소스 정보를 확인할 수 있다.

![image](./images/describe-service.png)

그림을 보면 서비스를 상세 조회한 경우 엔드포인트 리소스를 확인할 수 있다.
또한 `kubectl get endpoints {service-name}` 으로도 조회할 수 있다.
파드 셀렉터는 서비스 스펙에 정의되어 있지만 파드와의 연결을 전달할 때 직접 사용하지 않는다. 대신 셀렉터는 ip/port 목록을 작성하는데 사용되며 엔드포인트 리소스에 저장한다.
클라이언트가 서비스로 요청한 경우, `서비스 프록시(kube-proxy)`는 요청된 ip/port를 확인하여 대상 파드로 전달한다.

### 외부 서비스 연결
엔드포인트를 수동으로 구성하여 서비스로 요청이 온 경우 외부와 통신하게 할 수 있다.

```
apiVersion: v1
kind: Endpoints
metadata:
  name: kimho
subsets:
  - addresses:
    - ip: 100.100.100.31
    ports:
    - port: 80
```
이런 매니페스트 파일이 있는 경우, 파드에서 kimho 서비스로 요청을 보낸 경우, 100.100.100.31로 요청을 전달할 수 있다.


### 외부 클라이언트에 서비스 노출
지금까진 내부에서 파드가 서비스를 사용하는 방법을 다뤘다.
이젠 서비스를 클러스터 외부에 노출하여 외부에서 접근할 수 있게 하는 방법을 다룬다.

외부에서 서비스에 액세스할 경우 다음과 같은 방법을 사용할 수 있다.
- 노드포트
  - 노드포트 서비스는 클러스터 노드 자체에서 포트를 열고 해당 포트에 요청된 트래픽을 서비스에 전달한다.
- 로드밸런서
  - 쿠버네티스가 실행중인 인프라에서 프로비저닝된 전용 로드밸런서로 서비스에 접근할 수 있다. 로드밸런서는 트래픽을 모든 노드의 노드포트에 전달한다. 클라이언트는 요청을 로드밸런서에 직접 요청한다.
- 인그레스

### 노드포트
```
apiVersion: v1
kind: Service
metadata:
  name: kimho-nodeport
spec:
  type: NodePort
  ports:
  - port: 80            # 내부 클러스트 IP 포트
    targetPort: 8080    # 서비스 대상 파드의 포트
    nodePort: 30123     # 각 클러스터 노드의 포트(생략 시 임의로 생성)
  selector:
    app: kimho
```
매니페스트을 통해 직접 노드포트를 만들어본다.
이렇게 만들어진 노드포트는
- {내부 클러스트 IP}:80
- {노드 IP}:30123
을 통해 접근할 수 있다.

![image](./images/req-nodeport.png)
실제 클러스터 IP, 그리고 노드의 IP로 요청하고 요청이 성공적으로 이루어진걸 알 수 있다.

### 로드밸런서
클라우드 공급자(AWS등)에서 실행되는 쿠버네티스는 공급자에서 제공한 로드밸런서를 통해 프로비저닝된 기능을 제공한다.
로드밸런서는 액세스 가능한 고유 IP주소를 가지며, 요청이 들어온 경우, 로드밸런서에 의해 필요한 서비스로 요청이 전달된다.

```
apiVersion: v1
kind: Service
metadata:
  name: kimho-loadbalancear
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kimho
```
- spce
  - ports
    - port : 80
      - targetPort: 8080
이 의미하는건 해당 서비스에 80포트로 요청이 올 경우, 파드의 8080포트로 전달하는 걸 의미한다.

실제 온프레미스 환경에 클러스터가 구축되어서 로드밸런서 서비스가 제대로 동작하지 않을 수 있다. 이런 경우, metalLB같은 로드밸런싱 도구를 사용하여 문제를 해결할 수 있다.

### 외부 연결을 사용하면서 발상핼 수 있는 문제(불필요흔 네트워크 홉)
쿠버네티스 클러스터에서 `Nodeport` 서비스 타입을 사용하는 경우 불필요한 네트워크 홉이 발생할 수 있다.

Nodeport를 사용하는 경우 외부의 요청은 클러스터의 모든 노드에 열려있는 특정 포트를 통해 들어올 수 있다.
이 요청은 각각의 `kube-proxy`에서 `iptables`를 사용하여 적절하게 라우팅됩니다.

kube-proxy에 의해 요청이 파드로 전달 됐을 때, 해당 파드가 실행중이지 않을 수 있다. 이 때 `kube-proxy는 클러스터 네트워크를 통해 다른 노드에 있는 적절한 파드로 전달`한다. 이 과정을 네트워크 단계가 추가되는 `불필요한 네트워크 홉`이다.

이 문제를 해결하기 위해 서비스 스펙의 `externalTrafficPolicy` 필드를 `Local`로 설정한다. 
local로 설정한 경우, 연결을 수신한 노드에서 실행 중인 파드로만 요청을 전달할 수 있다.

하지만 이 옵션을 사용하는 경우, `로컬에 실핼중인 파드가 있는 노드를 선택` 하므로 한 노드의 파드가 여러 개 실핼중인 경우, `로드 밸런싱의 불균형`이 올 수 있다.
예를 들어 다음과 같은 클러스터가 있다.
- A 노드
  - Pod-A1
- B 노드
  - Pod-B1
  - Pod-B2
이렇게 되어 있는 경우, 요청이 들어왔을 때, A, B노드는 균등하게 분배되어 균등해 보이지만 실질적으로 파드 입장에서 보면 A1(50%), B1(25%), B2(25%)형태로 분배된다.

### 외부 연결을 사용하면서 발상핼 수 있는 문제(클라이언트 IP 보존)
먼저 SNAT(Source Network Address Translation)은 네트워크 트패릭의 소스 IP주소를 다른 IP주소로 변환하는 기술이다.
외부의 요청이 온 경우, 노드의 kube-proxy가 서비스의 파드로 요청을 전달한다. 이 때 요청의 소스 IP가 SNAT에 의해 노드의 IP주소로 변경한다.
그래서 쿠버네티스에선 외부에서 들어오는 트래픽을 처리할 때, SNAT를 통해 소스 IP 주소를 변경하여 클러스터 내부 네트워크와의 충돌을 방지한다.

하지만 요청의 소스 IP(클라이언트 실제 IP)를 알아야 하는 경우(웹서버의 IP로깅, 사용자 위치 기반 서비스 등) 문제가 발생할 수 있다. 이 문제를 해결하기 위해 `externalTrafficPolicy : Local` 옵션으로 설정하여 SNAT가 수행되지 않게 하면 해결할 수 있다.

### 인그레스
인그레스는 한 IP 주소로 수십 개의 서비스에 접근이 가능하도록 지원한다.

![image](./images/ingress.png)

그림 처럼 인그레스를 통해 한 개의 클라이언트 IP로 여러 개의 서비스에 접근하게 할 수 있다. 이는 ingress가 애플리케이션 계층에서 작동하므로 url과 같은 많은 자원을 확인할 수 있기에 가능하다.

### 인그레스 컨트롤러
인그레스 리소스를 동작하려면 인그레스 컨트롤러를 설치해야 한다.

```
# 인그레스 컨트롤러 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# 설치 확인
kubectl get pods -n ingress-nginx -o wide
```

![image](./images/get-ingress.png)

인그레스 설치를 확인한다.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubia
spec:
  ingressClassName: nginx
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /ho
        pathType: Prefix
        backend:
          service:
            name: kimho
            port:
              number: 80
```
그리고 이 매니페스트 파일을 통해 인그레스 리소스를 생성한다.
`kubia.example.com` 의 도메인을 가지고 도메인/ho로 요청한 경우 미리 정의된 `kimho` 서비스로 80포트로 요청한다는 내용이다.

![image](./images/etc-hosts.png)

로컬에서 도메인을 테스트할 수 있게 `/etc/hosts`에 `컨트롤러 ip 도메인`값을 추가 했다.

![image](./images/req-ingress.png)

요청을 보면 도메인에 접근한 경우 404 에러, /ho 경로로 접근한 경우 요청이 잘 된걸 확인할 수 있다.

### 인그레스 동작 방식
여기에 추가

### 레디니스 프로브
레디니스 프로브는 `주기적으로 호출되며 특정 파드가 클라이언트 요청을 수신할 수 있는지 결정`한다. 컨테이너의 레디니스 프로브가 성공을 반환하며 요청을 처리할 준비가 됐다는 의미이다.

레디니스 프로브 유형
- Exec 프로브
  - 컨테이너 상태를 프로세스의 종료 상태 코드로 결정
- HTTP GET 프로브
  - HTTP GET 요청을 보내고 응답의 상태코드에 따라 준비 여부를 확인하는 프로브(Heartbeat)
- TCP 소켓 프로브
  - 컨테이너의 지정된 포트로 TCP 연결을 확인해 준비 여부를 확인하는 프로브
  
컨테이너가 시작될 떄 바로 레디니스 프로브가 동작할 수 있지만 임의의 시간이 경과된 후 레디니스 프로브로 검사하게 할 수 있다 이는 `initialDelaySeconds`옵션을 통해 설정할 수 있다.
이후 주기적으로 프로브를 호출하여 파드가 준비되면 서비스(엔드포인트 리소스)에 추가된다.

그럼 라이브니스 프로브랑 어떤 다른점이 있을까?
라이브니스 프로브는 상태 점검에 실패하면 컨테이너를 제거하고 새로 만든다. 하지만 레디니스 프로브는 상태 점검에 실패하더라도 삭제되지 않고 상태 점검에 성공한 파드만 요청을 수신하도록 한다.

레디니스 프로브의 중요성
만약 파드 그룹이 다른 파드에서 제공하는 서비스에 의존하는 상황에서 문제가 발생한 경우, 레디니스 프로브를 통해 해당 파드가 요청을 처리할 준비가 되지 않음을 쿠버네시트에 알린다. 이를 통해 문제가 발생한 파드로 요청을 보내지 않고 정상 상태인 파드하고 통신하여 문제가 발생했음에도 사용자는 문제를 인식하지 못하고 안정적으로 서비스를 사용할 수 있다.

예를 들어 백엔드 애플레이케이션 서버가 데이터베이스 파드에 의존하는 상황에서 문제가 발생한 경우, 문제가 생긴 파드말고 정상적으로 동작하는 파드로 요청하여 안정적인 상태를 유지할 수 있다.

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kimho
  template:
    metadata:
      labels:
        app: kimho
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - name: http
          containerPort: 8080
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
```
레디니스 프로브는 컨테이너 내부에서
- spec
  - template
    - spec
      - containers
        - readinessProbe
          - exec
명령을 주기적으로 수행한다. 이 예시는 주기적으로 `ls /var/ready` 명령어를 실행한다.
ls 명령어는 파일이 있으면 종료 코드 0을 반환하고 아니면 0 이외의 값을 반환한다.

![image](./images/rc-result.png)

결과를 보면 이미 생성된 파드 1개를 제외하고 나머지 2개가 생성되었다. 하지만 실제 컨테이너는 실행하지 않았다. 이는 레디니스 프로브에 의해 아직 실패 상태이기 때문이다.

`k exec {pod-name} -- touch /var/ready` 명령을 통해 파일을 만들었지만 컨테이너는 바로 생성되지 않고 일정 시간뒤에 생성되었다. 레디니스 프로브는 기본 10초 마다 프로브를 실행하여 검사하므로 실제론 파일이 생성되었지만 아직 상태 검사가 이루어지지 않아 발생한 일이다.

마지막으로 레디니스 프로브는 항상 정의되어야 한다. 그렇지 않으면 `파드가 생성되는 즉시 서비스 엔드포인트가 되므로 요청을 수행 받을 준비가 되어 있지 않더라도 요청을 받을 수 있다.` 그래서 항상 레디니스 프로브를 정의헤야 한다.


