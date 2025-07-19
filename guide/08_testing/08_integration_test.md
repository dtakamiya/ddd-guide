# 8.4 インテグレーションテスト

インテグレーションテストは、複数のコンポーネントを連携させたテストを行います。Spring Boot Testを活用して、APIエンドポイントの動作確認や、データベース永続化を含めた一連のフローをテストします。

## Spring Boot Testの活用

Spring Boot Testは、実際のアプリケーションコンテキストをロードして、複数のコンポーネントを連携させたテストを実行できます。

### 基本的なインテグレーションテスト

```groovy
// src/test/groovy/com/example/presentation/UserRestControllerIntegrationSpec.groovy
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc
import spock.lang.Specification

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf

@SpringBootTest
@AutoConfigureMockMvc
class UserRestControllerIntegrationSpec extends Specification {

    @Autowired
    private MockMvc mockMvc
    
    @Autowired
    private ObjectMapper objectMapper // JSONシリアライズ/デシリアライズ用

    @Autowired
    private SpannerUserRepository userRepository // DBの状態をセットアップ/検証するために使用

    def cleanup() {
        userRepository.deleteAll()
    }

    @WithMockUser(username = "testuser", roles = ["USER"]) // 認証済みユーザーとしてテストを実行
    def "POST /api/v1/users should create a new user and return HATEOAS links"() {
        given: "ユーザー作成リクエスト"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        def requestBody = objectMapper.writeValueAsString(command)

        when: "ユーザー作成APIを呼び出す"
        def result = mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(requestBody)
                .with(csrf())) // CSRFトークンを付与

        then: "ユーザーが正常に作成される"
        result.andExpect(status().isCreated())
              .andExpect(jsonPath('$.fullName', is('Taro Yamada')))
              .andExpect(jsonPath('$.email', is('taro.yamada@example.com')))
              .andExpect(jsonPath('$._links.self.href', containsString('/api/v1/users/')))
    }

    def "GET /api/v1/users/{id} without authentication should return 401 Unauthorized"() {
        when: "認証なしでユーザー取得APIを呼び出す"
        def result = mockMvc.perform(get("/api/v1/users/some-id"))

        then: "401 Unauthorizedが返される"
        result.andExpect(status().isUnauthorized())
    }
}
```

## ログ出力のテスト

ログ出力の内容を確認するテストは、アプリケーションの動作を追跡し、重要なイベントが正しく記録されているかを検証するために重要です。

### Spring Boot Testでのログキャプチャ

```groovy
// src/test/groovy/com/example/presentation/LoggingIntegrationSpec.groovy
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.context.TestPropertySource
import spock.lang.Specification
import ch.qos.logback.classic.Logger
import ch.qos.logback.classic.spi.ILoggingEvent
import ch.qos.logback.core.read.ListAppender
import org.slf4j.LoggerFactory

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status

@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(properties = [
    "logging.level.com.example=DEBUG",
    "logging.level.org.springframework.security=DEBUG"
])
class LoggingIntegrationSpec extends Specification {

    @Autowired
    private MockMvc mockMvc
    
    @Autowired
    private ObjectMapper objectMapper

    private ListAppender<ILoggingEvent> listAppender
    private Logger logger

    def setup() {
        // ログをキャプチャするためのListAppenderを設定
        listAppender = new ListAppender<>()
        listAppender.start()
        
        // アプリケーションロガーにListAppenderを追加
        logger = (Logger) LoggerFactory.getLogger("com.example")
        logger.addAppender(listAppender)
    }

    def cleanup() {
        // テスト後にListAppenderを削除
        if (logger != null) {
            logger.detachAppender(listAppender)
        }
    }

    def "should log user creation event"() {
        given: "ユーザー作成リクエスト"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        def requestBody = objectMapper.writeValueAsString(command)

        when: "ユーザー作成APIを呼び出す"
        mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(requestBody))

        then: "ユーザー作成のログが出力される"
        def logEvents = listAppender.list
        logEvents.any { event ->
            event.message.contains("User created") && 
            event.message.contains("taro.yamada@example.com")
        }
        
        and: "ログレベルがINFO以上である"
        logEvents.any { event ->
            event.level.isGreaterOrEqual(ch.qos.logback.classic.Level.INFO)
        }
    }

    def "should log security events"() {
        when: "認証なしでAPIにアクセス"
        mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content("{}"))

        then: "セキュリティ関連のログが出力される"
        def logEvents = listAppender.list
        logEvents.any { event ->
            event.loggerName.contains("security") &&
            (event.message.contains("Unauthorized") || 
             event.message.contains("Authentication"))
        }
    }

    def "should log database operations"() {
        given: "ユーザー作成リクエスト"
        def command = new CreateUserCommand("Jiro", "Sato", "jiro.sato@example.com")
        def requestBody = objectMapper.writeValueAsString(command)

        when: "ユーザー作成APIを呼び出す"
        mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(requestBody))

        then: "データベース操作のログが出力される"
        def logEvents = listAppender.list
        logEvents.any { event ->
            event.loggerName.contains("repository") &&
            (event.message.contains("save") || 
             event.message.contains("insert"))
        }
    }
}
```

