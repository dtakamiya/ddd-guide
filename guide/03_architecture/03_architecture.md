## 3. アーキテクチャ概要

この章では、ドメイン駆動設計（DDD）とオニオンアーキテクチャの基本概念について説明します。これらの概念は、複雑なビジネス要件に対応できる、保守性・拡張性の高いアプリケーションを構築するための基盤となります。

### 3.1. ドメイン駆動設計（DDD）の基本概念

#### 3.1.1. ドメインとは

**ドメイン**とは、アプリケーションが解決しようとするビジネス領域のことです。例えば、ECサイトであれば「商品管理」「注文処理」「顧客管理」などがドメインとなります。

DDDでは、このドメインの知識を深く理解し、その知識をソフトウェアの設計に反映させることを重視します。

#### 3.1.2. ユビキタス言語（Ubiquitous Language）

**ユビキタス言語**とは、ビジネス担当者と開発者が共通して使用する用語体系のことです。これにより、以下のメリットが得られます：

*   **コミュニケーションの改善**: ビジネス担当者と開発者が同じ用語を使うことで、誤解を減らす
*   **モデルの一貫性**: ソフトウェアのモデルがビジネスの概念と一致する
*   **保守性の向上**: コードの意図が明確になり、理解しやすくなる

**例：ECサイトのユビキタス言語**
```
ビジネス用語 → ソフトウェア用語
商品 → Product
在庫 → Inventory
注文 → Order
顧客 → Customer
```

#### 3.1.3. 境界づけられたコンテキスト（Bounded Context）

大きなシステムを、一貫したモデルを持つ小さな単位に分割することを**境界づけられたコンテキスト**と呼びます。各コンテキストは、独自のユビキタス言語とドメインモデルを持ちます。

**例：ECサイトの境界づけられたコンテキスト**

```mermaid
graph TB
    subgraph "商品管理Context"
        A[Product]
        B[Category]
        C[Inventory]
    end
    
    subgraph "注文処理Context"
        D[Order]
        E[OrderItem]
        F[Payment]
    end
    
    subgraph "顧客管理Context"
        G[Customer]
        H[Address]
        I[Preference]
    end
```

### 3.2. オニオンアーキテクチャ

#### 3.2.1. オニオンアーキテクチャとは

**オニオンアーキテクチャ**は、依存関係の方向を制御することで、保守性の高いシステムを構築するためのアーキテクチャパターンです。中心にドメイン層を置き、外側に向かって依存関係が向かう構造になっています。

#### 3.2.2. レイヤー構成

オニオンアーキテクチャは、以下の4つのレイヤーで構成されます：

```mermaid
graph TB
    subgraph "プレゼンテーション層"
        PL[Presentation Layer]
        PL --> AL
    end
    
    subgraph "アプリケーション層"
        AL[Application Layer]
        AL --> DL
    end
    
    subgraph "インフラストラクチャ層"
        IL[Infrastructure Layer]
        IL --> DL
        IL --> AL
    end
    
    subgraph "ドメイン層"
        DL[Domain Layer]
        subgraph "Domain Components"
            E[Entities]
            VO[Value Objects]
            AG[Aggregates]
            DS[Domain Services]
        end
    end
    
    style DL fill:#e1f5fe
    style AL fill:#f3e5f5
    style IL fill:#e8f5e8
    style PL fill:#fff3e0
```

#### 3.2.3. 各レイヤーの役割

**1. ドメイン層（Domain Layer）**
*   **役割**: ビジネスの核心的なロジックとルールを担当
*   **構成要素**: エンティティ、値オブジェクト、集約、ドメインサービス
*   **依存関係**: 他のレイヤーに依存しない（依存関係の方向の中心）

**2. アプリケーション層（Application Layer）**
*   **役割**: ユースケースの調整とオーケストレーション
*   **構成要素**: アプリケーションサービス、コマンド、クエリ
*   **依存関係**: ドメイン層のみに依存

**3. インフラストラクチャ層（Infrastructure Layer）**
*   **役割**: 外部システムとの連携（データベース、メッセージング、外部API）
*   **構成要素**: リポジトリ実装、メッセージングアダプター、設定
*   **依存関係**: ドメイン層とアプリケーション層に依存

**4. プレゼンテーション層（Presentation Layer）**
*   **役割**: ユーザーインターフェース（REST API、Web UI）
*   **構成要素**: コントローラー、DTO、セキュリティ設定
*   **依存関係**: アプリケーション層のみに依存

#### 3.2.4. 依存関係のルール

