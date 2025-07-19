# 8.3 ユニットテスト

ユニットテストは、個々のコンポーネント（クラス、メソッド）を独立してテストする手法です。DDDアーキテクチャでは、各層のコンポーネントに対して適切なユニットテストを実装することが重要です。

## ドメインオブジェクトのテスト

ドメイン層のオブジェクト（エンティティ、値オブジェクト、ドメインサービス）は、ビジネスロジックの中核となるため、特に慎重にテストする必要があります。

### 値オブジェクトのテスト

```groovy
// src/test/groovy/com/example/domain/EmailSpec.groovy
import spock.lang.Specification
import spock.lang.Unroll

class EmailSpec extends Specification {
    
    @Unroll
    def "有効なメールアドレス #email でEmailオブジェクトを作成できる"() {
        when: "Emailオブジェクトを作成する"
        def emailObj = new Email(email)
        
        then: "正しく作成される"
        emailObj.getValue() == email
        
        where: "有効なメールアドレス"
        email << [
            "test@example.com",
            "user@domain.org",
            "admin@test.co.jp",
            "user+tag@example.com"
        ]
    }
    
    @Unroll
    def "無効なメールアドレス #email で例外が発生する"() {
        when: "Emailオブジェクトを作成する"
        new Email(email)
        
        then: "IllegalArgumentExceptionが発生する"
        thrown(IllegalArgumentException)
        
        where: "無効なメールアドレス"
        email << [
            "invalid-email",
            "@example.com",
            "user@",
            "user.example.com",
            "",
            null
        ]
    }
    
    def "Emailオブジェクトの等価性テスト"() {
        given: "同じ値のEmailオブジェクト"
        def email1 = new Email("test@example.com")
        def email2 = new Email("test@example.com")
        def email3 = new Email("different@example.com")
        
        expect: "同じ値のオブジェクトは等しい"
        email1 == email2
        email1.hashCode() == email2.hashCode()
        
        and: "異なる値のオブジェクトは等しくない"
        email1 != email3
        email1.hashCode() != email3.hashCode()
    }
}
```

### エンティティのテスト

```groovy
// src/test/groovy/com/example/domain/UserSpec.groovy
import spock.lang.Specification

class UserSpec extends Specification {
    
    def "ユーザーを正しく作成できる"() {
        given: "ユーザー作成に必要なデータ"
        def fullName = new FullName("Taro", "Yamada")
        def email = new Email("taro.yamada@example.com")
        
        when: "ユーザーを作成する"
        def user = User.create(fullName, email)
        
        then: "ユーザーが正しく作成される"
        user.getId() != null
        user.getFullName() == fullName
        user.getEmail() == email
        user.getCreatedAt() != null
        user.getUpdatedAt() != null
    }
    
    def "ユーザー名を変更できる"() {
        given: "ユーザーを作成"
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        def newFullName = new FullName("Jiro", "Sato")
        
        when: "ユーザー名を変更する"
        user.changeName(newFullName)
        
        then: "ユーザー名が更新される"
        user.getFullName() == newFullName
        user.getUpdatedAt() > user.getCreatedAt()
    }
    
    def "メールアドレスを変更できる"() {
        given: "ユーザーを作成"
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        def newEmail = new Email("new.email@example.com")
        
        when: "メールアドレスを変更する"
        user.changeEmail(newEmail)
        
        then: "メールアドレスが更新される"
        user.getEmail() == newEmail
        user.getUpdatedAt() > user.getCreatedAt()
    }
    
    def "ユーザーを削除できる"() {
        given: "ユーザーを作成"
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        
        when: "ユーザーを削除する"
        user.delete()
        
        then: "ユーザーが削除状態になる"
        user.isDeleted()
        user.getDeletedAt() != null
    }
    
    def "削除されたユーザーは変更できない"() {
        given: "削除されたユーザー"
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        user.delete()
        
        when: "ユーザー名を変更しようとする"
        user.changeName(new FullName("Jiro", "Sato"))
        
        then: "例外が発生する"
        thrown(IllegalStateException)
    }
}
```

### ドメインサービスのテスト

