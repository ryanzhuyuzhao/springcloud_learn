与上节服务注册一样我们通过eureka client与eureka server来分析代码，服务续约我们也通过这两个工程来分析代码。

首先，我们来分析eureka client包下的DiscoveryClient。DiscoveryClient类中的renew()方法是负责用来续约的，通过发送心跳到server端。具体代码如下。调用sendHeartBeat方法向server发送心跳，如果server端返回的是NOT_FOUND状态则方法调用register()方法对当前eureka应用进行注册。

通过代码追踪，renew()方法是DiscoveryClient里的HeartbeatThread线程类的run()方法调用。在DiscoveryClient的initScheduledTasks()初始化调度任务的方法中，我们看以找到对心跳任务heartbeatTask的初始化。

```java
boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
    }
```

分析完eureka client的代码，我们知道了eureka客户端是怎么通过发起心跳来续约服务的，那现在，我们就要来分析被调用的服务端的代码是如何实现的了。在InstanceResource中我们可以找到renewLease()方法，renewLease通过http暴露接口供eureka client调用。服务的续约是在这个方法里完成的。我们来分析下具体的代码。

```java
public Response renewLease(
            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
            @QueryParam("overriddenstatus") String overriddenStatus,
            @QueryParam("status") String status,
            @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
        boolean isFromReplicaNode = "true".equals(isReplication);
    	//具体的服务续约调用
        boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

        // Not found in the registry, immediately ask for a register
        if (!isSuccess) {
            logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
            return Response.status(Status.NOT_FOUND).build();
        }
    	//续约成功后，检查服务是否要更新相关注册信息
        // Check if we need to sync based on dirty time stamp, the client
        // instance might have changed some value
        Response response;
        if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
            response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
            // Store the overridden status since the validation found out the node that replicates wins
            if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
                    && (overriddenStatus != null)
                    && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
                    && isFromReplicaNode) {
                registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
            }
        } else {
            response = Response.ok().build();
        }
        logger.debug("Found (Renew): {} - {}; reply status={}", app.getName(), id, response.getStatus());
        return response;
    }
```

而具体的服务续约逻辑则是在registry.renew(app.getName(), id, isFromReplicaNode)调用时完成的。我们来进到renew()方法实现中查看具体的逻辑。

```
public boolean renew(String appName, String id, boolean isReplication) {
        RENEW.increment(isReplication);
        //通过app名获取相关注册信息
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToRenew = null;
        if (gMap != null) {
        	//通过id获取续约实体数据Lease
            leaseToRenew = gMap.get(id);
        }
        //如果Lease数据为空则返回false
        if (leaseToRenew == null) {
            RENEW_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
            return false;
        } else {
            InstanceInfo instanceInfo = leaseToRenew.getHolder();
            if (instanceInfo != null) {
                // touchASGCache(instanceInfo.getASGName());
                InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                        instanceInfo, leaseToRenew, isReplication);
                //如果续约Lease中的注册状态为UNKNOWN未知则返回false
                if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                    logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                            + "; re-register required", instanceInfo.getId());
                    RENEW_NOT_FOUND.increment(isReplication);
                    return false;
                }
                //如果续约Lease中的状态和服务端保存的状态不一致则更新服务端的状态
                if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                    logger.info(
                            "The instance status {} is different from overridden instance status {} for instance {}. "
                                    + "Hence setting the status to overridden status", instanceInfo.getStatus().name(),
                                    overriddenInstanceStatus.name(),
                                    instanceInfo.getId());
                    instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);

                }
            }
            renewsLastMin.increment();
            leaseToRenew.renew();
            return true;
        }
    }
```

