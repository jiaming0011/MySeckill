# 秒杀项目简介
这是一个针对当下热门的秒杀行为（如抢红包，淘宝秒杀等）而做的一个小项目，项目使用SSM框架搭建的，使用maven对项目进行构建和管理。在这里主要记录下项目是如何优化高并发问题的。
### 对于前端
前端方面主要是CDN的使用，项目前端是用bootstrap框架搭建的，其中的js，css等静态资源都是通过cdn的方式去获取。原因如下：<br>
1. 用户不断刷新，使用CDN可以减少去后端访问的静态资源
2. CDN(内容分发网络)加速用户获取数据的系统
3. CDN部署在用户最近的网络节点上,访问速度快
4. 命中CDN不需要访问后端服务器
### 对于后端
通过对整个项目的了解与分析，我总结出容易产生高并发操作的两个地方：<br>
### 1. 秒杀地址接口的获取，即查询一个商品的详情<br>
#### 优化分析：
频繁访问数据库
#### 优化思路：
设置服务器缓存端存数据
#### 具体方法：
主要用到的是redis优化“地址暴露接口”，即减少向数据库取商品详情的操作，第一次从数据库取到商品详情之后会缓存在redis中，直到超时结束。<br>
首先设计一个RedisDao接口实现set和put方法（注意pom.xml要写入redis的相关依赖）：
```java
public class RedisDao {
	private final Logger logger = LoggerFactory.getLogger(this.getClass());
	private final JedisPool jedisPool;
	
	public RedisDao(String ip,int port){
		jedisPool = new JedisPool(ip,port);
	}
	
	private RuntimeSchema<Seckill> schema = RuntimeSchema.createFrom(Seckill.class);
	
	public Seckill getSeckill(long seckillId){
		//redis操作逻辑
		try {
			Jedis jedis = jedisPool.getResource();
			try {
				String key = "seckill:"+seckillId;
				//并没有实现内部序列化的操作
				//get->byte[]->反序列化->Object(Seckill)
				//采用自定义序列化
				//protostuff:pojo
				byte[] bytes = jedis.get(key.getBytes());
				//缓存重获取到
				if(bytes != null){
					//空对象
					Seckill seckill = schema.newMessage();
					ProtostuffIOUtil.mergeFrom(bytes, seckill, schema);
					//seckill被反序列化
					return seckill;
				}
			}finally{
				jedis.close();
			}
			
		}catch (Exception e) {
			logger.error(e.getMessage(), e);
		}
		
		return null;
	}
	public String putSeckill(Seckill seckill){
		//set Object(Seckill) ->序列化 ->byte[]
		try {
			Jedis jedis = jedisPool.getResource();
			try {
				String key = "seckill:"+seckill.getSeckillId();	
				byte[] bytes = ProtostuffIOUtil.toByteArray(seckill, schema,
						LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE));
				//超时缓存
				int timeout = 60 * 60;//一小时
				String result = jedis.setex(key.getBytes(), timeout, bytes);
				return result;
				
			}finally{
				jedis.close();
			}			
		}catch (Exception e) {
			logger.error(e.getMessage(), e);
		}
		return null;
	}
	
}
```
注意：在序列化的操作中，我们为了使高并发优化到极致，并不采取JDK默认的自己的序列化机制，从相关文档中可以发现protostuff序列化的速度和使用的空间都要优于JDK默认的序列化机制，所以我们使用protostuff进行序列化。（pom.xml写入相关依赖）
### 2. 秒杀操作的执行也是高并发产生的地方，如update操作和insert操作，涉及到行级锁
#### 优化分析：
行级锁在commit之后释放-->优化方向减少行级锁持有时间
#### 优化思路：
把客户端逻辑放到Mysql服务端，避免网络延迟和GC影响，减少事务锁时间
#### 具体操作:
在未做优化之前，事务要想结束，往往发生在java代码出现异常（roolback（仅限运行期异常）），成功返回（commit）的时候，而我们要做的是把事务的执行全交给数据库端去执行，这样可以减少事务锁时间，我们利用存储过程来达到优化的目的,在sql端新建一个存储过程：
```sql
CREATE PROCEDURE `seckill`.`execute_seckill`
     (in v_seckill_id bigint,in v_phone bigint,
	in v_kill_time timestamp,out r_result int)
     BEGIN
      DECLARE insert_count int DEFAULT 0;
      START TRANSACTION;
      insert ignore into success_killed
	(seckill_id,user_phone,create_time)
	values (v_seckill_id,v_phone,v_kill_time);
	select row_count() into insert_count;
       IF (insert_count = 0) THEN
	ROLLBACK;
	SET r_result = -1;
       ELSEIF(insert_count < 0) THEN
	ROLLBACK;
	SET r_result = -2;
       ELSE
	update seckill
	set number = number-1
	where seckill_id = v_seckill_id
	  and end_time > v_kill_time
	  and start_time < v_kill_time
          and number > 0;
	select row_count() into insert_count;
	IF(insert_count = 0) THEN
	  ROLLBACK;
	  SET r_result = 0;
	ELSEIF(insert_count < 0) THEN
	  ROLLBACK;
	  SET r_result = -2;
	ELSE
	  COMMIT;
	  SET r_result = 1;
	END IF;
      END IF;
     END;
```
这个存储过程已经帮你把事务的执行全部做好，并且会放回一个result告诉你执行的状态
```java
public SeckillExecution executeSeckillProcedure(long seckillId,
			long userPhone, String md5) {
		if (md5 == null || !md5.equals(getMD5(seckillId))) {
			return new SeckillExecution(seckillId,SeckillStatEnum.DATA_REWRITE);			
		}
		Date killTime = new Date();
		Map<String,Object> map = new HashMap<String,Object>();
		map.put("seckillId", seckillId);
		map.put("phone", userPhone);
		map.put("killTime", killTime);
		map.put("result", null);
		//执行存储过程，result被复制
		try {
			seckillDao.killByProcedure(map);
			//获取result
			int result = MapUtils.getInteger(map,"result",-2);
			if(result == 1){
				SuccessKilled sk = successKilledDao.queryByIdWithSeckill(seckillId, userPhone);
				return new SeckillExecution(seckillId,SeckillStatEnum.SUCCESS,sk);				
			}else{
				return new SeckillExecution(seckillId,SeckillStatEnum.stateOf(result));	
			}
		} catch (Exception e) {
			logger.error(e.getMessage(), e);
			return new SeckillExecution(seckillId,SeckillStatEnum.INNER_ERROR);
		}
		
	}
```
新增加的业务逻辑判断方法。
### 其他小亮点
1. 为了防止用户识别秒杀地址的拼接方法而提前去获得秒杀地址，造成对其他用户的不公平，我设置了一个盐值md5,里面的内容是我随便输入的字符串，通过spring下的DigestUtils.md5DigestAsHex（）方法来得到一个新生成的地址返回，这样用户也不知道秒杀地址的拼写规则。
2. 商品界面秒杀按钮设置了只能点击一次，防止用户因为卡顿不断地点击按钮，造成大量的请求影响服务器端的速度。
