/etc/resolv.conf  和 /etc/hosts 两个文件的区别和作用


• 区别很大，虽然它们都和“域名解析”有关。

  /etc/hosts
  作用是：本地静态域名映射表。

  它直接写死“某个名字对应哪个 IP”，比如：

  127.0.0.1 localhost
  192.168.1.10 myapp.local

  意思是：

  - 访问 localhost，直接当成 127.0.0.1
  - 访问 myapp.local，直接当成 192.168.1.10

  特点：

  - 本机优先使用
  - 不需要 DNS 服务器
  - 适合手工指定少量域名
  - 是“名字 -> IP”的固定映射

  /etc/resolv.conf
  作用是：告诉系统去哪里查 DNS，以及怎么查。

  比如里面常见有：

  nameserver 8.8.8.8
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

  意思是：

  - nameserver：DNS 服务器是谁
  - search：当你写短名字时，自动补哪些后缀去查
  - options：DNS 查询规则参数

  在 Kubernetes Pod 里，这个文件特别重要，因为你写：

  test-k8s

  系统之所以能尝试解析成：

  test-k8s.default.svc.cluster.local

  靠的就是 /etc/resolv.conf 里的 search 域配置。

  所以可以直接这样区分：

  - /etc/hosts：本地手写的固定映射
  - /etc/resolv.conf：DNS 查询配置文件

  解析顺序上通常可以粗略理解为：

  1. 先看 /etc/hosts
  2. 没命中再按 /etc/resolv.conf 指定的 DNS 配置去问 DNS 服务器

  在 K8s 里它们的典型作用是：

  - /etc/hosts：保存 Pod 自己、localhost 等基础映射
  - /etc/resolv.conf：决定 Pod 怎么解析 test-k8s、mongodb 这种 Service 名称

  一句话记忆：

  - hosts 管“直接写死”
  - resolv.conf 管“去哪里问、怎么补全域名”

  如果你要，我可以继续结合 Kubernetes Pod 里的真实示例，给你解释 search 和 ndots:5 为什么会影响 test-k8s 这种短域名解析。


› kubectl exec -it test-k8s-6c97f5c8d5-sb6hb -- bash
  root@test-k8s-6c97f5c8d5-sb6hb:/app# cat /etc/resolv.conf
  nameserver 10.96.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
  10.96.0.10 是哪个节点ip


• 10.96.0.10 不是某个节点的 IP，它是 集群内 DNS Service 的 ClusterIP。

  你这个集群里对应的就是：

  - kube-system 命名空间下的 kube-dns Service
  - 之前我查到它是：10.96.0.10

  也就是说：

  nameserver 10.96.0.10

  表示这个 Pod 做 DNS 解析时，会去问集群里的 DNS 服务，而不是直接问某台 node。

  在 Kubernetes 里通常链路是这样：

  Pod
   -> 10.96.0.10
   -> kube-dns / CoreDNS
   -> 返回 Service / Pod 的解析结果

  你可以自己验证：

  kubectl get svc -n kube-system

  会看到类似：

  kube-dns   ClusterIP   10.96.0.10

  所以结论是：

  - 10.96.0.10 不是 Node IP
  - 它是集群 DNS 服务的虚拟 IP
  - 背后实际对应的是 CoreDNS/kube-dns 这组 Pod 提供解析能力

  如果你要，我可以继续讲一下：为什么 Pod 里看到的是 Service IP，而不是 CoreDNS Pod 的真实 IP。



  当你在 test-k8s 的 Pod 里访问 mongodb-0 时，一般会先按 /etc/resolv.conf 的规则去 DNS 服务器 10.96.0.10 查询，然后由 CoreDNS 返
  回 mongodb-0 对应的 Pod IP。

  也就是这条链路：

  test-k8s Pod
   -> 看 /etc/resolv.conf
   -> 去 nameserver 10.96.0.10 查询
   -> CoreDNS 返回 mongodb-0 的解析结果
   -> 得到 10.244.0.179

  不过这里有两个细节要分清。

  1. mongodb-0 不是普通 Service 名
  它是 StatefulSet Pod 在 Headless Service 下的稳定 DNS 名的一部分。更准确常用的是：

  mongodb-0.mongodb

  完整是：

  mongodb-0.mongodb.default.svc.cluster.local

  所以在别的 Pod 里访问时，通常写：

  mongodb-0.mongodb

  比只写 mongodb-0 更稳妥。

  2. /etc/hosts 是给 mongodb-0 自己这个 Pod 用的
  你在 mongodb-0 里看到：

  10.244.0.179 mongodb-0.mongodb.default.svc.cluster.local mongodb-0

  这表示这个 Pod 自己本地知道“我自己是谁”。
  但 test-k8s Pod 里的 /etc/hosts 不会自动有这条记录，所以 test-k8s 要解析 mongodb-0...，主要还是走 DNS，也就是去问 10.96.0.10。

  所以你的这句话：

  > test-k8s 的pod 回去10.96.0.10 这nameserve 中查找dns 记录然后找到mongodb-0 对应的ip

  可以改成更准确版本：

  - test-k8s Pod 会向 10.96.0.10 这个集群 DNS 查询
  - DNS 根据 Headless Service mongodb 和 StatefulSet Pod 记录
  - 返回 mongodb-0.mongodb.default.svc.cluster.local -> 10.244.0.179

  一句话记忆：

  - 查普通 Service：DNS 返回 Service 的 ClusterIP
  - 查 Headless Service 下的 StatefulSet Pod：DNS 返回具体 Pod IP

  如果你要，我可以继续把 mongodb、mongodb-0.mongodb、mongodb.default.svc.cluster.local 这几个名字分别会解析成什么，给你列成一张
  表。