# redis架构演变与redis-cluster群集读写方案

## 导言

redis-cluster是近年来redis架构不断改进中的相对较好的redis高可用方案。本文涉及到近年来redis多实例架构的演变过程，包括普通主从架构（Master、slave可进行写读分离）、哨兵模式下的主从架构、redis-cluster高可用架构（redis官方默认cluster下不进行读写分离）的简介。同时还介绍使用Java的两大redis客户端：Jedis与Lettuce用于读写redis-cluster的数据的一般方法。再通过官方文档以及互联网的相关技术文档，给出redis-cluster架构下的读写能力的优化方案，包括官方的推荐的扩展redis-cluster下的Master数量以及非官方默认的redis-cluster的读写分离方案，案例中使用Lettuce的特定方法进行redis-cluster架构下的数据读写分离。

## 近年来redis多实例用架构的演变过程

redis是基于内存的高性能key-value数据库，若要让redis的数据更稳定安全，需要引入多实例以及相关的高可用架构。而近年来redis的高可用架构亦不断改进，先后出现了本地持久化、主从备份、哨兵模式、redis-cluster群集高可用架构等等方案。

### 1、redis普通主从模式

通过持久化功能，Redis保证了即使在服务器重启的情况下也不会损失（或少量损失）数据，因为持久化会把内存中数据保存到硬盘上，重启会从硬盘上加载数据。   。但是由于数据是存储在一台服务器上的，如果这台服务器出现硬盘故障等问题，也会导致数据丢失。为了避免单点故障，通常的做法是将数据库复制多个副本以部署在不同的服务器上，这样即使有一台服务器出现故障，其他服务器依然可以继续提供服务。为此，  Redis 提供了复制（replication）功能，可以实现当一台数据库中的数据更新后，自动将更新的数据同步到其他数据库上。

