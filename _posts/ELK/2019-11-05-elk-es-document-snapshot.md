---
layout: post
title:  "Elasticsearch官方文档-Snapshot And Restore(快照和恢复)"
date:   2019-11-05 08:38:00
categories: ELK 
tags: Elasticsearch
---

* content
{:toc}

Snapshot快照是从正在运行的ES集群中得到的备份。可以获取各个索引或者集群中的全部索引快照，并把它们存储在一个共享的文件系统库中，并且有一些插件支持S3，HDFS，Azure，Google Cloud Storage等的远程存储库。`Snapshots快照是递增的`，意味着当创建索引快照时，Elasticsearch避免复制任何已经存储在仓库作为同一索引的早期快照的一部分数据，因此可以频繁地为群集创建快照

- 通过`restore API`可以将快照恢复到正在运行的群集中，当恢复一个索引时，可以更改已恢复索引的名称及其某些设置。快照和恢复功能的使用方式具有很大的灵活性

- 可以通过使用SLM(`snapshot lifecycle management`)自动备份索引和恢复


【Note】: 不能通过获取所有节点的数据目录副本来备份Elasticsearch集群，Elasticsearch可能会在运行时更改其数据目录的内容；不能指望复制其数据目录来捕获其内容的一致镜像。如果尝试从此类备份还原群集，则该群集可能会失败并报告损坏或丢失文件，或者它似乎成功了但默默地丢失了一些数据。`备份群集的唯一可靠方法是使用快照和恢复功能`





## 版本兼容

版本兼容性是指基本的Lucene索引兼容性。 在版本之间进行迁移时，请遵循`升级文档`。快照包含组成索引在磁盘上数据结构副本，这意味着快照只能还原到可以读取索引的Elasticsearch版本

每个快照包含了各种版本Elasticsearch中创建的索引，并且在恢复快照时，必须有可能将所有索引恢复到目标集群中。如果快照中的任何索引是在不兼容的版本中创建的，则将无法还原快照

【Note】: 在升级之前备份数据时，如果快照中包含与升级版本不兼容的索引，则升级后将无法还原快照


如果最终遇到需要还原的快照与当前运行的群集版本不兼容的情况时，则可以将其还原到最新的兼容版本上，并使用`reindex-from-remote`在当前版本中重建索引。只有在原始索引源可使用时，才可以从远程重新索引。`与简单地还原快照相比，检索和重新索引数据所花费的时间可能要长得多`。如果有大量数据，建议使用数据的一部分测试远程过程中的重新索引，以了解时间要求



## 仓库

在执行快照和恢复操作之前必须注册一个快照仓库，建议为每个主要版本创建一个新的快照仓库，有效的仓库设置取决于仓库类型。`如果使用多个集群注册相同的快照存储库，则只有一个群集应具有对该仓库的写访问权限，连接到该仓库的所有其他集群应将存储库设置为readonly` 模式。

快照格式可以在主要版本之间更改，因此，如果在不同的版本上有集群，尝试编写相同的存储库，则一个版本编写的快照可能对其他版本不可见，并且仓库可能已损坏。虽然将仓库设置为只读，但是除了一个集群之外，其他集群都应该能够处理多个主要版本不同的集群，但是不支持这种配置


### 仓库操作API

- 注册仓库

```json
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "my_backup_location"
  }
}

```

- 获取注册仓库信息

1): 获取单个注册仓库信息

```json
GET /_snapshot/my_backup

{
  "my_backup": {
    "type": "fs",
    "settings": {
      "location": "my_backup_location"
    }
  }
}
```

2): 获取多个注册仓库信息，以逗号分隔。当执行仓库名时，可以使用`*`通配符

```json
GET /_snapshot/repo*,*backup*
```

3): 获取所有已注册仓库信息

```json
GET /_snapshot
OR
GET /_snapshot/_all

```

- 注销仓库

```json
DELETE /_snapshot/my_backup

```

