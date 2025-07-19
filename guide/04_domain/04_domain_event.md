# 4.7 ドメインイベント（Domain Events）の実装

ドメインイベントは、ドメイン内で発生した重要な状態変化を他のオブジェクトに通知するためのパターンです。過去形での命名規則を守り、不変オブジェクトとして実装します。

## 🎯 ドメインイベントの特徴

### 1. **過去形での命名**
- 既に発生した事実を表現
- 過去形の動詞を使用
- 明確で理解しやすい名前

### 2. **不変オブジェクト**
- 作成後に変更不可能
- スレッドセーフ
- 予測可能な動作

### 3. **発生時刻の管理**
- イベントの発生時刻を記録
- 時系列での追跡が可能
- 監査ログの基盤

## 📢 基本的なドメインイベント

### DomainEvent 基底クラス

```java
package com.example.domain.event;

import java.time.LocalDateTime;
import java.util.UUID;

/**
 * ドメインイベントの基底クラス
 * 
 * すべてのドメインイベントが継承する基底クラスで、
 * 共通のプロパティを定義します。
 */
public abstract class DomainEvent {
    
    private final String eventId;
    private final LocalDateTime occurredAt;
    private final String eventType;
    
    /**
     * ドメインイベントを作成
     * 
     * @param eventType イベントタイプ
     */
    protected DomainEvent(String eventType) {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = LocalDateTime.now();
        this.eventType = eventType;
    }
    
    /**
     * 既存のイベントを復元
     */
    protected DomainEvent(String eventId, LocalDateTime occurredAt, String eventType) {
        this.eventId = eventId;
        this.occurredAt = occurredAt;
        this.eventType = eventType;
    }
    
    // Getter メソッド
    public String getEventId() { return eventId; }
    public LocalDateTime getOccurredAt() { return occurredAt; }
    public String getEventType() { return eventType; }
    
    @Override
    public String toString() {
        return String.format("%s{eventId='%s', occurredAt=%s}", 
                           getClass().getSimpleName(), eventId, occurredAt);
    }
}
```

### UserCreatedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Email;
import com.example.domain.valueobject.FullName;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * ユーザー作成イベント
 * 
 * 新しいユーザーが作成された際に発行されるイベントです。
 */
public class UserCreatedEvent extends DomainEvent {
    
    private final UserId userId;
    private final Email email;
    private final FullName fullName;
    
    /**
     * ユーザー作成イベントを作成
     */
    public UserCreatedEvent(UserId userId, Email email, FullName fullName) {
        super("UserCreated");
        this.userId = userId;
        this.email = email;
        this.fullName = fullName;
    }
    
    /**
     * 既存のイベントを復元
     */
    public UserCreatedEvent(String eventId, LocalDateTime occurredAt, 
                           UserId userId, Email email, FullName fullName) {
        super(eventId, occurredAt, "UserCreated");
        this.userId = userId;
        this.email = email;
        this.fullName = fullName;
    }
    
    // Getter メソッド
    public UserId getUserId() { return userId; }
    public Email getEmail() { return email; }
    public FullName getFullName() { return fullName; }
    
    @Override
    public String toString() {
        return String.format("UserCreatedEvent{userId=%s, email=%s, fullName=%s, occurredAt=%s}", 
                           userId, email, fullName, getOccurredAt());
    }
}
```

### UserUpdatedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Email;
import com.example.domain.valueobject.FullName;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * ユーザー更新イベント
 * 
 * ユーザー情報が更新された際に発行されるイベントです。
 */
public class UserUpdatedEvent extends DomainEvent {
    
    private final UserId userId;
    private final Email email;
    private final FullName fullName;
    
    /**
     * ユーザー更新イベントを作成
     */
    public UserUpdatedEvent(UserId userId, Email email, FullName fullName) {
        super("UserUpdated");
        this.userId = userId;
        this.email = email;
        this.fullName = fullName;
    }
    
    /**
     * 既存のイベントを復元
     */
    public UserUpdatedEvent(String eventId, LocalDateTime occurredAt,
                           UserId userId, Email email, FullName fullName) {
        super(eventId, occurredAt, "UserUpdated");
        this.userId = userId;
        this.email = email;
        this.fullName = fullName;
    }
    
    // Getter メソッド
    public UserId getUserId() { return userId; }
    public Email getEmail() { return email; }
    public FullName getFullName() { return fullName; }
    
    @Override
    public String toString() {
        return String.format("UserUpdatedEvent{userId=%s, email=%s, fullName=%s, occurredAt=%s}", 
                           userId, email, fullName, getOccurredAt());
    }
}
```

### UserDeactivatedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * ユーザー無効化イベント
 * 
 * ユーザーが無効化された際に発行されるイベントです。
 */
public class UserDeactivatedEvent extends DomainEvent {
    
    private final UserId userId;
    private final String reason;
    
    /**
     * ユーザー無効化イベントを作成
     */
    public UserDeactivatedEvent(UserId userId, String reason) {
        super("UserDeactivated");
        this.userId = userId;
        this.reason = reason;
    }
    
    /**
     * 既存のイベントを復元
     */
    public UserDeactivatedEvent(String eventId, LocalDateTime occurredAt,
                               UserId userId, String reason) {
        super(eventId, occurredAt, "UserDeactivated");
        this.userId = userId;
        this.reason = reason;
    }
    
    // Getter メソッド
    public UserId getUserId() { return userId; }
    public String getReason() { return reason; }
    
    @Override
    public String toString() {
        return String.format("UserDeactivatedEvent{userId=%s, reason='%s', occurredAt=%s}", 
                           userId, reason, getOccurredAt());
    }
}
```

## 📦 注文関連のドメインイベント

### OrderCreatedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Money;
import com.example.domain.valueobject.OrderId;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * 注文作成イベント
 * 
 * 新しい注文が作成された際に発行されるイベントです。
 */
public class OrderCreatedEvent extends DomainEvent {
    
    private final OrderId orderId;
    private final UserId userId;
    private final Money totalAmount;
    private final int itemCount;
    
    /**
     * 注文作成イベントを作成
     */
    public OrderCreatedEvent(OrderId orderId, UserId userId, Money totalAmount, int itemCount) {
        super("OrderCreated");
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.itemCount = itemCount;
    }
    
    /**
     * 既存のイベントを復元
     */
    public OrderCreatedEvent(String eventId, LocalDateTime occurredAt,
                            OrderId orderId, UserId userId, Money totalAmount, int itemCount) {
        super(eventId, occurredAt, "OrderCreated");
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.itemCount = itemCount;
    }
    
    // Getter メソッド
    public OrderId getOrderId() { return orderId; }
    public UserId getUserId() { return userId; }
    public Money getTotalAmount() { return totalAmount; }
    public int getItemCount() { return itemCount; }
    
    @Override
    public String toString() {
        return String.format("OrderCreatedEvent{orderId=%s, userId=%s, totalAmount=%s, itemCount=%d, occurredAt=%s}", 
                           orderId, userId, totalAmount, itemCount, getOccurredAt());
    }
}
```

### OrderConfirmedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Money;
import com.example.domain.valueobject.OrderId;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * 注文確定イベント
 * 
 * 注文が確定された際に発行されるイベントです。
 */
public class OrderConfirmedEvent extends DomainEvent {
    
    private final OrderId orderId;
    private final UserId userId;
    private final Money totalAmount;
    private final String confirmationMethod;
    
    /**
     * 注文確定イベントを作成
     */
    public OrderConfirmedEvent(OrderId orderId, UserId userId, Money totalAmount, String confirmationMethod) {
        super("OrderConfirmed");
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.confirmationMethod = confirmationMethod;
    }
    
    /**
     * 既存のイベントを復元
     */
    public OrderConfirmedEvent(String eventId, LocalDateTime occurredAt,
                              OrderId orderId, UserId userId, Money totalAmount, String confirmationMethod) {
        super(eventId, occurredAt, "OrderConfirmed");
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.confirmationMethod = confirmationMethod;
    }
    
    // Getter メソッド
    public OrderId getOrderId() { return orderId; }
    public UserId getUserId() { return userId; }
    public Money getTotalAmount() { return totalAmount; }
    public String getConfirmationMethod() { return confirmationMethod; }
    
