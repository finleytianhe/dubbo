高性能 开源rpc

基于rpc的高性能接口，对于用户是透明的
智能负载均衡,可以感知下游服务状态，减少整体延迟，提高系统吞吐量
自动服务注册与发现
高扩展性，插件性设计
运行时流量路由，运行时配置，流量可以根据不同的规则进行路由，支持蓝绿部署，数据中心感知路由
可视化服务化治理，查询服务元数据，健康状态和统计信息
故障转移，容错

monitor 监视器统计服务调用的次数和耗时

可伸缩，自动服务注册与发现




参数验证
结果缓存
异步调用
本地调用
回调参数
事件通知
本地存根
本地mock
延迟发布
延迟连接
粘性连接
token验证
路由规则


注册中心
zk
redis
广播注册
简单注册

协议
dubbo
rmi
hession

网络通讯
netty
mina
grizzly

序列化
hession
dubbo
json
java


代理
javassist
jdk


容错
Failover 故障自动切换，当故障发生时，重试其他服务器，通常用于读操作。(推荐) 重试将导致更长的延迟
Failfast 快速失败，只需一次调用，失败立即报告，通常用于非幂等写。如果服务器正在重新启动，可能会导致呼叫失败
Failsafe 故障安全，当出现异常时，直接忽略，通常用于写入审计日志和其他操作 调用信息丢失
Failback 故障自动恢复，后台记录故障请求，定期重传，通常用于消息通知操作 不可靠，重启服务器时丢失
Forking 只要返回一个成功，就会并行调用多个服务器，通常用于高实时的读取操作。需要浪费更多的服务资源
Broadcast 广播一个接一个地调用所有提供者，并且错误地报告任何错误，通常用于更新提供者的本地状态 速度慢，任何虚假的报道都是错误的。

负载均衡
随机 随机概率，根据权重设置随机概率(推荐) 在一个截面上发生碰撞的概率很高。当重新尝试时，可能会有不相等的瞬时压力。
轮询 轮循时，按设定轮后的重量比例约定 有一个缓慢的机器积累请求问题，和极端情况下可能导致雪崩
最小活跃数 最少活动的呼叫号码，同一活动号码的随机号码，活动号码是呼叫前后的计数差，使慢机收到的请求更少。不支持重量，在容量规划中，不以压力机为导向，按重量测量压力容量
一致性hash 一致性散列，相同的参数总是请求同一个提供者，当一个提供者挂起时，最初发送给该提供者的请求，基于虚拟节点，传播给其他提供者，不会引起剧烈的变化 压力分布不均匀

条件路由 路由规则基于条件表达式，简单易用 有一些复杂的多分支条件，规则难以描述
script路由 基于路由规则的脚本引擎，功能强大 没有沙盒在运行，脚本能力太强大，可能是后门

以超时为例，下面是优先级，从高到低(重试、loadbalance、活动也适用相同的规则):

方法级别、接口级别、默认/全局级别。
在同一级别上，使用者比提供者具有更高的优先级
提供者端配置以URL的形式通过registry传递给使用者端。

配置覆盖和优先级
-D
xml
properties

指定调用
reference.setUrl("dubbo://10.20.130.230:20880/com.xxx.XxxService");

Dubbo支持多级配置，并根据预先确定的优先级自动覆盖配置。最终，将所有配置聚合到数据总线URL，以驱动后续的服务公开、引用和其他流程。
ApplicationConfig、ServiceConfig和ReferenceConfig可以被视为配置源，它们通过直接面向用户的编程收集配置。

配置优先级
-D
外部配置
ServiceConfig, ReferenceConfig配置
本地 dubbo.properties

check=true启动检查服务是否可用，服务不可用得到一个空引用
check=false时不检查，循环依赖这种必须先启动起来，得到一个引用，调用时服务可用会自动重连
<dubbo:reference interface = "com.foo.BarService" check = "false" />
<dubbo:consumer check = "false" />
<dubbo:registry check="false" />

执行流程
容错-router script和条件路由-load balance负载均衡-invoker执行

路由器负责根据来自多个调用者的路由规则选择子集，如读写分离、应用隔离等。

故障自动切换，当有故障时，重试另一个服务器(默认)。通常用于读操作，但是重试可能会导致更长的延迟。
重试次数可以通过重试次数=2 retries="2"来设置(不包括第一次)。

cluster
failfast 快速失败，调用一次，失败立即出错。通常用于非幂等的写操作，例如添加记录
failsafe 安全失败，异常，直接忽略。通常用于写审计日志和其他操作。
Failback 故障自动恢复，无法记录后台请求，定期重传。通常用于消息通知操作。
Forking 同时调用多个服务器，一旦其中一个成功就返回。通常用于实时性要求高的读操作，但会浪费更多的业务资源。当fork =2时，可以设置最大并行数。
Broadcast 广播调用所有的提供者，一个接一个地调用，任何错误都会被报告(2.1.0+)。它通常用于通知所有提供者更新本地资源信息，如缓存或日志。

<dubbo:service cluster="failsafe" />
<dubbo:reference cluster="failsafe" />

Load Balance

Ramdom
Dubbo为集群负载均衡提供了许多均衡策略，默认情况下是random。
随机，按权重设定随机概率。
在一个区域发生碰撞的概率很高，但是调用的数量越大，分布就越均匀。当使用基于概率的权重时，分布是均匀的，这也有助于动态调整提供者的权重。

RoundRobin 使用权重的common advisor来确定轮询比例。
流向较慢的提供商的流量可能会导致请求堆积，例如，如果有一个提供商以非常慢的速度处理请求，但它仍然活着，这意味着它可以正常接收请求。根据RoundRobin策略，使用者将不断地以预定的速度向该提供者发送请求，而不知道提供者的糟糕状态。最后，我们会有很多请求被卡在这个不健康的提供者上。

LeastActive 一个基于active的随机机制，active意味着消费者已经发送但还没有返回的请求数量。
较慢的提供者将收到较少的请求，因为较慢的提供者有较高的活动。

