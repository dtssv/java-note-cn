#### prepareRefresh
整个过程中```prepareRefresh()```完成了刷新前的准备工作，核心代码如下：  
```
	protected void prepareRefresh() {
		// 切换active状态.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);
		....
		//由子类实现，初始化propertySource，其实现是MutablePropertySources
		//其内里实现是一个COWList，private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();
		initPropertySources();
		//验证属性合法性，一般情况下可以继承对应ApplicationContext类添加
		getEnvironment().validateRequiredProperties();
		....
	}
```
举例添加校验属性：  
```
public class CustomApplicationContext extends XmlWebApplicationContext {

    @Override
    protected void initPropertySources() {
        super.initPropertySources();
        //把"MYSQL_HOST"作为启动的时候必须验证的环境变量
        getEnvironment().setRequiredProperties("MYSQL_HOST");
    }
}
```
```initPropertySources()```初始化的是我们的所有的配置信息，初始化有严格的顺序，如果一个key在第一个```PropertySource```被找到，那么之后其他```PropertySource```的同名key会被忽略，所以我们要实现自己的配置中心就是要将我们的```PropertySource```放在首位：  
```
	protected void initPropertySources() {
		//此处env类型为StandardServletEnvironment
		ConfigurableEnvironment env = getEnvironment();
		//env.propertySourceList就是PropertySource属性列表
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
```