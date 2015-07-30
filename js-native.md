# Cordova 3.x 源码分析（6） -- cordova.js 本地交互 JS<->Native

src/android/android/nativeapiprovider.js JS->Native 的具体交互形式 

Js **代码**

```
// file: src/android/android/nativeapiprovider.js
define("cordova/android/nativeapiprovider", function(require, exports, module) {

// WebView中是否通过addJavascriptInterface提供了访问ExposedJsApi.java的_cordovaNative对象
// 如果不存在选择prompt()形式的交互方式
var nativeApi = this._cordovaNative || require('cordova/android/promptbasednativeapi');
var currentApi = nativeApi;

module.exports = {
    // 获取当前交互方式
    get: function() { return currentApi; },

    // 设置使用prompt()交互方式
    // （true:prompt false:自动选择）
    setPreferPrompt: function(value) {
        currentApi = value ? require('cordova/android/promptbasednativeapi') : nativeApi;
    },

    // 直接设置交互方式对象（很少用到）
    set: function(value) {
        currentApi = value;
    }
};

});
```

src/android/android/promptbasednativeapi.js 通过 prompt()和 Native 交互（Android2.3 simulator 的 Bug） 

Js **代码**

```
// file: src/android/android/promptbasednativeapi.js
define("cordova/android/promptbasednativeapi", function(require, exports, module) {

// 由于Android2.3模拟器存在Bug，不支持addJavascriptInterface()
// 所以借助prompt()来和Native进行交互
// Native端会在CordovaChromeClient.onJsPrompt（）中拦截处理
module.exports = {
    // 调用Native API
    exec: function(service, action, callbackId, argsJson) {
        return prompt(argsJson, 'gap:'+JSON.stringify([service, action, callbackId]));
    },

    // 设置Native->JS的桥接模式
    setNativeToJsBridgeMode: function(value) {
        prompt(value, 'gap_bridge_mode:');
    },

    // 接收消息
    retrieveJsMessages: function(fromOnlineEvent) {
        return prompt(+fromOnlineEvent, 'gap_poll:');
    }
};

});
```

src/android/exec.js 执行 JS->Native 交互 

Js **代码**

