## 日志查看
```shell
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