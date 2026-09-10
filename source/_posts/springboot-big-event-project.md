---
title: Spring Boot 项目实战复盘：big-event 文章管理系统（JWT + Redis + MyBatis + MinIO）
date: 2026-09-10 16:30:00
description: 学完 Java 和 Spring Boot 基础后完成的第一个完整后端项目：用户注册登录（MD5 + JWT + Redis 双重校验）、文章分类与文章管理（分组校验、自定义校验注解、PageHelper 分页 + MyBatis 动态 SQL）、MinIO 对象存储文件上传。本文以「一次请求的生命周期」为主线，把拦截器、ThreadLocal、全局异常处理、统一响应这些散落的知识点串成一条完整链路，最后复盘项目留下的坑与下一步的改进方向。
categories:
  - [后端, Spring Boot]
tags:
  - Java
  - Spring Boot
  - JWT
  - Redis
  - MyBatis
  - MinIO
---

学 Spring Boot 的时候，知识点是一个一个学的：依赖注入、拦截器、注解校验、MyBatis……每个都能看懂，但始终没回答一个问题——**它们在真实项目里是怎么协同工作的**。big-event 是我学完 Java 后按课程完成的第一个完整后端项目，一个文章管理系统，从建表到接口全部跑通。

> **单个知识点和完整项目之间的差距，在于「横切关注点」怎么组织**：认证放在拦截器里统一做，而不是每个接口手写一遍；当前用户身份用 ThreadLocal 传递，而不是在参数里层层透传；异常和响应用全局处理器统一格式，而不是各接口自理。这个项目把这四件事各做了一遍，本文就以「一次请求的生命周期」为主线来复盘。

```text
                    big-event 文章管理系统
┌─────────────────────────────────────────────────────┐
│  用户模块      分类模块      文章模块      文件上传     │
│  注册/登录     CRUD         CRUD+分页    本地/MinIO   │
├─────────────────────────────────────────────────────┤
│  横切关注点：拦截器认证 / ThreadLocal / 全局异常 /    │
│              统一响应 Result / Bean Validation       │
├─────────────────────────────────────────────────────┤
│  MyBatis（注解 + XML 动态 SQL）   JWT + Redis         │
├─────────────────────────────────────────────────────┤
│  MySQL                Redis              MinIO      │
└─────────────────────────────────────────────────────┘
```

<!-- more -->

## 一、项目概览

big-event 是一个三模块的 REST 接口服务：用户（注册、登录、个人信息、头像、改密）、文章分类（CRUD）、文章（CRUD + 条件分页），外加文件上传。技术栈如下：

| 技术 | 版本 | 用途 |
| --- | --- | --- |
| Spring Boot | 4.1.0 | 基础框架，Java 17 |
| MyBatis | 4.0.1 starter | ORM，注解与 XML 混用 |
| MySQL | - | 三张表：user / category / article |
| java-jwt | 4.4.0 | 生成 / 解析 token |
| Redis | spring-data-redis | token 白名单，实现服务端主动失效 |
| PageHelper | 4.1.1 | 物理分页 |
| MinIO | 8.5.17 | 对象存储，存头像等文件 |
| springdoc-openapi | 3.0.0 | Swagger 接口文档 |
| Lombok + Validation | - | 简化 POJO / 参数校验 |

包结构是经典的三层架构：

```text
com.big_event
├── contorller/     # 接口层：参数校验、调用 service、返回 Result
├── service/impl/   # 业务层：补时间戳、取当前用户、组织查询条件
├── mapper/         # 持久层：MyBatis 注解 + 一个 XML（动态 SQL）
├── pojo/           # User / Category / Article / Result / PageBean
├── interceptors/   # LoginInterceptor：token 双重校验
├── config/         # WebConfig / MinioConfig / SwaggerConfig
├── anno/           # @State 自定义校验注解、@AutoFill（未完成）
├── validation/     # StateValidation：注解的校验器实现
├── exception/      # GlobalExceptionHandler
└── utils/          # JwtUtil / Md5Util / ThreadLocalUtil
```

## 二、一次请求的生命周期

**先给结论，再给论证**：这个项目里最值得复盘的不是某个接口，而是每个请求都要走的这条固定链路。

以「查询我的文章列表」为例，一个带着 `Authorization` 头的请求进来之后：

