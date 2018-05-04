seajs 拜读
============

**Sea.js 3.0.1**

## IIFE ( Immediately-Invoked Function Expression )的使用

```javascript
;(function(global, undefined)) {
  
})(this);
```

其中有个undefined参数，这是为了避免 ES5 之前 `undefined` 和 `NaN` 并未在标准中指明是个常量，在 IE8 中是可以覆盖的。如果在 IE8 中使用 `undefined = false` ，那之后对 `undefined` 的使用很可能受到影响。如果采用代码中所示的做法， `undefined` 的值由于参数未传，仍为 `undefined` 。

另外这也是闭包的使用。

## 避免重复导入

```javascript
if (global.seajs) {
  return ;
}
```

在执行前进行检测 seajs 是否存在，如果存在，则不执行，能避免重复加载

## 声明并赋值seajs全局变量

```javascript
var seajs = gloabl.seajs = {
  version: '3.0.1'
}

var data = seajs.data = { };
```

声明了 seajs ，赋值了两个属性： `version` 和 `data` 。

## lang 模块 util-lang.js

这是一个简单的语言增强的模块。

主要是添加了类型判断函数。做法是通过调用 `Object.prototype.toString` 。其中使用了 curring 技术。curry 函数如下：

```javascript
function isType(type) {
  return function(obj) {
    return { }.toString.call(obj) == '[object ' + type + ']';
  }
}
```

然后可以通过以下方式建立一个个的类型判断函数。

```javascript
var isObject = isType('Object');
var isString = isType('String');
var isArray  = Array.isArray || isType('Array');
var isFunction = isType('Function');
var isUndefined = isType('Undefined');
```

这些判断都很简单，没有像一些库函数，会将 `NaN` 之类的提出来，但是足够让 seajs 使用。

在 lang 模块中，seajs 还定义了一个生成 id 的函数，并且定义了一个 `_cid` 。

```javascript
var _cid = 0;
function cid() {
  return _cid++;
}
```

具体使用在哪之后补全。

## event 模块 util-event.js

event 模块是 seajs 内部使用的一个小的事件模块支持。

首先，通过 `var events = data.events = { };` 在 `seajs.data` 中创建了 `events` 命名空间。然后为 `seajs` 对象添加事件函数支持。主要支持 `seajs.on` 、 `seajs.off` 、 `seajs.emit` 这三个函数。因为是 seajs 自身使用，所以实现起来很简单，下面一一进行分析。

### seajs.on 绑定事件

具体代码如下：

```javascript
seajs.on = function(name, callback) {
  var list = events[name] || (events[name] = [ ]);
  list.push(callback);
  return seajs;
}
```

绑定事件的代码很简单，接收事件名和回调两个参数。 `events` 对象中的结构是这样的： `events = { evt_name: [ callback_array ], evt_name_2: [ callback_array_2 ] };` 。如果之前没有注册过该事件，则会将 `name` 作为属性加在 `events` 中。最后将 `callback` 压入回调函数数组中即可。

### seajs.off 解绑事件

具体代码如下：

```javascript
seajs.off = function(name, callback) {
  if (!(name || callback)) {
    events = data.events = { };
  }

  var list = events[name];

  if (list) {
    if (callback) {
      for (var i = list.length - 1; i >= 0; --i) {
        if (list[i] === callback) {
          list.splice(i, 1);
        }
      }
    }
    else {
      delete events[name];
    }
  }

  return seajs;
}
```

解绑事件的代码也很清晰易懂。主要分为以下三种情况：
- 当未传参数时，会解绑所有事件。（但是从代码看，其实如果传的都是 == 假值，也会全部解绑。）
- 当传了 `name` ，未传 `callback` 时，会解绑 `name` 的所有事件。
- 当传了 `name` 和 `callback` 时，会在 `name` 的 `callback_array` 中查找 `callback` ，找到了就删除.

### seajs.emit 触发事件

所谓触发一个事件，也就是发出事件，seajs 根据事件名调用相应的函数，其中参数为相应的数据。seajs.emit 函数代码如下：

```javascript
var emit = seajs.emit = function(name, data) {
  var list = events[name];

  if (list) {
    list = list.slice();

    for (var i = 0, len = list.length; i < len; ++i) {
      list[i](data);
    }
  }

  return seajs;
}
```

整个函数中， `list = list.slice();` 操作执行的是一个浅复制，防止对原有 `callback_array` 进行了更改。

### event 模块总结

event 模块总共定义了 `seajs.on` 、 `seajs.off` 、 `seajs.emit` 三个函数，每个函数的返回值都是 `seajs` 对象，可以执行链式操作。总的还是那句话， event 模块相对实现的比较简单。稍微齐全一点的可以看 Eventer 模块（自己造的轮子，懒得搞了），另外就是 eventproxy 模块。

