## 8. テスト

アプリケーションの品質と保守性を高く保つために、テストは不可欠なプロセスです。この章では、Spockフレームワークを中心としたテスト戦略と、ArchUnitによるアーキテクチャの健全性を保つためのテストについて解説します。

### 8.1. Spockを使ったテストの基本

SpockはGroovy言語で記述する、表現力豊かで可読性の高いテスティングフレームワークです。BDD（ビヘイビア駆動開発）スタイルでテストを記述できるため、テストの意図が明確になります。

*   **`given:`**: テストの前提条件（セットアップ）を記述します。
*   **`when:`**: テスト対象のコードを実行します。
*   **`then:`**: 実行結果が期待通りであることを検証します。
*   **`where:`**: パラメータ化テストのデータを提供します。

### 8.2. ユニットテストの実装例

ドメイン層のオブジェクト（エンティティ、値オブジェクト、ドメインサービス）や、アプリケーションサービス内のロジックなどを対象とします。外部依存（データベースや外部API）はモックに置き換えてテストします。

**`User`エンティティのテスト例:**
```groovy
// src/test/groovy/com/example/domain/UserSpec.groovy
import spock.lang.Specification

class UserSpec extends Specification {

    def "changeName should update the user's name"() {
        given:
        def user = new User("1", "old-name", "test@example.com")

        when:
        user.changeName("new-name")

        then:
        user.getName() == "new-name"
    }
}
```

### 8.3. インテグレーションテストの実装例 (SpringBootTest)

Spring Bootコンテキストをロードして、複数のコンポーネントを連携させたテストを行います。APIエンドポイントの動作確認や、データベース永続化を含めた一連のフローをテストするのに適しています。

**`UserRestController`のテスト例:**
```groovy
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc
import spock.lang.Specification

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf

@SpringBootTest
@AutoConfigureMockMvc
class UserRestControllerIntegrationSpec extends Specification {

    @Autowired
    private MockMvc mockMvc
    
    @Autowired
    private ObjectMapper objectMapper // JSONシリアライズ/デシリアライズ用

    @Autowired
    private SpannerUserRepository userRepository // DBの状態をセットアップ/検証するために使用

    def cleanup() {
        userRepository.deleteAll()
    }

    @WithMockUser(username = "testuser", roles = ["USER"]) // 認証済みユーザーとしてテストを実行
    def "POST /api/v1/users should create a new user and return HATEOAS links"() {
        given:
        def command = new CreateUserCommand("Taro", "Yamada", "taro.yamada@example.com")
        def requestBody = objectMapper.writeValueAsString(command)

        when:
        def result = mockMvc.perform(post("/api/v1/users")
                .contentType("application/json")
                .content(requestBody)
                .with(csrf())) // CSRFトークンを付与

        then:
        result.andExpect(status().isCreated())
              .andExpect(jsonPath('$.fullName', is('Taro Yamada')))
              .andExpect(jsonPath('$.email', is('taro.yamada@example.com')))
              .andExpect(jsonPath('$._links.self.href', containsString('/api/v1/users/')))
    }

    def "GET /api/v1/users/{id} without authentication should return 401 Unauthorized"() {
        when:
        def result = mockMvc.perform(get("/api/v1/users/some-id"))

        then:
        result.andExpect(status().isUnauthorized())
    }
}
```
このテストでは、`@WithMockUser`を使って認証済みの状態をシミュレートし、リクエストが成功することを検証します。また、認証なしでのアクセスが`401 Unauthorized`で正しく拒否されることもテストしています。`jsonPath`マッチャーを使い、レスポンスボディの内容まで詳細に検証している点も重要です。

### 8.4. ArchUnitによるアーキテクチャテスト (Spock版)

ArchUnitのテストも、ガイド全体の一貫性を保つためにSpock Specificationとして記述できます。

