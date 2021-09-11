## 1 性能概述
### 程序性能的几个表现方面
* 执行速度
* 内存分配: 是否合理
* 启动时间: 程序运行到正常处理业务
* 负载承受能力: 即系统压力上升时, 程序的执行速度和响应时间上升曲线是否平缓.

### 性能的参考指标
> 之间的关系: 木桶原理, 取决于最差的一个.
* 执行时间: 程序开始到结束
* CPU时间
* 内存分配:
* 磁盘吞吐量
* 网络吞吐量
* 响应时间

### 根据以上指标, 最有可能成为系统瓶颈的资源是
* (常见) 磁盘IO
* (常见) 网络操作
* (常见) CPU资源
* (常见) 数据库
* 内存, 一般不会成为瓶颈, 除非进行高频率的内存交换和扫描.
* 异常, Java捕获异常和处理非常消耗资源.
* 锁竞争, 锁竞争带来切换上下文开销很大

### Amdahl定律
> 定义串行系统并行化后的加速比的计算公式和理论上线.
* 加速比 = 优化前系统耗时 / 优化后系统耗时
> 必须串行化的系统比重F, CPU处理器数量N
* Amdahl定律: speedup <= 1 / (F + (1 - F) / N).  当N趋向于无穷, F为50%, 则最大加速比为2.
* 由此可见, 单纯的添加CPU数量不一定存在效果(N趋向于无穷时, 可见CPU数量超过一定量之后speedup <=  1/ F), 这时必须要改变F才行.


### 性能调优的层次
* 设计调优: 特点是直接规避一个组件的性能问题, 而非改良组件的实现. 例子: A循环监控E事件 或者改为 E完成后通知A.
* 代码调优: 坏的实现改为好的实现, 例子: 文件读写Stream方式 和 java NIO方式.
* JVM调优: 
* (本书不讨论)数据库调优: 1. 对SQL语句优化, 2. 对数据库结构(表,库)优化, 3. 对数据库软件本身优化.
* (本书不讨论)操作系统调优.

### 优化注意事项
* 不要为了调优而调优, 当前程序没有明显的性能问题, 只是根据主观臆测而改进程序, 往往风险大于收益.



## 2. 设计优化
> 主要介绍各种设计模式的使用和实现, 其次还有缓冲和对象池的.

### 单例模式
* 好处: 1.忽略频繁创建对象的时间, 2.new次数减少, 系统内存使用频率降低, GC压力减少, 缩短GC的停顿时间.
* 实现方法一:
```java
public class Singleton{
    private Singleton(){
        System.out.println("Singleon is create!"); // 创建单例的过程可能很慢
    }
    private static Singleton instance = new Singleton();
    public static Singleton getInstance(){
        return instance;
    } 
}
```
* 本示例的缺点: 因为instance是static的, JVM加载单例类时就会加载instance, 不能对instance实例做延迟加载(也就是不能用的时候再加载).
* 实现方法二:
```java
public class LazySingleton{
    private LazySingleton(){
        System.out.println("Singleon is create!"); 
    }
    private static LazySingleton instance = null; // 因为是null, 所以不会有额外负担
    public static synchronized Singleton getInstance(){ 
        // 但是防止多线程环境下多个实例被创建, 加上了synchronized关键字.
        if(instance == null) instance = new LazaSingleton();
        return instance;
    } 
}
``` 
* 本实例缺点: synchronized的代价很高. 比第一个还要高.
* 实现方法三:
```java
public class StaticSingleton{
    private StaticSingleton(){
        System.out.println("Singleon is create!"); 
    }
    // 利用系统加载StaticSingleton时不会初始化内部类.
    private static class SingleHolder{
        private static StaticSingleton instance = new StaticSingleton();
    }
    public static StaticSingleton getInstance(){ 
        return SingleHolder.instance;
    } 
}
// 序列化：把Java对象转换为字节序列的过程。
// 反序列化：把字节序列恢复为Java对象的过程。
// 对象的序列化主要有两种用途：
// 把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；（持久化对象）
// 在网络上传送对象的字节序列。（网络传输对象）
```
* 本实例的缺点: 1. 反射机制强行调用单例类的私有构造函数, 生成多个单例(本书不做讨论)
* 2. 序列化和反序列化会破坏单例. (情况虽然不多见, 但是要注意).
* 解决方案, 使用readSolve()函数.
```java
public class SerSingleton implements java.io.Serializable{
    String name;
    private SerSingleton(){
        System.out.println("Singleon is create!"); 
        name = "SerSingleton";
    }
    private static StaticSingleton instance = new StaticSingleton();
    public static StaticSingleton getInstance(){ 
        return instance;
    } 
    public static void createString(){
        System.out.println("createString in Singleon!"); 
    }
    // readObject 调用的实际就是readResolve
    private Object readResolve(){
        return instance;
    }
}
```

