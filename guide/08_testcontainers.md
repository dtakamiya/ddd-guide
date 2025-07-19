# 8.6 Testcontainersによるテスト

Testcontainersは、テスト実行時にDockerコンテナを自動的に起動・管理し、実際の外部サービス（データベース、メッセージングシステムなど）を使用したテストを可能にするライブラリです。これにより、モックではなく実際のサービスを使用した信頼性の高いテストを実行できます。

## Spannerエミュレータの活用

Google Cloud Spannerはローカルで動作するエミュレータを提供しており、Testcontainersライブラリと組み合わせることで、インテグレーションテストの際にSpannerエミュレータのDockerコンテナを簡単に管理できます。

### 依存関係の追加

```groovy
// build.gradle
testImplementation 'org.testcontainers:spanner:1.19.7'
testImplementation 'org.testcontainers:junit-jupiter:1.19.7'
testImplementation 'com.google.cloud:google-cloud-spanner:6.53.0' // Spannerクライアント
```

### テスト用プロファイルの設定

```properties
# src/test/resources/application-test.properties
spring.cloud.gcp.spanner.emulator.enabled=true
```

## Singleton Container Patternの適用

テストスイート全体で単一のコンテナを共有する**Singleton Container Pattern**を適用することで、テストの実行時間を大幅に短縮できます。

### 基底クラスの実装

```groovy
// src/test/groovy/com/example/infrastructure/AbstractIntegrationSpec.groovy
import com.google.cloud.spanner.*
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.SpannerEmulatorContainer
import org.testcontainers.utility.DockerImageName
import spock.lang.Shared
import spock.lang.Specification

import java.nio.file.Files
import java.nio.file.Paths

abstract class AbstractIntegrationSpec extends Specification {

    @Shared
    private static SpannerEmulatorContainer spannerEmulator

    // staticイニシャライザでコンテナを一度だけ起動する
    static {
        spannerEmulator = new SpannerEmulatorContainer(
                DockerImageName.parse("gcr.io/cloud-spanner-emulator/emulator:latest"))
        spannerEmulator.start()
        
        // テスト用のDBとテーブルを作成
        setupDatabase()
    }

    @DynamicPropertySource
    static void spannerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.cloud.gcp.spanner.emulator-host",
                { -> spannerEmulator.getEmulatorGrpcEndpoint() })
    }

    private static void setupDatabase() {
        // Spannerクライアントの設定
        def options = SpannerOptions.newBuilder()
                .setEmulatorHost(spannerEmulator.getEmulatorGrpcEndpoint())
                .build()
        
        def spanner = options.getService()
        def databaseClient = spanner.getDatabaseClient(
                DatabaseId.of("test-project", "test-instance", "test-database"))
        
        // DDLの実行
        def ddl = """
            CREATE TABLE IF NOT EXISTS users (
                id STRING(MAX) NOT NULL,
                first_name STRING(MAX) NOT NULL,
                last_name STRING(MAX) NOT NULL,
                email STRING(MAX) NOT NULL,
                created_at TIMESTAMP NOT NULL,
                updated_at TIMESTAMP NOT NULL,
                deleted_at TIMESTAMP
            ) PRIMARY KEY (id)
        """
        
        databaseClient.write(Arrays.asList(
                Mutation.of(com.google.cloud.spanner.Mutation.newDdlBuilder()
                        .setOperation(ddl)
                        .build())))
        
        spanner.close()
    }
}
```

### テスト実装

