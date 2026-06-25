# 先给结论：你的理解大体正确，但有两处关键细节纠正，整个链路不是控制器直接读 etcd，而是靠 **Informer 缓存机制** 间接感知变更，避免频繁直连 etcd。
完整逐段拆解 `kubectl apply` 全链路，帮你精准理清每一步：

## 一、步骤1：kubectl apply 提交 YAML → kube-apiserver
1. `kubectl apply -f deploy.yaml` 会把 YAML 解析为 JSON 结构化数据；
2. 通过 HTTPS 向 **kube-apiserver** 发送 REST 请求（POST/PUT/PATCH）；
3. apiserver 依次做前置校验：
   - 身份认证（证书/Token）
   - RBAC 权限鉴权（你有没有创建 Deployment 的权限）
   - 资源 Schema 合法性校验（字段类型、必填项、API版本是否合法）
   - 准入控制器（MutatingAdmissionWebhook 修改配置、ValidatingAdmissionWebhook 拦截非法配置）

## 二、步骤2：apiserver 写入 etcd（集群唯一数据源）
校验全部通过后：
apiserver **唯一操作 etcd 的组件**，把完整 Deployment 对象序列化存入 etcd 对应路径。
> 重点：**Controller、kubelet、kubectl 任何组件都不能直接读写 etcd**，所有读写必须经过 apiserver。

## 三、步骤3：控制器如何感知 etcd 里的数据变更（你理解最容易错的地方）
❌ 错误理解：控制器循环去查 etcd
✅ 真实机制：**apiserver Watch + Informer 本地缓存**
1. Deployment 控制器启动时，会通过 client-go 向 apiserver 发起 **Watch 长连接**；
2. apiserver 一旦发现 etcd 内 Deployment 发生新增/修改/删除事件，立刻通过 Watch 流式推送事件给客户端；
3. client-go 的 Informer 收到事件后：
   - 更新**本地内存缓存**（本地副本，后续查资源优先读缓存，不用频繁请求 apiserver）
   - 将变更对象 key（namespace/name）推入控制器的工作队列 WorkQueue

## 四、步骤4：控制器取出队列任务，执行 Reconcile 调和逻辑
1. 控制器 worker 从队列取出待调和 Deployment；
2. 先从 Informer 本地缓存拿到 Deployment、关联 ReplicaSet、Pod 全集（不用访问 apiserver/etcd）；
3. 对比「期望状态（Deployment.spec）」和「集群当前实际状态」；
4. 如果状态不一致，控制器会调用 client-go 发起创建/修改/删除 ReplicaSet 的请求，**再次走 apiserver → 最终写入 etcd**。

## 五、步骤5：ReplicaSet 控制器继续调和，最终管控 Pod
1. RS 变更同样触发 RS 控制器 Informer 感知，进入自身 Reconcile；
2. RS 计算需要创建/删除多少 Pod，调用 apiserver 增删 Pod；
3. Pod 写入 etcd 后，所有节点 kubelet 通过 Watch 感知本机 Pod 变化；
4. kubelet 调用容器运行时（containerd/docker）创建、启停容器。

## 极简完整版流转图
```
kubectl apply
    ↓ HTTP请求
kube-apiserver（认证/鉴权/准入校验）
    ↓ 写入
etcd（集群唯一真实数据）
    ↓ Watch推送变更事件
client-go Informer（更新本地缓存 + 塞入队列）
    ↓
Deployment Controller 取出任务执行 Reconcile
    ↓ 按需增删改 ReplicaSet（调用apiserver写入etcd）
ReplicaSet Controller 感知变更，Reconcile调谐Pod数量
    ↓ 创建/删除Pod（写入etcd）
kubelet 感知本机Pod变更，操作容器启停
```

## 补充两个核心设计目的
1. **不让控制器直连 etcd**：保证 apiserver 统一收口、权限可控、数据一致性；
2. **Informer 本地缓存**：海量集群下避免成千上万组件频繁轮询 apiserver，大幅降低集群压力。

需要我顺带区分 `kubectl create` 和 `kubectl apply` 在这个链路里的差异吗？