# 開発ガイド

このドキュメントは、ドメイン駆動設計（DDD）とオニオンアーキテクチャに基づいた、Spring BootとGoogle Cloud Spannerによるアプリケーション開発のための包括的なガイドです。

## 目次

### 📚 基礎編
1. [はじめに](./guide/01_introduction.md)
   - DDDとオニオンアーキテクチャの概要
   - 開発ガイドの構成と学習方法
   - 前提知識と開発環境

2. [開発環境のセットアップ](./guide/02_setup.md)
   - 必要なツールとライブラリのインストール
   - プロジェクト構造の設定
   - 開発環境の構築手順

3. [アーキテクチャ概要](./guide/03_architecture.md)
   - オニオンアーキテクチャの設計原則
   - レイヤー間の依存関係
   - ドメイン駆動設計の戦略的パターン

### 🏗️ 実装編
4. [ドメイン層の実装](./guide/04_domain_layer.md)
   - [ドメイン層の概要](./guide/04_domain_overview.md)
   - [値オブジェクト](./guide/04_value_object.md)
   - [エンティティ](./guide/04_entity.md)
   - [集約](./guide/04_aggregate.md)
   - [ドメインサービス](./guide/04_domain_service.md)
   - [リポジトリインターフェース](./guide/04_repository.md)
   - [ドメインイベント](./guide/04_domain_event.md)
   - [ファクトリパターン](./guide/04_factory.md)
   - [ドメイン層のテスト](./guide/04_domain_test.md)

5. [アプリケーション層の実装](./guide/05_application_layer.md)
   - [アプリケーション層の概要](./guide/05_application_overview.md)
   - [アプリケーションサービスの実装](./guide/05_application_service.md)
   - [コマンド・クエリオブジェクト](./guide/05_command_query.md)
   - [DTO（Data Transfer Object）](./guide/05_dto.md)
   - [オブジェクトマッピング](./guide/05_mapping.md)
   - [バリデーション](./guide/05_validation.md)
   - [例外処理](./guide/05_exception.md)
   - [アプリケーション層のテスト](./guide/05_application_test.md)

6. [インフラストラクチャ層の実装](./guide/06_infrastructure_layer.md)
   - [インフラストラクチャ層の概要](./guide/06_infrastructure_overview.md)
   - [Google Cloud Spannerとの連携](./guide/06_spanner_integration.md)
   - [エンティティマッピング](./guide/06_entity_mapping.md)
   - [RabbitMQメッセージング](./guide/06_rabbitmq_messaging.md)
   - [非同期処理](./guide/06_async_processing.md)
   - [キャッシュ戦略](./guide/06_caching.md)
   - [メトリクスとモニタリング](./guide/06_metrics.md)
   - [暗号化とセキュリティ](./guide/06_security.md)
   - [インフラストラクチャ層のテスト](./guide/06_infrastructure_test.md)

7. [プレゼンテーション層の実装](./guide/07_presentation_layer.md)
   - [REST APIコントローラの実装](./guide/07_rest_controller.md)
   - [リクエスト・レスポンスの設計](./guide/07_request_response.md)
   - [エラーハンドリング](./guide/07_error_handling.md)
   - [バリデーション](./guide/07_validation.md)
   - [API文書化](./guide/07_api_documentation.md)
   - [セキュリティ](./guide/07_security.md)

### 🧪 品質保証編
8. [テスト](./guide/08_testing.md)
   - [テスト戦略の概要](./guide/08_testing_overview.md)
   - [Spockフレームワークの基本](./guide/08_spock_basics.md)
   - [ユニットテスト](./guide/08_unit_test.md)
   - [インテグレーションテスト](./guide/08_integration_test.md)
   - [アーキテクチャテスト](./guide/08_architecture_test.md)
   - [Testcontainersによるテスト](./guide/08_testcontainers.md)
   - [テストデータ管理](./guide/08_test_data.md)

### 📖 総括編
9. [まとめと発展的なトピック](./guide/09_summary.md)
   - 学習内容の振り返り
   - 実践的な開発プロセス
   - 発展的なトピックと今後の学習方向

## 🎯 学習の進め方

### 初心者向け
1. **基礎編**（1-3章）から始めて、アーキテクチャの全体像を理解
2. **ドメイン層**（4章）でビジネスロジックの実装を学習
3. **アプリケーション層**（5章）でユースケースの実装を学習

### 中級者向け
1. **インフラストラクチャ層**（6章）で外部システムとの連携を学習
2. **プレゼンテーション層**（7章）でAPIの実装を学習
3. **テスト**（8章）で品質保証の実装を学習

### 上級者向け
1. **アーキテクチャテスト**で設計の健全性を保証
2. **発展的なトピック**で実践的な開発プロセスを学習
3. **実プロジェクトへの適用**を検討

## 🛠️ 技術スタック

- **フレームワーク**: Spring Boot 3.x
- **データベース**: Google Cloud Spanner
- **メッセージング**: RabbitMQ
- **テスト**: Spock Framework, Testcontainers, ArchUnit
- **ドキュメント**: OpenAPI (Swagger)
- **セキュリティ**: Spring Security, JWT

## 📝 開発ガイドライン

- **ドメイン駆動設計**: ビジネスロジックを中心とした設計
- **オニオンアーキテクチャ**: 依存関係の方向性を制御
- **テスト駆動開発**: 品質を保証するためのテストファースト
- **継続的改善**: リファクタリングとアーキテクチャの進化

## 🤝 貢献

このガイドの改善や拡張にご協力いただける場合は、プルリクエストやイシューの作成をお願いします。 