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

