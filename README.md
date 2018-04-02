# EasyBridge

[EasyBridge](https://github.com/easilycoder/EasyBridge)是一个简单易用的js-bridge的工具库，提供了日常开发中，JavaScript与Java之间通讯的能力，与其他常见的js-bridge工具库实现方案不同，**EasyBridge**具备以下几个特点：

* 基于Android `WebView`的`addJavascriptInterface`特性实现
* 提供了基于接口粒度的安全管理接口
* 轻量级，并且简单易用。以这个工具库作为依赖，只需要编写实际通讯接口

## 实现原理说明

混合开发一直是工业界移动端开发比较看好的技术手段，结合h5的特性，能够更好的支持业务发展的需要，不仅快速上线、部署功能而且能够快速响应线上的bug。目前混合开发的方案包括：

* JSBridge
* [Cordova](https://cordova.apache.org)
* [React Native](https://facebook.github.io/react-native/)
* [Flutter](https://flutter.io)

**EasyBridge**就是一种简单的JSBridge解决方案。在众多的解决方案中，都是在利用系统的`WebView`所开放的权限和接口，打开Java与JavaScript通讯的渠道，这些方案的实现原理分别包括：

* 拦截`onJsPrompt()`方法

  当`WebView`中的页面调用了JavaScript当中的`window.prompt()`方法的时候，这个方法会被回调。而且这个方法不仅能获取到JavaScript传递过来的string字符串内容，同时也能返回一段string字符串内容被JavaScript接收到，是一个相当适合构建bridge的入口方法。

* 拦截`shouldOverrideUrlLoading()`方法

  当页面重新load URL或者页面的iframe元素重新加载新的URL的时候，这个方法被回调。

* `addJavascriptInterface()`接口

  这个接口简单却强大，通过这个接口，我们能够直接把Java中定义的对象在JavaScript中映射出一个对应的对象，使其直接调用Java当中的方法，但是，在android 4.1及之前的版本存在着严重的漏洞，所以一直被忽视。

**EasyBridge**在众多的解决方案中，最终了选择了`addJavascriptInterface()`接口作为方案的基础，主要基于以下几点考量：

* 目前Android版本已经到了9.0版本，市面上Android4.4之前的版本手机占有率已经很低，很多业务都已经把最低兼容版本定在了4.2以上，因此不需要考量4.1以下存在的漏洞问题；
* `addJavascriptInterface()`能够提供最简单的同步调用
* `addJavascriptInterface()`与`evaluateJavascript()`/`loadUrl`结合，能够带来更加简单的异步调用的解决方案

## 方案设计说明

**EasyBridge**最终方案实现，只支持了异步调用的方式，主要是基于以下两点的考量：

* 同步的调用可以转化为异步调用的方式，保留一种调用方式会使得整个方案更加简单；
* 目前iOS不支持同步的调用。

### 方案结构

**EasyBridge**的方案结构如下图所示：

![](/src/easybridge-architecture.png)

**EasyBridge**总共会向页面中注入两个JavaScript对象，：

* **easyBridge**

  在页面加载完成`onPageFinished()`回调的时候，通过执行工具库中的一个js文件注入的。这个对象主要的作用是定义了业务页面的JavaScript代码调用native的Java代码的规范入口，对象中定义的一个最关键的函数就是`callHandler(handlerName, args, callback)`，这就是桥梁的入口。实际上在这个方法的内部，最终就是通过下面的**_easybridge**对象进入到Java代码层。

* _easybridge

  通过`addJavascriptInterface()`映射和注入的一个对象，这个对象提供了实质的入口方法`enqueue()`，在这个方法当中代码的路线从JavaScript层进入到了Java层，开启了两者的交互。

### 接口分发

实际上，我们可以通过`@JavascriptInterface`注解开放很多的接口给JavaScript层调用，也可以通过`addJavascriptInterface()`映射多个Java对象到JavaScript层，但是为了维护简单和通讯方便，**EasyBridge**的设计只提供了一个入口和一个出口。所有需要开放给JavaScript层的功能，都是通过构建接口实例进行处理。

接口的定义如下：

```java
public interface BridgeHandler {

    String getHandlerName();

    void onCall(String parameters, ResultCallBack callBack);

    SecurityPolicyChecker securityPolicyChecker();
}
```

实际的工作流程如下图所示：

![](/src/handler-execute.png)

最开始初始化的时候需要注册所有可以被JavaScript层调用的业务接口。在运行的过程中，`enqueue()`入口当中会根据协议定义，通过接口名称找到对应的处理接口实例，并触发接口响应。并且最终的接口响应都在入口处进行回传。因此，实际上，_easybridge对象（在Java层中，其实是`EasyBridge`的实例）就是一个枢纽站，做任务的分派和结果的传递。

### 安全控制

每一个`BridgeHandler`实例，都可以定义自己的安全控制策略，对应的是一个`SecurityPolicyChecker`的实例，其定义如下：

```java
public interface SecurityPolicyChecker {
    boolean check(String url, String parameters);
}
```

每一个接口在接收到分派的指令之前，会先调用其安全控制策略，根据当前加载的页面地址以及传入的指令参数判断是否需要进行指令的分派，否则将会直接命令安全受限，错误返回，结果调用。

## 方案使用

**EasyBridge**是一个极其简单易用的方案，只需要简单的几步即可具备JavaScript层与Java层通讯的能力。在引入**EasyBridge**库作为依赖之后：

1. 继承/直接使用`EasyBridgeWebView`

   `EasyBridgeWebView`是功能的承载者，负责了bridge对象的注入，以及handler接口的管理（内部使用`EasyBridge对象管理）

2. 根据业务以及协议定义实现对应的`BridgeHandler`实例

3. 在加载第一步的webview的实例页面绑定第二步构造的handler实例

以上三步即完成了所有的工作。

如果你需要调试这个方案的实际工作，你可以在把手机连接到电脑之后，使用chrome进行调试。**EasyBridge**会把传递的结果信息以及错误信息打印在控制台之上。你将会很容易的感知和发现问题。关于在Chrome中调试web页面，你可以参考官方的教程文档[Remote Debugging WebViews](https://developers.google.com/web/tools/chrome-devtools/remote-debugging/webviews)