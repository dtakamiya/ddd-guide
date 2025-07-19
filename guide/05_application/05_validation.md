### 5.6. バリデーションの実装

アプリケーション層では、ドメイン層のバリデーションに加えて、アプリケーション固有のバリデーションを実装します。これにより、ビジネスルールの一貫性を保ち、不正なデータの流入を防ぎます。

#### 5.6.1. カスタムバリデーター

**EmailValidator.java:**
```java
package com.example.dddspanner.application.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

/**
 * メールアドレスバリデーター
 */
public class EmailValidator implements ConstraintValidator<ValidEmail, String> {
    
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.trim().isEmpty()) {
            return false;
        }
        
        return EMAIL_PATTERN.matcher(email.trim()).matches();
    }
}

/**
 * メールアドレスバリデーションアノテーション
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = EmailValidator.class)
public @interface ValidEmail {
    String message() default "Invalid email format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**PasswordValidator.java:**
```java
package com.example.dddspanner.application.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

/**
 * パスワード強度バリデーター
 */
public class PasswordValidator implements ConstraintValidator<ValidPassword, String> {
    
    private static final Pattern PASSWORD_PATTERN = 
        Pattern.compile("^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$");
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null || password.trim().isEmpty()) {
            return false;
        }
        
        return PASSWORD_PATTERN.matcher(password).matches();
    }
}

/**
 * パスワード強度バリデーションアノテーション
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordValidator.class)
public @interface ValidPassword {
    String message() default "Password must be at least 8 characters long and contain at least one uppercase letter, one lowercase letter, one number, and one special character";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**PhoneNumberValidator.java:**
```java
package com.example.dddspanner.application.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

/**
 * 電話番号バリデーター
 */
public class PhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {
    
    private static final Pattern PHONE_PATTERN = 
        Pattern.compile("^\\+?[1-9]\\d{1,14}$");
    
    @Override
    public boolean isValid(String phoneNumber, ConstraintValidatorContext context) {
        if (phoneNumber == null || phoneNumber.trim().isEmpty()) {
            return false;
        }
        
        // ハイフンやスペースを除去
        String cleaned = phoneNumber.replaceAll("[\\s\\-()]", "");
        return PHONE_PATTERN.matcher(cleaned).matches();
    }
}

/**
 * 電話番号バリデーションアノテーション
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneNumberValidator.class)
public @interface ValidPhoneNumber {
    String message() default "Invalid phone number format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

#### 5.6.2. バリデーショングループ

**ValidationGroups.java:**
```java
package com.example.dddspanner.application.validation;

/**
 * バリデーショングループ
 */
public interface ValidationGroups {
    
    interface Create {}
    interface Update {}
    interface Delete {}
    interface PartialUpdate {}
}
```

#### 5.6.3. 複合バリデーション

**OrderValidationService.java:**
```java
package com.example.dddspanner.application.validation;

import com.example.dddspanner.domain.model.order.Order;
import com.example.dddspanner.domain.model.order.OrderItem;
import com.example.dddspanner.domain.model.product.Product;
import com.example.dddspanner.domain.model.user.User;
import com.example.dddspanner.domain.repository.ProductRepository;
import com.example.dddspanner.domain.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

/**
 * 注文の複合バリデーションサービス
 */
@Service
@RequiredArgsConstructor
public class OrderValidationService {
    
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    
    /**
     * 注文作成時のバリデーション
     */
    public ValidationResult validateOrderCreation(CreateOrderCommand command) {
        ValidationResult result = new ValidationResult();
        
        // ユーザーの存在確認
        Optional<User> user = userRepository.findById(command.getUserId());
        if (user.isEmpty()) {
            result.addError("userId", "User not found");
        }
        
        // 商品の存在確認と在庫チェック
        for (OrderItemRequest itemRequest : command.getItems()) {
            Optional<Product> product = productRepository.findById(itemRequest.getProductId());
            if (product.isEmpty()) {
                result.addError("items", "Product not found: " + itemRequest.getProductId());
                continue;
            }
            
            Product p = product.get();
            if (p.getStockQuantity() < itemRequest.getQuantity()) {
                result.addError("items", "Insufficient stock for product: " + p.getName());
            }
        }
        
        // 注文金額の最小値チェック
        if (command.getTotalAmount().isLessThan(Money.of(1000))) {
            result.addError("totalAmount", "Order amount must be at least 1000 yen");
        }
        
        return result;
    }
    
    /**
     * 注文更新時のバリデーション
     */
    public ValidationResult validateOrderUpdate(UpdateOrderCommand command, Order existingOrder) {
        ValidationResult result = new ValidationResult();
        
        // 注文ステータスの変更可否チェック
        if (existingOrder.getStatus() == OrderStatus.CONFIRMED && 
            command.getStatus() == OrderStatus.DRAFT) {
            result.addError("status", "Cannot revert confirmed order to draft");
        }
        
        // キャンセル済み注文の変更禁止
        if (existingOrder.getStatus() == OrderStatus.CANCELLED) {
            result.addError("status", "Cannot modify cancelled order");
        }
        
        return result;
    }
}
```

**ValidationResult.java:**
```java
package com.example.dddspanner.application.validation;

import lombok.Getter;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * バリデーション結果
 */
@Getter
public class ValidationResult {
    
    private final Map<String, List<String>> errors = new HashMap<>();
    private final List<String> globalErrors = new ArrayList<>();
    
    public void addError(String field, String message) {
        errors.computeIfAbsent(field, k -> new ArrayList<>()).add(message);
    }
    
    public void addGlobalError(String message) {
        globalErrors.add(message);
    }
    
