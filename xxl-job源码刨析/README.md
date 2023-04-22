# 一、架构设计项目工程分析

**注：**本源码解析基于：版本 v2.3.1

### 一、架构分析： <a href="#mxtim" id="mxtim"></a>

#### 1.1.1架构图： <a href="#pmklh" id="pmklh"></a>

<div>

<img src="https://cdn.nlark.com/yuque/0/2023/png/12408588/1673180022937-ef43e62d-83e6-4a98-8ea0-9d8b2ab61419.png" alt="">

 

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

</div>

### 同类产品对比 <a href="#bkf5u" id="bkf5u"></a>

|         | QuartZ                  | xxl-job                          | SchedulerX 2.0                        | **PowerJob**                                     |
| ------- | ----------------------- | -------------------------------- | ------------------------------------- | ------------------------------------------------ |
| 定时类型    | CRON                    | CRON                             | CRON、固定频率、固定延迟、OpenAPI                | **CRON、固定频率、固定延迟、OpenAPI**                       |
| 任务类型    | 内置Java                  | 内置Java、GLUE Java、Shell、Python等脚本 | 内置Java、外置Java（FatJar）、Shell、Python等脚本 | **内置Java、外置Java（容器）、Shell、Python等脚本**            |
| 分布式任务   | 无                       | 静态分片                             | MapReduce 动态分片                        | **MapReduce 动态分片**                               |
| 在线任务治理  | 不支持                     | 支持                               | 支持                                    | **支持**                                           |
| 日志白屏化   | 不支持                     | 支持                               | 不支持                                   | **支持**                                           |
| 调度方式及性能 | 基于数据库锁，有性能瓶颈            | 基于数据库锁，有性能瓶颈                     | 不详                                    | **无锁化设计，性能强劲无上限**                                |
| 报警监控    | 无                       | 邮件                               | 短信                                    | **邮件，提供接口允许开发者扩展**                               |
| 系统依赖    | 关系型数据库（MySQL、Oracle...） | MySQL                            | 人民币                                   | **任意 Spring Data Jpa支持的关系型数据库（MySQL、Oracle...）** |
| DAG 工作流 | 不支持                     | 不支持                              | 支持                                    | **支持**                                           |

**1.1.1 设计思想**

将调度行为抽象形成“调度中心”公共平台，而平台自身并不承担业务逻辑，“调度中心”负责发起调度请求。

将任务抽象成分散的JobHandler，交由“执行器”统一管理，“执行器”负责接收调度请求并执行对应的JobHandler中业务逻辑。

因此，“调度”和“任务”两部分可以相互解耦，提高系统整体稳定性和扩展性；

### 二、项目目录： <a href="#dbpew" id="dbpew"></a>

<div>

<img src="https://cdn.nlark.com/yuque/0/2023/png/12408588/1673181162748-19abf691-fc67-4bb4-8b70-df38152bf7d5.png" alt="">

 

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

</div>

#### 2.1.1xxl-job-admin工程结构： <a href="#vd2lu" id="vd2lu"></a>

<div>

<img src="https://cdn.nlark.com/yuque/0/2023/png/12408588/1673184959530-265ffcaa-5050-4671-b270-80e09950e3f0.png" alt="">

 

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

</div>

| 目录                               | 说明                   |
| -------------------------------- | -------------------- |
| com.xxl.job.admin.core.alarm     | 报警相关处理，默认实现了一个报警邮件发送 |
| com.xxl.job.admin.core.config    | 配置项                  |
| com.xxl.job.admin.core.cron      | 提供cron表达式的解析器        |
| com.xxl.job.admin.core.exception | 自定义异常类               |
| com.xxl.job.admin.core.model     | 实体类bean(与DB表映射)      |
| com.xxl.job.admin.core.old       | 移除历史实现               |
| com.xxl.job.admin.core.thread    | 调度器的各个守护线程           |
| com.xxl.job.admin.core.trigger   | 调度器下发任务到执行器trigger   |
| com.xxl.job.admin.dao            | 数据库操作层               |
| com.xxl.job.admin.service        | 业务开发层                |
| com.xxl.job.admin.controller     | 控制器 提供result api     |
