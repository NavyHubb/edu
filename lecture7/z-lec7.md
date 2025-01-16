# lecture-7 : 배포 전략, nodeSelector, Affinity
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture7
```

# 1. node lables
```sh

kubectl get nodes
kubectl get nodes --show-labels 
kubectl get nodes --show-labels | grep kubernetes.io/hostname


## node에 label 설정하기 
kubectl get nodes
kubectl label node master01 nodename=master01                     
kubectl label node worker01 nodename=worker01 web=true
kubectl label node worker02 nodename=worker02 web=true db=true 
kubectl get nodes --show-labels | grep nodename
kubectl get nodes --show-labels | grep db

## label 삭제 
kubectl label nodes worker02   db-
kubectl get nodes --show-labels | grep db
```

# 2. nodeSelector 적용 nginx 배포
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:      
      labels:
        app: nginx    
    spec:
      nodeSelector:
        web: "true"
      containers:
        - name: nginx
          image: nginx:1.17
          ports:
          - name: http
            containerPort: 80

```
```sh

## web: "true"
kubectl apply -f nodeSelector.yaml

# 조회 결과 : 실행 Node 확인, Worker01, 02 모두 실행 
kubectl get pod -o wide

kubectl delete -f nodeSelector.yaml
kubectl label nodes worker02   web-
kubectl get nodes --show-labels | grep web
kubectl apply -f nodeSelector.yaml

# 조회 결과 : 실행 Node 확인, Worker01만 실행 
kubectl get pod -o wide
kubectl delete -f nodeSelector.yaml
```

# 3. nodeAffinity

## 3.1 requiredDuringSchedulingIgnoredDuringExecution
- Affinity 매핑 조건 web: "true"
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:      
      labels:
        app: nginx    
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:       # pod 스케줄링에 꼭 필요한 조건
            nodeSelectorTerms:
            - matchExpressions:
              - key: web
                operator: In
                values:
                - "true"
      containers:
        - name: nginx
          image: nginx:1.17
          ports:
          - name: http
            containerPort: 80
```
```sh

kubectl apply -f  nodeRequireAffinity.yaml

# 조회 결과 : 실행 Node 확인, worker01만 실행 
kubectl get pod -o wide

## 삭제
kubectl delete -f nodeRequireAffinity.yaml
kubectl label nodes worker02 web=true
kubectl apply -f nodeRequireAffinity.yaml

# 조회 결과 : 실행 Node 확인, worker01, worker02 모두 실행 
kubectl get pod -o wide
kubectl delete -f nodeRequireAffinity.yaml
```

## 3.2 preferredDuringSchedulingIgnoredDuringExecution
- Affinity 매핑 조건에 맞지 않는 web: True 로 생성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:     # 만족하면 좋은 조건, 반드시 만족할 필요 없음
            - weight: 100                                      # 1 ~ 100 사이, 높을 수록 중요도 큼
              preference:
                matchExpressions:
                  - key: web
                    operator: In
                    values:
                      - "True"
      containers:
        - name: nginx
          image: nginx:1.17
          ports:
            - name: http
              containerPort: 80
```
```bash
kubectl label nodes worker02 web-
kubectl apply -f  nodePreferAffinity.yaml

# 조회 결과 : 실행 Node 확인, worker01, worker02 모두 실행 
kubectl get pod -o wide

## 삭제 
kubectl delete -f  nodePreferAffinity.yaml
```

# 4. podAffinity
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: web
                    operator: In
                    values:
                      - "true"
      containers:
        - name: nginx
          image: nginx:1.17
          ports:
            - name: http
              containerPort: 80

```
```sh 
kubectl label nodes worker02 web=true
kubectl get node --show-labels | grep web

kubectl apply -f nginx-deploy.yaml
kubectl get pod -o wide

# 각자 pod 이름 수정후 라벨 설정 : pod가 1개 실행된 노드에 실행된 pod를  S2 로 설정 
kubectl label pods nginx-765cbd9ff8-5zp6s security=S1
kubectl label pods nginx-765cbd9ff8-q68fs security=S2
kubectl label pods nginx-765cbd9ff8-qsn75 security=S3

kubectl get pod --show-labels

