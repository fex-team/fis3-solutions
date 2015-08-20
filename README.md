fis 解决方案规范
======================================

目前基于 fis 的解决方案越来越多，以后还会有更多，所以希望对各种解决方案有一个统一的说明。

编写此规范主要有以下几个出发点：

* 让初学者全面地认识什么是 fis 解决方案。
* 如果需要自定义解决方案，此文档可以用来作为有效的参考资料。
* 以此规范作为标准，统一现有解决方案，让使用者根据后端不同的技术选型能快速地适应到其他解决方案中。

在阅读此文前，请先对 [fis](http://fis.baidu.com) 编译工具有一定的了解。

**如果有任何问题或建议，请提交 [issues](https://github.com/fex-team/fis3-solutions/issues/new)**

## 什么是 fis 解决方案？

fis 解决方案是一个基于 fis 编译工具，针对特定后端和特定模板引擎，包含模板语法糖、线下调试、模块化开发、组件化开发、目录规范、后端运行时框架以及子站点拆分等一系列有助于提高前端开发效率的各种方案集合。

### 模板语法糖

当选定某一种模板引擎后，需要扩展一些必要的标签（语法）以便于资源加载引用、自动完成性能优化、支撑模块化开发和组件化开发等。

之所以需要扩展新语法最主要的原因是不污染原有 html 标签的语义。因为各种模板引擎的语法各不相同，以下语法示例仅作为参考。

* `import("静态资源ID")` 

  用来加载某一静态资源，通过 `静态资源ID` 指定。

  fis 中所有资源都可以根据配置随意部署到各种目录，或者 cdn 服务器上，最终产出的路径是相对不固定的。对于开发人员来说，最终产出的路径是不可靠的，所以需要通过`静态资源ID`来指定资源。`静态资源ID` 是相对固定的，他的值为该资源在项目中的绝对路径去掉开头的斜线`/`。如：

  * `static/js/mod.js`
  * `static/css/global.css`
  * `widget/header/header.tpl`

  如果该项目设置了 `namespace`。

  ```js
  fis.set('namespace', 'common');
  ```

  那么`静态资源ID`会变成 `${命名空间}/${资源在该项目中绝对路径}`，如：

  * `common:static/js/mod.js`
  * `common:static/css/global.css`

  fis 编译工具，会把项目中的静态资源生成一张静态资源表存放在 `map.json` 文件中，该表通过 `静态资源ID` 标识了所有静态资源的类型、最终产出路径和依赖信息。当通过 `import` 加载某一资源时，程序需要读取该表，将真实的产出路径输出。

  除了加载资源本身外，还应该进一步分析该资源依赖表，递归加载所有依赖，减少人工分析成本。这也是对 fis [三种语言能力](http://fis.baidu.com/fis3/docs/user-dev/extlang.html)中[声明依赖](http://fis.baidu.com/fis3/docs/user-dev/require.html)能力的落实。

  示例

  ```php
  ...
  @import("widget/ui/jquery/jquery.js")
  @import("static/sidebar/sidebar.css")
  ...
  ```

  `import` 是通过`静态资源ID`来加载资源的，说明他是可以跨[子站点](#子站点拆分)加载资源的。当 `import` 只是加载`当前站点`下的资源时，可以使用相对路径或者基于项目的绝对路径。如：

  文件：/widget/header/header.tpl

  ```php
  @import("./header.js")
  @import("/static/js/lib.js")
  ```

  此类路径，需要借助编译工具，最终替换成`静态资源ID`。
* `url("静态资源ID")` 

  用来输出静态资源的访问路径，并不加载该资源。

  ```php
  ...
  <div data-src="@url('common:static/images/icon.png')">
  </div>
  ...
  ```
* `@script()@endscript` 

  与`html` 中 `<script></script>` 语法类似, 主要区别在于，通过此语法加载的 `script`, 会被收集，无论在模板什么位置使用，最终都会被合并在页面页脚处统一输出，自动性能优化。

  有了此功能，js 脚本可以按就近原则，忽略性能问题与关联的 html 写在一起，提高代码可读性和易维护性。

  1. `@script()js content@endscript`

    ```php
    <div class="xxx">dom</div>
    @script()
    require(['./script.js'], function(init) {
      init('div.xxx');
    });
    @endscript
    ```
  2. `@script('远程 js 地址')@endscript` 用来加载线上 js。
  3. `@script('资源ID')@endscript` 等价于 `@import('资源ID')`
* `@style()@endstyle` 

  与`html` 中 `<style></style>` 语法类似, 主要区别在于，通过此语法加载的 `css`, 会被收集，无论在模板什么位置使用，最终都会被合并在页面头部统一输出，自动性能优化。

  有了此功能，css 内容可以按就近原则，忽略性能问题与关联的 html 写在一起，提高代码可读性和易维护性。

  支持以下三种用法。

  1. `@style()css content@endstyle`

    ```php
    <div class="xxx">dom</div>
    @style()
    div.xxx {
      color: red;
    }
    @endstyle
    ```
  2. `@style('远程 css 地址')@endstyle` 用来加载线上 css。
  3. `@style('资源ID')@endstyle` 等价于 `@import('资源ID')`
* `@framework('资源ID')`
  
  指定前端运行时框架，用来支撑 CommonJs 和 AMD 模块化开发。如果项目采用 `CommonJs` 规范请使用 [mod.js](https://github.com/fex-team/mod/blob/master/mod.js), 如果项目采用 AMD 规范请使用 `require.js`、`esl.js` 或者其他 AMD Loader.

  后端框架需要根据不同的方案对异步 js 模块，生成映射表和依赖表（AMD 方案不会生成依赖表）。


  


### 线下调试

一个完整的解决方案应当集成一个简单的调试服务器，支持页面预览、页面重定向和假数据模拟三部分功能。用户可以结合这三个功能，完全模拟线上环境，而无需**等待**后端开发人员提供环境支持。可以达到并行开发的效果。

#### 页面预览

项目中的所有页面文件，应当支持直接在调试服务器中预览的功能。

* 对于静态的 html 页面直接预览即可。
* 对于动态模板页面可以结合用户提供假数据自动与模板变量绑定完成预览。

预览规则为 `http://ip:port/path/in/project`, 即页面文件在项目中的路径。如：`http://127.0.0.1:8080/page/index.tpl`

如果该项目设置了 `namesapce`。

```js
fis.set('namesapce', 'foo');
```

那么预览的地址为 `http://ip:port/foo/path/in/project`，即在原来规则的基础上加上 `namespace` 前缀。

#### 页面重定向

该服务器除了能够直接页面预览之外，还应当支持页面重定向功能，用来实现线上地址模拟。如：当访问`http://ip:port/user`的时候，可以重定向到 `http://ip:port/page/user/index.tpl` 页面。

用户可以通过配置项目根目录的 `server.conf` 文件来设置重定向规则。

示例：

```ini
# 重定向 / => /page/home.tpl
rewrite \/$ /page/home.tpl

# 重定向用户查看页面
# /user/1 => /page/user/view?id=1
rewrite ^\/user\/(\d*)$ /page/user/view.tpl?id=$1

# redirect /jump /page/about.tpl
redirect \/jump /page/about.tpl
```

语法规则为

```
指令名称 匹配规则（用来匹配原始请求地址） 目标地址 
```

* `指令` 应当至少支持以下两种指令：

  1. `rewrite` 重定向页面，浏览器地址栏不会发生变化。
  2. `redirect` 跳转页面，浏览器地址栏发生变化。
* `匹配规则` 统一使用正则来配置，应当支持分组。如：`\/user\/(\d+)$`。
* `目标地址` 可以是服务器内任意资源路径或者访问路径，可以通过 `$数字` 来获取正则规则中分组的捕获。

#### 数据 mock

数据 mock 分为两块。

1. **模板数据**

  对于动态的模板页面，需要支持结合用户提供的假数据完成简单预览功能。

  假数据应该支持两种形式：

  1. 静态 json 数据，以 xxx.json 文件提供。数据内容用 json 格式存放。

    ```json
    {
      "title": "用户列表",
      "desc": "页面描述"
    }
    ```
  2. 根据后端选型，通过一种特定脚本支持动态数据。如：`xxx.jsp` 或者 `xxx.php`。

    ```php
    <?php

    // 支持动态逻辑，甚至去线上拉取真实数据。
    // 或者去 api 文档平台拉取数据。

    return array(
      "timestamp" => time(),
    );
    ```

  模板页面中模板数据应当根据`假数据`存放规范自动加载相应的`假数据`文件，并完成绑定。

  如上示例，在模板中的 title 变量应该被自动赋值为 `用户列表`。

  假定后端模板引擎就是纯 php

  ```php
  <title><?php echo $title ?></title>
  ```

  页面预览时，标题应该输出为“用户列表”。

  假定`假数据文件`全部存放在 `mock` 文件夹下面，页面文件全部存放在 `page` 目录下面，那么当访问 `page/a/b/c.tpl` 页面时，应当按以下顺序尝试加载 `假数据`，并将所有数据按顺序合并起来，后加载的数据覆盖先加载的数据，采用类似 `jQuery.extend` 的合并策略。

  * /mock/global.json
  * /mock/page.json
  * /mock/page/a.json
  * /mock/page/a/b.json
  * /mock/page/a/b/c.json

  动态`假数据`文件（通过动态脚本提供的数据文件）应当也有同样的加载策略。

  如果`静态假数据`和`动态假数据`文件都同时存在，应当都同时加载，且 `动态假数据` 后加载，使其数据优先级更高。

  `假数据`存放目录规则除了能按页面在项目中的路径来之外，还需支持按该页面的访问地址来存放。

  举个例子，如上面例子中的页面 `page/a/b/c.tpl` 如果用户配置了 url rewrite.

  ```
  rewrite \/clean\/url$ /page/a/b/c.tpl
  ```

  那么当页面通过 `http://ip:port/clean/url` 访问的时候，应当除了按页面存放路径规则的`假数据`被加载外，还需按同样的策略加载以下`假数据`。

  * /mock/clean.json
  * /mock/clean/url.json

  之所以把`假数据`能按各种目录存放，主要是考虑到，页面与页面之间的公用的`假数据`，用户可以根据公用程度选择存放在不同的文件。
2. **ajax 数据**

  可以结合 url rewrite 和静态 json 文件，完全模拟异步 ajax 数据。

  如：/mock/ajax/user/list.json

  ```json
  {
    "data": [
      {
        "id": 1,
        "name": "foo"
      }
    ],
    "status": 0,
    "message": "ok"
  }
  ```

  /server.conf

  ```
  rewrite ^\/user\/list /mock/ajax/user/list.json
  ```

  当用户请求 `http://ip:port/user/list` 时，返回的是 list.json 中的 json 静态数据。

  除了 url rewrite 和 静态 json 文件结合外，还需支持 url rewrite 和动态脚本结合，满足动态数据模拟的需求。

  ```php
  {
    "data": <?php echo time();?>,
    "status": 0,
    "message": "ok"
  }
  ```

  当然如果是动态脚本，返回的数据类型可以由脚本编写者定，可以是 `xml` 也可以是 `jsonp` 等等。

### 前端目录规范

### 模块化开发

## 现有解决方案集合
