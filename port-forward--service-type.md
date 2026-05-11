ClusterIP 和 NodePort 两种类型的 service 都可以通过 kubectl port-forward来转发吧


• 对，这两种 Service 都可以用 kubectl port-forward 转发。

  kubectl port-forward 本质上不是依赖 Service 的暴露类型，而是通过 Kubernetes API 建
  立一条临时隧道，所以无论是：

  - ClusterIP
  - NodePort

  都可以执行类似：

  kubectl port-forward svc/test-k8s 8080:8080

  然后你本地访问：

  http://127.0.0.1:8080

  几点区别要分清：

  - ClusterIP：默认只能集群内访问，但你仍然可以用 kubectl port-forward 从本机临时访
    问
  - NodePort：本来就可以通过 NodeIP:NodePort 访问，也同样可以再用 kubectl port-
    forward
  - port-forward 是开发/调试手段，不等于真正对外暴露服务

  一句话记忆：

  Service 能不能 port-forward，和它是 ClusterIP 还是 NodePort 没强绑定；这两者都可以
  转发。


  