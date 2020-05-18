# Java高频知识点

## 1.类初始化与实例初始化

### 1.1 类初始化

一个类要创建实例需要先加载并初始化该类。

PS：main方法所在的类需要先加载和初始化

子类初始化需要先初始化父类，一个类初始化就是执行<clinit>方法，该方法由**静态变量赋值代码和静态代码块组成，由上到下顺序执行。**

### 1.2 实例初始化过程

实例初始化就是执行<init>方法。该方法由**非静态变量赋值代，非静态代码块，对应构造器代码组成，其中前两者由上到下顺序执行，对应构造器代码最后执行**。同样，调用子类构造器前需要调用父类的构造器。

PS：如果子类重写了父类的方法，通过子类对象调用的一定是子类重写的方法。

## 2.方法的参数传递机制

形参是基本类型：传递数据值的副本

形参是引用类型：传递地址值的副本

## 3.循环和递归

```properties
有N阶台阶，每次只能走一步或两步，问走完台阶总共有多少走法
```

```java
//递归,精简，但是浪费空间，且递归过深容易堆栈溢出
public int f(int N){
    if(N==1||N==2)
    	return N;
    else return f（N-1）+f（N-2）
}
//循环迭代，尽管不够简洁，但是运行效率好
public int loop(int N){
    if(N==1||N==2)
    	return N;
    int one = 2;//初始化走到第二级台阶的走法
    int two = 1;//初始化走到第一级台阶的走法
    int sum = 0;
    for(int i=3;i<=N;i++){
        //最后跨两部+最后跨一步的走法
        sum = two+one；
 		two = one;
        one = sum;
    }
    
}
```

## 4.成员变量和局部变量

变量对应存在就近原则;

**非静态代码块的执行：每次创建实例对象都会执行**；

所有的对象实例和数组都在堆上分配；虚拟机栈存储局部变量表（基本类型，对象应用）；方法区存储已经被虚拟机加载的类信息、常量、静态变量。

## 5.git分支命令

创建分支：git branch <分支名>

切换分支：git checkout <分支名>

一步完成：git checkout -b <分支名>

合并分支：git checkout master; git merge  <分支名> 先切换到主干再合并

删除分支：git checkout master; git branch -D <分支名> 先切换到主干再删除

## 6.Volatile

是Java虚拟机提供的轻量级同步机制，主要作用：

- 保证可见性
- 不保证原子性
- 禁止指令重排

### 6.1 内存模型的可见性

JMM（Java Memory Model）是一种抽象概念，需要保证**原子性，可见性，有序性**。规定所有变量存储在主内存，主内存是共享内存区域，每个线程在创建时JVM都会为其创建一个工作内存，是线程的私有数据区域。**线程对变量的操作必须在工作内存中进行，将变量从主内存拷贝至工作内存，操作完毕后写回主内存。各工作内存存储主内存的变量副本拷贝。**

### 6.2 Volatile不保证原子性

可以利用**synchronized**或者对应原子类**AtomicXXX**

### 6.3 禁止指令重排

多线程环境下线程交替执行，由于编译器优化重排的存在，多个线程使用的变量能否保证一致性无法确定。

## 7.CAS

### 7.1 原理

CompareAndSwap  比较并交换，是一条**CPU并发原语，完全依赖于硬件，原语执行连续且不可中断，即CAS是一条CPU的原子指令。**

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;
    private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");

    private volatile int value;
```

Unsafe是CAS的核心类，而Unsafe中很多方法是由native修饰的，可以直接调用操作系统底层资源执行任务。

以如下方法为例，变量offset即该变量在内存中的偏移地址。VALUE又用volatile修饰，保证内存可见性 。循环即使自旋锁理念的体现。

```java
public final int getAndIncrement() {
        return U.getAndAddInt(this, VALUE, 1);
    }

  public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));
        return v;
    }
