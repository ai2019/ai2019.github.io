---
layout: post
title:  "Sentinel限流RocketMQ的消费速度"
date:   2019-08-05 14:00:00
categories: 中间件
tags: RocketMQ Sentinel 限流 MQ
excerpt: 使用Sentinel对MQ的消费进行限流
---

* content
{:toc}

### 1. 问题场景

在应用中，RocketMQ的consumer中调用了一个下游的RPC服务，然而下游的RPC服务进行了限流，导致MQ的consumer因为被限流而造成消费逻辑错误。
因此，为了保证业务流程的正确性，需要控制MQ消费者消费速度。

### 2. 解决方案

目的是要限制调用下游RPC服务的QPS，那么应该首先确认影响调用下游RPC服务QPS的因子，然后调节并加以控制。

#### 2.1 影响调用RPC服务QPS的因子

（1）MQ消费者数量

RocketMQ消费负载均衡是Topic下队列数据的消费负载均衡，当消费者集群的数量大于topic下队列的数量时，多出来的消费者集群不参与消费，因此消费者的数量是Topic下队列的数量。

> MQ消费者数量 = Topic下队列数量

（2）消费线程数

RocketMQ的每个消费者内部使用一个`ThreadPoolExecutor`并发消费拉取的消息，详情见RocketMQ官方文档[Best Practice For Consumer](https://rocketmq.apache.org/docs/best-practice-consumer/)。

因此，`ThreadPoolExecutor`线程池的大小也同样影响调用下游RPC的QPS：

- `setConsumeThreadMin`设置线程池的核心线程数；

- `setConsumeThreadMax`设置线程池的最大线程数。


因此：

> 单个消费者的最大并发数 = 消费者消费线程池的最大线程数

#### 2.2 QPS计算

通过上述的分析，调用下游RPC服务的最大QPS计算公式为：

> 调用最大QPS = Topic下队列数 * 消费者线程池最大线程


#### 2.3 限流方案1：调整消费者线程池的大小

由于Topic下的队列数是在申请Topic时已经固定的，后面无法再次修改，因此只能调节消费者的线程池大小。

> 注：当消费速度慢时，为了避免大量消息缓存在消费者本地，要调节`拉取消息的间隔`以及`单个队列在客户端缓存的最大消息数`。

#### 2.4 限流方案2：Sentinel限流

如果不考虑影响调用QPS的因素，最直接的方法是在调用下游RPC服务时，通过限流控制。当MQ消费者的调用QPS超过阈值时，让消费线程直接sleep，直到不触发限流。

为了代码的通用性，本文封装了一个带有限流消费的RocketMQ消费者的抽象类。

（1）限流消费的消费者抽象类

```java
@Slf4j
public abstract class AbstractLimitMessageListener implements MessageListenerConcurrently {

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
        if (CollectionUtils.isEmpty(list)) {
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
        try {
            list.forEach(this::processMessage);
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        } catch (Exception e) {
            log.error("AbstractLimitMessageListener exception", e);
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
    }

    /**
     * 单个消息处理
     *
     * @param msg
     */
    private void processMessage(MessageExt msg) {
        String resourceName = getSentinelResourceName();
        if (StringUtils.isBlank(resourceName)) {
            throw new RuntimeException("sentinel resource name is blank");
        }
        while (Boolean.FALSE.equals(SphO.entry(resourceName))) {
            log.info("{} execute limited", getSentinelResourceName());
            // 被限流
            try {
                TimeUnit.MILLISECONDS.sleep(getSleepTimeAfterLimit());
            } catch (InterruptedException e) {
                log.error("{} sleep exception", getSentinelResourceName(), e);
            }
        }
        try {
            doBusiness(msg);
        } finally {
            SphO.exit();
        }
    }

    /**
     * 获取被限流后，消费线程休眠的时间，单位：毫秒
     *
     * @return
     */
    protected abstract long getSleepTimeAfterLimit();

    /**
     * 业务逻辑处理
     *
     * @param msg
     */
    protected abstract void doBusiness(MessageExt msg);

    /**
     * Sentinel配置的资源名
     *
     * @return
     */
    protected abstract String getSentinelResourceName();
}
```

(2)Demo限流消费者

```java
@Service
@Slf4j
public class DemoLimitMessageListener extends AbstractLimitMessageListener {

    @Override
    protected long getSleepTimeAfterLimit() {
        return 100;
    }

    @Override
    protected void doBusiness(MessageExt msg) {
        // 执行具体的业务逻辑
    }

    @Override
    protected String getSentinelResourceName() {
        return this.getClass().getSimpleName();
    }
}
```

### 3. 总结

通常，使用MQ消费的时候一般都是调用本地的方法，一般不考虑对消费的速度进行限流。但是，当在消费的逻辑中调用了其他应用功能的服务时，就要考虑对其他应用造成的压力。

因此，为了保证服务的整体稳定性：

- 作为服务调用方时，要考虑对下游依赖的降级方案；
- 作为服务提供方时，要考虑对上游调用的限流方案。

此外，即使消费者只调用本应用内的服务，也要考虑当消息量非常大时，默认情况下会让本应用进入全功率服务状态，有可能造成应用集群的资源水位处于高位。
为了保证集群的稳定性，最好采取以下措施：
- 合理配置消费者的参数；
- 对消费者消费速度进行限流。

### 参考资料

- [RocketMQ官方文档：Simple Message Example](https://rocketmq.apache.org/docs/simple-example/)
- [RocketMQ官方文档：Best Practice For Consumer](https://rocketmq.apache.org/docs/best-practice-consumer/)
