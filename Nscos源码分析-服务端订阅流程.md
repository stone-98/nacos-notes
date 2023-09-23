# 服务订阅分析

服务订阅故名思意，客户端对服务端的服务进行订阅，当服务信息发送改变时，通知客户端。



## 服务端分析

服务端的核心处理逻辑在SubscribeServiceRequestHandler中，核心源码如下所示：

```java
    public SubscribeServiceResponse handle(SubscribeServiceRequest request, RequestMeta meta) throws NacosException {
        String namespaceId = request.getNamespace();
        String serviceName = request.getServiceName();
        String groupName = request.getGroupName();
        String app = request.getHeader("app", "unknown");
        String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
        // 首先，先为订阅过来的请求创建一个Service
        Service service = Service.newService(namespaceId, groupName, serviceName, true);
        // 创建一个Subscriber对象
        Subscriber subscriber = new Subscriber(meta.getClientIp(), meta.getClientVersion(), app, meta.getClientIp(),
                namespaceId, groupedServiceName, 0, request.getClusters());
        // serviceStorage.getData(service)通过服务实体做唯一标识，去获取服务的所有信息，例如实例信息等等...
        // 然后通过MetaDataManager获取相关的元数据
        // 集群信息、ip等等
        // 然后过滤出健康的服务的实例信息
        ServiceInfo serviceInfo = ServiceUtil.selectInstancesWithHealthyProtection(serviceStorage.getData(service),
                metadataManager.getServiceMetadata(service).orElse(null), subscriber.getCluster(), false,
                true, subscriber.getIp());
        // 分别进行订阅或者注销订阅
        if (request.isSubscribe()) {
            // 依托EphemeralClientOperationServiceImpl进行订阅
            clientOperationService.subscribeService(service, subscriber, meta.getConnectionId());
            // 发布trace event
            NotifyCenter.publishEvent(new SubscribeServiceTraceEvent(System.currentTimeMillis(),
                    meta.getClientIp(), service.getNamespace(), service.getGroup(), service.getName()));
        } else {
            // 依托EphemeralClientOperationServiceImpl注销订阅
            clientOperationService.unsubscribeService(service, subscriber, meta.getConnectionId());
            // 发布trace event
            NotifyCenter.publishEvent(new UnsubscribeServiceTraceEvent(System.currentTimeMillis(),
                    meta.getClientIp(), service.getNamespace(), service.getGroup(), service.getName()));
        }
        return new SubscribeServiceResponse(ResponseCode.SUCCESS.getCode(), "success", serviceInfo);
    }
```

上述的逻辑进行总结：

- 首先获取服务的相关信息（例如：服务、实例、集群、元数据等等），作为响应进行返回
- 分别发布了订阅Trace事件和注册订阅Trace事件
- 核心逻辑还是在EphemeralClientOperationServiceImpl当中

接下来继续EphemeralClientOperationServiceImpl的逻辑分析，源码如下：

```java
    @Override
    public void subscribeService(Service service, Subscriber subscriber, String clientId) {
        // 从Nacos Server中获取到Service、Client
        Service singleton = ServiceManager.getInstance().getSingletonIfExist(service).orElse(service);
        Client client = clientManager.getClient(clientId);
        // 判断客户端的合法性
        if (!clientIsLegal(client, clientId)) {
            return;
        }
        // 将订阅者相关消息存储到Client中
        client.addServiceSubscriber(singleton, subscriber);
        // 更新最后更新事件
        client.setLastUpdatedTime();
        // 发布客户端订阅事件
        NotifyCenter.publishEvent(new ClientOperationEvent.ClientSubscribeServiceEvent(singleton, clientId));
    }
    
    @Override
    public void unsubscribeService(Service service, Subscriber subscriber, String clientId) {
        Service singleton = ServiceManager.getInstance().getSingletonIfExist(service).orElse(service);
        Client client = clientManager.getClient(clientId);
        if (!clientIsLegal(client, clientId)) {
            return;
        }
        client.removeServiceSubscriber(singleton);
        client.setLastUpdatedTime();
        NotifyCenter.publishEvent(new ClientOperationEvent.ClientUnsubscribeServiceEvent(singleton, clientId));
    }
```