ConsistentHash 请求的相同参数总是发送到相同的提供程序。
当一个提供者失败时，根据虚拟节点算法对提供者的原始请求向其他提供者取平均值，不会引起剧烈的变化。默认只有第一个参数哈希，如果要修改，请配置<dubbo:parameter key="hash.arguments" value="0,1" />
默认160个虚拟节点，如果需要修改，请配置 <dubbo:parameter key="hash.nodes" value="320" />

<dubbo:service interface="..." loadbalance="roundrobin" />
<dubbo:reference interface="..." loadbalance="roundrobin" />
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:reference>

在dubbo中配置线程池模型
如果事件处理可以在不发送新请求(如在内存中标记)的情况下快速执行。事件应该由I/O线程处理，因为它减少了线程调度。
如果事件处理执行缓慢，或者需要发送新的I/O请求(如从数据库查询)，则应该在线程池中处理事件。否则，I/O线程将被阻塞，然后将无法接收请求。
如果事件是由I/O线程处理的，并且在处理过程中发送新的I/O请求，比如在connect事件中发送一个l登录请求，它将警告“可能导致死锁”，但死锁实际上不会发生。
因此，我们需要不同的调度策略和不同的线程池配置来面对不同的场景。
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />

Dispatcher
all 所有消息将被分派到线程池，包括请求、响应、连接事件、断开连接事件和心跳。
direct:所有消息都不会被分派到线程池，而是由I/O线程直接执行。
message:只有请求、响应消息将被分派到I/O线程。其他消息如disconnect、connect、heartbeat消息将由I/O线程执行。
execution:只有请求消息将被分派到线程池。其他消息如response、connect、disconnect、heartbeat将由I/O线程直接执行。
connection:I/O线程将在队列中放置断开和连接事件并按顺序执行，其他消息将被分派到线程池。

Thread pool
fixed 线程池的固定大小。它在启动时创建线程，永远不会关闭(默认)。
cached:缓存的线程池。当线程空闲一分钟时自动删除线程。在需要的时候重新创建。
limit:弹性线程池。但是它只能增加线程池的大小。其原因是为了避免在减少线程池大小时由于流量峰值而导致的性能问题。

在开发和测试环境中，通常有必要绕过注册中心，只测试指定的服务提供者。在这种情况下，可能需要点对点的直接连接，服务提供者将忽略提供者注册提供者的列表。接口A配置点对点，不影响接口B从注册表中获取列表。

如果是在线需求需要点对点功能，可以将指定的提供者url配置为<dubbo:reference>。它将绕过注册表，多个地址以分号分隔，如下配置:
<dubbo:reference id="xxxService" interface="com.alibaba.xxx.XxxService" url="dubbo://localhost:20890" />

然后在映射文件xxx中添加配置。属性，其中键是服务名称，值是服务提供者URL:
com.alibaba.xxx.XxxService=dubbo://localhost:20890

为了避免使在线环境复杂化，不要在线使用这个特性，应该只在测试阶段使用

配置只在dubbo订阅
为了方便测试的开发，在开发环境中拥有所有可用服务的注册中心是很常见的。正在开发中的服务提供商的注册可能会影响消费者无法运行。
您可以让服务提供者开发人员只订阅服务(所开发的服务可能依赖于其他服务)，不注册正在开发的服务，并使用直接连接测试正在开发的服务。
<dubbo:registry address="10.20.153.10:9090" register="false" />
<dubbo:registry address="10.20.153.10:9090?register=false" />

在dubbo中配置多个协议
Dubbo允许您配置多个协议，支持不同服务上的不同协议，或支持同一服务上的多个协议。每个服务分别导出到一个特定的协议
不同的协议性能是不一样的。如大数据应采用短连接协议，小数据和并发应采用长连接协议。

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <dubbo:registry id="registry" address="10.20.141.150:9090" username="admin" password="hello1234" />
    <!-- multiple protocols -->
    <dubbo:protocol name="dubbo" port="20880" />
    <dubbo:protocol name="rmi" port="1099" />
    <!-- Use dubbo protocol to expose the service -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" protocol="dubbo" />
    <!-- Use rmi protocol to expose services -->
    <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" protocol="rmi" />
</beans>

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <dubbo:registry id="registry" address="10.20.141.150:9090" username="admin" password="hello1234" />
    <!-- multiple protocols-->
    <dubbo:protocol name="dubbo" port="20880" />
    <dubbo:protocol name="hessian" port="8080" />
    <!-- Service exposes multiple protocols -->
    <dubbo:service id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" protocol="dubbo,hessian" />
</beans>

在dubbo中配置多个注册表
Dubbo支持用相同的服务注册多个注册中心，或者不同的服务被注册到不同的注册中心，甚至从不同的注册中心引用相同的名称服务。此外，注册中心还支持自定义扩展1。

一个服务注册到多个注册
例如:阿里巴巴有些服务没有部署在青岛，只部署在杭州。当青岛的其他应用需要引用此服务时，您可以同时向两个注册中心注册您的服务。

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <!-- Multi registries -->
    <dubbo:registry id="hangzhouRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="qingdaoRegistry" address="10.20.141.151:9010" default="false" />
    <!-- Service register to multiple registries -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="hangzhouRegistry,qingdaoRegistry" />
</beans>

不同的服务注册到不同的注册中心
例如:有些CRM服务是专门为国际站设计的，有些服务是专门为中国站设计的。
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <!-- Multi registries -->
    <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
    <!-- Service register to Chinese station registry -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="chinaRegistry" />
    <!-- Service register to international station registry -->
    <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" registry="intlRegistry" />
</beans>

从多个注册中心引用服务
例如:CRM需要同时呼叫中国站和国际站的PC2业务。PC2部署在中国站和国际站。接口和版本号相同，只是使用的数据库不同。
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <!-- Multi registries -->
    <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
    <!-- Reference Chinese station service -->
    <dubbo:reference id="chinaHelloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" registry="chinaRegistry" />
    <!-- Reference international station service -->
    <dubbo:reference id="intlHelloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" registry="intlRegistry" />
</beans>

测试时，服务需要临时注册到两个注册中心，这可以使用垂直符号来分隔多个不同的注册中心地址:
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <!-- The vertical separation means that multiple registries are connected at the same time. Multiple cluster addresses of the same registry are separated by commas -->
    <dubbo:registry address="10.20.141.150:9090|10.20.154.177:9010" />
    <!-- service reference -->
    <dubbo:reference id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" />
