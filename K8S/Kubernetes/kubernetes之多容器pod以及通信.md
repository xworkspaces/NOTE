容器經常是為了解決單一的,窄範圍的問題,比如說微服務.然而現實中,一些複雜問題的完成往往需要多個容器.這裡我們討論一下如何把多個容器放在同一個pod里以及容器間的通信

## 什麼是pod

pod是kubernetes裡的一個基本概念,可能我們從一開始接觸kubernetes的時候就開始接觸pod,並被灌輸pod是kubernetes裡最小的不可分割的工作單元,這裡再從多容器的角度對其進行一些基本闡釋.

簡言之,pod是kubernetes可以部署和管理的最小單元,換言之也就是說如果你想要運行一個容器,你先要為這個容器創建一個pod.同時,一個pod也可以包含多個容器,之所以多個容器包含在一個pod裡,前面講到過的初始容器為普通應用容器準備環境是一種使用場景,往往由於業務上的緊密耦合,一組容器也需要放在同一個pod裡.

> 需要注意的是,以上場景都非必須把不同的容器放在同一個pod裡,但是這樣往往更便於管理,甚至後面會講到的,緊密耦合的業務容器放置在同一個容器里通信效率更高.具體怎麼使用還要看實際情況,綜合權衡.

## kubernetes為什麼使用pod作為最小單元,而不是container

直接部署一個容器看起來更簡單,但是這裡也有更好的原因為什麼在容器基礎上抽像一層.容器是一個存在的實體,並指向一個具體的事物.這個具體的事物可能是一個docker容器,但也可能是一個rtk容器,或者一個Virtlet管理的虛擬機.它們有不同的要求.

更深層的原因是為了管理容器,kubernetes需要更多的信息,比如重啟策略,它定義了容器終止後要採取的策略;或者是一個可用性探針,從應用程序的角度去探測是否一個進程還存活著.

基於這些原因,kubernetes架構師決定使用一個新的實體,也就是pod,而不是重載容器的信息添加更多屬性,用來在邏輯上包裝一個或者多個容器的管理所需要的信息

## kubernetes為什麼允許一個pod裡有多個容器

pod裡的容器運行在一個邏輯上的"主機"上,它們使用相同的網絡名稱空間(也就是說,同一pod裡的容器使用相同的ip和相同的端口段區間)和相同的IPC名稱空間.它們也可以共享存儲卷.這些特性使它們可以更有效的通信.並且pod可以使你把緊密耦合的應用容器作為一個單元來管理.

因此一個應用如果需要多個運行在同一主機上的容器時,為什麼為把它們放在同一個容器裡呢?首先,這樣何故違反了一個容器只負責一個應用的原則.這點非常重要,如果我們把多個應用放在同一個容器裡,這將使解決問題變得非常麻煩因為它們的日誌記錄混合在了一起,並且它們的生命週期也很難管理.因此一個應用使用多個容器將更簡單,更透明,並且使應用依賴解偶.並且粒度更小的容器更便於不同的開發團隊共享和復用.

> 這裡說到為了解偶把應用分別放在不同容器裡,前面我們也強調為了便於管理管緊耦合的應用把它們的容器放在同一個pod裡.一會強調耦合,一個強調解偶看似矛盾,實際上普遍存在,高內聚低耦合是我們的追求,然而一個應用的業務邏輯模塊不可能完全完獨立不存在耦合,這就需要我們從實際上來考量,做出決策.

## 多容器pod的使用場景舉例

-   **Sidecar containers** "幫助"主容器,比如日誌文件監視器.一個日誌監視器構建完成以後,可以由不同的應用來使用.另一個示例是`sidecar 容器`為主容器加載文件和運行需要的數據
    
-   代理,橋接和適配器使主容器與外部世界聯通.比如Apache http服務器或者nginx可承載靜態文件.也可以做為一個web的反向代理服務器.
    

你可以使用一個pod來承載一個多層應用(比如wordpress),但是更建議使用不同的pod來承載不同的層,因這這樣你可以為每一個層單獨擴容並且把它們分佈到集群的不同節點上.

## 同一pod間的容器通信

把不同的容器放在同一個pod裡讓它們之間的通信變得非常直接和簡單,它們可以通過以下幾種方法達到通信目的.

### 同一pod內的容器共識存儲卷

你可以使用一個共享的存儲卷來簡單高效的地在容器間共享數據.大多數情況下,使用一個共享目錄在同一pod裡的不同容器間共享數據就夠了.