我这里以订阅服务为例，因为注销大致思路一致，只要你理解了订阅服务的逻辑，看注销的流程也没有什么难点了。

ClientOperationEvent.ClientSubscribeServiceEvent是ClientOperationEvent的子类，所以当发布ClientOperationEvent.ClientSubscribeServiceEvent时，ClientServiceIndexesManager将会接收，源码如下所示：

```java
    public void onEvent(Event event) {
        if (event instanceof ClientOperationEvent.ClientReleaseEvent) {
            handleClientDisconnect((ClientOperationEvent.ClientReleaseEvent) event);
        } else if (event instanceof ClientOperationEvent) {
            handleClientOperation((ClientOperationEvent) event);
        }
    }

    private void handleClientOperation(ClientOperationEvent event) {
        Service service = event.getService();
        String clientId = event.getClientId();
        if (event instanceof ClientOperationEvent.ClientRegisterServiceEvent) {
            addPublisherIndexes(service, clientId);
        } else if (event instanceof ClientOperationEvent.ClientDeregisterServiceEvent) {
            removePublisherIndexes(service, clientId);
        } else if (event instanceof ClientOperationEvent.ClientSubscribeServiceEvent) {
            // 处理订阅事件
            addSubscriberIndexes(service, clientId);
        } else if (event instanceof ClientOperationEvent.ClientUnsubscribeServiceEvent) {
            removeSubscriberIndexes(service, clientId);
        }
    }

    private void addSubscriberIndexes(Service service, String clientId) {
        // 以<Service, Set<ClientId>>的形式存储订阅者
        // 然后发布服务订阅事件
        subscriberIndexes.computeIfAbsent(service, key -> new ConcurrentHashSet<>());
        // Fix #5404, Only first time add need notify event.
        if (subscriberIndexes.get(service).add(clientId)) {
            NotifyCenter.publishEvent(new ServiceEvent.ServiceSubscribedEvent(service, clientId));
        }
    }
```

对上述逻辑进行总结：

- 将服务订阅以<Service, Set<ClientId>>形式进行存储
- 然后发布服务订阅事件

发布服务订阅事件后，将会回调到NamingSubscriberServiceV2Impl中，源码如下所示：

```java
    @Override
    public void onEvent(Event event) {
        if (event instanceof ServiceEvent.ServiceChangedEvent) {
            // If service changed, push to all subscribers.
            ServiceEvent.ServiceChangedEvent serviceChangedEvent = (ServiceEvent.ServiceChangedEvent) event;
            Service service = serviceChangedEvent.getService();
            delayTaskEngine.addTask(service, new PushDelayTask(service, PushConfig.getInstance().getPushTaskDelay()));
            MetricsMonitor.incrementServiceChangeCount(service.getNamespace(), service.getGroup(), service.getName());
        } else if (event instanceof ServiceEvent.ServiceSubscribedEvent) {
            // If service is subscribed by one client, only push this client.
            ServiceEvent.ServiceSubscribedEvent subscribedEvent = (ServiceEvent.ServiceSubscribedEvent) event;
            Service service = subscribedEvent.getService();
            // 给订阅请求一个回调？
            delayTaskEngine.addTask(service, new PushDelayTask(service, PushConfig.getInstance().getPushTaskDelay(),
                    subscribedEvent.getClientId()));
        }
    }
```

然后再交给PushExecuteTask去执行具体的任务。

