
![[Pasted image 20230113142127.png]]

日志文件：/var/log/clickhouse-server/clickhouse-server.log

数据位置：/mnt/pest/software/clickhouse/AIS_data

查看进程：ps -ef | grep clickhouse-server

修改/data所属用户组即可chown -R  clickhouse:clickhouse /data



### clickhouse基础操作
#### 设置密码
```
1. 生成密码（这样可以得到两行数据，第一行是密码明文，第二行是密码密文）
	PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
	
2. 设置密码
	1. vim  /etc/clickhouse-server/users.xml，找到 users --> default --> 标签下的password修改成password_sha256_hex，并把密文填进去
	2. <password_sha256_hex>ba4d1ca428b4b7b6fb7f3054a1f0a4501c61e9ccc9cfa31b41c4fdc27c215240</password_sha256_hex>

明文密码：DjijmDL/

```

#### 命令行启动
```
clickhouse-client -h 211.157.180.220 -d default -m -u default --password s8SKh8fe

clickhouse-client -h 124.70.14.81 -d default -m -u default --password 0uKJM5Bj

clickhouse-client -h 192.168.10.102 -d default -m -u default --password DjijmDL/
```

#### 卸载clickhouse
```bash
1、查看已安装包

rpm -qa | grep clickhouse

2、卸载clickhouse相关软件

rpm -e clickhouse-client-21.7.3.14-2.noarch --nodeps rpm -e clickhouse-server-21.7.3.14-2.noarch --nodeps rpm -e clickhouse-common-static-21.7.3.14-2.x86_64 --nodeps rpm -e clickhouse-common-static-dbg-21.7.3.14-2.x86_64 --nodeps

3、删除相关的目录和数据

#数据目录
rm -rf /var/lib/clickhouse

#删除集群配置文件
rm -rf /etc/metrika.xml

#删除配置文件
rm -rf /etc/clickhouse-*

#删除日志文件
rm -rf /var/log/clickhouse-server


4、全局查找clickhouse文件和目录，如果存在，则全部删除

find / -name clickhouse
```
#### 备份
```bash
sudo mkdir -p /var/lib/clickhouse/shadow/
# 将新建的文件夹用户改成clickhouse
chown clickhouse:clickhouse shadow
# 执行备份语句
echo -n 'alter table t_order_mt freeze' | clickhouse-client
#创建备份存储路径
sudo mkdir -p /var/lib/clickhouse/backup/
#拷贝数据到备份路径
sudo cp -r /var/lib/clickhouse/shadow/ /var/lib/clickhouse/backup/my-backup-name
#为下次备份准备，删除 shadow 下的数据
sudo rm -rf /var/lib/clickhouse/shadow/*
# 模拟删除备份过的表
echo ' drop table t_order_mt ' | clickhouse-client
# 重新创建表
cat events.sql | clickhouse-client
将备份复制到 detached 目录
sudo cp -rl backup/my-backup-name/1/store/37e/37e67f35-2477-42be-b7e6-7f35247782be/* data/AIS/ais/detached/
# 将复制的备份所在文件夹改成clickhouse
chown -R clickhouse:clickhouse detached/
执行 attach
echo 'alter table t_order_mt attach partition 20200601' | clickhouse-client
```

### 日志

#### 修改日志文件存储时间
```sql
ALTER TABLE system.query_log MODIFY TTL event_date + INTERVAL 1 day;

ALTER TABLE system.query_thread_log MODIFY TTL event_date + INTERVAL 1 day;
```
- 官方推荐，但是经操作后不适用
```sql

vim /etc/clickhouse-server/config.xml

<query_log>
    <database>system</database>
    <table>query_log</table>
    <engine>Engine = MergeTree PARTITION BY event_date ORDER BY event_time TTL event_date + INTERVAL 30 day</engine>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
</query_log>


    <query_thread_log>
        <database>system</database>
        <table>query_thread_log</table>
<!--    
    <partition_by>toYYYYMM(event_date)</partition_by>
-->
<engine>Engine = MergeTree PARTITION BY event_date ORDER BY event_time TTL event_date + INTERVAL 3 day</engine>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_thread_log>
```


