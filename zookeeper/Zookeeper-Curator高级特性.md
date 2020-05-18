**提醒：首先你必须添加curator-recipes依赖，下文仅仅对recipes一些特性的使用进行解释和举例，不打算进行源码级别的探讨**

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```

**重要提醒：强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，curator-recipes下的所有Api将会失效或者过期，尽管后面所有的例子都没有使用到ConnectionStateListener。**

可以添加test module的依赖方便进行测试，不需要启动真实的zookeeper服务端：

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-test</artifactId>
    <version>2.12.0</version>
</dependency>
```

### 缓存

Zookeeper原生支持通过注册Watcher来进行事件监听，但是开发者需要反复注册(Watcher只能单次注册单次使用)。Cache是Curator中对事件监听的包装，可以看作是对事件监听的本地缓存视图，能够自动为开发者处理反复注册监听。Curator提供了三种Watcher(Cache)来监听结点的变化。

#### Path Cache

Path Cache用来监控一个ZNode的子节点. 当一个子节点增加， 更新，删除时， Path Cache会改变它的状态， 会包含最新的子节点， 子节点的数据和状态，而状态的更变将通过PathChildrenCacheListener通知。

实际使用时会涉及到四个类：

- PathChildrenCache
- PathChildrenCacheEvent
- PathChildrenCacheListener
- ChildData

通过下面的构造函数创建Path Cache:

```java
public PathChildrenCache(CuratorFramework client, String path, boolean cacheData)
```

想使用cache，必须调用它的`start`方法，使用完后调用`close`方法。 可以设置StartMode来实现启动的模式，

StartMode有下面几种：

1. NORMAL：正常初始化。
2. BUILD_INITIAL_CACHE：在调用`start()`之前会调用`rebuild()`。
3. POST_INITIALIZED_EVENT： 当Cache初始化数据后发送一个PathChildrenCacheEvent.Type#INITIALIZED事件

`public void addListener(PathChildrenCacheListener listener)`可以增加listener监听缓存的变化。

`getCurrentData()`方法返回一个`List<ChildData>`对象，可以遍历所有的子节点。

**设置/更新、移除其实是使用client (CuratorFramework)来操作, 不通过PathChildrenCache操作：**

```java
public class PathCacheDemo {

	private static final String PATH = "/example/pathCache";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
		client.start();
		PathChildrenCache cache = new PathChildrenCache(client, PATH, true);
		cache.start();
		PathChildrenCacheListener cacheListener = (client1, event) -> {
			System.out.println("事件类型：" + event.getType());
			if (null != event.getData()) {
				System.out.println("节点数据：" + event.getData().getPath() + " = " + new String(event.getData().getData()));
			}
		};
		cache.getListenable().addListener(cacheListener);
		client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test01", "01".getBytes());
		Thread.sleep(10);
		client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test02", "02".getBytes());
		Thread.sleep(10);
		client.setData().forPath("/example/pathCache/test01", "01_V2".getBytes());
		Thread.sleep(10);
		for (ChildData data : cache.getCurrentData()) {
			System.out.println("getCurrentData:" + data.getPath() + " = " + new String(data.getData()));
		}
		client.delete().forPath("/example/pathCache/test01");
		Thread.sleep(10);
		client.delete().forPath("/example/pathCache/test02");
		Thread.sleep(1000 * 5);
		cache.close();
		client.close();
		System.out.println("OK!");
	}
}
```

**注意：**如果new PathChildrenCache(client, PATH, true)中的参数cacheData值设置为false，则示例中的event.getData().getData()、data.getData()将返回null，cache将不会缓存节点数据。

**注意：**示例中的Thread.sleep(10)可以注释掉，但是注释后事件监听的触发次数会不全，这可能与PathCache的实现原理有关，不能太过频繁的触发事件！

#### Node Cache

Node Cache与Path Cache类似，Node Cache只是监听某一个特定的节点。它涉及到下面的三个类：

- `NodeCache` - Node Cache实现类
- `NodeCacheListener` - 节点监听器
- `ChildData` - 节点数据

**注意：**使用cache，依然要调用它的`start()`方法，使用完后调用`close()`方法。

getCurrentData()将得到节点当前的状态，通过它的状态可以得到当前的值。

```java
public class NodeCacheDemo {

	private static final String PATH = "/example/cache";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
		client.start();
		client.create().creatingParentsIfNeeded().forPath(PATH);
		final NodeCache cache = new NodeCache(client, PATH);
		NodeCacheListener listener = () -> {
			ChildData data = cache.getCurrentData();
			if (null != data) {
				System.out.println("节点数据：" + new String(cache.getCurrentData().getData()));
			} else {
				System.out.println("节点被删除!");
			}
		};
		cache.getListenable().addListener(listener);
		cache.start();
		client.setData().forPath(PATH, "01".getBytes());
		Thread.sleep(100);
		client.setData().forPath(PATH, "02".getBytes());
		Thread.sleep(100);
		client.delete().deletingChildrenIfNeeded().forPath(PATH);
		Thread.sleep(1000 * 2);
		cache.close();
		client.close();
		System.out.println("OK!");
	}
}
```

**注意：**示例中的Thread.sleep(10)可以注释，但是注释后事件监听的触发次数会不全，这可能与NodeCache的实现原理有关，不能太过频繁的触发事件！

**注意：**NodeCache只能监听一个节点的状态变化。

#### Tree Cache

Tree Cache可以监控整个树上的所有节点，类似于PathCache和NodeCache的组合，主要涉及到下面四个类：

- TreeCache - Tree Cache实现类
- TreeCacheListener - 监听器类
- TreeCacheEvent - 触发的事件类
- ChildData - 节点数据

```java
public class TreeCacheDemo {

	private static final String PATH = "/example/cache";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
		client.start();
		client.create().creatingParentsIfNeeded().forPath(PATH);
		TreeCache cache = new TreeCache(client, PATH);
		TreeCacheListener listener = (client1, event) ->
				System.out.println("事件类型：" + event.getType() +
						" | 路径：" + (null != event.getData() ? event.getData().getPath() : null));
		cache.getListenable().addListener(listener);
		cache.start();
		client.setData().forPath(PATH, "01".getBytes());
		Thread.sleep(100);
		client.setData().forPath(PATH, "02".getBytes());
		Thread.sleep(100);
		client.delete().deletingChildrenIfNeeded().forPath(PATH);
		Thread.sleep(1000 * 2);
		cache.close();
		client.close();
		System.out.println("OK!");
	}
}
```

**注意：**在此示例中没有使用Thread.sleep(10)，但是事件触发次数也是正常的。

**注意：**TreeCache在初始化(调用`start()`方法)的时候会回调`TreeCacheListener`实例一个事TreeCacheEvent，而回调的TreeCacheEvent对象的Type为INITIALIZED，ChildData为null，此时`event.getData().getPath()`很有可能导致空指针异常，这里应该主动处理并避免这种情况。

### Leader选举

在分布式计算中， **leader elections**是很重要的一个功能， 这个选举过程是这样子的： 指派一个进程作为组织者，将任务分发给各节点。 在任务开始前， 哪个节点都不知道谁是leader(领导者)或者coordinator(协调者). 当选举算法开始执行后， 每个节点最终会得到一个唯一的节点作为任务leader. 除此之外， 选举还经常会发生在leader意外宕机的情况下，新的leader要被选举出来。

在zookeeper集群中，leader负责写操作，然后通过Zab协议实现follower的同步，leader或者follower都可以处理读操作。