```java
    @Override
    public void run() {
        try {
            // 生成需要push的数据
            PushDataWrapper wrapper = generatePushData();
            ClientManager clientManager = delayTaskEngine.getClientManager();
            for (String each : getTargetClientIds()) {
                Client client = clientManager.getClient(each);
                if (null == client) {
                    // means this client has disconnect
                    continue;
                }
                // 获取订阅者
                Subscriber subscriber = client.getSubscriber(service);
                // skip if null
                if (subscriber == null) {
                    continue;
                }
                // 执行带回调的推送
                delayTaskEngine.getPushExecutor().doPushWithCallback(each, subscriber, wrapper,
                        new ServicePushCallback(each, subscriber, wrapper.getOriginalData(), delayTask.isPushToAll()));
            }
        } catch (Exception e) {
            Loggers.PUSH.error("Push task for service" + service.getGroupedServiceName() + " execute failed ", e);
            delayTaskEngine.addTask(service, new PushDelayTask(service, 1000L));
        }
    }
```

由PushExecutorRpcImpl去执行具体的带回调的推送：

```java
    @Override
    public void doPushWithCallback(String clientId, Subscriber subscriber, PushDataWrapper data,
            NamingPushCallback callBack) {
        ServiceInfo actualServiceInfo = getServiceInfo(data, subscriber);
        callBack.setActualServiceInfo(actualServiceInfo);
        // 推送带回调
        pushService.pushWithCallback(clientId, NotifySubscriberRequest.buildNotifySubscriberRequest(actualServiceInfo),
                callBack, GlobalExecutor.getCallbackExecutor());
    }
```

由RpcPushService去执行具体的推送：

```java
@Service
public class RpcPushService {
    
    @Autowired
    private ConnectionManager connectionManager;
    
    /**
     * push response with no ack.
     *
     * @param connectionId    connectionId.
     * @param request         request.
     * @param requestCallBack requestCallBack.
     */
    public void pushWithCallback(String connectionId, ServerRequest request, PushCallBack requestCallBack,
            Executor executor) {
        // 通过connectionId获取连接
        Connection connection = connectionManager.getConnection(connectionId);
        if (connection != null) {
            try {
                // 通过connection发送异步请求
                // 请求成功则打印相关日志
                // 请求失败则进行重试
                connection.asyncRequest(request, new AbstractRequestCallBack(requestCallBack.getTimeout()) {
                    
                    @Override
                    public Executor getExecutor() {
                        return executor;
                    }
                    
                    @Override
                    public void onResponse(Response response) {
                        if (response.isSuccess()) {
                            requestCallBack.onSuccess();
                        } else {
                            requestCallBack.onFail(new NacosException(response.getErrorCode(), response.getMessage()));
                        }
                    }
                    
                    @Override
                    public void onException(Throwable e) {
                        requestCallBack.onFail(e);
                    }
                });
            } catch (ConnectionAlreadyClosedException e) {
                connectionManager.unregister(connectionId);
                requestCallBack.onSuccess();
            } catch (Exception e) {
                Loggers.REMOTE_DIGEST
                        .error("error to send push response to connectionId ={},push response={}", connectionId,
                                request, e);
                requestCallBack.onFail(e);
            }
            // 如果连接为空，则说明客户端节点和服务端断开，那么客户端会发生重连，通过RedoService进行重新订阅，所以这里连接为空默认回调成功。
        } else {
            requestCallBack.onSuccess();
        }
    }
    
    /**
     * push response with no ack.
     *
     * @param connectionId connectionId.
     * @param request      request.
     */
    public void pushWithoutAck(String connectionId, ServerRequest request) {
        Connection connection = connectionManager.getConnection(connectionId);
        if (connection != null) {
            try {
                connection.request(request, 3000L);
            } catch (ConnectionAlreadyClosedException e) {
                connectionManager.unregister(connectionId);
            } catch (Exception e) {
                Loggers.REMOTE_DIGEST
                        .error("error to send push response to connectionId ={},push response={}", connectionId,
                                request, e);
            }
        }
    }
    
}
```

接下来看看PushExecuteTask的相关处理逻辑：

