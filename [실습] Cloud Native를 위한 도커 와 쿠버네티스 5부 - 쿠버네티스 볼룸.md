# Cloud Native를 위한 도커 와 쿠버네티스 5부 - 볼륨

## Volume

###   EmptyDir (GCP 에서 수행)

####   Docker 이미지 만들기

아래와 같이 폴더를 만들고 ./fortune/docimg 폴더로 이동합니다.

```{bash}
$ mkdir -p ./fortune/docimg
$ mkdir -p ./fortune/kubetmp

```

아래와 같이 docker 이미지를 작성하기 위해 bash 로 Application을 작성 합니다.

파일명 : fortuneloop.sh

```{bash}
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep 10
done
```

Dockerfile 을 작성 합니다.

```{dockerfile}
FROM ubuntu:latest
RUN apt-get update; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod 755 /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh
```

Dcoker 이미지를 만듭니다.

```{bash}
$ docker build -t dangtong/fortune .
```

Docker 이미지를 Docker Hub 에 push 합니다.

```{bash}
$ docker login
$ docker push dangtong/fortune
```

####   Deployment 작성

fortune APP을 적용하기 위해 Deployment 를 작성 합니다.

```{bash}
cd ../ktmp/
vi fortune-deploy.yaml
```

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortune-deployment
  labels:
    app: fortune
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fortune
  template:
    metadata:
      labels:
        app: fortune
    spec:
      containers:
      - image: dangtong/fortune
        name: html-generator
        volumeMounts:
        - name: html
          mountPath: /var/htdocs
      - image: nginx:alpine
        name: web-server
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
        ports:
          - containerPort: 80
            protocol: TCP
      volumes:
      - name: html
        emptyDir: {}
```

> html 볼륨을 html-generator 및 web-seerver 컨테이너에 모두 마운트 하였습니다.
>
> html 볼륨에는 /var/htdocs 및 /usr/share/nginx/html 이름 으로 서로 따른 컨테이너에서 바라 보게 됩니다.
>
> 다만, web-server 컨테이너는 읽기 전용(reeadOnly) 으로만 접근 하도록 설정 하였습니다.

> emptDir 을 디스크가 아닌 메모리에 생성 할 수도 있으며, 이를 위해서는 아래와 같이 설정을 바꾸어 주면 됩니다.
>
> emptyDir:
>
> medium: Memory

####  LoadBalancer 작성

```{bash}
vi fortune-lb.yaml
```

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: fortune-lb
spec:
  selector:
    app: fortune
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
  externalIPs:
  - 192.168.56.108
```

####  Deployment 및 Loadbalancer 생성

```{bash}
$ kubectl apply -f ./fortune-deploy.yaml
$ kubectl apply -f ./fortune-lb.yaml
```

#### 서비스 확인

```{bash}
curl http://192.168.56.108
```

###  Git EmptyDir

####  웹서비스용 Git 리포지토리 생성

Appendix3 . Git 계정 생성 및 Sync 참조

####  Deployment 용 yaml 파일 작성

```{bash}
$ cd ./gitvolume/kubetmp
$ vi gitvolume-deploy.yaml
```

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitvolume-deployment
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
      containers:
      - image: nginx:alpine
        name: web-server
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
        ports:
          - containerPort: 80
            protocol: TCP
      volumes:
      - name: html
        gitRepo:
          repository: https://github.com/dangtong76/k8s-web.git
          revision: master
          directory: .

```

####  Deployment 생성

```{bash}
$ kubectl apply -f ./gitvolume-deploy.yaml
```

####  Service 생성

```{bash}
apiVersion: v1
kind: Service
metadata:  name: gitvolume-lbspec:  selector:    app: nginx  ports:    - port: 80      targetPort: 80  type: LoadBalancer
```



###  GCE Persisteent DISK 사용하기

#### Persistent DISK 생성

- 리전/존 확인

```{bash}
$ gcloud container clusters list
```

- Disk 생성

```{bash}
$ gcloud compute disks create --size=16GiB --zone asia-northeast1-b  mongodb
#삭제 gcloud compute disks delete mongodb --zone asia-northeast1-b
```

####  Pod 생성을 위한 yaml 파일 작성

- 파일명 : gce-pv.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:  
  name: mongodb
spec:  
  containers:  
  - image: mongo    
    name:  mongodb    
    volumeMounts:    
    -  name: mongodb-data       
       mountPath: /data/db    
    ports:    
    - containerPort: 27017      
      protocol: TCP
  volumes:  
  - name: mongodb-data    
    gcePersistentDisk:      
      pdName: mongodb      
      fsType: ext4  
```

