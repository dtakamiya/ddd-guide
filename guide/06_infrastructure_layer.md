## 6. インフラストラクチャ層の実装

インフラストラクチャ層は、外部システムとの連携とデータの永続化を担当します。この章では、Google Cloud SpannerとRabbitMQを使用した、クラウドネイティブなインフラストラクチャ層の実装について学びます。

### 6.1. インフラストラクチャ層の役割と設計原則

#### 6.1.1. インフラストラクチャ層の位置づけ

インフラストラクチャ層は、オニオンアーキテクチャの最外層として以下の役割を担います：

**1. データアクセス層**
*   リポジトリパターンの実装
*   データベースとの連携
*   クエリの最適化

**2. 外部システム連携**
*   メッセージングシステム
*   外部APIクライアント
*   ファイルシステム

**3. 技術的関心事の抽象化**
*   データベース固有の実装の隠蔽
*   外部システムの詳細の隠蔽
*   設定と環境の管理

#### 6.1.2. クラウドネイティブ設計の原則

**1. 疎結合**
*   外部システムへの直接依存を避ける
*   インターフェースによる抽象化
*   依存性注入の活用

**2. 耐障害性**
*   リトライ機能の実装
*   サーキットブレーカーパターン
*   フォールバック機能

**3. スケーラビリティ**
*   水平スケーリングへの対応
*   非同期処理の活用
*   キャッシュ戦略

### 6.2. Google Cloud Spannerとの連携

#### 6.2.1. Spring Data Spannerの設定

**SpannerConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import com.google.cloud.spring.data.spanner.core.SpannerTemplate;
import com.google.cloud.spring.data.spanner.core.mapping.SpannerMappingContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.spanner.core.convert.SpannerCustomConversions;
import org.springframework.data.spanner.core.convert.SpannerEntityProcessor;

/**
 * Google Cloud Spanner設定
 * 
 * クラウドネイティブな特徴：
 * - グローバル分散データベース
 * - 強整合性と高可用性
 * - 自動スケーリング
 */
@Configuration
public class SpannerConfig {
    
    @Bean
    public SpannerCustomConversions spannerCustomConversions() {
        return new SpannerCustomConversions(Arrays.asList(
            new MoneyConverter(),
            new EmailConverter(),
            new FullNameConverter()
        ));
    }
    
    @Bean
    public SpannerEntityProcessor spannerEntityProcessor(
            SpannerMappingContext mappingContext,
            SpannerCustomConversions conversions) {
        return new SpannerEntityProcessor(mappingContext, conversions);
    }
}
```

#### 6.2.2. カスタムコンバーターの実装

**MoneyConverter.java:**
```java
package com.example.dddspanner.infrastructure.converter;

import com.example.dddspanner.domain.valueobject.Money;
import com.google.cloud.spanner.Value;
import org.springframework.core.convert.converter.Converter;
import org.springframework.data.convert.ReadingConverter;
import org.springframework.data.convert.WritingConverter;

import java.math.BigDecimal;
import java.util.Currency;

/**
 * Money値オブジェクトのSpanner変換器
 */
@WritingConverter
public class MoneyConverter implements Converter<Money, String> {
    
    @Override
    public String convert(Money source) {
        if (source == null) return null;
        return source.amount() + ":" + source.currency().getCurrencyCode();
    }
}

@ReadingConverter
public class MoneyReadConverter implements Converter<String, Money> {
    
    @Override
    public Money convert(String source) {
        if (source == null || source.isEmpty()) return null;
        
        String[] parts = source.split(":");
        if (parts.length != 2) {
            throw new IllegalArgumentException("Invalid money format: " + source);
        }
        
        BigDecimal amount = new BigDecimal(parts[0]);
        Currency currency = Currency.getInstance(parts[1]);
        
        return new Money(amount, currency);
    }
}
```

#### 6.2.3. リポジトリの実装

**UserRepositoryImpl.java:**
```java
package com.example.dddspanner.infrastructure.repository;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.repository.UserRepository;
import com.example.dddspanner.domain.valueobject.*;
import com.example.dddspanner.infrastructure.entity.UserEntity;
import com.google.cloud.spring.data.spanner.core.SpannerTemplate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

