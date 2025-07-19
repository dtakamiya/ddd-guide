### 7.2. リクエストとレスポンスの設計

コントローラは、ドメインオブジェクトを直接APIの入出力として使うべきではありません。代わりに、そのユースケースに特化したDTO (Data Transfer Object) を使います。

**DTOを使用する主な理由:**
*   **関心の分離**: ドメインモデルと、外部に公開するAPIのデータモデルを分離します。
*   **堅牢性**: ドメインモデルの内部的な変更（フィールドの追加など）が、意図せずAPIの互換性を破壊するのを防ぎます。
*   **セキュリティ**: `User`エンティティにパスワードハッシュのような機微な情報が含まれていても、レスポンス用のDTOにそれを含めないことで、情報漏洩のリスクをなくします。
*   **最適化**: APIが必要とするデータだけを過不足なく含んだ、軽量なオブジェクトを作成できます。

本ガイドでは、入力には`CreateUserRequest`を、出力には`UserResponse`を使用します。

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