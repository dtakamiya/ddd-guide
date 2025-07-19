### 5.8. アプリケーション層のテスト

#### 5.8.1. アプリケーションサービスのテスト

**UserApplicationServiceTest.groovy:**
```groovy
package com.example.dddspanner.application.service

import com.example.dddspanner.application.dto.CreateUserRequest
import com.example.dddspanner.application.dto.UserResponse
import com.example.dddspanner.application.exception.UserRegistrationException
import com.example.dddspanner.application.mapper.UserMapper
import com.example.dddspanner.domain.entity.User
import com.example.dddspanner.domain.factory.UserFactory
import com.example.dddspanner.domain.repository.UserRepository
import com.example.dddspanner.domain.service.UserDuplicateChecker
import com.example.dddspanner.domain.valueobject.Email
import com.example.dddspanner.domain.valueobject.FullName
import com.example.dddspanner.domain.valueobject.UserId
import spock.lang.Specification

class UserApplicationServiceTest extends Specification {
    
    UserRepository userRepository = Mock()
    UserFactory userFactory = Mock()
    UserDuplicateChecker duplicateChecker = Mock()
    UserMapper userMapper = Mock()
    
    UserApplicationService service
    
    def setup() {
        service = new UserApplicationService(
            userRepository, userFactory, duplicateChecker, userMapper
        )
    }
    
    def "should register user successfully"() {
        given:
        def request = new CreateUserRequest("Taro", "Yamada", "taro.yamada@example.com")
        def fullName = new FullName("Taro", "Yamada")
        def email = new Email("taro.yamada@example.com")
        def user = Mock(User)
        def userId = UserId.generate()
        def savedUser = Mock(User)
        def response = Mock(UserResponse)
        
        when:
        def result = service.registerUser(request)
        
        then:
        1 * duplicateChecker.validateUserCreation(email)
        1 * userFactory.createUser(fullName, email) >> user
        1 * userRepository.save(user) >> savedUser
        1 * userMapper.toResponse(savedUser) >> response
        
        and:
        result == response
    }
    
    def "should throw exception when user creation fails"() {
        given:
        def request = new CreateUserRequest("Taro", "Yamada", "taro.yamada@example.com")
        def email = new Email("taro.yamada@example.com")
        
        and:
        duplicateChecker.validateUserCreation(email) >> { throw new RuntimeException("Database error") }
        
        when:
        service.registerUser(request)
        
        then:
        thrown(UserRegistrationException)
    }
    
    def "should throw exception for invalid request"() {
        when:
        service.registerUser(null)
        
        then:
        thrown(IllegalArgumentException)
    }
}
``` 