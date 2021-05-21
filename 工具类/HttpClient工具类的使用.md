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

下面的代码片段说明了使用HttpClient classic API执行HTTP GET和POST请求的过程。

```java
public class HttpGetDemo {
    public static void main(String[] args) throws Exception {
        try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
            HttpGet httpGet = new HttpGet("http://localhost:8080/test/get");
            /*
                底层HTTP连接仍由响应对象保留，以允许直接从网络套接字流式传输响应内容。
                为了确保正确释放系统资源，用户必须从finally子句中调用CloseableHttpResponse＃close（）。
                请注意，如果未完全使用响应内容，则无法安全地重用基础连接，连接管理器将关闭并丢弃该连接。
             */
            try (CloseableHttpResponse response1 = httpclient.execute(httpGet)) {
                System.out.println(response1.getCode() + " " + response1.getReasonPhrase());
                HttpEntity entity1 = response1.getEntity();
                // 对响应主体做一些有用的事情，并确保它被完全消耗
                String res = EntityUtils.toString(entity1);
                Result<List<User>> result = JSON.parseObject(res, new TypeReference<Result<List<User>>>() {});
                System.out.println(result);
                EntityUtils.consume(entity1);
            }
        
            HttpPost httpPost = new HttpPost("http://localhost:8080/test/post");
            List<NameValuePair> nvps = new ArrayList<>();
            // 设置请求参数
            nvps.add(new BasicNameValuePair("username", "vip"));
            nvps.add(new BasicNameValuePair("password", "secret"));
            httpPost.setEntity(new UrlEncodedFormEntity(nvps));
        
            try (CloseableHttpResponse response2 = httpclient.execute(httpPost)) {
                System.out.println(response2.getCode() + " " + response2.getReasonPhrase());
                HttpEntity entity2 = response2.getEntity();
                String res = EntityUtils.toString(entity2);
                Result<List<User>> result = JSON.parseObject(res, new TypeReference<Result<List<User>>>() {});
                System.out.println(result);
                // 对响应主体做一些有用的事情，并确保它被完全消耗
                EntityUtils.consume(entity2);
            }
        }
    }
}
```

#### fluent API

可以使用更简单，更灵活，更流畅的API来执行相同的请求

```java
public class FluentApiDemo {
    
    public static void main(String[] args) throws Exception {
        Response getResponse = Request.get("http://localhost:8080/test/get")
                .execute();
        // 获取Http相应码
//        System.out.println(getResponse.returnResponse().getCode());
        // 获取响应结果并设置解析字符编码
        String getResStr = getResponse.returnContent().asString(StandardCharsets.UTF_8);
        // 把json字符串转换为具体对象
        Result<List<User>> getResult = JSON.parseObject(getResStr, new TypeReference<Result<List<User>>>() {});
        System.out.println(getResult);
    
        Response postResponse = Request.post("http://localhost:8080/test/post")
                .bodyForm(Form.form().add("username", "vip").add("password", "secret").build())
                .execute();
        String postResStr = postResponse.returnContent().asString(StandardCharsets.UTF_8);
        Result<List<User>> postResult = JSON.parseObject(postResStr, new TypeReference<Result<List<User>>>() {});
        System.out.println(postResult);
    }
}
```

####  async API

下面的代码片段说明了使用HttpClient异步API执行HTTP请求的过程。

```java
public class AsyncApiDemo {
    public static void main(String[] args) throws Exception {
        try (CloseableHttpAsyncClient httpclient = HttpAsyncClients.createDefault()) {
            // Start the client
            httpclient.start();
        
            // Execute request
            SimpleHttpRequest request1 = SimpleHttpRequests.get("http://localhost:8080/test/get");
            Future<SimpleHttpResponse> future = httpclient.execute(request1, null);
            // and wait until response is received
            SimpleHttpResponse response1 = future.get();
            System.out.println(request1.getRequestUri() + "->" + response1.getCode());
            System.out.println(response1.getBody().getBodyText());
    
            // One most likely would want to use a callback for operation result
            final CountDownLatch latch1 = new CountDownLatch(1);
            final SimpleHttpRequest request2 = SimpleHttpRequests.get("http://localhost:8080/test/get");
            httpclient.execute(request2, new FutureCallback<SimpleHttpResponse>() {
                @Override
                public void completed(SimpleHttpResponse response2) {
                    latch1.countDown();
                    System.out.println(request2.getRequestUri() + "->" + response2.getCode());
                }
            
                @Override
                public void failed(Exception ex) {
                    latch1.countDown();
                    System.out.println(request2.getRequestUri() + "->" + ex);
                }
            
                @Override
                public void cancelled() {
                    latch1.countDown();
                    System.out.println(request2.getRequestUri() + " cancelled");
                }
            
            });
            latch1.await();
        
            // In real world one most likely would want also want to stream
            // request and response body content
            final CountDownLatch latch2 = new CountDownLatch(1);
            AsyncRequestProducer producer3 = AsyncRequestBuilder.get("http://localhost:8080/test/get").build();
            AbstractCharResponseConsumer<HttpResponse> consumer3 = new AbstractCharResponseConsumer<HttpResponse>() {
            
                HttpResponse response;
            
                @Override
                protected void start(HttpResponse response, ContentType contentType) throws HttpException, IOException {
                    this.response = response;
                }
            
                @Override
                protected int capacityIncrement() {
                    return Integer.MAX_VALUE;
                }
            
                @Override
                protected void data(CharBuffer data, boolean endOfStream) throws IOException {
                    // Do something useful
                }
            
                @Override
                protected HttpResponse buildResult() throws IOException {
                    return response;
                }
            
                @Override
                public void releaseResources() {
                }
            
            };
            httpclient.execute(producer3, consumer3, new FutureCallback<HttpResponse>() {
            
                @Override
                public void completed(HttpResponse response3) {
                    latch2.countDown();
                    System.out.println(request2.getRequestUri() + "->" + response3.getCode());
                }
            
                @Override
                public void failed(Exception ex) {
                    latch2.countDown();
                    System.out.println(request2.getRequestUri() + "->" + ex);
                }
            
                @Override
                public void cancelled() {
                    latch2.countDown();
                    System.out.println(request2.getRequestUri() + " cancelled");
                }
            
            });
            latch2.await();
        
        }
    }
}
```





