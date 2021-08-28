# Cloud Native를 위한 도커 와 쿠버네티스 2부 - 쿠버네티스 기본

## 쿠버네티스 간단하게 맛보기

###   도커 허브 이미지로 컨테이너 생성 및 확인

- 컨테이너 생성 : run/v1 으로 수행 합니다.

  ```{bash}
  # POD 및 Replication Controller 생성 (향후 버전에서 deprecated 될 예정)
  $ kubectl run goapp-project --image=dangtong/goapp --port=8080 --generator=run/v1
  # POD 만 생성
  $ kubectl run goapp-project --image=dangtong/goapp --port=8080 --generator=run-pod/v1
  ```

  > generator 를 run/v1 으로 수행 할 경우 내부적으로 goapp-project-{random-String} 이라는 컨테이너를 만들면서 goapp-project- 이름의 replication controller 도 생기게 됩니다.
  >
  > 

- kubectl CLI 생성 예제

  | Resource                             | API group          | kubectl command                                   |
  | :----------------------------------- | :----------------- | :------------------------------------------------ |
  | Pod                                  | v1                 | `kubectl run --generator=run-pod/v1`              |
  | ReplicationController _(deprecated)_ | v1                 | `kubectl run --generator=run/v1`                  |
  | Deployment _(deprecated)_            | extensions/v1beta1 | `kubectl run --generator=deployment/v1beta1`      |
  | Deployment _(deprecated)_            | apps/v1beta1       | `kubectl run --generator=deployment/apps.v1beta1` |
  | Job _(deprecated)_                   | batch/v1           | `kubectl run --generator=job/v1`                  |
  | CronJob _(deprecated)_               | batch/v2alpha1     | `kubectl run --generator=cronjob/v2alpha1`        |
  | CronJob _(deprecated)_               | batch/v1beta1      | `kubectl run --generator=cronjob/v1beta1`         |

  

- 컨테이너 확인

  ```{bash}
  $ kubectl get pods
  $ kubectl get rc
  
  ```

  

  ```{bash}
  $ kubectl get pods
  NAME             READY   STATUS    RESTARTS   AGE
  goapp-project-bcv5q   1/1     Running   0          2m26s
  
  $ kubectl get rc
  NAME            DESIRED   CURRENT   READY   AGE
  goapp-project   1         1         1       8m58s
  ```

  

  > 아래 명령어를 추가적으로 수행해 보세요
  >
  > ```{bash}
  > kubectl get pods -o wide
  > kubectl describe pod goapp-project-bcv5q2.1 도커 허브 이미지로 컨테이너 생성 및 확인
  > ```

###   서비스 생성 및 테스트 

- k8s 서비스 생성

  ```{bash}
  $ kubectl expose rc goapp-project --type=LoadBalancer --name goapp-http
  ```

  ```{text}
  service/firstapp-http exposed
  ```

- 생성한 서비스 조회

  ```{bash}
  $ kubectl get services
  ```

  ```{text}
  NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
  goapp-http   LoadBalancer   10.96.225.172   <pending>     8080:31585/TCP   104s
  kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          14d
  ```

- 서비스 테스트 (여러번 수행)

  ```{bash}
  curl http://10.96.225.172:8080
  ```

  [출력]

  ```{text}
  hostname: goapp-project-bcv5q
  hostname: goapp-project-bcv5q
  hostname: goapp-project-bcv5q
  hostname: goapp-project-bcv5q
  ```

###   scale out  수행 및 서비스 테스트

- replication controller 를 통한 scale-out 수행

  ```{bash}
  kubectl scale rc goapp-project --replicas=3
  ```

  [출력]

  ```{txt}
  replicationcontroller/goapp-project scaled
  ```

- Scale-Out 결과 확인

  ```{bash}
  kubectl get pod
  ```

  [출력]

  ```{txt}
  NAME                  READY   STATUS    RESTARTS   AGE
  goapp-project-bcv5q   1/1     Running   0          16m
  goapp-project-c8hml   1/1     Running   0          26s
  goapp-project-r7kx5   1/1     Running   0          26s
  ```

