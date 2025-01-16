# lecture-6 : Volume(PV & PVC)
- Master Node에서 실행

```bash

# cd ~
# git clone https://github.com/yeongdeokcho/edu.git
cd  ~/edu/lecture6
```

# 1. emptyDir
```yaml
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
        - name: log-volume
          mountPath: /var/log/nginx
      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:2.1.0
        volumeMounts:
        - name: log-volume
          mountPath: /var/log/nginx
      volumes: # 볼륨 선언
      - name: log-volume
        emptyDir: {}
```
```bash 

kubectl apply -f emptydir-vol.yaml

# 컨네이너 내 접속
kubectl get pod

# 컨네이너 이름만 확인
kubectl get pod nginx-deployment-59b4f55b6f-cbwbx -o jsonpath='{.spec.containers[*].name}'
# 조회 결과 : 공백으로 분리된 문자열로 컨테이너 이름 출력
nginx fluent-bit

## emptyDir volume인 log-volume으로 설정해 놓았기 때문에 
##  fluent-bit container에서 이제 nginx 의 access.log  error.log 를 읽을수 있도록 가능해 졌다 
## nginx pod의 fluent-bit container로 접속하여 아래와 같이  nginx의 로그파일이 조회 되는지 확인

# 컨테이너 접속 
kubectl exec -it nginx-deployment-59b4f55b6f-cbwbx -c fluent-bit -n default -- bash
# 컨테이너 내에서 실행
ls -l /var/log/nginx

# describe 명령어로 컨네이너 이름 확인 방법
# kubectl describe pod nginx-deployment-59b4f55b6f-cbwbx
#kubectl describe pod nginx-deployment-59b4f55b6f-cbwbx
#Name:             nginx-deployment-59b4f55b6f-cbwbx
#... 생략
#Controlled By:  ReplicaSet/nginx-deployment-59b4f55b6f
#Containers:
#  nginx:
#    Container ID:   containerd://41b691ca6ec6599674c2f86c4ea9be09e63be873dad235a2c2071c9d92b0fbf7
#   ... 생략
#      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fhgfl (ro)
#  fluent-bit:
#    Container ID:   containerd://23a98674db932817c3cf8e2512f412b4990f1003d8144327b0a3abbad26a9553
#   ... 생략

## clear 
kubectl delete -f emptydir-vol.yaml
```

# 2. hostPath
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-hostpath-nginx
    image: nginx:1.17
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/zz.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      path: /var/local/freesia    # 파일 디렉터리가 생성되었는지 확인
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/freesia/zz.txt
      type: FileOrCreate
```
```sh
## nginx를 배포한다 
kubectl apply -f hostpath-vol.yaml

## pod가 배포된 node의 에서 디렉토리및 파일이 생성 되었는지 확인  
## pod가 배포된 노드 확인 
kubectl get pod test-webserver -o wide
## node에 로그인 하여 host에 생성 되었는지 조회 
ls /var/local/freesia
echo 'Hi, hostPath : zz.txt' >> /var/local/freesia/zz.txt


## pod의 디렉토리및 파일이 생성 되었는지 확인 : Master1 Node에서 
kubectl exec -it test-hostpath-nginx -- ls /var/local/freesia
kubectl exec -it test-hostpath-nginx -- cat /var/local/freesia/zz.txt


## 다른 Worker 노드에서 확인 (dir 생성 안됨) 
## ls /var/local/freesia 

## clear 
kubectl delete -f hostpath-vol.yaml
rm -rf /var/local/freesia
```

# 3. pv/pvc

## 3.1  노드에 index.html 파일 생성
```sh
# 사용자 노드에서 슈퍼유저로 명령을 수행하기 위하여
## worker1 에만 생성 
mkdir -p /mnt/data
sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
cat /mnt/data/index.html
```

## 3.2 storageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
```bash

