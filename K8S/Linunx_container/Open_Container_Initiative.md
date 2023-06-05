# Open Container Initiative(OCI)

`Container` 本身是種概念，背後有標準規範其相關設定與運作，透過標準的制定能夠讓各個 `Contaienr` 相關的解決方案能夠有更高的相容性，就譬如我們可以很輕鬆的於 `Kubernetes` 的環境內替換各種不同的 `Container` 實作

在 `Container` 的世界中，該介面就是所謂的 `Open Container Iinitiative (OCI)`

-   隸屬於 Linux Foundation 專案底下
-   2015/01/22 專案啟動
-   目標希望能夠提供基於作業系統層級的虛擬化介面
-   主要定義兩大標準
    -   Runtime Specification (運行標準)
    -   Image Specification (容器映像檔案標準)

熟悉 `Docker` 的人應該對於 `docker images` 相關指令不陌生，透過 `docker images/pull/push` 等指令能夠讓所有的使用者與開發者享受與分享各種已經建置好的虛擬化環境，減少重複打造輪子的困境與時間，帶來的好處可是說也說不完。  
此外使用者也可以透過 `docker build` 建置自己的環境來使用，這一切的操作都與所謂的 `image` 脫不了關係，如果有仔細看過 `docker images`, `docker pull` 的內容應該會觀察到一層又一層的 `layer`， 而所謂的 `Image Spec` 就是用來規範 `Image` 的格式。

而 `Runtime Spec` 則是規範如何控制所謂的 `Container`, 包含了 `Container` 生命週期的操作，如 `create/delete/start/stop` 或者是運行時期的互動，如 `attach/exec` 等，這些需要的資訊與參數都在 `Runtime Spec` 裡面去制定。

`Docker` 底層用到的相關元件都有滿足 `OCI` 的需求，所以如果今天有其他的解決方案也都遵循 `OCI` 的標準來設計，則理論上他們之間的容器映像檔案 `Container Image` 要可以交替使用而不影響功能。

# Runtime spec

`Runtime Spec` 的文件可以在 [Github](https://github.com/opencontainers/runtime-spec/blob/master/spec.md) 上參閱， `Runtime Spec` 規範的範疇有三，分別是

1.  設定檔案格式
    -   該內容基本上會根據不同平台而有不同的規範，同時也明確表示創建該平台上 `container` 所需要的一切參數。
2.  執行環境一致
    -   確保 `container` 運行期間能夠有一致的運行環境。
3.  `Container`的生命週期
    -   必須支持下列的操作行為，參考[閱覽](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md#operations)
        -   Query State
        -   Create
        -   Start
        -   Delete
        -   Kill
        -   Hooks

#### Configuration

講到設定檔案的部分，我們可以稍微看一下到底在 `Linux` 上創建一個 `Container` 會需要哪些[設定](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)。

首先是資源隔離，這部分是仰賴 `Linux Kernel` 內的 `namespace` 來完成的，藉由資源隔離可以達到讓 `namespace` 內部與 `host` 本身互相使用該資源卻不衝突的優點。

根據上述的規範，會需要用到的 `namespace` 如下。

-   pid
    -   process 相關的隔離。在 `docker container` 的世界內，用 `ps auxw` 能夠看到的 `process` 比想像中的少?
-   network
    -   精準來說是 `Network Stack` 的隔離，體現出來簡單的範例就是 `網卡`,`iptables 規則` 等諸多與網路有關的資源。
-   ipc
    -   `Inter Process Communication` 意味者 `namespace` 內的 `process` 只能跟同 `namespace` 內的其他 `process` 透過 `IPC` 的機制溝通
-   mount
    -   常常使用 `mount` 指令的人就可以發現到 `docker container` 內與外面看到的結果不同也是因為該 `namespace` 將資源區隔了
-   uts
    -   `UNIX Timesharing System` 主要負責的就是 `hostname`, 包含給 `NIS` 使用的 `domain name`
-   user
    -   使用者與群組清單，透過 `id` 指令等對應到的 `UID/GID` 都是獨立的
-   cgroup
    -   `Control Group`，主要是跟資源存取限制有關，譬如 `CPU`，`Memory` 等資源的存取上限。再 `docker container` 的使用中就可以透過該機制設定一些 `soft, hard/limit memory` 等設定來避免 `OOM` 或是減少 `CPU` 存取。

每個 `namespace` 都有自己的設定檔案，因此 `Configuration` 在設定的時候則必須要仔細的設定每個 `namespace` 的資訊。 譬如

```json
        {
            "type": "pid",
            "path": "/proc/1234/ns/pid"
        },
        {
            "type": "network",
            "path": "/var/run/netns/neta"
        },
        {
            "type": "mount"
        },
        {
            "type": "ipc"
        },
        {
            "type": "uts"
        },
        {
            "type": "user"
        },
        {
            "type": "cgroup"
        }
```

除了 `namespace` 之外，`container` 內部也還有許多的資源需要設定，譬如 `Devices`, `Sysctl`, `Seccomp(SECure COMPuting)`，參考[官方文件](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)。

到這邊對於 `Runtime Spec` 已經有一些基本的認知，用來定義不同平台上如何管理 `Container`，為了管理這些 `Container` 必須要有一些相關的設定來描述該 `Container` 的資訊，而這些設定本身並非跨平台，需要針對每個平台去獨立設計來描述如何產生一個符合需求的 `Container`.

# Image spec

接下來來看一下到底所謂的 `Image` 是什麼，我們常用的 `Docker pull/push/build/images` 所產生的 `image` 檔案是怎麼被定義的。

下圖片清楚明瞭的說明到底 `Image` 要處理什麼事情
![[Pasted image 20230109112357.png]]
reference : https://github.com/opencontainers/image-spec/blob/main/spec.md

一個應用程式包裝成 `Container` 後，整個 `Image Layer` 處理了下列事情

1.  `Layer`: 相關的檔案系統配置，檔案的位置/內容/權限
2.  `Image Index`: 用來描述該 `Image` 的檔案
3.  `Configuration`: 應用程式相關的設定檔案，包含了使用的參數，用到的環境變數等

如果仔細去研究的話，其實總共會有[7大相關的類別](https://github.com/opencontainers/image-spec/blob/master/spec.md#understanding-the-specification)被標準化，分別是

1. -   Image Manifest - a document describing the components that make up a container image
2. -   Image Index - an annotated index of image manifests
3. -   Image Layout - a filesystem layout representing the contents of an image
4. -   Filesystem Layer - a changeset that describes a container's filesystem
5. -   Image Configuration - a document determining layer ordering and configuration of the image suitable for translation into a runtime bundle
6. -   Conversion - a document describing how this translation should occur
7. -   Descriptor - a reference that describes the type, metadata and content address of referenced content

稍微留意`Docker Images` 的話就會觀察到有很多個 `Layer` 的產生，每個 `Layer` 都有獨立的 `Image ID`，這邊可以參閱 [Filesystem Layer](https://github.com/opencontainers/image-spec/blob/master/layer.md) 來了解更多[[Image_Layer_Filesystem_changeset]]，到底這些 `Layer` 代表甚麼，以及底層到底怎麼實現。


