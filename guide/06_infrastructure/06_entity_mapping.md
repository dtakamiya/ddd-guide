### 6.3. エンティティマッピング

#### 6.3.1. 永続化エンティティの実装

**UserEntity.java:**
```java
package com.example.dddspanner.infrastructure.entity;

import com.google.cloud.spring.data.spanner.core.mapping.Column;
import com.google.cloud.spring.data.spanner.core.mapping.PrimaryKey;
import com.google.cloud.spring.data.spanner.core.mapping.Table;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;

/**
 * ユーザー永続化エンティティ
 * 
 * Spannerエンティティの特徴：
 * - テーブル構造との対応
 * - カラムマッピング
 * - インデックス最適化
 */
@Data
@NoArgsConstructor
@Table(name = "users")
public class UserEntity {
    
    @PrimaryKey
    @Column(name = "user_id")
    private String userId;
    
    @Column(name = "first_name")
    private String firstName;
    
    @Column(name = "last_name")
    private String lastName;
    
    @Column(name = "email")
    private String email;
    
    @Column(name = "status")
    private String status;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

#### 6.3.2. エンティティマッパーの実装

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
 * マッピングの特徴：
 * - ドメインオブジェクトと永続化オブジェクトの変換
 * - 値オブジェクトの適切な処理
 * - 型安全性の保証
 */
@Mapper(componentModel = "spring")
public interface UserEntityMapper {
    
    /**
     * ドメインオブジェクトからエンティティへの変換
     */
    @Mapping(source = "id.value", target = "userId")
    @Mapping(source = "fullName.firstName", target = "firstName")
    @Mapping(source = "fullName.lastName", target = "lastName")
    @Mapping(source = "email.value", target = "email")
    @Mapping(source = "status", target = "status", qualifiedByName = "statusToString")
    UserEntity toEntity(User user);
    
    /**
     * エンティティからドメインオブジェクトへの変換
     */
    @Mapping(target = "id", source = "userId", qualifiedByName = "stringToUserId")
    @Mapping(target = "fullName", expression = "java(new FullName(entity.getFirstName(), entity.getLastName()))")
    @Mapping(target = "email", source = "email", qualifiedByName = "stringToEmail")
    @Mapping(target = "status", source = "status", qualifiedByName = "stringToUserStatus")
    @Mapping(target = "domainEvents", ignore = true)
    User toDomain(UserEntity entity);
    
    /**
     * UserIdへの変換
     */
    @Named("stringToUserId")
    default UserId stringToUserId(String userId) {
        return userId != null ? UserId.of(userId) : null;
    }
    
    /**
     * Emailへの変換
     */
    @Named("stringToEmail")
    default Email stringToEmail(String email) {
        return email != null ? new Email(email) : null;
    }
    
    /**
     * UserStatusへの変換
     */
    @Named("stringToUserStatus")
    default UserStatus stringToUserStatus(String status) {
        return status != null ? UserStatus.valueOf(status) : null;
    }
    
    /**
     * UserStatusから文字列への変換
     */
    @Named("statusToString")
    default String statusToString(UserStatus status) {
        return status != null ? status.name() : null;
    }
}
``` 