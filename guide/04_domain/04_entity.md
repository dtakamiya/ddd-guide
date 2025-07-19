# 4.3 エンティティ（Entities）の実装

エンティティは、ドメイン駆動設計において識別子を持つ重要なドメインオブジェクトです。値オブジェクトとは異なり、同一性（Identity）によって識別され、ライフサイクルを通じて状態が変化します。

## 🎯 エンティティの特徴

### 1. **識別子による同一性**
- 一意の識別子（ID）を持つ
- 内容が同じでも異なるエンティティとして扱う
- 参照による等価性

### 2. **ライフサイクル管理**
- 作成、変更、削除の状態変化
- ドメインイベントの発行
- 状態遷移の制御

### 3. **ビジネスロジックのカプセル化**
- エンティティ自身がビジネスルールを実装
- 外部からの直接的な状態変更を防止
- 整合性の保証

## 🆔 識別子の実装

### UserId 値オブジェクト

```java
package com.example.domain.valueobject;

import java.util.Objects;
import java.util.UUID;

/**
 * ユーザーIDを表現する値オブジェクト
 * 
 * UUIDを使用して一意性を保証し、
 * 型安全性を提供します。
 */
public record UserId(String value) {
    
    /**
     * ユーザーIDを作成します
     * 
     * @param value ユーザーID文字列
     * @throws IllegalArgumentException 無効なユーザーIDの場合
     */
    public UserId {
        Objects.requireNonNull(value, "ユーザーIDは必須です");
        
        if (value.trim().isEmpty()) {
            throw new IllegalArgumentException("ユーザーIDは空文字列にできません");
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

## 👤 エンティティの実装

### User エンティティ

```java
package com.example.domain.entity;

import com.example.domain.valueobject.Email;
import com.example.domain.valueobject.FullName;
import com.example.domain.valueobject.UserId;
import com.example.domain.event.UserCreatedEvent;
import com.example.domain.event.UserUpdatedEvent;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * ユーザーを表現するエンティティ
 * 
 * ユーザーのライフサイクルを管理し、
 * ビジネスルールをカプセル化します。
 */
public class User {
    
    private UserId id;
    private Email email;
    private FullName fullName;
    private UserStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private List<Object> domainEvents;
    
    /**
     * プライベートコンストラクタ
     * ファクトリメソッド経由での作成を強制
     */
    private User() {
        this.domainEvents = new ArrayList<>();
    }
    
