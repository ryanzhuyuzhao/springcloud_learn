### 1.服务注册

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

```java
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

```java
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



说完eureka客户端是怎么进行注册的现在我们来看一看eureka服务端是怎么进行相关注册的。源码eureka-core包下的EurekaBootStrap类继承了ServletContextListener。EurekaBootStrap在容器启动时进行相关初始化。eureka服务端的相关服务注册也是在EurekaBootStrap进行初始化的。

在初始化eureka服务器上下文的方法initEurekaServerContext中我们可以看到PeerAwareInstanceRegistry这么一个类，单从名字上看这个类应该是和注册实例有关的一个类。进到这个类里果然有register这个方法。接下来，让我们看下这个方法的具体实现。在抽象类AbstractInstanceRegistry里有register方法的具体实现。

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        read.lock();
        try {
        	//首先通过app应用名获取对应的注册信息
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            //服务注册数加1，服务注册数是用AtomicLong来实现的避免多线程出现数据不同步问题
            REGISTER.increment(isReplication);
            if (gMap == null) {//如果注册信息在原有的注册数据中没有则将注册信心存入ConcurrentHashMap数据中
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                //如果existingLastDirtyTimestamp > registrationLastDirtyTimestamp注册信息没有更新信息就获取原有的数据
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
            	//如果注册信息是以前不存的的，就更新注册信息
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // Since the client wants to register it, increase the number of clients sending renews
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            //将注册服务名称和Id存储到队列中
            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            //将信息存入最近改变队列中
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
```

PeerAwareInstanceRegistry的register()方法，我们通过代码追踪可以看到最终是被ApplicationResource类的addInstance方法调用。ApplicationResource类是一个Http API接口。此类里的方法通过Http接口暴露出来供eureka client调用。而addInstance方法是用来注册eureka服务的。