# Spring Boot Starter - Полное руководство

## 1. Spring vs Spring Boot - ключевые отличия

### Spring Framework
- **Что это**: Мощный фреймворк для создания enterprise Java приложений
- **Конфигурация**: Требует много ручной настройки (XML или Java Config)
- **Зависимости**: Нужно самому подбирать версии библиотек
- **Сервер**: Нужно отдельно настраивать Tomcat/Jetty
- **Старт проекта**: Много boilerplate кода

### Spring Boot
- **Что это**: Надстройка над Spring для быстрого старта
- **Конфигурация**: Автоконфигурация из коробки
- **Зависимости**: Starter'ы с подобранными версиями
- **Сервер**: Встроенный сервер (embedded Tomcat)
- **Старт проекта**: Минимум кода для запуска

```java
// Классический Spring (много конфигурации)
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.example")
public class WebConfig extends WebMvcConfigurerAdapter {
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
    // ... еще много бинов
}

// Spring Boot (всё из коробки)
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 2. Что такое Spring Boot Starter

**Spring Boot Starter** - это специальный модуль, который:
- Объединяет набор зависимостей для конкретной задачи
- Предоставляет автоконфигурацию
- Упрощает подключение функциональности

### Примеры стандартных стартеров:
```xml
<!-- Web приложения -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Работа с БД -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Безопасность -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

## 3. Как работает автоконфигурация

### Ключевые компоненты:

1. **@EnableAutoConfiguration** - включает автоконфигурацию
2. **@Conditional** аннотации - условия создания бинов
3. **spring.factories** - регистрация конфигураций
4. **application.properties** - настройки

### Процесс автоконфигурации:
```
1. Spring Boot сканирует classpath
2. Находит файлы META-INF/spring.factories
3. Загружает классы автоконфигурации
4. Проверяет условия (@Conditional)
5. Создает бины если условия выполнены
```

## 4. Структура кастомного стартера

### Naming Convention:
- Официальные: `spring-boot-starter-{name}`
- Кастомные: `{name}-spring-boot-starter`

### Модули стартера:
```
my-awesome-spring-boot-starter/
├── my-awesome-spring-boot-autoconfigure/  # Автоконфигурация
│   ├── src/main/java/
│   │   └── com/example/autoconfigure/
│   │       ├── MyAwesomeAutoConfiguration.java
│   │       └── MyAwesomeProperties.java
│   └── src/main/resources/
│       └── META-INF/
│           └── spring.factories
└── my-awesome-spring-boot-starter/        # Сам стартер
    └── pom.xml (зависимости)
```

## 5. Создание своего стартера - пошаговый план

### Шаг 1: Создаем проект autoconfigure
```xml
<groupId>com.example</groupId>
<artifactId>logging-spring-boot-autoconfigure</artifactId>
<version>1.0.0</version>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### Шаг 2: Создаем Properties класс
```java
@ConfigurationProperties(prefix = "my.logging")
public class LoggingProperties {
    private boolean enabled = true;
    private String level = "INFO";
    private String pattern = "[%level] %msg";
    
    // getters/setters
}
```

### Шаг 3: Создаем сервис
```java
public class CustomLogger {
    private final LoggingProperties properties;
    
    public CustomLogger(LoggingProperties properties) {
        this.properties = properties;
    }
    
    public void log(String message) {
        if (properties.isEnabled()) {
            String formatted = properties.getPattern()
                .replace("%level", properties.getLevel())
                .replace("%msg", message);
            System.out.println(formatted);
        }
    }
}
```

### Шаг 4: Создаем автоконфигурацию
```java
@Configuration
@ConditionalOnClass(CustomLogger.class)
@EnableConfigurationProperties(LoggingProperties.class)
public class LoggingAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(
        prefix = "my.logging", 
        name = "enabled", 
        havingValue = "true", 
        matchIfMissing = true
    )
    public CustomLogger customLogger(LoggingProperties properties) {
        return new CustomLogger(properties);
    }
}
```

### Шаг 5: Регистрируем в spring.factories
```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfigure.LoggingAutoConfiguration
```

### Шаг 6: Создаем стартер
```xml
<groupId>com.example</groupId>
<artifactId>logging-spring-boot-starter</artifactId>
<version>1.0.0</version>