- Pod 생성

```{bash}
$ kubectl apply -f ./gce-pv.yaml$ kubectl get poNAME      READY   STATUS    RESTARTS   AGEmongodb   1/1     Running   0          8m42s
```

- Disk 확인

```{bash}
$ kubectl describe pod mongodb

...(중략)

Volumes:
  mongodb-data:
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     mongodb  # 디스크이름
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
  default-token-dgkd5:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-dgkd5
    Optional:    false

...(중략)

```

####  Mongodb 접속 및 데이터 Insert

- 접속

```{bash}
kubectl exec -it mongodb -- mongo
```

- 데이터 Insert

```{bash}
> use mystore
> db.foo.insert({"first-name" : "dangtong"})

> db.foo.find()
{ "_id" : ObjectId("5f9c4127caf2e6464e18331c"), "first-name" : "dangtong" }

> exit
```

####  MongoDB Pod 재시작

- MongoDB 중단

```{bash}
$ kubectl delete pod mongodb
```

- MongoDB Pod 재생성

```{bash}
$ kubectl apply -f ./gce-pv.yaml
```

- 기존에 Insert 한 데이터 확인

```{bash}
$ kubectl exec -it mongodb mongo

> use mystore

> db.foo.find()
{ "_id" : ObjectId("5e9684134384860bc207b1f9"), "first-name" : "dangtong" }
```

####  Pod 삭제

```{bash}
$ kubectl delete po mongodb
```



###  PersistentVolume 및 PersistentVolumeClaim

####  PersistentVolume 생성

- gce-pv2.yaml 로 작성

```{yaml}
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
   pdName: mongodb
   fsType: ext4
```

```{bash}
kubectl apply -f ./gce-pv2.yaml
```

```{bash}
kubectl get pv
```

####  PersistentVolumeClaim 생성

gce-pvc.yaml 로 작성

```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

```{bash}
kubectl apply -f ./gce-pvc.yaml
```

```{bash}
kubectl get pvc
```

####  PV, PVC 를 이용한 Pod 생성

gce-pod.yaml 파일 생성

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

```bash
$ kubectl apply -f ./gce-pod.yaml
```

```{bash}
$ kubectl get po,pv,pvc
```

####  Mongodb 접속 및 데이터 확인

```{bash}
$ kubectl exec -it mongodb -- mongo

> use mystore

> db.foo.find()

```





###  Persistent Volume 의 동적 할당

####  StorageClass 를 이용해 스토리지 유형 정의

- 클라우드에서 제공하는 Default Storage Class  확인 해보기

```{bash}
kubectl get sc

NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
fast                 kubernetes.io/gce-pd    Delete          Immediate              false                  3m37s
premium-rwo          pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   10d
standard (default)   kubernetes.io/gce-pd    Delete          Immediate              true                   10d
standard-rwo         pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   10d
```

- 상세 내역 확인

```{bash}
kubectl describe sc standard

Name:                  standard
IsDefaultClass:        Yes
Annotations:           storageclass.kubernetes.io/is-default-class=true
Provisioner:           kubernetes.io/gce-pd
Parameters:            type=pd-standard # 일반 디스크
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>

kubectl describe sc premium-rwo

Name:                  premium-rwo
IsDefaultClass:        No
Annotations:           components.gke.io/component-name=pdcsi-addon,components.gke.io/component-version=0.8.7
Provisioner:           pd.csi.storage.gke.io
Parameters:            type=pd-ssd  # SSD 
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
```

- Storage Class 생성하기 위한 Zone 확인

```{bash}
gcloud container clusters list

