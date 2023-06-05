其實 [Kubernetes 官方的文件](https://kubernetes.io/)，已經將 Kubernetes 上的每個元件的功能都已解釋的非常清楚。官網上現在也提供 [互動式學習的教學](https://kubernetes.io/docs/tutorials/kubernetes-basics/cluster-interactive/)，對於有心想學習Kubernetes讀者，這兩者都是非常有幫助的資源。然而筆者在自學的過程中，發覺若是能先`暸解 Kubernetes 內部如何運作`，對於之後學習 `Kubernetes 的基礎元件`是非常有幫助的，因此今天的學習筆記，除了介紹 [Node 元件](https://kubernetes.io/docs/concepts/architecture/nodes) 以外，也會從一個概觀的角度去介紹 Kubernetes 的內部運作。

今日內容如下：

-   介紹 Node 是什麼
-   概觀 Kubernetes 的內部運作

# Node 是什麼

在 Kubernetes 中，[Node](https://kubernetes.io/docs/concepts/architecture/nodes) 通常是指實體機、虛擬機。一個Node，可以是指 [AWS的一台EC2](https://aws.amazon.com/tw/ec2/) 或 [GCP上的一台computer engine](https://cloud.google.com/compute/docs/instances/) ，也可以是你的筆電，甚至是一台 [Raspberry Pi](https://zh.wikipedia.org/wiki/%E6%A0%91%E8%8E%93%E6%B4%BE)。只要他們上面裝有 [Docker Engine](https://docs.docker.com/) ，足以跑起 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)，就可以被加入 Kubernetes Cluster。

將 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 加到 Kubernetes 中之後，Kubernetes 會建立一個Node 物件，並進行一連串檢查，包含網路連線，Pod 是否能被正常啟動等，若都通過則會將該Node物件的狀態設為 `Ready`，若是無法通過則會顯示 `Not Ready`

若還是記得 [第二天在本機端架好的 minikube](https://ithelp.ithome.com.tw/articles/10192490)，輸入 `kubectl get nodes` 指令，就可以發現 `minikube` 已在其中

![kubectl-get-nodes](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day06/kubectl-get-nodes.png?raw=true)

當然也可以使用 `kubectl describe nodes` 取得該 Node 的詳細資料，以下以 minikube 為例 (更多 log 可參考 [kubectl-describe-node-minikube](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day06/kubectl-describe-node-minikube.log))，

```
$ kubectl describe nodes minikub
```

> 由於我們目前是在本機端操作，只有一個 Node。在 [未來的學習筆記](https://ithelp.ithome.com.tw/articles/10193248) 中，也會介紹到如何利用第三方套件 [kops](https://github.com/kubernetes/kops) 在 AWS 上打造多台機器的 Kubernetes Cluster

### 用 Kubernetes 管理 Nodes 的好處

過去是一台 Node 指運行可能只運行一個應用服務(application)，若是該應用服務(application) 只使用 I/O 資源，免不了 Node 上閒置 memory 的浪費；若是為了避免閒置資源，而將多個服務同時運行在同一台 Node 上，運維人員還需隨時掌控目前各個應用服務使用資源狀況，避免資源不足情形。而在 Kubernetes 上，當我們將 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 都加到 Kubernetes Cluster 後，系統會根據目前 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 的設定檔去決定要部署在哪個 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 上。在未來讀書筆記 [[Day 27] 如何限制每個服務資源 - Resource Quotas](https://ithelp.ithome.com.tw/articles/10193248) 也會分享這部分。

# 概觀 Kubernetes 的內部運作

在實際場景應用上，Kubernetes 是會由多台 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 組成，今天將藉由底下這張 `簡化的 Kubernetes 系統架構圖` 講解 Kubernetes 的內部是如何溝通(圖中並無 [Master Node](https://kubernetes.io/docs/concepts/architecture/master-node-communication/))：

> [Master Node](https://kubernetes.io/docs/concepts/architecture/master-node-communication/) 主要用來管理與調度Kubernetes Cluster 中所有物件的狀態，讀者只需有有該概念即可，在 [[Day 26] 介紹Kubernetes調度中心 - Master Node](https://ithelp.ithome.com.tw/articles/10193248) 會更詳細介紹。

![[Pasted image 20230201134047.png]]

-   該圖中的 Kubernetes Cluster 有`N 個 Nodes`，Node 1, Node 2, ... Node N
    
-   每個 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 都必須跑起 [Container Engine](https://docs.docker.com/engine/) 以便於運行 Pods。  
    而在30天教學筆記中使用的是 [Docker Engine](https://docs.docker.com/)。
    
-   圖中，**一個橘色框分別代表一個 Pod ，而 Pod 中綠色的空心方型則代表一個container**。以 `Pod 3 與 Pod 5` 為例，這兩個 Pod 分別只運行一個 container；而在 `Pod 1` 中則運行三個 containers。
    
-   圖中每個 Node 都有屬於它自己的 [iptables](https://zh.wikipedia.org/wiki/Iptables)，`iptables` 是 Linux 上的防火牆(firewall)，不只限制哪些連線可以連進來，也會管理網路連線，決定收到的 request 要交給哪個 Pod。
    

在對這張架構圖有基本認識之後，我們會藉由這張圖來解釋每個 Kubernetes 的內部是如何溝通：

### 1. 同一個 Pod 中的 containers，可以透過 Pod 內部的網路直接溝通

由於每個Pod中有自己的網路。所以同一個Pod中的containers之間可以透過 **<localhost:port_num>** 互相溝通。而這樣無需透過外網的性質，讓我們可以將性質相近的服務放在同一個Pod裡，好比一個後端API service，與一個後端認證service，可以放在同一個pod裡互相溝通。

### 2. Kubelet & kube-proxy

在每個 [Node物件](https://kubernetes.io/docs/concepts/architecture/nodes) 中都會有，`kubelet`與`kube-proxy`

-   **kubelet**  
    kubelet相當於 node agent，用於`管理該 Node 上的所有 pods`以及`與 master node 即時溝通`。
    
-   **kube-proxy**  
    kube-prox y則是會將目前該 Node 上所有 Pods 的資訊傳給 `iptables`，讓 `iptables` 即時獲得在該 Node 上所有 Pod 的最新狀態。好比當一個 [Pod物件](https://kubernetes.io/docs/concepts/architecture/pods) 被建立時，**kube-proxy** 會通知 **iptables**，以確保該 Pod 可以被 Kubernetes Cluster 中的其他物件存取。
    
-   **Load balancing**  
    在實際場景中，收到外部傳來的 requests 都會先交由 [Load balancer](https://en.wikipedia.org/wiki/Load_balancing_(computing)) 處理，再由 Load balancer 決定要將 request 給哪個 Node ，通常 Load balancer 都是由 Node 供應商提供，好比 [AWS 的 ELB](https://aws.amazon.com/tw/elasticloadbalancing/)，或是自己架設 [Nginx](https://nginx.org/en/) 或 [HAProxy](http://www.haproxy.org/) 等專門做負載平衡的服務。
    

### 3. 所以，當我們創建一個新的 Pod 後，收到使用者發送的 request 時，ks8 內部會發生什麼事呢

-   首先，**kubelet** 會先收到 **master node** 指令，創建一個 **Pod**。創建好後，`kube-proxy` 會去告知 `iptables`，目前該 Pod 可用。
    
-   當使用者透過 **網路(internet)** 發送 requests 時，requests 會先送到 **Load Balancer**，由 **Load Balancer** 決定要把request交給哪個 **Node**，這時收到 requests的 **Node** 會經由 **iptables** 決定要送給哪個 **Pod** 。
    
-   但如果收到request的 **Node** 恰好沒有相對應可以處理 request **Pod** 的話，原本收到 request 的 **Node** 會透過 **iptables** 把 request 轉給其他有可以處理這個 request 的 **Node** 。

Kubernetes 的 [Github](https://github.com/kubernetes/kubernetes)