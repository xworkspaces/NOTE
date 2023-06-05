前言 [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E5%89%8D%E8%A8%80)

為因應環境、需求改變，工程師會替 Application 不斷增添新功能。然而這樣的做法卻會讓 Application 逐漸「特化」導致最後無法普遍應用在異質環境。 顯然隨時間演進會引發幾種問題:

1.  Application 越趨複雜、難以偵錯，最後變成巨型單體 (monolith)
2.  Git tree 出現許多分枝，管理困難
3.  容器映像檔、程式碼重用性 (reusability) 降低

但正如`微服務架構`的概念一樣，適當的將主應用功能與新功能進行解耦，便能`提高重用性`與`降低複雜度`。

以下將會為各位介紹邊車模式如何解決上述問題

邊車模式 [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E9%82%8A%E8%BB%8A%E6%A8%A1%E5%BC%8F)

不知道各位是否有印象，在看美劇或電影時常會看到有人騎哈雷時為了多載人會在旁邊外掛一輛車

![sidecar](https://tachingchen.com/img/desigining-distributed-systems/sidecar.jpg)

整輛車的主體是位於中心的`哈雷「負責前進」`，而旁邊的`側車「增加載物、載人的可能性」`。

這正是邊車模式的概念來源:

> 透過其他容器來「增添、擴展」主要容器的功能性

作為 Single-Node Pattern 的第一個設計模式，邊車模式的架構非常簡單

![[Pasted image 20230201141859.png]]

它將數個容器 (原文寫兩個，但實際情況可能為多個) 置於同個 Container Group 內 (Kubernetes 內稱為 Pod)，讓容器能夠一同分配 (coscheduled) 到同個運算節點上。 由於容器位處在同個 Pod 內，彼此間會共享 disk, network namespace 等等資源。而容器則被分成`應用容器`與`邊車容器`兩種，以下針對兩種容器分別作介紹。

容器分類 [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E5%AE%B9%E5%99%A8%E5%88%86%E9%A1%9E)

應用容器 (Application Container) [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E6%87%89%E7%94%A8%E5%AE%B9%E5%99%A8-application-container)

> Core logic for the application

通常個人暱稱此類容器為`主應用`，原因在於該容器是`負責主要業務邏輯之容器`。

除一開始就採用微服務概念進行開發外，否則通常主應用會有幾種常見狀況:

1.  Codebase 龐大且難以增加新功能
2.  可以移植進容器但已無人維護的 Legacy 服務

所以當主應用因故無法進行再次開發時，透過增加`邊車容器`增加功能性就是個不錯的可行手段。

邊車容器 (Sidecar Container) [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E9%82%8A%E8%BB%8A%E5%AE%B9%E5%99%A8-sidecar-container)

> Augment and improve the application container

個人暱稱此類容器為`輔應用`，因為它`透過不包含在主容器的功能，來擴增及改善主應用容器的功能性`。

邊車容器本身並不負責主要業務邏輯，因此會盡量`提高通用性`使其能夠搭配不同的應用容器，減少維護不同容器映像檔的成本。 但通用性高低僅是作為參考並非鐵則，實際上仍需視現場狀況而定。

-   高通用性
    -   透過`組合`搭配不同主應用實現多種可用性
    -   受到通用性影響，少數情況可能無法完全發揮作用
-   低通用性
    -   輔應用只對應單個主應用，可針對主應用客製化
    -   過多的容器會造成管理及開發上的困擾
    -   耦合性高

如何設計邊車容器以提高「模組性」與「可重用性」 [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E5%A6%82%E4%BD%95%E8%A8%AD%E8%A8%88%E9%82%8A%E8%BB%8A%E5%AE%B9%E5%99%A8%E4%BB%A5%E6%8F%90%E9%AB%98%E6%A8%A1%E7%B5%84%E6%80%A7%E8%88%87%E5%8F%AF%E9%87%8D%E7%94%A8%E6%80%A7)

文章內提到三種方式來提高「模組性」與「可重用性」:

-   Parameterizing your containers
-   Creating the API surface of your container
-   Documenting the operation of your container

參數化容器 (Parameterized Containers) [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E5%8F%83%E6%95%B8%E5%8C%96%E5%AE%B9%E5%99%A8-parameterized-containers)

顧名思義就是將容器可調整的變數參數化，讓第三者使用時能夠根據不同場景進行調整。

常見的參數化方式有兩種

-   CLI flag
-   環境變數

兩種方式都可以，但個人偏好採用`環境變數`的方式進行參數調整，因為較為簡單、明瞭。

以後面應用範例的 YAML 作為示範 ([deploy.yaml](https://github.com/life1347/designing-distributed-systems-sample/blob/5911d1bebb16eeb3af0b0da0ce990651f293f8ff/sidecar-pattern/ssl-termination-proxy/deploy.yaml#L33-L39))， 透過修改環境變數，我們就可以將 proxy 的後方服務進行調整而無需更改映像檔本身。

```yaml
- name: ssl-termination-proxy
  image: tachingchen/nginx-ssl-proxy
  env:
  - name: ENABLE_SSL
    value: "true"
  - name: TARGET_SERVICE
    value: "127.0.0.1:8080"
```

制定容器 API 介面 (Define Each Container’s API) [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E5%88%B6%E5%AE%9A%E5%AE%B9%E5%99%A8-api-%E4%BB%8B%E9%9D%A2-define-each-containers-api)

不管在營運過程中或是平常運作，容器本身都需要提供對外 API 介面以供外部存取、監控。比方說常用的 `/healthz`，只要容器提供該 API 就能立刻納入叢集的監控環節中。

一份定義完整的`服務契約 (API Service Contract)`，能讓開發者理解該如何呼叫該容器，也可以在當介面出現異動時做出對應的程式修正。 值得注意的是，任何的 API 改動必須考慮到`向後兼容性 (backward compatibility)`，這是由於叢集運作中很可能發生不同版本的容器同時存在的狀況，導致服務出現異常。

以 REST API 為例，若需要改變 API (breaking API change) 會建議加入`版本號 (Versioning)`的設計。

```bash
https://w.x.y.z/api/v1.0/search
https://w.x.y.z/api/v2.0/search
```

如此一來便可確保進行版本升級時仍然能夠維持舊有服務的可靠性。

撰寫容器相關文件 (Documenting Your Containers) [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E6%92%B0%E5%AF%AB%E5%AE%B9%E5%99%A8%E7%9B%B8%E9%97%9C%E6%96%87%E4%BB%B6-documenting-your-containers)

前面兩者主要著重在提高可用性以及清楚的 API 介面能夠避免服務異常，第三點則是注重在撰寫容器的文件部分。

原文內的文件係指該容器的 Dockerfile，透過註解標明該容器的相關資訊，比如説:

```dockerfile
EXPOSE 8080
```

若容器同時聽很多 port，只看到這麼一行可能會無法知道是屬於哪個服務使用，加入`註解`或`LABEL`都能增加 Dockerfile 的可讀性。

```dockerfile
# Main web server runs on port 8080
EXPOSE 8080

LABEL "org.label-shcema.version"="1.0.3"
```

另外，也可以利用`ENV`來為容器設定環境變數值，比方說:

```dockerfile
ENV PROXY_PORT 8080
```

接下來會示範一個基本常見的邊車模式的應用場景

應用範例: 為服務增加 HTTPS (SSL Termination Proxy) [](https://tachingchen.com/tw/blog/desigining-distributed-systems-the-sidecar-pattern-concept/#%E6%87%89%E7%94%A8%E7%AF%84%E4%BE%8B-%E7%82%BA%E6%9C%8D%E5%8B%99%E5%A2%9E%E5%8A%A0-https-ssl-termination-proxy)

假設我們有個已經無人維護 legacy web server 且僅支援 HTTP 協定，由於安全性的需求必須要改採 HTTPS。在不更動程式的原則下我們便可以透過`邊車模式` 來為該服務增加 HTTPS 功能。

-   Github Repo: [https://github.com/life1347/designing-distributed-systems-sample](https://github.com/life1347/designing-distributed-systems-sample)

![[Pasted image 20230201141917.png]]

首先，我們先產生所需要的 ssl cert files

```bash
$ openssl dhparam -out dhparam.pem 512
$ openssl req -x509 -nodes -days 365 -newkey rsa:512 -keyout nginx.key -out nginx.crt
```

建立 secret 跟 deployment

```bash
$ kubectl create secret generic ssl-crt --from-file=dhparam.pem --from-file=nginx.crt--from-file=nginx.key
$ kubectl apply -f sidecar-pattern/ssl-termination-proxy/deploy.yaml
```

建立 port-forward

```bash
$ kubectl port-forward <ssl-termination-proxy-pod-name> 8080:443
```

到瀏覽器造訪 `[https://127.0.0.1:8080](https://127.0.0.1:8080/)` 就能看到我們成功加上 HTTPS 連線摟!

![ssl web page](https://tachingchen.com/img/desigining-distributed-systems/web-page-min.png)