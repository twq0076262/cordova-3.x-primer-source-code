# Cordova 3.x 源码分析（4） -- cordova.js 事件通道 pub/sub

作为观察者模式(Observer)的一种变形，很多 MV*框架（比如：Dojo、Backbone.js）中都提供发布/订阅模型来对代码进行解耦。cordova.js 中也提供了一个自定义的 pub-sub 模型，基于该模型提供了一些事件通道，用来控制通道中的事件什么时候以什么样的顺序被调用，以及各个事件通道的调用。 

src/common/channel.js 的代码结构也是一个很经典的定义结构（构造函数、实例、修改函数原型共享实例方法），它提供事件通道上事件的订阅（subscribe）、撤消订阅（unsubscribe）、调用（fire）。最后发布了8个事件通道。 

源码如下： 

Java **代码**

```
// file: src/common/channel.js
define("cordova/channel", function(require, exports, module) {

var utils = require('cordova/utils'),
    nextGuid = 1;

// ①事件通道的构造函数
var Channel = function(type, sticky) {
    // 通道名称
    this.type = type;
    // 通道上的所有事件处理函数Map（索引为guid）
    this.handlers = {};
    // 通道的状态（0：非sticky, 1:sticky但未调用, 2:sticky已调用）
    this.state = sticky ? 1 : 0;
    // 对于sticky事件通道备份传给fire()的参数
    this.fireArgs = null;
    // 当前通道上的事件处理函数的个数
    this.numHandlers = 0;
    // 订阅第一个事件或者取消订阅最后一个事件时调用自定义的处理
    this.onHasSubscribersChange = null;
},

// ②事件通道外部接口
    channel = {

        // 把指定的函数h订阅到c的各个通道上，保证h在每个通道的最后被执行
        join: function(h, c) {
            var len = c.length,
                i = len,
                f = function() {
                    if (!(--i)) h();
                };
            // 把事件处理函数h订阅到c的各个事件通道上
            for (var j=0; j<len; j++) {
                // 必须是sticky事件通道
                if (c[j].state === 0) {
                    throw Error('Can only use join with sticky channels.');
                }
                c[j].subscribe(f);
            }
            // 执行h
            if (!len) h();
        },

        // 创建事件通道
        create: function(type) {
            return channel[type] = new Channel(type, false);
        },

        // 创建sticky事件通道
        createSticky: function(type) {
            return channel[type] = new Channel(type, true);
        },

        // 保存deviceready事件之前要调用的事件
        deviceReadyChannelsArray: [],
        deviceReadyChannelsMap: {},

        // 设置deviceready事件之前必须要完成的事件
        waitForInitialization: function(feature) {
            if (feature) {
                var c = channel[feature] || this.createSticky(feature);
                this.deviceReadyChannelsMap[feature] = c;
                this.deviceReadyChannelsArray.push(c);
            }
        },

        // 以前版本的代码，现在好像没有用！
        initializationComplete: function(feature) {
            var c = this.deviceReadyChannelsMap[feature];
            if (c) {
                c.fire();
            }
        }
    };

// 这个应该放入utils里，就是一个函数的判断
function forceFunction(f) {
    if (typeof f != 'function') throw "Function required as first argument!";
}

// ③修改函数原型共享实例方法-----------------------

// 向事件通道订阅事件处理函数(subscribe部分）
// f:事件处理函数 c:事件的上下文（可省略）
Channel.prototype.subscribe = function(f, c) {
    // 事件处理函数校验
    forceFunction(f);

    // 如果是被订阅过的sticky事件，就直接调用。
    if (this.state == 2) {
        f.apply(c || this, this.fireArgs);
        return;
    }

    var func = f,
        guid = f.observer_guid;

    // 如果事件有上下文，要先把事件函数包装一下带上上下文
    if (typeof c == "object") { func = utils.close(c, f); }

    // 自增长的ID
    if (!guid) {
        guid = '' + nextGuid++;
    }
    // 把自增长的ID反向设置给函数，以后撤消订阅或内部查找用
    func.observer_guid = guid;
    f.observer_guid = guid;

    // 判断该guid索引的事件处理函数是否存在（保证订阅一次）
    if (!this.handlers[guid]) {
        // 订阅到该通道上（索引为guid）
        this.handlers[guid] = func;
        // 通道上的事件处理函数的个数增1
        this.numHandlers++;
        if (this.numHandlers == 1) {
            // 订阅第一个事件时调用自定义的处理（比如：第一次按下返回按钮提示“再按一次...”）
            this.onHasSubscribersChange && this.onHasSubscribersChange();
        }
    }
};

// 撤消订阅通道上的某个函数（guid）
Channel.prototype.unsubscribe = function(f) {
    // 事件处理函数校验
    forceFunction(f);

    // 事件处理函数的guid索引
    var guid = f.observer_guid,
    // 事件处理函数
        handler = this.handlers[guid];
    if (handler) {
        // 从该通道上撤消订阅（索引为guid）
        delete this.handlers[guid];
        // 通道上的事件处理函数的个数减1
        this.numHandlers--;
        if (this.numHandlers === 0) {
            // 撤消订阅最后一个事件时调用自定义的处理
            this.onHasSubscribersChange && this.onHasSubscribersChange();
        }
    }
};

// 调用所有被发布到该通道上的函数
Channel.prototype.fire = function(e) {
    var fail = false,
        fireArgs = Array.prototype.slice.call(arguments);

    // sticky事件被调用时，标示为已经调用过。
    if (this.state == 1) {
        this.state = 2;
        this.fireArgs = fireArgs;
    }

    if (this.numHandlers) {
        // 把该通道上的所有事件处理函数拿出来放到一个数组中。
        var toCall = [];
        for (var item in this.handlers) {
            toCall.push(this.handlers[item]);
        }
        // 依次调用通道上的所有事件处理函数
        for (var i = 0; i < toCall.length; ++i) {
            toCall[i].apply(this, fireArgs);
        }
        // sticky事件是一次性全部被调用的，调用完成后就清空。
        if (this.state == 2 && this.numHandlers) {
            this.numHandlers = 0;
            this.handlers = {};
            this.onHasSubscribersChange && this.onHasSubscribersChange();
        }
    }
};

// ④创建事件通道（publish部分）-----------------------

// （内部事件通道）页面加载后DOM解析完成
channel.createSticky('onDOMContentLoaded');

// （内部事件通道）Cordova的native准备完成
channel.createSticky('onNativeReady');

// （内部事件通道）所有Cordova的JavaScript对象被创建完成可以开始加载插件
channel.createSticky('onCordovaReady');

// （内部事件通道）所有自动load的插件js已经被加载完成
channel.createSticky('onPluginsReady');

// Cordova全部准备完成
channel.createSticky('onDeviceReady');

// 应用重新返回前台
channel.create('onResume');

// 应用暂停退到后台
channel.create('onPause');

// （内部事件通道）应用被关闭（window.onunload）
channel.createSticky('onDestroy');

// ⑤设置deviceready事件之前必须要完成的事件
// ***onNativeReady和onPluginsReady是平台初期化之前要完成的。
channel.waitForInitialization('onCordovaReady');
channel.waitForInitialization('onDOMContentLoaded');

module.exports = channel;

});
```


参考：   
[http://stackoverflow.com/questions/13512949/why-would-one-use-the-publish-subscribe-pattern-in-js-jquery](http://stackoverflow.com/questions/13512949/why-would-one-use-the-publish-subscribe-pattern-in-js-jquery)