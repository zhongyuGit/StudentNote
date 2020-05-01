SpringCloud Ribbon（负载均衡）

我们知道要开启restTemplate的负载均衡，我们需要注册RestTemplate，同时添加注解@Loanbalanced.

![1588260901489](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588260901489.png)

该注解是用来给restTemplate做标记的，让客户端(LoadBalancerClient)使用负载均衡的方式配置RestTemplate.

**LoadBalancerClient.class**

![1588261254742](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588261254742.png)

```java
public interface ServiceInstanceChooser {

	/**
	 * Chooses a ServiceInstance from the LoadBalancer for the specified service.
	 * @param serviceId The service ID to look up the LoadBalancer.
	 * @return A ServiceInstance that matches the serviceId.
	 */
	ServiceInstance choose(String serviceId);

}
```

![1588261336930](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588261336930.png)

![1588264066370](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588264066370.png)

![1588264096972](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588264096972.png)

```java


package org.springframework.cloud.client.loadbalancer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.SmartInitializingSingleton;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingClass;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.retry.support.RetryTemplate;
import org.springframework.web.client.RestTemplate;

@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

	@LoadBalanced//维护实例
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}

	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
	}

	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {

		@Bean//注册LoadBalancerInterceptor，
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean//注册RestTemplateCustomizer,给restTemplate添加拦截器
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}

	}
//...忽略了重试
}

```

我们接下来看一下LoadBalancerInterceptor是怎么样将restTemplate配置成客户端负载均衡的：

**LoadBalancerInterceptor.class**

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;

	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer,
			LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null,
				"Request URI does not contain a valid hostname: " + originalUri);
        //调用了LoadBalancerClient的ececute方法执行请求，LoadBalancerClient就是我们用@LoadBalanced‘激活的接口类’，也就是restTemplate发送请求要走这个路径
		return this.loadBalancer.execute(serviceName,
				this.requestFactory.createRequest(request, body, execution));
	}

}
```

当一个被@LoadBalanced标记的restplemate时，

![1588266074844](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588266074844.png)

![1588266240997](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588266240997.png)

```java
//当我们发出restTemplate请求的时候就会被LoadBalancerInterceptor的intercept捕获，调用了相应的//LoadBalancerClient来执行‘包装请求’的excute，这里不用太注意这个类，我只是给大家拿出来看一下啊
//相应重要的方法我会单独挑出来的



package org.springframework.cloud.netflix.ribbon;


public class RibbonLoadBalancerClient implements LoadBalancerClient {
    private SpringClientFactory clientFactory;

    public RibbonLoadBalancerClient(SpringClientFactory clientFactory) {
        this.clientFactory = clientFactory;
    }

    public URI reconstructURI(ServiceInstance instance, URI original) {
        Assert.notNull(instance, "instance can not be null");
        String serviceId = instance.getServiceId();
        RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
        URI uri;
        Server server;
        if (instance instanceof RibbonLoadBalancerClient.RibbonServer) {
            RibbonLoadBalancerClient.RibbonServer ribbonServer = (RibbonLoadBalancerClient.RibbonServer)instance;
            server = ribbonServer.getServer();
            uri = RibbonUtils.updateToSecureConnectionIfNeeded(original, ribbonServer);
        } else {
            server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
            IClientConfig clientConfig = this.clientFactory.getClientConfig(serviceId);
            ServerIntrospector serverIntrospector = this.serverIntrospector(serviceId);
            uri = RibbonUtils.updateToSecureConnectionIfNeeded(original, clientConfig, serverIntrospector, server);
        }

        return context.reconstructURIWithServer(server, uri);
    }

    public ServiceInstance choose(String serviceId) {
        return this.choose(serviceId, (Object)null);
    }

    public ServiceInstance choose(String serviceId, Object hint) {
        Server server = this.getServer(this.getLoadBalancer(serviceId), hint);
        return server == null ? null : new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
    }

