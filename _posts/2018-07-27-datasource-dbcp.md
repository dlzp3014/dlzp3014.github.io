---
layout: post
title:  "dbcp2数据源配置"
date:   2018-07-26 22:34:00
categories: DataSource 
tags: DataSource 
---

* content
{:toc}

DBCP（DataBase Connection Pool）是由apace提供的数据库连接池组件，在使用过程中需要引入comms-logging和commons-pool2。使用Mysql数据库时要导入mysql-connector 5.0以上的版本   
[官方文档](http://commons.apache.org/proper/commons-dbcp/)




## DBCP2配置属性

### 链接配置

| Parameter|Description |
| :--------|:--------|
|username	|建立连接的用户名|
|password	|建立连接的密码|
|url	|建立连接的URL|
|driverClassName|JDBC驱动，有效的java 类名|
|connectionProperties|建立新连接时被发送给JDBC驱动的连接参数，格式必须是 [propertyName=property;]。注意：参数user/password将被明确传递，所以不需要包括在这里。|

### 事务配置

| Parameter|Default|Description |
| :--------|:--------|:--------|
|defaultAutoCommit|driver default|默认auto-commit|
|defaultReadOnly|driver default	|默认read-only|
|defaultTransactionIsolation|driver default	|默认的TransactionIsolation状态:NONE,READ_COMMITTED,   READ_UNCOMMITTED,REPEATABLE_READ,SERIALIZABLE|
|defaultCatalog||连接池创建的连接的默认的catalog|
|cacheState|true|如果为true，则池化连接将在首次读取或写入时以及在所有后续写入时缓存当前的readOnly和autoCommit设置。这消除了对进一步调用getter的额外数据库查询的需要。如果直接访问基础连接并且readOnly和/或autoCommit设置已更改，则缓存的值将不会反映当前状态。在这种情况下，应通过将此属性设置为false来禁用缓存。
|defaultQueryTimeout|null|	如果为非null，则此Integer属性的值确定将用于从池管理的连接创建的语句的查询超时。null表示将使用驱动程序默认值.|
|enableAutoCommitOnReturn|true|如果为true，则在返回连接时，如果自动提交设置为 false，则将使用Connection.setAutoCommit（true）检查并配置返回到池 的连接.|
|rollbackOnReturn|true|True表示如果未启用自动提交且连接不是只读，则在返回池时将回滚连接。|

### 连接池配置

| Parameter|Default|Description|
| :--------|:--------|:--------|
|INITIALSIZE|0|连接池启动时创建的初始化连接数量|
|maxTotal|8|同一时间能够分配的最大活动连接的数量|
|maxidle|8|连接池中容许保持空闲状态的最大连接数量,超过的空闲连接将被释放,如果设置为负数表示不限制|
|minIdle|0|连接池中容许保持空闲状态的最小连接数量,低于这个数量将创建新的连接,如果设置为0则不创建|
|maxWaitMillis|indefinitely|当没有可用连接时,连接池等待连接被归还的最大时间(以毫秒计数),超过时间则抛出异常,如果设置为-1表示无限等待|

 【NOTE】:如果在负载很重的系统上将maxIdle设置得太低，您可能会看到连接被关闭，几乎立即打开新连接。这是因为活动线程暂时关闭连接的速度比打开它们的速度快，导致空闲连接的数量超过maxIdle。对于负载较重的系统，maxIdle的最佳值会有所不同，但默认值是一个很好的起点。

### 连接状况检查
 
| Parameter|Default|Description|
| :--------|:--------|:--------|
|validationQuery||在将它们返回给调用者之前将用于验证来自此池的连接的SQL查询。如果指定，则此查询 必须是返回至少一行的SQL SELECT语句。如果未指定，将通过调用isValid（）方法验证连接。|
|validationQueryTimeout||连接验证查询失败前的超时（以秒为单位）。如果设置为正值，则通过 用于执行验证查询的Statement的setQueryTimeout方法将此值传递给驱动程序 。|
|testOnCreat|false|指示在创建后是否验证对象。如果对象无法验证，则触发对象创建的借用尝试将失败。|
|testOnBorrow|true|	指明是否在从池中取出连接前进行检验,如果检验失败,则从池中去除连接并尝试取出另一个.注意: 设置为true后如果要生效,validationQuery参数必须设置为非空字符串|
|testOnReturn|false|在返回池之前是否验证对象的指示|
|testWhileIdle|false|指示对象是否将由空闲对象逐出器验证（如果有）。如果对象无法验证，它将从池中删除。|
|timeBetweenEvictionRunsMillis|-1|在空闲对象逐出器线程的运行之间休眠的毫秒数。当非正数时，将不运行空闲对象逐出器线程。|
|numTestsPerEvictionRun	|3|在每次运行空闲对象逐出器线程（如果有）期间要检查的对象数。|
|minEvictableIdleTimeMillis|1000 * 60 * 30|在空闲对象逐出器（如果有的话）有资格驱逐之前，对象在池中闲置的最短时间。|
|softMinEvictableIdleTimeMillis	|-1|	在空闲连接逐出器符合驱逐条件的情况下，连接可能在池中空闲的最短时间，其中额外条件是至少“minIdle”连接保留在池中。当minEvictableIdleTimeMillis设置为正值时，minEvictableIdleTimeMillis首先由空闲连接逐出器检查 - 即，当逐出器访问空闲连接时，首先将空闲时间与minEvictableIdleTimeMillis进行比较（不考虑池中空闲连接的数量）然后针对softMinEvictableIdleTimeMillis，包括minIdle约束。|
|maxConnLifetimeMillis	|-1|	连接的最大生命周期（以毫秒为单位）。超过此时间后，连接将无法进行下一次激活，钝化或验证测试。值为零或更小意味着连接具有无限生命周期。|
|logExpiredConnections	|true|	用于记录消息的标志，指示由于超出maxConnLifetimeMillis而池正在关闭连接。将此属性设置为false可以禁止默认情况下已启用的过期连接日志记录。|
|connectionInitSqls|null|一组SQL语句，用于在首次创建物理连接时初始化它们。这些语句只执行一次-当配置的连接工厂创建连接时。
|lifo|true|True表示borrowObject返回池中最近使用的（“lastin”）连接（如果有空闲连接可用）。False表示池表现为FIFO队列 - 连接是从空闲实例池中按照它们返回池中的顺序获取的。|

### 缓存

| Parameter|Default|Description|
| :--------|:--------|:--------|
|poolPreparedStatements|false|开启池的prepared statement 池功能|
|maxOpenPreparedStatements|unlimited|statement池能够同时分配的打开的statements的最大数量, 如果设置为0表示不限制|

该组件还具有池PreparedStatements的能力。启用后，将为每个Connection创建一个语句池，并且将汇集由以下方法之一创建的PreparedStatements：

- public PreparedStatement prepareStatement（String sql）
- public PreparedStatement prepareStatement（String sql，int resultSetType，int resultSetConcurrency）
 注 - 确保您的连接有一些资源留给其他语句。池化PreparedStatements可能会使其游标在数据库中保持打开状态，从而导致游标用完，特别是如果maxOpenPreparedStatements保留为默认值（无限制）且应用程序为每个连接打开大量不同的PreparedStatements。要避免此问题，应将maxOpenPreparedStatements设置为小于可在Connection上打开的最大游标数的值。
 

### 连接泄露回收

| Parameter|Default|Description|
| :--------|:--------|:--------|
|removeAbandoned|false|标记是否删除泄露的连接,如果他们超过了removeAbandonedTimout的限制.如果设置为true,连接被认为是被泄露并且可以被删除,如果空闲时间超过removeAbandonedTimeout.设置为true可以为写法糟糕的没有关闭连接的程序修复数据库连接.|
|removeAbandonedTimeout|300|泄露的连接可以被删除的超时值, 单位秒|
|logAbandoned|false|标记当Statement或连接被泄露时是否打印程序的stack traces日志。被泄露的Statements和连接的日志添加在每个连接打开或者生成新的Statement,因为需要生成stack trace。|

### Note

Java数据库连接有“8小时问题”，所以destroy-method="close"一定要加上。“8小时问题”是指一个连接空闲8小时数据库会自动关闭，而数据源并不知道。高并发下，可以testOnBorrow设置false，testWhileIdle设置为true，这样就会定时对后台空链接进行检测发现无用连接就会清除掉，不会每次都去都去检测是否8小时的空链接。




