#### 查看日志文件大小
```sql
SELECT 
    sum(rows) AS `总行数`,
    formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,
    formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,
    round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`,
    `table` AS `表名`
FROM system.parts where database = 'system' group by `table`
```


#### 删除日志文件
```sql
alter table system.query_log drop partition '202201';
```

### 报错code

#### clickhouse连接报错1002
```
vim /etc/clickhouse-server/config.xml

<!-- 600 可以设置成自己所需要的> -->
 <keep_alive_timeout>600</keep_alive_timeout>
```

#### 断电数据丢失，clickhouse无法启动
```
问题描述：
	断电重启后，元数据与数据不一致导致重启失败，报错： Application: DB::Exception: Suspiciously **many (244)** broken parts to remove.
解决方法：
	修改config.xml
	<merge_tree>
        <max_suspicious_broken_parts>500</max_suspicious_broken_parts>
	</merge_tree>
参考链接：
	https://blog.csdn.net/Vector97/article/details/125222928
```

#### 159
```
出现报错：ClickHouse exception, code: 159, host: 10.100.xx.xxx, port: 8123; Read timed out

解决方法：
在连接的路径后面加上?socket_timeout=300000
spring:  
  datasource:  
    type: com.alibaba.druid.pool.DruidDataSource  
    click:  
      driverClassName: ru.yandex.clickhouse.ClickHouseDriver  
      url: jdbc:clickhouse://124.70.14.81:8123/AIS_global?socket_timeout=300000  
      username: default  
      password: 0uKJM5Bj  
      initialSize: 10  
      maxActive: 100  
      minIdle: 10  
      maxWait: 60000  
      timeOut: 300000