### ログ検証のヘルパーメソッド

```groovy
// src/test/groovy/com/example/test/LoggingTestHelper.groovy
import ch.qos.logback.classic.Logger
import ch.qos.logback.classic.spi.ILoggingEvent
import ch.qos.logback.core.read.ListAppender
import org.slf4j.LoggerFactory
import spock.lang.Specification

class LoggingTestHelper extends Specification {
    
    protected ListAppender<ILoggingEvent> listAppender
    protected Logger logger
    
    def setupLogging(String loggerName = "com.example") {
        listAppender = new ListAppender<>()
        listAppender.start()
        
        logger = (Logger) LoggerFactory.getLogger(loggerName)
        logger.addAppender(listAppender)
    }
    
    def cleanupLogging() {
        if (logger != null) {
            logger.detachAppender(listAppender)
        }
    }
    
    def assertLogContains(String message) {
        assert listAppender.list.any { event ->
            event.message.contains(message)
        }
    }
    
    def assertLogContains(String message, ch.qos.logback.classic.Level level) {
        assert listAppender.list.any { event ->
            event.message.contains(message) && event.level == level
        }
    }
    
    def assertLogCount(int expectedCount) {
        assert listAppender.list.size() == expectedCount
    }
    
    def getLogEvents() {
        return listAppender.list
    }
    
    def getLogMessages() {
        return listAppender.list*.message
    }
}
```

### アプリケーションサービスでのログテスト

```groovy
// src/test/groovy/com/example/application/UserApplicationServiceLoggingSpec.groovy
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.TestPropertySource
import spock.lang.Specification

@SpringBootTest
@TestPropertySource(properties = [
    "logging.level.com.example.application=DEBUG"
])
class UserApplicationServiceLoggingSpec extends LoggingTestHelper {

    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    SpannerUserRepository userRepository
    
    def setup() {
        setupLogging("com.example.application")
    }
    
    def cleanup() {
        cleanupLogging()
        userRepository.deleteAll()
    }
    
    def "should log user creation process"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        
        when: "アプリケーションサービスでユーザーを作成"
        userApplicationService.createUser(command)
        
        then: "ユーザー作成プロセスのログが出力される"
        assertLogContains("Creating user with email: taro.yamada@example.com")
        assertLogContains("User created successfully")
        
        and: "ログレベルが適切である"
        assertLogContains("User created successfully", ch.qos.logback.classic.Level.INFO)
    }
    
    def "should log validation errors"() {
        given: "無効なユーザー作成コマンド"
        def command = new CreateUserCommand("", "", "invalid-email")
        
        when: "無効なコマンドでユーザーを作成しようとする"
        userApplicationService.createUser(command)
        
        then: "バリデーションエラーのログが出力される"
        thrown(ValidationException)
        assertLogContains("Validation failed")
        assertLogContains("invalid-email", ch.qos.logback.classic.Level.WARN)
    }
    
    def "should log business rule violations"() {
        given: "既存のユーザー"
        def existingCommand = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        userApplicationService.createUser(existingCommand)
        
        and: "同じメールアドレスでユーザー作成コマンド"
        def duplicateCommand = new CreateUserCommand("Jiro", "Sato", "taro.yamada@example.com")
        
        when: "重複するメールアドレスでユーザーを作成しようとする"
        userApplicationService.createUser(duplicateCommand)
        
        then: "ビジネスルール違反のログが出力される"
        thrown(UserAlreadyExistsException)
        assertLogContains("Business rule violation: User already exists")
        assertLogContains("taro.yamada@example.com", ch.qos.logback.classic.Level.WARN)
    }
}
```

