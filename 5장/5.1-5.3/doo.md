## CI/CD : 지속적 통합과 지속적 배포

### CI

소스 코드를 커밋하고 빌드했을 때, 정상적으로 작동하는지 반복적으로 검증해 애플리케이션의 신뢰성을 높이는 과정
→ 개발자가 코드를 커밋하고 푸시하면, 애플리케이션이 자동 빌드되고 테스트를 거쳐 배포할 수 있는 애플리케이션인지 확인한다.

### CD

애플리케이션을 컨테이너 이미지로 만들어서 파드, 디플로이먼트, 스테이트풀셋 등 다양한 오브젝트 조건에 맞춰 미리 설정한 파일을 통해 배포하는 과정

## 배포 간편화 도구들

### kubectl - 큐브시티엘

쿠버네티스에 기본적으로 포함된 커맨드라인 도구

- 추가 설치 없이 바로 사용 가능하다.
- 가변적 환경에서 야믈 파일을 수정해야해서 대응이 가장 어렵다.

### kustomize - 커스터마이즈

- 오브젝트를 사용자의 의도에 따라 유동적으로 배포할 수 있다.
- 별도의 커스터마이즈 실행 파일을 활용해 커스터마이즈 명세를 따르는 야믈 파일을 생성할 수 있다.
- 운영중인 환경에서 배포 시, 가변적인 요소를 적용하는데 적합하다.

### helm - 헬름

- 오브젝트 배포에 필요한 사양이 이미 정의된 파트라는 패키지를 활용한다.
→ 헬름 차트 저장소가 따로 온라인이 있기 때문에 패키지를 검색하고 내려받아 사용하기가 매우 간편하다.
- 헬름 차트는 자체적인 템플릿 문법을 사용하므로 가변적인 인자를 배포할 때 적용해 다양한 배포 환경에 맞추거나 원하는 조건을 적용할 수 있다.

| 구분 | kubectl | kustomize | helm |
| --- | --- | --- | --- |
| 설치 방법 | 쿠버네티스에 기본 포함 | 별도 실행 파일 또는 쿠버네티스에 통합 | 별도 설치 |
| 배포 대상 | 정적인 야믈 파일 | 커스터마이즈 파일 | 패키지(차트) |
| 주 용도 | 오브젝트 관리 및 배포 | 오브젝트의 가변적 배포 | 패키지 단위 오브젝트 배포 및 관리 |
| 가변적 환경 | 대응 힘듦(야믈 수정 필요) | 간단한 대응 가능 | 복잡한 대응 가능 |
| 기능 복잡도 | 단순함 | 보통 | 복잡함 |

## 커스터마이즈 배포

커스터마이즈를 통한 배포는 kubectl에 구성돼 있는 매니페스트를 고정적으로 이용해야하는 기존 방식을 유연하게 만든다.

- 커스터마이즈는 야믈 파일에 정의된 값을 사용자가 원하는 값으로 변경할 수 있다.
- 오브젝트에 대한 수정사항을 반영하려면 사용자가 직접 편집기로 야믈 파일을 수정할 수도 있다. 
→ 하지만, 수정해야할 파일이 매우 많거나 하나의 야믈 파일로 다른 여러 개의 쿠버네티스 클러스터에 배포해야하는 경우엔 어려움이 있다.

[ 실습 ]

`kustomize create --namespace=metallb-system --resources r1.yaml,r2.yaml,r3.yaml`

- kustomize 명령을 이용하여 kustomization.yaml 이라는 기본 매니페스트를 만들어서 변경해야하는 값들을 적용한다.

`kustomize build` 

- build 옵션으로 변경할 내용이 적용된 최종 야믈 파일을 저장하거나 변경된 내용이 바로 실행되도록 지정한다
- `kustomize build | kubectl apply -f -` 명령을 통해 빌드한 결과를 바로 kubectl apply에 인자로 전달할 수도 있다.

## 헬름 배포

헬름 : 쿠버네티스에 패키지를 손쉽게 배포할 수 있도록 패키지를 관리하는 쿠버네티스 전용 패키지 매니저

- 커스터마이즈에서 변경할 수 없는 주소 할당 영역과 같은 값이 아닌 부분을 변경할 수 있다.
- 패키지 : 실행 파일 뿐만 아니라 실행 환경에 필요한 의존성 파일과 환경 정보들의 묶음
- 패키지 매니저 : 설치에 필요한 의존성 파일들을 관리하고 간편하게 설치할 수 있도록 도와주는 것
→ 자바의 maven, 리눅스의 apt, yum 등

[ 패키지 매니저 ]

- 패키지 검색 : 설정한 저장소에서 패키지를 검색하는 기능을 제공
- 패키지 관리 : 저장소에서 패키지 정보를 확인하고, 사용자 시스템에 패키지 설치, 삭제, 업그레이드, 되돌리기 등을 할 수 있다.
- 패키지 의존성 관리 : 패키지를 설치할 때 의존하는 소프트웨어를 같이 설치하고, 삭제할 때 같이 삭제한다.
- 패키지 보완 관리 : 디지털 인증서와 패키지에 고유하게 발행되는 체크섬이라는 값으로 해당 패키지의 소프트웨어나 의존성이 변조됐는지 검사한다.

[ 장점 ] 

- 다수의 오브젝트 배포 야믈을 분리하여 관리할 때, 각 디렉토리별로 야믈 파일을 만들고 kubectl apply -f 명령을 수행해야한다.
→ 헬름을 통해 요구 조건별로 리소스를 편집하거나 변수를 넘겨서 처리하는 패키지(차트)를 만들어 처리할 수 있다.
- 다양한 요구 조건을 처리할 수 있는 패키지인 차트를 헬름 저장소에 공개해 여러 사용자와 공유할 수 있다.
→ 헬름 기본 저장소를 ‘아티팩트허브’라고 한다.
- 배포한 애플리케이션을 업그레이드하거나 되돌릴 수 있는 기능, 삭제하는 기능을 제공한다.
- 하나의 패키지로 다양한 사용자가 원하는 각자의 환경을 구성할 수 있고, 이를 자유롭게 배포, 관리, 삭제할 수 있다.