```text
HTTP 请求（携带 Authorization: <token>）
   │
   ▼
LoginInterceptor.preHandle
   ├── Redis 里还有这个 token 吗？── 没有 ──► 401，请求结束
   └── JwtUtil.parseToken 校验签名/有效期 ──► claims 存入 ThreadLocal
   │
   ▼
Controller（@Validated 参数校验，自定义 @State 注解）
   │
   ▼
Service（业务逻辑，ThreadLocalUtil.get() 拿当前用户 id）
   │
   ▼
Mapper（MyBatis：注解 SQL / XML 动态 SQL）──► MySQL
   │
   ▼
正常：Result.success(data)
异常：GlobalExceptionHandler 兜底 ──► Result.error(msg)
   │
   ▼
afterCompletion ──► ThreadLocalUtil.remove()（清理线程上下文）
```

链路的最后一步是**统一响应**。所有接口都返回 `Result<T>`，前端只需要认一种结构：

```java
public class Result<T> {
    private Integer code;   // 200 成功，1 失败
    private String message;
    private T data;

    public static <E> Result<E> success(E data) {
        return new Result<E>(200, "操作成功", data);
    }
    public static <T> Result<T> error(String message) {
        return new Result<T>(1, message, null);
    }
}
```

配套的是**全局异常处理**。没有它之前，任何一次参数校验失败都会抛出 `MethodArgumentNotValidException`，前端收到的是一整个堆栈页；有了它，所有未被捕获的异常都收敛成统一 JSON：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        e.printStackTrace();
        return Result.error(StringUtils.hasLength(e.getMessage())
                ? e.getMessage() : "操作失败");
    }
}
```

拦截器本身在 `WebConfig` 里注册，登录、注册、文件上传和 Swagger 文档这些无需认证的路径显式排除：

```java
registry.addInterceptor(loginInterceptor).excludePathPatterns(
    "/user/login", "/user/register",
    "/minio/**", "/swagger-ui/**", "/swagger-ui.html",
    "/v3/api-docs/**", "/v3/api-docs"
);
```

## 三、登录认证：JWT + Redis 双重校验

**这是整个项目最核心的一条链路。** 单独用 JWT 有一个绕不过去的问题：token 一旦签发，在过期之前始终有效，服务端没有任何办法作废它——用户改了密码、被封号，旧 token 依然能畅通无阻。解法是把「token 是否有效」的裁决权收回服务端：**JWT 负责携带身份，Redis 负责决定生死**。

### 3.1 登录：签发 token 并登记到 Redis

密码校验通过后，两件事一起做：用 JWT 把 `id`、`username` 打包进 token（有效期 12 小时），同时把 token 存进 Redis（TTL 1 小时）：

```java
if (Md5Util.md5(password).equals(loginUser.getPassword())) {
    Map<String, Object> claims = new HashMap<>();
    claims.put("id", loginUser.getId());
    claims.put("username", loginUser.getUsername());
    String token = JwtUtil.genToken(claims);

    // 把 token 存储到 Redis 中，作为「当前有效」的白名单
    ValueOperations<String, String> operations = stringRedisTemplate.opsForValue();
    operations.set(token, token, Duration.ofHours(1));
    return Result.success(token);
}
```

JWT 的生成与解析工具很薄，HMAC256 签名 + 过期时间：

```java
public class JwtUtil {
    private static final String KEY = "itheima";

    public static String genToken(Map<String, Object> claims) {
        return JWT.create()
                .withClaim("claims", claims)
                .withExpiresAt(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 12))
                .sign(Algorithm.HMAC256(KEY));
    }

    public static Map<String, Object> parseToken(String token) {
        return JWT.require(Algorithm.HMAC256(KEY))
                .build().verify(token)
                .getClaim("claims").asMap();
    }
}
```

### 3.2 拦截器：先查 Redis，再解析 JWT

每个受保护的请求都要过这道闸。注意校验顺序：**先查 Redis 再解析 JWT**——Redis 里没有，说明服务端已经作废了这个 token，直接 401：

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        try {
            // 从 Redis 中获取 token，没有说明已失效（被作废或过期）
            ValueOperations<String, String> operations = stringRedisTemplate.opsForValue();
            String redisToken = operations.get(token);
            if (redisToken == null) {
                throw new RuntimeException();
            }

            Map<String, Object> claims = JwtUtil.parseToken(token);
            // 把业务数据存储到 ThreadLocal 中
            ThreadLocalUtil.set(claims);
            return true;
        } catch (Exception e) {
            response.setStatus(401);   // http 响应状态码 401
            return false;
        }
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        ThreadLocalUtil.remove();     // 清空，防止内存泄漏
    }
}
```

### 3.3 ThreadLocal：把用户身份「随身携带」

