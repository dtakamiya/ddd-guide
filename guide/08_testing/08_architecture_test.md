# 8.5 アーキテクチャテスト

アーキテクチャテストは、アプリケーションの設計原則や依存関係ルールが正しく守られているかを検証するテストです。ArchUnitを使用して、オニオンアーキテクチャの依存関係ルールやDDDの設計原則をコードレベルで強制します。

## ArchUnitによる依存関係チェック

ArchUnitは、アーキテクチャの制約をテストとして記述し、CI/CDパイプラインで自動的に検証できるライブラリです。

### 基本的なアーキテクチャテスト

```groovy
// src/test/groovy/com/example/architecture/ArchitectureSpec.groovy
import com.tngtech.archunit.core.domain.JavaClasses
import com.tngtech.archunit.core.importer.ClassFileImporter
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition
import spock.lang.Shared
import spock.lang.Specification

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses

class ArchitectureSpec extends Specification {

    @Shared
    JavaClasses importedClasses = new ClassFileImporter()
            .withImportOption({ location -> !location.contains("/test/") }) // テストコードを除外
            .importPackages("com.example.dddspanner")

    def "domain layer should not depend on other layers"() {
        expect: "ドメイン層は他の層に依存してはいけない"
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..application..", "..infrastructure..", "..presentation..")
            .check(importedClasses)
    }

    def "application layer should only depend on domain layer"() {
        expect: "アプリケーション層はドメイン層のみに依存できる"
        ArchRuleDefinition.classes()
            .that().resideInAPackage("..application..")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage(
                "..application..", 
                "..domain..", 
                "java..", 
                "org.springframework..",
                "lombok.." // プロジェクトで使用しているライブラリを適宜追加
            )
            .check(importedClasses)
    }

    def "infrastructure layer should not depend on presentation layer"() {
        expect: "インフラストラクチャ層はプレゼンテーション層に依存してはいけない"
        noClasses()
            .that().resideInAPackage("..infrastructure..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..presentation..")
            .check(importedClasses)
    }

    def "presentation layer should only depend on application layer"() {
        expect: "プレゼンテーション層はアプリケーション層のみに依存できる"
        noClasses()
            .that().resideInAPackage("..presentation..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..domain..", "..infrastructure..")
            .check(importedClasses)
    }
}
```

## オニオンアーキテクチャの検証

オニオンアーキテクチャの核心原則である「内側のレイヤーは外側のレイヤーを知ってはならない」を厳密に検証します。

### レイヤー間の依存関係ルール

```groovy
def "onion architecture dependency rules should be enforced"() {
    expect: "オニオンアーキテクチャの依存関係ルールが強制される"
    
    // プレゼンテーション層はアプリケーション層のみに依存できる
    def presentationRule = noClasses()
        .that().resideInAPackage("..presentation..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..domain..", "..infrastructure..")
    
    // アプリケーション層はドメイン層以外の層に依存してはいけない
    def applicationRule = noClasses()
        .that().resideInAPackage("..application..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..infrastructure..")
    
    // インフラストラクチャ層はドメイン層とアプリケーション層に依存できる
    def infrastructureRule = ArchRuleDefinition.classes()
        .that().resideInAPackage("..infrastructure..")
        .should().onlyDependOnClassesThat()
        .resideInAnyPackage(
            "..infrastructure..",
            "..domain..",
            "..application..",
            "java..",
            "org.springframework..",
            "com.google.cloud..",
            "org.springframework.amqp.."
        )
    
    presentationRule.check(importedClasses)
    applicationRule.check(importedClasses)
    infrastructureRule.check(importedClasses)
}
```

### インターフェースと実装の分離

```groovy
def "domain interfaces should not have infrastructure implementations"() {
    expect: "ドメインインターフェースはインフラストラクチャ実装を持たない"
    noClasses()
        .that().resideInAPackage("..domain..")
        .and().areInterfaces()
        .should().dependOnClassesThat()
        .resideInAnyPackage("..infrastructure..")
        .check(importedClasses)
}

def "application services should use domain interfaces"() {
    expect: "アプリケーションサービスはドメインインターフェースを使用する"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..application..")
        .and().haveSimpleNameEndingWith("Service")
        .should().dependOnClassesThat()
        .resideInAPackage("..domain..")
        .and().areInterfaces()
        .check(importedClasses)
}
```

## レイヤー間の境界テスト

### パッケージ構造の検証

