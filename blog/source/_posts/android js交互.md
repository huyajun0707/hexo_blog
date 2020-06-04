---
title: android js交互
date: 2019-5-15 20:52:40
updated: 2019-5-16 10:21:15
tags: 
  - android
categories:
  - [webview] 
toc: true
top_image: https://rmt.dogedoge.com/fetch/fluid/storage/bg/vdysjx.png?w=1920&fmt=webp
excerpt: WebView是Android系统提供能显示网页的系统控件，它是一个特殊的View，同时它也是一个ViewGroup可以有很多其他子View。在Android 4.4以下(不包含4.4)系统WebView底层实现是采用WebKit内核，而在Android 4.4及其以上Google 采用了chromium内核作为系统WebView的底层内核支持。

---
WebView是Android系统提供能显示网页的系统控件，它是一个特殊的View，同时它也是一个ViewGroup可以有很多其他子View。在Android 4.4以下(不包含4.4)系统WebView底层实现是采用WebKit内核，而在Android 4.4及其以上Google 采用了chromium内核作为系统WebView的底层内核支持。

目前开源的浏览器内核sdk不多，主要有以下几个：ChromeView、Crosswalk、TbsX5(腾讯浏览服务)。
1. 基于Chromium内核的开源ChromeView目前基本上没有维护，另一个问题是所编译出来的动态库太大，ARM 29M，x86 38M，这无疑对app体积来说是个大难题。因此放弃采用基于Chromium的ChromeView。
1. Crosswalk同样是基于Chromium内核，同样存在上述app体积问题，因此也放弃。
1. TbsX5基于谷歌Blink内核，并提供两种集成方案：1)只共享微信手Q空间的x5内核(for share)，2)独立下载x5内核(with download)。

## webview与js交互的几种方式

### Android 调用js代码

#### 1. 通过webview.loadUrl()

```bash
webView.loadUrl("javascript:callJs('方式一')");
```
 优点：调用方式简单
 
 缺点：获取返回值麻烦，效率低
 注意：
1. 这种方式Native调用js方法一定要在 onPageFinished() 回调之后才能调用，否则不会调用到。
2. 会让页面每次都会刷新，如果页面加载较慢很容易引起屏幕闪一下或者白屏，效果较差。

#### 2. 通过webview.evaluateJavascript()

```
webView.evaluateJavascript("javascript:callJs('方式二')", new ValueCallback<String>() {
    @Override
    public void onReceiveValue(String value) {
        //此处为 js 返回的结果
        Log.d("callJs返回值", "onReceiveValue: "+value);
    }
});
```
优点：效率高

缺点：仅Android4.4以上版本适用

### js调动Native的代码：（三种方式）

#### 1. WebView.addJavascriptInterface（）进行对象映射
Android native

```bash
webView.addJavascriptInterface(new NativeJsInterface(),"android");
//本地方法
public class NativeJsInterface{
    @JavascriptInterface
    public void goActivity(){
        startActivity(new Intent(MainActivity.this,JsBridgeActivity.class));
    }
}
```
H5  

```bash
<!DOCTYPE html>
<html>
<head> <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    <title>js交互演示</title>
    <!--JS代码-->
    <script>
    <!--Android需要调用的方法-->
    function callJs(str){
       alert("Android调用了JS的callJS方法！来自"+str);
   }
   </script>
</head>
<body>
    <b>测试</b><br/>
    <button onClick="window.android.goActivity()">
    跳转Activity
    </button>
</body>
</html>
```
 优点：方式简单
 
 缺点：Android4.2以下有漏洞

####  2. WebViewClient.shouldOverrideUrlLoading()回调拦截
与前端约定好协议，然后在WebviewClient的shouldOverrdeUrlLoading()拦截url，根据url调用对应本地方法。

Android Native代码

