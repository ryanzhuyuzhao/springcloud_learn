编程式使用IoC容器

```java
ClassPathResource res = new ClassPathResource();
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

1）创建IoC配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息

2）创建一个BeanFactory，这里使用DefaultListableBeanFactory

3）创建一个载入BeanDefinition的读取器，这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition，通过一个 回调配置给BeanFactory

4）从定义好的资源位置读入信息，具体的解析过程由XmlBeanDefinitionReader来完成。

完成整个载入和注册Bean定义之后，需要的IoC容器就建立起来了。这个时候就可以直接使用IoC容器了。



IoC容器的初始化：定位、载入和注册。

FileSystemXmlApplicationContext类中构造方法中的refresh()在容器初始化中是非常重要的一个入口，在分析IoC容器初始化中，这是一个药着重分析的方法。

```java
public FileSystemXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```

refresh()的代码在AbstractApplicationContext类中具体实现，具体代码如下所示：

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
			// Prepare this context for refreshing.
			prepareRefresh();
			// 这里是在子类中启动refreshFactory()的地方
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);
			try {
				// 设置BeanFactory的后置处理
				postProcessBeanFactory(beanFactory);
				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// 调用BeanFactory的后置处理器，这些后处理器是在Bean定义中向容器注册的
				invokeBeanFactoryPostProcessors(beanFactory);
				// 注册Bean的后处理器，在Bean创建过程中调用
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();
				// 对上下文中的消息源进行初始化
				initMessageSource();
				// 初始化上下文中的事件机制
				initApplicationEventMulticaster();
				// 初始化其他的特殊Bean
				onRefresh();
				// 检查监听Bean并且将这些Bean向容器注册
				registerListeners();
				// 实例化所有的（non-lazy-init）单件
				finishBeanFactoryInitialization(beanFactory);
				// 发布容器事件，结束Refresh过程
				finishRefresh();
			}
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}
				// 为防止Bean资源占用，在异常处理中，销毁已经在前面过程中生成的单件Bean
				destroyBeans();
				// 重置'active'标志
				cancelRefresh(ex);
				// Propagate exception to caller.
				throw ex;
			}
			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
				contextRefresh.end();
			}
		}
	}
```

