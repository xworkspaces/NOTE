# 

前言 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-2/#%E5%89%8D%E8%A8%80)

[前篇文章](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-1/)介紹 Service 不同存取路徑間的差異，這次我們來討論 Service 和 Label 之間是如何互相影響的。

# 

Service 與 Label Selector 共舞 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-2/#service-%E8%88%87-label-selector-%E5%85%B1%E8%88%9E)

> We encourage use of a unique collection of labels rather than a single unique label value since the additional attributes are generally needed — [bgrant0607](https://github.com/kubernetes/kubernetes/issues/36859#issuecomment-262022027)

在 Kubernetes 最佳實踐中，Pod 本身會帶著許多不同標籤 (Label) 來辨別其實際用途。透過賦予一組唯一的標籤組合 (a unique collection of labels)，不僅能擁有更精確的粒度 (granularity) 以外，也能避免操作上出現異常。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: python-http-server
spec:
  replicas: 1
  selector:
    matchLabels:
      env: python
      svc: http-server
  template:
    metadata:
      labels:
        env: python
        svc: http-server
    spec:
      containers:
      - name: http-server
        image: trinitronx/python-simplehttpserver
        command:
          - python
        args:
          - -m
          - SimpleHTTPServer
          - "80"
        ports:
        - containerPort: 80
          protocol: TCP
```

從上面 `Deployment` 範例可以看到，其所控管的 Pod 會帶有 `env: python` 及 `svc: http-server` 兩種標籤。更甚者會攜帶所屬的環境標籤進行更細緻的區分，比方說 `stage: production` 或 `stage: canary`。

進到叢集內的流量會根據 Service 的 `spec.selector` 流至對應的 Pod 內。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-example
  namespace: default
spec:
  selector:
    svc: http-server
  ports:
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
```

以上面例子作為舉例，流量會流向全部帶有 `svc: http-server` 標籤的 Pod (示意圖如下)。

![[Pasted image 20230201143914.png]]

若要提昇操控流量的粒度時，可以透過新增額外標籤的方式，改變其流向。

```yaml
spec:
  selector:
    svc: http-server
    env: python
    ...
```

此時流量便會導向有 `svc: http-server` 且 (AND) 擁有 `env: python` 標籤的 Pod 內。

![[Pasted image 20230201143921.png]]

# 

注意要點 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-2/#%E6%B3%A8%E6%84%8F%E8%A6%81%E9%BB%9E)

-   在 Kubernetes 1.5 時，多個 Deployment 只攜帶一個標籤且彼此間重覆時，會觸發 selector overlapping 的檢測。解決辦法是增加唯一的標籤組合，此問題在 1.6+ 已透過 `Ower Reference` 的方式解決。
    -   [https://github.com/kubernetes/kubernetes/issues/30899#issuecomment-287075939](https://github.com/kubernetes/kubernetes/issues/30899#issuecomment-287075939)
    -   [https://github.com/kubernetes/kubernetes/issues/36859#issuecomment-262022027](https://github.com/kubernetes/kubernetes/issues/36859#issuecomment-262022027)

# 

參考連結 [](https://tachingchen.com/tw/blog/kubernetes-service-in-detail-2/#%E5%8F%83%E8%80%83%E9%80%A3%E7%B5%90)

-   [https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)