拦截器解析出的用户信息，Controller 和 Service 是怎么拿到的？答案是不用参数传，用 `ThreadLocal` 存取——同一个请求由同一个线程处理，线程内部全局可见：

```java
public class ThreadLocalUtil {
    private static final ThreadLocal THREAD_LOCAL = new ThreadLocal();

    public static <T> T get() {
        return (T) THREAD_LOCAL.get();
    }
    public static void set(Object value) {
        THREAD_LOCAL.set(value);
    }
    // 清除，防止内存泄漏
    public static void remove() {
        THREAD_LOCAL.remove();
    }
}
```

于是业务代码里取当前用户就一行：

```java
Map<String, Object> map = ThreadLocalUtil.get();
Integer userId = (Integer) map.get("id");
```

这里有一个**必须理解的坑**：Tomcat 处理请求用的是线程池，线程是复用的。Entry 的 key 是弱引用会被 GC，但 value 是强引用——如果不在请求结束时 `remove()`，上一个请求的用户信息会残留在线程里，轻则串号（A 看到 B 的数据），重则内存泄漏。所以 `afterCompletion` 里的 `ThreadLocalUtil.remove()` 不是可选的收尾，而是正确性的一部分。

### 3.4 改密码 = 强制下线

有了 Redis 这一层，「修改密码后强制重新登录」变得非常自然——把当前 token 从 Redis 删掉即可，下次请求在 3.2 的第一道检查就会被拒：

```java
// 2.调用 service 完成更新密码
userService.updatePwd(newPwd);

// 删除 Redis 中对应的 token
ValueOperations<String, String> operations = stringRedisTemplate.opsForValue();
operations.getOperations().delete(token);
```

纯 JWT 方案做不到这一点，这就是引入 Redis 的全部理由。

## 四、参数校验体系：从内置注解到自定义注解

项目里参数校验分三个层次，复杂度依次递进。

**第一层：内置注解直接标在参数上。** 注册接口的用户名密码规则，一条 `@Pattern` 搞定（配合类上的 `@Validated`），替代了之前手写的一长串 if：

```java
public Result register(@Pattern(regexp = "^\\S{5,16}$") String username,
                       @Pattern(regexp = "^\\S{5,16}$") String password) {
```

**第二层：分组校验，一个 POJO 两套规则。** 分类的「新增」不需要 id（数据库自增），「修改」必须带 id。为同一个 POJO 声明两个分组，让 `id` 只在 Update 组生效：

```java
@Data
public class Category {
    @NotNull(groups = Update.class)   // 只在修改时要求必填
    private Integer id;
    @NotEmpty
    private String categoryName;
    @NotEmpty
    private String categoryAlias;

    // 没指定 groups 的校验项默认属于 Default 分组
    // 分组之间可以继承：Add extends Default，则 Add 组拥有 Default 的全部校验项
    public interface Add extends Default {}
    public interface Update extends Default {}
}
```

Controller 里指定用哪套规则：`@Validated(Category.Add.class)` 新增、`@Validated(Category.Update.class)` 修改。

**第三层：自定义校验注解。** 文章的 `state` 字段只允许「已发布」或「草稿」，内置注解表达不了「枚举值集合」这种语义（`@Pattern` 写中文正则太绕），于是自己写一个 `@State`。自定义校验注解三个要素——注解本身、校验器、注册关联：

```java
// 注解：用 @Constraint 指定校验器
@Documented
@Target({FIELD})
@Retention(RUNTIME)
@Constraint(validatedBy = {StateValidation.class})
public @interface State {
    String message() default "state参数的值只能是发布或者草稿";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 校验器：实现 ConstraintValidator<注解类型, 被校验字段类型>
public class StateValidation implements ConstraintValidator<State, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return false;
        }
        return value.equals("发布") || value.equals("草稿");
    }
}
```

POJO 上的用法和内置注解完全一致，业务代码零感知：

```java
@State
private String state;
```

## 五、文章列表：PageHelper 分页 + MyBatis 动态 SQL

文章列表接口的要求是：**按分类、状态可选筛选，只看自己的文章，按更新时间倒序，分页返回**。两个组件配合完成。

### 5.1 PageHelper：一行开启物理分页

PageHelper 的用法非常「隐式」——在查询前调一行 `startPage`，它就会拦截**紧随其后的那一条** MyBatis 查询，自动改写成 `limit` 语句并附带一次 count 查询：

