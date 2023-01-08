---
title: kubebuilder初体验
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["k8s"]
tags: ["k8s"]
---

# kubebuilder初体验
#k8s

最近在学习k8s operator扩展的开发，查阅资料发现kubebuilder是当前比较成熟并且方便的方式来进行自定义资源类型的开发，因此查询资料、跟着文档走了一遍，对此进行一个简要的记录

## 为什么需要开发Operator
k8s内置了很多常见的api资源类型，例如pod、deployment、job、cronjob等。但有可能有时候这些资源不能完全满足我们的需求，我们需要自定义一些资源类型，满足我们自身的业务逻辑，此时就可以使用到k8s operator的功能。

## 概念
- CRD
定制资源的API定义
- CR
*资源（Resource）* 是  [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/)  中的一个端点， 其中存储的是某个类别的  [API 对象](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/)  的一个集合。 例如内置的 *pods* 资源包含一组 Pod 对象。
*定制资源（Custom Resource）* 是对 Kubernetes API 的扩展，不一定在默认的 Kubernetes 安装中就可用。定制资源所代表的是对特定 Kubernetes 安装的一种定制
- CC
就定制资源本身而言，它只能用来存取结构化的数据。 当你将定制资源与 *定制控制器（Custom Controller）* 相结合时，定制资源就能够 提供真正的 *声明式 API（Declarative API）*
你可以在一个运行中的集群上部署和更新定制控制器，这类操作与集群的生命周期无关。 定制控制器可以用于任何类别的资源，不过它们与定制资源结合起来时最为有效。  [Operator 模式](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/) 就是将定制资源 与定制控制器相结合的

## kubebuilder
kubebuilder是一个快速构建定制资源的工具。它可以为用户生成一个定制资源的脚手架，用户不用关心太多和k8s集群交互、权限管理的复杂逻辑，而只需要关注如何定义自己的资源与定制控制器的处理逻辑即可。

## 实战
完成一个简单的demo，创建一个定制资源，这个资源管理k8s的job资源，每5秒根据yaml定义的job模版创建一个job，并且创建的job都与该定制资源进行绑定

### 初始化项目
```sh
mkdir mycronjob
cd mycronjob
# we'll use a domain of tutorial.kubebuilder.io,
# so all API groups will be <group>.tutorial.kubebuilder.io.
kubebuilder init --domain wzj.kubebuilder.io --repo wzj.kubebuilder.io/project
```

kubebuilder工具帮助我们初始化好了一个脚手架，里面的内容如下：
```
➜  mycronjob tree
.
├── Dockerfile
├── Makefile
├── PROJECT
├── bin
│   ├── controller-gen
│   └── kustomize
├── config
│   ├── crd
│   │   ├── bases
│   │   │   └── batch.wzj.kubebuilder.io_mycronjobs.yaml
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_mycronjobs.yaml
│   │       └── webhook_in_mycronjobs.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager
│   │   ├── controller_manager_config.yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── mycronjob_editor_role.yaml
│   │   ├── mycronjob_viewer_role.yaml
│   │   ├── role.yaml
│   │   ├── role_binding.yaml
│   │   └── service_account.yaml
│   └── samples
│       └── batch_v1_mycronjob.yaml
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go

14 directories, 40 files
```
可以看到，里面有一个main.go，这是我们之后要完成的定制控制器的函数入口。
里面还有很多yaml文件，这些yaml文件主要是控制器在集群中运行所需用到的。

### 创建API和Controller
```sh
kubebuilder create api --group batch --version v1 --kind MyCronJob
```

#### API 
`api/v1/`目录被创建，它还为我们的MyCronJobKind添加了一个文件， api/v1/mycronjob_types.go。
```
➜  mycronjob tree api
api
└── v1
    ├── groupversion_info.go
    ├── mycronjob_types.go
    └── zz_generated.deepcopy.go

1 directory, 3 files
```

