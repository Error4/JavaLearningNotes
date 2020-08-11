# 1.接口概述

 Spring IoC容器允许`BeanFactoryPostProcessor`在容器实例化任何`bean`之前读取`bean`的定义(配置元数据)，并可以修改它。同时可以定义多个`BeanFactoryPostProcessor`，通过设置`order`属性来确定各个`BeanFactoryPostProcessor`执行顺序。

看一下它的定义：

```java
public interface BeanFactoryPostProcessor {
    	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. 
	 * All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

总结下他的注释：`postProcessBeanFactory()` 工作与 BeanDefinition 加载完成之后，**Bean 实例化之前**，其主要作用是对加载 BeanDefinition 进行修改。

> 在 `postProcessBeanFactory()` 中千万不能进行 `Bean` 的实例化工作，因为这样会导致 bean 过早实例化，会产生严重后果，我们始终需要注意的是 `BeanFactoryPostProcessor` 是与 `BeanDefinition` 打交道的，如果想要与 `Bean` 打交道，请使用 `BeanPostProcessor`。

# 2.源码分析

老规矩，还是从`AbstractApplicationContext`类的`refresh`方法看起

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
                
                ......
```

显然，`invokeBeanFactoryPostProcessors()`方法就是我们的目标了，继续看它的定义

```java
	/**
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
        ......
	}
```

可以看到，实际操作是由`PostProcessorRegistrationDelegate`去执行的。注意`invokeBeanFactoryPostProcessors`方法的第二个入参是`getBeanFactoryPostProcessors()`方法，该方法返回的是`applicationContext`的成员变量`beanFactoryPostProcessors`，而该成员变量来自于`AbstractApplicationContext.addBeanFactoryPostProcessor`方法的调用，是留给业务扩展时用的。

```java
	@Override
	public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
		Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
		this.beanFactoryPostProcessors.add(postProcessor);
	}
```

让我们再回到`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors`方法，看看`BeanFactoryPostProcessor`是如何被处理的。由于该方法内容较多，分为了好几部分去处理，截取其中我们关心的部分即可：

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors){
		......    
       //获取容器中的所有BeanFactoryPostProcessor，并进行存储，
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// 分离那些实现了PriorityOrdered,Ordered接口的和没有实现接口的    
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    	//遍历获得的所有的BeanFactoryPostProcessors
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
            //如果是实现了PriorityOrdered接口的话，添加到priorityOrderedPostProcessors之中
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
             //如果是实现了Ordered接口的话，添加到orderedPostProcessors之中
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
                //剩下的就加到这个nonOrderedPostProcessorNames之中
				nonOrderedPostProcessorNames.add(ppName);
			}
		}
    	......
            
		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
    
}
```

需要注意的是`nonOrderedPostProcessorNames`，我们自定义的实现`BeanFactoryPostProcessor`接口的`bean`就会在此处被找出来。

最后，所有实现了`BeanFactoryPostProcessor`接口的`bean`，都被作为入参调用`invokeBeanFactoryPostProcessors`方法去处理。

```java
	private static void invokeBeanFactoryPostProcessors(
			Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

		for (BeanFactoryPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessBeanFactory(beanFactory);
		}
	}
```

# 3.实战演练

定义User类如下

```java
@Component
public class User {

    private String name = "AAA";

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

定义BeanFactoryPostProcessor如下：

```java
@Component
public class CustomizeBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println(">> BeanFactoryPostProcessor 开始执行了");
        String[] names = beanFactory.getBeanDefinitionNames();
        for (String name : names) {
            if("user".equals(name)){
                BeanDefinition beanDefinition = beanFactory.getBeanDefinition(name);
                MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();
                propertyValues.addPropertyValue("name", "BBB");
            }
        }
        System.out.println(">> BeanFactoryPostProcessor 执行结束");
    }
}
```

上述代码的功能很简单，找到名为”user”的`bean`的定义对象，通过调用`addPropertyValue`方法，将定义中的`name`属性值改为”BBB”，这样等名为”USER”的`bean`被实例化出来后，其成员变量`name`的值就是”BBB”，我们执行测试类

```java
@SpringBootTest
class ApplicationTests {
    @Autowired
    User user;
    @Test
    void contextLoads() {
        System.out.println(user.getName());
    }
}
```

可以看到控制台输出“BBB”