    /**
     * ユーザーを作成するファクトリメソッド
     */
    public static User create(Email email, FullName fullName) {
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
     * 既存ユーザーを復元するファクトリメソッド
     */
    public static User reconstruct(
            UserId id, 
            Email email, 
            FullName fullName, 
            UserStatus status,
            LocalDateTime createdAt,
            LocalDateTime updatedAt) {
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
     * ユーザー情報を更新
     */
    public void updateProfile(FullName newFullName) {
        Objects.requireNonNull(newFullName, "氏名は必須です");
        
        if (this.status != UserStatus.ACTIVE) {
            throw new IllegalStateException("非アクティブなユーザーは更新できません");
        }
        
        this.fullName = newFullName;
        this.updatedAt = LocalDateTime.now();
        
        // ドメインイベントを発行
        addDomainEvent(new UserUpdatedEvent(this.id, this.email, this.fullName));
    }
    
    /**
     * ユーザーを無効化
     */
    public void deactivate() {
        if (this.status == UserStatus.INACTIVE) {
            throw new IllegalStateException("既に無効化されています");
        }
        
        this.status = UserStatus.INACTIVE;
        this.updatedAt = LocalDateTime.now();
    }
    
    /**
     * ユーザーを再有効化
     */
    public void reactivate() {
        if (this.status == UserStatus.ACTIVE) {
            throw new IllegalStateException("既に有効です");
        }
        
        this.status = UserStatus.ACTIVE;
        this.updatedAt = LocalDateTime.now();
    }
    
    /**
     * ユーザーがアクティブかどうかを判定
     */
    public boolean isActive() {
        return this.status == UserStatus.ACTIVE;
    }
    
    /**
     * メールアドレスが特定のドメインかどうかを判定
     */
    public boolean isFromDomain(String domain) {
        return this.email.isFromDomain(domain);
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
    public UserId getId() { return id; }
    public Email getEmail() { return email; }
    public FullName getFullName() { return fullName; }
    public UserStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    
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
        return String.format("User{id=%s, email=%s, fullName=%s, status=%s}", 
                           id, email, fullName, status);
    }
}
```

### UserStatus 列挙型

```java
package com.example.domain.entity;

/**
 * ユーザーステータスを表現する列挙型
 */
public enum UserStatus {
    ACTIVE("有効"),
    INACTIVE("無効"),
    SUSPENDED("一時停止");
    
    private final String displayName;
    
    UserStatus(String displayName) {
        this.displayName = displayName;
    }
    
    public String getDisplayName() {
        return displayName;
    }
}
```

## 📢 ドメインイベントの実装

### UserCreatedEvent

```java
package com.example.domain.event;

import com.example.domain.valueobject.Email;
import com.example.domain.valueobject.FullName;
import com.example.domain.valueobject.UserId;

import java.time.LocalDateTime;

/**
 * ユーザー作成イベント
 */
public record UserCreatedEvent(
    UserId userId,
    Email email,
    FullName fullName,
    LocalDateTime occurredAt
) {
    
    public UserCreatedEvent(UserId userId, Email email, FullName fullName) {
        this(userId, email, fullName, LocalDateTime.now());
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
 */
public record UserUpdatedEvent(
    UserId userId,
    Email email,
    FullName fullName,
    LocalDateTime occurredAt
) {
    
    public UserUpdatedEvent(UserId userId, Email email, FullName fullName) {
        this(userId, email, fullName, LocalDateTime.now());
    }
}
```

## 🎨 エンティティの設計指針

### 1. **識別子の適切な設計**
- 一意性を保証する識別子を使用
- 値オブジェクトとして実装
- 型安全性を確保

### 2. **ファクトリメソッドの活用**
- 複雑な作成ロジックをカプセル化
- 不正な状態のエンティティ作成を防止
- ドメインイベントの発行

### 3. **ビジネスロジックのカプセル化**
- エンティティ自身がビジネスルールを実装
- 外部からの直接的な状態変更を防止
- 整合性の保証

### 4. **ドメインイベントの発行**
- 重要な状態変化をイベントとして発行
- 他のエンティティやサービスへの通知
- イベント駆動設計の実現

## 🧪 エンティティのテスト

```groovy
package com.example.domain.entity

import com.example.domain.valueobject.Email
import com.example.domain.valueobject.FullName
import spock.lang.Specification

class UserTest extends Specification {
    
    def "ユーザーを作成できる"() {
        given:
        def email = new Email("test@example.com")
        def fullName = new FullName("田中", "太郎")
        
        when:
        def user = User.create(email, fullName)
        
        then:
        user.id != null
        user.email == email
        user.fullName == fullName
        user.status == UserStatus.ACTIVE
        user.isActive()
        user.createdAt != null
        user.updatedAt != null
        
        and: "ドメインイベントが発行される"
        def events = user.getDomainEvents()
        events.size() == 1
        events[0] instanceof UserCreatedEvent
    }
    
    def "ユーザー情報を更新できる"() {
        given:
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        def newFullName = new FullName("佐藤", "次郎")
        
        when:
        user.updateProfile(newFullName)
        
        then:
        user.fullName == newFullName
        user.updatedAt > user.createdAt
        
        and: "ドメインイベントが発行される"
        def events = user.getDomainEvents()
        events.size() == 1
        events[0] instanceof UserUpdatedEvent
    }
    
    def "非アクティブなユーザーは更新できない"() {
        given:
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        user.deactivate()
        
        when:
        user.updateProfile(new FullName("佐藤", "次郎"))
        
        then:
        thrown(IllegalStateException)
    }
    
    def "ユーザーを無効化できる"() {
        given:
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        
        when:
        user.deactivate()
        
        then:
        user.status == UserStatus.INACTIVE
        !user.isActive()
    }
    
    def "ユーザーを再有効化できる"() {
        given:
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        user.deactivate()
        
        when:
        user.reactivate()
        
        then:
        user.status == UserStatus.ACTIVE
        user.isActive()
    }
    
    def "特定のドメインかどうかを判定できる"() {
        given:
        def user = User.create(
            new Email("test@example.com"),
            new FullName("田中", "太郎")
        )
        
        expect:
        user.isFromDomain("example.com")
        !user.isFromDomain("other.com")
    }
    
    def "同じIDのユーザーは等しい"() {
        given:
        def user1 = User.reconstruct(
            UserId.of("123e4567-e89b-12d3-a456-426614174000"),
            new Email("test1@example.com"),
            new FullName("田中", "太郎"),
            UserStatus.ACTIVE,
            LocalDateTime.now(),
            LocalDateTime.now()
        )
        def user2 = User.reconstruct(
            UserId.of("123e4567-e89b-12d3-a456-426614174000"),
            new Email("test2@example.com"),
            new FullName("佐藤", "次郎"),
            UserStatus.INACTIVE,
            LocalDateTime.now(),
            LocalDateTime.now()
        )
        
        expect:
        user1 == user2
        user1.hashCode() == user2.hashCode()
    }
}
```

## 🔄 次のステップ

エンティティの実装が完了したら、次は集約の実装に進みます。集約は一貫性の境界を定義し、複雑なビジネスルールを管理する重要なパターンです。 