/**
 * ユーザーリポジトリのSpanner実装
 * 
 * 実装の特徴：
 * - ドメインオブジェクトと永続化オブジェクトの分離
 * - クエリの最適化
 * - エラーハンドリング
 */
@Slf4j
@Repository
@RequiredArgsConstructor
public class UserRepositoryImpl implements UserRepository {
    
    private final SpannerTemplate spannerTemplate;
    private final UserEntityMapper userEntityMapper;
    
    @Override
    public User save(User user) {
        try {
            UserEntity entity = userEntityMapper.toEntity(user);
            UserEntity savedEntity = spannerTemplate.save(entity);
            return userEntityMapper.toDomain(savedEntity);
        } catch (Exception e) {
            log.error("Failed to save user: {}", user.getId(), e);
            throw new UserPersistenceException("Failed to save user", e);
        }
    }
    
    @Override
    public Optional<User> findById(UserId id) {
        try {
            String sql = "SELECT * FROM users WHERE user_id = @userId";
            UserEntity entity = spannerTemplate.queryForObject(
                sql, 
                Map.of("userId", id.value()), 
                UserEntity.class
            );
            
            return Optional.ofNullable(entity)
                .map(userEntityMapper::toDomain);
        } catch (Exception e) {
            log.error("Failed to find user by id: {}", id, e);
            throw new UserQueryException("Failed to find user", e);
        }
    }
    
    @Override
    public Optional<User> findByEmail(Email email) {
        try {
            String sql = "SELECT * FROM users WHERE email = @email LIMIT 1";
            UserEntity entity = spannerTemplate.queryForObject(
                sql, 
                Map.of("email", email.value()), 
                UserEntity.class
            );
            
            return Optional.ofNullable(entity)
                .map(userEntityMapper::toDomain);
        } catch (Exception e) {
            log.error("Failed to find user by email: {}", email, e);
            throw new UserQueryException("Failed to find user by email", e);
        }
    }
    
    @Override
    public List<User> findAll() {
        try {
            String sql = "SELECT * FROM users ORDER BY created_at DESC";
            List<UserEntity> entities = spannerTemplate.query(
                sql, 
                UserEntity.class
            );
            
            return entities.stream()
                .map(userEntityMapper::toDomain)
                .toList();
        } catch (Exception e) {
            log.error("Failed to find all users", e);
            throw new UserQueryException("Failed to find all users", e);
        }
    }
    
    @Override
    public List<User> findActiveUsers() {
        try {
            String sql = "SELECT * FROM users WHERE status = @status ORDER BY created_at DESC";
            List<UserEntity> entities = spannerTemplate.query(
                sql, 
                Map.of("status", "ACTIVE"), 
                UserEntity.class
            );
            
            return entities.stream()
                .map(userEntityMapper::toDomain)
                .toList();
        } catch (Exception e) {
            log.error("Failed to find active users", e);
            throw new UserQueryException("Failed to find active users", e);
        }
    }
    
    @Override
    public void delete(UserId id) {
        try {
            String sql = "DELETE FROM users WHERE user_id = @userId";
            spannerTemplate.executeDmlStatement(sql, Map.of("userId", id.value()));
        } catch (Exception e) {
            log.error("Failed to delete user: {}", id, e);
            throw new UserPersistenceException("Failed to delete user", e);
        }
    }
    
    @Override
    public boolean existsById(UserId id) {
        try {
            String sql = "SELECT COUNT(*) FROM users WHERE user_id = @userId";
            Long count = spannerTemplate.queryForObject(sql, Map.of("userId", id.value()), Long.class);
            return count != null && count > 0;
        } catch (Exception e) {
            log.error("Failed to check user existence: {}", id, e);
            throw new UserQueryException("Failed to check user existence", e);
        }
    }
    
