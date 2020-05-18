# 1.什么是循环依赖

循环依赖就是循环引用，两个或多个bean之间互相持有对方，比如如下所示，`StudentA`引用`StudentB`，`StudentB`引用`StudentC`，`StudentC`引用`StudentA`。

```java

public class StudentA {
 
    private StudentB studentB ;
 
    public void setStudentB(StudentB studentB) {
        this.studentB = studentB;
    }
 
    public StudentA() {
    }
 
    public StudentA(StudentB studentB) {
        this.studentB = studentB;
    }
 
}
 
 
public class StudentB {
 
    private StudentC studentC ;
 
    public void setStudentC(StudentC studentC) {
        this.studentC = studentC;
    }
 
    public StudentB() {
 
    }
 
    public StudentB(StudentC studentC) {
        this.studentC = studentC;
    }
 
}
 
 
public class StudentC {
 
    private StudentA studentA ;
 
    public void setStudentA(StudentA studentA) {
        this.studentA = studentA;
    }
 
    public StudentC() {
 
    }
 
    public StudentC(StudentA studentA) {
        this.studentA = studentA;
    }
 
}
```

# 2.Spring如何解决循环依赖

Spring容器循环依赖包括构造器循环依赖和setter循环依赖，在Spring中将循环依赖分为了3种情况：

## 2.1 构造器循环依赖

通过构造器注入构成的循环依赖，Spring无法解决，只能抛出`BeanCurrentlyInCreationException`异常表示循环依赖。

Spring容器会将每一个正在创建的Bean 标识符放在一个“当前创建Bean池”中，Bean标识符在创建过程中将一直保持在这个池中，因此如果在创建Bean过程中发现自己已经在“当前创建Bean池”里时将抛出`BeanCurrentlyInCreationException`异常表示循环依赖；而对于创建完毕的Bean将从“当前创建Bean池”中清除掉。

还是上文中的例子，Spring容器先创建单例StudentA，StudentA依赖StudentB，然后将A放在“当前创建Bean池”中，此时创建StudentB，StudentB依赖StudentC ,然后将B放在“当前创建Bean池”中，此时创建StudentC，StudentC又依赖StudentA， 但是，此时StudentA已经在池中，所以会报错，，因为在池中的Bean都是未初始化完的，所以会产生依赖错误 。

## 2.2 setter循环依赖

熟悉Spring中Bean过程的读者，应该知道，**Spring是先将Bean对象实例化之后再设置对象属性的**。

对于setter注入造成的依赖是通过Spring容器提前暴露刚完成构造器注入但未完成其他步骤（如setter注入）的bean来完成的，通过提前暴露一个单例工厂方法，从而使其它bean能引用到该bean.

在Spring的Bean包中的`DefaultSingletonBeanRegistry.java`类由如下代码所示：

```java
/** Cache of singleton objects: bean name --> bean instance（缓存单例实例化对象的Map集合） */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);
 
    /** Cache of singleton factories: bean name --> ObjectFactory（单例的工厂Bean缓存集合） */
    private final Map<String, ObjectFactory> singletonFactories = new HashMap<String, ObjectFactory>(16);
 
    /** Cache of early singleton objects: bean name --> bean instance（提前暴光的单例对象的缓存集合） */
    private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
 
    /** Set of registered singletons, containing the bean names in registration order（单例的实例化对象名称集合） */
    private final Set<String> registeredSingletons = new LinkedHashSet<String>(64);
    /**
     * 添加单例实例
     * 解决循环引用的问题
     * Add the given singleton factory for building the specified singleton
     * if necessary.
     * <p>To be called for eager registration of singletons, e.g. to be able to
     * resolve circular references.
     * @param beanName the name of the bean
     * @param singletonFactory the factory for the singleton object
     */
    protected void addSingletonFactory(String beanName, ObjectFactory singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        synchronized (this.singletonObjects) {
            if (!this.singletonObjects.containsKey(beanName)) {
                this.singletonFactories.put(beanName, singletonFactory);
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }
        }
    }
```

## 2.3 prototype范围的依赖处理

对于prototype作用域的bean，Spring容器无法完成依赖注入，因为Spring容器不进行缓存prototype作用域的bean，因此也无法提前暴露一个创建中的bean。

# 3.补充说明

在2.2节中我们看到，spring为了解决循环依赖，利用了三级缓存的结构

```java
/** Cache of singleton objects: bean name --> bean instance（缓存单例实例化对象的Map集合） */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);
 
    /** Cache of singleton factories: bean name --> ObjectFactory（单例的工厂Bean缓存集合） */
    private final Map<String, ObjectFactory> singletonFactories = new HashMap<String, ObjectFactory>(16);
 
    /** Cache of early singleton objects: bean name --> bean instance（提前暴光的单例对象的缓存集合） */
    private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```

在创建bean的时候，首先想到的是从cache中获取这个单例的bean，这个缓存就是`singletonObjects`。主要调用方法就就是：

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

上面的代码需要解释两个参数：

- `isSingletonCurrentlyInCreation()`：判断当前单例bean是否正在创建中，也就是没有初始化完成(比如A的构造器依赖了B对象所以得先去创建B对象，这时的A就是处于创建中的状态。)
- `allowEarlyReference` 是否允许从`singletonFactories`中通过`getObject`拿到对象

代码过程简单的说就是，Spring首先从一级缓存`singletonObjects`中获取。如果获取不到，并且对象正在创建中，就再从二级缓存`earlySingletonObjects`中获取。如果还是获取不到且允许`singletonFactories`通过`getObject()`获取，就从三级缓存`singletonFactory.getObject()`(三级缓存)获取，如果获取到了则：

```java
this.earlySingletonObjects.put(beanName, singletonObject);
this.singletonFactories.remove(beanName);
```

从`singletonFactories`中移除，并放入`earlySingletonObjects`中。其实也就是从三级缓存移动到了二级缓存。

可以看到，最终都要落到`singletonFactories`这个最后的cache上，继而落在上文提到的`addSingletonFactory`方法

```java
protected void addSingletonFactory(String beanName, ObjectFactory singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        synchronized (this.singletonObjects) {
            if (!this.singletonObjects.containsKey(beanName)) {
                this.singletonFactories.put(beanName, singletonFactory);
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }
        }
```

综上，我们就更清晰的知道，为什么构造器的循环依赖无法解决了？因为**加入`singletonFactories`三级缓存的前提是执行了构造器**。