我们目前暂时只需要关心mycronjob_types.go文件即可
```go
// MyCronJobSpec defines the desired state of MyCronJob
type MyCronJobSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	Foo string `json:"foo"`
}

// MyCronJobStatus defines the observed state of MyCronJob
type MyCronJobStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// MyCronJob is the Schema for the mycronjobs API
type MyCronJob struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   MyCronJobSpec   `json:"spec,omitempty"`
	Status MyCronJobStatus `json:"status,omitempty"`
}
```
可以看到，生成的代码有我们需要的API类型MyCronJob。但是目前的字段都只是一些基础的字段，需要我们根据自己的需求来对其进行填充。

MyCronJobSpec是该API需要的字段，MyCronJobStatus描述该API对象的状态

MyCronJob代表该API对象的实体

此时我们根据自己的需求填充即可，因为我们需要该API对象管理Job，因此该对象需要支持在其声明中定义Job模版。
```go
// MyCronJobSpec defines the desired state of MyCronJob
type MyCronJobSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Specifies the job that will be created when executing a CronJob.
	JobTemplate v1.JobTemplateSpec `json:"jobTemplate"`
}

```

而对于MyCronJobStatus而言，目前我们暂时用不到。

#### Controller
controller的代码生成在
```sh
➜  mycronjob tree controllers
controllers
├── mycronjob_controller.go
└── suite_test.go

0 directories, 2 files
```

此时也只有一个框架
```go
func (r *MyCronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Complete(r)
}
```
我们的逻辑主要在Reconcile中完成，SetupWithManager的作用之后学习补充，本次实验暂时用不上。

我们可以开始写我们的控制器逻辑了。
需求非常简单：我们自定义的mycronjob，提交给集群后，它可以每5秒根据自己的模版创建一个job

```go
func (r *MyCronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	var cronJob batchv1.MyCronJob
	log.Info("my cron job namespacedName", "req.NamespacedName", req.NamespacedName)
	if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
		log.Error(err, "unable to fetch CronJob")
		// we'll ignore not-found errors, since they can't be fixed by an immediate
		// requeue (we'll need to wait for a new notification), and we can get them
		// on deleted requests.
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
	 constructJobForCronJob := func(cronJob *batchv1.MyCronJob, scheduledTime time.Time) (*v1.Job, error) {
		// We want job names for a given nominal start time to have a deterministic name to avoid the same job being created twice
		name := fmt.Sprintf("%s-%d", cronJob.Name, scheduledTime.Unix())

		job := &v1.Job{
			ObjectMeta: metav1.ObjectMeta{
				Labels:      make(map[string]string),
				Annotations: make(map[string]string),
				Name:        name,
				Namespace:   cronJob.Namespace,
			},
			Spec: *cronJob.Spec.JobTemplate.Spec.DeepCopy(),
		}
		for k, v := range cronJob.Spec.JobTemplate.Annotations {
			job.Annotations[k] = v
		}
		//job.Annotations[scheduledTimeAnnotation] = scheduledTime.Format(time.RFC3339)
		for k, v := range cronJob.Spec.JobTemplate.Labels {
			job.Labels[k] = v
		}
		if err := ctrl.SetControllerReference(cronJob, job, r.Scheme); err != nil {
			log.Info("create job", "success", false)
			return nil, err
		}

		if err := r.Create(ctx, job); err != nil {
			log.Error(err, "unable to create Job for CronJob", "job", job)
			return nil, err
		}
		log.Info("create job", "success", true)
		return job, nil
	}
	constructJobForCronJob(&cronJob, time.Now())

	return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
}

```

- r.Get方法，能获取到mycronjob API的一个mycronjob实体
- constructJobForCronJob闭包根据当前mycronjob的job模版创建一个新的job，并且将其与当前mycronjob绑定，由mycronjob来管理其生命周期，最后提交给集群创建
- 最后返回给集群的信息是，下一次Reconcile请求将会是5秒后

### 安装controller
```sh
make install
```

### 启动controller
```sh
make run
```

