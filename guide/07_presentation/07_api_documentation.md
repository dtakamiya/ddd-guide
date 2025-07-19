### 7.5. API文書化

#### 7.5.1. OpenAPI（Swagger）の設定

Spring Bootでは、SpringDoc OpenAPIを使用してAPI仕様書を自動生成できます。

**OpenApiConfig.java:**
```java
package com.example.dddspanner.presentation.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.servers.Server;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

/**
 * OpenAPI設定
 * 
 * 文書化の特徴：
 * - 自動生成されるAPI仕様書
 * - インタラクティブなテスト環境
 * - バージョン管理
 */
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("DDD Spanner API")
                .description("Domain-Driven Design with Google Cloud Spanner")
                .version("1.0.0")
                .contact(new Contact()
                    .name("Development Team")
                    .email("dev@example.com")
                    .url("https://github.com/example/ddd-spanner"))
                .license(new License()
                    .name("MIT License")
                    .url("https://opensource.org/licenses/MIT")))
            .servers(List.of(
                new Server().url("http://localhost:8080").description("Development server"),
                new Server().url("https://api.example.com").description("Production server")
            ));
    }
}
```

#### 7.5.2. API仕様書の自動生成

コントローラとDTOにアノテーションを追加して、詳細なAPI仕様書を生成します。

**UserRestController.java（OpenAPIアノテーション付き）:**
```java
package com.example.dddspanner.presentation.controller;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

/**
 * ユーザーREST APIコントローラ
 */
@Tag(name = "User", description = "ユーザー管理API")
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserRestController {
    
    private final UserApplicationService userApplicationService;
    private final UserResponseAssembler userResponseAssembler;
    
    @Operation(
        summary = "ユーザー作成",
        description = "新しいユーザーを作成します"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "201",
            description = "ユーザーが正常に作成されました",
            content = @Content(schema = @Schema(implementation = UserResponse.class))
        ),
        @ApiResponse(
            responseCode = "400",
            description = "リクエストが無効です",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        ),
        @ApiResponse(
            responseCode = "409",
            description = "メールアドレスが既に使用されています",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    @PostMapping
    public ResponseEntity<EntityModel<UserResponse>> createUser(
            @Parameter(description = "ユーザー作成リクエスト", required = true)
            @Valid @RequestBody CreateUserRequest request) {
        
        log.info("Creating user with email: {}", request.email());
        
        UserResponse user = userApplicationService.createUser(request);
        EntityModel<UserResponse> response = userResponseAssembler.toModel(user);
        
        return ResponseEntity
            .created(response.getRequiredLink("self").toUri())
            .body(response);
    }
    
    @Operation(
        summary = "ユーザー取得",
        description = "指定されたIDのユーザーを取得します"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "200",
            description = "ユーザーが見つかりました",
            content = @Content(schema = @Schema(implementation = UserResponse.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "ユーザーが見つかりません",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    @GetMapping("/{id}")
    public ResponseEntity<EntityModel<UserResponse>> getUserById(
            @Parameter(description = "ユーザーID", required = true)
            @PathVariable String id) {
        
        log.info("Getting user by id: {}", id);
        
        UserResponse user = userApplicationService.getUserById(id);
        EntityModel<UserResponse> response = userResponseAssembler.toModel(user);
        
        return ResponseEntity.ok(response);
    }
    
    @Operation(
        summary = "ユーザー一覧取得",
        description = "すべてのユーザーを取得します"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "200",
            description = "ユーザー一覧が正常に取得されました",
            content = @Content(schema = @Schema(implementation = UserResponse.class))
        )
    })
    @GetMapping
    public ResponseEntity<CollectionModel<EntityModel<UserResponse>>> getAllUsers() {
        log.info("Getting all users");
        
        List<UserResponse> users = userApplicationService.getAllUsers();
        CollectionModel<EntityModel<UserResponse>> response = userResponseAssembler.toCollectionModel(users);
        
        return ResponseEntity.ok(response);
    }
}
```

#### 7.5.3. テストケースの文書化

APIテストケースも文書化して、開発者やQAチームが理解しやすくします。

**UserApiTest.java:**
```java
package com.example.dddspanner.presentation.test;

import com.example.dddspanner.presentation.dto.CreateUserRequest;
import com.example.dddspanner.presentation.dto.UserResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * ユーザーAPIテスト
 * 
 * テストの特徴：
 * - 統合テスト
 * - API仕様書との整合性
 * - エラーケースの検証
 */
@SpringBootTest
@AutoConfigureTestDatabase
@ActiveProfiles("test")
@DisplayName("ユーザーAPIテスト")
class UserApiTest {
    
    @Autowired
    private WebApplicationContext webApplicationContext;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    private MockMvc mockMvc;
    
    @Test
    @DisplayName("正常なユーザー作成")
    void shouldCreateUserSuccessfully() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest(
            "Taro",
            "Yamada",
            "taro.yamada@example.com"
        );
        
        // When & Then
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.firstName").value("Taro"))
            .andExpect(jsonPath("$.lastName").value("Yamada"))
            .andExpect(jsonPath("$.email").value("taro.yamada@example.com"))
            .andExpect(jsonPath("$._links.self.href").exists());
    }
    
    @Test
    @DisplayName("無効なメールアドレスでユーザー作成を試行")
    void shouldFailWithInvalidEmail() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest(
            "Taro",
            "Yamada",
            "invalid-email"
        );
        
        // When & Then
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errorCode").value("VALIDATION_ERROR"))
            .andExpect(jsonPath("$.fieldErrors[0].field").value("email"));
    }
    
    @Test
    @DisplayName("存在しないユーザーの取得")
    void shouldReturnNotFoundForNonExistentUser() throws Exception {
        // When & Then
        mockMvc.perform(get("/api/v1/users/non-existent-id"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.errorCode").value("USER_NOT_FOUND"));
    }
}
``` 