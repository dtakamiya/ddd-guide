## 5. アプリケーション層の実装

アプリケーション層は、ユースケースの調整とオーケストレーションを担当します。この章では、ドメイン層のオブジェクトを活用して、ビジネスフローを実装する方法を学びます。

### 5.1. アプリケーション層の役割と責任

#### 5.1.1. アプリケーション層の位置づけ

アプリケーション層は、オニオンアーキテクチャの中間層として以下の役割を担います：

**1. ユースケースの実装**
*   ビジネスフローの調整
*   複数のドメインオブジェクトの協調
*   トランザクション境界の管理

**2. 外部システムとの連携調整**
*   ドメイン層とインフラストラクチャ層の橋渡し
*   外部APIの呼び出し調整
*   メッセージングの制御

**3. セキュリティと認証**
*   ユーザー認証の確認
*   権限チェックの実行
*   セキュリティコンテキストの管理

#### 5.1.2. アプリケーション層の設計原則

**1. 薄いアプリケーション層**
*   ビジネスロジックはドメイン層に配置
*   アプリケーション層は調整のみを担当
*   複雑なロジックを避ける

**2. トランザクション境界の管理**
*   一つのユースケース = 一つのトランザクション
*   適切なトランザクション境界の設定
*   分散トランザクションの回避

**3. エラーハンドリング**
*   ドメイン例外の適切な処理
*   技術的例外の変換
*   クライアントに適切なエラー情報を提供

### 5.2. アプリケーションサービスの実装

#### 5.2.1. 基本的なアプリケーションサービス

