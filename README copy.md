配套视频教程地址：https://www.bilibili.com/video/BV1Tg411P7EB

配套文档地址：https://k8s.easydoc.net


› ClusterIP 类型的 service 只能在集群内部访问，NodePort 既可以在集群内部访问也可以通过
  nodeip 访问是吗


• 对，理解基本正确。

  - ClusterIP：
      - 只能通过集群内部访问
      - 常见方式是 Service IP:port 或 Service 名称:port
  - NodePort：
      - 仍然可以在集群内部访问
  可以把 NodePort 理解成：

  - 它本身先有一个 ClusterIP
  - 再额外在每个节点上开一个端口给外部进来

  所以对于 NodePort Service，通常同时存在两种入口：

  集群内:   ServiceIP:port
  集群外:   NodeIP:nodePort

  拿你这个例子就是：

  集群内:   10.104.242.49:8080
  集群外:   192.168.49.2:31000

  但要注意一句：

  NodePort 是“可以通过 NodeIP 访问”，不是“任何环境下都一定能从外部访问成功”。
  是否真能通，还取决于节点网络是否可达、防火墙、安全组、minikube 驱动这些因素。

  更严谨地说：

  - ClusterIP：仅集群内入口
  - NodePort：集群内入口 + 节点级入口

  如果你愿意，我可以接着把 LoadBalancer 和 Ingress 也放进这张对比表里，帮你一次性理清。


› 加上ingress 对比


• 可以，放在一起看最清楚。

  | 类型 | 访问范围 | 典型访问方式 | 是否有固定 Service IP | 是否能被集群外直接访问 | 典
  型用途 |
  |---|---|---|---|---|---|
  | ClusterIP | 仅集群内 | service-name:port / ClusterIP:port | 有 | 不行 | 服务间调用 |
  | NodePort | 集群内 + 节点入口 | service-name:port / NodeIP:NodePort | 有 | 可以，但依
  赖节点网络 | 测试、简单暴露 |
  | LoadBalancer | 集群内 + 外部负载均衡入口 | ExternalIP:port | 有 | 可以 | 云上对外暴露
  服务 |
  | Ingress | 主要面向集群外七层访问 | http://域名/path | Ingress 本身不是 Service 类型 |
  可以 | 多服务统一入口、域名路由 |

  再用一句话分别概括：

  - ClusterIP：只有集群内部能进来
  - NodePort：在每个节点开一个端口让外部进来
  - LoadBalancer：云厂商给你一个外部 IP，再转到 Service
  - Ingress：通过域名、路径、HTTPS 规则，把请求转发到不同 Service

  可以画成这样：

  1. ClusterIP
  集群内Pod -> Service(ClusterIP) -> Pod

  2. NodePort
  外部/宿主机 -> NodeIP:NodePort -> Service -> Pod

  3. LoadBalancer
  外部用户 -> LoadBalancer EXTERNAL-IP -> Service -> Pod

  4. Ingress
  外部用户 -> Ingress(域名/path)
             -> Service A -> Pod A
             -> Service B -> Pod B

  结合你的理解，Ingress 和前面几个最大的区别是：

  - ClusterIP / NodePort / LoadBalancer 是 Service 的暴露方式
  - Ingress 不是 Service 类型
  - Ingress 是一层“HTTP/HTTPS 路由规则”
  - 它通常把外部请求按域名或路径转发到后面的 Service

  一个典型生产链路是：

  用户
   -> Ingress
   -> ClusterIP Service
   -> Pod

  例如：

  api.example.com/user   -> user-service:8080
  api.example.com/order  -> order-service:8080

  所以很多时候：

  - Service 负责把流量转给 Pod
  - Ingress 负责决定“这个请求该进哪个 Service”

  如果你想，我可以下一条直接给你一张“ClusterIP、NodePort、LoadBalancer、Ingress 的记忆版
  总结表”，专门方便背。


  • | 类型 | 能否集群内访问 | 能否集群外访问 | 访问方式 | 主要用途 |
  |---|---|---|---|---|
  | ClusterIP | 可以 | 不可以 | Service名:port / ClusterIP:port | 集群内部服务调用 |
  | NodePort | 可以 | 可以 | NodeIP:NodePort | 临时对外暴露、测试 |
  | LoadBalancer | 可以 | 可以 | EXTERNAL-IP:port | 云上正式对外服务 |
  | Ingress | 间接可以 | 可以 | 域名 + 路径 | HTTP/HTTPS 统一入口 |

  记忆版：

  - ClusterIP：只给集群内部用
  - NodePort：在节点上开端口给外部用
  - LoadBalancer：云厂商给一个公网入口
  - Ingress：按域名/路径把请求转发到不同 Service

  再记一个常见链路：

  外部用户
   -> Ingress / LoadBalancer
   -> Service
   -> Pod

  你这个例子如果套进去：

  - 集群内访问：test-k8s:8080
  - NodePort 访问：<NodeIP>:31000
