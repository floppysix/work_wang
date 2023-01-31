# windows
[[登录]]
```
mysql -h 主机名 -P 端口号 -u 用户名 -p密码


```

# Linux安装MySQL
```
//卸载虚拟机中的mysql  
 rpm -qa | grep -i -E mysql\|mariadb | xargs -n1 sudo rpm -e --nodeps  
//安装mysql依赖  
rpm -ivh 01_mysql-community-common-5.7.16-1.el7.x86_64.rpm   
rpm -ivh 02_mysql-community-libs-5.7.16-1.el7.x86_64.rpm  
rpm -ivh 03_mysql-community-libs-compat-5.7.16-1.el7.x86_64.rpm  
//安装mysql-client  
rpm -ivh 04_mysql-community-client-5.7.16-1.el7.x86_64.rpm   
//安装mysql-server  
rpm -ivh 05_mysql-community-server-5.7.16-1.el7.x86_64.rpm  
//启动mysql  
systemctl start mysqld  
//查看MySQL密码  
cat /var/log/mysqld.log | grep password
```

**配置mysql**
```
//登录MySQL（密码要用单引号''）  
mysql -uroot -p'OXdHF(nvr7EP'  
//设置mysql密码（MySQL密码要求复杂）  
set password=password("Qs23=zs23");  
//更改mysql密码策略  
set global validate_password_length=4;  
set global validate_password_policy=0;  
//设置简单密码  
set password=password("000000");  
//进入mysql库  
use mysql  
//查询user表  
select user, host from user;  
//修改user表，把host表内容修改为%  
update user set host="%" where user="root";  
//刷新  
flush privileges;  
//退出  
quit
```

# 一些问题
## 数据库实现id主键删除之后不自增的问题(没有实验)
```mysql
alter table info auto_increment = 1
```
从id的最大位数开始增加，删除数据之后最大id是9，所以就从10开始增加了

# 数据库常用字段
**参考**  https://blog.csdn.net/weixin_45369440/article/details/116742709?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-116742709-blog-116891833.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-116742709-blog-116891833.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=2
```mysql
DROP TABLE IF EXISTS `tablename`;
CREATE TABLE `tablename` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT,
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `is_deleted` tinyint(1) unsigned zerofill DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3;

```