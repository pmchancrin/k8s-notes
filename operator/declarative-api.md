# 命令式（Imperative ）配置文件操作 vs. Declarative API（声明式API）

> https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#understanding-kubernetes-objects

```
$ kubectl create -f nginx.yaml
$ kubectl replace -f nginx.yaml
```

通过先create再replace操作，叫做**命令式配置文件操作**。告诉机器先干什么，再干什么，机器按照一条一条命令去干。

```
$ kubectl apply -f nginx.yaml
```

通过apply进行创建和滚动更新，叫做**声明式API操作**。告诉机器去干什么，不告诉具体指令，机器自动识别完成任务。

replace的执行过程，是使用新的YAML文件中的API对象，**替换原有的API对象**。  kube-apiserver在响应命令式请求时，一次只能处理一个写请求，否则就会产生冲突的可能，一条一条按顺序执行响应。  

apply，是执行了一个**对原有API对象的PATCH操作**。类似还要set image和edit。 kube-apiserver在响应声明式请求时，一次能处理多个写操作，并且具备Merge的能力。

> A `declarative API` allows you to declare or specify the desired state of your resource and tries to match the actual state to this desired state. Here, the controller interprets the structured data as a record of the user’s desired state, and continually takes action to achieve and maintain this state.


# 声明式API 原理

> * https://kubernetes.io/docs/concepts/overview/kubernetes-api/ 
> * https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/  
> * https://github.com/resouer/k8s-controller-custom-resource  
> * https://github.com/trstringer/k8s-controller-custom-resource  
> * https://medium.com/@trstringer/create-kubernetes-controllers-for-core-and-custom-resources-62fc35ad64a3  


Kubernetes的所有API对象，如下面树状图：

![kubernetes api](/images/kubernetes-api.png)

一个API对象在Etcd里面完整的资源路径由：Group（API组）、verison（API版本）和Resource（API资源类型）三个部门组成。

举例：  
CronJob 就是API的Resource，batch是它的Group，v2alpha1就是它的Version。
```
apiVersion: batch/v2alpha1
kind: CronJob
...
```

Kubernetes对Resource，Group，Version解析过程：
* 先匹配API对象的Group。
* 再匹配到API对象的Version。
* 最后匹配API对象的Resource。

Kubernetes的Core API 对象比如Pod，Node等在Core Group下，其他对象在Named Group下：
* Core Group：REST path is `/api/v1` and use `apiVersion:v1`
* Named Group: REST path is `/apis/$GROUP_NAME/$VERSION` and use `apiVersion: $GROUP_NAME/$VERSION`


创建过程：

![kubernetes api process](/images/kubernetes-api-process.png)

* 发起创建CronJob的POST请求后，YAML信息被提交给APIServer，APIServer第一个功能就是过滤这个请求，完成一些前置性工作，授权，超时处理，审计等。  
* 请求进入MUX和Routes流程。完成URL和Handler绑定。  
* APIServer根据定义，创建一个CronJob对象。在这个过程中，APIServer会进行一个Convert工作，把YAML文件转换成Super Version的对象，这个对象是该API资源类型所有版本的字段全集，用户提交的不同版本的YAML文件，都可以用这个对象来处理。
* APIServer先后进行Admission和Validation操作。Validation负责验证这个对象各个字段是否合法，验证通过的API对象，保存在API的一个叫Registry的数据结构里。只要一个API对象在Registry里面能查到，就是一个有效的API对象。
* APIServer将API对象转换成用户最初提交的版本，进行序列化操作，调用ETCD的API保存。

Declarative API是Kubernetes项目的重中之重，是Google Borg设计思想的集中体现，也是Kubernetes项目中唯一一个被Google和RedHat公司双重控制，其他势力无法从参与的组件。

由于需要监控性能，API完备性，版本化，向后兼容等多种工程化指标，所使用APIServer项目中大量使用了Go语言的代码生成功能，比如自动化Convert，DeepCopy等与API资源相关的操作。

## Custom Resource

> * https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
> * https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/
> * https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#understanding-kubernetes-objects

