### 외부 사용자가 파드를 이용하는 방법

service: 외부에서 쿠버네티스 클러스터에 접속하는 방법

- NodePort Service

1) 모든 워커 노드의 특정 포트를 열고 
2) 여기오는 모든 요청을 노드포트 서비스로 전달
3) 노드포트 서비스는 해당업무를 처리할 수 있는 파드로 요청을 전달

[사진]
실습 
1) 파드 생성
```
kubectl create deployment np-pods --image=(해당 이미지)
```

2) 파드 확인
```
kubectl get pods
```

3) service 생성
```
kubectl create -f (*.yaml:해당하는 서비스 정보)
```

4) 생성 확인
```
kubectl get services
```

6) 워커노드 ip 확인
```
kubectl get nodes -o wide
```

7) web browser 에서 ip 입력 및 파드 이름 확인

어떻게 추가된 파드를 외부에서 추적해 접속하는 것일까?
-> 노드 포트의 오브젝트 스펙에 적힌 np-pods와 deployment의 이름을 확인하여 일치하면 동일 파드라고 인식하였다.
이외에도 여러 기준을 가지고 동일 파드라고 인식시킬 수 있다고 한다.

expose 로 노드포트 service 생성하기

오브젝트 spec file이외에도 expose 키워드를 이용하여 생성할 수 있다.
```
kubectl expose deployment np-pods --type=NodePort --name=np-svc-v2 --port=80
kubectl get services
```

* 주의할 점: 오브젝트 스펙으로 만드는 service와 달리 NodePort 번호를 사용자가 임의 지정 불가능(30000~32767 사이의 임의의 수가 지정됨)

여기서 한가지 특징이 나타나는 데 노드포트 서비스는 포트 중복 사용이 불가능하다는 점이다. 

1개의 노드포트 <=> 1개의 deployment 

그렇다면 여러개의 deployment를 사용해야한다면 어떻게 해야할까? Ingree를 사용하면 된다.
Ingress 특징
- 고유한 주소 제공, 사용목적에 따른 다른 응답 제공
- 트래픽에 대한 L4/L7 load balancer와 보안 인증서 처리기능 제공

Ingress를 사용하기 위해서는 Ingress Controller가 필요하다
우리는 여러 controller중에서 nginx Ingress Controller를 사용하도록 하겠다.

1) 사용자가 노드포트를 통해서 노드포트서비스(줄여서 노포서라고 지칭하겠다.)에 접속한다.이때 노포서는 nginx controller이다.
2) nginx는 사용자 접속 경로에 따라 적합한 클러스터 ip service로 경로 제공
3) 클러스터 ip service는 사용자를 해당 파드로 연결

실습
1) Deployment 2개 배포 
2) 파드 상태확인
3) controller install
4) ingress controller pod 확인
5) 사용자 요구 사항에 맞게 경로와 작동 정의
6) 등록 확인
7) ingress에 요청 내용 적용 확인
8) 노포서로 nginx ingress 외부 노출
9) kubeclt get services -n ingress-nginx
10) expose로 deployment도 서비스로 노출

### Load Balancer Service

노드포트를 이용한 방식은 굉장히 비효율적이라고 한다.

그래서 쿠버네티스는 Load Balancer라는 서비스 타입을 제공 -> 간단한 구조로 파드를 외부에 노출 및 부하 분산

로드밸런서를 사용하기 위해서는 이미 구현해둔 서비스 업체의 도움을 받아 쿠버네티스 클러스터 외부에 구현해야하기 때문이다.

### 온프레미스에서 로드밸런서를 제공하는 MetaLB

온프레미스에서 로드밸런서를 사용하려면 내부에 로드밸런서 서비스를 받아주는 구성이 필요한데 이를 지원하는 것이 MetaLB이다.

특별한 네트워크 설정이나 구성없이 기존의 L2, L3네트워크로 로드밸런서를 구현

따라서 네트워크를 새로배워야할 부담x, 연동이 쉬움

