---
title: Maven可执行jar包
date: 2015-10-16 16:48:05
copyright:
tags: [Maven]
categories: 工程、工具
---
使用maven-compiler-plugin插件，打可执行的jar包，pom文件build部分。

```
<build>
    <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
            <source>1.8</source>
            <target>1.8</target>
            <encoding>utf8</encoding>
        </configuration>
    </plugin>

    <plugin>
        <artifactId>maven-assembly-plugin</artifactId>

        <!-- 对项目的组装进行配置 -->
        <configuration>

            <descriptorRefs>
                <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>

            <archive>
                <manifest><!--project 入口-->
                    <mainClass>com.zxm.gitpic.Gitpic</mainClass>
                </manifest>
            </archive>
        </configuration>
        <executions>
            <execution>
                <id>make-assembly</id>
                <!-- 将组装绑定到maven生命周期的哪一阶段 -->
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