### Custom Controller
**Custom resource** simply let you store and retrieve structured data. It is combined with a controller that they become a true `declarative API`.  

**Custom controller** is a controller that users can deploy and  update on a running cluster,independently of the cluster's own lifecycle.  

**Custom controller** can work any kind of resource, but they are especially effective  when combined with `custom resources`. The `Operator` pattern is one example of such combination.

### Adding custom resources

* **CRD(Custom Resource Definition)** is simple and can be created without and programming.
* **API Aggregation** requires programming,but allows more over API behaviors like how data is stored and convertion between API versions.

### CustomResourceDefinition
CRD API resource allows you to define custom resources.

**CRD并不是万能的，有很多场景不适用，还有性能瓶颈。比如：不支持protobuf，当API Object 数量 > 1K 或者单个对象 > 1KB，或者高频请求时，CRD的响应都会又问题，所以CRD不能也不应该当作数据库使用。**  

**像Kubernete，或者Etcd本身，最佳使用场景就是作为配置管理的依赖。如果业务需求不能用CRD进行建模的时候，比如需要等待API最终返回，或者需要坚持API的返回值，也是不能用CRD的。同时，当你需要完成的APIServer而不是之关系API对象的时候，需要使用API Aggregator。**

## Demo - Add new resource using CRD

> Demo 文件：https://github.com/resouer/k8s-controller-custom-resource

为Kubernetes添加一个叫Network的API资源类型。  作用是用户一旦创建一个Network对象，Kubernetes就应该使用这个对象定义的网络参数，调用真实的网络插件，为用户创建一个真正的网络，用户创建Pod就可以使用这个网络了。

Network对象的YAML文件：

```
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"

```

上面这个文件就是一个CR（Custom Resource），为了能让Kubernetes认识这个CR，需要让Kubernetes明白这个CR的定义是什么，就是CRD（Custom Resource Definition）

CRD的YAML：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # <plural>:<group>
  name: networks.samplecrd.k8s.io
spec:
  #  to use for REST API:/apis/<group>/<version>
  group: samplecrd.k8s.io
  version: v1
  names:
    # CamelCased singular. CR manifests use <kind>.
    kind: Network
    # to use for the URL:/apis/<group>/<verison>/<plural>
    plural: networks
  # Namespaced or Cluster
  scope: Namespaced
```

scope是Namespaced，定义这个Network是一个属于Namespace的对象，类似Pod。

创建CRD和CR

```
yzw@yzw-vm:~$ kubectl apply -f network.yaml 
customresourcedefinition.apiextensions.k8s.io/networks.samplecrd.k8s.io created

yzw@yzw-vm:~$ kubectl apply -f example-network.yaml 
network.samplecrd.k8s.io/example-network created

yzw@yzw-vm:~$ kubectl get network
NAME              AGE
example-network   13s

