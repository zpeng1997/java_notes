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

#### 经典应用: Hibernate框架中.
> Hibernate的延迟加载分为: 1.属性延迟加载, 2.关联表的延时加载.
```java
// 代码在书里, 不抄了.
```


### 享元模式
> 少数几个以提高系统性能为目的模式之一; 用来复用大对象.

> 解决的问题: 一个系统的多个对象, 只需要一个对象的拷贝; 然后因为需要生产和管理对象, 所以一般需要一个工厂类.

> 提升了哪里的性能? 1.创建的对象减少, 耗时减少. 2. 需要内存减少, GC的压力少.

#### 享元模式的组成部分
* 享元工厂: 创建具体的享元类, 维护享元对象. 算法类似单例模式.
* 抽象享元: 定义接口
* 具体享元: 实现接口
* Main: 通过享元工厂获的对象. 
![享元模式]()

#### 享元模式的例子
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

### 装饰者模式
> 合成/聚合复用原则, 可以动态添加对象功能.

> 代码复用尽可能使用委托而不是用继承, 继承是一种紧密耦合, 任何父类的改动都会影响子类, 不利于系统维护; 而委托是松散耦合, 只要接口不变, 委托类的改动并不会影响其上层对象.
![装饰者模式]()

#### 装饰者模式的组成
* 组件接口: 是装饰者和被装饰者的超类或者接口, 它定义了被装饰者的核心功能和被装饰者需要加强的功能点.
* 装饰者: 实现组件接口, 并持有一个具体的被装饰对象.
* 具体组件: 实现组件接口的核心方法, 完成某一个具体的业务逻辑. 也是被装饰者的对象.
* 具体装饰者: 实现装饰的业务逻辑, 即实现分离的各个增强功能点。各个具体装饰者可以相互叠加, 构成一个功能更强大的组件对象.

#### 装饰者模式典型案例
> 装饰者模式一个典型案例就是对输出结果进行增强. 例如需要将一个文本, 通过HTML发布, 首先要转化为HTML文本, 其次还要加上一个HTTP头, 还可能要安置TCP等等. 装饰者模式可以将这些分为独立的模块, 灵活的装配.
![装饰者模式示例]()

```java
// 装饰接口
public interface IPacketCreator{
    public String handleContent(); // 用于内容处理
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
        sb.apped(component.handleContent());
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
    public String handleContent(){   // 数据加上HTTP文件头.
        StringBuffer sb = new StringBuffer();
        sb.append("Cache-Control:no-cache\n");
        sb.append("Date:Mon,11Sep202111:31:00GMT\n");
        sb.append(component.handleContent());
        return sb.toString();
    }
}
// 装饰者模式的使用方法
public class Main(){
    public static void main(String[] args){
        // java用法?
        IPacketCreator pc = new PacketHTTPHeaderCreator(new PackerHTMLHeaderCreator(new PacketBodyCreator()));
        System.out.println(pc.handleContent());
    }
}
```
> 装饰者模型在JDK的例子: OutputStream 和 InputStream类族
![装饰者在OutputStream类族]()
![装饰者在OutputStream类族2]()
![装饰者在OutputStream类族3]()

### 观察者模式
> (非常常见)一个对象需要另一个对象的状态时需要.
![观察者模式结构]()

#### 观察者模式结构
* 主题接口: 被观察的对象, 若状态发生改变, 它会通知观察者
* 具体主题: 实现主题接口中的方法. 内部维护一个观察者的列表
* 观察者接口: 定义观察者的基本方法, 对象状态发生改变时, 主题接口会调用观察者update方法
* 具体观察者: 实现观察者接口的update().

```java
// 主题接口的实现
public interface ISubject{
    void attach(IObserver observer); // 添加观察者
    void detach(IObserver observer); // 删除观察者
    void inform();
}
// 观察者接口的实现
public interface IObserver{
    void update(Event evt); // 更新观察者
}
//
public class ConcreteSubject implements ISubject{
    Vector<IObserver> observers = new Vector<IOvserver>();
    public void attach(IObserver observer){
        observers.addElement(observer);
    }
    public void detach(IObserver observer){
        observers.removeElement(observer);
    }
    public void inform(){
        Event evt = new Event();
        for(IObserver ob : observers){
            ob.update(evt); // 注意, 这里通知观察者
        }
    }
}
// 具体观察者的实现
public class ConcreteObserver implements IObserver{
    public void update(Event evt){
        System.out.println("observer receives information");
    }
}
```
> 观察者模式非常常用, JDK中有为开发人员准备到一套观察者模式, java.util.Observable类 和 java.util.Observer接口.
![java观察者模式_JDK内置的观察者]()

