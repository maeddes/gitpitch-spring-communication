

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
            //messageList.add(in);
    }
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

---

# Containers

+++

```bash
$ docker run -d -p 5672:5672 -p 15672:15672 tutum/rabbitmq
Unable to find image 'tutum/rabbitmq:latest' locally
latest: Pulling from tutum/rabbitmq
faecf96fd5ab: Pulling fs layer
995977506e98: Pulling fs layer
efb63fb8dcb6: Pulling fs layer
a3ed95caeb02: Pulling fs layer
bc0233a814ac: Pulling fs layer
bfa9468b9a43: Pulling fs layer
```

+++

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                              NAMES
964df5273f31        tutum/rabbitmq      "/run.sh"                6 minutes ago       Up 6 minutes        0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp   keen_visvesvaraya
```

+++

```bash
$ docker logs 964df5273f31
=> Securing RabbitMQ with a random password
=> Done!
========================================================================
You can now connect to this RabbitMQ server using, for example:

    curl --user admin:6pjXZIEU5eam http://<host>:<port>/api/vhosts

Please remember to change the above password as soon as possible!
========================================================================

              RabbitMQ 3.6.1. Copyright (C) 2007-2016 Pivotal Software, Inc.
  ##  ##      Licensed under the MPL.  See http://www.rabbitmq.com/
  ##  ##
  ##########  Logs: /var/log/rabbitmq/rabbit@964df5273f31.log
  ######  ##        /var/log/rabbitmq/rabbit@964df5273f31-sasl.log
  ##########
              Starting broker... completed with 6 plugins.
```

+++

```properties
spring.jpa.hibernate.ddl-auto=create
spring.datasource.url=jdbc:mysql://192.168.99.100:3306/mydb
spring.datasource.username=matthias
spring.datasource.password=matthias

spring.rabbitmq.host = 192.168.99.100
spring.rabbitmq.port = 5672
spring.rabbitmq.username = admin
spring.rabbitmq.password = 6pjXZIEU5eam
```


