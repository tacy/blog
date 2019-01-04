---
title: "Byteman notes"
date: 2014-06-11
lastmod: 2014-06-11
draft: false
tags: ["tech", "java", "tools", "troubeshooting", "performance", "trace"]
categories: ["tech"]
description: "个人使用byteman的记录，byteman是一个非常方便的java分析工具，能够拦截字节码执行，执行代码和修改变量，是一个诊断问题的利器。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Byteman

byteman是jboss下的一个项目，是一个非常方便的java分析工具，能够拦截字节码执行，执
行代码和修改变量，是一个诊断问题的利器。

在linux下使用起来非常方便，不用对目标应用做任何修改，可以动态打开目标应用的监听
端口，当然仅限于openjdk，hotspot和jrockit，ibm jdk不支持，需要启动时配置监听[fn:1]。

具体的语法参考文档参见[[http://downloads.jboss.org/byteman/2.1.4/ProgrammersGuide.html][Byteman Programmer's Guide]]

## 使用
只需要配置BYTEMAN_HOME变量就可以了，正常启动你的应用，然后通过byteman提供的脚本
动态打开byteman agent监听，提交byteman脚本，执行你需要分析的业务功能即可。

## 诊断classloader争用问题
一般情况下，我们指定classloader不太容易导致争用，毕竟class只需要加载一次。但是有
些情况下可不向你想象的那样，看下面这个这个线程栈：
``` shell
"[ACTIVE] ExecuteThread: '1' for queue: 'weblogic.kernel.Default (self-tuning)'" daemon prio=10 tid=0x00007fbccc001000 nid=0x6dbc waiting for monitor entry [0x00007fbd11e20000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:270)
	at java.io.ObjectInputStream.resolveClass(ObjectInputStream.java:624)
	at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1611)
	at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1516)
	at java.io.ObjectInputStream.readClass(ObjectInputStream.java:1482)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1332)
	at java.io.ObjectInputStream.readArray(ObjectInputStream.java:1705)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1343)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:369)
	at java.util.HashMap.readObject(HashMap.java:1047)
	at sun.reflect.GeneratedMethodAccessor2.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:622)
	at java.io.ObjectStreamClass.invokeReadObject(ObjectStreamClass.java:1001)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1892)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1797)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1349)
	at java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:1989)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1914)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1797)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1349)
	at java.io.ObjectInputStream.readArray(ObjectInputStream.java:1705)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1343)
	at java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:1989)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1914)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1797)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1349)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:369)
	at com.primeton.access.client.impl.processor.CommonServiceProcessor.process(CommonServiceProcessor.java:49)
```
堵在classloader上了，正常情况下，我们当然想知道到底什么class需要这么频繁的加载。
以前你只能debug，加日志调试，现在我们来看看神奇的byteman怎么做到的。

我们知道加载class必须走java.lang.ClassLoader的loadClass，我们可以用byteman拦截
这个方法的执行，打印出调用栈和方法参数即可。

### 环境配置
我这里的测试环境是oracle-jdk6/ubuntu 12.04/weblogic10.3.6，byteman版本是2.1.4。
需要我们设置BYTEMAN_HOME环境变量，并把BYTEMAN_HOME/bin加入到PATH中。

### 动态打开byteman监听
正常启动需要profile的weblogic应用，之后通过jps获取进程号：
``` shell
root@tacy:/home/tacy/mytools/systemtap# jps -l
14677 org.apache.catalina.startup.Bootstrap
18224 weblogic.Server
28485 /home/tacy/mytools/java/eclipse//plugins/org.eclipse.equinox.launcher_1.2.0.v20110502.jar
18256 sun.tools.jps.
```

获取到weblogic的pid为18224，然后另开一终端，通过bminstall动态打开监听：
: root@tacy:~# bminstall.sh -b -Dorg.jboss.byteman.transform.all -Dorg.jboss.byteman.verbose -p 60000 18224
打开之后你能在weblogic的终端看到如下输出：
``` shell
Setting org.jboss.byteman.transform.all=
Setting org.jboss.byteman.verbose=
```
同时你能通过netstat查看到weblogic增加了一个监听端口在60000上。

注意这里我们设置了verbose，真正使用的时候你不需要，主要是前提脚本调试阶段需要看
看规则脚本是否执行了，确定脚本没问题就可以去掉。

另外就是org.jboss.byteman.transform.all这个参数，如果你要拦截java.lang类似的包，
必须设置，否则拦截不到。
``` shell
org.jboss.byteman.transform.all
is set then the agent will allow rules to be injected into methods of classes
in the java.lang hierarchy. Note that this will require the Byteman jar to be
installed in the bootstrap classpath using the boot: option to the -javaagent
JVM command line argument.
```

### 脚本
接下来我们先写一个如下的byteman脚本：
``` shell
RULE classloader profile
CLASS java.lang.ClassLoader
METHOD loadClass(java.lang.String, boolean)
IF callerCheck("com\.primeton.*", true, true,true,0,200)
DO traceln("load class: " + $1),traceStack("Thread Stack**********************:")
ENDRULE
```
具体语法我就不解释了，自己看文档，简单说就是拦截loadclass方法，如果发现调用栈里
面有匹配com.primeton.*字符串，就输出方法参数和调用栈。

脚本是否正确可以通过bmcheck来进行校验。

### 提交脚本
直接使用bmsubmit进行脚本提交：
``` shell
root@tacy:~# bmsubmit.sh -p 60000 -l test.bm
define rule classloader profile
```
之后回到weblogic终端窗口就能看到效果了

### 输出
执行你的业务功能，你可以看到如下输出：
``` shell
load class: long
Thread Stack**********************:java.lang.ClassLoader.loadClass(ClassLoader.java:-1)
java.lang.ClassLoader.loadClass(ClassLoader.java:295)
sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:301)
java.lang.ClassLoader.loadClass(ClassLoader.java:295)
java.lang.ClassLoader.loadClass(ClassLoader.java:295)
java.lang.ClassLoader.loadClass(ClassLoader.java:247)
weblogic.utils.classloaders.GenericClassLoader.loadClass(GenericClassLoader.java:179)
weblogic.utils.classloaders.FilteringClassLoader.findClass(FilteringClassLoader.java:101)
weblogic.utils.classloaders.FilteringClassLoader.loadClass(FilteringClassLoader.java:86)
java.lang.ClassLoader.loadClass(ClassLoader.java:295)
java.lang.ClassLoader.loadClass(ClassLoader.java:295)
java.lang.ClassLoader.loadClass(ClassLoader.java:247)
weblogic.utils.classloaders.GenericClassLoader.loadClass(GenericClassLoader.java:179)
weblogic.utils.classloaders.ChangeAwareClassLoader.loadClass(ChangeAwareClassLoader.java:52)
java.lang.Class.forName0(Class.java:-2)
java.lang.Class.forName(Class.java:249)
java.io.ObjectInputStream.resolveClass(ObjectInputStream.java:602)
java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1589)
java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1494)
java.io.ObjectInputStream.readClass(ObjectInputStream.java:1460)
java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1310)
java.io.ObjectInputStream.readArray(ObjectInputStream.java:1683)
java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1321)
java.io.ObjectInputStream.readObject(ObjectInputStream.java:349)
java.util.HashMap.readObject(HashMap.java:1030)
sun.reflect.GeneratedMethodAccessor2.invoke (Unknown Source)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
java.lang.reflect.Method.invoke(Method.java:597)
java.io.ObjectStreamClass.invokeReadObject(ObjectStreamClass.java:969)
java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1871)
java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1775)
java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1327)
java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:1969)
java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1893)
java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1775)
java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1327)
java.io.ObjectInputStream.readArray(ObjectInputStream.java:1683)
java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1321)
java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:1969)
java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1893)
java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1775)
java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1327)
java.io.ObjectInputStream.readObject(ObjectInputStream.java:349)
com.primeton.access.client.impl.processor.CommonServiceProcessor.process(CommonServiceProcessor.java:49)
...
```
问题一目了然了吧，人家在加载long.class，我们知道这是个java的原生类型，哪有啥
class啊，其实这个问题是ObjectInputStream里没有处理好，请看下面代码：
``` shell
    protected Class<?> resolveClass(ObjectStreamClass desc)
	throws IOException, ClassNotFoundException
    {
	String name = desc.getName();
	try {
	    return Class.forName(name, false, latestUserDefinedLoader());
	} catch (ClassNotFoundException ex) {
	    Class cl = (Class) primClasses.get(name);
	    if (cl != null) {
		return cl;
	    } else {
		throw ex;
	    }
	}
    }
```
这里他先去通过class.forName找，classloader也比较蠢，不分青红皂白，发现没有加载过
，就去加载，发现找不到就扔异常，然后resolvClass尝试看看是不是原生类型。解决这个
问题你可以自己写一个扩展类，把primClasses.get(name)放到前面即可。

这种问题在小并发情况下其实问题不大，但是当并发量一大的时候，都挂在classloader上
了。

## property
两个和性能相关的property：
``` shell
If system property
org.jboss.byteman.compileToBytecode
is set (with any value) then the rule execution engine will compile rules to bytecode before executing them. If this property is unset it will execute rules by interpreting the rule parse tree.
Transformations performed by the agent can be observed by setting several environment variables which cause the transformed bytecode to be dumped to disk.
```

``` shell
If system property
org.jboss.byteman.skip.overriding.rules
is set then the agent will not perform injection into overriding method implementations. If an overriding rule is installed the agent will print a warning to System.err and treat the rule as if it applies only to the class named in the CLASS clause. This setting is not actually provided to allow rules to be misused in this way. It is a performance tuning option. The agent has to check every class as it is loaded in order to see if there are rules which apply to it. It also has to check all loaded classes when rules are dynamically loaded via the agent listener. This requires traversing the superclass chain to locate overriding rules attached to superclasses. This increases the cost of running the agent (testig indicates that the cost goes from negligible (<< 1%) to, at worst, noticeable (~ 2%) but not to significant) So, if you do not intend to use overriding rules then setting this property helps to minimise the extent to which the agent perturbs the timing of application runs. This is particularly important when testing multi-threaded applications where timing is highly significant.
```

## 脚本编写TIPS
脚本编写的时候有些地方比较晦涩，下面一些例子

### callerEquals
: public boolean callerEquals(String name)
这是要你只要写方法名，不能带其他的东西，比如你只关心某个入口的方法调用效率：
: IF callerEquals("methodname")
你也可以可以带上类名，前提是需要用下面方法：
: public boolean callerEquals(String name,boolean includeClass)
写法如下：
: IF callerEquals("classname.methodname",true)
依此类推，你也可以带上报名，但是千万别这么写：
: IF callerEquals("classname.methodname")
这是匹配不上的，byteman会把"classname.methodname"当成方法名，当然就悲剧了～

## 脚本例子
### 方法执行效率统计（methodExectime）
``` shell
RULE entey readobject
CLASS com.primeton.access.client.impl.processor.CommonServiceProcessor
METHOD process
AT INVOKE readObject
IF true
DO resetTimer($0)
ENDRULE

RULE exit readobject
CLASS com.primeton.access.client.impl.processor.CommonServiceProcessor
METHOD process
AFTER INVOKE readObject
IF true
DO System.out.println(String.valueOf(getElapsedTimeFromTimer($0)))
ENDRULE
```
注意这里的输出用的是Systemoutprint，byteman内置的traceln有性能问题。

## 自定义RULE Helper
几个参考[fn:2][fn:3][fn:4]


# Footnotes

[fn:1] [[https://community.jboss.org/wiki/ABytemanTutorial][A Byteman Tutorial]]

[fn:2] [[http://www.mastertheboss.com/byteman/byteman-advanced-tutorial][Byteman advanced tutorial]]

[fn:3] [[http://rreddy.blogspot.jp/2013/08/byteman-oracle-jdbc-tracing.html][Byteman Oracle JDBC Tracing]]

[fn:4] [[http://blog.c2b2.co.uk/2012/07/using-custom-helpers-with-byteman.html][Using Custom Helpers with Byteman]]