    public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
        return this.execute(serviceId, (LoadBalancerRequest)request, (Object)null);
    }

    public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
        //获得负载均衡器
        ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
        Server server = this.getServer(loadBalancer, hint);
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        } else {
            RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
            return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
        }
    }

    public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
        Server server = null;
        if (serviceInstance instanceof RibbonLoadBalancerClient.RibbonServer) {
            server = ((RibbonLoadBalancerClient.RibbonServer)serviceInstance).getServer();
        }

        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        } else {
            RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
            RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

            try {
                T returnVal = request.apply(serviceInstance);
                statsRecorder.recordStats(returnVal);
                return returnVal;
            } catch (IOException var8) {
                statsRecorder.recordStats(var8);
                throw var8;
            } catch (Exception var9) {
                statsRecorder.recordStats(var9);
                ReflectionUtils.rethrowRuntimeException(var9);
                return null;
            }
        }
    }

    private ServerIntrospector serverIntrospector(String serviceId) {
        ServerIntrospector serverIntrospector = (ServerIntrospector)this.clientFactory.getInstance(serviceId, ServerIntrospector.class);
        if (serverIntrospector == null) {
            serverIntrospector = new DefaultServerIntrospector();
        }

        return (ServerIntrospector)serverIntrospector;
    }

    private boolean isSecure(Server server, String serviceId) {
        IClientConfig config = this.clientFactory.getClientConfig(serviceId);
        ServerIntrospector serverIntrospector = this.serverIntrospector(serviceId);
        return RibbonUtils.isSecure(config, serverIntrospector, server);
    }

    protected Server getServer(String serviceId) {
        return this.getServer(this.getLoadBalancer(serviceId), (Object)null);
    }

    protected Server getServer(ILoadBalancer loadBalancer) {
        return this.getServer(loadBalancer, (Object)null);
    }

    protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
        return loadBalancer == null ? null : loadBalancer.chooseServer(hint != null ? hint : "default");
    }

    protected ILoadBalancer getLoadBalancer(String serviceId) {
        return this.clientFactory.getLoadBalancer(serviceId);
    }

    public static class RibbonServer implements ServiceInstance {
        private final String serviceId;
        private final Server server;
        private final boolean secure;
        private Map<String, String> metadata;

        public RibbonServer(String serviceId, Server server) {
            this(serviceId, server, false, Collections.emptyMap());
        }

        public RibbonServer(String serviceId, Server server, boolean secure, Map<String, String> metadata) {
            this.serviceId = serviceId;
            this.server = server;
            this.secure = secure;
            this.metadata = metadata;
        }

      //省略serviceInstance的getter()方法
}

```

# **RibbonLoadBalancerClient的主要工作步骤：**

## **1）处理LoadBalancerInterceptor的interceptor（）方法的excute（serviceName,LoadBalancerRequest<ClientHttpResponse>）**请求

1. 根据传入的服务名去获得具体的服务实例，

   ```java
     public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
         //获得负载均衡器，默认是ZoneAwareLoadBalancer,下面会告诉你为什么是他
           ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
         //获得服务实例，进入这个
           Server server = this.getServer(loadBalancer, hint);
           if (server == null) {
               throw new IllegalStateException("No instances available for " + serviceId);
           } else {
               //封装Server成RibbonServer
               RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
               return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
           }
       }
   ```

   我们发现**this.getServer(loadBalancer, hint);**里面有一个ILoadBalancer的参数，这个就是我们的负载负载均衡的负载均衡器。查看一下loadBalancer接口：

**loadBalancer.class**

```java

public interface ILoadBalancer {

	public void addServers(List<Server> newServers);
	

	public Server chooseServer(Object key);
	

	public void markServerDown(Server server);
	

	public List<Server> getServerList(boolean availableOnly);

    public List<Server> getReachableServers();

	public List<Server> getAllServers();
}

```

解读一下这个接口的主要方法：

![1588305209399](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588305209399.png)

我们查看一下这个接口的实现，整理一下类图：



![1588305403094](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588305403094.png)

可以看出**BaseLoadBalancer**实现了基础的负载均衡，**DynamicServerListLoadBalancer**和**ZoneAwareLoadBalancer**是在它基础上的拓展。但是在**RibbonClientConfiguration**中我们知道它和spring cloud整合的时候默认采用的就是**ZoneAwareLoadBalancer**这个负载均衡器。有图有真相吧，直接上图：

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config, ServerList<Server> serverList, ServerListFilter<Server> serverListFilter, IRule rule, IPing ping, ServerListUpdater serverListUpdater) 
{    return (ILoadBalancer)(this.propertiesFactory.isSet(ILoadBalancer.class, this.name) ? 
                            (ILoadBalancer)this.propertiesFactory.get(ILoadBalancer.class, config, this.name) : 
                            //看这里
new ZoneAwareLoadBalancer(config, rule, ping, serverList, serverListFilter, serverListUpdater));}
```

这是一个三目运算符，如果没有在配置文件中指定负载均衡器，那么就采用默认的负载均衡器**ZoneAwareLoadBalancer**，这个相信大家应该没问题。而且**ZoneAwareLoadBalancer**也相当于继承了**BaseLoadBalancer**.

记得做笔记，要不然绕着绕着就不知道在哪里了。

我们拿到了负载均衡器，是不是还没拿到我们服务实例呢？继续回到我们刚才的**this.getServer(loadBalancer, hint)**，进去这个方法 

