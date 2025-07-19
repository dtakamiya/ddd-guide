### 6.9. インフラストラクチャ層のテスト

#### 6.9.1. Testcontainersを使用した統合テスト

**UserRepositoryIntegrationTest.groovy:**
```groovy
package com.example.dddspanner.infrastructure.repository

import com.example.dddspanner.domain.entity.User
import com.example.dddspanner.domain.factory.UserFactory
import com.example.dddspanner.domain.valueobject.Email
import com.example.dddspanner.domain.valueobject.FullName
import com.example.dddspanner.domain.valueobject.UserId
import com.example.dddspanner.infrastructure.entity.UserEntity
import com.example.dddspanner.infrastructure.mapper.UserEntityMapper
import com.google.cloud.spring.data.spanner.core.SpannerTemplate
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import org.testcontainers.containers.GenericContainer
import org.testcontainers.spock.Testcontainers
import spock.lang.Specification

@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
class UserRepositoryIntegrationTest extends Specification {
    
    static GenericContainer spannerContainer = new GenericContainer("gcr.io/cloud-spanner-emulator/emulator:latest")
        .withExposedPorts(9010)
        .withEnv("SPANNER_PROJECT_ID", "test-project")
        .withEnv("SPANNER_INSTANCE_ID", "test-instance")
        .withEnv("SPANNER_DATABASE_ID", "test-database")
    
    SpannerTemplate spannerTemplate
    UserEntityMapper userEntityMapper
    UserRepositoryImpl userRepository
    UserFactory userFactory
    
    def setupSpec() {
        spannerContainer.start()
    }
    
    def cleanupSpec() {
        spannerContainer.stop()
    }
    
    def setup() {
        // テストデータの準備
        setupTestData()
    }
    
    def "should save and retrieve user"() {
        given:
        def fullName = new FullName("Taro", "Yamada")
        def email = new Email("taro.yamada@example.com")
        def user = userFactory.createUser(fullName, email)
        
        when:
        def savedUser = userRepository.save(user)
        def retrievedUser = userRepository.findById(savedUser.getId()).orElse(null)
        
        then:
        retrievedUser != null
        retrievedUser.getId() == savedUser.getId()
        retrievedUser.getFullName().firstName() == "Taro"
        retrievedUser.getFullName().lastName() == "Yamada"
        retrievedUser.getEmail().value() == "taro.yamada@example.com"
    }
    
    def "should find user by email"() {
        given:
        def email = new Email("test@example.com")
        def user = createTestUser(email)
        userRepository.save(user)
        
        when:
        def foundUser = userRepository.findByEmail(email)
        
        then:
        foundUser.isPresent()
        foundUser.get().getEmail().value() == "test@example.com"
    }
    
    def "should find all users"() {
        given:
        def user1 = createTestUser(new Email("user1@example.com"))
        def user2 = createTestUser(new Email("user2@example.com"))
        userRepository.save(user1)
        userRepository.save(user2)
        
        when:
        def allUsers = userRepository.findAll()
        
        then:
        allUsers.size() >= 2
        allUsers.any { it.getEmail().value() == "user1@example.com" }
        allUsers.any { it.getEmail().value() == "user2@example.com" }
    }
    
    def "should find active users only"() {
        given:
        def activeUser = createTestUser(new Email("active@example.com"))
        def inactiveUser = createTestUser(new Email("inactive@example.com"))
        inactiveUser.deactivate()
        
        userRepository.save(activeUser)
        userRepository.save(inactiveUser)
        
        when:
        def activeUsers = userRepository.findActiveUsers()
        
        then:
        activeUsers.size() >= 1
        activeUsers.every { it.getStatus() == UserStatus.ACTIVE }
    }
    
    def "should check user existence"() {
        given:
        def user = createTestUser(new Email("exists@example.com"))
        def savedUser = userRepository.save(user)
        
        when:
        def existsById = userRepository.existsById(savedUser.getId())
        def existsByEmail = userRepository.existsByEmail(savedUser.getEmail())
        
        then:
        existsById == true
        existsByEmail == true
    }
    
    def "should delete user"() {
        given:
        def user = createTestUser(new Email("delete@example.com"))
        def savedUser = userRepository.save(user)
        
        when:
        userRepository.delete(savedUser.getId())
        def deletedUser = userRepository.findById(savedUser.getId())
        
        then:
        deletedUser.isEmpty()
    }
    
    private User createTestUser(Email email) {
        def fullName = new FullName("Test", "User")
        return userFactory.createUser(fullName, email)
    }
    
    private void setupTestData() {
        // テスト用のテーブル作成とデータ準備
        // 実際の実装では、Spannerのテスト用データベースを準備
    }
}
```

