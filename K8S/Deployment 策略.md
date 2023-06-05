### Canary Deployment 金絲雀部署

Canary Deployment（金絲雀部屬）是一種軟體部署發布策略/模式(deployment/release strategy/pattern)。

![礦工與金絲雀](https://www.birdsoutsidemywindow.org/wp-content/uploads/2016/10/canary_with_miner.jpg)  
  

Canary Deployment的名稱是源自於礦工進入礦坑工作前會先利用金絲雀(Canary)測試礦坑是否有毒氣的做法。金絲雀比人對毒氣敏感，所以金絲雀中毒代表有毒氣的警示。Canary Deployment的小部分先行使用者即如同金絲雀般用來測試新版本程式在生產環境中是否穩定。

  

Canary Deployment是種階段式部屬模式。做法是將新版本應用程式先部署到一小部分的生產環境，讓少數使用者可先實際測試新的功能並提供回饋，確定這小部分的佈署都沒問題後才將新的應用程式部署到全部的生產環境。好處是若新版本有問題時僅會影響到小部分使用者，減少對整個生產環境的衝擊。

部署新應用程式前。

```
                  |
                  |
                  v
+------------------------------------+
|           Load Balancer            |
+--+-------+------+-------+-------+--+
   |       |      |       |       |
   |       |      |       |       |
   |       |      |       |       |
   v       v      v       v       v
+----+  +----+  +----+  +----+  +----+
| V1 |  | V1 |  | V1 |  | V1 |  | V1 |
+----+  +----+  +----+  +----+  +----+
```

  

金絲雀部署新應用程式到少部分生產環境。

```
                  |
                  |
                  v
+------------------------------------+
|           Load Balancer            |
+--+-------+------+-------+-------+--+
   |       |      |       |       |
   |       |      |       |       |
   |       |      |       |       |
   v       v      v       v       v
+----+  +----+  +----+  +----+  +----+
| V2 |  | V1 |  | V1 |  | V1 |  | V1 |
+----+  +----+  +----+  +----+  +----+
   ^
   |
Canary Deployment
```

  

金絲雀部署沒問題後再部屬到全部的環境。

```
                  |
                  |
                  v
+------------------------------------+
|           Load Balancer            |
+--+-------+------+-------+-------+--+
   |       |      |       |       |
   |       |      |       |       |
   |       |      |       |       |
   v       v      v       v       v
+----+  +----+  +----+  +----+  +----+
| V2 |  | V2 |  | V2 |  | V2 |  | V2 |
+----+  +----+  +----+  +----+  +----+
   ^       ^      ^       ^       ^
   |       |      |       |       |
```

  

-   [Intro to deployment strategies: blue-green, canary, and more](https://dev.to/mostlyjason/intro-to-deployment-strategies-blue-green-canary-and-more-3a3)
-   [[好文分享]應用部署的六種策略](https://blog.marsen.me/2018/01/07/2018/six_strategies_for_application_deployment/)