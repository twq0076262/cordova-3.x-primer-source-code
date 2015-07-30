#  Cordova 3.x 源码分析（5） -- cordova.js 导入、初始化、启动、加载插件

执行 cordova.js 的入口就以下2行代码： 

Js **代码** 

```
// 导入cordova
window.cordova = require('cordova');
// 启动处理
require('cordova/init');
```

src/cordova.js 事件的处理和回调，外部访问 cordova.js 的入口 

其中第一步是加载整个模块系统和外部访问 cordova.js 的入口，基于事件通道提供了整体的事件拦截控制及回调。代码不是很复杂。 

源码： 

Js **代码**

```
// file: src/cordova.js
define("cordova", function(require, exports, module) {

// 调用通道和平台模块
var channel = require('cordova/channel');
var platform = require('cordova/platform');

// 备份document和window的事件监听器
var m_document_addEventListener = document.addEventListener;
var m_document_removeEventListener = document.removeEventListener;
var m_window_addEventListener = window.addEventListener;
var m_window_removeEventListener = window.removeEventListener;

// 保存自定义的document和window的事件监听器
var documentEventHandlers = {},
    windowEventHandlers = {};

// 拦截document和window的事件监听器（addEventListener/removeEventListener）
// 存在自定义的事件监听器的话，使用自定义的；不存在的话调用备份document和window的事件监听器
document.addEventListener = function(evt, handler, capture) {
    var e = evt.toLowerCase();
    if (typeof documentEventHandlers[e] != 'undefined') {
        documentEventHandlers[e].subscribe(handler);
    } else {
        m_document_addEventListener.call(document, evt, handler, capture);
    }
};
window.addEventListener = function(evt, handler, capture) {
    var e = evt.toLowerCase();
    if (typeof windowEventHandlers[e] != 'undefined') {
        windowEventHandlers[e].subscribe(handler);
    } else {
        m_window_addEventListener.call(window, evt, handler, capture);
    }
};
document.removeEventListener = function(evt, handler, capture) {
    var e = evt.toLowerCase();
    if (typeof documentEventHandlers[e] != "undefined") {
        documentEventHandlers[e].unsubscribe(handler);
    } else {
        m_document_removeEventListener.call(document, evt, handler, capture);
    }
};
window.removeEventListener = function(evt, handler, capture) {
    var e = evt.toLowerCase();
    if (typeof windowEventHandlers[e] != "undefined") {
        windowEventHandlers[e].unsubscribe(handler);
    } else {
        m_window_removeEventListener.call(window, evt, handler, capture);
    }
};

// 创建一个指定type的事件。
// 参考：https://developer.mozilla.org/en-US/docs/Web/API/document.createEvent#Notes
function createEvent(type, data) {
    var event = document.createEvent('Events');
    // 指定事件名、不可冒泡、不可取消
    event.initEvent(type, false, false);
    // 自定义数据
    if (data) {
        for (var i in data) {
            if (data.hasOwnProperty(i)) {
                event[i] = data[i];
            }
        }
    }
    return event;
}

// 外部访问cordova.js的入口
var cordova = {
    // 模块系统
    define:define,
    require:require,
    // 版本号和平台名
    version:CORDOVA_JS_BUILD_LABEL,
    platformId:platform.id,

    // 为了拦截document和window的事件监听器,添加或删除自定义的事件监听器
    addWindowEventHandler:function(event) {
        return (windowEventHandlers[event] = channel.create(event));
    },
    // sticky 是指一旦被调用那么它以后都保持被调用的状态，所定义的监听器会被立即执行。
    // 比如： deviceready事件只触发一次，以后的所有监听都是立即执行的。
    addStickyDocumentEventHandler:function(event) { 
        return (documentEventHandlers[event] = channel.createSticky(event));
    },
    addDocumentEventHandler:function(event) {
        return (documentEventHandlers[event] = channel.create(event));
    },
    removeWindowEventHandler:function(event) {
        delete windowEventHandlers[event];
    },
    removeDocumentEventHandler:function(event) {
        delete documentEventHandlers[event];
    },

    // 获取拦截前的document和window的事件监听器
    getOriginalHandlers: function() {
        return {'document': {'addEventListener': m_document_addEventListener, 'removeEventListener': m_document_removeEventListener},
        'window': {'addEventListener': m_window_addEventListener, 'removeEventListener': m_window_removeEventListener}};
    },

    // 调用document的事件
    fireDocumentEvent: function(type, data, bNoDetach) {
        var evt = createEvent(type, data);
        if (typeof documentEventHandlers[type] != 'undefined') {
            // 判断是否需要抛出事件异常
            if( bNoDetach ) {
                // 通过Channel的fire方法来调用事件（apply）
                documentEventHandlers[type].fire(evt);
            }
            else {
                // setTimeout(callback, 0) 的意思是DOM构成完毕、事件监听器执行完后立即执行
                setTimeout(function() {
                    // 调用加载cordova.js之前定义的那些deviceready事件
                    if (type == 'deviceready') {
                        document.dispatchEvent(evt);
                    }
                    // 通过Channel的fire方法来调用事件（apply）
                    documentEventHandlers[type].fire(evt);
                }, 0);
            }
        } else {
            // 直接调用事件
            document.dispatchEvent(evt);
        }
    },

    // 调用window的事件
    fireWindowEvent: function(type, data) {
        var evt = createEvent(type,data);
        if (typeof windowEventHandlers[type] != 'undefined') {
            setTimeout(function() {
                windowEventHandlers[type].fire(evt);
            }, 0);
        } else {
            window.dispatchEvent(evt);
        }
    },

    // 插件回调相关-------------------------------------

    // 回调ID中间的一个随机数(真正的ID：插件名+随机数)
    callbackId: Math.floor(Math.random() * 2000000000),
    // 回调函数对象，比如success,fail
    callbacks:  {},
    // 回调状态
    callbackStatus: {
        NO_RESULT: 0,
        OK: 1,
        CLASS_NOT_FOUND_EXCEPTION: 2,
        ILLEGAL_ACCESS_EXCEPTION: 3,
        INSTANTIATION_EXCEPTION: 4,
        MALFORMED_URL_EXCEPTION: 5,
        IO_EXCEPTION: 6,
        INVALID_ACTION: 7,
        JSON_EXCEPTION: 8,
        ERROR: 9
    },

    // 以后使用callbackFromNative代替callbackSuccess和callbackError 
    callbackSuccess: function(callbackId, args) {
        try {
            cordova.callbackFromNative(callbackId, true, args.status, [args.message], args.keepCallback);
        } catch (e) {
            console.log("Error in error callback: " + callbackId + " = "+e);
        }
    },
    callbackError: function(callbackId, args) {
        try {
            cordova.callbackFromNative(callbackId, false, args.status, [args.message], args.keepCallback);
        } catch (e) {
            console.log("Error in error callback: " + callbackId + " = "+e);
        }
    },

    // 调用回调函数
    callbackFromNative: function(callbackId, success, status, args, keepCallback) {
        var callback = cordova.callbacks[callbackId];
        // 判断是否定义了回调函数
        if (callback) {
            if (success && status == cordova.callbackStatus.OK) {
                // 调用success函数
                callback.success && callback.success.apply(null, args);
            } else if (!success) {
                // 调用fail函数
                callback.fail && callback.fail.apply(null, args);
            }
            // 如果设置成不再保持回调，删除回调函数对象
            if (!keepCallback) {
                delete cordova.callbacks[callbackId];
            }
        }
    },

    // 没有地方用到！
    // 目的是把你自己的函数在注入到Cordova的生命周期中。
    addConstructor: function(func) {
        channel.onCordovaReady.subscribe(function() {
            try {
                func();
            } catch(e) {
                console.log("Failed to run constructor: " + e);
            }
        });
    }
};

module.exports = cordova;

});
```

