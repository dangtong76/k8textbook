# Cloud Native 를위한 도커 와 쿠버네티스 6부 - 리소스 제어

## 리소스 제어

###  부하 발생용 애플리 케이션 작성

####  PHP 애플리 케이션 작성

파일명 : index.php

```{php}
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

####  도커 이미지 빌드

```{dockerfile}
FROM php:5-apache
ADD index.php /var/www/html/index.php
RUN chmod a+rx index.php
```



```{bash}
docker build -t dangtong/php-apache .
docker login
docker push dangtong/php-apache
```

###   Deployment 및 서비스 생성

####  Deployment 생성

파일명 : php-apache-deploy.yaml

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache-dp
spec:
  selector:
    matchLabels:
      app: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: dangtong/php-apache
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```

####  로드밸런서  생성

파일명 : php-apache-svc.yaml

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: php-apache-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: php-apache
```

```{bash}
kubectl apply -f ./php-apache-deploy.yaml
kubectl apply -f ./php-apache-svc.yaml
```

###  HPA 리소스 생성

####  HPA 리소스 생성

```{bash}
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5
```

####  HPA 리소스 생성 확인

```{bash}
kubectl get hpa
```

###  Jmeter 를 이용한 부하 발생 테스트 

####  Jmeter 설치를 위해 필요한 것들

- JDK 1.8 이상 설치 [오라클 java SE jdk 다운로드](https://www.oracle.com/java/technologies/javase-downloads.html)

- Jmeter 다운로드 [Jmeter 다운로드](https://jmeter.apache.org/download_jmeter.cgi)

  플러그인 다운로드 [Jmeter-plugin 다운로드](https://jmeter-plugins.org/install/Install/)

  - Plugins-manager.jar 다운로드 하여 jmeter 내에 lib/ext 밑에 복사 합니다.  jpgc

####  Jmeter 를 통한 부하 발생

![jmeter](/Users/dangtongbyun/Dropbox/05.Lecture/01.Kubernetes/textbook/강의교재/textbook/img/jmeter.png)

####  부하 발생후 Pod 모니터링

```{bash}
$ kubectl get hpa

$ kubectl top pods

NAME                          CPU(cores)   MEMORY(bytes)
nodejs-sfs-0                  0m           7Mi
nodejs-sfs-1                  0m           7Mi
php-apache-6997577bfc-27r95   1m           9Mi


$ kubectl exec -it nodejs-sfs-0 top

Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  4.0 us,  1.0 sy,  0.0 ni, 95.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   3786676 total,  3217936 used,   568740 free,   109732 buffers
KiB Swap:        0 total,        0 used,        0 free.  2264392 cached Mem
    PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
      1 root      20   0  813604  25872  19256 S  0.0  0.7   0:00.17 node
     11 root      20   0   21924   2408   2084 R  0.0  0.1   0:00.00 top
```

###  연습문제

- 아래 요구 사항에 맞는 Deployment 를 구현 하세요

| 항목            | 내용  |
| --------------- | ----- |
| Deployment 이름 | nginx |
| 도커 이미지     | nginx |
| CPU MAX         | 200m  |
| 메모리 MAX      | 300Mi |
| 포트            | 80    |



- 생성된 Deployment 에 아래 요건으로 HPA를 적용하세요

| 항목       | 내용                 |
| ---------- | -------------------- |
| Max        | 8                    |
| Min        | 2                    |
| CPU 사용율 | 40% 이상되면 Scaling |

## 노드 패치및 업그레이드를 위한 Cordon 과 Drain 사용하기

###  Cordon 사용하기

```{bash}
kubectl get nodes

NAME                                       STATUS   ROLES    AGE   VERSION
gke-cluster-1-default-pool-20e07d73-4ksw   Ready    <none>   28h   v1.14.10-gke.27
gke-cluster-1-default-pool-20e07d73-6mr8   Ready    <none>   28h   v1.14.10-gke.27
gke-cluster-1-default-pool-20e07d73-8q3g   Ready    <none>   28h   v1.14.10-gke.27
```

- cordon 설정

```{bash}
kubectl cordon gke-cluster-1-default-pool-20e07d73-6mr8
```

```{bash}
kubectl get nodes

