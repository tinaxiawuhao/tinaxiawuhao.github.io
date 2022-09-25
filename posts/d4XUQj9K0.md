---
title: 'springBoot参数和业务校验'
date: 2022-02-25 11:05:50
tags: [springBoot]
published: true
hideInList: false
feature: /post-images/d4XUQj9K0.png
isTop: false
---
## 为什么需要参数校验

在日常的接口开发中，为了防止非法参数对业务造成影响，经常需要对接口的参数做校验，例如登录的时候需要校验用户名密码是否为空，创建用户的时候需要校验邮件、手机号码格式是否准确。靠代码对接口参数一个个校验的话就太繁琐了，代码可读性极差。

Validator框架就是为了解决开发人员在开发的时候少写代码，提升开发效率

> Validator校验框架遵循了JSR-303验证规范（参数校验规范）, JSR是`Java Specification Requests`的缩写。

接下来我们看看在SpringbBoot中如何集成参数校验框架。

## SpringBoot中集成参数校验

### 第一步，引入依赖

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

> 注：从`springboot-2.3`开始，校验包被独立成了一个`starter`组件，所以需要引入validation和web，而`springboot-2.3`之前的版本只需要引入 web 依赖就可以了。

### 第二步，定义要参数校验的实体类

```
@Data
public class ValidVO {
    private String id;

    @Length(min = 6,max = 12,message = "appId长度必须位于6到12之间")
    private String appId;

    @NotBlank(message = "名字为必填项")
    private String name;

    @Email(message = "请填写正确的邮箱地址")
    private String email;

    private String sex;

    @NotEmpty(message = "级别不能为空")
    private String level;
}
```



在实际开发中对于需要校验的字段都需要设置对应的业务提示，即message属性。

常见的约束注解如下：

| 注解         | 功能                                                         |
| :----------- | :----------------------------------------------------------- |
| @AssertFalse | 可以为null,如果不为null的话必须为false                       |
| @AssertTrue  | 可以为null,如果不为null的话必须为true                        |
| @DecimalMax  | 设置不能超过最大值                                           |
| @DecimalMin  | 设置不能超过最小值                                           |
| @Digits      | 设置必须是数字且数字整数的位数和小数的位数必须在指定范围内   |
| @Future      | 日期必须在当前日期的未来                                     |
| @Past        | 日期必须在当前日期的过去                                     |
| @Max         | 最大不得超过此最大值                                         |
| @Min         | 最大不得小于此最小值                                         |
| @NotNull     | 不能为null，可以是空                                         |
| @Null        | 必须为null                                                   |
| @Pattern     | 必须满足指定的正则表达式                                     |
| @Size        | 集合、数组、map等的size()值必须在指定范围内                  |
| @Email       | 必须是email格式                                              |
| @Length      | 长度必须在指定范围内                                         |
| @NotBlank    | 字符串不能为null,字符串trim()后也不能等于“”                  |
| @NotEmpty    | 不能为null，集合、数组、map等size()不能为0；字符串trim()后可以等于“” |
| @Range       | 值必须在指定范围内                                           |
| @URL         | 必须是一个URL                                                |

注：此表格只是简单的对注解功能的说明，并没有对每一个注解的属性进行说明；可详见源码。

### 第三步，定义校验类进行测试

```
@RestController
@Slf4j
@Validated
public class ValidController {

    @ApiOperation("RequestBody校验")
    @PostMapping("/valid/test1")   
    public String test1(@Validated @RequestBody ValidVO validVO){
        log.info("validEntity is {}", validVO);
        return "test1 valid success";
    }

    @ApiOperation("Form校验")
    @PostMapping(value = "/valid/test2")
    public String test2(@Validated ValidVO validVO){
        log.info("validEntity is {}", validVO);
        return "test2 valid success";
    }
  
   @ApiOperation("单参数校验")
    @PostMapping(value = "/valid/test3")
    public String test3(@Email String email){
        log.info("email is {}", email);
        return "email valid success";
    }
}
```