yzw@yzw-vm:~$ kubectl describe network example-network 
Name:         example-network
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"samplecrd.k8s.io/v1","kind":"Network","metadata":{"annotations":{},"name":"example-network","namespace":"default"},"spec":{...
API Version:  samplecrd.k8s.io/v1
Kind:         Network
Metadata:
  Creation Timestamp:  2019-01-04T09:01:10Z
  Generation:          1
  Resource Version:    768534
  Self Link:           /apis/samplecrd.k8s.io/v1/namespaces/default/networks/example-network
  UID:                 47d514b4-0fff-11e9-9e53-080027c2b927
Spec:
  Cidr:     192.168.0.0/16
  Gateway:  192.168.0.1
Events:     <none>
```

## Controller Pattern

Controller都放在 *pkg/controller* 目录下，都遵循Kubernetes项目中的一个通用编排模式：控制循环（control loop）

```
for {
  实际状态 := 获取集群中对象 X 的实际状态（Actual State）
  期望状态 := 获取集群中对象 X 的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

实际状态一般来自于Kubernetes集群本身。  
期望状态一般来自于用户提交的YAML文件。

Deployment 控制器的实现步骤：

1. Deployment 控制器从Etcd里面获取所有携带“app:nginx”标签的Pod，统计他们的数量，这是实际状态；
2. Deployment 对象的Replicas字段是期望状态；
3. Deployment 控制器讲两个状态进行对比，根据对比结果，确定删除还是创建Pod。 这种对比操作叫调谐（Reconcile），这哥调谐的过程，称作调谐循环（Reconcile Loop）或者同步循环（Sync Loop）。

![kubernetes deployment yaml](/images/kubernetes-deployment-yaml.png)

* 上半部分是控制器定义，包括期望状态；
* 下半部分是被控制对象的模板叫PodTemplete组成。

## Custom Controller

> https://github.com/kubernetes/sample-controller  
> https://github.com/resouer/k8s-controller-custom-resource
> https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/

创建一个GO项目：

```
$ tree ~/go/src/github.com/resouer/k8s-controller-custom-resource
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── register.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
```

`pkg/apis/samplecrd/`下创建`register.go` 用来放全局变量

```
package samplecrd

const (
	GroupName = "samplecrd.k8s.io"
	Version   = "v1"
)
```

`pkg/apis/samplecrd/v1` 下创建`doc.go` (Golang的文档源文件)

```
// +k8s:deepcopy-gen=package

// +groupName=samplecrd.k8s.io
package v1
```

* `+<tag_name>[=value]` 格式的注释，是K8s进行代码生成要用到的Annotation风格的注释。  
* ` +k8s:deepcopy-gen=package` 为整个v1包里面的所有类型定义自动生成DeepCopy方法。  
* `+groupName=samplecrd.k8s.io` 定义这个包对应的API组的名字。
* 这些注释又叫 **Global Tags** 。

`pkg/apis/samplecrd/v1` 下创建`types.go`,定义CDR类型有哪些字段，spec字段的内容。

```
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Network describes a Network resource
type Network struct {
	// TypeMeta is the metadata for the resource, like kind and apiversion
	metav1.TypeMeta `json:",inline"`
	// ObjectMeta contains the metadata for the particular object, including
	// things like...
	//  - name
	//  - namespace
	//  - self link
	//  - labels
	//  - ... etc ...
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec is the custom resource spec
	Spec NetworkSpec `json:"spec"`
}

// NetworkSpec is the spec for a Network resource
type NetworkSpec struct {
	// Cidr and Gateway are example custom spec fields
	//
	// this is where you would put your custom resource data
	Cidr    string `json:"cidr"`
	Gateway string `json:"gateway"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// NetworkList is a list of Network resources
type NetworkList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []Network `json:"items"`
}
```

* `TypeMeta`（API元数据）和`ObjectMeta`（对象元数据），标准K8s对象都有的字段； 
* `NetworkSpec` 自定义类型的字段；  
* `NetworkList` 类型用来描述一组Network对象，因为Kubernetes获取所有对象的List()方法，返回值是List类型，而不是对象的类型的数组。
* `+genclient` 为下面API资源类型生成对应的Client代码。
* `+genclient:noStatus` 这个API资源类型没有Status字段，否正生成的Client就会自动带上UpdateStatus字段。
* `+genclient` 只需要写在Network类型，这是主类型；不用写在NetworkList上，这是返回值类型。
* 不用加`+k8s:deepcopy-gen=`,Global Tags里面已经定义了。
* `+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object` 生成DeepCopy的时候，实现K8s提供的runtime.Object接口。否则会有编译错误，固定操作。

`pkg/apis/samplecrd/v1` 下创建`register.go`文件

```
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"

	"github.com/resouer/k8s-controller-custom-resource/pkg/apis/samplecrd"
)

// GroupVersion is the identifier for the API which includes
// the name of the group and the version of the API
var SchemeGroupVersion = schema.GroupVersion{
	Group:   samplecrd.GroupName,
	Version: samplecrd.Version,
}

// create a SchemeBuilder which uses functions to add types to
// the scheme
var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme   = SchemeBuilder.AddToScheme
)

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

