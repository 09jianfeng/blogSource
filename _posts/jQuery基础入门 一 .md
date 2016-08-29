---
title: jQuery基础入门 一
date: 2016-06-18 21:22:11
categories: [前端,jQuery]
tags: [前端,jQuery]
---

参考资料：

[慕课网 jQuery基础](http://www.imooc.com/learn/11)


# jQuery环境配置
去官网<http://jquery.com>下载最新的 jQuery库。

通过下面的方式引入：

```
<script src="jquery-3.0.0.min.js" type="text/javascript"></script>
```

# 初识jQuery
如果你了解JavaScript语言，那将对你掌握jQuery如虎添翼，因为jQuery本身就是JavaScript，只不过是把JavaScript代码包装成拿过来就能实现特定功能的代码库！例如，我们想改变页面中所有段落标签中的文本内容：

javaScript代码：

```
var page_ps = document.getElementsByName("p");
for(var i = 0 ; i < page_ps.length ; i++){
	page_ps[i].innerHTML = "Hello mukewang";
}
```
jQuery代码：

```
$("p").html("hello mukewang");
```

以上两段代码完成的功能是一样的。由此可以看出，jQuery更加的简洁方便，我们在处理DOM时不必关心功能的实现细节。    `$()`就是`jQuery`中的函数，它的功能是获得`（）`中指定的标签元素。如示例中`$(“p”)`会得到一组P标签元素,其中`“p”`表示CSS中的标签选择器。`$()`中的`()`不一定是指定元素，也可能是函数。

在jQuery中 `$()`方法等价于`jQuery()`方法,前者比较常用，是后者的简写。一般只有在`$()`与其它语言冲突时才会使用`jQuery()`方法。

# 基础选择器

## id选择器

格式 `$("#my_id")`,其中`#my_id`表示根据id选择器获取页面中指定的标签元素，且返回唯一一个元素。 

## 元素选择器
格式 `$(“element”)` ,其中 `element`就是元素名称

## .class选择器
格式 `$(“.class”)`,其中`.class`参数表示元素的CSS类别(类选择器)名称

## * 选择器
格式为 `$(“*”)`,它的功能是获取页面中的全部元素，“全部”啊!包括`<head>、<body>、<script>`这些元素，相当于可以取走你文具盒中的所有铅笔。由于该选择器的特殊性，它常与其他元素组合使用，表示获取其他元素中的全部子元素。

```
    <body>
        <form action="#">
        <input id="Button1" type="button" value="button" />
        <input id="Text1" type="text" />
        <input id="Radio1" type="radio" />
        <input id="Checkbox1" type="checkbox" />
        </form>
        
        <script type="text/javascript">
            $("form *").attr("disabled", "true");
        </script>
    </body>
```

## 多项选择器
格式为 `$(“sele1,sele2,seleN”)`,其中参数`sele1、sele2`到`seleN`为有效选择器，每个选择器之间用“，”号隔开，它们可以是之前提及的各种类型选择器，如`$(“#id”)、$(“.class”)、$(“selector”)`选择器等。 **这个会比较影响性能，慎用。**

```
<body>
        <div class="red">选我吧！我是red</div>
        <div class="green">选我吧！我是green</div>
        <div class="blue">选我吧！我是blue</div>
        
        <script type="text/javascript">
            $(".red,.green,.blue").html("hi,我们的样子很美哦!");
        </script>
    </body>
```

## 层次选择器
### ance,desc选择器
格式 `$("ance desc")`,其中ance desc是使用空格隔开的两个参数。ance参数（ancestor祖先的简写）表示父元素；desc参数（descendant后代的简写）表示后代元素，即包括子元素、孙元素等等。两个参数都可以通过选择器来获取。比如家族姓氏“div”，家族几代人里，都有名字里带“span”的，就可以用这个ance desc选择器把这几个人给定位出来。

```
	<body>
        <div>码农家族
            <p>
               <label></label>
            </p>
            <label></label>
        </div>
        
        <script type="text/javascript">
            $("div label").css("background-color","blue");
        </script>
    </body>
    
    把label的背景色都变成了 蓝色。
```

### parent > child选择器
格式 `$(“parent > child”)` , 与上面的 ance desc选择器不一样的是，这个选择器止步于子元素，不包括孙子元素以及更深的层次

### prev + next选择器
格式 `$(“prev + next”)`,其中参数prev为任何有效的选择器，参数“next”为另外一个有效选择器，它们之间的“+”表示一种上下的层次关系，也就是说，“prev”元素最紧邻的下一个元素由“next”选择器返回的并且只返回唯的一个元素。

### prev ~ siblings选择器
与上一节中介绍的prev + next层次选择器相同，prev ~ siblings选择器也是查找prev 元素之后的相邻元素，但前者只获取第一个相邻的元素，而后者则获取prev 元素后面全部相邻的元素，它的调用格式如下：`$(“prev ~ siblings”`。

例如：

**index.html文件**

```
<!DOCTYPE html>
<html>
    <head>
        <title>prev ~ siblings选择器</title>
        <script src="http://libs.baidu.com/jquery/1.9.0/jquery.js" type="text/javascript"></script>
        <link href="style.css" rel="stylesheet" type="text/css" />
    </head>
    
    <body>
        <div>
            码农家族
            <label></label>
            <p></p>
            <label></label>
            <label></label>
        </div>
        <label></label>
        
        <script type="text/javascript">
            $("p ~ label").css("border", "solid 1px red");
            $("p ~ label").html("我们都是p先生的粉丝");
        </script>
    </body>
```

**style.css文件**

```
div, p, label
{
    float: left;
    border: solid 1px #ccc;
    margin: 5px;
    padding: 5px;
}
p,label
{
    width:230px;
    height:30px;
}
p
{
    border: solid 1px red;
}
```

## 过滤选择器
### :first/:last 选择器
该类型的选择器是根据某过滤规则进行元素的匹配，书写时以“：”号开头,通常用于查找集合元素中的某一位置的单个元素。例如想得到一组相同标签元素中的第1个元素的格式是这样的：
` $(“li:first”)` 。`:first`过滤选择器的功能是获取第一个元素，常常与其它选择器一起使用，获取指定的一组元素中的第一个元素。 `:last`是选择最后一个，

### :eq(index)过滤选择器
如果想从一组标签元素数组中，灵活选择任意的一个标签元素，我们可以使用`:eq(index)`。其中参数index表示索引号（即：一个整数），它从0开始，如果index的值为3，表示选择的是第4个元素。

```
	<body>
        <div>改变中间行"葡萄"背景颜色：</div>
        <ol>
        <li>橘子</li>
        <li>香蕉</li>
        <li>葡萄</li>
        <li>苹果</li>
        <li>西瓜</li>
        </ol>
        
        <script type="text/javascript">
            $("li:eq(2)").css("background-color", "#60F");
        </script>
    </body>
```

### :contains(text)过滤选择器
与上一节介绍的`:eq(index)`选择器按索引查找元素相比，有时候我们可能希望按照文本内容来查找一个或多个元素，那么使用`:contains(text)`选择器会更加方便， 它的功能是选择包含指定字符串的全部元素，它通常与其他元素结合使用，获取**包含`“text”`字符串内容**的全部元素对象。其中参数`text`表示页面中的文字。

```
<body>
        <div>改变包含"jQuery"字符内容的背景色：</div>
        <ol>
            <li>强大的"jQuery"</li>
            <li>"javascript"也很实用</li>
            <li>"jQuery"前端必学</li>
            <li>"java"是一种开发语言</li>
            <li>前端利器——"jQuery"</li>
        </ol>
        
        <script type="text/javascript">
            $("li:contains('jQuery')").css("background", "green");
        </script>
    </body>
```

### :has(selector)过滤选择器
除了在上一小节介绍的使用包含的字符串内容过滤元素之外，还可以使用包含的元素名称来过滤，`:has(selector)`过滤选择器的功能是获取选择器中**包含指定元素名称**的全部元素，其中selector参数就是包含的元素名称，是被包含元素。

```
	<body>
        <div>改变包含"label"元素的背景色：</div>
        <ol>
            <li><p>我是P先生</p></li>
            <li><label>L妹纸就是我</label></li>
            <li><p>我也是P先生</p></li>
            <li><label>我也是L妹纸哦</label></li>
            <li><p>P先生就是我哦</p></li>
        </ol>
        
         <script type="text/javascript">
            $("li:has('label')").css("background-color", "blue");
        </script>
    </body>
```

### :hidden过滤选择器
`:hidden`过滤选择器的功能是获取全部不可见的元素，这些不可见的元素中包括type属性值为hidden的元素。

### :visible过滤选择器
与上一节的`:hidden`过滤选择器相反，:visible过滤选择器获取的是全部可见的元素，也就是说，只要不将元素的display属性值设置为“none”，那么，都可以通过该选择器获取。

### [attribute=value]属性选择器
属性作为DOM元素的一个重要特征，也可以用于选择器中，从本节开始将介绍通过元素属性获取元素的选择器，`[attribute=value]`属性选择器的功能是获取与属性名和属性值完全相同的全部元素，其中[]是专用于属性选择器的括号符，参数attribute表示属性名称，value参数表示属性值。

```
	<body>
        <h3>改变"title"属性值为"蔬菜"的背景色</h3>
        <ul>
            <li title="蔬菜">茄子</li>
            <li title="水果">香蕉</li>
            <li title="蔬菜">芹菜</li>
            <li title="水果">苹果</li>
            <li title="水果">西瓜</li>
        </ul>
        
        <script type="text/javascript">
            $("li[title='蔬菜']").css("background-color", "green");
        </script>
    </body>
```

反之还有 `[attribute!=value]` 这个选择器

### [attribute*=value]属性选择器
`[attribute*=value]`，它可以获取属性值中包含指定内容的全部元素，其中[]是专用于属性选择器的括号符，参数attribute表示属性名称，value参数表示对应的属性值。

```
	<body>
        <h3>改变"title"属性值包含"果"的背景色</h3>
        <ul>
            <li title="蔬菜">茄子</li>
            <li title="水果">香蕉</li>
            <li title="蔬菜">芹菜</li>
            <li title="人参果">小西红柿</li>
            <li title="水果">西瓜</li>
        </ul>
        
        <script type="text/javascript">
            $("li[title*='果']").css("background-color", "green");
        </script>
    </body>
```
选择上面的例子是 属性包含 `果`字的元素

### :first-child子元素过滤选择器
我们知道使用`:first`过滤选择器可以获取指定父元素中的首个子元素，但该选择器返回的只有一个元素，并不是一个集合，而使用`:first-child`子元素过滤选择器则可以获取每个父元素中返回的首个子元素，它是一个集合，常用多个集合数据的选择处理。

```
   <body>
        <h3>改变每个"蔬菜水果"中第一行的背景色</h3>
        <ol>
            <li>芹菜</li>
            <li>茄子</li>
            <li>萝卜</li>
            <li>大白菜</li>
            <li>西红柿</li>
        </ol>
        <ol>
            <li>橘子</li>
            <li>香蕉</li>
            <li>葡萄</li>
            <li>苹果</li>
            <li>西瓜</li>
        </ol>
        
        <script type="text/javascript">
            $("li:first-child").css("background-color", "green");
        </script>
    </body>
```

上面的例子选择了 `芹菜与西瓜` 这两个元素

 反之 `:last-child `可以推理出来

## 表单选择器
### :input表单选择器
如何获取表单全部元素`？:input`表单选择器可以实现，它的功能是返回全部的表单元素，不仅包括所有`<input>`标记的表单元素，而且还包括`<textarea>`、`<select>` 和 `<button>`标记的表单元素，因此，它选择的表单元素是最广的。

```
index.html

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <title>:input表单选择器</title>
        <script src="http://libs.baidu.com/jquery/1.9.0/jquery.js" type="text/javascript"></script>
        <link href="style.css" rel="stylesheet" type="text/css" />
    </head>
    
    <body>
        <h3>修改全部表单元素的背景色</h3>
        <form id="frmTest" action="#">
        <input type="button" value="Input Button" /><br />
        <select>
            <option>Option</option>
        </select><br />
        <textarea rows="3" cols="8"></textarea><br />
        <button>
            Button</button><br />
        </form>
        
        <script type="text/javascript">
            $("#frmTest :input").addClass("bg_blue");
        </script>
    </body>
</html>
    
 style.css--
 
 .bg_blue
{
    background-color: blue;
}
 
```

上面的代码给表单所有的元素加上蓝色背景


### :text表单文本选择器
`:text`表单文本选择器可以获取表单中全部单行的文本输入框元素，单行的文本输入框就像一个不换行的字条工具，使用非常广泛。

### :password表单密码选择器
如果想要获取密码输入文本框，可以使用`:password`选择器，它的功能是获取表单中全部的密码输入文本框元素。

### :radio单选按钮选择器
表单中的单选按钮常用于多项数据中仅选择其一，而使用`:radio`选择器可轻松获取表单中的**全部单选按钮元素**。

### :checkbox复选框选择器
表单中的复选框常用于多项数据的选择，使用`:checkbox`选择器可以快速定位并获取表单中的复选框元素。

### :submit提交按钮选择器
通常情况下，一个表单中只允许有一个“type”属性值为“submit”的提交按钮，使用:submit选择器可获取表单中的这个提交按钮元素。

```
    <body>
        <h3>修改表单中提交按钮的背景色</h3>
        <form id="frmTest" action="#">
        <input type="button" value="Input Button" /><br />
        <input type="submit" value="点我就提交了" /><br />
        <button>
            Button</button><br />
        </form>
        
        <script type="text/javascript">
            $("form input:submit").addClass("bg_red");
        </script>
    </body>
```


### :image图像域选择器
当一个`<input>`元素的`“type”`属性值设为`“image”`时，该元素就是一个图像域，使用:image选择器可以快速获取该类全部元素。

### :button表单按钮选择器
表单中包含许多类型的按钮，而使用`:button`选择器能获取且只能获取`“type”`属性值为`“button”`的`<input>`和`<button>`这两类普通按钮元素。

### :checked选中状态选择器
有一些元素存在选中状态，如复选框、单选按钮元素，选中时`“checked”`属性值为`“checked”`，调用`:checked`可以获取处于选中状态的全部元素。

### :selected选中状态选择器
与`:checked`选择器相比，`:selected`选择器只能获取`<select>`下拉列表框中全部处于选中状态的`<option>`选项元素。

```
    <body>
        <h3>获取处于选中状态元素的内容</h3>
        <form id="frmTest" action="#">
        <select id="Select1" multiple="multiple">
            <option value="0">苹果</option>
            <option value="1" selected="selected">桔子</option>
            <option value="2">荔枝</option>
            <option value="3" selected="selected">葡萄</option>
            <option value="4">香蕉</option>
        </select><br /><br />
        <div id="tip"></div>
        </form>
        
        <script type="text/javascript">
            var $txtOpt = $("#frmTest :selected").text();
            $("#tip").html("选中内容为:" + $txtOpt);
        </script>
    </body>
```




# jQuery操作DOM元素
## 使用attr()方法控制元素的属性
`attr()`方法的作用是设置或者返回元素的属性，其中`attr(属性名)`格式是获取元素属性名的值，`attr(属性名，属性值)`格式则是设置元素属性名的值。

例如，使用attr(属性名)的格式获取页面中`<a>`元素的`“href”`属性值，并将该值的内容显示在`<span>`元素中，如下图所示：

![511](/img/jquery_5_1_1.jpg)

在浏览器中显示的效果：

![512](/img/jquery_5_1_2.jpg)

从图中可以看出，通过attr()方法可以方便地获取元素中指定属性名称的内容，并将获取的内容通过html()方法显示在页面中。

## 操作元素的内容
使用html()和text()方法操作元素的内容，当两个方法的参数为空时，表示获取该元素的内容，而如果方法中包含参数，则表示将参数值设置为元素内容。

## 操作元素的样式
通过addClass()和css()方法可以方便地操作元素中的样式，前者括号中的参数为增加元素的样式名称，后者直接将样式的属性内容写在括号中。

## 移除属性和样式
使用removeAttr(name)和removeClass(class)分别可以实现移除元素的属性和样式的功能，前者方法中参数表示移除属性名，后者方法中参数则表示移除的样式名

## 使用append()方法向元素内追加内容
`append(content)`方法的功能是向指定的元素中追加内容，被追加的`content`参数，可以是字符、`HTML`元素标记，还可以是一个返回字符串内容的函数。

例如，在页面的`<body>`元素中追加一个具有“id”、“title”属性和显示内容的`<div>`元素，如下图所示：

![551](/img/jry_551.jpg)

在浏览器中显示的效果：

![552](/img/jry_552.jpg)

从图中可以看出，由于使用append()方法在<body>元素中追加了一些HTML 元素标记，因此追加后，这些元素标记直接生成对应的元素和属性显示在页面中。

## 使用appendTo()方法向被选元素内插入内容
`appendTo()`方法也可以向指定的元素内插入内容，它的使用格式是：

`$(content).appendTo(selector)`

参数`content`表示需要插入的内容，参数`selector`表示被选的元素，即把`content`内容插入`selector`元素内，默认是在尾部。

例如，使用`appendTo()`方法，将`<div>`外的`<span>`元素插入`<div>`内，如下图所示：

![561](/img/jqy561.jpg)

在浏览器中显示的效果：

![562](/img/jqr562.jpg)

从图中可以看出，使用`appendTo()`方法将类别名为“red”的`<span>`元素插入到`<div>`元素的尾部，相当于追加的效果。


## 使用before()和after()在元素前后插入内容
使用`before()`和`after()`方法可以在元素的前后插入内容，它们分别表示在整个元素的前面和后面插入指定的元素或内容，调用格式分别为：

`$(selector).before(content)`和`$(selector).after(content)`

其中参数`content`表示插入的内容，该内容可以是元素或HTML字符串。

例如，调用`before()`方法在一个`<span>`元素插入另一个`<span>`元素，如下图所示：

![571](/img/jqy571.jpg)

在浏览器中显示的效果：

![572](/img/jqy572.jpg)

从图中可以看出，使用`before()`方法将`HTML`格式的内容插入到原有`<span>`元素内容之前，而并不仅是它的内部文本。

## 使用clone()方法复制元素
调用`clone()`方法可以生成一个被选元素的副本，即复制了一个被选元素，包含它的节点、文本和属性，它的调用格式为：

`$(selector).clone()`

其中参数`selector`可以是一个元素或`HTML`内容。

```
    <body>
        <h3>使用clone()方法复制元素</h3>
        <span class="red" title="hi">我是美猴王</span>
        
        <script type="text/javascript">
            $("body").append($(".red").clone());
        </script>
    </body>
```

上面的例子复制了 `<span>` 在 `<span>`后面

## 替换内容
`replaceWith()`和`replaceAll()`方法都可以用于替换元素或元素中的内容，但它们调用时，内容和被替换元素所在的位置不同，分别为如下所示：

`$(selector).replaceWith(content)`和`$(content).replaceAll(selector)`

参数`selector`为被替换的元素，`content`为替换的内容。

```
    <body>
        <h3>使用replaceAll()方法替换元素内容</h3>
        <span class="green" title="hi">我是屌丝</span>
        
        <script type="text/javascript">
            var $html = "<span class='red' title='hi'>我是土豪</span>";
            $(".green").replaceAll($html);
        </script>
    </body>
```

## 使用wrap()和wrapInner()方法包裹元素和内容
`wrap()`和`wrapInner()`方法都可以进行元素的包裹，但前者用于包裹元素本身，后者则用于包裹元素中的内容，它们的调用格式分别为：

`$(selector).wrap(wrapper)`和`$(selector).wrapInner(wrapper)`

参数`selector`为被包裹的元素，`wrapper`参数为包裹元素的格式。

```
    <body>
        <h3>使用wrapInner()方法包裹元素</h3>
        <span class="red" title='hi'>我的身体有点歪</span>
        
        <script type="text/javascript">
            $(".red").wrapInner("<i><i>");
        </script>
    </body>
```

例子调用wrapInner()方法将页面中的`<span>`元素内的文字字体变成斜体。

## 使用each()方法遍历元素
使用`each()`方法可以遍历指定的元素集合，在遍历时，通过回调函数返回遍历元素的序列号，它的调用格式为：

`$(selector).each(function(index))`

参数function为遍历时的回调函数，index为遍历元素的序列号，它从0开始。

```
    <body>
        <h3>使用each()方法遍历元素</h3>
        <span class="green">香蕉</span>
        <span class="green">桃子</span>
        <span class="green">葡萄</span>
        <span class="green">荔枝</span>
        
        <script type="text/javascript">
            $("span").each(function (index) {
                if (index == 1) {
                    $(this).attr("class", "red");
                }
            });
        </script>
    </body>
```


通过each()遍历方法，改变第2个元素<span>元素的背景色为红色。

## 使用remove()和empty()方法删除元素
`remove()`方法删除所选元素本身和子元素，该方法可以通过添加过滤参数指定需要删除的某些元素，而`empty()`方法则只删除所选元素的子元素。

```
    <body>
        <h3>使用empty()方法删除元素</h3>
        <span class="green">香蕉</span>
        <span class="red">桃子</span>
        <span class="green">葡萄</span>
        <span class="green">荔枝</span>
        
        <script type="text/javascript">
            $("span").empty()
            $("span").remove(".red")
        </script>
    </body>
```

使用empty()方法删除全部<span>元素的子元素内容,删除css类为 red的元素

# jQuery事件与应用
## 页面加载时触发ready()事件
`ready()`事件类似于`onLoad()`事件，但前者只要页面的DOM结构加载后便触发，而后者必须在页面全部元素加载成功才触发，`ready()`可以写多个，按顺序执行。此外，下列写法是相等的：

`$(document).ready(function(){})`等价于`$(function(){})`;


例如，当触发页面的`ready()`事件时，在`<div>`元素中显示一句话。如下图所示：

![611](/img/jqy611.jpg)

在浏览器中显示的效果：

![612](/img/jqy612.jpg)

从图中可以看出，当页面的DOM框架完成加载后，便触发ready()事件，在该事件中，通过id号为“tip”的元素，调用html()方法在页面中显示一段字符。

## 使用bind()方法绑定元素的事件
`bind()`方法绑定元素的事件非常方便，绑定前，需要知道被绑定的元素名，绑定的事件名称，事件中执行的函数内容就可以，它的绑定格式如下：

`$(selector).bind(event,[data] function)`

参数`event`为事件名称，多个事件名称用空格隔开，`function`为事件执行的函数。

```
    <body>
        <h3>页面载入时触发ready()事件</h3>
        <div id="tip"></div>
        <input id="btntest" type="button" value="点下我" />
        
        <script type="text/javascript">
            $(document).ready(function() {
                $("#btntest").bind("click", function () {
                    $("#tip").html("我被点击了！");
                });
            });
        </script>
    </body>
```

在ready()事件中，绑定一个按钮的单击事件。

```
    <body>
        <h3>bind()方法绑多个事件</h3>
        <input id="btntest" type="button" value="点击或移出就不可用了" />
        
        <script type="text/javascript">
            $(function () {
                $("#btntest").bind("click mouseout", function () {
                    $(this).attr("disabled", "true");
                })
            });
        </script>
    </body>
```

使用bind()方法绑定单击(click)和鼠标移出(mouseout)这两个事件，触发这两个事件中，按钮将变为不可用。


## 使用hover()方法切换事件
hover()方法的功能是当鼠标移到所选元素上时，执行方法中的第一个函数，鼠标移出时，执行方法中的第二个函数，实现事件的切实效果，调用格式如下：

`$(selector).hover(over，out);`

over参数为移到所选元素上触发的函数，out参数为移出元素时触发的函数。

```
    <body>
        <h3>hover()方法切换事件</h3>
        <div>别走！你就是土豪</div>
        
        <script type="text/javascript">
            $(function () {
                $("div").hover(
                function () {
                    $(this).addClass("orange");
                },
                function () {
                    $(this).removeClass("orange")
                })
            });
        </script>
    </body>
```

调用hover方法实现元素`<div>别走！你就是土豪</div>`背景色的切换。

## 使用toggle()方法绑定多个函数
toggle()方法可以在元素的click事件中绑定两个或两个以上的函数，同时，它还可以实现元素的隐藏与显示的切换，绑定多个函数的调用格式如下：

`$(selector).toggle(fun1(),fun2(),funN(),...)`

其中，fun1，fun2就是多个函数的名称

```
    <body>
        <h3>toggle()方法绑定多个函数</h3>
        <input id="btntest" type="button" value="点一下我" />
        <div>我是动态显示的</div>
        
        <script type="text/javascript">
            $(function () {
                $("#btntest").bind("click", function () {
                    $("div").toggle();
                });
            });
        </script>
    </body>
```

上面的代码例子是，使用toggle()方法控制元素的显示与隐藏属性,toggle()函数不传入参数的时候实现这个效果。

```
    <body>
        <h3>toggle()方法绑定多个函数</h3>
        <input id="btntest" type="button" value="点一下我" />
        <div>我是动态显示的</div>
        
        <script type="text/javascript">
            $(function () {
                $("div").toggle(function(){
                    $(this).html("aaa");
                },function(){
                    $(this).html("bbb");
                },function(){
                    $(this).html("ccc");
                });
            });
        </script>
    </body>
```

上面的代码例子是，toggle()函数传入参数，实现点击div元素，内容不断切换 aaa->bbb->ccc->aaa...

## unbind()
解除绑定的事件。

```
    <body>
        <h3>unbind()移除绑定的事件</h3>
        <input id="btntest" type="button" value="移除事件" />
        <div>土豪，咱们交个朋友吧</div>
        
        <script type="text/javascript">
            $(function () {
                $("div").bind("click",
	                function () {
	                    $(this).removeClass("backcolor").addClass("color");
	                }).bind("dblclick", function () {
	                    $(this).removeClass("color").addClass("backcolor");
	             });
	                
                $("#btntest").bind("click", function () {
                    $("div").unbind(;
                    $(this).attr("disabled", "true");
                });
            });
        </script>
    </body>
```

解除 div绑定的所有事件


## 使用one()方法绑定元素的一次性事件
one()方法可以绑定元素任何有效的事件，但这种方法绑定的事件只会触发一次，它的调用格式如下：

`$(selector).one(event,[data],fun)`

参数event为事件名称，data为触发事件时携带的数据，fun为触发该事件时执行的函数。

```
    <body>
        <h3>one()方法执行一次绑定事件</h3>
        <div>请点击我一下</div>
        
        <script type="text/javascript">
            $(function () {
                var intI = 0;
                $("div").one("click", function () {
                    intI++;
                    $(this).css("font-size", intI + "px");
                })
            });
        </script>
    </body>
```

点击 div会缩到最小，one里面的代码只会执行一次。

## 调用trigger()方法手动触发指定的事件
trigger()方法可以直接手动触发元素指定的事件，这些事件可以是元素自带事件，也可以是自定义的事件，总之，该事件必须能执行，它的调用格式为：

`$(selector).trigger(event)`

其中event参数为需要手动触发的事件名称。

```
    <body>
        <h3>trigger()手动触发事件</h3>
        <div>土豪，咱们交个朋友吧</div>
        
        <script type="text/javascript">
            $(function () {
                $("div").bind("change-color", function () {
                    $(this).addClass("color");
                });
                $("div").trigger("change-color");
            });
        </script>
    </body>
```

例如 在Ready事件中调用 trigger 触发 change-color这个事件

## 文本框的focus和blur事件
focus事件在元素获取焦点时触发，如点击文本框时，触发该事件；而blur事件则在元素丢失焦点时触发，如点击除文本框的任何元素，都会触发该事件。

例如，在触发文本框的“focus”事件时，<div>元素显示提示内容，如下图所示：

![681](/img/jqy681.jpg)

在浏览器中显示的效果：

![682](/img/jqy682.jpg)

从图中可以看出，当点击文本框时，触发文本框的“focus”事件，在该事件中，页面中的`<div>`元素显示提示信息。 绑定的 blur事件就是当离开文本框的时候，显示的信息。例如： 

```
$("input").bind("blur", function () {
                    if ($(this).val().length == 0)
                        $("div").html("你的名称不能为空！");
                })
```


## 下拉列表框的change事件
当一个元素的值发生变化时，将会触发change事件，例如在选择下拉列表框中的选项时，就会触change事件。

```
    <body>
        <h3>下拉列表的change事件</h3>
        <select id="seltest">
            <option value="葡萄">葡萄</option>
            <option value="苹果">苹果</option>
            <option value="荔枝">荔枝</option>
            <option value="香焦">香焦</option>
        </select>
        
        <script type="text/javascript">
            $(function () {
                $("select").bind("change", function () {
                    if ($(this).val() == "苹果")
                        $(this).css("background-color", "red");
                    else
                        $(this).css("background-color", "green");
                })
            });
        </script>
    </body>
```

如果下拉选中的是苹果，则背景是红色。否则是绿色

## 调用live()方法绑定元素的事件
与bind()方法相同，live()方法与可以绑定元素的可执行事件，除此相同功能之外，live()方法还可以绑定动态元素，即使用代码添加的元素事件，格式如下：

`$(selector).live(event,[data],fun)`

参数event为事件名称，data为触发事件时携带的数据，fun为触发该事件时执行的函数。

`jQuery 1.9以后不再支持 live()` 来绑定事件














