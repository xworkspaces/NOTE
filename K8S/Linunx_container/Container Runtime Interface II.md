# Container Runtime Interface (CRI)

對於 `kubernetes` 來說，希望能夠透過一個標準介面與各個 `contaienr rumtime` 的解決方案銜接，這個銜接的接口標準就是所謂的 `Container Runtime Interface`

由於 `CRI` 的標準就是一些相關的介面，這意味只要任何 `CRI Runtime` 有實作這些介面，都可以跟 `kubernetes` 銜接來處理所有跟 `Pod` 有關的操作。

剩下唯一問題就是，之前所探討過的 docker 運作流程

> docker client -> docker engine -> docker-containerd -> docker-containerd-shim -> runc -> container

這個架構要怎麼跟 `kubernetes & CRI` 整合？

###### [](https://hackmd.io/@aes/ryYuGY1Zu#tags-kubernetes-k8s-container-runtime-interface-cri-docker-containerd-cri-o "tags-kubernetes-k8s-container-runtime-interface-cri-docker-containerd-cri-o")tags: `kubernetes`, `k8s`, `container runtime interface`, `cri`, `docker`, `containerd`, `cri-o`

## [](https://hackmd.io/@aes/ryYuGY1Zu#Docker-amp-Kubernetes "Docker-amp-Kubernetes")Docker & Kubernetes

`CRI` 本身是個溝通介面，這代表溝通的兩方都需要根據這個界面去實現與滿足。對於 `kubernetes` 來說，`kubelet` 自己維護與開發的，要支援 `CRI` 本身就不是什麼困難的事情。

但是另外一端如果要使用 `docker` 的話，那到底要怎麼辦？ `docker`背後也是有公司再經營，也不是說改就改，這種情況下到底要如何將 `docker` 給整合進來？

最直觀的想法就是如果沒有辦法使得 `docker` 本身支援 `CRI` 的標準，那就額外撰寫一個轉接器，其運作在 `kubelet` 與 `Docker`，該應用程式上承 CRI 與 `kubernetes` 溝通，下承 `Docker API` 與 `Docker Engine` 溝通。

早期的 `kubernetes` 採取了這種做法，`kubelet` 內建相關了 `dockershim` 的程式碼來處理這段邏輯。這種做法可行，但是其實效能大大則扣，同時也把整體架構帶到了更複雜的境界，引進愈來愈多的元件會對開發與除錯帶來更大的成本。

可以參考下圖中的上半部份，而圖中的下半部分則是後來的改變之處

![[Pasted image 20230110084630.png]]
反正最後都是透過 `containerd` 進行操作，而本身也不太需要 `docker` 自己的功能，那是否就直接將 `dockershim` 溝通的角色從 `docker engine` 轉移到 `containerd` 即可。因此後來又有所謂的 `CRI-Containerd` 的出現。

 留言

到這個階段，已經減少了一個溝通的 `Daemon`，也就是 `docker engine`，但是這樣並不能滿足想要最佳化的心情。

伴隨者 `Containerd` 本身的功能開發，提供了 `Plugin` 這種外掛功能的特色後，將 `CRI-Containerd` 的功能直接整合到該 `Plugin` 內就可以直接再次減少一個元件，直接讓 `kubelet` 與 `containerd` 直接溝通來創建相關的 `container`。

相關的演進可以參考下圖

![[Pasted image 20230110084647.png]]

這種架構下，使用者可以在一台伺服器中同時安裝 `kubernetes` 與 `docker`，同時彼此會共用 `containerd` 來管理自己所需要的 `container`。

 留言

架構如下圖，有趣的一點在於這種情況下要如何確保 `docker` 的指令不會看到 `kubernetes` 所要求創建的 `container`，反之亦然。  
兩者都是透過 `containerd` 來創建 `Container`，幸好有鑒於 `containerd` 本身提供的 `namespace` 的功能，可以確保不同的客戶端 `docekrd`，`CRI-plugin` 都可以有自己的 `namespace`，所以用 `docker ps` 就不會互相影響到彼此的運作。

![[Pasted image 20230110084703.png]]

不過上述的假設是 啟用 containerd 於 kubernetes cluster 內才會有這個效果。

 留言

根據 這篇官方文章 install-kubeadm，目前預設的情況下都還是採用第一種方案, dockershim 的方式來使用，若需要使用 containerd 則必須要先安裝 containerd 到系統之中並且安裝 kubernetes 時設定特定的參數來切換過去。

