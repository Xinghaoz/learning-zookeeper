ZooKeeper中最最重要的两个概念：**ZNodes** 和 **Watches**。
ZNode是ZooKeeper中的数据节点，用来储存不超过1MB大小的数据，跟文件系统中的“文件”的概念相类似。Watches是监测ZNodes变化的工具。

### ZNodes

#### Znode的特性：

1. 在Zookeeper中，虽然说ZNode像是一个“文件”，但每个ZNode可以有子节点，由此构成树状结构。这类似于文件系统中一个文件同时也可以是目录。不同的是，每个ZNode都有着与其关联的数据以及子节点的数据。

2. 父节点和子节点由"/"隔开，类似于文件目录的形式：
  * /a
  * /a/b/c
3. Zookeeper只支持绝对路径，不支持相对路径。所以路径中不能带.和..，例如
  * /a/./c
  * /../a
都是不合法的

#### ZooKeeper里的时间

1. Zxid (ZooKeeper Transaction Id)
每次ZooKeeper的状态变化都会有个与其对应的zxid作为时间戳。zxid用来保证所有的变化都是有序的。例如：如果zxid1 < zxid2，那么zxid1的变化发生在zxid2之前

2. Version numbers
每次ZNode有变化就会导致version numbers其中的一个变化。如果熟悉版本控制（例如git）就知道版本号的意义。下面会说到，在ZNode中有三种版本号，分别是version, cversion, aversion。

3. Ticks
ZooKeeper中的最小的时间单位，可以在配置文件中自行设置。例如，设置为2000(ms)，Zookeeper中最小的时间单位就是2秒。设置ZooKeeper中的timeout等都是以Tick为单位的。如果你设置的时间比tick小，那么server就会警告你timeout还是以最小的单位2000ms来计算。

4. Real time
ZooKeeper不用现实时间。（除了ctime与mtime是相对于epoch的）

#### 知道ZooKeeper里关于时间与时间戳的定义，现在就可以来看ZNode中保存了什么。每个ZNode维持着许多状态，其中包含:

1. czxid:
这个ZNode创建时的zxid

2. mzxid:
这个ZNode最后一次更改的zxid

3. ctime
这个ZNode创建时距epoch(1970-01-01 00:00:00 UTC)的时间（以毫秒为单位）

4. mtime
这个ZNode创建时距epoch(1970-01-01 00:00:00 UTC)的时间（以毫秒为单位）

5. version
这个ZNode的变化的次数

6. cversion
这个ZNode的子节点变化的次数

7. aversion
这个ZNode的ACL变化的次数

8. ephemeralOwner
如果这个ZNode是ephemeral node　－　这个ZNode的所有着的session id
如果这个ZNode不是ephemeral node －　0

9. dataLength
这个ZNode数据域的长度

10. numChildren
这个ZNode的子节点数量

---

### ZooKeeper Watches
所有读操作：**getData()**, **getChildren()**, **exists()** 都会在对应的ZNode上面设置一个Watch。
一个watch是一个一次性触发器，当其监测的数据有变动时返回消息给client，它有三个重要的特性:

1. 一次性的，触发一次就没了，需要再次设置

2. 当client设置watch后，保证下一个变化这个watch一定会提醒（此处有疑问－a client will never see a change for which it has set a watch until it first sees the watch event.）

3. 监测的数据类型：getData()和exists()设置的是data watch，getChildren()设置的是child watch。具体如下：
  * setData()会触发data watch
  * crete()会触发data watch以及父节点的child watch
  * delete()会触发data watch以及child watch

#### Watches能够保证：
1. 每个watch相对于其它的watches, events, asynchronous replies都是有序的。
2. client会先接收到watch event（如果之前已经设置了watch)，之后才会看见相应data的变化。就是说，保证在变化之前一定先告诉你一下让你知道有变化。
3. ZooKeeper的event的顺便与ZooKeeper service上更新的顺序是一致的。

#### Watches的注意事项：
1. 是一次性的，用完了之后要再申请
2. 因为是一次性的，所以不能保证每次变化都会收到通知。很容易理解，就是变化太快了，就算watch用完了马上设置也可能赶不上，从而漏掉一些变化。不过只是你没有“看到”这些变化，这些变化还是实实在在地按顺序地执行了。
3. 一个事件只能提醒一次。例如，getData和exists分别对同一个data设置了watch。然后这个data被delete，理论上两个watch都会被触发，但是client只能看到一次提醒。
4. 当你与server失去连接的时候，在重连之前是不会接收到任何watches传过来的消息的。
