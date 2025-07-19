## 7. プレゼンテーション層の実装

この層は、ユーザーや外部システムとのインターフェースを担当します。本ガイドでは、Spring MVCを利用したREST APIの実装に焦点を当てます。

### 7.1. REST APIコントローラの実装 (Spring MVC)

`@RestController`アノテーションをクラスに付与し、`@GetMapping`, `@PostMapping`などのアノテーションをメソッドに付与することで、簡単にRESTfulなエンドポイントを定義できます。

*   **責務:**
    *   HTTPリクエストを受け取る。
    *   リクエストのペイロード（JSONなど）をDTOにマッピングする。
    *   アプリケーションサービスを呼び出し、ユースケースを実行する。
    *   アプリケーションサービスから返された結果を、HTTPレスポンス（JSONなど）に変換して返却する。
*   **注意点:**
    *   コントローラにはビジネスロジックを記述しません。ロジックはアプリケーション層またはドメイン層に配置します。
    *   リクエストのバリデーションはコントローラの責務の一部ですが、ドメインの不変条件の検証とは区別します。（詳細は後述）

### 7.2. リクエストとレスポンスの設計

コントローラは、ドメインオブジェクトを直接APIの入出力として使うべきではありません。代わりに、そのユースケースに特化したDTO (Data Transfer Object) を使います。

**DTOを使用する主な理由:**
*   **関心の分離**: ドメインモデルと、外部に公開するAPIのデータモデルを分離します。
*   **堅牢性**: ドメインモデルの内部的な変更（フィールドの追加など）が、意図せずAPIの互換性を破壊するのを防ぎます。
*   **セキュリティ**: `User`エンティティにパスワードハッシュのような機微な情報が含まれていても、レスポンス用のDTOにそれを含めないことで、情報漏洩のリスクをなくします。
*   **最適化**: APIが必要とするデータだけを過不足なく含んだ、軽量なオブジェクトを作成できます。

本ガイドでは、入力には`CreateUserCommand`を、出力には`UserResponse`を使用します。

**UserResponse.java (レスポンスDTO):**
```java
import org.springframework.hateoas.RepresentationModel;
import org.springframework.hateoas.server.core.Relation;

/**
 * プレゼンテーション層に返すユーザー情報を格納するDTO。
 * RepresentationModelを継承することで、HATEOASのリンクを付与できるようになる。
 */
@Relation(collectionRelation = "users", itemRelation = "user") // JSONレスポンスのルート要素名を指定
public class UserResponse extends RepresentationModel<UserResponse> {
    private final String id;
    private final String fullName;
    private final String email;

    // Getter, Constructor
}
```

#### 7.2.1. HATEOASによるAPIの自己文書化

HATEOAS (Hypermedia as the Engine of Application State) は、レスポンスにリソースに関連する次のアクションへのリンクを含めるRESTアーキテクチャの原則です。これにより、クライアントはレスポンス内のリンクを辿るだけでAPIを操作でき、URLをハードコーディングする必要がなくなります。

Spring HATEOASを利用すると、このリンク生成を簡単に行えます。

**UserResponseAssembler.java (HATEOASリンクを付与するAssembler):**
```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

import org.springframework.hateoas.server.mvc.RepresentationModelAssemblerSupport;
import org.springframework.stereotype.Component;

@Component
public class UserResponseAssembler extends RepresentationModelAssemblerSupport<User, UserResponse> {

    public UserResponseAssembler() {
        super(UserRestController.class, UserResponse.class);
    }

    @Override
    public UserResponse toModel(User user) {
        UserResponse response = new UserResponse(user.getId(), user.getFullName().getFullName(), user.getEmail().value());
        // "self"リレーションシップのリンクを追加 (例: /api/v1/users/{id})
        response.add(linkTo(methodOn(UserRestController.class).getUserById(user.getId())).withSelfRel());
        return response;
    }
}
```
この`Assembler`は、`User`ドメインオブジェクトを`UserResponse` DTOに変換し、HATEOASリンクを付与する責務を持ちます。

**UserRestController.java (Assemblerを利用):**
```java
// ...
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserRestController {

    private final CreateUserApplicationService createUserApplicationService;
    private final UserResponseAssembler userResponseAssembler; // Assemblerを注入

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserCommand command) {
        User user = createUserApplicationService.createUser(command);
        // Assemblerを使ってDTOに変換し、リンクを付与
        UserResponse response = userResponseAssembler.toModel(user);
        return ResponseEntity.created(response.getRequiredLink("self").toUri()).body(response);
    }
    
    // @GetMapping("/{id}") のような他のメソッドでも同様にAssemblerを利用する
}
```

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

### 7.4. Spring SecurityによるAPI保護 (モダンな設定)

`WebSecurityConfigurerAdapter`は非推奨となったため、コンポーネントベースの`SecurityFilterChain` Beanを定義する方法が推奨されます。