在复制的概念中，数据库分为两类，一类是主数据库（master），另一类是从数据库（slave）。主数据库可以进行读写操作，当写操作导致数据变化时会自动将数据同步给从数据库。而从数据库一般是只读的，并接受主数据库同步过来的数据。一个主数据库可以拥有多个从数据库，而一个从数据库只能拥有一个主数据库。
 ![img](https://oscimg.oschina.net/oscnet/ad6dc257ef59041bb301c91fc2f82517a33.jpg)

主从模式的配置，一般只需要再作为slave的redis节点的conf文件上加入“slaveof master*ip master*port”， 或者作为slave的redis节点启动时使用如下参考命令： 

```
redis-server --port 6380 --slaveof masterIp masterPort   
```

redis的普通主从模式，能较好地避免单独故障问题，以及提出了读写分离，降低了Master节点的压力。互联网上大多数的对redis读写分离的教程，都是基于这一模式或架构下进行的。但实际上这一架构并非是目前最好的redis高可用架构。

### 2、redis哨兵模式高可用架构

当主数据库遇到异常中断服务后，开发者可以通过手动的方式选择一个从数据库来升格为主数据库，以使得系统能够继续提供服务。然而整个过程相对麻烦且需要人工介入，难以实现自动化。  为此，Redis 2.8开始提供了哨兵工具来实现自动化的系统监控和故障恢复功能。  哨兵的作用就是监控redis主、从数据库是否正常运行，主出现故障自动将从数据库转换为主数据库。

顾名思义，哨兵的作用就是监控Redis系统的运行状况。它的功能包括以下两个。 

（1）监控主数据库和从数据库是否正常运行。
 （2）主数据库出现故障时自动将从数据库转换为主数据库。 

![img](https://oscimg.oschina.net/oscnet/30d8e5a0156044f175e28f77ba5ab235282.jpg)

可以用info replication查看主从情况 例子： 1主2从 1哨兵,可以用命令起也可以用配置文件里 可以使用双哨兵，更安全，参考命令如下： 

```
redis-server --port 6379 
redis-server --port 6380 --slaveof 192.168.0.167 6379 
redis-server --port 6381 --slaveof 192.168.0.167 6379
redis-sentinel sentinel.conf 
```

其中，哨兵配置文件sentinel.conf参考如下： 

```
sentinel monitor mymaster 192.168.0.167 6379 1  
```

其中mymaster表示要监控的主数据库的名字。配置哨兵监控一个系统时，只需要配置其监控主数据库即可，哨兵会自动发现所有复制该主数据库的从数据库。
 Master与slave的切换过程：
 （1）slave leader升级为master
 （2）其他slave修改为新master的slave
 （3）客户端修改连接
 （4）老的master如果重启成功，变为新master的slave 

### 3、redis-cluster群集高可用架构

即使使用哨兵，redis每个实例也是全量存储，每个redis存储的内容都是完整的数据，浪费内存且有木桶效应。为了最大化利用内存，可以采用cluster群集，就是分布式存储。即每台redis存储不同的内容。
   采用redis-cluster架构正是满足这种分布式存储要求的集群的一种体现。redis-cluster架构中，被设计成共有16384个hash  slot。每个master分得一部分slot，其算法为：hash_slot = crc16(key) mod 16384  ，这就找到对应slot。采用hash  slot的算法，实际上是解决了redis-cluster架构下，有多个master节点的时候，数据如何分布到这些节点上去。key是可用key，如果有{}则取{}内的作为可用key，否则整个可以是可用key。群集至少需要3主3从，且每个实例使用不同的配置文件。  

![img](https://oscimg.oschina.net/oscnet/091fbfd2d70bf4e1a92f8a0b5109597feee.jpg)

在redis-cluster架构中，redis-master节点一般用于接收读写，而redis-slave节点则一般只用于备份，其与对应的master拥有相同的slot集合，若某个redis-master意外失效，则再将其对应的slave进行升级为临时redis-master。
 **在redis的官方文档中，对redis-cluster架构上，有这样的说明：在cluster架构下，默认的，一般redis-master用于接收读写，而redis-slave则用于备份，当有请求是在向slave发起时，会直接重定向到对应key所在的master来处理。但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离**。具体可以参阅redis官方文档（https://redis.io/commands/readonly）等相关内容： 

```
Enables read queries for a connection to a Redis Cluster slave node.    
Normally slave nodes will redirect clients to the authoritative master for the hash slot involved in a given command, however clients can use slaves in order to scale reads using the READONLY command.    
READONLY tells a Redis Cluster slave node that the client is willing to read possibly stale data and is not interested in running write queries.    
When the connection is in readonly mode, the cluster will send a redirection to the client only if the operation involves keys not served by the slave's master node. This may happen because:  
The client sent a command about hash slots never served by the master of this slave.
The cluster was reconfigured (for example resharded) and the slave is no longer able to serve commands for a given hash slot.
```

例如，我们假设已经建立了一个三主三从的redis-cluster架构，其中A、B、C节点都是redis-master节点，A1、B1、C1节点都是对应的redis-slave节点。在我们只有master节点A，B，C的情况下，对应redis-cluster如果节点B失败，则群集无法继续，因为我们没有办法再在节点B的所具有的约三分之一的hash   slot集合范围内提供相对应的slot。然而，如果我们为每个主服务器节点添加一个从服务器节点，以便最终集群由作为主服务器节点的A，B，C以及作为从服务器节点的A1，B1，C1组成，那么如果节点B发生故障，系统能够继续运行。节点B1复制B，并且B失效时，则redis-cluster将促使B的从节点B1作为新的主服务器节点并且将继续正确地操作。但请注意，如果节点B和B1在同一时间发生故障，则Redis群集无法继续运行。  

Redis群集配置参数:在继续之前，让我们介绍一下Redis Cluster在redis.conf文件中引入的配置参数。有些命令的意思是显而易见的，有些命令在你阅读下面的解释后才会更加清晰。 

（1）cluster-enabled ：如果想在特定的Redis实例中启用Redis群集支持就设置为yes。 否则，实例通常作为独立实例启动。
 （2）cluster-config-file ：请注意，尽管有此选项的名称，但这不是用户可编辑的配置文件，而是Redis群集节点每次发生更改时自动保留群集配置（基本上为状态）的文件。
 （3）cluster-node-timeout ：Redis群集节点可以不可用的最长时间，而不会将其视为失败。 如果主节点超过指定的时间不可达，它将由其从属设备进行故障切换。
  （4）cluster-slave-validity-factor  ：如果设置为0，无论主设备和从设备之间的链路保持断开连接的时间长短，从设备都将尝试故障切换主设备。  如果该值为正值，则计算最大断开时间作为节点超时值乘以此选项提供的系数，如果该节点是从节点，则在主链路断开连接的时间超过指定的超时值时，它不会尝试启动故障切换。
 （5）cluster-migration-barrier ：主设备将保持连接的最小从设备数量，以便另一个从设备迁移到不受任何从设备覆盖的主设备。有关更多信息，请参阅本教程中有关副本迁移的相应部分。
 （6）cluster-require-full-coverage ：如果将其设置为yes，则默认情况下，如果key的空间的某个百分比未被任何节点覆盖，则集群停止接受写入。 如果该选项设置为no，则即使只处理关于keys子集的请求，群集仍将提供查询。 

以下是最小的Redis集群配置文件： 

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

注意：
 （1）redis-cluster最小配置为三主三从，当1个主故障，大家会给对应的从投票，把从立为主，若没有从数据库可以恢复则redis群集就down了。
  （2）在这个redis cluster中，如果你要在slave读取数据，那么需要带上readonly指令。redis  cluster的核心的理念，主要是用slave做高可用的，每个master挂一两个slave，主要是做数据的热备，当master故障时的作为主备切换，实现高可用的。redis  cluster默认是不支持slave节点读或者写的，跟我们手动基于replication搭建的主从架构不一样的。slave  node要设置readonly，然后再get，这个时候才能在slave node进行读取。对于redis  -cluster主从架构，若要进行读写分离，官方其实是不建议的，但也能做，只是会复杂一些。具体见下面的章节。
  （3）redis-cluster的架构下，实际上本身master就是可以任意扩展的，你如果要支撑更大的读吞吐量，或者写吞吐量，或者数据量，都可以直接对master进行横向扩展就可以了。也扩容master，跟之前扩容slave进行读写分离，效果是一样的或者说更好。
 （4）可以使用自带客户端连接：使用redis-cli -c -p cluster中任意一个端口，进行数据获取测试。 

## Java中对redis-cluster数据的一般读取方法简介

### 使用Jedis读写redis-cluster的数据

由于Jedis类一般只能对一台redis-master进行数据操作，所以面对redis-cluster多台master与slave的群集，Jedis类就不能满足了。这个时候我们需要引用另外一个操作类：JedisCluster类。
 例如我们有6台机器组成的redis-cluster：
 172.20.52.85:7000、 172.20.52.85:7001、172.20.52.85:7002、172.20.52.85:7003、172.20.52.85:7004、172.20.52.85:7005
 其中master机器对应端口：7000、7004、7005
 slave对应端口：7001、7002、7003 

使用JedisCluster对redis-cluster进行数据操作的参考代码如下： 

```
// 添加nodes服务节点到Set集合
Set<HostAndPort> hostAndPortsSet = new HashSet<HostAndPort>();
// 添加节点
hostAndPortsSet.add(new HostAndPort("172.20.52.85", 7000));
hostAndPortsSet.add(new HostAndPort("172.20.52.85", 7001));
hostAndPortsSet.add(new HostAndPort("172.20.52.85", 7002));
hostAndPortsSet.add(new HostAndPort("172.20.52.85", 7003));
hostAndPortsSet.add(new HostAndPort("172.20.52.85", 7004));
hostAndPortsSet.add(new HostAndPort("172.20.52.85", 7005));

// Jedis连接池配置
JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
jedisPoolConfig.setMaxIdle(100);
jedisPoolConfig.setMaxTotal(500);
jedisPoolConfig.setMinIdle(0);
jedisPoolConfig.setMaxWaitMillis(2000); // 设置2秒
jedisPoolConfig.setTestOnBorrow(true);

JedisCluster jedisCluster = new JedisCluster(hostAndPortsSet ,jedisPoolConfig);
String result = jedisCluster.get("event:10");
System.out.println(result);    
```

运行结果截图如下图所示：
 ![img](https://oscimg.oschina.net/oscnet/3d748b439f622af9cd3c02f13bae423faee.jpg)

第一节中我们已经介绍了redis-cluster架构下master提供读写功能，而slave一般只作为对应master机器的数据备份不提供读写。如果我们只在hostAndPortsSet中只配置slave，而不配置master，实际上还是可以读到数据，但其内部操作实际是通过slave重定向到相关的master主机上，然后再将结果获取和输出。  

上面是普通项目使用JedisCluster的简单过程，若在spring  boot项目中，可以定义JedisConfig类，使用@Configuration、@Value、@Bean等一些列注解完成JedisCluster的配置，然后再注入该JedisCluster到相关service逻辑中引用，这里介绍略。  

### 使用Lettuce读写redis-cluster数据

Lettuce 和 Jedis 的定位都是Redis的client。Jedis在实现上是直接连接的redis  server，如果在多线程环境下是非线程安全的，这个时候只有使用连接池，为每个Jedis实例增加物理连接，每个线程都去拿自己的 Jedis  实例，当连接数量增多时，物理连接成本就较高了。
  Lettuce的连接是基于Netty的，连接实例（StatefulRedisConnection）可以在多个线程间并发访问，应为StatefulRedisConnection是线程安全的，所以一个连接实例（StatefulRedisConnection）就可以满足多线程环境下的并发访问，当然这个也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。
 其中spring boot 2.X版本中，依赖的spring-session-data-redis已经默认替换成Lettuce了。
 同样，例如我们有6台机器组成的redis-cluster：
 172.20.52.85:7000、 172.20.52.85:7001、172.20.52.85:7002、172.20.52.85:7003、172.20.52.85:7004、172.20.52.85:7005
 其中master机器对应端口：7000、7004、7005
 slave对应端口：7001、7002、7003
 在spring boot 2.X版本中使用Lettuce操作redis-cluster数据的方法参考如下：
 （1）pom文件参考如下：
 parent中指出spring boot的版本，要求2.X以上： 

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

依赖中需要加入spring-boot-starter-data-redis，参考如下：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

（2）springboot的配置文件要包含如下内容： 

```
spring.redis.database=0
spring.redis.lettuce.pool.max-idle=10
spring.redis.lettuce.pool.max-wait=500
spring.redis.cluster.timeout=1000
spring.redis.cluster.max-redirects=3

spring.redis.cluster.nodes=172.20.52.85:7000,172.20.52.85:7001,172.20.52.85:7002,172.20.52.85:7003,172.20.52.85:7004,172.20.52.85:7005   
```

（3）新建RedisConfiguration类，参考代码如下： 

```
@Configuration
public class RedisConfiguration {
    [@Resource](https://my.oschina.net/u/929718)
    private LettuceConnectionFactory myLettuceConnectionFactory;


    @Bean
    public RedisTemplate<String, Serializable> redisTemplate() {

        RedisTemplate<String, Serializable> template = new RedisTemplate<>();

        template.setKeySerializer(new StringRedisSerializer());

        //template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());

        template.setConnectionFactory(myLettuceConnectionFactory);

        return template;

    }
}  
```

（4）新建RedisFactoryConfig类，参考代码如下： 

```
@Configuration
public class RedisFactoryConfig {

    @Autowired
    private Environment environment;


    @Bean
    public RedisConnectionFactory myLettuceConnectionFactory() {
        Map<String, Object> source = new HashMap<String, Object>();

        source.put("spring.redis.cluster.nodes", environment.getProperty("spring.redis.cluster.nodes"));
        source.put("spring.redis.cluster.timeout", environment.getProperty("spring.redis.cluster.timeout"));
        source.put("spring.redis.cluster.max-redirects", environment.getProperty("spring.redis.cluster.max-redirects"));

        RedisClusterConfiguration redisClusterConfiguration;

        redisClusterConfiguration = new RedisClusterConfiguration(new MapPropertySource("RedisClusterConfiguration", source));

        return new LettuceConnectionFactory(redisClusterConfiguration);

    }
}  
```

（5）在业务类service中注入Lettuce相关的RedisTemplate，进行相关操作。以下是我化简到了springbootstarter中进行，参考代码如下： 

```
@SpringBootApplication
public class NewRedisClientApplication {

    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(NewRedisClientApplication.class, args);
        RedisTemplate redisTemplate  = (RedisTemplate)context.getBean("redisTemplate");
        String rtnValue = (String)redisTemplate.opsForValue().get("event:10");
        System.out.println(rtnValue);

    }
}  
```

运行结果的截图如下： 

![img](https://oscimg.oschina.net/oscnet/227a574f3313d00fe1c5aa52ef2fcd2e837.jpg)

以上的介绍，是采用Jedis以及Lettuce对redis-cluster数据的简单读取。Jedis也好，Lettuce也好，其对于redis-cluster架构下的数据的读取，都是默认是按照redis官方对redis-cluster的设计，自动进行重定向到master节点中进行的，哪怕是我们在配置中列出了所有的master节点和slave节点。查阅了Jedis以及Lettuce的github上的源码，默认不支持redis-cluster下的读写分离，可以看出Jedis若要支持redis-cluster架构下的读写分离，需要自己改写和构建多一些包装类，定义好Master和slave节点的逻辑；而Lettuce的源码中，实际上预留了方法（setReadForm(ReadFrom.SLAVE)）进行redis-cluster架构下的读写分离，相对来说修改会简单一些，具体可以参考后面的章节。  

## redis-cluster架构下的读写能力的优化方案

在上面的一些章节中，已经有讲到redis近年来的高可用架构的演变，以及在redis-cluster架构下，官方对redis-master、redis-slave的其实有使用上的建议，即redis-master节点一般用于接收读写，而redis-slave节点则一般只用于备份，其与对应的master拥有相同的slot集合，若某个redis-master意外失效，则再将其对应的slave进行升级为临时redis-master。但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离。
 具体可以参阅redis官方文档（https://redis.io/commands/readonly），以下是reids在线文档中，对slave的readonly说明内容： 

![img](https://oscimg.oschina.net/oscnet/4a106be4b042d98809d3fdacdf42072a333.jpg)

实际上本身master就是可以任意扩展的，所以如果要支撑更大的读吞吐量，或者写吞吐量，或者数据量，都可以直接对master进行横向水平扩展就可以了。也就是说，扩容master，跟之前扩容slave并进行读写分离，效果是一样的或者说更好。
 所以下面我们将按照redis-cluster架构下分别进行水平扩展Master，以及在redis-cluster架构下对master、slave进行读写分离两套方案进行讲解。 

### 一）水平扩展Master实例来进行redis-cluster性能的提升

redis官方在线文档以及一些互联网的参考资料都表明，在redis-cluster架构下，实际上不建议做物理的读写分离。那么如果我们真的不做读写分离的话，能否通过简单的方法进行redis-cluster下的性能的提升？我们可以通过master的水平扩展，来横向扩展读写吞吐量，并且能支撑更多的海量数据。
   对master进行水平扩展有两种方法，一种是单机上面进行master实例的增加（建议每新增一个master，也新增一个对应的slave），另一种是新增机器部署新的master实例（同样建议每新增一个master，也新增一个对应的slave）。当然，我们也可以进行这两种方法的有效结合。

（1）单机上通过多线程建立新redis-master实例，即逻辑上的水平扩展：
 一般的，对于redis单机，单线程的读吞吐是4w/s~5W/s，写吞吐为2w/s。
   单机合理开启redis多线程情况下（一般线程数为CPU核数的倍数），总吞吐量会有所上升，但每个线程的平均处理能力会有所下降。例如一个2核CPU，开启2线程的时候，总读吞吐能上升是6W/s~7W/s，即每个线程平均约3W/s再多一些。但过多的redis线程反而会限制了总吞吐量。  

（2）扩展更多的机器，部署新redis-master实例，即物理上的水平扩展：
  例如，我们可以再原来只有3台master的基础上，连入新机器继续新实例的部署，最终水平扩展为6台master（建议每新增一个master，也新增一个对应的slave）。例如之前每台master的处理能力假设是读吞吐5W/s,写吞吐2W/s,扩展前一共的处理能力是：15W/s读，6W/s写。如果我们水平扩展到6台master，读吞吐可以达到总量30W/s，写可以达到12w/s，性能能够成倍增加。  

（3）若原本每台部署redis-master实例的机器都性能良好，则可以通过上述两者的结合，进行一个更优的组合。 

使用该方案进行redis-cluster性能的提升的优点有：
 （1）符合redis官方要求和数据的准确性。
 （2）真正达到更大吞吐量的性能扩展。
 （3）无需代码的大量更改，只需在配置文件中重新配置新的节点信息。 

当然缺点也是有的：
 （1）需要新增机器，提升性能，即成本会增加。
 （2）若不新增机器，则需要原来的实例所运行的机器性能较好，能进行以多线程的方式部署新实例。但随着线程的增多，而机器的能力不足以支撑的时候，实际上总体能力会提升不太明显。
 （3）redis-cluster进行新的水平扩容后，需要对master进行新的hash slot重新分配，这相当于需要重新加载所有的key，并按算法平均分配到各个Master的slot当中。 

### （二）引入Lettuce以及修改相关方法，达到对redis-cluster的读写分离

通过上面的一些章节，我们已经可以了解到Lettuce客户端读取redis的一些操作，使用Lettuce能体现出了简单，安全，高效。实际上，查阅了Lettuce对redis的读写，许多地方都进行了redis的读写分离。但这些都是基于上述redis架构中最普通的主从分离架构下的读写分离，而对于redis-cluster架构下，Lettuce可能是遵循了redis官方的意见，在该架构下，Lettuce在源码中直接设置了只由master上进行读写（具体参见gitHub的Lettuce项目）：  

![img](https://oscimg.oschina.net/oscnet/3f540c1e69927367a385e8be6dc9c03dea9.jpg)

那么如果真的需要让Lettuce改为能够读取redis-cluster的slave，进行读写分离，是否可行？实际上还是可以的。这就需要我们自己在项目中进行二次加工，即不使用spring-boot中的默认Lettuce初始化方法，而是自己去写一个属于自己的Lettuce的新RedisClusterClient的连接，并且对该RedisClusterClient的连接进行一个比较重要的设置，那就是由connection.setReadFrom(ReadFrom.MASTER)改为connection.setReadFrom(ReadFrom.SLAVE)。  

下面我们开始对之前章节中的Lettuce读取redis-cluster数据的例子，进行改写，让Lettuce能够支持该架构下的读写分离： 

spring boot 2.X版本中，依赖的spring-session-data-redis已经默认替换成Lettuce了。
 同样，例如我们有6台机器组成的redis-cluster：
 172.20.52.85:7000、 172.20.52.85:7001、172.20.52.85:7002、172.20.52.85:7003、172.20.52.85:7004、172.20.52.85:7005
 其中master机器对应端口：7000、7004、7005
 slave对应端口：7001、7002、7003
 在spring boot 2.X版本中使用Lettuce操作redis-cluster数据的方法参考如下：
 （1）pom文件参考如下：
 parent中指出spring boot的版本，要求2.X以上： 

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

依赖中需要加入spring-boot-starter-data-redis，参考如下：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

（2）springboot的配置文件要包含如下内容： 

```
spring.redis.database=0
spring.redis.lettuce.pool.max-idle=10
spring.redis.lettuce.pool.max-wait=500
spring.redis.cluster.timeout=1000
spring.redis.cluster.max-redirects=3

spring.redis.cluster.nodes=172.20.52.85:7000,172.20.52.85:7001,172.20.52.85:7002,172.20.52.85:7003,172.20.52.85:7004,172.20.52.85:7005   
```

（3）我们回到RedisConfiguration类中，删除或屏蔽之前的RedisTemplate方法，新增自定义的redisClusterConnection方法，并且设置好读写分离，参考代码如下： 

```
@Configuration
public class RedisConfiguration {

    @Autowired
    private Environment environment;


    @Bean
    public StatefulRedisClusterConnection redisClusterConnection(){

        String strRedisClusterNodes = environment.getProperty("spring.redis.cluster.nodes");
        String[] listNodesInfos = strRedisClusterNodes.split(",");

        List<RedisURI> listRedisURIs = new ArrayList<RedisURI>();
        for(String tmpNodeInfo : listNodesInfos){
            String[] tmpInfo = tmpNodeInfo.split(":");
            listRedisURIs.add(new RedisURI(tmpInfo[0],Integer.parseInt(tmpInfo[1]),Duration.ofDays(10)));
        }

        RedisClusterClient clusterClient  = RedisClusterClient.create(listRedisURIs);
        StatefulRedisClusterConnection<String, String> connection = clusterClient.connect();
        connection.setReadFrom(ReadFrom.SLAVE);

        return connection;
    }
}
```

其中，这三行代码是能进行redis-cluster架构下读写分离的核心： 

```
RedisClusterClient clusterClient  = RedisClusterClient.create(listRedisURIs);
StatefulRedisClusterConnection<String, String> connection = clusterClient.connect();
connection.setReadFrom(ReadFrom.SLAVE);  
```

在业务类service中注入Lettuce相关的redisClusterConnection，进行相关读写操作。以下是我直接化简到了springbootstarter中进行，参考代码如下： 

```
@SpringBootApplication
public class NewRedisClientApplication {

    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(NewRedisClientApplication.class, args);

        StatefulRedisClusterConnection<String, String> redisClusterConnection = (StatefulRedisClusterConnection)context.getBean("redisClusterConnection");
        System.out.println(redisClusterConnection.sync().get("event:10"));


    }
}
```

运行的结果如下图所示： 

![img](https://oscimg.oschina.net/oscnet/05b30533852d0f7c68de85bd162702cdd6e.jpg)

可以看到，经过改写的redisClusterConnection的确能读取到redis-cluster的数据。但这一个数据我们还需要验证一下到底是不是通过slave读取到的，又或者还是通过slave重定向给master才获取到的？
   带着疑问，我们可以开通debug模式，在redisClusterConnection.sync().get("event:10")等类似的获取数据的代码行上面打上断点。通过代码的走查，我们可以看到，在ReadFromImpl类中，最终会select到key所在的slave节点，进行返回，并在该slave中进行数据的读取：  

ReadFromImpl显示： 

![img](https://oscimg.oschina.net/oscnet/4878d0cf37d3906218ff926810a090be285.jpg)

另外我们通过connectFuture中的显示也验证了对于slave的readonly生效了： 

![img](https://oscimg.oschina.net/oscnet/7dba55d68e47fec819eac1c3d5c29e365ff.jpg)

这样，就达到了通过Lettuce客户端对redis-cluster的读写分离了。 

使用该方案进行redis-cluster性能的提升的优点有：
 （1）直接通过代码级更改，而不需要配置新的redis-cluster环境。
 （2）无需增加机器或升级硬件设备。 

但同时，该方案也有缺点：
 （1）非官方对redis-cluster的推荐方案，因为在redis-cluster架构下，进行读写分离，有可能会读到过期的数据。
  （2）需对项目进行全面的替换，将Jedis客户端变为Lettuce客户端，对代码的改动较大，而且使用Lettuce时，使用的并非spring  boot的自带集成Lettuce的redisTemplate配置方法，而是自己配置读写分离的  redisClusterConnetcion，日后遇到问题的时候，可能官方文档的支持率或支撑能力会比较低。
 （3）需修改redis-cluster的master、slave配置，在各个节点中都需要加入slave-read-only yes。
 （4）性能的提升没有水平扩展master主机和实例来得直接干脆。

## 总结

总体上来说，redis-cluster高可用架构方案是目前最好的redis架构方案，redis的官方对redis-cluster架构是建议redis-master用于接收读写，而redis-slave则用于备份（备用），默认不建议读写分离。但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离。Jedis、Lettuce都可以进行redis-cluster的读写操作，而且默认只针对Master进行读写，若要对redis-cluster架构下进行读写分离，则Jedis需要进行源码的较大改动，而Lettuce开放了setReadFrom()方法，可以进行二次封装成读写分离的客户端，相对简单，而且Lettuce比Jedis更安全。redis-cluster架构下可以直接通过水平扩展master来达到性能的提升。

## 参考文档

1，网文《关于redis主从、哨兵、集群的介绍》：https://blog.csdn.net/c295477887/article/details/52487621
 2，知乎《lettuce与jedis对比介绍》：https://www.zhihu.com/question/53124685
 3，网文《Redis 高可用架构最佳实践问答集锦》：http://www.talkwithtrend.com/Article/178165
 4，网文《Redis进阶实践之十一 Redis的Cluster集群搭建》：https://www.cnblogs.com/PatrickLiu/p/8458788.html
 5，redis官方在线文档：https://redis.io/
 6，网文《redis cluster的介绍及搭建(6)》：https://blog.csdn.net/qq1137623160/article/details/79184686
 7，网文《Springboot2.X集成redis集群(Lettuce)连接》：http://www.cnblogs.com/xymBlog/p/9303032.html
 8，Jedis的gitHub地址：https://github.com/xetorthio/jedis
 9，Lettuce的gitHub地址：https://github.com/lettuce-io/lettuce-core/ 