### 6.8. 暗号化とセキュリティ

#### 6.8.1. データ暗号化の実装

**EncryptionService.java:**
```java
package com.example.dddspanner.infrastructure.security;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Base64;

/**
 * 暗号化サービス
 * 
 * セキュリティの特徴：
 * - データの暗号化
 * - ハッシュ化
 * - セキュアな鍵管理
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class EncryptionService {
    
    @Value("${app.encryption.key}")
    private String encryptionKey;
    
    private static final String ALGORITHM = "AES";
    private static final String HASH_ALGORITHM = "SHA-256";
    
    /**
     * データの暗号化
     */
    public String encrypt(String data) {
        try {
            SecretKeySpec secretKey = new SecretKeySpec(
                encryptionKey.getBytes(StandardCharsets.UTF_8), 
                ALGORITHM
            );
            
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey);
            
            byte[] encryptedBytes = cipher.doFinal(data.getBytes());
            return Base64.getEncoder().encodeToString(encryptedBytes);
            
        } catch (Exception e) {
            log.error("Failed to encrypt data", e);
            throw new EncryptionException("Failed to encrypt data", e);
        }
    }
    
    /**
     * データの復号化
     */
    public String decrypt(String encryptedData) {
        try {
            SecretKeySpec secretKey = new SecretKeySpec(
                encryptionKey.getBytes(StandardCharsets.UTF_8), 
                ALGORITHM
            );
            
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, secretKey);
            
            byte[] decryptedBytes = cipher.doFinal(
                Base64.getDecoder().decode(encryptedData)
            );
            return new String(decryptedBytes);
            
        } catch (Exception e) {
            log.error("Failed to decrypt data", e);
            throw new EncryptionException("Failed to decrypt data", e);
        }
    }
    
    /**
     * データのハッシュ化
     */
    public String hash(String data) {
        try {
            MessageDigest digest = MessageDigest.getInstance(HASH_ALGORITHM);
            byte[] hashBytes = digest.digest(data.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hashBytes);
            
        } catch (Exception e) {
            log.error("Failed to hash data", e);
            throw new EncryptionException("Failed to hash data", e);
        }
    }
    
    /**
     * パスワードの検証
     */
    public boolean verifyPassword(String password, String hashedPassword) {
        String hashedInput = hash(password);
        return hashedInput.equals(hashedPassword);
    }
}
```

#### 6.8.2. セキュリティ設定

**SecurityConfig.java:**
```java
package com.example.dddspanner.infrastructure.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

/**
 * セキュリティ設定
 * 
 * セキュリティの特徴：
 * - 認証・認可の実装
 * - パスワードエンコーダー
 * - セキュリティフィルター
 */
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    /**
     * パスワードエンコーダー
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    /**
     * セキュリティフィルター設定
     */
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasRole("USER")
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable())
            .httpBasic();
        
        return http.build();
    }
}
```

#### 6.8.3. JWT認証の実装

**JwtTokenService.java:**
```java
package com.example.dddspanner.infrastructure.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

/**
 * JWTトークンサービス
 * 
 * JWT認証の特徴：
 * - ステートレス認証
 * - トークンベース認証
 * - セキュアな署名
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class JwtTokenService {
    
    @Value("${app.jwt.secret}")
    private String jwtSecret;
    
    @Value("${app.jwt.expiration}")
    private Long jwtExpiration;
    
    /**
     * JWTトークンの生成
     */
    public String generateToken(String userId, String email) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", userId);
        claims.put("email", email);
        
        return createToken(claims, userId);
    }
    
    /**
     * トークンの作成
     */
    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(subject)
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
            .signWith(SignatureAlgorithm.HS256, jwtSecret)
            .compact();
    }
    
    /**
     * トークンの検証
     */
    public Boolean validateToken(String token, String userId) {
        final String tokenUserId = extractUserId(token);
        return (userId.equals(tokenUserId) && !isTokenExpired(token));
    }
    
    /**
     * ユーザーIDの抽出
     */
    public String extractUserId(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    /**
     * メールアドレスの抽出
     */
    public String extractEmail(String token) {
        return extractClaim(token, claims -> claims.get("email", String.class));
    }
    
    /**
     * 有効期限の抽出
     */
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    /**
     * クレームの抽出
     */
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    /**
     * 全クレームの抽出
     */
    private Claims extractAllClaims(String token) {
        return Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token).getBody();
    }
    
    /**
     * トークンの有効期限チェック
     */
    private Boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}
``` 