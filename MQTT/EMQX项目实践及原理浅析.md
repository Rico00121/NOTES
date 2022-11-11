# EMQX项目实践及原理浅析

**问题：**

**实现mqtt协议的emqx是消息队列吗？**

**emqx和消息队列区别？**

**emqx的应用场景？**

## 1.MQTT协议简介

**概览**

[MQTT](https://mqtt.org/) 是一种基于发布/订阅模式的轻量级消息传输协议，专门针对低带宽和不稳定网络环境的物联网应用而设计，可以用极少的代码为联网设备提供实时可靠的消息服务。MQTT 协议广泛应用于物联网、移动互联网、智能硬件、车联网、智慧城市、远程医疗、电力、石油与能源等领域。

MQTT 协议由 [Andy Stanford-Clark](http://en.wikipedia.org/wiki/Andy_Stanford-Clark) （IBM）和 Arlen Nipper（Arcom，现为 Cirrus Link）于 1999 年发布。 按照 Nipper 的介绍，MQTT 必须具备以下几点：

- 简单容易实现

- 支持 QoS（设备网络环境复杂）

- 轻量且省带宽（因为那时候带宽很贵）

- 数据无关（不关心 Payload 数据格式）

- 有持续地会话感知能力（时刻知道设备是否在线）

### 1.1MQTT协议和AMQP对比

**mqtt**：emqx主流实现,包含client、broker

**amqp**：rabbit_mq主流实现、spring boot支持，包含client、broker、exchange、queue

**概要**

1. MQTT具有客户机/代理体系结构，而AMQP具有客户机或代理以及客户机或<u>**服务器体系结构**</u>。

2. MQTT遵循发布和订阅的**<u>抽象</u>**，而AMQP遵循响应或请求以及发布或订阅方法。

3. AMQP的头大小为<u>**8bytes**</u>，MQTT的头大小为**<u>2bytes</u>**。MQTT的消息大小很小且已定义，而AMQP具有可协商的和未定义的。

4. MQTT提供的服务质量是依赖**<u>QoS</u>**。AMQP提供的服务质量是消费端的**<u>ACK</u>**机制及事务消息等。

5. 用户安全方面，MQTT主要通过账密和鉴权服务等，而AMQP更多通过防火墙等。

### 1.2MQTT协议和HTTP对比

**概要**

1.MQTT 的最小报文仅为 2 个字节，比 HTTP 占用更少的网络开销。

2.MQTT 与 HTTP 都能使用 TCP 连接，并实现稳定、可靠的网络连接。

3.MQTT 基于发布订阅模型，HTTP 基于请求响应，因此 MQTT 支持双工通信。

4.MQTT 可实时推送消息，但 HTTP 需要通过轮询获取数据更新。

5.MQTT 是有状态的，但是 HTTP 是无状态的。

6.MQTT 可从连接异常断开中恢复，HTTP 无法实现此目标。

​                                                                                                1-2-1 emqx和mq架构图

![image-20221111110544597](/Users/mingchuan.mao/Library/Application Support/typora-user-images/image-20221111110544597.png)

### 1.2QoS简介

MQTT 设计了 3 个 QoS 等级。

- QoS 0：消息最多传递一次，如果当时客户端不可用，则会丢失该消息。
- QoS 1：消息传递至少 1 次。
- QoS 2：消息仅传送一次。

QoS 0 是一种 "fire and forget" 的消息发送模式：Sender (可能是 Publisher 或者 Broker) 发送一条消息之后，就不再关心它有没有发送到对方，也不设置任何重发机制。

QoS 1 包含了简单的重发机制，Sender 发送消息之后等待接收者的 ACK，如果没收到 ACK 则重新发送消息。这种模式能保证消息至少能到达一次，但无法保证消息重复。

QoS 2 设计了重发和重复消息发现机制，保证消息到达对方并且严格只到达一次。

#### 工作原理

**QoS 0 - 最多分发一次**

当 QoS 为 0 时，消息的分发依赖于底层网络的能力。发布者只会发布一次消息，接收者不会应答消息，发布者也不会储存和重发消息。消息在这个等级下具有最高的传输效率，但可能送达一次也可能根本没送达。

![MQTT_1.png](https://assets.emqx.com/images/b6e2c8b638f4f1b6f3388de0901c24e0.png?imageMogr2/thumbnail/1520x)

**Qos 1 - 至少分发一次**

当 QoS 为 1 时，可以保证消息至少送达一次。MQTT 通过简单的 ACK 机制来保证 QoS 1。发布者会发布消息，并等待接收者的 PUBACK 报文的应答，如果在规定的时间内没有收到 PUBACK 的应答，发布者会将消息的 DUP 置为 1 并重发消息。接收者接收到 QoS 为 1 的消息时应该回应 PUBACK 报文，接收者可能会多次接受同一个消息，无论 DUP 标志如何，接收者都会将收到的消息当作一个新的消息并发送 PUBACK 报文应答。

![MQTT_2.png](https://assets.emqx.com/images/a54f70242a83f7d39b51800008a724dd.png?imageMogr2/thumbnail/1520x)

**QoS 2 - 只分发一次**

当 QoS 为 2 时，发布者和订阅者通过两次会话来保证消息只被传递一次，这是最高等级的服务质量，消息丢失和重复都是不可接受的。使用这个服务质量等级会有额外的开销。

发布者发布 QoS 为 2 的消息之后，会将发布的消息储存起来并等待接收者回复 PUBREC 的消息，发送者收到 PUBREC 消息后，它就可以安全丢弃掉之前的发布消息，因为它已经知道接收者成功收到了消息。发布者会保存 PUBREC 消息并应答一个 PUBREL，等待接收者回复 PUBCOMP 消息，当发送者收到 PUBCOMP 消息之后会清空之前所保存的状态。

当接收者接收到一条 QoS 为 2 的 PUBLISH 消息时，他会处理此消息并返回一条 PUBREC 进行应答。当接收者收到 PUBREL 消息之后，它会丢弃掉所有已保存的状态，并回复 PUBCOMP。

无论在传输过程中何时出现丢包，发送端都负责重发上一条消息。不管发送端是 Publisher 还是 Broker，都是如此。因此，接收端也需要对每一条命令消息都进行应答。

![MQTT_3.png](https://assets.emqx.com/images/6656481cb11432b89be67f4e937f39cf.png?imageMogr2/thumbnail/1520x)

#### QoS 在发布与订阅中的区别

MQTT 发布与订阅操作中的 QoS 代表了不同的含义，发布时的 QoS 表示消息发送到服务端时使用的 QoS，订阅时的 QoS 表示服务端向自己转发消息时可以使用的最大 QoS。

- 当客户端 A 的发布 QoS 大于客户端 B 的订阅 QoS 时，服务端向客户端 B 转发消息时使用的 QoS 为客户端 B 的订阅 QoS。
- 当客户端 A 的发布 QoS 小于客户端 B 的订阅 QoS 时，服务端向客户端 B 转发消息时使用的 QoS 为客户端 A 的发布 QoS。

不同情况下客户端收到的消息 QoS 可参考下表：

**qos_level = min(sub(qos),pub(qos))**

| 发布消息的 QoS | 主题订阅的 QoS | 接收消息的 QoS |
| --------- | --------- | --------- |
| 0         | 0         | 0         |
| 0         | 1         | 0         |
| 0         | 2         | 0         |
| 1         | 0         | 0         |
| 1         | 1         | 1         |
| 1         | 2         | 1         |
| 2         | 0         | 0         |
| 2         | 1         | 1         |
| 2         | 2         | 2         |

## 2.EMQX基本使用

### 2.1主题通配符

MQTT 主题通配符包含单层通配符 `+` 及多层通配符 `#`，主要用于客户端一次订阅多个主题。

> **注意**：通配符只能用于订阅，不能用于发布。

#### 单层通配符

加号 (“+” U+002B) 是用于单个主题层级匹配的通配符。在使用单层通配符时，单层通配符必须占据整个层级，例如：

```lsl
+ 有效
sensor/+ 有效
sensor/+/temperature 有效
sensor+ 无效（没有占据整个层级）
```

如果客户端订阅了主题 `sensor/+/temperature`，将会收到以下主题的消息：

```awk
sensor/1/temperature
sensor/2/temperature
...
sensor/n/temperature
```

但是不会匹配以下主题：

```bash
sensor/temperature
sensor/bedroom/1/temperature
```

#### 多层通配符

井字符号（“#” U+0023）是用于匹配主题中任意层级的通配符。多层通配符表示它的父级和任意数量的子层级，在使用多层通配符时，它必须占据整个层级并且必须是主题的最后一个字符，例如：

```awk
# 有效，匹配所有主题
sensor/# 有效
sensor/bedroom# 无效（没有占据整个层级）
sensor/#/temperature 无效（不是主题最后一个字符）
```

如果客户端订阅主题 `senser/#`，它将会收到以下主题的消息：

```lsl
sensor
sensor/temperature
sensor/1/temperature
```

### 2.2广播和负载均衡

emqx发送的普通topic消息默认是广播模式，如果想要处在同一集群的客户端只消费一次，可以使用特殊的topic。

```bash
$share/{groupName}/{TopicName}
```

下图中，3 个订阅者用共享订阅的方式订阅了同一个主题 `$share/g/topic`，其中`topic` 是它们订阅的真实主题名，而 `$share/g/` 是共享订阅前缀（`g/` 是群组名，可为任意 UTF-8 编码字符串）。

![MQTT 共享订阅](https://assets.emqx.com/images/c248e9334ff6d32cbec0ed71cde98b1f.png?imageMogr2/thumbnail/1520x)

### 2.3延时消息

EMQ X 的延迟发布功能可以实现按照用户配置的时间间隔延迟发布 PUBLISH 报文的功能。当客户端使用特殊主题前缀 `$delayed/{DelayInteval}` 发布消息到 EMQ X 时，将触发延迟发布功能。

延迟发布主题的具体格式如下：

```bash
$delayed/{DelayInterval}/{TopicName}
```

- `$delayed`: 使用 `$delay` 作为主题前缀的消息都将被视为需要延迟发布的消息。延迟间隔由下一主题层级中的内容决定。
- `{DelayInterval}`: 指定该 MQTT 消息延迟发布的时间间隔，单位是秒，允许的最大间隔是 4294967 秒。如果 `{DelayInterval}` 无法被解析为一个整型数字，EMQ X 将丢弃该消息，客户端不会收到任何信息。
- `{TopicName}`: MQTT 消息的主题名称。

例如:

- `$delayed/15/x/y`: 15 秒后将 MQTT 消息发布到主题 `x/y`。
- `$delayed/60/a/b`: 1 分钟后将 MQTT 消息发布到 `a/b`。
- `$delayed/3600/$SYS/topic`: 1 小时后将 MQTT 消息发布到 `$SYS/topic`。

## 3.EMQX结合BEE_ART使用

### 3.1权限配置

内置 ACL 是优先级最低规则表，在所有的 ACL 检查完成后，如果仍然未命中则检查默认的 ACL 规则。

该规则文件以 Erlang 语法的格式进行描述：

```erlang
%% 允许 "dashboard" 用户 订阅 "$SYS/#" 主题
{allow, {user, "dashboard"}, subscribe, ["$SYS/#"]}.

%% 允许 IP 地址为 "127.0.0.1" 的用户 发布/订阅 "$SYS/#"，"#" 主题
{allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.

%% 拒绝 "所有用户" 订阅 "$SYS/#" "#" 主题
{deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

%% 允许其它任意的发布订阅操作
{allow, all}.
```

![image-20221111113007306](/Users/mingchuan.mao/Library/Application Support/typora-user-images/image-20221111113007306.png)

### 3.2配置文件映射

### 3.3后端使用流程

### 3.4优化

1.acl权限文件后端取消对event topic的订阅

![image-20221111114607258](/Users/mingchuan.mao/Library/Application Support/typora-user-images/image-20221111114607258.png)

2.后端qos等级下调

![image-20221111113411911](/Users/mingchuan.mao/Library/Application Support/typora-user-images/image-20221111113411911.png)