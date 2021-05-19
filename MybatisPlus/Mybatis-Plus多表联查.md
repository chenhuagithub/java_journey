## 教你如何优雅地使用Mybatis-Plus的多表联查

​	我想大家在工作中都会遇到这样的问题，就是想使用Mybatis的多表联查的功能，由于Mybatis-Plus没有封装多表联查的Api，所以我们只能使用xml的方式实现多表联查的功能点。 但是公司有自己的http请求模版，使用xml的方式的话需要使用动态sql解析模版，这对开发人员来说是极其不友好的，本文针对这个问题，讲述如何才能优雅地利用Mybatis-Plus已经提供的Api结合xm进行多表联查的实现。

​	在讲解Mybatis-Plus的多表联查之前，我们先来看看我们Mybatis原始的多表联查是如何实现的

#### xml实现多表联查及分页

##### 创建数据库

我们先来创建两个数据表

> film数据表

 ```mysql
create table film(
    film_id int primary key auto_increment comment 'id',
    film_name varchar(20) default '' not null comment '电影名',
    film_title varchar(20) default '' not null comment '标题'
)character set utf8;
 ```

> film_actor

```mysql
create table film_actor(
    actor_id int primary key auto_increment comment 'id',
    actor_name varchar(10) default '' not null comment '演员名字',
    film_id int not null comment '电影id'
)character set utf8;
```

##### 编写po实体类

> Film.java

```java
@TableName("film")
@Data
public class Film {
    @TableId(value = "film_id", type = IdType.AUTO)
    private Integer filmId;
    @TableField("film_name")
    private String filmName;
    @TableField("film_title")
    private String filmTitle;
}
```

> FilmActor.java

```java
@TableName("film_actor")
@Data
public class FilmActor {
    @TableId(value = "actor_id", type = IdType.AUTO)
    private Integer actorId;
    @TableField("actor_name")
    private String actorName;
    @TableField("film_id")
    private Integer filmId;
}
```

为了使用多表联查，但是为了代码不侵入定义好的po实体类，所以我们将新建一个关系类

> FilmAndActor.java

```java
@Data
public class FilmAndActor {
    
    private Integer filmId;
    private String filmName;
    private String filmTitle;
   	/**
   		关联FilmActor
   	*/
    private List<FilmActor> actors;
}
```

##### 在dao接口中自定义方法

```java
@Repository
public interface FilmMapper extends BaseMapper<Film> {
  
    List<FilmAndActor> pageQueryFilmAndActorFun(@Param("vo") FilmAndActorVo filmAndActorVo);
    
    
}
```

说明： `FilmAndActorVo`为请求http请求模版

```java
@Data
public class FilmAndActorVo extends PageQueryCondition implements Serializable {
    private String filmName;
    private String filmTitle;
    private String actorName;
}
```

```java
@Data
@Accessors(chain = true)
public class PageQueryCondition {
    private Integer pageSize;
    private Integer pageNo;
    private List<SortCondition> sortList;
}
```

```java
@Data
@Accessors(chain = true)
public class SortCondition {
    private String sortField;
    private boolean isAsc;
}
```

##### 编写xml映射文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.maoyan.mybatisplusproject.mapper.FilmMapper">

    <sql id="result_sql">
        film.film_id,
        film.film_name,
        film.film_title,
        actor.actor_id,
        actor.actor_name
    </sql>

    <resultMap id="filmResultMap" type="com.maoyan.mybatisplusproject.model.FilmAndActor">
        <id property="filmId" column="film_id"/>
        <result property="filmName" column="film_name"/>
        <result property="filmTitle" column="film_title"/>
        <collection property="actors" ofType="com.maoyan.mybatisplusproject.po.FilmActor">
            <result property="actorId" column="actor_id"/>
            <result property="actorName" column="actor_name"/>
        </collection>
    </resultMap>

    <select id="pageQueryFilmAndActorFun" resultMap="filmResultMap">
        select
            <include refid="result_sql"/>
        from
            film
        inner join
            film_actor as actor
        on
            film.film_id = actor.film_id
        <where>
            <if test="vo.filmName != null">
                film.film_name like ${vo.filmName}
            </if>
            <if test="vo.filmTitle != null">
                and film.film_title like ${vo.filmTitle}
            </if>
            <if test="vo.actorName != null">
                and actor.actor_name like ${vo.actorName}
            </if>
        </where>
        order by
        <if test="vo.sortList != null">
            <foreach collection="vo.sortList" item="item" index="index" separator=",">
                #{item.sortField}
                <if test="item.isAsc">
                    asc
                </if>
                <if test="!item.isAsc">
                    desc
                </if>
            </foreach>
        </if>
        <if test="vo.pageNo != null and vo.pageSize != null">
            limit #{vo.pageNo}, #{vo.pageSize}
        </if>
    </select>
