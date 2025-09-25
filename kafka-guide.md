# Руководство по Kafka для Java/Kotlin разработчика

## Содержание
1. [Архитектура и основные концепции Kafka](#архитектура-и-основные-концепции-kafka)
2. [Гарантии доставки сообщений](#гарантии-доставки-сообщений)
3. [Работа с Kafka в Spring Boot](#работа-с-kafka-в-spring-boot)
4. [Обработка ошибок и отказоустойчивость](#обработка-ошибок-и-отказоустойчивость)
5. [Транзакционная обработка в Kafka](#транзакционная-обработка-в-kafka)
6. [Мониторинг и диагностика](#мониторинг-и-диагностика)
7. [Тестирование с Kafka](#тестирование-с-kafka)
8. [Ответы на вопросы с собеседований](#ответы-на-вопросы-с-собеседований)

## Архитектура и основные концепции Kafka

### Ключевые компоненты
- **Брокер**: Сервер Kafka, обрабатывающий публикацию и подписку на сообщения
- **Топик**: Категория или поток сообщений, разделенный на партиции
- **Партиция**: Единица параллелизма, упорядоченная последовательность сообщений
- **Producer**: Клиент, отправляющий сообщения в топики
- **Consumer**: Клиент, читающий сообщения из топиков
- **Consumer Group**: Группа потребителей, балансирующих нагрузку
- **Offset**: Позиция сообщения внутри партиции
- **Zookeeper/KRaft**: Координация кластера (в новых версиях ZK заменяется на KRaft)

### Схема архитектуры
```
                          ┌───────────┐
                          │   Topic   │
                          └───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌─────────┐      ┌─────────┐      ┌─────────┐
        │Partition│      │Partition│      │Partition│
        │    0    │      │    1    │      │    2    │
        └─────────┘      └─────────┘      └─────────┘
              ▲                ▲                ▲
              │                │                │
    ┌─────────┴────────────────┴────────────────┴─────────┐
    │                     Producer                         │
    └───────────────────────────────────────────────────┬─┘
                                                        │
                                                        ▼
    ┌───────────────────────────────────────────────────┐
    │                Kafka Cluster                       │
    │  ┌───────────┐    ┌───────────┐    ┌───────────┐  │
    │  │  Broker 1 │    │  Broker 2 │    │  Broker 3 │  │
    │  └───────────┘    └───────────┘    └───────────┘  │
    └───────────────────────────────────────────────────┘
                          ▲
                          │
    ┌─────────────────────┴──────────────────────────────┐
    │                Consumer Group                       │
    │  ┌───────────┐    ┌───────────┐    ┌───────────┐   │
    │  │Consumer 1 │    │Consumer 2 │    │Consumer 3 │   │
    │  └───────────┘    └───────────┘    └───────────┘   │
    └───────────────────────────────────────────────────┘
```

### Особенности Kafka
- **Распределенная система**: горизонтальное масштабирование
- **Высокая пропускная способность**: обработка миллионов сообщений в секунду
- **Долговременное хранение**: хранение данных с настраиваемым retention
- **Репликация**: отказоустойчивость с настраиваемым фактором репликации
- **Параллельная обработка**: чтение/запись в разные партиции одновременно

## Гарантии доставки сообщений

### Уровни гарантий

#### 1. At-most-once (не более одного раза)
```java
// Java Producer конфигурация
Properties props = new Properties();
props.put(ProducerConfig.ACKS_CONFIG, "0"); // No acknowledgment
props.put(ProducerConfig.RETRIES_CONFIG, "0"); // No retries
```

```kotlin
// Kotlin Producer конфигурация
val props = Properties().apply {
    put(ProducerConfig.ACKS_CONFIG, "0") // No acknowledgment
    put(ProducerConfig.RETRIES_CONFIG, "0") // No retries
}
```

- **Применение**: метрики, логи, где потеря данных не критична
- **Преимущество**: наивысшая производительность
- **Недостаток**: возможна потеря сообщений

#### 2. At-least-once (как минимум один раз)
```java
// Java Producer конфигурация
Properties props = new Properties();
props.put(ProducerConfig.ACKS_CONFIG, "all"); // Wait for all replicas
props.put(ProducerConfig.RETRIES_CONFIG, "3"); // Retry 3 times
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "false");
```

```kotlin
// Kotlin Producer конфигурация
val props = Properties().apply {
    put(ProducerConfig.ACKS_CONFIG, "all") // Wait for all replicas
    put(ProducerConfig.RETRIES_CONFIG, "3") // Retry 3 times
    put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "false")
}
```

- **Применение**: стандартный вариант для большинства приложений
- **Преимущество**: гарантирует, что данные не будут потеряны
- **Недостаток**: возможны дубликаты сообщений

#### 3. Exactly-once (ровно один раз)
```java
// Java Producer конфигурация
Properties props = new Properties();
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-id-1");
```

```kotlin
// Kotlin Producer конфигурация
val props = Properties().apply {
    put(ProducerConfig.ACKS_CONFIG, "all")
    put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE.toString())
    put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true")
    put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-id-1")
}
```

- **Применение**: финансовые транзакции, критичные бизнес-операции
- **Преимущество**: гарантирует отсутствие дубликатов и потерь
- **Недостаток**: снижение производительности

### Настройки Producer для надежности

```java
// Java
Properties props = new Properties();
// Ожидание подтверждения от всех реплик
props.put(ProducerConfig.ACKS_CONFIG, "all");
// Максимальное количество попыток отправки
props.put(ProducerConfig.RETRIES_CONFIG, "5");
// Время ожидания между повторными попытками
props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, "100");
// Включение идемпотентности для избежания дубликатов
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
// Размер буфера сообщений в памяти
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, "33554432");
// Максимальное количество сообщений в batch
props.put(ProducerConfig.BATCH_SIZE_CONFIG, "16384");
// Время ожидания для заполнения batch
props.put(ProducerConfig.LINGER_MS_CONFIG, "10");
// Максимальный размер запроса
props.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG, "1048576");
```

### Настройки Consumer для надежности

```java
// Java
Properties props = new Properties();
// Автоматический коммит смещений
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
// Интервал автокоммита (если включен)
props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "5000");
// Стратегия при отсутствии сохраненных смещений
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
// Таймаут сессии для consumer group
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000");
// Heart beat interval
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, "10000");
// Максимальное количество записей в одном poll
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "500");
// Максимальное время обработки между polls
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, "300000");
```

## Работа с Kafka в Spring Boot

### Зависимости

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```kotlin
// Gradle (Kotlin DSL)
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
}
```

### Конфигурация в application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
    consumer:
      group-id: my-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.mycompany.model
      enable-auto-commit: false
```

### Producer с Spring Kafka

```java
// Java
@Service
public class PaymentEventProducer {
    
    private static final String TOPIC = "payment-events";
    
    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    public PaymentEventProducer(KafkaTemplate<String, PaymentEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public CompletableFuture<SendResult<String, PaymentEvent>> sendPaymentEvent(PaymentEvent event) {
        // Используем ID платежа как ключ для обеспечения упорядоченности
        // сообщений для одного платежа (попадут в одну партицию)
        String key = event.getPaymentId();
        
        return kafkaTemplate.send(TOPIC, key, event)
                .completable()
                .exceptionally(e -> {
                    // Обработка ошибок при отправке
                    log.error("Unable to send payment event: {}", event, e);
                    throw new KafkaProducerException("Could not send message", e);
                });
    }
}
```

```kotlin
// Kotlin
@Service
class PaymentEventProducer(
    private val kafkaTemplate: KafkaTemplate<String, PaymentEvent>
) {
    
    companion object {
        private const val TOPIC = "payment-events"
    }
    
    fun sendPaymentEvent(event: PaymentEvent): CompletableFuture<SendResult<String, PaymentEvent>> {
        // Используем ID платежа как ключ для обеспечения упорядоченности
        val key = event.paymentId
        
        return kafkaTemplate.send(TOPIC, key, event)
            .completable()
            .exceptionally { e ->
                // Обработка ошибок при отправке
                log.error("Unable to send payment event: {}", event, e)
                throw KafkaProducerException("Could not send message", e)
            }
    }
}
```

### Consumer с Spring Kafka

```java
// Java
@Service
public class PaymentEventConsumer {
    
    private final PaymentService paymentService;
    
    public PaymentEventConsumer(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
    
    @KafkaListener(
        topics = "payment-events",
        groupId = "payment-processing-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consumePaymentEvent(
            @Payload PaymentEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {
        
        try {
            log.info("Received payment event: {} from partition {} at offset {}", 
                    event, partition, offset);
                    
            // Обработка платежа
            paymentService.processPayment(event);
            
            // Ручное подтверждение обработки сообщения
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            log.error("Error processing payment event: {}", event, e);
            // Решение: пропустить, повторить, отправить в DLQ?
            // В данном примере пропускаем, но в реальности нужна более сложная логика
            acknowledgment.acknowledge();
        }
    }
}
```

```kotlin
// Kotlin
@Service
class PaymentEventConsumer(
    private val paymentService: PaymentService
) {
    
    @KafkaListener(
        topics = ["payment-events"],
        groupId = "payment-processing-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    fun consumePaymentEvent(
        @Payload event: PaymentEvent,
        @Header(KafkaHeaders.RECEIVED_PARTITION) partition: Int,
        @Header(KafkaHeaders.OFFSET) offset: Long,
        acknowledgment: Acknowledgment
    ) {
        try {
            log.info("Received payment event: {} from partition {} at offset {}", 
                event, partition, offset)
                
            // Обработка платежа
            paymentService.processPayment(event)
            
            // Ручное подтверждение обработки сообщения
            acknowledgment.acknowledge()
            
        } catch (e: Exception) {
            log.error("Error processing payment event: {}", event, e)
            // Решение: пропустить, повторить, отправить в DLQ?
            // В данном примере пропускаем, но в реальности нужна более сложная логика
            acknowledgment.acknowledge()
        }
    }
}
```

### Конфигурация Producer и Consumer

```java
// Java
@Configuration
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.mycompany.model");
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        
        // Настройка параллелизма
        factory.setConcurrency(3); // 3 потока на партицию
        
        // Настройка обработчика ошибок
        factory.setErrorHandler((exception, data) -> {
            log.error("Error in Kafka listener: {}", exception.getMessage());
        });
        
        return factory;
    }
}
```

```kotlin
// Kotlin
@Configuration
class KafkaConfig {
    
    @Bean
    fun producerFactory(): ProducerFactory<String, Any> {
        val configProps = mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to JsonSerializer::class.java,
            ProducerConfig.ACKS_CONFIG to "all",
            ProducerConfig.RETRIES_CONFIG to 3,
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true
        )
        
        return DefaultKafkaProducerFactory(configProps)
    }
    
    @Bean
    fun kafkaTemplate(): KafkaTemplate<String, Any> = KafkaTemplate(producerFactory())
    
    @Bean
    fun consumerFactory(): ConsumerFactory<String, Any> {
        val props = mapOf(
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ConsumerConfig.GROUP_ID_CONFIG to "my-group",
            ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
            ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to JsonDeserializer::class.java,
            ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG to false,
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "earliest",
            JsonDeserializer.TRUSTED_PACKAGES to "com.mycompany.model"
        )
        
        return DefaultKafkaConsumerFactory(props)
    }
    
    @Bean
    fun kafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, Any> {
        return ConcurrentKafkaListenerContainerFactory<String, Any>().apply {
            consumerFactory = consumerFactory()
            containerProperties.ackMode = ContainerProperties.AckMode.MANUAL_IMMEDIATE
            
            // Настройка параллелизма
            setConcurrency(3) // 3 потока на партицию
            
            // Настройка обработчика ошибок
            setErrorHandler { exception, data ->
                log.error("Error in Kafka listener: {}", exception.message)
            }
        }
    }
}
```

## Обработка ошибок и отказоустойчивость

### Стратегии обработки ошибок

#### 1. Retry (повторная обработка)

```java
// Java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    
    // Настройка политики повторов
    SeekToCurrentErrorHandler errorHandler = 
            new SeekToCurrentErrorHandler(new FixedBackOff(1000L, 3L));
    factory.setErrorHandler(errorHandler);
    
    return factory;
}
```

```kotlin
// Kotlin
@Bean
fun kafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, Any> {
    return ConcurrentKafkaListenerContainerFactory<String, Any>().apply {
        consumerFactory = consumerFactory()
        
        // Настройка политики повторов
        val errorHandler = SeekToCurrentErrorHandler(FixedBackOff(1000L, 3L))
        setErrorHandler(errorHandler)
    }
}
```

#### 2. Dead Letter Queue (DLQ)

```java
// Java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    
    // Добавляем DeadLetterPublishingRecoverer для отправки в DLQ
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate(),
            (record, exception) -> new TopicPartition(record.topic() + ".DLT", record.partition())
    );
    
    // Настройка политики повторов с последующей отправкой в DLQ
    SeekToCurrentErrorHandler errorHandler = 
            new SeekToCurrentErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
    factory.setErrorHandler(errorHandler);
    
    return factory;
}
```

```kotlin
// Kotlin
@Bean
fun kafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, Any> {
    return ConcurrentKafkaListenerContainerFactory<String, Any>().apply {
        consumerFactory = consumerFactory()
        
        // Добавляем DeadLetterPublishingRecoverer для отправки в DLQ
        val recoverer = DeadLetterPublishingRecoverer(
            kafkaTemplate(),
            { record, exception -> TopicPartition("${record.topic()}.DLT", record.partition()) }
        )
        
        // Настройка политики повторов с последующей отправкой в DLQ
        val errorHandler = SeekToCurrentErrorHandler(recoverer, FixedBackOff(1000L, 3L))
        setErrorHandler(errorHandler)
    }
}
```

#### 3. Обработка исключений на уровне слушателя

```java
// Java
@KafkaListener(topics = "payment-events")
public void consumePaymentEvent(PaymentEvent event, Acknowledgment ack) {
    try {
        paymentService.processPayment(event);
        ack.acknowledge();
    } catch (TemporaryException e) {
        // Временная ошибка - можно повторить
        log.warn("Temporary error processing event: {}", event, e);
        // Не подтверждаем сообщение, оно будет доставлено снова
        throw new RetryableException("Temporary failure", e);
    } catch (PermanentException e) {
        // Постоянная ошибка - нет смысла повторять
        log.error("Permanent error processing event: {}", event, e);
        // Подтверждаем сообщение, чтобы перейти к следующему
        ack.acknowledge();
        // Отправляем событие в DLQ вручную
        kafkaTemplate.send("payment-events.DLT", event);
    }
}
```

```kotlin
// Kotlin
@KafkaListener(topics = ["payment-events"])
fun consumePaymentEvent(event: PaymentEvent, ack: Acknowledgment) {
    try {
        paymentService.processPayment(event)
        ack.acknowledge()
    } catch (e: TemporaryException) {
        // Временная ошибка - можно повторить
        log.warn("Temporary error processing event: {}", event, e)
        // Не подтверждаем сообщение, оно будет доставлено снова
        throw RetryableException("Temporary failure", e)
    } catch (e: PermanentException) {
        // Постоянная ошибка - нет смысла повторять
        log.error("Permanent error processing event: {}", event, e)
        // Подтверждаем сообщение, чтобы перейти к следующему
        ack.acknowledge()
        // Отправляем событие в DLQ вручную
        kafkaTemplate.send("payment-events.DLT", event)
    }
}
```

### Паттерны обработки ошибок

#### Circuit Breaker

```java
// Java
// С использованием Spring Cloud Circuit Breaker
@Service
public class KafkaProducerWithCircuitBreaker {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final CircuitBreakerFactory circuitBreakerFactory;
    
    public KafkaProducerWithCircuitBreaker(
            KafkaTemplate<String, Object> kafkaTemplate,
            CircuitBreakerFactory circuitBreakerFactory) {
        this.kafkaTemplate = kafkaTemplate;
        this.circuitBreakerFactory = circuitBreakerFactory;
    }
    
    public CompletableFuture<SendResult<String, Object>> sendMessage(String topic, Object message) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("kafka-send");
        
        return circuitBreaker.run(
            () -> kafkaTemplate.send(topic, message).completable(),
            throwable -> {
                log.error("Circuit breaker opened, failing fast", throwable);
                return CompletableFuture.failedFuture(
                    new KafkaException("Failed to send message due to circuit open", throwable)
                );
            }
        );
    }
}
```

#### Idempotent Consumer

```java
// Java
@Service
public class IdempotentPaymentProcessor {
    
    private final ProcessedEventsRepository processedEventsRepository;
    private final PaymentService paymentService;
    
    @Transactional
    @KafkaListener(topics = "payment-events")
    public void processPaymentEvent(
            @Payload PaymentEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {
        
        String eventId = event.getEventId();
        
        // Проверяем, было ли уже обработано это событие
        if (processedEventsRepository.existsByEventId(eventId)) {
            log.info("Event already processed, skipping: {}", eventId);
            ack.acknowledge();
            return;
        }
        
        try {
            // Обработка события
            paymentService.processPayment(event);
            
            // Сохраняем запись о том, что событие обработано
            ProcessedEvent processedEvent = new ProcessedEvent(
                eventId,
                "payment-events",
                partition,
                offset,
                LocalDateTime.now()
            );
            processedEventsRepository.save(processedEvent);
            
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process event: {}", eventId, e);
            throw e; // Перевыбрасываем для обработки глобальным обработчиком ошибок
        }
    }
}
```

```kotlin
// Kotlin
@Service
class IdempotentPaymentProcessor(
    private val processedEventsRepository: ProcessedEventsRepository,
    private val paymentService: PaymentService
) {
    
    @Transactional
    @KafkaListener(topics = ["payment-events"])
    fun processPaymentEvent(
        @Payload event: PaymentEvent,
        @Header(KafkaHeaders.RECEIVED_PARTITION) partition: Int,
        @Header(KafkaHeaders.OFFSET) offset: Long,
        ack: Acknowledgment
    ) {
        val eventId = event.eventId
        
        // Проверяем, было ли уже обработано это событие
        if (processedEventsRepository.existsByEventId(eventId)) {
            log.info("Event already processed, skipping: {}", eventId)
            ack.acknowledge()
            return
        }
        
        try {
            // Обработка события
            paymentService.processPayment(event)
            
            // Сохраняем запись о том, что событие обработано
            val processedEvent = ProcessedEvent(
                eventId = eventId,
                topic = "payment-events",
                partition = partition,
                offset = offset,
                processedAt = LocalDateTime.now()
            )
            processedEventsRepository.save(processedEvent)
            
            ack.acknowledge()
        } catch (e: Exception) {
            log.error("Failed to process event: {}", eventId, e)
            throw e // Перевыбрасываем для обработки глобальным обработчиком ошибок
        }
    }
}
```

## Транзакционная обработка в Kafka

### Транзакционный Producer

```java
// Java
@Service
public class TransactionalPaymentProducer {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public TransactionalPaymentProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    // Настройка транзакционного producer
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        // ... базовые настройки ...
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-payment-producer");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Transactional
    public void processAndSendPayment(Payment payment) {
        // Сохраняем платеж в БД
        Payment savedPayment = paymentRepository.save(payment);
        
        // Отправляем событие в Kafka в рамках той же транзакции
        PaymentEvent event = new PaymentEvent(savedPayment.getId(), savedPayment.getStatus());
        kafkaTemplate.send("payment-events", event);
        
        // Если здесь произойдет исключение, транзакция откатится,
        // и сообщение не будет отправлено в Kafka
    }
}
```

```kotlin
// Kotlin
@Service
class TransactionalPaymentProducer(
    private val kafkaTemplate: KafkaTemplate<String, Any>
) {
    
    // Настройка транзакционного producer
    @Bean
    fun producerFactory(): ProducerFactory<String, Any> {
        val props = mapOf(
            // ... базовые настройки ...
            ProducerConfig.TRANSACTIONAL_ID_CONFIG to "tx-payment-producer",
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true
        )
        return DefaultKafkaProducerFactory(props)
    }
    
    @Transactional
    fun processAndSendPayment(payment: Payment) {
        // Сохраняем платеж в БД
        val savedPayment = paymentRepository.save(payment)
        
        // Отправляем событие в Kafka в рамках той же транзакции
        val event = PaymentEvent(savedPayment.id, savedPayment.status)
        kafkaTemplate.send("payment-events", event)
        
        // Если здесь произойдет исключение, транзакция откатится,
        // и сообщение не будет отправлено в Kafka
    }
}
```

### Паттерн Outbox для согласованности данных

```java
// Java
@Service
public class OutboxPaymentService {
    
    private final PaymentRepository paymentRepository;
    private final OutboxEventRepository outboxRepository;
    private final ObjectMapper objectMapper;
    
    // Шаг 1: Сохранение в БД и создание записи в таблице outbox
    @Transactional
    public void processPayment(Payment payment) {
        // Сохраняем платеж
        Payment savedPayment = paymentRepository.save(payment);
        
        // Создаем событие и сохраняем в outbox таблицу
        OutboxEvent outboxEvent = new OutboxEvent();
        outboxEvent.setAggregateType("Payment");
        outboxEvent.setAggregateId(savedPayment.getId().toString());
        outboxEvent.setEventType("PaymentCreated");
        
        try {
            PaymentEvent paymentEvent = new PaymentEvent(
                    savedPayment.getId(), 
                    savedPayment.getStatus()
            );
            outboxEvent.setPayload(objectMapper.writeValueAsString(paymentEvent));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event", e);
        }
        
        outboxRepository.save(outboxEvent);
    }
}

// Шаг 2: Отдельный сервис для чтения из outbox и отправки в Kafka
@Service
public class OutboxEventPublisher {
    
    private final OutboxEventRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedRate = 5000) // Запуск каждые 5 секунд
    @Transactional
    public void publishOutboxEvents() {
        List<OutboxEvent> events = outboxRepository.findByPublishedFalse();
        
        for (OutboxEvent event : events) {
            String topic = event.getAggregateType().toLowerCase() + "-events";
            
            try {
                // Отправляем в Kafka
                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload())
                    .get(10, TimeUnit.SECONDS); // Ждем подтверждения с таймаутом
                
                // Помечаем как отправленное
                event.setPublished(true);
                event.setPublishedAt(LocalDateTime.now());
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish event: {}", event.getId(), e);
                // Можно добавить счетчик попыток и логику обработки ошибок
            }
        }
    }
}
```

```kotlin
// Kotlin
@Service
class OutboxPaymentService(
    private val paymentRepository: PaymentRepository,
    private val outboxRepository: OutboxEventRepository,
    private val objectMapper: ObjectMapper
) {
    
    // Шаг 1: Сохранение в БД и создание записи в таблице outbox
    @Transactional
    fun processPayment(payment: Payment) {
        // Сохраняем платеж
        val savedPayment = paymentRepository.save(payment)
        
        // Создаем событие и сохраняем в outbox таблицу
        val outboxEvent = OutboxEvent().apply {
            aggregateType = "Payment"
            aggregateId = savedPayment.id.toString()
            eventType = "PaymentCreated"
            
            try {
                val paymentEvent = PaymentEvent(
                    savedPayment.id,
                    savedPayment.status
                )
                payload = objectMapper.writeValueAsString(paymentEvent)
            } catch (e: JsonProcessingException) {
                throw RuntimeException("Failed to serialize event", e)
            }
        }
        
        outboxRepository.save(outboxEvent)
    }
}

// Шаг 2: Отдельный сервис для чтения из outbox и отправки в Kafka
@Service
class OutboxEventPublisher(
    private val outboxRepository: OutboxEventRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>
) {
    
    @Scheduled(fixedRate = 5000) // Запуск каждые 5 секунд
    @Transactional
    fun publishOutboxEvents() {
        val events = outboxRepository.findByPublishedFalse()
        
        for (event in events) {
            val topic = "${event.aggregateType.toLowerCase()}-events"
            
            try {
                // Отправляем в Kafka
                kafkaTemplate.send(topic, event.aggregateId, event.payload)
                    .get(10, TimeUnit.SECONDS) // Ждем подтверждения с таймаутом
                
                // Помечаем как отправленное
                event.apply {
                    published = true
                    publishedAt = LocalDateTime.now()
                }
                outboxRepository.save(event)
            } catch (e: Exception) {
                log.error("Failed to publish event: {}", event.id, e)
                // Можно добавить счетчик попыток и логику обработки ошибок
            }
        }
    }
}
```

## Мониторинг и диагностика

### Метрики Kafka и их интерпретация

#### Ключевые метрики Producer
- **record-send-rate**: скорость отправки сообщений
- **request-latency-avg**: среднее время ожидания ответа от брокера
- **record-error-rate**: частота ошибок отправки

#### Ключевые метрики Consumer
- **records-consumed-rate**: скорость потребления сообщений
- **records-lag**: отставание потребителя (разница между последним опубликованным и потребленным сообщением)
- **bytes-consumed-rate**: скорость потребления данных в байтах

#### Ключевые метрики Broker
- **under-replicated-partitions**: количество партиций с репликацией ниже настроенного фактора
- **request-handler-avg-idle-percent**: загруженность обработчиков запросов
- **partition-count**: количество партиций

### Интеграция с Prometheus и Grafana

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

### Логирование для выявления проблем

```java
// Java
@Service
public class KafkaLoggingAspect {
    
    @Around("execution(* org.springframework.kafka.core.KafkaTemplate.send(..))")
    public Object logKafkaProducerCalls(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        String topic = args[0].toString();
        Object key = args.length > 1 ? args[1] : null;
        Object payload = args.length > 2 ? args[2] : args[1];
        
        log.debug("Sending message to topic={}, key={}, payload={}", topic, key, payload);
        
        long startTime = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            long endTime = System.currentTimeMillis();
            log.debug("Message sent to topic={} in {}ms", topic, (endTime - startTime));
            return result;
        } catch (Exception e) {
            log.error("Error sending message to topic={}, error={}", topic, e.getMessage(), e);
            throw e;
        }
    }
}
```

```kotlin
// Kotlin
@Service
class KafkaLoggingAspect {
    
    @Around("execution(* org.springframework.kafka.core.KafkaTemplate.send(..))")
    fun logKafkaProducerCalls(joinPoint: ProceedingJoinPoint): Any? {
        val args = joinPoint.args
        val topic = args[0].toString()
        val key = if (args.size > 1) args[1] else null
        val payload = if (args.size > 2) args[2] else args[1]
        
        log.debug("Sending message to topic={}, key={}, payload={}", topic, key, payload)
        
        val startTime = System.currentTimeMillis()
        return try {
            val result = joinPoint.proceed()
            val endTime = System.currentTimeMillis()
            log.debug("Message sent to topic={} in {}ms", topic, (endTime - startTime))
            result
        } catch (e: Exception) {
            log.error("Error sending message to topic={}, error={}", topic, e.message, e)
            throw e
        }
    }
}
```

### Диагностика потери данных между микросервисами

#### Шаг 1: Проверка настроек Producer
- Убедитесь, что `acks=all` (ожидание подтверждения от всех реплик)
- Проверьте настройки `retries` (достаточное количество повторов)
- Проверьте `enable.idempotence=true` для предотвращения дубликатов

#### Шаг 2: Проверка настроек Consumer
- Убедитесь, что `enable.auto.commit=false` (ручное подтверждение)
- Проверьте политику `auto.offset.reset` (обычно `earliest` безопаснее)
- Проверьте обработку исключений в Consumer (не теряются ли сообщения при ошибках)

#### Шаг 3: Настройка Consumer Lag мониторинга
```java
// Интеграция с lag мониторингом
@Bean
public KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry() {
    return new KafkaListenerEndpointRegistry();
}

@Bean
public ApplicationListener<ApplicationReadyEvent> lagMonitor(
        KafkaListenerEndpointRegistry registry,
        KafkaConsumerEndpointCustomizer customizer) {
    return event -> {
        MeterRegistry meterRegistry = new SimpleMeterRegistry();
        Map<String, Object> consumerConfigs = new HashMap<>();
        
        KafkaConsumerMetrics kafkaConsumerMetrics = new KafkaConsumerMetrics(
                registry, customizer, meterRegistry, consumerConfigs);
        
        kafkaConsumerMetrics.bindTo(meterRegistry);
    };
}
```

#### Шаг 4: Трассировка сообщений
```java
// Java
@Service
public class MessageTracer {
    
    private final MessageTraceRepository traceRepository;
    
    // На стороне Producer
    public <T> void traceOutgoingMessage(String topic, String key, T message) {
        String messageId = UUID.randomUUID().toString();
        String payload = null;
        try {
            payload = objectMapper.writeValueAsString(message);
        } catch (Exception e) {
            log.error("Failed to serialize message", e);
        }
        
        MessageTrace trace = new MessageTrace();
        trace.setMessageId(messageId);
        trace.setTopic(topic);
        trace.setKey(key);
        trace.setPayload(payload);
        trace.setDirection("OUT");
        trace.setSentTimestamp(LocalDateTime.now());
        
        traceRepository.save(trace);
        
        // Добавляем ID сообщения в заголовки для трассировки
        kafkaTemplate.send(topic, null, key, message, message -> {
            message.headers().add("X-Message-ID", messageId.getBytes());
            return message;
        });
    }
    
    // На стороне Consumer
    public <T> void traceIncomingMessage(
            String topic, 
            String key, 
            T message, 
            @Header("X-Message-ID") byte[] messageIdBytes) {
        
        String messageId = new String(messageIdBytes, StandardCharsets.UTF_8);
        String payload = null;
        try {
            payload = objectMapper.writeValueAsString(message);
        } catch (Exception e) {
            log.error("Failed to serialize message", e);
        }
        
        MessageTrace trace = new MessageTrace();
        trace.setMessageId(messageId);
        trace.setTopic(topic);
        trace.setKey(key);
        trace.setPayload(payload);
        trace.setDirection("IN");
        trace.setReceivedTimestamp(LocalDateTime.now());
        
        traceRepository.save(trace);
    }
}
```

## Тестирование с Kafka

### Тестирование Producer с Mock

```java
// Java
@ExtendWith(MockitoExtension.class)
class PaymentEventProducerTest {
    
    @Mock
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    @InjectMocks
    private PaymentEventProducer producer;
    
    @Test
    void shouldSendPaymentEvent() {
        // Given
        PaymentEvent event = new PaymentEvent("123", PaymentStatus.COMPLETED);
        ProducerRecord<String, PaymentEvent> record = 
                new ProducerRecord<>("payment-events", event.getPaymentId(), event);
        
        CompletableFuture<SendResult<String, PaymentEvent>> future = 
                CompletableFuture.completedFuture(new SendResult<>(record, null));
                
        when(kafkaTemplate.send(anyString(), anyString(), any(PaymentEvent.class)))
                .thenReturn(future);
                
        // When
        CompletableFuture<SendResult<String, PaymentEvent>> resultFuture = 
                producer.sendPaymentEvent(event);
                
        // Then
        assertNotNull(resultFuture);
        verify(kafkaTemplate).send("payment-events", "123", event);
    }
}
```

```kotlin
// Kotlin
@ExtendWith(MockitoExtension::class)
class PaymentEventProducerTest {
    
    @Mock
    private lateinit var kafkaTemplate: KafkaTemplate<String, PaymentEvent>
    
    @InjectMocks
    private lateinit var producer: PaymentEventProducer
    
    @Test
    fun shouldSendPaymentEvent() {
        // Given
        val event = PaymentEvent("123", PaymentStatus.COMPLETED)
        val record = ProducerRecord("payment-events", event.paymentId, event)
        
        val future = CompletableFuture.completedFuture(SendResult(record, null))
                
        whenever(kafkaTemplate.send(anyString(), anyString(), any(PaymentEvent::class.java)))
                .thenReturn(future)
                
        // When
        val resultFuture = producer.sendPaymentEvent(event)
                
        // Then
        assertNotNull(resultFuture)
        verify(kafkaTemplate).send("payment-events", "123", event)
    }
}
```

### Тестирование Consumer с Embedded Kafka

```java
// Java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"payment-events"})
class PaymentEventConsumerTest {
    
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    @Autowired
    private PaymentService paymentService;
    
    @SpyBean
    private PaymentEventConsumer consumer;
    
    @Test
    void shouldConsumePaymentEvent() throws Exception {
        // Given
        PaymentEvent event = new PaymentEvent("123", PaymentStatus.COMPLETED);
        
        // When
        kafkaTemplate.send("payment-events", event.getPaymentId(), event).get();
        
        // Then
        verify(paymentService, timeout(5000)).processPayment(eq(event));
    }
}
```

```kotlin
// Kotlin
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = ["payment-events"])
class PaymentEventConsumerTest {
    
    @Autowired
    private lateinit var kafkaTemplate: KafkaTemplate<String, PaymentEvent>
    
    @Autowired
    private lateinit var paymentService: PaymentService
    
    @SpyBean
    private lateinit var consumer: PaymentEventConsumer
    
    @Test
    fun shouldConsumePaymentEvent() {
        // Given
        val event = PaymentEvent("123", PaymentStatus.COMPLETED)
        
        // When
        kafkaTemplate.send("payment-events", event.paymentId, event).get()
        
        // Then
        verify(paymentService, timeout(5000)).processPayment(eq(event))
    }
}
```

### Тестирование с Testcontainers

```java
// Java
@SpringBootTest
@Testcontainers
class KafkaIntegrationTest {
    
    @Container
    static KafkaContainer kafkaContainer = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:latest"));
    
    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafkaContainer::getBootstrapServers);
    }
    
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    @Autowired
    private PaymentService paymentService;
    
    @Test
    void shouldProduceAndConsumeMessage() throws Exception {
        // Given
        PaymentEvent event = new PaymentEvent("123", PaymentStatus.COMPLETED);
        
        // When
        kafkaTemplate.send("payment-events", event.getPaymentId(), event).get(10, TimeUnit.SECONDS);
        
        // Then
        await().atMost(10, TimeUnit.SECONDS).untilAsserted(() -> {
            verify(paymentService).processPayment(eq(event));
        });
    }
}
```

```kotlin
// Kotlin
@SpringBootTest
@Testcontainers
class KafkaIntegrationTest {
    
