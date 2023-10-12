# 

```java
public class ClientWorker implements Closeable {
    private void initRpcClientHandler(final RpcClient rpcClientInner) {
        /*
         * Register Config Change /Config ReSync Handler
         */
        rpcClientInner.registerServerRequestHandler((request, connection) -> {
            if (request instanceof ConfigChangeNotifyRequest) {
                ConfigChangeNotifyRequest configChangeNotifyRequest = (ConfigChangeNotifyRequest) request;
                LOGGER.info("[{}] [server-push] config changed. dataId={}, group={},tenant={}",
                            rpcClientInner.getName(), configChangeNotifyRequest.getDataId(),
                            configChangeNotifyRequest.getGroup(), configChangeNotifyRequest.getTenant());
                // 获取对应的groupKey
                String groupKey = GroupKey.getKeyTenant(configChangeNotifyRequest.getDataId(),
                                                        configChangeNotifyRequest.getGroup(), configChangeNotifyRequest.getTenant());

                CacheData cacheData = cacheMap.get().get(groupKey);
                if (cacheData != null) {
                    synchronized (cacheData) {
                        // 将cacheData标记为已经发生变化
                        cacheData.getReceiveNotifyChanged().set(true);
                        cacheData.setConsistentWithServer(false);
                        // 然后再触发listenConfig
                        notifyListenConfig();
                    }

                }
                return new ConfigChangeNotifyResponse();
            }
            return null;
        });
        // 其他逻辑处理
        ...
    }
}
```

当配置发生变更时，通知客户端首先将cacheData标记为和服务端不一致，然后再通知监听配置。

```java
        public void startInternal() {
            executor.schedule(() -> {
                while (!executor.isShutdown() && !executor.isTerminated()) {
                    try {
                        // 每隔5秒执行一次配置监听
                        listenExecutebell.poll(5L, TimeUnit.SECONDS);
                        if (executor.isShutdown() || executor.isTerminated()) {
                            continue;
                        }
                        executeConfigListen();
                    } catch (Throwable e) {
                        LOGGER.error("[rpc listen execute] [rpc listen] exception", e);
                        try {
                            Thread.sleep(50L);
                        } catch (InterruptedException interruptedException) {
                            //ignore
                        }
                        notifyListenConfig();
                    }
                }
            }, 0L, TimeUnit.MILLISECONDS);
            
        }
```





```java
		@Override
        public void executeConfigListen() {
            
            Map<String, List<CacheData>> listenCachesMap = new HashMap<>(16);
            Map<String, List<CacheData>> removeListenCachesMap = new HashMap<>(16);
            long now = System.currentTimeMillis();
            boolean needAllSync = now - lastAllSyncTime >= ALL_SYNC_INTERNAL;
            for (CacheData cache : cacheMap.get().values()) {
                
                synchronized (cache) {
                    // 如果与服务端一致，那么只需要检查cache中的md5与listener中的md5是否一致就好了，
                    // 在checkListenerMd5()中，如果不一致则会进行通知
                    // 如果说cache与服务端一致，并且距离最后一次同步没有达到需要全部同步的间隔，那么则跳过该cache的处理
                    //check local listeners consistent.
                    if (cache.isConsistentWithServer()) {
                        cache.checkListenerMd5();
                        if (!needAllSync) {
                            continue;
                        }
                    }
                    
                    // 判断缓存数据是否应该丢弃，当前关于cache的discard属性有几点赋值的地方
                    // 1、当新增listener时，那么肯定将discard = true
                    // 2、当移除listener时，并且listener为空，那么肯定将discard = false
                    if (!cache.isDiscard()) {
                        //get listen  config
                        // 判断cache是否使用本地的配置信息
                        // 如果没有使用本地的配置信息，那么则将cache加入listenCachesMap中
                        if (!cache.isUseLocalConfigInfo()) {
                            List<CacheData> cacheDatas = listenCachesMap.computeIfAbsent(
                                    String.valueOf(cache.getTaskId()), k -> new LinkedList<>());
                            cacheDatas.add(cache);
                            
                        }
                    } else if (cache.isDiscard() && CollectionUtils.isEmpty(cache.getListeners())) {
                        // 如果说cache是使用本地的配置，并且listeners为空，那么将cache加入removeListenCachesMap中
                        if (!cache.isUseLocalConfigInfo()) {
                            List<CacheData> cacheDatas = removeListenCachesMap.computeIfAbsent(
                                    String.valueOf(cache.getTaskId()), k -> new LinkedList<>());
                            cacheDatas.add(cache);
                            
                        }
                    }
                }
                
            }
            
            // 判断有listen的是否有内容改变
            //execute check listen ,return true if has change keys.
            boolean hasChangedKeys = checkListenCache(listenCachesMap);
            
            // 执行remove cacheData
            //execute check remove listen.
            checkRemoveListenCache(removeListenCachesMap);
			
            // 更新最后同步时间
            if (needAllSync) {
                lastAllSyncTime = now;
            }
            
            // 如果说有改变的key，那么重新同步md5，其实这里我初看也有一些疑问，明明上面已经对md5进行改变了，为什么这里还需要重新触发？
            //If has changed keys,notify re sync md5.
            if (hasChangedKeys) {
                notifyListenConfig();
            }

        }
```





