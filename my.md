# Raft Java 项目部署和运行记录

## 项目依赖
```xml
<dependency>
    <groupId>com.baidu</groupId>
    <artifactId>brpc-java</artifactId>
    <version>2.5.9</version>
</dependency>
```

## 环境配置

### 编译配置修改
- **Java版本**: 升级到 Java 17 (原为 Java 7/8)
- **Maven编译器插件**: 升级到 3.11.0
- **Lombok版本**: 升级到 1.18.30 (原为 1.18.4)
- **Log4j2版本**: 升级到 2.20.0 (原为 2.9.1)

### 日志配置
修改 `raft-java-example/src/main/resources/log4j2.xml`:
```xml
<Logger name="com.github.wenweihu86.raft" level="debug" additivity="false">
    <AppenderRef ref="Console" />
    <AppenderRef ref="RollingFile"/>
</Logger>
<Logger name="com.baidu.brpc" level="debug" additivity="false">
    <AppenderRef ref="RollingFile"/>
</Logger>
```

## 部署步骤

### 1. 构建项目
```bash
cd raft-java-core
mvn clean install -DskipTests

cd ../raft-java-example
mvn clean package
```

### 2. 部署节点 (Windows)
```powershell
# 创建目录
New-Item -ItemType Directory -Force -Path env\example1, env\example2, env\example3, env\client

# 复制部署包
Copy-Item target\raft-java-example-1.9.0-deploy.tar.gz env\example1\
Copy-Item target\raft-java-example-1.9.0-deploy.tar.gz env\example2\
Copy-Item target\raft-java-example-1.9.0-deploy.tar.gz env\example3\
Copy-Item target\raft-java-example-1.9.0-deploy.tar.gz env\client\

# 解压
cd env\example1; tar -xzf raft-java-example-1.9.0-deploy.tar.gz
cd ..\example2; tar -xzf raft-java-example-1.9.0-deploy.tar.gz
cd ..\example3; tar -xzf raft-java-example-1.9.0-deploy.tar.gz
cd ..\client; tar -xzf raft-java-example-1.9.0-deploy.tar.gz

# 创建数据目录
New-Item -ItemType Directory -Force -Path example1\data, example2\data, example3\data
```

### 3. 启动节点 (Windows)
```powershell
# 节点1
cd env\example1
$jarFiles = Get-ChildItem lib\*.jar | ForEach-Object { $_.FullName }
$classpath = "conf;" + ($jarFiles -join ';')
java --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED -cp $classpath com.github.wenweihu86.raft.example.server.ServerMain ./data "127.0.0.1:8051:1,127.0.0.1:8052:2,127.0.0.1:8053:3" "127.0.0.1:8051:1"

# 节点2
cd env\example2
$jarFiles = Get-ChildItem lib\*.jar | ForEach-Object { $_.FullName }
$classpath = "conf;" + ($jarFiles -join ';')
java --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED -cp $classpath com.github.wenweihu86.raft.example.server.ServerMain ./data "127.0.0.1:8051:1,127.0.0.1:8052:2,127.0.0.1:8053:3" "127.0.0.1:8052:2"

# 节点3
cd env\example3
$jarFiles = Get-ChildItem lib\*.jar | ForEach-Object { $_.FullName }
$classpath = "conf;" + ($jarFiles -join ';')
java --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED -cp $classpath com.github.wenweihu86.raft.example.server.ServerMain ./data "127.0.0.1:8051:1,127.0.0.1:8052:2,127.0.0.1:8053:3" "127.0.0.1:8053:3"
```

## Raft集群配置

### 节点信息
- **节点1**: 127.0.0.1:8051:1 (Leader)
- **节点2**: 127.0.0.1:8052:2 (Follower)
- **节点3**: 127.0.0.1:8053:3 (Follower)

### 集群配置字符串
```
127.0.0.1:8051:1,127.0.0.1:8052:2,127.0.0.1:8053:3
```

## Raft算法运行状态

### 选举过程
- 节点1发起Pre-vote和RequestVote请求
- 获得多数票后成为Leader
- 其他节点转为Follower状态

### 心跳机制
- Leader定期向所有Follower发送心跳
- Follower接收心跳后重置选举超时
- 维持Leader的权威性

