# spring-boot加载自定义的配置文件

## 原理

实现EnvironmentPostProcessor接口，并且在spring-factories文件中加入org.springframework.boot.env.EnvironmentPostProcessor=

org.springframework.context.ApplicationListener=

## 流程

第一步：我们第一个要关注的就是spring boot首先初始化了一个全局的事件监听器，这个事件监听器会伴随着springboot的整个生命周期，这个我们以后也会多次接触这个组件，即是

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting();
```

初始化全局的事件监听器EventPublishingRunListener，如图

![image-20220810185248343](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101852253.png)

第二步：接下来就是开始准备springboot所有配置文件存储的仓库Environment，这个其实也很好理解，spring是管理bean的，bean里面也有很多属性，所以优先收集整个上下文的配置属性信息，将其放在一个Environment里面，然后以后想要什么，就从环境里面去获取。

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
```

第三步: 我们进入prepareEnvironment方法，关注它是如何实现。

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
    	// 创建环境
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
    	// 配置环境上下文
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        ConfigurationPropertySources.attach((Environment)environment);
        listeners.environmentPrepared((ConfigurableEnvironment)environment);
        this.bindToSpringApplication((ConfigurableEnvironment)environment);
        if (!this.isCustomEnvironment) {
            environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
        }

        ConfigurationPropertySources.attach((Environment)environment);
        return (ConfigurableEnvironment)environment;
    }
```

我们可以看到首先先调用getOrCreateEnvironment创建好一个上下文环境，接着看下一行的#configureEnvironment方法。

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
		if (this.addConversionService) {
			ConversionService conversionService = ApplicationConversionService.getSharedInstance();
			environment.setConversionService((ConfigurableConversionService) conversionService);
		}
        
        //加载一些基本的环境变量
		configurePropertySources(environment, args);
		configureProfiles(environment, args);
	}
```

主要是加载如下的配置文件，运行时一般会配置启动参数，其实也就是在这个地方，**启动参数被会spring boot解析到作为默认选项，加载到上下文中，作为启动的核心参数启动，但是一般这些参数叫做默认参数，优先级是最低的，如果你在代码有同key值的时候，就会覆盖运行配置的系统级变量值。**

![image-20220810185629623](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101856522.png)

然后通过： listeners.environmentPrepared(environment);

我们可以看到方法名叫做environmentPrepared，调用者是listener，是一个监听器，也就是我们上文说的EventPublishingRunListener监听器。

依次追踪代码到了ConfigFileApplicationListener的监听器实例，它监听的是一个ApplicationEnvironmentPreparedEvent，故其名曰"应用环境准备好事件"，虽然有点绕口，也不通顺，但是看到这边我们就动了，springboot是靠一种事件订阅的方式来做解耦合的，源码如下

![image-20220810185719894](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101857044.png)

紧接着我们进入onApplicationEnvironmentPreparedEvent这个方法

```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
		postProcessors.add(this);
		AnnotationAwareOrderComparator.sort(postProcessors);
		for (EnvironmentPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
		}
	}
```

继续追查代码怎么拿到postprocessors

```java
List<EnvironmentPostProcessor> loadPostProcessors() {
		return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class, getClass().getClassLoader());
	}
```

![image-20220810185947989](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101859782.png)

然后继续查看SpringFactoriesLoader

![image-20220810190059205](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101900927.png)

![image-20220810190109640](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101901482.png)

然后可以看到自定义环境postprocessors

![image-20220810190257070](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101902929.png)

可以看到当loadPostProcessors执行完之后，看方法名我们也是是加载当前项目中EnvironmentPostProcessor,然后排序，最后调用我们刚刚说的postProcessorEnvironment方法

```java
for (EnvironmentPostProcessor postProcessor : postProcessors) {
	postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
}
```

即我们实现的EnvironmentPostProcessor的类

![image-20220810190500006](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202208101905781.png)

至此自定义的配置文件加载完成