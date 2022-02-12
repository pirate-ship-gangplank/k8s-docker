# Jenkins
* Jenkins 는 CI/CD 서비스를 제공해주는 대표적인 툴이다.
* 빌드 파이프라인을 구성하여 테스트를 자동화하거나 각종 배치 작업을 스케줄링 할 수 있다.
* 파이프라인을 통해 빠르면서도 신뢰성 있는 어플리케이션을 지속적으로 통합/배포 할 수 있다.
* 각종 플러그인을 풍부하게 지원하기 때문에 확장성 있는 CI/CD 구축이 가능하다.

### Item
* Item 이란 젠킨스의 작업 단위이다.
* Freestyle project, Pipeline 등 여러 아이템이 존재한다.

### Pipeline
* 가장 많이 쓰이는 프로젝트로, 고유의 문법으로 작성된 파일을 통해 Pipeline-As-Code 를 구현할 수 있다.
* 크게 2가지 문법으로 코드를 작성할 수 있는데, 하나는 스크립트 문법이고 다른 하나는 선언적인 문법이다.
* Jenkinsfile 파일에 배포 파이프라인을 정의한다.

#### Jenkinsfile 구성요소
* pipeline: 선언적 문법의 시작 부분이다.
* agent: 작업을 수행할 에이전트를 지정하고 필요한 설정을 한다. 지정된 에이전트 내부에서 젠킨스 빌드 작업이 실제로 수행된다.
* stages: stage 들이 정의된 작업을 순서대로 진행하도록 설정합니다.
* stage: step 들을 정의하는 영역이고, 젠킨스에서 빌드가 될 때 stage별로 진행 단계를 확인할 수 있다.
* steps: stage 내부에서 실제 작업 내용을 작성하는 영역이다. 해당 영역에서 script, sh, git 과 같은 작업을 통해서 동작한다.

