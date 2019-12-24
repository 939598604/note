# MapReduce的wordcount计算入门案例

## 1.创建maven工程

在pom.xml文件中添加如下依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.gzstrong</groupId>
    <artifactId>mapreduce</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>mapreduce</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.8.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.7.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.7.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.7.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-assembly-plugin </artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>com.gzstrong.mapreduce.wordcount.WordDriver</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

## 2.日志文件配置

在项目的src/main/resources目录下，新建一个文件，命名为“log4j.properties”，内容如下

```properties
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
log4j.appender.logfile=org.apache.log4j.FileAppender
log4j.appender.logfile.File=target/spring.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
```

## 3.编写Mapper类

WordcountMapper.java

```java
package com.gzstrong.mapreduce.wordcount;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

public class WordcountMapper extends Mapper<LongWritable,Text,Text,IntWritable> {
    Text k=new Text();
    IntWritable v=new IntWritable();

    @Override
    protected void map(LongWritable keyIn, Text valueIn, Context context) throws IOException, InterruptedException {
        String line = valueIn.toString();
        String[] words = line.split(" ");
        for (String word : words) {
            k.set(word);
            v.set(1);
            context.write(k,v);
        }
    }
}
```

## 4.编写Reducer类

WordcountReduce.java

```java
package com.gzstrong.mapreduce.wordcount;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class WordcountReduce extends Reducer<Text,IntWritable,Text,IntWritable> {
    IntWritable v=new IntWritable();
    int sum=0;

    @Override
    protected void reduce(Text keyIn, Iterable<IntWritable> valuesIn, Context context) throws IOException, InterruptedException {
        for (IntWritable intWritable : valuesIn) {
            sum = sum+ intWritable.get();
        }
        v.set(sum);
        context.write(keyIn, v);
    }
}
```

## 5.编写Driver驱动类

WordDriver.java

```java
package com.gzstrong.mapreduce.wordcount;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
public class WordDriver {
    public static void main(String[] args) throws Exception {
        args=new String[]{"D:\\input\\1.txt","D:\\output"+System.currentTimeMillis()};

        // 1 获取配置信息以及封装任务
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);

        // 2 设置jar加载路径
        job.setJarByClass(WordDriver.class);

        //3.关联map和reduce
        job.setMapperClass(WordcountMapper.class);
        job.setReducerClass(WordcountReduce.class);

        // 4 设置map输出
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        // 5 设置最终输出kv类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // 6 设置输入和输出路径
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 7 提交
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

## 6.准备数据文件

在D:\\input\\1.txt的内容如下

```txt
User Controls Download
在 JiJiDownForWPF UserControls Download UserControl Size Changed
在 System Windows RoutedEventArgs InvokeHandler
在 System Windows RoutedEventHandlerInfo InvokeHandler
在 System indows EventRoute InvokeHandlersImpl
在 System Windows UIElement RoutedEventArgsargs
在 System Windows FrameworkElement 
在 System Windows ContextLayoutManager fireSizeChangedEvents
在 System Windows ContextLayoutManager UpdateLayout
在 System Windows ContextLayoutManager UpdateLayout Callback Objectarg
在 System Windows Media MediaContext FireInvokeOnRender Callback
在 System Windows Media MediaContext RenderMessage Handler
在 System Windows Media MediaContext AnimatedRenderMessage
在 System Windows Threading ExceptionWrapper InternalReal
在 System Windows Threading ExceptionWrapper TryCatchWhen 
```

## 7.本地测试

如果电脑系统是win7的就将win7的hadoop jar包解压到非中文路径，并在Windows环境上配置HADOOP_HOME环境变量。如果是电脑win10操作系统，就解压win10的hadoop jar包，并配置HADOOP_HOME环境变量。

注意：win8电脑和win10家庭版操作系统可能有问题，需要重新编译源码或者更改操作系统。

## 8.集群上测试

用maven打jar包，需要添加的打包插件依赖

注意：mainClass中的部分需要替换为自己工程主类

```
<build>
		<plugins>
			<plugin>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>2.3.2</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<artifactId>maven-assembly-plugin </artifactId>
				<configuration>
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>
					<archive>
						<manifest>
							<mainClass>
							    com.gzstrong.mapreduce.wordcount.WordDriver
							</mainClass>
						</manifest>
					</archive>
				</configuration>
				<executions>
					<execution>
						<id>make-assembly</id>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
```

- 将程序打成jar包，然后拷贝到Hadoop集群中

步骤详情：

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/QQ截图20191224111335.png)

等待编译完成就会在项目的target文件夹中生成jar包。修改不带依赖的jar包名称为wc.jar，并拷贝该jar包到Hadoop集群。

- 启动Hadoop集群
- 执行WordCount程序

```shell
hadoop jar wc.jar com.gzstrong.mapreduce.wordcount.WordDriver /input /output
```