## path 模块 util-path.js

path 模块是对 id、uri 进行处理的模块。毫无疑问，肯定一堆正则。函数也比较多一点。这个慢慢的补坑。

### 提取一个路径的文件夹部分

代码如下：

```javascript
var DIRNAME_RE = /[^?#]*\//;

function dirname(path) {
  return path.match(DIANAME_RE)[0];
}
```

`DIRNAME_RE` 是为了鉴别非 `?t=123` 部分和 `#xx/zz` 部分的正则。 `String.prototype.match` 返回的是一个数组， `dirname` 返回匹配的部分，也即 `match` 后的数组的第一项。

### 获取 uri 的真实路径

由于可能会传一些类似于 `"http://test.com/a//./b/../c"` 这样的 uri，需要对其进行规范化(Canoicalize a path)。代码如下：

```javascript
var DOT_RE = /\/\.\//g;
var DOUBLE_DOT_RE = /\/[^/]+\/\.\.\//;
var MULTI_SLASH_RE = /([^:/])\/+\//g;

// Canonicalize a path
// realpath("http://test.com/a//./b/../c") ==> "http://test.com/a/c"
function realpath(path) {
  // /a/b/./c/./d ==> /a/b/c/d
  path = path.replace(DOT_RE, '/');

  /*
    @author wh1100717
    a//b/c ==> a/b/c
    a///b/////c ==> a/b/c
    DOUBLE_DOT_RE matches a/b/c//../d path correctly only if replace // with / first
  */
  path = path.replace(MULTI_SLASH_RE, '$1/');

  // a/b/c/../../d  ==>  a/b/../d  ==>  a/d
  while(path.match(DOUBLE_DOT_RE)) {
    path = path.replace(DOUBLE_DOT_RE, '/');
  }

  return path;
}
```

 `DOT_RE` 是为了匹配 `/./` 类似于这种模式的字符串， `DOUBLE_DOT_RE` 是为了匹配类似于 `/abb/../` 这种模式的字符串， `MULTI_SLASH_RE` 是为了匹配类似于 `abc//////a/` 这种模式的字符串。

在 `realpath` 函数中，首先进行单个 dot 的替换，然后进行多 `/` 的替换，最后一个个的替换掉两个 dot。最终返回 `path` 。

### 标准化 id（Normalize an id）

 `normalize` 函数代码如下：

```javascript
function normalize(path) {
  var last = path.length - 1;
  var lastC = path.charCodeAt(last);

  // If the uri ends with `#`, just return it without '#'
  if (lastC === 35 /* "#" */) {
    return path.substring(0, last);
  }

  return (path.substring(last - 2) === ".js" ||
      path.indexOf("?") > 0 ||
      lastC === 47 /* "/" */) ? path : path + ".js";
}
```

在 `normalize` 中主要包含以下三种情况：
- 当最后一个字符是 `#` 时，就直接返回不带 `#` 的 path
- 当 path 以 `.js` 结尾或者 path 中有 `?` 字符或者最后一个字符是 `/` 时，直接返回 path
- 否则，返回 `path + '.js'`

### path 和 alias 处理

在seajs的使用中，我们可以配置 path 和 alias，来让使用更加便捷。但是 seajs 的处理，反正又是正则咯。代码如下：

```javascript
var PATHS_RE = /^([^/:]+)(\/.+)$/;
var VARS_RE = /{([^{]+)}/g;

/**
 * 将 alias 解析
 * seajs.config({
 *   alias: {
 *     'jquery': 'path/to/jquery.js',
 *   }
 * });
 * require('jquery') => require('path/to/jquery.js');
 */
function parseAlias(id) {
  var alias = data.alias;
  return alias && isString(alias[id]) ? alias[id] : id;
}

/**
 * 将 path 解析
 * seajs.config({
 *   paths: {
 *     'p': 'path/to/p',
 *   }
 * });
 * require('p/jquery') => require('path/to/p/jquery.js');
 */
function parsePaths(id) {
  var paths = data.paths;
  var m;

  if (paths && (m = id.match(PATHS_RE)) && isString(paths[m[1])) {
    id = paths[m[1]] + m[2];
  }

  return id;
}

/**
 * 将 var 解析
 * seajs.config({
 *   vars: {
 *     'locale': 'zh-cn'
 *   }
 * });
 * require('./i18n/{locale}.js') => require('./i18n/zh-cn.js');
 */
function parseVars(id) {
  var vars = data.vars;

  if (vars && id.indexOf("{") > -1) {
    id = id.replace(VARS_RE, function(m, key) {
      return isString(vars[key]) ? vars[key] : m;
    });
  }

  return id;
}

/**
 * 将 map 解析
 * seajs.config({
 *   map: [
 *     [ '.js', '-debug.js' ]
 *   ]
 * });
 * require('./a.js') => require('./a-debug.js');
 */
function parseMap(id) {
  var map = data.map;
  var ret = uri;

  if (map) {
    for (var i = 0, len = map.length; i < len; ++i) {
      var rule = map[i];

      ret = isFunction(rule) ?
            (rule(uri) || uri) :
            uri.replace(rule[0], rule[1]);

      // Only apply the first matched rule
      // 如果匹配了，除非 String(rule[0]) === String(rule[1])，否则不会是相同的
      if (ret !== uri) {
        break;
      }
    }
  }
}
```

