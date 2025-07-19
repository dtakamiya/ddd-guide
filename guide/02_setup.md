## 2. 開発環境のセットアップ

このガイドで紹介するサンプルコードは、以下の環境で動作確認をしています。

*   **Java**: 17
*   **ビルドツール**: Gradle 8.5
*   **フレームワーク**: Spring Boot 3.5.3
*   **データベース**: Google Cloud Spanner
*   **メッセージングキュー**: RabbitMQ
*   **テストフレームワーク**: Spock 2.4-M6-groovy-4.0

### 2.1. 必要なツールのインストール

#### 2.1.1. Java 17のインストール

Java 17は、2021年9月にリリースされたLTS（Long Term Support）バージョンです。以下の理由から、このガイドではJava 17を採用しています：

*   **`record`型**: 値オブジェクトの実装に最適な不変データクラス
*   **パターンマッチング**: より簡潔で読みやすいコードの記述
*   **テキストブロック**: マルチライン文字列の可読性向上
*   **Sealed Classes**: 型安全性の向上

**macOSでのインストール例:**
```bash
# Homebrewを使用
brew install openjdk@17

# 環境変数の設定
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**Windowsでのインストール例:**
*   [Oracle JDK 17](https://www.oracle.com/java/technologies/downloads/#java17)または[OpenJDK 17](https://adoptium.net/)をダウンロード
*   インストーラーを実行し、環境変数`JAVA_HOME`を設定

**Linuxでのインストール例:**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install openjdk-17-jdk

# CentOS/RHEL
sudo yum install java-17-openjdk-devel
```

#### 2.1.2. Gradle 8.5のインストール

Gradleは、柔軟で高性能なビルドツールです。以下の理由から採用しています：

*   **Kotlin DSL**: 型安全なビルドスクリプト
*   **依存関係管理**: 自動的な依存関係解決
*   **プラグインエコシステム**: 豊富なプラグイン
*   **ビルドキャッシュ**: 高速なビルド実行

**インストール方法:**
```bash
# macOS
brew install gradle

# Windows (Chocolatey)
choco install gradle

# Linux
sudo apt install gradle
```

#### 2.1.3. Dockerのインストール

Testcontainersを使用するために、Dockerが必要です。

