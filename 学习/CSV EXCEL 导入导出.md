## 1. CSV
### 1.1 CSV 导出
#### 导入依赖
```xml
<dependency>  
    <groupId>com.univocity</groupId>  
    <artifactId>univocity-parsers</artifactId>  
    <version>2.9.1</version>  
</dependency>
```
#### 查询数据库返回数据
```java
public void exportUserListWithCsv(String startTime, String endTime, String fileName , Boolean title, Double minlongitude,Double maxlongitude, Double minlatitude, Double maxlatitude) {  
  
        List<AISMessageExportDto> exportItem = new ArrayList<>();  
        // 查询数据  
        List<AISMessage> dbData = aisMessageExportService.selectMessage(startTime, endTime, minlongitude, maxlongitude,  minlatitude,  maxlatitude);  
        // 将数据转换成导出需要的实际数据格式,此处只是演示  
        for (AISMessage aisMessage : dbData) {  
            AISMessageExportDto vo = new AISMessageExportDto();  
            BeanUtil.copyProperties(aisMessage, vo);  
            exportItem.add(vo);  
        }  
//         使用bean字段的方式作为表头，导出csv文件  
        CsvExportUtil.exportCsvWithBean(fileName, new AISMessageExportDto(), exportItem , title);  
        log.info("{} 成功导出数据至-{}",startTime, fileName);    
    }
```

#### 将返回数据导出为csv文件
```java
package com.example.csvtoclickhouse.utils;  
  
  
import com.univocity.parsers.common.processor.BeanWriterProcessor;  
import com.univocity.parsers.csv.CsvWriter;  
import com.univocity.parsers.csv.CsvWriterSettings;  
import lombok.extern.slf4j.Slf4j;  
  
import javax.servlet.http.HttpServletResponse;  
import java.io.FileOutputStream;  
import java.io.IOException;  
import java.util.List;  
import java.util.Objects;  

/**  
 * @Author it-learning  
 * @Description csv文件导出包  
 * @Date 2022/5/16 15:55  
 * @Version 1.0  
 */@Slf4j  
public class CsvExportUtil<T> {  
  
    /**  
     * 导出csv文件(表头和行都以实体的方式)  
     *     * @param head  
     * @param rowDataList  
     */  
    public static <T> void exportCsvWithBean( String fileName, T head, List<T> rowDataList, Boolean title) {  
        CsvWriter writer = null;  
        try {  
            // 设置响应头格式  
           // buildingCsvExportResponse(response,fileName);  
  
            // 设置导出格式  
            CsvWriterSettings setting = getDefaultWriteSetting();  
  
            // 设置是否自动写入标题  
            setting.setHeaderWritingEnabled(title);  
  
            // 创见bean处理器，用于处理写入数据  
            BeanWriterProcessor<?> beanWriter = new BeanWriterProcessor<>(head.getClass());  
            setting.setRowWriterProcessor(beanWriter);  
  
            // 导出数据  
            FileOutputStream outputStream = new FileOutputStream(fileName,true); 
            /***  
 * 设置文件的BOM(byte-order-mark，字节顺序标记，是位于码点U+FEFF的Unicode字符的名称)头，，防止csv文件使用office打开时中文出现乱码  
 * 原因：【Windows就是使用BOM来标记文本文件的编码方式的。UTF-8不需要BOM来表明字节顺序，但可以用BOM来表明编码方式。当文本程序读取到以EF BB BF开头的字节流时，就知道这是UTF-8编码了】  
 * 因为CSV文件是纯文本文件，office打开这种文件默认是以excel的方式，这个时候需要通过元信息来识别打开它的编码，  
 * 微软就定义了一个自己的格式叫 BOM 头，上面的代码就是给csv文件添加这个BOM头，没有指定时，工具会使用默认的编码，编码不匹配则会导致乱码。  
 *  
 */ 
            outputStream.write(0xef);  
            outputStream.write(0xbb);  
            outputStream.write(0xbf);  
            writer = new CsvWriter(outputStream, setting);  
            writer.processRecords(rowDataList);  
            writer.flush();  
        } catch (Exception e) {  
            log.error("CsvExportUtil exportCsvWithBean in error:{}", e);  
        } finally {  
            if (Objects.nonNull(writer)) {  
                writer.close();  
            }  
        }  
    }  
  
    
  
    /**  
     * 获取默认的CSV写入配置对象  
     *  
     * @return  
     */  
    private static CsvWriterSettings getDefaultWriteSetting() {  
        CsvWriterSettings settings = new CsvWriterSettings();  
        /**  
         * 如果要设置值之间的间隔可以使用下面示例代码  
         * FixedWidthFields lengths = new FixedWidthFields(10, 10, 35, 10, 40);  
         * FixedWidthWriterSettings settings = new FixedWidthWriterSettings(lengths);         **/        // 设置值为null时替换的值  
        settings.setNullValue("");  
        // 修改分隔符,默认是逗号  
        //settings.getFormat().setDelimiter("");  
        // 设置是否自动写入标题  
      //  settings.setHeaderWritingEnabled(Boolean.TRUE);  
        return settings;  
    }  
  
}

```