</beans>

dubbo的分组服务
当一个接口有多个impls时，可以用group来区分。
<dubbo:service group="feedback" interface="com.xxx.IndexService" />
<dubbo:service group="member" interface="com.xxx.IndexService" />

<dubbo:reference id="feedbackIndexService" group="feedback" interface="com.xxx.IndexService" />
<dubbo:reference id="memberIndexService" group="member" interface="com.xxx.IndewxService" />

任何一组
<dubbo:reference id="barService" interface="com.foo.BarService" group="*" />

在dubbo配置静态服务
有时我们需要手动管理服务提供者的注册和注销，我们需要将注册表设置为非动态模式。
<dubbo:registry address="10.20.141.150:9090" dynamic="false" />
<dubbo:registry address="10.20.141.150:9090?dynamic=false" />
动态模式在服务提供者初始注册时被禁用，我们需要手动启用它。当断开连接时，设置不会自动删除，需要手动禁用。
对于像“memcachd”这样的第三方服务提供者，它可以直接将服务提供者的地址信息写入registry，供使用者使用。

在dubbo中为服务配置多个版本
当某个接口实现不兼容升级时，可以使用版本号转换。不同版本的服务不相互引用。

您可以按照以下步骤进行版本迁移:

在低压期，将一半的供应商升级到新版本
然后将所有消费者升级到新版本
然后将剩下的一半提供程序升级到新版本
旧版本的服务提供者配置:
<dubbo:service interface="com.foo.BarService" version="1.0.0" />
<dubbo:service interface="com.foo.BarService" version="2.0.0" />
<dubbo:reference id="barService" interface="com.foo.BarService" version="2.0.0" />
<dubbo:reference id="barService" interface="com.foo.BarService" version="*" />

在dubbo的group合并
根据group来调用服务器和返回合并结果1,如菜单服务,相同的接口,但也有不同的实现,使用组区别,消费者打电话给每组,得到结果,合并可以合并的成果,这样你就可以实现聚合菜单项。

合并所有组
<dubbo:reference interface="com.xxx.MenuService" group="*" merger="true" />

合并指定的组
<dubbo:reference interface="com.xxx.MenuService" group="aaa,bbb" merger="true" />

用于合并结果的指定方法，以及其他未指定的方法，将只调用一个组
<dubbo:reference interface="com.xxx.MenuService" group="*">
    <dubbo:method name="getMenuItems" merger="true" />
</dubbo:reference>

指定的a方法不合并结果，其他方法合并结果
<dubbo:reference interface="com.xxx.MenuService" group="*" merger="true">
    <dubbo:method name="getMenuItems" merger="false" />
</dubbo:reference>

指定合并策略时，默认根据类型返回值自动匹配，如果同一类型有两个合并，则需要指定名称为merger2
<dubbo:reference interface="com.xxx.MenuService" group="*">
    <dubbo:method name="getMenuItems" merger="mymerge" />
</dubbo:reference>

指定归并方法，它将调用返回类型的方法进行归并，归并方法的参数类型必须是返回类型
<dubbo:reference interface="com.xxx.MenuService" group="*">
    <dubbo:method name="getMenuItems" merger=".addAll" />
</dubbo:reference>

dubbo中的参数验证
参数验证1基于[JSR303]，用户只需添加JSR303的验证注释，并声明验证2的过滤器。

<dubbo:reference id="validationService" interface="org.apache.dubbo.examples.validation.api.ValidationService" validation="true" />
<dubbo:service interface="org.apache.dubbo.examples.validation.api.ValidationService" ref="validationService" validation="true" />

在dubbo缓存结果
缓存结果用于加速对流行数据的访问。Dubbo提供了声明式缓存，以减少用户添加缓存1的工作。
lru 根据最近最少使用的原则删除多余的缓存。缓存的是最热的数据。
threadlocal 当前线程缓存。例如，一个页面有很多门户，每个门户都需要检查用户信息，您可以使用此缓存减少这种冗余访问。
jcache 与JSR107集成后，您可以桥接各种缓存实现。
<dubbo:reference interface="com.foo.BarService" cache="lru" />
<dubbo:reference interface="com.foo.BarService">
    <dubbo:method name="findBar" cache="lru" />
</dubbo:reference>

dubbo中的泛型引用
一般调用主要用于客户端没有API接口或模型类时，参数和返回值中的所有pojo都用Map表示。通常用于框架集成，例如:实现公共服务测试框架，所有服务实现都可以通过GenericService调用。
<dubbo:reference id="barService" interface="com.foo.BarService" generic="true" />

GenericService barService = (GenericService) applicationContext.getBean("barService");
Object result = barService.$invoke("sayHello", new String[] { "java.lang.String" }, new Object[] { "World" });

dubbo的通用服务
通用接口的实现主要用于服务器端没有API接口和模型类的情况。参数和返回值中的所有pojo都由映射表示，通常用于框架集成。例如，要实现通用远程服务模拟框架，需要通过实现GenericService接口来处理所有服务请求。
<bean id="genericService" class="com.foo.MyGenericService" />
<dubbo:service interface="com.foo.BarService" ref="genericService" />

dubbo上下文
当前调用期间的所有环境信息将放入上下文，所有配置信息将URL实例的参数、Ref转换为模式配置参考书中的URL参数列
RpcContext是ThreadLocal的一个临时状态记录器，当接受RPC请求或发送RPC请求时，RpcContext将被更改。例如:A呼叫B和B呼叫C。在B机上，B呼叫C之前，RpcContext会记录A呼叫B的信息。B呼叫C之后，RpcContext记录B呼叫C的信息。

在使用者和提供者之间传递隐式参数
您可以通过RpcContext上的setAttachment和getAttachment隐式地在服务消费者和提供者之间传递参数。
在服务使用者端设置隐式参数
通过RpcContext上的setAttachment设置隐式传递参数的键/值对。当完成一次远程调用时，将被清除，因此多调用必须设置多次。
RpcContext.getContext().setAttachment("index", "1"); // implicitly pass parameters,behind the remote call will implicitly send these parameters to the server side, similar to the cookie, for the framework of integration, not recommended for regular business use
xxxService.xxx(); // remote call
在服务提供者端获取隐式参数
public class XxxServiceImpl implements XxxService {

