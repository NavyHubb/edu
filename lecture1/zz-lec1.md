# lecture-1 : Pod
- Replication Controller, RelicaSet, Deployment, Rollout
- Master Node에서 실행 
  
```sh

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture1
```

# 1.  pod 생성

## 1.1 kubectl run
```sh

# default namepsace에서 실행된다 
kubectl run my-nginx --image=nginx

# pod 조회
kubectl get pod
kubectl get pods

kubectl exec -it my-nginx -n default -- curl localhost
# default namespace 생략 가능
kubectl exec -it my-nginx  -- curl localhost
# 삭제
kubectl delete pod/my-nginx
```


## 1.2 pod.yaml(k8s manifest) 로  pod 생성
- pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

```sh

kubectl create -f pod.yaml
## apply 추천
kubectl apply -f pod.yaml
## 확인
kubectl exec -it my-nginx -n default -- curl localhost

## 삭제
kubectl delete -f pod.yaml

```

### 1.2.1 pod 조회 
```bash

kubectl get pod 
kubectl get pod -o wide       # o : 출력 형식 wide(테이블), json, yaml 등 
kubectl describe pod my-nginx 
kubectl events pod/my-nginx 
kubectl logs pod/my-nginx 
kubectl logs -f pod/my-nginx 
kubectl exec -it my-nginx -- /bin/bash
curl http://localhost

# log 저장 위치 : Worker Node
# /var/log/pods/{Namespace명}_{Pod명}_{Pod-uid}
#ex) /var/log/pods/default_my-nginx_0c9a0e8f-716a-4204-88ea-49829437165a/nginx

# clear 
kubectl delete -f pod.yaml
```

## 1.3 Replication Controller
- rc.yaml
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
          - containerPort: 80

```
```bash

# 생성
kubectl apply -f rc.yaml

# 조회
kubectl get replicationcontroller -n default
kubectl get rc

## 확인
kubectl exec -it my-nginx-66txs -n default -- curl localhost

## pod 하나를 지워본다, 다시 생성된다 
kubectl get pods
kubectl delete pod my-nginx-l2258 

## 삭제
kubectl delete -f rc.yaml
```


## 1.4  ReplicaSet
레플리카셋을 직접 사용하기보다는 디플로이먼트를 사용하는 것을 권장

- rs.yaml
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-nginx
  labels:
    app: nginx
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
          - containerPort: 80
```
```bash

# 생성 
kubectl apply -f rs.yaml

# replicaset 확인
kubectl get replicaset
kubectl get rs

# pod 
kubectl get pods
kubectl exec -it my-nginx-26hzf -n default -- curl localhost

# pod 하나 삭제하면, 다시 생성 됨 
kubectl delete pod my-nginx-l2258
```

replicaset은 replication controller와 똑같이 동작하지만 더 풍부한 표현식 pod selector를 갖는다.
```yaml
apiVersion: apps/v1
kind: ReplicaSet                                                             
metadata:
  name: myrs
spec:
  replicas: 2  
  selector:                  
    matchExpressions:                             
      - {key: myname, operator: In, values: [niraj, krishna, somesh]}
      - {key: env, operator: NotIn, values: [production]}
  template:      
    metadata:
      name: testpod7
      labels:              
        myname: Bhupinder
    spec:
     containers:
       - name: c00
         image: ubuntu
         command: ["/bin/bash", "-c", "while true; do echo Technical-Guftgu; sleep 5 ; done"]
```

### 1.4.1 clear
```sh

 # clear
kubectl delete -f rs.yaml
```

## 1.5  deployment
```
- 파드와 레플리카셋(ReplicaSet)에 대한 선언적 업데이트를 제공한다
- Rolling Update
```

- nginx-deploy.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3 
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
```

### 1.5.1 생성및 조회
```sh

##생성
kubectl apply -f deploy.yaml
## pod 이름 확인: nginx-deployment-7d5c8d9554-8nxb2

# 조회 
kubectl get deployment -n default
kubectl get deploy

kubectl describe deployment nginx-deployment
kubectl get pods -l app=nginx
kubectl get pods --show-labels
```


### 1.5.2 update 

- deploy.yaml 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.18   ## Update the version of nginx from 1.17 to 1.18
        ports:
        - containerPort: 80
```
```sh

## update 반영
kubectl apply -f deploy-u.yaml

## 상세보기 
kubectl describe deployment nginx-deployment  

kubectl get rs ## replicas 정보확인( DESIRED, CURRENT) , histroy 
```

### 1.5.3 deployment 삭제 
```bash

kubectl delete deployment nginx-deployment
kubectl delete -f deploy-u.yaml

```

### 1.5.4 pod 리소스 확인 
```bash

kubectl apply -f deploy.yaml
kubectl get apiservices | grep metrics

# pod 리소스 사용량 확인
kubectl top pod
kubectl top pod -n <namespace>

# node 리소스 사용량 확인 
kubectl top node
```

# 2. rollout 
- kubernetes 클러스터에서 롤아웃 작업을 관리하는 명령어이며  이 명령어를 사용하면, 디플로이먼트, 데몬셋, 상태풀셋 등의 리소스에 대한 롤아웃 작업을 수행할 수 있다
- rollout를 하기 위해서는 --record 옵션을 둘수 있다 하지만 이것은 deprecated 되어서 앞으로 사용하지 말것
- rollout 명령어 
  - kubectl rollout status: 롤아웃 작업의 상태를 확인
  - kubectl rollout history: 롤아웃 작업의 이력을 확인
  - kubectl rollout undo: 롤아웃 작업을 취소하고 이전 버전으로 롤백
  - kubectl rollout restart: 롤아웃 작업을 재시작 
  - kubectl rollout pause/resume: 롤링 업데이트를 일시 중지하거나 다시 시작

```bash

kubectl apply -f deploy.yaml
kubectl get deploy 

kubectl rollout status deploy/nginx-deployment

kubectl rollout history deploy/nginx-deployment

## CHANGE-CAUSE에 정보가 나오지 않는다 
#REVISION  CHANGE-CAUSE
#1         <none>
```

## 2.1 rollout history CHANGE-CAUSE 설정 

- deploy-rollout.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  annotations:
    kubernetes.io/change-cause: "image updated to 1.17"

spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3 
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

```
```sh

## 이전 배포 clear 
kubectl delete -f deploy.yaml

## 배포
kubectl apply -f deploy-rollout.yaml

kubectl rollout history deploy/nginx-deployment

#REVISION  CHANGE-CAUSE
#1         image updated to 1.17
```
## 2.2  image update
```sh

# nginx:1.18 이미지 변경
kubectl set image deployment/nginx-deployment nginx=nginx:1.18
## annotations update
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.18"

# nginx:1.19 이미지 변경
kubectl set image deployment/nginx-deployment nginx=nginx:1.19
## annotations update
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.19"

kubectl rollout history deploy/nginx-deployment

```

## 2.3 rollout undo 
```sh

kubectl rollout undo deploy/nginx-deployment
kubectl describe deployment nginx-deployment 
kubectl rollout history deploy/nginx-deployment
kubectl rollout undo deploy/nginx-deployment
kubectl rollout history deploy/nginx-deployment

# 지정된 revision으로 undo
kubectl rollout undo deploy/nginx-deployment --to-revision=1
kubectl rollout history deploy/nginx-deployment
```

## [참고] k8s resource api 및 단축키 조회 명령어
```bash

kubectl api-resources
```
