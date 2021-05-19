## HttpClient

> HttpClient提供了对Http协议的强大支持，感兴趣的童鞋可以直接阅读官方文档：http://hc.apache.org/httpcomponents-client-5.0.x/index.html

Apache主要给HttpClient提供了三个使用的模块：

- classic  API
- fluent API
- async API

首先，我们需要引入HttpClient的maven依赖

```xml
<dependency>
  <groupId>org.apache.httpcomponents.client5</groupId>
  <artifactId>httpclient5</artifactId>
  <version>5.0</version>
</dependency>
<dependency>
  <groupId>org.apache.httpcomponents.client5</groupId>
  <artifactId>httpclient5-fluent</artifactId>
  <version>5.0</version>
</dependency>
<dependency>
  <groupId>org.apache.httpcomponents.client5</groupId>
  <artifactId>httpclient5-cache</artifactId>
  <version>5.0</version>
</dependency>
```

首先，我们来分别编写一个get请求和post请求作为测试案例

```java
@RestController
public class TestController {
    
    @GetMapping("/test/get")
    public Result<List<User>> testGet () {
        User user1 = new User(1, "111", "111");
        User user2 = new User(2, "222", "222");
        List<User> res = new ArrayList<>();
        res.add(user1);
        res.add(user2);
        return new Result<>(200, "post请求获取数据成功", res);
    }
    
    @PostMapping("/test/post")
    public Result<List<User>> testPost (String username, String password) {
        User user1 = new User(3, username, password);
        List<User> res = new ArrayList<>();
        res.add(user1);
        return new Result<>(200, "get请求获取数据成功", res);
    }
}
```

#### classic  API