    public void xxx() {
        // get parameters which passed by the consumer side,for the framework of integration, not recommended for regular business use
        String index = RpcContext.getContext().getAttachment("index");
    }
}

dubbo中的异步调用
由于dubbo基于非阻塞NIO网络层，客户端可以启动对多个远程服务的并行调用，而无需显式启动多线程，这样消耗的资源相对较少。
您可以在consumer.xml中配置，用于设置异步调用某些远程服务。
<dubbo:reference id="fooService" interface="com.alibaba.foo.FooService">
      <dubbo:method name="findFoo" async="true" />
</dubbo:reference>
<dubbo:reference id="barService" interface="com.alibaba.bar.BarService">
      <dubbo:method name="findBar" async="true" />
</dubbo:reference>

配置上述配置信息，您就可以在代码中调用远程服务。

您也可以设置是否等待消息发送:

send ="true" 等待消息发送，如果发送失败，将抛出异常。
sent="false" 不要等待消息被发送，当消息将推入io队列时，将立即返回。
<dubbo:method name="findFoo" async="true" sent="true" />

如果您只想异步调用，而不关心返回。您可以配置return="false"，以减少创建和管理Future对象的成本。
<dubbo:method name="findFoo" async="true" return="false" />

在dubbo提供者端异步执行
provider上的异步执行将阻塞的服务从Dubbo的内部线程池切换到服务自定义线程，以避免Dubbo线程池的过度占用，这有助于避免不同服务之间的相互影响。
异步执行不利于节省资源或改善RPC响应性，因为如果业务执行需要被阻塞，那么总是有一个线程负责执行。
提供者上的异步执行和使用者上的异步执行是相互独立的。你可以配置任意正交组合的端点。

在使用者上同步执行——在提供者上同步执行
使用者上的异步执行——提供者上的同步执行
在使用者上同步执行——在提供者上异步执行
在使用者上的异步执行——在提供者上的异步执行

接口，该接口定义CompletableFuture签名
public interface AsyncService {
    CompletableFuture<String> sayHello(String name);
}

public class AsyncServiceImpl implements AsyncService {
    @Override
    public CompletableFuture<String> sayHello(String name) {
        RpcContext savedContext = RpcContext.getContext();
        // It is recommended to provide a custom thread pool for supplyAsync to avoid using the JDK common thread pool.
        return CompletableFuture.supplyAsync(() -> {
            System.out.println(savedContext.getAttachment("consumer-key1"));
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "async response from provider.";
        });
    }
}

通过return CompletableFuture.supplyAsync()，业务执行已经从Dubbo线程切换到业务线程，避免了阻塞Dubbo线程池。此处也会有上下文切换的开销和netty handler中开启线程池执行任务一样

使用AsyncContext
Dubbo提供了一个类似于Serverlet 3.0的异步接口AsyncContext。它还可以在没有CompletableFuture签名接口的情况下实现提供程序的异步执行。

public interface AsyncService {
    String sayHello(String name);
}

服务导出，与普通服务完全一样:
<bean id="asyncService" class="org.apache.dubbo.samples.governance.impl.AsyncServiceImpl"/>
<dubbo:service interface="org.apache.dubbo.samples.governance.api.AsyncService" ref="asyncService"/>

public class AsyncServiceImpl implements AsyncService {
    public String sayHello(String name) {
        final AsyncContext asyncContext = RpcContext.startAsync();
        new Thread(() -> {
            // If you want to use context, you must do it at the very beginning
            asyncContext.signalContextSwitch();
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // Write to response
            asyncContext.write("Hello " + name + ", response from provider.");
        }).start();
        return null;
    }
}

dubbo的本地调用
本地调用使用injvm协议，这是一种伪协议，不打开端口，不发起远程调用，直接与JVM关联，但执行Dubbo过滤器链。

配置injvm协议
<dubbo:protocol name="injvm" />

配置默认提供者
<dubbo:provider protocol="injvm" />

配置默认服务
<dubbo:service protocol="injvm" />

先用injvm
<dubbo:consumer injvm="true" .../>
<dubbo:provider injvm="true" .../>

<dubbo:reference injvm="true" .../>
<dubbo:service injvm="true" .../>

注意:默认情况下，Dubbo服务从2.2.0在本地公开。它可以在本地引用而不需要任何配置。如果你不希望服务被远程公开，你只需要在提供程序中将协议设置为injvm。

自动公开的本地服务引用

2.2.0或更高版本，每个服务默认在本地公开。当引用服务时，默认引用本地服务。如果要引用远程服务，可以使用以下配置强制引用远程服务。
<dubbo:reference ... scope="remote" />

dubbo中的回调参数
callback参数与调用本地回调或监听器相同，只需在Spring的配置文件中声明哪个参数是回调类型，Dubbo将基于长连接生成反向代理，以便可以从服务器调用客户端逻辑。
<bean id="callbackService" class="com.callback.impl.CallbackServiceImpl" />
<dubbo:service interface="com.callback.CallbackService" ref="callbackService" connections="1" callbacks="1000">
    <dubbo:method name="addListener">
        <dubbo:argument index="1" callback="true" />
        <!--also can via specified argument type-->
        <!--<dubbo:argument type="com.demo.CallbackListener" callback="true" />-->
    </dubbo:method>
</dubbo:service>

事件通知在dubbo
调用前，调用后，当发生异常时，将触发oninvoke, onreturn, onthrow事件。您可以配置在事件发生时通知哪个方法。
<dubbo:application name="rpc-callback-demo" />
<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
<bean id="demoService" class="org.apache.dubbo.callback.implicit.NormalDemoService" />
<dubbo:service interface="org.apache.dubbo.callback.implicit.IDemoService" ref="demoService" version="1.0.0" group="cn"/>

<bean id ="demoCallback" class = "org.apache.dubbo.callback.implicit.NotifyImpl" />
<dubbo:reference id="demoService" interface="org.apache.dubbo.callback.implicit.IDemoService" version="1.0.0" group="cn" >
      <dubbo:method name="get" async="true" onreturn = "demoCallback.onreturn" onthrow="demoCallback.onthrow" />