**UserApplicationService.java:**
```java
package com.example.dddspanner.application.service;

import com.example.dddspanner.application.dto.*;
import com.example.dddspanner.application.mapper.UserMapper;
import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.factory.UserFactory;
import com.example.dddspanner.domain.repository.UserRepository;
import com.example.dddspanner.domain.service.UserDuplicateChecker;
import com.example.dddspanner.domain.valueobject.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

/**
 * ユーザーアプリケーションサービス
 * 
 * アプリケーションサービスの特徴：
 * - ユースケースの調整
 * - トランザクション境界の管理
 * - ドメインオブジェクトの協調
 */
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class UserApplicationService {
    
    private final UserRepository userRepository;
    private final UserFactory userFactory;
    private final UserDuplicateChecker duplicateChecker;
    private final UserMapper userMapper;
    
    /**
     * ユーザー登録
     * 
     * ユースケース：
     * 1. 入力値の検証
     * 2. 重複チェック
     * 3. ユーザー作成
     * 4. 永続化
     * 5. 結果の返却
     */
    public UserResponse registerUser(CreateUserRequest request) {
        log.info("User registration requested: {}", request.email());
        
        try {
            // 1. 入力値の検証
            validateCreateUserRequest(request);
            
            // 2. 値オブジェクトの作成
            FullName fullName = new FullName(request.firstName(), request.lastName());
            Email email = new Email(request.email());
            
            // 3. 重複チェック
            duplicateChecker.validateUserCreation(email);
            
            // 4. ユーザー作成
            User user = userFactory.createUser(fullName, email);
            
            // 5. 永続化
            User savedUser = userRepository.save(user);
            
            // 6. ドメインイベントの処理（非同期）
            publishDomainEvents(savedUser);
            
            log.info("User registered successfully: {}", savedUser.getId());
            
            // 7. レスポンスの作成
            return userMapper.toResponse(savedUser);
            
        } catch (Exception e) {
            log.error("User registration failed: {}", e.getMessage(), e);
            throw new UserRegistrationException("Failed to register user", e);
        }
    }
    
    /**
     * ユーザー情報取得
     */
    @Transactional(readOnly = true)
    public Optional<UserResponse> getUser(String userId) {
        try {
            UserId id = UserId.of(userId);
            return userRepository.findById(id)
                .map(userMapper::toResponse);
        } catch (Exception e) {
            log.error("Failed to get user: {}", e.getMessage(), e);
            throw new UserNotFoundException("User not found: " + userId, e);
        }
    }
    
    /**
     * ユーザー情報更新
     */
    public UserResponse updateUser(String userId, UpdateUserRequest request) {
        log.info("User update requested: {}", userId);
        
        try {
            UserId id = UserId.of(userId);
            User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
            
            // 名前の更新
            if (request.firstName() != null || request.lastName() != null) {
                String firstName = request.firstName() != null ? request.firstName() : user.getFullName().firstName();
                String lastName = request.lastName() != null ? request.lastName() : user.getFullName().lastName();
                FullName newFullName = new FullName(firstName, lastName);
                user.changeName(newFullName);
            }
            
            // メールアドレスの更新
            if (request.email() != null) {
                Email newEmail = new Email(request.email());
                duplicateChecker.validateEmailChange(newEmail, user.getId());
                user.changeEmail(newEmail);
            }
            
            // 永続化
            User updatedUser = userRepository.save(user);
            
            // ドメインイベントの処理
            publishDomainEvents(updatedUser);
            
            log.info("User updated successfully: {}", updatedUser.getId());
            
            return userMapper.toResponse(updatedUser);
            
        } catch (Exception e) {
            log.error("User update failed: {}", e.getMessage(), e);
            throw new UserUpdateException("Failed to update user", e);
        }
    }
    
    /**
     * ユーザー削除
     */
    public void deleteUser(String userId) {
        log.info("User deletion requested: {}", userId);
        
        try {
            UserId id = UserId.of(userId);
            User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
            
            user.deactivate();
            userRepository.save(user);
            
            // ドメインイベントの処理
            publishDomainEvents(user);
            
            log.info("User deleted successfully: {}", userId);
            
        } catch (Exception e) {
            log.error("User deletion failed: {}", e.getMessage(), e);
            throw new UserDeletionException("Failed to delete user", e);
        }
    }
    
    /**
     * ユーザー一覧取得
     */
    @Transactional(readOnly = true)
    public List<UserResponse> getAllUsers() {
        try {
            return userRepository.findAll().stream()
                .map(userMapper::toResponse)
                .toList();
        } catch (Exception e) {
            log.error("Failed to get all users: {}", e.getMessage(), e);
            throw new UserQueryException("Failed to get users", e);
        }
    }
    
    /**
     * アクティブユーザー一覧取得
     */
    @Transactional(readOnly = true)
    public List<UserResponse> getActiveUsers() {
        try {
            return userRepository.findActiveUsers().stream()
                .map(userMapper::toResponse)
                .toList();
        } catch (Exception e) {
            log.error("Failed to get active users: {}", e.getMessage(), e);
            throw new UserQueryException("Failed to get active users", e);
        }
    }
    
    /**
     * 入力値の検証
     */
    private void validateCreateUserRequest(CreateUserRequest request) {
        if (request == null) {
            throw new IllegalArgumentException("Request cannot be null");
        }
        if (request.firstName() == null || request.firstName().trim().isEmpty()) {
            throw new IllegalArgumentException("First name is required");
        }
        if (request.lastName() == null || request.lastName().trim().isEmpty()) {
            throw new IllegalArgumentException("Last name is required");
        }
        if (request.email() == null || request.email().trim().isEmpty()) {
            throw new IllegalArgumentException("Email is required");
        }
    }
    
    /**
     * ドメインイベントの非同期発行
     */
    private void publishDomainEvents(User user) {
        // 実際の実装では、イベントバスやメッセージングシステムを使用
        List<DomainEvent> events = user.getDomainEvents();
        if (!events.isEmpty()) {
            log.debug("Publishing {} domain events for user: {}", events.size(), user.getId());
            // イベントの発行処理
        }
    }
}
```

#### 5.2.2. 複雑なアプリケーションサービス