**ArchitectureSpec.groovy:**
```groovy
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
        expect:
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..application..", "..infrastructure..", "..presentation..")
            .check(importedClasses)
    }

    def "application layer should only depend on domain layer"() {
        expect:
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

    def "onion architecture dependency rules should be enforced"() {
        expect:
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

        presentationRule.check(importedClasses)
        applicationRule.check(importedClasses)
    }
}
```
オニオンアーキテクチャの依存関係ルールをより厳密に定義し直しています。「内側のレイヤーは外側のレイヤーを知ってはならない」という原則をコードで強制します。

### 8.5. Testcontainersによる効率的なインテグレーションテスト

インフラストラクチャ層、特にリポジトリの実装をテストする際には、実際のデータベースに近い環境で検証することが理想です。Google Cloud Spannerはローカルで動作するエミュレータを提供しており、Testcontainersライブラリと組み合わせることで、インテグレーションテストの際にSpannerエミュレータのDockerコンテナを簡単に管理できます。

これにより、コストをかけずに、オフライン環境でも信頼性の高いインテグレーションテストを実行できます。

#### 8.5.1. 依存関係の追加

`build.gradle`にTestcontainersとSpanner関連のライブラリを追加します。

```groovy
// build.gradle
testImplementation 'org.testcontainers:spanner:1.19.7'
testImplementation 'org.testcontainers:junit-jupiter:1.19.7'
testImplementation 'com.google.cloud:google-cloud-spanner:6.53.0' // Spannerクライアント
```

#### 8.5.2. テスト用プロファイルの設定

テスト実行時にエミュレータを使用するように、`src/test/resources/application-test.properties` を作成します。

```properties
# src/test/resources/application-test.properties
spring.cloud.gcp.spanner.emulator.enabled=true
```

#### 8.5.3. Singleton Container Patternの適用

テストスイート全体で単一のコンテナを共有する**Singleton Container Pattern**を適用することで、テストの実行時間を大幅に短縮できます。コンテナの起動はコストが高いため、テストクラスごとに起動するのは非効率です。

**AbstractIntegrationSpec.groovy (Singleton Container):**
```groovy
import com.google.cloud.spanner.*
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.SpannerEmulatorContainer
import org.testcontainers.utility.DockerImageName
import spock.lang.Shared
import spock.lang.Specification

import java.nio.file.Files
import java.nio.file.Paths

abstract class AbstractIntegrationSpec extends Specification {

    @Shared
    private static SpannerEmulatorContainer spannerEmulator

    // staticイニシャライザでコンテナを一度だけ起動する
    static {
        spannerEmulator = new SpannerEmulatorContainer(
                DockerImageName.parse("gcr.io/cloud-spanner-emulator/emulator:latest"))
        spannerEmulator.start()
        
        // テスト用のDBとテーブルを作成
        setupDatabase()
    }

    @DynamicPropertySource
    static void spannerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.cloud.gcp.spanner.emulator-host",
                { -> spannerEmulator.getEmulatorGrpcEndpoint() })
    }

    private static void setupDatabase() {
        // ... (DBクライアント設定とDDL実行、内容は変更なし)
    }
}
```
`@Testcontainers`アノテーションを外し、Groovyの`static`イニシャライザを使ってコンテナをJVMのクラスロード時に一度だけ起動するように変更しました。これにより、この基底クラスを継承する全てのテストで同じコンテナが再利用され、テストスイートの実行が高速になります。

**SpannerUserRepositoryIntegrationSpec.groovy (テスト実装):**
```groovy
@SpringBootTest
@ActiveProfiles("test")
class SpannerUserRepositoryIntegrationSpec extends AbstractIntegrationSpec {
    
    @Autowired
    private SpannerUserRepository userRepository
    
    def "should save and find user by id"() {
        given:
        def fullName = new FullName("Test", "User")
        def email = new Email("test.user@example.com")
        def user = User.create(fullName, email)

        when:
        userRepository.save(user)
        def found = userRepository.findById(user.getId())

        then:
        found.isPresent()
        found.get().getId() == user.getId()
        found.get().getFullName().getFullName() == "Test User"
    }
}
```
テストクラスは`AbstractIntegrationSpec`を継承するだけで、セットアップ済みのSpannerエミュレータを利用できます。 