#### 相关实体类-AISMessage
```java
package com.example.csvtoclickhouse.model;  
  
  
import com.baomidou.mybatisplus.annotation.TableName;  
import com.fasterxml.jackson.annotation.JsonFormat;  
import com.univocity.parsers.annotations.Parsed;  
import lombok.Data;  
import org.springframework.format.annotation.DateTimeFormat;  
  
import java.io.Serializable;  
import java.util.Date;  
  
/**  
 * @author WangDongping  
 * @create 2022-11-07 14:55  
 */@Data  
@TableName(value = "ais")  
public class AISMessage implements Serializable {  
    private static final long serialVersionUID = 1L;  
  
  
    private String mmsi;  
  
    private String imo;  
  
    private String vesselName;  
  
    private String callsign;  
  
    private String vesselType;  
  
    private Integer vesselTypeCode;  
  
    private String vesselTypeCargo;  
  
    private String vesselClass;  
  
    private Integer length;  
  
    private Integer width;  
  
    private String flagCountry;  
  
    private Integer flagCode;  
  
    private String destination;  
  
    private String eta;  
  
    private Float draught;  
  
    private String longitude;  
  
    private String latitude;  
  
    private Float sog;  
  
    private Float cog;  
  
    private Float rot;  
  
    private Float heading;  
  
    private String navStatus;  
  
    private Integer navStatusCode;  
  
    private String source;  
  
  
    private String tsPosUtc;  
  
    private String tsStaticUtc;  
  
    @JsonFormat(locale="zh", timezone="GMT+8", pattern="yyyy-MM-dd HH:mm:ss")  
    private String dtPosUtc;  
  
    @JsonFormat(locale="zh", timezone="GMT+8", pattern="yyyy-MM-dd HH:mm:ss")  
    private String dtStaticUtc;  
  
    private String vesselTypeMain;  
  
    private String vesselTypeSub;  
  
  
  
}
```
#### 相关实体类-AISMessageExportDto
```java 
package com.example.csvtoclickhouse.model;  
  
import com.fasterxml.jackson.annotation.JsonFormat;  
import com.univocity.parsers.annotations.Parsed;  
import lombok.Data;  
import org.springframework.format.annotation.DateTimeFormat;  
  
import java.util.Date;  
  
/**  
 * @author WangDongping  
 * @create 2023-01-07 10:04  
 */@Data  
public class AISMessageExportDto {  
    /**  
     * Parsed注解将属性名称映射到CSV文件中  
     */  
    @Parsed(field="mmsi")  
    private String mmsi;  
  
    @Parsed(field="imo")  
    private String imo;  
  
    @Parsed(field="vessel_name")  
    private String vesselName;  
  
    @Parsed(field="callsign")  
    private String callsign;  
  
    @Parsed(field="vessel_type")  
    private String vesselType;  
  
    @Parsed(field="vessel_type_code")  
    private Integer vesselTypeCode;  
  
    @Parsed(field="vessel_type_cargo")  
    private String vesselTypeCargo;  
  
    @Parsed(field="vessel_class")  
    private String vesselClass;  
  
    @Parsed(field="length")  
    private Integer length;  
  
    @Parsed(field="width")  
    private Integer width;  
  
    @Parsed(field="flag_country")  
    private String flagCountry;  
  
    @Parsed(field="flag_code")  
    private Integer flagCode;  
  
    @Parsed(field="destination")  
    private String destination;  
  
    @Parsed(field="eta")  
    private String eta;  
  
    @Parsed(field="draught")  
    private Float draught;  
  
    @Parsed(field="longitude")  
    private String longitude;  
  
    @Parsed(field="latitude")  
    private String latitude;  
  
    @Parsed(field="sog")  
    private Float sog;  
  
    @Parsed(field="cog")  
    private Float cog;  
  
    @Parsed(field="rot")  
    private Float rot;  
  
    @Parsed(field="heading")  
    private Float heading;  
  
    @Parsed(field="nav_status")  
    private String navStatus;  
  
    @Parsed(field="nav_status_code")  
    private Integer navStatusCode;  
  
    @Parsed(field="source")  
    private String source;  
  
    @Parsed(field="ts_pos_utc")  
    private String tsPosUtc;  
  
    @Parsed(field="ts_static_utc")  
    private String tsStaticUtc;  
  
    @JsonFormat(locale="zh", timezone="GMT+8", pattern="yyyy-MM-dd HH:mm:ss")  
    @Parsed(field="dt_pos_utc")  
    private String dtPosUtc;  
  
    @JsonFormat(locale="zh", timezone="GMT+8", pattern="yyyy-MM-dd HH:mm:ss")  
    @Parsed(field="dt_static_utc")  
    private String dtStaticUtc;  
  
    @Parsed(field="vessel_type_main")  
    private String vesselTypeMain;  
  
    @Parsed(field="vessel_type_sub")  
    private String vesselTypeSub;  
  
  
}
```

