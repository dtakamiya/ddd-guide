# 4.8 ファクトリパターン（Factory Pattern）の実装

ファクトリパターンは、複雑なオブジェクト生成をカプセル化し、不整合な状態の防止を実現する重要なパターンです。ドメイン駆動設計では、エンティティや集約の作成ロジックをファクトリに集約します。

## 🎯 ファクトリパターンの特徴

### 1. **複雑な作成ロジックのカプセル化**
- 複数のパラメータを組み合わせた作成
- ビジネスルールに基づく作成
- 作成ロジックの再利用

### 2. **不整合な状態の防止**
- 不正な状態のオブジェクト作成を防止
- バリデーションロジックの集約
- 整合性の保証

### 3. **ドメイン知識の集約**
- 作成に関するビジネスルールを集約
- ドメインエキスパートの知識を反映
- 一貫した作成ロジック

## 🏭 ユーザーファクトリ

### UserFactory クラス

```java
package com.example.domain.factory;

import com.example.domain.entity.User;
import com.example.domain.valueobject.Email;
import com.example.domain.valueobject.FullName;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;
import java.util.Objects;

/**
 * ユーザー作成を担当するファクトリクラス
 * 
 * ユーザーの作成ロジックをカプセル化し、
 * ビジネスルールに基づく安全な作成を保証します。
 */
public class UserFactory {
    
    /**
     * 基本的なユーザーを作成
     * 
     * @param email メールアドレス
     * @param fullName 氏名
     * @return 作成されたユーザー
     * @throws IllegalArgumentException 無効なパラメータの場合
     */
    public static User createUser(Email email, FullName fullName) {
        validateUserCreation(email, fullName);
        
        User user = new User();
        user.id = UserId.generate();
        user.email = email;
        user.fullName = fullName;
        user.status = UserStatus.ACTIVE;
        user.createdAt = LocalDateTime.now();
        user.updatedAt = LocalDateTime.now();
        
        // ドメインイベントを発行
        user.addDomainEvent(new UserCreatedEvent(user.id, user.email, user.fullName));
        
        return user;
    }
    
    /**
     * 管理者ユーザーを作成
     * 
     * @param email メールアドレス
     * @param fullName 氏名
     * @param adminLevel 管理者レベル
     * @return 作成された管理者ユーザー
     */
    public static User createAdminUser(Email email, FullName fullName, AdminLevel adminLevel) {
        validateUserCreation(email, fullName);
        Objects.requireNonNull(adminLevel, "管理者レベルは必須です");
        
        User user = createUser(email, fullName);
        user.adminLevel = adminLevel;
        user.permissions = adminLevel.getDefaultPermissions();
        
        return user;
    }
    
    /**
     * ゲストユーザーを作成
     * 
     * @param email メールアドレス
     * @param fullName 氏名
     * @return 作成されたゲストユーザー
     */
    public static User createGuestUser(Email email, FullName fullName) {
        validateUserCreation(email, fullName);
        
        User user = createUser(email, fullName);
        user.status = UserStatus.GUEST;
        user.permissions = List.of("READ_ONLY");
        
        return user;
    }
    
    /**
     * 既存ユーザーを復元
     * 
     * @param id ユーザーID
     * @param email メールアドレス
     * @param fullName 氏名
     * @param status ステータス
     * @param createdAt 作成日時
     * @param updatedAt 更新日時
     * @return 復元されたユーザー
     */
    public static User reconstructUser(
            UserId id, 
            Email email, 
            FullName fullName, 
            UserStatus status,
            LocalDateTime createdAt,
            LocalDateTime updatedAt) {
        
        Objects.requireNonNull(id, "ユーザーIDは必須です");
        Objects.requireNonNull(email, "メールアドレスは必須です");
        Objects.requireNonNull(fullName, "氏名は必須です");
        Objects.requireNonNull(status, "ステータスは必須です");
        Objects.requireNonNull(createdAt, "作成日時は必須です");
        Objects.requireNonNull(updatedAt, "更新日時は必須です");
        
        User user = new User();
        user.id = id;
        user.email = email;
        user.fullName = fullName;
        user.status = status;
        user.createdAt = createdAt;
        user.updatedAt = updatedAt;
        
        return user;
    }
    
    /**
     * ユーザー作成のバリデーション
     */
    private static void validateUserCreation(Email email, FullName fullName) {
        Objects.requireNonNull(email, "メールアドレスは必須です");
        Objects.requireNonNull(fullName, "氏名は必須です");
        
        // ビジネスルールの検証
        if (email.getDomain().equals("spam.com")) {
            throw new IllegalArgumentException("スパムドメインからのユーザー作成は許可されません");
        }
        
        if (fullName.getFullName().length() > 100) {
            throw new IllegalArgumentException("氏名が長すぎます（最大100文字）");
        }
    }
}
```

### AdminLevel 列挙型