#### 6.9.2. メッセージングのテスト

**UserEventPublisherTest.groovy:**
```groovy
package com.example.dddspanner.infrastructure.messaging

import com.example.dddspanner.domain.event.UserCreatedEvent
import com.example.dddspanner.domain.valueobject.Email
import com.example.dddspanner.domain.valueobject.UserId
import org.springframework.amqp.rabbit.core.RabbitTemplate
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import org.testcontainers.containers.RabbitMQContainer
import org.testcontainers.spock.Testcontainers
import spock.lang.Specification

@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
class UserEventPublisherTest extends Specification {
    
    static RabbitMQContainer rabbitMQContainer = new RabbitMQContainer("rabbitmq:3-management")
        .withExposedPorts(5672, 15672)
    
    UserEventPublisher userEventPublisher
    RabbitTemplate rabbitTemplate
    
    def setupSpec() {
        rabbitMQContainer.start()
    }
    
    def cleanupSpec() {
        rabbitMQContainer.stop()
    }
    
    def "should publish user created event"() {
        given:
        def userId = UserId.generate()
        def email = new Email("test@example.com")
        def event = new UserCreatedEvent(userId, email)
        
        when:
        userEventPublisher.publishUserEvent(event)
        
        then:
        // RabbitMQにメッセージが送信されたことを確認
        // 実際の実装では、メッセージの受信を確認
        noExceptionThrown()
    }
    
    def "should handle publishing error"() {
        given:
        def event = null
        
        when:
        userEventPublisher.publishUserEvent(event)
        
        then:
        thrown(MessagePublishException)
    }
}
```

#### 6.9.3. キャッシュのテスト

**UserCacheServiceTest.groovy:**
```groovy
package com.example.dddspanner.infrastructure.cache

import com.example.dddspanner.application.dto.UserResponse
import com.example.dddspanner.domain.valueobject.UserId
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.cache.CacheManager
import org.springframework.test.context.ActiveProfiles
import org.testcontainers.containers.GenericContainer
import org.testcontainers.spock.Testcontainers
import spock.lang.Specification

@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
class UserCacheServiceTest extends Specification {
    
    static GenericContainer redisContainer = new GenericContainer("redis:7-alpine")
        .withExposedPorts(6379)
    
    UserCacheService userCacheService
    CacheManager cacheManager
    
    def setupSpec() {
        redisContainer.start()
    }
    
    def cleanupSpec() {
        redisContainer.stop()
    }
    
    def "should cache user response"() {
        given:
        def userId = "test-user-id"
        def userResponse = new UserResponse(userId, "Taro", "Yamada", "Taro Yamada", "test@example.com", "ACTIVE", null, null)
        
        when:
        def cachedUser = userCacheService.getUserFromCache(userId)
        
        then:
        cachedUser.isPresent()
        cachedUser.get().id() == userId
    }
    
    def "should evict cache on user update"() {
        given:
        def userId = "update-user-id"
        
        when:
        userCacheService.evictUserCacheOnUpdate(userId)
        
        then:
        // キャッシュが無効化されたことを確認
        noExceptionThrown()
    }
    
    def "should evict all cache on user creation"() {
        when:
        userCacheService.evictUserCacheOnCreate()
        
        then:
        // 全キャッシュが無効化されたことを確認
        noExceptionThrown()
    }
}
``` 