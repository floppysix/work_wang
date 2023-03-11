- 配置文件 https://blog.csdn.net/qq_26707371/article/details/101467644


## 1.  MybatisPlus官网

官网：

```css
https://baomidou.com/

```

## 2. 入门案例

开发环境：

-   开发工具：IDEA
-   构建工具：Maven
-   数据库：MySQL
-   项目框架：SpringBoot

**引入依赖**

```xml
<!-- mybatis-plus 依赖-->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.0</version>
</dependency>
<!-- mysql驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

**创建数据库表**

user 表：

```sql
CREATE TABLE `user` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT 'id',
  `name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL COMMENT '姓名',
  `age` int DEFAULT NULL COMMENT '年龄',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1508421137384648706 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=DYNAMIC;
复制代码
```

**实体类**

```java
public class User {
    @TableId(value = "id", type = IdType.ASSIGN_ID)
    private Long id;
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

**修改 application.yml**

```yml
server:
  port: 8082
  servlet:
    context-path: /mybatisplus_demo
# 数据源配置
spring:
  datasource:
    username: root
    password: 12345678
    url: jdbc:mysql://localhost:3306/ssm?allowPublicKeyRetrieval=true&useSSL=false
    driver-class-name: com.mysql.cj.jdbc.Driver
```

**新建 UserMapper 接口** 
![[506d7b96d90042a78df4173cd1b45fcc_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
**注**：​​
(1)因为 mybatis 规定：mapper.xml 文件的名字要和接口名字一样，所以很多人习惯将 Dao 接口命名为 xxxMapper​​。

(2)BaseMapper 是 MybatisPlus 内置的接口，它包含基本的 CRUD 方法。

**启动类添加 @MapperScan 注解** 
![[a406cda92fe7496db30aab5412e6fb79_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
**测试**

```java
@SpringBootTest
public class MybatisPlusDemoApplicationTests {

    @Resource
    private UserMapper userMapper;

    @Test
    void testMybatisPlus() {
        for (int i = 18; i < 20; i++) {
            User user = new User("王小波" + i, i);
            userMapper.insert(user);
        }
    }
}

```
​

## 3. 基本增删改查

1.新增

```java
User user = new User("王小波", 19);
userMapper.insert(user);

```

2.编辑

根据 id 更新数据

```java
int rows = userMapper.updateById(user);
if(rows>0){
    System.out.println("更新成功"); 
}

```

3.删除

根据主键删除信息

```java
userMapper.deleteById("152635612");

```

根据 map 条件删除信息

```java
Map<String, Object> param = new HashMap<>();
param.put("age", 18);
int rows = userMapper.deleteByMap(param);
if (rows > 0) {
    System.out.println("删除成功！");
}

```

根据 id 集合批量删除

```java
List<Integer> ids = Stream.of(110, 112, 113, 115).collect(Collectors.toList());
int rows = userMapper.deleteBatchIds(ids);
if (rows > 0) {
  System.out.println("删除成功！");
}

```

4.查询

根据 id 查询

```java
User user = userMapper.selectById(152382374);

```

根据 map 条件查询

```java
Map<String, Object> param = new HashMap<>();
param.put("age", 18);
List<User> userList = userMapper.selectByMap(param);

```

根据 id 集合批量查询

```java
List<Integer> ids = Stream.of(110, 112, 113, 115).collect(Collectors.toList());
List<User> userList = userMapper.selectBatchIds(ids);

```

## 4. 构造器

> MybatisPlus 提供了查询构造器和更新构造器用来生成带有 where 条件的 sql 语句。

(1)封装查询条件的构造器：

```css
QueryWrapper
复制代码
```

常用查询条件：

等于：eq

```java
QueryWrapper<User> userWrapper = new QueryWrapper<>();
// 查询名字是张三的用户
userWrapper.eq("name","张三");
List<User> userList = userMapper.selectList(userWrapper);