### ログ設定のテスト

```groovy
// src/test/resources/application-integrationtest.properties
# ログテスト用の設定
logging.level.com.example=DEBUG
logging.level.org.springframework.security=DEBUG
logging.level.org.springframework.web=DEBUG
logging.level.org.springframework.data=DEBUG

# ログパターンの設定
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n

# ファイルログの設定（テスト時は無効）
logging.file.name=logs/application.log
logging.file.max-size=10MB
logging.file.max-history=30
```

### ログテストのベストプラクティス

1. **ログの分離**: テスト用のログ設定を使用し、他のテストに影響しないようにする
2. **ログレベルの確認**: 適切なログレベル（INFO、WARN、ERROR）が使用されているかを検証
3. **メッセージの検証**: 重要なビジネスイベントがログに記録されているかを確認
4. **パフォーマンス**: ログキャプチャがテストの実行時間に影響しないようにする
5. **クリーンアップ**: テスト後にログキャプチャを適切にクリーンアップする

### ログテストの利点

- **デバッグ支援**: テスト失敗時にログを確認して問題を特定
- **監査証跡**: 重要なビジネスイベントが記録されていることを保証
- **セキュリティ**: セキュリティ関連のイベントが適切にログに記録されることを確認
- **運用支援**: 本番環境での問題追跡に必要なログが出力されることを保証

### データベースを含むインテグレーションテスト

```groovy
@SpringBootTest
@ActiveProfiles("test")
class UserServiceIntegrationSpec extends Specification {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    SpannerUserRepository userRepository
    
    def cleanup() {
        userRepository.deleteAll()
    }
    
    def "should create and retrieve user through application service"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        
        when: "アプリケーションサービスでユーザーを作成"
        def createdUser = userApplicationService.createUser(command)
        
        and: "作成されたユーザーを検索"
        def foundUser = userApplicationService.findUser(createdUser.getId())
        
        then: "ユーザーが正しく作成・検索される"
        foundUser.isPresent()
        foundUser.get().getId() == createdUser.getId()
        foundUser.get().getFullName() == "Taro Yamada"
        foundUser.get().getEmail() == "taro.yamada@example.com"
    }
    
    def "should handle duplicate email during user creation"() {
        given: "既存のユーザー"
        def existingCommand = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        userApplicationService.createUser(existingCommand)
        
        and: "同じメールアドレスでユーザー作成コマンド"
        def duplicateCommand = new CreateUserCommand("Jiro", "Sato", "taro.yamada@example.com")
        
        when: "重複するメールアドレスでユーザーを作成しようとする"
        userApplicationService.createUser(duplicateCommand)
        
        then: "例外が発生する"
        thrown(UserAlreadyExistsException)
    }
}
```

## REST APIエンドポイントのテスト

### 完全なAPIフローのテスト

