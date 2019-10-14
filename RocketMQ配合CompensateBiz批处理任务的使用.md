## RocketMQ配合CompensateBiz批处理任务的使用

#### CompensateBiz任务

1. CompensateBiz表结构

   ```sql
   drop table if exists `compensate_biz`;
   CREATE TABLE `compensate_biz` (
     `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
     `biz_id` VARCHAR(128) COMMENT '业务id',
     `biz_type` VARCHAR(128) NOT NULL COMMENT '业务类型',
     `context` LONGTEXT  COMMENT '明细内容',
     `status` VARCHAR(32) COMMENT '状态',
     `cnt` tinyint(4) DEFAULT 0 COMMENT '失败次数',
     `priority` TINYINT(4) COMMENT '优先级',
     `last_failed_reason` TEXT  COMMENT '上次失败原因',
     `created_at` datetime DEFAULT NULL COMMENT '创建时间',
     `updated_at` datetime DEFAULT NULL COMMENT '更新时间',
     PRIMARY KEY (`id`)
   )  COMMENT='业务处理表';
   ```

   

2. compensate_biz表的用途

   > 在任何一个模块中可能存在一些不同的业务需要处理，如宝胜电子券业务中存在，注册送券，批量推送活动券，发券之后的消息推送等等。对于不同的业务，我们为了保证业务数据的完整性，安全性。所有的数据需要先落入数据库。之后通过对compensate_biz这张表的操作进行业务处理

3. compensate_biz处理流程（以宝胜电子券注册送券，宝胜中台发货单发货为例）

   > - 宝胜电子券注册送券业务是一个高频业务，也是一个跨系统调用的业务，主要是会员模块发起，会员调用电子券的注册送券接口之后，之前是一个同步的方法耗时较多，容易超时。为了解决这个问题，经过必要的参数校验之后，直接在compensate_biz表中插入一条biz_type为注册送券类型的记录，然后发送mq消息，mq消息的消费端接收到消息之后进行具体的业务处理。如果mq消息消费失败之后，电子券有相应的Scheduled五分钟扫描一下调度任务表，将待处理的数据进行处理。
   > - 宝胜中台发货单发货是一个批量推送的接口，是一个跨系统调用的业务，在下游的EDI发货之后，会将已经发货的发货单通知给宝胜中台，中台对于数据做必要的校验之后直接在compensate_biz表中插入一条biz_type为发货单发货类型的记录，之后中台Scheduled一分钟扫描一下任务，将待处理的任务处理，将发货结果通知中台，通知第三方。

4. 为什么需要compensate_biz任务

   > 对于大型业务系统来讲，同步操作涉及的链路一般较长，我们可以将较长的链路进行拆分，拆分成一个个较小的链路，有利于提高效率。
   >
   > 比如会员调用电子券批量送券的业务，电子券校验好基本参数之后直接将数据插入compensate_biz表先生成批量处理的任务，批量处理完了之后，生成回调营销中心的任务，以及更新实际发券数量的任务。
   >
   > 这样业务的复杂度降低，业务细粒度增强，同时系统也做到了解耦，模块化。

5. compensate_biz任务的领域模型。

   > 1.  实际上是一个工厂方法的设计模式或者是一个Facade的设计模式，如下：
   >
   > ```java
   > 
   > /**
   >  * Author:  <a href="mailto:zhaoxiaotao@terminus.io">tony</a>
   >  * Date: 2018/5/28
   >  * pousheng-middle
   >  */
   > public interface CompensateBizService {
   > 
   >     /**
   >      * 业务处理过程
   >      * @param compensateBiz 业务处理domain
   >      * @return
   >      */
   >     public void doProcess(CompensateBiz compensateBiz);
   > }
   > 
   > ```
   >
   > 实际上所有的业务处理都实现了CompensateBizService这个接口，通过biz_type来实现具体调用哪个接口。

6. CompensateBiz补偿任务Job

   > job每隔三分钟处理一次
   >
   > 查询到compensateBiz任务之后，以乐观锁的方式更新处理任务为处理中

   ```java
   @Slf4j
   @RestController
   public class CompensateBizWaitHandleJob {
       @Autowired
       private CompensateBizReadService compensateBizReadService;
       @Autowired
       private CompensateBizWriteService compensateBizWriteService;
       @Autowired
       private CompensateBizProcessor compensateBizProcessor;
   
       @Scheduled(cron = "0 */3 * * * ?")
       @GetMapping("/api/compensate/biz/wait/handle/job")
       public void processWaitHandleJob() {
           log.info("[pousheng-middle-compensate-biz-wait-handle-job] start...");
           Stopwatch stopwatch = Stopwatch.createStarted();
           Integer pageNo = 1;
           Integer pageSize = 100;
           while (true) {
               CompensateBizCriteria criteria = new CompensateBizCriteria();
               criteria.setPageNo(pageNo);
               criteria.setPageSize(pageSize);
               criteria.setStatus(CompensateBizStatus.WAIT_HANDLE.name());
               Response<Paging<CompensateBiz>> response = compensateBizReadService.paging(criteria);
               if (!response.isSuccess()) {
                   pageNo++;
                   continue;
               }
               List<CompensateBiz> compensateBizs =  response.getResult().getData();
               if (compensateBizs.isEmpty()){
                   break;
               }
               for (CompensateBiz compensateBiz:compensateBizs){
                   if (compensateBiz.getCnt()>3){
                       continue;
                   }
                   //乐观锁控制更新为处理中
                   Response<Boolean> rU=  compensateBizWriteService.updateStatus(compensateBiz.getId(),compensateBiz.getStatus(),CompensateBizStatus.PROCESSING.name());
                   if (!rU.isSuccess()){
                       continue;
                   }
                   try{
                       //业务处理
                       compensateBizProcessor.doProcess(compensateBiz);
                       
                   }catch (BizException e0){
                       //抛出异常后将任务重置为处理失败
                		
                   }catch (Exception e1){
                    	//抛出异常后将任务重置为处理失败
                   }
   
               }
               pageNo++;
   
           }
           stopwatch.stop();
           log.info("[pousheng-middle-compensate-biz-wait-handle-job] end");
       }
   ```

### RocketMq配置

1. RocketMq包引入

   ```
           <!--rocketMQ 依赖包-->
           <dependency>
               <groupId>org.apache.rocketmq</groupId>
               <artifactId>spring-boot-starter-rocketmq</artifactId>
               <version>1.0.0-SNAPSHOT</version>
           </dependency>
   ```

   

2. yml文件配置

   > name-server配置，连接broker
   >
   > producer-group:一个工程配置一个就可以了

   ```yaml
   spring:
    rocketmq:
        name-server: 127.0.0.1:9876   
        producer:
           group: pousheng-promotion-producer-group
   ```

3. MQ消息的生成代码

   > 消息的生成封装了两种方法
   >
   > 1.立即发送
   >
   > 2.设置了延迟级别
   >
   > 3.默认发送超时时间为3000毫秒

   ```java
   
   /**
    * Author:  <a href="mailto:zhaoxiaotao@terminus.io">tony</a>
    * Date: 2018/5/28
    * pousheng-middle
    */
   @Slf4j
   @Service
   public class RocketMqProducerService {
       @Autowired(required = false)
       private RocketMQTemplate rocketMQTemplate;
   
       /**
        * 立即发送MQ消息
        * @param message messge必须是json字符串
        */
       public void sendMessage(String topic, String message) {
           if (rocketMQTemplate == null) {
               log.error("Error of sending message to MQ: no configure here, message: {}",message);
               return;
           }
   
           // 开始发送消息
           try {
               if (log.isDebugEnabled()) {
                   log.debug("Sending message to MQ: {}",message);
               }
   
               byte[] bytes = message.getBytes(Charset.forName("UTF-8"));
   
               DefaultMQProducer producer = rocketMQTemplate.getProducer();
               org.apache.rocketmq.common.message.Message msg =
                       new org.apache.rocketmq.common.message.Message(topic, "", bytes);
               //消息异步发送
               SendCallback sendCallback  = new SendCallback() {
                   @Override
                   public void onSuccess(SendResult sendResult) {
                       //必须打印msgId用来以备查验
                       log.info("Sending message to MQ: {},msgId {}",message,sendResult.getMsgId());
                   }
   
                   @Override
                   public void onException(Throwable e) {
                       log.error("ending message to MQ failed,msg {}",message);
                   }
               };
               //消息发送
               producer.send(msg,sendCallback,(long)producer.getSendMsgTimeout());
   
           }
           catch (Exception ex) {
               log.error("Error of send message to MQ! message: {}, stackTrace: {}",
                       message, Throwables.getStackTraceAsString(ex));
           }
       }
   
       /**
        * 尝试发送RocketMQ的延迟消息，如果level为空，那么将立即发送消息
        * @param message message必须是json字符串
        * @param level
        */
       public void sendMessage(String topic, String message, MessageLevel level) {
           if (level == null) {
               if (log.isDebugEnabled()) {
                   log.debug("No messageLevel, choose normal channel...");
               }
   
               this.sendMessage(topic, message);
               return;
           }
   
           if (rocketMQTemplate == null) {
               log.error("Error of sending message to MQ: no configure here, message: {}", message);
               return;
           }
   
           try {
               long now = System.currentTimeMillis();
               byte[] bytes = message.getBytes(Charset.forName("UTF-8"));
   
               DefaultMQProducer producer = rocketMQTemplate.getProducer();
               org.apache.rocketmq.common.message.Message msg =
                       new org.apache.rocketmq.common.message.Message(topic, "", bytes);
   
                // 设置延迟级别
               msg.setDelayTimeLevel(level.resolve());
               //消息异步发送
               SendCallback sendCallback  = new SendCallback() {
                   @Override
                   public void onSuccess(SendResult sendResult) {
                       //必须打印msgId用来以备查验
                       log.info("Sending message to MQ: {},msgId          {}",message,sendResult.getMsgId());
                   }
                   @Override
                   public void onException(Throwable e) {
                       log.error("ending message to MQ failed,msg {}",message);
                   }
               };
               
               producer.send(msg,sendCallback, (long)producer.getSendMsgTimeout());
               
               long costTime = System.currentTimeMillis() - now;
   
           } catch (Exception ex) {
               log.error("Error of send message to MQ! message: {}, stackTrace: {}",
                       JSON.toJSON(message), Throwables.getStackTraceAsString(ex));
           }
   
       }
   }
   
   ```

   

4. MQ消息消费的代码

   > 消费端目前只是消费了compensate_biz表的记录，可以设置不同topic的consumer
   >
   > @RocketMQMessageListener详解
   >
   > 1. 目前我们使用的默认使用的是集群模式，非广播模式。
   > 2. 默认使用推送消息(push)模式，而非pull模式
   > 3. 实现该注解的实现对应的topic，会收到相应的topic的消息。
   > 4. message消息是经过UTF-8编码，收到消息记得解析
   >
   > 

   ```java
   
   /**
    * 消费线程数量设置为6个(具体情况需要根据机器性能做适配)
    * CompenSateBiz mq处理
    */
   @Slf4j
   @Service
   @RocketMQMessageListener(topic = MqConstant.COMPENSATE_BIZ_TOPIC,
           consumerGroup = MqConstant.POUSHENG_PROMOTION_MQ_CONSUMER_GROUP,consumeThreadMax = 6)
   public class CompensateBizConsumerMessageListener implements RocketMQListener<String> {
       @Autowired
       private CompensateBizReadService compensateBizReadService;
       @Autowired
       private CompensateBizWriteService compensateBizWriteService;
       @Autowired
       private CompensateBizProcessor compensateBizProcessor;
       @Override
       public void onMessage(String message) {
           CompensateBiz compensateBiz = null;
           try{
               if (log.isDebugEnabled()){
                   log.debug("CompensateBizConsumerMessageListener onMessage,message {}",message);
               }
               //获取bizId，为什么不把整个Compensatebiz拿过来，这边还要查一遍，因为biz的bean可能很大，太大了的话对于网络传输有影响，可能导致mq挂掉
               String compensateBizId = JsonMapper.nonEmptyMapper().fromJson(message,String.class);
               
               //业务处理
               compensateBizProcessor.doProcess(compensateBiz);
   
           }catch (BizException e0){
               //失败之后业务处理
           }catch (Exception e1){
               //失败之后业务处理
           }
       }
   }
   
   ```



## 总结

> 1. 在宝胜电子券业务中使用了CompensateBiz任务结合RocketMq高效处理 批处理任务，尝试了一下，给265万用户发送券，如果不设置RocketMq消费线程默认consumeThreadMax=64，10分钟可以给用户发送530万张券，基于（2018-07-04）压测结果。但是CPU负载很高，实际情况如果需要实现这样的性能，需要水平扩展。
>
> 2. 补偿任务Job，3分钟扫描一次，查看是否有待处理的任务。主要是用于RocketMq一旦消息丢失，做一个兜底的任务。
>
>    




