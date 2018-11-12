# Redis 关于key超时监听

## 需求
在做一个限时投票的活动的时候(服务用的spring-cloud分布式搭建的，业务服务在不同的两台虚拟机上，使用同一个redis服务和数据库)，需求要求：在1分钟在场所有人可以参与投票，过期后不允许投票(200人左右)。

## 分析
由于在不同的虚拟机上，在java中处理1分钟限时不好处理(当时外网隔离，两台服务的所在的虚拟机上的时间存在差异)，使用数据库来控制投票可能导致压力过大，处理过慢，导致投票时间产生过大的误差。故采用redis来控制投票开关，redis内存型noSql，读写速度快、原子操作和超时监听api等优点，投票开关使用set设置一个1分钟的key，每次投票的时候然后java中一直监听这个key值是否存在，获取key剩余的时间作为投票剩余的时间，key过期后有一个回调方法，可以用来做投票数据统计处理。

## redis超时监听
过期事件通过Redis的订阅与发布功能（pub/sub）来进行分发，在测试中使用模式匹配，采用PSUBSCRIBE订阅expired频道。

## redis配置
而对超时的监听呢，并不需要自己发布，只有修改配置文件redis.conf中的：notify-keyspace-events Ex，默认为notify-keyspace-events ""

  K    键空间通知，以__keyspace@<db>__为前缀
  E    键事件通知，以__keysevent@<db>__为前缀
  g    del , expipre , rename 等类型无关的通用命令的通知, ...
  $    String命令
  l    List命令
  s    Set命令
  h    Hash命令     z    有序集合命令
  x    过期事件（每次key过期时生成）
  e    驱逐事件（当key在内存满了被清除时生成）
  A    g$lshzxe的别名，因此”AKE”意味着所有的事件

### 测试(windows)
启动redis,启动命令窗口，切到**redis-cli.exe**对应的目录下，控制台输入redis-cli.exe

	D:\Program Files (x86)\Redis-x64-3.2.100>redis-cli.exe
	127.0.0.1:6379> psubscribe __keyevent@0__:expired
	Reading messages... (press Ctrl-C to quit)
	1) "psubscribe"
	2) "__keyevent@0__:expired"
	3) (integer) 1

在启动另外一个redis命令窗口

	D:\Program Files (x86)\Redis-x64-3.2.100>redis-cli.exe
	127.0.0.1:6379> set aa bb  EX 10
	OK
	127.0.0.1:6379> get aa
	"bb"
	...(10秒后)
	127.0.0.1:6379> get aa
	(nil)

此时第一个控制台显示

	1) "pmessage"
	2) "__keyevent@0__:expired"
	3) "__keyevent@0__:expired"
	4) "aa"
显示以上结果，说明显示配置成功，#keyevent@**0** 中的0 代表0号库

## 在Java中的处理
### spring-application.xml 配置文件
```java
<!-- redis链接配置 -->
<bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory" p:host-name="${redis.host}" p:port="${redis.port}" />
<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate"
	p:connection-factory-ref="jedisConnectionFactory">
</bean>
<!-- 监听实现类，监听key过期事件 -->
<bean id="messageListener" class="org.springframework.data.redis.listener.adapter.MessageListenerAdapter">
    <constructor-arg>
        <bean class="com.xxx.xxx.MyRedisKeyExpiredMessageDelegate" />
    </constructor-arg>
</bean>
<bean id="redisContainer" class="org.springframework.data.redis.listener.RedisMessageListenerContainer">
    <property name="connectionFactory" ref="jedisConnectionFactory" />
    <property name="messageListeners">
        <map>
            <entry key-ref="messageListener">
                <list>
                    <bean class="org.springframework.data.redis.listener.PatternTopic">
                        <constructor-arg value="__keyevent@0__:expired" />
                    </bean>
                </list>
            </entry>
        </map>
    </property>
</bean>
	```

### MyRedisKeyExpiredMessageDelegate

```java
/**
 * 监听redis中key值过期的事件
 *
**/
public class MyRedisKeyExpiredMessageDelegate implements MessageListener {

	@Override
	public void onMessage(Message message, byte[] arg1) {
		try {
			System.out.println("channel:" + new String(message.getChannel())
		        + ",message:" + new String(message.getBody(), "utf-8"));
		} catch (UnsupportedEncodingException e) {
			e.printStackTrace();
		}
	}
}
```
