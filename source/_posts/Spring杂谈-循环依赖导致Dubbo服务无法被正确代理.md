---
title: Spring杂谈-循环依赖导致Dubbo服务无法被正确代理
tags: []
categories: []
date: 2018-09-30 14:33:49
---


## 背景

在阿里重启开源之前，基于dubbo注解的服务暴露有很多缺陷。公司小伙伴找到我帮他分析一个Dubbo服务使用aop不生效的问题，最后分析发现是在使用aop场景下如果接口包含循环依赖，Dubbo服务暴露是拿不到代理，所以就导致不生效了。

为了简化背景，业务方同学对外暴露controller接收http请求，在controller依赖`DeliveryOperateService`，在`DeliveryOperateService`依赖`CancelDistOrderProcessor`，在`CancelDistOrderProcessor`又依赖了`DeliveryOperateService`，这里大家应该发现循环依赖了，这种场景在开源版本`dubbo 2.5.8`之前是无法正确处理的。

- Controller

```
@RestController
@RequestMapping("/trade-dc/operate")
public class DeliveryOperateController {

  @Resource
  private DeliveryOperateService deliveryOperateService;
  ...
```

- DeliveryOperateService

```
@com.alibaba.dubbo.config.annotation.Service(protocol = {"dubbo"}, registry = {"haunt"})
public class DeliveryOperateServiceImpl implements DeliveryOperateService {

  @Resource
  private CancelDistOrderProcessor cancelDistOrderProcessor;
  ...
```

- CancelDistOrderProcessor

```
@Component
public class CancelDistOrderProcessor implements IComponent {

  @Resource
  private DeliveryOperateService deliveryOperateService;
  ...
```

在正常情况下，如果使用aop在dubbo暴露服务时会传递正确的spring动态代理后的对象:

![image](https://github.com/zonghaishang/images/1538291854892.png)

Dubbo服务暴露实际持有的对象是代理后的对象。但是因为循环依赖Dubbo无法正确处理实际持有`DeliveryOperateService`非代理的实例。

我们来看下为什么这种场景Dubbo无法处理？

Dubbo无法正确处理循环依赖aop代理.png![image](https://github.com/zonghaishang/images/1538293549011.png)