    @Override
    public boolean existsByEmail(Email email) {
        try {
            String sql = "SELECT COUNT(*) FROM users WHERE email = @email";
            Long count = spannerTemplate.queryForObject(sql, Map.of("email", email.value()), Long.class);
            return count != null && count > 0;
        } catch (Exception e) {
            log.error("Failed to check email existence: {}", email, e);
            throw new UserQueryException("Failed to check email existence", e);
        }
    }
}
```

#### 6.2.4. エンティティマッパーの実装

**UserEntityMapper.java:**
```java
package com.example.dddspanner.infrastructure.mapper;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.valueobject.*;
import com.example.dddspanner.infrastructure.entity.UserEntity;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Named;

/**
 * ユーザーエンティティマッパー
 * 
 * ドメインオブジェクトと永続化オブジェクトの変換
 */
@Mapper(componentModel = "spring")
public interface UserEntityMapper {
    
    @Mapping(source = "id.value", target = "userId")
    @Mapping(source = "fullName.firstName", target = "firstName")
    @Mapping(source = "fullName.lastName", target = "lastName")
    @Mapping(source = "email.value", target = "email")
    @Mapping(source = "status", target = "status")
    UserEntity toEntity(User user);
    
    @Mapping(source = "userId", target = "id", qualifiedByName = "stringToUserId")
    @Mapping(source = "firstName", target = "fullName", qualifiedByName = "toFullName")
    @Mapping(source = "email", target = "email", qualifiedByName = "stringToEmail")
    @Mapping(source = "status", target = "status", qualifiedByName = "stringToUserStatus")
    User toDomain(UserEntity entity);
    
    @Named("stringToUserId")
    default UserId stringToUserId(String userId) {
        return userId != null ? UserId.of(userId) : null;
    }
    
    @Named("toFullName")
    default FullName toFullName(String firstName, String lastName) {
        return new FullName(firstName, lastName);
    }
    
    @Named("stringToEmail")
    default Email stringToEmail(String email) {
        return email != null ? new Email(email) : null;
    }
    
    @Named("stringToUserStatus")
    default UserStatus stringToUserStatus(String status) {
        return status != null ? UserStatus.valueOf(status) : null;
    }
}
```

### 6.3. RabbitMQとの連携

#### 6.3.1. RabbitMQ設定

**RabbitMQConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * RabbitMQ設定
 * 
 * メッセージングの特徴：
 * - 非同期処理
 * - 疎結合なシステム間通信
 * - 信頼性の高いメッセージ配信
 */
@Configuration
public class RabbitMQConfig {
    
    // エクスチェンジの定義
    public static final String USER_EVENTS_EXCHANGE = "user.events";
    public static final String ORDER_EVENTS_EXCHANGE = "order.events";
    
    // キューの定義
    public static final String USER_CREATED_QUEUE = "user.created";
    public static final String USER_UPDATED_QUEUE = "user.updated";
    public static final String ORDER_CONFIRMED_QUEUE = "order.confirmed";
    public static final String ORDER_CANCELLED_QUEUE = "order.cancelled";
    
    // デッドレターキュー
    public static final String DEAD_LETTER_QUEUE = "dead.letter";
    public static final String DEAD_LETTER_EXCHANGE = "dead.letter.exchange";
    
    @Bean
    public Jackson2JsonMessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }
    
    // ユーザーイベントエクスチェンジ
    @Bean
    public TopicExchange userEventsExchange() {
        return new TopicExchange(USER_EVENTS_EXCHANGE);
    }
    
    // 注文イベントエクスチェンジ
    @Bean
    public TopicExchange orderEventsExchange() {
        return new TopicExchange(ORDER_EVENTS_EXCHANGE);
    }
    
    // デッドレターエクスチェンジ
    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(DEAD_LETTER_EXCHANGE);
    }
    
    // ユーザー作成キュー
    @Bean
    public Queue userCreatedQueue() {
        return QueueBuilder.durable(USER_CREATED_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", "user.created.dlq")
            .build();
    }
    
    // ユーザー更新キュー
    @Bean
    public Queue userUpdatedQueue() {
        return QueueBuilder.durable(USER_UPDATED_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", "user.updated.dlq")
            .build();
    }
    
    // 注文確定キュー
    @Bean
    public Queue orderConfirmedQueue() {
        return QueueBuilder.durable(ORDER_CONFIRMED_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", "order.confirmed.dlq")
            .build();
    }
    
    // デッドレターキュー
    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DEAD_LETTER_QUEUE).build();
    }
    
    // バインディング
    @Bean
    public Binding userCreatedBinding() {
        return BindingBuilder.bind(userCreatedQueue())
            .to(userEventsExchange())
            .with("user.created");
    }
    
    @Bean
    public Binding userUpdatedBinding() {
        return BindingBuilder.bind(userUpdatedQueue())
            .to(userEventsExchange())
            .with("user.updated");
    }
    
    @Bean
    public Binding orderConfirmedBinding() {
        return BindingBuilder.bind(orderConfirmedQueue())
            .to(orderEventsExchange())
            .with("order.confirmed");
    }
    
    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
            .to(deadLetterExchange())
            .with("dead.letter");
    }
}
```