```java
	private boolean checkListenCache(Map<String, List<CacheData>> listenCachesMap) {
            final AtomicBoolean hasChangedKeys = new AtomicBoolean(false);
            if (listenCachesMap.isEmpty()) {
                return false;
            }
            List<Future> listenFutures = new ArrayList<>();
            for (Map.Entry<String, List<CacheData>> entry : listenCachesMap.entrySet()) {
                String taskId = entry.getKey();
                ExecutorService executorService = ensureSyncExecutor(taskId);
                Future future = executorService.submit(() -> {
                    List<CacheData> listenCaches = entry.getValue();
                    //reset notify change flag.
                    // 重置接受改变通知的标识
                    for (CacheData cacheData : listenCaches) {
                        cacheData.getReceiveNotifyChanged().set(false);
                    }
                    // 将所有有listen的config发送请求到nacos server查询有改动的配置
                    ConfigBatchListenRequest configChangeListenRequest = buildConfigRequest(listenCaches);
                    configChangeListenRequest.setListen(true);
                    try {
                        RpcClient rpcClient = ensureRpcClient(taskId);
                        ConfigChangeBatchListenResponse listenResponse = (ConfigChangeBatchListenResponse) requestProxy(
                                rpcClient, configChangeListenRequest);
                        if (listenResponse != null && listenResponse.isSuccess()) {

                            Set<String> changeKeys = new HashSet<>();

                            List<ConfigChangeBatchListenResponse.ConfigContext> changedConfigs = listenResponse.getChangedConfigs();
                            // 如果改动的配置不为空，那么将hasChangeKeys置为true，并且去检查并且刷新内容
                            //handle changed keys,notify listener
                            if (!CollectionUtils.isEmpty(changedConfigs)) {
                                hasChangedKeys.set(true);
                                for (ConfigChangeBatchListenResponse.ConfigContext changeConfig : changedConfigs) {
                                    String changeKey = GroupKey.getKeyTenant(changeConfig.getDataId(),
                                            changeConfig.getGroup(), changeConfig.getTenant());
                                    changeKeys.add(changeKey);
                                    boolean isInitializing = cacheMap.get().get(changeKey).isInitializing();
                                    refreshContentAndCheck(changeKey, !isInitializing);
                                }

                            }
							
                            // 遍历所有的listenCache，上面按道理都将cacheData.receiveNotifyChanged置为false
                            // 如果说cacheData.receiveNotifyChanged == true，那么说明接收到了服务器的配置变更请求
                            // 那么则判断changeKeys是否包含changeKey，如果说已经包含了，那么说明其实在上一步操作中就已经refresh了，这里就没必要重复refresh了
                            // 当然这里还包含一种情况，就是说是在refresh之后再接收到配置变更的请求，那么changeKeys也包含changeKey，那么就只能等下一次执行了~
                            for (CacheData cacheData : listenCaches) {
                                if (cacheData.getReceiveNotifyChanged().get()) {
                                    String changeKey = GroupKey.getKeyTenant(cacheData.dataId, cacheData.group,
                                            cacheData.getTenant());
                                    if (!changeKeys.contains(changeKey)) {
                                        boolean isInitializing = cacheMap.get().get(changeKey).isInitializing();
                                        refreshContentAndCheck(changeKey, !isInitializing);
                                    }
                                }
                            }
							
                            // 将所有的配置的initializing状态置为false
                            // 如果这个cacheData没有发生改变，并且没有接收到服务器的配置变更，那么则将consistentWithServer = true，
                            // 代表与服务器同步
                            //handler content configs
                            for (CacheData cacheData : listenCaches) {
                                cacheData.setInitializing(false);
                                String groupKey = GroupKey.getKeyTenant(cacheData.dataId, cacheData.group,
                                        cacheData.getTenant());
                                if (!changeKeys.contains(groupKey)) {
                                    synchronized (cacheData) {
                                        if (!cacheData.getReceiveNotifyChanged().get()) {
                                            cacheData.setConsistentWithServer(true);
                                        }
                                    }
                                }
                            }

                        }
                    } catch (Throwable e) {
                        LOGGER.error("Execute listen config change error ", e);
                        try {
                            Thread.sleep(50L);
                        } catch (InterruptedException interruptedException) {
                            //ignore
                        }
                        // 如果在执行过程中，发生异常，那么重新执行
                        notifyListenConfig();
                    }
                });
                listenFutures.add(future);

            }
            // 等待futrue执行完成
            for (Future future : listenFutures) {
                try {
                    future.get();
                } catch (Throwable throwable) {
                    LOGGER.error("Async listen config change error ", throwable);
                }
            }
            // 返回是否有配置变更
            return hasChangedKeys.get();
        }
```