```

### **7.2 缺点**：

- 如果CAS失败，会一直进行尝试，可能会带来较大的CPU开销。

  在Java1.8中引入LongAdder类

  ```java
  public class LongAdder extends Striped64 implements Serializable {}
  abstract class Striped64 extends Number {
       transient volatile Cell[] cells;
      transient volatile long base;
  }
  ```

  在LongAdder的底层实现中，首先有一个base值，刚开始没有并发冲突，都是对base进行累加的，比如刚开始累加成了base = 5。

  接着如果发现并发更新的线程，就会开始施行**分段CAS的机制**，也就是内部会搞一个Cell数组，每个数组是一个数值分段。

  这时，让大量的线程分别去对不同Cell内部的value值进行CAS累加操作，这样就把CAS计算压力分散到了不同的Cell分段数值中了！

  而且他内部实现了**自动分段迁移的机制**，也就是如果某个Cell的value执行CAS失败了，那么就会自动去找另外一个Cell分段内的value值进行CAS操作。

- 只能保证一个共享变量的原子操作
- 造成ABA问题

### 7.3 原子引用

```java
AtomicReference<USer> atomicReference = new AtomicReference<>();
atomicReference.set();
```

更进一步，利用AtomicStampedReference可以解决ABA问题,它附带有一个可以原子更新的stamp值

```java
/**
 * An {@code AtomicStampedReference} maintains an object reference
 * along with an integer "stamp", that can be updated atomically.
 */
public class AtomicStampedReference<V> {
    
    public AtomicStampedReference(V initialRef, int initialStamp)
    
}
```

## 8 集合类并发不安全

### 8.1.List

并发修改时可能引发java.utill.ConcurrentModificationException异常。、

解决方法：

- ```java
  Collections.synchronizedList()
  ```

- ```java
  /**
  读写分离的思想，往容器中添加元素时，不直接往当前容器直接添加，而是复制当前容器，在新的容器中进行添加，完毕后将原容器的引用指向新的容器。可以并发读不需要加锁。
  **/
  CopyOnWriteArrayList 
  ```

### 8.2 Set

与List解决方法一致：

- ```java
  Collections.synchronizedSet()
  ```

- ```java
  CopyOnWriteArraySet
  ```

### 8.3 hashmap

在1.8中，**当链表长度>8且数组长度大于64时，才会将链表转化为红黑树，否则先对数组扩容**

### 8.4 用LinkedHashMap实现LRU

LinkedHashMap底层就是用的【**HashMap 加 双链表**】实现的，而且本身已经实现了按照访问顺序的存储。
此外，LinkedHashMap中本身就实现了一个方法removeEldestEntry用于判断是否需要移除最不常读取的数，
方法默认是直接返回false，不会移除元素，所以需要重写该方法,即当缓存满后就移除最不常用的数。

```java
public class LRU<K,V> {
	private LinkedHashMap<K, V> map;
	private int cacheSize;
	
	public LRU(int cacheSize)
	{
		this.cacheSize = cacheSize;
		map = new LinkedHashMap<K,V>(16,0.75F,true){
			
			@Override
			protected boolean removeEldestEntry(Entry eldest) {
				if(cacheSize + 1 == map.size()){
					return true;
				}else{
					return false;
				}
			}
		};  //end map
	}
	
	public synchronized V get(K key) {
		return map.get(key);
	}
	public synchronized void put(K key,V value) {
		map.put(key, value);
	}
	public synchronized void clear() {
		map.clear();
	}
	public synchronized int usedSize() {
		return map.size();
	}
	public void print() {
		for (Map.Entry<K, V> entry : map.entrySet()) {
			System.out.print(entry.getValue() + "--");
		}
		System.out.println();
	}
	
	public static void main(String[] args) {
		LRU<String, Integer> LRUMap = new LRU<>(3);
		LRUMap.put("key1", 1);
		LRUMap.put("key2", 2);
		LRUMap.put("key3", 3);
		LRUMap.print();
		System.out.println();
		LRUMap.put("key4", 4);
		LRUMap.print();
	}
}
```

主要是构造函数

```
public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder)
```

排序模式accessOrder，该属性为boolean型变量，对于访问顺序，为true；对于插入顺序，则为false。

故，基于LRU思想，构造一个LinkedHashMap，并打算按从近期访问最少到近期访问最多的顺序（即访问顺序）来保存元素，设置accessOrder为true即可，默认为false。

同时，LinkedHashMap提供了`removeEldestEntry(Map.Entry<K,V> eldest)`方法，在将新条目插入到映射后，put和 putAll将调用此方法。该方法可以提供在每次添加新条目时移除最旧条目的实现程序，默认返回false，

即永远不能移除最旧的元素。构造LRU时，根据条件设置返回true即可。

```java
private static final int MAX_ENTRIES = 100;  
protected boolean removeEldestEntry(Map.Entry eldest) {  
    return size() > MAX_ENTRIES;  
}  
```

### 8.5 不使用LinkedHashMap实现LRU

`ConcurrentHashMap` + `ConcurrentLinkedQueue` +`ReadWriteLock`

`ConcurrentHashMap` 是线程安全的Map，我们可以利用它缓存 key,value形式的数据。`ConcurrentLinkedQueue`是一个线程安全的基于链表的队列（先进先出），我们可以用它来维护 key 。每当我们put/get(缓存被使用)元素的时候，我们就将这个元素对应的 key 存放在队列尾部，这样就能保证队列头部的元素是最近最少使用的。当我们的缓存容量不够的时候，我们直接移除队列头部对应的key以及这个key对应的缓存即可！

另外，我们用到了`ReadWriteLock`(读写锁)来保证线程安全。

```java
public class MyLruCache<K, V> {

