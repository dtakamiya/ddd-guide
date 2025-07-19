# 8.7 テストデータ管理

テストデータ管理は、テストの品質と保守性を左右する重要な要素です。適切なテストフィクスチャの設計、テストデータのクリーンアップ、テストの独立性確保について解説します。

## テストフィクスチャの設計

### ファクトリパターンの活用

テストデータの作成を効率化し、一貫性を保つためにファクトリパターンを活用します。

```groovy
// src/test/groovy/com/example/test/fixture/UserFixture.groovy
class UserFixture {
    
    static User createValidUser() {
        return User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
    }
    
    static User createUserWithCustomData(String firstName, String lastName, String email) {
        return User.create(
            new FullName(firstName, lastName),
            new Email(email)
        )
    }
    
    static User createDeletedUser() {
        def user = createValidUser()
        user.delete()
        return user
    }
    
    static List<User> createMultipleUsers(int count) {
        return (1..count).collect { index ->
            createUserWithCustomData(
                "User${index}",
                "Test",
                "user${index}@example.com"
            )
        }
    }
    
    static CreateUserCommand createValidCommand() {
        return new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
    }
    
    static CreateUserCommand createCommandWithInvalidEmail() {
        return new CreateUserCommand("Taro", "Yamada", "invalid-email")
    }
}
```

### ビルダーパターンの活用

複雑なテストデータを作成する際にビルダーパターンを活用します。

```groovy
// src/test/groovy/com/example/test/fixture/UserBuilder.groovy
class UserBuilder {
    private String firstName = "Taro"
    private String lastName = "Yamada"
    private String email = "taro.yamada@example.com"
    private boolean deleted = false
    
    UserBuilder withFirstName(String firstName) {
        this.firstName = firstName
        return this
    }
    
    UserBuilder withLastName(String lastName) {
        this.lastName = lastName
        return this
    }
    
    UserBuilder withEmail(String email) {
        this.email = email
        return this
    }
    
    UserBuilder deleted() {
        this.deleted = true
        return this
    }
    
    User build() {
        def user = User.create(
            new FullName(firstName, lastName),
            new Email(email)
        )
        
        if (deleted) {
            user.delete()
        }
        
        return user
    }
    
    static UserBuilder aUser() {
        return new UserBuilder()
    }
}
```

### テストでのファクトリ活用

```groovy
// src/test/groovy/com/example/domain/UserSpec.groovy
class UserSpec extends Specification {
    
    def "should create user with valid data"() {
        given: "有効なユーザーデータ"
        def user = UserFixture.createValidUser()
        
        when: "ユーザーを作成"
        def result = user
        
        then: "ユーザーが正しく作成される"
        result.getId() != null
        result.getFullName().getFullName() == "Taro Yamada"
        result.getEmail().getValue() == "taro.yamada@example.com"
    }
    
    def "should create user with builder pattern"() {
        given: "ビルダーでユーザーを作成"
        def user = UserBuilder.aUser()
            .withFirstName("Jiro")
            .withLastName("Sato")
            .withEmail("jiro.sato@example.com")
            .build()
        
        when: "ユーザーを作成"
        def result = user
        
        then: "ユーザーが正しく作成される"
        result.getFullName().getFullName() == "Jiro Sato"
        result.getEmail().getValue() == "jiro.sato@example.com"
    }
    
    def "should create deleted user"() {
        given: "削除されたユーザー"
        def user = UserBuilder.aUser().deleted().build()
        
        expect: "ユーザーが削除状態"
        user.isDeleted()
        user.getDeletedAt() != null
    }
}
```

## テストデータのクリーンアップ

### 自動クリーンアップの実装

テストの独立性を保つために、各テスト後にデータをクリーンアップします。

```groovy
// src/test/groovy/com/example/infrastructure/AbstractRepositoryTest.groovy
@SpringBootTest
@ActiveProfiles("test")
abstract class AbstractRepositoryTest extends Specification {
    
    @Autowired
    SpannerUserRepository userRepository
    
    def cleanup() {
        userRepository.deleteAll()
    }
    
    def cleanupSpec() {
        // テストクラス終了時のクリーンアップ
        userRepository.deleteAll()
    }
}

// src/test/groovy/com/example/infrastructure/UserRepositoryTest.groovy
class UserRepositoryTest extends AbstractRepositoryTest {
    
    def "should save and retrieve user"() {
        given: "ユーザーを作成"
        def user = UserFixture.createValidUser()
        
        when: "ユーザーを保存して検索"
        userRepository.save(user)
        def found = userRepository.findById(user.getId())
        
        then: "ユーザーが正しく保存・検索される"
        found.isPresent()
        found.get().getId() == user.getId()
    }
    
    // 各テスト後にcleanup()が自動実行される
}
```

### トランザクション管理

```groovy
// src/test/groovy/com/example/application/UserServiceTest.groovy
@SpringBootTest
@Transactional
@Rollback
class UserServiceTest extends Specification {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    SpannerUserRepository userRepository
    
    def "should create user in transaction"() {
        given: "ユーザー作成コマンド"
        def command = UserFixture.createValidCommand()
        
        when: "ユーザーを作成"
        def user = userApplicationService.createUser(command)
        
        then: "ユーザーが作成される"
        user.getId() != null
        
        and: "データベースに保存される"
        def saved = userRepository.findById(user.getId())
        saved.isPresent()
    }
    
    // @Rollbackにより、テスト終了時に自動的にロールバックされる
}
```

