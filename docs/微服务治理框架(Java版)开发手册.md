# 微服务治理框架(Java版)开发手册

## grpc简介
grpc是一个多语言、高性能、开源的通用远程过程调用(RPC)框架。

- 来自于Google的开源项目，2016年8月19日发布了成熟的1.0.0版本
- 基于HTTP/2等技术
- grpc支持Java, C，C++, Python等多种常用的编程语言，并且客户端和服务端可以采用不同的编程语言来实现
- 数据序列化使用[Protocol Buffers](https://github.com/google/protobuf)
- grpc社区非常活跃，版本迭代也比较快速，2019年4月11日发布了1.20.0版本

官网地址： [http://www.grpc.io/](http://www.grpc.io/)

源码地址： [https://github.com/grpc](https://github.com/grpc)

## 功能列表

### 公共
- 基于Zookeeper的注册中心
- 支持Zookeeper开启ACL

### 服务端
- 服务端启动时，注册服务端信息
- 服务端关闭时，注销服务端信息
- 服务端流量控制(并发请求数流量控制、并发连接数控制流量控制)
- 服务端配置信息监听

### 客户端
- 客户端启动时，注册客户端信息
- 客户端关闭时，注销客户端信息
- 监听服务端列表、服务端权重信息
- 监听路由规则，获取可访问的服务列表
- 支持路由规则可以设置为IP段、项目
- 客户端监听配置信息的更新
- 客户端流量控制：限制客户端对某个服务每秒钟的请求次数（Requests Per Second）
- 两种负载均衡模式（连接负载均衡、请求负载均衡）
- 四种负载均衡算法（随机、轮询、加权轮询、一致性Hash）
- grpc断线重连指数退避算法支持参数配置功能
- 服务容错：配置服务调用出错后自动重试次数后，可以启用服务容错功能，当调用某个服务端出错后，框架自动尝试切换到提供相同服务的服务端再次发起请求。

## 使用示例
下面以maven项目orientsec-grpc-java-demo([源码](https://github.com/grpc-nebula/grpc-nebula-samples-java/tree/master/orientsec-grpc-java-demo))作为示例，具体介绍如何使用微服务治理框架(Java版)。

### 1. pom.xml文件配置

依赖信息如下：

	<properties>
        <orientsec.grpc.version>1.1.0</orientsec.grpc.version>               
    </properties>

	 <dependencies>
        <!-- orientsec-grpc-java -->
        <dependency>
            <groupId>com.orientsec.grpc</groupId>
            <artifactId>orientsec-grpc-netty-shaded</artifactId>
            <version>${orientsec.grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>com.orientsec.grpc</groupId>
            <artifactId>orientsec-grpc-protobuf</artifactId>
            <version>${orientsec.grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>com.orientsec.grpc</groupId>
            <artifactId>orientsec-grpc-stub</artifactId>
            <version>${orientsec.grpc.version}</version>
        </dependency>
	</dependencies>

这里配置的依赖信息和原生grpc-java依赖信息的区别是：

- `groupId` 从 `io.grpc` 修改为 `com.orientsec.grpc`
- `artifactId` 在原来的基础上增加了前缀 `orientsec-`
- `version` 使用 `${orientsec.grpc.version}` ( 即 `1.1.0` )


### 2. 框架配置文件dfzq-grpc-config.properties

应用启动时，按以下顺序搜索配置文件；如果没找到，则顺延到下一条：

(a) 用户可以通过启动参数-Ddfzq.grpc.config=/xxx/xxx 配置grpc配置文件所在的目录的绝对路径

(b) 从启动目录下的config中查找grpc配置文件（如果找不到从jar包内的classpath:/config/目录下查找）

(c) 从启动目录下查找grpc配置文件（如果找不到从jar包内的classpath:/目录下查找）

maven项目，可以将配置文件放在源码/src/main/resources/config目录下，也可以放在源码/src/main/resources/目录下。

`dfzq-grpc-config.properties` 的内容如下：

	# ------------ begin of common config ------------
	
	# 必填,类型string,说明:当前应用名称
	common.application=grpc-test-application
	
	# 必填,类型string,说明:当前项目名
	common.project=grpc-test-project
	
	# 必填,类型string,说明:项目负责人,员工工号,多个工号之间使用英文逗号
	common.owner=1023,1234
	
	# 可选,类型string,说明:服务注册根路径,默认值/Application/grpc
	common.root=/Application/grpc
	
	# 可选,类型string,说明:服务注册使用的IP地址
	# 如果不配置该参数值，当前服务器的IP地址为"非127.0.0.1的第一个网卡的IP地址"
	# 使用场合:一台服务器安装有多个网卡,如果需要指定不是第一个网卡的IP地址为服务注册的IP地址
	#common.localhost.ip=xxx.xxx.xxx.xxx
	
	# ------------ end of common config ------------
	
	
	
	
	# ------------ begin of provider config ------------
	
	# 必填,类型string,说明:服务的版本信息，一般表示服务接口的版本号
	provider.version=1.0.0
	
	# ----------------------------------------
	# 可选,类型int,缺省值20,说明:服务提供端可处理的最大连接数，即同一时刻最多有多少个消费端与当前服务端建立连接
	# 如果不限制连接数，将这个值配置为0
	# 对连接数的控制，无法控制到指定的服务，只能控制到指定的IP:port
	provider.default.connections=20
	
	# 可选,类型int,缺省值2000,说明:服务提供端可处理的最大并发请求数
	# 如果不限制并发请求数，将这个值配置为0
	# 备注：同一个连接发送多次请求
	provider.default.requests=2000
	
	# 可选,类型int,缺省值100,说明:服务provider权重，是服务provider的容量，在负载均衡基于权重的选择算法中用到
	provider.weight=100
	
	# 可选,类型String,固定值provider,说明:provider表示服务提供端，consumer表示服务消费端
	provider.side=provider
	
	# 可选,类型string,缺省值1.0.0,说明:gRPC 协议版本号
	provider.grpc=1.1.0
	# ------------ end of provider config ------------
	
	
	
	
	# ------------ begin of consumer config ------------
	
	# 可选,类型string,缺省值connection,说明：负载均衡模式
	# 可选值为 connection 和 request,分别表示“连接负载均衡”、“请求负载均衡”
	# “连接负载均衡”适用于大部分业务场景，服务端和客户端消耗的资源较小。
	# “请求负载均衡”适用于服务端业务逻辑复杂、并有多台服务器提供相同服务的场景。
	consumer.loadbalance.mode=request
	
	# 可选,类型string,缺省值round_robin,说明:负载均衡策略，
	# 可选范围：pick_first、round_robin、weight_round_robin、consistent_hash
	# 参数值的含义分别为：随机、轮询、加权轮询、一致性Hash
	consumer.default.loadbalance=weight_round_robin
	
	# 可选,类型string,负载均衡策略选择是consistent_hash(一致性Hash)，配置进行hash运算的参数名称的列表
	# 多个参数之间使用英文逗号分隔，例如 id,name
	# 如果负载均衡策略选择是consistent_hash，但是该参数未配置参数值、或者参数值列表不正确，则取第一个参数的参数值返回
	# 备注：该参数只支持通过配置文件配置
	# consumer.consistent.hash.arguments=id
	
	# 可选,类型integer,缺省值5,说明：连续多少次请求出错，自动切换到提供相同服务的新服务器
	consumer.switchover.threshold=5
	
	# 可选,类型为long,单位为秒,缺省值为60,说明：服务提供者不可用时的惩罚时间，即多次请求出错的服务提供者一段时间内不再去请求
	# 属性值大于或等于0，等于0表示没有惩罚时间，如果客户端只剩下一个服务提供者，即使服务提供者不可用，也不做剔除操作。
	consumer.unavailable.provider.punish.time=60
	
	# 可选,类型String,默认值consumers,说明:所属范畴
	consumer.category=consumers
	
	# 可选,类型String,固定值consumer,说明:provider表示服务提供端，consumer表示服务消费端
	consumer.side=consumer
	
	# 可选,类型int,缺省值0,0表示不进行重试,说明:服务调用出错后自动重试次数
	consumer.default.retries=2
	
	# 指数退避协议https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md
	# 可选,类型long,缺省值120,单位秒,说明:grpc断线重连指数退避协议"失败重试等待时间上限"参数
	consumer.backoff.max=120
	
	# ------------ end of consumer config ------------
	
	
	
	
	# ------------ begin of zookeeper config ------------
	
	# zookeeper主机列表
	# zookeeper.host.server=168.61.2.23:2181,168.61.2.24:2181,168.61.2.25:2181
	zookeeper.host.server=127.0.0.1:2181
	
	# 可选,类型int,缺省值86400000,单位毫秒,即缺省值为1天,说明:zk断线重连最长时间
	zookeeper.retry.time=86400000
	
	# 可选,类型int,缺省值5000,单位毫秒,说明:连接超时时间
	zookeeper.connectiontimeout=5000
	
	# 可选,类型int,缺省值4000,单位毫秒,说明:会话超时时间
	zookeeper.sessiontimeout=4000
	
	# 可选,类型string,访问控制用户名
	zookeeper.acl.username=admin
	
	# 可选,类型string,访问控制密码
	# 这里的密码配置的是密文，使用com.orientsec.grpc.common.util.DesEncryptUtils#encrypt(String plaintext)进行加密
	zookeeper.acl.password=9b579c35ca6cc74230f1eed29064d10a
	
	# ------------ end of zookeeper config ------------


### 3. 客户端调用的写法变化
- 使用原生grpc-java的写法

		String host = "192.168.0.1";
		int port = 50051;

		channel = ManagedChannelBuilder.forAddress(host, port)
				.usePlaintext()
				.build();	

		blockingStub = GreeterGrpc.newBlockingStub(channel);
	
- 使用微服务治理框架(Java版)的写法

	 	String target = "zookeeper:///" + GreeterGrpc.SERVICE_NAME;
	
	    channel = ManagedChannelBuilder.forTarget(target)
	            .usePlaintext()
	            .build();
	
	    blockingStub = GreeterGrpc.newBlockingStub(channel);

- 区别在于使用字符串“ **zookeeper:///** + **服务名称** ”的方式指定服务。这样可以根据服务名称从多台提供相同服务的服务器中动态选择一台服务器。  


### 4. Protocol Buffers文件

Protocol Buffers文件用来定义服务名称、方法、入参、出参。maven项目中可以通过protobuf-maven-plugin插件，根据Protocol Buffers文件生成Java代码。

示例： src/main/proto/greeter.proto

	syntax = "proto3";
	option java_multiple_files = true;
	option java_package = "com.orientsec.demo";
	option java_outer_classname = "GreeterProto";
	package com.orientsec.demo;
	
	
	service Greeter {
	    rpc sayHello (GreeterRequest) returns (GreeterReply) {}
	}
	
	message GreeterRequest {
	    int32  no   = 1;
	    string name = 2;
	    bool sex = 3;
	    double salary = 4;
	    string desc = 5;
	}
	
	message GreeterReply {
	    bool success = 1;
	    string message = 2;
	    int32  no   = 3;
	    double salary = 4;
	    int64 total = 5;
	}

### 5. 服务实现类

服务实现类，是对Protocol Buffers文件定义的方法sayHello进行具体的业务实现。

示例：com.orientsec.demo.service.GreeterImpl

	/**
	 * Greeter服务实现类
	 */
	public class GreeterImpl extends GreeterGrpc.GreeterImplBase {
	  /**
	   * sayHello方法实现
	   */
	  public void sayHello(GreeterRequest request, StreamObserver<GreeterReply> responseObserver) {
	    int no = request.getNo();
	    String name = request.getName();
	    boolean sex = request.getSex();// true:male,false:female
	    double salary = request.getSalary();
	    String desc = request.getDesc();
	
	    String appellation;
	    if (sex) {
	      appellation = "Mr " + name;
	    } else {
	      appellation = "Miss " + name;
	    }
	
	    GreeterReply reply = GreeterReply.newBuilder()
	            .setSuccess(true)
	            .setMessage(appellation + ", well done.(" + desc + ")")
	            .setNo(no + 100)
	            .setSalary(salary * 1.2)
	            .setTotal(System.currentTimeMillis())
	            .build();
	
	    responseObserver.onNext(reply);
	    responseObserver.onCompleted();
	  }
	
	}

### 6. 服务提供者(服务端)

服务提供者，启动一个Server对象，等待客户端的连接。

服务提供者可以提供一个服务，也可以提供多个服务（addService方法）。

示例：com.orientsec.demo.server.GreeterServer
	
	public class GreeterServer {
	  private static final Logger logger = LoggerFactory.getLogger(GreeterServer.class);
	
	  private Server server;
	  private int port = Constants.Port.GREETER_SERVICE_SERVER;
	
	  private void start() throws IOException {
	    server = ServerBuilder.forPort(port)
	            .addService(new GreeterImpl())
	            .build()
	            .start();
	
	    logger.info("GreeterServer start...");
	
	    Runtime.getRuntime().addShutdownHook(new Thread() {
	
	      @Override
	      public void run() {
	        // Use stderr here since the logger may have been reset by its JVM shutdown hook.
	        System.err.println("*** shutting down gRPC server since JVM is shutting down");
	        GreeterServer.this.stop();
	        System.err.println("*** GreeterServer shut down");
	      }
	    });
	  }
	
	  private void stop() {
	    if (server != null) {
	      logger.info("stop GreeterServer...");
	      server.shutdown();
	    }
	  }
	
	  private void blockUntilShutdown() throws InterruptedException {
	    if (server != null) {
	      server.awaitTermination();
	    }
	  }
	
	
	  public static void main(String[] args) throws Exception {
	    GreeterServer server = new GreeterServer();
	    server.start();
	
	    server.blockUntilShutdown();
	  }
	}

### 7. 客户端（单例）

客户端通过调用 **ManagedChannelBuilder.forTarget(String target)** 方法建立Channel，其中target的组成组成规则为“ **zookeeper:/// + 服务名称** ”。框架会到注册中心zookeeper上去寻找对应服务名称的服务提供者。

示例：com.orientsec.demo.client.GreeterClient

	public class GreeterClient {
	  private static final Logger logger = LoggerFactory.getLogger(GreeterClient.class);
	
	  private final ManagedChannel channel;
	  private final GreeterGrpc.GreeterBlockingStub blockingStub;
	
	  public static GreeterClient getInstance() {
	    return SingletonHolder.INSTANCE;
	  }
	
	  // 懒汉式单例模式--直到使用时才创建对象
	  private static class SingletonHolder {
	    private static final GreeterClient INSTANCE = new GreeterClient();
	  }
	
	  private GreeterClient() {
	    //channel = ManagedChannelBuilder.forAddress(host, port)
	    //        .usePlaintext()
	    //        .build();
	
	    String target = "zookeeper:///" + GreeterGrpc.SERVICE_NAME;
	
	    channel = ManagedChannelBuilder.forTarget(target)
	            .usePlaintext()
	            .build();
	
	    blockingStub = GreeterGrpc.newBlockingStub(channel);
	  }
	
	
	  public void shutdown() throws InterruptedException {
	    channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
	  }
	
	  public void greet() {
	    try {
	      int no = 100;
	      String name = "Alice";
	      boolean sex = false;// true:male,false:female
	      double salary = 6000.0;
	      String desc = "我爱夏天";
	
	      GreeterRequest request = GreeterRequest.newBuilder()
	              .setNo(no)
	              .setName(name)
	              .setSex(sex)
	              .setSalary(salary)
	              .setDesc(desc)
	              .build();
	
	      GreeterReply reply = blockingStub.sayHello(request);
	
	      logger.info(String.valueOf(reply.getSuccess()));
	      logger.info(reply.getMessage());
	    } catch (Exception e) {
	      logger.error(e.getMessage(), e);
	    }
	  }
	
	  /**
	   * main
	   */
	  public static void main(String[] args) throws Exception {
	    GreeterClient client = GreeterClient.getInstance();
	
	    long count = -1;
	    long interval = 9000L;// 时间单位为毫秒
	    long LOOP_NUM = 10;
	
	    while (true) {
	      count++;
	      if (count >= LOOP_NUM) {
	        break;
	      }
	
	      client.greet();
	      TimeUnit.MILLISECONDS.sleep(interval);
	    }
	
	    client.shutdown();
	  }
	}

### 8. 编译

	mvn clean
	mvn install

使用eclipse编译时源代码路径会自动更新。使用IntelliJ IDEA时，需要手动将.proto文件生成的java文件所在目录设置为源代码目录，如下图：
![IntelliJ IDEA设置额外的源代码路径](https://raw.githubusercontent.com/grpc-nebula/grpc-nebula/master/images/idea-protobuf-sources-setting.png)


### 9. 运行示例程序

启动框架配置文件中指定的zookeeper，运行 `com.orientsec.demo.server.GreeterServer` ，再运行`com.orientsec.demo.client.GreeterClient` 。


## 如何在Spring Boot项目中使用框架
详见代码示例 [orientsec-grpc-springboot-demo](https://github.com/grpc-nebula/grpc-nebula-samples-java/tree/master/orientsec-grpc-springboot-demo)


## 常见问题
### 1.安装多张网卡的服务器如何指定服务注册使用的IP地址

一台服务器安装有多个网卡的情况下, 支持将【服务注册使用的IP地址】设置为某个指定的IP地址。使用方法如下：
在配置文件dfzq-grpc-config.properties中增加 common.localhost.ip=xxx.xxx.xxx.xxx 配置。

	# 可选,类型string,说明:服务注册使用的IP地址
	# 如果不配置该参数值，当前服务器的IP地址为"非127.0.0.1的第一个网卡的IP地址"
	# 使用场合:一台服务器安装有多个网卡,如果需要指定不是第一个网卡的IP地址为服务注册的IP地址
	common.localhost.ip=xxx.xxx.xxx.xxx

如果配置的IP地址不合法，忽略该配置；如果配置的IP地址不包含在本机网卡的IP地址中，忽略该配置。


### 2.客户端调用服务端出错原因汇总
 - 服务端未启动
 - 客户端与服务端配置的注册中心地址不一样(即zookeeper服务器地址不一样）
 - 网络限制：客户端能ping通服务端，但是无法telnet到服务端指定端口
 - 微服务治理框架(Java版)，服务端默认并发连接数为20，并发请求数为2000。即默认情况下，最多只能有20个客户端同时连接服务端，同一时刻最多能接受2000个请求。可以通过调整服务端的配置文件 dfzq-grpc-config.properties 进行解决。

		# 可选,类型int,缺省值20,说明:服务提供端可处理的最大连接数，即同一时刻最多有多少个消费端与当前服务端建立连接
		# 如果不限制连接数，将这个值配置为0
		# 对连接数的控制，无法控制到指定的服务，只能控制到指定的IP:port
		# provider.default.connections=
		
		# 可选,类型int,缺省值2000,说明:服务提供端可处理的最大并发请求数
		# 如果不限制并发请求数，将这个值配置为0
		# 备注：同一个连接发送多次请求
		# provider.default.requests= 