**SecurityConfig.java:**
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtRequestFilter jwtRequestFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // CSRFはステートレスなAPIでは不要なため無効化
            .csrf(csrf -> csrf.disable())
            // 認証が必要なエンドポイントの設定
            .authorizeHttpRequests(authz -> authz
                // /api/authenticate エンドポイントは認証なしでアクセスを許可
                .requestMatchers("/api/authenticate").permitAll()
                // それ以外の /api/v1/** 以下のリクエストはすべて認証が必要
                .requestMatchers("/api/v1/**").authenticated()
                // その他のリクエストはすべて拒否
                .anyRequest().denyAll()
            )
            // 認証失敗時のエントリーポイントを設定
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
            )
            // セッション管理をステートレスに設定
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );

        // 自作のJWTフィルターをUsernamePasswordAuthenticationFilterの前に挿入
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```
この設定では、非推奨の`antMatchers`の代わりに`requestMatchers`を使用し、より明確かつ安全にエンドポイントごとの認可ルールを定義しています。

#### 7.4.1. 必要な依存関係の追加

`build.gradle`にJWTライブラリの依存関係を追加します。ここでは、広く使われている`io.jsonwebtoken:jjwt`を利用します。

```groovy
// build.gradle
dependencies {
    // ...
    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.11.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.11.5'
}
```

#### 7.4.2. JWTユーティリティクラスの作成

JWTの生成、検証、クレームの抽出などを行うユーティリティクラスを作成します。

**JwtTokenUtil.java:**
```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import java.security.Key;
import java.util.Date;
import java.util.function.Function;

@Component
public class JwtTokenUtil {

    public static final long JWT_TOKEN_VALIDITY = 5 * 60 * 60; // 5 hours

    @Value("${jwt.secret}")
    private String secret;

    private Key getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }

    // JWTからユーザー名を抽出
    public String getUsernameFromToken(String token) {
        return getClaimFromToken(token, Claims::getSubject);
    }

    // JWTから有効期限を抽出
    public Date getExpirationDateFromToken(String token) {
        return getClaimFromToken(token, Claims::getExpiration);
    }

    public <T> T getClaimFromToken(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = getAllClaimsFromToken(token);
        return claimsResolver.apply(claims);
    }

    // JWTの全てのクレームを抽出
    private Claims getAllClaimsFromToken(String token) {
        return Jwts.parserBuilder().setSigningKey(getSigningKey()).build().parseClaimsJws(token).getBody();
    }

    // トークンの有効期限が切れているかチェック
    private Boolean isTokenExpired(String token) {
        final Date expiration = getExpirationDateFromToken(token);
        return expiration.before(new Date());
    }

    // ユーザー情報に基づいてJWTを生成
    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + JWT_TOKEN_VALIDITY * 1000))
                .signWith(getSigningKey(), SignatureAlgorithm.HS512)
                .compact();
    }

    // トークンの検証
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = getUsernameFromToken(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
}
```
`application.properties`にJWTの秘密鍵を設定する必要があります。
`jwt.secret=your-super-strong-and-long-secret-key-for-jwt`

#### 7.4.3. 認証フィルタの実装

リクエスト毎にJWTを検証し、認証コンテキストを設定するカスタムフィルタを実装します。

**JwtRequestFilter.java:**
```java
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component
@RequiredArgsConstructor
public class JwtRequestFilter extends OncePerRequestFilter {

    private final UserDetailsService userDetailsService;
    private final JwtTokenUtil jwtTokenUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        final String requestTokenHeader = request.getHeader("Authorization");

        String username = null;
        String jwtToken = null;

        if (requestTokenHeader != null && requestTokenHeader.startsWith("Bearer ")) {
            jwtToken = requestTokenHeader.substring(7);
            try {
                username = jwtTokenUtil.getUsernameFromToken(jwtToken);
            } catch (IllegalArgumentException e) {
                logger.warn("Unable to get JWT Token");
            } catch (Exception e) {
                logger.warn("JWT Token has expired or is invalid");
            }
        } else {
            logger.warn("JWT Token does not begin with Bearer String");
        }

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

            if (jwtTokenUtil.validateToken(jwtToken, userDetails)) {
                UsernamePasswordAuthenticationToken usernamePasswordAuthenticationToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                usernamePasswordAuthenticationToken
                        .setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(usernamePasswordAuthenticationToken);
            }
        }
        chain.doFilter(request, response);
    }
}
```

#### 7.4.4. SecurityFilterChainの設定

最後に、`SecurityFilterChain`を構成し、作成したフィルタを組み込み、エンドポイント毎のアクセス制御を設定します。

**SecurityConfig.java:**
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtRequestFilter jwtRequestFilter;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/authenticate", "/api/v1/register").permitAll() // 認証・登録APIは公開
                .anyRequest().authenticated() // その他のAPIは認証が必要
            )
            .exceptionHandling(ex -> ex.authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)); // ステートレスセッション

        // 作成したJWTフィルタをUsernamePasswordAuthenticationFilterの前に追加
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```
認証の入口となる`JwtAuthenticationEntryPoint`も定義します。
```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint, Serializable {

    private static final long serialVersionUID = -7858869558953243875L;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
    }
}
```
この設定により、`/api/v1/authenticate`と`/api/v1/register`エンドポイントは認証なしでアクセス可能となり、それ以外のすべてのリクエストは有効なJWTを`Authorization: Bearer <token>`ヘッダーに含める必要があります。 