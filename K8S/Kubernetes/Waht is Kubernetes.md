## 為什麼該用 Kubernetes?

Kubernetes 中間共有八個字母有點長，所以大家就將他簡稱為 K8S。但使用 K8S 有什麼好處？

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#1-%E5%8F%AF%E4%BB%A5%E6%9B%B4%E5%A5%BD%E7%9A%84%E9%81%8B%E7%94%A8%E9%9B%B2%E7%AB%AF%E6%88%96%E6%98%AF%E5%AF%A6%E9%AB%94%E8%B3%87%E6%BA%90 "1. 可以更好的運用雲端或是實體資源")1. 可以更好的運用雲端或是實體資源

所有的資源集中成了一個大平台，所以調度上更靈活，以往我們以實體機為單位的方式很沒有效率，要調度資源的時候需要開一台實體機器，或是虛擬機器，都很耗費 CPU、記憶體等等的資源。

而 K8S 內所有的東西都是容器，可以很快啟動，很快的刪除，並且靈活部屬在 Kubernetes 所擁有的資源上。

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#2-%E8%AE%93%E4%B8%80%E5%88%87%E7%9A%84%E5%9F%BA%E7%A4%8E%E8%A8%AD%E6%96%BD%E9%83%BD%E5%AF%AB%E6%88%90%E7%A8%8B%E5%BC%8F%E7%A2%BC "2. 讓一切的基礎設施都寫成程式碼")2. 讓一切的基礎設施都寫成程式碼

應用程式容器化之後，所有需要安裝的套件都會寫成 Dockerfile。這樣在新增或修改的時候，不再像是以前的伺服器是個黑盒子，需要花大量的時間除錯。

部屬的資源則用 Kubernetes 的描述方式撰寫，要前端服務要開幾台，後端服務要開幾台，要自動擴展？ 沒問題，這些 K8S 都可以輕鬆幫你做到。所以你如果要了解整個基礎設施架構時，可以很快的藉由程式碼來認識。

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#3-%E5%8F%AF%E4%BB%A5%E5%B9%AB%E5%8A%A9%E9%96%8B%E7%99%BC%E8%80%85%E8%81%9A%E7%84%A6%E9%96%8B%E7%99%BC "3. 可以幫助開發者聚焦開發")3. 可以幫助開發者聚焦開發

減少開發者在基礎設施上花的時間，將硬體統一看做一個大平台，開發者只需要寫應用的描述，其他的 K8S 幫你搞定。例如：有節點當機，會自動生成一個新的節點，以維持服務的穩定。

## [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#%E4%B8%80%E5%88%87%E5%BE%9E-Container-%E9%96%8B%E5%A7%8B "一切從 Container 開始")一切從 Container 開始

使用 Kubernetes 之前，你需要把你的服務先容器化，或者用人家包好的 Image 建立。例如：你有一個 Node.js 的應用程式、一個 MySQL 的資料庫，都可以架設在 K8S 上面。

K8S 提供了豐富的、可以應用於產品環境的一切資源給你。例如：自動擴展、負載均衡、定時工作 … 等等一切你想得到的東西。

但是在開始使用 K8S 之前，你需要把你的服務先容器化。雖然一開始很痛苦，需要花很多時間做原本不必要做的事情，但是你容器化你的服務之後會發現，以前需要在不知道被做過什麼事情的機器上摸索的體力活，通通都自動化、或是很易於找到解法，因為在程式碼裡面都有跡可尋。

## [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#%E7%90%86%E8%A7%A3-Node%E3%80%81Pod%E3%80%81Container-%E4%B9%8B%E9%96%93%E7%9A%84%E9%97%9C%E4%BF%82 "理解 Node、Pod、Container 之間的關係")理解 Node、Pod、Container 之間的關係

Node 是 K8S 中的一台實體機器、或是雲端上的一台機器，又稱作是工作者。他有個別名叫做奴隸 (minion) ，挺有趣的。

Pod 是 K8S 中基本的單位，負責裝一個或多個多個 Container (容器)。

Container 中就是我們容器化好的應用程式，例如：Node.js 應用程式、MySQL 服務 … 等等