    public boolean hasErrors() {
        return !errors.isEmpty() || !globalErrors.isEmpty();
    }
    
    public boolean hasFieldErrors(String field) {
        return errors.containsKey(field) && !errors.get(field).isEmpty();
    }
    
    public List<String> getFieldErrors(String field) {
        return errors.getOrDefault(field, new ArrayList<>());
    }
}
```

#### 5.6.4. アプリケーションサービスでのバリデーション統合

**CreateOrderApplicationService.java:**
```java
package com.example.dddspanner.application.service;

import com.example.dddspanner.application.command.CreateOrderCommand;
import com.example.dddspanner.application.validation.OrderValidationService;
import com.example.dddspanner.application.validation.ValidationResult;
import com.example.dddspanner.domain.model.order.Order;
import com.example.dddspanner.domain.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 注文作成アプリケーションサービス
 */
@Service
@RequiredArgsConstructor
public class CreateOrderApplicationService {
    
    private final OrderRepository orderRepository;
    private final OrderValidationService validationService;
    
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // バリデーション実行
        ValidationResult validationResult = validationService.validateOrderCreation(command);
        
        if (validationResult.hasErrors()) {
            throw new ValidationException("Order validation failed", validationResult);
        }
        
        // ドメインオブジェクトの作成
        Order order = OrderFactory.createOrder(
            command.getUserId(),
            command.getItems(),
            command.getTotalAmount()
        );
        
        // 永続化
        return orderRepository.save(order);
    }
}
```

**ValidationException.java:**
```java
package com.example.dddspanner.application.exception;

import com.example.dddspanner.application.validation.ValidationResult;
import lombok.Getter;

/**
 * バリデーション例外
 */
@Getter
public class ValidationException extends RuntimeException {
    
    private final ValidationResult validationResult;
    
    public ValidationException(String message, ValidationResult validationResult) {
        super(message);
        this.validationResult = validationResult;
    }
}
```

#### 5.6.5. コマンド・クエリでのバリデーション

**CreateUserCommand.java:**
```java
package com.example.dddspanner.application.command;

import com.example.dddspanner.application.validation.ValidEmail;
import com.example.dddspanner.application.validation.ValidPassword;
import com.example.dddspanner.application.validation.ValidPhoneNumber;
import com.example.dddspanner.application.validation.ValidationGroups;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Value;

/**
 * ユーザー作成コマンド
 */
@Value
public class CreateUserCommand {
    
    @NotBlank(message = "Full name is required", groups = ValidationGroups.Create.class)
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters", 
          groups = ValidationGroups.Create.class)
    String fullName;
    
    @NotBlank(message = "Email is required", groups = ValidationGroups.Create.class)
    @ValidEmail(message = "Invalid email format", groups = ValidationGroups.Create.class)
    String email;
    
    @NotBlank(message = "Password is required", groups = ValidationGroups.Create.class)
    @ValidPassword(groups = ValidationGroups.Create.class)
    String password;
    
    @ValidPhoneNumber(message = "Invalid phone number format")
    String phoneNumber;
    
    @Size(max = 500, message = "Bio must not exceed 500 characters")
    String bio;
}
```

**UpdateUserCommand.java:**
```java
package com.example.dddspanner.application.command;

import com.example.dddspanner.application.validation.ValidEmail;
import com.example.dddspanner.application.validation.ValidPhoneNumber;
import com.example.dddspanner.application.validation.ValidationGroups;
import jakarta.validation.constraints.Size;
import lombok.Value;

/**
 * ユーザー更新コマンド
 */
@Value
public class UpdateUserCommand {
    
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters", 
          groups = ValidationGroups.Update.class)
    String fullName;
    
    @ValidEmail(message = "Invalid email format", groups = ValidationGroups.Update.class)
    String email;
    
    @ValidPhoneNumber(message = "Invalid phone number format")
    String phoneNumber;
    
    @Size(max = 500, message = "Bio must not exceed 500 characters")
    String bio;
}
```

#### 5.6.6. グローバルバリデーション設定

**GlobalValidationConfig.java:**
```java
package com.example.dddspanner.application.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.validation.beanvalidation.MethodValidationPostProcessor;

/**
 * グローバルバリデーション設定
 */
@Configuration
public class GlobalValidationConfig {
    
    @Bean
    public LocalValidatorFactoryBean validator() {
        return new LocalValidatorFactoryBean();
    }
    
    @Bean
    public MethodValidationPostProcessor methodValidationPostProcessor() {
        return new MethodValidationPostProcessor();
    }
}
```

#### 5.6.7. バリデーション戦略のベストプラクティス

**1. レイヤー別バリデーション**
- **プレゼンテーション層**: 基本的な形式チェック（必須項目、文字数制限など）
- **アプリケーション層**: ビジネスルールに基づく複合バリデーション
- **ドメイン層**: ドメインルールに基づく不変条件のチェック

**2. バリデーションの優先順位**
```java
// 1. 基本的な形式チェック（@NotNull, @Size等）
// 2. カスタムバリデーター（@ValidEmail, @ValidPassword等）
// 3. 複合バリデーション（ビジネスルール）
// 4. ドメイン層での最終チェック
```

**3. エラーメッセージの国際化**
```properties
# messages.properties
validation.email.invalid=Invalid email format
validation.password.weak=Password must be at least 8 characters long
validation.user.notfound=User not found
```

**4. バリデーションのパフォーマンス**
- 早期リターンで不要な処理を避ける
- データベースアクセスを最小限に抑える
- キャッシュを活用する

この包括的なバリデーション戦略により、アプリケーションの堅牢性とユーザビリティを大幅に向上させることができます。 