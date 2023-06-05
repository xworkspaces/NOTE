# 

前言 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E5%89%8D%E8%A8%80)

[前篇文章](https://tachingchen.com/tw/blog/kubernetes-service/)介紹 Service 在 Kubernetes 裡的基礎概念及應用，但 Service 透過不同路徑存取 Pod 間的差異在哪裡，以及服務營運上的細節、注意事項 ，接下來在本篇文章中將為各位逐一說明。

# 

Service 存取路徑裡跟外 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#service-%E5%AD%98%E5%8F%96%E8%B7%AF%E5%BE%91%E8%A3%A1%E8%B7%9F%E5%A4%96)

前篇文章內有提到，有兩種角色會透過 Service 存取其他的 Pod，分別是 Kubernetes 集群外部使用者以及集群內部其他 Pod。儘管兩種路徑的結果都相同，但之間仍存在一個關鍵差異在於: `存取 IP 不同`。

實際上在 Kubernetes 內共有三種 IP 存在

-   External IP: 裸露的網路位址，供外部使用者連線
-   Cluster IP: 叢集內部的網路位址，在 Cluster 內的 Pod 可以透過此位址存取該服務
-   Pod IP: 每個 Pod 獨有的網路位址，只有叢集內可連線

```bash
$ kubectl get service
NAME          CLUSTER-IP      EXTERNAL-IP       PORT(S)     AGE
example-svc   10.31.251.56    35.185.145.186    80/TCP      2d
```

一個 Service 會擁有兩種 IP

-   Cluster IP: 該服務在集群內用來存取的 IP 位址
-   External IP: 指定 service 為 `type: LoadBalancer` 時，Cloud Provider (如 GCP、AWS) 配發的 Public IP，

```bash
$ kubectl get pod -o wide
NAME           READY   STATUS    RESTARTS   AGE    IP           NODE
example-pod    1/1     Running   0          14d    10.4.3.162   example-node
```

每個 Pod 本身也有自己獨立的 `Pod IP`，此 IP 可以被叢集內的 Pod 直接存取。

-   三者關係圖

![[Pasted image 20230201142317.png]]

## 

路徑 1 -> 2: [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E8%B7%AF%E5%BE%91-1---2)

外部使用者連線到時的 destination 為 `External IP`，當使用者請求送到 Service 時會經由 iptables NAT 將 destination 替換成 `Pod IP`。

整個 iptables 的轉換如下

```bash
// 收到封包且 dst IP 為該服務的 external IP 時，將封包送到 KUBE-FW-HQT2B7A3AOLJKY7O 處理
-A KUBE-SERVICES -d 104.123.111.1/32 -p tcp -m comment --comment "default/example:svc loadbalancer IP" -m tcp --dport 3457 -j KUBE-FW-HQT2B7A3AOLJKY7O

// 將封包送到 KUBE-SVC-HQT2B7A3AOLJKY7O 處理
-A KUBE-FW-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc loadbalancer IP" -j KUBE-MARK-MASQ
-A KUBE-FW-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc loadbalancer IP" -j KUBE-SVC-HQT2B7A3AOLJKY7O
-A KUBE-FW-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc loadbalancer IP" -j KUBE-MARK-DROP

// 透過隨機 (--probability) 的方式將封包分送到不同 iptables chain
-A KUBE-SVC-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc" -m statistic --mode random --probability 0.33333333333 -j KUBE-SEP-3LCXQQA74ENIYBWW
-A KUBE-SVC-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-KEBKTHZKZPI5ZKRT
-A KUBE-SVC-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc" -j KUBE-SEP-UNUQIMG47GFWO2LF

// 將封包的 dst IP 更換成 Pod IP 後將封包送至對應 Pod
-A KUBE-SEP-366YKQOU5GJVFPAM -s 10.4.2.226/32 -m comment --comment "default/example:svc" -j KUBE-MARK-MASQ
-A KUBE-SEP-366YKQOU5GJVFPAM -p tcp -m comment --comment "default/example:svc" -m tcp -j DNAT --to-destination 10.4.2.226:3457
```

## 

路徑 3 -> 2: [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E8%B7%AF%E5%BE%91-3---2)

內部 Pod存取服務時的 destination 為 `Cluster IP`，當使用者請求送到 Service 時會經由 iptables NAT 將 destination 替換成 `Pod IP`。

整個 iptables 的轉換如下

```bash
// 從該服務的 port 進來的封包，送到 KUBE-SVC-HQT2B7A3AOLJKY7O
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/example:svc" -m tcp --dport 30928 -j KUBE-SVC-HQT2B7A3AOLJKY7O

// 透過隨機 (--probability) 的方式將封包分送到不同 iptables chain
-A KUBE-SVC-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc" -m statistic --mode random --probability 0.33333333333 -j KUBE-SEP-3LCXQQA74ENIYBWW
-A KUBE-SVC-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-KEBKTHZKZPI5ZKRT
-A KUBE-SVC-HQT2B7A3AOLJKY7O -m comment --comment "default/example:svc" -j KUBE-SEP-UNUQIMG47GFWO2LF

// 將封包的 dst IP 更換成 Pod IP 後將封包送至對應 Pod
-A KUBE-SEP-366YKQOU5GJVFPAM -s 10.4.2.226/32 -m comment --comment "default/example:svc" -j KUBE-MARK-MASQ
-A KUBE-SEP-366YKQOU5GJVFPAM -p tcp -m comment --comment "default/example:svc" -m tcp -j DNAT --to-destination 10.4.2.226:3457
```

上面兩種服務存取路徑的方式都是透過替換封包內 dst 資訊，再透過隨機 (--probability) 的方式將封包分送到不同 Pod 處理，進而達到負載平衡 (Load balancing) 的作用。

---

# 

注意要點 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E6%B3%A8%E6%84%8F%E8%A6%81%E9%BB%9E)