</dubbo:reference>

回调函数和异步函数是正交分解的。async = true表示立即返回结果。onreturn意味着回调是必需的。

使用这两个属性[^2]的情况有几种。

异步回调模式:async=true onreturn="xxx"
同步回调模式:async=false onreturn="xxx"
异步回调:异步= true
同步回调:异步= true

dubbo的本地存根
当使用rpc时，客户机通常只提供接口，但有时客户机还希望在客户机中执行部分逻辑。例如:执行ThreadLocal缓存，验证参数，在调用失败时返回模拟数据。等。
要解决这个问题，您可以在API中配置存根，这样当客户端生成代理实例时，它就会通过构造函数1将代理传递给存根，然后您就可以在存根实现代码中实现逻辑。

在spring配置文件中配置如下:
<dubbo:service interface="com.foo.BarService" stub="true" />
<dubbo:service interface="com.foo.BarService" stub="com.foo.BarServiceStub" />

存根必须有一个可以传入代理的构造函数。↩︎
BarServiceStub实现了BarService，它有一个传递到远程BarService实例(︎)的构造函数

dubbo中本地mock
本地mock 1通常用于服务降级，例如验证服务，当服务提供者始终挂起时，客户机不会抛出异常，而是通过模拟数据返回失败的授权。

在spring配置文件中配置如下:
<dubbo:reference interface="com.foo.BarService" mock="true" />
<dubbo:reference interface="com.foo.BarService" mock="com.foo.BarServiceMock" />
考虑更改为模拟实现并在模拟实现中返回null。如果你只是想简单地忽略这个异常，2.0.11版本或更高版本是可用的:
<dubbo:reference interface="com.foo.BarService" mock="return null" />

高级用法

return
return可用于将对象的字符串表示形式作为模拟的返回值返回。法律价值包括:

空:空值，主类型为默认值，集合为空值。
空:零
真:真
假:假
JSON格式:JSON格式的模拟返回值，将在运行时反序列化

throw
throw可用于抛出异常对象作为模拟的返回值。

当调用出错时，抛出默认的RPCException:
<dubbo:reference interface="com.foo.BarService" mock="throw" />
<dubbo:reference interface="com.foo.BarService" mock="throw com.foo.MockException" />

从2.6.6及以上版本开始，可以在Spring的XML配置中使用fail:和force:来定义模拟行为。
force:表示无论调用是否错误，都强制使用所模拟的值，实际上，远程调用根本不会发生。fail:与默认行为一致，也就是说，mock只在调用出错时发生。
此外，force:和fail:都可以与throw或return一起使用，以进一步定义模拟行为。

强制返回指定的值:
<dubbo:reference interface="com.foo.BarService" mock="force:return fake" />

强制抛出指定的异常:
<dubbo:reference interface="com.foo.BarService" mock="force:throw com.foo.MockException" />

仅为特定方法指定Mock
模拟行为可以在方法级别上指定。假设com.foo上有几个方法。我们可以只为一个特定的方法指定模拟行为，比如sayHello()。在下面的例子中，每次调用sayHello()时都强制返回" fake "，但不会影响其他方法:

<dubbo:reference id="demoService" check="false" interface="com.foo.BarService">
    <dubbo:parameter key="sayHello.mock" value="force:return fake"/>
</dubbo:reference>

Mock是存根的一个子集。如果使用存根，则可能需要依赖RpcException类。如果使用Mock，则不需要依赖RpcException，当抛出RpcException时，它将回调模拟实现类。↩︎

BarServiceMock实现了BarService，并有一个无参数构造函数。↩︎

延迟发布dubbo服务
如果您的服务需要时间预热，例如:初始化缓存或其他参考资源必须准备好。您可以使用延迟特性来延迟发布服务。
我们在Dubbo 2.6.5中微调了服务延迟暴露逻辑，将需要延迟暴露的服务倒计时延迟到Spring初始化完成。你在使用Dubbo时不会知道这个变化，所以请放心使用。

延迟5秒发布
<dubbo:service delay="5000" />

延迟到Spring初始化完成后才公开服务
<dubbo:service delay="-1" />

所有服务都将在Spring初始化完成后公开，如果不需要延迟公开服务，则不需要配置延迟。
延迟5秒发布
<dubbo:service delay="5000" />

dubbo中的并发控制

配置的例子
示例1:控制服务器端指定服务接口的所有方法的并发性
限制com.foo的每个方法。不超过10个并发的服务器端执行(或占用线程池线程):
<dubbo:service interface="com.foo.BarService" executes="10" />

例2:控制服务器端指定服务接口的指定方法的并发性
限制com.foo的sayHello方法。不超过10个并发的服务器端执行(或占用线程池线程):

<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" executes="10" />
</dubbo:service>

例3:在客户端控制指定服务接口的所有方法的并发性。不超过10个并发客户端执行(或占用线程池线程):
<dubbo:service interface="com.foo.BarService" actives="10" />
<dubbo:reference interface="com.foo.BarService" actives="10" />

例4:在客户端控制指定服务接口的指定方法的并发性。不超过10个并发客户端执行(或占用线程池线程):
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>

<dubbo:reference interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>

如果<dubbo:service>和<dubbo:reference>都配置了active，则优先使用<dubbo:reference>。引用到:配置覆盖策略。

Load Balance
您可以在服务器端或客户端使用leastactive配置loadbalance属性，然后框架将使消费者调用并发的最少数量。
<dubbo:reference interface="com.foo.BarService" loadbalance="leastactive" />
<dubbo:service interface="com.foo.BarService" loadbalance="leastactive" />

在dubbo中配置连接

控制服务器端连接
限制服务器端接受不超过10个连接
<dubbo:provider protocol="dubbo" accepts="10" />
<dubbo:protocol name="dubbo" accepts="10" />

在客户端控制连接
限制客户端为com.foo.BarService接口创建的连接不超过10个。
<dubbo:reference interface="com.foo.BarService" connections="10" />
<dubbo:service interface="com.foo.BarService" connections="10" />

