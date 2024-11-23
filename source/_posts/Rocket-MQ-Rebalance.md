---
title: Rocket MQ Rebalance
date: 2024-11-23 13:24:29
tags:
---
# Rocket MQ Rebalance 窥探
> 🍥因为只看了RocketMQ的一点源码，所以就只写了RocketMQ。当然也会根据查到的资料尽量与Kafka做对比。
> 
> 下文会简称RocketMQ为RMQ或者火箭队列（对打字友好🥳）

## 什么是Rebalance？
中文直译为**再均衡**。就RMQ来说是指将一个Topic下的**多个队列**（在Kafka中称之为Partition）在同一个消费者组下的多个消费实例之间进行重新分配。

Rebalance的本质是为了提升消息的**并行处理能力**。

举个🌰。假设一个Topic下有5个队列，在有两个消费者的情况下我们可以把其中3个队列个消费者1，另外两个队列给消费者2。
> 请暂时记住这个例子，下文会用到——在下文中简称上述的示例为例一

![rebalance_reason.png](Rocket-MQ-Rebalance/rebalance_reason.png)

但是rebalance也会对系统稳定性带来危害——这也是本文要着重表述的点。

## Rebalance的代价
### 消费暂停
考虑Topic下有5个队列且只对应一个消费实例。为了转化到例一的情景，消费者1需要暂停它对其中两个队列的消费。

等到消费者2就绪后这两个队列才能被消费者2消费。
### 重复消费
出于性能考虑在大多数场景下，消费offset的提交都是异步的。

在消费暂停的例子中，假设broker记录的消费offset为8，消费者1可能已经消费到offset为10的消息但是还未提交到队列中。
这个时候Rebalance机制介入，消费者2就绪后并不会等待消费者1提交最新的offset。它只会根据broker记录的offset继续往后消费。

这个时候就会出现消费者2消费了已经被消费者1处理过的消息。

### 消息积压&消费突刺
结合上述提到的暂停和重复消费，有以下两个场景会导致消息积压或消费突刺。
1. 重复消息过多：新的消费者快速就绪并准备消费，但是原来的消费者由于某些原因导致broker记录的offset与真实的偏差过大。那么新的消费者就要面临很多重复消息的消费。
2. 暂停时间过长：新消费者就绪时间过长，导致消息积压的量过大。

> 下文将从生产者和消费者两个角度来剖析Rebalance机制。只有了解相应的细节，在排查问题的时候才不会像无头苍蝇一样。

## Broker端Rebalance机制
从本质上来说，触发再平衡的根本因素无非两个。
1. 订阅Topic的队列数量发生变化
2. 消费者组发生变化

|  队列信息变化 | 消费组发生变化                                                    |
|---|------------------------------------------------------------|
| broker Crash<br/> broker upgrade<br/> 队列扩缩容  | 消费者Crash<br/> 网络异常导致消费者下线<br/> 消费者主动扩缩容<br/> Topic订阅信息发生变化 |

本文中将上述信息称之为Rebalance**元数据**。Broker负责维护上述数据，在其发生变化时以某种方式通知对应的消费者组进行Rebalance。

Broker有一个元数据管理器来管理Rebalance所需的元数据
![broker-meta.png](Rocket-MQ-Rebalance/broker-meta.png)
### 队列信息变化
队列信息维护在TopicConfigManager中。每个broker都会将自己的信息上报给NameServer，由NameServer来组装最终形成Topic的路由。
这个过程是动态的。