    companion object {
        @Container
        @JvmStatic
        val kafkaContainer: KafkaContainer = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:latest"))
        
        @JvmStatic
        @DynamicPropertySource
        fun kafkaProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.kafka.bootstrap-servers", kafkaContainer::getBootstrapServers)
        }
    }
    
    @Autowired
    private lateinit var kafkaTemplate: KafkaTemplate<String, PaymentEvent>
    
    @Autowired
    private lateinit var paymentService: PaymentService
    
    @Test
    fun shouldProduceAndConsumeMessage() {
        // Given
        val event = PaymentEvent("123", PaymentStatus.COMPLETED)
        
        // When
        kafkaTemplate.send("payment-events", event.paymentId, event).get(10, TimeUnit.SECONDS)
        
        // Then
        await().atMost(10, TimeUnit.SECONDS).untilAsserted {
            verify(paymentService).processPayment(eq(event))
        }
    }
}
```

## Ответы на вопросы с собеседований

### 1. Есть ли у Kafka гарантии доставки сообщений?

**Ответ**: Да, Kafka предоставляет три уровня гарантий доставки сообщений:

1. **At-most-once** (не более одного раза):
   - Сообщение может быть доставлено или потеряно
   - Настройка Producer: `acks=0`, отключенные повторы
   - Настройка Consumer: автоматический коммит смещений

2. **At-least-once** (как минимум один раз):
   - Сообщение всегда доставляется, но может быть дублировано
   - Настройка Producer: `acks=all`, включенные повторы
   - Настройка Consumer: ручное управление смещениями, коммит после обработки

3. **Exactly-once** (ровно один раз):
   - Сообщение доставляется ровно один раз, без потерь и дубликатов
   - Реализуется через идемпотентность Producer (`enable.idempotence=true`)
   - Для сквозных транзакций (producer-to-consumer) используются транзакционные ID
   - В реальности часто реализуется на уровне приложения (идемпотентность обработки)

### 2. Два микросервиса, между ними Kafka. Данные теряются в каком-то месте. Что делать?

**Ответ**: Последовательный подход к диагностике и решению проблемы:

1. **Проверить конфигурацию Producer:**
   - Настройка `acks`: должна быть `all` для надежности
   - Параметр `retries`: должен быть достаточно большим (3+ или max_int)
   - Настройка `enable.idempotence`: должна быть `true`
   - Проверить логи ошибок отправки

2. **Проверить Consumer:**
   - Настройка `enable.auto.commit`: лучше `false` для ручного контроля
   - Управление смещениями: коммит должен происходить после успешной обработки
   - Проверить обработку исключений: могут ли исключения привести к пропуску сообщений
   - Настройка `auto.offset.reset`: при первом запуске должна быть `earliest`

3. **Мониторинг отставания (lag):**
   - Настроить мониторинг consumer lag для выявления проблем
   - Отслеживать метрики `records-lag` и `records-lag-max`

4. **Внедрить трассировку сообщений:**
   - Добавить уникальные идентификаторы к каждому сообщению
   - Логировать сообщения на входе и выходе каждого сервиса
   - Использовать распределенную трассировку (Sleuth, Jaeger)

5. **Реализовать идемпотентность на уровне приложения:**
   - Хранить ID обработанных сообщений
   - Проверять перед обработкой, было ли сообщение уже обработано

6. **Внедрить паттерн Outbox:**
   - Транзакционно сохранять действия в БД и outbox таблицу
   - Отдельным процессом отправлять сообщения из outbox в Kafka
   - Гарантировать консистентность между БД и сообщениями

### 3. Какие еще гарантии предоставляет Kafka?

**Ответ**: Помимо гарантий доставки, Kafka предоставляет:

1. **Гарантии упорядоченности:**
   - Сообщения в рамках одной партиции обрабатываются в том порядке, в котором они записаны
   - Для сохранения порядка сообщений для определенного ключа нужно использовать один и тот же ключ (сообщения с одинаковым ключом попадут в одну партицию)

2. **Гарантии долговременного хранения:**
   - Сообщения хранятся в течение настроенного периода retention
   - Возможность хранить даже после потребления (log compaction)

3. **Гарантии масштабируемости:**
   - Горизонтальное масштабирование через добавление брокеров и партиций
   - Балансировка нагрузки между потребителями в группе

4. **Гарантии высокой доступности:**
   - Репликация данных между брокерами
   - Automatic leader election при отказе ведущего брокера

5. **Гарантии производительности:**
   - Высокая пропускная способность благодаря последовательному доступу к диску
   - Предсказуемая латентность

### 4. Как транзакционно выполнить действия с БД и Kafka?

**Ответ**: Существует несколько способов обеспечить транзакционность между БД и Kafka:

1. **Внутренние транзакции Kafka (ограниченное применение):**
   - Работает только для операций внутри Kafka
   - Настройка `transactional.id` на Producer
   - Использование `kafkaTemplate.executeInTransaction()`

2. **Spring's @Transactional с Kafka (частичное решение):**
   - Использование `@Transactional` аннотации
   - Работает только если есть один source of truth (обычно БД)
   - Не гарантирует atomic commit между разными системами

3. **Паттерн Outbox (надежное решение):**
   - Сохранение событий в таблицу outbox в рамках транзакции с основными данными
   - Отдельный процесс для чтения из outbox и отправки в Kafka
   - Атомарность в рамках БД, eventual consistency с Kafka
   - Идемпотентная обработка на стороне Consumer

4. **Паттерн Saga (для распределенных транзакций):**
   - Разбиение крупной транзакции на микротранзакции с компенсирующими действиями
   - Использование Kafka как канала коммуникации между шагами

5. **Change Data Capture (CDC):**
   - Отслеживание изменений в БД (например, через Debezium)
   - Автоматическая публикация изменений в Kafka
   - Подходит для репликации данных между сервисами

**Пример реализации Outbox паттерна:**
```java
@Service
@Transactional
public class PaymentService {
    