需要 Pod 來作為基本單位的原因是，如果每個 Container 都作為 K8S 的最小單位，那麼管理網路會變得非常的困難。以 Pod 來區隔，同一個 Pod 裡面的 Container 能夠在本地端互相的連線，只有需要提供給外部呼叫的 API 才需要暴露出來。

示意圖如下：

![[Pasted image 20230112131821.png]]

## [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#%E7%90%86%E8%A7%A3-Kubernetes-Cluster "理解 Kubernetes Cluster")理解 Kubernetes Cluster

Kubernetes 集群由控制面板 Control Panel 與節點 Node 所組成。控制面板又稱為是 Kubernetes Master。

![[Pasted image 20230112131838.png]]

控制面板由幾個元件 (Component) 所組成：

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#1-Kube-API-Server "1. Kube API Server")1. Kube API Server

控制面板中用來暴露 Kubernetes API 的元件，讓其他服務可以讀寫 K8S 的資源物件 (Resouce Object)。

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#2-Kube-Schedular "2. Kube Schedular")2. Kube Schedular

調度器，需要調度軟體、硬體資源的時候就要靠調度器囉。例如：如果新建立的 pod 沒有 node 可以放的時候，調度器就會開啟一個新的 node，來放置剛剛需要建立的 pod。

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#3-Kube-Controller-Manager "3. Kube Controller Manager")**3. Kube Controller Manager**

是一個在背景持續執行的程序 (daemon)，用來調節系統狀態，透過 api-server 可以監視 Cluster 共享的狀態。

需要變更目前狀態的時候 Kube Controller 就會將目前的狀態變更到想要變更的狀態，例如：本來 2 個副本 (Replica) 擴展到 4 個副本。

包含了副本控制器 (Replication Controller)，端點控制器 (Endpoint Controller)、命名空間控制器(Namepsace Controller)與服務帳號控制器

### [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#4-Cloud-Controller-Manager "4. Cloud Controller Manager")**4. Cloud Controller Manager**

基於 Kube Controller Manager，各個雲平台提供者（Provider）的實作。而每個 Node 則包含：

-   kubelet — 用來跟 Master 溝通的元件。
    
-   kube-proxy — 網路代理，用來反應 K8S 各個 Node 上的網路服務
    

## [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#%E8%AE%80-Kubernetes-API-%E5%88%9D%E6%8E%A2-K8S-%E7%9A%84%E8%B3%87%E6%BA%90%E7%89%A9%E4%BB%B6 "讀 Kubernetes API 初探 K8S 的資源物件")讀 Kubernetes API 初探 K8S 的資源物件

我們可以透過 Kubernetes API 讀寫 K8S 的資源物件 (Resource Object)，剛剛說的 Kubernetes Cluster 就分為 Kubernetes API 總共分為五大類，分別是：

-   Workload 物件 — 用來「管理或是運行 Container」 在 Cluster 上。
    
-   服務發現與負載均衡物件 — 讓 Workload 可以「縫住」形成可被外部存取到的服務，或是有負載均衡能力的服務。
    
-   Config 與 Storage 物件 — Config 用來將配置注入你的應用程式中。Storage 讓 Container 的資料可以永久保存在 Container 之外。
    
-   Cluster 物件 — 用來定義集群本身的物件。
    
-   Meta 物件 — 用來配置資源之間的行為的物件。
    

這種分類法較接近開發者，可以藉此看看開發者在想些什麼。

![[Pasted image 20230112131906.png]]

還有 kubectl 範例可以使用，很方便。

![[Pasted image 20230112131916.png]]

## [](https://luka.tw/uncategorized/past/2020-02-11-kubernetes-tutorial-01-cd76ae4c25e5/#kubectl-%E2%80%94-%E8%B7%9F-K8S-Cluster-%E6%BA%9D%E9%80%9A%E7%9A%84%E5%B7%A5%E5%85%B7 "kubectl — 跟 K8S Cluster 溝通的工具")kubectl — 跟 K8S Cluster 溝通的工具

我們絕大多數對 K8S 的操作都需要透過 kubectl，kubectl 的是什麼呢？DevOps 開發者用 kubectl 命令列工具，可以透過 Kubernetes Master 上的 api-server 對各個 Node 下達指令。

![[Pasted image 20230112131940.png]]