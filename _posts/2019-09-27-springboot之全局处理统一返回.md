# springboot之全局处理统一返回

## 简介

在REST风格的开发中，避免通常会告知前台返回是否成功以及状态码等信息。这里我们通常返回的时候做一次`util`的包装处理工作，如：`Result`类似的类，里面包含`succ`、`code`、`msg`、`data`等字段。

接口调用返回类似如下:

```
{
  "succ": false,        // 是否成功
  "ts": 1566467628851,  // 时间戳
  "data": null,         // 数据
  "code": "CLOUD800",   // 错误类型
  "msg": "业务异常",    // 错误描述
  "fail": true
}
```

当然在每个接口里返回要通过`Result`的工具类将这些信息给封装一下，这样导致业务和技术类的代码耦合在一起。

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

我们将这些操抽出一个公共`starter`包，各个服务依赖即可，做一层统一拦截处理的工作，进行技术解耦。

## 配置

> unified-dispose-springboot-starter

这个模块里包含异常处理以及全局返回封装等功能，下面。

完整目录结构如下：

```
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── purgetiem
│   │   │           └── starter
│   │   │               └── dispose
│   │   │                   ├── GlobalDefaultConfiguration.java
│   │   │                   ├── GlobalDefaultProperties.java
│   │   │                   ├── Interceptors.java
│   │   │                   ├── Result.java
│   │   │                   ├── advice
│   │   │                   │   └── CommonResponseDataAdvice.java
│   │   │                   ├── annotation
│   │   │                   │   ├── EnableGlobalDispose.java
│   │   │                   │   └── IgnorReponseAdvice.java
│   │   │                   └── exception
│   │   │                       ├── GlobalDefaultExceptionHandler.java
│   │   │                       ├── category
│   │   │                       │   └── BusinessException.java
│   │   │                       └── error
│   │   │                           ├── CommonErrorCode.java
│   │   │                           └── details
│   │   │                               └── BusinessErrorCode.java
│   │   └── resources
│   │       ├── META-INF
│   │       │   └── spring.factories
│   │       └── dispose.properties
│   └── test
│       └── java
```

---

> 统一返回处理

按照一般的模式，我们都需要创建一个可以进行处理包装的工具类以及一个返回对象。

**Result(返回类):**

创建`Result<T>` `T`为`data`的数据类型，这个类包含了前端常用的字段，还有一些常用的静态初始化`Result`对象的方法。

```
/**
 * 返回统一数据结构
 *
 * @author purgeyao
 * @since 1.0
 */
@Data
@ToString
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> implements Serializable {

  /**
   * 是否成功
   */
  private Boolean succ;

  /**
   * 服务器当前时间戳
   */
  private Long ts = System.currentTimeMillis();

  /**
   * 成功数据
   */
  private T data;

  /**
   * 错误码
   */
  private String code;

  /**
   * 错误描述
   */
  private String msg;

  public static Result ofSuccess() {
    Result result = new Result();
    result.succ = true;
    return result;
  }

  public static Result ofSuccess(Object data) {
    Result result = new Result();
    result.succ = true;
    result.setData(data);
    return result;
  }

  public static Result ofFail(String code, String msg) {
    Result result = new Result();
    result.succ = false;
    result.code = code;
    result.msg = msg;
    return result;
  }

  public static Result ofFail(String code, String msg, Object data) {
    Result result = new Result();
    result.succ = false;
    result.code = code;
    result.msg = msg;
    result.setData(data);
    return result;
  }

  public static Result ofFail(CommonErrorCode resultEnum) {
    Result result = new Result();
    result.succ = false;
    result.code = resultEnum.getCode();
    result.msg = resultEnum.getMessage();
    return result;
  }

  /**
   * 获取 json
   */
  public String buildResultJson(){
    JSONObject jsonObject = new JSONObject();
    jsonObject.put("succ", this.succ);
    jsonObject.put("code", this.code);
    jsonObject.put("ts", this.ts);
    jsonObject.put("msg", this.msg);
    jsonObject.put("data", this.data);
    return JSON.toJSONString(jsonObject, SerializerFeature.DisableCircularReferenceDetect);
  }
}
```

这样已经满足一般返回处理的需求了，在接口可以这样使用：

```
  @GetMapping("hello")
  public Result list(){
    return Result.ofSuccess("hello");
  }
```

当然这样是耦合的使用，每次都需要调用`Result`里的包装方法。

---

> `ResponseBodyAdvice` 返回统一拦截处理

`ResponseBodyAdvice`在 spring 4.1 新加入的一个接口，在消息体被`HttpMessageConverter`写入之前允许`Controller` 中 `@ResponseBody`修饰的方法或`ResponseEntity`调整响应中的内容，比如做一些返回处理。

**`ResponseBodyAdvice`接口里一共包含了两个方法**

- `supports`:该组件是否支持给定的控制器方法返回类型和选择的{@code HttpMessageConverter}类型

- `beforeBodyWrite`:在选择{@code HttpMessageConverter}之后调用，在调用其写方法之前调用。

**那么我们就可以在这两个方法做一些手脚。**

- `supports`用于判断是否需要做处理。

- `beforeBodyWrite`用于做返回处理。

`CommonResponseDataAdvice`类实现`ResponseBodyAdvice`两个方法。

`filter(MethodParameter methodParameter)` 私有方法里进行判断是否要进行拦截统一返回处理。

**如:**
- 添加自定义注解`@IgnorReponseAdvice`忽略拦截。
- 判断某些类不进行拦截.
- 判断某些包下所有类不进行拦截。如`swagger`的`springfox.documentation`包下的接口忽略拦截等。

