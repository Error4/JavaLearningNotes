在上一篇文章中，我们已经对RabbitMQ有了一个初步的认知，在讲解其他内容之前，我们有必要先来了解下RabbitMQ是如何管理用户的。

# 1.添加用户

如下所示，可以直接通过控制台添加用户

![](https://s1.ax1x.com/2020/06/27/NcAqsS.png)

同时，也可以给用户设置角色，角色说明如下：

- Admin

  可登陆管理控制台，可查看所有信息，并且可以对用户、策略进行设置

- Monitoring

  可登陆管理控制台，同时可以查看节点的相关信息（进程数，内存使用情况等）

- Policymaker

  可登陆管理控制台，可以对策略进行设置，但无法查看节点的相关信息

- Management

  仅可以登录管理控制台

- 其他

  无法登录管理控制台，通常就是普通的生产者和消费者

点击对应的用户名，就能做修改或删除

![](https://s1.ax1x.com/2020/06/27/NcEpR0.md.png)

当然，如上操作也可以使用命令来进行，在rabbitMQ安装目录../sbin目录下：

- 查看用户

  ```shell
  rabbitmqctl.bat list_users
  ```

- 新建用户

  ```shell
  rabbitmqctl.bat add_user username passwpord
  ```

- 删除用户

  ```shell
  rabbitmqctl.bat delete_user username
  ```

- 修改用户密码

  ```shell
  rabbitmqctl change_password passwpord new-passwpord
  ```

- 角色设置

  ```shell
  rabbitmqctl.bat set_user_tags username administrator
  ```

# 2.权限配置

当我们新建好用户后，就可以直接使用了吗？如下修改代码，添加我们维护的账号信息

```java
public class Producer {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException, TimeoutException {
        // 创建连接
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 RabbitMQ 的主机名
        factory.setHost("localhost");
        factory.setUsername("wyf");
        factory.setPassword("123456");
        factory.setPort(5672);

        // 创建一个连接
        Connection connection = factory.newConnection();
        // 创建一个通道
        Channel channel = connection.createChannel();
        // 指定一个队列,不存在的话自动创建
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 发送消息
        String message = "Hello World!";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

运行代码会发现代码报错

![](https://s1.ax1x.com/2020/06/27/NcEuz6.md.png)

提示拒绝用户连接。至于原因，我们返回控制台看一下就明白了

![](https://s1.ax1x.com/2020/06/27/NcEeiR.png)

可以看到，是由于我们没有设置用户可以访问Virtual host导致的。点击用户名称设置即可

![](https://s1.ax1x.com/2020/06/27/NcEnRx.png)

也可以通过命令来执行

```shell
rabbitmqctl set_permissions -p / wyf ".*" ".*" ".*"
```