一個標準的同一pod內容器共享存儲卷的用例是一個容器往共享存儲卷裡寫入數據,其它的則從共享目錄裡讀取數據

```yml
apiVersion: v1
kind: Pod
metadata:
  name: mc1
spec:
  volumes:
  - name: html
    emptyDir: {}
  containers:
  - name: 1st
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  - name: 2nd
    image: debian
    volumeMounts:
    - name: html
      mountPath: /html
    command: ["/bin/sh", "-c"]
    args:
      - while true; do
          date >> /html/index.html;
          sleep 1;
        done
```

這個示例裡我們定義了一個存儲卷叫作html,它是`emptyDir`類型的.當一個pod在一個節點上創建的時候,它就被分配,只要pod一直運行在這個節點上它就一直存在(emptyDir生命週期和pod相同).就像它的名字所暗示的一樣,它是一個空的目錄,第一個容器運行了一個nginx server並且把它掛載到`/usr/share/nginx/html`,第二個容器使用了一個Debian鏡像並把emptyDir掛載到`/html`.每一秒鐘,第二個容器就會把當前日期寫入到`index.html`裡,它位於共享存儲卷裡.當用戶發起一個http請求,nginx就會讀取它並響應給用戶.

![[Pasted image 20230201140938.png]]

你可以通過把nginx的端口暴露出去然後通過瀏覽器查看或者查看共享目錄裡的文件來檢測以上是否有效果

```bash
 kubectl exec mc1 -c 1st -- /bin/cat /usr/share/nginx/html/index.html
 ...
 Fri Aug 25 18:36:06 UTC 2017
 
 $ kubectl exec mc1 -c 2nd -- /bin/cat /html/index.html
 ...
 Fri Aug 25 18:36:06 UTC 2017
 Fri Aug 25 18:36:07 UTC 2017
```

## 進程間通信(IPC)

同一個pod裡的容器共享IPC名稱空間,這就意味著他們可以通過進程間通信的手段來進行通信,比如使用`SystemV semaphores`或者`POSIX`共享內存

以下示例中,我們定義一個包含了兩個容器的pod.它們使用相同的docker鏡像,第一個容器是一個生產者,創建一個標準的linux消息隊列,寫一些隨機的消息,然後寫一個特殊的退出消息.第二個容器是一個消費者,打開同一個消息隊列來讀取數據直到讀到退出消息,我們把重啟策略設置為`Never`,這樣當兩個pod都中止的時候pod就會停止.

```yml
apiVersion: v1
kind: Pod
metadata:
  name: mc2
spec:
  containers:
  - name: producer
    image: allingeek/ch6_ipc
    command: ["./ipc", "-producer"]
  - name: consumer
    image: allingeek/ch6_ipc
    command: ["./ipc", "-consumer"]
  restartPolicy: Never
```

然後通過kubectl create來創建pod,用下面命令來查看狀態

```yml
$ kubectl get pods --show-all -w
NAME      READY     STATUS              RESTARTS  AGE
mc2       0/2       Pending             0         0s
mc2       0/2       ContainerCreating   0         0s
mc2       0/2       Completed           0         29
```

這時候你可以檢測每一個容器的日誌來檢測第二個隊列是否消費了第一個隊列生產的所有消息包括退出消息

```yml
$ kubectl logs mc2 -c producer
...
Produced: f4
Produced: 1d
Produced: 9e
Produced: 27
$ kubectl logs mc2 -c consumer
...
Consumed: f4
Consumed: 1d
Consumed: 9e
Consumed: 27
Consumed: done
```

![[Pasted image 20230201141002.png]]

這裡面存在一個問題,就是生產者所在容器要早於消費者所在容器,我們必須處理容器的啟動順序問題

### 容器的依賴關係和啟動順序

當前,同一個pod裡的所有容器都是並行啟動並且沒有辦法確定哪一個容器必須早於哪一個容器啟動.就上面的IPC示例,我們不能總保證第一個容器早於第二個容器啟動.這時候就需要使用到初始容器(init container)來保證啟動順序,關於初始容器,可以查看[官方文檔](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/),本系列也對其進行了翻譯,以便查看.

### 同一pod的容器間網絡通信

同一pod下的容器使用相同的網絡名稱空間,這就意味著他們可以通過'localhost'來進行通信,它們共享同一個Ip和相同的端口空間

### 同一個pod暴露多個容器

通常pod裡的容器監聽不同的端口,想要被外部訪問都需要暴露出去.你可以通過在一個服務裡暴露多個端口或者使用不同的服務來暴露不同的端口來實現