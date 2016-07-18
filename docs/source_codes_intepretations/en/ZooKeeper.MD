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
### WatchRegistration
其中Z​KWatchManager类是管理这个client object **所有** Watches的。
ZKWatchManager维持着三个Map：
1. dataWatches
2. existWatches
3. childWatches

ZKWatchManager的主要作用是帮助外层ZooKeeper类管理这三个Map。但主要是删除(removeWatches())和查询是否存在(containsWatcher())。往这三个Map里添加Watches的方法是定义在WatchRegistration以及继承它的三个子类中的。

### WatchRegistration (ExistsWatchRegistration, DataWatchRegistration, ChildWatchRegistration)
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

        public void register(int rc) {}
    }

    class ExistsWatchRegistration extends WatchRegistration {}
    class DataWatchRegistration extends WatchRegistration {}
    class ChildWatchRegistration extends WatchRegistration {}

    public enum States {
        CONNECTING, ASSOCIATING, CONNECTED, CONNECTEDREADONLY,
        CLOSED, AUTH_FAILED, NOT_CONNECTED;
    }

    // Constructor: 9 in total
    public ZooKeeper(...) {}

    // TODO:

    // Close "THIS" client object.
    public synchronized void close() throws InterruptedException {}

    /**
     * Because the client might be in the subnode, this method
     * change the the path from the client's view to servers' view.
     * (client): /b/c --> (server): /a/b/c
     */
    private String prependChroot(String clientPath) {}

    /**
     * Create a node with the given path.  
     * Will trigger watches of getData and getChildren if succeed.
     * 2 synchronous and 2 asynchronous overload methods in total.
     */
    public String create(...) {}

    /**
     * Delete the node with the given path.
     * Will trigger watches of exists and getChildren of its parent node if succeed.
     * 1 synchronous and 1 asynchronous overload methods in total.
     */
    public void delete(...) {}

    /**
     * Return the stat of the node of the given path.  Return null if no such a node exists.
     * Will left a watch triggered by a successful creates/delete.
     * 2 synchronous and 2 asynchronous overload methods in total.
     */
     public void exists(...) {}

    /**
     * return the data and the stat of the node of the given path.
     * Will left a watch triggered by a successful operation that sets data on the node.
     * 2 synchronous and 2 asynchronous overload methods in total.
     */
     public byte[] getData(...) {}

    /**
     * Set the data for the node of the given path if:
     * 1. Such a node exists
     * 2. The given version matches the version of the node.
     * Will trigger all the getData watches on the node.
     * 1 synchronous and 1 asynchronous overload methods in total.
     */

    /**
     * Return the last committed configuration and stat of the configuration.
     * Will left a watch on the configuration node triggered by a successful reconfig.
     * 2 synchronous and 2 asynchronous overload methods in total.
     */
     public byte[] getConfig(...) {}

    /**
     * Reconfigure - add/remove servers.  Return the new configuration.
     * 2 synchronous and 2 asynchronous overload methods in total.
     */
     public void reconfig(...) {}

    /**
     * Return the ACL and stat of the node of the given path.
     * 1 synchronous and 1 asynchronous overload methods in total.
     */
     public void getACL(...) {}

    /**
     * Set the ACL for the node of the given path if:
     * 1. Such a node exists.
     * 2. The given version matches the version of the node.
     * 1 synchronous and 1 asynchronous overload methods in total.
     */
     public Stat setACL(...) {}

    /**
     * Return the list of the children of the node of the given path.
     * 4 synchronous and 4 asynchronous overload methods in total.
     */
    public List<String> getChildren(...) {}

    /**
     * For the given ZNode path, removes the specified watcher of given watcherType.
     * 1 synchronous and 1 asynchronous overload methods in total.
     */
    public void removeWatches(...) {}

    /**
     * For the given ZNode path, removes *ALL* the registered watchers of given watcherType.
     * 1 synchronous and 1 asynchronous overload methods in total.
     */
    public void removeAllWatches(...) {}

    /**
     * String representation of this ZooKeeper client.
     */
    public String toString() {}

    // Some test methods remaining.
}

```
