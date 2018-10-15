---
title: kylin master-slave同步原理及问题排查
date: 2017-07-06 15:11:38
tags:
    - Kylin
    - infrastructure
    - 源码
    - 成长
---

### 背景

最近俩个月，团队整个数据基础架构慢慢转移到kylin上面来。而kylin也不负众望，对于一些复杂的聚合查询响应速度远超于hive。随着数据量的上来，kylin的单体部署逐渐无法支撑大量的并行读写任务。于是，自然而然的考虑到kylin的读写分离。一写多读，正好也符合kylin官方文档上的cluster架构。然而在实际的使用中也出现了一些问题:

- 主节点更新了schema而从节点未sync
- 从节点中部分sync成功，而不是全部

而很明显的是kylin中所有的数据，包括所有元数据都是落地在HBase中的，那唯一导致节点间数据不一致的可能就只有各个节点都有本地缓存的情况了。为了理解原理方便debug，我对kylin master-slave的同步原理做了一些源代码层面的剖析。


### 原理剖析

#### 主从配置方式

关于配置的格式，不得不吐槽官方文档的滑水。并没有给出详细的节点配置格式，查阅相关源码才发现正确的配置格式：

```
//kylin.properties下面的配置，根据源码，配置的格式为：user:pwd@host:port
kylin.server.cluster-servers=user:password@host:port,user:password@host:port,user:password@host:port

```

#### 流程解析