    private final PaymentRepository paymentRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;
    
    public void processPayment(Payment payment) {
        // 1. Сохраняем основные данные
        Payment savedPayment = paymentRepository.save(payment);
        
        // 2. В той же транзакции создаем запись в outbox
        OutboxEvent outboxEvent = new OutboxEvent();
        outboxEvent.setAggregateType("Payment");
        outboxEvent.setAggregateId(savedPayment.getId().toString());
        outboxEvent.setEventType("PaymentProcessed");
        
        try {
            outboxEvent.setPayload(objectMapper.writeValueAsString(savedPayment));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event", e);
        }
        
        outboxRepository.save(outboxEvent);
        // Транзакция автоматически коммитится или откатывается
    }
}

// Отдельный сервис для отправки сообщений из outbox
@Service
public class OutboxEventPublisher {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedRate = 5000)
    @Transactional
    public void publishEvents() {
        List<OutboxEvent> events = outboxRepository.findByPublishedFalse();
        
        for (OutboxEvent event : events) {
            try {
                String topic = event.getAggregateType().toLowerCase() + "-events";
                kafkaTemplate.send(topic, event.getPayloadAsString()).get();
                
                event.setPublished(true);
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish event: {}", event.getId(), e);
            }
        }
    }
}
```

### 5. При потере сообщения в Kafka, какие причины и какие варианты исправления?

**Ответ**: Причины потери сообщений и варианты решения:

**Причины на стороне Producer:**
1. **Низкий уровень подтверждений (`acks=0` или `acks=1`):**
   - Producer не ждет подтверждения или ждет только от лидера
   - При сбое брокера сообщения могут быть потеряны
   - **Решение:** установить `acks=all`

2. **Недостаточное количество повторов:**
   - При временных сбоях сообщение не отправляется повторно
   - **Решение:** увеличить `retries` (5+ или Integer.MAX_VALUE)

3. **Неправильная обработка исключений:**
   - Исключение при отправке сообщения не обрабатывается
   - **Решение:** добавить обработку исключений и логику повторов

**Причины на стороне Broker:**
1. **Недостаточный фактор репликации:**
   - При отказе брокера данные могут быть потеряны
   - **Решение:** настроить `replication.factor >= 3`

2. **Неверные настройки `min.insync.replicas`:**
   - Слишком низкое значение снижает надежность
   - **Решение:** установить `min.insync.replicas >= 2`

**Причины на стороне Consumer:**
1. **Автоматический коммит смещений:**
   - Смещения коммитятся до фактической обработки сообщения
   - **Решение:** использовать `enable.auto.commit=false` и ручной коммит

2. **Неправильная обработка исключений:**
   - Исключение при обработке приводит к пропуску сообщения
   - **Решение:** правильно реагировать на исключения, не коммитить смещения при ошибках

3. **Rebalance во время обработки:**
   - Перебалансировка может привести к повторной обработке или пропуску
   - **Решение:** настроить `max.poll.interval.ms` и обрабатывать сообщения быстрее

**Варианты исправления:**
1. **Идемпотентность Producer:**
   - Настройка `enable.idempotence=true`
   - Предотвращает дублирование сообщений при повторной отправке

2. **Транзакционный Producer:**
   - Настройка `transactional.id`
   - Обеспечивает атомарность batch-операций

3. **Ручное управление смещениями:**
   - Коммит смещений только после успешной обработки
   - Обработка в рамках транзакции с БД

4. **Dead Letter Queue (DLQ):**
   - Отдельная очередь для проблемных сообщений
   - Позволяет не терять сообщения при ошибках обработки

5. **Мониторинг и алертинг:**
   - Отслеживание consumer lag
   - Быстрое выявление проблем с потерей данных

6. **Трассировка сообщений:**
   - Добавление уникальных идентификаторов
   - Подтверждение получения на всех этапах
