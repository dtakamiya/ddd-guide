# 8.1 テスト戦略の概要

アプリケーションの品質と保守性を高く保つために、テストは不可欠なプロセスです。このセクションでは、DDDとオニオンアーキテクチャに適したテスト戦略について解説します。

## テストピラミッドの理解

テストピラミッドは、テストの種類とその比率を示す重要な概念です。DDDアプリケーションでは、以下のような構成を目指します：

```
    /\
   /  \     E2Eテスト（少数）
  /____\    
 /      \   
/        \  インテグレーションテスト（中程度）
/________\  
/          \ 
/            \ ユニットテスト（多数）
/______________\
```

### 各層でのテスト戦略

#### 1. ユニットテスト（基盤）
- **対象**: ドメインオブジェクト、アプリケーションサービス、ユーティリティクラス
- **特徴**: 高速、外部依存なし、詳細なロジック検証
- **実行頻度**: 開発中に頻繁に実行

#### 2. インテグレーションテスト（中間）
- **対象**: リポジトリ実装、メッセージング、外部API連携
- **特徴**: 実際のデータベースや外部サービスとの連携確認
- **実行頻度**: コミット時、プルリクエスト時

#### 3. E2Eテスト（頂点）
- **対象**: 完全なユーザーシナリオ、APIエンドポイント全体
- **特徴**: 実際の環境に近い条件での動作確認
- **実行頻度**: リリース前、重要な機能変更時

## DDDアーキテクチャでのテスト戦略

### ドメイン層のテスト
```groovy
// ドメインオブジェクトのテスト例
class UserSpec extends Specification {
    
    def "should create user with valid data"() {
        given:
        def fullName = new FullName("Taro", "Yamada")
        def email = new Email("taro.yamada@example.com")
        
        when:
        def user = User.create(fullName, email)
        
        then:
        user.getId() != null
        user.getFullName() == fullName
        user.getEmail() == email
    }
    
    def "should not create user with invalid email"() {
        given:
        def fullName = new FullName("Taro", "Yamada")
        def invalidEmail = "invalid-email"
        
        when:
        new Email(invalidEmail)
        
        then:
        thrown(IllegalArgumentException)
    }
}
```

### アプリケーション層のテスト
```groovy
// アプリケーションサービスのテスト例
class UserApplicationServiceSpec extends Specification {
    
    @Subject
    UserApplicationService userService
    
    UserRepository userRepository = Mock()
    UserEventPublisher eventPublisher = Mock()
    
    def setup() {
        userService = new UserApplicationService(userRepository, eventPublisher)
    }
    
    def "should create user successfully"() {
        given:
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        def savedUser = User.create(new FullName("Taro", "Yamada"), new Email("taro.yamada@example.com"))
        
        when:
        def result = userService.createUser(command)
        
        then:
        1 * userRepository.save(_) >> savedUser
        1 * eventPublisher.publish(_)
        result.getId() == savedUser.getId()
    }
}
```

### インフラストラクチャ層のテスト
```groovy
// リポジトリ実装のテスト例
@SpringBootTest
@ActiveProfiles("test")
class SpannerUserRepositorySpec extends AbstractIntegrationSpec {
    
    @Autowired
    SpannerUserRepository userRepository
    
    def "should save and retrieve user"() {
        given:
        def user = User.create(
            new FullName("Test", "User"),
            new Email("test@example.com")
        )
        
        when:
        userRepository.save(user)
        def found = userRepository.findById(user.getId())
        
        then:
        found.isPresent()
        found.get().getId() == user.getId()
    }
}
```

## テストの自動化と継続的品質保証

### CI/CDパイプラインでのテスト実行

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Run unit tests
      run: ./gradlew test
    
    - name: Run integration tests
      run: ./gradlew integrationTest
    
    - name: Run architecture tests
      run: ./gradlew architectureTest
```

### テスト実行の段階的アプローチ

1. **開発中**: ユニットテストを頻繁に実行
2. **コミット時**: 全ユニットテスト + 関連インテグレーションテスト
3. **プルリクエスト時**: 全テストスイート実行
4. **リリース前**: E2Eテストを含む完全なテスト実行

## テスト品質の指標

### カバレッジ目標
- **ユニットテスト**: 80%以上（特にドメインロジック）
- **インテグレーションテスト**: 重要なフローをカバー
- **E2Eテスト**: 主要なユーザーシナリオをカバー

### テスト実行時間の最適化
- **ユニットテスト**: 1分以内
- **インテグレーションテスト**: 5分以内
- **E2Eテスト**: 10分以内

## 学習のポイント

1. **テストファーストの考え方**: 品質を保証するためのテスト駆動開発
2. **適切なテストレベル**: ユニット、インテグレーション、E2Eテストの使い分け
3. **テストの保守性**: 読みやすく、理解しやすいテストコードの作成
4. **自動化の重要性**: CI/CDパイプラインでの継続的テスト実行

次のセクションでは、Spockフレームワークの基本について詳しく解説します。 