NAME      LOCATION           MASTER_VERSION    MASTER_IP       MACHINE_TYPE  NODE_VERSION      NUM_NODES  STATUS
istiok8s  asia-northeast1-a  1.18.12-gke.1210  35.189.139.222  e2-medium     1.18.12-gke.1210  3          RUNNING
```

- GCP 지원 디스크 종류 확인 하기

```{bash}
gcloud compute disk-types list | grep asia-northeast3-a

local-ssd    asia-northeast1-a          375GB-375GB
pd-balanced  asia-northeast1-a          10GB-65536GB
pd-ssd       asia-northeast1-a          10GB-65536GB
pd-standard  asia-northeast1-a          10GB-65536GB
```

- Stroage Class 생성 (파일명 : sc.yaml)

```{yaml}
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zone: asia-northeast3-a  #클러스터를 만든 지역으로 설정 해야함
```

```{bash}
$ kubectl apply -f ./gce-sclass.yaml
```

####  Storage Class 이용한 PVC 생성

- gce-pvc-with-sclass.yaml

```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: pvc-fast-10g
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 10Gi
  accessModes:
    - ReadWriteOnce
```

```{bash}
$ kubectl apply -f ./gce-pvc-sclass.yaml
```



####  PV 및 PVC 확인

- pvc 확인

```{bash}
$ kubectl get pvc mongdb-pvc
```

- pv 확인

```{bash}
$ kubectl get pv
```

####  PVC 를 이용한 POD 생성

파일명 : gce-pod.yaml 파일 생성

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```



### 연습문제

- 아래와 같은 요건으로 Volume 구성 요소를 생성하세요

| 종류                    | 이름                      | 요건                                                        |
| ----------------------- | ------------------------- | ----------------------------------------------------------- |
| Storage Class           | fast-ssd-retain-resizable | 회수정책: Retain / 생성 후 크기조정 : True                  |
| Persistent Volume Claim | pvc-fast-10g-retain       | 용량 : 10G  / accessModes : [ReadWriteOnce , ReadOnlyMany ] |

> allowVolumeExpansion: true 
> reclaimPolicy: Retain



## ConfigMap

###  도커에서 매개변수 전달

####  디렉토리 생성 및 소스 코드 작성

```{bash}
mkdir -p ./configmap/docimg/indocker
mkdir -p ./configmap/kubetmp

cd ./configmap/docimg/indocker
vi fortuneloop.sh
```

```{bash}
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1 #파라메터로 첫번째 매개 변수를 INTERVAL 로 저장
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```

```{bash}
chmod 755 fortuneloop.sh
```

####  Docker 이미지 생성

> Docker CMD 는 3가지 형태로 사용 가능 합니다.
>
> - CMD [ "command" ,  "파라메터1" , "파라메터2" ] → **실행파일 형태**
> - CMD command 파라메터1 파라매터2 → **쉘 명령어 형태**
> - CMD [ "파라메터1" , "파라메터2"]  → **ENTRYPOINT 의 디펄트 파라메터**

```{bash}
$ vi dockerfile
```

```{dockerfile}
FROM ubuntu:latest
RUN apt-get update;  apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod 755 /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]  # args가 없으면 10초
```

```{bash}
$ docker build -t dangtong/fortune:args .
$ docker push dangtong/fortune:args
```

> 테스트를 위해 다음과 같이 수행 가능
>
> $ docker run -it dangtong/fortune:args # 기본 10초 수행
>
> $ docker rm -f [docker-id]
>
> $ docker run -it dangtong/fortune:args 15 # 15초로 매개변수 전달
>
> $ docker rm -f [docker-id]

####  Pod생성

파일명 : config-fortune-pod.yaml

```{bash}
cd ../../kubetmp
vi config-fortune-indocker-pod.yaml
```

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune5s
spec:
  containers:
  - image: dangtong/fortune:args
    args: ["5"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}

