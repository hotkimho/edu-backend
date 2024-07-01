
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