上面的主要是对 seajs.config 中的 `alias` 、 `paths` 、 `vars` 、 `map` 参数进行解析。代码不是很难，代码中我也添加了若干注释。
可以比较明了的看懂。

### more about path

这一块是关于路径的，具体先看代码吧。

```javascript
// 绝对路径检测 '//a.b.com' or ':/'
var ABSOLUTE_RE = /^\/\/.|:\//;
// .*? 最小可能匹配
// 'a//b/c//d/e'.match(ROOT_DIR_RE) => ['a//b/']
// 如果没有 ?，那就是 ['a//b/c//d/']
var ROOT_DIR_RE = /^.*?\/\/.*?\//;

function addBase(id, refUri) {
  var ret;
  var first = id.charCodeAt(0);

  // Absolute
  if (ABSOLUTE_RE.test(id)) {
    ret = id;
  }
  // Relative
  else if (first === 46 /* "." */) {
    ret = (refUri ? dirname(refUri) : data.cwd) + id;
  }
  // Root
  else if (first === 47 /* "/" */) {
    var m = data.cwd.match(ROOT_DIR_RE);
    ret = m ? m[0] + id.substring(1) : id;
  }
  // Top-level
  else {
    ret = data.base + id;
  }

  // Add default protocal when uri begins with "//"
  // 当以'//'开头
  if (ret.indexOf("//") === 0) {
    ret = location.protocol + ret;
  }

  return realpath(ret);
}

function id2Uri(id, refUri) {
  if (!id) {
    return "";
  }
  // 我只想说- -写的好有条理性
  // 之所以多次调用 `parseAlias(id)` 是防止层层转换的过程中，存在 alias
  id = parseAlias(id);
  id = parsePaths(id);
  id = parseAlias(id);
  id = parseVars(id);
  id = parseAlias(id);
  id = normalize(id);
  id = parseAlias(id);

  var uri = addBase(id, refUri);
  uri = parseAlias(uri);
  uri = parseMap(uri);

  return uri;
}

// 为了开发者分析，seajs 增加了一个 `resolve` 接口
seajs.resolve = id2Uri;
```

不造怎么去写了o(╯□╰)o，先放着吧。

### 环境检测等

代码如下：

```
// Check environment
// 监测是不是在 web worker 环境，有点不懂为什么不 `typeof importScripts === 'function'`
var isWebWorker = typeof window === 'undefined' && typeof importScripts !== 'undefined' && isFunction(importScripts);

// Ignore about:xxx and blob:xxx
var IGNORE_LOCATION_RE = /^(about|blob):/;

//
var loaderDir
// Sea.js's full path
var loaderPath
// Location is read-only from web worker, should be ok though
var cwd = (!location.href || IGNORE_LOCATION_RE.test(location.href)) ? '' : dirname(location.href);

if (isWebWorker) {
  // 这个我不想看了= =反正我不会用在这个里面
  // Web worker doesn't create DOM object when loading scripts
  // Get sea.js's path by stack trace.
  var stack
  try {
    var up = new Error()
    throw up
  } catch (e) {
    // IE won't set Error.stack until thrown
    stack = e.stack.split('\n')
  }
  // First line is 'Error'
  stack.shift()

  var m
  // Try match `url:row:col` from stack trace line. Known formats:
  // Chrome:  '    at http://localhost:8000/script/sea-worker-debug.js:294:25'
  // FireFox: '@http://localhost:8000/script/sea-worker-debug.js:1082:1'
  // IE11:    '   at Anonymous function (http://localhost:8000/script/sea-worker-debug.js:295:5)'
  // Don't care about older browsers since web worker is an HTML5 feature
  var TRACE_RE = /.*?((?:http|https|file)(?::\/{2}[\w]+)(?:[\/|\.]?)(?:[^\s"]*)).*?/i
  // Try match `url` (Note: in IE there will be a tailing ')')
  var URL_RE = /(.*?):\d+:\d+\)?$/
  // Find url of from stack trace.
  // Cannot simply read the first one because sometimes we will get:
  // Error
  //  at Error (native) <- Here's your problem
  //  at http://localhost:8000/_site/dist/sea.js:2:4334 <- What we want
  //  at http://localhost:8000/_site/dist/sea.js:2:8386
  //  at http://localhost:8000/_site/tests/specs/web-worker/worker.js:3:1
  while (stack.length > 0) {
    var top = stack.shift()
    m = TRACE_RE.exec(top)
    if (m != null) {
      break
    }
  }
  var url
  if (m != null) {
    // Remove line number and column number
    // No need to check, can't be wrong at this point
    var url = URL_RE.exec(m[1])[1]
  }
  // Set
  loaderPath = url
  // Set loaderDir
  loaderDir = dirname(url || cwd)
  // This happens with inline worker.
  // When entrance script's location.href is a blob url,
  // cwd will not be available.
  // Fall back to loaderDir.
  if (cwd === '') {
    cwd = loaderDir
  }
}
else {
  var doc = document
  var scripts = doc.scripts

  // Recommend to add `seajsnode` id for the `sea.js` script element
  // 源码中是这样的话，那可能产生 bug 诶 哈哈，下次要给 script 加一个 "seajsnode" 的 id
  var loaderScript = doc.getElementById("seajsnode") ||
    scripts[scripts.length - 1]

  // 获取 src 的值
  function getScriptAbsoluteSrc(node) {
    return node.hasAttribute ? // non-IE6/7
      node.src :
      // see http://msdn.microsoft.com/en-us/library/ms536429(VS.85).aspx
      node.getAttribute("src", 4)
  }
  loaderPath = getScriptAbsoluteSrc(loaderScript)
  // When `sea.js` is inline, set loaderDir to current working directory
  loaderDir = dirname(loaderPath || cwd)
}
```

