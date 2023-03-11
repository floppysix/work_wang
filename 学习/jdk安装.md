- 配置环境变量
```
tar -zxvf jdk-8u212-linux-x64.tar.gz -C /mnt/pest/module/


vim /etc/profile.d/my_env.sh

#JAVA_HOME
export JAVA_HOME=/mnt/pest/module/jdk1.8.0_212
export PATH=$PATH:$JAVA_HOME/bin

source /etc/profile
```