```

不等于：ne

```java
QueryWrapper<User> userWrapper = new QueryWrapper<>();
userWrapper.ne("name","张三");
// 查询名字不是张三的用户
List<User> userList = userMapper.selectList(userWrapper);

```

模糊查询：like

```java
QueryWrapper<User> userWrapper = new QueryWrapper<>();
// 模糊查询
userWrapper.like("name","张");
List<User> userList = userMapper.selectList(userWrapper);

```

降序：orderByDesc

```java
QueryWrapper<User> userWrapper = new QueryWrapper<>();
// 模糊查询并根据 number 倒序
userWrapper.like("name","张").orderByDesc("number");
List<User> userList = userMapper.selectList(userWrapper);

```

升序：orderByAsc

```java
QueryWrapper<User> userWrapper = new QueryWrapper<>();
// 模糊查询并根据 number 降序
userWrapper.like("name","张").orderByAsc("number");
List<User> userList = userMapper.selectList(userWrapper);

```

其他常用的条件可以去官网查看相关文档，这里不再过多赘述：

```css
https://baomidou.com/pages/10c804/#in
```
​

(2)封装更新条件的构造器：

```css
UpdateWrapper

```

UpdateWrapper 的 where 条件和 QueryWrapper 的一样，只不过需要 set 值。

```java
UpdateWrapper<User> userWrapper = new UpdateWrapper<>();
userWrapper.set("name","王小波").set("age",22)
    .eq("name","张三");

```

## 5. 通用 Service

MybatisPlus 中有一个通用的接口 Iservice 和实现类，封装了常用的增删改查等操作。

1.新建 service 和 实现类

UserService

```java
public interface UserService extends IService<User> {

}

```

UserServiceImpl

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

}

```


## 6. 常用注解

1.@TableIdmailto:1.@TableId

MybatisPlus 会默认将实体类中的 id 作为主键。

@TableId 表示 id 的生成策略，常用的有两种：

