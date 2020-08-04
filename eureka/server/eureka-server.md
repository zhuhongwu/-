1.入口注解

```
@EnableEurekaServer
注解用于激活（active）eureka server的配置类org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration配置。
```

2.EurekaServerAutoConfiguration

```
@Configuration //标记成为配置类
@Import(EurekaServerInitializerConfiguration.class)//引入EurekaServerInitializerConfiguration
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
      InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter
```

EurekaServerAutoConfiguration注册了一系列的bean

```
@Configuration
protected static class EurekaServerConfigBeanConfiguration {

	//注册加载eureka.server开头的相关配置信息
   @Bean
   @ConditionalOnMissingBean
   public EurekaServerConfig eurekaServerConfig(EurekaClientConfig clientConfig) {
      EurekaServerConfigBean server = new EurekaServerConfigBean();
      if (clientConfig.shouldRegisterWithEureka()) {
         // Set a sensible default if we are supposed to replicate
         server.setRegistrySyncRetries(5);
      }
      return server;
   }
}

@Bean
@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
public EurekaController eurekaController() {	
//springcloud 提供的一些对外接口，用来加载eureka信息（仪表盘信息）
   return new EurekaController(this.applicationInfoManager);
}

...省略一部分代码


	@Bean//对等节点感知实例注册器（集群模式下注册服务使用到的注册器）集群中各个节点是对等的，没有主从之分
	public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
			ServerCodecs serverCodecs) {
		this.eurekaClient.getApplications(); // force initialization
		return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
				serverCodecs, this.eurekaClient,
				this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
				this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
	}

	@Bean//配置服务节点信息，这里的作用主要是为了配置Eureka的peer节点，也就是说当有收到有节点注册上来的时候，需要通知给哪些节点
	//注入PeerEurekaNodes，辅助封装对等节点相关信息和操作，比如更新集群当中的对等节点
	@ConditionalOnMissingBean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
			ServerCodecs serverCodecs) {
		return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
				this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
	}
	
	
	
	

...省略一部分代码

	@Bean//配置Eurekaserver上下文
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
			PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
				registry, peerEurekaNodes, this.applicationInfoManager);
	}

	@Bean// 初始化Eureka-server，会同步其他注册中心的数据到当前注册中心
	public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
			EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(this.applicationInfoManager,
				this.eurekaClientConfig, this.eurekaServerConfig, registry,
				serverContext);
	}
    
    
    @Bean
	public javax.ws.rs.core.Application jerseyApplication(Environment environment,
			ResourceLoader resourceLoader) {

		ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
				false, environment);

		// Filter to include only classes that have a particular annotation.
		//
		provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
		provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

		// Find classes in Eureka packages (or subpackages)
		//
		Set<Class<?>> classes = new HashSet<>();
		for (String basePackage : EUREKA_PACKAGES) {
			Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
			for (BeanDefinition bd : beans) {
				Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
						resourceLoader.getClassLoader());
				classes.add(cls);
			}
		}
.....



		// Construct the Jersey ResourceConfig
		//创建Jersey  类似于springmvc的一个框架
		Map<String, Object> propsAndFeatures = new HashMap<>();
		propsAndFeatures.put(
				// Skip static content used by the webapp
				ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
				EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

		DefaultResourceConfig rc = new DefaultResourceConfig(classes);
		rc.setPropertiesAndFeatures(propsAndFeatures);

		return rc;
	}
```

3.EurekaServerInitializerConfiguration

org.springframework.cloud.netflix.eureka.server.EurekaServerInitializerConfiguration

在实例化EurekaServerAutoConfiguration之前  已经实例化了EurekaServerInitializerConfiguration，再来看看EurekaServerInitializerConfiguration



```
//会开启一个线程  执行一些初始化以及事件发布的工作
@Override
public void start() {
   new Thread(new Runnable() {
      @Override
      public void run() {
         try {
            //TODO: is this class even needed now?
            //初始化eureka-server，启动eureka-server EurekaServerBootstrap
            eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
            log.info("Started Eureka Server");
			 // 发布EurekaServer的注册事件	
            publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
            //设置启动状态
            EurekaServerInitializerConfiguration.this.running = true;
          
	        // 发送Eureka Start 事件 ， 其他还有各种事件，我们可以监听这种事件，然后做一些特定的业务需求，后面会讲到。
            publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
         }
         catch (Exception ex) {
            // Help!
            log.error("Could not initialize Eureka servlet context", ex);
         }
      }
   }).start();
}
```