// Kind takes an unqualified kind and returns back a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// addKnownTypes adds our types to the API scheme by registering
// Network and NetworkList
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(
		SchemeGroupVersion,
		&Network{},
		&NetworkList{},
	)

	// register the type in the scheme
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}

```

* 通过`addKnownTypes`让生成的client指定Network和NetworkList类型的定义。
* 这个文件属于模板写法。

通过`k8s.io/code-generator`工具自动生成代码：

```
# 代码生成的工作目录，也就是我们的项目路径
$ ROOT_PACKAGE="github.com/resouer/k8s-controller-custom-resource"
# API Group
$ CUSTOM_RESOURCE_NAME="samplecrd"
# API Version
$ CUSTOM_RESOURCE_VERSION="v1"

# 安装 k8s.io/code-generator
$ go get -u k8s.io/code-generator/...
$ cd $GOPATH/src/k8s.io/code-generator

# 执行代码自动生成，其中 pkg/client 是生成目标目录，pkg/apis 是类型定义目录
$ ./generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
enerating deepcopy funcs
Generating clientset for samplecrd:v1 at github.com/resouer/k8s-controller-custom-resource/pkg/client/clientset
Generating listers for samplecrd:v1 at github.com/resouer/k8s-controller-custom-resource/pkg/client/listers
Generating informers for samplecrd:v1 at github.com/resouer/k8s-controller-custom-resource/pkg/client/informers
```

`./generate-groups.sh`参数使用的是跟import一样的相对路径。

生成完成后，项目目录如下：

```
$ tree
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    ├── apis
    │   └── samplecrd
    │       ├── constants.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        ├── informers
        └── listers
```

* `zz_generated.deepcopy.go` 自动生成的DeepCopy代码文件。
* 生成client目录，K8s为Network类型生成的客户端库，后面写自定义控制器的时候会用到。

`main.go` 里面完成初始化和启动一共自定义controller。

![kubernetes custom controller](/images/kubernetes-custom-controller.png)

```
package main

var (
	masterURL  string
	kubeconfig string
)

func main() {
	flag.Parse()

	// set up signals so we handle the first shutdown signal gracefully
	stopCh := signals.SetupSignalHandler()

	cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
	if err != nil {
		glog.Fatalf("Error building kubeconfig: %s", err.Error())
	}

	kubeClient, err := kubernetes.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building kubernetes clientset: %s", err.Error())
	}

	networkClient, err := clientset.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building example clientset: %s", err.Error())
	}

	networkInformerFactory := informers.NewSharedInformerFactory(networkClient, time.Second*30)

	controller := NewController(kubeClient, networkClient,
		networkInformerFactory.Samplecrd().V1().Networks())

	go networkInformerFactory.Start(stopCh)

	if err = controller.Run(2, stopCh); err != nil {
		glog.Fatalf("Error running controller: %s", err.Error())
	}
}

