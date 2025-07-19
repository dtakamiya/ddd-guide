## 4. ドメイン層の実装

ドメイン層は、アプリケーションの心臓部であり、ビジネスの核心的なロジックとルールを担当します。この章では、DDDの戦術的設計パターンを使用して、堅牢で保守性の高いドメインモデルを実装する方法を学びます。

### 4.1. ドメインモデリングの実践的アプローチ

#### 4.1.1. ドメインモデリングのプロセス

効果的なドメインモデリングには、以下のステップが重要です：

**1. ドメイン知識の収集**
*   ビジネス担当者とのインタビュー
*   既存システムの分析
*   ドキュメントの調査

**2. ユビキタス言語の確立**
*   ビジネス用語の整理
*   技術用語との対応付け
*   用語集の作成

**3. 境界づけられたコンテキストの識別**
*   ビジネス機能の分析
*   データの依存関係の調査
*   チーム構造の考慮

**4. 戦術的設計パターンの適用**
*   エンティティと値オブジェクトの識別
*   集約境界の決定
*   ドメインサービスの設計

#### 4.1.2. モデリングの原則

**1. ドメインの深い理解**
*   ビジネスルールの正確な把握
*   制約条件の明確化
*   例外ケースの考慮

**2. シンプルさの追求**
*   過度に複雑なモデルを避ける
*   実際のビジネスニーズに基づく設計
*   不要な抽象化を避ける

**3. 一貫性の維持**
*   ユビキタス言語の一貫した使用
*   設計パターンの統一
*   コーディング規約の遵守

### 4.2. 値オブジェクト（Value Objects）の実装

値オブジェクトは、Java 17の`record`型を使用することで、より簡潔で安全な実装が可能になります。

#### 4.2.1. 基本的な値オブジェクト

**Email.java:**
```java
package com.example.dddspanner.domain.valueobject;

import java.util.Objects;

/**
 * メールアドレスを表現する値オブジェクト
 * 
 * 値オブジェクトの特徴：
 * - 不変（Immutable）
 * - 値による等価性
 * - 自己完結した検証ロジック
 */
public record Email(String value) {
    
    public Email {
        // コンパクトコンストラクタで検証を実行
        if (value == null || value.trim().isEmpty()) {
            throw new IllegalArgumentException("Email cannot be null or empty");
        }
        
        if (!isValidEmailFormat(value)) {
            throw new IllegalArgumentException("Invalid email format: " + value);
        }
        
        // 正規化（小文字に統一）
        value = value.trim().toLowerCase();
    }
    
    /**
     * メールアドレスの形式を検証
     * 
     * 実用的な実装では、より厳密な正規表現や
     * ライブラリ（Apache Commons Validator等）を使用することを推奨
     */
    private static boolean isValidEmailFormat(String email) {
        if (email == null) return false;
        
        // 基本的な形式チェック
        String emailRegex = "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$";
        return email.matches(emailRegex);
    }
    
    /**
     * ドメイン部分を取得
     */
    public String getDomain() {
        int atIndex = value.indexOf('@');
        return atIndex > 0 ? value.substring(atIndex + 1) : "";
    }
    
    /**
     * ローカル部分を取得
     */
    public String getLocalPart() {
        int atIndex = value.indexOf('@');
        return atIndex > 0 ? value.substring(0, atIndex) : value;
    }
    
    @Override
    public String toString() {
        return value;
    }
}
```

**FullName.java:**
```java
package com.example.dddspanner.domain.valueobject;

/**
 * 氏名を表現する値オブジェクト
 * 
 * ビジネスルール：
 * - 姓と名は必須
 * - 最大50文字まで
 * - 特殊文字は許可しない
 */
public record FullName(String firstName, String lastName) {
    
    public FullName {
        validateName(firstName, "firstName");
        validateName(lastName, "lastName");
        
        // 正規化（前後の空白を除去）
        firstName = firstName.trim();
        lastName = lastName.trim();
    }
    
    private static void validateName(String name, String fieldName) {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException(fieldName + " cannot be null or empty");
        }
        
        if (name.length() > 50) {
            throw new IllegalArgumentException(fieldName + " cannot exceed 50 characters");
        }
        
        // 特殊文字をチェック（基本的な実装）
        if (!name.matches("^[a-zA-Z\\s\\u3040-\\u309F\\u30A0-\\u30FF\\u4E00-\\u9FAF]+$")) {
            throw new IllegalArgumentException(fieldName + " contains invalid characters");
        }
    }
    
    /**
     * フルネームを取得（姓 + 名）
     */
    public String getFullName() {
        return lastName + " " + firstName;
    }
    
    /**
     * 英語表記のフルネームを取得（名 + 姓）
     */
    public String getEnglishFullName() {
        return firstName + " " + lastName;
    }
    
    @Override
    public String toString() {
        return getFullName();
    }
}
```

