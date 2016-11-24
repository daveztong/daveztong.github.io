---
title: Dubbo学习-理解动态代理
date: 2016-11-23 22:38:15
categories: dubbo
tags: [dubbo,proxy,javassist]
---

# Dubbo学习-理解动态代理



在之前的一篇post中了解了[spring可扩展的XML配置](http://daveztong.github.io/2016/11/01/Spring%E5%8F%AF%E6%89%A9%E5%B1%95%E7%9A%84XML%E9%85%8D%E7%BD%AE/)是怎么一会事，接下来继续研究dubbo consumer端如何解析service并执行远程调用。

## 本次研究目标

1. 代理如何创建的。仅仅只是配置了`<dubbo:reference interface="x.y.z.ServiceInterface" id="serviceId"/>`并将其交给了spring container，然后直接注入并使用该接口的方法就可以完成调用了，然而我并没有为该接口实现具体的类，how does it works? 
2. ~~远程调用如何执行的。假设已经有了具体的实现类，怎么实现远程调用的呢，Thingking?~~ 由于第一个分析就很长，这个目标列入下一次分析。

<!-- more -->

## 预备知识

为了顺利的研究上述两个目标，需要了解以下知识:

1. SPI. [官方教程](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html)。用于服务扩展。
2. Netty. [User Guide](http://netty.io/wiki/user-guide-for-4.x.html)。用于远程调用。
3. Javassist. [官方Tutorial](https://jboss-javassist.github.io/javassist/tutorial/tutorial.html)。动态类生成。



## How to dig in?

从何处入手是个问题，我的习惯是顺藤摸瓜，所以从service被注入开始:

dubbo service配置:

```xml
<!-- DemoService is just a interface -->
<dubbo:reference interface="x.y.z.DemoService" id="demoService"/>
```

Service注入:

```java
@Resource
private DemoService demoService;
```

当使用这个`demoService`的时候可以肯定的是一定有个DemoService的具体实现可供使用，`DubboNamespaceHandler`中有这样一段代码:

```java
registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class,false));
```

解析上面配置在namespace dubbo下的reference。`DubboBeanDefinitionParser`相关代码如下:

```java
public DubboBeanDefinitionParser(Class<?> beanClass, boolean required) {
  this.beanClass = beanClass;
  this.required = required;
}

public BeanDefinition parse(Element element, ParserContext parserContext) {
  return parse(element, parserContext, beanClass, required);
}

// 关键部分
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    // 设置bean class,in this case it is ReferenceBean.class
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    String id = element.getAttribute("id");
    // some ops...
    if (id != null && id.length() > 0) {
        if (parserContext.getRegistry().containsBeanDefinition(id))  {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    
    // some ops...

    // 遍历setter and getter设置属性值
    for (Method setter : beanClass.getMethods()) {
        String name = setter.getName();
        if (name.length() > 3 && name.startsWith("set")
                && Modifier.isPublic(setter.getModifiers())
                && setter.getParameterTypes().length == 1) {
            Class<?> type = setter.getParameterTypes()[0];
            String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
            if(...){
                // some ops...
            }else {
                String value = element.getAttribute(property);
                if (value != null) {
                    value = value.trim();
                    if (value.length() > 0) {
                           // 部分省略...
                            Object reference;
                            if (isPrimitive(type)) {
                                if ("async".equals(property) && "false".equals(value)
                                        || "timeout".equals(property) && "0".equals(value)
                                        || "delay".equals(property) && "0".equals(value)
                                        || "version".equals(property) && "0.0.0".equals(value)
                                        || "stat".equals(property) && "-1".equals(value)
                                        || "reliable".equals(property) && "false".equals(value)) {
                                    // 兼容旧版本xsd中的default值
                                    value = null;
                                }
                                reference = value;
                            } 

                            // 大批省略...

                            // 这里会将interface这个attribute的值设置在ReferenceBean的父类ReferenceConfig的interfaceName上通过method:setInterface(String)
                            beanDefinition.getPropertyValues().addPropertyValue(property, reference);
                        }
                    }
                }
            }
        }
    }
    // 省略...
    return beanDefinition;
}
```

上面两个关键的属性值id,interface都设置在了bean上，后面会用到。ReferenceConfig中的settter:setInterface用来设置interface：

```java
public void setInterface(String interfaceName) {
        this.interfaceName = interfaceName;
        if (id == null || id.length() == 0) {
            id = interfaceName;
        }
    }
```

参考ReferenceBean Hierarchy:

![ReferenceBean Hierarchy](http://ww1.sinaimg.cn/mw690/50508d62gw1fa1b4h76xgj21960van07.jpg)



ReferenceBean中包含以几个两个方法:

```java
@Override
// inherit from ApplicationContextAware
public void setApplicationContext(ApplicationContext applicationContext) {
  this.applicationContext = applicationContext;
  SpringExtensionFactory.addApplicationContext(applicationContext);
}

@Override
// inherit from FactoryBean
public Object getObject() throws Exception {
  return get();// 调用ReferenceConfig#get()方法创建类型为getObjectType()的对象，which in this case is instance of x.y.z.DemoService
}
// inherit from FactoryBean
public Class<?> getObjectType() {
  return getInterfaceClass();
}
```

如此，重点关注ReferenceConfig#get(),follow up:

```java
public synchronized T get() {
  if (destroyed){
    throw new IllegalStateException("Already destroyed!");
  }
  if (ref == null) {
    init();
  }
  return ref;
}
```

进入init():

```java
private void init() {
    // some ops...
     {
        try {
            // 之前设置的interface:x.y.z.DemoService
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
    }
    // a lot of ops...
    ref = createProxy(map);
}
```

进入createProxy(Map):

```java
private T createProxy(Map<String, String> map) {
    //some ops...

    if (urls.size() == 1) {
        invoker = refprotocol.refer(interfaceClass, urls.get(0));
    } 

    // some ops...

    // 创建服务代理
    return (T) proxyFactory.getProxy(invoker);
}
```

到这里已经知道代理是由ProxyFactory创建了，接下重点看看ProxyFactory.

## ProxyFactory

ProxyFactory  Hierarchy:

![ProxyFactory Hierarchy](http://ww4.sinaimg.cn/mw690/50508d62gw1fa27gqakyxj20l8080gmy.jpg)





### AbstractProxyFactory

```java
public <T> T getProxy(Invoker<T> invoker) throws RpcException {
  Class<?>[] interfaces = null;
  // createProxy时创建invoker时已将interface传入
  String config = invoker.getUrl().getParameter("interfaces");
  if (config != null && config.length() > 0) {
    String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
    if (types != null && types.length > 0) {
      interfaces = new Class<?>[types.length + 2];
      interfaces[0] = invoker.getInterface();
      interfaces[1] = EchoService.class;
      for (int i = 0; i < types.length; i ++) {
        interfaces[i + 1] = ReflectUtils.forName(types[i]);
      }
    }
  }
  if (interfaces == null) {
    interfaces = new Class<?>[] {invoker.getInterface(), EchoService.class};
  }
  // 调用子类的实现
  return getProxy(invoker, interfaces);
```

### JavassistProxyFactory

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

`getProxy`的相关代码:

```java
public static Proxy getProxy(Class<?>... ics)
{
  return getProxy(ClassHelper.getCallerClassLoader(Proxy.class), ics);
}
```

动态类的实现:

```java
public static Proxy getProxy(ClassLoader cl, Class<?>... ics)
    {
        //some ops...
        try
        {
            ccp = ClassGenerator.newInstance(cl);

            Set<String> worked = new HashSet<String>();
            List<Method> methods = new ArrayList<Method>();

            // 反射获取interface的相关信息并build code string，然后交给javassist动态生成实现类。
            for(int i=0;i<ics.length;i++)
            {
                if( !Modifier.isPublic(ics[i].getModifiers()) )
                {
                    String npkg = ics[i].getPackage().getName();
                    if( pkg == null )
                    {
                        pkg = npkg;
                    }
                    else
                    {
                        if( !pkg.equals(npkg)  )
                            throw new IllegalArgumentException("non-public interfaces from different packages");
                    }
                }
                ccp.addInterface(ics[i]);

                for( Method method : ics[i].getMethods() )
                {
                    String desc = ReflectUtils.getDesc(method);
                    if( worked.contains(desc) )
                        continue;
                    worked.add(desc);

                    int ix = methods.size();
                    Class<?> rt = method.getReturnType();
                    Class<?>[] pts = method.getParameterTypes();

                    StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                    for(int j=0;j<pts.length;j++)
                        code.append(" args[").append(j).append("] = ($w)$").append(j+1).append(";");
                    // 注意这里 handler.invoke(),代理的统一处理
                    code.append(" Object ret = handler.invoke(this, methods[" + ix + "], args);");
                    if( !Void.TYPE.equals(rt) )
                        code.append(" return ").append(asArgument(rt, "ret")).append(";");

                    methods.add(method);
                    ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
                }
            }

            if( pkg == null )
                pkg = PACKAGE_NAME;

            // 接口的实现类
            String pcn = pkg + ".proxy" + id;
            ccp.setClassName(pcn);
            ccp.addField("public static java.lang.reflect.Method[] methods;");
            ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
            ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{ InvocationHandler.class }, new Class<?>[0], "handler=$1;"); // $1等表示传入的参数，具体参考javassist官方文档
            ccp.addDefaultConstructor();
            Class<?> clazz = ccp.toClass();
            clazz.getField("methods").set(null, methods.toArray(new Method[0]));

            // 生成当前Proxy的子类，实现newInstance()方法
            String fcn = Proxy.class.getName() + id;
            ccm = ClassGenerator.newInstance(cl);
            ccm.setClassName(fcn);
            ccm.addDefaultConstructor();
            ccm.setSuperClass(Proxy.class);
            ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
            Class<?> pc = ccm.toClass();
            proxy = (Proxy)pc.newInstance();
        }
        // some ops...
        return proxy;
    }
```

`toClass()` 为ClassGenerator的方法，实现也是通过javassist生成动态类。 这儿返回的是Proxy的子类实例。JavassistProxyFactory的getProxy中`Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));`这儿的newInstance()就属于这个子类实例。该方法的定义：

```java
abstract public Object newInstance(InvocationHandler handler);
```

再来看看InvocationHandler的实现:

```java
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;
    
    public InvokerInvocationHandler(Invoker<?> handler){
        this.invoker = handler;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        // 返回远程调用的结果
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }

}
```

看到这儿可能还有点迷糊，invoke是在哪里调用的呢？回头看看动态类生成部分有这样一段代码:

`code.append(" Object ret = handler.invoke(this, methods[" + ix + "], args);");`这就将代理调用衔接起来了。看起来可能有点抽象，待下面来试验一把。



#### 试验

`DemoService.java`

```java
package com.alibaba.dubbo.examples.x.y.z;

/**
 * Created by tangwei on 2016/11/23.
 */
public interface DemoService {
    void sayHi(String name);
}
```

Main.java

```java
package com.alibaba.dubbo.examples.demo;

import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.examples.x.y.z.DemoService;
import com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory;
import com.alibaba.dubbo.rpc.support.MockInvoker;

/**
 * Created by tangwei on 2016/11/22.
 */
public class Main {
    public static void main(String[] args) {
        new JavassistProxyFactory().getProxy(new MockInvoker<DemoService>(new URL("", "", 8888)),
                new Class[]{DemoService.class});

    }
}
```

在`ClassGenerator`中 toClass返回前加入以下代码:

```java
try {
  // 将class写入文件
  mCtc.writeFile("/path/to/save/classfile/"+mSuperClass);
} catch (IOException e) {
  e.printStackTrace();
}

return mCtc.toClass(loader, pd);
```

debug一下main,在动态生成类的时候将类写入文件中，结果如下：

DemoService的实现:

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.common.bytecode.ClassGenerator.DC;
import com.alibaba.dubbo.examples.x.y.z.DemoService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class proxy0 implements DC, DemoService {
    public static Method[] methods;
    private InvocationHandler handler;

    public void sayHi(String var1) {
        Object[] var2 = new Object[]{var1};
        this.handler.invoke(this, methods[0], var2);
    }

    public proxy0() {
    }

    public proxy0(InvocationHandler var1) {
        this.handler = var1;
    }
}
```

Proxy的子类:

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.common.bytecode.Proxy;
import com.alibaba.dubbo.common.bytecode.proxy0;
import com.alibaba.dubbo.common.bytecode.ClassGenerator.DC;
import java.lang.reflect.InvocationHandler;

public class Proxy0 extends Proxy implements DC {
    public Object newInstance(InvocationHandler var1) {
        // 返回的是DemoService的实例
        return new proxy0(var1);
    }

    public Proxy0() {
    }
}
```

其中DC接口只是一个标识接口，表示该类是动态生成的。这样看起来就比较清晰明了了。下面再来看看jdk的proxy。

### JdkProxyFactory

相关代码如下:

```java
public class JdkProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
    }

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                Method method = proxy.getClass().getMethod(methodName, parameterTypes);
                return method.invoke(proxy, arguments);
            }
        };
    }

}
```

可以看到`Proxy.newProxyInstance`直接使用的是JDK的动态代理机制，因此就不再跟下去了。

## 总结

至此，对dubbo本地动态代理有一个清晰的理解了，总的来说一路顺藤摸瓜还算顺畅。对于选择哪一中方式作为生产使用，Dubbo推荐使用Javassist的代理机制，因为jdk原生的动态代理性能较差，生产环境不宜使用。