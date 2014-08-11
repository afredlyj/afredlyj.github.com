---  
layout: post  
title: 用Jmeter 测试netty服务器接口     
category: program
 
---

Netty服务器接口采用google protobuf序列化，同时需要模拟多个不同的请求参数，所以自己实现了Jmeter的`AbstractJavaSamplerClient`，并且用`CSV Data Set Config`配置原件读取不同的参数值。以下将在windows环境下编写自己的Java Sampler记录下来，以供参考。


### 实现自己的AbstractJavaSamplerClient  

AbstractJavaSamplerClient是Jave Sample的一个抽象实现，如果要实现自己的Java Sample，有几个方法是需要自己重写的：  

 * runTest 方法  
方法签名如下：  

~~~~  
SampleResult runTest(JavaSamplerContext context)
~~~~  

AbstractJavaSamplerClient并没有实现这个方法，需要子类自己实现，这个方法用来执行一次测试用例，在测试代码块的前后，需要调用`SampleResult.sampleStart()`和`SampleResult.sampleEnd()`方法，同时，根据测试代码块的返回，设置SampleResult的状态，最后将该SampleResult返回，如果使用带GUI的JMeter，可以在`查看结果树`中看到相应的结果返回。我这里需要模拟客户端请求，测试代码如下：

~~~~  
   @Override
    public SampleResult runTest(JavaSamplerContext javaSamplerContext) {

        String url = javaSamplerContext.getParameter("url");
        String type = javaSamplerContext.getParameter("type");
        String cardId = javaSamplerContext.getParameter("cardId");
        String cardPwd = javaSamplerContext.getParameter("cardPwd");
        String barCode = javaSamplerContext.getParameter("barCode");
        accessToken = javaSamplerContext.getParameter("accessToken");

        SampleResult result = new SampleResult();
        VcCardRechargeBean.VcCardRechargeRes res = null;
        try {

            byte[] bs = new byte[0];
            try {
                bs = buildRequestData(type, barCode, cardId, cardPwd);
            } catch (Exception e) {
                e.printStackTrace();
                result.setSuccessful(false);
                result.setResponseMessage("Exception: " + e);
            }

            result.sampleStart();
            bs = HttpConnectUtil.getResponseByPost(url, null, bs);
            result.sampleEnd();
            if (bs == null) {
                result.setSuccessful(false);
            } else {
                res = VcCardRechargeBean.VcCardRechargeRes.parseFrom(bs);
              /*  if ("1001".equals(res.getResultCode())) {
                    result.setSuccessful(true);
                } else {
                    result.setSuccessful(false);
                }*/
                result.setResponseCodeOK();
                result.setResponseMessageOK();
                result.setSuccessful(true);
                result.setResponseData(res.getResultCode(), "UTF-8");
            }

        } catch (Exception e) {
            e.printStackTrace();
            result.sampleEnd();
            result.setSuccessful(false);
            result.setResponseMessage("Exception: " + e);
        }

        return result;
    }  
~~~~  

可以通过`JavaSamplerContext.getParameter()`方法获取执行一次测试用例的请求参数，参数值的具体设置方法稍后提到。需要注意的是，我这个测试代码依赖`httpclient`，所以生成测试jar包后，需要将所有的依赖包copy到jmeter的lib目录下。

 * getDefaultParameters方法  
 方法签名如下：  
 
~~~~  
public Arguments getDefaultParameters()  
~~~~  
 
 该方法会返回一个参数列表，最后会在GUI界面现实，方便自定义参数值，如果该方法返回null，将不会现实参数列表，所以，实现自己的getDefaultParamters()方法是必要的，在GUI界面现实的参数列表，最终在`runTest`方法中会用到，即上文的`JavaSamplerContext.getParameter()`获取的一系列参数。所以我的实现如下：    

~~~~  
@Override  
    public Arguments getDefaultParameters() {
    
        Arguments arguments = new Arguments();
        arguments.addArgument("url", "");
        arguments.addArgument("type", "");
        arguments.addArgument("cardId", "");
        arguments.addArgument("cardPwd", "");
        arguments.addArgument("barCode", "");
        arguments.addArgument("accessToken", "");
        return arguments;
    }  
~~~~  

为了方便，我都返回了空串，也可以设定默认值后返回。

其他方法比如`setupTest`和`teardownTest` 用法估计和JUnit中的方法差不多，就没有尝试了。

### 结合CSV Data为每次请求设置不同的请求参数

[用JMeter进行压力测试](http://afredlyj.github.io/posts/jmeter-intro.html) 一文中介绍过`CSV Data Set Config`的用法，准备好参数文件(test.txt)后，将文件放到bin/test目录下，本次压测可变参数是cardId和cardPwd，所以只需要在这两个参数的value栏中设置`${key}`, `${value}`即可。

### copay jar包到相应目录

 * 实现自己的AbstractJavaSamplerClinet后，打包成jar，并将该jar包copy到lib/ext目录下；
 * 将测试用例的依赖包，比如httpclient相关的包copy到lib目录下；
 
准备工作已经完成，剩下的就是在GUI界面下，添加线程组和Java Sample，选择自己的test case，设置请求参数，即可跑压测了。
 
 