`当仓库注销时，Elasticsearch只删除对仓库存储快照位置的引用，快照本身则保持原样`( left untouched and in place)




### 共享文件系统仓库

使用共享文件系统存储快照。`为了注册共享文件系统仓库，必须将相同的共享文件系统挂载到所有主节点和数据节点上的相同位置`。此位置（或其父目录之一）必须在所有主节点和数据节点的elasticsearch.yml文件中设置`path.repo`

```json
path.repo: ["/mount/backups", "/mount/longterm_backups"]
```

然后重启所有节点，执行注册命令：使用自定义的仓库名注册共享文件系统仓库


```json
PUT /_snapshot/my_fs_backup(仓库名)
{
    "type": "fs",
    "settings": {
        "location": "/mount/backups/my_fs_backup_location",(共享文件系统仓库目录)
        "compress": true
    }
}
```

如果将仓库位置指定为相对路径，则将根据`path.repo`中指定的第一个路径解析此路径

支持的设置如下：

- location: 快照位置[必选]

- compress: 默认true，打开快照文件压缩。压缩只应用于文件元数据(index mapping and settings)，数据文件不压缩

- chunk_size: 大文件可以在快照过程中分解成块。如下:1GB, 10MB, 5KB, 500B. Defaults to null (unlimited chunk size)

- max_restore_bytes_per_sec: 每个节点恢复速率的节流，默认40mb每秒

- max_snapshot_bytes_per_sec: 每个节点快照速率的节流，默认为每秒40mb。

- readonly: 让库只读。默认值为假

### 只读仓库URL

URL仓库("type": "url")可以用作访问共享文件系统仓库创建数据的另一种只读方式，URL参数中指定的URL应该指向共享文件系统仓库的根路径，支持如下配置：

```java
url 快照位置，必选
```

URL仓库支持如下协议：http、https、ftp、file、jar。URLs必须为在`repositories.url.allowed_urls`设置的白名单中，且支持通配符

```java
repositories.url.allowed_urls: ["http://www.example.org/root/*", "https://*.mydomain.com/*?*#*"]
```

使用`file:`的URL:URL存储只能指向在`path.repo`的位置，类似于共享文件系统存仓库


### 源仓库


源仓库创建的快照占用的磁盘空间将减少50%，源仓库只包含存储的字段和索引的源数据。不包括索引或文档的值结构，并且在恢复后不可搜索。恢复源快照后，必须使用`reindex`数据到新的索引。源仓库委托给另一快照仓库存储

源快照支持条件：`_source`属性启用且`source-filtering`没有应用。还原仅源快照时：

- 恢复的索引是只读的，并且只能提供match_all搜索或滚动请求以重建索引

- 除match_all和_get请求以外的查询均不受支持。

- 恢复的索引映射为空，但是可以从类型顶级meta元素中获取原始映射。


当创建源快照时，必须指定`type`和委托的存储快照的仓库名：

```java
PUT _snapshot/my_src_only_repository
{
  "type": "source",
  "settings": {
    "delegate_type": "fs",
    "location": "my_backup_location"
  }
}
```

### 仓库插件

```java
repository-s3 for S3 repository support
repository-hdfs for HDFS repository support in Hadoop environments
repository-azure for Azure storage repositories
repository-gcs for Google Cloud Storage repositories
```

### 仓库校验

注册仓库后，应立即在所有主节点和数据节点上对仓库进行验证，以确保该仓库在集群中的所有节点上(当前存在)都可以正常运行。`verify`参数可用于在注册或更新存仓库时显式禁用仓库验证

```json
PUT /_snapshot/my_unverified_backup?verify=false
{
  "type": "fs",
  "settings": {
    "location": "my_unverified_backup_location"
  }
}
```

验证过程也可以通过运行以下命令手动执行：

```json
POST /_snapshot/my_unverified_backup/_verify
```

返回已成功验证存储库的节点列表，或者返回验证过程失败的错误消息


### 仓库清理

