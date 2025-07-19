### 5.5. オブジェクトマッピングの実装

#### 5.5.1. MapStructを使用したマッピング

**UserMapper.java:**
```java
package com.example.dddspanner.application.mapper;

import com.example.dddspanner.application.dto.UserResponse;
import com.example.dddspanner.domain.entity.User;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Named;

/**
 * ユーザーオブジェクトマッパー
 * 
 * MapStructの特徴：
 * - コンパイル時にマッピングコードを生成
 * - 型安全性
 * - パフォーマンスの向上
 */
@Mapper(componentModel = "spring")
public interface UserMapper {
    
    /**
     * ドメインオブジェクトからレスポンスDTOへの変換
     */
    @Mapping(source = "id.value", target = "id")
    @Mapping(source = "fullName.firstName", target = "firstName")
    @Mapping(source = "fullName.lastName", target = "lastName")
    @Mapping(source = "fullName", target = "fullName", qualifiedByName = "fullNameToString")
    @Mapping(source = "email.value", target = "email")
    @Mapping(source = "status", target = "status", qualifiedByName = "statusToString")
    UserResponse toResponse(User user);
    
    /**
     * フルネームを文字列に変換
     */
    @Named("fullNameToString")
    default String fullNameToString(FullName fullName) {
        return fullName != null ? fullName.getFullName() : null;
    }
    
    /**
     * ステータスを文字列に変換
     */
    @Named("statusToString")
    default String statusToString(UserStatus status) {
        return status != null ? status.name() : null;
    }
}
```

**OrderMapper.java:**
```java
package com.example.dddspanner.application.mapper;

import com.example.dddspanner.application.dto.OrderResponse;
import com.example.dddspanner.application.dto.OrderItemResponse;
import com.example.dddspanner.domain.entity.Order;
import com.example.dddspanner.domain.entity.OrderItem;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

/**
 * 注文オブジェクトマッパー
 */
@Mapper(componentModel = "spring", uses = {OrderItemMapper.class})
public interface OrderMapper {
    
    @Mapping(source = "id.value", target = "id")
    @Mapping(source = "userId.value", target = "userId")
    @Mapping(source = "totalAmount.amount", target = "totalAmount")
    @Mapping(source = "totalAmount.currency.currencyCode", target = "currency")
    @Mapping(source = "status", target = "status")
    OrderResponse toResponse(Order order);
}

/**
 * 注文商品オブジェクトマッパー
 */
@Mapper(componentModel = "spring")
public interface OrderItemMapper {
    
    @Mapping(source = "productId.value", target = "productId")
    @Mapping(source = "product.name", target = "productName")
    @Mapping(source = "unitPrice.amount", target = "unitPrice")
    @Mapping(source = "subtotal.amount", target = "subtotal")
    OrderItemResponse toResponse(OrderItem orderItem);
}
``` 