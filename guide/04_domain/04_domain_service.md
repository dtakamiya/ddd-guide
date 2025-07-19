# 4.5 ドメインサービス（Domain Services）の実装

ドメインサービスは、複数の集約にまたがるビジネスロジックや、単一のエンティティに属さない複雑なドメインルールを実装するためのパターンです。副作用のない設計を心がけ、ドメイン層に配置します。

## 🎯 ドメインサービスの特徴

### 1. **複数集約にまたがるロジック**
- 単一の集約では実装できないビジネスルール
- 複数の集約の状態を考慮した処理
- ドメイン知識の集約

### 2. **副作用のない設計**
- 純粋な関数として実装
- 外部依存を最小限に抑制
- テストしやすい設計

### 3. **ドメイン層での配置**
- インフラストラクチャ層に依存しない
- ドメインルールのカプセル化
- 再利用可能な設計

## 🔍 ユーザー重複チェックサービス

### UserDuplicateChecker ドメインサービス

```java
package com.example.domain.service;

import com.example.domain.entity.User;
import com.example.domain.valueobject.Email;

import java.util.List;

/**
 * ユーザー重複チェックを行うドメインサービス
 * 
 * メールアドレスの重複チェックや、
 * 類似ユーザーの検出を行います。
 */
public class UserDuplicateChecker {
    
    /**
     * メールアドレスの重複をチェック
     * 
     * @param email チェック対象のメールアドレス
     * @param existingUsers 既存のユーザーリスト
     * @return 重複している場合はtrue
     */
    public boolean isEmailDuplicate(Email email, List<User> existingUsers) {
        if (email == null || existingUsers == null) {
            return false;
        }
        
        return existingUsers.stream()
                .anyMatch(user -> user.getEmail().equals(email));
    }
    
    /**
     * 類似ユーザーを検出
     * 
     * @param targetUser 対象ユーザー
     * @param existingUsers 既存のユーザーリスト
     * @return 類似ユーザーのリスト
     */
    public List<User> findSimilarUsers(User targetUser, List<User> existingUsers) {
        if (targetUser == null || existingUsers == null) {
            return List.of();
        }
        
        return existingUsers.stream()
                .filter(user -> !user.getId().equals(targetUser.getId()))
                .filter(user -> isSimilarUser(targetUser, user))
                .toList();
    }
    
    /**
     * ユーザーが類似しているかどうかを判定
     */
    private boolean isSimilarUser(User user1, User user2) {
        // 同じドメインのメールアドレス
        boolean sameDomain = user1.getEmail().getDomain().equals(user2.getEmail().getDomain());
        
        // 類似した氏名（部分一致）
        boolean similarName = isSimilarName(user1.getFullName(), user2.getFullName());
        
        return sameDomain || similarName;
    }
    
    /**
     * 氏名が類似しているかどうかを判定
     */
    private boolean isSimilarName(FullName name1, FullName name2) {
        String fullName1 = name1.getFullName().toLowerCase();
        String fullName2 = name2.getFullName().toLowerCase();
        
        // 完全一致
        if (fullName1.equals(fullName2)) {
            return true;
        }
        
        // 部分一致（姓または名が一致）
        return fullName1.contains(fullName2) || fullName2.contains(fullName1);
    }
    
    /**
     * 重複リスクスコアを計算
     * 
     * @param user 対象ユーザー
     * @param existingUsers 既存のユーザーリスト
     * @return 0.0-1.0のスコア（高いほど重複リスクが高い）
     */
    public double calculateDuplicateRiskScore(User user, List<User> existingUsers) {
        if (user == null || existingUsers == null) {
            return 0.0;
        }
        
        long exactEmailMatches = existingUsers.stream()
                .filter(existing -> existing.getEmail().equals(user.getEmail()))
                .count();
        
        long similarUserCount = findSimilarUsers(user, existingUsers).size();
        
        // 重複リスクスコアの計算
        double emailRisk = exactEmailMatches > 0 ? 1.0 : 0.0;
        double similarityRisk = Math.min(similarUserCount * 0.2, 0.8);
        
        return Math.max(emailRisk, similarityRisk);
    }
}
```