仓库随时间积累未被任何现有快照引用的数据，数据安全的结果以确保快照功能在创建快照期间失败场景的提供，并且快照创建过程的分散性质。这些未引用的数据绝不会对快照存仓库的性能或安全性产生负面影响，但会导致超出必要的仓库使用量。为了清理此未引用的数据，可以调用仓库的清理端点，这将触发对存仓库内容的完整记帐，并随后删除找到的所有未引用数据。

```json
POST /_snapshot/my_repository/_cleanup
{
  "results": {
    "deleted_bytes": 20,
    "deleted_blobs": 5
  }
}
```


根据具体的仓库实现，显示的可用字节数以及删除的blob数将是近似值或精确结果。移除任何非零值的二进制数意味着找到了未引用的二进制，并随后对其进行了清理。

【Note】: 从仓库删除任何快照时，将自动执行此端点的大多数清理操作。如果定期删除快照，则在大多数情况下，使用此功能将无法节省任何空间或仅节省很少的空间，因此应相应地降低其调用频率。


## 快照

### 创建快照

一个仓库中可以包含同一集群的多个快照，`在集群中快照由唯一的名字进行标识`。

```json
PUT /_snapshot/my_backup/snapshot_1?wait_for_completion=true
```
`wait_for_completion`参数指定请求是否在初始化快照（默认）后立即返回还是等待快照完成后返回。在快照初始化期间，有关先前所有快照的信息会加载到内存中，这意味着在大仓库中，即使将`wait_for_completion`参数设置为false，此命令也可能需要几秒钟（甚至几分钟）才能返回。

默认情况下，将创建集群中所有打开和启动的索引的快照，可以通过在快照请求主体中指定索引列表来更改此行为：

```json
PUT /_snapshot/my_backup/snapshot_2?wait_for_completion=true
{
  "indices": "index_1,index_2", (指定索引)
  "ignore_unavailable": true, （忽略不存在的所有）
  "include_global_state": false, （是否存储集群状态到快照中)
  "metadata": {
    "taken_by": "kimchy",
    "taken_because": "backup before upgrading"
  }
}
```

- indices参数: 可以使用indices(支持`multi index syntax`)参数来指定应包含在快照中的索引列表

- ignore_unavailable: 将其设置为true将导致在创建快照时忽略不存在的索引；默认情况下，如果`ignore_unavailable`未设置选项并且索引丢失，则快照请求将失败

- include_global_state: false时可以防止将群集全局状态存储为快照的一部分。默认情况下，如果参与快照的一个或多个索引没有所有可用的主分片时，则整个快照将失败，可以通过设置`partial`为来更改此行为true

- metadata: 用于将任意元数据附加到快照，这可能是有关谁拍摄创建的快照，为什么创建快照或任何其他有用数据的记录


快照名称可以使用`date math expressions`自动导出，类似于创建新索引时。`【Note】: 特殊字符需要进行URI编码`

```json
# PUT /_snapshot/my_backup/<snapshot-{now/d}>
PUT /_snapshot/my_backup/%3Csnapshot-%7Bnow%2Fd%7D%3E
```

索引快照过程是增量的，在创建索引快照的过程Elasticsearch分析已经存储在仓库的索引文件列表，且只复制自上次快照以来创建或者更改的文件，这样可以将多个快照以紧凑的形式(in a compact form)保存在仓库中。快照过程以非阻塞方式执行，对正在创建快照的索引可以继续执行索引和搜索操作。但是，快照表示创建快照时索引的时间点视图(point-in-time view of the index at the moment)，因此快照进程启动后添加到索引中的记录将不会出现在快照中，对于已经启动且目前没有重新定位的主分片，快照进程将立即启动。从Elasticsearch1.2版本后，Elasticsearch在快照之前，将等待重新或初始化分片完成