- 서비스 테스트 (여러번 수행)

  ```{bash}
  curl http://10.96.225.172:8080
  ```

  [출력]

  ```{txt}
  NAME                  READY   STATUS    RESTARTS   AGE
  goapp-project-bcv5q   1/1     Running   0          16m
  goapp-project-c8hml   1/1     Running   0          26s
  goapp-project-r7kx5   1/1     Running   0          26s
  ```

  POD 삭제

Replication Controller 를 통해 생성된 POD 는 개별 POD 가 삭제 되지 않습니다. Replication Controller 자체를 삭제 해야 합니다.

```{bash}
kubectl delete rc goapp-project
```

## PODS

###   POD 설정을 yaml 파일로 가져오기

```{bash}
kubectl get pod goapp-project-bcv5q -o yaml
kubectl get po goapp-project-bcv5q -o json
```

크게 **metadata, spec, status** 항목으로 나누어 집니다.

[출력 - yaml]

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-01-10T07:37:49Z"
  generateName: goapp-project-
  labels:
    run: goapp-project
  name: goapp-project-bcv5q
  namespace: default
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicationController
    name: goapp-project
    uid: e223237f-17e2-44f4-aaee-07e8ff4592b2
  resourceVersion: "3082141"
  selfLink: /api/v1/namespaces/default/pods/goapp-project-bcv5q
  uid: 72bebcbd-8e6e-4906-9f66-1c3821fa49ec
spec:
  containers:
  - image: dangtong/goapp
    imagePullPolicy: Always
    name: goapp-project
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-qz4fh
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: worker01.sas.com
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-qz4fh
    secret:
      defaultMode: 420
      secretName: default-token-qz4fh
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T07:37:49Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T07:37:55Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T07:37:55Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T07:37:49Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://de3418e51fea7f7a19e7b1d6ff5a4a6f386f337578838e6f1ec0589275ca6f48
    image: dangtong/goapp:latest
    imageID: docker-pullable://dangtong/goapp@sha256:e5872256539152aecd2a8fb1f079e132a6a8f247c7a2295f0946ce2005e36d05
    lastState: {}
    name: goapp-project
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-01-10T07:37:55Z"
  hostIP: 192.168.56.103
  phase: Running
  podIP: 10.40.0.2
  podIPs:
  - ip: 10.40.0.2
  qosClass: BestEffort
  startTime: "2020-01-10T07:37:49Z"
```

###   POD 생성을 위한 YAML 파일 만들기

아래와 같이 goapp.yaml 파일을 만듭니다.

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: goapp-pod
spec:
  containers:
  - image: dangtong/goapp
    name: goapp-container
    ports:
    - containerPort: 8080
      protocol: TCP
```

> ports 정보를 yaml 파일에 기록 하지 않으면 아래 명령어로 향후에 포트를 할당해도 됩니다.
>
> ```
> kubectl port-forward goapp-pod 8080:8080
> ```

###   YAML 파일을 이용한 POD 생성 및 확인

```{bash}
$ kubectl create -f goapp.yaml
```

[output]

```{txt}
 pod/goapp-pod created
```

```{bash}
$ kubectl get pod
```

[output]

```{txt}
NAME                  READY   STATUS    RESTARTS   AGE
goapp-pod             1/1     Running   0          12m
goapp-project-bcv5q   1/1     Running   0          41m
goapp-project-c8hml   1/1     Running   0          25m
goapp-project-r7kx5   1/1     Running   0          25m
```

###   POD 및 Container 로그 확인

- POD 로그 확인

```{bash}
kubectl logs goapp-pod
```

[output]

```{bash}
Starting GoApp Server......
```

- Container 로그 확인

```{bash}
kubectl logs goapp-pod -c goapp-container
```

[output]

```{bash}
Starting GoApp Server......
```

> 현재 1개 POD 내에 Container 가 1이기 때문에 출력 결과는 동일 합니다. POD 내의 Container 가 여러개 일 경우 모든 컨테이너의 표준 출력이 화면에 출력됩니다.

###   연습문제 

1. Yaml 파일로 nginx 1.18.0 버전기반의 이미지를 사용해 nginx-app 이라는 이름의 Pod 를 만드세요(port : 80)
2. curl 명령어르 사용해. Nginx 서비스에 접속
3. nginx Pod 의 정보를 yaml 파일로 출력 하세요
4. nginx-app Pod 를 삭제 하세요

## Lable