```groovy
// src/test/groovy/com/example/infrastructure/SpannerUserRepositoryIntegrationSpec.groovy
@SpringBootTest
@ActiveProfiles("test")
class SpannerUserRepositoryIntegrationSpec extends AbstractIntegrationSpec {
    
    @Autowired
    private SpannerUserRepository userRepository
    
    def "should save and find user by id"() {
        given: "ユーザーを作成"
        def fullName = new FullName("Test", "User")
        def email = new Email("test.user@example.com")
        def user = User.create(fullName, email)

        when: "ユーザーを保存して検索"
        userRepository.save(user)
        def found = userRepository.findById(user.getId())

        then: "ユーザーが正しく保存・検索される"
        found.isPresent()
        found.get().getId() == user.getId()
        found.get().getFullName().getFullName() == "Test User"
        found.get().getEmail().getValue() == "test.user@example.com"
    }
    
    def "should handle multiple users"() {
        given: "複数のユーザーを作成"
        def users = [
            User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com")),
            User.create(new FullName("Jiro", "Sato"), new Email("jiro@example.com")),
            User.create(new FullName("Saburo", "Tanaka"), new Email("saburo@example.com"))
        ]
        
        when: "全てのユーザーを保存"
        users.each { userRepository.save(it) }
        
        and: "全ユーザーを検索"
        def allUsers = userRepository.findAll()
        
        then: "全てのユーザーが保存される"
        allUsers.size() == 3
        allUsers.every { user ->
            users.any { original -> original.getId() == user.getId() }
        }
    }
    
    def "should handle user updates"() {
        given: "ユーザーを作成・保存"
        def user = User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com"))
        userRepository.save(user)
        
        when: "ユーザー情報を更新"
        user.changeName(new FullName("Jiro", "Sato"))
        userRepository.save(user)
        
        and: "更新されたユーザーを検索"
        def updatedUser = userRepository.findById(user.getId())
        
        then: "ユーザー情報が更新される"
        updatedUser.isPresent()
        updatedUser.get().getFullName().getFullName() == "Jiro Sato"
        updatedUser.get().getUpdatedAt() > updatedUser.get().getCreatedAt()
    }
    
    def "should handle soft delete"() {
        given: "ユーザーを作成・保存"
        def user = User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com"))
        userRepository.save(user)
        
        when: "ユーザーを削除"
        user.delete()
        userRepository.save(user)
        
        and: "削除されたユーザーを検索"
        def deletedUser = userRepository.findById(user.getId())
        
        then: "ユーザーが削除状態になる"
        deletedUser.isPresent()
        deletedUser.get().isDeleted()
        deletedUser.get().getDeletedAt() != null
    }
}
```

## RabbitMQを含むテスト

### RabbitMQコンテナの設定

```groovy
// src/test/groovy/com/example/infrastructure/AbstractMessagingIntegrationSpec.groovy
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.RabbitMQContainer
import org.testcontainers.utility.DockerImageName
import spock.lang.Shared
import spock.lang.Specification

abstract class AbstractMessagingIntegrationSpec extends Specification {

    @Shared
    private static RabbitMQContainer rabbitMQ

    static {
        rabbitMQ = new RabbitMQContainer(DockerImageName.parse("rabbitmq:3-management"))
        rabbitMQ.start()
    }

    @DynamicPropertySource
    static void rabbitMQProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.rabbitmq.host", { -> rabbitMQ.getHost() })
        registry.add("spring.rabbitmq.port", { -> rabbitMQ.getAmqpPort() })
        registry.add("spring.rabbitmq.username", { -> rabbitMQ.getAdminUsername() })
        registry.add("spring.rabbitmq.password", { -> rabbitMQ.getAdminPassword() })
    }
}
```

### メッセージングテストの実装

```groovy
// src/test/groovy/com/example/infrastructure/UserMessagingIntegrationSpec.groovy
@SpringBootTest
@ActiveProfiles("test")
class UserMessagingIntegrationSpec extends AbstractMessagingIntegrationSpec {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    RabbitTemplate rabbitTemplate
    
    @Autowired
    SpannerUserRepository userRepository
    
    def cleanup() {
        userRepository.deleteAll()
        // メッセージキューのクリーンアップ
        rabbitTemplate.execute { channel ->
            channel.queuePurge("user.events")
        }
    }
    
    def "should publish user created event when user is created"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        
        when: "ユーザーを作成"
        def user = userApplicationService.createUser(command)
        
        then: "ユーザー作成イベントが発行される"
        def message = rabbitTemplate.receiveAndConvert("user.events", 5000)
        message != null
        message.userId == user.getId()
        message.email == "taro.yamada@example.com"
        message.eventType == "USER_CREATED"
    }
    
    def "should handle multiple user creation events"() {
        given: "複数のユーザー作成コマンド"
        def commands = [
            new CreateUserCommand("Taro", "Yamada", "taro@example.com"),
            new CreateUserCommand("Jiro", "Sato", "jiro@example.com"),
            new CreateUserCommand("Saburo", "Tanaka", "saburo@example.com")
        ]
        
        when: "複数のユーザーを作成"
        def users = commands.collect { userApplicationService.createUser(it) }
        
        then: "全てのユーザー作成イベントが発行される"
        def messages = []
        3.times {
            def message = rabbitTemplate.receiveAndConvert("user.events", 5000)
            messages << message
        }
        
        messages.size() == 3
        messages.every { message ->
            message.eventType == "USER_CREATED"
            users.any { user -> user.getId() == message.userId }
        }
    }
}
```

## Redisキャッシュを含むテスト

### Redisコンテナの設定