#### 6.3.2. メッセージプロデューサーの実装

**UserEventPublisher.java:**
```java
package com.example.dddspanner.infrastructure.messaging;

import com.example.dddspanner.domain.event.DomainEvent;
import com.example.dddspanner.domain.event.UserCreatedEvent;
import com.example.dddspanner.domain.event.UserUpdatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

/**
 * ユーザーイベントパブリッシャー
 * 
 * メッセージングの特徴：
 * - 非同期処理
 * - 疎結合なシステム間通信
 * - 信頼性の高いメッセージ配信
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class UserEventPublisher {
    
    private final RabbitTemplate rabbitTemplate;
    
    /**
     * ユーザー作成イベントの送信
     */
    public void publishUserCreated(UserCreatedEvent event) {
        try {
            String routingKey = "user.created";
            rabbitTemplate.convertAndSend(
                RabbitMQConfig.USER_EVENTS_EXCHANGE,
                routingKey,
                event
            );
            log.info("User created event published: {}", event.userId());
        } catch (Exception e) {
            log.error("Failed to publish user created event: {}", event.userId(), e);
            throw new MessagePublishException("Failed to publish user created event", e);
        }
    }
    
    /**
     * ユーザー更新イベントの送信
     */
    public void publishUserUpdated(UserUpdatedEvent event) {
        try {
            String routingKey = "user.updated";
            rabbitTemplate.convertAndSend(
                RabbitMQConfig.USER_EVENTS_EXCHANGE,
                routingKey,
                event
            );
            log.info("User updated event published: {}", event.userId());
        } catch (Exception e) {
            log.error("Failed to publish user updated event: {}", event.userId(), e);
            throw new MessagePublishException("Failed to publish user updated event", e);
        }
    }
    
    /**
     * ドメインイベントの送信
     */
    public void publishDomainEvent(DomainEvent event) {
        if (event instanceof UserCreatedEvent userCreatedEvent) {
            publishUserCreated(userCreatedEvent);
        } else if (event instanceof UserUpdatedEvent userUpdatedEvent) {
            publishUserUpdated(userUpdatedEvent);
        } else {
            log.warn("Unknown domain event type: {}", event.getClass().getSimpleName());
        }
    }
}
```

#### 6.3.3. メッセージコンシューマーの実装