如果使用默认协议(dubbo协议)，并且connections属性的值大于0，则每个服务引用将拥有自己的连接，否则属于同一个远程服务器的所有服务将只共享一个连接。在这个框架中，我们称之为私有连接或共享连接。

dubbo中的Lazy connect
懒惰连接可以减少保持连接的数量。当一个呼叫被发起时，创建一个keep-alive连接
<dubbo:protocol name="dubbo" lazy="true" />

在dubbo中配置粘性连接
粘着连接尽可能多地用于有状态服务，以便客户机总是调用同一提供者，除非提供者挂起并连接到另一个提供者。

固定连接将自动打开延迟连接，以减少长连接的数量。
<dubbo:reference id="xxxService" interface="com.xxx.XxxService" sticky="true" />

Dubbo支持方法级的粘性连接，如果你想要更细粒度的控制，你也可以按照下面的方式进行配置。
<dubbo:reference id="xxxService" interface="com.xxx.XxxService">
    <dubbo:mothod name="sayHello" sticky="true" />
</dubbo:reference>

在dubbo中配置基于令牌的授权
通过注册中心的令牌授权控制中心来决定是否向消费者发放令牌，可以防止消费者绕过注册中心访问提供者，另一个通过注册中心可以灵活更改授权而无需修改或升级提供者

你可以全局开启令牌认证:
<dubbo:provider interface="com.foo.BarService" token="true" />
<dubbo:provider interface="com.foo.BarService" token="123456" />

当然可以在服务级别开启令牌身份验证:
<dubbo:service interface="com.foo.BarService" token="true" />
<dubbo:service interface="com.foo.BarService" token="123456" />

也可以在协议级别开启令牌认证:
<dubbo:protocol name="dubbo" token="true" />
<dubbo:protocol name="dubbo" token="123456" />

配置dubbo路由规则
路由规则1确定一次服务呼叫的目标服务器。它有两种路由规则:条件路由规则和脚本路由规则。它还支持extension2。
写路由规则
向注册表写入路由规则通常由监控中心或控制台页面完成。
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("route://0.0.0.0/com.foo.BarService?category=routers&dynamic=false&rule=" + URL.encode("host = 10.20.153.10 => host = 10.20.153.11")));

route://路由规则类型，支持路由规则和脚本路由规则，可扩展。必需的。
0.0.0.0表示所有的IP地址都有效。如果只对一个IP地址生效，请填写IP地址。必需的。
com.foo。BarService指定的服务有效。必需的。
group=foo指定组内指定服务生效。缺席时，未配置组的指定业务生效。
version=1.0表示指定版本的指定服务生效。缺席时，未配置版本的指定服务生效。
category=routers 表示该数据是动态配置类型。必需的。
dynamic=false表示持久化数据。当注册者退出时，数据仍然存储在注册表中。必需的。
enabled=true表示该路由规则是否有效。选项，默认有效。
force=false当路由结果为空时是否强制执行。如果不执行，路由将自动失效。选项，默认为false。

runtime=false表示是否在每次调用时都执行路由规则。如果没有，则只在提供者的地址列表更改时预执行并缓存结果。
当调用服务时，它将从缓存获取路由结果。如果使用参数路由，则必须将其配置为true。请谨慎配置，以免影响性能。选项，默认为false。

priority=1路由规则的优先级。它用于排序，优先级越高，执行越前端。选项，默认为0。
规则= URL。编码("host = 10.20.153.10 => host = 10.20.153.11")表示路由规则的内容，必需的。

有条件的路由规则
基于条件表达式的路由规则，如:host = 10.20.153.10 => host = 10.20.153.11

规则:
前面的=>是消费者的匹配条件。所有参数都与使用者的URL进行比较。当消费者满足条件时，它将继续为消费者执行后面的筛选规则。
在=之后>的目的是过滤提供者地址列表。所有参数都将与提供者的URL进行比较，最终使用者只能获得经过过滤的地址列表。
如果前面的消费者条件为空，则意味着所有的消费者都可以匹配。例如:=> host != 10.20.153.11
如果provider的过滤条件为空，则表示禁止访问。例如:host = 10.20.153.10 =>

表达式:
参数支持:

服务调用信息，如:方法、参数等。当前不支持参数路由
URL字段(在URL本身)，如:协议，主机，端口等。
URL上的所有参数。如:申请、组织等。
条件支持:

等号=表示匹配。例如:host = 10.20.153.10
不等于符号!=表示“不匹配”。例如:host != 10.20.153.10。
价值支持:

多个值之间用逗号分隔。例如:host != 10.20.153.10,10.20.153.11
以*结尾表示通配符。例如:host != 10.20.*
以$开头表示对消费者参数的引用。例如:host = $host

Samples

排除预发布机:
=> host != 172.22.3.91

白名单3:
register.ip != 10.20.153.10,10.20.153.11 =>

黑名单:
register.ip = 10.20.153.10,10.20.153.11 =>

服务登机应用程序只暴露机器的一部分，以防止整个集群挂起:
=> host = 172.22.3.1*,172.22.3.2*

用于重要应用的附加机器:
application != kylin => host != 172.22.3.95,172.22.3.96

读写分离:
method = find*,list*,get*,is* => host = 172.22.3.94,172.22.3.95,172.22.3.96
method != find*,list*,get*,is* => host = 172.22.3.97,172.22.3.98

前端与后台应用分离:
application = bops => host = 172.22.3.91,172.22.3.92,172.22.3.93
application != bops => host = 172.22.3.94,172.22.3.95,172.22.3.96

隔离不同网段:
host != 172.22.3.* => host != 172.22.3.*

供应商和消费者部署在同一个集群中，机器只访问本地服务:
=> host = $host

脚本路由规则
脚本路由规则4支持JDK脚本引擎的所有脚本。如:javascript, jruby, groovy等。按type=javascript配置脚本类型，默认为javascript。
"script://0.0.0.0/com.foo.BarService?category=routers&dynamic=false&rule=" + URL.encode("(function route(invokers) { ... } (invokers))")

基于脚本引擎的路由规则如下:

标记路由规则
标签路由规则5、当应用程序配置标签路由器时，带有标签的dubbo调用可以智能地路由到具有相应标签的服务提供者。

未配置标记的应用程序将被视为默认应用程序，当调用未能匹配提供程序时，这些默认应用程序将被视为降级应用程序。

