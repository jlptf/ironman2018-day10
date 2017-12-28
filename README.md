## Day 10 - 建構組件

### 本日共賞

* k8s 中的建構組件 (Building Blocks)
* Pods, ReplicaSets, Deployments
* Label
* Namespace

### 希望你知道
* [k8s 架構](https://ithelp.ithome.com.tw/articles/10192409)
* [運行 minikube](https://ithelp.ithome.com.tw/articles/10193237)
* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### k8s 中的建構組件

在 k8s 中有著各式各樣不同的物件 (Pod, Service, ...)，而這些物件在 k8s 中描述著

* 運行何種應用程式 (application) 且在哪運行 (node, namespace, ...)
* 資源使用狀態 (resource limits/occupied)
* 應用程式運行策略，例如是否要重啟 (restart policy)

簡單來說，使用者的任務就是清楚地描述每個物件“想要”的樣子。當 k8s 接收到描述的物件後，便會“努力”的將物件調整至使用者想要的樣子

> “想要”：表示不一定能達到。原因很多，可能是因為資源不足等等...
>
> “努力”：表示 k8s 會不斷嘗試調整狀態，同樣表示不一定能達到。原因很多，可能是因為權限不足無法取得映像檔等等...

那麼使用者要如何描述物件呢？答案是透過攥寫 spec。而在 k8s 中，我們可以建立 .yaml 檔案來攥寫 spec，也就是描述物件。底下是一個描述 Deployment 物件，名為 simple.yaml 的簡單範例

```
# simple.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  template:
    metadata:
      labels: 
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
``` 

> 還記得 [Day 9 - yaml](https://ithelp.ithome.com.tw/articles/10193509) 的說明嗎？

以下針對上述 simple.yaml 檔案進行說明

* `apiVerion `：通知 api server 要存取的 api 版本
* `kind`：物件種類。範例中為 Deployment 物件
* `metadata`：用來描述目前物件的資料，另如 name: nginx
* `spec`：這裡就是描述 “想要” 的樣子。就本例來說，就是描述 Deployment 這個物件 ”想要“ 的樣子，包含
  * 運行三個 Pod (replicas: 3)
  * 描述每個 Pod “想要“ 的樣子 (spec.template)
    * 標記為 app: nginx (labels)
    * 使用 nignx 映像檔 (image: nginx)
    * 使用 80 Port (containerPort: 80)

有了 yaml 檔後，我們可以透過 kubectl 執行下列命令進行部署

```bash
$ kubectl apply -f simple.yaml
deployment "nginx" created
```

> 你有看出來 `nginx` 是哪裡的設定嗎？再看一下 `metadata` 吧！

由於我們並沒有指定命名空間 (namespace) ，因此 nginx 會被部署到 `default` 這個預設的命名空間內，可執行下列指令查詢

```bash
$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
nginx-75f4785b7-6fbd4   1/1       Running   0          1m
nginx-75f4785b7-g284n   1/1       Running   0          1m
nginx-75f4785b7-vd4tt   1/1       Running   0          1m
```

會發現命名為 `nginx-75f4785b7-6fbd4`, `nginx-75f4785b7-g284n` 與 `nginx-75f4785b7-vd4tt` 共三個 Pod 正在運行，即表示我們已經成功完成一次部署

> 每次物件建立會被賦予不同的名稱，因此上面的 Pod 的名稱可能會有所不同。

看到這裡，你應該會很好奇該如何連到這個運行 nignix 的 Pod。先別著急，接下來我們來討論一下 k8s 中常用的物件。

#### Pod

k8s 中最小單位的物件，也是 k8s 中部署的基本單位。你也可以把 Pod 想成是應用程式的一個 instance 。而一個 Pod 會運行一個到多個容器 (container)，而運行在同一個 Pod 中的容器會

>一個應用程式可能會有很多個 instances ，即很多個 Pods
>
>可以把應用程式想像成網站，而該網站可以在 k8s 中被部署多份 (多個 Pod)，用來達到水平擴展 (Horizontal Scaling) 以應付更多的存取需求。
> 
> ![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062EhlngoBs3W.png)

* 被安排到同一台主機
* 共享 network namespace
* 連接同一個外部儲存區 (Volume)

![](https://ithelp.ithome.com.tw/upload/images/20171224/201070621MGqhfmGwm.png)

請注意！ Pod 在特性上是屬於臨時性的 (ephemeral)，意思就是需要的時候被產生而不需要的時候就結束 (包含儲存區)。另外，Pod 本身無法自行完成複製 (Replication)、修復 (self-heal) 等等不同的功能，因此需要透過其他的控制項目 (Controller) 來達到。而常見的控制項目有 Deployment, ReplicaSets 等等。

#### ReplicaSet

ReplicaSet 最主要的功能是在確保 Pod 指定的運行數量能被達到。上例中，我們希望能夠運行三個 Pod 在 k8s 叢集中。當 ReplicaSet 偵測到這個需求時，它會去檢查目前狀態是否滿足，如果不滿足，則會嘗試將狀態調整為指定狀態。

在大多數情況下，我們不需要直接去操作 ReplicaSet，而是透過宣告 Deployment 物件。因為 Deployment 就已經包含了 ReplicaSet 的處理。

#### Deployment

Deployment 物件內可以描述 Pod 的內容 (spec) 以及想要運行的個數 (ReplicaSet)。另外，如果想要變更 Pod 的內容，也可以直接修改 Deployment 物件後再套用新內容即可。

> 更多的物件說明請參考 [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)


#### Label

在應用上，k8s 叢集中可能會運行數十到數百個物件，而 k8s 使用 Label (key-value) 來標記 k8s 物件 (不僅是 Pod )。透過適當的標記，k8s 可以區別不同物件所對應的環境。例如 `app:fronend`, `app:backend` 可區分為前端與後端應用程式，`env:dev`, `env:pro` 可以區分開發與正式環境。我們用 Pod 物件當例子

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062BX8vIwxAtQ.png)

上圖表示 k8s 內有 6 個 Pod 正在運行，而其中 Frontend 與 Backend 各佔三個。正式環境有兩個 Pod 而測試環境運行一個 Pod。

> Label 內容 (key-value) 可視環境自行定義，並無硬性規定

#### Namespace

如果有不同的群組想使用同一個 k8s 但是又不想要互相影響。又或者想要區分開發環境與正式環境 (上述例子是運行在同一個 Namespace)，這個時候可以利用 k8s 提供的 Namespace 來區隔。

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062B3DwckOYJS.png)

>如果你想要替不同部門分配 k8s 資源，利用命名空間可以很容易做到。不但可以不互相影響還可以限定每個命名空間能使用的資源。例如：限定開發部門使用 `develop` 這個命名空間，然後限制 `develop` 最多只能使用 2 GB 的記憶體以及 30% CPU 資源等等。

*建立 Namespace*

[與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502) 就有說明過如何建立一個 namespace 如下

```
$ kubectl create namespace [name]
```

`[name]` 可以自行命名，例如建立一個 `develop` 的命名空間

```bash
$ kubectl create namespace develop
namespace "develop" created
```

*取得 Namespace*

```bash
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    20h
develop       Active    1m    <===== 剛剛建立的
kube-public   Active    20h
kube-system   Active    20h
```

你會發現 `develop` 已經被建立在 k8s 中。

k8s 預設有三個 namespaces，分別是 `default`, `kube-public` 與 `kube-system`。如未特別指定，k8s 會使用 `default` 這個 namespace 當作預設的命名空間部署物件。


本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day10/](https://jlptf.github.io/ironman2018-day10/)