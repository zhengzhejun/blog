---
layout: post
title: Hadoop源码系列-1-example程序的运行.md
category: Hadoop
tag: Hadoop源码
---
这里记录一下运行example程序的过程中遇到的问题，和解决方案。

首先，在[上一篇博客](http://zhengzhejun.com/hadoop%EF%BC%8Chadoop%E6%BA%90%E7%A0%81/2016/06/14/Hadoop%E6%BA%90%E7%A0%81%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%92%8C%E8%B0%83%E8%AF%95.html)的最后，我们在Intellij内直接run了example的main函数。这里，我想把这个example程序打包出来，在命令行中运行。这里，我们先用mvn -U clean package -DskipTests，对example项目打包。然后在target文件下，找到hadoop-mapreduce-examples-2.6.4.jar。接下来，我们运行该jar包。
{% highlight vim %}
➜  target java -jar ./hadoop-mapreduce-examples-2.6.4.jar WordCount
Picked up JAVA_TOOL_OPTIONS: -Djava.security.krb5.conf=/etc/krb5.conf
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/util/ProgramDriver
	at org.apache.hadoop.examples.ExampleDriver.main(ExampleDriver.java:37)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.util.ProgramDriver
	at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
	... 1 more
{% endhighlight %}
这里有两个问题：
* 缺少org/apache/hadoop/util/ProgramDriver。包没引进来？
* 为啥执行的是examples.ExampleDriver？

我看了一下hadoop-mapreduce-examples-2.6.4.jar，里面确实没有引进来hadoop相关的包，这里我们需要在pom中引入一个plugin，如下：
{% highlight xml %}
	<plugin>
		<groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
		   </execution>
		</executions>
	</plugin>
{% endhighlight %}
通过这个插件，可以把该项目依赖的所有dependencies放入最终打包的jar中。
重新运行mvn -U clean package -DskipTest，我们能在hadoop-mapreduce-examples-2.6.4.jar中发现hadoop相关依赖包。

另外，我们在pom文件中发现如下插件，用于指定mainClass
{% highlight xml %}
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>org.apache.hadoop.examples.ExampleDriver</mainClass>
                        </manifest>
                    </archive>
                </configuration>
</plugin>
{% endhighlight %}
通过这个配置，我们在hadoop集群中运行example程序时，使用命令hadoop jar ***.jar wordcount，这个命令就会调用example.jar的mainClass ExampleDriver，在ExampleDriver中，运行wordcount。详细请见[上一篇博客](http://zhengzhejun.com/hadoop%EF%BC%8Chadoop%E6%BA%90%E7%A0%81/2016/06/14/Hadoop%E6%BA%90%E7%A0%81%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%92%8C%E8%B0%83%E8%AF%95.html)

现在我们打包的jar中包含了hadoop相关的依赖，所以直接用下面的命令运行wordcount程序。
{% highlight vim %}
java -cp ./hadoop-mapreduce-examples-2.6.4.jar org.apache.hadoop.examples.WordCount
{% endhighlight %}
这里有一个小提示。

> jar -cp lib/referenced.jar -jar myworks.jar 如果我们使用-jar选项的话java会忽略-cp,-classpath，以及环境变量CLASSPATH的参数, 具体详见[这里](http://luzl.iteye.com/blog/684435)

下面我们运行jar包，命令如下：
{% highlight java %}

java -cp hadoop-mapreduce-examples-2.6.4.jar org.apache.hadoop.examples.WordCount input output
//报如下错误
log4j:WARN No appenders could be found for logger (org.apache.hadoop.util.Shell).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
Exception in thread "main" java.io.IOException: Cannot initialize Cluster. Please check your configuration for mapreduce.framework.name and the correspond server addresses.
	at org.apache.hadoop.mapreduce.Cluster.initialize(Cluster.java:120)
	at org.apache.hadoop.mapreduce.Cluster.<init>(Cluster.java:82)
	at org.apache.hadoop.mapreduce.Cluster.<init>(Cluster.java:75)
	at org.apache.hadoop.mapreduce.Job$9.run(Job.java:1267)
	at org.apache.hadoop.mapreduce.Job$9.run(Job.java:1263)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:415)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1656)
	at org.apache.hadoop.mapreduce.Job.connect(Job.java:1262)
	at org.apache.hadoop.mapreduce.Job.submit(Job.java:1291)
	at org.apache.hadoop.mapreduce.Job.waitForCompletion(Job.java:1315)
	at org.apache.hadoop.examples.WordCount.main(WordCount.java:89)

{% endhighlight %}

针对这个问题，我想把jar包解压看一下，里面有什么hadoop的配置文件。但是用jar xf hadoop-mapreduce-examples-2.6.4.jar解压时，遇到如下错误
{% highlight java %}
java.io.IOException: META-INF/license: 无法创建目录
	at sun.tools.jar.Main.extractFile(Main.java:988)
	at sun.tools.jar.Main.extract(Main.java:924)
	at sun.tools.jar.Main.run(Main.java:264)
	at sun.tools.jar.Main.main(Main.java:1231)
{% endhighlight %}
google了一下，这是macbook特有的问题，[详见这里](http://mapserver000-gmail-com.iteye.com/blog/1462056)
根据这篇博客，在pom中的maven-shade-plugin中添加如下配置
{% highlight xml %}
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                        <exclude>META-INF/license/*</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
{% endhighlight %}

之后可以解压jar包，看里面的hadoop配置文件。





























