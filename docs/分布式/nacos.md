官网https://nacos.io/zh-cn/index.html

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

## 源码解析

```java
ConfigService configService=NacosFactory.createConfigService(properties);
String content=configService.getConfig(dataId,groupId,3000);
```

agent->代理

http的方式去发起请求.

clientWorker . -> 具体的工作有关

```java
agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
agent.start();
worker = new ClientWorker(agent, configFilterChainManager, properties);
```

### 长轮训的时间间隔

客户端会有一个长轮训的任务去检查服务器端的配置是否发生了变化，如果发生了变更，那么客户端会拿到变更的 groupKey 再根据 groupKey 去获取配置项的最新值更新到本地的缓存以及文件中，那么这种每次都靠客户端去请求，那请求的时间间隔设置多少合适呢？

如果间隔时间设置的太长的话有可能无法及时获取服务端的变更，如果间隔时间设置的太短的话，那么频繁的请求对于服务端来说无疑也是一种负担，所以最好的方式是客户端每隔一段长度适中的时间去服务端请求，而在这期间如果配置发生变更，服务端能够主动将变更后的结果推送给客户端，这样既能保证客户端能够实时感知到配置的变化，也降低了服务端的压力。 我们来看看nacos设置的间隔时间是多久

### 长轮训的概念

那么在讲解原理之前，先给大家解释一下什么叫长轮训

客户端发起一个请求到服务端，服务端收到客户端的请求后，并不会立刻响应给客户端，而是先把这个请求hold住，然后服务端会在hold住的这段时间检查数据是否有更新，如果有，则响应给客户端，如果一直没有数据变更，则达到一定的时间（长轮训时间间隔）才返回。

长轮训典型的场景有： 扫码登录、扫码支付。