### 日志复制
- Leader将日志条目复制到Follower
- Follower确认接收并持久化
- Leader等待多数节点确认后提交日志

### 日志输出示例
```
2026-03-20 22:37:58,005 INFO [pool-3-thread-1]  Running pre-vote in term 0
2026-03-20 22:38:27,046 INFO [pool-2-thread-10] AppendEntries response[RES_CODE_SUCCESS] from server 3 in term 1 (my term is 1)
2026-03-20 22:38:27,546 DEBUG [pool-3-thread-2] start new heartbeat, peers=[2, 3]
2026-03-20 22:38:30,092 DEBUG [server-work-thread-12]   heartbeat request from peer=1 at term=1, my term=1
```

## 关键技术点

### Java模块化问题
由于Java 17的模块化限制，需要添加以下JVM参数：
```
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
```

### 类路径配置
在Windows上需要使用分号(;)分隔jar文件：
```powershell
$classpath = "conf;" + ($jarFiles -join ';')
```

## 项目结构
```
raft-java/
├── raft-java-core/          # Raft核心实现
│   ├── src/main/java/
│   │   └── com/github/wenweihu86/raft/
│   │       ├── RaftNode.java
│   │       ├── service/
│   │       ├── storage/
│   │       └── proto/
│   └── pom.xml
├── raft-java-example/       # 示例应用
│   ├── src/main/resources/
│   │   └── log4j2.xml
│   └── pom.xml
└── my.md                   # 本文档
```

## Client调用示例

### 1. 设置数据
```powershell
cd env\client
$jarFiles = Get-ChildItem lib\*.jar | ForEach-Object { $_.FullName }
$classpath = "conf;" + ($jarFiles -join ';')
java --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED -cp $classpath com.github.wenweihu86.raft.example.client.ClientMain "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" "test_key" "test_value"
```

**结果**：`set request, key=test_key value=test_value response={"success": true}`

### 2. 获取数据
```powershell
cd env\client
$jarFiles = Get-ChildItem lib\*.jar | ForEach-Object { $_.FullName }
$classpath = "conf;" + ($jarFiles -join ';')
java --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED -cp $classpath com.github.wenweihu86.raft.example.client.ClientMain "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" "test_key"
```

**结果**：`get request, key=test_key, response={"value": "test_value"}`

### 3. 设置更多数据
```powershell
cd env\client
$jarFiles = Get-ChildItem lib\*.jar | ForEach-Object { $_.FullName }
$classpath = "conf;" + ($jarFiles -join ';')
java --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED -cp $classpath com.github.wenweihu86.raft.example.client.ClientMain "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" "key1" "value1"
```

**结果**：`set request, key=key1 value=value1 response={"success": true}`

## Raft日志观察

### 日志复制过程
```
2026-03-20 22:53:14,529 INFO AppendEntries response[RES_CODE_SUCCESS] from server 3 in term 1
2026-03-20 22:53:14,529 INFO AppendEntries response[RES_CODE_SUCCESS] from server 2 in term 1
2026-03-20 22:53:14,529 DEBUG newCommitIndex=1, oldCommitIndex=1
```

### 心跳机制
```
2026-03-20 22:53:15,027 DEBUG start new heartbeat, peers=[2, 3]
2026-03-20 22:53:15,030 INFO AppendEntries response[RES_CODE_SUCCESS] from server 3 in term 1
2026-03-20 22:53:15,030 INFO AppendEntries response[RES_CODE_SUCCESS] from server 2 in term 1
```

### 提交索引增长
```
2026-03-20 22:53:14,529 DEBUG newCommitIndex=1, oldCommitIndex=1
2026-03-20 22:53:53,622 DEBUG newCommitIndex=2, oldCommitIndex=2
2026-03-20 22:54:13,667 DEBUG newCommitIndex=4, oldCommitIndex=4
```

## 关键观察点

1. **客户端连接**：使用`list://`协议连接到所有Raft节点
2. **数据一致性**：通过Raft算法确保数据在所有节点间一致复制
3. **日志复制**：Leader将写操作日志复制到所有Follower
4. **心跳维持**：Leader定期发送心跳维持集群状态
5. **提交确认**：只有多数节点确认后日志才被提交