###  Lable 정보를 추가해서 POD 생성하기

- goapp-with-lable.yaml 이라 파일에 아래 내용을 추가 하여 작성 합니다.

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: goapp-pod2
  labels:
    env: prod
spec:
  containers:
  - image: dangtong/goapp
    name: goapp-container
    ports:
    - containerPort: 8080
      protocol: TCP
```

- yaml 파일을 이용해 pod 를 생성 합니다.

```{bash}
$ kubectl create -f ./goapp-with-lable.yaml
```

[output]

```{txt}
pod/goapp-pod2 created
```

- 생성된 POD를 조회 합니다.

```{bash}
kubectl get po --show-labels
```

[output]

```{txt}
NAME                  READY   STATUS    RESTARTS   AGE     LABELS
goapp-pod             1/1     Running   0          160m    <none>
goapp-pod2            1/1     Running   0          3m53s   env=prod
goapp-project-bcv5q   1/1     Running   0          9h      run=goapp-project
goapp-project-c8hml   1/1     Running   0          9h      run=goapp-project
goapp-project-r7kx5   1/1     Running   0          9h      run=goapp-project
```

- Lable 태그를 출력 화면에 컬럼을 분리해서 출력

```{bash}
kubectl get pod -L env
```

[output]

```{txt}
NAME                  READY   STATUS    RESTARTS   AGE     ENV
goapp-pod             1/1     Running   0          161m
goapp-pod2            1/1     Running   0          5m19s   prod
goapp-project-bcv5q   1/1     Running   0          9h
goapp-project-c8hml   1/1     Running   0          9h
goapp-project-r7kx5   1/1     Running   0          9h
```

- Lable을 이용한 필터링 조회

```{bash}
kubectl get pod -l env=prod
```

[output]

```{txt}
NAME         READY   STATUS    RESTARTS   AGE
goapp-pod2   1/1     Running   0          39h
```

- Label 추가 하기

```{bash}
kubectl label pod goapp-pod2 app="application" tier="frondEnd"
```

- Label 삭제 하기

```{bash}
kubectl label pod goapp-pod2 app- tier-
```

###   Label 셀렉터 사용

- AND 연산

```{bash}
kubectl get po -l 'app in (application), tier in (frontEnd)'
```

- OR 연산

```{bash}
kubectl get po -l 'app in (application,backEnd)'

kubectl get po -l 'app in (application,frontEnd)'
```

###  생성된 POD 로 부터 yaml 파일 얻기

```{bash}
kubectl get pod goapp-pod -o yaml
```

[output]

```{txt}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-01-10T08:07:04Z"
  name: goapp-pod
  namespace: default
  resourceVersion: "3086366"
  selfLink: /api/v1/namespaces/default/pods/goapp-pod
  uid: 18cf0ed0-be56-4b54-869c-4473117800b1
spec:
  containers:
  - image: dangtong/goapp
    imagePullPolicy: Always
    name: goapp-container
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-qz4fh
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: worker02.sas.com
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-qz4fh
    secret:
      defaultMode: 420
      secretName: default-token-qz4fh
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T08:07:04Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T08:07:09Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T08:07:09Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-01-10T08:07:04Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://d76af359c556c60d3ac1957d7498513f42ace14998c763456190274a3e4a1d5e
    image: dangtong/goapp:latest
    imageID: docker-pullable://dangtong/goapp@sha256:e5872256539152aecd2a8fb1f079e132a6a8f247c7a2295f0946ce2005e36d05
    lastState: {}
    name: goapp-container
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-01-10T08:07:08Z"
  hostIP: 10.0.2.5
  phase: Running
  podIP: 10.32.0.4
  podIPs:
  - ip: 10.32.0.4
  qosClass: BestEffort
  startTime: "2020-01-10T08:07:04Z"