オニオンアーキテクチャでは、以下の依存関係ルールを厳格に守ります：

```mermaid
graph LR
    subgraph "依存関係の方向"
        A[内側のレイヤー] --> B[外側のレイヤー]
        C[外側のレイヤー] -.->|禁止| D[内側のレイヤー]
    end
    
    style A fill:#e1f5fe
    style B fill:#fff3e0
    style C fill:#ffebee
    style D fill:#ffebee
```

**具体例：**
*   ✅ ドメイン層は他のレイヤーに依存しない
*   ✅ アプリケーション層はドメイン層のみに依存
*   ✅ インフラストラクチャ層はドメイン層とアプリケーション層に依存
*   ✅ プレゼンテーション層はアプリケーション層のみに依存
*   ❌ プレゼンテーション層がドメイン層に直接依存してはいけない

### 3.3. 他のアーキテクチャパターンとの比較

#### 3.3.1. レイヤードアーキテクチャ

**特徴：**
*   水平方向のレイヤー分割
*   各レイヤーが上下のレイヤーに依存
*   シンプルだが、依存関係の制御が困難

**問題点：**
*   ビジネスロジックが技術的な詳細に埋もれる
*   テストが困難
*   変更の影響範囲が大きい

#### 3.3.2. ヘキサゴナルアーキテクチャ（ポート・アダプター）

**特徴：**
*   中心にビジネスロジック
*   外部との連携はポートとアダプターで抽象化
*   依存関係の方向を制御

**オニオンアーキテクチャとの関係：**
*   概念的に非常に似ている
*   オニオンアーキテクチャはヘキサゴナルアーキテクチャの一種
*   より具体的な実装指針を提供

#### 3.3.3. クリーンアーキテクチャ

**特徴：**
*   Robert C. Martinが提唱
*   依存関係の方向を厳格に制御
*   フレームワークに依存しない設計

**オニオンアーキテクチャとの関係：**
*   同じ原則に基づく
*   オニオンアーキテクチャはクリーンアーキテクチャの実装パターン

### 3.4. DDDの戦術的設計パターン

#### 3.4.1. エンティティ（Entity）

**定義**: 識別子を持ち、時間とともに変化するオブジェクト

**特徴：**
*   一意の識別子を持つ
*   同一性（Identity）で識別される
*   状態が変化しても同じオブジェクトとして扱われる

**例：**
```java
public class User {
    private UserId id;        // 識別子
    private String name;      // 属性
    private Email email;      // 属性
    
    // 同一性はIDで判断
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        User user = (User) obj;
        return Objects.equals(id, user.id);
    }
}
```

#### 3.4.2. 値オブジェクト（Value Object）

**定義**: 属性の値によって識別される不変オブジェクト

**特徴：**
*   不変（Immutable）
*   属性の値で等価性を判断
*   副作用がない

**例：**
```java
public record Email(String value) {
    public Email {
        if (value == null || !value.contains("@")) {
            throw new IllegalArgumentException("Invalid email format");
        }
    }
    
    // 値による等価性
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Email email = (Email) obj;
        return Objects.equals(value, email.value);
    }
}
```

#### 3.4.3. 集約（Aggregate）

**定義**: 一貫性の境界を持つエンティティのグループ

**特徴：**
*   集約ルート（Aggregate Root）が外部との唯一のインターフェース
*   集約内のオブジェクトは直接外部からアクセスできない
*   トランザクション境界となる

**例：**
```java
public class Order {  // 集約ルート
    private OrderId id;
    private List<OrderItem> items;
    private OrderStatus status;
    
    // 集約内のオブジェクトへのアクセスは集約ルート経由
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        items.add(new OrderItem(product, quantity));
    }
    
    public void confirm() {
        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        status = OrderStatus.CONFIRMED;
    }
}
```

#### 3.4.4. リポジトリ（Repository）

**定義**: 集約の永続化を抽象化するインターフェース

**特徴：**
*   ドメイン層に定義される
*   インフラストラクチャ層で実装される
*   集約単位での操作を提供

**例：**
```java
// ドメイン層
public interface UserRepository {
    User findById(UserId id);
    void save(User user);
    void delete(UserId id);
    List<User> findByEmail(Email email);
}

// インフラストラクチャ層
@Repository
public class SpannerUserRepository implements UserRepository {
    // 実装
}
```

#### 3.4.5. ドメインサービス（Domain Service）

**定義**: エンティティや値オブジェクトに属さないビジネスロジック

**特徴：**
*   複数の集約にまたがるロジック
*   ドメイン層に配置
*   副作用がない