[ 헬름의 전반적인 흐름 ] 

- 생산자 영역 : 헬름 명령으로 작업 공간을 생성하면 templates 디렉토리로 애플리케이션 배포에 필요한 여러 야믈 파일과 구성 파일을 작성할 수 있다.
- 아티팩트허브 영역 : 아티팩트허브 검색을 통해 사용자가 찾고자 하는 애플리케이션 패키지를 검색하면 해당 패키지가 저장된 주소를 확인한다.
- 사용자 영역 : 사용자가 설치하려는 애플리케이션 차트 저장소 주소를 아티팩트허브에서 얻으면 헬름을 통해 주소를 등록하고, 이를 최신으로 업데이트한 후에 차트를 내려받고 설치한다.

# 젠킨스

### taints

쉽게 접근하지 못하는 소중한 것.

키(key), 값(value), 효과(effect)의 조합을 통해 테인트를 설정한 노드에 파드 배치의 기준을 설정한다.

```yaml
# kubectl get node m-k8s -o yaml | nl
...
taints:
  - effect: NoSchedule  # taint와 toleration이 맞지 않을때의 동작
    key: node-role.kubernetes.io/master   # 어떤 노드가 마스터의 역할을 하는지 알려준다.
...
```

- 키와 값은 테인트를 설정한 노드가 어떤 노드인지를 구분하기 위해 사용한다.
→ 키는 필수로 설정되어야하지만, 값은 생략 가능하다.
- 효과는 taint와 toleration의 요소인 키 또는 값이 일치하지 않는 파드가 노드에 스케줄되려고 하는 경우에 어떤 동작을 할것인지 나타낸다.

[ effect에 따른 노드의 동작 ]

| 효과 | 노드에 파드 신규 배치되는 경우 | 노드에 파드가 이미 배치된 경우 |
| --- | --- | --- |
| NoSchedule | 노드에 파드 배치를 거부  | 노드에 존재하는 파드 유지 |
| PreferNoSchedule | 다른 노드에 파드 배치가 불가능할 때는 노드에 파드 배치 | 노드에 존재하는 파드 유지 |
| NoExecute | 노드에 파드 배치를 거부 | 파드를 노드에서 제거 |

### toleration

taint에 접근하기 위한 특별한 키

키(key), 값(value), 효과(effect), 그리고 연산자(operator)를 추가로 가진다.

```yaml
# kubectl get deployments jenkins -o yaml | nl
...
tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists
...
```

- taint가 설정된 노드로 들어가기 위한 특별한 열쇠의 역할을 하며, 키와 효과가 반드시 일치해야 한다.
- operator엔 Equal과 Exists가 있다.
    - Equal : taint와 toleration의 키와 값 그리고 효과까지 일치하는 경우
    - Exists : 키와 효과의 일치 여부를 판단하고, 이때 값은 반드시 생략해야한다.
    → 키와 효과를 모두 생략한 상태에서 Exists 연산자만 사용한다면, 묵시적으로 taint의 키와 효과는 모든 키와 모든 효과를 의미하게 되어 Exists 연산자 하나만으로 taint가 설정된 모든 노드에 대해서 설정할 수 있다.

### 서비스 어카운트

서비스 어카운트 : 쿠버네티스 클러스터 및 오브젝트의 정보를 조회하기 위한 계정

`kubectl create clusterrolebinding jenkins-cluster-admin --clusterrole=cluster-admin --serviceaccount=default:jenkins` 

- **역할 기반 접근 제어** : 유저가 속한 Role에 따라 접근 권한을 결정하는 방식. 즉, 정보에 대한 접근 권한이 역할에 따라 배정된다.

### 쿠버네티스 역할 부여 구조

쿠버네티스 역할 부여 구조는 `할 수 있는 일`과 `할 수 있는 주체`의 결합으로 이루어진다.

- Rule : 할 수 있는 일과 관련된 Role, ClusterRole이 가지고 있는 자세한 행동 규칙이다. Rules는 apiGroups, resources, verbs 속성을 가진다.
    - apiGroups : 접근할 수 있는 api 집합
    - resources : API 그룹에 분류된 자원 중 접근 가능한 자원 집합
    - verbs : resources에 정의된 자원에 대해 할 수 있는 행동들. 
    → ex) get, list, create, delete 등..
- Role, ClusterRole : 할 수 있는 일을 대표하는 오브젝트. Rules에 적용된 규칙에 따른 행동을 할 수 있다.
    - Role : 해당 Role을 가진 주체가 특정 namespace에 접근할 수 있다.
    - ClusterRole : 해당 ClusterRole을 가진 주체가 쿠버네티스 클러스터 전체에 접근할 수 있다.
- RoleBinding, ClusterRoleBinding : Role,ClusterRole과 Subjects를 연결시켜주는 역할
    - RoleBinding : Role과 결합하여 namespace 범위의 접근 제어를 수행
    - ClusterRoleBinding : ClusterRole과 결합하여 클러스터 전체 범위의 접근 제어를 수행
- Subjects : 역할 기반 접근 제어에서 행위를 수행하는 주체. 사용자 혹은 그룹, 서비스 어카운트를 속성으로 가질 수 있다.
    - 사용자 : 쿠버네티스에 접근을 수행하는 실제 이용자
    - 서비스 어카운트 : 파드 내부의 프로세스에 적용되는 개념