    /**
     * 缓存的最大容量
     */
    private final int maxCapacity;

    private ConcurrentHashMap<K, V> cacheMap;
    private ConcurrentLinkedQueue<K> keys;
    /**
     * 读写锁
     */
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private Lock writeLock = readWriteLock.writeLock();
    private Lock readLock = readWriteLock.readLock();

    public MyLruCache(int maxCapacity) {
        if (maxCapacity < 0) {
            throw new IllegalArgumentException("Illegal max capacity: " + maxCapacity);
        }
        this.maxCapacity = maxCapacity;
        cacheMap = new ConcurrentHashMap<>(maxCapacity);
        keys = new ConcurrentLinkedQueue<>();
    }

    public V put(K key, V value) {
        // 加写锁
        writeLock.lock();
        try {
            //1.key是否存在于当前缓存
            if (cacheMap.containsKey(key)) {
                moveToTailOfQueue(key);
                cacheMap.put(key, value);
                return value;
            }
            //2.是否超出缓存容量，超出的话就移除队列头部的元素以及其对应的缓存
            if (cacheMap.size() == maxCapacity) {
                System.out.println("maxCapacity of cache reached");
                removeOldestKey();
            }
            //3.key不存在于当前缓存。将key添加到队列的尾部并且缓存key及其对应的元素
            keys.add(key);
            cacheMap.put(key, value);
            return value;
        } finally {
            writeLock.unlock();
        }
    }

    public V get(K key) {
        //加读锁
        readLock.lock();
        try {
            //key是否存在于当前缓存
            if (cacheMap.containsKey(key)) {
                // 存在的话就将key移动到队列的尾部
                moveToTailOfQueue(key);
                return cacheMap.get(key);
            }
            //不存在于当前缓存中就返回Null
            return null;
        } finally {
            readLock.unlock();
        }
    }

    public V remove(K key) {
        writeLock.lock();
        try {
            //key是否存在于当前缓存
            if (cacheMap.containsKey(key)) {
                // 存在移除队列和Map中对应的Key
                keys.remove(key);
                return cacheMap.remove(key);
            }
            //不存在于当前缓存中就返回Null
            return null;
        } finally {
            writeLock.unlock();
        }
    }

    /**
     * 将元素添加到队列的尾部(put/get的时候执行)
     */
    private void moveToTailOfQueue(K key) {
        keys.remove(key);
        keys.add(key);
    }

    /**
     * 移除队列头部的元素以及其对应的缓存 (缓存容量已满的时候执行)
     */
    private void removeOldestKey() {
        K oldestKey = keys.poll();
        if (oldestKey != null) {
            cacheMap.remove(oldestKey);
        }
    }

