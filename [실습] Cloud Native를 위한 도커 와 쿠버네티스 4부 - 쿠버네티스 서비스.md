# Cloud Native를 위한 도커 와 쿠버네티스 4부 - 서비스

## nodes app 생성

###  app.js 소스코드 작성

```{javascript}
const http = require('http');
const os = require('os');

console.log("Kubia server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

###  Dockerfile 작성

```{bash}
# FROM 으로 BASE 이미지 로드
FROM node:7

# ADD 명령어로 이미지에 app.js 파일 추가
ADD app.js /app.js

# ENTRYPOINT 명령어로 node 를 실행하고 매개변수로 app.js 를 전달
ENTRYPOINT ["node", "app.js"]
```

###  pod 생성

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp-pod
  template:
    metadata:
      labels:
        app: nodeapp-pod
    spec:
      containers:
      - name: nodeapp-container
        image: dangtong/nodeapp
        ports:
        - containerPort: 8080
```

###  yaml을 통한 ClusterIP 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: nodeapp-service
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```

###  서비스 상태 확인

```{bash}
kubectl get  po,deploy,svc

NAME                                      READY   STATUS    RESTARTS   AGE
pod/nodeapp-deployment-55688d9d4b-8pzsk   1/1     Running   0          2m45s
pod/nodeapp-deployment-55688d9d4b-pslvb   1/1     Running   0          2m46s
pod/nodeapp-deployment-55688d9d4b-whbk8   1/1     Running   0          2m46s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nodeapp-deployment   3/3     3            3           2m46s

NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
nodeapp-service   ClusterIP      10.101.249.42    <none>           80/TCP         78s
```

###  서비스 확인

- local VirtualBox

```{bash}
curl http://10.101.249.42  #여러번 수행 하기

You've hit nodeapp-deployment-55688d9d4b-8pzsk
```

- GCP Cloud

먼저 Pod 를 조회 합니다.

```{bash}
kubectl get po, svc

NAME                              READY   STATUS    RESTARTS   AGE
nodeapp-deploy-6dc7c5dd68-lh26q   1/1     Running   0          116m
nodeapp-deploy-6dc7c5dd68-r78cj   1/1     Running   0          116m
nodeapp-deploy-6dc7c5dd68-wcm7d   1/1     Running   0          116m

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.116.0.1      <none>        443/TCP        32h
nodeapp-nodeport   NodePort    10.116.11.242   <none>        80:30123/TCP   121m
```

조회된 Pod 중 하나에  exec 옵션을 사용해 sh 로 접속 합니다.

```{bash}
kubectl exec nodeapp-deploy-6dc7c5dd68-lh26q -- sh
```

curl 을 설치 하고 Cluster IP 로 접속합니다.

```{bash}
apt-get install curl

curl http://10.116.11.242

```

###   원격 Pod에서 curl 명령 수행하기

```{bash}
kubectl exec nodeapp-deployment-55688d9d4b-8pzsk -- curl -s http://10.101.249.42

You've hit nodeapp-deployment-55688d9d4b-whbk8
```

> .더블 대시는 kubectl 명령의의 종료를 가르킴

###   서비스 삭제

```{bash}
kubectl delete svc nodeapp-service
```

## NodePort

###   yaml 을 이용한 NodePort 생성 (GCP 에서 수행 하기)

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: node-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app:  nodeapp-pod
```

###   NodePort 조회

```{bash}
kubectl get po,rs,svc

NAME                                      READY   STATUS    RESTARTS   AGE
pod/nodeapp-deployment-55688d9d4b-8pzsk   1/1     Running   0          145m
pod/nodeapp-deployment-55688d9d4b-pslvb   1/1     Running   0          145m
pod/nodeapp-deployment-55688d9d4b-whbk8   1/1     Running   0          145m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/nodeapp-deployment-55688d9d4b   3         3         3       145m

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes         ClusterIP   10.96.0.1      <none>        443/TCP        10d
service/nodeapp-nodeport   NodePort    10.108.30.68   <none>        80:30123/TCP   4m14s
```

###   NodePort 를 통한 서비스 접속 확인(여러번 수행)

- Vmware VM

```{bash}
$ curl http://localhost:30123
You've hit nodeapp-deployment-55688d9d4b-pslvb

$ curl http://localhost:30123
You've hit nodeapp-deployment-55688d9d4b-whbk8

$ curl http://localhost:30123
You've hit nodeapp-deployment-55688d9d4b-pslvb

```

- GCP Cloud

```{bash}
kubectl get no -o wide # 결과에서 External-IP 를 참조

NAME                                      STATUS   ROLES    AGE   VERSION             INTERNAL-IP   EXTERNAL-IP     OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-istiok8s-default-pool-36a6222b-33vv   Ready    <none>   32h   v1.18.12-gke.1210   10.146.0.21   35.221.70.145   Container-Optimized OS from Google   5.4.49+          docker://19.3.9
gke-istiok8s-default-pool-36a6222b-6rhk   Ready    <none>   32h   v1.18.12-gke.1210   10.146.0.20   35.221.82.6     Container-Optimized OS from Google   5.4.49+          docker://19.3.9
gke-istiok8s-default-pool-36a6222b-xrzj   Ready    <none>   32h   v1.18.12-gke.1210   10.146.0.22   34.84.27.67     Container-Optimized OS from Google   5.4.49+          docker://19.3.9
```

