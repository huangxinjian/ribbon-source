Ribbon
======

Ribbon is a client side IPC(Inter-Process Communication) library that is battle-tested in cloud(经过云上生产验证的). 
It provides the following features

* Load balancing 负载均衡
* Fault tolerance 容错性
* Multiple protocol (HTTP, TCP, UDP) support in an asynchronous and reactive model 异步和reactive模式下的多协议支持
* Caching and batching 缓存和批处理

To get ribbon binaries, go to [maven central](http://search.maven.org/#search%7Cga%7C1%7Cribbon).
Here is an example to add dependency in Maven:

```xml
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon</artifactId>
    <version>2.2.2</version>
</dependency>
```

## Modules 模块划分，OOA 面向对象分析之系统设计之模块划分

* ribbon: APIs that integrate load balancing, fault tolerance, 
          caching/batching on top of other ribbon modules and [Hystrix](https://github.com/netflix/hystrix)
          整个项目中最顶层的、提供给调用方使用的接口，主要整合了负载均衡、容错、缓存/批处理还有熔断器 hystrix 的功能
    
* ribbon-loadbalancer: 
          Load balancer APIs that can be used independently(单独的、独立的) or with other modules
          负载均衡项目可以被单独使用或者结合其它模块使用

* ribbon-eureka: 
          APIs using [Eureka client](https://github.com/netflix/eureka) to provide dynamic server list for cloud
          使用了 eureka 来为云服务提供动态的服务列表
  
* ribbon-transport: 
          Transport clients that support HTTP, TCP and UDP protocols using [RxNetty](https://github.com/netflix/rxnetty) with 
            load balancing capability
          使用具有负载平衡功能的 RxNetty 支持 HTTP、TCP 和 UDP 协议的传输客户端
        
* ribbon-httpclient: 
          REST client built on top of Apache HttpClient integrated with load balancers 
          (deprecated and being replaced by ribbon module)
          REST 客户端构建在与负载均衡器集成的 Apache HttpClient 之上 （已过时，并由 ribbon-module 代替）
  
* ribbon-core: 
         Client configuration APIs and other shared APIs
         客户端配置相关的 API 以及其它共享的 API，例如工具类巴拉巴拉的

* ribbon-example: Examples

## Project Status: On Maintenance  项目状态：维护中
Ribbon comprises of multiple components some of which are used in production internally and some of which were replaced by non-OSS solutions over time.
This is because Netflix started moving into a more componentized architecture for RPC with a focus on single-responsibility modules. So each Ribbon component gets a different level of attention at this moment.

More specifically, here are the components of Ribbon and their level of attention by our teams:
* ribbon-core: **deployed at scale in production 生产大规模部署**
* ribbon-eureka: **deployed at scale in production 生产大规模部署**
* ribbon-evcache: **not used**
* ribbon-guice: **not used**
* ribbon-httpclient: **we use everything not under com.netflix.http4.ssl. 
                       Instead, we use an internal solution developed by our cloud security team**
                     **com.netflix.http4.ssl. 这包下的不用**
* ribbon-loadbalancer: **deployed at scale in production 生产大规模部署**
* ribbon-test: **this is just an internal integration test suite 内部集成的测试套件**
* ribbon-transport: **not used 不用了**
* ribbon: **not used 不用了**

Even for the components deployed in production we have wrapped them in a Netflix internal http client and we are not adding new functionality since they’ve been stable for a while.
 Any new functionality has been added to internal wrappers on top of Ribbon (such as request tracing and metrics(指标统计)). 
We have not made an effort to make those components Netflix-agnostic（与netflix无关） under Ribbon.

Recognizing these realities and deficiencies, we are placing Ribbon in maintenance mode.
This means that if an external user submits a large feature request, internally we wouldn’t prioritize it highly（高度重视它）.

However, if someone were to do work on their own and submit complete pull requests, we’d be happy to review and accept.
Our team has instead started building an RPC solution on top of gRPC.
We are doing this transition for two main reasons: multi-language support and better extensibility/composability through request interceptors.
That’s our current plan moving forward.

We currently contribute to the gRPC code base regularly.
To help our teams migrate to a gRPC-based solution in production (and battle-test it),
we are also adding load-balancing and discovery interceptors to achieve feature parity with the functionality Ribbon and Eureka provide.
The interceptors are Netflix-internal at the moment. When we reach that level of confidence we hope to open-source this new approach.
We don’t expect this to happen before Q3 of 2016.

## Release notes

See https://github.com/Netflix/ribbon/releases

## Code example

### Access HTTP resource using template ([full example](https://github.com/Netflix/ribbon/blob/master/ribbon-examples/src/main/java/com/netflix/ribbon/examples/rx/template/RxMovieTemplateExample.java))

```java
HttpResourceGroup httpResourceGroup = Ribbon.createHttpResourceGroup("movieServiceClient",
            ClientOptions.create()
                    .withMaxAutoRetriesNextServer(3)
                    .withConfigurationBasedServerList("localhost:8080,localhost:8088"));
HttpRequestTemplate<ByteBuf> recommendationsByUserIdTemplate = httpResourceGroup.newTemplateBuilder("recommendationsByUserId", ByteBuf.class)
            .withMethod("GET")
            .withUriTemplate("/users/{userId}/recommendations")
            .withFallbackProvider(new RecommendationServiceFallbackHandler())
            .withResponseValidator(new RecommendationServiceResponseValidator())
            .build();
Observable<ByteBuf> result = recommendationsByUserIdTemplate.requestBuilder()
                        .withRequestProperty("userId", "user1")
                        .build()
                        .observe();
```

### Access HTTP resource using annotations ([full example](https://github.com/Netflix/ribbon/blob/master/ribbon-examples/src/main/java/com/netflix/ribbon/examples/rx/proxy/RxMovieProxyExample.java))

```java
public interface MovieService {
    @Http(
            method = HttpMethod.GET,
            uri = "/users/{userId}/recommendations",
            )
    RibbonRequest<ByteBuf> recommendationsByUserId(@Var("userId") String userId);
}

MovieService movieService = Ribbon.from(MovieService.class);
Observable<ByteBuf> result = movieService.recommendationsByUserId("user1").observe();
```

### Create an AWS-ready load balancer with [Eureka](https://github.com/Netflix/eureka) dynamic server list and zone affinity enabled

```java
        IRule rule = new AvailabilityFilteringRule();
        ServerList<DiscoveryEnabledServer> list = new DiscoveryEnabledNIWSServerList("MyVIP:7001");
        ServerListFilter<DiscoveryEnabledServer> filter = new ZoneAffinityServerListFilter<DiscoveryEnabledServer>();
        ZoneAwareLoadBalancer<DiscoveryEnabledServer> lb = LoadBalancerBuilder.<DiscoveryEnabledServer>newBuilder()
                .withDynamicServerList(list)
                .withRule(rule)
                .withServerListFilter(filter)
                .buildDynamicServerListLoadBalancer();   
        DiscoveryEnabledServer server = lb.chooseServer();         
```

### Use LoadBalancerCommand to load balancing IPC calls made by HttpURLConnection ([full example](https://github.com/Netflix/ribbon/blob/master/ribbon-examples/src/main/java/com/netflix/ribbon/examples/loadbalancer/URLConnectionLoadBalancer.java))

```java
CommandBuilder.<String>newBuilder()
        .withLoadBalancer(LoadBalancerBuilder.newBuilder().buildFixedServerListLoadBalancer(serverList))
        .build(new LoadBalancerExecutable<String>() {
            @Override
            public String run(Server server) throws Exception {
                URL url = new URL("http://" + server.getHost() + ":" + server.getPort() + path);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                return conn.getResponseMessage();
            }
        }).execute();
```

## License

Copyright 2014 Netflix, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

## Questions?

Email ribbon-users@googlegroups.com or [join us](https://groups.google.com/forum/#!forum/ribbon-users)