```

###  Lable을 이용한 POD 스케줄링

- 노드목록 조회

```{bash}
kubectl get nodes
```

[output]

```{txt}
NAME               STATUS   ROLES    AGE   VERSION
master.sas.com     Ready    master   16d   v1.17.0
worker01.sas.com   Ready    <none>   16d   v1.17.0
worker02.sas.com   Ready    <none>   16d   v1.17.0
```

- 특정 노드에 레이블 부여

```{bash}
kubectl label node worker02.sas.com memsize=high
```

- 레이블 조회 필터 사용하여 조회

```{bash}
kubectl get nodes -l memsize=high
```

[output]

```{txt}
NAME               STATUS   ROLES    AGE   VERSION
worker02.sas.com   Ready    <none>   17d   v1.17.0
```

- 특정 노드에 신규 POD 스케줄링

  아래 내용과 같이 goapp-label-node.yaml 파을을 작성 합니다.

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: goapp-pod-memhigh
spec:
  nodeSelector:
    memsize: "high"
  containers:
  - image: dangtong/goapp
    name: goapp-container-memhigh
```

- YAML 파일을 이용한 POD 스케줄링

```{bash}
kubectl create -f ./goapp-lable-node.yaml
```

[output]

```{txt}
pod/goapp-pod-memhigh created
```

- 생성된 노드 조회

```{bash}
kubectl get pod -o wide
```

[output]

```{txt}
NAME                  READY   STATUS    RESTARTS   AGE    IP          NODE               NOMINATED NODE   READINESS GATESgoapp-pod-memhigh     1/1     Running   0          17s    10.32.0.5   worker02.sas.com   <none>           <none>
```

##  Annotation

###   POD 에 Annotation 추가하기

```{bash}
kubectl annotate pod goapp-pod-memhigh maker="dangtong" team="k8s-team"
```

[output]

```{txt}
pod/goapp-pod-memhigh annotated
```

###  Annotation 확인하기

- YAML 파일을 통해 확인하기

```{bash}
kubectl get po goapp-pod-memhigh -o yaml
```

[output]

```{txt}
kind: Podmetadata:  annotations:    maker: dangtong    team: k8s-team  creationTimestamp: "2020-01-12T15:25:05Z"  name: goapp-pod-memhigh  namespace: default  resourceVersion: "3562877"  selfLink: /api/v1/namespaces/default/pods/goapp-pod-memhigh  uid: a12c35d7-d0e6-4c01-b607-cccd267e39ecspec:  containers:
```

- DESCRIBE 를 통해 확인하기

```{bash}
kubectl describe pod goapp-pod-memhigh
```

[output]

```{txt}
Name:         goapp-pod-memhigh
Namespace:    default
Priority:     0
Node:         worker02.sas.com/10.0.2.5
Start Time:   Mon, 13 Jan 2020 00:25:05 +0900
Labels:       <none>
Annotations:  maker: dangtong
              team: k8s-team
Status:       Running
IP:           10.32.0.5
```

###  Annotation 삭제

```{bash}
kubectl annotate pod  goapp-pod-memhigh maker- team-
```

###  연습문제

- bitnami/apache 이미지로 Pod 를 만들고 tier=FronEnd, app=apache 라벨 정보를 포함하세요

- Pod 정보를 출력 할때 라벨을 함께 출력 하세요

- app=apache 라벨틀 가진 Pod 만 조회 하세요

- 만들어진 Pod에 env=dev 라는 라벨 정보를 추가 하세요

- created_by=kevin 이라는 Annotation을 추가 하세요

- apache Pod를 삭제 하세요

  

## Namespace

###  네임스페이스 조회

```{bash}
kubectl get namespace
```

> kubectl get ns 와 동일함

[output]

```{bash}
NAME              STATUS   AGE
default           Active   17d
kube-node-lease   Active   17d
kube-public       Active   17d
kube-system       Active   17d
```

###  특정 네임스페이스의 POD 조회

```{bash}
kubectl get pod --namespace kube-system
# kubectl get po -n kube-system
```

> kubectl get pod -n kube-system 과 동일함

[output]

```{txt}
coredns-6955765f44-glcdc                 1/1     Running   0          17d
coredns-6955765f44-h7fbb                 1/1     Running   0          17d
etcd-master.sas.com                      1/1     Running   1          17d
kube-apiserver-master.sas.com            1/1     Running   1          17d
kube-controller-manager-master.sas.com   1/1     Running   1          17d
kube-proxy-gm44f                         1/1     Running   1          17d
kube-proxy-ngqr6                         1/1     Running   0          17d
kube-proxy-wmq7d                         1/1     Running   0          17d
kube-scheduler-master.sas.com            1/1     Running   1          17d
weave-net-2pm2x                          2/2     Running   0          17d
weave-net-4wksv                          2/2     Running   0          17d
weave-net-7j7mn                          2/2     Running   0          17d
```

