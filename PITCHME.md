

# Communication in Spring

---

# Synchronous communication

+++

## Using REST Template

+++

Service with REST Endpoint exposed

```java

@SpringBootApplication
@RestController
public class SpringApplication {

    @GetMapping("/getSomething")
    public String getSomething() {

        // do something
	return "something";

    }

```

+++

Other service calling REST Endpoint

```java
String callService(){

    RestTemplate template = new RestTemplate();
    String url = "http://localhost:8080/getSomething";

    ResponseEntity<String> response = template.getForEntity(url, String.class);

    return response.getBody();

}
```

+++

```java
@SpringBootApplication
@RestController
public class SpringAmqpApplication {

    @Autowired
    AMQPSender sender;

    @Autowired
    AMQPReceiver receiver;
```


---

# Asynchronous Communication

---

### Queue based
## Using AMQP (RabbitMQ)

+++

```java
@Configuration
public class AMQPConfig {

    @Bean
    public Queue hello() {
        return new Queue("my-queue");
    }

    @Bean
    public AMQPReceiver receiver() {
        return new AMQPReceiver();
    }

    @Bean
    public AMQPSender sender() {
        return new AMQPSender();
    }

}
```

+++

```java
@Configuration
public class AMQPConfig {

    @Bean
    public Queue hello() {
        return new Queue("my-queue");
    }

    @Bean
    public AMQPReceiver receiver() {
        return new AMQPReceiver();
    }

    @Bean
    public AMQPSender sender() {
        return new AMQPSender();
    }

}
```

+++

```java
public class AMQPSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private Queue queue;

    public void send(String message) {
        this.rabbitTemplate.convertAndSend(queue.getName(), message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}
```

+++


```java
@RabbitListener(queues = "my-queue")
public class AMQPReceiver {

    @RabbitHandler
    public void receive(String in) {
            System.out.println("Received '" + in + "'");
            messageList.add(in);
    }

}
```

---


### Publish/Subscribe

## Using Redis

+++

```java
    @Bean
    JedisConnectionFactory jedisConnectionFactory() {
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        final RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(jedisConnectionFactory());
        template.setValueSerializer(new GenericToStringSerializer<Object>(Object.class));
        return template;
    }

    @Bean
    MessageListenerAdapter messageListener() {
        return new MessageListenerAdapter(new RedisMessageSubscriber());
    }
```

+++

```java
    @Bean
    RedisMessageListenerContainer redisContainer() {
        final RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(jedisConnectionFactory());
        container.addMessageListener(messageListener(), topic());
        return container;
    }

    @Bean
    MessagePublisher redisPublisher() {
        return new RedisMessagePublisher(redisTemplate(), topic());
    }

    @Bean
    ChannelTopic topic() {
        return new ChannelTopic("pubsub:queue");
    }
```

+++

```java
@Service
public class RedisMessagePublisher implements MessagePublisher {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private ChannelTopic topic;

    public RedisMessagePublisher() {

        this.redisTemplate = redisTemplate;

    }

    public RedisMessagePublisher(final RedisTemplate<String, Object> redisTemplate, final ChannelTopic topic) {
        this.topic = topic;
    }

    public void publish(final String message) {
        redisTemplate.convertAndSend(topic.getTopic(), message);
    }
}
```

+++

```java
@Service
public class RedisMessageSubscriber implements MessageListener {

    public static List<String> messageList = new ArrayList<String>();

    public void onMessage(final Message message, final byte[] pattern) {
        messageList.add(message.toString());
        System.out.println("Message received: " + new String(message.getBody()));
    }

    public String getMessages(){

        return messageList.toString();

    }
}

```

+++

```java
@SpringBootApplication
@RestController
public class SpringRedisPubsubApplication {

	@Autowired
	private RedisMessagePublisher redisMessagePublisher;

	@Autowired
	private RedisMessageSubscriber redisMessageSubscriber;

```



