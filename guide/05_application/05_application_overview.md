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

### 5.2. アプリケーション層の構成要素

#### 5.2.1. アプリケーションサービス

アプリケーションサービスは、ユースケースを実装する主要なコンポーネントです。

**UserApplicationService.java:**
```java
package com.example.dddspanner.application.service;

import com.example.dddspanner.application.command.CreateUserCommand;
import com.example.dddspanner.application.command.UpdateUserCommand;
import com.example.dddspanner.application.dto.UserDto;
import com.example.dddspanner.domain.model.user.User;
import com.example.dddspanner.domain.model.user.UserId;
import com.example.dddspanner.domain.repository.UserRepository;
import com.example.dddspanner.domain.service.UserDuplicateChecker;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

/**
 * ユーザー管理アプリケーションサービス
 */
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserApplicationService {
    
    private final UserRepository userRepository;
    private final UserDuplicateChecker userDuplicateChecker;
    private final UserDtoMapper userDtoMapper;
    
    /**
     * ユーザー作成
     */
    @Transactional
    public UserDto createUser(CreateUserCommand command) {
        // 重複チェック
        if (userDuplicateChecker.isDuplicate(command.getEmail())) {
            throw new UserAlreadyExistsException("User with email " + command.getEmail() + " already exists");
        }
        
        // ドメインオブジェクトの作成
        User user = UserFactory.createUser(
            command.getFullName(),
            command.getEmail(),
            command.getPassword()
        );
        
        // 永続化
        User savedUser = userRepository.save(user);
        
        // DTOに変換して返却
        return userDtoMapper.toDto(savedUser);
    }
    
    /**
     * ユーザー更新
     */
    @Transactional
    public UserDto updateUser(UserId userId, UpdateUserCommand command) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
        
        // ドメインオブジェクトの更新
        user.updateProfile(command.getFullName(), command.getEmail());
        
        // 永続化
        User updatedUser = userRepository.save(user);
        
        return userDtoMapper.toDto(updatedUser);
    }
    
    /**
     * ユーザー削除
     */
    @Transactional
    public void deleteUser(UserId userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
        
        userRepository.delete(user);
    }
    
    /**
     * ユーザー取得
     */
    public UserDto getUser(UserId userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
        
        return userDtoMapper.toDto(user);
    }
    
    /**
     * ユーザー一覧取得
     */
    public List<UserDto> getUsers() {
        List<User> users = userRepository.findAll();
        return users.stream()
            .map(userDtoMapper::toDto)
            .collect(Collectors.toList());
    }
}
```

#### 5.2.2. コマンド・クエリオブジェクト

**CreateUserCommand.java:**
```java
package com.example.dddspanner.application.command;

import com.example.dddspanner.application.validation.ValidEmail;
import com.example.dddspanner.application.validation.ValidPassword;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Value;

/**
 * ユーザー作成コマンド
 */
@Value
public class CreateUserCommand {
    
    @NotBlank(message = "Full name is required")
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters")
    String fullName;
    
    @NotBlank(message = "Email is required")
    @ValidEmail(message = "Invalid email format")
    String email;
    
    @NotBlank(message = "Password is required")
    @ValidPassword
    String password;
}
```

**UpdateUserCommand.java:**
```java
package com.example.dddspanner.application.command;

import com.example.dddspanner.application.validation.ValidEmail;
import jakarta.validation.constraints.Size;
import lombok.Value;

/**
 * ユーザー更新コマンド
 */
@Value
public class UpdateUserCommand {
    
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters")
    String fullName;
    
    @ValidEmail(message = "Invalid email format")
    String email;
}
```

#### 5.2.3. DTOとマッピング

**UserDto.java:**
```java
package com.example.dddspanner.application.dto;

import lombok.Value;
import java.time.LocalDateTime;

/**
 * ユーザーDTO
 */
@Value
public class UserDto {
    String id;
    String fullName;
    String email;
    String status;
    LocalDateTime createdAt;
    LocalDateTime updatedAt;
}
```

**UserDtoMapper.java:**
```java
package com.example.dddspanner.application.dto;

import com.example.dddspanner.domain.model.user.User;
import org.springframework.stereotype.Component;

/**
 * ユーザーDTOマッパー
 */
@Component
public class UserDtoMapper {
    
    public UserDto toDto(User user) {
        return new UserDto(
            user.getId().getValue(),
            user.getFullName().getFullName(),
            user.getEmail().getValue(),
            user.getStatus().name(),
            user.getCreatedAt(),
            user.getUpdatedAt()
        );
    }
}
```

### 5.3. トランザクション管理

#### 5.3.1. トランザクション境界の設計