```java
   protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
        return loadBalancer == null ? null : loadBalancer.chooseServer(hint != null ? hint : "default");
    }
```

是不是看到这里获得实例居然不是我们的**LoadBalancerClient的choose（）**而是**loadBalancer.chooseServer**,如果你忘记了就回到页顶。

好，我们进去一探究竟，这里**loadBalancer**对象是不是我们上面说的默认**ZoneAwareLoadBalancer**负载均衡器啊，真是环环相扣，所以我们进入ZoneAwareLoadBalancer，这个大家应该没有问题，所有实现我们不应该找接口吧，肯定要找实现类吧，好，我们进入ZoneAwareLoadBalancer的chooseServer（object）。

```java
 @Override
    public Server chooseServer(Object key) {
        //当服务实例只有一个的时候，是不是也没法使用负载均衡啊，再怎么均衡放回的都是一个，没必要了
        if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
            logger.debug("Zone aware logic disabled or there is only one zone");
            return super.chooseServer(key);
        }
        Server server = null;
        try {
            LoadBalancerStats lbStats = getLoadBalancerStats();
            Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
            logger.debug("Zone snapshots: {}", zoneSnapshot);
            if (triggeringLoad == null) {
                triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                        "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
            }

            if (triggeringBlackoutPercentage == null) {
                triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                        "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
            }
            Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
            logger.debug("Available zones: {}", availableZones);
            if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
                String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
                logger.debug("Zone chosen: {}", zone);
                if (zone != null) {
                    BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                    //上面巴拉巴拉一堆，不重要（因为我们也看不懂，反正就是以恶环境判断，日记啊这些），zone可以认为就是服务实例名字，那接下来的逻辑相信大家也懂，
                    server = zoneLoadBalancer.chooseServer(key);
                }
            }
        } catch (Exception e) {
            logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
        }
        if (server != null) {
            return server;
        } else {
            logger.debug("Zone avoidance logic is not invoked.");
            //这里也有一个#############，我们说过，该类间接的父类就是BaseLoadBalancer
            return super.chooseServer(key);
        }
    }
```

少bb，我们直接进入chooseServer，毫无疑问，我们进入的是**BaseLoadBalancer**负载均衡器的chooseServer（Object），

```java
    public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
                //看这里看这里
                return rule.choose(key);
            } catch (Exception e) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
                return null;
            }
        }
    }
```

我们翻查一下该类，IRule就是我们的负载均衡算法，负载均衡算法有：随机负载均衡，轮询负载均衡，重试负载均衡，相应的实现类大家自己查看idea中进入IRule接口ctrl+h就可以看到。

![1588307482057](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588307482057.png)

那我们这里的rule是什么呢？

![1588307413239](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588307413239.png)

英语稍微没过四级都应该知道这个单词我们要百度！！！！没错这个就是轮询算法，轮询就是分你一个女朋友，我一个女朋友，然后你一个，我又一个。总之就没偏袒谁。（还记得我们绕到哪里了吗），是的，讲了这么久我们连服务实例都没拿到。

继续进入**rule.choose(key);**,告诉我，我们要进入哪个实现类？？？？那肯定是轮询（RoundRobinRule）的啊，

找到我们chooose重载方法的最终要走的，

```java
  public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
            //获取或有能接触到的服务实例，这里负载均衡器已经是由对应的服务名了，只是不知道找
            //哪台机器而已，这里的server包含主机，端口啊之类信息
            List<Server> reachableServers = lb.getReachableServers();
            //获得所有的服务实例
            List<Server> allServers = lb.getAllServers();
            //可用的服务实例多少
            int upCount = reachableServers.size();
             //全部的服务实例多少
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }
			//这里便是轮询负载均衡的算法了，我们每一台机器，都对应由一个index下标，因为我们上面
            //是保存再list里面，也就是有序数组，获得下标就相当于获取到具体的服务实例机器
            int nextServerIndex = incrementAndGetModulo(serverCount);
            //这里就获取到了
            server = allServers.get(nextServerIndex);

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }

```

我们这里的**ILoadBalancer** 实现类已经是**BaseLoadBalancer**,因为ZoneAwareLoadBalancer**和**BaseLoadBalancer的底层负载均衡肯定是用BaseLoadBalancer啊，他们两个只是拓展，那你们搞好了拓展，那基础的负载均衡肯定是由他们爸爸（BaseLoadBalancer,）来干了吧，具体的步骤说明上面代码有写。

那我们看一下负载均衡算法是如何轮询发女友的：