```groovy
@SpringBootTest
@AutoConfigureMockMvc
class UserApiIntegrationSpec extends Specification {
    
    @Autowired
    MockMvc mockMvc
    
    @Autowired
    ObjectMapper objectMapper
    
    @Autowired
    SpannerUserRepository userRepository
    
    def cleanup() {
        userRepository.deleteAll()
    }
    
    @WithMockUser(username = "testuser", roles = ["USER"])
    def "complete user lifecycle through API"() {
        given: "ユーザー作成リクエスト"
        def createRequest = [
            firstName: "Taro",
            lastName: "Yamada",
            email: "taro.yamada@example.com"
        ]
        
        when: "ユーザーを作成"
        def createResult = mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(objectMapper.writeValueAsString(createRequest))
                .with(csrf()))
        
        then: "ユーザーが作成される"
        createResult.andExpect(status().isCreated())
        
        and: "レスポンスからユーザーIDを取得"
        def responseJson = objectMapper.readTree(createResult.andReturn().response.contentAsString)
        def userId = responseJson.get("id").asText()
        
        when: "作成されたユーザーを取得"
        def getResult = mockMvc.perform(get("/api/v1/users/${userId}")
                .with(csrf()))
        
        then: "ユーザー情報が正しく返される"
        getResult.andExpect(status().isOk())
                .andExpect(jsonPath('$.id', is(userId)))
                .andExpect(jsonPath('$.fullName', is('Taro Yamada')))
                .andExpect(jsonPath('$.email', is('taro.yamada@example.com')))
        
        when: "ユーザー情報を更新"
        def updateRequest = [
            firstName: "Jiro",
            lastName: "Sato"
        ]
        def updateResult = mockMvc.perform(put("/api/v1/users/${userId}")
                .contentType("application/json")
                .content(objectMapper.writeValueAsString(updateRequest))
                .with(csrf()))
        
        then: "ユーザー情報が更新される"
        updateResult.andExpect(status().isOk())
                .andExpect(jsonPath('$.fullName', is('Jiro Sato')))
        
        when: "ユーザーを削除"
        def deleteResult = mockMvc.perform(delete("/api/v1/users/${userId}")
                .with(csrf()))
        
        then: "ユーザーが削除される"
        deleteResult.andExpect(status().isNoContent())
        
        when: "削除されたユーザーを取得しようとする"
        def getDeletedResult = mockMvc.perform(get("/api/v1/users/${userId}")
                .with(csrf()))
        
        then: "404 Not Foundが返される"
        getDeletedResult.andExpect(status().isNotFound())
    }
}
```

### バリデーションエラーのテスト

```groovy
@WithMockUser(username = "testuser", roles = ["USER"])
def "should return validation errors for invalid user creation request"() {
    given: "無効なユーザー作成リクエスト"
    def invalidRequest = [
        firstName: "", // 空の名前
        lastName: "Yamada",
        email: "invalid-email" // 無効なメールアドレス
    ]
    
    when: "無効なリクエストでユーザー作成APIを呼び出す"
    def result = mockMvc.perform(post("/api/v1/users")
            .contentType("application/json")
            .content(objectMapper.writeValueAsString(invalidRequest))
            .with(csrf()))
    
    then: "バリデーションエラーが返される"
    result.andExpect(status().isBadRequest())
            .andExpect(jsonPath('$.errors').exists())
            .andExpect(jsonPath('$.errors[?(@.field == "firstName")]').exists())
            .andExpect(jsonPath('$.errors[?(@.field == "email")]').exists())
}
```

## 認証・認可のテスト

### Spring Security Testの活用