```java
public PageBean<Article> list(Integer pageNum, Integer pageSize,
                              Integer categoryId, String state) {
    PageBean<Article> pb = new PageBean<>();

    // 开启分页查询（只对下一条 SQL 生效）
    PageHelper.startPage(pageNum, pageSize);

    Map<String, Object> map = ThreadLocalUtil.get();
    Integer userId = (Integer) map.get("id");
    List<Article> as = articleMapper.list(userId, categoryId, state);

    // Page 中提供了获取总记录数和当前页数据的方法
    Page<Article> p = (Page<Article>) as;
    pb.setTotal(p.getTotal());
    pb.setItems(p.getResult());
    return pb;
}
```

返回值收敛到统一的分页 VO，前端不用关心 PageHelper 的存在：

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PageBean<T> {
    private Long total;      // 总条数
    private List<T> items;   // 当前页数据集合
}
```

### 5.2 动态 SQL：XML 登场

这个查询的筛选条件是可选的，SQL 得按参数动态拼接——这正是注解 SQL 的短板、XML 的主场：

```xml
<select id="list" resultType="com.big_event.pojo.Article">
    select id, title, content, cover_img, state, category_id,
           create_user, create_time, update_time
    from article
    <where>
        <if test="categoryId != null">
            category_id = #{categoryId}
        </if>
        <if test="state != null and state != ''">
            and state = #{state}
        </if>
        and create_user = #{userId}
    </where>
    order by update_time desc
</select>
```

`<where>` 标签会自动处理首个 `and` 的拼接问题；`create_user = #{userId}` 不在 `<if>` 里——**数据隔离是硬条件，不是筛选项**。项目里 UserMapper、CategoryMapper 全用注解（`@Select`/`@Insert`），只有这条多条件查询用了 XML，这个取舍是对的：简单 SQL 注解直观，动态 SQL 交给 XML。

另外一个小但重要的配置——数据库下划线命名到 Java 驼峰命名的自动映射，不然 `cover_img` 映射不到 `coverImg`：

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```

## 六、文件上传：从本地磁盘到 MinIO

项目里写了两个版本的上传接口，放在一起对比正好能说明**为什么要用对象存储**。

### 6.1 本地磁盘版：能用，但处处是坑

第一个版本把文件写到本机目录，文件名用 UUID 防覆盖。问题很快浮现：路径硬编码（换台机器就失效）、多实例部署时文件散落在各台机器上不共享、还得额外写静态资源映射才能让 URL 访问到磁盘文件。它的问题不在代码，在架构。

### 6.2 MinIO 版：配置、建桶、上传三步走

MinIO 是开源的 S3 兼容对象存储，一个独立服务专门管文件，应用只管调 API。第一步是把连接配置做成类型安全的属性绑定：

```yaml
minio:
  endpoint: http://localhost:9000   # API 地址（注意是 9000，不是控制台 9001）
  access-key: minioadmin
  secret-key: minioadmin123
  bucket: big-event                 # 存储桶名称，不存在会自动创建
```

```java
@ConfigurationProperties(prefix = "minio")
@Data
public class MinioProperties {
    private String endpoint;
    private String accessKey;
    private String secretKey;
    private String bucket;
}
```

第二步是注册 `MinioClient`，并且**在启动时自动初始化桶**：不存在则创建，再统一设置成「公开读」策略——桶里的对象可以直接通过 URL 访问，无需签名：

```java
@Bean
public MinioClient minioClient() {
    MinioClient client = MinioClient.builder()
            .endpoint(endpoint)
            .credentials(properties.getAccessKey(), properties.getSecretKey())
            .build();
    // 客户端就绪后，立刻初始化 bucket（保证「先有客户端、再操作桶」的顺序）
    initBucket(client);
    return client;
}

private void initBucket(MinioClient client) {
    try {
        boolean exists = client.bucketExists(
                BucketExistsArgs.builder().bucket(properties.getBucket()).build());
        if (!exists) {
            client.makeBucket(MakeBucketArgs.builder().bucket(properties.getBucket()).build());
        }
        // 公开读策略：允许任何人执行 s3:GetObject
        // Version 必须是固定的 "2012-10-17"（S3 策略语言的版本标识符，不是日期），
        // 写成其它值 MinIO 会拒绝该策略，导致桶保持私有、图片无法直接访问
        String policy = """
                {
                  "Version": "2012-10-17",
                  "Statement": [{
                      "Effect": "Allow",
                      "Principal": {"AWS": ["*"]},
                      "Action": ["s3:GetObject"],
                      "Resource": ["arn:aws:s3:::%s/*"]
                  }]
                }
                """.formatted(properties.getBucket());
        client.setBucketPolicy(
                SetBucketPolicyArgs.builder().bucket(properties.getBucket()).config(policy).build());
    } catch (Exception e) {
        // 初始化失败只打日志，不阻塞应用启动
        System.err.println("[MinIO] 初始化 bucket 失败：" + e.getMessage());
    }
}
```

有一个细节值得记下：无论桶是新建还是已存在，策略都**重新应用一次**——这样即使某次启动时设置策略失败、桶停留在私有状态，下次启动也能自动修正。

第三步是上传接口本身，UUID 生成唯一对象名（保留原始扩展名），流式上传后拼接可直接访问的 URL：

```java
String objectName = UUID.randomUUID().toString().replace("-", "") + suffix;

