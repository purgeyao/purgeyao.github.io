# SpringBoot让你的Bean动起来(自定义参数解析HandlerMethodArgumentResolver)

## 简介

我们 `Controller` 用到的一些 `Bean` 需要通过一定的方式去获取的，可以通过注入方式获取其他获取方式进行获取。

比如：需要用到用户实例，我们通常做法为下

```
@Resource
private UserService userService;

@GetMapping("getUserByUsername")
public String getUserByUsername(HttpServletRequest request) {

    ...
    AuthUser user = userService.getUserByUsername("小明");
    ...
}
```

这样是一般的做法，我们可以发现 `HttpServletRequest` 可以通过注入方式加载，也可以直接在 `public String getUserByUsername()` 方法参数获取到。

这样的方式 也把我们的 `User` 对象封装到方法参数里。

当我们执行 `Controller` 就会有一个对应用户的对象存在了。


## 功能使用

### 自定义 `UserArgumentResolver`

这里我们需要使用到一个 `HandlerMethodArgumentResolver` 接口。

自定义 `UserArgumentResolver` 处理 `User` 。

```
/**
 * 封装 参数方式获取 {@link AuthUser} 对象
 *
 * @author purgeyao
 * @since 1.0
 */
public class UserArgumentResolver implements HandlerMethodArgumentResolver {

  @Override
  public boolean supportsParameter(MethodParameter methodParameter) {
    Class<?> aClass = methodParameter.getParameterType();
    // 判断是否为 AuthUser 类
    return aClass == AuthUser.class;
  }

  @Override
  public Object resolveArgument(MethodParameter methodParameter,
      ModelAndViewContainer modelAndViewContainer, NativeWebRequest nativeWebRequest,
      WebDataBinderFactory webDataBinderFactory) throws Exception {

    HttpServletRequest request = nativeWebRequest.getNativeRequest(HttpServletRequest.class);

    // 获取参数
    
    String name = request.getParameter("name");

    // TODO 获取对应的用户对象 查询操作

    AuthUser user = userService.getUserByUsername(name);

    return user;
  }
}

```

上述代码会在执行 `Controller` 之前处理。通过获取请求头 或者 请求里的参数(具体看自己的业务)。

执行相应的对象查询操作。接下来就可以在 `Controller` 参数里可以方便使用了。

### `Controller` 方法参数使用

```
@GetMapping("getUserByUsername")
public String getUserByUsername(AuthUser user) {
    ...
    user.getAge();
    ...
}
```

## 总结

上述介绍 `HandlerMethodArgumentResolver` 使用，原理请参考相关文章。

> 示例代码地址:[UserArgumentResolver](https://github.com/purgeyao/springboot-related/blob/master/Java%E5%85%A8%E5%AE%B6%E6%A1%B6/%E6%9D%82%E4%B9%B1/%E6%96%B9%E6%B3%95%E5%8F%82%E6%95%B0%E8%87%AA%E5%8A%A8%E6%B3%A8%E5%85%A5/adapter-Demo/src/main/java/com/example/config/UserArgumentResolver.java)

> 作者GitHub:
[Purgeyao](https://github.com/purgeyao) 欢迎关注

> qq交流群: `812321371` 微信交流群: `MercyYao`

> 微信公众号:

![微信公众号二维码](https://purgeyao.github.io/img/about-my-mp-8cm.jpg)