这里我们先定义三个方法test1，test2，test3，test1使用了`@RequestBody`注解，用于接受前端发送的json数据，test2模拟表单提交，test3模拟单参数提交。**注意，当使用单参数校验时需要在Controller上加上@Validated注解，否则不生效**。

### 第四步，体验效果

1. 调用test1方法，提示的是`org.springframework.web.bind.MethodArgumentNotValidException`异常

```
POST http://localhost:8080/valid/test1
Content-Type: application/json

{
  "id": 1,
  "level": "12",
  "email": "47693899",
  "appId": "ab1c"
}
{
  "status": 500,
  "message": "Validation failed for argument [0] in public java.lang.String com.jianzh5.blog.valid.ValidController.test1(com.jianzh5.blog.valid.ValidVO) with 3 errors: [Field error in object 'validVO' on field 'email': rejected value [47693899]; codes [Email.validVO.email,Email.email,Email.java.lang.String,Email]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [validVO.email,email]; arguments []; default message [email],[Ljavax.validation.constraints.Pattern$Flag;@26139123,.*]; default message [不是一个合法的电子邮件地址]]...",
  "data": null,
  "timestamp": 1628239624332
}
```

1. 调用test2方法，提示的是`org.springframework.validation.BindException`异常

```
POST http://localhost:8080/valid/test2
Content-Type: application/x-www-form-urlencoded

id=1&level=12&email=476938977&appId=ab1c
{
  "status": 500,
  "message": "org.springframework.validation.BeanPropertyBindingResult: 3 errors\nField error in object 'validVO' on field 'name': rejected value [null]; codes [NotBlank.validVO.name,NotBlank.name,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [validVO.name,name]; arguments []; default message [name]]; default message [名字为必填项]...",
  "data": null,
  "timestamp": 1628239301951
}
```

1. 调用test3方法，提示的是`javax.validation.ConstraintViolationException`异常

```
POST http://localhost:8080/valid/test3
Content-Type: application/x-www-form-urlencoded

email=476938977
{
  "status": 500,
  "message": "test3.email: 不是一个合法的电子邮件地址",
  "data": null,
  "timestamp": 1628239281022
}
```

通过加入`Validator`校验框架可以帮助我们自动实现参数的校验。

## 参数异常加入全局异常处理器

`Validator`校验框架返回的错误提示太臃肿了，不便于阅读，为了方便前端提示，我们需要将其简化一下。

创建`RestExceptionHandler`，单独拦截参数校验的三个异常：`javax.validation.ConstraintViolationException`，`org.springframework.validation.BindException`，`org.springframework.web.bind.MethodArgumentNotValidException`，代码如下：

```
@ExceptionHandler(value = {BindException.class, ValidationException.class, MethodArgumentNotValidException.class})
public ResponseEntity<ResultData<String>> handleValidatedException(Exception e) {
  ResultData<String> resp = null;

  if (e instanceof MethodArgumentNotValidException) {
    // BeanValidation exception
    MethodArgumentNotValidException ex = (MethodArgumentNotValidException) e;
    resp = ResultData.fail(HttpStatus.BAD_REQUEST.value(),
                           ex.getBindingResult().getAllErrors().stream()
                           .map(ObjectError::getDefaultMessage)
                           .collect(Collectors.joining("; "))
                          );
  } else if (e instanceof ConstraintViolationException) {
    // BeanValidation GET simple param
    ConstraintViolationException ex = (ConstraintViolationException) e;
    resp = ResultData.fail(HttpStatus.BAD_REQUEST.value(),
                           ex.getConstraintViolations().stream()
                           .map(ConstraintViolation::getMessage)
                           .collect(Collectors.joining("; "))
                          );
  } else if (e instanceof BindException) {
    // BeanValidation GET object param
    BindException ex = (BindException) e;
    resp = ResultData.fail(HttpStatus.BAD_REQUEST.value(),
                           ex.getAllErrors().stream()
                           .map(ObjectError::getDefaultMessage)
                           .collect(Collectors.joining("; "))
                          );
  }

  return new ResponseEntity<>(resp,HttpStatus.BAD_REQUEST);
}
```

### 体验效果