kubectl apply -f task-pv-sc.yaml
kubectl get storageclass -n default
kubectl get sc -n default
# STATUS 확인
root@master01:~/kubernetes/lecture6# kubectl get storageclass -n default
NAME     PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
manual   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  2m15s
```

## 3.3 pv
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

```
```bash

kubectl apply -f task-pv-pv.yaml
kubectl get pv -n default
# STATUS 확인
root@master01:~/kubernetes/lecture6# kubectl get pv -n default
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
task-pv-volume   100Mi      RWO            Retain           Available           manual         <unset>                          40s
```
## 3.4 pvc
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 300Mi
```
```sh

kubectl apply -f task-pv-pvc.yaml

## pvc 조회,  status가 Pending 상태 확인  -> 사용하는 Pod가 한개도 없어서..   
kubectl get pvc
## pv 조회한다  status가 Available 상태 확인 
kubectl get pv -n default
```

## 3.5 pod 배포
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
```
```sh
## pod 배포
kubectl apply -f task-pv-pod.yaml
kubectl get pod -o wide

# pv, pvc STATUS : Bound 확인 -> Pending 상태면 다른 Node에도 dir 생성 
kubectl get pv
kubectl get pvc

## pod 안으로 들어간다 
kubectl exec -it task-pv-pod -- /bin/bash

## 최신버전 nginx image 들은 보안 때문에 curl 이 없을수 있음. curl 설치  
apt update
apt install curl
## nginx를 조회 해 본다 
curl http://localhost/    ## 'Hello from Kubernetes storage' 조회 안될 수 있음
## 파일이 존재하는지 확인 
cat /usr/share/nginx/html/index.html

## 조회 안될 경우 이는 /mnt/data/index.html 생성한 노드에 pod가 생성되지 않는 경우
## 다른 노드에 /mnt/data/index.html를 생성

```
## 3.6 clean
```bash
kubectl delete pod task-pv-pod
kubectl delete pvc task-pv-pvc
kubectl delete pv task-pv-pv
kubectl delete pv task-pv-sc

# Dir 생성한 모든 Worker 노드에서 
rm -pr /mnt/data
```

# 4. nfs(Network File System) 구성 

## 4.1 VM disk 할당
```bash

# NFS용 disk 추가 
  - host : master01
  - disk명 : mn01nfs01
  
# 아래 순서로 진행 : 
# 용량 확인 > 디스크 확인(볼륨 확인) > 포멧 > 마운트 포인트 생성 > 마운트 > 용량 확인 > fstab 등록   
df -h
fdisk -l
# 포멧 : 반드시 볼륨 확인하고 볼륨 수정 후 진행  
mkfs.ext4 /dev/xvdc
mkdir /data/nfs
mount /dev/xvdc /data/nfs
df -h
touch /data/nfs/aa.txt; ls -l /data/nfs

# 반드시 nfs용 볼륨, 마운트 포인트 확인, 수정 후 진행 
echo "/dev/xvdc /data/nfs ext4 defaults 0 2" >> /etc/fstab

cat /etc/fstab
```  

## 4.2 nfs 서버 구성

```bash

# Master Node 
apt update
apt-get install nfs-kernel-server

# 각자 Master 노드 사설 IP 대역 확인 후 수정 
echo "/data/nfs 172.27.0.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports

/usr/sbin/exportfs -a 
# sudo /usr/sbin/exportfs -r
systemctl restart nfs-kernel-server.service

# systemctl enable nfs-kernel-server.service
# systemctl stop nfs-kernel-server.service
# systemctl start nfs-kernel-server.service
# systemctl status nfs-kernel-server.service
``` 
- Master 노드 사설 IP
  ![Master Node 사설 IP](/lecture6/img/lecture6-nfs-master-ip.png)


## 4.3 nfs 클라이언트 구성