### Value Object 模式
> 减少交互次数, 直接把需要的信息封装后统一发出.
![java程序性能优化_原来的模式]()
![java程序性能优化_现在的模式]()

```java
// RMI服务端的接口实现
public interface IOrderManager extends Remote{
    // Value Object模式
    public Order getOrder(int id) throws RemoteException;
    // 原来的模式
    public String getClientName(int id) throws RemoteException;
    public String getProdName(int id) throws RemoteException;
    public int getNumber(int id) throws RemoteException;
}

// 一个最简单的IOrderManager的实现, 什么也没有做, 只是返回数据.
public class OrderManager extends UnicastRemoteObject implements IorderManager{
    protected OrderManager() throws RemoteException{
        super();
    }
    private static final long serialVersionUID = -1717013007581295639L;
    @Override
    public Order getOrder(int id) throws RemoteException{
        // 返回订单消息
        Order o = new Order();
        o.setClientName("billy");
        o.setNumber(20);
        o.setProdunctName("desk");
        return o;
    }
    @Override
    public String getClientName(int id) throws RemoteException{
        // 返回订单的客户名
        return "billy";
    }
    @Override
    public String getProdName(int id) throws RemoteException{
        return "desk";
    }   
    @Override
    public int getNumber(int id) throws RemoteException{
        // 返回数量
        return 20;
    }
}

// value Object的Order对象实现如下
public class Order implements java.io.Serializable{
    private int orderid;
    private String clientName;
    private int number;
    private String prodenctName;
    // 省略 setter getter方法
}

// 业务逻辑层注册并开启RMI服务器
public class OrderManagerServer{
    public static void main(String[] argv){
        try
        {
            // 注册RMI端口
            LocateRegistry.createRegistry(1099);
            // RMI远程对象
            IOrderManager usermanager = new OrderManager();
            // 绑定RMI对象
            Naming.rebind("OrderManager", usermanager);
            System.out.println("OrderManager is ready.");
        }
        catch(Exception e)
        {
            System.out.println("OrderManager Server failed：" + e);
        }
    }
}

// 客户端测试代码, 分别展示使用Value Object模式和不使用的性能差异
public static void main(String[] argv){
    try{
        IOrderManager usermanager = (IOrderManager) Naming.lookup("OrderManager");
        long begin = System.currentTimeMillis();
        for(int i = 0; i < 1000; i ++){
            usermanager.getOrder(i); // Value Object 模式
        }
        System.out.println("getOrder spend:" + 
                            (System.currentTimeMillis() - begin));
        begin = System.currentTimeMillis();
        for(int i = 0; i < 1000; i ++){
            // 多次交互获取数据
            usermanager.getClientName(i);
            usermanager.getNumber(i);
            usermanager.getProdName(i);
        }
        System.out.println("3 Method call spend:"
                            + (System.currentTimeMillis() - begin));
        System.out.println(usermanager.getOrder(0).getClientName());
    } catch(Exception e){
        System.out.println("OrderManager exception: " + e);
    }
}
// 结果使用getOrder方法相对耗时469ms
// 连续使用3次离散的远程调用需要766ms
```

### 业务代理模式
> 类比Value Object模式, 业务代理是把一组由远程方法调用构成的业务流程, 封装在代理类中.

> 问题: 1. 展示层存在大量并发线程. 2. 缺乏对订单的有效封装, 将来流程发生变化时, 展示层组件需要修改时不好修改.
![java程序性能优化_业务代理1]()
![java程序性能优化_业务代理2]()