```groovy
// src/test/groovy/com/example/domain/UserDomainServiceSpec.groovy
import spock.lang.Specification

class UserDomainServiceSpec extends Specification {
    
    @Subject
    UserDomainService userDomainService
    
    def setup() {
        userDomainService = new UserDomainService()
    }
    
    def "ユーザーが重複していない場合、trueを返す"() {
        given: "既存ユーザーのリスト"
        def existingUsers = [
            User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com")),
            User.create(new FullName("Jiro", "Sato"), new Email("jiro@example.com"))
        ]
        def newEmail = new Email("new@example.com")
        
        when: "重複チェックを実行"
        def isUnique = userDomainService.isEmailUnique(existingUsers, newEmail)
        
        then: "重複していないためtrueが返される"
        isUnique
    }
    
    def "ユーザーが重複している場合、falseを返す"() {
        given: "既存ユーザーのリスト"
        def existingUsers = [
            User.create(new FullName("Taro", "Yamada"), new Email("taro@example.com")),
            User.create(new FullName("Jiro", "Sato"), new Email("jiro@example.com"))
        ]
        def duplicateEmail = new Email("taro@example.com")
        
        when: "重複チェックを実行"
        def isUnique = userDomainService.isEmailUnique(existingUsers, duplicateEmail)
        
        then: "重複しているためfalseが返される"
        !isUnique
    }
}
```

## アプリケーションサービスのテスト

アプリケーションサービスは、ドメインオブジェクトとインフラストラクチャ層の橋渡し役を果たすため、モックを活用したテストが重要です。

### 基本的なアプリケーションサービステスト

```groovy
// src/test/groovy/com/example/application/UserApplicationServiceSpec.groovy
import spock.lang.Specification

class UserApplicationServiceSpec extends Specification {
    
    @Subject
    UserApplicationService userApplicationService
    
    UserRepository userRepository = Mock()
    UserEventPublisher eventPublisher = Mock()
    UserDomainService userDomainService = Mock()
    
    def setup() {
        userApplicationService = new UserApplicationService(
            userRepository, 
            eventPublisher, 
            userDomainService
        )
    }
    
    def "ユーザー作成が成功する"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        def savedUser = User.create(
            new FullName("Taro", "Yamada"), 
            new Email("taro.yamada@example.com")
        )
        
        when: "ユーザーを作成する"
        def result = userApplicationService.createUser(command)
        
        then: "リポジトリに保存され、イベントが発行される"
        1 * userRepository.save(_) >> savedUser
        1 * eventPublisher.publish(_)
        result.getId() == savedUser.getId()
        result.getFullName() == "Taro Yamada"
        result.getEmail() == "taro.yamada@example.com"
    }
    
    def "重複メールアドレスでユーザー作成が失敗する"() {
        given: "ユーザー作成コマンド"
        def command = new CreateUserCommand("Taro", "Yamada", "existing@example.com")
        
        and: "ドメインサービスが重複を検出"
        userDomainService.isEmailUnique(_, _) >> false
        
        when: "ユーザーを作成する"
        userApplicationService.createUser(command)
        
        then: "例外が発生する"
        thrown(UserAlreadyExistsException)
        
        and: "リポジトリは呼ばれない"
        0 * userRepository.save(_)
        0 * eventPublisher.publish(_)
    }
    
    def "ユーザー検索が成功する"() {
        given: "存在するユーザー"
        def userId = "user-1"
        def user = User.create(
            new FullName("Taro", "Yamada"), 
            new Email("taro.yamada@example.com")
        )
        userRepository.findById(userId) >> Optional.of(user)
        
        when: "ユーザーを検索する"
        def result = userApplicationService.findUser(userId)
        
        then: "ユーザーが返される"
        result.isPresent()
        result.get().getId() == userId
        result.get().getFullName() == "Taro Yamada"
    }
    
    def "存在しないユーザー検索で空の結果が返される"() {
        given: "存在しないユーザーID"
        def userId = "non-existent"
        userRepository.findById(userId) >> Optional.empty()
        
        when: "ユーザーを検索する"
        def result = userApplicationService.findUser(userId)
        
        then: "空の結果が返される"
        !result.isPresent()
    }
}
```

### 複雑なビジネスロジックのテスト