```bash

# Worker01, 02 Node 
apt update
apt-get install nfs-common

# nfs 마운트 가능 스토리지 조회 (각자 Master01 사설 IP로 변경)
showmount -e 172.27.0.179
# 조회 결과
root@worker02:~# showmount -e 172.27.0.179
Export list for 172.27.0.179:
/data/nfs 172.27.0.0/24

# 마운트
mkdir -p /data/nfs
mount 172.27.0.179:/data/nfs /data/nfs
ls -l /data/nfs

# 각자 master 노드 사설 IP 대역 확인 후 수정 : 재시동 이후에도 자동 인식하게 
echo "172.27.0.179:/data/nfs    /data/nfs    nfs    defaults    0 0" >> /etc/fstab 
 
cat /etc/exports
``` 


## 4.4 nfs provisioner 설치
### 4.4.1 설치 파일 다운로드
```bash

# Master01 Node 
mkdir -p ~/kubernetes/lecture6/manifest/nfs
cd ~/kubernetes/lecture6/manifest/nfs
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
cd nfs-subdir-external-provisioner/deploy
```

### 4.4.2 NFS 서버 접속 정보 설정
- deployment.yaml 수정
```yaml
#### deployment 내 nfs 환경 설정 수정 : 
env:
  - name: PROVISIONER_NAME
    value: k8s-sigs.io/nfs-subdir-external-provisioner
  - name: NFS_SERVER
    value: 172.27.0.179           ## NFS 서버 IP (각자 Master01 사설 IP)
  - name: NFS_PATH
    value: /data/nfs              ## NFS Path
volumes:
  - name: nfs-client-root
    nfs:
      server: 172.27.0.179            ## NFS 서버 IP (각자 Master01 사설 IP)
      path: /data/nfs              ## NAS Path
```
```bash

# 위 yaml 파일 참고하여 nfs 서버 IP 변경
vi deployment.yaml
```
- class.yaml 수정
```yaml
#### storageClass 환경 설정 수정 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storageclass    ## Storage Class 이름 변경 -> 모두 소문자료.
allowVolumeExpansion: true  ## 용량증설 지원  
```

```bash

# 위 yaml 파일 참고하여 name, 용량증설 설정 추가
vi vi class.yaml
```
### 4.4.4 provisioner 배포 
```bash

kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
kubectl apply -f class.yaml
```
### 4.4.5 provisioner 서비스 확인
- storageClassName 이름 변경 후
```yaml
#### Storage Class 이름 변경 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-storageclass   # : nfs-client --> nfs-storageclass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```
- 테스트 pvc 배포, 확인
```bash

# 위 내용 참고하여 storageClass 이름 변경
vi test-claim.yaml.yaml
```
```bash
      
kubectl apply -f test-claim.yaml
kubectl get pv
kubectl get pvc
# pv, pvc 조회 결과 : pv를 생성하지 않았는데, 자동으로 생성 됨 
#root@master01:~/kubernetes/lecture6/manifest/nfs/nfs-subdir-external-provisioner/deploy# kubectl get pv
#NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS       VOLUMEATTRIBUTESCLASS   REASON   AGE
#pvc-e121bea0-a6c9-4fe2-9d7d-8b53588268d9   1Mi        RWX            Delete           Bound    default/test-claim   nfs-storageclass   <unset>                          20s
#root@master01:~/kubernetes/lecture6/manifest/nfs/nfs-subdir-external-provisioner/deploy# kubectl get pvc
#NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       VOLUMEATTRIBUTESCLASS   AGE
#test-claim   Bound    pvc-e121bea0-a6c9-4fe2-9d7d-8b53588268d9   1Mi        RWX            nfs-storageclass   <unset>                 57s
#root@master01:~/kubernetes/lecture6/manifest/nfs/nfs-subdir-external-provisioner/deploy#

kubectl apply -f test-pod.yaml
``` 
### 4.4.6 파일 생성 확인
- Worker01 서버에서
```bash

ls -l /data/nfs/
```
### 4.4.7 nfs 테스트 clear
```bash

kubectl delete -f test-pod.yaml
kubectl delete -f test-claim.yaml
``` 
