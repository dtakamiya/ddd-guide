# 07 プレゼンテーション層

この章では、ユーザーや外部システムとのインターフェースを担当するプレゼンテーション層を実装します。Spring MVCを利用したREST APIの実装に焦点を当てます。

## 📋 章の構成

### 7.1 🎯 REST APIコントローラの実装
- [07_rest_controller.md](07_rest_controller.md)
  - Spring MVCを使用したRESTful APIの実装
  - コントローラの責務と設計原則
  - HTTPリクエスト・レスポンスの処理
  - RESTful設計原則の適用

### 7.2 📦 リクエスト・レスポンスの設計
- [07_request_response.md](07_request_response.md)
  - DTO（Data Transfer Object）の設計
  - HATEOASによるAPIの自己文書化
  - レスポンスアセンブラーの実装
  - APIバージョニング戦略

### 7.3 ⚠️ エラーハンドリング
- [07_error_handling.md](07_error_handling.md)
  - グローバル例外ハンドラーの実装
  - エラーレスポンスの標準化
  - カスタムビジネス例外の定義
  - エラーログとモニタリング

### 7.4 ✅ バリデーション
- [07_validation.md](07_validation.md)
  - Bean Validationの活用
  - カスタムバリデーターの実装
  - バリデーションエラーの処理
  - 入力値の検証とセキュリティ

### 7.5 📚 API文書化
- [07_api_documentation.md](07_api_documentation.md)
  - OpenAPI（Swagger）の設定
  - API仕様書の自動生成
  - テストケースの文書化
  - APIガバナンス

### 7.6 🔒 セキュリティ
- [07_security.md](07_security.md)
  - Spring Securityの設定
  - JWT認証の実装
  - CORS設定とセキュリティヘッダー
  - 認可とアクセス制御

### 7.7 🧪 テスト
- [07_presentation_test.md](07_presentation_test.md)
  - コントローラの単体テスト
  - 統合テストの実装
  - APIテストの自動化
  - セキュリティテスト

## 🎯 学習のポイント

### 1. **関心の分離**
- プレゼンテーション層はアプリケーション層のみに依存し、ドメイン層に直接依存しない
- コントローラは薄く保ち、ビジネスロジックは下位層に委譲
- インターフェースの一貫性を保つ

### 2. **DTOの活用**
- ドメインオブジェクトとAPIのデータモデルを分離し、堅牢性とセキュリティを確保
- 入力値の検証と変換
- レスポンスの最適化

### 3. **HATEOAS**
- ハイパーメディアリンクによるAPIの自己文書化とクライアントの自律性
- リソース間の関係性の表現
- APIの進化への対応

### 4. **エラーハンドリング**
- 一貫性のあるエラーレスポンスと適切なHTTPステータスコード
- ユーザーフレンドリーなエラーメッセージ
- デバッグ情報の適切な管理

### 5. **バリデーション**
- 入力値の検証とドメインの不変条件の区別
- セキュリティ上の脅威の防止
- パフォーマンスへの配慮

### 6. **セキュリティ**
- 認証・認可の実装とセキュリティヘッダーの設定
- 入力値の検証とサニタイゼーション
- セキュリティ監査とログ

## 📚 実装例

この章では、以下のプレゼンテーション層コンポーネントを実装します：

### REST APIコントローラ
- **UserController**: ユーザー管理API
- **OrderController**: 注文管理API
- **ProductController**: 商品管理API

### DTOとレスポンス
- **リクエストDTO**: `CreateUserRequest`, `UpdateUserRequest`, `CreateOrderRequest`
- **レスポンスDTO**: `UserResponse`, `OrderResponse`, `ProductResponse`
- **HATEOAS**: `UserResource`, `OrderResource`, `ProductResource`

### エラーハンドリング
- **GlobalExceptionHandler**: グローバル例外ハンドラー
- **ErrorResponse**: 標準化されたエラーレスポンス
- **ValidationErrorResponse**: バリデーションエラーレスポンス

### セキュリティ
- **SecurityConfiguration**: セキュリティ設定
- **JwtAuthenticationFilter**: JWT認証フィルター
- **UserDetailsService**: ユーザー詳細サービス

## 🔄 次のステップ

次の章では、テスト戦略について学び、各層のテスト実装について詳しく説明します。Spockフレームワークを中心としたテスト戦略と、ArchUnitによるアーキテクチャの健全性を保つためのテストについて解説します。 