### 创建一个mycronjob API对象
```yaml
apiVersion: batch.wzj.kubebuilder.io/v1
kind: MyCronJob
metadata:
  name: mycronjob-sample
spec:
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              args:
                - /bin/sh
                - -c
                - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
该文件也是由kubebuilder自动生成的，但是crd新增的字段需要我们自己补充，例如本例的jobTemplate

```sh
➜  mycronjob kubectl apply -f config/samples/batch_v1_mycronjob.yaml
```

观察可以发现，每5秒生成了一个job
```sh
➜  mycronjob kubectl get job
NAME                          COMPLETIONS   DURATION   AGE
mycronjob-sample-1641736135   1/1           5s         16s
mycronjob-sample-1641736140   1/1           4s         11s
mycronjob-sample-1641736145   1/1           4s         6s
mycronjob-sample-1641736150   0/1           1s         1s
```

# webhook
Operator中的webhook，在外部对CRD资源的变更，在Controller处理之前，会先交给webhook提前处理。
webhook可以做两种类型的事：
- 修改（mutating）
- 验证（validating）
基于kubebuilder制作的webhook和controller，在同一个进程中

## 创建webhook
```
kubebuilder create webhook --group bytewang123-redis --version v1 --kind RedisCluster --defaulting --programmatic-validation
```
创建完成后，会在api/v1 目录下生成rediscluster_webhook.go脚手架文件

## 编写逻辑
```go
func (r *RedisCluster) Default() {
	redisclusterlog.Info("default", "name", r.Name)
	// TODO(user)
}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *RedisCluster) ValidateCreate() error {
	redisclusterlog.Info("validate create", "name", r.Name)
	// TODO(user)
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *RedisCluster) ValidateUpdate(old runtime.Object) error {
	redisclusterlog.Info("validate update", "name", r.Name)

	// TODO(user): fill in your validation logic upon object update.
	return nil
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *RedisCluster) ValidateDelete() error {
	redisclusterlog.Info("validate delete", "name", r.Name)

	// TODO(user): fill in your validation logic upon object deletion.
	return nil
}