    public int size() {
        return cacheMap.size();
    }

}
```

## 9.Java锁机制

### 9.1 概念补充

可重入锁  避免死锁，指的是外层函数获得锁之后，在进入内层函数可以自动获取该锁。

PS：使用Lock对象时，需要注意加锁几次就必须要解锁几次，多次加解锁无影响。

### 9.2 CountDownLatch,CyclicBarrier,Semaphore

-  CountDownLatch

```java
public class CountDownLatchDemo {
	/**
	 * 类java.util.concurrent.CountDownLatch是一个同步辅助类，当多个（具体数量等于初始化CountDownLatch时count参数的值）线程
	 * 都达到了预期状态或完成预期工作时触发事件，其他线程可以等待这个事件来触发自己的后续工作
	 * 使用指定的数字初始化CountDownLatch，一个线程调用 await()方法后，在当前计数到达0之前，会一直受阻塞，
	 * 其他线程 调用 countDown() 方法，会使计数器递减，所以，计数器的值为 0 后，会释放所有等待的线程，即实现的是多个工作线程完成任务后通知多个等待线程开始工作
	 * 调用countdown()计数减1，调用await()只进行阻塞。不可重复利用
	 * CyclicBarrier则不同，计数达到指定值时释放等待线程，调用await()方法计数加1，如果不等于构造方法的值，则线程阻塞
	 * 线程在countDown()之后，会继续执行自己的任务，而CyclicBarrier则会等待所有线程任务结束之后再进行后续任务
	 * CountDownLatch不能循环使用，计数器减为0就减为0了，不能被重置；CyclicBarrier提供了reset()方法，支持循环使用
	 */
	public static void main(String[] args) throws InterruptedException{
		CountDownLatch countDownLatch = new CountDownLatch(5);
		for (int i = 0; i < 5; i++) {
			new Thread(new readNum(i, countDownLatch)).start();
		}
		countDownLatch.await();
		System.out.println("所有线程组结束");
		
	}
	static class readNum implements Runnable{
		private int id;
		private CountDownLatch latch;
		
