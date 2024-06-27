### 스테이트풀셋
스테이트풀셋은 쿠버네티스에서 상태를 가지는 애플리케이션을 관리하기 위한 워크로 API 오브젝트입니다.

디플로이먼트와 유사하게, 동일한 컨테이너 스펙(template)을 기반으로 둔 파드들을 관리합니다. 디플로이먼트와 다른점은 스테이트풀셋은 파드들의 순서 및 고유성을 보장합니다.

스테이트풀셋은 다음과 같은 기능을 가지고 있습니다.
- 고유한 네트워크 식별자
  - 스테이트풀셋에 속한 각 파드는 고유한 식별자를 가집니다.
  - 파드의 이름이 <pod-name>-0, <pod-name>-1과 같이 순차적으로 부여됩니다.
- 지속적인 스토리지
  - 각 파드는 고유한 퍼시스턴트 볼륨(PV)를 사용하여 데이터를 저장합니다. 이는 파드가 재시작 되더라도 데이터가 유지됩니다.
- 순차적 배포와 스케일링
  - 스테이트풀셋은 파드들을 순서대로 생성하고 배포하며, 확장할 때도 순차적으로 진행합니다.

![alt text](./images/cluster.png)

이런 기능을 가지는 스테이트풀셋은 다음과 같은 상황에서 유용합니다.
1. 고유한 네트워크 식별자가 필요한 경우(카산드라 클러스터 구축)
- 카산드라와 같은 분산 데이터베이스 클러스터는 각 노드가 고유한 네트워크 식별자를 필요로 합니다.
- 스테이트풀셋을 사용하면 각 파드는 cassandra-0, cassandra-1... 등으로 명명되며 이를 통해 클러스터 내에서 각 노드는 서로를 식별하고 통신할 수 있습니다.
- 고유한 네트워크 식별자를 제공하기 위해 헤드리스 서비스를 사용합니다.


![alt text](./images/statefulset-pv.png)
1. 지속적으로 스토리지가 필요한 경우(Mysql)
- MySQL 같이 관계형 데이터베이스는 데이터를 지속적으로 저장할 필요가 있습니다.
- 스테이트풀셋을 사용하면 각 mysql 파드는 고유한 PV(Retain)를 가지며, 파드가 재시작되거나 다른 노드로 이동하더라도 데이터는 보존됩니다.
- 각각의 파드는 mysql-0, mysql-pv-0, mysql-1, mysql-pv-1과 같이 각각의 PV를 사용합니다.

### 스테이트풀셋 생성
```YAML
# 헤드리스 서비스
apiVersion: v1
kind: Service
metadata:
  name: nginx-state
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
# 스테이트풀셋
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # .spec.template.metadata.labels 와 일치해야 한다
  serviceName: "nginx"
  replicas: 3 # 기본값은 1
  minReadySeconds: 10 # 기본값은 0
  template:
    metadata:
      labels:
        app: nginx # .spec.selector.matchLabels 와 일치해야 한다
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
  ```

- spec
  - selector
    - matchLabels
      - 스테이트풀셋이 관리하는 파드들을 선택하는 라벨 셀렉터
  - serviceName: 
    - 스테이트풀셋이 사용하는 헤드리스 서비스 이름
  - replicas: (기본값 1)
  - minReadySeconds:10 (기본값 0)
    - 파드가 준비 상태로 간주되기 전 파드의 모든 컨테이너가 문제 없이 실행되고 준비되는 최소 시간(초)
  - template(생성될 파드 템플릿)
    - metadata
      - labels
        - 생성될 파드의 레이블
    - spec
      - terminationGracePeriodSecond: (기본값 30)
        - 파드가 종료될 때 가지는 유예 시간(초)
      - containers
        - ...
        - volumeMounts
          - name
            - 볼륨 이름
            - mountPath
              - 마운트 경로
  - volumeClaimTemplates(파드에서 사용할 PVC 템플릿)
    - metadata
      - name
        - PVC 이름
        - 생성되는 PVC 이름은 <volumeClaimTemplates-name>-<statefullset-name>-<파드 인덱스> 형식
    - spec
      - accessModes
        - 볼륨 접근 권한
      - storageClassName
        - PVC가 사용할 스토리지 클래스 이름
      - resources(PVC의 리소스 요청(제한 X))
        - request:
          - storage
            - PVC에서 요청하는 스토리지 용량