책에서는 L2 네트워크로 로드밸런서를 구현, 테스트 목적으로 2개의 MetaLB로드밸런서 서비스를 구현한다

1) 디플로이먼트를 이용해 2 종류의 파드를 생성, scale은 3으로 지정하여 노드당 1개씩 파드가 배포되게 한다.
2) 2종류의 파드가 3개씩 총 6개가 배포됐는지 확인
3) 인그레스와 마찬가지로 사전에 정의된 오브젝트 스펙 사용(namespace==metallb-system으로 생성됨)
4) 배포된 metallb의 파드가 5개인지 확인, ip상태도 확인
5) 인그레스와 마찬가지로 MetalLB도 설정을 해야하기 때문에 ConfigMap을 사용합니다.
6) ConfigMap이 생성되었는지 확인
7) -o yaml옵션을 주고 다시 실행해 설정 적용 확인
8) 디플로이먼트를 로드밸런서 서비스로 노출
9) 생성된 로드밸런서 서비스별로 cluster-ip와 external-ip가 잘 적용됐는지 확인
10) 실행

부하에 따라 자동으로 파드 수를 조절하는 HPA(Horizontal Pod Autoscaler)

실습
1) 디플로이먼스 생성
2) 앞에서 구성한 MetalLB로 expose명령어를 통해 로드벨런서 서비스를 바로 설정
3) 설정된 로드밸런서 서비스와 부여된 ip 확인
4) 파드의 자원이 어느정도인지 확인
=> 이때 metrics-server가 없으면 에러가 발생
5) 메트릭 서버도 오브젝트 스펙 파일로 설치할 수 있다.
6) 다시 4)를 반복
7) 처음에 생성한 deployment에 autoscale을 설정하여자동으로 scale 명령이 수행되도록 하자

## 알아두면 쓸모 있는 쿠버네티스 오브젝트

### 데몬셋

데몬셋 : deployment의 replicas가 노드수만큼 정해져있는 형태, 노드하나당 파드한개만을 생성한다.

=> 노드를 관리하는 파드라면 데몬셋으로 만드는것이 가장 효율적이다.

### ConfigMap

ConfigMap: 이름 그대로 설정을 목적으로 이용하는 오브젝트

Ingress 때에는 오브젝트를 인스레스로 선언하였지만 MetalLB에서는 configmap을 사용하였다. 그 이유는 ingress와 달리
metallb는 정해진 프로젝트 타입이 따로 없기 때문에 configmap을 사용해되 되기 때문이다.

### PV와 PVC
PVC(PersistentVolumeClaim):지속적으로 사용가능한 볼륨요청

PVC를 사용하기 위해서는 PV로 볼륨을 선언해야한다.
간단하게 이야기하면 pv는 볼륨을 사용할 수 있게 준비하는 단계이고 pvc는 준비된 볼륨에서 일정 공간을 할당받는 것이다

책에서는 nfs 서버를 마스터 노드에 구성하고 볼륨을 pv로 선언할 볼륨을 만듭니다

### 스테이트 풀셋

지금까지는 파드가 replicas에 선언된 만큼 무작위로 생성될 뿐이다. 파드가 만들어지는 이름과 순서를 예측해야할 때가 있다. 주로 마스터슬레이브
주로 마스터-슬레이브 구조 시스템에서 필요, 이때 스테이트풀셋이 사용된다.

스테이트풀셋은 volumeClaimTemplates기능을 사용해 pvc를 자동으로 생성할 수 있고, 각파드가 순서대로 생성되기 때문에 고정된 이름, 볼륨, 설정들을 가질 수 있다.

그래서 StatefulSet이라고 한다. 하지만 효율성면에서 좋은 구조는 아니기 때문에 요구사항에 맞게 적절히 사용하는 것이 좋다

일반적으로 스테이트풀셋은 volumeClaimTemplates를 이용해 자동으로 각 파드에 독립적인 스토리지를 할당해 구성할 수 있다