**OrderApplicationService.java:**
```java
package com.example.dddspanner.application.service;

import com.example.dddspanner.application.command.CreateOrderCommand;
import com.example.dddspanner.application.dto.OrderDto;
import com.example.dddspanner.domain.model.order.Order;
import com.example.dddspanner.domain.model.user.User;
import com.example.dddspanner.domain.repository.OrderRepository;
import com.example.dddspanner.domain.repository.UserRepository;
import com.example.dddspanner.domain.service.OrderCalculationService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 注文管理アプリケーションサービス
 */
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderApplicationService {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final OrderCalculationService orderCalculationService;
    private final OrderDtoMapper orderDtoMapper;
    
    /**
     * 注文作成（トランザクション境界）
     */
    @Transactional
    public OrderDto createOrder(CreateOrderCommand command) {
        // 1. ユーザーの存在確認
        User user = userRepository.findById(command.getUserId())
            .orElseThrow(() -> new UserNotFoundException("User not found"));
        
        // 2. 注文の作成
        Order order = OrderFactory.createOrder(
            user.getId(),
            command.getItems(),
            command.getTotalAmount()
        );
        
        // 3. 注文金額の計算と検証
        Money calculatedTotal = orderCalculationService.calculateTotal(order);
        if (!calculatedTotal.equals(command.getTotalAmount())) {
            throw new InvalidOrderAmountException("Order amount mismatch");
        }
        
        // 4. 注文の永続化
        Order savedOrder = orderRepository.save(order);
        
        // 5. 注文確認イベントの発行
        order.confirm();
        
        return orderDtoMapper.toDto(savedOrder);
    }
}
```

#### 5.3.2. 分散トランザクションの回避

**Sagaパターンの実装例:**
```java
package com.example.dddspanner.application.saga;

import com.example.dddspanner.application.command.CreateOrderCommand;
import com.example.dddspanner.domain.event.OrderCreatedEvent;
import com.example.dddspanner.domain.event.PaymentProcessedEvent;
import com.example.dddspanner.domain.event.InventoryReservedEvent;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

/**
 * 注文作成Saga
 */
@Service
@RequiredArgsConstructor
public class CreateOrderSaga {
    
    private final OrderApplicationService orderApplicationService;
    private final PaymentApplicationService paymentApplicationService;
    private final InventoryApplicationService inventoryApplicationService;
    
    /**
     * Sagaの実行
     */
    public void execute(CreateOrderCommand command) {
        try {
            // Step 1: 注文作成
            OrderDto order = orderApplicationService.createOrder(command);
            
            // Step 2: 在庫確保
            inventoryApplicationService.reserveInventory(order.getId(), command.getItems());
            
            // Step 3: 支払い処理
            paymentApplicationService.processPayment(order.getId(), command.getPaymentMethod());
            
            // Step 4: 注文確定
            orderApplicationService.confirmOrder(order.getId());
            
        } catch (Exception e) {
            // 補償トランザクションの実行
            compensate(e);
        }
    }
    
    /**
     * 補償トランザクション
     */
    private void compensate(Exception e) {
        // 各ステップの補償処理を実行
        // 1. 支払いのキャンセル
        // 2. 在庫の解放
        // 3. 注文のキャンセル
    }
}
```

### 5.4. エラーハンドリング

#### 5.4.1. アプリケーション例外の定義

**ApplicationException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * アプリケーション例外の基底クラス
 */
public abstract class ApplicationException extends RuntimeException {
    
    private final String errorCode;
    
    protected ApplicationException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
    
    protected ApplicationException(String message, String errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() {
        return errorCode;
    }
}
```

**UserNotFoundException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * ユーザーが見つからない例外
 */
public class UserNotFoundException extends ApplicationException {
    
    public UserNotFoundException(String message) {
        super(message, "USER_NOT_FOUND");
    }
}
```

**UserAlreadyExistsException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * ユーザーが既に存在する例外
 */
public class UserAlreadyExistsException extends ApplicationException {
    
    public UserAlreadyExistsException(String message) {
        super(message, "USER_ALREADY_EXISTS");
    }
}
```

#### 5.4.2. 例外変換サービス

**ExceptionTranslationService.java:**
```java
package com.example.dddspanner.application.service;

import com.example.dddspanner.application.exception.ApplicationException;
import com.example.dddspanner.domain.exception.DomainException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * 例外変換サービス
 */
@Slf4j
@Service
public class ExceptionTranslationService {
    
    /**
     * ドメイン例外をアプリケーション例外に変換
     */
    public ApplicationException translate(DomainException e) {
        log.error("Domain exception occurred: {}", e.getMessage(), e);
        
        // ドメイン例外の種類に応じて適切なアプリケーション例外に変換
        return new ApplicationException(e.getMessage(), "DOMAIN_ERROR", e);
    }
    