src/common/init.js 初始化处理 

第二步是就是执行初始化处理 

源码： 

Js **代码**

```
// file: src/common/init.js
define("cordova/init", function(require, exports, module) {

var channel = require('cordova/channel');
var cordova = require('cordova');
var modulemapper = require('cordova/modulemapper');
var platform = require('cordova/platform');
var pluginloader = require('cordova/pluginloader');

// 定义平台初期化处理必须在onNativeReady和onPluginsReady之后进行
var platformInitChannelsArray = [channel.onNativeReady, channel.onPluginsReady];

// 输出事件通道名到日志
function logUnfiredChannels(arr) {
    for (var i = 0; i < arr.length; ++i) {
        if (arr[i].state != 2) {
            console.log('Channel not fired: ' + arr[i].type);
        }
    }
}

// 5秒之后deviceready事件还没有被调用将输出log提示
// 出现这个错误的情况比较复杂，比如，加载的plugin太多等等
window.setTimeout(function() {
    if (channel.onDeviceReady.state != 2) {
        console.log('deviceready has not fired after 5 seconds.');
        logUnfiredChannels(platformInitChannelsArray);
        logUnfiredChannels(channel.deviceReadyChannelsArray);
    }
}, 5000);

// 替换window.navigator
function replaceNavigator(origNavigator) {
    // 定义新的navigator，把navigator的原型链赋给新的navigator的原型链
    var CordovaNavigator = function() {};
    CordovaNavigator.prototype = origNavigator;
    var newNavigator = new CordovaNavigator();
    // 判断是否存在Function.bind函数
    if (CordovaNavigator.bind) {
        for (var key in origNavigator) {
            if (typeof origNavigator[key] == 'function') {
                // 通过bind创建一个新的函数（this指向navigator）后赋给新的navigator
                // 参考：https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
                newNavigator[key] = origNavigator[key].bind(origNavigator);
            }
        }
    }
    return newNavigator;
}

// 替换webview的BOM对象navigator
// Cordova提供的接口基本都是：navigator.<plugin_name>.<action_name>
if (window.navigator) {
    window.navigator = replaceNavigator(window.navigator);
}

// 定义console.log()
if (!window.console) {
    window.console = {
        log: function(){}
    };
}

// 定义console.warn()
if (!window.console.warn) {
    window.console.warn = function(msg) {
        this.log("warn: " + msg);
    };
}

// 注册pause，resume，deviceready事件通道，并应用到Cordova自定义的事件拦截
// 这样页面定义的事件监听器就能订阅到相应的通道上了。
channel.onPause = cordova.addDocumentEventHandler('pause');
channel.onResume = cordova.addDocumentEventHandler('resume');
channel.onDeviceReady = cordova.addStickyDocumentEventHandler('deviceready');

// 如果此时DOM加载完成，触发onDOMContentLoaded事件通道中的事件处理
if (document.readyState == 'complete' || document.readyState == 'interactive') {
    channel.onDOMContentLoaded.fire();
} else {
    // 如果此时DOM没有加载完成，定义一个监听器在DOM完成后触发事件通道的处理
    // 注意这里调用的webview的原生事件监听
    document.addEventListener('DOMContentLoaded', function() {
        channel.onDOMContentLoaded.fire();
    }, false);
}

// 以前版本是在CordovaLib中反向执行js把_nativeReady设置成true后触发事件通道
// 现在已经改成在平台启动处理中立即触发
// 参考：https://issues.apache.org/jira/browse/CB-3066
if (window._nativeReady) {
    channel.onNativeReady.fire();
}

// 给常用的模块起个别名
// 比如：就可以直接使用cordova.exec(...)来代替var exec = require('cordova/exec'); exec(...);
// 不过第一行第二个参数应该是“Cordova”，c应该大写！！！
modulemapper.clobbers('cordova', 'cordova');
modulemapper.clobbers('cordova/exec', 'cordova.exec');
modulemapper.clobbers('cordova/exec', 'Cordova.exec');

// 调用平台初始化启动处理
platform.bootstrap && platform.bootstrap();

// 所有插件加载完成后，触发onPluginsReady事件通道中的事件处理
pluginloader.load(function() {
    channel.onPluginsReady.fire();
});

// 一旦本地代码准备就绪，创建cordova所需的所有对象
channel.join(function() {
    // 把所有模块附加到window对象上
    modulemapper.mapModules(window);

    // 如果平台有特殊的初始化处理，调用它（目前来看都没有）
    platform.initialize && platform.initialize();

    // 触发onCordovaReady事件通道，标示cordova准备完成
    channel.onCordovaReady.fire();

    // 一切准备就绪后，执行deviceready事件通道上的所有事件。
    channel.join(function() {
        require('cordova').fireDocumentEvent('deviceready');
    }, channel.deviceReadyChannelsArray); // onCordovaReady、onDOMContentLoaded

}, platformInitChannelsArray); // onNativeReady、onPluginsReady


});
```