## 💰 注文計算サービス

### OrderCalculationService ドメインサービス

```java
package com.example.domain.service;

import com.example.domain.aggregate.Order;
import com.example.domain.entity.OrderItem;
import com.example.domain.valueobject.Money;

import java.math.BigDecimal;
import java.util.List;

/**
 * 注文計算を行うドメインサービス
 * 
 * 割引計算、税計算、配送料計算など、
 * 複雑な計算ロジックを実装します。
 */
public class OrderCalculationService {
    
    /**
     * 小計を計算
     */
    public Money calculateSubtotal(List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            return Money.zero(Currency.getInstance("JPY"));
        }
        
        Money subtotal = Money.zero(items.get(0).getUnitPrice().currency());
        for (OrderItem item : items) {
            subtotal = subtotal.add(item.getTotalPrice());
        }
        
        return subtotal;
    }
    
    /**
     * 消費税を計算
     * 
     * @param subtotal 小計
     * @param taxRate 税率（例：0.1 = 10%）
     * @return 消費税額
     */
    public Money calculateTax(Money subtotal, BigDecimal taxRate) {
        if (subtotal == null || taxRate == null) {
            return Money.zero(subtotal.currency());
        }
        
        if (taxRate.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("税率は負の値にできません");
        }
        
        return subtotal.multiply(taxRate);
    }
    
    /**
     * 割引を計算
     * 
     * @param subtotal 小計
     * @param discountRate 割引率（例：0.1 = 10%割引）
     * @return 割引額
     */
    public Money calculateDiscount(Money subtotal, BigDecimal discountRate) {
        if (subtotal == null || discountRate == null) {
            return Money.zero(subtotal.currency());
        }
        
        if (discountRate.compareTo(BigDecimal.ZERO) < 0 || discountRate.compareTo(BigDecimal.ONE) > 0) {
            throw new IllegalArgumentException("割引率は0.0-1.0の範囲である必要があります");
        }
        
        return subtotal.multiply(discountRate);
    }
    
    /**
     * 配送料を計算
     * 
     * @param subtotal 小計
     * @param itemCount 商品数
     * @return 配送料
     */
    public Money calculateShippingFee(Money subtotal, int itemCount) {
        if (subtotal == null) {
            return Money.zero(Currency.getInstance("JPY"));
        }
        
        Currency currency = subtotal.currency();
        
        // 無料配送の条件
        if (subtotal.amount().compareTo(new BigDecimal("5000")) >= 0) {
            return Money.zero(currency);
        }
        
        // 基本配送料
        Money baseShippingFee = Money.of("500", currency);
        
        // 商品数による追加料金
        Money additionalFee = Money.of("100", currency).multiply(new BigDecimal(Math.max(0, itemCount - 1)));
        
        return baseShippingFee.add(additionalFee);
    }
    
    /**
     * 総額を計算
     * 
     * @param subtotal 小計
     * @param tax 消費税
     * @param discount 割引
     * @param shippingFee 配送料
     * @return 総額
     */
    public Money calculateTotal(Money subtotal, Money tax, Money discount, Money shippingFee) {
        if (subtotal == null) {
            return Money.zero(Currency.getInstance("JPY"));
        }
        
        Currency currency = subtotal.currency();
        
        // nullの場合はゼロ金額として扱う
        Money safeTax = tax != null ? tax : Money.zero(currency);
        Money safeDiscount = discount != null ? discount : Money.zero(currency);
        Money safeShippingFee = shippingFee != null ? shippingFee : Money.zero(currency);
        
        return subtotal.add(safeTax).subtract(safeDiscount).add(safeShippingFee);
    }
    
    /**
     * 注文の詳細計算結果を取得
     */
    public OrderCalculationResult calculateOrderDetails(Order order) {
        if (order == null) {
            return null;
        }
        
        List<OrderItem> items = order.getItems();
        Money subtotal = calculateSubtotal(items);
        
        // 消費税計算（10%）
        Money tax = calculateTax(subtotal, new BigDecimal("0.1"));
        
        // 配送料計算
        Money shippingFee = calculateShippingFee(subtotal, items.size());
        
        // 割引計算（条件に応じて）
        Money discount = calculateDiscountByOrderCondition(order, subtotal);
        
        // 総額計算
        Money total = calculateTotal(subtotal, tax, discount, shippingFee);
        
        return new OrderCalculationResult(
            subtotal,
            tax,
            discount,
            shippingFee,
            total
        );
    }
    
    /**
     * 注文条件に基づく割引を計算
     */
    private Money calculateDiscountByOrderCondition(Order order, Money subtotal) {
        // 初回注文割引（例：1000円以上で10%割引）
        if (isFirstOrder(order) && subtotal.amount().compareTo(new BigDecimal("1000")) >= 0) {
            return calculateDiscount(subtotal, new BigDecimal("0.1"));
        }
        
        // 大量注文割引（例：5000円以上で5%割引）
        if (subtotal.amount().compareTo(new BigDecimal("5000")) >= 0) {
            return calculateDiscount(subtotal, new BigDecimal("0.05"));
        }
        
        return Money.zero(subtotal.currency());
    }
    
    /**
     * 初回注文かどうかを判定（簡易実装）
     */
    private boolean isFirstOrder(Order order) {
        // 実際の実装では、ユーザーの注文履歴を確認
        return true; // 簡易実装
    }
}
```

