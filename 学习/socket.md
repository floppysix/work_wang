# websocket
## 在websocket使用@Autowired取出数据报错[空指针异常]
### 1. 在你的项目内新建一个名称为ApplicationContextRegister类，把下面代码复制进去，这个就是个工具类
```java
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Component;
 
 
@Component
@Lazy(false)
public class ApplicationContextRegister implements ApplicationContextAware {
    private static ApplicationContext APPLICATION_CONTEXT;
 
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        APPLICATION_CONTEXT = applicationContext;
    }
 
    public static ApplicationContext getApplicationContext() {
        return APPLICATION_CONTEXT;
    }
 
}

```
### 2. 请参考下面websocket配置代码，自行编写代码，VwWorkBcCheckMappe就是你的DAO层代码
```java
/**
     * 连接建立成功调用的方法*/
    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        webSocketSet.add(this);     //加入set中
        addOnlineCount();           //在线数加1
        System.out.println("有新连接加入:"+"！当前在线人数为" + getOnlineCount());
            try {
                VwWorkBcCheckMapper vwWorkBcCheckMapper =  (VwWorkBcCheckMapper) ApplicationContextRegister.getApplicationContext().getBean(VwWorkBcCheckMapper.class);
                ArrayList<VwWorkBcCheck> list =  vwWorkBcCheckMapper.getVwWorkBcCheck();
                String json= JSON.toJSONString(list);
                sendMessage(json);
            } catch (IOException e) {
                System.out.println("IO异常");
        }
    }

```

