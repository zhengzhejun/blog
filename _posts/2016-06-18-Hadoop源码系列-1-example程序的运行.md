---
layout: post
title: Hadoop源码系列-1-example程序的运行
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

## 在hadoop源码中添加自己的Log。
首先，要在example项目中中添加log4j的配置文件。在src/main/目录下，创建resouces，在Intellij下，将该resources文件夹设置为Mark directory as Resources Root，这样该文件下的配置文件将会被打包到jar包中的根目录中，进而被log4j读取。
这里，我的log配置文件为：
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">
<log4j:configuration>

    <appender name="myConsole" class="org.apache.log4j.ConsoleAppender">
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="[%d{dd HH:mm:ss,SSS\} %-5p] [%t] %c{2\} - %m%n" />
        </layout>
    </appender>


    <appender name="myFile" class="org.apache.log4j.RollingFileAppender">
        <param name="File" value="/Users/zhengzhejun/software/hadoop-learn/logs/examples.log" /><!-- 设置日志输出文件名 -->
        <!-- 设置是否在重新启动服务时，在原有日志的基础添加新日志 -->
        <param name="Append" value="true" />
        <param name="MaxBackupIndex" value="10" />
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%p (%c:%L)- %m%n" />
        </layout>
    </appender>



    <!-- 指定logger的设置，additivity指示是否遵循缺省的继承机制-->
    <logger name="org.apache.hadoop.examples" additivity="false">
        <level value ="debug"/>
        <appender-ref ref="myConsole" />
        <appender-ref ref="myFile" />
    </logger>

    <logger name="org.apache.hadoop" additivity="false">
        <level value="debug" />
        <appender-ref ref="myConsole" />
        <appender-ref ref="myFile" />
    </logger>

    <!-- 根logger的设置-->
    <root>
        <level value ="debug"/>
        <appender-ref ref="myConsole"/>
        <appender-ref ref="myFile"/>
    </root>

</log4j:configuration>
{% endhighlight %}
这里简单总结一下log4j的配置文件用法，每次用的时候都忘，都需要临时查一下用法。。。最简答的配置文件里面有三个元素，appender，logger，root。这里appender设置日志类型，比如往console打日志，或者往File打日志。接下来是logger，logger用来指定每个包下用什么appender。root用来指定那些没有指定logger的包下的日志数据应该用什么appender。

添加完配置文件之后，我想看一下Configuration类里面的运行情况，所以在Configuration里面打了几个日志，然后到example项目下，重新mvn -U clean package -DskipTests，获取hadoop-mapreduce-examples-2.6.4.jar。然后运行java -cp ./hadoop-mapreduce-example-2.6.4.jar org.apache.hadoop.examples.WordCount，发现我刚才加在Configuration的日志没有打出来。

我查看了mvn -U clean package -DskipTests的输出，发现有一行为[INFO] Including org.apache.hadoop:hadoop-common:jar:2.6.4 in the shaded jar
所以我推测，在example项目中打包的话，会从maven本地库中得到org.apache.hadoop:hadoop-common:jar，所以我到hadoop-common-project下的hadoop-common项目中，运行mvn -U clean package install，将更改后的Configuration打包到本地的maven库中。最后回到example项目中，重新打包，并运行jar包，发现添加在Configuration中的日志成功打出来了。
























