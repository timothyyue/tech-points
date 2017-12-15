主要用于框架中，动态生成代码，加载到jvm中，涉及到主要类包括：
ctClass, ctMethod, ctField, CtConstructor等，附demo如下


**Demo**  

```java

import javassist.*;

import java.io.IOException;


public class Demo {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IOException, IllegalAccessException, InstantiationException {


        ClassPool pool = ClassPool.getDefault();
        CtClass pt = pool.makeClass("asdf", pool.get("com.jumei.arch.assist.Bean"));

        CtMethod method1 = new CtMethod(pool.get("java.lang.String"), "getM", null, pt);
        method1.setBody("{return \"你好\";}");
        pt.addMethod(method1);

        CtConstructor cc = new CtConstructor(null, pt);
        cc.setBody("this.field=\"why?\";");
        pt.addConstructor(cc);


        CtMethod method2 = new CtMethod(CtClass.voidType, "setF", new CtClass[]{pool.get("java.lang.String")}, pt);
        method2.setBody("this.field=$1;");
        pt.addMethod(method2);

        Bean bean = (Bean) pt.toClass().newInstance();
        System.out.println(bean.getM());
        System.out.println(bean.getF());
        bean.setF("setf");
        System.out.println(bean.getF());

    }
}
```

Bean  

```java

public abstract class Bean {

    public String field;

    public abstract String getM();

    public abstract void setF(String f);

    public String getF() {
        return this.field;
    }
}
```