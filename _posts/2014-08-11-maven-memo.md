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