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

## 学習のポイント

1. **実際のコンテキストでのテスト**: Spring Boot Testで実際のアプリケーションコンテキストを使用
2. **エンドツーエンドの検証**: APIエンドポイントからデータベースまで一連のフローをテスト
3. **セキュリティの検証**: 認証・認可の動作を実際の環境で確認
4. **メッセージングの検証**: 非同期処理やイベント発行の動作を確認

次のセクションでは、アーキテクチャテストについて詳しく解説します。 