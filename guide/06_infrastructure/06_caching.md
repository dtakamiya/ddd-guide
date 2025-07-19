### 6.6. キャッシュ戦略

#### 6.6.1. キャッシュ設定

**CacheConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

/**
 * キャッシュ設定
 * 
 * キャッシュ戦略の特徴：
 * - レスポンス時間の短縮
 * - データベース負荷の軽減
 * - スケーラビリティの向上
 */
@Configuration
@EnableCaching
public class CacheConfig {
    
    /**
     * ユーザーキャッシュ設定
     */
    @Bean
    public RedisCacheConfiguration userCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
    
    /**
     * 注文キャッシュ設定
     */
    @Bean
    public RedisCacheConfiguration orderCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(15))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
    
    /**
     * 商品キャッシュ設定
     */
    @Bean
    public RedisCacheConfiguration productCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
    
    /**
     * キャッシュマネージャー
     */
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(userCacheConfiguration())
            .withCacheConfiguration("users", userCacheConfiguration())
            .withCacheConfiguration("orders", orderCacheConfiguration())
            .withCacheConfiguration("products", productCacheConfiguration())
            .build();
    }
}
```

#### 6.6.2. キャッシュサービスの実装

**UserCacheService.java:**
```java
package com.example.dddspanner.infrastructure.cache;

import com.example.dddspanner.application.dto.UserResponse;
import com.example.dddspanner.domain.valueobject.UserId;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

/**
 * ユーザーキャッシュサービス
 * 
 * キャッシュ戦略の特徴：
 * - 読み取り専用データのキャッシュ
 * - 書き込み時のキャッシュ無効化
 * - キャッシュキーの設計
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class UserCacheService {
    
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    
    /**
     * ユーザー情報のキャッシュ取得
     */
    @Cacheable(value = "users", key = "#userId")
    public Optional<UserResponse> getUserFromCache(String userId) {
        log.debug("Cache miss for user: {}", userId);
        
        UserId id = UserId.of(userId);
        return userRepository.findById(id)
            .map(userMapper::toResponse);
    }
    
    /**
     * ユーザー一覧のキャッシュ取得
     */
    @Cacheable(value = "users", key = "'all'")
    public List<UserResponse> getAllUsersFromCache() {
        log.debug("Cache miss for all users");
        
        return userRepository.findAll().stream()
            .map(userMapper::toResponse)
            .toList();
    }
    
    /**
     * アクティブユーザー一覧のキャッシュ取得
     */
    @Cacheable(value = "users", key = "'active'")
    public List<UserResponse> getActiveUsersFromCache() {
        log.debug("Cache miss for active users");
        
        return userRepository.findActiveUsers().stream()
            .map(userMapper::toResponse)
            .toList();
    }
    
    /**
     * ユーザー作成時のキャッシュ無効化
     */
    @CacheEvict(value = "users", allEntries = true)
    public void evictUserCacheOnCreate() {
        log.debug("Evicting all user cache entries");
    }
    
    /**
     * ユーザー更新時のキャッシュ無効化
     */
    @CacheEvict(value = "users", key = "#userId")
    public void evictUserCacheOnUpdate(String userId) {
        log.debug("Evicting user cache for: {}", userId);
    }
    
    /**
     * ユーザー削除時のキャッシュ無効化
     */
    @CacheEvict(value = "users", allEntries = true)
    public void evictUserCacheOnDelete() {
        log.debug("Evicting all user cache entries");
    }
}
```

#### 6.6.3. キャッシュキー生成器

**CustomCacheKeyGenerator.java:**
```java
package com.example.dddspanner.infrastructure.cache;

import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.Arrays;

/**
 * カスタムキャッシュキー生成器
 * 
 * キー生成の特徴：
 * - メソッド名と引数の組み合わせ
 * - 一意性の保証
 * - 可読性の向上
 */
@Component
public class CustomCacheKeyGenerator implements KeyGenerator {
    
    @Override
    public Object generate(Object target, Method method, Object... params) {
        StringBuilder keyBuilder = new StringBuilder();
        keyBuilder.append(target.getClass().getSimpleName());
        keyBuilder.append(".");
        keyBuilder.append(method.getName());
        
        if (params != null && params.length > 0) {
            keyBuilder.append(".");
            keyBuilder.append(Arrays.toString(params));
        }
        
        return keyBuilder.toString();
    }
}
``` 