util-path.js 暂时就先这样了，之后有补充再补充。

## request 模块 util-request.js

request 模块是为了请求 scripts 和 stylesheets 的

```
if (isWebWorker) {
  // 
}
else {
  // 
}
```

下面分开分析。

### web worker env.

在 web worker 环境中，有 `importScripts` 方法。seajs 对其再封装了一层，代码如下。

```
function requestFromWebWorker(url, callback, charset, crossorigin) {
  // Load with importScripts
    var error;
    try {
      importScripts(url);
    } catch (e) {
      error = e;
    }
    callback(error);
  }
  // For Developers
  seajs.request = requestFromWebWorker;
}
```

因为有 importScripts 函数，所以 requestFromWebWorker 很简单。
注意 callback 的处理，这个大概是受 node 的风格影响，第一个参数传入的是 err。
所以我们在传入 callback 的时候，需要判断 ` if (err) { } `。

### 非 web worker env.

代码如下。

```
{
  var doc = document;
  var head = doc.head || doc.getElementsByTagName('head')[0] || doc.documentElement;

  // HTML 的 base 标签
  var baseElement = head.getLementsByTagName('base')[0];

  // 下面这个是为了在 IE6-8 中阻止 immediately load，不过我没看懂
  var currentlyAddingScript;

  // request function，用来请求 script 的，和上面的 requestFromWebWorker 类似
  function request(url, callback, charset, crossorigin) {
    var node = doc.createElement('script');

    if (charset) {
      node.charset = charset;
    }

    if (!isUndefined(crossorigin)) {
      node.setAttribute('crossorigin', crossorigin);
    }
    // 为 node 添加加载完成的 callback
    addOnload(node, callback, url);

    // 异步加载
    node.async = true;
    node.src = url;

    // 这个.....我真没看懂
    // For some cache cases in IE 6-8, the script executes IMMEDIATELY after
    // the end of the insert execution, so use `currentlyAddingScript` to
    // hold current node, for deriving url in `define` call
    currentlyAddingScript = node

    // ref: #185 & http://dev.jquery.com/ticket/2709
    baseElement ?
        head.insertBefore(node, baseElement) :
        head.appendChild(node)

    currentlyAddingScript = null
  }

  // 为 node 增加 回调函数，包括 onload 和 onerror 的
  function addOnload(node, callback, url) {
    var supportOnload = 'onload' in node;

    if (supportOnload) {
      node.onload = onload;
      node.onerror = function() {
        emit("error", { uri: url, node: node });
        // 参数为 true 代表有错误
        onload(true);
      }
    }
    // 也有不支持 onload 的，貌似是 IE 较低版本，具体我忘了
    else {
      // 在这些浏览器中，只能采用 onreadystatechange 来设置。每一次的 node.readState
      // 的改变，都会调用这个函数，所以可以通过在 onreadystatechange 中设置。
      node.onreadystatechange = function() {
        if (/loadede|complete/.test(node.readyState)) {
          onload();
        }
      }
    }

    // onload 是给 callback 再包装了一层，主要是确保 onload 或者 onerror 执行过之后
    // 就赋值其它的几个事件监听器为 null，这样就减少调用了。还能防止 IE 的 memory leak
    // 貌似主要是 IE 的问题才导致需要这样做。
    function onload(error) {
      // Ensure only run once and handle memory leak in IE
      node.onload = node.onerror = node.onreadystatechange = null

      // Remove the script to reduce memory leak
      if (!data.debug) {
        head.removeChild(node);
      }

      // Dereference the node
      node = null;

      callback(error);
    }
  }

  // For Developers
  seajs.request = request;
}
```

