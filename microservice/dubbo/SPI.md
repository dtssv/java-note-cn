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