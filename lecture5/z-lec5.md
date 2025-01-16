# lecture-5 : Namespace & ConfigMap & Secret
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture5
```

# 1. namespace

## 1.1 web1 namespace
```bash

kubectl create ns web1

kubectl get namespace
kubectl get ns 
# 결과 조회
#root@master01:~/kubernetes/lecture5# kubectl get namespace
#NAME              STATUS   AGE
#default           Active   10d
#kube-node-lease   Active   10d
#kube-public       Active   10d
#kube-system       Active   10d
#web1              Active   28s

kubectl delete ns web1
```
- nginx.yaml
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
```
```bash

kubectl apply -f namespace.yaml

## web1 namespace에 배포 
kubectl apply -f nginx.yaml -n web1
```

## 1.2 web2 namespace
- httpd.yaml
```yaml
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
```bash

## web2 namespace에 배포 
kubectl apply -f httpd.yaml -n web2
```
## 1.3 ingress-rule
- ingress-rule은 각 namespace에 위치 해야 됨


- web1-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web1-ing
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
```
```bash

kubectl apply -f web1-ing.yaml -n web1
```
- ### 서비스 확인 : 브라우저에서
  - http://nginx.211.253.25.128.sslip.io


- web2-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web2-ing
spec:
  ingressClassName: nginx
  rules:
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
```bash

kubectl apply -f web2-ing.yaml -n web2
```
- ### 서비스 확인 : 브라우저에서
    - http://apache.211.253.25.128.sslip.io

---
# 2. ConfigMap
configmap을 통해서 nginx index.html을 변경

- configmap-example.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configmap
data:
  index.html: |     # 멀티라인 문자열 작성할 때 사용, 포멧 유지 -> HTML, JSON, YAML 등 사용
    <html>
        <body>
            <h1>Welcome to nginx!</h1>
            <h3> = nginx configmap test index html = </h3>
        </body>
    </html>
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
        volumeMounts:
          - name: nginx-index-config-vol
            mountPath:  /usr/share/nginx/html/index.html  # 컨테이너 내부의 파일 시스템 경로 지정
            subPath: index.html                 # 볼륨의 특정 파일이나 디렉터리만 마운트하는 데 사용(아래 volumes path 중 일부)
      volumes:
      - name: nginx-index-config-vol
        configMap:
          name: nginx-configmap
          items:
            - key: index.html                # ConfigMap의 특정 데이터 (여기서는 data.index.html: )
              path: index.html               # ConfigMap key에 해당하는 값(데이터) 마운트할 사용 할 내부 파일 이름
---
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
```
```bash

