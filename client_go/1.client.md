# client-go 

k8s系统使用的go语言编程交互库，位于k8s源码的`vendor/k8s.io/client-go`

## 源码目录结构
#### 客户端
- discovery     提供 DiscoveryClient 服务发现客户端
- dynamic       提供 DynamicClient  动态客户端
- scale         提供 scaleClient 客户端，用于扩缩容 deployment，replicaSet，replication controller等资源对象
- kubernetes    提供 clientSet 客户端
- rest          提供 RESTClient 客户端，对kubernetes  api server 执行RESTful操作
#### informer 实现
- informers     每种Kubernetes资源的Infomer 实现

#### listen
- listers       为每一个 kubernetes 资源提供lister功能，该功能对get和list 请求提供只读的缓存数据
#### 插件
- plugins       提供 OpenStack、GCP、azure 等云服务商授权插件
#### 工具包
- tools         提供常用工具，如 sharedInformer，reflector，dealtFIFO，Indexers。提供client查询和缓存机制，以减少向kube-apiserver发起的请求数等
- transport     提供安全的tcp连接，支持http stream，某些操作需要在客户端和容器之间传输二进制流，例如exec、attach等操作。该功能由内部spdy包提供
- util          提供常用方法，例如workqueue工作队列，certificate证书管理等

## 客户端对象
```  
                          ┏━> ClientSet
kubeconfig -> RESTClient ━╋━> DynamicClient
                          ┗━> DiscoveryClient
```
RESTClient 为最基础客户端，是对http client的封装。  
ClientSet、DynamicClient和DiscoveryClient都是基于RESTClient实现的。   

三种客户端的区别：
- ClientSet 仅能处理内置资源，它是通过client-gen代码生成器生成的。
- DynamicClient 能够处理k8s中所有资源对象，包括内置资源和 CRD 自定义资源。
- DiscoveryClient 用于发现 kube-apiserver 所支持的资源组（Group）、资源版本（Versions）、资源信息（Resources）



### kubeconfig 配置
配置内容示例
```yaml
apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CQo=
    server: https://192.168.31.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: default
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CQo=
    client-key-data: LS0tLS1CRUdJTiB==
```
- clusters: 定义k8s集群信息，例如 api地址、证书信息
- contexts: 定义k8s集群用户信息和命名空间等，用于将信息发送到指定集群
- user:  定义k8s集群用户身份认证的客户端凭据

### RESTClient
最基础的客户端，其他客户端都是基于RESTClient实现，使用时需要指定 Resource 和 Version 等信息，编写之前需要提前知道 Resource 所在的 Group 和对应的 Version 信息。

使用RESTClient列出Pod资源对象示例代码
```golang
package main

import (
	"context"
	"fmt"
	"time"

	corev1 "k8s.io/api/core/v1"
	matev1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/deprecated/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 解析kubeconfig配置文件
	config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
	if err != nil {
		panic(err)
    }
    // 设置 APIPATH 请求路径
    config.APIPath = "api"
    // 设置请求资源组、资源版本
    config.GroupVersion = &corev1.SchemeGroupVersion
    // 设置编解码器
    config.NegotiatedSerializer = scheme.Codecs
    // 通过配置实例化 RESTClient 客户端
	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
    result := &corev1.PodList{}
    // Get 设置请求方式为GET
    // Namespace 设置命名空间
    // Resource 设置请求的资源名称
    // VersionedParams 添加查询参数
    // Do 执行该请求 （ 对net/http进行封装）
    // Into 通过编解码器，解析返回结果
	err = restClient.Get().Namespace("").Resource("pods").VersionedParams(&matev1.ListOptions{Limit: 200}, scheme.ParameterCodec).Do(ctx).Into(result)
	if err != nil {
		panic(err)
	}
	for _, pod := range result.Items {
		fmt.Printf("namespace: %v\tname: %v\t status: %v\n", pod.Namespace, pod.Name, pod.Status.Phase)
	}
}
```

### ClientSet 客户端
ClientSet 在 RESTClient 的基础上封装了对 Resource 和 Version 的管理方法。每一个 Resource 可以理解为一个客户端，而 ClientSet 则是多个客户端的集合，每一个 Resource 和 Version 都以函数的方式暴露给开发者。

>注：ClientSet仅能访问k8s自身内置的资源（即客户端集合内的资源），不能直接访问CRD自定义资源。

