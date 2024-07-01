
# 쿠버네티스 아키텍쳐
![image](./images/architecture.png)

## 컨트롤 플레인(마스터 노드)
컨트롤 플레이은 클러스터 기능을 제어하고 전체 클러스터가 동작하게 만드는 역할을 합니다.
구성 요소는 다음고 같습니다.
- etcd(분산 저장 스토리지)
- API 서버
- 스케줄러
- 컨트롤러 매니저


이 구성요들은 클러스터 상태를 저장하고 관리하지만, 애플리케이션 컨테이너를 직접 실행하진 않습니다.

### etcd
- 클러스터의 모든 정보를 영구히 저장하는 (key, value) 저장소
- 둘 이상(홀수)의 etcd 인스턴스를 실행하여 고가용성을 보장할 수 있음
- 쿠버네티스 구성 요소 중 API를 통해서만 직접적으로 접근 가능
- RAFT 합의 알고리즘 사용

### etcd 데이터 조회
```
# 조회할 파드(default namespace)
NAME                           READY   STATUS    RESTARTS   AGE     IP               NODE               NOMINATED NODE   READINESS GATES
nginx                          1/1     Running   0          124m    172.32.146.127   hkim-host-master   <none>           <none>
```

etcdctl을 사용하여 etcd에 위에서 조회한 파드의 데이터를 조회합니다.([etcdctl](https://etcd.io/docs/v3.4/dev-guide/interacting_v3/))

![alt text](./images/etcd-get-pod-key.png)

파드가 etcd에 존재하는지 확인했습니다. 이제 상세 데이터를 조회해봅니다.

![alt text](./images/etcd-get-pod-data.png)

### 클러스터링된 etcd의 일관성 보장
고가용성을 보장하기 위해 2개 이상(홀수)의 etcd 인스턴스를 실행하는 것이 일반적입니다.

RAFT 합의 알고리즘을 사용하여, 대다수의 노드가 현재 상태(데이터 일관)를 보장합니다.

RAFT 합의 알고리즘은 다음 상태(데이터 변경)가 되기 위해서는 과반수가 필요합니다.

![alt text](./images/etcd-raft.png)

### API 서버
- 다른 모든 구성 요소와 통신하며, RESTful API로 상태에 대해 CRUD 가능(etcd에 저장)
- 두 개 이상의 인스턴스를 실행하여 고가용성 보장할 수 있음

![image](./images/api-server.png)

API 서버는 다음과 같은 과정을 진행합니다.

1. 인증 플러그인으로 클라이언트 인증 

먼저 API 서버는 요청을 보낸 클라이언트를 인증합니다. 이 작업은 API 서버에 구성된 하나 이상의 플러그인으로 수행됩니다.
- 인증서(Client Certificate(x509))
- 토큰
- 사용자 이름과 비밀번호(권장 X)


2. 인가 플러그인을 통한 클라이언트 인가

API 서버는 하나 이상의 인가 플러그인을 사용하도록 설정되어 있습니다.(기본은 rbac) 이 작업은 인증된 사용자가 요청한 작업이 리소스에 수행 가능한지 확인합니다.
- ABAC
- RBAC
- Webhook

3. 어드미션 컨트롤 플러그인으로 요청 리소스 확인 및 수정

생성, 수정, 삭제 하려는 요청인 경우 어드미션 컨트롤로 보내지며, 어드미션 컨트롤러에 의해 다음과 같은 기능이 수행됩니다.
- 누락된 필드 기본값으로 생성
- 요청에 없는 관계된 리소스를 수정하거나 요청을 거부

어드미션 컨트롤러 종류는 다음과 같습니다.
- AlwaysPullImages: 파드의 imagePullPolicy를 Always로 변경해 파드가 배포될 때 마다 이미지를 가져오도록 재정의
- ServiceAccount: 명시적으로 서비스어카운트를 지정하지 않은 경우 default 적용
- NamespaceLifecycle: 삭제되는 과정에 있는 네임스페이스와 존재하지 않는 네임스페이스 안에 파드 생성 방지
- LimitRanger: 기본 limitrange가 정의되어 있는 경우, 리소스 요청과 제한을 위반하는지 확인하며, 기본 리소스를 설정

### API 서버 낙관적 잠금
낙관적 잠금은 데이터 변경이 발생할 때 리소스버전(metadata.resourceVersion)이 etcd에 있는 리소스버전과 일치해야 데이터가 변경됩니다.

리로스버전이 일치하지 않은 경우 내용은 거부되고(409 status code)를 반환합니다.

metadata.resourceVersion은 모든 쿠버네티스 리소스에 존재합니다.
