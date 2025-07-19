### 6.5. 非同期処理

#### 6.5.1. 非同期設定

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
 * - リソースの効率的利用
 * - スケーラビリティの向上
 */
@Configuration
@EnableAsync
public class AsyncConfig {
    
    /**
     * ユーザー処理用スレッドプール
     */
    @Bean(name = "userTaskExecutor")
    public Executor userTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("UserAsync-");
        executor.initialize();
        return executor;
    }
    
    /**
     * 注文処理用スレッドプール
     */
    @Bean(name = "orderTaskExecutor")
    public Executor orderTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("OrderAsync-");
        executor.initialize();
        return executor;
    }
    
    /**
     * 通知処理用スレッドプール
     */
    @Bean(name = "notificationTaskExecutor")
    public Executor notificationTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("NotificationAsync-");
        executor.initialize();
        return executor;
    }
}
```

#### 6.5.2. 非同期サービスの実装

**AsyncUserService.java:**
```java
package com.example.dddspanner.infrastructure.service;

import com.example.dddspanner.domain.entity.User;
import com.example.dddspanner.domain.event.DomainEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;

/**
 * 非同期ユーザーサービス
 * 
 * 非同期処理の特徴：
 * - ブロッキング処理の回避
 * - レスポンス時間の短縮
 * - エラーハンドリング
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AsyncUserService {
    
    private final EmailService emailService;
    private final NotificationService notificationService;
    private final AnalyticsService analyticsService;
    
    /**
     * ユーザー作成後の非同期処理
     */
    @Async("userTaskExecutor")
    public CompletableFuture<Void> processUserCreatedAsync(User user, List<DomainEvent> events) {
        try {
            log.info("Processing user created async for user: {}", user.getId());
            
            // 1. ウェルカムメールの送信
            emailService.sendWelcomeEmailAsync(user.getEmail().value());
            
            // 2. 通知の送信
            notificationService.sendUserCreatedNotificationAsync(user.getId().value());
            
            // 3. 分析データの送信
            analyticsService.trackUserRegistrationAsync(user);
            
            // 4. イベントの処理
            for (DomainEvent event : events) {
                processDomainEventAsync(event);
            }
            
            log.info("Completed async processing for user: {}", user.getId());
            return CompletableFuture.completedFuture(null);
            
        } catch (Exception e) {
            log.error("Failed to process user created async for user: {}", user.getId(), e);
            return CompletableFuture.failedFuture(e);
        }
    }
    
    /**
     * ユーザー更新後の非同期処理
     */
    @Async("userTaskExecutor")
    public CompletableFuture<Void> processUserUpdatedAsync(User user, List<DomainEvent> events) {
        try {
            log.info("Processing user updated async for user: {}", user.getId());
            
            // 1. 更新通知の送信
            notificationService.sendUserUpdatedNotificationAsync(user.getId().value());
            
            // 2. 分析データの更新
            analyticsService.trackUserUpdateAsync(user);
            
            // 3. イベントの処理
            for (DomainEvent event : events) {
                processDomainEventAsync(event);
            }
            
            log.info("Completed async processing for user update: {}", user.getId());
            return CompletableFuture.completedFuture(null);
            
        } catch (Exception e) {
            log.error("Failed to process user updated async for user: {}", user.getId(), e);
            return CompletableFuture.failedFuture(e);
        }
    }
    
    /**
     * ドメインイベントの非同期処理
     */
    @Async("userTaskExecutor")
    public CompletableFuture<Void> processDomainEventAsync(DomainEvent event) {
        try {
            log.debug("Processing domain event async: {}", event.getEventType());
            
            // イベントタイプに応じた処理
            switch (event.getEventType()) {
                case "UserCreated":
                    handleUserCreatedEventAsync((UserCreatedEvent) event);
                    break;
                case "UserEmailChanged":
                    handleUserEmailChangedEventAsync((UserEmailChangedEvent) event);
                    break;
                case "UserDeactivated":
                    handleUserDeactivatedEventAsync((UserDeactivatedEvent) event);
                    break;
                default:
                    log.warn("Unknown domain event type: {}", event.getEventType());
            }
            
            return CompletableFuture.completedFuture(null);
            
        } catch (Exception e) {
            log.error("Failed to process domain event async: {}", event.getEventType(), e);
            return CompletableFuture.failedFuture(e);
        }
    }
    
    /**
     * ユーザー作成イベントの非同期処理
     */
    private void handleUserCreatedEventAsync(UserCreatedEvent event) {
        // ウェルカムメールの送信
        emailService.sendWelcomeEmailAsync(event.email().value());
        
        // 分析データの送信
        analyticsService.trackUserRegistrationAsync(event);
    }
    
    /**
     * メールアドレス変更イベントの非同期処理
     */
    private void handleUserEmailChangedEventAsync(UserEmailChangedEvent event) {
        // メールアドレス変更確認メールの送信
        emailService.sendEmailChangeConfirmationAsync(event.email().value());
        
        // 分析データの更新
        analyticsService.trackEmailChangeAsync(event);
    }
    
    /**
     * ユーザー無効化イベントの非同期処理
     */
    private void handleUserDeactivatedEventAsync(UserDeactivatedEvent event) {
        // 無効化通知の送信
        notificationService.sendUserDeactivatedNotificationAsync(event.userId().value());
        
        // 分析データの更新
        analyticsService.trackUserDeactivationAsync(event);
    }
}
``` 