src/android/platform.js 平台启动处理 

源码： 

Js **代码** 

```
// file: src/android/platform.js
define("cordova/platform", function(require, exports, module) {

module.exports = {
    id: 'android',
    // 平台启动处理(各个平台处理都不一样，比如ios就只需要触发onNativeReady)
    bootstrap: function() {
        var channel = require('cordova/channel'),
            cordova = require('cordova'),
            exec = require('cordova/exec'),
            modulemapper = require('cordova/modulemapper');

        // 把exec()的执行从WebCore线程变到UI线程上来
        // 后台PluginManager初始化的时候默认添加了一个'PluginManager的插件
        exec(null, null, 'PluginManager', 'startup', []);

        // 触发onNativeReady事件通道，告诉JS本地代码已经完成
        channel.onNativeReady.fire();

        // app插件。现在没有被单独抽出去，没找到合适的地方。
        modulemapper.clobbers('cordova/plugin/android/app', 'navigator.app');

        // 给返回按钮注意个监听器
        var backButtonChannel = cordova.addDocumentEventHandler('backbutton');
        backButtonChannel.onHasSubscribersChange = function() {
            // 如果只为返回按钮定义了1个事件监听器的话，通知后台覆盖默认行为
            exec(null, null, "App", "overrideBackbutton", [this.numHandlers == 1]);
        };

        // 添加菜单和搜素的事件监听
        cordova.addDocumentEventHandler('menubutton');
        cordova.addDocumentEventHandler('searchbutton');

        // 启动完成后，告诉本地代码显示WebView
        channel.onCordovaReady.subscribe(function() {
            exec(null, null, "App", "show", []);
        });
    }
};

});
```