```

可以看到有如上几个重要的方法：
1. Default
这是提交cr资源时，会执行的方法。如果我们希望提交资源时，默认给该资源加上一些特定的逻辑（修改属性、增加sidecar容器等），那我们可以把逻辑写在这个函数中
2. ValidateCreate/ValidateUpdate/ValidateDelete
这三个方法，是资源分别在创建/更新/删除时进行校验

## 创建cert-manager
和controller类似，webhook既能在kubernetes环境中运行，也能在kubernetes环境之外运行；
如果webhook在kubernetes环境之外运行，是有些麻烦的，需要将证书放在所在环境，默认地址是：
`/tmp/k8s-webhook-server/serving-certs/tls.{crt,key}`

为了省事儿，也为了更接近生产环境的用法，接下来的实战的做法是将webhook部署在kubernetes环境中
为了让webhook在kubernetes环境中运行，咱们要做一点准备工作安装cert- manager，执行以下操作：
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
```
这个命令会创建cert-manager相关的所有资源，创建完成后，我们需要让kubuilder工具使用certmanager，因此需要修改相关的配置文件，将之前本地调试模式的一些配置给打开，详情可见：[Deploying webhooks - The Kubebuilder Book](https://book.kubebuilder.io/cronjob-tutorial/running-webhook.html)

## 创建controller的镜像，推送到docker hub
```
make docker-build IMG=bytewang123/rediscluster:v0.0.6
make docker-push IMG=bytewang123/rediscluster:v0.0.6
```
## 以pod的形式部署controller
部署controller
```
make deploy IMG=bytewang123/rediscluster:v0.0.6
```

如果controller成功运行，说明部署成功，webhook和controller的逻辑都在该pod中
```sh
➜  redisclusterV2 kubectl get pod --all-namespaces -l control-plane=controller-manager
NAMESPACE               NAME                                                READY   STATUS    RESTARTS   AGE
redisclusterv2-system   redisclusterv2-controller-manager-b6fffc6c5-lnggm   2/2     Running   0          96m
```
## 提交资源
```sh
kubectl apply -f config/samples/bytewang123-redis_v1_rediscluster.yaml
```
## 试验
如果我们要求自定义的RedisCluster资源的addr属性必须是ip:port，但是提交的资源addr是空，此时会报错。
```sh
➜  redisclusterV2 kubectl apply -f config/samples/bytewang123-redis_v1_rediscluster.yaml
The RedisCluster "rediscluster-sample" is invalid: spec.addr: Invalid value: "": invalid ip port, ip:port
```

实现方式是Validate
```go
// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *RedisCluster) ValidateCreate() error {

	redisclusterlog.Info("validate create", "name", r.Name)
	allErrList := field.ErrorList{}
	if err := validate(
		r.Spec.Addr,
		field.NewPath("spec").Child("addr")); err != nil {
		allErrList = append(allErrList, err)
	}
	if len(allErrList) != 0 {
		return errors.NewInvalid(
			schema.GroupKind{Group: "bytewang123-redis", Kind: "RedisCluster"},
			r.Name, allErrList)
	}
	return nil
}

func validate(addr string, fldPath *field.Path) *field.Error {
	ipPort := strings.Split(addr, ":")
	if len(ipPort) != 2 {
		return field.Invalid(fldPath, addr, "invalid ip port, ip:port")
	}
	return nil
}
```


如果我们提交的资源，size是3，但是需要将其改为至少为6，这个逻辑可以放在Default中做
```go
// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *RedisCluster) Default() {
	redisclusterlog.Info("default", "name", r.Name)
	if r.Spec.Size < 6 {
		r.Spec.Size = 6
	}
}
```
此时可以看到，虽然我们提交yaml时，size是3，但是最终生成的cr size是6。在RedisCluster的controller中，逻辑是如果该资源所属的pod小于期望size，那么会创建pod直到满足需求，所以最终也会创建出6个pod。
```sh
➜  redisclusterV2 kubectl get pod -l app=redis
NAME               READY   STATUS    RESTARTS   AGE
redis-1642314756   1/1     Running   0          108m
redis-1642314757   1/1     Running   0          108m
redis-1642314758   1/1     Running   0          108m
redis-1642314759   1/1     Running   0          108m
redis-1642314760   1/1     Running   0          108m
redis-1642314761   1/1     Running   0          108m
```



#  TODO
1. k8s的插件机制是怎么样的？只需编写客户端代码就可以在集群中创建
	1. crd提交到集群，告诉集群自定义资源有哪些字段
	2. controller需要单独启动
		1. 控制器要做的第一件事，是从 Kubernetes 的 APIServer 里获取它所关心的对象，也就是用户自定义的CR对象。
		2. 这个操作，依靠的是一个叫作 Informer（可以翻译为：通知器）的代码库完成的。Informer 与 API 对象是一一对应的，所以初始化自定义控制器时传递了一个 CR 对象的 Informer。这个informer有一个CR对应的client，跟 APIServer 建立了连接。不过，真正负责维护这个连接的，则是 Informer 所使用的 Reflector 包。更具体地说，Reflector 使用的是一种叫作 ListAndWatch 的方法，来“获取”并“监听”这些 Network 对象实例的变化
		3. 在 ListAndWatch 机制下，一旦 APIServer 端有新的 Network 实例被创建、删除或者更新，Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为增量（Delta），它会被放进一个 Delta FIFO Queue（即：增量先进先出队列）中。而另一方面，Informe 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 Store
		4. 比如，如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象。
		5. 这个同步本地缓存的工作，是 Informer 的第一个职责，也是它最重要的职责。而 Informer 的第二个职责，则是根据这些事件的类型，触发事先注册好的 ResourceEventHandler。这些 Handler，需要在创建控制器的时候注册给它对应的 Informer。
		6. controller去api server查找自定义资源的前提，是集群里已经创建了对应的crd
2. controller的SetupWithManager函数如何使用？
3. crd的MyCronJobStatus字段如何使用