```java
// 未使用业务代理模式
public static void main(String[] argv){
    try{
        IOrderManager usermanager = (IOrderManager) Naming.lookup("OrderManager");
        // 所有的远程调用都会被执行
        // 当并发量较大时, 严重影响性能
        if(usermanager.checkUser(1)){
            Order o = usermanager.getOrder(1);
            o.setNumber(10);
            usermanager.updateOrder(o);
        }catch (Exception e){
            System.out.println("OrderManager exception: " + e);
        }
    }
}

// 使用业务代理模式
// 只会把updateOrder提供给展示层.
public static void main(String[] argv) throws Exception{
    BusinessDelegate bd = new BusinessDelegate();
    Order o = bd.getOrder(11);
    o.setNumber(11);
    bd.updateOrder(o); // 使用业务代理完成订单
}

// BussinessDelegate中, 可以增加缓存, 从而减少
// 远程调用方法调用的数
public class BusinessDelegate{
    // 封装远程方法调用的流程
    IOrderManager usermanager = null;
    public BussinessDelegate(){
        try{
            usermanager = (IOrderManager) Naming.lookup("OrderManager");
        }catch(MalformedURLException e){
            e.printStackTrace();
        }catch(RemoteExecption e){
            e.printStackTrace();
        }catch(NotBoundException e){
            e.printStackTrace();
        }
    }

    public boolean checkUserFormCache(int uid){
        return true;
    }
    public boolean checkUser(int uid) throws RemoteException{
        // 当前对象被多个客户端共享
        // 可以在本地缓存中校验用户
        if(!checkUserFormCachce(uid)){
            return usermanager.checkUser(1);
        }
        return true;
    }
    public Order getOrderFormCache(int oid){
        return null;
    }
    public Order getOrder(int oid) throws RemoteException{
        // 可以在本地缓存中获取订单
        // 减少远程方法调用次数
        Order order = getOrderFromCache(oid);
        if(order == null){
            return usermanager.getOrder(oid);
        }
        return order;
    }
    public boolean updateOrder(Order order) throws Execption{
        // 暴露给展示层的方法
        // 封装了业务流程, 可能在缓存中执行
        if(checkUser(1)){
            Order o = getOrder(1);
            o.setNumber(10);
            usermanager.updateOrder(o);
        }
        return true;
    }
}
```


## 2.2 常用优化组件和方法

### 缓冲
> 缓冲过小不会起作用, 缓冲过大则会浪费系统内存, 增加GC负担.
```java
// BufferWriter 为FileWriter对象增加缓冲能力
public BufferedWriter(Writer out);
// 可以指定缓冲区大小
public BufferedWriter(Writer out, int sz);

// BufferedOutputStream为所有OutputStream提供缓冲能力
public BufferedOutputStream(OutputStream out);
public BufferedOutputStream(OutputStream out, int size);

// NIO的Buffer类族提供更为强大的专业的缓冲控制能力(第三章).
// 比如画图, 直接动图会有点卡, 若是把点放在内存, 最后一次性的给出, 这样就会快很多.
```

### 缓存
> EHCache, OSCache 和 JBossCache

### 对象复用 —— "池"
> C3P0 和 Proxool 数据库链接池组件
> JDK中, new操作的效率相当高, 不需要担心频繁的new给性能的影响; 但是new操作时调用的类构造函数可能是非常费时的, 对于这些对象可以考虑池化.

### 并行代替串行

### 负载均衡
* 使用tomcat集群
> 黏性session模式: 均匀分配到tomcat集群, 但是一旦一个节点宕机, 它所维护的Session信息将丢失.

> 复制session模式: session信息每个机器都有一份, 但是一旦一个节点Session信息修改了, 会通过广播修改其它所有的机器上的Session信息.

* 使用terracotta: 跨JVM虚拟机
> 多个Java应用服务器之间共享缓存.

### 时间换空间

### 空间换时间

# 3. Java程序优化

## 3.1 字符串优化处理
* String对象的定义: 1. char数组 2. offset偏移 3. count长度
* String对象的3个基本特点:
* 1. 不变性: String对象一旦生成, 不能再发生改变.
* 2. 针对常量池的优化: 当两个String对象拥有相同值时, 它们只引用常量池中的同一个拷贝. 同一个字符串反复出现时, 会大幅节省内存空间.
![String优化]()
* 3. final类: 不能有任何子类。使用final定义, 有助于虚拟机寻找机会, 内联所有的final方法, 从而提升系统效率。(优化方法在JDK1.5以后, 效果就不明显了);