```

```{bash}
kubectl apply -f ./config-fortune-indockeer-pod.yaml
```

###   Yaml 파일을 통한 매개변수 전달(환경변수)

####  fortuneloop.sh 작성

```{bash}
cd ../
mkdir -p ./docimg/inyaml
cd ./docimg/inyaml
vi fortuneloop.sh
```

```{bash}
#!/bin/bash
# 매개변수르 받지 않고 환경변로에서 바로 참조
trap "exit" SIGINT
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```

####  Dockerfile 작성 및 build

```{bash}
$ vi dockerfile
```

```{dockerfile}
FROM ubuntu:latest
RUN apt-get update;  apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod 755 /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
```

```{bash}
$ docker build -t dangtong/fortune:env .
$ docker push dangtong/fortune:env
```

####  Pod 생성및 매개변수 전달을 위한 yaml 파일 작성

```{bash}
cd ../../kubetmp
vi config-fortune-inyaml-pod.yaml
```

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune30s
spec:
  containers:
  - image: dangtong/fortune:env
    env:
    - name: INTERVAL 
      value: "30"
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

###  ConfigMap 을 통한 설정

####  ConfigMap 생성 및 확인

- configMap 생성

```{bash}
$ kubectl create configmap fortune-config --from-literal=sleep-interval=7
```

> ConfigMap 생성시 이름은 영문자,숫자, 대시, 밑줄, 점 만 포함 할 수 있습니다.
>
> 만약 여러개의 매개변수를 저장 할려면 --from-literal=sleep-interval=5 --from-literal=sleep-count=10 와 같이 from-literal 부분을 여러번 반복해서 입력 하면 됩니다.

- configMap 확인

```{bash}
$ kubectl get cm fortune-config

NAME             DATA   AGE
fortune-config   1      15s

$ kubectl get configmap fortune-config. -o yaml

apiVersion: v1
data:
  sleep-interval: "7"
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-16T09:09:33Z"
  name: fortune-config
  namespace: default
  resourceVersion: "1044269"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 85723e0a-1959-4ace-9365-47c101ebef82
```

####  ConfigMap을 환경변수로 전달한는 yaml 파일 작성

- Yaml 파일 생성

```{bash}
vi config-fortune-mapenv-pod.yaml
```

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune7s
spec:
  containers:
  - image: dangtong/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
    
```

####  Pod 생성 및 확인

- Pod 생성

```{bash}
$ kubectl create -f ./config-fortune-mapenv-pod.yaml
```

- Pod 생성 확인

```{bash}
$ kubectl get po, cm

$ kubectl describe cm

Name:         fortune-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
sleep-interval:
----
7
Events:  <none>
```

- configMap 삭제

```{bash}
$ kubectl delete configmap fortune-config
```

###   ConfigMap Volume 사용 (파일숨김)

####  ConfigMap 생성

```{bash}
mkdir config-dir
cd config-dir
vi custom-nginx-config.conf
```

파일명 : custom-nginx-config.conf

```{conf}
server {
	listen				8080;
	server_name		www.acron.com;

	gzip on;
	gzip_types text/plain application/xml;
	location / {
		root	/usr/share/nginx/html;
		index	index.html index.htm;
	}
}
```

```{bash}
$ kubectl create configmap fortune-config --from-file=config-dir
```

####  configMap volume 이용 Pod 생성

- Yaml 파일 작성 : config-fortune-mapvol-pod.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: fortune-config
```

> 서버에 접속해서 디렉토리 구조를 한번 보는것이 좋습니다.
>
> kubectl exec -it nginx-pod bash

```{bash}
$ kubectl apply -f ./config-fortune-mapvol-pod.yaml
```

####  ConfigMap Volume 참조 확인

```{bash}
$ kubectl get pod -o wide

AME              READY   STATUS    RESTARTS   AGE     IP          NODE
nginx-configvol   1/1     Running   0          9m25s   10.32.0.2   worker01.acorn.com
```

- Response 에 압축(gzip)을 사용 하는지 확인

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080
```

- 마운트된 configMap 볼륨 내용확인

```{bash}
$ kubectl exec nginx-configvol -c web-server ls /etc/nginx/conf.d

nginx-config.conf


$ kubectl exec -it nginx-configvol sh
```

####  ConfigMap 동적 변경

- 사전 테스트

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
Content-Encoding: gzip
```

- configMap 변경

```{bash}
$ kubectl edit cm fortune-config