除了创建每个索引的副本之外，快照过程还可以存储全局群集元数据，其中包括持久化群集设置和模板。`瞬态设置和注册的快照存储库不存储为快照的一部分`。`群集中任何时间都只能执行一个快照过程`。在创建特定分片的快照时，该分片无法移动到另一个节点，这可能会干扰重新平衡过程和分配筛选(rebalancing process and allocation filtering)。一旦快照完成后，Elasticsearch只能做到将分片移动到另一个节点（根据当前的分配过滤设置和重新平衡算法：according to the current allocation filtering settings and rebalancing algorithm）。

### 获取快照信息

1): 获取单个快照信息

```json
GET /_snapshot/my_backup/snapshot_1
```

当前命令返回有关快照的基础信息：开始和结束时间、Elasticsearch创建快照的版本、包含的索引列表、快照的当前状态以及快照过程中发生的failures列表。快照state可以为：

- IN_PROGRESS: 快照当前正在运行

- SUCCESS: 快照已经完成且所有的分片已存储成功

- FAILED: 快照以错误结束，并未能存储任何数据

- PARTIAL: 全部集群状态已经存储，但是至少一个分片数据没有存储成功。在这种情况下，失败部分应包含有关未正确处理的分片的更多详细信息。

- INCOMPATIBLE: 快照的创建使用旧的Elasticsearch的版本，因此与当前版本的集群不兼容

2): 获取多个快照信息，支持通配符

```json
GET /_snapshot/my_backup/snapshot_*,some_other_snapshot
```

3): 获取当前存储在仓库中的所有快照

```json
GET /_snapshot/my_backup/_all

```

如果某些快照不可用，则该命令将失败。布尔参数`ignore_unavailable`可用于返回当前可用的所有快照

从成本和性能的角度(perspective)来看，在基于云(cloud-based repositories)的仓库中获取所有快照的成本可能很高。如果只获取仓库中的快照名或者uuids和每个快照的索引信息需求时，`verbose`布尔选项参数可设置false，以在仓库中执行更高效、更经济的快照检索。`verbose`设置为false时将忽略(omit)关于快照的所有其他信息，例如status信息,快照分片数量，默认`verbose`参数为false

4): 检索当前运行的快照

```json
GET /_snapshot/my_backup/_current
```

### 删除快照

```json
DELETE /_snapshot/my_backup/snapshot_2

```

当从快照中删除快照时，Elasticsearch将删除所有相关的要删除快照和没有被其他任何快照使用的文件。如果删除的快照操作是在创建快照时执行的，那么快照过程将中止，作为快照过程一部分创建的所有文件将被清除。因此删除快照操作可用来取消误启动(started by mistake)长时间运行的快照操作

## 恢复

```json
POST /_snapshot/my_backup/snapshot_1/_restore
```
默认情况下，存储在快照中的所有索引将被恢复，但集群状态不恢复。通过在恢复请求体中使用`indices`和`include_global_state`选项，可以选择需要恢复的索引和全局集群状态，索引列表支持`multi index syntax`。`rename_pattern`和`rename_replacement`选项可用于使用正则表达式在恢复时重建索引，支持引用原始文本。`include_aliases`为false时，可防止别名与相关索引一起被还原

```json
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": true,
  "rename_pattern": "index_(.+)",
  "rename_replacement": "restored_index_$1"
}
```

可以正在运行中集群上执行恢复操作，但是只有在已关闭且分片数量与快照中的索引相同的情况下，才能恢复现有索引。如果要恢复的索引已关闭，恢复操作将自动打开它们;如果在集群中不存在，则创建新索引。如果使用`include_global_state`(默认false)参数恢复集群状态，则将添加当前集群中不存在的要恢复模板，并将具有相同名称的现有模板替换为要恢复模板，要恢复的持久性设置将添加到现有的持久性设置中

### 局部恢复

默认情况下，如果一个或多个参与该操作的索引没有所有可用分片快照时，则整个恢复操作将失败(如某些分片快照时失败)，通过设置`partial`为true仍可恢复一些索引`(这种情况下，只有成功快照的分片被恢复，所有丢失的分片将重新重建为空)`


### 恢复时修改索引设置

多数索引设置在恢复时可以被重写

