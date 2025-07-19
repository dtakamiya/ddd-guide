# 06 インフラストラクチャ層

この章では、外部システムとの連携とデータの永続化を担当するインフラストラクチャ層を実装します。Google Cloud SpannerとRabbitMQを使用したクラウドネイティブな設計について学びます。

## 📋 章の構成

### 6.1 🎯 インフラストラクチャ層の概要と設計原則
- [06_infrastructure_overview.md](06_infrastructure_overview.md)
  - インフラストラクチャ層の役割と責任
  - クラウドネイティブ設計の原則
  - 疎結合と耐障害性の実現
  - アーキテクチャ境界の維持

### 6.2 🗄️ Google Cloud Spannerとの連携
- [06_spanner_integration.md](06_spanner_integration.md)
  - Spring Data Spannerの設定
  - カスタムコンバーターの実装
  - リポジトリの実装
  - 分散トランザクションの活用

### 6.3 🔄 エンティティマッピング
- [06_entity_mapping.md](06_entity_mapping.md)
  - ドメインオブジェクトと永続化オブジェクトの分離
  - UserEntityMapper, OrderEntityMapperの実装
  - クエリの最適化
  - パフォーマンスチューニング

### 6.4 📨 RabbitMQメッセージング
- [06_rabbitmq_messaging.md](06_rabbitmq_messaging.md)
  - RabbitMQ設定とメッセージング
  - メッセージプロデューサーとコンシューマー
  - デッドレターキューとリトライ戦略
  - メッセージングパターンの実装

### 6.5 ⚡ 非同期処理
- [06_async_processing.md](06_async_processing.md)
  - Springの@Asyncを使用した非同期処理
  - スレッドプール設定
  - 非同期サービスの実装
  - 非同期処理の監視

### 6.6 💾 キャッシュ戦略
- [06_caching.md](06_caching.md)
  - Spring Cache抽象化
  - Redisキャッシュの実装
  - キャッシュ戦略の設計
  - キャッシュの無効化と更新

### 6.7 📊 メトリクスとモニタリング
- [06_metrics.md](06_metrics.md)
  - Micrometerを使用したメトリクス収集
  - カスタムメトリクスの実装
  - モニタリング設定
  - アラートとダッシュボード

### 6.8 🔒 暗号化とセキュリティ
- [06_security.md](06_security.md)
  - データ暗号化の実装
  - セキュリティ設定
  - 認証・認可の実装
  - セキュリティ監査

### 6.9 🧪 インフラストラクチャ層のテスト
- [06_infrastructure_test.md](06_infrastructure_test.md)
  - Testcontainersを使用した統合テスト
  - リポジトリのテスト
  - メッセージングのテスト
  - パフォーマンステスト

## 🎯 学習のポイント

### 1. **疎結合の実現**
- 外部システムへの直接依存を避け、インターフェースによる抽象化
- 依存性注入によるテスタビリティの向上
- アダプターパターンの活用

### 2. **耐障害性**
- リトライ機能、サーキットブレーカーパターン、フォールバック機能
- 分散システムでの障害対応
- 監視とログ出力の重要性

### 3. **スケーラビリティ**
- 水平スケーリングへの対応と非同期処理の活用
- データベースのスケーリング戦略
- メッセージングによる負荷分散

### 4. **クラウドネイティブ**
- Google Cloud SpannerとRabbitMQを活用した設計
- マネージドサービスの活用
- クラウドネイティブな設計原則

### 5. **テストの重要性**
- Testcontainersを使用した統合テストとモニタリング
- 本番環境に近いテスト環境の構築
- パフォーマンステストの実施

## 📚 実装例

この章では、以下のインフラストラクチャコンポーネントを実装します：

### データベース層
- **UserRepositoryImpl**: ユーザー情報の永続化
- **OrderRepositoryImpl**: 注文情報の永続化
- **ProductRepositoryImpl**: 商品情報の永続化

### メッセージング層
- **OrderEventPublisher**: 注文イベントの発行
- **OrderEventConsumer**: 注文イベントの消費
- **NotificationService**: 通知サービスの実装

### キャッシュ層
- **UserCacheService**: ユーザー情報のキャッシュ
- **ProductCacheService**: 商品情報のキャッシュ
- **CacheConfiguration**: キャッシュ設定

### セキュリティ層
- **EncryptionService**: データ暗号化サービス
- **SecurityConfiguration**: セキュリティ設定
- **AuditService**: 監査サービス

## 🔄 次のステップ

次の章では、プレゼンテーション層を実装して、RESTful APIとWebインターフェースを提供します。Spring MVCを利用したREST APIの実装に焦点を当てます。 