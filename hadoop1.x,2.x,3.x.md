## hadoop1.0
- MapReduce（计算+资源调度）
- HDFS（数据存储）
- Common（辅助工具）

## hadoop2.0
- MapReduce（计算）
- Yarn（资源调度）
- HDFS（数据存储）
- Common（辅助工具）

## 在Hadoop1.x时代中，MapReduce同时负责业务逻辑运算和资源的调度，耦合性大
## 在Hadoop2.x中，增加了Yarn，Yarn只负责资源的调度MapReduce只负责运算
## hadoop3.x在组成上没有变化

### 常用端口号说明

端口名称 | Hadoop2.x | Hadoop3.x
---|---|---
NameNode内部通信端口 | 8020/9000 | 8020/9000/9820
NameNode HTTP UI | 50070 | 9870
MapReduce 查看执行任务端口 | 8088 | 8088
历史服务器通信端口 | 19888 | 19888

### 常用配置文件
- hadoop3.x workers core-site.xml hdfs-site.xml yarn-site.xml mapred-site.xml
- hadoop2.x slavers core-site.xml hdfs-site.xml yarn-site.xml mapred-site.xml



