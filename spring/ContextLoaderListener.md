### ContextLoaderListener
每一个spring项目在配置时，在```web.xml```总少不了这一步：  
```
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```
我们查看```ContextLoaderListener```的实现可知它实现了```ServletContextListener```的两个接口即```contextInitialized```容器的初始化和```contextDestroyed```容器的销毁：  
```
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }

    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
    }

    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}

```