```
// file: src/android/exec.js
define("cordova/exec", function(require, exports, module) {

var cordova = require('cordova'),
    nativeApiProvider = require('cordova/android/nativeapiprovider'),
    utils = require('cordova/utils'),
    base64 = require('cordova/base64'),

    // JS->Native的可选交互形式一览
    jsToNativeModes = {
        // 基于prompt()的交互 
        PROMPT: 0,
        // 基于JavascriptInterface的交互 
        JS_OBJECT: 1,
        // 基于URL的交互 
        // ***由于安全问题，默认已经设置成不可用的！！！
        // NativeToJsMessageQueue.ENABLE_LOCATION_CHANGE_EXEC_MODE=false
        LOCATION_CHANGE: 2
    },

    // Native->JS的可选交互形式一览
    nativeToJsModes = {
        // 轮询（JS->Native自助获取消息）
        POLLING: 0,
        // 使用 webView.loadUrl("javascript:") 来执行消息
        // 解决软键盘的Bug
        LOAD_URL: 1,
        // 拦截事件监听,使用online/offline事件来告诉JS获取消息
        // 默认值 NativeToJsMessageQueue.DEFAULT_BRIDGE_MODE=2
        ONLINE_EVENT: 2,
        // 反射Webview的私有API来执行JS（需要Android 3.2.4以上版本）
        PRIVATE_API: 3
    },

    // 当前JS->Native的交互形式
    jsToNativeBridgeMode,
    // 当前Native->JS的交互形式
    nativeToJsBridgeMode = nativeToJsModes.ONLINE_EVENT,
    pollEnabled = false,
    messagesFromNative = [];

// 执行Cordova提供的API
// 比如： exec(successCallback, errorCallback, "Camera", "takePicture", args);
function androidExec(success, fail, service, action, args) {
    // 默认采用JavascriptInterface交互方式
    if (jsToNativeBridgeMode === undefined) {
        androidExec.setJsToNativeBridgeMode(jsToNativeModes.JS_OBJECT);
    }

    // 如果参数中存在ArrayBuffer类型的参数，转化成字符串
    for (var i = 0; i < args.length; i++) {
        if (utils.typeName(args[i]) == 'ArrayBuffer') {
            args[i] = base64.fromArrayBuffer(args[i]);
        }
    }

    var callbackId = service + cordova.callbackId++,
        // 把所有参数转换成JSON串
        argsJson = JSON.stringify(args);

    // 设置回调函数
    if (success || fail) {
        cordova.callbacks[callbackId] = {success:success, fail:fail};
    }

    if (jsToNativeBridgeMode == jsToNativeModes.LOCATION_CHANGE) {
        // 基于URL的交互(需要手动修改NativeToJsMessageQueue.java的常量配置才能起效)
        // Native端会在CordovaWebViewClient.shouldOverrideUrlLoading()中拦截处理
        window.location = 'http://cdv_exec/' + service + '#' + action + '#' + callbackId + '#' + argsJson;
    } else {
        // 选择合适的交互方式和Native进行交互
        // 根据Native端NativeToJsMessageQueue.DISABLE_EXEC_CHAINING的配置，回传消息可以是同步或者异步
        // 默认是同步的，返回PluginResult对象的JSON串。异步的话messages为空。
        var messages = nativeApiProvider.get().exec(service, action, callbackId, argsJson);

        if (jsToNativeBridgeMode == jsToNativeModes.JS_OBJECT && messages === "@Null arguments.") {
            // 如果参数被传递到Java端，但是接收到的是null，切换交互方式到prompt()在执行一次
            // Galaxy S2在传递某些Unicode字符的时候少数情况下有问题，
            // 参考 https://issues.apache.org/jira/browse/CB-2666
            androidExec.setJsToNativeBridgeMode(jsToNativeModes.PROMPT);
            androidExec(success, fail, service, action, args);

            // 执行完成后，把交互方式再切回JavascriptInterface
            androidExec.setJsToNativeBridgeMode(jsToNativeModes.JS_OBJECT);
            return;
        } else {
            // 处理Native返回的消息
            androidExec.processMessages(messages);
        }
    }
}

function pollOnceFromOnlineEvent() {
    pollOnce(true);
}

// 从Native的消息队列中获取消息
function pollOnce(opt_fromOnlineEvent) {
    var msg = nativeApiProvider.get().retrieveJsMessages(!!opt_fromOnlineEvent);
    androidExec.processMessages(msg);
}

function pollingTimerFunc() {
    if (pollEnabled) {
        pollOnce();
        setTimeout(pollingTimerFunc, 50);
    }
}

function hookOnlineApis() {
    function proxyEvent(e) {
        cordova.fireWindowEvent(e.type);
    }
    window.addEventListener('online', pollOnceFromOnlineEvent, false);
    window.addEventListener('offline', pollOnceFromOnlineEvent, false);
    cordova.addWindowEventHandler('online');
    cordova.addWindowEventHandler('offline');
    document.addEventListener('online', proxyEvent, false);
    document.addEventListener('offline', proxyEvent, false);
}

// 添加online/offline事件
hookOnlineApis();

// 外部可以访问到交互方式的常量
androidExec.jsToNativeModes = jsToNativeModes;
androidExec.nativeToJsModes = nativeToJsModes;

// 设置JS->Native的交互方式
androidExec.setJsToNativeBridgeMode = function(mode) {
    // JavascriptInterface方式但是Native无法提供_cordovaNative对象的时候强制切到prompt（）
    if (mode == jsToNativeModes.JS_OBJECT && !window._cordovaNative) {
        console.log('Falling back on PROMPT mode since _cordovaNative is missing. Expected for Android 3.2 and lower only.');
        mode = jsToNativeModes.PROMPT;
    }
    nativeApiProvider.setPreferPrompt(mode == jsToNativeModes.PROMPT);
    jsToNativeBridgeMode = mode;
};

// 设置Native->JS的交互方式
androidExec.setNativeToJsBridgeMode = function(mode) {
    if (mode == nativeToJsBridgeMode) {
        return;
    }

    // 如果以前是Poll的方式,，先回置到非Poll
    if (nativeToJsBridgeMode == nativeToJsModes.POLLING) {
        pollEnabled = false;
    }

    nativeToJsBridgeMode = mode;

    // 告诉Native端，JS端获取消息的方式
    nativeApiProvider.get().setNativeToJsBridgeMode(mode);

·   // 如果是在JS端Poll的方式的话
    if (mode == nativeToJsModes.POLLING) {
        pollEnabled = true;
        // 停顿后执行exec获取消息message
        setTimeout(pollingTimerFunc, 1);
    }
};

// 处理从Native返回的一条消息
// 
// 回传消息的完整格式：
// （1）消息的长度+空格+J+JavaScript代码
// 44 Jcordova.callbackFromNative('InAppBrowser1478332075',true,1,[{"type":"loadstop","url":"http:\/\/www.baidu.com\/"}],true});
// （2）消息的长度+空格+成功失败标示（J/S/F）+keepCallback标示+具体的状态码+空格+回调ID+空格+回传数据
// 78 S11 InAppBrowser970748887 {"type":"loadstop","url":"http:\/\/www.baidu.com\/"}
// 28 S01 Notification970748888 n0
//
// 默认是关闭了返回Js代码，使用返回数据。
// NativeToJsMessageQueue.FORCE_ENCODE_USING_EVAL=false
function processMessage(message) {
    try {
        var firstChar = message.charAt(0);
        if (firstChar == 'J') {
            // 执行回传的JavaScript代码
            eval(message.slice(1));
        } else if (firstChar == 'S' || firstChar == 'F') {
            // S代表处理成功（包含没有数据），F代表处理失败
            var success = firstChar == 'S';
            var keepCallback = message.charAt(1) == '1';
            var spaceIdx = message.indexOf(' ', 2);
            var status = +message.slice(2, spaceIdx);
            var nextSpaceIdx = message.indexOf(' ', spaceIdx + 1);
            var callbackId = message.slice(spaceIdx + 1, nextSpaceIdx);
            // 回传值类型
            var payloadKind = message.charAt(nextSpaceIdx + 1);
            var payload;
            if (payloadKind == 's') {
                // 字符串:s+字符串
                payload = message.slice(nextSpaceIdx + 2);
            } else if (payloadKind == 't') {
                // 布尔值:t/f
                payload = true;
            } else if (payloadKind == 'f') {
                // 布尔值:t/f
                payload = false;
            } else if (payloadKind == 'N') {
                // Null:N
                payload = null;
            } else if (payloadKind == 'n') {
                // 数值：n+具体值
                payload = +message.slice(nextSpaceIdx + 2);
            } else if (payloadKind == 'A') {
                // ArrayBuffer：A+数据
                var data = message.slice(nextSpaceIdx + 2);
                var bytes = window.atob(data);
                var arraybuffer = new Uint8Array(bytes.length);
                for (var i = 0; i < bytes.length; i++) {
                    arraybuffer[i] = bytes.charCodeAt(i);
                }
                payload = arraybuffer.buffer;
            } else if (payloadKind == 'S') {
                // 二进制字符串：S+字符串
                payload = window.atob(message.slice(nextSpaceIdx + 2));
            } else {
                // JSON：JSON串
                payload = JSON.parse(message.slice(nextSpaceIdx + 1));
            }
            // 调用回调函数
            cordova.callbackFromNative(callbackId, success, status, [payload], keepCallback);
        } else {
            console.log("processMessage failed: invalid message:" + message);
        }
    } catch (e) {
        console.log("processMessage failed: Message: " + message);
        console.log("processMessage failed: Error: " + e);
        console.log("processMessage failed: Stack: " + e.stack);
    }
}

// 处理Native返回的消息
androidExec.processMessages = function(messages) {
    // 如果消息存在的话（异步没有直接返回）
    if (messages) {

        // 把传入的消息放到数组中
        messagesFromNative.push(messages);

        // messagesFromNative是全局的，而processMessages方法可重入，
        // 所以只需放到数组，后边循环处理即可
        if (messagesFromNative.length > 1) {
            return;
        }

        // 遍历从Native获得所有消息
        while (messagesFromNative.length) {
            // 处理完成后才可以从数组中删除
            messages = messagesFromNative[0];

            // Native返回星号代表消息需要等一会儿再取
            if (messages == '*') {
                // 删除数组的第一个元素
                messagesFromNative.shift();
                // 再次去获取消息
                window.setTimeout(pollOnce, 0);
                return;
            }

            // 获取消息的长度
            var spaceIdx = messages.indexOf(' ');
            var msgLen = +messages.slice(0, spaceIdx);
            // 获取第一个消息
            var message = messages.substr(spaceIdx + 1, msgLen);
            // 截取掉第一个消息
            messages = messages.slice(spaceIdx + msgLen + 1);

            // 处理第一个消息
            processMessage(message);

            // 如果消息包含多个，继续处理；单个的话删除本地消息数组中的数据
            if (messages) {
                messagesFromNative[0] = messages;
            } else {
                messagesFromNative.shift();
            }
        }
    }
};

module.exports = androidExec;

});
```