```json
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1",
  "index_settings": {
    "index.number_of_replicas": 0 (恢复时不创建任何副本)
  },
  "ignore_index_settings": [
    "index.refresh_interval" (切换回默认的刷新间隔)
  ]
}
```

一些选项设置如`index.number_of_shards`索引分片不能再恢复操作时修改

### 从不同的集群中恢复

快照中存储的信息没有绑定到特定的集群或集群名称，因此可以从一个集群中恢复快照到另一个集群中，所需要做的就是在新集群中注册包含快照的仓库并启动恢复过程。新的集群不必具有相同的大小或拓扑，新集群的版本应该与用于创建快照的集群相同或更新(only 1 major version newer:只有一个主要版本更新)

如果新集群的规模较小，则需要考虑其他因素：

- 首先必须确保新集群有足够的空间存储快照中的所有索引

- 可以在恢复期间更改索引设置以减少副本的数量，有助于在小集群中存储快照

- 也可以使用`indices`参数只选择索引的子集

如果使用`shard allocation filtering分片分配过滤`将原始集群中的索引分配给特定节点，在新的集群中将执行(enforced)相同的规则。因此，如果新集群不包含具有可分配要恢复索引的适当属性节点，则除非在恢复操作期间更改了这些索引分配设置，否则无法成功还原该索引

恢复操作也检查要恢复的持久性设置是否与当前群集兼容，以避免意外恢复不兼容的设置。如果需要使用不兼容的持久性设置还原快照，请尝试在不具有全局群集状态的情况下还原快照。


## 快照状态

- 获得当前正在运行的快照及其详细状态信息列表(全部):

```json
GET /_snapshot/_status (all currently running snapshots.)
```

- 指定仓库运行中的快照

```json
GET /_snapshot/my_backup/_status
```

- 指定快照的详细，即使快照没有运行

```json
GET /_snapshot/my_backup/snapshot_1/_status
{
  "snapshots": [
    {
      "snapshot": "snapshot_1",
      "repository": "my_backup",
      "uuid": "XuBo4l4ISYiVg0nYUen9zg",
      "state": "SUCCESS",
      "include_global_state": true,
      "shards_stats": {
        "initializing": 0,
        "started": 0,
        "finalizing": 0,
        "done": 5,
        "failed": 0,
        "total": 5
      },
      "stats": {
        "incremental": {
          "file_count": 8,
          "size_in_bytes": 4704
        },
        "processed": {
          "file_count": 7,
          "size_in_bytes": 4254
        },
        "total": {
          "file_count": 8,
          "size_in_bytes": 4704
        },
        "start_time_in_millis": 1526280280355,
        "time_in_millis": 358
      }
    }
  ]
}
```

输出由不同的部分组成，`stats`子对象提供了快照文件的数量和大小的详细信息，由于快照是增量的，只复制没有在仓库中的Lucene segments(片段)，因此该`stats`对象包含快照引用的所有文件`total`部分，以及实际需要复制，作为增量快照一部分的`incremental`部分。如果快照仍在进行中，则还有processed部分包含有关正在复制的文件的信息

- 多ids支持

```josn
GET /_snapshot/my_backup/snapshot_1,snapshot_2/_status
```

## 监控快照/恢复过程

有几种方法可以监视快照的进度并在进程运行时还原它们，两种操作都支持`wait_for_completion `参数，阻塞客户端直到操作完成，这是用于获得操作完成通知的最简单方法

- 快照操作也可以通过对快照信息的定期(periodic)调用进行监视

```json
GET /_snapshot/my_backup/snapshot_1

```

`快照信息操作使用与快照操作相同的资源和线程池。 因此，在一个大分片快照时执行一个快照信息操作，将引起在返回结果之前，快照信息操作将等待可用资源，对于非常大的分片，等待时间可能很长`

- 获得关于快照的更直接和完整的信息，可以使用snapshot status命令

```json
GET /_snapshot/my_backup/snapshot_1/_status

```