```java
    private int incrementAndGetModulo(int modulo) {
        //这里自循坏，不是死循坏
        //nextServerCyclicCounter是个原子操作，不会引发竞态条件，线程是安全的
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }
```

服务实例index = (当前的已经请求的过请求数+1)%所有集群中服务实例的台数。

**小提示：**

**只要是负载均衡的restTemplate,那么他的实例获取最终肯定会由负载均衡类来获取对应的server.**

终于终于我们能拿到服务实例啊（感动）

为了不用你们翻，我把刚才没之执行完的excute()拿了过来

```java
  public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
      //获得负载均衡器，上面已经跑了
        ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
      //获得服务实例，这个也跑了
        Server server = this.getServer(loadBalancer, hint);
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        } else {
            //封装Server成RibbonServer
            RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
            return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
        }
    }
```



## 2）接下来我们就是封装Service为RibbonServer，

Ribbon对象除了存储服务实例外，还增加了服务名serviceid和使用需要使用https安全请求等信息。同时它也是ServiceInstance的实现类。区分一下，serviceid是服务名，InstanceId是实例名，一个服务可以对应多个实例。

Server和ServiceInstance没有直接的关系，只是他们由一些相似的属性，类似po和bo的关系吧。

封装完了我们的Server成RibbonServer,接下来走另一个重载的方法

**this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request)**

```java
    public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
        Server server = null;
        if (serviceInstance instanceof RibbonLoadBalancerClient.RibbonServer) {
            server = ((RibbonLoadBalancerClient.RibbonServer)serviceInstance).getServer();
        }

        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        } else {
            RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
            RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

            try {
                //走这个请求
                T returnVal = request.apply(serviceInstance);
                statsRecorder.recordStats(returnVal);
                return returnVal;
            } catch (IOException var8) {
                statsRecorder.recordStats(var8);
                throw var8;
            } catch (Exception var9) {
                statsRecorder.recordStats(var9);
                ReflectionUtils.rethrowRuntimeException(var9);
                return null;
            }
        }
    }
```

**request.apply(serviceInstance)**就向我们刚才获取到的那个具体的服务实例发送请求，从而实现开始以服务名为host的URI到host:post的形式的实际访问，（我们的RibbonServer里面就包含有获得的Server,里面有host，post信息。**这些信息是负载均衡器从eureka的服务发现中获取的**。ribbonServer是ServiceInstance的实现类别忘记了噢）

看一下两个类

**ServiceInstance.class**

```java
public interface ServiceInstance {
	default String getInstanceId() {
		return null;
	}
	String getServiceId();
	String getHost();
	int getPort();
	boolean isSecure();
	URI getUri();
	Map<String, String> getMetadata();
	default String getScheme() {
		return null;
	}

}
```

**RibbonServer.class**

```java
 public static class RibbonServer implements ServiceInstance {
        private final String serviceId;
        private final Server server;
        private final boolean secure;
        private Map<String, String> metadata;
       }
```

我们都是直接访问服务名去获得不用ip端口的服务实例。记得我们eureka中说到，Ribbon维护的服务清单在和Eureka结合之后，服务清单被重写覆盖了。

我们看一下**request.apply(serviceInstance)**这个方法，这个方法有点特殊，它没有实现类这时候我们发现apply方法是LoadBalancerRequest接口中的一个方法，且LoadBalancerRequest接口没有实现类，那么apply方法的实现是在哪里实现的呢？此时我们发现LoadBalancerRequest中的apply方法在执行的时候，这个request是从LoadBalancerInterceptor拦截器里边传来的，我们再回到LoadBalancerInterceptor的intercept方法中，在这个方法中最终通过`requestFactory.createRequest(request, body, execution)`来创建一个LoadBalancerRequest，在这个方法中，我们找到了apply（不用版本可能改了，但是思路是一致的）的实现：

```javascript
public LoadBalancerRequest<ClientHttpResponse> createRequest(final HttpRequest request,
        final byte[] body, final ClientHttpRequestExecution execution) {
    return new LoadBalancerRequest<ClientHttpResponse>() {

        @Override
        public ClientHttpResponse apply(final ServiceInstance instance)
                throws Exception {
            HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
            if (transformers != null) {
                for (LoadBalancerRequestTransformer transformer : transformers) {
                    serviceRequest = transformer.transformRequest(serviceRequest, instance);
                }
            }
            return execution.execute(serviceRequest, body);
        }

    };
}
```

这里执行的时候，把我们的request包装成了ServiceRequestWrapper(),且该类重写了getURI()