```


### 建表sql

#### 创建数据库
```sql
CREATE DATABASE IF NOT EXISTS AIS;   --使用默认库引擎创建库
```

### 全球数据入库建表语句
```sql
CREATE TABLE AIS.ais
(

    `mmsi` String,

    `imo` Nullable(String),

    `vessel_name` Nullable(String),

    `callsign` Nullable(String),

    `vessel_type` Nullable(String),

    `vessel_type_code` Nullable(Int32),

    `vessel_type_cargo` Nullable(String),

    `vessel_class` Nullable(String),

    `length` Nullable(Int32),

    `width` Nullable(Int32),

    `flag_country` Nullable(String),

    `flag_code` Nullable(Int32),

    `destination` Nullable(String),

    `eta` Nullable(String),

    `draught` Nullable(Float32),

    `position` Nullable(String),

    `longitude` Nullable(Decimal64(6)),

    `latitude` Nullable(Decimal64(6)),

    `sog` Nullable(Float32),

    `cog` Nullable(Float32),

    `rot` Nullable(Float32),

    `heading` Nullable(Int32),

    `nav_status` Nullable(String),

    `nav_status_code` Nullable(Int32),

    `source` Nullable(String),

    `ts_pos_utc` Nullable(String),

    `ts_static_utc` Nullable(String),

    `dt_pos_utc` DateTime,

    `dt_static_utc` DateTime,

    `vessel_type_main` Nullable(String),

    `vessel_type_sub` Nullable(String),

    `message_type` Nullable(Int32),

    `dtg` Nullable(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(dt_pos_utc)
ORDER BY mmsi
SETTINGS index_granularity = 8192;
```

### 全球物化视图
```sql

```

#### 2016-2017历史数据
```sql
CREATE TABLE AIS_temp1.ais
(

    `mmsi` String,

    `imo` Nullable(String),

    `vessel_name` Nullable(String),

    `callsign` Nullable(String),

    `vessel_type` Nullable(String),

    `vessel_type_code` Nullable(Int32),

    `vessel_type_cargo` Nullable(String),

    `vessel_class` Nullable(String),

    `length` Nullable(Int32),

    `width` Nullable(Int32),

    `flag_country` Nullable(String),

    `flag_code` Nullable(Int32),

    `destination` Nullable(String),

    `eta` Nullable(String),

    `draught` Nullable(Float32),


    `longitude` Nullable(String),

    `latitude` Nullable(String),

    `sog` Nullable(Float32),

    `cog` Nullable(Float32),

    `rot` Nullable(Float32),

    `heading` Nullable(Float32),

    `nav_status` Nullable(String),

    `nav_status_code` Nullable(Int32),

    `source` Nullable(String),

    `ts_pos_utc` Nullable(String),

    `ts_static_utc` Nullable(String),

    `dt_pos_utc` DateTime,

    `dt_static_utc` DateTime,

    `vessel_type_main` Nullable(String),

    `vessel_type_sub` Nullable(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(dt_pos_utc)
ORDER BY mmsi
SETTINGS index_granularity = 8192;
```

#### 全球数据建表语句（长江）
```sql
CREATE TABLE AIS_global.ais
(

    `mmsi` String,

    `imo` Nullable(String),

    `vessel_name` Nullable(String),

    `callsign` Nullable(String),

    `vessel_type` Nullable(String),

    `vessel_type_code` Nullable(Int32),

    `vessel_type_cargo` Nullable(String),

    `vessel_class` Nullable(String),

    `length` Nullable(Int32),

    `width` Nullable(Int32),

    `flag_country` Nullable(String),

    `flag_code` Nullable(Int32),

    `destination` Nullable(String),

    `eta` Nullable(String),

    `draught` Nullable(Float32),

    `position` Nullable(String),

    `longitude` Nullable(String),

    `latitude` Nullable(String),

    `sog` Nullable(Float32),

    `cog` Nullable(Float32),

    `rot` Nullable(Float32),

    `heading` Nullable(Int32),

    `nav_status` Nullable(String),

    `nav_status_code` Nullable(Int32),

    `source` Nullable(String),

    `ts_pos_utc` Nullable(String),

    `ts_static_utc` Nullable(String),

    `dt_pos_utc` DateTime,

    `dt_static_utc` DateTime,

    `vessel_type_main` Nullable(String),

    `vessel_type_sub` Nullable(String),

    `message_type` Nullable(Int32),

    `dtg` Nullable(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(dt_pos_utc)
ORDER BY mmsi
SETTINGS index_granularity = 8192;

```

```sql
CREATE TABLE AIS_all_changjiang.ais
(

    `mmsi` String,

    `imo` Nullable(String),

    `vessel_name` Nullable(String),

    `callsign` Nullable(String),

    `vessel_type` Nullable(String),

    `vessel_type_code` Nullable(Int32),

    `vessel_type_cargo` Nullable(String),

    `vessel_class` Nullable(String),

    `length` Nullable(Int32),

    `width` Nullable(Int32),

    `flag_country` Nullable(String),

    `flag_code` Nullable(Int32),

    `destination` Nullable(String),

    `eta` Nullable(String),

    `draught` Nullable(Float32),

    `position` Nullable(String),

    `longitude` Nullable(String),

    `latitude` Nullable(String),

    `sog` Nullable(Float32),

    `cog` Nullable(Float32),

    `rot` Nullable(Float32),

    `heading` Nullable(Int32),

    `nav_status` Nullable(String),

    `nav_status_code` Nullable(Int32),

    `source` Nullable(String),

    `ts_pos_utc` Nullable(String),

    `ts_static_utc` Nullable(String),

    `dt_pos_utc` DateTime,

    `dt_static_utc` DateTime,

    `vessel_type_main` Nullable(String),

    `vessel_type_sub` Nullable(String),

    `message_type` Nullable(Int32),

    `dtg` Nullable(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(dt_pos_utc)
ORDER BY mmsi
SETTINGS index_granularity = 8192;
```

#### 物化视图
```sql
CREATE MATERIALIZED VIEW AIS.realtime
(

    `mmsi` String,

    `imo` Nullable(String),

    `vessel_name` Nullable(String),

    `callsign` Nullable(String),

    `vessel_type` Nullable(String),

    `vessel_type_code` Nullable(Int32),

    `vessel_type_cargo` Nullable(String),

    `vessel_class` Nullable(String),

    `length` Nullable(Int32),

    `width` Nullable(Int32),

    `flag_country` Nullable(String),

    `flag_code` Nullable(Int32),

    `destination` Nullable(String),

    `eta` Nullable(String),

    `draught` Nullable(Float32),

    `position` Nullable(String),

    `longitude` Nullable(Decimal64(6)),

    `latitude` Nullable(Decimal64(6)),

    `sog` Nullable(Float32),

    `cog` Nullable(Float32),

    `rot` Nullable(Float32),

    `heading` Nullable(Int32),

    `nav_status` Nullable(String),

    `nav_status_code` Nullable(Int32),

    `source` Nullable(String),

    `ts_pos_utc` Nullable(String),

    `ts_static_utc` Nullable(String),

    `dt_pos_utc` DateTime,

    `dt_static_utc` DateTime,

    `vessel_type_main` Nullable(String),

    `vessel_type_sub` Nullable(String),

    `message_type` Nullable(Int32),

    `dtg` Nullable(String)
)
ENGINE=ReplacingMergeTree
primary key (mmsi)
order by (mmsi)
populate AS 
SELECT mmsi, imo, vessel_name, callsign, vessel_type, vessel_type_code, vessel_type_cargo, vessel_class, `length`, width, flag_country, flag_code, destination, eta, draught, `position`, longitude, latitude, sog, cog, rot, heading, nav_status, nav_status_code, `source`, ts_pos_utc, ts_static_utc, dt_pos_utc, dt_static_utc, vessel_type_main, vessel_type_sub, message_type, dtg
FROM AIS.ais
WHERE dt_pos_utc > '2023-02-06 14:14:00';
```


### 时区问题

**在公司电脑中dbserver默认设置时区状态下，ais数据的时区是正常的，在此情况下，通过dbserver向其他服务器迁移数据时，时间类型的数据是正常的**

#### 使用DBServer连接clickhouse时，需提前设置**全局属性**

![[a7c1368fdea740308a5749d45c604a8c-1.png]]

![[Pasted image 20230112163603.png]]

#### 通过指令查看服务器以及clickhouse数据库的时区
```sql
Clickhouse> select now();
 
SELECT now()
 
┌───────────────now()─┐
│ 2020-07-11 23:47:56 │
└─────────────────────┘
 
1 rows in set. Elapsed: 0.003 sec. 
 
Clickhouse> exit;
Bye.
[root@hadoop ~]# date
Sat Jul 11 23:48:01 CST 2020
 
 
 
此时操作系统的时区和时间是：
# timedatectl
      Local time: Sat 2020-07-11 23:49:06 CST
  Universal time: Sat 2020-07-11 15:49:06 UTC
        RTC time: Sat 2020-07-11 15:49:05
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: n/a
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a
 
操作系统的命令：
# timedatectl list-timezones
ist-timezones 列出系统上支持的时区
set-timezone 设定时区
set-time 设置时间
set-btp 设置同步ntp
 
示例:设置时区示例：
timedatec修改时区
timedatectl set-timezone "America/New_York"
 
# timedatectl set-timezone Asia/Shanghai
ntp设置：
yum -y install ntp 
systemctl enable ntpd 
systemctl start ntpd
 同步时间
ntpdate -u cn.pool.ntp.org
```

#### clickhouse提供了配置的参数选型
```xml
1.修改设置
sudo vim /etc/clickhouse-server/config.xml
 
<timezone>Asia/Shanghai</timezone>
 由于clickhouse是俄罗斯人主导开发的，默认设置为Europe/Moscow
2.重启服务器：
sudo service clickhouse-server restart
 
 
我们可以看到选型的说明如下：
 <!-- Server time zone could be set here.
         Time zone is used when converting between String and DateTime types,
          when printing DateTime in text formats and parsing DateTime from text,
          it is used in date and time related functions, if specific time zone was not passed as an argument.
         Time zone is specified as identifier from IANA time zone database, like UTC or Africa/Abidjan.
         If not specified, system time zone at server startup is used.
         Please note, that server could display time zone alias instead of specified name.
         Example: W-SU is an alias for Europe/Moscow and Zulu is an alias for UTC.
    -->
    <!-- <timezone>Europe/Moscow</timezone> -->
 
时区在日期时间相关的函数，若指定时区作为参数。在Datetime和String类型之间进行转换。
时区的指定是按照IANA标准的时区库指定的，可以在Linux系统中通过命令查询
若不指定则使用系统启动的时区。
```

#### 返回json时区不对解决方法
**statDate字段要使用@JsonFormat格式化日期字符串**
```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@TableName(value = "tb_stat")
@ApiModel(value = "Stat对象", description = "")
public class Stat implements Serializable {
    private static final long serialVersionUID = 1L;
    @ApiModelProperty(value = "ID")
    private String id;
    @ApiModelProperty(value = "区域")
    private String region;
    @ApiModelProperty(value = "分组")
    private String group;
    @ApiModelProperty(value = "昨天")
    private Integer yesterday;
    @ApiModelProperty(value = "今天")
    private Integer today;
    @ApiModelProperty(value = "时间")
    @JsonFormat(locale="zh", timezone="GMT+8", pattern="yyyy-MM-dd HH:mm:ss")
    private Date statDate;
```

**参考**
https://blog.csdn.net/qq_41671629/article/details/128231822?spm=1001.2101.3001.6650.11&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-11-128231822-blog-127635133.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-11-128231822-blog-127635133.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=13
- https://blog.csdn.net/vkingnew/article/details/107227037
- https://www.cnblogs.com/traditional/p/15238877.html
- https://blog.csdn.net/weixin_39168541/article/details/118606084 


### clickhouse数据文件目录移动

源目录：/var/lib/clickhouse
目标目录：/test/clickhouse

1. 复制数据
```shell
cp /var/lib/clickhouse/data -r  /test/clickhouse
cp /var/lib/clickhouse/flags -r  /test/clickhouse
cp /var/lib/clickhouse/format_schemas -r  /test/clickhouse
cp /var/lib/clickhouse/metadata -r  /test/clickhouse
cp /var/lib/clickhouse/preprocessed_configs -r  /test/clickhouse
cp /var/lib/clickhouse/tmp -r  /test/clickhouse
cp /var/lib/clickhouse/user_files -r  /test/clickhouse

```
2. 在目录/var/lib/clickhouse删除
```shell
rm -r data
rm -r flags/
rm -r format_schemas/
rm -r metadata/
rm -r preprocessed_configs/
rm -r tmp
rm -r user_files/
```
3. 建立软链接
```shell
ln -s /test/clickhouse/data /var/lib/clickhouse
ln -s /test/clickhouse/flags /var/lib/clickhouse
ln -s /test/clickhouse/format_schemas /var/lib/clickhouse
ln -s /test/clickhouse/metadata /var/lib/clickhouse
ln -s /test/clickhouse/preprocessed_configs /var/lib/clickhouse
ln -s /test/clickhouse/tmp /var/lib/clickhouse
ln -s /test/clickhouse/user_files /var/lib/clickhouse
```
4. 给/test/clickhouse 目录权限
```shell
chown -R clickhouse.clickhouse /test/clickhouse
```
**参考**
- https://blog.csdn.net/qq_21383435/article/details/113996322?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7EPayColumn-1-113996322-blog-122821087.pc_relevant_3mothn_strategy_and_data_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7EPayColumn-1-113996322-blog-122821087.pc_relevant_3mothn_strategy_and_data_recovery&utm_relevant_index=1
- https://blog.csdn.net/dair6/article/details/122821087