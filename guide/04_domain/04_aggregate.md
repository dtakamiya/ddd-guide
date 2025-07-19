# 4.4 集約（Aggregates）の設計

集約は、ドメイン駆動設計において一貫性の境界を定義する重要なパターンです。関連するエンティティと値オブジェクトをグループ化し、集約ルートを通じて外部からのアクセスを制御します。

## 🎯 集約の特徴

### 1. **一貫性の境界**
- 関連するオブジェクトをグループ化
- トランザクション境界の定義
- 整合性の保証

### 2. **集約ルート**
- 外部からのアクセスの唯一の入口
- 内部オブジェクトの状態変更を制御
- ビジネスルールの強制

### 3. **不変条件の保証**
- 集約内の整合性を保証
- ビジネスルールの強制
- 不正な状態の防止

## 📦 注文集約の実装

### OrderId 値オブジェクト

```java
package com.example.domain.valueobject;

import java.util.Objects;
import java.util.UUID;

/**
 * 注文IDを表現する値オブジェクト
 */
public record OrderId(String value) {
    
    public OrderId {
        Objects.requireNonNull(value, "注文IDは必須です");
        
        if (value.trim().isEmpty()) {
            throw new IllegalArgumentException("注文IDは空文字列にできません");
        }
        
        // UUID形式の検証
        try {
            UUID.fromString(value);
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException("無効なUUID形式です: " + value);
        }
        
        value = value.trim();
    }
    
    /**
     * 新しい注文IDを生成
     */
    public static OrderId generate() {
        return new OrderId(UUID.randomUUID().toString());
    }
    
    /**
     * 文字列から注文IDを作成
     */
    public static OrderId of(String value) {
        return new OrderId(value);
    }
    
    @Override
    public String toString() {
        return value;
    }
}
```

### OrderItem エンティティ

```java
package com.example.domain.entity;

import com.example.domain.valueobject.Money;
import com.example.domain.valueobject.ProductId;

import java.util.Objects;

/**
 * 注文項目を表現するエンティティ
 */
public class OrderItem {
    
    private ProductId productId;
    private String productName;
    private Money unitPrice;
    private int quantity;
    private Money totalPrice;
    
    /**
     * 注文項目を作成
     */
    public OrderItem(ProductId productId, String productName, Money unitPrice, int quantity) {
        this.productId = Objects.requireNonNull(productId, "商品IDは必須です");
        this.productName = Objects.requireNonNull(productName, "商品名は必須です");
        this.unitPrice = Objects.requireNonNull(unitPrice, "単価は必須です");
        
        if (quantity <= 0) {
            throw new IllegalArgumentException("数量は1以上である必要があります");
        }
        
        this.quantity = quantity;
        this.totalPrice = unitPrice.multiply(new java.math.BigDecimal(quantity));
    }
    
    /**
     * 数量を変更
     */
    public void changeQuantity(int newQuantity) {
        if (newQuantity <= 0) {
            throw new IllegalArgumentException("数量は1以上である必要があります");
        }
        
        this.quantity = newQuantity;
        this.totalPrice = unitPrice.multiply(new java.math.BigDecimal(newQuantity));
    }
    
    /**
     * 単価を変更
     */
    public void changeUnitPrice(Money newUnitPrice) {
        this.unitPrice = Objects.requireNonNull(newUnitPrice, "単価は必須です");
        this.totalPrice = newUnitPrice.multiply(new java.math.BigDecimal(quantity));
    }
    
    // Getter メソッド
    public ProductId getProductId() { return productId; }
    public String getProductName() { return productName; }
    public Money getUnitPrice() { return unitPrice; }
    public int getQuantity() { return quantity; }
    public Money getTotalPrice() { return totalPrice; }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        OrderItem orderItem = (OrderItem) obj;
        return Objects.equals(productId, orderItem.productId);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(productId);
    }
    
    @Override
    public String toString() {
        return String.format("OrderItem{productId=%s, productName='%s', quantity=%d, totalPrice=%s}", 
                           productId, productName, quantity, totalPrice);
    }
}
```

### Order 集約ルート

