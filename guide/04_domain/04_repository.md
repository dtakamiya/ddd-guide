# 4.6 リポジトリインターフェースの定義

リポジトリは、集約の永続化を抽象化するパターンです。ドメイン層ではインターフェースのみを定義し、インフラストラクチャ層で具体的な実装を行います。これにより、ドメイン層とインフラストラクチャ層の依存関係を逆転させます。

## 🎯 リポジトリの特徴

### 1. **集約単位での操作**
- 集約ルートを中心とした操作
- 集約内の整合性を保証
- トランザクション境界の明確化

### 2. **ドメイン層でのインターフェース定義**
- インフラストラクチャ層に依存しない
- ドメインの意図を表現
- テストの容易性

### 3. **クエリの抽象化**
- 複雑なクエリロジックのカプセル化
- ドメインに特化したクエリメソッド
- パフォーマンスの最適化

## 🗄️ ユーザーリポジトリ

### UserRepository インターフェース

```java
package com.example.domain.repository;

import com.example.domain.entity.User;
import com.example.domain.valueobject.Email;
import com.example.domain.valueobject.UserId;

import java.util.List;
import java.util.Optional;

/**
 * ユーザー集約のリポジトリインターフェース
 * 
 * ユーザーの永続化と検索を抽象化し、
 * ドメイン層とインフラストラクチャ層の境界を定義します。
 */
public interface UserRepository {
    
    /**
     * ユーザーを保存
     * 
     * @param user 保存するユーザー
     * @return 保存されたユーザー
     */
    User save(User user);
    
    /**
     * ユーザーを更新
     * 
     * @param user 更新するユーザー
     * @return 更新されたユーザー
     */
    User update(User user);
    
    /**
     * ユーザーを削除
     * 
     * @param userId 削除するユーザーのID
     */
    void delete(UserId userId);
    
    /**
     * IDでユーザーを検索
     * 
     * @param userId 検索するユーザーのID
     * @return ユーザー（存在しない場合は空）
     */
    Optional<User> findById(UserId userId);
    
    /**
     * メールアドレスでユーザーを検索
     * 
     * @param email 検索するメールアドレス
     * @return ユーザー（存在しない場合は空）
     */
    Optional<User> findByEmail(Email email);
    
    /**
     * すべてのユーザーを取得
     * 
     * @return ユーザーリスト
     */
    List<User> findAll();
    
    /**
     * アクティブなユーザーのみを取得
     * 
     * @return アクティブなユーザーリスト
     */
    List<User> findActiveUsers();
    
    /**
     * 特定のドメインのユーザーを検索
     * 
     * @param domain 検索するドメイン
     * @return 該当するユーザーリスト
     */
    List<User> findByEmailDomain(String domain);
    
    /**
     * 氏名でユーザーを検索（部分一致）
     * 
     * @param name 検索する氏名
     * @return 該当するユーザーリスト
     */
    List<User> findByNameContaining(String name);
    
    /**
     * ユーザーが存在するかどうかを確認
     * 
     * @param userId 確認するユーザーのID
     * @return 存在する場合はtrue
     */
    boolean existsById(UserId userId);
    
    /**
     * メールアドレスが重複しているかどうかを確認
     * 
     * @param email 確認するメールアドレス
     * @return 重複している場合はtrue
     */
    boolean existsByEmail(Email email);
    
    /**
     * ユーザー数を取得
     * 
     * @return ユーザー数
     */
    long count();
    
    /**
     * アクティブなユーザー数を取得
     * 
     * @return アクティブなユーザー数
     */
    long countActiveUsers();
}
```

## 📦 注文リポジトリ

### OrderRepository インターフェース

