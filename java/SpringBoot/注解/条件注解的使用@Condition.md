## @Conditional的使用

> 条件注入作为Spring框架提供给开发者的高级特性而存在,开发者希望能针对某些特定的条件满足的情况下，才注入Bean到Spring的容器中,这种特性提供了很好的可扩展性。

针对条件注入,Spring提供了`@Conditional`注解来解决这个问题,`@Conditional`的源码如下：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
	/**
	 * All {@link Condition} classes that must {@linkplain Condition#matches match}
	 * in order for the component to be registered.
	 */
	Class<? extends Condition>[] value();

}
```

`@Condtional`注解提供了一个属性value,该属性声明了一个`Condition`的class，`Condition`是Spring提供的接口

源码：

```java
public interface Condition {
	/**
	 * Determine if the condition matches.
	 * @param context the condition context
	 * @param metadata the metadata of the {@link org.springframework.core.type.AnnotationMetadata class}
	 * or {@link org.springframework.core.type.MethodMetadata method} being checked
	 * @return {@code true} if the condition matches and the component can be registered,
	 * or {@code false} to veto the annotated component's registration
	 */
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

从源码可以得知,`@Conditional`注解可以作用于类、方法，只要提供的Condition全部满足的情况下,才会将实体Bean注入到所有的容器中.

如果`@Conditional`作用于拥有`@Configuration`注解的类上,那么该类下的所有Bean的创建注入都需要满足`@Conditional`注解的条件才可以注入

如果`@Conditional`作用于方法上,那么该方法需要注入Bean时,只有满足了条件的情况下才会注入.

Spring Boot为我们提供了很多默认的`Condition`实现类,通过默认提供的Condition基本可以满足我们的日常需求,如果不满足,开发者可自定义Condtion的实现开发自己的装载Bean需求。

接下来,先看Spring Boot为我们提供的默认Condition实现,包径：`org.springframework.boot.autoconfigure.condition`

常用应用程序使用注解，主要包含以下:

| 注解                         | 说明                         |
| -------------------------- | -------------------------- |
| @ConditionalOnProperty     | 根据特定的属性进行条件注入              |
| @ConditionalOnExpression   | 根据SPEL表达式组合复杂情况,满足的情况下条件注入 |
| @ConditionalOnBean         | 根据容器中存在外部某个实体Bean的情况下条件注入  |
| @ConditionalOnMissingBean  | 容器中不存在某个实体Bean的情况下条件注入     |
| @ConditionalOnClass        |                            |
| @ConditionalOnMissingClass |                            |

下面，我们逐个来了解这些注解！

####  @ConditionalOnProperty

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520102003.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520102537.png)

条件解析是`application.properties`或者`application.yml`文件中`mybean.enable=true`才会分别加载MyBeanComponent这个bean，`mybean.enable=false`或者缺失`mybean.enable`配置都不能加载MyBeanComponent这个Bean。

但是如果设置`matchIfMissing=true`

则在application.propertie或者application.yml缺失user.bean配置时加载MyBeanCompo这个Bean。

#### @ConditionalOnBean 和 ConditionalOnMissingBean

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520103152.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520103442.png)如果OtherBean 已经存在应用上下文时才会加载Mybean这个Bean。同理，如果是`ConditionalOnMissingBean`则是上下文不存在OtherBean这个bean才会加载MyBean这个bean。

#### @ConditionalOnClass 和 @ConditionalOnMissingClass

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520103736.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520103849.png)

根据某个类是否存在于 classpath 中来判断是否加载MyBean这个bean,`@ConditionalOnMissingClass`刚好于`@ConditionalOnClass`作用相反。

#### @ConditionalOnExpression

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520104816.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210520104908.png)

只有当`mybean.enable`和`otherbean.enable`两个属性都为 true 的时候才加载 MyBeanComponent这个bean。