```groovy
@SpringBootTest
@AutoConfigureMockMvc
class UserSecurityIntegrationSpec extends Specification {
    
    @Autowired
    MockMvc mockMvc
    
    @Autowired
    ObjectMapper objectMapper
    
    def "should allow access to public endpoints without authentication"() {
        when: "認証なしでヘルスチェックエンドポイントにアクセス"
        def result = mockMvc.perform(get("/actuator/health"))
        
        then: "アクセスが許可される"
        result.andExpect(status().isOk())
    }
    
    @WithMockUser(username = "user", roles = ["USER"])
    def "should allow user to access their own data"() {
        given: "ユーザーID"
        def userId = "user-1"
        
        when: "自分のユーザー情報にアクセス"
        def result = mockMvc.perform(get("/api/v1/users/${userId}")
                .with(csrf()))
        
        then: "アクセスが許可される"
        result.andExpect(status().isOk())
    }
    
    @WithMockUser(username = "admin", roles = ["ADMIN"])
    def "should allow admin to access all user data"() {
        given: "任意のユーザーID"
        def userId = "any-user-id"
        
        when: "管理者が任意のユーザー情報にアクセス"
        def result = mockMvc.perform(get("/api/v1/users/${userId}")
                .with(csrf()))
        
        then: "アクセスが許可される"
        result.andExpect(status().isOk())
    }
    
    @WithMockUser(username = "user", roles = ["USER"])
    def "should deny user access to other user's data"() {
        given: "他のユーザーのID"
        def otherUserId = "other-user-id"
        
        when: "他のユーザーの情報にアクセス"
        def result = mockMvc.perform(get("/api/v1/users/${otherUserId}")
                .with(csrf()))
        
        then: "アクセスが拒否される"
        result.andExpect(status().isForbidden())
    }
    
    def "should require authentication for protected endpoints"() {
        when: "認証なしで保護されたエンドポイントにアクセス"
        def result = mockMvc.perform(get("/api/v1/users/any-id"))
        
        then: "認証が必要"
        result.andExpect(status().isUnauthorized())
    }
}
```

### JWT認証のテスト

```groovy
def "should authenticate with valid JWT token"() {
    given: "有効なJWTトークン"
    def validToken = generateValidJwtToken("testuser", ["USER"])
    
    when: "JWTトークン付きでAPIにアクセス"
    def result = mockMvc.perform(get("/api/v1/users/profile")
            .header("Authorization", "Bearer ${validToken}")
            .with(csrf()))
    
    then: "アクセスが許可される"
    result.andExpect(status().isOk())
}

def "should reject invalid JWT token"() {
    given: "無効なJWTトークン"
    def invalidToken = "invalid.jwt.token"
    
    when: "無効なJWTトークン付きでAPIにアクセス"
    def result = mockMvc.perform(get("/api/v1/users/profile")
            .header("Authorization", "Bearer ${invalidToken}")
            .with(csrf()))
    
    then: "アクセスが拒否される"
    result.andExpect(status().isUnauthorized())
}
```

## メッセージングのインテグレーションテスト

### RabbitMQを含むテスト

```groovy
@SpringBootTest
@ActiveProfiles("test")
class UserMessagingIntegrationSpec extends Specification {
    
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
}
```

## テストの分離とパフォーマンス最適化

### テストプロファイルの活用

```groovy
// application-integrationtest.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# テスト用の設定
spring.profiles.active=test
logging.level.org.springframework.security=DEBUG
logging.level.com.example=DEBUG
```

### テストコンテキストのキャッシュ

```groovy
@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
class UserServiceIntegrationSpec extends Specification {
    
    // 各テストメソッド後にコンテキストをクリーンアップ
    // テスト間の独立性を保証
}
```

### テストデータの管理

```groovy
@SpringBootTest
@Transactional
@Rollback
class UserRepositoryIntegrationSpec extends Specification {
    
    @Autowired
    UserRepository userRepository
    
    def "should persist and retrieve user"() {
        given: "テストユーザー"
        def user = new User("test@example.com", "Test User")
        
        when: "ユーザーを保存"
        def savedUser = userRepository.save(user)
        
        then: "ユーザーが正しく保存される"
        savedUser.id != null
        savedUser.email == "test@example.com"
        
        when: "保存されたユーザーを検索"
        def foundUser = userRepository.findById(savedUser.id)
        
        then: "ユーザーが正しく検索される"
        foundUser.isPresent()
        foundUser.get().email == "test@example.com"
    }
}
```

## セキュリティテストの拡充

### CSRF保護のテスト

