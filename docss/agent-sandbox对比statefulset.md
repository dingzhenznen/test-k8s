 与 StatefulSet 更接近

  实际上，Sandbox Controller 与 StatefulSet Controller 更相似：

  ┌──────────────┬──────────────────────────────┬────────────────────┐
  │     特性     │         StatefulSet          │      Sandbox       │
  ├──────────────┼──────────────────────────────┼────────────────────┤
  │ 单例/有序    │ 有序部署（0..N-1）           │ 纯单例             │
  ├──────────────┼──────────────────────────────┼────────────────────┤
  │ 稳定网络标识 │ 是（pod-name.service）       │ 是（sandbox-name） │
  ├──────────────┼──────────────────────────────┼────────────────────┤
  │ 持久存储     │ 支持（volumeClaimTemplates） │ 支持               │
  ├──────────────┼──────────────────────────────┼────────────────────┤
  │ Pod 命名     │ 固定（name-0, name-1）       │ 固定（name）       │
  ├──────────────┼──────────────────────────────┼────────────────────┤
  │ 暂停/恢复    │ ❌ 不支持                    │ ✅ 支持            │
  ├──────────────┼──────────────────────────────┼────────────────────┤
  │ 预热池       │ ❌ 不支持                    │ ✅ 支持            │
  └──────────────┴──────────────────────────────┴────────────────────┘

  6. 总结

  ┌────────────────────────────────────────────────────────────┐
  │                     关系总结                                │
  ├────────────────────────────────────────────────────────────┤│
  │                                                            │
  │   架构模式：三者相同（Informer + WorkQueue + Reconcile）      │
  │                                                            │
  │   ReplicaSet ──────►  管理多个无状态 Pod（数量一致性）        │
  │       ▲                                                    │
  │       │                                                    │
  │   StatefulSet ─────►  管理多个有状态 Pod（顺序 + 稳定身份）    │
  │       ▲                                                    │
  │       │                                                    │
  │   Sandbox ─────────►  管理单个有状态 Pod（生命周期 + 预热池）  │
  │                                                            │
  │   Sandbox = StatefulSet（size=1） + 暂停/恢复 + 预热池机制    │
  │                                                            │
  └────────────────────────────────────────────────────────────┘

  所以你的理解是对的：Agent-Sandbox controller 确实类似于内置的 ReplicaSet 
  controller，但它是为"单例有状态工作负载"专门设计的增强版本，更接近 StatefulSet
  的简化版加上生命周期管理和预热优化。

✻ Sautéed for 1m 49s

※ recap: Analyzing how Agent-Sandbox controller reconciles resources. Next: explain the warm pool 
  adoption mechanism in detail. (disable recaps in /config)

❯ deployment 会通过副本集来管理pod ，StatefulSet 会创建副本集吗                                       
  ⎿  ⧉ Selected 1 lines from controllers/sandbox_controller.go in Visual Studio Code

