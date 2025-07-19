### 5.4. DTO（Data Transfer Object）の実装

#### 5.4.1. リクエストDTO

**CreateUserRequest.java:**
```java
package com.example.dddspanner.application.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

/**
 * ユーザー作成リクエストDTO
 * 
 * DTOの特徴：
 * - プレゼンテーション層との境界
 * - 入力値の検証
 * - 技術的な詳細の隠蔽
 */
public record CreateUserRequest(
    @NotBlank(message = "First name is required")
    @Size(max = 50, message = "First name cannot exceed 50 characters")
    String firstName,
    
    @NotBlank(message = "Last name is required")
    @Size(max = 50, message = "Last name cannot exceed 50 characters")
    String lastName,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email
) {
    
    /**
     * コマンドオブジェクトへの変換
     */
    public CreateUserCommand toCommand() {
        return new CreateUserCommand(firstName, lastName, email);
    }
}
```

**UpdateUserRequest.java:**
```java
package com.example.dddspanner.application.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Size;

/**
 * ユーザー更新リクエストDTO
 */
public record UpdateUserRequest(
    @Size(max = 50, message = "First name cannot exceed 50 characters")
    String firstName,
    
    @Size(max = 50, message = "Last name cannot exceed 50 characters")
    String lastName,
    
    @Email(message = "Invalid email format")
    String email
) {
    
    /**
     * コマンドオブジェクトへの変換
     */
    public UpdateUserCommand toCommand() {
        return new UpdateUserCommand(firstName, lastName, email);
    }
}
```

#### 5.4.2. レスポンスDTO

**UserResponse.java:**
```java
package com.example.dddspanner.application.dto;

import com.fasterxml.jackson.annotation.JsonFormat;
import java.time.Instant;

/**
 * ユーザー情報レスポンスDTO
 */
public record UserResponse(
    String id,
    String firstName,
    String lastName,
    String fullName,
    String email,
    String status,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    Instant createdAt,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    Instant updatedAt
) {
    
    /**
     * フルネームの取得
     */
    public String getFullName() {
        return lastName + " " + firstName;
    }
}
```

**OrderResponse.java:**
```java
package com.example.dddspanner.application.dto;

import com.fasterxml.jackson.annotation.JsonFormat;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

/**
 * 注文情報レスポンスDTO
 */
public record OrderResponse(
    String id,
    String userId,
    List<OrderItemResponse> items,
    String status,
    BigDecimal totalAmount,
    String currency,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    Instant createdAt,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    Instant updatedAt
) {
    
    /**
     * 商品数の取得
     */
    public int getItemCount() {
        return items != null ? items.size() : 0;
    }
    
    /**
     * 注文が確定済みかチェック
     */
    public boolean isConfirmed() {
        return "CONFIRMED".equals(status);
    }
}

/**
 * 注文商品情報レスポンスDTO
 */
public record OrderItemResponse(
    String productId,
    String productName,
    int quantity,
    BigDecimal unitPrice,
    BigDecimal subtotal
) {}
``` 