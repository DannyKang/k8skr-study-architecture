# Chapter 11 Understanding Kubernetes internals

## 11.1 아키텍처 이해

## Component Architecture

![](img/arch.png)
 - 참조 : 
   - [kubernetes : From beginner to Advanced - ep1](https://www.slideshare.net/InhoKang2/kubernetes-from-beginner-to-advanced)  by inho kang
   - [kubernetes : From beginner to Advanced - ep2](https://www.slideshare.net/InhoKang2/k8s-beginner-2advancedep02201904221130post) by inho kang



### Control Plane Component
 - etcd 분산 스토리지
 - API 서버
 - Scheduler
 - Controller Manager

### Worker Node Component
 - Kubelet
 - kube-proxy
 - Container Runtime

 ### Add-on Component
  - 쿠버네티스 DNS 서버
  - 대시보드
  - 인그레스 컨트롤러
  - Heapster
  - Container Network Interface Plugin


### Checking the status of Control Plane components

```bash
$ kubectl get componentstatuses
NAME	 			STATUS	 MESSAGE			 ERROR
scheduler 			Healthy	 ok
controller-manager 		Healthy	 ok
etcd-0 				Healthy	 {"health": "true"}

```
#### How these components communicate
 - 쿠버네티스 시스템 컴포넌트는 오직 API서버와 통신한다. 서로 직접 통신하지 않는다. 
 - API 서버가 etcd와 통신할 수 있는 유일한 컴포넌트이다. 
 - API 서버와의 연결은 대부분 컴포넌트쪽에서 API서버에 연결을 요청하는 식이지만 kubectl을 이용한 `attach` 또는 `port-forward`등의 경우는 API서버가 먼저 Kubelet에 연결을 요청한다. 


#### Running Multiple instances of Individual component
 - Control Plane의 컴포넌트는 여러 서버에 분산되어 여러 인스턴스를 띄울수 있다. 
 - 그러나 Worker node의 컴포넌트는 같은 node에서 실행되어야 한다. 

#### How components are run
 - 각각의 컴포넌트는 직접 설치해서 실행할 수 있고, `pod`형태로 실행할 수 있다. (`어떤 버전부터인가와 self contained 원칙 자료 확인 필요 - 아마도 Design Principaㅣ`)

``` bash
$ kubectl get po -o custom-columns=POD:metadata.name,NODE:spec.nodeName
➥ --sort-by spec.nodeName -n kube-system
POD 								NODE
kube-controller-manager-master 		master
kube-dns-2334855451-37d9k 			master
etcd-master 						master
kube-apiserver-master 				master
kube-scheduler-master 				master
kube-flannel-ds-tgj9k 				node1
kube-proxy-ny3xm 					node1
kube-flannel-ds-0eek8 				node2
kube-proxy-sp362 					node2
kube-flannel-ds-r5yf4 				node3
kube-proxy-og9ac 					node3

```


[Kubernetes Design and Architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md)  - 여기 내용을 조금 차용해 볼까?

## Components Detail - from Master to Node
 - 참조 : [Kubernetes - Beyond a Black Box](https://www.slideshare.net/harryzhang735/kubernetes-beyond-a-black-box-part-1) by Hao Zhang
 - 책의 내용에 덧붙여 위의 장표를 활용해 설명한다.  

### 1. API 서버 
- 쿠버네티스 API를 노출하는 마스터 노드의 컴포넌트. 쿠버네티스 컨트롤 플레인에 대한 프론트엔드이다.(from  k8s.io)
- 외부 Client와의 통신은 REST와 내부와는 REST/protobuf를 기반으로 통신한다. 


#### k8s의 통신 원칙
 - 외부와의 통신은 HTTP 또는 WebSockets으로 통신하고 내부적으로만 gRPC를 이용한다는 것은 꽤나 좋은 사상이다. 이 부분에 대한 불만(gPRC를 Default로 하자는 제안을 k8s 제안(?)란에서 봤으나, maintainer입장은 명확했다. No!, 왜냐하면 gRPC가 없는? 안되는 Client도 많기 때문에…)
   > TL;DR: Using gRPC with Kubernetes, cluster-internally, is straight-forward. Exposing a gRPC service cluster-externally not so much. Maybe a good practice could be: use gRPC cluster-internally and offer public interfaces using HTTP and/or WebSockets?  
[Reviewing gRPC on Kubernetes](https://link.medium.com/ZRJkql0ixW) by Michael Hausenblas 

#### 1.1 REST API 
![](img/api-1.png)
 - 객체등을 저장하는 일관된 방법을 제공하는 것 외에도, 객체의 유효성 검사도 수행하므로 클라이언트가 잘못 구성된 객체를 저장할 수 없다. 
 - 또한 검증과 함께 `낙관적인 잠금`도 처리하므로 동시 업데이트 시 다른 클라이언트가 객체의 변경 사항을 재정의하지 않는다. 

#### API 서버 동작 (책 내용)
![](img/api-4.png)

##### AUTHENTICATING THE CLIENT WITH AUTHENTICATION PLUGINS
- 클라이언트에 대한 인증을 인증 플러그인을 사용해 인증을 수행한다. 
- 하나 이상의 인증 플러그인 구성 가능
- 인증 플러그인의 동작 방식에 따라서 8장에서 사용한 클라이언트 인증서나 HTTP헤더에서 user정보를 추출해서 인증을 수행하고 `Authorization`으로 넘긴다. 


##### AUTHORIZING THE CLIENT WITH AUTHORIZATION PLUGINS
 - 인증된 사용자가 요청한 액션을 해당 리소스에 대해서 수행할 수 있는지를 결정한다. 
 - 예를 들어 포드를 생성할 때 API 서버는 모든 권한 승인 플러그인에게 차례대로 물어보고 사용자가 요청한 네임스페이스에 포드를 생성할 수 있는지를 결정

##### VALIDATING AND/OR MODIFYING THE RESOURCE IN THE REQUEST WITH ADMISSION CONTROL PLUGINS
 - 인증과 권한체크가 끝난 요청을 object를 저장하기 전에 수행되는 일련을 코드들
 - 승인 제어 플러그인은 리소스 스펙에 누락된 필드를 디폴트 값이나 상속받은 값으로 초기화 수행하며 심지어 연관된 다른 리소스도 변경할 수 있다.
 - 다양한 이유로 요청을 거절할 수 있다. 

 ##### examples of admission control plugins
  - AlwaysPullImages : Overrides the pod’s `imagePullPolicy` to Always, forcing the image to be pulled every time the pod is deployed.
  - ServiceAccount : Applies the default `service account` to pods that don’t specify it explicitly.
  - NamespaceLifecycle : Prevents creation of pods in namespaces that are in the process of being deleted, as well as in non-existing namespaces.
  - ResourceQuota : Ensures pods in a certain namespace only use as much CPU and memory as has been allocatted to the namespace.


 >  참조  : Admission Control plugin
https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/  
 > `mutating and validate controller`

##### VALIDATING THE RESOURCE AND STORING IT PERSISTENTLY
 - 요청이 승인 제어 플러그인을 통과하면 API 서버의 오브젝트의 유효성 검사를 하고 etcd에 저장한 다음 클라이언트에 응답을 전달.


#### 1.2 REST API - Group
 ![](img/api-2.png)


 - **API Group** is a collection of `Kinds` that are logically related. For example, all batch objects like `Job` or `ScheduledJob` are in the `batch` API Group.
 - **Version**. Each API Group can exist in multiple versions. For example, a group first appears as `v1alpha1` and is then promoted to `v1beta1` and finally graduates to `v1`. An object created in one version (e.g. v1beta1) can be retrieved in each of the supported versions (for example as v1). The API server does lossless conversion to return objects in the requested version.
 - **Resource** is the representation of a system entity sent or retrieved as JSON via HTTP; can be exposed as an individual resource (such as `.../namespaces/default`) or collections of resources (like `.../jobs`). 
 [Kubernetes deep dive : API Server -part1](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-1/)

  > REST API의 endpoint는 기본적으로 `Kind`를 기준으로 grouping되어 있지만, 이전 버전과의 호환성을 지키지 위해 기존 버전도 존속한다. (ex: `/api/v1` vs `/apis/core/v1`) 

 ![](img/api-5.png)


#### 1.3 REST API - Object Definition
![](img/api-3.png)

 

#### 1.4 Persist Object 

 - k8s의 빠른 개발 주기로 인해 같은 기능의 다양한 버전의 api가 존재한다. 
안정성 다양한 버전이 존재 (ex: `/apis/batch/v1` and `/apis/batch/v2alpha1`)
 - k8s는 이를 API 서버의 `lossless conversion` 으로 처리 
![](img/api-etcd-1.png)


```bash
$ http localhost:8080/apis/extensions/v1beta1/namespaces/api-server-deepdive/horizontalpodautoscalers/rcex > hpa-v1beta1.json
$ http localhost:8080/apis/autoscaling/v1/namespaces/api-server-deepdive/horizontalpodautoscalers/rcex > hpa-v1.json
$ diff -u hpa-v1beta1.json hpa-v1.json
{
  "kind": "HorizontalPodAutoscaler",
-  "apiVersion": "extensions/v1beta1",
+  "apiVersion": "autoscaling/v1",
  "metadata": {
    "name": "rcex",
    "namespace": "api-server-deepdive",
-    "selfLink": "/apis/extensions/v1beta1/namespaces/api-server-deepdive/horizontalpodautoscalers/rcex",
+    "selfLink": "/apis/autoscaling/v1/namespaces/api-server-deepdive/horizontalpodautoscalers/rcex",
    "uid": "ad7efe42-50ed-11e7-9882-5254009543f6",
    "resourceVersion": "267762",
    "creationTimestamp": "2017-06-14T10:39:00Z"
  },
  "spec": {
-    "scaleRef": {
+    "scaleTargetRef": {
      "kind": "ReplicationController",
      "name": "rcex",
-      "apiVersion": "v1",
-      "subresource": "scale"
+      "apiVersion": "v1"
    },
    "minReplicas": 2,
    "maxReplicas": 5,
-    "cpuUtilization": {
-      "targetPercentage": 80
-    }
+    "targetCPUUtilizationPercentage": 80
```

 > When the API server receives an object, for example, from kubectl, it will know from the HTTP path which version to expect. It creates a matching empty object using the Scheme in the right version and converts the HTTP payload using a JSON or protobuf decoder. The decoder converts the binary payload into the created object.
 - k8s API 서버가 `kubectl`등의 Request로 Object를 받으면 HTTP path를 통해서 version을 파악하고 해당 Scheme(pkg/scheme)에 해당하는 빈 object를 만드리고 payload를 이용해서 object로 변환한다.  
 > The decoded object is now in one of the supported versions for the given type. For some types there are a handful of versions throughout its development. To avoid problems with that, the API server has to know how to convert between each pair of those versions (for example, `v1 ⇔ v1alpha1, v1 ⇔ v1beta1, v1beta1 ⇔ v1alpha1`), the API server uses one special “internal” version for each type. This internal version of a type is a kind of superset of all supported versions for the type with all their features. It converts the incoming object to this internal version first and then to the storage version:  [Kubernetes deep dive : API Server -part2](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-2/)
 - 주어진 타입의 지원하는 버전의 object로 변환되는데 어떤 타입같은 경우는 개발과정에서 너무 많은 버전이 있다. API서버는 이런 문제를 해결하기 위해서 `internel` version을 가지고 각각을 변환 한다. 


  ![](img/api-etcd-2.png)


### 2. ETCD
![api & etcd](img/etcd.png)

 - unix 계열에서 /etc는 각종 config Data를 가지고 있는 것 처럼 etcd는 각종 config data의 저장 장소라는 데에서 영감을 받아 지어진 이름이다. “d”는 Distribed이다. 
 - etcd2와 etcd3의 가장 큰 특징은 etcd2는 계층구조(Hierarchy)를 가지지만 etcd3는 flat구조를 가진다는 것(하위 호환성을 위해서 계층구조로 보여진다)과 etcd3는 gRPC 기반이라는 것이다.  

#### WHY THE NUMBER OF ETCD INSTANCES SHOULD BE AN ODD NUMBER

 - Raft consensus algorithm : 분산환경의 합의 알고리즘(상태에 대한 합의를 과반수(quorum)를 이용해 처리)
 - 과반수 처리를 위한 홀수개로 구성해야 하며 보통 대규모 클러스터에서는 5개나 7개로 구성


```bash
// in etcd pod
bash-4.2# ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get / --prefix --keys-only > keylog.txt

```
![](img/etcdctl.png)





### API Server Swagger UI로 접근과 & CORS
* Swagger UI로 k8s api swagger.json 접근
    * Swagger UI로 k8s api에 접근하려는데 CORS(Cross-origin Resource Sharing) 때문에 문제다 여러가지를 찾다가 k8s서버가 설치되어 있는 곳에 /etc/kubernetest/manifest/kube-apiserver.yaml 등에서 어떤 pod로 서버를 띄우는지를 발견할 수 있다.
        * K8s api server yaml file 을 CORS 를 허용하도록 변경 :https://stackoverflow.com/questions/38081819/enabling-cors-in-kubernetes-api-server-with-https
            * /etc/kubernetes/manifests/kube-apiserver.yaml 에 - --cors-allowed-origins=http://www.example.com,https://*.example.com 를 추가한다. 
            * 이런 형식도 가능하다고 하는데 모르겠다. --cors-allowed-origins=["http://*”] 
            * 현재는 아래 처럼 http://localhost 와 http://*를 추가해 놓았는데 뭐가 먹는지 모르겠다. 
                * 실험결과 http://localhost만으로도 동작한다. ["http://*”] 는 안먹고 http://*만 썼는때 동작한다. 
            * 그리고 yaml 파일을 변경하면 Pod를 재시작한다. 

![](img/swagger.png)

 - Swagger UI
```bash
$docker run —rm -p 80:8080 swaggerapi/swagger-ui

// Swagger UI 에서 http://localhost:8001/openapi/v2
```

#### Debugging 방법
 - Debugging 방법 


### 3. Controller Manager
 - 컨트롤러를 구동하는 마스터 상의 컴포넌트.
 - 논리적으로, 각 컨트롤러는 개별 프로세스이지만, 복잡성을 낮추기 위해 모두 단일 바이너리로 컴파일되고 단일 프로세스 내에서 실행된다.
 - API 서버를 통해서 `Shared State(ETCD)`의 정보를 `Control loop`를 통해 `감시(watch)`하고 있다가 `Current State`를 `Desired State`로 변경을 시도한다. 

#### Controller Pattern
>In applications of robotics and automation, a control loop is a non-terminating loop that regulates the state of the system. In Kubernetes, a controller is a control loop that watches the shared state of the cluster through the API server and makes changes attempting to move the current state towards the desired state. Examples of controllers that ship with Kubernetes today are the replication controller, endpoints controller, namespace controller, and serviceaccounts controller.  
>
> Kubernetes official documentation, [Kube-controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager/)

 - Control loop
```go
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}
```

#### 3.1 Controller Manager 구조
* 심화 : [A Deep Dive into Kubernetes Controller](https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html)
    *  각각의 Contoller가(약 28개) API서버로 부터 상태 정보를 받아 오기 위해 매번 Request 정보를 요청하는 구조는 Overhead가 심하기 때문에 "cache"등을 구현한 모종의 framework이 제공 된다. 
		* Reflector, SharedInformer

![](img/ctrl-manager-1.png)

#### 3.2 ShareInfomer

![](img/controller-manager.png)

* SharedInformer  
      * 하나의 리소스를 여러개의 Controller가 바라보고 있기 때문에 informer를 공유해서 사용하는 하나의 Cache이다. 
        * In this case, the SharedInformer helps to create a single shared cache among controllers. This means cached resources won't be duplicated and by doing that, the memory overhead of the system is reduced



#### 3.2 Shared Resource Informer Factory

![](img/ctrl-manager-2.png)

#### 추가적으로 다뤄야하는 내용
 -  `resourceVersion` 을 이용한 resync 방안

#### 참조  
 - [DeltaFIFO Store](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores)
 - [Writing Controllers](https://github.com/kubernetes/community/blob/8decfe4/contributors/devel/controllers.md#rough-structure)
 - [A Deep Dive into Kubernetes Controller](https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html)
 - [stay informed k8s](https://www.firehydrant.io/blog/stay-informed-with-kubernetes-informers/)

<details>
<summary>Controller Rough Structure</summary>

[Controller Rough Structure](https://github.com/kubernetes/community/blob/8decfe4/contributors/devel/controllers.md#rough-structure)

```go
type Controller struct{
	// podLister is secondary cache of pods which is used for object lookups
	podLister cache.StoreToPodLister

	// queue is where incoming work is placed to de-dup and to allow "easy"
	// rate limited requeues on errors
	queue workqueue.RateLimitingInterface
}

func (c *Controller) Run(threadiness int, stopCh chan struct{}){
	// don't let panics crash the process
	defer utilruntime.HandleCrash()
	// make sure the work queue is shutdown which will trigger workers to end
	defer c.queue.ShutDown()

	glog.Infof("Starting <NAME> controller")

	// wait for your secondary caches to fill before starting your work
	if !framework.WaitForCacheSync(stopCh, c.podStoreSynced) {
		return
	}

	// start up your worker threads based on threadiness.  Some controllers
	// have multiple kinds of workers
	for i := 0; i < threadiness; i++ {
		// runWorker will loop until "something bad" happens.  The .Until will
		// then rekick the worker after one second
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	// wait until we're told to stop
	<-stopCh
	glog.Infof("Shutting down <NAME> controller")
}

func (c *Controller) runWorker() {
	// hot loop until we're told to stop.  processNextWorkItem will
	// automatically wait until there's work available, so we don't worry
	// about secondary waits
	for c.processNextWorkItem() {
	}
}

// processNextWorkItem deals with one key off the queue.  It returns false
// when it's time to quit.
func (c *Controller) processNextWorkItem() bool {
	// pull the next work item from queue.  It should be a key we use to lookup
	// something in a cache
	key, quit := c.queue.Get()
	if quit {
		return false
	}
	// you always have to indicate to the queue that you've completed a piece of
	// work
	defer c.queue.Done(key)

	// do your work on the key.  This method will contains your "do stuff" logic
	err := c.syncHandler(key.(string))
	if err == nil {
		// if you had no error, tell the queue to stop tracking history for your
		// key. This will reset things like failure counts for per-item rate
		// limiting
		c.queue.Forget(key)
		return true
	}

	// there was a failure so be sure to report it.  This method allows for
	// pluggable error handling which can be used for things like
	// cluster-monitoring
	utilruntime.HandleError(fmt.Errorf("%v failed with : %v", key, err))

	// since we failed, we should requeue the item to work on later.  This
	// method will add a backoff to avoid hotlooping on particular items
	// (they're probably still not going to work right away) and overall
	// controller protection (everything I've done is broken, this controller
	// needs to calm down or it can starve other useful work) cases.
	c.queue.AddRateLimited(key)

	return true
}
```
</details>


### 3. Watch를 이용한 이벤트 모델 이해
[Efficient detection of changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)
 1. List all of the pods in a given namespace.
 ```
 GET /api/v1/namespaces/test/pods
---
200 OK
Content-Type: application/json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {"resourceVersion":"10245"},
  "items": [...]
}
```

 2. Starting from resource version 10245, receive notifications of any creates, deletes, or updates as individual JSON objects.

```
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
---
200 OK
Transfer-Encoding: chunked
Content-Type: application/json
{
  "type": "ADDED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
}
{
  "type": "MODIFIED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
}
...
```
 - API의 Object 변경사항 Watch
    * 처음에 든 생각은 watch같은 것은 Client-go를 이용해서 내부적으로는 gRPC를 쓸거라 기대했지만, 소스코드를 보고 몇가지 실험을 한 결론은 URL + ?watch=true 또는 /watch/ 를 이용해서 http 1.1.의 chucked형태로 받는다는 것이다.  
        * 예를 들면 
            * $curl -X GET -i http://127.0.0.1:8001/api/v1/pods/?watch=true
            * $curl -X GET -i http://127.0.0.1:8001/api/v1/watch/pods
 - Client-go을 이용해서 Controller를 만들때는 Controller 공통으로 사용하는 SharedInformer에서 cache에 새로 만들 Controller에 EventHandler를 추가하는 구조로 되어 있기 때문에 직접 watch를 할 필요가 없다.
 - 참고 : kubectl log나 exec, port-forward등은 kubelet과 통신하여 WebSocket으로 통신한다.(https://stackoverflow.com/questions/52890977/kubernetes-api-server-serving-pod-logs) 

#### Watch 실험

 > 실험? kubectl proxy 하고 curl 로 watch를 본다??

#### Debugging API Server

> **Activating Additional Logs**  
> Kubernetes uses the github.com/golang/glog leveled logging package for its logging. Using the --v flag on the API server you can adjust the level of logging verbosity. In general, the Kubernetes project has set log verbosity level 2 (--v=2) as a sane default for logging relevant, but not too spammy messages. If you are looking into specific problems, you can raise the logging level to see more (possibly spammy) messages. Because of the performance impact of excessive logging, we recommend not running with a verbose log level in production. If you are looking for more targeted logging, the --vmodule flag enables increasing the log level for individual source files. This can be useful for very targeted verbose logging restricted to a small set of files. [Managing k8s](https://www.oreilly.com/library/view/managing-kubernetes/9781492033905/ch04.html)

> **Debugging kubectl Requests**
> In addition to debugging the API server via logs, it is also possible to debug interactions with the API server, via the kubectl command-line tool. Like the API server, the kubectl command-line tool logs via the github.com/golang/glog package and supports the --v verbosity flag. Setting the verbosity to level 10 (--v=10) turns on maximally verbose logging. In this mode, kubectl logs all of the requests that it makes to the server, as well as attempts to print curl commands that you can use to replicate these requests. Note that these curl commands are sometimes incomplete.

 - API Server가 사작할때 --v flag를 이용해 로그레벨을 올리는 방법으로 API 서버의 동작을 확인할 수 있다.  
		 api pod의 시작 yaml (/etc/kubernetes/manifest/tkube-apiserver.yaml)에 “--v=3”으로 바꿔서 로그 레벨을 올리는 방법과 kubectl 에 -v (10이 최고) 를 올려서 이용하는 방법이 있다.
        * "$kubectl logs -f <pod-name> -v 9 "
        * #curl -k -v -X GET -H "Accept: application/json" 'http://127.0.0.1:8001/api/v1/namespaces/kube-system/pods/kube-apiserver-k8s-master-ol-setup/log?follow=true'
       
    * 실험을 해본 결과 
        * Master 노드에서 docker를 이용해서 직접 log에 접근한 내용과 kubectl(verbose level 3, 8) 또는 curl 을 이용해서 log을 tail 한 내용이 모두 같았다.
        * 따라서 kubectl -v로 주는 pod 레벨의 verbose를 높여도 서버가 생산해 내는 로그 이상을 내뱉지는 못한다. (당연한 얘기다.)
            * $docker logs —follow <containerId>
            * $kubectl logs -f <pod-name> -v 8     vs. -v 3
        * 유일하게 생산하는 log를 증가하는 방법은 api pod의 시작 yaml을 변경하는 방법 뿐이다. 
        * 그런데 docker inspect <container> | grep LogPath 로 찾은 /var/lib/docker/containers/<container id>/<container id>-json.log 는 뭐지? 이 로그에는 Event만 계속 쌓이던데…. 마치 watch event 마냥
        * —> docker logs 는 ’STDOUT’ 과 ’STDERR’을 보여준다. 
            * 위의 /var/lib/xxxxxx.log는 (non-interactive app) api-server가 output을 logfile로 쌓는 내용이 포함되는 것이고, kubectl logs 는 docker logs의 기능을 활용해서 ’STDOUT’과 ’STDERR’를 보여주는 것이기 때문에 다를 수 밖에 없다. : https://docs.docker.com/v17.09/engine/admin/logging/view_container_logs/
            * 예를 들어 niginx는 심볼릭 링크를 통해 /dev/stdout 를 /var/log/nginx/access.log 로 /dev/stderr /var/log/nginx/error.log 로 보낸다. http는 /proc/self/fd/d1(STDOUT) /proc/self/fd2 (STDERR) 

### 4. Scheduler

![](img/scheduler.png)


### 5. Deployment 발생시 Sequence Diagram을 이용한 컴포넌트간의 연동관계 이해
 ![](img/sequence.png)



> 참조 : Kubernetes Reaction Chain : Animation from pdf 



## from place Deployment to create pod 
- 표준 Nginx Deployment to create pod 

<details>
<summary>from deployment to creat replicatset</summary>

```go
// In pkg/deployment/depolyment_controller.go
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
    eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: client.CoreV1().Events("")})
.........    
    dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
            AddFunc:    dc.addDeployment,
            UpdateFunc: dc.updateDeployment,
            // This will enter the sync loop and no-op, because the deployment has been deleted from the store.
            DeleteFunc: dc.deleteDeployment,
        })

// in pkg/deployment/sync.go

// Returns a replica set that matches the intent of the given deployment. Returns nil if the new replica set doesn't exist yet.
// 1. Get existing new RS (the RS that the given deployment targets, whose pod template is the same as deployment's).
// 2. If there's existing new RS, update its revision number if it's smaller than (maxOldRevision + 1), where maxOldRevision is the max revision number among all old RSes.
// 3. If there's no existing new RS and createIfNotExisted is true, create one with appropriate revision number (maxOldRevision + 1) and replicas.
// Note that the pod-template-hash will be added to adopted RSes and pods.
func (dc *DeploymentController) getNewReplicaSet(d *apps.Deployment, rsList, oldRSs []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, error) {

    // new ReplicaSet does not exist, create one.
	newRSTemplate := *d.Spec.Template.DeepCopy()
	podTemplateSpecHash := controller.ComputeHash(&newRSTemplate, d.Status.CollisionCount)
	newRSTemplate.Labels = labelsutil.CloneAndAddLabel(d.Spec.Template.Labels, apps.DefaultDeploymentUniqueLabelKey, podTemplateSpecHash)
	// Add podTemplateHash label to selector.
	newRSSelector := labelsutil.CloneSelectorAndAddLabel(d.Spec.Selector, apps.DefaultDeploymentUniqueLabelKey, podTemplateSpecHash)

	// Create new ReplicaSet
	newRS := apps.ReplicaSet{
		ObjectMeta: metav1.ObjectMeta{
			// Make the name deterministic, to ensure idempotence
			Name:            d.Name + "-" + podTemplateSpecHash,
			Namespace:       d.Namespace,
			OwnerReferences: []metav1.OwnerReference{*metav1.NewControllerRef(d, controllerKind)},
			Labels:          newRSTemplate.Labels,
		},
		Spec: apps.ReplicaSetSpec{
			Replicas:        new(int32),
			MinReadySeconds: d.Spec.MinReadySeconds,
			Selector:        newRSSelector,
			Template:        newRSTemplate,
		},
	}
	allRSs := append(oldRSs, &newRS)
	newReplicasCount, err := deploymentutil.NewRSNewReplicas(d, allRSs, &newRS)
	if err != nil {
		return nil, err
	}

	*(newRS.Spec.Replicas) = newReplicasCount
	// Set new replica set's annotation
	deploymentutil.SetNewReplicaSetAnnotations(d, &newRS, newRevision, false)
	// Create the new ReplicaSet. If it already exists, then we need to check for possible
	// hash collisions. If there is any other error, we need to report it in the status of
	// the Deployment.
	alreadyExists := false
	createdRS, err := dc.client.AppsV1().ReplicaSets(d.Namespace).Create(&newRS)

}
```
 - 결국 sync에 새로운 replicaset을 생성하는 소스가 있었다. 

</details>



### 6. kubelet 

![](img/kubelet.png)


### 7. Kube Proxy & Service

   - [kubernetes : From beginner to Advanced - ep2](https://www.slideshare.net/InhoKang2/k8s-beginner-2advancedep02201904221130post) by inho kang

IPTable로는 큰 서비스 불가 : https://www.slideshare.net/LCChina/scale-kubernetes-to-support-50000-services

![](img/iptable-limit.png)



### 8. Networking