## 日志格式优化

### 日志格式修改
为了更好地学习和调试Raft算法，修改了日志格式以显示Java文件和行号信息。

#### 修改的文件
- `raft-java-example/src/main/resources/log4j2.xml`
- `raft-java-example/env/example1/conf/log4j2.xml`
- `raft-java-example/env/example2/conf/log4j2.xml`
- `raft-java-example/env/example3/conf/log4j2.xml`
- `raft-java-example/env/client/conf/log4j2.xml`

#### 日志格式配置
```xml
<PatternLayout pattern="%d %p [%t] %C.%M(%F:%L)\t%m%n" />
```

#### Logger配置
```xml
<Logger name="com.github.wenweihu86.raft" level="debug" additivity="false" includeLocation="true">
    <AppenderRef ref="Console" />
    <AppenderRef ref="RollingFile"/>
</Logger>
<Root level="info" includeLocation="true">
    <AppenderRef ref="Console" />
    <AppenderRef ref="RollingFile"/>
</Root>
```

### 日志格式对比

**修改前**：
```
2026-03-20 23:33:18,160 DEBUG [pool-3-thread-2]    start new heartbeat, peers=[2, 3]
```

**修改后**：
```
2026-03-20 23:33:18,160 DEBUG [pool-3-thread-2] com.github.wenweihu86.raft.RaftNode.startNewHeartbeat(RaftNode.java:724)       start new heartbeat, peers=[2, 3]
```

## Raft日志分析指南

### 与Raft算法直接相关的日志

#### 1. 心跳机制
```
com.github.wenweihu86.raft.RaftNode.startNewHeartbeat(RaftNode.java:724) - start new heartbeat, peers=[2, 3]
```
- **作用**：Leader定期向Follower发送心跳，维持权威性
- **关键信息**：peers=[2, 3] 表示向节点2和节点3发送心跳

#### 2. 日志复制
```
com.github.wenweihu86.raft.RaftNode.appendEntries(RaftNode.java:212) - is need snapshot=false, peer=2
```
- **作用**：Leader将日志条目复制到Follower
- **关键信息**：peer=2 表示向节点2复制日志，snapshot=false表示不需要快照

```
com.github.wenweihu86.raft.RaftNode.appendEntries(RaftNode.java:267) - AppendEntries response[RES_CODE_SUCCESS] from server 3 in term 3 (my term is 3)
```
- **作用**：接收Follower的日志复制响应
- **关键信息**：RES_CODE_SUCCESS表示成功，term 3表示任期号

#### 3. 提交索引管理
```
com.github.wenweihu86.raft.RaftNode.advanceCommitIndex(RaftNode.java:751) - newCommitIndex=6, oldCommitIndex=6
```
- **作用**：推进已提交的日志索引
- **关键信息**：newCommitIndex=6表示当前已提交到第6条日志

#### 4. 选举过程
```
com.github.wenweihu86.raft.RaftNode.startElection(RaftNode.java:xxx) - Running pre-vote in term 0
com.github.wenweihu86.raft.RaftNode.handleRequestVote(RaftNode.java:xxx) - RequestVote response from server X
```
- **作用**：节点发起选举和投票过程
- **关键信息**：term表示任期号，投票结果

### 与Raft算法无关的日志

#### 1. RPC框架日志
```
com.baidu.brpc.client.handler.ClientWorkTask.run(ClientWorkTask.java:72) - handle response, correlationId=323
```
- **作用**：RPC客户端处理响应
- **说明**：这是底层RPC框架的日志，不是Raft算法本身

#### 2. 线程管理
```
create thread:client-work-thread-1
create thread:timeout-timer-thread-1
```
- **作用**：RPC框架的线程池管理
- **说明**：与Raft算法逻辑无关

#### 3. 协议注册
```
register protocol:1 success
register load balance factory:RandomLoadBalanceFactory success
```
- **作用**：RPC协议和负载均衡策略注册
- **说明**：RPC框架初始化日志

#### 4. 通道管理
```
channel is non active, ip=127.0.0.1,port=8053
remove channel failed:Invalidated object not currently part of this pool
```
- **作用**：网络连接管理
- **说明**：RPC框架的连接池管理