**OrderApplicationService.java:**
```java
package com.example.dddspanner.application.service;

import com.example.dddspanner.application.dto.*;
import com.example.dddspanner.application.mapper.OrderMapper;
import com.example.dddspanner.domain.entity.Order;
import com.example.dddspanner.domain.entity.Product;
import com.example.dddspanner.domain.repository.OrderRepository;
import com.example.dddspanner.domain.repository.ProductRepository;
import com.example.dddspanner.domain.repository.UserRepository;
import com.example.dddspanner.domain.service.OrderCalculationService;
import com.example.dddspanner.domain.valueobject.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

/**
 * 注文アプリケーションサービス
 * 
 * 複雑なビジネスフローの調整
 */
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class OrderApplicationService {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final OrderCalculationService calculationService;
    private final OrderMapper orderMapper;
    
    /**
     * 注文作成
     */
    public OrderResponse createOrder(String userId, CreateOrderRequest request) {
        log.info("Order creation requested by user: {}", userId);
        
        try {
            // 1. ユーザーの存在確認
            UserId user = UserId.of(userId);
            if (!userRepository.existsById(user)) {
                throw new UserNotFoundException("User not found: " + userId);
            }
            
            // 2. 注文作成
            Order order = Order.create(user);
            
            // 3. 商品の追加
            for (OrderItemRequest itemRequest : request.items()) {
                ProductId productId = ProductId.of(itemRequest.productId());
                Product product = productRepository.findById(productId)
                    .orElseThrow(() -> new ProductNotFoundException("Product not found: " + itemRequest.productId()));
                
                order.addItem(product, itemRequest.quantity());
            }
            
            // 4. 永続化
            Order savedOrder = orderRepository.save(order);
            
            // 5. ドメインイベントの処理
            publishDomainEvents(savedOrder);
            
            log.info("Order created successfully: {}", savedOrder.getId());
            
            return orderMapper.toResponse(savedOrder);
            
        } catch (Exception e) {
            log.error("Order creation failed: {}", e.getMessage(), e);
            throw new OrderCreationException("Failed to create order", e);
        }
    }
    
    /**
     * 注文確定
     */
    public OrderResponse confirmOrder(String orderId) {
        log.info("Order confirmation requested: {}", orderId);
        
        try {
            OrderId id = OrderId.of(orderId);
            Order order = orderRepository.findById(id)
                .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));
            
            order.confirm();
            Order confirmedOrder = orderRepository.save(order);
            
            // ドメインイベントの処理
            publishDomainEvents(confirmedOrder);
            
            log.info("Order confirmed successfully: {}", orderId);
            
            return orderMapper.toResponse(confirmedOrder);
            
        } catch (Exception e) {
            log.error("Order confirmation failed: {}", e.getMessage(), e);
            throw new OrderConfirmationException("Failed to confirm order", e);
        }
    }
    
    /**
     * 注文キャンセル
     */
    public OrderResponse cancelOrder(String orderId) {
        log.info("Order cancellation requested: {}", orderId);
        
        try {
            OrderId id = OrderId.of(orderId);
            Order order = orderRepository.findById(id)
                .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));
            
            order.cancel();
            Order cancelledOrder = orderRepository.save(order);
            
            // ドメインイベントの処理
            publishDomainEvents(cancelledOrder);
            
            log.info("Order cancelled successfully: {}", orderId);
            
            return orderMapper.toResponse(cancelledOrder);
            
        } catch (Exception e) {
            log.error("Order cancellation failed: {}", e.getMessage(), e);
            throw new OrderCancellationException("Failed to cancel order", e);
        }
    }
    
    /**
     * 注文詳細取得
     */
    @Transactional(readOnly = true)
    public Optional<OrderResponse> getOrder(String orderId) {
        try {
            OrderId id = OrderId.of(orderId);
            return orderRepository.findById(id)
                .map(orderMapper::toResponse);
        } catch (Exception e) {
            log.error("Failed to get order: {}", e.getMessage(), e);
            throw new OrderQueryException("Failed to get order", e);
        }
    }
    
    /**
     * ユーザーの注文一覧取得
     */
    @Transactional(readOnly = true)
    public List<OrderResponse> getUserOrders(String userId) {
        try {
            UserId user = UserId.of(userId);
            return orderRepository.findByUserId(user).stream()
                .map(orderMapper::toResponse)
                .toList();
        } catch (Exception e) {
            log.error("Failed to get user orders: {}", e.getMessage(), e);
            throw new OrderQueryException("Failed to get user orders", e);
        }
    }
    
    /**
     * ドメインイベントの非同期発行
     */
    private void publishDomainEvents(Order order) {
        List<DomainEvent> events = order.getDomainEvents();
        if (!events.isEmpty()) {
            log.debug("Publishing {} domain events for order: {}", events.size(), order.getId());
            // イベントの発行処理
        }
    }
}
```

