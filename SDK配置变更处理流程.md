# `SDK`配置变更处理流程

## `SDK`配置变更处理流程

此文章主要讲解，当`SDK`监听的配置发生变更时，`Server`端发送`ConfigCHangeNotifyRequest`至`SDK`，`SDK`的处理流程图如下所示：

![`SDK`配置变更处理流程](https://skyxg.oss-cn-beijing.aliyuncs.com/images/202310232226852.png)

## 源码分析

首先`SDK`的入口在`ClientWorker#initRpcClientHandler`中，当`SDK`启动时，会调用该方法，将相关请求处理器进行注册，这里我们主要关注`ConfigChangeNotifyRequest`（配置变更通知请求）,主要的逻辑如下：

- 当配置发生变更时，`Server`端会发送`ConfigChangeNotifyRequest`（配置变更通知请求）通知客户端
- `SDK`端首先根据发生变更的配置的元数据组成`groupKey`，再从`ClientWorker#cacheMap`中获取到对应的`cahceMap`
- 并且将`cacheMap`的`receiveNotifyChanged == true` (用于标识是否接收变更的通知，当接收到服务端变更通知时，肯定是将该标识置为`true`)，`consistentWithServer == false`(用于标识是否与服务端一致，当接收到服务端变更通知时，肯定是与服务端信息不一致，所以置为`false`)
- 然后触发`ClientWorker#notifyListenConfig`

源码如下：

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
                // SDK端首先根据发生变更的配置的元数据组成groupKey
                String groupKey = GroupKey.getKeyTenant(configChangeNotifyRequest.getDataId(),
                                                        configChangeNotifyRequest.getGroup(), configChangeNotifyRequest.getTenant());
			   // 通过groupKey获取到CacheData
                CacheData cacheData = cacheMap.get().get(groupKey);
                if (cacheData != null) {
                    synchronized (cacheData) {
                        // 将cacheData标记为已接收通知改变
                        cacheData.getReceiveNotifyChanged().set(true);
                        // 将cacheData标记为与服务端不一致
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

`notifyListenConfig`的逻辑向`listenExecutebell`提供一个元素，`listenExecutebell`具体的工作流程如下：

- 当`ClientWorker`启动时，会启动一个定时任务，定时从`listenExecutebell`每隔拉取数据，如果超过`5s`没有元素，则执行`executeConfigListen`
- 当服务端配置发生变更时，通知`SDK`，此时`notifyListenConfig`的向`listenExecutebell`提供一个元素，那么立马会执行`executeConfigListen`

源码如下所示：

```java
        public void startInternal() {
            executor.schedule(() -> {
                while (!executor.isShutdown() && !executor.isTerminated()) {
                    try {
                        // 轮询从listenExecutebell拉取元素，超时时间5s
                        listenExecutebell.poll(5L, TimeUnit.SECONDS);
                        if (executor.isShutdown() || executor.isTerminated()) {
                            continue;
                        }
                        // 执行配置监听以及移除配置的相关处理
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

`executeConfigListen`大体就是执行、移除配置监听以及配置变更通知`listener`相关的操作，具体的流程如下：

- 首先通过`checkLocalConfig`处理故障转移等逻辑，判断此配置是否是使用本地的配置，设置设置`cacheData`相关的标识
- 判断`cacheData`是否与服务端同步，如果不同步那么肯定是需要进行处理的，如果同步那么检查`listener`的`md5`是否与`cacheData`的`md5`一致，不一致则需要触发`listener`并且更新`listener`的`md5`，然后再判断是否是需要全部同步，因为如果与服务端同步达到了全部同步的间隔，还是需要进行处理该`cacheData`
- 判断是否使用本地配置，如果说使用本地配置，那么则不需要接下来的处理，因为在上一步处理已经处理完了
- 然后判断是否丢弃，分别`add`至`listenCachesMap`、`removeListenCachesMap`
- 通过调用`Server`端检查`listenCachesMap`是否有没有内容改变，当内容发生改变时，通知`listener`，并且返回true，这里主要是对`listenCachesMap`进行处理
- 将`removeListenCachesMap`远程调用`Server`移除`listen`，并且从`CacheMap`中移除对应的缓存
- 更新最后同步的时间
- 如果说配置有变化，那么需要再次进行处理，这里主要原因是采用的是最终一致性，当配置发生变化时，短时间内可能获取到的时旧的配置，所以需要再次进行拉取一下，保证获取到最新的配置

```java
        @Override
        public void executeConfigListen() {
            
            Map<String, List<CacheData>> listenCachesMap = new HashMap<>(16);
            Map<String, List<CacheData>> removeListenCachesMap = new HashMap<>(16);
            long now = System.currentTimeMillis();
            boolean needAllSync = now - lastAllSyncTime >= ALL_SYNC_INTERNAL;
            for (CacheData cache : cacheMap.get().values()) {
                
                synchronized (cache) {
					
                    // 处理故障转移等逻辑，判断此配置是否是使用本地的配置，设置cacheData相关的标识
                    checkLocalConfig(cache);
					
                    // 如果与服务端一致，那么只需要检查cache中的md5与listener中的md5是否一致就好了，
                    // 在checkListenerMd5()中，如果不一致则会进行通知
                    // 如果说cache与服务端一致，并且距离最后一次同步没有达到需要全部同步的间隔，那么则跳过该cache的处理    
                    // check local listeners consistent.
                    if (cache.isConsistentWithServer()) {
                        // 检查listener的md5是否与cacheData的md5一致，不一致则需要触发listener并且更新listener的md5
                        cache.checkListenerMd5();
                        if (!needAllSync) {
                            continue;
                        }
                    }
    
                    // 如果使用的是本地的配置信息，那么其实不需要对该cacheData进行处理，上一步应该把本地配置处理完了
                    // If local configuration information is used, then skip the processing directly.
                    if (cache.isUseLocalConfigInfo()) {
                        continue;
                    }
    				
                    // 如果配置已经丢弃，那么则加入removeListenCachesMap，否则加入listenCachesMap
                    if (!cache.isDiscard()) {
                        List<CacheData> cacheDatas = listenCachesMap.computeIfAbsent(String.valueOf(cache.getTaskId()),
                                k -> new LinkedList<>());
                        cacheDatas.add(cache);
                    } else {
                        List<CacheData> cacheDatas = removeListenCachesMap.computeIfAbsent(
                                String.valueOf(cache.getTaskId()), k -> new LinkedList<>());
                        cacheDatas.add(cache);
                    }
                }
                
            }
            
            // 通过调用server端检查listenCachesMap是否有没有内容改变，当内容发生改变时，通知listener，并且返回true
            //execute check listen ,return true if has change keys.
            boolean hasChangedKeys = checkListenCache(listenCachesMap);
            
            // 将removeListenCachesMap远程调用Server移除listen，并且从CacheMap中移除对应的缓存
            //execute check remove listen.
            checkRemoveListenCache(removeListenCachesMap);
            
            // 如果这次处理是因为达到了需要全部同步的间隔，那么则更新最后同步的时间
            if (needAllSync) {
                lastAllSyncTime = now;
            }
            
            // 如果说配置有变化，那么需要再次进行处理，这里主要原因是采用的是最终一致性，当配置发生变化时，短时间内可能获取到的时旧的配置，所以需要再次进行拉取一下，保证获取到最新的配置
            //If has changed keys,notify re sync md5.
            if (hasChangedKeys) {
                notifyListenConfig();
            }
    
        }
		
		/**
		 * 主要是用于检查是否使用本地配置，并且更新cacheData对象
         * Checks and handles local configuration for a given CacheData object. This method evaluates the use of
         * failover files for local configuration storage and updates the CacheData accordingly.
         *
         * @param cacheData The CacheData object to be processed.
         */
        public void checkLocalConfig(CacheData cacheData) {
            final String dataId = cacheData.dataId;
            final String group = cacheData.group;
            final String tenant = cacheData.tenant;
            final String envName = cacheData.envName;
    
            // Check if a failover file exists for the specified dataId, group, and tenant.
            File file = LocalConfigInfoProcessor.getFailoverFile(envName, dataId, group, tenant);
    
            // If not using local config info and a failover file exists, load and use it.
            if (!cacheData.isUseLocalConfigInfo() && file.exists()) {
                String content = LocalConfigInfoProcessor.getFailover(envName, dataId, group, tenant);
                final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
                cacheData.setUseLocalConfigInfo(true);
                cacheData.setLocalConfigInfoVersion(file.lastModified());
                cacheData.setContent(content);
                LOGGER.warn(
                        "[{}] [failover-change] failover file created. dataId={}, group={}, tenant={}, md5={}, content={}",
                        envName, dataId, group, tenant, md5, ContentUtils.truncateContent(content));
                return;
            }
    
            // If use local config info, but the failover file is deleted, switch back to server config.
            if (cacheData.isUseLocalConfigInfo() && !file.exists()) {
                cacheData.setUseLocalConfigInfo(false);
                LOGGER.warn("[{}] [failover-change] failover file deleted. dataId={}, group={}, tenant={}", envName,
                        dataId, group, tenant);
                return;
            }
    
            // When the failover file content changes, indicating a change in local configuration.
            if (cacheData.isUseLocalConfigInfo() && file.exists()
                    && cacheData.getLocalConfigInfoVersion() != file.lastModified()) {
                String content = LocalConfigInfoProcessor.getFailover(envName, dataId, group, tenant);
                final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
                cacheData.setUseLocalConfigInfo(true);
                cacheData.setLocalConfigInfoVersion(file.lastModified());
                cacheData.setContent(content);
                LOGGER.warn(
                        "[{}] [failover-change] failover file changed. dataId={}, group={}, tenant={}, md5={}, content={}",
                        envName, dataId, group, tenant, md5, ContentUtils.truncateContent(content));
            }
        }
```

配置变更的`SDK`流程到这里也从差不多了，具体的一些分支的代码因为篇幅原因没有贴源码具体分析，但是只要理解了整个流程相关分支还是非常简单的。

