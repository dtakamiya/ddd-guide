### 6.4. RabbitMQメッセージング

#### 6.4.1. RabbitMQ設定

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
 * - 疎結合
 * - 耐障害性
 */
@Configuration
public class RabbitMQConfig {
    
    public static final String USER_EVENTS_QUEUE = "user.events";
    public static final String ORDER_EVENTS_QUEUE = "order.events";
    public static final String USER_EVENTS_EXCHANGE = "user.events.exchange";
    public static final String ORDER_EVENTS_EXCHANGE = "order.events.exchange";
    public static final String DEAD_LETTER_QUEUE = "dead.letter.queue";
    public static final String DEAD_LETTER_EXCHANGE = "dead.letter.exchange";
    
    /**
     * ユーザーイベントキュー
     */
    @Bean
    public Queue userEventsQueue() {
        return QueueBuilder.durable(USER_EVENTS_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", "dead.letter")
            .build();
    }
    
    /**
     * 注文イベントキュー
     */
    @Bean
    public Queue orderEventsQueue() {
        return QueueBuilder.durable(ORDER_EVENTS_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", "dead.letter")
            .build();
    }
    
    /**
     * デッドレターキュー
     */
    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DEAD_LETTER_QUEUE).build();
    }
    
    /**
     * ユーザーイベントエクスチェンジ
     */
    @Bean
    public TopicExchange userEventsExchange() {
        return new TopicExchange(USER_EVENTS_EXCHANGE);
    }
    
    /**
     * 注文イベントエクスチェンジ
     */
    @Bean
    public TopicExchange orderEventsExchange() {
        return new TopicExchange(ORDER_EVENTS_EXCHANGE);
    }
    
    /**
     * デッドレターエクスチェンジ
     */
    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(DEAD_LETTER_EXCHANGE);
    }
    
    /**
     * バインディング設定
     */
    @Bean
    public Binding userEventsBinding() {
        return BindingBuilder.bind(userEventsQueue())
            .to(userEventsExchange())
            .with("user.*");
    }
    
    @Bean
    public Binding orderEventsBinding() {
        return BindingBuilder.bind(orderEventsQueue())
            .to(orderEventsExchange())
            .with("order.*");
    }
    
    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
            .to(deadLetterExchange())
            .with("dead.letter");
    }
    
    /**
     * JSONメッセージコンバーター
     */
    @Bean
    public Jackson2JsonMessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    /**
     * RabbitTemplate設定
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }
}
```

#### 6.4.2. メッセージプロデューサー

**UserEventPublisher.java:**
```java
package com.example.dddspanner.infrastructure.messaging;

import com.example.dddspanner.domain.event.DomainEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

/**
 * ユーザーイベントパブリッシャー
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class UserEventPublisher {
    
    private final RabbitTemplate rabbitTemplate;
    
    /**
     * ユーザーイベントを発行
     */
    public void publishUserEvent(DomainEvent event) {
        try {
            String routingKey = "user." + event.getEventType().toLowerCase();
            rabbitTemplate.convertAndSend(
                RabbitMQConfig.USER_EVENTS_EXCHANGE,
                routingKey,
                event
            );
            log.info("Published user event: {} with routing key: {}", event.getEventType(), routingKey);
        } catch (Exception e) {
            log.error("Failed to publish user event: {}", event.getEventType(), e);
            throw new MessagePublishException("Failed to publish user event", e);
        }
    }
}
```

#### 6.4.3. メッセージコンシューマー

**UserEventConsumer.java:**
```java
package com.example.dddspanner.infrastructure.messaging;

import com.example.dddspanner.domain.event.UserCreatedEvent;
import com.example.dddspanner.domain.event.UserEmailChangedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * ユーザーイベントコンシューマー
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class UserEventConsumer {
    
    private final EmailService emailService;
    private final NotificationService notificationService;
    
    /**
     * ユーザー作成イベントの処理
     */
    @RabbitListener(queues = RabbitMQConfig.USER_EVENTS_QUEUE)
    public void handleUserEvent(DomainEvent event) {
        try {
            log.info("Received user event: {}", event.getEventType());
            
            switch (event.getEventType()) {
                case "UserCreated":
                    handleUserCreated((UserCreatedEvent) event);
                    break;
                case "UserEmailChanged":
                    handleUserEmailChanged((UserEmailChangedEvent) event);
                    break;
                default:
                    log.warn("Unknown user event type: {}", event.getEventType());
            }
        } catch (Exception e) {
            log.error("Failed to handle user event: {}", event.getEventType(), e);
            throw new MessageProcessingException("Failed to handle user event", e);
        }
    }
    
    /**
     * ユーザー作成イベントの処理
     */
    private void handleUserCreated(UserCreatedEvent event) {
        log.info("Processing user created event for user: {}", event.userId());
        
        // ウェルカムメールの送信
        emailService.sendWelcomeEmail(event.email().value());
        
        // 通知の送信
        notificationService.sendUserCreatedNotification(event.userId().value());
    }
    
    /**
     * メールアドレス変更イベントの処理
     */
    private void handleUserEmailChanged(UserEmailChangedEvent event) {
        log.info("Processing user email changed event for user: {}", event.userId());
        
        // メールアドレス変更確認メールの送信
        emailService.sendEmailChangeConfirmation(event.email().value());
    }
}
``` 