```bash
webView.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        Uri uri = Uri.parse(url);
        Log.d("--->", "shouldOverrideUrlLoading:"+url);
        if (uri.getScheme().equals("js")) {
            String param = uri.getQueryParameter("param");
            try {
                JSONObject jsonObject = new JSONObject(param);
                String method = jsonObject.getString("method");
                switch (method){
                    case "goActivity":
                        startActivity(new Intent(MainActivity.this, JsBridgeActivity.class));
                        break;
                }

            } catch (JSONException e) {
                e.printStackTrace();
            }


        } else
            view.loadUrl(url);

        return true;

    }
});
```
H5代码

```bash
function callAndroid(){
   <!--约定的url协议为：js://webview?arg1=111&arg2=222-->
       document.location = "js://webview?param="+"{\"method\":\"goActivity\",\"params\":\"具体参数\"}";
    }
```
优点：可以和ios统一

缺点：没有返回值，需要提前和前端商定协议。

#### 3. WebChromeClient中onJsAlert()、onJsConfirm()、onJsPrompt（）
利用js 的alert confirm prompt方法时， WebChromeClient中相应的方法会有回调，其中alert没有返回值，confirm返回布尔值，true为确认，false为取消，prompt可以返回任何值， 所以一般都会利用prompt来做为通信手段。

Android native 

```bash
//js调用本地方法，通过prompt拦截
webView.setWebChromeClient(new WebChromeClient() {
    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
        Uri uri = Uri.parse(message);
        if (uri.getScheme().equals("js")) {
            String param = uri.getQueryParameter("param");
            try {
                JSONObject jsonObject = new JSONObject(param);
                String method = jsonObject.getString("method");
                switch (method) {
                    case "goActivity":
                        startActivity(new Intent(MainActivity.this, JsBridgeActivity.class));
                        String ret = "js调用了native,这是返回结果";
                        result.confirm(ret);
                        return true;
                    default:
                        break;
                }

            } catch (JSONException e) {
                e.printStackTrace();
            }
        }
        return super.onJsPrompt(view, url, message, defaultValue, result);
    }
});
```
H5代码

```bash
//定义协议，可接收到返回值
function callAppByPrompt(){
    var result=prompt("js://webview?param="+"{\"method\":\"goActivity\",\"params\":\"具体参数\"}");
    alert("demo " + result);
}
```


优点：可以和IOS统一调用方式，让前端调用方便，没有漏洞问题

缺点：需要提前和前端商定协议，用户感知较明显，会弹出对话框（当然可以在Native层选择不弹框，但这样会让代码复杂性上升）

## jsbridge介绍及使用 (android)
jsbridge是对js与webveiw交互的封装，相互调用更简单方便。

```bash
//调用js方法，同时接收返回数据
bridgeWebView.callHandler("callJs", json, new CallBackFunction() {
    @Override
    public void onCallBack(String data) {
        Log.d("---->", "onCallBack: "+data);
    }
});
//声明natvie方法，供h5调用
bridgeWebView.registerHandler("callApp", new BridgeHandler() {

    @Override
    public void handler(String data, CallBackFunction function) {
        Log.i("---->", "handler = callApp, data from web = " + data);
        function.onCallBack("callApp exe, response data 测试 from Java");
    }

});
```
## 原理
Native调用js方法