#### 4.2.2. 複雑な値オブジェクト

**Money.java:**
```java
package com.example.dddspanner.domain.valueobject;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;

/**
 * 金額を表現する値オブジェクト
 * 
 * ビジネスルール：
 * - 金額は正の数
 * - 通貨は必須
 * - 計算はBigDecimalで行う
 */
public record Money(BigDecimal amount, Currency currency) {
    
    public Money {
        if (amount == null) {
            throw new IllegalArgumentException("Amount cannot be null");
        }
        if (currency == null) {
            throw new IllegalArgumentException("Currency cannot be null");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        
        // 小数点以下2桁に丸める
        amount = amount.setScale(2, RoundingMode.HALF_UP);
    }
    
    /**
     * ゼロ金額を作成
     */
    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }
    
    /**
     * 加算
     */
    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    
    /**
     * 減算
     */
    public Money subtract(Money other) {
        validateSameCurrency(other);
        BigDecimal result = amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Result cannot be negative");
        }
        return new Money(result, currency);
    }
    
    /**
     * 乗算
     */
    public Money multiply(BigDecimal factor) {
        if (factor == null || factor.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Factor must be positive");
        }
        return new Money(amount.multiply(factor), currency);
    }
    
    /**
     * 割合計算
     */
    public Money percentage(BigDecimal percentage) {
        if (percentage == null || percentage.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Percentage must be non-negative");
        }
        return multiply(percentage.divide(BigDecimal.valueOf(100), 4, RoundingMode.HALF_UP));
    }
    
    /**
     * 通貨が同じかチェック
     */
    private void validateSameCurrency(Money other) {
        if (!Objects.equals(currency, other.currency)) {
            throw new IllegalArgumentException("Cannot operate on different currencies: " + 
                currency + " and " + other.currency);
        }
    }
    
    /**
     * 金額がゼロかチェック
     */
    public boolean isZero() {
        return amount.compareTo(BigDecimal.ZERO) == 0;
    }
    
    /**
     * 金額が指定値より大きいかチェック
     */
    public boolean isGreaterThan(Money other) {
        validateSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }
    
    @Override
    public String toString() {
        return currency.getSymbol() + amount.toString();
    }
}
```

#### 4.2.3. 値オブジェクトの設計指針

**1. 不変性の保証**
*   `record`型を使用して自動的に不変性を保証
*   コンパクトコンストラクタで検証を実行
*   副作用のないメソッドのみを提供

**2. 値による等価性**
*   `record`型が自動的に`equals`と`hashCode`を生成
*   必要に応じてカスタム実装を追加

**3. ドメインロジックの集約**
*   関連するビジネスルールを値オブジェクト内に集約
*   外部からの不正な状態を防ぐ

**4. 自己完結した検証**
*   作成時に必要な検証をすべて実行
*   無効な状態のオブジェクトを作成不可能にする

### 4.3. エンティティ（Entities）の実装

エンティティは、識別子を持ち、時間とともに変化するオブジェクトです。

#### 4.3.1. 識別子の実装

**UserId.java:**
```java
package com.example.dddspanner.domain.valueobject;

import java.util.UUID;

/**
 * ユーザーIDを表現する値オブジェクト
 * 
 * 識別子の特徴：
 * - 一意性
 * - 不変性
 * - 型安全性
 */
public record UserId(String value) {
    
    public UserId {
        if (value == null || value.trim().isEmpty()) {
            throw new IllegalArgumentException("User ID cannot be null or empty");
        }
        
        // UUID形式の検証
        try {
            UUID.fromString(value);
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException("Invalid UUID format: " + value);
        }
    }
    
    /**
     * 新しいユーザーIDを生成
     */
    public static UserId generate() {
        return new UserId(UUID.randomUUID().toString());
    }
    
    /**
     * 文字列からユーザーIDを作成
     */
    public static UserId of(String value) {
        return new UserId(value);
    }
    
    @Override
    public String toString() {
        return value;
    }
}
```

