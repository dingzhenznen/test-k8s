续讲一下：kubectl port-forward pod/...、svc/...、
    deployment/... 三种写法有什么区别。


• 区别在于你把“转发目标”指向了哪一层对象。

  1. pod/...
  最直接，转发到某一个具体 Pod。

  kubectl port-forward pod/test-k8s-abcde 8080:8080

  特点：

  - 精确连到这个 Pod
  - 适合排查单个 Pod
  - 如果这个 Pod 重建、删除、名字变了，转发就失效
  - 不经过 Service 的负载均衡语义

  2. svc/...
  转发到某个 Service，再由 K8s 选后端 Pod。

  kubectl port-forward svc/test-k8s 8080:8080

  特点：

  - 用的是稳定的 Service 名称
  - 背后仍然会落到某个 Pod
  - 更贴近“访问服务”而不是“访问某个实例”
  - 适合日常开发、调试服务

  注意一点：port-forward 不是像真正访问 Service 那样长期做负载均衡，它只是帮你建立到
  后端的一条连接，不是生产流量入口。

  3. deployment/...
  转发到某个 Deployment 管理的一组 Pod 中的一个 Pod。

  kubectl port-forward deployment/test-k8s 8080:8080

  特点：

  - 不需要你自己找 Pod 名
  - kubectl 会从 Deployment 选一个 Pod 来转发
  - 比 pod/... 方便，比 svc/... 更偏“工作负载视角”
  - 如果你想验证某个应用是否起来了，这种写法很省事

  怎么选
  通常可以这样记：

  - 调单个实例问题：pod/...
  - 调一个稳定服务入口：svc/...
  - 懒得找 Pod，只想连上应用：deployment/...

  你这个 test-k8s 场景里，最常用一般是：

  kubectl port-forward svc/test-k8s 8080:8080

  因为这最符合“我就是想访问这个服务”。

  再补一句：

  - pod/... 是直接找 Pod
  - svc/... 是通过 Service 名义找后端
  - deployment/... 是通过 Deployment 找它管的某个 Pod

  如果你要，我可以继续补一个细节：这三种写法在“多副本场景下到底会连到谁”有什么实际表
  现差异。