**[컨테이너 인프라 환경을 위한 쿠버네티스/도커 Jenkins 예제](http://www.kyobobook.co.kr/product/detailViewKor.laf?ejkGb=KOR&mallGb=KOR&barcode=9791165215743&orderClick=LEa&Kc=)**
```
pipeline {
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: blue-green-deploy
        name: blue-green-deploy
      spec:
        containers:
        - name: kustomize
          image: sysnet4admin/kustomize:3.6.1
          tty: true
          volumeMounts:
          - mountPath: /bin/kubectl
            name: kubectl
          command:
          - cat
        serviceAccount: jenkins
        volumes:
        - name: kubectl
          hostPath:
            path: /bin/kubectl
      '''
    }
  }
  stages {
    stage('git scm update'){
      steps {
        git url: 'https://github.com/icarus8050/blue-green.git', branch: 'main'
      }
    }
    stage('define tag'){
      steps {
        script {
          if(env.BUILD_NUMBER.toInteger() % 2 == 1){
            env.tag = "blue"
          } else {
            env.tag = "green"
          }
        }
      }
    }
    stage('deploy configmap and deployment'){
      steps {
        container('kustomize'){
          dir('deployment'){
            sh '''
            kubectl apply -f ./configmap.yaml
            kustomize create --resources ./deployment.yaml
            echo "deploy new deployment"
            kustomize edit add label deploy:$tag -f
            kustomize edit set namesuffix -- -$tag
            kustomize edit set image sysnet4admin/dashboard:$tag
            kustomize build . | kubectl apply -f -
            echo "retrieve new deployment"
            kubectl get deployments -o wide
            '''
          }
        }
      }    
    }
    stage('switching LB'){
      steps {
        container('kustomize'){
          dir('service'){
            sh '''
            kustomize create --resources ./lb.yaml
            while true;
            do
              export replicas=$(kubectl get deployments \
              --selector=app=dashboard,deploy=$tag \
              -o jsonpath --template="{.items[0].status.replicas}")
              export ready=$(kubectl get deployments \
              --selector=app=dashboard,deploy=$tag \
              -o jsonpath --template="{.items[0].status.readyReplicas}")
              echo "total replicas: $replicas, ready replicas: $ready"
              if [ "$ready" -eq "$replicas" ]; then
                echo "tag change and build deployment file by kustomize" 
                kustomize edit add label deploy:$tag -f
                kustomize build . | kubectl apply -f -
                echo "delete $tag deployment"
                kubectl delete deployment --selector=app=dashboard,deploy!=$tag
                kubectl get deployments -o wide
                break
              else
                sleep 1
              fi
            done
            '''
          }
        }
      }
    }
  }
}
```

---
# 배포 전략

### Blue-Green
* 새 어플리케이션을 배포하기 위해 이전 버전을 blue 버전으로, 새 버전은 green 버전으로 나누어 정의한다.
1. green 버전이 완전히 배포될 떄까지 앞 단에서 LB가 blue 환경으로 트래픽을 전달한다.
2. green 버전이 완전히 기동되면 LB 는 blue 에 라우팅하던 트래픽을 green 으로 전달한다.
3. 만약 green 에 문제가 발생 시, blue 버전으로 다시 원복한다.
4. green 에 문제가 없다면 blue 버전은 셧다운되고, green 이 새로운 blue 버전이 된다.

#### Blue-Green
* 이전 버전, 현재 버전이 동시에 떠 있는 시간을 매우 짧게 처리할 수 있다.
* 롤백을 신속하게 진행할 수 있다.
* 배포 과정에서 인스턴스 수가 줄지 않으므로 요청량을 처리하는 데서 오는 장애의 부담이 적다.
* 배포를 위해서는 시스템 자원이 두 배로 필요하다는 단점이 있다.

### Rolling Update
* 사용중인 자원 내에서 새 버전을 점진적으로 배포해 나가며 무중단 배포한다.
* 쿠버네티스에서 기본적으로 사용하는 배포 방법이다.
* 애플리케이션 마다 순차적으로 배포하므로 요구되는 시스템 자원이 Blue-Green 보다 훨씬 적다.
* 배포가 진행되는 동안 이전 버전과 현재 버전이 동시에 공존하는 시간이 길어지므로 호환성 문제가 발생할 수 있다.

### Canary
* 신버전의 애플리케이션을 띄워놓고, LB를 통해 구버전과 신버전의 제공 범위를 조절해가면서 모니터링 및 피드백을 거쳐가며 배포를 할 수 있다.
* 두 버전에 대해 단계적인 비율 조절을 통해 상황에 트래픽을 조절하고, 롤백을 할 수도 있다.
* 롤링 업데이트와 마찬가지로 이전 버전과 현재 버전이 동시에 공존하는 시간이 길어지므로 호환성 문제에 주의해야 한다.

---
# GitOps
* 애플리케이션에서 배포와 운영에 관련된 모든 요소를 코드화하여 Git에서 관리(Ops)하는 것이다.
* 코드를 이용하여 인프라를 프로비저닝 하고 관리하는 IaC(Infrastructure as Code)에서 나온 것으로 깃옵스에서는 인프라에서 전체 애플리케이션 범위로 확장했다.
* 선언형으로 코드를 작성하고 업데이트하면 오퍼레이터(like Jenkins..)가 변경을 감지해서 대상 시스템에 배포한다.
* 깃옵스를 활용하면 코드 저장소의 내용과 실제 운영 환경의 내용을 동일하게 설정할 수 있다.
  * 덕분에 모든 내용을 단일화하여 관리할 수 있고, 히스토리 추적에 용이하다.
  * 문제가 발생하면 빠르게 복원할 수 있다.
* 배포를 표준화하여 자동화 시킬 수 있고, 이로인해 휴먼에러를 줄일 수 있다.

### 배포 전략
#### Push Type
* 저장소에 있는 매니페스트가 변경되었을 때 배포 파이프라인을 실행시키는 구조이다.
* 아키텍처가 단순하여 인기있는 방법이며, 젠킨스, 서클CI 등 구현에 사용할 수 있는 도구들도 다양하다.
* 푸시 타입의 경우 배포 환경에 대한 자격 증명 정보를 외부에서 관리하여 읽기 쓰기 권한을 부여하기 때문에 증명 정보가 노출될 큰 피해가 생길 수 있다.

#### Pull Type
* 배포 환경에 위치한 오퍼레이터가 배포 파이프라인을 대신한다.
* 오퍼레이터는 저장소의 매니페스트와 배포 환경을 지속적으로 비교하다가 diff가 발생하면 저장소의 매니페스트를 기준으로 배포 환경을 유지시킨다.
* 만약 누군가가 직접 허가 받지 않은 작업으로 배포 환경을 변경했을 경우에는 다시 되돌려지게 되는데, 이는 배포 환경의 변경은 저장소의 매니페스트를 통해서만 유효하다는 것을 보장한다.
* 아르고CD, 젠킨스X, 플럭스 도구를 사용할 수 있다.
* 오퍼레이터가 애플리케이션과 동일한 환경에서 동작중이기 때문에 자격 증명 정보가 외부에 노출될 일이 없고, 배포 환경에 대한 자격 증명 정보도 관리 비용을 최소화 할 수 있다.