#### 4.3.2. エンティティの実装

**User.java:**
```java
package com.example.dddspanner.domain.entity;

import com.example.dddspanner.domain.valueobject.*;
import lombok.Getter;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * ユーザーエンティティ
 * 
 * 集約ルートとしての役割：
 * - 外部からのアクセスを制御
 * - 一貫性の保証
 * - ドメインイベントの発行
 */
@Getter
public class User {
    
    private final UserId id;
    private FullName fullName;
    private Email email;
    private UserStatus status;
    private Instant createdAt;
    private Instant updatedAt;
    
    // ドメインイベントのリスト
    private final List<DomainEvent> domainEvents = new ArrayList<>();
    
    /**
     * プライベートコンストラクタ
     * ファクトリメソッド経由での作成を強制
     */
    private User(UserId id, FullName fullName, Email email) {
        this.id = id;
        this.fullName = fullName;
        this.email = email;
        this.status = UserStatus.ACTIVE;
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }
    
    /**
     * ファクトリメソッド：新規ユーザー作成
     */
    public static User create(FullName fullName, Email email) {
        UserId id = UserId.generate();
        User user = new User(id, fullName, email);
        
        // ドメインイベントを発行
        user.addDomainEvent(new UserCreatedEvent(
            user.getId(),
            user.getFullName(),
            user.getEmail(),
            Instant.now()
        ));
        
        return user;
    }
    
    /**
     * ファクトリメソッド：既存ユーザーの再構築
     */
    public static User reconstruct(
            UserId id, 
            FullName fullName, 
            Email email, 
            UserStatus status,
            Instant createdAt,
            Instant updatedAt) {
        User user = new User(id, fullName, email);
        user.status = status;
        user.createdAt = createdAt;
        user.updatedAt = updatedAt;
        return user;
    }
    
    /**
     * 名前の変更
     */
    public void changeName(FullName newFullName) {
        if (status != UserStatus.ACTIVE) {
            throw new IllegalStateException("Cannot change name of inactive user");
        }
        
        this.fullName = newFullName;
        this.updatedAt = Instant.now();
        
        // ドメインイベントを発行
        addDomainEvent(new UserNameChangedEvent(
            this.getId(),
            this.getFullName(),
            Instant.now()
        ));
    }
    
    /**
     * メールアドレスの変更
     */
    public void changeEmail(Email newEmail) {
        if (status != UserStatus.ACTIVE) {
            throw new IllegalStateException("Cannot change email of inactive user");
        }
        
        this.email = newEmail;
        this.updatedAt = Instant.now();
        
        // ドメインイベントを発行
        addDomainEvent(new UserEmailChangedEvent(
            this.getId(),
            this.getEmail(),
            Instant.now()
        ));
    }
    
    /**
     * ユーザーの無効化
     */
    public void deactivate() {
        if (status == UserStatus.INACTIVE) {
            throw new IllegalStateException("User is already inactive");
        }
        
        this.status = UserStatus.INACTIVE;
        this.updatedAt = Instant.now();
        
        // ドメインイベントを発行
        addDomainEvent(new UserDeactivatedEvent(
            this.getId(),
            Instant.now()
        ));
    }
    
    /**
     * ユーザーの再有効化
     */
    public void reactivate() {
        if (status == UserStatus.ACTIVE) {
            throw new IllegalStateException("User is already active");
        }
        
        this.status = UserStatus.ACTIVE;
        this.updatedAt = Instant.now();
        
        // ドメインイベントを発行
        addDomainEvent(new UserReactivatedEvent(
            this.getId(),
            Instant.now()
        ));
    }
    
    /**
     * ドメインイベントの追加
     */
    private void addDomainEvent(DomainEvent event) {
        domainEvents.add(event);
    }
    
    /**
     * ドメインイベントの取得とクリア
     */
    public List<DomainEvent> getDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
    
    /**
     * 同一性の判定
     */
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        User user = (User) obj;
        return Objects.equals(id, user.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
    
    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", fullName=" + fullName +
                ", email=" + email +
                ", status=" + status +
                '}';
    }
}
```

#### 4.3.3. 列挙型の実装

