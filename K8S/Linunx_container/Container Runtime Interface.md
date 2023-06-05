容器運行時插件（Container Runtime Interface，簡稱CRI）是Kubernetes v1.5 引入的容器運行時接口，它將Kubelet 與容器運行時解耦，將原來完全面向Pod 級別的內部接口拆分成面向Sandbox 和Container 的gRPC 接口，並將鏡像管理和容器管理分離到不同的服務。
![[Pasted image 20230109173223.png]]

CRI 最早從從1.4 版就開始設計討論和開發，在v1.5 中發布第一個測試版。在v1.6 時已經有了很多外部容器運行時，如frakti 和cri-o 等。v1.7 中又新增了cri-containerd 支持用Containerd 來管理容器。

採用CRI 後，Kubelet 的架構如下圖所示：

![[Pasted image 20230109173234.png]]

## CRI 接口

CRI 基於gRPC 定義了RuntimeService 和ImageService 等兩個gRPC 服務，分別用於容器運行時和鏡像的管理。其定義在

-   v1.14 以以上：[https://github.com/kubernetes/cri-api/tree/master/pkg/apis/runtime](https://github.com/kubernetes/cri-api/tree/master/pkg/apis/runtime)
-   v1.10-v1.13: [pkg/kubelet/apis/cri/runtime/v1alpha2](https://github.com/kubernetes/kubernetes/tree/release-1.13/pkg/kubelet/apis/cri/runtime/v1alpha2)
-   v1.7-v1.9: [pkg/kubelet/apis/cri/v1alpha1/runtime](https://github.com/kubernetes/kubernetes/tree/release-1.9/pkg/kubelet/apis/cri/v1alpha1/runtime)
-   v1.6: [pkg/kubelet/api/v1alpha1/runtime](https://github.com/kubernetes/kubernetes/tree/release-1.6/pkg/kubelet/api/v1alpha1/runtime)

Kubelet 作為CRI 的客戶端，而容器運行時則需要實現CRI 的服務端（即gRPC server，通常稱為CRI shim）。容器運行時在啟動gRPC server 時需要監聽在本地的Unix Socket （Windows 使用tcp 格式）。

### [](https://feisky.gitbooks.io/kubernetes/content/plugins/CRI.html#%E5%BC%80%E5%8F%91-cri-%E5%AE%B9%E5%99%A8%E8%BF%90%E8%A1%8C%E6%97%B6)開發CRI 容器運行時

開發新的容器運行時只需要實現CRI 的gRPC Server，包括RuntimeService 和ImageService。該gRPC Server 需要監聽在本地的unix socket（Linux 支持unix socket 格式，Windows 支持tcp 格式）。

一個簡單的示例為

```
import (
    // Import essential packages
    "google.golang.org/grpc"
    runtime "k8s.io/kubernetes/pkg/kubelet/apis/cri/runtime/v1alpha2"
)

// Serivice implements runtime.ImageService and runtime.RuntimeService.
type Service struct {
    ...
}

func main() {
    service := &Service{}
    s := grpc.NewServer(grpc.MaxRecvMsgSize(maxMsgSize),
        grpc.MaxSendMsgSize(maxMsgSize))
    runtime.RegisterRuntimeServiceServer(s, service)
    runtime.RegisterImageServiceServer(s, service)
    lis, err := net.Listen("unix", "/var/run/runtime.sock")
    if err != nil {
        logrus.Fatalf("Failed to create listener: %v", err)
    }
    go s.Serve(lis)

    // Other codes
}
```

對於Streaming API（Exec、PortForward 和Attach），CRI 要求容器運行時返回一個streaming server 的URL 以便Kubelet 重定向API Server 發送過來的請求。在v1.10 及更早版本中，容器運行時必需返回一個API Server 可直接訪問的URL（通常跟Kubelet 使用相同的監聽地址）；而從v1.11 開始，Kubelet 新增了`--redirect-container-streaming`（默認為false），默認不再轉發而是代理Streaming 請求，這樣運行時可以返回一個localhost 的URL（當然也不再需要配置TLS）。


![[Pasted image 20230109173335.png]]

詳細的實現方法可以參考[dockershim](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/dockershim)或者[cri-o](https://github.com/kubernetes-incubator/cri-o)。

### [](https://feisky.gitbooks.io/kubernetes/content/plugins/CRI.html#kubelet-%E9%85%8D%E7%BD%AE)Kubelet 配置

在啟動kubelet 時傳入容器運行時監聽的Unix Socket 文件路徑，比如

```
kubelet --container-runtime=remote --container-runtime-endpoint=unix:///var/run/runtime.sock --image-service-endpoint=unix:///var/run/runtime.sock
```

## 容器運行時

| **CRI** **容器運行時** | **維護者** | **主要特性**                 | **容器引擎**               |
| ---------------------- | ---------- | ---------------------------- | -------------------------- |
| **Dockershim**         | Kubernetes | 內置實現、特性最新           | docker                     |
| **cri-o**              | Kubernetes | OCI標準不需要Docker          | OCI（runc、kata、gVisor…） |
| **cri-containerd**     | Containerd | 基于 containerd 不需要Docker | OCI（runc、kata、gVisor…） |
| **Frakti**             | Kubernetes | 虛擬化容器                   | hyperd、docker             |
| **rktlet**             | Kubernetes | 支援rkt                      | rkt                        |
| **Virtlet**            | Mirantis   | 虛擬機和QCOW2image           | Libvirt（KVM）             |





### Containerd

以Containerd 為例，在1.0 及以前版本將dockershim 和docker daemon 替換為cri-containerd + containerd，而在1.1 版本直接將cri-containerd 內置在Containerd 中，簡化為一個CRI 插件。


![[Pasted image 20230109173430.png]]

Containerd 內置的CRI 插件實現了Kubelet CRI 接口中的Image Service 和Runtime Service，通過內部接口管理容器和鏡像，並通過CNI 插件給Pod 配置網絡。

![[Pasted image 20230109173445.png]]

## 運行時類

RuntimeClass 是v1.12 引入的新API 對象，用來支持多容器運行時，比如

-   Kata 容器/gVisor + runc
-   Windows進程隔離+Hyper-V隔離容器

RuntimeClass 表示一個運行時對象，在使用前需要開啟特性開關`RuntimeClass`，並創建RuntimeClass CRD：

```
kubectl apply -f https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/runtimeclass/runtimeclass_crd.yaml
```

然後就可以定義RuntimeClass 對象

```
apiVersion: node.k8s.io/v1alpha1  # RuntimeClass is defined in the node.k8s.io API group
kind: RuntimeClass
metadata:
  name: myclass  # The name the RuntimeClass will be referenced by
  # RuntimeClass is a non-namespaced resource
spec:
  runtimeHandler: myconfiguration  # The name of the corresponding CRI configuration
```

而在Pod 中定義使用哪個RuntimeClass：

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  # ...
```