```{bash}
$ curl http://35.221.70.145:30123
You've hit nodeapp-deploy-6dc7c5dd68-wcm7d

$ curl http://35.221.70.145:30123
You've hit nodeapp-deploy-6dc7c5dd68-lh26q

$ curl http://35.221.70.145:30123
You've hit nodeapp-deploy-6dc7c5dd68-r78cj
```

###   NodePort 삭제

```{bash}
kubectl delete svc nodeapp-nodeport
```

##  LoadBalancer (GCP 에서 수행)

###   yaml 파일로 deployment 생성

```{bash}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deployment
  labels:
    app: nodeapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp-pod
  template:
    metadata:
      labels:
        app: nodeapp-pod
    spec:
      containers:
      - name: nodeapp-container
        image: dangtong/nodeapp
        ports:
        - containerPort: 8080
```

###  서비스 확인

```{bash}
$ kubectl get po,rs,deploy

NAME                                      READY   STATUS    RESTARTS   AGE
pod/nodeapp-deployment-7d58f5d487-7hphx   1/1     Running   0          20m
pod/nodeapp-deployment-7d58f5d487-d74rp   1/1     Running   0          20m
pod/nodeapp-deployment-7d58f5d487-r8hq8   1/1     Running   0          20m
NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.extensions/nodeapp-deployment-7d58f5d487   3         3         3       20m
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nodeapp-deployment   3/3     3            3           20m
```

```{bash}
kubectl get po -o wide

NAME                                  READY   STATUS    RESTARTS   AGE   IP           NODE                                  NOMINATED NODE   READINESS GATES
nodeapp-deployment-7d58f5d487-7hphx   1/1     Running   0          21m   10.32.2.10   gke-gke1-default-pool-ad44d907-cq8j
nodeapp-deployment-7d58f5d487-d74rp   1/1     Running   0          21m   10.32.2.12   gke-gke1-default-pool-ad44d907-cq8j
nodeapp-deployment-7d58f5d487-r8hq8   1/1     Running   0          21m   10.32.2.11   gke-gke1-default-pool-ad44d907-cq8j
```

###  nodeapp 접속 해보기

```{bash}
$ kubectl exec nodeapp-deployment-7d58f5d487-7hphx -- curl -s http://10.32.2.10:8080
또는
$ kubectl exec -it nodeapp-deployment-7d58f5d487-7hphx bash

$ curl http://10.32.2.10:8080
You've hit nodeapp-deployment-7d58f5d487-7hphx
```

###   yaml 파일을 이용해 LoadBalancer 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name:  nodeapp-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```

###  LoadBalancer 생성 확인

```{bash}
kubectl get svc

NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.36.0.1      <none>        443/TCP        7d21h
nodeapp-lb   LoadBalancer   10.36.14.234   <pending>     80:31237/TCP   33s
```

현재 pending 상태임 20초 정도 지나면

```{bash}
kubectl get svc

NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.36.0.1      <none>           443/TCP        7d21h
nodeapp-lb   LoadBalancer   10.36.14.234   35.221.179.171   80:31237/TCP   45s
```

###   서비스 확인

```{bash}
curl http://35.221.179.171

You've hit nodeapp-deployment-7d58f5d487-r8hq8
```

## Ingress (GCP 에서 수행)

###  Deployment 생성

- nginx / goapp deployment 생성

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      - name: nginx-container
        image: nginx:1.7.9
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapp-deployment
  labels:
    app: goapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: goapp
  template:
    metadata:
      labels:
        app: goapp
    spec:
      containers:
      - name: goapp-container
        image: dangtong/goapp
        ports:
        - containerPort: 8080
```

###   Service 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name:  nginx-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx

---
apiVersion: v1
kind: Service
metadata:
  name:  goapp-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: goapp
```



###   Ingress 생성

```{yaml}
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-goapp-ingress
spec:
  tls:
  - hosts:
    - nginx.acorn.com
    - goapp.acorn.com
    secretName: acorn-secret
  rules:
  - host: nginx.acorn.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-lb
          servicePort: 80
  - host: goapp.acorn.com
    http:
      paths:
      - path: /
        backend:
          serviceName: goapp-lb
          servicePort: 80
```

###   Ingress 조회

```{bash}
kubectl get ingress

NAME                  HOSTS                             ADDRESS   PORTS     AGE
nginx-goapp-ingress   nginx.acorn.com,goapp.acorn.com             80, 443   15s
```

> Ingress 가 완전히 생성되기 까지 시간이 걸립니다. 2~5분 소요

다시 조회 합니다

```{bash}
kubectl get ingress

NAME                  HOSTS                             ADDRESS          PORTS     AGE
nginx-goapp-ingress   nginx.acorn.com,goapp.acorn.com   35.227.227.127   80, 443   13m
```

###  /etc/hosts 파일 수정

```{bash}
sudo vi /etc/hosts

35.227.227.127 nginx.acorn.com goapp.acorn.com
```

###  서비스 확인

```{bash}
$ curl http://goapp.acorn.com
hostname: goapp-deployment-d7564689f-6rrzw

$ curl http://nginx.acorn.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>

```

###   HTTPS 서비스 (TLS OffLoad)

- 인증서 생성 및 인증서 secret 등록

```{bash}
openssl genrsa -out server.key 2048

openssl req -new -x509 -key server.key -out server.cert -days 360 -subj /CN=nginx.acorn.com

kubectl create  secret tls acorn-secret --cert=server.cert --key=server.key
```

- 테스트

```{bash}
$ curl -k https://nginx.acorn.com

$ curl -k https://goapp.acorn.com
```



