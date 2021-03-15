1.用git下载eureka源代码，

git地址为：https://github.com/Netflix/eureka.git

idea搭建eureka不通过需要更改build.gradle文件中的仓库详情可见

https://www.jianshu.com/p/28d719b499ec

具体配置文件在同目录下的build.gradle文件中                                           



重装系统

https://windows.dqsspx.top/win1064.html



idea激活码

http://lookdiv.com/index/index/indexcodeindex.html

7788

code-iris idea类继承关系插件



EurekaHttpResponse存储eureka注册的相关信息

statusCode 状态码信息

headers 头部信息

location URI地址信息



> ![image-20210312110626805](C:\Users\kj00078\AppData\Roaming\Typora\typora-user-images\image-20210312110626805.png)



在eureka-client包中可以看到DiscoveryClient类。DiscoveryClient用于服务发现与服务注册，DiscoverClient类的继承关系如下图所示：



![DiscoveryClient类关系图](C:\Users\kj00078\Desktop\DiscoveryClient类关系图.png)

DiscoveryClient类中的register是用来服务注册的，该方法通过http请求向Eureka Server注册。

```
/**
     * Register with the eureka service by making the appropriate REST call.
     */
    boolean register() throws Throwable {
        logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
        //负责存放eureka注册的相关信息，状态码，headers头部信息，uri地址
        EurekaHttpResponse<Void> httpResponse;
        try {
        	//此处调用是eureka注册的具体逻辑,AbstractJerseyEurekaHttpClient类中实现了register方法，实现方法中是使用的Jersey框架来实现eureka客               户端与服务端之间的http通信,也可以自己实现EurekaHttpClient中的register方法
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
```



![EurekaHttpClient类图](C:\Users\kj00078\Desktop\EurekaHttpClient类图.png)

EurekaHttpClient类继承图



AbstractJerseyEurekaHttpClient类中实现的register方法具体代码如下：

```
public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
        	//使用jersey框架实现客户端与服务器之间的http通信
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }
```

分析到这里，我们怎能看到InstanceInfo这个类的身影，它出现的如此频繁，此类必定不是碌碌无为之辈。这个类在eureka注册中起到的作用非常关键，它是一系列注册数据存放的实体。限于篇幅我们在这里就不具体讲解这个类了。有兴趣的同学可以到eureka学习之InstanceInfo去了解。

好了我们在回来继续分析DiscoveryClient类里的register方法，通过追踪方法我们可以看到InstanceInfoReplicator类里的run方法调用了register方法。通过类描述，A task for updating and replicating the local instanceinfo to the remote server.这个类是负责更新和复制本地eureka相关数据到远程server服务端的。以下是run方法调用register方法的具体代码。而InstanceInfoReplicator实例是在DiscoveryClient的方法initScheduledTasks中new创造对象的。initScheduledTasks方法是在eureka客户端用来初始化所有调度任务对象的。

```
public void run() {
        try {
            discoveryClient.refreshInstanceInfo();
			//查询是否和服务器存储的信息是否有更新
            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            //dirtyTimestamp如果不为null则调用register()方法进行注册动作
            if (dirtyTimestamp != null) {
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
```