**UserEventConsumer.java:**
```java
package com.example.dddspanner.infrastructure.messaging;

import com.example.dddspanner.domain.event.UserCreatedEvent;
import com.example.dddspanner.domain.event.UserUpdatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * ユーザーイベントコンシューマー
 * 
 * イベント駆動アーキテクチャの特徴：
 * - 非同期処理
 * - 疎結合なシステム間通信
 * - スケーラブルな設計
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class UserEventConsumer {
    
    private final UserNotificationService userNotificationService;
    private final UserAuditService userAuditService;
    
    /**
     * ユーザー作成イベントの処理
     */
    @RabbitListener(queues = RabbitMQConfig.USER_CREATED_QUEUE)
    public void handleUserCreated(UserCreatedEvent event) {
        try {
            log.info("Processing user created event: {}", event.userId());
            
            // 通知サービスの呼び出し
            userNotificationService.sendWelcomeEmail(event.userId(), event.email());
            
            // 監査サービスの呼び出し
            userAuditService.recordUserCreation(event.userId(), event.occurredAt());
            
            log.info("User created event processed successfully: {}", event.userId());
        } catch (Exception e) {
            log.error("Failed to process user created event: {}", event.userId(), e);
            throw new MessageProcessingException("Failed to process user created event", e);
        }
    }
    
    /**
     * ユーザー更新イベントの処理
     */
    @RabbitListener(queues = RabbitMQConfig.USER_UPDATED_QUEUE)
    public void handleUserUpdated(UserUpdatedEvent event) {
        try {
            log.info("Processing user updated event: {}", event.userId());
            
            // 監査サービスの呼び出し
            userAuditService.recordUserUpdate(event.userId(), event.occurredAt());
            
            log.info("User updated event processed successfully: {}", event.userId());
        } catch (Exception e) {
            log.error("Failed to process user updated event: {}", event.userId(), e);
            throw new MessageProcessingException("Failed to process user updated event", e);
        }
    }
}
```

### 6.4. 非同期処理の実装

#### 6.4.1. 非同期設定

**AsyncConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

/**
 * 非同期処理設定
 * 
 * 非同期処理の特徴：
 * - レスポンス時間の短縮
 * - リソースの効率的な利用
 * - スケーラビリティの向上
 */
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("AsyncTask-");
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "eventExecutor")
    public Executor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("EventTask-");
        executor.initialize();
        return executor;
    }
}
```

#### 6.4.2. 非同期サービスの実装

**UserNotificationService.java:**
```java
package com.example.dddspanner.infrastructure.service;

import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.valueobject.UserId;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.Instant;

/**
 * ユーザー通知サービス
 * 
 * 非同期処理の特徴：
 * - レスポンス時間の短縮
 * - エラー処理の分離
 * - リトライ機能
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class UserNotificationService {
    
    private final EmailService emailService;
    private final NotificationTemplateService templateService;
    
    /**
     * ウェルカムメールの送信（非同期）
     */
    @Async("taskExecutor")
    public void sendWelcomeEmail(UserId userId, Email email) {
        try {
            log.info("Sending welcome email to user: {}", userId);
            
            String template = templateService.getWelcomeEmailTemplate();
            String subject = "Welcome to Our Service!";
            
            emailService.sendEmail(email, subject, template);
            
            log.info("Welcome email sent successfully to user: {}", userId);
        } catch (Exception e) {
            log.error("Failed to send welcome email to user: {}", userId, e);
            // 非同期処理では例外を再スローしない
            // 代わりにログに記録し、必要に応じて監視システムに通知
        }
    }
    
    /**
     * パスワードリセットメールの送信（非同期）
     */
    @Async("taskExecutor")
    public void sendPasswordResetEmail(UserId userId, Email email, String resetToken) {
        try {
            log.info("Sending password reset email to user: {}", userId);
            
            String template = templateService.getPasswordResetTemplate(resetToken);
            String subject = "Password Reset Request";
            
            emailService.sendEmail(email, subject, template);
            
            log.info("Password reset email sent successfully to user: {}", userId);
        } catch (Exception e) {
            log.error("Failed to send password reset email to user: {}", userId, e);
        }
    }
}
```

### 6.5. キャッシュの実装

#### 6.5.1. キャッシュ設定

**CacheConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * キャッシュ設定
 * 
 * キャッシュの特徴：
 * - レスポンス時間の短縮
 * - データベース負荷の軽減
 * - スケーラビリティの向上
 */
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
        cacheManager.setCacheNames(Arrays.asList(
            "users",
            "orders",
            "products"
        ));
        return cacheManager;
    }
}
```

