FIS3 解决方案规范定义
===========================

如果你还不了解 fis 的解决方案，请先阅读[解决方案的背景介绍](./intro.md)。

制定 fis3 解决方案规范，主要有以下考虑：

1. 统一现有解决方案，减少迁移学习成本。
2. 给解决方案开发者提供参考。

解决方案规范主要针对语法糖扩展和线下调试功能有要求。

## 语法糖扩展

为了不占用原有 html 标签，在解决方案中需要至少扩展 load、uri、script、style、widget、framework 和 placeholder 标签。

语法格式可以因模板引擎不同而不同。以下语法规则仅供参考，但是标签名称应当保持一直。

* `@load('静态资源ID'[, 'js' | 'css'])` 用来加载某一静态资源，通过 `静态资源ID` 指定。当该资源被加载的同时，所有其依赖的资源也应当被加载。

  ```
  @load('static/libs/lib.js')
  @load('static/scss/global.scss')
  ```
* `@uri` 用来获取某一资源的最终产出路径。
  
  ```
  <div data-image="@uri('common:static/img/bg.png')"></div>
  ```
* `@script() js content @endscript` 与`html` 中 `<script></script>` 语法类似, 主要区别在于，通过此语法加载的 `script`, 会被收集到队列中，无论在模板什么位置使用，最终都会被合并在页面页脚处统一输出，自动性能优化。

  ```
  <p>xxxx</p>
  @style()
  var $ = require('/widget/jquery/jquery.js');

  $(function() {
    alert('Ready');
  });
  @endstyle
  ```
* `@style() css content @endstyle`  与`html` 中 `<style></style>` 语法类似, 主要区别在于，通过此语法加载的 `css`, 会被收集到队列中，无论在模板什么位置使用，最终都会被合并在页面头部统一输出，自动性能优化。

  ```
  @style()
  div.clsA {
    color: red;
  }
  @endstyle
  ```
* `@widget('子模板资源ID'[, localVars])`
  
  类似于各种模板引擎中的 `include` 功能，应当支持以下功能：

  * 支持局部变量传递。
  * 自动加载模板中依赖的资源。


  ```php
  @if(!Auth::guest())
    @widget('/widget/userInfo/userInfo.tpl')
  @endif
  ```

* `@framework('资源ID')`
  
  指定前端运行时框架，用来支撑 CommonJs 或者 AMD 模块化开发。如果项目采用 `CommonJs` 规范请使用 [mod.js](https://github.com/fex-team/mod/blob/master/mod.js), 如果项目采用 AMD 规范请使用 `require.js`、`esl.js` 或者其他 AMD Loader。

  ```php
  ...
  @framework('/static/mod.js')
  ...
  ```
* `@placeholder('类型')` 

  后端框架需要把收集的 js 和 css 统一输出，同时为了支持模块化开发，还需输出前端框架资源路径以及异步 js 模块资源表信息，那么具体输出在什么位置需要支持以下占位符来控制。

  * `@placeholder('js')` 用来控制收集到的 js 输出位置，一般都放在 body 前面。
  * `@placeholder('css')` 用来控制收集到的 css 输出位置，一般都放在 head 前面。
  * `@placeholder('framework')` 用来控制前端框架 js 输出位置。
  * `@placeholder('resource_map')` 用来控制异步 js 模块资源表输出位置。

## 线下调试

一个完整的解决方案推荐集成一个简单的调试服务器，支持页面预览、页面重定向和假数据模拟三部分功能。用户可以结合这三个功能，完全模拟线上环境，而无需**等待**后端开发人员提供环境支持，达到能并行开发的效果。

### 页面预览

项目中的所有页面文件，经过 fis 编译以后，应当支持直接在调试服务器中预览的功能。

* 对于静态的 html 页面直接预览即可。
* 对于动态模板页面可以结合用户提供假数据自动与模板变量绑定完成预览。

预览规则为 `http://ip:port/path/in/project`, 即页面文件在项目中的路径。如：`http://127.0.0.1:8080/page/index.tpl`

如果该项目设置了 `namesapce`。

```js
fis.set('namesapce', 'foo');
```

那么预览的地址为 `http://ip:port/foo/path/in/project`，即在原来规则的基础上加上 `namespace` 前缀。

### 页面重定向

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

### 数据 mock

假数据主要包括模板数据和 ajax 异步数据两部分。

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

  那么当页面通过 `http://ip:port/clean/url` 访问的时候，应当除了按页面存放路径规则的`假数据`被加载外，还需额外按同样的策略加载以下`假数据`。

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

  / server.conf

  ```
  rewrite ^\/api\/now /mock/ajax/api/now.php
  ```
  /mock/ajax/api/now.php

  ```php
  {
    "data": <?php echo time();?>,
    "status": 0,
    "message": "ok"
  }
  ```

  当然如果是动态脚本，返回的数据类型可以由脚本编写者定，可以是 `xml` 也可以是 `jsonp` 等等。