#### 5. 系统关闭
```
com.baidu.brpc.thread.ShutDownManager$1.run(ShutDownManager.java:40) - Brpc do clean work...
```
- **作用**：RPC框架清理资源
- **说明**：系统关闭时的清理操作

### 学习Raft算法的关键日志

#### 核心Raft算法日志
1. **心跳**：`RaftNode.startNewHeartbeat`
2. **日志复制**：`RaftNode.appendEntries`
3. **提交索引**：`RaftNode.advanceCommitIndex`
4. **选举**：`RaftNode.startElection`
5. **投票**：`RaftNode.handleRequestVote`

#### 应该忽略的日志
1. **RPC框架**：`com.baidu.brpc.*`（除了与Raft交互的部分）
2. **线程管理**：`create thread:*`
3. **协议注册**：`register protocol:*`

### 从日志观察Raft运行状态

#### 集群状态示例
```
2026-03-20 23:34:31,371 DEBUG [pool-3-thread-2] com.github.wenweihu86.raft.RaftNode.startNewHeartbeat(RaftNode.java:724)       start new heartbeat, peers=[2, 3]
2026-03-20 23:34:31,371 DEBUG [pool-2-thread-15] com.github.wenweihu86.raft.RaftNode.appendEntries(RaftNode.java:212)  is need snapshot=false, peer=2
2026-03-20 23:34:31,371 DEBUG [pool-2-thread-17] com.github.wenweihu86.raft.RaftNode.appendEntries(RaftNode.java:212)  is need snapshot=false, peer=3
2026-03-20 23:34:31,375 INFO [pool-2-thread-17] com.github.wenweihu86.raft.RaftNode.appendEntries(RaftNode.java:267)   AppendEntries response[RES_CODE_SUCCESS] from server 3 in term 3 (my term is 3)
2026-03-20 23:34:31,376 DEBUG [pool-2-thread-17] com.github.wenweihu86.raft.RaftNode.advanceCommitIndex(RaftNode.java:751)     newCommitIndex=6, oldCommitIndex=6
```

#### 可观察到的信息
- **Leader状态**：节点1是Leader（发送心跳）
- **任期**：当前是term 3
- **日志状态**：已提交到索引6
- **集群健康**：心跳正常，日志复制成功

### Client与服务端日志的区别

#### Client日志特点
- **主要内容**：RPC框架的通信和连接管理
- **Raft相关**：几乎没有直接的Raft算法日志
- **原因**：Client只是通过RPC调用Raft集群的服务

#### 服务端日志特点
- **主要内容**：Raft算法的核心逻辑
- **Raft相关**：包含心跳、日志复制、选举等所有Raft机制
- **位置**：`env/example1/logs/raft_example.log`、`env/example2/logs/raft_example.log`、`env/example3/logs/raft_example.log`

### 日志文件位置

- **节点1日志**：`raft-java-example/env/example1/logs/raft_example.log`
- **节点2日志**：`raft-java-example/env/example2/logs/raft_example.log`
- **节点3日志**：`raft-java-example/env/example3/logs/raft_example.log`
- **Client日志**：`raft-java-example/env/client/logs/raft_example.log`

## Raft任期变化分析

### 任期变化过程

从日志中可以清楚地看到任期是如何递增的：

#### Term 0 → Term 1（第一次选举）
```
2026-03-20 22:38:17,268 INFO Running pre-vote in term 0
2026-03-20 22:38:17,393 INFO Running for election in term 1
2026-03-20 22:38:17,408 INFO Got vote from server 2 for term 1
2026-03-20 22:38:17,409 INFO Got majority vote, serverId=1 become leader
```

#### Term 1 → Term 2（第二次选举尝试）
```
2026-03-20 23:33:06,132 INFO Running pre-vote in term 1
2026-03-20 23:33:06,242 INFO Running for election in term 2
```

#### Term 2 → Term 3（第三次选举）
```
2026-03-20 23:33:16,062 INFO Running pre-vote in term 2
2026-03-20 23:33:16,108 INFO Running for election in term 3
2026-03-20 23:33:16,132 INFO Got vote from server 2 for term 3
2026-03-20 23:33:16,147 INFO Got majority vote, serverId=1 become leader
```

