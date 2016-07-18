ZooKeeper.java中包含了ZooKeeper这个类。 ZooKeeper这个类是client的主类，client中所有的操作（方法）包括对ZNodes的create, delete, getData, setData以及对Watches操作（方法）都定义在了这个类里面。

这个类总共包含3000多行代码。其中构造函数以及一些主要的方法也重载了很多次，并且几乎都带有synchronous和asynchronous两种不同的版本。在此主要讲解最重要的部分，即这些类和方法”干了什么“，比较重要的细节可能会挑出来讲，很多内置变量与内置方法会略去不提。

ZooKeeper类大致结构如下:
```java
class ZooKeeper {
    static class ZKWatchManager implements ClientWatchManager {}

    abstract class WatchRegistration{}
    class ExistsWatchRegistration extends WatchRegistration {}
    class DataWatchRegistration extends WatchRegistration {}
    class ChildWatchRegistration extends WatchRegistration {}
}
```
### ZKWatchManager
其中Z​KWatchManager类是管理这个client object **所有** Watches的。
ZKWatchManager维持着三个Map：
1. dataWatches
2. existWatches
3. childWatches

ZKWatchManager的主要作用是帮助外层ZooKeeper类管理这三个Map。但主要是删除(removeWatches())和查询是否存在(containsWatcher())。往这三个Map里添加Watches的方法是定义在WatchRegistration以及继承它的三个子类中的。

### 虚类WatchRegistration以及实现它的三个实类(ExistsWatchRegistration, DataWatchRegistration, ChildWatchRegistration)
这三个类提供了往ZKWatchManager里面三个Map注册（register）的方法，每一个分别对应着一个Watch类型。

在ZNodes and Watches中提到过:

`所有读操作：**getData()**, **getChildren()**, **exists()** 都会在对应的ZNode上面设置一个Watch。`

例如：在getData()这个方法中会new一个DataWatchRegistration的实例，往这个实例的构造方法里面传入自己的clientPath，就实现了在一个ZNode上添加Watch的操作：
```java
public byte[] getData(final String path, Watcher watcher, Stat stat)
        throws KeeperException, InterruptedException
     {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        // the watch contains the un-chroot path
        WatchRegistration wcb = null;
        if (watcher != null) {
            // 这里新建了一个DataWatchRegistration的实例，并往这个实例里传入自己的clientPath
            wcb = new DataWatchRegistration(watcher, clientPath);
        }

        // Omitting...
      }

```



---
## More details
``` java
class ZooKeeper {
    protected final ClientCnxn cnxn;
    private final HostProvider hostProvider;

    public void updateServerList(String connectString) throws IOException {}
    //　分别得到ZWWatchManager下面的三个Map
    List<String> getDataWatches() {}
    List<String> getExistWatches() {}
    List<String> getChildWatches() {}

    static class ZKWatchManager implements ClientWatchManager {
        private final Map<String, Set<Watcher>> dataWatches;
        private final Map<String, Set<Watcher>> existWatches;
        private final Map<String, Set<Watcher>> childWatches;
        private boolean disableAutoWatchReset;

        public Map<EventType, Set<Watcher>> remove Watcher(...) throws KeeperException {}

        void containsWatcher(...) throws NoWatcherException {}
        protected boolean removeWatches(...) throws KeeperException {}
        public Set<Watcher> materialize(...) {}
    }

    abstract class WatchRegistration {
        private Watcher watcher;
        private String clientPath;

        abstract protected Map<String, Set<Watcher>> getWatches(int rc);

        // 这个函数把这个watch写进三个对应的Map之中的一个
        public void register(int rc) {}
    }

    class ExistsWatchRegistration extends WatchRegistration {}
    class DataWatchRegistration extends WatchRegistration {}
    class ChildWatchRegistration extends WatchRegistration {}

    // 状态枚举变量，一共有7个，形容这个client可能处在的7个状态。
    // 1. 如果状态不是CLOSED或AUTH_FAILED，isAlived()就返回true
    // 2. 如果状态是CONNECTED和CONNECTEDREADONLY，isConnected()就返回true
    public enum States {
        CONNECTING, ASSOCIATING, CONNECTED, CONNECTEDREADONLY,
        CLOSED, AUTH_FAILED, NOT_CONNECTED;
    }

    // 一个有9个不构造函数
    public ZooKeeper(...) {}

    // TODO:

    // 关闭”THIS"client实例
    public synchronized void close() throws InterruptedException {}

    /**
     * 因为在设置root的时候可以设置在子节点下，所以这个方法返回的是完全路径
     * (client): /b/c --> (server): /a/b/c
     */
    private String prependChroot(String clientPath) {}

    /**
     * 根据给定的path新建一个ZNode。
     * 如果运行成功，这个方法会触发这个ZNode上的getData的watch和父节点的getChildren
     * 一共有2个同步和2个异步的重载的方法
     */
    public String create(...) {}

    /**
     * 根据给定的path删除一个ZNode
     * 如果运行成功，这个方法会触发这个ZNode上的getData和父节点的getChildren。
     * 一共有1个同步的和1个异步的重载的方法
     */
    public void delete(...) {}

    /**
     * 返回给定的path的ZNode的状态。返回null如果这个ZNode不存在。
     * 会在这个ZNode上面设置一个watch，这个watch会被crete/delete触发。
     * 一个共有2个同步的和2个异步的重载的方法
     */
     public void exists(...) {}

    /**
     * 返回数据以及这个节点的状态。
     * 会在这个ZNode上面设置一个watch，这个watch会被对这个data进行的操作触发。
     * 一共有2个同步的和两个异步的重载的方法
     */
     public byte[] getData(...) {}

    /**
     * 对给定的ZNode上的数据进行设置，如果满足以下条件的话：
     * 1. 这个给定的节点存在
     * 2. 给定的版本匹配这个ZNode的版本
     * 会触发getData设置的watches
     * 一个有1个同步的和1个异步的重载的方法
     */
     public Stat setData(...) {}

    /**
     * 返回最后一次commit的配置和这个配置的状态。
     * 会在这个ZNode上设置一个watch，这个watch会被reconifg触发。
     * 一共有2个同步的和2个异步的重载的方法
     */
     public byte[] getConfig(...) {}

    /**
     * Reconfigure - 添加或删除servers。返回最新的配置。
     * 一共有2个同步的和2个异步的重载的方法
     */
     public void reconfig(...) {}

    /**
     * 返回给定path上的ACL以及状态
     * 一共有1个同步的和1个异步的重载的方法
     */
     public void getACL(...) {}

    /**
     * 对给定path上ZNode的ACL进行设置，如果满足以下条件的话：
     * 1.　这个给定的ZNode存在
     * 2. 给定的版本号匹配这个ZNode上的版本号
     * 一共有1个同步的和1个异步的重载的方法
     */
     public Stat setACL(...) {}

    /**
     * 返回给定path的ZNode的子节点列表
     * 一共有4个同步的和4个异步的重载的方法
     */
    public List<String> getChildren(...) {}

    /**
     * 对于给定path下的ZNode，移除给定类型的watchr
     * 一共有1个同步的和1个异步的重载的​方法
     */
    public void removeWatches(...) {}

    /**
     * 对于给定path下的ZNode，移除*所有*注册了的给定的类型的watchers
     * 一共有1个同步的和1个异步的重载的方法
     */
    public void removeAllWatches(...) {}

    /**
     * 返回这个ZooKeeper client的String的表达
     */
    public String toString() {}

    // 剩下的是一些测试用的方法
}

```