### subString()方法的内存泄露
```java
public String substring(int beginIndex);
public String substring(int beginIndex, int endIndex);
// 第二种方法, 在JDK的实现中存在严重的内存泄露问题
public String substring(int beginIndex, int endIndex){
    if(beginIndex < 0){
        throw new StringIndexOutOfBoundsException(beginEndex);
    }
    if(endIndex > count){
        throw new StringIndexOutOfBoundsException(endIndex);
    }
    if(beginIndex > endIndex){
        throw new StringIndexOutOfBoundsException(endIndex - beginIndex);
    }
    // 问题出来新建String对象
    // 若偏移量offset很小, 但是原始的字符串很大
    // 就出现一个问题就是, 用的很少, 但是占用的空间很大.
    // 利用的是空间换时间的思想
    return ((beginIndex == 0) && （endIndex == count)) ? this : new String(offset + beginIndex, endIndex - beginIndex, value);
}

// String的构造函数
// package private constructor which shares value array for speed
String(int offset, int count, char value[]){
    this.value = value; // 直接复制原串
    this.offset = offset;
    this.count = count;
}
```
* 使用一个例子说明这个方法的弊端
```java
public static void main(String args[]){
    List<String> handler = new ArraryList<String>();
    // HugeStr 不到1000次就会内存溢出
    // 但是ImproveHuge不会
    for(int i = 0; i < 1000; i ++){
        HugeStr h = new HugeStr();
        ImproveHugeStr h = new ImproveHugeStr();
        handler.add(h.getSubString(1, 5));
    }
}

static class HugeStr{
    private String str = new String(new char[100000]);
    // 截取字符串, 有溢出
    public String getSubString(int begin, int end){
        return str.substring(begin, end);
    }
}

static class ImproveHugeStr{
    private String str = new String(new char[100000]);
    // 新的字符串, 没有溢出
    public String getSubString(int begin, int end){
        return new String(str.substring(begin, end));
    }
}
// 而且所有引用String(int offset, int count, char value[])构造函数的方法都存在一样的内存泄露.
toString(int);
toString(long);
concat(String);
replace(char, char);
substring(int, int);
toLowerCase(Locale);
toUpperCase(Locale);
valueOf(char);
```

### 字符串分割和查找
> split很强大, 但是对于简单的字符串分割功能而言, 它却不尽人意.
```java
// 第一种方法:
// split分割, 10000次, 3703ms
String orgStr = null;
StringBuffer sb = new StringBuffer();
for(int i = 0; i < 1000; i ++){
    sb.append(i);
    sb.append(";");
}
orgStr = sb.toString();
for(int i = 0; i < 10000; i ++){
    orgStr.split(";");
}

// 第二种方法:
//StringTokenizer类 构造函数
// delim是分隔符, s
public StringTokenizer(String str, String delim);
// 2704ms
StringTokenizer st = new StringTokenizer(orgStr, ";");
for(int i = 0; i < 10000; i ++){
    while(st.hasMoreTokens()){
        st.nextToken();
    }
    st = new StringTokenizer(ortStr, ";");
}

// 第三种方法: 671ms
// 说明indexOf 和 subString函数可以作为高频函数使用.
// 使用indexOf 和 subString() 注意内存泄露即可
public int indexOf(int ch); // 返回字符在String对象中的位置
String tmp = orgStr;
for(int i = 0; i < 10000; i ++){
    while(true){
        String splitStr = null;
        int j = tmp.indexOf(';'); // 找分隔符的位置
        if(j < 0) break;
        splitStr = tmp.substring(0, j);
        tep = tmp.substring(j + 1);
    }
    tem = orgStr;
}

// indexOf 功能相反, 但是同样高效的函数
public char charAt(int index);
// 通常我们会判断一个字符串的开始和结束是否等于某个子串
// 效率要比charAt要低很多
public boolean startsWith(String prefix);
public boolean endsWith(String suffix);
// charAt 每个比较要比 startWith 和 endsWith好
```

