# Cordova 3.x 源码分析（3） -- cordova.js 模块系统 require/define

类似于 Java 的 package/import，在 JavaScript 中也有类似的 define/require，它用来异步加载 module 化的 js，从而提高运行效率。 

- define 定义注册一个 module
- require 加载使用一个 module

模块化加载的必要性，起源于 nodejs 的出现。但是 JavaScript 并没有内置模块系统，所以就出现了很多规范。 
主要有2种：[CommonJS](http://wiki.commonjs.org/wiki/Modules/1.1) 和 [AMD（Asynchronous Module Definition）](http://en.wikipedia.org/wiki/Asynchronous_module_definition)。还有国内兴起的 [CMD（Common Module Definition）](https://github.com/cmdjs/specification/blob/master/draft/module.md) 

CommonJS 主要面对的是服务器，代表是 [Node.js](http://nodejs.org/)；AMD 针对浏览器进行了优化，主要实现 [require.js](http://www.requirejs.org/)；CMD 是 [seajs](http://seajs.org/docs/)。 

cordova-js 最开始采用的是 require.js 作者写的 [almond.js](http://github.com/jrburke/almond)（兼容 AMD 和 CommonJS），但之后由于特殊需求（比如模块不存在的时候要 throw 异常），最终从 almond.js fork 过来实现了一个简易 CommonJS 风格的模块系统，同时提供了和 nodejs 之间很好的交互。在 cordova.js 中可以直接使用 define()和 require()，在其他文件可以通过 cordova.define()和 cordova.require()来调用。所以 src/scripts/require.js 中定义的就是一个精简的 JavaScript 模块系统。 

源码如下： 

Js **代码**

```
// file: src/scripts/require.js

// 定义2个cordova.js内部使用的全局函数require/define
var require,
    define;

// 通过自调用的匿名函数来实例化全局函数require/define
(function () {
    // 全部模块
    var modules = {},
    // 正在build中的模块ID的栈
        requireStack = [],
    // 标示正在build中模块ID的Map
        inProgressModules = {},
        SEPARATOR = ".";

    // 模块build
    function build(module) {
        // 备份工厂方法
        var factory = module.factory,
        // 对require对象进行特殊处理
            localRequire = function (id) {
                var resultantId = id;
                if (id.charAt(0) === ".") {
                    resultantId = module.id.slice(0, module.id.lastIndexOf(SEPARATOR)) + SEPARATOR + id.slice(2);
                }
                return require(resultantId);
            };
        // 给模块定义一个空的exports对象，防止工厂类方法中的空引用
        module.exports = {};
        // 删除工厂方法
        delete module.factory;
        // 调用备份的工厂方法（参数必须是require,exports,module）
        factory(localRequire, module.exports, module);
        // 返回工厂方法中实现的module.exports对象
        return module.exports;
    }

    // 加载模块
    require = function (id) {
        // 如果模块不存在抛出异常
        if (!modules[id]) {
            throw "module " + id + " not found";
        // 如果模块正在build中抛出异常
        } else if (id in inProgressModules) {
            var cycle = requireStack.slice(inProgressModules[id]).join('->') + '->' + id;
            throw "Cycle in require graph: " + cycle;
        }
        // 如果模块存在工厂方法说明还未进行build（require嵌套）
        if (modules[id].factory) {
            try {
                // 标示该模块正在build
                inProgressModules[id] = requireStack.length;
                // 将该模块压入请求栈
                requireStack.push(id);
                // 模块build，成功后返回module.exports
                return build(modules[id]);
            } finally {
                // build完成后删除当前请求
                delete inProgressModules[id];
                requireStack.pop();
            }
        }
        // build完的模块直接返回module.exports
        return modules[id].exports;
    };

    // 定义模块
    define = function (id, factory) {
        // 如果已经存在抛出异常
        if (modules[id]) {
            throw "module " + id + " already defined";
        }
        // 模块以ID为索引包含ID和工厂方法
        modules[id] = {
            id: id,
            factory: factory
        };
    };

    // 移除模块
    define.remove = function (id) {
        delete modules[id];
    };

    // 返回所有模块
    define.moduleMap = modules;
})();

// 如果处于nodejs环境的话，把require/define暴露给外部
if (typeof module === "object" && typeof require === "function") {
    module.exports.require = require;
    module.exports.define = define;
}
```

其中 factory(localRequire, module.exports, module);  
第一个参数“localRequire”实质还是调用全局的 require()函数，只是把 ID 稍微加工了一下支持相对路径。cordova.js 没有用到相对路径的 require，但在一些 Plugin 的 js 中有，比如 Contact.js 中 ContactError = require('./ContactError'); 

不知道什么原因要把 module.exports 单做为工厂方法的第两个参数，cordova.js 中除了个别地方用到第二个参数（base64.js、uilder.js、modulemapper.js、pluginloader.js、utils.js），大部分都是在用第三个参数在工厂方法里调用 module.exports。Plugin 的 js 代码中也基本是 module.exports。 

参考：   
[https://github.com/seajs/seajs/issues/588](https://github.com/seajs/seajs/issues/588)   
[http://phonegap.com/2012/03/21/introducing-cordova-js/](http://phonegap.com/2012/03/21/introducing-cordova-js/)