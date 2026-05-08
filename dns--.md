
• 普通 Service 的固定 DNS 也是 Kubernetes 按规则自动生成的，一般不是在 YAML 里直接写完整域名，而是根据
  Service 名称和命名空间推出来。

  比如你这个普通服务 test-k8s：

  - Service 名：test-k8s
  - 命名空间：default

  它的固定 DNS 就是：

  test-k8s.default.svc.cluster.local

  同一个 namespace 里通常直接写短名也行：

  test-k8s

  或者：

  test-k8s.default.svc

  你可以从这里看出它的组成：

  kubectl get svc test-k8s -o yaml

  重点看：

  metadata:
    name: test-k8s
    namespace: default

  然后 Kubernetes 自动补上：

  所以普通 Service 的固定 DNS 规则就是：

  服务名.命名空间.svc.cluster.local

  例如：

  - kubernetes.default.svc.cluster.local
  - mongodb.default.svc.cluster.local
  - test-k8s.default.svc.cluster.local

  如果你想直接验证，可以这样：

  kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup test-
  k8s.default.svc.cluster.local

  一句话记忆：

  - 普通 Service DNS：service.namespace.svc.cluster.local
  - StatefulSet 单个 Pod DNS：podname.service.namespace.svc.cluster.local

  如果你要，我可以顺手把“普通 Service DNS”和“Headless Service + StatefulSet Pod DNS”整理成一张对比表。


› test-k8s 的完整 dns 是 test-
    k8s.default.svc.cluster.local ，我在 pod中使用test-k8s 这个服务名，能识别吗


• 能识别，前提是这个 Pod 和 test-k8s Service 在同一个 namespace。

  你这里 test-k8s 在 default namespace，所以如果你的 Pod 也在 default 里，直接写：

  test-k8s

  通常就能解析到：

  test-k8s.default.svc.cluster.local

  这是因为 Kubernetes Pod 里的 DNS 搜索域会自动补全当前 namespace，例如会带类似搜索域：

  default.svc.cluster.local
  svc.cluster.local
  cluster.local

  所以在 default namespace 的 Pod 中访问：

  http://test-k8s:8080

  一般就可以。

  如果你的 Pod 不在 default namespace，比如在 dev，那直接写 test-k8s 就会优先找：

  test-k8s.dev.svc.cluster.local

  这时通常找不到，你就要写成：

  test-k8s.default.svc.cluster.local

  或者至少写：

  test-k8s.default

  结论：

  - 同 namespace：test-k8s 可以识别
  - 不同 namespace：通常要写 test-k8s.default 或完整 FQDN

  你这个当前场景里，如果调用方 Pod 也在 default，直接用 test-k8s 没问题。


  对于无头服务 

kubectl exec -it mongodb-0 -- bash
root@mongodb-0:/# cat /etc/host


10.244.0.179    mongodb-0.mongodb.default.svc.cluster.local     mongodb-0