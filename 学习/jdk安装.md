- 配置环境变量
```
vim /etc/profile.d/my_env.sh

#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_212
export PATH=$PATH:$JAVA_HOME/bin

source /etc/profile
```