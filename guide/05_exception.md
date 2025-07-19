### 5.7. 例外処理の実装

#### 5.7.1. アプリケーション例外

**UserRegistrationException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * ユーザー登録例外
 */
public class UserRegistrationException extends RuntimeException {
    
    public UserRegistrationException(String message) {
        super(message);
    }
    
    public UserRegistrationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**UserNotFoundException.java:**
```java
package com.example.dddspanner.application.exception;

/**
 * ユーザー未発見例外
 */
public class UserNotFoundException extends RuntimeException {
    
    public UserNotFoundException(String message) {
        super(message);
    }
    
    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### 5.7.2. 例外ハンドラー

**ApplicationExceptionHandler.java:**
```java
package com.example.dddspanner.application.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.Instant;
import java.util.Map;

/**
 * アプリケーション例外ハンドラー
 */
@Slf4j
@RestControllerAdvice
public class ApplicationExceptionHandler {
    
    @ExceptionHandler(UserRegistrationException.class)
    public ResponseEntity<Map<String, Object>> handleUserRegistrationException(UserRegistrationException e) {
        log.error("User registration failed", e);
        
        Map<String, Object> response = Map.of(
            "error", "USER_REGISTRATION_FAILED",
            "message", e.getMessage(),
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleUserNotFoundException(UserNotFoundException e) {
        log.error("User not found", e);
        
        Map<String, Object> response = Map.of(
            "error", "USER_NOT_FOUND",
            "message", e.getMessage(),
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, Object>> handleIllegalArgumentException(IllegalArgumentException e) {
        log.error("Invalid argument", e);
        
        Map<String, Object> response = Map.of(
            "error", "INVALID_ARGUMENT",
            "message", e.getMessage(),
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
}
``` 