**UserStatus.java:**
```java
package com.example.dddspanner.domain.valueobject;

/**
 * ユーザーステータスを表現する列挙型
 * 
 * ビジネスルール：
 * - ACTIVE: アクティブなユーザー
 * - INACTIVE: 無効化されたユーザー
 * - SUSPENDED: 一時停止されたユーザー
 */
public enum UserStatus {
    ACTIVE("アクティブ"),
    INACTIVE("無効"),
    SUSPENDED("一時停止");
    
    private final String displayName;
    
    UserStatus(String displayName) {
        this.displayName = displayName;
    }
    
    public String getDisplayName() {
        return displayName;
    }
    
    /**
     * アクティブかどうかを判定
     */
    public boolean isActive() {
        return this == ACTIVE;
    }
    
    /**
     * 操作可能かどうかを判定
     */
    public boolean canOperate() {
        return this == ACTIVE;
    }
}
```

### 4.4. 集約（Aggregates）の設計

#### 4.4.1. 集約設計の原則

**1. 一貫性の境界**
*   集約内のオブジェクトは常に一貫した状態を保つ
*   外部からの直接アクセスを禁止
*   集約ルート経由でのみ操作を許可

**2. 集約のサイズ**
*   小さすぎる集約：パフォーマンスの問題
*   大きすぎる集約：一貫性の問題
*   ビジネスルールに基づいて適切なサイズを決定

**3. 集約間の関係**
*   集約間はID参照のみ
*   直接的なオブジェクト参照は避ける
*   イベントによる疎結合

#### 4.4.2. 集約の実装例

**Order.java（集約ルート）:**
```java
package com.example.dddspanner.domain.entity;

import com.example.dddspanner.domain.valueobject.*;
import lombok.Getter;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * 注文集約のルートエンティティ
 * 
 * 集約の特徴：
 * - 外部からのアクセスを制御
 * - 一貫性の保証
 * - トランザクション境界
 */
@Getter
public class Order {
    
    private final OrderId id;
    private final UserId userId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    private Instant createdAt;
    private Instant updatedAt;
    
    private final List<DomainEvent> domainEvents = new ArrayList<>();
    
    private Order(OrderId id, UserId userId) {
        this.id = id;
        this.userId = userId;
        this.items = new ArrayList<>();
        this.status = OrderStatus.DRAFT;
        this.totalAmount = Money.zero(Currency.getInstance("JPY"));
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }
    
    /**
     * ファクトリメソッド：新規注文作成
     */
    public static Order create(UserId userId) {
        OrderId id = OrderId.generate();
        Order order = new Order(id, userId);
        
        order.addDomainEvent(new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            Instant.now()
        ));
        
        return order;
    }
    
    /**
     * 商品の追加
     */
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        
        if (product == null) {
            throw new IllegalArgumentException("Product cannot be null");
        }
        
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        
        // 既存の商品があるかチェック
        OrderItem existingItem = findItemByProduct(product.getId());
        if (existingItem != null) {
            // 数量を更新
            existingItem.updateQuantity(existingItem.getQuantity() + quantity);
        } else {
            // 新しい商品を追加
            OrderItem newItem = OrderItem.create(product, quantity);
            items.add(newItem);
        }
        
        updateTotalAmount();
        this.updatedAt = Instant.now();
    }
    
    /**
     * 商品の削除
     */
    public void removeItem(ProductId productId) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        
        items.removeIf(item -> item.getProductId().equals(productId));
        updateTotalAmount();
        this.updatedAt = Instant.now();
    }
    
    /**
     * 商品の数量変更
     */
    public void updateItemQuantity(ProductId productId, int newQuantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        
        if (newQuantity <= 0) {
            removeItem(productId);
            return;
        }
        
        OrderItem item = findItemByProductId(productId);
        if (item == null) {
            throw new IllegalArgumentException("Item not found: " + productId);
        }
        
        item.updateQuantity(newQuantity);
        updateTotalAmount();
        this.updatedAt = Instant.now();
    }
    
    /**
     * 注文の確定
     */
    public void confirm() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Order is not in draft status");
        }
        
        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = Instant.now();
        
        addDomainEvent(new OrderConfirmedEvent(
            this.getId(),
            this.getUserId(),
            this.getTotalAmount(),
            Instant.now()
        ));
    }
    
    /**
     * 注文のキャンセル
     */
    public void cancel() {
        if (status == OrderStatus.CANCELLED) {
            throw new IllegalStateException("Order is already cancelled");
        }
        
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("Cannot cancel shipped order");
        }
        
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = Instant.now();
        
        addDomainEvent(new OrderCancelledEvent(
            this.getId(),
            this.getUserId(),
            Instant.now()
        ));
    }
    
    /**
     * 商品IDで商品を検索
     */
    private OrderItem findItemByProduct(ProductId productId) {
        return items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst()
            .orElse(null);
    }
    
    /**
     * 商品IDで商品を検索
     */
    private OrderItem findItemByProductId(ProductId productId) {
        return items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst()
            .orElse(null);
    }
    
    /**
     * 合計金額の更新
     */
    private void updateTotalAmount() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.zero(totalAmount.getCurrency()), Money::add);
    }
    
    /**
     * 商品リストの取得（読み取り専用）
     */
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
    
    /**
     * 商品数を取得
     */
    public int getItemCount() {
        return items.size();
    }
    
    /**
     * ドメインイベントの追加
     */
    private void addDomainEvent(DomainEvent event) {
        domainEvents.add(event);
    }
    
    /**
     * ドメインイベントの取得とクリア
     */
    public List<DomainEvent> getDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Order order = (Order) obj;
        return Objects.equals(id, order.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

### 4.5. ドメインサービス（Domain Services）の実装

#### 4.5.1. ドメインサービスの役割

ドメインサービスは、以下の場合に使用されます：

*   複数の集約にまたがるロジック
*   エンティティや値オブジェクトに属さないビジネスロジック
*   外部サービスとの連携が必要なロジック

#### 4.5.2. ドメインサービスの実装例

**UserDuplicateChecker.java:**
```java
package com.example.dddspanner.domain.service;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.repository.UserRepository;
import org.springframework.stereotype.Service;

