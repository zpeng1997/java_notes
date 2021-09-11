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
> 少数几个以提高系统性能为目的模式之一; 用来复用大对象.

> 解决的问题: 一个系统的多个对象, 只需要一个对象的拷贝; 然后因为需要生产和管理对象, 所以一般需要一个工厂类.

> 提升了哪里的性能? 1.创建的对象减少, 耗时减少. 2. 需要内存减少, GC的压力少.

### 享元模式的组成部分
* 享元工厂: 创建具体的享元类, 维护享元对象. 算法类似单例模式.
* 抽象享元: 定义接口
* 具体享元: 实现接口
* Main: 通过享元工厂获的对象. 
![享元模式]()

### 享元模式的例子
> 三个公司用同一套系统查询员工信息, 我们可以维护三个享元模式的数据库链接, 每一个数据库链接对象对应公司. 
> 相同公司的员工, 享用同一个数据库链接; 而且不同公司的数据库链接是不同的.
> 享元模式与对象池的区别: 享元模式的每个对象都是不同的, 对象池中所有对象都是相同的可以替换.
![java程序性能优化_享元模式示例]()

```java
// 享元对象接口, 用于创建一个报表
public interface IReportManager{
    public String createReport();
}

// 两个报表的生成实例, 分别对应员工财务收入报表和员工个人x信息报表
public class FinancialReportManager implements IReportManager{ // 财务报表
    protected String tenantID = null;
    // 租户
    public FinancialReportManager(String tenantId){
        this.tenantId = tenantId;
    }
    @Override
    public String createReport(){
        return "This is a financial report";
    }
}

public class EmployeeReportManager implements IReportManager{ // 员工报表
    protected String tenantId = null;
    // 租户
    public EmployeeReportManager(String tenantId){
        this.tenantId = tenantId;
    }
    @Override
    public String createReport(){
        return "This is a employee report";
    }
}

// 享元模式的精髓:
// 享元工厂类
public class ReportManagerFactory{
    Map<String, IReportManager> financialReportManager = new HashMap<String, IReportManager>();
    Map<String, IReportManager> employeeReportManager = new HashMap<String, IReportManager>();
    IReportManager getFinancialReportManager(String tenantId){
        IReportManager r = financialReportManager.get(tenantId);
        if(r == null) {
            r = new FinancialReportManager(tenantId);
            // 维护已创建 享元对象
            finfncialReportManager.put(tenantID, r);
        }
        return r;
    }

    IReportManager getEmployeeReportReportManager(String tenantId){
        IReportManager r = employeeReportManager.get(tenantId);
        if(r == null){
            r = new EmployeeReportManager(tenantId);
            employeeReportManger.put(tenantId, r);
        }
        return r;
    }
}

// 享元模式的使用方法如下:
public static void main(Stirng[] args){
    ReportManagerFactory rmf = new ReportManagerFactory();
    IReportManager rm = rmf.getFinancialReportManager("A");
    System.put.println(rm.createReport());
}
```

## 装饰者模式
> 合成/聚合复用原则, 可以动态添加对象功能.

> 代码复用尽可能使用委托而不是用继承, 继承是一种紧密耦合, 任何父类的改动都会影响子类, 不利于系统维护; 而委托是松散耦合, 只要接口不变, 委托类的改动并不会影响其上层对象.
![装饰者模式]()

### 装饰者模式的组成
* 组件接口: 是装饰者和被装饰者的超类或者接口, 它定义了被装饰者的核心功能和被装饰者需要加强的功能点.
* 装饰者: 实现组件接口, 并持有一个具体的被装饰对象.
* 具体组件: 实现组件接口的核心方法, 完成某一个具体的业务逻辑. 也是被装饰者的对象.
* 具体装饰者: 实现装饰的业务逻辑, 即实现分离的各个增强功能点。各个具体装饰者可以相互叠加, 构成一个功能更强大的组件对象.

### 装饰者模式典型案例
> 装饰者模式一个典型案例就是对输出结果进行增强. 例如需要将一个文本, 通过HTML发布, 首先要转化为HTML文本, 其次还要加上一个HTTP头, 还可能要安置TCP等等. 装饰者模式可以将这些分为独立的模块, 灵活的装配.
![装饰者模式示例]()

```java
// 装饰接口
public interface IPacketCreator{
    public String handleContent)(); // 用于内容处理
}
// 用于返回数据包的核心数据
public class PacketBodyCreator implements IPacketCreator{
    @Override
    public String handleContent(){
        return "Content of Packet"; // 构造核心数据, 但不包括格式
    }
}
// packetDecorator维护核心组件component对象, 他负责告诉其子类, 其和新业务逻辑应该全权委托component完成, 自己只是增强处理
public abstract class PacketDecorator implements IPacketCreator{
    IPacketCreator component;
    public PacketDecorator(IPacketCreator c){
        component = c;
    }
}
// PacketHTMLHeaderCreator 是具体的装饰器, 它负责对核心发布的内容进行HTML格式化操作
// 特别注意: 它委托了具体组件component进行核心业务处理
public class PacketHTMLHeaderCreator extends PacketDecorator{
    public PacketHTMLHeaderCreator(IPacketCreator c){
        super(c):
    }
    @Override
    // 将给定数据封装成HTML
    public String handleContent(){
        StirngBuffer sb = new StringBuffer();
        sb.append("<html>");
        sb.append("<body>");
        sb.apped(component.handleContend());
        sb.append("</body>");
        sb.append("</html>\n");
        return sb.toString();
    }
}
// PacketHTTPHeaderCreator与PacketHTMLHeaderCreator类似, 但是它完成HTTP头部处理, 其余依然交给内部的component完成
public class PacketHTTPHeaderCreator extends PacketDecorator{
    public PacketHTTPHeaderCreator(IPacketCreator c){
        super(c);
    }
    @Override
    public String handleContend(){
        StringBuffer sb = new StringBuffer();
        sb.append("Cache-Control:no-cache\n");
        sb.append("Date:Mon,11Sep202111:31:00GMT\n");
        sb.append(component.handleContend());
        return sb.toString();
    }
}
// 装饰者模式的使用方法
public class Main(){
    public static void main(String[] args){
        IPacketCreator pc = new PacketHTTPHeaderCreator(new PackerHTMLHeaderCreator(new PacketBodyCreator()));
        System.out.println(pc.handleContend());
    }
}
```