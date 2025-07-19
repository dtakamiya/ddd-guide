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