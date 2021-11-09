## Steps
### 为项目创建一个项目目录并初始化该项目
```bash
mkdir memcached-operator
cd memcached-operator
operator-sdk init --domain zhqqqy.com --skip-go-version-check --repo github.com/zhqqqy/memcached-operator #最后一个参数用作GitHub开源
```
`operator-sdk init` 生成一个使用go modules 的go.mod 文件。当在$GOPATH/src之外创建项目时，`——repo=<path>`标志是必需的，因为脚手架文件需要有效的模块路径。。
在使用SDK之前，请确保您通过运行export GO111MODULE=on来激活模块支持。


构建镜像
```bash
make docker-build docker-push IMG="example.com/memcached-operator:v0.0.1"
```

## Manager
main程序为operator main.go初始化并运行管理器。

## 创建一个简单的Memcached API
使用`cache` 版本是v1alpha1的group和Kind Memcached创建一个新的自定义资源定义(CRD) API。当出现提示时，输入yes y用于创建资源和控制器。
```bash
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
```
这将在`api/v1alpha1/memcached_types`中构建Memcached资源API. 并且controller 在 `controllers/memcached_controller.go`中

## 定义API

首先，我们将通过定义`Memcached`类型来表示API，该类型有一个`MemcachedSpec.Size`字段设置要部署的`memcached`实例的数量，以及`MemcachedStatus.Node`字段用于存储CR的Pod名称。



通过修改`api/v1alpha1/memcached_types.go`的Go类型定义来定义`Memcached`自定义资源（CR）的API，以具有以下规范和状态：

```go
// MemcachedSpec 定义了memcached的状态
type MemcachedSpec struct {
	//+kubebuilder:validation:Minimum=0
	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}

// MemcachedStatus defines the observed state of Memcached
type MemcachedStatus struct {
	// Nodes are the names of the memcached pods
	Nodes []string `json:"nodes"`
}
```

添加`+kubebuilder:subresource:status`标记，将状态子资源添加到CRD清单中，这样控制器就可以在不改变CR对象其余部分的情况下更新CR状态

```go
// Memcached is the Schema for the memcacheds API
//+kubebuilder:subresource:status
type Memcached struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   MemcachedSpec   `json:"spec,omitempty"`
	Status MemcachedStatus `json:"status,omitempty"`
}
```

修改* _types.go文件后，始终运行以下命令以更新该资源类型的生成代码：`make generate`

以上makefile 会调用 [controller-gen](https://sigs.k8s.io/controller-tools) 工具来更新 `api/v1alpha1/zz_generated.deepcopy.go` 文件，以确保api的Go类型定义实现了所有Kind类型必须实现的`runtime.Object`接口。
```go
func (in *MemcachedStatus) DeepCopyInto(out *MemcachedStatus) {
	*out = *in
	if in.Nodes != nil {
		in, out := &in.Nodes, &out.Nodes
		*out = make([]string, len(*in))
		copy(*out, *in)
	}
}
```
会在这个函数中新增对应的字段代码

### Manifest

> Manifest是一个JSON或YAML格式的"Kubernetes API对象描述"，它描述了你对该API对象所期望的状态，由Kubernetes负责完成该期望。 配置文件可以包含其中的一个或多个Manifest，即Deployment、ConfigMap、Secret、DaemonSet 等API对象描述。

使用spec / status字段和CRD验证标记定义API后，可以使用以下命令生成和更新CRD Manifest：

```shell
make manifests
```

这个makefile target将调用controller-gen来生成`config/crd/bases/cache.example.com_memcacheds.yaml`上的CRD manifests。

## 资源被controller 监视











## [子资源](https://cloudnative.to/kubebuilder/reference/generating-crd.html#子资源)

在 Kubernetes 1.13 中 CRD 可以选择实现 `/status` 和 `/scale` 这类[子资源](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#status-subresource)。

通常推荐你在所有资源上实现 `/status` 子资源的时候，要有一个状态字段。

两个子资源都有对应的[标签][crd markers]。

### [状态](https://cloudnative.to/kubebuilder/reference/generating-crd.html#状态)

通过 `+kubebuilder:subresource:status` 设置子资源的状态。当时启用状态时，更新主资源不会修改它的状态。类似的，更新子资源状态也只是修改了状态字段。