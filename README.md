# Graceful Response 4.0 使用文档

基于 Spring Boot 4.0 / Spring Framework 7 / Java 21+ 的优雅响应处理组件，提供一站式统一返回值封装、全局异常处理、自定义异常错误码等功能。

## 1. 环境要求

| 组件 | 最低版本 |
|---|---|
| Spring Boot | 4.0.0 |
| Spring Framework | 7.0.1+ |
| Java | 21（兼容 25） |
| Maven | 3.6.3+ |
| Jakarta EE | 11（jakarta.validation 3.1, jakarta.annotation 3.0） |

## 2. 快速开始

### 2.1 引入依赖

```xml
<dependency>
    <groupId>io.github.mydtx</groupId>
    <artifactId>graceful-response</artifactId>
    <version>4.0.0</version>
</dependency>
```

### 2.2 启动类添加 @EnableGracefulResponse

```java
@SpringBootApplication
@EnableGracefulResponse
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2.3 配置扫描包（必须）

> **重要**：`scan-packages` 是必须配置项，不配置则所有非 void 接口不会被包装。

```yaml
gr:
  scan-packages:
    - com.example.module.*.controller
```

支持 `*` 和 `**` 通配符匹配。

### 2.4 Controller 直接返回业务数据

```java
@RestController
public class UserController {
    @GetMapping("/user")
    public UserInfoVO getUser(Long id) {
        return userService.getById(id);
    }
}
```

响应自动包装为：

```json
{
  "status": {
    "code": "0",
    "msg": "ok"
  },
  "payload": {
    "id": 1,
    "name": "张三"
  }
}
```

## 3. 响应格式

### 3.1 内置 Style 0（默认）

嵌套结构，`status` 和 `payload` 分开：

```yaml
gr:
  response-style: 0
```

```json
{
  "status": { "code": "0", "msg": "ok" },
  "payload": { ... }
}
```

### 3.2 内置 Style 1

扁平结构，`code`、`msg`、`data` 平铺：

```yaml
gr:
  response-style: 1
```

```json
{
  "code": "0",
  "msg": "ok",
  "data": { ... }
}
```

### 3.3 自定义响应格式

当内置格式不满足需求时，实现 `Response` 接口：

```java
public class CustomResponseImpl implements Response {

    private Integer code;
    private String msg;
    private Object data = Collections.EMPTY_MAP;

    @Override
    public void setStatus(ResponseStatus statusLine) {
        this.code = Integer.valueOf(statusLine.getCode());
        this.msg = statusLine.getMsg();
    }

    @Override
    @JsonIgnore
    public ResponseStatus getStatus() {
        return null;
    }

    @Override
    public void setPayload(Object payload) {
        this.data = payload;
    }

    @Override
    @JsonIgnore
    public Object getPayload() {
        return null;
    }

    // getter/setter 省略...
}
```

配置全限定类名：

```yaml
gr:
  response-class-full-name: com.example.common.CustomResponseImpl