		public readNum(int id, CountDownLatch latch) {
			this.id = id;
			this.latch = latch;
		}
		@Override
		public void run() {
			synchronized (this) {
				System.out.println("id"+id);
				latch.countDown();
				System.out.println("线程组任务结束"+id+",其他任务继续");
			}
		}
	}
}
```

- CyclicBarrier

  ```java
  public class BarrierDemo {
  	/**
  	 * 实际应用中，有时需要多个线程同时工作来完成一件事情，而且在完成过程中，往往会需要所有线程到达某一阶段后再统一执行
  	 * java.utill.concurrrent.CyclicBarrier就允许一组线程互相等待，直到到达某个公共屏障点（common barrier point）
  	 */
  	// 徒步
  	private static int[] time4Walk={5,8,10};
  	//自驾游
  	private static int[] time4Self={2,1,4};
  	//巴士
  	private static int[] time4Bus={1,2,4};
  	static String nowTime(){
  		SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
  		return sdf.format(new Date())+":";
  	}
  	static class Tour implements Runnable{
  		private int[] time4Use;
  		private CyclicBarrier barrier;
  		private String tourName;
  		public Tour(int[] time4Use, CyclicBarrier barrier, String tourName) {
  			this.time4Use = time4Use;
  			this.barrier = barrier;
  			this.tourName = tourName;
  		}
  		@Override
  		public void run() {
  			try {
  				Thread.sleep(time4Use[0]*1000);
  				System.out.println(nowTime()+tourName+" 到达深圳");
  				barrier.await();
  				Thread.sleep(time4Use[1]*1000);
  				System.out.println(nowTime()+tourName+" 到达广州");
  				barrier.await();
  				Thread.sleep(time4Use[2]*1000);
  				System.out.println(nowTime()+tourName+" 到达重庆");
  				barrier.await();
  			} catch (InterruptedException|BrokenBarrierException e) {
  				// TODO Auto-generated catch block
  				e.printStackTrace();
  			}
  		}
  	}
  	public static void main(String[] args) {
  		//三个旅行团都到达后，执行以下操作
  		Runnable runner = new Runnable() {
  			@Override
  			public void run() {
  				System.out.println("已到达");
  			}
  		};
  		CyclicBarrier barrier = new CyclicBarrier(3, runner);
  		ExecutorService exec = Executors.newFixedThreadPool(3);
  		exec.submit(new Tour(time4Walk, barrier, "步行"));
  		exec.submit(new Tour(time4Self, barrier, "自驾游"));
  		exec.submit(new Tour(time4Bus, barrier, "巴士"));
  		exec.shutdown();
  	}
  
  }
  
  ```

- Future

  ```java
  public class FutureDemo {
  
  	/**
  	 * 接口public interface Future<V>表示异步计算的结果，计算完成后用get（）获取结果，
  	 * cancel()取消任务执行，如果任务已启动或已取消则尝试失败。如果调用成功，且此时任务未启动，则此任务永不执行
  	 * 在主线程中需要执行比较耗时的操作，但又不想阻塞主线程，便可以把这些作业交给Future完成
  	 */
  	public static void main(String[] args) {
  		Callable<Object> callable = new PrivateAccount();
  		FutureTask<Object> futureTask = new FutureTask<>(callable);
  		Thread thread = new Thread(futureTask);
  		System.out.println("future task开始于"+System.nanoTime());
  		thread.start();
  		System.out.println("主线程执行自己的任务");
  		int totalMoney = 1000;
  		System.out.println("其他账户拥有"+totalMoney);
  		System.out.println("后台获取私人账户数据");
  		//测试后台计算是否完成，如果没有则等待
  		while (!futureTask.isDone()) {
  			try {
  				Thread.sleep(5);
  			} catch (InterruptedException e) {
  				e.printStackTrace();
  			}
  		}
  		System.out.println("future task结束于"+System.nanoTime());
  		Integer privateMoney = 0;
  		try {
              //获取结果，如果没有计算完成会阻塞，故建议最后调用
  			privateMoney = (Integer)futureTask.get();
  		} catch (InterruptedException|ExecutionException e) {
  			e.printStackTrace();
  		} 
  		System.out.println("所有账户共拥有"+(totalMoney+privateMoney));
  	}
  }
  
  class PrivateAccount implements Callable<Object>{
  	Integer money;
  	public Integer call() throws Exception {
  		// TODO Auto-generated method stub
  		Thread.sleep(5000);
  		money = new Integer(new Random().nextInt(10000));
  		System.out.println("私人账户拥有"+money);
  		return money;
  	}
  }
  ```

- Semaphore

  ```java
  public class PoolSemaphoreDemo {
  
  	/**
  	 * 类Semaphore提供了一个计数信号量，信号量维护了一个许可集，如有必要，在许可可用前会阻塞每一个acquire(),然后再获取该许可，每个release()添加一个许可，从而可能释放一个正在阻塞的获取者。通常用于限制某些资源的线程数目。
  	 * Semaphore有两种模式，公平模式与非公平模式，公平模式遵循FIFO，非公平模式则是抢占式的。
  	 * 构造方法Semaphore(int permits, boolean fair)可以指定模式
  	 */
  	private static final int MAX_AVAILABLE = 5;
  	private final Semaphore available = new Semaphore(MAX_AVAILABLE); 
  	public static void main(String[] args) {
  		final PoolSemaphoreDemo pool = new PoolSemaphoreDemo();
  		Runnable runner = new Runnable() {
  			@Override
  			public void run() {
  				try {
  					Object o;
  					o = pool.getItem();
  					System.out.println(Thread.currentThread().getName()+"——acquire——"+o);
  					Thread.sleep(100);
  					pool.putItem(o);
  					System.out.println(Thread.currentThread().getName()+"——release——"+o);
  				} catch (InterruptedException e) {
  					e.printStackTrace();
  				}
  				
  			}
  		};
  		for (int i = 0; i < 10; i++) {
  			Thread thread = new Thread(runner);
  			thread.start();
  		}
  	}
  	/**
  	 * 获取数据，需要得到许可。公平模式下，会首先判断当前队列中有没有线程等待，如果有就等待。
  	 * 而非公平模式会首先试一把，说不定就可以获得许可，可以插队
  	 */
  	public Object getItem() throws InterruptedException{
  		available.acquire();
  		return getNextAvailableItem();
  	}
  	//放回数据，释放许可
  	public void putItem(Object x){
  		if (markAsUnused(x))
  			available.release();
  	}
  	protected Object[] items={"a","b","c","d","e"}; 
  	protected boolean[] uesd = new boolean[MAX_AVAILABLE];
  	protected synchronized Object getNextAvailableItem(){
  		for (int i = 0; i < MAX_AVAILABLE; i++) {
  			if (!uesd[i]) {
  				uesd[i]=true;
  				return items[i];
  			}
  		}
  		return null;
  	}
  	protected synchronized boolean markAsUnused(Object item){
  		for (int i = 0; i < MAX_AVAILABLE; i++) {
  			if (item==items[i]) {
  				if (uesd[i]) {
  					uesd[i]=false;
  					return true;
  				}else {
  					return false;
  				}
  					
  			}
  		}
  		return false;
  	}
  }
  ```

### 9.3 阻塞队列

利用阻塞队列，不用人为关心线程何时阻塞何时唤醒，具体由BlockingQueue解决。

```java
ArrayBlockingQueue ：一个由数组支持的有界队列，必须指定队列大小。FIFO原则排序元素