### StringBuffer 和 StringBuilder
> 因为String不可变对象, 若改变则会生成新的对象. 以上是对String创建和修改字符串的工具.
```java
// 方法一: 会生成多个String对象(编译器优化有只有一个) 0ms(5万次)
// 字符串常量静态的连接, 编译器会充分优化, 编译时就可以确定字符串操作.
String result = "String:" + "and" + "String" + "append";
// 方法二: 只生成一个result实例  16ms(5万次)
StringBuilder result = new StringBuilder();
result.append("String");
result.append("and");
result.append("String");
result.append("append");
// 方法三: 16ms(5万次)
String str1 = "String";
String str2 = "and";
String str3 = "String";
String str4 = "append";
// 这个java编译器会使用StringBuilder优化
// String s = (new StringBuilder(String.valueOf(str1))).append(str2).append(str3).append(str4).toString();
String result = str1 + str2 + str3 + str4;

// 情况三: 构建超大的String对象
// 编译器虽然会优化, 但是请显示使用StringBuilder 或者 StringBuffer
// A: 1062ms
for(int i = 0; i < 10000; i ++){
    str = str + i;
}
// A的反编译代码
for(int i = 0; i < CIRCLE; i ++){
    // 虽然是用来StringBuilder实现, 但是每次循环都会生成新的StringBuilder实例, 所以降低系统性能.
    // 而C只有一个StringBuilder实例
    str = (new StringBuilder(String.valueOf(str))).append(i).toString();
}
// B: 360ms
// 说明concat比+效率高, 但是要低于StringBuilder
for(int i = 0; i < 10000; i ++){
    result = result.concat(String.valueOf(i));
}
// C: 0ms
StringBuilder sb = new StringBuilder();
for(int i = 0; i < 10000; i ++){
    sb.append(i);
}
```

#### StringBuilder和StringBuffer的选择
* StringBuilder和StringBuffer 都是实现了AbstractStringBuilder抽象类; 区别在于StringBuffer对几乎所有的方法都做了同步(可用于多线程, StringBuilder不行)
* 非同步, StringBuilder要好一点.
  
#### StringBuilder和StringBuffer的容量参数
* 不指定容量参数, 默认是16个字节  
* 如果需要的容量超过实际的char数组长度, 会进行扩容, 直接扩大两倍. 若事先知道容量大小, 就可以避免扩容操作(提高系统性能).
* 实现开的容量大小合适 46ms, 相反 125 ms. 差点3倍, 所以要注意.


## 3.2 核心数据结构

### 3.2.1 List接口
* ArrayList, Vector, LinkedList(循环双向链表, 有header节点)都来自AbstratList的实现
* ArrayList, Vector都是对内部数组操作, 但是Vector是线程安全的, ArrayList不是.
* -Xmx512M -Xms512M 参数, 屏蔽GC对程序执行速度测试的干扰
* ArrayList扩容 是 扩大为 1.5倍.
* 不要对LinkedList进行 list.get(i)操作, 每次这个操作都会对它的列表遍历操作. for(int i = 0; i < size; i ++) tem = list.get(i); // 直接GG
* ForEach操作比迭代器迭代操作有多一部分的赋值操作, 所以ForEach操作比迭代器迭代操作稍微慢一点点.
  
### 3.2.2 Map接口
* Hashtable(线程安全), HashMap,  LinkedHashMap, TreeMap
![java程序性能优化_map的关系]()
* HashMap的内部结构: 主要依赖于hashCode()或者hash()方法的实现.
![java程序性能优化_HashMap内部结构]()

* LinkedHashMap 保存输入时的顺序 或者 最近访问的顺序; 迭代器中不可以修改元素, 否则迭代器无效; 当使用最近访问的顺序, 不可以使用get()方法, 因为它读完后会直接把该元素放到末尾(修改了).
* TreeMap: 性能没有HashMap好(低25%), 但是提供元素排序功能.


### 3.2.3 Set接口
* set不可以重复; 所有的知识对Map结构的封装
![java程序性能优化_Set类族结构]()

### 3.2.4 优化集合访问代码
* 1.分离循环中被重复调用的代码
```java
for(int i = 0; i < collection.size(); i ++); 
// int colsiz = collection.size() 代替 collection.size()
for(int i = 0; i < colsize; i ++)
```
* 2.省略相同的操作
```java
for(int i = 0; i < colsize; i ++){
    if( ((String)collection.get(i).indexOf("north") != -1) || (String)collection.get(i).indexOf("west") != -1) || (String)collection.get(i).indexOf("south") != -1)
    )
    count ++;
}
// 改为
int colsiz = collection.size()
for(int i = 0; i < colsize; i ++){
    if( (s = (String)collection.get(i)).indexOf("north") != -1) || s.indexOf("west") != -1) || s.indexOf("south") != -1)
    )
    count ++;
}
```
* 3.减少方法调用: 方法调用需要堆栈
```java
int colsiz = this.elementCount;
for(int i = 0; i < colsize; i ++){
    if( (s = (String) elementData[i]).indexOf("north") != -1) || s.indexOf("west") != -1) || s.indexOf("south") != -1)
    )
    count ++;
}
```