```

> 注意：配置了 `response-class-full-name` 后，`response-style` 不再生效。自定义类必须有无参构造器。

## 4. 异常与错误码处理

### 4.1 @ExceptionMapper — 自定义异常映射

在自定义异常类上标注 `@ExceptionMapper`，将异常与错误码绑定：

```java
@ExceptionMapper(code = "1404", msg = "找不到对象")
public class NotFoundException extends RuntimeException {
}
```

Service 中直接抛出即可：

```java
public User getById(Long id) {
    User user = mapper.findById(id);
    if (user == null) {
        throw new NotFoundException();
    }
    return user;
}
```

接口返回：

```json
{
  "status": { "code": "1404", "msg": "找不到对象" },
  "payload": {}
}
```

### 4.2 GracefulResponse 工具类 — 无需定义异常类

不想为每个错误码创建新异常类时，使用工具类直接抛出：

```java
public void transfer() {
    if (hasRisk()) {
        GracefulResponse.raiseException("1007", "有内鬼，终止交易");
    }
    // 也支持传入原始异常
    // GracefulResponse.raiseException("500", "系统异常", e);
}
```

### 4.3 @ExceptionAliasFor — 外部异常别名

为第三方异常（如 Spring 内置的 `NoHandlerFoundException`）定义错误码：

**第一步：创建别名异常类**

```java
@ExceptionAliasFor(code = "1404", msg = "接口不存在", aliasFor = NoHandlerFoundException.class)
public class NotFoundException extends RuntimeException {
}
```

**第二步：注册别名**

```java
@Configuration
public class GracefulResponseConfig extends AbstractExceptionAliasRegisterConfig {
    @Override
    protected void registerAlias(ExceptionAliasRegister aliasRegister) {
        aliasRegister.doRegisterExceptionAlias(NotFoundException.class);
    }
}
```

### 4.4 异常处理优先级

当 Controller 层抛出异常时，按以下顺序确定返回的错误码：

```
1. GracefulResponseException    → 直接取异常中的 code/msg
2. @ExceptionMapper 注解        → 从异常类的注解读取 code/msg
3. @ExceptionAliasFor 别名映射  → 从已注册的别名中查找
4. 兜底默认                     → code="1", msg="error"（可通过配置覆盖）
```

## 5. 参数校验

### 5.1 @ValidationStatusCode — 指定校验异常码

对入参类的字段标注：

```java
@Data
public class UserQuery {
    @NotNull(message = "userName不能为空")
    @Length(min = 6, max = 12)
    @ValidationStatusCode(code = "520")
    private String userName;
}
```

校验失败时返回：

```json
{
  "status": { "code": "520", "msg": "userName不能为空" },
  "payload": {}
}
```

对 Controller 方法参数标注：

```java
@GetMapping("/query")
@ValidationStatusCode(code = "1314")
public void query(@NotNull(message = "userId不能为空") Long userId) {
    // ...
}
```

### 5.2 错误码取值优先级

**字段级别校验（BindException）：**
```
字段上的 @ValidationStatusCode → 所在类上的 @ValidationStatusCode → gr.default-validate-error-code → 默认 "1"
```

**方法级别校验（MethodArgumentNotValidException）：**
```
方法上的 @ValidationStatusCode → 所在类上的 @ValidationStatusCode → gr.default-validate-error-code → 默认 "1"
```

## 6. 排除不需要包装的接口

使用 `@ExcludeFromGracefulResponse` 注解跳过特定接口：

```java
@RestController
public class FileController {

    @GetMapping("/download")
    @ExcludeFromGracefulResponse
    public byte[] download(String filename) {
        return fileService.download(filename);
    }
}
```

该接口将直接返回原始数据，不经过统一包装。

## 7. 配置参考

```yaml
gr:
  # Controller 包扫描路径（必须配置，支持 * 和 ** 通配符）
  scan-packages:
    - com.example.module.*.controller

  # 自定义 Response 类全限定名（配置后 response-style 不生效）
  response-class-full-name:

  # 响应格式风格：0=嵌套结构（默认），1=扁平结构
  response-style: 0

  # 默认成功响应码（默认 "0"）
  default-success-code: "0"

  # 默认成功提示（默认 "ok"）
  default-success-msg: "ok"

  # 默认失败响应码（默认 "1"）
  default-error-code: "1"

  # 默认失败提示（默认 "error"）
  default-error-msg: "error"

  # 全局参数校验错误码（默认等于 default-error-code）
  default-validate-error-code: "1"

  # 是否在全局异常处理器中打印异常日志（默认 true）
  print-exception-in-global-advice: true
```

## 8. 架构说明

### 8.1 核心组件

| 组件 | Order | 职责 |
|---|---|---|
| `ValidationExceptionAdvice` | 100 | 参数校验异常处理（最高优先级） |
| `GlobalExceptionAdvice` | 200 | 全局异常兜底处理 |
| `NotVoidResponseBodyAdvice` | 1000 | 非 void 返回值自动包装 |
| `VoidResponseBodyAdvice` | 1000 | void 返回值自动返回成功响应 |

### 8.2 扩展点

| 接口 | 作用 |
|---|---|
| `ResponseFactory` | 自定义 Response 对象的创建逻辑 |
| `ResponseStatusFactory` | 自定义 ResponseStatus 的创建逻辑 |
| `Response` | 自定义响应格式（实现此接口） |
| `ResponseStatus` | 自定义响应状态结构 |

所有 Bean 均使用 `@ConditionalOnMissingBean` 注册，可通过自定义实现覆盖默认行为。

### 8.3 注解汇总

| 注解 | 作用域 | 说明 |
|---|---|---|
| `@EnableGracefulResponse` | 启动类 | 激活组件 |
| `@ExceptionMapper` | 异常类 | 将异常映射到错误码 |
| `@ExceptionAliasFor` | 异常类 | 为外部异常定义别名 |
| `@ValidationStatusCode` | 字段/类/方法 | 指定参数校验的错误码 |
| `@ExcludeFromGracefulResponse` | Controller 方法 | 排除统一包装 |