-   Pod 之間是否能夠直接使用彼此 IP 溝通，而無需 Service 存在？

可以，但不建議這種作法。Kubernetes Service 用意在於將背後實際運作的細節隱藏、抽象化，讓開發者在撰寫程式時只需透過該服務的 DNS 便能查詢到該服務的 Cluster IP，如此不僅能降低程式複雜度，也可以確保運作過程中能正常存取服務，意即確保大部分情況下的服務高可用性。

在少數情況下 (比如說 [AKKA Cluster](https://charithe.github.io/akka-cluster-on-kubernetes.html)) 的確需要讓 Pod 可以知道彼此位址，仍然建議透過 Service 取得 Pod IP (非 Cluster IP)。只要在設定 Service 時指定 `clusterIP: None`，DNS 查詢時 Kubernetes DNS 會找出正在運作 Pod 並將回傳其 IP 位址。如此一來，即便該 Pod 意外重啟、刪除，仍然有辦法透過 Service 找到新的對口 Pod 存取服務。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-example
spec:
  clusterIP: None
  ...
```

-   叢集擴展導致效能降低

當叢集擴展時，Service 對應到 Pod 數量也可能隨之增加，間接使得 iptables 規則增加，最糟糕的情況會是使用者請求走到最後一條規則才送到 Pod 處理，導致請求時間(延遲)拉長。另外，隨著叢集不斷擴展、增加運算節點，節點有可能被配置在不同資料中心，致使節點間延遲升高。

## 

解法1: Sidecar Container [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E8%A7%A3%E6%B3%951-sidecar-container)

經常存取的服務改用 [Sidecar Container Pattern (16 頁)](https://www.slideshare.net/Docker/slideshare-burns)，放置在同一個 Pod 裡面如此便能透過 127.0.0.1 直接存取。

-   優點:
    -   延遲低 (同個 Pod 內的容器都處在同個 network namespace 及運算節點上)
    -   未來擴展性較佳
-   缺點:
    -   根據不同應用情境，必須適當修改整體系統架構，建議農閒期間進行(?)

## 

解法2: DNS Load Balancing [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E8%A7%A3%E6%B3%952-dns-load-balancing)

透過新增多筆 DNS A record 指向到多群 Kubernetes，避免單一叢集過多運算節點。

-   優點:
    -   立竿見影 (新增 DNS 紀錄即可)
-   缺點:
    -   Operation 負擔大，需要同時處理兩個叢集的資訊同步 (Kubernetes Federation 穩定後可能會有所改善)

# 

參考連結 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/#%E5%8F%83%E8%80%83%E9%80%A3%E7%B5%90)

-   [https://kubernetes.io/docs/user-guide/services/](https://kubernetes.io/docs/user-guide/services/)
-   [http://charithe.github.io/akka-cluster-on-kubernetes.html](https://charithe.github.io/akka-cluster-on-kubernetes.html)