## HDFS联邦机制
```
https://hadoop.apache.org/docs/r3.1.3/hadoop-project-dist/hadoop-hdfs/Federation.html
```
HDFS 有两个主要层：
- 命名空间
    - 由目录、文件和块组成。
    - 它支持所有与命名空间相关的文件系统操作，如创建、删除、修改和列出文件和目录。
- 块存储服务，它有两个部分：
    - 块管理（在 Namenode 中执行）
        - 通过处理注册和定期心跳来提供 Datanode 集群成员资格。
        - 处理块报告并维护块的位置。
        - 支持创建、删除、修改、获取区块位置等区块相关操作。
        - 管理副本放置、复制不足的块的复制，并删除复制过度的块。
  - 存储 - 由 Datanodes 通过在本地文件系统上存储块并允许读/写访问来提供。
之前的 HDFS 架构只允许整个集群使用一个命名空间。在该配置中，单个 Namenode 管理命名空间。HDFS 联盟通过向 HDFS 添加对多个 Namenodes/命名空间的支持来解决此限制。
### 多个命名节点/命名空间
为了横向扩展名称服务，联邦使用多个独立的 Namenodes/namespace。Namenodes 是联合的；Namenodes 是独立的，不需要相互协调。Datanodes 被所有 Namenodes 用作块的公共存储。每个 Datanode 向集群中的所有 Namenode 注册。Datanodes 定期发送心跳和块报告。它们还处理来自 Namenodes 的命令。