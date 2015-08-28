FIS3 解决方案规范定义
===========================

如果你还不了解 fis 的解决方案，请先阅读[解决方案的背景介绍](./intro.md)。

制定 fis3 解决方案规范，主要有以下考虑：

1. 统一现有解决方案，减少迁移学习成本。
2. 给解决方案开发者提供参考。

## 扩展语法糖

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