### 5.3. コマンド・クエリオブジェクトの実装

#### 5.3.1. コマンドオブジェクト

**CreateUserCommand.java:**
```java
package com.example.dddspanner.application.command;

import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.valueobject.FullName;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

/**
 * ユーザー作成コマンド
 * 
 * コマンドオブジェクトの特徴：
 * - 入力値の検証
 * - 不変オブジェクト
 * - ビジネス意図の明確化
 */
public record CreateUserCommand(
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
     * ドメインオブジェクトへの変換
     */
    public FullName toFullName() {
        return new FullName(firstName, lastName);
    }
    
    public Email toEmail() {
        return new Email(email);
    }
}
```

**UpdateUserCommand.java:**
```java
package com.example.dddspanner.application.command;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Size;

/**
 * ユーザー更新コマンド
 */
public record UpdateUserCommand(
    @Size(max = 50, message = "First name cannot exceed 50 characters")
    String firstName,
    
    @Size(max = 50, message = "Last name cannot exceed 50 characters")
    String lastName,
    
    @Email(message = "Invalid email format")
    String email
) {
    
    public boolean hasNameUpdate() {
        return firstName != null || lastName != null;
    }
    
    public boolean hasEmailUpdate() {
        return email != null;
    }
}
```

#### 5.3.2. クエリオブジェクト

**UserQuery.java:**
```java
package com.example.dddspanner.application.query;

import java.util.Optional;

/**
 * ユーザー検索クエリ
 */
public record UserQuery(
    Optional<String> email,
    Optional<String> status,
    Optional<Integer> page,
    Optional<Integer> size
) {
    
    public static UserQuery all() {
        return new UserQuery(Optional.empty(), Optional.empty(), Optional.empty(), Optional.empty());
    }
    
    public static UserQuery byEmail(String email) {
        return new UserQuery(Optional.of(email), Optional.empty(), Optional.empty(), Optional.empty());
    }
    
    public static UserQuery byStatus(String status) {
        return new UserQuery(Optional.empty(), Optional.of(status), Optional.empty(), Optional.empty());
    }
    
    public static UserQuery paginated(int page, int size) {
        return new UserQuery(Optional.empty(), Optional.empty(), Optional.of(page), Optional.of(size));
    }
}
```

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

### 5.5. オブジェクトマッピングの実装

#### 5.5.1. MapStructを使用したマッピング

**UserMapper.java:**
```java
package com.example.dddspanner.application.mapper;

import com.example.dddspanner.application.dto.UserResponse;
import com.example.dddspanner.domain.entity.User;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Named;

/**
 * ユーザーオブジェクトマッパー
 * 
 * MapStructの特徴：
 * - コンパイル時にマッピングコードを生成
 * - 型安全性
 * - パフォーマンスの向上
 */
@Mapper(componentModel = "spring")
public interface UserMapper {
    
    /**
     * ドメインオブジェクトからレスポンスDTOへの変換
     */
    @Mapping(source = "id.value", target = "id")
    @Mapping(source = "fullName.firstName", target = "firstName")
    @Mapping(source = "fullName.lastName", target = "lastName")
    @Mapping(source = "fullName", target = "fullName", qualifiedByName = "fullNameToString")
    @Mapping(source = "email.value", target = "email")
    @Mapping(source = "status", target = "status", qualifiedByName = "statusToString")
    UserResponse toResponse(User user);
    
    /**
     * フルネームを文字列に変換
     */
    @Named("fullNameToString")
    default String fullNameToString(FullName fullName) {
        return fullName != null ? fullName.getFullName() : null;
    }
    
    /**
     * ステータスを文字列に変換
     */
    @Named("statusToString")
    default String statusToString(UserStatus status) {
        return status != null ? status.name() : null;
    }
}
```