###  YAML 파일을 이용한 네임스페이스 생성

- YAML 파일 작성 : first-namespace.yaml 이름으로 파일 작성

```{bash}
apiVersion: v1kind: Namespacemetadata:  name: first-namespace
```

- YAML 파일을 이용한 네이스페이스 생성

```{bash}
kubectl create -f first-namespace.yaml
```

[output]

```{txt}
namespace/first-namespace created
```

- 생성된 네임스페이스 확인

```{bash}
kubectl get namespacekubectl get ns
```

[output]

```{txt}
NAME              STATUS   AGEdefault           Active   17dfirst-namespace   Active   5skube-node-lease   Active   17dkube-public       Active   17dkube-system       Active   17d
```

> kubectl create namespace first-namespace 와 동일 합니다.

###  특정 네임스페이스에 POD 생성

- first-namespace 에 goapp 생성

```{bash}
kubectl create -f goapp.yaml -n first-namespace
```

[output]

```{txt}
pod/goapp-pod created
```

- 생성된 POD 확인하기

```{bash}
kubectl get pod -n first-namespace
```

[output]

```{txt}
NAME        READY   STATUS    RESTARTS   AGEgoapp-pod   1/1     Running   0          12h
```

###8.5 POD 삭제

```{bash}
'kubectl' delete pod goapp-pod-memhigh
```

```{bash}
kubectl delete pod goapp-pod
```

```{bash}
kubectl delete pod goapp-pod -n first-namespace
```

> 현재 네임스페이스 에서 존재 하는 모든 리소스를 삭제하는 명령은 아래와 같습니다.
>
> kubectl delete all --all
>
> 현재 네임스페이스를 설정하고 조회 하는 명령은 아래와 같습니다.
>
> ```shell
> # 네임스페이스 설정kubectl config set-context --current --namespace=<insert-namespace-name-here># 확인kubectl config view --minify | grep namespace:
> ```

###  연습문제

1. 쿠버네티스 클러스터에 몇개의 네임스페이가 존재 하나요?

2. my-dev 라는 네임스페이를 생성하고 nginx Pod를 배포 하세요

##  kubectl 기본 사용법

###  단축형 키워드 사용하기

```{bash}
kubectl get po			# PODskubectl get svc			# Servicekubectl get rc			# Replication Controllerkubectl get deploy	# Deploymentkubectl get ns			# Namespacekubectl get no			# Nodekubectl get cm			# Configmapkubectl get pv			# PersistentVolumns
```

###  도움말 보기

```{bash}
kubectl -h
```

```{txt}
kubectl controls the Kubernetes cluster manager. Find more information at: https://kubernetes.io/docs/reference/kubectl/overview/Basic Commands (Beginner):  create         Create a resource from a file or from stdin.  expose         Take a replication controller, service, deployment or pod and expose it as a new Kubernetes Service  run            Run a particular image on the cluster  set            Set specific features on objectsBasic Commands (Intermediate):  explain        Documentation of resources  get            Display one or many resources  edit           Edit a resource on the server  delete         Delete resources by filenames, stdin, resources and names, or by resources and label selectorDeploy Commands:
```

```{bash}
kubectl get -h
```

```{txt}
Display one or many resources Prints a table of the most important information about the specified resources. You can filter the list using a labelselector and the --selector flag. If the desired resource type is namespaced you will only see results in your currentnamespace unless you pass --all-namespaces. Uninitialized objects are not shown unless --include-uninitialized is passed. By specifying the output as 'template' and providing a Go template as the value of the --template flag, you can filterthe attributes of the fetched resources.Use "kubectl api-resources" for a complete list of supported resources.Examples:  # List all pods in ps output format.  kubectl get pods  # List all pods in ps output format with more information (such as node name).  kubectl get pods -o wide
```

###  리소스 정의에 대한 도움말

```{bash}
kubectl explain pods
```