## 기존 배포된 nginx-deployemt가 변경
kubectl apply -f configmap-example.yaml -n web1
```
- ### 서비스 확인 : 브라우저에서
  - http://nginx.211.253.25.128.sslip.io/

---
# 3. secret
## 3.1 secretRef 기반 참조
- secret-example.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: db
data:
  MYSQL_DATABASE: freesia   ## 생성할 database 
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: db
stringData:
  MYSQL_ROOT_PASSWORD: "admin1234"  ## db 비번을 secret에 설정 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: db
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mariadb:10.7
        envFrom:                  # ConfigMap 및 Secret의 모든 키-값 쌍을 환경 변수로 컨테이너에 주입
        - configMapRef:
            name: mysql-config    # ConfigMap에 정의된 key mysql-config의 모든 데이터가 자동으로 환경 변수로 추가
        - secretRef:
            name: mysql-secret    # Secret에 정의된 key mysql-secret의 모든 데이터가 자동으로 환경 변수로 추가
```
```bash

## db namespace를 생성
kubectl create ns db
## mysql 배포한
## namespace yaml에 설정되어 있음
kubectl apply -f secret-example.yaml 

kubectl get pod -n db
# 결과 조회
#root@master01:~/kubernetes/lecture5# kubectl get pod -n db
#NAME                     READY   STATUS    RESTARTS   AGE
#mysql-647f8f98fd-kjhbq   1/1     Running   0          17s

## db namespace의 mysql pod로 들어가서 실행
kubectl exec -it mysql-647f8f98fd-kjhbq -n db -- /bin/bash
```
- mysqlDB 파드에서 실행
```bash
mysql -uroot -padmin1234
show databases;

#MariaDB [(none)]> show databases;
#+--------------------+
#| Database           |
#+--------------------+
#| freesia            |
#| information_schema |
#| mysql              |
#| performance_schema |
#| sys                |
#+--------------------+
## "freesia" database 존재 확인

use freesia

CREATE TABLE users (
    no SERIAL PRIMARY KEY,
    id VARCHAR(100) NOT NULL,   
    name VARCHAR(100) NOT NULL,  
    email VARCHAR(255)  
);
 
insert into users(id, name, email) values('cho', 'ydcho', 'yeongdeok.cho@kt.com');  
select * from users;
commit;

# DB 작업 결과
#MariaDB [(none)]> use freesia
#Database changed
#MariaDB [freesia]> CREATE TABLE users (
#    ->     no SERIAL PRIMARY KEY,
#    ->     id VARCHAR(100) NOT NULL,
#    ->     name VARCHAR(100) NOT NULL,
#    ->     email VARCHAR(255)
#    -> );
#Query OK, 0 rows affected (0.008 sec)

#MariaDB [freesia]> show tables;
#+-------------------+
#| Tables_in_freesia |
#+-------------------+
#| users             |
#+-------------------+
#1 row in set (0.000 sec)
#
#MariaDB [freesia]> insert into users(id, name, email) values('cho', 'ydcho', 'yeongdeok.cho@kt.com');
#Query OK, 1 row affected (0.001 sec)
#
#MariaDB [freesia]> select * from users;
#+----+-----+-------+----------------------+
#| no | id  | name  | email                |
#+----+-----+-------+----------------------+
#|  1 | cho | ydcho | yeongdeok.cho@kt.com |
#+----+-----+-------+----------------------+
#1 row in set (0.000 sec)
```
## 3.2 secretKeyRef 기반 참조
- secret-example2.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: db
data:
  MYSQL_DATABASE: freesia ## 생성할 database 
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: db
stringData:
  MYSQL_ROOT_PASSWORD: "admin123456"   ## db 비번을 다시 변경

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: db
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mariadb:10.7
        envFrom:                          # ConfigMap 및 Secret의 모든 키-값 쌍을 환경 변수로 컨테이너에 주입
        - configMapRef:
            name: mysql-config
        env:                              # 특정 키만 선택적으로 환경 변수로 설정
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:                 # Secret을 참고한다고 정의하는 블록
              name: mysql-secret          # 참조 할 secret name 참고한다고 정의
              key: MYSQL_ROOT_PASSWORD    # 참조 할 key
```
```bash

## 기존 mysql를 삭제한다 
kubectl delete -f secret-example.yaml

kubectl apply -f secret-example2.yaml

## db namespace의 mysql pod로 들어가서 실행
kubectl get pod -n db
# 조회 결과
#root@master01:~/kubernetes/lecture5# kubectl get pod -n db
#NAME                     READY   STATUS    RESTARTS   AGE
#mysql-646865fcb5-k72md   1/1     Running   0          23s

kubectl exec -it mysql-646865fcb5-k72md -n db -- bash
```
- mysqlDB 파드에서 실행
```bash

mysql -uroot -padmin123456!
show databases;

#MariaDB [(none)]> show databases;
#+--------------------+
#| Database           |
#+--------------------+
#| freesia            |
#| information_schema |
#| mysql              |
#| performance_schema |
#| sys                |
#+--------------------+
## "freesia" database 존재 확인
```

## 3.3 차이점 요약
| 구분          | envFrom                           | env                                 |  
|-------------|-----------------------------------|-------------------------------------|
|설정 대상      | ConfigMap 또는 Secret의 모든 키-값       | 개별 환경 변수(특정 키-값)             | 
|사용 목적      | ConfigMap/Secret의 전체 데이터를 한 번에 설정 | 환경 변수를 개별적으로 명시하여 설정    | 
|세부 제어      | 불가능 (모든 키-값이 환경 변수로 주입됨) | 가능 (필요한 키-값만 선택적으로 설정 가능)    |
|환경 변수 이름 충돌 | 충돌 가능 (모든 키가 주입되므로) | 충돌 위험 없음 (키를 명시적으로 설정하므로)    |
|사용 방식의 간결함  |간단하게 전체 데이터를 가져옴 | 필요한 키만 선택하므로 다소 번거로움   |
|보안성        |민감한 데이터도 모두 환경 변수로 노출될 위험 있음 | 민감한 데이터를 특정 키만 선택해 노출 방지 가능   |


# 4. clear
```sh

kubectl delete -f configmap-example.yaml -n web1
kubectl delete -f nginx.yaml -n web1
kubectl delete -f web1-ing.yaml -n web1
kubectl delete -f httpd.yaml -n web2
kubectl delete -f web2-ing.yaml -n web2
kubectl delete -f secret-example.yaml
kubectl delete -f secret-example2.yaml

kubectl delete ns db
kubectl delete -f namespace.yaml
```
