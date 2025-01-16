# lecture-8 : CI/CD 파이프 라인
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture8
```

# CI/CD
- git : github
- image Registry: docker-hub
- build :  docker build
- deploy: argocd
- github/docker-hub 계정 필요

## 1. github 소스 import 
```
# 기준 소스 github 접속
https://github.com/yeongdeokcho/edu-demo

## import 대상 소스 url 복사 : HTTPS 탭 선택 후 URL 복사 (default : SSH)
code 클릭 후 URL 복사

## 각자 githut 이동

## 상단 + 버튼 클릭 후 import 설정 추가 후 begin import 클릭
```
- 기준 소스 복사 : HTTPS
  ![docker build](/lecture8/img/lecture8-github-base.png)
- Import 메뉴 선택
  ![docker build](/lecture8/img/lecture8-github-imp.png)
- 신규 레포지토리 설정 : import 기준
    ![docker build](/lecture8/img/lecture8-github-imp-set.png)

# 2. Docker Build

```dtd
FROM openjdk:17-jdk-slim

# Install required tools
RUN apt-get update && apt-get install -y \
    telnet \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Set the working directory
WORKDIR /app

# Copy the JAR file into the container
COPY build/libs/*.jar demo.jar

# Expose the port Spring Boot runs on
EXPOSE 8080

# Run the Spring Boot application
CMD ["java", "-jar", "demo.jar"]
```
```bash

# 개인 PC에서 진행 : docker desktop 실행

## MAC/Linux 사용자
#cd /Users/doong2s/src/ktds/demo
#sudo docker build -f Dockerfile .  --tag k8s-edu --no-cache

## Windows 사용자
cd /Users/doong2s/src/ktds/demo
sudo docker build -f Dockerfile .  --tag k8s-edu --no-cache

# docker image 조회
docker images
docker login

# 도커허브 개인계정/Repo 변경 후 진행 
docker tag k8s-edu ydcho0902/k8s-edu:v0.0.1

# dockerHub 에 k8s-edu 레포지토리 생성 후 (개인계정/repo명:TAG) 
docker push ydcho0902/k8s-edu:v0.0.1
```
- docker buile
![docker build](/lecture8/img/lecture8-docker-build.png)
- docker hub push image 조회 
![docker hub](/lecture8/img/lecture8-docker-push.png)

# 3. Demo 서비스 수동 배포
## 3.1 docker hub secret 생성

```bash

# Docker 로그인 정보를 Base64로 인코딩
#echo -n 'my-docker-username:my-docker-password' | base64

## master01 서버에서 실행,  
echo -n 'mosy@paran.com:f..&..' | base64
#bW9zeUBwYXQhbJ5jb206ZnJvYXNzd29yZA==
```
- dockerconfig.json : auth 값 수정
```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "bW9zeUBwYXQhbJ5jb206ZnJvYXNzd29yZA=="
    }
  }
}
```
```bash

# 위에서 실행 auth 값 수정 : 아래 json 파일 참조하여 토큰값 변경 
vi dockerconfig.json 

## master01 서버에서 실행,
cat dockerconfig.json | base64
#ewogICJhdXRocyI6IHsKICAgICJodHRwczovL2luZGV4LmRv...생략...==        ##### 각자 개인계정 토근 값 변경 후 
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: k8s-edu-dockerhub-secret
  namespace: default
data:
  .dockerconfigjson: >
    eyJhdXRocyI6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MSguZG9ja...생략...==
type: kubernetes.io/dockerconfigjson
```
```bash

# Master01 노드에서 진행
kubectl apply -f docker-secret.yaml
kubectl get secret
```
## 3.2 서비스 배포(수동)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-edu-deployment
spec:
  selector:
    matchLabels:
      app: k8s-edu-app
  replicas: 1
  template:
    metadata:
      labels:
        app: k8s-edu-app
        version: blue
    spec:
      containers:
        - name: k8s-edu-app
          image: ydcho0902/k8s-edu:v0.0.1      # dockerhub image repository : repo/image명:Tag 
          imagePullPolicy: IfNotPresent        # k8s 클러스터에 다운로드 된 이미지 없으면 다운 or Always 
          ports:
            - name: http
              containerPort: 8080              # demo.jar 실행 포트
              protocol: TCP
          resources:                           # pod 사용 리소스 설정 블록
            requests:                          # 생성시 필요한 최소 리소스 
              cpu: "1"
              memory: "2Gi"
            limits:                            # pod가 사용 가능한 최대 리소스 
              cpu: "1"
              memory: "2Gi"
          env:                                 # pod 내 환경 변수 설정
            - name: SPRING_PROFILES_ACTIVE     # spring profile 설정 
              value: prod                      # 상용(prod) 환경 
      imagePullSecrets:                        # dockerhub 이미지 pull 위한 secret 
        - name: k8s-edu-dockerhub-secret
      nodeSelector:                            # pod가 실행 될 node 설정 
        kubernetes.io/hostname: worker01

---
apiVersion: v1
kind: Service
metadata:
  name: k8s-edu-service
spec:
  selector:
    app: k8s-edu-app
    version: blue  # 기본적으로 Blue 배포에 연결
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080                        # deployment에서 설정한 컨테이너 포트 매핑

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-edu-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: "k8s.211.253.25.128.sslip.io"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: k8s-edu-service
                port:
                  number: 80
```
```bash

# Master01 노드에서 진행
kubectl apply -f demo-deploy.yaml
```
- #### 서비스 확인 : 브라우저에서
  - http://k8s.211.253.25.128.sslip.io/api/v1/user/82265604

# 4. argocd로 서비스 배포하기
## 4.1 argocd 설치
```bash

curl https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml > argocd-install.yaml
vi argocd-install.yaml
## 아래 yaml 23744 라인 전후에 - --insecure  추가(https 비활성화 옵션) 참고
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: argocd-server
              topologyKey: kubernetes.io/hostname
            weight: 100
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/part-of: argocd
              topologyKey: kubernetes.io/hostname
            weight: 5
      containers:
      - args:
        - /usr/local/bin/argocd-server
        - --insecure        ## added by yeongdeok.cho 추가 https 비활성화
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              key: auth
              name: argocd-redis
```
![argocd 설치](/lecture8/img/lecture8-argocd-install.png)

## 4.2 argocd 접속 ingress 생성
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ing
  namespace: argocd
#  annotations:
#    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
#    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: "argocd.211.253.25.128.sslip.io"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: http
```
```bash

kubectl create namespace argocd

kubectl apply -n argocd -f argocd-install.yaml

## argocd password 확인 : root@ 전까지...
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# 결과 확인
#kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
#cMEREj0TXtE0wda4root@master01:~/kubernetes/lecture8#

kubectl apply -f argocd-ing.yaml
```

- ### argocd 접속
  - http://argocd.211.253.25.128.sslip.io/
  - 접속계정(위에서 확인 비번) : admin/cMEREj0TXtE0wda4


## 4.3 argocd 설정 
### 4.2.1 git repository 설정

```
# https://github.com/yeongdeokcho/edu.git -> 개인 레포지토리로 수정
# 데모 소스와 다른 github repository에 k8s배포용 manaifast 관리 권장 

o argocd 홈  >  Settings > Repositories > connect REPO
o VIA HTTPS 선택
o type: git
o project: default
o Repository URL : https://github.com/yeongdeokcho/edu.git
o Username : 각 개인 계정
o Password : github PAT (zz-github.md 참조)
```
![github PAT](/lecture8/img/letcure8-cicd-github-pat.png)
![argocd connect](/lecture8/img/lecture8-cicd-argo-conn.png)

### 4.2.2 git applications 설정
```
o Applications > create
o Name: demo
o Project Name: default
o Repository URL : 선택(https://github.com/yeongdeokcho/edu.git) 
o Revision: HEAD
o Path : 선택(lecture8/manifests)
o Cluster URL :  https://kubernetes.default.svc 선택 
o Namespace:  default
o kustomize : Images 부분에 이미지와 tag 버전이 맞는지 확인 
o 위의 CREATE 버튼 클릭
o SYNC 버튼 클릭 >  SYNCRONIZE
```
![aogocd app 등록](/lecture8/img/lecture8-cicd-argo-app.png)
![argocd app 등록완료](/lecture8/img/lecture8-cicd-argo-app-demo.png)
![argocd Sync](/lecture8/img/lecture8-cicd-argo-sync.png)
![argocd Dashboard](/lecture8/img/lecture8-cicd-argo-dash.png)