```

**gzip on** 부분을 **gzip off ** 로 변경 합니다.

```{bash}
apiVersion: v1
data:
  nginx-config.conf: "server {\n  listen\t80;\n  server_name\tnginx.acorn.com;\n\n
    \ gzip off;\n  gzip_types text/plain application/xml;\n  location / {\n    root\t/usr/share/nginx/html;\n
    \   index\tindex.html index.htm;\n  }  \n}\n\n\n"
  nginx-ssl-config.conf: '##Nginx SSL Config'
  sleep-interval: |
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-16T15:58:42Z"
  name: fortune-config
  namespace: default
  resourceVersion: "1115758"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 182302d8-f30f-4045-9615-36e24b185ecb
```

- 변경후 테스트

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
Content-Encoding: gzip
```

- nginx reload 명령으로 설정 갱신 

```{bash}
$ kubectl  exec nginx-configvol -c web-server -- nginx -s reload
```

- 최종 테스트

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
```

###  ConfigMap Volume(파일 추가)

####  configMap volume 이용 Pod 생성

- Yaml 파일 작성 : config-fortune-mapvol-pod2.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx-config.conf # 주의 : configmap 의 key 와 파일명이 일치 해야합니다.
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: fortune-config
      defaultMode: 0660
```

> nginx 1.7.9 이상 버전, 예를 들면 nginx:latest 로 하면 /etc/nginx/conf.d 폴더내에 default.conf 파일만 존재 합니다. Example_cat tssl.conf 파일은 없습니다. 테스트를 위해서 nginx:1.7.9 버전으로 설정 한것입니다.

```{bash}
$ kubectl apply -f ./config-fortune-mapvol-pod.yaml
```

####  서비스 및 ConfiMap 확인

- 서비스 확인

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080
```

- configMap 확인

```{bash}
$ kubectl exec nginx-configvol -c web-server ls /etc/nginx/conf.d
```

####  ConfigMap 추가

- configMap 추가

```{bash}
$ kubectl edit cm fortune-config
```

아래와 같이 nginx-ssl-config.conf 키값을 추가합니다.

```{bash}
apiVersion: v1data:  nginx-config.conf: "server {\n  listen\t80;\n  server_name\tnginx.acorn.com;\n\n    \ gzip on;\n  gzip_types text/plain application/xml;\n  location / {\n    root\t/usr/share/nginx/html;\n    \   index\tindex.html index.htm;\n  }  \n}\n\n\n"  nginx-ssl-config.conf: ""##Nginx SSL Config" # 이부분이 추가됨  sleep-interval: |    25kind: ConfigMapmetadata:  creationTimestamp: "2020-04-16T15:58:42Z"  name: fortune-config  namespace: default  resourceVersion: "1098337"  selfLink: /api/v1/namespaces/default/configmaps/fortune-config  uid: 182302d8-f30f-4045-9615-36e24b185ecb
```

####  ConfigMap 추가된 Pod 생성

- 설정 파일 추가 하기위해 yaml 파일 수정

파일명 : config-fortune-mapvol-pod3.yaml

```{yaml}
kind: Podmetadata:  name: nginx-configvolspec:  containers:  - image: nginx:1.7.9    name: web-server    volumeMounts:    - name: config #default.conf 파일 교체      mountPath: /etc/nginx/conf.d/default.conf      subPath: nginx-config.conf      readOnly: true    - name: config #example_ssl.conf 파일 교체      mountPath: /etc/nginx/conf.d/example_ssl.conf      subPath: nginx-ssl-config.conf      readOnly: true    ports:    - containerPort: 8080      protocol: TCP  volumes:  - name: config    configMap:      name: fortune-config      defaultMode: 0660
```

```{bash}
$ kubectl apply -f ./ config-fortune-mapvol-pod3.yaml
```

####  추가된 ConfigMap 확인

- 서비스 확인

```{bash}
$ curl http://10.32.0.4
```

- configMap 확인

```{bash}
$ kubectl exec -it nginx-configvol bash$ cd /etc/nginx/conf.d$ ls -al
```

## Secret

```{bash}
mkdir -p ./secret/certmkdir -p ./secret/configmkdir -p ./secret/kubetmp
```

###   Secret 생성 (fortune-https)

```{bash}
cd ./secret/cert
```

```{bash}
openssl genrsa -out https.key 2048 
openssl req -new -x509 -key https.key -out https.cert -days 360 -subj '/CN=*.acron.com'
```

```{bash}
kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert
```

###  SSL 용 niginx 설정 생성(fortune-config)

```{bash}
cd ./secret/configvi custom-nginx-config.conf
```

```{bash}
server {
	listen				8080;
  listen				443 ssl;
	server_name		www.acron.com;
	ssl_certificate		certs/https.cert;
	ssl_certificate_key	certs/https.key;
	ssl_protocols		TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers		HIGH:!aNULL:!MD5;
	gzip on;
	gzip_types text/plain application/xml;
	location / {
		root	/usr/share/nginx/html;
		index	index.html index.htm;
	}
}
```

```{bash}
vi sleep-interval7
```

```{bash}
kubectl create cm fortune-config --from-file=./config
```

###  POD 생성

- Pod 생성 : 8.3.2 의 yaml 파일 참조
- 파일명 : secret-pod.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: dangtong/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config # 추가
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs # 추가
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443 # 추가
  volumes:
  - name: html
    emptyDir: {}
  - name: config # 추가
    configMap:
      name: fortune-config
      items:
      - key: custom-nginx-config.conf
        path: https.conf
  - name: certs  #추가
    secret:
      secretName: fortune-https
```