```java
package com.example.domain.factory;

import java.util.List;

/**
 * 管理者レベルを表現する列挙型
 */
public enum AdminLevel {
    SUPER_ADMIN("スーパー管理者", List.of("ALL_PERMISSIONS")),
    ADMIN("管理者", List.of("USER_MANAGEMENT", "ORDER_MANAGEMENT", "REPORT_VIEW")),
    MODERATOR("モデレーター", List.of("USER_MODERATION", "CONTENT_MODERATION"));
    
    private final String displayName;
    private final List<String> defaultPermissions;
    
    AdminLevel(String displayName, List<String> defaultPermissions) {
        this.displayName = displayName;
        this.defaultPermissions = defaultPermissions;
    }
    
    public String getDisplayName() {
        return displayName;
    }
    
    public List<String> getDefaultPermissions() {
        return defaultPermissions;
    }
}
```

## 📦 注文ファクトリ

### OrderFactory クラス

```java
package com.example.domain.factory;

import com.example.domain.aggregate.Order;
import com.example.domain.entity.OrderItem;
import com.example.domain.valueobject.Money;
import com.example.domain.valueobject.OrderId;
import com.example.domain.valueobject.ProductId;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * 注文作成を担当するファクトリクラス
 * 
 * 注文の作成ロジックをカプセル化し、
 * ビジネスルールに基づく安全な作成を保証します。
 */
public class OrderFactory {
    
    /**
     * 基本的な注文を作成
     * 
     * @param userId ユーザーID
     * @param items 注文項目リスト
     * @return 作成された注文
     * @throws IllegalArgumentException 無効なパラメータの場合
     */
    public static Order createOrder(UserId userId, List<OrderItem> items) {
        validateOrderCreation(userId, items);
        
        Order order = new Order();
        order.id = OrderId.generate();
        order.userId = userId;
        order.items = new ArrayList<>(items);
        order.status = OrderStatus.PENDING;
        order.createdAt = LocalDateTime.now();
        order.updatedAt = LocalDateTime.now();
        
        // 合計金額を計算
        order.calculateTotalAmount();
        
        // ドメインイベントを発行
        order.addDomainEvent(new OrderCreatedEvent(order.id, order.userId, order.totalAmount, items.size()));
        
        return order;
    }
    
    /**
     * 単一商品の注文を作成
     * 
     * @param userId ユーザーID
     * @param productId 商品ID
     * @param productName 商品名
     * @param unitPrice 単価
     * @param quantity 数量
     * @return 作成された注文
     */
    public static Order createSingleItemOrder(
            UserId userId, 
            ProductId productId, 
            String productName, 
            Money unitPrice, 
            int quantity) {
        
        Objects.requireNonNull(productId, "商品IDは必須です");
        Objects.requireNonNull(productName, "商品名は必須です");
        Objects.requireNonNull(unitPrice, "単価は必須です");
        
        if (quantity <= 0) {
            throw new IllegalArgumentException("数量は1以上である必要があります");
        }
        
        OrderItem item = new OrderItem(productId, productName, unitPrice, quantity);
        return createOrder(userId, List.of(item));
    }
    
    /**
     * 既存注文を復元
     * 
     * @param id 注文ID
     * @param userId ユーザーID
     * @param items 注文項目リスト
     * @param totalAmount 合計金額
     * @param status ステータス
     * @param createdAt 作成日時
     * @param updatedAt 更新日時
     * @return 復元された注文
     */
    public static Order reconstructOrder(
            OrderId id,
            UserId userId,
            List<OrderItem> items,
            Money totalAmount,
            OrderStatus status,
            LocalDateTime createdAt,
            LocalDateTime updatedAt) {
        
        Objects.requireNonNull(id, "注文IDは必須です");
        Objects.requireNonNull(userId, "ユーザーIDは必須です");
        Objects.requireNonNull(items, "注文項目は必須です");
        Objects.requireNonNull(totalAmount, "合計金額は必須です");
        Objects.requireNonNull(status, "ステータスは必須です");
        Objects.requireNonNull(createdAt, "作成日時は必須です");
        Objects.requireNonNull(updatedAt, "更新日時は必須です");
        
        Order order = new Order();
        order.id = id;
        order.userId = userId;
        order.items = new ArrayList<>(items);
        order.totalAmount = totalAmount;
        order.status = status;
        order.createdAt = createdAt;
        order.updatedAt = updatedAt;
        
        return order;
    }
    
    /**
     * 注文作成のバリデーション
     */
    private static void validateOrderCreation(UserId userId, List<OrderItem> items) {
        Objects.requireNonNull(userId, "ユーザーIDは必須です");
        Objects.requireNonNull(items, "注文項目は必須です");
        
        if (items.isEmpty()) {
            throw new IllegalArgumentException("注文項目は必須です");
        }
        
        // 商品数の制限
        if (items.size() > 50) {
            throw new IllegalArgumentException("注文項目数が多すぎます（最大50個）");
        }
        
        // 合計金額の制限
        Money totalAmount = items.stream()
                .map(OrderItem::getTotalPrice)
                .reduce(Money.zero(items.get(0).getUnitPrice().currency()), Money::add);
        
        if (totalAmount.amount().compareTo(new java.math.BigDecimal("1000000")) > 0) {
            throw new IllegalArgumentException("注文金額が上限を超えています（最大100万円）");
        }
    }
}
```

