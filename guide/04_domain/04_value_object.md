# 4.2 値オブジェクト（Value Objects）の実装

値オブジェクトは、ドメイン駆動設計（DDD）において重要な戦術的パターンの一つです。Java 17の`record`型を使用することで、より簡潔で安全な実装が可能になります。

## 🎯 値オブジェクトの特徴

### 1. **不変性（Immutability）**
- 作成後に状態を変更できない
- スレッドセーフで副作用がない
- 予測可能な動作を保証

### 2. **値による等価性（Value-based Equality）**
- 内容が同じであれば等しいとみなす
- 識別子ではなく値で比較
- `equals()`と`hashCode()`の一貫性

### 3. **自己完結した検証ロジック**
- 作成時にバリデーションを実行
- 不正な状態のオブジェクトを作成不可能
- ドメインルールのカプセル化

## 📝 基本的な値オブジェクトの実装

### Email 値オブジェクト

```java
package com.example.domain.valueobject;

import java.util.Objects;
import java.util.regex.Pattern;

/**
 * メールアドレスを表現する値オブジェクト
 * 
 * 不変性と値による等価性を保証し、
 * メールアドレスの形式検証を行います。
 */
public record Email(String value) {
    
    // メールアドレスの正規表現パターン
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
    
    /**
     * メールアドレスを作成します
     * 
     * @param value メールアドレス文字列
     * @throws IllegalArgumentException 無効なメールアドレスの場合
     */
    public Email {
        Objects.requireNonNull(value, "メールアドレスは必須です");
        
        if (value.trim().isEmpty()) {
            throw new IllegalArgumentException("メールアドレスは空文字列にできません");
        }
        
        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new IllegalArgumentException("無効なメールアドレス形式です: " + value);
        }
        
        // 正規化（小文字に変換）
        value = value.toLowerCase().trim();
    }
    
    /**
     * メールアドレスのローカル部分を取得
     */
    public String getLocalPart() {
        return value.substring(0, value.indexOf('@'));
    }
    
    /**
     * メールアドレスのドメイン部分を取得
     */
    public String getDomain() {
        return value.substring(value.indexOf('@') + 1);
    }
    
    /**
     * メールアドレスが特定のドメインかどうかを判定
     */
    public boolean isFromDomain(String domain) {
        return getDomain().equalsIgnoreCase(domain);
    }
    
    @Override
    public String toString() {
        return value;
    }
}
```

### FullName 値オブジェクト

```java
package com.example.domain.valueobject;

import java.util.Objects;

/**
 * 氏名を表現する値オブジェクト
 * 
 * 姓と名を分けて管理し、表示用の形式を提供します。
 */
public record FullName(String lastName, String firstName) {
    
    /**
     * 氏名を作成します
     * 
     * @param lastName 姓
     * @param firstName 名
     * @throws IllegalArgumentException 無効な氏名の場合
     */
    public FullName {
        Objects.requireNonNull(lastName, "姓は必須です");
        Objects.requireNonNull(firstName, "名は必須です");
        
        if (lastName.trim().isEmpty()) {
            throw new IllegalArgumentException("姓は空文字列にできません");
        }
        
        if (firstName.trim().isEmpty()) {
            throw new IllegalArgumentException("名は空文字列にできません");
        }
        
        // 正規化（前後の空白を除去）
        lastName = lastName.trim();
        firstName = firstName.trim();
    }
    
    /**
     * フルネームを取得（姓 名）
     */
    public String getFullName() {
        return lastName + " " + firstName;
    }
    
    /**
     * 英語表記のフルネームを取得（名 姓）
     */
    public String getEnglishFullName() {
        return firstName + " " + lastName;
    }
    
    /**
     * イニシャルを取得
     */
    public String getInitials() {
        return lastName.charAt(0) + "." + firstName.charAt(0) + ".";
    }
    
    @Override
    public String toString() {
        return getFullName();
    }
}
```

## 💰 複雑な値オブジェクトの実装

### Money 値オブジェクト