src/common/pluginloader.js 加载所有 cordova_plugins.js 中定义的模块，执行完成后会触发onPluginsReady 

Js **代码**

```
// file: src/common/pluginloader.js
define("cordova/pluginloader", function(require, exports, module) {

var modulemapper = require('cordova/modulemapper');
var urlutil = require('cordova/urlutil');

// 创建<script>tag,把js文件动态添加到head中
function injectScript(url, onload, onerror) {
    var script = document.createElement("script");
    script.onload = onload;
    script.onerror = onerror || onload; // 出错的时候也执行onload处理
    script.src = url;
    document.head.appendChild(script);
}

// 加载到head中的插件js脚本定义如下：
//  cordova.define("org.apache.cordova.xxx", function(require, exports, module) { ... });
// 模块名称是cordova_plugins.js中定义的id，所以要把该id指向定义好的clobbers
function onScriptLoadingComplete(moduleList, finishPluginLoading) {

    for (var i = 0, module; module = moduleList[i]; i++) {
        if (module) {
            try {
                // 把该模块需要clobber的clobber到指定的clobbers里
                if (module.clobbers && module.clobbers.length) {
                    for (var j = 0; j < module.clobbers.length; j++) {
                        modulemapper.clobbers(module.id, module.clobbers[j]);
                    }
                }

                // 把该模块需要合并的部分合并到指定的模块里
                if (module.merges && module.merges.length) {
                    for (var k = 0; k < module.merges.length; k++) {
                        modulemapper.merges(module.id, module.merges[k]);
                    }
                }

                // 处理只希望require()的模块
                // <js-module src="www/xxx.js" name="Xxx">
                //    <runs />
                // </js-module>
                if (module.runs && !(module.clobbers && module.clobbers.length) && !(module.merges && module.merges.length)) {
                    modulemapper.runs(module.id);
                }
            }
            catch(err) {
            }
        }
    }

    // 插件js脚本加载完成后，执行回调!!!
    finishPluginLoading();
}

// 加载所有cordova_plugins.js中定义的js-module
function handlePluginsObject(path, moduleList, finishPluginLoading) {

    var scriptCounter = moduleList.length;

    // 没有插件,直接执行回调后返回
    if (!scriptCounter) {
        finishPluginLoading();
        return;
    }

    // 加载每个插件js的脚本的回调
    function scriptLoadedCallback() {
        // 加载完成一个就把计数器减1
        if (!--scriptCounter) {
            // 直到所有插件的js脚本都被加载完成后clobber
            onScriptLoadingComplete(moduleList, finishPluginLoading);
        }
    }

    // 依次把插件的js脚本添加到head中后加载
    for (var i = 0; i < moduleList.length; i++) {
        injectScript(path + moduleList[i].file, scriptLoadedCallback);
    }
}

// 注入插件的js脚本
function injectPluginScript(pathPrefix, finishPluginLoading) {
    var pluginPath = pathPrefix + 'cordova_plugins.js';

    // 根据cordova.js文件的路径首先把cordova_plugins.js添加到head中后加载
    injectScript(pluginPath, function() {
        try {
            // 导入cordova_plugins.jsz中定义的'cordova/plugin_list'模块
            // 这个文件的内容是根据所有插件的plugin.xml生成的。
            var moduleList = require("cordova/plugin_list");
            
            // 加载所有cordova_plugins.js中定义的js-module
            handlePluginsObject(pathPrefix, moduleList, finishPluginLoading);
        }
        catch (e) {
            // 忽略cordova_plugins.js记载失败、或者文件不存在等错误
            finishPluginLoading();
        }
    }, finishPluginLoading);
}

// 获取cordova.js文件的路径
function findCordovaPath() {
    var path = null;
    var scripts = document.getElementsByTagName('script');
    var term = 'cordova.js';
    for (var n = scripts.length-1; n>-1; n--) {
        var src = scripts[n].src;
        if (src.indexOf(term) == (src.length - term.length)) {
            path = src.substring(0, src.length - term.length);
            break;
        }
    }
    return path;
}

// 加载所有cordova_plugins.js中定义的js-module
// 执行完成后会触发onPluginsReady（异步执行）
exports.load = function(callback) {
    // 取cordova.js文件所在的路径
    var pathPrefix = findCordovaPath();
    if (pathPrefix === null) {
        console.log('Could not find cordova.js script tag. Plugin loading may fail.');
        pathPrefix = '';
    }

    // 注入插件的js脚本，执行完成后回调onPluginsReady
    injectPluginScript(pathPrefix, callback);
};


});
```