这一部分的代码在代码中注释已经解释清楚了。就不多说了。

### interactiveScript

这一部分是为了获取当前执行的脚本。

```javascript
var interactiveScript

function getCurrentScript() {
  if (currentlyAddingScript) {
    return currentlyAddingScript
  }

  // For IE6-9 browsers, the script onload event may not fire right
  // after the script is evaluated. Kris Zyp found that it
  // could query the script nodes and the one that is in "interactive"
  // mode indicates the current script
  // ref: http://goo.gl/JHfFW
  if (interactiveScript && interactiveScript.readyState === "interactive") {
    return interactiveScript
  }

  var scripts = head.getElementsByTagName("script")

  for (var i = scripts.length - 1; i >= 0; i--) {
    var script = scripts[i]
    if (script.readyState === "interactive") {
      interactiveScript = script
      return interactiveScript
    }
  }
}
```

## deps模块 - util-deps.js

deps模块是用于解析依赖的。
该模块中就一个函数，也就是 `parseDependencies` 。
代码和分析见下。

```javascript
/**
 * 该函数我看的感觉有点怪怪的，主要是 Parse dependencies according to the module factory code。
 * 使用的时候，传入参数为 `factory.toString()`，等看完再回头想吧
 * 另外，整个 seajs-debug.js 中，有一些算是 anti-pattern 的做法，可能是习惯问题吧
 */
function parseDependencies(s) {
  // 代码中首先检查是不是在字符串中有`require`，如果有，那 deps-array 就置为空的。
  if(s.indexOf('require') == -1) {
    return [];
  }
  // 没有`require`的处理
  // 变量声明
  // index <==> 遍历字符串中字符的索引
  // peek  <==> 当前字符
  // length <==> 字符串的长度
  // isReg <==>
  // modName <==>
  // res <==> 返回的依赖数组
  var index = 0, peek, length = s.length,
      isReg = 1, modName = 0, res = [];
  // 圆括号
  var parentheseState = 0, parentheseStack = [];
  // 大括号
  var braceState, barceStack = [];
  var isReturn;
  // 循环检测
  while(index < length) {
    // 遍历字符
	readch();
	// 字符检测
	// 如果是空白字符 且 在 return 状态 同时 当前字符时 `\n` 或者 `\r` 时，大括号和 return 的状态重置
	if (isBlank()) {
	  if (isReturn && (peek == '\n' || peek == '\r')) {
		braceState = 0;
		isReturn = 0;
	  }
	}
	// 如果是引号
	else if (isQuote()) {
	  dealQuote();
	  isReg = 1;
	  isReturn = 0;
	  braceState = 0;
	}
	// 如果是 `/`，也就是正则或者注释的情况
	else if (peek == '/') {
	  readch();
	  if (peek == '/') {
		index = s.indexOf('\n', index);
		if (index == -1) {
		  index = s.length;
		}
	  }
	  else if (peek == '*') {
		var i = s.indexOf('\n', index;
		var index = s.indexOf('*/, index);
		if (index == -1) {
		  index = length;
		}
		else {
		  index += 2;
		}
		if (isReturn && i != -1 && i < index ) {
		  braceState = 0;
		  isReturn = 0;
		}
	  }
	  else if (isReg) {

	  }
	  else {
		
	  }
	}
	else if (isWord()) {
	
	}
	else if (isNumber()) {
	}
	else if (peek == '(') {
	}
	else if (peek == ')') {
	}
	else if (peek == '{') {
	}
	else if (peek == '}') {
	}
	else {
	}
  }
  // 返回
  return res;
  // 在 seajs-debug.js 源文件中，下面定义了若干个 util 函数。
  // 下面我领出来一个一个分析。
}
```

### deps-util-readch 函数

该函数代码及注释如下。

```javascript
function readch() {
  peek = s.charAt(index++);
}
```

### deps-util-isBlank 函数

该函数代码及注释如下。

```javascript
function isBlank() {
  return /\s/.test(peek);
}
```

### deps-util-isQuote 函数

该函数代码及注释如下。

```javascript
function isQuote() {
  return peek === '"' || peek === "'";
}
```

### deps-util-dealQuote 函数

