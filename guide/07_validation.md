### 7.4. バリデーション

#### 7.4.1. Bean Validationの活用

Spring Bootでは、Bean Validationを使用してリクエストのバリデーションを実装できます。これにより、コントローラレベルでの入力値検証を効率的に行えます。

**CreateUserRequest.java:**
```java
package com.example.dddspanner.presentation.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

/**
 * ユーザー作成リクエストDTO
 * 
 * バリデーションの特徴：
 * - 入力値の検証
 * - エラーメッセージの国際化
 * - カスタムバリデーターの活用
 */
public record CreateUserRequest(
    @NotBlank(message = "名前は必須です")
    @Size(min = 1, max = 50, message = "名前は1文字以上50文字以下で入力してください")
    String firstName,
    
    @NotBlank(message = "姓は必須です")
    @Size(min = 1, max = 50, message = "姓は1文字以上50文字以下で入力してください")
    String lastName,
    
    @NotBlank(message = "メールアドレスは必須です")
    @Email(message = "有効なメールアドレスを入力してください")
    String email
) {}
```

#### 7.4.2. カスタムバリデーターの実装

ドメイン固有のバリデーションルールを実装するために、カスタムバリデーターを作成します。

**EmailValidator.java:**
```java
package com.example.dddspanner.presentation.validator;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

/**
 * メールアドレスカスタムバリデーター
 * 
 * バリデーションの特徴：
 * - ドメイン固有のルール
 * - パフォーマンスの最適化
 * - エラーメッセージのカスタマイズ
 */
public class EmailValidator implements ConstraintValidator<ValidEmail, String> {
    
    private static final Pattern EMAIL_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
    );
    
    @Override
    public void initialize(ValidEmail constraintAnnotation) {
        // 初期化処理（必要に応じて）
    }
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.trim().isEmpty()) {
            return false;
        }
        
        // 基本的な形式チェック
        if (!EMAIL_PATTERN.matcher(email).matches()) {
            return false;
        }
        
        // ドメイン固有のルール
        String domain = email.substring(email.indexOf('@') + 1);
        
        // 禁止ドメインのチェック
        if (isDisallowedDomain(domain)) {
            return false;
        }
        
        // ドメインの有効性チェック
        return isValidDomain(domain);
    }
    
    private boolean isDisallowedDomain(String domain) {
        // 一時的なメールアドレスや無効なドメインをチェック
        String[] disallowedDomains = {
            "10minutemail.com",
            "tempmail.org",
            "guerrillamail.com"
        };
        
        for (String disallowed : disallowedDomains) {
            if (domain.equalsIgnoreCase(disallowed)) {
                return true;
            }
        }
        
        return false;
    }
    
    private boolean isValidDomain(String domain) {
        // ドメインの有効性をチェック（簡易版）
        return domain.length() >= 3 && domain.contains(".");
    }
}
```

**ValidEmail.java:**
```java
package com.example.dddspanner.presentation.validator;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

/**
 * カスタムメールアドレスバリデーションアノテーション
 */
@Documented
@Constraint(validatedBy = EmailValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidEmail {
    String message() default "有効なメールアドレスを入力してください";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

#### 7.4.3. バリデーションエラーの処理

バリデーションエラーが発生した場合の処理を実装します。

**ValidationErrorHandler.java:**
```java
package com.example.dddspanner.presentation.handler;

import com.example.dddspanner.presentation.dto.ErrorResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;
import java.util.stream.Collectors;

/**
 * バリデーションエラーハンドラー
 * 
 * エラー処理の特徴：
 * - 詳細なエラー情報の提供
 * - クライアントフレンドリーなメッセージ
 * - ログ出力
 */
@Slf4j
@RestControllerAdvice
public class ValidationErrorHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(MethodArgumentNotValidException ex) {
        log.warn("Validation error occurred: {}", ex.getMessage());
        
        List<ErrorResponse.FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(this::mapToFieldError)
            .collect(Collectors.toList());
        
        ErrorResponse errorResponse = new ErrorResponse(
            "VALIDATION_ERROR",
            "入力値にエラーがあります",
            fieldErrors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }
    
    private ErrorResponse.FieldError mapToFieldError(FieldError fieldError) {
        return new ErrorResponse.FieldError(
            fieldError.getField(),
            fieldError.getDefaultMessage()
        );
    }
}
```

#### 7.4.4. バリデーショングループの活用

異なるユースケースで異なるバリデーションルールを適用するために、バリデーショングループを使用します。

**ValidationGroups.java:**
```java
package com.example.dddspanner.presentation.validator;

/**
 * バリデーショングループ
 * 
 * グループの特徴：
 * - ユースケース別のバリデーション
 * - 条件付きバリデーション
 * - 段階的バリデーション
 */
public interface ValidationGroups {
    
    interface Create {}
    interface Update {}
    interface Admin {}
}
```

**UpdateUserRequest.java:**
```java
package com.example.dddspanner.presentation.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Size;

/**
 * ユーザー更新リクエストDTO
 */
public record UpdateUserRequest(
    @Size(min = 1, max = 50, message = "名前は1文字以上50文字以下で入力してください", groups = {ValidationGroups.Update.class})
    String firstName,
    
    @Size(min = 1, max = 50, message = "姓は1文字以上50文字以下で入力してください", groups = {ValidationGroups.Update.class})
    String lastName,
    
    @Email(message = "有効なメールアドレスを入力してください", groups = {ValidationGroups.Update.class})
    String email
) {}
``` 