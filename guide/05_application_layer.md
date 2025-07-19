# 05 アプリケーション層

この章では、ドメイン層のオブジェクトを活用して、ビジネスフローの調整とオーケストレーションを行うアプリケーション層を実装します。

## 📋 章の構成

### 5.1 🎯 アプリケーション層の概要と設計原則
- [05_application_overview.md](05_application_overview.md)
  - アプリケーション層の役割と責任
  - ユースケースの実装
  - トランザクション境界の管理
  - ドメイン層との境界

### 5.2 🔧 アプリケーションサービスの実装
- [05_application_service.md](05_application_service.md)
  - 基本的なアプリケーションサービス（UserApplicationService）
  - 複雑なアプリケーションサービス（OrderApplicationService）
  - ユースケースの調整とオーケストレーション
  - トランザクション管理

### 5.3 📝 コマンド・クエリオブジェクト
- [05_command_query.md](05_command_query.md)
  - コマンドオブジェクト（CreateUserCommand, UpdateUserCommand）
  - クエリオブジェクト（UserQuery）
  - 入力値の検証とビジネス意図の明確化
  - CQRSパターンの適用

### 5.4 📦 DTO（Data Transfer Object）
- [05_dto.md](05_dto.md)
  - リクエストDTO（CreateUserRequest, UpdateUserRequest）
  - レスポンスDTO（UserResponse, OrderResponse）
  - プレゼンテーション層との境界
  - 型安全性の確保

### 5.5 🔄 オブジェクトマッピング
- [05_mapping.md](05_mapping.md)
  - MapStructを使用したマッピング
  - UserMapper, OrderMapperの実装
  - 型安全性とパフォーマンスの向上
  - カスタムマッピングの実装

### 5.6 ✅ バリデーション
- [05_validation.md](05_validation.md)
  - カスタムバリデーター（EmailValidator）
  - バリデーショングループ
  - 入力値の検証とエラーハンドリング
  - ドメインルールとの整合性

### 5.7 ⚠️ 例外処理
- [05_exception.md](05_exception.md)
  - アプリケーション例外（UserRegistrationException, UserNotFoundException）
  - 例外ハンドラー（ApplicationExceptionHandler）
  - 適切なエラー情報の提供
  - 例外の階層設計

### 5.8 🧪 アプリケーション層のテスト
- [05_application_test.md](05_application_test.md)
  - アプリケーションサービスのテスト
  - Spockフレームワークを使用したテスト
  - モックとスタブの活用
  - 統合テストの実装

## 🎯 学習のポイント

### 1. **薄いアプリケーション層**
- ビジネスロジックはドメイン層に配置し、調整のみを担当
- ユースケースの実装に集中
- ドメインオブジェクトの組み合わせによる実現

### 2. **トランザクション境界の管理**
- 一つのユースケース = 一つのトランザクション
- 集約境界を考慮したトランザクション設計
- 分散トランザクションの回避

### 3. **エラーハンドリング**
- ドメイン例外の適切な処理と技術的例外の変換
- ユーザーフレンドリーなエラーメッセージ
- ログ出力とモニタリング

### 4. **型安全性**
- コマンド・クエリオブジェクトとDTOによる型安全な設計
- コンパイル時のエラー検出
- リファクタリングの安全性

### 5. **テストの重要性**
- アプリケーションロジックの検証とユースケースの確認
- モックとスタブの適切な使用
- テストカバレッジの向上

## 📚 実装例

この章では、以下のアプリケーションサービスとコンポーネントを実装します：

### アプリケーションサービス
- **UserApplicationService**: ユーザー登録・更新・削除
- **OrderApplicationService**: 注文作成・キャンセル・確認

### コマンド・クエリオブジェクト
- **コマンド**: `CreateUserCommand`, `UpdateUserCommand`, `CreateOrderCommand`
- **クエリ**: `UserQuery`, `OrderQuery`

### DTO
- **リクエスト**: `CreateUserRequest`, `UpdateUserRequest`, `CreateOrderRequest`
- **レスポンス**: `UserResponse`, `OrderResponse`, `OrderDetailResponse`

### マッピング
- **UserMapper**: ユーザー関連のマッピング
- **OrderMapper**: 注文関連のマッピング

## 🔄 次のステップ

次の章では、インフラストラクチャ層を実装して、外部システムとの連携を実現します。Google Cloud SpannerとRabbitMQを使用したクラウドネイティブな設計について学びます。 