```groovy
@SpringBootTest
@AutoConfigureMockMvc
class CsrfIntegrationSpec extends Specification {
    
    @Autowired
    MockMvc mockMvc
    
    def "should require CSRF token for state-changing operations"() {
        given: "ユーザー作成リクエスト"
        def request = [
            firstName: "Taro",
            lastName: "Yamada",
            email: "taro@example.com"
        ]
        
        when: "CSRFトークンなしでPOSTリクエスト"
        def result = mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(objectMapper.writeValueAsString(request)))
        
        then: "403 Forbiddenが返される"
        result.andExpect(status().isForbidden())
        
        when: "CSRFトークン付きでPOSTリクエスト"
        def resultWithCsrf = mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(objectMapper.writeValueAsString(request))
                .with(csrf()))
        
        then: "リクエストが成功する"
        resultWithCsrf.andExpect(status().isCreated())
    }
}
```

### レート制限のテスト

```groovy
def "should enforce rate limiting"() {
    given: "複数のリクエスト"
    
    when: "短時間で複数のリクエストを送信"
    def results = (1..10).collect { i ->
        mockMvc.perform(get("/api/v1/users"))
    }
    
    then: "一部のリクエストが429 Too Many Requestsを返す"
    def tooManyRequests = results.count { result ->
        result.andReturn().response.status == 429
    }
    tooManyRequests > 0
}
```

## CI/CD統合

### GitHub Actionsでの結合テスト実行

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run integration tests
        run: ./gradlew integrationTest
        env:
          SPRING_PROFILES_ACTIVE: integration-test
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: postgres
          SPRING_DATASOURCE_PASSWORD: postgres
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: build/test-results/
```

### テストレポートの生成

```groovy
// build.gradle
test {
    useJUnitPlatform()
    
    reports {
        html.required = true
        junitXml.required = true
    }
    
    // 結合テスト用のタスク
    task integrationTest(type: Test) {
        description = 'Runs integration tests.'
        group = 'verification'
        
        testClassesDirs = sourceSets.test.output.classesDirs
        classpath = sourceSets.test.runtimeClasspath
        
        useJUnitPlatform {
            includeTags 'integration'
        }
        
        shouldRunAfter test
    }
}
```

## テストデータファクトリー

### テストデータの生成

```groovy
// src/test/groovy/com/example/test/TestDataFactory.groovy
class TestDataFactory {
    
    static User createUser(String email = "test@example.com", String name = "Test User") {
        return new User(email, name)
    }
    
    static CreateUserCommand createUserCommand(
        String firstName = "Taro",
        String lastName = "Yamada",
        String email = "taro.yamada@example.com"
    ) {
        return new CreateUserCommand(firstName, lastName, email)
    }
    
    static List<User> createUsers(int count) {
        return (1..count).collect { i ->
            createUser("user${i}@example.com", "User ${i}")
        }
    }
}
```

### テストデータベースのセットアップ

```groovy
@SpringBootTest
@ActiveProfiles("test")
class DatabaseIntegrationSpec extends Specification {
    
    @Autowired
    UserRepository userRepository
    
    @Autowired
    TestEntityManager entityManager
    
    def setup() {
        // テストデータのセットアップ
        def users = TestDataFactory.createUsers(5)
        users.each { user ->
            entityManager.persist(user)
        }
        entityManager.flush()
    }
    
    def cleanup() {
        // テストデータのクリーンアップ
        userRepository.deleteAll()
    }
    