```{txt}
KIND:     PodVERSION:  v1DESCRIPTION:     Pod is a collection of containers that can run on a host. This resource is     created by clients and scheduled onto hosts.FIELDS:   apiVersion	<string>     APIVersion defines the versioned schema of this representation of an     object. Servers should convert recognized schemas to the latest internal     value, and may reject unrecognized values. More info:     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources   kind	<string>     Kind is a string value representing the REST resource this object     represents. Servers may infer this from the endpoint the client submits     requests to. Cannot be updated. In CamelCase. More info:     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds   metadata	<Object>     Standard object's metadata. More info:     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata   spec	<Object>     Specification of the desired behavior of the pod. More info:     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status   status	<Object>     Most recently observed status of the pod. This data may not be up to date.     Populated by the system. Read-only. More info:     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

###  리소스 감시하기

- Kube-system 네임스페이스에 있는 모든 pod에 대해 모니터링 합니다.

```{bash}
kubectl get pods --watch -n kube-system
```

```{txt}
root@master:~# k get pods --watch -n kube-systemNAME                                     READY   STATUS    RESTARTS   AGEcoredns-6955765f44-glcdc                 1/1     Running   0          19dcoredns-6955765f44-h7fbb                 1/1     Running   0          19detcd-master.sas.com                      1/1     Running   1          19dkube-apiserver-master.sas.com            1/1     Running   1          19dkube-controller-manager-master.sas.com   1/1     Running   1          19dkube-proxy-gm44f                         1/1     Running   1          19dkube-proxy-ngqr6                         1/1     Running   0          19dkube-proxy-wmq7d                         1/1     Running   0          19dkube-scheduler-master.sas.com            1/1     Running   1          19dweave-net-2pm2x                          2/2     Running   0          19dweave-net-4wksv                          2/2     Running   0          19dweave-net-7j7mn                          2/2     Running   0          19d...
```

###  리소스 비교하기

```{bash}
kubectl diff -f goapp.yaml
```

###  kubectx 및 kubens 사용하기

현재 컨텍스트 및 네임스페이스를 확인하고 전환 할때 손쉽게 사용 할수 있는 도구

####  kubectx 및 kubens 설치

```{bash}
git clone https://github.com/ahmetb/kubectx.git ~/.kubectx
COMPDIR=$(pkg-config --variable=completionsdir bash-completion)
ln -sf ~/.kubectx/completion/kubens.bash $COMPDIR/kubens
ln -sf ~/.kubectx/completion/kubectx.bash $COMPDIR/kubectx

cat << FOE >> ~/.bashrc
export PATH=~/.kubectx:\$PATH
FOE
```

####  kubectx 및 kubens 사용

- kubectx 사용

```{bash}
kubectx
kubectx <변경하고 싶은 컨텍스트 이름>
```

- kubens 사용

현재 사용중인 네임스페이스를 조회 합니다. 현재 사용중인 네이스페이스는 하이라이트 됩니다.

```{bash}
kubens
```

```{txt}
default
first-namespace
kube-node-lease
kube-public
kube-system
```

Kube-system 네이스페이스로 전환 해봅니다.

```{bash}
kubens kube-system
```

```{txt}
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "kube-system".
```

pod 조회 명령을 내리면 아래와 같이 kube-system 네임스페이스의 pod 들이 조회 됩니다.

```{bash}
kubectl get po
```

```{bash}
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-6955765f44-glcdc                 1/1     Running   0          34d
coredns-6955765f44-h7fbb                 1/1     Running   0          34d
etcd-master.sas.com                      1/1     Running   1          34d
kube-apiserver-master.sas.com            1/1     Running   1          34d
kube-controller-manager-master.sas.com   1/1     Running   1          34d
kube-proxy-gm44f                         1/1     Running   1          34d
kube-proxy-ngqr6                         1/1     Running   0          34d
kube-proxy-wmq7d                         1/1     Running   0          34d
kube-scheduler-master.sas.com            1/1     Running   1          34d
weave-net-2pm2x                          2/2     Running   0          34d
weave-net-4wksv                          2/2     Running   0          34d
weave-net-7j7mn                          2/2     Running   0          34d
```

###  kubernetes 컨텍스트 및 네임스페이스 표시하기

kube-ps1을 다운로드 하여 /usr/local/kube-ps1 설치 하고 .bashrc 파일에 아래와 같이 설정 합니다. [링크](https://github.com/jonmosco/kube-ps1)

```{bash}
source /usr/local/kube-ps1/kube-ps1.sh
PS1='[\u@\h \W $(kube_ps1)]\$ '
```

```{bash}
[root@master ~ (⎈ |kubernetes-admin@kubernetes:kube-public)]#
```

###  kubectl Context 

. kube 디렉토리의 config 에 virtualBox 에 

####  Context 조회

```{bash}
kubectl config get-contexts
```



####  Context 추가

로컬 VM 의 control plane 의 $HOME/.kube/config 파일에서 다음 3가지 항목을 복사해서 로컬 디렉토에 각각 파일을 만듭니다.

- K8s 클러스터의 config 파일 내 인증 정보를 로컬의 아래 위치의 파일로 각각 복사 합니다.

| 항목                       | 로컬 파일                           |
| -------------------------- | ----------------------------------- |
| certificate-authority-data | $HOME/.kube/kmaster/ca-origin       |
| client-certificate-data    | $HOME/.kube/kmaster/cli-cert-origin |
| client-key-data            | $HOME/.kube/kmaster/cli-key-origin  |

- 내용이 BASE64 로 인코딩 되어 있기 때문에 각각의 파일을 아래와 같이 디코딩 해줍니다.
- Mac 및 Linux 에서는 아래와 같이 수행 합니다.

```{bash}
cat $HOME/.kube/kmaster/ca-origin | base64 -D > $HOME/.kube/kmaster/ca
cat $HOME/.kube/kmaster/cli-cert-origin | base64 -D > $HOME/.kube/kmaster/cli-cert
cat $HOME/.kube/kmaster/cli-key-origin | base64 -D > $HOME/.kube/kmaster/cli-key
```

- Window 에서는 아래와 같이 수행 합니다.

```{powershell}
certutil -decode ca-origin ca
certutil -decode cli-cert-origin cli-cert
certutil -decode cli-key-origin cli-key