LinkedBlockingQueue ：一个由链表支持的可选有界队列，可以不必指定大小。因为不指定大小时，默认指定大小是Integer.MAX_VALUE，如果是数组会造成大量空间浪费，链表则不会。同时，ArrayBlockingQueue只有一个ReentrantLock对象，这意味着生产者和消费者无法并行运行，而LinkedBlockingQueue的生产者和消费者都有自己的锁，可以并行处理。

PriorityBlockingQueue ：一个由优先级堆支持的无界优先级队列

DelayQueue ：一个由优先级堆支持的、基于时间的调度队列。加入到队列的元素必须实现Delayed接口。因为队列的大小没有界限，使得添加可以立刻返回，但是在延迟时间过去之前，不能从队列中取出元素。如果多个元素完成了延迟，那么最早失效/失效时间最长的元素将第一个取出。

SynchronousQueue ：一个利用 BlockingQueue 接口的简单聚集（rendezvous)机制，没有缓冲的等待队列，生产者直接把数据给消费者，也即单个元素的队列。即插入操作必须等待调用移除操作。

LinkedTransferQueue:链表结构支持的无界阻塞队列

LinkedBlockingDeque:链表结构支持的双向阻塞队列
```

| 方法类型 | 抛出异常                  | 特殊值             | 阻塞          | 超时                                            |
| -------- | ------------------------- | ------------------ | ------------- | ----------------------------------------------- |
| 插入     | boolean add(E e)          | boolean offer(E e) | void put(E e) | boolean offer(E e, long timeout, TimeUnit unit) |
| 移除     | boolean remove(Object o); | E poll()           | E take()      | E poll(long timeout, TimeUnit unit)             |
| 检查     | E element()               | E peek()           | 不可用        | E poll()                                        |

| 抛出异常 | 阻塞队列满时，再往队列插入元素会抛出IllegalStateException；阻塞队列空时，再从队列移除元素会抛出NoSuchElementException |
| -------- | ------------------------------------------------------------ |
| 特殊值   | 插入成功true失败false;移除成功，返回移除的元素，没有就返回null |
| 阻塞     | 队列满时，队列会阻塞生产者线程直至put成功或响应中断退出；队列空时，对于消费者亦然 |
| 超时     | 当队列满时，队列会阻塞生产者线程一段时间，超时则生产者推出   |

**PS：多线程判断基本利用while循环**

## 10.线程池

### 10.1 基本概念

主要特点：线程复用，降低资源损耗，提高反应速度；控制最大并发数，提高线程的可管理性。

```java
/**
		 * Executors中创建线程池静态方法
		 * 线程池ThreadPoolExecutor
		 * public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
		 * corePoolSize时，就会把多余的任务存储在  核心池的大小,默认情况下，线程池中没有任何线程，而是等待有任务到来才创建线程去执行任务。核心线程数达到corePoolSize时，就会把多余的任务存储在workQueue中。
		 * maximumPoolSize   池中允许的最大线程数,wordQueue满了，线程池中最多可以创建的线程数量
		 * keepAliveTime时，多余的空闲线程的存活时间。当线程池中的线程数大于corePoolSize时，空闲时间达到keepAliveTime时，多于空闲线程就会被销毁
		 * unit keepAliveTime单位
		 * workQueue	 存储还没来得及执行的任务
		 * threadFactory 执行程序创建新线程时使用的工厂
		 * handler  由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序
		 
		 Executors.newFixedThreadPool( int n) 指定大小的线程池
		 Executors.newSingleThreadExecutor() 一个线程的线程池
		 Executors.newCachedThreadPool() 一池若干个线程，根据实际情况适配
		 */
		ExecutorService executorService = Executors.newFixedThreadPool(2);
		for (int i = 0; i < 10; i++) {
			Runnable runner = new Runner(i);
			executorService.execute(runner);
		}
		executorService.shutdown();
		//使用awaitTermination() 方法的前提需要关闭线程池.主线程等待线程池中所有任务执行完毕:
		executorService.awaitTermination(1, TimeUnit.MINUTES);
```

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
//能实现定时、周期性任务的线程池。
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }

```

### 10.2 工作原理

![线程池工作原理](D:\Program Files\笔记\image\线程池工作原理.JPG)

1）当提交一个新任务到线程池时，线程池判断corePoolSize线程池是否都在执行任务，如果有空闲线程，则创建一个新的工作线程来执行任务，直到当前线程数等于corePoolSize；