使用 ClientSet 列出Pod资源对象示例代码
```golang
package main

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
	if err != nil {
		panic(err)
    }
    // 通过 config 配置实例化 ClientSet 对象
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
    }
    // 请求 core 资源组的 v1 资源版本下的 Pod 资源对象
    // 内部设置了 http路径、GroupVersion 请求的资源组、资源版本、NegotiatedSerializer 数据的编解码器
	podClient := clientSet.CoreV1().Pods(corev1.NamespaceAll)
    ctx := context.Background()
    // 通过 List 方法获取结果
	list, err := podClient.List(ctx, metav1.ListOptions{Limit: 200})
	if err != nil {
		panic(err)
	}
	for _, pod := range list.Items {
		fmt.Printf("namespace: %v\tname: %v\t status: %v\n", pod.Namespace, pod.Name, pod.Status.Phase)
	}
}
```

### DynamicClient 客户端
一种动态客户端，可以对任意k8s资源进行RESTful操作，包括CRD自定义资源。DynamicClient 内部实现了 Unstructured，用于处理非结构化的数据（即无法提前预知的数据结构），这是处理CRD自定义资源的关键。
> 注：DynamicClient 不是类型安全的，因此在访问CRD资源时需要特别注意。例如在指针操作不正当的情况下可能会导致panic。

DynamicClient 的处理过程是将 Resource 转化为 Unstructured 结构类型，k8s 的所有Resource都可以转化为该结构类型。处理完成后再将 Unstructured 转化为对应的 Resource 资源类型。
> Unstructured 结构类型是通过 map[string]interface{} 转化的。

使用 DynamicClient 列出Pod资源对象示例代码
```golang
package main

import (
	"context"
	"fmt"

	apiv1 "k8s.io/api/core/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
	if err != nil {
		panic(err)
	}
	// 通过config 实例化dynamic client
	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	// 设置gvr（groups、version、Resource）
	gvr := schema.GroupVersionResource{Version: "v1", Resource: "pods"}
    ctx := context.Background()
    // Resource 设置gvr
    // Namespace 指定命名空间
	// List 获取指定资源，得到 unstructured.UnstructuredList 指针类型
	unstructuredObj, err := dynamicClient.Resource(gvr).Namespace(apiv1.NamespaceAll).List(ctx, metav1.ListOptions{Limit: 20})
	if err != nil {
		panic(err)
	}

	podList := &corev1.PodList{}
	// unstructured.UnstructuredList 转 PodList
	err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructuredObj.UnstructuredContent(), podList)
	if err != nil {
		panic(err)
	}
	for _, pod := range podList.Items {
		fmt.Printf("namespace: %v\tname: %v\t status: %v\n", pod.Namespace, pod.Name, pod.Status.Phase)
	}
}

```

### DiscoveryClient 客户端
发现客户端，主要用于发现k8s api server 支持的资源组、资源版本、资源信息，还可以将这些信息存储到本地，用户本地缓存（cache），减轻对 k8s api-server访问压力。缓存默认存储于`~/.kube/cache`和`~/.kube/http-cache`下。

kubectl 的 api-versions 和 api-resources 命令输出就是通过 DiscoveryClient 实现的。

使用 DiscoveryClient 列出Pod资源对象示例代码
```golang
package main

import (
	"fmt"

	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
	if err != nil {
		panic(err)
	}
	// 通过config 实例化 discovery client
	discoverClient, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err)
	}
	// 返回支持的资源组，资源版本，资源信息
	_, APIResponse, err := discoverClient.ServerGroupsAndResources()
	if err != nil {
		panic(err)
	}
	for _, list := range APIResponse {
		gv, err := schema.ParseGroupVersion(list.GroupVersion)
		if err != nil {
			panic(err)
		}
		for _, resource := range list.APIResources {
			fmt.Printf("name: %v\t group: %v\tversion: %v\n", resource.Name, gv.Group, gv.Version)
		}
	}
}
```



## 小结
RESTClient 客户端是其他客户端的基础，其余客户端都是基于RESTClient提供的方法进行二次封装而来。ClientSet只可以操作k8s内置的资源对象，DynamicClient 不仅可以操作内置资源对象，还可以操作CRD自定义资源。DiscoveryClient 的用途在于发现api-server支持的资源组、资源版本、资源信息等。