⏺ 不会！ StatefulSet 不会创建 ReplicaSet，它和 ReplicaSet 是平级关系，都是 Kubernetes
  的核心工作负载控制器。

  让我澄清这个重要的架构区别：

  Kubernetes 工作负载控制器架构

  ┌─────────────────────────────────────────────────────────────────────┐
  │                     Kubernetes 工作负载控制器层级                      │
  ├─────────────────────────────────────────────────────────────────────┤
  │                                                                     │
  │   ┌─────────────┐                                                   │
  │   │ Deployment  │ ──────► 管理 ReplicaSet（滚动更新、扩缩容）          │
  │   └──────┬──────┘                                                   │
  │          │                                                          │
  │          ▼                                                          │
  │   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
  │   │ ReplicaSet  │     │ StatefulSet │     │  DaemonSet  │          │
  │   │  (RS)       │     │  (STS)      │     │             │          │
  │   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘          │
  │          │                   │                   │                  │
  │          ▼                   ▼                   ▼                  │
  │   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
  │   │ Pod x N     │     │ Pod-0,1,2.. │     │ Pod (每节点) │          │
  │   │ 无状态副本   │     │ 有状态副本   │     │ 守护进程    │          │
  │   └─────────────┘     └─────────────┘     └─────────────┘          │
  │                                                                     │
  │   关系：Deployment → ReplicaSet → Pod                              │
  │        StatefulSet ─────────────► Pod  （直接管理，无中间层）         │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘

  关键区别

  ┌──────────┬────────────────────────┬────────────────────────┬──────────────────────────┐
  │   特性   │       Deployment       │      StatefulSet       │      Agent-Sandbox       │
  ├──────────┼────────────────────────┼────────────────────────┼──────────────────────────┤
  │ 中间层   │ 创建 ReplicaSet        │ 无中间层               │ 无中间层                 │
  ├──────────┼────────────────────────┼────────────────────────┼──────────────────────────┤
  │ 直接管理 │ ReplicaSet             │ 直接管理 Pod           │ 直接管理 Pod/Service/PVC │
  ├──────────┼────────────────────────┼────────────────────────┼──────────────────────────┤
  │ Pod 命名 │ 随机（带 hash）        │ 有序（name-0,1,2）     │ 固定（name）             │
  ├──────────┼────────────────────────┼────────────────────────┼──────────────────────────┤
  │ 扩缩容   │ 通过 RS 控制副本数     │ 直接控制副本数         │ 单例（无扩缩容）         │
  ├──────────┼────────────────────────┼────────────────────────┼──────────────────────────┤
  │ 更新策略 │ RollingUpdate/Recreate │ RollingUpdate/OnDelete │ 暂停/恢复                │
  └──────────┴────────────────────────┴────────────────────────┴──────────────────────────┘

  StatefulSet 为什么不使用 ReplicaSet？

  1. ReplicaSet 的局限性

  ReplicaSet 的设计目标是无状态工作负载：
  - Pod 是可互换的（fungible）
  - Pod 名称是随机的（如 nginx-5d4f8b7c9-x2ab1）
  - 删除后重建的 Pod 没有稳定身份

  2. StatefulSet 的特殊需求

  StatefulSet 管理有状态工作负载，需要：

  # StatefulSet 创建的 Pod 有固定身份
  web-0, web-1, web-2  # 有序索引

  # 每个 Pod 有：
  # - 稳定的网络标识：web-0.nginx.default.svc.cluster.local
  # - 稳定的存储：pvc-web-0, pvc-web-1, pvc-web-2
  # - 有序部署/扩缩容（0→N 创建，N→0 删除）

  这些特性 ReplicaSet 无法提供，所以 StatefulSet 直接管理 Pod。

  代码层面的证据

  StatefulSet 控制器源码（简化）

  // pkg/controller/statefulset/stateful_set_control.go
  func (ssc *defaultStatefulSetControl) UpdateStatefulSet(...) error {
      // 1. 直接获取 Pod 列表（不通过 ReplicaSet）
      pods, err := ssc.getStatefulSetPods(set)

      // 2. 计算副本差异
      replicaCount := int(*set.Spec.Replicas)

      // 3. 有序创建/删除 Pod
      if len(pods) < replicaCount {
          // 按顺序创建：先创建 web-0，再 web-1...
          ssc.createPod(set, nextOrdinal)
      } else if len(pods) > replicaCount {
          // 按逆序删除：先删除 web-2，再 web-1...
          ssc.deletePod(pods[highestOrdinal])
      }

      // 4. 更新状态
      updateStatus(set, pods)
  }

  Agent-Sandbox 控制器（实际代码）

  // controllers/sandbox_controller.go:230-271
  func (r *SandboxReconciler) reconcileChildResources(ctx context.Context, sandbox
  *sandboxv1beta1.Sandbox) error {
      // 直接管理 Pod（无 ReplicaSet 中间层）
      pod, err := r.reconcilePod(ctx, sandbox, nameHash)

      // 直接管理 Service
      svc, err := r.reconcileService(ctx, sandbox, nameHash)

      // 直接管理 PVC
      err := r.reconcilePVCs(ctx, sandbox, nameHash)
  }

  总结

  ┌─────────────────────────────────────────────────────────────┐
  │                      核心结论                                │
  ├─────────────────────────────────────────────────────────────┤
  │                                                             │
  │  ❌ StatefulSet 不会创建 ReplicaSet                          │
  总结

  ┌─────────────────────────────────────────────────────────────┐
  │                      核心结论                                │
  ├─────────────────────────────────────────────────────────────┤
  │                                                             │
  │  ❌ StatefulSet 不会创建 ReplicaSet                          │
  │                                                             │
  │  ✅ StatefulSet 直接管理 Pod（像 Agent-Sandbox 一样）         │
  │                                                             │
  │  ✅ Agent-Sandbox 和 StatefulSet 架构更接近：                 │
  │     都是直接管理 Pod，没有中间层 ReplicaSet                   │
  │                                                             │
  │  ⚠️  只有 Deployment 使用 ReplicaSet 作为中间层               │
  │     （为了支持滚动更新和回滚）                                │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  所以 Agent-Sandbox 控制器实际上是 StatefulSet 的"单例简化版"，两者都是直接管理 Pod，而不是通过
  ReplicaSet。
                                                    