<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>logging-spring-boot-autoconfigure</artifactId>
        <version>1.0.0</version>
    </dependency>
    <!-- Другие необходимые зависимости -->
</dependencies>
```

## 6. Публикация в Maven Central

### Требования:
1. **GroupId**: Нужен домен (com.example)
2. **GPG ключ**: Для подписи артефактов
3. **Sonatype аккаунт**: JIRA на issues.sonatype.org

### Настройка pom.xml:
```xml
<distributionManagement>
    <snapshotRepository>
        <id>ossrh</id>
        <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    <repository>
        <id>ossrh</id>
        <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
</distributionManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <executions>
                <execution>
                    <id>sign-artifacts</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>sign</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Процесс публикации:
```bash
# Деплой в staging
mvn clean deploy

# Релиз из staging в Central
mvn nexus-staging:release
```

## 7. Использование стартера

### В pom.xml проекта:
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>logging-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### В application.properties:
```properties
my.logging.enabled=true
my.logging.level=DEBUG
my.logging.pattern=[%level] %msg - %timestamp
```

### В коде:
```java
@Component
public class MyService {
    private final CustomLogger logger;
    
    public MyService(CustomLogger logger) {
        this.logger = logger;
    }
    
    public void doSomething() {
        logger.log("Doing something important!");
    }
}
```

## 8. Best Practices

### Do:
- ✅ Используй префиксы для properties
- ✅ Предоставляй дефолтные значения
- ✅ Документируй properties через JavaDoc
- ✅ Используй @Conditional аннотации
- ✅ Тестируй автоконфигурацию
- ✅ Следуй naming convention

### Don't:
- ❌ Не перегружай стартер функциональностью
- ❌ Не создавай обязательные зависимости
- ❌ Не забывай про backward compatibility
- ❌ Не игнорируй версионирование

## 9. Тестирование стартера

```java
@SpringBootTest(classes = TestApplication.class)
@TestPropertySource(properties = {
    "my.logging.enabled=true",
    "my.logging.level=ERROR"
})
class LoggingAutoConfigurationTest {
    
    @Autowired
    private ApplicationContext context;
    
    @Test
    void contextLoads() {
        assertThat(context.getBean(CustomLogger.class)).isNotNull();
    }
    
    @Test
    void propertiesAreApplied() {
        LoggingProperties props = context.getBean(LoggingProperties.class);
        assertThat(props.getLevel()).isEqualTo("ERROR");
    }
}
```

## 10. Идеи для практики

### Простой стартер для начала:
**Request Logger Starter** - логирует все HTTP запросы
- Перехватывает входящие запросы
- Логирует метод, URL, время выполнения
- Настраиваемый формат логов
- Фильтрация по URL паттернам

### Компоненты:
1. `RequestLoggingFilter` - фильтр для перехвата
2. `RequestLoggingProperties` - настройки
3. `RequestLoggingAutoConfiguration` - автоконфигурация

### Использование:
```properties
request.logging.enabled=true
request.logging.include-headers=true
request.logging.exclude-patterns=/health,/metrics
```

## Полезные ресурсы

1. [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
2. [Creating Your Own Starter](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)
3. [Maven Central Publishing Guide](https://central.sonatype.org/publish/publish-guide/)
4. [Baeldung - Custom Spring Boot Starter](https://www.baeldung.com/spring-boot-custom-starter)

## Следующие шаги

1. **Теория**: Перечитай разделы 1-4 для понимания концепций
2. **Практика**: Начни с простого логгера из раздела 5
3. **Тестирование**: Напиши тесты для автоконфигурации
4. **Публикация**: Попробуй опубликовать в локальный Maven репозиторий
5. **Maven Central**: Когда будешь готов, следуй инструкциям раздела 6