/**
 * ユーザー重複チェックサービス
 * 
 * ドメインサービスの特徴：
 * - 複数の集約にまたがるロジック
 * - 副作用がない
 * - ドメイン層に配置
 */
@Service
public class UserDuplicateChecker {
    
    private final UserRepository userRepository;
    
    public UserDuplicateChecker(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    /**
     * メールアドレスの重複をチェック
     */
    public boolean isEmailDuplicate(Email email) {
        return userRepository.findByEmail(email).isPresent();
    }
    
    /**
     * ユーザー作成時の重複チェック
     */
    public void validateUserCreation(Email email) {
        if (isEmailDuplicate(email)) {
            throw new IllegalArgumentException("Email already exists: " + email);
        }
    }
    
    /**
     * メールアドレス変更時の重複チェック
     */
    public void validateEmailChange(Email newEmail, UserId currentUserId) {
        var existingUser = userRepository.findByEmail(newEmail);
        if (existingUser.isPresent() && !existingUser.get().getId().equals(currentUserId)) {
            throw new IllegalArgumentException("Email already exists: " + newEmail);
        }
    }
}
```

**OrderCalculationService.java:**
```java
package com.example.dddspanner.domain.service;

import com.example.dddspanner.domain.entity.Order;
import com.example.dddspanner.domain.valueobject.Money;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

/**
 * 注文計算サービス
 * 
 * 複雑な計算ロジックを集約から分離
 */
@Service
public class OrderCalculationService {
    
    /**
     * 注文の合計金額を計算
     */
    public Money calculateTotal(Order order) {
        return order.getItems().stream()
            .map(item -> item.getSubtotal())
            .reduce(Money.zero(order.getTotalAmount().getCurrency()), Money::add);
    }
    
    /**
     * 税込み金額を計算
     */
    public Money calculateTaxIncluded(Order order, BigDecimal taxRate) {
        Money subtotal = calculateTotal(order);
        return subtotal.add(subtotal.percentage(taxRate));
    }
    
    /**
     * 割引後の金額を計算
     */
    public Money calculateDiscounted(Order order, BigDecimal discountRate) {
        Money subtotal = calculateTotal(order);
        return subtotal.subtract(subtotal.percentage(discountRate));
    }
}
```

### 4.6. リポジトリインターフェースの定義

#### 4.6.1. 基本的なリポジトリインターフェース

**UserRepository.java:**
```java
package com.example.dddspanner.domain.repository;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.valueobject.UserId;

import java.util.List;
import java.util.Optional;

/**
 * ユーザーリポジトリインターフェース
 * 
 * リポジトリの特徴：
 * - ドメイン層に定義
 * - 集約単位での操作
 * - インフラストラクチャ層で実装
 */
public interface UserRepository {
    
    /**
     * ユーザーを保存
     */
    User save(User user);
    
    /**
     * IDでユーザーを検索
     */
    Optional<User> findById(UserId id);
    
