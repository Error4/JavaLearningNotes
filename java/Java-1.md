# 1.Java基本结构

## 1.1数据类型

我们知道，long类型的数值会添加一个后缀L或l,16进制数有一个前缀0x或0X，8进制有一个前缀0，而从Java 7开始，添加前缀0b或0B，就可以表示二进制数，比如0b1001就是9，还可以为数字字面量添加下划线，比如1_000_000，更加易读，Java编译器会自动去除这些下划线。

float类型数值有一个后缀F或f，没有后缀默认为double。在浮点数中，有三个特殊的值：正无穷大，负无穷大，NaN(not a number，不是一个数字)。以double为例，在其源码中可以看到分别表示如下：

```java
public static final double POSITIVE_INFINITY = 1.0 / 0.0;
public static final double NEGATIVE_INFINITY = -1.0 / 0.0;
public static final double NaN = 0.0d / 0.0;
```

注意：对于NaN，不能直接用=来判断，例如

```java
if(x=Double.NaN)
```

所有非数值的值都认为是不相同的，如上表达式永远为false，可以用`isNaN()`方法来判断。

## 1.2 控制流程

Java提供了一种带标签的break语句，用于跳出多重嵌套的循环语句。**标签必须放在希望跳出的最外层循环之前，并且必须紧跟一个冒号。**如下所示，`aa`是自定义的标签名称。

```java
public static void main(String[] args) {
    aa:
    while (true){
        for (int i = 0; i <10 ; i++) {
            System.out.println(i);
            if (i==3){
                break aa;
            }
        }
    }
}
```

continue同理，也可以带标签，将跳到与标签匹配的循环首部。

## 1.3 大数

我们知道，浮点数在进行运算时，由于浮点数值采用二进制系统表示，而二进制无法精确表示分数1/10，就好像十进制无法精确表示1/3一样，可能会存在精度丢失的问题，而基本的Int型或long类型有时也限于位数不够因此，可以使用`BigDecimal`和`BigInteger`来实现任意精度的整数和浮点数运算。

使用静态的`valueof()`方法就能将普通数值转换为对应的"Big"类型：

```java
BigInteger a = BigInteger.valueOf(10);
```

需要注意，不能使用运算符（+，*）等直接对他们进行运算，而要使用对应的方法，`add(),multiply()`等

# 2.接口，内部类

java8之前，接口的方法默认都是public修饰,且方法没有实现，字段都是public static final修饰，java8之后，允许接口定义默认方法和静态方法。静态方法增强了接口的功能，而对其子类和子接口没有影响，而对于默认方法，子类无须实现就可以拥有该方法，优雅的扩展了接口。

```java
public interface Test {
    default String getString(){
        return "";
    }
    static int getInt(){
        return 0;
    }
}
```

需要说明的是，如果默认方法冲突该怎么办呢?Java提供了如下两种规则：

- 超类优先：如果一个类继承一个超类且同时实现一个接口，如果超类提供了一个具体方法，那么同名且具有相同参数类型的默认方法会被忽略

- 接口冲突重写：如果两个接口都提供了相同的默认方法，则实现类必须覆盖这个方法解决冲突

  ```java
  interface A{
      default String getName(){
          return "A";
      }
  }
  interface B{
      default String getName(){
          return "B";
      }
  }
  class C implements A,B{
      public String getName(){
          return A.super.getName();
      }
  }
  ```

在Java9之后，接口的方法也可以是private的了，通常作为接口中其他方法的辅助方法。

## 2.1 comparable

一般而言，实现`comparable`接口时，都建议其`compareTo`方法与`equals`方法一致，即`x.equals(y)`时，`x.compareTo(y)`应当等于0，大多数类都遵从这个协议，但有一个重要的例外，`BigDecimal`

```java
BigDecimal x = new BigDecimal("1.0");
BigDecimal y = new BigDecimal("1.00");
System.out.println(x.equals(y)+","+x.compareTo(y));
```

打印输出，会发现`x.equals(y)`值为false，因为两个数精度不同。，而`x.compareTo(y)`等于0，理想结果应该不返回0才对，但是没有明确的方法能够确定这两个数的大小。