Consumer
RpcContext.getContext().setAttachment(Constants.REQUEST_TAG_KEY,"red");
请求的范围。标记用于每个调用，使用附件传递请求标记。请注意，存储在附件中的值将在完整的远程调用中持续传递，由于这个特性，我们只需要在调用开始时设置标记。
目前只支持硬编码来设置requestTag。注意，RpcContext是线程绑定的，优雅地使用了TagRouter特性，建议通过servlet过滤器(在web环境中)或自定义dubbo SPI过滤器设置请求标记。

Rules
请求。tag=red将首先选择配置为tag=red的提供程序。如果集群中没有与请求标记对应的服务，它将降级为tag=null提供程序，被视为默认提供程序。
当请求。tag=null，只匹配tag=null提供程序。即使集群中有可用的服务，标签不匹配，也不能调用它们。
这与规则1不同。带标记的调用可以降级为不带标记的服务，但是不带标记的调用/不带其他类型的标记的调用永远不能被其他标记服务访问。

在dubbo中配置规则
然后将动态配置写入注册中心，此功能通常由监控中心或中心页面完成。

RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&timeout=1000"));

在config override url中:
override://表示该数据被覆盖，支持覆盖和缺席，可以扩展，需要。
0.0.0.0表示该配置对所有IP地址都有效，如果只需要覆盖指定的IP数据，可以替换指定的IP地址。
com.foo。BarService对指定的服务有效，必需的。
category=configurators表示该数据是动态配置，需要配置。
dynamic=false表示数据是持久的，当被注册方退出时，数据仍然存储在所需的注册中心中。
enabled=true覆盖策略为启用，可以缺席，如果缺席，则启用。
application=foo表示对指定应用有效，can缺席，如果缺席，则对所有应用有效。
timeout=1000表示将满足上述条件的timeout参数值覆盖1000，如果需要覆盖其他参数，请直接在覆盖URL参数上添加。

例子:

禁用服务提供者。(通常用于暂时启动提供商机器，类似于禁止消费者访问，请使用路由规则)
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&disbaled=true

调整权重:(通常用于容量评估，默认为100)
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&weight=200

调整负载均衡策略。(默认随机)
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&loadbalance=leastactive

服务降级:(通常用于临时掩盖非关键服务的错误)
override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null

在dubbo降级服务
您可以通过服务降级暂时屏蔽非关键服务，并为其定义返回策略。

发布动态配置规则到注册表:
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null"));

配置mock=force:return+null意味着该服务的所有调用都将直接返回空值，而不进行远程调用。通常用于减少一些缓慢的非关键服务的影响。
您还可以将该配置更改为mock=fail:return+null。然后，在调用失败后，您将得到null值。如果调用成功，消费者将尝试进行远程调用以获得真正的结果，如果调用失败，您将获得null值。通常用于容忍一些非关键服务。

在dubbo优雅的关闭
Dubbo是通过JDK的关机来实现优雅关机的，所以如果你强制关机命令，例如kill -9 PID，就不会执行优雅关机，并且只有在kill PID被传递时才会执行。

Howto
Service provider
当停止时，首先标记为没有接收新请求，新请求直接返回错误，以便客户机重试其他机器。
然后检查线程池中的线程是否正在运行，如果有，等待所有线程完成执行，除非超时，然后强制关闭。

Service consumer
当停止时，不再发起新的请求，客户机上的所有请求都将得到一个错误。
然后检查请求是否返回响应，等待响应返回，除非超时，然后被迫关闭。

配置关闭等待时间
设置安全关机超时时间，超时时间强制关闭时，默认超时时间为10秒。
# dubbo.properties
dubbo.service.shutdown.wait=15000

如果ShutdownHook不起作用，你可以自己调用它，在tomcat中，建议扩展ContextListener并调用以下代码来实现优雅的关机:
DubboShutdownHook.destroyAll();

dubbo中的主机名绑定

默认主机IP查找顺序:

通过LocalHost.getLocalHost()获取本地地址。
如果是127。*环回地址，然后扫描网络的主机IP

注册地址如果不正确，如需要注册公共地址，可以这样做:

编辑/etc/hosts:添加机器名和公共ip，例如:
test1 205.182.23.201

在dubbo.xml中添加主机地址配置:
<dubbo:protocol host="205.182.23.201">

或者在dubbo.properties中配置:
dubbo.protocol.host=205.182.23.201

可以对端口进行如下配置:
in dubbo.xml add port configuration:
<dubbo:protocol name="dubbo" port="20880">

or config that in dubbo.properties:
dubbo.protocol.dubbo.port=20880

dubbo的缓存引用配置
ReferenceConfig的实例是重的。它封装了到注册中心的连接和到提供者的连接，因此需要对其进行缓存。否则，重复生成ReferenceConfig可能会导致性能问题、内存和连接泄漏。在API模式下编程时，这个问题很容易被忽略。
因此，从2.4.0开始，dubbo提供了一个简单的实用工具ReferenceConfigCache来缓存ReferenceConfig的实例。
在缓存中销毁ReferenceConfig，它也删除ReferenceConfig并释放相应的资源。

缺省情况下，ReferenceConfigCache为同一个服务组、接口、版本缓存一个ReferenceConfig。ReferenceConfigCache的键来自服务组、接口和版本组。
您可以修改策略。定义一个KeyGenerator实例，将其作为getCache方法的参数传递。有关信息，请参阅ReferenceConfigCache。

配置注册表服务在dubbo
您有两个镜像环境，两个注册中心。您只在一个注册中心部署了一个服务，另一个注册中心还没有时间部署，两个注册中心的其他应用程序需要依赖该服务。此时，服务提供者将服务注册到另一个注册商，但服务使用者不使用来自另一个注册商的服务。

禁用订阅配置
<dubbo:registry id="hzRegistry" address="10.20.153.10:9090" />
<dubbo:registry id="qdRegistry" address="10.20.141.150:9090" subscribe="false" />

<dubbo:registry id="hzRegistry" address="10.20.153.10:9090" />
<dubbo:registry id="qdRegistry" address="10.20.141.150:9090?subscribe=false" />

