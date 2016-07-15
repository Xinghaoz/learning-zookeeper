## Data Model
ZooKeeper中最最重要的两个概念：ZNodes和Watches。
ZNode是ZooKeeper中的基本功能单位，也就是常说的“节点”。Watches一个一次性的用于给client提供数据监测的工具。

### ZNodes
Zookeeper中的功能单位叫做ZNode，每个ZNode维持着一个状态结构，其中包含
1. 数据变更的版本号
2. ACL(Access Control List)变更的版本号

Znode的特性：
1. 在Zookeeper中，每个ZNode可以有儿子，由此构成树状结构。这类似于文件系统一个文件同时也可以是目录。
2. 父节点和子节点由"/"隔开，类似于文件目录的形式：
..* /a
..* /a/b/c
3. Zookeeper只支持绝对路径，不支持相对路径。

### ZooKeeper Watches
所有读操作：getData(), getChildren(), exists()都会设置一个Watch。
一个watch是一个一次性触发器，当其监测的数据有变动时返回消息给client，它有三个重要的特性:
1. 一次性的，触发一次就没了，需要再次设置
2. 当client设置watch后，除非他看到watch event，否则他永远不会看到任何变化（a client will never see a change for which it has set a watch until it first sees the watch event. 这句话到底什么意思？）
3. 监测的数据类型：getData()和exists()设置的是data watch，getChildren()设置的是child watch。具体如下：
..* setData()会触发data watch
..* crete()会触发data watch以及父节点的child watch
..* delete()会触发data watch以前child watch