```java
package com.example.domain.repository;

import com.example.domain.aggregate.Order;
import com.example.domain.valueobject.OrderId;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

/**
 * 注文集約のリポジトリインターフェース
 * 
 * 注文の永続化と検索を抽象化し、
 * 複雑なクエリロジックをカプセル化します。
 */
public interface OrderRepository {
    
    /**
     * 注文を保存
     * 
     * @param order 保存する注文
     * @return 保存された注文
     */
    Order save(Order order);
    
    /**
     * 注文を更新
     * 
     * @param order 更新する注文
     * @return 更新された注文
     */
    Order update(Order order);
    
    /**
     * 注文を削除
     * 
     * @param orderId 削除する注文のID
     */
    void delete(OrderId orderId);
    
    /**
     * IDで注文を検索
     * 
     * @param orderId 検索する注文のID
     * @return 注文（存在しない場合は空）
     */
    Optional<Order> findById(OrderId orderId);
    
    /**
     * ユーザーの注文を検索
     * 
     * @param userId 検索するユーザーのID
     * @return 該当する注文リスト
     */
    List<Order> findByUserId(UserId userId);
    
    /**
     * ユーザーの確定済み注文を検索
     * 
     * @param userId 検索するユーザーのID
     * @return 該当する注文リスト
     */
    List<Order> findConfirmedByUserId(UserId userId);
    
    /**
     * 期間内の注文を検索
     * 
     * @param startDate 開始日時
     * @param endDate 終了日時
     * @return 該当する注文リスト
     */
    List<Order> findByDateRange(LocalDateTime startDate, LocalDateTime endDate);
    
    /**
     * ステータスで注文を検索
     * 
     * @param status 検索するステータス
     * @return 該当する注文リスト
     */
    List<Order> findByStatus(OrderStatus status);
    
    /**
     * 高額注文を検索
     * 
     * @param minAmount 最小金額
     * @return 該当する注文リスト
     */
    List<Order> findByMinimumAmount(Money minAmount);
    
    /**
     * 注文が存在するかどうかを確認
     * 
     * @param orderId 確認する注文のID
     * @return 存在する場合はtrue
     */
    boolean existsById(OrderId orderId);
    
    /**
     * ユーザーの注文数を取得
     * 
     * @param userId ユーザーID
     * @return 注文数
     */
    long countByUserId(UserId userId);
    
    /**
     * 期間内の注文数を取得
     * 
     * @param startDate 開始日時
     * @param endDate 終了日時
     * @return 注文数
     */
    long countByDateRange(LocalDateTime startDate, LocalDateTime endDate);
    
    /**
     * 注文の総売上を取得
     * 
     * @param startDate 開始日時
     * @param endDate 終了日時
     * @return 総売上
     */
    Money calculateTotalSales(LocalDateTime startDate, LocalDateTime endDate);
}
```

## 🔍 高度なクエリインターフェース

### UserQueryService インターフェース

```java
package com.example.domain.repository;

import com.example.domain.entity.User;
import com.example.domain.valueobject.UserId;

import java.util.List;

/**
 * ユーザー検索の高度なクエリインターフェース
 * 
 * 複雑な検索条件や集計クエリを
 * ドメイン層で抽象化します。
 */
public interface UserQueryService {
    
    /**
     * 複数条件でのユーザー検索
     */
    List<User> searchUsers(UserSearchCriteria criteria);
    
    /**
     * ユーザー統計情報を取得
     */
    UserStatistics getUserStatistics();
    
    /**
     * 類似ユーザーを検索
     */
    List<User> findSimilarUsers(UserId userId, int limit);
    
    /**
     * 重複リスクの高いユーザーを検索
     */
    List<User> findHighRiskUsers(double riskThreshold);
}
```

### UserSearchCriteria 値オブジェクト

```java
package com.example.domain.repository;

import com.example.domain.valueobject.Email;

import java.time.LocalDateTime;
import java.util.Optional;

/**
 * ユーザー検索条件を表現する値オブジェクト
 */
public record UserSearchCriteria(
    Optional<String> name,
    Optional<String> emailDomain,
    Optional<UserStatus> status,
    Optional<LocalDateTime> createdAfter,
    Optional<LocalDateTime> createdBefore,
    Optional<Integer> limit,
    Optional<Integer> offset
) {
    
    /**
     * デフォルトの検索条件を作成
     */
    public static UserSearchCriteria defaultCriteria() {
        return new UserSearchCriteria(
            Optional.empty(),
            Optional.empty(),
            Optional.empty(),
            Optional.empty(),
            Optional.empty(),
            Optional.of(100),
            Optional.of(0)
        );
    }
    
    /**
     * 名前で検索する条件を作成
     */
    public static UserSearchCriteria byName(String name) {
        return new UserSearchCriteria(
            Optional.of(name),
            Optional.empty(),
            Optional.empty(),
            Optional.empty(),
            Optional.empty(),
            Optional.of(100),
            Optional.of(0)
        );
    }
    
    /**
     * ドメインで検索する条件を作成
     */
    public static UserSearchCriteria byEmailDomain(String domain) {
        return new UserSearchCriteria(
            Optional.empty(),
            Optional.of(domain),
            Optional.empty(),
            Optional.empty(),
            Optional.empty(),
            Optional.of(100),
            Optional.of(0)
        );
    }
    
    /**
     * ステータスで検索する条件を作成
     */
    public static UserSearchCriteria byStatus(UserStatus status) {
        return new UserSearchCriteria(
            Optional.empty(),
            Optional.empty(),
            Optional.of(status),
            Optional.empty(),
            Optional.empty(),
            Optional.of(100),
            Optional.of(0)
        );
    }
}
```

