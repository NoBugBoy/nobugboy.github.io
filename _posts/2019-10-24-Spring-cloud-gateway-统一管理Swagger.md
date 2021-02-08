# SpringCloud搭建专题【网关统一管理swagger】
所有的业务服务正常配置swagger即可，不需要引入swagger-ui,统一在gateway上配置。

从这开始就是gateway的配置，其余的eurekaclient和正常的单机项目一样配置即可

首先在已经创建好的gateway微服务中引入swagger jar包
```xml
        <dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger2</artifactId>
			<version>2.9.2</version>
		</dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
```
声明gateway中**不需要配置**下面这些,否则就会疯狂报错
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket docket(){
        return  new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .enable(true)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.basecloud.auth.controller"))
                .build();
    }
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                //页面标题
                .title("Auth Restful Api")
                //创建人
                .contact(new Contact("yujian", "http://www.basecloud.com/", ""))
                //版本号
                .version("1.0")
                //描述
                .description("API 描述")
                .build();
    }
}
但是swagger默认回去请求一些接口，没有配置上面的代码就需要手动去写一下接口

@RestController
@RequestMapping("/swagger-resources")
public class SwaggerResourceController {
    @Autowired
    private SwaggerResourcesConfig swaggerResources;

    @Autowired(required = false)
    private SecurityConfiguration securityConfiguration;
    @Autowired(required = false)
    private UiConfiguration uiConfiguration;

    @GetMapping("/configuration/security")
    public Mono<ResponseEntity<SecurityConfiguration>> securityConfiguration() {
        return Mono.just(new ResponseEntity<>(
                Optional.ofNullable(securityConfiguration).orElse(SecurityConfigurationBuilder.builder().build()), HttpStatus.OK));
    }

    @GetMapping("/configuration/ui")
    public Mono<ResponseEntity<UiConfiguration>> uiConfiguration() {
        return Mono.just(new ResponseEntity<>(
                Optional.ofNullable(uiConfiguration).orElse(UiConfigurationBuilder.builder().build()), HttpStatus.OK));
    }

    @GetMapping("")
    public Mono<ResponseEntity> swaggerResources() {
        return Mono.just((new ResponseEntity<>(swaggerResources.get(), HttpStatus.OK)));
    }
}

```

先拿配置文件来讲，根据路径名称转发，让网关转发到
http://localhost:8099/auth/v2/api-docs的路径上去，就会请求到注册为BASECLOUD-AUTH的服务（localhost:8091/v2/api-docs）拿到json
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191024174827146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
网关转发的地址：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191024175025352.png)
实际swagger的地址（auth服务）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191024175046189.png)
ok看到这里大概就能明白是如何统一管理swagger的了，我这里还单独配置了转发路径
```xml
swagger:
   names: auth,shop
```
然后配置swaggerResource的集合，下面是配置类
```java
@Component
@Primary
public class SwaggerResourcesConfig implements SwaggerResourcesProvider{

    @Value("${swagger.names}")
    private String[] apiNames;
    @Override
    public List<SwaggerResource> get() {
        List<SwaggerResource> collect = Arrays.asList(apiNames).stream().map(name -> {
            SwaggerResource swaggerResource = swaggerResource(name,  "/"+name+"/v2/api-docs");
            return swaggerResource;
        }).collect(Collectors.toList());

        return collect;
    }

    private SwaggerResource swaggerResource(String serviceId, String location){

        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(serviceId);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion("2.0");
        return swaggerResource;
    }
}

```
**到这就皆大欢喜，踩了一下午的坑，各种bug.....，希望不会再有人踩坑了，如果帮到了你点个赞谢谢啦！**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210120161709717.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)