### 1.2 CSV 导入
#### 导入依赖
```xml
<dependency>  
    <groupId>com.opencsv</groupId>  
    <artifactId>opencsv</artifactId>  
    <version>5.6</version>  
</dependency>
```

#### 使用缓冲流读取CSV文件
```java
package com.example.csvtoclickhouse.utils;  
  
import cn.hutool.core.collection.ListUtil;  
import cn.hutool.core.util.StrUtil;  
import com.example.csvtoclickhouse.model.AISMessageCsvDto;  
import com.example.csvtoclickhouse.service.AISMessageExportService;  
import com.opencsv.CSVReader;  
import com.opencsv.CSVReaderBuilder;  
import com.opencsv.exceptions.CsvValidationException;  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Component;  
  
import java.io.*;  
import java.util.ArrayList;  
import java.util.Iterator;  
import java.util.List;  
  
/**  
 * @author WangDongping  
 * @create 2023-01-11 14:26  
 */@Component  
@Slf4j  
public class CSVToClik {  
  
    private static List<AISMessageCsvDto> aisMessageList = new ArrayList<>();  
    private static AISMessageExportService aisMessageExportService;  
  
    @Autowired  
    public CSVToClik(AISMessageExportService aisMessageExportService) {  
        this.aisMessageExportService = aisMessageExportService;  
    }  
  
    public void readCsvByBufferedReader(String filePath) {  
  
        log.info("{} 开始入库", filePath);  
  
        Long i = 1L;  
        try {  
            CSVReader csvReader = new CSVReaderBuilder(  
                    new BufferedReader(new FileReader(filePath))).build();  
            Iterator<String[]> iterator = csvReader.iterator();  
            while (iterator.hasNext()) {  
                String[] next = iterator.next();  
                //去除第一行的表头，从第二行开始  
                if (i == 1L) {  
                    i++;  
                    continue;                }  
                AISMessageCsvDto aisMessageCsvDto = StringToAISMessage(next);  
                if (StrUtil.isNotBlank(aisMessageCsvDto.getDtPosUtc())) {  
                    aisMessageList.add(aisMessageCsvDto);  
                    if (aisMessageList.size() > 499) {  
                        // 执行数据持久化  
                        persistentBeanDataToDb(aisMessageList);  
                    }  
                }  
            }  
  
            if (aisMessageList.size() > 0) {  
                // 执行数据持久化  
                this.persistentBeanDataToDb(aisMessageList);  
            }  
  
        } catch (Exception e) {  
            System.out.println("CSV文件读取异常");  
  
        }  
  
        log.info("{} 结束入库", filePath);  
        log.info("================================================");  
    }  
  
  
    /**  
     * 将数据持久化到数据库中  
     * 具体数据落库的业务逻辑方法：此处的逻辑是将数据从csv中读取出来后，然后进行自己的业务处理，最后进行落库操作  
     */  
    private <T> void persistentBeanDataToDb(List<AISMessageCsvDto> data) {  
        // 对数据分组，批量插入  
        //      List<List<T>> dataList = ListUtil.split(data, ImportConstant.MAX_INSERT_COUNT);  
  
//        for (int i = 0; i < dataList.size(); i++) {  
//            List<AISMessageCsvDto> list = (List<AISMessageCsvDto>) dataList.get(i);  
//            aisMessageExportService.insertBatchSomeColumn(list);  
//        }  
        aisMessageExportService.insertBatchSomeColumn(data);  
        data.clear();  
  
  
    }  
  
  
    private static AISMessageCsvDto StringToAISMessage(String[] row) {  
  
        AISMessageCsvDto ais = new AISMessageCsvDto();  
  
        try {  
            String mmsi = row[0];  
            if (StrUtil.isNotBlank(mmsi)) {  
                ais.setMmsi(mmsi);  
            }  
  
            String imo = row[1];  
            if (StrUtil.isNotBlank(imo)) {  
                ais.setImo(imo);  
            }  
  
            String vesselName = row[2];  
            if (StrUtil.isNotBlank(vesselName)) {  
                ais.setVesselName(vesselName);  
            }  
            String callsign = row[3];  
            if (StrUtil.isNotBlank(callsign)) {  
                ais.setCallsign(callsign);  
            }  
  
            String vesselType = row[4];  
            if (StrUtil.isNotBlank(vesselType)) {  
                ais.setVesselType(vesselType);  
            }  
  
            if (StrUtil.isNotBlank(row[5])) {  
                Integer vesselTypeCode = Integer.valueOf(row[5]);  
  
                ais.setVesselTypeCode(vesselTypeCode);  
            }  
  
  
            String vesselTypeCargo = row[6];  
            if (StrUtil.isNotBlank(vesselTypeCargo)) {  
                ais.setVesselTypeCargo(vesselTypeCargo);  
            }  
            String vesselClass = row[7];  
            if (StrUtil.isNotBlank(vesselClass)) {  
                ais.setVesselClass(vesselClass);  
            }  
  
  
            if (StrUtil.isNotBlank(row[8])) {  
                Integer length = Integer.valueOf(row[8]);  
                ais.setLength(length);  
            }  
  
            if (StrUtil.isNotBlank(row[9])) {  
                Integer width = Integer.valueOf(row[9]);  
                ais.setWidth(width);  
            }  
  
            String flagCountry = row[10];  
            if (StrUtil.isNotBlank(flagCountry)) {  
                ais.setFlagCountry(flagCountry);  
            }  
  
            if (StrUtil.isNotBlank(row[11])) {  
                Integer flagCode = Integer.valueOf(row[11]);  
                ais.setFlagCode(flagCode);  
            }  
  
  
            String destination = row[12];  
            if (StrUtil.isNotBlank(destination)) {  
                ais.setDestination(destination);  
            }  
            String eta = row[13];  
            if (StrUtil.isNotBlank(eta)) {  
                ais.setEta(eta);  
            }  
  
            if (StrUtil.isNotBlank(row[14])) {  
                Float draught = Float.valueOf(row[14]);  
                ais.setDraught(draught);  
            }  
  
  
            String longitude = row[15];  
            if (StrUtil.isNotBlank(longitude)) {  
                ais.setLongitude(longitude);  
            }  
            String latitude = row[16];  
            if (StrUtil.isNotBlank(latitude)) {  
                ais.setLatitude(latitude);  
            }  
  
            if (StrUtil.isNotBlank(row[17])) {  
                Float sog = Float.valueOf(row[17]);  
                ais.setSog(sog);  
            }  
  
            if (StrUtil.isNotBlank(row[18])) {  
                Float cog = Float.valueOf(row[18]);  
                ais.setCog(cog);  
            }  
  
            if (StrUtil.isNotBlank(row[19])) {  
                Float rot = Float.valueOf(row[19]);  
                ais.setRot(rot);  
            }  
  
            if (StrUtil.isNotBlank(row[20])) {  
                Float heading = Float.valueOf(row[20]);  
                ais.setHeading(heading);  
            }  
  
            String navStatus = row[21];  
            if (StrUtil.isNotBlank(navStatus)) {  
                ais.setNavStatus(navStatus);  
            }  
  
            if (StrUtil.isNotBlank(row[22])) {  
                Integer navStatusCode = Integer.valueOf(row[22]);  
                ais.setNavStatusCode(navStatusCode);  
            }  
  
            String source = row[23];  
            if (StrUtil.isNotBlank(source)) {  
                ais.setSource(source);  
            }  
            String tsPosUtc = row[24];  
            if (StrUtil.isNotBlank(tsPosUtc)) {  
                ais.setTsPosUtc(tsPosUtc);  
            }  
            String tsStaticUtc = row[25];  
            if (StrUtil.isNotBlank(tsStaticUtc)) {  
                ais.setTsStaticUtc(tsStaticUtc);  
            }  
            String dtPosUtc = row[26];  
            if (StrUtil.isNotBlank(dtPosUtc)) {  
                ais.setDtPosUtc(dtPosUtc);  
            }  
            String dtStaticUtc = row[27];  
            if (StrUtil.isNotBlank(dtStaticUtc)) {  
                ais.setDtStaticUtc(dtStaticUtc);  
            }  
  
            if (row.length == 29) {  
                String vesselTypeMain = row[28];  
                if (StrUtil.isNotBlank(vesselTypeMain)) {  
                    ais.setVesselTypeMain(vesselTypeMain);  
                }  
  
            }  
  
            if (row.length == 30) {  
                String vesselTypeMain = row[28];  
                if (StrUtil.isNotBlank(vesselTypeMain)) {  
                    ais.setVesselTypeMain(vesselTypeMain);  
                }  
  
                String vesselTypeSub = row[29];  
                if (StrUtil.isNotBlank(vesselTypeSub)) {  
                    ais.setVesselTypeSub(vesselTypeSub);  
                }  
            }  
  
        } catch (Exception e) {  
        }  
  
        return ais;  
    }  
  
}
```

