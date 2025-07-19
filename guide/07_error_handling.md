### 7.3. 効果的なエラーハンドリング

REST APIでは、クライアントに対して一貫性のある分かりやすいエラーレスポンスを返すことが重要です。Spring Bootでは、`@ControllerAdvice`と`@ExceptionHandler`を組み合わせることで、グローバルな例外処理を効率的に実装できます。

#### 7.3.1. エラーレスポンスの標準化

まず、API全体で統一されたエラーレスポンスの形式を定義します。これにより、クライアント側はエラー処理を容易に実装できます。

**ErrorResponse.java:**
```java
import com.fasterxml.jackson.annotation.JsonInclude;

import java.util.Collections;
import java.util.List;

/**
 * APIのエラーレスポンスを表現する不変のデータキャリア。
 * recordで定義し、フィールドエラーがない場合はJSONに出力しないように設定。
 *
 * @param errorCode 一意のエラーコード
 * @param message エラーメッセージ
 * @param fieldErrors 各フィールドのバリデーションエラーリスト
 */
@JsonInclude(JsonInclude.Include.NON_EMPTY)
public record ErrorResponse(
    String errorCode,
    String message,
    List<FieldError> fieldErrors
) {

    /**
     * フィールドごとのエラー詳細。
     */
    public record FieldError(String field, String message) {}

    // 特定のビジネスエラー用
    public static ErrorResponse of(String errorCode, String message) {
        return new ErrorResponse(errorCode, message, Collections.emptyList());
    }

    // 入力値バリデーションエラー用
    public static ErrorResponse of(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(error -> new FieldError(error.getField(), error.getDefaultMessage()))
                .toList(); // Java 16+ の toList() を活用
        return new ErrorResponse("INVALID_INPUT", "入力値が無効です。", errors);
    }
}
```
この`ErrorResponse`は、一意のエラーコード、人間が読めるメッセージ、そして（必要であれば）どのフィールドでエラーが発生したかの詳細情報を含みます。`@JsonInclude(JsonInclude.Include.NON_EMPTY)`により、`fieldErrors`リストが空の場合はJSONレスポンスに含まれなくなり、レスポンスがスッキリします。

#### 7.3.2. グローバル例外ハンドラの実装

`@ControllerAdvice`アノテーションを付与したクラスを作成し、特定の例外を捕捉する`@ExceptionHandler`メソッドを定義します。これにより、複数のコントローラに散らばりがちな例外処理ロジックを一箇所に集約できます。

**GlobalExceptionHandler.java:**
```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Bean Validationによる入力値検証エラー
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleMethodArgumentNotValid(MethodArgumentNotValidException ex) {
        ErrorResponse errorResponse = ErrorResponse.of(ex);
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }

    // ドメイン層などで発生するカスタムビジネス例外
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        ErrorResponse errorResponse = ErrorResponse.of(ex.getErrorCode(), ex.getMessage());
        // ビジネス例外に応じたHTTPステータスを返す（例: 400, 404, 409など）
        return ResponseEntity.status(ex.getHttpStatus()).body(errorResponse);
    }
    
    // 予期せぬサーバーエラー
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpectedException(Exception ex) {
        // ログにはスタックトレースを出力する
        log.error("Unexpected error occurred", ex);
        ErrorResponse errorResponse = ErrorResponse.of("INTERNAL_SERVER_ERROR", "予期せぬエラーが発生しました。");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}
```

#### 7.3.3. カスタムビジネス例外の定義

ドメインのルール違反など、特定のビジネスロジックに起因するエラーは、専用のカスタム例外として定義します。これにより、どのような問題が発生したのかを型レベルで明確にできます。

**BusinessException.java:**
```java
@Getter
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;
    private final HttpStatus httpStatus;

    public BusinessException(String errorCode, String message, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }
}
```

**具体的な例外クラス:**
```java
public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(String userId) {
        super("USER_NOT_FOUND", "指定されたユーザーが見つかりません。 ID: " + userId, HttpStatus.NOT_FOUND);
    }
}

public class DuplicateEmailException extends BusinessException {
    public DuplicateEmailException(String email) {
        super("DUPLICATE_EMAIL", "指定されたメールアドレスは既に使用されています。 Email: " + email, HttpStatus.CONFLICT);
    }
}
```

このように実装することで、各コントローラは例外処理を意識することなく、正常系の処理に集中できます。例外が発生した場合、`GlobalExceptionHandler`がそれを捕捉し、定義されたフォーマットで一貫性のあるエラーレスポンスをクライアントに返却します。 