4.EurekaServerBootstrap(org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap)

```
public void contextInitialized(ServletContext context) {
   try {
   	  //初始化eureka环境
      initEurekaEnvironment();
      //初始化eurekaserver上下文
      initEurekaServerContext();

      context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
   }
   catch (Throwable e) {
      log.error("Cannot bootstrap eureka server :", e);
      throw new RuntimeException("Cannot bootstrap eureka server :", e);
   }
}

protected void initEurekaServerContext() throws Exception {
		// For backward compatibility
		JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
				XStream.PRIORITY_VERY_HIGH);
		XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
				XStream.PRIORITY_VERY_HIGH);

		if (isAws(this.applicationInfoManager.getInfo())) {
			this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
					this.eurekaClientConfig, this.registry, this.applicationInfoManager);
			this.awsBinder.start();
		}
		//为非ioc容器提供获取serverContext对象的接口
		EurekaServerContextHolder.initialize(this.serverContext);

		log.info("Initialized server context");

		// Copy registry from neighboring eureka node
		//从其他Server节点拷贝注册信息过来同步，每一个server对其他server来说也是客户端
		int registryCount = this.registry.syncUp();
		//更改实例状态为up并对外提供服务
		this.registry.openForTraffic(this.applicationInfoManager, registryCount);

		// Register all monitoring statistics.
		//注册统计器
		EurekaMonitors.registerAllStats();
	}
 
```

5.com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#syncUp

```
@Override
public int syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;

    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        if (i > 0) {
            try {
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
        Applications apps = eurekaClient.getApplications();//获取其他server的注册信息
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {//远程获取到的信息放入自己的注册信息中
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```

6.com.netflix.eureka.registry.AbstractInstanceRegistry#register

```
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try {
        read.lock();
        //从注册表中获取服务  
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        REGISTER.increment(isReplication);
        
       //判断有没有服务 没有服务就new
        if (gMap == null) {
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        //判断服务当中有没有实例  没有实例就创建lease
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        if (existingLease != null && (existingLease.getHolder() != null)) {
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                        " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            // The lease does not exist and hence it is a new registration
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        gMap.put(registrant.getId(), lease);
        synchronized (recentRegisteredQueue) {
            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));
        }
        // This is where the initial state transfer of overridden status happens
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                            + "overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        // Set the status based on the overridden status rules
        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);

        // If the lease is registered with UP status, set lease service up timestamp
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        registrant.setActionType(ActionType.ADDED);
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        registrant.setLastUpdatedTimestamp();
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```

7.eureka server的服务注册接口

ApplicationResource类的addInstance()⽅法中代码： registry.register(info,
"true".equals(isReplication));  



com.netflix.eureka.resources.ApplicationResource#addInstance



```
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
    // validate that the instanceinfo contains all the necessary required fields
    if (isBlank(info.getId())) {
        return Response.status(400).entity("Missing instanceId").build();
    } else if (isBlank(info.getHostName())) {
        return Response.status(400).entity("Missing hostname").build();
    } else if (isBlank(info.getIPAddr())) {
        return Response.status(400).entity("Missing ip address").build();
    } else if (isBlank(info.getAppName())) {
        return Response.status(400).entity("Missing appName").build();
    } else if (!appName.equals(info.getAppName())) {
        return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
    } else if (info.getDataCenterInfo() == null) {
        return Response.status(400).entity("Missing dataCenterInfo").build();
    } else if (info.getDataCenterInfo().getName() == null) {
        return Response.status(400).entity("Missing dataCenterInfo Name").build();
    }

    // handle cases where clients may be registering with bad DataCenterInfo with missing data
    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
    if (dataCenterInfo instanceof UniqueIdentifier) {
        String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
        if (isBlank(dataCenterInfoId)) {
            boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
            if (experimental) {
                String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                return Response.status(400).entity(entity).build();
            } else if (dataCenterInfo instanceof AmazonInfo) {
                AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                if (effectiveId == null) {
                    amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                }
            } else {
                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
            }
        }
    }

    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```