#### 配置mybatis-plus批量插入
1. 建mapper
```java
package com.example.csvtoclickhouse.mapper;  
  
  
import com.baomidou.mybatisplus.core.mapper.BaseMapper;  
import com.example.csvtoclickhouse.model.AISMessage;  
import com.example.csvtoclickhouse.model.AISMessageCsvDto;  
import org.apache.ibatis.annotations.Mapper;  
  
import java.util.Collection;  
import java.util.List;  
  
/**  
 * @author WangDongping  
 * @create 2023-01-07 10:28  
 */@Mapper  
public interface AISMessageExportMapper extends BaseMapper<AISMessageCsvDto> {    
    Integer insertBatchSomeColumn(Collection<AISMessageCsvDto> entityList);  

}
```
2. 配置类
```java
package com.example.csvtoclickhouse.config;  
  
import com.example.csvtoclickhouse.service.EasySqlInjector;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.context.annotation.Primary;  
import org.springframework.transaction.annotation.EnableTransactionManagement;  
  
/**  
 * @author WangDongping  
 * @create 2023-01-10 17:01  
 */@Configuration  
@EnableTransactionManagement //开启mybatis事务管理  
public class MPConfiguration {  
  
    @Bean  
    @Primary//批量插入配置  
    public EasySqlInjector easySqlInjector() {  
        return new EasySqlInjector();  
    }  
  
}
```
3. 创建EasySqlInjector
```java
package com.example.csvtoclickhouse.service;  
  
import com.baomidou.mybatisplus.core.injector.AbstractMethod;  
import com.baomidou.mybatisplus.core.injector.DefaultSqlInjector;  
import com.baomidou.mybatisplus.core.metadata.TableInfo;  
import com.baomidou.mybatisplus.extension.injector.methods.additional.InsertBatchSomeColumn;  
  
import java.util.List;  
  
/**  
 * @author WangDongping  
 * @create 2023-01-10 17:03  
 */public class EasySqlInjector extends DefaultSqlInjector {  
    @Override  
    public List<AbstractMethod> getMethodList(Class<?> mapperClass) {  
        List<AbstractMethod> methodList = super.getMethodList(mapperClass);  
        methodList.add(new InsertBatchSomeColumn());//添加批量插入方法  
        return methodList;  
    }  
}
```

