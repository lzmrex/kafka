

---

## 🎯 **场景分析：订阅 5 个 Topic，1 个不存在，账号有读写权限**

### **场景设定**
```
订阅的 Topics: [topic-a, topic-b, topic-c, topic-d, topic-e]
其中: topic-e 不存在
账号权限: Read + Write (对所有 5 个 Topic)
GroupId: my-consumer-group (一致)
```


---

## 📊 **代码执行流程详解**

### **第 1 步：解析请求（925-940 行）**

假设客户端使用 Protocol v4（常见版本）：

```scala
val metadataRequest = request.body[MetadataRequest]
val requestVersion = request.header.apiVersion()  // 假设 = 4

val topics = 
  if (metadataRequest.isAllTopics)  // false，因为是订阅指定的 5 个
    metadataCache.getAllTopics()
  else
    metadataRequest.topics.asScala.toSet  // Set("topic-a", "topic-b", "topic-c", "topic-d", "topic-e")
```


**结果**：
```
topics = Set("topic-a", "topic-b", "topic-c", "topic-d", "topic-e")
```


---

### **第 2 步：Describe 权限校验（942-943 行）**

```scala
var (authorizedTopics, unauthorizedForDescribeTopics) =
  topics.partition(topic => authorize(request.session, Describe, new Resource(Topic, topic)))
```


**关键点**：根据 `SimpleAclAuthorizer.scala` 第 133-134 行的**权限继承规则**：

```scala
val allowOps = operation match {
  case Describe => Set[Operation](Describe, Read, Write, Delete, Alter)
  ...
}
```


**因为账号有 `Read` 和 `Write` 权限，所以自动隐含 `Describe` 权限！**

**权限校验过程**（对每个 Topic）：
```
topic-a: 有 Read/Write → 自动继承 Describe → ✅ authorized
topic-b: 有 Read/Write → 自动继承 Describe → ✅ authorized
topic-c: 有 Read/Write → 自动继承 Describe → ✅ authorized
topic-d: 有 Read/Write → 自动继承 Describe → ✅ authorized
topic-e: 有 Read/Write → 自动继承 Describe → ✅ authorized
```


**结果**：
```scala
authorizedTopics = Set("topic-a", "topic-b", "topic-c", "topic-d", "topic-e")
unauthorizedForDescribeTopics = Set()  // 空集
```


> ⚠️ **注意**：即使 `topic-e` 不存在，权限校验仍然通过！因为权限检查的是"是否有资格访问"，而不是"资源是否存在"。

---

### **第 3 步：检查不存在的 Topic（947-955 行）**

```scala
if (authorizedTopics.nonEmpty) {  // true
  val nonExistingTopics = metadataCache.getNonExistingTopics(authorizedTopics)
  // nonExistingTopics = Set("topic-e")
  
  if (metadataRequest.allowAutoTopicCreation && config.autoCreateTopicsEnable && nonExistingTopics.nonEmpty) {
    // 假设 allowAutoTopicCreation = true, autoCreateTopicsEnable = true
    if (!authorize(request.session, Create, Resource.ClusterResource)) {
      // 检查是否有 Cluster 级别的 Create 权限
      // 通常消费者没有这个权限
      authorizedTopics --= nonExistingTopics
      unauthorizedForCreateTopics ++= nonExistingTopics
    }
  }
}
```


**关键判断**：消费者账号通常**没有** `Cluster` 级别的 `Create` 权限

**结果**（假设没有 Create 权限）：
```scala
authorizedTopics = Set("topic-a", "topic-b", "topic-c", "topic-d")  // 移除了 topic-e
unauthorizedForCreateTopics = Set("topic-e")
```


---

### **第 4 步：构建无 Create 权限的响应（957-959 行）**

```scala
val unauthorizedForCreateTopicMetadata = unauthorizedForCreateTopics.map(topic =>
  new MetadataResponse.TopicMetadata(
    Errors.TOPIC_AUTHORIZATION_FAILED,  // ❌ 错误码
    topic,                               // "topic-e"
    isInternal(topic),                   // false
    java.util.Collections.emptyList()    // 无分区信息
  ))
```


**结果**：
```
unauthorizedForCreateTopicMetadata = [
  TopicMetadata {
    error: TOPIC_AUTHORIZATION_FAILED
    topic: "topic-e"
    isInternal: false
    partitionMetadata: []
  }
]
```


---

### **第 5 步：构建无 Describe 权限的响应（962-968 行）**

```scala
val unauthorizedForDescribeTopicMetadata =
  if ((requestVersion == 0 && ...) || metadataRequest.isAllTopics)
    Set.empty[MetadataResponse.TopicMetadata]
  else
    unauthorizedForDescribeTopics.map(...)  // 不会执行，因为集合为空
```


