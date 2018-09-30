---
title: Spring杂谈-循环依赖导致Dubbo服务无法被正确代理
tags:
  - Spring杂谈
categories:
  - Spring杂谈
date: 2018-09-30 14:33:49
---


## 背景

在阿里重启开源之前，基于`dubbo`注解的服务暴露有很多缺陷。公司小伙伴找到我帮他分析一个`dubbo`服务使用`aop`不生效的问题，最后分析发现是在使用aop场景下如果接口包含循环依赖，`dubbo`服务暴露是拿不到代理，所以就导致不生效了。

为了简化背景，业务方同学对外暴露`controller`接收`http`请求, 依赖关系：

```
	Controller 
		-> DeliveryOperateService 
			-> CancelDistOrderProcessor 
				-> DeliveryOperateService
```

这里大家应该发现循环依赖了, `DeliveryOperateService `是应用了aop拦截，这种场景在开源版本`dubbo 2.5.8`之前是无法正确处理的。



``` java
// Controller
@RestController
@RequestMapping("/trade-dc/operate")
public class DeliveryOperateController {

  @Resource
  private DeliveryOperateService deliveryOperateService;
  ...
}

// DeliveryOperateService
@com.alibaba.dubbo.config.annotation.Service(protocol = {"dubbo"}, registry = {"haunt"})
public class DeliveryOperateServiceImpl implements DeliveryOperateService {

  @Resource
  private CancelDistOrderProcessor cancelDistOrderProcessor;
  ...
}

// CancelDistOrderProcessor
@Component
public class CancelDistOrderProcessor implements IComponent {

  @Resource
  private DeliveryOperateService deliveryOperateService;
  ...
}
```

在正常情况下，如果使用aop在dubbo暴露服务时会传递正确的spring动态代理后的对象:

![image](https://zonghaishang.github.io/images/1538291854892.png)

`Dubbo`服务暴露实际持有的对象是代理后的对象。但是因为循环依赖Dubbo无法正确处理实际持有`DeliveryOperateService`非代理的实例。

### 我们来看下为什么这种场景`Dubbo`无法处理？

![image](https://zonghaishang.github.io/images/1538293549011.png)

### Step1, `Spring`启动初始化`Controller`, 对属性进行注入

### Step2, `Controller`触发`DeliveryOperateService`创建实例

![image](https://zonghaishang.github.io/images/1538296119277.png)

第一次依赖注入就会触发bean的实例化并且保存在`exposedObject`中，，注意，这里是非代理对象。

### Step3, `DeliveryOperateService`依赖`CancelDistOrderProcessor`并触发它初始化

![image](https://zonghaishang.github.io/images/1538296756913.png)

这里也没什么特殊的，在`populateBean`会触发循环依赖`DeliveryOperateService`加载，这时候`earlySingletonExposure`值为true, 代表bean提前暴露。

### Step4, 在前一步触发，`DeliveryOperateService`其实会创建动态代理

![image](https://zonghaishang.github.io/images/1538297025076.png)

循环引用会导致提前暴露`earlySingletonExposure=true`，这个时候加载的是`getEarlyBeanReference`，在里面创建spring动态代理：

![image](https://zonghaishang.github.io/images/1538297199420.png)

在创建完动态代理后，`DeliveryOperateService`会加入`earlyProxyReferences`，后面再获取这个`bean`就不会再重复创建代理了。

![image](https://zonghaishang.github.io/images/1538297380747.png)

到此，`DeliveryOperateService`确实会创建，并且会用在`CancelDistOrderProcessor`对应注入的字段中。

### Step6, 触发Dubbo服务暴露的实例不是代理

因为在`CancelDistOrderProcessor`中已经触发了代理生成，所以第`Step1`中的实例不会再创建代理了。

![image](https://zonghaishang.github.io/images/1538297678652.png)

在代码`555`会触发`dubbo AnnotationBean`进行服务暴露，但是这个不是代理实例了，但是为什么spring还是正确返回代理后的实例呢？

![image](https://zonghaishang.github.io/images/1538297855578.png)

因为循环引用触发`earlySingletonExposure=true`, 并且在前面已经生成过动态代理了，可以直接在`getSingleton`拿到动态代理的返回了。


好了，基本原因已经分析的够清楚了，我觉的有2点注意事项：

- `dubbo`原来注解实现声明周期没搞清楚
- `spring`如果能提前判断循环引用获取`exposedObject`也没问题

``` java
if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
	if (earlySingletonReference != null) {
		if (exposedObject == bean) {
			exposedObject = earlySingletonReference;
		}
}
```
比如把循环引用逻辑提前到`populateBean`之前判断一下。

### 为什么在开源`dubbo 2.5.8`版本之后没有这个问题？

重写后的注解实现，我深入研究过，新版本实现不会错误的提前使用没有代理的bean，关键代码：

```
// ServiceAnnotationBeanPostProcessor
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

	Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

	if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
		registerServiceBeans(resolvedPackagesToScan, registry);
	} else {
		 if (logger.isWarnEnabled()) {
				 logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
			}
	}
}

```

`ServiceAnnotationBeanPostProcessor` 实现`BeanDefinitionRegistryPostProcessor`接口，会在所有`spring bean`真正初始化前完成`dubbo`服务的注册，整个生命周期中不会触碰到代理前的对象。

### 总结

我其实平时不太习惯写文章，但是发现分析后的问题记录下来可以让更多同学收益。