**OrderMapper.java:**
```java
package com.example.dddspanner.application.mapper;

import com.example.dddspanner.application.dto.OrderResponse;
import com.example.dddspanner.application.dto.OrderItemResponse;
import com.example.dddspanner.domain.entity.Order;
import com.example.dddspanner.domain.entity.OrderItem;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

/**
 * 注文オブジェクトマッパー
 */
@Mapper(componentModel = "spring", uses = {OrderItemMapper.class})
public interface OrderMapper {
    
    @Mapping(source = "id.value", target = "id")
    @Mapping(source = "userId.value", target = "userId")
    @Mapping(source = "totalAmount.amount", target = "totalAmount")
    @Mapping(source = "totalAmount.currency.currencyCode", target = "currency")
    @Mapping(source = "status", target = "status")
    OrderResponse toResponse(Order order);
}

/**
 * 注文商品オブジェクトマッパー
 */
@Mapper(componentModel = "spring")
public interface OrderItemMapper {
    
    @Mapping(source = "productId.value", target = "productId")
    @Mapping(source = "product.name", target = "productName")
    @Mapping(source = "unitPrice.amount", target = "unitPrice")
    @Mapping(source = "subtotal.amount", target = "subtotal")
    OrderItemResponse toResponse(OrderItem orderItem);
}
```

### 5.6. バリデーションの実装

#### 5.6.1. カスタムバリデーター

**EmailValidator.java:**
```java
package com.example.dddspanner.application.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

/**
 * メールアドレスバリデーター
 */
public class EmailValidator implements ConstraintValidator<ValidEmail, String> {
    
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.trim().isEmpty()) {
            return false;
        }
        
        return EMAIL_PATTERN.matcher(email.trim()).matches();
    }
}

/**
 * メールアドレスバリデーションアノテーション
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = EmailValidator.class)
public @interface ValidEmail {
    String message() default "Invalid email format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

#### 5.6.2. バリデーショングループ

**ValidationGroups.java:**
```java
package com.example.dddspanner.application.validation;

/**
 * バリデーショングループ
 */
public interface ValidationGroups {
    
    interface Create {}
    interface Update {}
    interface Delete {}
}
```

### 5.7. 例外処理の実装

#### 5.7.1. アプリケーション例外

**UserRegistrationException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * ユーザー登録例外
 */
public class UserRegistrationException extends RuntimeException {
    
    public UserRegistrationException(String message) {
        super(message);
    }
    
    public UserRegistrationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**UserNotFoundException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * ユーザー未発見例外
 */
public class UserNotFoundException extends RuntimeException {
    
    public UserNotFoundException(String message) {
        super(message);
    }
    
    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### 5.7.2. 例外ハンドラー

**ApplicationExceptionHandler.java:**
```java
package com.example.dddspanner.application.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.Instant;
import java.util.Map;

/**
 * アプリケーション例外ハンドラー
 */
@Slf4j
@RestControllerAdvice
public class ApplicationExceptionHandler {
    