**结果**：
```scala
unauthorizedForDescribeTopicMetadata = Set()  // 空集
```


---

### **第 6 步：获取授权 Topic 的元数据（973-978 行）**

```scala
val topicMetadata =
  if (authorizedTopics.isEmpty)  // false
    Seq.empty[MetadataResponse.TopicMetadata]
  else
    getTopicMetadata(
      metadataRequest.allowAutoTopicCreation,  // true
      authorizedTopics,                        // Set("topic-a", "topic-b", "topic-c", "topic-d")
      request.listenerName,
      errorUnavailableEndpoints
    )
```


进入 `getTopicMetadata` 方法（897-919 行）：

```scala
private def getTopicMetadata(...) {
  val topicResponses = metadataCache.getTopicMetadata(topics, listenerName, errorUnavailableEndpoints)
  // topicResponses = [topic-a, topic-b, topic-c, topic-d 的完整元数据]
  
  if (topics.isEmpty || topicResponses.size == topics.size) {  // 4 == 4, true
    topicResponses  // 直接返回
  } else {
    // 不会执行，因为所有授权的 Topic 都存在
  }
}
```


**结果**：
```
topicMetadata = [
  TopicMetadata { error: NONE, topic: "topic-a", partitions: [...], ... },
  TopicMetadata { error: NONE, topic: "topic-b", partitions: [...], ... },
  TopicMetadata { error: NONE, topic: "topic-c", partitions: [...], ... },
  TopicMetadata { error: NONE, topic: "topic-d", partitions: [...], ... }
]
```


---

### **第 7 步：合并所有元数据（980 行）**

```scala
val completeTopicMetadata = topicMetadata ++ unauthorizedForCreateTopicMetadata ++ unauthorizedForDescribeTopicMetadata
```


**最终合并结果**：
```
completeTopicMetadata = [
  // 正常的 4 个 Topic
  TopicMetadata { error: NONE,              topic: "topic-a", partitions: [...] },
  TopicMetadata { error: NONE,              topic: "topic-b", partitions: [...] },
  TopicMetadata { error: NONE,              topic: "topic-c", partitions: [...] },
  TopicMetadata { error: NONE,              topic: "topic-d", partitions: [...] },
  
  // 不存在的 topic-e
  TopicMetadata { error: TOPIC_AUTHORIZATION_FAILED, topic: "topic-e", partitions: [] }
]
```


---

### **第 8 步：构建并发送响应（987-994 行）**

```scala
sendResponseMaybeThrottle(request, requestThrottleMs =>
  new MetadataResponse(
    requestThrottleMs,           // 限流延迟（通常为 0）
    brokers.map(_.getNode(...)).asJava,  // Broker 列表
    clusterId,                   // 集群 ID
    metadataCache.getControllerId.getOrElse(NO_CONTROLLER_ID), // Controller ID
    completeTopicMetadata.asJava // 上述 5 个 Topic 的元数据
  ))
```


---

## 🎯 **最终返回结果**

### **MetadataResponse 结构**

```java
MetadataResponse {
  throttleTimeMs: 0,
  
  brokers: [
    Node { id: 0, host: "broker-0.example.com", port: 9092 },
    Node { id: 1, host: "broker-1.example.com", port: 9092 },
    Node { id: 2, host: "broker-2.example.com", port: 9092 }
  ],
  
  clusterId: "cluster-abc123",
  controllerId: 0,
  
  topicMetadata: [
    // ✅ 正常 Topic
    {
      error: NONE,
      topic: "topic-a",
      isInternal: false,
      partitionMetadata: [
        { partition: 0, leader: 0, replicas: [0,1,2], isr: [0,1,2] },
        { partition: 1, leader: 1, replicas: [1,2,0], isr: [1,2,0] }
      ]
    },
    {
      error: NONE,
      topic: "topic-b",
      isInternal: false,
      partitionMetadata: [...]
    },
    {
      error: NONE,
      topic: "topic-c",
      isInternal: false,
      partitionMetadata: [...]
    },
    {
      error: NONE,
      topic: "topic-d",
      isInternal: false,
      partitionMetadata: [...]
    },
    
    // ❌ 不存在的 Topic
    {
      error: TOPIC_AUTHORIZATION_FAILED,  // ⚠️ 关键错误码
      topic: "topic-e",
      isInternal: false,
      partitionMetadata: []  // 空列表
    }
  ]
}
```


---

## 🔍 **客户端行为分析**

### **Kafka Consumer 的处理逻辑**

当消费者收到这个响应后：