    def "should find all users"() {
        when: "全ユーザーを検索"
        def users = userRepository.findAll()
        
        then: "5人のユーザーが取得される"
        users.size() == 5
    }
}
```

## パフォーマンステスト

### レスポンス時間のテスト

```groovy
def "should respond within acceptable time"() {
    given: "大量のテストデータ"
    def users = TestDataFactory.createUsers(1000)
    users.each { user ->
        userRepository.save(user)
    }
    
    when: "ユーザー一覧を取得"
    def startTime = System.currentTimeMillis()
    def result = mockMvc.perform(get("/api/v1/users"))
    def endTime = System.currentTimeMillis()
    
    then: "レスポンスが成功する"
    result.andExpect(status().isOk())
    
    and: "レスポンス時間が500ms以内"
    (endTime - startTime) < 500
}
```

### メモリ使用量のテスト

```groovy
def "should not cause memory leaks"() {
    given: "大量のリクエスト"
    def initialMemory = Runtime.runtime.totalMemory() - Runtime.runtime.freeMemory()
    
    when: "100回のリクエストを実行"
    (1..100).each { i ->
        mockMvc.perform(get("/api/v1/users"))
    }
    
    then: "メモリ使用量が大幅に増加しない"
    def finalMemory = Runtime.runtime.totalMemory() - Runtime.runtime.freeMemory()
    def memoryIncrease = finalMemory - initialMemory
    memoryIncrease < 10 * 1024 * 1024 // 10MB以下
}
```

## エラーハンドリングのテスト

### 例外処理のテスト

```groovy
def "should handle database connection errors gracefully"() {
    given: "データベース接続エラーをシミュレート"
    // データベース接続を切断する処理
    
    when: "APIリクエストを実行"
    def result = mockMvc.perform(get("/api/v1/users"))
    
    then: "適切なエラーレスポンスが返される"
    result.andExpect(status().isServiceUnavailable())
            .andExpect(jsonPath('$.error').exists())
            .andExpect(jsonPath('$.message').exists())
}
```

### バリデーションエラーの詳細テスト

```groovy
def "should return detailed validation errors"() {
    given: "無効なリクエスト"
    def invalidRequest = [
        firstName: "", // 空文字
        lastName: "A", // 短すぎる
        email: "invalid-email" // 無効なメール形式
    ]
    
    when: "バリデーションエラーが発生するリクエスト"
    def result = mockMvc.perform(post("/api/v1/users")
            .contentType("application/json")
            .content(objectMapper.writeValueAsString(invalidRequest))
            .with(csrf()))
    
    then: "詳細なエラー情報が返される"
    result.andExpect(status().isBadRequest())
            .andExpect(jsonPath('$.errors').isArray())
            .andExpect(jsonPath('$.errors[?(@.field == "firstName")].message').value(containsString("空")))
            .andExpect(jsonPath('$.errors[?(@.field == "lastName")].message').value(containsString("短")))
            .andExpect(jsonPath('$.errors[?(@.field == "email")].message').value(containsString("メール")))
}
```

## 学習のポイント

1. **実際のコンテキストでのテスト**: Spring Boot Testで実際のアプリケーションコンテキストを使用
2. **エンドツーエンドの検証**: APIエンドポイントからデータベースまで一連のフローをテスト
3. **セキュリティの検証**: 認証・認可の動作を実際の環境で確認
4. **メッセージングの検証**: 非同期処理やイベント発行の動作を確認
5. **テストの分離**: プロファイルとコンテキスト管理でテスト間の独立性を保証
6. **パフォーマンス考慮**: レスポンス時間とメモリ使用量の監視
7. **CI/CD統合**: 自動化されたテスト実行とレポート生成
8. **エラーハンドリング**: 例外処理とバリデーションエラーの詳細テスト

## ベストプラクティス

### テスト設計の原則

1. **テストの独立性**: 各テストは他のテストに依存しない
2. **データのクリーンアップ**: テスト後にデータを確実にクリーンアップ
3. **プロファイル分離**: 結合テスト用の専用プロファイルを使用
4. **コンテキストキャッシュ**: 適切なコンテキスト管理でパフォーマンスを最適化

### セキュリティテスト

1. **認証・認可の検証**: 各ロールの権限を実際にテスト
2. **CSRF保護**: 状態変更操作でのCSRFトークン検証
3. **レート制限**: 過度なリクエストに対する制限の確認
4. **入力検証**: 悪意のある入力に対する適切な処理

### パフォーマンスとスケーラビリティ

1. **レスポンス時間**: 許容可能な時間内でのレスポンス
2. **メモリ使用量**: メモリリークの検出と防止
3. **データベース接続**: 接続プールの適切な管理
4. **非同期処理**: メッセージングとイベント処理の検証

次のセクションでは、アーキテクチャテストについて詳しく解説します。 