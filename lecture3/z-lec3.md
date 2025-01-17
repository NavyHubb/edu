# lecture-3 : DaemonSet & StatefulSet
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture3
```

# 1. DaemonSet
```dtd
- DaemonSet은 클러스터의 특정 노드에서 반드시 실행되어야 하는 Pod를 관리하기 위한 k8s 리소스로서, 이를 사용하면 노드마다 하나씩 고정된 Pod가 실행됨
  - 새로운 노드가 클러스터에 추가되면 DaemonSet에 의해 자동으로 해당 노드에서도 동일한 Pod가 생성됨
  - 위와 같은 특성에 따라 노드별 고유한 작업을 수행할 때 주로 사용됨
  - 대표적으로, 로그 수집
  - fluent bit은 대표적 로그 수집기중(LogStash, fluentd, fluentbit) 가장 가벼움(10배 이상 가볍다)
  - 분산 환경을 고려하여 만들어졌기에 최근 k8s의 로그 수집기로 사용
  - https://fluentbit.io/
```

- fluent-bit-daemonset.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
spec:
  selector:
    matchLabels:
      name: fluentbit
  template:
    metadata:
      labels:
        name: fluentbit
    spec:
      containers:
      - name: aws-for-fluent-bit
        image: amazon/aws-for-fluent-bit:2.32.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true       
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      
```

```sh

kubectl apply -f fluentbit-daemonset.yaml

## daemonset 조회
kubectl get daemonset
kubectl get ds
 
# replicas 설정 없음
# 각 node에 1개씩 fluentbit 파드가 생성
# 조회 결과
#root@master01:~/kubernetes/lecture3# kubectl get daemonset
#NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
#fluentbit   2         2         2       2            2           <none>          57s

## pod 조회
kubectl get pod -n default -o wide

# 실행 node 확인
#root@master01:~/kubernetes/lecture3# kubectl get pod -n default -o wide
#NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
#fluentbit-kdhrt                     1/1     Running   0          2m26s   10.42.2.38   worker02   <none>           <none>
#fluentbit-m54nm                     1/1     Running   0          2m26s   10.42.1.26   worker01   <none>           <none>


## clear
kubectl delete -f fluentbit-daemonset.yaml
```

---
# 2. StatefulSet
- StatefulSet은 Kubernetes에서 상태를 가지는 애플리케이션을 관리하기 위한 워크로드 API 오브젝트
- StatefulSet은 각 Pod에 고유한 ID와 네트워크 정체성을 부여하며, 데이터를 유지해야 하거나, 순서 보장이 필요한 워크로드에 적합

### 특징
1. 각 Pod에 고유한 정체성 보장
   - 각 Pod는 고유한 DNS 이름을 가짐으로써 네트워크 상 정체성을 가짐
2. 순차적 생성과 삭제
   - StatefulSet의 Pod은 순차적으로 생성되고 삭제됨
   - 삭제는 생성의 역순으로 진행됨
3. PVC와 통합
    - StatefulSet은 각 Pod에 독립적인 스토리지를 제공
    - 한 번 생성된 PersistentVolume은 삭제 후에도 유지되어, Pod이 재생성되더라도 동일한 데이터를 사용

- nginx-statefulset.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  selector:
    matchLabels:
      app: nginx 
  serviceName: "nginx-headless-svc"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx 
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
```
- 데모 위해 Volume 없이 생성한 예
- StatefulSet은 serviceName을 요구하며 해당 서비스는 headless (clusterIP: None) 이어야 한다

```sh

kubectl apply -f nginx-statefulset.yaml

## StatefulSet 조회
kubectl get statefulset
kubectl get sts
# 조회 결과
#root@master01:~/kubernetes/lecture3# kubectl get sts
#NAME                READY   AGE
#nginx-statefulset   3/3     102s

## pod 조회
kubectl get pod -n default -o wide

# 실행된 pod 이름 확인
#root@master01:~/kubernetes/lecture3# kubectl get pod -n default -o wide
#NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
#nginx-statefulset-0                 1/1     Running   0          2m40s   10.42.1.27   worker01   <none>           <none>
#nginx-statefulset-1                 1/1     Running   0          2m32s   10.42.2.39   worker02   <none>           <none>
#nginx-statefulset-2                 1/1     Running   0          2m25s   10.42.1.28   worker01   <none>           <none>
```

## headless service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx

```
- headless 서비스는 `clusterIP: None` 으로 설정 하면 된다

```sh

kubectl get svc 

# 조회 결과 : clusterIP: None 확인
#root@master01:~/kubernetes/lecture3# kubectl get svc
#NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
#kubernetes           ClusterIP   10.43.0.1    <none>        443/TCP   9d
#nginx-headless-svc   ClusterIP   None         <none>        80/TCP    5m8s

kubectl describe svc nginx-headless-svc
```
- 헤드리스 & ClusterIP
![서비스 비교](/lecture3/img/lecture3-diff-svc.png)
- 헤드리스
![헤드리스 서비스](/lecture3/img/lecture3-headless-svc.png)
- ClusterIP 
![ClusterIP 서비스](/lecture3/img/lecture3-clusterip-svc.png)
- cluster-ip에 ip가 할당 되지 않는다
- 따라서 다음과 같이 호출 해야 한다

```sh

## 서비스가 로드밸런스 하는지 확인해 본다 (3개 에 모두 적용)
## Master Node에서 실행
## nginx 3개 pod에 index.html 파일 생성해 LB 동작하는지 확인  
### k9s에서 진행하는것을 추천
kubectl get pods
kubectl  exec -it nginx-statefulset-0  -- bash

# 수정 전 index.html 내용 확인
cat /usr/share/nginx/html/index.html
# index.html 내용 수정
echo nginx-statefulset-0 > /usr/share/nginx/html/index.html
echo nginx-statefulset-1 > /usr/share/nginx/html/index.html
echo nginx-statefulset-2 > /usr/share/nginx/html/index.html

## mycurlpod에서 실행 
## 없다면 아래로 실행 
kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh

## cluster내에서 호출시 cluster domain 사용 
## [서비스명].[네임스페이스].svc.cluster.local
## [서비스명].[네임스페이스].svc
## [서비스명].[네임스페이스]  ## 다른 네임스페이스
## [서비스명] ## 같은 네임스페이스 
curl nginx-headless-svc.default.svc.cluster.local
curl nginx-headless-svc.default.svc
curl nginx-headless-svc

## cluster내 특정 기능을 수행하는 pod 호출 방법 
# [파드명].[서비스명].[네임스페이스].svc.cluster.local 
curl nginx-statefulset-0.nginx-headless-svc.default.svc.cluster.local
curl nginx-statefulset-1.nginx-headless-svc.default.svc.cluster.local
curl nginx-statefulset-2.nginx-headless-svc.default.svc.cluster.local

nslookup nginx-headless-svc.default.svc.cluster.local

#Server:         10.43.0.10
#Address:        10.43.0.10:53
#
#Name:   nginx-headless-svc.default.svc.cluster.local
#Address: 10.42.1.52
#Name:   nginx-headless-svc.default.svc.cluster.local
#Address: 10.42.1.51
#Name:   nginx-headless-svc.default.svc.cluster.local
#Address: 10.42.1.53
```

## 2.1  clear
```sh

kubectl delete -f nginx-statefulset.yaml
```