#### 6.5.2. キャッシュ付きリポジトリの実装

**CachedUserRepository.java:**
```java
package com.example.dddspanner.infrastructure.repository;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.repository.UserRepository;
import com.example.dddspanner.domain.valueobject.Email;
import com.example.dddspanner.domain.valueobject.UserId;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

/**
 * キャッシュ付きユーザーリポジトリ
 * 
 * キャッシュ戦略：
 * - 読み取り専用データのキャッシュ
 * - 更新時のキャッシュ無効化
 * - TTL（Time To Live）の設定
 */
@Repository
@RequiredArgsConstructor
public class CachedUserRepository implements UserRepository {
    
    private final UserRepositoryImpl userRepository;
    
    @Override
    @CacheEvict(value = "users", allEntries = true)
    public User save(User user) {
        return userRepository.save(user);
    }
    
    @Override
    @Cacheable(value = "users", key = "#id.value")
    public Optional<User> findById(UserId id) {
        return userRepository.findById(id);
    }
    
    @Override
    @Cacheable(value = "users", key = "'email:' + #email.value")
    public Optional<User> findByEmail(Email email) {
        return userRepository.findByEmail(email);
    }
    
    @Override
    @Cacheable(value = "users", key = "'all'")
    public List<User> findAll() {
        return userRepository.findAll();
    }
    
    @Override
    @Cacheable(value = "users", key = "'active'")
    public List<User> findActiveUsers() {
        return userRepository.findActiveUsers();
    }
    
    @Override
    @CacheEvict(value = "users", allEntries = true)
    public void delete(UserId id) {
        userRepository.delete(id);
    }
    
    @Override
    public boolean existsById(UserId id) {
        return userRepository.existsById(id);
    }
    
    @Override
    public boolean existsByEmail(Email email) {
        return userRepository.existsByEmail(email);
    }
}
```

### 6.6. 監視とログの実装

#### 6.6.1. メトリクス収集

**MetricsConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import io.micrometer.core.aop.TimedAspect;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * メトリクス設定
 * 
 * 監視の特徴：
 * - パフォーマンスの可視化
 * - 異常検知
 * - 容量計画
 */
