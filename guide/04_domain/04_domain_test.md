# 4.9 ドメイン層のテスト

ドメイン層のテストは、ビジネスルールの正確性を保証する重要な要素です。Spockフレームワークを使用して、読みやすく保守しやすいテストを実装します。

## 🎯 ドメイン層テストの特徴

### 1. **ビジネスルールの検証**
- ドメインルールの正確性を保証
- 不正な状態の防止を確認
- ビジネスロジックの動作を検証

### 2. **読みやすいテスト**
- Spockフレームワークの活用
- 自然言語に近いテスト記述
- 明確なテスト意図の表現

### 3. **包括的なテストカバレッジ**
- 正常系と異常系の両方をテスト
- 境界値のテスト
- エッジケースの検証

## 🧪 値オブジェクトのテスト

### EmailTest

```groovy
package com.example.domain.valueobject

import spock.lang.Specification
import spock.lang.Unroll

class EmailTest extends Specification {
    
    @Unroll
    def "有効なメールアドレス #email を作成できる"() {
        when:
        def email = new Email(inputEmail)
        
        then:
        email.value() == expectedValue
        email.getLocalPart() == expectedLocalPart
        email.getDomain() == expectedDomain
        
        where:
        inputEmail               | expectedValue             | expectedLocalPart | expectedDomain
        "test@example.com"      | "test@example.com"       | "test"            | "example.com"
        "USER@EXAMPLE.COM"      | "user@example.com"       | "user"            | "example.com"
        "test.user@domain.co.jp"| "test.user@domain.co.jp"| "test.user"       | "domain.co.jp"
        "user+tag@example.com"  | "user+tag@example.com"   | "user+tag"        | "example.com"
    }
    
    @Unroll
    def "無効なメールアドレス #email で例外が発生する"() {
        when:
        new Email(email)
        
        then:
        thrown(IllegalArgumentException)
        
        where:
        email << [
            null,
            "",
            "   ",
            "invalid-email",
            "@example.com",
            "test@",
            "test@.com",
            "test..user@example.com",
            "test@example",
            "test@.com",
            "test@example..com"
        ]
    }
    
    def "特定のドメインかどうかを判定できる"() {
        given:
        def email = new Email("test@example.com")
        
        expect:
        email.isFromDomain("example.com")
        !email.isFromDomain("other.com")
        email.isFromDomain("EXAMPLE.COM") // 大文字小文字を区別しない
    }
    
    def "メールアドレスの等価性が正しく動作する"() {
        given:
        def email1 = new Email("test@example.com")
        def email2 = new Email("test@example.com")
        def email3 = new Email("other@example.com")
        
        expect:
        email1 == email2
        email1.hashCode() == email2.hashCode()
        email1 != email3
    }
    
    def "メールアドレスの文字列表現が正しい"() {
        given:
        def email = new Email("test@example.com")
        
        expect:
        email.toString() == "test@example.com"
    }
}
```

### MoneyTest