func init() {
	flag.StringVar(&kubeconfig, "kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
	flag.StringVar(&masterURL, "master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
}
```

* Informer和API对象一一对应，我们自定义Network对象的Informer就是Network Informer。
* networkInformerFactory通过networkClient跟APIServer建立连接，负责维护这个连接的是Relflector包，通过ListAndWatch方法获取和监听Network对象实例化的变化。
* ListAndWatch一旦监控盗APIServer端有信的Network实例被创建，删除或者更新，Reflector就会收到“事件通知”。该事件及它对应的API对象 叫增量（Delta），放进FIFO中。
* Informer不断从FIFO中Pop增量，判断事件类型，创建或更新本地对象缓存（Store）。
* 如果事件是Added，Informer会通过Indexer把这个对象实例保存到本地缓存，并创建索引；如果是Deleted，则会从本地缓存删掉。
* Informer根据事件类型，触发事先注册好的ResourceEventHandler（AddFunc，DeleteFunc，UpdateFunc）。  
* main 里面创建2个client和Informer，完成初始化控制器。
* Informer除了监听盗事件会同步本地缓存外，还会经过resyncPeriod指定事件，维护缓存，使用最近一次LIST结果强制更新一次。在K8s中，这个缓存强制更新的操作叫**resync**。   

总之：**Informer有两个职责：1.维护本地缓存（store）；2.触发Handler。Informer就是一个带本地缓存和索引机制的，可以注册EventHandler的client。**；

```
func NewController(
  kubeclientset kubernetes.Interface,
  networkclientset clientset.Interface,
  networkInformer informers.NetworkInformer) *Controller {
  ...
  controller := &Controller{
    kubeclientset:    kubeclientset,
    networkclientset: networkclientset,
    networksLister:   networkInformer.Lister(),
    networksSynced:   networkInformer.Informer().HasSynced,
    workqueue:        workqueue.NewNamedRateLimitingQueue(...,  "Networks"),
    ...
  }
    networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: controller.enqueueNetwork,
    UpdateFunc: func(old, new interface{}) {
      oldNetwork := old.(*samplecrdv1.Network)
      newNetwork := new.(*samplecrdv1.Network)
      if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
        return
      }
      controller.enqueueNetwork(new)
    },
    DeleteFunc: controller.enqueueNetworkForDelete,
 return controller
}
```

* 创建了一个work queue，同步Informer和控制循环之间的数据。
* 为networkInformer注册3个Handler。
* 实际入队的不是API对象本身，是Key，`<namespace>/<name>`。
* 控制循环不停的从queue里面拿Key，开始执行真正的控制逻辑。

```
func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
 ...
  if ok := cache.WaitForCacheSync(stopCh, c.networksSynced); !ok {
    return fmt.Errorf("failed to wait for caches to sync")
  }
  
  ...
  for i := 0; i < threadiness; i++ {
    go wait.Until(c.runWorker, time.Second, stopCh)
  }
  
  ...
  return nil
}
```

* main 中调用controller.Run()启动控制循环。
* 等待Informer完成一次本地缓存数据同步后，启动多个无限循环的任务。

```
func (c *Controller) runWorker() {
  for c.processNextWorkItem() {
  }
}

func (c *Controller) processNextWorkItem() bool {
  obj, shutdown := c.workqueue.Get()
  
  ...
  
  err := func(obj interface{}) error {
    ...
    if err := c.syncHandler(key); err != nil {
     return fmt.Errorf("error syncing '%s': %s", key, err.Error())
    }
    
    c.workqueue.Forget(obj)
    ...
    return nil
  }(obj)
  
  ...
  
  return true
}

func (c *Controller) syncHandler(key string) error {

  namespace, name, err := cache.SplitMetaNamespaceKey(key)
  ...
  
  network, err := c.networksLister.Networks(namespace).Get(name)
  if err != nil {
    if errors.IsNotFound(err) {
      glog.Warningf("Network does not exist in local cache: %s/%s, will delete it from Neutron ...",
      namespace, name)
      
      glog.Warningf("Network: %s/%s does not exist in local cache, will delete it from Neutron ...",
    namespace, name)
    
     // FIX ME: call Neutron API to delete this network by name.
     //
     // neutron.Delete(namespace, name)
     
     return nil
  }
    ...
    
    return err
  }
  
  glog.Infof("[Neutron] Try to process network: %#v ...", network)
  
  // FIX ME: Do diff().
  //
  // actualNetwork, exists := neutron.Get(namespace, name)
  //
  // if !exists {
  //   neutron.Create(namespace, name)
  // } else if !reflect.DeepEqual(actualNetwork, network) {
  //   neutron.Update(namespace, name)
  // }
  
  return nil
}
```

* 从workqueue.Get一个Key；
* 如果返回IsNotFound，则标识这个Key是前面通过删除事件加入到队列的，虽然队列有这个Key，但是实例已经被删了。
* 控制器拿到的这个Key对应的Network对象，是APIServer里保存的“期望状态”
* “实际状态”得通过访问集群获取。
* 通过对比期望状态和实际状态的差异，完成一次协调的过程。

```
# Clone repo

$ go build -o samplecrd-controller .

