### Python执行器设计文档

[TOC]

### 背景需求

目前有些Python任务不够系统，处在整体框架之外；

例如，某类资产的下载、处理、上传，可能要手工分步操作；

引入Python执行器，可以将上述流程系统化、自动化，减少手工操作的成分，提升任务效率。

执行器具备高可用、高灵活、高并发、模块化、插件机制特性，可以快速往执行器中添加新任务模块，使模块之间高内聚低耦合，便于后续开发维护

#### 版本目标

完成Python执行器框架第一版本设计及实现。

- 执行器稳态运行并支持高并发、高可用
- 对外提供 HTTP API
- 支持自定义任务模块（插件）
- 支持分布式部署

#### 设计原则

- 高可用：避免单点故障、实现任务提交及运行的稳定
- 高性能：高并发异步处理任务性能
- 高稳定：系统长时间运行无故障，即便有，通过完善的故障修复机制，能极短时间恢复

#### 预期收益

- 统一管理、调度Python任务，只暴露接口给使用者，方便操作
- 模块化任务单元，让执行器易于开发、维护
- 利用Python数据计算优势，提高数据处理效率



### 三大框架对比

Python 生态圈有三个现象级的 Web 框架 `Flask，Django，Tornado`，在Python中他们公认度很高，且社区活跃；

这里主要围绕这三大框架做简要介绍和对比，从中选择契合需求的一个，做为Python执行器的底层支撑.

#### 三大框架特性

##### Django

* 比较全能的 web 框架，功能丰富完备，可维护性和开发速度很棒
* 并发比较低，高并发需要二次开发，自己写socket实现HTTP通信
* 内置ORM、Admin等自带组件，集成度很高，主要面向RDB，对于轻简场景显得比较heavy

* Django 其实主要 `慢在 Django ORM 与数据库的交互上`，是否选用 Django，可以看项目对数据库交互的要求以及优化代价

##### Flask

* 号称 Python 代码写得最好的项目之一，高灵活高扩展

* Flask 虽然是微框架，但其扩展工具很多，做成规模化项目也不在话下

* Flask `可以自由选择数据库交互组件`，支持原生SQL语句，可快速实现ORM（通过SQLAlchemy）、Admin等