该函数代码及注释如下。

```javascript
function dealQuote() {
  var start = index;
  var c = peek;
  var end = s.indexOf(c, start);
  // 如果没有找到
}
```

不行了，我看不下这个函数里的玩意了。。先放着。。反正知道是做啥的。


## module模块 module.ls

该模块是 模块加载器 的核心（The core of module loader）。

首先定义了一些辅助变量。代码如下。

```javascript
var cachedMods = seajs.cache = { };
var anonymousMeta;

var fetchingList = { };
var fetchedList = { };
var callbackList = { };

// Module 是因为声明提升 所以能访问到。
var STATUS = Module.STATUS = {
  // 1 - The `module.uri` is being fetched
  FETCHING: 1,
  // 2 - The meta data has been saved to cachedMods
  SAVED: 2,
  // 3 - The `module.dependencies` are being loaded
  LOADING: 3,
  // 4 - The module are ready to execute
  LOADED: 4,
  // 5 - The module is being executed
  EXECUTING: 5,
  // 6 - The `module.exports` is available
  EXECUTED: 6,
  // 7 - 404
  ERROR: 7
}
```

然后定义了模块构造函数。

```javascript
function Module(uri, deps) {
  this.uri = uri;
  this.dependencies = deps || [ ];
  // Ref the dependence modules
  this.deps = { };
  this.status = 0;

  this._entry = [ ];
}
```

### 模块决议 Module.prototype.resolve

```javascript
/**
 * 对依赖进行决议，解析成 uris 数组
 */
Module.prototype.resolve = function() {
  var mod = this;
  var ids = mod.dependencies;
  var uris = [ ];

  for (var i = 0, len = ids.length; i < len; i++) {
    uris[i] = Module.resolve(ids[i], mod.uri);
  }

  return uris;
}
```

### 模块pass（不造怎么翻译） Module.prototype.pass

```javascript
Module.prototype.pass = function() {
  var mod = this

  var len = mod.dependencies.length

  for (var i = 0; i < mod._entry.length; i++) {
    var entry = mod._entry[i]
    var count = 0
    for (var j = 0; j < len; j++) {
      var m = mod.deps[mod.dependencies[j]]
      // If the module is unload and unused in the entry, pass entry to it
      if (m.status < STATUS.LOADED && !entry.history.hasOwnProperty(m.uri)) {
        entry.history[m.uri] = true
        count++
        m._entry.push(entry)
        if(m.status === STATUS.LOADING) {
          m.pass()
        }
      }
    }
    // If has passed the entry to it's dependencies, modify the entry's count and del it in the module
    if (count > 0) {
      entry.remain += count - 1
      mod._entry.shift()
      i--
    }
  }
}
```

### 模块加载 Module.prototype.load

```javascript
/* 加载依赖，当所有加载完后，开始 onload */
Module.prototype.load = function() {
  var mod = this;

  // 如果已经加载好了，就只用等Onload call
  if (mod.status >= STATUS.LOADING) {
    return ;
  }

  mod.status = STATUS.LOADING;

  // 触发 `load` 事件（也服务于插件）
  emit('load', uris);

  for (var i = 0, len = uris.length; i < len; i++) {
    mod.deps[mod.dependencies[i]] = Module.get(uris[i]);
  }

  mod.pass();

  // 如果模块还有entries没有pass掉，调用 onload
  if (mod._entry.length) {
    mod.onload();
    return ;
  }

  // 开始并行加载
  var requestCache = { };
  var m;

  for (i = 0; i < len; i++) {
    m = cachedMods[uris[i]];

    if (m.status < STATUS.FETCHING) {
      m.fetch(requestCache);
    }
    else if (m.status === STATUS.SAVED) {
      m.load();
    }
  }

  // 发送所有请求，避免IE6-9的bug。Issues#808
  for (var requestUri in requestCache) {
    if (requestCache.hasOwnProperty(requestUri)) {
      requestCache[requestUri]()
    }
  }
}
```

### 模块加载完成后的回调 Module.prototype.onload

```javascript
Module.prototype.onload = function() {
  var mod = this;
  mod.status = STATUS.LOADED;

  // 执行entry的回调
  // When sometimes cached in IE, exec will occur before onload, make sure len is an number
  for (var i = 0, len = (mod._entry || []).length; i < len; i++) {
    var entry = mod._entry[i]
    if (--entry.remain === 0) {
      entry.callback()
    }
  }

  delete mod._entry
}
```

### 模块加载404后的回调 Module.prototype.error

```javascript
Module.prototype.error = function() {
  var mod = this;
  mid.onload();
  mod.status = STATUS.ERROR;
}
```

### 模块执行 Module.prototype.exec