### Raft算法中的任期机制

#### 1. 任期递增规则
- **初始状态**：所有节点从 term 0 开始
- **每次选举**：任期号 +1（0→1→2→3）
- **严格递增**：任期号只能增加，不能减少

#### 2. 选举过程
```
Pre-vote（预投票）→ 正式选举 → 获得多数票 → 成为Leader
```

**详细步骤**：
1. **Pre-vote**：节点先进行预投票，避免不必要的选举
2. **正式选举**：预投票成功后，发起正式选举
3. **投票收集**：向其他节点请求投票
4. **成为Leader**：获得多数票后成为Leader

#### 3. 任期变化的原因
- **Leader故障**：当前Leader失效，触发新选举
- **网络分区**：节点间通信中断，可能触发选举
- **重新启动**：节点重启后可能触发选举

### 关键日志解释

#### 选举相关日志

| 日志内容 | 含义 | 作用 |
|---------|------|------|
| `Running pre-vote in term X` | 在任期X进行预投票 | 避免不必要的选举，提高稳定性 |
| `Running for election in term X` | 在任期X进行正式选举 | 发起新的选举 |
| `Got vote from server Y for term X` | 获得节点Y在任期X的投票 | 收集投票，争取成为Leader |
| `Got majority vote, serverId=X become leader` | 获得多数票，节点X成为Leader | 选举成功，成为Leader |

#### 心跳相关日志
```
AppendEntries response[RES_CODE_SUCCESS] from server 2 in term 3 (my term is 3)
```
- **作用**：Leader向Follower发送心跳
- **任期检查**：确保所有节点在同一任期

### 为什么任期会从1变到3？

从日志时间线分析：
1. **22:38:17** - Term 1 选举成功，节点1成为Leader
2. **23:33:06** - Term 2 选举开始（可能由于网络问题或重启）
3. **23:33:16** - Term 3 选举成功，节点1再次成为Leader

**可能的原因**：
- 节点重启导致Leader切换
- 网络分区导致选举超时
- 手动停止/启动节点

### Raft算法核心概念

#### 任期的作用
1. **逻辑时钟**：区分不同的Leader任期
2. **冲突解决**：确保只有一个有效的Leader
3. **日志一致性**：通过任期号判断日志的新旧

#### 任期递增的重要性
- **防止脑裂**：确保同一时间只有一个Leader
- **日志覆盖**：新任期的日志可以覆盖旧任期的日志
- **安全性**：保证系统的一致性和正确性

### 总结

**任期变化规律**：
- ✅ **严格递增**：0 → 1 → 2 → 3
- ✅ **每次选举+1**：新的选举总是增加任期号
- ✅ **多数票原则**：获得多数票才能成为Leader
- ✅ **心跳维持**：Leader通过心跳维持权威性

这些日志完美展示了Raft算法的选举机制和任期管理，是理解分布式一致性算法的绝佳实例！

## Raft算法与RocksDB的关系

### 分层架构

Raft算法和RocksDB是**完全不同层次**的组件，它们各司其职：

```
┌─────────────────────────────────────┐
│         应用层 (Client)           │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│      Raft一致性算法层            │
│  (选举、日志复制、提交)          │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│      状态机层 (RocksDB)         │
│  (实际数据存储和查询)            │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│      持久化存储层               │
│  (文件系统)                     │
└─────────────────────────────────┘
```

### 各层职责

#### 1. Raft算法层
- **职责**：保证分布式一致性
- **功能**：
  - Leader选举
  - 日志复制
  - 提交确认
  - 集群管理
- **不负责**：实际数据的存储

#### 2. 状态机层 (RocksDB)
- **职责**：存储和查询实际数据
- **功能**：
  - 键值对存储
  - 数据持久化
  - 快速查询
- **不负责**：分布式一致性

#### 3. 持久化层
- **职责**：文件系统存储
- **存储内容**：
  - Raft日志
  - 快照数据
  - RocksDB数据文件

### 从代码中看架构

从 `ExampleStateMachine.java` 可以看到：

