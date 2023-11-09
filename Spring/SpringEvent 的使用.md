SpringEvent 的使用

1. 创建消息对象的实体类 （该类需要继承ApplicationEvent）

   ```java
   public class DemoEvent extends ApplicationEvent {
       @Getter
       @Setter
       private Long id;
       @Getter
       @Setter
       private String message;
   
   
   
       public DemoEvent(Object source, Long id, String message) {
           super(source);
           this.id = id;
           this.message = message;
       }
   }
   ```

2. 创建生产者

   生产者类需要实现ApplicationContextAware接口来注入ApplicationContext对象

   ```java
   @Component
   public class DemoPublish implements DisposableBean, ApplicationContextAware {
   
       @Autowired
       private ApplicationContext applicationContext;
   
       @Override
       public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
           this.applicationContext = applicationContext;
       }
           public void publish(long id, String message) {
   //        applicationContext.publishEvent(new DemoEvent(this, id, message));
           applicationContext.publishEvent(new DemoEvent(applicationContext,id,message));
       }
   
       @Override
       public void destroy() throws Exception {
           System.out.println("this bean destroy");
       }
   }
   ```

   

3. 创建消费者

   使用@EventListener注解

   如果需要异步的话，加上@Async注解

   示例：

   ```java
   @Component
   public class DemoListener {
   
       @Async
       @EventListener
       public void onApplicationEvent(DemoEvent demoEvent) {
           System.out.println(">>>>>>>>>DemoListener2>>>>>>>>>>>>>>>>>>>>>>>>>>>>");
           System.out.println("收到了：" + demoEvent.getSource() + "消息;时间：" + demoEvent.getTimestamp());
           System.out.println("消息：" + demoEvent.getId() + ":" + demoEvent.getMessage());
       }
   }
   ```

   