## security=S2
kubectl apply -f podAffinity.yaml   
## 확장해도 같은 pod의 위치에 배치됨
kubectl scale --current-replicas=4 --replicas=5 deployment/nginx
## 삭제
kubectl delete -f podAffinity.yaml
```

# 5. podAntiAffinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: web
                    operator: In
                    values:
                      - "true"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - nginx
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.17
          ports:
            - name: http
              containerPort: 80
```
```bash

## replicas=3
kubectl apply -f podAntiAffinity.yaml

## 1개는 pending 됨.
kubectl get pod

## replicas=2 줄임
kubectl scale --current-replicas=3 --replicas=2 deployment/nginx

## update nginx version 
## nginx:1.17 --> nginx:1.18
kubectl set image deployment/nginx nginx=nginx:1.18
## pending 발생 
```
# 5.1 podAntiAffinity
- maxUnavailable 설정으로 제어

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxUnavailable: 50%        # 동시에 삭제되는 pod 비율 or 개수 설정(default : 25%)
      maxSurge: 25%              # 동시에 생성되는 pod 비율 or 개수 설정(default : 25%)
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: web
                    operator: In
                    values:
                      - "true"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - nginx
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.18
          ports:
            - name: http
              containerPort: 80
```
```bash
## pending 발생, strategy.rollingUpdate의 maxUnavailable 설정으로 제어
## replicas=2, nginx:1.18
kubectl apply -f podAntiAffinity2.yaml
kubectl set image deployment/nginx nginx=nginx:1.19
```

## 5.2 maxSurge 설정으로 제어
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: web
                    operator: In
                    values:
                      - "true"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - nginx
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - name: http
              containerPort: 80
```
```bash

## strategy.rollingUpdate의 maxSurge 설정으로 제어
## replicas=2, nginx:1.21
kubectl apply -f podAntiAffinity3.yaml
kubectl set image deployment/nginx nginx=nginx:1.22

## clear
kubectl delete -f podAntiAffinity3.yaml
```

# 6. Taint/Toleration
- taint : 노드에 설정되며, 특정 Pod이 해당 노드에서 실행되는 것을 제한하거나 차단함
  - NoSchedule: Toleration이 없으면 Pod은 스케줄링되지 않음
  - PreferNoSchedule: Toleration이 없으면 가능하면 스케줄링되지 않도록 시도하지만, 강제적이지 않음
  - NoExecute: Toleration이 없으면 이미 실행 중인 Pod도 제거됨
- Toleration : Pod에 설정되며, 특정 Taint를 "허용"하도록 구성함

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:      
      labels:
        app: nginx    
    spec:      
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: web
                operator: In
                values:
                - "true"
      tolerations:
      - key: oss
        operator: Equal
        value: nginx
        effect: NoSchedule
      containers:
        - name: nginx
          image: nginx:1.17
          ports:
          - name: http
            containerPort: 80
```
```bash

## node에 taint 설정 : worker01 에 pod 생성 안됨
kubectl taint nodes worker01 oss=nginx:NoSchedule 
## taint 확인
kubectl describe node worker01

## node 라벨 확인 : kubectl get node --show-labels | grep web  
## 라벨 설정 :  kubectl label nodes worker01 web=true
kubectl apply -f nginx-deploy.yaml
## nginx pod가 생성된 node를 확인(모두  worker02 생성)
## taint 삭제 
kubectl taint nodes worker01 oss-

## nginx 1개의 pod를 삭제하여 다른 노드에 설치 되는지 확인(worker01에도 생성 되는지 확인)
kubectl get pod
## 각자 조회된 pod 이름 수정 후 실행 
kubectl delete pod nginx-765cbd9ff8-bqqf5

## nginx 를 모두 삭제 
kubectl delete -f nginx-deploy.yaml

## 다시 label web=true인 노드의 1개에 taint를 설정한다 (worker02)
kubectl label nodes worker01 web-
kubectl taint nodes worker02 oss=nginx:NoSchedule
kubectl describe node worker02

## Toleration이 설정된 nginx를 배포한다 
kubectl apply -f nginx-deploy.yaml
kubectl get pod
## 조회 결과 : 모든 pod의 STATUS Pending 확인 
kubectl delete -f nginx-deploy.yaml

## toleration 적용
kubectl apply -f nginx-deploy-toleration.yaml
kubectl get pod
## 조회 결과 : 모든 pod의 worker02에서 실행
```
## 6.1 Clear
```bash