# type config | findstr name
```

이제부터는 로컬 머신에서 아래와 같이 3단계로 kubectl 명령어를 사용해 kubernetes 클러스터를 추가 해줍니다.

- 1단계 클러스터 추가 (set-cluster)

>**kubectl config --kubeconfig=**[config-file-name] **set-cluster** [cluster-name] **--server=**[api-server-url] **--certificate-authority=**[ca-certificate]

```{bash}
kubectl config --kubeconfig=config set-cluster local-k8s --server=https://192.168.56.111:6443 --certificate-authority=/Users/dangtongbyun/.kube/kmaster/ca
```

- 2단계 접속 정보 추가 (set-credentials)

> **kubectl config --kubeconfig=**[config-file-name] **set-credentials** [cluster-name] **--client-certificate=**[client-cert-file] **--client-key=**[client-key-file]

```{bash}
kubectl config --kubeconfig=config set-credentials kubernetes-admin --client-certificate=/Users/dangtongbyun/.kube/kmaster/cli-cert --client-key=/Users/dangtongbyun/.kube/kmaster/cli-key
```

- 3단계 Context 추가 (set-context)

>**kubectl config --kubeconfig=**[config-file-name] **set-context** [context-name] **--cluster=**[cluster-name] **--namespace=**[namepsace-name] **--user=**[username]

```{bash}
kubectl config --kubeconfig=config set-context kubernetes-admin@local-k8s --cluster=local-k8s  --user=kubernetes-admin
```



####  Context 변경

```{bash}
kubectl config get-contexts 
# get-contexts 결과에서 혹인후 아래 명령어로 현재 클러스터 변경
kubectl config use-context <Context-Name>
```

####  Context 삭제

```{bash}
kubectl config delete-cluster [cluster-name]
kubectl config delete-context [context-name]
kubectl config delete-user [user-name]

or 

kubectl config unset users.[cluster-name]
kubectl config unset contexts.[context-name]
kubectl config unset clusters.[context-name]
```

> google cloud 에서는 cluster-name, context-name, user-name.  이 모두 동일함



####  alias 만들기

- Linux 및 MAC

```{bash}
alias chks='kubectl config use-context'
alias lsks='kubectl config get-contexts'
```

- Windows

```{powershell}
Set-Alias chks 'kubectl config use-context'
Set-Alias lsks 'kubectl config get-contexts'
```



