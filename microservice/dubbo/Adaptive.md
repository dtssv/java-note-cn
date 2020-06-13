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
其中代码生成逻辑大同小异，只是根据方法入参是否有```com.alibaba.dubbo.common.URL```类型或者是否有```com.alibaba.dubbo.rpc.Invocation```或者入参的属性是否有```com.alibaba.dubbo.common.URL```，或者接口是否为```com.alibaba.dubbo.rpc.Protocol```有略微差异。  
当入参有```com.alibaba.dubbo.common.URL```时，生成代码如下:  
```
        if (arg0 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("transporter", defaultExtName);
```
当入参不包含但是入参的属性包含```com.alibaba.dubbo.common.URL```时，生成代码如下：
```
        if (arg0 == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("transporter", defaultExtName);
```
当入参包含```com.alibaba.dubbo.rpc.Invocation```时，生成代码如下：
```
        if (arg0 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        if (arg1 == null) 
            throw new IllegalArgumentException("invocation == null");
        String methodName = arg1.getMethodName();
        String extName = url.getMethodParameter(methodName, "transporter", defaultExtName);

```
当接口为```com.alibaba.dubbo.rpc.Protocol```时，生成代码如下：
```
        if (arg0 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = ( url.getProtocol() == null ? defaultExtName : url.getProtocol() );
```
其中```defaultExtName```为```SPI```注解默认值，如果无默认值，则以上代码微调为类似```String extName =  url.getProtocol()```或```String extName = url.getParameter("transporter")```，不判断获取结果为空取默认值。  
##### 源码分析
```
    public static Transporter getTransporter() {
        //获取Transporter的ExtensionLoader，重要的方法是getAdaptiveExtension()
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
    //ExtensionLoader
    public T getAdaptiveExtension() {
        //从缓存获取代理对象
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            //createAdaptiveInstanceError默认为空
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            //重要方法，创建代理对象
                            instance = createAdaptiveExtension();
                            //将创建的代理对象放入缓存
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }
        return (T) instance;
    }

    private T createAdaptiveExtension() {
        try {
            //getAdaptiveExtensionClass生成代理对象，注入参数
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }

    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }

    private Class<?> createAdaptiveExtensionClass() {
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        //编译生成的代码，使用javassist扩展类编译
        return compiler.compile(code, classLoader);
    }
    
    private String createAdaptiveExtensionClassCode() {
        StringBuilder codeBuilder = new StringBuilder();
        //获取接口所有方法
        Method[] methods = type.getMethods();
        //接口所有方法是否有方法被@Adaptive注解修饰
        boolean hasAdaptiveAnnotation = false;
        for (Method m : methods) {
            if (m.isAnnotationPresent(Adaptive.class)) {
                hasAdaptiveAnnotation = true;
                break;
            }
        }
        //所有方法都未被@Adaptive注解修饰，无须生成代理类
        if (!hasAdaptiveAnnotation)
            throw new IllegalStateException("No adaptive method on extension " + type.getName() + ", refuse to create the adaptive class!");
        //package ***;
        //import ExtensionLoader;
        //public class ***$Adaptive implements ***{
        codeBuilder.append("package ").append(type.getPackage().getName()).append(";");
        codeBuilder.append("\nimport ").append(ExtensionLoader.class.getName()).append(";");
        codeBuilder.append("\npublic class ").append(type.getSimpleName()).append("$Adaptive").append(" implements ").append(type.getCanonicalName()).append(" {");

        for (Method method : methods) {
            Class<?> rt = method.getReturnType();//获取返回值
            Class<?>[] pts = method.getParameterTypes();//获取参数
            Class<?>[] ets = method.getExceptionTypes();//获取异常

            Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
            StringBuilder code = new StringBuilder(512);
            if (adaptiveAnnotation == null) {
                //如果方法不被@Adaptive注解修饰，该方法调用直接抛出异常
                code.append("throw new UnsupportedOperationException(\"method ")
                        .append(method.toString()).append(" of interface ")
                        .append(type.getName()).append(" is not adaptive method!\");");
            } else {
                int urlTypeIndex = -1;
                //在参数中查找类型为URL.class的参数，如果没有urlTypeIndex=-1，有则urlTypeIndex为对应参数下标
                for (int i = 0; i < pts.length; ++i) {
                    if (pts[i].equals(URL.class)) {
                        urlTypeIndex = i;
                        break;
                    }
                }
                if (urlTypeIndex != -1) {
                    //参数中发现URL.class类型
                    // 生成空值判断代码
                    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                            urlTypeIndex);
                    code.append(s);
                    //将参数赋值给url对象
                    s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
                    code.append(s);
                } else {
                    String attribMethod = null;
                    //参数列表中没有URL.class类型对象，则在参数的所有方法中查找getUrl()方法，如果找到直接返回
                    LBL_PTS:
                    for (int i = 0; i < pts.length; ++i) {
                        Method[] ms = pts[i].getMethods();
                        for (Method m : ms) {
                            String name = m.getName();
                            if ((name.startsWith("get") || name.length() > 3)
                                    && Modifier.isPublic(m.getModifiers())
                                    && !Modifier.isStatic(m.getModifiers())
                                    && m.getParameterTypes().length == 0
                                    && m.getReturnType() == URL.class) {
                                urlTypeIndex = i;
                                attribMethod = name;
                                break LBL_PTS;
                            }
                        }
                    }
                    //如果找不到则抛出异常
                    if (attribMethod == null) {
                        throw new IllegalStateException("fail to create adaptive class for interface " + type.getName()
                                + ": not found url parameter or url attribute in parameters of method " + method.getName());
                    }
                    //空值检测，将对象的getUrl()赋值给url对象
                    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
                            urlTypeIndex, pts[urlTypeIndex].getName());
                    code.append(s);
                    s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
                            urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
                    code.append(s);
                    s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
                    code.append(s);
                }

                String[] value = adaptiveAnnotation.value();//获取@Adaptive注解的值，如果没有则将类名放到数组里
                if (value.length == 0) {
                    char[] charArray = type.getSimpleName().toCharArray();
                    StringBuilder sb = new StringBuilder(128);
                    for (int i = 0; i < charArray.length; i++) {
                        if (Character.isUpperCase(charArray[i])) {
                            if (i != 0) {
                                sb.append(".");
                            }
                            sb.append(Character.toLowerCase(charArray[i]));
                        } else {
                            sb.append(charArray[i]);
                        }
                    }
                    //最终sb值为类似protocol或者servlet.http.binder
                    value = new String[]{sb.toString()};
                }
                boolean hasInvocation = false;//判断参数中是否有Invocation.class类型的对象
                for (int i = 0; i < pts.length; ++i) {
                    if (pts[i].getName().equals("com.alibaba.dubbo.rpc.Invocation")) {
                        // Invocation空值判断
                        String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
                        code.append(s);
                        //获取Invocation对象的methodName
                        s = String.format("\nString methodName = arg%d.getMethodName();", i);
                        code.append(s);
                        hasInvocation = true;
                        break;
                    }
                }

                String defaultExtName = cachedDefaultName;//SPI的默认值，无为空，如Transporter的默认值为netty，
                String getNameCode = null;
                for (int i = value.length - 1; i >= 0; --i) {
                    if (i == value.length - 1) {
                        if (null != defaultExtName) {
                            if (!"protocol".equals(value[i]))
                                if (hasInvocation)
                                    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                                else
                                    getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                        } else {
                            if (!"protocol".equals(value[i]))
                                if (hasInvocation)
                                    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                                else
                                    getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                            else
                                getNameCode = "url.getProtocol()";
                        }
                    } else {
                        if (!"protocol".equals(value[i]))
                            if (hasInvocation)
                                getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                        else
                            getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
                    }
                }
                code.append("\nString extName = ").append(getNameCode).append(";");//获取实际希望获取的扩展类型
                // 空值判断
                String s = String.format("\nif(extName == null) " +
                                "throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
                        type.getName(), Arrays.toString(value));
                code.append(s);
                //加载实际希望获取的扩展类型
                s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
                        type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
                code.append(s);

                // 调用方法
                if (!rt.equals(void.class)) {
                    code.append("\nreturn ");
                }

                s = String.format("extension.%s(", method.getName());
                code.append(s);
                for (int i = 0; i < pts.length; i++) {
                    if (i != 0)
                        code.append(", ");
                    code.append("arg").append(i);
                }
                code.append(");");
            }

            codeBuilder.append("\npublic ").append(rt.getCanonicalName()).append(" ").append(method.getName()).append("(");
            for (int i = 0; i < pts.length; i++) {
                if (i > 0) {
                    codeBuilder.append(", ");
                }
                codeBuilder.append(pts[i].getCanonicalName());
                codeBuilder.append(" ");
                codeBuilder.append("arg").append(i);
            }
            codeBuilder.append(")");
            if (ets.length > 0) {
                codeBuilder.append(" throws ");
                for (int i = 0; i < ets.length; i++) {
                    if (i > 0) {
                        codeBuilder.append(", ");
                    }
                    codeBuilder.append(ets[i].getCanonicalName());
                }
            }
            codeBuilder.append(" {");
            codeBuilder.append(code.toString());
            codeBuilder.append("\n}");
        }
        codeBuilder.append("\n}");
        if (logger.isDebugEnabled()) {
            logger.debug(codeBuilder.toString());
        }
        return codeBuilder.toString();
    }
```
上边分析了自适应扩展相关逻辑，类生成完成会向对象注入属性，注入实现方法为```injectExtension```     
```
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }

```
通过源码可知，注入实现方式是通过反射，获取类的所有set方法，如```public void setProtocol(Protocol protocol)```，通过```objectFactory.getExtension```获取注入对象，下面主要分析```objectFactory```初始化，和如何生成注入对象两部分。  
```objectFactory```初始化过程分析：  
```
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        //...各种判断忽略
        //从缓存获取，如果无法获取到就构造ExtensionLoader
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
    //ExtensionLoader构造方法，看到这里就感觉像是陷入了一个循环
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```
上边的方法像是陷入了一个怪圈，那么```objectFactory```初始化到底是在哪初始化的呢，其实是上边自适应扩展生成逻辑中一个被忽略的方法，即```getAdaptiveExtensionClass()```，那么我们来再分析这个方法： 
```
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        //...忽略
    }
    private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
    private Map<String, Class<?>> loadExtensionClasses() {
        //获取SPI默认值并赋值给cachedDefaultName，代码忽略
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        //重点关注
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
        //fileName = META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    loadResource(extensionClasses, classLoader, resourceURL);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
    private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), "utf-8"));
            try {
                String line;
                while ((line = reader.readLine()) != null) {
                    final int ci = line.indexOf('#');
                    if (ci >= 0) line = line.substring(0, ci);
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                            String name = null;
                            int i = line.indexOf('=');
                            if (i > 0) {
                                name = line.substring(0, i).trim();
                                line = line.substring(i + 1).trim();
                            }
                            if (line.length() > 0) {
                                //重点关注，上边都是读取SPI文件
                                loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                            exceptions.put(line, e);
                        }
                    }
                }
            } finally {
                reader.close();
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }

    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error when load extension class(interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + "is not subtype of interface.");
        }
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            //看到这里想必大家都已经清楚了，如果ExtensionFactory的实现类有被@Adaptive注解修饰，那么那个类就是objectFactory
            if (cachedAdaptiveClass == null) {
                cachedAdaptiveClass = clazz;
            } else if (!cachedAdaptiveClass.equals(clazz)) {
                throw new IllegalStateException("More than 1 adaptive class found: "
                        + cachedAdaptiveClass.getClass().getName()
                        + ", " + clazz.getClass().getName());
            }
        } else if (isWrapperClass(clazz)) {
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers == null) {
                cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                wrappers = cachedWrapperClasses;
            }
            wrappers.add(clazz);
        } else {
            clazz.getConstructor();
            if (name == null || name.length() == 0) {
                name = findAnnotationName(clazz);
                if (name == null || name.length() == 0) {
                    if (clazz.getSimpleName().length() > type.getSimpleName().length()
                            && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                        name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                    } else {
                        throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                    }
                }
            }
            String[] names = NAME_SEPARATOR.split(name);
            if (names != null && names.length > 0) {
                Activate activate = clazz.getAnnotation(Activate.class);
                if (activate != null) {
                    cachedActivates.put(names[0], activate);
                }
                for (String n : names) {
                    if (!cachedNames.containsKey(clazz)) {
                        cachedNames.put(clazz, n);
                    }
                    Class<?> c = extensionClasses.get(n);
                    if (c == null) {
                        extensionClasses.put(n, clazz);
                    } else if (c != clazz) {
                        throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                    }
                }
            }
        }
    }

```
通过上边的逻辑我们知道如果```ExtensionFactory```的实现类有被```@Adaptive```注解修饰，那么那个类就是```objectFactory```，我们看对应的实现有三个，SPI文件内容为：  
```
//看名字就可以猜到默认是它，
adaptive=com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
spi=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory
spring=com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory
//不放心我们再看一下源码
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {
```
上边的逻辑分析了```objectFactory```的初始化，那么又是怎么通过```objectFactory```像其他bean注入对象的呢，我们重点需要关注```Object object = objectFactory.getExtension(pt, property);```，下边我们来分析源码：  
```
    //AdaptiveExtensionFactory
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        //此处遍历spi和spring的ExtensionFactory实现并获取对应type实例的初始化，如果能够获取到则直接返回，现在我们期望的注入对象是Protocol，因为Protocol对象被SPI注解修饰，那么我们重点关注SpiExtensionFactory
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
    //SpiExtensionFactory
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                //回归上边自适应扩展逻辑，生成代理对象
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }
    //SpringExtensionFactory逻辑分析忽略
```
自适应扩展和```SPI```其实是相辅相成的，不过因为自适应扩展逻辑较多所以分开分析。