    /**
     * 技術的例外をアプリケーション例外に変換
     */
    public ApplicationException translate(TechnicalException e) {
        log.error("Technical exception occurred: {}", e.getMessage(), e);
        
        return new ApplicationException("Internal server error", "TECHNICAL_ERROR", e);
    }
}
```

### 5.5. セキュリティと認証

#### 5.5.1. セキュリティコンテキスト

**SecurityContext.java:**
```java
package com.example.dddspanner.application.security;

import lombok.Value;
import java.util.Set;

/**
 * セキュリティコンテキスト
 */
@Value
public class SecurityContext {
    String userId;
    String username;
    Set<String> roles;
    Set<String> permissions;
    
    public boolean hasRole(String role) {
        return roles.contains(role);
    }
    
    public boolean hasPermission(String permission) {
        return permissions.contains(permission);
    }
}
```

**SecurityContextHolder.java:**
```java
package com.example.dddspanner.application.security;

import org.springframework.stereotype.Component;

/**
 * セキュリティコンテキストホルダー
 */
@Component
public class SecurityContextHolder {
    
    private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();
    
    public void setContext(SecurityContext context) {
        contextHolder.set(context);
    }
    
    public SecurityContext getContext() {
        return contextHolder.get();
    }
    
    public void clearContext() {
        contextHolder.remove();
    }
}
```

#### 5.5.2. 権限チェックサービス

**AuthorizationService.java:**
```java
package com.example.dddspanner.application.security;

import com.example.dddspanner.application.exception.UnauthorizedException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

/**
 * 認可サービス
 */
@Service
@RequiredArgsConstructor
public class AuthorizationService {
    
    private final SecurityContextHolder securityContextHolder;
    
    /**
     * 権限チェック
     */
    public void checkPermission(String permission) {
        SecurityContext context = securityContextHolder.getContext();
        if (context == null || !context.hasPermission(permission)) {
            throw new UnauthorizedException("Insufficient permissions: " + permission);
        }
    }
    
    /**
     * ロールチェック
     */
    public void checkRole(String role) {
        SecurityContext context = securityContextHolder.getContext();
        if (context == null || !context.hasRole(role)) {
            throw new UnauthorizedException("Insufficient roles: " + role);
        }
    }
    
    /**
     * リソース所有者チェック
     */
    public void checkResourceOwner(String resourceUserId) {
        SecurityContext context = securityContextHolder.getContext();
        if (context == null || !context.getUserId().equals(resourceUserId)) {
            throw new UnauthorizedException("Access denied to resource");
        }
    }
}
```

### 5.6. アプリケーション層のベストプラクティス

#### 5.6.1. 設計原則

**1. 単一責任の原則**
- 各アプリケーションサービスは特定のユースケースに特化
- 複数のユースケースを混在させない

**2. 依存性注入の活用**
- 外部依存をインターフェースで抽象化
- テスト容易性の確保

**3. 不変性の活用**
- コマンド・クエリオブジェクトは不変
- 副作用を最小限に抑制

#### 5.6.2. パフォーマンス最適化

**1. 読み取り専用トランザクション**
```java
@Transactional(readOnly = true)
public List<UserDto> getUsers() {
    // 読み取り専用の処理
}
```

**2. バッチ処理**
```java
@Transactional
public void processBatch(List<CreateUserCommand> commands) {
    for (CreateUserCommand command : commands) {
        createUser(command);
    }
}
```

**3. キャッシュの活用**
```java
@Cacheable("users")
public UserDto getUser(UserId userId) {
    // キャッシュされた結果を返却
}
```

#### 5.6.3. テスト戦略

**1. ユニットテスト**
```java
@Test
void createUser_ValidCommand_ReturnsUserDto() {
    // Given
    CreateUserCommand command = new CreateUserCommand("John Doe", "john@example.com", "password");
    
    // When
    UserDto result = userApplicationService.createUser(command);
    
    // Then
    assertThat(result.getFullName()).isEqualTo("John Doe");
    assertThat(result.getEmail()).isEqualTo("john@example.com");
}
```

**2. 統合テスト**
```java
@SpringBootTest
class UserApplicationServiceIntegrationTest {
    
    @Autowired
    private UserApplicationService userApplicationService;
    
    @Test
    void createUser_WithValidData_SavesToDatabase() {
        // テスト実装
    }
}
```

この包括的なアプリケーション層の実装により、ビジネスロジックの調整とオーケストレーションを効果的に行い、保守性の高いシステムを構築できます。 