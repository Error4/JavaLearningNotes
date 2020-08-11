> 最近在接手公司之前的一个项目，熟悉代码的同时，发现项目利用了颇多Spring的扩展功能，有些虽然知道，但没怎么深入去看过，还有一些更是第一次看到......找资料熟悉的时候，发现一位博主写的系列文章挺全面，但是基于Spring 4.1.8分析的。所以，站在前人的肩膀上，准备基于现行的Spring5自己过一遍，一是为了自我学习，二是也借此分享给大家，大家也可以比对下，看看Spring两个版本间的区别。
>
> 原博主文章链接：[spring4源码分析与实战](https://blog.csdn.net/boling_cavalry/category_9277994.html)

本篇文章我们就先了解下如何实现**自定义环境变量验证**的功能，如果不存在则应用自动失败

# 1.Spring源码分析

首先，在Spring环境初始化的时候，AbstractApplicationContext的prepareRefresh方法会被调用，源码如下

```java
protected void prepareRefresh() {
		// Switch to active.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

不用我说，看官方的注释，大家应该都能定位到关键的方法，没错，就是`getEnvironment().validateRequiredProperties()`。

我们继续跟踪该方法，可以看到，最终对应的是`AbstractPropertyResolver`类的`validateRequiredProperties`方法

```java
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}
```

可以看到，spring容器初始化的时候，会从集合`requiredProperties`中取出所有key，然后获取这些`key`的环境变量（包括系统环境变量和进程环境变量），如果有一个`key`对应的环境变量为空，就会抛出异常，导致`spring`容器初始化失败。

# 2.扩展功能分析

综上，目前我们已经知道，只要我们把需要验证的环境变量的`key`存入集合`requiredProperties`中，即可达成我们需要的验证功能。那么，怎么存和何时存就成了我们需要解决的问题：

- 如何将环境变量的`key`存入集合`requiredProperties`

  通过查看`AbstractPropertyResolver`的其他方法，明显的，`setRequiredProperties`就是用来设置`requiredProperties`：

  ```java
  	public void setRequiredProperties(String... requiredProperties) {
  		Collections.addAll(this.requiredProperties, requiredProperties);
  	}
  ```

-  在什么时候执行`AbstractPropertyResolver`类的`setRequiredProperties`方法设置key？

  创建一个`AbstractApplicationContext`的子类，重写`initPropertySources`方法，在此方法中执行`AbstractPropertyResolver`类的`setRequiredProperties`。

# 3.实战演练

> 为了方便开发和测试，我们的扩展实战是在SpringBoot框架下进行的

创建一个类`CustomApplicationContext`，继承自`AnnotationConfigServletWebServerApplicationContext`，重写`initPropertySources`方法

```java
public class CustomApplicationContext extends AnnotationConfigServletWebServerApplicationContext {
    @Override
    protected void initPropertySources() {
        super.initPropertySources();
        //把"MYSQL_HOST"作为启动的时候必须验证的环境变量
        getEnvironment().setRequiredProperties("MYSQL_HOST");
    }
}
```

创建应用启动类Application，指定ApplicationContext的class为CustomApplicationContext：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(Application.class);
        springApplication.setApplicationContextClass(CustomApplicationContext.class);
        springApplication.run(args);
    }
}
```

利用Maven打包，得到对应jar包：`spring-0.0.1-SNAPSHOT.jar`

接下来我们验证是否实现了环境变量验证的功能，在target目录下执行：`java -jar spring-0.0.1-SNAPSHOT.jar`，会发现应用启动失败，日志中显示由于找不到环境变量”MYSQL_HOST”

```
org.springframework.core.env.MissingRequiredPropertiesException: The following properties were declared as required but could not be resolved: [MYSQL_HOST]
```

而当我们执行如下命令时，可以将”MYSQL_HOST”设置到进程环境变量中，这次顺利通过校验，应用启动成功

`java -D MYSQL_HOST="192.168.0.3" -jar spring-0.0.1-SNAPSHOT.jar`

综上，验证完成，我们可以通过自定义子类来强制要求指定的环境变量必须存在。