**filter方法:**
判断为false就不需要进行拦截处理。
```
  private Boolean filter(MethodParameter methodParameter) {
    Class<?> declaringClass = methodParameter.getDeclaringClass();
    // 检查过滤包路径
    long count = globalDefaultProperties.getAdviceFilterPackage().stream()
        .filter(l -> declaringClass.getName().contains(l)).count();
    if (count > 0) {
      return false;
    }
    // 检查<类>过滤列表
    if (globalDefaultProperties.getAdviceFilterClass().contains(declaringClass.getName())) {
      return false;
    }
    // 检查注解是否存在
    if (methodParameter.getDeclaringClass().isAnnotationPresent(IgnorReponseAdvice.class)) {
      return false;
    }
    if (methodParameter.getMethod().isAnnotationPresent(IgnorReponseAdvice.class)) {
      return false;
    }
    return true;
  }
```

**CommonResponseDataAdvice**类:

最核心的就在`beforeBodyWrite`方法处理里。

1. 判断`Object o`是否为`null`，为`null`构建`Result`对象进行返回。
2. 判断`Object o`是否是`Result`子类或其本身，该情况下，可能是接口返回时创建了`Result`,为了避免再次封装一次，判断是`Result`子类或其本身就返回`Object o`本身。
3. 判断`Object o`是否是为`String`,在测试的过程中发现`String`的特殊情况，在这里做了一次判断操作，如果为`String`就进行`JSON.toJSON(Result.ofSuccess(o)).toString()`序列号操作。
4. 其他情况默认返回`Result.ofSuccess(o)`进行包装处理。

```
/**
 * {@link IgnorReponseAdvice} 处理解析 {@link ResponseBodyAdvice} 统一返回包装器
 *
 * @author purgeyao
 * @since 1.0
 */
@RestControllerAdvice
public class CommonResponseDataAdvice implements ResponseBodyAdvice<Object> {

  private GlobalDefaultProperties globalDefaultProperties;

  public CommonResponseDataAdvice(GlobalDefaultProperties globalDefaultProperties) {
    this.globalDefaultProperties = globalDefaultProperties;
  }

  @Override
  @SuppressWarnings("all")
  public boolean supports(MethodParameter methodParameter,
      Class<? extends HttpMessageConverter<?>> aClass) {
    return filter(methodParameter);
  }

  @Nullable
  @Override
  @SuppressWarnings("all")
  public Object beforeBodyWrite(Object o, MethodParameter methodParameter, MediaType mediaType,
      Class<? extends HttpMessageConverter<?>> aClass, ServerHttpRequest serverHttpRequest,
      ServerHttpResponse serverHttpResponse) {

    // o is null -> return response
    if (o == null) {
      return Result.ofSuccess();
    }
    // o is instanceof ConmmonResponse -> return o
    if (o instanceof Result) {
      return (Result<Object>) o;
    }
    // string 特殊处理
    if (o instanceof String) {
      return JSON.toJSON(Result.ofSuccess(o)).toString();
    }
    return Result.ofSuccess(o);
  }

  private Boolean filter(MethodParameter methodParameter) {
    ···略
  }

}
```

这样基本完成了核心的处理工作。当然还少了上文提到的`@IgnorReponseAdvice`注解。

**@IgnorReponseAdvice**:
比较简单点，只作为一个标识的作用。
```
/**
 * 统一返回包装标识注解
 *
 * @author purgeyao
 * @since 1.0
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface IgnorReponseAdvice {

}
```

> 加入spring容器

最后将`GlobalDefaultExceptionHandler`以`bean`的方式注入`spring`容器。

```
@Configuration
@EnableConfigurationProperties(GlobalDefaultProperties.class)
@PropertySource(value = "classpath:dispose.properties", encoding = "UTF-8")
public class GlobalDefaultConfiguration {

  ···略
  
  @Bean
  public CommonResponseDataAdvice commonResponseDataAdvice(GlobalDefaultProperties globalDefaultProperties){
    return new CommonResponseDataAdvice(globalDefaultProperties);
  }

}
```

将`GlobalDefaultConfiguration`在`resources/META-INF/spring.factories`文件下加载。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.purgetime.starter.dispose.GlobalDefaultConfiguration
```

不过我们这次使用注解方式开启。其他项目依赖包后，需要添加`@EnableGlobalDispose`才可以将全局拦截的特性开启。

将刚刚创建的`spring.factories`注释掉，创建`EnableGlobalDispose`注解。

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(GlobalDefaultConfiguration.class)
public @interface EnableGlobalDispose {

}
```

使用`@Import`将`GlobalDefaultConfiguration`导入即可。

> 使用

添加依赖
```
<dependency>
  <groupId>com.purgeteam</groupId>
  <artifactId>unified-dispose-deepblueai-starter</artifactId>
  <version>0.1.1.RELEASE</version>
</dependency>
```

启动类开启`@EnableGlobalDispose`注解即可。

> 1. 业务使用

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
  "fail": false             
}
```
> 2. 忽略封装注解:@IgnorReponseAdvice

`@IgnorReponseAdvice`允许范围为：类 + 方法，标识在类上这个类下的说有方法的返回都将忽略返回封装。

接口：

```java
@IgnorReponseAdvice // 忽略数据包装 可添加到类、方法上
@GetMapping("test")
public String test(){
  return "test";
}
```

返回 `test`

## 总结

项目里很多重复的code，我们可以通过一定的方式去简化，以达到一定目的减少开发量。

> 示例代码地址:[unified-dispose-springboot](https://github.com/purgeteam/unified-dispose-springboot)

> 作者GitHub:
[Purgeyao](https://github.com/purgeyao) 欢迎关注