**例：**
```java
public class OrderCalculationService {
    public Money calculateTotal(Order order) {
        return order.getItems().stream()
            .map(item -> item.getPrice().multiply(item.getQuantity()))
            .reduce(Money.ZERO, Money::add);
    }
}
```

### 3.5. イベント駆動アーキテクチャ

#### 3.5.1. ドメインイベント

**定義**: ドメイン内で発生した重要な出来事を表現するオブジェクト

**特徴：**
*   過去形で命名（例：`UserCreated`、`OrderConfirmed`）
*   不変オブジェクト
*   集約ルートから発行される

**例：**
```java
public record UserCreated(
    UserId userId,
    String name,
    Email email,
    Instant occurredAt
) implements DomainEvent {}
```

#### 3.5.2. イベント駆動設計のメリット

*   **疎結合**: システム間の依存関係を減らす
*   **拡張性**: 新しい機能を既存システムに影響を与えずに追加
*   **監査性**: システムの動作履歴を追跡可能
*   **リアクティブ**: イベントに基づいて自動的に処理を実行

### 3.6. CQRS（Command Query Responsibility Segregation）

#### 3.6.1. CQRSとは

**CQRS**は、システムの責務を「コマンド（書き込み）」と「クエリ（読み取り）」に明確に分離する設計パターンです。

#### 3.6.2. CQRSの構成

```mermaid
graph LR
    subgraph "コマンド側 (Write)"
        C[正規化モデル]
        D[整合性重視]
        E[トランザクション]
    end
    
    subgraph "クエリ側 (Read)"
        F[非正規化モデル]
        G[パフォーマンス]
        H[読み取り最適化]
    end
    
    C --> F
    D --> G
    E --> H
```

#### 3.6.3. CQRSのメリット

*   **パフォーマンス**: 読み取りと書き込みを独立して最適化
*   **スケーラビリティ**: 読み取りと書き込みを別々にスケール
*   **柔軟性**: 読み取りモデルを用途に応じて最適化

### 3.7. 実践的な設計指針

#### 3.7.1. 境界づけられたコンテキストの設計

**1. コンテキストマップの作成**

```mermaid
graph TB
    subgraph "商品管理Context"
        A[Product]
        B[Category]
        C[Inventory]
    end
    
    subgraph "注文処理Context"
        D[Order]
        E[OrderItem]
        F[Payment]
    end
    
    subgraph "顧客管理Context"
        G[Customer]
        H[Address]
        I[Preference]
    end
    
    subgraph "共有Kernel Context"
        J[Shared Kernel]
    end
    
    A --> J
    D --> J
    G --> J
```

**2. コンテキスト間の関係**
*   **Customer/Supplier**: 顧客と供給者の関係
*   **Conformist**: 既存システムに合わせる
*   **Anticorruption Layer**: 腐敗を防ぐ層
*   **Open Host Service**: 公開ホストサービス
*   **Published Language**: 公開言語

#### 3.7.2. 集約設計のベストプラクティス

**1. 集約のサイズ**
*   小さすぎる集約：パフォーマンスの問題
*   大きすぎる集約：一貫性の問題
*   適切なサイズ：ビジネスルールに基づいて決定

**2. 集約間の関係**

```mermaid
graph LR
    subgraph "集約A"
        A1[Aggregate Root A]
        A2[Entity A1]
        A3[Entity A2]
    end
    
    subgraph "集約B"
        B1[Aggregate Root B]
        B2[Entity B1]
        B3[Entity B2]
    end
    
    A1 -.->|ID参照のみ| B1
    A2 -.->|直接参照禁止| B2
```

*   集約間はID参照のみ
*   直接的なオブジェクト参照は避ける
*   イベントによる疎結合

#### 3.7.3. ドメインイベントの設計

**1. イベントの命名**
*   過去形で命名
*   ビジネス的な意味を持つ
*   具体的で明確

**2. イベントの構造**
*   不変オブジェクト
*   必要な情報のみを含む
*   バージョニングを考慮

### 3.8. まとめ

この章では、DDDとオニオンアーキテクチャの基本概念について説明しました。これらの概念を理解することで、以下のようなメリットが得られます：

*   **保守性の向上**: 明確な責任分離と依存関係の制御
*   **拡張性の確保**: 新しい機能の追加が容易
*   **テスト容易性**: 各レイヤーを独立してテスト可能
*   **ビジネス価値の最大化**: ビジネスロジックの明確な表現

次の章では、これらの概念を実際のコードで実装していきます。 