```
POST http://localhost:8080/valid/test1
Content-Type: application/json

{
  "id": 1,
  "level": "12",
  "email": "47693899",
  "appId": "ab1c"
}
{
  "status": 400,
  "message": "名字为必填项; 不是一个合法的电子邮件地址; appId长度必须位于6到12之间",
  "data": null,
  "timestamp": 1628435116680
}
```

是不是感觉清爽多了？

## 自定义参数校验

虽然Spring Validation 提供的注解基本上够用，但是面对复杂的定义，我们还是需要自己定义相关注解来实现自动校验。

比如上面实体类中的sex性别属性，只允许前端传递传 M，F 这2个枚举值，如何实现呢？

### 第一步，创建自定义注解

```
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE})
@Retention(RUNTIME)
@Repeatable(EnumString.List.class)
@Documented
@Constraint(validatedBy = EnumStringValidator.class)//标明由哪个类执行校验逻辑
public @interface EnumString {
    String message() default "value not in enum values.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    /**
     * @return date must in this value array
     */
    String[] value();

    /**
     * Defines several {@link EnumString} annotations on the same element.
     *
     * @see EnumString
     */
    @Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE})
    @Retention(RUNTIME)
    @Documented
    @interface List {

        EnumString[] value();
    }
}
```

### 第二步，自定义校验逻辑

```
public class EnumStringValidator implements ConstraintValidator<EnumString, String> {
    private List<String> enumStringList;

    @Override
    public void initialize(EnumString constraintAnnotation) {
        enumStringList = Arrays.asList(constraintAnnotation.value());
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if(value == null){
            return true;
        }
        return enumStringList.contains(value);
    }
}
```

### 第三步，在字段上增加注解

```
@ApiModelProperty(value = "性别")
@EnumString(value = {"F","M"}, message="性别只允许为F或M")
private String sex;
```

### 第四步，体验效果

```
POST http://localhost:8080/valid/test2
Content-Type: application/x-www-form-urlencoded

id=1&name=javadaily&level=12&email=476938977@qq.com&appId=ab1cdddd&sex=N
{
  "status": 400,
  "message": "性别只允许为F或M",
  "data": null,
  "timestamp": 1628435243723
}
```

## 分组校验

一个VO对象在新增的时候某些字段为必填，在更新的时候又非必填。如上面的`ValidVO`中 id 和 appId 属性在新增操作时都是**非必填**，而在编辑操作时都为**必填**，name在新增操作时为**必填**，面对这种场景你会怎么处理呢？

在实际开发中我见到很多同学都是建立两个VO对象，`ValidCreateVO`，`ValidEditVO`来处理这种场景，这样确实也能实现效果，但是会造成类膨胀，而且极其容易被开发老鸟们嘲笑。

其实`Validator`校验框架已经考虑到了这种场景并且提供了解决方案，就是**分组校验**，只不过很多同学不知道而已。要使用分组校验，只需要三个步骤：

### 第一步：定义分组接口

```
public interface ValidGroup extends Default {
  
    interface Crud extends ValidGroup{
        interface Create extends Crud{

        }

        interface Update extends Crud{

        }

        interface Query extends Crud{

        }

        interface Delete extends Crud{

        }
    }
}
```

这里我们定义一个分组接口ValidGroup让其继承`javax.validation.groups.Default`，再在分组接口中定义出多个不同的操作类型，Create，Update，Query，Delete。至于为什么需要继承Default我们稍后再说。

### 第二步，在模型中给参数分配分组

```
@Data
@ApiModel(value = "参数校验类")
public class ValidVO {
    @ApiModelProperty("ID")
    @Null(groups = ValidGroup.Crud.Create.class)
    @NotNull(groups = ValidGroup.Crud.Update.class, message = "应用ID不能为空")
    private String id;

    @Null(groups = ValidGroup.Crud.Create.class)
    @NotNull(groups = ValidGroup.Crud.Update.class, message = "应用ID不能为空")
    @ApiModelProperty(value = "应用ID",example = "cloud")
    private String appId;

    @ApiModelProperty(value = "名字")
    @NotBlank(groups = ValidGroup.Crud.Create.class,message = "名字为必填项")
    private String name;
  
   @ApiModelProperty(value = "邮箱")
    @Email(message = "请填写正取的邮箱地址")
    privte String email;

    ...

}
```