* Flask+celery +redis 实现异步特性后，性能相对 Tornado 基本持恒，也许Flask 的灵活性可能是某些团队更需要的
* [flask 插件列表](https://pypi.org/search/?c=Framework+%3A%3A+Flask&page=1)

##### Tornado

* 天生异步，异步IO性能强悍，面向网络IO、高网络流量时性能突出
* 提供路由映射、模板渲染，复杂高效

* 相比 Django、Flask，Tornado  是较为原始的框架，诸多内容需要自己去实现、处理，需要造的轮子比较多。



#### 性能对比

从`JSON序列化+返回`、`HTTP响应`、`ORM读取数据库，渲染到模板`，三方面进行性能对比；

对比来源：http://klen.github.io/py-frameworks-bench/#methodic

##### JSON序列化

![序列化JSON.png](https://i.loli.net/2021/01/05/WAPalFjrzJU9RHZ.png)

| [Name](javascript:void(0);) | [50% (ms)](javascript:void(0);) | [75% (ms)](javascript:void(0);) | [Avg (ms)](javascript:void(0);) | [Req/s](javascript:void(0);) |
| --------------------------- | ------------------------------- | ------------------------------- | ------------------------------- | ---------------------------- |
| Django                      | 41.41                           | 42.14                           | 42.52                           | 4762                         |
| Flask                       | 43.03                           | 43.79                           | 43.33                           | 4630                         |
| Tornado                     | 77.17                           | 78.09                           | 77.51                           | 2578                         |

* Django 完成一次 json 序列化平均时间 `42.52` 毫秒，每秒请求量 `4762` 次
* Flask 完成一次 json 序列化平均时间 `43.33` 毫秒，每秒请求量 `4630` 次
* Tornado 完成一次 json 序列化平均时间 `77.51` 毫秒，每秒请求量 `2578` 次
* 结论：`序列化方面 Tornado 不如 Django 、Flask 快`



##### HTTP响应

![HTTP响应.png](https://i.loli.net/2021/01/05/h97NqQvdksBKf6W.png)

| [Name](javascript:void(0);) | [50% (ms)](javascript:void(0);) | [75% (ms)](javascript:void(0);) | [Avg (ms)](javascript:void(0);) | [Req/s](javascript:void(0);) |
| --------------------------- | ------------------------------- | ------------------------------- | ------------------------------- | ---------------------------- |
| Tornado                     | 1044.8                          | 1068.02                         | 1036.97                         | 187.8                        |
| Flask                       | 2679.56                         | 3990.17                         | 3344.27                         | 18.15                        |
| Django                      | 2908.53                         | 4165.52                         | 3477.36                         | 18.1                         |

* Tornado 完成 http 请求平均时间是 `1.04` 秒，每秒请求 `187.8` 次
* Flask 完成 http 请求平均时间是 `3.34` 秒，每秒请求 `18.15` 次
* Django 完成 http 请求平均时间是 `3.47` 秒，每秒请求 `18.1` 次
* 结论：`HTTP请求中Tornado比Flask、Django快3倍左右`



##### ORM性能

![ORM性能.png](https://i.loli.net/2021/01/05/scDEiqFZaUKJMO3.png)

| [Name](javascript:void(0);) | [50% (ms)](javascript:void(0);) | [75% (ms)](javascript:void(0);) | [Avg (ms)](javascript:void(0);) | [Req/s](javascript:void(0);) |
| --------------------------- | ------------------------------- | ------------------------------- | ------------------------------- | ---------------------------- |
| Tornado                     | 1395.43                         | 1409.24                         | 1344.69                         | 143                          |
| Flask                       | 1281.38                         | 1326.05                         | 1440.24                         | 123                          |
| Django                      | 2367.04                         | 2742.09                         | 2904.04                         | 42.9                         |

* Django DB、渲染平均耗时 `2904.04` ms，每秒处理请求量 `42.9` 次
* Flask DB、渲染平均耗时 `1440.24` ms，每秒处理请求量 `123` 次
* Tornado DB、渲染平均耗时 `1344.69` ms，每秒处理请求量 `143` 次
* 结论：`Flask和Tornado二者不相上下，且都比Django快`



#### 对比归纳

三种性能综合对比，三种对比数据相加（JSON序列化、HTTP响应、ORM性能）：

* **Tornado：2457 ms** （77+1036+1344）

* **Flask：4827 ms** （43+3344+1440）

* **Django：6423 ms** （42+3477+2904）

`Django`和`Flask`本身是同步框架，没有`Tornado`天生异步的优势，但对于前者同步特性导致吞吐量小的问题，可以通过Celery解决。

而Python执行器处理的任务，对网络IO并没太大要求（内部使用），其重心更多在于任务调度、服务稳定、高展性；

综上，Django比较重，对于执行器来说，Django很多功能模块用不到，Tornado优势体现在异步网络IO，更多功能需要专门造轮子；

Flask 灵活性、扩展性、服务稳定性都基本能满足执行器的需求。



### 现有选型介绍

因为 Flask 可扩展性、灵活性都很强，web框架该有的它基本都有，且该框架短小精悍，比较适合执行器需求；

目前选择的方案是 `Flask-Celery-Redis` ，下面对Celery以及 `Flask-Celery-Redis` 做简要介绍。

#### Celery

Celery 是一个强大的分布式任务队列，是一个独立运行的服务，内置socket，它可以让任务的执行完全脱离主程序，或者把Worker分布到其他主机上。

其本身不提供消息服务，可借助第三方MQ（RabbitMQ、Redis、MongoDB、RocketMQ）

* 高可用：当任务执行失败或执行过程中发生连接中断，celery 会自动尝试重新执行任务 

* 快速：一个单进程的celery每分钟可处理上百万个任务 

* 灵活： 几乎celery的各个组件都可以被扩展及自定制（broker、Backend、异步任务、定时任务）

  

#### Flask-Redis-Celery 

* Flask
  - [x] 作为执行器 Server 的载体
  - [x] 提供 API 接口
  - [x] 和Celery结合实现异步任务调度
* Redis（可换为其他MQ）
  - [x] 作为Celery任务队列（Queue 不提供订阅）
  - [x] 作为celery任务结果存储
  - [ ] 可分布式部署
* Celery
  - [x] Broker（消息中间件）
  - [x] Worker（任务执行单元）
  - [x] Backend（结果存储）
  - [x] 可多个worker并发执行任务
  - [ ] 可分布式部署
* Nginx负载均衡（`TODO`）
  - [ ] 对Flask Server做负载均衡

#### Flower

监控、管理Celery任务调度中的状态

参考：https://flower-docs-cn.readthedocs.io/zh/latest/

<img src="https://i.loli.net/2021/01/06/xdC6U1pIKqLQNOD.png" alt="flower.png"  />



### 流程图

#### Flask-Redis-Celery流程图

![falsk-celery-redis.png](https://i.loli.net/2021/01/05/5ui9VOcNedysWYP.png)

#### 类比

![flask-celery-redis类比.png](https://i.loli.net/2021/01/05/AUhwyZeirSCG3Ef.png)



### 程序目录结构

```Python
muji-data-job-pyexecutor
    │  .gitignore
    │  app.py  # Flask app，如果想项目后续扩展变大，可用蓝图管理
    │  README.md
    │  requirements.txt
    │
    ├─config
    │      config.yaml
    │
    ├─my_celery  # Celery 任务 
    │  │  main.py
    │  │  settings.py
    │  │
    │  ├─clean
    │  │      tasks.py
    │  │
    │  ├─exchange_spider
    │  │      tasks.py
    │  │
    │  └─fix
    │         tasks.py
    │
    ├─testcase
    │      compare_df.py
    │
    └─util
          spiderUtils.py
          utils.py
# app.py 用来启动 Flask Server 以及路由规则
# my_celery/main.py 启动celery服务
# my_celery/clean、my_celery/fix、my_celery/exchange_spider 分别为具体task实现
# testcase 测试用例
# util 工具类
```

### 环境说明

| 环境                         | 操作系统      | 环境依赖                                                     | 备注                                   |
| ---------------------------- | ------------- | ------------------------------------------------------------ | -------------------------------------- |
| 开发环境<br />服务端运行环境 | Windows/Linux | anyjson==0.3.3<br/>celery==3.1.25<br/>cPython==0.0.6<br/>dubbo-client==1.1.4<br/>Flask==1.0.2<br/>gevent==20.9.0<br/>grpcio==1.23.0<br/>grpcio-tools==1.23.0<br/>Jinja2==2.10.1<br/>kombu==3.0.37<br/>loguru==0.5.3<br/>numexpr==2.7.1<br/>numpy==1.19.4<br/>pandas==0.23.4<br/>pickleshare==0.7.5<br/>pyyaml==5.1.1<br/>redis==2.10.5<br/>requests==2.25.0<br/>six==1.15.0<br/>tables==3.6.1<br/>tqdm==4.54.1<br/>Werkzeug==0.14.1<br/>xmltodict==0.12.0<br/>BeautifulSoup4==4.6.3 | 开发环境为Windows<br />部署环境为Linux |