```groovy
package com.example.domain.valueobject

import spock.lang.Specification
import spock.lang.Unroll

class MoneyTest extends Specification {
    
    def "金額の加算が正しく動作する"() {
        given:
        def money1 = Money.of(new BigDecimal("100.50"), Currency.getInstance("JPY"))
        def money2 = Money.of(new BigDecimal("200.25"), Currency.getInstance("JPY"))
        
        when:
        def result = money1.add(money2)
        
        then:
        result.amount() == new BigDecimal("300.75")
        result.currency() == Currency.getInstance("JPY")
    }
    
    def "金額の減算が正しく動作する"() {
        given:
        def money1 = Money.of(new BigDecimal("500.00"), Currency.getInstance("JPY"))
        def money2 = Money.of(new BigDecimal("200.25"), Currency.getInstance("JPY"))
        
        when:
        def result = money1.subtract(money2)
        
        then:
        result.amount() == new BigDecimal("299.75")
        result.currency() == Currency.getInstance("JPY")
    }
    
    def "金額の乗算が正しく動作する"() {
        given:
        def money = Money.of(new BigDecimal("100.00"), Currency.getInstance("JPY"))
        def factor = new BigDecimal("1.1")
        
        when:
        def result = money.multiply(factor)
        
        then:
        result.amount() == new BigDecimal("110.00")
        result.currency() == Currency.getInstance("JPY")
    }
    
    def "異なる通貨間での計算で例外が発生する"() {
        given:
        def jpyMoney = Money.of(new BigDecimal("100"), Currency.getInstance("JPY"))
        def usdMoney = Money.of(new BigDecimal("1"), Currency.getInstance("USD"))
        
        when:
        jpyMoney.add(usdMoney)
        
        then:
        thrown(IllegalArgumentException)
    }
    
    def "負の金額で例外が発生する"() {
        when:
        Money.of(new BigDecimal("-100"), Currency.getInstance("JPY"))
        
        then:
        thrown(IllegalArgumentException)
    }
    
    def "ゼロ金額を作成できる"() {
        when:
        def zeroMoney = Money.zero(Currency.getInstance("JPY"))
        
        then:
        zeroMoney.isZero()
        zeroMoney.amount() == BigDecimal.ZERO
    }
    
    def "金額の比較が正しく動作する"() {
        given:
        def money1 = Money.of(new BigDecimal("100"), Currency.getInstance("JPY"))
        def money2 = Money.of(new BigDecimal("200"), Currency.getInstance("JPY"))
        
        expect:
        money2.isGreaterThan(money1)
        !money1.isGreaterThan(money2)
    }
    
    @Unroll
    def "表示用文字列が正しく生成される"() {
        given:
        def money = Money.of(amount, Currency.getInstance("JPY"))
        
        expect:
        money.getDisplayString() == expectedDisplay
        
        where:
        amount                    | expectedDisplay
        new BigDecimal("100")    | "¥100.00"
        new BigDecimal("1000.5") | "¥1000.50"
        new BigDecimal("0")      | "¥0.00"
    }
}
```

## 🧪 エンティティのテスト

### UserTest

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
    
    def "異なるIDのユーザーは等しくない"() {
        given:
        def user1 = User.create(
            new Email("test1@example.com"),
            new FullName("田中", "太郎")
        )
        def user2 = User.create(
            new Email("test2@example.com"),
            new FullName("佐藤", "次郎")
        )
        
        expect:
        user1 != user2
        user1.hashCode() != user2.hashCode()
    }
}
```

## 🧪 集約のテスト

### OrderTest

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
            new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 2)
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
    
    def "注文項目の数量を変更できる"() {
        given:
        def productId = ProductId.generate()
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(productId, "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        
        when:
        order.changeItemQuantity(productId, 3)
        
        then:
        order.items[0].quantity == 3
        order.totalAmount.amount() == new BigDecimal("3000")
    }
    
    def "存在しない商品の数量変更で例外が発生する"() {
        given:
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 1)]
        )
        def nonExistentProductId = ProductId.generate()
        
        when:
        order.changeItemQuantity(nonExistentProductId, 3)
        
        then:
        thrown(IllegalArgumentException)
    }
}
```

## 🧪 ドメインサービスのテスト

### UserDuplicateCheckerTest

```groovy
package com.example.domain.service

import com.example.domain.entity.User
import com.example.domain.valueobject.Email
import com.example.domain.valueobject.FullName
import spock.lang.Specification

class UserDuplicateCheckerTest extends Specification {
    
    def "メールアドレスの重複をチェックできる"() {
        given:
        def checker = new UserDuplicateChecker()
        def email = new Email("test@example.com")
        def existingUsers = [
            User.create(new Email("user1@example.com"), new FullName("田中", "太郎")),
            User.create(new Email("test@example.com"), new FullName("佐藤", "次郎")),
            User.create(new Email("user2@example.com"), new FullName("鈴木", "三郎"))
        ]
        
        when:
        def isDuplicate = checker.isEmailDuplicate(email, existingUsers)
        
        then:
        isDuplicate == true
    }
    
    def "類似ユーザーを検出できる"() {
        given:
        def checker = new UserDuplicateChecker()
        def targetUser = User.create(new Email("test@example.com"), new FullName("田中", "太郎"))
        def existingUsers = [
            User.create(new Email("user1@example.com"), new FullName("田中", "次郎")),
            User.create(new Email("test@other.com"), new FullName("佐藤", "三郎")),
            User.create(new Email("user2@example.com"), new FullName("鈴木", "四郎"))
        ]
        
        when:
        def similarUsers = checker.findSimilarUsers(targetUser, existingUsers)
        
        then:
        similarUsers.size() == 2
        similarUsers.any { it.getEmail().getDomain() == "example.com" }
        similarUsers.any { it.getFullName().getLastName() == "田中" }
    }
    
    def "重複リスクスコアを計算できる"() {
        given:
        def checker = new UserDuplicateChecker()
        def targetUser = User.create(new Email("test@example.com"), new FullName("田中", "太郎"))
        def existingUsers = [
            User.create(new Email("test@example.com"), new FullName("佐藤", "次郎")),
            User.create(new Email("user1@example.com"), new FullName("田中", "次郎"))
        ]
        
        when:
        def riskScore = checker.calculateDuplicateRiskScore(targetUser, existingUsers)
        
        then:
        riskScore == 1.0 // 完全一致があるため最大リスク
    }
}
```