2）如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；

3）如果阻塞队列满了，那就创建新的线程执行当前任务，直到线程池中的线程数达到maxPoolSize，这时再有任务来，由饱和策略来处理提交的任务

### 10.3 饱和策略

AbortPolicy（默认）：不处理，直接抛出异常。
CallerRunsPolicy：若线程池还没关闭，调用当前所在线程来运行任务，r.run()执行。
DiscardOldestPolicy：LRU策略，丢弃队列里最近最久不使用的一个任务，并把当前任务加入队列尝试再次提交。
DiscardPolicy：直接丢弃掉，不处理，不抛出异常。

### 10.4 线程池的选择

生产中使用ThreadPoolExecutor自定义。使用Executors的弊端如下：

- FixedThreadPool和SingleThread允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量请求，OOM
- CachedThreadPool和ScheduledThreadPool允许的创建线程数量为Integer.MAX_VALUE，可能会创建大量线程，OOM

### 10.5 线程池数量的选择

```java
Runtime.getRuntime().availableProcessors();//获取CPU核数
```

CPU密集：需要大量运算，没有阻塞。配置尽可能少的线程数量，一般配置CPU核数+1的线程池

IO密集：需要大量IO，即大量阻塞，使CPU的运算能力浪费在等待切换上。CPU并不是一直在执行任务，可配置尽可能多的线程，如CPU核数*2

## 11.JVM

### 11.1 GC Roots

为了解决引用计数法的循环引用问题，java使用了可达性分析的方法。

所谓**GC ROOT**或者说**Tracing GC**的“根集合”就是一组比较活跃的引用。

基本思路就是**通过一系列“GC Roots”的对象作为起始点**，从这个被称为GC Roots 的对象开始向下搜索，如果一个对象到GC Roots没有任何引用链相连时，则说明此对象不可用。也即**给定一个集合的引用作为根出发**，通过引用关系遍历对象图，能被便利到的对象就被判定为存活；没有被便利到的就被判定为死亡

**哪些对象可以作为GC Roots对象:**

- 虚拟机栈（栈帧中的局部变量区，也叫局部变量表）中应用的对象。
- 方法区中的类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI（native方法）引用的对象

### 11.2 参数

#### 1）如何查看运行中程序的JVM信息

jps  查看进程信息

jinfo -flag  配置项 进程号（ 查看该进程是否配置该配置项）

jinfo -flags  进程号 ( 查看所有配置)

#### 2）查看参数

- -XX:+PrintFlagsInitial

  查看初始化的参数

  e.g: java -XX:+PrintFlagsInitial -version

- -XX:+PrintFlagsFinal

  主要查看修改更新后的 `:=`说明是修改过的

  ![JVM参数查看](D:\Program Files\笔记\image\JVM参数查看.JPG)

- -XX:+PrintCommandLineFlags

  查看使用的垃圾回收器

#### 3) 常用参数配置

1. -Xms

   初始大小内存，默认为物理内存1/64，等价于-XX:InitialHeapSize

2. -Xmx

   最大分配内存，默认物理内存1/4，等价于-XX:MaxHeapSize

3. -Xss

   设置单个线程栈的大小，依赖于平台，默认512K~1024K ，等价于-XX:ThreadStackSize

4. -Xmn

   设置年轻代的大小

5. -XX:MetaspaceSize

   设置元空间大小

   元空间的本质和永久代类似，都是对JVM规范中方法区的实现，不过元空间与永久代最大的区别在于：**元空间并不在虚拟机中，而是在本地内存中。因此，默认元空间的大小仅受本地内存限制**.但默认大小可能不满足需要，根据实际情况调整。

6. -XX:+PrintGCDetails

   输出详细GC收集日志信息

   [名称：GC前内存占用->GC后内存占用(该区内存总大小)]

7. -XX:SurvivorRatio

   设置新生代中Eden和S0/S1空间的比例

   默认-XX:SurvivorRatio=8,Eden:S0:S1=8:1:1

8. -XX:NewRatio

   设置年轻代与老年代在堆结构的占比

   默认-XX:NewRatio=2 新生代在1，老年代2，年轻代占整个堆的1/3

   NewRatio值几句诗设置老年代的占比，剩下的1给新生代