给参数指定分组，对于未指定分组的则使用的是默认分组。

### 第三步，给需要参数校验的方法指定分组

```
@RestController
@Api("参数校验")
@Slf4j
@Validated
public class ValidController {

    @ApiOperation("新增")
    @PostMapping(value = "/valid/add")
    public String add(@Validated(value = ValidGroup.Crud.Create.class) ValidVO validVO){
        log.info("validEntity is {}", validVO);
        return "test3 valid success";
    }


    @ApiOperation("更新")
    @PostMapping(value = "/valid/update")
    public String update(@Validated(value = ValidGroup.Crud.Update.class) ValidVO validVO){
        log.info("validEntity is {}", validVO);
        return "test4 valid success";
    }
}
```

这里我们通过`value`属性给`add()`和`update()`方法分别指定Create和Update分组。

### 第四步，体验效果

```
POST http://localhost:8080/valid/add
Content-Type: application/x-www-form-urlencoded

name=javadaily&level=12&email=476938977@qq.com&sex=F
```

在Create时我们没有传递id和appId参数，校验通过。

当我们使用同样的参数调用update方法时则提示参数校验错误。

```
{
  "status": 400,
  "message": "ID不能为空; 应用ID不能为空",
  "data": null,
  "timestamp": 1628492514313
}
```

由于email属于默认分组，而我们的分组接口`ValidGroup`已经继承了`Default`分组，所以也是可以对email字段作参数校验的。如：

```
POST http://localhost:8080/valid/add
Content-Type: application/x-www-form-urlencoded

name=javadaily&level=12&email=476938977&sex=F
{
  "status": 400,
  "message": "请填写正取的邮箱地址",
  "data": null,
  "timestamp": 1628492637305
}
```

当然如果你的ValidGroup没有继承Default分组，那在代码属性上就需要加上`@Validated(value = {ValidGroup.Crud.Create.class, Default.class}`才能让`email`字段的校验生效。

## 业务规则校验

业务规则校验指接口需要满足某些特定的业务规则，举个例子：业务系统的用户需要保证其唯一性，用户属性不能与其他用户产生冲突，不允许与数据库中任何已有用户的用户名称、手机号码、邮箱产生重复。

这就要求在**创建用户时需要校验用户名称、手机号码、邮箱是否被注册**；**编辑用户时不能将信息修改成已有用户的属性**。

95%的程序员当面对这种业务规则校验时往往选择写在service逻辑中，常见的代码逻辑如下：

```
public void create(User user) {
    Account account = accountDao.queryByUserNameOrPhoneOrEmail(user.getName(),user.getPhone(),user.getEmail());
    if (account != null) {
        throw new IllegalArgumentException("用户已存在，请重新输入");
    }
}
```

**最优雅的实现方法应该是参考 Bean Validation 的标准方式，借助自定义校验注解完成业务规则校验。**

接下来我们通过上面提到的用户接口案例，通过自定义注解完成业务规则校验。

## 代码实战

需求很容易理解，注册新用户时，应约束不与任何已有用户的关键信息重复；而修改自己的信息时，只能与自己的信息重复，不允许修改成已有用户的信息。

这些约束规则不仅仅为这两个方法服务，它们可能会在用户资源中的其他入口被使用到，乃至在其他分层的代码中被使用到，在 Bean 上做校验就能全部覆盖上述这些使用场景。

### 自定义注解

首先我们需要创建两个自定义注解，用于业务规则校验：

- `UniqueUser`:表示一个用户是唯一的，唯一性包含：用户名，手机号码、邮箱