![流程解析](http://ol7zjjc80.bkt.clouddn.com/master-slave-kylin.png)

#### 源码解析

- 先来看看整个同步机制的核心BroadCaster类的实现

  ```
  //Broadcaster的构造函数
  private Broadcaster(final KylinConfig config) {
        this.config = config;
        //获取kylin.properties中"kylin.server.cluster-servers"配置的值
        //也就是集群中所有节点的配置了
        final String[] nodes = config.getRestServers();
        if (nodes == null || nodes.length < 1) {
            logger.warn("There is no available rest server; check the 'kylin.server.cluster-servers' config");
            broadcastEvents = null; // disable the broadcaster
            return;
        }
        logger.debug(nodes.length + " nodes in the cluster: " + Arrays.toString(nodes));

        //开一个单线程，不间断的循环从broadcastEvents队列里面获取注册的事件。
        Executors.newSingleThreadExecutor(new DaemonThreadFactory()).execute(new Runnable() {
            @Override
            public void run() {
                final List<RestClient> restClients = Lists.newArrayList();
                for (String node : config.getRestServers()) {
                    //根据配置的节点信息注册RestClient
                    restClients.add(new RestClient(node));
                }
                final ExecutorService wipingCachePool = Executors.newFixedThreadPool(restClients.size(), new DaemonThreadFactory());
                while (true) {
                    try {
                        final BroadcastEvent broadcastEvent = broadcastEvents.takeFirst();
                        logger.info("Announcing new broadcast event: " + broadcastEvent);
                        for (final RestClient restClient : restClients) {
                            wipingCachePool.execute(new Runnable() {
                                @Override
                                public void run() {
                                    try {
                                        restClient.wipeCache(broadcastEvent.getEntity(), broadcastEvent.getEvent(), broadcastEvent.getCacheKey());
                                    } catch (IOException e) {
                                        logger.warn("Thread failed during wipe cache at " + broadcastEvent, e);
                                    }
                                }
                            });
                        }
                    } catch (Exception e) {
                        logger.error("error running wiping", e);
                    }
                }
            }
        });
    }

  ```

  通过Broadcaster的构造函数其实就能清楚整个同步过程的大概逻辑了。无非就是启动一个线程去轮询阻塞队列里面的元素，有的话就消费下来广播到其他从节点从而达到清理缓存的目的。

- 再来看看广播的实际逻辑实现,基本封装在RestClient中

  ```

 //此处是根据配置的节点信息正则匹配："user:pwd@host:port"
 public RestClient(String uri) {
    Matcher m = fullRestPattern.matcher(uri);
    if (!m.matches())
        throw new IllegalArgumentException("URI: " + uri + " -- does not match pattern " + fullRestPattern);

    String user = m.group(1);
    String pwd = m.group(2);
    String host = m.group(3);
    String portStr = m.group(4);
    int port = Integer.parseInt(portStr == null ? "7070" : portStr);

    init(host, port, user, pwd);
  }

  ```

  根据配置的节点信息实例化RestClient，然后在init方法中，拼接wipe cache的url

  ```
  private void init(String host, int port, String userName, String password) {
    this.host = host;
    this.port = port;
    this.userName = userName;
    this.password = password;
    //拼接rest接口
    this.baseUrl = "http://" + host + ":" + port + "/kylin/api";

    client = new DefaultHttpClient();

    if (userName != null && password != null) {
        CredentialsProvider provider = new BasicCredentialsProvider();
        UsernamePasswordCredentials credentials = new UsernamePasswordCredentials(userName, password);
        provider.setCredentials(AuthScope.ANY, credentials);
        client.setCredentialsProvider(provider);
    }
  }

  ```
  发现kylin所有的交互接口基本上底层都是调用的自己的rest接口，它自己所谓的jdbc的查询方式其实也只是在rest接口上封装了一层，底层还是http请求。可谓是挂羊头卖狗肉了。看看RestClient中怎么去通知其他节点wipe cache的

  ```
  public void wipeCache(String entity, String event, String cacheKey) throws IOException {
     String url = baseUrl + "/cache/" + entity + "/" + cacheKey + "/" + event;
     HttpPut request = new HttpPut(url);

     try {
         HttpResponse response = client.execute(request);
         String msg = EntityUtils.toString(response.getEntity());

         if (response.getStatusLine().getStatusCode() != 200)
             throw new IOException("Invalid response " + response.getStatusLine().getStatusCode() + " with cache wipe url " + url + "\n" + msg);
     } catch (Exception ex) {
         throw new IOException(ex);
     } finally {
         request.releaseConnection();
     }
   }

  ```
  已经很明了了，就是调的rest接口：/kylin/api/cache/{entity}/{cacaheKey}/{event}

- 当slave节点接收到wipeCache的指令时的处理逻辑如下：

  ```
  public void notifyMetadataChange(String entity, Event event, String cacheKey) throws IOException {
       Broadcaster broadcaster = Broadcaster.getInstance(getConfig());

       //这里会判断当前节点是否注册为listener了，如果注册了，此逻辑会被ignored
       broadcaster.registerListener(cacheSyncListener, "cube");

       broadcaster.notifyListener(entity, event, cacheKey);
   }

   //注册listener的逻辑
   public void registerListener(Listener listener, String... entities) {
    synchronized (CACHE) {
        // ignore re-registration
        List<Listener> all = listenerMap.get(SYNC_ALL);
        if (all != null && all.contains(listener)) {
            return;
        }

        for (String entity : entities) {
            if (!StringUtils.isBlank(entity))
                addListener(entity, listener);
        }
        //注册几种事件类型
        addListener(SYNC_ALL, listener);
        addListener(SYNC_PRJ_SCHEMA, listener);
        addListener(SYNC_PRJ_DATA, listener);
    }
  }
  ```

  notifyListener主要就是对所有事件处理逻辑的划分，根据事件类型选择处理逻辑，一般sheme的更新走的是默认逻辑

  ```
  public void notifyListener(String entity, Event event, String cacheKey) throws IOException {
       synchronized (CACHE) {
           List<Listener> list = listenerMap.get(entity);
           if (list == null)
               return;

           logger.debug("Broadcasting metadata change: entity=" + entity + ", event=" + event + ", cacheKey=" + cacheKey + ", listeners=" + list);

           // prevents concurrent modification exception
           list = Lists.newArrayList(list);
           switch (entity) {
           case SYNC_ALL:
               for (Listener l : list) {
                   l.onClearAll(this);
               }
               clearCache(); // clear broadcaster too in the end
               break;
           case SYNC_PRJ_SCHEMA:
               ProjectManager.getInstance(config).clearL2Cache();
               for (Listener l : list) {
                   l.onProjectSchemaChange(this, cacheKey);
               }
               break;
           case SYNC_PRJ_DATA:
               ProjectManager.getInstance(config).clearL2Cache(); // cube's first becoming ready leads to schema change too
               for (Listener l : list) {
                   l.onProjectDataChange(this, cacheKey);
               }
               break;
           //大部分的走向
           default:
               for (Listener l : list) {
                   l.onEntityChange(this, entity, event, cacheKey);
               }
               break;
           }

           logger.debug("Done broadcasting metadata change: entity=" + entity + ", event=" + event + ", cacheKey=" + cacheKey);
       }
   }

  ```

  看到default分支会执行onEntityChange这个方法，看一下这个方法干的是什么

  ```
  private Broadcaster.Listener cacheSyncListener = new Broadcaster.Listener() {
     @Override
     public void onClearAll(Broadcaster broadcaster) throws IOException {
         removeAllOLAPDataSources();
         cleanAllDataCache();
     }

     @Override
     public void onProjectSchemaChange(Broadcaster broadcaster, String project) throws IOException {
         removeOLAPDataSource(project);
         cleanDataCache(project);
     }

     @Override
     public void onProjectDataChange(Broadcaster broadcaster, String project) throws IOException {
         removeOLAPDataSource(project); // data availability (cube enabled/disabled) affects exposed schema to SQL
         cleanDataCache(project);
     }

     @Override
     public void onEntityChange(Broadcaster broadcaster, String entity, Event event, String cacheKey) throws IOException {
         if ("cube".equals(entity) && event == Event.UPDATE) {
             final String cubeName = cacheKey;
             new Thread() { // do not block the event broadcast thread
                 public void run() {
                     try {
                         Thread.sleep(1000);
                         cubeService.updateOnNewSegmentReady(cubeName);
                     } catch (Throwable ex) {
                         logger.error("Error in updateOnNewSegmentReady()", ex);
                     }
                 }
             }.start();
         }
     }
   };

  ```

  看到对于cache的同步是单独实现了一个listener的，Event为update的时候，会单独启动一个线程去执行刷新缓存操作

### 加入简单的重试逻辑

由于目前对于同步失败的猜想是目标服务短暂不可用（响应超时或者处于失败重启阶段），于是我只是单纯的将失败的任务重新塞入broadcastEvents队列尾部供再一次调用。当然这种操作过于草率和暴力，却也是验证猜想最简单快速的方式。

```

 for (final RestClient restClient : restClients) {
            wipingCachePool.execute(new Runnable() {
              @Override
              public void run() {
                try {
                  restClient.wipeCache(broadcastEvent.getEntity(), broadcastEvent.getEvent(),
                      broadcastEvent.getCacheKey());
                } catch (IOException e) {
                  logger
                      .warn("Thread failed during wipe cache at {}, error msg: {}", broadcastEvent,
                          e.getMessage());
                  try {
                    //这里重新塞入队列尾部，等待重新执行
                    broadcastEvents.putLast(broadcastEvent);
                    logger.info("put failed broadcastEvent to queue. broacastEvent: {}",
                        broadcastEvent);
                  } catch (InterruptedException ex) {
                    logger.warn("error reentry failed broadcastEvent to queue, broacastEvent:{}, error: {} ",
                        broadcastEvent, ex);
                  }
                }
              }
            });
          }
        }

```

编译部署之后，日志中出现了如下错误：

```
Thread failed during wipe cache at java.lang.IllegalStateException: Invalid use of BasicClientConnManager: connection still allocated.

```

比较意外，不过也终于发现了问题的所在。Kylin在启动的时候会按照配置的nodes实例化一次RestClient，之后就直接从缓存中拿了，而kylin用的DefaultHttpClient每次只允许一次请求，请求完必须释放链接，否则无法复用HttpClient。所以需要修改wipeCache方法的逻辑如下:

```
    public void wipeCache(String entity, String event, String cacheKey) throws IOException {
        String url = baseUrl + "/cache/" + entity + "/" + cacheKey + "/" + event;
        HttpPut request = new HttpPut(url);

        HttpResponse response =null;
        try {
            response = client.execute(request);
            String msg = EntityUtils.toString(response.getEntity());

            if (response.getStatusLine().getStatusCode() != 200)
                throw new IOException("Invalid response " + response.getStatusLine().getStatusCode() + " with cache wipe url " + url + "\n" + msg);
        } catch (Exception ex) {
            throw new IOException(ex);
        } finally {
            //确保释放连接
            if(response!=null) {
              EntityUtils.consume(response.getEntity());
            }
            request.releaseConnection();
        }
    }

```
