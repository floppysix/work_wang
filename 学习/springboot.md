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