```{bash}
kubectl apply -f ./secret-pod.yaml
```

###  서비스 확인

```{bash}
kubectl port-forward fortune-https 8443:443 &
```

```{bash}
curl https://localhsot:8443 -k
curl https://localhost:8443 -k -v
```

## 연습문제

###  Mysql 구성하기

####  서비스 구성

파일명 : mysql-svc.yaml

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```

> 해드리스(Headless) 서비스를 만들면 명시적인 클러스터 IP 가 할당되지 않고 라우팅 됩니다. 헤드리스 서비스는 다른 네임스페이스의 서비스에 라우팅이 필요 할때도 유용 합니다. 헤드리스 서비스의 ExternalName 을 사용하 다른 네임스페이스의 서비스를 연결 할 수 있습니다. 원래 ExternalName은 K8s 외부의 서비스를 K8s 내부 서비스인 것처럼 참조할수 있게 해주는 기능 입니다.
>
> - 셀렉터가 있는경우 : 엔드포인트 컨트롤러가 엔드포인트 레코드를 생성하고 DNS 를 업데이트 하여 파드를 가르키는 A레코드(IP주소) 반환 하여 라우팅이 가능 합니다.
>
> - 셀렉터기 없는 경우 : DNS 시스템이 CNAME레코드 구성 하거나 서비스 이름으로 구성된 엔트포인트 레코드 참조
>
> - A 레코드와  CNAME 의 차이
>
>   - A레코드 : 간단하게 도메인 이름에 IP Address 를 매핑하는 레코드
>
>     ```{bash}
>     Name: www.acorn.com
>     Address: 10.0.24.3
>     ```
>
>   - CNAME(Canonical Name) : 도메인의 별칭을 부여하는 방식. 도메인의 또 다른 도메인 이름, A 레코드는 직접적으로 IP가 할당 되기 때문에 IP가 변경되면 직접 도메인에 영향을 미치지만, CNAME은 도메인에 매핑되어 있기 때문에 IP 변경에 직접적인 영향을 받지 않음
>
>     ```{bash}
>     www.acornstudy.com -> study.acorn.com (CNAME) 
>     study.acorn.com -> 211.231.99.250 (A record)
>     ```
>
> 

####   볼륨 구성

파일명 : mysql-pvc.yaml

```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

####   Pod 구성

- mysql 비번을 secret 을 생성하여 저장 합니다.

```{bash}
kubectl create secret generic mysql-pass  --from-literal=password='admin123'
```



파일명 : mysql-deploy.yaml

```{yaml}
apiVersion: apps/v1 #  이전 버전은 apps/v1beta2 를 사용하세요
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

```{bash}
kubectl apply -f ./mysql-deploy.yaml
```

###  WordPress 서비스 구성

####  LoadBalancer 구성

파일명 : wordpress-svc.yaml

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

####  PVC 구성

파일명  : wordpress-pvc.yaml

```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

####  WordPress 구성

파일명 : wordpress-deploy.yaml

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```
