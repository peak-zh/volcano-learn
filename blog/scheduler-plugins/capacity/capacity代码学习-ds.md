## Volcano调度器的"Capacity"插件：集群资源的智能管家

在Kubernetes集群中管理多个团队共享资源时，如何公平分配资源？如何确保每个团队都有足够的资源运行关键任务？这就是Volcano调度器的**Capacity插件**要解决的核心问题。本文将带您深入了解这个集群资源的智能管家如何工作。

---

### 🏗️ 什么是Capacity插件？
Capacity插件是Volcano调度器的核心组件，负责**队列级别的资源配额管理**。想象一个大型企业的IT部门：
- **集群资源**：公司总预算
- **队列**：不同部门（研发部、市场部等）
- **任务**：各部门的项目

Capacity插件就是财务总监，确保：
1. 每个部门不超预算
2. 关键部门有最低保障
3. 紧急项目能获得额外资源

---

### 🌳 核心架构：层次化队列系统

每个队列都有三个关键属性：
1. **Capability**：资源上限（部门预算上限）
2. **Guarantee**：保障资源（部门最低保障）
3. **Deserved**：应得资源（根据需求动态计算）

---

### ⚙️ 核心机制解析

#### 1. **资源跟踪 - 实时会计系统**
插件维护每个队列的资源账本：
```go
type queueAttr struct {
    allocated  *api.Resource // 已分配资源（正在运行的任务）
    request    *api.Resource // 总请求资源（包括等待中的任务）
    inqueue    *api.Resource // 排队中的资源需求
    elastic    *api.Resource // 弹性资源（可回收部分）
}
```
就像财务系统实时跟踪：
- 已支出金额（allocated）
- 已审批预算（inqueue）
- 可调整预算（elastic）

#### 2. **多层资源校验 - 严格的财务审批**
当新任务申请资源时，插件会进行多层检查：

```go
func (cp *capacityPlugin) checkQueueAllocatableHierarchically(...) bool {
    // 从叶子队列逐级向上检查到根队列
    for i := len(list)-1; i >= 0; i-- {
        if !cp.queueAllocatable(...) {
            return false // 任一层级不满足即拒绝
        }
    }
    return true
}
```
这就像大额采购需要部门主管→总监→CFO层层审批。

#### 3. **动态资源分配 - 智能预算调整**
根据实际需求动态计算队列应得资源：
```go
// 计算逻辑简化版
deserved = min(
    capability,      // 不超过上限
    max(
        guarantee,   // 不低于保障
        request     // 满足实际需求
    )
)
```

---

### 🚦 四大核心功能

#### 1. **任务调度准入控制（AddAllocatableFn）**
```go
ssn.AddAllocatableFn(cp.Name(), func(queue *api.QueueInfo, candidate *api.TaskInfo) bool {
    return cp.checkQueueAllocatableHierarchically(ssn, queue, candidate)
})
```
- **作用**：检查任务是否可分配到队列
- **规则**：`(已分配 + 新任务) ≤ 队列能力`

#### 2. **作业入队控制（AddJobEnqueueableFn）**
```go
ssn.AddJobEnqueueableFn(cp.Name(), func(obj interface{}) int {
    return cp.checkJobEnqueueableHierarchically(ssn, queue, job)
})
```
- **作用**：决定作业能否进入调度队列
- **规则**：`(已分配 + 排队中 - 弹性资源) ≤ 队列能力`

#### 3. **资源回收机制（AddReclaimableFn）**
```go
ssn.AddReclaimableFn(cp.Name(), func(reclaimer *api.TaskInfo, reclaimees []*api.TaskInfo) (...) {
    // 选择可回收的任务
})
```
- **场景**：高优先级任务需要资源时
- **规则**：优先回收超过保障资源的任务

#### 4. **抢占决策（AddPreemptiveFn）**
```go
ssn.AddPreemptiveFn(cp.Name(), func(obj interface{}, candidate interface{}) bool {
    futureUsed := allocated + candidate资源
    return futureUsed ≤ deserved // 不超过应得资源才允许抢占
})
```
- **作用**：决定任务是否可被抢占
- **规则**：抢占后队列资源不超过应得资源

---

### 📊 实时资源监控

插件通过Prometheus暴露关键指标：
```go
metrics.UpdateQueueDeserved(...)    // 队列应得资源
metrics.UpdateQueueAllocated(...)   // 已分配资源
metrics.UpdateQueueRequest(...)     // 总请求资源
metrics.UpdateQueueShare(...)       // 资源使用率
```
运维人员可以监控：
- 哪些队列资源紧张
- 哪些队列资源闲置
- 集群整体利用率

---

### 🛠️ 实际应用场景

#### 案例1：保障核心业务
```yaml
# 支付队列-高保障
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: payment
spec:
  capability: 
    cpu: "32"
    memory: 64Gi
  guarantee:
    cpu: "16"
    memory: 32Gi
```

#### 案例2：限制临时任务
```yaml
# 测试队列-无保障
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: testing
spec:
  capability: 
    cpu: "8"
    memory: 16Gi
```

#### 案例3：多层资源分配
```yaml
# 研发部父队列
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: dev-department
spec:
  capability: 
    cpu: "64"
    memory: 128Gi

---
# 后端子队列
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: backend-team
  parent: dev-department # 指定父队列
spec:
  capability: 
    cpu: "32"
    memory: 64Gi
```

---

### ⚠️ 关键错误处理

插件严格检查队列层级合理性：
```go
func (cp *capacityPlugin) checkHierarchicalQueue(attr *queueAttr) error {
    // 检查父队列能力≥子队列总和
    if parentCapability < childTotal {
        return fmt.Errorf("父队列能力不足")
    }
    // 检查保障资源一致性
    if parentGuarantee < childGuarantees {
        return fmt.Errorf("父队列保障资源不足")
    }
}
```
确保不会出现：
- 子队列预算超过父部门
- 保障资源分配冲突

---

### 💡 设计亮点

1. **层次化资源管理**  
   支持企业级的多层资源分配结构

2. **动态资源调整**  
   根据实际需求动态计算应得资源

3. **弹性资源回收**  
   智能识别可回收的非关键任务资源

4. **全链路资源跟踪**  
   从任务提交到运行全程监控

5. **Prometheus集成**  
   提供实时资源监控指标

---

### 🌟 总结
Volcano的Capacity插件如同集群资源的智能管家：
1. **资源会计**：精确跟踪每个队列的资源使用
2. **预算控制**：严格执行资源配额限制
3. **灵活分配**：平衡保障需求与弹性需求
4. **多层管理**：支持复杂组织结构
5. **智能回收**：优化资源利用率


让您的Kubernetes集群资源分配既公平又高效！