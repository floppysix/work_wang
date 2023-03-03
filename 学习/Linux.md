
## 压缩
```
# 压缩
zip -r 压缩包名 要压缩的文件
# 解压
unzip 压缩包名
```


## vim 命令
```shell
在光标的位置按“yy”，复制当前行，然后再光标的行按“p”,粘贴到下一行，原来的往下顺移；
删除当前行-------dd
复制多行----------nyy(比如3yy，复制3行)
删除多行----------ndd
复制多遍----------np
```
## 查看文件夹大小
```SHELL
# 列出当前文件以及文件夹的大小
du -sh *
# 查看单独文件的大小(backup.sh 文件名)
du -s  backup.sh ，ls -lh backup.sh

```
## yum 安装失败
```bash
[root@base ~]# scp /bin/rpm root@hadoop102:/bin/rpm
[root@base ~]# scp /usr/lib/rpm/rpmrc  root@hadoop102:/usr/lib/rpm/rpmrc
[root@base ~]# scp /usr/lib/rpm/macros  root@hadoop102:/usr/lib/rpm

```

## Linux系统时间
```shell
# 用 ntpdate从时间服务器更新时间
yum -y install ntp
date
# 将本地时间与网络时间同步
ntpdate time.nist.gov

# 保存配置
hwclock -w
```

## 防火墙以及开放端口
```shell
systemctl  status firewalld.service查看防火墙的状态；

systemctl  start firewalld.service启动防火墙；

systemctl  stop firewalld.service关闭防火墙；

systemctl  restart firewalld.service重启防火墙；

systemctl  enable firewalld.service开机启动防火墙；

systemctl  disable firewalld.service开机禁用防火墙；

systemctl  is-enabled firewalld.service查看防火墙是否开机启动



// 开放端口命令
 firewall-cmd --zone=public --add-port=8123/tcp --permanent
 // 或者
 /sbin/iptables -I INPUT -p tcp --dport 28080 -j ACCEPT

 // 重启防火墙命令
 systemctl restart firewalld.service

```


## 日志查看
```shell

# 查看服务日志
journalctl -xeu kubelet
# 动态查看日志
tail -f catalina.ou
# 从头打开日志文件
cat catalina.ou
# 输出某个新日志去查看
cat -n catalina.out |grep 717892466 >nanjiangtest.txt
# 查询日志尾部最后number行的日志
tail -n number catalina.out
# 查询number行之后的所有日志
tail -n +number catalina.out 
# 查询日志文件中的前number行日志
head -n number catalina.out
# 查询日志文件除了最后number行的其他所有日志
head -n -number catalina.out

根据关键词获得行号
# 得到关键日志的行号
cat -n test.log | grep “关键词” 

cat -n catalina.out|tail -n +13230539|head -n 10
tail -n +13230539表示查询13230539行之后的日志
head -n 10则表示在前面的查询结果里再查前10条记录

查看指定时间段内的日志
1. 首先要进行范围时间段内日志查询先查看是否在当前日之内存在
grep '11:07 18:29:20' catalina.out
grep '11:07 18:31:11' catalina.out
2. 时间范围内的查询
sed -n '/11:07 18:29:20/,/11:07 18:31:11/p' catalina.out 
sed -n '/11:07 18:29:/,/11:07 18:31:/p' catalina.out


查看日志中特定字符的匹配数目
grep '1175109632' catalina.out | wc -l

查询最后number行，并查找关键字“结果”
tail -n 20 catalina.out | grep 'INFO Takes:1'

查询最后number行，并查找关键字“结果”并且对结果进行标红
tail -n 20 catalina.out | grep 'INFO Takes:1' --color

查询最后number行，并查找关键字“结果”并且对结果进行标红，上下扩展两行
tail -n 20 catalina.out | grep 'INFO Takes:1' --color -a2

分页查看，使用空格翻页(使用more/less)
tail -n 2000 catalina.out | grep 'INFO Takes:1' --color -a2 | more
tail -n 2000 catalina.out | grep 'INFO Takes:1' --color -a2 | less
```