Curator 有两种leader选举的recipe,分别是**LeaderSelector**和**LeaderLatch**。

前者是所有存活的客户端不间断的轮流做Leader，大同社会。后者是一旦选举出Leader，除非有客户端挂掉重新触发选举，否则不会交出领导权。

#### LeaderLatch

LeaderLatch有两个构造函数：

```java
public LeaderLatch(CuratorFramework client, String latchPath)
public LeaderLatch(CuratorFramework client, String latchPath,  String id)
```

LeaderLatch的启动：

**leaderLatch.start( );**

一旦启动，LeaderLatch会和其它使用相同latch path的其它LeaderLatch交涉，然后其中一个最终会被选举为leader，可以通过`hasLeadership`方法查看LeaderLatch实例是否leader：

**leaderLatch.hasLeadership( );** //返回true说明当前实例是leader

类似JDK的CountDownLatch， LeaderLatch在请求成为leadership会block(阻塞)，一旦不使用LeaderLatch了，必须调用`close`方法。 如果它是leader,会释放leadership， 其它的参与者将会选举一个leader。

```java
public void await() throws InterruptedException,EOFException
/*Causes the current thread to wait until this instance acquires leadership
unless the thread is interrupted or closed.*/
public boolean await(long timeout,TimeUnit unit)throws InterruptedException
```

**异常处理：** LeaderLatch实例可以增加ConnectionStateListener来监听网络连接问题。 当 SUSPENDED 或 LOST 时, leader不再认为自己还是leader。当LOST后连接重连后RECONNECTED,LeaderLatch会删除先前的ZNode然后重新创建一个。LeaderLatch用户必须考虑导致leadership丢失的连接问题。 强烈推荐你使用ConnectionStateListener。

一个LeaderLatch的使用例子：

```java
public class LeaderLatchDemo extends BaseConnectionInfo {
	protected static String PATH = "/francis/leader";
	private static final int CLIENT_QTY = 10;


	public static void main(String[] args) throws Exception {
		List<CuratorFramework> clients = Lists.newArrayList();
		List<LeaderLatch> examples = Lists.newArrayList();
		TestingServer server=new TestingServer();
		try {
			for (int i = 0; i < CLIENT_QTY; i++) {
				CuratorFramework client
						= CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(20000, 3));
				clients.add(client);
				LeaderLatch latch = new LeaderLatch(client, PATH, "Client #" + i);
				latch.addListener(new LeaderLatchListener() {

					@Override
					public void isLeader() {
						// TODO Auto-generated method stub
						System.out.println("I am Leader");
					}

					@Override
					public void notLeader() {
						// TODO Auto-generated method stub
						System.out.println("I am not Leader");
					}
				});
				examples.add(latch);
				client.start();
				latch.start();
			}
			Thread.sleep(10000);
			LeaderLatch currentLeader = null;
			for (LeaderLatch latch : examples) {
				if (latch.hasLeadership()) {
					currentLeader = latch;
				}
			}
			System.out.println("current leader is " + currentLeader.getId());
			System.out.println("release the leader " + currentLeader.getId());
			currentLeader.close();

			Thread.sleep(5000);

			for (LeaderLatch latch : examples) {
				if (latch.hasLeadership()) {
					currentLeader = latch;
				}
			}
			System.out.println("current leader is " + currentLeader.getId());
			System.out.println("release the leader " + currentLeader.getId());
		} finally {
			for (LeaderLatch latch : examples) {
				if (null != latch.getState())
				CloseableUtils.closeQuietly(latch);
			}
			for (CuratorFramework client : clients) {
				CloseableUtils.closeQuietly(client);
			}
		}
	}
}
```

可以添加test module的依赖方便进行测试，不需要启动真实的zookeeper服务端：

```
复制<dependency>    <groupId>org.apache.curator</groupId>    <artifactId>curator-test</artifactId>    <version>2.12.0</version></dependency>
```

首先我们创建了10个LeaderLatch，启动后它们中的一个会被选举为leader。 因为选举会花费一些时间，start后并不能马上就得到leader。
通过`hasLeadership`查看自己是否是leader， 如果是的话返回true。
可以通过`.getLeader().getId()`可以得到当前的leader的ID。
只能通过`close`释放当前的领导权。
`await`是一个阻塞方法， 尝试获取leader地位，但是未必能上位。

#### LeaderSelector

LeaderSelector使用的时候主要涉及下面几个类：

- LeaderSelector
- LeaderSelectorListener
- LeaderSelectorListenerAdapter
- CancelLeadershipException

核心类是LeaderSelector，它的构造函数如下：

```java
public LeaderSelector(CuratorFramework client, String mutexPath,LeaderSelectorListener listener)
public LeaderSelector(CuratorFramework client, String mutexPath, ThreadFactory threadFactory, Executor executor, LeaderSelectorListener listener)
```

类似LeaderLatch,LeaderSelector必须`start`: `leaderSelector.start();` 一旦启动，当实例取得领导权时你的listener的`takeLeadership()`方法被调用。而takeLeadership()方法只有领导权被释放时才返回。 当你不再使用LeaderSelector实例时，应该调用它的close方法。

**异常处理** LeaderSelectorListener类继承ConnectionStateListener。LeaderSelector必须小心连接状态的改变。如果实例成为leader, 它应该响应SUSPENDED 或 LOST。 当 SUSPENDED 状态出现时， 实例必须假定在重新连接成功之前它可能不再是leader了。 如果LOST状态出现， 实例不再是leader， takeLeadership方法返回。

**重要**: 推荐处理方式是当收到SUSPENDED 或 LOST时抛出CancelLeadershipException异常.。这会导致LeaderSelector实例中断并取消执行takeLeadership方法的异常.。这非常重要， 你必须考虑扩展LeaderSelectorListenerAdapter. LeaderSelectorListenerAdapter提供了推荐的处理逻辑。

下面的一个例子摘抄自官方：

```java
public class LeaderSelectorAdapter extends LeaderSelectorListenerAdapter implements Closeable {
	private final String name;
	private final LeaderSelector leaderSelector;
	private final AtomicInteger leaderCount = new AtomicInteger();

	public LeaderSelectorAdapter(CuratorFramework client, String path, String name) {
		this.name = name;
		leaderSelector = new LeaderSelector(client, path, this);
		leaderSelector.autoRequeue();
	}

	public void start() throws IOException {
		leaderSelector.start();
	}

	@Override
	public void close() throws IOException {
		leaderSelector.close();
	}

	@Override
	public void takeLeadership(CuratorFramework client) throws Exception {
		final int waitSeconds = (int) (5 * Math.random()) + 1;
		System.out.println(name + " is now the leader. Waiting " + waitSeconds + " seconds...");
		System.out.println(name + " has been leader " + leaderCount.getAndIncrement() + " time(s) before.");
		try {
			Thread.sleep(TimeUnit.SECONDS.toMillis(waitSeconds));
		} catch (InterruptedException e) {
			System.err.println(name + " was interrupted.");
			Thread.currentThread().interrupt();
		} finally {
			System.out.println(name + " relinquishing leadership.\n");
		}
	}
}
```

你可以在takeLeadership进行任务的分配等等，并且不要返回，如果你想要要此实例一直是leader的话可以加一个死循环。调用 `leaderSelector.autoRequeue();`保证在此实例释放领导权之后还可能获得领导权。 在这里我们使用AtomicInteger来记录此client获得领导权的次数， 它是”fair”， 每个client有平等的机会获得领导权。