    @ExceptionHandler(UserRegistrationException.class)
    public ResponseEntity<Map<String, Object>> handleUserRegistrationException(UserRegistrationException e) {
        log.error("User registration failed", e);
        
        Map<String, Object> response = Map.of(
            "error", "USER_REGISTRATION_FAILED",
            "message", e.getMessage(),
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleUserNotFoundException(UserNotFoundException e) {
        log.error("User not found", e);
        
        Map<String, Object> response = Map.of(
            "error", "USER_NOT_FOUND",
            "message", e.getMessage(),
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, Object>> handleIllegalArgumentException(IllegalArgumentException e) {
        log.error("Invalid argument", e);
        
        Map<String, Object> response = Map.of(
            "error", "INVALID_ARGUMENT",
            "message", e.getMessage(),
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
}
```

### 5.8. アプリケーション層のテスト

#### 5.8.1. アプリケーションサービスのテスト

**UserApplicationServiceTest.groovy:**
```groovy
package com.example.dddspanner.application.service

import com.example.dddspanner.application.dto.CreateUserRequest
import com.example.dddspanner.application.dto.UserResponse
import com.example.dddspanner.application.exception.UserRegistrationException
import com.example.dddspanner.application.mapper.UserMapper
import com.example.dddspanner.domain.entity.User
import com.example.dddspanner.domain.factory.UserFactory
import com.example.dddspanner.domain.repository.UserRepository
import com.example.dddspanner.domain.service.UserDuplicateChecker
import com.example.dddspanner.domain.valueobject.Email
import com.example.dddspanner.domain.valueobject.FullName
import com.example.dddspanner.domain.valueobject.UserId
import spock.lang.Specification

class UserApplicationServiceTest extends Specification {
    
    UserRepository userRepository = Mock()
    UserFactory userFactory = Mock()
    UserDuplicateChecker duplicateChecker = Mock()
    UserMapper userMapper = Mock()
    
    UserApplicationService service
    
    def setup() {
        service = new UserApplicationService(
            userRepository, userFactory, duplicateChecker, userMapper
        )
    }
    
    def "should register user successfully"() {
        given:
        def request = new CreateUserRequest("Taro", "Yamada", "taro.yamada@example.com")
        def fullName = new FullName("Taro", "Yamada")
        def email = new Email("taro.yamada@example.com")
        def user = Mock(User)
        def userId = UserId.generate()
        def savedUser = Mock(User)
        def response = Mock(UserResponse)
        
        when:
        def result = service.registerUser(request)
        
        then:
        1 * duplicateChecker.validateUserCreation(email)
        1 * userFactory.createUser(fullName, email) >> user
        1 * userRepository.save(user) >> savedUser
        1 * userMapper.toResponse(savedUser) >> response
        
        and:
        result == response
    }
    
    def "should throw exception when user creation fails"() {
        given:
        def request = new CreateUserRequest("Taro", "Yamada", "taro.yamada@example.com")
        def email = new Email("taro.yamada@example.com")
        
        and:
        duplicateChecker.validateUserCreation(email) >> { throw new RuntimeException("Database error") }
        
        when:
        service.registerUser(request)
        
        then:
        thrown(UserRegistrationException)
    }
    
    def "should throw exception for invalid request"() {
        when:
        service.registerUser(null)
        
        then:
        thrown(IllegalArgumentException)
    }
}
```

### 5.9. まとめ

この章では、アプリケーション層の実装について詳しく説明しました。重要なポイントをまとめます：

**1. アプリケーションサービスの設計**
*   ユースケースの調整とオーケストレーション
*   トランザクション境界の管理
*   エラーハンドリングの実装

**2. コマンド・クエリオブジェクト**
*   入力値の検証とビジネス意図の明確化
*   型安全性の向上
*   再利用可能な設計

**3. DTOの活用**
*   プレゼンテーション層との境界
*   技術的な詳細の隠蔽
*   バリデーションの実装

**4. オブジェクトマッピング**
*   MapStructによる効率的なマッピング
*   型安全性の保証
*   パフォーマンスの向上

**5. 例外処理**
*   適切な例外設計
*   グローバル例外ハンドラー
*   クライアントへの適切なエラー情報提供

次の章では、インフラストラクチャ層を実装して、外部システムとの連携を実現します。 