```java
public class ExampleStateMachine implements StateMachine {
    private RocksDB db;  // RocksDB存储实际数据
    
    public void apply(byte[] data) {
        // Raft提交后，将数据应用到RocksDB
        ExampleProto.SetRequest request = 
            ExampleProto.SetRequest.parseFrom(data);
        db.put(request.getKey().getBytes(), 
               request.getValue().getBytes());
    }
    
    public ExampleProto.GetResponse get(ExampleProto.GetRequest request) {
        // 从RocksDB读取数据
        byte[] valueBytes = db.get(request.getKey().getBytes());
        // 返回查询结果
    }
}
```

### 为什么需要RocksDB？

#### 1. Raft算法的局限性
- **只负责一致性**：Raft只保证数据在多个节点间一致复制
- **不提供存储**：Raft本身不提供数据存储能力
- **需要状态机**：Raft需要配合状态机来实现完整的存储系统

#### 2. RocksDB的优势
- **高性能**：嵌入式KV数据库，读写速度快
- **持久化**：数据自动持久化到磁盘
- **可靠性**：支持快照和恢复
- **简单易用**：提供简单的键值对接口

### 数据流向示例

#### 写操作流程
```
1. Client发送写请求
   ↓
2. Raft Leader接收请求
   ↓
3. Raft将请求写入日志
   ↓
4. Raft复制日志到Follower
   ↓
5. 多数节点确认后提交
   ↓
6. Raft通知状态机应用数据
   ↓
7. RocksDB存储实际数据
```

#### 读操作流程
```
1. Client发送读请求
   ↓
2. Raft Leader接收请求
   ↓
3. Raft确保数据已提交
   ↓
4. 从RocksDB读取数据
   ↓
5. 返回查询结果
```

### 文件目录结构

```
example1/data/
├── rocksdb_data/           # RocksDB数据文件
│   ├── 000003.log         # RocksDB日志文件
│   ├── CURRENT            # 当前数据库状态
│   ├── LOCK               # 文件锁
│   ├── MANIFEST-000001    # 数据库清单
│   └── OPTIONS-000005    # 配置选项
├── raft_log/             # Raft日志文件
│   ├── log_segment_1
│   └── log_segment_2
└── snapshot/             # Raft快照
    └── data/             # 快照数据（包含RocksDB数据）
```

### Raft算法保证高可用的方式

#### 1. 数据复制
- Raft确保数据复制到多数节点
- 即使部分节点故障，数据仍然可用

#### 2. 自动故障转移
- Leader故障时自动选举新Leader
- 服务不中断

#### 3. 一致性保证
- 所有节点数据最终一致
- 避免脑裂问题

#### 4. 但不负责数据存储
- Raft只保证数据复制的一致性
- 实际数据存储由状态机（RocksDB）负责

### 关键概念

| 概念 | Raft算法 | RocksDB |
|------|----------|---------|
| **主要职责** | 分布式一致性 | 数据存储 |
| **核心功能** | 选举、日志复制 | KV存储 |
| **数据位置** | Raft日志 | 数据库文件 |
| **高可用** | ✅ 保证 | ❌ 不保证 |
| **数据持久化** | ✅ 保证 | ✅ 保证 |
| **查询能力** | ❌ 无 | ✅ 有 |

### 总结

**为什么会有RocksDB？**

1. **Raft是算法，不是存储系统**
   - Raft只负责一致性协议
   - 需要状态机来实际存储数据

2. **分层设计**
   - Raft层：保证多节点数据一致
   - RocksDB层：提供高性能数据存储

3. **各司其职**
   - Raft：解决分布式一致性问题
   - RocksDB：解决数据存储和查询问题

4. **协同工作**
   - Raft确保数据安全复制
   - RocksDB提供快速数据访问

**Raft算法保证高可用**指的是：
- ✅ 数据在多个节点间一致复制
- ✅ 自动故障转移
- ✅ 服务持续可用

但**不保证**：
- ❌ 单个节点的数据不丢失（需要RocksDB持久化）
- ❌ 数据的查询性能（需要RocksDB优化）

这是一个经典的**分层架构设计**，每一层专注于自己的职责，协同工作构建高可用的分布式存储系统！