### 3.2.5 RandomAccess接口
* 实现RandomAccess接口的对象是支持快速随机访问的对象

## 3.3 使用NIO提升性能
> NIO: New I/O的简称, 基于块(Block)

> Channel是双向通道(而且可读可写), Stream单向
```java
// 读文件的例子:
FileInputStream fin = new FileInputStream(new File("d:\\temp_buffer.tmp"));
FileChannel fc = fin.getChannel();
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
fc.read(byteBuffer); // 存在ByteBuffer中, 可以关闭Channel了
fc.close();
byteBuffer.flip();
// 文件复制的例子:
public static void nioCopyFile(String resource, String destination) throws IOException{
    FileInputStream fis = new FileInputStream(resource);
    FileOutputStream fos = new FileOutputStream(destionation);
    FileChannel readChannel = fis.getChannel();
    FileChannel writeChannel = fos.getChannel();
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while(true){
        buffer.clear();
        int len = readChannel.read(buffer);
        if(len == -1) break;
        buffer.flip(); // 充值position
        writeChannel.write(buffer);
    }
    readChannle.close();
    writeChannle.close();
}
// position 写或者读的起点
// limit 写或者读的长度
// capacity 总的容量
```

#### 3.3.3 Buffer的相关操作
```java
// 堆中分配
ByteBuffer buffer = ByteBuffer.allocate(1024);
// 从即有数组中创建
byte array[] = new byte[1024];
ByteBuffer buffer = ByteBuffer.wrap(array);
// 重置和清空缓冲区, 只是重置标志而已. 
public final Buffer rewind();
public final Buffer clear();
public final Buffer flip(); 
// rewind()函数, position置为零, 清除标志位mark
out.write(buf);   // 从Buffer读取数据写入Channel
buf.rewind();     // 回滚Buffer
buf.get(array);   // 将Buffer的有效数据复制到数组中
// clear()函数, position置为零, limit设置为capacity大小, 清除mark标志, 
buf.clear();
in.read(buf);
// flip()函数将limit设置到position所在的位置, position置为零, 清除mark. 通常在读写转换时使用
buf.put(magic);   // magic数组读入buf中
in.read(buf);
buf.flip();       // 写状态转化为读状态
out.write(buf);

// 标志缓冲区
ByteBuffer b = ByteBuffer.allocate(15);
for(int i = 0; i < 10; i ++){
    b.put((byte)i);
}
b.flip();
for(int i = 0; i < b.limit(); i ++){
    System.out.print(b.get());
    if(i == 4){
        b.mark(); // 在mark位置做标记
        System.out.print("(mark at " + i + ")");
    }
}
b.reset(); // 回到mark位, 处理后面的数据
System.out.println("\nreset to mark");
while(b.hasRemaining()){
    System.out.print(b.get());
}
System.out.println();

// 复制缓冲区
public ByteBuffer duplicate();
// 两个共享内存, 一方改动另外一个可以看到
// 但是双方的position, limit, mark是不一样的

// 缓冲区分片
slice() // 子缓冲区

// 只读缓冲区

// 文件映射到内存: 更加快!

// 处理结构化数据: 散射 与 聚集

```

#### 3.3.5 直接内存访问
* DirectBuffer
... 中间省略了好几节, 不太想看了 > 以后再补


# 4. 并行程序开发以及优化
## 并行程序设计模式
> Future模式, Master-Worker模式, Guarded Suspeionsion模式, 不变模式, 生产者-消费者模式