```java
public class LeaderSelectorDemo {

	protected static String PATH = "/francis/leader";
	private static final int CLIENT_QTY = 10;


	public static void main(String[] args) throws Exception {
		List<CuratorFramework> clients = Lists.newArrayList();
		List<LeaderSelectorAdapter> examples = Lists.newArrayList();
		TestingServer server = new TestingServer();
		try {
			for (int i = 0; i < CLIENT_QTY; i++) {
				CuratorFramework client
						= CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(20000, 3));
				clients.add(client);
				LeaderSelectorAdapter selectorAdapter = new LeaderSelectorAdapter(client, PATH, "Client #" + i);
				examples.add(selectorAdapter);
				client.start();
				selectorAdapter.start();
			}
			System.out.println("Press enter/return to quit\n");
			new BufferedReader(new InputStreamReader(System.in)).readLine();
		} finally {
			System.out.println("Shutting down...");
			for (LeaderSelectorAdapter exampleClient : examples) {
				CloseableUtils.closeQuietly(exampleClient);
			}
			for (CuratorFramework client : clients) {
				CloseableUtils.closeQuietly(client);
			}
			CloseableUtils.closeQuietly(server);
		}
	}
}
```

对比可知，LeaderLatch必须调用`close()`方法才会释放领导权，而对于LeaderSelector，通过`LeaderSelectorListener`可以对领导权进行控制， 在适当的时候释放领导权，这样每个节点都有可能获得领导权。从而，LeaderSelector具有更好的灵活性和可控性，建议有LeaderElection应用场景下优先使用LeaderSelector。

### 分布式锁

**提醒：**

1.推荐使用ConnectionStateListener监控连接的状态，因为当连接LOST时你不再拥有锁

2.分布式的锁全局同步， 这意味着任何一个时间点不会有两个客户端都拥有相同的锁。

#### 可重入共享锁—Shared Reentrant Lock

**Shared意味着锁是全局可见的**， 客户端都可以请求锁。 Reentrant和JDK的ReentrantLock类似，即可重入， 意味着同一个客户端在拥有锁的同时，可以多次获取，不会被阻塞。 它是由类`InterProcessMutex`来实现。 它的构造函数为：

```java
public InterProcessMutex(CuratorFramework client, String path)
```

通过`acquire()`获得锁，并提供超时机制：

```java
public void acquire()
//Acquire the mutex - blocking until it's available. Note: the same thread can call acquire re-entrantly. Each call to acquire must be balanced by a call to release()

public boolean acquire(long time,TimeUnit unit)
/**Acquire the mutex - blocks until it's available or the given time expires. Note: the same thread can call acquire re-entrantly. Each call to acquire that returns true must be balanced by a call to release()
    Parameters:
    time - time to wait
    unit - time unit
    Returns:
    true if the mutex was acquired, false if not**/
```

通过`release()`方法释放锁。 InterProcessMutex 实例可以重用。

**Revoking** ZooKeeper recipes wiki定义了可协商的撤销机制。 为了撤销mutex, 调用下面的方法：

```java
public void makeRevocable(RevocationListener<T> listener)
/**
将锁设为可撤销的. 当别的进程或线程想让你释放锁时Listener会被调用。
Parameters:
listener - the listener
**/
```

如果你请求撤销当前的锁， 调用`attemptRevoke()`方法,注意锁释放时`RevocationListener`将会回调。

```java
public static void attemptRevoke(CuratorFramework client,String path) throws Exception
/**
Utility to mark a lock for revocation. Assuming that the lock has been registered
with a RevocationListener, it will get called and the lock should be released. Note,
however, that revocation is cooperative.
Parameters:
client - the client
path - the path of the lock - usually from something like InterProcessMutex.getParticipantNodes()
**/
```

**二次提醒：错误处理** 还是强烈推荐你使用`ConnectionStateListener`处理连接状态的改变。 当连接LOST时你不再拥有锁。

首先让我们创建一个模拟的共享资源， 这个资源期望只能单线程的访问，否则会有并发问题。

```java
public class FakeLimitedResource {
	private final AtomicBoolean inUse = new AtomicBoolean(false);

	public void use() throws InterruptedException {
		// 真实环境中我们会在这里访问/维护一个共享的资源
		//这个例子在使用锁的情况下不会非法并发异常IllegalStateException
		//但是在无锁的情况由于sleep了一段时间，很容易抛出异常
		if (!inUse.compareAndSet(false, true)) {
			throw new IllegalStateException("Needs to be used by one client at a time");
		}
		try {
			Thread.sleep((long) (3 * Math.random()));
		} finally {
			inUse.set(false);
		}
	}
}
```

然后创建一个`InterProcessMutexDemo`类， 它负责请求锁， 使用资源，释放锁这样一个完整的访问过程。

```java
public class InterProcessMutexDemo {

	private InterProcessMutex lock;
	private final FakeLimitedResource resource;
	private final String clientName;

	public InterProcessMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
		this.resource = resource;
		this.clientName = clientName;
		this.lock = new InterProcessMutex(client, lockPath);
	}

	public void doWork(long time, TimeUnit unit) throws Exception {
		if (!lock.acquire(time, unit)) {
			throw new IllegalStateException(clientName + " could not acquire the lock");
		}
		try {
			System.out.println(clientName + " get the lock");
			resource.use(); //access resource exclusively
		} finally {
			System.out.println(clientName + " releasing the lock");
			lock.release(); // always release the lock in a finally block
		}
	}

	private static final int QTY = 5;
	private static final int REPETITIONS = QTY * 10;
	private static final String PATH = "/examples/locks";

	public static void main(String[] args) throws Exception {
		final FakeLimitedResource resource = new FakeLimitedResource();
		ExecutorService service = Executors.newFixedThreadPool(QTY);
		final TestingServer server = new TestingServer();
		try {
			for (int i = 0; i < QTY; ++i) {
				final int index = i;
				Callable<Void> task = new Callable<Void>() {
					@Override
					public Void call() throws Exception {
						CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
						try {
							client.start();
							final InterProcessMutexDemo example = new InterProcessMutexDemo(client, PATH, resource, "Client " + index);
							for (int j = 0; j < REPETITIONS; ++j) {
								example.doWork(10, TimeUnit.SECONDS);
							}
						} catch (Throwable e) {
							e.printStackTrace();
						} finally {
							CloseableUtils.closeQuietly(client);
						}
						return null;
					}
				};
				service.submit(task);
			}
			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);
		} finally {
			CloseableUtils.closeQuietly(server);
		}
	}
}
```

代码也很简单，生成10个client， 每个client重复执行10次 请求锁–访问资源–释放锁的过程。每个client都在独立的线程中。 结果可以看到，锁是随机的被每个实例排他性的使用。

既然是可重用的，你可以在一个线程中多次调用`acquire()`,在线程拥有锁时它总是返回true。

**你不应该在多个线程中用同一个InterProcessMutex**， 你可以在每个线程中都生成一个新的InterProcessMutex实例，它们的path都一样，这样它们可以共享同一个锁。

#### 不可重入共享锁—Shared Lock

这个锁和上面的`InterProcessMutex`相比，就是少了Reentrant的功能，也就意味着它不能在同一个线程中重入。这个类是`InterProcessSemaphoreMutex`,使用方法和`InterProcessMutex`类似

