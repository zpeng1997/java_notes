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

  
