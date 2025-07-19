### 7.2. リクエストとレスポンスの設計

コントローラは、ドメインオブジェクトを直接APIの入出力として使うべきではありません。代わりに、そのユースケースに特化したDTO (Data Transfer Object) を使います。

**DTOを使用する主な理由:**
*   **関心の分離**: ドメインモデルと、外部に公開するAPIのデータモデルを分離します。
*   **堅牢性**: ドメインモデルの内部的な変更（フィールドの追加など）が、意図せずAPIの互換性を破壊するのを防ぎます。
*   **セキュリティ**: `User`エンティティにパスワードハッシュのような機微な情報が含まれていても、レスポンス用のDTOにそれを含めないことで、情報漏洩のリスクをなくします。
*   **最適化**: APIが必要とするデータだけを過不足なく含んだ、軽量なオブジェクトを作成できます。

本ガイドでは、入力には`CreateUserRequest`を、出力には`UserResponse`を使用します。

#### 7.2.1. リクエストDTOの設計

**CreateUserRequest.java:**
```java
package com.example.dddspanner.presentation.request;

import com.example.dddspanner.presentation.validation.ValidEmail;
import com.example.dddspanner.presentation.validation.ValidPassword;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Value;

/**
 * ユーザー作成リクエスト
 */
@Value
public class CreateUserRequest {
    
    @NotBlank(message = "Full name is required")
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters")
    String fullName;
    
    @NotBlank(message = "Email is required")
    @ValidEmail(message = "Invalid email format")
    String email;
    
    @NotBlank(message = "Password is required")
    @ValidPassword(message = "Password must be at least 8 characters long")
    String password;
    
    @Size(max = 500, message = "Bio must not exceed 500 characters")
    String bio;
    
    @Size(max = 20, message = "Phone number must not exceed 20 characters")
    String phoneNumber;
}
```

**UpdateUserRequest.java:**
```java
package com.example.dddspanner.presentation.request;

import com.example.dddspanner.presentation.validation.ValidEmail;
import jakarta.validation.constraints.Size;
import lombok.Value;

/**
 * ユーザー更新リクエスト
 */
@Value
public class UpdateUserRequest {
    
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters")
    String fullName;
    
    @ValidEmail(message = "Invalid email format")
    String email;
    
    @Size(max = 500, message = "Bio must not exceed 500 characters")
    String bio;
    
    @Size(max = 20, message = "Phone number must not exceed 20 characters")
    String phoneNumber;
}
```

**CreateOrderRequest.java:**
```java
package com.example.dddspanner.presentation.request;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import lombok.Value;

import java.math.BigDecimal;
import java.util.List;

/**
 * 注文作成リクエスト
 */
@Value
public class CreateOrderRequest {
    
    @NotEmpty(message = "Order items are required")
    @Valid
    List<OrderItemRequest> items;
    
    @NotNull(message = "Total amount is required")
    @Positive(message = "Total amount must be positive")
    BigDecimal totalAmount;
    
    @NotNull(message = "Payment method is required")
    String paymentMethod;
    
    @Size(max = 1000, message = "Shipping address must not exceed 1000 characters")
    String shippingAddress;
}
```

**OrderItemRequest.java:**
```java
package com.example.dddspanner.presentation.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import lombok.Value;

/**
 * 注文アイテムリクエスト
 */
@Value
public class OrderItemRequest {
    
    @NotBlank(message = "Product ID is required")
    String productId;
    
    @Positive(message = "Quantity must be positive")
    int quantity;
}
```

#### 7.2.2. レスポンスDTOの設計

**UserResponse.java:**
```java
package com.example.dddspanner.presentation.response;

import com.fasterxml.jackson.annotation.JsonFormat;
import org.springframework.hateoas.RepresentationModel;
import org.springframework.hateoas.server.core.Relation;

import java.time.LocalDateTime;

/**
 * ユーザー情報レスポンス
 */
@Relation(collectionRelation = "users", itemRelation = "user")
public class UserResponse extends RepresentationModel<UserResponse> {
    
    private final String id;
    private final String fullName;
    private final String email;
    private final String status;
    private final String bio;
    private final String phoneNumber;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private final LocalDateTime createdAt;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private final LocalDateTime updatedAt;
    
    // Constructor, getters
    public UserResponse(String id, String fullName, String email, String status, 
                       String bio, String phoneNumber, LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.fullName = fullName;
        this.email = email;
        this.status = status;
        this.bio = bio;
        this.phoneNumber = phoneNumber;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }
    
    // Getters
    public String getId() { return id; }
    public String getFullName() { return fullName; }
    public String getEmail() { return email; }
    public String getStatus() { return status; }
    public String getBio() { return bio; }
    public String getPhoneNumber() { return phoneNumber; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

**OrderResponse.java:**
```java
package com.example.dddspanner.presentation.response;

