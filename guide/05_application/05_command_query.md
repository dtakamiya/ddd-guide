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