$ ./samplecrd-controller -kubeconfig=$HOME/.kube/config -alsologtostderr=true
I0915 12:50:29.051349   27159 controller.go:84] Setting up event handlers
I0915 12:50:29.051615   27159 controller.go:113] Starting Network control loop
I0915 12:50:29.051630   27159 controller.go:116] Waiting for informer caches to sync
E0915 12:50:29.066745   27159 reflector.go:134] github.com/resouer/k8s-controller-custom-resource/pkg/client/informers/externalversions/factory.go:117: Failed to list *v1.Network: the server could not find the requested resource (get networks.samplecrd.k8s.io)
...
```

编译工程，执行控制器，创建,更新，删除Network,可以看到打印信息。


**整个流程不仅可以用在自定义API资源，而且可以用在K8s原生的默认API对象上完成复杂的编排功能，比如可以定义一个Deployment的Informer等**

# Demo - Istio

Istio项目时一个基于Kubernetes项目的微服务治理框架：

![kubernetes istio](/images/kubernetes-istio-demo.jpg)

Istio最根本的组件，时运行在每个应用Pod里面的Envoy容器。   

Envoy项目是一个高性能C++网络代理，这个代理服务以sidecar容
器的方式，运行在每个被治理的应用Pod中，共享Network Namespace，Envoy容器通过Pod的iptables，把整个Pod的进出流量接管下来。  

Istio的控制层（Control Plane）里的Pilot组件，通过调用Envoy容器的API，对Envoy代理进行配置，实现微服务治理。  

## Dynamic Admission Control / Initializer

当一个Pod或者任何一个API对象被提交给APIServer后，总由一些“初始化”性质的工作需要在它们被Kubernetes项目正式处理之前进行，比如自动加Lables。  

“初始化”操作的实现，借助一个叫Admission的功能，就是Kubernetes项目里一组被成为Admission Controller的代码，可以选择性被编译进APIServer中，在API对象创建之初后被立刻调用。  

Kubernetes提供了一种不用重新编译APIServer的“热插拔”式的Admisson机制，叫Dynamic Admission Control，又叫Initializer。

**举例：**

> demo 地址：https://github.com/resouer/kubernetes-initializer-tutorial

创建一个Pod，只有一个用户容器：

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

Istio项目要做的，就是在这个Pod YAML被提交给Kubernetes之后，在它对应的API对象里面自动加上Envoy容器的配置：

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
  - name: envoy
    image: lyft/envoy:845747b88f102c0fd262ab234308e9e22f693a1
    command: ["/usr/local/bin/envoy"]
    ...
```

被Istio处理后，这个Pod里面，多了一个叫Envoy的容器。  

Istio要做的就是编写一个用来为Pod“自动注入”Envoy 容器的Initializer。

先Istio将这个Envoy容器本身的定义，以ConfigMap的方式保存在Kubernetes中, data部分式一个Pod对象的一部分定义：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-initializer
data:
  config: |
    containers:
      - name: envoy
        image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
        command: ["/usr/local/bin/envoy"]
        args:
          - "--concurrency 4"
          - "--config-path /etc/envoy/envoy.json"
          - "--mode serve"
        ports:
          - containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        volumeMounts:
          - name: envoy-conf
            mountPath: /etc/envoy
    volumes:
      - name: envoy-conf
        configMap:
          name: envoy
```

Initializer要干的工作就是把Envoy相关字段，自动添加到用户提交的Pod的API对象里。  

用户提交的Pod里面本来就有Containers，volume字段，Kubernetes在处理这些的更新请求时，就必须使用类似git merge这样的操作，将两部分内容合并。

Initializer更新用户Pod对象的时候，必须使用PATCH API来完成。这种PATCH API是声明式API最主要的能力。  

接下来，Istio将一个编写好的Initializer，作为一个Pod部署在Kubernetes中：

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: envoy-initializer
  name: envoy-initializer
spec:
  containers:
    - name: envoy-initializer
      image: envoy-initializer:0.0.1
      imagePullPolicy: Always
```

Kubernetes的Controller的就是一个死循环，不断获取实际状态，与期望状态作对比。  

