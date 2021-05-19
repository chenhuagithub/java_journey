## SpringBoot使用Dozer工具类

#### 引入依赖

```xml
<!--dozer -->
<dependency>
  <groupId>net.sf.dozer</groupId>
  <artifactId>dozer-spring</artifactId>
  <version>5.5.1</version>
</dependency>
<dependency>
  <groupId>net.sf.dozer</groupId>
  <artifactId>dozer</artifactId>
  <version>5.5.1</version>
</dependency>
```

#### 配置

##### 注解实现映射

```java
@Configuration
public class DozerConfig {
    @Bean
    public DozerBeanMapper getDozerBean() {
        DozerBeanMapper dozerBean = new DozerBeanMapper();
        return dozerBean;
    }
}
```

直接用`@Autowired`注入`DozerBeanMapper`就可以直接使用了，简单的配置智能映射两个类中变量名相同的属性。

> 实体类

```java
@Data
public class Users {
    private Integer id;
    private String name;
    private Integer age;
}
```

```java
@Data
public class UsersVo {
    private Integer id;
    private String name;
    private Integer age;
}
```

> 测试

```java
@Autowired
private DozerBeanMapper dozerBeanMapper;

@Test
public void dozerTest () {
  Users users = new Users();
  users.setId(2);
  users.setName("张三");
  users.setAge(23);
  UsersVo usersVo = dozerBeanMapper.map(users, UsersVo.class);
  System.out.println(usersVo);
}/**output
UsersVo(id=2, name=张三, age=23)
*//~
```

如果想要映射不同属性名之间的属性且不想使用xml配置文件的方式实现， 我们可以`@Mapping()`注解标识在相应的字段上实现不同名字的属性之间的映射，我们来看看下面的例子：

> 实体类

```java
@Data
public class Users {
    private Integer id;
    @Mapping("name")
    private String username;
    private Integer age;
}
```

```java
@Data
public class UsersVo {
    private Integer id;
    private String name;
    private Integer age;
}
```

> 测试

```java
@Autowired
private DozerBeanMapper dozerBeanMapper;
@Test
public void dozerTest () {
  Users users = new Users();
  users.setId(2);
  users.setUsername("张三");
  users.setAge(23);
  UsersVo usersVo = dozerBeanMapper.map(users, UsersVo.class);
  System.out.println(usersVo);
}/**output
UsersVo(id=2, name=张三, age=23)
*//~
```

可以看到，可以实现我们预期的结果～

##### xml实现映射

```java
@Configuration
public class DozerConfig {
    @Bean
    public DozerBeanMapper getDozerBean() {
        // 属性转换配置文件，可以有多个，后期如果一个文件过大，可以新增一个文件
        List<String> mappingFiles = Arrays.asList("dozer-conf.xml", "xxxx.xml");
        DozerBeanMapper dozerBean = new DozerBeanMapper();
        dozerBean.setMappingFiles(mappingFiles);
        return dozerBean;
    }
}
```

> 实体类

```java
@Data
public class Users {
    private Integer id;
    private String username;
    private Integer age;
}
```

```java
@Data
public class UsersVo {
    private Integer id;
    private String name;
    private Integer age;
}
```

可以看到， Users类中的username属性和UsersVo中的name名字是不一样的。

我们可以在resources目录下新建xml文件，我们这里以`dozer-conf.xml`文件为例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mappings xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://dozer.sourceforge.net"
          xsi:schemaLocation="http://dozer.sourceforge.net
          http://dozer.sourceforge.net/schema/beanmapping.xsd">

    <mapping>
        <class-a>com.maoyan.mybatisplusdemo.vo.Users</class-a>
        <class-b>com.maoyan.mybatisplusdemo.vo.UsersVo</class-b>
        <!-- 把username映射到name -->
        <field>
            <a>username</a>
            <b>name</b>
        </field>
    </mapping>
</mappings>

```

> 测试

```java
@Autowired
private DozerBeanMapper dozerBeanMapper;
@Test
public void dozerTest () {
  Users users = new Users();
  users.setId(2);
  users.setUsername("张三");
  users.setAge(23);
  UsersVo usersVo = dozerBeanMapper.map(users, UsersVo.class);
  System.out.println(usersVo);
}/**output
UsersVo(id=2, name=null, age=23)
*//～
```

可以看到， xml配置文件的方式实现了不同属性名之间的映射关系。

 