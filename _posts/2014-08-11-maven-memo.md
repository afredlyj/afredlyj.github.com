---
layout: post  
title: maven备忘 
category: funny  
---

#### 执行单元测试

 * 忽略测试用例结果
 
 Maven在构建过程中，如果一个测试用例失败，默认处理是终止整个构建过程，如果想要更改默认操作，可以通过Surefire插件修改，通过设置`testFailureIngore`属性为`true`，Maven将忽略单元测试的测试结果，继续整个构建。
 
~~~~  
 // todo : fix plugin configure  
~~~~
 
 还可以通过命令行控制:  
 
~~~~  
 mvn test -Dmaven.test.failure.ignore=true  
~~~~
 
 * 跳过单元测试
 
 Maven提供了完全跳过单元测试的方法，仍然是用Surefire插件：
 
~~~~  
 // todo : fix plugin configure  
~~~~
 
 
 依然可以通过命令行控制，只需要给目标加上`maven.test.skip`属性即可:  
 
~~~~  
 mvn install -Dmaven.test.skip=true  
~~~~
 
#### 依赖范围

 * complie
 * provided
 * runtime
 * test
 * system  
 必须提供systemPath

#### 利用maven打包jar项目

之前只会用shade插件打包成jar，这样打出的包，所有的依赖都在一个jar包中，动辄几十M，上传到服务器非常不方便，如果能把项目class和依赖包分开，每次都只需要上传项目class的jar包和更新的依赖包，服务器更新会快不少，这就需要`maven-jar-plugin`和`maven-dependency-plugin`两个插件支持了。pom配置如下，并附带注释：

~~~~  
    <build>  
       <finalName>${project.artifactId}-${maven.build.timestamp}</finalName>  
       <plugins>  
           <plugin>
               <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-jar-plugin</artifactId>
               <configuration>
                   <archive>
                       <manifest>
                           <addClasspath>true</addClasspath><!-- 加载class -->
                           <classpathPrefix>lib/</classpathPrefix><!-- 加载的class目录的前缀（依赖的jar目录） -->
                           <mainClass>com.keke.crm.App</mainClass><!-- 入口类名 -->
                       </manifest>
                   </archive>
               </configuration>
           </plugin>

           <plugin>
               <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-dependency-plugin</artifactId>
               <executions>
                   <execution>
                       <id>copy</id>
                       <phase>package</phase>
                       <goals>
                           <goal>copy-dependencies</goal>
                       </goals>
                       <configuration>
                           <outputDirectory>${project.build.directory}/lib</outputDirectory><!-- 拷贝所以依赖存放位置 -->
                       </configuration>
                   </execution>
               </executions>
           </plugin>
       </plugins>
    </build>

~~~~  

`maven-jar-plugin`指定了入口类和依赖jar包的路径，`maven-dependency-plugin` 将项目的依赖包拷贝到`${project.build.directory}/lib`目录，如果需要更新jar包，只需要将该目录的指定jar包上传到服务器即可，而最后的项目jar包包名为`>${project.artifactId}-${maven.build.timestamp}`，在target目录下，可以发现，这时候的jar包只有几十k的大小了。

#### parent节点的relativePath

>The relative path of the parent pom.xml file within the check out. If not specified, it defaults to ../pom.xml. Maven looks for the parent POM first in this location on the filesystem, then the local repository, and lastly in the remote repo. relativePath allows you to select a different location, for example when your structure is flat, or deeper without an intermediate parent POM. However, the group ID, artifact ID and version are still required, and must match the file in the location given or it will revert to the repository for the POM. This feature is only for enhancing the development in a local checkout of that project. Set the value to an empty string in case you want to disable the feature and always resolve the parent POM from the repositories. 
Default value is: ../pom.xml.