    /**
     * メールアドレスでユーザーを検索
     */
    Optional<User> findByEmail(Email email);
    
    /**
     * 全ユーザーを取得
     */
    List<User> findAll();
    
    /**
     * アクティブなユーザーのみを取得
     */
    List<User> findActiveUsers();
    
    /**
     * ユーザーを削除
     */
    void delete(UserId id);
    
    /**
     * ユーザーが存在するかチェック
     */
    boolean existsById(UserId id);
    
    /**
     * メールアドレスが存在するかチェック
     */
    boolean existsByEmail(Email email);
}
```

#### 4.6.2. 高度なリポジトリインターフェース

**OrderRepository.java:**
```java
package com.example.dddspanner.domain.repository;

import com.example.dddspanner.domain.entity.Order;
import com.example.dddspanner.domain.valueobject.OrderId;
import com.example.dddspanner.domain.valueobject.UserId;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

/**
 * 注文リポジトリインターフェース
 * 
 * 複雑なクエリ要件に対応
 */
public interface OrderRepository {
    
    /**
     * 注文を保存
     */
    Order save(Order order);
    
    /**
     * IDで注文を検索
     */
    Optional<Order> findById(OrderId id);
    
    /**
     * ユーザーの注文を取得
     */
    List<Order> findByUserId(UserId userId);
    
    /**
     * ユーザーの確定済み注文を取得
     */
    List<Order> findConfirmedOrdersByUserId(UserId userId);
    
    /**
     * 期間内の注文を取得
     */
    List<Order> findByCreatedAtBetween(Instant from, Instant to);
    
    /**
     * ステータス別の注文を取得
     */
    List<Order> findByStatus(OrderStatus status);
    
    /**
     * 高額注文を取得（指定金額以上）
     */
    List<Order> findHighValueOrders(Money minimumAmount);
    
    /**
     * 注文を削除
     */
    void delete(OrderId id);
    
    /**
     * 注文が存在するかチェック
     */
    boolean existsById(OrderId id);
}
```

### 4.7. ドメインイベントの実装

#### 4.7.1. ドメインイベントの基本構造

**DomainEvent.java:**
```java
package com.example.dddspanner.domain.event;

import java.time.Instant;

/**
 * ドメインイベントの基底インターフェース
 * 
 * ドメインイベントの特徴：
 * - 過去形で命名
 * - 不変オブジェクト
 * - 発生時刻を持つ
 */
public interface DomainEvent {
    
    /**
     * イベントの発生時刻を取得
     */
    Instant getOccurredAt();
    
    /**
     * イベントの種類を取得
     */
    String getEventType();
}
```

#### 4.7.2. 具体的なドメインイベント

**UserCreatedEvent.java:**
```java
package com.example.dddspanner.domain.event;

import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.valueobject.FullName;
import com.example.dddspanner.domain.valueobject.UserId;

import java.time.Instant;

/**
 * ユーザー作成イベント
 */
public record UserCreatedEvent(
    UserId userId,
    FullName fullName,
    Email email,
    Instant occurredAt
) implements DomainEvent {
    
    @Override
    public String getEventType() {
        return "UserCreated";
    }
}
```

**OrderConfirmedEvent.java:**
```java
package com.example.dddspanner.domain.event;

import com.example.dddspanner.domain.valueobject.Money;
import com.example.dddspanner.domain.valueobject.OrderId;
import com.example.dddspanner.domain.valueobject.UserId;

import java.time.Instant;

/**
 * 注文確定イベント
 */
public record OrderConfirmedEvent(
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    Instant occurredAt
) implements DomainEvent {
    
    @Override
    public String getEventType() {
        return "OrderConfirmed";
    }
}
```

### 4.8. ファクトリパターンの実装

#### 4.8.1. ファクトリの役割

ファクトリは、複雑なオブジェクトの生成ロジックをカプセル化します：

*   オブジェクト生成の複雑性を隠蔽
*   不整合な状態のオブジェクト作成を防止
*   生成ロジックの再利用

#### 4.8.2. ファクトリの実装例

**UserFactory.java:**
```java
package com.example.dddspanner.domain.factory;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.service.UserDuplicateChecker;
import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.valueobject.FullName;
import org.springframework.stereotype.Component;

/**
 * ユーザーファクトリ
 * 
 * 複雑なユーザー作成ロジックをカプセル化
 */
@Component
public class UserFactory {
    