</mapper>

```

在xml映射文件中使用`动态sql`解析传递进来的参数，然后拼接sql语句即可实现数据库的多表联查。

##### 测试

> 我们写个测试案例， 看看生成的sql是怎样的

```java
@Test
public void test5 () {
  FilmAndActorVo filmAndActorVo = new FilmAndActorVo();
  filmAndActorVo.setFilmName("'%04%'");
  filmAndActorVo.setFilmTitle("'%09%'");
  filmAndActorVo.setPageNo(1);
  filmAndActorVo.setPageSize(100);
  List<SortCondition> sortConditions = new ArrayList<>();
  SortCondition sortCondition = new SortCondition();
  sortCondition.setSortField("film_id").setAsc(true);
  SortCondition sortCondition1 = new SortCondition();
  sortCondition1.setSortField("film_name").setAsc(false);
  sortConditions.add(sortCondition);
  sortConditions.add(sortCondition1);
  filmAndActorVo.setSortList(sortConditions);
  List<FilmAndActor> filmAndActors = filmMapper.pageQueryFilmAndActorFun(filmAndActorVo);
  filmAndActors.forEach(System.out::println);
}
```

我们在控制台打印的sql语句为：

```mysql
==>  Preparing: select film.film_id, film.film_name, film.film_title, actor.actor_id, actor.actor_name from film inner join film_actor as actor on film.film_id = actor.film_id WHERE film.film_name like '%04%' and film.film_title like '%09%' order by ? asc , ? desc limit ?, ?
```

完美， 我们使用xml映射文件拼接sql的方式能够实现多表联查的功能并实现了分页查询，但是在我们日常开发的过程中，编写如此庞大而复杂的xml文件无论对我们的工作效率还是代码的可维护性都是不友好的， 那有没有什么办法能够简化xml映射文件的编写呢？欲知后事如何，请看下节～～～

#### xml结合Mybatis-Plus提供的Api实现多表联查及分页

Mybatis-Plus开发出来就是为了简化Mybatis的开发， 所以多表联查如果还用Mybatis那一套的话不免有点伤Mybatis-Plus的心了，下面，我们随便使用Mybatis-Plus的一个Api进行调试，看看会不会发现什么猫腻。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210120160938.png)

我们再来看看调试界面：

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210120161103.png)

只要仔细看，我们就可以发现，其实我们wrapper所拼接的所有sql语句都被保存到了`expression`变量里面了，我们再来看看Wrapper类：

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210120161524.png)

好家伙， 原来这里有个方法可以直接获取已经拼接好的sql语句，利用这一点，我们可以把之前在xml配置里面动态拼接的一些sql语句放在Wrapper里面拼接，然后我们可以调用这个方法并作为实参传递到xml映射文件中，那我们就可以省去编写复杂的动态sql语句了，下面我们就来试试吧

##### 编写xml映射文件

说明：关于数据库的创建及实体类都是沿用上面一节的。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.maoyan.mybatisplusproject.mapper.FilmMapper">

    <sql id="result_sql">
        film.film_id,
        film.film_name,
        film.film_title,
        actor.actor_id,
        actor.actor_name
    </sql>

    <resultMap id="filmResultMap" type="com.maoyan.mybatisplusproject.model.FilmAndActor">
        <id property="filmId" column="film_id"/>
        <result property="filmName" column="film_name"/>
        <result property="filmTitle" column="film_title"/>
        <collection property="actors" ofType="com.maoyan.mybatisplusproject.po.FilmActor">
            <result property="actorId" column="actor_id"/>
            <result property="actorName" column="actor_name"/>
        </collection>
    </resultMap>

    <select id="pageQueryFilmAndActor" resultMap="filmResultMap">
        select
            <include refid="result_sql"/>
        from
            film
        inner join
            film_actor as actor
        on
            film.film_id = actor.film_id
        ${ew.customSqlSegment}
    </select>
</mapper>

```