(1) 基于数据库的自增策略
![[eff23f713aa64c63b88355aa1226653f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
(2) 使用雪花算法策略随机生成 
![[82bd26dede1f4b79a5ebc700e94dfd58_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
2.@TableNamemailto:2.@TableName

如果实体类和数据库的表名不一致，可以使用这个注解做映射

例如： 
![[1b65832aa827424285e74e98fab4f527_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]

3.@TableFieldmailto:3.@TableField

当表属性和实体类中属性名不一致时，可以使用这个注解做映射： 
![[cad3aba7f7c74d07b51c8aa23034529f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
## 7. 分页

MybatisPlus 内部封装了分页插件，只用简单配置一下就能实现分页功能。

1.配置类

```java
@Configuration
@MapperScan("com.zhifou.mapper")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new
                PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

2.测试

```java
@Test
void testMybatisPlus() {
    int current = 1;
    int size = 10;
    Page<User> userPage = new Page<>(current, size);
    //获取分页数据
    List<User> list = userPage.getRecords();
    list.forEach(user->{
        System.out.println(user);
    });
    Page<User> page = userMapper.selectPage(userPage, null);
    System.out.println("当前页:" + page.getCurrent());
    System.out.println("每页条数:" + page.getSize());
    System.out.println("总记录数:" + page.getTotal());
    System.out.println("总页数:" + page.getPages());
}

```

## 8. 代码生成器

MybatisPlus 可以帮助我们自动生成 controller、service、dao、model、mapper.xml 等文件，极大地提高了我们的开发效率。

1.引入依赖

```xml
<!-- 代码生成器 -->
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-generator</artifactId>
  <version>3.5.1</version>
</dependency>
<dependency>
  <groupId>org.freemarker</groupId>
  <artifactId>freemarker</artifactId>
</dependency>
复制代码
```

2.代码生成器工具类

```java
public class CodeGenerator {
public static void main(String[] args) {
// 连接数据库
FastAutoGenerator.create("jdbc:mysql://localhost:3306/ssm?allowPublicKeyRetrieval=true&useSSL=false&useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC", "root", "123456")
    .globalConfig(builder -> {
        builder.author("知否技术") // 设置作者
                .fileOverride() // 覆盖已生成文件
                // 设置日期时间
                .dateType(DateType.ONLY_DATE)
                .outputDir("D:\\WorkSpace\\idea\\mybatisplus_demo\\src\\main\\java"); // 指定输出目录
    })
    .packageConfig(builder -> {
        builder.parent("com.zhifou") // 设置父包名
                .pathInfo(Collections.singletonMap(OutputFile.mapperXml, "D:\\WorkSpace\\idea\\mybatisplus_demo\\src\\main\\resources\\mapper")); // 设置mapperXml生成路径
    })
    .strategyConfig(builder -> {
        builder.addInclude("t_user") // 设置需要生成的表名
                .addTablePrefix("t_"); // 设置过滤表前

        // 新增数据，自动为创建时间赋值
        IFill createFill = new Column("created_date", FieldFill.INSERT);
        IFill updateFill = new Column("updated_date", FieldFill.UPDATE);
        builder.entityBuilder()
                // 设置id类型
                .idType(IdType.ASSIGN_ID)
                // 开启 Lombok
                .enableLombok()
                // 开启连续设置模式
                .enableChainModel()
                // 驼峰命名模式
                .naming(NamingStrategy.underline_to_camel)
                .columnNaming(NamingStrategy.underline_to_camel)
                // 自动为创建时间、修改时间赋值
                .addTableFills(createFill).addTableFills(updateFill)
                // 逻辑删除字段
                .logicDeleteColumnName("is_deleted");

        // Restful 风格
        builder.controllerBuilder().enableRestStyle();
        // 去除 Service 前缀的 I
        builder.serviceBuilder().formatServiceFileName("%sService");
        // mapper 设置
        builder.mapperBuilder()
                .enableBaseResultMap()
                .enableBaseColumnList();
    })
    // 固定
    .templateEngine(new FreemarkerTemplateEngine()) // 使用Freemarker引擎模板，默认的是Velocity引擎模板
    .execute();
  }
}

```

关键点：

(1)配置数据库连接信息。

自动生成代码需要连接数据库 
![[a3146c09f1304176bfe515b9c304807c_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
(2)指定输出目录，这里直接设置你项目的目录，到时候不用赋值粘贴了。 
![[f15d8f717fea43c5b7aef84d283e0d2b_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
(3)设置父包名。
![[ff391f2df2bf4466a03234ab91c9f7f8_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
(4)设置表名 
![[3c0f5756f13a4cc28274fcfb7dd85af2_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.jpg]]
然后右键运行，代码就会自动生成。

## 9. application.yml 配置

```ymal
# MybatisPlus
mybatis-plus:
  global-config:
    db-config:
      column-underline: true # 驼峰形式
      logic-delete-field: isDeleted # 全局逻辑删除的实体字段名
      logic-delete-value: 1 # 逻辑已删除值(默认为 1)
      logic-not-delete-value: 0 # 逻辑未删除值(默认为 0)
      db-type: mysql
      id-type: assign_id # id策略
      table-prefix: t_ # 配置表的默认前缀 
  mapper-locations: classpath*:/mapper/**Mapper.xml # mapper 文件位置
  type-aliases-package: com.zhifou.entity # 实体类别名
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl # 日志：打印sql 语句

```



## 10. 遇到的坑

1.传参为 0 时，查询语句失效。

例如传递的 age 为 0，查询就会失效

```bash
<select id="getUser" resultType="user">
    select id,name,age,sex from user
    <where>
        <if test="age != null and age !='' ">
            age = #{age}
        </if>
    </where>
</select>

```

原因：判断 int 是否为空只要 !=null 就行了，如果加上 type != ''，0 会被转为 null​​。

2.MybatisPlus 更新字段为 null 失败

解决办法：

```java
@TableField(updateStrategy = FieldStrategy.IGNORED)
private String name;

```

该注解会忽略为空的判断，