### 代理模式
> 意图: 1.安全原因, 2.屏蔽客户端直接访问真实对象, 3.远程调用中, 代理类处理远程方法的技术细节. 4.提升系统性能, 对真实对象封装, 从而达到延迟加载的目的(本节主要讨论).


##### 代理模式的结构
> 主要利用初始化时, 初始化的是代理类 而不是 真实的类(需要大量资源), 这样系统启动时速度不会很慢!
* 1. 主题接口: 定义代理类和真实主题对外的公共接口.
* 2. 真实主题: 真正实现业务逻辑的类.
* 3. 代理类: 代理和封装真实主题
* 4. Main: 客户端, 利用主题接口 和 代理类完成一些工作.
![java代理类](https://github.com/zpeng1997/java_notes/blob/master/picture/java程序性能优化_代理类.jpg) 
![java代理类实现](https://github.com/zpeng1997/java_notes/blob/master/picture/java%E7%A8%8B%E5%BA%8F%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96_%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F%E5%AE%9E%E7%8E%B0.jpg)
  
```java
// DBQuery的实现:
public class DBQuery inplements IDBQuery{
    public DBQuery(){
        try{
            Thread.sleep(1000); // 可能包含数据库链接等耗时操作
        }catch (InterruptedException e){
            e.printStackTrace();
        }
    }
    @Override
    public String request(){
        return "request string";
    }
}

// 代理类DBQueryProxy轻量级对象, 代理DBQuery
public class DBQueryProxy implements IDBQuery{
    private DBQuery real = null;
    @Override
    public String request(){
        // 真正需要的时候, 才会创建真实对象, 创建过程很慢
        if(real == null) real = new DBQuery();
        // 多线程模式下, 返回一个虚假类, 类似Future模式
        return real.request();
    }
}

// 主函数引用IDBQuery接口
public class Main{
    public static void main(String args[]){
        IDBQuery q = new DBQueryProxy(); // 使用代理
        q.request(); // 真正使用时才会创建.
    }
}
```

### 动态代理
> 不需要为真实主题写一个形式上完全一样的封装类, 加入主题接口中的方法很多, 为每一个接口写一个方法也是十分的烦人.

> 动态代理使用字节码动态生成加载技术, 在运行时生成并加载类.

> 生成动态代理类的方法: JDK自带的动态代理(弱), CGLIB(功能强大, 性能中等), Javassist(功能强大, 性能中等) 或者 ASM库(性能高, 维护性差).

> 动态代理实现
```java
// JDK的动态代理生成代理对象, 需要实现一个处理方法调用的Handler, 英语实现代理方法的内部逻辑.
public class JdkDbQueryHandler implements InvocationHandler{
    IDBQuery real = null; // 主题接口
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
        if(real == null) real = new DBQuery(); // 第一次调用, 则生成真是对象
        return real.request(); // 真实主题的完成的实际操作.
    }
}
// 使用handler生成动态对象.
public static IDBQuery createJdkProxy(){
    // newProxyInstance返回该代理的一个实例
    IDBQuery jdkProxy = (IDBQuery) Proxy.newProxyInstance(
        ClassLoader.getSystemClassLoader(),
        new Class[] {IDBQuery.clss},
        new JdkDbQueryHandler()
    );
    return jdkProxy;
}


// CGLIB生成动态代理
// 切入类
public class CglibSbQueryInterceptor implements MethodInterceptor{
    IDBQuery real = null;
    @Override
    public Object intercept(Object arg0, Method arg1, Object[] arg2, MethodProxy arg3) throws Throwable{
        if(real == null) real = new DBQuery();
        return real.request();
    }
}

public static IDBQuery createCglibProxy(){
    Enhancer enhancer = new Enhancer();
    // 指定切入器, 定义代理类逻辑
    enhancer.setCallback(new CglibDbQueryInterceptor());
    // 指定实现的接口
    enhancer.setInterfaces(new Class[] {IDBQuery.class});
    // 生成代理类的实例
    IDBQuery cglibProxy = (IDBQuery) enhancer.create();
    return cglibProxy;
}

// Javassist 有两种方法, 这里就不写了.
```

### 经典应用: Hibernate框架中.
> Hibernate的延迟加载分为: 1.属性延迟加载, 2.关联表的延时加载.
```java
// 代码在书里, 不抄了.
```


## 享元模式