```java
public class InterProcessSemaphoreMutexDemo {

	private InterProcessSemaphoreMutex lock;
	private final FakeLimitedResource resource;
	private final String clientName;

	public InterProcessSemaphoreMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
		this.resource = resource;
		this.clientName = clientName;
		this.lock = new InterProcessSemaphoreMutex(client, lockPath);
	}

	public void doWork(long time, TimeUnit unit) throws Exception {
		if (!lock.acquire(time, unit))
		{
			throw new IllegalStateException(clientName + " 不能得到互斥锁");
		}
		System.out.println(clientName + " 已获取到互斥锁");
		if (!lock.acquire(time, unit))
		{
			throw new IllegalStateException(clientName + " 不能得到互斥锁");
		}
		System.out.println(clientName + " 再次获取到互斥锁");
		try {
			System.out.println(clientName + " get the lock");
			resource.use(); //access resource exclusively
		} finally {
			System.out.println(clientName + " releasing the lock");
			lock.release(); // always release the lock in a finally block
			lock.release(); // 获取锁几次 释放锁也要几次
		}
	}

	private static final int QTY = 5;
	private static final int REPETITIONS = QTY * 10;
	private static final String PATH = "/examples/locks";

	public static void main(String[] args) throws Exception {
		final FakeLimitedResource resource = new FakeLimitedResource();
		ExecutorService service = Executors.newFixedThreadPool(QTY);
		final TestingServer server = new TestingServer();
		try {
			for (int i = 0; i < QTY; ++i) {
				final int index = i;
				Callable<Void> task = new Callable<Void>() {
					@Override
					public Void call() throws Exception {
						CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
						try {
							client.start();
							final InterProcessSemaphoreMutexDemo example = new InterProcessSemaphoreMutexDemo(client, PATH, resource, "Client " + index);
							for (int j = 0; j < REPETITIONS; ++j) {
								example.doWork(10, TimeUnit.SECONDS);
							}
						} catch (Throwable e) {
							e.printStackTrace();
						} finally {
							CloseableUtils.closeQuietly(client);
						}
						return null;
					}
				};
				service.submit(task);
			}
			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);
		} finally {
			CloseableUtils.closeQuietly(server);
		}
		Thread.sleep(Integer.MAX_VALUE);
	}
}
```

运行后发现，有且只有一个client成功获取第一个锁(第一个`acquire()`方法返回true)，然后它自己阻塞在第二个`acquire()`方法，获取第二个锁超时；其他所有的客户端都阻塞在第一个`acquire()`方法超时并且抛出异常。

这样也就验证了`InterProcessSemaphoreMutex`实现的锁是不可重入的。

#### 可重入读写锁—Shared Reentrant Read Write Lock

类似JDK的**ReentrantReadWriteLock**。一个读写锁管理一对相关的锁。一个负责读操作，另外一个负责写操作。读操作在写锁没被使用时可同时由多个进程使用，而写锁在使用时不允许读(阻塞)。

此锁是可重入的。**一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁**。这也意味着**写锁可以降级成读锁， 比如请求写锁 —>请求读锁—>释放读锁 ---->释放写锁**。从读锁升级成写锁是不行的。

可重入读写锁主要由两个类实现：`InterProcessReadWriteLock`、`InterProcessMutex`。使用时首先创建一个`InterProcessReadWriteLock`实例，然后再根据你的需求得到读锁或者写锁，读写锁的类型是`InterProcessMutex`。

```java
public class ReentrantReadWriteLockDemo {

	private final InterProcessReadWriteLock lock;
	private final InterProcessMutex readLock;
	private final InterProcessMutex writeLock;
	private final FakeLimitedResource resource;
	private final String clientName;

	public ReentrantReadWriteLockDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
		this.resource = resource;
		this.clientName = clientName;
		lock = new InterProcessReadWriteLock(client, lockPath);
		readLock = lock.readLock();
		writeLock = lock.writeLock();
	}

	public void doWork(long time, TimeUnit unit) throws Exception {
		// 注意只能先得到写锁再得到读锁，不能反过来！！！
		if (!writeLock.acquire(time, unit)) {
			throw new IllegalStateException(clientName + " 不能得到写锁");
		}
		System.out.println(clientName + " 已得到写锁");
		if (!readLock.acquire(time, unit)) {
			throw new IllegalStateException(clientName + " 不能得到读锁");
		}
		System.out.println(clientName + " 已得到读锁");
		try {
			resource.use(); // 使用资源
			Thread.sleep(1000);
		} finally {
			System.out.println(clientName + " 释放读写锁");
			readLock.release();
			writeLock.release();
		}
	}

	private static final int QTY = 5;
	private static final int REPETITIONS = QTY ;
	private static final String PATH = "/examples/locks";

	public static void main(String[] args) throws Exception {
		final FakeLimitedResource resource = new FakeLimitedResource();
		ExecutorService service = Executors.newFixedThreadPool(QTY);
		final TestingServer server = new TestingServer();
		try {
			for (int i = 0; i < QTY; ++i) {
				final int index = i;
				Callable<Void> task = new Callable<Void>() {
					@Override
					public Void call() throws Exception {
						CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
						try {
							client.start();
							final ReentrantReadWriteLockDemo example = new ReentrantReadWriteLockDemo(client, PATH, resource, "Client " + index);
							for (int j = 0; j < REPETITIONS; ++j) {
								example.doWork(10, TimeUnit.SECONDS);
							}
						} catch (Throwable e) {
							e.printStackTrace();
						} finally {
							CloseableUtils.closeQuietly(client);
						}
						return null;
					}
				};
				service.submit(task);
			}
			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);
		} finally {
			CloseableUtils.closeQuietly(server);
		}
	}
}
```

#### 信号量—Shared Semaphore

一个计数的信号量类似JDK的Semaphore。 JDK中Semaphore维护的一组许可(**permits**)，而Curator中称之为租约(**Lease**)。 有两种方式可以决定semaphore的最大租约数。第一种方式是用户给定path并且指定最大LeaseSize。第二种方式用户给定path并且使用`SharedCountReader`类。**如果不使用SharedCountReader, 必须保证所有实例在多进程中使用相同的(最大)租约数量,否则有可能出现A进程中的实例持有最大租约数量为10，但是在B进程中持有的最大租约数量为20，此时租约的意义就失效了。**

这次调用`acquire()`会返回一个租约对象。 客户端必须在finally中close这些租约对象，否则这些租约会丢失掉。 但是， 但是，如果客户端session由于某种原因比如crash丢掉， 那么这些客户端持有的租约会自动close， 这样其它客户端可以继续使用这些租约。 租约还可以通过下面的方式返还：

```java
public void returnAll(Collection<Lease> leases)
public void returnLease(Lease lease)
```

注意你可以一次性请求多个租约，如果Semaphore当前的租约不够，则请求线程会被阻塞。 同时还提供了超时的重载方法。

```java
public Lease acquire()
public Collection<Lease> acquire(int qty)
public Lease acquire(long time, TimeUnit unit)
public Collection<Lease> acquire(int qty, long time, TimeUnit unit)
```

Shared Semaphore使用的主要类包括下面几个：

- `InterProcessSemaphoreV2`
- `Lease`
- `SharedCountReader`

