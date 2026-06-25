# Deployment 控制器完整调和（Reconcile）逻辑梳理
先明确核心定位：
Deployment 控制器属于 **kube-controller-manager** 内部的一个控制器，依靠 **Informer 监听 Deployment、ReplicaSet、Pod 资源变化**，触发 `Reconcile` 调和逻辑，**持续让实际集群状态 = Deployment 期望状态**。
核心中间载体：**ReplicaSet（副本集）**，Deployment 并不直接管理 Pod，而是管理 ReplicaSet。

## 一、前置基础概念
1. 期望状态：Deployment `.spec.replicas`、镜像版本、标签、滚动策略等
2. 当前状态：集群现存所有 RS、对应 Pod 数量、是否就绪、版本新旧
3. 调和本质：一轮循环 `取期望 → 对比现状 → 做增减/新建/删除动作`

## 二、触发 Reconcile 的时机（什么时候开始调和）
任意下面事件发生，都会进入调和逻辑：
1. 用户 `kubectl apply` 创建/修改 Deployment
2. Deployment 本身被删除
3. 关联的 ReplicaSet 被增删改
4. 关联的 Pod 被删除、异常退出、就绪状态变化
5. 控制器定时周期性兜底调和（防止事件丢失导致状态不一致）

## 三、Reconcile 整体总流程（7大步完整链路）
### 步骤1：获取当前 Deployment 对象 + 容错校验
1. 根据队列里的 Deployment Namespace/Name，从 Informer 本地缓存拿到 Deployment 实例
2. 判断 Deployment 是否被删除（带有 `deletionTimestamp` 删除时间戳）
   - 若标记删除：清理所有关联 ReplicaSet，更新状态后直接结束调和
   - 正常存在：继续往下执行
3. 校验 Deployment 配置合法性（副本数不能负数、模板合法性等）

### 步骤2：查找该 Deployment 名下所有 ReplicaSet
匹配规则：
- ReplicaSet 的 `ownerReference` 指向当前 Deployment
- ReplicaSet 标签匹配 Deployment `.spec.selector`

分类归集所有 RS：
1. **活跃 RS**：非零副本、有 Pod 在运行
2. **历史旧 RS**：版本过期、已经缩容到 0 的历史版本 RS（用于回滚）
3. 统计：所有 RS 总副本数、当前实际运行 Pod 总数

### 步骤3：判断是否需要新建 ReplicaSet（版本变更核心）
比较：Deployment 里的 Pod 模板 `.spec.template` 与每个 RS 的 Pod 模板
- 存在**模板完全一致**的 RS：不需要新建 RS，用这个 RS 作为目标 RS
- **没有匹配模板的 RS**（改了镜像、环境变量、探针、标签等）：
  1. 新建一个新版本 ReplicaSet，副本数初始默认 0
  2. 将新 RS 纳入管理，作为后续扩缩容目标版本

> 关键点：Deployment 滚动更新 = 新旧两个 RS 交替调整副本数

### 步骤4：根据更新策略（RollingUpdate / Recreate）执行副本调谐
Deployment 两种更新策略逻辑完全不同

#### 模式A：Recreate（重建更新）
1. 先把**所有旧 ReplicaSet 副本数缩为 0**，等待旧 Pod 全部删除完毕
2. 再将新 ReplicaSet 副本数设置为 `.spec.replicas`，创建新版本 Pod
特点：业务会短暂断服，适合有状态、不能新旧实例共存的应用

#### 模式B：RollingUpdate（默认滚动更新，最常用）
依赖两个参数：
- `maxSurge`：滚动过程最多可超出期望副本的数量（绝对值/百分比）
- `maxUnavailable`：滚动过程最多不可用 Pod 数量

调和动作：
1. 逐步**增加新 ReplicaSet 副本数**
2. 逐步**减少旧 ReplicaSet 副本数**
3. 全程约束总 Pod 数量、不可用数量不超过阈值
4. 反复迭代，直到新 RS 副本 = 期望 replicas，所有旧 RS 副本缩至 0

### 步骤5：清理多余历史 ReplicaSet（保留版本修订历史）
Deployment 配置 `revisionHistoryLimit`（默认10），用来控制保留多少个历史 RS：
1. 所有副本=0 的旧 RS 算作历史版本
2. 超过保留数量的最老历史 RS，直接删除
3. 保留的历史 RS 用于 `kubectl rollout undo` 版本回滚

### 步骤6：更新 Deployment status 状态字段
调和完成后，回填状态给 apiserver：
1. `replicas`：当前总副本数
2. `readyReplicas`：就绪副本数
3. `updatedReplicas`：已更新到最新版本副本数
4. `availableReplicas`：可用副本
5. `observedGeneration`：控制器已感知到的 Deployment 版本号
外部 `kubectl get deploy` 看到的状态就来源于这里

### 步骤7：处理 Deployment 暂停/恢复（suspend）
若 Deployment 设置 `spec.paused: true`：暂停滚动更新，不再新建新版本 RS，只做普通扩缩容；取消暂停后继续正常版本迭代。

## 四、扩缩容场景单独调和逻辑（只改 replicas，不改镜像）
1. Deployment 副本数修改触发调和
2. 找到当前活跃的最新版本 ReplicaSet
3. 直接修改该 RS 的 `.spec.replicas` 为目标值
4. ReplicaSet 控制器再去增减 Pod，Deployment 本身不直接操作 Pod

## 五、回滚（rollout undo）底层逻辑
1. 用户发起回滚，Deployment 控制器找到指定历史 RS
2. 把该历史 RS 的 Pod 模板覆盖到当前 Deployment
3. 触发新一轮 Reconcile 调和，走一遍上面滚动更新流程，把旧版本重新扩起来，当前新版本逐步缩容归零

## 六、关键依赖关系链（从上到下调用链路）
```
Deployment Controller（Reconcile循环）
    ↓ 管理
ReplicaSet
    ↓ 管理（ReplicaSet控制器调和）
Pod
```
一句话极简总结：
**Deployment 不直接管 Pod，通过管控多个 ReplicaSet 的副本数量，实现滚动更新、扩缩容、版本历史保留；Reconcile 循环不停对比期望与现实，驱动状态收敛。**

## 补充：读源码切入点（如果你后续要看源码）
1. 入口：`pkg/controller/deployment/deployment_controller.go` → `ReconcileDeployment` 主函数
2. 新建RS逻辑：`syncDeployment` 核心同步函数
3. 滚动更新逻辑：`rollout.RolloutDeployment`
4. RS扩缩容、版本比对、历史清理都在同包下细分方法

需要我画一份极简版**文字流程图**，或者整理一段对应源码关键函数逐行注释解析吗？