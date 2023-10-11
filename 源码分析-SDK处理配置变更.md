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
                    
                    //check local listeners consistent.
                    if (cache.isConsistentWithServer()) {
                        // 如果和服务端同步则检查listener的md5和cacheData中的是否一致
                        // 如果不一致则通知
                        cache.checkListenerMd5();
                        // 然后跳过全部同步
                        if (!needAllSync) {
                            continue;
                        }
                    }
                    
                    if (!cache.isDiscard()) {
                        //get listen  config
                        if (!cache.isUseLocalConfigInfo()) {
                            List<CacheData> cacheDatas = listenCachesMap.computeIfAbsent(
                                    String.valueOf(cache.getTaskId()), k -> new LinkedList<>());
                            cacheDatas.add(cache);
                            
                        }
                    } else if (cache.isDiscard() && CollectionUtils.isEmpty(cache.getListeners())) {
                        
                        if (!cache.isUseLocalConfigInfo()) {
                            List<CacheData> cacheDatas = removeListenCachesMap.computeIfAbsent(
                                    String.valueOf(cache.getTaskId()), k -> new LinkedList<>());
                            cacheDatas.add(cache);
                            
                        }
                    }
                }
                
            }
            
            //execute check listen ,return true if has change keys.
            boolean hasChangedKeys = checkListenCache(listenCachesMap);
            
            //execute check remove listen.
            checkRemoveListenCache(removeListenCachesMap);

            if (needAllSync) {
                lastAllSyncTime = now;
            }
            //If has changed keys,notify re sync md5.
            if (hasChangedKeys) {
                notifyListenConfig();
            }

        }
```

