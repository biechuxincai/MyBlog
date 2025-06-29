---
title: Valid与Validated
toc: true
description: more
tags:
  - java
  - springboot
categories:
  - java
date: 2025-06-17 12:43:54
---
@Valid 和 @Validated 的区别和使用场景：



1. 来源不同

- @Valid 是 JSR-303 规范定义的标准注解，位于 javax.validation 包

- @Validated 是 Spring 框架定义的注解，是对 @Valid 的扩展，位于 org.springframework.validation 包



2. 主要区别

- 分组校验：@Validated 支持分组校验，而 @Valid 不支持

- 使用位置：@Validated 只能用在类型、方法和方法参数上，而 @Valid 可以用在方法、构造函数、方法参数和成员属性上

- 嵌套校验：@Valid 支持嵌套校验，而 @Validated 不支持



3. 具体使用示例：



基础校验：

```java

@RestController

public class UserController {

    

    @PostMapping("/user")

    public String createUser(@Valid @RequestBody User user) {

        // 使用 @Valid 验证请求体

        return "success";

    }

}



public class User {

    @NotNull

    private String name;

    

    @Email

    private String email;

}

```



分组校验：

```java

// 定义分组接口

public interface Create {}

public interface Update {}



public class User {

    @Null(groups = Create.class)  // 创建时 ID 必须为空

    @NotNull(groups = Update.class)  // 更新时 ID 不能为空

    private Long id;

    

    @NotBlank(groups = {Create.class, Update.class})

    private String name;

}



@RestController

@Validated  // 在类上标注 @Validated

public class UserController {

    

    @PostMapping

    public User create(@Validated(Create.class) @RequestBody User user) {

        return userService.create(user);

    }

    

    @PutMapping

    public User update(@Validated(Update.class) @RequestBody User user) {

        return userService.update(user);

    }

}

```



嵌套校验：

```java

public class User {

    @NotNull

    private String name;

    

    @Valid  // 需要使用 @Valid 才能验证嵌套对象

    private Address address;

}



public class Address {

    @NotBlank

    private String street;

    

    @NotBlank

    private String city;

}

```



4. 最佳实践建议：



- 如果需要分组校验，使用 @Validated

- 如果需要嵌套验证，使用 @Valid

- 在 Controller 层推荐使用 @Validated，可以结合分组校验实现更细粒度的控制

- 在实体类中使用 @Valid 进行嵌套验证

- 对于简单的参数校验，两者都可以，选择 @Valid 更符合 JSR 标准



5. 异常处理：

```java

@RestControllerAdvice

public class ValidationExceptionHandler {

    

    @ExceptionHandler(MethodArgumentNotValidException.class)

    public ResponseEntity<String> handleValidationExceptions(MethodArgumentNotValidException ex) {

        // 处理校验异常

        String errors = ex.getBindingResult()

            .getAllErrors()

            .stream()

            .map(ObjectError::getDefaultMessage)

            .collect(Collectors.joining(", "));

        return ResponseEntity.badRequest().body(errors);

    }

}

```



需要注意的是，要使用这些验证功能，需要确保项目中引入了相关依赖：



```xml

<dependency>

    <groupId>org.springframework.boot</groupId>

    <artifactId>spring-boot-starter-validation</artifactId>

</dependency>

```