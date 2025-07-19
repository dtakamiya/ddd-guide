# 8.2 Spockフレームワークの基本

SpockはGroovy言語で記述する、表現力豊かで可読性の高いテスティングフレームワークです。BDD（ビヘイビア駆動開発）スタイルでテストを記述できるため、テストの意図が明確になります。

## BDDスタイルのテスト記述

Spockのテストは、自然言語に近い形で記述できます。これにより、技術者以外の人でもテストの意図を理解しやすくなります。

### 基本的な構造

```groovy
import spock.lang.Specification

class UserSpec extends Specification {
    
    def "ユーザー名を変更できる"() {
        given: "有効なユーザーが存在する"
        def user = new User("1", "old-name", "test@example.com")
        
        when: "ユーザー名を新しい名前に変更する"
        user.changeName("new-name")
        
        then: "ユーザー名が新しい名前に更新される"
        user.getName() == "new-name"
    }
}
```

## given-when-then構造

Spockのテストは、以下の4つのブロックで構成されます：

### 1. given（前提条件）
テストの前提条件やセットアップを記述します。

```groovy
def "ユーザー作成のテスト"() {
    given: "ユーザー作成に必要なデータを準備"
    def firstName = "Taro"
    def lastName = "Yamada"
    def email = "taro.yamada@example.com"
    
    // テストデータの準備
    def fullName = new FullName(firstName, lastName)
    def emailObj = new Email(email)
}
```

### 2. when（実行）
テスト対象のコードを実行します。

```groovy
when: "ユーザーを作成する"
def user = User.create(fullName, emailObj)
```

### 3. then（検証）
実行結果が期待通りであることを検証します。

```groovy
then: "ユーザーが正しく作成される"
user.getId() != null
user.getFullName() == fullName
user.getEmail() == emailObj
```

### 4. where（パラメータ化）
パラメータ化テストのデータを提供します。

```groovy
def "メールアドレスの検証"() {
    given: "テストデータを準備"
    def email = testEmail
    
    when: "Emailオブジェクトを作成する"
    def result = new Email(email)
    
    then: "期待される結果になる"
    result.getValue() == expectedEmail
    
    where: "以下のテストケースを実行"
    testEmail           | expectedEmail
    "test@example.com"  | "test@example.com"
    "user@domain.org"   | "user@domain.org"
    "admin@test.co.jp"  | "admin@test.co.jp"
}
```

## パラメータ化テスト

Spockの強力な機能の一つが、パラメータ化テストです。複数のテストケースを効率的に記述できます。

### 基本的なパラメータ化テスト

```groovy
def "数値の加算テスト"() {
    expect: "a + b = c"
    a + b == c
    
    where: "テストデータ"
    a | b | c
    1 | 2 | 3
    5 | 3 | 8
    0 | 0 | 0
    -1| 1 | 0
}
```

### 複雑なパラメータ化テスト

```groovy
def "ユーザー作成の境界値テスト"() {
    given: "テストデータ"
    def firstName = testFirstName
    def lastName = testLastName
    def email = testEmail
    
    when: "ユーザーを作成する"
    def user = User.create(new FullName(firstName, lastName), new Email(email))
    
    then: "期待される結果になる"
    user.getFullName().getFirstName() == expectedFirstName
    user.getFullName().getLastName() == expectedLastName
    user.getEmail().getValue() == expectedEmail
    
    where: "境界値テストケース"
    testFirstName | testLastName | testEmail              | expectedFirstName | expectedLastName | expectedEmail
    "A"           | "B"          | "a.b@example.com"      | "A"               | "B"              | "a.b@example.com"
    "VeryLongName"| "VeryLong"   | "very.long@example.com"| "VeryLongName"    | "VeryLong"       | "very.long@example.com"
    "Taro"        | ""           | "taro@example.com"     | "Taro"            | ""               | "taro@example.com"
}
```

## モックとスタブの活用

Spockは優れたモック機能を提供しており、外部依存を簡単にモックできます。

### 基本的なモック

```groovy
class UserServiceSpec extends Specification {
    
    @Subject
    UserService userService
    
    UserRepository userRepository = Mock()
    EmailService emailService = Mock()
    
    def setup() {
        userService = new UserService(userRepository, emailService)
    }
    
    def "ユーザー作成時にメール送信される"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro@example.com")
        def savedUser = User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com"))
        
        when: "ユーザーを作成する"
        def result = userService.createUser(command)
        
        then: "リポジトリに保存され、メールが送信される"
        1 * userRepository.save(_) >> savedUser
        1 * emailService.sendWelcomeEmail(savedUser.getEmail())
        result.getId() == savedUser.getId()
    }
}
```

### スタブの活用

```groovy
def "ユーザー検索のテスト"() {
    given: "リポジトリのスタブを設定"
    def expectedUser = User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com"))
    userRepository.findById("user-1") >> Optional.of(expectedUser)
    
    when: "ユーザーを検索する"
    def result = userService.findUser("user-1")
    
    then: "期待されるユーザーが返される"
    result.isPresent()
    result.get().getId() == "user-1"
    result.get().getFullName().getFullName() == "Taro Yamada"
}
```

## 例外テスト

Spockでは、例外のテストも簡単に記述できます。

```groovy
def "無効なメールアドレスで例外が発生する"() {
    given: "無効なメールアドレス"
    def invalidEmail = "invalid-email"
    
    when: "Emailオブジェクトを作成する"
    new Email(invalidEmail)
    
    then: "IllegalArgumentExceptionが発生する"
    thrown(IllegalArgumentException)
}

def "特定のメッセージを含む例外テスト"() {
    given: "無効なメールアドレス"
    def invalidEmail = "invalid-email"
    
    when: "Emailオブジェクトを作成する"
    new Email(invalidEmail)
    
    then: "特定のメッセージを含む例外が発生する"
    def exception = thrown(IllegalArgumentException)
    exception.message.contains("Invalid email format")
}
```

## データテーブルの活用

Spockのデータテーブル機能により、複雑なテストケースも簡潔に記述できます。

```groovy
def "ユーザー権限のテスト"() {
    given: "ユーザーと権限を準備"
    def user = new User("user-1", "Test User", "test@example.com")
    user.setRole(userRole)
    
    when: "権限チェックを実行"
    def hasPermission = userService.hasPermission(user, requiredPermission)
    
    then: "期待される権限結果"
    hasPermission == expectedResult
    
    where: "権限テストケース"
    userRole    | requiredPermission | expectedResult
    "ADMIN"     | "READ"            | true
    "ADMIN"     | "WRITE"           | true
    "USER"      | "READ"            | true
    "USER"      | "WRITE"           | false
    "GUEST"     | "READ"            | false
    "GUEST"     | "WRITE"           | false
}
```

## 学習のポイント

1. **BDDスタイルの活用**: 自然言語に近い形でテストの意図を明確に記述
2. **given-when-then構造**: テストの構造を統一し、可読性を向上
3. **パラメータ化テスト**: 複数のテストケースを効率的に記述
4. **モックとスタブ**: 外部依存を適切にモックし、テストの独立性を確保

次のセクションでは、具体的なユニットテストの実装について詳しく解説します。 