```bash
//调用js方法，同时接收返回数据
bridgeWebView.callHandler("callJs", json, new CallBackFunction() {
    @Override
    public void onCallBack(String data) {
        Log.d("---->", "onCallBack: "+data);
    }
});
public void callHandler(String handlerName, String data, CallBackFunction callBack) {
       doSend(handlerName, data, callBack);
}

private void doSend(String handlerName, String data, CallBackFunction responseCallback) {
   //封装message对象
   Message m = new Message();
   if (!TextUtils.isEmpty(data)) {
      //将js方法的传入参数设置到message对象中
      m.setData(data);
   }
   if (responseCallback != null) {
      //创建一个callbackId做为标识，作用是做后续回调处理
      String callbackStr = String.format(BridgeUtil.CALLBACK_ID_FORMAT, ++uniqueId + (BridgeUtil.UNDERLINE_STR + SystemClock.currentThreadTimeMillis()));
      //将callback对象存储至本地MAP集合中
      responseCallbacks.put(callbackStr, responseCallback);
      //设置callbackId
      m.setCallbackId(callbackStr);
   }
   if (!TextUtils.isEmpty(handlerName)) {
       //将callJs方法名设置到message对象中
      m.setHandlerName(handlerName);
   }
   //处理当前js方法
   queueMessage(m);
}
private void queueMessage(Message m) {
   if (startupMessage != null) {
      //证明js环境未准备好，先添加到集合中，等待onPageFinish调用时再处理
      startupMessage.add(m);
   } else {
      //直接处理当前方法
      dispatchMessage(m);
   }
}
 /**
    * 分发message 必须在主线程才分发成功
    * @param m Message
    */
void dispatchMessage(Message m) {
       String messageJson = m.toJson();
       ......
       //组装最终要调用的callJs方法名，将messageJson以参数形式传入
       String javascriptCommand = String.format(BridgeUtil.JS_HANDLE_MESSAGE_FROM_JAVA, messageJson);
       // 必须要找主线程才会将数据传递出去 --- 划重点
       if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
           //最终通过loadurl调用js方法
           this.loadUrl(javascriptCommand);
       }
   }
   
   //接下来就是js端处理了，WebViewJavascriptBridge.js 
   
//提供给native调用,receiveMessageQueue 在会在页面加载完后赋值为null,所以直接调用—dispatchMessageFromNative
function _handleMessageFromNative(messageJSON) {
    console.log(messageJSON);
    if (receiveMessageQueue) {
        receiveMessageQueue.push(messageJSON);
    }
    _dispatchMessageFromNative(messageJSON);
   
}
  
function _dispatchMessageFromNative(messageJSON) {
    setTimeout(function() {
        var message = JSON.parse(messageJSON);
        var responseCallback;
        //java call finished, now need to call js callback function
        if (message.responseId) {
            //responseId不为空，处理callApp方法后的callback处理
        //根据responseId取callback对象
            responseCallback = responseCallbacks[message.responseId];
            if (!responseCallback) {
                return;
            }
            //调用回调方法
            responseCallback(message.responseData);
            //从集合中移除当前callback对象
            delete responseCallbacks[message.responseId];
        } else {
            //处理来自Java层的主动调用，callJs 直接发送 从loadurl直接过来的
            if (message.callbackId) {
                var callbackResponseId = message.callbackId;
                responseCallback = function(responseData) {
                    _doSend({
                        responseId: callbackResponseId,
                        responseData: responseData
                    });
                };
            }
            //查找指定handler，即获取对应的js函数方法，并调用
            var handler = WebViewJavascriptBridge._messageHandler;
            if (message.handlerName) {
                handler = messageHandlers[message.handlerName];
            }
            try {
                //处理handle即调用calljs方法后把回调传给native
                handler(message.data, responseCallback);
            } catch (exception) {
                if (typeof console != 'undefined') {
                    console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                }
            }
        }
    });
} 

//dosend方法会执行shouldOverrideUrlLoading中webView.flushMessageQueue();


void flushMessageQueue() {
   if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
      loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA, new CallBackFunction() {

         @Override
         public void onCallBack(String data) {
            // deserialize Message 反序列化消息
            List<Message> list = null;
            try {
               list = Message.toArrayList(data);
            } catch (Exception e) {
                       e.printStackTrace();
               return;
            }
            if (list == null || list.size() == 0) {
               return;
            }
            for (int i = 0; i < list.size(); i++) {
               Message m = list.get(i);
               String responseId = m.getResponseId();
               // 是否是response  CallBackFunction
               if (!TextUtils.isEmpty(responseId)) {
                   //如果responseId不为空则表示是callJs方法的callback对象
                  CallBackFunction function = responseCallbacks.get(responseId);
                  String responseData = m.getResponseData();
                  function.onCallBack(responseData);
                  responseCallbacks.remove(responseId);
               } else {
                   //如果responseId为空，则表示是callApp方法的callback对象
                  CallBackFunction responseFunction = null;
                  // if had callbackId 如果有回调Id
                  final String callbackId = m.getCallbackId();
                  if (!TextUtils.isEmpty(callbackId)) {
                     responseFunction = new CallBackFunction() {
                        @Override
                        public void onCallBack(String data) {
                           Message responseMsg = new Message();
                           responseMsg.setResponseId(callbackId);
                           responseMsg.setResponseData(data);
                           //通过loadurl将返回值传回js
                           queueMessage(responseMsg);
                        }
                     };
                  } else {
                     responseFunction = new CallBackFunction() {
                        @Override
                        public void onCallBack(String data) {
                           // do nothing
                        }
                     };
                  }
                  // BridgeHandler执行callApp的方法
                  BridgeHandler handler;
                  if (!TextUtils.isEmpty(m.getHandlerName())) {
                     handler = messageHandlers.get(m.getHandlerName());
                  } else {
                     handler = defaultHandler;
                  }
                  if (handler != null){
                     handler.handler(m.getData(), responseFunction);
                  }
               }
            }
         }
      });
   }
}
////webView.flushMessageQueue()首先去执行Js的_flushQueue()方法，并附带着CallBackFunction。
//Js的_flushQueue()方法会把sendMessageQueue中的所有message都回传给Java层。
//CallBackFunction就是把messageQueue解析出来后一个一个Message在for循环中处理，
//也正是在for循环中，”callJs”的Java层回调方法被执行了。
//刷新消息队列
public void loadUrl(String jsUrl, CallBackFunction returnCallback) {
    //直接调用javascript的fetchQueue
   this.loadUrl(jsUrl);
       // 添加至 Map<String, CallBackFunction>
   responseCallbacks.put(BridgeUtil.parseFunctionName(jsUrl), returnCallback);
}
```
js调用Native


