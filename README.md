# simple-scheduler
# 节点角色
## Worker Cluster
执行具体任务
维护本地任务队列和线程池以及任务状态（待处理、执行中、已完成）。

## Coordinator Cluster
协调节点
负责任务分片、任务调度、负载均衡
维护任务状态（待调度、已调度）
任务超时/故障/异常处理（基于DB）

# 中间件
## MySQL
任务元数据存储
任务状态维护（行锁+CAS更新来避免重复调度/重复执行）

## ZooKeeper
Worker心跳检测、负载监控
Worker服务发现

# 简要设计
## 思路
- 无状态Worker水平扩展任务执行能力
- 行锁+CAS确保任务唯一执行
- zk用于解决worker组网问题以及负载均衡的依据表
- Coordinator去中心化，无主Gossip通信进行组网。依赖ZK list-watch 内存存储Worker状态。
- Coordinator 一致性哈希解决分片调度，Gossip同时生成全局环形Hash表（FingerTable）
## Task Table
```
id, status, owner, hash
```
行锁+CAS更新
```
if update set where id = ? and owner = ? and status = ? >0
  then do somthing
else
  abort
```
预处理任务哈希
hash = mmhash\(id\)
## Worker
### 服务发现
启动时向zk上报信息，定时发送心跳
### 负载监控
定期上报任务执行数（等待数）
负载阈值控制在自身资源允许范围内的60%，允许部分超限，封顶80%
### 任务接收
同步接收Coordinator请求
收到时判断自身资源，如超限则拒收
未超则投入内部任务管理中
### 任务执行
- prepare: 预分配资源，更新DB，成功继续
- running: 执行
- finish: 释放资源，更新DB（如更新DB失败，则需配合checkpoint或2PC确保能够恢复）

## Coordinator
### 服务发现
启动时向任意 Coordinator 发起PINGPONG
定期向网络中发起 SYNC
### 任务调度
- 定时线程池扫Task Table
  ```
  select TASK where hash > PREV_NODE and hash <= MYSELF
  ```
- batch 分批处理（基于内存中的worker负载预分配）
- 调用投放请求
- 根据结果容错处理，再分配或更新DB
### 服务容错
若Cluster网络存在变化，则判断PREV_NODE是否变化（只关心自己的上一个节点）取代他的hash区间
### 超时处理
处理掉停留过久的任务，更新任务状态为初始状态，迭代进入下一批次的调度
### 数据清理
Purge线程，处理历史数据，降低MYSQL行数