```java
public class InterProcessSemaphoreDemo {

	private static final int MAX_LEASE = 10;
	private static final String PATH = "/examples/locks";

	public static void main(String[] args) throws Exception {
		FakeLimitedResource resource = new FakeLimitedResource();
		try (TestingServer server = new TestingServer()) {

			CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.start();

			InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, PATH, MAX_LEASE);
			Collection<Lease> leases = semaphore.acquire(5);
			System.out.println("get " + leases.size() + " leases");
			Lease lease = semaphore.acquire();
			System.out.println("get another lease");

			resource.use();

			Collection<Lease> leases2 = semaphore.acquire(5, 10, TimeUnit.SECONDS);
			System.out.println("Should timeout and acquire return " + leases2);

			System.out.println("return one lease");
			semaphore.returnLease(lease);
			System.out.println("return another 5 leases");
			semaphore.returnAll(leases);
		}
	}
}
```

首先我们先获得了5个租约， 最后我们把它还给了semaphore。 接着请求了一个租约，因为semaphore还有5个租约，所以请求可以满足，返回一个租约，还剩4个租约。 然后再请求一个租约，因为租约不够，**阻塞到超时，还是没能满足，返回结果为null(租约不足会阻塞到超时，然后返回null，不会主动抛出异常；如果不设置超时时间，会一致阻塞)。**

上面说讲的锁都是公平锁(fair)。 总ZooKeeper的角度看， 每个客户端都按照请求的顺序获得锁，不存在非公平的抢占的情况。

#### 多共享锁对象 —Multi Shared Lock

Multi Shared Lock是一个锁的容器。 当调用`acquire()`， 所有的锁都会被`acquire()`，如果请求失败，所有的锁都会被release。 同样调用release时所有的锁都被release(**失败被忽略**)。 基本上，它就是组锁的代表，在它上面的请求释放操作都会传递给它包含的所有的锁。

主要涉及两个类：

- `InterProcessMultiLock`
- `InterProcessLock`

它的构造函数需要包含的锁的集合，或者一组ZooKeeper的path。

```java
public InterProcessMultiLock(List<InterProcessLock> locks)
public InterProcessMultiLock(CuratorFramework client, List<String> paths)
```

用法和Shared Lock相同。

```java
public class MultiSharedLockDemo {

	private static final String PATH1 = "/examples/locks1";
	private static final String PATH2 = "/examples/locks2";

	public static void main(String[] args) throws Exception {
		FakeLimitedResource resource = new FakeLimitedResource();
		try (TestingServer server = new TestingServer()) {
			CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.start();

			InterProcessLock lock1 = new InterProcessMutex(client, PATH1);
			InterProcessLock lock2 = new InterProcessSemaphoreMutex(client, PATH2);

			InterProcessMultiLock lock = new InterProcessMultiLock(Arrays.asList(lock1, lock2));

			if (!lock.acquire(10, TimeUnit.SECONDS)) {
				throw new IllegalStateException("could not acquire the lock");
			}
			System.out.println("has got all lock");

			System.out.println("has got lock1: " + lock1.isAcquiredInThisProcess());
			System.out.println("has got lock2: " + lock2.isAcquiredInThisProcess());

			try {
				resource.use(); //access resource exclusively
			} finally {
				System.out.println("releasing the lock");
				lock.release(); // always release the lock in a finally block
			}
			System.out.println("has got lock1: " + lock1.isAcquiredInThisProcess());
			System.out.println("has got lock2: " + lock2.isAcquiredInThisProcess());
		}
	}
}
```

新建一个`InterProcessMultiLock`， 包含一个重入锁和一个非重入锁。 调用`acquire()`后可以看到线程同时拥有了这两个锁。 调用`release()`看到这两个锁都被释放了。

**最后再重申一次， 强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，锁将会丢失。**

### 分布式计数器

顾名思义，计数器是用来计数的, 利用ZooKeeper可以实现一个集群共享的计数器。 只要使用相同的path就可以得到最新的计数器值， 这是由ZooKeeper的一致性保证的。Curator有两个计数器， 一个是用int来计数(`SharedCount`)，一个用long来计数(`DistributedAtomicLong`)。

#### 分布式int计数器—SharedCount

这个类使用int类型来计数。 主要涉及三个类。

- SharedCount
- SharedCountReader
- SharedCountListener

`SharedCount`代表计数器， 可以为它增加一个`SharedCountListener`，当计数器改变时此Listener可以监听到改变的事件，而`SharedCountReader`可以读取到最新的值， 包括字面值和带版本信息的值VersionedValue。

```java
public class SharedCounterDemo implements SharedCountListener {

	private static final int QTY = 5;
	private static final String PATH = "/examples/counter";

	public static void main(String[] args) throws IOException, Exception {
		final Random rand = new Random();
		SharedCounterDemo example = new SharedCounterDemo();
		try (TestingServer server = new TestingServer()) {
			CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.start();

			SharedCount baseCount = new SharedCount(client, PATH, 0);
			baseCount.addListener(example);
			baseCount.start();

			List<SharedCount> examples = Lists.newArrayList();
			ExecutorService service = Executors.newFixedThreadPool(QTY);
			for (int i = 0; i < QTY; ++i) {
				final SharedCount count = new SharedCount(client, PATH, 0);
				examples.add(count);
				Callable<Void> task = () -> {
					count.start();
					Thread.sleep(rand.nextInt(10000));
					System.out.println("Increment:" + count.trySetCount(count.getVersionedValue(), count.getCount() + rand.nextInt(10)));
					return null;
				};
				service.submit(task);
			}

			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);

			for (int i = 0; i < QTY; ++i) {
				examples.get(i).close();
			}
			baseCount.close();
		}
		Thread.sleep(Integer.MAX_VALUE);
	}

	@Override
	public void stateChanged(CuratorFramework arg0, ConnectionState arg1) {
		System.out.println("State changed: " + arg1.toString());
	}

	@Override
	public void countHasChanged(SharedCountReader sharedCount, int newCount) throws Exception {
		System.out.println("Counter's value is changed to " + newCount);
	}
}
```

在这个例子中，我们使用`baseCount`来监听计数值(`addListener`方法来添加SharedCountListener )。 任意的SharedCount， 只要使用相同的path，都可以得到这个计数值。 然后我们使用5个线程为计数值增加一个10以内的随机数。相同的path的SharedCount对计数值进行更改，将会回调给`baseCount`的SharedCountListener。

```java
count.trySetCount(count.getVersionedValue(), count.getCount() + rand.nextInt(10))
```

这里我们使用`trySetCount`去设置计数器。 **第一个参数提供当前的VersionedValue,如果期间其它client更新了此计数值， 你的更新可能不成功， 但是这时你的client更新了最新的值，所以失败了你可以尝试再更新一次。 而setCount是强制更新计数器的值**。

注意计数器必须`start`,使用完之后必须调用`close`关闭它。

强烈推荐使用`ConnectionStateListener`。 在本例中`SharedCountListener`扩展`ConnectionStateListener`。

#### 分布式long计数器—DistributedAtomicLong

再看一个Long类型的计数器。 除了计数的范围比`SharedCount`大了之外， 它首先尝试使用乐观锁的方式设置计数器， 如果不成功(比如期间计数器已经被其它client更新了)， 它使用`InterProcessMutex`方式来更新计数值。

可以从它的内部实现`DistributedAtomicValue.trySet()`中看出：

```java
AtomicValue<byte[]>   trySet(MakeValue makeValue) throws Exception
 {
     MutableAtomicValue<byte[]>  result = new MutableAtomicValue<byte[]>(null, null, false);

     tryOptimistic(result, makeValue);
     if ( !result.succeeded() && (mutex != null) )
     {
         tryWithMutex(result, makeValue);
     }

     return result;
 }
```

此计数器有一系列的操作：