```java
public class ServiceRequestWrapper extends HttpRequestWrapper {

	private final ServiceInstance instance;

	private final LoadBalancerClient loadBalancer;

	public ServiceRequestWrapper(HttpRequest request, ServiceInstance instance,
			LoadBalancerClient loadBalancer) {
		super(request);
		this.instance = instance;
		this.loadBalancer = loadBalancer;
	}

	@Override
	public URI getURI() {
        //调用我们RibbonLoadBalancerClient的reconstructURI方法发送请求
		URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
		return uri;
        //看到reconstructURI是不是看到了希望了，嗯哼
	}

}

```

![1588315858388](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1588315858388.png)

**InterceptingClientHttpRequest.class**

```java
        public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
            if (this.iterator.hasNext()) {
                ClientHttpRequestInterceptor nextInterceptor = (ClientHttpRequestInterceptor)this.iterator.next();
                return nextInterceptor.intercept(request, body, this);
            } else {
                HttpMethod method = request.getMethod();
                Assert.state(method != null, "No standard HTTP method");
                
                //############看这里###############
                ClientHttpRequest delegate = InterceptingClientHttpRequest.this.requestFactory.createRequest(request.getURI(), method);
                request.getHeaders().forEach((key, value) -> {
                    delegate.getHeaders().addAll(key, value);
                });
                if (body.length > 0) {
                    if (delegate instanceof StreamingHttpOutputMessage) {
                        StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage)delegate;
                        streamingOutputMessage.setBody((outputStream) -> {
                            StreamUtils.copy(body, outputStream);
                        });
                    } else {
                        StreamUtils.copy(body, delegate.getBody());
                    }
                }

                return delegate.execute();
            }
        }
```

可以看到在

```
ClientHttpRequest delegate = InterceptingClientHttpRequest.this.requestFactory.createRequest(request.getURI(), method)
```

;的时候，这里的request.getUri()已经是我们包装过的了，所以这里会使用重写的getURI函数。此时，它就会调用RibbonLoadBalancerClient中实现的reconstructURI来组织具体请求的服务实例地址了。

ServiceRequestWrapper.class,在上面可以看到

URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());

```java
    public URI reconstructURI(ServiceInstance instance, URI original) {
        Assert.notNull(instance, "instance can not be null");
        String serviceId = instance.getServiceId();
        RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
        URI uri;
        Server server;
        if (instance instanceof RibbonLoadBalancerClient.RibbonServer) {
            RibbonLoadBalancerClient.RibbonServer ribbonServer = (RibbonLoadBalancerClient.RibbonServer)instance;
            server = ribbonServer.getServer();
            uri = RibbonUtils.updateToSecureConnectionIfNeeded(original, ribbonServer);
        } else {
            server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
            IClientConfig clientConfig = this.clientFactory.getClientConfig(serviceId);
            ServerIntrospector serverIntrospector = this.serverIntrospector(serviceId);
            uri = RibbonUtils.updateToSecureConnectionIfNeeded(original, clientConfig, serverIntrospector, server);
        }

        return context.reconstructURIWithServer(server, uri);
    }
```

从reconstructUri函数中，我们可以看到，它通过ServiceInstance实例对象的serviceId,从SpringClientFactory类的对象clientFactory中获取对应的serviceid的负载均衡的上下文RibbonLoadBalancerContext对象。然后根据ServiceInstance中的信息来构建具体服务实例子信息的Server对象，并使用RibbonLoadBalancerContext对象的reconstructURIWithServer(server, uri)来构建服务实例的URI

```
 public URI reconstructURIWithServer(Server server, URI original) {
        String host = server.getHost();
        int port = server.getPort();
        String scheme = server.getScheme();
        
        if (host.equals(original.getHost()) 
                && port == original.getPort()
                && scheme == original.getScheme()) {
            return original;
        }
        if (scheme == null) {
            scheme = original.getScheme();
        }
        if (scheme == null) {
            scheme = deriveSchemeAndPortFromPartialUri(original).first();
        }

        try {
            StringBuilder sb = new StringBuilder();
            sb.append(scheme).append("://");
            if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
                sb.append(original.getRawUserInfo()).append("@");
            }
            sb.append(host);
            if (port >= 0) {
                sb.append(":").append(port);
            }
            sb.append(original.getRawPath());
            if (!Strings.isNullOrEmpty(original.getRawQuery())) {
                sb.append("?").append(original.getRawQuery());
            }
            if (!Strings.isNullOrEmpty(original.getRawFragment())) {
                sb.append("#").append(original.getRawFragment());
            }
            URI newURI = new URI(sb.toString());
            return newURI;            
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }
```

获得对应的url之后就

```
T returnVal = request.apply(serviceInstance);
statsRecorder.recordStats(returnVal);//ok了
return returnVal;
```