import com.fasterxml.jackson.annotation.JsonFormat;
import org.springframework.hateoas.RepresentationModel;
import org.springframework.hateoas.server.core.Relation;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

/**
 * 注文情報レスポンス
 */
@Relation(collectionRelation = "orders", itemRelation = "order")
public class OrderResponse extends RepresentationModel<OrderResponse> {
    
    private final String id;
    private final String userId;
    private final String status;
    private final BigDecimal totalAmount;
    private final String paymentMethod;
    private final String shippingAddress;
    private final List<OrderItemResponse> items;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private final LocalDateTime createdAt;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private final LocalDateTime updatedAt;
    
    // Constructor, getters
    public OrderResponse(String id, String userId, String status, BigDecimal totalAmount,
                        String paymentMethod, String shippingAddress, List<OrderItemResponse> items,
                        LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.userId = userId;
        this.status = status;
        this.totalAmount = totalAmount;
        this.paymentMethod = paymentMethod;
        this.shippingAddress = shippingAddress;
        this.items = items;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }
    
    // Getters
    public String getId() { return id; }
    public String getUserId() { return userId; }
    public String getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public String getPaymentMethod() { return paymentMethod; }
    public String getShippingAddress() { return shippingAddress; }
    public List<OrderItemResponse> getItems() { return items; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

**OrderItemResponse.java:**
```java
package com.example.dddspanner.presentation.response;

import lombok.Value;
import java.math.BigDecimal;

/**
 * 注文アイテムレスポンス
 */
@Value
public class OrderItemResponse {
    String productId;
    String productName;
    int quantity;
    BigDecimal unitPrice;
    BigDecimal totalPrice;
}
```

#### 7.2.3. HATEOASによるAPIの自己文書化

HATEOAS (Hypermedia as the Engine of Application State) は、レスポンスにリソースに関連する次のアクションへのリンクを含めるRESTアーキテクチャの原則です。これにより、クライアントはレスポンス内のリンクを辿るだけでAPIを操作でき、URLをハードコーディングする必要がなくなります。

Spring HATEOASを利用すると、このリンク生成を簡単に行えます。

**UserResponseAssembler.java:**
```java
package com.example.dddspanner.presentation.assembler;

import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

import com.example.dddspanner.domain.model.user.User;
import com.example.dddspanner.presentation.controller.UserRestController;
import com.example.dddspanner.presentation.response.UserResponse;
import org.springframework.hateoas.server.mvc.RepresentationModelAssemblerSupport;
import org.springframework.stereotype.Component;

/**
 * ユーザーDTOアセンブラー
 */
@Component
public class UserResponseAssembler extends RepresentationModelAssemblerSupport<User, UserResponse> {

    public UserResponseAssembler() {
        super(UserRestController.class, UserResponse.class);
    }

    @Override
    public UserResponse toModel(User user) {
        UserResponse response = new UserResponse(
            user.getId().getValue(),
            user.getFullName().getFullName(),
            user.getEmail().getValue(),
            user.getStatus().name(),
            user.getBio().orElse(null),
            user.getPhoneNumber().map(phone -> phone.getValue()).orElse(null),
            user.getCreatedAt(),
            user.getUpdatedAt()
        );
        
        // リンクの追加
        response.add(linkTo(methodOn(UserRestController.class).getUserById(user.getId().getValue())).withSelfRel());
        response.add(linkTo(methodOn(UserRestController.class).updateUser(user.getId().getValue(), null)).withRel("update"));
        response.add(linkTo(methodOn(UserRestController.class).deleteUser(user.getId().getValue())).withRel("delete"));
        response.add(linkTo(methodOn(UserRestController.class).getUsers()).withRel("users"));
        
        return response;
    }
}
```

**OrderResponseAssembler.java:**
```java
package com.example.dddspanner.presentation.assembler;

import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

import com.example.dddspanner.domain.model.order.Order;
import com.example.dddspanner.presentation.controller.OrderRestController;
import com.example.dddspanner.presentation.response.OrderResponse;
import com.example.dddspanner.presentation.response.OrderItemResponse;
import org.springframework.hateoas.server.mvc.RepresentationModelAssemblerSupport;
import org.springframework.stereotype.Component;

import java.util.stream.Collectors;

/**
 * 注文DTOアセンブラー
 */
@Component
public class OrderResponseAssembler extends RepresentationModelAssemblerSupport<Order, OrderResponse> {

    public OrderResponseAssembler() {
        super(OrderRestController.class, OrderResponse.class);
    }

    @Override
    public OrderResponse toModel(Order order) {
        List<OrderItemResponse> itemResponses = order.getItems().stream()
            .map(item -> new OrderItemResponse(
                item.getProductId().getValue(),
                item.getProductName(),
                item.getQuantity(),
                item.getUnitPrice().getAmount(),
                item.getTotalPrice().getAmount()
            ))
            .collect(Collectors.toList());
        
        OrderResponse response = new OrderResponse(
            order.getId().getValue(),
            order.getUserId().getValue(),
            order.getStatus().name(),
            order.getTotalAmount().getAmount(),
            order.getPaymentMethod().name(),
            order.getShippingAddress(),
            itemResponses,
            order.getCreatedAt(),
            order.getUpdatedAt()
        );
        
        // リンクの追加
        response.add(linkTo(methodOn(OrderRestController.class).getOrderById(order.getId().getValue())).withSelfRel());
        response.add(linkTo(methodOn(OrderRestController.class).confirmOrder(order.getId().getValue())).withRel("confirm"));
        response.add(linkTo(methodOn(OrderRestController.class).cancelOrder(order.getId().getValue())).withRel("cancel"));
        response.add(linkTo(methodOn(OrderRestController.class).getOrders()).withRel("orders"));
        
        return response;
    }
}
```

#### 7.2.4. ページネーション対応

**PagedResponse.java:**
```java
package com.example.dddspanner.presentation.response;

import com.fasterxml.jackson.annotation.JsonProperty;
import org.springframework.hateoas.RepresentationModel;

import java.util.List;

/**
 * ページネーション対応レスポンス
 */
public class PagedResponse<T> extends RepresentationModel<PagedResponse<T>> {
    
    private final List<T> content;
    private final PageMetadata metadata;
    
    public PagedResponse(List<T> content, PageMetadata metadata) {
        this.content = content;
        this.metadata = metadata;
    }
    
    @JsonProperty("content")
    public List<T> getContent() {
        return content;
    }
    
    @JsonProperty("metadata")
    public PageMetadata getMetadata() {
        return metadata;
    }
    
    /**
     * ページメタデータ
     */
    public static class PageMetadata {
        private final int page;
        private final int size;
        private final long totalElements;
        private final int totalPages;
        private final boolean hasNext;
        private final boolean hasPrevious;
        
        public PageMetadata(int page, int size, long totalElements, int totalPages, 
                          boolean hasNext, boolean hasPrevious) {
            this.page = page;
            this.size = size;
            this.totalElements = totalElements;
            this.totalPages = totalPages;
            this.hasNext = hasNext;
            this.hasPrevious = hasPrevious;
        }
        
        // Getters
        public int getPage() { return page; }
        public int getSize() { return size; }
        public long getTotalElements() { return totalElements; }
        public int getTotalPages() { return totalPages; }
        public boolean isHasNext() { return hasNext; }
        public boolean isHasPrevious() { return hasPrevious; }
    }
}
```

#### 7.2.5. エラーレスポンスの統一

**ErrorResponse.java:**
```java
package com.example.dddspanner.presentation.response;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Value;

import java.time.LocalDateTime;
import java.util.List;

/**
 * エラーレスポンス
 */
@Value
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    
    private final String errorCode;
    private final String message;
    private final String path;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private final LocalDateTime timestamp;
    
    private final List<FieldError> fieldErrors;
    
    public ErrorResponse(String errorCode, String message, String path, 
                        LocalDateTime timestamp, List<FieldError> fieldErrors) {
        this.errorCode = errorCode;
        this.message = message;
        this.path = path;
        this.timestamp = timestamp;
        this.fieldErrors = fieldErrors;
    }
    
    /**
     * フィールドエラー
     */
    @Value
    public static class FieldError {
        String field;
        String message;
        Object rejectedValue;
    }
}
```

#### 7.2.6. コントローラーでの実装例

**UserRestController.java:**
```java
package com.example.dddspanner.presentation.controller;

import com.example.dddspanner.application.command.CreateUserCommand;
import com.example.dddspanner.application.command.UpdateUserCommand;
import com.example.dddspanner.application.service.UserApplicationService;
import com.example.dddspanner.presentation.assembler.UserResponseAssembler;
import com.example.dddspanner.presentation.request.CreateUserRequest;
import com.example.dddspanner.presentation.request.UpdateUserRequest;
import com.example.dddspanner.presentation.response.UserResponse;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.hateoas.PagedModel;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

/**
 * ユーザー管理コントローラー
 */
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
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        CreateUserCommand command = new CreateUserCommand(
            request.getFullName(),
            request.getEmail(),
            request.getPassword()
        );
        
        var user = userApplicationService.createUser(command);
        UserResponse response = userResponseAssembler.toModel(user);
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(response);
    }
    
    /**
     * ユーザー取得
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable String id) {
        var user = userApplicationService.getUser(new UserId(id));
        UserResponse response = userResponseAssembler.toModel(user);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * ユーザー一覧取得（ページネーション対応）
     */
    @GetMapping
    public ResponseEntity<PagedModel<UserResponse>> getUsers(Pageable pageable) {
        Page<User> userPage = userApplicationService.getUsers(pageable);
        
        List<UserResponse> userResponses = userPage.getContent().stream()
            .map(userResponseAssembler::toModel)
            .collect(Collectors.toList());
        
        PagedModel<UserResponse> pagedModel = PagedModel.of(
            userResponses,
            new PagedModel.PageMetadata(
                pageable.getPageSize(),
                pageable.getPageNumber(),
                userPage.getTotalElements(),
                userPage.getTotalPages()
            )
        );
        
        return ResponseEntity.ok(pagedModel);
    }
    
    /**
     * ユーザー更新
     */
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(@PathVariable String id, 
                                                 @Valid @RequestBody UpdateUserRequest request) {
        UpdateUserCommand command = new UpdateUserCommand(
            request.getFullName(),
            request.getEmail()
        );
        
        var user = userApplicationService.updateUser(new UserId(id), command);
        UserResponse response = userResponseAssembler.toModel(user);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * ユーザー削除
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable String id) {
        userApplicationService.deleteUser(new UserId(id));
        return ResponseEntity.noContent().build();
    }
}
```

#### 7.2.7. リクエスト・レスポンス設計のベストプラクティス

**1. バージョニング戦略**
```java
// URLベースのバージョニング
@RequestMapping("/api/v1/users")
@RequestMapping("/api/v2/users")

// ヘッダーベースのバージョニング
@GetMapping(value = "/users", headers = "API-Version=1")
@GetMapping(value = "/users", headers = "API-Version=2")
```

**2. フィルタリングとソート**
```java
@GetMapping("/users")
public ResponseEntity<PagedModel<UserResponse>> getUsers(
    @RequestParam(required = false) String email,
    @RequestParam(required = false) String status,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") String sortDirection,
    Pageable pageable) {
    // 実装
}
```

**3. 部分更新（PATCH）の実装**
```java
@PatchMapping("/{id}")
public ResponseEntity<UserResponse> partialUpdateUser(@PathVariable String id,
                                                    @RequestBody Map<String, Object> updates) {
    // 部分更新の実装
}
```

**4. バッチ操作**
```java
@PostMapping("/batch")
public ResponseEntity<List<UserResponse>> createUsers(@Valid @RequestBody List<CreateUserRequest> requests) {
    // バッチ作成の実装
}
```

この包括的なリクエスト・レスポンス設計により、堅牢で使いやすいAPIを構築できます。 