- get(): 获取当前值
- increment()： 加一
- decrement(): 减一
- add()： 增加特定的值
- subtract(): 减去特定的值
- trySet(): 尝试设置计数值
- forceSet(): 强制设置计数值

你**必须**检查返回结果的`succeeded()`， 它代表此操作是否成功。 如果操作成功， `preValue()`代表操作前的值， `postValue()`代表操作后的值。

```java
public class DistributedAtomicLongDemo {

	private static final int QTY = 5;
	private static final String PATH = "/examples/counter";

	public static void main(String[] args) throws IOException, Exception {
		List<DistributedAtomicLong> examples = Lists.newArrayList();
		try (TestingServer server = new TestingServer()) {
			CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.start();
			ExecutorService service = Executors.newFixedThreadPool(QTY);
			for (int i = 0; i < QTY; ++i) {
				final DistributedAtomicLong count = new DistributedAtomicLong(client, PATH, new RetryNTimes(10, 10));

				examples.add(count);
				Callable<Void> task = () -> {
					try {
						AtomicValue<Long> value = count.increment();
						System.out.println("succeed: " + value.succeeded());
						if (value.succeeded())
							System.out.println("Increment: from " + value.preValue() + " to " + value.postValue());
					} catch (Exception e) {
						e.printStackTrace();
					}
					return null;
				};
				service.submit(task);
			}

			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);
			Thread.sleep(Integer.MAX_VALUE);
		}
	}
}
```

### 分布式队列

使用Curator也可以简化Ephemeral Node (**临时节点**)的操作。Curator也提供ZK Recipe的分布式队列实现。 利用ZK的 PERSISTENTS_EQUENTIAL节点， 可以保证放入到队列中的项目是按照顺序排队的。 如果单一的消费者从队列中取数据， 那么它是先入先出的，这也是队列的特点。 如果你严格要求顺序，你就的使用单一的消费者，可以使用Leader选举只让Leader作为唯一的消费者。

