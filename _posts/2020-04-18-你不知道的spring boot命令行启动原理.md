---
layout:     post
title:      你不知道的spring boot命令行启动原理
subtitle:   学习《spring boot编程思想》笔记
date:       2020-04-18
author:     Kent
header-img: img/post-bg-spring-boot-java-jar.jpg
catalog: true
tags:
    - spring boot
    - 原理
---

# 你不知道的spring boot命令行启动原理

## 打包原理

1. 在pom.xml中配置<packaging>标签。
    ```
    <packaging>jar</packaging>
    ```
   
   ![](https://mmbiz.qpic.cn/mmbiz_jpg/YTej5CfI92OnHg3lMnxgOzporMHLM8JMXWyicFMdsqQsVQZjqPAK2KbRc6EO9tsz2TnuNnsCZ1AWMmESZoodMUA/0?wx_fmt=jpeg)

2. 添加spring-boot打包的maven插件。
    ```
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>2.0.2.RELEASE</version>
    </plugin>
    ```
3. 执行maven命令将spring boot应用打包
    ```
    mvn package
    ```
   观察日志如下图：
   
   ![](https://mmbiz.qpic.cn/mmbiz_jpg/YTej5CfI92OnHg3lMnxgOzporMHLM8JMcVfL0Q2MGjagTg7lMh1ZvGPS4kFAtV7IDtZzmKkia5oB7qohhQUeqzg/0?wx_fmt=jpeg)
   
   spring-boot-maven-plugin插件执行repackage goal命令生成了spring boot 可执行jar包，也叫FAT JAR。

## 包目录结构
    
为了一探究竟，我们将这个FAT JAR解压，了解其目录结构。
    
```
unzip xxx.jar -d temp
```

+ BOOT-INF/classes目录存放应用编译后的class文件

+ BOOT-INF/lib目录存放应用依赖的JAR包

+ META-INF/目录存放应用相关的元信息，如MANIFEST.MF文件

+ org/目录存放Spring Boot相关的class文件

## MAINIFEST.MF文件含义

查看MANIFEST.MF文件中的内容

```
$ cat MANIFEST.MF
Manifest-Version: 1.0
Implementation-Title: first-app-by-gui
Implementation-Version: 0.0.1-SNAPSHOT
Built-By: dengcechao
Implementation-Vendor-Id: thinking-in-spring-boot
Spring-Boot-Version: 2.0.2.RELEASE
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: thinkinginspringboot.firstappbygui.FirstAppByGuiApplicati on
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Created-By: Apache Maven 3.6.2
Build-Jdk: 1.8.0_221
Implementation-URL: https://projects.spring.io/spring-boot/#/spring-boot-starter-parent/first-app-by-gui

```

这里我们主要注意两个属性，Main-Class和Start-Class。

显然，Start-Class正是我们Spring boot应用里面的启动类。

根据Java的规定，java -jar命令引导的具体启动类就是Main-Class，
而这里Start-Class才是我们真正要启动的类，大胆猜一下，是Main-Class启动了Start-Class。

## 启动类源码解读

我们已经知道java -jar执行的类是Main-Class: org.springframework.boot.loader.JarLauncher,
我们引入这个类的依赖。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-loader</artifactId>
    <scope>provided</scope>
</dependency>
```

找到org.springframework.boot.loader.JarLauncher类，
当执行java -jar命令时，会调用org.springframework.boot.loader.JarLauncher#main()。

```java
public class JarLauncher extends ExecutableArchiveLauncher {

	static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

	static final String BOOT_INF_LIB = "BOOT-INF/lib/";

	public JarLauncher() {
	}

	protected JarLauncher(Archive archive) {
		super(archive);
	}

	@Override
	protected boolean isNestedArchive(Archive.Entry entry) {
		if (entry.isDirectory()) {
			return entry.getName().equals(BOOT_INF_CLASSES);
		}
		return entry.getName().startsWith(BOOT_INF_LIB);
	}

	public static void main(String[] args) throws Exception {
		new JarLauncher().launch(args);
	}

}
```

跟踪代码进入到org.springframework.boot.loader.Launcher#launch(java.lang.String[])

```java
public abstract class Launcher {
	protected void launch(String[] args) throws Exception {
		JarFile.registerUrlProtocolHandler();
		ClassLoader classLoader = createClassLoader(getClassPathArchives());
		launch(args, getMainClass(), classLoader);
	}
}
```

值得注意的是createClassLoader中的参数，参数调用的是org.springframework.boot.loader.ExecutableArchiveLauncher#getClassPathArchives()，
根据方法名我们可以判断是获取依赖有关的东西。

```java
public abstract class ExecutableArchiveLauncher extends Launcher {
	@Override
	protected List<Archive> getClassPathArchives() throws Exception {
		List<Archive> archives = new ArrayList<>(
				this.archive.getNestedArchives(this::isNestedArchive));
		postProcessClassPathArchives(archives);
		return archives;
	}
    
	protected abstract boolean isNestedArchive(Archive.Entry entry);

    protected void postProcessClassPathArchives(List<Archive> archives) throws Exception {
	}
}
```

这里调用了两个方法org.springframework.boot.loader.ExecutableArchiveLauncher#isNestedArchive()和
org.springframework.boot.loader.ExecutableArchiveLauncher#postProcessClassPathArchives()，
其中后面一个方法是一个空方法，我们可以忽略。

前面一个方法是一个抽象方法，可以看到他有两个实现类中实现了该方法，分别是
org.springframework.boot.loader.JarLauncher
和org.springframework.boot.loader.WarLauncher，
我们先看JarLauncher。

![](https://mmbiz.qpic.cn/mmbiz_jpg/YTej5CfI92OnHg3lMnxgOzporMHLM8JMe34UuILPpQb7ibGOBHEbEaoCgdG8upz6NDVCJmXOYcibrUkbbQbmNcKA/0?wx_fmt=jpeg)

```java
public class JarLauncher extends ExecutableArchiveLauncher {

	static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

	static final String BOOT_INF_LIB = "BOOT-INF/lib/";

	@Override
	protected boolean isNestedArchive(Archive.Entry entry) {
		if (entry.isDirectory()) {
			return entry.getName().equals(BOOT_INF_CLASSES);
		}
		return entry.getName().startsWith(BOOT_INF_LIB);
	}

}
```
这个类就是我们最先看的那个类，我把无关方法都删掉了，只看我们关注的方法。
很明显，这个方法，就是根据路径来判断是否需要加载，符合路径规范的内容就返回true，否则返回false。

回到org.springframework.boot.loader.Launcher#launch(java.lang.String[])

```java
public abstract class Launcher {

	protected void launch(String[] args) throws Exception {
		JarFile.registerUrlProtocolHandler();
		ClassLoader classLoader = createClassLoader(getClassPathArchives());
		launch(args, getMainClass(), classLoader);
	}

    protected void launch(String[] args, String mainClass, ClassLoader classLoader)
			throws Exception {
		Thread.currentThread().setContextClassLoader(classLoader);
		createMainMethodRunner(mainClass, args, classLoader).run();
	}

    protected MainMethodRunner createMainMethodRunner(String mainClass, String[] args,
			ClassLoader classLoader) {
		return new MainMethodRunner(mainClass, args);
	}

}
```

该方法里面调用了org.springframework.boot.loader.ExecutableArchiveLauncher#getMainClass()
来获取mainClass做为参数。

```java
public abstract class ExecutableArchiveLauncher extends Launcher {
	@Override
	protected String getMainClass() throws Exception {
		Manifest manifest = this.archive.getManifest();
		String mainClass = null;
		if (manifest != null) {
			mainClass = manifest.getMainAttributes().getValue("Start-Class");
		}
		if (mainClass == null) {
			throw new IllegalStateException(
					"No 'Start-Class' manifest entry specified in " + this);
		}
		return mainClass;
	}
}
```

很明显，这里就是获取MANIFEST.MF中的Start-Class做为mainClass参数传给
org.springframework.boot.loader.Launcher#createMainMethodRunner(String mainClass, String[] args,ClassLoader classLoader)
来构造一个org.springframework.boot.loader.MainMethodRunner对象，
并调用其org.springframework.boot.loader.MainMethodRunner#run()
来执行java -jar的核心逻辑，也就是启动Start-Class。

```java
public class MainMethodRunner {

	private final String mainClassName;

	private final String[] args;

	public MainMethodRunner(String mainClass, String[] args) {
		this.mainClassName = mainClass;
		this.args = (args != null ? args.clone() : null);
	}

	public void run() throws Exception {
		Class<?> mainClass = Thread.currentThread().getContextClassLoader()
				.loadClass(this.mainClassName);
		Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
		mainMethod.invoke(null, new Object[] { this.args });
	}

}
```

由此可见，我们之前的猜想"是Main-Class启动了Start-Class"是正确的。

## 你可能不知道的

前面说到org.springframework.boot.loader.ExecutableArchiveLauncher抽象类有两个实现类，
org.springframework.boot.loader.JarLauncher
和org.springframework.boot.loader.WarLauncher，
JarLauncher我们已经分析过了，并且我们知道，
在java -jar整个启动过程中，只有
org.springframework.boot.loader.ExecutableArchiveLauncher#isNestedArchive()
的实现是不一样的，而这个方法的作用是根据路径判断是否是需要加载的依赖，
至此，我们同样可以大胆猜测，java -jar命令也能对war执行，只不过war包里面的文件路径与FAT JAR不一样罢了。

我们将pom.xml中的<packaging>改一下。

```
<packaging>war</packaging>
```

重新执行打包命令。

```
mvn clean package
```

打包完毕后，执行java -jar命令，启动应用。

```
java -jar xxx.war
```

如图，启动成功。

![](https://mmbiz.qpic.cn/mmbiz_jpg/YTej5CfI92OnHg3lMnxgOzporMHLM8JM07iaM8dSK3mU7guiaKl21OQ55Vjh8ia8TlEO5YbUVz4UBzC637onLgRMQ/0?wx_fmt=jpeg)

同样方法，解压war，并查看目录。

```
unzip xxx.war -d temp
```

解压后查看目录。

```
tree temp
```

图片过长，就不截图了，主要看两个目录，META-INF和WEB-INF，这是传统的servlet应用的路径。
META-INF与FAT JAR的目录结构一样，值得注意的是WEB-INF相比FAT JAR中的BOOT-INF多了lib-provided目录，
该目录存放的是<scope>provided</scope>的JAR文件，
根据org.springframework.boot.loader.ExecutableArchiveLauncher#isNestedArchive()
的WarLauncher实现可以知道，这个目录下的文件在java -jar命令启动的时候，是会被加载的。

```java
public class WarLauncher extends ExecutableArchiveLauncher {

	private static final String WEB_INF = "WEB-INF/";

	private static final String WEB_INF_CLASSES = WEB_INF + "classes/";

	private static final String WEB_INF_LIB = WEB_INF + "lib/";

	private static final String WEB_INF_LIB_PROVIDED = WEB_INF + "lib-provided/";

	@Override
	public boolean isNestedArchive(Archive.Entry entry) {
		if (entry.isDirectory()) {
			return entry.getName().equals(WEB_INF_CLASSES);
		}
		else {
			return entry.getName().startsWith(WEB_INF_LIB)
					|| entry.getName().startsWith(WEB_INF_LIB_PROVIDED);
		}
	}

}
```

但是如果使用传统的Web部署时，这个目录下的jar是会被忽略的，
所以这里可能是一种兼容的处理方式，使war包既支持java -jar命令启动，又支持传统的Web部署方式。