## [](https://hackmd.io/@aes/ryYuGY1Zu#Docker-vs-Containerd-vs-CRI-O "Docker-vs-Containerd-vs-CRI-O")Docker vs Containerd vs CRI-O

直接透過一張圖與 `docker`, `containerd`, `cri-o` 的比較，來解釋各個的角色與地位。

![[Pasted image 20230110084718.png]]

該圖片從縱軸來看，有兩條主要的黑線，代表的是不同的標準架構，分別是 `CRI` 以及 `OCI`。  
`kubelet` 本身透過 `CRI` 的介面與各式各樣相容於 `CRI` 的解決方案溝通，而這些解決方案最後都會透過符合 `OCI` 標準的 `OCI runtime` 去創造出真正的 `Container` 供使用者使用。

 留言

從上而下分別是之前介紹過 `kubernetes` 內與 `docker`，`contained` 以及 `cri-o` 的架構演進圖。

1.  `Docker`，`kubelet` 透過 `Dockershim` 與 `docker engine` 連接，最後一路串接到 `containerd` 來創建 `container`。
2.  `Containerd 1.0`，繞過 `Docker` 直接與後端的 `Containerd` 溝通，為了滿足這個需求也需要一個額外的應用程式 `CRI-Containerd` 來作為中間溝通的橋樑。
3.  `Containerd 1.1`， `CRI-Containerd` 本身的功能已經可以透過 `plugin` 的方式實現於 `containerd` 中，可以再少掉一層溝通的耗損，這也是前面所介紹。
4.  `cri-o`，一個完全針對 `kubernetes` 需求的解決方案，讓整體的溝通變得更快速與簡單。

看完上述比較後會對 `cri-o` 有個初步的理解，知道其被設計出來的目的就是要提供更好地整合，減少多餘的 `IPC` 溝通，並且作為一個針對 `kubernetes` 設計的解決方案。

## [](https://hackmd.io/@aes/ryYuGY1Zu#CRI-O "CRI-O")CRI-O

`CRI-O` 的標題開宗明義直接闡明

> CRI-O - OCI-based implementation of Kubernetes Container Runtime Interface

作為一個滿足 `CRI` 標準且能夠產生出相容於 `OCI` 標準 `container` 的解決方案，從整個設計到特色全部都是針對 `kubernetes` 來打造。

1.  本身的軟體版本與 `kubernetes` 一致，同時所有的測試都是基於 `kubernetes` 的使用去測試，確保穩定性。
2.  目標是支援所有相容於 OCI Runtime 的解決方案，譬如 Runc, Kata Containers
3.  支援不同的 `container image`，譬如 `docker` 自己本身就有 [schema 2/version 1](https://docs.docker.com/registry/spec/manifest-v2-1/) 與 [schema 2/version 2](https://docs.docker.com/registry/spec/manifest-v2-2/)
4.  使用 `Container Network Interface (CNI)` 來管理 `Container` 網路

### [](https://hackmd.io/@aes/ryYuGY1Zu#%E9%81%8B%E4%BD%9C%E6%B5%81%E7%A8%8B "運作流程")運作流程

![[Pasted image 20230110084757.png]]

1.  `kubelet` 決定要創建一個 `Pod`，於是透過 `gRPC` 的方式發送基於 `CRI` 標準的請求到 `cri-o`。
2.  `cri-o` 基於 `containerts/image` 的函式庫去該 `Pod` 裡面描述的 `Container Image Registry` 抓取該 `container image` 。
3.  下載的 `container image` 會被解開，接下來會透過 `containers/storage` 相關的函式庫去處理 `container` 本身的 `root filesystem`。
4.  `CRI-O` 接著會使用 `OCI` 提供的工具去產生一個用來描述該 `container` 要如何運行的 `json` 檔案。
5.  接者會根據設定去運行相容於 `OCI Runtime` 的解決方案來執行該 `container` 。
6.  每一個 `container` 都會被獨立的 `process conmon (container monitor)` 給監控，處理者關於 pseudotty，log 以及 exit code。
7.  接下來會透過 `CNI` 的介面來幫該 `Pod` 建立網路。

實際上 CNI 操作的對象是所謂的 infra container (pause container)，而非任何使用者請求的 container。