```javascript
Module.prototype.exec = function () {
  var mod = this

  // 避免重复执行
  // When module is executed, DO NOT execute it again. When module
  // is being executed, just return `module.exports` too, for avoiding
  // circularly calling
  if (mod.status >= STATUS.EXECUTING) {
    return mod.exports;
  }

  mod.status = STATUS.EXECUTING

  if (mod._entry && !mod._entry.length) {
    delete mod._entry;
  }

  // non-cmd module has no property factory and exports
  if (!mod.hasOwnProperty('factory')) {
    mod.non = true;
    return ;
  }

  // Create require
  var uri = mod.uri;

  function require(id) {
    var m = mod.deps[id] || Module.get(require.resolve(id));
    if (m.status == STATUS.ERROR) {
      throw new Error('module was broken: ' + m.uri);
    }
    return m.exec();
  }

  require.resolve = function(id) {
    return Module.resolve(id, uri);
  }

  require.async = function(ids, callback) {
    Module.use(ids, callback, uri + "_async_" + cid());
    return require;
  }

  // Exec factory
  var factory = mod.factory

  // 由此可见，其实`factory`的参数就是 require, mod.exports, mod 这三个，虽然我们用的时候并非这样
  var exports = isFunction(factory) ?
    factory.call(mod.exports = { }, require, mod.exports, mod) :
    factory

  if (exports === undefined) {
    exports = mod.exports
  }

  // Reduce memory leak
  delete mod.factory

  mod.exports = exports
  mod.status = STATUS.EXECUTED

  // Emit `exec` event
  emit("exec", mod)

  return mod.exports
}
```

### fetch模块 Module.prototype.fetch

```javascript
Module.prototype.fetch = function(requestCache) {
  var mod = this;
  var uri = mod.uri;

  mod.status = STATUS.FETCHING;

  // 触发 `fetch` 事件
  var emitData = { uri: uri };
  emit('fetch', emitData);
  var requestUri = emitData.requestUri || uri;

  // 如果是空的uri 或者 非cmd模块（但是这个没有体现出来）
  if (!requestUri || fetchedList.hasOwnProperty(requestUri)) {
    mod.load();
    return ;
  }

  // 如果在正在 fetch 队列中
  if (fetchingList.hasOwnProperty(requestUri)) {
    callbackList[requestUri].push(mod);
    return ;
  }

  fetchingList[requestUri] = true;
  callbackList[requestUri] = [ mod ];

  // 触发 `request` 事件
  emit('request', emitData = {
    uri: uri,
    requestUri: requestUri,
    onRequest: onRequest,
    charset: isFunction(data.charset) ? data.charset(requestUri) : data.charset,
    crossorigin: isFunction(data.crossorigin) ? data.crossorigin(requestUri) : data.crossorigin
  });

  if (!emitData.requested) {
    requestCache ?
      requestCache[emitData.requestUri] = sendRequest :
      sendRequest()
  }

  // 发送请求
  function sendRequest() {
    seajs.request(emitData.requestUri, emitData.onRequest, emitData.charset, emitData.crossorigin)
  }

  // 请求成功后的回调
  function onRequest(error) {
    delete fetchingList[requestUri]
    fetchedList[requestUri] = true

    // Save meta data of anonymous module
    if (anonymousMeta) {
      Module.save(uri, anonymousMeta)
      anonymousMeta = null
    }

    // Call callbacks
    var m, mods = callbackList[requestUri]
    delete callbackList[requestUri]
    while ((m = mods.shift())) {
      // When 404 occurs, the params error will be true
      if(error === true) {
        m.error()
      }
      else {
        m.load()
      }
    }
  }
}
```

### 决议模块 Module.prototype.resolve

```javascript
// Resolve id to uri
Module.resolve = function(id, refUri) {
  // Emit `resolve` event for plugins such as text plugin
  var emitData = { id: id, refUri: refUri }
  emit("resolve", emitData)

  return emitData.uri || seajs.resolve(emitData.id, refUri)
}
```

### 模块定义 Module.define

```javascript
Module.define = function(id, deps, factory) {
  var argsLen = arguments.length;

  // define(factory) 形式
  if (argsLen === 1) {
    factory = id;
    id = undefined;
  }
  else if (argsLen === 2) {
    factory = deps;

    // define(deps, factory)形式
    if (isArray(id)) {
      deps = id;
      id = undefined;
    }
    // define(id, factory)
    else {
      deps = undefined;
    }
  }

  // 根据模块 factory 代码解析依赖
  if (! isArray(deps) && isFunction(factory)) {
    deps = typeof parseDependencies === 'undefined' ? [ ] : parseDependencies(factory.toString);
  }

  var meta = {
    id: id,
    uri: Module.resolve(id),
    deps: deps,
    factory: factory
  };

  // Try to derive uri in IE6-9 for anonymous modules
  if (!isWebWorker && !meta.uri && doc.attachEvent && typeof getCurrentScript !== "undefined") {
    var script = getCurrentScript()

    if (script) {
      meta.uri = script.src
    }

    // NOTE: If the id-deriving methods above is failed, then falls back
    // to use onload event to get the uri
  }

  // Emit `define` event, used in nocache plugin, seajs node version etc
  emit("define", meta)

  meta.uri ? Module.save(meta.uri, meta) :
    // Save information for "saving" work in the script onload event
    anonymousMeta = meta
}
```