NAME                                       STATUS                     ROLES    AGE   VERSION
gke-cluster-1-default-pool-20e07d73-4ksw   Ready                      <none>   28h   v1.14.10-gke.27
gke-cluster-1-default-pool-20e07d73-6mr8   Ready,SchedulingDisabled   <none>   28h   v1.14.10-gke.27
gke-cluster-1-default-pool-20e07d73-8q3g   Ready                      <none>   28h   v1.14.10-gke.27
```

- 실제로 스케줄링 되지 않는지 확인해보기

```{bash}
kubectl run nginx --image=nginx:1.7.8 --replicas=4 --port=80
```

- 스케줄링 확인

```{bash}
kubectl get po -o wide

NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE
   NOMINATED NODE   READINESS GATES
nginx-84569d7db5-cgf95   1/1     Running   0          9s    10.4.0.17   gke-cluster-1-default-pool-20e07d73-8q3g
   <none>           <none>
nginx-84569d7db5-d7cgg   1/1     Running   0          8s    10.4.1.8    gke-cluster-1-default-pool-20e07d73-4ksw
   <none>           <none>
nginx-84569d7db5-gnbnq   1/1     Running   0          9s    10.4.1.9    gke-cluster-1-default-pool-20e07d73-4ksw
   <none>           <none>
nginx-84569d7db5-xqn7b   1/1     Running   0          9s    10.4.0.16   gke-cluster-1-default-pool-20e07d73-8q3g
   <none>           <none>
```

- uncodon 설정

```{bash}
kubectl uncordon gke-cluster-1-default-pool-20e07d73-6mr8
```

```{bash}
kubectl get nodes
```

### Drain 사용하기

- Drain 설정

```{bash}
kubeclt get po -o wide

NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE
     NOMINATED NODE   READINESS GATES
nginx-84569d7db5-cgf95   1/1     Running   0          3m44s   10.4.0.17   gke-cluster-1-default-pool-20e07d73-8q
3g   <none>           <none>
nginx-84569d7db5-d7cgg   1/1     Running   0          3m43s   10.4.1.8    gke-cluster-1-default-pool-20e07d73-4k
sw   <none>           <none>
nginx-84569d7db5-gnbnq   1/1     Running   0          3m44s   10.4.1.9    gke-cluster-1-default-pool-20e07d73-4k
sw   <none>           <none>
nginx-84569d7db5-xqn7b   1/1     Running   0          3m44s   10.4.0.16   gke-cluster-1-default-pool-20e07d73-8q
3g   <none>           <none>
```

- Pod 가 존재 하는 노드를 drain 모드로 설정하기

```{bash}
kubectl drain gke-cluster-1-default-pool-20e07d73-8q3g

error: unable to drain node "gke-cluster-1-default-pool-20e07d73-8q3g", aborting command...
There are pending nodes to be drained:
 gke-cluster-1-default-pool-20e07d73-8q3g
error: cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/fluentd-gcp-v3.1.1-
t6mnn, kube-system/prometheus-to-sd-96fdn
```

- daemonset 이 존재 하는 노드일 경우 옵션 추가 해서 drain 시킴

```{bash}
kubectl drain gke-cluster-1-default-pool-20e07d73-8q3g --ignore-daemonsets
```

- drian 확인

```{bash}
kubectl get po -o wide

NAME                     READY   STATUS    RESTARTS   AGE     IP         NODE
    NOMINATED NODE   READINESS GATES
nginx-84569d7db5-8qrd2   1/1     Running   0          59s     10.4.2.7   gke-cluster-1-default-pool-20e07d73-6mr
8   <none>           <none>
nginx-84569d7db5-d7cgg   1/1     Running   0          8m20s   10.4.1.8   gke-cluster-1-default-pool-20e07d73-4ks
w   <none>           <none>
nginx-84569d7db5-gnbnq   1/1     Running   0          8m21s   10.4.1.9   gke-cluster-1-default-pool-20e07d73-4ks
w   <none>           <none>
nginx-84569d7db5-s6xsm   1/1     Running   0          59s     10.4.2.9   gke-cluster-1-default-pool-20e07d73-6mr
8   <none>           <none>
```

> --delete-local-data --force 등의 추가 옵션 있음

- uncordon

```{bash}
kubectl drain gke-cluster-1-default-pool-20e07d73-8q3g
```

## 