[TOC]
#### SPI
SPI全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来启用框架扩展和替换组。  
##### Java SPI
1.定义接口
```
public interface IShout {
    void shout();
}
```
2.定义实现类
```
public class Cat implements IShout {
    @Override
    public void shout() {
        System.out.println("miao miao");
    }
}
public class Dog implements IShout {
    @Override
    public void shout() {
        System.out.println("wang wang");
    }
}
```
3.定义SPI接口文件，文件路径为```src->main->resources->META-INF->services->com.demo.IShout```，文件内容为
```
com.demo.impl.Cat
com.demo.impl.Dog
```
4.使用
```
public class SPIMain {
    public static void main(String[] args) {
        ServiceLoader<IShout> shouts = ServiceLoader.load(IShout.class);
        for (IShout s : shouts) {
            s.shout();
        }
    }
}
```
##### Dubbo SPI
Dubbo 并未使用 Java SPI，而是重新实现了一套功能更强的 SPI 机制。Dubbo SPI 的相关逻辑被封装在了 ExtensionLoader 类中，通过 ExtensionLoader，我们可以加载指定的实现类。
1、定义接口
```
@SPI
public interface IShout {
    void shout();
}
```
2.定义实现类
```
public class Cat implements IShout {
    @Override
    public void shout() {
        System.out.println("miao miao");
    }
}
public class Dog implements IShout {
    @Override
    public void shout() {
        System.out.println("wang wang");
    }
}
```
3.定义SPI接口文件，dubbo会从以下三个路径加载SPI接口文件，文件路径可为  
```src->main->resources->META-INF->services->com.demo.IShout```  
```src->main->resources->META-INF->dubbo->com.demo.IShout``` 
```src->main->resources->META-INF->dubbo->internal->com.demo.IShout``` 

文件内容为
```
cat=com.demo.impl.Cat
dog=com.demo.impl.Dog
```
4.使用
```
public class SPIMain {
    public static void main(String[] args) {
        ExtensionLoader<IShout> extensionLoader = 
            ExtensionLoader.getExtensionLoader(IShout.class);
        IShout cat = extensionLoader.getExtension("cat");
        cat.shout();
        IShout dog = extensionLoader.getExtension("dog");
        dog.shout();
    }
}
```
#### Adaptive
即自适应扩展，自适应扩展通过代理模式实现。目的是只有当扩展类使用时才加载目标类。自适应扩展类需要被注解@Adaptive修饰，被@Adaptive修饰的方法会被生成代理方法，如果方法未被修饰，则默认抛出异常。下面以dubbo.Transporter举例。   
Transporter源码如下：
```
@SPI("netty")
public interface Transporter {
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```
Transporter生成代理类代码如下：
```
package com.alibaba.dubbo.remoting;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Transporter$Adaptive implements Transporter{
    public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException{
        if (arg0 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("transporter", "netty");
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(Transporter) name from url(" + url.toString() + ") use keys(server,transport)");
        Transporter extension = (Transporter)ExtensionLoader.getExtensionLoader(Transporter.class).getExtension(extName);
        return extension.bind(arg0,arg1);
    }
    //忽略
}
```
#### Server
#### Client
#### LoadBlance