```bash
bridgeWebView.registerHandler("callApp", new BridgeHandler() {

    @Override
    public void handler(String data, CallBackFunction function) {
        Log.i("---->", "handler = callApp, data from web = " + data);
        function.onCallBack("callApp exe, response data 测试 from Java");
    }

});
public void registerHandler(String handlerName, BridgeHandler handler) {
   if (handler != null) {
       //将方法保存到集合中
      messageHandlers.put(handlerName, handler);
   }
}

//在shouldOverrideurlLoading方法中拦截
@Override
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    try {
        url = URLDecoder.decode(url, "UTF-8");
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }

    if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) { // 如果是返回数据
        webView.handlerReturnData(url);
        return true;
    } else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) { //处理js调用本地方法
        webView.flushMessageQueue();
        return true;
    } else {
        return this.onCustomShouldOverrideUrlLoading(url)?true:super.shouldOverrideUrlLoading(view, url);
    }
}
```

##### 总结：
android调用js方法，流程

Android
callHandler
1. dosend
    将方法名与callback封装成Message对象，设置callbackId，交将callback对象存储到Map集合中
2. queueMessage
   判断当前js是否已注入成功
   未成功：先将Message添加到startupMessage集合中
   成功：下一步操作
3. dispatchMessage
   组装要调用的js方法，最终通过loadurl调用javascript:WebViewJavascriptBridge._handleMessageFromNative(参数)
H5
4. _handleMessageFromNative
   和Android端一样，判断页面是否加载完成
   未完成：将Message消息保存至数组
   完成：下一步
5. _dispatchMessageFromNative
  将MessageJson解析为Message对象，此处会判断message.responseId是否为空，如果是空将表示是接收到来自Native的方法，
  如果responseId不为空则表示是主动向Native发送的方法，这里由于是native调用js方法，所以responseId为空,调用_doSend()和
  handler(message.data, responseCallback);执行对应的js方法
6. _doSend
  这里处理回调，将返回数据传回native，执行messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
  会调用shouldOverrideUrlLoading中webView.flushMessageQueue();
7. flushMessageQueue
   webView.flushMessageQueue()首先去执行Js的_flushQueue()方法，并附带着CallBackFunction。

  Js的_flushQueue()方法会把sendMessageQueue中的所有message都回传给Java层。

  CallBackFunction就是把messageQueue解析出来后一个一个Message在for循环中处理，
   也正是在for循环中，”callJs”的Java层回调方法被执行了。