## clear 
kubectl delete -f nginx-deploy-toleration.yaml
kubectl label node master01 nodename-               
kubectl label node worker01 nodename- web-
kubectl label node worker02 nodename- web- db-
```

# 7. Blue & Green 배포
## 7.1 Blue : 현재 운영 버전
```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configmap-blue
data:
  index.html: |    # 멀티라인 문자열 작성할 때 사용, 포멧 유지 -> HTML, JSON, YAML 등 사용 
    <html>
        <body>
            <h1>Welcome to nginx!</h1>
            <h3> = nginx version : v1.17 = </h3>
        </body>
    </html>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-blue-deployment
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
          volumeMounts:
            - name: nginx-index-config-vol
              mountPath:  /usr/share/nginx/html/index.html    # 컨테이너 내부의 파일 시스템 경로 지정
              subPath: index.html                             # 볼륨의 특정 파일이나 디렉터리만 마운트하는 데 사용(아래 volumes path 중 일부)
      volumes:
        - name: nginx-index-config-vol
          configMap:
            name: nginx-configmap-blue
            items:
              - key: index.html                               # ConfigMap의 특정 데이터 (여기서는 data.index.html: )
                path: index.html                              # ConfigMap key에 해당하는 값(데이터) 마운트할 사용 할 내부 파일 이름

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
    version: blue  # 기본적으로 Blue 배포에 연결
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

---
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
                name: nginx-service
                port:
                  number: 80

```
```bash

kubectl apply -f nginx-deploy-blue.yaml
kubectl get pods -l app=nginx,version=blue
# 조회 결과 
#root@master01:~/kubernetes/lecture7# kubectl get pods -l app=nginx,version=blue
#NAME                                     READY   STATUS    RESTARTS   AGE
#nginx-blue-deployment-5cc4df98b4-lx7zx   1/1     Running   0          55s
#nginx-blue-deployment-5cc4df98b4-zdnjs   1/1     Running   0          54s

kubectl get service nginx-service
```
- ### 서비스 확인(버전 1.17) : 브라우저에서
  - http://nginx.211.253.25.128.sslip.io
  
## 7.2 Green : 신규 버전
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configmap-green
data:
  index.html: |    # 멀티라인 문자열 작성할 때 사용, 포멧 유지 -> HTML, JSON, YAML 등 사용 
    <html>
        <body>
            <h1>Welcome to nginx!</h1>
            <h3> = nginx version : v1.22 = </h3>
        </body>
    </html>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-green-deployment
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
          image: nginx:1.22
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-index-config-vol
              mountPath:  /usr/share/nginx/html/index.html    # 컨테이너 내부의 파일 시스템 경로 지정
              subPath: index.html                             # 볼륨의 특정 파일이나 디렉터리만 마운트하는 데 사용(아래 volumes path 중 일부)
      volumes:
        - name: nginx-index-config-vol
          configMap:
            name: nginx-configmap-green
            items:
              - key: index.html                               # ConfigMap의 특정 데이터 (여기서는 data.index.html: )
                path: index.html                              # ConfigMap key에 해당하는 값(데이터) 마운트할 사용 할 내부 파일 이름
```
```bash

# 서비스 신규 버전 배포 : green deploy 
kubectl apply -f nginx-deploy-green.yaml
kubectl get pods -l app=nginx,version=green
```

## 7.3 서비스 전환
```bash

# 서비스 전환 전(blue) 상태 : Selector, EndPoing 확인  
kubectl describe service nginx-service
kubectl get endpoints nginx-service

## 서비스 Green 으로 전환 
kubectl patch service nginx-service -p '{"spec":{"selector":{"app":"nginx","version":"green"}}}'
# yaml 파일 적용 : 서비스 전체를 변경하는 작업
# 서비스 내 일부만 변경할 때는 json 규격으로 커멘드 라인에 파라미터 직접 입력 (파일 작접 참조 방식 지원 안됨)
# kubectl apply -f nginx-service-green.yaml 

# 서비스 전환 후(blue) 상태 : Selector, EndPoing 확인  
kubectl describe service nginx-service
kubectl get endpoints nginx-service

## 서비스 오류 시 롤백 : 이전 버전 인 Blue
kubectl patch service nginx-service -p '{"spec":{"selector":{"app":"nginx","version":"blue"}}}'
```
- ### 서비스 확인(버전 1.22) : 브라우저에서
  - http://nginx.211.253.25.128.sslip.io

## 7.4 Clear
```bash

kubectl delete -f nginx-deploy-blue.yaml
kubectl delete -f nginx-deploy-green.yaml
```


