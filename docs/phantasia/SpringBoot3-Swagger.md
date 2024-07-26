SpringBoot3.x使用Swagger
======================

*   当前开发主流是前后端分离，有完整文档可以使团队配合更加流畅
*   Spring生态中通常使用springfox，但是当前springfox并不支持SpringBoot3.x版本
*   使用替代产品：[Springdoc.org](https://links.jianshu.com/go?to=https%3A%2F%2Fspringdoc.org%2F)
    *   Springdoc在v1.7.0版本之后不支持SpringBoot2.x和1.x！！！
*   项目启动后，Swagger默认地址：[http://localhost:8080/swagger-ui/index.html](https://links.jianshu.com/go?to=http%3A%2F%2Flocalhost%3A8080%2Fswagger-ui%2Findex.html)
*   版本：
    *   Java: 17.0.7
    *   SpringBoot: 3.1.5
    *   springdoc: 2.2.0
*   [同步发布BiliBili视频](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1nQ4y1877C%2F)

设置
==

*   Springdoc同时支持WebMvc和WebFlux
*   因为没用WebFlux实践过，所以下面只介绍WebMvc中的应用

依赖(pom.xml)
-----------

*   实际开发中，通常会和校验搭配使用

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.2.0</version>
</dependency>
<!--  搭配校验使用，使用与SpringBoot相同的版本号  -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
    <version>3.1.5</version>
</dependency>
```

*   示例中使用的其他依赖

```xml
<!--  SpringBoot  -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.1.5</version>
</dependency>
<!-- 让响应结果更美观 -->
<dependency>
    <groupId>com.alibaba.cola</groupId>
    <artifactId>cola-component-dto</artifactId>
    <version>4.3.2</version>
</dependency>
<!--  lombok  -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.26</version>
    <optional>true</optional>
</dependency>
<!--  日志  -->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.4.11</version>
</dependency>
```

环境配置(application.yml)
---------------------

*   开发环境
*   开发环境通常会开启Swagger文档，方便前端查阅文档
*   如果使用微服务，为避免Swagger地址冲突，通常会加上前缀
    *   如鉴权服务: "/auth-service/v3/api-docs"和"/auth-service/swagger-ui/index.html"
    *   如用户服务: "/user-service/v2/api-docs"和"/user-service/swagger-ui/index.html"

```yml
springdoc:
  api-docs:
    enabled: true # 开启OpenApi接口
    path: /user-service/v3/api-docs  # 自定义路径，默认为 "/v3/api-docs"
  swagger-ui:
    enabled: true # 开启swagger界面，依赖OpenApi，需要OpenApi同时开启
    path: /user-service/swagger-ui/index.html # 自定义路径，默认为"/swagger-ui/index.html"
```

*   生产环境
*   切记生产环境要关闭文档

```yml
springdoc:
  api-docs:
    enabled: false # 关闭OpenApi接口
  swagger-ui:
    enabled: false # 关闭swagger界面
```

配置
--

*   在SwaggerUI中增加描述
*   项目中新建"SwaggerConfig.java"文件

```java
@Configuration
public class SwaggerConfig {
    @Bean
    public OpenAPI swaggerOpenApi() {
        return new OpenAPI()
                .info(new Info().title("XXX平台YYY微服务")
                        .description("描述平台多牛逼")
                        .version("v1.0.0"))
                .externalDocs(new ExternalDocumentation()
                        .description("设计文档")
                        .url("https://juejin.cn/user/254742430749736/posts"));
    }
}
```

*   全局异常处理
*   参数校验错误返回提示信息
*   捕捉Exception大类，后面示例中补充更详细的类
*   响应结果Response为"cola-component-dto"中的一个包装类

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(Exception.class)
    public Response handleException(Exception e) {
        log.warn("未知异常", e);
        return Response.buildFailure("未知异常", e.getMessage());
    }
}
```

Restful接口
=========

Post
----

*   Post用于新增
*   通常使用body传递参数

### 用户类

*   新增用户模型，说明
    *   @Data: Lombok的写法，可以省略get/set
    *   @Schema: Swagger文档的注解，用于说明类/字段
        *   title: 类/字段说明
        *   example: 示例，Swagger中会将这个字段作为示例
        *   minLength/maxLength: 最小/最大长度，字段为String类型时生效(仅用于文档说明，不会抛出异常)
        *   minimum/maximum: 最小/最大值，字段为数字时有效(仅用于文档说明，不会抛出异常)
    *   @NotBlank: 校验不能为空(String生效)，为空或空字符则抛出异常。文档将该字段解析为必填项
    *   @NotNull: 校验不能为空(包装类生效，如Integer/Long/Boolean)，为空将抛出异常。文档将该字段解析为必填项
    *   @Range: 检查数值范围，不符合将抛出异常

```java
@Data
@Schema(title = "新增用户模型")
public class UserAddCO {
    @Schema(title = "名字", example = "老王", minLength = 1, maxLength = 5)
    @NotBlank(message = "名字不能为空")
    private String name;

    @Schema(title = "年龄", example = "18", minimum = "0", maximum = "150")
    @NotNull(message = "年龄不能为空")
    @Range(min = 0, max = 150, message = "年龄在0~150之间")
    private Integer age;
//    private int age;
//    既然不能为空，为什么不使用int？
//    1. 因为int是基本类型不会为空，所以@NotNull校验无效
//    2. 类初始化时生成默认值(int默认为0)，@Range中最小值包含0，所以没有age参数校验通过，不符合预期

    @Schema(title = "电话（可选）")
    private String phone;
}
```

### 请求方法

*   @Tag: 控制器说明
    *   name: 名称
    *   description: 描述说明
*   @PostMapping: 使用post方法，一般用于新增记录
*   @Operation: 请求说明
    *   summary: 说明，Swagger页面在方法后面，不会被折叠
    *   descirption: 描述，会被折叠到方法说明中
*   @Validated: 校验数据，有了这个注解模型中的@NotBlank@NotNull@Range才会生效
*   @RequestBody: 从请求body中读取数据

```java
@RestController
@RequestMapping("/demo")
@Tag(name = "示例控制器", description = "演示Restful接口")
public class DemoController {
    @PostMapping("/")
    @Operation(summary = "Post方法示例", description = "Post通常用于新增")
    public SingleResponse<Long> add(@Validated @RequestBody UserAddCO user) {
        // TODO:添加到数据库中，然后返回记录id
        return SingleResponse.of(1L);
    }
}
```

### 异常处理

*   BindException: 如@NotBlank@NotNull@Range等校验失败时会报该异常
*   HttpMessageConversionException: 转换失败时，如age参数传入字符串会报该异常

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(BindException.class)
    public Response handleBindException(BindException e) {
        // 拼接错误信息，用于多个校验不通过的错误信息拼接
        List<ObjectError> allErrors = e.getBindingResult().getAllErrors();
        String message = allErrors.stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(Collectors.joining(";"));
        log.info("参数校验不通过：{}", message);
        return Response.buildFailure("参数校验不通过", message);
    }

    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(HttpMessageConversionException.class)
    public Response handleHttpMessageConversionException(HttpMessageConversionException e) {
        log.info("参数转换失败：{}", e.getMessage());
        return Response.buildFailure("参数转换失败", e.getMessage());
    }
}
```

Post(FormData)
--------------

*   上传通常使用FormData上传数据
*   @PostMapping: 上传使用Post
    *   consumes: 这个值一定要设置成"MediaType.MULTIPART\_FORM\_DATA\_VALUE"，否则Swagger将错误识别为json格式（file字段错误识别为string），而不是FormData
*   @RequestPart: 比较少用到，就是用于FormData
*   MultipartFile: 接收文件的对象，可以将流保存到本地

```java
    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @Operation(summary = "上传文件示例")
    public SingleResponse<String> upload(@RequestPart("file") MultipartFile file) {
        log.info("接收到文件：{}, 大小：{}", file.getOriginalFilename(), file.getSize());
        try {
            // 保存到本地
            File localTempFile = new File(file.getOriginalFilename());
            localTempFile.createNewFile();
            file.transferTo(localTempFile);
            return SingleResponse.of(localTempFile.getAbsolutePath());
        } catch (IOException e) {
            throw new RuntimeException("文件上传失败");
        }
    }

    @PostMapping(value = "/upload/multi", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @Operation(summary = "上传多个文件示例")
    public MultiResponse<String> uploadMulti(@RequestPart("files") MultipartFile[] files) {
        List<String> result = new LinkedList<>();
        for (MultipartFile file : files) {
            log.info("接收到文件：{}, 大小：{}", file.getOriginalFilename(), file.getSize());
            result.add(file.getOriginalFilename());
            // TODO:如上保存到本地
        }
        return MultiResponse.of(result);
    }
```

Get(分页)
-------

*   Get常用于获取分页数据

### 用户详情类(响应结果)

*   用户详情
*   @Schema: 字段/类的Swagger描述注解
    *   title: 字段/类说明，实际开发中，这个是必填，其他为可选
    *   如example/minLength/maxLength/minimum/maximum等为可选项，部分HTTP工具可以利用这些描述生成Mock数据

```java
@Data
@Schema(title = "用户详情")
public class UserDetailCO {
    @Schema(title = "用户id", minimum = "1")
    private long id;

    @Schema(title = "用户名", minLength = 1, maxLength = 5, example = "言午日尧耳总")
    private String name;

    @Schema(title = "手机")
    private String phone;
}
```

### 请求分页类

*   @Parameter: QueryString的参数定义
    *   required: 是否必填
    *   description: 描述
    *   example: 示例值，部分HTTP工具请求时会使用这个当做默认值(开发自测调试，不用每次写参数)

```java
@Data
public class UserPageQry {
    @Parameter(required = true, description = "页码，从1开始", example = "1")
    @Min(value = 1, message = "pageIndex必须大于1")
    private int pageIndex;
//    @NotNull(message="pageIndex参数必须填写")
//    private Integer pageIndex;
//    1. 此处可以使用int类型，因为限制最小值为1，不填写默认赋值0，@Min校验不通过会抛出异常
//    2. 也可以写成Integer+@NotNull的组合(如下pageSize)，更加语义化

    @Parameter(required = true, description = "页面大小", example = "20")
    @NotNull(message = "pageSize参数必须填写")
    @Range(min = 1, max = 100, message = "pageSize必须在1-100之间")
    private Integer pageSize;

    @Parameter(description = "搜索名字(模糊搜索)，不搜索就传null或空字符")
    private String phone;
}
```

### 请求方法1(推荐)

*   Operation: 请求描述
*   @Validated: 启用校验
*   @ParameterObject: 参数对象，这个注解可以将对象字段解析为QueryString参数
*   PageResponse: 分页响应结果，Swagger中解析响应字段

```java
    @GetMapping("/")
    @Operation(summary = "获取分页列表")
    public PageResponse<UserDetailCO> getPageList(@Validated @ParameterObject UserPageQry qry) {
        log.info("{}", qry);
        List<UserDetailCO> result = List.of(new UserDetailCO());
        return PageResponse.of(result, 1, qry.getPageSize(), qry.getPageIndex());
    }
```

### 请求方法2(不推荐，没有方法1优雅)

*   字段单独设置
*   @Validated: 控制器上的@Validated必须设置，否则校验注解无效
*   @Operation: 方法说明
*   @Parameter: 对每一个字段进行描述
    *   name: 与字段名一一对应，才能正确解析
    *   description: 描述
    *   example: 示例值，部分HTTP工具请求时会使用这个当做默认值(开发自测调试，不用每次写参数)

```java
@Slf4j
@Validated // 控制器加上改注解到才能进行校验
@RestController
@RequestMapping("/demo")
@Tag(name = "示例控制器", description = "演示Restful接口")
public class DemoController {
    @GetMapping("/other/")
    @Operation(summary = "另一种获取分页列表方式")
    @Parameter(name = "pageIndex", description = "页码，从1开始", example = "1")
    @Parameter(name = "pageSize", description = "数量，不小于1", example = "20")
    @Parameter(name = "name", description = "搜索名字(模糊搜索)，不搜索就传null或空字符")
    public PageResponse<UserDetailCO> getPage(@NotNull(message = "pageIndex不能为空") @Min(value = 1, message = "pageIndex必须大于1") Integer pageIndex,
                                              @NotNull(message = "pageIndex不能为空") @Range(min = 1, max = 100, message = "pageSize必须在1-100之间") Integer pageSize,
                                              @Nullable String name) {
        log.info("{} {} {}", pageIndex, pageSize, name);
        List<UserDetailCO> result = List.of(new UserDetailCO());
        return PageResponse.of(result, 1, pageSize, pageIndex);
    }
}
```

*   异常处理

```java
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(ValidationException.class)
    public Response handleValidationException(ValidationException e) {
        log.info("参数验证失败: {}", e.getMessage());
        return Response.buildFailure("参数验证失败", e.getMessage());
    }
```

Get(一条记录详情)
-----------

*   @GetMapping: Restful风格，在请求路径中放id
*   @PathVariable: 读取路径中的id
    *   接收参数可以使用long这样的基本类型，id必定不为空(如果为空，路径错误不会进入该方法)

```java
    @GetMapping("/{id}/")
    @Operation(summary = "获取用户详情")
    public SingleResponse<UserDetailCO> getUser(@PathVariable("id") long id) {
        log.info("{}", id);
        return SingleResponse.of(new UserDetailCO());
    }
```

Put/Patch
---------

*   区别
    *   Put: 修改对象的所有参数
    *   Patch: 修改对象的部分参数
*   id从路径中读取，遵循Restful风格
*   值从body中读取
    *   body接收对象UserPutCO包含User的所有字段，UserPatchCO只包含一部分字段

```java
    @PutMapping("/{id}/")
    @Operation(summary = "修改参数")
    public Response putUser(@PathVariable("id") long id, @Validated @RequestBody UserPutCO user) {
        // TODO:使用id在数据查找到用户，然后修改值并保存
        return Response.buildSuccess();
    }

    @PatchMapping("/{id}/")
    @Operation(summary = "修改部分参数")
    public Response patchUser(@PathVariable("id") long id, @Validated @RequestBody UserPatchCO user) {
        // TODO:使用id在数据查找到用户，然后修改值并保存
        return Response.buildSuccess();
    }
```

Delete
------

```java
    @DeleteMapping("/{id}/")
    @Operation(summary = "删除")
    public Response deleteUser(@PathVariable("id") long id) {
        // TODO:根据id直接删除用户
        return Response.buildSuccess();
    }
```

其他
==

屏蔽过滤器
-----

*   用于前后端分离项目
*   有时候为了鉴权方便，会使用全局过滤器过滤权限
*   将Swagger相关接口排除掉

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {

    @Bean
    public JwtInterceptor jwtInterceptor() {
        return new JwtInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 除了swagger和登录，其他全部拦截验证jwt
        registry.addInterceptor(jwtInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns(
                        "/**/*.html",
                        "/**/*.js",
                        "/**/*.css",
                        "/**/*.woff",
                        "/**/*.ttf",
                        "/**/*.js",
                        "/**/*.map",
                        "/**/*.png",
                        "/v3/api-docs", // 如果配置里改了，这里也记得修改
                        "/v3/api-docs/swagger-config",
                        "/auth/login"); // 登录接口
    }
}
```

不同风格注解
------

*   将所有配置写在控制器中

```java
@Tag(name = "User", description = "用户管理")
@RestController
@RequestMapping("/user")
public class UserController {
    // 这个样式看起来更整齐
    @Operation(summary = "获取用户信息", description = "用户详情")
    @Parameter(name = "id", description = "用户id")
    @ApiResponse(responseCode = "200", description = "用户信息")
    @GetMapping("/{id}")
    public Resp<?> detail(@PathVariable("id") Integer id) {
        User user = new User();
        return new Resp<>(user);
    }

    @Operation(summary = "新增", description = "新增用户")
    @io.swagger.v3.oas.annotations.parameters.RequestBody(description = "用户")
    @ApiResponse(responseCode = "201", description = "成功")
    @PostMapping("/")
    public Resp<User> add(@RequestBody User user) {
        return new Resp<>(user);
    }

    // 写到一起的示例
    @Operation(summary = "获取列表",
            description = "获取用户列表",
            parameters = {@Parameter(name = "page", description = "页码"),
                    @Parameter(name = "size", description = "每页数量")},
            responses = {@ApiResponse(responseCode = "200", description = "用户列表")})
    @GetMapping("/")
    public Resp<List<User>> list(@PathParam("page") Integer page, @PathParam("size") Integer size) {
        return new Resp<>(new ArrayList<>());
    }
}
```

使用Apifox自测
==========

*   HTTP请求工具可以读取OpenApi(Swagger)生成HTTP请求
*   以Apifox为例
    *   打开Apifox，新建项目
    *   项目设置 > 数据管理 > 导入数据 > URL方式导入
    *   填入：[http://localhost:8080/v3/api-docs](https://links.jianshu.com/go?to=http%3A%2F%2Flocalhost%3A8080%2Fv3%2Fapi-docs)
    *   提交，即可直接导入API接口
    *   就会根据Swagger生成HTTP文档及请求

完整示例代码
======

*   Swagger设置

```java
@Configuration
public class SwaggerConfig {
    @Bean
    public OpenAPI swaggerOpenApi() {
        return new OpenAPI()
                .info(new Info().title("XXX平台YYY服务")
                        .description("描述平台多牛逼")
                        .version("v0.0.1"))
                .externalDocs(new ExternalDocumentation()
                        .description("设计文档")
                        .url("https://juejin.cn/user/254742430749736/posts"));
    }
}
```

*   异常处理

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(Exception.class)
    public Response handleException(Exception e) {
        log.warn("未知异常", e);
        return Response.buildFailure("未知异常", e.getMessage());
    }

    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(BindException.class)
    public Response handleBindException(BindException e) {
        // 拼接错误信息，用于多个校验不通过的错误信息拼接
        List<ObjectError> allErrors = e.getBindingResult().getAllErrors();
        String message = allErrors.stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(Collectors.joining(";"));
        log.info("参数校验不通过：{}", message);
        return Response.buildFailure("参数校验不通过", message);
    }

    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(HttpMessageConversionException.class)
    public Response handleHttpMessageConversionException(HttpMessageConversionException e) {
        log.info("参数转换失败：{}", e.getMessage());
        return Response.buildFailure("参数转换失败", e.getMessage());
    }

    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(ValidationException.class)
    public Response handleValidationException(ValidationException e) {
        log.info("参数验证失败: {}", e.getMessage());
        return Response.buildFailure("参数验证失败", e.getMessage());
    }
}
```

*   控制器

```java
@Slf4j
@Validated
@RestController
@RequestMapping("/demo")
@Tag(name = "示例控制器", description = "演示Restful接口")
public class DemoController {
    @PostMapping("/")
    @Operation(summary = "Post方法示例", description = "Post通常用于新增")
    public SingleResponse<Long> add(@Validated @RequestBody UserAddCO user) {
        // TODO:添加到数据库中，然后返回记录id
        return SingleResponse.of(1L);
    }

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @Operation(summary = "上传文件示例")
    public SingleResponse<String> upload(@RequestPart("file") MultipartFile file) {
        log.info("接收到文件：{}, 大小：{}", file.getOriginalFilename(), file.getSize());
        try {
            // 保存到本地
            File localTempFile = new File(file.getOriginalFilename());
            localTempFile.createNewFile();
            file.transferTo(localTempFile);
            return SingleResponse.of(localTempFile.getAbsolutePath());
        } catch (IOException e) {
            throw new RuntimeException("文件上传失败");
        }
    }

    @PostMapping(value = "/upload/multi", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @Operation(summary = "上传多个文件示例")
    public MultiResponse<String> uploadMulti(@RequestPart("files") MultipartFile[] files) {
        List<String> result = new LinkedList<>();
        for (MultipartFile file : files) {
            log.info("接收到文件：{}, 大小：{}", file.getOriginalFilename(), file.getSize());
            result.add(file.getOriginalFilename());
            // TODO:如上保存到本地
        }
        return MultiResponse.of(result);
    }

    @GetMapping("/")
    @Operation(summary = "获取分页列表")
    public PageResponse<UserDetailCO> getPageList(@Validated @ParameterObject UserPageQry qry) {
        log.info("{}", qry);
        List<UserDetailCO> result = List.of(new UserDetailCO());
        return PageResponse.of(result, 1, qry.getPageSize(), qry.getPageIndex());
    }

    @GetMapping("/other/")
    @Operation(summary = "另一种获取分页列表方式")
    @Parameter(name = "pageIndex", description = "页码，从1开始", example = "1")
    @Parameter(name = "pageSize", description = "数量，不小于1", example = "20")
    @Parameter(name = "name", description = "搜索名字(模糊搜索)，不搜索就传null或空字符")
    public PageResponse<UserDetailCO> getPage(@NotNull(message = "pageIndex不能为空") @Min(value = 1, message = "pageIndex必须大于1") Integer pageIndex,
                                              @NotNull(message = "pageIndex不能为空") @Range(min = 1, max = 100, message = "pageSize必须在1-100之间") Integer pageSize,
                                              @Nullable String name) {
        log.info("{} {} {}", pageIndex, pageSize, name);
        List<UserDetailCO> result = List.of(new UserDetailCO());
        return PageResponse.of(result, 1, pageSize, pageIndex);
    }

    @GetMapping("/{id}/")
    @Operation(summary = "获取用户详情")
    public SingleResponse<UserDetailCO> getUser(@PathVariable("id") long id) {
        // TODO:从数据库查询
        return SingleResponse.of(new UserDetailCO());
    }

    @PutMapping("/{id}/")
    @Operation(summary = "修改参数")
    public Response putUser(@PathVariable("id") long id, @Validated @RequestBody UserPutCO user) {
        // TODO:使用id在数据查找到用户，然后修改值并保存
        return Response.buildSuccess();
    }

    @PatchMapping("/{id}/")
    @Operation(summary = "修改部分参数")
    public Response patchUser(@PathVariable("id") long id, @Validated @RequestBody UserPatchCO user) {
        // TODO:使用id在数据查找到用户，然后修改值并保存
        return Response.buildSuccess();
    }

    @DeleteMapping("/{id}/")
    @Operation(summary = "删除")
    public Response deleteUser(@PathVariable("id") long id) {
        // TODO:根据id直接删除用户
        return Response.buildSuccess();
    }
}
```

*   用户新增

```java
@Data
@Schema(title = "新增用户模型")
public class UserAddCO {
    @Schema(title = "名字", example = "老王", minLength = 1, maxLength = 5)
    @NotBlank(message = "名字不能为空")
    private String name;

    @Schema(title = "年龄", example = "18", minimum = "0", maximum = "150")
    @NotNull(message = "年龄不能为空")
    @Range(min = 0, max = 150, message = "年龄在0~150之间")
    private Integer age;
//    private int age;
//    既然不能为空，为什么不使用int？
//    1. 因为int是基本类型不会为空，所以@NotNull校验无效
//    2. 类初始化时生成默认值(int默认为0)，@Range中最小值包含0，所以没有age参数校验通过，不符合预期

    @Schema(title = "电话（可选）")
    private String phone;
}
```

*   用户详情

```java
@Data
@Schema(title = "用户详情")
public class UserDetailCO {
    @Schema(title = "用户id", minimum = "1")
    private long id;

    @Schema(title = "用户名", minLength = 1, maxLength = 5, example = "言午日尧耳总")
    private String name;

    @Schema(title = "手机")
    private String phone;
}
```

*   分页请求

```java
@Data
public class UserPageQry {
    @Parameter(required = true, description = "页码，从1开始", example = "1")
    @Min(value = 1, message = "pageIndex必须大于1")
    private int pageIndex;
//    @NotNull(message="pageIndex参数必须填写")
//    private Integer pageIndex;
//    1. 此处可以使用int类型，因为限制最小值为1，不填写默认赋值0，@Min校验不通过会抛出异常
//    2. 也可以写成Integer+@NotNull的组合(如下pageSize)，更加语义化

    @Parameter(required = true, description = "页面大小", example = "20")
    @NotNull(message = "pageSize参数必须填写")
    @Range(min = 1, max = 100, message = "pageSize必须在1-100之间")
    private Integer pageSize;

    @Parameter(description = "搜索名字(模糊搜索)，不搜索就传null或空字符")
    private String phone;
}
```

*   Patch修改用户

```java
@Data
@Schema(title = "新增用户模型")
public class UserPatchCO {
    @Schema(title = "电话（可选）")
    private String phone;
}
```

*   Put修改用户

```java
@Data
@Schema(title = "新增用户模型")
public class UserPutCO {
    @Schema(title = "名字", example = "老王", minLength = 1, maxLength = 5)
    @NotBlank(message = "名字不能为空")
    private String name;

    @Schema(title = "年龄", example = "18", minimum = "0", maximum = "150")
    @NotNull(message = "年龄不能为空")
    @Range(min = 0, max = 150, message = "年龄在0~150之间")
    private Integer age;

    @Schema(title = "电话（可选）")
    private String phone;
}
```

本文转自 <https://www.jianshu.com/p/52c889098e6b?v=1699760710283>，如有侵权，请联系删除。