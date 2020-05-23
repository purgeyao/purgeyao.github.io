# SpringCloud异常处理统一封装我来做-使用篇

## 简介

重复功能我来写。在 `SpringBoot` 项目里都有全局异常处理以及返回包装等，返回前端是带上`succ`、`code`、`msg`、`data`等字段。单个项目情况下很好解决，当微服务模块多的情况下，很多情况开发都是复制原有代码进行构建另外一个项目的，导致这些功能升级需要修改多个服务，在这个基础上，我们封装了一个组件 `unified-dispose-spring-cloud-starter` 里面包含了一些基础的异常处理以及返回包装功能。

## 依赖添加启动功能

添加依赖
**ps:** 实际version版本请使用最新版
**最新版本:** [![Maven Central](https://img.shields.io/maven-central/v/com.purgeteam.cloud/unified-dispose-spring-cloud-starter.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:com.purgeteam.cloud%20AND%20a:unified-dispose-spring-cloud-starter)

[点击查看最新新版本](https://search.maven.org/search?q=g:com.purgeteam.cloud%20AND%20a:unified-dispose-spring-cloud-starter)

```
<dependency>
  <groupId>com.purgeteam.cloud</groupId>
  <artifactId>unified-dispose-spring-cloud-starter</artifactId>
  <version>0.3.0.RELEASE</version>
</dependency>
```

启动类添加 `@EnableGlobalDispose` 注解开启以下功能。
```
@EnableGlobalDispose
@SpringBootApplication
public class GlobalDisposeSpringBootApplication {

  public static void main(String[] args) {
    SpringApplication.run(GlobalDisposeSpringBootApplication.class, args);
  }

}
```

## One 异常处理⚠️

在项目中经常出现系统异常的情况，比如`NullPointerException`等等。如果默认未处理的情况下，`springboot`会响应默认的错误提示，这样对用户体验不是友好，系统层面的错误，用户不能感知到，即使为`500`的错误，可以给用户提示一个类似`服务器开小差`的友好提示等。

模块里以及包含了一些基本的异常处理方式(及不需要做任何代码编写已经具有基本异常处理)，以及一些常见的异常code，用户只需要关心业务异常处理即可，直接通过 `throw new 异常` 的方式抛出即可。

### 异常处理包含类型

```
# 通用500异常
Exception 类捕获 500 异常处理

# Feign 异常
FeignException 类捕获
ClientException 类捕获

# 业务自定义
BusinessException 类捕获 业务通用自定义异常

# 参数校验异常
HttpMessageNotReadableException 参数错误异常
BindException 参数错误异常
```

### 程序主动抛出异常

```
throw new BusinessException(BusinessErrorCode.BUSINESS_ERROR);
// 或者
throw new BusinessException("CLOUD800","没有多余的库存");
```

通常不建议直接抛出通用的BusinessException异常，应当在对应的模块里添加对应的领域的异常处理类以及对应的枚举错误类型。

如会员模块：
创建`UserException`异常类、`UserErrorCode`枚举。

UserException:

继承 `BusinessException` 

```java
/**
 * {@link RuntimeException} user 业务异常
 *
 * @author purgeyao
 * @since 1.0
 */
@Getter
public class UserException extends BusinessException {

  private String code;
  private boolean isShowMsg = true;

  /**
   * 使用枚举传参
   *
   * @param errorCode 异常枚举
   */
  public UserException(UserErrorCode errorCode) {
    super(errorCode.getCode(), errorCode.getMessage());
    this.code = errorCode.getCode();
  }

}
```

UserErrorCode:

```java
@Getter
public enum UserErrorCode {
    /**
     * 权限异常
     */
    NOT_PERMISSIONS("CLOUD401","您没有操作权限"),
    ;

    private String code;

    private String message;

    UserErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }
}
```

最后业务使用如下：

```java
// 判断是否有权限抛出异常
throw new UserException(UserErrorCode.NOT_PERMISSIONS);
```

**上述方式，抛出异常后会被模块处理。前台返回如下**：

```
{
  "succ": false,        // 是否成功
  "ts": 1566467628851,  // 时间戳
  "data": null,         // 数据
  "code": "CLOUD800",   // 错误类型
  "msg": "业务异常",    // 错误描述
}
```

## Tow 统一返回封装🗳

在REST风格的开发中，避免通常会告知前台返回是否成功以及状态码等信息。这里我们通常返回的时候做一次`util`的包装处理工作，如：`Result`类似的类，里面包含`succ`、`code`、`msg`、`data`等字段。


接口调用处理类似如下:

```
  @GetMapping("hello")
  public Result list(){
    return Result.ofSuccess("hello");
  }
```
结果:
```
{
  "succ": ture,         // 是否成功
  "ts": 1566467628851,  // 时间戳
  "data": "hello",      // 数据
  "code": null,         // 错误类型
  "msg": null,          // 错误描述
  "fail": true
}
```
### 功能使用

默认情况所有的 `web controller` 都会被封装为以下返回格式。

接口：

```java
@GetMapping("test")
public String test(){
  return "test";
}
```

返回

```json
{
  "succ": true,             // 是否成功
  "ts": 1566386951005,      // 时间戳
  "data": "test",           // 数据
  "code": null,             // 错误类型
  "msg": null,              // 错误描述
}
```

### 忽略封装注解 `@IgnoreResponseAdvice`

```
public @interface IgnoreResponseAdvice {

    /**
     * 是否进行全局异常处理封装
     * @return true:进行处理;  false:不进行异常处理
     */
    boolean errorDispose() default true;

}
```

`@IgnoreResponseAdvice`允许范围为：**类 + 方法**，标识在类上这个类下的所有方法的返回都将忽略返回封装。

接口：

```java
@IgnoreResponseAdvice // 忽略数据包装 可添加到类、方法上
@GetMapping("test")
public String test(){
  return "test";
}
```

返回 `test`

### `FeignClient` 调用异常返回处理

默认 `FeignClient` 调用异常会返回 `FeignException` 500 异常，由于开启了统一异常处理，被调用方feign异常会被处理为包装过后结果返回。

**服务端：**
```
@Override
public Boolean testBoolean() throws Exception {
    throw new Exception("模拟异常");
}
```

**调用端：**
```
@GetMapping("testBoolean")
public Boolean testBoolean() throws Exception {
    // 调用服务端异常
    // 如果服务端不添加 @IgnoreResponseAdvice(errorDispose = false) 会出现下面异常
    // There was an unexpected error (type=Internal Server Error, status=500).
    // Error while extracting response for type [class java.lang.Boolean] and content type [application/json;charset=UTF-8]; nested exception is org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot deserialize instance of `java.lang.Boolean` out of START_OBJECT token; nested exception is com.fasterxml.jackson.databind.exc.MismatchedInputException: Cannot deserialize instance of `java.lang.Boolean` out of START_OBJECT token at [Source: (PushbackInputStream); line: 1, column: 1]
    return exampleFeignClient.testBoolean();
}
```

因此 `exampleFeignClient#testBoolean` 调用出现了 feign 转换异常了。

这里需要关闭提供者的 feign 接口异常处理。或者在此接口类上添加 `@IgnoreResponseAdvice(errorDispose = false)` errorDispose 设置为 false

```
/**
 * 添加 IgnoreResponseAdvice#errorDispose 设置为异常不需要处理包装
 */
@Override
@IgnoreResponseAdvice(errorDispose = false)
public Boolean testBoolean() throws Exception {
    throw new Exception("模拟异常");
}
```

## 总结

项目里很多重复的code，我们可以通过一定的方式去简化，以达到一定目的减少开发量。PurgeTeam 具有一些优秀的开源组件，减少日常的开发量。

> 示例代码地址:[unified-dispose-spring-cloud-starter](https://github.com/purgeteam/spirng-cloud-purgeteam/tree/master/spring-cloud-purgeteam-examples/unified-dispose-spring-cloud-starter-examples)

> 作者GitHub:
[Purgeyao](https://github.com/purgeyao) 欢迎关注

> qq交流群: `812321371` 微信交流群: `MercyYao`

> 微信公众号:

![微信公众号二维码](https://purgeyao.github.io/img/about-my-mp-8cm.jpg)