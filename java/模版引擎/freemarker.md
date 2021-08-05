# FreeMarker实战

​	Apache FreeMarker™ 是一个模板引擎：一个基于模板和变化数据生成文本输出（如HTML 网页、电子邮件、配置文件、源代码等）的 Java 库。模板是用 FreeMarker 模板语言 (FTL) 编写的，这是一种简单的专用语言（不是像 PHP 那样成熟的编程语言）。通常，使用通用编程语言（如 Java）来准备数据（发出数据库查询、进行业务计算）。然后，Apache FreeMarker 使用模板显示准备好的数据。在模板中，您专注于如何呈现数据，而在模板之外，您专注于要呈现的数据。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210701112958.png)

这种方法通常被称为 MVC（模型视图控制器）模式，尤其适用于动态网页。它有助于将网页设计者（HTML 作者）与开发者（通常是 Java 程序员）分开。设计人员不会面对模板中复杂的逻辑，并且无需程序员更改或重新编译代码即可更改页面的外观。

虽然 FreeMarker 最初是为在 MVC Web 应用程序框架中生成 HTML 页面而创建的，但它不受 servlet 或 HTML 或任何与 Web 相关的约束。它也用于非 Web 应用程序环境。



FreeMarker的具体介绍可以看官网：https://freemarker.apache.org/docs/dgui_quickstart.html



我们现在主要是用FreeMarker动态生成HTML页面和邮件模版进行简单的实战！！！

首先我们引入FreeMarker的Maven依赖

```xml
<dependency>
  <groupId>org.freemarker</groupId>
  <artifactId>freemarker</artifactId>
  <version>2.3.29</version>
</dependency>
```

然后对FreeMarker进行配置

```java

/**
 * @author chenhua11
 * @date 2021/7/1  10:53 上午
 */

@org.springframework.context.annotation.Configuration
public class FreeMarkerConfig {
    
    @Bean
    public Configuration freemarkerconfig() throws IOException {
        // Create your Configuration instance, and specify if up to what FreeMarker
        // version (here 2.3.29) do you want to apply the fixes that are not 100%
        // backward-compatible. See the Configuration JavaDoc for details.
        Configuration cfg = new Configuration(Configuration.VERSION_2_3_29);

        // Specify the source where the template files come from. Here I set a
        // plain directory for it, but non-file-system sources are possible too:
        // 配置模版所在的位置
        cfg.setDirectoryForTemplateLoading(new File("/Users/user/ideaproject/study-project/freemarker-project/src/main/resources/templates"));

        // From here we will set the settings recommended for new projects. These
        // aren't the defaults for backward compatibilty.

        // Set the preferred charset template files are stored in. UTF-8 is
        // a good choice in most applications:
        cfg.setDefaultEncoding("UTF-8");

        // Sets how errors will appear.
        // During web page *development* TemplateExceptionHandler.HTML_DEBUG_HANDLER is better.
        cfg.setTemplateExceptionHandler(TemplateExceptionHandler.RETHROW_HANDLER);

        // Don't log exceptions inside FreeMarker that it will thrown at you anyway:
        cfg.setLogTemplateExceptions(false);

        // Wrap unchecked exceptions thrown during template processing into TemplateException-s:
        cfg.setWrapUncheckedExceptions(true);

        // Do not fall back to higher scopes when reading a null loop variable:
        cfg.setFallbackOnNullLoopVariable(false);
        
        return cfg;
    }

}
```

该配置是直接从官网中拉取的（https://freemarker.apache.org/docs/pgui_quickstart_createconfiguration.html）

配置工作已经准备就绪，下面进行一些简单的测试



#### 动态生成html页面

首先，我们要在上面配置的模版目录下创建一个模版文件（test.html）：

```html
<html>
    <head>
        <title>freemarker</title>
    </head>
    <body>
        <p>welcome ${user.username}</p>
    </body>
</html>
```

单元测试：

```java
@Autowired
private Configuration cfg;

@Test
void contextLoads() throws IOException, TemplateException {

  Template temp = cfg.getTemplate("test.html");
  StringWriter writer = new StringWriter();
  Map<String, User> root = new HashMap<>();
  User user = new User();
  user.setUsername("chenhua11");
  root.put("user", user);
  temp.process(root,writer);
  System.out.println(writer.toString());
}/** output
<html>
    <head>
        <title>freemarker</title>
    </head>
    <body>
        <p>welcome chenhua11</p>
    </body>
</html>
*//~
```

可以看到，我们已经成功动态渲染出html模版了



#### 动态生成邮件

​	在我们平时的开发中，不同用户使用的合同模板一样，只是用户信息不一样，静态模板可在后台进行维护操作；用户在前台进行查看时将用户的信息动态渲染到静态模板上再到前台进行展示。但页面是静态页面不利于维护，弊端就是每次合同发生更改，需要开发人员进行更改、发布、上线；关键工作繁琐毫无技术含量，这是对资源的一种严重浪费。

​	于是将静态模板保存在数据库中，管理员可以在文本编辑器中进行维护，现在问题的关键是如何将带参数的模板将参数动态渲染为用户信息并输出：



当然，我们不能使用传统文件路径加载模版的方式加载模版，因此我们需要改变FreeMarker的模版加载器，FreeMarker给我们提供了几种模版加载器，如下：

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210701120027.png)

`StringTemplateLoader`这个模版加载器可以让我们直接加载字符串模版，我们也就可以从数据库中加载出相应的邮件模版进行处理，代码如下：

```java
@Autowired
private Configuration cfg;

@Test
void test1() throws IOException, TemplateException {
  // 从数据库中获取，这里为了方便就自己定义了
  String content = "客户：${username}"; 
  StringTemplateLoader templateLoader = new StringTemplateLoader();
  templateLoader.putTemplate("emailTemplate", content);
  cfg.setTemplateLoader(templateLoader);
  Template temp = cfg.getTemplate("emailTemplate", "uft-8");
  System.out.println(temp);
  User user = new User();
  user.setUsername("张三");
  OutputStreamWriter out = new OutputStreamWriter(System.out);
  temp.process(user, out);
}/** output
客户：${username}
客户：张三
*//～
```

可见，模版渲染成功。























