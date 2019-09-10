# springboot之swagger快速启动

# 简介
> 介绍

可能大家都有用过`swagger`,可以通过`ui`页面显示接口信息，快速和前端进行联调。

没有接触的小伙伴可以参考[官网](https://swagger.io/)文章进行了解下[demo页面](http://editor.swagger.io)。

> 多应用

当然在单个应用大家可以配置`SwaggerConfig`类加载下`buildDocket`,就可以快速构建好`swagger`了。

代码大致如下：

```
/**
 * Swagger2配置类
 * 在与spring boot集成时，放在与Application.java同级的目录下。
 * 通过@Configuration注解，让Spring来加载该类配置。
 * 再通过@EnableSwagger2注解来启用Swagger2。
 */
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    /**
     * 创建API应用
     * apiInfo() 增加API相关信息
     * 通过select()函数返回一个ApiSelectorBuilder实例,用来控制哪些接口暴露给Swagger来展现，
     * 本例采用指定扫描的包路径来定义指定要建立API的目录。
     * 
     * @return
     */
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.swaggerTest.controller"))
                .paths(PathSelectors.any())
                .build();
    }
    
    /**
     * 创建该API的基本信息（这些基本信息会展现在文档页面中）
     * 访问地址：http://项目实际地址/swagger-ui.html
     * @return
     */
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2构建RESTful APIs")
                .description("更多请关注http://www.baidu.com")
                .termsOfServiceUrl("http://www.baidu.com")
                .contact("sunf")
                .version("1.0")
                .build();
    }
}

```

# 模块化-Starter

> 缘由

有开发过微服务的小伙伴应该体会过。当微服务模块多的情况下，每个模块都需要配置这样的一个类进行加载`swagger`。造成每个模块都存在大致一样的`SwaggerConfig`,极端的情况下，有些朋友复制其他模块的`SwaggerConfig`进行改造之后，发现仍然加载不出`swagger`的情况，造成明明是复制的，为何还加载不出，排查此bug及其费时间。

在此之上，可以构建出一个`swagger-starter`模块，只需要引用一个`jar`，加载一些特殊的配置，就可以快速的使用到`swagger`的部分功能了。

> 设计

创建模块`swagger-spring-boot-starter`。
功能大致如下：

1. 加载SwaggerConfig。
2. 通过配置化配置swagger。
3. Enable加载注解。

### 1. 创建SwaggerConfig

`SwaggerConfig`和之前的一致，只是里面的配置需要外部化。

```
@Configuration
@PropertySource(value = "classpath:swagger.properties", ignoreResourceNotFound = true, encoding = "UTF-8")
@EnableConfigurationProperties(SwaggerProperties.class)
public class SwaggerConfig {

  @Resource
  private SwaggerProperties swaggerProperties;

  @Bean
  public Docket buildDocket() {
    return new Docket(DocumentationType.SWAGGER_2)
        .apiInfo(buildApiInf())
        .select()
        .apis(RequestHandlerSelectors.basePackage(""))
        .paths(PathSelectors.any())
        .build();
  }

  private ApiInfo buildApiInf() {
    return new ApiInfoBuilder()
        .title(swaggerProperties.getTitle())
        .description(swaggerProperties.getDescription())
        .termsOfServiceUrl(swaggerProperties.getTermsOfServiceUrl())
        .contact(new Contact("skyworth", swaggerProperties.getTermsOfServiceUrl(), ""))
        .version(swaggerProperties.getVersion())
        .build();
  }
}
```

### 2. 创建SwaggerProperties 配置相关
配置通过`@PropertySource`注解加载`resources`目录下的`swagger.properties`。

创建`SwaggerProperties`配置类,这个类里包含了一般swagger初始化要使用的一些常用的属性，如扫描包路径、title等等。

```
@Data
@ToString
@ConfigurationProperties(SwaggerProperties.PREFIX)
public class SwaggerProperties {

  public static final String PREFIX = "swagger";

  /**
   * 文档扫描包路径
   */
  private String basePackage = "";

  /**
   * title 如: 用户模块系统接口详情
   */
  private String title = "深兰云平台系统接口详情";

  /**
   * 服务文件介绍
   */
  private String description = "在线文档";

  /**
   * 服务条款网址
   */
  private String termsOfServiceUrl = "https://www.deepblueai.com/";

  /**
   * 版本
   */
  private String version = "V1.0";


}
```

做好这两件事情基本大工搞成了，为了更好的使用配置，在idea里和官方`starter`包一样，我们还需要配置一个`additional-spring-configuration-metadata.json`,让我们自己的配置也具有提示的功能，具体介绍请产考：[配置提示](https://segmentfault.com/a/1190000020247777) [配置提示](https://blog.csdn.net/weixin_43367055/article/details/100174407) [配置提示](https://juejin.im/post/5d6a3250518825618e67a4a9) [配置提示](https://www.cnblogs.com/Purgeyao/p/11439555.html) [配置提示](https://my.oschina.net/u/4185467/blog/3100215) ...
![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/swagger-ml.png)

![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/swagger-tis.png)


### 3. 加载SwaggerConfig等特性

因为是starter模块，可能他人的项目目录和starter模块的目录不一致，导致加载不到`SwaggerConfig`类，我们需要使用`spring.factories`把`SwaggerConfig`类装载到spring容器。


resources/META-INF 
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  io.purge.swagger.SwaggerConfig
```

当然本次基于Enable方式去加载`SwaggerConfig`。

创建@EnableSwaggerPlugins注解类，使用`@Import(SwaggerConfig.class)`将`SwaggerConfig`导入大工搞成。

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(SwaggerConfig.class)
@EnableSwagger2
public @interface EnableSwaggerPlugins {

}
```

# 使用

> 添加依赖

把自己编写好的`swagger`通过`maven`打包，自己项目引用。

```
<dependency>
  <groupId>com.purge.swagger</groupId>
  <artifactId>swagger-spring-boot-starter<factId>
  <version>0.1.0.RELEASE</version>
</dependency>
```

> 配置swagger.properties文件
- 在自己项目模块的`resources`目录下 创建`swagger.properties`配置

- swagger.properties 大致配置如下

```
swagger.basePackage="swagger扫描项目包路径"
swagger.title="swagger网页显示标题"
swagger.description="swagger网页显示介绍"
```

> 启动类添加`@EnableSwaggerPlugins`注解。

```
@EnableSwaggerPlugins
@SpringBootApplication
public class FrontDemoApplication {

  public static void main(String[] args) {
    SpringApplication.run(FrontDemoApplication.class, args);
  }

}
```
访问`http://ip:端口/swagger-ui.html`检查swagger-ui是否正常。

![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/swagger-ui.png)

# 总结

简单的`starter`代码编写可以减少新模块的复杂性，只需要简单的配置就可以使用相应的特性，减少复制代码不必要的错误。

> 示例代码地址: [swagger-spring-boot](https://github.com/purgeteam/swagger-spring-boot)