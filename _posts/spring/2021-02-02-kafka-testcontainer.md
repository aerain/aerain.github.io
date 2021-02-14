---
layout: post
title: "Avro Schema을 기반으로 하는 카프카 메시지를 소비하는 Spring Kafka Consumer 통합 테스트 만들기."
category: "spring"
tags: ["Spring", "Kafka", "TestContainer", "Schema Registry", "Avro"]
---

# 개요
현재 재직 중인 회사에서는 Schema Registry를 통해 카프카의 메시지 포맷을 관리하고 있다. 특히나 [CDC](https://www.qlik.com/us/change-data-capture/cdc-change-data-capture)의 도입을 통해 데이터베이스의 변경사항을 카프카로 전송하고 있는 만큼, 메시지 포맷에 대한 통합 테스트 또한 반드시 필요하다. 

이번 포스팅에서는 cdc를 통해 카프카로 전송되는 db의 변경사항을 Schema Registry에서 Avro Schema 로 관리되는 kafka message를 컨슘하는 스프링 애플리케이션의 통합테스트를 진행하는 방법에 대해 알아 보고자 한다.


# 사용 도구
통합테스트에 사용할 카프카는 각자의 취향에 맞춰 `on local`, `docker run`, `docker-compose` 등, 자유롭게 지정해도 무방하다. 본 게시글에서는 코드 내부에서 docker container를 띄워서 사용할 수 있는 [TestContainer](https://www.testcontainers.org/)를 사용하고자 한다.

코드를 개발할 때 사용한 환경은 다음과 같다.
* Docker Engine 19.03.12
* macOS Big Sur
* Gradle

# 전제 조건
* cdc를 통해 Schema Registry 에 저장되는 Schema는 임의로 sometopic 이라는 토픽의 스키마로 설정한다.
* 해당 스키마는 sometopic-key, sometopic-value라는 subject으로 Schema Registry에 등록되어 있다.
* 해당 스키마는 java code generation 을 통해 빌드 시 생성되는 pojo로 관리되고 있다.
    * 이 글에서는 schema registry로부터 스키마를 받는 법등에 대한 설명은 언급하지 않으므로, 일부 코드에 의존성 누락이 발생할 수 있다.
    * 해당 부분에 대해서는 [schema-registry-gradle-plugin](https://github.com/ImFlog/schema-registry-plugin)를 참고하면 된다.

# 카프카 컨슘 코드

Spring에서 카프카를 컨슘하기 위해 다음 라이브러리를 의존성에 추가했다.
또한 Schema Registry의 Avro Schema 를 받기 위해 [schema-registry-gradle-plugin](https://github.com/ImFlog/schema-registry-plugin)을 사용했다.

버전은 이 글을 작성할 때 기준으로 작성하였으니 알아서 참고하면 된다.


build.gradle.kts
```kotlin

dependencies {
    ...
    // kafka 의존성 추가
    implementation("org.springframework.kafka:spring-kafka")
    // Kafka Avro Serializer
    implementation("io.confluent:kafka-avro-serializer:${kafka_avro_serializer_version}")
}

// schema
```


의존성에 추가가 완료되었다면 다음과 같이 KafkaConsumer를 추가해주자. 토픽 명은 임의로 application.properties에 지정하였다.
또한 개발시 사용할 카프카와 Schema Registry의 url을 추가해주었다.
application.properties
```properties
kafka.topic=sometopic

spring.kafka.bootstrap-servers=...
spring.kafka.consumer.group-id=...

# Avro Schema로 관리되는 레코드 역직렬화를 위한 deserializer 정의 
spring.kafka.consumer.key-deserializer=io.confluent.kafka.serializers.KafkaAvroDeserializer
spring.kafka.consumer.value-deserializer=io.confluent.kafka.serializers.KafkaAvroDeserializer

# Schema Registry url 등록
spring.kafka.properties.schema.registry.url=...

```

TestConsumer.kt
```kotlin
@Service
class TestConsumer {

    @KafkaListener(topics = "sometopic")
    fun consume(record: ConsumerRecord<Key, Envelope>) {
        // some code
    }
}
```

# 테스트 코드 작성
테스트 코드 환경은 JUnit5로 작성되었다. 통합테스트 환경 구축을 시도해 보겠다.
일단 통합테스트를 위한 카프카 컨테이너를 띄우기 위해 TestContainer 코드를 작성하겠다.

```kotlin
object KafkaContainerInitializer {
    
    val kafka = KafkaContainer(DockerImageName.parse("confluentic/cpkafka:5.4.3"))
        .apply { start() }
    
    class KafkaInitializer : ApplicationContextInitializer<ConfigurableApplicationContext> {
        override fun initialize(context: ConfigurableApplicationContext) {
            val bootstrapServers = kafka.bootstrapServers
            val values = TestPropertyValues.of(
                "spring.kafka.bootstrap-servers=$bootstrapServers"
            )
            values.applyTo(context)
        }
    }
}
```
해당 코드를 요약 하면 다음과 같다. 
* KafkaContainer를 애플리케이션 실행 시점에 초기화 시켜 통합 테스트에 사용할 카프카를 띄운다.
* 추후 KafkaInitializer 내부에서 kafka의 bootstrapServers를 받아 프로퍼티를 동적으로 주입한다. 

그리고 통합테스트에 사용할 추상 클래스를 정의하여 ContextConfiguration 의 initalizer에 등록한다.

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ContextConfiguration(
    initializers = [
        KafkaContainerInitializer.KafkaInitializer::class // 여기에 정의한 클래스가 스프링 테스트 컨테이너 초기화 시점에 수행된다.
    ]
)
@ActiveProfiles("test")
abstract class IntegrationTest {

}
```

test에서만 사용할 프로퍼티를 따로 정의하기 위해 프로필을 test로 정의하였는데, 이에 따라 application-test.properties를 추가로 정의하였다.

```properties
spring.kafka.properties.schema.registry.url=mock://test
```

여기에서 mock 프로토콜을 사용한게 눈에 띈다. 내부적으로 spring-kakfa에서 해당 카프카 프로퍼티를 등록하면 KafkaAvroDeserializer가 해당 url 로 통신하는 SchemaRegistryClient를 내부적으로 생성한다. 그런데 mock://를 prefix 로하는 url을 등록하면 MockSchemaRegistry가 관리하는 MockSchemaRegistryClient를 생성해 KafkaAvroDeserializer에게 제공해준다.

```java
/**
 * A repository for mocked Schema Registry clients, to aid in testing.
 *
 * <p>Logically independent "instances" of mocked Schema Registry are created or retrieved
 * via named scopes {@link MockSchemaRegistry#getClientForScope(String)}.
 * Each named scope is an independent registry.
 * Each named-scope registry is statically defined and visible to the entire JVM.
 * Scopes can be cleaned up when no longer needed via {@link MockSchemaRegistry#dropScope(String)}.
 * Reusing a scope name after cleanup results in a completely new mocked Schema Registry instance.
 *
 * <p>This registry can be used to manage scoped clients directly, but scopes can also be registered
 * and used as {@code schema.registry.url} with the special pseudo-protocol 'mock://'
 * in serde configurations, so that testing code doesn't have to run an actual instance of
 * Schema Registry listening on a local port. For example,
 * {@code schema.registry.url: 'mock://my-scope-name'} corresponds to
 * {@code MockSchemaRegistry.getClientForScope("my-scope-name")}.
 */
 public final class MockSchemaRegistry {
  private static final String MOCK_URL_PREFIX = "mock://";
  private static final Map<String, SchemaRegistryClient> SCOPED_CLIENTS = new HashMap<>();
  ...
 }
```

이제 Consumer를 테스트하기 위해 KafkaProducer를 생성하기로한다.

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
@ContextConfiguration(
    initializers = [
        DynamoDbContainerIntializer.DynamoDbInitializer::class,
        KafkaContainerInitalizer.KafkaInitializer::class
    ]
)
@ExtendWith(MockKExtension::class)
@ActiveProfiles("test")
abstract class IntegrationTest {

    @Value("\${spring.kafka.consumer.properties.schema.registry.url}")
    private lateinit var schemaRegistryUrl: String

    @Value("\${spring.kafka.bootstrap-servers}")
    private lateinit var kafkaBootstrapServers: String

    protected lateinit var kafkaProducer: KafkaProducer<Any, Any>

    @PostConstruct
    fun initialize() {
        val config = mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to kafkaBootstrapServers,
            ProducerConfig.CLIENT_ID_CONFIG to UUID.randomUUID().toString(),
            AbstractKafkaSchemaSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG to schemaRegistryUrl,
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to KafkaAvroSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to KafkaAvroSerializer::class.java
        )
        kafkaProducer = KafkaProducer(config)
    }

    companion object : KLogging()
}
```


이제 본격적으로 테스트를 위한 코드를 작성하였다.

```kotlin

class ConsumerIntegrationTest : IntegrationTest() {
    @Value("\${kafka.topic}")
    private lateinit var topic: String

    @Test
    fun test() {
        val key: Key = getKey() // 테스트할 Key 인스턴스를 생성한다.
        val envelope: Envelope = getEnvelope() // 테스트할 Envelope 인스턴스를 생성한다.
        val producerRecord: ProducerRecord<Any, Any> = ProducerRecord(topic, key, envelope)
        
        kafkaProducer.send(producerRecord).get()
        delay(1000) // 여기서 consumer가 카프카로부터 레코드를 consume할만큼 충분히 대기하여야 한다.

        // 여기서 consumer에 대한 assert를 수행한다.
        assert()

    }
}
``` 

java code generation 을 사용하지 않는다면 레코드를 만드는 법은 더 복잡하다. AvroSchema를 직접 받아와서 파싱해야한다. [Confluent 도큐먼트](https://docs.confluent.io/platform/current/schema-registry/serdes-develop/serdes-avro.html)에서 더 raw한 내용을 찾을 수 있다.

간단하게 카프카 데이터를 컨슘하는 애플리케이션을 개발하는데, 배보다 배꼽이 큰것 같지만, 역시 모든 코드에는 그에 해당하는 테스트코드는 반드시 필요하다고 생각했기에, 단위 테스트 외에도 직접 환경을 만들어보겠다는 생각에 작성하였다.
