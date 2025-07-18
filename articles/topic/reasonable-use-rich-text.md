# 跨端富文本显示问题

先来几个前提条件：

1. 用户协议等内容多存在于 word 文件中
2. 需要字号响应，避免字号过大且不可改而失去美观
3. 内容字符不宜过长而过大，比如直接另存为 .htm 文件会有 1M 大小

然后需要解决几个问题：

1. word 文件能转为 html 或 markdown 等在前端显示的格式
2. 去掉转为富文本时很多无用的标签及属性
3. 去掉或替换掉内容的字号等尺寸信息
4. 有个较好的结果文件的存储方案，不影响加载和渲染

这样不起眼的需求却也能成为课题，前端真是细致而丰富呀。

## 转格式

### 转 html

将 word 文件里的内容，复制到富文本编辑器中，即可得到 html 字符串，<br />
这应该是最容易想到的方案了，但它也有着一些小缺点。

1. 如果偷懒不是自建的富文本编辑器，不太好拿到实例，进而获取结果
2. 如果在 F12 中复制 dom，则会掺杂一些编辑器特有的东西，比如图片缩放的框/只有视频封面没视频等

因此，想更纯粹地获取 html 字符串，算是有两个方案：

1. 要么本地跑富文本编辑器，通过程序实例获取
2. 要么直接用工具导出为 html 文件，比如 word 里 `文件 -> 另存为 -> *.htm` 等工具
3. 转其他格式后再转 html 文件，比如下文的 markdown

## 转 markdown

而其实能转为 markdown 字符串也是不错的选择，甚至能比 html 还干净，但这也是它的缺点。

1. 会丢失很多样式，比如特别字号/字体/图片尺寸或浮动等
2. 转换失败不可控，比如表格样式等，发现出问题了也不好调整

因此对于纯文本或布局简单的 word 转 markdown 就很轻松，再转 html 也很容易。

个人比较推荐 [Typora](https://www.typora.io/) 这款软件，从 word 复制内容粘贴进去，即可另存为 md 文件。<br />
欢迎分享更多 markdown 转 html 的工具，嘤嘤嘤~

## 去冗余

拿到了 html 字符串后，可能会有一堆无用的标签和属性，或者低版本废弃的标签。如下：

```html
<body style="tab-interval:21pt;text-justify-trim:punctuation;" ><!--StartFragment--><div class="Section0"  style="layout-grid:15.6000pt;" ><h1 align=center  style="text-align:center;" ><b><span style="mso-spacerun:'yes';font-family:΢���ź�;color:rgb(0,0,0);
font-weight:bold;font-size:14.0000pt;mso-font-kerning:22.0000pt;" ><font face="΢���ź�" >�����ղ�</font></body>
```

因此，有两个方向，要么换用转 markdown 方案，要么为其去冗余。

在去冗余方面有以下几种方案：

1. 将文本保存为 html 文件，浏览器打开，按 F12 复制 dom，浏览器会自动忽略一些
2. 将文本 append 到临时 div 中再获取新 innerHTML，用于运行时环境，原理同上
3. 将文本中 `<([^ >]+)([^>]*?)>` 标签部分不标准的 replace 掉
4. 将文本转为虚拟 dom 的数组，递归遍历掉无用的部分

```js
/// 比如小程序端我曾用过这样的方法
function simplifyRichText(txt) {
  return txt.replace(/<([^ >]+)([^>]*?)>/g, (all, tag, other) => {
    if (["a", "abbr", "address", "article", "aside", "b", "bdi", "bdo", "big", "blockquote", "br", "caption", "center", "cite", "code", "col", "colgroup", "dd", "del", "div", "dl", "dt", "em", "fieldset", "font", "footer", "h1", "h2", "h3", "h4", "h5", "h6", "header", "hr", "i", "img", "ins", "label", "legend", "li", "mark", "nav", "ol", "p", "pre", "q", "rt", "ruby", "s", "section", "small", "span", "strong", "sub", "sup", "table", "tbody", "td", "tfoot", "th", "thead", "tr", "tt", "u", "ul"].some((e) => e === tag)) {
      return `<${tag}${other}>`;
    }
    return '';
  });
}
```

如果能做好转虚拟 dom 方案，其实也挺一劳永逸的，比如用下面的函数将 html 字符串转为数组。<br />
之后是递归遍历还是其他算法就看大佬的本事了。

```js
/// 将 html 字符串转为虚拟 dom 数组
// eg: '<p><b>粗</b></p>' => [{ tag: 'p', childNodes: [{ tag: 'b', innerHTML: '粗' }] }]
function initTemplate(html, re = []) {
  html = html.replace(/[\n\t]\s*/gi, ""); // 去除换行和不必要的空白
  // TODO: 去掉 <!-- --> 之类的注释节点
  const htmlReg2 = /<([a-z]+)([^<]+)*(?:>(.*)<\/\1>|\s*\/>)|(?!:\>)[^<]+(?!:<)/gi;
  html.replace(htmlReg2, (outerHTML, tagName, attrs, innerHTML) => {
    // 将 attrs 字符串转为对象形式
    attrs = stringToObject(attrs, " ", ":");
    // 如果还有 innerHTML 则继续寻找子级
    const childNodes = innerHTML ? initTemplate(innerHTML, []) : [];
    // 放入父级数组之下，纯文本无 tagName
    re.push({ tagName, attrs, outerHTML, innerHTML, childNodes });
  });
  return re;
}
```

## 样式调整

如果用的 html 方案，那么会有一堆行内样式 `style="font-size:14pt"`，<br />
显然不够响应式，如果根元素字号 `10px` 则富文本这块的文字回显就会很大。<br />
修改起来还比较简单，比如字符串替换为 `rem` 之类的。

如果用的 markdown 方案（或 md 再转 html），则使用的是渲染环境的默认样式。<br />
那么其实可修改程度就大很多了，用 css 重新设置之类的。<br />
可见，此方案有着明显的环境要求，<br />
比如小程序的 rich-text 不支持外层 css 之类的，或者 flutter 中仍需转为 dart 语言等。
故而在不同环境，markdown 如何再转换和样式如何调整则需因地制宜，不像 html 那样通用。

其次，还有一些问题存在，需要注意。
图片等富媒体的 max-width，视频封面问题，表格超出问题。

## 存储方案

最终以 html 格式存储，是比较符合前端思维的，<br />
html 也比较常见，可通过函数再转其他显示方案。

而这段 html 存在哪，各有有缺点吧，还可以略做考虑：

1. 存为变量字符串（直接打包调用方便/但占内存）
2. 存在数据库（异步获取可后台修改/但流程复杂）
3. 存为 txt 或 json 文件走请求获取（同上）
4. 存为 html 文件走 web-view（简便且跨平台/但启动缓慢）

再加一些奇奇怪怪的环境限制：

1. 是否在弹窗中显示（小程序 web-view 只能全屏）
2. 是否有内存限制（小程序主包不可大于 2M）
3. 是否有网速要求（若字符串大小有 500k 在 3G 网下会比较慢）

想必经过你的抉择，已经能选出适用于你项目的方案了，加油大宝贝。
