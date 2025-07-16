# 稳定币管理套件

稳定币管理平台包括三个定制开发组件

- 管理员多签工具
- 助记词分片管理工具
- 托管账户服务

## 管理员多签工具

一个定制开发的Web页面工具，稳定币管理员可以通过这个Web页面实现对稳定币合约的安全管控。

```mermaid
graph TD
    subgraph VPN ["VPN 网络环境"]
        subgraph "客户端 (Client Side)"
            Browser["🌐 客户端浏览器"]
            subgraph "Browser"
                WebApp["💻 Web 页面 (稳定币管理)"]
                MetaMask["🦊 MetaMask 插件"]
                TronLink["🔗 TronLink 插件"]
            end
        end

        subgraph "服务端 (Server Side)"
            WebServer["🖥️ 应用服务器 (Web Server)"]
            subgraph "私有区块链节点"
                EthRPC["[以太坊私有 RPC 节点]"]
                TronRPC["[波场私有 RPC 节点]"]
            end
        end

        Browser -- "加载管理页面 (1)" --> WebServer
        WebApp -- "请求签名 (多签) (2)" --> MetaMask
        WebApp -- "请求签名 (多签) (2)" --> TronLink
        MetaMask -- "返回签名数据 (3)" --> WebApp
        TronLink -- "返回签名数据 (3)" --> WebApp
        WebApp -- "发送已签名的交易到服务端 (4)" --> WebServer
        WebServer -- "广播交易 (5)" --> EthRPC
        WebServer -- "广播交易 (5)" --> TronRPC
    end

    style VPN fill:#f0f7ff,stroke:#b3d1ff,stroke-width:2px,stroke-dasharray: 5 5
```

[source code](https://github.com/bitlistfun/usdv/tree/main/web)


## 助记词分片管理工具

分为客户端和服务端，客户端由管理人员启动运行，服务端由运维人员启动运行。

分为两个过程：
1. 分发过程：分发密钥助记词（有且只有一次），服务端启动后，客户端需要在一个很短的时间窗口启动进行密钥助记词获取，每个客户端都需要保存助记词。默认情况下，密钥助记词分发给三个管理人员
2. 启动过程：服务每次启动后（允许多次），都需要等待所有客户端启动并输入正确的助记词，服务端通过助记词生成密钥对，生成的密钥对用于加密和解密数据。

```mermaid
graph TD
    subgraph KMC服务 - 密钥管理逻辑
        direction LR

        A[开始] --> B{运维启动KMC服务端};

        subgraph "1. 密钥分发过程 (有且只有一次)"
            direction TB
            B -- 首次启动/分发模式 --> C{服务端进入短时间窗口};
            C -- 窗口开启 --> D["管理人员启动KMC客户端 1"];
            C --> E["管理人员启动KMC客户端 2"];
            C --> F["管理人员启动KMC客户端 3"];

            D --> G[客户端 1 获取密钥助记词];
            E --> H[客户端 2 获取密钥助记词];
            F --> I[客户端 3 获取密钥助记词];

            G --> J[客户端 1 保存助记词];
            H --> K[客户端 2 保存助记词];
            I --> L[客户端 3 保存助记词];

            J & K & L --> M{是否所有客户端已保存助记词?};
            M -- 是 --> N[密钥助记词分发完成];
            M -- 否 --> O[分发失败: 客户端未全部完成];

            C -- 窗口关闭/超时 --> P[分发失败: 窗口已过];
        end

        subgraph "2. KMC启动过程 (每次启动)"
            direction TB
            B -- 后续启动/运行模式 --> Q{服务端等待客户端启动};
            Q --> R["管理人员启动KMC客户端 1"];
            Q --> S["管理人员启动KMC客户端 2"];
            Q --> T["管理人员启动KMC客户端 3"];

            R --> U[客户端 1 输入助记词];
            S --> V[客户端 2 输入助记词];
            T --> W[客户端 3 输入助记词];

            U & V & W --> X{所有客户端助记词正确?};
            X -- 是 --> Y[服务端通过助记词生成密钥对];
            Y --> Z[密钥对用于数据加密/解密];
            Z --> AA[KMC服务正常运行];
            X -- 否 --> AB[助记词错误，等待重试或失败];
        end

        N --> AA;
        P --> AC[结束];
        O --> AC;
        AA --> AC;
        AB --> AC;
    end
```

[source code](https://github.com/bitlistfun/usdv/tree/main/kmc/cmd)

## 托管账户

每个客户端钱包和托管钱包存在一一对应关系，提供托管账户的创建和管理服务。托管账户的所有数据加密存储在数据库。

```mermaid
graph TD
    A[用户通过浏览器 + 钱包插件发起托管账户管理操作] --> B{是否为创建托管账户};
    B -- 是 --> C[客户端生成托管账户公钥];
    C --> D[客户端请求KMC服务端创建托管账户];
    D --> E{KMC服务端是否收到创建请求};
    E -- 是 --> F[KMC服务端生成托管账户私钥];
    F --> G[KMC服务端加密存储托管账户私钥和公钥到数据库];
    G --> H[KMC服务端返回托管账户创建成功响应给客户端];
    H --> I[客户端存储托管账户公钥并与当前钱包关联];
    I --> J{托管账户创建完成};
    B -- 否 (管理) --> K[客户端请求KMC服务端执行操作];
    K --> L{KMC服务端是否收到操作请求};
    L -- 是 --> M[KMC服务端查询数据库获取托管账户加密后的私钥和公钥];
    M --> N[KMC服务端对私钥进行解密];
    N --> O{操作类型是查询余额};
    O -- 是 --> P[KMC服务端调用以太坊或波场RPC节点查询托管账户余额];
    P --> Q[KMC服务端返回托管账户余额给客户端];
    Q --> R{托管账户管理完成};
    O -- 否 (操作是交易) --> S[客户端构建交易，并使用客户端钱包对交易进行部分签名];
    S --> T[客户端将部分签名的交易发送给KMC服务端];
    T --> U{KMC服务端是否收到部分签名的交易};
    U -- 是 --> V[KMC服务端使用托管账户私钥对交易进行签名];
    V --> W[KMC服务端将完整签名的交易广播到以太坊或波场网络];
    W --> X{交易是否广播成功};
    X -- 是 --> Y[KMC服务端更新数据库中托管账户交易记录];
    Y --> Z[KMC服务端返回交易广播成功响应给客户端];
    Z --> R;
    X -- 否 --> AA[KMC服务端返回交易广播失败响应给客户端];
    AA --> R;
    L -- 否 --> AB[KMC服务端返回操作失败响应给客户端];
    AB --> R;
    E -- 否 --> AC[KMC服务端返回创建失败响应给客户端];
    AC --> J;
```

[source code](https://github.com/bitlistfun/usdv/tree/main/kmc)
