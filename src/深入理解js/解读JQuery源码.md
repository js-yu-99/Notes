

```js
/**
 * global -> 在浏览器环境下运行（或者webview等） 它的值是window 在node环境下运行值可能是global
 * factory -> 回调函数
 */
(function (global, factory) {
    "use strict";
    // 支持CommonJS模块的导入导出规范 node  or  webpack
    if (typeof module === "object" && typeof module.exports === "object") {
        module.exports = global.document ? // global.document 存在说明是 webpack 环境  存在window
            factory(global, true) : // 基于webpack中的CommonJS ES6Module规范导入的JQ， window -> window  noGlobal -> true
            function (w) { // 在其余的没有window的环境下，导出一个函数，后期执行函数，可以传入一个window进来，则也可以正常使用，否则会报错
                if (!w.document) {
                    throw new Error("jQuery requires a window with a document");
                }
                return factory(w);
            };
    } else { // 浏览器script直接导入的 浏览器不支持CommonJS规范 window -> window  noGlobal -> undefined
        factory(global);
    }
})(typeof window !== 'undefined' ? window : this, function factory(window, noGlobal) {

    var version = "3.6.0", jQuery = function (selector, context) {
        return new jQuery.fn.init(selector, context);
    };

    // 多库共存
    var

        // Map over jQuery in case of overwrite
        _jQuery = window.jQuery,

        // Map over the $ in case of overwrite
        _$ = window.$;

    jQuery.noConflict = function (deep) {
        if (window.$ === jQuery) {
            window.$ = _$;
        }

        if (deep && window.jQuery === jQuery) {
            window.jQuery = _jQuery;
        }

        return jQuery;
    };

    // AMD
    if (typeof define === "function" && define.amd) {
        define("jquery", [], function () {
            return jQuery;
        });
    }

    // 浏览器直接引入
    if (typeof noGlobal === "undefined") {
        window.jQuery = window.$ = jQuery;
    }
    // 基于webpack处理
    return jQuery;
});
```