```java
package com.example.domain.valueobject;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;

/**
 * 金額を表現する値オブジェクト
 * 
 * 通貨と金額を組み合わせて、正確な計算を保証します。
 * BigDecimalを使用して浮動小数点の誤差を回避します。
 */
public record Money(BigDecimal amount, Currency currency) {
    
    /**
     * 金額を作成します
     * 
     * @param amount 金額
     * @param currency 通貨
     * @throws IllegalArgumentException 無効な金額の場合
     */
    public Money {
        Objects.requireNonNull(amount, "金額は必須です");
        Objects.requireNonNull(currency, "通貨は必須です");
        
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金額は負の値にできません");
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
     * 指定した通貨で金額を作成
     */
    public static Money of(BigDecimal amount, Currency currency) {
        return new Money(amount, currency);
    }
    
    /**
     * 指定した通貨で金額を作成（文字列から）
     */
    public static Money of(String amount, Currency currency) {
        return new Money(new BigDecimal(amount), currency);
    }
    
    /**
     * 金額を加算
     */
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("異なる通貨間での計算はできません");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    /**
     * 金額を減算
     */
    public Money subtract(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("異なる通貨間での計算はできません");
        }
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("結果が負の値になります");
        }
        return new Money(result, this.currency);
    }
    
    /**
     * 金額を乗算
     */
    public Money multiply(BigDecimal factor) {
        if (factor.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("乗数は負の値にできません");
        }
        return new Money(this.amount.multiply(factor), this.currency);
    }
    
    /**
     * 金額を比較
     */
    public boolean isGreaterThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("異なる通貨間での比較はできません");
        }
        return this.amount.compareTo(other.amount) > 0;
    }
    
    /**
     * 金額がゼロかどうかを判定
     */
    public boolean isZero() {
        return amount.compareTo(BigDecimal.ZERO) == 0;
    }
    
    /**
     * 表示用の文字列を取得
     */
    public String getDisplayString() {
        return currency.getSymbol() + amount.toString();
    }
    
    @Override
    public String toString() {
        return getDisplayString();
    }
}
```

## 🎨 値オブジェクトの設計指針

### 1. **不変性の保証**
- フィールドを`final`にする
- セッターメソッドを提供しない
- 状態変更メソッドは新しいインスタンスを返す

### 2. **値による等価性の実装**
- `equals()`と`hashCode()`を適切に実装
- 識別子ではなく内容で比較
- `record`型を使用すると自動実装される

### 3. **検証ロジックの集約**
- 作成時にバリデーションを実行
- 不正な状態のオブジェクトを作成不可能にする
- ドメインルールをカプセル化

### 4. **自己完結した設計**
- 外部依存を最小限に抑える
- 必要なロジックを内部に持つ
- 単一責任の原則を守る

## 🧪 値オブジェクトのテスト

```groovy
package com.example.domain.valueobject

import spock.lang.Specification
import spock.lang.Unroll

class EmailTest extends Specification {
    
    @Unroll
    def "有効なメールアドレス #email を作成できる"() {
        when:
        def email = new Email(email)
        
        then:
        email.value() == expectedValue
        email.getLocalPart() == expectedLocalPart
        email.getDomain() == expectedDomain
        
        where:
        email                    | expectedValue              | expectedLocalPart | expectedDomain
        "test@example.com"      | "test@example.com"        | "test"            | "example.com"
        "USER@EXAMPLE.COM"      | "user@example.com"        | "user"            | "example.com"
        "test.user@domain.co.jp"| "test.user@domain.co.jp" | "test.user"       | "domain.co.jp"
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
            "test..user@example.com"
        ]
    }
    
    def "特定のドメインかどうかを判定できる"() {
        given:
        def email = new Email("test@example.com")
        
        expect:
        email.isFromDomain("example.com")
        !email.isFromDomain("other.com")
    }
}

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
    
    def "異なる通貨間での加算で例外が発生する"() {
        given:
        def jpyMoney = Money.of(new BigDecimal("100"), Currency.getInstance("JPY"))
        def usdMoney = Money.of(new BigDecimal("1"), Currency.getInstance("USD"))
        
        when:
        jpyMoney.add(usdMoney)
        
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
}
```

## 🔄 次のステップ

値オブジェクトの実装が完了したら、次はエンティティの実装に進みます。エンティティは識別子を持ち、ライフサイクルを管理する重要なドメインオブジェクトです。 