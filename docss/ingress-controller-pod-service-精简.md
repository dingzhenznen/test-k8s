  1. 官方 Ingress Controller（你已经熟悉）

  # Deployment
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ingress-nginx-controller
    namespace: ingress-nginx
  spec:
    replicas: 1
    selector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx
    template:
      spec:
        containers:
        - name: controller
          image: registry.k8s.io/ingress-nginx/controller:v1.8.1
          # 这个容器里运行着 Nginx + 控制器逻辑


通过 deployment 创建 Ingress Controller pod  （把controller 打包成镜像）

然后 通过service 来访问 Ingress Controller pod ，controller在pod中 来处理 Ingress 资源