### Future模式
> JDK支持的Future模式更加强大, 以下只是简陋的初步.
![java程序性能优化_Future模式]()
```java
// 一个是代理模式. 
// 另一个是使用了同步的思想.

// 1.Main函数的实现: 主要负责调用Client发起请求, 并使用返回的数据
public static void main(String[] args){
    Client client = new Client();
    // 这里会立即返回, 因为得到的是FutureData而不是RealData
    Data data = client.request("name");
    System.out.println("请求完毕");
    try{
        // 这里可以用一个sleep代替对其他的业务逻辑处理
        // 处理这些业务的过程中, RealData被创建, 充分利用等待时间 
        Thread.sleep(2000);
    }catch(InterruptedException e){}
    // 使用真实的数据
    System.out.println("数据 = " + data.getResult());
}

// Client的实现
// 主要实现了获取FutureData, 开启构造RealData的线程, 接受请求后很快的返回FutureData.
public class Client{
    public Data request(final String queryStr){
        final FutureData future = new FutureData();
        new Thread(){
            public void run(){
                // RealData的构建很慢
                // 所以在单独的线程中进行
                RealData realdata = new RealData(queryStr);
                future.setRealData(realdata);
            }
        }.start();
        return future; // FutureData会被立即返回.
    }
}

// Data的实现
// 提供了getResult()方法, 无论FutureData或者RealData都实现了这个接口.
public interface Data{
    public String getResult();
}

// FutureData实现
// 实际上就是RealData的代理, 封装了获取RealData的过程
public class FutureData implements Data{
    protected RealData realdata = null; // FutureData 是 RealData的包装
    protected boolean isReady = false;
    public synchronized void setRealData(RealData realdata){
        if(isReady) return;
        this.realdata = realdata;
        isReady = true;
        notifyAll(); // 通知getResult()
    }
    public synchronized String getResult(){
        // 等待RealData构造完成
        while(!isReady){
            try{
                wait(); // 一直等待, 知道RealData被注入
            } catch (InterruptedException e){
            }
        }
        return realdata.result; // 由RealData实现
    }
}

// RealData的实现
public class RealData implements Data{
    protected final String result;
    public RealData(String para){
        // RealData 的构造可能很慢, 需要用户等待很很久, 这里使用sleep模拟
        StringBuffer sb = new StringBuffer();
        for(int i = 0; i < 10; i ++){
            sb.append(para);
            try{
                // 这里sleep函数, 代替一个很慢的操作
                Thread.sleep(100);
            }catch{InterruptedException e}{
            }
        }
        result = sb.toString();
    }
    public String getResult(){
        return result;
    }
}
```

### Master-Worker模式
> Master负责接受和分配工作, Worker则做实际的运行
```java
public class Master{
    // 任务队列
    // ConcurrentLinkedQueue是线程安全的, 
    protected Queue<Object> workQueue = new ConcurrentLinkedQueue<Object>();
    // Worker进程队列
    protected Map<String, Thread> threadMap = new HashMap<String, Thread>();
    // 子任务处理结果集
    protected Map<String, Object> resultMap = new ConcurrentHashMap<String, Object>();

    // 是否所有的子任务都结束了
    public boolean isComplete(){
        for(Map.Entry<String, Thread> entry : threadMap.entrySet()){
            if(entry.getValue().getState() != Thread.State.TERMINATED) return false;
        }
        return true;
    }

    // Master的构造, 需要一个Worker进程逻辑, 和需要的Worker进程数量
    public Master(Worker worker, int countWorker){
        worker.setWorkQueue(workQueue);
        worker.setResultMap(resultMap);
        for(int i = 0; i < countWorker; i ++){
            threadMap.put(Integer.toString(i), new Thread(worker, Integer.toString(i)));
        }
    }

    // 提交一个任务
    public void submit(Object job){
        workQueue.add(job);
    }

    // 返回子任务结果集
    public Map<String, Object> getResultMap(){
        return resultMap;
    }

    // 开始运行所有得儿Worker进程
    public void execute(){
        for(Map.Entry<String, Thread> entry:threadMap.entrySet()){
            entry.getValue().start();
        }
    }
}


// 对应的Worker进程
public class Worker implements Runnable{
    // 任务队列, 用于取得子任务
    protected Queue<Object> workQueue;
    // 子任务处理结果集合
    protected Map<String, Object> resultMap;
    public void setWorkQueue(Queue<Object> workQueue){
        this.workQueue = workQueue;
    }
    public void setResultMap(Map<String, Object> resultMap){
        this.resultMap = resultMap;
    }
    // 子任务处理的逻辑, 在子类中实现具体逻辑
    public Object handle(Object input){
        return input;
    }
    @Override
    public void run(){
        while(true){
            // 获取子任务
            Object input = workQueue.poll();
            if(inpuu == null) break;
            // 处理子任务
            Object re = handle(input);
            // 处理结果写入结果集
            resultMap.put(Integer.toString(input.hashCode()), re);    
        }
    }
}
```

### Guarded Suspension 模式


## JDK多任务执行框架