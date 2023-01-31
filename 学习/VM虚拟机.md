## 没有ens33 文件
- 当关闭主机时，没有关闭虚拟机会出现
- 参考：https://blog.csdn.net/qq_44823283/article/details/125817375
- 解决方案：
```
ifconfig ens33 up
systemctl stop NetworkManager
systemctl disable NetworkManager
ifup ens33
service network restart
```
- 结果![[1666871408096.png]]
