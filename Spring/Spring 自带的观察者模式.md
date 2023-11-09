# Spring 自带的观察者模式

## 1.概述

在设计模式中，**观察者模式**是一个比较常用的设计模式。维基百科解释如下：

> 观察者模式是软件设计模式的一种。在此种模式中，一个目标对象管理所有相依于它的观察者对象，并且在它本身的状态改变时主动发出通知。这通常透过呼叫各观察者所提供的方法来实现。
>
> 此种模式通常被用来实时事件处理系统。

在我们日常业务开发中，观察者模式对我们很大的一个作用，在于实现业务的**解耦**。以用户注册的场景来举例子，假设在用户注册完成时，需要给该用户发送邮件、发送优惠劵等等操作，如下图所示：

![](https://gitee.com/wanxianbo/pic-bed/raw/master/img/2021/04/20210409155117.jpg)

- UserService 在完成自身的用户注册逻辑之后，仅仅只需要发布一个 UserRegisterEvent 事件，而无需关注其它拓展逻辑。
- 其它 Service 可以**自己**订阅 UserRegisterEvent 事件，实现自定义的拓展逻辑。

> 友情提示：很多时候，我们会把**观察者模式**和**发布订阅模式**放在一起对比。
>
> 简单来说，发布订阅模式属于**广义上**的观察者模式，在观察者模式的 Subject 和 Observer 的基础上，引入 Event Channel 这个**中介**，进一步解耦。如下图所示：

![](https://gitee.com/wanxianbo/pic-bed/raw/master/img/2021/04/20210409155240.jpg)

## 2.Spring 事件机制

Spring 基于观察者模式，实现了自身的事件机制，由三部分组成：

- 事件 ApplicationEvent：通过继承它，实现自定义事件。另外，通过它的 source 属性可以获取事件源， timestamp 属性可以获取发生事件。
- 事件发布者 ApplicationEventPublisher：通过它，可以进行事件的发布。
- 事件监听器 ApplicationListener：通过实现它，进行指定类型的事件的监听。

> 友情提示：JDK 也内置了事件机制的实现，考虑到通用性，Spring 的事件机制是基于它之上进行拓展。因此，ApplicationEvent 继承自 `java.util.EventObject`，ApplicationListener 继承自 `java.util.EventListener`

3.入门示例