![image-20190812135251658](http://ww2.sinaimg.cn/large/006tNc79ly1g5wunl9araj31fu0nigvc.jpg)

### 客户端长轮训

在ClientWorker这个类里面，找到 checkUpdateConfigStr 这个方法，这里面就是去服务器端查询发生变化的groupKey。

```java
/**
     * 从Server获取值变化了的DataID列表。返回的对象里只有dataId和group是有效的。 保证不返回NULL。
     */
    List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) throws IOException {

        List<String> params = Arrays.asList(Constants.PROBE_MODIFY_REQUEST, probeUpdateString);

        List<String> headers = new ArrayList<String>(2);
        headers.add("Long-Pulling-Timeout");
        headers.add("" + timeout);

        // told server do not hang me up if new initializing cacheData added in
        if (isInitializingCacheList) {
            headers.add("Long-Pulling-Timeout-No-Hangup");
            headers.add("true");
        }

        if (StringUtils.isBlank(probeUpdateString)) {
            return Collections.emptyList();
        }

        try {
            HttpResult result = agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params,
                agent.getEncode(), timeout);

            if (HttpURLConnection.HTTP_OK == result.code) {
                setHealthServer(true);
                return parseUpdateDataIdResponse(result.content);
            } else {
                setHealthServer(false);
                LOGGER.error("[{}] [check-update] get changed dataId error, code: {}", agent.getName(), result.code);
            }
        } catch (IOException e) {
            setHealthServer(false);
            LOGGER.error("[" + agent.getName() + "] [check-update] get changed dataId exception", e);
            throw e;
        }
        return Collections.emptyList();
    }
```

这个方法最终会发起http请求，注意这里面有一个 `timeout` 的属性

```java
HttpResult result = agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params,agent.getEncode(), timeout);
```

timeout是在init这个方法中赋值的，默认情况下是30秒，可以通过configLongPollTimeout进行修改

```java
private void init(Properties properties) {

        timeout = Math.max(NumberUtils.toInt(properties.getProperty(PropertyKeyConst.CONFIG_LONG_POLL_TIMEOUT),
            Constants.CONFIG_LONG_POLL_TIMEOUT), Constants.MIN_CONFIG_LONG_POLL_TIMEOUT);

        taskPenaltyTime = NumberUtils.toInt(properties.getProperty(PropertyKeyConst.CONFIG_RETRY_TIME), Constants.CONFIG_RETRY_TIME);

        enableRemoteSyncConfig = Boolean.parseBoolean(properties.getProperty(PropertyKeyConst.ENABLE_REMOTE_SYNC_CONFIG));
    }
```

所以从这里得出的一个基本结论是

> 客户端发起一个轮询请求，超时时间是30s。 那么客户端为什么要等待30s才超时呢？不是越快越好吗？

### 客户端长轮训的时间间隔

我们可以在nacos的日志目录下 $NACOS_HOME/nacos/logs/config-client-request.log 文件

```verilog
2019-08-04 13:22:19,736|0|nohangup|127.0.0.1|polling|1|55|0
2019-08-04 13:22:49,443|29504|timeout|127.0.0.1|polling|1|55
2019-08-04 13:23:18,983|29535|timeout|127.0.0.1|polling|1|55
2019-08-04 13:23:48,493|29501|timeout|127.0.0.1|polling|1|55
2019-08-04 13:24:18,003|29500|timeout|127.0.0.1|polling|1|55
2019-08-04 13:24:47,509|29501|timeout|127.0.0.1|polling|1|55
```

可以看到一个现象，在配置没有发生变化的情况下，客户端会等29.5s+以上，才请求到服务器端的结果。然后客户端拿到服务器端的结果之后，在做后续的操作。

如果在配置变更的情况下，由于客户端基于长轮训的连接保持，所以返回的时间会非常的短，我们可以
做个小实验，在nacos console中频繁修改数据然后再观察一下

`config-client-request.log` 的变化

```verilog
2019-08-04 13:30:17,022|3|null|127.0.0.1|get|example|DEFAULT_GROUP||e10e4d5973c497e490a8d7a
9e4e9be64|unknown
2019-08-04 13:30:20,807|10|true|0:0:0:0:0:0:0:1|publish|example|DEFAULT_GROUP||81360b7e732a5dbb37d62d81cebb85d2|null
2019-08-04 13:30:20,843|0|inadvance|127.0.0.1|polling|1|55|example+DEFAULT_GROUP
2019-08-04 13:30:20,848|1|null|127.0.0.1|get|example|DEFAULT_GROUP||81360b7e732a5dbb37d62d8
1cebb85d2|unknown
```

![image-20190812205710316](http://ww4.sinaimg.cn/large/006tNc79gy1g5x6x06x4yj322u0h6wm0.jpg)

### 服务端的处理

分析完客户端之后，随着好奇心的驱使，服务端是如何处理客户端的请求的？那么同样，我们需要思考几个问题

客户端的长轮训响应时间受到哪些因素的影响

客户端的超时时间为什么要设置30s

客户端发送的请求地址是: /v1/cs/configs/listener 找到服务端对应的方法

#### ConfigController

nacos是使用spring mvc提供的rest api。这里面会调用inner.doPollingConfig进行处理

```java
		/**
     * 比较MD5
     */
    @RequestMapping(value = "/listener", method = RequestMethod.POST)
    public void listener(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
        request.setAttribute("org.apache.catalina.ASYNC_SUPPORTED", true);
        String probeModify = request.getParameter("Listening-Configs");
        if (StringUtils.isBlank(probeModify)) {
            throw new IllegalArgumentException("invalid probeModify");
        }

        probeModify = URLDecoder.decode(probeModify, Constants.ENCODE);

        Map<String, String> clientMd5Map;
        try {
            clientMd5Map = MD5Util.getClientMd5Map(probeModify);
        } catch (Throwable e) {
            throw new IllegalArgumentException("invalid probeModify");
        }

        // do long-polling
        inner.doPollingConfig(request, response, clientMd5Map, probeModify.length());
    }
```

#### doPollingConfig

这个方法中，兼容了长轮训和短轮询的逻辑，我们只需要关注长轮训的部分。再次进入到

`longPollingService.addLongPollingClient`

```java
/**
     * 轮询接口
     */
    public String doPollingConfig(HttpServletRequest request, HttpServletResponse response,
                                  Map<String, String> clientMd5Map, int probeRequestSize)
        throws IOException, ServletException {

        // 长轮询
        if (LongPollingService.isSupportLongPolling(request)) {
            longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize);
            return HttpServletResponse.SC_OK + "";
        }

        // else 兼容短轮询逻辑
        List<String> changedGroups = MD5Util.compareMd5(request, response, clientMd5Map);

        // 兼容短轮询result
        String oldResult = MD5Util.compareMd5OldResult(changedGroups);
        String newResult = MD5Util.compareMd5ResultString(changedGroups);

        String version = request.getHeader(Constants.CLIENT_VERSION_HEADER);
        if (version == null) {
            version = "2.0.0";
        }
        int versionNum = Protocol.getVersionNumber(version);

        /**
         * 2.0.4版本以前, 返回值放入header中
         */
        if (versionNum < START_LONGPOLLING_VERSION_NUM) {
            response.addHeader(Constants.PROBE_MODIFY_RESPONSE, oldResult);
            response.addHeader(Constants.PROBE_MODIFY_RESPONSE_NEW, newResult);
        } else {
            request.setAttribute("content", newResult);
        }

        // 禁用缓存
        response.setHeader("Pragma", "no-cache");
        response.setDateHeader("Expires", 0);
        response.setHeader("Cache-Control", "no-cache,no-store");
        response.setStatus(HttpServletResponse.SC_OK);
        return HttpServletResponse.SC_OK + "";
    }
```

#### longPollingService.addLongPollingClient

从方法名字上可以推测出，这个方法应该是把客户端的长轮训请求添加到某个任务中去。

- 获得客户端传递过来的超时时间，并且进行本地计算，提前500ms返回响应，这就能解释为什么客户端响应超时时间是29.5+了。当然如果 isFixedPolling=true 的情况下，不会提前返回响应
- 根据客户端请求过来的md5和服务器端对应的group下对应内容的md5进行比较，如果不一致，则通过 generateResponse 将结果返回
- 如果配置文件没有发生变化，则通过 scheduler.execute 启动了一个定时任务，将客户端的长轮询请求封装成一个叫 ClientLongPolling 的任务，交给 scheduler 去执行

```java
public void addLongPollingClient(HttpServletRequest req, HttpServletResponse rsp, Map<String, String> clientMd5Map,
                                     int probeRequestSize) {

        String str = req.getHeader(LongPollingService.LONG_POLLING_HEADER);
        String noHangUpFlag = req.getHeader(LongPollingService.LONG_POLLING_NO_HANG_UP_HEADER);
        String appName = req.getHeader(RequestUtil.CLIENT_APPNAME_HEADER);
        String tag = req.getHeader("Vipserver-Tag");
        int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);
        /**
         * 提前500ms返回响应，为避免客户端超时 @qiaoyi.dingqy 2013.10.22改动  add delay time for LoadBalance
         */
        long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
        if (isFixedPolling()) {
            timeout = Math.max(10000, getFixedPollingInterval());
            // do nothing but set fix polling timeout
        } else {
            long start = System.currentTimeMillis();
            List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);
            if (changedGroups.size() > 0) {
                generateResponse(req, rsp, changedGroups);
                LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}|{}",
                    System.currentTimeMillis() - start, "instant", RequestUtil.getRemoteIp(req), "polling",
                    clientMd5Map.size(), probeRequestSize, changedGroups.size());
                return;
            } else if (noHangUpFlag != null && noHangUpFlag.equalsIgnoreCase(TRUE_STR)) {
                LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "nohangup",
                    RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
                    changedGroups.size());
                return;
            }
        }
        String ip = RequestUtil.getRemoteIp(req);
        // 一定要由HTTP线程调用，否则离开后容器会立即发送响应
        final AsyncContext asyncContext = req.startAsync();
        // AsyncContext.setTimeout()的超时时间不准，所以只能自己控制
        asyncContext.setTimeout(0L);

        scheduler.execute(
            new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));
    }
```

#### ClientLongPolling

接下来我们来分析一下，clientLongPolling到底做了什么操作。或者说我们可以先猜测一下应该会做什么事情

- 这个任务要阻塞29.5s才能执行，因为立马执行没有任何意义，毕竟前面已经执行过一次了
- 如果在29.5s+之内，数据发生变化，需要提前通知。需要有一种监控机制

基于这些猜想，我们可以看看它的实现过程

从代码粗粒度来看，它的实现似乎和我们的猜想一致，在run方法中，通过scheduler.schedule实现了一个定时任务，它的delay时间正好是前面计算的29.5s。在这个任务中，会通过MD5Util.compareMd5来进行计算

那另外一个，当数据发生变化以后，肯定不能等到29.5s之后才通知呀，那怎么办呢？我们发现有一个allSubs 的东西，它似乎和发布订阅有关系。那是不是有可能当前的clientLongPolling订阅了数据变化的事件呢？

```java
@Override
public void run() {
  asyncTimeoutFuture = scheduler.schedule(new Runnable() {
    @Override
    public void run() {
      try {
        getRetainIps().put(ClientLongPolling.this.ip, System.currentTimeMillis());
        /**
                         * 删除订阅关系
                         */
        allSubs.remove(ClientLongPolling.this);

        if (isFixedPolling()) {
          LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}",
                                 (System.currentTimeMillis() - createTime),
                                 "fix", RequestUtil.getRemoteIp((HttpServletRequest)asyncContext.getRequest()),
                                 "polling",
                                 clientMd5Map.size(), probeRequestSize);
          List<String> changedGroups = MD5Util.compareMd5(
            (HttpServletRequest)asyncContext.getRequest(),
            (HttpServletResponse)asyncContext.getResponse(), clientMd5Map);
          if (changedGroups.size() > 0) {
            sendResponse(changedGroups);
          } else {
            sendResponse(null);
          }
        } else {
          LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}",
                                 (System.currentTimeMillis() - createTime),
                                 "timeout", RequestUtil.getRemoteIp((HttpServletRequest)asyncContext.getRequest()),
                                 "polling",
                                 clientMd5Map.size(), probeRequestSize);
          sendResponse(null);
        }
      } catch (Throwable t) {
        LogUtil.defaultLog.error("long polling error:" + t.getMessage(), t.getCause());
      }

    }

  }, timeoutTime, TimeUnit.MILLISECONDS);

  allSubs.add(this);
}
```

#### allSubs

allSubs是一个队列，队列里面放了ClientLongPolling这个对象。这个队列似乎和配置变更有某种关联关系

```java
/**
* 长轮询订阅关系
*/
final Queue<ClientLongPolling> allSubs;
allSubs.add(this);
```

那这个时候，我的第一想法是，先去看一下当前这个类的类图，发现LongPollingService继承了AbstractEventListener，事件监听？果然没猜错。

![image-20190812211244670](http://ww4.sinaimg.cn/large/006tNc79gy1g5x7d7r8u0j312k0deaan.jpg)

#### AbstractEventListener

这里面有一个抽象的onEvent方法，明显是用来处理事件的方法，而抽象方法必须由子类实现，所以意味着LongPollingService里面肯定实现了onEvent方法

```java
static public abstract class AbstractEventListener {

        public AbstractEventListener() {
            /**
             * automatic register
             */
            EventDispatcher.addEventListener(this);
        }

        /**
         * 感兴趣的事件列表
         *
         * @return event list
         */
        abstract public List<Class<? extends Event>> interest();

        /**
         * 处理事件
         *
         * @param event event
         */
        abstract public void onEvent(Event event);
    }
```

#### LongPollingService.onEvent

这个事件的实现方法中

- 判断事件类型是否为LocalDataChangeEvent
- 通过scheduler.execute执行DataChangeTask这个任务

```java
@Override
    public void onEvent(Event event) {
        if (isFixedPolling()) {
            // ignore
        } else {
            if (event instanceof LocalDataChangeEvent) {
                LocalDataChangeEvent evt = (LocalDataChangeEvent)event;
                scheduler.execute(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));
            }
        }
    }
```

#### DataChangeTask.run

从名字可以看出来，这个是数据变化的任务，最让人兴奋的应该是，它里面有一个循环迭代器，从allSubs里面获得ClientLongPolling

最后通过clientSub.sendResponse把数据返回到客户端。所以，这也就能够理解为何数据变化能够实时触发更新了。

```java
@Override
public void run() {
  try {
    ConfigService.getContentBetaMd5(groupKey);
    for (Iterator<ClientLongPolling> iter = allSubs.iterator(); iter.hasNext(); ) {
      ClientLongPolling clientSub = iter.next();
      if (clientSub.clientMd5Map.containsKey(groupKey)) {
        // 如果beta发布且不在beta列表直接跳过
        if (isBeta && !betaIps.contains(clientSub.ip)) {
          continue;
        }

        // 如果tag发布且不在tag列表直接跳过
        if (StringUtils.isNotBlank(tag) && !tag.equals(clientSub.tag)) {
          continue;
        }

        getRetainIps().put(clientSub.ip, System.currentTimeMillis());
        iter.remove(); // 删除订阅关系
        LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}|{}",
                               (System.currentTimeMillis() - changeTime),
                               "in-advance",
                               RequestUtil.getRemoteIp((HttpServletRequest)clientSub.asyncContext.getRequest()),
                               "polling",
                               clientSub.clientMd5Map.size(), clientSub.probeRequestSize, groupKey);
        clientSub.sendResponse(Arrays.asList(groupKey));
      }
    }
  } catch (Throwable t) {
    LogUtil.defaultLog.error("data change error:" + t.getMessage(), t.getCause());
  }
}
```

那么接下来还有一个疑问是，数据变化之后是如何触发事件的呢？ 所以我们定位到数据变化的请求类中，在ConfigController这个类中，找到POST请求的方法

找到配置变更的位置， 发现数据持久化之后，会通过EventDispatcher进行事件发EventDispatcher.fireEvent 但是这个事件似乎不是我们所关心的事件，原因是这里发布的事件是ConfigDataChangeEvent , LongPollingService感兴趣的事件是 LocalDataChangeEvent

```java
/**
     * 增加或更新非聚合数据。
     *
     * @throws NacosException
     */
    @RequestMapping(method = RequestMethod.POST)
    @ResponseBody
    public Boolean publishConfig(//...省略参数代码)
        throws NacosException {
        //省略部分代码
        ConfigInfo configInfo = new ConfigInfo(dataId, group, tenant, appName, content);
        if (StringUtils.isBlank(betaIps)) {
            if (StringUtils.isBlank(tag)) {
                persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, false);
                EventDispatcher.fireEvent(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
            } else {
                persistService.insertOrUpdateTag(configInfo, tag, srcIp, srcUser, time, false);
                EventDispatcher.fireEvent(new ConfigDataChangeEvent(false, dataId, group, tenant, tag, time.getTime()));
            }
        } else { // beta publish
            persistService.insertOrUpdateBeta(configInfo, betaIps, srcIp, srcUser, time, false);
            EventDispatcher.fireEvent(new ConfigDataChangeEvent(true, dataId, group, tenant, time.getTime()));
        }
        ConfigTraceService.logPersistenceEvent(dataId, group, tenant, requestIpApp, time.getTime(),
            LOCAL_IP, ConfigTraceService.PERSISTENCE_EVENT_PUB, content);

        return true;
    }
```

后来我发现，在Nacos中有一个DumpService，它会定时把变更后的数据dump到磁盘上，DumpService在spring启动之后，会调用init方法启动几个dump任务。然后在任务执行结束之后，会触发一个LocalDataChangeEvent 的事件

```java
@PostConstruct
public void init() {
    LogUtil.defaultLog.warn("DumpService start");
    DumpProcessor processor = new DumpProcessor(this);
    DumpAllProcessor dumpAllProcessor = new DumpAllProcessor(this);
    DumpAllBetaProcessor dumpAllBetaProcessor = new
    DumpAllBetaProcessor(this);
    DumpAllTagProcessor dumpAllTagProcessor = new DumpAllTagProcessor(this);
```

### 简单总结

简单总结一下刚刚分析的整个过程。

- 客户端发起长轮训请求
- 服务端收到请求以后，先比较服务端缓存中的数据是否相同，如果不同，则直接返回
- 如果相同，则通过schedule延迟29.5s之后再执行比较
- 为了保证当服务端在29.5s之内发生数据变化能够及时通知给客户端，服务端采用事件订阅的方式
  来监听服务端本地数据变化的事件，一旦收到事件，则触发DataChangeTask的通知，并且遍历
  allStubs队列中的ClientLongPolling,把结果写回到客户端，就完成了一次数据的推送
- 如果 DataChangeTask 任务完成了数据的 “推送” 之后，ClientLongPolling 中的调度任务又开始执
  行了怎么办呢？
  很简单，只要在进行 “推送” 操作之前，先将原来等待执行的调度任务取消掉就可以了，这样就防
  止了推送操作写完响应数据之后，调度任务又去写响应数据，这时肯定会报错的。所以，在
  ClientLongPolling方法中，最开始的一个步骤就是删除订阅事件

所以总的来说，Nacos采用推+拉的形式，来解决最开始关于长轮训时间间隔的问题。当然，30s这个时
间是可以设置的，而之所以定30s，应该是一个经验值。

## 集群选举问题

Nacos支持集群模式，很显然。

而一旦涉及到集群，就涉及到主从，那么nacos是一种什么样的机制来实现的集群呢？

nacos的集群类似于zookeeper， 它分为leader角色和follower角色， 那么从这个角色的名字可以看出来，这个集群存在选举的机制。 因为如果自己不具备选举功能，角色的命名可能就是master/slave了，当然这只是我基于这么多组件的命名的一个猜测。

### 选举算法

Nacos集群采用`raft`算法来实现，它是相对zookeeper的选举算法较为简单的一种。

选举算法的核心在 RaftCore 中，包括数据的处理和数据同步

[raft算法演示地址]([http://thesecretlivesofdata.com/raft/](http://thesecretlivesofdata.com/raft/))

在Raft中，节点有三种角色：

- Leader：负责接收客户端的请求

- Candidate：用于选举Leader的一种角色

- Follower：负责响应来自Leader或者Candidate的请求


选举分为两个节点

- 服务启动的时候

- leader挂了的时候

所有节点启动的时候，都是follower状态。 如果在一段时间内如果没有收到leader的心跳（可能是没有
leader，也可能是leader挂了），那么follower会变成Candidate。然后发起选举，选举之前，会增加term，这个term和zookeeper中的epoch的道理是一样的。

- follower会投自己一票，并且给其他节点发送票据vote，等到其他节点回复

- 在这个过程中，可能出现几种情况
  - 收到过半的票数通过，则成为leader
  - 被告知其他节点已经成为leader，则自己切换为follower
  - 一段时间内没有收到过半的投票，则重新发起选举
- **约束条件在任一term中，单个节点最多只能投一票**

选举的几种情况

- 第一种情况，赢得选举之后，leader会给所有节点发送消息，避免其他节点触发新的选举

- 第二种情况，比如有三个节点A B C。A B同时发起选举，而A的选举消息先到达C，C给A投了一票，当B的消息到达C时，已经不能满足上面提到的第一个约束，即C不会给B投票，而A和B显然都不会给对方投票。A胜出之后，会给B,C发心跳消息，节点B发现节点A的term不低于自己的term，知道有已经有Leader了，于是转换成follower
- 第三种情况， 没有任何节点获得majority投票，可能是平票的情况。假如总共有四个节点（A/B/C/D），Node C、Node D同时成为了candidate，但Node A投了NodeD一票，NodeB投了Node C一票，这就出现了平票 split vote的情况。这个时候大家都在等啊等，直到超时后重新发起选举。如果出现平票的情况，那么就延长了系统不可用的时间,因此raft引入了randomizedelection timeouts来尽量避免平票情况

## 数据的处理

对于事务操作，请求会转发给leader

非事务操作上，可以任意一个节点来处理

下面这段代码摘自 RaftCore ， 在发布内容的时候，做了两个事情

- 如果当前的节点不是leader，则转发给leader节点处理

- 如果是，则向所有节点发送onPublish

```java
public void signalPublish(String key, Record value) throws Exception {

        if (!isLeader()) {
            JSONObject params = new JSONObject();
            params.put("key", key);
            params.put("value", value);
            Map<String, String> parameters = new HashMap<>(1);
            parameters.put("key", key);

            raftProxy.proxyPostLarge(getLeader().ip, API_PUB, params.toJSONString(), parameters);
            return;
        }

        try {
            OPERATE_LOCK.lock();
            long start = System.currentTimeMillis();
            final Datum datum = new Datum();
            datum.key = key;
            datum.value = value;
            if (getDatum(key) == null) {
                datum.timestamp.set(1L);
            } else {
                datum.timestamp.set(getDatum(key).timestamp.incrementAndGet());
            }

            JSONObject json = new JSONObject();
            json.put("datum", datum);
            json.put("source", peers.local());

            onPublish(datum, peers.local());

            final String content = JSON.toJSONString(json);

            final CountDownLatch latch = new CountDownLatch(peers.majorityCount());
            for (final String server : peers.allServersIncludeMyself()) {
                if (isLeader(server)) {
                    latch.countDown();
                    continue;
                }
                final String url = buildURL(server, API_ON_PUB);
                HttpClient.asyncHttpPostLarge(url, Arrays.asList("key=" + key), content, new AsyncCompletionHandler<Integer>() {
                    @Override
                    public Integer onCompleted(Response response) throws Exception {
                        if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                            Loggers.RAFT.warn("[RAFT] failed to publish data to peer, datumId={}, peer={}, http code={}",
                                datum.key, server, response.getStatusCode());
                            return 1;
                        }
                        latch.countDown();
                        return 0;
                    }

                    @Override
                    public STATE onContentWriteCompleted() {
                        return STATE.CONTINUE;
                    }
                });

            }

            if (!latch.await(UtilsAndCommons.RAFT_PUBLISH_TIMEOUT, TimeUnit.MILLISECONDS)) {
                // only majority servers return success can we consider this update success
                Loggers.RAFT.error("data publish failed, caused failed to notify majority, key={}", key);
                throw new IllegalStateException("data publish failed, caused failed to notify majority, key=" + key);
            }

            long end = System.currentTimeMillis();
            Loggers.RAFT.info("signalPublish cost {} ms, key: {}", (end - start), key);
        } finally {
            OPERATE_LOCK.unlock();
        }
    }
```

