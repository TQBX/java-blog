[TOC]

# 系统结构

- 前台系统：
  - jQuery
  - SemanticUI框架
- 后台系统



# 插件

## Markdown编辑器editormd

地址：[https://pandao.github.io/editor.md/](https://pandao.github.io/editor.md/)

```html
<!DOCTYPE html>
<html lang="zh">
    <head>
        <meta charset="utf-8" />
        <title>Simple example - Editor.md examples</title>
        <link rel="stylesheet" href="css/style.css" />
        <link rel="stylesheet" href="../css/editormd.css" /> <!-- 主要css -->
        <link rel="shortcut icon" href="https://pandao.github.io/editor.md/favicon.ico" type="image/x-icon" />
    </head>
    <body>
        <div id="layout">
            <header>
                <h1>Simple example</h1>
            </header>
            <div id="test-editormd">
                <textarea style="display:none;">

	<!-- 文章内容 -->

</textarea>
            </div>
        </div>
        <script src="js/jquery.min.js"></script>
        <script src="../editormd.min.js"></script> <!-- 主要js -->
        <script type="text/javascript">
            // 初始化
			var testEditor;

            $(function() {
                testEditor = editormd("test-editormd", {
                    width   : "90%",
                    height  : 640,
                    syncScrolling : "single",
                    path    : "../lib/"     //初始化lib库的路径
                });
                
                /*
                // or
                testEditor = editormd({
                    id      : "test-editormd",
                    width   : "90%",
                    height  : 640,
                    path    : "../lib/"
                });
                */
            });
        </script>
    </body>
</html>
```

解决全屏重叠问题

```java
<div id="editormd" style="z-index: 1 !important;">
    <textarea style="display:none;" name="content"  placeholder="博客内容"></textarea>
    </div>
```

## 排版插件typo.css

地址：[https://github.com/sofish/typo.css/](https://github.com/sofish/typo.css/)

预览：[https://typo.sofi.sh/](https://typo.sofi.sh/)

## 动画插件Animate.css

地址：[https://animate.style/](https://animate.style/)

github：[https://github.com/animate-css/animate.css](https://github.com/animate-css/animate.css)

简单使用：

```html
<head>
  <link
    rel="stylesheet"
    href="https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.1.1/animate.min.css"
  />
</head>

<h1 class="animate__animated animate__bounce">An animated element</h1>
```

## 代码高亮插件prism

地址：[https://prismjs.com/](https://prismjs.com/)

## 自动生成目录tocbot

地址：[https://github.com/tscanlin/tocbot](https://github.com/tscanlin/tocbot)

## 二维码生成插件qrcode

地址：[https://github.com/davidshimjs/qrcodejs](https://github.com/davidshimjs/qrcodejs)

## 平滑滚动jQuery.scrollTo

地址：[https://github.com/flesler/jquery.scrollTo](https://github.com/flesler/jquery.scrollTo)

## 滚动侦测waypoints

地址：[https://github.com/imakewebthings/waypoints](https://github.com/imakewebthings/waypoints)

文档：[http://imakewebthings.com/waypoints/guides/getting-started/](http://imakewebthings.com/waypoints/guides/getting-started/)