```java
package com.example.domain.aggregate;

import com.example.domain.entity.OrderItem;
import com.example.domain.entity.User;
import com.example.domain.valueobject.*;
import com.example.domain.event.OrderCreatedEvent;
import com.example.domain.event.OrderConfirmedEvent;
import com.example.domain.event.OrderCancelledEvent;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * 注文を表現する集約ルート
 * 
 * 注文のライフサイクルを管理し、
 * 注文項目の整合性を保証します。
 */
public class Order {
    
    private OrderId id;
    private UserId userId;
    private List<OrderItem> items;
    private Money totalAmount;
    private OrderStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private List<Object> domainEvents;
    
    /**
     * プライベートコンストラクタ
     */
    private Order() {
        this.items = new ArrayList<>();
        this.domainEvents = new ArrayList<>();
    }
    
    /**
     * 注文を作成するファクトリメソッド
     */
    public static Order create(UserId userId, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("注文項目は必須です");
        }
        
        Order order = new Order();
        order.id = OrderId.generate();
        order.userId = Objects.requireNonNull(userId, "ユーザーIDは必須です");
        order.items = new ArrayList<>(items);
        order.status = OrderStatus.PENDING;
        order.createdAt = LocalDateTime.now();
        order.updatedAt = LocalDateTime.now();
        
        // 合計金額を計算
        order.calculateTotalAmount();
        
        // ドメインイベントを発行
        order.addDomainEvent(new OrderCreatedEvent(order.id, order.userId, order.totalAmount));
        
        return order;
    }
    
    /**
     * 既存注文を復元するファクトリメソッド
     */
    public static Order reconstruct(
            OrderId id,
            UserId userId,
            List<OrderItem> items,
            Money totalAmount,
            OrderStatus status,
            LocalDateTime createdAt,
            LocalDateTime updatedAt) {
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
     * 注文項目を追加
     */
    public void addItem(OrderItem item) {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("確定済みの注文には項目を追加できません");
        }
        
        // 既存の同じ商品がある場合は数量を追加
        for (OrderItem existingItem : items) {
            if (existingItem.getProductId().equals(item.getProductId())) {
                int newQuantity = existingItem.getQuantity() + item.getQuantity();
                existingItem.changeQuantity(newQuantity);
                calculateTotalAmount();
                return;
            }
        }
        
        // 新しい商品の場合は追加
        items.add(item);
        calculateTotalAmount();
        this.updatedAt = LocalDateTime.now();
    }
    
    /**
     * 注文項目を削除
     */
    public void removeItem(ProductId productId) {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("確定済みの注文から項目を削除できません");
        }
        
        items.removeIf(item -> item.getProductId().equals(productId));
        calculateTotalAmount();
        this.updatedAt = LocalDateTime.now();
    }
    
    /**
     * 注文項目の数量を変更
     */
    public void changeItemQuantity(ProductId productId, int newQuantity) {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("確定済みの注文の数量を変更できません");
        }
        
        for (OrderItem item : items) {
            if (item.getProductId().equals(productId)) {
                item.changeQuantity(newQuantity);
                calculateTotalAmount();
                this.updatedAt = LocalDateTime.now();
                return;
            }
        }
        
        throw new IllegalArgumentException("指定された商品が見つかりません: " + productId);
    }
    
    /**
     * 注文を確定
     */
    public void confirm() {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("確定済みまたはキャンセル済みの注文は確定できません");
        }
        
        if (items.isEmpty()) {
            throw new IllegalStateException("注文項目がない注文は確定できません");
        }
        
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = LocalDateTime.now();
        
        // ドメインイベントを発行
        addDomainEvent(new OrderConfirmedEvent(this.id, this.userId, this.totalAmount));
    }
    
    /**
     * 注文をキャンセル
     */
    public void cancel() {
        if (this.status == OrderStatus.CANCELLED) {
            throw new IllegalStateException("既にキャンセル済みです");
        }
        
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
        
        // ドメインイベントを発行
        addDomainEvent(new OrderCancelledEvent(this.id, this.userId, this.totalAmount));
    }
    
    /**
     * 合計金額を計算
     */
    private void calculateTotalAmount() {
        if (items.isEmpty()) {
            this.totalAmount = Money.zero(items.get(0).getUnitPrice().currency());
            return;
        }
        
        Money total = Money.zero(items.get(0).getUnitPrice().currency());
        for (OrderItem item : items) {
            total = total.add(item.getTotalPrice());
        }
        this.totalAmount = total;
    }
    
    /**
     * 注文が確定済みかどうかを判定
     */
    public boolean isConfirmed() {
        return this.status == OrderStatus.CONFIRMED;
    }
    
    /**
     * 注文がキャンセル済みかどうかを判定
     */
    public boolean isCancelled() {
        return this.status == OrderStatus.CANCELLED;
    }
    
    /**
     * 注文項目数を取得
     */
    public int getItemCount() {
        return items.size();
    }
    
    /**
     * ドメインイベントを追加
     */
    private void addDomainEvent(Object event) {
        this.domainEvents.add(event);
    }
    
    /**
     * ドメインイベントを取得してクリア
     */
    public List<Object> getDomainEvents() {
        List<Object> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear();
        return events;
    }
    
    // Getter メソッド
    public OrderId getId() { return id; }
    public UserId getUserId() { return userId; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    public Money getTotalAmount() { return totalAmount; }
    public OrderStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    
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
    
    @Override
    public String toString() {
        return String.format("Order{id=%s, userId=%s, itemCount=%d, totalAmount=%s, status=%s}", 
                           id, userId, getItemCount(), totalAmount, status);
    }
}
```

### OrderStatus 列挙型

```java
package com.example.domain.aggregate;

/**
 * 注文ステータスを表現する列挙型
 */
public enum OrderStatus {
    PENDING("保留中"),
    CONFIRMED("確定済み"),
    CANCELLED("キャンセル済み");
    
    private final String displayName;
    
    OrderStatus(String displayName) {
        this.displayName = displayName;
    }
    
    public String getDisplayName() {
        return displayName;
    }
}
```

