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
            String sql = "SELECT * FROM users WHERE email = @email";
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
                Map.of(), 
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
            Long count = spannerTemplate.queryForObject(
                sql, 
                Map.of("userId", id.value()), 
                Long.class
            );
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
            Long count = spannerTemplate.queryForObject(
                sql, 
                Map.of("email", email.value()), 
                Long.class
            );
            return count != null && count > 0;
        } catch (Exception e) {
            log.error("Failed to check email existence: {}", email, e);
            throw new UserQueryException("Failed to check email existence", e);
        }
    }
}
``` 