### UserStatistics 値オブジェクト

```java
package com.example.domain.repository;

import java.time.LocalDateTime;

/**
 * ユーザー統計情報を表現する値オブジェクト
 */
public record UserStatistics(
    long totalUsers,
    long activeUsers,
    long inactiveUsers,
    long newUsersThisMonth,
    double averageAge,
    LocalDateTime lastUpdated
) {
    
    /**
     * アクティブ率を計算
     */
    public double getActiveRate() {
        if (totalUsers == 0) {
            return 0.0;
        }
        return (double) activeUsers / totalUsers;
    }
    
    /**
     * 今月の新規ユーザー率を計算
     */
    public double getNewUserRate() {
        if (totalUsers == 0) {
            return 0.0;
        }
        return (double) newUsersThisMonth / totalUsers;
    }
}
```

## 🎨 リポジトリの設計指針

### 1. **集約単位での操作**
- 集約ルートを中心とした操作
- 集約内の整合性を保証
- トランザクション境界の明確化

### 2. **ドメイン層でのインターフェース定義**
- インフラストラクチャ層に依存しない
- ドメインの意図を表現
- テストの容易性

### 3. **クエリの抽象化**
- 複雑なクエリロジックのカプセル化
- ドメインに特化したクエリメソッド
- パフォーマンスの最適化

### 4. **型安全性の確保**
- 強力な型システムを活用
- コンパイル時のエラー検出
- 実行時エラーの最小化

## 🧪 リポジトリのテスト

```groovy
package com.example.domain.repository

import com.example.domain.entity.User
import com.example.domain.valueobject.Email
import com.example.domain.valueobject.FullName
import spock.lang.Specification

class UserRepositoryTest extends Specification {
    
    def "ユーザーを保存できる"() {
        given:
        def repository = Mock(UserRepository)
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        
        when:
        def savedUser = repository.save(user)
        
        then:
        1 * repository.save(user) >> user
        savedUser == user
    }
    
    def "IDでユーザーを検索できる"() {
        given:
        def repository = Mock(UserRepository)
        def userId = UserId.generate()
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        
        when:
        def foundUser = repository.findById(userId)
        
        then:
        1 * repository.findById(userId) >> Optional.of(user)
        foundUser.isPresent()
        foundUser.get() == user
    }
    
    def "メールアドレスでユーザーを検索できる"() {
        given:
        def repository = Mock(UserRepository)
        def email = new Email("test@example.com")
        def user = User.create(email, new FullName("田中", "太郎"))
        
        when:
        def foundUser = repository.findByEmail(email)
        
        then:
        1 * repository.findByEmail(email) >> Optional.of(user)
        foundUser.isPresent()
        foundUser.get() == user
    }
    
    def "アクティブなユーザーのみを取得できる"() {
        given:
        def repository = Mock(UserRepository)
        def activeUsers = [
            User.create(new Email("user1@example.com"), new FullName("田中", "太郎")),
            User.create(new Email("user2@example.com"), new FullName("佐藤", "次郎"))
        ]
        
        when:
        def users = repository.findActiveUsers()
        
        then:
        1 * repository.findActiveUsers() >> activeUsers
        users.size() == 2
        users.every { it.isActive() }
    }
    
    def "特定のドメインのユーザーを検索できる"() {
        given:
        def repository = Mock(UserRepository)
        def domain = "example.com"
        def domainUsers = [
            User.create(new Email("user1@example.com"), new FullName("田中", "太郎")),
            User.create(new Email("user2@example.com"), new FullName("佐藤", "次郎"))
        ]
        
        when:
        def users = repository.findByEmailDomain(domain)
        
        then:
        1 * repository.findByEmailDomain(domain) >> domainUsers
        users.size() == 2
        users.every { it.getEmail().getDomain() == domain }
    }
}
```

## 🔄 次のステップ

リポジトリインターフェースの実装が完了したら、次はドメインイベントの実装に進みます。ドメインイベントは、重要な状態変化を他のオブジェクトに通知する重要なパターンです。 