## 🎨 ファクトリパターンの設計指針

### 1. **複雑な作成ロジックのカプセル化**
- 複数のパラメータを組み合わせた作成
- ビジネスルールに基づく作成
- 作成ロジックの再利用

### 2. **不整合な状態の防止**
- 不正な状態のオブジェクト作成を防止
- バリデーションロジックの集約
- 整合性の保証

### 3. **ドメイン知識の集約**
- 作成に関するビジネスルールを集約
- ドメインエキスパートの知識を反映
- 一貫した作成ロジック

### 4. **適切な粒度の設計**
- 単一責任の原則を守る
- 過度に複雑にしない
- テストしやすい設計

## 🧪 ファクトリのテスト

```groovy
package com.example.domain.factory

import com.example.domain.valueobject.Email
import com.example.domain.valueobject.FullName
import spock.lang.Specification

class UserFactoryTest extends Specification {
    
    def "基本的なユーザーを作成できる"() {
        given:
        def email = new Email("test@example.com")
        def fullName = new FullName("田中", "太郎")
        
        when:
        def user = UserFactory.createUser(email, fullName)
        
        then:
        user.id != null
        user.email == email
        user.fullName == fullName
        user.status == UserStatus.ACTIVE
        user.createdAt != null
        user.updatedAt != null
        
        and: "ドメインイベントが発行される"
        def events = user.getDomainEvents()
        events.size() == 1
        events[0] instanceof UserCreatedEvent
    }
    
    def "管理者ユーザーを作成できる"() {
        given:
        def email = new Email("admin@example.com")
        def fullName = new FullName("管理者", "太郎")
        def adminLevel = AdminLevel.ADMIN
        
        when:
        def user = UserFactory.createAdminUser(email, fullName, adminLevel)
        
        then:
        user.adminLevel == adminLevel
        user.permissions.contains("USER_MANAGEMENT")
        user.permissions.contains("ORDER_MANAGEMENT")
    }
    
    def "ゲストユーザーを作成できる"() {
        given:
        def email = new Email("guest@example.com")
        def fullName = new FullName("ゲスト", "太郎")
        
        when:
        def user = UserFactory.createGuestUser(email, fullName)
        
        then:
        user.status == UserStatus.GUEST
        user.permissions.contains("READ_ONLY")
    }
    
    def "スパムドメインからのユーザー作成で例外が発生する"() {
        given:
        def email = new Email("test@spam.com")
        def fullName = new FullName("田中", "太郎")
        
        when:
        UserFactory.createUser(email, fullName)
        
        then:
        thrown(IllegalArgumentException)
    }
    
    def "氏名が長すぎる場合に例外が発生する"() {
        given:
        def email = new Email("test@example.com")
        def longName = "A".repeat(101)
        def fullName = new FullName(longName, "太郎")
        
        when:
        UserFactory.createUser(email, fullName)
        
        then:
        thrown(IllegalArgumentException)
    }
}

class OrderFactoryTest extends Specification {
    
    def "基本的な注文を作成できる"() {
        given:
        def userId = UserId.generate()
        def items = [
            new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 2)
        ]
        
        when:
        def order = OrderFactory.createOrder(userId, items)
        
        then:
        order.id != null
        order.userId == userId
        order.items.size() == 1
        order.status == OrderStatus.PENDING
        order.totalAmount.amount() == new BigDecimal("2000")
        
        and: "ドメインイベントが発行される"
        def events = order.getDomainEvents()
        events.size() == 1
        events[0] instanceof OrderCreatedEvent
    }
    
    def "単一商品の注文を作成できる"() {
        given:
        def userId = UserId.generate()
        def productId = ProductId.generate()
        def productName = "商品A"
        def unitPrice = Money.of("1000", Currency.getInstance("JPY"))
        def quantity = 3
        
        when:
        def order = OrderFactory.createSingleItemOrder(userId, productId, productName, unitPrice, quantity)
        
        then:
        order.items.size() == 1
        order.items[0].productId == productId
        order.items[0].productName == productName
        order.items[0].quantity == quantity
        order.totalAmount.amount() == new BigDecimal("3000")
    }
    
    def "空の注文項目で例外が発生する"() {
        given:
        def userId = UserId.generate()
        def items = []
        
        when:
        OrderFactory.createOrder(userId, items)
        
        then:
        thrown(IllegalArgumentException)
    }
    
    def "注文項目数が多すぎる場合に例外が発生する"() {
        given:
        def userId = UserId.generate()
        def items = (1..51).collect { i ->
            new OrderItem(ProductId.generate(), "商品${i}", Money.of("100", Currency.getInstance("JPY")), 1)
        }
        
        when:
        OrderFactory.createOrder(userId, items)
        
        then:
        thrown(IllegalArgumentException)
    }
}
```

## 🔄 次のステップ

ファクトリパターンの実装が完了したら、次はドメイン層のテストの実装に進みます。ドメイン層のテストは、ビジネスルールの正確性を保証する重要な要素です。 