minioClient.putObject(
        PutObjectArgs.builder()
                .bucket(minioProperties.getBucket())
                .object(objectName)
                .stream(file.getInputStream(), file.getSize(), -1)  // -1 让 SDK 自动定分片大小
                .contentType(file.getContentType())
                .build());

// 桶是公开读，这个 URL 可以直接在浏览器打开
String url = endpoint + "/" + minioProperties.getBucket() + "/" + objectName;
```

对比一下两个版本：

| | 本地磁盘 | MinIO |
| --- | --- | --- |
| 部署耦合 | 路径硬编码，换机失效 | 独立服务，配置驱动 |
| 多实例 | 文件不共享 | 所有实例访问同一存储 |
| URL 访问 | 需额外静态资源映射 | 公开读策略，直接可访问 |
| 容量 | 受单机磁盘限制 | 横向扩展 |

## 七、不足与改进

复盘要看做得对的，也要看留了坑的。以下是这个项目当前明确的问题和对应的改进方向：

**密码用了裸 MD5。** MD5 是快速哈希，彩虹表可以秒查常见口令。改进方向是 BCrypt：自带随机盐、可调慢速因子，同一个密码每次哈希结果都不同。学习项目可以接受 MD5，但值得在第一次写时就养成用 BCrypt 的习惯。

**Redis TTL（1 小时）短于 JWT 有效期（12 小时）。** 两个过期时间不一致，实际效果是 token 一小时就失效，用户被迫频繁重新登录——JWT 的 12 小时成了摆设。应该让两者对齐，或者干脆让 JWT 只做载体、有效期完全交给 Redis 管理。

**登录没有作废旧 token。** 每次登录都往 Redis 新增一个 token，旧 token 只要没过期依然有效，Redis 里会积累一个用户的多个有效 token。更严谨的做法是登录时按用户维度删旧 token，或者改用 `userId → token` 的结构存储。

**自动填充没做完。** 项目里已经建了 `@AutoFill` 注解和 `OperationType`（INSERT/UPDATE）枚举，但 AOP 依赖和切面类都没写，`createTime`/`updateTime` 目前是每个 ServiceImpl 里手动 set 的。补一个切面拦截标了 `@AutoFill` 的 mapper 方法、通过反射统一填充公共字段，是一个很好的下一步练习——这也是把这个项目从「跟教程」推向「自己设计」的一步。

**异常处理粒度太粗。** `GlobalExceptionHandler` 只有一个 `Exception.class` 兜底，参数校验异常和业务异常返回的信息格式并不友好。可以按异常类型细分：`MethodArgumentNotValidException` 提取第一条字段错误、自定义 `BusinessException` 区分业务语义。

## 八、总结

用一条主线收束全文：**big-event 的价值不在于功能多，而在于每个功能都踩在一条完整的工程链路上**。

- 认证：JWT 携带身份 + Redis 掌握生死，拦截器统一校验，`ThreadLocal` 全链路传递用户上下文，`afterCompletion` 里 `remove()` 防串号防泄漏；
- 校验：内置注解 → 分组校验（一个 POJO 两套规则）→ 自定义 `@State` 注解（注解 + 校验器 + `@Constraint` 关联），复杂度随语义递进；
- 数据：PageHelper 一行开启物理分页，`<where>/<if>` 动态 SQL 应对可选筛选，`Result`/`PageBean` 统一响应结构，`@RestControllerAdvice` 兜底所有异常；
- 存储：文件从本地磁盘迁到 MinIO，`@ConfigurationProperties` 类型安全配置，启动时自动建桶 + 公开读策略，上传接口 UUID 防覆盖。

<!-- 下一步的计划：补上 AOP 自动填充、把 MD5 换成 BCrypt、对齐两边的过期时间。坑都记在上一节了，改完再来一篇续集。 -->