##### dao接口自定义方法

在Mybatis-Plus官网中，有这么一句话：

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210120162740.png)

因此我们在xml映射文件中无需做分页，我们可以调用Mybatis-Plus的分页Api就能够享用分页的福利了。

```java
@Repository
public interface FilmMapper extends BaseMapper<Film> {
   IPage<FilmAndActor> pageQueryFilmAndActor(IPage<FilmAndActor> page, @Param(Constants.WRAPPER) QueryWrapper<FilmAndActor> wrapper);
}
```

接下来， 我们开始使用wrapper拼接sql

##### wrapper拼接sql

```Java
@Repository
public class FilmAndActorRepo { 
    @Autowired
    private FilmMapper filmMapper;
    
    public IPage<FilmAndActor> pageQueryFilmAndActor(FilmAndActorVo pageCondition) {
        QueryWrapper<FilmAndActor> wrapper = new QueryWrapper<>();
        if (StringUtils.isNotEmpty(pageCondition.getFilmName())) {
            wrapper.like("film.film_name", pageCondition.getFilmName());
        }
        if (StringUtils.isNotEmpty(pageCondition.getFilmTitle())) {
            wrapper.like("film.film_title", pageCondition.getFilmName());
        }
        if (StringUtils.isNotEmpty(pageCondition.getActorName())) {
            wrapper.like("actor.actor_name", pageCondition.getActorName());
        }
        // 排序
        List<SortCondition> sortList = pageCondition.getSortList();
        if (CollectionUtils.isNotEmpty(sortList)) {
            MybatisPlusUtils.orderBy(wrapper, sortList);
        }
        IPage<FilmAndActor> page = new Page<>(pageCondition.getPageNo(), pageCondition.getPageSize());
        String sql = wrapper.getExpression().getNormal().getSqlSegment();
        System.out.println("sql=" + sql);
        return filmMapper.pageQueryFilmAndActor(page, wrapper);
    } 
}
```

```
@Service
public class FilmAndActorBiz {
    
    @Autowired
    private FilmAndActorRepo filmAndActorRepo;
    
    public List<FilmAndActor> pageQueryFilmAndActor (FilmAndActorVo filmAndActorVo) {
        IPage<FilmAndActor> filmAndActorIPage = filmAndActorRepo.pageQueryFilmAndActor(filmAndActorVo);
        return filmAndActorIPage.getRecords();
    }
}
```

注意： 在使用wrapper拼接sql的时候，拼接的key(比如film.film_name)一定要跟xml配置文件中的数据表别名一致，否则会报错。

编写好拼接方法后，我们下面来测试一番：

##### 测试

```java
@Test
public void test4 () {
  FilmAndActorVo filmAndActorVo = new FilmAndActorVo();
  filmAndActorVo.setFilmName("04");
  filmAndActorVo.setFilmTitle("09");
  filmAndActorVo.setPageNo(1);
  filmAndActorVo.setPageSize(100);
  List<SortCondition> sortConditions = new ArrayList<>();
  SortCondition sortCondition = new SortCondition();
  sortCondition.setSortField("film_id").setAsc(true);
  SortCondition sortCondition1 = new SortCondition();
  sortCondition1.setSortField("film_name").setAsc(false);
  sortConditions.add(sortCondition);
  sortConditions.add(sortCondition1);
  filmAndActorVo.setSortList(sortConditions);
  List<FilmAndActor> filmAndActors = filmAndActorBiz.pageQueryFilmAndActor(filmAndActorVo);
  System.out.println(filmAndActors.size());
  filmAndActors.forEach(System.out::println);
}
```

控制台打印的sql语句为：

```mysql
Preparing: select film.film_id, film.film_name, film.film_title, actor.actor_id, actor.actor_name from film inner join film_actor as actor on film.film_id = actor.film_id WHERE (film.film_name LIKE ? AND film.film_title LIKE ?) ORDER BY film_id ASC,film_name DESC LIMIT ?
```

看打印的sql语句， 是我们想要的， 含泪拿下。



至此， 我们对Mybatis-Plus进行多表联查就说到这里了， 小伙伴可以根据自己的爱好进行使用， 不过我个人比较喜欢wrapper拼接的这种方式，可读性比较强一些，而且代码比较容易维护。