**macOS/Windows:**
*   [Docker Desktop](https://www.docker.com/products/docker-desktop/)をダウンロードしてインストール

**Linux:**
```bash
# Ubuntu
sudo apt update
sudo apt install docker.io
sudo usermod -aG docker $USER
```

#### 2.1.4. IDEの設定

**IntelliJ IDEA（推奨）:**
*   [IntelliJ IDEA Community Edition](https://www.jetbrains.com/idea/download/)をダウンロード
*   Spring Boot、Gradle、Spockプラグインを有効化
*   Java 17をプロジェクトSDKとして設定

**VS Code:**
*   [VS Code](https://code.visualstudio.com/)をダウンロード
*   Extension Pack for Javaをインストール
*   Spring Boot Extension Packをインストール

### 2.2. build.gradle

プロジェクトの依存関係を管理するために、`build.gradle`を以下のように設定します。

**build.gradle:**
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.3'
    id 'io.spring.dependency-management' version '1.1.5'
    id 'groovy'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    sourceCompatibility = '17'
}

repositories {
    mavenCentral()
}

ext {
    set('springCloudGcpVersion', "5.3.0")
    set('springCloudVersion', "2024.0.0")
}

dependencies {
    // Spring Boot Starters
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-amqp' // RabbitMQ

    // Spring Cloud GCP for Spanner
    implementation 'com.google.cloud:spring-cloud-gcp-starter-data-spanner'

    // JWT Support
    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.11.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.11.5'

    // Lombok for boilerplate code reduction
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // MapStruct for object mapping
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

    // Test Dependencies
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testImplementation 'org.springframework.amqp:spring-rabbit-test'

    // Spock Framework for testing
    testImplementation 'org.spockframework:spock-core:2.4-M6-groovy-4.0'
    testImplementation 'org.spockframework:spock-spring:2.4-M6-groovy-4.0'
    testImplementation 'org.codehaus.groovy:groovy:4.0.21' // Groovy version for Spock

    // ArchUnit for architecture testing
    testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'

    // Testcontainers for integration testing
    testImplementation 'org.testcontainers:spanner'
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:rabbitmq'
}

dependencyManagement {
    imports {
        mavenBom "com.google.cloud:spring-cloud-gcp-dependencies:${springCloudGcpVersion}"
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### 2.3. 主要な依存関係の解説

#### 2.3.1. Spring Boot Starters

*   `org.springframework.boot:spring-boot-starter-*`: Spring Bootの各機能を利用するためのスターター依存関係です。Web、Validation、Security、AMQP（RabbitMQ）を含めています。
*   **Web**: REST APIの実装に必要なSpring MVC、Tomcat、Jackson
*   **Validation**: 入力値検証のためのBean Validation
*   **Security**: 認証・認可機能の実装
*   **AMQP**: RabbitMQとの連携

#### 2.3.2. Google Cloud Spanner

*   `com.google.cloud:spring-cloud-gcp-starter-data-spanner`: Spring Dataを利用してGoogle Cloud Spannerにアクセスするためのライブラリです。
*   **分散データベース**: グローバルな一貫性と高可用性
*   **Spring Data統合**: JPAライクなAPIでSpannerにアクセス
*   **エミュレータ対応**: ローカル開発時のエミュレータ使用

#### 2.3.3. セキュリティ関連

*   `io.jsonwebtoken:*`: JWT（JSON Web Token）を生成・検証するために利用します。
*   **ステートレス認証**: セッション管理不要の認証方式
*   **マイクロサービス対応**: サービス間の認証に適している

#### 2.3.4. 開発効率化ツール

*   `org.projectlombok:lombok`: `record`が利用できない、あるいはしたくない場面で、アノテーションを使ってボイラープレートコード（`getter`, `setter`, `toString`など）を削減します。
*   `org.mapstruct:mapstruct`: オブジェクト間のマッピングを自動生成するライブラリです。

#### 2.3.5. テスト関連

*   `org.spockframework:*`: Java/Groovy向けのテスティングフレームワークです。振る舞い駆動開発（BDD）スタイルで記述でき、可読性の高いテストコードを作成できます。
*   `com.tngtech.archunit:archunit-junit5`: アーキテクチャのルール（例：レイヤー間の依存関係）をテストするためのライブラリです。
*   `org.testcontainers:*`: Dockerコンテナを使って、SpannerエミュレータやRabbitMQなどの外部サービスと連携するインテグレーションテストを容易に実行できるようにします。

#### 2.3.6. 依存関係管理

*   `dependencyManagement`: Spring Cloud GCPとSpring CloudのBOM（Bill of Materials）をインポートし、互換性のあるライブラリバージョンを自動的に管理します。

### 2.4. アプリケーション設定

#### 2.4.1. application.properties

**src/main/resources/application.properties:**
```properties
# Server Configuration
server.port=8080

# Google Cloud Spanner Configuration
spring.cloud.gcp.spanner.project-id=your-project-id
spring.cloud.gcp.spanner.instance-id=your-instance-id
spring.cloud.gcp.spanner.database=your-database

# RabbitMQ Configuration
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

# JWT Configuration
jwt.secret=your-super-strong-and-long-secret-key-for-jwt-signing

# Logging Configuration
logging.level.com.example=DEBUG
logging.level.org.springframework.security=DEBUG
```

#### 2.4.2. テスト用設定

**src/test/resources/application-test.properties:**
```properties
# Test Configuration
spring.cloud.gcp.spanner.emulator.enabled=true
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672

# Disable security for testing
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

### 2.5. プロジェクト構造

推奨するプロジェクト構造は以下の通りです：

```
src/
├── main/
│   ├── java/
│   │   └── com/example/dddspanner/
│   │       ├── domain/           # ドメイン層
│   │       │   ├── entity/       # エンティティ
│   │       │   ├── valueobject/  # 値オブジェクト
│   │       │   ├── repository/   # リポジトリインターフェース
│   │       │   └── service/      # ドメインサービス
│   │       ├── application/       # アプリケーション層
│   │       │   ├── service/      # アプリケーションサービス
│   │       │   ├── dto/          # データ転送オブジェクト
│   │       │   └── mapper/       # オブジェクトマッピング
│   │       ├── infrastructure/    # インフラストラクチャ層
│   │       │   ├── repository/   # リポジトリ実装
│   │       │   ├── messaging/    # メッセージング
│   │       │   └── config/       # 設定クラス
│   │       └── presentation/      # プレゼンテーション層
│   │           ├── controller/    # RESTコントローラ
│   │           ├── dto/          # リクエスト/レスポンスDTO
│   │           └── security/     # セキュリティ設定
│   └── resources/
│       ├── application.properties
│       └── db/migration/         # データベースマイグレーション
└── test/
    ├── java/
    │   └── com/example/dddspanner/
    │       ├── domain/           # ドメイン層のテスト
    │       ├── application/      # アプリケーション層のテスト
    │       ├── infrastructure/   # インフラストラクチャ層のテスト
    │       └── presentation/     # プレゼンテーション層のテスト
    └── groovy/
        └── com/example/dddspanner/
            └── integration/       # インテグレーションテスト
```

### 2.6. 開発環境の検証

セットアップが完了したら、以下のコマンドで環境を検証してください：

```bash
# Java バージョンの確認
java -version

# Gradle バージョンの確認
gradle --version

# Docker の確認
docker --version

# プロジェクトのビルド
./gradlew build

# テストの実行
./gradlew test
```

すべてのコマンドが正常に実行されれば、開発環境のセットアップは完了です。 