### OrderCalculationServiceTest

```groovy
package com.example.domain.service

import com.example.domain.aggregate.Order
import com.example.domain.entity.OrderItem
import com.example.domain.valueobject.Money
import spock.lang.Specification

class OrderCalculationServiceTest extends Specification {
    
    def "小計を正しく計算できる"() {
        given:
        def service = new OrderCalculationService()
        def items = [
            new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 2),
            new OrderItem(ProductId.generate(), "商品B", Money.of("500", Currency.getInstance("JPY")), 1)
        ]
        
        when:
        def subtotal = service.calculateSubtotal(items)
        
        then:
        subtotal.amount() == new BigDecimal("2500")
    }
    
    def "消費税を正しく計算できる"() {
        given:
        def service = new OrderCalculationService()
        def subtotal = Money.of("1000", Currency.getInstance("JPY"))
        def taxRate = new BigDecimal("0.1")
        
        when:
        def tax = service.calculateTax(subtotal, taxRate)
        
        then:
        tax.amount() == new BigDecimal("100")
    }
    
    def "配送料を正しく計算できる"() {
        given:
        def service = new OrderCalculationService()
        def subtotal = Money.of("3000", Currency.getInstance("JPY"))
        def itemCount = 3
        
        when:
        def shippingFee = service.calculateShippingFee(subtotal, itemCount)
        
        then:
        shippingFee.amount() == new BigDecimal("700") // 基本500円 + 追加200円
    }
    
    def "無料配送の条件を満たす場合"() {
        given:
        def service = new OrderCalculationService()
        def subtotal = Money.of("6000", Currency.getInstance("JPY"))
        def itemCount = 2
        
        when:
        def shippingFee = service.calculateShippingFee(subtotal, itemCount)
        
        then:
        shippingFee.isZero()
    }
    
    def "注文の詳細計算結果を取得できる"() {
        given:
        def service = new OrderCalculationService()
        def order = Order.create(
            UserId.generate(),
            [new OrderItem(ProductId.generate(), "商品A", Money.of("1000", Currency.getInstance("JPY")), 2)]
        )
        
        when:
        def result = service.calculateOrderDetails(order)
        
        then:
        result.subtotal.amount() == new BigDecimal("2000")
        result.tax.amount() == new BigDecimal("200") // 10%の消費税
        result.shippingFee.amount() == new BigDecimal("500") // 基本配送料
        result.total.amount() == new BigDecimal("2700") // 2000 + 200 + 500
    }
}
```

## 🎨 ドメイン層テストの設計指針

### 1. **ビジネスルールの検証**
- ドメインルールの正確性を保証
- 不正な状態の防止を確認
- ビジネスロジックの動作を検証

### 2. **読みやすいテスト**
- Spockフレームワークの活用
- 自然言語に近いテスト記述
- 明確なテスト意図の表現

### 3. **包括的なテストカバレッジ**
- 正常系と異常系の両方をテスト
- 境界値のテスト
- エッジケースの検証

### 4. **テストデータの管理**
- テストフィクスチャの適切な設計
- テストデータの再利用
- テストの独立性の保証

## 🔄 次のステップ

ドメイン層のテストの実装が完了したら、次はアプリケーション層の実装に進みます。アプリケーション層は、ドメイン層のオブジェクトを活用してビジネスフローの調整とオーケストレーションを行います。 