```java
// 伪代码展示客户端行为
for (TopicMetadata metadata : response.topicMetadata()) {
    if (metadata.error() == Errors.NONE) {
        // topic-a, b, c, d: 正常订阅
        subscribe(metadata.topic());
    } else if (metadata.error() == Errors.TOPIC_AUTHORIZATION_FAILED) {
        // topic-e: 抛出异常
        throw new TopicAuthorizationException(
            "Not authorized to access topics: [" + metadata.topic() + "]"
        );
    }
}
```


### **实际表现**

```
✅ topic-a: 正常消费
✅ topic-b: 正常消费
✅ topic-c: 正常消费
✅ topic-d: 正常消费
❌ topic-e: 抛出 TopicAuthorizationException
```


**消费者日志**：
```
ERROR org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - 
Not authorized to access topics: [topic-e]
org.apache.kafka.common.errors.TopicAuthorizationException: 
Not authorized to access topics: [topic-e]
```


---

## 💡 **关键发现与注意事项**

### **1️⃣ 错误码的语义混淆**

```
TOPIC_AUTHORIZATION_FAILED 的实际含义：
- 不是"没有权限"
- 而是"没有自动创建的权限"
```


**原因追溯**：
- 第 950 行检查的是 `Create` on `Cluster` 权限
- 消费者通常只有 `Read/Write` on `Topic` 权限
- 没有 Cluster 级别的 Create 权限
- 所以返回 `TOPIC_AUTHORIZATION_FAILED`

**这是一个设计上的语义模糊点**！

---

### **2️⃣ 如果配置不同会怎样？**

#### **情况 A：关闭自动创建（`autoCreateTopicsEnable=false`）**

```scala
// 第 949 行条件不满足
if (metadataRequest.allowAutoTopicCreation && config.autoCreateTopicsEnable && ...)
  // 不会进入

// 直接进入 getTopicMetadata
val topicMetadata = getTopicMetadata(..., authorizedTopics, ...)
// authorizedTopics 仍包含 topic-e

// 在 getTopicMetadata 第 914 行
else {
  new MetadataResponse.TopicMetadata(
    Errors.UNKNOWN_TOPIC_OR_PARTITION,  // ✅ 更准确的错误码
    topic, false, java.util.Collections.emptyList())
}
```


**返回结果**：
```
topic-e: { error: UNKNOWN_TOPIC_OR_PARTITION, ... }
```


**客户端异常**：
```java
org.apache.kafka.common.errors.UnknownTopicOrPartitionException
```


---

#### **情况 B：消费者有 Cluster Create 权限**

```scala
// 第 950 行检查通过
if (!authorize(request.session, Create, Resource.ClusterResource)) {
  // 不会进入，因为有权限
}

// topic-e 保留在 authorizedTopics 中
// 在 getTopicMetadata 第 912 行自动创建
createTopic(topic, config.numPartitions, config.defaultReplicationFactor)

// 返回
TopicMetadata {
  error: LEADER_NOT_AVAILABLE,  // Leader 尚未选举完成
  topic: "topic-e",
  partitions: []
}
```


**结果**：`topic-e` 被自动创建，稍后重试即可正常消费！

---

### **3️⃣ 权限继承的关键作用**

```
Read/Write on Topic → 自动继承 Describe on Topic ✅
但不继承 Create on Cluster ❌
```


**这就是为什么**：
- ✅ 可以获取已存在 Topic 的元数据
- ❌ 不能触发不存在的 Topic 自动创建

---

## 📋 **总结表格**

| Topic | 存在性 | 权限 | 错误码 | 客户端行为 |
|-------|--------|------|--------|-----------|
| topic-a | ✅ 存在 | Read/Write | NONE | 正常消费 |
| topic-b | ✅ 存在 | Read/Write | NONE | 正常消费 |
| topic-c | ✅ 存在 | Read/Write | NONE | 正常消费 |
| topic-d | ✅ 存在 | Read/Write | NONE | 正常消费 |
| topic-e | ❌ 不存在 | Read/Write（无 Cluster Create） | **TOPIC_AUTHORIZATION_FAILED** | 抛出异常，无法订阅 |

---

## 🎓 **最佳实践建议**

### **1. 预先创建 Topic**
```bash
# 不要依赖自动创建
bin/kafka-topics.sh --create \
  --topic topic-e \
  --partitions 3 \
  --replication-factor 3
```


### **2. 授予正确的权限**
```bash
# 为消费者授予 Read + Describe（虽然 Read 已隐含 Describe）
bin/kafka-acls.sh --add \
  --allow-principal User:consumer-user \
  --operation Read \
  --operation Describe \
  --topic topic-e \
  --group my-consumer-group
```


### **3. 监控异常日志**
```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {}
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        logger.info("Assigned partitions: {}", partitions);
    }
});
```