```java
private class ServicePushCallback implements NamingPushCallback {
        
        private final String clientId;
        
        private final Subscriber subscriber;
        
        private final ServiceInfo serviceInfo;
        
        /**
         * Record the push task execute start time.
         */
        private final long executeStartTime;
        
        private final boolean isPushToAll;
        
        /**
         * The actual pushed service info, the host list of service info may be changed by selector. Detail see
         * implement of {@link com.alibaba.nacos.naming.push.v2.executor.PushExecutor}.
         */
        private ServiceInfo actualServiceInfo;
        
        private ServicePushCallback(String clientId, Subscriber subscriber, ServiceInfo serviceInfo,
                boolean isPushToAll) {
            this.clientId = clientId;
            this.subscriber = subscriber;
            this.serviceInfo = serviceInfo;
            this.isPushToAll = isPushToAll;
            this.executeStartTime = System.currentTimeMillis();
            this.actualServiceInfo = serviceInfo;
        }
        
        @Override
        public long getTimeout() {
            return PushConfig.getInstance().getPushTaskTimeout();
        }
        
        @Override
        public void onSuccess() {
            long pushFinishTime = System.currentTimeMillis();
            long pushCostTimeForNetWork = pushFinishTime - executeStartTime;
            long pushCostTimeForAll = pushFinishTime - delayTask.getLastProcessTime();
            long serviceLevelAgreementTime = pushFinishTime - service.getLastUpdatedTime();
            // 打印push成功的相关日志
            if (isPushToAll) {
                Loggers.PUSH
                        .info("[PUSH-SUCC] {}ms, all delay time {}ms, SLA {}ms, {}, originalSize={}, DataSize={}, target={}",
                                pushCostTimeForNetWork, pushCostTimeForAll, serviceLevelAgreementTime, service,
                                serviceInfo.getHosts().size(), actualServiceInfo.getHosts().size(), subscriber.getIp());
            } else {
                Loggers.PUSH
                        .info("[PUSH-SUCC] {}ms, all delay time {}ms for subscriber {}, {}, originalSize={}, DataSize={}",
                                pushCostTimeForNetWork, pushCostTimeForAll, subscriber.getIp(), service,
                                serviceInfo.getHosts().size(), actualServiceInfo.getHosts().size());
            }
            PushResult result = PushResult
                    .pushSuccess(service, clientId, actualServiceInfo, subscriber, pushCostTimeForNetWork,
                            pushCostTimeForAll, serviceLevelAgreementTime, isPushToAll);
            // 发布一些trace事件
            NotifyCenter.publishEvent(getPushServiceTraceEvent(pushFinishTime, result));
            // 执行push成功的回调钩子，就是记录一些请求相关的监控指标
            PushResultHookHolder.getInstance().pushSuccess(result);
        }
        
        @Override
        public void onFail(Throwable e) {
            // 打印push失败的相关日志
            long pushCostTime = System.currentTimeMillis() - executeStartTime;
            Loggers.PUSH.error("[PUSH-FAIL] {}ms, {}, reason={}, target={}", pushCostTime, service, e.getMessage(),
                    subscriber.getIp());
            if (!(e instanceof NoRequiredRetryException)) {
                // 如果抛出的是可重试的异常，那么进行重试。
                Loggers.PUSH.error("Reason detail: ", e);
                delayTaskEngine.addTask(service,
                        new PushDelayTask(service, PushConfig.getInstance().getPushTaskRetryDelay(), clientId));
            }
            PushResult result = PushResult
                    .pushFailed(service, clientId, actualServiceInfo, subscriber, pushCostTime, e, isPushToAll);
            // 执行push失败的回调钩子，就是记录一些请求相关的监控指标
            PushResultHookHolder.getInstance().pushFailed(result);
        }
        
        public void setActualServiceInfo(ServiceInfo actualServiceInfo) {
            this.actualServiceInfo = actualServiceInfo;
        }
        
        private PushServiceTraceEvent getPushServiceTraceEvent(long eventTime, PushResult result) {
            return new PushServiceTraceEvent(eventTime, result.getNetworkCost(), result.getAllCost(),
                    result.getSla(), result.getSubscriber().getIp(), result.getService().getNamespace(),
                    result.getService().getGroup(), result.getService().getName(), result.getData().getHosts().size());
        }
        
    }
```

## 总结

### 流程图

TODO



TODO