## 📢 集約関連のドメインイベント

### OrderCreatedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Money;
import com.example.domain.valueobject.OrderId;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * 注文作成イベント
 */
public record OrderCreatedEvent(
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    LocalDateTime occurredAt
) {
    
    public OrderCreatedEvent(OrderId orderId, UserId userId, Money totalAmount) {
        this(orderId, userId, totalAmount, LocalDateTime.now());
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
 */
public record OrderConfirmedEvent(
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    LocalDateTime occurredAt
) {
    
    public OrderConfirmedEvent(OrderId orderId, UserId userId, Money totalAmount) {
        this(orderId, userId, totalAmount, LocalDateTime.now());
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
 */
public record OrderCancelledEvent(
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    LocalDateTime occurredAt
) {
    
    public OrderCancelledEvent(OrderId orderId, UserId userId, Money totalAmount) {
        this(orderId, userId, totalAmount, LocalDateTime.now());
    }
}
```

## 🎨 集約の設計指針

### 1. **一貫性の境界の明確化**
- 関連するオブジェクトを論理的にグループ化
- トランザクション境界の定義
- 整合性の保証

### 2. **集約ルートの設計**
- 外部からのアクセスの唯一の入口
- 内部オブジェクトの状態変更を制御
- ビジネスルールの強制

### 3. **不変条件の保証**
- 集約内の整合性を保証
- ビジネスルールの強制
- 不正な状態の防止

### 4. **適切なサイズの決定**
- 小さすぎる集約は避ける
- 大きすぎる集約も避ける
- ビジネス要件に基づく設計

## 🧪 集約のテスト

```groovy
package com.example.domain.aggregate

import com.example.domain.entity.OrderItem
import com.example.domain.valueobject.Money
import com.example.domain.valueobject.ProductId
import com.example.domain.valueobject.UserId
import spock.lang.Specification

class OrderTest extends Specification {
    
    def "注文を作成できる"() {
        given:
        def userId = UserId.generate()
        def items = [
            new OrderItem(
                ProductId.generate(),
                "商品A",
                Money.of("1000", Currency.getInstance("JPY")),
                2
            )
        ]
        
        when:
        def order = Order.create(userId, items)
        
        then:
        order.id != null
        order.userId == userId
        order.items.size() == 1
        order.totalAmount.amount() == new BigDecimal("2000")
        order.status == OrderStatus.PENDING
        order.isConfirmed() == false
        order.isCancelled() == false
        
        and: "ドメインイベントが発行される"
        def events = order.getDomainEvents()
        events.size() == 1
        events[0] instanceof OrderCreatedEvent
    }
    
    def "注文項目を追加できる"() {
        given:
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        def newItem = new OrderItem(ProductId.generate(), "商品B", Money.of("500", Currency.getInstance("JPY")), 2)
        
        when:
        order.addItem(newItem)
        
        then:
        order.items.size() == 2
        order.totalAmount.amount() == new BigDecimal("2000")
    }
    
    def "同じ商品を追加すると数量が合算される"() {
        given:
        def productId = ProductId.generate()
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(productId, "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        def additionalItem = new OrderItem(productId, "商品A", Money.of("1000", Currency.getInstance("JPY")), 2)
        
        when:
        order.addItem(additionalItem)
        
        then:
        order.items.size() == 1
        order.items[0].quantity == 3
        order.totalAmount.amount() == new BigDecimal("3000")
    }
    
    def "注文を確定できる"() {
        given:
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        
        when:
        order.confirm()
        
        then:
        order.status == OrderStatus.CONFIRMED
        order.isConfirmed()
        
        and: "ドメインイベントが発行される"
        def events = order.getDomainEvents()
        events.size() == 1
        events[0] instanceof OrderConfirmedEvent
    }
    
    def "確定済みの注文には項目を追加できない"() {
        given:
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        order.confirm()
        def newItem = new OrderItem(ProductId.generate(), "商品B", Money.of("500", Currency.getInstance("JPY")), 1)
        
        when:
        order.addItem(newItem)
        
        then:
        thrown(IllegalStateException)
    }
    
    def "注文をキャンセルできる"() {
        given:
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        
        when:
        order.cancel()
        
        then:
        order.status == OrderStatus.CANCELLED
        order.isCancelled()
        
        and: "ドメインイベントが発行される"
        def events = order.getDomainEvents()
        events.size() == 1
        events[0] instanceof OrderCancelledEvent
    }
    
    def "空の注文項目では注文を作成できない"() {
        when:
        Order.create(UserId.generate(), [])
        
        then:
        thrown(IllegalArgumentException)
    }
}
```

## 🔄 次のステップ

集約の実装が完了したら、次はドメインサービスの実装に進みます。ドメインサービスは、複数の集約にまたがるビジネスロジックを実装する重要なパターンです。 