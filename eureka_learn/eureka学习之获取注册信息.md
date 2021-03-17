eureka客户端需要从server服务器获取所有注册的服务信息，通过这些信息客户端才能通过server去调用其他的服务。那eureka客户端是如何获取这些注册信息的呢？逻辑上客户端也是通过调用server提供的API接口来获取server端返回的所有的应用的注册信息。那在源码上市如何实现的呢？

具体的实现还是在DiscoveryClient中实现的，由此可以看出DiscoveryClient在eureka客户端中的重要性，几乎所有的逻辑实现（包括注册、续约、获取列表信息、下线）都是在这个类里实现的。而获取注册信息是在fetchRegistry()这个方法中实现的。fetchRegistry源码如下所示

```
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", (applications == null));
                logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
                //服务首次注册全量获取所有的应用的注册信息
                getAndStoreFullRegistry();
            } else {
            	//服务不是首次注册则增量的获取相关应用注册信息并在本地更新应用注册信息
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.info(PREFIX + "{} - was unable to refresh its cache! This periodic background refresh will be retried in {} seconds. status = {} stacktrace = {}",
                    appPathIdentifier, clientConfig.getRegistryFetchIntervalSeconds(), e.getMessage(), ExceptionUtils.getStackTrace(e));
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();

        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
```

