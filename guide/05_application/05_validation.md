### 5.6. バリデーションの実装

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
} 