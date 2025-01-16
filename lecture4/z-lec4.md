# lecture-4 : Ingress
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture4
```


# 1. ingress

## 1.1 ingress controller
- nginx ingresscontroller
```
# rke2는 기본적으로 nginx-ingressController가 설치 됨
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.5/deploy/static/provider/cloud/deploy.yaml
```
```sh

## ingressController 조회
kubectl get pod -n kube-system | grep ingress-nginx-controller
kubectl get svc -n kube-system | grep ingress-nginx-controller

# 조회 결과
#root@master01:~/kubernetes/lecture4# kubectl get pod -n kube-system | grep ingress-nginx-controller
#rke2-ingress-nginx-controller-67lpx                     1/1     Running     2 (20m ago)   10d
#rke2-ingress-nginx-controller-nrppk                     1/1     Running     2 (18m ago)   10d
#rke2-ingress-nginx-controller-zhw7x                     1/1     Running     2 (18m ago)   10d
#root@master01:~/kubernetes/lecture4# kubectl get svc -n kube-system | grep ingress-nginx-controller
#rke2-ingress-nginx-controller-admission   ClusterIP   10.43.196.190   <none>        443/TCP         10d
```
## 1.2 backend 서비스 용 nginx/apache 배포

- nginx-apache.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
  labels:
    app: "httpd"
spec:
  type: ClusterIP
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: "httpd"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
  labels:
    app: "httpd"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: "httpd"
  template:
    metadata:
      labels:
        app: "httpd"
    spec:
      containers:
      - name: httpd
        image: httpd:latest
        ports:
        - name: http
          containerPort: 80
```
```sh

## nginx/apache 배포
kubectl apply -f nginx-apache.yaml
```

## 1.3 path 방식 ingress rule 설정
ingress 적용하면 외부에서 접속 가능

- ingress-rule.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
      - path: /apache
        pathType: Prefix
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
```
```sh

kubectl apply -f ingress-rule.yaml

kubectl get ingress
## 바로 적용 안될수 있다 조금 시간이 걸릴수 있다 
#root@master01:~/kubernetes/lecture4# kubectl get ingress
#NAME      CLASS   HOSTS   ADDRESS   PORTS   AGE
#web-ing   nginx   *                 80      12s

curl http://172.27.0.179/nginx
curl http://172.27.0.179/apache
curl http://172.27.0.136/nginx
curl http://172.27.0.136/apache
curl http://172.27.0.48/nginx
curl http://172.27.0.48/apache

## external-ip로 접속 가능
## 모든 노드의 external-ip 접속 가능(default: 80 오픈)  
```

## 1.3.1 방화벽 오픈
```
kt cloud 외부 접속 포트 (80) 오픈 : master01,worker01,worker02
- lecture0 VM생성 : d. 접속 설정 (방화벽 오픈) 참고
```
## 1.3.2 서비스 확인
- 각자 서버 공인IP로 변경 후 확인
- http://211.253.25.128/nginx    
- http://211.253.25.128/apache   
- http://211.253.30.100/nginx   
- http://211.253.30.100/apache   
- http://211.253.8.141/nginx     
- http://211.253.8.141/apache

## 1.3.3 path 방식 룰 clear 
```bash

kubectl delete -f ingress-rule.yaml 
```

## 1.4 host 기반 ingress rule 설정
```
o DNS wildcard 서비스 사용하여 테스트 : 
  - IP 주소를 서브도메인에 포함시켜 DNS를 자동으로 매핑해주는 역할(sslip.io, nip.io)
  - Ingress 가 요청의 헤더 확인, 헤더의 HOST 값을 ingress rule 에서 찾음
  - Ingress 룰(host 설정, ex:nginx.211.253.25.128.sslip.io)에 있으면 해당 service로 이동  
```  
- ingress-host-rule.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
spec:
  ingressClassName: nginx
  rules:
  - host: "nginx.211.253.25.128.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  - host: "apache.211.253.25.128.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
```
```sh
kubectl apply -f ingress-host-rule.yaml

kubectl get ingress
kubectl get ing

# 조회 결과 : 포트 80만 할성화 (443포트 비활성화)
#root@master01:~/kubernetes/lecture4# kubectl get ingress
#NAME      CLASS   HOSTS                                                          ADDRESS   PORTS   AGE
#web-ing   nginx   nginx.211.253.25.128.sslip.io,apache.211.253.25.128.sslip.io             80      3s
```
## 1.4.1 서비스 확인
http://nginx.211.253.25.128.sslip.io
http://apache.211.253.25.128.sslip.io
http://nginx.211.253.30.100.sslip.io
http://nginx.211.253.30.100.sslip.io


# 2 TLS Termination(Self-signed)

```sh

## 인증서를 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginxsvc/O=nginxsvc"

## TLS Secret
kubectl create secret tls nginx-tls-secret --key tls.key --cert tls.crt

## default namespace에서 조회
kubectl get secret

# 조회 결과
#root@master01:~/kubernetes/lecture4# kubectl get secret
#NAME               TYPE                DATA   AGE
#nginx-tls-secret   kubernetes.io/tls   2      9s
```

- ingress-tls.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
  # annotations:
    # nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - secretName: nginx-tls-secret
  rules:
  - host: "nginx.211.253.25.128.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```
```sh

## ingress-rule 배포
kubectl apply -f ingress-tls.yaml

kubectl get ing
# 조회 결과 : 80, 443포트 활성화 
# IngressController가 80, 443 모두 활성화 시킴
#root@master01:~/kubernetes/lecture4# kubectl get ingress
#NAME      CLASS   HOSTS                           ADDRESS   PORTS     AGE
#web-ing   nginx   nginx.211.253.25.128.sslip.io             80, 443   19s
```
## 2.1 방화벽 오픈
```
kt cloud 외부 접속 포트 (443) 오픈 : master01 공인 IP에
- lecture0 VM생성 : d. 접속 설정 (방화벽 오픈) 참고
```
## 2.2 서비스 확인
- http://nginx.211.253.25.128.sslip.io
- https://nginx.211.253.25.128.sslip.io

## 2.3 redirect http to https(강제적)
```yaml
annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

# 2.4 tls clear
```bash

kubectl delete -f ingress-tls.yaml
kubectl delete -f nginx-apache.yaml

kubectl delete secret nginx-tls-secret
```