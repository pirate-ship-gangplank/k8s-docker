# 쿠버네티스 클러스터와 외부 네트워크

## 서비스
* 외부에서 쿠버네티스 클러스터에서 실행중인 파드에 접근할 수 있도록 네트워크를 노출하는 추상화 방법.
* 서비스 덕분에 서비스 디스커버리 메커니즘을 사용하기 위해 애플리케이션을 수정할 필요가 없다.
  * 쿠버네티스는 파드에게 고유한 IP주소와 파드 집합에 대한 단일 DNS 명을 부여하고, 그것들 간에 로드밸런싱을 수행할 수 있다.

### 노드포트 (NodePort)
* 고정 포트로 각 노드의 IP에 서비스를 노출시키고, 해당 포트로 들어오는 모든 요청을 노드포트 서비스로 라우팅한다.
* 노드포트 서비스가 라우팅되는 ClusterIP 서비스가 자동으로 생성된다.
  * ClusterIP : 서비스를 클러스터 내부 IP에 노출시킨다. 해당 타입은 클러스터 내에서만 서비스에 도달할 수 있다. ServiceTypes의 기본값.
```yaml
# 노드포트 Spec Example
apiVersion: v1
kind: Service
metadata:
  name: np-svc
spec:
  selector:
    app: np-pods
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30000
  type: NodePort
```
![NodePort](./images/node_port.png)
* nodePort: 워커노드에서 노출되는 포트 (옵셔널한 설정이며, 기본적으로 쿠버네티스 컨트롤 플레인은 30000-32767 범위 내에서 포트를 할당한다.)
* port: 클러스터 내부에서 사용할 서비스 객체의 포트 (편의상 targetPort와 port는 같은 값으로 설정한다.)
* targetPort: 서비스 객체로 전달된 요청을 파드로 전달할 때 사용하는 포트
* 서비스 셀렉터와 일치하는 파드를 지속적으로 검색하고, "np-pods"라는 엔드포인트 오브젝트에 대해 이름이 동일하다면 같은 파드로 간주하여 여러 파드에 부하를 분산시킬 수 있다.

### expose 명령어
* 노드포트 서비스는 오브젝트 스펙 파일이 아닌, expose 명령어로 생성할 수도 있다.
```shell
> kubectl expose deployment np-pods --type=NodePort --name=np-svc-v2 --port=80
```
* 서비스 이름: np-svc-v2
* type: NodePort
* port: 80
* expose를 사용하면 노드포트의 포트 번호를 지정할 수 없다.
  * 포트번호는 30000~32767 에서 임의로 선택된다.

### 인그레스(Ingress), 인그레스 컨트롤러 (Ingress Controller)
* 클러스터 내의 서비스에 대한 외부 접근을 관리하는 API 오브젝트. (일반적으로 HTTP를 관리함)
* 외부의 요청을 처리할 수 있는 NodePort, ExternalIP는 일반적으로 L4에서 처리하며, L7 에서의 요청을 처리할 수 없다.
* 인그레스는 고유한 주소를 제공해 사용 목적에 따라 다른 응답을 제공할 수 있고, 트래픽에 대한 L4/L7 로드밸런서와 SSL을 처리하는 기능을 제공한다.
* 인그레스는 규칙을 정의하는 선언적인 오브젝트일 뿐, 외부의 요청을 받아들이는 실제 서버가 아니다.
* 인그레스 컨트롤러라고 하는 특수한 서버 컨테이너에 적용되어야 인그레스에 적용된 규칙이 활성화 된다.
* 사용하는 환경에 따라 Nginx 기반의 ingress-nginx를 사용하거나 클라우드 플랫폼에서 제공하는 인그레스 컨트롤러를 이용할 수 있다.

```yaml
# Ingress Spec Example
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rootSvc
                port:
                  number: 80
          - path: /testpath
            pathType: Prefix
            backend:
              service:
                name: test
                port:
                  number: 80
```
