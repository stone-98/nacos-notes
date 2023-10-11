# 任务执行引擎设计与分析

Nacos有很多任务需要在后台执行，所以Nacos设计了一套任务执行引擎。

Nacos任务执行引擎主要组成如下：

## 任务

![image-20231010145939223](https://skyxg.oss-cn-beijing.aliyuncs.com/images/202310101459842.png)

- NacosTask：任务执行引擎基类。
- AbstractExecuteTask：抽象的任务，代表需要立马执行的任务。
- AbstractDelayTask：抽象的延迟任务，代表延迟执行的任务。

## 执行引擎

![image-20231010150427909](https://skyxg.oss-cn-beijing.aliyuncs.com/images/202310101504014.png)

- `NacosTaskExecuteEngine`：任务执行引擎基类
- `AbstractNacosTaskExecuteEngine`：抽象的任务执行引擎，使用`defaultTaskProcessor`存储处理器，并且对`processor`相关接口进行实现。

- `NacosExecuteTaskExecuteEngine`：Nacos任务引擎的实现类，用于处理执行需要立马执行的任务。
- `NacosDelayTaskExecuteEngine`：Nacos延迟任务执行引擎，用于执行延迟执行的任务，内部通过一个线程池进行实现。