```groovy
// src/test/groovy/com/example/infrastructure/AbstractCacheIntegrationSpec.groovy
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.GenericContainer
import org.testcontainers.utility.DockerImageName
import spock.lang.Shared
import spock.lang.Specification

abstract class AbstractCacheIntegrationSpec extends Specification {

    @Shared
    private static GenericContainer redis

    static {
        redis = new GenericContainer(DockerImageName.parse("redis:7-alpine"))
        redis.withExposedPorts(6379)
        redis.start()
    }

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", { -> redis.getHost() })
        registry.add("spring.redis.port", { -> redis.getMappedPort(6379) })
    }
}
```

### キャッシュテストの実装

```groovy
// src/test/groovy/com/example/infrastructure/UserCacheIntegrationSpec.groovy
@SpringBootTest
@ActiveProfiles("test")
class UserCacheIntegrationSpec extends AbstractCacheIntegrationSpec {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    CacheManager cacheManager
    
    def cleanup() {
        // キャッシュのクリーンアップ
        cacheManager.getCacheNames().each { cacheName ->
            cacheManager.getCache(cacheName).clear()
        }
    }
    
    def "should cache user data"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        
        when: "ユーザーを作成"
        def user = userApplicationService.createUser(command)
        
        and: "同じユーザーを複数回検索"
        def user1 = userApplicationService.findUser(user.getId())
        def user2 = userApplicationService.findUser(user.getId())
        def user3 = userApplicationService.findUser(user.getId())
        
        then: "全ての検索結果が同じ"
        user1.isPresent()
        user2.isPresent()
        user3.isPresent()
        user1.get().getId() == user2.get().getId()
        user2.get().getId() == user3.get().getId()
        
        and: "キャッシュが使用される"
        def cache = cacheManager.getCache("users")
        cache != null
    }
    
    def "should evict cache on user update"() {
        given: "ユーザーを作成"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        def user = userApplicationService.createUser(command)
        
        and: "キャッシュをウォームアップ"
        userApplicationService.findUser(user.getId())
        
        when: "ユーザー情報を更新"
        def updateCommand = new UpdateUserCommand(user.getId(), "Jiro", "Sato")
        userApplicationService.updateUser(updateCommand)
        
        and: "更新されたユーザーを検索"
        def updatedUser = userApplicationService.findUser(user.getId())
        
        then: "更新された情報が返される"
        updatedUser.isPresent()
        updatedUser.get().getFullName() == "Jiro Sato"
    }
}
```

## 複合インテグレーションテスト

### 複数のコンテナを使用したテスト

```groovy
// src/test/groovy/com/example/infrastructure/CompleteIntegrationSpec.groovy
@SpringBootTest
@ActiveProfiles("test")
class CompleteIntegrationSpec extends AbstractIntegrationSpec {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    RabbitTemplate rabbitTemplate
    
    @Autowired
    CacheManager cacheManager
    
    def cleanup() {
        // 各コンポーネントのクリーンアップ
        // データベース、メッセージキュー、キャッシュ
    }
    
    def "complete user lifecycle with all components"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        
        when: "ユーザーを作成"
        def user = userApplicationService.createUser(command)
        
        then: "ユーザーがデータベースに保存される"
        def savedUser = userApplicationService.findUser(user.getId())
        savedUser.isPresent()
        savedUser.get().getFullName() == "Taro Yamada"
        
        and: "イベントがメッセージキューに発行される"
        def message = rabbitTemplate.receiveAndConvert("user.events", 5000)
        message != null
        message.userId == user.getId()
        
        and: "ユーザー情報がキャッシュされる"
        def cachedUser = userApplicationService.findUser(user.getId())
        cachedUser.isPresent()
        
        when: "ユーザー情報を更新"
        def updateCommand = new UpdateUserCommand(user.getId(), "Jiro", "Sato")
        userApplicationService.updateUser(updateCommand)
        
        then: "更新された情報が反映される"
        def updatedUser = userApplicationService.findUser(user.getId())
        updatedUser.isPresent()
        updatedUser.get().getFullName() == "Jiro Sato"
        
        and: "更新イベントが発行される"
        def updateMessage = rabbitTemplate.receiveAndConvert("user.events", 5000)
        updateMessage != null
        updateMessage.eventType == "USER_UPDATED"
    }
}
```

## 学習のポイント

1. **実際のサービスでのテスト**: モックではなく実際の外部サービスを使用
2. **効率的なコンテナ管理**: Singleton Container Patternでテスト実行時間を短縮
3. **複合コンポーネントの検証**: データベース、メッセージング、キャッシュの連携をテスト
4. **信頼性の高いテスト**: 本番環境に近い条件での動作確認

次のセクションでは、テストデータ管理について詳しく解説します。 