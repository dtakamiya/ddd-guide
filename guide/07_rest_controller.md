### 7.1. REST APIコントローラの実装 (Spring MVC)

`@RestController`アノテーションをクラスに付与し、`@GetMapping`, `@PostMapping`などのアノテーションをメソッドに付与することで、簡単にRESTfulなエンドポイントを定義できます。

*   **責務:**
    *   HTTPリクエストを受け取る。
    *   リクエストのペイロード（JSONなど）をDTOにマッピングする。
    *   アプリケーションサービスを呼び出し、ユースケースを実行する。
    *   アプリケーションサービスから返された結果を、HTTPレスポンス（JSONなど）に変換して返却する。
*   **注意点:**
    *   コントローラにはビジネスロジックを記述しません。ロジックはアプリケーション層またはドメイン層に配置します。
    *   リクエストのバリデーションはコントローラの責務の一部ですが、ドメインの不変条件の検証とは区別します。（詳細は後述）

**UserRestController.java:**
```java
package com.example.dddspanner.presentation.controller;

import com.example.dddspanner.application.dto.CreateUserRequest;
import com.example.dddspanner.application.dto.UpdateUserRequest;
import com.example.dddspanner.application.dto.UserResponse;
import com.example.dddspanner.application.service.UserApplicationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.hateoas.CollectionModel;
import org.springframework.hateoas.EntityModel;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import jakarta.validation.Valid;
import java.util.List;

/**
 * ユーザーREST APIコントローラ
 * 
 * コントローラの特徴：
 * - HTTPリクエスト・レスポンスの処理
 * - アプリケーションサービスの呼び出し
 * - HATEOASリンクの付与
 */
@Slf4j
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserRestController {
    
    private final UserApplicationService userApplicationService;
    private final UserResponseAssembler userResponseAssembler;
    
    /**
     * ユーザー作成
     */
    @PostMapping
    public ResponseEntity<EntityModel<UserResponse>> createUser(@Valid @RequestBody CreateUserRequest request) {
        log.info("Creating user with email: {}", request.email());
        
        UserResponse user = userApplicationService.createUser(request);
        EntityModel<UserResponse> response = userResponseAssembler.toModel(user);
        
        return ResponseEntity
            .created(response.getRequiredLink("self").toUri())
            .body(response);
    }
    
    /**
     * ユーザー取得
     */
    @GetMapping("/{id}")
    public ResponseEntity<EntityModel<UserResponse>> getUserById(@PathVariable String id) {
        log.info("Getting user by id: {}", id);
        
        UserResponse user = userApplicationService.getUserById(id);
        EntityModel<UserResponse> response = userResponseAssembler.toModel(user);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * ユーザー一覧取得
     */
    @GetMapping
    public ResponseEntity<CollectionModel<EntityModel<UserResponse>>> getAllUsers() {
        log.info("Getting all users");
        
        List<UserResponse> users = userApplicationService.getAllUsers();
        CollectionModel<EntityModel<UserResponse>> response = userResponseAssembler.toCollectionModel(users);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * ユーザー更新
     */
    @PutMapping("/{id}")
    public ResponseEntity<EntityModel<UserResponse>> updateUser(
            @PathVariable String id,
            @Valid @RequestBody UpdateUserRequest request) {
        log.info("Updating user with id: {}", id);
        
        UserResponse user = userApplicationService.updateUser(id, request);
        EntityModel<UserResponse> response = userResponseAssembler.toModel(user);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * ユーザー削除
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable String id) {
        log.info("Deleting user with id: {}", id);
        
        userApplicationService.deleteUser(id);
        
        return ResponseEntity.noContent().build();
    }
    
    /**
     * アクティブユーザー一覧取得
     */
    @GetMapping("/active")
    public ResponseEntity<CollectionModel<EntityModel<UserResponse>>> getActiveUsers() {
        log.info("Getting active users");
        
        List<UserResponse> users = userApplicationService.getActiveUsers();
        CollectionModel<EntityModel<UserResponse>> response = userResponseAssembler.toCollectionModel(users);
        
        return ResponseEntity.ok(response);
    }
}
``` 