dubbo中的分布式事务支持

自动线程转储在dubbo

当业务线程池满时，我们需要知道有哪些资源/条件在等待线程，以找到系统的瓶颈点或异常点。dubbo自动通过Jstack导出线程堆栈，以保持场景，方便故障排除。

默认策略:

导出文件路径,用户。主目录
导出时间间隔，最短的时间间隔允许您每10分钟导出一次
指定的导出文件路径:
# dubbo.properties
dubbo.application.dump.directory=/tmp

<dubbo:application ...>
    <dubbo:parameter key="dump.directory" value="/tmp" />
</dubbo:application>

在dubbo中配置netty4支持
在dubbo 2.5.6版本中增加了对netty4通信模块的支持，启用如下:
<dubbo:protocol server="netty4" />
<dubbo:provider server="netty4" />
<dubbo:consumer client="netty4" />

如果供应商需要为不同的协议使用不同的通信层框架，请分别配置多个协议。

消费者配置如下:

<dubbo:consumer client="netty">
    <dubbo:reference />
</dubbo:consumer>

<dubbo:consumer client="netty4">
    <dubbo:reference />
</dubbo:consumer>

接下来我们将继续做一些事情:我们将提供性能测试指标和与netty 3版本性能测试比较的参考数据。

序列化
在Dubbo (Kryo和FST)中使用高效的Java序列化
使用Kryo和FST非常简单，只需添加一个属性到dubbo RPC XML配置:
<dubbo:protocol name="dubbo" serialization="kryo"/>
<dubbo:protocol name="dubbo" serialization="fst"/>

注册序列化的类
为了释放Kryo和FST的高能力，最好注册需要序列化到dubbo系统中的类。例如，我们可以实现以下回调接口:

然后添加XML配置:
<dubbo:protocol name="dubbo" serialization="kryo" optimizer="org.apache.dubbo.demo.SerializationOptimizerImpl"/>
注册这些类之后，序列化性能可以大大提高，特别是对于数量较少的嵌套对象。

当然，在序列化一个类时，您还可以对许多类进行级联引用，比如Java集合类。在这种情况下，我们已经自动注册了JDK中的公共类，所以你不需要重复注册它们(当然，如果你再次注册它们也没关系)
因为注册序列化的类只是为了优化性能，所以如果忘记注册一些类也没关系。事实上，Kryo和FST通常比Hessian和Dubbo序列化性能更好，即使没有注册类。

模式配置参考
dubbo的XML模式配置参考
下面的页面显示了所有配置属性1，并以XML配置为例。其他配置请参考:属性配置，注释配置，API配置。

所有配置属性分为三类，见下表中的“函数”。

服务发现:用于服务注册和发现，以便为消费者找到提供者。
服务治理:用于服务管理和治理，例如为开发或测试提供便利。
Performance optimization:性能优化。不同的属性可能会有不同的性能影响。
所有属性都将转换为URL 3，由提供者生成。该url将由使用者通过注册中心订阅。请参见下表中每个属性对应的URL参数。

注意:组、接口和版本这三个属性决定了服务。所有其他属性都用于服务治理或性能优化。↩︎

dubbo:application
Application configuration. The corresponding class: org.apache.dubbo.config.ApplicationConfig

name “应用名称”是应用的唯一标识。它是用于注册中心梳理应用程序的依赖关系。注意:使用者和提供者应用程序名称不应该相同，而且这个参数不是匹配条件。作为建议，您可以将其命名为您的项目名称。例如，kylin应用程序调用morgan应用程序的服务，则可以将kylin应用程序命名为“kylin”，将morgan应用程序命名为“morgan”。也许kylin也可以作为提供者，但kylin还是应该被称为“kylin”。通过这种方式，registry可以理解应用程序的依赖性
version 当前应用程序的版本
owner 应用程序管理器。请填写负责人的邮箱前缀
organization 组织名称用于注册中心区分服务的来源。作为一个建议，这个属性应该直接写入配置文件。如china,intl,itu,crm,asc,dw，全球速卖通等。
architecture 服务分层的体系结构。比如国际，中国等等。不同的体系结构使用不同的层
environment 应用程序环境。像开发、测试产品。作为开发新功能的极限条件
compiler Java类编译。它用于生成动态类。选项有JDK和javassist

dubbo:argument
Method argument configuration. The corresponding class：org.apache.dubbo.config.ArgumentConfig. This tag is child of <dubbo:method>, which is for feature description of method argument, such as:

<dubbo:method name="findXxx" timeout="3000" retries="2">
    <dubbo:argument index="0" callback="true" />
</dubbo:method>

index method name
type 通过它找到参数的索引
callback 标记此参数是否为回调服务。如果为真，提供者将生成反向代理，而反向代理又可以调用使用者。通常对于事件推送

dubbo:config-center
Configuration center. Corresponding configuration class: org.apache.dubbo.config.ConfigCenterConfig

protocol 使用哪个配置中心:apollo, zookeeper, nacos等。
         以zookeeper为例
         1. 如果指定了protocol，则address可以简化为127.0.0.1:2181;
         2. 如果未指定protocol，则address为zookeeper://127.0.0.1:2181
address 配置中心地址。
        请参阅协议说明获取值
highest-priority 来自配置中心的配置项具有最高的优先级，这意味着本地配置项将被覆盖。
namespace 用于多租户隔离通常，实际含义因配置中心的不同而不同。
          例如:
          zookeeper -环境隔离，默认dubbo;
          区分不同域的配置集，并在默认的dubbo和application中使用它们
cluster 根据选择的配置中心不同，含义不同。
        例如，它用于区分阿波罗中不同的配置集群
group 根据选择的配置中心不同，含义不同。
      隔离不同的配置集
      zookeeper -隔离不同的配置集
check 是否在配置集线器连接失败时终止应用程序启动。
config-file 该键映射到全局级别概要文件
            动物园管理员:$ DEFAULT_PATH /达博/ config /达博/ dubbo.properties
            阿波罗——达波。属性键在dubbo命名空间
timeout 获取配置的超时。
username 如果配置中心需要验证，请输入用户名
         阿波罗还没有启动
password 如果配置中心需要检查，请输入密码 阿波罗还没有启动
parameters