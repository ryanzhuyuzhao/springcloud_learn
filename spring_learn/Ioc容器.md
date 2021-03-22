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



IoC容器的初始化：载入、定位和注册。