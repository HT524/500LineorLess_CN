# Web 电子表格

_作者：Audrey Tang_

_翻译：Claude, [skhe](https://github.com/skhe)_

_原文：[Web Spreadsheet](http://aosabook.org/en/500L/web-spreadsheet.html)_

---

_Audrey 是一位自学成才的程序员和翻译者，以独立承包商的身份与 Apple 合作，从事云服务本地化和自然语言技术方面的工作。Audrey 此前设计并领导了第一个可运行的 Perl 6 实现，并在 Haskell、Perl 5 和 Perl 6 的计算机语言设计委员会中任职。目前 Audrey 是 g0v 的全职贡献者，并领导台湾首个电子法规制定项目。_

本章介绍了一个仅用 99 行代码编写的 Web 电子表格 (web spreadsheet)，使用了浏览器原生支持的三种语言：HTML、JavaScript 和 CSS。

本项目的 ES5 版本可在 [jsFiddle](http://jsfiddle.net/audreyt/LtDyP/) 上获取。

_(本章也有[繁体中文](https://github.com/aosabook/500lines/blob/master/spreadsheet/spreadsheet.zh-tw.markdown)版本)。_

## 引言

当 Tim Berners-Lee 在 1990 年发明万维网时，_网页_ (web pages) 是用 HTML 编写的，通过用尖括号标签 (angle-bracketed tags) 标记文本来为内容分配逻辑结构。用 `<a>…</a>` 标记的文本成为_超链接_ (hyperlinks)，可以将用户引导到 Web 上的其他页面。

在 20 世纪 90 年代，浏览器在 HTML 词汇表中加入了各种展示性标签 (presentational tags)，其中包括一些臭名昭著的非标准标签，如 Netscape Navigator 的 `<blink>…</blink>` 和 Internet Explorer 的 `<marquee>…</marquee>`，导致了广泛的可用性和浏览器兼容性问题。

为了将 HTML 限制在其最初的用途——描述文档的逻辑结构——浏览器厂商最终同意支持另外两种语言：CSS 用于描述页面的展示样式，JavaScript (JS) 用于描述其动态交互。

从那时起，这三种语言在二十年的协同演化中变得更加简洁和强大。特别是 JS 引擎的改进使得部署大规模 JS 框架（如 AngularJS）成为可能。

如今，跨平台 Web 应用程序（如 Web 电子表格）已经像上个世纪的平台专属应用程序（如 VisiCalc、Lotus 1-2-3 和 Excel）一样无处不在且广受欢迎。

一个使用 AngularJS 的 Web 应用程序能在 99 行代码内提供多少功能？让我们来看看实际效果！

## 概览

电子表格目录包含了我们对 2014 年末版本的三种 Web 语言的展示：HTML5 用于结构，CSS3 用于展示，以及 JS ES6 "Harmony" 标准用于交互。它还使用了 Web Storage 进行数据持久化，以及 Web Workers 在后台运行 JS 代码。截至撰写本文时，这些 Web 标准已被 Firefox、Chrome 和 Internet Explorer 11+ 以及 iOS 5+ 和 Android 4+ 上的移动浏览器所支持。

现在让我们在浏览器中打开电子表格（图 19.1）：

![图 19.1 - 初始界面](spreadsheet-images/01-initial.png)

## 基本概念

电子表格跨越两个维度，列从 A 开始，行从 1 开始。每个单元格都有一个唯一坐标（如 A1）和内容（如 "1874"），内容属于以下四种类型之一：

- **文本**：B1 中的 "+" 和 D1 中的 "->"，左对齐。
- **数字**：A1 中的 "1874" 和 C1 中的 "2046"，右对齐。
- **公式**：E1 中的 `=A1+C1`，计算结果为 "3920"，以浅蓝色背景显示。
- **空值**：第 2 行的所有单元格当前为空。

点击 "3920" 将焦点设置到 E1，在输入框中显示其公式（图 19.2）。

![图 19.2 - 输入框](spreadsheet-images/02-input.png)

现在将焦点设置到 A1 并将其内容更改为 "1"，导致 E1 重新计算其值为 "2047"（图 19.3）。

![图 19.3 - 已更改内容](spreadsheet-images/03-changed.png)

按 ENTER 将焦点设置到 A2 并将其内容更改为 `=Date()`，然后按 TAB，将 B2 的内容更改为 `=alert()`，然后再次按 TAB 将焦点设置到 C2（图 19.4）。

![图 19.4 - 公式错误](spreadsheet-images/04-error.png)

这表明公式可以计算为数字（E1 中的 "2047"）、文本（A2 中的当前时间，左对齐）或错误（B2 中的红色字母，居中对齐）。

接下来，让我们尝试输入 `=for(;;){}`，这是一个永不终止的无限循环的 JS 代码。电子表格会在尝试更改后自动恢复 C2 的内容来阻止这种情况。

现在用 Ctrl-R 或 Cmd-R 在浏览器中重新加载页面，以验证电子表格内容是持久化的，在浏览器会话之间保持不变。要将电子表格重置为其原始内容，请按左上角的"弯箭头"按钮。

## 渐进增强

在我们深入 99 行代码之前，值得在浏览器中禁用 JS，重新加载页面，并注意其中的差异（图 19.5）。

- 大网格不再显示，屏幕上只剩下一个 2x2 的表格，只有一个内容单元格。
- 行和列标签被替换为 `{{ row }}` 和 `{{ col }}`。
- 按下重置按钮没有效果。
- 按 TAB 或点击内容的第一行仍然会显示一个可编辑的输入框。

![图 19.5 - 禁用 JavaScript 后](spreadsheet-images/05-nojs.png)

当我们禁用动态交互 (JS) 时，内容结构 (HTML) 和展示样式 (CSS) 仍然有效。如果一个网站在 JS 和 CSS 都被禁用的情况下仍然可用，我们就说它遵循了渐进增强 (progressive enhancement) 原则，使其内容对尽可能多的受众可访问。

因为我们的电子表格是一个没有服务器端代码的 Web 应用程序，我们必须依赖 JS 来提供所需的逻辑。然而，当 CSS 没有被完全支持时，如使用屏幕阅读器和文本模式浏览器时，它仍然能正确工作。

![图 19.6 - 禁用 CSS 后](spreadsheet-images/06-nocss.png)

如图 19.6 所示，如果我们在浏览器中启用 JS 并禁用 CSS，效果是：

- 所有背景色和前景色都消失了。
- 输入框和单元格值同时显示，而不是一次只显示一个。
- 除此之外，应用程序仍然与完整版本一样工作。

## 代码详解

图 19.7 展示了 HTML 和 JS 组件之间的关联。为了理解这个图表，让我们按照浏览器加载的顺序逐一查看四个源代码文件。

![图 19.7 - 架构图](spreadsheet-images/00-architecture.png)

- **index.html**: 19 行
- **main.js**: 38 行（不包括注释和空行）
- **worker.js**: 30 行（不包括注释和空行）
- **styles.css**: 12 行

### HTML

`index.html` 的第一行声明它是用 UTF-8 编码的 HTML5：

```html
<!DOCTYPE html><html><head><meta charset="UTF-8">
```

如果没有字符集声明，浏览器可能会将重置按钮的 Unicode 符号显示为 `â†»`，这是乱码 (mojibake) 的一个例子：由编码问题导致的文本混乱。

接下来的三行是 JS 声明，按惯例放在 `head` 部分中：

```html
  <script src="lib/angular.js"></script>
  <script src="main.js"></script>
  <script>
      try { angular.module('500lines') }
      catch(e){ location="es5/index.html" }
  </script>
```

`<script src="…">` 标签从与 HTML 页面相同的路径加载 JS 资源。例如，如果当前 URL 是 `http://abc.com/x/index.html`，那么 `lib/angular.js` 指向 `http://abc.com/x/lib/angular.js`。

`try{ angular.module('500lines') }` 行测试 `main.js` 是否正确加载；如果没有，它会告诉浏览器导航到 `es5/index.html`。这种基于重定向的优雅降级 (graceful degradation) 技术确保了对于不支持 ES6 的 2015 年之前的浏览器，我们可以使用转译为 ES5 版本的 JS 程序作为回退方案。

接下来的两行加载 CSS 资源，关闭 `head` 部分，并开始包含用户可见部分的 `body` 部分：

```html
  <link href="styles.css" rel="stylesheet">
</head><body ng-app="500lines" ng-controller="Spreadsheet" ng-cloak>
```

上面的 `ng-app` 和 `ng-controller` 属性告诉 AngularJS 调用 `500lines` 模块的 `Spreadsheet` 函数，该函数会返回一个模型 (model)：一个提供文档视图绑定 (bindings) 的对象。（`ng-cloak` 属性在绑定就位之前隐藏文档。）

作为一个具体的例子，当用户点击下一行定义的 `<button>` 时，其 `ng-click` 属性会触发并调用 `reset()` 和 `calc()`，这两个是 JS 模型提供的命名函数：

```html
  <table><tr>
    <th><button type="button" ng-click="reset(); calc()">↻</button></th>
```

下一行使用 `ng-repeat` 在顶行显示列标签列表：

```html
    <th ng-repeat="col in Cols">{{ col }}</th>
```

例如，如果 JS 模型将 `Cols` 定义为 `["A","B","C"]`，那么将有三个相应标签的表头单元格 (`th`)。`{{ col }}` 标记告诉 AngularJS 进行表达式插值 (interpolate)，用 `col` 的当前值填充每个 `th` 的内容。

类似地，接下来的两行遍历 `Rows` 中的值——`[1,2,3]` 等等——为每个值创建一行，并在最左边的 `th` 单元格中标注其数字：

```html
  </tr><tr ng-repeat="row in Rows">
    <th>{{ row }}</th>
```

因为 `<tr ng-repeat>` 标签尚未被 `</tr>` 关闭，`row` 变量仍可用于表达式。下一行在当前行中创建一个数据单元格 (`td`)，并在其 `ng-class` 属性中同时使用 `col` 和 `row` 变量：

```html
    <td ng-repeat="col in Cols" ng-class="{ formula: ('=' === sheet[col+row][0]) }">
```

这里发生了几件事。在 HTML 中，`class` 属性描述了一组类名，允许 CSS 以不同方式设置样式。这里的 `ng-class` 计算表达式 `('=' === sheet[col+row][0])`；如果为真，那么 `<td>` 会获得 `formula` 作为额外的类，这会给单元格一个浅蓝色背景，如 `styles.css` 第 8 行中用 `.formula` 类选择器定义的那样。

上面的表达式通过测试 `=` 是否是 `sheet[col+row]` 字符串的初始字符（`[0]`）来检查当前单元格是否是公式，其中 `sheet` 是一个 JS 模型对象，以坐标（如 `"E1"`）为属性，以单元格内容（如 `"=A1+C1"`）为值。注意，因为 `col` 是字符串而不是数字，`col+row` 中的 `+` 表示连接而非加法。

在 `<td>` 内部，我们给用户一个输入框来编辑存储在 `sheet[col+row]` 中的单元格内容：

```html
       <input id="{{ col+row }}" ng-model="sheet[col+row]" ng-change="calc()"
        ng-model-options="{ debounce: 200 }" ng-keydown="keydown( $event, col, row )">
```

这里的关键属性是 `ng-model`，它在 JS 模型和输入框的可编辑内容之间建立双向绑定 (two-way binding)。实际上，这意味着每当用户在输入框中进行更改时，JS 模型会更新 `sheet[col+row]` 以匹配内容，并触发其 `calc()` 函数来重新计算所有公式单元格的值。

为了避免用户按住某个键时重复调用 `calc()`，`ng-model-options` 将更新速率限制为每 200 毫秒一次。

这里的 `id` 属性使用坐标 `col+row` 进行插值。HTML 元素的 `id` 属性必须与同一文档中所有其他元素的 `id` 不同。这确保了 `#A1` ID 选择器引用的是单个元素，而不是像类选择器 `.formula` 那样引用一组元素。当用户按下 UP/DOWN/ENTER 键时，`keydown()` 中的键盘导航逻辑将使用 ID 选择器来确定将焦点设置到哪个输入框。

在输入框之后，我们放置一个 `<div>` 来显示当前单元格的计算值，在 JS 模型中由 `errs` 和 `vals` 对象表示：

```html
      <div ng-class="{ error: errs[col+row], text: vals[col+row][0] }">
        {{ errs[col+row] || vals[col+row] }}</div>
```

如果在计算公式时发生错误，文本插值使用 `errs[col+row]` 中包含的错误消息，`ng-class` 将 `error` 类应用到元素上，允许 CSS 以不同的样式呈现它（红色字母、居中对齐等）。

当没有错误时，`||` 右侧的 `vals[col+row]` 被插值替代。如果它是一个非空字符串，初始字符（`[0]`）将计算为真，从而将 `text` 类应用到元素上，使文本左对齐。

因为空字符串和数值没有初始字符，`ng-class` 不会给它们分配任何类，所以 CSS 可以将它们设置为右对齐作为默认情况。

最后，我们在列级别用 `</td>` 关闭 `ng-repeat` 循环，用 `</tr>` 关闭行级别循环，并以以下内容结束 HTML 文档：

```html
    </td>
  </tr></table>
</body></html>
```

### JS：主控制器

`main.js` 文件定义了 `500lines` 模块及其 `Spreadsheet` 控制器函数，这是 `index.html` 中 `<body>` 元素所要求的。

作为 HTML 视图和后台 Worker 之间的桥梁，它有四个任务：

1. 定义列和行的维度和标签。
2. 为键盘导航和重置按钮提供事件处理程序。
3. 当用户更改电子表格时，将新内容发送给 Worker。
4. 当计算结果从 Worker 返回时，更新视图并保存当前状态。

图 19.8 中的流程图更详细地展示了控制器与 Worker 之间的交互：

![图 19.8 - 控制器-Worker 流程图](spreadsheet-images/00-flowchart.png)

现在让我们逐步查看代码。在第一行中，我们请求 AngularJS 的 `$scope`：

```js
angular.module('500lines', []).controller('Spreadsheet', function ($scope, $timeout) {
```

`$scope` 中的 `$` 是变量名的一部分。这里我们还从 AngularJS 请求了 `$timeout` 服务函数；稍后我们将用它来阻止无限循环公式。

要将 `Cols` 和 `Rows` 放入模型中，只需将它们定义为 `$scope` 的属性：

```js
  // 开始 $scope 属性; 从列/行标签开始
  $scope.Cols = [], $scope.Rows = [];
  for (col of range( 'A', 'H' )) { $scope.Cols.push(col); }
  for (row of range( 1, 20 )) { $scope.Rows.push(row); }
```

ES6 的 `for...of` 语法使得用起止点遍历范围变得很简单，辅助函数 `range` 被定义为一个生成器 (generator)：

```js
  function* range(cur, end) { while (cur <= end) { yield cur;
```

上面的 `function*` 表示 `range` 返回一个迭代器 (iterator)，其中的 `while` 循环每次 `yield` 一个值。每当 `for` 循环需要下一个值时，它会在 `yield` 行之后恢复执行：

```js
    // 如果是数字，加一；否则移到下一个字母
    cur = (isNaN( cur ) ? String.fromCodePoint( cur.codePointAt()+1 ) : cur+1);
  } }
```

为了生成下一个值，我们用 `isNaN` 检查 `cur` 是否表示一个字母（NaN 代表"不是一个数字"）。如果是，我们获取该字母的码点值，加一，然后将码点转换回来以获得下一个字母。否则，我们简单地将数字加一。

接下来，我们定义处理跨行键盘导航的 `keydown()` 函数：

```js
  // UP(38) 和 DOWN(40)/ENTER(13) 将焦点移到上方(-1)和下方(+1)的行
  $scope.keydown = ({which}, col, row)=>{ switch (which) {
```

箭头函数从 `<input ng-keydown>` 接收参数 `($event, col, row)`，使用解构赋值 (destructuring assignment) 将 `$event.which` 赋给 `which` 参数，并检查它是否是三个导航键码之一：

```js
    case 38: case 40: case 13: $timeout( ()=>{
```

如果是，我们使用 `$timeout` 来安排在当前 `ng-keydown` 和 `ng-change` 处理程序之后执行焦点切换。因为 `$timeout` 需要一个函数作为参数，`()=>{…}` 语法构造了一个表示焦点切换逻辑的函数，它首先检查移动方向：

```js
      const direction = (which === 38) ? -1 : +1;
```

`const` 声明符意味着 `direction` 在函数执行期间不会改变。移动方向是向上 (`-1`，从 A2 到 A1) 如果键码是 38 (UP)，否则是向下 (`+1`，从 A2 到 A3)。

接下来，我们使用 ID 选择器语法（如 `"#A3"`）检索目标元素，使用一对反引号编写的模板字符串 (template string) 构造，连接前导的 `#`、当前的 `col` 和目标 `row + direction`：

```js
      const cell = document.querySelector( `#${ col }${ row + direction }` );
      if (cell) { cell.focus(); }
    } );
  } };
```

我们对 `querySelector` 的结果进行额外检查，因为从 A1 向上移动会产生选择器 `#A0`，它没有对应的元素，因此不会触发焦点切换——在最底行按 DOWN 也是如此。

接下来，我们定义 `reset()` 函数，让重置按钮可以恢复工作表的内容：

```js
  // 默认工作表内容, 包含一些数据单元格和一个公式单元格
  $scope.reset = ()=>{
    $scope.sheet = { A1: 1874, B1: '+', C1: 2046, D1: '->', E1: '=A1+C1' }; }
```

`init()` 函数尝试从 `localStorage` 恢复工作表内容的先前状态，如果是首次运行应用程序则使用初始内容：

```js
  // 定义初始化函数，并立即调用它
  ($scope.init = ()=>{
    // 恢复先前的 .sheet; 如果是首次运行则重置为默认值
    $scope.sheet = angular.fromJson( localStorage.getItem( '' ) );
    if (!$scope.sheet) { $scope.reset(); }
    $scope.worker = new Worker( 'worker.js' );
  }).call();
```

上面的 `init()` 函数中有几点值得注意：

- 我们使用 `($scope.init = ()=>{…}).call()` 语法来定义函数并立即调用它。
- 因为 `localStorage` 只存储字符串，我们使用 `angular.fromJson()` 从 JSON 表示中解析工作表结构。
- 在 `init()` 的最后一步，我们创建一个新的 Web Worker 线程并将其赋给 `worker` 作用域属性。虽然 Worker 没有直接在视图中使用，但习惯上使用 `$scope` 来共享跨模型函数使用的对象，在这里就是 `init()` 和下面的 `calc()` 之间共享。

虽然 `sheet` 保存用户可编辑的单元格内容，但 `errs` 和 `vals` 包含计算结果——错误和值——对用户来说是只读的：

```js
  // 公式单元格可能在 .errs 中产生错误; 正常单元格内容在 .vals 中
  [$scope.errs, $scope.vals] = [ {}, {} ];
```

有了这些属性，我们可以定义 `calc()` 函数，当用户对 `sheet` 进行更改时触发：

```js
  // 定义计算处理程序; 暂不调用
  $scope.calc = ()=>{
    const json = angular.toJson( $scope.sheet );
```

这里我们对 `sheet` 的状态做一个快照并将其存储在常量 `json` 中，一个 JSON 字符串。接下来，我们从 `$timeout` 构造一个 Promise，如果接下来的计算耗时超过 99 毫秒则取消它：

```js
    const promise = $timeout( ()=>{
      // 如果 Worker 在 99 毫秒内没有返回，终止它
      $scope.worker.terminate();
      // 回退到先前的状态并创建新的 Worker
      $scope.init();
      // 使用最后已知的状态重新进行计算
      $scope.calc();
    }, 99 );
```

由于我们通过 HTML 中的 `<input ng-model-options>` 属性确保 `calc()` 每 200 毫秒最多调用一次，这种安排为 `init()` 留出了 101 毫秒来将 `sheet` 恢复到最后一个已知正常的状态并创建新的 Worker。

Worker 的任务是根据 `sheet` 的内容计算 `errs` 和 `vals`。因为 `main.js` 和 `worker.js` 通过消息传递 (message-passing) 进行通信，我们需要一个 `onmessage` 处理程序来接收结果：

```js
    // 当 Worker 返回时，将其效果应用到作用域上
    $scope.worker.onmessage = ({data})=>{
      $timeout.cancel( promise );
      localStorage.setItem( '', json );
      $timeout( ()=>{ [$scope.errs, $scope.vals] = data; } );
    };
```

如果 `onmessage` 被调用，我们知道 `json` 中的工作表快照是稳定的（即不包含无限循环公式），所以我们取消 99 毫秒的超时，将快照写入 `localStorage`，并通过一个 `$timeout` 函数安排 UI 更新，将 `errs` 和 `vals` 更新到用户可见的视图中。

有了处理程序，我们可以将 `sheet` 的状态发送给 Worker，在后台开始计算：

```js
    // 将当前工作表内容发送给 Worker 处理
    $scope.worker.postMessage( $scope.sheet );
  };

  // Worker 就绪时开始计算
  $scope.worker.onmessage = $scope.calc;
  $scope.worker.postMessage( null );
});
```

### JS：后台 Worker

使用 Web Worker 来计算公式而不是使用主 JS 线程有三个原因：

1. 当 Worker 在后台运行时，用户可以自由地继续与电子表格交互，而不会被主线程中的计算阻塞。
2. 因为我们在公式中接受任何 JS 表达式，Worker 提供了一个沙箱 (sandbox)，防止公式干扰包含它们的页面，比如弹出一个 `alert()` 对话框。
3. 公式可以将任何坐标作为变量引用。其他坐标可能包含另一个公式，最终可能形成循环引用 (cyclic reference)。为了解决这个问题，我们使用 Worker 的全局作用域对象 `self`，并将这些变量定义为 `self` 上的 getter 函数来实现循环防止逻辑。

有了这些考虑，让我们看看 Worker 的代码。

Worker 的唯一目的是定义其 `onmessage` 处理程序。处理程序接收 `sheet`，计算 `errs` 和 `vals`，然后将它们发送回主 JS 线程。当我们收到消息时，首先重新初始化这三个变量：

```js
let sheet, errs, vals;
self.onmessage = ({data})=>{
  [sheet, errs, vals] = [ data, {}, {} ];
```

为了将坐标转换为全局变量，我们首先使用 `for…in` 循环遍历 `sheet` 中的每个属性：

```js
  for (const coord in sheet) {
```

ES6 引入了 `const` 和 `let` 来声明块级作用域 (block scoped) 的常量和变量；上面的 `const coord` 意味着循环中定义的函数会捕获每次迭代中 `coord` 的值。

相比之下，早期版本的 JS 中的 `var coord` 会声明一个函数作用域 (function scoped) 的变量，循环每次迭代中定义的函数最终都会指向同一个 `coord` 变量。

按照惯例，公式变量不区分大小写，并且可以有一个可选的 `$` 前缀。因为 JS 变量是区分大小写的，我们使用 `map` 遍历同一坐标的四个变量名：

```js
    // 四个变量名指向同一坐标: A1, a1, $A1, $a1
    [ '', '$' ].map( p => [ coord, coord.toLowerCase() ].map(c => {
      const name = p+c;
```

注意上面的箭头函数简写语法：`p => ...` 等同于 `(p) => { ... }`。

对于每个变量名，如 `A1` 和 `$a1`，我们在 `self` 上定义一个访问器属性 (accessor property)，每当它们在表达式中被求值时计算 `vals["A1"]`：

```js
      // Worker 在多次计算中被重用，所以每个变量只定义一次
      if ((Object.getOwnPropertyDescriptor( self, name ) || {}).get) { return; }

      // 定义 self['A1']，与全局变量 A1 是同一回事
      Object.defineProperty( self, name, { get() {
```

上面的 `{ get() { … } }` 语法是 `{ get: ()=>{ … } }` 的简写。因为我们只定义了 `get` 而没有定义 `set`，变量变成了只读的，不能被用户提供的公式修改。

`get` 访问器首先检查 `vals[coord]`，如果它已经被计算过，就直接返回：

```js
        if (coord in vals) { return vals[coord]; }
```

如果没有，我们需要从 `sheet[coord]` 计算 `vals[coord]`。

首先我们将它设置为 `NaN`，这样像将 A1 设置为 `=A1` 这样的自引用会得到 `NaN` 而不是无限循环：

```js
        vals[coord] = NaN;
```

接下来我们通过用前缀 `+` 将 `sheet[coord]` 转换为数字来检查它是否是一个数字，将数字赋给 `x`，并将其字符串表示与原始字符串进行比较。如果不同，我们将 `x` 设置为原始字符串：

```js
        // 将数字字符串转换为数字，这样当两者都是数字时 =A1+C1 才能正常工作
        let x = +sheet[coord];
        if (sheet[coord] !== x.toString()) { x = sheet[coord]; }
```

如果 `x` 的初始字符是 `=`，那么它是一个公式单元格。我们用 `eval.call()` 计算 `=` 之后的部分，使用第一个参数 `null` 告诉 `eval` 在全局作用域中运行，从而将 `x` 和 `sheet` 等词法作用域变量隐藏在求值过程之外：

```js
        // 计算以 = 开头的公式单元格
        try { vals[coord] = (('=' === x[0]) ? eval.call( null, x.slice( 1 ) ) : x);
```

如果求值成功，结果存储在 `vals[coord]` 中。对于非公式单元格，`vals[coord]` 的值就是 `x`，可能是数字或字符串。

如果 `eval` 产生错误，`catch` 块会测试是否因为公式引用了尚未在 `self` 中定义的空单元格：

```js
        } catch (e) {
          const match = /\$?[A-Za-z]+[1-9][0-9]*\b/.exec( e );
          if (match && !( match[0] in self )) {
```

在这种情况下，我们将缺失单元格的默认值设置为 `"0"`，清除 `vals[coord]`，并使用 `self[coord]` 重新运行当前计算：

```js
            // 公式引用了未初始化的单元格; 将其设置为 0 并重试
            self[match[0]] = 0;
            delete vals[coord];
            return self[coord];
          }
```

如果用户稍后在 `sheet[coord]` 中给缺失的单元格赋予内容，那么临时值会被 `Object.defineProperty` 覆盖。

其他类型的错误存储在 `errs[coord]` 中：

```js
          // 否则，将捕获的异常转为字符串存储在 errs 对象中
          errs[coord] = e.toString();
        }
```

在出错的情况下，`vals[coord]` 的值将保持为 `NaN`，因为赋值没有执行完成。

最后，`get` 访问器返回存储在 `vals[coord]` 中的计算值，它必须是数字、布尔值或字符串：

```js
        // 如果 vals[coord] 不是数字或布尔值，将其转为字符串
        switch (typeof vals[coord]) {
            case 'function': case 'object': vals[coord]+='';
        }
        return vals[coord];
      } } );
    }));
  }
```

所有坐标的访问器定义完成后，Worker 再次遍历所有坐标，通过 `self[coord]` 调用每个访问器，然后将结果 `errs` 和 `vals` 发送回主 JS 线程：

```js
  // 对于工作表中的每个坐标，调用上面定义的属性 getter
  for (const coord in sheet) { self[coord]; }
  return [ errs, vals ];
}
```

### CSS

`styles.css` 文件只包含几个选择器及其展示样式。首先，我们设置表格样式以合并所有单元格边框，不在相邻单元格之间留空隙：

```css
table { border-collapse: collapse; }
```

表头单元格和数据单元格共享相同的边框样式，但我们可以通过背景色区分它们：表头单元格是浅灰色，数据单元格默认为白色，公式单元格获得浅蓝色背景：

```css
th, td { border: 1px solid #ccc; }
th { background: #ddd; }
td.formula { background: #eef; }
```

每个单元格的计算值显示宽度是固定的。空单元格获得最小高度，长行以尾随省略号截断：

```css
td div { text-align: right; width: 120px; min-height: 1.2em;
         overflow: hidden; text-overflow: ellipsis; }
```

文本对齐和装饰由每个值的类型决定，反映在 `text` 和 `error` 类选择器中：

```css
div.text { text-align: left; }
div.error { text-align: center; color: #800; font-size: 90%; border: solid 1px #800 }
```

至于用户可编辑的输入框，我们使用绝对定位将其覆盖在单元格上方，并使其透明以便底层带有单元格值的 `div` 可以透过显示：

```css
input { position: absolute; border: 0; padding: 0;
        width: 120px; height: 1.3em; font-size: 100%;
        color: transparent; background: transparent; }
```

当用户将焦点设置到输入框时，它会跳到前景：

```css
input:focus { color: #111; background: #efe; }
```

此外，底层的 `div` 被折叠成单行，因此完全被输入框覆盖：

```css
input:focus + div { white-space: nowrap; }
```

## 结语

由于本书的主题是 500 行或更少，99 行的 Web 电子表格是一个最简示例——请随意在任何方向上进行实验和扩展。

以下是一些想法，都可以在剩余的 401 行空间内轻松实现：

- 使用 ShareJS、AngularFire 或 GoAngular 的协作在线编辑器。
- 使用 angular-marked 为文本单元格提供 Markdown 语法支持。
- 来自 OpenFormula 标准的常用公式函数（SUM、TRIM 等）。
- 通过 SheetJS 与流行的电子表格格式（如 CSV 和 SpreadsheetML）互操作。
- 从在线电子表格服务（如 Google Spreadsheet 和 EtherCalc）导入和导出。

## 关于 JS 版本的说明

本章旨在演示 ES6 中的新概念，因此我们使用 Traceur 编译器将源代码转译为 ES5 以在 2015 年之前的浏览器上运行。

如果你更愿意直接使用 2010 版的 JS，`as-javascript-1.8.5` 目录中有以 ES5 风格编写的 `main.js` 和 `worker.js`；其源代码与 ES6 版本逐行对应，行数相同。

对于偏爱更简洁语法的人，`as-livescript-1.3.0` 目录使用 LiveScript 代替 ES6 来编写 `main.ls` 和 `worker.ls`；它比 JS 版本少了 20 行。

在 LiveScript 语言的基础上，`as-react-livescript` 目录使用 ReactJS 框架；它比 AngularJS 的等价物多了 10 行，但运行速度明显更快。

如果你有兴趣将这个示例翻译为其他的 JS 语言，请发送一个 pull request——我很乐意了解！
