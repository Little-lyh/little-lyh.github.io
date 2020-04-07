---
layout: post
title: Mongo 分片组件之 Config Server-42
date: 2018-12-10 11:35:23 +0800
categories: [sql]
tags: [sql, nosql, mongo, TODO, sh]
published: true
excerpt: Mongo 分片组件之 Config Server
---

# Config Server

> 重要

从3.4开始，不再支持使用不推荐使用的镜像mongod实例作为配置服务器（SCCC）。

在将分片群集升级到3.4之前，必须将配置服务器从SCCC转换为CSRS。

要将配置服务器从SCCC转换为CSRS，请参阅MongoDB 3.4手册将配置服务器升级到副本集。

配置服务器存储分片集群的元数据。元数据反映了分片集群中所有数据和组件的状态和组织。元数据包括每个分片上的块列表以及定义块的范围。

mongos实例缓存此数据并使用它将读取和写入操作路由到正确的分片。当群集有元数据更改时，mongos会更新缓存，例如Chunk Splits或添加分片。 

Shards还从配置服务器读取块元数据。

配置服务器还存储身份验证配置信息，例如基于角色的访问控制或群集的内部身份验证设置。

MongoDB还使用配置服务器来管理分布式锁。

每个分片群集必须具有自己的配置服务器。不要将相同的配置服务器用于不同的分片群集。

> 警告

在配置服务器上执行的管理操作可能会对分片群集性能和可用性产生重大影响。

根据受影响的配置服务器的数量，群集可能是只读的或离线一段时间。

# 副本集配置服务器

版本3.4中已更改。

从MongoDB 3.2开始，可以将分片群集的配置服务器部署为副本集（CSRS），而不是三个镜像配置服务器（SCCC）。 

使用配置服务器的副本集可以提高配置服务器的一致性，因为MongoDB可以利用配置数据的标准副本集读写协议。 

此外，使用配置服务器的副本集允许分片群集具有3个以上的配置服务器，因为副本集最多可包含50个成员。 

要将配置服务器部署为副本集，配置服务器必须运行WiredTiger存储引擎。

在3.4版本中，MongoDB删除了对SCCC配置服务器的支持。

用于配置服务器时，以下限制适用于副本集配置：

1. 必须有零仲裁者。

2. 必须没有延迟的节点。

3. 必须构建索引（即没有成员应将buildIndexes设置设置为false）。

# 配置服务器上的读写操作

配置服务器上存在admin数据库和配置数据库。

## 写入配置服务器

admin数据库包含与身份验证和授权相关的集合以及供内部使用的其他系统集合。

config数据库包含包含分片群集元数据的集合。当元数据发生更改时，MongoDB会将数据写入配置数据库，例如在块迁移或块拆分之后。

用户应避免在正常操作或维护过程中直接写入配置数据库。

写入配置服务器时，MongoDB使用 `majority` 写入问题。

## 从配置服务器读取

MongoDB从管理数据库中读取身份验证和授权数据以及其他内部用途。

当mongos启动时或在元数据发生更改后，MongoDB从配置数据库中读取，例如在块迁移之后。 

Shards还从配置服务器读取块元数据。

从副本集配置服务器读取时，MongoDB使用“读取关注”(Read Concern)级别为 `majority`。

# 配置服务器可用性

如果配置服务器副本集丢失其主节点而无法选择主节点，则群集的元数据将变为只读。您仍然可以从分片读取和写入数据，但在副本集可以选择主分区之前，不会发生块迁移或块分割。

在分片群集中，mongod和mongos实例监视分片群集中的副本集（例如，分片副本集，配置服务器副本集）。

如果所有配置服务器都不可用，则群集可能无法运行。为确保配置服务器保持可用且完整，配置服务器的备份至关重要。配置服务器上的数据与存储在群集中的数据相比较小，并且配置服务器的活动负载相对较低。

对于3.2分片群集，如果连续不成功监视配置服务器副本集的次数超过replMonitorMaxFailedChecks参数值，则在重新启动实例之前，监视mongos或mongod实例将变为不可用。

有关解决方法，请参阅MongoDB 3.2。

有关详细信息，请参阅配置服务器副本集成员变为不可用。

# 分片群集元数据

配置服务器在配置数据库中存储元数据。

> 重要

在对配置服务器进行任何维护之前，始终备份配置数据库。

要访问配置数据库，请从mongo shell发出以下命令：

```
use config
```

通常，您不应该直接编辑配置数据库的内容。 config数据库包含以下集合：

- changelog

- chunks

- collections

- databases

- lockpings

- locks

- mongos

- settings

- shards

- version

有关这些集合及其在分片集群中的角色的更多信息，请参阅配置数据库。 

有关元数据读取和更新的更多信息，请参阅配置服务器上的读取和写入操作。

# 分片群集安全性

使用内部身份验证可强制实施群集内安全性，并防止未经授权的群集组件访问群集。 

您必须使用适当的安全设置启动群集中的每个mongod，以强制执行内部身份验证。

有关部署安全分片群集的教程，请参阅使用密钥文件访问控制部署分片群集。

# 参考资料

https://docs.mongodb.com/manual/core/sharded-cluster-config-servers/

* any list
{:toc}