envoy-initializer使用的镜像是一个事先编译好的Custom Controller，主要功能是：  
不断获取到“实际状态”，就是用户新创建的Pod。它的“期望状态”，则是：这个Pod里被添加Envoy容器的定义。

```
for {
  // 获取新创建的 Pod
  pod := client.GetLatestPod()
  // Diff 一下，检查是否已经初始化过
  if !isInitialized(pod) {
    // 没有？那就来初始化一下
    doSomething(pod)
  }
}

func doSomething(pod) {
  cm := client.Get(ConfigMap, "envoy-initializer")

  newPod := Pod{}
  newPod.Spec.Containers = cm.Containers
  newPod.Spec.Volumes = cm.Volumes

  // 生成 patch 数据
  patchBytes := strategicpatch.CreateTwoWayMergePatch(pod, newPod)

  // 发起 PATCH 请求，修改这个 pod 对象
  client.Patch(pod.Name, patchBytes)
}
```

判断如果Pod里面没有添加过Envoy容器，则进行Initialize操作，修改Pod的API（doSomething）。  

Initializer：  
* 先从APIServer里拿到ConfigMap；  
* 把ConfigMap里面的containers和volumes字段，直接添加进一个空Pod对象里面；  
* 调用Kubernetes API库提供的合并两个Pod对象的方法CreateTwoWayMergePatch生成woWayMergePatch；  
* 使用这个patch数据，调用Client，发起一个PATCH请求。

这样就完成了对一个用户提交的Pod，自动加上Envoy容器相关字段。

Kubernetes还允许你通过配置，来指定要对什么样的资源进行Initialize操作：

```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: envoy-config
initializers:
  // 这个名字必须至少包括两个 "."
  - name: envoy.initializer.kubernetes.io
    rules:
      - apiGroups:
          - "" // 前面说过， "" 就是 core API Group 的意思
        apiVersions:
          - v1
        resources:
          - pods
```

这个配置对所有Pod进行Initialize操作，指定负责操作的Initializer叫envoy-initializer。  

一旦这个InitializerConfiguration被创建，Kubernetes就会把这个Initializer的名字，加到新Pod的Metadata上：

```
apiVersion: v1
kind: Pod
metadata:
  initializers:
    pending:
      - name: envoy.initializer.kubernetes.io
  name: myapp-pod
  labels:
    app: myapp
...
```

每一个新创的Pod，都会自动携带metadata.initializers.pending的Metadata信息。

这个Metadata，就是Initializer的控制器判断这个Pod有没有执行过所负责的初始化的操作的重要一句，就是isInitialized方法的含义。  

我们还可以在具体Pod的Annotation里添加一个字段，声明要使用这个Initializer：

```
apiVersion: v1
kind: Pod
metadata
  annotations:
    "initializer.kubernetes.io/envoy": "true"
    ...
```

就会使用到我们之前定义的envoy-initializer了。

**Istio项目的核心，就是由无数个运行Pod中的Envoy容器组成的服务代理网络，这正是Service Mesh的含有。**  

Kubernetes 声明式API具有对API对象进行在线的更新的能力：
* 声明式，指的是我职须提交一个定义好的API对象来声明，我所期望的状态是什么样子。
* 声明式API运行有多个API写端，以PATCH的方式对API对象进行修改，无需关心本原始YAML文件的内容。
* 有了上面两个能力，Kubernetes项目才可以基于对API对象的增删改查，在完全无需外界干预的情况下，实现对实际状态和期望状态的调谐（Reconcile）过程。  

**无论对sidecar容器的巧妙设计，还是对Initializer的合理利用，Istio项目的设计和实现，都依托于Kubernetes的声明式API和它所提供的各种编排能力**

# Note
* kubectl apply 是通过mvcc实现并发写的。
* https://www.envoyproxy.io/docs/envoy/latest/intro/comparison
* Initializer和Preset都能注入Pod配置，Preset是Initializer的子集，比较适合发布流程离处理比较简单的情况，Initializer是需要写代码的。
* Envoy相对Nginx，HAProxy的优势在于编程友好的API，方便容器化，配置方便。