```java
	private void checkRemoveListenCache(Map<String, List<CacheData>> removeListenCachesMap) {
            if (!removeListenCachesMap.isEmpty()) {
                List<Future> listenFutures = new ArrayList<>();

                for (Map.Entry<String, List<CacheData>> entry : removeListenCachesMap.entrySet()) {
                    String taskId = entry.getKey();

                    ExecutorService executorService = ensureSyncExecutor(taskId);
                    Future future = executorService.submit(() -> {
                        List<CacheData> removeListenCaches = entry.getValue();
                        // 通知服务端，不再进行监听
                        ConfigBatchListenRequest configChangeListenRequest = buildConfigRequest(removeListenCaches);
                        configChangeListenRequest.setListen(false);
                        try {
                            RpcClient rpcClient = ensureRpcClient(taskId);
                            boolean removeSuccess = unListenConfigChange(rpcClient, configChangeListenRequest);
                            if (removeSuccess) {
                                for (CacheData cacheData : removeListenCaches) {
                                    synchronized (cacheData) {
                                        // 如果cacheData已经丢弃，并且listener为空，那么代表sdk不需要在存储该缓存，直接进行移除
                                        if (cacheData.isDiscard() && cacheData.getListeners().isEmpty()) {
                                            ClientWorker.this.removeCache(cacheData.dataId, cacheData.group,
                                                    cacheData.tenant);
                                        }
                                    }
                                }
                            }

                        } catch (Throwable e) {
                            LOGGER.error("Async remove listen config change error ", e);
                            try {
                                Thread.sleep(50L);
                            } catch (InterruptedException interruptedException) {
                                //ignore
                            }
                            notifyListenConfig();
                        }
                    });
                    listenFutures.add(future);

                }
                // 等待removeListen执行完成
                for (Future future : listenFutures) {
                    try {
                        future.get();
                    } catch (Throwable throwable) {
                        LOGGER.error("Async remove listen config change error ", throwable);
                    }
                }
            }
        }
```