### 模块保存 Module.save

```javascript
// 保存 元数据 为 cachedMods，也即缓存
Module.save = function(uri, meta) {
  var mod = Module.get(uri);

  // 注意不要覆盖已经保存的模块
  if (mod.status < STATUS.SAVED) {
    mod.id = meta.id || uri;
    mod.dependencies = meta.deps || [ ];
    mod.factory = meta.factory;
    mod.status = STATUS.SAVED;

    emit('save', mod);
  }
}
```

### 获取一个模块 Module.get

```javascript
/**
 * 获取一个模块
 * 如果有缓存，就直接从缓存中获取，如果没有，就新建一个，并存入`cachedMods`中
 */
Module.get = function(uri, deps) {
  return cachedMods[uri] || (cachedMods[uri] = new Module(uri, deps));
}
```

### 使用一个模块 Module.use

```javascript
/**
 * ids 可能是数组，也可能不是，如果是数组，就use 多个模块
 * callback 执行之后的回调
 */
Module.use = function(ids, callback, uri) {
  var mod = Module.get(uri, isArray(ids) ? ids : [ids]);

  mod._entry.push(mod);
  mod.history = { };
  mod.remain = 1;

  mod.callback = function() {
    var exports = [ ];
    var uris = mod.resolve();

    for (var i = 0, len = uris.length; i < len; i++) {
      exports[i] = cachedMods[uris[i]].exec();
    }

    if (callback) {
      callback.apply(global, exports);
    }

    delete mod.callback;
    delete mod.history;
    delete mod.remain;
    delete mod._entry;
  }

  mod.load();
}
```

## Public API

### seajs.use

```javascript
seajs.use = function(isd, callback) {
  Module.use(ids, callback, data.cwd + '__use__' + cid());
  return seajs;
}

Module.define.cmd = { };

glbal.define = Module.define;

// 为开发者方便而导出的 public api
seajs.Module = Module;
data.fetchedList = fetchedList;
data.cid = cid;

seajs.require = function(id) {
  var mod = Module.get(Module.resolve(id));
  if (mod.status < STATUS.EXECUTING) {
    mod.onload();
    mod.exec();
  }
  return mod.exports;
}
```

## Config - 模块加载器的配置

这个代码就挺简单了，可以学习下对于配置项的处理。
对配置项分为 Object 和 Array 以及 'base' 还有其它 这 ** 4 ** 种情况。
最后`return seajs;` chain operation

```javascript
// 一些初始配置
// The root path to use for id2uri parsing
data.base = loaderDir;

// The loader directory
data.dir = loaderDir;

// The loader's full path
data.loader = loaderPath;

// The current working directory
data.cwd = cwd;

// The charset for requesting files
data.charset = "utf-8";

// @Retention(RetentionPolicy.SOURCE)
// The CORS options, Don't set CORS on default.
//
//data.crossorigin = undefined

// data.alias - An object containing shorthands of module id
// data.paths - An object containing path shorthands in module id
// data.vars - The {xxx} variables in module id
// data.map - An array containing rules to map module uri
// data.debug - Debug mode. The default value is false

seajs.config = function(configData) {

  for (var key in configData) {
    var curr = configData[key];
    var prev = data[key];

    // Merge object config such as alias, vars
    if (prev && isObject(prev)) {
      for (var k in curr) {
        prev[k] = curr[k];
      }
    }
    else {
      // Concat array config such as map
      if (isArray(prev)) {
        curr = prev.concat(curr);
      }
      // Make sure that `data.base` is an absolute path
      else if (key === "base") {
        // Make sure end with "/"
        if (curr.slice(-1) !== "/") {
          curr += "/";
        }
        curr = addBase(curr);
      }

      // Set config
      data[key] = curr;
    }
  }

  emit("config", configData);
  return seajs;
}
```

## Flow

之后按照使用的流程进行划分。

Need to fill.

> Module.use() => Module.get() => Module.prototype.load() =>
    Module.prototype.resolve() => Module.resolve() =>
    seajs.resolve() =>Module.prototype.pass() =>
    Module.prototype.onload()
    Module.prototype.fetch()

## TODOS

看完之后，再看一遍，完善文档，加强理解。
