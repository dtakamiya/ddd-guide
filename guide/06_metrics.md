### 6.7. メトリクスとモニタリング

#### 6.7.1. メトリクス設定

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
 * モニタリングの特徴：
 * - パフォーマンス測定
 * - エラー率の監視
 * - ビジネスメトリクスの収集
 */
@Configuration
public class MetricsConfig {
    
    /**
     * タイマーアスペクト
     */
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

#### 6.7.2. カスタムメトリクスの実装

**UserMetricsService.java:**
```java
package com.example.dddspanner.infrastructure.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * ユーザー関連メトリクスサービス
 * 
 * メトリクスの特徴：
 * - ビジネス指標の測定
 * - パフォーマンス監視
 * - エラー率の追跡
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class UserMetricsService {
    
    private final MeterRegistry meterRegistry;
    
    // カウンター
    private final Counter userRegistrationCounter;
    private final Counter userUpdateCounter;
    private final Counter userDeletionCounter;
    private final Counter userLoginCounter;
    
    // タイマー
    private final Timer userRegistrationTimer;
    private final Timer userQueryTimer;
    private final Timer userUpdateTimer;
    
    public UserMetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // カウンターの初期化
        this.userRegistrationCounter = Counter.builder("user.registration.total")
            .description("Total number of user registrations")
            .register(meterRegistry);
        
        this.userUpdateCounter = Counter.builder("user.update.total")
            .description("Total number of user updates")
            .register(meterRegistry);
        
        this.userDeletionCounter = Counter.builder("user.deletion.total")
            .description("Total number of user deletions")
            .register(meterRegistry);
        
        this.userLoginCounter = Counter.builder("user.login.total")
            .description("Total number of user logins")
            .register(meterRegistry);
        
        // タイマーの初期化
        this.userRegistrationTimer = Timer.builder("user.registration.duration")
            .description("User registration duration")
            .register(meterRegistry);
        
        this.userQueryTimer = Timer.builder("user.query.duration")
            .description("User query duration")
            .register(meterRegistry);
        
        this.userUpdateTimer = Timer.builder("user.update.duration")
            .description("User update duration")
            .register(meterRegistry);
    }
    
    /**
     * ユーザー登録メトリクスの記録
     */
    public void recordUserRegistration(boolean success, long durationMs) {
        if (success) {
            userRegistrationCounter.increment();
            userRegistrationTimer.record(durationMs, TimeUnit.MILLISECONDS);
            log.debug("Recorded successful user registration");
        } else {
            meterRegistry.counter("user.registration.failed").increment();
            log.debug("Recorded failed user registration");
        }
    }
    
    /**
     * ユーザー更新メトリクスの記録
     */
    public void recordUserUpdate(boolean success, long durationMs) {
        if (success) {
            userUpdateCounter.increment();
            userUpdateTimer.record(durationMs, TimeUnit.MILLISECONDS);
            log.debug("Recorded successful user update");
        } else {
            meterRegistry.counter("user.update.failed").increment();
            log.debug("Recorded failed user update");
        }
    }
    
    /**
     * ユーザー削除メトリクスの記録
     */
    public void recordUserDeletion(boolean success) {
        if (success) {
            userDeletionCounter.increment();
            log.debug("Recorded successful user deletion");
        } else {
            meterRegistry.counter("user.deletion.failed").increment();
            log.debug("Recorded failed user deletion");
        }
    }
    
    /**
     * ユーザーログインメトリクスの記録
     */
    public void recordUserLogin(boolean success) {
        if (success) {
            userLoginCounter.increment();
            log.debug("Recorded successful user login");
        } else {
            meterRegistry.counter("user.login.failed").increment();
            log.debug("Recorded failed user login");
        }
    }
    
    /**
     * ユーザークエリメトリクスの記録
     */
    public void recordUserQuery(String queryType, long durationMs) {
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(Timer.builder("user.query.duration")
            .tag("query.type", queryType)
            .register(meterRegistry));
        
        log.debug("Recorded user query: {} in {}ms", queryType, durationMs);
    }
    
    /**
     * アクティブユーザー数の記録
     */
    public void recordActiveUserCount(long count) {
        meterRegistry.gauge("user.active.count", count);
        log.debug("Recorded active user count: {}", count);
    }
    
    /**
     * ユーザー登録率の記録
     */
    public void recordUserRegistrationRate(double rate) {
        meterRegistry.gauge("user.registration.rate", rate);
        log.debug("Recorded user registration rate: {}", rate);
    }
}
```

#### 6.7.3. ヘルスチェックの実装

**HealthCheckService.java:**
```java
package com.example.dddspanner.infrastructure.health;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.sql.Connection;
import java.sql.DriverManager;

/**
 * ヘルスチェックサービス
 * 
 * ヘルスチェックの特徴：
 * - システム状態の監視
 * - 依存関係の確認
 * - 障害の早期発見
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class HealthCheckService implements HealthIndicator {
    
    private final SpannerTemplate spannerTemplate;
    
    @Override
    public Health health() {
        try {
            // Spanner接続の確認
            String sql = "SELECT 1";
            spannerTemplate.queryForObject(sql, Map.of(), Integer.class);
            
            return Health.up()
                .withDetail("database", "Spanner")
                .withDetail("status", "Connected")
                .build();
                
        } catch (Exception e) {
            log.error("Health check failed", e);
            return Health.down()
                .withDetail("database", "Spanner")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
``` 