```groovy
def "package structure should follow DDD conventions"() {
    expect: "パッケージ構造がDDDの慣例に従う"
    
    // ドメイン層のパッケージ構造
    def domainPackages = [
        "com.example.dddspanner.domain.entity",
        "com.example.dddspanner.domain.valueobject",
        "com.example.dddspanner.domain.service",
        "com.example.dddspanner.domain.repository",
        "com.example.dddspanner.domain.event"
    ]
    
    // アプリケーション層のパッケージ構造
    def applicationPackages = [
        "com.example.dddspanner.application.service",
        "com.example.dddspanner.application.command",
        "com.example.dddspanner.application.query",
        "com.example.dddspanner.application.dto"
    ]
    
    // インフラストラクチャ層のパッケージ構造
    def infrastructurePackages = [
        "com.example.dddspanner.infrastructure.persistence",
        "com.example.dddspanner.infrastructure.messaging",
        "com.example.dddspanner.infrastructure.config"
    ]
    
    // プレゼンテーション層のパッケージ構造
    def presentationPackages = [
        "com.example.dddspanner.presentation.controller",
        "com.example.dddspanner.presentation.dto",
        "com.example.dddspanner.presentation.exception"
    ]
    
    // 各パッケージが存在することを確認
    domainPackages.each { pkg ->
        assert importedClasses.containPackage(pkg)
    }
    
    applicationPackages.each { pkg ->
        assert importedClasses.containPackage(pkg)
    }
    
    infrastructurePackages.each { pkg ->
        assert importedClasses.containPackage(pkg)
    }
    
    presentationPackages.each { pkg ->
        assert importedClasses.containPackage(pkg)
    }
}
```

### 命名規則の検証

```groovy
def "domain entities should follow naming conventions"() {
    expect: "ドメインエンティティが命名規則に従う"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..domain.entity..")
        .should().haveSimpleNameNotEndingWith("Repository")
        .and().haveSimpleNameNotEndingWith("Service")
        .and().haveSimpleNameNotEndingWith("Event")
        .check(importedClasses)
}

def "application services should follow naming conventions"() {
    expect: "アプリケーションサービスが命名規則に従う"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..application.service..")
        .should().haveSimpleNameEndingWith("Service")
        .check(importedClasses)
}

def "infrastructure implementations should follow naming conventions"() {
    expect: "インフラストラクチャ実装が命名規則に従う"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..infrastructure.persistence..")
        .and().implement("..domain.repository..")
        .should().haveSimpleNameEndingWith("Repository")
        .check(importedClasses)
}
```

## 設計原則の検証

### 依存性注入の検証

```groovy
def "application services should use constructor injection"() {
    expect: "アプリケーションサービスがコンストラクタ注入を使用する"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..application.service..")
        .should().haveOnlyFinalFields()
        .check(importedClasses)
}

def "domain entities should not have dependencies"() {
    expect: "ドメインエンティティが依存関係を持たない"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..domain.entity..")
        .should().notDependOnClassesThat()
        .resideInAnyPackage("org.springframework..", "javax.inject..")
        .check(importedClasses)
}
```

### 不変性の検証

```groovy
def "value objects should be immutable"() {
    expect: "値オブジェクトが不変である"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..domain.valueobject..")
        .should().beFinal()
        .and().haveOnlyFinalFields()
        .check(importedClasses)
}

def "domain events should be immutable"() {
    expect: "ドメインイベントが不変である"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..domain.event..")
        .should().beFinal()
        .and().haveOnlyFinalFields()
        .check(importedClasses)
}
```

## セキュリティアーキテクチャの検証

### 認証・認可の検証

```groovy
def "presentation layer should handle security"() {
    expect: "プレゼンテーション層がセキュリティを処理する"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..presentation.controller..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("org.springframework.security..")
        .check(importedClasses)
}

def "domain layer should not handle security"() {
    expect: "ドメイン層がセキュリティを処理しない"
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("org.springframework.security..")
        .check(importedClasses)
}
```

## パフォーマンスアーキテクチャの検証

### キャッシュの検証

```groovy
def "caching should be handled in infrastructure layer"() {
    expect: "キャッシュがインフラストラクチャ層で処理される"
    ArchRuleDefinition.classes()
        .that().resideInAPackage("..infrastructure..")
        .and().haveSimpleNameContaining("Cache")
        .should().dependOnClassesThat()
        .resideInAnyPackage("org.springframework.cache..", "org.springframework.data.redis..")
        .check(importedClasses)
}

def "domain layer should not handle caching"() {
    expect: "ドメイン層がキャッシュを処理しない"
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("org.springframework.cache..", "org.springframework.data.redis..")
        .check(importedClasses)
}
```

## 学習のポイント

1. **アーキテクチャの自動検証**: ArchUnitで設計原則をコードレベルで強制
2. **依存関係の明確化**: レイヤー間の依存関係を明確に定義・検証
3. **命名規則の統一**: 一貫した命名規則でコードの可読性を向上
4. **設計原則の遵守**: DDDとオニオンアーキテクチャの原則を確実に遵守

次のセクションでは、Testcontainersによるテストについて詳しく解説します。 