    private final UserDuplicateChecker duplicateChecker;
    
    public UserFactory(UserDuplicateChecker duplicateChecker) {
        this.duplicateChecker = duplicateChecker;
    }
    
    /**
     * 新規ユーザーを作成
     */
    public User createUser(FullName fullName, Email email) {
        // 重複チェック
        duplicateChecker.validateUserCreation(email);
        
        // ユーザー作成
        return User.create(fullName, email);
    }
    
    /**
     * 管理者ユーザーを作成
     */
    public User createAdminUser(FullName fullName, Email email) {
        User user = createUser(fullName, email);
        // 管理者特有の設定を追加
        return user;
    }
    
    /**
     * ゲストユーザーを作成
     */
    public User createGuestUser(FullName fullName, Email email) {
        User user = createUser(fullName, email);
        // ゲスト特有の設定を追加
        return user;
    }
}
```

### 4.9. ドメイン層のテスト

#### 4.9.1. 値オブジェクトのテスト

**EmailTest.groovy:**
```groovy
package com.example.dddspanner.domain.valueobject

import spock.lang.Specification
import spock.lang.Unroll

class EmailTest extends Specification {
    
    @Unroll
    def "should create valid email: #email"() {
        when:
        def emailObj = new Email(email)
        
        then:
        emailObj.value() == email.toLowerCase()
        emailObj.getDomain() == expectedDomain
        emailObj.getLocalPart() == expectedLocalPart
        
        where:
        email                    | expectedDomain    | expectedLocalPart
        "test@example.com"      | "example.com"     | "test"
        "USER@DOMAIN.COM"       | "domain.com"      | "user"
        "test.email@sub.domain" | "sub.domain"      | "test.email"
    }
    
    def "should throw exception for invalid email: #email"() {
        when:
        new Email(email)
        
        then:
        thrown(IllegalArgumentException)
        
        where:
        email << [
            null,
            "",
            "invalid",
            "@example.com",
            "test@",
            "test..test@example.com"
        ]
    }
}
```

#### 4.9.2. エンティティのテスト

**UserTest.groovy:**
```groovy
package com.example.dddspanner.domain.entity

import com.example.dddspanner.domain.valueobject.Email
import com.example.dddspanner.domain.valueobject.FullName
import spock.lang.Specification

class UserTest extends Specification {
    
    def "should create user successfully"() {
        given:
        def fullName = new FullName("Taro", "Yamada")
        def email = new Email("taro.yamada@example.com")
        
        when:
        def user = User.create(fullName, email)
        
        then:
        user.getId() != null
        user.getFullName() == fullName
        user.getEmail() == email
        user.getStatus().isActive()
        user.getCreatedAt() != null
        user.getUpdatedAt() != null
        
        and: "domain event should be published"
        def events = user.getDomainEvents()
        events.size() == 1
        events[0].getEventType() == "UserCreated"
    }
    
    def "should change name successfully"() {
        given:
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        def newName = new FullName("Jiro", "Tanaka")
        
        when:
        user.changeName(newName)
        
        then:
        user.getFullName() == newName
        user.getUpdatedAt() > user.getCreatedAt()
        
        and: "domain event should be published"
        def events = user.getDomainEvents()
        events.size() == 2 // UserCreated + UserNameChanged
        events[1].getEventType() == "UserNameChanged"
    }
    
    def "should not allow name change for inactive user"() {
        given:
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        user.deactivate()
        
        when:
        user.changeName(new FullName("Jiro", "Tanaka"))
        
        then:
        thrown(IllegalStateException)
    }
}
```

### 4.10. まとめ

この章では、ドメイン層の実装について詳しく説明しました。重要なポイントをまとめます：

**1. 値オブジェクトの設計**
*   Java 17の`record`型を活用
*   不変性と値による等価性を保証
*   自己完結した検証ロジック

**2. エンティティの設計**
*   識別子による同一性の管理
*   ファクトリメソッドによる安全な作成
*   ドメインイベントの発行

**3. 集約の設計**
*   一貫性の境界の明確化
*   外部からのアクセス制御
*   適切なサイズの決定

**4. ドメインサービスの活用**
*   複数の集約にまたがるロジック
*   副作用のない設計
*   ドメイン層での配置

**5. テストの重要性**
*   ドメインロジックの検証
*   ビジネスルールの確認
*   境界条件のテスト

次の章では、これらのドメインオブジェクトを活用してアプリケーション層を実装していきます。 