```groovy
def "ユーザー一括作成が成功する"() {
    given: "複数のユーザー作成コマンド"
    def commands = [
        new CreateUserCommand("Taro", "Yamada", "taro@example.com"),
        new CreateUserCommand("Jiro", "Sato", "jiro@example.com"),
        new CreateUserCommand("Saburo", "Tanaka", "saburo@example.com")
    ]
    
    and: "作成されるユーザー"
    def users = commands.collect { cmd ->
        User.create(
            new FullName(cmd.getFirstName(), cmd.getLastName()),
            new Email(cmd.getEmail())
        )
    }
    
    when: "ユーザーを一括作成する"
    def results = userApplicationService.createUsers(commands)
    
    then: "全てのユーザーが作成される"
    commands.size() * userRepository.save(_)
    commands.size() * eventPublisher.publish(_)
    results.size() == commands.size()
    
    and: "各ユーザーが正しく作成される"
    results.eachWithIndex { result, index ->
        result.getId() == users[index].getId()
        result.getFullName() == "${commands[index].getFirstName()} ${commands[index].getLastName()}"
        result.getEmail() == commands[index].getEmail()
    }
}

def "一括作成で一部失敗した場合の処理"() {
    given: "複数のユーザー作成コマンド"
    def commands = [
        new CreateUserCommand("Taro", "Yamada", "taro@example.com"),
        new CreateUserCommand("Jiro", "Sato", "existing@example.com"), // 重複
        new CreateUserCommand("Saburo", "Tanaka", "saburo@example.com")
    ]
    
    and: "ドメインサービスが2番目のユーザーを重複として検出"
    userDomainService.isEmailUnique(_, _) >>> [true, false, true]
    
    when: "ユーザーを一括作成する"
    def results = userApplicationService.createUsers(commands)
    
    then: "成功したユーザーのみ作成される"
    2 * userRepository.save(_)
    2 * eventPublisher.publish(_)
    results.size() == 2
    
    and: "失敗したユーザーは含まれない"
    results.every { result ->
        result.getEmail() != "existing@example.com"
    }
}
```

## モックとスタブの活用

### 詳細なモック検証

```groovy
def "ユーザー作成時の詳細なモック検証"() {
    given: "ユーザー作成コマンド"
    def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
    def savedUser = User.create(
        new FullName("Taro", "Yamada"), 
        new Email("taro.yamada@example.com")
    )
    
    when: "ユーザーを作成する"
    def result = userApplicationService.createUser(command)
    
    then: "リポジトリが正しい引数で呼ばれる"
    1 * userRepository.save({ User user ->
        user.getFullName().getFirstName() == "Taro"
        user.getFullName().getLastName() == "Yamada"
        user.getEmail().getValue() == "taro.yamada@example.com"
    }) >> savedUser
    
    and: "イベントが正しい引数で発行される"
    1 * eventPublisher.publish({ UserCreatedEvent event ->
        event.getUserId() == savedUser.getId()
        event.getEmail() == "taro.yamada@example.com"
    })
    
    and: "ドメインサービスが呼ばれる"
    1 * userDomainService.isEmailUnique(_, { Email email ->
        email.getValue() == "taro.yamada@example.com"
    }) >> true
}
```

### 例外シナリオのテスト

```groovy
def "リポジトリ例外時の処理"() {
    given: "ユーザー作成コマンド"
    def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
    
    and: "リポジトリが例外を投げる"
    userRepository.save(_) >> { throw new DatabaseException("Connection failed") }
    userDomainService.isEmailUnique(_, _) >> true
    
    when: "ユーザーを作成する"
    userApplicationService.createUser(command)
    
    then: "アプリケーション例外が発生する"
    thrown(UserCreationFailedException)
    
    and: "イベントは発行されない"
    0 * eventPublisher.publish(_)
}
```

## 学習のポイント

1. **ドメインロジックの重点テスト**: ビジネスルールの正確性を確実に検証
2. **モックの適切な活用**: 外部依存を適切にモックし、テストの独立性を確保
3. **境界値テスト**: 正常系と異常系の両方をテスト
4. **テストの可読性**: BDDスタイルでテストの意図を明確に記述

次のセクションでは、インテグレーションテストについて詳しく解説します。 