# 컨테이너 인프라 환경의 CI/CD

### CI (Continuous Integration)
* 코드를 커밋하고 빌드했을 때 정상적으로 작동하는지 반복적으로 검증하여 애플리케이션의 신뢰성을 높이는 작업이다.
* 개발자가 코드를 커밋하고 푸시하면 CI 단계로 진입하며, 자동으로 빌드하고 테스트를 거쳐 배포할 수 있는 신뢰성 있는 애플리케이션인지 검증한다.
* CI 단계를 통해 검증이 끝났으면 CD 로 넘어간다.

### CD (Continuous Deployment)
* 애플리케이션을 컨테이너 이미지로 만들어서 파드, 디플로이먼트 등 다양한 오브젝트 조건에 맞추어 미리 설정한 파일을 통해 배포한다.

## 배포 간편화 도구들

### kubectl
* 쿠버네티스에 기본으로 포함된 커맨드라인 도구이며, 추가 설치 없이 바로 사용할 수 있다.
* 오브젝트 생성과 쿠버네티스 클러스터에 존재하는 오브젝트, 이벤트 등의 정보를 확인할 수 있어서 사용하는 활용도가 높다.
* 정의된 Manifest 파일을 그대로 배포하기 때문에 개별적인 오브젝트를 관리하거나 배포할 때 사용하는 것이 좋다.

### kustomize
* kustomization 파일을 통해 쿠버네티스 오브젝트를 사용자의 필요에 따라 변경하는 도구이다.
* 별도의 kustomize 실행 파일을 활용하여 kustomize 명세를 따르는 yaml 파일을 생성할 수 있다.
* kubectl 로도 yaml 파일을 통해 배포할 수 있는 옵션인 (-k)를 제공한다.
* 명령어로 배포 대상 오브젝트 이미지 태그와 레이블 같은 명세를 변경하거나 일반 파일을 이용해 Config Map과 Secret을 생성하는 기능을 지원한다.
* 운영중인 환경에서 배포 시 가변적인 요소를 적용하는데 적합하다.

### Helm
* Kubernetes 패키지 관리툴이다.
* 오브젝트 배포에 피룡한 사양이 이미 정의된 차트(Chart)라는 패키지를 활용한다.
* 헬름 차트는 자체적인 템플릿 문법으로 사용하므로 가변적인 인자를 배포할 때 적용해 다양한 배포 환경에 맞추거나 원하는 조건을 적용할 수 있다.

## Kustomize 실습해 보기
* 쿠버네티스에서 오브젝트에 대한 수정 사항을 반영하려면 사용자가 직접 yaml 파일을 수정해야 한다.
* 커스터마이즈는 yaml 파일에 정의된 값을 사용자가 원하는 값으로 변경할 수 있다.
* kustomize 명령과 create 옵션으로 kustomizaion.yaml 이라는 기본 manifest 파일을 민들고, 이 파일에 변경해야 하는 값을 적용할 수 있다.

```shell
# kustomize create 명령을 통해 kustomization.yaml 파일을 생성할 수 있다.
> kustomize create --namespace=metallb-system --resources namespace.yaml,metallb.yaml,metallb-l2config.yaml
```
* --namespace는 작업의 네임스페이스를 설정한다.
* --resources는 kustomize 명령을 이용해서 kustomization.yaml 을 만들기 위한 소스 파일을 정의한다.
```yaml
# 아래는 위 명령을 통해 생성된 kustomization.yaml 내용이다.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - metallb.yaml
  - metallb-l2config.yaml
namespace: metallb-system
````


* kustomize edit set image 옵션을 통해 태그를 지정할 수 있다.
```shell
> kustomize edit set image metallb/controller:v0.8.2
> kustomize edit set image metallb/speaker:v0.8.2
```
```yaml
# 아래는 위 명령어를 통해 수정된 kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace.yaml
- metallb.yaml
- metallb-l2config.yaml
namespace: metallb-system
images:
- name: metallb/controller
  newTag: v0.8.2
- name: metallb/speaker
  newTag: v0.8.2
```

* kustomize build 명령으로 MetalLB 설치를 위한 Manifest 를 생성할 수 있다.
* kubectl apply -f 를 통해 빌드된 Manifest 를 적용할 수 있다.
* 아래의 명령에서는 생성된 Manifest 의 결과가 kubectl apply 인자로 전달되도록 배포하는 방법이다.
```shell
> kustomize build | kubectl apply -f -
```

## Helm 실습해 보기
* 헬름은 쿠버네티스에 패키지를 손쉽게 배포할 수 있도록 도와주는 패키지 매니저이다.
* 다양한 요구 조건을 처리할 수 있도록 리소스를 편집하거나 변수를 넘겨서 패키지를 만들 수 있는데, 이를 차트라고 한다.
* 헬름의 기본 저장소는 artifacthub.io 이다.

```shell
# 아래 명령을 통해 헬름 차트 저장소를 추가한다.
> helm repo add <name> <repo uri>

# 아래의 명령을 통해 헬름 차트 저장소 목록을 확인할 수 있다.
> helm repo list

# 아래의 명령을 통해 저장소가 추가된 이휴에 변경된 차트가 있다면 변경된 정보를 캐시에 업데이트하여 최신 차트 정보를 동기화한다.
> helm repo update
```
* helm 차트를 설치할 때는 helm install 을 이용하면 된다.
```shell
# helm install <release name> [-f <config path>] [--namespace=<namespace name>] [--set <key>=<value>,...]
helm install metallb edu/metallb \
> --namespace=metallb-system \
> --create=namespace \
> --set controller.tag=v0.8.3 \
> --set speaker.tag=v0.8.3 \
> --set configmap.ipRange=192.168.56.11-192.168.56.29
```
* --namespace: 헬름 차트를 통해서 생성되는 애플리케이션이 위치할 네임스페이스를 지정한다.
* --create-namespace: 네임스페이스 옵션으로 지정된 네임스페이스가 존재하지 않는 경우 네임스페이스를 생성한다.
* --set: 헬름에서 사용할 변수를 명령 인자로 전달한다. key1=value1, key2=value2 와 같이 ,(쉼표)를 사용하여 한 줄에서 여러 인자를 넘겨줄 수도 있다.

```shell
# 헬름에 필요한 변수를 확인하는 명령어
> helm show values <차트>
# or
> helm inspect values <차트>
```
```shell
# 차트 설치 후, 설정 내용을 수정하여 다시 배포한다.
# helm upgrade <release name> [-f config path] <chart>
> helm upgrade metallb edu/metallb \
> --namespace=metallb-system \
> --set configmap.ipRange=192.168.56.11-192.168.56.29
```
```shell
# 설치된 차트 목록을 검색한다.
> helm ls

# 모든 네임스페이스의 차트 목록을 검색한다.
> helm ls --all-namespace
# or
> helm ls -A
```
```shell
# 설치된 차트를 제거한다.
# helm uninstall <release name>
> helm uninstall metallb
```