但是， 根据Netflix的Curator作者所说， ZooKeeper真心不适合做Queue，或者说ZK没有实现一个好的Queue，详细内容可以看 [Tech Note 4](https://cwiki.apache.org/confluence/display/CURATOR/TN4)， 原因有五：

1. ZK有1MB 的传输限制。 实践中ZNode必须相对较小，而队列包含成千上万的消息，非常的大。
2. 如果有很多节点，ZK启动时相当的慢。 而使用queue会导致好多ZNode. 你需要显著增大 initLimit 和 syncLimit.
3. ZNode很大的时候很难清理。Netflix不得不创建了一个专门的程序做这事。
4. 当很大量的包含成千上万的子节点的ZNode时， ZK的性能变得不好
5. ZK的数据库完全放在内存中。 大量的Queue意味着会占用很多的内存空间。

尽管如此， Curator还是创建了各种Queue的实现。 如果Queue的数据量不太多，数据量不太大的情况下，酌情考虑，还是可以使用的。

#### 分布式队列—DistributedQueue

DistributedQueue是最普通的一种队列。 它设计以下四个类：

- QueueBuilder - 创建队列使用QueueBuilder,它也是其它队列的创建类
- QueueConsumer - 队列中的消息消费者接口
- QueueSerializer - 队列消息序列化和反序列化接口，提供了对队列中的对象的序列化和反序列化
- DistributedQueue - 队列实现类

QueueConsumer是消费者，它可以接收队列的数据。处理队列中的数据的代码逻辑可以放在QueueConsumer.consumeMessage()中。

正常情况下先将消息从队列中移除，再交给消费者消费。但这是两个步骤，不是原子的。可以调用Builder的lockPath()消费者加锁，当消费者消费数据时持有锁，这样其它消费者不能消费此消息。如果消费失败或者进程死掉，消息可以交给其它进程。这会带来一点性能的损失。最好还是单消费者模式使用队列。

```java
public class DistributedQueueDemo {

	private static final String PATH = "/example/queue";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework clientA = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
		clientA.start();
		CuratorFramework clientB = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
		clientB.start();
		DistributedQueue<String> queueA;
		QueueBuilder<String> builderA = QueueBuilder.builder(clientA, createQueueConsumer("A"), createQueueSerializer(), PATH);
		queueA = builderA.buildQueue();
		queueA.start();

		DistributedQueue<String> queueB;
		QueueBuilder<String> builderB = QueueBuilder.builder(clientB, createQueueConsumer("B"), createQueueSerializer(), PATH);
		queueB = builderB.buildQueue();
		queueB.start();
		for (int i = 0; i < 100; i++) {
			queueA.put(" test-A-" + i);
			Thread.sleep(10);
			queueB.put(" test-B-" + i);
		}
		Thread.sleep(1000 * 10);// 等待消息消费完成
		queueB.close();
		queueA.close();
		clientB.close();
		clientA.close();
		System.out.println("OK!");
	}

	/**
	 * 队列消息序列化实现类
	 */
	private static QueueSerializer<String> createQueueSerializer() {
		return new QueueSerializer<String>() {
			@Override
			public byte[] serialize(String item) {
				return item.getBytes();
			}

			@Override
			public String deserialize(byte[] bytes) {
				return new String(bytes);
			}
		};
	}

	/**
	 * 定义队列消费者
	 */
	private static QueueConsumer<String> createQueueConsumer(final String name) {
		return new QueueConsumer<String>() {
			@Override
			public void stateChanged(CuratorFramework client, ConnectionState newState) {
				System.out.println("连接状态改变: " + newState.name());
			}

			@Override
			public void consumeMessage(String message) throws Exception {
				System.out.println("消费消息(" + name + "): " + message);
			}
		};
	}
}
```

例子中定义了两个分布式队列和两个消费者，因为PATH是相同的，会存在消费者抢占消费消息的情况。

#### 带Id的分布式队列—DistributedIdQueue

DistributedIdQueue和上面的队列类似，**但是可以为队列中的每一个元素设置一个ID**。 可以通过ID把队列中任意的元素移除。 它涉及几个类：

- QueueBuilder
- QueueConsumer
- QueueSerializer
- DistributedQueue

通过下面方法创建：

```java
builder.buildIdQueue()
```

放入元素时：

```java
queue.put(aMessage, messageId);
```

移除元素时：

```java
int numberRemoved = queue.remove(messageId);
```

在这个例子中， 有些元素还没有被消费者消费前就移除了，这样消费者不会收到删除的消息。

```java
public class DistributedIdQueueDemo {

	private static final String PATH = "/example/queue";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = null;
		DistributedIdQueue<String> queue = null;
		try {
			client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.getCuratorListenable().addListener((client1, event) -> System.out.println("CuratorEvent: " + event.getType().name()));

			client.start();
			QueueConsumer<String> consumer = createQueueConsumer();
			QueueBuilder<String> builder = QueueBuilder.builder(client, consumer, createQueueSerializer(), PATH);
			queue = builder.buildIdQueue();
			queue.start();

			for (int i = 0; i < 10; i++) {
				queue.put(" test-" + i, "Id" + i);
				Thread.sleep((long) (15 * Math.random()));
				queue.remove("Id" + i);
			}

			Thread.sleep(20000);

		} catch (Exception ex) {

		} finally {
			CloseableUtils.closeQuietly(queue);
			CloseableUtils.closeQuietly(client);
			CloseableUtils.closeQuietly(server);
		}
	}

	private static QueueSerializer<String> createQueueSerializer() {
		return new QueueSerializer<String>() {

			@Override
			public byte[] serialize(String item) {
				return item.getBytes();
			}

			@Override
			public String deserialize(byte[] bytes) {
				return new String(bytes);
			}

		};
	}

	private static QueueConsumer<String> createQueueConsumer() {

		return new QueueConsumer<String>() {

			@Override
			public void stateChanged(CuratorFramework client, ConnectionState newState) {
				System.out.println("connection new state: " + newState.name());
			}

			@Override
			public void consumeMessage(String message) throws Exception {
				System.out.println("consume one message: " + message);
			}

		};
	}
}
```

#### 优先级分布式队列—DistributedPriorityQueue

优先级队列对队列中的元素按照优先级进行排序。 **Priority越小， 元素越靠前， 越先被消费掉**。 它涉及下面几个类：

- QueueBuilder
- QueueConsumer
- QueueSerializer
- DistributedPriorityQueue

通过builder.buildPriorityQueue(minItemsBeforeRefresh)方法创建。 当优先级队列得到元素增删消息时，它会暂停处理当前的元素队列，然后刷新队列。minItemsBeforeRefresh指定刷新前当前活动的队列的最小数量。 主要设置你的程序可以容忍的不排序的最小值。

放入队列时需要指定优先级：

```
queue.put(aMessage, priority);
```

例子：

```java
public class DistributedPriorityQueueDemo {

	private static final String PATH = "/example/queue";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = null;
		DistributedPriorityQueue<String> queue = null;
		try {
			client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.getCuratorListenable().addListener((client1, event) -> System.out.println("CuratorEvent: " + event.getType().name()));

			client.start();
			QueueConsumer<String> consumer = createQueueConsumer();
			QueueBuilder<String> builder = QueueBuilder.builder(client, consumer, createQueueSerializer(), PATH);
			queue = builder.buildPriorityQueue(0);
			queue.start();

			for (int i = 0; i < 10; i++) {
				int priority = (int) (Math.random() * 100);
				System.out.println("test-" + i + " priority:" + priority);
				queue.put("test-" + i, priority);
				Thread.sleep((long) (50 * Math.random()));
			}

			Thread.sleep(20000);

		} catch (Exception ex) {

		} finally {
			CloseableUtils.closeQuietly(queue);
			CloseableUtils.closeQuietly(client);
			CloseableUtils.closeQuietly(server);
		}
	}

	private static QueueSerializer<String> createQueueSerializer() {
		return new QueueSerializer<String>() {

			@Override
			public byte[] serialize(String item) {
				return item.getBytes();
			}

			@Override
			public String deserialize(byte[] bytes) {
				return new String(bytes);
			}

		};
	}

	private static QueueConsumer<String> createQueueConsumer() {

		return new QueueConsumer<String>() {

			@Override
			public void stateChanged(CuratorFramework client, ConnectionState newState) {
				System.out.println("connection new state: " + newState.name());
			}

			@Override
			public void consumeMessage(String message) throws Exception {
				Thread.sleep(1000);
				System.out.println("consume one message: " + message);
			}

		};
	}

}
```

有时候你可能会有错觉，优先级设置并没有起效。那是因为优先级是对于队列积压的元素而言，如果消费速度过快有可能出现在后一个元素入队操作之前前一个元素已经被消费，这种情况下DistributedPriorityQueue会退化为DistributedQueue。

#### 分布式延迟队列—DistributedDelayQueue

JDK中也有DelayQueue，不知道你是否熟悉。 DistributedDelayQueue也提供了类似的功能， 元素有个delay值， 消费者隔一段时间才能收到元素。 涉及到下面四个类。

- QueueBuilder
- QueueConsumer
- QueueSerializer
- DistributedDelayQueue

通过下面的语句创建：

```java
QueueBuilder<MessageType> builder = QueueBuilder.builder(client, consumer, serializer, path);
... more builder method calls as needed ...
DistributedDelayQueue<MessageType> queue = builder.buildDelayQueue();
```

放入元素时可以指定`delayUntilEpoch`：

```java
queue.put(aMessage, delayUntilEpoch);
```

注意`delayUntilEpoch`不是离现在的一个时间间隔， 比如20毫秒，而是未来的一个时间戳，如 System.currentTimeMillis() + 10秒。 如果delayUntilEpoch的时间已经过去，消息会立刻被消费者接收。

```java
public class DistributedDelayQueueDemo {

	private static final String PATH = "/example/queue";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = null;
		DistributedDelayQueue<String> queue = null;
		try {
			client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.getCuratorListenable().addListener((client1, event) -> System.out.println("CuratorEvent: " + event.getType().name()));

			client.start();
			QueueConsumer<String> consumer = createQueueConsumer();
			QueueBuilder<String> builder = QueueBuilder.builder(client, consumer, createQueueSerializer(), PATH);
			queue = builder.buildDelayQueue();
			queue.start();

			for (int i = 0; i < 10; i++) {
				queue.put("test-" + i, System.currentTimeMillis() + 10000);
			}
			System.out.println(new Date().getTime() + ": already put all items");


			Thread.sleep(20000);

		} catch (Exception ex) {

		} finally {
			CloseableUtils.closeQuietly(queue);
			CloseableUtils.closeQuietly(client);
			CloseableUtils.closeQuietly(server);
		}
	}

	private static QueueSerializer<String> createQueueSerializer() {
		return new QueueSerializer<String>() {

			@Override
			public byte[] serialize(String item) {
				return item.getBytes();
			}

			@Override
			public String deserialize(byte[] bytes) {
				return new String(bytes);
			}

		};
	}

	private static QueueConsumer<String> createQueueConsumer() {

		return new QueueConsumer<String>() {

			@Override
			public void stateChanged(CuratorFramework client, ConnectionState newState) {
				System.out.println("connection new state: " + newState.name());
			}

			@Override
			public void consumeMessage(String message) throws Exception {
				System.out.println(new Date().getTime() + ": consume one message: " + message);
			}

		};
	}
}
```

#### SimpleDistributedQueue

前面虽然实现了各种队列，但是你注意到没有，这些队列并没有实现类似JDK一样的接口。 `SimpleDistributedQueue`提供了和JDK基本一致的接口(但是没有实现Queue接口)。 创建很简单：

```java
public SimpleDistributedQueue(CuratorFramework client,String path)
```

增加元素：

```java
public boolean offer(byte[] data) throws Exception
```

删除元素：

```java
public byte[] take() throws Exception
```

另外还提供了其它方法：

```java
public byte[] peek() throws Exception
public byte[] poll(long timeout, TimeUnit unit) throws Exception
public byte[] poll() throws Exception
public byte[] remove() throws Exception
public byte[] element() throws Exception
```

没有`add`方法， 多了`take`方法。

`take`方法在成功返回之前会被阻塞。 而`poll`方法在队列为空时直接返回null。

```java
public class SimpleDistributedQueueDemo {

	private static final String PATH = "/example/queue";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = null;
		SimpleDistributedQueue queue;
		try {
			client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.getCuratorListenable().addListener((client1, event) -> System.out.println("CuratorEvent: " + event.getType().name()));
			client.start();
			queue = new SimpleDistributedQueue(client, PATH);
			Producer producer = new Producer(queue);
			Consumer consumer = new Consumer(queue);
			new Thread(producer, "producer").start();
			new Thread(consumer, "consumer").start();
			Thread.sleep(20000);
		} catch (Exception ex) {

		} finally {
			CloseableUtils.closeQuietly(client);
			CloseableUtils.closeQuietly(server);
		}
	}

	public static class Producer implements Runnable {

		private SimpleDistributedQueue queue;

		public Producer(SimpleDistributedQueue queue) {
			this.queue = queue;
		}

		@Override
		public void run() {
			for (int i = 0; i < 100; i++) {
				try {
					boolean flag = queue.offer(("zjc-" + i).getBytes());
					if (flag) {
						System.out.println("发送一条消息成功：" + "zjc-" + i);
					} else {
						System.out.println("发送一条消息失败：" + "zjc-" + i);
					}
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		}
	}

	public static class Consumer implements Runnable {

		private SimpleDistributedQueue queue;

		public Consumer(SimpleDistributedQueue queue) {
			this.queue = queue;
		}

		@Override
		public void run() {
			try {
				byte[] datas = queue.take();
				System.out.println("消费一条消息成功：" + new String(datas, "UTF-8"));
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}


}
```

但是实际上发送了100条消息，消费完第一条之后，后面的消息无法消费，目前没找到原因。查看一下官方文档推荐的demo使用下面几个Api：

```java
Creating a SimpleDistributedQueue

public SimpleDistributedQueue(CuratorFramework client,
                              String path)
Parameters:
client - the client
path - path to store queue nodes
Add to the queue

public boolean offer(byte[] data)
             throws Exception
Inserts data into queue.
Parameters:
data - the data
Returns:
true if data was successfully added
Take from the queue

public byte[] take()
           throws Exception
Removes the head of the queue and returns it, blocks until it succeeds.
Returns:
The former head of the queue
NOTE: see the Javadoc for additional methods
```

但是实际使用发现还是存在消费阻塞问题。

**补充：**以上是原作者说明的消费阻塞问题，是由于采用`queue.take()`主动接受的，简单的解决方案，循环接受处理就可以了，例如：

```java
try {
    while(true){
    byte[] datas = queue.take();
    System.out.println(“消费一条消息成功：” + new String(datas, “UTF-8”));
}
} catch (Exception e) {
	e.printStackTrace();
}
```

### 分布式屏障—Barrier

分布式Barrier是这样一个类： 它会阻塞所有节点上的等待进程，直到某一个被满足， 然后所有的节点继续进行。

比如赛马比赛中， 等赛马陆续来到起跑线前。 一声令下，所有的赛马都飞奔而出。

#### DistributedBarrier

`DistributedBarrier`类实现了栅栏的功能。 它的构造函数如下：

```java
public DistributedBarrier(CuratorFramework client, String barrierPath)
Parameters:
client - client
barrierPath - path to use as the barrier
```

首先你需要设置栅栏，它将阻塞在它上面等待的线程:

```java
setBarrier();
```

然后需要阻塞的线程调用方法等待放行条件:

```java
public void waitOnBarrier()
```

当条件满足时，移除栅栏，所有等待的线程将继续执行：

```java
removeBarrier();
```

**异常处理** DistributedBarrier 会监控连接状态，当连接断掉时`waitOnBarrier()`方法会抛出异常。

```java
public class DistributedBarrierDemo {

	private static final int QTY = 5;
	private static final String PATH = "/examples/barrier";

	public static void main(String[] args) throws Exception {
		try (TestingServer server = new TestingServer()) {
			CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.start();
			ExecutorService service = Executors.newFixedThreadPool(QTY);
			DistributedBarrier controlBarrier = new DistributedBarrier(client, PATH);
			controlBarrier.setBarrier();

			for (int i = 0; i < QTY; ++i) {
				final DistributedBarrier barrier = new DistributedBarrier(client, PATH);
				final int index = i;
				Callable<Void> task = () -> {
					Thread.sleep((long) (3 * Math.random()));
					System.out.println("Client #" + index + " waits on Barrier");
					barrier.waitOnBarrier();
					System.out.println("Client #" + index + " begins");
					return null;
				};
				service.submit(task);
			}
			Thread.sleep(10000);
			System.out.println("all Barrier instances should wait the condition");
			controlBarrier.removeBarrier();
			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);

			Thread.sleep(20000);
		}
	}
}
```

这个例子创建了`controlBarrier`来设置栅栏和移除栅栏。 我们创建了5个线程，在此Barrier上等待。 最后移除栅栏后所有的线程才继续执行。

如果你开始不设置栅栏，所有的线程就不会阻塞住。

#### 双栅栏—DistributedDoubleBarrier

双栅栏允许客户端在计算的开始和结束时同步。当足够的进程加入到双栅栏时，进程开始计算， 当计算完成时，离开栅栏。 双栅栏类是`DistributedDoubleBarrier`。 构造函数为:

```java
public DistributedDoubleBarrier(CuratorFramework client,
                                String barrierPath,
                                int memberQty)
Creates the barrier abstraction. memberQty is the number of members in the barrier. When enter() is called, it blocks until
all members have entered. When leave() is called, it blocks until all members have left.

Parameters:
client - the client
barrierPath - path to use
memberQty - the number of members in the barrier
```

`memberQty`是成员数量，当`enter()`方法被调用时，成员被阻塞，直到所有的成员都调用了`enter()`。 当`leave()`方法被调用时，它也阻塞调用线程，直到所有的成员都调用了`leave()`。 就像百米赛跑比赛， 发令枪响， 所有的运动员开始跑，等所有的运动员跑过终点线，比赛才结束。

DistributedDoubleBarrier会监控连接状态，当连接断掉时`enter()`和`leave()`方法会抛出异常。

```java
public class DistributedDoubleBarrierDemo {

	private static final int QTY = 5;
	private static final String PATH = "/examples/barrier";

	public static void main(String[] args) throws Exception {
		try (TestingServer server = new TestingServer()) {
			CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
			client.start();
			ExecutorService service = Executors.newFixedThreadPool(QTY);
			for (int i = 0; i < QTY; ++i) {
				final DistributedDoubleBarrier barrier = new DistributedDoubleBarrier(client, PATH, QTY);
				final int index = i;
				Callable<Void> task = () -> {

					Thread.sleep((long) (3 * Math.random()));
					System.out.println("Client #" + index + " enters");
					barrier.enter();
					System.out.println("Client #" + index + " begins");
					Thread.sleep((long) (3000 * Math.random()));
					barrier.leave();
					System.out.println("Client #" + index + " left");
					return null;
				};
				service.submit(task);
			}

			service.shutdown();
			service.awaitTermination(10, TimeUnit.MINUTES);
			Thread.sleep(Integer.MAX_VALUE);
		}
	}

}
```

## 3.6 实现分布式锁

在获取分布式锁的时候在locker节点下创建临时顺序节点，释放锁的时候删除该临时节点。客户端调用createNode方法在locker下创建临时顺序节点，

然后调用getChildren(“locker”)来获取locker下面的所有子节点，注意此时不用设置任何Watcher。客户端获取到所有的子节点path之后，如果发现自己创建的节点在所有创建的子节点序号最小，那么就认为该客户端获取到了锁。

如果发现自己创建的节点并非locker所有子节点中最小的，说明自己还没有获取到锁，此时客户端需要找到比自己小的那个节点，然后对其调用exist()方法，同时对其注册事件监听器。之后，让这个被关注的节点删除，则客户端的Watcher会收到相应通知，此时再次判断自己创建的节点是否是locker子节点中序号最小的，如果是则获取到了锁，如果不是则重复以上步骤继续获取到比自己小的一个节点并注册监听。当前这个过程中还需要许多的逻辑判断。