## テストの独立性確保

### データベースの分離

```groovy
// src/test/groovy/com/example/infrastructure/IsolatedDatabaseTest.groovy
@SpringBootTest
@ActiveProfiles("test")
@TestPropertySource(properties = [
    "spring.cloud.gcp.spanner.database=test-db-${System.currentTimeMillis()}"
])
class IsolatedDatabaseTest extends Specification {
    
    @Autowired
    SpannerUserRepository userRepository
    
    def "should work with isolated database"() {
        given: "ユーザーを作成"
        def user = UserFixture.createValidUser()
        
        when: "ユーザーを保存"
        userRepository.save(user)
        
        then: "ユーザーが保存される"
        def found = userRepository.findById(user.getId())
        found.isPresent()
    }
}
```

### メッセージキューの分離

```groovy
// src/test/groovy/com/example/infrastructure/IsolatedMessagingTest.groovy
@SpringBootTest
@ActiveProfiles("test")
@TestPropertySource(properties = [
    "spring.rabbitmq.queue.prefix=test-${System.currentTimeMillis()}-"
])
class IsolatedMessagingTest extends Specification {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    RabbitTemplate rabbitTemplate
    
    def cleanup() {
        // テスト用キューをクリーンアップ
        rabbitTemplate.execute { channel ->
            channel.queuePurge("test-user-events")
        }
    }
    
    def "should publish event to isolated queue"() {
        given: "ユーザー作成コマンド"
        def command = UserFixture.createValidCommand()
        
        when: "ユーザーを作成"
        def user = userApplicationService.createUser(command)
        
        then: "イベントが分離されたキューに発行される"
        def message = rabbitTemplate.receiveAndConvert("test-user-events", 5000)
        message != null
        message.userId == user.getId()
    }
}
```

## テストデータの管理戦略

### データセットの定義

```groovy
// src/test/groovy/com/example/test/fixture/TestDataSets.groovy
class TestDataSets {
    
    static List<User> createStandardUsers() {
        return [
            UserFixture.createUserWithCustomData("Taro", "Yamada", "taro@example.com"),
            UserFixture.createUserWithCustomData("Jiro", "Sato", "jiro@example.com"),
            UserFixture.createUserWithCustomData("Saburo", "Tanaka", "saburo@example.com")
        ]
    }
    
    static List<User> createAdminUsers() {
        return [
            UserFixture.createUserWithCustomData("Admin", "User", "admin@example.com"),
            UserFixture.createUserWithCustomData("Manager", "User", "manager@example.com")
        ]
    }
    
    static List<User> createDeletedUsers() {
        return [
            UserBuilder.aUser().withFirstName("Deleted").deleted().build(),
            UserBuilder.aUser().withFirstName("Removed").deleted().build()
        ]
    }
}
```

### テストデータの事前準備

```groovy
// src/test/groovy/com/example/infrastructure/DataSetupTest.groovy
@SpringBootTest
@ActiveProfiles("test")
class DataSetupTest extends Specification {
    
    @Autowired
    SpannerUserRepository userRepository
    
    def setup() {
        // テストデータを事前に準備
        def users = TestDataSets.createStandardUsers()
        users.each { userRepository.save(it) }
    }
    
    def "should find pre-created users"() {
        when: "事前に作成されたユーザーを検索"
        def allUsers = userRepository.findAll()
        
        then: "標準ユーザーが存在する"
        allUsers.size() >= 3
        allUsers.any { it.getFullName().getFullName() == "Taro Yamada" }
        allUsers.any { it.getFullName().getFullName() == "Jiro Sato" }
        allUsers.any { it.getFullName().getFullName() == "Saburo Tanaka" }
    }
}
```

## パフォーマンス最適化

### バッチ処理のテスト

```groovy
// src/test/groovy/com/example/application/BatchUserServiceTest.groovy
@SpringBootTest
@ActiveProfiles("test")
class BatchUserServiceTest extends Specification {
    
    @Autowired
    UserApplicationService userApplicationService
    
    @Autowired
    SpannerUserRepository userRepository
    
    def cleanup() {
        userRepository.deleteAll()
    }
    
    def "should handle batch user creation efficiently"() {
        given: "大量のユーザー作成コマンド"
        def commands = (1..100).collect { index ->
            UserFixture.createUserWithCustomData(
                "User${index}",
                "Test",
                "user${index}@example.com"
            )
        }
        
        when: "バッチでユーザーを作成"
        def startTime = System.currentTimeMillis()
        def results = userApplicationService.createUsers(commands)
        def endTime = System.currentTimeMillis()
        
        then: "全てのユーザーが作成される"
        results.size() == 100
        
        and: "パフォーマンス要件を満たす"
        (endTime - startTime) < 5000 // 5秒以内
        
        and: "データベースに保存される"
        def allUsers = userRepository.findAll()
        allUsers.size() == 100
    }
}
```

## 学習のポイント

1. **テストフィクスチャの設計**: 再利用可能で保守しやすいテストデータの作成
2. **テストの独立性**: 各テストが他のテストに影響されない設計
3. **効率的なクリーンアップ**: テストデータの適切な管理とクリーンアップ
4. **パフォーマンスの考慮**: 大量データでのテスト実行時の最適化

これで、テスト章の分割が完了しました。各セクションが独立したファイルとして管理され、詳細なコード例と説明が含まれています。 