# springboot 配置顺序问题

## Spring配置机制简介

​ 为了找到问题发生的原因，首先需要了解配置是如何在SpringBoot项目中生效的。查阅资料后，我知道了在SpringBoot中，存在一个名为**Application**的变量，其中保存着Spring中启动的所有信息。在这所有的变量中，配置信息主要同变量**Environment**相关，诸如**JVM参数、环境变量、Apollo配置**等配置用**PropertySource**封装后，存放在**Environment**里的。

​ 除了存储配置以外，SpringBoot还设计了**propertyResolver**用于管控当前的配置信息，并负责对配置进行填充。
![[f33a80d378dd4400a9c362daf6adf998_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]



​ 至于**PropertyResolver**和**PropertySource**的关系，形象点来说，**PropertyResolver**就是一位翻译官，他会根据现有的词典**PropertySource**对我们的语言${xxx.url}做翻译，并最终得到所配置的信息。倘若字典中没有对应的信息，那么很自然"翻译官"是无法做出翻译的。

![[2337e4ffb33e46f184cd7156f9d1c188_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]

因此，不难分析问题的原因应该是**切换写法后，配置发生了加载顺序上的变化，使得配置解析先于apollo里配置加载，从而出现解析失败的情况**。

## 配置加载顺序梳理

​ 认识到问题原因可能是由于配置加载顺序导致的，我们需要**对Apollo、@Value、@FeignClient三者的配置加载顺序**进行了解。

### Apollo加载顺序梳理

​ 首先我们来了解Apollo的配置加载顺序，结合[Apollo的文档中的内容](https://link.juejin.cn/?target=https%3A%2F%2Fwww.apolloconfig.com%2F%23%2Fzh%2Fusage%2Fjava-sdk-user-guide%3Fid%3D_32-spring%25e6%2595%25b4%25e5%2590%2588%25e6%2596%25b9%25e5%25bc%258f "https://www.apolloconfig.com/#/zh/usage/java-sdk-user-guide?id=_32-spring%e6%95%b4%e5%90%88%e6%96%b9%e5%bc%8f")，不难得到apollo配置的加载顺序会有三种情况：
|apollo.bootstrap.enabled|apollo.bootstrap.eagerLoad.enabled|对应SpringBoot的运行阶段|  
|----|----|---|  
|true|true|prepareEnvironment|
|true|false|prepareContext|
|false|false|refreshContext|



​ 这里简单介绍下这三种情况对应的Springboot运行阶段分别负责的功能是：

1.  _**prepareEnvironment**_，是最早加载配置的地方，**bootstrap.yml配置**、**系统启动参数中的环境变量**都会在这个阶段被加载。
2.  _**prepareContext**_，主要对上下文做初始化，如**设置bean名字命名器、设置加载.class文件加载器**等。
3.  _**refreshContext**_，该阶段主要负责对bean容器进行加载，包括扫**描文件得到BeanDefinition和BeanFactory工厂、Bean工厂生产Bean对象、对Bean对象再进行属性注入**等工作。

​ 这三个阶段在现有SpringBoot启动过程中顺序如下所示：

![[0e62d42567844dc6b69b883072511480_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]


#### prepareEnviroment

​ 在_preparenEnvironment_阶段，Spring会发出异步消息**ApplicationEnvironmentPreparedEvent**，同时名为**ConfigFileApplicationListener**对象会监听该消息，并对实现了`EnvironmentPostProcessor`接口的对象进行调用。
![[cb39922824c641b695ced5e022536b72_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]

​ 在Apollo源码中，**ApolloApplicationContextInitializer**类也实现了`EnvironmentPostProcessor`的接口。其实现方法中进行apollo配置的加载。
![[dd804ade6122488db7f685d23b40f7f4_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]


#### prepareContext

​ 在_prepareContext_的阶段，主要依赖于方法_applyInitializers_。该方法会对所有实现了`ApplicationContextInitializer`接口的对象进行调用。在Apollo中，**ApolloApplicationContextInitializer**类也实现了该接口，并在方法中进行配置加载。
![[770a5c75e5234d32a21452da034b0a9b_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]

#### refreshContext

​ _refreshContext_为Apollo的默认加载阶段。在_refreshContext_中，会调用_invokeBeanFactoryPostProcessors_方法对实现了`BeanFactoryPostProcessor`接口的对象进行调用。在apollo源码中，对象**PropertySourcesProcessor**就实现了该接口。且该对象在_postProcessBeanFactory_方法中，进行了对配置信息的加载。

![[f698dc0902e0466980501f7fcf6d4181_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]


#### 小结

由此梳理下来，Apollo三个阶段的加载顺序及配置控制逻辑，如下图所示：
![[762d994f1446422587fefb62ec82eab1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]


### @Value 加载顺序梳理

​ 了解了apollo的加载顺序后。我们要了解下@Value的加载顺序，@Value的实现思想很纯粹，**当你的Bean对象创建好后，我再把属性通过getter、setter方法注入进去**，就实现注入的功能。

​ 因此@Value的实现主要在Bean生成后。在_refreshContext_阶段，会调用_finishBeanFactoryInitialization_方法对所有单例bean对象做初始化逻辑。其中在**AbstractAutowireCapableBeanFactory**会有一个方法_populateBean_，其会对bean属性做填充。同上述类似，这里也会对所有继承了`BeanPostProcessor`接口的对象进行调用。其中包含一个特殊的对象**AutowiredAnnotationBeanPostProcessor**
![[96ed52eac50b4c0a90c4b5d0a93bedcf_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]


​ **AutowiredAnnotationBeanPostProcessor**会将用@Value注解修饰的对象扫描出来，并从配置中找到对应的配置信息，注入到对象中。结合上述apollo配置加载顺序图，我们可以得到@Value和Apollo的配置优先级大概如下所示：
![[610132384a164f878f93d3c84021492a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]

​ 可以看到，@Value的配置晚于apollo的配置，因此在切换写法前，apollo的配置可以被正常注入。

### @FeignClient 加载顺序梳理

​ 了解完@Value的加载顺序后，我们还需要了解下@FeignClient的配置加载顺序。对于FeignClient来说，它通常采用接口做实现，因此需要根据@FeignClient生成新的Bean对象，并注册到容器中。因此，其配置的加载顺序在Bean对象生成之前。

​ 类**ConfigurationClassPostProcessor**继承自接口`AutowiredAnnotationBeanPostProcessor`，其_postProcessBeanDefinitionRegistry_方法会对BeanDefinition做注入处理。（BeanDefinition，简写为BeanDef，是Bean容器未生成的形态，如果将Bean比作一辆汽车，那么BeanDefinition就是汽车的图纸。）

​ 同时，类**ConfigurationClassBeanDefinitionReader**会调用_loadBeanDefinitionsFromRegistrars_方法，该方法会将实现了`ImportBeanDefinitionRegistrar`接口的对象逐一进行调用。这其中包含一个**FeignClientsRegistrar**对象，其实现的_registerFeignClients_方法会扫描所有被@FeignClient注解的对象。
![[73eabe09baad4e70ae0bce63364b6bd3_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]

​ 同时，对单个BeanDef对象，还会调用**FeignClientsRegistrar**下的_registerFeignClient_方法做处理，将我们其中的url、path等属性都用propertyResolver做翻译处理，倘若此时，配置中不存在相应的属性，就不会更新。**这就是造成本次问题的关键点。**
![[d62a1b80e0a4483e93dd57d3ebc971c6_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]


​ 关注到加载顺序上，@FeignClient注解所依赖的接口为`BeanDefinitionRegistryPostProcessor`，而Apollo中默认加载的情况则依赖于`BeanFactoryPostProcessor`接口。两者几乎在同一处方法调用内，但`BeanDefinitionRegistryPostProcessor`接口执行稍微先于`BeanFactoryPostProcessor`。因此在加载顺序上，**@FeignClient会先于默认情况下的Apollo加载**。
![[28dbcf1e661448ef85ca1a3cd0a2d8e5_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp]]

​至此也就不难理解为什么Apollo注解没法生效了。因为在@FeignClient注解的情况下，beanDef注入时，apollo的配置还没有加载，**PropertyResolver**找不到对应的配置，自然也就无法进行注入了。


# 时间相关
## 在MySQL中表示时间的类型
![[Pasted image 20220814150141.png]]

## TIMESTAMP和DATETIME比较
- 相同点
两者都可用来表示YYYY-MM-DD HH:MM:SS 类型的日期。
- 不同点：
他们的的存储方式，大小(字节)，表示的范围不同。
TIMESTAMP，它把客户端插入的时间从当前时区转化为UTC（世界标准时间）进行存储。查询时，将其又转化为客户端当前时区进行返回。
DATETIME，不做任何改变，基本上是原样输入和输出。

**总结**：TIMESTAMP和DATETIME 都可用来表示YYYY-MM-DD HH:MM:SS 类型的日期， 除了存储方式和存储范围以及大小不一样，没有太大区别。但对于跨时区的业务，TIMESTAMP更为合适。

##  时间与时间戳之间转换
有些应用生成的时间戳是比这个多出三位，是毫秒表示，如果要转换，需要先将最后三位去掉（标准的10位数字，如果是13位的话可以以除以1000的方式），否则返回NULL

    #将时间转换为时间戳unix_timestamp
    SELECT UNIX_TIMESTAMP('2019-02-22 13:25:07'); #1550813107
     
    #将时间戳转换为时间from_unixtime
    SELECT FROM_UNIXTIME(1550813107); #2019-02-22 13:25:07
     
    #NOW
    SELECT UNIX_TIMESTAMP(NOW()); #1550813420
    SELECT FROM_UNIXTIME(1550813420); #2019-02-22 13:30:20

## 前端后端传输时间
### 1.  使用@DateTimeFormat和@JsonFormat注解
	@DateTimeFormat是前端往后段传的时候使用，加在[实体类](https://so.csdn.net/so/search?q=%E5%AE%9E%E4%BD%93%E7%B1%BB&spm=1001.2101.3001.7020)中，然后controller中直接使用这个实体类接收参数。当前端传固定格式的字符串的时候会转换成date ； @JsonFormat是后端往前端传输的时候使用。

```java
@Data
public class SearchDTO {
	// 前端需要传这种格式的字符串
	@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
	@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
	private Date time;
}
```

### 2. 配置文件中统一配置（尝试后并没有作用）
```properties
spring:
  mvc:
    date-format: yyyy-MM-dd HH:mm:ss
    throw-exception-if-no-handler-found: true
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    serialization:
      write-dates-as-timestamps: true
```

### 3. converter手动转换（用于时间戳的转换）
	参考：https://www.cnblogs.com/eknown/p/11968101.html
	https://blog.csdn.net/qq_36963762/article/details/117085922

#### 3.1 需要实现Converter接口，重写其中的convert方法，将string类型转化为Date类	
```java
/**
 * 日期转换类
 * 将标准日期、标准日期时间、时间戳转换成Date类型
 */
/*@Deprecated*/
public class DateConverter implements Converter<String, Date> {

    private Logger logger = LoggerFactory.getLogger(DateConverter.class);

    private static final String dateFormat = "yyyy-MM-dd HH:mm:ss";
    private static final String shortDateFormat = "yyyy-MM-dd";
    private static final String timeStampFormat = "^\\d+$";

    @Override
    public Date convert(String value) {
        logger.info("转换日期：" + value);

        if(value == null || value.trim().equals("") || value.equalsIgnoreCase("null")) {
            return null;
        }

        value = value.trim();

        try {
            if (value.contains("-")) {
                SimpleDateFormat formatter;
                if (value.contains(":")) {
                    formatter = new SimpleDateFormat(dateFormat);
                } else {
                    formatter = new SimpleDateFormat(shortDateFormat);
                }
                return formatter.parse(value);
            } else if (value.matches(timeStampFormat)) {
                Long lDate = new Long(value);
                return new Date(lDate);
            }
        } catch (Exception e) {
            throw new RuntimeException(String.format("parser %s to Date fail", value));
        }
        throw new RuntimeException(String.format("parser %s to Date fail", value));
    }
}

```

#### 3.2 将DateConverter注册
```
import com.aegis.yqmanagecenter.config.date.DateConverter;
import com.aegis.yqmanagecenter.model.bean.common.JsonResult;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.propertyeditors.CustomDateEditor;
import org.springframework.core.convert.support.GenericConversionService;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;

import java.beans.PropertyEditorSupport;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;

@ControllerAdvice
public class ControllerHandler {

    private Logger logger = LoggerFactory.getLogger(ControllerHandler.class);

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 方法1，注册converter
        GenericConversionService genericConversionService = (GenericConversionService) binder.getConversionService();
        if (genericConversionService != null) {
            genericConversionService.addConverter(new DateConverter());
        }

        // 方法2，定义单格式的日期转换，可以通过替换格式，定义多个dateEditor，代码不够简洁
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        CustomDateEditor dateEditor = new CustomDateEditor(df, true);
        binder.registerCustomEditor(Date.class, dateEditor);


        // 方法3，同样注册converter
        binder.registerCustomEditor(Date.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                setValue(new DateConverter().convert(text));
            }
        });

    }
}

```
#### 3.3 我们controller里用@RequestParam就能直接接收时间戳转化成Date
```java
public Result<List<InvoiceVO>> invoiceList(
            @RequestParam(required = false) LocalDateTime beginTime) {
}
```
#### 3.4 @RequestBody注解里面的时间类型现在还不可以转换
```java
/**
 * 日期转换配置
 * 解决@RequestAttribute、@RequestParam和@RequestBody三种类型的时间类型参数接收与转换问题
 */
@Configuration
public class DateConfig {

    /**
     * 默认日期时间格式
     */
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";

    /**
     * Date转换器，用于转换RequestParam和PathVariable参数
     */
    @Bean
    public Converter<String, Date> dateConverter() {
        return new DateConverter();
    }

    /**
     * Json序列化和反序列化转换器，用于转换Post请求体中的json以及将我们的对象序列化为返回响应的json
     * 使用@RequestBody注解的对象中的Date类型将从这里被转换
     */
    @Bean
    public ObjectMapper objectMapper(){
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        objectMapper.disable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE);

        JavaTimeModule javaTimeModule = new JavaTimeModule();

        //Date序列化和反序列化
        javaTimeModule.addSerializer(Date.class, new JsonSerializer<Date>() {
            @Override
            public void serialize(Date date, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
                SimpleDateFormat formatter = new SimpleDateFormat(DEFAULT_DATE_TIME_FORMAT);
                String formattedDate = formatter.format(date);
                jsonGenerator.writeString(formattedDate);
            }
        });
        javaTimeModule.addDeserializer(Date.class, new JsonDeserializer<Date>() {
            @Override
            public Date deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
                return new DateConverter().convert(jsonParser.getText());
            }
        });

        objectMapper.registerModule(javaTimeModule);
        return objectMapper;
    }

}
```

## 前端传参里包含实体类集合以及其他参数
- 解决方法：
	- 将实体类集合和其他参数重新作为一个实体类去接收前端传递的参数
**实体类1**
```java
@Data  
public class CouponInfo {  
    private String commodityId;  
    private String couponId;  
    private String diductAmount;  
}
```
**实体类2**
```java
@Data  
public class DeliveryInfo {  
    private String merchantId;  
    private String deliveryFee;  
}
```
**实体类3  用于整合实体类集合以及其他参数**
```java
@Data  
public class BuyShoppingCartParam {  
    private String customerId;  
    private List<DeliveryInfo> deliveryList;  
    private List<CouponInfo> coupons;  
}
```
**controller**
```java
@PostMapping(value = "/buy")  
public String buyShoppingCart(@RequestBody BuyShoppingCartParam duyShoppingCartParam) {  
    log.info("接口请求参数如下:\n duyShoppingCartParam:{}",duyShoppingCartParam);  
    String msg = "成功!";// 提示信息  
    String data = JSON.toJSONString(duyShoppingCartParam.getDeliveryList());// 返回内容  
    System.out.println("打印测试customerId:"+duyShoppingCartParam.getCustomerId());  
  
    System.out.println("打印测试deliveryInfo："+JSON.toJSONString(duyShoppingCartParam.getDeliveryList()));  
  
    System.out.println("打印测试couponInfo："+JSON.toJSONString(duyShoppingCartParam.getCoupons()));  
  
    return "xixi";  
}
```
**post测试**
![[Pasted image 20221104190202.png]]
```json
{

    "deliveryList": [

        {

            "merchantId": "10086",

            "deliveryFee": "200"

        },

        {

            "merchantId": "10010",

            "deliveryFee": "100"

        }

    ],

    "coupons": [

        {

            "commodityId": "2311111",

            "couponId": "45544333",

            "diductAmount": "13"

        },

        {

            "commodityId": "2323133133",

            "couponId": "1234456444",

            "diductAmount": "233"

        }

    ],

    "customerId": "6666666"

}
```
# 异常
## 传参问题
- No primary or single public constructor found for interface java.util.List
	- 后端接口参数没有加@RequestBody参数