@Configuration
public class MetricsConfig {
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

#### 6.6.2. カスタムメトリクス

**UserMetrics.java:**
```java
package com.example.dddspanner.infrastructure.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

/**
 * ユーザー関連メトリクス
 */
@Component
@RequiredArgsConstructor
public class UserMetrics {
    
    private final MeterRegistry meterRegistry;
    
    private final Counter userCreatedCounter;
    private final Counter userUpdatedCounter;
    private final Counter userDeletedCounter;
    
    public UserMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.userCreatedCounter = Counter.builder("user.created")
            .description("Number of users created")
            .register(meterRegistry);
        this.userUpdatedCounter = Counter.builder("user.updated")
            .description("Number of users updated")
            .register(meterRegistry);
        this.userDeletedCounter = Counter.builder("user.deleted")
            .description("Number of users deleted")
            .register(meterRegistry);
    }
    
    public void incrementUserCreated() {
        userCreatedCounter.increment();
    }
    
    public void incrementUserUpdated() {
        userUpdatedCounter.increment();
    }
    
    public void incrementUserDeleted() {
        userDeletedCounter.increment();
    }
}
```

### 6.7. セキュリティの実装

#### 6.7.1. 暗号化サービス

**EncryptionService.java:**
```java
package com.example.dddspanner.infrastructure.security;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;

/**
 * 暗号化サービス
 * 
 * セキュリティの特徴：
 * - データの機密性保護
 * - 転送時の暗号化
 * - 保存時の暗号化
 */
@Service
public class EncryptionService {
    
    @Value("${app.encryption.key}")
    private String encryptionKey;
    
    private static final String ALGORITHM = "AES";
    
    /**
     * 文字列の暗号化
     */
    public String encrypt(String data) {
        try {
            SecretKeySpec key = new SecretKeySpec(encryptionKey.getBytes(), ALGORITHM);
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, key);
            
            byte[] encryptedBytes = cipher.doFinal(data.getBytes());
            return Base64.getEncoder().encodeToString(encryptedBytes);
        } catch (Exception e) {
            throw new EncryptionException("Failed to encrypt data", e);
        }
    }
    
    /**
     * 文字列の復号化
     */
    public String decrypt(String encryptedData) {
        try {
            SecretKeySpec key = new SecretKeySpec(encryptionKey.getBytes(), ALGORITHM);
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, key);
            
            byte[] decryptedBytes = cipher.doFinal(Base64.getDecoder().decode(encryptedData));
            return new String(decryptedBytes);
        } catch (Exception e) {
            throw new EncryptionException("Failed to decrypt data", e);
        }
    }
}
```

### 6.8. インフラストラクチャ層のテスト

#### 6.8.1. リポジトリのテスト

**UserRepositoryTest.groovy:**
```groovy
package com.example.dddspanner.infrastructure.repository

import com.example.dddspanner.domain.entity.User
import com.example.dddspanner.domain.valueobject.Email
import com.example.dddspanner.domain.valueobject.FullName
import com.example.dddspanner.domain.valueobject.UserId
import com.example.dddspanner.infrastructure.entity.UserEntity
import com.example.dddspanner.infrastructure.mapper.UserEntityMapper
import com.google.cloud.spring.data.spanner.core.SpannerTemplate
import spock.lang.Specification

class UserRepositoryTest extends Specification {
    
    SpannerTemplate spannerTemplate = Mock()
    UserEntityMapper userEntityMapper = Mock()
    
    UserRepositoryImpl repository
    
    def setup() {
        repository = new UserRepositoryImpl(spannerTemplate, userEntityMapper)
    }
    
    def "should save user successfully"() {
        given:
        def user = User.create(
            new FullName("Taro", "Yamada"),
            new Email("taro.yamada@example.com")
        )
        def entity = Mock(UserEntity)
        def savedEntity = Mock(UserEntity)
        def savedUser = Mock(User)
        
        when:
        def result = repository.save(user)
        
        then:
        1 * userEntityMapper.toEntity(user) >> entity
        1 * spannerTemplate.save(entity) >> savedEntity
        1 * userEntityMapper.toDomain(savedEntity) >> savedUser
        
        and:
        result == savedUser
    }
    
    def "should find user by id successfully"() {
        given:
        def userId = UserId.generate()
        def entity = Mock(UserEntity)
        def user = Mock(User)
        
        and:
        spannerTemplate.queryForObject(_, _, UserEntity.class) >> entity
        userEntityMapper.toDomain(entity) >> user
        
        when:
        def result = repository.findById(userId)
        
        then:
        result.isPresent()
        result.get() == user
    }
}
```

### 6.9. まとめ

この章では、インフラストラクチャ層の実装について詳しく説明しました。重要なポイントをまとめます：

**1. クラウドネイティブ設計**
*   Google Cloud Spannerとの連携
*   RabbitMQによるメッセージング
*   非同期処理の活用

**2. データアクセス層**
*   リポジトリパターンの実装
*   ドメインオブジェクトと永続化オブジェクトの分離
*   クエリの最適化

**3. メッセージング**
*   イベント駆動アーキテクチャ
*   非同期処理による疎結合
*   信頼性の高いメッセージ配信

**4. パフォーマンス最適化**
*   キャッシュ戦略の実装
*   非同期処理の活用
*   メトリクス収集

**5. セキュリティ**
*   データの暗号化
*   アクセス制御
*   監査ログ

次の章では、プレゼンテーション層を実装して、REST APIとセキュリティを実現します。 