### 1.3 参考
- https://juejin.cn/post/7159126025002024974
- https://blog.csdn.net/qq_40891009/article/details/124874311
- https://blog.csdn.net/weixin_40109343/article/details/125449008


## 2. EXCEL
**依赖**
```xml

```
### 2.1 EXCEL 导出
### 2.2 EXCEL 导入
### 2.3 参考
- https://mp.weixin.qq.com/s/MM1uDx2zQOgJA9qE2NJhDQ
- https://www.cnblogs.com/wxw7blog/p/8706797.html
- https://www.jb51.net/article/33365.htm

```java
// EasyExcel的读取Excel数据的API
@Test
public void import2DBFromExcel10wTest() {
    String fileName = "D:\\StudyWorkspace\\JavaWorkspace\\java_project_workspace\\idea_projects\\SpringBootProjects\\easyexcel\\exportFile\\excel300w.xlsx";
    //记录开始读取Excel时间,也是导入程序开始时间
    long startReadTime = System.currentTimeMillis();
    System.out.println("------开始读取Excel的Sheet时间(包括导入数据过程):" + startReadTime + "ms------");
    //读取所有Sheet的数据.每次读完一个Sheet就会调用这个方法
    EasyExcel.read(fileName, new EasyExceGeneralDatalListener(actResultLogService2)).doReadAll();
    long endReadTime = System.currentTimeMillis();
    System.out.println("------结束读取Excel的Sheet时间(包括导入数据过程):" + endReadTime + "ms------");
}
// 事件监听
public class EasyExceGeneralDatalListener extends AnalysisEventListener<Map<Integer, String>> {
    /**
     * 处理业务逻辑的Service,也可以是Mapper
     */
    private ActResultLogService2 actResultLogService2;

    /**
     * 用于存储读取的数据
     */
    private List<Map<Integer, String>> dataList = new ArrayList<Map<Integer, String>>();

    public EasyExceGeneralDatalListener() {
    }

    public EasyExceGeneralDatalListener(ActResultLogService2 actResultLogService2) {
        this.actResultLogService2 = actResultLogService2;
    }

    @Override
    public void invoke(Map<Integer, String> data, AnalysisContext context) {
        //数据add进入集合
        dataList.add(data);
        //size是否为100000条:这里其实就是分批.当数据等于10w的时候执行一次插入
        if (dataList.size() >= ExcelConstants.GENERAL_ONCE_SAVE_TO_DB_ROWS) {
            //存入数据库:数据小于1w条使用Mybatis的批量插入即可;
            saveData();
            //清理集合便于GC回收
            dataList.clear();
        }
    }

    /**
     * 保存数据到DB
     *
     * @param
     * @MethodName: saveData
     * @return: void
     */
    private void saveData() {
        actResultLogService2.import2DBFromExcel10w(dataList);
        dataList.clear();
    }

    /**
     * Excel中所有数据解析完毕会调用此方法
     *
     * @param: context
     * @MethodName: doAfterAllAnalysed
     * @return: void
     */
    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        saveData();
        dataList.clear();
    }
}
//JDBC工具类
public class JDBCDruidUtils {
    private static DataSource dataSource;

    /*
   创建数据Properties集合对象加载加载配置文件
    */
    static {
        Properties pro = new Properties();
        //加载数据库连接池对象
        try {
            //获取数据库连接池对象
            pro.load(JDBCDruidUtils.class.getClassLoader().getResourceAsStream("druid.properties"));
            dataSource = DruidDataSourceFactory.createDataSource(pro);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /*
    获取连接
     */
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }


    /**
     * 关闭conn,和 statement独对象资源
     *
     * @param connection
     * @param statement
     * @MethodName: close
     * @return: void
     */
    public static void close(Connection connection, Statement statement) {
        if (connection != null) {
            try {
                connection.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        if (statement != null) {
            try {
                statement.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 关闭 conn , statement 和resultset三个对象资源
     *
     * @param connection
     * @param statement
     * @param resultSet
     * @MethodName: close
     * @return: void
     */
    public static void close(Connection connection, Statement statement, ResultSet resultSet) {
        close(connection, statement);
        if (resultSet != null) {
            try {
                resultSet.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    /*
    获取连接池对象
     */
    public static DataSource getDataSource() {
        return dataSource;
    }

}
# druid.properties配置
    driverClassName=oracle.jdbc.driver.OracleDriver
    url=jdbc:oracle:thin:@localhost:1521:ORCL
        username=mrkay
        password=******
        initialSize=10
        maxActive=50
        maxWait=60000
        // Service中具体业务逻辑

        /**
 * 测试用Excel导入超过10w条数据,经过测试发现,使用Mybatis的批量插入速度非常慢,所以这里可以使用 数据分批+JDBC分批插入+事务来继续插入速度会非常快
 *
 * @param
 * @MethodName: import2DBFromExcel10w
 * @return: java.util.Map<java.lang.String, java.lang.Object>
 */
        @Override
        public Map<String, Object> import2DBFromExcel10w(List<Map<Integer, String>> dataList) {
        HashMap<String, Object> result = new HashMap<>();
        //结果集中数据为0时,结束方法.进行下一次调用
        if (dataList.size() == 0) {
            result.put("empty", "0000");
            return result;
        }
        //JDBC分批插入+事务操作完成对10w数据的插入
        Connection conn = null;
        PreparedStatement ps = null;
        try {
            long startTime = System.currentTimeMillis();
            System.out.println(dataList.size() + "条,开始导入到数据库时间:" + startTime + "ms");
            conn = JDBCDruidUtils.getConnection();
            //控制事务:默认不提交
            conn.setAutoCommit(false);
            String sql = "insert into ACT_RESULT_LOG (onlineseqid,businessid,becifno,ivisresult,createdby,createddate,updateby,updateddate,risklevel) values";
            sql += "(?,?,?,?,?,?,?,?,?)";
            ps = conn.prepareStatement(sql);
            //循环结果集:这里循环不支持"烂布袋"表达式
            for (int i = 0; i < dataList.size(); i++) {
                Map<Integer, String> item = dataList.get(i);
                ps.setString(1, item.get(0));
                ps.setString(2, item.get(1));
                ps.setString(3, item.get(2));
                ps.setString(4, item.get(3));
                ps.setString(5, item.get(4));
                ps.setTimestamp(6, new Timestamp(System.currentTimeMillis()));
                ps.setString(7, item.get(6));
                ps.setTimestamp(8, new Timestamp(System.currentTimeMillis()));
                ps.setString(9, item.get(8));
                //将一组参数添加到此 PreparedStatement 对象的批处理命令中。
                ps.addBatch();
            }
            //执行批处理
            ps.executeBatch();
            //手动提交事务
            conn.commit();
            long endTime = System.currentTimeMillis();
            System.out.println(dataList.size() + "条,结束导入到数据库时间:" + endTime + "ms");
            System.out.println(dataList.size() + "条,导入用时:" + (endTime - startTime) + "ms");
            result.put("success", "1111");
        } catch (Exception e) {
            result.put("exception", "0000");
            e.printStackTrace();
        } finally {
            //关连接
            JDBCDruidUtils.close(conn, ps);
        }
        return result;
    }
```
