## Java 学习

[toc]



### Json Object

1. 可以将 String 转换 为 class 

   ```java
   // 这是一个 JSON string
   String bodyStr = new String(JSON_String );
   
   // 这边 开始看着这个 String 嫩个不能 复合 JSON 格式
   JSONObject parsedStringObject = JSONObject.parseObject(bodyStr);
   
   // 最后 将 String 转换成我们需要的 Class
   UserMoment userMoment = JSONObject.toJavaObject(parsedStringObject, UserMoment.class)
   ```

### Configuration 

1. 在 Configuration 中 定一个 bean 的方法，然后在别的 service 中调用 该方法

   这样的话就是定义了一个 Bean 

   ```java
   @Configuration
   public class RocketMqConfig {
    @Bean("readyToUseBean")
       public String test(){
                return "Should receive this value";
       }
   }
   ```

   现在我们要在别的 方法里面调用 这个 Bean 

   ```java
   // 这个使整个运行程序的上下文
   @Autowired
   private ApplicationContext applicationContext;
   
   public void addUserMoments(UserMoment userMoment) throws MQBrokerException, RemotingException, InterruptedException, MQClientException {
       String sampleString = String applicationContext.getBean("readyToUseBean");
        return sampleString;
   }
   ```

   