```
@Documented
@Retention(RUNTIME)
@Target({FIELD, METHOD, PARAMETER, TYPE})
@Constraint(validatedBy = UserValidation.UniqueUserValidator.class)
public @interface UniqueUser {

    String message() default "用户名、手机号码、邮箱不允许与现存用户重复";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

- `NotConflictUser`:表示一个用户的信息是无冲突的，无冲突是指该用户的敏感信息与其他用户不重合

```
@Documented
@Retention(RUNTIME)
@Target({FIELD, METHOD, PARAMETER, TYPE})
@Constraint(validatedBy = UserValidation.NotConflictUserValidator.class)
public @interface NotConflictUser {
    String message() default "用户名称、邮箱、手机号码与现存用户产生重复";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

### 实现业务校验规则

想让自定义验证注解生效，需要实现 `ConstraintValidator` 接口。接口的第一个参数是 **自定义注解类型**，第二个参数是 **被注解字段的类**，因为需要校验多个参数，我们直接传入用户对象。需要提到的一点是 `ConstraintValidator` 接口的实现类无需添加 `@Component` 它在启动的时候就已经被加载到容器中了。

```
@Slf4j
public class UserValidation<T extends Annotation> implements ConstraintValidator<T, User> {

    protected Predicate<User> predicate = c -> true;

    @Resource
    protected UserRepository userRepository;

    @Override
    public boolean isValid(User user, ConstraintValidatorContext constraintValidatorContext) {
        return userRepository == null || predicate.test(user);
    }

    /**
     * 校验用户是否唯一
     * 即判断数据库是否存在当前新用户的信息，如用户名，手机，邮箱
     */
    public static class UniqueUserValidator extends UserValidation<UniqueUser>{
        @Override
        public void initialize(UniqueUser uniqueUser) {
            predicate = c -> !userRepository.existsByUserNameOrEmailOrTelphone(c.getUserName(),c.getEmail(),c.getTelphone());
        }
    }

    /**
     * 校验是否与其他用户冲突
     * 将用户名、邮件、电话改成与现有完全不重复的，或者只与自己重复的，就不算冲突
     */
    public static class NotConflictUserValidator extends UserValidation<NotConflictUser>{
        @Override
        public void initialize(NotConflictUser notConflictUser) {
            predicate = c -> {
                log.info("user detail is {}",c);
                Collection<User> collection = userRepository.findByUserNameOrEmailOrTelphone(c.getUserName(), c.getEmail(), c.getTelphone());
                // 将用户名、邮件、电话改成与现有完全不重复的，或者只与自己重复的，就不算冲突
                return collection.isEmpty() || (collection.size() == 1 && collection.iterator().next().getId().equals(c.getId()));
            };
        }
    }

}
```

这里使用Predicate函数式接口对业务规则进行判断。

### 使用

```
@RestController
@RequestMapping("/senior/user")
@Slf4j
@Validated
public class UserController {
    @Autowired
    private UserRepository userRepository;
    

    @PostMapping
    public User createUser(@UniqueUser @Valid User user){
        User savedUser = userRepository.save(user);
        log.info("save user id is {}",savedUser.getId());
        return savedUser;
    }

    @SneakyThrows
    @PutMapping
    public User updateUser(@NotConflictUser @Valid @RequestBody User user){
        User editUser = userRepository.save(user);
        log.info("update user is {}",editUser);
        return editUser;
    }
}
```

使用很简单，只需要在方法上加入自定义注解即可，业务逻辑中不需要添加任何业务规则的代码。

### 测试

调用接口后出现如下错误，说明业务规则校验生效。

```
{
  "status": 400,
  "message": "用户名、手机号码、邮箱不允许与现存用户重复",
  "data": null,
  "timestamp": 1644309081037
}
```

### 小结

通过上面几步操作，业务校验便和业务逻辑就完全分离开来，在需要校验时用`@Validated`注解自动触发，或者通过代码手动触发执行，可根据你们项目的要求，将这些注解应用于控制器、服务层、持久层等任何层次的代码之中。

这种方式比任何业务规则校验的方法都优雅，推荐大家在项目中使用。在开发时可以将不带业务含义的格式校验注解放到 Bean 的类定义之上，将带业务逻辑的校验放到 Bean 的类定义的外面。**这两者的区别是放在类定义中的注解能够自动运行，而放到类外面则需要像前面代码那样，明确标出注解时才会运行。**