---
layout: post
title:  "有限状态机(FSM)的入门介绍及使用"
date:   2018-12-31 12:50:00
categories: 设计模式
tags: 状态机,Spring statemachine
excerpt: 有限状态机的入门介绍及使用
mathjax: true
---

* content
{:toc}

在实际开发中，可能会遇到使用一大块的if...else...的代码，并且通常情况下是多层if...else...的嵌套。如果代码有注释还好，可以通过注释梳理整个业务逻辑。如果没有注释，那么只能靠阅读每一个分支，拼出整个的逻辑，并且不一定能保证最终的正确性和完整性。

为了不给别人添麻烦，也为了让自己更容易地维护项目，在这种情况下，如果可以将整个流程分成一个个的状态的变换的话，可以考虑使用有限状态机来编写或重构上述的代码。对于学习过随机过程相关课程的同学，可以回想下，马尔可夫链中的状态转移图从而类比理解状态机中状态变换。整个业务流程在某一个时刻只能处于一个状态，只能执行给定的事件集合，从而触发状态变换。

举一个简单的例子来理解理解状态机。通常情况下，灯有两种状态：灯灭、灯亮，我们能做的事情也有两件：打开开关、关闭开关。灯灭时，只能执行打开开关事件，使得灯亮；灯亮时，只能执行关闭开关事件，使得灯灭。

使用Spring Statemachine来实现一个灯的状态机：

#### 1. 定义状态

```
public enum LightState {

    /**
     * 灯灭
     */
    OFF,

    /**
     * 灯亮
     */
    ON;
}
```

#### 2. 定义事件

```
public enum LightEvent {

    /**
     * 打开开关
     */
    SWICH_ON,

    /**
     * 关闭开关
     */
    SWITCH_OFF;
}
```

#### 3. 配置状态机

```
@Configuration
@EnableStateMachine
public class LightSateMachineConfig extends EnumStateMachineConfigurerAdapter<LightState, LightEvent> {

    @Override
    public void configure(StateMachineConfigurationConfigurer<LightState, LightEvent> config)
            throws Exception {
        config
                .withConfiguration()
                .machineId("lightStateMachine")
                .autoStartup(true)
                .listener(listener());
    }

    @Override
    public void configure(StateMachineStateConfigurer<LightState, LightEvent> states)
            throws Exception {
        states
                .withStates()
                .initial(LightState.OFF)
                .states(EnumSet.allOf(LightState.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<LightState, LightEvent> transitions)
            throws Exception {
        transitions
                .withExternal()
                .source(LightState.OFF).target(LightState.ON).event(LightEvent.SWICH_ON)
                .and()
                .withExternal()
                .source(LightState.ON).target(LightState.OFF).event(LightEvent.SWITCH_OFF);
    }

    @Bean
    public StateMachineListener<LightState, LightEvent> listener() {
        return new StateMachineListenerAdapter<LightState, LightEvent>() {
            @Override
            public void stateChanged(State<LightState, LightEvent> from, State<LightState, LightEvent> to) {
                System.out.println("light state change to " + to.getId());
            }
        };
    }
}
```

#### 4. 使用状态机

```
@Autowired
private StateMachine<LightState, LightEvent> lightStateMachine;

@Test
public void lightMachine() {
    lightStateMachine.sendEvent(LightEvent.SWICH_ON);
    lightStateMachine.sendEvent(LightEvent.SWICH_ON);
    lightStateMachine.sendEvent(LightEvent.SWITCH_OFF);
}
```

输出结果：

```
light state change to ON
light state change to OFF
```

可以看到，虽然向状态机发送了3个事件：打开开关、打开开关、关闭开关，但是状态只变换了2次，原因是当灯已经被打开时，再次发送打开开关的事件时，灯已经是灯亮的状态，灯亮时只能执行关闭事件，再次打开的事件会被忽略。

### 总结

使用状态机时，抽象出业务中的一个个状态，然后定义好两个相邻状态间的事件集合，然后向状态机发送事件，使得状态机运转起来。期间，可以监听状态变化的过程，实现想要的业务逻辑。