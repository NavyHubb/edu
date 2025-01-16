# lecture-9 : Service Mash(istio)
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture9
```

### istio 구성 요소
![github PAT](/lecture9/img/lecture9-istio-arch.png)

- Control Plane

  서비스 메시의 전반적인 동작을 관리하고 구성하는 역할
  - Pilot : 서비스 메시 동작 관리하고 구성, 라우팅 규칙을 Envoy별 구성하고 프록시에 전파
  - Citadel : 서비스간 통신을 위한 인증서 관리, TLS 인증 
  - Galley : istio 구성 데이터 관리, 서비스 전체 일관성 유지, Configuration 체크
- Data Plane

  마이크로 서비스와 함께 배포되는 프록시 세트(istio에서 Envoy 사용)
  - Envoy : 마이크로 서비스간 인/아웃 트래픽을 가로채는 프록시, 로드밸런싱, 라우팅, 통신 관리

# 1. istio 설치

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator

metadata:
  namespace: istio-system
  name: istiocontrolplane

spec:
  profile: default
  components:

    egressGateways:
    - name: istio-egressgateway
      enabled: true
      k8s:
        hpaSpec:
          minReplicas: 1

    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        hpaSpec:
          minReplicas: 1

    pilot:
      enabled: true
      k8s:
        hpaSpec:
          minReplicas: 1

  meshConfig:
    enableTracing: true
    defaultConfig:
      holdApplicationUntilProxyStarts: true
    accessLogFile: /dev/stdout
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
```

```bash

curl -L https://istio.io/downloadIstio | sh -

# 버전 확인 후 
cp istio-1.24.2/bin/istioctl /usr/local/bin/

kubectl create namespace istio-system
istioctl install -f istio.yaml

# 설치 확인
kubectl get all -n istio-system
## LB가 없어 istio-ingressgateway pending 상태
#NAME                                        READY   STATUS    RESTARTS   AGE
#pod/istio-egressgateway-fdd6c7bf-7t6ft      1/1     Running   0          48s
#pod/istio-ingressgateway-79ddd4cf94-gbmhw   1/1     Running   0          48s
#pod/istiod-6fbf849d94-znm8x                 1/1     Running   0          58s

#NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                      AGE
#service/istio-egressgateway    ClusterIP      10.43.111.98    <none>        80/TCP,443/TCP                               48s
#service/istio-ingressgateway   LoadBalancer   10.43.11.130    <pending>     15021:31428/TCP,80:30453/TCP,443:30965/TCP   48s
#service/istiod                 ClusterIP      10.43.210.110   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        58s

# http에 매핑 된 노드 포트 확인
kubectl get service istio-ingressgateway -n istio-system -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
```
## 1.1 방화벽 오픈 : istio 접속 노드 포트
- kt cloud master01 서버에 http에 매핑 된 노드 포트(여기서는 30453) 오픈

# 2. 데모 서비스 배포

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```
```bash

# 데모 서비스 배포용 namespace 생성
kubectl create ns istio-demo-ns

# istio-demo-ns에 배포되는 서비스 Envoy sidecar proxy 자동으로 연결되게 설정 
kubectl label namespace istio-demo-ns istio-injection=enabled

## istio 사용할 수 있게 라벨 설정한 Namespace 반드시 선택하여 배포 ##
kubectl apply -f istio-1.24.2/samples/bookinfo/platform/kube/bookinfo.yaml -n  istio-demo-ns

## Gateway, VirtualService 배포
kubectl apply -f istio-1.24.2/samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo-ns
```
- ### 서비스 확인 : 부라우저에서 
  - http://211.253.25.128.sslip.io:30453/productpage

# 3. istio 기능 확인 Addon 서비스 배포

```bash

# Kiali 및 기타 애드온 설치 : promethus, grafana, jaeger, kiali
kubectl apply -f istio-1.24.2/samples/addons -n istio-system

#kiali 및 grafana 노드 포트로 변경
kubectl rollout status deployment/kiali -n istio-system 
kubectl rollout status deployment/grafana -n istio-system 

# 서비스를 clusterip type -> nodePort 형태로 변경 (외부 접속)
kubectl patch -n istio-system svc kiali -p '{"spec": {"type": "NodePort"}}'
kubectl patch -n istio-system svc grafana -p '{"spec": {"type": "NodePort"}}'
```
## 3.1 방화벽 오픈 : Addon 접속 노드 포트
```bash

# 실행 한 결과 확인 후 포트 변경 
kubectl get svc -n istio-system
```
- kt cloud master01 서버에 kiali에 매핑 된 노드 포트(여기서는 30899) 오픈
- kt cloud master01 서버에 grafana에 매핑 된 노드 포트(여기서는 31140) 오픈

## 3.2 서비스 확인 : 브라우저에서
- ### kiali 접속 확인
  - http://211.253.25.128.sslip.io:30899
- ### grafana 접속 확인
  - http://211.253.25.128.sslip.io:31140

```dtd
# 부하 발생 용 curl_product.sh로 생성
for i in $(seq 1 300); do
  curl -s -o /dev/null "http://211.253.25.128.sslip.io:30453/productpage"
done
```

```bash

sh curl_product.sh
```
