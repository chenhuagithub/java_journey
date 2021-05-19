## @ConfigurationProperties和@Value的使用

`@ConfigurationProperties`和`@Value`的使用都是加载外部配置文件的注解，`@ConfigurationProperties`可以加载多个配置并封装成一个对象，而`@Value`只能加载单个配置，下面我们详细来介绍这两个注解的用法。

​	

首先，我们现在application.properties中编写Mysql的连接配置信息：

```properties
db.url=jdbc:mysql://localhost:3306/test？characterEncoding=utf8&useSSL=true
db.username=root
db.password=1234
db.driver-class-name=com.mysql.jdbc.Driver
```



#### @Value

> @Value只能获取单个配置项

Mysql配置类：

```java
@Data
@Component
public class MysqlProperties {
    @Value("${db.username}")
    private String username;
    @Value("${db.password}")
    private String password;
    @Value("${db.url}")
    private String url;
    @Value("${db.driver-class-name}")
    private String driverClassName;
}
```

容器配置：

```java
@Configuration
public class AdminKernelConfig {
    @Bean
    public DynamicWechatRoute dynamicWechatRoute(MysqlProperties mysqlProperties) {
        return new DynamicWechatRoute(mysqlProperties);
    }
}
```



#### @ConfigurationProperties

>  @ConfigurationProperties可以配置多个配置项，且一般@ConfigurationProperties会结合@EnableConfigurationProperties一起使用，@EnableConfigurationProperties主要作用是指明配置类是哪一个

Mysql配置类：

```java
@ConfigurationProperties(prefix = "db")
@Data
public class MysqlProperties {
    private String username;
    private String password;
    private String url;
    private String driverClassName;
}
```

容器配置：

```java
@Configuration
@EnableConfigurationProperties(MysqlProperties.class) // 用于指明配置类
public class AdminKernelConfig {
    @Bean
    public DynamicWechatRoute dynamicWechatRoute(MysqlProperties mysqlProperties) {
        return new DynamicWechatRoute(mysqlProperties);
    }
}
```



结语：

@ConfigurationProperties和@Value的使用并不难，只要写一个小demo就能够很好的掌握了～

