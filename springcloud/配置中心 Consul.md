	在上一篇文章中，我们已经知道Spring Cloud Config，它提供了配置中心的功能，但是需要配合 git、svn 或外部存储（例如各种数据库），且需要配合 Spring Cloud Bus 实现配置自动刷新。那么有没有更优秀的替代方案呢？Consul值得一试。

# 1.概述

​			Consul 是 HashiCorp 公司推出的开源工具，用于实现分布式系统的服务发现与配置。与其他分布式服务注册与发现的方案，Consul的方案更“一站式” ，内置了服务注册与发现框 架、具有以下性质：

​		● 分布一致性协议实现
​		● 健康检查
​		● Key/Value存储
​		● 多数据中心方案

​		可以看到，Consul 还具有类似Eureka，作为服务注册中心的能力，详情可以参考[Spring Cloud 系列之 Consul 注册中心](https://www.cnblogs.com/mrhelloworld/p/consul1.html)

# 2.入门案例

## 2.1 Consul相关准备

​		访问 Consul 官网：`https://www.consul.io` 下载 Consul 的最新版本。解压缩后，cd 到对应的目录下，使用 cmd 启动 Consul

```
# -dev表示开发模式运行
consul agent -dev -client=0.0.0.0
```

​		访问管理后台：`http://localhost:8500/` 看到下图意味着我们的 Consul 服务启动成功了。

![](https://s1.ax1x.com/2020/04/30/Jq40zV.md.jpg)

​		接着，就可以创建我们的配置文件了

​		点击菜单 `Key/Value` 再点击 `Create` 按钮，创建 `config/` 基本目录，可以理解为配置文件所在的最外层文件夹。

​		继续点击进入 `config`文件夹，创建 `service/` 应用目录，存储对应微服务应用的环境配置信息。

最后，准备结果如下

![](https://s1.ax1x.com/2020/04/30/JLuMHx.jpg)

​		可以看到，基于路径`config/service/`中创建了key为`serviceConfig`，value为`name: service-dev`属性

## 2.2 程序准备

### 添加依赖

​		需要从 Consul 获取配置信息的项目主要添加 `spring-cloud-starter-consul-config` 依赖，如下：

```yaml
<dependencies>
        <!-- spring boot web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- spring boot actuator 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- spring cloud consul discovery 服务发现依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
        <!-- spring cloud consul config 配置中心依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-config</artifactId>
        </dependency>

        <!-- spring boot test 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

```

### 配置文件

​		与Spring Cloud Config一样，依然需要配置在bootstrap.yml中

```yaml
server:
  port: 7012 # 端口
spring:
  application:
    name: consul # 应用名称
  cloud:
    consul:
      # Consul 服务器地址
      host: localhost
      port: 8500
      # 配置中心相关配置
      config:
        # 是否启用配置中心，默认值 true 开启
        enabled: true
        # 设置配置的基本文件夹，默认值 config 可以理解为配置文件所在的最外层文件夹
        prefix: config
        # 设置应用的文件夹名称，默认值 application 一般建议设置为微服务应用名称
        default-context: service
        # 指定配置格式为 yaml
        format: YAML
        # Consul 的 Key/Values 中的 Key，Value 对应整个配置文件
        data-key: serviceConfig
        # 以上配置可以理解为：加载 config/service/ 文件夹下 Key 为 serviceConfig 的 Value 对应的配置信息
        watch:
          # 是否开启自动刷新，默认值 true 开启
          enabled: true
          # 刷新频率，单位：毫秒，默认值 1000
          delay: 1000
```

### 启动类

​		普通的Springboot启动，无需其他注解。

```java
@SpringBootApplication
public class ConsulApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsulApplication.class,args);
    }
}
```

**需要变更配置的类上，添加@RefreshScope注解，否则不会刷新配置。**

```java
@RestController
@RefreshScope
public class ConfigController {
    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }
}
```

### 测试

​		请求`http://localhost:7012/name`，可以得到正确的配置信息`service-dev`

​		将consul中的配置信息修改，可以看到IDEA控制台打印信息如下，自动更新生效。

```
INFO 7700 --- [TaskScheduler-1] o.s.c.e.event.RefreshEventListener       : Refresh keys changed: [name]
```

​		Consul 使用 Spring 定时任务 `Spring TaskScheduler`来监听配置文件的更新。

​		再次请求`http://localhost:7012/name`，会发现返回值已经自动更新。

# 3.利用Apollo配置中心

​		除了Consul，基于携程框架部门研发的分布式配置中心Apollo（阿波罗）也是理想的选择，参考文章

[SPRING CLOUD 系列之 APOLLO 配置中心](https://mrhelloworld.cn/articles/spring/spring-cloud/apollo/)



相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看