    @Override
    public String toString() {
        return String.format("OrderConfirmedEvent{orderId=%s, userId=%s, totalAmount=%s, confirmationMethod='%s', occurredAt=%s}", 
                           orderId, userId, totalAmount, confirmationMethod, getOccurredAt());
    }
}
```

### OrderCancelledEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Money;
import com.example.domain.valueobject.OrderId;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * 注文キャンセルイベント
 * 
 * 注文がキャンセルされた際に発行されるイベントです。
 */
public class OrderCancelledEvent extends DomainEvent {
    
    private final OrderId orderId;
    private final UserId userId;
    private final Money totalAmount;
    private final String cancellationReason;
    
    /**
     * 注文キャンセルイベントを作成
     */
    public OrderCancelledEvent(OrderId orderId, UserId userId, Money totalAmount, String cancellationReason) {
        super("OrderCancelled");
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.cancellationReason = cancellationReason;
    }
    
    /**
     * 既存のイベントを復元
     */
    public OrderCancelledEvent(String eventId, LocalDateTime occurredAt,
                              OrderId orderId, UserId userId, Money totalAmount, String cancellationReason) {
        super(eventId, occurredAt, "OrderCancelled");
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.cancellationReason = cancellationReason;
    }
    
    // Getter メソッド
    public OrderId getOrderId() { return orderId; }
    public UserId getUserId() { return userId; }
    public Money getTotalAmount() { return totalAmount; }
    public String getCancellationReason() { return cancellationReason; }
    
    @Override
    public String toString() {
        return String.format("OrderCancelledEvent{orderId=%s, userId=%s, totalAmount=%s, cancellationReason='%s', occurredAt=%s}", 
                           orderId, userId, totalAmount, cancellationReason, getOccurredAt());
    }
}
```

## 🎨 ドメインイベントの設計指針

### 1. **過去形での命名規則**
- 既に発生した事実を表現
- 過去形の動詞を使用
- 明確で理解しやすい名前

### 2. **不変オブジェクトの実装**
- 作成後に変更不可能
- スレッドセーフ
- 予測可能な動作

### 3. **発生時刻の管理**
- イベントの発生時刻を記録
- 時系列での追跡が可能
- 監査ログの基盤

### 4. **適切な粒度の設計**
- 重要な状態変化のみをイベント化
- 過度に細かいイベントは避ける
- ビジネス価値のあるイベント

## 🧪 ドメインイベントのテスト

```groovy
package com.example.domain.event

import com.example.domain.valueobject.Email
import com.example.domain.valueobject.FullName
import com.example.domain.valueobject.UserId
import spock.lang.Specification

class UserCreatedEventTest extends Specification {
    
    def "ユーザー作成イベントを作成できる"() {
        given:
        def userId = UserId.generate()
        def email = new Email("test@example.com")
        def fullName = new FullName("田中", "太郎")
        
        when:
        def event = new UserCreatedEvent(userId, email, fullName)
        
        then:
        event.eventId != null
        event.occurredAt != null
        event.eventType == "UserCreated"
        event.userId == userId
        event.email == email
        event.fullName == fullName
    }
    
    def "ユーザー作成イベントを復元できる"() {
        given:
        def eventId = "test-event-id"
        def occurredAt = LocalDateTime.now()
        def userId = UserId.generate()
        def email = new Email("test@example.com")
        def fullName = new FullName("田中", "太郎")
        
        when:
        def event = new UserCreatedEvent(eventId, occurredAt, userId, email, fullName)
        
        then:
        event.eventId == eventId
        event.occurredAt == occurredAt
        event.eventType == "UserCreated"
        event.userId == userId
        event.email == email
        event.fullName == fullName
    }
}

class OrderCreatedEventTest extends Specification {
    
    def "注文作成イベントを作成できる"() {
        given:
        def orderId = OrderId.generate()
        def userId = UserId.generate()
        def totalAmount = Money.of("1000", Currency.getInstance("JPY"))
        def itemCount = 2
        
        when:
        def event = new OrderCreatedEvent(orderId, userId, totalAmount, itemCount)
        
        then:
        event.eventId != null
        event.occurredAt != null
        event.eventType == "OrderCreated"
        event.orderId == orderId
        event.userId == userId
        event.totalAmount == totalAmount
        event.itemCount == itemCount
    }
}
```

## 🔄 次のステップ

ドメインイベントの実装が完了したら、次はファクトリパターンの実装に進みます。ファクトリは複雑なオブジェクト生成をカプセル化し、不整合な状態の防止を実現します。 