실제 위 매니페스트 파일로 만들어진 결과입니다.
```
# PV
pvc-7393ff6b-1819-4080-afd0-a6154ec590a1   1Gi        RWO            Retain           Bound      default/www-web-1                                                                                     accordion-storage            70m
pvc-dbe5e5b1-6475-4ea7-ac0a-886dea58c486   1Gi        RWO            Retain           Bound      default/www-web-2                                                                                     accordion-storage            70m
pvc-fec4bd76-4743-4ada-97e6-aa85239636f4   1Gi        RWO            Retain           Bound      default/www-web-0

# PVC
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
www-web-0    Bound    pvc-fec4bd76-4743-4ada-97e6-aa85239636f4   1Gi        RWO            accordion-storage   70m
www-web-1    Bound    pvc-7393ff6b-1819-4080-afd0-a6154ec590a1   1Gi        RWO            accordion-storage   69m
www-web-2    Bound    pvc-dbe5e5b1-6475-4ea7-ac0a-886dea58c486   1Gi        RWO            accordion-storage   68m

# statefulset
NAME   READY   AGE   CONTAINERS   IMAGES
web    3/3     72m   nginx        registry.k8s.io/nginx-slim:0.8

# POD
NAME                                      READY   STATUS    RESTARTS   AGE   IP               NODE               NOMINATED NODE   READINESS GATES
web-0                                     1/1     Running   0          71m   172.32.146.91    hkim-host-master   <none>           <none>
web-1                                     1/1     Running   0          71m   172.32.103.27    hkim-host-node     <none>           <none>
web-2                                     1/1     Running   0          70m   172.32.146.101   hkim-host-master   <none>           <none>

# EndPoint
NAME                  ENDPOINTS                                                         AGE
nginx-state           172.32.103.27:80,172.32.103.46:80,172.32.146.101:80 + 2 more...   73m
```

위 예시는
1. 이름이 nginx-state라는 헤드리스 서비스는 네트워크 도메인을 컨트롤하는데 사용됩니다.

헤드리스 서비스는 파드와 직접적으로 통신하며, 통신을 하려면 다른 파드의 IP를 알아야합니다. FQDN을 사용하여 파드들의 IP를 조회해봅니다.
```
# 파트 접속
kubectl exec -it <pod-name> -- /bin/bash

# 헤드리스 서비스에 속한 파드 질의
nslookup nginx-state
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx-state.default.svc.cluster.local
Address: 172.32.146.91
Name:   nginx-state.default.svc.cluster.local
Address: 172.32.103.27
Name:   nginx-state.default.svc.cluster.local
Address: 172.32.146.101

kube-dns에 질의하여 nginx-state 헤드리스 서비스에 속한 IP 정보를 확인할 수 있습니다.
```

2. 스테이트풀셋에 의해 생성된 파드는 각각의 PV를 가집니다.
스테이트풀셋은 volueClaimTemplates를 설정하여 각각의 파드에서 동적 또는 정적으로 프로비저닝된 안정적인 스토리지를 사용할 수 있습니다.

### 파드 신원(특정 파드를 고유하게 식별할 수 있는 속성)

스테이트풀셋은 각 파드에 고유한 신원을 부여합니다. 이 말은 파드가 어떤 노드에 위치하든 동일한 신원을 유지합니다.

1. 순서 색인

N개의 레플리카가 있는 스테이트풀셋은 각 파드에 0 ~ N 까지의 정수가 순서대로 할당되고 해당 스테이트풀셋 내에서 고유합니다. 

```
pod-0
pod-1
pod-2
```
파드가 0~2번이 존재할 때, 레플리카가 2로 변경되면 가장 최근의 파드(pod-2)가 삭제되며 이후 새로 생성되면 2번부터 이어서 생성됩니다.

2. 네트워크 신원

스테이스풀셋의 각 파드는 스테이스풀셋의 이름과 파드 순번에서 `호스트 이름`을 얻습니다.

이 호스트 이름은 `<statefulset-name>-<ordinal>`입니다.

스테이트풀셋에 있는 파드의 도메인을 제어하기 위해 헤드리스 서비스를 사용할 수 있으며, 서비스가 관리하는 도메인은
- `<service-name>.namespace.svc.cluster.local` 형식입니다.

각 파드는 생성되면 `<pod-name>.<governing service domain>`형식의 서브도메인을 가지게됩니다.

```
# nslookup <service-name>.<namespace>.svc.cluster.local
# 서비스가 관리하는 도메인 질의
nslookup nginx-state

# 각 파드 질의
# nslookup <pod-name>.<governing-service-name>

# 결과
nslookup web-1.nginx-state
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-1.nginx-state.default.svc.cluster.local
Address: 172.32.103.27
```

3. 안정된 스토리지
스테이트풀셋에 정의된 VolumeClaimTemplate 항목마다 각 파드는 하나의 PVC를 받습니다. 각 파드는 스토리지 클래스와 프로비저닝된 PV를 받게 됩니다.

`스토리지 클래스를 명시 하지 않은 경우 기본 스토리지 클래스가 사용되며`, 기본 스토리지 클래스도 정의되어 있지 않으면 만족하는 PVC를 찾을 수 없어 정상적으로 동작되지 않습니다.

PVC, PV는 스테이트풀셋이 삭제되더라도 삭제되지 않습니다. 이는 반드시 관리자가 수동으로 조작해야 합니다.

실제로 스토리지 클래스를 명시하지 않을 경우 파드가 pending 상태이며, 이벤트가 다음과 같이 출력됩니다.

```
Warning  FailedScheduling  46s   default-scheduler  0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling..
```