当某个broker crash之后，NameSever由于收不到该broker上报的Topic信息，会更新路由信息。这个时候客户端到NameServer拉取信息时会发现队列**变少**从而触发Rebalance。
![registerConsumer.png](Rocket-MQ-Rebalance/registerConsumer.png)
```java
public boolean registerConsumer(final String group, final ClientChannelInfo clientChannelInfo,
        ConsumeType consumeType, MessageModel messageModel, ConsumeFromWhere consumeFromWhere,
        final Set<SubscriptionData> subList, boolean isNotifyConsumerIdsChangedEnable, boolean updateSubscription) {
        long start = System.currentTimeMillis();
        // 查找消费者组信息
        ConsumerGroupInfo consumerGroupInfo = this.consumerTable.get(group);
        // 没有就创建一个默认的
        if (null == consumerGroupInfo) {
            callConsumerIdsChangeListener(ConsumerGroupEvent.CLIENT_REGISTER, group, clientChannelInfo,
                subList.stream().map(SubscriptionData::getTopic).collect(Collectors.toSet()));
            ConsumerGroupInfo tmp = new ConsumerGroupInfo(group, consumeType, messageModel, consumeFromWhere);
            ConsumerGroupInfo prev = this.consumerTable.putIfAbsent(group, tmp);
            consumerGroupInfo = prev != null ? prev : tmp;
        }
        // 消费者组信息是否更新
        boolean r1 =
            consumerGroupInfo.updateChannel(clientChannelInfo, consumeType, messageModel,
                consumeFromWhere);
        boolean r2 = false;
        if (updateSubscription) {
            // 订阅信息是否更新
            r2 = consumerGroupInfo.updateSubscription(subList);
        }
        
        if (r1 || r2) {
            if (isNotifyConsumerIdsChangedEnable) {
                // 通知消费组内下的所有消费者实例进行rebalance
                callConsumerIdsChangeListener(ConsumerGroupEvent.CHANGE, group, consumerGroupInfo.getAllChannel());
            }
        }
        if (null != this.brokerStatsManager) {
            this.brokerStatsManager.incConsumerRegisterTime((int) (System.currentTimeMillis() - start));
        }

        callConsumerIdsChangeListener(ConsumerGroupEvent.REGISTER, group, subList);

        return r1 || r2;
    }
```
callConsumerIdsChangeListener在处理ConsumerGroupEvent.CHANGE事件时，会给每个Consumer都发送一个NOTIFY_CONSUMER_IDS_CHANGED通知，这个消费者组下的所有实例在收到通知后，各自进行Rebalance，如下图所示：
![broker_2_consumer.png](Rocket-MQ-Rebalance/broker_2_consumer.png)

```java
// Client 端代码
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
        switch (request.getCode()) {
            // ... ignore
            case RequestCode.NOTIFY_CONSUMER_IDS_CHANGED:
                return this.notifyConsumerIdsChanged(ctx, request);
            // ... ignore
            default:
                break;
        }
        return null;
    }
    public RemotingCommand notifyConsumerIdsChanged(ChannelHandlerContext ctx,
                                                    RemotingCommand request) throws RemotingCommandException {
        try {
            final NotifyConsumerIdsChangedRequestHeader requestHeader =
                    (NotifyConsumerIdsChangedRequestHeader) request.decodeCommandCustomHeader(NotifyConsumerIdsChangedRequestHeader.class);
            log.info("receive broker's notification[{}], the consumer group: {} changed, rebalance immediately",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.getConsumerGroup());
            // 客户端收到该条消息后会立即开始rebalance
            this.mqClientFactory.rebalanceImmediately();
        } catch (Exception e) {
            log.error("notifyConsumerIdsChanged exception", RemotingHelper.exceptionSimpleDesc(e));
        }
        return null;
    }
```
>Broker通知每个消费者各自Rebalance，即每个消费者自己给自己重新分配队列，而不是Broker将分配好的结果告知Consumer。从这个角度，RocketMQ与Kafka Rebalance机制类似，二者Rebalance分配都是在客户端进行，不同的是：
>- Kafka：会在消费者组的多个消费者实例中，选出一个作为Group Leader，由这个Group Leader来进行分区分配，分配结果通过Cordinator(特殊角色的broker)同步给其他消费者。相当于Kafka的分区分配只有一个大脑，就是Group Leader。
>- RocketMQ：每个消费者，自己负责给自己分配队列，相当于每个消费者都是一个大脑。
  此时，我们需要思考2个问题：
  问题1：每个消费者自己给自己分配，如何避免脑裂的问题呢？
  因为每个消费者都不知道其他消费者分配的结果，会不会出现一个队列分配给了多个消费者，或者有的队列分配给了多个消费者。
  问题2：如果某个消费者没有收到Rebalance通知怎么办？
  每个消费者都会**定时**触发Rebalance，以避免Rebalance通知丢失。