### OrderCalculationResult 値オブジェクト

```java
package com.example.domain.service;

import com.example.domain.valueobject.Money;

/**
 * 注文計算結果を表現する値オブジェクト
 */
public record OrderCalculationResult(
    Money subtotal,
    Money tax,
    Money discount,
    Money shippingFee,
    Money total
) {
    
    public OrderCalculationResult {
        // バリデーション
        if (subtotal == null || tax == null || discount == null || 
            shippingFee == null || total == null) {
            throw new IllegalArgumentException("計算結果の各項目は必須です");
        }
        
        // 通貨の一致を確認
        Currency currency = subtotal.currency();
        if (!tax.currency().equals(currency) || 
            !discount.currency().equals(currency) ||
            !shippingFee.currency().equals(currency) ||
            !total.currency().equals(currency)) {
            throw new IllegalArgumentException("通貨が一致しません");
        }
    }
    
    /**
     * 割引後の金額を取得
     */
    public Money getDiscountedSubtotal() {
        return subtotal.subtract(discount);
    }
    
    /**
     * 税込小計を取得
     */
    public Money getSubtotalWithTax() {
        return subtotal.add(tax);
    }
    
    /**
     * 割引率を計算
     */
    public BigDecimal getDiscountRate() {
        if (subtotal.amount().compareTo(BigDecimal.ZERO) == 0) {
            return BigDecimal.ZERO;
        }
        return discount.amount().divide(subtotal.amount(), 4, RoundingMode.HALF_UP);
    }
    
    @Override
    public String toString() {
        return String.format("OrderCalculationResult{subtotal=%s, tax=%s, discount=%s, shippingFee=%s, total=%s}", 
                           subtotal, tax, discount, shippingFee, total);
    }
}
```

## 🎨 ドメインサービスの設計指針

### 1. **副作用のない設計**
- 純粋な関数として実装
- 外部状態に依存しない
- 同じ入力に対して同じ出力を保証

### 2. **ドメイン層での配置**
- インフラストラクチャ層に依存しない
- ドメインルールのカプセル化
- 再利用可能な設計

### 3. **適切な粒度の設計**
- 単一責任の原則を守る
- 過度に複雑にしない
- テストしやすい設計

### 4. **型安全性の確保**
- 強力な型システムを活用
- コンパイル時のエラー検出
- 実行時エラーの最小化

## 🧪 ドメインサービスのテスト

```groovy
package com.example.domain.service

import com.example.domain.entity.User
import com.example.domain.valueobject.Email
import com.example.domain.valueobject.FullName
import com.example.domain.valueobject.Money
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
}

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
}
```

## 🔄 次のステップ

ドメインサービスの実装が完了したら、次はリポジトリインターフェースの実装に進みます。リポジトリは集約の永続化を抽象化し、ドメイン層とインフラストラクチャ層の境界を定義します。 