9. -XX:MaxTenuringThreshold

   设置垃圾的最大年龄

   默认-XX:MaxTenuringThreshold=15

   如果设置为0，年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大的值，则年轻代对象回在Survivor区进行多次复制，这样可以增加对对象在年轻代的存活时间，增加在年轻代即被回收的概率。

10. -XX:+UseSerialGC

    串行垃圾回收器

11. -XX:+UseParallelGC

    并行垃圾回收器

### 11.3 OOM

- `java.lang.StackOverflowError`

  栈空间溢出 ，递归调用卡死

- `java.lang.OutOfMemoryError:Java heap space`

  堆内存溢出 ， 对象过大

- `java.lang.OutOfMemoryError:GC overhead limit exceeded`

  GC回收时间过长

  过长的定义是超过98%的时间用来做GC并且回收了而不倒2%的堆内存

  连续多次GC，都回收了不到2%的极端情况下才会抛出

  如果不抛出，那就是GC清理的一点内存很快会被再次填满，迫使GC再次执行，这样就恶性循环，

  cpu使用率一直是100%，二GC却没有任何成果

- java.lang.OutOfMemoryError:unable to create new native thread

  应用创建了太多线程，一个应用进程创建了多个线程，超过系统承载极限

  你的服务器并不允许你的应用程序创建这么多线程，linux系统默认允许单个进程可以创建的线程数是1024，超过这个数量，就会报错

  解决办法：

  降低应用程序线程的数量，分析应用给是否针对需要这么多线程，如果不是，减到最低

  修改linux服务器配置

- ```java
  java.lang.OutOfMemoryError:Metaspace
  ```

  元空间主要存放了虚拟机加载的类的信息、常量池、静态变量、即时编译后的代码

## 12 单例模式

- 懒汉式双重检查锁

```java
public class SingleTon3 {

         private SingleTon3(){};             //私有化构造方法

         private static volatile SingleTon3 singleTon=null;

         public static SingleTon3 getInstance(){
                  //第一次校验
                 if(singleTon==null){     
                synchronized(SingleTon3.class){
                           //第二次校验
                        if(singleTon==null){     
                         singleTon=new SingleTon3();
                         }
                }
     }
     return singleTon;
}
```

第一次校验：由于单例模式只需要创建一次实例，如果后面再次调用getInstance方法时，则直接返回之前创建的实例，因此大部分时间不需要执行同步方法里面的代码，大大提高了性能。如果不加第一次校验的话，每次都要去竞争锁。

 第二次校验：如果没有第二次校验，假设线程t1执行了第一次校验后，判断为null，这时t2也获取了CPU执行权，也执行了第一次校验，判断也为null。接下来t2获得锁，创建实例。这时t1又获得CPU执行权，由于之前已经进行了第一次校验，结果为null（不会再次判断），获得锁后，直接创建实例。结果就会导致创建多个实例。所以需要在同步代码里面进行第二次校验，如果实例为空，则进行创建。

- 基于静态内部类

  ```java
  public class StaticInnerClassSingleton {
  
      private static class InnerClass {
          private static StaticInnerClassSingleton staticInnerClassSingleton = new StaticInnerClassSingleton();
      }
      public static StaticInnerClassSingleton getInstance(){
          return InnerClass.staticInnerClassSingleton;
      }
      private StaticInnerClassSingleton(){}
  }
  ```

  外部类加载时并不需要立即加载内部类，内部类不被加载则不去初始化INSTANCE，故而不占内存。但由于是静态内部类的形式去创建单例的，故**外部无法传递参数进去**且以上写法无法避免**反射**。尝试修改如下

  ```java
  public class StaticInnerClassSingleton {
  
      private static class InnerClass {
          private static StaticInnerClassSingleton staticInnerClassSingleton = new StaticInnerClassSingleton();
      }
      public static StaticInnerClassSingleton getInstance(){
          return InnerClass.staticInnerClassSingleton;
      }
      private StaticInnerClassSingleton(){
      	if(InnerClass!=null){
      		throw new RuntimeException();
      	}
      }
  }
  ```

  虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，就可能造成多个进程阻塞。其他线程虽然会被阻塞，但如果执行<clinit>()方法后，其他线程唤醒之后不会再次进入<clinit>()方法。同一个加载器下，一个类型只会初始化一次。

- 枚举

```java
public enum  EnumSingleton {
    INSTANCE;
}

EnumSingleton.INSTANCE
```

